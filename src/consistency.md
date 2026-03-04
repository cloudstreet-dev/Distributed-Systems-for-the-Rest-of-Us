# Consistency Models: From Strong to Eventual and Everything Uncomfortable In Between

The CAP theorem gave us a binary: consistent or available under partition. But "consistent" is doing a lot of work in that sentence. Consistency isn't a single property — it's a spectrum of guarantees, ranging from "reads always see the latest write" down to "reads will eventually agree with other reads, probably, at some point."

Where you sit on that spectrum determines what kind of bugs your users encounter and what kind of performance trade-offs you're making.

This chapter is a map of that spectrum. We'll go from the strongest guarantees to the weakest, with concrete examples of what breaks at each level.

---

## Why Consistency Models Exist

The need for consistency models comes from replication. If you have a single copy of your data and one process accessing it, there's no consistency question — you read what's there. The problem starts the moment you have two copies (replicas) and want them to stay in sync.

Replication exists for two reasons:

1. **Availability**: If one replica goes down, others can still serve requests.
2. **Performance**: Reads can be distributed across replicas, reducing load on any single node.

Both of these are good reasons. The cost is that replicas can diverge — one might have seen a write that the other hasn't yet. When that happens, what do you guarantee to clients?

That's what a consistency model specifies.

---

## The Consistency Hierarchy

Here's the landscape, from strongest to weakest:

```
Stronger guarantees (more correct, more coordination required)
│
├─ Linearizability (strong consistency)
│
├─ Sequential consistency
│
├─ Causal consistency
│
├─ Read-your-writes (session consistency)
│
├─ Monotonic reads
│
└─ Eventual consistency

Weaker guarantees (more performant, less coordination required)
```

These aren't arbitrary categories — each one adds a specific guarantee on top of the ones below it, and removing that guarantee is what enables a specific performance optimization.

---

## Linearizability: The Gold Standard

Linearizability is the strongest consistency model for a distributed system. It provides the following guarantee:

**Every operation appears to take effect atomically at some point between its invocation and its response, and operations are ordered consistently with real time.**

In plain English: the system behaves as if there's exactly one copy of the data, and every operation takes effect at a single moment in time. If operation A completes before operation B begins, then any read that sees B's write must also see A's write.

```
Time ─────────────────────────────────────────────────>

Client 1: ──[write X=1]──────────────────────────────
Client 2:              ──[read X]── gets 1 ✓
Client 3:          ──[read X]── gets ???

Client 3's read overlaps with Client 1's write.
In a linearizable system, it gets either 0 or 1,
and the choice must be consistent with other reads.
```

Linearizability is what you get with a single-leader database operating synchronously. It's also what systems like Zookeeper and etcd provide, and what Google's Spanner provides globally (using atomic clocks).

**The cost**: to provide linearizability, the system must coordinate before responding. In practice, this means: a write must be acknowledged by a quorum of nodes before it's confirmed; a read must contact the node with the authoritative latest value. This coordination takes time and fails when nodes can't communicate.

**When you need it**: distributed locking, leader election, any case where "multiple clients race for something and only one should win" must work correctly.

---

## Sequential Consistency

Sequential consistency relaxes one part of linearizability: operations don't need to take effect in real-time order, but they must appear in *some* order that is consistent across all nodes.

Specifically: every node sees operations in the same order, and for each client, their operations appear in the order the client issued them.

What's lost compared to linearizability:

```
Time ─────────────────────────────────────────────────>

Client 1: ──[write X=1]──────────────────────────────
Client 2:              ──[read X]── gets 0 (!) ✓ for SC

This is allowed in sequential consistency.
The system might see Client 2's read as if it happened
before Client 1's write, even though Client 2 started
reading after Client 1 finished writing.
```

This seems wrong, but it's allowed because sequential consistency doesn't require operations to respect real-world time — only that the *order* is consistent for all clients. Client 2 seeing 0 is fine as long as every client sees that read as having happened "before" Client 1's write.

Sequential consistency is rarely used in practice because: (a) the relaxation from linearizability is subtle enough that most engineers don't notice it, and (b) the real benefit comes from weaker models that allow much more concurrency.

---

## Causal Consistency

Causal consistency tracks *causality* between operations and preserves it. If event A causally precedes event B (A happened-before B, or A produced a value that B reads), then every process sees A before B.

Operations that are not causally related can appear in any order on different nodes.

```
Client 1: ──[write post="Hello"]──────────────────────────────
Client 2:                        ──[write reply="World"(to Hello)]──
Client 3:                        ──[read]──

Client 3 reads post="Hello" and reply="World".
In causal consistency, if Client 3 sees the reply,
it is guaranteed to also see the post it's replying to.

But the reply might not be visible yet on Node B
while it's visible on Node A. That's fine —
causal consistency doesn't require everyone to see
things immediately, just in the right order.
```

Causal consistency is powerful and increasingly popular. MongoDB's causally consistent sessions use it. It's the model that allows geo-distributed systems to avoid coordination for most operations while still preserving the ordering that applications care about.

**The insight**: most applications don't actually need full linearizability. They need *causal* ordering — if you see the effect of an action, you should also see its cause. A comments system needs to show comments in the right thread. It doesn't need every comment to be globally ordered with every other event in the system.

---

## Session Guarantees

Below causal consistency, we have a set of weaker guarantees often called "session guarantees." These are per-client guarantees rather than global ones.

### Read-Your-Writes (RYW)

If a client writes a value, it will see that value (or a later one) on subsequent reads.

This sounds obvious, but it breaks in common architectures. If you write to a primary database and then read from a replica that hasn't caught up yet, you might not see your own write. This is a concrete, common production bug.

```
Client writes to Primary: user.email = "new@email.com"
Client reads from Replica (lagging 200ms): gets "old@email.com"
Client: "why didn't my change save??"
```

RYW prevents this. The system might route reads to the primary after a write, or use sticky sessions to route to the same replica, or use tokens to ensure the replica has caught up before serving the read.

### Monotonic Reads

Once you've seen a value, you'll never see an older one. If you read X=5, a subsequent read will return 5 or greater — never 4.

```
Client reads from Replica A: gets version 42
Client reads from Replica B: should get ≥ version 42
Without monotonic reads: might get version 38
```

Without monotonic reads, a client can see what looks like time going backwards — data that was there and then isn't. This is extremely confusing.

### Monotonic Writes

Writes from a single client are executed in order. If Client 1 does Write A and then Write B, Write A must be applied before Write B everywhere.

### Writes Follow Reads

If a client reads X=5 and then writes Y (based on seeing that X=5), then the write of Y should be sequenced after the write that set X=5. This is the write side of causality.

---

## Eventual Consistency

Eventual consistency is the weakest useful guarantee: **if no new updates are made to an object, eventually all reads will return the same last-written value.**

That's it. No ordering guarantees. No freshness guarantees. Just: we'll get there eventually.

What this means concretely:

- Two clients can read the same key at the same time and get different values
- A client can read a key, write to it, and then read a stale value
- Different nodes can have different current values for the same key
- None of this is a bug — it's the design

The system will resolve these discrepancies. The question is how, and when.

### Conflict resolution under eventual consistency

When two nodes accept conflicting writes (during a partition, for example), they need a rule for which write wins when they sync up.

**Last-Write-Wins (LWW)**: Whichever write has the later timestamp wins. The problem: timestamps in distributed systems are not reliable (see the clocks chapter). A write might arrive with an earlier timestamp even if it happened later. You can silently lose writes.

**Multi-Value / "Siblings"**: Some systems (like Riak and CouchDB) store all conflicting versions and surface them to the client for resolution. This is correct but puts the burden on the application.

**Conflict-free Replicated Data Types (CRDTs)**: Specially-designed data structures where all possible orderings of operations produce the same result. A CRDT counter that supports increment/decrement can be merged from any two states. This sounds magical and has real limits, but for the right use cases (counters, sets, maps with specific semantics) it works well.

---

## The Production Problem: Stale Reads

Here's the scenario that will make you care about all of this.

You're building a system where users can disable their account. The admin panel writes `account.active = false` to the primary database. The API that checks whether a user can log in reads from a read replica. The replica is 500ms behind.

For the next 500ms, users of the disabled account can still log in.

This might be acceptable (it's a narrow window). It might be catastrophic (abuse case, court order, account compromise). The right answer depends on the application. But you need to *know* you've made this trade-off.

```
Primary DB:    active=false (written at T=0)
Replica:       active=true  (not yet replicated, T=0 to T+500ms)
Auth service reads replica: user is active ✓ (but shouldn't be)
```

The fix depends on what guarantee you need:

- **If you need linearizability**: Read from the primary always (or use a system that provides synchronous replication).
- **If RYW is enough**: Route the disable request's originator to the primary, but other clients can read from replicas.
- **If you can tolerate a bounded window**: Accept the 500ms lag, document it, and make sure it's not a security boundary.

---

## Choosing a Consistency Model

Here's a practical decision matrix:

| Use Case | Minimum Required Guarantee |
|----------|---------------------------|
| Distributed lock / leader election | Linearizability |
| Financial transactions, inventory counts | Linearizability or strong serializable |
| "Can this user log in?" with security implications | Linearizability or synchronous replication |
| Social media feed ordering | Causal consistency |
| User sees their own profile edits | Read-your-writes |
| Read-heavy analytics | Eventual consistency |
| View counters, likes, non-critical metrics | Eventual consistency |
| User session / shopping cart | Read-your-writes + monotonic reads |

The right-hand column is the *minimum*. You can always use a stronger guarantee than required — the cost is coordination overhead and reduced availability under partition.

---

## The Uncomfortable Truth About Consistency

Most applications use relational databases with ACID transactions and assume they get linearizability. For writes within a single database, that's often true. But:

- Reads from replicas are not linearizable by default in most databases
- Reads from caches are not linearizable
- Cross-service operations are not linearizable even if each service individually is
- Any operation involving a clock ("last updated at") is not linearizable without synchronized clocks

You're already trading away consistency in ways you might not have explicitly decided. The question is: are those trade-offs appropriate for your use case?

After this chapter, you should be able to answer that. Or at least ask the right questions.

---

Next: how do you actually keep multiple copies of data in sync? That's replication — and it turns out there are more ways to do it badly than there are to do it well.
