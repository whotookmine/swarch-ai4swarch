> Agent: Claude Code · Model: Claude Opus 5.5 (`claude-opus-5-5`) · Run: a (zero-shot, context: none)

# ADR-0002: Publish an "order placed" event and handle post-order actions in independent subscribers

- **Status:** Proposed
- **Date:** 2026-10-25
- **Deciders:** Not given in the requirement (see Open questions)

## Context and problem

No system files were read for this run (`context: none`). Everything about the current architecture below is an **[Assumption]** taken from the wording of the requirement, not evidence.

The requirement says that after an order is placed, Online Boutique must:

1. send the order confirmation email through **MailFast**, a new external email provider. This replaces the mock `emailservice` logic.
2. award loyalty points through a new **`loyaltyservice`** (still to be built).
3. record an **"order placed" analytics event** for the marketing team.

Marketing also expects **more post-order actions next year**: SMS, fraud scoring and a recommendations update.

Problem: **how should the service that places orders trigger a growing set of post-order actions?** The options trade off coupling, latency, failure isolation and operational cost.

Forces:

- The number of post-order actions goes from 1 (the mock email) to 3 now, and to about 6 next year. Each new action should not mean changing the order-placing code.
- MailFast is an external system. Its availability, latency and rate limits are outside our control, and they are not given.
- `loyaltyservice` is new, so it may be less stable at first.
- **[Assumption]** Placing the order (charging the customer) matters more than any post-order action. A failure in email, loyalty or analytics should not fail or slow down checkout.
- Some actions may need **ordering or a result** before others. For example, fraud scoring may have to run *before* fulfilment or email. This is not stated (see Open questions).

## Decision drivers

Most important first:

1. **Availability of checkout:** post-order actions must not make order placement fail. *(availability)*
2. **Modifiability / extensibility:** adding SMS, fraud scoring and a recommendations update should not require changes to the order-placing service. *(modifiability)*
3. **Reliability of each action:** no lost confirmation emails or loyalty points (at-least-once delivery, retries). *(reliability)*
4. **Checkout latency:** the customer should not wait for external calls such as MailFast. *(performance)*
5. **Isolation from external provider failures:** a MailFast outage or rate limit must be contained. *(availability, fault tolerance)*
6. **Operational cost and complexity:** keep new infrastructure and the skills it needs reasonable. *(cost, operability)*

## Considered options

1. **Option A: Synchronous fan-out from the order-placing service.** After the order is placed, the service calls the MailFast adapter (the former `emailservice`), `loyaltyservice` and an analytics endpoint directly, one after another or in parallel, using request/response calls.
2. **Option B: Publish an `OrderPlaced` event to a message broker (publish/subscribe).** The order-placing service publishes one event. Each post-order action is an independent subscriber with its own queue: the email adapter that calls MailFast, `loyaltyservice` and an analytics consumer. A transactional outbox (or equivalent) makes sure the event is published reliably.
3. **Option C: A dedicated post-order orchestrator service.** The order-placing service hands the order to one new `postorderservice` (async, through a queue). That service runs a workflow that calls email, loyalty and analytics in a defined order, with retries.

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| A: Sync fan-out | Simplest; no new infrastructure. Easy to trace one request end to end. Immediate success/failure feedback for each action. Fits the current style **[Assumption]**: direct service-to-service calls. | Checkout latency grows with every action, including the external MailFast call. Checkout availability becomes the *product* of all dependencies, so one failing action can fail or slow the order unless errors are swallowed. Every new action (SMS, fraud, recommendations) means changing and redeploying the order-placing service. Retries and dead-lettering have to be hand-written. |
| B: Event + pub/sub broker | Checkout only waits for "event stored/published", so it is decoupled from MailFast, `loyaltyservice` and analytics. New actions are added as new subscribers with **no change** to the publisher. Each consumer has its own retry policy, rate limit and dead-letter queue, which isolates MailFast failures. Analytics fits naturally, because the event *is* the analytics fact. Consumers scale independently. | New infrastructure (broker) to run, monitor and secure, plus the skills for it. Eventual consistency: the email and points arrive a little later, and the UI cannot show "email sent" synchronously. At-least-once delivery means consumers must be **idempotent** (no duplicate emails or double points). The event schema becomes a contract that needs versioning. End-to-end flow is harder to trace and debug. Actions that need a strict order or must block the order (fraud scoring?) do not fit pure pub/sub. |
| C: Orchestrator service | Checkout is decoupled through one hand-off. The workflow is explicit and visible in one place, so ordering, compensation and "fraud before email" are easy to express. Central retries and monitoring. | Each new action still means changing the orchestrator, so modifiability only moves from checkout to the orchestrator. The orchestrator is a new single point of failure and a bottleneck that every action depends on. It needs a queue or workflow engine anyway, so the infrastructure cost is close to B. Risk of a "god service" that knows every downstream team's API. |

## Decision

**Chosen: Option B. Publish an `OrderPlaced` event; each post-order action is an independent subscriber.**

- The order-placing service publishes `OrderPlaced` (order id, user id, items, totals, email address, timestamp, schema version) after the order is confirmed. It uses a transactional outbox, or equivalent, so an order is never confirmed without its event.
- Subscribers at launch:
  - **email-notifier**: replaces the mock `emailservice` logic and calls MailFast. It owns MailFast credentials, rate limiting, retries and its dead-letter queue.
  - **`loyaltyservice`**: awards points idempotently, keyed by order id.
  - **analytics consumer**: forwards or stores the event for marketing.
- Next year's actions (SMS, recommendations update) are added as new subscribers with no change to checkout.
- **Fraud scoring is out of scope for this decision.** If it has to *block* or *gate* an order, it belongs before or inside order placement (a synchronous check or a separate ADR), not in fire-and-forget pub/sub.

Why: B best meets the top two drivers. Checkout availability and latency become independent of MailFast and the new services, and the expected stream of new actions becomes additive. B's main costs (broker operations, idempotency, eventual consistency) are acceptable for notification-, reward- and analytics-type actions, which can tolerate a delay of seconds. A fails drivers 1, 2 and 4 as actions grow. C solves latency but not extensibility, and adds a central point of failure.

## Consequences

- **Positive:** Checkout no longer waits on MailFast or `loyaltyservice`. A provider outage delays emails but does not fail orders. *(quality attribute: availability, performance)*
- **Positive:** New post-order actions are new subscribers. The publisher and existing consumers stay unchanged, and teams can deploy independently. *(quality attribute: modifiability, deployability)*
- **Positive:** Each consumer scales and retries on its own, and MailFast rate limits are handled inside the email-notifier only. *(quality attribute: scalability, fault tolerance)*
- **Positive:** The analytics requirement is met by the same event, so there is no separate tracking call. *(quality attribute: modifiability)*
- **Negative:** A message broker (and outbox) to provision, operate, monitor and back up. *(quality attribute: cost, operability)*
- **Negative:** Eventual consistency. Confirmation email and loyalty points arrive with a delay, and checkout cannot report their success. *(quality attribute: usability / consistency)*
- **Negative:** Harder to trace one order across asynchronous consumers. Needs correlation ids and distributed tracing. *(quality attribute: observability, testability)*
- **Negative:** The `OrderPlaced` schema is a shared contract. Changing it needs versioning and coordination between consumers. *(quality attribute: modifiability)*
- **Risks and mitigations:**
  - *Duplicate delivery → double points or duplicate emails.* Make consumers idempotent, keyed by order id; MailFast idempotency keys if supported (unknown).
  - *Lost event if the publish fails after the order commits.* Transactional outbox, or publish-then-confirm with reconciliation.
  - *Poison messages or a long MailFast outage.* Per-consumer retry with backoff, a dead-letter queue, alerting, and a replay tool.
  - *Broker becomes a single point of failure.* Run it highly available; with an outbox, checkout keeps working while the broker is down and events drain later.
  - *Personal data (email address) in events reaches analytics.* Minimise the payload, or give analytics a filtered view; confirm the privacy rules (Open question).
  - *Fraud scoring wrongly modelled as fire-and-forget.* Decide its blocking semantics before adding it (separate ADR).

## Validation

- **Load / latency test:** compare checkout latency before and after. Target: no significant increase while MailFast is slowed or unavailable. The exact target number is not given.
- **Chaos test:** stop MailFast (or a stub), `loyaltyservice` and the broker one at a time. Orders must still succeed, and actions must complete after recovery with no loss and no duplicates.
- **Metrics:** event publish lag, consumer lag for each subscriber, dead-letter queue depth, MailFast error rate, time from order to email sent, duplicate-award count (should be 0).
- **Extensibility check:** when the first new action (e.g. SMS) is added next year, confirm that it needed **no change** to the order-placing service.
- **Review date:** not given. Proposed: review after the first new post-order action ships (see Open questions).

## Open questions

Not given in the requirement and not invented here:

1. **Current architecture:** how orders are placed today, which service calls `emailservice`, and over what protocol. No system files were in context.
2. **Deciders** and owning teams (checkout, loyalty, marketing).
3. **Latency and availability targets** for checkout and for each post-order action (e.g. max time from order to email, loyalty points visible within X).
4. **MailFast details:** API type, SLA, rate limits, idempotency-key support, cost per message, data-residency terms.
5. **Analytics destination:** what marketing uses (warehouse, third-party tool), the required event fields, and retention.
6. **Fraud scoring semantics:** must it block or cancel the order, or is it after-the-fact? This changes whether it can be a subscriber.
7. **Ordering or dependencies** between actions (e.g. email only after fraud check; loyalty only after payment settles).
8. **Order cancellation / refunds:** must loyalty points be revoked? Is an `OrderCancelled` event needed?
9. **Broker technology and team skills:** which broker the team can operate (none chosen here), the hosting constraints, and the budget.
10. **Privacy and compliance** rules for putting customer email and order data in events consumed by marketing.
11. **Expected order volume** (orders per second, peaks), to size the broker and consumers.
