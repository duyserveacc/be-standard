# Backend Implementation Phases: Key-Concept Refresher

Use this guide to restore the mental model behind a phase quickly. It intentionally does not repeat the roadmap's deliverables or acceptance criteria.
The [backend learning roadmap](../BACKEND_LEARNING_ROADMAP.md) remains authoritative for what to build and how completion is judged.

## How the phases fit together

| Range | Focus | Question it answers |
| --- | --- | --- |
| Phase 0 | Domain understanding | What problem and rules are we actually implementing? |
| Phases 1-10 | One production service | Can one service preserve data, fail safely, and be operated? |
| Phases 11-13 | Asynchrony and distribution | Can work survive restarts, duplication, delay, and partitions? |
| Phases 14-18 | Service extraction | Can independently deployed services remain compatible and diagnosable? |
| Phases 19-23 | Scale and operations | Can the system isolate tenants, run securely, and meet reliability goals? |
| Phases 24-26 | Integration and leadership | Can the system evolve safely and can its decisions be defended? |

The order matters. Distribution magnifies weak domain models, unsafe data handling, vague contracts, and poor observability. Build those foundations before adding service boundaries.

## Phase 0: Domain discovery and modeling

### Core ideas

- Requirements are hypotheses until examples expose their ambiguities. Turn statements such as "cancel an order" into concrete preconditions, transitions, outcomes, and rejected cases.
- A ubiquitous language gives each important term one agreed meaning across product discussion, code, APIs, events, and operations.
- Entities retain identity through change; value objects are interchangeable when their values match. Commands request change, while events name facts that already happened.
- Invariants are rules that must always hold. A state machine makes legal transitions, terminal states, and invalid actions explicit.
- A bounded context owns a coherent model and vocabulary. It is a reasoning boundary, not automatically a deployment boundary.

### Memory anchors

- Model business behavior before transport or storage.
- Examples reveal rules; invariants protect them; state machines expose gaps.
- Keep confirmed facts, assumptions, open questions, and deferred scope visibly separate.

### Self-check

- Can I explain why `Product` may mean different things in catalog and inventory contexts?
- Can I list every valid and invalid transition for an order without referring to HTTP or SQL?
- Which business rule must remain true during concurrent order placement?

## Phase 1: Decisions and executable skeleton

### Core ideas

- The composition root constructs concrete dependencies. Business packages should not discover configuration or initialize global infrastructure themselves.
- Parse configuration once into typed values, validate it at startup, and fail early with actionable errors.
- An architecture decision record captures context, options, the chosen tradeoff, and consequences. It preserves reasoning, not just the final choice.
- Graceful shutdown is a bounded protocol: stop accepting work, allow in-flight work to finish within a deadline, then release resources.
- Local and CI commands should exercise the same formatting, static analysis, tests, race checks, vulnerability checks, and build.

### Memory anchors

- A clean clone should have one reproducible path to a running service.
- Startup either establishes valid preconditions or fails loudly.
- Record why a decision was made so it can later be revisited intelligently.

### Self-check

- What owns dependency construction and process lifecycle?
- What happens from `SIGTERM` until process exit?
- Can every CI failure be reproduced locally with a documented command?

## Phase 2: HTTP platform

### Core ideas

- HTTP handlers translate between a public protocol and application operations. They should decode, validate transport concerns, call a use case, and map outcomes to stable responses.
- Enforce content type, body size, unknown-field rejection, and exactly one JSON value before untrusted input reaches business logic.
- Problem details give errors a stable machine-readable shape without exposing internal implementation details.
- Middleware order is behavior: request identity must exist before logging and recovery need it; authentication must run before authorization.
- Liveness answers "is the process alive?" Readiness answers "should this instance receive traffic?"
- Server, header, body, and shutdown limits keep slow or malicious clients from holding resources indefinitely.

### Memory anchors

- Validate the envelope at the edge and business rules in the use case.
- A panic becomes a generic response plus a correlated diagnostic record.
- Every request consumes bounded time and memory.

### Self-check

- Which malformed requests should be rejected before a use case runs?
- Why must liveness avoid checking the database?
- Where is an internal error translated into its public status and error code?

## Phase 3: Persistence foundation

### Core ideas

- A connection pool is a bounded concurrency resource. Its size, acquisition timeout, connection lifetime, and health affect the whole service.
- Database constraints are the final integrity barrier when application checks race or another writer bypasses the service.
- Parameterized queries keep data separate from executable SQL and prevent injection.
- Context cancellation must reach pool acquisition and query execution. Rows and transactions must be closed on every path.
- Migrations are ordered production changes. They must work from an empty database and account for existing data and mixed application versions.
- Integration tests use a real database because mocks cannot reproduce constraints, locks, isolation, query syntax, or driver behavior.

### Memory anchors

- Enforce critical invariants twice: expressive application rules and authoritative database constraints.
- Acquire late, release early, and bound the pool.
- Treat schema changes like code deployments.

### Self-check

- What protects uniqueness if two requests pass an application-level pre-check together?
- Which resources must be closed or rolled back after a failed query?
- Can the complete schema be reproduced from migrations alone?

## Phase 4: Product slice

### Core ideas

- A vertical slice keeps the transport, use case, persistence, and tests for one capability close while preserving inward dependency direction.
- Authentication establishes the principal. Authorization decides whether that principal may perform a specific action.
- Stable cursor pagination encodes the ordered position of the last result. Its ordering must be deterministic, commonly using a unique tie-breaker.
- Domain/application errors should have stable identity so the HTTP layer can map them without inspecting error text.
- Duplicate creation is a conflict only when the uniqueness rule is part of the public contract.

### Memory anchors

- Organize around capabilities, not global controller/service/repository buckets.
- A cursor is a position in a stable ordering, not a disguised page number.
- Keep unauthenticated and authenticated-but-forbidden outcomes distinct.

### Self-check

- What columns form a total order for product pagination?
- Where is admin authorization enforced and tested?
- How does a unique-constraint failure become a stable API conflict?

## Phase 5: Order transaction

### Core ideas

- A transaction boundary follows one business operation. Either the order, its items, and inventory reservation all commit, or none do.
- Concurrency correctness requires a database strategy such as an atomic conditional update, row lock, or appropriate isolation level. An earlier application read is not protection.
- Lock acquisition order should be deterministic to reduce deadlocks when an order touches multiple inventory rows.
- Object-level authorization belongs in the data access condition: select the order by both order ID and owner ID.
- Public errors reveal what clients need to act, not table names, SQL, or whether another user's object exists.

### Memory anchors

- Read-check-write is unsafe unless the database makes it one protected operation.
- Commit the business invariant, not individual repository calls.
- Scope the query, not just the response.

### Self-check

- How can two buyers contend for the last unit without producing negative stock?
- Which failures cause rollback, and who owns that rollback?
- Why should wrong-owner and nonexistent-order reads look the same externally?

## Phase 6: Idempotency and failure semantics

### Core ideas

- Idempotency lets repeated attempts for the same logical operation produce one effect. It is essential when a client cannot tell whether a timed-out request committed.
- The idempotency key identifies the attempt; a request fingerprint prevents that key from being reused for different input.
- Store the operation state and completed response within the same correctness boundary as the business effect, or explicitly handle the gap.
- Concurrent duplicates need serialization: one request performs the operation while others wait for or reuse its outcome.
- Retention must balance the client's retry window against storage growth. Expiry changes how old retries behave and is part of the contract.

### Memory anchors

- Same key plus same payload means same logical operation.
- Same key plus different payload is a conflict, not a retry.
- A disconnected client must be able to retry without guessing whether to create again.

### Self-check

- What exact data is fingerprinted, and is its encoding canonical?
- What happens if the process dies after the order commits but before the response is sent?
- When may an idempotency record be deleted safely?

## Phase 7: Observability and operational controls

### Core ideas

- Logs explain discrete events, metrics reveal aggregate trends, and traces connect work across boundaries. None substitutes completely for the others.
- Correlation identifiers and trace context let an operator follow a request without logging sensitive payloads.
- Useful service signals cover traffic, errors, latency distributions, and saturation. Audit events separately record security-relevant actions.
- Metric labels must have bounded cardinality. User IDs, order IDs, raw URLs, and error strings can make telemetry unusable or expensive.
- Alerts should identify actionable sustained user impact or imminent exhaustion, not every internal anomaly.
- A runbook joins symptoms to investigation, mitigation, rollback, and escalation.

### Memory anchors

- Instrument for questions an operator will need to answer under pressure.
- High-cardinality identifiers belong in traces or logs, not metric labels.
- Observability includes the action path, not just the detection signal.

### Self-check

- Can I reconstruct one failed request from its public error to the underlying dependency?
- Which labels are bounded, and which could grow without limit?
- What precise operator action follows each page?

## Phase 8: Deployment hardening

### Core ideas

- Build one immutable artifact and vary runtime configuration between environments. Rebuilding per environment destroys provenance.
- Containers should run as a non-root user, write only to explicit ephemeral locations, and expose the smallest practical attack surface.
- Readiness gates traffic; graceful shutdown drains it. Their timings must fit the platform's rollout and termination behavior.
- Expand-and-contract migrations allow old and new application versions to coexist: add compatible schema, migrate usage/data, then remove old schema later.
- A backup is only a possibility; a successful restore drill is evidence.
- Resource requests and limits turn accidental unbounded use into explicit operational behavior.

### Memory anchors

- Artifact promotion, runtime configuration, and verified provenance belong together.
- Deploy application and schema changes for mixed-version operation.
- Recovery claims require restores, not checkboxes.

### Self-check

- Can the old version run safely after the new migration is applied?
- What happens to an in-flight request when the platform terminates a container?
- When was the last restore tested and what was verified afterward?

## Phase 9: Performance and resilience based on evidence

### Core ideas

- Throughput is completed work per time; latency is a distribution, not an average. Utilization and saturation show how close a bounded resource is to its limit.
- Queueing causes latency to rise sharply near capacity. Load generation must avoid coordinated omission, which hides delays by slowing request issuance when the system slows.
- Profile the limiting workload before optimizing. CPU, heap, allocations, goroutines, mutex contention, blocking, query plans, and dependency time answer different questions.
- Representative data shape, traffic mix, concurrency, warmup, and environment are part of a performance result.
- Overload controls reject or shed excess work early so accepted work can still complete.
- Caching adds invalidation and consistency costs; introduce it only after evidence identifies repeated expensive reads.

### Memory anchors

- Measure, locate the bottleneck, change one thing, measure again.
- Tail latency and saturation usually matter more than averages.
- Predictable rejection is safer than global exhaustion.

### Self-check

- What workload assumptions support the latency target?
- Which profile proves the suspected bottleneck?
- How does the service behave just beyond capacity?

## Phase 10: Advanced PostgreSQL operations and recovery

### Core ideas

- MVCC gives transactions snapshots while retaining old row versions. Isolation level determines which anomalies are prevented and what may block or abort.
- Deadlocks arise from cyclic waits; PostgreSQL aborts one participant. Consistent lock order prevents many deadlocks, while safe whole-operation retries handle expected transient aborts.
- Vacuum, statistics, bloat, checkpoints, and WAL affect query planning, storage reuse, write latency, replication, and recovery.
- Large migrations should be lock-aware, batched, resumable, observable, and verifiable. A transaction around everything is not automatically safer.
- Base backups plus continuous WAL archiving enable point-in-time recovery. Replication and standby promotion primarily address availability, not accidental deletion by themselves.
- RPO measures tolerable data loss; RTO measures tolerable recovery time. Both must be proven by drills.

### Memory anchors

- Isolation is a chosen anomaly/availability tradeoff, not a generic "safe" switch.
- Retry only the entire operation and only when repeating its effects is safe.
- Backup, restore, replication, and failover solve different problems.

### Self-check

- Which anomaly can occur at the selected isolation level?
- How does an interrupted backfill resume without skipping or duplicating work?
- What measured evidence supports the stated RPO and RTO?

## Phase 11: Durable background jobs and scheduling

### Core ideas

- A goroutine disappears with its process. Durable jobs persist intent outside the worker and survive crashes and deployments.
- Most queues provide at-least-once execution: a job can run more than once, especially if a worker finishes the side effect but dies before acknowledgement.
- A visibility lease temporarily grants work. Expired work becomes available again, so handlers must be idempotent.
- Retry only transient failures, use bounded exponential backoff with jitter, and quarantine poison jobs after a documented attempt limit.
- Backpressure and bounded worker concurrency prevent the job system from overwhelming its dependencies.
- Schedulers must assume multiple replicas may trigger the same period. Make the scheduled action idempotent or coordinate it safely.

### Memory anchors

- Persist intent, lease work, make effects idempotent, then acknowledge.
- Queue age shows user delay better than queue length alone.
- Shutdown stops intake before finishing or safely releasing leased work.

### Self-check

- What happens if a worker dies after sending an email but before acknowledging the job?
- Which errors retry, which fail permanently, and which go to quarantine?
- Can two scheduler replicas trigger the cleanup safely?

## Phase 12: Reliable event publication and consumption

### Core ideas

- A database transaction cannot normally commit atomically with a broker publish. The transactional outbox commits domain state and publication intent together.
- A relay repeatedly publishes unpublished outbox entries. It must tolerate publishing successfully and crashing before recording success.
- Consumers assume duplicate delivery. An inbox or deduplication record must commit atomically with the consumer's state change when duplicate effects would be unsafe.
- Ordering exists only within a defined scope, often a partition key. Global ordering is expensive and usually unnecessary.
- Event contracts need ownership, semantic meaning, compatibility rules, version strategy, retention, and replay expectations.
- "Exactly once" always has a boundary; external side effects still require idempotency or reconciliation.

### Memory anchors

- Outbox prevents lost publication; inbox prevents unsafe repeated consumption.
- Publish facts in past tense, with enough identity to deduplicate and correlate.
- Design replay before relying on it for recovery.

### Self-check

- What happens if the relay crashes immediately after the broker accepts an event?
- Which key defines ordering for `OrderPlaced` events?
- Can replay rebuild the same final state without repeating an external side effect?

## Phase 13: Distributed-systems mechanics lab

### Core ideas

- Distributed failure is partial: one direction, node, or message can fail while everything else appears healthy.
- Wall clocks can jump and disagree. Use monotonic time for elapsed durations and avoid inferring a total event order from timestamps alone.
- Quorum choices trade availability, consistency, and latency under partitions; the useful answer depends on the operation and failure model.
- A lease says an owner may act for a time, but a paused old owner can resume after expiry. A monotonically increasing fencing token lets the protected resource reject stale owners.
- Consensus establishes a replicated order despite failures. Application teams should use maintained implementations rather than inventing their own.
- Deterministic simulation makes delay, loss, duplication, and reordering reproducible without fragile sleeps.

### Memory anchors

- Timeouts create uncertainty; they do not prove the remote operation failed.
- Leases need fencing when stale owners can still reach the resource.
- Consistent hashing limits key movement; it does not provide consistency.

### Self-check

- Which operations remain available during a partition, and what can they return?
- How does a protected resource recognize a stale lease holder?
- What reconciliation rule runs after partition healing?

## Phase 14: Service-boundary discovery and first extraction

### Core ideas

- A service boundary should align with a business capability, data ownership, and team responsibility, not a table or technical layer.
- A distributed monolith has separate deployments but retains tight coordination, shared data, or lockstep releases, giving cost without autonomy.
- The owning service is the only writer and contract authority for its data. Cross-service table reads bypass that authority.
- Extraction adds network latency, partial failure, compatibility work, telemetry, deployment pipelines, security boundaries, and on-call burden.
- Branch by abstraction introduces a stable interface, runs old and new implementations during migration, and shifts traffic incrementally.

### Memory anchors

- Extract to gain measurable autonomy or isolation, not to obtain a fashionable topology.
- Data ownership is stricter than "different schemas in the same database."
- Prefer a low-risk asynchronous capability for the first extraction.

### Self-check

- What evidence shows this module changes or scales independently?
- Can the extracted service fail without blocking order placement?
- Which operational costs increased after the split?

## Phase 15: Synchronous service communication and the system edge

### Core ideas

- Choose HTTP/JSON or gRPC from client compatibility, tooling, contract, streaming, and latency needs, not slogans.
- A caller owns an end-to-end deadline budget. Connection, retries, and downstream processing must fit within the remaining time.
- Retries multiply across layers. Select one intentional layer, bound attempts, add backoff and jitter, and retry only idempotent or protected operations.
- Circuit breakers limit repeated calls to a failing dependency; bulkheads limit shared-resource damage; load shedding rejects excess work; fallbacks must be semantically honest.
- Gateways centralize edge concerns such as authentication, TLS, routing, and coarse rate limits. Business authorization stays with the service that owns the rule.
- Connection pools and concurrency limits must be bounded per dependency.

### Memory anchors

- Propagate cancellation and spend one finite deadline budget.
- A retry is extra load during failure; earn it with idempotency and a clear benefit.
- Bound waiting, attempts, connections, and concurrency.

### Self-check

- Can any outbound call outlive its parent request?
- Which single layer retries, and how is retry multiplication prevented?
- What response does the caller receive when inventory is unavailable?

## Phase 16: Distributed workflows and eventual consistency

### Core ideas

- Independently owned databases cannot share an ordinary local ACID transaction. Intermediate states are therefore part of the domain model.
- A saga is a durable sequence of local transactions. Orchestration centralizes progression; choreography reacts to events and can become harder to understand as interactions grow.
- Compensation is a new business action, not time travel. It can fail and must be idempotent, observable, and sometimes manual.
- Durable workflow state records current step, attempts, deadlines, correlation, and the decisions needed to resume after a crash.
- Delayed, duplicate, and out-of-order messages are normal inputs. Transition guards must make their handling explicit.
- Operators need safe commands to inspect, retry, compensate, or mark resolved without editing tables by hand.

### Memory anchors

- Model temporary inconsistency explicitly and bound how long it may last.
- Every step and compensation must survive repetition.
- A workflow is incomplete until stuck-state detection and repair exist.

### Self-check

- What valid state follows failure at each transition?
- What happens when confirmation arrives after cancellation?
- Which repair actions require human judgment and an audit trail?

## Phase 17: Contract evolution and independent delivery

### Core ideas

- Independent deployment means provider and consumer versions can coexist inside a documented compatibility window.
- Compatibility is semantic as well as structural. Changing defaults, validation, ordering, timing, or error meaning can break a consumer without changing schema shape.
- Additive fields are generally safer when readers ignore unknown data and writers do not make new data immediately mandatory.
- Consumer-driven contract tests express behavior real consumers depend on; provider verification checks those expectations against the provider build.
- Contract tests complement integration and end-to-end tests because they do not prove infrastructure, configuration, or whole workflows.
- Deprecation needs an owner, usage telemetry, notice, migration path, deadline, and evidence before removal.

### Memory anchors

- Design changes for mixed versions and both deployment orders.
- Schemas describe shape; tests and prose must capture semantics.
- Removal is a measured migration, not a date on a calendar.

### Self-check

- Can the consumer deploy before the provider and vice versa?
- Which seemingly additive change could still alter existing behavior?
- What evidence proves a deprecated contract has no remaining callers?

## Phase 18: Distributed observability and resilience validation

### Core ideas

- Trace context must propagate through HTTP/RPC calls, event metadata, and background jobs so one logical operation remains connected.
- Baggage propagates broadly and can leak or amplify sensitive/high-cardinality data; keep it minimal and safe.
- Distributed systems create emergent failures: retry storms, synchronized recovery, fan-out amplification, and backlog growth may not appear in component tests.
- Tail-aware or error-biased sampling preserves diagnostically valuable slow and failed traces while controlling cost.
- A chaos experiment needs a hypothesis, bounded blast radius, abort condition, observation plan, and proof of recovery.
- Symptom-based runbooks start from what users see and guide the operator to the first failing dependency.

### Memory anchors

- Correlate the whole workflow, including asynchronous gaps.
- Inject faults to test a specific claim, not to create random drama.
- Recovery behavior is part of the experiment result.

### Self-check

- Can one order be traced through API, broker, worker, and downstream services?
- Which telemetry distinguishes an originating failure from cascading symptoms?
- What automatically stops a fault experiment if impact exceeds its boundary?

## Phase 19: Data scaling, caching, and multi-tenancy

### Core ideas

- Cache-aside reads from cache first, loads from the source of truth on a miss, then populates the cache. TTL, invalidation, stampede protection, bypass, and failure behavior define its consistency.
- Read replicas reduce primary read load but introduce lag. Route only operations whose staleness and read-after-write behavior are acceptable.
- Partitioning limits the amount of data each operation scans or maintains; archival moves old data; retention deletes data. They solve different growth problems.
- Tenant isolation spans identity, queries, jobs, events, caches, quotas, encryption, telemetry, and operator tools, not just a `tenant_id` column.
- Shared-table, schema-per-tenant, and database-per-tenant designs trade density, isolation, operational effort, and customization.
- Search and analytics are derived views that require replay, rebuild, and reconciliation with an authoritative source.

### Memory anchors

- The source of truth must remain correct when every cache is empty or stale.
- Propagate tenant identity across every synchronous and asynchronous boundary.
- Scale from measured pressure, with rollback and reconciliation.

### Self-check

- What read-after-write behavior does a user receive?
- Can a tenant identifier be omitted or forged in any worker, cache key, or query?
- How is a corrupted search index rebuilt completely?

## Phase 20: Security, privacy, and software supply chain

### Core ideas

- Threat modeling follows data and trust boundaries: identify assets, actors, entry points, abuse cases, mitigations, and residual risk.
- User identity represents an end user; workload identity authenticates a service or job. Do not grant a service every privilege any user might need.
- Mutual authentication, encryption in transit, short-lived credentials, least privilege, and rotation reduce lateral movement and credential lifetime.
- Privacy engineering inventories personal data by purpose, owner, location, retention, access, export, and deletion behavior.
- Backups and immutable event logs may require policy-based expiry or cryptographic erasure rather than immediate physical deletion; document the limitation honestly.
- An SBOM records components; provenance and signatures connect source, build, and artifact; vulnerability response turns detection into a verified deployment.

### Memory anchors

- Every new service, queue, datastore, and admin endpoint is a trust boundary.
- Collect less, retain less, grant less, and rotate safely.
- Supply-chain controls must verify what is deployed, not merely scan source code.

### Self-check

- What can a stolen workload credential access, and for how long?
- Where does each sensitive field flow and when is it deleted?
- Can a vulnerable artifact be identified, rebuilt, verified, and replaced quickly?

## Phase 21: Cloud infrastructure and IAM

### Core ideas

- Accounts/projects are administrative boundaries; regions and availability zones are failure and latency boundaries. Design assumptions must match the provider's actual guarantees.
- Routes determine possible paths; firewalls/security groups restrict allowed paths; NAT enables controlled outbound access; private endpoints avoid public transit to managed services.
- DNS, certificates, TLS termination, load balancers, and health checks form a layered edge whose failures must be distinguishable.
- Human and workload access should use federated or short-lived identity, narrowly scoped roles, and audited elevation, not shared static credentials.
- Managed services transfer infrastructure work but applications still own schema, access policy, capacity choices, failure handling, and recovery validation.
- Infrastructure as code requires protected state, review, policy checks, drift detection, ownership labels, budget controls, and precisely scoped teardown.

### Memory anchors

- Default-deny connectivity and least-privilege identity reduce the reachable blast radius.
- Reproducibility means no required console-only steps.
- Teardown is a dangerous production operation and must resolve exact ownership boundaries.

### Self-check

- Can the database or an admin endpoint be reached from the public internet?
- Which exact resources can each workload identity access?
- Can the environment be recreated and safely removed from version-controlled definitions?

## Phase 22: Orchestration and platform operations

### Core ideas

- An orchestrator continuously reconciles desired and observed state. It restarts processes but cannot repair incorrect application state or unsafe retry behavior.
- Resource requests guide scheduling; limits cap use; autoscaling reacts to signals; disruption budgets and rollout settings constrain simultaneous unavailability.
- Startup means initialization is complete, readiness means traffic is safe, and liveness means restart may help. Confusing them causes restart loops or premature traffic.
- Services, ingress, configuration, secret references, network policies, and volumes are explicit platform contracts.
- Autoscaling on a poor signal can amplify overload downstream. Scale from a bottleneck-relevant signal and respect database or broker capacity.
- Platform abstractions should provide safe defaults while preserving visibility into failures and generated resources.

### Memory anchors

- Reconciliation restores declared process state, not business correctness.
- Probe semantics affect rollouts and recovery directly.
- Scale the system bottleneck, not merely the frontmost workload.

### Self-check

- What happens during node loss, failed readiness, and termination?
- Can a rollout preserve capacity while old and new contracts coexist?
- Which signal scales workers without overwhelming their dependency?

## Phase 23: Reliability engineering, incidents, and cost

### Core ideas

- An SLI is a measured aspect of user experience; an SLO is its target over a window. The error budget is the allowed unreliability within that target.
- Good paging alerts indicate actionable user impact or imminent exhaustion and balance detection speed against false alarms.
- Capacity planning relates expected demand to resource saturation, dependency ceilings, headroom, growth, and recovery needs.
- Incident response separates coordination, investigation, communication, and mitigation roles so urgent work stays coherent.
- A blameless review explains contributing system conditions, evidence, response quality, and corrective actions without reducing the cause to one person's mistake.
- Cost includes infrastructure, engineering time, operational complexity, and lost opportunities. A cheaper resource can create a more expensive system.

### Memory anchors

- Reliability targets begin with the user's successful operation.
- Error budgets make the reliability-versus-delivery tradeoff explicit.
- Corrective actions need owners, deadlines, verification, and evidence of completion.

### Self-check

- Does the SLI measure user success or an internal proxy?
- What is the first saturated dependency at forecast traffic plus headroom?
- Could an on-call engineer act on every page without additional context?

## Phase 24: Common production integration boundaries

### Core ideas

- Third-party calls need narrow adapters, explicit timeouts, quotas, retry policy, stable internal errors, fakes, sandbox tests, and operational visibility.
- A timeout is ambiguous: the provider may have completed the request. Use provider idempotency keys, status lookup, or reconciliation before repeating effects.
- Webhook receivers verify a signature over the exact raw payload, enforce a timestamp/replay window, deduplicate deliveries, and acknowledge according to a documented retry contract.
- Large object transfers should stream directly to object storage with authorization, size/type limits, checksums, lifecycle, and asynchronous scanning or processing.
- Email/notification success is asynchronous; bounces, complaints, suppressions, preferences, and provider callbacks change later state.
- Every derived read model needs a complete rebuild and reconciliation path from authoritative data or events.

### Memory anchors

- Treat external systems as slow, quota-bound, fallible, and semantically ambiguous.
- Authenticate and deduplicate webhooks before applying effects.
- Stream large data; never buffer it without a strict bound.

### Self-check

- What happens if a provider times out after accepting a write?
- How are forged, replayed, duplicated, and reordered webhooks handled?
- Can a derived view be deleted and rebuilt without losing authoritative data?

## Phase 25: Maintenance and legacy-system evolution

### Core ideas

- Actual behavior is discovered from code, tests, telemetry, stored data, documentation, and user dependence. Documentation alone may describe intended rather than current behavior.
- Characterization tests pin important existing behavior before refactoring or changing poorly specified code.
- Feature flags, canaries, shadow traffic, and compatibility windows make change observable and reversible, but temporary mechanisms become debt if not removed.
- Upgrades require release-note review, compatibility tests, staged rollout, and a rollback or roll-forward plan specific to the changed layer.
- Data repair is production code: constrain scope, support dry run and resume, make it idempotent, record an audit trail, and verify results independently.
- Refactoring preserves public behavior; migration deliberately changes behavior through compatible steps.

### Memory anchors

- First learn what the system does, then protect it, then change it.
- Reversibility is designed before rollout.
- Delete flags, compatibility branches, and obsolete dependencies when the migration is proven complete.

### Self-check

- Which evidence defines the legacy behavior that must be preserved?
- Can the rollout be stopped or reversed after partial exposure?
- How does a repair command prove it cannot cross its declared scope?

## Phase 26: Capstone delivery and engineering leadership

### Core ideas

- Engineering leadership connects a user problem to requirements, alternatives, tradeoffs, implementation, tests, telemetry, rollout, operations, and cost.
- A design document makes assumptions and decisions reviewable before implementation expense hardens them. It should include rejected alternatives and consequences.
- Critical review is part of the work. Record what changed because of feedback and why any unresolved risk is accepted.
- Small compatible changes reduce review size, isolate risk, and allow evidence to influence later steps.
- Operational ownership includes migration, staged rollout, dashboards, failure drills, recovery rehearsal, and a handoff another engineer can use.
- Simplicity is an active design criterion: be able to name what could be removed if the requirements shrink.

### Memory anchors

- Claims should be traceable from requirement to code, test, telemetry, and runbook.
- Deliver evidence, not confidence statements.
- Ownership includes reviewing others and teaching the system clearly.

### Self-check

- Can each requirement be traced to implementation and production-like evidence?
- Which failure, concurrency, restart, duplicate, and mixed-version cases matter for this design?
- What complexity would I remove first if the problem became simpler?

## Cross-phase recall checklist

Before calling any phase complete, be able to answer these questions:

- **Contract:** What inputs, outputs, errors, compatibility promises, and ownership boundaries are public?
- **Data:** Which invariants are authoritative, where are they enforced, and what happens under concurrency?
- **Time:** Which deadlines, leases, retention periods, retry windows, and clock assumptions exist?
- **Resources:** Which pools, queues, payloads, retries, workers, and cardinalities are bounded?
- **Failure:** What can fail partially, what becomes ambiguous, and how does the system converge or get repaired?
- **Security:** Who or what is the principal, what may it access, and how small is credential exposure?
- **Observability:** Can an operator detect, diagnose, mitigate, and verify recovery without inspecting private data?
- **Delivery:** Can old and new versions coexist, and is rollback or roll-forward tested?
- **Evidence:** Which deterministic test, measurement, drill, or review supports each important claim?
- **Simplicity:** Which complexity is essential now, and which can be deferred or removed?
