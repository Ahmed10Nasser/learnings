# Transactional Outbox

Reliably publish messages/events by writing them to an "outbox" table in the **same
local transaction** as the business data change, then relaying them to a message
broker afterward — avoiding the dual-write problem.

## The problem it solves

A service often needs to do two things when handling a request:

1. Update its own database (e.g. create an `Order`).
2. Publish an event so other services react (e.g. `OrderCreated` to Kafka/RabbitMQ).

These live in two different systems, so there's no shared transaction. If you do them
as two separate writes ("dual write"), any crash between them leaves the system
inconsistent:

- DB commit succeeds, broker publish fails → other services never learn about the order.
- Broker publish succeeds, DB commit fails/rolls back → an event exists for data that
  doesn't.

You cannot make a database commit and a broker publish atomic without expensive and
often unavailable distributed transactions (2PC).

## How it works

1. In the **same DB transaction** as the business change, insert a row into an
   `outbox` table describing the event (aggregate id, event type, payload, timestamp).
   Because it's one local ACID transaction, the business change and the outbox record
   commit together or not at all.
2. A separate **message relay** process reads unpublished rows from the outbox and
   publishes them to the broker.
3. After a successful publish, the relay marks the row as sent (or deletes it).

```
┌── local transaction ──────────────┐
│  INSERT INTO orders (...)          │
│  INSERT INTO outbox (event, ...)   │
└────────────────────────────────────┘
                 │
                 ▼
        ┌─────────────────┐      publish      ┌──────────┐
        │  Message relay   │ ────────────────▶ │  Broker  │
        │ (poll or CDC)    │                    └──────────┘
        └─────────────────┘
```

### Two ways to run the relay

- **Polling publisher** — the relay periodically queries the outbox table for
  unsent rows and publishes them. Simple, but adds query load and polling latency.
- **Change Data Capture (CDC)** — a tool (e.g. Debezium) tails the database
  transaction log and streams outbox inserts to the broker. Lower latency and no
  polling load, but more infrastructure to operate.

## Key properties & trade-offs

- **At-least-once delivery**: a crash after publishing but before marking the row sent
  causes a re-publish. Consumers **must be idempotent** (dedupe on event id) or use
  the [Inbox pattern](inbox-pattern.md) on the receiving side.
- **Ordering**: publish in insertion order (e.g. by monotonic id) if consumers need
  ordering; partition-aware brokers may need a partition key per aggregate.
- **Latency**: polling introduces delay; tune interval vs. load. CDC is near-real-time.
- **Cleanup**: prune or archive sent rows so the outbox table doesn't grow unbounded.
- **Same database only**: the guarantee holds only because the business data and the
  outbox share one transactional database.

## When to use it

- Microservices that must publish domain events reliably alongside state changes.
- Any place you'd otherwise be tempted to "save to DB, then call the broker" as two
  separate steps.

## When it's overkill

- Single monolith with no cross-service events.
- Cases where losing an occasional notification is acceptable, or where the broker
  itself is your source of truth (event sourcing).

## References

- Chris Richardson, *Microservices Patterns* — Transactional Outbox pattern.
- microservices.io — [Pattern: Transactional outbox](https://microservices.io/patterns/data/transactional-outbox.html)
- Debezium docs — [Outbox Event Router](https://debezium.io/documentation/reference/transformations/outbox-event-router.html)
