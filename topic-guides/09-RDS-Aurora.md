# Section 9: RDS & Aurora

## The idea

Running your own database server means patching the OS, applying engine updates, taking backups at 2 a.m., and rebuilding everything when the hardware dies. **RDS (Relational Database Service) = AWS runs the database for you.** You pick an engine — **MySQL, PostgreSQL, MariaDB, Oracle, SQL Server, or Db2** — and AWS handles the machinery underneath.

The price of that convenience: **you get NO OS access**. You can't SSH in, can't install custom extensions at the OS level, can't tweak the engine binaries. If a question demands **host OS access or extensive engine customization**, standard RDS is out — consider **database on EC2** or **RDS Custom** for supported Oracle and SQL Server workloads.

Now separate **high availability** from **read scaling**. RDS has two Multi-AZ deployment types:

- **Multi-AZ DB instance deployment**: one primary plus a synchronous standby in another AZ. The standby is for failover and does **not** serve reads.
- **Multi-AZ DB cluster deployment**: one writer plus **two readable standby instances** in different AZs. These standbys can serve reads and are promoted automatically on failover.

Read replicas are asynchronous copies used for read scaling, reporting, or some DR designs. They can be in the same Region or another Region, depending on engine support; promotion is generally manual. An Aurora Replica is also a reader, but Aurora can automatically promote one during cluster failover.

| | **Multi-AZ DB instance** | **Multi-AZ DB cluster** | **Read replica** |
|---|---|---|---|
| Replication | Synchronous | Synchronous | Asynchronous |
| Topology | Primary + standby in another AZ | Writer + 2 readable standbys across AZs | Same AZ, cross-AZ, or cross-Region (engine-dependent) |
| Read from standby/replica? | **No** | **Yes**, use reader endpoint | **Yes** |
| Failover | Automatic; commonly about 60–120 seconds | Automatic; typically under 35 seconds | Usually manual promotion (Aurora cluster replicas are an HA exception) |
| Main use | Availability | Availability plus read capacity | Read scaling, reporting, or DR |

RDS read-replica quotas are generally up to **15 per primary**, subject to engine and configuration limits. Aurora supports up to **15 Aurora Replicas**. Read replicas are not a substitute for a Multi-AZ design when automatic failover is required.

**THE trap:** *"use the Multi-AZ DB instance standby to serve read traffic"* → **not supported**. Distinguish it from a **Multi-AZ DB cluster**, whose two standby instances are readable.

Production designs can combine Multi-AZ for availability with read replicas for scale. Choose based on read needs, recovery objectives, engine support, and cost.

## Backups & encryption



- **Automated backups** → enable **PITR (Point-In-Time Recovery)**, with retention from 1 to **35 days**. When deleting a DB instance, you can choose to retain automated backups for their remaining retention period; otherwise they are deleted.
- **Manual snapshots** → kept **forever**, until you delete them. **"Retain backups for 5 years / 7 years / compliance" → manual snapshots or AWS Backup with a suitable lifecycle policy** — automated backup retention is capped at 35 days.

**Encryption is a birth decision.** You choose encryption when creating an RDS database; you cannot turn storage encryption on in place. To encrypt an existing unencrypted database, copy a snapshot with encryption enabled, then restore a new database from that encrypted copy:

```
unencrypted DB → snapshot → COPY the snapshot (enable encryption) → restore from encrypted copy → repoint the app
```



## RDS Proxy

Databases hate being swarmed. Every connection costs memory, and **Lambda** — which can scale to thousands of concurrent executions in seconds — will happily open thousands of connections and knock your database over. **RDS Proxy = a connection pool** that sits in front of RDS, letting thousands of clients share a small, warm set of database connections.

- **Lambda + connection churn or too many DB connections → consider RDS Proxy.** It pools and reuses backend connections, which is useful for many short-lived Lambda invocations. It adds cost/latency and some session behavior can pin connections, so it is not automatically needed for every Lambda + RDS setup.
- The proxy can make failovers less disruptive by maintaining client connections and routing new work to the new writer; in-flight transactions can still be interrupted, so applications need retry handling.

Also worth one neuron: **storage autoscaling** — RDS storage autoscaling can increase allocated storage when it runs low, within engine, storage-type, and configured maximum limits ("database disk growth is unpredictable" → consider storage autoscaling).

## Aurora

Aurora is AWS's own cloud-native engine, **compatible with MySQL and PostgreSQL** (your app connects the same way). The architectural trick: Aurora **separates compute from storage**.

```
   [Writer node]  [Reader]  [Reader] ...   ← compute: disposable, pluggable
        │            │         │
   ═════╧════════════╧═════════╧═════════
     Shared storage: 6 copies across 3 AZs, self-healing,
     auto-grows to 256 TiB on supported engine versions
```

Because every node plugs into the **same shared storage**, a failed writer is replaced by simply promoting a reader that already sees all the data:

- **Failover is typically under 30 seconds** for Aurora intra-Region failover; actual recovery depends on the event and configuration
- **Up to 15 Aurora Replicas**, with typically low replication lag; it is not a fixed millisecond guarantee
- **Storage auto-grows to 256 TiB on supported engine versions** — no provisioning
- **Cluster endpoints**: the **writer endpoint** tracks the current writer across failover; the **reader endpoint** distributes read connections across available replicas. Apps use the appropriate endpoint and avoid hardcoding instance addresses.

**Aurora's special features — feature → scenario:**

| Feature | The scenario it answers |
|---|---|
| **Serverless v2** | Spiky or variable workloads; compute scales within a configured ACU range. Supported engine versions can scale to **0 ACUs and auto-pause** when idle, but first connection after pause has resume delay |
| **Global Database** | Cross-Region low-latency reads and DR. Replication is asynchronous; RPO is typically measured in seconds and RTO is generally in minutes, depending on outage and recovery procedure. Planned switchovers can achieve zero data loss. |
| **Cloning** | Fast copy-on-write clone for staging/test; it initially uses little additional storage, then storage grows as either cluster diverges. |
| **Backtrack** | Rewind an Aurora MySQL cluster to a consistent earlier point without a snapshot restore. Must be enabled at cluster creation; target window is up to 72 hours, and a backtrack briefly disrupts connections. |

RPO (Recovery Point Objective) = how much data you may lose; RTO (Recovery Time Objective) = how long until you're back. Treat Global Database RPO/RTO as workload and event dependent, not a guaranteed fixed number.

## Question patterns

> *"Database must survive an AZ failure with no manual intervention"* → **Multi-AZ DB instance** for a non-readable synchronous standby, or **Multi-AZ DB cluster** when readable standbys/read capacity are also useful.

> *"Reporting/analytics reads are slowing down the production database"* → **Read replica** to offload read queries. For repeatedly requested hot data, an application cache such as ElastiCache may reduce database reads further, depending on freshness needs.

> *"Lambda functions create too many short-lived database connections"* → consider **RDS Proxy** for connection pooling and reuse.

> *"Encrypt an existing unencrypted RDS database"* → **snapshot → copy with encryption → restore** (encryption is creation-time only; this is the snapshot dance).

> *"Global application needs cross-Region reads and low-lag DR"* → **Aurora Global Database** (asynchronous replication; RPO typically seconds, RTO generally minutes; exact recovery depends on event and procedure).

> *"Dev database sits idle nights and weekends; minimize cost"* → **Aurora Serverless v2** (variable capacity; eligible versions can auto-pause at 0 ACUs, with resume delay).

> *"Need a full copy of the production database for testing, quickly and cheaply"* → **Aurora cloning** (fast copy-on-write; starts with little extra storage, then uses more as changes accumulate).

> *"Developer ran a bad DELETE; rewind an eligible Aurora MySQL cluster"* → **Aurora Backtrack**, if enabled beforehand and within its configured backtrack window (up to 72 hours).

> *"Compliance requires database backups kept for 7 years"* → **Manual snapshots or AWS Backup with a lifecycle policy** (automated backup retention caps at 35 days).

> *"Application needs OS-level access / a custom database engine"* → **EC2** (or RDS Custom for Oracle/SQL Server) — plain RDS gives no OS access.

## Pocket card

| Keyword in question | Answer |
|---|---|
| survive AZ failure, auto-failover | **Multi-AZ DB instance** (synchronous standby, commonly 60–120s) or Multi-AZ DB cluster (two readable standbys, typically <35s) |
| Multi-AZ DB instance standby serves reads | **No** (Multi-AZ DB cluster standbys are readable) |
| scale reads, offload reporting | **Read Replica** (async, up to 15, manual promotion) |
| read replica promotion | Usually manual; Aurora replicas can be promoted automatically during cluster failover |
| same queries over and over | ElastiCache (not a replica) |
| Lambda + RDS connections | **RDS Proxy** |
| backups > 35 days / "retain X years" | Manual snapshots or AWS Backup lifecycle policy |
| restore to any point in time (≤35 days) | Automated backups / PITR |
| encrypt existing DB | **Snapshot → copy-encrypted → restore** |
| OS access / custom engine | EC2 or RDS Custom |
| typical fast failover, up to 15 low-lag replicas, up to 256 TiB storage on supported versions | **Aurora** |
| variable compute; eligible clusters can pause when idle | **Aurora Serverless v2** (0 ACUs supported by some engine versions) |
| cross-Region reads and seconds-scale typical RPO | **Aurora Global Database** (RTO generally minutes) |
| fast copy-on-write staging copy | **Aurora cloning** |
| rewind eligible Aurora MySQL cluster (enabled in advance; max target window 72h) | **Aurora Backtrack** |

You now know how to keep the database standing and fast — but when the exam's read traffic is the *same* hot data thousands of times per second, even fifteen replicas is the wrong answer, and that's where caching with ElastiCache takes over.




