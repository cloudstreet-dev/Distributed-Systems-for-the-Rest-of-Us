# Making Real Decisions: How to Think About Your Actual System

Everything in this book up to this point has been building toward this chapter. You now have a mental model of the failure modes, the theoretical constraints, and the patterns. The question is: how do you apply all of that to your actual system, with your actual team, your actual scale, and your actual constraints?

This chapter is opinionated. That's the point.

---

## Start With the Actual Problem

The most common mistake in distributed systems design is solving the wrong problem at the wrong scale.

Before you reach for a distributed database, a message queue, or a saga pattern, ask:

**What is the simplest system that solves the actual problem?**

A monolith with a single PostgreSQL database is not an embarrassing architecture. It's a legitimate choice that solves real problems with significantly less complexity than a microservices deployment. Shopify ran on a single Rails monolith for years while processing billions in transactions. Stack Overflow runs on three web servers and two SQL Server instances serving millions of requests per day.

The question isn't "can this scale?" The question is "does it need to scale, and in what way, right now?"

Distributed systems problems are overhead. You pay the overhead in exchange for something: more capacity, better fault isolation, geographic distribution, independent deployability. If you're not getting the benefit, you're paying the overhead for nothing.

---

## The Decision Framework

When you need to make an architectural decision, work through this sequence:

### 1. What is the consistency requirement?

For each piece of state your system manages, determine the minimum consistency guarantee that correctness requires.

**Examples**:

| Data | Required Consistency |
|------|---------------------|
| User account balance | Strong (linearizable) — incorrect balance is a real error |
| Shopping cart contents | Read-your-writes + monotonic reads — user must see their own edits |
| Social media feed | Causal or eventual — stale feed is acceptable |
| View/like counts | Eventual — approximate counts are fine |
| Distributed lock / leader | Linearizable — split-brain is catastrophic |
| User profile data | Read-your-writes — user must see their own profile changes |
| Session data | Read-your-writes — user must stay logged in |
| Analytics/metrics | Eventual — approximate is sufficient |

Do this exercise explicitly, per-data-type. Not as a team assumption, not as "we'll use eventual consistency" applied uniformly. Per. Data. Type.

### 2. What is the availability requirement?

How much downtime is acceptable? Translate that into concrete numbers:

- 99% uptime = 87.6 hours/year of downtime acceptable
- 99.9% = 8.76 hours/year
- 99.99% = 52.6 minutes/year
- 99.999% = 5.26 minutes/year

The cost to achieve each additional 9 is roughly an order of magnitude more expensive. 99% is achievable with a single well-managed server. 99.9% requires redundancy. 99.99% requires active-active multi-zone setups. 99.999% ("five nines") requires geo-distribution, extremely careful engineering, and probably a dedicated platform team.

**What do your users actually experience?** A B2B SaaS with enterprise customers has different availability requirements than a consumer app. A midnight maintenance window on an enterprise product is far less painful than 10 minutes of downtime during peak hours on a consumer product.

**Be honest about your SLA.** If you're a startup with no SLA, you have more flexibility than you might think. If you have contractual 99.9% SLA, you need to engineer for it.

### 3. What are the partition scenarios?

Your system will be partitioned. When it is, what should happen?

**Option A (CP)**: Some users get errors. Acceptable if the alternative is corrupted data.

**Option B (AP)**: Some users get stale data. Acceptable if errors are worse than staleness.

**Option C (AP with compensation)**: Some users get stale data, and you have a process to reconcile when the partition heals.

Make this decision *now*, not during an incident. Write it down. Make sure your team agrees.

### 4. What's the actual scale?

Not the imagined scale. The actual, measured, or rigorously estimated scale.

**Reads per second**: How many reads does your system serve? A typical small SaaS might see 100-1,000 RPS. A medium-scale product might see 10,000-100,000. Large-scale is 100,000+. These are very different engineering problems.

**Write throughput**: Writes are more expensive than reads in most systems. A single PostgreSQL primary can handle ~10,000 writes/second if they're simple. More complex writes (with indexing, constraints) are lower.

**Data volume**: How much data are you storing? PostgreSQL handles terabytes without breaking a sweat on appropriate hardware. Beyond that, you start needing specialized tooling.

**Latency requirements**: What's the acceptable latency at p99? Consumer-facing products typically need p99 < 500ms. Latency-sensitive systems (trading, gaming) need p99 < 50ms or less.

Measure first. Estimate if you can't measure. Don't guess.

---

## The Database Decision

Most distributed systems problems are really "which database should I use?" in disguise.

**Start with a relational database.** PostgreSQL specifically. It handles ACID transactions, foreign keys, complex queries, JSON, full-text search, time-series-like queries, geospatial queries, and replication — and it scales further than most teams need. If you outgrow it, you'll know, and you'll have good reasons to reach for something else.

The "when to add a read replica" question: when your primary's CPU is consistently above 70% due to read load. Or when you need geographic proximity for reads. Not before.

The "when to shard" question: when a single node genuinely can't hold your data or handle your write volume. This is much rarer than people think. Before sharding, profile what's actually causing your problems. It's usually schema design, query performance, or missing indexes — not the fundamental limits of a single-node database.

### When to reach for a distributed database

**CockroachDB or Spanner (distributed SQL)**: When you need:
- Geo-distributed writes with acceptable latency
- Horizontal write scaling across nodes
- Strong consistency across a distributed cluster
- SQL interface

The cost: higher write latency (consensus required per transaction), operational complexity, cost.

**Cassandra or DynamoDB (distributed KV/wide-column)**: When you need:
- Very high write throughput (millions of writes/second)
- Linear horizontal scalability
- Acceptable eventual consistency
- Simple access patterns (key-based lookups, range scans)

The cost: no joins, limited query flexibility, eventual consistency trade-offs, conflict resolution complexity.

**Redis (in-memory)**: When you need:
- Sub-millisecond reads and writes
- Caching, session storage, rate limiting, pub/sub
- Data that fits in memory or where cache eviction is acceptable

Not a primary database. A cache and operational data store.

**Kafka (distributed log)**: When you need:
- Durable, ordered event streaming
- Multiple consumers reading the same events
- Temporal decoupling between producers and consumers
- Long event retention for replay

Not a database. A durable message log.

---

## The Microservices Decision

Microservices are a deployment and organizational pattern, not a performance optimization. They solve specific problems:

- **Independent deployability**: Service A can be deployed without affecting Service B
- **Independent scalability**: Scale the services that need scaling without scaling everything
- **Organizational boundaries**: Different teams own different services with clear interfaces
- **Technology diversity**: Different services can use different languages/databases where appropriate

They create specific problems:
- **Network calls replace function calls**: Added latency, failure modes, serialization overhead
- **Distributed transactions**: The saga chapter exists because of this
- **Operational complexity**: N services = N deployment pipelines, N monitoring dashboards, N on-call escalation paths
- **Observability**: Debugging a problem that spans 5 services is significantly harder than debugging a monolith

**The honest trade-off**: Microservices are appropriate at organizational scale. When you have multiple teams, they need boundaries. When services have truly different scaling requirements, they need independent scaling. When you're deploying dozens of times per day, independent deployability matters.

They're frequently inappropriate for small teams. A team of 5 engineers managing 20 microservices is paying enormous overhead for problems they don't have.

**The "modular monolith"** is often the right intermediate step: a single deployable that's internally organized into well-separated modules with clean interfaces. You get development velocity, simple deployment, easy local development — and when a module genuinely needs to be extracted, it's easier to do from a clean module boundary than from spaghetti code.

---

## Designing for Operations

A system that works in development and fails mysteriously in production is not a working system.

### Observability as a first-class concern

Build logging, metrics, and tracing *before* you need them. The format: structured JSON logs with correlation IDs. The minimum metrics: request rate, error rate, latency (p50/p95/p99), saturation (connection pool usage, queue depth, memory usage).

Instrument your application so that when something goes wrong, you can answer: "What was happening when this broke?" Without observability, you're debugging blindfolded.

### Runbooks

For every operational procedure (deploying a new version, adding a read replica, failing over to a standby database, rolling back a bad deployment), write the steps down. A runbook doesn't need to be beautiful — it needs to be accurate. Update it when the procedure changes.

The runbook's audience is: you, at 3am, after being woken up, slightly panicked. Write for that person.

### Graceful degradation in the design phase

Decide, for every external dependency your service has, what to do if that dependency is unavailable. Document it. Implement it. Test it.

"This service will be unavailable if the payment service is down" is a valid choice if the entire function of the service is payment processing. "This service will show cached recommendations if the recommendation service is down" is the right choice if recommendations are non-critical.

Both are reasonable. Undocumented implicit behavior is not.

---

## Debugging Distributed Systems

When something goes wrong in a distributed system, the debugging process is different from debugging a single-process application.

**Step 1: Establish a timeline.** What happened, in what order, on which services? This requires correlated logs with timestamps and a common correlation ID. If you don't have this, you're guessing.

**Step 2: Find the divergence point.** At what point did the behavior stop being what you expected? Work backwards from the visible symptom to find where the bad state originated.

**Step 3: Check the boring stuff first.** Disk full? Memory exhausted? Connection pool exhausted? CPU pinned? Replication lag spiked? Before assuming a subtle consensus bug, check that the system has the resources it needs.

**Step 4: Check external dependencies.** Is the downstream service healthy? Is there network degradation between your services? Is the database having lock contention? Distributed system bugs are often not bugs in your code — they're emergent behavior from the interaction of multiple systems under stress.

**Step 5: Reproduce in isolation.** Can you reproduce the problem with a single service in a test environment? If so, you've narrowed the problem significantly. If not, it's likely a timing or interaction issue that requires the full system.

**The hardest class of bugs**: race conditions that appear under specific timing conditions. These are reproducible only under load and require either distributed tracing (to see the actual sequence of events) or careful log analysis (to reconstruct it after the fact). This is why observability is not optional.

---

## The Checklist

When designing a new distributed system component, answer these questions before writing code:

**Consistency**
- [ ] What consistency guarantee does each data type require?
- [ ] Are reads from replicas acceptable for this data? What's the maximum acceptable lag?
- [ ] Do you need read-your-writes consistency? How will you implement it?

**Failures**
- [ ] What happens when downstream Service X is unavailable?
- [ ] What happens when a database write succeeds but the subsequent publish fails?
- [ ] What happens when a network partition occurs between your nodes?
- [ ] Is every operation idempotent, or have you designed safe retry behavior?

**Operations**
- [ ] How do you deploy a new version with zero downtime?
- [ ] How do you roll back a bad deployment?
- [ ] How do you fail over to a standby database?
- [ ] What are your health check endpoints?
- [ ] What metrics are you tracking?
- [ ] What alerts fire, and at what thresholds?

**Testing**
- [ ] Do you have integration tests that simulate failure scenarios?
- [ ] Have you tested failover? (Test it in staging before production needs it.)
- [ ] Have you tested with realistic data volumes?

**Documentation**
- [ ] Is the consistency model documented for each data type?
- [ ] Is the failure behavior documented for each external dependency?
- [ ] Is there a runbook for the key operational procedures?

You don't need to answer all of these perfectly before you start. You do need to know which ones you're deferring and why.

---

## The "Good Enough" Question

Not every system needs to be engineered for five nines and global scale. Not every problem needs the most correct solution — it needs a solution that's correct enough for the use case, with acceptable failure modes, and maintainable by the team that will operate it.

The goal is **deliberate decisions with known trade-offs**, not perfect decisions.

A shopping cart that occasionally shows one fewer item than expected is a minor bug. A payment system that occasionally double-charges is a serious incident. They deserve different levels of engineering rigor.

Know which kind of system you're building. Apply the rigor that situation requires. Don't apply the same rigor uniformly across everything — you'll either over-engineer the low-stakes parts or under-engineer the high-stakes parts. Usually both.

---

## The Meta-Skill

The meta-skill in distributed systems design is: **thinking concurrently about what can go wrong.**

When you write `db.write()` followed by `queue.publish()`, think: what is the state of the system if the first succeeds and the second fails? Is that state valid? Is it recoverable?

When you design a service that reads from a replica, think: what do users experience when the replica is lagging? Is that acceptable?

When you design a multi-step workflow, think: if the process crashes at step 3, what state is the system in? What needs to happen to complete or roll back from that state?

This habit — examining the gaps between operations — is the difference between code that works in testing and infrastructure that holds up in production.

---

That's the practical core. The last chapter points you toward where to go from here.
