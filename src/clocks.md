# Time Is a Flat Circle: Clocks, Ordering, and Logical Time

Here is a thing that should disturb you: **no two computers in a distributed system share the same clock.**

They have their own clocks, which are continuously drifting relative to each other and relative to real time. They synchronize against NTP servers periodically, but synchronization is imperfect, and between synchronizations they drift. By milliseconds, usually. Sometimes by more.

This means "the timestamp on this event" is not a fact about when the event happened — it's a fact about what the clock on that particular machine said at that particular moment. And those clocks disagree.

This is a problem because ordering matters. "Last write wins" only works if you can reliably say which write was last. "This event happened before that event" is only meaningful if you can actually determine the order.

In a single-process system, ordering is free — the program counter gives you a total order of all operations. In a distributed system, ordering is something you have to build.

---

## The Clock Problem

### Clock drift

Quartz oscillators — the hardware basis of computer clocks — are accurate to roughly 1 part per million. This sounds precise. It means a clock drifts about 86 milliseconds per day. Over a week without resynchronization, two clocks on separate machines could diverge by over half a second.

NTP (Network Time Protocol) corrects this by synchronizing clocks against a hierarchy of reference servers. NTP can get clocks to within a few milliseconds on a local network, and within tens of milliseconds over the internet.

**A few milliseconds is not zero.** And NTP can fail, be slow, or be skewed.

### The leap second problem

Once or twice a year, a leap second is added to UTC to account for the Earth's variable rotation. Every computer that knows about this gets a correction. Some computers don't. Some handle it by "smearing" the second across hours (Google's solution). Some insert it literally (causing `23:59:60` to appear in logs). Some crash because they assumed time is monotonically increasing.

The 2012 leap second caused massive failures across the internet, including Reddit, LinkedIn, and various Java applications. The root cause: code that assumed the wall clock never goes backwards.

### NTP clock jumps

When NTP corrects a clock that's drifted forward, it can *set the clock backwards*. Code that takes a timestamp at the start of an operation and a timestamp at the end can observe a negative elapsed time.

Robust code uses a **monotonic clock** for measuring elapsed time (which never goes backwards) and the **wall clock** only for human-readable timestamps. Many languages expose both:

```python
# Wall clock — suitable for timestamps, not duration measurement
import time
wall = time.time()  # Can go backwards after NTP correction

# Monotonic clock — suitable for measuring elapsed time
import time
mono = time.monotonic()  # Never goes backwards
```

---

## The Ordering Problem

The fundamental issue: **without shared clocks, you can't reliably order events across machines using timestamps.**

Consider this scenario:

```
Machine A (clock: 10:00:00.000):   Write X=1
Machine B (clock: 09:59:59.998):   Write X=2
```

Machine B's clock is 2ms behind. Both writes are "simultaneous" in real time, but if you order by timestamp, Machine A's write appears to come last. If you're using "last write wins," Machine A's value persists — even though B wrote later by real-world time.

Now consider the even worse case: you need to audit a system and determine whether Event A caused Event B. "Event A happened at 10:00:00.001 on Server 1 and Event B happened at 10:00:00.003 on Server 2" doesn't tell you if A caused B — not with clocks that can be off by 50ms.

---

## Lamport Clocks: Logical Time

In 1978, Leslie Lamport published "Time, Clocks, and the Ordering of Events in a Distributed System" — one of the most cited papers in computer science. In it, he proposed **logical clocks**: a way to order events based on causality rather than physical time.

The key insight is Lamport's **happens-before** relation, written →.

We say A → B ("A happened before B") if:
1. A and B are events in the same process, and A comes before B in program order
2. A is the sending of a message and B is the receipt of that same message
3. There exists some C where A → C and C → B (transitivity)

If neither A → B nor B → A holds, A and B are **concurrent** — neither caused the other.

### How Lamport timestamps work

Each process maintains a counter, initially 0.

**Rule 1**: Before executing an event, increment the counter.

**Rule 2**: When sending a message, include the current counter value.

**Rule 3**: When receiving a message with timestamp T, set the counter to max(own_counter, T) + 1.

```
Process A       Message      Process B
L=0                          L=0
event a1
L=1
           ──(ts=1)──>
                             L=max(0,1)+1 = 2
                             event b1
                             L=3
                             event b2
                             L=4
           <──(ts=4)──
L=max(1,4)+1 = 5
event a2
L=6
```

If A → B, then timestamp(A) < timestamp(B).

But the converse doesn't hold: timestamp(A) < timestamp(B) does NOT mean A → B. Concurrent events can have any timestamp relationship.

**Lamport clocks give you a partial order, not a total order.** You can use them to identify when one event definitely happened before another. You cannot use them to identify when two events are concurrent — two events with different Lamport timestamps might still be concurrent.

---

## Vector Clocks: Tracking Causality

To detect concurrency, you need **vector clocks**.

Instead of a single counter, each process maintains a vector of counters — one per process.

```
System with 3 processes: A, B, C
Each vector clock: [A_count, B_count, C_count]
```

**Rule 1**: On an internal event, increment your own counter.

**Rule 2**: When sending a message, increment your own counter and send the full vector.

**Rule 3**: When receiving a message with vector V, take the component-wise maximum of your vector and V, then increment your own counter.

```
Process A:  [0,0,0]
Process B:  [0,0,0]
Process C:  [0,0,0]

A performs event: A=[1,0,0]
A sends to B (ts=[1,0,0]):
  B receives: B=[max(0,1), max(0,0)+1, max(0,0)] = [1,1,0]

B performs event: B=[1,2,0]
B sends to C (ts=[1,2,0]):
  C receives: C=[max(0,1), max(0,2), max(0,0)+1] = [1,2,1]

A performs another event: A=[2,0,0]
```

Now we can compare events:

Vector clock V1 **happens-before** V2 if every component of V1 is ≤ every component of V2, and at least one component is strictly less.

V1 and V2 are **concurrent** if neither dominates the other.

```
[1,0,0] vs [1,1,0]: [1,0,0] happens-before [1,1,0] ✓
[2,0,0] vs [1,2,1]: concurrent (neither dominates)
[1,2,1] vs [2,2,1]: [1,2,1] happens-before [2,2,1] ✓
```

This is exactly what you need for conflict detection in distributed databases. When two writes have concurrent vector clocks, they're conflicting writes — neither caused the other, and you need a merge strategy.

DynamoDB's original design used vector clocks for exactly this purpose (though they called them "version vectors" and later moved to a simpler model). Riak still uses them.

### The cost: vector clocks grow

Vector clocks have one entry per process. In a system with thousands of processes, this is impractical. Various optimizations exist (dotted version vectors, pruning), but vector clocks scale poorly to very large systems. This is why many systems use simpler approaches with weaker guarantees.

---

## Hybrid Logical Clocks (HLC)

HLC is a practical combination of physical and logical time. The idea: track a logical timestamp that's always at least as large as the current wall clock. When you need to create a timestamp:

- If your wall clock is ahead of your current HLC, advance the HLC to match the wall clock
- If your HLC is already ahead of the wall clock, increment the logical component

The result: events get timestamps that are close to real time (within NTP error bounds) and still maintain the happens-before property.

CockroachDB uses HLC. It enables a clever optimization: instead of 2PC coordination for distributed transactions, CockroachDB uses HLC timestamps and a brief uncertainty window to determine if a historical read can be served safely without coordination.

---

## TrueTime: The Google Solution

In 2012, Google published the Spanner paper describing their globally-distributed relational database. At the heart of Spanner is **TrueTime**: a system that provides bounded time uncertainty using GPS receivers and atomic clocks in every datacenter.

Instead of returning a single timestamp, TrueTime returns an interval: [earliest, latest], with a guarantee that the true current time lies within that interval. The interval width is typically 1-7 milliseconds.

Spanner uses this to provide external consistency (essentially linearizability) for distributed transactions:

1. Acquire locks for the transaction
2. Get TrueTime interval [T_earliest, T_latest]
3. Choose commit timestamp T_s = T_latest
4. Wait until the current time is definitely past T_s (i.e., wait for the interval to close)
5. Release locks and commit

By waiting for the uncertainty interval to pass, Spanner guarantees that any transaction started after this one will see it — the commit timestamp is in the real past by the time the commit is visible.

This is elegant and expensive. It requires dedicated hardware (GPS clocks, atomic clocks) in every datacenter. It's not available to the rest of us, but understanding it helps clarify what "global consistency" actually requires.

---

## What This Means in Practice

### Don't use timestamps for ordering

"Last write wins by timestamp" loses data silently. Timestamps in distributed systems are wrong by up to tens of milliseconds and can go backwards. Using them to determine causality is asking for trouble.

If you must use LWW, use a monotonically increasing logical counter (like a database sequence or a Lamport timestamp) rather than a wall clock timestamp.

### NTP failure is a real failure mode

Monitor clock drift. Alert when clocks diverge from expected values. If a node's clock drifts significantly, its logical clock-based timestamps will be wrong, and any order-dependent logic will misbehave.

### Duration measurement: use monotonic clocks

For measuring "how long did this take," always use a monotonic clock. Never subtract two `time.now()` calls if you care about the result being non-negative.

### For conflict detection: vector clocks or CRDTs

If you need to detect concurrent writes and merge them correctly, you need either vector clocks or data types designed to merge without conflicts. "Last write wins" is not conflict *resolution* — it's conflict *hiding*.

### The production moment

Here's when this bites you: you have two services, both reading from a database and writing results somewhere. You add a timestamp to the result to enable "show the latest version." One service's server has a clock that's 200ms ahead. Its results consistently beat the other service's results in "last write wins" comparisons, even when the other service's results are newer.

You debug this for hours. Finally you check the NTP status on each server. One of them hasn't resynchronized in three days. Clocks diverged. Logic based on clock ordering was wrong.

This is not a hypothetical. It happens.

---

## Summary

Physical clocks drift, can go backwards, and are imprecise. You cannot use wall clock timestamps to reliably order events across machines.

Lamport clocks provide a partial order that respects happens-before relationships. They're simple, cheap, and sufficient for many use cases.

Vector clocks provide true causality tracking — they can detect when two events are concurrent, not just ordered. They're more expensive and don't scale to large process counts.

Hybrid logical clocks combine physical and logical time, giving you timestamps that are close to real time and still logically ordered.

TrueTime provides bounded physical time uncertainty using specialized hardware, enabling external consistency at Google scale.

For most applications: use monotonic clocks for duration, log-sequence-based ordering for event ordering, and be very skeptical of any logic that depends on wall clock timestamps for correctness.

Next: when nodes fail (and they will), how do you keep the system running? Fault tolerance.
