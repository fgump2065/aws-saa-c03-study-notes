# Section 5: Elastic Load Balancing & Auto Scaling

## The idea

Imagine a popular restaurant with one front door and a host standing at it. Guests don't wander in and pick their own tables — they walk up to the host, and the host seats them at whichever table is free. If a table breaks (wobbly leg, spilled soup everywhere), the host simply stops seating people there until it's fixed. Guests never know or care which table they got; they just know the restaurant's one address.

That host is a **load balancer**. Your servers (EC2 instances) are the tables. The load balancer gives clients **one single DNS name** (one front door), **spreads incoming traffic** across many servers, and constantly runs **health checks** — little "are you okay?" pings — so it stops sending traffic to any server that's sick. When the server recovers, traffic flows to it again.

Auto Scaling is the restaurant manager who watches how busy it is and **adds or removes tables** to match demand. Put them together and you get the classic AWS architecture: an Elastic Load Balancer (ELB) out front, an Auto Scaling Group (ASG) behind it, growing and shrinking with load, healing itself when instances die.

## OSI layers in 60 seconds (you need this to pick the right LB)

Networking is described in "layers" — the OSI model — and the exam expects you to know layers 3, 4, and 7. Think of it as **mail**:

- **Layer 3 (Network)** = the **address on the envelope**. Just an IP address — which building does this go to?
- **Layer 4 (Transport)** = the envelope address **plus the apartment number** — the **port** (like TCP port 443 or UDP port 3000). Which building, AND which door inside it?
- **Layer 7 (Application)** = the mail carrier **opens the envelope and reads the letter**. Now you can see HTTP details: the URL path (`/api/orders`), headers, cookies, hostnames.

**The layer a load balancer works at = how deep it looks into traffic = what decisions it CAN make.** An L4 balancer never opens the envelope, so it can't route based on URL paths — it's blind to them. But not opening envelopes makes it *fast*. That trade-off is the whole ALB-vs-NLB story.

## The four load balancers

| | Layer | Protocols | Superpower | Exam smell |
|---|---|---|---|---|
| **ALB** (Application) | 7 | HTTP, HTTPS, gRPC | Smart routing by path/host/header | "route /api to...", microservices, WAF |
| **NLB** (Network) | 4 | TCP, UDP, TLS | Speed + **static IP** | "millions of requests", UDP, "whitelist IPs" |
| **GWLB** (Gateway) | 3 | IP packets (GENEVE) | Sends traffic *through* appliance fleets | "third-party firewall/IDS/IPS" |
| **CLB** (Classic) | 4/7 | Legacy | Older, limited feature set | Prefer ALB/NLB for new designs; CLB may fit explicit legacy scenarios |

### ALB — the smart host (Layer 7)

The ALB opens every envelope. Because it reads HTTP, it can make **content-based routing** decisions:

- **Path routing**: `/api/*` → the API target group, `/images/*` → the image servers
- **Host routing**: `app.example.com` → one group, `admin.example.com` → another
- **Header / query-string routing**: route by a custom header or `?version=beta`

Key facts to lock in:

- Targets can be **EC2 instances, private IPs, Lambda functions, or containers (ECS)** — grouped into **target groups**.
- The ALB terminates the client connection and opens a new one to the target, so the target sees the ALB's IP. The real client IP is passed in the **X-Forwarded-For** header. *"App needs the client's IP behind an ALB"* → read X-Forwarded-For.
- **AWS WAF attaches to ALB** (a web application firewall inspects HTTP — it needs Layer 7, so it can't attach to an NLB).
- **Sticky sessions**: a cookie pins a user to the same target — for legacy apps that store session state locally. (Better design: externalize session to ElastiCache/DynamoDB, but "sticky sessions" is the quick fix answer.)
- **Dynamic port mapping** with ECS: multiple containers of the same app on one instance, each on a random port — the ALB tracks and routes to them. This is how you pack containers densely.
- ALB gives you a **DNS name, not a static IP**. THE trap: *"customers must whitelist a fixed IP"* → ALB **cannot** do this → put an **NLB** in front, or use **Global Accelerator**.

### NLB — the fast, no-questions host (Layer 4)

The NLB operates at Layer 4 for TCP/UDP/TLS traffic, using connection metadata rather than HTTP paths/headers. A TLS listener can terminate TLS, but it still does not route on HTTP content:

- Handles **millions of requests per second** with **ultra-low latency** — the "extreme performance" keyword answer.
- Speaks **any TCP or UDP** — gaming servers, IoT, MQTT, custom binary protocols. ALB can't do UDP; NLB can.
- **One static IP per enabled AZ**, and you can assign your own **Elastic IPs**. This is the common whitelisting answer.
- **Can preserve the client source IP**, but the default depends on target type and protocol: instance targets preserve it; IP targets using TCP/TLS default to off, while UDP-family protocols preserve it. Check/configure `preserve_client_ip.enabled`; Proxy Protocol v2 is another option.
- For a PrivateLink endpoint service, use an **NLB or GWLB** as the service target. If the application needs ALB features, an NLB can target an ALB. (More in Section 6.)

### GWLB — the security-appliance plumber (Layer 3)

The Gateway Load Balancer has exactly **one job**: transparently push all traffic **through a fleet of third-party security appliances** — intrusion detection/prevention systems (IDS/IPS), next-gen firewalls, deep packet inspectors — before it reaches your app. It's a "bump in the wire": traffic goes in one side, through the appliances, out the other, and nobody has to change IP addresses.

- Uses the **GENEVE protocol on port 6081** (a recognizable exam factoid).
- Keyword mapping is mechanical: *"inspect traffic with third-party / partner security appliances"* → **GWLB**.

### CLB — the retired host

Classic Load Balancer is the legacy generation. Prefer ALB/NLB for new designs, but do not treat CLB as categorically invalid if a question explicitly describes an existing legacy architecture or a CLB-specific constraint.

## Shared ELB features worth points

- **Cross-zone load balancing**: without it, each LB node sends traffic only to targets in *its own* AZ — uneven target counts can lead to uneven load. Cross-zone lets nodes use targets in other enabled AZs. **ALB: on at the LB level by default (can be disabled per target group); cross-AZ traffic is not billed as NLB-style cross-zone EC2 transfer. NLB/GWLB: off by default; enabling NLB cross-zone can incur EC2 data-transfer charges.**
- **Connection draining / deregistration delay**: when a target is removed (or ASG scales in), the LB stops NEW requests but lets in-flight requests **finish gracefully** (default 300s). *"Users get errors during scale-in"* → tune deregistration delay.
- **SNI (Server Name Indication)**: lets one load balancer host **multiple TLS certificates** — the client says which hostname it wants during the TLS handshake, and the LB serves the matching cert. *"Host many HTTPS domains on one ALB"* → SNI.
- **TLS termination**: the LB decrypts HTTPS itself (certificate lives on the LB, usually from ACM) and can talk plain HTTP to targets — offloading crypto work from your instances. (TLS = Transport Layer Security, the encryption behind the "s" in https.)

## Auto Scaling Groups

An ASG is a rule that says: "keep a fleet of instances alive, sized between these bounds, built from this recipe."

```
   Launch Template  ──►  ┌──────────── ASG ────────────┐
   (AMI, type, SG,       │  min: 2   desired: 4  max: 10│
    user data, key)      │  [EC2] [EC2] [EC2] [EC2]     │
                         │   AZ-a   AZ-a   AZ-b   AZ-b  │
                         └──────────────┬───────────────┘
                                        ▲
                             ELB spreads traffic, health-checks
```

- **Launch template** = the recipe (AMI, instance type, security group, user data). Launch *configurations* are the legacy version — prefer templates.
- **min / desired / max**: the ASG always steers actual count toward **desired**, clamped between min and max. Scaling policies work by changing desired.
- If an instance dies, the ASG **replaces it automatically** — self-healing for free.

### Scaling policies — match the keyword

| Policy | What it does | Exam keyword |
|---|---|---|
| **Target Tracking** | "Keep this metric at this value" (e.g., CPU at 40%) — AWS does the math | **Simplest**, "maintain X%" |
| **Step Scaling** | Add/remove N instances at metric thresholds you define | Fine-grained thresholds |
| **Scheduled** | Scale at known times | "**every Monday 9am**", "month-end batch", predictable |
| **Predictive** | Machine learning forecasts load and scales **ahead** of it | Recurring patterns, "proactively" |

- **Cooldown and instance warm-up are different.** The default cooldown for simple scaling is 300 seconds; it suppresses repeated simple-scaling actions. Instance warm-up controls when a new instance contributes metrics to dynamic scaling and is **not configured by default** (if unset, scaling policies can fall back to cooldown). Set it to the application startup/stabilization time; 300 seconds is only a starting point. *"ASG launches too many instances in bursts"* → review policy type and warm-up/cooldown.

### THE health-check trap (this WILL be on your exam)

By default, an ASG uses **EC2 status checks only** — is the VM itself running? But an instance can be perfectly *running* while the app on it is crashed, hung, or returning 500s. The ASG shrugs: "VM's alive, looks fine to me."

THE trap: *"the load balancer marks instances unhealthy but the ASG never replaces them"* → **enable ELB health checks on the ASG**. Then the ASG trusts the LB's application-level verdict and terminates + replaces app-dead instances.

### The ASG + SQS pattern

Classic decoupling architecture: workers in an ASG consume jobs from an SQS queue. Scale on the queue metric — ideally a **custom metric of backlog per instance** (`ApproximateNumberOfMessagesVisible` ÷ instance count) with target tracking. *"Scale workers based on pending jobs"* → **SQS queue depth drives the ASG**, not CPU.

### Termination policy (scale-in: who dies first?)

Default behavior balances capacity across AZs first, then prefers instances on outdated launch configurations/templates (launch configuration, different template, then older version). If there is still a choice, it may terminate the instance closest to the next billing hour; this matters less now that most EC2 usage is billed per second.

## The layered HA picture

Two different tools survive two different disasters:

```
Route 53 (DNS)  ──►  can direct traffic away from a REGION   (configured failover/health checks; DNS caches affect timing)
     │
     ▼
   ELB          ──►  stops routing to unhealthy targets after health checks
     │
     ▼
   ASG          ──►  replaces unhealthy instances when health checks identify them
```

**ELB distributes traffic within its configured Region. Route 53 can direct clients between regional endpoints when failover routing and health checks are configured; resolver/client DNS caching affects timing.** A second Region and a ready recovery design are still required.

## Question patterns

> *"Route `/api/*` to one set of servers and `/images/*` to another"* → **ALB path-based routing** (only L7 sees URLs)

> *"UDP-based multiplayer game needs low latency and a static IP"* → **NLB** (ALB can't do UDP or static IPs)

> *"Inspect all traffic with a third-party firewall/IDS appliance fleet"* → **Gateway Load Balancer (GWLB)** (GENEVE 6081, bump-in-the-wire)

> *"LB marks instances unhealthy, but ASG doesn't replace them"* → **attach the LB/target group and enable ELB health checks on the ASG** (EC2 health checks are always enabled; ELB checks are ignored by default)

> *"Payroll traffic spikes every month-end at a known time"* → **Scheduled scaling** (predictable = scheduled)

> *"Keep average CPU at 40% with minimal configuration"* → **Target Tracking** ("keep metric at value" + "simplest")

> *"Serve many HTTPS domains with different certificates on one ALB"* → **SNI** (multiple TLS certs, one listener)

> *"Corporate clients must whitelist fixed IP addresses for your load balancer"* → **NLB with Elastic IPs** (or Global Accelerator in front of an ALB)

> *"App behind ALB needs the original client IP"* → **X-Forwarded-For header** (NLB can preserve source IP depending on target/protocol configuration)

> *"Scale workers based on the number of unprocessed jobs"* → **ASG scaling on SQS queue depth** (backlog per instance)

> *"Millions of TCP requests per second, extreme performance"* → **NLB** ("millions" + "ultra-low latency" = L4)

## Pocket card

| Keyword | Answer |
|---|---|
| Path / host / header routing | ALB |
| HTTP, microservices, containers, Lambda target | ALB |
| WAF on a load balancer | ALB (L7 only) |
| UDP, extreme performance, millions req/s | NLB |
| Static IP / Elastic IP / whitelisting | NLB (or Global Accelerator) |
| PrivateLink endpoint service front | NLB or GWLB (NLB can target an ALB) |
| Third-party security appliances, GENEVE 6081 | GWLB |
| Classic Load Balancer | Legacy; usually choose ALB/NLB for new designs |
| Multiple TLS certs, one LB | SNI |
| Errors during scale-in | Deregistration delay (draining) |
| Balance targets across AZs | ALB cross-zone on by default; NLB/GWLB off by default (NLB cross-zone incurs data-transfer charges) |
| Client IP behind ALB | X-Forwarded-For |
| "Keep CPU at X%" simply | Target Tracking |
| Known time spikes | Scheduled scaling |
| ML-forecasted scaling | Predictive scaling |
| ASG ignores app-level failures | Enable ELB health checks on ASG |
| Scale on job backlog | SQS queue depth metric |
| Unhealthy target | ELB health checks stop routing; ASG replacement needs ELB checks enabled |
| Region-level failover | Route 53 (DNS, slower) |

One layer down from the load balancer sits the network it all lives in — subnets, gateways, and firewalls — which is exactly where Section 6 (VPC) picks up.


## Current AWS references

- [ELB load balancer types](https://docs.aws.amazon.com/elasticloadbalancing/latest/userguide/what-is-load-balancing.html)
- [ELB cross-zone behavior](https://docs.aws.amazon.com/elasticloadbalancing/latest/userguide/how-elastic-load-balancing-works.html)
- [NLB client IP preservation](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/edit-target-group-attributes.html)
- [ASG health checks](https://docs.aws.amazon.com/autoscaling/ec2/userguide/health-checks-overview.html)
- [ASG instance warm-up](https://docs.aws.amazon.com/autoscaling/ec2/userguide/ec2-auto-scaling-default-instance-warmup.html)
- [ASG default termination policy](https://docs.aws.amazon.com/autoscaling/ec2/userguide/ec2-auto-scaling-termination-policies.html)
- [PrivateLink service targets](https://docs.aws.amazon.com/vpc/latest/privatelink/privatelink-share-your-services.html)


- [ELB pricing and load balancer capacity units](https://aws.amazon.com/elasticloadbalancing/pricing/)