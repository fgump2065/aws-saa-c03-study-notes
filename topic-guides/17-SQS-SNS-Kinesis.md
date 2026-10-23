# Section 17: SQS, SNS & Kinesis — the messaging block

## The idea

This section is one of the **heaviest question sources** on the exam, so let's build it from zero.

Imagine your web tier calls your order processor **directly**:

```
[Web server] --(sync call)--> [Order processor]
```

Three ways this hurts you:
1. **The spinner**: processor is slow → the customer stares at a loading wheel.
2. **Lost orders**: processor crashes mid-call → the order evaporates.
3. **Drowning**: Black Friday spike → processor gets 100x traffic and falls over.

Now put a **buffer** in between:

```
[Web server] --> [ QUEUE:  msg | msg | msg ] --> [Order processor]
   (instant "OK!")    (holds everything)        (works at its own pace)
```

One buffer fixes all three: the web tier answers instantly, messages survive crashes because they sit safely in the queue, and spikes just make the queue longer instead of killing the worker. This is **decoupling**, and the vivid analogy for the whole section: SQS is a **mailbox** (letters wait until you collect them), SNS is a **megaphone** (pushes notifications; failed deliveries follow protocol retries and an optional subscription DLQ), and Kinesis is a **river** (many people can stand on the bank reading the same water, and you can wade back upstream to re-read it).

## SQS — the mailbox (queue, PULL, one message → ONE worker)

**SQS (Simple Queue Service)**: producers put messages in; consumers **pull** them out. A message is normally hidden from other consumers while one worker processes it, but SQS is at-least-once: duplicate delivery and processing are possible, so consumers should be idempotent.

**Visibility timeout** — the mechanic the exam adores:
- When a worker reads a message, it is **hidden, not deleted** (default **30 seconds**).
- Worker finishes → explicitly deletes it. Worker **crashes** → timeout expires → message **reappears** for another worker. That's your built-in retry.
- **THE trap: "messages are being processed twice."** Cause: processing takes longer than the visibility timeout, so the message reappears while worker #1 is still on it. Fix: **increase the visibility timeout.**

More SQS facts:
- **DLQ (Dead-Letter Queue)**: a "poison message" that keeps failing gets moved to a side queue after **MaxReceiveCount** attempts — inspect failures without clogging the main queue.
- **Long polling** (up to **20 seconds**): consumer waits for messages instead of hammering empty-poll requests. "Reduce cost / empty responses from polling" → **long polling**.
- **Maximum message size: 1 MiB.** For larger payloads, use an S3-backed extended client or store the object in S3 and send a pointer.
- **Retention: 4 days default, 14 days max.** Not a database — collect your mail.
- **Standard vs FIFO**:

| | Standard | FIFO |
|---|---|---|
| Throughput | Nearly unlimited | Lower default throughput than Standard; high-throughput FIFO is available and quotas vary by Region/configuration |
| Delivery / deduplication | At-least-once (duplicates possible) | Deduplicates producer sends within a 5-minute window; consumers should still be idempotent because redelivery/duplicate processing can occur |
| Order | Best-effort | **Strict order** |
| Trigger | Default choice | "Order matters" / "no duplicates" |

- Classic pattern: **Auto Scaling Group of workers scaling on queue depth** (CloudWatch metric `ApproximateNumberOfMessagesVisible`) — the queue gets long, more workers spawn.

## SNS — the megaphone (PUSH, one → MANY)

**SNS (Simple Notification Service)**: publish one message to a **topic**, and SNS **pushes** it to ALL **subscribers** simultaneously — **Lambda, SQS, HTTP endpoints, email, SMS, mobile push**.

SNS is a **push notification service**, not a general-purpose durable message store. Delivery retries depend on the endpoint protocol; AWS-managed SQS/Lambda endpoints have extensive retries, while other protocols have different retry windows. Attach a subscription DLQ to retain undeliverable messages. **Message filtering** lets each subscriber receive only messages matching a policy (e.g., billing gets only `order_type: refund`).

**THE fan-out pattern** — the exam's darling:

```
                        +--> [SQS: analytics queue] --> analytics workers
[Order service] --> [SNS topic]
                        +--> [SQS: fulfillment queue] --> fulfillment workers
                        +--> [SQS: fraud queue]      --> fraud checker
```

SNS → **multiple SQS queues**. Why not SNS alone? **Durability**: if the fraud service is down for an hour, its **queue holds the messages** until it recovers. For consumers that need a durable backlog and independent processing pace, fan out to **SQS queues**; for other endpoint types, understand the retry policy and configure a subscription DLQ when needed.

## Kinesis Data Streams — the river

**Real-time streaming** with two superpowers SQS simply lacks:
1. **MULTIPLE consumers read the SAME data** (in SQS, one worker takes the message and it's gone; in the river, everyone on the bank sees the same water).
2. **REPLAY**: data is retained (**24 hours default, up to 365 days**) — rewind and reprocess.

Capacity unit = the **shard**: **1 MB/s in, 2 MB/s out**, and **ordering is guaranteed per shard** (partition key decides the shard). More throughput = more shards.

Kinesis's kill-shots vs SQS: **"reprocess yesterday's data"** or **"two applications must read the same stream"** → Kinesis Data Streams, never SQS.

## Amazon Data Firehose (formerly Kinesis Data Firehose) — the delivery truck

Streams is a river; **Data Firehose** is a delivery truck on a schedule: **near-real-time (~60 seconds buffering)**, **ZERO code, fully managed**, and it delivers to fixed destinations: **S3, Redshift, OpenSearch, Splunk** (plus optional **Lambda transform** en route).

Trigger phrase: **"stream data into S3 with the LEAST operational effort / no code"** → **Amazon Data Firehose** (formerly Kinesis Data Firehose). If the answer needs custom consumers or replay → Streams.

## Amazon MQ — the lift-and-shift broker

Existing on-prem app speaking **open broker protocols — MQTT, AMQP, JMS, STOMP** (RabbitMQ / ActiveMQ)? Rewriting for SQS/SNS is work. **Amazon MQ** = managed RabbitMQ/ActiveMQ, so you **lift-and-shift without code changes**. Protocol names in the question = Amazon MQ.

## Four-way decision table

| Service | Model | Consumers | Storage/Replay | Pick when |
|---|---|---|---|---|
| **SQS** | Queue, PULL | ONE per message | 4–14 days, no replay after delete | Decouple, buffer spikes, work distribution |
| **SNS** | Pub/sub, PUSH | MANY, all at once | No pullable backlog/replay; protocol retries and subscription DLQ available | Notify many systems instantly (fan out to SQS for durable consumer backlog) |
| **Kinesis Streams** | Stream (river) | MANY read SAME data | 24h–365d, **replayable** | Real-time analytics, multiple readers, reprocessing |
| **Amazon Data Firehose** (formerly Kinesis Data Firehose) | Delivery pipe | Fixed destinations | Buffers ~60s | Dump streams into S3/Redshift/OpenSearch/Splunk, zero code |
| **Amazon MQ** | Traditional broker | Protocol-based | Broker semantics | MQTT/AMQP/JMS migration, no code changes |

## Question patterns

> *"Web tier overwhelmed by traffic spikes; orders being lost"* → **SQS + Auto Scaling worker group** (queue buffers the spike, workers scale on depth)
> *"Messages must be processed in strict order and producer retries must not enqueue duplicate messages"* → **SQS FIFO** (order within each MessageGroupId + 5-minute send deduplication; make consumers idempotent because redelivery can still occur)
> *"One event must notify three downstream systems, and none may miss it even if temporarily down"* → **SNS fan-out to multiple SQS queues** (queues hold messages for sleeping services)
> *"Some SQS messages are being processed twice"* → **Increase the visibility timeout** (processing outlasts the hiding window)
> *"Messages that repeatedly fail are blocking the queue"* → **Dead-Letter Queue with MaxReceiveCount** (quarantine poison messages)
> *"Clickstream analytics: two applications must consume the same real-time data, with ability to reprocess yesterday"* → **Kinesis Data Streams** (multiple consumers + replay = its two kill-shots)
> *"Stream IoT telemetry into S3 with the least operational overhead, no custom code"* → **Amazon Data Firehose** (formerly Kinesis Data Firehose; zero-code delivery truck to S3)
> *"Reduce cost of consumers polling an often-empty queue"* → **Long polling (20s)** (fewer empty receives)
> *"Migrate an on-prem RabbitMQ/ActiveMQ app using AMQP/JMS/MQTT without rewriting"* → **Amazon MQ** (protocol names = MQ)
> *"Message payload exceeds 1 MiB"* → **Use the SQS Extended Client Library or store it in S3 and send a pointer**
> *"Each subscriber should only receive certain message types from a topic"* → **SNS message filtering** (filter policy per subscription)

## Pocket card

| Keyword | Answer |
|---|---|
| Decouple / buffer spikes | SQS |
| One message → one worker, PULL | SQS |
| Processing outlasts visibility timeout | Tune/extend timeout; make consumer idempotent (default 30s) |
| Poison messages | DLQ + MaxReceiveCount |
| Empty-poll cost | Long polling (20s) |
| Strict per-group order + producer deduplication | FIFO (5-minute dedup window; idempotent consumer still needed) |
| Message > 1 MiB | S3 + pointer / Extended Client Library |
| Retention | 4 days default / 14 max |
| One → many, PUSH | SNS (no pullable consumer backlog; protocol retries and subscription DLQ are available) |
| Notify many, durably | SNS → multiple SQS (fan-out) |
| Multiple consumers, same data, replay | Kinesis Data Streams (24h–365d) |
| Shard throughput | 1 MB/s in, 2 MB/s out, order per shard |
| Stream → S3, zero code, ~60s | Firehose (S3/Redshift/OpenSearch/Splunk) |
| MQTT / AMQP / JMS / RabbitMQ | Amazon MQ |
| Workers scale on queue depth | ASG + CloudWatch queue metric |

Every architecture you've built so far — containers, Beanstalk apps, API Gateway backends — becomes bulletproof the moment you slide one of these buffers between its pieces; decoupling is the quiet superpower behind almost every correct SAA-C03 answer.
