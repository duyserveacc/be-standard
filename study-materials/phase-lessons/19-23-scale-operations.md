# Phases 19-23: Scale, Security, Cloud, and Operations

These phases address growth and production operation without treating scale as a
reason to weaken correctness, tenant isolation, or ownership.

## Phase 19: Data Scaling, Caching, and Multi-Tenancy

### Goal and mental model

Scale each bottleneck from measurement. Caches, replicas, partitions, archives, and
tenant models solve different problems and introduce different consistency or
operational costs.

Cache-aside reads check cache, load authoritative data on miss, then populate with a
bounded TTL. Define key/version, maximum staleness, invalidation after writes,
behavior during cache outage, stampede protection, size/eviction, and telemetry.
Cache failure must increase load or latency, not corrupt authoritative state.

Read replicas reduce primary read load but replicate asynchronously. After a write,
a replica may be stale. Choose per-operation behavior: read from primary for a
bounded read-your-write window, expose version/freshness, or accept documented
staleness. Never send correctness-critical inventory checks to a stale replica.

Partitioning splits a large table/index by a key to improve management or access
patterns; it is not automatically faster. Archival moves cold data with retention
and retrieval rules. Test pruning, cross-partition queries, unique constraints,
migrations, backup, and deletion before adoption.

Multi-tenancy is an isolation model across authentication, authorization, rows,
jobs, events, caches, telemetry, quotas, backups, exports, repairs, and operator
tools. Tenant identity comes from trusted identity mapping, never a client-controlled
body field.

For shared tables, include `tenant_id` in every tenant-owned primary/access path and
in unique and foreign-key constraints where ownership must match. If PostgreSQL row
security uses a tenant setting, set it transaction-locally for every checked-out
connection and default-deny when absent; never rely on session state that can leak
through a pool to the next tenant. Privileged maintenance roles and bypass behavior
need separate authorization and tests.

Compare:

| Model | Strength | Cost/risk |
| --- | --- | --- |
| Shared tables with tenant key | Efficient and simple fleet | Every access must fail closed on tenant scope |
| Schema per tenant | More namespace separation | Migration and connection complexity |
| Database per tenant | Stronger isolation/restore flexibility | Highest fleet and cost overhead |

Choose from security, compliance, scale, restore, and operating constraints.

### Guided work

1. Measure a product-read bottleneck before adding cache.
2. Implement cache-aside with versioned keys, TTL, invalidation, bypass, and bounded
   single-flight/stampede protection.
3. Test cache outage, eviction, stale value, and concurrent miss behavior.
4. Add a read replica and define which reads may tolerate lag.
5. Measure lag and demonstrate read-after-write policy.
6. Run partition/archive experiments at representative volume and inspect plans.
7. Select a tenant model through a documented tradeoff analysis.
8. Propagate tenant identity through API, saga, job, event, cache, and audit paths.
9. Add tenant-scoped keys/constraints and database scoping or row security, including
   connection-reuse and privileged-role negative tests.
10. Enforce per-tenant rate, concurrency, storage, and background-work quotas.

### Failure checks

Flush or disconnect cache, delay replication, move a tenant ID between message and
payload, poison a cache key, replay a job under another tenant, and run operator
repair/export with ambiguous scope. Reuse the same pooled database connection across
two tenant transactions and attempt a cross-tenant foreign key. Drive one tenant far
above quota and verify other tenants retain service.

### Knowledge check and expected answers

1. **Why is cache-aside not source of truth?** Entries can expire, evict, or become
   stale; authoritative durable state and invariants remain elsewhere.
2. **Why can a replica violate read-your-write?** Asynchronous apply may lag behind
   the acknowledged primary commit.
3. **Why is a tenant filter convention insufficient?** One forgotten path can expose
   data; isolation must fail closed across every synchronous and asynchronous path.

### Completion evidence

Caching has before/after evidence and safe outage behavior; replica staleness is
measured and routed by operation; partition/archive changes have plans and rollback;
cross-tenant tests cover APIs, jobs, events, cache, and operator tools; and noisy-
neighbor quotas protect other tenants.

## Phase 20: Security, Privacy, and Software Supply Chain

### Goal and mental model

Every new service, queue, datastore, administrative command, and CI identity adds a
trust boundary. Update threat models and data flows as architecture changes.

User identity represents the end actor and should propagate only as required for
authorization/audit. Workload identity authenticates one service or job to another.
Do not reuse a user's bearer token as a general service credential. Prefer short-
lived workload credentials and least-privilege audience/resource permissions.

TLS protects transport; mutual authentication or signed workload identity proves
the peer. Encryption does not authorize an action by itself. Broker topics, database
roles, object prefixes, secret access, and operator endpoints each need explicit
least privilege.

Secret rotation needs overlap: publish new credential/certificate, let workloads
reload or roll, verify new use, revoke old after the window, and confirm old access
fails. Never make rotation depend on a simultaneous fleet restart.

Privacy starts with an inventory:

```text
field -> purpose -> legal/business owner -> locations -> access -> retention
      -> export behavior -> deletion behavior -> backup/derived-data policy
```

Deletion is a workflow across primary data, caches, search, events, analytics, and
provider systems. Immutable audit or backup retention may require documented policy
rather than immediate physical removal; access and eventual expiry must be explicit.

Supply-chain controls connect source to artifact: locked dependencies, reviewed
updates, reproducible build inputs, SBOM, provenance, signing, verification before
deployment, vulnerability inventory, and a normal patched release path.

### Guided work

1. Update system data-flow and trust-boundary diagrams.
2. Threat-model impersonation, lateral movement, queue forgery, data exfiltration,
   operator abuse, and build compromise.
3. Create per-workload identities and resource-specific permissions.
4. Enforce transport authentication and authorization service-to-service.
5. Automate secret/certificate rotation with overlap and revocation verification.
6. Build the sensitive-data inventory and minimize unnecessary fields.
7. Implement export and deletion as durable, idempotent, auditable workflows.
8. Include caches, messages, backups policy, derived stores, and providers.
9. Generate SBOM/provenance, sign artifacts, and verify before deployment.
10. Run a vulnerability drill from advisory to affected artifact, patch, rebuild,
   verification, rollout, and residual-risk closure.

### Failure checks

Use a stolen credential against unrelated resources, expire old credentials after
rotation, replay a service token to the wrong audience, delete a user with cached and
queued data, and identify every artifact containing a vulnerable dependency. Block
deployment of an unsigned or provenance-mismatched artifact.

### Knowledge check and expected answers

1. **Why separate user and workload identity?** They represent different principals,
   lifetimes, audiences, and authorization responsibilities.
2. **Why is deletion a workflow?** Data exists in multiple authoritative, cached,
   derived, queued, provider, and backup locations with different guarantees.
3. **What does an SBOM provide?** An inventory linking components to artifacts so
   exposure can be identified; it does not itself fix vulnerabilities.

### Completion evidence

Compromised credentials have tested bounded reach; rotation completes without outage
and revokes old access; every sensitive field has purpose/owner/retention/export/
deletion treatment; deletion/export workflows reconcile all locations; signed
artifacts are verified; and the vulnerability drill completes through normal release.

## Phase 21: Cloud Infrastructure and IAM

### Goal and mental model

Provision a reproducible non-production environment while making administrative,
network, and identity boundaries explicit. Managed services reduce infrastructure
operation but do not own application schemas, retry safety, authorization, or data
recovery verification.

Accounts/projects limit administration and billing. Regions and availability zones
define failure domains. Select them from latency, data residency, service availability,
recovery, and cost requirements.

Network flow should be explainable hop by hop:

```text
Internet -> managed DNS/TLS/load balancer -> private workload
workload -> private database/queue/secret endpoint
workload -> controlled egress/NAT only where required
```

Subnets, route tables, firewalls/security groups, network ACLs where applicable,
private endpoints, and DNS form the connectivity policy. A private IP alone is not
authorization.

Humans use federated identity and audited roles. Workloads use short-lived attached
identity, not static access keys. Encryption keys, secret access, and deployment
roles have separate permissions.

Infrastructure as code needs protected remote state, locking, review, plan output,
policy checks, drift detection, ownership tags, budgets, audit logs, and safe scoped
teardown. Console-only changes become undocumented drift.

### Guided work

1. Choose account/project, region, and availability-zone strategy in an ADR.
2. Define network, private/public subnets, routes, ingress, and controlled egress as
   code.
3. Provision managed DNS, certificate, TLS load balancer, and health routing.
4. Place workloads privately and keep database/operator endpoints non-public.
5. Attach workload identities with only database, queue, object, and secret actions
   each process needs.
6. Encrypt data using managed keys with explicit administrators and users.
7. Enable access, audit, network-flow, and service logs with retention.
8. Protect IaC state and add validation, policy, review, and drift detection.
9. Add ownership/cost tags and budget alerts.
10. Recreate and tear down the isolated environment using reviewed code only.

### Failure checks

Deny DNS, route, security-group, secret, key, database, and queue permissions one at a
time. Verify diagnostics distinguish network from IAM from application errors.
Attempt public database access. Rotate workload identity. Run teardown preview and
prove it cannot select shared or production resources.

### Knowledge check and expected answers

1. **Why use availability zones?** They provide separate failure domains inside a
   region, though application/data architecture must still use them correctly.
2. **Why prefer workload identity to static keys?** Credentials are short-lived,
   scoped, attached to runtime identity, and easier to rotate/audit.
3. **Why protect IaC state?** It can contain resource topology and sensitive values
   and controls changes to real infrastructure.

### Completion evidence

Version-controlled code recreates the environment; only the edge is public;
workloads have tested least privilege; identity rotation is outage-free; network/IAM
denials are diagnosable; audit and budgets operate; cost is estimated; and scoped
teardown removes only intended resources.

## Phase 22: Orchestration and Platform Operations

### Goal and mental model

An orchestrator reconciles declared workload state. It cannot make an application
start safely, report readiness truthfully, drain requests, or use compatible schemas;
those remain application contracts.

Core Kubernetes-style objects serve different roles:

- workload controller maintains replicas and rollout state;
- Service gives stable discovery over changing instances;
- ingress/gateway routes edge traffic;
- ConfigMap and secret references supply runtime values;
- network policy restricts allowed communication;
- persistent volumes preserve state only for workloads designed to use them.

Requests influence scheduling; limits bound consumption. Too-low CPU limits can
throttle latency, and memory limit violation terminates the process. Autoscaling must
use a meaningful saturation/demand signal and respect database/broker capacity.

Readiness removes an instance from traffic, liveness restarts a stuck process, and a
startup probe protects slow initialization. Misconfigured probes can create restart
loops. Disruption budgets constrain voluntary removal but do not guarantee capacity
during involuntary failure.

Controlled rollout settings, pre-stop/drain timing, schema compatibility, and
rollback criteria work together. Platform templates should expose critical limits
and failure behavior rather than hide them behind opaque abstraction.

### Guided work

1. Deploy to a local or isolated non-production Kubernetes environment.
2. Add small reviewed manifests/templates for workloads, Services, ingress,
   configuration, secret references, and network policies.
3. Set resource requests/limits from Phase 9 evidence.
4. Configure startup, readiness, liveness, and termination grace coherently.
5. Add rollout surge/unavailable limits and a disruption budget.
6. Configure autoscaling from relevant saturation while capping fleet database use.
7. Validate manifests/IaC in CI and document bootstrap/teardown.
8. Test failed rollout, bad config, unschedulable pod, node loss, and capacity
   exhaustion.
9. Provide application-team diagnostics that do not require cluster-admin access.
10. Write rollback and recovery runbooks for each tested platform symptom.

### Failure checks

Use a bad readiness path, impossible image, insufficient resource request, denied
network policy, node drain, and slow termination. Scale replicas until the database
budget would be exceeded and confirm caps prevent it. Roll out old/new schema-compatible
versions and abort midway.

### Knowledge check and expected answers

1. **Why are requests and limits different?** Requests guide scheduling/reservation;
   limits cap runtime consumption and can throttle or terminate work.
2. **Why can autoscaling worsen an outage?** More replicas can multiply connections,
   retries, and pressure on the actual bottleneck.
3. **What must the app own despite orchestration?** Valid startup, truthful health,
   context cancellation, graceful drain, and compatible state changes.

### Completion evidence

Rolling deployment preserves availability; failed readiness, node loss, and
termination match predictions; network policy enforces ownership; autoscaling uses a
relevant signal without exceeding dependency budgets; bootstrap/teardown are
reproducible; and application teams can diagnose common failures with limited access.

## Phase 23: Reliability Engineering, Incidents, and Cost

### Goal and mental model

Reliability is a user outcome with an explicit objective, not maximum uptime at any
price. SLIs measure behavior, SLOs set a target over a window, and error budgets make
the permitted unreliability visible for delivery decisions.

Define separate SLIs for order placement and retrieval using valid requests at the
user boundary. Specify latency threshold, successful outcome, exclusions, window,
data source, and treatment of missing telemetry. Avoid measuring only internal
component health.

Capacity planning links expected traffic shape, per-request cost, saturation, burst,
headroom, dependency quotas, and failure mode. State the first expected bottleneck
and test it. Include regional/AZ loss or failover capacity where required.

Incident response needs roles: commander coordinates, operations mitigate,
communications updates stakeholders, and recorder preserves timeline. Diagnosis
follows evidence; mitigation prioritizes user/data safety over root-cause certainty.

A blameless review explains impact, detection, timeline, contributing technical and
organizational conditions, what helped, and corrective actions. Actions need owner,
priority, due/trigger, verification, and closure evidence.

Cost includes compute, database, storage, network, observability, licenses, engineering
time, cognitive load, and on-call burden. Optimize cost per useful outcome without
removing required safety margin.

### Guided work

1. Define placement/retrieval SLIs, SLOs, and error-budget policy.
2. Build dashboards from the exact SLI queries.
3. Create multi-window alerts for fast/slow budget burn and saturation risk.
4. Model normal, peak, and one-failure-domain capacity with dependency limits.
5. Run a representative load test and identify the first bottleneck.
6. Create on-call roles, escalation, communication templates, and symptom runbooks.
7. Run a game day with partial dependency/region failure and backlog growth.
8. Measure detection, acknowledgement, diagnosis, mitigation, recovery, and
   verification times.
9. Write a blameless review and track corrective actions to evidence-based closure.
10. Compare cost/performance/reliability before and after proposed optimization.

### Failure checks

Remove one failure domain, slow a critical dependency, grow queue age, and exhaust a
quota. Verify alerts page on user impact rather than noise, responders can operate
from runbooks, status communication is timely, and recovery includes data convergence.
Challenge a cost cut that removes headroom and quantify its SLO risk.

### Knowledge check and expected answers

1. **What is an error budget?** The allowed unsuccessful fraction implied by the SLO,
   used to balance reliability work and delivery risk.
2. **Why start runbooks from symptoms?** Responders observe user impact before they
   know which component caused it.
3. **Why include engineering/on-call time in cost?** Distributed complexity consumes
   people and opportunity, not only cloud resources.

### Completion evidence

SLIs reflect user outcomes; alerts are actionable and budget-based; capacity model
matches load evidence and identifies the bottleneck; the game day measures the full
response timeline; corrective actions have owners and verification; and cost choices
state their reliability and operational tradeoffs.
