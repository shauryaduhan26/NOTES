# The Outbox Pattern: A Detailed Guide

## Overview

The outbox pattern lets a service update its database and publish an event reliably, without a distributed transaction. Instead of sending the event to the broker directly, the service writes it to an `outbox` table in the same database transaction as its business data. A separate relay process later reads the table and publishes to the broker.

The result: the state change and the event are committed atomically (both or neither), and event delivery becomes an asynchronous, retryable step that cannot silently lose messages. The price is at-least-once delivery, so consumers must tolerate duplicates.

## The problem: dual writes

A service often must change its own state and tell other services about it, for example saving an order and emitting `OrderCreated`. The database and the message broker are two separate systems that cannot share one transaction, so any ordering of the two writes can fail:

| Approach | Failure | Outcome |
| --- | --- | --- |
| Commit DB, then publish | Process crashes between the two steps | Order exists, event never sent; downstream systems never learn of it |
| Publish, then commit DB | DB commit fails or rolls back | Event sent for an order that does not exist |
| Publish inside the DB transaction | Broker acks but commit later fails (or the reverse) | Same inconsistency, plus the broker call holds the transaction open |

A distributed transaction (two-phase commit / XA) would close the gap, but most modern brokers (Kafka, SNS/SQS, many cloud services) do not support it. Where it is supported it adds latency, couples availability of both systems, and is operationally fragile. The outbox avoids it by using only one transactional resource: the service's own database.

## How the pattern works

The pattern has three moving parts: an atomic write, a relay, and idempotent consumers.

1. **Atomic write.** In one local transaction, the service updates its business tables and inserts a row describing the event into the outbox table. Commit applies both; rollback discards both.
2. **Relay.** A separate process (a poller or a change-data-capture connector) reads new outbox rows and publishes each to the broker.
3. **Consume.** Downstream services receive the event. Because the relay may publish a row more than once, they de-duplicate by event ID.

```
┌─ One local DB transaction ─────────┐
│  Business table (orders)           │
│  Outbox table (event row)  ────────┼──▶ Relay ──▶ Broker ──▶ Consumers
└────────────────────────────────────┘   (poller    (Kafka,     (inbox
                                          or CDC)    etc.)       de-dup)
```

Only the left box is transactional; everything to its right is asynchronous and retryable.

### What "commit" means

A transaction is a group of operations (inserts, updates) that the database treats as one unit. While it runs, its changes are private: other transactions and queries cannot see them. It ends in one of two ways:

- **COMMIT:** the changes become permanent and visible to all other readers, all at once.
- **ROLLBACK:** the changes are discarded as if they never happened.

This is why the outbox's atomic write works. The business row and the outbox row are inserted in one transaction, so commit makes both visible together and rollback discards both. It also explains the ordering pitfall in the polling section: a row inserted by a transaction is invisible to the relay until that transaction commits, and the database transaction log also records changes at commit time.

```sql
BEGIN;                                  -- start transaction
INSERT INTO orders ...;                 -- private to this transaction
INSERT INTO outbox ...;                 -- private to this transaction
COMMIT;                                 -- now both are permanent and visible
```

### The outbox table

The table must live in the same database as the business data, so one transaction can cover both. A typical shape:

```sql
CREATE TABLE outbox (
  id             UUID PRIMARY KEY,         -- also the idempotency key for consumers
  aggregate_type VARCHAR(100) NOT NULL,    -- e.g. 'Order' (maps to topic/routing)
  aggregate_id   VARCHAR(100) NOT NULL,    -- message key; preserves per-entity order
  event_type     VARCHAR(100) NOT NULL,    -- e.g. 'OrderCreated'
  payload        JSONB        NOT NULL,    -- event body
  created_at     TIMESTAMP    NOT NULL DEFAULT now(),
  published_at   TIMESTAMP    NULL        -- polling variant only
);
CREATE INDEX outbox_unpublished ON outbox (created_at) WHERE published_at IS NULL;
```

Store the full event as a snapshot at the moment of the change rather than a pointer to be resolved later, since the underlying row may have changed again by the time the relay runs.

## Relay strategies

There are two common ways to move rows from the outbox to the broker. CDC is the more robust choice at scale; polling is the simpler place to start.

| | Polling publisher | Log tailing (CDC) |
| --- | --- | --- |
| How it works | Job queries unpublished rows on an interval, publishes, then marks or deletes them | Connector (for example Debezium) reads the DB write-ahead log / binlog and streams outbox inserts |
| Latency | Poll interval (typically hundreds of ms to seconds) | Near real time |
| DB load | Repeated queries against the table | Minimal; reads the log |
| Ordering | Needs care: auto-increment IDs can commit out of order, so "read everything after last ID" may skip rows | Commit order, as recorded in the log |
| Infrastructure | None beyond the service | Connector, Kafka Connect (or equivalent), log retention config |
| Cleanup | Mark `published_at` or delete after publish | Row can be deleted right after insert; the log already captured it |
| Multiple relay instances | Must coordinate, for example `SELECT ... FOR UPDATE SKIP LOCKED` or a lease | Handled by the connector's offset tracking |
| Best for | Low volume, small teams, no CDC platform | High volume, low-latency, strict ordering needs |

### Polling publisher in detail

A scheduled job loops: select unpublished rows, publish each, mark it published (or delete it), commit.

```sql
SELECT * FROM outbox
WHERE published_at IS NULL
ORDER BY created_at
LIMIT 100
FOR UPDATE SKIP LOCKED;
```

```
 Service ──1──▶ ┌─ Database ───────────┐
                │ orders (business)    │
                │ outbox (event rows) ◀┼──2, 4──▶ Relay ──3──▶ Broker
                └──────────────────────┘        (timer-driven)
```

1. Service writes the business row and the outbox row in one transaction.
2. Relay runs `SELECT` on the outbox table for rows where `published_at` is null.
3. Relay publishes each row to the broker.
4. Relay marks the rows as published (`UPDATE`) or deletes them. This is its memory.

- **Latency is bounded by the poll interval.** Polling every 500 ms adds up to 500 ms. Polling faster costs more DB load, and empty polls still hit the database continuously.
- **Ordering pitfall.** Sequences and auto-increment IDs are allocated at insert time, not commit time. Transaction A can get ID 10 and commit after transaction B (ID 11). A relay that tracks "last ID seen = 11" skips ID 10 forever. Safe options: poll by `published_at IS NULL` instead of a high-water mark, accept a small lag window, or order by a commit-time source.
- **Scaling relays.** Several instances need `SKIP LOCKED` or a lease. Even then, per-key ordering can break if two instances take different events for the same `aggregate_id`; partition rows by key hash, with one worker per partition.
- **Write amplification.** Marking rows is an UPDATE per row. Deleting is cheaper but loses the audit trail.
- **Table bloat.** High churn means vacuum pressure (Postgres) and a need for frequent cleanup.
- **Upside.** No infrastructure beyond the service and its database, state you can inspect with a query, and full control of retries and batching in code.

### The ordering pitfall, step by step

A database assigns an ID when a row is inserted, but other transactions only see the row once it commits. A relay that tracks "the highest ID I have seen" can therefore skip a row permanently.

| Time | Transaction 1 | Transaction 2 | Relay (polls `WHERE id > last_seen`) |
| --- | --- | --- | --- |
| t1 | Inserts outbox row, gets id 10 (uncommitted, invisible to others) | | |
| t2 | | Inserts outbox row, gets id 11, commits | |
| t3 | | | Polls `id > 9`: sees only 11. Publishes it, saves `last_seen = 11` |
| t4 | Commits; row 10 is now visible | | |
| t5 | | | Polls `id > 11`: row 10 is never returned |

Event 10 is committed but never published, and nothing flags it as missing.

Fixes:

- **No high-water mark.** Poll `WHERE published_at IS NULL`; row 10 appears on the next poll because it is still unpublished.
- **Lag window.** Ignore rows newer than the longest expected transaction, so late commits are caught before the relay moves past them.
- **CDC.** The log is ordered by commit, so the problem does not exist.

Even with `published_at IS NULL`, events for the same entity normally stay in order, because two transactions changing the same aggregate usually block each other on a row lock and commit in sequence.

### CDC (log tailing) in detail

The database already writes every committed change to its transaction log (Postgres WAL via a logical replication slot, MySQL binlog). A connector such as Debezium reads that log and emits an event for each insert into the outbox table. The app only inserts rows. Because the log is the source, the row can be deleted immediately and the connector still sees the insert.

```
 Service ──1──▶ ┌─ Database ───────────┐
                │ orders (business)    │
                │ outbox (event rows)  │
                │        │ commit is logged
                │        ▼             │
                │ transaction log (WAL)├──2──▶ Connector ──3──▶ Broker
                └──────────────────────┘      (e.g. Debezium)
```

1. Service writes the business row and the outbox row in one transaction.
2. The database logs the commit itself; the connector reads new log entries.
3. Connector publishes to the broker, then saves its log position (offset). This is its memory.

The outbox table is never queried, so rows can be deleted right after insert.

- **Commit order.** The log is ordered by commit, so the out-of-order ID problem disappears.
- **Low latency.** Changes stream as they are committed, typically in milliseconds, with no polling load and no `published_at` updates.
- **Offset tracking is built in.** The connector stores its log position and resumes where it left off after a restart.
- **Declarative routing.** Debezium's outbox router maps `aggregate_type` to a topic and `aggregate_id` to the message key through configuration, not code.
- **More infrastructure.** You run and monitor the connector (usually on Kafka Connect), and upgrade it over time.
- **DB configuration.** You need logical replication enabled (Postgres `wal_level=logical`), a replication slot and the right permissions. Managed databases sometimes restrict this.
- **Replication slot risk (Postgres).** If the connector stops, the slot forces the DB to retain WAL and disk can fill up. Alert on connector lag and disk usage.
- **Still at-least-once.** If the connector crashes after publishing but before saving its offset, events repeat, so consumers must still de-duplicate.
- **Harder to customize.** Per-row retry or custom error handling is less direct than in your own code, and Debezium is most natural with Kafka.

### Why CDC does not have the ordering pitfall

The pitfall comes from using the ID as a stand-in for "newer". An ID reflects when a row was inserted, not when it became visible. CDC never uses the ID as a position marker.

The transaction log is one sequential record of changes. A transaction's changes enter it as part of committing, and every entry gets an increasing position (in Postgres, an LSN). Replaying the earlier example:

| Time | Transaction 1 | Transaction 2 | Log and connector |
| --- | --- | --- | --- |
| t1 | Inserts row id 10 (uncommitted) | | Nothing for the connector yet; uncommitted changes are held back |
| t2 | | Inserts row id 11, commits | Log position 100: T2's commit with row 11. Connector emits 11, saves offset 100 |
| t4 | Commits | | Log position 200: T1's commit with row 10. Connector emits 10, saves offset 200 |

The connector's offset is the log position (100, then 200), not the row ID. Row 10 committed later, so it sits at a later position, and the connector reaches it by reading forward. There is no step that says "I have seen up to ID 11, so ignore anything lower". Nothing can appear behind a position the connector has already passed.

Two caveats: events come out in commit order, which may differ from ID order (11 before 10 here, and that is correct). And CDC does not remove duplicates: if the connector crashes before saving its offset, it re-reads and re-publishes, so consumers still de-duplicate by event ID.

### Side by side on the hard parts

| Concern | Polling | CDC |
| --- | --- | --- |
| Skipping rows | Possible if you use an ID high-water mark | No; the log is complete and ordered |
| Duplicates | After a crash between publish and mark | After a crash between publish and offset commit |
| Backpressure | Rows pile up in the table | Log is retained (watch disk on Postgres slots) |
| Throughput ceiling | Limited by query and update rate | Much higher |
| Relay failure | Table grows; app unaffected | Log retention grows; app unaffected |

### Why duplicates happen in both

Publishing to the broker and recording "I already did this" are two steps in two different systems, so no crash-proof atomic step joins them. The only difference is where the record lives.

- **Polling:** the relay publishes row 42 and the broker acknowledges it. The relay crashes before it marks row 42 as published. On restart the row still looks unpublished, so it is sent again.
- **CDC:** the connector publishes the event for log position X. It crashes before saving "read up to X". On restart it resumes from its last saved position, re-reads the same log entries and publishes them again.

The fix is the same either way: consumers de-duplicate by event ID.

### Choosing a relay

Start with polling when volume is modest (say, under a few hundred events per second), latency in the hundreds of milliseconds is fine, and you do not already run Kafka Connect. Move to CDC when you need low latency, high throughput or strict ordering, or when you already operate a CDC platform. Keep the outbox table schema the same for both, so you can switch relays later without changing application code.

## Delivery guarantees, idempotency and ordering

The outbox guarantees at-least-once delivery, not exactly-once. The relay can publish an event and crash before recording that it did, so after restart the same event is published again. Every consumer must therefore be idempotent.

### Idempotent consumers (the inbox pattern)

Each event carries a unique ID (the outbox row's `id`). The consumer records processed IDs in an `inbox` or `processed_messages` table and skips any it has seen. For the guarantee to hold, the ID check and the consumer's own state change must commit in the same local transaction. Natural idempotency (for example "set status to SHIPPED") also works and needs no table.

```sql
BEGIN;
  INSERT INTO processed_messages (event_id) VALUES (:id)
    ON CONFLICT DO NOTHING;           -- 0 rows inserted => duplicate, skip
  -- apply business change only if the insert succeeded
COMMIT;
```

### Ordering

Use `aggregate_id` as the broker message key so all events for one entity land in the same partition and stay in order. There is no global order across aggregates. In the polling variant, ordering by an auto-increment ID is unsafe under concurrent writers because a lower ID can commit after a higher one; ordering by commit position (CDC) or a monotonically assigned sequence avoids this. If a consumer needs strict per-entity order, a failed event must block later events for that key rather than be skipped.

### Concurrent threads on the same aggregate

Two users or threads can process the same order at once. The database keeps the order of events consistent as long as both touch the same row:

- **Same row:** the second `UPDATE` blocks on the row lock until the first transaction commits (or, with optimistic locking on a `version` column, it fails the check and retries on the new state). The transactions run one after the other, so their outbox rows commit in the same order. Update the aggregate row before inserting the outbox row, so the lock decides the event order.
- **Different rows:** there is no conflict, so the transactions run in parallel and may commit in either order. That is fine because the operations are independent; consumers should not assume any other order.

Further protections: publish with `aggregate_id` as the message key so one order's events share a partition; include a per-aggregate `version` in each event so consumers can ignore an event older than the last one applied; and with polling, run one worker per key partition so two relay instances never publish the same order's events out of order.

## Failure scenarios

No single crash can lose an event or publish one for a rolled-back change. The worst case is a duplicate.

| Failure point | What happens | Result |
| --- | --- | --- |
| Service crashes before commit | Both the business row and the outbox row roll back | Nothing changed, nothing published |
| Service crashes after commit | Both rows are durable; the relay picks up the outbox row later | Event delivered, slightly late |
| Relay crashes before publishing | Row is still unpublished (or not yet read from the log) | Event published after restart |
| Relay crashes after publishing, before marking or recording its position | Row is processed again on restart | Duplicate event; consumers de-duplicate by ID |
| Broker unavailable | Relay retries; rows accumulate in the outbox | Delayed delivery; service writes are unaffected |
| Consumer fails mid-processing | Offset not committed; broker redelivers | Retried; idempotency prevents double effects |
| Poison message (always fails) | Can block a strictly ordered stream | Needs retry limit, dead-letter queue and alerting |

## Reference implementation and operations

A minimal Spring/JPA sketch with a polling relay. The same structure applies in any stack.

```java
// 1. Atomic write: business row + outbox row in ONE transaction
@Transactional
public void placeOrder(Order order) {
    orderRepo.save(order);
    outboxRepo.save(new OutboxEvent(
        UUID.randomUUID(), "Order", order.getId(),
        "OrderCreated", toJson(order)));
}

// 2. Polling relay (run as a single logical worker or use SKIP LOCKED)
@Scheduled(fixedDelay = 500)
@Transactional
public void publish() {
    List<OutboxEvent> batch = outboxRepo.lockNextBatch(100); // FOR UPDATE SKIP LOCKED
    for (OutboxEvent e : batch) {
        broker.send(topicFor(e), e.getAggregateId(), e.getPayload()); // key = aggregate_id
        e.markPublished();
    }
}
```

### Operational checklist

- **Monitor lag.** Alert on the age of the oldest unpublished row (or CDC connector lag). A growing backlog means the relay is stuck or the broker is down.
- **Clean up.** Delete or archive published rows on a schedule so the table and its index do not grow without bound.
- **Handle poison messages.** Set a retry limit, route failures to a dead-letter queue, and alert, so one bad row does not stall the stream.
- **Version events.** The outbox decouples write time from consume time, so old-format events may still be in flight after a deploy. Include an event type and schema version.
- **Size payloads.** Large payloads bloat the table and the log; keep events small or reference external storage.
- **One outbox per datastore.** The table must share a transaction with the data it describes. Services with several databases need an outbox in each.
- **Test the duplicate path.** Deliberately re-publish events in tests to confirm consumers are idempotent.

## Alternatives and when to use it

Use the outbox whenever "update my state and notify others" must be reliable: microservice event publishing, saga steps, and syncing caches or search indexes. Skip it when a missed or late notification is harmless, or when one datastore already acts as the event log.

| Alternative | How it avoids dual writes | Trade-off |
| --- | --- | --- |
| Event sourcing | The append-only event log is the source of truth; state is derived | Large modelling shift; hard to retrofit |
| Listen to yourself | Service publishes first, then updates its own state by consuming its own event | Eventual consistency for the service's own reads; read-your-writes is harder |
| CDC on business tables | Connector streams table changes directly, no outbox table | Couples consumers to your internal schema; events are row changes, not business events |
| Two-phase commit (XA) | Coordinator commits DB and broker together | Limited broker support, latency, availability coupling |
| Transactional broker writes | Some systems let the database and queue share a transaction | Ties you to one vendor or product |

### Event sourcing

Instead of storing current state and emitting events on the side, the service stores only an append-only sequence of events per aggregate; current state is rebuilt by replaying them. Appending an event is a single write, so there is nothing to keep in sync, and the same log feeds consumers.

- **Strengths:** a complete audit trail, the ability to rebuild state or new read models from history, and natural fit with CQRS.
- **Costs:** a different programming model, event schema evolution that lasts forever, snapshots for long-lived aggregates, and awkward ad-hoc queries without projections. Retrofitting it onto an existing relational service is a rewrite, not a patch.
- **Choose it when** history and auditability are core requirements, not just a way to avoid dual writes.

### Listen to yourself

The service publishes the event first, then consumes its own event from the broker and applies the state change in that handler. The broker becomes the single write; the database update follows from it.

- **Strengths:** no outbox table or relay, and one source of truth at the moment of the write.
- **Costs:** the service's own state lags behind the request, so the caller cannot read its own write immediately. Validation must happen before publishing, and a handler that keeps failing leaves the event accepted but the state unchanged.
- **Choose it when** eventual consistency is acceptable even for the service's own reads, for example fire-and-forget commands.

### CDC on business tables (no outbox table)

A connector such as Debezium streams row changes from the business tables straight to the broker. There is no extra table and no application code for publishing.

- **Strengths:** zero changes to the write path, low latency, and works on legacy code you cannot modify.
- **Costs:** consumers see row-level changes (insert/update/delete of internal columns), not intent such as `OrderShipped`. Renaming a column or splitting a table breaks consumers, so your internal schema becomes a public contract. A single business action touching several tables arrives as several unrelated messages.
- **Choose it when** you need to replicate data (for example into a search index or warehouse) rather than publish domain events. An outbox table with CDC gives the same plumbing with an explicit, stable event contract.

### Two-phase commit (XA)

A transaction coordinator asks both the database and the broker to prepare, then commits both, or aborts both.

- **Strengths:** true atomicity across two resources, with no duplicates by design.
- **Costs:** few modern brokers and cloud services support XA. It adds round-trips, holds locks across systems, and makes each system's availability a dependency of the other. A coordinator crash can leave transactions in doubt until it recovers.
- **Choose it when** you are inside a single application server stack that already supports it (for example JMS with a relational database) and the throughput is modest.

### Transactional broker writes

Some platforms let the broker and the datastore share one transaction. Examples include a database-backed queue in the same database as the business data, or a product that exposes both as one engine.

- **Strengths:** the same atomicity as an outbox, with less code, because the queue is transactional by construction.
- **Costs:** you are tied to that product, and the queue shares capacity and failure modes with the main database. Consumers outside that system still need a bridge, which is the relay again.
- **Choose it when** the database-backed queue already meets your throughput and integration needs.

### Choosing between them

| If you need... | Prefer |
| --- | --- |
| Reliable domain events from an existing relational service | Outbox (with CDC at scale) |
| Full history and replay as a core feature | Event sourcing |
| Replication of data to other stores, no domain semantics | CDC on business tables |
| Simplicity and tolerance of lagging own reads | Listen to yourself |
| Atomicity inside one supported stack | XA or transactional queue |

The outbox is the default choice when the broker has no transaction support and the team wants to keep a normal relational data model.
