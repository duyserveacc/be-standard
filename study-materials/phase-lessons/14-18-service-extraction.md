# Phases 14-18: Service Extraction and Distributed Delivery

Extraction starts only after the earlier phases provide data, failure, and operational
evidence. Each service owns its contract, data, runtime, deployment, and on-call cost.

## Phase 14: Service-Boundary Discovery and First Extraction

### Goal and mental model

A useful service boundary follows a business capability with coherent rules, data,
ownership, and change cadence. Splitting by controllers, tables, or technical layers
creates a distributed monolith: network and deployment cost without autonomy.

Gather evidence before deciding:

- which modules change together in version history;
- which teams or owners need independent releases;
- which workload needs independent scale or failure isolation;
- current call/data coupling and transaction boundaries;
- baseline latency, deployment effort, incidents, and operator load;
- alternatives such as clearer modules inside the monolith.

Notification is a low-risk first extraction because order placement should commit
without waiting for delivery. The service consumes durable `OrderPlaced` facts and
owns templates, preferences, delivery attempts, provider state, and its datastore.
Ordering owns order truth and outbox events.

No service reads another service's tables. Shared database credentials or cross-schema
queries preserve deployment coupling and bypass contracts. When data is needed, use
an owned API, event, or deliberately replicated read model.

Use branch by abstraction: define the existing notification capability, place old
and new implementations behind it, mirror or gradually route controlled traffic,
compare outcomes, and remove the old path only after evidence.

### Guided work

1. Update the context map with ownership, data, calls, events, and team responsibility.
2. Analyze change history and current coupling for extraction candidates.
3. Record baseline latency, availability, deployment, incidents, and maintenance.
4. Write an ADR comparing notification extraction with keeping it in-process.
5. Define the event contract and notification-owned data model.
6. Create independent config, runtime identity, schema/database access, CI, dashboard,
   alert, and runbook.
7. Implement the consumer with inbox dedupe and provider idempotency/reconciliation.
8. Migrate traffic incrementally while the monolith path remains available.
9. Inject notification failure and verify order placement remains healthy.
10. Compare measured benefit and added operational cost with the ADR hypothesis.

### Failure checks

Stop notification, delay its database, redeliver events, and break its provider.
Verify orders still commit and backlog becomes observable. Attempt a cross-service
table read and ensure credentials prevent it. Roll traffic back to the in-process
path without data loss, then complete migration and remove obsolete code.

### Knowledge check and expected answers

1. **What makes a distributed monolith?** Services cannot change independently but
   still pay remote-call, deployment, and operational cost.
2. **Why start with notification?** Its asynchronous failure can be isolated from the
   core order transaction.
3. **Why forbid cross-service schema reads?** They bypass owned contracts and couple
   releases, data changes, security, and operations.

### Completion evidence

The ADR cites real coupling data; notification deploys/fails independently; order
placement survives its outage; no cross-data access exists; incremental migration
and rollback are rehearsed; and benefits plus ongoing operational costs are measured.

## Phase 15: Synchronous Service Communication and the System Edge

### Goal and mental model

Extract inventory only after defining the remote contract and accepting that a local
transaction no longer protects order plus inventory. Every remote call can be slow,
unavailable, duplicated, or ambiguous.

Choose HTTP/JSON when broad compatibility, inspection, and existing tooling matter.
Choose gRPC when strongly typed contracts, streaming, and controlled clients justify
its tooling. Protocol choice does not remove deadlines, compatibility, authorization,
or failure handling.

Set a total request deadline, reserve time for local work and response, and give the
dependency only the remaining budget. Propagate cancellation. Configure bounded
connection pools, response/body limits, and keep-alive behavior.

Retry only transient, idempotent or idempotency-protected calls. One layer owns a
small attempt limit with jitter and total deadline. Never multiply retries at client,
gateway, and service.

Never hold a local database transaction open across the remote call or pretend the
two service databases commit atomically. Before enabling the write path, persist an
order intent in a pending state with a stable reservation operation ID. Confirm only
after inventory reports that exact reservation. If reservation succeeds but local
confirmation fails, or if the response is ambiguous, durable reconciliation must
query by operation ID and either confirm or issue an idempotent release. Do not create
an uncorrelated second reservation or report the order confirmed prematurely. Phase
16 expands this minimum safe bridge into the complete durable saga state machine.

Protect dependencies with distinct tools:

- concurrency limit: caps simultaneous calls;
- bulkhead: isolates resources by dependency/workload;
- circuit breaker: temporarily avoids calls after strong failure evidence;
- load shedding: rejects work that cannot finish within objectives;
- fallback: returns deliberately degraded behavior only when semantically safe.

The gateway owns edge authentication, TLS, routing, request limits, and coarse rate
limits. It must not become the shared location for order/inventory business rules.
Inventory must also authenticate the ordering workload and authorize only its
required operations; network reachability is not trust. Propagate end-user identity
only when the inventory authorization or audit contract requires it, and never use
an unverified forwarded header as a principal.

### Guided work

1. Define inventory ownership and remove direct ordering access to inventory tables.
2. Choose protocol through an ADR with client/tooling/latency evidence.
3. Define reserve/release/query contracts, stable errors, idempotency, and versions.
4. Build the order client with propagated context, remaining-deadline calculation,
   bounded connections, response limits, and stable error mapping.
5. Add authenticated workload identity and least-privilege inventory authorization.
6. Configure runtime service discovery and load balancing.
7. Add one intentional retry policy only for safe transient outcomes.
8. Add dependency concurrency limit, breaker/load-shed policy, and recovery behavior.
9. Route external traffic through an edge proxy with auth and bounded rate limits.
10. Add attempt, latency, status, saturation, and circuit-state metrics.
11. Persist pending intent before reservation, then reconcile ambiguous results and
    successful reservations whose local confirmation fails.
12. Test outage, slow response, overload, recovery, and rolling endpoint changes.

### Failure checks

Make inventory slower than the caller deadline, accept then lose a response, return
malformed/oversized payload, remove all discovered instances, and overload one
instance. Call with missing, expired, and wrong-service workload credentials. Verify
no call exceeds remaining budget, retries do not multiply, goroutine and connection
counts stay bounded, authorization fails closed, accepted inventory cannot be
orphaned by a local confirmation failure, ambiguous acceptance remains pending until
reconciled, and recovery closes the breaker deliberately.

### Knowledge check and expected answers

1. **Why is a per-call timeout not enough?** Several calls and retries can exceed the
   original request budget unless they consume one total deadline.
2. **When is a retry unsafe?** When the first write may have committed and no
   idempotency identity protects repetition.
3. **Why keep business rules out of the gateway?** Shared edge logic couples service
   releases and obscures capability ownership.
4. **Why persist pending intent before reserving?** A remote success and local commit
   cannot be atomic; durable identity/state lets reconciliation confirm or compensate
   without guessing.

### Completion evidence

Inventory owns its datastore and contract; no outbound call is unbounded; retry
ownership and idempotency are explicit; outage/overload produce bounded stable
errors; dependency telemetry explains attempts and saturation; and healthy recovery
is demonstrated.

## Phase 16: Distributed Workflows and Eventual Consistency

### Goal and mental model

Separate databases cannot participate in the former local order transaction without
specialized distributed transactions and their own availability cost. Model a saga:
a durable sequence of local commits with expected intermediate states and explicit
compensation.

Example order state machine:

```text
pending
  -> inventory_reservation_requested
  -> inventory_reserved
  -> confirmed

inventory_reservation_requested -> reservation_failed
inventory_reserved -> cancel_requested -> canceled
timeout/unknown -> needs_reconciliation
```

Compensation is a business action, not time reversal. Releasing inventory may fail,
arrive twice, or happen after another state change. It needs its own idempotency key,
authorization, deadline, retry, and terminal outcome.

Choose orchestration when one durable workflow owner should decide transitions and
provide a clear operator view. Choreography reduces a central coordinator but can
hide the workflow across many event reactions. Use the simplest style that keeps
state and ownership explicit.

Every transition stores current state, version, correlation, attempts, timestamps,
and next action durably. Outbox/inbox protects each event boundary. Optimistic
version checks reject out-of-order transitions. Timeouts are durable scheduled facts,
not process-local timers.

Manual repair is a designed transition with validation, dry-run/preview, audit,
idempotency, and postcondition verification. Direct database edits are not a repair
API.

### Guided work

1. Define all workflow states, commands, events, timeouts, and valid transitions.
2. State acceptable temporary inconsistency and maximum duration for each state.
3. Choose orchestration or choreography in an ADR.
4. Persist saga state/version and next action in ordering.
5. Add reserve and release operation IDs for inventory idempotency.
6. Use outbox/inbox on every asynchronous transition.
7. Add durable timeouts and reconciliation queries.
8. Build an operator view for age, state, last event, next action, and error.
9. Add scoped repair commands with audit and expected-state guards.
10. Test every transition under duplicate, delayed, reordered, and missing messages.

### Failure checks

Crash before and after every local commit and event publish. Delay success beyond a
timeout, then deliver it. Duplicate reservation and compensation. Deliver confirm
before reserved. Keep one workflow stuck beyond objective and repair it using only
the operator path.

### Knowledge check and expected answers

1. **Why is compensation not rollback?** External effects and elapsed business time
   cannot be erased; compensation is a new fallible operation.
2. **Why persist workflow state?** Restarts, retries, timeouts, and operators need a
   durable source of transition truth.
3. **Why model intermediate states publicly to operators?** Eventual consistency
   makes them expected, diagnosable business conditions rather than hidden errors.

### Completion evidence

Every injected transition failure converges to a documented state; duplicate/out-of-
order events preserve invariants; compensation is idempotent and distinguishable;
timeouts are durable; stuck work is alerted and repairable; and temporary consistency
bounds are measured.

## Phase 17: Contract Evolution and Independent Delivery

### Goal and mental model

Independent deployment exists only inside a compatibility window understood by both
provider and consumers. Schema syntax is necessary but insufficient: semantics,
validation, defaults, ordering, error behavior, and timing can break clients.

Use OpenAPI for HTTP, Protocol Buffers for typed RPC when chosen, and AsyncAPI or an
equivalent owned schema for events. Publish ownership, lifecycle, compatibility
rules, and examples with each contract.

Usually compatible changes include adding an optional response/event field and new
error detail that old clients ignore. Common hidden breaks include:

- making optional input required;
- rejecting values previously accepted;
- changing units or field meaning;
- adding an enum value to an exhaustive consumer;
- changing default sorting or pagination stability;
- removing an event field before all consumers stop using it.

Consumer-driven contract tests record the provider behavior a real consumer relies
on. Provider verification detects drift but does not replace component tests,
authorization tests, or operational compatibility.

Deprecation requires owner, replacement, announced date, usage telemetry, consumer
inventory, support window, and removal criteria. Absence of complaints is not proof
of zero use.

### Guided work

1. Write HTTP/RPC/event compatibility rules and supported-version policy.
2. Store published contracts in a versioned repository or registry.
3. Add schema diff checks and provider verification to CI.
4. Capture real consumer expectations including semantic examples.
5. Add one optional field and deploy consumer-first, then provider-first in separate
   rehearsals.
6. Demonstrate CI blocking a removed/renamed field and a semantic expectation break.
7. Mark one behavior deprecated with owner, replacement, date, and telemetry.
8. Detect every remaining consumer and communicate migration.
9. Remove only after the compatibility window and zero-use evidence.
10. Delete compatibility adapters once no supported version needs them.

### Failure checks

Deploy old/new combinations in both orders. Add an enum value to an exhaustive test
consumer. Tighten validation without schema change. Delay an old event in the broker
until after a new consumer deploys. Simulate incomplete usage telemetry and ensure
removal is blocked.

### Knowledge check and expected answers

1. **Why can an additive change break?** Consumers may validate strictly, exhaust
   enums, or depend on previous semantics/defaults.
2. **What do consumer contracts add?** Evidence of the exact provider behavior active
   consumers rely on.
3. **What permits removal?** An elapsed support window plus complete consumer/usage
   evidence, not merely a new version's availability.

### Completion evidence

Contracts and policies are published; CI blocks demonstrated schema and semantic
breaks; both deployment orders work for a compatible change; delayed old events are
handled; deprecation identifies consumers; and removal follows telemetry evidence.

## Phase 18: Distributed Observability and Resilience Validation

### Goal and mental model

Once a workflow crosses services and queues, local logs are insufficient. Propagate
trace context through HTTP/RPC metadata, event envelopes, and job metadata. For
asynchronous work, create a consumer span linked or parented according to the chosen
semantics and preserve correlation across retries.

Baggage crosses boundaries and can spread widely; allowlist only bounded non-sensitive
values. Never propagate credentials, personal data, raw order content, or arbitrary
user strings.

Service metrics show request SLIs and dependency attempts, latency, outcomes,
in-flight work, circuit state, queue age, lag, and saturation. Use stable service,
operation, and result dimensions rather than IDs.

Sampling must retain evidence for errors and slow traces while controlling cost.
Head sampling decides early but may miss later failures; tail sampling can choose
after observing the trace but needs more infrastructure and bounded buffering.

Chaos/failure experiments are controlled hypothesis tests with owner, environment,
blast radius, steady-state measure, injection, abort condition, maximum duration,
recovery action, and postcheck. They are not random production sabotage.

### Guided work

1. Propagate standard trace context API -> saga -> broker -> worker -> service.
2. Define an allowlist for baggage/event correlation metadata.
3. Add per-service and dependency RED metrics plus queue age/backlog.
4. Build an end-to-end order dashboard and trace search workflow.
5. Define sampling that retains errors and slow operations within cost bounds.
6. Write experiments for dependency latency, connection loss, instance loss,
   duplicate messages, and broker interruption.
7. Simulate retry amplification and queue backlog; verify limits and load shedding.
8. Use symptom-based runbooks starting from "orders are slow/failing/stuck."
9. Verify recovery and data convergence after every experiment.
10. Record the first failing boundary and whether telemetry made it obvious.

### Failure checks

Break propagation on one boundary and see whether diagnosis detects the gap. Inject
inventory latency that triggers order retries, then observe whether controls prevent
a storm. Interrupt broker consumption and measure oldest event age. Sample heavily
and confirm selected failures remain diagnosable. Inspect baggage for sensitive data.

### Knowledge check and expected answers

1. **Why is a trace ID insufficient without propagation?** Each process creates an
   unrelated trace and the workflow cannot be reconstructed.
2. **Why can retries create emergent failure?** Independent callers amplify load on
   an already slow dependency, causing cascading saturation.
3. **What bounds a safe chaos experiment?** Explicit steady state, blast radius,
   abort condition, duration, owner, and recovery verification.

### Completion evidence

One failed order is reconstructed across every sync/async boundary; baggage contains
no sensitive/unbounded data; service and dependency dashboards identify the first
failure; controlled experiments remain inside abort bounds; retry/backlog controls
work; and recovery plus convergence are verified.
