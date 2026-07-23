# Inbox Pattern

Reliably process incoming messages exactly-once (effectively) by first recording each
message's id in an "inbox" table within the same transaction as its handling — the
consumer-side counterpart to the [Transactional Outbox](transactional-outbox.md).

## The problem it solves

Message brokers give **at-least-once** delivery: the same message can arrive more than
once (redelivery after a consumer crash, network retry, or a producer using the
[outbox pattern](transactional-outbox.md), which re-publishes on failure). If a
consumer isn't careful, processing the same `OrderCreated` twice can double-charge a
customer, create duplicate records, or re-trigger side effects.

The consumer therefore needs **idempotent processing**: handling a message twice must
have the same effect as handling it once.

## How it works

1. Every message carries a unique **message id** (set by the producer).
2. When the consumer receives a message, in a **single local transaction** it:
   - Inserts the message id into an `inbox` (a.k.a. processed-messages / dedup) table
     with a `UNIQUE` constraint on the id.
   - Performs the business change (update state, write results).
   - Commits both together.
3. If the id already exists in the inbox, the message is a duplicate → **skip** the
   business logic and just acknowledge it.

Because the dedup record and the business change share one ACID transaction, you never
end up "processed the work but forgot to record it" or vice versa.

```
      message (id=abc)
             │
             ▼
   ┌── local transaction ─────────────┐
   │  INSERT INTO inbox (id) ...        │  ◀── UNIQUE(id): duplicate → rollback/skip
   │  UPDATE orders SET ...             │
   └────────────────────────────────────┘
             │
             ▼
      ack to broker
```

### Handling the duplicate

Two common styles:

- **Insert-first**: attempt `INSERT INTO inbox(id)`. If it violates the unique
  constraint, the message was already processed → ack and stop.
- **Check-then-act**: `SELECT` the id; if present, skip. (Prefer insert-first — it's
  atomic and avoids a check/act race.)

## Key properties & trade-offs

- **Effectively exactly-once**: at-least-once delivery + idempotent consumer =
  each message's effect applied once.
- **Storage & cleanup**: the inbox grows with every processed message. Prune old ids
  (e.g. by TTL / retention window longer than the broker's max redelivery window).
- **Scope of the id**: ids must be unique per logical stream; be careful if different
  producers could collide. A composite `(source, id)` key is safer.
- **Pairs with the outbox**: outbox guarantees reliable *sending*; inbox guarantees
  reliable *receiving*. Together they give reliable end-to-end messaging without
  distributed transactions.
- **Only works with a transactional store**: the inbox insert and business change must
  share one database transaction.

## When to use it

- Any consumer of an at-least-once broker where duplicate processing is harmful.
- The receiving side of systems already using the outbox pattern.

## When it's overkill

- The operation is **naturally idempotent** (e.g. `SET status = 'shipped'`, upserts by
  key) — a duplicate is harmless, so no dedup table is needed.
- Best-effort, non-critical events where occasional duplicates are acceptable.

## References

- microservices.io — [Idempotent Consumer](https://microservices.io/patterns/communication-style/idempotent-consumer.html)
- Chris Richardson, *Microservices Patterns* — Idempotent Consumer / message dedup.
- Related: [Transactional Outbox](transactional-outbox.md).
