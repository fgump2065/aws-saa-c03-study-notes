# Section 10: DynamoDB

## The idea

Picture a coat-check counter at a giant stadium. You hand over your coat, you get ticket **#48291**. Later you hand back the ticket and — instantly — your coat appears. The coat-check clerk never *searches* the racks; the ticket number tells her exactly which hook to walk to. It doesn't matter if the stadium holds 500 people or 5 million: one ticket, one hook, one grab. **That's DynamoDB.**

DynamoDB is AWS's **serverless NoSQL database**. Spell that out:

- **Serverless** = you never see, patch, size, or connect to a server. No database servers to provision or patch; you access it through AWS APIs. AWS manages the underlying infrastructure.
- **NoSQL** = data lives as **items** (like rows) with flexible attributes (like columns, but each item can have different ones). No joins, no rigid schema.

DynamoDB is designed for **single-digit millisecond performance at scale** when tables, keys, and capacity are designed for the workload. It does not guarantee identical latency or unlimited throughput: hot keys, throttling, quotas, and request patterns still matter.

### DynamoDB vs RDS — the first fork in every question

| Scenario says... | Pick |
|---|---|
| Joins, complex queries, existing SQL app, relational schema | **RDS / Aurora** |
| Massive scale, serverless, key-value lookups, millisecond latency, unpredictable growth | **DynamoDB** |

**THE trap:** *"migrate with no code changes from MySQL"* → that's **RDS/Aurora**, never DynamoDB. NoSQL means rewriting queries.

### Keys and hot partitions

Every table needs a **partition key** (the coat-check ticket — determines which physical partition stores the item), plus an optional **sort key** (orders items *within* a partition, so one customer can have many orders sorted by date).

DynamoDB spreads data across partitions by hashing the partition key. If everyone's key is `country = "USA"`, all traffic slams one partition — a **hot partition** — and you get throttled while the rest of the table sits idle.

**Rule: choose a partition key that distributes both data and request traffic well.** High-cardinality keys such as `user_id` often help, but a skewed workload can still create hot keys even when many key values exist. Avoid a single heavily accessed key or low-cardinality key unless your access pattern is deliberately distributed.

```
Good key (user_id):            Bad key (country):
[P1][P2][P3][P4]               [P1][P2][P3][P4]
 ▲▲  ▲▲  ▲▲  ▲▲                ████  .   .   .
 even spread                    hot!  idle idle idle
```

### Capacity modes — same logic as EC2 pricing

- **Provisioned**: you declare read/write capacity up front. **Cheaper for steady, predictable traffic.** Add **auto scaling** to flex within bounds. (This is your Reserved-Instance instinct.)
- **On-Demand**: pay per request without setting read/write units. Often a good starting point for spiky, unpredictable, or new workloads. It can still throttle at hot partitions, configured throughput limits, or service quotas; sudden large increases may need warm throughput planning.

**THE trap:** *"traffic is unpredictable"* → consider **On-Demand mode**, but it does not fix a hot partition or bypass throughput quotas.

### RCU / WCU — the exam arithmetic

| Unit | Buys you |
|---|---|
| **1 RCU** | **1 strongly consistent** read/sec, item ≤ 4 KB — or **2 eventually consistent** reads/sec |
| **1 WCU** | **1 write**/sec, item ≤ **1 KB** |

Read units are rounded up by 4 KB; writes by 1 KB. An eventually consistent read uses half the read units of a strongly consistent read. Standard reads default to eventual consistency. Strong reads are available for tables and LSIs in a Region, but not GSIs; a strong read does not make MREC global-table replicas synchronous.

### Feature zoo → scenario matcher

| Feature | The one-liner |
|---|---|
| **DAX** (DynamoDB Accelerator) | Managed in-memory cache for DynamoDB with microsecond read latency on cache hits. Uses a DAX client with a few application code changes. It serves eventual reads from cache; strongly consistent and transactional reads pass through to DynamoDB. |
| **Global Tables** | Multi-Region, multi-active tables. Default **MREC** asynchronously replicates writes between Regions (usually within seconds); Streams are enabled and managed for replication. **MRSC** is also available in supported Region sets for strongly consistent cross-Region reads/writes, with higher write latency and deployment/API restrictions. Aurora Global Database has a single primary writer Region at a time. |
| **Streams** | A **change feed**: every insert/update/delete recorded for **24 hours**, typically triggering **Lambda**. Keyword: *"react to item changes."* |
| **TTL** (Time To Live) | Set an expiry timestamp in Unix epoch seconds → DynamoDB deletes expired items asynchronously, typically within a few days. The source TTL delete does not consume write capacity, but MREC global-table replicas incur replicated write costs. TTL is unavailable for MRSC global tables. |
| **GSI** (Global Secondary Index) | Query by alternate key attributes. Can be added after table creation. In provisioned mode it has separately configured read/write capacity; in on-demand mode it is billed per request. Adding/backfilling an index consumes resources. |
| **LSI** (Local Secondary Index) | Alternate *sort* key, same partition key — can be created **only when the table is created**. **THE trap:** if the table already exists, LSI is the wrong answer; pick **GSI**. An item collection sharing one partition-key value is limited to 10 GB when LSIs are present. |
| **Transactions** | ACID, all-or-nothing operations across items and tables (within documented transaction limits). Transactional reads/writes use **2x** the capacity units of standard operations for the same item sizes. Keyword: *"atomic"*, *"all succeed or none."* |
| **PITR** (Point-In-Time Recovery) | Continuous backups; restore a table to a selected second within the last **35 days** (restore creates a new table). |

### Two more exam staples

**The serverless poster-child stack** — memorize the chain:

```
User → CloudFront → API Gateway → Lambda → DynamoDB
        (CDN)       (front door)   (code)    (data)
```

This is a common pattern for a fully managed/serverless web API; choose components to match the application needs.

**400 KB item limit.** The item size includes attribute names and values. For images/documents, store the object in S3 and keep its object key or metadata in DynamoDB (generate access URLs as needed). This S3-plus-pointer pattern is a recurring correct answer.

## Question patterns

> *"Traffic is spiky or unknown"* → consider **On-Demand mode**; still design for hot keys, warm throughput, and quotas
> *"Reduce repeated eventually consistent DynamoDB reads to microseconds"* → **DAX** (use its client; strongly consistent/transactional reads go to DynamoDB)
> *"Global users need local DynamoDB reads and writes"* → **Global Tables** (MREC is asynchronous; MRSC offers cross-Region strong consistency with added latency and constraints)
> *"Run custom logic whenever items are added or modified"* → **DynamoDB Streams + Lambda** (24-hour change feed triggers functions)
> *"Expire old session records automatically"* → **TTL** (asynchronous cleanup, typically within a few days; global-table replica writes may incur capacity charges)
> *"Table keyed on user_id, but app must also query by email"* → **GSI** (new query attribute, addable anytime — LSI is creation-time-only bait)
> *"Debit one account and credit another — both or neither"* → **DynamoDB Transactions** (ACID, 2x capacity cost)
> *"Store 2 MB documents per record"* → **S3 for the object, DynamoDB holds the pointer** (400 KB item limit)
> *"Read the latest successful write in this Region"* → **Strongly consistent read** (table/LSI in one Region; costs twice an eventual read)
> *"Reporting needs joins across many tables"* → **RDS/Aurora**, not DynamoDB (relational = SQL)

## Pocket card

| Keyword | Answer |
|---|---|
| Managed/serverless NoSQL, key-value/document access patterns | DynamoDB |
| Joins / complex SQL | RDS or Aurora instead |
| Spiky / unknown traffic | On-Demand mode |
| Steady, predictable traffic | Provisioned (+ auto scaling) |
| Microsecond cached reads | DAX (client integration; eventual reads from cache) |
| Multi-Region replication | Global Tables (MREC async by default; MRSC available with constraints) |
| React to item changes | Streams → Lambda |
| Auto-expire items asynchronously | TTL (replicated deletes can cost in global-table replicas) |
| Query by another attribute, existing table | GSI (LSI = creation-time trap) |
| Atomic multi-item | Transactions (transactional reads/writes use 2x units) |
| Restore to any second, 35 days | PITR |
| Item > 400 KB | S3 + pointer |
| 1 RCU | 1 strong or 2 eventual reads ≤ 4 KB |
| 1 WCU | 1 write ≤ 1 KB |

DynamoDB is a managed key-value and document database; for frequently reused data that needs even lower latency, Section 11 covers ElastiCache.


