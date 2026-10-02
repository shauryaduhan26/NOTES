# Designing a Distributed Notification System

## 1. Overview and Goals

A notification system delivers messages to users across multiple channels — push (mobile/web), SMS, email, and in-app — triggered either by real-time events (an order shipped, a password reset, a new message) or by scheduled/bulk campaigns (a marketing blast, a system-wide alert). It sits between the business services that decide "something happened, tell the user" and the third-party gateways (APNs, FCM, Twilio, SES) that actually reach the user's device or inbox.

**Functional requirements:**

- Accept notification requests from many internal services (order service, chat service, billing service, marketing tooling) through one consistent API.
- Support multiple channels — push, SMS, email, in-app — with a single logical notification able to fan out across more than one.
- Respect per-user preferences: channel opt-in/opt-out, category-level subscriptions, quiet hours, and language/locale.
- Render templated content with per-user personalization and localization.
- Guarantee delivery is attempted reliably (retries, failure isolation) without ever delivering the *same* notification to a user twice due to an internal retry.
- Support both a single targeted notification (one user) and a broadcast/campaign (millions of users) through the same core pipeline.
- Track delivery outcomes (sent, delivered, opened, clicked, bounced, failed) for analytics and for feeding preference/frequency decisions.

**Non-functional requirements:**

- **Scalability**: tens of millions of notifications per hour during peak events, for a platform with hundreds of millions of registered users.
- **Low latency for time-sensitive notifications**: end-to-end delivery latency under a few seconds for real-time/transactional notifications, even while bulk campaigns are also in flight.
- **High deliverability**: industry targets are commonly cited around 95%+ for push and 98%+ for email, net of invalid tokens/addresses and provider-side filtering.
- **Reliability without duplication**: at-least-once delivery through the internal pipeline, with deduplication at the boundary so "at least once internally" doesn't become "more than once to the user."
- **Extensibility**: adding a new channel or a new third-party provider shouldn't require changes to the core orchestration logic.
- **Respect for the user**: notification fatigue (too many, too often) is a product and retention risk, not just a technical one — frequency capping and quiet hours are first-class requirements, not an afterthought.

These pull against each other in familiar ways: low latency for a single transactional notification competes for the same infrastructure as a 50-million-user broadcast; strict delivery guarantees (retry until success) compete with strict duplicate-avoidance (never send twice); and respecting quiet hours/frequency caps means some notifications are deliberately *not* sent immediately, which cuts against "low latency" as a blanket goal — it only applies to notifications that are supposed to go out right now.

## 2. Basic Data Model

```
Notification Request:
  <notification_id, event_type, user_id (or audience_query for broadcast),
   category, priority: {critical, transactional, promotional},
   template_id, template_params, requested_channels[], created_at>

User Preference Record:
  <user_id, channel, category, opt_in: bool, quiet_hours: {start, end, timezone},
   frequency_cap: {max_count, window}>

Device/Contact Record:
  <user_id, channel, address (device_token | phone_number | email | connection_id),
   platform (ios | android | web), last_validated_at, status: {active, invalid}>

Delivery Record:
  <notification_id, user_id, channel, status: {queued, sent, delivered, opened,
   clicked, bounced, failed}, provider_message_id, attempt_count, last_attempt_at>
```

Splitting these into four records (rather than one giant notification blob) matters for the same reason it mattered in other high-throughput designs: each has a different size, update frequency, and access pattern. Preferences and contact records are small, read very frequently (once per notification attempt) and updated rarely; delivery records are written frequently (once per attempt, plus status updates from provider callbacks) and read mostly by analytics, not by the hot send path.

## 3. High-Level Architecture

```
 Internal Services                    Marketing / Campaign Tooling
 (order, chat, billing)                         │
        │ single notification                   │ one broadcast request
        ▼                                        ▼
 Notification API                      Campaign Orchestrator (§10)
        │                               (cursor-paginated batches
        │                                of 10k–50k users each)
        │                                        │ one batch message/batch
        │                                        ▼
        │                               batch-expansion topic + workers
        │                               (expands 1 batch → N individual
        │                                notification requests, §10)
        │                                        │
        └───────────────────┬────────────────────┘
                             ▼  every notification request, single or
                                campaign-sourced, converges here:
                 ┌───────────────────────────┐
                 │  Preference Check (§5):   │
                 │  opt-in, quiet hours,     │
                 │  frequency cap            │
                 └─────────────┬─────────────┘
                                │ allowed (not dropped/deferred)
                                ▼
                 ┌───────────────────────────┐
                 │  Template Renderer (§6)   │
                 │  (personalization+locale) │
                 └─────────────┬─────────────┘
                                ▼
                 ┌───────────────────────────┐
                 │   Router (§7.1): picks     │
                 │   channel × priority topic│
                 └──┬─────┬─────┬─────┬──────┘
                    ▼     ▼     ▼     ▼
              12 topics (4 channels × 3 priority tiers, §7.1)
                    │     │     │     │
              ┌─────┘  ┌──┘  ┌──┘  ┌──┘
              ▼        ▼     ▼     ▼
         Push Worker  SMS   Email  In-App   (§8 — one pool per channel,
              │      Worker Worker Worker    each with its own consumer
              ▼        ▼     ▼     ▼         group scaling policy, §7.1)
          APNs/FCM  Twilio  SES/   WebSocket
                     /SNS  SendGrid  conn.
              │        │     │     │
              └────┬───┴──┬──┴─────┘
                   ▼       ▼
          Delivery status callbacks/webhooks (§14)
                   │
                   ├──► Delivery Record Store + Analytics (§15, §18)
                   │
                   └──► Cross-Channel Fallback Orchestrator (§9):
                        watches delivery/open signals; if this channel
                        doesn't confirm in time, re-injects a *new*
                        notification request (next channel in the
                        fallback chain) back into the Preference Check
                        stage above — closing the loop.
```

A request flows in through one of two entry points — a direct API call for a single notification, or a campaign orchestrator that turns one broadcast request into many individual requests (§10) — but **both converge on the exact same pipeline** from that point on: preference check, template rendering, routing to the correct channel/priority topic, channel-specific delivery, and status tracking. Delivery outcomes don't just dead-end in the analytics store; they also feed the fallback orchestrator (§9), which can loop a *new* request for the next channel back into the very top of this same pipeline. There is one pipeline, not a different one per entry point or per retry path.

### 3.1 End-to-End Walkthrough: A Single Transactional Notification

To make the diagram above concrete, here's exactly what happens, component by component, when the order service ships a package and wants to tell the user — with a push-then-email fallback chain configured.

1. **Trigger**: the order service calls the Notification API with `notification_id = "ord-9981-shipped"` (a caller-generated idempotency key, §11), `user_id`, `category = "transactional"`, `requested_channels = [push, email]` in that fallback order (§9), and the template parameters (order ID, ETA).
2. **Preference check (§5)**: the service looks up the user's preference record (§2) for the `shipping` category on the `push` channel. Suppose push is opted in and it's not currently the user's quiet hours (transactional notifications would proceed through quiet hours anyway, but this one happens not to need that override) — the request proceeds. The frequency cap (§5) is checked too, but transactional notifications are typically exempt from it by policy (§4).
3. **Template rendering (§6)**: the push-specific template for `order_shipped` is rendered with this user's locale and the order parameters, producing the actual push payload text.
4. **Routing (§7.1)**: the router reads `channel = push`, `priority = transactional`, and publishes the rendered notification onto the `push-transactional` topic.
5. **Consumption**: `push-transactional`'s consumer group — provisioned with always-warm capacity (§7.1), unaffected by whatever volume is currently sitting in `push-promotional` — picks up the message within milliseconds.
6. **Idempotency check (§11)**: before dispatching, the push worker checks the dedup cache for `"ord-9981-shipped"`. Not present (first attempt) — it writes the key to the cache and proceeds.
7. **Dispatch (§8)**: the worker looks up the user's device/contact record (§2) for a push token and sends the payload to FCM. Suppose, in this run, the token has actually gone stale (the user uninstalled the app weeks ago) — FCM returns an invalid-token error.
8. **Failure classification (§13)**: an invalid-token error is non-retryable (retrying it would never succeed), so the worker immediately marks the contact record invalid (§8) and records this delivery attempt as `failed` (§2) — it does *not* enter the retry/backoff path, which is reserved for transient failures.
9. **Fallback (§9)**: the cross-channel orchestrator, watching for a delivery confirmation on the push attempt, sees the immediate `failed` status (no need to even wait for a timeout, since the failure was definitive) and generates a *new* notification request for the next channel in the chain — email — with a fresh notification context. This new request re-enters the pipeline at the very top of §3's diagram: preference check again (is email opted in for shipping?), template rendering again (the email-specific template, not the push one), routing to `email-transactional`.
10. **Second dispatch**: the email worker (§8) sends via SES; SES accepts it immediately, and a delivery webhook (§14) confirms actual delivery a few seconds later.
11. **Final state**: the Delivery Record store (§15) now shows two attempts for this logical notification — `push: failed (invalid_token)`, `email: delivered` — and the analytics pipeline (§18) rolls this up into the platform's push-deliverability metrics, which is exactly the kind of signal that should (per §8) prompt investigation if invalid-token failures are trending upward across many users, not just this one.

This same numbered sequence — preference check, render, route, queue, dispatch, classify-on-failure, fall back if needed, track — is the *only* path any notification takes, whether it originated from a single API call or from the campaign fan-out below. Nothing about §9's fallback logic, or §13's retry logic, is a separate code path; both operate by generating ordinary new requests that flow through the same pipeline.

### 3.2 End-to-End Walkthrough: A Broadcast Campaign

Now trace the other entry point from §3's diagram — a marketing campaign targeting 50 million users by email — showing exactly where it merges into the same pipeline above.

1. **Campaign creation**: a marketer defines a campaign record (§10) — target audience criteria, the promotional template, a scheduled send time.
2. **Batching**: at the scheduled time, the campaign orchestrator queries the user store with cursor-based pagination (§10), producing roughly 2,500 batches of 20,000 users each for 50 million total users (§16's worked capacity numbers).
3. **Batch publication**: each batch becomes one message on the `batch-expansion` topic — 2,500 messages, not 50 million, keeping the orchestrator's own publish load trivial regardless of campaign size.
4. **Expansion**: a pool of batch-expansion workers consumes `batch-expansion`, and for each of the 20,000 users in a batch, emits one individual notification request with its own `notification_id` — these are now indistinguishable, structurally, from the single order-shipped request in §3.1 above, except `category = "promotional"`.
5. **Convergence (the key point)**: every one of these up-to-50-million individual requests goes through **the exact same preference check (§5) as step 2 of the single-notification walkthrough** — a user who's opted out of marketing email, or currently within their quiet hours, is filtered out right here, meaning the campaign's *effective* send volume is typically well under the full 50 million target, not a guaranteed 50 million sends.
6. **Rendering and routing**: surviving requests are rendered (§6) and routed (§7.1) to `email-promotional` — mixed in with any other promotional email traffic in flight, but still fully isolated from `email-transactional` and `email-critical`.
7. **Autoscaling under load**: `email-promotional`'s consumer group, which may have been scaled down to near-zero between campaigns (§7.1), observes rising consumer lag as this flood of messages lands and scales up its worker pool over the following minutes.
8. **Provider-level rate limiting (§12)**: even with more consumers running, the aggregate send rate to SES is still capped by a token bucket matching SES's documented ceiling — so the campaign's actual send rate out to the internet is governed by the provider limit, not by however many consumer instances happen to be running, which is exactly why step 7's autoscaling has a ceiling rather than scaling without bound.
9. **Meanwhile, transactional traffic is unaffected**: a password-reset email arriving on `email-transactional` during this entire campaign is handled by its own always-warm consumer group (§7.1) and never waits behind any of this campaign's 50 million candidate requests — this is the concrete, worked proof of the isolation property described in §7.1's topic-structure explanation.
10. **Tracking at scale**: delivery, bounce, and complaint webhooks (§14) for this campaign's sends flow into the same Delivery Record store as any other notification (§15), and the campaign's aggregate open/click/bounce rates become visible in analytics (§18) without needing any campaign-specific storage or tracking infrastructure.

## 4. Notification Types and Priority — Deep Dive

Not all notifications are equal, and treating them uniformly either wastes infrastructure on low-value sends or delays high-value ones behind a backlog. Three broad categories:

- **Critical/system alerts** (security alert, payment failure): must be delivered with minimal latency and highest retry persistence; typically bypass frequency caps and quiet hours entirely (a fraud alert shouldn't wait until 9am).
- **Transactional** (order confirmed, package shipped, password reset): time-sensitive, tied to a specific user action, expected by the user, and generally exempt from marketing-style frequency caps but still subject to quiet hours for non-urgent cases.
- **Promotional/bulk** (marketing campaigns, digest emails, re-engagement pushes): the least time-sensitive, most subject to frequency capping and quiet hours, and the primary source of the broadcast fan-out problem (§10).

Each category maps to its own queue/topic with its own consumer scaling policy (§8) — critical and transactional traffic gets dedicated, always-warm consumer capacity, while promotional traffic can be processed on a more elastic, lower-priority pool that scales down between campaigns, so a large marketing send never starves a password-reset email of throughput.

## 5. Preference and Subscription Management — Deep Dive

**Preference checks happen before rendering or queuing**, not after — there's no point personalizing content or paying queue/provider costs for a notification that's about to be dropped. The check evaluates, in order:

1. **Category opt-out**: has the user disabled this notification category on this channel? If so, drop immediately (unless the category is marked non-optional, like security alerts).
2. **Quiet hours**: is it currently within the user's configured quiet window, evaluated in *the user's own timezone*, not the server's? If so — and the notification isn't marked critical — the notification is deferred (re-queued with a scheduled delivery time at the end of the quiet window) rather than dropped outright.
3. **Frequency cap**: has this user already received `max_count` notifications (in this category, or overall) within the configured rolling `window`? If the cap is already hit, the notification is either dropped or deferred to the next open slot, depending on the category's policy.

**Frequency capping mechanics.** A per-user, per-category counter (commonly held in a fast in-memory store like Redis, since this check has to happen on every single notification attempt) tracks how many sends have occurred in the current rolling window; each attempt does an atomic increment-and-check, and the notification is allowed only if the result stays under the cap — the same request-path pattern as a rate limiter, just scoped to "one user's inbox" instead of "one API client."

## 6. Template Rendering and Localization

A template defines the structure of a message (`"Your order {{order_id}} has shipped and will arrive by {{eta}}"`) separately from any specific notification instance; rendering substitutes the actual parameters and applies the user's locale (date/currency formatting, and translated copy where available, falling back to a default locale if a translation is missing). Templates are versioned, and each rendered notification records which template version it used — useful both for debugging ("why did this notification look wrong") and for A/B testing different copy on the same event type.

Rendering happens **per channel**, not once globally: the same logical event ("order shipped") produces a short, punchy push notification, a longer email with more detail and an image, and a plain-text SMS with a shortened link — three different templates keyed to the same event type and category, chosen based on which channels the preference check (§5) actually allowed for this user.

## 7. Message Queue — Deep Dive

Putting a durable queue between "decided to send" and "actually sent" is what makes the rest of the system's reliability properties possible: it absorbs bursts (a campaign of 50 million sends doesn't need 50 million workers instantaneously, just a queue that holds the backlog while workers catch up), it isolates failures (a slow or down third-party provider backs up that channel's queue without blocking any other channel), and it's the natural place to implement retries (a failed send is just re-enqueued rather than lost).

### 7.1 Topic Structure, in Detail

**Why one shared queue per channel isn't good enough.** A queue (or a Kafka-style topic) is consumed roughly in the order messages arrive. If push notifications of every priority (§4) all went into one `push` topic, a burst of 10 million promotional pushes landing at 2:00:00pm would sit *ahead* of a password-reset push that happens to arrive at 2:00:01pm, because the consumer has no way to "jump the queue" for the more urgent message without reading and discarding everything in front of it first — this is **head-of-line blocking**, and it's exactly the failure mode the priority tiers in §4 exist to prevent. A priority *field* on the message doesn't fix this by itself; something still has to act on that field, and the simplest, most robust way to act on it is to never let the two priorities share a queue in the first place.

**The actual structure: one topic per (channel × priority) pair.** For 4 channels and 3 priority tiers, that's 12 independent topics:

```
push-critical        sms-critical        email-critical        inapp-critical
push-transactional   sms-transactional   email-transactional   inapp-transactional
push-promotional     sms-promotional     email-promotional     inapp-promotional
```

A lightweight **router**, sitting right after template rendering (§6), reads each notification's channel and priority and publishes it to exactly one of these 12 topics — there's no shared infrastructure downstream of that point between, say, `email-promotional` and `email-transactional` except that they both eventually call the same email provider (§8).

```
                                   ┌──────────┐
 Rendered notification ──────────►│  Router  │
 (channel=push,                   │ (channel │
  priority=critical)              │  × tier) │
                                   └────┬─────┘
                                        │ publishes to exactly one topic
        ┌───────────────┬──────────────┼──────────────┬───────────────┐
        ▼               ▼              ▼               ▼               ▼
 push-critical   push-transactional  push-promotional  sms-critical   ... (8 more)
        │               │              │               │
   ┌────┴────┐     ┌────┴────┐    ┌────┴────┐     ┌────┴────┐
   │ Consumer │     │ Consumer │    │ Consumer │     │ Consumer │
   │  Group   │     │  Group   │    │  Group   │     │  Group   │
   │ (fixed,  │     │ (fixed,  │    │(autoscale│     │ (fixed,  │
   │ always-  │     │ always-  │    │ 0 → N on │     │ always-  │
   │  warm,   │     │  warm,   │    │  queue   │     │  warm)   │
   │ min N≥k) │     │ min N≥k) │    │  depth)  │     │          │
   └────┬────┘     └────┬────┘    └────┬────┘     └────┬────┘
        ▼               ▼              ▼               ▼
   APNs/FCM        APNs/FCM        APNs/FCM          Twilio/SNS
   (same worker *image*, 3 separate, independently-scaled deployments)
```

**What "always-warm" actually means.** A consumer group's instances are ordinary long-running processes (pods/containers) that stay connected to the broker and sit in a poll loop, already assigned their partitions, whether or not there's currently any traffic. "Always-warm" means the minimum instance count for that group is a fixed floor **above zero** — it never scales down to nothing — specifically so that when a critical notification arrives, there is already a live, broker-connected, partition-assigned process free to pick it up immediately. The alternative — scaling to zero between messages and spinning up a fresh instance on demand — adds real, user-visible latency before the first message of a burst is even touched: container start, process/runtime startup, re-establishing the broker connection, and the consumer-group rebalance protocol re-assigning partitions can together take anywhere from a few seconds to tens of seconds. That's immaterial for a marketing email, but it's an unacceptable delay bolted onto a password-reset push or a fraud alert. So the critical and transactional consumer groups pay for idle standby capacity around the clock specifically to buy near-zero first-message latency; the promotional group makes the opposite trade (§"Autoscaling the promotional tier" below) because its SLA can absorb a cold start.

**What is — and isn't — actually shared across the 3 push topics.** The earlier version of this diagram said the push worker fleet was "shared across all 3 push topics' consumer groups," which overstates it and is worth correcting explicitly: the three push consumer groups (`push-critical`, `push-transactional`, `push-promotional`) are three **separate** consumer groups, each with its own independently-scaled pool of running instances — a critical-tier instance never dequeues from the promotional topic and vice versa. If they *did* share running instances, a single worker process would eventually have to finish working through a promotional backlog before it got back around to checking the critical topic, which is precisely the head-of-line blocking this whole topic split exists to prevent. What genuinely *is* shared across the three is only the **application code** — the same "push worker" image/binary is deployed three times with three different topic subscriptions and three different scaling configurations, so there's one codebase to maintain instead of three forks of near-identical logic — and, at the infrastructure level, the same downstream APNs/FCM provider account and connection pool, since the provider doesn't know or care which internal topic a given push came from. Compute capacity and the act of consuming messages are never shared; only the deployable artifact and the downstream provider connection are.

**Why this, specifically, prevents the scenario described.** `push-critical` and `push-transactional` each have their *own* consumer group, provisioned with a minimum number of always-running consumer instances (a floor above zero, sized for comfortably more than average load, not just average load, so there's headroom for a spike). These consumers only ever read from their own topic — they have no visibility into, and no contention with, whatever backlog is piling up in `push-promotional`. A 10-million-message flood into `push-promotional` can only ever slow down *other promotional* traffic; it physically cannot delay a message that was routed to `push-critical`, because that message was never in the same topic, queue, or consumer group to begin with.

**Autoscaling the promotional tier.** `push-promotional`'s consumer group is configured to scale its instance count based on **consumer lag** (how many messages are enqueued but not yet processed, or how far behind the consumer is from the latest offset) — scaling from a small baseline (or even zero between campaigns) up to a large pool during an active broadcast (§10), and back down afterward. This is the right trade-off specifically because the SLA for a promotional send tolerates being a few minutes late during a spike, while the critical/transactional tiers' SLA does not — so only the tier that can tolerate delay is the one allowed to experience it.

**Partitioning within a topic.** Each of these 12 topics is itself typically split into multiple partitions (in the Kafka sense) for parallelism — commonly partitioned by a hash of `user_id`, so all notifications for *that user on that specific topic* land on the same partition and are processed in a defined order relative to each other (useful, for instance, if two promotional pushes for the same user are generated moments apart and need to be delivered in the order they were produced), while different users' notifications spread across many partitions and many consumer instances for throughput. This is the same shape of trade-off as sharding any high-throughput stream: pick a partition key that keeps *related* messages together for ordering, while scattering *unrelated* messages for parallelism.

**A correction worth being precise about: this ordering doesn't extend across the fallback chain.** It's tempting to read the paragraph above as also covering §9's fallback scenario, but it doesn't — a push-then-email fallback produces one message on `push-transactional` and, if needed, a second on `email-transactional`: two different topics, each with its own independent partitioning and its own independent consumer group. Kafka (or any topic-based queue) gives no cross-topic ordering guarantee, so there is no queue-level mechanism that keeps "the push attempt" ordered before "the email attempt" the way same-topic partitioning keeps two promotional pushes for one user in order. Correctness across the fallback chain is enforced entirely by §9's own state machine — the durable per-attempt timer and the compare-and-set against the logical notification's status — not by anything the partitioning scheme provides. The two mechanisms solve different problems: partitioning orders *same-topic* traffic for throughput without losing order; the fallback orchestrator sequences *cross-topic* traffic for correctness, by tracking state explicitly rather than relying on delivery order.

## 8. Channel Workers — Deep Dive

Each channel has genuinely different failure modes and constraints, which is why they're separate worker pools rather than one generic "send" function with an if/else on channel type.

**Push (APNs/FCM).** A push worker holds a persistent, authenticated connection (APNs) or makes authenticated HTTP requests (FCM) to deliver a payload to a specific device token. Device tokens are inherently unstable: they invalidate when a user uninstalls the app, reinstalls, restores from backup, or clears app data, and FCM treats a token unused for roughly nine months as stale on its own. Both APNs and FCM return explicit error responses (or, for APNs, feedback through the send connection itself) identifying invalid tokens — **the worker must consume these error signals and mark the corresponding contact record invalid** (§2); backends that ignore this feedback accumulate a growing table of dead tokens that silently drags down measured delivery rates and wastes provider quota on sends that can never succeed.

**SMS (Twilio/SNS or a carrier aggregator).** SMS has a real per-message cost and is subject to carrier-level throttling and country-specific regulations (sender ID requirements, opt-out keyword handling like "STOP"). The SMS worker respects a stricter, provider-imposed rate limit than push typically needs, and a `STOP` reply from a user must be treated as an authoritative, immediate opt-out — overriding the preference store (§5) rather than waiting for the next preference sync.

**Email (SES/SendGrid).** Email delivery is judged heavily on sender reputation: bounce rate and spam-complaint rate directly affect whether a provider's infrastructure gets throttled or blacklisted by receiving mail servers. The email worker must process bounce and complaint webhooks (§14) and immediately suppress future sends to a hard-bounced address or a user who's marked the sender as spam, and every email must carry a compliant one-click unsubscribe mechanism (both as a practical deliverability requirement and a legal one, §17).

**In-app (WebSocket/long-lived connection).** For a user with an active session, in-app notifications can be pushed directly over an existing WebSocket connection with sub-second latency and no third-party provider at all. If the user has no active session, the in-app worker's job is to recognize that and **fall back to push** (§9's cross-channel orchestration) rather than silently dropping the notification.

## 9. Cross-Channel Fallback and Orchestration

A single logical notification often specifies more than one acceptable channel, with an ordered preference and a fallback rule — e.g., "try in-app first; if no active session within N seconds, send push; if no push delivery confirmation within M minutes and this is high-priority, also send SMS." This orchestration logic lives above the individual channel workers: it dispatches to the first channel, watches for a delivery/open signal (§14), and conditionally dispatches to the next channel in the fallback chain if the signal doesn't arrive in time.

**Mechanically, "watches for a signal" means a durable, per-attempt timer, not a blocking wait.** The orchestrator doesn't hold a thread open waiting for each notification's outcome — that wouldn't scale past a tiny number of concurrent notifications. Instead, when it dispatches to channel 1, it writes a durable timer/scheduled-job entry keyed by this notification's `logical_id` (distinct from the per-channel-attempt `notification_id`, §11) with a deadline (`now + M minutes`). Two things race against that deadline: a delivery/open webhook (§14) arriving and updating the logical notification's status to `confirmed`, or the timer firing. Whichever happens first wins — if the webhook arrives first, the orchestrator cancels the pending timer and does nothing further; if the timer fires first, the orchestrator checks the logical notification's current status one more time (in case the webhook arrived a moment before the timer but hasn't been fully processed yet) and only then generates the next channel's request.

**The race this needs to close explicitly: what if the timer fires and the fallback request is dispatched right as the original channel's delayed confirmation also arrives?** This is checked, not assumed away: the status check immediately before generating the fallback request (above) is a conditional update — "mark this logical notification as having moved to channel 2, but only if its status is still `pending` for channel 1" — implemented as a compare-and-set against the logical notification's stored status, the same pattern used anywhere a check-then-act sequence needs to be safe against a concurrent writer. If the delayed confirmation's status update wins that race, the compare-and-set fails, the timer handler sees that the status has already changed to `confirmed`, and it does *not* generate a fallback request — the user doesn't get a redundant SMS for a push that, it turns out, actually did arrive just late.

This keeps fallback orchestration and the channel worker's own retry logic (§13) from double-triggering each other: a *transient* failure on channel 1 is retried on channel 1 by §13's retry logic without ever involving the orchestrator, while only a channel-1 attempt that's either definitively failed (§13's non-retryable classification) or has gone unconfirmed past its deadline causes the orchestrator to move to channel 2 — the two mechanisms operate on different signals (retry logic reacts to dispatch-time errors; fallback orchestration reacts to the *absence* of a confirmation after dispatch succeeded) and don't overlap in what triggers them.

## 10. Fan-Out for Broadcast Notifications — Deep Dive

A campaign targeting millions of users can't be handled as millions of individual API calls arriving at once — the standard pattern is a **campaign orchestrator** that turns one broadcast request into many individually-tracked notification requests, gradually:

1. The campaign is stored as its own record (target audience criteria, template, schedule).
2. The orchestrator queries the user store for matching users in **batches** (commonly on the order of 10,000–50,000 users per batch), using **cursor-based pagination** rather than offset-based, so the query stays efficient even deep into a many-million-row result set.
3. For each batch, the orchestrator publishes one batch message onto a dedicated `batch-expansion` topic — **not** directly onto one of the final channel×priority topics from §7.1. A pool of batch-expansion workers consumes this topic and turns each batch into individual notification requests, one per user in the batch, each carrying its own `notification_id` (§11).
4. **Each individual request produced by expansion re-enters the exact same pipeline a single API-triggered notification uses** — it still goes through the preference check (§5), template rendering (§6), and the §7.1 router — rather than skipping straight to the channel worker. This is the important correction to make explicit: fan-out only multiplies *how many* individual notification requests exist; it never bypasses preference checking or rendering for any of them. A user who's opted out, or currently in quiet hours, is filtered out at this stage exactly as they would be for a one-off transactional notification — the campaign simply produces up to 50 million *candidate* requests, not 50 million guaranteed sends.
5. Only after surviving the preference check does each rendered notification land in its channel's `-promotional` topic (§7.1) — e.g., `email-promotional` for an email campaign — where it's indistinguishable from any other promotional-tier message as far as the consumer group and channel worker (§8) are concerned.

This keeps the architecture consistent end-to-end: there is exactly one pipeline (preference check → render → route → queue → worker → provider), and "fan-out" describes how a single campaign request generates many *inputs* to that one pipeline, not a separate, parallel code path with its own rules.

**Worked example.** A campaign targets 50 million users across push and email. The orchestrator issues cursor-paginated queries producing 1,000–5,000 batches of 10,000–50,000 users each, publishing one batch message per batch rather than 50 million individual messages up front — this keeps the orchestrator's own memory and query load bounded and lets the batch messages themselves be rate-controlled (throttling how fast new batches are published) if downstream queues start to back up, rather than dumping the entire campaign's load onto the queue in one shot.

## 11. Idempotency and Deduplication — Deep Dive

Because the pipeline is built on queues offering **at-least-once** delivery (a consumer can crash after processing a message but before acknowledging it, causing redelivery), the same notification request can genuinely be processed more than once internally. Without a safeguard, that becomes a duplicate notification reaching the user — visibly broken behavior, not just an internal inefficiency.

**The fix**: every notification request carries a unique idempotency key (the `notification_id`, generated by the originating service, not by the notification system itself — so a *retried request from the caller* also collapses onto the same key). Before a channel worker actually dispatches to a provider, it checks a fast dedup cache (commonly Redis, with a TTL comfortably longer than the pipeline's realistic redelivery window) for that key; if it's already present, the worker skips the send and simply re-acknowledges the message as handled. If not present, it writes the key to the cache *before* dispatching (or atomically as part of dispatch, to close the race where two redeliveries are being processed concurrently) and proceeds.

**Why not exactly-once instead?** A distributed exactly-once delivery guarantee (transactional coordination between the queue and every downstream side effect) is achievable but adds real coordination overhead and complexity across every channel worker and third-party call. At-least-once delivery combined with idempotency-key deduplication at the boundary gets the same practical outcome (the user never sees a duplicate) with a far simpler, cheaper mechanism — this is the standard industry trade-off, not a shortcut.

**A correction worth making explicit: dedup state needs more than a presence bit.** The mechanism as described above has a real flaw if implemented literally: if the dedup cache is written the moment a worker *begins* dispatching — before it knows whether the provider call actually succeeds — then a transient failure on the first attempt (a throttle response, a network blip) still leaves the key present in the cache. When §13's retry logic re-attempts that same `notification_id` moments later, the worker checks the cache, finds the key already present, and — per the rule as stated — treats it as a duplicate and skips the send, permanently swallowing a notification that was never actually delivered. The fix is to store a **status**, not just a presence flag: `in_flight` when dispatch begins, `delivered` only once the provider confirms success (synchronously for providers like SES/Twilio that accept inline, or asynchronously via the §14 webhook for providers that confirm later), and `failed` once a non-retryable error is classified (§13). The dedup check's rule then becomes: skip the send only if the stored status is `delivered`; a `failed` status, or an `in_flight` entry that's older than a short staleness threshold (covering a worker that crashed mid-dispatch without ever recording an outcome), allows the retry to proceed normally. This is the same three-state pattern idempotency keys use in payment APIs — Stripe's idempotency records, for instance, distinguish `in_progress` / `succeeded` / `errored` for exactly this reason — because a bare presence/absence check can't distinguish "already delivered, don't resend" from "attempted and failed, please do resend."

## 12. Rate Limiting — Deep Dive

Rate limiting in this system operates at three distinct levels, each protecting a different resource:

- **Per-user, at the orchestrator/preference-check layer**: the frequency cap from §5 — protects the *user* from notification fatigue, not the infrastructure.
- **Per-worker/in-process**: a local rate limiter bounds how fast a single worker instance dispatches to a given provider, smoothing bursts before they even reach the provider-level check below.
- **Per-provider, respecting vendor-imposed caps**: APNs, FCM, Twilio, and email providers all enforce their own rate limits and will throttle or reject a sender that exceeds them — the channel worker fleet's aggregate send rate to each provider must stay under that provider's documented ceiling, commonly enforced with a **token bucket** (allows short bursts up to the bucket's capacity while enforcing a steady long-term average rate) rather than a hard fixed-window cap, which would needlessly reject bursty-but-compliant traffic.

**A correction worth making explicit: one bucket per provider isn't actually enough.** A single, aggregate token bucket shared by *all three priority tiers'* workers for a given provider reintroduces the exact head-of-line-blocking problem §7.1's topic split was built to solve — just one layer downstream. `push-critical` and `push-promotional` have fully separate topics and fully separate worker pools, but if both pools draw from the same APNs token bucket, a promotional flood can still exhaust that shared budget, and a critical send that arrives at that moment waits for a token alongside everything else — delayed by traffic it was supposed to be completely isolated from.

**The fix: split the provider-level bucket by tier, not just by provider.** Instead of one bucket per provider, reserve a guaranteed minimum allocation of the provider's total allowed rate exclusively for critical/transactional traffic — e.g., if APNs tolerates 1,000 req/sec in aggregate, carve out 200 req/sec that only critical/transactional workers can draw from regardless of promotional demand, and let promotional draw from the remaining capacity (or from its own separate, smaller bucket capped so it can never crowd out the reserved share). This is the same shape of trade-off as strict-priority or guaranteed-minimum-bandwidth queuing in network QoS, applied to an API rate limit instead of a network link. It also means §7.1's promotional autoscaling has a ceiling that isn't really about instance count: once the promotional pool's own bucket is the bottleneck, adding more worker instances doesn't increase throughput — they just queue up waiting for tokens from a budget that's already fully allocated.

A stronger, more expensive alternative exists where the provider supports it: issuing a **separate credential per tier** (a distinct FCM sender ID, a distinct Twilio messaging-service SID, a distinct SES configuration set) gets an independently-enforced quota from the provider's own side, rather than one quota partitioned in application code. More robust isolation, but it means managing multiple provider accounts/credentials, potentially losing volume-based pricing tiers, and some providers still apply a higher-level cap (IP or domain reputation) above the per-credential limit — so even this isn't perfect isolation, just a stronger version of the same idea.

### 12.1 Rate-Limiting Algorithms, Compared

Token bucket was named above as the mechanism for the per-provider check, specifically to avoid a problem worth walking through in full: the fixed-window counter's failure modes, and why the alternatives exist.

**Fixed window counter.** The simplest possible design: pick a window size (say, 1 second), keep a counter per key, increment it on every request, reject once the counter hits the limit, and reset the counter to zero at the start of the next window. It has two distinct, independent flaws, not just one:

1. **It needlessly rejects legitimate bursts that follow idle time.** If a worker is idle for 9 seconds and then needs to send a batch of 150 messages in the 10th second, that's only 15 req/sec averaged over the full 10 seconds — well under a 100/sec sustained limit — but a fixed window only ever looks at the *current* window in isolation. It has no memory of the idle time that preceded it, so it rejects 50 of those 150 messages even though the provider was never actually at risk.
2. **It under-enforces at window boundaries.** 100 requests landing at 0.99s and another 100 at 1.01s each pass their own window's check individually — each window saw exactly 100, right at the limit — but that's 200 requests inside roughly 20 milliseconds of real time, double the intended rate, and a naive fixed-window scheme never notices because "boundary" is an artifact of how the counter is reset, not a property of the actual traffic.

Both flaws trace back to the same root cause: a fixed window makes an accept/reject decision based on an arbitrary, resettable slice of the clock rather than on the actual traffic in any given rolling period.

**Sliding window log.** The exact fix: instead of one counter per window, store the *timestamp* of every request that's happened for a given key. On each new request, evict every stored timestamp older than `now − window_size`, count what's left, and reject if that count is already at the limit; otherwise record the new timestamp and allow it. This is perfectly accurate for any rolling window, by construction — there's no boundary to game because there's no fixed boundary at all, just a continuously-sliding `now − window_size` cutoff. The cost is memory and compute that scale with the limit itself: a key allowed 10,000 requests/sec needs up to 10,000 resident timestamps and an eviction pass on every single check, which gets expensive fast at the throughput this system operates at (§16).

**Sliding window counter (the practical compromise).** Most production systems don't use the full log; they approximate it with two fixed-window counters — the current window's count and the *previous* window's count — and a weighted estimate: `estimated_count = previous_window_count × (fraction of the previous window still inside the rolling window from now) + current_window_count`. Worked example: window size 1s, limit 100, and `now` is 0.3 seconds into the current window. The rolling window therefore still overlaps 70% of the previous window, so if the previous window saw 100 requests and the current window has 20 so far, the estimate is `100 × 0.7 + 20 = 90` — under the limit, so the request is allowed. This is O(1) memory per key (two counters, not one timestamp per request), closely approximates the sliding log's accuracy, and is what most API gateways and CDNs actually run in production rather than a true log.

**Token bucket, restated against this backdrop.** A bucket holds up to a fixed number of tokens, refills continuously at the target sustained rate (fractions of a token per millisecond, not "a tokens at the top of each window"), and a request consumes one token to proceed or waits/gets rejected if the bucket is empty. Because refilling is continuous rather than reset-on-a-clock-tick, there's no discrete boundary to exploit the way a fixed window has, and because unused refill capacity accumulates in the bucket up to its cap, a legitimate burst following idle time is paid for out of the accumulated tokens rather than being rejected outright — which is exactly the "bursty-but-compliant" case the fixed window handles badly.

**Leaky bucket — a similarly-named but functionally different idea, worth not confusing with token bucket.** A leaky bucket is a request *queue* with a hole in the bottom: requests go in at whatever (possibly bursty) rate they arrive, and they leak out — get forwarded/processed — at a strictly constant rate, no matter how bursty the input was; if the queue fills up faster than it drains, the overflow is dropped. The distinction that matters: token bucket is an **admission-control budget** — it lets a burst through immediately, all at once, as long as tokens are banked, and only the long-run average is capped. Leaky bucket is a **traffic shaper** — its whole point is that the output rate is always smooth and constant, and it deliberately never lets a burst pass through unshaped, even if the burst was perfectly legitimate.

**Which algorithm actually belongs at which of this design's three rate-limiting layers.** This distinction isn't just trivia — it changes which algorithm is the right fit for each layer in §12's three-level list. The **per-provider** check wants token bucket, exactly as described: its job is to admit legitimate bursts (a worker pool catching up after being idle) while still capping the provider's true sustained ceiling, so needlessly rejecting compliant bursts is a real cost worth avoiding. The **per-worker/in-process** check, by contrast, has a different job — its stated purpose is to *smooth* a single worker's own burst before it ever reaches the shared check, i.e., to turn "this worker just dequeued 500 messages at once" into a steady trickle rather than to decide whether to admit the burst. That's precisely what leaky bucket is for, not token bucket: a per-worker limiter that used a token bucket would simply let the worker's own burst straight through (up to its local capacity) rather than shaping it, which defeats the stated purpose of existing at that layer in the first place. So the two layers should deliberately run two different algorithms — leaky bucket locally, for shaping; token bucket at the shared provider boundary, for burst-tolerant admission against a real external ceiling — rather than reusing the same mechanism at both layers out of convenience.

| Algorithm | Boundary-safe? | Memory per key | Handles idle-then-burst without rejecting? |
|---|---|---|---|
| Fixed window counter | No — under-enforces at boundaries | O(1) | No — rejects needlessly |
| Sliding window log | Yes — exact | O(requests in window) | Yes, up to the true limit |
| Sliding window counter | Approximately | O(1) | Approximately |
| Token bucket | Yes — no fixed boundary | O(1) | Yes, up to bucket capacity |
| Leaky bucket | Yes — no fixed boundary | O(1) (plus queue) | No, by design — it shapes to constant output instead |

A well-behaved rate limiter also surfaces *why* a send was delayed or rejected (throttle metadata) back through the pipeline rather than failing silently, since a request that's throttled (retry later) and a request that's genuinely invalid (bad token, bad address) need very different downstream handling (§13).

## 13. Retry Logic and Failure Handling — Deep Dive

**Retry policy**: a failed send (network error, provider 5xx, throttle response) is retried with **exponential backoff plus jitter** (randomizing the exact delay within a growing window) rather than a fixed interval — jitter specifically prevents a large batch of simultaneously-failed sends from all retrying at exactly the same moment and creating a synchronized retry storm that immediately re-triggers the same overload. Retries are bounded by both a maximum attempt count and a maximum total retry duration; exceeding either moves the notification to a **dead-letter queue** rather than retrying indefinitely.

**Why two independent bounds, not just one.** Attempt count and total duration catch two different failure shapes, and each is incomplete on its own. If only attempt count were bounded — say, 5 attempts, no time limit — exponential backoff's growing delays (1s, 2s, 4s, 8s, 16s, often with a ceiling on the maximum delay between tries) mean those 5 attempts could, in the worst case, stretch out well past the point where the notification still matters: a password-reset or fraud-alert notification that's still "pending retry" 20 minutes after it was triggered has already lost most of its value, whether or not the 5th attempt eventually succeeds. So time-sensitive tiers specifically need a tight ceiling on *elapsed time*, independent of how many attempts that allows. Conversely, if only total duration were bounded — say, "keep retrying for up to 10 minutes," no attempt-count limit — a misconfigured or buggy backoff schedule (delays staying small instead of growing, or jitter miscomputed) could fire dozens or hundreds of attempts at the provider inside that window, which defeats the entire reason backoff exists: to back off, not to let a failing notification turn into a hammer against an already-struggling provider. Bounding only one side leaves the other side's failure mode uncovered, so both apply together, and whichever one is hit first ends the retry loop and dead-letters the message.

**Worked example.** A critical-tier notification uses backoff starting at 1s, doubling each attempt, capped at 30s between tries, with a 5-attempt limit and a 3-minute total-duration limit. Attempts land at roughly 0s, 1s, 3s, 7s, and 15s — well inside 3 minutes, so attempt count is the binding constraint here, and the notification dead-letters after the 5th failure at around 15 seconds in. A promotional-tier notification might use the same backoff shape but a 15-attempt limit and a 10-minute total-duration cap; with a 30-second delay ceiling, 15 attempts could take over 7 minutes to exhaust, so in that case duration — not attempt count — is the one more likely to bind first if the provider stays down that long. The two tiers don't just differ in SLA target; they genuinely want different values for *both* bounds, which is exactly why this is two independent, tier-configurable knobs rather than one global constant.

**This also has to stay coordinated with §9's fallback timer, not set independently of it.** If a single channel's retry window is allowed to run longer than the fallback orchestrator's own per-channel deadline (§9's `now + M minutes`), channel 1 could still be silently retrying well after the orchestrator has already dispatched channel 2 — producing exactly the kind of overlapping-attempt race the compare-and-set in §9 exists to close. In practice this means a channel's max total retry duration should be set to finish (success, non-retryable failure, or dead-letter) *before* that channel's fallback deadline fires, so the fallback decision is always made with a settled outcome rather than racing an in-flight retry.

**Distinguishing retryable from non-retryable failures matters.** A throttle response or a transient network error should be retried; an invalid device token, a hard-bounced email address, or a malformed phone number should not be — retrying a permanently-invalid destination just wastes retry budget and delays the dead-letter classification that should be triggering a contact-record invalidation (§8) instead.

**Provider outage isolation (circuit breaker).** If a specific third-party provider starts failing at a high rate (its own outage, not a transient blip), the worker fleet trips a circuit breaker for that provider — stopping new dispatch attempts to it for a cooldown period rather than continuing to retry against a provider that's clearly down, which would just burn through retry budgets across every affected notification simultaneously. Where a fallback channel exists (§9), a tripped circuit can also trigger an earlier-than-usual fallback to the next channel in the chain.

## 14. Delivery Tracking and Analytics

Providers report outcomes asynchronously via webhooks/callbacks: a push provider reports delivery success/failure per token; an email provider reports delivery, bounce, complaint, open (via a tracking pixel), and click (via redirect links); an SMS provider reports delivery receipts and inbound `STOP` replies. These callbacks update the Delivery Record (§2) and feed back into two places: the analytics store (aggregate delivery/open/click rates, per campaign and per channel) and the cross-channel fallback logic (§9), which is watching for exactly these signals to decide whether to escalate to the next channel.

## 15. Storage Layer — Deep Dive

Following the same tiering logic as any high-throughput system: **preference and contact records** (§2) are small, read on essentially every notification attempt, and need fast point lookups by user — these sit in a low-latency store built for high read QPS. **Delivery records** are write-heavy (one write per attempt, plus subsequent status updates) and read mostly by analytics and the fallback logic, not by the hot send path — these can live in a store optimized for high write throughput with eventual read consistency being perfectly acceptable. **Templates and campaign definitions** are small, read-heavy, and change infrequently — a simple, heavily-cached store suffices. Keeping these three concerns in separate stores means a burst of delivery-status writes from a large campaign never contends with the latency-sensitive preference lookups a transactional notification needs on its own fast path.

### 15.1 Preference and Contact Records — Access Pattern and Technology Fit

Every single notification attempt — one-off or campaign-sourced — does a point lookup against this store during the preference check (§5), which at the peak throughput worked out in §16 (tens of thousands of lookups/sec, driven higher still during a large campaign's fan-out) means this store's read latency sits directly on the critical path of every send. The access pattern is simple and uniform: fetch by `user_id` (and typically `channel` + `category` as part of the key or a secondary index), never a scan or a range query, which is exactly the shape a key-value or wide-column store is built for (DynamoDB, Cassandra, or an equivalent) rather than a general-purpose relational database tuned for joins it will never need to do here.

**Why `channel` and `category` need to be part of the key at all.** A user doesn't have one preference row — per §2's data model, they have one per `(channel, category)` combination (push+shipping, push+marketing, email+marketing, sms+security, and so on), easily a few dozen rows per user. The preference check never needs "all of this user's preferences," only the answer for one exact combination, so `user_id` alone is too coarse a key for the hot path. The precise fix is a composite key — `user_id` as the partition key, `channel#category` as the sort/clustering key — so a single `GetItem`-style call with both components specified returns exactly the one row needed, with no in-memory filtering and no extra data transferred. A secondary index on `channel`/`category` serves a different, much rarer query shape instead — "every user opted into promotional SMS," independent of any specific `user_id` — which is a bulk/analytics access pattern, not something the per-notification send path ever needs to do.

Because reads vastly outnumber writes for this record type — a user's preferences and device tokens change rarely compared to how often they're read — a **cache-aside layer** (Redis or Memcached) in front of the durable store is the natural fit: a notification attempt checks the cache first, falls back to the durable store on a miss, and populates the cache on the way back. This is where the storage choice has to be precise rather than just "add a cache," covered in §15.4 below.

### 15.2 Delivery Records — Access Pattern and Technology Fit

This record type inverts the read/write ratio of §15.1: one write per attempt (§2), plus additional writes as provider webhooks report delivery, bounce, open, or click events (§14) — a single logical notification can accumulate several writes to its delivery record over its lifetime. Reads come from two very different consumers: the analytics pipeline (§18), doing large aggregate scans well after the fact, and the cross-channel fallback orchestrator (§9), doing a single-row read/conditional-write at the moment a timer fires. A wide-column store (Cassandra, Bigtable-style) suited for high write throughput is the right fit for the write side; the two different read patterns are addressed separately in §15.4.

**Partitioning this store needs the same care §7.1 gave the topic structure.** If the partition key were something naive like a plain timestamp, a 50-million-record campaign landing within a short window (§10, §16) would concentrate all of that write volume onto whichever partition owns "right now" — a hot partition, the storage-layer equivalent of the head-of-line-blocking problem the topic split was built to avoid. The fix is the same shape as partitioning the topics themselves (§7.1): key on something high-cardinality and well-distributed, such as a hash of `notification_id` (or `user_id`), with time as a secondary/clustering component rather than the partition key itself — spreading a campaign's write burst evenly across many partitions instead of funneling it through one.

### 15.3 Templates and Campaign Definitions — Access Pattern and Technology Fit

This is the smallest and simplest of the three: a given platform has thousands, not billions, of template versions and campaign definitions, each read very frequently (once per render, on the hot path, §6) but written rarely (only when a template is authored or a campaign is scheduled). Because templates are **versioned** (§6) and a given version's content never changes once published, this is a near-ideal caching case — there's no invalidation problem to solve, since a cache entry keyed by `template_id + version` is immutable for as long as it's cached. In practice this usually means the full working set of active templates is simply held in-memory (or in a fast, widely-replicated distributed cache) across every rendering worker, with the durable store behind it — commonly just a conventional relational database, since this is one of the few places in the system where the simplicity of joins and ACID transactions actually helps (e.g., atomically publishing a new template version alongside the campaign definitions that reference it) and the throughput demands never come close to needing anything more specialized.

### 15.4 A Consistency Requirement That Cuts Across the Tiering

The three-way split above is necessary but not sufficient — two of the three stores have a consistency requirement that's easy to miss if "eventually consistent is fine for this tier" is applied too broadly.

**Preference writes need read-after-write consistency specifically for opt-outs, not just low latency for reads.** §17 requires that an unsubscribe be honored "immediately and durably," and §8 requires a `STOP` SMS reply to be treated as an "authoritative, immediate opt-out." If the cache-aside layer in §15.1 relies on a time-based TTL to stay fresh, there's a real window — however short — where a user who just opted out could still have their *stale, cached* "opted in" preference record served to a notification attempt that lands moments later, sending them a message they explicitly just asked to stop. The fix is to make the opt-out write path **invalidate (or synchronously update) the cache entry at write time**, not rely on the TTL to eventually expire it — a write-through or explicit-invalidation pattern, rather than pure cache-aside with time-based expiry, specifically for this one field. Every other preference field can tolerate the TTL-based staleness a plain cache-aside design implies; the opt-out flag cannot.

**The delivery record store needs a strongly-consistent conditional write for exactly one operation, layered on top of an otherwise eventually-consistent store.** §15.2 is right that analytics reads tolerate eventual consistency — a dashboard being a few seconds stale doesn't matter. But §9's fallback orchestrator performs a **compare-and-set** against a logical notification's current status ("mark as moved to channel 2, but only if status is still pending for channel 1") to close the exact race condition described there, and a compare-and-set is only meaningful against a strongly consistent view of that one row — if the orchestrator's read could return a stale replica, the whole race-prevention mechanism silently stops working and the double-send §9 was built to prevent becomes possible again. The resolution isn't to make the entire delivery-record store strongly consistent (that would undo the write-throughput benefit §15.2 exists for); it's that the chosen store needs to support strongly consistent, single-row conditional writes for this specific status field even while bulk reads elsewhere in the same store remain eventually consistent — a capability stores like DynamoDB (conditional writes against a strongly consistent read) and Cassandra (lightweight transactions, via Paxos, scoped to a single partition) both offer without requiring the whole system to pay for strong consistency on every read.

### 15.5 Retention and Cold-Storage Tiering

§16's capacity estimate puts delivery records at a multi-terabyte-per-month write volume — a figure that makes "keep everything in the hot, write-optimized store forever" both expensive and unnecessary, since the only consumers that need low-latency access (the fallback orchestrator, §9) only ever care about a notification's *current*, recent status, not its full history from months ago. The practical pattern is a rolling retention window: delivery records stay in the hot store for a bounded recent period (long enough to cover the realistic fallback/retry/dead-letter lifecycle of any notification, plus a buffer for near-real-time analytics and support/debugging lookups — commonly on the order of 30–90 days), after which they're rolled into cheaper, columnar cold storage (object storage plus a format like Parquet) for long-term analytics, trend reporting, and compliance retention, while the hot store prunes the aged-out partitions to keep its own working set — and therefore its p99 write and CAS latency — from degrading as volume accumulates indefinitely.

## 16. Back-of-the-Envelope Capacity Estimate

Assume a platform with 100 million users, generating an average of 5 notification-worthy events per user per day, plus periodic large campaigns:

- **Average throughput**: `100,000,000 × 5 / 86,400 ≈ 5,787` notifications/second sustained average.
- **Peak throughput**: a major campaign or a viral event can spike this an order of magnitude above average — provisioning for tens of millions of notifications per hour during a peak event (on the order of several thousand to tens of thousands per second sustained for the campaign's duration) is a realistic target, well above the daily average.
- **Fan-out batch sizing**: a 50-million-user campaign, batched at 20,000 users per batch (§10), produces 2,500 batch messages — a manageable number for the orchestrator to publish and rate-control, versus 50 million individual messages hitting the queue at once.
- **Delivery record volume**: at ~6 billion notification-events per day at moderate scale (5 events × ~1B total sends across users and campaigns, rough order of magnitude), delivery records alone are a multi-terabyte-per-month write workload, reinforcing why they're isolated in their own write-optimized store (§15) rather than sharing infrastructure with the low-latency preference store.

These numbers directly motivate the architecture above: no single queue or worker pool could sustain peak campaign traffic without starving transactional sends, hence per-priority topics (§4/§7); no single store could serve both high-QPS preference lookups and high-volume delivery-status writes efficiently, hence storage tiering (§15).

## 17. Security and Compliance

- **PII handling**: phone numbers, email addresses, and device tokens are personal data — encrypted at rest and in transit, with access scoped to the services that genuinely need them (channel workers, not arbitrary internal services, which should only ever see a `user_id`).
- **Regulatory compliance**: unsubscribe/opt-out must be honored immediately and durably (CAN-SPAM for email, equivalent SMS regulations requiring `STOP` keyword support, GDPR-style consent requirements for EU users) — this isn't just a channel worker's job (§8) but a hard requirement enforced at the preference-check layer (§5) so no send path can bypass it.
- **Content and rate abuse prevention**: the same rate-limiting infrastructure (§12) that protects providers from being overwhelmed also limits how the system itself could be misused to spam users, intentionally or via a bug in an upstream calling service.

## 18. Monitoring and Metrics

Key operational signals: delivery rate and latency per channel and priority tier (a critical notification's p99 latency is watched far more tightly than a promotional one's); provider-specific error rates (a rising error rate against one provider, isolated from the others, is the trigger for the circuit breaker in §13 and an operational alert); queue depth per topic (a growing backlog on the transactional topic is a much more urgent signal than the same growth on the promotional topic); frequency-cap and quiet-hour suppression rates (tracking how much traffic is being deferred/dropped, both for capacity planning and for product visibility into whether caps are tuned sensibly); and bounce/complaint rates per sending domain (an early warning for email deliverability degradation before it becomes a full blacklisting event).

## 19. Summary Table

| Goal / Problem | Technique |
|---|---|
| Decoupling triggering from sending | Durable message queue between orchestration and channel workers |
| Different channels, different failure modes | Separate worker pools per channel (push/SMS/email/in-app) |
| Avoiding notification fatigue | Per-user, per-category frequency capping + quiet hours |
| Personalized, localized content | Template engine rendered per channel, per locale |
| Not starving urgent sends behind bulk campaigns | Priority-tiered topics with independent consumer scaling |
| Broadcasting to millions of users | Campaign orchestrator + cursor-paginated batching + fan-out |
| No duplicate sends under at-least-once delivery | Idempotency key + dedup cache at the dispatch boundary |
| Protecting users, workers, and providers from overload | Rate limiting at user, worker, and provider levels (token bucket) |
| Surviving transient failures | Exponential backoff with jitter, bounded retries, DLQ |
| Isolating a failing third-party provider | Circuit breaker per provider + cross-channel fallback |
| Cleaning up dead push tokens / bounced addresses | Provider feedback/webhook processing → contact record invalidation |
| Serving both low-latency lookups and high-volume writes | Storage tiering: preference store, delivery-record store, template store |

## 20. Interviewer's Guide: Follow-Up Questions

**On the core architecture:**
- Why put a queue between the API and the channel workers instead of sending synchronously from the API request? What specifically would break under a traffic spike if you didn't?
- A notification specifies both push and email as acceptable channels. Walk through exactly what determines which one(s) actually get used, and in what order.

**On preferences, quiet hours, and frequency capping:**
- A user in Tokyo has quiet hours set 10pm–7am. The notification service's servers run on UTC. Where exactly does timezone conversion need to happen, and what breaks if it's done in the wrong place?
- Should a critical security alert respect frequency capping? Justify the answer either way.
- How would you implement the frequency-cap counter so that a burst of concurrent notification attempts for the same user can't all "pass" the check simultaneously and blow past the cap?

**On fan-out and broadcast:**
- Why cursor-based pagination instead of offset-based pagination for the campaign orchestrator's user query? What actually goes wrong with offset-based pagination at 50 million rows?
- If the orchestrator publishes batches faster than the channel workers can consume them, what happens, and what should happen?

**On idempotency and retries:**
- Where does the idempotency key actually come from, and why does it matter that it's generated by the *caller* rather than by the notification system itself?
- Design the interaction between retry logic and idempotency precisely: a send fails, gets retried, and *this* retry attempt succeeds right as the *original* attempt's delayed provider response also comes back reporting success. Does the user get two notifications? Why or why not?
- Why exponential backoff with jitter specifically, rather than plain exponential backoff? What failure mode does jitter alone fix?

**On channel-specific mechanics:**
- What's the concrete failure mode of a backend that never processes APNs/FCM invalid-token feedback? How would you notice this happening in production before deliverability metrics visibly degrade?
- Why does a hard email bounce need to immediately and durably suppress future sends, rather than just being logged and retried later?

**On failure isolation:**
- One of three notification channels' third-party provider is down. Walk through, precisely, what happens to notifications queued for that channel, and what should *not* happen to the other two channels.
- What's the difference between a rate-limit rejection and a circuit-breaker rejection, and why does a channel worker need to treat them differently?

**Concepts worth raising even if the candidate doesn't bring them up:**
- **At-least-once vs. exactly-once**: ask the candidate to justify *not* building exactly-once delivery — a strong candidate explains the idempotency-key trade-off explicitly rather than assuming exactly-once is obviously better.
- **Notification storms**: what stops a bug in an upstream service from triggering the same notification thousands of times for one user in a loop? (Frequency capping is the backstop here, not just a UX nicety.)
- **Cross-service dependency risk**: the notification system depends on the user-preference store being available; what happens to a send if that store is temporarily unreachable — fail open (send anyway) or fail closed (drop it)? Both have real consequences worth naming.
- **Testing deliverability claims**: how would you actually verify quiet-hour and frequency-cap logic is correct across timezones and DST transitions, rather than trusting the code review?
- **Analytics feedback loop**: delivery/open/click data isn't just for dashboards — should it feed back into future frequency-cap or channel-preference decisions automatically (e.g., a user who never opens push notifications getting deprioritized for push over time)? A candidate who raises this is thinking about the system as a product, not just a pipe.

## 21. A Critical Second Look: Gaps Worth Naming Out Loud

A few things the design above doesn't fully resolve — real trade-offs and open edges, not mistakes, but exactly the kind of thing worth surfacing unprompted in an interview. Each is stated as a problem first, then as a concrete fix, the same way the §12 provider-rate-limit gap was handled above.

### 21.1 The preference store is a shared resource the topic isolation doesn't protect

**The problem.** §7.1's whole argument is that a promotional flood can't delay a critical send because they never share a topic, a consumer group, or (per the §12 fix) a provider rate-limit budget. But every one of those up-to-50-million campaign candidate requests still has to pass the preference check (§5) *before* any of that isolation kicks in — and that check is a point lookup against the same preference store a transactional notification's preference check also depends on (§15 separates preference storage from delivery-record storage, but doesn't further separate "preference reads driven by a campaign" from "preference reads driven by one-off traffic"). Concretely: if batch-expansion workers (§10) can produce and preference-check candidates fast enough to clear a 50-million-user campaign in, say, 20 minutes, that's roughly 42,000 additional preference-check reads/sec landing on the exact same store a transactional notification's own preference check depends on — several times the platform's steady-state average from §16 — for the campaign's entire duration. The noisy-neighbor problem resurfaces one layer earlier than the provider rate limit does, on a resource none of the queue-level isolation ever touches.

**The fix.** Two complementary options, not mutually exclusive:
- **Rate-limit the read, not just the write.** Cap how fast batch-expansion workers are allowed to issue preference-check reads, independent of how fast they could otherwise produce candidate requests — the same per-worker-smoothing idea from §12.1's leaky-bucket discussion, just protecting a shared store instead of a shared provider connection. The campaign simply takes longer to fully expand; nothing about correctness depends on expanding it as fast as possible.
- **Give campaign-driven reads their own replica pool.** Most stores suited to §15.1's access pattern (DynamoDB, Cassandra) support read replica fan-out; routing batch-expansion's reads to a dedicated replica set, separate from the replica the single-notification hot path reads from, means a campaign's read volume can never add latency to a transactional lookup even if that volume is large — the two paths simply aren't contending for the same physical resource anymore.

### 21.2 Fallback chains need an explicit policy for promotional traffic, not an implicit one

**The problem.** §9 describes fallback as something a request "specifies" — an ordered channel preference the *caller* configures — which technically means a marketing campaign simply isn't required to configure a fallback chain. But the design never says this out loud, and the consequence of getting it wrong compounds across two different sections at once. If a campaign *did* request push-then-SMS fallback, a low push-open-rate campaign (common and expected for promotional sends — plenty of recipients simply won't have their phone in hand) would silently balloon into millions of fallback SMS messages, each with Twilio's real per-message cost. Worse, that fallback traffic lands on `sms-critical`'s and `sms-transactional`'s shared provider resource exactly as described in §21's predecessor, §12: a wave of promotional-triggered fallback SMS is itself a new source of noisy-neighbor pressure on the very provider rate-limit budget §12 just finished protecting — a misconfigured fallback chain could undo that fix from an entirely different angle.

**The fix.**
- **Restrict fallback eligibility by construction, not by convention.** The Notification Request schema (§2) should treat `priority` as gating which requests are even allowed to carry an ordered, multi-channel fallback chain — the Notification API rejects or silently collapses a fallback chain on a promotional-category request to a single channel, rather than trusting every calling service to never misconfigure one.
- **A budget-based circuit breaker as a backstop.** Even for legitimately-configured fallback on critical/transactional traffic, cap how many fallback escalations a single campaign or batch can trigger within a time window before an operator is paged — defense in depth for the case where the schema-level restriction above is bypassed upstream or simply doesn't exist yet in an older integration.

### 21.3 The dead-letter queue is a destination, not a resolved problem

**The problem.** §13 moves a notification to a DLQ once retries are exhausted, and that's the right call to stop burning retry budget — but the design doesn't say what happens to a message *after* it lands there. Picture a provider outage that lasts longer than the max total retry duration (§13) for every notification affected: an entire wave of notifications dead-letters during the outage window, even though nothing was actually wrong with any of them individually — the provider was simply down, and would have accepted every one of them a few minutes later. Without a reprocessing story, "bounded retries plus a DLQ" quietly becomes "give up after N attempts and lose it," which is a materially different guarantee than it sounds like at first glance — tolerable for promotional traffic, not for a payment-failure alert.

**The fix.**
- **Monitor DLQ depth and composition, not just queue depth.** Extend §18 to alert on DLQ growth rate per channel/priority tier specifically (a DLQ filling up for `push-critical` is a far more urgent page than for `push-promotional`), and tag each dead-lettered message with its terminal failure reason so "this will never succeed" (invalid token, bad address) is distinguishable at a glance from "this timed out during an outage and might well succeed now."
- **Add a replay path gated on failure reason.** Once a tripped circuit breaker (§13) closes again after an outage, a DLQ reprocessor can selectively replay only the messages whose terminal reason was transient/retryable — giving them a fresh retry budget now that the underlying cause is resolved — while never touching messages that failed for a genuinely non-retryable reason, which would just waste the replay pass.
- **Give the DLQ itself a relevance TTL.** A dead-lettered critical notification still sitting unprocessed after, say, 24 hours is usually no longer actionable — the order already shipped some other way, the password was already reset through a different channel — so the DLQ needs its own policy for when a message stops being worth replaying at all and is archived or discarded instead of accumulating indefinitely.

### 21.4 "Right to erasure" gets harder, not easier, once storage is tiered

**The problem.** §17 cites GDPR-style consent requirements, and GDPR also grants a "right to erasure" — a user can request that their personal data be deleted. §15's storage tiering, while solving throughput, makes this materially harder than it would be with one single store: a user's data is now scattered across the preference/contact store and its cache, the hot delivery-record store, *and*, per §15.5, cold-storage archives written in a columnar format specifically for analytical efficiency. That last tier is the problem — object storage holding Parquet-style files is immutable by design; finding and removing one user's rows out of a multi-terabyte archive means rewriting the files that contain them, which doesn't scale to doing it on demand for every erasure request a large platform receives.

**The fix.**
- **Crypto-shredding for the cold tier.** Encrypt each user's personally-identifying fields (or their whole record) with a per-user data-encryption key, itself wrapped by a master key, before it's written to cold storage. "Erasing" a user's data on request then means destroying just that one per-user key — the ciphertext can physically remain in the archive forever (preserving aggregate, de-identified analytics and whatever legally-required retention applies) but becomes permanently unrecoverable, satisfying the erasure requirement without ever touching or rewriting the archive file itself. This is the standard real-world answer to "how do you delete one row from an immutable analytical store."
- **Ordinary tombstone-and-compact for the hot tiers.** The preference, contact, and hot delivery-record stores are already built for per-row mutation (§15.1, §15.2), so an erasure request there is a straightforward delete, or a tombstone that clears on the store's normal compaction cycle — no new mechanism needed.
- **One erasure-orchestration record per request.** Because an erasure here is several independent operations across different systems (hot-store deletes, cache invalidation, cold-storage key destruction) rather than one atomic action, GDPR's "without undue delay" expectation needs an auditable record tracking which of those operations have actually confirmed completion for a given request — otherwise "we think we erased everything" isn't something the system can actually prove if asked.

## References

1. Notification system design overview and interview scope: [How to Design a Notification System: A Complete Guide 2026](https://www.systemdesignhandbook.com/guides/design-a-notification-system/)
2. Architecture pattern and scale requirements: [Notification System Design Interview: The Complete Walkthrough](https://spacecomplexity.ai/blog/notification-system-design-interview)
3. Rate limiting (token bucket, multi-level enforcement): [Designing Scalable Rate Limiting Systems: Algorithms, Architecture, and Distributed Solutions](https://arxiv.org/pdf/2602.11741)
4. Retry strategy and idempotency patterns: [Low Level Design — Scalable Notification System](https://vishalsheth4.medium.com/ldesign-a-scalable-notification-system-3c77b5314304)
5. Push notification device token lifecycle and fan-out architecture: [Scaling Push Notifications to 50 Million Devices: The Architecture Behind Every Alert](https://designgurus.substack.com/p/push-notification-architecture-apns)
6. Preference management, quiet hours, and frequency capping: [Notification System Architecture: Channels, Fan-Out, and Delivery at Scale](https://codelit.io/blog/notification-system-architecture)
