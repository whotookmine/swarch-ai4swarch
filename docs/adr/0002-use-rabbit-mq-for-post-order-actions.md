> Agent: Claude Code · Model: Claude Opus 5.5 (`claude-opus-5-5`) · Run: b (with context)

# ADR-0002: Publish an "order placed" event to RabbitMQ for post-order actions

- **Status:** Proposed
- **Date:** 2026-10-25
- **Deciders:** Online Boutique architecture team (names not given; see Open questions)

## Context and problem

Requirement R-12 (`docs/scenario.md`) adds three post-order actions: send the confirmation email through the external provider **MailFast** (replacing the mock `emailservice` logic), award **loyalty points** through a new `loyaltyservice`, and record an **"order placed" analytics event**. Marketing expects **more post-order actions next year** (SMS, fraud scoring, recommendations update).

How it works today (`system/online-boutique/src/checkoutservice/main.go`):

- `PlaceOrder` (line 230) runs one sequential, synchronous chain over gRPC: cart → product catalog / currency → shipping quote → `chargeCard` (line 252) → `shipOrder` (line 258) → `emptyUserCart` (line 263) → `sendOrderConfirmation` (line 273).
- An email failure is only logged as a warning (line 274), and the order still succeeds.
- `sendOrderConfirmation` (line 379) uses the caller's `ctx` with no timeout of its own, so a slow email call delays the checkout response directly.
- Each downstream service is wired in through an `*_SERVICE_ADDR` env var plus a gRPC connection (lines 111–123). Every new post-order action would mean another address, another connection and another call in `PlaceOrder`.
- `checkoutservice` has no database. The order exists only in memory and in the response (`orderResult`, line 265). The order ID comes from `uuid.NewUUID()` (line 233).

Facts that shape the decision (`docs/scenario.md`):

| Topic | Fact | Consequence for the design |
|---|---|---|
| Checkout latency | p95 `PlaceOrder` ≤ **500 ms**; today approximately **300 ms** | About 200 ms of headroom for any new work done inline |
| Peak load | **200 orders/s** (11.11 sale) | Every post-order path must absorb 200 events/s |
| MailFast | p95 **1.5 s**, p99 **8 s**, approximately **0.5 %** failures, **50 emails/s** limit | Calling it inline breaks the 500 ms goal by itself. At peak, demand is 4× the rate limit, so roughly **150 emails/s pile up** and need a buffer plus throttling |
| Loyalty | within **1 min**, **exactly once** | Needs durable delivery, retries and deduplication, and must not wait behind the email backlog |
| Availability | an order must **never fail** because email, loyalty or analytics is down | Post-order actions must be decoupled from the order's success |
| Team | 4 devs, strong in Go and gRPC, **2 have used RabbitMQ** | Prefer technology the team already knows |
| Operations | Kubernetes; prefer **open-source, self-hosted** | Rules out cloud-only managed brokers (e.g. a hosted pub/sub) |

The problem: **how should `checkoutservice` trigger post-order actions** so checkout stays fast and available, MailFast's limits are respected, loyalty is awarded exactly once, and new actions can be added cheaply?

## Decision drivers

1. **Availability:** an order must never fail because email, loyalty or analytics is down.
2. **Performance:** p95 `PlaceOrder` stays ≤ 500 ms at 200 orders/s, despite MailFast's 8 s p99.
3. **Reliability / correctness:** loyalty points are awarded within 1 minute and exactly once. Emails are not lost when MailFast fails (0.5 %) or throttles (50/s).
4. **Modifiability:** adding SMS, fraud scoring or a recommendations update next year should not require changing `PlaceOrder`.
5. **Team fit and operations:** the team of 4 should be able to build and run it: Go/gRPC skills, some RabbitMQ experience, self-hosted on Kubernetes.

## Considered options

1. **Extend the synchronous gRPC chain.** `PlaceOrder` calls the MailFast adapter (`emailservice`), `loyaltyservice` and an analytics service in sequence, each with a timeout.
2. **In-process async calls.** `PlaceOrder` returns, then goroutines inside `checkoutservice` make the three calls, retrying in memory.
3. **Publish an `OrderPlaced` event to RabbitMQ (self-hosted).** A fanout/topic exchange with one durable queue per consumer (`emailservice`, `loyaltyservice`, analytics).
4. **Publish an `OrderPlaced` event to Apache Kafka (self-hosted, e.g. via an operator).** One topic, one consumer group per action.

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| 1. Sync gRPC chain | Smallest change and matches today's code style; the team knows gRPC well; easy to trace | MailFast p95 1.5 s / p99 8 s alone breaks the 500 ms goal, and a short timeout drops emails instead; nothing absorbs the 150/s overflow above MailFast's 50/s limit; each new action adds latency and another dependency to `PlaceOrder` (lines 111–123 keep growing); no durable retry for exactly-once loyalty |
| 2. In-process async goroutines | No new infrastructure; checkout latency stays near today's 300 ms | Work is lost if a `checkoutservice` pod restarts or is rescheduled, so loyalty "exactly once" and "within 1 min" can't be guaranteed; rate limiting is per replica, so the global 50 emails/s is hard to enforce; `checkoutservice` still has to know every consumer (poor modifiability) |
| 3. RabbitMQ event | Checkout only waits for an in-cluster publish confirm, not for MailFast; durable per-consumer queues buffer the email backlog, so loyalty isn't delayed behind email; per-message ack/retry and dead-letter queues; new actions = new queue bound to the exchange, with no change to `checkoutservice`; open-source, runs on Kubernetes; 2 of 4 devs already know it | New stateful component to operate (clustering, quorum queues, monitoring); delivery is at-least-once, so `loyaltyservice` must deduplicate; checkout now depends on the broker being reachable at publish time; no built-in replay of past events for a consumer added later |
| 4. Kafka event | High throughput; retained log lets new consumers (e.g. next year's recommendations update) replay history; strong ordering per partition | Nobody on the team has used it (not given as a skill); heavier to run self-hosted for 200 events/s; still at-least-once by default, so loyalty still needs dedup; per-message rate-limited retries to MailFast are more awkward than with per-message acks |

## Decision

Choose **option 3: publish an `OrderPlaced` event to a self-hosted RabbitMQ cluster on Kubernetes.**

- **Producer:** after `shipOrder` succeeds and the cart is emptied, `PlaceOrder` publishes one `OrderPlaced` message (order ID from line 233, user email, items, totals) to an `orders` exchange with **publisher confirms**, under a short timeout. This replaces the direct `sendOrderConfirmation` gRPC call (lines 273–277). If the publish fails, checkout logs it and still returns success, as it does today for email (driver 1). How the event is recovered after that is an open risk, covered below.
- **Consumers,** each with its own durable (quorum) queue and dead-letter queue:
  - `emailservice` → **MailFast**, with a single global throttle of **≤ 50 emails/s** and retry with backoff for the approximately 0.5 % failures.
  - `loyaltyservice` → awards points **idempotently**: it records the order ID under a uniqueness constraint and acks only after the write. A redelivered message changes nothing. At-least-once delivery plus dedup gives the required "exactly once" effect.
  - analytics consumer → records the "order placed" event.
- New actions (SMS, fraud scoring, recommendations) bind new queues to the same exchange.

Why: this is the only option that meets drivers 1–3 together. MailFast's latency and rate limit are moved off the checkout path, and the email backlog sits in its own queue instead of delaying loyalty. It also meets driver 4 (new consumers without touching `PlaceOrder`) and driver 5 (open-source, self-hosted, and known to half the team). Kafka's main advantage, replay, isn't required by R-12, and it costs more operationally for a team without Kafka experience.

## Consequences

- **Positive:** `PlaceOrder` no longer waits on MailFast (1.5 s p95 / 8 s p99). It only waits for an in-cluster publish, so the ≤ 500 ms p95 target becomes achievable **[Inferred: publish latency is not measured in the provided files]** (quality attribute: performance).
- **Positive:** email, loyalty or analytics being down only builds up a queue. Orders keep succeeding (quality attribute: availability).
- **Positive:** the 11.11 email surge (200/s vs. 50/s) is buffered and drained at MailFast's limit instead of being rejected (quality attributes: reliability, scalability).
- **Positive:** new post-order actions are added as new subscribers. `checkoutservice` and its env/connection wiring (lines 111–123) stay unchanged (quality attribute: modifiability).
- **Negative:** confirmation emails arrive late during the peak. Backlog grows by about 150 emails per second of peak traffic (quality attribute: performance / user experience).
- **Negative:** RabbitMQ is a new stateful cluster to run, upgrade and monitor (quality attributes: cost, operability).
- **Negative:** at-least-once delivery means every consumer must be idempotent, and `loyaltyservice` needs its own durable store for processed order IDs (quality attributes: modifiability, complexity).
- **Risks and mitigations:**
  - *Lost event on publish failure (dual-write).* Payment has already been charged (line 252) when the publish happens. If the broker is unreachable, the order succeeds but loyalty, email and analytics never happen. `checkoutservice` has no database, so a transactional outbox is not possible today. Mitigations: run RabbitMQ as a 3-node quorum-queue cluster; retry the publish briefly within the latency budget; for a stronger guarantee, add a small outbox store to `checkoutservice` (a separate decision, see Open questions).
  - *Loyalty misses the 1-minute target.* Mitigation: alert on `loyalty` queue age, and scale `loyaltyservice` consumers independently of email.
  - *Duplicate points.* Mitigation: a unique constraint on order ID in the loyalty store, written in the same transaction as the points.
  - *Poison messages.* Mitigation: a dead-letter queue per consumer, with an alert on DLQ depth.
  - *Broker becomes a bottleneck or SPOF.* Mitigation: quorum queues, pod anti-affinity, a load test at 200 msg/s or more.

## Validation

- **Load test** at **200 orders/s** (e.g. with the Locust `loadgenerator`, as in ADR-0001). Pass criteria: p95 `PlaceOrder` ≤ **500 ms**, and zero failed orders.
- **Fault injection:** stop `emailservice`, `loyaltyservice`, analytics and MailFast (a stub) one at a time during the load test. Orders must still succeed (driver 1).
- **Loyalty SLO:** measure order-to-points time. At peak, 100 % within **60 s**. Replay duplicate messages and assert **zero** double awards.
- **Email:** confirm the MailFast send rate never exceeds **50/s**, and that every message is eventually sent or dead-lettered.
- **Review** after the first 11.11 sale using broker metrics (queue depth/age, DLQ counts).

## Open questions

- **Peak duration** of the 11.11 sale is not given, so the maximum email backlog (about 150 × seconds of peak) and the RabbitMQ disk/memory sizing can't be computed.
- **Acceptable email delay:** no target is given for how late a confirmation email may arrive.
- **Lost events when the broker is down at publish time:** is that acceptable for loyalty (which must be "exactly once")? If not, `checkoutservice` needs a persistent outbox, which requires a data store it does not have today.
- **Analytics sink:** where marketing's events must land, and their delivery and latency requirements. Not given.
- **`loyaltyservice` data store:** technology and ownership. Not given.
- **MailFast API details:** protocol, idempotency keys (to avoid duplicate emails on retry), and behaviour when the rate limit is exceeded. Not given.
- **RabbitMQ operations experience:** the two developers "have used RabbitMQ", but whether anyone has *operated* a cluster on Kubernetes is not stated.
- **Deciders:** the names and roles of the decision makers are not given.
- **Exactly-once for future actions:** whether SMS or fraud scoring will also need exactly-once semantics is not stated.
