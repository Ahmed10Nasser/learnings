# System Design

Architecture styles, scalability, reliability, and distributed-systems concepts —
from monoliths and microservices to caching, messaging, and consistency trade-offs.

## Notes

- [Transactional Outbox](transactional-outbox.md) — reliably publish events alongside DB changes, avoiding the dual-write problem.
- [Inbox Pattern](inbox-pattern.md) — idempotent message consumption; dedupe redelivered messages for effectively exactly-once processing.
