# System Design: Zero to Hero
### A Deep-Dive Reference for Every Component in paperdraw.dev

> **How to use this guide:** Each section maps to a category on paperdraw.dev. Read the concept, understand the *why*, then drag the component onto your canvas and wire it up. By the end of Week 1 you should be able to design any production system from scratch.

---

## Table of Contents

1. [Traffic & Edge](#1-traffic--edge)
2. [Network](#2-network)
3. [Compute](#3-compute)
4. [Storage](#4-storage)
5. [Messaging](#5-messaging)
6. [Fintech & Banking](#6-fintech--banking)
7. [Security](#7-security)
8. [Data Engineering](#8-data-engineering)
9. [My Services](#9-my-services)
10. [AI & Agents](#10-ai--agents)
11. [Techniques](#11-techniques)
12. [Week-1 Study Plan & Design Challenges](#12-week-1-study-plan--design-challenges)

---

## 1. Traffic & Edge

> **Mental model:** These are the gatekeepers. Before a single byte reaches your servers, it passes through these layers. Think of them as a funnel — the internet is wide and wild, and this layer narrows and cleans the traffic.

---

### DNS (Domain Name System)

**What it is:** The phonebook of the internet. Translates human-readable names like `api.myapp.com` into IP addresses like `13.234.56.78`.

**How it works:**
1. User types `myapp.com` in the browser.
2. Browser asks a **Recursive Resolver** (usually your ISP or Google's `8.8.8.8`).
3. Resolver walks the DNS tree: Root → TLD (`.com`) → Authoritative Name Server for `myapp.com`.
4. Returns the IP. Browser caches it for the TTL (Time To Live) duration.

**Why it matters in system design:**
- **DNS-based load balancing:** Return multiple IPs (round-robin) to spread traffic.
- **GeoDNS:** Return different IPs based on user's location — route Indian users to Mumbai servers, US users to Virginia.
- **Failover:** If your primary server dies, update DNS to point to standby. But watch out for TTL — if TTL is 24h, clients won't see the change for 24 hours.
- **Low TTL = faster failover, higher DNS query load.** High TTL = less DNS traffic, slower failover.

**paperdraw.dev tip:** Place DNS at the very top of your canvas, before everything else. Every user request starts here.

---

### CDN (Content Delivery Network)

**What it is:** A globally distributed network of cache servers (called **Points of Presence / PoPs**) that store copies of your static content (images, JS, CSS, videos) close to users.

**How it works:**
- First user in Tokyo requests `myapp.com/logo.png` → CDN doesn't have it → fetches from your **origin server** → caches it at the Tokyo PoP.
- Second user in Tokyo requests the same file → CDN serves it from Tokyo cache. No trip to your origin.

**Cache invalidation strategies:**
- **TTL-based:** Content expires after N seconds.
- **Versioned URLs:** `logo.v3.png` — new version = new URL = cache miss = fresh content.
- **API invalidation:** Manually purge a CDN key when content changes (e.g., after a CMS publish).

**Why it matters:**
- **Latency:** Serving from 20ms away vs 200ms away is a 10x difference users notice.
- **Cost savings:** Offloads 80–95% of static traffic from your origin.
- **DDoS absorption:** CDNs absorb volumetric attacks because they have massive bandwidth capacity.

**Examples:** Cloudflare, AWS CloudFront, Akamai, Fastly.

---

### Load Balancer

**What it is:** Distributes incoming requests across multiple backend servers so no single server is overwhelmed.

**Two flavours:**
| | L4 Load Balancer | L7 Load Balancer |
|---|---|---|
| Operates at | Transport layer (TCP/UDP) | Application layer (HTTP/HTTPS) |
| Sees | IP addresses, ports | URLs, headers, cookies, body |
| Speed | Extremely fast | Slightly slower (but still ms) |
| Smart routing | No | Yes |
| Use case | Raw throughput, DB connections | HTTP microservices, A/B routing |

**Algorithms:**
- **Round Robin:** Request 1 → Server A, Request 2 → Server B, Request 3 → Server C, repeat.
- **Least Connections:** Route to the server with fewest active connections.
- **IP Hash:** Hash the client IP — same client always hits same server (useful for sessions).
- **Weighted Round Robin:** Server A gets 70% (bigger), Server B gets 30% (smaller).

**Health checks:** The LB periodically pings `/health` on each server. If a server fails 3 checks, it's removed from rotation automatically.

**paperdraw.dev tip:** Place Load Balancer between the CDN/API Gateway and your App Servers.

---

### API Gateway

**What it is:** A single entry point for all API calls. Acts as a "smart reverse proxy" that routes, transforms, and enforces policies on API traffic.

**Core responsibilities:**
- **Routing:** `GET /users` → User Service, `POST /orders` → Order Service.
- **Authentication:** Validates JWT tokens or API keys before forwarding requests.
- **Rate limiting:** Enforce "100 requests per minute per user" centrally.
- **Request/response transformation:** Convert XML to JSON, add/remove headers.
- **SSL termination:** Decrypt HTTPS at the gateway, forward plain HTTP internally (saves CPU on app servers).
- **Caching:** Cache responses for read-heavy endpoints.

**Why not just a Load Balancer?** A Load Balancer just distributes. An API Gateway understands the *content* of requests and applies business logic.

**Examples:** AWS API Gateway, Kong, Nginx Plus, Apigee, Traefik.

---

### WAF (Web Application Firewall)

**What it is:** A filter that inspects HTTP traffic for malicious patterns and blocks attacks before they reach your app.

**What it blocks:**
- **SQL Injection:** `' OR 1=1 --` in query parameters.
- **XSS (Cross-Site Scripting):** `<script>steal_cookie()</script>` in form fields.
- **CSRF:** Cross-site request forgery attempts.
- **OWASP Top 10:** A standard list of the most critical web security risks.

**Modes:**
- **Detection mode:** Logs attacks but doesn't block (use during tuning).
- **Prevention mode:** Blocks matching requests.

**Rule sets:** Managed rule sets (AWS Managed Rules, ModSecurity CRS) give you pre-built protection. Custom rules let you block specific IPs, user agents, or patterns specific to your app.

---

### Ingress (Kubernetes context)

**What it is:** In a Kubernetes cluster, an Ingress is an API object that manages external HTTP/HTTPS access to services running inside the cluster.

**How it works:**
- An **Ingress Controller** (like Nginx Ingress or Traefik) runs as a pod inside the cluster.
- You define **Ingress rules** in YAML: "route `api.myapp.com/users` to the `user-service` on port 8080."
- The controller watches these rules and configures itself accordingly.

**Why it matters:** Without Ingress, you'd need a separate LoadBalancer service (and cloud load balancer) for every single microservice — expensive and unmanageable. Ingress lets one entry point handle routing to hundreds of services.

---

## 2. Network

> **Mental model:** If Traffic & Edge is about getting packets to your datacenter, Network is about routing them intelligently *within* your infrastructure.

---

### Load Balancer (L4/L7)

*(See the detailed breakdown in Section 1 — the same component appears here for its internal network role.)*

In the Network category, L4/L7 load balancers are used for **east-west traffic** (service-to-service) inside a VPC, not just north-south (internet-to-service) traffic.

---

### Reverse Proxy

**What it is:** A server that sits in front of your backend servers and forwards client requests to them. The client never knows which backend server handled its request.

**Forward proxy vs Reverse proxy:**
- **Forward proxy:** Client-side. Sits in front of *clients* (e.g., corporate proxies for internet filtering). The server doesn't know the real client.
- **Reverse proxy:** Server-side. Sits in front of *servers*. The client doesn't know the real server.

**Use cases:**
- **SSL termination** (decrypt HTTPS once, forward HTTP internally)
- **Caching** static responses
- **Compression** (gzip responses)
- **Security** (hide backend server IPs)
- **A/B testing** (route % of traffic to new version)

**Examples:** Nginx, HAProxy, Caddy.

---

### API Gateway (Network)

Same as the Traffic & Edge API Gateway, but placed here to emphasize its role in **internal network routing** — e.g., routing between microservices inside a VPC.

---

### Edge Router

**What it is:** A physical or virtual router at the boundary of your network (the "edge") that connects your infrastructure to the public internet or to other networks.

**Responsibilities:**
- BGP peering with ISPs
- Announcing your IP prefixes to the internet
- First line of defence for routing policies

---

### VPC (Virtual Private Cloud)

**What it is:** A logically isolated section of a cloud provider's network where you launch resources. Think of it as your private datacenter inside AWS/Azure/GCP.

**Key concepts:**
- **CIDR block:** The IP range for your VPC, e.g., `10.0.0.0/16` (65,536 IPs).
- **Subnets:** Sub-divisions of the VPC CIDR.
- **Route tables:** Define where traffic goes.
- **Internet Gateway:** Allows resources in your VPC to communicate with the internet.

**Why isolation matters:** Resources in different VPCs can't talk to each other by default. You must explicitly set up **VPC Peering** or **Transit Gateway** to connect them.

---

### Subnet

**What it is:** A segment of a VPC's IP address range.

**Public subnet vs Private subnet:**
| | Public Subnet | Private Subnet |
|---|---|---|
| Has route to internet | Yes (via Internet Gateway) | No |
| Use for | Load Balancers, Bastion Hosts, NAT Gateways | App Servers, Databases |
| Directly reachable from internet | Yes | No |

**Best practice:** Never put databases in public subnets. App servers should be in private subnets, with only the load balancer in the public subnet.

---

### NAT Gateway

**What it is:** Allows resources in a **private subnet** to initiate *outbound* connections to the internet (e.g., to download packages, call external APIs) without being reachable *inbound* from the internet.

**How it works:**
- App server (private IP `10.0.1.5`) sends a request to `8.8.8.8`.
- NAT Gateway replaces the source IP with its own public IP (Network Address Translation).
- Response comes back to the NAT Gateway, which forwards it to `10.0.1.5`.
- Internet never sees the private IP.

---

### VPN Gateway

**What it is:** Creates an encrypted tunnel between your VPC and another network (your on-premise datacenter, another VPC, or a remote office).

**Site-to-Site VPN:** Connects two entire networks (your office ↔ AWS VPC).
**Client VPN:** Connects individual users' devices to your VPC.

---

### Service Mesh

**What it is:** Infrastructure layer that handles service-to-service communication in microservices architectures. Adds observability, security, and reliability to inter-service calls *without changing application code*.

**How it works (sidecar pattern):**
- A lightweight proxy (called a **sidecar**) is deployed alongside each service pod.
- All network traffic in/out of the pod goes through the sidecar.
- Sidecars handle: mTLS encryption, retries, circuit breaking, rate limiting, distributed tracing.

**Control Plane vs Data Plane:**
- **Data Plane:** The sidecars (e.g., Envoy) — handle actual traffic.
- **Control Plane:** The management layer (e.g., Istio, Linkerd) — configures all sidecars centrally.

**Examples:** Istio, Linkerd, Consul Connect.

---

### DNS Server

*(Internal DNS — see Section 1 for external DNS.)*

Inside a VPC, you run internal DNS so services can find each other by name (`order-service.internal`) rather than IP addresses (which change when pods restart).

**AWS Route 53 Resolver** and **CoreDNS** (used in Kubernetes) are examples.

---

### Discovery Service / Service Discovery

**What it is:** A mechanism for services to find each other in a dynamic environment where service instances start and stop frequently.

**Two patterns:**
- **Client-side discovery:** Service A queries the Service Registry to get the IP list for Service B, then load-balances itself.
- **Server-side discovery:** Service A sends request to a Load Balancer/Router, which queries the registry and forwards.

**Registry database:** Stores the live list of service instances and their health status (e.g., Consul, Eureka, etcd, AWS Cloud Map).

---

### Rate Limiter

**What it is:** Controls the rate at which requests are accepted — protects your system from being overwhelmed.

**Algorithms:**
- **Token Bucket:** A bucket fills with tokens at a fixed rate. Each request consumes one token. If the bucket is empty, request is rejected. Allows short bursts.
- **Leaky Bucket:** Requests enter a queue (bucket) and are processed at a fixed outflow rate. Smoothes out bursts.
- **Fixed Window Counter:** Count requests in fixed time windows (e.g., 100 req/minute). Simple but has boundary burst problem.
- **Sliding Window Log:** Track exact timestamps of requests. Accurate but memory-intensive.
- **Sliding Window Counter:** Hybrid of fixed window and sliding log. Best balance.

**Where to implement:** At API Gateway level (coarse-grained) and within services (fine-grained per user/plan).

---

### Health Check Service

**What it is:** Continuously monitors the health of services and infrastructure components.

**Types:**
- **Active health checks:** Proactively send requests to `/health` endpoints.
- **Passive health checks:** Monitor real traffic — if a server returns too many 5xx errors, mark it unhealthy.

**Health check responses:**
- `200 OK` = Healthy
- `503 Service Unavailable` = Unhealthy (triggers removal from LB rotation)

A good `/health` endpoint checks: database connectivity, dependency availability, memory/CPU thresholds.

---

### Sidecar Proxy

**What it is:** A proxy container deployed alongside your application container in the same pod (Kubernetes) or on the same host.

**What it does:** Intercepts all network traffic to/from the application and adds:
- Mutual TLS (mTLS) for encrypted service-to-service communication
- Distributed tracing headers
- Circuit breaking
- Retry logic

The app code stays clean — it just talks to `localhost`. The sidecar handles all the network complexity.

---

### Security Group

**What it is:** A virtual firewall for cloud resources that controls inbound and outbound traffic at the instance/resource level.

**Key characteristics:**
- **Stateful:** If you allow inbound port 80, the response traffic is automatically allowed out.
- **Whitelist model:** Default-deny. You only specify what's allowed; everything else is blocked.
- **Can reference other security groups:** "Allow inbound from the app-server security group" — no IP needed.

**Example rule set for a database:**
- Inbound: Allow port 5432 from `app-server-sg` only.
- Outbound: Allow all (or restrict to VPC CIDR).

---

### Firewall Rule

**What it is:** Rules at the network level (not instance level) that control traffic based on IP, port, and protocol.

**Network ACLs (in AWS context):**
- Applied to subnets, not individual instances.
- **Stateless:** You must explicitly allow both inbound and outbound.
- Evaluated in rule number order; first match wins.

---

### Routing Table

**What it is:** A set of rules in a VPC that determines where network traffic is directed.

**Common routes:**
- `10.0.0.0/16` → `local` (stay within the VPC)
- `0.0.0.0/0` → `igw-xxxx` (send everything else to the internet gateway)
- `10.1.0.0/16` → `pcx-xxxx` (send to peered VPC)

---

### Routing Policy

**What it is:** Higher-level traffic routing decisions, often at DNS or load balancer level.

**Common DNS routing policies (AWS Route 53 examples):**
- **Simple:** One record, one destination.
- **Weighted:** 80% to v1, 20% to v2 (canary deployments).
- **Latency-based:** Route to the AWS region with lowest latency for the user.
- **Failover:** Primary target normally; failover to secondary if primary health check fails.
- **Geolocation:** Route Indian users to `ap-south-1`, US users to `us-east-1`.

---

## 3. Compute

> **Mental model:** Compute is where your actual business logic runs. These are the workers — they receive requests, process them, and produce results.

---

### App Server

**What it is:** The workhorse of your backend. Receives HTTP requests, executes business logic, reads/writes to databases, and returns responses.

**Stateless vs stateful:**
- **Stateless (preferred for scale):** The server holds no per-user session state. Any server can handle any request. Scale horizontally by adding more instances.
- **Stateful:** Session data lives on a specific server. Harder to scale — you must use sticky sessions or externalize state to Redis.

**Scaling patterns:**
- **Horizontal scaling:** Add more servers (scale out). Load balancer distributes requests.
- **Vertical scaling:** Give the server more CPU/RAM (scale up). Has limits.

---

### Worker

**What it is:** A background process that executes tasks from a queue, without serving HTTP requests directly.

**When to use workers:**
- Sending emails (don't block the HTTP response while waiting for SMTP)
- Processing images/videos (CPU-intensive, can take minutes)
- Generating reports
- Any task that can be done asynchronously

**Pattern:**
1. API Server receives request, drops a job onto a Message Queue, returns `202 Accepted` immediately.
2. Worker picks up the job, processes it.
3. Worker updates the database or notifies via WebSocket/push.

---

### Serverless (FaaS — Functions as a Service)

**What it is:** Run code without managing servers. You deploy a function; the cloud provider handles provisioning, scaling, and billing (per invocation).

**Characteristics:**
- **Event-driven:** Triggered by HTTP, queue messages, cron schedules, file uploads, database changes.
- **Stateless:** Each invocation is independent.
- **Cold starts:** First invocation after idle period has extra latency (~100ms–1s) while the runtime spins up.
- **Short-lived:** Max execution time (e.g., 15 minutes on AWS Lambda).

**Good use cases:** Infrequent workloads, event processing, webhooks, scheduled tasks.
**Bad use cases:** Long-running processes, latency-sensitive hot paths, high-state applications.

**Examples:** AWS Lambda, Azure Functions, Google Cloud Functions.

---

### Auth Service

**What it is:** A dedicated service responsible for authentication (who are you?) and authorization (what can you do?).

**Authentication methods:**
- **Username/Password:** Classic. Hash passwords with bcrypt/Argon2 (never MD5/SHA1).
- **OAuth 2.0:** Delegate auth to a third party (Google, GitHub). Your app gets an access token.
- **JWT (JSON Web Tokens):** Self-contained token. Contains user ID, roles, expiry. Signed with a secret key. Backend can verify without hitting a database.
- **API Keys:** Long-lived tokens for service-to-service or developer access.

**JWT anatomy:** `header.payload.signature`
- Header: algorithm used (HS256, RS256).
- Payload: claims (user_id, roles, exp).
- Signature: HMAC of header+payload. Cannot be tampered without detection.

**Session vs Token:**
- Sessions: State stored server-side (in Redis). Scalable but requires shared storage.
- JWT: Stateless. No server storage needed. But can't revoke individual tokens without a blocklist.

---

### Notification Service

**What it is:** Centralized service for sending notifications across multiple channels.

**Channels:**
- **Push notifications:** APNs (Apple), FCM (Firebase/Google).
- **Email:** SMTP, SES, SendGrid.
- **SMS:** Twilio, SNS.
- **WebSockets:** Real-time in-app notifications.
- **Webhooks:** HTTP callbacks to external systems.

**Design considerations:**
- **Deduplication:** Don't send the same notification twice if the upstream retries.
- **Rate limiting per user:** Don't spam users with 100 emails in a minute.
- **Priority queues:** Critical alerts (OTP) bypass the queue, marketing emails are batched.

---

### Search Service

**What it is:** A service optimized for full-text search, faceted filtering, and relevance ranking — things relational databases are terrible at.

**How it works:**
- Data is indexed in an **inverted index** (for each word, which documents contain it?).
- Queries are scored by **relevance** (TF-IDF, BM25).
- Supports fuzzy matching, synonyms, autocomplete.

**Sync pattern:** Your DB is the source of truth → changes are pushed (via CDC or event stream) to the search index.

**Examples:** Elasticsearch, OpenSearch, Apache Solr, Algolia, Typesense.

---

### Analytics Service

**What it is:** Collects, processes, and serves analytical queries on large datasets — typically read-heavy, aggregation-heavy workloads.

**OLTP vs OLAP:**
| | OLTP (Transactional) | OLAP (Analytical) |
|---|---|---|
| Use case | Day-to-day operations | Business intelligence, reporting |
| Query type | Simple, fast (ms) | Complex, slow (seconds-minutes) |
| Data size | Row-level | Aggregates over millions of rows |
| Database | PostgreSQL, MySQL | Redshift, BigQuery, ClickHouse |

---

### Scheduler

**What it is:** Triggers jobs at defined times or intervals.

**Types:**
- **Cron-based:** Run every day at 2 AM. Standard cron syntax: `0 2 * * *`.
- **Interval-based:** Run every 5 minutes.
- **One-time:** Run once at a future timestamp.

**Challenge in distributed systems:** Multiple instances of a scheduler can fire the same job simultaneously. Use **distributed locks** (Redis, ZooKeeper) or a **leader election** pattern to ensure exactly-one execution.

**Examples:** Kubernetes CronJob, AWS EventBridge Scheduler, Quartz Scheduler, Celery Beat.

---

### Config Service

**What it is:** Centralized store for application configuration values, allowing config changes without redeployment.

**Why externalize config?**
- Same code deployed to dev/staging/prod with different config values.
- Change feature flags or rate limits without a code deploy.
- Audit trail of config changes.

**Examples:** AWS AppConfig, HashiCorp Consul KV, Spring Cloud Config, etcd.

---

### Secrets Manager

**What it is:** Secure storage and rotation of sensitive credentials: database passwords, API keys, TLS certificates.

**Never do this:** Hardcode secrets in code or commit them to git. Even in environment variables is risky.

**Features:**
- **Encryption at rest and in transit.**
- **Access policies:** Only specific services/roles can read specific secrets.
- **Automatic rotation:** Secrets Manager can automatically rotate DB passwords and update all applications.
- **Audit logging:** Who accessed which secret and when.

**Examples:** AWS Secrets Manager, HashiCorp Vault, Azure Key Vault.

---

### Feature Flags

**What it is:** Runtime toggles that enable/disable features without code deployment.

**Use cases:**
- **Dark launches:** Deploy code to production but keep feature off. Turn on for internal users first.
- **Canary releases:** Enable feature for 1% of users, monitor errors, gradually roll out.
- **Kill switches:** Instantly disable a buggy feature without rolling back a deployment.
- **A/B testing:** Route users to variant A or B to measure impact.

**Examples:** LaunchDarkly, AWS AppConfig feature flags, Unleash, Flipt.

---

### Metrics Collector Agent

**What it is:** A lightweight agent running on each host/container that collects system and application metrics and ships them to a central metrics store.

**What it collects:**
- System: CPU, memory, disk I/O, network I/O.
- Application: Request rate, error rate, latency (p50/p95/p99).
- JVM/runtime metrics for Java apps.

**Examples:** Prometheus Node Exporter, CloudWatch Agent, Datadog Agent, Telegraf.

---

### Log Collector Agent

**What it is:** Collects logs from files, stdout, systemd journal, and ships them to a central log aggregation service.

**Examples:** Fluentd, Fluent Bit, Filebeat, Logstash.

---

### Log Aggregation Service

**What it is:** Central platform that receives logs from all services, indexes them, and makes them searchable.

**The ELK/EFK Stack:**
- **E**lasticsearch (or OpenSearch) — storage and search
- **L**ogstash or **F**luent Bit — collection and processing
- **K**ibana — visualization and dashboards

**Why aggregate?** In a microservices system with 50 services and 100 pods, logs are scattered everywhere. Aggregation gives you one place to search across all logs, correlate events, and debug issues.

---

### Distributed Tracing Collector

**What it is:** Tracks a single request as it flows through multiple microservices.

**How it works:**
- Each request gets a unique **Trace ID**.
- Each service-to-service call creates a **Span** (with start time, end time, metadata).
- All spans are sent to the tracing collector, which assembles the full **Trace**.
- You can visualize a waterfall diagram showing time spent in each service.

**Why it matters:** Without tracing, if a request takes 800ms and you have 10 services, you have no idea which service is slow.

**Examples:** Jaeger, Zipkin, AWS X-Ray, Tempo.

---

### Alerting Engine

**What it is:** Evaluates metric and log data against rules and fires alerts when thresholds are breached.

**Alert types:**
- **Threshold alerts:** CPU > 90% for 5 minutes.
- **Anomaly detection:** Traffic is 3 standard deviations below normal.
- **Absence alerts:** No logs received from service X in 10 minutes.
- **Error rate alerts:** Error rate > 1% on `/checkout` endpoint.

**SLA-driven alerting:** Instrument based on **SLOs (Service Level Objectives)** — e.g., "p99 latency must be < 500ms" — and alert when the SLO is at risk.

**Examples:** Alertmanager (Prometheus), PagerDuty, OpsGenie.

---

### Health Check Monitor

**What it is:** Periodically pings services and infrastructure to verify they are alive and healthy, feeding data to load balancers and alerting systems.

Separate from the Alerting Engine — the monitor *observes*, the alerting engine *reacts*.

---

### Media Processor

**What it is:** Handles compute-intensive media operations: video transcoding, image resizing, thumbnail generation, format conversion.

**Architecture pattern:**
1. User uploads raw video to Object Store (S3).
2. Upload event triggers a queue message.
3. Media Processor Worker picks it up, transcodes to multiple resolutions (360p, 720p, 1080p).
4. Outputs stored back in Object Store.
5. CDN serves the transcoded content.

**Why async?** Transcoding a 1-hour video takes 10+ minutes. You cannot block an HTTP request for that long.

---

## 4. Storage

> **Mental model:** Not all data is the same. Choose storage based on *how* you query the data, not just *what* you store.

---

### Cache

**What it is:** A fast, in-memory store that holds frequently accessed data to avoid repeated expensive database queries.

**Cache patterns:**
- **Cache-aside (Lazy loading):** App checks cache first → miss → read from DB → populate cache → return.
- **Write-through:** Write to cache and DB simultaneously. Cache always consistent.
- **Write-behind:** Write to cache immediately, flush to DB asynchronously. Faster writes, risk of data loss.
- **Read-through:** Cache sits in front of DB. Cache handles the DB read on miss.

**Cache eviction policies:**
- **LRU (Least Recently Used):** Evict the item that hasn't been accessed for the longest time.
- **LFU (Least Frequently Used):** Evict the item accessed least often.
- **TTL-based:** Evict after a time-to-live regardless of usage.

**Cache invalidation — the hard problem:** When does cached data become stale? Strategies: TTL, explicit invalidation on write, event-driven invalidation.

**Examples:** Redis, Memcached.

---

### Database (Relational / SQL)

**What it is:** A structured store that organizes data into tables with rows and columns, with strict schema and ACID guarantees.

**ACID:**
- **Atomicity:** Transaction either fully succeeds or fully fails.
- **Consistency:** Database always goes from one valid state to another.
- **Isolation:** Concurrent transactions behave as if sequential.
- **Durability:** Committed transactions survive crashes.

**Indexing:** B-tree indexes allow O(log n) lookups instead of O(n) full scans. But indexes slow down writes — add indexes thoughtfully.

**Replication:**
- **Primary-Replica:** All writes go to primary, reads distributed across replicas.
- **Multi-Primary:** Multiple writable nodes. Conflict resolution needed.

**Examples:** PostgreSQL (preferred for new projects), MySQL, Aurora.

---

### Object Store

**What it is:** Stores unstructured binary data (files, images, videos, backups) as objects with metadata, accessible via HTTP APIs.

**Key concepts:**
- **Object = data + metadata + unique key.**
- **Flat namespace (no real folders)** — but you can simulate folders with key prefixes (`user/123/profile.jpg`).
- **Infinitely scalable** — designed to store petabytes.
- **Immutable** — you don't update objects in-place; you upload a new version.

**Use cases:** User uploads, static website hosting, backups, data lake raw storage.

**Examples:** AWS S3, GCS, Azure Blob Storage, MinIO.

---

### KV Store (Key-Value Store)

**What it is:** The simplest data model — a dictionary. Given a key, get a value. No query language, no joins.

**Why use it?**
- Extremely fast (O(1) lookups)
- Simple mental model
- Horizontal scaling is easy (partition by key)

**Use cases:** Session storage, user preferences, rate limiting counters, feature flag state, distributed locks.

**Examples:** Redis (also a cache), DynamoDB, etcd.

---

### Time Series DB

**What it is:** A database optimized for storing and querying data points indexed by timestamp.

**What makes it special:**
- **Write-optimized:** Metrics come in at millions of data points per second.
- **Time-range queries:** "Give me CPU usage for the last 6 hours."
- **Downsampling/rollup:** Automatically compress old data (1-second resolution → 1-minute average after 7 days).
- **Retention policies:** Automatically delete data older than N days.

**Use cases:** Infrastructure metrics, IoT sensor data, financial tick data, application performance monitoring.

**Examples:** InfluxDB, TimescaleDB, Prometheus TSDB, VictoriaMetrics.

---

### Graph DB

**What it is:** Stores data as nodes (entities) and edges (relationships), optimized for traversing connected data.

**When relational databases struggle:**
- "Find all friends-of-friends of user X who are also in user X's city and share 3+ mutual interests."
- In a relational DB, this is a series of expensive JOINs that scale poorly.
- In a graph DB, it's a natural traversal.

**Use cases:** Social networks, recommendation engines, fraud detection (detect suspicious connection patterns), knowledge graphs, identity & access management.

**Examples:** Neo4j, Amazon Neptune, TigerGraph.

---

### Vector DB

**What it is:** Stores high-dimensional vectors (numerical embeddings) and supports **nearest-neighbour search** — "find the N most similar vectors to this one."

**Why it matters for AI:**
- LLMs convert text to vectors (embeddings).
- To give an LLM "memory" or do semantic search, you store embeddings in a vector DB and search for semantically similar content.
- Powers **RAG (Retrieval-Augmented Generation)** systems.

**ANN algorithms:** HNSW, FAISS, IVF. Approximate nearest neighbour — fast but not always 100% accurate. The tradeoff between accuracy and speed is tunable.

**Examples:** Pinecone, Weaviate, Qdrant, pgvector (PostgreSQL extension), Chroma.

---

### Search Index

**What it is:** A pre-processed data structure that enables fast full-text search. The search service (Section 3) writes to this index.

**Inverted index structure:**
- Word → [list of documents containing that word + frequency/position]
- Query "order status" → look up "order" (docs: 1,5,7) and "status" (docs: 2,5,8) → intersection: doc 5.

---

### Data Warehouse

**What it is:** A central repository for structured, historical, and integrated data from multiple sources — designed for analytical queries by business intelligence tools.

**Characteristics:**
- **Column-oriented storage:** Storing data column-by-column (not row-by-row) makes aggregations (`SUM`, `AVG`, `COUNT`) extremely fast.
- **Schema-on-write:** Schema defined upfront.
- **Batch-loaded:** Data loaded periodically (hourly, daily).
- **Read-optimized:** Not for transactional writes.

**Examples:** Amazon Redshift, Google BigQuery, Snowflake, ClickHouse.

---

### Data Lake

**What it is:** A massive, cheap storage repository that stores raw data in its native format (files, JSON, CSV, Parquet) at scale — *before* you know how you'll use it.

**Data Lake vs Data Warehouse:**
| | Data Lake | Data Warehouse |
|---|---|---|
| Data format | Raw (JSON, CSV, images, logs) | Structured (tables) |
| Schema | Schema-on-read | Schema-on-write |
| Storage cost | Cheap (object store) | Expensive |
| Query flexibility | Very flexible | Structured SQL only |
| Typical users | Data scientists, ML engineers | Business analysts |

**The Lakehouse architecture:** Combines the best of both — raw storage of a lake with the query performance of a warehouse (Delta Lake, Apache Iceberg).

---

## 5. Messaging

> **Mental model:** Messaging decouples producers from consumers. Producer doesn't care if the consumer is up, slow, or doesn't exist yet. This is the backbone of resilient, async architectures.

---

### Message Queue

**What it is:** A durable buffer where producers push messages and consumers pull them. Each message is processed by **exactly one consumer** (point-to-point).

**Key concepts:**
- **Producer:** Publishes messages to the queue.
- **Consumer:** Reads and processes messages.
- **Acknowledgement (ACK):** Consumer tells the queue "I've processed this message." Only then is it deleted.
- **Dead Letter Queue (DLQ):** Messages that fail processing N times are moved here for investigation.
- **Visibility timeout:** After a consumer picks up a message, it becomes invisible to others for a period. If not ACKed in time, it becomes visible again for retry.

**Use cases:** Email sending, order processing, payment processing, background job execution.

**Examples:** AWS SQS, RabbitMQ, ActiveMQ.

---

### Pub/Sub (Publish-Subscribe)

**What it is:** A messaging pattern where producers publish to **topics** and multiple consumers can subscribe. Each subscriber gets a copy of every message.

**Key difference from Queue:** Queue = one consumer per message. Pub/Sub = many consumers per message.

**Use cases:**
- Event-driven architectures: "Order Placed" event triggers: inventory service, email service, analytics service simultaneously.
- Real-time notifications.
- System-wide broadcasts (e.g., config updates).

**Examples:** Google Cloud Pub/Sub, AWS SNS, Redis Pub/Sub, NATS.

---

### Stream

**What it is:** An ordered, persistent, replayable log of events. Unlike a queue (delete-on-ACK), events in a stream are **retained** for a configurable period and can be replayed.

**Key concepts:**
- **Partition/Shard:** A stream is split into partitions for parallelism. Events with the same key go to the same partition (ordering guarantee within a partition).
- **Consumer Group:** A group of consumers that collectively process a stream. Each partition is assigned to one consumer in the group.
- **Offset:** Each event has a sequential number (offset). Consumers track their offset — they can rewind and re-process.

**Use cases:** Event sourcing, CDC (change data capture), real-time analytics, audit logs, microservices communication at scale.

**Examples:** Apache Kafka, AWS Kinesis, Azure Event Hubs, Redpanda.

---

## 6. Fintech & Banking

> **Mental model:** In fintech, correctness > performance. The most important words are *idempotency*, *consistency*, and *auditability*.

---

### Payment Gateway

**What it is:** The intermediary that facilitates payment transactions between the customer, your application, the payment networks (Visa/Mastercard), and the banks.

**Payment flow:**
1. Customer enters card details on your frontend.
2. Details are tokenized (never sent to your server raw — PCI DSS compliance).
3. Your server sends a charge request to the Payment Gateway API.
4. Gateway routes to the card network → customer's issuing bank → authorization response.
5. Gateway returns `approved` or `declined` to your server.
6. At end of day, settlement occurs — funds move.

**Idempotency is critical:** Network failures can cause duplicate requests. Always use idempotency keys so retrying a payment charge doesn't charge the customer twice.

**Examples:** Stripe, Razorpay (India-focused), Braintree, Adyen.

---

### Ledger Service

**What it is:** A double-entry bookkeeping system that records all financial transactions with complete auditability.

**Double-entry principle:** Every financial transaction has two sides — a debit and a credit. They must always balance. This is fundamental accounting.

**Ledger design principles:**
- **Append-only:** Never update or delete ledger entries. If you make an error, record a correcting entry.
- **Immutable records:** Historical state must be always reproducible.
- **Balance verification:** Sum of all debits must equal sum of all credits.

**Modern implementation:** Event-sourced ledger — the ledger is a stream of immutable events. Current balance is derived by replaying events.

---

### Fraud Detection

**What it is:** Real-time system that scores transactions for fraudulent intent and blocks or flags suspicious activity.

**Detection approaches:**
- **Rule-based:** If transaction > $10,000 AND country ≠ user's home country AND time = 3 AM, flag it.
- **ML-based:** Train models on historical fraud patterns. Score every transaction in real-time (<100ms).
- **Graph-based:** Detect fraud rings by identifying suspicious patterns in transaction graphs (multiple accounts sharing a device, card, or address).

**Challenge:** Balancing false positives (blocking legitimate transactions = angry customers) vs. false negatives (missing fraud = financial loss).

---

### HSM (Hardware Security Module)

**What it is:** A physical hardware device designed to securely generate, store, and manage cryptographic keys. Keys never leave the HSM in plaintext.

**Why hardware?** Software-based key storage can be extracted by a compromised OS. An HSM provides a physical tamper-resistant boundary.

**Use cases:**
- Storing root CA private keys.
- Signing JWTs or documents with guaranteed key security.
- PCI DSS requirements for payment card encryption.

**Examples:** AWS CloudHSM, Thales Luna HSM.

---

## 7. Security

---

### DDoS Shield

**What it is:** Specialized infrastructure that absorbs and filters Distributed Denial-of-Service attacks — attempts to overwhelm your system with fake traffic.

**DDoS types:**
- **Volumetric:** Flood with massive amounts of traffic (UDP flood, ICMP flood). Measured in Gbps.
- **Protocol:** Exploit protocol weaknesses (SYN flood — exhaust TCP connection table). Measured in Mpps.
- **Application layer (L7):** Send legitimate-looking HTTP requests at scale (HTTP flood). Measured in RPS.

**Defense mechanisms:**
- **Traffic scrubbing:** Route all traffic through scrubbing centers that identify and drop attack traffic.
- **Anycast diffusion:** Spread attack traffic across many PoPs so no single one is overwhelmed.
- **Rate limiting and CAPTCHAs** for application-layer attacks.
- **BGP blackholing:** In extreme cases, null-route the attack traffic at the ISP level.

**Examples:** Cloudflare DDoS Protection, AWS Shield, Akamai Prolexic.

---

### SIEM (Security Information and Event Management)

**What it is:** Platform that aggregates security logs from across your infrastructure, correlates them, detects threats, and helps with incident response and compliance.

**What it does:**
- **Collect:** Ingest logs from firewalls, servers, applications, IAM systems, network devices.
- **Normalize:** Convert all log formats into a common schema.
- **Correlate:** "User logged in from Russia, then 10 seconds later from India — impossible travel, flag it."
- **Alert:** Notify security team of detected threats.
- **Store:** Retain logs for compliance (SOC2, PCI DSS, HIPAA typically require 1+ year retention).

**Examples:** Splunk, IBM QRadar, Microsoft Sentinel, Elastic SIEM.

---

## 8. Data Engineering

> **Mental model:** Raw data is useless. Data engineering transforms raw events into clean, structured, queryable datasets that power business decisions and ML models.

---

### ETL Pipeline (Extract, Transform, Load)

**What it is:** A process that extracts data from source systems, transforms it (clean, enrich, aggregate), and loads it into a destination (data warehouse, data lake).

**ETL vs ELT:**
- **ETL:** Transform before loading. Traditional approach. Source → Transform → Warehouse.
- **ELT:** Load raw first, transform inside the warehouse. Modern approach enabled by cheap columnar storage. Source → Data Lake → Transform in Warehouse.

**Transformation examples:**
- Normalize date formats.
- Enrich orders with product category from a lookup table.
- Aggregate daily transactions into weekly summaries.
- Deduplicate records.

**Tools:** Apache Spark, dbt (data build tool), AWS Glue, Airflow, Fivetran.

---

### CDC Service (Change Data Capture)

**What it is:** Captures every INSERT, UPDATE, DELETE from a database and streams those changes in real-time to downstream systems.

**How it works:**
- Reads the database's **transaction log** (WAL in PostgreSQL, binlog in MySQL) — the log used for crash recovery.
- Extracts change events without adding load to the DB.
- Streams to Kafka, which fans out to data warehouses, search indexes, caches, etc.

**Why it's powerful:** Keep all downstream systems (analytics, search, cache) in sync with the primary database in near-real-time, without tightly coupling them.

**Examples:** Debezium, AWS DMS (Database Migration Service).

---

### Schema Registry

**What it is:** A centralized repository for event/message schemas, ensuring producers and consumers agree on data formats.

**The problem it solves:** Service A produces events in format V1. Service B consumes them. Service A "upgrades" the schema (removes a field) without telling Service B. Service B crashes.

**Solution:** Both services agree on a schema. Schema Registry enforces compatibility rules:
- **Backward compatibility:** New schema can read data written with old schema.
- **Forward compatibility:** Old schema can read data written with new schema.
- **Full compatibility:** Both directions work.

**Examples:** Confluent Schema Registry (Avro/Protobuf), AWS Glue Schema Registry.

---

### Batch Processor

**What it is:** Processes large volumes of data in scheduled batches rather than in real-time.

**When to use batch vs stream:**
| | Batch | Stream |
|---|---|---|
| Latency | Minutes to hours | Milliseconds to seconds |
| Use case | End-of-day reports, ML training | Fraud detection, real-time dashboards |
| Efficiency | Very high (process in bulk) | Lower (process each event) |
| Complexity | Lower | Higher |

**Examples:** Apache Spark (batch mode), AWS Batch, Hadoop MapReduce.

---

### Feature Store

**What it is:** A centralized platform for storing, managing, sharing, and serving ML features — the transformed data inputs used to train and serve ML models.

**Problem it solves:** Data scientist A computes "user's average transaction value in last 30 days" for their fraud model. Data scientist B computes the same feature for their churn model — duplicated work, possibly different implementations.

**Two serving modes:**
- **Offline store:** Historical features for model training. Stored in a data warehouse.
- **Online store:** Low-latency feature serving for real-time inference. Stored in a KV store (Redis).

**Examples:** Feast, Tecton, AWS SageMaker Feature Store, Vertex AI Feature Store.

---

## 9. My Services

---

### Users Service

**What it is:** Manages user identity and profile data — creation, authentication, profile updates, account management.

**Typical responsibilities:**
- User registration and email verification.
- Password management (hashing with bcrypt, reset flows).
- Profile CRUD operations.
- Integrating with Auth Service for token issuance.
- Managing user preferences and settings.

**Database considerations:**
- Primary key: UUID (not auto-increment integers — avoids enumeration attacks).
- Index on email (unique) for login lookups.
- Soft deletes: Set `deleted_at` timestamp instead of hard-deleting user records.

---

### Service

**What it is:** A generic placeholder for any domain-specific microservice in your architecture. In paperdraw.dev, this represents your custom business logic services.

**Microservice design principles:**
- **Single Responsibility:** Each service owns one bounded context (Orders, Inventory, Payments).
- **Own your data:** Each service has its own database. Never share a database between services.
- **API contracts:** Services communicate via well-defined APIs (REST, gRPC, events).
- **Independent deployability:** You can deploy Service A without deploying Service B.

---

## 10. AI & Agents

> **Mental model:** These components form the infrastructure layer for building AI-powered applications and autonomous agents.

---

### LLM Gateway

**What it is:** A proxy layer between your application and LLM providers (OpenAI, Anthropic, Google). Similar to an API Gateway but specifically for LLM traffic.

**Features:**
- **Provider abstraction:** Switch between OpenAI and Anthropic without changing application code.
- **Cost tracking:** Track token usage and costs per service/user.
- **Rate limiting:** Enforce LLM API quotas centrally.
- **Caching:** Exact-match caching for identical prompts saves API calls and cost.
- **Semantic caching:** Cache responses for semantically similar (not just identical) prompts.
- **Fallback:** If OpenAI is down, route to Anthropic automatically.
- **PII filtering:** Strip sensitive data from prompts before sending to external LLMs.

**Examples:** LiteLLM, PortKey, OpenRouter.

---

### Tool Registry

**What it is:** A catalogue of tools (functions, APIs, services) available to AI agents. When an agent needs to take an action, it consults the registry to discover what tools are available and how to use them.

**What a tool definition contains:**
- Tool name and description (the LLM reads this to decide when to use it).
- Input schema (what parameters does it accept?).
- Output schema (what does it return?).
- Authentication requirements.

**Examples:** Used in OpenAI Function Calling, LangChain tools, Anthropic tool use.

---

### Memory Fabric

**What it is:** Infrastructure for AI agent memory — allowing agents to remember information across conversations and sessions.

**Types of memory:**
- **In-context (short-term):** The current conversation window. Limited by context length.
- **External (long-term):** Information stored outside the model. Retrieved and injected into context when relevant.
  - **Episodic memory:** Past conversations and events.
  - **Semantic memory:** Facts about the user, world knowledge.
  - **Procedural memory:** How to perform certain tasks (stored prompts/workflows).

**Architecture:** Events → Embedding Model → Vector DB (semantic search) → Retrieved context injected into prompts.

---

### Agent Orchestrator

**What it is:** The control system that manages multi-step agent workflows — planning, tool use, memory retrieval, and coordination between multiple agents.

**ReAct pattern (Reasoning + Acting):**
1. **Thought:** Agent reasons about the task.
2. **Action:** Agent calls a tool.
3. **Observation:** Agent receives the tool's result.
4. **Repeat** until the task is complete.

**Multi-agent patterns:**
- **Sequential:** Agent A's output is Agent B's input.
- **Parallel:** Multiple agents work simultaneously on sub-tasks.
- **Hierarchical:** Orchestrator agent delegates to specialist sub-agents.
- **Debate:** Multiple agents argue different positions; a judge agent decides.

**Examples:** LangGraph, CrewAI, AutoGen, Semantic Kernel.

---

### Safety & Observability

**What it is:** Guardrails and monitoring specifically for AI systems.

**Safety components:**
- **Input guardrails:** Filter harmful, off-topic, or policy-violating inputs before they reach the LLM.
- **Output guardrails:** Check LLM responses for toxicity, hallucinations, PII leakage, off-brand content.
- **Constitutional AI / RLHF:** Training-time alignment techniques.

**AI observability:**
- **Prompt logging:** Log all prompts and responses for debugging and compliance.
- **Hallucination detection:** Fact-check LLM outputs against ground truth.
- **Latency and cost tracking:** p50/p95 latency per LLM call, cost per feature.
- **Drift detection:** Is the model behaving differently over time?

**Examples:** Guardrails AI, LlamaGuard, LangSmith, Weights & Biases.

---

## 11. Techniques

> **Mental model:** These are architectural strategies — not components you deploy but patterns you apply when designing how data is distributed, replicated, and processed.

---

### Sharding

**What it is:** Horizontally partitioning data across multiple database servers (shards). Each shard holds a subset of the data.

**When to shard:** Single database can't handle the data volume or write throughput even after vertical scaling and read replicas.

**Sharding strategies:**
- **Range-based:** User IDs 1–1M → Shard A, 1M–2M → Shard B. Simple, but can create hotspots.
- **Hash-based:** `shard = hash(user_id) % num_shards`. Evenly distributed, but range queries are hard.
- **Directory-based:** A lookup table maps each record to its shard. Flexible but the directory is a bottleneck.

**Challenges:**
- **Cross-shard queries:** JOINs across shards are expensive or impossible.
- **Rebalancing:** Adding a new shard requires re-distributing data. Consistent hashing minimizes this.
- **Hotspots:** A "celebrity" user or viral post may overload one shard.

---

### Hashing

**What it is:** A function that maps input data to a fixed-size output (hash). Used for data distribution, caching, and data integrity.

**Consistent Hashing:**
The most important hashing technique for distributed systems.

**Problem with naive hashing:** `shard = hash(key) % N`. If N changes (add/remove a server), almost all keys need to be remapped — massive disruption.

**Consistent hashing solution:**
- Both servers and keys are placed on a virtual ring (0 to 2^32).
- A key is assigned to the first server clockwise from its position on the ring.
- When a server is added/removed, only the keys between it and its predecessor are remapped.
- Adding a server → only ~1/N of keys are remapped.

**Virtual nodes:** Each physical server is represented by multiple points on the ring — prevents uneven distribution.

---

### Shard Node

**What it is:** A physical server or database instance that holds one shard of data.

In paperdraw.dev, when you show data sharding, you'll represent each shard with a Shard Node component, typically connected to a Routing/Proxy layer that directs queries to the correct shard.

---

### Partition Node

**What it is:** In the context of streaming systems (Kafka), a partition is an ordered, immutable sequence of records within a topic. A Partition Node represents one such partition.

**Key properties:**
- Records within a partition are strictly ordered.
- Each partition is replicated across N broker nodes (configurable).
- Each partition is consumed by exactly one consumer in a consumer group.
- Increasing partition count = more parallelism.

---

### Replica Node

**What it is:** A copy of a data node (database, partition, shard) for redundancy and read scaling.

**Replication types:**
- **Synchronous:** Primary waits for replica to confirm write before acknowledging client. Strong consistency, higher latency.
- **Asynchronous:** Primary acknowledges immediately, replicates in background. Lower latency, possible data loss if primary crashes.
- **Semi-synchronous:** Wait for at least one replica. Balance of both.

**Replica lag:** The delay between a write on the primary and its appearance on the replica. Reading from a lagging replica = stale data. Important to account for in systems where consistency matters.

---

### Input Source

**What it is:** The origin of data entering your pipeline. In data engineering and stream processing architectures, this represents where events/records originate.

**Examples:**
- User clicks (web/mobile events)
- IoT sensors
- Database change events (CDC)
- Application logs
- External API webhooks

---

### Output Sink

**What it is:** The destination where processed data lands after going through your pipeline.

**Examples:**
- Data Warehouse (for analytics)
- Database (updated state)
- Search Index (updated search)
- Message Queue (for downstream services)
- Object Store (for archival)
- Dashboard / Real-time visualization

---

## 12. Week-1 Study Plan & Design Challenges

### Day-by-Day Plan

| Day | Focus | Chapters to Re-read | Design Challenge |
|-----|-------|---------------------|------------------|
| **Day 1** | Traffic & Edge + Network basics | Sections 1 & 2 | Design a URL shortener (tinyurl.com clone) |
| **Day 2** | Compute + Storage | Sections 3 & 4 | Design Instagram (image upload, feed, CDN) |
| **Day 3** | Messaging + Data Engineering | Sections 5 & 8 | Design a notification system |
| **Day 4** | Fintech + Security | Sections 6 & 7 | Design a payment processing system |
| **Day 5** | AI & Agents | Section 10 | Design a RAG-based customer support bot |
| **Day 6** | Techniques (Sharding, Hashing, Replicas) | Section 11 | Design Twitter/X at 500M users |
| **Day 7** | Full system review + paperdraw.dev free build | All sections | Design YouTube from scratch |

---

### 5 Classic System Design Blueprints for paperdraw.dev

#### Blueprint 1: Web Application (Standard 3-Tier)
```
User → DNS → CDN → WAF → Load Balancer (L7) → App Server (Auto-scaled)
                                                      ↓
                                              Cache (Redis)
                                                      ↓
                                              Database (Primary + Replica)
                                                      ↓
                                              Object Store (S3)
```

#### Blueprint 2: Microservices with Async Processing
```
User → API Gateway → Auth Service → [Order Service | User Service | Product Service]
                                           ↓
                                    Message Queue
                                           ↓
                                    [Email Worker | Notification Worker | Analytics Worker]
```

#### Blueprint 3: Event-Driven Data Pipeline
```
App DB → CDC Service → Stream (Kafka) → ETL Pipeline → Data Warehouse
                                    ↓
                             Search Index (Elasticsearch)
                                    ↓
                             Cache (Redis - for hot data)
```

#### Blueprint 4: AI Agent System
```
User → API Gateway → LLM Gateway → Agent Orchestrator
                                          ↓
                          [Tool Registry | Memory Fabric | Safety & Observability]
                                          ↓
                    [Web Search Tool | DB Query Tool | Email Tool]
```

#### Blueprint 5: Fintech Payment Flow
```
User → WAF → API Gateway → Auth Service → Payment Gateway
                                                 ↓
                                          Ledger Service (append-only)
                                                 ↓
                                          Fraud Detection (real-time ML)
                                                 ↓
                                          Message Queue → Notification Service
```

---

### Key Principles to Internalize

**CAP Theorem:** In a distributed system, you can only guarantee 2 of 3: **C**onsistency, **A**vailability, **P**artition tolerance. Since network partitions always happen, you choose between CP (banks) or AP (social networks).

**SLA / SLO / SLI:**
- **SLI (Indicator):** The actual measured metric (e.g., p99 latency = 230ms).
- **SLO (Objective):** The target (e.g., p99 latency < 300ms, 99.9% of the time).
- **SLA (Agreement):** The contractual promise with consequences if broken.

**The "Fallacies of Distributed Computing":** Network is reliable, latency is zero, bandwidth is infinite, network is secure, topology doesn't change, there is one administrator, transport cost is zero, network is homogeneous. These are all *false* — your designs must account for each one.

**Back-of-envelope estimation (for interviews):**
- 1 million requests/day = ~12 req/sec
- p99 latency: 99% of requests are faster than this number
- Disk write = 0.1ms, SSD read = 0.1ms, RAM access = 100ns, Network round trip = 1–100ms

---

*Built with ❤️ for Faizal's Zero-to-Hero System Design sprint. Go build something great on paperdraw.dev.*
