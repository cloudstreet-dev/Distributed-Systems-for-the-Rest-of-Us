# Network Partitions: When the Network Lies to You

The network partition is the defining failure mode of distributed systems. Not because it's the most common failure — it isn't — but because it's the most philosophically troubling, the most difficult to reason about, and the one that forces you to confront the hardest trade-offs.

A network partition is not "the network is down." It's more subtle: **two parts of your system are running and healthy, but they can't talk to each other.** Both sides are operational. Both sides are processing requests. Both sides believe they're correct. Neither side can tell whether the other is down or whether the link between them has failed.

From inside either partition, there's no way to know which scenario is true.

---

## What a Partition Actually Looks Like

Consider a database cluster: three nodes, replicating state to each other.

```
Normal operation:
   Node A ──── Node B ──── Node C
      (all three talking, all agree)
```

A partition occurs — the network link between Node A and Nodes B+C fails:

```
After partition:
   Node A  ╳╳╳  Node B ──── Node C
   (A can't reach B or C)
   (B and C can't reach A)
```

From Node A's perspective: Nodes B and C have stopped responding. Are they crashed? Is the network between them failed? Node A cannot know. Node A has been receiving client requests during this time. Should it serve them?

From Nodes B and C's perspective: Node A has stopped responding. Same questions. Same uncertainty.

This is the fundamental problem. Decisions must be made. The system can't just pause and wait — clients are waiting for responses right now.

---

## The Split-Brain Problem

If your system doesn't handle this correctly, you get **split-brain**: both partitions believe they're the authoritative half and accept writes independently.

```
After partition:
   Node A (believes it's primary):     Node B ──── Node C (elect new primary)
   Client 1 ─> write X=1 to A         Client 2 ─> write X=2 to B

   Both writes succeed.
   Both clients get a success response.
   X=1 on Node A.
   X=2 on Node B and C.
   When partition heals:
   ??? (which value is correct?)
```

If you're a bank and X is an account balance, one of those writes disappears. Which one? Last write wins? But you confirmed both to your clients.

Split-brain is why distributed systems design is hard. It's not a hypothetical — it happens every time there's a network partition without proper partition handling.

---

## The Three Responses to a Partition

When a partition occurs, your system can do one of three things:

### 1. Stop accepting writes on the minority side (CP behavior)

The minority partition (the side that can't reach a quorum) refuses to accept writes. It may also refuse reads if it can't guarantee freshness.

This is the correct approach if you can't tolerate inconsistency. You pay with availability: clients on the minority partition get errors instead of responses.

**How quorums help**: With 5 nodes, a write requires 3 nodes to acknowledge. If 2 nodes are partitioned off, they can't form a quorum and refuse writes. The other 3 can still form a quorum and continue operating.

```
Partition: [Node A, Node B] ╳╳╳ [Node C, Node D, Node E]

Node A: "I can only reach Node B. That's 2 of 5 — no quorum."
         → refuse writes

Node C: "I can reach D and E. That's 3 of 5 — quorum achieved."
         → accept writes
```

The minority partition is effectively read-only (or offline) until the partition heals. The majority continues as normal. When the partition heals, the minority nodes resync from the majority.

### 2. Accept writes on both sides (AP behavior)

Both partitions accept writes, accepting that they'll diverge. When the partition heals, conflicts must be resolved.

This maximizes availability — no client gets an error due to a partition. The cost is that you might have conflicting writes to resolve.

This is viable if:
- Your data types support automatic merging (CRDTs: counters, sets, maps with appropriate semantics)
- Your business logic can tolerate last-write-wins (and the occasional lost write)
- You have an application-level conflict resolution strategy

E-commerce shopping carts are a classic example. If two browser sessions add different items to a cart during a partition, both additions should survive. The merged cart is the union of both. This is a CRDT-friendly operation.

Account balances are a counterexample. If two ATMs dispense cash during a partition, the total balance might go negative. Automatic merging doesn't work here — you need coordination.

### 3. Detect and queue

Don't respond immediately. Queue the operation, detect the partition, and either fail it or hold it until the partition heals.

This is what message queues and write-ahead logs do. Instead of writing to the database directly, you write to a durable queue. The queue acknowledges receipt. Later, when the database is available, the operation is applied.

This trades latency for durability: the client gets a "queued" response, not a "committed" response. The operation will be applied eventually. This works for asynchronous workflows (sending emails, processing jobs) but not for synchronous interactions ("is this username taken?").

---

## Detecting Partitions

From inside a distributed system, there's no reliable way to distinguish a partition from a crashed node. Both look the same: the remote node has stopped responding.

**Heartbeats**: Send periodic messages to check aliveness. If X consecutive heartbeats fail, assume the remote node is unreachable (either crashed or partitioned). Kubernetes does this; etcd's leader heartbeats do this; most distributed databases do this.

**The timeout problem**: Choose the heartbeat timeout. Too short: false positives trigger unnecessary failovers. Too long: real failures take a long time to detect. There's no universally right answer.

**Phi Accrual Failure Detector**: Instead of a binary "up or down" threshold, compute a probability score based on historical heartbeat timing. As heartbeats come in late or not at all, the score rises. Different systems trigger at different threshold scores. This adapts better to variable network conditions. Cassandra uses it.

---

## Partition Tolerance in Practice

Here's the uncomfortable truth about partition tolerance in production systems: **partitions are rare but not exceptional**.

In a large enough system running long enough, every possible failure mode occurs. Partitions happen due to:

- A misconfigured firewall rule that drops traffic between two services
- A NIC that starts dropping some percentage of packets (not all — *some*)
- A cloud provider's network maintenance window that wasn't communicated
- A BGP routing change that briefly splits your datacenter
- A firewall that times out long-lived connections mid-request
- A load balancer misconfiguration that routes some traffic to unreachable backends

Many of these produce *partial* partitions — some traffic gets through, some doesn't. This is worse than a complete partition, because it's harder to detect and triggers edge cases in partition detection logic.

**Gray failures** are a specific horror: the network doesn't drop packets, it just delays them. Delays that are long enough to trigger timeouts but not long enough to produce connection errors. The system thinks the remote node is unreachable (timeout), but it's actually just slow. If you trigger failover based on this, you might cause a split-brain on a system that was actually fine.

---

## Designing for Partitions

### Use quorums correctly

Ensure your write quorum overlaps with your read quorum. With N=3 replicas, W=2 writes, R=2 reads: W+R=4>3=N. Any read will include at least one node that received the last write.

During a 1-node partition, writes and reads can still proceed (1 node is down; 2 remain; 2>1 = quorum). During a 2-node partition (only 1 node reachable), operations are refused: no quorum achievable.

### Make operations idempotent everywhere

During a partition, queued operations may be retried when the partition heals. If they're not idempotent, retries cause duplicates. Design every operation to be safe to execute multiple times with the same effect as once.

### Design for eventual reconciliation

If your system uses AP behavior, plan the reconciliation strategy before you need it. What happens when a node that was partitioned comes back? Does it accept the state it missed? Does it replay its writes to the rejoining nodes? Does it have conflicts to resolve?

Most databases handle this automatically (replication catchup). At the application level, you may have jobs that need to be reconciled, state that needs to be merged, or events that need to be replayed.

### Test with real partition simulation

Tools like `tc netem` (Linux traffic control) and `iptables` let you simulate partitions in test environments:

```bash
# Drop all traffic to/from a specific host
iptables -A INPUT -s 192.168.1.100 -j DROP
iptables -A OUTPUT -d 192.168.1.100 -j DROP

# Simulate 50% packet loss
tc qdisc add dev eth0 root netem loss 50%

# Simulate 200ms latency
tc qdisc add dev eth0 root netem delay 200ms

# Restore
iptables -D INPUT -s 192.168.1.100 -j DROP
iptables -D OUTPUT -d 192.168.1.100 -j DROP
tc qdisc del dev eth0 root
```

Tools like Toxiproxy (Shopify), Chaos Mesh (Kubernetes), and Pumba (Docker) provide higher-level partition simulation. Netflix's Chaos Monkey and its successors do this in production.

---

## The Fencing Problem in Detail

When a partition heals and a previously-partitioned leader tries to rejoin, you have a specific problem: the old leader may still believe it's the leader, while a new leader has been elected.

This is the **fencing** problem, and it requires an active mechanism to prevent the old leader from acting.

**STONITH (Shoot The Other Node In The Head)**: Literally power off the old leader before electing a new one. Ensures the old leader cannot take any actions. Requires hardware-level access (IPMI, iLO, cloud API to terminate the instance).

**Fencing tokens**: When a leader acquires its lease, it gets a monotonically increasing token. Every write must include this token. If a node attempts a write with a lower token than what the storage system has seen, the write is rejected.

```
Leader 1 acquires lease, gets token=42
         [network partition]
Leader 2 elected, gets token=43
Leader 2 starts writing with token=43
         [partition heals]
Leader 1 attempts write with token=42
Storage: "I've seen token=43. Token=42 is stale. Rejected."
```

This prevents the "zombie leader" problem where an old leader comes back and stomps over writes made by the new leader.

---

## The Humbling Lesson

Network partitions are a reminder that **your system has no global view**. Every node knows only what it has seen. It cannot know what other nodes have seen, or what the true current state of shared data is, without coordination — and coordination requires a network that isn't always reliable.

This is the core reason distributed systems are hard: not the complexity of any single algorithm, but the impossibility of perfect shared knowledge. Every decision in a distributed system is made under partial information.

The right response isn't despair. It's: understand your system's partition behavior, make the consistency/availability trade-off deliberately for each data type in your system, design for reconciliation, test partition handling, and monitor for the early signs of network degradation before it becomes a full partition.

Most importantly: know whether your system will be CP or AP under partition, *before* the partition happens. "We'll figure it out when it occurs" is not a partition strategy.

---

Next: the specific patterns — sagas, outbox, circuit breakers, and others — that turn these principles into working code.
