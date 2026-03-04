# CAP Theorem: Consistency, Availability, and the Lie You Have to Pick

In 2000, Eric Brewer gave a keynote at PODC (Principles of Distributed Computing) in which he conjectured that a distributed system can provide at most two of three properties: Consistency, Availability, and Partition Tolerance. Two years later, Gilbert and Lynch formalized it as a theorem.

For about a decade afterward, "CAP theorem" became a kind of magic incantation in systems design discussions. "We chose AP over CP" was said with the confidence of someone who knew exactly what they were talking about. Often, they didn't.

CAP theorem is real, important, and widely misunderstood. Let's fix that.

---

## The Three Properties

### Consistency (C)

In CAP, consistency means **linearizability**: every read sees the most recent write, or an error. If I write a value to the system and you immediately read that key, you get that value. No exceptions, no "maybe."

This is a strong guarantee. It means the system behaves as if there's a single up-to-date copy of the data, even if there are many replicas behind the scenes.

Important: this is not the same as the "C" in ACID (which is about constraints within a transaction). CAP consistency is specifically about the freshness and ordering guarantees for reads and writes across a distributed system. The naming collision is genuinely unfortunate and has confused more engineers than you'd think.

### Availability (A)

Availability means **every request receives a response** — not a timeout, not an error, but an actual answer. The response might not be the most current value, but the system will respond.

Importantly, CAP availability doesn't mean "99.9% uptime." It's a binary property in the formal definition: a system is either always available (every node always responds) or it isn't.

### Partition Tolerance (P)

A network partition is when the network between nodes drops messages. Nodes can't tell whether the other nodes are down or whether the network between them has failed. The system is split into two (or more) groups that can't communicate.

Partition tolerance means **the system continues operating even when partitions occur**.

---

## Why You Can't Have All Three

Here's the problem. Suppose you have two nodes, Node A and Node B, that replicate data to each other. A network partition occurs — they can no longer communicate.

```
          [PARTITION]
    Node A -----X----- Node B
     |                    |
   Client 1            Client 2
```

Now a client writes to Node A. Node A can't tell Node B about the write because of the partition.

Client 2 then reads from Node B.

You now have to make a choice:

**Option 1: Respond with the (possibly stale) value on Node B**
- Node B responds, so the system is *available*
- But the response might be wrong (pre-partition value), so the system is *not consistent*
- This is AP behavior

**Option 2: Refuse to respond until the partition heals**
- Node B won't answer read requests it can't verify
- The system is *consistent* (it never serves stale data)
- But the system is *not available* (it's refusing requests)
- This is CP behavior

There is no Option 3 that provides both. If you're partitioned, you have to pick.

---

## The Part That's Usually Glossed Over

Here's what the simplified explanation misses: **you can't just "drop" partition tolerance**.

A network partition — messages being lost between nodes — is not a hypothetical. It happens. In any real distributed system running over a real network, partitions occur. Hardware fails. Cables get pulled. Routers go wrong. Cloud providers have network incidents. You cannot build a distributed system that is immune to partitions.

This means in practice, the CAP theorem is not a choice between three options — it's a choice between two options *when partitions occur*:

- **CP**: Refuse requests that you can't serve consistently. Sacrifice availability.
- **AP**: Serve requests even if the answer might be stale. Sacrifice consistency.

When there's no partition — the normal, happy-path case — you can have both consistency and availability. The trade-off only bites during partitions.

The practical framing, then, is: **what does your system do when the network between your nodes goes down?**

---

## CP Systems: Consistent Under Partition

A CP system responds to a partition by refusing to answer queries it can't guarantee are correct.

The classic examples are:

- **Zookeeper**: A distributed coordination service. During a partition, the minority partition (nodes that can't reach the quorum) will refuse writes and may refuse reads. Better to be unavailable than to be wrong.
- **HBase** (and most strongly-consistent databases): Writes are refused if replication can't complete. Reads may be refused if they can't be guaranteed fresh.
- **etcd**: Used in Kubernetes for cluster state. Correctness of cluster configuration trumps availability.

The pattern here is: **these are systems where being wrong is worse than being unavailable**. If your cluster coordination service tells half of your workers to take on a job that the other half already took, you have a bigger problem than "service was down for a minute."

In CP systems, during a partition:
```
Client ─── request ───> Node (minority partition)
Node: "I can't guarantee this is correct. Returning error."
Client: gets an error
```

The client has to handle errors. The system maintains consistency guarantees for clients that do reach a healthy partition.

---

## AP Systems: Available Under Partition

An AP system responds to a partition by serving requests from whatever data it has, even if that data might be stale.

The classic examples:

- **DynamoDB** (in its default configuration): Prioritizes availability. You'll get a response, but it might not reflect the most recent write.
- **Cassandra**: AP by default. Writes go to multiple nodes, but depending on consistency level, you might read from a node that hasn't received the latest write yet.
- **CouchDB**: Allows multiple writers to diverge during a partition and resolves conflicts later.

The pattern: **these are systems where being unavailable is worse than being potentially stale**. A shopping cart that occasionally shows one fewer item than you added is annoying. A shopping cart that returns a 503 for 10 minutes during a network blip is a business problem.

In AP systems, during a partition:
```
Client ─── request ───> Node (minority partition)
Node: "Here's what I know, which might be stale."
Client: gets a (possibly stale) response
```

The client gets an answer. Whether that answer reflects the latest reality is not guaranteed.

---

## The Part That Actually Matters

The formal CAP theorem — with its binary properties — is useful as a mental model but not as a practical design tool. Real systems operate on a spectrum. Here's why:

### Consistency is a spectrum

CAP's "C" (linearizability) is the strongest consistency model. There are weaker ones: sequential consistency, causal consistency, eventual consistency. A system might not be linearizable but still be strong enough for your use case. (Chapter 4 covers this in detail.)

### Availability is a spectrum

"The system responds" can mean different things. Is 99% uptime "available"? 99.9%? The formal definition is unhelpful here. What matters is: how often does your system return errors or timeouts to clients under normal operation, under degraded operation, and under partition?

### Partitions are rare but not exceptional

You're going to spend 99.9% of your time in the happy path where there's no partition. The CP vs AP choice mostly affects *rare* failure scenarios. What are the actual consequences of your choice in those scenarios? Design for that.

---

## A More Useful Framework: PACELC

In 2012, Daniel Abadi proposed PACELC as an extension of CAP that captures the tradeoff more completely.

PACELC says:
- During a **P**artition, choose between **A**vailability and **C**onsistency (CAP's core insight)
- **E**lse (during normal operation), choose between **L**atency and **C**onsistency

The second part is the addition. Even when there's no partition, there's a trade-off: to guarantee consistency across multiple nodes, you need to coordinate those nodes (which takes time and adds latency). Or you can skip the coordination (lower latency) and accept that reads might not reflect the latest write.

This is the trade-off that actually dominates day-to-day system behavior:

| System | Partition | Normal |
|--------|-----------|--------|
| DynamoDB | PA | EL (with eventual consistency) |
| Zookeeper | PC | EC |
| Cassandra | PA | EL |
| PostgreSQL (single node) | N/A | EC |
| CockroachDB | PC | EC |

The last two rows illustrate something: single-node PostgreSQL sidesteps CAP entirely (there's no distributed state to partition), but once you add replicas, you're back in the trade-off.

---

## How to Actually Use CAP in System Design

Here's the practical question you should ask:

**What happens in my system when two of my nodes can't talk to each other?**

If your answer is "one of them stops accepting writes" — that's CP behavior. You've decided consistency matters more than availability during partitions.

If your answer is "they both keep accepting writes and sync up later" — that's AP behavior. You've decided availability matters more. You now need to think about conflict resolution when the partition heals.

**What happens to your users in each case?**

CP failure: Some users get errors or time out. Clear, if unpleasant.

AP failure: Some users might see stale data or have their writes conflict with others. Subtle, and potentially confusing.

**What does your business logic require?**

If you're storing inventory counts and you can't afford to oversell: lean CP. Errors are preferable to selling something you don't have.

If you're storing user sessions or view counts or shopping carts: lean AP. A stale session is recoverable. An unavailable login is a lost user.

---

## A Note on "Eventual Consistency"

You'll often hear "we use eventual consistency" as a response to CAP, as if it neatly settles the question. It doesn't.

"Eventual consistency" means: if no new writes happen, all replicas will *eventually* converge to the same value. This is a liveness property — it tells you things will get consistent *eventually* but says nothing about when, or about what "consistent" means in the interim.

What it means in practice:

- **How eventual is "eventual"?** Milliseconds? Seconds? Minutes? Under what conditions?
- **What do reads return during the eventually-consistent window?** The old value? A random one of several competing values?
- **How are write conflicts resolved?** Last-write-wins (by timestamp)? Application-level merge? Manual intervention?

"We use eventual consistency" is a starting point for a design conversation, not an answer to one.

---

## The Real Lesson

CAP theorem tells you that **perfection is not available** in a distributed system under partition. You cannot be simultaneously consistent and available when the network breaks.

But the more important lesson is: **you need to understand your failure modes and design for them**. Most of the time your network isn't partitioned. But when it is — or when a single node fails, or when a database is slow, or when a third-party service is down — what does your system do? Who sees what? What is the blast radius?

Answering those questions requires understanding consistency models deeply. Which is why that's the next chapter.
