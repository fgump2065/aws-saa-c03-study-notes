# Section 12: The Other Databases (keyword zoo)

## The idea

Think of a hardware store wall: fifty screwdrivers, and each one exists for exactly one screw head. You don't need to *master* any of them — you need to glance at the screw and grab the right handle. AWS's specialty databases are that wall. **Each engine has ONE identifying keyword**, and the exam question always hands you the screw. This section is pure pattern-matching: keyword in, service out.

But first, one concept from zero, because it anchors half the zoo:

### OLTP vs OLAP

- **OLTP** — OnLine **Transaction** Processing: **many small operations**. *"Fetch order #123." "Update this user's address."* Millions per day, each touching a few rows. This is your day-to-day app database: RDS, Aurora, DynamoDB. Row-based storage (grab the whole record at once).
- **OLAP** — OnLine **Analytical** Processing: **few huge scans**. *"Average revenue per region per month across five years."* One query reads billions of rows but only 3 columns. Purpose-built warehouses use **columnar storage** — data stored column-by-column, so scanning `revenue` doesn't drag every other field off disk. Massively faster for analytics, plus columns compress beautifully.

**THE trap (the cardinal sin):** *"analysts run big reporting queries against the production database and the app slows down."* Never run analytics on your prod OLTP database. Choose an analytics path that fits the data and workload: **Redshift** for a managed warehouse, **Athena** for SQL on S3 data, or a **read replica** when the goal is to offload compatible read queries. Large analytics may need a purpose-built warehouse/lakehouse rather than a replica.

### The zoo — one call each

| Service | The ONE identifying call |
|---|---|
| **Redshift** | Managed **columnar data warehouse** for OLAP, BI, and large analytical workloads. **Redshift Spectrum** lets a Redshift query read structured/semi-structured data directly from S3 external tables without loading it into Redshift tables. **Redshift Serverless** runs the warehouse without managing provisioned clusters; billing is based on usage. |
| **Athena** | Serverless SQL on data in S3 (and supported federated sources). Pricing for standard queries is commonly **$5 per TB scanned** in many Regions; verify current regional pricing. Use columnar formats, compression, and partitioning to reduce scanned bytes. Keyword: *"query logs in S3 with SQL, minimal infrastructure."* |
| **Neptune** | **Graph database.** Data that is mostly *relationships*: **social networks, recommendation engines, fraud rings, knowledge graphs.** "Friends-of-friends" → Neptune. |
| **QLDB** | **Retired.** Support ended **July 31, 2025**; do not select it for a new design. For existing ledger data, AWS documented migration to Aurora PostgreSQL. Choose an audit/ledger architecture based on required immutability, verifiability, and trust model. |
| **Amazon Managed Blockchain (AMB)** | Managed public-chain access (Ethereum/Bitcoin/Polygon) and private Hyperledger Fabric networks for multi-party applications. **Service status:** stops accepting new customers on **October 29, 2026** and ends support **September 29, 2027**; existing customers only until then. Do not recommend for new designs; evaluate migration/alternative providers. |
| **Amazon Timestream** | Time-series workloads: IoT sensors, telemetry, and DevOps metrics. Distinguish **Timestream for LiveAnalytics** (serverless time-series store; closed to new customers since **June 20, 2025**) from **Timestream for InfluxDB** managed InfluxDB offerings. AWS currently recommends InfluxDB 3 for many large-scale/new analytics workloads and InfluxDB 2 for some low-latency operational workloads; check fit and availability. |
| **DocumentDB** | Managed document database with **MongoDB API compatibility** (including MongoDB 8.0 compatibility). It is not identical to MongoDB; check supported APIs/features and test before migration. |
| **Keyspaces** | Serverless managed Apache Cassandra-compatible service using CQL and compatible Cassandra drivers. Compatibility is not complete; check supported APIs/features and limits. |
| **OpenSearch** | **Full-text / fuzzy search** + **log analytics with dashboards** (the old Elasticsearch/Kibana). Search "runing shoes" and still find running shoes. |

### The two comparisons worth 30 extra seconds

**Athena vs Redshift** — both do SQL analytics, so which?

```
Data stays in S3, ad-hoc, occasional, zero infra  →  ATHENA
Data loaded into a warehouse, heavy repeated BI,
complex joins, dashboards all day                 →  REDSHIFT
(Middle child: query S3 FROM your existing
 Redshift cluster without loading                 →  SPECTRUM)
```

**Athena cost optimization** — this appears almost **verbatim**: *"reduce Athena query costs / improve performance."* Athena bills per TB **scanned**, so scan less:
1. Convert files to **columnar formats — Parquet or ORC** (read only needed columns),
2. **Compress** the data,
3. **Partition** it (e.g., by date, so queries skip irrelevant folders).

All three shrink bytes scanned = smaller bill. Memorize the trio.

**OpenSearch combo pattern:** DynamoDB **cannot do fuzzy or full-text search** (it's a coat-check, not a librarian). A common pattern is **DynamoDB + Streams → Lambda → OpenSearch Service** — writes land in DynamoDB, a stream-driven consumer indexes documents in OpenSearch, and search queries go there. This is an application integration pattern, not an automatic native index.

## Question patterns

> *"BI team needs a petabyte-scale data warehouse for complex analytical queries"* → **Redshift** (OLAP, columnar, built for BI)
> *"Query data in S3 from the existing Redshift cluster without loading it"* → **Redshift Spectrum** (extends the warehouse into S3)
> *"Run occasional SQL queries on ALB/VPC logs stored in S3, minimal setup"* → **Athena** (serverless SQL-on-S3)
> *"Reduce Athena costs and speed up queries"* → **Parquet/ORC + compression + partitioning** (bill = TB scanned, so scan less)
> *"Social app needs friend-of-friend recommendations / detect fraud rings"* → **Neptune** (relationships = graph)
> *"Need an immutable, verifiable ledger"* → **QLDB is retired** (support ended July 31, 2025); design a current audit/ledger solution around the trust and immutability requirements.
> *"Several parties need a shared blockchain network"* → Amazon Managed Blockchain historically supported public chains and private Hyperledger Fabric; it stops accepting new customers Oct 29, 2026 and ends support Sep 29, 2027, so evaluate alternatives for new designs.
> *"Millions of IoT sensor readings per minute, analyzed over time windows"* → **Timestream** (time-series)
> *"Evaluate a managed AWS target for a MongoDB workload"* → **DocumentDB** (MongoDB API-compatible; validate feature differences and workload compatibility before migration).
> *"Evaluate a managed/serverless target for a Cassandra workload"* → **Keyspaces** (CQL/driver compatibility; verify supported features and limits).
> *"Product catalog needs fuzzy, typo-tolerant full-text search; data is in DynamoDB"* → **OpenSearch** (via DynamoDB Streams → Lambda; DynamoDB's native Query/Scan is not a full-text or fuzzy-search engine)
> *"Analysts' reports are slowing production OLTP"* → offload reads to an appropriate analytics system (Redshift for warehouse workloads, Athena for S3 data, or a read replica for compatible queries).

## Pocket card

| Keyword | Answer |
|---|---|
| Data warehouse / OLAP / BI / petabyte | Redshift |
| Query S3 from Redshift, no loading | Redshift Spectrum |
| SQL on S3, serverless, ad-hoc / logs | Athena |
| Cut Athena cost | Parquet/ORC + compress + partition |
| Graph / social / fraud ring / recommendations | Neptune |
| QLDB ledger | Retired July 31, 2025; choose a current audit/ledger design |
| Multi-party blockchain | AMB (ends support Sep 29, 2027; no new customers after Oct 29, 2026) |
| Time-series / IoT / telemetry / metrics | Timestream (LiveAnalytics closed to new customers; evaluate InfluxDB 2/3) |
| MongoDB API compatible | DocumentDB (check compatibility before migration) |
| Cassandra CQL / drivers | Keyspaces (check feature support) |
| Full-text / fuzzy search, log dashboards | OpenSearch |
| Analytics burden on a production OLTP DB | Offload appropriately — Redshift, Athena, or a read replica |

That covers the main database selection signals. Section 13 turns to event-driven compute with Lambda.