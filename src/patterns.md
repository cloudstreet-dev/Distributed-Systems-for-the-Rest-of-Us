# Patterns That Work: Sagas, Outbox, Circuit Breakers, and Friends

Previous chapters have been about problems. This one is about solutions — or more precisely, about the patterns that smart engineers have repeatedly converged on when solving distributed systems problems. These aren't novel ideas; they're battle-tested approaches with known trade-offs.

If chapters 1-9 are the theory, this is the practice.

---

## The Transactional Outbox Pattern

**The problem it solves**: You need to perform a database write *and* publish an event/send a message, and you need both to happen atomically (either both succeed or neither does).

This comes up constantly. You save a new order to the database and need to publish an `OrderCreated` event to a queue. If the database write succeeds but the message publish fails, your system is inconsistent. If the message publish succeeds but the database write fails, you've published a lie.

The naive solution — write to DB, then publish — breaks because these are two separate operations with no transaction spanning them:

```
// This is broken
db.save(order)  // succeeds
queue.publish(OrderCreated)  // network error!
// order exists in DB, event never published
```

**The outbox pattern**:

Write both to the database, in the same transaction:

```sql
BEGIN TRANSACTION;

INSERT INTO orders (id, ...) VALUES (...);
INSERT INTO outbox (id, event_type, payload, published)
  VALUES (uuid(), 'OrderCreated', '{"orderId": ...}', false);

COMMIT;
```

A separate process (the "outbox poller" or "relay") reads unpublished outbox entries and publishes them to the queue:

```python
def outbox_relay():
    while True:
        events = db.query("""
            SELECT * FROM outbox
            WHERE published = false
            ORDER BY created_at
            LIMIT 100
            FOR UPDATE SKIP LOCKED
        """)

        for event in events:
            try:
                queue.publish(event.event_type, event.payload)
                db.execute(
                    "UPDATE outbox SET published = true WHERE id = ?",
                    event.id
                )
            except Exception:
                log.error(f"Failed to publish {event.id}, will retry")

        time.sleep(0.1)  # poll interval
```

The guarantee: if the transaction commits, the event is in the outbox. The relay will publish it eventually, retrying if necessary. Because the relay can retry (and the queue consumer must be idempotent), this gives you at-least-once delivery with the ordering guarantees of your database.

**For high-throughput systems**, replace polling with Change Data Capture (CDC): tools like Debezium tail the database's replication log (binlog for MySQL, WAL for Postgres) and publish events for each committed change. Zero polling overhead; events published in near-real-time.

---

## Saga Pattern: Distributed Transactions Without 2PC

**The problem it solves**: A business operation spans multiple services, each with its own database. You need the operation to be consistent — either all services complete their step, or the operation is rolled back across all of them.

The classic example: placing an order requires:
1. Reserving inventory (Inventory Service)
2. Charging the payment method (Payment Service)
3. Creating the shipment (Shipping Service)
4. Sending the confirmation email (Notification Service)

If payment succeeds but inventory reservation fails, you've charged a customer for something you can't ship. If shipment succeeds but payment fails, you've shipped for free. You need transactional guarantees across services that don't share a database.

Two-phase commit across services is technically possible but terrible in practice: it requires all services to implement 2PC, creates tight coupling, and leaves the system blocked if the coordinator fails.

**Sagas** take a different approach: decompose the distributed transaction into a sequence of local transactions, each with a corresponding **compensating transaction** that undoes it.

```
Place Order Saga:
    Step 1: Reserve inventory        → compensate: release reservation
    Step 2: Charge payment           → compensate: refund charge
    Step 3: Create shipment          → compensate: cancel shipment
    Step 4: Send confirmation email  → compensate: (none / send cancellation)
```

If any step fails, execute the compensating transactions for all completed steps in reverse order:

```
Step 1: Reserve inventory       ✓
Step 2: Charge payment          ✓
Step 3: Create shipment         ✗ (failure!)
→ Compensate step 2: Refund charge
→ Compensate step 1: Release reservation
Saga complete (rolled back)
```

### Choreography vs Orchestration

There are two approaches to implementing sagas:

**Choreography**: Each service publishes events. Other services listen for events and take the next action.

```
Order Service publishes OrderCreated
  → Inventory Service hears it, reserves inventory, publishes InventoryReserved
    → Payment Service hears it, charges card, publishes PaymentProcessed
      → Shipping Service hears it, creates shipment
```

Advantages: decoupled, no single point of failure.
Disadvantages: the business logic is distributed across services, hard to see the full flow, complex to debug.

**Orchestration**: A central "saga orchestrator" tells each service what to do next.

```
Order Orchestrator:
  1. Call Inventory Service: reserve(orderId)
  2. Call Payment Service: charge(orderId)
  3. Call Shipping Service: create_shipment(orderId)
  if any step fails:
  4. Call Payment Service: refund(orderId)  (if charged)
  5. Call Inventory Service: release(orderId)  (if reserved)
```

Advantages: business logic is in one place, easy to visualize and debug, easier to implement complex compensation logic.
Disadvantages: the orchestrator is a central point of failure (mitigated by making it stateless and persisting state to a database).

**The orchestration implementation**: Store the saga state in a database. On each step completion (success or failure), update the state. The orchestrator can be restarted from any point — it just picks up from the last saved state.

```python
class OrderSaga:
    def __init__(self, db, inventory_svc, payment_svc, shipping_svc):
        self.db = db
        self.inventory = inventory_svc
        self.payment = payment_svc
        self.shipping = shipping_svc

    def execute(self, order_id):
        state = self.db.get_or_create_saga_state(order_id)

        if state.step < 1:
            result = self.inventory.reserve(order_id)
            if not result.success:
                return self.compensate(order_id, step=0)
            self.db.update_saga_state(order_id, step=1, inventory_reservation_id=result.id)

        if state.step < 2:
            result = self.payment.charge(order_id)
            if not result.success:
                return self.compensate(order_id, step=1)
            self.db.update_saga_state(order_id, step=2, payment_id=result.id)

        if state.step < 3:
            result = self.shipping.create_shipment(order_id)
            if not result.success:
                return self.compensate(order_id, step=2)
            self.db.update_saga_state(order_id, step=3, shipment_id=result.id)

        self.db.update_saga_state(order_id, status="completed")

    def compensate(self, order_id, step):
        state = self.db.get_saga_state(order_id)

        if step >= 2 and state.payment_id:
            self.payment.refund(state.payment_id)

        if step >= 1 and state.inventory_reservation_id:
            self.inventory.release(state.inventory_reservation_id)

        self.db.update_saga_state(order_id, status="compensated")
```

### What sagas don't give you

Sagas provide atomicity (either all steps complete or they're compensated) but not isolation. While a saga is executing, other sagas can see intermediate states. If two sagas both try to reserve the same last unit of inventory, one might succeed and then later fail at payment, while the other was told "no inventory" incorrectly.

This is the **dirty reads** problem in saga terms. Mitigations: semantic locks (mark resources as "reserved" before they're fully committed), process managers that coordinate conflicting sagas, careful ordering of steps (reserve last-item-in-stock at the end, not the beginning).

---

## Circuit Breaker (Detailed Implementation Notes)

Covered in the fault tolerance chapter, but worth expanding on implementation details.

**What counts as a failure?**

- HTTP 5xx responses (server errors)
- Network timeouts
- Connection refused errors

**What doesn't count as a failure?**

- HTTP 4xx responses (client errors — these aren't the downstream service's fault)
- Business logic errors (an order can't be fulfilled because it's invalid — that's not a service failure)

Misclassifying 4xx as failures causes circuit breakers to trip on bad client input, which is not what you want.

**Sliding window vs count-based thresholds**

Count-based: "trip after 5 failures." Simple but doesn't account for recovery time. If the service had 5 failures in the last hour, the circuit would be tripped even though it's been healthy for 55 minutes.

Time-based: "trip if error rate exceeds 50% in the last 10 seconds." More accurate. Requires tracking recent history. Libraries like Resilience4j implement sliding windows.

**Half-open probing strategy**

Don't just send one probe when moving to HALF_OPEN. The single probe might fail by chance, sending you back to OPEN when the service has actually recovered. Consider:
- Sending a configurable number of probes before fully closing
- Using a lower failure threshold to re-open from HALF_OPEN
- Incrementally increasing allowed traffic (10% → 25% → 50% → 100%)

---

## Read-Your-Writes via Sticky Sessions or Version Tokens

**The problem**: You write to a primary database, immediately redirect to a page that reads from a replica. The replica hasn't caught up. Your write is invisible. User reports a bug.

**Solution 1: Sticky reads to primary after writes**

Track a "recently wrote" flag (in a session cookie, JWT claim, or request header). For a short window (e.g., 30 seconds) after a write, route reads to the primary.

```python
def handle_write(user_id, data):
    db.primary.write(data)
    session["wrote_at"] = time.time()
    return redirect(...)

def handle_read(user_id):
    wrote_recently = (
        "wrote_at" in session and
        time.time() - session["wrote_at"] < 30
    )
    db_to_use = db.primary if wrote_recently else db.replica
    return db_to_use.read(user_id)
```

Disadvantage: routes potentially expensive reads to the primary, partially defeating the purpose of replicas.

**Solution 2: Replication position tokens**

After a write, capture the WAL/replication position (a log sequence number). Pass this as a token to subsequent requests. On the replica, wait until the replica has caught up to at least that position before serving the read.

```
Client writes → Primary returns LSN: 12345
Client next request includes token: min_lsn=12345
Replica checks: is my current LSN ≥ 12345?
  If yes: serve read
  If no: wait briefly, recheck (or route to primary)
```

This is more precise than time-based routing. PostgreSQL exposes its WAL LSN; MySQL exposes binlog coordinates. CockroachDB has built-in follower reads with this mechanism.

---

## Event Sourcing

**The pattern**: Instead of storing the current state of an entity, store the sequence of events that produced that state. Current state is computed by replaying events.

```
Traditional approach:
  accounts table: { id: 1, balance: 150 }

Event sourcing approach:
  events table:
    { account_id: 1, type: "Deposit", amount: 100, at: T1 }
    { account_id: 1, type: "Deposit", amount: 100, at: T2 }
    { account_id: 1, type: "Withdrawal", amount: 50, at: T3 }

  Current balance: sum events to get 150
```

**Advantages**:
- Full audit trail: you know exactly how you got to current state
- Time travel: compute state at any historical point
- Event replay: rebuild projections from scratch (useful for adding new views)
- Natural fit with CQRS and event-driven architecture

**Disadvantages**:
- Querying current state requires aggregation (mitigated with snapshots and read models)
- Schema evolution of events is hard (old events must still be interpretable)
- Mental model shift — requires discipline to work with
- Storage grows without bound unless snapshots and compaction are implemented

Event sourcing is not universally applicable, but for domains where audit, history, and the ability to re-derive state are important (financial systems, legal records, collaborative editing), it's a natural fit.

---

## CQRS: Command Query Responsibility Segregation

**The pattern**: Separate the read path from the write path. Write operations (commands) update an authoritative data store. Read operations (queries) read from one or more separate read models optimized for specific queries.

```
Write path:
  Command → validate → execute → update write store → publish events

Read path:
  Event listener → update read model(s)
  Query → read from read model
```

**Why it helps**: Write stores don't need to be optimized for queries. Read models can be denormalized, pre-aggregated, and optimized for specific access patterns. Multiple read models can be maintained for different query patterns.

```
Write store: normalized relational tables
Read model A: denormalized table for customer order history (optimized for "show me my orders")
Read model B: Elasticsearch index for full-text order search
Read model C: materialized view for monthly revenue reporting
```

**The trade-off**: Read models are derived from events and may lag behind the write store. Queries may see slightly stale data. This is the eventual consistency in CQRS.

CQRS pairs naturally with event sourcing (events update the read models) but can also be used with traditional state-based persistence.

---

## Sidecar/Proxy Pattern for Cross-Cutting Concerns

**The pattern**: Run a proxy process alongside each service instance. The proxy handles cross-cutting concerns (retries, circuit breaking, load balancing, tracing, mutual TLS) so the service doesn't have to.

```
Service code:
  "Call http://payment-service/charge"
  (no retry logic, no circuit breaker, no tracing)

Sidecar proxy intercepts:
  - Adds distributed trace headers
  - Enforces mTLS
  - Applies retry policy
  - Applies circuit breaker
  - Load balances across payment-service instances
  - Reports metrics

Service sees a simple HTTP call.
The sidecar handles everything else.
```

This is the service mesh pattern: Envoy, Linkerd, Consul Connect. The infrastructure concerns are handled at the infrastructure level, not in application code.

**The trade-off**: Operational complexity. You're now running two processes per service instance. The proxy is another thing that can fail, another thing to configure, another thing to understand. The benefit (uniform policy enforcement, language-agnostic implementation) is real, but the cost is real too.

---

## Inbox Pattern (Idempotent Message Consumption)

Companion to the outbox pattern. When consuming messages from a queue, you need to handle duplicates — the queue might deliver the same message more than once (at-least-once delivery).

**The pattern**: Before processing a message, check if you've already processed it. If yes, skip processing (return success). If no, process it and record that you've done so.

```python
def handle_message(message):
    message_id = message.headers["message-id"]

    # Check if already processed (atomic with processing via DB transaction)
    with db.transaction():
        if db.query("SELECT 1 FROM processed_messages WHERE id = ?", message_id):
            return  # Already processed, skip

        # Process the message
        process_business_logic(message.body)

        # Record as processed
        db.execute(
            "INSERT INTO processed_messages (id, processed_at) VALUES (?, NOW())",
            message_id
        )
```

This requires messages to have a stable, unique ID. Most message brokers (Kafka, SQS, RabbitMQ) provide message IDs. For Kafka, the combination of topic + partition + offset is a unique identifier.

---

## Patterns Summary

| Pattern | Problem Solved | Key Trade-off |
|---------|---------------|---------------|
| Transactional Outbox | Atomic DB write + event publish | Added latency; requires outbox poller |
| Saga (choreography) | Distributed transactions | Hard to visualize; complex compensation |
| Saga (orchestration) | Distributed transactions | Orchestrator is central point of concern |
| Circuit Breaker | Cascade failure prevention | Configuration complexity; false trips |
| Idempotency Key | Safe retries | Requires unique ID generation; storage |
| Read-Your-Writes | Replication lag consistency | Routes load to primary |
| Event Sourcing | Audit trail, history, replay | Storage growth; schema evolution |
| CQRS | Read/write optimization | Eventual consistency on reads |
| Sidecar/Service Mesh | Cross-cutting concerns | Operational complexity |
| Inbox / Deduplication | Idempotent message consumption | Storage for processed message IDs |

None of these patterns are magic. Each adds complexity in exchange for solving a specific problem. The discipline is matching the pattern to the problem — not reaching for a saga when a simple transaction would do, and not "just using a transaction" when you actually need a saga.

---

Next: the chapter that ties everything together — how to make actual architecture decisions for your actual system, with your actual constraints.
