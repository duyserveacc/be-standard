# Phases 11-13: Durable and Distributed Foundations

These phases introduce asynchronous work and partial failure while the production
architecture still relies on maintained databases, brokers, and coordination tools.

## Phase 11: Durable Background Jobs and Scheduling

### Goal and mental model

A goroutine disappears with its process. Durable work records intent before it can
be acknowledged and assumes at-least-once execution: a worker may complete an effect
and crash before marking the job complete.

A queue record needs identity, type, payload or reference, state, attempts,
availability time, lease owner/expiry, last error category, and timestamps. Keep
payloads bounded and versioned. Prefer durable references when data is large or
sensitive.

Workers claim a bounded batch in a short transaction. PostgreSQL queues commonly use
`FOR UPDATE SKIP LOCKED` so workers claim different available rows without waiting
behind one another. A lease makes abandoned work eligible again after expiry; it is
not proof that the old worker stopped. Every claim needs a monotonically increasing
attempt or random claim token. Completion, retry, and lease extension update only
when both job ID and current claim token match; a resumed stale worker cannot
acknowledge or overwrite a newer claim. Handlers still remain idempotent because
external effects can occur before this guarded acknowledgement.

Classify results:

- success: record completion;
- transient: schedule another bounded attempt with exponential backoff and jitter;
- permanent: move directly to dead-letter/quarantine;
- attempts exhausted: dead-letter with operator-visible reason.

Queue age shows how long users wait. Length alone can look harmless while one old
poison job or insufficient worker capacity violates the objective.

Schedulers run on more than one replica. Prefer an idempotent, concurrency-safe
operation such as deleting expired idempotency rows in bounded batches. If exactly
one active scheduler is required, use a maintained lease/lock with clear expiry and
fencing rather than an in-memory boolean.

### Guided work

1. Define the confirmation-job schema, states, bounded payload, and retention.
2. Insert the job in the same transaction as the state requiring it.
3. Build a separate worker command with configurable bounded concurrency.
4. Claim available rows atomically using lease owner, expiry, and a unique claim
   token; guard every acknowledgement and retry update with that token.
5. Make the confirmation handler idempotent using a durable effect identity.
6. Classify retryable and permanent failures; cap attempts and backoff.
7. Add dead-letter inspection, replay, and quarantine commands with audit records.
8. Add an idempotent multi-replica cleanup scheduler.
9. Expose oldest age, ready/leased/dead counts, attempts, duration, and failures.
10. Implement shutdown: stop claims, finish or release leased work, then exit by
   deadline.

### Failure checks

Kill a worker before effect, after effect, and before acknowledgement. Expire a lease
while a paused worker later resumes, then prove its old claim token cannot modify the
new attempt. Feed one permanently invalid job among healthy jobs. Run two scheduler
replicas. Fill the queue faster than workers and verify bounded storage/backpressure
plus rising age alerts.

### Knowledge check and expected answers

1. **Why is a lease not exactly-once execution?** The old worker can resume after
   expiry, so two attempts may overlap.
2. **Why must a handler be idempotent?** Completion and acknowledgement cannot be one
   atomic action across every external side effect.
3. **Why monitor oldest age?** It measures user-visible delay and detects stuck work
   that total count can hide.

### Completion evidence

Worker-kill tests lose no job and create only safe duplicate attempts; poison work
reaches a bounded dead-letter state; two schedulers cause no unsafe duplicate effect;
backlog and oldest age are observable; operator replay is scoped/audited; and shutdown
finishes or releases leases within its deadline.

## Phase 12: Reliable Event Publication and Consumption

### Goal and mental model

A local database commit and broker publish are separate failure domains. Publishing
after commit can lose the event on crash; publishing before commit can announce a
change that later rolls back. The transactional outbox stores both business state
and publication intent in one database transaction.

An event inserted only after the business commit is not a transactional outbox. If an
event is deliberately derived from authoritative state instead, name that weaker
guarantee and run a bounded, tested reconciliation process that reconstructs missing
intent after crashes.

An outbox event includes immutable event ID, aggregate ID, event type, schema version,
occurred time, bounded payload, ordering key, attempt state, and publication metadata.
`OrderPlaced` is written beside the order before commit.

A relay claims unpublished events, publishes, then records progress. If it crashes
after broker acceptance but before marking published, it publishes again. That is
expected. Consumers assume duplicates.

An inbox/deduplication record is written in the same transaction as the consumer's
state change. If event ID already exists, the consumer acknowledges without repeating
the effect. A dedupe record written separately before the effect can lose work; one
written separately after the effect can repeat it.

Ordering is scoped, not global. Choose a partition/order key such as order ID when
events for one order must remain ordered. Different orders can progress independently.
Consumers must still handle redelivery and version-compatible older events.

"Exactly once" must name the boundary. A broker may deduplicate its log writes while
an email provider still receives a duplicate request. End-to-end safety comes from
idempotent effect identity and reconciliation.

### Guided work

1. Define an `OrderPlaced` contract with owner, event ID, version, ordering key,
   semantics, privacy classification, and examples.
2. Add an outbox table and write events in the order transaction.
3. Build a bounded relay with claim leases and retry/dead-letter behavior.
4. Publish using event ID as the stable message identity.
5. Add a notification consumer with inbox dedupe in its state transaction.
6. Make provider calls idempotent where supported; otherwise record/reconcile the
   ambiguous side effect explicitly.
7. Define compatibility: required fields never change meaning; additions remain
   optional to older consumers; breaking semantics use a new version/type.
8. Add replay by bounded range without bypassing dedupe.
9. Expose outbox age, broker lag, redelivery, consumer failure, and dead letters.
10. Document retention for broker, outbox, inbox, and audit evidence.

### Failure checks

Crash before business commit, after commit before relay, before publish, after broker
acceptance, and before published marking. Redeliver the same event concurrently.
Replay an old range. Deploy a consumer that ignores a newly added optional field.
Interrupt the broker and observe bounded queue growth and recovery.

### Knowledge check and expected answers

1. **What gap does the outbox close?** Business state and publication intent commit
   atomically in one local database.
2. **Why can the relay still duplicate?** Broker acceptance and outbox acknowledgement
   are not one transaction.
3. **Where must inbox dedupe commit?** In the same transaction as the consumer state
   effect it protects.

### Completion evidence

Crash-point tests lose no committed intent and produce no unsafe duplicate effect;
range replay converges to the same state; producer/consumer deployment order works
for a compatible change; contracts state ordering and ownership; and lag,
redelivery, oldest age, and dead letters are actionable.

## Phase 13: Distributed-Systems Mechanics Lab

### Goal and mental model

Build deterministic experiments that make partial failure visible before production
services depend on it. The goal is understanding and prediction, not writing a
custom consensus implementation.

Wall clocks can jump because of synchronization or manual change. Use monotonic time
for elapsed durations inside one process. Across nodes, timestamps do not establish
a perfect order; use explicit sequence, version, or causality information when order
matters.

Networks can delay, drop, duplicate, reorder, or deliver asymmetrically. A timeout
means the caller stopped waiting, not that the remote side did nothing. A partition
can leave both sides healthy but unable to communicate.

A lease grants authority until an expiry assumption, but a paused old owner can wake
after another owner takes over. A monotonically increasing fencing token travels
with writes; the protected resource rejects tokens older than the highest accepted.

Quorum behavior depends on replica count and required acknowledgements. With `N`
replicas, read quorum `R`, and write quorum `W`, overlap when `R + W > N` can help a
read observe a successful write under stated assumptions, but conflict resolution,
failures, and sloppy quorums still need explicit design.

Consistent hashing reduces key movement as nodes change. Virtual nodes improve
distribution. It does not replicate data, resolve concurrent writes, or guarantee
availability.

Raft uses leader election, replicated logs, majority commitment, and term/index rules
to preserve safety. Learn to explain it, but delegate consensus to maintained
databases, brokers, orchestrators, or coordination systems.

### Build a deterministic simulator

Represent nodes, messages, and a logical event queue. Tests explicitly advance
logical time and choose delivery actions:

```text
send -> queued
queued -> delivered | dropped | duplicated | delayed
delayed messages may be delivered in chosen order
```

Use seeded or enumerated schedules and print the schedule on failure. Never depend on
real sleeps or scheduler luck.

### Guided experiments

1. Predict then simulate delayed, dropped, duplicated, and reordered messages.
2. Create an asymmetric partition where A reaches B but B cannot reach A.
3. Demonstrate a stale lease holder writing after a new owner.
4. Add fencing tokens and prove the stale write is rejected.
5. Run quorum reads/writes during each partition shape; record stale and unavailable
   outcomes.
6. Define conflict/reconciliation rules before healing the partition.
7. Measure key movement and balance for modulo hashing versus consistent hashing with
   virtual nodes.
8. Walk through Raft election, log replication, majority commit, leader failure, and
   a follower with conflicting uncommitted entries.
9. Map each experiment to the service's jobs, outbox, and future extraction risks.
10. Record which production component owns consensus and why it is trusted.

### Failure checks

Pause the lease owner beyond expiry and resume it. Lose quorum, then restore it.
Deliver an old leader's message after a new term. Heal replicas with conflicting
values using the defined reconciliation rule. Add/remove hashing nodes and compare
movement plus maximum imbalance.

### Knowledge check and expected answers

1. **Why does timeout not prove failure?** The operation may have completed after the
   response was delayed or lost.
2. **What does fencing add beyond a lease?** The protected resource rejects authority
   from an older owner even if that process resumes.
3. **What does consistent hashing not solve?** Replication, correctness, conflict
   resolution, and service availability.
4. **Why not build production consensus here?** Safety under all failure schedules is
   specialized, deeply tested work already provided by maintained systems.

### Completion evidence

Tests reproduce split-brain, stale-owner, clock-skew, quorum-loss, and recovery cases
without sleeps; predictions are recorded before execution; fencing blocks old owners;
partition healing has explicit reconciliation; hashing measurements are explained;
and the production design delegates consensus to a maintained component.
