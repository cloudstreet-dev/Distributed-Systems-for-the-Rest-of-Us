# Fault Tolerance: Designing for the Failures You Know and the Ones You Don't

Everything fails. Hardware fails. Software fails. Operators make mistakes. Networks have incidents. Cloud providers have outages that knock out three availability zones simultaneously (this has happened). The goal of fault tolerance isn't to prevent failure — it's to ensure that when failure happens, the blast radius is limited, the system degrades gracefully, and recovery is possible.

This chapter is about the concrete patterns and techniques for building systems that fail less badly.

---

## A Taxonomy of Failure

Before designing for failure, be precise about what can fail and how.

### Hardware failures

Disks fail. Estimated annualized failure rate for enterprise SSDs: 0.5-2%. For spinning disks: 2-5%. In a cluster of 1,000 disks, expect 5-50 disk failures per year. This is normal. RAID and replication exist because hardware failure is expected, not exceptional.

Servers fail. Memory fails with bit errors (ECC memory catches most of these). Network interface cards fail. Power supplies fail. Datacenter power fails (and sometimes the backup generator doesn't start).

### Software failures

Programs crash. Processes run out of memory and get killed by the OS. Bugs appear under edge conditions that testing didn't cover. Libraries have concurrency bugs that only trigger under specific scheduling conditions. Garbage collectors pause for 10 seconds and everything downstream times out.

### Operator failures

Configurations get changed. The wrong service gets restarted. A deployment pushes a bad version. A database migration locks a table for longer than expected. An autoscaling rule triggers during a traffic spike and scales down instead of up.

### Cascading failures

The most insidious: a failure in one component causes failures in others. A slow database causes queries to pile up, which increases memory usage, which causes the application server to slow down, which causes the load balancer to mark it unhealthy, which concentrates load on fewer servers, which causes them to slow down too. A cascade can take a system from "one slow node" to "total outage" in minutes.

---

## Idempotency: The Foundation of Safe Retries

Almost every fault tolerance technique involves retrying operations. Retries only work safely if operations are **idempotent** — producing the same result whether executed once or many times.

Consider two operations:

```
// NOT idempotent:
account.balance += 10

// Idempotent:
account.balance = 100
```

If you execute the first operation twice, you've added 20 instead of 10. If you execute the second operation twice, you get the same result as executing it once.

### Making operations idempotent

**Idempotency keys**: Assign each operation a unique ID. The server stores which IDs it has processed. On receipt, check: have I seen this ID? If yes, return the stored result without re-executing. If no, execute and store the result.

```python
def process_payment(payment_id: str, amount: int, account_id: str):
    # Check if we've already processed this payment
    existing = db.query(
        "SELECT result FROM processed_payments WHERE id = ?",
        payment_id
    )
    if existing:
        return existing.result  # Idempotent: return stored result

    # Execute the actual operation
    result = charge_account(account_id, amount)

    # Store the result atomically
    db.execute(
        "INSERT INTO processed_payments (id, result) VALUES (?, ?)",
        payment_id, result
    )

    return result
```

This pattern requires generating a unique ID per logical operation (not per attempt). The ID should be generated on the client side before the first attempt, and reused on retries.

**Natural idempotency**: Some operations are naturally idempotent. Setting a key to a value is idempotent. Deleting a record that might not exist (and returning success either way) is idempotent. Incrementing a counter is not.

**Conditional operations**: "Set X to 5 if it's currently 3" is idempotent — it only succeeds if the precondition holds. Using optimistic locking / compare-and-swap (CAS) operations makes many things idempotent.

---

## Retries: When to Retry and When Not To

The naive retry: if a request fails, wait a moment and try again.

The problems:
1. **Retrying a non-idempotent operation** causes the operation to execute multiple times
2. **Retrying immediately** puts more load on a system that's already struggling
3. **Retrying non-retryable errors** wastes time (a 400 Bad Request won't succeed on retry)
4. **Infinite retries** without a circuit breaker can mask outages and fill up queues

### What to retry

**Retry transient errors**: network timeouts, connection resets, 503 Service Unavailable, 429 Too Many Requests (after the indicated backoff).

**Don't retry permanent errors**: 400 Bad Request, 401 Unauthorized, 404 Not Found, 422 Validation Error. These won't succeed on retry.

**Maybe retry**: 500 Internal Server Error (could be transient, could be a bug). 408 Request Timeout (might succeed on retry with different timing).

### Exponential backoff with jitter

Don't retry immediately. Wait, then retry. Wait longer, then retry again. Each successive wait should be longer than the last.

Why? If 1,000 clients are all timing out and all retry at the same time, you've just generated a second wave of requests that's just as large as the first. The system was already overwhelmed. Now it's being hit by 1,000 retries simultaneously. This is called a **thundering herd**.

Exponential backoff spreads retries out over time:

```python
import random
import time

def retry_with_backoff(operation, max_attempts=5, base_delay=1.0, max_delay=60.0):
    for attempt in range(max_attempts):
        try:
            return operation()
        except TransientError as e:
            if attempt == max_attempts - 1:
                raise

            # Exponential backoff: 1s, 2s, 4s, 8s, 16s (capped at max_delay)
            delay = min(base_delay * (2 ** attempt), max_delay)

            # Add jitter: randomize within [delay/2, delay]
            # This spreads retries from different clients
            jittered_delay = delay * (0.5 + random.random() * 0.5)

            time.sleep(jittered_delay)
```

The jitter is important. Without it, all clients that started failing at the same time will retry at the same times. With jitter, their retry windows are spread randomly.

---

## Timeouts: How Long Is Long Enough?

Every network call needs a timeout. Every one. A call without a timeout can hang indefinitely, holding resources (threads, connections, memory) while the caller waits for something that will never come.

Choosing a timeout:

1. **Profile the normal case.** If your p99 latency is 200ms, a timeout of 5 seconds is probably too generous — you're waiting 25x longer than usual for failure. A timeout of 500ms might be too tight (you'd fail 1% of healthy requests). Something like 2-3x your p99 is a reasonable starting point.

2. **Consider what's downstream.** If Service A calls Service B which calls Service C, set A→B timeout > B→C timeout + B's processing time. If B times out calling C, it needs time to return a response to A before A times out.

3. **Set timeouts at multiple levels.** Connection timeout (how long to wait for the connection to establish) vs. read timeout (how long to wait for data after the connection is established) vs. total timeout (maximum total time).

4. **Tune over time.** Timeouts should be informed by real latency data. Set up percentile dashboards, observe p99/p99.9, and set timeouts accordingly.

### The timeout cascade problem

```
User request (30s timeout)
  └─ Service A calls B (25s timeout)
       └─ Service B calls C (20s timeout)
            └─ Service C is down
                 [C hangs for 20s]
            Service B times out after 20s, returns error to A
       Service A has only 5s left, tries one retry, times out
  User experiences ~25s wait before getting an error
```

This is a cascade timeout: each layer's slow failure eats into the timeout budget of its callers. The solution: set tighter timeouts at lower levels, fail fast, and propagate deadlines (a hint about when the original caller will give up anyway).

**Deadline propagation**: Pass the original request's deadline through the call chain. If a downstream service receives a request with 100ms remaining before deadline, it should fail immediately rather than starting work it won't complete in time. gRPC supports this natively via context propagation.

---

## Circuit Breakers: Failing Fast

The circuit breaker pattern is named after the electrical device that disconnects a circuit when it detects excessive current — protecting the rest of the circuit from a fault.

In distributed systems: when a service is failing repeatedly, stop calling it for a while. Fail fast and let it recover.

### Circuit breaker states

```
         ┌──── too many failures ──────┐
         │                             ▼
    [CLOSED]                       [OPEN]
    (normal)                   (fail immediately)
         ▲                             │
         │                    timeout expires
         │                             ▼
         └── success ──── [HALF-OPEN]
                          (probe with one request)
```

**CLOSED**: Normal operation. Requests flow through. Failures are counted in a sliding window.

**OPEN**: Failures exceeded threshold. Requests fail immediately without even attempting the call. The downstream service gets a chance to recover without being hammered by retries.

**HALF-OPEN**: After a timeout, allow one request through. If it succeeds, close the circuit. If it fails, return to OPEN and reset the timer.

```python
class CircuitBreaker:
    def __init__(self, failure_threshold=5, timeout=30.0, success_threshold=2):
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.success_threshold = success_threshold

        self.failure_count = 0
        self.success_count = 0
        self.state = "CLOSED"
        self.opened_at = None

    def call(self, operation):
        if self.state == "OPEN":
            if time.time() - self.opened_at > self.timeout:
                self.state = "HALF_OPEN"
                self.success_count = 0
            else:
                raise CircuitOpenError("Circuit breaker is open")

        try:
            result = operation()
            self._on_success()
            return result
        except Exception as e:
            self._on_failure()
            raise

    def _on_success(self):
        if self.state == "HALF_OPEN":
            self.success_count += 1
            if self.success_count >= self.success_threshold:
                self.state = "CLOSED"
                self.failure_count = 0
        elif self.state == "CLOSED":
            self.failure_count = 0

    def _on_failure(self):
        self.failure_count += 1
        if self.failure_count >= self.failure_threshold:
            self.state = "OPEN"
            self.opened_at = time.time()
```

Circuit breakers are implemented in libraries like Resilience4j (Java), Polly (.NET), and available as service mesh features in Envoy/Istio (where they're called "outlier detection").

---

## Bulkheads: Containing Blast Radius

On a ship, bulkheads are watertight compartments. If one compartment floods, the bulkhead prevents it from flooding the rest of the ship.

In distributed systems, bulkheads isolate failures in one part of the system from cascading to others.

### Thread pool bulkheads

Instead of one shared thread pool for all downstream calls, use separate pools:

```
Incoming request thread pool: 100 threads
├─ Service A thread pool: 20 threads
├─ Service B thread pool: 20 threads
├─ Database thread pool: 30 threads
└─ Other: 30 threads
```

If Service A is slow and backs up all 20 threads, Service B calls still have their own 20 threads available. Without bulkheads, Service A's slowness would fill up all 100 threads and take down Service B too.

### Connection pool bulkheads

Same principle for database/HTTP connection pools. Separate pools for separate services. A connection leak to Service A doesn't exhaust the connections to Service B.

### Rate limiting as a bulkhead

Limit the number of requests from any single source. Prevents one bad client from consuming all your capacity at the expense of others.

---

## Graceful Degradation

Sometimes, rather than failing completely, a system can return a degraded but useful response.

**Examples**:

- A recommendation service is down → show popular items instead of personalized recommendations
- The user preference service is slow → show the default UI without personalization
- A non-critical analytics write fails → log the error, return success to the user, investigate later
- Full search is unavailable → fall back to basic string matching

Graceful degradation requires explicitly deciding what's critical and what's optional in each request path, then designing fallbacks for optional components.

```python
def get_user_homepage(user_id):
    # Critical: must succeed or return error
    user = user_service.get_user(user_id)

    # Optional: show default if fails
    try:
        recommendations = recommendation_service.get(user_id, timeout=0.5)
    except Exception:
        recommendations = cache.get("popular_items", default=[])

    # Optional: skip if fails
    try:
        notifications = notification_service.get(user_id, timeout=0.3)
    except Exception:
        notifications = []

    return render_homepage(user, recommendations, notifications)
```

The key discipline: be explicit about which failures are acceptable and which aren't. "Swallow all exceptions" is not graceful degradation — it's hiding failures.

---

## Health Checks and Readiness Probes

A service that's running but not healthy is often worse than a service that's down — it accepts requests and fails them, rather than being routed around.

**Liveness probe**: Is this process alive? If not, restart it.

**Readiness probe**: Is this process ready to serve traffic? If not, stop sending it traffic.

These are different. A service starting up might be alive but not ready (still loading caches). A service that's overwhelmed might be alive but temporarily not ready. A service in an unrecoverable error state should fail its liveness probe to trigger a restart.

```
GET /healthz
→ 200 if process is alive and functional
→ 503 if process should be restarted

GET /readyz
→ 200 if process can serve traffic
→ 503 if process is alive but not ready (starting, draining, overloaded)
```

Kubernetes, load balancers, and service meshes all use these to route traffic intelligently.

---

## Observability: You Can't Fix What You Can't See

Fault tolerance includes the ability to diagnose failures when they happen.

**Structured logging**: Log events as structured data (JSON) not free-text strings. This makes logs queryable. Log correlation IDs so you can trace a request through multiple services.

**Metrics**: Track the four golden signals:
1. **Latency**: How long do requests take? (p50, p95, p99)
2. **Traffic**: How many requests per second?
3. **Errors**: What fraction of requests fail?
4. **Saturation**: How full is your most constrained resource (CPU, memory, connection pool)?

**Distributed tracing**: Track a request's journey through multiple services. Each service adds a span to the trace. The trace shows the full call tree, with timing at each step. Jaeger, Zipkin, and OpenTelemetry are common implementations.

**Alerting on symptoms, not causes**: Alert on "error rate > 5%" (symptom that users are experiencing problems). Don't alert on "CPU > 80%" unless that CPU usage is actually causing user-facing problems. Alert fatigue is a real operational hazard.

---

## Chaos Engineering: Breaking Things on Purpose

The most reliable way to know your fault tolerance works is to test it.

**Chaos engineering** is the practice of deliberately introducing failures into production (or production-like environments) to verify that systems handle them gracefully and to discover failure modes before they discover you.

Netflix's Chaos Monkey randomly terminates production instances. Their reasoning: instances will fail eventually; it's better to fail on a Tuesday afternoon when engineers are watching than at 2am on a Friday.

Starting small:
- Kill a single instance and verify failover works
- Simulate network latency between services and verify timeouts/circuit breakers trigger
- Fill up a disk and verify error handling
- Cut a database connection pool in half and observe the effect

The goal isn't to cause outages — it's to verify that your fault tolerance actually works before you need it in anger.

---

## The Production Moment

Here's the cascade failure that this chapter exists to prevent:

Your recommendation service starts responding slowly (a bad deploy, a database lock, whatever). Your application calls it synchronously in every page load. The slow calls pile up, consuming threads. Your thread pool fills up. New requests can't be served — they wait for a thread that's busy waiting for the recommendation service. Your application starts returning 503s. Users retry. More requests. More 503s. Your load balancer marks your app servers as unhealthy. Traffic routes to fewer servers. Those servers are now overloaded. They start failing too.

In five minutes, you went from "recommendations are slow" to "entire application is down."

With a circuit breaker on the recommendation service, that slow call would have tripped the breaker after a few failures. Subsequent calls would fail immediately (not after waiting 5 seconds). The thread pool stays healthy. The rest of the application keeps running. Users see "we couldn't load recommendations" but everything else works.

With a bulkhead, the recommendation thread pool fills up, but that doesn't affect the rest of the application's threads.

With graceful degradation, users see popular items instead of personalized recommendations and never notice an outage at all.

That's the difference between a fault-tolerant system and one that isn't.

Next: the specific failure mode that causes the most philosophical distress in distributed systems — network partitions, where both sides are running, neither can see the other, and neither can tell which one has the correct view of reality.
