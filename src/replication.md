# Replication Strategies: Leaders, Followers, and Who Gets to Be Right

Replication is how distributed systems achieve two things simultaneously: fault tolerance (if one copy of the data disappears, others remain) and scalability (many nodes can serve reads). The catch is that keeping multiple copies of data consistent is, in practice, one of the hardest problems in distributed systems.

This chapter covers how replication actually works, what the trade-offs are, and what breaks when it goes wrong.

---

## Why Replication Is Hard

Imagine you have a single database. You can read and write to it. It's either up or down. Its data is whatever you last wrote. This is simple.

Now add a second database that mirrors the first. You need:

1. Writes to the first to propagate to the second
2. Reads to return consistent data regardless of which copy you hit
3. A decision about what to do when the two copies disagree
4. A plan for when the network between them fails
5. A way to handle the primary going down

Each of these is a non-trivial problem. The interactions between them are the hard part.

---

## Single-Leader Replication

The most common approach. One node is designated the **leader** (also called primary or master). All writes go to the leader. The leader propagates changes to **followers** (replicas, standbys, secondaries).

```
         Clients
         /     \
    Writes       Reads
       |         |  \
    Leader     Follower1  Follower2
       |           |           |
       └──── replication ──────┘
```

Writes always go to the leader. Reads can go to the leader (for strong consistency) or to followers (for scalability, with potential staleness).

### Synchronous vs Asynchronous Replication

This is where the first major trade-off appears.

**Synchronous replication**: The leader waits for at least one follower to confirm it received the write before acknowledging success to the client.

```
Client ──[write]──> Leader
                    Leader ──[replicate]──> Follower
                                            Follower ──[ack]──> Leader
                    Leader ──[success]──> Client
```

Advantages: If the leader crashes immediately after acknowledging a write, a follower has the data. You don't lose confirmed writes.

Disadvantages: A write is only as fast as your slowest synchronous follower. If that follower is slow or unreachable, writes stall.

**Asynchronous replication**: The leader acknowledges the write immediately, replicates to followers in the background.

```
Client ──[write]──> Leader
                    Leader ──[success]──> Client
                    Leader ──[replicate, eventually]──> Follower
```

Advantages: Writes are fast. Network issues between leader and followers don't block writes.

Disadvantages: If the leader crashes before replication completes, those writes are gone. Followers may lag, serving stale data.

Most databases use asynchronous replication by default because synchronous replication is too expensive for most use cases. PostgreSQL's synchronous replication, MySQL with `rpl_semi_sync`, and similar features exist specifically for use cases where you can't afford to lose any confirmed write.

### Replication Lag

With asynchronous replication, followers will always be behind the leader by some amount — the **replication lag**. Under normal conditions, this is milliseconds. Under high write load or a slow network, it can be seconds or minutes.

During that lag window, followers are serving stale data. This is often acceptable. It becomes a problem when:

- You write to the leader and immediately read from a follower (read-your-writes violation)
- A follower is used for "current" data in a security check
- The lag is measured in minutes and your users notice

**Monitoring replication lag is not optional.** It's a key operational metric. Unexpected lag spikes are early warning signs of follower trouble.

---

## Failover: When the Leader Dies

Single-leader replication's critical operational question is: what happens when the leader fails?

The sequence for a typical automated failover:

1. Leader stops responding (crash, network failure, overload)
2. Follower(s) detect the leader is gone (via heartbeats, timeouts)
3. A new leader is elected (usually the most up-to-date follower)
4. Remaining followers are reconfigured to replicate from the new leader
5. Clients are redirected to the new leader

This sounds clean. In practice:

**Problem 1: What if the old leader wasn't really dead?**

Maybe it was just slow. Now you have two nodes that both think they're the leader. This is called **split-brain**, and it can cause writes to diverge on two separate leaders simultaneously. Systems handle this with "fencing" — ensuring the old leader can't accept writes even if it comes back (by using lease-based locks, STONITH, or other mechanisms).

**Problem 2: The new leader might not have all the data.**

If replication was asynchronous, the new leader (the most up-to-date follower) might be missing some writes that the old leader confirmed to clients. Those writes are gone. The clients that received "success" now have data that never existed. What do you do about the clients that received a success and cached the response?

**Problem 3: The old leader comes back.**

It comes back with writes that the new leader doesn't have. These need to be discarded or reconciled. Most systems discard them. This is another way confirmed writes can disappear.

**Problem 4: What's the right timeout?**

Too short: you do unnecessary failovers when the leader is just temporarily slow (false positive). Too long: you're down for longer than necessary when the leader really is dead. There's no right answer; it depends on your workload.

---

## Multi-Leader Replication

What if every node can accept writes? Now you have multi-leader (or multi-master) replication.

The use case: you want to accept writes in multiple datacenters, each datacenter has its own leader, and they replicate to each other. Writes from users in Europe go to the European datacenter immediately; they replicate to the US datacenter asynchronously.

```
US Datacenter           EU Datacenter
    |                       |
 US Leader <─── sync ──> EU Leader
    |                       |
 US Followers           EU Followers
```

Advantage: writes are fast locally. You're not waiting for cross-Atlantic replication before acknowledging to the user.

The problem: **write conflicts**.

If a user edits a document from two browser tabs at the same time — one tab going to the US datacenter, one to the EU — both writes succeed locally. Now you have two different versions of the document and no canonical truth.

### Conflict Resolution Strategies

**Last-write-wins (LWW)**: Whichever write has the later timestamp wins. The earlier write is silently discarded. This is simple and lossy. Real data disappears without any indication.

**Merge**: Keep all values and present them to the application for resolution. Git does this — a merge conflict. It's correct but requires application logic.

**CRDT**: Use data types specifically designed so that merges are always well-defined. Increment-only counters are the simplest example. A counter that's been incremented by 3 on node A and 5 on node B gives 8 when merged, regardless of order.

**Custom resolution**: The application receives the conflicting values and decides. Works for specific cases (a collaborative text editor can do operational transform). Hard to get right in general.

Multi-leader replication is generally avoided unless there's a strong geographic latency requirement driving it. Offline-first applications (mobile apps that sync when they reconnect) are effectively multi-leader systems. CouchDB and similar databases are designed specifically for this model.

---

## Leaderless Replication

What if there's no designated leader at all? Any node can accept any write. Reads and writes go to multiple nodes simultaneously, and the client (or a coordinator) decides what to believe.

This is the model used by Dynamo-style databases: Cassandra, Riak, and others.

### Quorums

With N replicas, a write is sent to W nodes, and a read is sent to R nodes. For the system to be consistent, you need:

**W + R > N**

If this holds, the read set and the write set must overlap — at least one node that received the write must be included in the read. That node has the latest value.

Common configuration: N=3, W=2, R=2.

```
         Write to 2 of 3:          Read from 2 of 3:
         ┌───────────────┐         ┌───────────────┐
    W -> │ Node A ✓      │    R -> │ Node A ✓      │
    W -> │ Node B ✓      │    R -> │ Node B ✓      │
         │ Node C (skip) │         │ Node C (skip) │
         └───────────────┘         └───────────────┘

At least one node in the read set was in the write set.
The reader sees the latest value.
```

If W=N (synchronous write to all nodes), you can read from any single node and always get the latest value — but writes are slow and unavailable if any node is down.

If R=1 (read from any single node), reads are fast but might be stale unless W=N.

### Sloppy Quorums and Hinted Handoff

During a partition, the quorum nodes might not be reachable. What do you do?

**Strict quorum**: Refuse the write. Maintain consistency, lose availability.

**Sloppy quorum**: Accept the write at a different node (not in the normal N replicas), record that it's holding data meant for the real replicas, and hand it off when the partition heals. This maintains availability at the cost of consistency guarantees.

Cassandra uses sloppy quorums by default. This is why Cassandra is AP — it will accept writes even when the normal replicas can't be reached.

### Concurrent Writes and Version Vectors

With leaderless replication, two clients can write to the same key simultaneously on different nodes. Unlike multi-leader where there's at least a per-datacenter ordering, here there's no ordering at all.

Databases handle this with **version vectors** (sometimes called vector clocks in this context): each key carries metadata tracking which nodes have seen which writes. When two versions are concurrent (neither happened-before the other), the database keeps both and requires the application to merge.

This is correct but places burden on the application. Dynamo's original design required the application to merge conflicts. DynamoDB simplified this by using Last-Write-Wins (at the cost of occasional data loss). CRDTs solve it for specific data types.

---

## Replication in Practice: What Actually Goes Wrong

### The "read your own writes" bug

You write to the leader, then immediately redirect to a page that reads from a follower. The follower hasn't replicated yet. The data you just wrote isn't there. The user reports a bug. You can't reproduce it in testing because testing has no replication lag.

Fix: after a write, route subsequent reads to the leader for a short window, or use sticky sessions, or check timestamps to ensure the replica has caught up.

### The "disappearing write" after failover

User submits a form. Write goes to the leader. Leader confirms. Leader dies. New leader is elected but didn't have that write (it was async). User reloads page. Their submission is gone.

Fix: synchronous replication for critical writes, or wait for acknowledgment from N nodes before confirming to the user.

### The replication lag spike causing incorrect data

Your reporting dashboard reads from a replica. Replication lag spikes to 10 minutes during a high-write period. Your dashboard is showing 10-minute-old data. You make a business decision based on it.

Fix: monitor replication lag. Have alerts. Know your lag when it matters.

### The split-brain write conflict

Network partition causes two nodes to both believe they're the leader. Both accept writes for the same key. Partition heals. Both values exist. One gets silently discarded. Data is lost with no error, no log entry, no indication that anything went wrong.

Fix: proper fencing, or use a consensus-based system for leader election. Which brings us to the next chapter.

---

## Choosing a Replication Strategy

| Requirement | Strategy |
|-------------|----------|
| Simple, strong consistency | Single-leader, synchronous to at least one follower |
| High write throughput, tolerate some lag | Single-leader, asynchronous |
| Low-latency writes across geographic regions | Multi-leader |
| High availability, tolerate eventual consistency | Leaderless (Dynamo-style) |
| Offline-first / sync-on-connect | Multi-leader / CRDTs |

The right choice depends on what you can afford to lose. Single-leader async is the default for most applications because it's simple and fast and the failure modes are well-understood. Move to synchronous or multi-datacenter replication when the cost of data loss exceeds the cost of coordination.

Whatever you choose: monitor lag, test failover, and know what happens to in-flight writes when a leader dies. Because it will.
