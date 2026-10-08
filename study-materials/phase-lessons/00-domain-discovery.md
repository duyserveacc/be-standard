# Phase 0: Domain Discovery and Modeling

This lesson expands
[Phase 0 of the roadmap](../../BACKEND_LEARNING_ROADMAP.md#phase-0-domain-discovery-and-modeling).
Complete it before choosing endpoints, tables, packages, or services. The output is
a shared model of behavior, not application code.

## Learning outcomes

You should be able to turn vague requirements into examples, define a shared
language, distinguish entities from value objects, identify invariants, draw a state
machine, and propose bounded contexts without treating them as deployments.

## 1. Treat requirements as hypotheses

Statements such as "customers can cancel orders" hide decisions:

- Which order states are cancelable?
- Does cancellation release inventory immediately?
- What happens if fulfillment already started?
- Can an administrator cancel on behalf of a customer?
- Is repeating cancellation successful, rejected, or idempotently acknowledged?
- Which timestamp decides a cutoff, and whose clock is authoritative?

Separate information into four categories:

| Category | Meaning | Example |
| --- | --- | --- |
| Confirmed fact | Reviewed requirement | A customer can view only their own orders. |
| Assumption | Working belief needing review | Cancellation is allowed before fulfillment. |
| Open question | Missing decision | Does cancellation restore stock synchronously? |
| Deferred scope | Explicitly excluded for now | Partial shipment and partial cancellation. |

Never let an assumption silently become a permanent API or schema contract.

## 2. Build ubiquitous language

Use one meaning for each important term across product discussion, code, APIs,
events, dashboards, and runbooks.

An initial glossary should define:

- **Product:** an item offered in the catalog; it does not itself mean available
  inventory.
- **Inventory item:** the quantity tracked for one product in one inventory scope.
- **Reservation:** a durable claim reducing inventory available to other orders.
- **Order:** a customer's request to acquire specified quantities at captured prices.
- **Order item:** one product, quantity, and price captured within an order.
- **Cancellation:** a business transition that ends an eligible order and defines
  how reservations are released.
- **Fulfillment:** the process that delivers a confirmed order; it is initially out
  of scope but affects which transitions are safe.
- **Customer:** the authenticated subject that owns an order.

Record disputed terms instead of forcing artificial agreement. `Product` may mean
catalog description in one bounded context and a sellable stock-keeping unit in
another. The model should make that difference explicit.

## 3. Use examples to discover rules

Write examples in business language with concrete values:

```text
Given product TEA-001 has 3 units available
And customer C-17 requests 2 units
When the customer places order O-91
Then O-91 is placed with quantity 2
And TEA-001 has 1 unit available
And the captured item price does not change if the catalog price changes later
```

Then add rejected and concurrent examples:

```text
Given product TEA-001 has 1 unit available
When customers C-17 and C-22 concurrently request 1 unit
Then exactly one order is placed
And the other request reports insufficient inventory
And inventory never becomes negative
```

Cover success, invalid input, missing product, insufficient inventory, wrong owner,
duplicate retry, cancellation, dependency failure, and concurrent contention. Each
example should reveal a decision or protect an invariant.

## 4. Commands, events, entities, and value objects

A **command** asks the system to change state and may be rejected: `PlaceOrder`,
`CancelOrder`, `AdjustInventory`.

An **event** names a fact that already happened: `OrderPlaced`, `OrderCanceled`,
`InventoryAdjusted`. Use past tense. Events are immutable records, not instructions.

An **entity** has identity across change. Order O-91 remains the same order when its
status changes. A **value object** is interchangeable when all values match, such as
a quantity plus unit or a money amount plus currency.

Identity is not a reason to make every field mutable. Prefer explicit operations
that preserve invariants over public setters.

## 5. State machines expose missing decisions

An initial order state machine might be:

```text
PlaceOrder:     nonexistent -> placed
CancelOrder:    placed -> canceled

Terminal: canceled
Invalid: canceled -> placed
Invalid: canceled -> canceled unless cancellation is explicitly idempotent
```

For each transition record:

- initiating actor and command;
- preconditions;
- state changes;
- inventory effect;
- emitted event;
- repeat behavior;
- timeout or failure behavior;
- terminal-state status.

Do not draw a transition merely because it is technically possible. It must have a
business meaning and an owner.

## 6. Write invariants independently of technology

Initial invariants include:

1. Inventory available quantity is never negative.
2. An order belongs to exactly one customer.
3. A customer cannot observe another customer's order.
4. Every order item has positive quantity.
5. An order's total equals the sum of captured item prices multiplied by quantities.
6. Placing an order and reserving its inventory succeed or fail together.
7. One logical retry creates at most one order.
8. A terminal order cannot transition to an earlier state.

Write them without HTTP statuses, Go types, SQL, or broker terminology. Later layers
implement and enforce them; they do not define them.

## 7. Map bounded contexts and information flow

Candidate contexts are:

| Context | Owns | Does not own |
| --- | --- | --- |
| Catalog | Product description, SKU, current offered price | Customer orders |
| Inventory | Available quantity and reservations | Product marketing text |
| Ordering | Order lifecycle, captured items and totals | Identity verification |
| Identity | Verified subject and roles | Order state |
| Notification | Delivery attempts and preferences | Whether an order is valid |

Draw information flow: Ordering needs product/price and inventory decisions;
Identity supplies a verified principal; Notification reacts to completed facts.
These are reasoning boundaries. Keep them as modules until deployment separation is
justified by ownership, scaling, isolation, or release evidence.

## 8. Guided work

Create a domain-discovery document and complete these steps:

1. List actors and the outcomes each wants.
2. Write the glossary and flag overloaded terms.
3. Write at least two concrete examples per initial API use case.
4. Add concurrent, retry, wrong-owner, cancellation, and dependency-failure examples.
5. Extract named invariants from the examples.
6. Draw order and reservation state machines.
7. Draw the context map with ownership and information flow.
8. Record facts, assumptions, open questions, and deferred scope separately.
9. Review it with someone who did not write it and record changed interpretations.
10. Apply the microservice lens without proposing deployment infrastructure.

## 9. Failure checks

- Give two reviewers the phrase "cancel an order." If they describe different
  outcomes, the model remains ambiguous.
- Try to express every initial endpoint as examples. Any endpoint without an
  invariant or owner needs clarification.
- Remove all technical nouns. If the behavior can no longer be explained, the model
  was describing implementation rather than the domain.
- Ask what happens when the same command is repeated and when two valid commands
  race. Missing answers become explicit open questions.

## 10. Knowledge check

1. Why are requirements treated as hypotheses?
2. What is the difference between a command and an event?
3. Why is an order an entity while money is usually a value object?
4. What does a state machine reveal that a list of endpoints does not?
5. Why is a bounded context not automatically a microservice?

### Expected answers

1. Natural-language requirements omit edge cases and conflicting interpretations;
   concrete examples expose them before implementation hardens them.
2. A command requests change and may fail; an event records a completed fact.
3. An order retains identity while changing; equal money values are interchangeable
   when amount and currency match.
4. Legal, invalid, and terminal transitions plus their preconditions and effects.
5. It is a model and ownership boundary; network deployment adds independent
   failure, compatibility, security, telemetry, and operational costs.

## 11. Completion evidence

Phase 0 is complete when every initial API use case maps to reviewed examples and
named invariants; at least two conflicting interpretations are explicitly resolved;
state and context diagrams use only domain language; the decision log separates
facts from assumptions; and an outside reviewer can explain valid and invalid order
transitions without referring to HTTP or SQL.
