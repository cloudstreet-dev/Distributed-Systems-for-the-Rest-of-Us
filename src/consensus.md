# Consensus: How Nodes Agree on Anything at All

At the heart of many distributed systems problems is a deceptively simple question: **how do multiple nodes agree on a single value?**

This is the consensus problem. It appears whenever you need to:

- Elect a single leader from multiple candidates
- Decide which write wins when two nodes have conflicting values
- Commit a transaction that spans multiple nodes
- Append to a distributed log in a consistent order

Consensus is what makes distributed systems *correct* rather than merely *fast*. It's also what makes them slow, complicated, and expensive. Understanding consensus is understanding the boundary between what's possible in a distributed system and what isn't.

---

## Why Consensus Is Hard

Consider two nodes that need to agree on a value. The naive approach:

1. Node A proposes a value
2. Node B accepts the value
3. Both nodes now agree

What if the message from A to B gets lost? B never accepts. Now what? A doesn't know whether B received the proposal or not. A can resend, but what if the original message arrived after all, and B accepted it, and then B accepted the resent message too?

What if A crashes after sending the proposal but before receiving B's acceptance?

What if the network delivers A's message, B sends an acceptance, the acceptance gets lost, and then B crashes? A never hears from B. Does A proceed alone? If it does, and B comes back, they might disagree.

These scenarios escalate quickly. The impossibility result at the heart of this problem — called FLP (Fischer, Lynch, and Paterson, 1985) — says that in an asynchronous network where even one process can crash, you cannot guarantee that a consensus algorithm terminates.

This isn't a proof that consensus is impossible in practice. It's a proof that no algorithm can *guarantee* progress under *all* conditions. Real systems get around this by making reasonable assumptions about timing and failure rates.

---

## The Consensus Requirements

A correct consensus algorithm must satisfy:

**Agreement**: All correct nodes decide on the same value.

**Validity**: The decided value must have been proposed by some node (you can't just make up a value).

**Termination**: Every correct node eventually decides on a value (the algorithm doesn't run forever).

The FLP result says you can't guarantee all three in an asynchronous system with failures. In practice, this means algorithms relax the termination guarantee slightly: they guarantee termination unless the system is in a particularly bad state (lots of simultaneous failures, poor timing).

---

## Paxos: The Algorithm That Launched a Thousand Papers

Paxos was described by Leslie Lamport in 1989 (published 1998 after significant delay). It was the first practical consensus algorithm. It's also famously difficult to understand, implement correctly, and reason about.

Single-decree Paxos (consensus on one value) has two phases:

### Phase 1: Prepare

A **proposer** selects a proposal number N (larger than any it has used before) and sends a `Prepare(N)` to a majority of **acceptors**.

Each acceptor:
- If N is larger than any prepare it has seen, responds with `Promise(N)` and the highest-numbered proposal it has already accepted (if any)
- Otherwise, ignores the prepare

```
Proposer ──Prepare(N=5)──> Acceptor 1
Proposer ──Prepare(N=5)──> Acceptor 2  ← quorum (3 of 5 acceptors)
Proposer ──Prepare(N=5)──> Acceptor 3

Acceptor 1 ──Promise(N=5, accepted=none)──> Proposer
Acceptor 2 ──Promise(N=5, accepted=none)──> Proposer
Acceptor 3 ──Promise(N=5, accepted=none)──> Proposer
```

### Phase 2: Accept

If the proposer gets promises from a majority of acceptors, it sends `Accept(N, value)` to those acceptors. The value is:
- The value from the highest-numbered accepted proposal any acceptor reported, or
- The proposer's own value if no acceptor has accepted anything

Each acceptor:
- Accepts the proposal if it hasn't promised to ignore proposals with number ≥ N
- Notifies **learners** of the accepted value

When a majority of acceptors have accepted the same proposal number with the same value, consensus is reached.

### Why this is subtle

The proposal number juggling is not bureaucracy — it's safety. If a proposer fails midway through and a new proposer takes over, the promise mechanism ensures the new proposer can figure out whether any value was already accepted (by a previous majority) and carries it forward rather than overwriting it.

Multi-Paxos extends this for sequences of values (a log), which is what you actually need in practice. And this is where Paxos gets complicated quickly.

**The honest assessment**: Paxos is correct but implementing it in a production system is extraordinarily hard. Lamport himself said that every published variant he'd seen contained subtle bugs. Most production systems use Raft instead.

---

## Raft: Paxos for the Rest of Us

Raft was designed explicitly to be understandable. The paper is titled "In Search of an Understandable Consensus Algorithm." It achieves roughly the same guarantees as Multi-Paxos but through a cleaner decomposition.

Raft decomposes consensus into three sub-problems:

1. **Leader election**: Which node is the leader?
2. **Log replication**: The leader accepts entries and replicates them to followers
3. **Safety**: Guarantees that the right entries are in the right place

### Leader Election

Time in Raft is divided into **terms** — monotonically increasing integers. Each term begins with an election.

Nodes start as followers. If a follower doesn't hear from a leader within an election timeout (randomized to prevent ties), it becomes a **candidate** and starts an election.

```
            Term 1                 Term 2
         ─────────────────────────────────────────
         [   Leader   ]    [leader fails]
         [  Follower  ]             [  Leader  ]
         [  Follower  ]             [ Follower ]
                              ^ election happens here
```

A candidate sends `RequestVote` to all nodes. A node grants its vote if:
- It hasn't voted in this term yet
- The candidate's log is at least as up-to-date as the voter's log

If a candidate gets votes from a majority, it becomes the new leader and starts sending heartbeats. If the election times out (split vote), start again with a new term.

The randomized timeout is key: if all nodes started elections simultaneously, you'd always get a split vote. Randomization means one node usually starts first and wins before others wake up.

### Log Replication

The leader accepts writes, appends them to its **log**, and sends `AppendEntries` RPCs to followers to replicate the log.

```
Leader Log:  [1: set x=1] [2: set y=2] [3: set x=3]
                |               |               |
             replicate       replicate       replicate
                |               |               |
Follower Log: [1: set x=1] [2: set y=2] [3: set x=3]
```

A log entry is **committed** once the leader has replicated it to a majority of nodes. Once committed, it will persist regardless of future leader changes.

The leader tracks the **commitIndex** — the highest log entry known to be committed — and piggybacks this in AppendEntries. Followers apply committed entries to their state machines.

### Safety: Why Log Entries Don't Disappear

Raft's critical safety property: **if a log entry is committed at one term, no future leader will overwrite it.**

This is guaranteed by the voting rule: a candidate can only win an election if it has a log at least as up-to-date as a majority of nodes. Since committed entries exist on a majority, any winner must have them.

This means you never need to "fix" a follower's log by rolling back committed entries — you only need to find the point where the follower diverges from the leader and replicate forward from there.

### Why Raft Became the Default

Raft is used in:
- **etcd**: The key-value store at the heart of Kubernetes
- **CockroachDB**: Distributed SQL
- **TiDB**: Distributed database
- **Consul**: Service mesh and configuration
- Many others

It's not perfect (there are edge cases in Raft's log compaction and joint consensus phases), but it's understandable enough that implementations can be reasoned about and audited. The Raft paper includes a user study showing that students understood Raft significantly better than Paxos.

---

## Viewstamped Replication

An older algorithm (1988, predating Paxos) that's equivalent in power but less well-known. VR is arguably cleaner than Paxos and predates it. It influenced much of the later work on consensus. Worth knowing as context, though Raft is the practical choice today.

---

## Byzantine Fault Tolerant Consensus

Everything above assumes **crash-stop failures**: nodes either work correctly or stop responding. They don't lie, fabricate messages, or collude.

**Byzantine faults** are the harder case: nodes can behave arbitrarily — sending conflicting messages to different nodes, delaying messages strategically, pretending to accept and then reneging.

Byzantine fault tolerance (BFT) requires at least **3f+1** nodes to tolerate **f** Byzantine failures. With 4 nodes, you can tolerate 1 Byzantine node. With 7 nodes, you can tolerate 2.

The most well-known BFT algorithm is PBFT (Practical Byzantine Fault Tolerance, Castro and Liskov, 1999). It works but is expensive — O(n²) message complexity — and doesn't scale beyond tens of nodes.

BFT consensus is used in:
- **Blockchain systems**: Nodes are adversarial by assumption (different economic interests)
- **Safety-critical systems**: Aviation, nuclear, where a single faulty sensor can't cause a disaster
- **Systems crossing trust boundaries**: Multiple organizations running nodes with different incentives

For most application databases, Byzantine faults are not in scope. Your database replicas are operated by you — you trust them to be correct even if they crash. BFT matters when nodes are operated by parties that might act against your interests.

---

## Consensus in Practice: What This Means for You

### "Don't implement consensus yourself"

This is the advice. Consensus algorithms have subtle failure modes that appear only in specific timing conditions. Even the best implementers have gotten them wrong. Use a battle-tested implementation: etcd, Zookeeper, Consul.

### Leader election via external coordination

The most common pattern: use an external consensus service (Zookeeper, etcd, Consul) to perform leader election. Your application nodes try to acquire a lease. The lease holder is the leader. When the lease expires or the holder crashes, another node acquires it.

```python
# Pseudo-code for leader election via etcd
lease = etcd.grant_lease(ttl=10)  # 10-second lease
result = etcd.put_if_absent("/leader", my_node_id, lease=lease)

if result.succeeded:
    # We are the leader. Keep the lease alive.
    while is_leader:
        do_leader_work()
        etcd.keepalive(lease)  # Reset TTL before expiry
else:
    # Someone else is leader. Watch for changes.
    etcd.watch("/leader", callback=on_leader_change)
```

### Distributed transactions via Two-Phase Commit (2PC)

2PC is not a consensus algorithm but it uses similar principles to make a distributed transaction atomic.

**Phase 1 (Prepare)**: Coordinator asks all participants: "Are you ready to commit?"

**Phase 2 (Commit or Abort)**: If all respond "yes," coordinator sends "commit." If any respond "no" or time out, coordinator sends "abort."

```
Coordinator ──PREPARE──> Participant A
Coordinator ──PREPARE──> Participant B

Participant A ──READY──> Coordinator
Participant B ──READY──> Coordinator

Coordinator ──COMMIT──> Participant A
Coordinator ──COMMIT──> Participant B
```

The critical failure mode: if the coordinator crashes after sending PREPARE but before sending COMMIT, participants are stuck. They've locked resources and voted "yes" but don't know the final decision. This is the 2PC blocking problem.

Three-phase commit (3PC) attempts to solve this, but introduces other failure modes and isn't widely used in practice. Modern distributed databases (CockroachDB, Spanner) use Paxos/Raft for the commit decision instead of 2PC to avoid the blocking problem.

### The performance cost of consensus

Consensus requires at least one round-trip to a quorum. In a 3-node cluster with nodes in the same datacenter, this adds maybe 1-5ms per committed operation. In a geo-distributed cluster with nodes on different continents, a quorum round-trip is the speed of light plus processing — 100-300ms.

This is the fundamental tension between consistency and performance in distributed systems. Linearizable operations require consensus. Consensus requires network round-trips. Network round-trips take time. If you want consistency in a geo-distributed system, you're paying in milliseconds. Spanner's atomic clock approach reduces the coordination cost but doesn't eliminate it.

---

## The Aha Moment

Here's the thing that clicks when you really understand consensus: **a single-leader database with synchronous replication to at least one follower is already doing consensus**, informally. The leader is the proposer; the synchronous follower is the required quorum member; the write isn't committed until both have it.

The difference between "informal single-leader" consensus and Raft/Paxos consensus is what happens when the leader fails. Informal: you run a script or Pacemaker or some ops procedure to elect a new leader. Formal consensus algorithm: the nodes elect a new leader automatically, safely, without external intervention.

As your system grows and manual operations become impractical, the formal approach becomes necessary. That's why etcd is at the center of Kubernetes — Kubernetes can't afford to wait for an operator to manually promote a new leader when the current one dies.

---

The next chapter tackles something consensus relies on but we've been hand-waving: time. Specifically, the fact that distributed nodes don't share a clock, and the many interesting ways this causes problems.
