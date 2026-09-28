# Asynchronous Flow

A hop is `ASYNC` when the caller continues without waiting for the receiver's result. The
receiver runs later, in another process, thread, or scheduler tick. Its failure does not reach
the caller directly, which is why async hops need their own delivery, retry, and consistency
analysis.

## Contents

1. Kinds of async work
2. Not async just because it is called an event
3. The publish half and the delivery half
4. Matching producers to consumers
5. Per-hop record
6. Broker checklists: RabbitMQ, Kafka, NATS, job queues
7. Delivery semantics, idempotency, ordering
8. Eventual consistency

## 1. Kinds of async work

| Kind | Evidence to find | Protocol label |
|---|---|---|
| Broker message or event | a publish call **and** a consumer subscription | `RABBITMQ`, `KAFKA`, `NATS`, or `OTHER_EXTERNAL_API (SQS)` and similar |
| Job queue | an enqueue call plus a worker registration (Sidekiq, Celery, BullMQ, asynq, Hangfire, DB-backed jobs) | the job queue's backing store, e.g. `REDIS` for Sidekiq/BullMQ/asynq, `RABBITMQ` for Celery on RabbitMQ, `DATABASE` for DB-backed jobs |
| In-process background work | `go func()`, executors, `@Async`, `setImmediate`, `asyncio.create_task`, thread pools | `IN_PROCESS` with mode `ASYNC` |
| Outbox relay | a row written in a transaction, and a poller or CDC connector that publishes it later | `DATABASE`, then the broker |
| Scheduled pickup | a cron job that scans for rows in some state | `DATABASE` with mode `ASYNC` |
| Outbound webhook | an HTTP call to a subscriber URL, often sent from a worker | `WEBHOOK` |

## 2. Not async just because it is called an event

Many in-process event mechanisms run listeners in the caller's thread, and the caller waits:

- Spring `ApplicationEventPublisher`: synchronous unless the listener is `@Async`.
  `@TransactionalEventListener` runs after commit, still in the caller's thread by default.
- Node `EventEmitter.emit`: calls listeners synchronously.
- MediatR `Publish`: awaits every handler.
- Django signals and Rails `ActiveSupport::Notifications`: synchronous.

Always check the dispatcher. If handlers run in the caller's thread and the caller waits, the
hop is `IN_PROCESS SYNC`, and a listener's exception may fail the caller.

## 3. The publish half and the delivery half

Publishing is usually a `SYNC` call to the broker client. Delivery to consumers is `ASYNC`.
Record the two halves separately:

```text
[Booking API]
   |  KAFKA SYNC: produce booking.confirmed (awaits broker ack?)
   v
[Kafka] topic booking.confirmed
   :  KAFKA ASYNC: delivery to consumer group ticketer
   v
[Ticket Worker]
```

For the publish half, the key question is whether the call waits for the broker to confirm
receipt:

- RabbitMQ: publisher confirms (`Confirm()`, `ConfirmSelect`, `waitForConfirms`) and the
  `mandatory` flag.
- Kafka: producer `acks`, and whether the send result is awaited or checked.
- NATS: core `Publish` is fire-and-forget; JetStream publish returns an ack.

Without confirmation, a successful return from `publish` does not mean the broker has the
message (`INFERRED`). Also record what the caller does when publish returns an error: fail the
request, retry, log and continue, or write to a fallback.

## 4. Matching producers to consumers

Link a producer to its consumers by the routing identifiers, not by names that look alike.
Resolve constants and config keys to their values.

- **RabbitMQ:** the producer publishes to an **exchange** with a **routing key**; the consumer
  reads from a **queue**. The binding (`QueueBind`, a declarations file, `definitions.json`)
  connects them. Apply the exchange type: `direct` matches the key exactly, `topic` matches
  `*`/`#` patterns, `fanout` ignores the key, `headers` matches headers.
- **Kafka:** the **topic** links them. The **consumer group** decides whether consumers share
  the partitions (same group) or each receive every message (different groups).
- **NATS:** the **subject**, with `*` and `>` wildcards. Queue groups load-balance within the
  group.

If no consumer exists in the repository, write `Consumer: UNKNOWN (not in this repository)`.
Never invent a consumer from the event's name. If there are several consumers, each is a
separate branch of the flow.

## 5. Per-hop record

```text
Event / message   Producer   Broker   Exchange | Topic | Subject   Routing key   Queue
Consumer (group)  Payload type   Ack mode   Retry   Dead letter   Ordering   Idempotency   Evidence
```

Leave nothing blank. Each field is a value with evidence, `none` (a `CONFIRMED` absence), or
`UNKNOWN`.

## 6. Broker checklists

### RabbitMQ

| Topic | Look for |
|---|---|
| Declarations | `ExchangeDeclare` (type, durable), `QueueDeclare` (durable, and arguments: `x-dead-letter-exchange`, `x-dead-letter-routing-key`, `x-message-ttl`, `x-max-length`, `x-queue-type`, `x-delivery-limit`), `QueueBind`. Note **who** declares each object. If only the consumer declares and binds the queue, messages published before the consumer first starts are unroutable (`INFERRED`). |
| Publish | persistent delivery mode, `mandatory` (unroutable messages are returned vs. silently dropped), publisher confirms |
| Consume | `autoAck` (true means the message is gone once delivered, so at-most-once), prefetch (`Qos`), and where `Ack`, `Nack`, or `Reject` are called, with their `requeue` flag |
| Requeue semantics | `Nack`/`Reject` with requeue=true redelivers immediately, which can loop hot on a poison message. With requeue=false the message is dead-lettered if the queue has a dead-letter exchange, and dropped otherwise. Check whether the dead-letter exchange itself is declared and bound anywhere in the repository. |
| Frameworks | Spring AMQP requeues on a listener exception by default (unless `AmqpRejectAndDontRequeueException` or `defaultRequeueRejected=false`), and retry comes from a configured `RetryInterceptor`. Other frameworks (MassTransit, Celery, and so on) have their own retry configuration; find it. |
| Broker-side policies | Policies set with `rabbitmqctl set_policy` or imported definitions live outside the code. If they are not in the repository, they are `UNKNOWN`. |

### Kafka

| Topic | Look for |
|---|---|
| Produce | topic, message key (the key picks the partition, which gives per-key ordering), `acks`, idempotence, producer retries, whether the send result is checked |
| Consume | group id, auto-commit vs. manual commit, and **when** the commit happens relative to the side effect (before means at-most-once, after means at-least-once). Defaults differ between raw clients and frameworks; record a relied-upon default as `INFERRED`. |
| Errors | Spring Kafka error handlers and `DeadLetterPublishingRecoverer`, `@RetryableTopic`, kafkajs retry options, and manual retry-topic producers. Read the configured backoff and recoverer; do not assume one. |
| Poison messages | A message that always fails can block its partition unless the code skips it or dead-letters it. |

### NATS

| Mode | Facts |
|---|---|
| Core NATS | At-most-once. No persistence, no ack. If no subscriber is connected, the message is gone. `QueueSubscribe` load-balances within a queue group. |
| JetStream | Stream (subjects, retention, storage) and consumer (durable name, `AckPolicy`, `AckWait`, `MaxDeliver`, `BackOff`). `Ack`, `Nak` (optionally with a delay), `Term`, `InProgress`. When `MaxDeliver` is exceeded the server emits an advisory; there is no automatic DLQ unless the code handles the advisory. |

### Job queues

Retry count, backoff, and dead or failed sets come from job options and worker configuration.
If you rely on a framework default, record it as `INFERRED`. Scheduled jobs also need overlap
handling (can two runs execute at once?) and partial-batch failure behavior.

## 7. Delivery semantics, idempotency, ordering

Derive the delivery semantics from the evidence. Do not assume them.

| Evidence | Semantics |
|---|---|
| Auto-ack, or commit before processing | At-most-once: a crash mid-processing loses the message |
| Ack or commit after processing | At-least-once: redelivery after a crash or error produces duplicates |
| At-least-once plus a dedupe check (inbox table, unique constraint on event ID, conditional update, idempotency key) | Effectively-once for that side effect |
| Documentation says "exactly once" | Verify it. It is usually at-least-once plus idempotency, or not true. |

**Consumer idempotency.** Look for a processed-message (inbox) table, a unique constraint on the
event ID, an upsert, a guarded update (`WHERE status = 'PENDING'`), or an idempotency-key
store. With at-least-once delivery and none of these, duplicate side effects are an `INFERRED`
risk. Name the side effect: a second email, a double credit.

**Ordering.** Ordering holds only within one partition or queue, with a single active consumer
and no redelivery. Prefetch greater than 1 with several consumers, requeues, and retry topics all
break it. State ordering guarantees only with evidence.

## 8. Eventual consistency

For each async hop, document:

- what is committed **before** the hop, and what is changed **after** it
- what a reader observes in between (for example, `GET /bookings/{id}` shows `CONFIRMED`
  before the ticket exists)
- what happens if the async half never completes: a stuck state, a reconciliation job, a
  replay mechanism, or `UNKNOWN`
