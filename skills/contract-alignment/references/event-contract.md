# Event Contract

Compare what a producer publishes with what each consumer subscribes to and decodes, across
RabbitMQ, Kafka, NATS, SNS/SQS, Redis Streams, webhooks, and in-repo message buses. An event
contract has two layers, and both must agree:

```text
Routing   which messages reach the consumer at all (exchange + routing key + binding, topic, subject, queue, event type)
Payload   what the consumer can read from a message that did reach it (envelope, headers, body)
```

A payload that matches perfectly is still `MISALIGNED` if routing never delivers it.

## Contents

1. Pair producers and consumers by routing identifiers
2. Routing rules per broker
3. Envelope, headers, and payload
4. Event type and version
5. Consumer expectations to extract
6. Structural vs semantic mismatch
7. Evidence to collect
8. Common false positives

## 1. Pair producers and consumers by routing identifiers

Link a publish call to a subscription only through routing identifiers, never through names that
look alike. Resolve every identifier to its value: follow constants, config keys, and environment
variables to in-repo config files. A value that comes from deploy-time configuration not in the
repository makes the pairing `INFERRED` or `UNKNOWN`.

```text
publish site -> identifier value(s) -> broker routing rule -> subscription -> handler -> decode type
```

- Record every producer of the identifier. Several producers may publish the same event with
  different payloads; each one must satisfy each consumer.
- Record every consumer. Each consumer is its own pairing with its own status.
- A producer with no consumer in the repository: `Consumer: UNKNOWN (not in this repository)`.
  A consumer with no producer in the repository: `Producer: UNKNOWN`. Never invent the other side
  from the event name.

## 2. Routing rules per broker

| Broker | The consumer receives the message when | Drift to look for |
|---|---|---|
| RabbitMQ | the publish exchange and routing key match a binding of the consumer's queue under the exchange type: `direct` exact key, `topic` `*` = one word and `#` = zero or more words, `fanout` any key, `headers` header match | renamed routing key with an old binding; binding to the wrong exchange; queue declared and bound only by a consumer that is not deployed; default exchange (`""`) where the routing key must equal the queue name |
| Kafka | the consumer subscribes to the topic (or a matching pattern). The group decides sharing: the same group splits partitions, different groups each get every message | topic renamed on one side; environment prefix or suffix applied on one side only (`prod.orders` vs `orders`); two services in one group each silently receiving half |
| NATS | the subject matches the subscription (`*` = one token, `>` = one or more trailing tokens); JetStream also needs a stream whose subjects include it | subject hierarchy changed; consumer filter subject narrower than the published subject |
| SNS -> SQS | the queue is subscribed to the topic and the subscription filter policy matches the message attributes | filter policy on an attribute the producer does not set |
| Redis Streams | the consumer reads the same stream key (and group) | key built with a different prefix or tenant |
| Webhook (outbound) | the receiver's URL is the one configured on the sender, and the receiver accepts the method, path, and event type | receiver not in the repository (`UNKNOWN`); sender and receiver disagree on the event type header or field |
| In-process bus | the handler is registered for the same event type or name | handler registered for a type the publisher never dispatches |

Routing mismatches are usually `CRITICAL`: the consumer never acts, nothing errors, and the
producer's side looks healthy.

## 3. Envelope, headers, and payload

Compare each layer the consumer depends on:

| Layer | Producer side | Consumer side |
|---|---|---|
| Envelope | wrapper structure (CloudEvents `specversion`/`type`/`data`, a custom `{type, version, payload}`, or a bare body) | the structure the handler unwraps; does it expect `data` or read the body directly? |
| Headers / attributes | content type, event type, schema version, correlation or tenant headers the producer sets | headers the consumer reads or requires |
| Encoding | JSON, protobuf, Avro, string, compression | the decoder the consumer uses |
| Body | fields, types, units, enums (compare with `data-contract.md`) | fields the handler reads and the assumptions it makes |

With Avro or protobuf and a schema registry, compatibility rules live in the registry. If the
registry configuration is not in the repository, its compatibility mode is `UNKNOWN`.

Find the **last** place the payload is built. With an outbox relay, a retry republisher, or an
enrichment step, the wire payload is what that component emits, not what the business code wrote.

## 4. Event type and version

- **Type discriminator.** Many consumers switch on a type field or header (`type`, `eventType`,
  `x-event-name`). Compare the exact strings. A consumer that switches on a value the producer
  never sends has a dead branch; a producer value no branch matches goes to the default branch.
  Record what the default branch does.
- **Version.** Compare the version the producer writes (field, header, topic suffix, subject
  token) with the versions the consumer handles. Record what the consumer does with a version it
  does not know: reject, drop, try anyway, or crash.
- **Upcasters and adapters.** A consumer may convert old versions to new ones before handling.
  When one exists, compare the producer's version with the upcaster's input.

## 5. Consumer expectations to extract

Read the whole handler, not only the decode struct:

- fields it reads, and the fields it requires: dereferenced, indexed, used in a lookup, or
  compared without a presence check
- values it compares against: `if status == "ACTIVE"`, `case "SETTLED":`
- units it computes with: `time.Unix(x, 0)`, `amount / 100`
- what it does when decoding fails, a field is missing, or a value is unknown: nack, drop, log and
  ack, send to a dead-letter queue. Report this only as far as it changes the contract outcome
  (the message is silently ignored). Full ack, retry, and dead-letter analysis belongs to
  `distributed-flow`.
- ordering or co-occurrence assumptions **the code depends on**: the handler looks up a record
  that only another event creates, or rejects a transition that arrives "early". Report these
  only with code evidence on both sides.

## 6. Structural vs semantic mismatch

| Kind | Example |
|---|---|
| Structural | routing key `invoice.issued` vs binding `invoice.created`; body key `invoice_id` vs `invoiceId`; consumer expects a `data` envelope the producer does not send; protobuf field number reused |
| Semantic | `status: "completed"` means payment captured to the producer and order fulfilled to the consumer; `occurred_at` is when the payment settled vs when the event was published; the event fires on every retry attempt while the consumer treats each one as a new payment |

## 7. Evidence to collect

```text
Producer         publish call site; payload type and serializer; identifier constant or config key and its resolved value
Routing          exchange + type, topic, subject, queue; declaration and binding sites (who declares what)
Consumer         subscription site; binding or topic value; handler; decode type; every value comparison and unit conversion
Transformation   relay, adapter, upcaster, or mapper between publish and handle, if any
Tests            fixtures and sample messages on either side; do they match the producer's real payload?
```

## 8. Common false positives

- **Wildcards.** A binding `invoice.*` does receive `invoice.issued` on a topic exchange. A
  binding `invoice.*` does **not** receive `invoice.issued.v2` (use `#`).
- **Fanout and default exchanges.** A `fanout` exchange ignores the routing key.
- **Intentional filtering.** A consumer that handles only some event types or statuses, and does
  so explicitly (`case "DRAFT": return nil // nothing to do`), is aligned for the others.
  It is a mismatch only when a value the producer sends falls through by accident.
- **Consumer in another group.** Two consumers in different Kafka groups both receive every
  message; that is not a duplicate-delivery contract problem.
- **Shared payload type.** Both sides importing one struct shows agreement on the payload
  shape, not on routing, encoding, or meaning. Check those separately.
- **Same event name, different broker.** `OrderPaid` on Kafka and `OrderPaid` on an in-process
  bus are different contracts unless code bridges them.
