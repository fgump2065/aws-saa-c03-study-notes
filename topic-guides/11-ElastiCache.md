# Section 11: ElastiCache

## The idea

Imagine a librarian who keeps getting asked the same question: *"What time do you close?"* The first time, she walks to the back office, checks the policy binder, walks back — thirty seconds. After the hundredth ask, she tapes a sticky note to the front desk: **"We close at 9."** Now every answer takes half a second. She didn't make the binder faster — she stopped consulting it for questions she'd already answered. **That sticky note is a cache.**

A **cache** keeps the answers to repeated questions in **RAM** (memory) instead of making the database work them out again. Memory access is much faster than a database round trip, so a cache hit can reduce latency and database load. End-to-end response time depends on the network, hit ratio, item size, and application.

**ElastiCache** is AWS's managed cache service. It supports **Valkey, Redis OSS, and Memcached**, as node-based clusters or ElastiCache Serverless. AWS manages the cache infrastructure; available replication, backup, and failover features depend on engine and deployment type. ElastiCache generally requires application integration: your app uses a cache client/pattern to check or update the cache. DAX is DynamoDB-specific and also requires use of a compatible DAX client, usually with only a few code changes.

The flow your app implements:

```
App request
    │
    ▼
[Check ElastiCache] ──hit──► return quickly on a cache hit ✔
    │
   miss
    │
    ▼
[Query database] ──► get result ──► write into cache ──► return
                                    (next asker gets the sticky note)
```

### Valkey / Redis OSS vs Memcached — the main differences

| Capability | **Valkey / Redis OSS** | **Memcached** |
|---|---|---|
| Replication | Yes (primary + replicas) | No engine-level replication in node-based clusters; Serverless is managed across AZs |
| Multi-AZ automatic failover | Supported with replicas (cluster mode enabled requires it) | No for node-based clusters; Serverless provides managed multi-AZ resilience |
| Backup & restore | Snapshots supported | Not for node-based Memcached; Serverless Memcached supports snapshots |
| **Sorted sets**, pub/sub | Yes | No |
| Multi-threading | Primarily command-threaded; scales through sharding | Yes |

ElastiCache offers both **node-based** and **Serverless** deployment options. Serverless automatically scales and replicates across multiple AZs; it is useful when you want to reduce capacity planning. Feature availability still depends on engine and deployment type.

Rule of thumb: choose **Valkey/Redis OSS** for data structures such as sorted sets, pub/sub, or node-based replication and failover. Choose **Memcached** for a simple, disposable, multi-threaded cache. A cache is not a durable system of record.
**Free question alert:** *"gaming leaderboard with real-time ranking"* → **Valkey/Redis OSS sorted sets**. Sorted sets keep members ordered by score automatically — top-10 lookups are instant. If you see "leaderboard," you're done reading.

### The session store pattern

Picture a web tier behind an **ALB and Auto Scaling Group**. A user logs in; server #3 remembers her session — in its own memory. Then the ASG scales in, terminates server #3, and she's suddenly logged out. That's the exam giveaway phrase: **"users are logged out when instances are terminated/scaled in."**

The fix: make the web tier **stateless** by moving sessions to a shared, fast store outside the instances — **ElastiCache for Valkey or Redis OSS** (or DynamoDB). Any server can now handle any request.

```
Users → ALB → [EC2] [EC2] [EC2]  ← stateless, disposable
                 └─────┴─────┘
                       ▼
              ElastiCache (Valkey/Redis OSS)  ← sessions live here
```

### Caching strategies — when do you write the sticky note?

- **Lazy loading** (cache-aside): populate the cache **on a read miss**. Like writing a cheat sheet *during* the exam — you only jot down answers to questions actually asked. Efficient (only hot data is cached), but data can be **stale** if the database changes underneath, and every miss pays a penalty.
- **Write-through**: after a database write, update the cache as part of the application write path. This can reduce staleness for that path, but adds write latency and can still become stale if any writer bypasses the cache/update or if the cache write fails. It can also cache data nobody reads.
- **TTL** (time-to-live) on cached entries is the middle ground: stale data expires automatically after N seconds.

**THE trap:** a cache alone cannot guarantee never-stale prices or inventory. Keep the database authoritative; use write-through/invalidation and appropriate consistency controls, and read from the source when strict freshness is required. TTL limits how long an entry can remain, but does not guarantee freshness.

### The read-offloader triangle

Three services all claim to "reduce database read load" — the scenario tells you which:

| Read pattern | Answer |
|---|---|
| **Repeated, identical** reads (same product page 10,000x) | **ElastiCache** |
| Reads that cannot reuse cached values but can be offloaded to a separate database copy | **Read Replica** (if supported by the DB engine; analytics platforms may fit large analytical workloads better) |
| DynamoDB + low-latency read cache | **DAX** (compatible client integration) |

**THE trap:** a cache helps only when results are reused with an acceptable freshness window. A read replica can offload eligible read-only database queries; for large analytical workloads, consider a purpose-built analytics service.

## Question patterns

> *"Popular product pages generate thousands of identical reads that overload RDS"* → **ElastiCache** (repeated reads belong in memory)
> *"Real-time gaming leaderboard with player rankings"* → **Valkey/Redis OSS sorted sets** (ranking is built in — free points)
> *"Users get logged out when the Auto Scaling Group terminates instances"* → **store sessions in ElastiCache for Valkey or Redis OSS** (make the web tier stateless)
> *"Cache should recover quickly from a node failure"* → **Valkey/Redis OSS with replicas and Multi-AZ automatic failover**; replication is asynchronous, so some recent writes can be lost. For stronger durability, evaluate supported durability settings and snapshots.
> *"Simple disposable, multi-threaded cache"* → **Memcached** (node-based Memcached has no engine-level replication or backup)
> *"Cached prices must be strictly current"* → read from the authoritative database or use strict consistency controls; cache write-through alone does not guarantee freshness
> *"Read-only reporting queries are adding load to the production database"* → **Read Replica** when appropriate; use an analytics service for large analytical workloads
> *"Speed up DynamoDB reads with minimal application changes"* → **DAX** (use its compatible client; ElastiCache requires an application-managed cache pattern)
> *"Publish/subscribe messaging between app components using the cache layer"* → **Valkey / Redis OSS** (pub/sub support)

## Pocket card

| Keyword | Answer |
|---|---|
| Repeated identical reads hammering DB | ElastiCache |
| Leaderboard / ranking | Valkey / Redis OSS sorted sets |
| Sessions / stateless web tier | Valkey / Redis OSS (or DynamoDB) |
| Sorted sets, pub/sub, replicated node-based cache | Valkey / Redis OSS |
| Simple disposable multi-threaded cache | Memcached (node-based) |
| Reduce staleness on application writes | Write-through + invalidation (not a strict freshness guarantee) |
| Stale-tolerant, cache only hot data | Lazy loading (+ TTL) |
| Read-only DB offload | Read Replica (large analytics may need a purpose-built service) |
| DynamoDB caching with compatible client | DAX (small application change) |
| Lower latency for cache hits | ElastiCache / DAX (actual latency varies) |

You now have the main caching patterns and engine tradeoffs. Section 12 covers specialty databases and their use cases.


