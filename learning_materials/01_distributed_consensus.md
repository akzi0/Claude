# Distributed Consensus: From Impossibility to Production Systems

---
> **Navigation** | [← CURRICULUM](../CURRICULUM.md) | [Solutions →](../solutions/01_solution.md)
>
> **Prerequisites:** networking basics, basic Python, set theory
>
> **Related Lessons:**
> - 🔗 Lesson 02 — Database Internals: Databases use Raft for replication — etcd backs Kubernetes
> - 🔗 Lesson 07 — Cryptography Internals: TLS 1.3 protects Raft RPC channels
>
> **Estimated time:** 3 hours reading + 2 hours exercise
---

*Graduate Seminar Material — Distributed Systems*
*References: Lamport (1998, 2001), Ongaro & Ousterhout (2014), Chandra et al. (2007), Fischer, Lynch & Paterson (1985)*

---

## Table of Contents

1. [Why Consensus Is Hard](#1-why-consensus-is-hard)
   - Two Generals Problem
   - FLP Impossibility
   - Safety vs. Liveness / CAP
2. [Paxos from First Principles](#2-paxos-from-first-principles)
   - Roles, Quorum Intersection
   - Phase 1 & 2 Protocol
   - Safety Invariant Proof
   - Multi-Paxos
   - Python Single-Decree Paxos Implementation
3. [Raft](#3-raft)
   - Log Structure & Election
   - Log Replication & Log Matching
   - Commitment Rule & Figure 8
   - Leader Completeness
   - Cluster Membership Changes
4. [Python Implementation — Simplified Raft](#4-python-implementation--simplified-raft)
5. [Byzantine Fault Tolerance](#5-byzantine-fault-tolerance)
   - PBFT, HotStuff, Tendermint
6. [What Textbooks Don't Tell You](#6-what-textbooks-dont-tell-you)
7. [Edge Cases & Failure Modes](#7-edge-cases--failure-modes)
8. [Hard Exercise](#8-hard-exercise-trace-through-a-complex-partition)
9. [Consistency Models Deep Dive](#9-consistency-models-deep-dive)
10. [Consensus in Production Systems](#10-consensus-in-production-systems)
11. [Multi-Raft and Sharding](#11-multi-raft-and-sharding)
12. [Performance & Optimization](#12-performance--optimization)
13. [Glossary](#13-glossary)

---

## 1. Why Consensus Is Hard

Consensus — the problem of getting a set of processes to agree on a single value — appears deceptively simple. It is not. Three foundational impossibility results bound what is achievable, and engineers who do not internalize them build systems that fail in the field in confusing ways.

### The Two Generals Problem

**Definition.** Two armies (A and B) must coordinate an attack on a fortified city. They communicate only via messengers that may be captured (i.e., messages may be lost). Both generals must attack simultaneously to win; attacking alone means defeat. Can they reach guaranteed agreement?

**Formal statement.** Let $G_A$ and $G_B$ be two processes communicating over an unreliable channel. Each process must output `ATTACK` or `WAIT`. Agreement requires that both output the same value, and validity requires that if both initially prefer `ATTACK`, they can agree on `ATTACK`.

**Impossibility proof sketch.**

Assume for contradiction that a protocol $P$ solves this. Let $M$ be the set of messages exchanged in an execution where both generals attack. Consider removing the last message in $M$ — call it $m_k$ sent by $G_A$ to $G_B$.

In the modified execution $M' = M \setminus \{m_k\}$, $G_B$ cannot distinguish this from the case where $m_k$ was lost. Yet in $M'$, $G_B$ still outputs `ATTACK` (it cannot know $m_k$ was never delivered). Now $G_A$ sent $m_k$ but received no acknowledgment. $G_A$ cannot distinguish this from the case where $m_k$ was lost. So $G_A$ must still commit to `ATTACK` even without confirmation.

We can repeat this argument inductively: strip $m_{k-1}$, $m_{k-2}$, ... down to the empty message set. At the empty set, both generals must still attack (because any finite protocol must eventually decide), but neither has received *any* communication. Now flip $G_A$'s input to `WAIT`: same empty-message execution, $G_A$ outputs `ATTACK` — violating validity.

The core issue: any protocol that achieves agreement must tolerate message loss up to some bound $k$, but no finite $k$ suffices because the adversary can always drop one more message. Agreement over unreliable channels is unachievable without additional assumptions (e.g., bounded message delay, which is precisely what reliable TCP provides by treating a connection drop as a crash).

**Engineering consequence.** TCP's three-way handshake does not solve Two Generals — it fails if the final ACK is lost. Distributed systems solve this by adding timeouts and idempotent retries, but this trades formal safety for *practical* reliability.

---

### FLP Impossibility (Fischer, Lynch, Paterson 1985)

**Theorem.** No deterministic algorithm solves consensus in an asynchronous system if even one process may crash.

This is the most important theorem in distributed systems. It rules out a whole class of algorithms before you write a line of code.

**Model.** $N$ processes communicate by asynchronous message passing. Messages are delivered eventually but with unbounded delay. Processes can fail by crashing (stopping silently). An algorithm solves consensus if:

- **Termination**: Every non-faulty process eventually decides.
- **Agreement**: No two processes decide differently.
- **Validity**: If all processes propose the same value $v$, the only possible decision is $v$.

**Bivalency argument.**

A configuration $C$ is **bivalent** if both decision values (0 and 1) are reachable from $C$. It is **univalent** (0-valent or 1-valent) if only one value is reachable.

*Lemma 1: There exists a bivalent initial configuration.*

Proof: Suppose not — every initial configuration is univalent. Consider a sequence of initial configs $C_0, C_1, ..., C_N$ where $C_0$ has all processes proposing 0 and $C_N$ has all proposing 1. By validity, $C_0$ is 0-valent and $C_N$ is 1-valent. There must be adjacent configs $C_i$ (0-valent) and $C_{i+1}$ (1-valent) differing only in process $p$'s initial value. But if $p$ crashes immediately, the two configs are indistinguishable — contradiction.

*Lemma 2: From any bivalent configuration, there is always a step that leads to another bivalent configuration.*

Proof: Let $C$ be bivalent. Consider event $e = (p, m)$ (process $p$ receives message $m$). Let $\mathcal{D}$ be the set of configurations reachable from $C$ without applying $e$. Some configs in $\mathcal{D}$ are 0-valent, some are 1-valent (because $C$ is bivalent). There must be adjacent configs $C', C''$ in $\mathcal{D}$ where applying $e$ to $C'$ gives 0-valent and to $C''$ gives 1-valent. But $C'$ and $C''$ differ by one event $e'$ applied to some process $p' \neq p$.

If $p \neq p'$, then the two events commute and we get a contradiction by applying them in both orders.

If $p = p'$, then $p$ has crashed (it processes two messages in different orders), and again we reach contradiction.

Therefore, there exists a configuration reachable from $C$ (after applying $e$) that is still bivalent.

**Corollary.** Starting from the bivalent initial configuration, any deterministic algorithm can be kept in bivalent states indefinitely by a carefully chosen message schedule. The algorithm never terminates — violating termination.

**Practical resolution.** FLP rules out *deterministic* algorithms in *purely asynchronous* systems. Real systems break one of these constraints:

1. **Add randomization** (Lamport's approach, Ben-Or's randomized consensus): randomness breaks the adversary's ability to keep the system bivalent.
2. **Assume partial synchrony** (Dwork, Lynch, Stockmeyer 1988): there exists an unknown global stabilization time (GST) after which messages are delivered within $\Delta$. Paxos and Raft operate in this model.
3. **Failure detectors** (Chandra & Toueg 1996): an oracle that hints at which processes may have crashed. The weakest failure detector sufficient for consensus is $\Diamond W$ (eventually weak).

---

### Safety vs. Liveness

**Formal definitions.**

A property $P$ of an execution is a **safety property** if: for every execution $\sigma$ that violates $P$, there exists a finite prefix $\hat{\sigma}$ of $\sigma$ such that no extension of $\hat{\sigma}$ satisfies $P$.

Informally: "nothing bad ever happens." You can verify a safety violation in finite time by examining a finite prefix.

A property $P$ is a **liveness property** if: for every finite execution prefix $\hat{\sigma}$, there exists an extension $\sigma$ of $\hat{\sigma}$ that satisfies $P$.

Informally: "something good eventually happens." A liveness violation can never be detected in finite time — you must wait forever.

In consensus:
- **Agreement** is a safety property: once two processes disagree, no future steps can undo it.
- **Termination** is a liveness property: we cannot certify at any finite point that a process *will never* decide.

**CAP and the safety/liveness tradeoff.**

Brewer's CAP theorem (formalized by Gilbert & Lynch 2002): in the presence of a network partition, a distributed system cannot simultaneously guarantee **Consistency** (linearizability — safety) and **Availability** (every request receives a response — liveness).

Proof sketch: Consider two nodes separated by a partition. If both must respond (availability), they may respond with stale data (violating consistency). If they must agree before responding (consistency), one must block indefinitely if the partition never heals (violating availability).

Consensus protocols like Paxos and Raft choose **CP** — they sacrifice availability (leaders refuse requests during partitions) to preserve safety (no divergent decisions). Systems like Dynamo choose **AP** — they serve stale reads and reconcile later.

---

## 2. Paxos from First Principles

Paxos (Lamport 1989, published 1998) is a protocol for reaching consensus in a partially synchronous distributed system tolerating $f$ crash failures with $2f+1$ nodes.

### Roles

- **Proposers**: Initiate consensus rounds with candidate values.
- **Acceptors**: Store promises and accept values. Their votes constitute quorums.
- **Learners**: Observe the chosen value; in practice, usually the proposer acts as learner too.

Nodes commonly play all three roles simultaneously.

**Quorum intersection property.** Any two quorums $Q_1, Q_2 \subseteq \text{Acceptors}$ must satisfy $Q_1 \cap Q_2 \neq \emptyset$. The canonical quorum is a majority ($\lfloor N/2 \rfloor + 1$). This single property is what makes Paxos safe.

### Phase 1: Prepare / Promise

**Proposer** chooses a globally unique ballot number $n$ (typically node_id + sequence_number) and broadcasts `Prepare(n)` to all acceptors.

**Acceptor** upon receiving `Prepare(n)`:
1. If $n > \text{highestPromised}$: set `highestPromised = n`, respond `Promise(n, acceptedBallot, acceptedValue)` where `acceptedBallot` and `acceptedValue` are the highest-ballot value previously accepted (null if none).
2. Otherwise: ignore or respond with `Nack`.

### Phase 2: Accept / Accepted

**Proposer** upon receiving Promise from a quorum $Q$:
1. Let $v^* = \text{value with highest acceptedBallot among responses}$, or proposer's own value if all returned null.
2. Broadcast `Accept(n, v^*)` to all acceptors.

**Acceptor** upon receiving `Accept(n, v)`:
1. If $n \geq \text{highestPromised}$: set `acceptedBallot = n`, `acceptedValue = v`, respond `Accepted(n, v)`.
2. Otherwise: ignore or Nack.

**Proposer** upon receiving `Accepted` from a quorum: value $v$ is **chosen**.

### Safety Invariant Proof

**Claim.** If value $v$ is chosen in ballot $b$, then any ballot $b' > b$ will also propose $v$.

**Proof** by induction on ballot number.

Base case: $v$ is chosen in ballot $b$. This means some quorum $Q$ accepted $\text{Accept}(b, v)$.

Inductive step: Assume all ballots $b, b+1, ..., b'-1$ propose $v$. We show ballot $b'$ also proposes $v$.

In Phase 1 of ballot $b'$, the proposer receives Promises from some quorum $Q'$. Since $|Q| + |Q'| > N$ (both are majorities), $Q \cap Q' \neq \emptyset$ — at least one acceptor $a \in Q \cap Q'$ exists.

Acceptor $a$ accepted $\text{Accept}(b, v)$, so $a$'s `acceptedBallot` $\geq b$. When $a$ responds to `Prepare(b')` (with $b' > b \geq a$'s highest promise), it reports its accepted value with ballot $\geq b$.

In Phase 2, the proposer picks the value with the highest ballot in the responses. By inductive hypothesis, all ballots $b$ through $b'-1$ proposed $v$. So the highest-ballot response from $Q'$ has value $v$. The proposer must propose $v$.

**Corollary.** At most one value can ever be chosen (Agreement/Safety holds).

### Flexible Paxos (Howard et al. 2016)

The requirement that every quorum intersect is sufficient for safety, but traditional Paxos uses the *same* quorum size for both Phase 1 and Phase 2. Flexible Paxos relaxes this: you only need Phase 1 quorums and Phase 2 quorums to intersect — they do not have to be majority quorums individually.

**Theorem (Flexible Paxos).** Consensus is safe if and only if every Phase 1 quorum $Q_1$ and every Phase 2 quorum $Q_2$ satisfy $Q_1 \cap Q_2 \neq \emptyset$.

This enables a continuum of configurations:

| Phase 1 quorum | Phase 2 quorum | Interpretation |
|---|---|---|
| $n/2 + 1$ (majority) | $n/2 + 1$ (majority) | Classic Paxos |
| $n$ (all) | $1$ (single node) | Fastest writes, worst recovery |
| $1$ (single node) | $n$ (all) | Fastest leader election, slowest writes |
| $\lceil n/3 \rceil + 1$ | $\lceil 2n/3 \rceil$ | Optimized for write-heavy workloads |

**Practical implication.** In a geo-distributed 5-node cluster spanning 3 data centers, you can set Phase 2 quorum = 2 (both nodes in the primary DC) for lowest write latency, and Phase 1 quorum = 4 (requiring nodes from at least 2 DCs) for leader election safety. This trades slow leader elections for fast steady-state writes.

```python
def flexible_paxos_safety_check(n_nodes: int,
                                 phase1_quorum: int,
                                 phase2_quorum: int) -> dict:
    """
    Check whether a Flexible Paxos configuration is safe.
    The necessary and sufficient condition: phase1_quorum + phase2_quorum > n_nodes
    (ensures any Phase 1 quorum intersects any Phase 2 quorum).
    """
    intersects = phase1_quorum + phase2_quorum > n_nodes
    max_failures_p1 = n_nodes - phase1_quorum
    max_failures_p2 = n_nodes - phase2_quorum

    return {
        'safe': intersects,
        'n': n_nodes,
        'q1': phase1_quorum,
        'q2': phase2_quorum,
        'phase1_fault_tolerance': max_failures_p1,
        'phase2_fault_tolerance': max_failures_p2,
        'notes': (
            'SAFE: Q1 ∩ Q2 guaranteed non-empty'
            if intersects else
            f'UNSAFE: Q1({phase1_quorum}) + Q2({phase2_quorum}) = '
            f'{phase1_quorum + phase2_quorum} ≤ {n_nodes}'
        )
    }


def print_flexible_paxos_tradeoffs():
    n = 5
    configs = [
        (3, 3, "Classic majority"),
        (5, 1, "Phase2=1: fastest writes, leader election needs all"),
        (2, 4, "Phase2=4: slowest writes, very fast leader change"),
        (4, 2, "Phase2=2: fast writes (2 nearby nodes), slow leader election"),
        (2, 3, "UNSAFE: 2+3=5 not > 5"),
    ]
    print(f"\n=== Flexible Paxos Configurations (n={n}) ===\n")
    for q1, q2, desc in configs:
        r = flexible_paxos_safety_check(n, q1, q2)
        status = "✓" if r['safe'] else "✗"
        print(f"{status} {desc}")
        print(f"  Q1={q1} (tolerates {r['phase1_fault_tolerance']} failures for election)")
        print(f"  Q2={q2} (tolerates {r['phase2_fault_tolerance']} failures for writes)")
        print(f"  {r['notes']}\n")
```

### Multi-Paxos

Single-decree Paxos agrees on one value. Multi-Paxos extends it to an ordered log of values (a replicated state machine):

1. **Stable leader.** One proposer acts as leader for an epoch. After Phase 1 succeeds for ballot $b$, it can skip Phase 1 for all future log slots while it remains leader.
2. **Log structure.** Each slot $i$ is an independent Paxos instance. The leader drives all slots using its established ballot.
3. **Gap handling.** Slots with no learned value (e.g., the leader crashed mid-Phase-2) are filled by a new leader running Phase 1 on that specific slot.

**Underspecification.** Lamport's original paper deliberately omits:
- How leaders are elected (and what happens with dueling proposers)
- How to handle repeated ballot numbers across leaders
- Exactly how log truncation works
- How clients discover the leader

Chandra et al. (2007) "Paxos Made Live" describes Google's production implementation (Chubby) and lists the significant engineering challenges: multi-master read, snapshot/GC, epoch changes, lease management. The gap between "Paxos the algorithm" and "Paxos the system" is enormous.

**Dueling proposers and liveness.** Two proposers can livelock: Proposer A prepares ballot 1, B prepares ballot 2 (invalidating A), A prepares ballot 3 (invalidating B), ad infinitum. Solution: randomized backoff or a single stable leader with a lease.

### Python: Single-Decree Paxos

```python
import threading
import random
import time
from dataclasses import dataclass, field
from typing import Optional, Dict, List, Tuple

@dataclass
class Promise:
    promised_ballot: int
    accepted_ballot: Optional[int]
    accepted_value: Optional[str]

@dataclass
class AcceptorState:
    highest_promised: int = -1
    accepted_ballot: Optional[int] = None
    accepted_value: Optional[str] = None
    _lock: threading.Lock = field(default_factory=threading.Lock)

    def prepare(self, ballot: int) -> Optional[Promise]:
        with self._lock:
            if ballot > self.highest_promised:
                self.highest_promised = ballot
                return Promise(ballot, self.accepted_ballot, self.accepted_value)
            return None  # Nack

    def accept(self, ballot: int, value: str) -> bool:
        with self._lock:
            if ballot >= self.highest_promised:
                self.highest_promised = ballot
                self.accepted_ballot = ballot
                self.accepted_value = value
                return True
            return False  # Nack


class Proposer:
    def __init__(self, node_id: int, acceptors: List[AcceptorState]):
        self.node_id = node_id
        self.acceptors = acceptors
        self.seq = 0

    def _make_ballot(self) -> int:
        # Ballot encoding: (seq << 8) | node_id — globally unique if node_ids unique
        self.seq += 1
        return (self.seq << 8) | self.node_id

    def propose(self, value: str) -> Optional[str]:
        """
        Run single-decree Paxos. Returns the chosen value (may differ from proposed).
        Returns None if consensus could not be reached (quorum unavailable).
        """
        quorum = len(self.acceptors) // 2 + 1

        # Phase 1: Prepare
        ballot = self._make_ballot()
        promises: List[Promise] = []
        for acceptor in self.acceptors:
            result = acceptor.prepare(ballot)
            if result is not None:
                promises.append(result)

        if len(promises) < quorum:
            return None  # Could not get quorum of promises

        # Determine value to propose (highest previously accepted, or our own)
        highest_ballot = -1
        chosen_value = value
        for p in promises:
            if p.accepted_ballot is not None and p.accepted_ballot > highest_ballot:
                highest_ballot = p.accepted_ballot
                chosen_value = p.accepted_value

        # Phase 2: Accept
        accepted_count = 0
        for acceptor in self.acceptors:
            if acceptor.accept(ballot, chosen_value):
                accepted_count += 1

        if accepted_count >= quorum:
            return chosen_value  # chosen_value is now the consensus decision
        return None


def demonstrate_paxos():
    """
    Three acceptors, two competing proposers.
    Shows that despite competition, the safety invariant holds:
    once a value is chosen, all subsequent proposals converge to it.
    """
    print("\n=== Single-Decree Paxos Demonstration ===\n")
    acceptors = [AcceptorState() for _ in range(3)]

    p1 = Proposer(node_id=1, acceptors=acceptors)
    p2 = Proposer(node_id=2, acceptors=acceptors)

    # Proposer 1 runs Phase 1 with ballot (1<<8)|1 = 257
    # Proposer 2 runs Phase 1 with ballot (1<<8)|2 = 258 (higher, wins)
    # P1 gets promises but P2 invalidates them before P1 reaches Phase 2.
    # To simulate: P1 does prepare, then P2 does full propose, then P1 tries accept.

    # Step 1: P1 gets promises
    ballot_p1 = p1._make_ballot()  # 257
    promises_p1 = [a.prepare(ballot_p1) for a in acceptors]
    print(f"P1 prepared ballot {ballot_p1}, got {sum(1 for p in promises_p1 if p)} promises")

    # Step 2: P2 runs a full round (higher ballot, wins)
    result_p2 = p2.propose("value_from_P2")
    print(f"P2 proposed 'value_from_P2', result: {result_p2}")

    # Step 3: P1 tries Phase 2 with its stale ballot — rejected
    accepted_p1 = sum(1 for a in acceptors if a.accept(ballot_p1, "value_from_P1"))
    quorum = len(acceptors) // 2 + 1
    print(f"P1 tried accept with stale ballot {ballot_p1}: {accepted_p1}/{len(acceptors)} accepted (need {quorum})")
    print(f"P1 succeeded: {accepted_p1 >= quorum}")

    # Step 4: P1 retries with new ballot — must pick up P2's value
    result_p1_retry = p1.propose("value_from_P1")
    print(f"P1 retry result: {result_p1_retry}  (must equal P2's value if P2 committed)")

    print("\nFinal acceptor states:")
    for i, a in enumerate(acceptors):
        print(f"  Acceptor {i}: highest_promised={a.highest_promised}, "
              f"accepted=({a.accepted_ballot}, {a.accepted_value!r})")


if __name__ == '__main__':
    demonstrate_paxos()
```

**Key observation.** In `demonstrate_paxos()`, P1's retry will return `'value_from_P2'` — not `'value_from_P1'`. This is the safety invariant in action: P2's value was accepted by a quorum, so any subsequent proposer that runs Phase 1 will see it in the promises and is forced to adopt it. The protocol is self-healing.

---

## 3. Raft

Raft (Ongaro & Ousterhout 2014) was designed for understandability while achieving the same safety guarantees as Paxos. Its key innovation is decomposing consensus into three relatively independent sub-problems: leader election, log replication, and safety.

### Log Structure

Each log entry is a triple `(term, index, command)`. Terms are monotonically increasing integers, one per leader epoch. Indices are 1-based positions in the log.

**Invariants:**
- Entries are immutable once appended by the leader.
- Leaders never delete or overwrite their own entries.
- A leader has all committed entries in its log (Leader Completeness).

### Leader Election

Raft uses randomized timeouts (150–300ms typical) to avoid split votes. Each node starts as a follower. If it receives no heartbeat within its timeout, it becomes a candidate.

**RequestVote RPC:**
```
RequestVote(term, candidateId, lastLogIndex, lastLogTerm) -> (term, voteGranted)
```

**Vote grant rule (Election Restriction §5.4.1):** A node votes for a candidate only if:
1. The candidate's term $\geq$ voter's current term.
2. The voter has not voted in this term.
3. The candidate's log is **at least as up-to-date**: either candidate's `lastLogTerm > voter's lastLogTerm`, or they are equal and candidate's `lastLogIndex >= voter's lastLogIndex`.

The election restriction is the crucial safety mechanism: it ensures that any elected leader contains all committed entries.

A candidate wins by receiving votes from a majority. It immediately sends `AppendEntries` heartbeats to assert leadership and prevent new elections.

### Log Replication

**AppendEntries RPC:**
```
AppendEntries(term, leaderId, prevLogIndex, prevLogTerm, entries[], leaderCommit) 
  -> (term, success)
```

Upon receiving AppendEntries:
1. Reject if `term < currentTerm`.
2. Reject if log does not contain an entry at `prevLogIndex` with term `prevLogTerm` — this is the **consistency check**.
3. If an existing entry conflicts with a new one (same index, different term), delete it and all subsequent entries.
4. Append any new entries.
5. If `leaderCommit > commitIndex`, set `commitIndex = min(leaderCommit, index of last new entry)`.

**Log Matching Property:** If two logs contain an entry with the same index and term, then:
- The entries are identical (command is the same).
- All preceding entries are identical.

**Proof by induction:**
- Base: Leaders create at most one entry per (index, term) pair — enforced by the protocol.
- Inductive step: AppendEntries includes `prevLogIndex/prevLogTerm`. If a follower accepts new entries, its log at `prevLogIndex` matches the leader's (by the consistency check). By induction, all earlier entries match. ∎

### Commitment Rule

A leader commits an entry when it has been replicated to a majority **in the current term**. This subtle rule prevents a dangerous scenario.

**Figure 8 scenario:**

```
Timeline:
S1 (leader, term 2): [1:a, 2:b]
S2:                  [1:a, 2:b]
S3,S4,S5:            [1:a]

S1 crashes. S5 wins election (term 3), log = [1:a].
S5 crashes. S1 recovers, wins election (term 4).
S1 replicates (2:b) to S3 — now majority have it.
Q: Can S1 commit (2:b)?

Answer: NO. S1 must first append and replicate a term-4 entry.
If S1 commits (2:b) directly and then crashes, S5 could win 
another election (its log [1:a] is valid) and overwrite (2:b).
```

The fix: leaders only commit entries from *their own term*. Older entries are committed *indirectly* by committing a newer entry that follows them (Log Matching ensures they are replicated everywhere the newer entry is).

### Leader Completeness

**Theorem.** If a log entry is committed in term $t$, every leader elected in terms $> t$ contains that entry.

**Proof.** An entry is committed when a majority $Q$ has it. A leader in term $t' > t$ won votes from majority $Q'$. Since $Q \cap Q' \neq \emptyset$, some voter $v$ had the committed entry and voted for the new leader. By the Election Restriction, the new leader's log is at least as up-to-date as $v$'s. Since the committed entry is in $v$'s log and the leader's log is at least as up-to-date, the leader has the committed entry. ∎

### Leader Completeness: Closing the Proof Gap

The proof of Leader Completeness above has a subtle gap that the original paper glosses over. Let us close it rigorously.

**The gap.** We claimed: "the leader's log is at least as up-to-date as $v$'s, and since the committed entry is in $v$'s log, the leader has the committed entry." But "at least as up-to-date" compares only the *last* entry. What if the leader's log is longer but has a different entry at the committed index?

**Closing the gap with Log Matching.** Let the committed entry be at index $i$ with term $t$. Voter $v$ has this entry at index $i$. The new leader won, so its log is at least as up-to-date as $v$'s: either `lastLogTerm_leader > lastLogTerm_v` or they are equal and `lastLogIndex_leader >= lastLogIndex_v`.

Case 1: `lastLogTerm_leader > t`. The leader has an entry with term `lastLogTerm_leader > t` at some index $j$. The leader cannot have invented this entry — some quorum $Q''$ accepted it (leaders only append entries from Phase 2). By the Paxos safety invariant applied to Raft: entries at index $j$ with term $>t$ can only be created after the committed entry at index $i$ (earlier terms cannot overwrite committed entries). But wait — the leader could have entries at index $i$ with a term $\neq t$. Does Log Matching help?

Log Matching says: if two logs share `(index, term)`, all preceding entries match. But if the leader's entry at index $i$ has a *different* term from the committed entry's term $t$, Log Matching does not apply directly.

Here is the resolution. The committed entry was accepted by a quorum $Q$ in term $t$ at index $i$. Any log that contains an entry at `(index=i, term=t')` with $t' \neq t$ was created by a leader in term $t'$. If $t' < t$: this leader came before the committing leader, so it cannot have overwritten an entry committed in a later term (committed entries are never overwritten — that's what we're proving). Circular? No: we prove it by strong induction on the term of the committed entry.

**Formal proof by strong induction on term $t$:**

- **Base case** $t=1$: If an entry is committed in term 1, the first leader (term 1) replicated it to a majority. No entry with term 0 exists (terms start at 1). The first election winner has an empty log or only term-1 entries. Any future leader must have received votes from majority $Q'$. Since $|Q| \cap |Q'| \geq 1$, the voter has the committed entry. The election restriction ensures the new leader's log is at least as long with at least the same last term (term 1). By Log Matching, the new leader has the same entries through the committed index.

- **Inductive step**: Assume all entries committed in terms $1, ..., t-1$ are preserved by all future leaders. Consider an entry committed in term $t$ at index $i$. The committing leader (call it $L$) replicated this to quorum $Q$ and committed it. Any future leader $L'$ (term $t'>t$) wins votes from quorum $Q'$. By quorum intersection, some voter $v \in Q \cap Q'$ has the committed entry at index $i$, term $t$. The election restriction ensures $L'$ has `(lastLogTerm, lastLogIndex) >= v's (lastLogTerm, lastLogIndex)`. Since $v$'s last log entry is at least at index $i$ with term $t$, $L'$'s last entry has term $\geq t$. If $L'$'s entry at index $i$ has term $t$ → Log Matching gives us exactly what we need. If $L'$'s entry at index $i$ has term $t'' > t$ → $t''$ was created by a leader in term $t''$. This leader ran after term $t$ (since terms are monotone). By the inductive hypothesis, that leader's log also contained the committed entry at index $i$, term $t$. But then it created an entry at index $i$ with a *different* term $t''$ — which contradicts the leader's rule of never rewriting entries (leaders only append). **Contradiction**. So $L'$'s entry at index $i$ must have term $t$. By Log Matching, all entries through index $i$ match $L$'s. ∎

### Single-Node Raft and the No-Op Entry

When a Raft leader is elected, it does not immediately know what is committed. Consider: the previous leader committed entry $e$ to 3/5 nodes and crashed. The new leader has $e$ in its log. But `commitIndex` on the new leader reflects only what *it* has applied. The new leader cannot safely serve reads referencing $e$ until it commits something in *its own term*.

**The no-op entry.** Immediately upon election, a Raft leader appends a no-op entry (an empty command) at the current term and replicates it. When this commits, the leader knows its log is fully up-to-date, and all entries from previous terms are also committed (via Log Matching). Only then does it begin serving reads.

This is why you should always see a no-op commit in Raft traces right after an election:

```python
def _become_leader(self):
    self.state = State.LEADER
    self.leader_id = self.node_id
    # Initialize nextIndex and matchIndex
    for peer in self.peers:
        self.next_index[peer.node_id] = len(self.log)
        self.match_index[peer.node_id] = -1
    # Append no-op to commit any uncommitted entries from previous terms
    noop = LogEntry(term=self.current_term, command='NO-OP')
    self.log.append(noop)
    self.logger.info(
        f"Became LEADER term={self.current_term}, appended no-op at "
        f"index={len(self.log)-1}"
    )
```

Without the no-op, a newly elected leader serving read-index reads would correctly confirm leadership via heartbeat but would not know which prior-term entries are committed. The no-op drives this to convergence in one consensus round.

### Cluster Membership Changes

Adding or removing servers is dangerous: naive simultaneous updates create two independent majorities.

**Single-server changes.** Ongaro's dissertation proves that adding or removing one server at a time is safe — no configuration change creates two disjoint majorities if done one node at a time.

**Joint consensus.** The original Raft paper describes a two-phase approach: first transition to $C_{old,new}$ (requiring majorities from both old and new configs), then transition to $C_{new}$. This is more general but complex to implement correctly.

---

## 4. Python Implementation — Simplified Raft

```python
import threading
import time
import random
import logging
from enum import Enum
from dataclasses import dataclass, field
from typing import List, Optional, Dict, Tuple

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s [%(name)s] %(message)s'
)


class State(Enum):
    FOLLOWER = 'follower'
    CANDIDATE = 'candidate'
    LEADER = 'leader'


@dataclass
class LogEntry:
    term: int
    command: str

    def __repr__(self):
        return f"({self.term},{self.command})"


class RaftNode:
    def __init__(self, node_id: int, peers: List['RaftNode']):
        self.node_id = node_id
        self.peers = peers
        self.logger = logging.getLogger(f"Node{node_id}")

        # Persistent state (in production, these must be written to stable storage before responding to RPCs)
        self.current_term: int = 0
        self.voted_for: Optional[int] = None
        self.log: List[LogEntry] = []

        # Volatile state
        self.commit_index: int = -1
        self.last_applied: int = -1
        self.state: State = State.FOLLOWER
        self.leader_id: Optional[int] = None

        # Leader volatile state (reinitialized after election)
        self.next_index: Dict[int, int] = {}
        self.match_index: Dict[int, int] = {}

        # Election timer
        self.election_timeout: float = random.uniform(0.15, 0.30)
        self.last_heartbeat: float = time.time()

        # Partition simulation
        self.partitioned_from: set = set()

        self._lock = threading.RLock()
        self._timer_thread = threading.Thread(target=self._election_loop, daemon=True)
        self._timer_thread.start()

    def _can_communicate(self, peer_id: int) -> bool:
        return peer_id not in self.partitioned_from

    def reset_election_timer(self):
        with self._lock:
            self.last_heartbeat = time.time()
            self.election_timeout = random.uniform(0.15, 0.30)

    def _election_loop(self):
        while True:
            time.sleep(0.01)
            with self._lock:
                if self.state == State.LEADER:
                    self._send_heartbeats()
                    continue
                elapsed = time.time() - self.last_heartbeat
                if elapsed > self.election_timeout:
                    self.logger.info(
                        f"Election timeout after {elapsed:.3f}s, starting election"
                    )
            self.start_election()

    def start_election(self):
        with self._lock:
            self.state = State.CANDIDATE
            self.current_term += 1
            self.voted_for = self.node_id
            votes = 1
            term = self.current_term
            last_log_index = len(self.log) - 1
            last_log_term = self.log[-1].term if self.log else -1
            self.reset_election_timer()

        vote_results = []
        for peer in self.peers:
            if not self._can_communicate(peer.node_id):
                continue
            result = peer.request_vote(term, self.node_id, last_log_index, last_log_term)
            vote_results.append(result)

        with self._lock:
            if self.state != State.CANDIDATE or self.current_term != term:
                return

            for result in vote_results:
                if result['term'] > self.current_term:
                    self._become_follower(result['term'])
                    return
                if result['vote_granted']:
                    votes += 1

            total = len(self.peers) + 1
            if votes > total // 2:
                self._become_leader()

    def _become_leader(self):
        self.state = State.LEADER
        self.leader_id = self.node_id
        self.logger.info(
            f"Became LEADER for term {self.current_term} with log {self.log}"
        )
        for peer in self.peers:
            self.next_index[peer.node_id] = len(self.log)
            self.match_index[peer.node_id] = -1

    def _become_follower(self, term: int):
        self.state = State.FOLLOWER
        self.current_term = term
        self.voted_for = None
        self.reset_election_timer()

    def _send_heartbeats(self):
        for peer in self.peers:
            if not self._can_communicate(peer.node_id):
                continue
            next_idx = self.next_index.get(peer.node_id, len(self.log))
            prev_log_index = next_idx - 1
            prev_log_term = self.log[prev_log_index].term if prev_log_index >= 0 and prev_log_index < len(self.log) else -1
            entries = self.log[next_idx:]
            result = peer.append_entries(
                self.current_term,
                self.node_id,
                prev_log_index,
                prev_log_term,
                list(entries),
                self.commit_index
            )
            if result is None:
                continue
            if result['term'] > self.current_term:
                self._become_follower(result['term'])
                return
            if result['success']:
                new_match = next_idx + len(entries) - 1
                self.match_index[peer.node_id] = max(
                    self.match_index.get(peer.node_id, -1), new_match
                )
                self.next_index[peer.node_id] = new_match + 1
                self._advance_commit_index()
            else:
                self.next_index[peer.node_id] = max(0, next_idx - 1)

    def _advance_commit_index(self):
        for n in range(len(self.log) - 1, self.commit_index, -1):
            if self.log[n].term != self.current_term:
                continue
            replicated_count = 1
            for peer in self.peers:
                if self.match_index.get(peer.node_id, -1) >= n:
                    replicated_count += 1
            total = len(self.peers) + 1
            if replicated_count > total // 2:
                self.commit_index = n
                self.logger.info(
                    f"Committed up to index {n} (entry: {self.log[n]}), "
                    f"log={self.log[:n+1]}"
                )
                break

    def request_vote(
        self,
        term: int,
        candidate_id: int,
        last_log_index: int,
        last_log_term: int
    ) -> dict:
        with self._lock:
            if term > self.current_term:
                self._become_follower(term)

            vote_granted = False
            my_last_log_index = len(self.log) - 1
            my_last_log_term = self.log[-1].term if self.log else -1

            candidate_log_ok = (
                last_log_term > my_last_log_term or
                (last_log_term == my_last_log_term and last_log_index >= my_last_log_index)
            )

            if (
                term >= self.current_term
                and (self.voted_for is None or self.voted_for == candidate_id)
                and candidate_log_ok
            ):
                self.voted_for = candidate_id
                vote_granted = True
                self.reset_election_timer()
                self.logger.info(
                    f"Voted for node {candidate_id} in term {term}"
                )

            return {'term': self.current_term, 'vote_granted': vote_granted}

    def append_entries(
        self,
        term: int,
        leader_id: int,
        prev_log_index: int,
        prev_log_term: int,
        entries: List[LogEntry],
        leader_commit: int
    ) -> dict:
        with self._lock:
            if term < self.current_term:
                return {'term': self.current_term, 'success': False}

            if term > self.current_term:
                self._become_follower(term)

            self.leader_id = leader_id
            self.reset_election_timer()

            # Consistency check
            if prev_log_index >= 0:
                if prev_log_index >= len(self.log):
                    return {'term': self.current_term, 'success': False}
                if self.log[prev_log_index].term != prev_log_term:
                    # Delete conflicting entry and everything after it
                    self.log = self.log[:prev_log_index]
                    return {'term': self.current_term, 'success': False}

            # Append new entries, replacing conflicts
            insert_index = prev_log_index + 1
            for i, entry in enumerate(entries):
                idx = insert_index + i
                if idx < len(self.log):
                    if self.log[idx].term != entry.term:
                        self.log = self.log[:idx]
                        self.log.append(entry)
                    # else: entry already exists and matches
                else:
                    self.log.append(entry)

            if leader_commit > self.commit_index:
                self.commit_index = min(leader_commit, len(self.log) - 1)
                self.logger.info(
                    f"Updated commit_index to {self.commit_index}, "
                    f"log={self.log[:self.commit_index+1]}"
                )

            return {'term': self.current_term, 'success': True}

    def client_request(self, command: str) -> bool:
        with self._lock:
            if self.state != State.LEADER:
                self.logger.warning(
                    f"Rejected client request '{command}': not leader "
                    f"(leader is node {self.leader_id})"
                )
                return False
            entry = LogEntry(term=self.current_term, command=command)
            self.log.append(entry)
            self.logger.info(
                f"Appended to log: {entry} at index {len(self.log)-1}"
            )
        return True

    def status(self) -> dict:
        with self._lock:
            return {
                'node_id': self.node_id,
                'state': self.state.value,
                'term': self.current_term,
                'leader': self.leader_id,
                'log': list(self.log),
                'commit_index': self.commit_index,
            }


def simulate_cluster():
    """
    Simulate a 3-node Raft cluster:
    - Wait for leader election
    - Submit 5 commands
    - Observe log replication and commit index advancing
    """
    print("\n" + "="*60)
    print("RAFT CLUSTER SIMULATION: 3 nodes")
    print("="*60)

    # Create nodes with forward references
    nodes: List[RaftNode] = []
    for i in range(3):
        # peers set after all nodes created
        nodes.append(RaftNode.__new__(RaftNode))

    for i, node in enumerate(nodes):
        peers = [n for n in nodes if n is not node]
        node.__init__(i, peers)

    # Wait for leader election
    print("\n[Phase 1] Waiting for leader election...")
    leader = None
    for _ in range(50):
        time.sleep(0.1)
        for node in nodes:
            s = node.status()
            if s['state'] == 'leader':
                leader = node
                break
        if leader:
            break

    if not leader:
        print("ERROR: No leader elected within timeout")
        return

    print(f"Leader elected: Node {leader.node_id} (term {leader.status()['term']})")

    # Submit 5 commands
    print("\n[Phase 2] Submitting 5 client commands...")
    commands = ['SET x=1', 'SET y=2', 'DEL z', 'INCR counter', 'SET name=raft']
    for cmd in commands:
        success = leader.client_request(cmd)
        print(f"  Command '{cmd}': {'accepted' if success else 'rejected'}")
        time.sleep(0.05)

    # Wait for replication
    print("\n[Phase 3] Waiting for replication and commitment...")
    time.sleep(0.5)

    # Print final state
    print("\n[Final State]")
    print(f"{'Node':<8} {'State':<12} {'Term':<6} {'CommitIdx':<12} {'Log'}")
    print("-" * 70)
    for node in nodes:
        s = node.status()
        print(
            f"  {s['node_id']:<6} {s['state']:<12} {s['term']:<6} "
            f"{s['commit_index']:<12} {s['log']}"
        )

    # Verify all committed entries match leader
    leader_log = leader.status()['log']
    leader_commit = leader.status()['commit_index']
    all_consistent = True
    for node in nodes:
        s = node.status()
        for i in range(min(leader_commit + 1, len(s['log']))):
            if i < len(s['log']) and s['log'][i] != leader_log[i]:
                print(f"\nINCONSISTENCY at node {s['node_id']} index {i}")
                all_consistent = False

    if all_consistent:
        print(f"\nAll nodes consistent through commit index {leader_commit}.")
        print(f"Committed entries: {leader_log[:leader_commit+1]}")


if __name__ == '__main__':
    simulate_cluster()
```

### Extended Simulation: Partition Recovery and Log Divergence

The following extends the cluster simulation to demonstrate partition, divergence, and recovery — the most educational scenario in Raft.

```python
def simulate_partition_and_recovery():
    """
    5-node cluster. Shows:
    1. Leader election
    2. Network partition {0,1} | {2,3,4}
    3. New leader elected in majority partition
    4. Writes committed on new leader
    5. Partition heals
    6. Old leader's uncommitted entries overwritten
    7. Log convergence verified
    """
    print("\n" + "="*65)
    print("RAFT PARTITION + RECOVERY SIMULATION: 5 nodes")
    print("="*65)

    # Bootstrap 5 nodes
    nodes: List[RaftNode] = [RaftNode.__new__(RaftNode) for _ in range(5)]
    for i, node in enumerate(nodes):
        node.__init__(i, [n for n in nodes if n is not node])

    def get_leader() -> Optional[RaftNode]:
        for n in nodes:
            if n.status()['state'] == 'leader':
                return n
        return None

    def print_cluster_state(label: str):
        print(f"\n  [{label}]")
        print(f"  {'Node':<6} {'State':<12} {'Term':<6} {'CmtIdx':<8} {'Log'}")
        print(f"  " + "-"*62)
        for n in nodes:
            s = n.status()
            print(f"  {s['node_id']:<6} {s['state']:<12} {s['term']:<6} "
                  f"{s['commit_index']:<8} {s['log']}")

    # Phase 1: Wait for initial election
    print("\n[Phase 1] Initial election...")
    for _ in range(60):
        time.sleep(0.1)
        if get_leader():
            break
    leader = get_leader()
    assert leader, "No leader elected"
    print(f"  Leader: Node {leader.node_id} (term {leader.status()['term']})")

    # Submit 3 commands before partition
    for cmd in ['SET a=1', 'SET b=2', 'SET c=3']:
        leader.client_request(cmd)
    time.sleep(0.4)
    print_cluster_state("Before partition")

    # Phase 2: Create partition {old_leader, one_follower} | {rest}
    minority = [leader.node_id, (leader.node_id + 1) % 5]
    majority = [i for i in range(5) if i not in minority]
    print(f"\n[Phase 2] Partitioning: minority={minority}, majority={majority}")

    # Each node in minority cannot reach majority and vice versa
    for nid in minority:
        for mid in majority:
            nodes[nid].partitioned_from.add(mid)
            nodes[mid].partitioned_from.add(nid)

    # Old leader tries to write — will NOT commit (no majority)
    for cmd in ['SET x=LOST', 'SET y=LOST']:
        leader.client_request(cmd)

    # Wait for new leader in majority partition
    time.sleep(1.5)
    new_leader = None
    for n in nodes:
        s = n.status()
        if s['state'] == 'leader' and s['node_id'] not in minority:
            new_leader = n
            break
    assert new_leader, "No new leader in majority partition"
    print(f"  New leader: Node {new_leader.node_id} (term {new_leader.status()['term']})")

    # New leader commits entries
    for cmd in ['SET p=committed', 'SET q=committed', 'SET r=committed']:
        new_leader.client_request(cmd)
    time.sleep(0.6)
    print_cluster_state("During partition (after new leader commits)")

    # Phase 3: Heal partition
    print(f"\n[Phase 3] Healing partition...")
    for nid in minority:
        for mid in majority:
            nodes[nid].partitioned_from.discard(mid)
            nodes[mid].partitioned_from.discard(nid)

    time.sleep(1.0)
    print_cluster_state("After partition healed")

    # Verify convergence
    logs = [tuple(str(e) for e in n.status()['log']) for n in nodes]
    commits = [n.status()['commit_index'] for n in nodes]
    converged = len(set(logs)) == 1
    print(f"\n  Logs converged: {converged}")
    print(f"  Commit indices: {commits}")

    # Verify the 'LOST' entries are gone
    final_log = nodes[0].status()['log']
    lost = any('LOST' in e.command for e in final_log)
    print(f"  'LOST' entries in final log: {lost} (should be False)")


if __name__ == '__main__':
    simulate_cluster()
    simulate_partition_and_recovery()
```

### Log Compaction: Snapshot Implementation

```python
import json
import io

@dataclass 
class SnapshotData:
    last_included_index: int
    last_included_term: int
    state_machine: dict  # the actual KV store state

class SnapshotRaftNode(RaftNode):
    """
    Extends RaftNode with snapshot support.
    Demonstrates InstallSnapshot RPC (§7 of the Raft paper).
    """
    SNAPSHOT_THRESHOLD = 5  # take snapshot every 5 applied entries

    def __init__(self, node_id: int, peers: List['RaftNode']):
        super().__init__(node_id, peers)
        self.state_machine: Dict[str, str] = {}  # KV store
        self.last_snapshot: Optional[SnapshotData] = None
        self.snapshot_index: int = -1
        self.snapshot_term: int = -1

    def _apply_entry(self, entry: LogEntry):
        """Apply a committed log entry to the state machine."""
        # Parse simple "SET key=value" or "DEL key" commands
        if entry.command.startswith("SET "):
            parts = entry.command[4:].split("=", 1)
            if len(parts) == 2:
                self.state_machine[parts[0]] = parts[1]
        elif entry.command.startswith("DEL "):
            self.state_machine.pop(entry.command[4:], None)

    def maybe_take_snapshot(self):
        applied = self.commit_index
        since_last = applied - self.snapshot_index
        if since_last >= self.SNAPSHOT_THRESHOLD and applied >= 0:
            self._take_snapshot(applied)

    def _take_snapshot(self, up_to_index: int):
        with self._lock:
            if up_to_index >= len(self.log):
                return
            snap = SnapshotData(
                last_included_index=up_to_index,
                last_included_term=self.log[up_to_index].term,
                state_machine=dict(self.state_machine)
            )
            # Discard log entries up to snapshot (keep remainder)
            self.log = self.log[up_to_index + 1:]
            self.last_snapshot = snap
            self.snapshot_index = up_to_index
            self.snapshot_term = snap.last_included_term
            self.logger.info(
                f"Snapshot taken at index={up_to_index}, "
                f"state={snap.state_machine}, "
                f"remaining_log={self.log}"
            )

    def install_snapshot(self, snap: SnapshotData) -> bool:
        """
        Called when leader sends InstallSnapshot RPC.
        This node is so far behind that the leader has discarded 
        the log entries the node needs.
        """
        with self._lock:
            if snap.last_included_index <= self.snapshot_index:
                return False  # already have a newer snapshot
            # Reset log: discard everything up to snap.last_included_index
            if (snap.last_included_index < len(self.log) and
                    self.log[snap.last_included_index].term == snap.last_included_term):
                # Keep log entries after the snapshot
                self.log = self.log[snap.last_included_index + 1:]
            else:
                self.log = []
            self.state_machine = dict(snap.state_machine)
            self.last_snapshot = snap
            self.snapshot_index = snap.last_included_index
            self.snapshot_term = snap.last_included_term
            self.commit_index = snap.last_included_index
            self.last_applied = snap.last_included_index
            self.logger.info(
                f"Installed snapshot at index={snap.last_included_index}, "
                f"state={snap.state_machine}"
            )
            return True
```

**Running the simulation** will show output similar to:

```
[Phase 1] Waiting for leader election...
Leader elected: Node 2 (term 1)

[Phase 2] Submitting 5 client commands...
  Command 'SET x=1': accepted
  Command 'SET y=2': accepted
  ...

[Final State]
Node     State        Term   CommitIdx    Log
----------------------------------------------------------------------
  0      follower     1      4            [(1,SET x=1), (1,SET y=2), ...]
  1      follower     1      4            [(1,SET x=1), (1,SET y=2), ...]
  2      leader       1      4            [(1,SET x=1), (1,SET y=2), ...]

All nodes consistent through commit index 4.
```

The commit index advances to 4 (0-based index of 5 entries) only after the leader receives acknowledgments from a majority. Notice that log entries are tagged with term 1 throughout because no leadership change occurred.

---

## 5. Byzantine Fault Tolerance

CFT (Crash Fault Tolerance) algorithms like Paxos and Raft assume failures are benign: a node either works correctly or stops responding. In adversarial environments (open networks, multi-tenant hardware, supply-chain attacks), nodes may behave *arbitrarily*: sending incorrect values, selectively dropping messages, or actively colluding.

### Why CFT Fails Against Byzantine Nodes

In Raft, a Byzantine leader can send different `AppendEntries` to different followers, causing them to commit different values at the same index — a safety violation. A Byzantine majority can do anything.

In Paxos, a Byzantine acceptor can lie about its `highestPromised` ballot, causing proposers to use a value other than the one previously chosen. Safety is violated.

**Lower bound.** BFT requires $3f+1$ nodes to tolerate $f$ Byzantine faults. Intuition: the $f$ faulty nodes can impersonate any of the $n-f$ remaining nodes. For a quorum $q$ to always overlap with the $f+1$ honest nodes: $2q - n \geq 1$, and $q \leq n - f$, giving $n \geq 3f+1$.

### PBFT (Practical Byzantine Fault Tolerance, Castro & Liskov 1999)

PBFT requires $n \geq 3f+1$ replicas and uses a **primary** (leader) with three-phase protocol:

1. **Pre-prepare:** Primary assigns sequence number $s$ to request $m$, broadcasts `PRE-PREPARE(v, s, d)` where $d = \text{digest}(m)$.
2. **Prepare:** Each replica broadcasts `PREPARE(v, s, d, i)`. A replica is **prepared** when it has the pre-prepare and $2f$ matching prepares.
3. **Commit:** Each replica broadcasts `COMMIT(v, s, d, i)`. A replica commits when it has $2f+1$ commits (including its own).

**View change** triggers when the primary appears faulty (no progress within timeout). Replicas broadcast `VIEW-CHANGE(v+1, n, C, P, i)` where $P$ contains prepared certificates. A new primary collects $2f+1$ view-change messages and broadcasts `NEW-VIEW`.

**Complexity.** O($n^2$) message complexity per operation. Unsuitable for large-scale open membership.

### HotStuff (Yin et al. 2019, used in Diem/LibraBFT)

HotStuff achieves **linear** communication complexity using **threshold signatures** (BLS threshold crypto). Instead of broadcasting $2f+1$ individual signatures, each replica signs a partial share; a threshold combiner aggregates them into one signature verifiable by anyone.

Three-phase pipeline (Prepare, Pre-commit, Commit) with a Pacemaker for leader rotation. Key insight: by adding one extra round compared to PBFT, the leader can collect votes and produce a combined QC (Quorum Certificate) with O($n$) messages per phase rather than O($n^2$).

**Safety** relies on the two-chain rule: a block is safe to commit when it extends a two-chain of QCs (grandparent and parent both certified).

### Python: Simplified PBFT State Machine

The following implements PBFT's three-phase protocol for a single request, with view-change logic omitted for clarity. It demonstrates why $O(n^2)$ message complexity arises and how the prepare/commit certificates are formed.

```python
import hashlib
import json
from collections import defaultdict
from dataclasses import dataclass, field
from typing import Dict, List, Optional, Set, Tuple

@dataclass(frozen=True)
class PBFTMessage:
    phase: str         # 'PRE-PREPARE', 'PREPARE', 'COMMIT'
    view: int
    seq: int
    digest: str        # hash of request
    sender: int

class PBFTReplica:
    def __init__(self, replica_id: int, n_replicas: int):
        self.id = replica_id
        self.n = n_replicas
        self.f = (n_replicas - 1) // 3  # max Byzantine faults
        self.view = 0
        self.seq = 0

        # Message log: (phase, view, seq, digest) -> set of senders
        self.msg_log: Dict[Tuple, Set[int]] = defaultdict(set)
        # Requests we have executed
        self.executed: Set[str] = set()
        # The actual request payloads indexed by digest
        self.requests: Dict[str, str] = {}

    @property
    def is_primary(self) -> bool:
        return self.id == self.view % self.n

    def _digest(self, request: str) -> str:
        return hashlib.sha256(request.encode()).hexdigest()[:16]

    def handle_client_request(self, request: str) -> List[PBFTMessage]:
        """Primary only. Assign seq number and broadcast PRE-PREPARE."""
        assert self.is_primary, "Non-primary received client request directly"
        self.seq += 1
        d = self._digest(request)
        self.requests[d] = request
        msg = PBFTMessage('PRE-PREPARE', self.view, self.seq, d, self.id)
        print(f"[Replica {self.id}] Sending PRE-PREPARE(v={self.view}, "
              f"n={self.seq}, d={d[:8]})")
        return [msg]  # broadcast to all replicas

    def handle_message(self, msg: PBFTMessage,
                        request: Optional[str] = None) -> List[PBFTMessage]:
        """
        Process an incoming PBFT message. Returns messages to broadcast.
        """
        outgoing: List[PBFTMessage] = []

        if msg.phase == 'PRE-PREPARE':
            if msg.view != self.view:
                return []
            if request:
                self.requests[msg.digest] = request
            # Non-primary: accept pre-prepare, broadcast PREPARE
            key = (msg.phase, msg.view, msg.seq, msg.digest)
            self.msg_log[key].add(msg.sender)
            prepare = PBFTMessage('PREPARE', self.view, msg.seq, msg.digest, self.id)
            outgoing.append(prepare)
            print(f"[Replica {self.id}] Sending PREPARE(v={msg.view}, "
                  f"n={msg.seq}, d={msg.digest[:8]})")

        elif msg.phase == 'PREPARE':
            key = (msg.phase, msg.view, msg.seq, msg.digest)
            self.msg_log[key].add(msg.sender)
            # Prepared certificate: 2f matching PREPAREs
            if len(self.msg_log[key]) >= 2 * self.f:
                commit_key = ('PREPARE_CERT', msg.view, msg.seq, msg.digest)
                if commit_key not in self.msg_log or not self.msg_log[commit_key]:
                    self.msg_log[commit_key].add(self.id)  # mark as prepared
                    commit = PBFTMessage('COMMIT', self.view, msg.seq,
                                         msg.digest, self.id)
                    outgoing.append(commit)
                    print(f"[Replica {self.id}] Prepared! Sending COMMIT("
                          f"v={msg.view}, n={msg.seq}, d={msg.digest[:8]})")

        elif msg.phase == 'COMMIT':
            key = (msg.phase, msg.view, msg.seq, msg.digest)
            self.msg_log[key].add(msg.sender)
            # Commit certificate: 2f+1 COMMITs
            if (len(self.msg_log[key]) >= 2 * self.f + 1
                    and msg.digest not in self.executed):
                self.executed.add(msg.digest)
                request_text = self.requests.get(msg.digest, '?')
                print(f"[Replica {self.id}] COMMITTED request: {request_text!r} "
                      f"(seq={msg.seq})")

        return outgoing


def simulate_pbft():
    """
    4 replicas (f=1), one request, no failures.
    Demonstrates message flow: PRE-PREPARE → PREPARE (x3) → COMMIT (x4).
    Total messages: 1 + 3 + 4 = O(n^2) = 8 (for n=4).
    """
    print("\n" + "="*60)
    print("PBFT SIMULATION: 4 replicas (f=1)")
    print("="*60)
    n = 4
    replicas = [PBFTReplica(i, n) for i in range(n)]
    primary = replicas[0]  # view=0, primary=replica 0

    request = "TRANSFER: Alice -> Bob, $100"

    # Primary handles request → PRE-PREPARE
    pp_msgs = primary.handle_client_request(request)
    request_digest = pp_msgs[0].digest

    # Broadcast PRE-PREPARE to all non-primaries
    prepare_msgs = []
    for replica in replicas[1:]:
        outgoing = replica.handle_message(pp_msgs[0], request)
        prepare_msgs.extend(outgoing)

    # All replicas handle all PREPAREs (n-1 PREPAREs each, O(n^2) total)
    commit_msgs = []
    for prepare in prepare_msgs:
        for replica in replicas:
            if replica.id != prepare.sender:
                outgoing = replica.handle_message(prepare)
                commit_msgs.extend(outgoing)

    # All replicas handle all COMMITs
    for commit in commit_msgs:
        for replica in replicas:
            if replica.id != commit.sender:
                replica.handle_message(commit)

    print(f"\nTotal PREPARE messages: {len(prepare_msgs)}")
    print(f"Total COMMIT messages:  {len(commit_msgs)}")
    print(f"All replicas committed: "
          f"{all(request_digest in r.executed for r in replicas)}")


if __name__ == '__main__':
    simulate_pbft()
```

**Why O(n²).** Each of the $n$ replicas broadcasts PREPARE to all $n$ replicas: $n(n-1)$ messages. Then each broadcasts COMMIT: another $n(n-1)$. Total is $O(n^2)$. For $n=100$ replicas, that is ~10,000 messages per request — this is why PBFT is impractical beyond ~20 nodes.

### HotStuff: O(n) via Threshold Signatures

HotStuff's key insight: instead of each replica broadcasting its vote to all others (O(n²)), all replicas send their vote (a BLS partial signature) to the **leader only**. The leader aggregates $2f+1$ partial signatures into a single **Quorum Certificate (QC)** of constant size, then broadcasts this single QC to all replicas (O(n) total).

```python
# Pseudocode for HotStuff QC chaining (conceptual — real BLS crypto omitted)

@dataclass
class Block:
    height: int
    parent_qc: Optional['QuorumCert']  # QC for the parent block
    payload: str

@dataclass
class QuorumCert:
    block_height: int
    combined_signature: bytes  # aggregated from 2f+1 partial sigs
    signers: List[int]

class HotStuffReplica:
    """
    HotStuff uses a 3-chain commit rule:
    A block B is SAFE to vote on if:
      - B extends the highest QC known (locks)
    A block B is COMMITTED when:
      - B has a QC (b.qc exists)
      - B's parent has a QC (b.parent_qc)
      - B's grandparent has a QC (b.parent_qc.block has a parent_qc)
    i.e., when there is a chain of 3 consecutive certified blocks.
    This 3-chain rule gives HotStuff its linear communication — 
    compared to PBFT's 2-phase (prepare+commit), HotStuff adds one 
    extra round but eliminates the quadratic broadcast.
    """
    def __init__(self, replica_id: int, n: int):
        self.id = replica_id
        self.f = (n - 1) // 3
        self.locked_qc: Optional[QuorumCert] = None  # highest known QC
        self.committed_height: int = -1

    def safe_to_vote(self, block: Block) -> bool:
        """Liveness rule: vote if block extends our locked QC or is higher."""
        if self.locked_qc is None:
            return True
        return (block.parent_qc is not None and
                block.parent_qc.block_height >= self.locked_qc.block_height)

    def on_receive_qc(self, qc: QuorumCert, block: Block,
                       grandparent_block: Optional[Block]):
        """Update lock and check for commit."""
        if self.locked_qc is None or qc.block_height > self.locked_qc.block_height:
            self.locked_qc = qc

        # Commit rule: three consecutive QC'd blocks
        if (grandparent_block is not None
                and grandparent_block.height == block.height - 2
                and block.parent_qc is not None
                and grandparent_block.height == block.parent_qc.block_height - 1):
            if grandparent_block.height > self.committed_height:
                self.committed_height = grandparent_block.height
                print(f"Replica {self.id}: Committed block at height "
                      f"{grandparent_block.height}: {grandparent_block.payload!r}")
```

### Tendermint (Buchman et al. 2016, used in Cosmos)

Tendermint is a BFT consensus protocol with **fast finality** — once a block is committed, it is permanent (no forks, unlike Nakamoto consensus).

Protocol: each height has rounds. In each round, the proposer broadcasts a block. Validators go through `PROPOSE → PREVOTE → PRECOMMIT`. Committing requires $2f+1$ precommits (a **commit certificate**). If no commit, move to next round.

**Round timeout.** Tendermint is **partially synchronous**: the timeout for each round starts small and doubles on failure, ensuring eventual termination when the network stabilizes.

**Lock mechanism.** A validator locks on a value when it sees $2f+1$ prevotes for it. Locked validators only prevote for the value they're locked on, or a newer value with a valid unlock proof. This prevents equivocation.

**Accountability.** Unlike Raft, Tendermint can cryptographically identify Byzantine validators (equivocation proofs), enabling slashing in proof-of-stake systems. If a validator signs two conflicting prevotes for the same height and round, both signatures serve as proof of misbehavior.

```python
from enum import Enum

class TendermintStep(Enum):
    PROPOSE = 'propose'
    PREVOTE = 'prevote'
    PRECOMMIT = 'precommit'

@dataclass
class TendermintState:
    """Per-height, per-round state for a Tendermint validator."""
    height: int
    round: int = 0
    step: TendermintStep = TendermintStep.PROPOSE
    locked_value: Optional[str] = None
    locked_round: int = -1
    valid_value: Optional[str] = None   # highest prevoted value
    valid_round: int = -1
    prevotes: Dict[int, Dict[str, int]] = field(default_factory=dict)   # round -> value -> count
    precommits: Dict[int, Dict[str, int]] = field(default_factory=dict) # round -> value -> count

    def add_prevote(self, rnd: int, value: str, f: int) -> Optional[str]:
        """Returns value if 2f+1 prevotes reached."""
        self.prevotes.setdefault(rnd, {}).setdefault(value, 0)
        self.prevotes[rnd][value] += 1
        if self.prevotes[rnd][value] >= 2 * f + 1:
            return value
        return None

    def add_precommit(self, rnd: int, value: str, f: int) -> Optional[str]:
        """Returns value if 2f+1 precommits reached (block committed)."""
        self.precommits.setdefault(rnd, {}).setdefault(value, 0)
        self.precommits[rnd][value] += 1
        if self.precommits[rnd][value] >= 2 * f + 1:
            return value
        return None
```

---

## 6. What Textbooks Don't Tell You

### Pre-vote Extension

In vanilla Raft, a partitioned node increments its term on every election timeout. When it reconnects, its high term forces the current leader to step down — even though the partitioned node's log is stale and it could never win an election.

**Pre-vote** (Ongaro's dissertation §9.6): before incrementing term and becoming a candidate, a node sends `PreVote` RPCs. Other nodes respond positively only if they have not heard from a leader recently *and* the candidate's log is up-to-date. Only if the node would win a pre-vote does it proceed with a real election.

This prevents term inflation and unnecessary leader disruptions — critical for production stability.

### Leader Stickiness

After a leadership change, followers may receive heartbeats from the old leader (delayed network packets) and immediately trigger a new election. **Leader stickiness**: a follower rejects `RequestVote` if it has received a heartbeat from any node recently (within the election timeout), even if the candidate has a higher term.

This heuristic prevents election thrashing while keeping safety intact.

### Read Linearizability

A naive read from a Raft leader is *not* linearizable. The leader might be deposed (a new leader elected in a partition) and not know it. Its stale reads would violate linearizability.

Three approaches:

1. **Quorum reads:** Route reads through the Raft log (append a read barrier, commit it, then read). Fully linearizable, high latency.

2. **Lease reads (etcd default):** Leader holds a time-bounded lease. It will not lose leadership before `leaseStart + leaseDuration`. Reads during the lease are linearizable without consensus round trips. **Danger:** requires bounded clock skew between nodes. If clocks drift more than the lease duration, a deposed leader may serve stale reads.

3. **Read index (etcd's hybrid):** Leader records its current `commitIndex`. Before serving a read, it sends a heartbeat round to confirm it's still the leader. If it gets majority acknowledgments, it waits until `applyIndex >= recordedCommitIndex` and then serves the read. No log entry needed, but requires one network round trip.

### Disk fsync Requirement

Raft's safety proofs assume that `currentTerm`, `votedFor`, and log entries are written to **durable storage** before sending any RPC response. Specifically:

- If a node crashes after voting but before fsync, on restart it may vote again (two votes for different candidates in the same term) — violating Election Safety.
- If log entries are not fsynced before acknowledging AppendEntries, a crash-recovery may lose committed entries.

In practice, fdatasync or O_DSYNC must be used. This is one of the most common production bugs: developers skip fsync for performance and observe subtle data loss that only manifests on hardware failure.

### TXN ID Wraparound and Epoch Numbers

Long-running systems (PostgreSQL, Spanner) must handle identifier wraparound. PostgreSQL's 32-bit transaction IDs wrap after ~2 billion transactions; a wraparound causes rows to appear in the future.

In distributed consensus, **epoch numbers** (equivalent to Raft terms) can exhaust if stored in 32-bit integers on very busy clusters. Production systems use 64-bit counters. Additionally, ballot numbers in Paxos must be globally unique; typical encoding is `(sequence_number << 8) | node_id`, but this limits node count or sequence length. Epoch transitions must be carefully fenced to prevent old-epoch messages from interfering.

### Clock Skew: Quantifying the Danger

Lease-based reads are safe only if the leader's clock does not drift significantly relative to its followers. Let's quantify exactly when things break.

```python
import time
import statistics
from dataclasses import dataclass
from typing import List

@dataclass
class ClockReading:
    """Simulates what a node believes the current time is."""
    node_id: int
    true_time: float    # actual time (ground truth)
    skew_ms: float      # node's clock is ahead by this many milliseconds

    @property
    def local_time(self) -> float:
        return self.true_time + self.skew_ms / 1000.0


def analyze_lease_safety(
    lease_duration_ms: float,
    clock_skew_ms: float,
    rtt_ms: float
) -> dict:
    """
    Determines whether a lease of `lease_duration_ms` is safe given
    known clock skew and network RTT.

    A leader L holds a lease granted at real time T_grant for duration D.
    L believes it holds the lease until T_grant + D (by L's clock).
    But if L's clock is AHEAD by `skew_ms`, L's "T_grant + D" in real time
    is actually T_grant + D - skew_ms.

    Meanwhile, a new leader can be elected after followers stop receiving
    heartbeats for their election timeout. The minimum time for a new leader
    to send a heartbeat that reaches followers is:
      election_timeout + heartbeat_interval + network_rtt

    For safety: lease must expire before new leader can assert leadership.
    """
    # Maximum safe lease duration
    # New leader cannot be elected until at minimum: election_timeout + rtt
    # But leader's perceived lease end is ahead by skew_ms
    # So effective lease = lease_duration - clock_skew
    effective_lease_ms = lease_duration_ms - clock_skew_ms

    # Round trip adds latency to when leader gets confirmation
    network_overhead_ms = rtt_ms

    # Safe if effective lease is positive AND less than network round trip
    # (ensures we don't serve reads after leadership is lost)
    safe = effective_lease_ms > 0 and effective_lease_ms > network_overhead_ms

    return {
        'lease_duration_ms': lease_duration_ms,
        'clock_skew_ms': clock_skew_ms,
        'effective_lease_ms': effective_lease_ms,
        'network_rtt_ms': rtt_ms,
        'safe': safe,
        'recommendation': (
            'SAFE: lease expires before clock drift causes staleness'
            if safe else
            f'UNSAFE: {clock_skew_ms}ms skew eats into {lease_duration_ms}ms lease '
            f'leaving only {effective_lease_ms:.1f}ms effective window'
        )
    }


def clock_skew_scenarios():
    scenarios = [
        # (lease_ms, skew_ms, rtt_ms, description)
        (5000, 50,  5,    "etcd default: 5s lease, 50ms NTP skew, 5ms LAN RTT"),
        (5000, 200, 5,    "Degraded NTP: 200ms skew"),
        (5000, 5001,5,    "Clock jump (VM live migration): skew > lease"),
        (1000, 50,  5,    "Short lease (1s): more frequent renewal, less skew risk"),
        (5000, 50,  4999, "High latency WAN (5s RTT): effective window too small"),
    ]
    print("\n=== Clock Skew Safety Analysis for Lease Reads ===\n")
    for lease, skew, rtt, desc in scenarios:
        result = analyze_lease_safety(lease, skew, rtt)
        status = "✓" if result['safe'] else "✗"
        print(f"{status} {desc}")
        print(f"  Effective lease: {result['effective_lease_ms']:.0f}ms  "
              f"→  {result['recommendation']}\n")
```

**Practical takeaway.** etcd's default lease is 5 seconds with a 500ms heartbeat interval. NTP clock skew is typically <10ms on well-administered hosts, so the effective lease window is ~4.99 seconds — safe. But VM live migration can cause clock jumps of seconds; cloud providers (AWS, GCP) publish guidelines for NTP discipline in VMs that run distributed databases.

### Write-Ahead Log (WAL) Anatomy

Understanding how Raft's durable state is actually persisted on disk is critical. The WAL is not just an append-only file — it must handle partial writes and corruption.

```python
import struct
import hashlib
import os
from typing import Iterator

# WAL record format:
# [4 bytes: CRC32] [4 bytes: record_length] [record_length bytes: data]
# CRC covers the record_length + data fields.
# This allows detecting partial writes (truncated records on crash).

WAL_MAGIC = b'RAFT'
WAL_VERSION = 1

@dataclass
class WALRecord:
    record_type: int   # 0=entry, 1=hard_state, 2=snapshot_metadata
    data: bytes

    def encode(self) -> bytes:
        payload = struct.pack('>B', self.record_type) + self.data
        length = len(payload)
        crc = int(hashlib.crc32(struct.pack('>I', length) + payload) & 0xFFFFFFFF)
        return struct.pack('>II', crc, length) + payload

    @classmethod
    def decode(cls, buf: bytes, offset: int) -> Tuple['WALRecord', int]:
        if offset + 8 > len(buf):
            raise EOFError("Truncated WAL record header")
        crc, length = struct.unpack('>II', buf[offset:offset+8])
        if offset + 8 + length > len(buf):
            raise EOFError(f"Truncated WAL record body at offset {offset}")
        payload = buf[offset+8:offset+8+length]
        actual_crc = int(hashlib.crc32(buf[offset+4:offset+8+length]) & 0xFFFFFFFF)
        if actual_crc != crc:
            raise ValueError(f"WAL CRC mismatch at offset {offset}: "
                             f"expected {crc:#010x}, got {actual_crc:#010x}")
        return cls(record_type=payload[0], data=payload[1:]), offset + 8 + length


class WAL:
    """
    Crash-safe write-ahead log.
    Key invariant: data is durable (on disk) BEFORE we respond to any RPC.
    """
    def __init__(self, path: str):
        self.path = path
        self._fd = open(path, 'ab+')  # append + read

    def append(self, record: WALRecord):
        encoded = record.encode()
        self._fd.write(encoded)
        # CRITICAL: fsync before returning. Without this, a crash can
        # lose the record even though write() returned successfully
        # (OS may buffer the write in page cache).
        self._fd.flush()
        os.fdatasync(self._fd.fileno())

    def read_all(self) -> Iterator[WALRecord]:
        """Read all valid records, stopping at first corruption."""
        with open(self.path, 'rb') as f:
            buf = f.read()
        offset = 0
        while offset < len(buf):
            try:
                record, offset = WALRecord.decode(buf, offset)
                yield record
            except (EOFError, ValueError) as e:
                # Truncation at end is normal (crash mid-write).
                # Corruption in the middle is a bug.
                print(f"WAL scan stopped at offset {offset}: {e}")
                break

    def truncate_after(self, valid_offset: int):
        """Used during recovery to discard partial writes past valid_offset."""
        self._fd.close()
        with open(self.path, 'r+b') as f:
            f.truncate(valid_offset)
        self._fd = open(self.path, 'ab+')
```

### Partition Detection Heuristics

Raft's election timeout is the primary partition detector, but production systems add layered heuristics:

```python
import statistics

class PartitionDetector:
    """
    Tracks heartbeat latency to detect asymmetric partitions
    (where a leader can send but not receive, or vice versa).
    These are invisible to simple timeout-based detection.
    """
    def __init__(self, window_size: int = 20, p99_threshold_ms: float = 50.0):
        self.window_size = window_size
        self.threshold_ms = p99_threshold_ms
        self.latencies: Dict[int, List[float]] = {}  # peer_id -> recent RTTs

    def record_heartbeat_rtt(self, peer_id: int, rtt_ms: float):
        hist = self.latencies.setdefault(peer_id, [])
        hist.append(rtt_ms)
        if len(hist) > self.window_size:
            hist.pop(0)

    def is_suspicious(self, peer_id: int) -> bool:
        hist = self.latencies.get(peer_id, [])
        if len(hist) < 5:
            return False
        p99 = sorted(hist)[int(len(hist) * 0.99)]
        return p99 > self.threshold_ms

    def asymmetric_partition_score(self) -> Dict[int, float]:
        """
        Returns a score per peer indicating likelihood of one-way partition.
        High score = likely asymmetric partition (leader can send but peer
        cannot receive, or vice versa).
        This is used by some systems to proactively step down rather than
        waiting for election timeout to fire.
        """
        scores = {}
        all_p99s = []
        for peer_id, hist in self.latencies.items():
            if len(hist) >= 5:
                p99 = sorted(hist)[int(len(hist) * 0.99)]
                all_p99s.append(p99)
        if not all_p99s:
            return scores
        median_p99 = statistics.median(all_p99s)
        for peer_id, hist in self.latencies.items():
            if len(hist) >= 5:
                p99 = sorted(hist)[int(len(hist) * 0.99)]
                scores[peer_id] = p99 / (median_p99 + 0.001)
        return scores
```

---

## 7. Edge Cases & Failure Modes

### Split-Brain: 5-Node Partition

Consider a 5-node cluster: nodes {1,2,3,4,5}. Node 1 is leader at term 3.

**Partition occurs:** {1,2} cannot communicate with {3,4,5}.

**Partition {1,2}:**
- Node 1 remains leader but can only reach node 2 — 2 nodes out of 5, not a majority.
- Node 1 continues accepting client writes, appending to its log.
- **But**: it cannot commit anything because it cannot replicate to a majority. Appended entries remain uncommitted.
- Node 1 returns errors to clients attempting committed writes.

**Partition {3,4,5}:**
- Nodes 3, 4, 5 stop receiving heartbeats. Their election timeouts fire.
- Say Node 3 wins election at term 4 (it has a majority — 3 out of 5 nodes).
- Node 3 becomes leader and can commit new entries (it can reach 3/5 nodes).
- Clients routed to this partition see progress.

**Healing:**
- Node 1 receives Node 3's heartbeat with term 4 > 3. Node 1 steps down immediately.
- Node 2 similarly reverts to follower.
- Nodes 1 and 2 receive AppendEntries from Node 3 (term 4). Their logs may have uncommitted entries from term 3 that conflict. These are overwritten (because they were never committed — safety is preserved).

**Key insight:** The uncommitted entries on Node 1 are simply discarded. No client was told these were committed (the protocol refused to commit without a majority). Safety holds because no committed value is ever lost. Liveness was sacrificed during the partition — the {1,2} partition was unavailable for writes.

### Log Divergence Recovery

When a new leader is elected, it must reconcile divergent follower logs. The leader uses `nextIndex[peer]` (initially set to `len(leader.log)`) and decrements it on each rejected AppendEntries until finding the point of divergence.

**Optimization (Fast Backup, §5.3 appendix):** Instead of decrementing by 1, the follower returns the term of the conflicting entry and the first index it stored for that term. The leader can skip back to that term boundary in one round trip, reducing catch-up time from O(divergence) to O(terms).

### Leader Crash Mid-Commit

Scenario: Leader appends entry $e$ to its log and sends AppendEntries to all followers. Majority acknowledges but before the leader can send the next heartbeat (which would carry the updated `leaderCommit`), the leader crashes.

**What survives:** Entry $e$ is in the log of a majority of nodes. When a new leader is elected (it must win from the majority that has $e$, by Election Restriction), it will eventually commit $e$ indirectly by committing an entry from its own term (Log Matching ensures $e$ is replicated wherever the new entry is).

**What doesn't survive:** The original clients' connection to the old leader is lost. They don't know if their request was committed. They must retry — idempotency is the client's responsibility. This is why Raft clients must use idempotent operations or include client-side deduplication IDs.

---

## 8. Hard Exercise: Trace Through a Complex Partition

### Setup

5-node Raft cluster (nodes N1–N5). N1 is leader at term 3. Log state before partition:

```
N1 (leader): [(1,a), (1,b), (2,c), (3,d)]
N2:          [(1,a), (1,b), (2,c), (3,d)]
N3:          [(1,a), (1,b), (2,c)       ]
N4:          [(1,a), (1,b), (2,c)       ]
N5:          [(1,a), (1,b)              ]
```

Entry `(3,d)` is replicated to N1 and N2, but N1 has NOT yet committed it (only 2/5 nodes have it — not a majority).

**Partition:** {N1, N2} | {N3, N4, N5}

N3 wins election at term 4 (it has a majority — 3 nodes). N3's log: `[(1,a), (1,b), (2,c)]` — this satisfies the Election Restriction because N3's `lastLogTerm=2` and N5's is 1, so N3 beats N5; N3 and N4 are tied, and the first to time out wins.

N3 appends and commits `(4,e)` and `(4,f)`:

```
N3 (leader, term 4): [(1,a), (1,b), (2,c), (4,e), (4,f)]
N4:                  [(1,a), (1,b), (2,c), (4,e), (4,f)]
N5:                  [(1,a), (1,b), (2,c), (4,e), (4,f)]
```

`commitIndex = 4` (index of `(4,f)`) on N3, N4, N5.

**Partition heals.**

N1 and N2 receive AppendEntries from N3 (term 4 > 3). Both step down.

N3 sends AppendEntries with `prevLogIndex=2, prevLogTerm=2` (to send entries starting at index 3). N1 and N2 have `log[2] = (2,c)` — matches. They accept entries `(4,e), (4,f)` at indices 3 and 4. **Critically**: the conflicting entry `(3,d)` at index 3 on N1/N2 is deleted (it conflicts with `(4,e)` at the same index) and replaced.

**Final logs (all nodes):**
```
N1: [(1,a), (1,b), (2,c), (4,e), (4,f)]  commitIndex=4
N2: [(1,a), (1,b), (2,c), (4,e), (4,f)]  commitIndex=4
N3: [(1,a), (1,b), (2,c), (4,e), (4,f)]  commitIndex=4
N4: [(1,a), (1,b), (2,c), (4,e), (4,f)]  commitIndex=4
N5: [(1,a), (1,b), (2,c), (4,e), (4,f)]  commitIndex=4
```

**Entries that survive:** `(1,a), (1,b), (2,c), (4,e), (4,f)`  
**Entry that does not survive:** `(3,d)` — it was overwritten.

**Is this safe?** Yes. `(3,d)` was never committed — N1 had it but only N2 acknowledged it (2/5 nodes, not a majority). N1 never told any client that `(3,d)` was committed. Safety (Agreement) is not violated.

**Why by Leader Completeness?**  
Leader Completeness states: if an entry is committed in term $t$, every future leader has it. Since `(3,d)` was *never committed*, Leader Completeness says nothing about it — it is free to be overwritten.

**Proof that `(4,e)` and `(4,f)` can never be overwritten:**  
They were committed at term 4 with a majority {N3, N4, N5}. Any future leader must win votes from a majority. Any majority overlaps with {N3, N4, N5} in at least one node. That node has `lastLogTerm=4`. For a future leader to beat it, the leader must have `lastLogTerm >= 4`. If `lastLogTerm=4`, the leader's log must be at least as long as 5 entries — so it contains `(4,e)` and `(4,f)`. By Log Matching, the leader has them. ∎

---

### Variant: What if (3,d) was committed before partition?

If `(3,d)` had been committed before the partition, N1 would have replicated it to a majority first. That means at least 3 nodes (including one from {N3, N4, N5}) would have `(3,d)`.

Suppose N3 and N4 have `[(1,a), (1,b), (2,c), (3,d)]` when the partition forms.

Now in the election for term 4, N3's `lastLogTerm=3` beats N5's `lastLogTerm=1`. N3 is elected. N3 already has `(3,d)`.

N3 appends `(4,e)` and `(4,f)`. Final log:
```
[(1,a), (1,b), (2,c), (3,d), (4,e), (4,f)]
```

`(3,d)` is preserved — Leader Completeness guaranteed it.

When N1 and N2 reconnect, N3's AppendEntries consistency check succeeds immediately (N1/N2 already have entries through index 3, term 3, which matches). N1/N2 simply append `(4,e)` and `(4,f)`.

This is the design working as intended: committed entries are durable across leadership changes by the Leader Completeness theorem, proven via the Election Restriction.

---

## 9. Formal Verification of Consensus Protocols

Informal proofs of distributed algorithms are notoriously error-prone. Lamport's original Paxos paper had a subtle bug in the liveness argument. The formal verification community has developed tools that find these bugs mechanically.

### TLA+ (Temporal Logic of Actions)

TLA+ is a specification language for concurrent and distributed systems. It describes systems as state machines and verifies properties using model checking (TLC) or theorem proving (TLAPS).

**Raft's TLA+ specification** (written by Diego Ongaro, available at https://github.com/ongardie/raft.tla) encodes all node states and transitions. The model checker exhaustively explores all possible orderings of messages up to a small bound (typically 3–5 nodes, up to 4 terms).

**Key properties verified in Raft.tla:**

```tla
(* Election Safety: at most one leader per term *)
ElectionSafety ==
    \A i, j \in Server :
        (/\ state[i] = Leader
         /\ state[j] = Leader
         /\ currentTerm[i] = currentTerm[j])
        => i = j

(* Leader Completeness: committed entries appear in future leaders *)
LeaderCompleteness ==
    \A i \in Server :
        state[i] = Leader =>
            \A v \in committed :
                v.term <= currentTerm[i] =>
                    \E idx \in DOMAIN log[i] :
                        /\ log[i][idx].term = v.term
                        /\ log[i][idx].cmd  = v.cmd

(* Log Matching: same (index, term) implies same prefix *)
LogMatching ==
    \A i, j \in Server :
        \A n \in DOMAIN log[i] :
            n \in DOMAIN log[j] =>
                log[i][n].term = log[j][n].term =>
                    SubSeq(log[i], 1, n) = SubSeq(log[j], 1, n)

(* State Machine Safety: applied values are identical across nodes *)
StateMachineSafety ==
    \A i, j \in Server :
        \A n \in 1..commitIndex[i] :
            n \in 1..commitIndex[j] =>
                log[i][n] = log[j][n]
```

The TLC model checker for a 3-node Raft cluster with 3 terms explores approximately 10 million states. Running `tlc Raft.tla -config Raft.cfg` takes ~30 minutes on a modern laptop and produces a counterexample if any invariant is violated.

### Python-Encoded Invariant Checking

We can run lightweight invariant checks on our Python simulation at each step:

```python
class ConsensusInvariantChecker:
    """
    Runtime invariant checker. Attach to a cluster and call check()
    at each simulation step to catch violations immediately.
    
    These match the TLA+ invariants above, translated to Python.
    """
    def __init__(self, nodes: List[RaftNode]):
        self.nodes = nodes
        self.violations: List[str] = []

    def check(self) -> bool:
        """Returns True if all invariants hold, False otherwise."""
        ok = True
        ok &= self._check_election_safety()
        ok &= self._check_log_matching()
        ok &= self._check_commit_monotonicity()
        return ok

    def _check_election_safety(self) -> bool:
        """At most one leader per term."""
        leaders_by_term: Dict[int, List[int]] = {}
        for node in self.nodes:
            s = node.status()
            if s['state'] == 'leader':
                t = s['term']
                leaders_by_term.setdefault(t, []).append(s['node_id'])

        ok = True
        for term, leaders in leaders_by_term.items():
            if len(leaders) > 1:
                msg = (f"ELECTION SAFETY VIOLATION: "
                       f"Multiple leaders in term {term}: {leaders}")
                self.violations.append(msg)
                print(f"*** {msg} ***")
                ok = False
        return ok

    def _check_log_matching(self) -> bool:
        """If two nodes have an entry with the same (index, term), prefixes match."""
        ok = True
        statuses = [n.status() for n in self.nodes]
        for i in range(len(statuses)):
            for j in range(i + 1, len(statuses)):
                log_i = statuses[i]['log']
                log_j = statuses[j]['log']
                min_len = min(len(log_i), len(log_j))
                for idx in range(min_len):
                    if log_i[idx].term == log_j[idx].term:
                        # Prefixes must match
                        for k in range(idx + 1):
                            if log_i[k] != log_j[k]:
                                msg = (
                                    f"LOG MATCHING VIOLATION: "
                                    f"nodes {statuses[i]['node_id']} and "
                                    f"{statuses[j]['node_id']} agree at "
                                    f"index={idx} term={log_i[idx].term} "
                                    f"but differ at index={k}: "
                                    f"{log_i[k]} vs {log_j[k]}"
                                )
                                self.violations.append(msg)
                                print(f"*** {msg} ***")
                                ok = False
        return ok

    def _check_commit_monotonicity(self) -> bool:
        """Commit index never decreases."""
        ok = True
        if not hasattr(self, '_prev_commits'):
            self._prev_commits = {}
        for node in self.nodes:
            s = node.status()
            nid = s['node_id']
            ci = s['commit_index']
            prev = self._prev_commits.get(nid, -1)
            if ci < prev:
                msg = (f"COMMIT MONOTONICITY VIOLATION: "
                       f"node {nid} commit_index went from {prev} to {ci}")
                self.violations.append(msg)
                print(f"*** {msg} ***")
                ok = False
            self._prev_commits[nid] = ci
        return ok

    def summary(self):
        if not self.violations:
            print("All invariants held throughout simulation.")
        else:
            print(f"{len(self.violations)} invariant violation(s) found:")
            for v in self.violations:
                print(f"  - {v}")


def simulate_with_invariant_checking():
    """Run the Raft cluster with invariant checking at every step."""
    nodes: List[RaftNode] = [RaftNode.__new__(RaftNode) for _ in range(5)]
    for i, node in enumerate(nodes):
        node.__init__(i, [n for n in nodes if n is not node])

    checker = ConsensusInvariantChecker(nodes)

    # Wait for election, periodically checking invariants
    for _ in range(50):
        time.sleep(0.1)
        checker.check()

    leader = next((n for n in nodes if n.status()['state'] == 'leader'), None)
    if leader:
        for cmd in ['SET a=1', 'SET b=2', 'SET c=3']:
            leader.client_request(cmd)
            time.sleep(0.05)
            checker.check()

    time.sleep(0.5)
    checker.check()
    checker.summary()
```

### Coq Proofs of Consensus

The **Verdi** project (University of Washington) provides mechanically verified implementations of distributed systems in Coq. Their `raft-verified` library proves Raft's Log Matching Property and State Machine Safety using the Coq proof assistant.

The key insight from Verdi: proofs about distributed algorithms decompose into:
1. **Network semantics** (how messages are delivered — the environment)
2. **Protocol correctness** (invariant preservation across each state transition)
3. **Composition** (the protocol + environment → property)

This decomposition mirrors how we reason about Raft in this document, but made machine-checkable. The Verdi Raft proof is ~10,000 lines of Coq and took ~2 person-years to complete — which gives you a sense of how non-trivial these proofs are even when the algorithm appears "simple."

---

## 10. Testing Distributed Consensus

Distributed systems are notoriously difficult to test because the bugs live in rare interleavings of concurrent events. Standard unit tests cannot exercise them. This section covers the methodology used to find real bugs in production systems.

### Fault Injection Framework

```python
import random
import functools
from typing import Callable, Any
from enum import Enum

class FaultType(Enum):
    MESSAGE_DROP    = 'drop'
    MESSAGE_DELAY   = 'delay'
    NODE_CRASH      = 'crash'
    NODE_RESTART    = 'restart'
    PARTITION       = 'partition'
    CLOCK_JUMP      = 'clock_jump'
    DISK_SLOW       = 'disk_slow'

@dataclass
class FaultSpec:
    fault_type: FaultType
    probability: float       # 0.0 to 1.0
    duration_ms: float = 0   # for delay/partition/slow
    target_node: Optional[int] = None  # None = any node
    target_peer: Optional[int] = None  # for partition/directed faults

class FaultInjector:
    """
    Wraps a RaftNode cluster and randomly injects faults according to a
    schedule. This is the core of model-based testing for consensus systems.
    """
    def __init__(self, nodes: List[RaftNode], faults: List[FaultSpec],
                 seed: int = 42):
        self.nodes = nodes
        self.faults = faults
        self.rng = random.Random(seed)
        self._active_partitions: Set[Tuple[int, int]] = set()
        self._crashed: Set[int] = set()

    def step(self):
        """Apply one random fault from the fault schedule."""
        for spec in self.faults:
            if self.rng.random() < spec.probability:
                self._apply_fault(spec)

    def _apply_fault(self, spec: FaultSpec):
        target = (spec.target_node if spec.target_node is not None
                  else self.rng.randint(0, len(self.nodes) - 1))
        node = self.nodes[target]

        if spec.fault_type == FaultType.PARTITION:
            peer = (spec.target_peer if spec.target_peer is not None
                    else self.rng.randint(0, len(self.nodes) - 1))
            if peer != target:
                node.partitioned_from.add(peer)
                self.nodes[peer].partitioned_from.add(target)
                print(f"FAULT: Partition {target} ↔ {peer}")

        elif spec.fault_type == FaultType.NODE_CRASH:
            self._crashed.add(target)
            node.partitioned_from = set(range(len(self.nodes))) - {target}
            print(f"FAULT: Crash node {target}")

        elif spec.fault_type == FaultType.NODE_RESTART:
            if target in self._crashed:
                node.partitioned_from = set()
                self._crashed.discard(target)
                print(f"FAULT: Restart node {target}")

    def heal_all(self):
        for node in self.nodes:
            node.partitioned_from = set()
        self._crashed.clear()


class LinearizabilityChecker:
    """
    Simplified Knossos-style linearizability checker for a KV store.
    
    Records a history of operations (invocations and responses), then
    checks whether any linearization of those operations exists that is
    consistent with sequential KV store semantics.
    
    Full Knossos uses WGL (Wing-Gong-Lamport) algorithm; this is a 
    simplified version for educational purposes.
    """

    @dataclass
    class Op:
        kind: str           # 'write' or 'read'
        key: str
        value: Optional[str]
        invoke_time: float
        return_time: float
        result: Optional[str]  # for reads: the value returned

    def __init__(self):
        self.history: List['LinearizabilityChecker.Op'] = []

    def record_write(self, key: str, value: str,
                     invoke_t: float, return_t: float):
        self.history.append(self.Op('write', key, value, invoke_t, return_t, None))

    def record_read(self, key: str, invoke_t: float, return_t: float,
                    result: Optional[str]):
        self.history.append(self.Op('read', key, None, invoke_t, return_t, result))

    def check(self) -> Tuple[bool, Optional[str]]:
        """
        Check linearizability by trying all valid linearization orderings.
        An op A can be ordered before op B if A.return_time <= B.invoke_time
        (real-time order) or they overlap (concurrent, any order valid).
        
        This brute-force approach is O(n!) in operation count — use
        only for small histories. Production checkers use smarter 
        state-space search (Porcupine in Go, Knossos in Clojure).
        """
        from itertools import permutations

        ops = sorted(self.history, key=lambda o: o.invoke_time)

        def valid_order(perm) -> bool:
            """Check if a permutation of ops is a valid linearization."""
            # Check real-time ordering constraint
            for i in range(len(perm)):
                for j in range(i + 1, len(perm)):
                    # perm[i] must come before perm[j] in linearization
                    # This is valid only if perm[i].return_time <= perm[j].invoke_time
                    # (otherwise they overlap and any order is OK)
                    if perm[j].return_time < perm[i].invoke_time:
                        return False  # perm[j] strictly before perm[i] in real time

            # Simulate the KV store against this ordering
            state: Dict[str, Optional[str]] = {}
            for op in perm:
                if op.kind == 'write':
                    state[op.key] = op.value
                elif op.kind == 'read':
                    expected = state.get(op.key)
                    if op.result != expected:
                        return False
            return True

        if len(ops) > 8:
            return False, "History too large for brute-force check (use Porcupine)"

        for perm in permutations(ops):
            if valid_order(perm):
                return True, None

        # Find the first violating read to produce a useful error message
        for op in ops:
            if op.kind == 'read':
                return False, (
                    f"Linearizability violation: read({op.key}) returned "
                    f"{op.result!r} at [{op.invoke_time:.3f}, {op.return_time:.3f}]"
                )
        return False, "Linearizability violated (no valid linearization found)"


def run_fault_injection_test():
    """
    Integration test: 5-node cluster under random faults.
    Records all client operations, then checks linearizability.
    """
    print("\n=== Fault Injection + Linearizability Test ===\n")

    nodes: List[RaftNode] = [RaftNode.__new__(RaftNode) for _ in range(5)]
    for i, node in enumerate(nodes):
        node.__init__(i, [n for n in nodes if n is not node])

    checker = LinearizabilityChecker()

    fault_schedule = [
        FaultSpec(FaultType.PARTITION, probability=0.05),
        FaultSpec(FaultType.NODE_CRASH, probability=0.02),
        FaultSpec(FaultType.NODE_RESTART, probability=0.03),
    ]
    injector = FaultInjector(nodes, fault_schedule, seed=12345)

    # Wait for initial election
    time.sleep(0.5)

    kv_store_truth = {}  # what we expect (from leader's perspective)
    n_ops = 0

    for iteration in range(30):
        time.sleep(0.05)
        injector.step()

        # Find current leader
        leader = next((n for n in nodes if n.status()['state'] == 'leader'), None)
        if leader is None:
            continue

        # Submit a write
        key, value = 'x', f'v{iteration}'
        t0 = time.time()
        success = leader.client_request(f'SET {key}={value}')
        t1 = time.time()

        if success:
            checker.record_write(key, value, t0, t1)
            n_ops += 1

    injector.heal_all()
    time.sleep(0.3)

    ok, err = checker.check()
    print(f"Operations recorded: {n_ops}")
    print(f"Linearizability check: {'PASS' if ok else 'FAIL'}")
    if err:
        print(f"Error: {err}")
```

### Jepsen Methodology

Jepsen (Kyle Kingsbury, 2013–present) is a framework for testing distributed systems by running real clusters under network partitions and checking operation histories for consistency violations. It has found bugs in:

- **etcd** (stale reads during leader elections before read-index was implemented)
- **Cassandra** (lost acknowledged writes under partition)
- **MongoDB** (rollback of acknowledged writes)
- **CockroachDB** (serializable isolation violations)
- **Zookeeper** (stale reads from followers)

**Jepsen's approach:**
1. Start a real cluster (5 nodes, Docker or VMs)
2. Run client operations (reads and writes) concurrently from multiple threads
3. Inject network partitions via `iptables` or `tc netem` at random
4. Record the complete history of invocations and responses with real timestamps
5. Run Knossos (a model checker) against the history to verify linearizability

**Key insight.** The checker does not need to know *why* an operation returned a value — it only checks whether *some* sequential history exists that is consistent with the operation semantics. A system passes Jepsen if no violation is found in 1M+ operations under aggressive faults. A single violation (e.g., a read returning a value that was never written, or a stale read after a write was acknowledged) constitutes a linearizability bug.

---

## 10. Consistency Models Deep Dive

Consensus algorithms exist to enforce consistency guarantees. But "consistency" is not a single property — there is a strict hierarchy of models, and choosing the wrong one leads to subtle correctness bugs.

### The Consistency Hierarchy

From strongest to weakest:

```
Linearizability (Herlihy & Wing 1990)
  ↓
Sequential Consistency (Lamport 1979)
  ↓
Causal Consistency (Ahamad et al. 1995)
  ↓
PRAM / Pipeline RAM
  ↓
Eventual Consistency (Vogels 2009)
```

### Linearizability

**Definition.** An execution is linearizable if there exists a total ordering of all operations such that:
1. The ordering is consistent with the real-time order of non-overlapping operations.
2. Each operation appears to execute atomically at some point between its invocation and response.

Linearizability is the **gold standard** for distributed systems. It is what clients expect from a single machine: reads see the latest write, and there is a single consistent view of history.

**Example of non-linearizable execution:**

```
Client A:  write(x=1) ----[returns OK]----
Client B:               read(x) --[returns 0]--  read(x) --[returns 1]--
```

If B's first read overlaps with A's write and returns 0, that is fine (concurrent). But if B's second read (entirely after A's write completed) returns 0, that violates linearizability — the write is "missing" from a completed operation's view.

**How Raft provides linearizability.** All writes go through the leader's log. Reads via read-index (or quorum) ensure the leader has applied all committed writes before responding. This linearizes the history at the log's commit order.

### Sequential Consistency

**Definition.** An execution is sequentially consistent if there is a total order of operations that:
1. Is consistent with each *individual process's* program order.
2. Each operation appears to execute atomically.

Unlike linearizability, sequential consistency does not require respecting real-time order between different processes. Two clients can observe operations in different orders as long as each client's own operations appear in order.

**Example where sequential consistency holds but linearizability does not:**

```
Process 1: write(x=1)
Process 2: write(y=1)
Process 3: read(y=1), read(x=0)  ← sees y before x
Process 4: read(x=1), read(y=0)  ← sees x before y
```

Processes 3 and 4 disagree on the order of writes to x and y. This violates linearizability (there's no single total order consistent with all reads) but satisfies sequential consistency (each process's program order is respected).

CPUs with out-of-order execution and store buffers implement sequential consistency within a core but weaker models (TSO, PSO) across cores — which is why concurrent programs need memory barriers.

### Causal Consistency

**Definition.** If operation A causally precedes operation B (A happened-before B in the Lamport sense), then every process that sees B must also have seen A before B.

Causally related operations are ordered; concurrent operations may be observed in any order.

**Practical implementation.** Vector clocks or version vectors track causal dependencies. MongoDB's causal sessions use cluster time + operationTime to enforce this. Cassandra's lightweight transactions (using Paxos) provide linearizability for specific keys; ordinary reads/writes are only eventually consistent.

```python
from typing import Dict, List, Tuple

class VectorClock:
    """Tracks causality for causal consistency enforcement."""

    def __init__(self, node_id: str, all_nodes: List[str]):
        self.node_id = node_id
        self.clock: Dict[str, int] = {n: 0 for n in all_nodes}

    def tick(self) -> Dict[str, int]:
        self.clock[self.node_id] += 1
        return dict(self.clock)

    def update(self, received: Dict[str, int]):
        """Merge a received vector clock (happens-before relationship)."""
        for node, ts in received.items():
            self.clock[node] = max(self.clock.get(node, 0), ts)
        self.clock[self.node_id] += 1  # local event triggered by receipt

    def happens_before(self, vc_a: Dict[str, int], vc_b: Dict[str, int]) -> bool:
        """Returns True if vc_a happened-before vc_b (A causally precedes B)."""
        return (
            all(vc_a.get(n, 0) <= vc_b.get(n, 0) for n in set(vc_a) | set(vc_b))
            and any(vc_a.get(n, 0) < vc_b.get(n, 0) for n in set(vc_a) | set(vc_b))
        )

    def concurrent(self, vc_a: Dict[str, int], vc_b: Dict[str, int]) -> bool:
        return (
            not self.happens_before(vc_a, vc_b)
            and not self.happens_before(vc_b, vc_a)
        )


def causal_consistency_example():
    nodes = ['A', 'B', 'C']
    vc_a = VectorClock('A', nodes)
    vc_b = VectorClock('B', nodes)
    vc_c = VectorClock('C', nodes)

    # A writes x=1, gets timestamp t1
    t1 = vc_a.tick()
    print(f"A writes x=1 at {t1}")

    # A sends message to B (B learns about A's write causally)
    vc_b.update(t1)
    t2 = vc_b.tick()
    print(f"B writes y=1 (after seeing A's write) at {t2}")

    # C receives B's message — must have seen A's write too
    vc_c.update(t2)
    print(f"C's clock after receiving B's message: {vc_c.clock}")
    print(f"C has seen A's write: {vc_c.clock['A'] >= t1['A']}")

    # Two independent writes are concurrent (no causal order)
    vc_x = VectorClock('X', nodes)
    vc_y = VectorClock('Y', nodes)
    tx = vc_x.tick()
    ty = vc_y.tick()
    print(f"\nConcurrent? {vc_a.concurrent(tx, ty)}")  # True

if __name__ == '__main__':
    causal_consistency_example()
```

### Eventual Consistency

**Definition.** If no new updates are made, all replicas eventually converge to the same state.

This is the weakest useful model. It makes no guarantee about when convergence occurs or what any individual read returns. Systems like Cassandra (default), DynamoDB, and Riak use eventual consistency for their highest-throughput paths.

**Conflict resolution.** When concurrent writes conflict, the system must pick a winner:
- **Last Write Wins (LWW):** Use wall clock or Lamport timestamp. Loses data silently.
- **CRDTs (Conflict-free Replicated Data Types):** Data structures (counters, sets, maps) where all concurrent updates can be merged deterministically. No coordination needed.
- **Application-level merge:** Client provides a merge function. Used by CouchDB.

### Why This Matters for Consensus

Paxos and Raft provide **linearizability** — the strongest model — which is why they require $2f+1$ nodes and majority quorums. Weaker consistency models (causal, eventual) can be implemented without consensus, with much higher availability and lower latency. The choice is always an engineering tradeoff: use consensus where you need ordering guarantees, and weaker models where you can tolerate staleness.

---

## 10. Consensus in Production Systems

### etcd

etcd is the most widely deployed Raft implementation (it is the backing store for Kubernetes). Key implementation decisions:

**Storage backend.** etcd v3 uses bbolt (a B+tree key-value store) as its persistent storage. Log entries are written to bbolt with `fdatasync` before AppendEntries returns. Snapshots are taken periodically to truncate the log.

**Read index for linearizable reads.** etcd implements the read-index approach: on a read, the leader appends the current `commitIndex`, sends a round of heartbeats to confirm leadership, then waits until `appliedIndex >= savedCommitIndex` before serving the read. This avoids Raft log entries for reads while guaranteeing linearizability.

**Watch mechanism.** etcd v3 provides a server-side watch API: clients subscribe to key ranges and receive a stream of events. Each event carries the revision at which it occurred. This revision is the Raft `commitIndex` at the time the write was applied — a monotonically increasing counter that clients use to resume watches after reconnection without missing events.

**Lease mechanism for distributed locks.** etcd leases are TTL-bounded. A client acquires a lease (attached to a key), keeps it alive by sending KeepAlive RPCs, and if the client dies, the lease expires and the key is deleted. This is the basis of Kubernetes leader election.

### CockroachDB

CockroachDB uses multi-Raft across shards. Each range (64 MB by default) has its own Raft group.

**Transaction protocol.** CockroachDB uses a variant of 2PC (two-phase commit) layered on top of Raft. The transaction coordinator writes intents (provisional values) to the Raft log, then commits by writing a transaction record. Readers that encounter an intent check the transaction record to determine whether to use the intent or the previous value.

**HLC (Hybrid Logical Clocks).** CockroachDB uses HLCs to provide causality guarantees across transactions and ranges. An HLC is `(physical_time, logical_counter)`. Physical time is bounded by NTP; the logical counter breaks ties. This enables reading at a consistent snapshot across all ranges without a global lock manager.

**Follower reads.** CockroachDB v2.0+ supports reading from followers (non-leaders) with a bounded staleness guarantee. The follower knows the leader's `closedTimestamp` — a timestamp before which all writes are committed. Reads with `AS OF SYSTEM TIME` older than `closedTimestamp` can be served locally, reducing tail latency and enabling geo-distributed reads.

### Apache ZooKeeper (ZAB Protocol)

ZooKeeper uses ZAB (ZooKeeper Atomic Broadcast), a protocol similar to but distinct from Paxos.

**Zxid.** Each transaction has a 64-bit zxid = `(epoch, counter)`. Epoch increments on leader change; counter increments on each write. This provides a total order across leader epochs.

**Phase 1 (Discovery/Sync).** New leader discovers the latest committed transaction, syncs all followers to agree on a common prefix, then begins serving writes.

**Phase 2 (Broadcast).** Leader proposes updates (PROPOSAL), followers ACK, leader COMMITs when quorum ACKs. Unlike Paxos, the leader does not need a separate phase to establish its authority per transaction — authority is established once per epoch.

**Key difference from Raft.** ZAB provides "primary order" — a stronger property than Raft's Log Matching. The leader ensures that if it has broadcast a proposal, all future leaders will also have it. This allows followers to serve reads directly (with appropriate fencing) because they are guaranteed to have all committed writes in order.

### Google Spanner

Spanner uses Paxos for each shard (called a "directory"). What makes Spanner unique is its use of TrueTime to provide **external consistency** (a distributed form of linearizability) across shards without a global coordinator.

**TrueTime API.** TrueTime returns `[earliest, latest]` bounds on the current time, guaranteed by GPS receivers and atomic clocks. The uncertainty interval is typically 1–7ms.

**Commit wait.** Before committing a read-write transaction, Spanner waits until `TT.now().earliest > commit_timestamp`. This ensures that the commit timestamp is in the past before the transaction is visible — any subsequent transaction in the real world that reads this data will have a start time strictly after the commit time, preserving external consistency.

**Read-only transactions.** Spanner provides snapshot reads at a specific timestamp. Because TrueTime bounds are known, a read at timestamp $T$ is safe to serve from any replica that has applied all writes up to $T$ — no locks needed, no consensus round trips.

---

## 11. Multi-Raft and Sharding

A single Raft group can handle ~100k operations/second on modern hardware. Real databases need much more. The solution is **multi-Raft**: partition data across many Raft groups, each responsible for a subset of the keyspace.

### Range Partitioning

```python
from dataclasses import dataclass, field
from typing import List, Optional, Dict, Tuple
import bisect

@dataclass
class Range:
    """A contiguous key range managed by one Raft group."""
    start_key: str
    end_key: str  # exclusive
    raft_group_id: int
    leader_node: int

class RangeRouter:
    """
    Routes client requests to the correct Raft group based on key.
    In production (CockroachDB, TiKV), this metadata is itself stored
    in a Raft group (the 'meta range').
    """
    def __init__(self):
        self.ranges: List[Range] = []
        self._start_keys: List[str] = []

    def add_range(self, r: Range):
        idx = bisect.bisect_left(self._start_keys, r.start_key)
        self.ranges.insert(idx, r)
        self._start_keys.insert(idx, r.start_key)

    def route(self, key: str) -> Optional[Range]:
        idx = bisect.bisect_right(self._start_keys, key) - 1
        if idx < 0:
            return None
        r = self.ranges[idx]
        if r.start_key <= key < r.end_key:
            return r
        return None

    def split_range(self, range_id: int, split_key: str) -> Tuple[Range, Range]:
        """
        Range split: one Raft group becomes two.
        The split is itself a Raft-committed operation in the meta range.
        Steps:
          1. Leader of old range commits a split intent.
          2. Left range [start, split_key) retains old group.
          3. Right range [split_key, end) bootstraps a new Raft group,
             copying leader's snapshot as initial state.
          4. Meta range is updated atomically.
        """
        r = next(x for x in self.ranges if x.raft_group_id == range_id)
        left = Range(r.start_key, split_key, r.raft_group_id, r.leader_node)
        right = Range(split_key, r.end_key, r.raft_group_id + 1000, r.leader_node)
        self.ranges.remove(r)
        self._start_keys.remove(r.start_key)
        self.add_range(left)
        self.add_range(right)
        return left, right


def demonstrate_routing():
    router = RangeRouter()
    router.add_range(Range('a', 'm', raft_group_id=1, leader_node=0))
    router.add_range(Range('m', 'z', raft_group_id=2, leader_node=3))

    for key in ['apple', 'mango', 'zebra', 'banana']:
        r = router.route(key)
        print(f"key={key!r:10} → group={r.raft_group_id if r else None}")

    print("\nSplitting group 1 at 'f'...")
    left, right = router.split_range(1, 'f')
    print(f"Left:  [{left.start_key}, {left.end_key})  group={left.raft_group_id}")
    print(f"Right: [{right.start_key}, {right.end_key})  group={right.raft_group_id}")

    for key in ['apple', 'fig', 'grape']:
        r = router.route(key)
        print(f"key={key!r:10} → group={r.raft_group_id if r else None}")
```

### Cross-Shard Transactions

When a transaction touches multiple Raft groups, you need distributed commit. The standard approach is **2PC (Two-Phase Commit)** with the transaction coordinator being a Raft-replicated state machine itself:

1. **Prepare phase.** Coordinator sends `Prepare(txn_id, writes)` to each involved shard's Raft group. Each shard votes YES (locking the rows) or NO (conflict).
2. **Commit phase.** If all vote YES, coordinator writes a durable commit record to *its own* Raft group, then sends `Commit(txn_id)` to all shards. Shards apply the writes and release locks.

**Failure modes in 2PC + Raft:**
- Coordinator crashes after writing commit record but before notifying shards → shards are locked. Recovery: any node can read the coordinator's Raft log, find the committed record, and send the Commit messages. This is called *distributed recovery* and is why the coordinator's state must be Raft-replicated.
- A shard's Raft group loses quorum mid-transaction → the prepare vote never completes → coordinator times out and aborts. The abort is also Raft-replicated.

---

## 12. Performance & Optimization

### Batching and Pipelining

The naive Raft implementation processes one client request per consensus round. This is catastrophically slow. Production systems batch:

```python
import queue
import threading
import time
from typing import List, Callable
from dataclasses import dataclass

@dataclass
class PendingWrite:
    command: str
    callback: Callable[[bool], None]

class BatchingRaftLeader:
    """
    Collects client writes for up to `batch_interval_ms` milliseconds,
    then submits them as a single AppendEntries batch. This amortizes the
    network RTT across many client requests.
    """
    def __init__(self, batch_interval_ms: float = 2.0, max_batch_size: int = 500):
        self.batch_interval = batch_interval_ms / 1000.0
        self.max_batch_size = max_batch_size
        self._pending: List[PendingWrite] = []
        self._lock = threading.Lock()
        self._batch_event = threading.Event()
        self._flush_thread = threading.Thread(target=self._flush_loop, daemon=True)
        self._flush_thread.start()

    def submit(self, command: str, callback: Callable[[bool], None]):
        with self._lock:
            self._pending.append(PendingWrite(command, callback))
            if len(self._pending) >= self.max_batch_size:
                self._batch_event.set()

    def _flush_loop(self):
        while True:
            self._batch_event.wait(timeout=self.batch_interval)
            self._batch_event.clear()
            with self._lock:
                batch = self._pending[:]
                self._pending.clear()
            if batch:
                self._commit_batch(batch)

    def _commit_batch(self, batch: List[PendingWrite]):
        # In real implementation: append all commands to log as a single
        # AppendEntries RPC, wait for quorum, then invoke all callbacks.
        print(f"Committing batch of {len(batch)} commands: "
              f"{[w.command for w in batch]}")
        for write in batch:
            write.callback(True)
```

**Throughput impact.** A single Raft round trip over a LAN (1ms RTT) yields ~1,000 ops/sec without batching. With batching 500 commands per round, the same latency yields ~500,000 ops/sec. etcd achieves ~200k writes/sec on typical hardware with batching.

### Log Compaction and Snapshots

Without truncation, the Raft log grows unboundedly. Compaction takes a snapshot of the state machine at index $I$ and deletes log entries $\leq I$.

```python
@dataclass
class Snapshot:
    last_included_index: int
    last_included_term: int
    state: dict  # serialized state machine

class SnapshotManager:
    def __init__(self, snapshot_threshold: int = 10_000):
        self.threshold = snapshot_threshold
        self.last_snapshot: Optional[Snapshot] = None

    def should_snapshot(self, log_size: int, last_snapshot_index: int,
                        applied_index: int) -> bool:
        entries_since_snapshot = applied_index - last_snapshot_index
        return entries_since_snapshot >= self.threshold

    def take_snapshot(self, state_machine: dict,
                      applied_index: int, applied_term: int) -> Snapshot:
        snap = Snapshot(
            last_included_index=applied_index,
            last_included_term=applied_term,
            state=dict(state_machine)
        )
        self.last_snapshot = snap
        print(f"Snapshot taken at index={applied_index}, "
              f"state has {len(state_machine)} keys")
        return snap

    def install_snapshot(self, snap: Snapshot, raft_node) -> bool:
        """
        Used when a follower is so far behind that log entries have been
        discarded. Leader sends InstallSnapshot RPC instead of AppendEntries.
        Follower must:
          1. Save snapshot to durable storage (fsync).
          2. Discard any existing log entries conflicting with snapshot.
          3. Reset state machine to snapshot state.
          4. Set commitIndex = lastApplied = snap.last_included_index.
        """
        print(f"Installing snapshot from index {snap.last_included_index}")
        # In production: write to disk before applying to state machine
        self.last_snapshot = snap
        return True
```

### Network Optimizations

**Write coalescing.** Multiple AppendEntries in flight simultaneously (pipelining). The leader does not wait for acknowledgment of batch $k$ before sending batch $k+1$. This keeps the network pipe full.

**Parallel disk writes.** The leader can send AppendEntries to all followers while simultaneously fsyncing to its own disk — these are independent. The commit can proceed as soon as $f$ followers ACK and the leader's fsync completes, whichever comes last.

**Compression.** Log entries for large payloads (e.g., Kubernetes etcd storing pod specs) benefit from LZ4/zstd compression before transmission. etcd 3.5+ compresses snapshots.

**Flow control.** If followers fall behind, the leader must limit in-flight entries to prevent memory exhaustion. etcd uses a `maxInflightMsgs` parameter (default 256) to cap outstanding AppendEntries per follower.

---

## 13. Glossary

| Term | Definition |
|------|-----------|
| **Bivalent configuration** | A system state from which both decision values 0 and 1 are still reachable. Key concept in FLP proof. |
| **Ballot / Proposal number** | Globally unique, monotonically increasing identifier for a Paxos round. Typically encoded as `(seq << bits) | node_id`. |
| **Commit index** | In Raft, the highest log index known to be committed (replicated to a majority). |
| **Dueling proposers** | Livelock in Paxos where two proposers repeatedly invalidate each other's prepare phase. |
| **Election restriction** | Raft rule: a node only votes for a candidate whose log is at least as up-to-date as the voter's. Ensures Leader Completeness. |
| **Epoch / Term** | Monotonically increasing integer identifying a leadership tenure. Lamport calls it epoch; Raft calls it term; ZAB calls it epoch. |
| **External consistency** | Property where transactions appear to execute in real-time order globally. Stronger than linearizability for multi-partition systems. Used by Spanner. |
| **FLP** | Fischer-Lynch-Paterson (1985). Proved no deterministic consensus algorithm works in an asynchronous system if one process may crash. |
| **fsync** | System call forcing OS buffer cache to disk. Mandatory for Raft's durability guarantee — never skip. |
| **HLC** | Hybrid Logical Clock: `(physical_time, logical_counter)`. Provides causality tracking while staying close to wall-clock time. |
| **Joint consensus** | Raft membership change approach using a transitional config that requires majorities from both old and new configs. |
| **Learner** | Paxos role that receives the chosen value but does not vote. In Raft, learners are read-only replicas added before becoming full voters. |
| **Linearizability** | Strongest consistency model: every operation appears to execute atomically at some point between invocation and response, consistent with real-time order. |
| **Log compaction** | Replacing a prefix of the Raft log with a snapshot to bound log size. |
| **Log Matching Property** | Raft invariant: if two logs share an entry at `(index, term)`, all preceding entries are identical. |
| **Pacemaker** | HotStuff component responsible for leader rotation and liveness. Equivalent to Raft's election timeout mechanism. |
| **Pre-vote** | Raft extension where a node solicits hypothetical votes before actually starting an election, preventing term inflation from partitioned nodes. |
| **Quorum certificate (QC)** | HotStuff/Tendermint term for a threshold signature aggregating $2f+1$ votes. Compact proof that a quorum agreed. |
| **Read index** | etcd's approach to linearizable reads: record `commitIndex`, confirm leadership via heartbeat, serve read once `applyIndex >= savedCommitIndex`. |
| **Safety property** | A property violated by a finite execution prefix. "Nothing bad ever happens." |
| **Split-brain** | Scenario where two partitioned subsets each believe they are the primary. Raft prevents this by requiring majority quorums. |
| **State machine replication** | Running identical deterministic state machines on multiple nodes, all receiving the same sequence of commands via consensus. |
| **TrueTime** | Google Spanner's API providing `[earliest, latest]` wall-clock bounds, implemented with GPS + atomic clocks. Enables external consistency. |
| **Two Generals** | Impossibility result: two processes cannot achieve guaranteed agreement over an unreliable channel in finite time. |
| **Vector clock** | Per-node integer counters tracking causality: `vc[i]` = number of events node $i$ knows about. Used to detect concurrent vs. causally ordered events. |
| **View / View change** | PBFT term for the equivalent of Raft's leader epoch + leader election. |
| **Zxid** | ZooKeeper transaction ID: `(epoch, counter)`. Provides total ordering of all writes across leader changes. |

---

## 14. Common Interview Questions — Rigorous Answers

These are questions asked at systems engineering interviews at companies running large-scale distributed databases. Shallow answers fail; the expected depth is approximately what is covered in this document.

---

**Q1. Can two Raft leaders exist simultaneously?**

**A.** Two leaders *from different terms* can coexist transiently — one is stale and does not know it is deposed. Safety is preserved because the stale leader cannot commit anything: it needs a majority quorum to commit, and the new leader has already won a majority that will reject the stale leader's AppendEntries (their terms are higher). The stale leader's entries are uncommitted and will be overwritten when it receives the new leader's messages.

Two leaders *in the same term* are impossible. A leader requires a majority vote (more than half of nodes). If node A is leader in term $t$, it received votes from majority $Q_A$. For node B to also become leader in term $t$, it needs votes from majority $Q_B$. Since $|Q_A| + |Q_B| > N$, they must share at least one voter. That voter can only vote once per term (enforced by persisted `votedFor`). Contradiction — B cannot get a majority without the voter who already voted for A.

---

**Q2. Why does Raft's commitment rule say "only commit entries from the current term"? Give the exact failure scenario if you violate this.**

**A.** This is Figure 8 from the Ongaro dissertation. The failure scenario:

```
Timeline (5 nodes: S1-S5):
Term 1: S1 is leader. Replicates (1,a) to S1,S2,S3. Committed.
Term 2: S1 is leader. Appends (2,b) to S1, S2. Crashes.
Term 3: S5 is leader (voted by S3,S4,S5 whose logs show only (1,a)).
         S5 appends (3,c) to S5 only. Crashes.
Term 4: S1 recovers. S1 is leader (voted by S1,S2,S3 — S2 has (2,b)).
         S1 replicates (2,b) to S3. Now S1,S2,S3 have (2,b).
         **If S1 now commits (2,b) directly:**
         S1 then crashes. S5 wins election (term 5) because:
           S5's log = [(1,a), (3,c)] — lastLogTerm=3 > S3's lastLogTerm=2
           S5 beats S3 and S4, becomes leader.
           S5 overwrites (2,b) with (3,c) at index 2.
         Now (2,b) was "committed" (S1 told clients) but is gone. SAFETY VIOLATED.

Fix: S1 must first replicate a term-4 entry (e.g., a no-op) to a majority.
     Once the no-op is committed, (2,b) is committed transitively via Log Matching.
     S5 cannot win any future election because term-4 entries on S1,S2,S3 
     make their logs more up-to-date than S5's (3,c).
```

---

**Q3. What is the minimum number of nodes needed to tolerate 2 simultaneous failures in Raft? In PBFT?**

**A.**  
- **Raft (crash fault tolerance):** Need $2f+1$ nodes. For $f=2$: **5 nodes**.
- **PBFT (Byzantine fault tolerance):** Need $3f+1$ nodes. For $f=2$: **7 nodes**.

The extra node in BFT accounts for the fact that a Byzantine faulty node actively lies. In a $3f+1$ system with quorum size $2f+1$: any two quorums share at least $f+1$ nodes, of which at most $f$ are Byzantine, so at least 1 honest node is in every quorum intersection.

---

**Q4. You observe that your etcd cluster has "high tail latency on writes during leader elections." What are the root causes and how do you fix each?**

**A.** Three root causes:

1. **Election timeout fires unnecessarily.** Symptom: P99 write latency spikes every ~150–300ms. Cause: a follower's heartbeat timer fires before the leader's heartbeat arrives. Fix: tune `heartbeat-interval` to be well below `election-timeout` (ratio ≥ 5:1). etcd defaults to 100ms heartbeat, 1000ms election timeout.

2. **Stale leader serving reads during election.** If using lease-based reads, the old leader may serve stale data while the election is ongoing. Fix: switch to read-index reads, or reduce lease duration so it expires before the new leader is established.

3. **Log catch-up after election.** New leader sends AppendEntries to bring followers up-to-date. If followers are far behind (due to previous crash or slow disk), catch-up takes many round trips. Fix: ensure `max_inflight_msgs` is high enough to pipeline catch-up (default 256 in etcd); use snapshot transfer for very lagged followers.

4. **No-op commit latency.** Every new leader appends a no-op and must commit it before serving reads. This adds one consensus round (typically 2× RTT) to every leadership change. Fix: you cannot eliminate this, but you can minimize by placing all nodes in the same AZ (low RTT).

---

**Q5. Explain why you cannot implement a distributed lock using a single Redis instance and what you need instead.**

**A.** A single Redis instance is a single point of failure and has no consensus mechanism. Specifically:

1. **No quorum.** If the Redis master crashes, a replica may be promoted. The replica may not have the latest keys (async replication → lost writes). A client that acquired a lock on the old master believes it holds the lock; a second client acquires the same lock on the new master. **Split-brain on the lock.**

2. **Expiry-clock drift.** Redis uses local clock for TTL expiry. If the Redis host's clock jumps forward (NTP correction, VM migration), locks expire prematurely. If it jumps backward, locks live longer than intended.

**Correct alternatives:**
- **Redlock (Redis cluster):** Acquire lock on $\lceil N/2 \rceil$ independent Redis instances. But Redlock is still debated (see Martin Kleppmann's "How to do distributed locking") — it does not handle GC pauses and clock skew correctly.
- **etcd leases:** Backed by Raft consensus. A lease is a Raft-committed object; its TTL is enforced by the etcd cluster, not a single process. Use `KeepAlive` RPCs to renew. If the client dies, the lease expires and the key is deleted — guaranteed, even if one etcd node crashes.
- **Zookeeper ephemeral nodes:** Ephemeral znodes are deleted when the client session expires, which is enforced by the ZAB consensus layer.

---

**Q6. A 3-node Raft cluster has nodes A (leader), B, C. A and B have committed entry (5, "SET x=1"). C has not received it. B and C are partitioned from A. What happens?**

**A.** Step by step:

1. A is leader (term 5). Entry `(5, "SET x=1")` is committed on A and B (majority quorum of 3 = 2).
2. Partition: B and C cannot reach A.
3. B and C do not receive heartbeats from A. Their election timers fire.
4. **Who can win the election?** B has the committed entry at term 5. C does not. B's `lastLogTerm=5`, C's `lastLogTerm < 5`. For C to vote for itself, B must grant a vote — but B will check C's `lastLogTerm`: if C's is less than 5, B rejects. C cannot win.
5. B wins the election (C votes for B since B's log is at least as up-to-date as C's). B becomes leader at term 6.
6. B replicates `(5, "SET x=1")` to C (via AppendEntries). C catches up.
7. B appends a no-op `(6, "NO-OP")` and commits it. Commit index on B and C advances.
8. Partition heals. A receives B's heartbeat (term 6 > 5). A steps down to follower.
9. A receives AppendEntries from B. A already has `(5, "SET x=1")` — Log Matching confirms it matches. A gets the no-op. All three nodes converge.

**The committed entry `(5, "SET x=1")` is preserved.** Safety holds.

---

**Q7. What is Linearizability and how does it differ from Serializability?**

**A.** These are frequently conflated.

**Linearizability** (Herlihy & Wing 1990): A property of *individual operations* on a single object. An operation appears to execute atomically at some instant between its invocation and response, consistent with real-time order. It is a *single-object* property about ordering in time.

**Serializability** (database theory): A property of *transactions* across *multiple objects*. A schedule of transactions is serializable if it is equivalent to some serial execution of those transactions. It makes no guarantee about real-time ordering — two transactions could execute "out of order" relative to wall clock as long as some serial order exists.

**Strict serializability = Linearizability + Serializability**: transactions appear to execute in real-time order AND as if one at a time. This is what Spanner provides via TrueTime.

**Example of the gap:**
- Session A: writes `x=1` at 10:00:00, then reads `y` at 10:00:01.
- Session B: writes `y=1` at 10:00:00.5, then reads `x` at 10:00:01.

A serializable database might order these as: [B writes y, A writes x, B reads x=1, A reads y=1] — consistent, serializable, but not with real-time order (B reads x before A wrote it in this serial order, even though A's write completed before B's read in wall clock time). Strict serializability would require B to see A's write.

Raft provides **linearizability** for individual key reads/writes (when using read-index). Transactions across multiple keys require an additional protocol (2PC, as in CockroachDB) to achieve serializability.

---

## 15. Research Frontiers (2019–2025)

### Shoal and Shoal++ (Meta, 2023)

DAG-based BFT consensus. Instead of a linear chain of blocks, validators build a Directed Acyclic Graph of proposals. Each vertex includes 2f+1 references to previous vertices, forming an implicit quorum certificate without explicit voting rounds. Shoal++ achieves sub-second latency at 50 validators by pipelining DAG construction with consensus rounds. Used in Meta's Diem successor and academic follow-ons.

### Mysticeti (Mysten Labs / Sui, 2023)

Uncertified DAG consensus with parallel commitment. Mysticeti eliminates the explicit certificate step by having validators commit vertices when they observe 2f+1 votes in the DAG structure itself. This reduces latency by ~40% vs. certified DAG protocols. Production deployed in the Sui blockchain.

### Twins Testing (Distributed Systems Testing, 2021)

An alternative to Jepsen that runs protocol testing without a real network. Each node is "twinned" — run twice with different message schedules — to systematically explore message orderings. The LibraBFT team used Twins to find bugs in HotStuff view-change that Jepsen did not catch, because Twins can explore orderings that are statistically rare even under random fault injection.

### Scalog and Compartmentalized Paxos (NSDI 2020–2021)

**Scalog** decouples ordering from consensus: a separate ordering layer (a gossip protocol) determines the global order of records, and consensus is used only to agree on epoch boundaries. This allows the ordering throughput to scale with the number of replicas rather than being bottlenecked by the Paxos leader.

**Compartmentalized MultiPaxos** parallelizes the Paxos bottleneck by separating: proposers (who assign slots), acceptors (who vote), and a separate "proxy leader" layer. This achieves near-linear throughput scaling by eliminating the leader bottleneck while preserving safety.

### Hotstuff-2 and Linear BFT (2023)

Jolteon (VMware Research) and Hotstuff-2 achieve BFT consensus in 2 phases (vs. 3 in original HotStuff) by leveraging a stronger signature scheme (threshold BLS) and a Pacemaker that provides stricter view synchrony. This further reduces commit latency by 33%.

### Dissecting Performance of BFT (2022)

Arun et al. ("Dissecting BFT Consensus: In Trusted Components We Trust") show that BFT systems with a Trusted Execution Environment (TEE, e.g., Intel SGX) can tolerate $f$ faults with only $2f+1$ replicas (same as crash-fault-tolerant systems) because the TEE prevents Byzantine behavior. This is BFT "for free" given hardware trust assumptions.

---

## Summary and Further Reading

Distributed consensus is not a solved problem in practice. The theory — FLP impossibility, quorum intersection, Leader Completeness — is tight and beautiful. The engineering gaps are where systems actually fail: missed fsyncs, clock skew under lease reads, term inflation from partitioned nodes, membership change races, no-op timing, and snapshot installation races.

The progression from theory to production looks like this:

```
FLP Impossibility
    ↓ (partial synchrony assumption)
Paxos / Raft (CFT)
    ↓ (adversarial nodes assumption)
PBFT / HotStuff / Tendermint (BFT)
    ↓ (scale assumption)
Multi-Raft / Compartmentalized Paxos / DAG-BFT
    ↓ (global consistency assumption)
Spanner TrueTime / HLC-based cross-shard transactions
    ↓ (verification requirement)
TLA+ model checking / Verdi Coq proofs / Jepsen testing
```

Each layer adds complexity and addresses a constraint that the previous layer assumed away. A production distributed database (CockroachDB, Spanner, TiDB) implements all layers simultaneously.

---

## Annotated Bibliography

**Foundational Theory**

- **Fischer, M.J., Lynch, N.A., Paterson, M.S. (1985).** "Impossibility of Distributed Consensus with One Faulty Process." *JACM 32(2), 374–382.* The FLP theorem. Read the original — it is 9 pages and surprisingly accessible. The bivalency argument in §3 is the key.

- **Lamport, L. (1978).** "Time, Clocks, and the Ordering of Events in a Distributed System." *CACM 21(7).* Defines happened-before and Lamport clocks. The foundation of causal consistency.

- **Dwork, C., Lynch, N., Stockmeyer, L. (1988).** "Consensus in the presence of partial synchrony." *JACM 35(2).* Defines the partial synchrony model that Paxos and Raft operate in. Defines GST (Global Stabilization Time). Required reading to understand *why* Raft can guarantee liveness even though FLP says you can't.

- **Chandra, T. & Toueg, S. (1996).** "Unreliable Failure Detectors for Reliable Distributed Systems." *JACM 43(2).* Introduces the failure detector abstraction. The weakest failure detector sufficient for consensus is $\Diamond W$. This is the theoretical justification for leader leases.

**Paxos**

- **Lamport, L. (1998).** "The Part-Time Parliament." *ACM TOCS 16(2).* The original Paxos paper, submitted in 1989, rejected, finally published in 1998. Reads like a description of a Greek parliament. The actual algorithm is in §2.

- **Lamport, L. (2001).** "Paxos Made Simple." *ACM SIGACT News 32(4).* Two-page distillation of Paxos. This is the canonical reference everyone cites. The multi-Paxos extension is in §3.

- **Chandra, T., Griesemer, R., Redstone, J. (2007).** "Paxos Made Live: An Engineering Perspective." *PODC.* How Google built Chubby on Paxos. Lists 15 engineering challenges not in Lamport's paper. The most important systems paper on Paxos.

- **Howard, H., Schwarzkopf, M., Madhavapeddy, A., Crowcroft, J. (2016).** "Flexible Paxos: Quorum Intersection Revisited." *arXiv:1608.06696.* Proves that Phase 1 and Phase 2 quorums only need to intersect each other — opens up asymmetric configurations for geo-distributed systems.

**Raft**

- **Ongaro, D. & Ousterhout, J. (2014).** "In Search of an Understandable Consensus Algorithm (Extended Version)." *USENIX ATC.* The Raft paper. The extended version (available at raft.github.io) has more detail on membership changes and the no-op entry.

- **Ongaro, D. (2014).** "Consensus: Bridging Theory and Practice." *PhD dissertation, Stanford.* The authoritative reference. Chapters 6 (membership changes), 7 (log compaction), and 9 (performance and extensions including pre-vote) are especially important for implementors.

**Byzantine Fault Tolerance**

- **Castro, M. & Liskov, B. (1999).** "Practical Byzantine Fault Tolerance." *OSDI.* The first BFT protocol suitable for real systems. O(n²) complexity is the price; the paper shows it can handle 1000s of req/s. The view-change protocol in §4.4 is complex but crucial.

- **Yin, M., Malkhi, D., Reiter, M.K., Gueta, G.G., Abraham, I. (2019).** "HotStuff: BFT Consensus with Linearity and Responsiveness." *PODC.* Linear communication via threshold signatures. The protocol description in §5 is self-contained. Used in Diem (Facebook's blockchain) as LibraBFT.

- **Buchman, E. (2016).** "Tendermint: Byzantine Fault Tolerance in the Age of Blockchains." *MSc thesis, University of Guelph.* Tendermint's design. The lock mechanism in §3.3 is the key to understanding why validators can safely precommit.

**Production Systems**

- **Corbett, J. et al. (2012).** "Spanner: Google's Globally Distributed Database." *OSDI.* TrueTime, external consistency, and globally distributed Paxos groups. §3 (TrueTime) and §4.1.2 (RW transactions) are the core.

- **Huang, J. et al. (2020).** "TiDB: A Raft-based HTAP Database." *VLDB.* How TiDB combines multi-Raft for OLTP with a columnar store (TiFlash) for OLAP, using Raft learner replicas to async-replicate to the analytical engine.

- **etcd contributors.** "etcd v3 Architecture." *etcd.io/docs.* The v3 design (gRPC, MVCC, leases, watches) is well-documented. The "learning" section explains why learner nodes were added (§ "Design" in etcd docs).

**Formal Verification**

- **Wilcox, J.R. et al. (2015).** "Verdi: A Framework for Implementing and Formally Verifying Distributed Systems." *PLDI.* The Coq framework for verified distributed systems. The raft-verified artifact includes ~10k lines of Coq proofs.

- **Lamport, L. (2002).** "Specifying Systems: The TLA+ Language and Tools for Hardware and Software Engineers." *Addison-Wesley.* The TLA+ book. Chapters 1–8 cover enough to write and check a Raft specification.

**Testing**

- **Kingsbury, K. (2013–present).** "Jepsen: Distributed systems safety research." *jepsen.io.* Analysis reports for etcd, MongoDB, CockroachDB, Cassandra, and 30+ other systems. Each report is a case study in finding real bugs. The etcd report (2014) is the most relevant to Raft.

- **Losa, G. et al. (2021).** "Twins: BFT Systems Made Robust." *DISC.* The Twins testing methodology that found HotStuff view-change bugs. A good complement to Jepsen for protocol-level testing.
