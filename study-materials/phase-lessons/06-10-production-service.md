# Phases 6-10: Production Service

These lessons take the correct order workflow and make it safe to retry, observable,
deployable, measurable, and recoverable.

## Phase 6: Idempotency and Failure Semantics

### Goal and mental model

A client timeout does not reveal whether a write committed. Idempotency lets the
client repeat one logical request without creating another effect.

The idempotency key identifies an attempt within a documented scope, normally
authenticated customer plus operation. A request fingerprint binds the key to one
canonical payload. Same key and same fingerprint reuses the outcome; same key and a
different fingerprint is a conflict.

Canonicalization must be deterministic. Parse and validate JSON, normalize the
application command, serialize a defined field order, and hash those bytes. Do not
hash raw JSON because whitespace and object-key order can differ without changing
meaning. Exclude transport fields that do not affect the operation.

Model record states explicitly:

```text
in_progress -> completed
in_progress -> failed_retryable or removed according to policy
completed   -> replay stored response
```

Create the idempotency record, order, items, inventory changes, and completed outcome
inside one correctness boundary whenever possible. Concurrent duplicates must
serialize on the scoped key. One performs work; others wait briefly, observe the
committed outcome, or return a stable in-progress response according to the contract.

Store enough response data to reproduce the public status and body. Avoid storing
secrets or unstable transport headers. Define retention from the maximum client
retry window plus operational margin; cleanup must be bounded and observable.

### Guided work

1. Require one bounded `Idempotency-Key` on order creation.
2. Define key syntax, scope, retention, and response-replay contract in OpenAPI.
3. Canonicalize the validated order command and hash it.
4. Add a table keyed by customer and key with fingerprint, state, outcome, and expiry.
5. Acquire or inspect the record inside the placement transaction.
6. Reject mismatched fingerprints with a stable conflict.
7. Serialize concurrent matches and return one compatible stored outcome.
8. Persist completion atomically with the order transaction.
9. Add a bounded cleanup job and metrics for records, age, conflicts, and replays.
10. Document which client failures may be retried and for how long.

### Failure checks

- Send identical requests concurrently and prove one order exists.
- Reuse a key with reordered but semantically equal JSON and verify canonical match.
- Reuse it with one changed quantity and verify explicit conflict.
- Disconnect after commit but before reading the response, then retry.
- Crash at each state transition and verify no second effect or permanently hidden
  committed result.
- Expire a record and prove the documented old-retry behavior.

### Knowledge check and expected answers

1. **Why not hash raw JSON?** Equivalent requests may differ in whitespace and key
   order, producing false conflicts.
2. **Why scope a key by customer?** Different principals may legitimately choose the
   same opaque value and must not observe one another's outcome.
3. **Why store the response?** The caller needs a compatible answer when retrying an
   operation whose first response was lost.

### Completion evidence

Concurrent duplicates create one order; mismatched payloads conflict; disconnect
retry returns the stored outcome; crash tests cover each transition; record growth
has a measured retention/cleanup policy; and the client retry contract is explicit.

## Phase 7: Observability and Operational Controls

### Goal and mental model

Observability lets an operator answer why user-visible behavior occurred without
adding debug code during the incident. Logs describe events, metrics show aggregate
trends, traces connect a request path, and audit events record privileged actions.

Use structured logs with bounded fields: timestamp, level, message, request ID,
trace ID, route template, status, duration, and stable error code. Log an unexpected
error once at the boundary with context. Never record tokens, cookies, bodies,
credentials, or unrestricted personal data.

Metrics should cover traffic, errors, latency distributions, and saturation.
Database pool acquisition wait often reveals pressure before requests fail. Labels
must be bounded; IDs, raw paths, SKUs, user values, and error text belong elsewhere.

Traces connect HTTP, use case, and PostgreSQL operations. Name spans by stable
operation, propagate context, record result category, and omit parameters or
sensitive payload. Sampling must retain enough errors and slow traces for diagnosis.

Dashboards follow user symptoms toward resources: request SLI, endpoint breakdown,
dependency latency/errors, pool saturation, process resources, and deployments.
Alerts page only on actionable sustained impact or imminent exhaustion.

### Guided work

1. Add build revision/version metadata to startup logs and telemetry resources.
2. Emit JSON access logs with request and trace correlation.
3. Add request count, latency histogram, in-flight requests, and error metrics.
4. Export PostgreSQL pool size, use, wait, and named-operation duration.
5. Trace handler and database boundaries with redacted attributes.
6. Add audit events for admin product changes and sensitive operator actions.
7. Build a user-first dashboard for order placement and retrieval.
8. Define an SLI and SLO, then error-rate, latency, and saturation alerts.
9. Write a runbook from symptom to queries, mitigation, rollback, and escalation.
10. Capture one successful and one failed request across all telemetry types.

### Failure checks

Trigger invalid input, an unexpected application error, pool exhaustion, a slow
query, and database unavailability. Verify each produces distinguishable signals.
Search exported telemetry for authorization headers, token fragments, database URLs,
order payloads, and unbounded IDs. Force alert conditions and follow the runbook.

### Knowledge check and expected answers

1. **Why are raw paths dangerous metric labels?** Embedded IDs create unbounded time
   series and may expose user data.
2. **Why are audit events separate?** Security actions need explicit actor/outcome,
   stricter access, and different retention from ordinary diagnostics.
3. **What makes an alert actionable?** It maps meaningful user impact to tested
   diagnosis, mitigation, escalation, and recovery verification.

### Completion evidence

One request is reconstructable through logs and traces; dashboards expose SLIs and
resource pressure; labels are bounded; sensitive-data review passes; alerts cover
sustained errors, latency, and saturation; and the runbook successfully guides a
failure drill.

## Phase 8: Deployment Hardening

### Goal and mental model

Promote one immutable artifact and supply environment-specific configuration at
runtime. Deployment correctness includes process identity, filesystem, resources,
health, termination, secrets, schema compatibility, recovery, and provenance.

Use a multi-stage container whose final image contains only required runtime files.
Run as a non-root user, support a read-only root filesystem, and write only to
explicit ephemeral locations. Pin base images and dependencies; scan both source
dependencies and the built image.

Inject secrets through the platform or secret manager, never image layers, Git, or
startup logs. Separate runtime database privileges from migration privileges.

Set CPU/memory requests and limits from measurements. Ensure HTTP concurrency,
database pool size, telemetry queues, and shutdown deadline fit platform behavior.
Readiness gates traffic; liveness does not restart a process for dependency failure.

Schema and app versions overlap during rollout. Use expand-and-contract: add
compatible structure, deploy mixed-version code, backfill, switch usage, stop old
writes, then remove old structure in a later release.

Backups are unproven until an isolated restore verifies application invariants and
measures recovery time and recoverable point.

### Guided work

1. Add a pinned multi-stage container and strict `.dockerignore`.
2. Run the final binary as non-root with read-only-root compatibility.
3. Validate runtime configuration and secret references before listening.
4. Add resource requests/limits and align pool/concurrency budgets.
5. Configure readiness, liveness, startup, and termination timings.
6. Define a controlled migration step with separate credentials.
7. Rehearse one expand-and-contract schema change with old/new binaries.
8. Configure backup and perform an isolated restore with data checks.
9. Add dependency, image, and secret scanning to CI/release.
10. Record artifact digest, source revision, schema version, and deployment result.

### Failure checks

Run with missing secret, read-only filesystem, denied database privilege, and memory
or CPU pressure. Terminate during an active request. Deploy new schema with old app,
then old schema expectations with new app. Restore into isolation and verify counts,
constraints, order totals, and owner access.

### Knowledge check and expected answers

1. **Why use the same artifact in every environment?** Tested bytes and provenance
   stay constant while runtime configuration varies.
2. **Why separate migration credentials?** Schema modification has greater power than
   ordinary runtime queries and should have a smaller exposure window.
3. **Why is a backup status insufficient?** Only restoration proves bytes are usable
   and meet measured recovery objectives.

### Completion evidence

The container runs non-root/read-only; runtime secrets never enter the image or logs;
resource and shutdown limits align; old/new versions coexist through a schema
change; scanners run on source and artifact; and a restore drill verifies data and
records measured recovery.

## Phase 9: Performance and Resilience Based on Evidence

### Goal and mental model

Optimize the limiting user workload, not a guessed hot function. Performance claims
must state data size, traffic mix, concurrency, environment, warmup, duration, and
target latency/throughput.

Latency is a distribution. Report percentiles and errors alongside throughput.
Utilization shows resource use; saturation shows queued demand. Near a resource's
capacity, queueing can increase latency sharply before throughput improves.

A load generator must maintain the intended arrival pattern. Closed-loop tools that
wait for each slow response can hide delay, called coordinated omission. Use an
appropriate constant-arrival or corrected measurement for service objectives.

Profile the measured workload. CPU, heap, allocations, goroutines, mutex, blocking,
GC, database plan, connection wait, and downstream time answer different questions.
Do not interpret a CPU profile as evidence about database lock wait.

Overload controls include bounded concurrency, queues, rate limits, deadlines, and
load shedding. Recovery after pressure is as important as peak throughput.

Caching is justified only when repeated reads create measured latency/load and the
team can define source of truth, key, TTL, invalidation, stampede prevention,
failure behavior, and observability.

### Guided work

1. Define order/list workloads and measurable latency, throughput, and error targets.
2. Generate representative products, customers, orders, and pagination shapes.
3. Establish a reproducible environment and healthy baseline.
4. Run a load test that avoids coordinated omission.
5. Capture CPU, heap, allocation, goroutine, mutex, block, pool, and query-plan data.
6. Identify one limiting resource and state a hypothesis.
7. Make the smallest targeted change: query/index, pool, allocation, or limit.
8. Repeat the identical workload and compare confidence intervals or stable runs.
9. Add overload, dependency slowdown, and recovery experiments.
10. Add cache only if measurement and consistency design justify it.

### Failure checks

Drive beyond capacity and verify bounded rejection rather than memory growth. Slow
PostgreSQL and observe whether concurrency limits prevent pool collapse. Stop load
and measure recovery. Repeat correctness/concurrency tests during load. Invalidate
or remove any cache and ensure authoritative behavior remains correct.

### Knowledge check and expected answers

1. **Why are averages insufficient?** They can hide the slow tail experienced by a
   meaningful fraction of requests.
2. **What is coordinated omission?** A generator slows its issuance when the system
   slows, omitting waits that real arrivals would experience.
3. **When is caching justified?** After measurement identifies repeated-read cost and
   a safe freshness/invalidation/failure policy is defined.

### Completion evidence

Targets and workload assumptions are written; profiles identify the bottleneck;
before/after evidence supports each optimization; overload is bounded and recovery
measured; query and pool tuning preserve correctness; and any cache has explicit
consistency and removal behavior.

## Phase 10: Advanced PostgreSQL Operations and Recovery

### Goal and mental model

Understand PostgreSQL as a concurrent, maintained, recoverable system rather than a
query endpoint. MVCC gives transactions snapshots; isolation controls anomalies but
can add blocking or aborts. Locks, maintenance, WAL, backups, and replication all
affect service behavior.

At read committed, separate statements may observe different committed data. Repeatable
read preserves a snapshot but can still reject conflicting writes. Serializable aims
to reproduce serial outcomes by aborting some transactions; callers must retry the
entire safe operation, never only the last statement.

Deadlocks are cycles of lock waiting. PostgreSQL aborts one participant. Consistent
lock order reduces risk, while short transactions and bounded statement/lock
timeouts limit impact. Diagnose with lock and activity views rather than guessing.

Vacuum reclaims dead tuples for reuse and protects transaction ID safety. Statistics
guide plans. Bloat, checkpoints, WAL volume, long transactions, and replication lag
are operational signals, not one-time tuning values.

Online backfills must be restartable: select a bounded key range, update idempotently,
record progress, commit, throttle, observe locks/lag, and verify. Schema constraints
can be added in validation stages when supported to limit blocking.

Base backup plus continuous WAL archiving enables point-in-time recovery. Streaming
replication supports standby availability but does not replace backups: deletion or
corruption can replicate. RPO and RTO are measured by restore and promotion drills.

### Guided work

1. Build deterministic two-session tests for non-repeatable read, blocking,
   serialization failure, and deadlock.
2. Document the anomaly allowed by each isolation level used by the service.
3. Add bounded whole-operation retry only for safe serialization/deadlock failures.
4. Inspect activity, lock waits, query statistics, maintenance state, and plans.
5. Create representative data and a restartable, throttled backfill with progress.
6. Interrupt it repeatedly and prove resume plus verification.
7. Configure base backups and continuous WAL archive in a disposable environment.
8. Record a target timestamp, mutate data, and restore to that point.
9. Promote a standby, measure lag and failover, and reconnect the application.
10. Write the operator runbook with prerequisites, abort conditions, and validation.

### Failure checks

Hold a lock and verify statement/lock timeouts. Create a deadlock using deliberate
opposite lock order and inspect the chosen victim. Run the backfill during traffic
and observe locks, latency, WAL, and replica lag. Corrupt or remove a backup segment
in a disposable drill and confirm verification fails loudly. After restore, test
order totals, nonnegative inventory, ownership, and schema version.

### Knowledge check and expected answers

1. **Why retry the entire serializable transaction?** Its prior reads and decisions
   were based on a snapshot that can no longer be treated as valid.
2. **Why are replicas not backups?** Destructive or corrupt writes may replicate,
   while backups preserve historical recovery points.
3. **What makes a backfill restartable?** Bounded idempotent batches, durable progress,
   repeat-safe selection, verification, and controlled throttling.

### Completion evidence

Tests reproduce and explain anomalies without timing sleeps; retry is limited to
whole safe operations; database diagnostic evidence is captured; interrupted
backfills resume without gaps or duplicates; PITR reaches a chosen timestamp;
standby promotion is rehearsed; and measured RPO, RTO, lag, and failover match the
documented objectives.
