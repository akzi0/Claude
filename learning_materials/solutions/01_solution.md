# Solution: Lesson 01 — Distributed Consensus
## Hard Exercise: Trace Through a Complex Partition

---

## 1. Full Exercise Restatement

**Setup:** 5-node Raft cluster (N1–N5). N1 is leader at term 3. Initial log state:

```
N1 (leader): [(1,a), (1,b), (2,c), (3,d)]
N2:          [(1,a), (1,b), (2,c), (3,d)]
N3:          [(1,a), (1,b), (2,c)       ]
N4:          [(1,a), (1,b), (2,c)       ]
N5:          [(1,a), (1,b)              ]
```

Entry `(3,d)` is on N1 and N2 only — not yet committed (only 2/5 nodes, not a majority).

**Phase 1 — Partition:** `{N1, N2} | {N3, N4, N5}`

N3 wins election at term 4 (majority partition). N3 appends and commits `(4,e)` and `(4,f)`. N3/N4/N5 end up with log `[(1,a), (1,b), (2,c), (4,e), (4,f)]`, commitIndex = 4.

**Phase 2 — Heal:** Partition heals. N3 sends AppendEntries to N1 and N2.

**Questions:**
1. Which entries survive? Which are overwritten?
2. Is this safe? Does it violate the Agreement property?
3. Why does Leader Completeness guarantee `(4,e)` and `(4,f)` can never be overwritten?
4. Variant: what if `(3,d)` had been committed before the partition?

---

## 2. Conceptual Solution Walkthrough

### Step 1: Election Restriction Analysis

When the partition forms, the minority partition `{N1, N2}` cannot elect a new leader — they have only 2 out of 5 nodes, which does not form a quorum. N1 remains leader until its timeouts cause it to step down, at which point it stalls (no quorum to commit anything).

In the majority partition `{N3, N4, N5}`, an election occurs for term 4. The **Election Restriction** requires that a candidate can only be elected if its log is "at least as up-to-date" as any voter's log. Concretely:

> Node A's log is more up-to-date than B's if: `lastLogTerm_A > lastLogTerm_B`, or if terms are equal, `lastLogIndex_A > lastLogIndex_B`.

In `{N3, N4, N5}`:
- N3: `lastLogTerm=2, lastLogIndex=2` (entries up to `(2,c)`)
- N4: same as N3
- N5: `lastLogTerm=1, lastLogIndex=1` (only `(1,a)`, `(1,b)`)

N3 beats N5 because `lastLogTerm=2 > 1`. N3 and N4 are tied; the first to time out wins. N3 is elected for term 4.

### Step 2: N3 Commits (4,e) and (4,f)

N3 successfully replicates to N4 and N5 (quorum of 3). Both entries are committed:

```
N3 (leader, term 4): [(1,a), (1,b), (2,c), (4,e), (4,f)]  commitIndex=4
N4:                  [(1,a), (1,b), (2,c), (4,e), (4,f)]  commitIndex=4
N5:                  [(1,a), (1,b), (2,c), (4,e), (4,f)]  commitIndex=4
```

### Step 3: Partition Heals — Log Conflict Resolution

N3 sends AppendEntries to N1 and N2. Both receive a message with `term=4 > 3`, so they immediately step down from leader/follower at term 3 and update their term to 4.

N3's AppendEntries specifies `prevLogIndex=2, prevLogTerm=2` (the entry just before what N3 wants to send). N1 and N2 check: `log[2] = (2,c)` — match. They accept the new entries.

**Critical:** N1 has `log[3] = (3,d)` — a conflict with the incoming `log[3] = (4,e)`. Raft's log reconciliation rule is:

> If an existing entry conflicts with a new one (same index, different term), **delete the existing entry and all that follow it**, then append the new entries.

So N1 and N2 **delete** `(3,d)` at index 3 and replace it with `(4,e)`, `(4,f)`.

### Step 4: Final State

All 5 nodes converge to:

```
N1: [(1,a), (1,b), (2,c), (4,e), (4,f)]  commitIndex=4
N2: [(1,a), (1,b), (2,c), (4,e), (4,f)]  commitIndex=4
N3: [(1,a), (1,b), (2,c), (4,e), (4,f)]  commitIndex=4
N4: [(1,a), (1,b), (2,c), (4,e), (4,f)]  commitIndex=4
N5: [(1,a), (1,b), (2,c), (4,e), (4,f)]  commitIndex=4
```

`(3,d)` does **not** survive. This is **safe**: `(3,d)` was never committed — N1 never sent a success response to any client for that entry. No client ever observed it as durable.

### Step 5: Why Leader Completeness Guarantees (4,e) and (4,f)

**Leader Completeness Theorem (Raft):** If an entry is committed in term T, then every leader elected in term T' > T contains that entry in its log.

**Proof sketch for (4,e):** It was committed at term 4 with quorum `{N3, N4, N5}`. Any future leader must win votes from a quorum. Any quorum overlaps with `{N3, N4, N5}` in at least one node. That node's `lastLogTerm = 4`. For the candidate to win against this node, the candidate's log must be at least as up-to-date: either `lastLogTerm > 4` (meaning it already has even newer committed entries that include all term-4 entries by induction) or `lastLogTerm = 4` with `lastLogIndex ≥ 4` (meaning its log has at least 4 entries through term 4, all of which must include `(4,e)` and `(4,f)` by Log Matching). Either way, the future leader has both entries. ∎

### Step 6: The Committed Variant

If `(3,d)` had been committed before the partition, N1 must have replicated it to a quorum — at least one node in `{N3, N4, N5}` has it (since any quorum of 5 overlaps with any other quorum). Say N3 and N4 have `[(1,a), (1,b), (2,c), (3,d)]`.

Now the Election Restriction changes: N3's `lastLogTerm=3` beats N5's `lastLogTerm=1`. N3 wins the election. N3 already has `(3,d)`. N3 appends `(4,e)`, `(4,f)` after it:

```
N3 (leader, term 4): [(1,a), (1,b), (2,c), (3,d), (4,e), (4,f)]
```

When N1 and N2 reconnect, N3's consistency check passes immediately (N1/N2 already have entries through index 3 with matching terms). They simply append the two new entries. `(3,d)` is **preserved**. Leader Completeness worked as intended.

---

## 3. Full Python Simulation

```python
"""
Raft partition scenario simulation.

This code simulates the exact partition/heal scenario from the exercise
and verifies that invariants hold throughout. It is self-contained and
requires only the Python standard library.
"""

from __future__ import annotations
import threading
import time
import random
from dataclasses import dataclass, field
from typing import List, Optional, Dict, Tuple
from enum import Enum


# ── Data structures ────────────────────────────────────────────────────────────

@dataclass(frozen=True)
class LogEntry:
    term: int
    command: str

    def __repr__(self):
        return f"({self.term},{self.command})"


class NodeState(Enum):
    FOLLOWER  = "follower"
    CANDIDATE = "candidate"
    LEADER    = "leader"


# ── Simplified Raft node for simulation ───────────────────────────────────────

@dataclass
class RaftNode:
    node_id: int
    current_term: int = 0
    voted_for: Optional[int] = None
    log: List[LogEntry] = field(default_factory=list)
    commit_index: int = -1  # index of last committed entry (-1 = none)
    state: NodeState = NodeState.FOLLOWER

    def last_log_index(self) -> int:
        return len(self.log) - 1

    def last_log_term(self) -> int:
        return self.log[-1].term if self.log else 0

    def is_up_to_date(self, other_last_term: int, other_last_index: int) -> bool:
        """Election Restriction: return True if other's log is >= ours."""
        if other_last_term != self.last_log_term():
            return other_last_term > self.last_log_term()
        return other_last_index >= self.last_log_index()

    def append_entries(
        self,
        leader_term: int,
        prev_index: int,
        prev_term: int,
        entries: List[LogEntry],
        leader_commit: int,
    ) -> bool:
        """
        Process AppendEntries RPC.
        Returns True on success, False on failure (consistency check failed).
        """
        # Rule 1: reject stale leaders
        if leader_term < self.current_term:
            return False

        # Step down if we see a higher term
        if leader_term > self.current_term:
            self.current_term = leader_term
            self.state = NodeState.FOLLOWER
            self.voted_for = None

        # Consistency check: does our log contain the expected prev entry?
        if prev_index >= 0:
            if prev_index >= len(self.log):
                return False  # we don't have that entry yet
            if self.log[prev_index].term != prev_term:
                return False  # term mismatch

        # Delete conflicting entries and append new ones
        insert_pos = prev_index + 1
        for i, entry in enumerate(entries):
            pos = insert_pos + i
            if pos < len(self.log):
                if self.log[pos].term != entry.term:
                    # Conflict: truncate from here
                    self.log = self.log[:pos]
                    self.log.append(entry)
                # else: identical entry, skip
            else:
                self.log.append(entry)

        # Advance commit index
        if leader_commit > self.commit_index:
            self.commit_index = min(leader_commit, len(self.log) - 1)

        return True

    def request_vote(
        self,
        candidate_term: int,
        candidate_id: int,
        last_log_index: int,
        last_log_term: int,
    ) -> Tuple[int, bool]:
        """
        Process RequestVote RPC.
        Returns (current_term, vote_granted).
        """
        if candidate_term < self.current_term:
            return self.current_term, False

        if candidate_term > self.current_term:
            self.current_term = candidate_term
            self.state = NodeState.FOLLOWER
            self.voted_for = None

        already_voted = (
            self.voted_for is not None and self.voted_for != candidate_id
        )
        if already_voted:
            return self.current_term, False

        if not self.is_up_to_date(last_log_term, last_log_index):
            return self.current_term, False

        self.voted_for = candidate_id
        return self.current_term, True


# ── Election helper ────────────────────────────────────────────────────────────

def run_election(candidate: RaftNode, peers: List[RaftNode]) -> bool:
    """
    Candidate requests votes from peers.
    Returns True if it wins a majority (including itself).
    """
    candidate.current_term += 1
    candidate.voted_for = candidate.node_id
    candidate.state = NodeState.CANDIDATE

    votes = 1  # self-vote
    total = 1 + len(peers)

    for peer in peers:
        _, granted = peer.request_vote(
            candidate.current_term,
            candidate.node_id,
            candidate.last_log_index(),
            candidate.last_log_term(),
        )
        if granted:
            votes += 1

    if votes > total // 2:
        candidate.state = NodeState.LEADER
        return True
    else:
        candidate.state = NodeState.FOLLOWER
        return False


def replicate_entries(
    leader: RaftNode,
    followers: List[RaftNode],
    entries: List[LogEntry],
) -> int:
    """
    Append entries to leader's log and replicate to followers.
    Returns number of followers that successfully accepted.
    """
    # Append to leader
    for entry in entries:
        leader.log.append(entry)

    # Replicate to each follower
    successes = 0
    for follower in followers:
        # Find the last common entry (simplified: send from where follower's log diverges)
        prev_index = len(follower.log) - 1
        prev_term = follower.log[prev_index].term if prev_index >= 0 else 0

        # Compute entries to send
        send_from = prev_index + 1
        new_entries = leader.log[send_from:]

        ok = follower.append_entries(
            leader.current_term,
            prev_index,
            prev_term,
            new_entries,
            leader.commit_index,
        )
        if ok:
            successes += 1

    # Commit if quorum achieved (leader + followers)
    total_nodes = 1 + len(followers)  # simplified: only partition members
    if 1 + successes > total_nodes // 2:
        # Advance leader commit index
        leader.commit_index = len(leader.log) - 1
        # Update committed followers
        for follower in followers:
            follower.append_entries(
                leader.current_term,
                leader.last_log_index(),
                leader.last_log_term(),
                [],
                leader.commit_index,
            )

    return successes


# ── Invariant checker ──────────────────────────────────────────────────────────

def check_invariants(nodes: List[RaftNode], label: str = "") -> bool:
    """
    Verify Raft's core safety invariants.
    1. Election Safety: at most one leader per term.
    2. Log Matching: same (index, term) implies identical prefixes.
    3. State Machine Safety: committed entries are identical across nodes.
    """
    ok = True
    prefix = f"[{label}] " if label else ""

    # Election Safety
    leaders_by_term: Dict[int, List[int]] = {}
    for n in nodes:
        if n.state == NodeState.LEADER:
            leaders_by_term.setdefault(n.current_term, []).append(n.node_id)
    for term, leaders in leaders_by_term.items():
        if len(leaders) > 1:
            print(f"{prefix}VIOLATION: Multiple leaders {leaders} in term {term}")
            ok = False

    # Log Matching + State Machine Safety
    for i in range(len(nodes)):
        for j in range(i + 1, len(nodes)):
            li, lj = nodes[i].log, nodes[j].log
            min_len = min(len(li), len(lj))
            for idx in range(min_len):
                if li[idx].term == lj[idx].term and li[idx] != lj[idx]:
                    print(
                        f"{prefix}LOG MATCHING VIOLATION at index {idx}: "
                        f"N{nodes[i].node_id}={li[idx]} N{nodes[j].node_id}={lj[idx]}"
                    )
                    ok = False
            # State Machine Safety: committed region must match
            min_commit = min(nodes[i].commit_index, nodes[j].commit_index)
            for idx in range(min_commit + 1):
                if idx < len(li) and idx < len(lj) and li[idx] != lj[idx]:
                    print(
                        f"{prefix}STATE MACHINE SAFETY VIOLATION at committed index {idx}: "
                        f"N{nodes[i].node_id}={li[idx]} N{nodes[j].node_id}={lj[idx]}"
                    )
                    ok = False

    return ok


# ── Main simulation ────────────────────────────────────────────────────────────

def simulate_partition_scenario():
    """
    Reproduce the exact exercise scenario step by step.
    """
    print("=" * 65)
    print("Raft Partition Scenario Simulation")
    print("=" * 65)

    # Initial setup: N1 is leader at term 3, logs as given
    nodes = [RaftNode(node_id=i) for i in range(1, 6)]
    N1, N2, N3, N4, N5 = nodes

    # Build initial log state from the exercise
    initial_log_data = {
        N1: [(1,'a'), (1,'b'), (2,'c'), (3,'d')],
        N2: [(1,'a'), (1,'b'), (2,'c'), (3,'d')],
        N3: [(1,'a'), (1,'b'), (2,'c')          ],
        N4: [(1,'a'), (1,'b'), (2,'c')          ],
        N5: [(1,'a'), (1,'b')                   ],
    }
    for node, entries in initial_log_data.items():
        node.log = [LogEntry(term=t, command=c) for t, c in entries]

    N1.current_term = 3
    N1.state = NodeState.LEADER
    N1.commit_index = 2   # (2,c) at index 2 was committed; (3,d) is NOT
    N2.current_term = 3
    N2.commit_index = 2
    for n in [N3, N4, N5]:
        n.current_term = 3
        n.commit_index = 2

    print("\n--- Initial State ---")
    for n in nodes:
        print(f"  N{n.node_id}: log={n.log}  commit={n.commit_index}  term={n.current_term}  state={n.state.value}")

    assert check_invariants(nodes, "Initial"), "Initial state invariants failed"

    # Partition: {N1, N2} | {N3, N4, N5}
    print("\n--- Partition: {N1,N2} | {N3,N4,N5} ---")
    minority_partition = [N1, N2]
    majority_partition = [N3, N4, N5]

    # N3 runs election in majority partition
    peers_for_n3 = [N4, N5]
    won = run_election(N3, peers_for_n3)
    assert won, "N3 should win election with majority partition"
    print(f"  N3 elected leader at term {N3.current_term}")

    # Check: N1 and N2 cannot elect a new leader (only 2 nodes, need 3)
    # We verify this by noting they have no quorum — the simulation just
    # stalls them (in a real system, election timeouts fire but CAS fails).

    # N3 appends and commits (4,e) and (4,f)
    new_entries = [LogEntry(4, 'e'), LogEntry(4, 'f')]
    replicate_entries(N3, [N4, N5], new_entries)

    print(f"  After N3 commits:")
    for n in majority_partition:
        print(f"    N{n.node_id}: log={n.log}  commit={n.commit_index}")

    assert N3.commit_index == 4, f"Expected commit_index=4, got {N3.commit_index}"
    assert check_invariants(majority_partition, "After N3 commits"), \
        "Majority partition invariants failed"

    # Partition heals: N1 and N2 receive AppendEntries from N3 (term 4 > 3)
    print("\n--- Partition Heals ---")

    for follower in [N1, N2]:
        # N3's AppendEntries: prevLogIndex=2, prevLogTerm=2, entries=[(4,e),(4,f)]
        ok = follower.append_entries(
            leader_term=4,
            prev_index=2,
            prev_term=2,
            entries=[LogEntry(4, 'e'), LogEntry(4, 'f')],
            leader_commit=4,
        )
        assert ok, f"N{follower.node_id} should accept N3's AppendEntries"

    print("  After heal:")
    for n in nodes:
        print(f"    N{n.node_id}: log={n.log}  commit={n.commit_index}  term={n.current_term}")

    # Verify (3,d) is gone and (4,e),(4,f) are present
    for n in nodes:
        assert len(n.log) == 5, f"N{n.node_id} should have 5 entries, has {len(n.log)}"
        assert n.log[3] == LogEntry(4, 'e'), f"N{n.node_id} index 3 should be (4,e)"
        assert n.log[4] == LogEntry(4, 'f'), f"N{n.node_id} index 4 should be (4,f)"
        assert n.commit_index == 4, f"N{n.node_id} commit_index should be 4"

    print("\n  (3,d) was overwritten on N1 and N2 — as expected.")
    assert check_invariants(nodes, "Final"), "Final state invariants failed"
    print("\nAll invariants hold. Simulation complete.")

    # ── Variant: (3,d) committed before partition ──────────────────────────────
    print("\n" + "=" * 65)
    print("Variant: (3,d) committed before partition")
    print("=" * 65)

    # Reset
    nodes2 = [RaftNode(node_id=i) for i in range(1, 6)]
    V1, V2, V3, V4, V5 = nodes2

    committed_before = {
        V1: [(1,'a'), (1,'b'), (2,'c'), (3,'d')],
        V2: [(1,'a'), (1,'b'), (2,'c'), (3,'d')],
        V3: [(1,'a'), (1,'b'), (2,'c'), (3,'d')],  # committed to quorum
        V4: [(1,'a'), (1,'b'), (2,'c'), (3,'d')],
        V5: [(1,'a'), (1,'b')                   ],
    }
    for node, entries in committed_before.items():
        node.log = [LogEntry(term=t, command=c) for t, c in entries]

    for n in [V1, V2, V3, V4, V5]:
        n.current_term = 3
        n.commit_index = 3  # (3,d) is committed

    V1.state = NodeState.LEADER

    # N3 elections in {V3, V4, V5}: V3's lastLogTerm=3 beats V5's lastLogTerm=1
    peers_v3 = [V4, V5]
    won = run_election(V3, peers_v3)
    assert won
    print(f"  V3 elected leader at term {V3.current_term}")

    # V3 appends (4,e) and (4,f) after (3,d)
    replicate_entries(V3, [V4, V5], [LogEntry(4, 'e'), LogEntry(4, 'f')])

    # Heal: V1 and V2 receive AppendEntries from V3
    # prevLogIndex=3, prevLogTerm=3 — V1/V2 already have (3,d), so match
    for follower in [V1, V2]:
        ok = follower.append_entries(
            leader_term=4,
            prev_index=3,
            prev_term=3,
            entries=[LogEntry(4, 'e'), LogEntry(4, 'f')],
            leader_commit=5,
        )
        assert ok

    print("  After heal (variant):")
    for n in nodes2:
        print(f"    N{n.node_id}: log={n.log}  commit={n.commit_index}")

    # (3,d) must still be at index 3 in all nodes
    for n in nodes2:
        assert n.log[3] == LogEntry(3, 'd'), \
            f"N{n.node_id}: (3,d) should be preserved at index 3"

    print("\n  (3,d) was PRESERVED — Leader Completeness in action.")
    assert check_invariants(nodes2, "Variant final"), "Variant invariants failed"
    print("  All invariants hold.")


if __name__ == "__main__":
    simulate_partition_scenario()
```

---

## 4. Common Wrong Approaches and Why They Fail

### Wrong Approach 1: Assuming Uncommitted Entries Are Safe

**Mistake:** "N1 was leader and had `(3,d)` in its log, so it must have committed it before the partition."

**Why it fails:** A Raft entry is not committed just because the leader has it. Commitment requires acknowledgment from a quorum. Here, only N1 and N2 had `(3,d)` (2/5 nodes). N1 never sent a commit response to the client for `(3,d)`. The distinction between "in the log" and "committed" is the entire point of the two-phase commit-like structure in Raft.

### Wrong Approach 2: Thinking the Higher Term Means Safety

**Mistake:** "Term 3 is lower than term 4, so (3,d) is obsolete and unsafe."

**Why it fails:** Term numbers convey *election freshness*, not semantic correctness of entries. An entry from term 1 can be perfectly durable if it was committed. The *commitment* of (1,a), (1,b), (2,c) is what makes them permanent — not their term number.

### Wrong Approach 3: Believing Election Restriction Protects All Log Entries

**Mistake:** "The Election Restriction ensures all log entries from previous terms are preserved."

**Why it fails:** The Election Restriction only preserves *committed* entries. An uncommitted entry like `(3,d)` is deliberately allowed to be lost. The subtle implication: a node cannot unilaterally decide that an entry from a previous term is committed — it must replicate a new entry in the current term to confirm it (Raft's "commitment by proxy" rule).

### Wrong Approach 4: Thinking Both Partitions Can Have Active Leaders

**Mistake:** "N1 stays leader in the minority partition and commits (3,d) there."

**Why it fails:** N1 cannot commit anything without a quorum of 3. With only N1 and N2, any client request will stall indefinitely. N1 does not commit `(3,d)` during the partition — it just has it in its log without a quorum acknowledgment. This is a key Raft safety property: availability is sacrificed in the minority partition in exchange for consistency.

### Wrong Approach 5: Split-Brain via Stale Leader Messages

**Mistake:** "After the partition heals, if N1 still thinks it's leader (term 3), it could conflict with N3 (term 4)."

**Why it fails:** When N1 receives N3's AppendEntries with `term=4`, the very first check is `if leader_term < self.current_term: reject`. Here `4 > 3`, so N1 immediately updates its term to 4 and steps down. This is the "term as a logical clock" mechanism — any stale leader is automatically demoted the moment it receives a message from a higher-term node.

---

## 5. Extensions: Scale and Adversarial Scenarios

### 1,000-Node Cluster

With 1,000 nodes, election timeouts become critical. A straightforward election requires 501 votes; at typical 1ms per RPC, this takes ~500ms of sequential voting, too slow for most workloads.

**Solution:** Pre-vote extension (Raft paper §9.6). Before incrementing its term, a candidate asks peers whether they would vote for it. This prevents disruptive elections from nodes that rejoined from a long partition. At 1,000 nodes, Raft is typically layered with a configuration change mechanism (joint consensus) to limit quorum overhead.

Production-scale deployments (TiKV, CockroachDB) use **Multi-Raft** — sharding data into thousands of Raft groups of 3–5 nodes each, rather than running 1,000-node Raft.

### 1TB Logs

At 1TB of log data, N3 cannot send the full log to N1. The solution is **log compaction via snapshots** (Raft §7). The leader takes a snapshot of the state machine at a given log index and discards log entries before it. When a lagging follower needs entries the leader has already compacted, the leader sends the snapshot via `InstallSnapshot` RPC.

In our scenario: N5 has only `[(1,a), (1,b)]`. If the rest of the log has been compacted, N3 sends N5 a snapshot first, then streams newer entries. This is exactly how etcd handles lagging members.

### 10ms End-to-End Latency Budget

With a 10ms round-trip latency budget, the write path has:
- Client → Leader: ~1ms
- Leader → Quorum (parallel AppendEntries): ~2ms
- Followers write to WAL and respond: ~3ms
- Leader receives quorum acks and applies: ~1ms
- Leader responds to client: ~1ms

Total: ~8ms, leaving minimal margin. Optimizations used in practice:
- **Pipeline AppendEntries**: don't wait for one batch to commit before sending the next
- **Batching**: accumulate 100–1,000 client requests into one AppendEntries
- **Follower read with lease**: reads from the leader without an extra round trip (etcd)

### Adversarial Inputs: Byzantine Faults

Standard Raft assumes crash-fault tolerance (CFT) — nodes can crash but not lie. An adversarial node could send arbitrary messages, forge signatures, or selectively drop replies to cause liveness failures.

**PBFT / Tendermint / HotStuff** handle Byzantine faults with `3f+1` nodes tolerating `f` Byzantine nodes. The key addition: two phases of quorum acknowledgment (prepare + commit) to prevent equivocation (a Byzantine leader sending different values to different followers).

For production systems with Byzantine fault requirements (consortium blockchains, cross-organizational consensus), **HotStuff** (used in LibraBFT/Diem) achieves O(n) message complexity per round via a chain of threshold signatures, vs. PBFT's O(n²).

---

## 6. Real-World Production System References

### etcd (Kubernetes Control Plane)

etcd's Raft implementation in Go (`go.etcd.io/etcd/raft`) handles exactly this partition/heal scenario. The `maybeCommit()` function in `raft.go` only advances `commitIndex` when an entry from the *current term* has a quorum — this is the mechanism that prevents `(3,d)` from being committed in the minority partition.

Key source: `etcd/raft/raft.go` — `func (r *raft) maybeCommit()` — checks `r.log.maybeCommit(r.raftLog.lastIndex(), r.Term)`.

### CockroachDB (Distributed SQL)

CockroachDB uses Raft per-range (default 64MB ranges). The `Store.processRaftRequest()` path in `pkg/kv/kvserver/replica_raft.go` handles log truncation and snapshot installation for lagging replicas. Their "log truncation pending" mechanism prevents compacting logs below the oldest known applied index across all replicas.

### TiKV (Distributed KV, used by TiDB)

TiKV's `raft-rs` crate (Rust) is the most cited Raft library in production. The `handle_append_entries` function implements the exact log conflict resolution described above: find the last matching entry, truncate after it, append new entries. Source: `components/raft-rs/src/raft.rs`.

### Formal Verification: Raft.tla

Diego Ongaro's TLA+ specification (`github.com/ongardie/raft.tla`) encodes the ElectionSafety and LeaderCompleteness invariants as model-checkable properties. The TLC model checker verifies them for 3–5 nodes exhaustively (~10M states for 3 nodes, 3 terms). Running it catches subtle bugs that informal reasoning misses.

### The Lesson from Raft vs. Paxos

The partition scenario above is precisely what Paxos handles with its `prepare` phase but without the intuitive invariant structure Raft provides. Lamport's MultiPaxos requires a separate argument for why `(3,d)` cannot be committed after the partition — in Raft, it follows directly from the Election Restriction theorem. This is why Raft was adopted so rapidly: the invariants are teachable and mechanically verifiable.
