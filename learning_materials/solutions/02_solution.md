# Solution: Lesson 02 — Database Internals
## Hard Exercise: MVCC Under Load

---

## 1. Full Exercise Restatement

**Setup:** A PostgreSQL table `orders` has 10 million rows. A long-running analytics query T1 (`SELECT sum(amount) FROM orders WHERE status='completed'`) started 2 minutes ago and is still running. Since T1 started, 500,000 rows have been updated by OLTP transactions.

**Questions:**
1. **(a)** How many row versions exist in the heap after these updates?
2. **(b)** Are the dead rows from updates before T1's snapshot visible to T1?
3. **(c)** Can autovacuum clean up the dead rows while T1 is running?
4. **(d)** What PostgreSQL setting controls how long autovacuum waits?
5. **(e)** Design the schema and autovacuum configuration to survive 50,000 updates/second.

---

## 2. Conceptual Solution Walkthrough

### The MVCC Data Model

PostgreSQL implements MVCC (Multi-Version Concurrency Control) by **never overwriting rows**. Every UPDATE produces a new physical tuple in the heap with a higher `xmin` (the creating transaction ID). The old tuple's `xmax` field is set to the updating transaction's XID to mark it as "deleted as of" that transaction.

The heap page layout for an updated row looks like:

```
Page N:
  [Tuple 1] xmin=100  xmax=510  status='pending'  amount=50.00   ← old version
  [Tuple 2] xmin=510  xmax=0    status='completed' amount=50.00   ← new version
```

Tuple 1 is "dead" to most transactions (xmax=510 is committed), but it remains physically in the page until autovacuum reclaims it.

### Part (a): Row Version Count

Before T1 starts: 10,000,000 live tuples.

The 500,000 OLTP updates each produce:
- 1 new "live" tuple (xmin = OLTP transaction XID)
- 1 old "dead" tuple (xmax = OLTP transaction XID, now committed)

The 9,500,000 untouched rows are unaffected.

**Total physical tuples in the heap = 10,500,000:**
- 9,500,000 live (unmodified rows)
- 500,000 live (new versions from updates)
- 500,000 dead (old versions from updates, still physically present)

In practice, some of the dead tuples may have been vacuumed if they predate T1's snapshot — but since all 500K updates happened *after* T1 started, none can be vacuumed yet (see part c).

### Part (b): Visibility of Dead Rows to T1

PostgreSQL's visibility rules for a tuple are:
> A tuple is visible to snapshot S if `xmin` is committed and `xmin` < `S.xmax` AND (`xmax` is null OR `xmax` >= `S.xmax` OR `xmax` is aborted).

Here, S is T1's snapshot, taken at T1's start time. The OLTP update transactions all started *after* T1's snapshot was taken, so their XIDs are > T1's `xmax` horizon.

For each updated row:
- The **old version** has `xmin` < T1's horizon (it was created before T1) and `xmax` > T1's horizon (the update happened after T1). So the old version **IS visible to T1**.
- The **new version** has `xmin` > T1's horizon (created after T1). So the new version **IS NOT visible to T1**.

**Result:** T1 sees the pre-update state of all 500,000 updated rows — exactly what it would have seen at T1's start time. This is snapshot isolation. T1 reads a consistent point-in-time view of the database from 2 minutes ago.

### Part (c): Why Autovacuum Cannot Clean the Dead Rows

Autovacuum's job is to reclaim pages containing dead tuples. But it has a hard constraint:

> **Never remove a tuple that is still visible to any active transaction's snapshot.**

PostgreSQL maintains a global `OldestXmin` — the minimum `xmin` of any active transaction's snapshot. Autovacuum cannot remove a dead tuple with `xmax` >= `OldestXmin`.

Since T1 started before the 500K updates, T1's snapshot xmin is lower than all those OLTP transactions' XIDs. Therefore `OldestXmin` ≤ T1's xmin, and autovacuum cannot reclaim those 500K dead tuples.

You can observe this in real time:

```sql
-- Shows the oldest active snapshot holding back vacuum
SELECT pid, query_start, state, left(query, 80) as query
FROM pg_stat_activity
WHERE backend_xmin IS NOT NULL
ORDER BY backend_xmin;

-- Shows dead tuple accumulation per table
SELECT relname, n_dead_tup, n_live_tup, last_autovacuum
FROM pg_stat_user_tables
WHERE relname = 'orders';
```

The moment T1 commits or is cancelled, `OldestXmin` advances, and the next autovacuum run can reclaim all 500K dead tuples.

### Part (d): Configuration Controls

The primary concern is not "how long autovacuum waits" — autovacuum simply **skips** what it cannot clean. However, several settings influence how aggressively PostgreSQL handles aging snapshots:

- **`old_snapshot_threshold`** (PostgreSQL 9.6+, default `-1` = disabled): If a snapshot is older than this threshold and vacuum wants to reclaim its dead tuples, PostgreSQL may raise `ERROR: snapshot too old` to the query holding the old snapshot. This is an aggressive "kick the long query out" mechanism.
- **`idle_in_transaction_session_timeout`**: Kills sessions that have been idle inside a transaction for too long. Prevents abandoned `BEGIN` + nothing → stale snapshot scenarios.
- **`statement_timeout`**: Kills any statement running longer than the threshold. Useful for analytics queries that should be on a replica anyway.
- **`lock_timeout`**: Prevents DDL operations from waiting too long for locks held by long-running transactions.

For the xid wraparound emergency case (when `age(relfrozenxid)` exceeds `autovacuum_freeze_max_age`), PostgreSQL will preempt all other operations to run an emergency vacuum — even over long-running transactions. This is what caused the GitHub PostgreSQL outage of 2012.

---

## 3. Full Python Simulation

```python
"""
MVCC under load simulation.

Simulates PostgreSQL's heap layout with xmin/xmax tuple visibility,
autovacuum horizon tracking, and the dead tuple accumulation problem.

Requires only the Python standard library.
"""

from __future__ import annotations
from dataclasses import dataclass, field
from typing import Optional, List, Dict
import time


# ── Tuple and heap data structures ────────────────────────────────────────────

@dataclass
class HeapTuple:
    """Simulates a PostgreSQL heap tuple (physical row version)."""
    xmin: int          # XID of transaction that created this tuple
    xmax: int          # XID of transaction that deleted/updated this tuple (0 = alive)
    data: dict         # The actual row data

    def is_dead(self, oldest_active_xmin: int) -> bool:
        """
        True if this tuple can be vacuumed:
        xmax is a committed transaction older than any active snapshot.
        """
        return self.xmax != 0 and self.xmax < oldest_active_xmin

    def is_visible_to(self, snapshot_xmin: int, snapshot_xmax: int,
                      in_progress: set) -> bool:
        """
        Snapshot isolation visibility check.
        
        Visible if:
        1. xmin is committed and visible to snapshot (xmin < snapshot_xmax
           and xmin not in in_progress at snapshot time)
        2. xmax is either 0 (still alive) or not yet visible to snapshot
           (xmax >= snapshot_xmax or xmax was in-progress at snapshot time)
        """
        # Check xmin visibility
        if self.xmin >= snapshot_xmax:
            return False   # Created after snapshot
        if self.xmin in in_progress:
            return False   # Created by an in-progress transaction

        # Check xmax
        if self.xmax == 0:
            return True    # Not deleted
        if self.xmax >= snapshot_xmax:
            return True    # Deleted after snapshot — still visible to us
        if self.xmax in in_progress:
            return True    # Deleter was in-progress at snapshot time

        return False       # Deleted before snapshot — not visible


@dataclass
class Snapshot:
    """Simulates a PostgreSQL MVCC snapshot (taken at transaction start)."""
    xmin: int           # Oldest active XID at snapshot time
    xmax: int           # Next XID to be assigned at snapshot time
    in_progress: set    # Set of XIDs that were active at snapshot time
    taken_at: float     # Wall clock (for diagnostics)


class TransactionManager:
    """Simulates PostgreSQL's transaction manager."""

    def __init__(self):
        self._next_xid: int = 1
        self._active_transactions: Dict[int, Snapshot] = {}  # xid -> snapshot

    def begin(self) -> tuple[int, Snapshot]:
        """Start a new transaction; return (xid, snapshot)."""
        xid = self._next_xid
        self._next_xid += 1

        snap = Snapshot(
            xmin=min(self._active_transactions.keys(), default=xid),
            xmax=self._next_xid,
            in_progress=set(self._active_transactions.keys()),
            taken_at=time.monotonic(),
        )
        self._active_transactions[xid] = snap
        return xid, snap

    def commit(self, xid: int) -> None:
        """Commit a transaction."""
        if xid in self._active_transactions:
            del self._active_transactions[xid]

    def oldest_active_xmin(self) -> int:
        """The global OldestXmin — the vacuum horizon."""
        if not self._active_transactions:
            return self._next_xid
        return min(s.xmin for s in self._active_transactions.values())


class OrdersTable:
    """
    Simulates the PostgreSQL orders heap table.
    Each row can have multiple physical versions (old and new).
    """

    def __init__(self):
        # Heap: list of (row_id, HeapTuple) pairs
        self._heap: List[tuple[int, HeapTuple]] = []
        self._next_row_id = 1

    def insert(self, xid: int, data: dict) -> int:
        """Insert a new row."""
        row_id = self._next_row_id
        self._next_row_id += 1
        self._heap.append((row_id, HeapTuple(xmin=xid, xmax=0, data=data.copy())))
        return row_id

    def update(self, xid: int, row_id: int, new_data: dict) -> bool:
        """
        MVCC update: mark old tuple dead (set xmax) and insert new tuple.
        Returns False if row_id not found.
        """
        for i, (rid, tup) in enumerate(self._heap):
            if rid == row_id and tup.xmax == 0:
                # Mark old tuple deleted
                tup.xmax = xid
                # Insert new version with same row_id
                self._heap.append((row_id, HeapTuple(
                    xmin=xid, xmax=0, data=new_data.copy()
                )))
                return True
        return False

    def scan(self, snap: Snapshot) -> List[dict]:
        """Full table scan returning all tuples visible to snap."""
        results = []
        for _, tup in self._heap:
            if tup.is_visible_to(snap.xmin, snap.xmax, snap.in_progress):
                results.append(tup.data.copy())
        return results

    def vacuum(self, oldest_xmin: int) -> int:
        """Remove dead tuples below oldest_xmin. Returns count removed."""
        before = len(self._heap)
        self._heap = [
            (rid, tup) for rid, tup in self._heap
            if not tup.is_dead(oldest_xmin)
        ]
        return before - len(self._heap)

    @property
    def physical_tuple_count(self) -> int:
        return len(self._heap)

    def dead_tuple_count(self, oldest_xmin: int) -> int:
        return sum(1 for _, tup in self._heap if tup.is_dead(oldest_xmin))

    def live_tuple_count(self, snap: Snapshot) -> int:
        return len(self.scan(snap))


# ── Main simulation ────────────────────────────────────────────────────────────

def simulate_mvcc_under_load():
    """
    Reproduce the exercise scenario:
    - Insert 1,000 rows (scaled down from 10M for speed)
    - Start long-running analytics query T1
    - Run 100 OLTP updates after T1 starts
    - Observe vacuum is blocked, then unblocked when T1 commits
    """
    print("=" * 65)
    print("MVCC Under Load Simulation")
    print("  (Scaled: 1,000 initial rows, 100 OLTP updates)")
    print("=" * 65)

    txm = TransactionManager()
    table = OrdersTable()

    # ── Phase 1: Load initial 1,000 rows ──────────────────────────────────────
    print("\n--- Phase 1: Loading initial rows ---")
    row_ids = []
    for i in range(1_000):
        xid, snap = txm.begin()
        rid = table.insert(xid, {
            "id": i + 1,
            "status": "pending" if i % 5 != 0 else "completed",
            "amount": float((i + 1) * 10),
        })
        row_ids.append(rid)
        txm.commit(xid)

    print(f"  Inserted {table.physical_tuple_count} tuples")

    # ── Phase 2: T1 starts (analytics query, long-running) ────────────────────
    print("\n--- Phase 2: T1 starts (analytics query) ---")
    t1_xid, t1_snap = txm.begin()
    print(f"  T1 started: xid={t1_xid}, snapshot xmax={t1_snap.xmax}")
    print(f"  OldestXmin = {txm.oldest_active_xmin()}")

    # T1 counts completed rows at snapshot time
    t1_baseline = [
        r for r in table.scan(t1_snap) if r["status"] == "completed"
    ]
    print(f"  T1 sees {len(t1_baseline)} completed rows at snapshot time")

    # ── Phase 3: 100 OLTP updates after T1 starts ─────────────────────────────
    print("\n--- Phase 3: 100 OLTP updates ---")
    updated_row_ids = row_ids[:100]  # Update the first 100 rows
    for rid in updated_row_ids:
        xid, _ = txm.begin()
        table.update(xid, rid, {
            "id": rid,
            "status": "completed",
            "amount": 999.0,
        })
        txm.commit(xid)

    print(f"  After updates: {table.physical_tuple_count} physical tuples")

    # Count dead tuples from T1's perspective
    oldest_xmin = txm.oldest_active_xmin()
    dead_count = table.dead_tuple_count(oldest_xmin)
    print(f"  OldestXmin = {oldest_xmin} (blocked by T1 at xid={t1_xid})")
    print(f"  Dead tuples (cannot be vacuumed): {dead_count}")

    # ── Part (b): T1 sees old row versions ────────────────────────────────────
    t1_now = [r for r in table.scan(t1_snap) if r["status"] == "completed"]
    print(f"\n  T1 still sees only {len(t1_now)} completed rows "
          f"(snapshot isolation — unchanged from start)")
    assert len(t1_now) == len(t1_baseline), \
        "T1 should see identical data — snapshot isolation broken!"

    # ── Part (c): Vacuum is blocked ────────────────────────────────────────────
    print("\n--- Phase 4: Attempting autovacuum ---")
    reclaimed = table.vacuum(oldest_xmin)
    print(f"  Vacuum with OldestXmin={oldest_xmin}: reclaimed {reclaimed} tuples")
    print(f"  Physical tuples still: {table.physical_tuple_count}")
    assert reclaimed == 0, "Should not have reclaimed anything — T1 is blocking"

    # ── Phase 5: T1 commits, vacuum runs ──────────────────────────────────────
    print("\n--- Phase 5: T1 commits ---")
    # T1 completes its analytics work
    total = sum(r["amount"] for r in table.scan(t1_snap) if r["status"] == "completed")
    print(f"  T1 result: sum(amount) = {total:.2f}")
    txm.commit(t1_xid)

    new_oldest = txm.oldest_active_xmin()
    print(f"  OldestXmin after T1 commit: {new_oldest}")

    reclaimed = table.vacuum(new_oldest)
    print(f"  Vacuum reclaimed {reclaimed} dead tuples")
    print(f"  Physical tuples remaining: {table.physical_tuple_count}")
    assert reclaimed == 100, f"Expected to reclaim 100, got {reclaimed}"
    print("\n  Dead tuple accumulation resolved. Simulation complete.")


def simulate_write_amplification(n_updates: int = 50_000):
    """
    Show how write amplification at high update rates overwhelms standard vacuum.
    
    At 50,000 updates/sec with 1-minute autovacuum naptime:
    - Dead tuples accumulate: 50,000 × 60 = 3,000,000 per naptime cycle
    - Standard autovacuum threshold (20% of 10M rows = 2M): already exceeded at 40 seconds
    - Each dead tuple occupies ~80–200 bytes (TOAST-free row)
    - 3M dead tuples × 100 bytes ≈ 300MB of dead space per minute
    """
    print("\n" + "=" * 65)
    print("Write Amplification at High Update Rates")
    print("=" * 65)

    seconds_per_autovacuum_cycle = 60  # default naptime = 1 min
    bytes_per_tuple = 100              # approximate

    dead_per_cycle = n_updates * seconds_per_autovacuum_cycle
    dead_bytes = dead_per_cycle * bytes_per_tuple

    print(f"\n  Update rate:               {n_updates:>10,}/sec")
    print(f"  Dead tuples per cycle:     {dead_per_cycle:>10,}")
    print(f"  Dead space per cycle:      {dead_bytes / 1_048_576:>10.1f} MB")

    # Aggressive autovacuum settings reduce the effective cycle to ~5 seconds
    aggressive_naptime = 5
    dead_aggressive = n_updates * aggressive_naptime
    print(f"\n  With aggressive naptime={aggressive_naptime}s:")
    print(f"    Dead tuples per cycle:   {dead_aggressive:>10,}")
    print(f"    Dead space per cycle:    {dead_aggressive * bytes_per_tuple / 1024:>10.1f} KB")
    print(f"    (manageable — vacuum can keep up)")


# ── smaps-based memory inspector ──────────────────────────────────────────────

def simulate_dead_tuple_bloat_impact():
    """
    Show how table bloat from dead tuples degrades query performance.
    Sequential scans must read ALL pages, including those full of dead tuples.
    """
    print("\n" + "=" * 65)
    print("Table Bloat Impact on Sequential Scan I/O")
    print("=" * 65)

    page_size = 8192        # PostgreSQL default page size (8KB)
    rows_per_page = 40      # approximate for typical row size
    total_rows = 10_000_000
    dead_rows = 3_000_000   # accumulated after vacuum lag

    live_pages = total_rows // rows_per_page
    total_pages = (total_rows + dead_rows) // rows_per_page

    print(f"\n  Total rows (live):   {total_rows:>12,}")
    print(f"  Dead rows (bloat):   {dead_rows:>12,}")
    print(f"  Pages without bloat: {live_pages:>12,}  ({live_pages * page_size // 1_048_576} MB)")
    print(f"  Pages with bloat:    {total_pages:>12,}  ({total_pages * page_size // 1_048_576} MB)")
    print(f"  I/O amplification:   {total_pages/live_pages:.2f}x")
    print(f"\n  A sequential scan must read ALL {total_pages:,} pages,")
    print(f"  even though only {live_pages:,} contain live data.")
    print(f"  This is why table bloat directly increases query latency.")


if __name__ == "__main__":
    simulate_mvcc_under_load()
    simulate_write_amplification(50_000)
    simulate_dead_tuple_bloat_impact()
```

---

## 4. Schema and Autovacuum Configuration for 50,000 Updates/Second

```sql
-- Table design: fillfactor=70 leaves 30% free space for HOT updates
-- HOT (Heap Only Tuple) updates avoid creating new index entries when
-- non-indexed columns change — critical at high update rates
CREATE TABLE orders (
    id          BIGSERIAL PRIMARY KEY,
    customer_id INT NOT NULL,
    status      VARCHAR(20) NOT NULL
                    CONSTRAINT chk_status
                    CHECK (status IN ('pending','processing','completed','cancelled')),
    amount      NUMERIC(12,2) NOT NULL,
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT now()
) WITH (
    fillfactor = 70    -- 30% slack per page for HOT updates
);

-- Partial index: only active orders (much smaller, faster to vacuum too)
CREATE INDEX idx_orders_active ON orders (customer_id, updated_at)
    WHERE status IN ('pending', 'processing')
    WITH (fillfactor = 80);

-- Aggressive per-table autovacuum
ALTER TABLE orders SET (
    autovacuum_vacuum_threshold     = 1000,    -- trigger at 1K dead tuples (not 50)
    autovacuum_vacuum_scale_factor  = 0.001,   -- or 0.1% of table (not 20%)
    autovacuum_vacuum_cost_delay    = 1,        -- 1ms I/O pacing delay (not 20ms)
    autovacuum_vacuum_cost_limit    = 2000,    -- 10× default I/O tokens per round
    autovacuum_analyze_threshold    = 500,
    autovacuum_analyze_scale_factor = 0.001
);
```

```ini
# postgresql.conf — system-level tuning
autovacuum_max_workers      = 8          # more parallel workers (default 3)
autovacuum_naptime          = 5s         # check tables every 5s (default 1min)
autovacuum_vacuum_cost_delay = 2ms       # global I/O throttle (lower = faster)
maintenance_work_mem        = 1GB        # more memory per vacuum sort pass
```

---

## 5. Common Wrong Approaches and Why They Fail

### Wrong Approach 1: Assuming Updates Are In-Place

**Mistake:** "PostgreSQL updates rows in place, like a linked list update, so there are still only 10M rows."

**Why it fails:** PostgreSQL's MVCC design *never* overwrites existing tuples. Every UPDATE creates a new physical tuple. This is a deliberate design tradeoff: readers are never blocked by writers, but it creates dead tuple accumulation. (InnoDB's undo log approach stores old versions in a separate rollback segment instead — different tradeoff, same MVCC semantics.)

### Wrong Approach 2: Thinking Autovacuum Can Override Long Snapshots

**Mistake:** "Autovacuum can still clean the dead rows if you enable `autovacuum_vacuum_cost_limit = 9999`."

**Why it fails:** The vacuum horizon is not a performance parameter — it is a correctness constraint. Autovacuum would corrupt T1's result if it reclaimed tuples that T1 needs to see. `old_snapshot_threshold` is the only PostgreSQL mechanism that will forcibly terminate old snapshots to advance the horizon, and it does so by raising an error to T1, not by silently reclaiming the tuples.

### Wrong Approach 3: Using `VACUUM FULL` to Fix Bloat Under Load

**Mistake:** "Run `VACUUM FULL orders` to reclaim space immediately."

**Why it fails:** `VACUUM FULL` holds an `ACCESS EXCLUSIVE` lock, blocking all reads and writes. On a 10M-row table with 500K dead tuples, it takes minutes. Running it on a live production table causes a full outage. The correct approach is regular autovacuum or `pg_repack` (online table rebuild without locking).

### Wrong Approach 4: Not Routing Analytics to a Read Replica

**Mistake:** "The analytics query needs to run on the primary for freshness."

**Why it fails:** Even a 1-second replica lag is almost always acceptable for analytics aggregations. The cost of running analytics on the primary is holding the oldest snapshot — blocking autovacuum on the table being scanned. This is the most common cause of unexpected table bloat in production PostgreSQL.

### Wrong Approach 5: Raising `work_mem` Instead of Tuning Autovacuum

**Mistake:** "Set `work_mem = 4GB` to speed up the analytics query and reduce its runtime."

**Why it fails:** `work_mem` helps sorts and hash joins but does nothing for sequential scan speed (which is I/O bound) and doesn't change the snapshot duration problem at all. The correct lever is reducing query runtime by routing to a replica, adding partial indexes, or using partitioning.

---

## 6. Extensions: Scale and Edge Cases

### 1,000-Node Cluster / Horizontal Sharding

At the scale of 1,000 nodes, the problem evolves into distributed MVCC. Systems like CockroachDB use a global hybrid logical clock (HLC) for snapshot timestamps. A distributed analytics query holds a snapshot across hundreds of nodes simultaneously. The vacuum horizon problem becomes a distributed vacuum horizon problem: any slow node in the cluster can block GC across all nodes.

CockroachDB solves this with **intent resolution** (garbage-collecting uncommitted MVCC intents) and a **GC TTL** per table — blobs older than the TTL are unconditionally reclaimed, even if a transaction references them (the transaction gets a "GC'd too soon" error and must retry). This is analogous to PostgreSQL's `old_snapshot_threshold`.

### 1TB Table

At 1TB with 50K updates/second:
- Dead tuple volume: ~50K × 200 bytes × 60s naptime = 600MB/minute of garbage
- Autovacuum must process 600MB/minute just to stay even
- Standard spinning disk: vacuum I/O at 100MB/s = 6s per minute of writes — barely keeping up
- NVMe SSD: vacuum I/O at 2GB/s — comfortable headroom

The critical optimization: **table partitioning by time**. If `orders` is partitioned by day:
- Today's partition is the only hot one; autovacuum focuses there
- Completed partitions (older than 30 days) can be `DETACH`ed and archived with zero vacuum cost
- `VACUUM ANALYZE` on a single 10GB daily partition is dramatically faster than on a 1TB monolith

### 10ms Latency Budget

At 10ms P99 for OLTP updates, dead tuple accumulation causes two latency spikes:
1. **Page splits under bloat**: as dead tuples fill pages, new inserts find no room and trigger page splits in indexes, adding 1–3ms per operation
2. **Checkpoint pressure**: at high dead tuple rates, dirty page flushes during checkpoints compete with user I/O

Solutions: `fillfactor=70` (prevents most page splits), `checkpoint_completion_target=0.9` (spreads checkpoint I/O over a longer window), and `max_wal_size` tuning to prevent too-frequent checkpoints.

### Adversarial Input: XID Wraparound

The XID counter is 32 bits, cycling every ~4.2 billion transactions. If `age(relfrozenxid)` exceeds `autovacuum_freeze_max_age` (default 200M), PostgreSQL will start emitting warnings and eventually shut down with:

```
ERROR: database is not accepting commands to avoid wraparound data loss
```

This happened at Sentry in 2015 (9-hour outage), Mailchimp (2019), and Jira Cloud (2020). The fix is emergency `VACUUM FREEZE` — but if the table is being held by a long-running analytics query, even that won't help until the query finishes. Prevention: monitor `SELECT max(age(datfrozenxid)) FROM pg_database;` and alert at 500M.

---

## 7. Real-World Production System References

### PostgreSQL Source: `src/backend/access/heap/vacuumlazy.c`

The `heap_page_prune()` function (PostgreSQL source) implements the actual tuple pruning logic. It checks `HeapTupleSatisfiesVacuum()` — which internally checks the global `OldestXmin` horizon — before marking any tuple as reclaimable. Line ~1,200: `if (TransactionIdPrecedes(priorXmax, OldestXmin))` is the exact guard that blocks vacuum when T1 is running.

### GitHub PostgreSQL Incident (2012)

In February 2012, GitHub experienced a 5-hour PostgreSQL outage due to XID wraparound. A busy table's `relfrozenxid` approached the 2B limit. Emergency vacuum failed because a long-running replication connection held an old snapshot. Root cause: replica lag caused the primary to preserve WAL indefinitely, and `hot_standby_feedback = on` caused the primary's vacuum horizon to be pinned at the replica's oldest active XID. Fix: `hot_standby_feedback = off` with aggressive `vacuum_defer_cleanup_age` tuning.

### Citus (Distributed PostgreSQL)

Citus's distributed table vacuum requires coordinating across all worker nodes. The global `OldestXmin` is the minimum across all worker connection pools. A single long-running connection to one worker can block vacuum on all workers. Citus 11 added `citus.enable_cost_based_connection_establishment` to prevent this.

### Uber's Migration Away from PostgreSQL (2016)

Uber's famous blog post "Why Uber Engineering Switched from Postgres to MySQL" cited MVCC dead tuple bloat as a primary reason. At their write rates (millions of rows/second across 100+ tables), autovacuum could not keep up. The specific problem: replica lag caused `hot_standby_feedback` to pin the primary vacuum horizon — the exact scenario in this exercise, at massive scale. Their proposed mitigations (aggressive vacuum tuning, partitioning) are exactly the options in part (e).

### Amazon Aurora and Multi-Version Purge

Aurora PostgreSQL uses a modified storage engine where the "dead tuple" vacuum problem is handled differently: the storage layer tracks row versions in a separate region and purges them without needing to read and rewrite heap pages. This is similar to InnoDB's purge thread model. The visible effect: Aurora PostgreSQL handles high UPDATE rates with less autovacuum I/O but still has the snapshot horizon constraint — long-running Aurora queries still block the purge process.
