# Production-Ready Go Backend: Learning and Implementation Roadmap

## 1. Purpose

This repository will be both a working service and a guided path from senior frontend engineering to competent backend delivery in Go.

The immediate target is readiness to contribute backend code next week. The longer target is the ability to design, implement, test, operate, and safely change a production service.

Completing this roadmap should produce strong, demonstrable backend competence, but a repository cannot replace feedback from experienced reviewers or real production responsibility. The later phases therefore require design review, failure drills, operational evidence, and written tradeoffs in addition to working code.

This plan optimizes decisions in this order:

1. Data safety and security
2. Correctness
3. Operability
4. Simplicity
5. Performance

"Production-ready" is not a folder structure or a framework. It means the service has explicit contracts, protects data, behaves predictably during failure, can be observed and operated, and can be changed safely.

## 2. Target learning project

Build an **Order API** as a modular monolith.

The domain is intentionally small but realistic:

- customers can view products, place an order, and view only their own orders;
- administrators can create products and adjust inventory;
- placing an order reserves inventory and is safe to retry;
- order and inventory updates are committed atomically;
- every request is validated, authorized, logged, measured, and traceable.

This domain teaches the parts that simple task-list tutorials omit: object-level authorization, transactions, concurrent updates, idempotency, auditability, and operational failure.

### Initial API surface

| Method | Route | Purpose | Important concern |
| --- | --- | --- | --- |
| `GET` | `/health/live` | Process liveness | Must not depend on external systems |
| `GET` | `/health/ready` | Dependency readiness | Database reachability with a tight timeout |
| `GET` | `/v1/products` | List available products | Cursor pagination and stable ordering |
| `POST` | `/v1/products` | Create a product | Admin authorization and validation |
| `POST` | `/v1/orders` | Place an order | Transaction, inventory race, idempotency |
| `GET` | `/v1/orders/{id}` | Read an order | Object-level authorization |
| `GET` | `/v1/orders` | List the caller's orders | Tenant/user scoping and pagination |

Non-goals for the first version: microservices, Kubernetes, event sourcing, a custom authentication server, a generic repository framework, and premature caching.

## 3. Recommended architecture

Use a **modular monolith with vertical feature packages**. Keep one deployable service and one database while maintaining clear module boundaries inside the codebase.

```text
Client
  -> HTTP middleware
  -> feature handler (transport and validation)
  -> feature service/use case (business rules and transaction boundary)
  -> repository (SQL and persistence mapping)
  -> PostgreSQL

External identity provider -> authentication middleware
Service -> structured logs, metrics, and traces
```

Dependency direction should point inward:

- handlers depend on use-case interfaces;
- use cases own business rules and depend on narrow persistence interfaces;
- PostgreSQL adapters implement those interfaces;
- `main` constructs concrete dependencies and owns process lifecycle;
- domain code must not import HTTP, SQL-driver, or telemetry packages.

Do not force every feature to have the same number of layers. A read-only health endpoint can be simple. Order placement deserves a use case and an explicit transaction boundary.

### Proposed repository layout

This is a starting constraint, not a template to fill with empty files.

```text
.
|-- cmd/
|   `-- api/
|       `-- main.go              # Composition root and process lifecycle
|-- internal/
|   |-- platform/
|   |   |-- config/              # Typed configuration and validation
|   |   |-- database/            # Pool and transaction plumbing
|   |   |-- httpserver/          # Router, common middleware, errors
|   |   |-- observability/       # Logs, metrics, traces
|   |   `-- auth/                # Identity verification and principal
|   |-- product/                 # Product handler, use cases, repository
|   `-- order/                   # Order handler, domain rules, repository
|-- migrations/                  # Forward schema migrations
|-- api/
|   `-- openapi.yaml             # Public HTTP contract
|-- tests/                       # Cross-package integration/acceptance tests only
|-- deployments/                 # Container and local runtime definitions
|-- Makefile                     # Small, discoverable developer commands
|-- go.mod
`-- README.md
```

Keep most tests beside the Go code they exercise. Use `tests/` only when a test genuinely crosses package or process boundaries.

### Architecture vocabulary

These terms overlap; learn the principle rather than arguing over labels:

- **Layered architecture** separates transport, business logic, and persistence. It is easy to understand but becomes coupled when every feature reaches through global layers.
- **Hexagonal architecture (ports and adapters)** keeps business logic behind interfaces (ports), with HTTP and PostgreSQL as replaceable adapters. Use it where a real boundary exists.
- **Clean architecture** adds a strict inward dependency rule. Apply the dependency rule, but avoid ceremonial layers and one-method wrappers.
- **Vertical slices** group code by business capability such as `order`, not globally by technical type such as `controllers`. This keeps related changes local.
- **Modular monolith** combines independently understandable feature modules in one deployable. It provides simple operations and transactions while preserving future extraction seams.
- **Microservices** are independently deployed services. Adopt them only when team ownership, independent scaling, isolation, or release cadence provides enough value to pay the distributed-systems cost.

## 4. Technology choices

Start with a small, explicit toolset. Check dependency versions and maintenance status when implementation begins.

| Concern | Choice | Why |
| --- | --- | --- |
| Language | Current supported Go toolchain | Strong standard library, simple deployment, concurrency primitives |
| HTTP | `net/http` | Learn the protocol and Go conventions before a framework |
| Database | PostgreSQL | Common relational model, strong constraints and transactions |
| Driver/pool | `pgx` | PostgreSQL-native driver and pool |
| Queries | SQL first; add `sqlc` once core queries are understood | Keeps SQL visible while adding generated type safety |
| Migrations | Goose | Versioned, repeatable schema changes |
| Configuration | Environment variables parsed once into a typed struct | Explicit startup validation and deployment portability |
| Logging | `log/slog` JSON output | Structured standard-library logging |
| Telemetry | OpenTelemetry for traces and metrics | Vendor-neutral instrumentation |
| API contract | OpenAPI | Reviewable client/server contract and documentation |
| Local dependencies | Docker Compose | Reproducible PostgreSQL and telemetry services |
| CI | Repository-native CI running Go checks and migrations | Prevents unverified changes from merging |
| Authentication | External OIDC provider; validate access tokens | Do not build password or token issuance as a learning shortcut |

Dependency policy:

- prefer the standard library when it is clear and sufficient;
- add a dependency for meaningful correctness or operational value, not syntax preference;
- pin versions, review licenses and maintenance, and run vulnerability checks;
- wrap external packages only at boundaries where replacement or test control is valuable;
- never create a generic `utils`, `common`, or `helpers` dumping ground.

## 5. Seven-day readiness track

Budget about three focused hours per day: 45 minutes reading, 90 minutes coding, 30 minutes testing/debugging, and 15 minutes writing what failed and why. If time is limited, complete the **must finish** item before the stretch item.

### Day 1: Go as a backend language

Learn:

- modules, packages, visibility, zero values, structs, methods, slices, and maps;
- pointers and value semantics;
- interfaces are satisfied implicitly and should usually be declared by the consumer;
- multiple return values, error wrapping with `%w`, and `errors.Is`/`errors.As`;
- `defer`, resource ownership, and deterministic cleanup;
- `context.Context` carries cancellation and deadlines, not arbitrary optional parameters;
- goroutines, channels, mutexes, atomics, happens-before relationships, and the race detector;
- goroutine leaks, ownership, synchronization, and why race-free code can still have business-level concurrency bugs.

Build:

- initialize one Go module;
- implement and test a small in-memory product store;
- return explicit errors for not-found and duplicate SKU cases;
- test one concurrent access path and prove every goroutine has a termination condition;
- run formatting, unit tests, vet, and the race detector.

Must finish: explain when to return a value versus a pointer, how an interface is implemented, and how an error keeps its original identity when wrapped.

Stretch: write one fuzz test for a parser or validator.

### Day 2: HTTP contracts and request lifecycle

Learn:

- HTTP method semantics, status codes, headers, content types, and caching basics;
- the request path through DNS, TCP, TLS, reverse proxies/load balancers, and the Go server;
- connection reuse, keep-alives, file descriptors, and why leaking response bodies or connections exhausts a process;
- safe versus idempotent operations;
- JSON decoding with size limits, unknown-field rejection, and one-object enforcement;
- request validation versus business-rule validation;
- middleware ordering and request-scoped identity;
- consistent errors using `application/problem+json`-style fields;
- timeouts, cancellation, and graceful shutdown.

Build:

- start `net/http` with explicit server timeouts;
- add request IDs, panic recovery, access logs, and body-size limits;
- implement `GET /health/live` and in-memory product create/list endpoints;
- test handlers with `httptest` rather than a real port.

Must finish: trace one request from socket to response and identify who owns validation, business rules, and status-code mapping.

Stretch: implement cursor pagination without exposing a database offset.

### Day 3: Relational data and PostgreSQL

Learn:

- tables, keys, foreign keys, unique/check constraints, normalization, and nullability;
- indexes as a tradeoff between read speed, write cost, and storage;
- parameterized SQL and why string-built queries are unsafe;
- connection pools are bounded shared resources, not a single connection;
- transactions, isolation, locks, deadlocks, and atomicity;
- migrations must be ordered, reviewed, backward-aware, and tested;
- `EXPLAIN`/`EXPLAIN ANALYZE`, N+1 queries, and pagination costs.

Build:

- run PostgreSQL locally;
- add migrations for products, orders, order items, and idempotency records;
- implement product persistence with parameterized queries;
- write integration tests against a real disposable database;
- enforce duplicate SKU and invalid quantity rules in the database as well as the application.

Must finish: demonstrate that a failed order cannot leave inventory decremented without an order record.

Stretch: inspect and explain the query plan for product pagination.

### Day 4: Business logic and architecture

Learn:

- transport DTOs are not automatically domain or database models;
- business invariants belong in the use case/domain, with critical integrity also enforced by the database;
- interfaces belong at actual substitution boundaries and should remain narrow;
- dependency injection can be ordinary constructor parameters;
- transaction boundaries follow business operations, not repository methods;
- package APIs should expose as little as possible.

Build:

- reorganize products and orders into vertical feature packages;
- implement `PlaceOrder` as one use case;
- reserve inventory using a concurrency-safe SQL strategy;
- map domain errors to stable API errors at the HTTP boundary;
- add unit tests for business rules and integration tests for the transaction.

Must finish: two concurrent requests for the last unit of inventory must result in one success, not negative stock.

Stretch: document the transaction and locking choice in a short architecture decision record.

### Day 5: Identity and API security

Learn:

- authentication proves identity; authorization decides allowed actions;
- object-level authorization must scope the data query, not only hide UI controls;
- OAuth 2.0 and OpenID Connect roles at a high level: issuer, client, resource server, access token;
- OIDC discovery, JWKS rotation, issuer/audience/algorithm validation, expiry, clock skew, and key-cache failure;
- opaque tokens versus JWTs, revocation limitations, browser sessions, secure cookies, and logout;
- password storage, token issuance, and account recovery are separate security products;
- CORS is a browser policy, not API authentication;
- CSRF matters when browsers automatically attach credentials such as cookies;
- secret storage, least privilege, audit events, rate limits, and abuse controls;
- mass assignment, over-posting, injection, and sensitive-data exposure.

Build:

- introduce an authenticated principal at middleware;
- validate tokens against a local test issuer and rotate its signing key without downtime;
- authorize admin product writes and customer order reads;
- ensure order lookup includes both order ID and caller identity;
- add tests for missing, invalid, expired, wrong-role, and wrong-owner credentials;
- redact authorization headers, tokens, and personal data from logs.

Must finish: prove that one user cannot read another user's order even when they know its ID.

Stretch: write a compact threat model covering assets, trust boundaries, likely attacks, and mitigations.

### Day 6: Tests, observability, and operations

Learn:

- unit tests isolate business rules; integration tests prove boundaries; acceptance tests prove workflows;
- table-driven tests, fakes, test fixtures, and deterministic clocks/IDs;
- logs describe discrete events, metrics describe trends, and traces describe request paths;
- service-level indicators commonly include latency, traffic, errors, and saturation;
- health endpoints, graceful shutdown, bounded concurrency, and dependency timeouts;
- retry only transient operations, use exponential backoff plus jitter, and retry writes only when they are idempotent;
- an alert must lead to an actionable diagnosis or runbook.

Build:

- emit JSON logs with request/trace ID, route, status, duration, and error code;
- add request count, latency, error, and database-pool metrics;
- trace HTTP and database operations without putting secrets in attributes;
- add shutdown handling that stops new traffic and drains in-flight work;
- simulate database unavailability and a canceled request.

Must finish: use telemetry to distinguish a validation failure, a server bug, and a slow database call.

Stretch: define a latency/error service-level objective and one burn-rate alert.

### Day 7: Delivery and production rehearsal

Learn:

- containers package processes; orchestration and platforms operate them;
- build-time configuration differs from runtime configuration and secrets;
- schema changes and application deployments may not be atomic;
- expand-and-contract migrations preserve compatibility during rolling deployment;
- rollback, roll-forward, backups, restore tests, and incident response;
- capacity limits include HTTP concurrency, database connections, memory, and downstream quotas.

Build:

- create a minimal multi-stage container image running as a non-root user;
- add CI for formatting, vet, unit/integration tests, race detection, vulnerability scanning, build, and migration verification;
- run the built image with a clean database using only documented commands;
- test startup with missing configuration, graceful termination, and readiness loss;
- conduct a 30-minute incident drill: introduce a database failure, diagnose it from telemetry, recover, and record the timeline.

Must finish: another developer can clone the repository, run the service, migrate the database, run tests, and understand a failure using only repository documentation.

Stretch: deploy to a non-production environment and perform a rollback or roll-forward rehearsal.

## 6. Implementation roadmap after the first week

Each phase should be a small, reviewable change. Do not begin the next phase until its acceptance criteria pass. Every phase must be delivered through a reviewable change, receive critical feedback, and record how that feedback changed the implementation or decision.

The progression is deliberate:

| Stages | Outcome |
| --- | --- |
| Phase 0 | Discover the domain and turn ambiguity into explicit models and examples |
| Phases 1-10 | Build, secure, test, deploy, tune, and recover one production service and database |
| Phases 11-13 | Add durable background work, reliable events, and distributed-systems foundations |
| Phases 14-18 | Extract services and handle distributed contracts, consistency, failure, and observability |
| Phases 19-23 | Learn data scaling, tenant isolation, security, cloud infrastructure, orchestration, and production operations |
| Phases 24-26 | Integrate external systems, maintain a mature codebase, and complete an independently reviewed capstone |

Microservices begin only after Phase 13. Before extraction, record baseline latency, failure modes, deployment effort, and module coupling so the learner can show whether the split improved anything.

### Phase 0: Domain discovery and modeling

Learn:

- requirements begin as incomplete statements that must be converted into examples, decisions, assumptions, and open questions;
- ubiquitous language keeps product, API, code, events, and operations aligned;
- actors issue commands, domain rules protect invariants, and events describe completed facts;
- entities have identity, value objects are defined by value, and state machines make lifecycle rules explicit;
- bounded contexts separate language and ownership; they are not automatically microservices;
- domain models express business behavior and should not be shaped around HTTP handlers or database tables.

Deliver:

- a glossary for product, inventory, order, customer, reservation, cancellation, and fulfillment terms;
- concrete examples for successful, rejected, concurrent, canceled, and retried order workflows;
- order and inventory state diagrams with commands, transitions, invariants, and terminal states;
- a context map showing responsibilities and information flow between catalog, ordering, inventory, identity, and notification capabilities;
- a decision log that distinguishes confirmed requirements, assumptions, unresolved questions, and deliberately deferred scope.

Acceptance criteria:

- every initial API use case maps to examples and named invariants;
- two conflicting interpretations are resolved explicitly rather than hidden in implementation;
- domain descriptions contain no dependency on Go packages, HTTP status codes, or SQL schema details;
- a reviewer outside the implementation can explain the lifecycle and identify invalid transitions;
- later architecture decisions can cite the model rather than inventing boundaries from technical layers.

### Phase 1: Decisions and executable skeleton

Deliver:

- Go module, basic command, typed configuration, Make targets, and local PostgreSQL;
- short architecture decision records for the modular monolith, PostgreSQL, auth boundary, and SQL approach;
- CI with formatting, vet, tests, race detection, vulnerability scanning, and build.

Acceptance criteria:

- a clean clone has one documented setup path;
- invalid or missing required configuration fails startup with a useful error;
- SIGTERM produces a bounded graceful shutdown;
- all CI checks run locally.

### Phase 2: HTTP platform

Deliver:

- router, response encoding, problem-details error shape, request IDs, recovery, body limits, and access logging;
- liveness and readiness endpoints;
- OpenAPI contract for implemented routes.

Acceptance criteria:

- malformed, oversized, unknown-field, and wrong-content-type bodies have deterministic 4xx responses;
- panics return a generic 500 while retaining a diagnostic log correlation ID;
- server timeouts and maximum header/body sizes are explicit.

### Phase 3: Persistence foundation

Deliver:

- PostgreSQL pool with bounded settings and health reporting;
- migration workflow and product schema;
- product repository and integration-test harness.

Acceptance criteria:

- constraints protect invariants when the application is bypassed;
- every query accepts a context and is parameterized;
- rows, transactions, and cancellations release resources;
- migrations succeed from an empty database in CI.

### Phase 4: Product slice

Deliver:

- create and list product use cases, handlers, repositories, and tests;
- stable cursor pagination and API documentation;
- admin authorization for writes.

Acceptance criteria:

- duplicate SKUs return a stable conflict error;
- pagination cannot skip or duplicate records under the documented ordering assumptions;
- unauthorized and forbidden cases are distinct and tested.

### Phase 5: Order transaction

Deliver:

- order, order-item, and inventory schema;
- atomic order placement with a documented concurrency strategy;
- order retrieval scoped to the caller.

Acceptance criteria:

- partial orders and negative inventory are impossible;
- concurrent inventory tests are deterministic and pass under the race detector;
- wrong-owner reads do not leak whether an order exists;
- database errors do not leak schema or SQL details to clients.

### Phase 6: Idempotency and failure semantics

Deliver:

- required `Idempotency-Key` for order creation;
- request fingerprint, stored outcome, key expiry policy, and conflict behavior;
- documented retry contract for clients.

Acceptance criteria:

- identical concurrent requests create one order and receive compatible responses;
- reusing a key for a different payload fails explicitly;
- a client disconnect or uncertain network response can be retried safely;
- storage growth is bounded by a cleanup policy.

### Phase 7: Observability and operational controls

Deliver:

- structured logs, standard service metrics, distributed traces, dashboards, alerts, and a runbook;
- build/version metadata and audit events for privileged actions.

Acceptance criteria:

- one request can be followed across logs and traces;
- alerts cover sustained error rate, latency, and resource saturation;
- telemetry has bounded cardinality and contains no credentials or sensitive payloads;
- the runbook explains symptoms, queries, mitigation, rollback, and escalation.

### Phase 8: Deployment hardening

Deliver:

- minimal non-root container, read-only-compatible runtime, deployment configuration, resource limits, and secret injection;
- backup/restore and migration deployment procedures;
- dependency and image vulnerability scanning.

Acceptance criteria:

- the same artifact moves between environments with runtime configuration;
- readiness prevents traffic before dependencies are usable;
- shutdown drains within the platform termination window;
- a restore is tested, not merely configured;
- an old and new application version can coexist during schema rollout.

### Phase 9: Performance and resilience based on evidence

Learn:

- throughput, latency distributions, utilization, saturation, queueing, and coordinated omission;
- Go benchmarks and `pprof` profiles for CPU, heap, allocations, goroutines, mutexes, and blocking;
- garbage collection, allocation pressure, escape behavior, and memory limits at an operational level;
- database query plans and network/dependency time often dominate local code optimization;
- performance tests need representative data, traffic shape, concurrency, warmup, and a reproducible environment.

Deliver only after measuring:

- representative load test and resource profile;
- CPU, heap, goroutine, and contention profiles for the limiting workload;
- query-plan review and tuned indexes/pool sizes;
- bounded rate limiting and overload behavior;
- caching only if measurements justify its consistency cost.

Acceptance criteria:

- latency and throughput targets are stated with workload assumptions;
- the service degrades predictably at capacity instead of exhausting resources;
- optimization results are captured before and after;
- correctness tests continue to pass under load and failure injection.

### Phase 10: Advanced PostgreSQL operations and recovery

Learn:

- MVCC snapshots and isolation levels allow different anomalies and blocking behavior;
- row/table locks, lock ordering, deadlocks, statement timeouts, and retry policy affect correctness and availability;
- statistics, vacuum, bloat, checkpoints, WAL, and query plans influence production behavior;
- large indexes, constraints, type changes, and backfills require lock-aware, resumable migration strategies;
- backups, continuous WAL archiving, point-in-time recovery, replication, and standby promotion solve different recovery and availability problems;
- recovery point objective (RPO) and recovery time objective (RTO) are claims that must be measured by drills.

Deliver:

- reproducible tests that demonstrate non-repeatable reads, serialization failure, blocking, and deadlock behavior;
- an online, restartable backfill over representative data with progress, throttling, verification, and rollback/roll-forward guidance;
- database diagnostics using lock views, query statistics, `EXPLAIN (ANALYZE, BUFFERS)`, and maintenance statistics;
- automated base backup and WAL archive configuration for a disposable environment;
- a point-in-time restore and standby-promotion drill with an operator runbook.

Acceptance criteria:

- concurrency tests explain why the selected isolation and locking strategy preserves each invariant;
- a failed or interrupted migration resumes safely without corrupting or silently skipping data;
- the service handles serialization/deadlock retries only where the whole operation is safe to retry;
- restoration reaches a chosen timestamp and verifies application-level invariants afterward;
- measured RPO, RTO, replication lag, and failover behavior match the documented recovery objectives.

### Phase 11: Durable background jobs and scheduling

Learn:

- an in-process goroutine is not a durable job system;
- at-least-once execution, visibility leases, acknowledgements, retry schedules, and dead-letter handling;
- idempotent handlers, deduplication, poison jobs, backpressure, and graceful worker shutdown;
- periodic scheduling, leader election, clock behavior, and why multiple replicas may run the same task;
- queue age and backlog are often more useful than queue length alone.

Deliver:

- a worker command that processes order-confirmation jobs from a durable PostgreSQL-backed queue;
- bounded workers with leases, retry classification, exponential backoff with jitter, and a dead-letter state;
- a scheduler for idempotency-record cleanup that is safe when multiple instances run;
- metrics and operator commands for backlog, attempts, failures, replay, and quarantine.

Acceptance criteria:

- killing a worker during processing loses no job and may cause only a safe duplicate attempt;
- poison jobs stop retrying after a documented bound without blocking healthy work;
- two scheduler replicas do not perform an unsafe duplicate action;
- shutdown stops intake, finishes or releases leased work, and completes within its deadline.

### Phase 12: Reliable event publication and consumption

Learn:

- a database commit and broker publish cannot normally be one local atomic transaction;
- the transactional outbox makes state change and publication intent atomic;
- consumers should assume duplicate delivery and persist an inbox/deduplication record with their state change;
- delivery guarantees, ordering scope, partition keys, offsets, consumer groups, replay, schema evolution, and retention;
- "exactly once" is always scoped to particular components and side effects.

Deliver:

- an `OrderPlaced` event written to an outbox in the order transaction;
- an outbox relay that publishes to a local broker and marks publication safely;
- a notification consumer with an inbox/deduplication strategy and retry/dead-letter behavior;
- an AsyncAPI or equivalent event contract containing ownership, schema version, ordering key, and compatibility rules.

Acceptance criteria:

- crashes before and after publish cannot lose an event or create an unsafe duplicate side effect;
- replaying an event range produces the same final state;
- a consumer can deploy before or after a backward-compatible producer change;
- lag, redelivery, dead-letter count, and oldest-event age are observable.

### Phase 13: Distributed-systems mechanics lab

Learn:

- wall clocks can jump while monotonic time measures elapsed duration; distributed nodes do not share a perfectly ordered clock;
- network partitions, delay, loss, duplication, and asymmetric reachability create partial failure;
- replication, quorum reads/writes, consistency, and availability have workload-specific tradeoffs;
- leases without fencing can let an expired owner corrupt state after a pause;
- leader election and consensus coordinate replicated state, but application teams should normally use proven implementations;
- consistent hashing changes redistribution behavior, not data correctness by itself.

Deliver:

- a deterministic simulation for delayed, duplicated, dropped, and reordered messages between nodes;
- a lease exercise that first demonstrates a stale-owner bug and then prevents it with fencing tokens;
- quorum experiments that show stale reads, loss of availability, and recovery during partitions;
- a consistent-hashing experiment measuring key movement as nodes join and leave;
- a written walkthrough of Raft leader election, log replication, quorum commitment, and safety without using a custom implementation in production code.

Acceptance criteria:

- tests reproduce split-brain, stale-leader, clock-skew, and quorum-loss scenarios without timing-based sleeps;
- the learner predicts which operations remain available and which may be stale before running each experiment;
- fencing prevents an old lease holder from committing after a new owner takes control;
- recovery after healing a partition has an explicit reconciliation rule;
- the production architecture delegates consensus to a maintained database, broker, orchestrator, or coordination system.

### Phase 14: Service-boundary discovery and first extraction

Learn:

- service boundaries should follow business capabilities and ownership, not tables or technical layers;
- a distributed monolith has service deployment costs without independent boundaries;
- each service owns its data and contract; cross-service schema reads destroy autonomy;
- extraction costs include latency, partial failure, compatibility, deployment, telemetry, and on-call load;
- branch by abstraction and incremental traffic migration are safer than a rewrite.

Deliver:

- a context map showing catalog, ordering, inventory, identity, and notification responsibilities;
- coupling and change-history evidence for one extraction decision;
- an independently deployable notification service as the low-risk first extraction;
- separate configuration, runtime identity, datastore/schema ownership, CI, dashboards, and runbook;
- an architecture decision record comparing extraction with keeping the module in-process.

Acceptance criteria:

- the notification service can deploy, fail, and recover without preventing order placement;
- no service reads or writes another service's tables;
- the monolith remains functional while traffic is migrated incrementally;
- measured benefits and added operational costs are recorded honestly.

### Phase 15: Synchronous service communication and the system edge

Learn:

- choose HTTP/JSON or gRPC from compatibility, tooling, latency, and client requirements;
- API gateways handle edge concerns but must not become a shared business-logic layer;
- service discovery and load balancing map a logical dependency to changing instances;
- callers need a total deadline budget; each downstream call consumes part of it;
- retries require idempotency, bounded attempts, backoff, jitter, and one intentional retry layer;
- circuit breakers, concurrency limits, bulkheads, load shedding, and fallbacks have specific tradeoffs.

Deliver:

- an inventory service with its own API and datastore ownership;
- an order-to-inventory client with propagated cancellation, explicit deadlines, bounded connections, and stable error mapping;
- local service discovery/load balancing through the chosen runtime;
- an edge route through a gateway or reverse proxy with authentication and rate limits;
- dependency metrics covering attempts, latency, status, saturation, and circuit state.

Acceptance criteria:

- no outbound call can wait forever or exceed the request's remaining deadline;
- retry multiplication is prevented and unsafe operations are never retried blindly;
- an unavailable inventory service causes bounded, explainable failure rather than resource exhaustion;
- overload tests demonstrate fair limits and recovery after the dependency becomes healthy.

### Phase 16: Distributed workflows and eventual consistency

Learn:

- a local ACID transaction cannot atomically update independently owned service databases;
- eventual consistency means intermediate states are expected and must be modeled explicitly;
- sagas coordinate local transactions through orchestration or choreography;
- compensating actions are business operations, not automatic database rollbacks;
- workflows need durable state, timeouts, correlation, idempotency, and manual-repair paths.

Deliver:

- an order state machine such as `pending -> inventory_reserved -> confirmed` with failure/cancel states;
- a durable order saga that reserves inventory and confirms or compensates it;
- outbox/inbox handling at every event boundary;
- an operator view and commands for stuck, timed-out, and manually repaired workflows;
- invariants describing which temporary inconsistencies are acceptable and for how long.

Acceptance criteria:

- failures at every transition converge to a documented valid state;
- duplicate, delayed, and out-of-order messages do not violate inventory or payment-like invariants;
- compensation is idempotent and distinguishable from successful completion;
- a stuck workflow is detectable, diagnosable, and safely repairable.

### Phase 17: Contract evolution and independent delivery

Learn:

- provider and consumer compatibility determines whether services can deploy independently;
- OpenAPI, Protocol Buffers, and AsyncAPI describe different contract styles;
- additive changes can still break consumers through stricter validation or changed semantics;
- consumer-driven contract tests complement, but do not replace, integration tests;
- deprecation needs ownership, usage evidence, communication, and a removal date.

Deliver:

- versioned HTTP/RPC and event contracts with compatibility policy;
- provider verification against real consumer expectations;
- schema compatibility checks in CI and a registry or repository for published contracts;
- a backward-compatible change deployed in both consumer-first and provider-first order;
- a deprecation exercise that detects remaining consumers before removal.

Acceptance criteria:

- either side can deploy independently within the documented compatibility window;
- CI blocks a demonstrated breaking contract or event-schema change;
- contract tests cover semantics used by consumers, not only schema syntax;
- deprecated behavior is removed only after telemetry shows no remaining use.

### Phase 18: Distributed observability and resilience validation

Learn:

- trace context must cross HTTP, RPC, broker, and background-job boundaries;
- service-level metrics need dependency dimensions without unbounded cardinality;
- fan-out, retry storms, cascading failure, queue buildup, and coordinated recovery create emergent behavior;
- sampling must retain enough errors and slow traces to support diagnosis;
- chaos experiments require a hypothesis, bounded blast radius, abort condition, and recovery check.

Deliver:

- end-to-end traces from API request through saga, broker, worker, and downstream service;
- per-service and dependency dashboards with service-level indicators;
- controlled fault injection for latency, connection loss, duplicate messages, unavailable instances, and broker interruption;
- a retry-storm and queue-backlog experiment with protective controls;
- runbooks that begin from user-visible symptoms rather than internal components.

Acceptance criteria:

- one failed order can be reconstructed across all service and asynchronous boundaries;
- trace or baggage propagation carries no credentials or personal data;
- each experiment stays inside its declared blast radius and proves recovery;
- telemetry identifies the first failing dependency instead of only reporting downstream symptoms.

### Phase 19: Data scaling, caching, and multi-tenancy

Learn:

- caches trade freshness and invalidation complexity for latency or load reduction;
- read replicas introduce staleness and read-after-write choices;
- partitioning, archival, and retention address different data-growth problems;
- optimistic concurrency, advisory locks, and distributed locks solve different coordination problems;
- multi-tenancy requires an explicit isolation model across data, compute, identity, quotas, telemetry, and operations;
- search indexes and analytical stores are derived views, not silent sources of truth.

Deliver:

- a cache-aside product read path with TTL, invalidation policy, stampede protection, and bypass;
- a read-replica experiment with documented consistency behavior;
- a measured partitioning or archival exercise using representative data volume;
- tenant identity propagated and enforced in APIs, jobs, events, queries, and telemetry;
- a comparison of shared-table, schema-per-tenant, and database-per-tenant approaches.

Acceptance criteria:

- cache loss or staleness does not corrupt authoritative state;
- users cannot access another tenant through any synchronous or asynchronous path;
- noisy-neighbor tests demonstrate enforceable per-tenant quotas or isolation;
- scaling changes are justified by measurements and include an operational rollback.

### Phase 20: Security, privacy, and software supply chain

Learn:

- every new service, queue, datastore, and operator endpoint adds a trust boundary;
- user identity propagation differs from workload identity between services;
- transport encryption, mutual authentication, least privilege, and secret rotation reduce lateral movement;
- privacy requires data inventory, purpose, minimization, retention, deletion, export, and auditability;
- dependency provenance, reproducible builds, SBOMs, artifact signing, and vulnerability response protect delivery.

Deliver:

- updated threat models and data-flow diagrams for the distributed system;
- workload identities and least-privilege authorization for service and broker access;
- automated secret/certificate rotation without service interruption;
- user-data export and deletion workflows that include caches, events, backups policy, and derived stores;
- SBOM generation, artifact signing/verification, dependency update automation, and a vulnerability-response drill.

Acceptance criteria:

- compromised credentials have a bounded, documented blast radius;
- rotation succeeds without accepting old credentials beyond the overlap policy;
- sensitive-data inventory maps every field to purpose, owner, retention, and deletion behavior;
- a vulnerable dependency can be identified, patched, rebuilt, verified, and deployed through the normal pipeline.

### Phase 21: Cloud infrastructure and IAM

Learn:

- cloud accounts/projects, regions, and availability zones define administrative and failure boundaries;
- virtual networks, subnets, route tables, firewalls/security groups, NAT, private endpoints, and controlled egress shape connectivity;
- DNS, certificate management, TLS termination, load balancers, and health checks form the service edge;
- human identity, workload identity, roles, policies, short-lived credentials, and encryption keys require different controls;
- managed databases, queues, object storage, and secret stores transfer some operations but not application responsibility;
- infrastructure as code needs protected state, review, drift detection, policy checks, budgets, audit logs, and safe teardown.

Deliver:

- provision one non-production cloud environment through infrastructure as code;
- deploy the service behind managed DNS, TLS, and a load balancer with a non-public database;
- use workload identity and least-privilege roles instead of static application credentials;
- restrict ingress and egress, encrypt data with managed keys, and enable access/audit logging;
- configure budget alerts, resource ownership tags/labels, drift detection, and a documented teardown path.

Acceptance criteria:

- the environment can be recreated from version-controlled code without console-only steps;
- the database and administrative endpoints are unreachable from the public internet;
- each workload can access only its required resources and credential rotation causes no outage;
- network and IAM denial failures are distinguishable from application failures through logs and diagnostics;
- teardown removes the intended environment without deleting shared or production resources, and expected monthly cost is documented.

### Phase 22: Orchestration and platform operations

Learn:

- an orchestrator schedules and reconciles workloads; application correctness still owns startup, readiness, and shutdown;
- resource requests, limits, autoscaling, disruption budgets, and rollout policy interact;
- configuration, secrets, service discovery, ingress, network policy, and persistent storage are platform contracts;
- infrastructure as code needs review, state management, drift detection, and rollback;
- platform abstractions should reduce cognitive load without hiding critical failure behavior.

Deliver:

- deploy the system to a local or non-production Kubernetes environment;
- manifests or a small template for workloads, services, ingress, configuration, secrets references, and network policies;
- resource requests/limits, health probes, autoscaling signals, disruption budgets, and controlled rollouts;
- infrastructure-as-code validation and an environment bootstrap/teardown procedure;
- a platform runbook for failed rollout, unschedulable workload, bad configuration, and capacity exhaustion.

Acceptance criteria:

- rolling deployment preserves availability and contract compatibility;
- failed readiness, node loss, and termination produce the expected recovery behavior;
- autoscaling responds to a relevant saturation signal without overwhelming the database or broker;
- application teams can diagnose the platform boundary without cluster-admin access.

### Phase 23: Reliability engineering, incidents, and cost

Learn:

- user-centered service-level indicators and objectives define acceptable reliability;
- error budgets connect reliability to delivery decisions;
- capacity planning combines demand, saturation, headroom, and dependency limits;
- incidents require clear roles, communication, mitigation, evidence, and blameless follow-up;
- architecture cost includes infrastructure, engineering time, operational load, and opportunity cost.

Deliver:

- SLIs/SLOs and error-budget policy for order placement and retrieval;
- capacity model and load test tied to an expected traffic shape;
- on-call dashboards, paging alerts, escalation paths, and symptom-based runbooks;
- a game day covering partial region/dependency failure, backlog growth, and recovery;
- a blameless post-incident review with corrective actions and cost/performance analysis.

Acceptance criteria:

- alerts page only on actionable user impact or imminent exhaustion;
- the team can estimate safe capacity and identify the first scaling bottleneck;
- the game day measures detection, diagnosis, mitigation, and recovery time;
- corrective actions have owners, priorities, verification, and completion evidence.

### Phase 24: Common production integration boundaries

Learn:

- third-party APIs have quotas, rate limits, unstable latency, versioning, and ambiguous failures;
- webhooks need signature verification, replay protection, idempotency, retries, and observability;
- object storage uses streaming, checksums, metadata, authorization, lifecycle, and malware/content controls;
- email and notification delivery is asynchronous and exposes bounce, complaint, and preference workflows;
- search and analytics are eventually consistent derived systems.

Deliver:

- one outbound third-party integration behind a narrow adapter with a fake and sandbox test;
- inbound and outbound signed webhooks with replay protection and delivery history;
- a direct-to-object-storage upload flow with size/type limits and asynchronous processing;
- notification preferences and provider callback handling;
- a search/read model rebuilt entirely from authoritative data or events.

Acceptance criteria:

- provider timeout, throttling, malformed response, and outage paths are deterministic and observable;
- webhook duplicates, reordering, forgery, and endpoint downtime are tested;
- large files stream without unbounded memory and unauthorized objects cannot be accessed;
- every derived view has a documented rebuild and reconciliation process.

### Phase 25: Maintenance and legacy-system evolution

Learn:

- existing behavior must be discovered from code, tests, telemetry, data, documentation, and users before it is changed;
- characterization tests protect behavior that is important but poorly specified;
- deprecations, feature flags, canaries, shadow traffic, and compatibility windows make change reversible;
- language, dependency, database, and infrastructure upgrades each have distinct compatibility and rollback risks;
- data repair tools must be scoped, idempotent, observable, reviewable, and dry-run capable;
- obsolete flags, compatibility branches, dependencies, and abstractions must be removed after migration.

Deliver:

- select a mature project area, document its actual behavior and risks, and add characterization tests before changing it;
- refactor one coupled or oversized area incrementally while preserving public behavior;
- upgrade the Go toolchain plus at least one important dependency or infrastructure component using release notes and compatibility tests;
- perform a feature-flagged or canary rollout, observe it, roll it back once, then complete it and remove the flag;
- create and rehearse a bounded data-repair command with dry-run, resume, audit, and verification behavior;
- diagnose and correct a seeded regression using repository and production-like evidence rather than prior knowledge.

Acceptance criteria:

- contract, data, performance, and operational compatibility are verified before and after each change;
- rollback or roll-forward procedures are executed rather than merely described;
- the data repair cannot cross its declared scope and produces a durable audit record;
- temporary compatibility code and completed feature flags are removed with evidence that no consumer remains;
- another engineer can understand the legacy behavior, the change rationale, and the remaining risk from the handoff.

### Phase 26: Capstone delivery and engineering leadership

The capstone should be reviewed as if it were a production change owned by a team.

Deliver:

- identify a user problem and write a design document covering alternatives, data model, APIs/events, security, failure modes, rollout, observability, capacity, and cost;
- obtain critical review, record decisions, and revise the design before implementation;
- plan and deliver the feature in small compatible changes with tests and documentation;
- run a migration, staged rollout, dashboard review, failure drill, and rollback or roll-forward rehearsal;
- review another change for correctness, security, operability, and simplicity;
- write a concise operational handoff and teach one concept from the system to another engineer.

Acceptance criteria:

- reviewers can trace every requirement to implementation, tests, telemetry, and operational guidance;
- the feature survives concurrency, dependency failure, duplicate delivery, restart, and mixed-version deployment tests relevant to its design;
- production-like evidence supports the final reliability, performance, and cost claims;
- review findings are addressed or explicitly accepted with owners and rationale;
- the learner can explain which complexity should be removed if requirements become simpler.

## 7. Concepts a backend contributor must understand

### Domain discovery and modeling

Backend design starts by discovering language, behavior, invariants, ownership, and failure cases. Convert vague requirements into concrete examples and state transitions before selecting endpoints or tables. Keep facts, assumptions, and unresolved questions separate so implementation does not silently turn guesses into permanent contracts.

Domain-driven terms are tools, not required ceremony. Use entities, value objects, aggregates, domain services, bounded contexts, and events only when they make rules and ownership clearer. A bounded context can remain a package inside a modular monolith; it does not require a network boundary.

### Processes, networking, and runtime resources

A deployed Go service is an operating-system process. Understand its arguments, environment, identity and permissions, working directory, signals, exit codes, filesystem access, memory, CPU, threads, file descriptors, and sockets. Containers isolate and constrain those resources; they do not remove the need to manage them.

Be able to trace a request through DNS resolution, TCP connection establishment, TLS, a proxy or load balancer, the HTTP server, application code, and downstream connections. Know where timeouts apply, which connections are pooled or reused, how trust is established, and which layer generated an error. Use packet or connection-level tools only after checking application telemetry and with appropriate authorization.

### Request, time, and cancellation

A backend request consumes shared resources while the caller is waiting. Propagate the request context through use cases and database/external calls. Apply timeouts at boundaries. Never store a context in a struct or start background work from a request without giving that work an independent lifecycle.

Every network result can be ambiguous: the client may time out after the server committed. Design write operations around idempotency rather than assuming a timeout means "nothing happened."

### Data ownership and integrity

The database is durable shared state, so correctness must survive multiple service instances and concurrent requests. In-memory mutexes do not protect a distributed deployment. Use database constraints, atomic statements, locks, and transactions intentionally.

Put rules in two places when they serve different purposes:

- the domain/use case explains business intent and produces useful errors;
- database constraints provide the final integrity boundary.

Choose data types by meaning rather than convenience:

- represent money with an exact decimal type or integer minor units, never binary floating point;
- store the price charged on each order item so later product-price changes do not rewrite history;
- use UTC instants for events and retain a named time zone separately when civil time matters;
- distinguish missing, empty, and zero values explicitly at API and database boundaries;
- treat identifiers as opaque values, not as proof of ownership or authorization.

### Transactions

A transaction represents one atomic business operation. Keep it short, do not make slow network calls while holding it, and make the use case own its boundary. Know which rows are read and written, in what order, and how concurrent transactions interact.

Always handle commit failure. A deferred rollback is safe cleanup, not a substitute for checking commit.

Database ownership also includes operations and recovery. Know how isolation, locks, deadlocks, statistics, vacuum, WAL, replication, backups, and schema changes affect the running service. A backup is not trusted until a restore has verified application-level invariants, and a standby is not a recovery plan until promotion and client reconnection have been exercised.

### API design

Treat the API as a public contract:

- use nouns for resources and HTTP methods for actions where practical;
- distinguish syntax validation, business conflicts, authentication, authorization, and internal failure;
- make errors stable and machine-readable with a code, message, and correlation ID;
- use bounded pagination and filtering;
- document optionality, nullability, formats, limits, ordering, and retry behavior;
- avoid breaking fields; use additive changes and explicit deprecation;
- never expose database models directly as request or response types.

### Authentication and authorization

Authorization belongs at both the use-case boundary and the query/data scope. A valid token is not permission to access every object. Default to deny, use least privilege, and test horizontal access between two ordinary users, not just admin versus anonymous.

Token validation must pin trusted issuers, audiences, algorithms, and key sources; handle expiry and bounded clock skew; and survive signing-key rotation. JWTs reduce introspection calls but make immediate revocation difficult. Opaque sessions or tokens move state to an authorization service. Browser cookies add `Secure`, `HttpOnly`, `SameSite`, session fixation, logout, and CSRF concerns that bearer-token APIs may not share.

### Concurrency and bounded resources

Goroutines are cheap, not free. Every goroutine needs an owner, termination condition, and bounded input. Protect shared memory with clear ownership or synchronization. Bound worker pools, queues, request sizes, connection pools, retries, and caches.

Understand which mutex, channel, atomic, and goroutine operations establish happens-before relationships. Prefer simple ownership and synchronization over clever lock-free code. Use `go test -race` regularly, but remember it detects races only on executed paths and cannot prove freedom from goroutine leaks, deadlocks, or business-level concurrency errors.

### Errors and failure handling

Classify errors by what callers can do:

- invalid input: correct the request;
- unauthenticated/forbidden: obtain identity or permission;
- not found/conflict: change target or state;
- transient dependency failure: retry only when safe, preferably after server guidance;
- internal bug: return a generic response and record diagnostic context.

Wrap errors with operation context while preserving identity. Log an error once at the boundary that has request context; repeated logging at every layer creates noise.

### Observability

Use:

- **logs** for discrete diagnostic events;
- **metrics** for aggregate rate, latency, error, and saturation trends;
- **traces** for causal paths and time spent across boundaries;
- **profiles** for CPU, memory, lock, and goroutine analysis when needed.

Avoid high-cardinality metric labels such as user ID, request ID, or raw URL. Keep those dimensions in logs or traces.

### Deployments and schema evolution

Deployments are normal failure events. The service must handle termination, unavailable dependencies, mixed versions, and partial rollout. For breaking schema changes, use expand-and-contract:

1. add a backward-compatible schema;
2. deploy code that can use it while remaining compatible;
3. backfill and verify data;
4. switch reads/writes;
5. remove the old shape only after rollback is no longer required.

### Cloud infrastructure and identity

Cloud systems combine network, identity, compute, storage, and managed-service boundaries. Keep application data services private, expose only intentional edges, use short-lived workload identities, restrict ingress and egress, encrypt data with owned access policy, and retain audit evidence. Infrastructure as code should reproduce environments, reveal reviewable changes, detect drift, protect state, and make teardown as deliberate as creation.

Kubernetes is one workload platform within this environment, not a replacement for cloud networking, IAM, DNS, certificates, databases, backups, or cost control.

### Background work and messaging

Once work outlives an HTTP request, it needs a durable lifecycle. Define who creates it, where intent is stored, how workers claim it, how long a claim lasts, which failures retry, how duplicates are made harmless, when work is quarantined, and how an operator replays or cancels it.

Message delivery and business effect are different guarantees. A broker can redeliver a committed message after a consumer crashes. Consumers should therefore expect at-least-once delivery, make effects idempotent, and record progress atomically with local state when possible. Ordering is normally guaranteed only within a stated key or partition, not across an entire system.

### Microservices and distributed systems

A service is an ownership and deployment boundary, not just another process. It should own a business capability, its data, its public contracts, its delivery pipeline, its telemetry, and its operational responsibility. A shared database or lockstep release schedule is evidence that the boundary may not be real.

Distribution introduces partial failure: one component can be healthy while its network path, dependency, or caller is not. Every remote call needs a deadline, cancellation, bounded concurrency, an error contract, and an explicit retry decision. Adding retries at multiple layers multiplies traffic during the exact failure period when capacity is scarce.

Cross-service workflows cannot rely on one database transaction. Model durable intermediate states and convergence. Use outbox/inbox patterns to bridge local commits and messages, and sagas when a business workflow requires multiple local transactions and compensations. Design for duplicates, delay, reordering, replay, and manual repair.

Prefer asynchronous communication when the caller does not require an immediate result and temporary decoupling is valuable. Prefer synchronous communication when the caller needs a current answer. Neither style is inherently more scalable or correct; both require contracts, limits, observability, and ownership.

### Data growth and tenancy

Scale the current bottleneck rather than applying every technique. Indexes, query changes, archival, partitions, replicas, caches, and sharding solve different problems. Each introduces consistency, routing, migration, or operational costs that must be stated and tested.

Treat tenant identity as an invariant carried through requests, jobs, events, queries, caches, logs, metrics, and operator tools. Choose the isolation model from security, compliance, scale, restore, and cost requirements. A tenant filter added by convention is weaker than a boundary that fails closed.

### Professional backend engineering

A valued engineer reduces uncertainty for the team. They clarify requirements, expose invariants, write reviewable designs, choose the smallest sufficient architecture, make failure visible, test risky assumptions, and leave systems easier to operate. They distinguish facts from hypotheses and support performance or reliability claims with measurements.

Backend ownership includes reviewing code, migrations, dashboards, alerts, rollouts, incidents, and documentation. It also includes saying when a microservice, cache, retry, abstraction, or new dependency adds more risk than value.

Most valuable work happens in systems that already have users, data, undocumented behavior, and constraints. Trace actual behavior before changing it, pin risk with characterization tests, prefer reversible migrations, remove temporary compatibility mechanisms, and communicate residual risk. Treat critical feedback as an input to better decisions rather than a gate to work around.

### Testing strategy

| Test type | Best use | Avoid |
| --- | --- | --- |
| Unit | Business rules, validation, error mapping | Mocking every internal function |
| Handler | HTTP decoding/encoding and status contract | Re-testing database behavior |
| Repository integration | Real SQL, constraints, transactions | SQLite as a PostgreSQL substitute |
| Component | One service with real local dependencies and controlled externals | A full fleet for every test |
| Contract | Consumer/provider compatibility and event schemas | Assuming schema compatibility proves semantic compatibility |
| Acceptance | Critical user workflows through the service | Exhaustive permutations |
| Concurrency | Inventory/idempotency races | Timing-based sleeps |
| Fuzz | Parsers, decoders, invariant-heavy pure functions | Unbounded CI runs |
| Load | Capacity and latency under a stated workload | Treating one laptop result as production truth |
| Failure/chaos | Recovery hypotheses and protective controls | Unbounded experiments without abort conditions |
| Migration | Empty, populated, mixed-version, and rollback/roll-forward behavior | Testing only the final schema |

Tests should control time, IDs, and randomness where assertions depend on them. A test must fail for the intended reason before a bug fix when practical.

## 8. Frontend-to-backend mental model

| Familiar frontend idea | Backend analogue | Critical difference |
| --- | --- | --- |
| Component boundary | Package/module boundary | Backend boundaries also protect data and deployment behavior |
| Event handler | HTTP handler | Requests are concurrent, untrusted, and may be retried |
| Client state/store | Database | Durable shared state requires transactions and migrations |
| TypeScript interface | Go interface | Go interfaces are implicit and best kept small near consumers |
| AbortController | `context.Context` | Cancellation should propagate through every blocking boundary |
| Error boundary | Recovery middleware | Recovery prevents process-wide impact but must not hide bugs |
| Frontend router | HTTP router | HTTP method semantics, auth, limits, and status contracts matter |
| Loading/retry state | Timeout/retry/idempotency policy | A timed-out write may already have committed |
| E2E test | Acceptance test | Real database state and cleanup are central |
| Build artifact | Service binary/container | The process must remain healthy, observable, and gracefully stoppable |

Common transition traps:

- trusting request fields because the UI already validated them;
- treating authorization as route visibility rather than a data-access invariant;
- modeling the database like a client-side object store;
- returning immediately after starting unowned background work;
- retrying a write without idempotency;
- using in-memory state that silently breaks with multiple instances;
- introducing abstractions before a second real implementation or use case exists;
- optimizing individual functions before measuring database and network behavior;
- logging entire requests, tokens, or personal data during debugging.

## 9. Standard engineering workflow

For every feature:

1. State the use case and acceptance criteria in user/business language.
2. Separate confirmed requirements, assumptions, open questions, and deferred scope.
3. Identify assets, actors, trust boundaries, failure modes, and authorization rules.
4. Define or update the HTTP/RPC/event contract and examples.
5. Model database constraints and migration compatibility.
6. Write the failing business, characterization, or regression test.
7. Implement the smallest vertical path: handler -> use case -> repository.
8. Test success, validation failure, unauthorized/forbidden access, not found/conflict, cancellation, and dependency failure as applicable.
9. Run formatting, static checks, unit tests, integration tests, race detection, and vulnerability checks.
10. Verify logs, metrics, and traces for the new path.
11. Request critical review of correctness, security, operability, simplicity, and rollback; record substantive changes from feedback.
12. Update the owning documentation and operational runbook.
13. Keep the change small enough to review and roll back, then remove completed flags or compatibility scaffolding.

Suggested verification commands once the project exists:

```sh
gofmt -w .
go vet ./...
go test ./...
go test -race ./...
go test -cover ./...
govulncheck ./...
go build ./...
```

The eventual Make targets should wrap these commands consistently; CI must use the same underlying checks. Coverage is a signal for untested risk, not a target that proves correctness.

## 10. Production-readiness checklist

### Contract and correctness

- [ ] Domain language, lifecycle states, ownership, assumptions, and invariants are explicit.
- [ ] API behavior, limits, error codes, pagination, and retry semantics are documented.
- [ ] Boundary inputs are validated and request bodies are bounded.
- [ ] Critical invariants exist in business logic and database constraints.
- [ ] Transactions and concurrent-update behavior are documented and tested.
- [ ] Idempotency exists for retriable writes.
- [ ] Time and identifier generation are controllable in tests.

### Security

- [ ] Authentication verifies issuer, audience, signature, expiry, and allowed algorithms.
- [ ] Signing-key rotation, browser session/cookie behavior, logout, revocation constraints, and CSRF are tested where applicable.
- [ ] Authorization covers roles and object ownership with default deny.
- [ ] SQL is parameterized and output does not expose internal details.
- [ ] Secrets and sensitive data are excluded from source, logs, metrics, and traces.
- [ ] Dependencies and container images are scanned.
- [ ] Threat model and abuse limits are reviewed.

### Reliability and operations

- [ ] Client, server, database, and downstream timeouts are explicit.
- [ ] Retries are bounded, jittered, and limited to safe/idempotent work.
- [ ] Queues, pools, goroutines, request bodies, and caches are bounded.
- [ ] Liveness, readiness, and graceful shutdown behave correctly.
- [ ] Logs, metrics, traces, dashboards, alerts, and runbooks are usable.
- [ ] Point-in-time restore and standby promotion meet measured RPO/RTO objectives.
- [ ] Deployment rollback/roll-forward is rehearsed.

### Delivery

- [ ] A clean checkout can build, test, migrate, and run from documentation.
- [ ] CI reproduces local quality checks and blocks warnings/failures.
- [ ] The artifact is minimal, non-root, immutable, and configured at runtime.
- [ ] Migrations support mixed application versions during rollout.
- [ ] Resource limits and capacity assumptions are documented.
- [ ] Dependencies have an owner and an update policy.
- [ ] Each substantial change receives critical review and records how concrete findings were resolved.

### Cloud infrastructure

- [ ] Runtime networks, subnets, routes, ingress, egress, DNS, TLS, and load balancers are defined as code.
- [ ] Data services and operator endpoints are private unless exposure is explicitly justified.
- [ ] Workloads use short-lived, least-privilege identities rather than static credentials.
- [ ] Infrastructure state, drift, audit logs, ownership, budgets, and teardown are controlled.

### Asynchronous and distributed behavior

- [ ] Jobs and consumers are idempotent, bounded, observable, and safely replayable.
- [ ] Outbox/inbox or an equivalent design prevents lost intent and unsafe duplicate effects.
- [ ] Service data ownership is explicit and no service depends on another service's schema.
- [ ] Remote calls have deadline budgets, bounded retries, load limits, and stable error contracts.
- [ ] Distributed workflows model intermediate, timeout, compensation, and manual-repair states.
- [ ] API and event contracts can evolve without lockstep deployment.
- [ ] Traces and correlation cross service, queue, and worker boundaries without sensitive baggage.

### Data, privacy, and supply chain

- [ ] Tenant isolation covers APIs, jobs, events, caches, telemetry, and operator paths.
- [ ] Cache, replica, partition, and derived-view consistency behavior is documented and tested.
- [ ] Sensitive data has purpose, owner, retention, export, deletion, and audit policies.
- [ ] Workload identities and permissions follow least privilege and support rotation.
- [ ] Builds produce verifiable artifacts and SBOMs with a tested vulnerability-response path.
- [ ] Capacity and cost assumptions are measured and reviewed.

## 11. Definition of ready for next week

You are ready to contribute safely, not to know everything, when you can do all of the following without guessing:

- navigate a Go module and explain package visibility and `internal`;
- implement a handler with strict input validation and stable error output;
- pass `context.Context` through a use case into a database query;
- write a parameterized query and explain its indexes and transaction boundary;
- distinguish authentication, role authorization, and object ownership;
- add a unit test and a real-database integration test;
- identify whether a failed write can be safely retried;
- follow one request through structured logs and a trace;
- run the repository's complete verification pipeline;
- ask for the service's API contract, migration process, deployment model, SLOs, and incident runbook before changing risky behavior.

If one item is weak, turn it into a pairing goal for the first backend task rather than hiding the uncertainty.

## 12. Definition of competent completion

Completing phases is not a checkbox exercise. Competence is demonstrated when the learner can repeatedly produce and defend evidence across these areas:

| Capability | Evidence required in this repository |
| --- | --- |
| Go engineering | Idiomatic packages, explicit errors, safe concurrency, bounded resources, profiling, and race-tested code |
| API design | Reviewable HTTP/RPC/event contracts, compatibility policy, validation, idempotency, and useful error semantics |
| Data correctness | Constraints, migrations, transaction/concurrency tests, query plans, recovery, and restore evidence |
| Security and privacy | Threat models, least privilege, object/tenant authorization tests, secret rotation, and data-lifecycle workflows |
| Distributed systems | Owned service boundaries, deadline/retry policy, outbox/inbox, durable saga, contract tests, and failure experiments |
| Operability | Structured telemetry, SLOs, dashboards, actionable alerts, runbooks, graceful lifecycle, and capacity limits |
| Cloud/platform | Reproducible networks and environments, least-privilege workload identity, controlled exposure, drift detection, and cost evidence |
| Delivery | Reproducible CI, verifiable artifacts, compatible rollouts, migration safety, feature-flag cleanup, and rollback/roll-forward rehearsal |
| Engineering judgment | ADRs with alternatives, measured tradeoffs, deletion of unjustified complexity, and clear escalation of uncertainty |
| Maintenance | Characterization of existing behavior, safe upgrades, scoped data repair, regression diagnosis, compatibility preservation, and removal of temporary code |
| Team contribution | Useful code reviews, response to critical feedback, concise design communication, operational handoffs, incident learning, and teaching another engineer |

The final standard is that another engineer can safely review, deploy, operate, debug, and change the system using the contracts and evidence the learner produced. Real production experience and experienced review remain necessary to calibrate judgment, but the repository should make the learner ready to earn that responsibility rather than merely discuss it.

## 13. Questions to ask on an unfamiliar backend team

- What invariants would cause financial, security, or data-integrity harm if violated?
- Where are transaction boundaries, and what is the concurrency strategy?
- Which service owns each piece of data?
- Which operations cross service boundaries, and where can partial failure leave intermediate state?
- What are the delivery, ordering, deduplication, replay, and dead-letter guarantees for messages?
- How are authentication and object-level authorization enforced?
- What operations are idempotent, and what may clients retry?
- What are the timeout, retry, rate-limit, and connection-pool policies?
- How do schema migrations remain compatible with rolling deployments?
- What are the SLOs, highest-volume endpoints, and current bottlenecks?
- Which dashboards, alerts, and runbooks are used during incidents?
- How are secrets, backups, restores, rollbacks, and dependency updates handled?
- What are the measured RPO/RTO objectives and when was recovery last exercised?
- Which network and IAM boundaries protect workloads and data services?
- What checks must pass before merge and deployment?

## 14. Primary references in study order

Prefer primary documentation over architecture-blog folklore.

1. [A Tour of Go](https://go.dev/tour/) - language fundamentals through executable exercises.
2. [Effective Go](https://go.dev/doc/effective_go) - core idioms; supplement it with newer module, testing, and error guidance.
3. [Organizing a Go module](https://go.dev/doc/modules/layout) - official package, `internal`, `cmd`, and server layout guidance.
4. [Go Code Review Comments](https://go.dev/wiki/CodeReviewComments) - practical conventions reviewers expect.
5. [Go blog: Error handling and Go](https://go.dev/blog/error-handling-and-go) and [`errors` package](https://pkg.go.dev/errors) - explicit error flow and wrapping.
6. [Go blog: Context](https://go.dev/blog/context) and [canceling database operations](https://go.dev/doc/database/cancel-operations) - deadlines and cancellation propagation.
7. [`net/http` package](https://pkg.go.dev/net/http), [HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110), and [Problem Details for HTTP APIs](https://www.rfc-editor.org/rfc/rfc9457) - server behavior and standard HTTP contracts.
8. [Go database guide](https://go.dev/doc/database/) and [executing transactions](https://go.dev/doc/database/execute-transactions) - pools, queries, cancellation, and transactions.
9. [Go testing package](https://pkg.go.dev/testing), [fuzzing](https://go.dev/doc/security/fuzz/), and [race detector](https://go.dev/doc/articles/race_detector) - built-in verification tools.
10. [PostgreSQL documentation: concurrency control](https://www.postgresql.org/docs/current/mvcc.html), [indexes](https://www.postgresql.org/docs/current/indexes.html), and [`EXPLAIN`](https://www.postgresql.org/docs/current/using-explain.html) - data correctness and query behavior.
11. [OWASP API Security Top 10](https://owasp.org/API-Security/), [OWASP Application Security Verification Standard](https://owasp.org/www-project-application-security-verification-standard/), and [OAuth 2.0 Security Best Current Practice](https://www.rfc-editor.org/rfc/rfc9700) - threat and verification guidance.
12. [OpenAPI Specification](https://spec.openapis.org/oas/latest.html) - HTTP contract format.
13. [OpenTelemetry Go](https://opentelemetry.io/docs/languages/go/) - traces, metrics, and instrumentation.
14. [The Twelve-Factor App](https://12factor.net/) - useful deployment principles; apply with judgment rather than as an architecture template.
15. [Google SRE book](https://sre.google/sre-book/table-of-contents/) - service levels, monitoring, incident response, and operations.
16. [gRPC guides: deadlines](https://grpc.io/docs/guides/deadlines/), [retries](https://grpc.io/docs/guides/retry/), and [health checking](https://grpc.io/docs/guides/health-checking/) - explicit remote-call behavior.
17. [Apache Kafka design: message delivery semantics](https://kafka.apache.org/43/design/design/#message-delivery-semantics) - at-most-once, at-least-once, and scoped exactly-once guarantees.
18. [AsyncAPI Specification](https://www.asyncapi.com/docs/reference/specification/latest) - asynchronous message contracts.
19. [OpenTelemetry context propagation](https://opentelemetry.io/docs/concepts/context-propagation/) - correlating work across service and messaging boundaries.
20. [Pact documentation](https://docs.pact.io/) - consumer-driven contract testing concepts.
21. [Kubernetes concepts](https://kubernetes.io/docs/concepts/) - workloads, networking, configuration, security, and resource management.
22. [PostgreSQL logical replication](https://www.postgresql.org/docs/current/logical-replication.html) and [row security policies](https://www.postgresql.org/docs/current/ddl-rowsecurity.html) - replication and database-enforced isolation tools.
23. [Amazon Builders' Library: making retries safe with idempotent APIs](https://aws.amazon.com/builders-library/making-retries-safe-with-idempotent-APIs/) and [controlling retries](https://docs.aws.amazon.com/wellarchitected/latest/framework/rel_mitigate_interaction_failure_limit_retries.html) - practical distributed-failure guidance.
24. [OpenSSF Scorecard](https://scorecard.dev/) and [SLSA specification](https://slsa.dev/spec/) - software supply-chain assessment and build provenance.
25. [The Go Memory Model](https://go.dev/ref/mem) - synchronization, happens-before relationships, and data-race semantics.
26. [PostgreSQL backup and restore](https://www.postgresql.org/docs/current/backup.html), [continuous archiving and point-in-time recovery](https://www.postgresql.org/docs/current/continuous-archiving.html), and [standby servers](https://www.postgresql.org/docs/current/warm-standby.html) - recovery and high-availability foundations.
27. [In Search of an Understandable Consensus Algorithm](https://raft.github.io/raft.pdf) - Raft leader election, log replication, and safety.
28. [AWS Well-Architected Framework](https://docs.aws.amazon.com/wellarchitected/latest/framework/welcome.html) - cloud architecture review across operations, security, reliability, performance, cost, and sustainability.

## 15. Optional specialization paths

The core roadmap builds general backend competence. Choose optional paths based on the target role rather than treating every technology as mandatory:

- **Payments and ledgers:** double-entry accounting, reconciliation, settlement, disputes, precision, immutable audit, and regulatory boundaries.
- **Event sourcing and CQRS:** event streams as source of truth, projections, snapshots, versioning, replay, and operational complexity.
- **Real-time systems:** WebSockets or streaming RPC, presence, fan-out, ordering, backpressure, reconnect, and resume semantics.
- **Data-intensive systems:** change-data capture, stream processing, search, warehouses, data quality, lineage, and batch/stream reconciliation.
- **Global systems:** multi-region routing, replication lag, conflict resolution, data residency, disaster recovery, and consistency tradeoffs.
- **Platform engineering:** reusable service templates, policy as code, developer portals, workload identity, and paved-road ownership.
- **Serverless systems:** event-driven execution, concurrency controls, cold starts, runtime limits, local testing, and cost behavior.
- **Specialized protocols:** GraphQL, advanced gRPC streaming, or domain-specific protocols when product needs justify them.

Specializations must follow the same standard as core phases: implement a real use case, define failure semantics, verify security and operability, measure claims, and document when the approach should not be used.
