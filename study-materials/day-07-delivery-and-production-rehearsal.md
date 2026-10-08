# Day 7: Delivery and Production Rehearsal

This lesson expands Day 7 of the
[backend learning roadmap](../BACKEND_LEARNING_ROADMAP.md#day-7-delivery-and-production-rehearsal).
It packages the service into one reproducible artifact and rehearses operating it
under failure.

## Learning outcomes

By the end of this lesson, you should be able to:

- build a minimal non-root container artifact;
- separate build inputs, runtime configuration, and secrets;
- design CI as a reproducible version of local verification;
- deploy schema and application changes compatibly;
- choose rollback or roll-forward based on data safety;
- explain why backups require restore tests;
- identify HTTP, database, memory, and downstream capacity limits;
- execute and document a database incident drill;
- prove that another developer can operate the project from its documentation.

## 1. One immutable artifact, many environments

Build source code once and promote the same artifact through environments. Runtime
configuration selects environment-specific endpoints, limits, and credentials.

Do not bake production configuration or secrets into the binary or image. Rebuilding
per environment weakens provenance: the artifact tested in staging is no longer the
artifact deployed to production.

Useful build metadata includes source revision, build time, Go version, and artifact
digest. Expose only non-sensitive version identity in logs or a protected diagnostic
endpoint.

## 2. Build a small non-root container

A multi-stage build compiles with the Go toolchain and copies only the runtime
artifact into the final image:

```dockerfile
FROM golang:1.26.3-bookworm AS build

WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .

RUN CGO_ENABLED=0 GOOS=linux go build \
    -trimpath \
    -ldflags="-s -w" \
    -o /out/api \
    ./cmd/api

FROM gcr.io/distroless/static-debian12:nonroot

COPY --from=build /out/api /api
USER nonroot:nonroot
EXPOSE 8080
ENTRYPOINT ["/api"]
```

Before adopting these example tags, pin versions and preferably immutable digests
verified by the project. If the application needs CA roots, time-zone data, or CGO,
select and test a runtime image that provides them. Do not copy the compiler, source
tree, Git metadata, local credentials, or `.env` files into the final image.

Use a `.dockerignore` to reduce accidental build context:

```text
.git
.env
coverage.out
tmp/
```

The process runs as a non-root user and writes only to explicitly allowed ephemeral
paths. The root filesystem can be read-only when the runtime environment supports
it.

## 3. Configuration is a startup contract

Parse environment variables once into typed configuration. Validate required values
before opening the server:

- listen address;
- database URL;
- trusted OIDC issuer and audience;
- request and shutdown limits;
- database pool maximum;
- telemetry exporter configuration;
- environment name and log level.

Missing, malformed, or contradictory configuration must fail startup with an
actionable error that does not reveal secret values.

Classify configuration:

| Kind | Example | Delivery mechanism |
| --- | --- | --- |
| Build input | Go version, source revision | Build system |
| Runtime non-secret | Port, pool limit, issuer URL | Environment/config service |
| Runtime secret | Database password, exporter credential | Secret manager/injected secret |

Do not silently fall back to development credentials in a production environment.
Defaults are appropriate only when they are safe and documented.

## 4. CI reproduces the local quality contract

CI should run repository-owned commands rather than contain a second implementation
of the workflow. A small Makefile or scripts can define:

```text
make format-check
make vet
make test
make test-integration
make test-race
make vulnerability-check
make migration-check
make build
make container-build
```

The pipeline should:

1. Check formatting without rewriting files.
2. Run static analysis and vet.
3. Run unit and handler tests.
4. Start disposable PostgreSQL and apply migrations from empty state.
5. Run integration and acceptance tests.
6. Run the race detector in a suitable job.
7. Run the pinned vulnerability scanner.
8. Build the binary and container.
9. record artifact identity and checksums.
10. publish only after all required checks pass.

Pin CI actions, tool versions, and base images. Grant the pipeline the minimum token
permissions. Do not expose secrets to untrusted pull-request code. Cache dependencies
for speed only when the cache key and trust boundary are safe.

A failure should be reproducible locally through the same command. Treat compiler,
vet, static-analysis, migration, and test warnings as failures.

## 5. Application and schema deployment are not atomic

During a rolling deployment, old and new application versions may run at the same
time against one schema. Design migrations for compatibility.

Use expand-and-contract for a renamed column:

1. **Expand:** add the new nullable column without removing the old one.
2. Deploy code that can operate while both representations exist, often writing
   both and reading with a deliberate fallback.
3. Backfill old rows in bounded batches with progress and verification.
4. Switch reads to the new column after the backfill is complete.
5. Stop writing the old column.
6. **Contract:** remove the old column in a later independently reviewed release.

Adding a required column with an immediate default or rewriting a large table can
lock or overload production. Evaluate PostgreSQL version behavior, table size, lock
duration, and rollback strategy before deployment.

Run migrations as one controlled deployment step with clear ownership. Do not let
every replica race to modify the schema at startup.

## 6. Rollback and roll-forward protect different failures

Application rollback is useful when the previous version remains compatible with
the current schema and data. It is unsafe when a migration removed data or the new
version wrote a format the old version cannot understand.

Roll-forward applies a new corrective change. It is often safer for schema and data
problems because it preserves evidence and avoids destructive reverse migrations.

Before deployment, record:

- artifact to deploy;
- schema compatibility window;
- health and SLO signals to watch;
- automatic and manual stop conditions;
- rollback compatibility;
- roll-forward owner and expected time;
- post-deployment verification queries and user workflow.

Do not claim rollback support until it has been rehearsed with real artifacts and
representative data.

## 7. Backups become trustworthy through restore drills

A backup job reporting success proves only that bytes were written somewhere. A
restore drill proves that the bytes can recreate usable data within recovery goals.

Define:

- **RPO (recovery point objective):** maximum acceptable data loss measured in time;
- **RTO (recovery time objective):** maximum acceptable time to restore service;
- backup frequency and retention;
- encryption and access control;
- restore destination and procedure;
- integrity and application-level verification after restore.

A restore drill should use an isolated database, restore the selected backup, run
schema/version checks, count and sample critical records, verify constraints, start
the application against it when safe, and record actual RPO/RTO evidence.

Never test restoration by overwriting the only production database.

## 8. Capacity is a chain of bounded resources

The maximum safe request rate is constrained by the tightest part of the path:

- load balancer and HTTP connection limits;
- active handler and goroutine count;
- request/response memory;
- CPU and garbage collection;
- database pool and PostgreSQL connection capacity;
- row-lock contention and transaction duration;
- telemetry queues;
- identity-provider/JWKS traffic;
- downstream quotas and rate limits.

If 20 instances each open 20 database connections, the potential application total
is 400. Capacity planning must include administrative, migration, monitoring, and
failover connections.

Queueing hides overload briefly and increases latency. Keep queues bounded, measure
their depth and wait time, and shed load predictably when work cannot complete within
the service objective.

Run load tests with representative data, concurrency, warmup, request mix, and
environment. Report latency distributions, errors, throughput, saturation, and the
first limiting resource. A laptop result is not a production capacity promise.

## 9. Rehearse startup and termination failures

Verify these process behaviors using the built artifact:

### Missing configuration

Remove one required variable. Expect immediate nonzero exit, an actionable field
name, no secret values, and no listening socket.

### Clean startup

Start with a freshly migrated empty database. Expect one structured startup event,
liveness success, readiness success after dependencies are available, and no manual
database edits.

### Readiness loss

Make PostgreSQL unavailable. Expect readiness to fail within its deadline while
liveness stays healthy. Restore PostgreSQL and confirm readiness recovers without a
process restart.

### Graceful termination

Begin a controlled long request, send `SIGTERM`, and verify new traffic stops while
the in-flight request finishes within the shutdown budget. Also test a request that
exceeds the budget so process exit remains bounded.

### Clean database reproduction

Destroy only the disposable local database state, recreate it, apply every migration,
and rerun the acceptance workflow using documented commands. Never use this step
against valuable or ambiguously targeted data.

## 10. Conduct a 30-minute incident drill

Use a local or isolated non-production environment.

### Scenario

PostgreSQL becomes unavailable while clients place and read orders.

### Roles

One person can perform all roles during learning, but label them:

- incident commander keeps time and decisions;
- investigator uses telemetry;
- operator applies mitigation;
- recorder maintains the timeline.

### Drill sequence

1. Record the healthy baseline and exact start time.
2. Introduce the database failure.
3. Detect it through the chosen alert or dashboard, not by inspecting the sabotage.
4. Classify user impact by route and error/latency signal.
5. Correlate one failed request across problem code, log, trace, and database metric.
6. Verify liveness remains healthy and readiness removes the instance from service.
7. Apply the documented recovery action.
8. Verify database health, readiness, request success, latency, and order integrity.
9. Declare recovery using explicit criteria.
10. End at 30 minutes even if unresolved and record the exact blocker.

### Timeline template

```text
00:00 Baseline captured
00:03 Failure introduced
00:__ First detection
00:__ Impact confirmed
00:__ Mitigation chosen
00:__ Database restored
00:__ Readiness recovered
00:__ User workflow verified
00:__ Incident closed
```

### Review

Document detection delay, diagnosis evidence, actions that helped, unsafe or missing
runbook steps, actual recovery time, data-integrity checks, and corrective actions
with owners. Focus on system improvements, not blame.

## 11. Make the repository sufficient for another developer

The root documentation should give a clean-clone path containing:

- required tool versions;
- configuration names with safe examples;
- local PostgreSQL startup and health check;
- migration commands;
- application run command;
- unit, integration, race, vet, and vulnerability commands;
- container build and run commands;
- expected health endpoints;
- common failure symptoms and fixes;
- safe cleanup limited to disposable resources.

Test these instructions from a clean checkout or equivalent isolated directory.
Commands copied from shell history are not documentation until another person can
execute them successfully.

## 12. Guided delivery build

Complete these steps in order:

1. Add a multi-stage, non-root container build and `.dockerignore`.
2. Validate typed runtime configuration and missing-config startup behavior.
3. Define repository-owned verification commands.
4. Add CI using those same commands and a disposable PostgreSQL service.
5. Verify migrations from an empty database before integration tests.
6. Build and run the exact container artifact with runtime-only configuration.
7. Test readiness loss and recovery plus bounded termination.
8. Document migration compatibility, rollback, and roll-forward decisions.
9. Perform a restore drill or document the exact environment prerequisite if no
   backup system exists yet.
10. Conduct and record the 30-minute database incident drill.
11. Have another developer follow only repository documentation.

## 13. Optional non-production deployment

Deploy the immutable artifact to an isolated environment. Apply migrations through
the controlled release step, exercise the public workflow, observe telemetry, and
perform either a rehearsed rollback or roll-forward. Record artifact digest, schema
version, timings, and verification evidence.

Do not introduce Kubernetes merely to complete this stretch goal. Use the simplest
available non-production platform; orchestration is a later phase.

## 14. Knowledge check

1. Why promote one artifact instead of rebuilding per environment?
2. What belongs in build input, runtime configuration, and secret storage?
3. Why should CI call repository-owned commands?
4. Why must old and new app versions coexist with a migration?
5. When is rollback unsafe?
6. What does a restore drill prove that a backup-success message does not?
7. Why is database pool size multiplied across instances?
8. What should happen to readiness and liveness during a database outage?
9. What evidence closes an incident?

### Expected answers

1. The tested artifact retains the same provenance and bytes through promotion.
2. Tool/source choices at build time, non-secret environment behavior at runtime,
   and credentials in a controlled secret mechanism.
3. Local and CI behavior stay reproducible and do not drift into two workflows.
4. Rolling releases and rollback can run multiple versions against one schema.
5. When the current schema/data is incompatible with the old version or reversal
   would destroy data.
6. That retained bytes can recreate usable, internally consistent application data
   within measured recovery goals.
7. PostgreSQL sees the sum of every process pool plus operational reserve.
8. Readiness fails promptly; liveness remains healthy because restart cannot repair
   the dependency.
9. User-visible signals recover, data integrity is checked, and the recovery remains
   stable for the defined observation period.

## 15. Completion evidence

Day 7 is complete when:

- the image contains the runtime binary without source, toolchain, or credentials
  and runs as non-root;
- configuration is validated before serving and secret values are never logged;
- local and CI verification use the same repository-owned commands;
- a clean database reaches the current schema only through migrations;
- the application survives readiness loss/recovery and drains on termination within
  a fixed deadline;
- rollback or roll-forward compatibility is documented and rehearsed where possible;
- backup claims are supported by restore evidence or explicitly marked not yet
  established;
- a 30-minute incident timeline demonstrates detection, diagnosis, mitigation, and
  recovery verification;
- another developer completes the documented clean-clone workflow without hidden
  steps.

The seven-day readiness track is then complete. Continue with Phase 0 of the longer
implementation roadmap, using the phase refresher before each new deliverable.
