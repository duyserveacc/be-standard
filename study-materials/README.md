# Study Materials

These materials complement the main [backend learning roadmap](../BACKEND_LEARNING_ROADMAP.md). The roadmap remains the source of truth for sequence, deliverables, and acceptance criteria. Detailed lessons teach the material without requiring an external tutorial; refresher guides support later recall.

## Detailed lessons

- [Day 1: Go backend foundations](day-01-go-backend-foundations.md) - a self-contained lesson on Go modules, types, ownership, interfaces, errors, context, concurrency, testing, and the in-memory product-store exercise.
- [Day 2: HTTP contracts and request lifecycle](day-02-http-contracts-and-lifecycle.md) - HTTP semantics, strict request handling, middleware, public errors, handler tests, and graceful shutdown.
- [Day 3: PostgreSQL and relational data](day-03-postgresql-and-relational-data.md) - relational constraints, migrations, pools, parameterized SQL, transactions, concurrency, integration tests, and query plans.
- [Day 4: Business logic and architecture](day-04-business-logic-and-architecture.md) - vertical features, representation boundaries, invariants, narrow interfaces, dependency injection, transaction ownership, and error mapping.
- [Day 5: Identity and API security](day-05-identity-and-api-security.md) - OAuth/OIDC roles, token and JWKS validation, principals, role and object authorization, browser security, redaction, and threat modeling.
- [Day 6: Testing, observability, and operations](day-06-testing-observability-and-operations.md) - test boundaries, deterministic fixtures, logs, metrics, traces, service objectives, limits, retries, failure drills, alerts, and runbooks.
- [Day 7: Delivery and production rehearsal](day-07-delivery-and-production-rehearsal.md) - containers, runtime configuration, CI, compatible migrations, rollback, restore testing, capacity, incident rehearsal, and clean-clone verification.

External links in the roadmap are optional primary references for deeper study. They are not prerequisites for completing a detailed lesson.

## Detailed implementation phases

- [Phases 0-26 lesson index](phase-lessons/README.md) - self-contained lessons covering domain discovery, one production service, durable work, distributed systems, service extraction, scaling, security, cloud/platform operations, integrations, maintenance, and the capstone.

## Refresher guides

- [Backend phase key concepts](backend-phase-key-concepts.md) - concise recall notes for implementation Phases 0-26, including mental models, failure modes, and self-check questions.

## Suggested refresher method

1. Read the phase's **Core ideas** before starting its work.
2. Explain the **Memory anchors** without looking at the notes.
3. Answer the **Self-check** questions in your own words.
4. Apply the roadmap's **cross-cutting microservice lens** to the phase, even while its implementation remains in-process.
5. Return to the roadmap for the phase's deliverables and acceptance criteria.
