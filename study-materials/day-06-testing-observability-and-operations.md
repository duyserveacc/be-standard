# Day 6: Testing, Observability, and Operations

This lesson expands Day 6 of the
[backend learning roadmap](../BACKEND_LEARNING_ROADMAP.md#day-6-tests-observability-and-operations).
It turns correctness and operability claims into repeatable evidence.

## Learning outcomes

By the end of this lesson, you should be able to:

- choose unit, integration, and acceptance tests according to the risk;
- build deterministic tests with explicit clocks, IDs, and fixtures;
- distinguish logs, metrics, traces, and audit events;
- choose bounded telemetry attributes and metric labels;
- define latency, traffic, error, and saturation indicators;
- implement liveness, readiness, and bounded graceful shutdown;
- apply deadlines, concurrency limits, and safe retry rules;
- diagnose validation, application, and database failures from telemetry;
- connect an actionable alert to a runbook.

## 1. Tests answer different questions

No single test style proves the whole service.

| Test type | Main question | Real boundaries? | Typical speed |
| --- | --- | --- | --- |
| Unit | Does one rule or component behave correctly? | No | Fast |
| Handler | Is HTTP translated correctly? | In-process HTTP only | Fast |
| Integration | Does code work with PostgreSQL or another real dependency? | Yes | Moderate |
| Acceptance | Does a user-visible workflow work through the public interface? | Process-level | Slowest |

Use the lowest-cost test that can reproduce the risk. Quantity validation needs a
unit test. A unique constraint, rollback, or row lock needs PostgreSQL. Public JSON
and authentication behavior need handler or acceptance tests.

Avoid an oversized end-to-end suite for every branch. Slow, broad tests are harder
to diagnose and make feedback less reliable.

## 2. Table-driven tests make cases visible

When many inputs exercise the same behavior, use a table:

```go
func TestValidateQuantity(t *testing.T) {
    tests := []struct {
        name    string
        quantity int64
        wantErr error
    }{
        {name: "positive", quantity: 1},
        {name: "zero", quantity: 0, wantErr: ErrInvalidQuantity},
        {name: "negative", quantity: -1, wantErr: ErrInvalidQuantity},
    }

    for _, test := range tests {
        t.Run(test.name, func(t *testing.T) {
            err := validateQuantity(test.quantity)
            if !errors.Is(err, test.wantErr) {
                t.Fatalf("error = %v, want %v", err, test.wantErr)
            }
        })
    }
}
```

Give each case a behavioral name. Assert stable outputs and error identities rather
than complete incidental strings.

Parallel subtests are appropriate only when fixtures and dependencies are isolated.
Do not add `t.Parallel()` reflexively to database tests sharing tables or limits.

## 3. Prefer small fakes over behavior-heavy mocks

A fake implements the narrow consumer interface and records useful calls:

```go
type fakePlacementRepository struct {
    request PlacementRequest
    order   Order
    err     error
    calls   int
}

func (f *fakePlacementRepository) Place(_ context.Context, request PlacementRequest) (Order, error) {
    f.calls++
    f.request = request
    return f.order, f.err
}
```

This verifies application coordination without reproducing PostgreSQL. If a fake
starts implementing locks, SQL-like filtering, and transactions, the test is
validating another invented database rather than the real boundary.

Use a test double only for an interface the production consumer genuinely needs.
Do not change production APIs solely to satisfy a mocking framework.

## 4. Control time, identity, and randomness

Tests become flaky when they depend on wall-clock timing or random values. Inject
small deterministic capabilities:

```go
type fixedClock struct {
    now time.Time
}

func (c fixedClock) Now() time.Time { return c.now }

type sequenceIDs struct {
    values []string
    index  int
}

func (s *sequenceIDs) NewID() string {
    value := s.values[s.index]
    s.index++
    return value
}
```

Production uses a real UTC clock and cryptographically appropriate ID generator;
tests use known values. For timeout tests, prefer context cancellation and channels
that signal exact state transitions over arbitrary sleeps.

Fixtures should be minimal and explicit. A test should create only the rows it
needs, name important values, and clean up or isolate its state. Large shared fixture
dumps obscure why a case passes.

## 5. Logs, metrics, and traces answer different questions

### Logs: discrete events

Logs describe something that happened. Use structured JSON fields:

```json
{
  "level": "INFO",
  "message": "HTTP request completed",
  "request_id": "req-123",
  "trace_id": "4f...",
  "method": "POST",
  "route": "/v1/orders",
  "status": 409,
  "duration_ms": 12,
  "error_code": "insufficient_inventory"
}
```

Record the route template, not a raw URL containing IDs. Log an unexpected error at
the boundary that can add request context; avoid logging the same error at every
layer. Preserve the wrapped error for diagnosis while the public response remains
generic.

Security audit events are distinct from access logs. An audit event records who
attempted a sensitive action, which action, target category, outcome, time, and
correlation ID. Apply retention and access controls appropriate to security data.

### Metrics: aggregate trends

Metrics answer questions across many requests:

- request count by route, method, status class, and stable error code;
- latency histogram by route and method;
- in-flight request gauge;
- database pool acquired, idle, total, and acquisition-wait measures;
- dependency operation count and latency;
- order outcomes such as placed or insufficient inventory.

Metric labels must have bounded cardinality. Never use customer ID, order ID,
request ID, trace ID, raw path, error text, or SKU as labels. Unbounded label values
create a new time series per value and can make monitoring expensive or unusable.

### Traces: one request path

A trace connects spans for HTTP handling, application work, and database operations.
Useful span attributes include route template, HTTP method, stable result category,
database system, and named operation. Do not attach SQL parameters, tokens, full
statements with user data, or raw request bodies.

Propagate standard trace context across boundaries. Request ID helps human support
and logs; trace ID connects telemetry. They may be related but are not the same
protocol.

## 6. Instrument at stable boundaries

Instrument HTTP middleware for request duration, status, and in-flight count.
Instrument PostgreSQL adapters for named operations and pool behavior. Add business
metrics only for durable, well-defined outcomes.

Avoid adding telemetry calls throughout pure domain functions. Pass observation
through boundaries or decorators so rules remain testable without an SDK.

Every telemetry field needs an answer to:

- Which operational question does it answer?
- Is its value set bounded?
- Could it contain secrets or personal data?
- Who will use it during an incident?
- What retention and cost does it create?

If no useful answer exists, omit it.

## 7. Service indicators and objectives

Four useful service signals are:

- **Latency:** how long successful and failed requests take as a distribution;
- **Traffic:** request or operation rate;
- **Errors:** proportion of requests failing the defined user outcome;
- **Saturation:** pressure on bounded resources such as workers or database pool.

Use percentiles or histogram buckets, not only averages. An average can hide a slow
tail experienced by many users.

Define an SLI before an SLO. Example:

```text
SLI: proportion of valid POST /v1/orders requests completed without a 5xx response
within 750 ms, measured at the service boundary.

SLO: at least 99.9% over a rolling 30-day window.
```

Clarify exclusions. Invalid 4xx requests normally do not count as server failures,
but they still consume capacity and should be visible.

An error budget is the allowed unsuccessful fraction. Multi-window burn-rate alerts
identify when the budget is being consumed much faster than planned. Choose alert
thresholds only after the SLO and traffic volume are defined.

## 8. Health and graceful shutdown are protocols

Liveness checks only whether the process is alive. Readiness determines whether it
should receive traffic and checks critical dependencies with strict deadlines.

During shutdown:

1. Mark readiness false.
2. Allow the load balancer time to observe the change when the platform requires it.
3. Stop accepting new work.
4. Drain active requests within a deadline.
5. Stop background workers using cancellation.
6. Close pools and telemetry exporters.
7. Exit; do not hang forever.

Test both success and deadline exhaustion. A request that ignores context can prevent
clean drain, so cancellation must reach every blocking database and network call.

## 9. Bound concurrency and dependencies

Timeouts bound duration. Concurrency limits bound how many operations may consume a
resource simultaneously. Both are required.

Useful limits include:

- HTTP header and body bytes;
- active request count for expensive operations;
- database pool connections and acquisition wait;
- page size and batch size;
- goroutine and worker count;
- queue length;
- downstream connection count;
- telemetry export queue.

Reject or shed excess work predictably rather than allowing memory and latency to
grow without bound. A semaphore implemented with a buffered channel can limit a
specific expensive operation; acquisition must honor request context.

## 10. Retry only when all conditions are satisfied

A retry is safe only when:

- the failure is plausibly transient;
- the operation is idempotent or protected by an idempotency mechanism;
- the caller's total deadline permits another attempt;
- the attempt count is strictly bounded;
- backoff includes jitter;
- the retry does not multiply across several layers.

Do not retry validation errors, permission failures, insufficient inventory, or
arbitrary 500 responses. Do not retry an ambiguous order write without an
idempotency key because the first attempt may have committed.

A common backoff shape is exponential growth capped at a maximum plus randomized
jitter. The exact schedule matters less than having a total time budget, a small
attempt limit, and observable attempt outcomes.

Choose one layer to own a retry. If client, API, and database wrapper each retry
three times, one request can cause 27 attempts during an outage.

## 11. Failure drills

Exercise failures deliberately in a local or isolated environment.

### Validation failure

Send malformed or invalid input. Expect:

- stable 400 problem code;
- no database operation span;
- access log with 4xx status and request/trace IDs;
- client-error metric increment;
- no error-level stack trace.

### Server bug

Use a test-only handler or injected failure that returns an unexpected error or
panics. Expect:

- generic 500 response;
- correlated error log with internal cause;
- 5xx metric increment;
- trace marked as error;
- recovery keeps the process alive for a request panic.

### Slow or unavailable database

Pause, disconnect, or point a test environment at an unavailable PostgreSQL. Expect:

- query or acquisition deadline stops the request;
- readiness fails within its own tight timeout;
- liveness remains healthy;
- database span identifies the named operation, not credentials;
- pool wait/saturation and request latency reveal the bottleneck;
- shutdown remains bounded.

### Canceled request

Cancel the request while a database call waits. Expect the context error to reach
the query, work to stop, and the handler not to continue expensive processing for a
client that has gone away.

## 12. Alerts and runbooks

An alert should indicate sustained user impact or imminent exhaustion and lead to a
specific response. Avoid paging on every individual error.

A runbook should contain:

```text
Alert: Order API fast error-budget burn
User impact: Order placement may be failing or exceeding the latency objective.
Dashboards: request outcomes, latency, DB pool, dependency health, deployments.
First checks: scope by route/status; compare deploy time; inspect pool saturation.
Mitigation: stop rollout, reduce traffic, disable noncritical work, or fail over.
Escalation: named owning team and database/platform contact.
Verification: SLI recovers and stays below alert threshold for the defined period.
Follow-up: incident record, contributing causes, and corrective actions.
```

Do not include untested commands that can destroy data. Link every mitigation to
permissions, rollback criteria, and verification.

## 13. Guided build

Complete the work in this order:

1. Add deterministic clock and ID boundaries where tests need them.
2. Review the suite and place each test at the narrowest correct boundary.
3. Add JSON access logs with request ID, trace ID, route, status, duration, and code.
4. Add HTTP count, latency, error, and in-flight metrics.
5. Export database pool state and named operation latency.
6. Trace HTTP and database work with redacted bounded attributes.
7. Implement bounded shutdown for HTTP, workers, pool, and exporters.
8. Run the four failure drills and capture expected evidence.
9. Define one latency/error SLI, SLO, alert, and runbook.

Use OpenTelemetry at external and process boundaries. Keep the vendor/exporter
configuration outside business packages.

## 14. Knowledge check

1. Which behavior requires a database integration test instead of a fake?
2. Why are deterministic clocks better than timing sleeps?
3. What question does each of logs, metrics, and traces answer?
4. Why is order ID unsafe as a metric label?
5. Why should liveness remain healthy during a database outage?
6. What two distinct resource dimensions do timeouts and concurrency limits bound?
7. When is retrying a write safe?
8. Why should only one layer own retries?
9. What makes an alert actionable?

### Expected answers

1. Constraints, SQL, locks, isolation, commit, rollback, and driver behavior.
2. They create exact reproducible state without scheduler and machine-speed variance.
3. Logs explain events, metrics reveal aggregate trends, and traces connect one path.
4. It creates an unbounded time-series dimension and can expose customer data.
5. Restarting the application cannot repair the database and can amplify the outage.
6. Maximum elapsed time and simultaneous resource consumers.
7. When it is idempotent or idempotency-protected, the failure is transient, and
   attempts and total time are bounded.
8. Layered retries multiply attempts and worsen overloaded dependencies.
9. It identifies meaningful impact and links to tested diagnosis, mitigation,
   escalation, and recovery verification.

## 15. Completion evidence

Day 6 is complete when:

- each important risk is covered at the narrowest boundary that can prove it;
- tests do not depend on arbitrary sleeps, wall time, or random identifiers;
- telemetry distinguishes invalid input, an internal failure, and a slow database;
- all log fields, metric labels, and trace attributes are bounded and reviewed for
  sensitive data;
- readiness, liveness, request cancellation, and shutdown behave differently by
  design and are tested;
- retries are bounded, observable, and limited to safe transient operations;
- one SLO alert leads to a usable runbook;
- all configured code, integration, race, vet, and vulnerability checks pass.

Then continue to Day 7.
