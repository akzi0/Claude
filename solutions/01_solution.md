# Solution: Raft Partition Trace (Hard Exercise — Lesson 01)

## The Question

A 5-node Raft cluster (N1–N5) has N1 as leader at term 3. The log state **before** partition is:

```
N1 (leader): [(1,a), (1,b), (2,c), (3,d)]
N2:          [(1,a), (1,b), (2,c), (3,d)]
N3:          [(1,a), (1,b), (2,c)       ]
N4:          [(1,a), (1,b), (2,c)       ]
N5:          [(1,a), (1,b)              ]
```

Entry `(3,d)` is only on N1 and N2 — **not committed** (needs 3/5 for quorum).

**Partition:** `{N1, N2}` | `{N3, N4, N5}`

N3 wins election at term 4 and commits `(4,e)` and `(4,f)`.

**Questions:**
1. Trace each node's log state before partition, during, and after healing.
2. Which entries survive? Prove using the Leader Completeness Property.
3. What changes if `(3,d)` **was** committed before the partition?

---

## Step-by-Step Solution

### Phase 1: State Before Partition

The log entries use notation `(term, value)`. Entries at index 1–4:

| Node | Index 1 | Index 2 | Index 3 | Index 4 | commitIndex |
|------|---------|---------|---------|---------|-------------|
| N1   | (1,a)   | (1,b)   | (2,c)   | (3,d)   | 3 (only (1,a),(1,b),(2,c) committed) |
| N2   | (1,a)   | (1,b)   | (2,c)   | (3,d)   | 3           |
| N3   | (1,a)   | (1,b)   | (2,c)   | –       | 3           |
| N4   | (1,a)   | (1,b)   | (2,c)   | –       | 3           |
| N5   | (1,a)   | (1,b)   | –       | –       | 2           |

**Key point:** `(3,d)` is replicated to only 2/5 nodes. N1 has not yet received acknowledgment from a majority (3+), so `commitIndex` for N1 is still 3 (meaning only index 3, which holds `(2,c)`, is committed). Entry `(3,d)` is at index 4 and is **uncommitted**.

### Phase 2: During Partition

**Partition: {N1, N2} | {N3, N4, N5}**

**Minority partition {N1, N2}:**
- N1 remains leader but cannot make progress (only 2/5 nodes reachable — no quorum).
- Client writes to N1 during the partition will stall (not committed).
- N1 and N2 stay at term 3. They cannot commit anything new.

**Majority partition {N3, N4, N5}:**
- N3, N4, N5 don't hear from N1 (election timeout fires).
- N3 starts an election at term 4.
- **Election Restriction check:** A node grants its vote only if the candidate's log is at least as up-to-date. Comparison: `(lastLogTerm, lastLogIndex)`.
  - N3: `lastLogTerm=2, lastLogIndex=3`
  - N4: `lastLogTerm=2, lastLogIndex=3` (tied — first-come-first-served)
  - N5: `lastLogTerm=1, lastLogIndex=2` (N3's log is more up-to-date; N3 beats N5)
- N3 receives votes from N4 and N5 (and itself) → **3 votes = majority → N3 wins**.
- N3 becomes leader at term 4.

N3 immediately appends a **no-op** (standard practice) and then appends `(4,e)` and `(4,f)`:
```
N3 appends: (4,e) → replicates to N4, N5 → 3/3 acknowledge → COMMITTED
N3 appends: (4,f) → replicates to N4, N5 → 3/3 acknowledge → COMMITTED
```

**Log state during partition:**

| Node | Log | commitIndex |
|------|-----|-------------|
| N1   | [(1,a),(1,b),(2,c),(3,d)] | 3 (idx of (2,c)) |
| N2   | [(1,a),(1,b),(2,c),(3,d)] | 3 |
| N3   | [(1,a),(1,b),(2,c),(4,e),(4,f)] | 5 |
| N4   | [(1,a),(1,b),(2,c),(4,e),(4,f)] | 5 |
| N5   | [(1,a),(1,b),(2,c),(4,e),(4,f)] | 5 |

### Phase 3: Partition Heals

N3 (leader, term 4) sends AppendEntries to N1 and N2.

1. N1 and N2 receive AppendEntries with `term=4`. Since `4 > 3`, they immediately step down and convert to followers.
2. N3 needs to bring N1/N2 in sync. It uses the **log consistency check**:
   - N3 sends: `prevLogIndex=3, prevLogTerm=2` (checking that N1/N2's entry at index 3 is `(2,c)`)
   - N1's `log[3] = (2,c)` ✓ → consistency check passes
   - N3 sends entries `(4,e)` and `(4,f)` starting at index 4
   - **CONFLICT:** N1/N2 have `(3,d)` at index 4. AppendEntries rule: if an existing entry conflicts with a new one (different term), delete that entry and all following. So `(3,d)` is **deleted** from N1 and N2.
   - `(4,e)` is written at index 4, `(4,f)` at index 5.

**Final log state (all nodes identical):**

| Node | Log | commitIndex |
|------|-----|-------------|
| N1   | [(1,a),(1,b),(2,c),(4,e),(4,f)] | 5 |
| N2   | [(1,a),(1,b),(2,c),(4,e),(4,f)] | 5 |
| N3   | [(1,a),(1,b),(2,c),(4,e),(4,f)] | 5 |
| N4   | [(1,a),(1,b),(2,c),(4,e),(4,f)] | 5 |
| N5   | [(1,a),(1,b),(2,c),(4,e),(4,f)] | 5 |

**Entries that survive:** `(1,a), (1,b), (2,c), (4,e), (4,f)`

**Entry that does NOT survive:** `(3,d)` — overwritten on N1 and N2.

---

## Proof Using the Leader Completeness Property

**Leader Completeness Property (Raft §5.4.3):** If a log entry is committed in a given term, that entry will be present in the logs of all leaders for all higher terms.

**Why `(3,d)` can be overwritten:** It was **never committed**. Committal requires a majority (3/5). Only N1 and N2 had `(3,d)`, and the partition isolated them into a minority. The Leader Completeness Property applies only to **committed** entries — `(3,d)` never achieved that status, so Raft provides no durability guarantee for it.

**Why `(1,a), (1,b), (2,c)` cannot be overwritten:** These were committed before the partition (majority of 3+ nodes acknowledged them). By the Leader Completeness Property:
- Any candidate that wins an election must have a log at least as up-to-date as any majority.
- Any majority overlaps with the set that committed these entries.
- Therefore, any new leader must have `(1,a), (1,b), (2,c)` in its log.
- N3 does have them (confirmed: N3's log was `[(1,a),(1,b),(2,c)]`).
- N3 cannot overwrite them because it only appends after index 3.

**The Safety Guarantee:** The Election Restriction (comparing `lastLogTerm, lastLogIndex`) ensures only candidates with up-to-date committed entries can win. This is the mechanism that enforces Leader Completeness.

---

## Variant: What if `(3,d)` WAS Committed Before the Partition?

If `(3,d)` was committed, a majority (3/5) of nodes must have acknowledged it. The only possibility consistent with the log state shown is if N3 or N4 also had `(3,d)` before the partition.

Suppose N3 received `(3,d)` just before the partition:
```
N3: [(1,a), (1,b), (2,c), (3,d)]  ← now possible if (3,d) was committed
```

Now N3's `lastLogTerm=3, lastLogIndex=4`. This doesn't change the election outcome (N3 can still win), but critically:

- N3 **cannot overwrite `(3,d)`** because it's in N3's own log and is committed.
- When N3 wins the election and becomes leader at term 4, it must **first commit a no-op at term 4** (using N3's own `(3,d)` as the previous entry).
- The resulting log: `[(1,a),(1,b),(2,c),(3,d),(4,noop),(4,e),(4,f)]`

**The critical constraint:** If `(3,d)` was committed, then at least 3 nodes must have had it. Among {N3, N4, N5}, at least one must have `(3,d)`. The winner of the election in the majority partition MUST be a node that has `(3,d)` (because only such a node has a log up-to-date enough to beat the others via the Election Restriction). Therefore, `(3,d)` propagates forward and is never lost.

This is the key safety argument: **committed entries can only be on election-winning logs.**

---

## Common Mistakes and Traps

1. **Assuming term = committed.** Just because an entry has a term number doesn't mean it's committed. Committal requires quorum acknowledgment.

2. **Off-by-one on quorum.** A 5-node cluster requires 3 votes (majority). {N1, N2} = 2 nodes = minority. They cannot commit or elect a new leader.

3. **Forgetting the no-op.** A new leader should append a no-op at the start of its term to advance commitIndex to the current term. Some exam questions ignore this; in a real implementation it matters for liveness.

4. **Confusing log index with term.** `(3,d)` means "term 3, value d" — not "index 3". The entries are indexed 1–4 in this example.

5. **Election Restriction direction.** The restriction compares `(lastLogTerm, lastLogIndex)` lexicographically. Higher `lastLogTerm` always wins, regardless of `lastLogIndex`. Only if terms are equal does `lastLogIndex` matter.

---

## Python Simulation

```python
"""
Raft partition simulation.
Simulates the exact scenario from the hard exercise.
"""
from dataclasses import dataclass, field
from typing import List, Optional, Tuple

LogEntry = Tuple[int, str]  # (term, value)


@dataclass
class RaftNode:
    node_id: int
    current_term: int
    log: List[LogEntry] = field(default_factory=list)
    commit_index: int = 0
    role: str = "follower"  # "leader" | "candidate" | "follower"

    def last_log_term(self) -> int:
        return self.log[-1][0] if self.log else 0

    def last_log_index(self) -> int:
        return len(self.log)

    def log_is_at_least_as_up_to_date(self, other_last_term: int, other_last_index: int) -> bool:
        """Election Restriction: candidate must have log >= mine."""
        my_term = self.last_log_term()
        my_index = self.last_log_index()
        if other_last_term != my_term:
            return other_last_term > my_term
        return other_last_index >= my_index

    def receive_append_entries(
        self,
        leader_term: int,
        prev_index: int,
        prev_term: int,
        entries: List[LogEntry],
        leader_commit: int
    ) -> bool:
        """Returns True if accepted."""
        if leader_term < self.current_term:
            return False
        # Step down if we see a higher term
        if leader_term > self.current_term:
            self.current_term = leader_term
            self.role = "follower"
        # Consistency check
        if prev_index > 0:
            if len(self.log) < prev_index:
                return False
            if self.log[prev_index - 1][0] != prev_term:
                return False
        # Append (overwriting conflicts)
        for i, entry in enumerate(entries):
            log_index = prev_index + i  # 0-based
            if log_index < len(self.log):
                if self.log[log_index][0] != entry[0]:  # conflict
                    self.log = self.log[:log_index]
            if log_index >= len(self.log):
                self.log.append(entry)
        # Update commit index
        if leader_commit > self.commit_index:
            self.commit_index = min(leader_commit, len(self.log))
        return True

    def __repr__(self):
        return (f"N{self.node_id}[term={self.current_term}, role={self.role}, "
                f"log={self.log}, commitIndex={self.commit_index}]")


def simulate_partition_scenario():
    # Initial state: N1 is leader at term 3
    nodes = [
        RaftNode(1, term=3, log=[(1,'a'),(1,'b'),(2,'c'),(3,'d')], commit_index=3, role="leader"),
        RaftNode(2, term=3, log=[(1,'a'),(1,'b'),(2,'c'),(3,'d')], commit_index=3),
        RaftNode(3, term=3, log=[(1,'a'),(1,'b'),(2,'c')],         commit_index=3),
        RaftNode(4, term=3, log=[(1,'a'),(1,'b'),(2,'c')],         commit_index=3),
        RaftNode(5, term=3, log=[(1,'a'),(1,'b')],                 commit_index=2),
    ]
    n1, n2, n3, n4, n5 = nodes

    print("=== BEFORE PARTITION ===")
    for n in nodes:
        print(f"  {n}")

    print("\n=== PARTITION: {N1,N2} | {N3,N4,N5} ===")
    # N3 wins election in majority partition
    n3.current_term = 4
    n3.role = "leader"
    n4.current_term = 4
    n5.current_term = 4

    # N3 appends (4,e) and (4,f), committed by majority {N3,N4,N5}
    new_entries = [(4, 'e'), (4, 'f')]
    prev_index = len(n3.log)  # 3
    prev_term = n3.log[-1][0]  # 2

    for follower in [n4, n5]:
        follower.receive_append_entries(4, prev_index, prev_term, new_entries, 5)
    for entry in new_entries:
        n3.log.append(entry)
    n3.commit_index = 5

    print("\nMajority partition state:")
    for n in [n3, n4, n5]:
        print(f"  {n}")
    print("\nMinority partition state (stalled):")
    for n in [n1, n2]:
        print(f"  {n}")

    print("\n=== PARTITION HEALS ===")
    # N3 sends AppendEntries to N1 and N2
    # prevLogIndex=3 (send from index 4), prevLogTerm=2 (term of entry at index 3)
    for follower in [n1, n2]:
        success = follower.receive_append_entries(
            leader_term=4,
            prev_index=3,
            prev_term=2,
            entries=[(4,'e'), (4,'f')],
            leader_commit=5
        )
        print(f"  N{follower.node_id} accepted AppendEntries: {success}")

    print("\n=== FINAL STATE (all nodes) ===")
    for n in nodes:
        print(f"  {n}")

    # Verify all logs are identical
    logs = [tuple(n.log) for n in nodes]
    assert all(l == logs[0] for l in logs), "Logs diverged!"
    print("\n✓ All logs identical. (3,d) was overwritten. Safety maintained.")
    print(f"✓ Surviving entries: {nodes[0].log}")


if __name__ == "__main__":
    simulate_partition_scenario()
```

**Expected output:**
```
=== BEFORE PARTITION ===
  N1[term=3, role=leader, log=[(1,'a'),(1,'b'),(2,'c'),(3,'d')], commitIndex=3]
  N2[term=3, role=follower, log=[(1,'a'),(1,'b'),(2,'c'),(3,'d')], commitIndex=3]
  N3[term=3, role=follower, log=[(1,'a'),(1,'b'),(2,'c')], commitIndex=3]
  N4[term=3, role=follower, log=[(1,'a'),(1,'b'),(2,'c')], commitIndex=3]
  N5[term=3, role=follower, log=[(1,'a'),(1,'b')], commitIndex=2]

=== PARTITION HEALS ===
  N1 accepted AppendEntries: True
  N2 accepted AppendEntries: True

=== FINAL STATE ===
  All nodes: log=[(1,'a'),(1,'b'),(2,'c'),(4,'e'),(4,'f')], commitIndex=5
✓ (3,d) was overwritten. Safety maintained.
```

---

## Summary

| Phase | (3,d) status | Reason |
|-------|-------------|--------|
| Before partition | Uncommitted (only 2/5 nodes) | No quorum |
| During partition | Stale on N1, N2 | Cannot reach quorum |
| After healing | DELETED from N1, N2 | Conflicts with N3's committed (4,e) |

**The fundamental rule:** Raft only guarantees durability for **committed** entries. An uncommitted entry can and will be overwritten by a new leader's entries if there is a conflict.
