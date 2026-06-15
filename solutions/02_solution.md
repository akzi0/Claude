# Solution: MVCC Under Load (Hard Exercise — Lesson 02)

## The Question

A PostgreSQL table `orders` has **10 million rows**. A long-running analytics query `T1` (`SELECT sum(amount) FROM orders WHERE status = 'completed'`) started **2 minutes ago** and is still running.

Since `T1` started, OLTP transactions have updated **500,000 rows**.

**Questions:**
- (a) How many row versions exist in the heap?
- (b) Are the dead rows from updates visible to T1?
- (c) Can autovacuum clean up the dead rows while T1 is running?
- (d) What setting controls how long autovacuum waits?
- (e) Provide schema + autovacuum config for 50,000 updates/second.

---

## (a) Row Versions in the Heap

**Answer: ~10,500,000 row versions.**

PostgreSQL MVCC never overwrites rows in-place. Every `UPDATE` creates a new row version (new tuple) and marks the old version as deleted by setting its `xmax` to the committing transaction's XID.

At the time of measurement:
- **9,500,000** rows that were never updated: exist as single live versions.
- **500,000** rows that were updated: each has **two** versions:
  - The **old version** (dead to transactions that started after the update, but still physically present).
  - The **new version** (live to transactions that started after the update).

Total physical tuples = 9,500,000 × 1 + 500,000 × 2 = **10,500,000**.

**Why the old versions are still present:** PostgreSQL cannot remove them yet — T1's snapshot still "sees" them as the authoritative version of those rows. Removing them would corrupt T1's query result.

**In reality it can be more:** If rows were updated multiple times in the 2-minute window (e.g., a row updated 3 times = 3 versions in the heap), the total could exceed 10.5M. The 500K figure is likely a count of distinct rows affected, not total update operations.

---

## (b) Are Dead Rows Visible to T1?

**Answer: Yes — T1 sees the OLD versions (which are "dead" to other transactions) as its live data.**

Here is the precise MVCC visibility logic:

When T1 started, PostgreSQL took a **snapshot**:
```
snapshot = {
    xmin:  <smallest active XID at T1's start>,
    xmax:  <next XID to be assigned at T1's start>,
    xip:   [list of active XIDs at T1's start]
}
```

A tuple is visible to T1 if:
1. `xmin` (the XID that inserted the tuple) is committed and `xmin < T1.xmax` and `xmin ∉ T1.xip`, AND
2. Either `xmax = 0` (not deleted), or `xmax` is not committed, or `xmax >= T1.xmax`, or `xmax ∈ T1.xip`.

The 500K updates all occurred **after** T1 started. Their `xmin` (for new versions) is **greater** than T1's `xmax` — so new versions are **invisible** to T1.

For old versions: their `xmax` is the XID of the OLTP transaction that performed the update — committed **after** T1's snapshot was taken. Therefore `xmax >= T1.xmax`, which means the deletion is invisible to T1.

**Result:** T1 reads the **old versions** of all 500,000 updated rows — exactly the pre-update state. From T1's perspective, those old tuples are "live". From other transactions' perspective, they are "dead" (xmax is committed). This is the core of snapshot isolation: different transactions can see different live sets simultaneously.

```
              T1 snapshot taken
                    │
Time ──────────────┼──────────────────────────────► now
                    │         │
                    │    OLTP updates 500K rows
                    │    (these occur AFTER T1's snapshot)
                    │
                    ▼
T1 sees: old versions (xmax > T1.xmax → deletion invisible to T1)
Others:  new versions (xmin < others' xmax, and old xmax is committed)
```

---

## (c) Can Autovacuum Clean Up Dead Rows While T1 Runs?

**Answer: No.** Autovacuum cannot remove dead tuples that are still visible to any active transaction's snapshot.

PostgreSQL tracks the **oldest active snapshot** (`pg_stat_activity.backend_xmin` or the system-wide `pg_stat_activity` minimum). Autovacuum uses this as its "horizon" — it will not remove any tuple with `xmax >= horizon`, because doing so could make a live row disappear from an active transaction's view.

Since T1 started 2 minutes ago, T1's `xmin` is 2 minutes old. The 500K updated rows have old versions with `xmax` values that are **greater than T1's xmin** (they were updated after T1 started). But from the vacuum's perspective, it checks: "Is this dead tuple still visible to the oldest snapshot?" Since T1's snapshot can see these old versions, the answer is yes, and vacuum skips them.

**The mechanism:**
```sql
-- Check what vacuum's horizon currently is:
SELECT
    datname,
    age(datfrozenxid) as db_age,
    (SELECT min(backend_xmin) FROM pg_stat_activity WHERE backend_xmin IS NOT NULL) 
        AS oldest_active_snapshot_xmin
FROM pg_database WHERE datname = current_database();

-- See which tables have bloat:
SELECT
    relname,
    n_live_tup,
    n_dead_tup,
    round(n_dead_tup::numeric / nullif(n_live_tup + n_dead_tup, 0) * 100, 2) AS dead_pct
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
```

Autovacuum will instead work on **other tables** (or rows that became dead before T1's snapshot), but it **cannot touch** the 500K rows that T1 can still see.

**Once T1 completes:** The oldest active snapshot advances, the horizon moves forward, and autovacuum can immediately reclaim those 500K dead tuples on its next run.

---

## (d) What Setting Controls This?

**Primary control: `old_snapshot_threshold` (PostgreSQL 9.6+)**

- **Default: `-1`** (disabled — vacuum never ignores old snapshots).
- When set (e.g., `old_snapshot_threshold = '30min'`), PostgreSQL may vacuum dead tuples older than the threshold even if an old snapshot could see them.
- **Trade-off:** Transactions with snapshots older than the threshold will receive `ERROR: snapshot too old` if they try to read a vacuumed tuple. This is an intentional design: force long queries onto a read replica, or terminate them, rather than letting them block vacuum indefinitely.

**Secondary controls:**
- `idle_in_transaction_session_timeout`: Kills sessions that are idle inside a transaction (holding a snapshot without doing work). Set to `'10min'` to prevent accidental snapshot-holding from idle connections.
- `statement_timeout`: Maximum time for any single statement. Prevents runaway analytics queries from running indefinitely.
- `lock_timeout`: Maximum time to wait for a lock (doesn't help here directly, but prevents lock contention cascades).

**The real solution:** Route analytics queries to a **streaming read replica**. The replica's vacuum operates independently of the primary's snapshot horizon. This is the architecturally correct fix.

---

## (e) Schema + Autovacuum Config for 50,000 Updates/Second

At 50,000 updates/sec, the table accumulates **3 million dead tuples per minute**. Default autovacuum (scale factor 20%, cost delay 20ms) cannot keep up. Here's the full production setup:

### Schema Design

```sql
-- Partition by time to isolate vacuum to smaller, bounded tables
-- (Each partition is independently vacuumed; old complete partitions can be DROPped instantly)
CREATE TABLE orders (
    id          BIGSERIAL NOT NULL,
    customer_id INT NOT NULL,
    status      VARCHAR(20) NOT NULL,
    amount      NUMERIC(12,2) NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (updated_at);

-- Create partitions (automate with pg_partman in production)
CREATE TABLE orders_2024_01 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01')
    WITH (fillfactor = 70);  -- 30% free space for HOT updates

CREATE TABLE orders_2024_02 PARTITION OF orders
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01')
    WITH (fillfactor = 70);

-- Index: hot columns only; avoid indexing frequently-updated columns
CREATE INDEX CONCURRENTLY idx_orders_customer_status
    ON orders (customer_id, status, updated_at)
    WHERE status IN ('pending', 'processing')  -- partial: exclude terminal states
    WITH (fillfactor = 80);

-- Covering index for the analytics query (avoids heap fetch entirely)
CREATE INDEX CONCURRENTLY idx_orders_analytics
    ON orders (status, amount)
    INCLUDE (id)
    WITH (fillfactor = 90);
```

**Why `fillfactor = 70`:** HOT (Heap Only Tuple) updates avoid creating a new index entry when only non-indexed columns change. HOT requires free space on the same page. With `fillfactor=70`, each page has 30% free space → HOT updates can chain within the same page → no index bloat, much faster autovacuum.

### Table-Level Autovacuum Configuration

```sql
ALTER TABLE orders SET (
    -- Trigger vacuum after just 0.1% dead tuples (vs 20% default)
    autovacuum_vacuum_scale_factor = 0.001,
    -- Also trigger after absolute threshold of 500 dead tuples (low for fast trigger)
    autovacuum_vacuum_threshold = 500,
    -- Run analyze after 0.05% changes (keep stats fresh for planner)
    autovacuum_analyze_scale_factor = 0.0005,
    autovacuum_analyze_threshold = 200,
    -- Minimal IO throttling: 2ms delay (vs 20ms default), 4000 cost/round (vs 200)
    autovacuum_vacuum_cost_delay = 2,
    autovacuum_vacuum_cost_limit = 4000,
    -- Freeze aggressively to avoid wraparound emergency
    autovacuum_freeze_max_age = 50000000
);
```

### System-Level postgresql.conf

```ini
# More parallel autovacuum workers (default 3)
autovacuum_max_workers = 6

# Check more often (default 60s)
autovacuum_naptime = 5s

# Faster global vacuum
autovacuum_vacuum_cost_delay = 2ms
autovacuum_vacuum_cost_limit = 2000

# More memory for vacuum sort (critical for large tables)
maintenance_work_mem = 2GB

# Prevent runaway transactions from holding old snapshots
idle_in_transaction_session_timeout = '5min'
statement_timeout = '30min'

# Safety valve: allow vacuum to work past old snapshots (with error risk)
# old_snapshot_threshold = '60min'  # enable cautiously
```

### Diagnostic SQL to Monitor Bloat

```python
"""Diagnostic queries for MVCC bloat and autovacuum health."""

BLOAT_QUERY = """
SELECT
    schemaname,
    relname AS table_name,
    n_live_tup,
    n_dead_tup,
    round(n_dead_tup::numeric / NULLIF(n_live_tup + n_dead_tup, 0) * 100, 2) AS dead_pct,
    last_vacuum,
    last_autovacuum,
    vacuum_count,
    autovacuum_count,
    pg_size_pretty(pg_total_relation_size(relid)) AS total_size
FROM pg_stat_user_tables
WHERE n_live_tup > 0
ORDER BY n_dead_tup DESC
LIMIT 20;
"""

SNAPSHOT_AGE_QUERY = """
SELECT
    pid,
    usename,
    application_name,
    state,
    backend_xmin,
    age(backend_xmin) AS xmin_age,   -- how many transactions old is this snapshot?
    now() - xact_start AS xact_duration,
    left(query, 80) AS query_preview
FROM pg_stat_activity
WHERE backend_xmin IS NOT NULL
ORDER BY age(backend_xmin) DESC;     -- worst offender first
"""

AUTOVACUUM_ACTIVITY_QUERY = """
SELECT
    pid,
    now() - pg_stat_activity.query_start AS duration,
    query,
    state
FROM pg_stat_activity
WHERE query LIKE 'autovacuum:%'
ORDER BY duration DESC;
"""

TABLE_BLOAT_ESTIMATE_QUERY = """
-- Estimated bloat from pgstattuple (requires extension)
-- SELECT * FROM pgstattuple('orders');

-- Alternative: use n_dead_tup with avg tuple size
SELECT
    relname,
    n_dead_tup,
    n_live_tup,
    round(n_dead_tup::numeric / NULLIF(n_live_tup, 0) * 100, 1) AS dead_pct,
    pg_size_pretty(
        n_dead_tup * (SELECT avg_width FROM pg_stats WHERE tablename = relname LIMIT 1)::bigint
    ) AS estimated_dead_bytes
FROM pg_stat_user_tables
WHERE relname = 'orders';
"""

import psycopg2
import os

def run_diagnostics(dsn: str):
    """Run all diagnostic queries and print a health report."""
    with psycopg2.connect(dsn) as conn:
        with conn.cursor() as cur:
            print("=== TABLE BLOAT ===")
            cur.execute(BLOAT_QUERY)
            rows = cur.fetchall()
            print(f"{'Table':<30} {'Live':>10} {'Dead':>10} {'Dead%':>7} {'Last AV':<20}")
            for r in rows:
                last_av = str(r[6])[:19] if r[6] else "never"
                print(f"{r[1]:<30} {r[2]:>10,} {r[3]:>10,} {r[4]:>6}% {last_av:<20}")

            print("\n=== OLDEST ACTIVE SNAPSHOTS ===")
            cur.execute(SNAPSHOT_AGE_QUERY)
            rows = cur.fetchall()
            for r in rows:
                print(f"PID={r[0]} user={r[1]} xmin_age={r[5]} duration={r[6]} query={r[7]}")

# Usage: run_diagnostics("host=localhost dbname=mydb user=postgres")
```

### The Feedback Loop to Break

```
50K updates/sec
    → 3M dead tuples/min
    → autovacuum must process 3M/min to keep up
    → at default cost settings: vacuum processes ~500K tuples/min (too slow)
    → bloat grows → table scan reads more pages → analytics query slows
    → slower analytics query holds snapshot longer → vacuum blocked longer
    → cycle accelerates
```

**Breaking points:**
1. **Partitioning:** Vacuum operates on partitions independently. A daily partition with 4.3B rows is hopeless; a 1-hour partition with 180M rows vacuums cleanly.
2. **Read replicas:** Route T1 to a replica so primary's vacuum horizon is not held back by analytics.
3. **Aggressive autovacuum settings:** Per-table overrides above cut vacuum latency from ~60s to ~5s.
4. **fillfactor + HOT:** Reduces index-bloat writes by ~40% under typical OLTP patterns.

---

## Why This Matters

This is the most common cause of PostgreSQL performance degradation in high-throughput OLTP systems. The pattern:
1. Someone runs an analytics query on the primary (instead of a replica).
2. The query takes 10+ minutes.
3. Autovacuum cannot clean dead tuples from the 10 minutes of updates.
4. Table bloat causes queries to slow down further.
5. Eventually, `VACUUM FREEZE` must run to prevent XID wraparound, blocking all writes.

The solution architecture is: **analytics on replicas, aggressive per-table autovacuum config, time-based partitioning, and snapshot horizon monitoring**.
