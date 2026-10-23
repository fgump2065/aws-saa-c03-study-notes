# Section 7: Route 53

## The idea

You type `www.example.com` into your browser. Your computer has no idea what that means — the internet runs on IP addresses, not names. So the first thing that happens, before a single byte of your website loads, is a lookup: *"what IP address is example.com?"*

**DNS (Domain Name System) is the internet's phonebook.** You give it a name, it gives you back a number, and then it's done. Route 53 is AWS's DNS service — and the single most important thing to understand about it is this:

**Route 53 answers the question, then EXITS. It never carries your traffic.**

Compare that to a load balancer, which is a **doorman standing in the path of every request** — every packet flows through it, forever. Route 53 is more like the receptionist in the lobby who points you to the right building and never sees you again.

That difference sets the two services at **different zoom levels**:

```
        "Which SITE/REGION?"           "Which INSTANCE?"
              Route 53          →      Load Balancer (ELB)
           (phonebook —                  (doorman —
         out of the path)             in the path always)
```

- **Route 53 picks between REGIONS or sites** (us-east-1 vs eu-west-1, primary vs DR).
- **ELB picks between instances** inside a region.

Because resolvers and clients **cache** DNS answers according to the record TTL, DNS changes and failover are not an immediate signal to every client. Route 53 can stop returning an unhealthy record after health evaluation, but clients may continue using cached answers until their TTL expires (and some clients cache longer). ELB health checks also take time to detect failures; after detection, the load balancer stops routing new requests to unhealthy targets. DNS and load-balancer health checks operate at different layers, and neither guarantees literally instant failover.

The standard architecture sandwich you'll see over and over:

```
User → Route 53 → picks a REGION → ALB in that region → picks an INSTANCE
        (DNS)                        (load balancer)
```

## Record types

| Record | Maps | Notes |
|---|---|---|
| **A** | name → IPv4 address | The bread and butter |
| **AAAA** | name → IPv6 address | "Quad-A" = IPv6 |
| **CNAME** | name → **another name** | **CANNOT exist at the root/apex** (`example.com` itself) — only on subdomains like `www.example.com` |
| **Alias** | name → supported AWS resource or another record in the same hosted zone | AWS extension. Works at the apex for supported targets; Route 53 queries to alias records targeting AWS resources are free, and Route 53 follows the target resource's current addresses. TTL is determined by the target. |

Alias records can point to supported AWS resources such as ELB load balancers, CloudFront distributions, S3 website endpoints, API Gateway, and other supported targets. Alias records can also point to another record in the same hosted zone. AWS resources like ALBs have IPs that change constantly, so you can never hardcode an A record to one.

**THE trap:** *"point the root domain `example.com` at an ALB"* → **Alias A/AAAA record** for a supported AWS target (CNAME cannot be used at the apex; a fixed A record would not track changing load-balancer addresses).

## Hosted zones

A **hosted zone** is a container for all the DNS records of one domain.

- **Public hosted zone** — answers queries from the whole internet.
- **Private hosted zone** — answers through the Route 53 Resolver for associated VPCs. For on-premises access, configure hybrid DNS forwarding (for example, Resolver inbound endpoints); association alone does not make it resolvable from on-prem. This is how you create internal names like `db.internal.company.com`. Exam signal: *"DNS names resolvable only within the VPC"* → private hosted zone.

## Routing policies

This is the heart of the Route 53 exam material. Learn the **signal phrases**:

| Policy | What it does | Exam signal phrase |
|---|---|---|
| **Simple** | One record, one answer, no health checks | The default; rarely the answer |
| **Weighted** | Split traffic by percentage you assign | *"A/B test"*, *"send 10% to the new version"* — **canary** |
| **Latency** | Answers with the region Route 53 estimates will provide the lowest latency for the query source | *"best performance for global users"*, *"lowest latency"* |
| **Failover** | Primary + secondary; flips when **health check** fails | *"active-passive"*, *"disaster recovery site"* |
| **Geolocation** | Routes according to the query source IP location as estimated by Route 53 | *"Serve users in a country/continent with specific content"*. Add a **Default** record for locations without a matching record; otherwise they get no answer. This is not a precise guarantee of a user's physical location. |
| **Geoproximity** | Routes toward the closest configured resource location; a **bias** expands or shrinks its geographic coverage | *"gradually shift/increase the traffic share to a region"*, the literal word **"bias"** |
| **IP-based** | Routes based on the originating IP address range (CIDR) | *"route users from these networks to a specific resource"* |
| **Multi-Value answer** | Returns up to **8 healthy values** per response (or unhealthy values if all are unhealthy) | *"return multiple health-checked IPs via DNS"*. Basic DNS distribution, not a substitute for a load balancer; multivalue alias records are not supported. |

**Latency vs Geolocation** — the classic confusion pair:
- **Latency** = a **performance** decision. "What's FASTEST for this user?" (A user in London might get served from us-east-1 if that's genuinely faster right now.)
- **Geolocation** = a **content/location rule** based on the query source IP location. It can support content localization, but it is not a precise physical-location or legal-compliance control.

Ask yourself: does the scenario care about *speed* or about *the query source location*? Speed → Latency. Rule → Geolocation.

**Geoproximity's name tag:** if the question says **"bias"** or talks about **expanding one region's share of traffic** by turning a dial, that's Geoproximity. Geolocation has no dial; Geoproximity is all dial.

## Health checks

Route 53 endpoint health checks are performed by distributed checkers outside your VPC. That creates one important limitation:

**Global checkers live on the public internet — they CANNOT see private endpoints** (instances in private subnets, private IPs). For a private resource, publish a suitable metric to CloudWatch, create an alarm for that metric, and configure a Route 53 health check to monitor the alarm state. Ensure the alarm is in the same AWS account as the health check.

**THE trap:** *"health check a private endpoint"* → global checkers can't reach it → **CloudWatch alarm-based health check**.

You can also build **calculated health checks** — combine up to 256 child checks with AND/OR/NOT logic into one parent check ("healthy only if at least 3 of 5 children are healthy").

## Failover: who handles what

| Failure level | Who fixes it | Speed |
|---|---|---|
| Instance/target fails health checks | **ELB** stops routing new requests to it | After health-check detection and target-state update |
| Whole region/site fails | **Route 53** health-checked failover can return the standby record | Depends on health evaluation and DNS/client caching (TTL) |

One neat corollary: if your DR design is **a single instance per region**, there's nothing for an ELB to balance — Route 53 Failover pointing at the instances directly is enough, and you can **skip the ELB entirely** (saves money; the exam likes this).

## Question patterns

> *"Point the apex/root domain example.com at an Application Load Balancer"* → **Alias record** (CNAME is forbidden at the apex; Alias is the AWS answer, free and auto-tracking).

> *"Active-passive setup: send traffic to the standby region only when the primary fails"* → **Failover routing policy + health check** ("active-passive" is the giveaway).

> *"Test a new app version by sending 10% of users to it"* → **Weighted routing** (canary = percentages = weights).

> *"Users in Germany must see the German-language site"* → **Geolocation** (user's location is a RULE, not a performance choice — and don't forget the Default record).

> *"Serve global users with the best possible performance"* → **Latency routing** ("fastest" = latency).

> *"Gradually increase the share of traffic going to the new region"* → **Geoproximity with bias** ("shift traffic share" / "bias" is Geoproximity's name tag).

> *"Health-check an endpoint that has a private IP inside a VPC"* → **CloudWatch alarm-based health check** (global checkers can't see inside your VPC).

> *"Return several healthy IPs and let clients pick, without a load balancer"* → **Multi-Value answer routing** (up to 8 healthy records).

> *"Multi-region app needs fast failover without waiting for DNS record changes"* → consider **AWS Global Accelerator** with health checks and static anycast IPs; it is in the traffic path, though failover still depends on health detection and client/network behavior.

## Pocket card

| Keyword in question | Answer |
|---|---|
| Root/apex domain → ALB/CloudFront/S3 | **Alias record** |
| name → name, NOT at apex | CNAME |
| "A/B test", "10% canary" | **Weighted** |
| "best performance", "lowest latency" | **Latency** |
| "active-passive", "DR site" | **Failover** + health check |
| "users in country X get X content", compliance | **Geolocation** (+ Default record) |
| "bias", "shift traffic share between regions" | **Geoproximity** |
| up to 8 healthy answers, DNS-level LB | **Multi-Value answer** |
| DNS only inside a VPC | **Private hosted zone** |
| health check a private endpoint | **CloudWatch alarm-based check** |
| combine many health checks | Calculated health check |
| Static anycast IPs and traffic-path failover across Regions | **Global Accelerator** (health detection still takes time) |

Route 53 hands out addresses and steps aside. For traffic-path failover with static anycast IPs, Section 8 introduces AWS Global Accelerator.



