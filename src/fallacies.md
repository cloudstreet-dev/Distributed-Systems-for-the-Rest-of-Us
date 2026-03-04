# The Eight Fallacies of Distributed Computing (They Were Right)

In the early 1990s, Peter Deutsch and colleagues at Sun Microsystems compiled a list of assumptions that developers routinely make when building distributed systems. They called them "the fallacies of distributed computing."

The list has survived 30 years because the fallacies are *sticky*. Every developer who has shipped distributed code has violated at least four of them. Most have violated all eight, several times, in production.

Here they are, with commentary on how they kill you.

---

## Fallacy 1: The Network Is Reliable

This is the foundational mistake. It feels true locally — you open a connection, you send bytes, you receive bytes. The connection is either up or down, and it's usually up. So you code as if that's always the case.

But at any given moment, your network call might:

- Succeed normally
- Fail immediately with a connection error
- Start, then hang indefinitely
- Start, deliver the request, and fail before delivering the response
- Succeed, but the response arrives after your timeout has already fired

That last one is particularly treacherous. **You timed out and retried, but the original request already succeeded.** You've now done the operation twice.

```
Client          Network         Server
  |                |               |
  |--- request --->|               |
  |               -|-- request --->|
  |               -|               |--- processing...
  [timeout fires] -|               |
  |--- retry ----->|               |
  |               -|-- retry ----->|
  |               -|               |--- processing again...
  |                |<-- response --|  (first response, delivered late)
  |<-- response ---|               |
  |                |<-- response --|  (second response)
  |<-- response ---|               |
```

The client got two responses. The server did the work twice. If that work was "charge this card" or "place this order," you have a problem.

The fix isn't to use a more reliable network. The fix is to design operations to be *idempotent* — safe to execute multiple times with the same effect as executing once. More on that in the fault tolerance chapter.

**What this means in practice:** Every network call needs a timeout. Every network call that can be retried should be designed to handle retries safely. Never assume a call succeeded because you didn't get an error.

---

## Fallacy 2: Latency Is Zero

Local function calls take nanoseconds. Database calls on the same machine take microseconds. A network call to another service takes milliseconds. Depending on what's between you and the other service, it can take *tens* of milliseconds.

That doesn't sound like much until you have a request handler that makes 20 sequential database calls. Each one is 5ms. That's 100ms of just database time, minimum, and that's assuming everything is on the same local network. Add a downstream service call and you're at 150ms. Add another and you're at 200ms.

```
Request comes in
  └─ Auth check (5ms)
  └─ Load user (5ms)
  └─ Load user preferences (5ms)
  └─ Load feature flags (5ms)
  └─ Fetch recommendations (30ms via HTTP)
  └─ Load each recommendation's details (10ms × 5 = 50ms)
  └─ Log the request (5ms)
Total: ~105ms minimum, not counting processing time
```

And that's the happy path, with no retries and no queueing.

**What this means in practice:** Latency accumulates. Sequential calls multiply. Parallel calls help but add complexity. Think about the critical path of every request — the longest sequential chain of operations — and optimize that specifically. Caching and batching exist because latency isn't zero; use them.

---

## Fallacy 3: Bandwidth Is Infinite

This one was more relevant in the 1990s when networks were genuinely slow. But it still bites in predictable ways.

Serializing a large object graph and sending it over the wire costs more than you think. Sending a 50-field database record when you need 3 fields wastes bandwidth *and* CPU on serialization and deserialization. Sending the same data to 100 subscribers instead of once puts load on the network that you didn't account for.

The more interesting modern version of this fallacy is **chatty interfaces**. A REST API that requires 10 round-trips to get the data needed to render one screen isn't just slow because of latency — each round-trip also consumes bandwidth proportional to the payload size. At scale, this becomes a significant cost.

**What this means in practice:** Design APIs around the data consumers need, not around the entities in your data model. GraphQL exists partly because of this fallacy. Paginate large result sets. Don't send the whole object when you only need the ID.

---

## Fallacy 4: The Network Is Secure

The network is not secure. Traffic can be intercepted. Services can be spoofed. A caller claiming to be your authentication service might not be.

The good news is that the industry has mostly internalized this one for public-facing traffic. HTTPS is the default. TLS is standard.

The bad news is that it's still widely violated for internal traffic. "It's fine, it's all behind the firewall." Until it isn't — because someone gets in, or an employee misbehaves, or a misconfigured cloud security group exposes an internal service.

Zero-trust networking exists because "behind the firewall" is not a security model, it's an assumption. Every service-to-service call should authenticate. Every piece of data should be encrypted in transit. Treat internal traffic with the same skepticism as external traffic.

**What this means in practice:** mTLS for service-to-service communication. Don't trust network location as a substitute for authentication. Assume breach — design systems that limit what an attacker can do even if they get access to your internal network.

---

## Fallacy 5: Topology Doesn't Change

The network layout you have today will not be the network layout you have in six months. Services get added. Services get removed. Load balancers get reconfigured. Cloud instances get terminated and replaced. IP addresses change.

Code that hardcodes IP addresses is the obvious violation, but the more subtle violation is code that assumes stable routing. If Service A always talks to Service B via a fixed path, and that path changes due to a load balancer reconfiguration, A might not notice. Or it might start getting errors it doesn't understand.

Service discovery exists to handle this. DNS exists to handle this. Health checks and circuit breakers exist to handle the case where a formerly reachable endpoint is no longer there.

**What this means in practice:** Use service discovery or DNS, not hardcoded addresses. Assume your topology will change and build graceful handling for the case where a previously reliable endpoint is no longer available.

---

## Fallacy 6: There Is One Administrator

This one is about operational reality. In a system with a single administrator, you can coordinate changes. You can push a config change and know it's applied everywhere. You can reason about the state of the system because there's one authoritative view.

In a distributed system — especially one with multiple teams, multiple services, and multiple deployment pipelines — there are many administrators, and they don't always coordinate.

Team A deploys a new version of Service X at 2pm. Team B is planning a database migration at 3pm. Nobody told Team A that the migration changes a column that Service X reads. Now there's an incident at 3:05pm and two teams are in a Slack channel trying to figure out who broke what.

This is Conway's Law meeting distributed systems. The system's architecture will mirror the communication structure of the organization building it. That's not a platitude — it's a real constraint that affects how you design.

**What this means in practice:** API versioning. Schema migrations that maintain backwards compatibility. Contract testing between services. Runbooks that describe dependencies. Change management that crosses team boundaries.

---

## Fallacy 7: Transport Cost Is Zero

Related to the bandwidth fallacy, but distinct. Sending a message across a network isn't just about the bits. There's CPU time to serialize and deserialize. There's the overhead of establishing a connection. There's the cost of encryption/decryption. There's memory allocation for buffers. There's the latency of the network stack itself.

All of this is cheap per message. None of it is zero. At scale, it adds up.

A system that makes a network call for every event processed is going to have very different performance characteristics than one that batches events and processes them together. This isn't just about latency — it's about total compute cost.

The classic example is n+1 queries: loading a list of objects and then making one database call per object to fetch a related entity. It works fine with 10 objects. With 10,000 objects, you've just made 10,000 individual database calls when one batched query would have sufficed.

**What this means in practice:** Batch where you can. Connection pooling is important. Profile the actual cost of your serialization layer — protobuf and JSON have meaningfully different costs. Consider the transport cost when deciding whether to call a service or bundle its functionality.

---

## Fallacy 8: The Network Is Homogeneous

The network is not one thing. It's composed of multiple segments with different characteristics: different physical media, different speeds, different reliability profiles, different MTU sizes, different routing rules.

The link between your application server and its database is not the same as the link between your datacenter and your CDN. The connection across an ocean is not the same as the connection within a datacenter. Your VPN connection home has different characteristics than the office network.

This matters for several reasons:

- **MTU fragmentation:** Packets too large for a segment get fragmented. Fragmented packets can get lost differently than whole packets. This causes bizarre, intermittent failures that are very hard to debug.
- **Latency variance:** Different paths have wildly different latency profiles. Traffic routing changes can suddenly change your p99 latency.
- **Reliability variance:** Some network segments are more reliable than others. Designing as if all segments are equally reliable leads to surprised faces when the undersea cable has issues.

**What this means in practice:** Test your system with realistic network conditions, not local loopback. Use tools that simulate latency, packet loss, and jitter. Don't assume the network between services in the same datacenter is as reliable as local memory access — it isn't.

---

## The Meta-Fallacy

There's an unspoken ninth fallacy underneath all of these: **that you can reason about a distributed system from a single point of view.**

When you write code in a single process, you're god. You can read all the state, control all the execution, and see the whole picture. In a distributed system, every process has a limited, potentially stale view of the world. There is no omniscient observer. The "current state" of the system is not a single thing — it's the union of the current states of all nodes, which may disagree.

This cognitive shift — from a coherent single view to a collection of partial views — is the hardest thing about distributed systems. The fallacies are symptoms of thinking with the wrong mental model. The right mental model is: **assume partial information, assume partial failure, and design for correctness under both.**

---

The fallacies tell you what to worry about. The next chapters tell you how to think about it. We start with the framework that captures the deepest trade-off: CAP.
