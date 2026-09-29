# OJP on AWS: Surviving the Connection Storm

> **RFC / PoC proposal for the Open J Proxy community**
> **Status:** Draft, open for review. Nothing below has been executed yet.
> **Author:** Roberto
> **Target:** OJP `1.0.0` (GA, 2026‑09‑20) · Amazon Aurora PostgreSQL · 3 AZs
> **Feedback wanted:** see [§10 Open questions for maintainers](#10-open-questions-for-ojp-maintainers)

---

## TL;DR

We propose a reproducible proof of concept that runs **three OJP nodes, one per Availability Zone, in front of Aurora PostgreSQL**. The same settlement workload is run under controlled failures and compared against two alternatives: the application connecting **directly** to Aurora, and (optionally) **Amazon RDS Proxy**.

The framing is **OJP as a database control plane**. It separates how far the application can scale from how many connections the database can physically handle.

This PoC does **not** claim that OJP makes the database faster or creates capacity. The claim we want to test is narrower and more useful:

> **When load or failures exceed what the database can absorb, OJP turns a database collapse into bounded, observable load shedding, and recovers without a reconnection storm.**

Every claim in this document carries a tag, so readers can tell what is known and what is to be measured:

| Tag | Meaning |
|---|---|
| **`[DOC]`** | Stated in the official OJP documentation |
| **`[SRC]`** | Verified by reading the OJP source code |
| **`[HYP]`** | Hypothesis. This PoC will measure it. |
| **`[ASK]`** | Needs confirmation from OJP maintainers before we build |

We will publish all results, including the ones that do not favour OJP.

---

## Contents

1. [Why this scenario](#1-why-this-scenario)
2. [Hypotheses](#2-hypotheses)
3. [Architecture](#3-architecture)
4. [Design decisions](#4-design-decisions)
5. [Comparison arms and sizing](#5-comparison-arms-and-sizing)
6. [Experiments](#6-experiments)
7. [Observability](#7-observability)
8. [Deliverables](#8-deliverables)
9. [Plan, cost and live demo format](#9-plan-cost-and-live-demo-format)
10. [Open questions for OJP maintainers](#10-open-questions-for-ojp-maintainers)
11. [Risks and mitigations](#11-risks-and-mitigations)
12. [Out of scope](#12-out-of-scope)
13. [References](#13-references)

---

## 1. Why this scenario

Settlement and ledger systems have a characteristic failure pattern. The database is sized for steady state, and the incidents come from **synchronized behaviour of many clients**:

- **Traffic spikes.** Batch cut-offs, market open, payroll day.
- **Restart storms.** A deployment or an autoscaling event starts dozens of JVMs at once, and every connection pool opens `minimumIdle` connections at the same moment.
- **Retry storms.** A short database hiccup makes every client retry at once, which multiplies the load exactly when capacity is lowest.

With a pool inside every application instance, the number of database connections grows with the number of instances, not with what the database can serve. OJP moves the pool out of the application and into a shared proxy tier, where it can be capped globally. That is the property this PoC puts under stress.

The workload is a **double-entry transfer** (debit account A, credit account B, append a ledger entry, all in one transaction, with an idempotency key). It is small enough to reason about and realistic enough for the finance and fintech audience.

---

## 2. Hypotheses

Each hypothesis has a pass/fail criterion. A failed hypothesis is still a publishable result.

| ID | Hypothesis | Pass criterion |
|---|---|---|
| **H1** | Under steady load, the extra hop through OJP adds low, stable latency. | We report the p50/p99 delta between Direct and OJP at 150 tx/s. Expected: single-digit milliseconds at p99. `[HYP]` |
| **H2** | During a traffic spike and a restart storm, OJP keeps Aurora connections at or below the configured global pool, while the Direct arm exceeds `max_connections`. | Aurora `DatabaseConnections` ≤ 60 (+ admin) for OJP. Zero `too many connections` errors. `[HYP]` |
| **H3** | Under overload, OJP rejects excess work quickly and in bounded time instead of letting requests queue without limit. | Rejected requests fail within the admission budget (2 s). Steady-state p99 returns within 30 s after the spike ends. `[HYP]` |
| **H4** | Losing one OJP node only affects the sessions bound to that node. The remaining nodes absorb the pool share, and the cluster rebalances on recovery. | Errors limited to in-flight sessions on the lost node. Total DB connections stay ≤ 60. Rebalanced after the node returns. `[DOC]` behaviour, `[HYP]` timings |
| **H5** | During an Aurora writer failover, OJP prevents a reconnection storm against the new writer. | Connection count on the new writer stays ≤ 60 during recovery. We report time-to-recovery and error count. `[HYP]` |
| **H6** | When database latency rises, OJP's admission control protects the database and the application degrades predictably. | No DB overload. Load shedding matches the Little's law ceiling (see E4). `[HYP]` |

### The four pillars

For the article and the live session, the hypotheses and experiments are grouped into four pillars:

| Pillar | Question it answers | Experiments |
|---|---|---|
| **1 · A/B baseline** | What does OJP change compared with the usual setup, under identical load? | E0, E1 on Arms A, B, C |
| **2 · Overload protection** | Does admission control (and, separately, Slow Query Segregation) keep the database healthy when demand exceeds capacity? | E1, E4 |
| **3 · Chaos engineering** | What happens when an OJP node, an AZ or the Aurora writer fails under load? | E2, E3, E5 |
| **4 · End-to-end observability** | Can we see every effect above from client to database on one screen? | §7, all experiments |

Pillar 2 deliberately tests the thundering herd and Slow Query Segregation **in separate runs**. If both change in one run, we cannot tell which one caused the result.

---

## 3. Architecture

```mermaid
flowchart TB
    subgraph REGION["AWS Region · single VPC · private subnets only (no NAT, no public IPs)"]
        FIS["AWS FIS<br/>experiments + CloudWatch-alarm stop conditions"]
        EP["VPC endpoints<br/>SSM · Secrets Manager · CloudWatch · S3 (gateway)"]
        ALB["Internal ALB (application tier only)"]

        subgraph AZA["AZ-a (experiment control)"]
            K6["k6 load generator"]
            OBS["Prometheus · Grafana · Jaeger"]
            APPA["settlement-service<br/>4 JVMs · OJP JDBC driver"]
            OJPA["ojp-a.poc.internal:1059<br/>ASG size 1 · Graviton"]
        end

        subgraph AZB["AZ-b"]
            APPB["settlement-service<br/>4 JVMs · OJP JDBC driver"]
            OJPB["ojp-b.poc.internal:1059<br/>ASG size 1 · Graviton"]
            W[("Aurora writer")]
        end

        subgraph AZC["AZ-c"]
            APPC["settlement-service<br/>4 JVMs · OJP JDBC driver"]
            OJPC["ojp-c.poc.internal:1059<br/>ASG size 1 · Graviton"]
            R[("Aurora reader<br/>failover target")]
        end
    end

    K6 --> ALB
    ALB --> APPA & APPB & APPC
    APPA & APPB & APPC -.->|"jdbc:ojp[ojp-a,ojp-b,ojp-c]_postgresql://… (client-side, load-aware)"| OJPA & OJPB & OJPC
    OJPA & OJPB & OJPC --> W
    W -.->|replication| R
    OBS -.->|"scrape :9159 / :9404"| OJPA & OJPB & OJPC
```

**How a request flows.** k6 → internal ALB → a `settlement-service` JVM. The JVM uses the OJP JDBC driver with a **multinode URL** listing all three OJP nodes. The driver picks the least-loaded node for new connections and keeps a session on the same node for its whole lifetime `[DOC]`. The OJP node runs the real PostgreSQL JDBC driver and a server-side pool against the Aurora **cluster (writer) endpoint**.

**Placement is deliberate.**

- **AZ-a** hosts the load generator and the observability stack. The observer must survive the experiment, so AZ-a is never the AZ we isolate.
- The **Aurora writer** runs in AZ-b and the **reader** (failover target) in AZ-c. Experiment E3b moves the writer into the isolated AZ on purpose.

---

## 4. Design decisions

Each decision is driven by how OJP actually works. The tag says where that knowledge comes from.

### 4.1 No load balancer in front of OJP

The OJP driver does its own load balancing and failover from the multinode URL `jdbc:ojp[host1:1059,host2:1059,host3:1059]_postgresql://…` `[DOC]`. For three things to work, the driver must see each node individually:

- keeping a session on one node (session stickiness),
- sending new connections to the least-loaded node,
- dividing the global pool among healthy nodes.

An NLB would hide the nodes and break all three. The ALB in this design sits **only in front of the application tier**.

### 4.2 Fixed OJP tier: three ASGs of size 1, no autoscaling

Two documented facts rule out autoscaling the proxy tier:

- **Nodes are listed explicitly in the URL.** Automatic discovery is not supported yet `[DOC]`.
- **The global pool is divided among healthy nodes.** Adding nodes does not add database capacity. For example, `maximumPoolSize=30` with 3 nodes gives 10 per node, and 15 per node if one fails `[DOC]`.

Each OJP node is therefore an Auto Scaling Group with min = max = desired = 1, used only for self-healing. Each node gets a stable Route 53 private name (`ojp-a.poc.internal`, …), which is updated when the instance is replaced.

This also means predictive scaling is not the right lever for this problem. It additionally needs at least 24 hours of history before it can forecast.

### 4.3 Graviton through the runnable JAR

The official image `rrobetti/ojp` is published for **amd64 only**, on every tag including `1.0.0` (checked on Docker Hub). The OJP server requires **Java 25** `[DOC]`. Each node therefore runs `ojp-server-1.0.0-shaded.jar` on **Amazon Corretto 25 (arm64)**, as a systemd service.

- **Instance size:** `c7g.large` (2 vCPU, 4 GiB) is the minimum. One vCPU is too little for gRPC, the default 200 worker threads and GC together.
- **JVM flags:**
  - `-XX:+UseG1GC`, set explicitly as the official Dockerfile does.
  - `-Duser.timezone=UTC`, required by OJP `[DOC]`.
  - A short DNS cache (`networkaddress.cache.ttl=5`), so the node follows the Aurora cluster endpoint after a failover.

The cost-per-transaction comparison with x86 is reported as a **measured result** `[HYP]`, not taken from a marketing percentage.

### 4.4 The pool is declared by the client and divided by the servers

The pool configuration lives in the **application** (Spring properties / `ojp.properties` / `OJP_CONNECTION_POOL_*` environment variables) and is forwarded to the servers `[DOC]`.

- `maximumPoolSize=60` means **60 connections globally**: 20 per node with 3 healthy nodes, 30 per node with 2.
- The application uses **no local pool**. The OJP Spring Boot starter configures `SimpleDriverDataSource` for this reason `[DOC]`.

All application instances send the same URL, user and password. The server keys its pool on a hash of **URL + user + password + datasource name** `[SRC: ConnectionHashGenerator]`, so all 12 JVMs share **one pool per OJP node**. That sharing is the connection multiplexing this PoC measures.

### 4.5 Admission control is the protection mechanism

Before a request can borrow a database connection, it must pass an **admission gate** `[DOC]`:

- The gate has one semaphore per datasource, sized to the pool.
- Waiters are capped by `ojp.server.admissionControl.maxQueueDepth`. The default `0` means auto: 2 × the pool slots.
- Beyond that cap, requests are rejected immediately with `RESOURCE_EXHAUSTED`.
- The pool borrow itself is fail-fast.

The PoC sets `ojp.connection.pool.connectionTimeout=2000` so the admission wait is visible and bounded. The application maps `RESOURCE_EXHAUSTED` to **HTTP 503 with `Retry-After`**.

**Slow Query Segregation** is off by default. We enable it explicitly (`ojp.server.slowQuerySegregation.enabled=true`) for the mixed-load variant in E4. We always write the name out in full: "SQS" would be read as Amazon SQS by an AWS audience.

### 4.6 Secure by default

**Network**
- No NAT gateway and no public IPs.
- Security groups allow only the paths that are needed:
  - ALB → app on 8080
  - app → OJP on 1059
  - OJP → Aurora on 5432
  - observability → metrics ports
- On top of the security groups, OJP's own IP allow lists are restricted. `ojp.server.allowedIps` is limited to the app subnets, and `ojp.prometheus.allowedIps` to the observability subnet. Both default to `0.0.0.0/0` `[DOC]`.

**Encryption in transit.** OJP's gRPC is **plaintext by default** `[DOC]`, which is not acceptable for a financial ledger.
- App → OJP uses **mTLS**, which OJP supports `[DOC]`.
- OJP → Aurora uses TLS with `sslmode=verify-full`, with the certificates configured on the OJP server `[DOC]`.
- E0 measures how much TLS costs in latency.

**Encryption at rest.** KMS on the Aurora cluster, EBS encryption by default, SSE on the artifact bucket.

**Identity and access**
- Operators use **SSM Session Manager**. There is no SSH and no bastion. Grafana is reached through SSM port forwarding.
- IMDSv2 is required on every instance.
- Each instance role gets least privilege. Only the app role can read the application secret.
- Database credentials live in **Secrets Manager** (Aurora-managed master secret plus an application user).
- CI/CD uses **GitHub Actions with OIDC**, with no long-lived AWS keys. The IAM role's trust policy is scoped to the repository and branch.

**Supply chain.** Artifacts (OJP JAR, JDBC driver, JMX exporter, k6 binary, application JAR) are staged in an S3 bucket with checksums verified and pinned versions. Instances reach it through the S3 gateway endpoint.

**Detection.** CloudTrail and VPC Flow Logs are on. GuardDuty is optional for the PoC.

### 4.7 Chaos with AWS FIS and guardrails

We use AWS FIS only; Gremlin is not needed. Every experiment template has **stop conditions bound to CloudWatch alarms**, such as an ALB 5xx ratio or Aurora CPU. A running experiment aborts itself if the blast radius exceeds what was planned. The templates are managed in Terraform (`aws_fis_experiment_template`).

### 4.8 Static stability: capacity after losing an AZ

The system must survive losing one AZ **without launching new capacity**. Replacement instances and Route 53 updates restore redundancy afterwards, but availability must not depend on them.

- **Requirement:** 2 OJP nodes and 8 JVMs carry **100% of the steady load (150 tx/s)** with p99 inside the SLO (§7).
- **Per OJP node:** each surviving node goes from 20 to 30 connections and must have CPU and heap headroom for that.
- **Checked twice:**
  - in E0, by running `steady` with one OJP node and one app host removed,
  - in E2 and E3, under live failure.

### 4.9 Health checks that do not cascade

- The ALB uses a **shallow** health check (`/health/live`) that does not touch the database.
  - If it went through OJP to Aurora, one database problem would mark **every** target unhealthy at once.
- A **deep** check (`/health/ready`, app → OJP → `SELECT 1`) feeds alarms and dashboards only.
- The OJP nodes' ASG health checks use EC2 status plus a local check on the gRPC port.

### 4.10 Operations

- **OJP upgrades** are rolling, one node at a time. Each step is the same event as E2, so E2 also validates the upgrade procedure.
- **Logs:** app and OJP logs go to CloudWatch Logs with a short retention.
- **Metric cardinality:** OJP's SQL metrics carry the SQL text as a label `[DOC]`. The workload has fewer than 10 distinct statements, so this is safe here, but we note it as a production concern.

### 4.11 Well-Architected alignment and declared trade-offs

| Pillar | How the design addresses it |
|---|---|
| **Reliability** | Multi-AZ; fail fast with bounded queues (admission control); throttling; retries with backoff and jitter; static stability (§4.8); shallow health checks (§4.9); failure testing with FIS |
| **Security** | Private network; mTLS and TLS in transit; KMS at rest; least-privilege IAM; OIDC; Secrets Manager; IMDSv2; CloudTrail and Flow Logs (§4.6) |
| **Operational excellence** | Everything as code; runbooks per experiment; game days; SLOs; one dashboard |
| **Performance efficiency** | Instance choices measured, not assumed; Graviton; open-model load tests |
| **Cost optimization** | Estimate, budget alarm, `make down`; Graviton; cross-AZ transfer measured |
| **Sustainability** | Graviton; environment exists only while experiments run |

**Trade-offs we accept on purpose**

- **Cross-AZ routing vs AZ independence.** AWS's Multi-AZ guidance favours keeping traffic within one AZ, so a partial failure there stays contained. OJP's load-aware routing does not consider the AZ, so about 2/3 of calls cross AZs. We accept this, measure it (E0), and ask maintainers about AZ-aware routing (Q5).
- **Single-AZ test harness.** The load generator and observability run only in AZ-a. They are not part of the system under test, and keeping them out of the failures is the point.
- **No disaster recovery.**
  - Aurora automated backups are on, with 7-day retention. That gives point-in-time recovery.
  - Deletion protection is off so `make down` works.
  - Multi-region is out of scope, and so are an RTO and RPO for a regional failure.
- **Aurora Standard storage** rather than I/O-Optimized. Short runs make Standard cheaper. We will report the I/O cost the storms generate.

---

## 5. Comparison arms and sizing

The same application binary runs in every arm. Only the datasource configuration changes.

| | **Arm A: Direct** | **Arm B: OJP** | **Arm C: RDS Proxy** *(optional, see Q7)* |
|---|---|---|---|
| Application fleet | 3 EC2 × 4 JVMs = **12 JVMs** | same | same |
| Pool location | HikariCP in each JVM | OJP servers (no local pool) | HikariCP in each JVM, connecting to the proxy |
| Pool config | `max=20`, `minIdle=10` per JVM | `maximumPoolSize=60` global (20/node) | `max=20` per JVM, `MaxConnectionsPercent=30` |
| **Worst-case DB connections** | **240** (restart storm: **120** at once) | **60** | **~60** |
| Overload behaviour | DB saturation / connection errors `[HYP]` | Admission queue, then `RESOURCE_EXHAUSTED` `[DOC]` | Proxy queueing / borrow timeout `[HYP]` |

**Aurora configuration**

- `db.r7g.large` writer plus reader, PostgreSQL 16.
- `max_connections=200`, set explicitly in the DB parameter group.
- This deliberately represents a database **sized for steady state**, which is exactly the situation where storms hurt. The value is documented, not hidden.

**Load profiles (k6)**

| Profile | Shape |
|---|---|
| `steady` | Constant arrival rate, 150 tx/s |
| `spike` | 150 → 1,500 tx/s over 10 s, held for 2 min, back to 150 |
| `restart-storm` | Steady load while all 12 JVMs restart within 5 s |
| `retry-storm` *(variant)* | `spike` with naive client retries versus exponential backoff with jitter |

All profiles use k6's **open model** (`constant-arrival-rate` / `ramping-arrival-rate`), not a fixed number of virtual users.

- With fixed VUs, each virtual user waits for a response before sending the next request.
- So when latency rises, the offered load drops by itself, and the storm hides exactly when we want to see it (coordinated omission).
- In the open model, requests keep arriving at the planned rate no matter how slow the system gets, which is how real traffic behaves.

Each run lasts 10 minutes after a 5-minute warm-up and is repeated **3 times**. We report the median run and the spread.

---

## 6. Experiments

The execution order is fixed. Each experiment runs on Arm B, and E0 to E2 also run on Arms A and C.

### E0 · Baseline and proxy overhead (H1)

- **Load:** `steady`, 150 tx/s, no faults.
- **Measure:** p50/p95/p99 latency (client and server side), DB connections, Aurora CPU, OJP connection-acquisition time.
- **Extra finding to report: cross-AZ traffic.** Load-aware selection does not consider the AZ `[DOC]`, so about 2/3 of JDBC calls should cross AZ boundaries `[HYP]`. We will measure the latency and data-transfer cost this adds.
- **Variant E0-TLS:** the same run with plaintext and with mTLS + TLS, to report what encryption costs in latency.
- **Variant E0-N-1:** the same run with one OJP node and one app host removed, to verify the capacity requirement in §4.8.
- **Output:** the SLO targets (§7) are fixed from this run, before any storm or chaos experiment.

### E1 · Thundering herd (H2, H3)

- **Load:** `spike`, then `restart-storm`, then the `retry-storm` variant.
- **Measure:**
  - peak Aurora `DatabaseConnections`
  - database errors
  - Aurora CPU
  - rejection latency (time to 503)
  - time to return to steady-state p99
- **Expected, Arm A:** connections climb toward 240 and hit `max_connections`. Errors and latency surge `[HYP]`.
- **Expected, Arm B:** DB connections stay ≤ 60. Excess requests are rejected within 2 s. Fast recovery `[HYP]`.
- **Stop condition:** Aurora CPU > 95% for 3 min (protects the run, not the thesis).

### E2 · Lose one OJP node (H4)

- **Injection:** FIS `aws:ec2:stop-instances` on `ojp-b`, with `startInstancesAfterDuration=PT3M`.
- **Expected `[DOC]`:**
  - Sessions bound to `ojp-b` get a `SQLException`. This is intentional, so no transaction silently moves to another node.
  - New connections go to `ojp-a` and `ojp-c`, whose pool share grows from 20 to 30.
  - When `ojp-b` returns, the health checker (default 5 s interval, 5 s threshold) marks it healthy and connections rebalance.
- **Measure:** error count and duration, total DB connections (must stay ≤ 60), time to rebalance.
- **Variant:** kill the process instead of the instance (`AWSFIS-Run-Kill-Process`) to compare fast and slow failure detection.

### E3 · Availability Zone isolation (H4, H5)

- **E3a, writer outside the AZ.** FIS `aws:network:disrupt-connectivity` (`scope=all`) on every AZ-c subnet for 5 min. This takes out `app-c`, `ojp-c` and the Aurora reader.
- **E3b, writer inside the AZ (compound failure).** Before the run, fail over so the writer sits in AZ-c. Then isolate AZ-c **and** run `aws:rds:failover-db-cluster`.
  - Isolation through network ACLs does not guarantee that Aurora fails over by itself. The explicit action makes the scenario realistic.
- **Also measure the ALB.** Its nodes in the isolated AZ keep receiving traffic until DNS and health checks drain them. We record those client timeouts separately. An optional variant uses an ARC zonal shift on the ALB.

### E4 · Latency between OJP and Aurora (H6)

- **Injection:** FIS `aws:ssm:send-command` with `AWSFIS-Run-Network-Latency-Sources` on the three OJP nodes:
  - `DelayMilliseconds=200`
  - `Sources` = the Aurora subnet CIDRs
  - `TrafficType=egress`
  - 5 min
- **Expected ceiling (Little's law):** a transfer is about 4 round trips (2 × UPDATE, INSERT, COMMIT). With 200 ms added per round trip, one transaction holds a connection for ≥ 0.8 s. With 60 connections, the ceiling is about **75 tx/s**, so roughly **half** of the 150 tx/s offered load must be shed.
- **Measure:** whether shedding matches this ceiling, OJP circuit breaker state, admission queue depth, and p99 of admitted requests.
- **Variant:** add a slow reporting query at 5 req/s, with Slow Query Segregation off and then on.

### E5 · Aurora writer failover (H5)

- **Injection:** FIS `aws:rds:failover-db-cluster` under `steady` load.
- **E5a: OJP + native pgjdbc.**
  - Expected: in-flight transactions fail. This error comes from the database connection being dropped, **not** from an OJP design decision.
  - The OJP pools then rebuild against the new writer through the cluster endpoint.
  - Measure: time-to-recovery, number of failed transactions, peak connection count on the new writer.
- **E5b: OJP + AWS Advanced JDBC Wrapper** (placed in `ojp-libs`, URL `jdbc:ojp[…]_aws-wrapper:postgresql://…`, `failover2` plugin).
  - OJP detects the database type by the `POSTGRESQL:` substring in the URL `[SRC: DatabaseUtils]`, so detection should work.
  - The full path has **not been validated** `[ASK]`.
- **E5c: Arm C (RDS Proxy)** for reference.

### E6 · Credential rotation *(exploratory)*

- **Injection:** rotate the application user's secret in Secrets Manager under `steady` load.
- **Hypothesis:** the pool hash includes the password `[SRC]`, so the new password creates a **new pool next to the old one**, and DB connections may briefly double `[HYP]`.
- **Unknown:** what happens to the old pool (drained, closed, or left until idle timeout) `[ASK]`.

---

## 7. Observability

All metrics go to one Grafana dashboard, which is exported as JSON in the repository.

| Source | What | How |
|---|---|---|
| **OJP (native)** | HikariCP active/idle/pending/max, **connection-acquisition time histogram**, SQL execution time per statement, slow executions, circuit breaker state/trips | Prometheus endpoint `:9159/metrics` `[DOC]` |
| **OJP JVM** | Heap, GC pauses, threads | Prometheus **JMX exporter** java agent on `:9404` (the native endpoint does not expose JVM metrics) |
| **settlement-service** | HTTP rate/latency/status, business outcomes (committed, rejected, failed), 503s caused by `RESOURCE_EXHAUSTED` | Micrometer / Prometheus |
| **k6** | Client-side latency and errors (the latency users actually see) | Prometheus remote write |
| **Aurora** | `DatabaseConnections`, CPU, commit latency, failover events | Grafana CloudWatch datasource |
| **Tracing** | End-to-end spans: app → OJP → SQL | OJP OTLP exporter (`ojp.tracing.enabled=true`, sample rate 0.1) → Jaeger |
| **Experiment markers** | FIS start and stop shown as Grafana annotations | EventBridge → annotation script |

Prometheus discovers OJP nodes through `ec2_sd_configs` filtered by tag, so a replaced node is picked up automatically. App and OJP logs go to CloudWatch Logs.

**SLOs (steady state).** They are fixed from E0, before any storm or chaos run, so they cannot be tuned after seeing the results:

- **Success rate:** ≥ 99.9% of transfers commit.
- **Latency:** p99 ≤ 1.5 × the E0 p99 of the Direct arm.

During experiments we report how much of the error budget each failure consumed and how long the system took to return inside the SLO.

---

## 8. Deliverables

```
ojp-aws-connection-storm/
├── PROPOSAL.md                  # this document
├── README.md                    # quick start: make artifacts → make up → make run E1 → make down
├── terraform/
│   ├── envs/poc/                # root module, variables, outputs
│   └── modules/
│       ├── network/             # VPC, 3 AZs, private subnets, VPC endpoints
│       ├── aurora/              # cluster, parameter group (max_connections), secrets
│       ├── ojp-node/            # ASG(1) + launch template + Route 53 record (instantiated ×3)
│       ├── app/                 # app ASG + internal ALB
│       ├── loadgen/             # k6 host
│       ├── observability/       # Prometheus, Grafana, Jaeger host
│       ├── chaos/               # FIS templates + CloudWatch alarms as stop conditions
│       └── github-oidc/         # OIDC provider + scoped CI role
├── app/settlement-service/      # Spring Boot + spring-boot-starter-ojp (+ direct profile for Arm A)
├── load-test/                   # k6 profiles: steady, spike, restart-storm, retry-storm
├── chaos/                       # experiment runbooks (E0–E6), one per file
├── observability/               # prometheus.yml, Grafana dashboard JSON, JMX exporter config
├── results/                     # raw CSV/JSON per run + scripts that generate every chart
└── docs/
    ├── architecture.md          # C4 context/container diagrams, packet and connection-state flow
    └── adr/                     # one ADR per decision in §4
```

**Rules for publication**

- Every chart in the write-up is generated from `results/` by a script in the repository. No hand-edited numbers.
- Each result states the OJP version, instance types, region and date.
- Failed hypotheses are published with the same prominence as passed ones.

---

## 9. Plan, cost and live demo format

### Phases

| Phase | Output | Gate |
|---|---|---|
| **P0 · Validate** | This proposal, reviewed | Maintainers answer [§10](#10-open-questions-for-ojp-maintainers) |
| **P1 · Build** | Terraform, service, dashboards; E0 on Arms A and B | E0 reproducible across 3 runs |
| **P2 · Storm** | E1 on all arms | Results reviewed with maintainers |
| **P3 · Chaos** | E2–E6 | Every experiment has a runbook and raw data |
| **P4 · Publish** | Article, repository tagged `v1.0`, live session | Community review |

### Cost (estimate, to be confirmed with the AWS Pricing Calculator)

- **Everything running:** 2 × `db.r7g.large`, 3 × `c7g.large` (OJP), 3 × `c7g.large` (app), 1 × `c7g.xlarge` (k6), 1 observability host, internal ALB, VPC interface endpoints. Expected on the order of **US$ 1.5–2 per hour** on-demand in us-east-1, plus data transfer.
- **A full experiment day** (about 6 hours) is expected to stay **below US$ 15**.
- `make down` destroys everything. A budget alarm is part of the Terraform.
- **Known cost drivers:**
  - the VPC interface endpoints (charged per AZ per hour; chosen over a NAT gateway for security),
  - cross-AZ data transfer,
  - Aurora I/O during storms (Standard storage).
- **The resources can be counted, but not priced from here.** The per-hour total must be confirmed in the [AWS Pricing Calculator](https://calculator.aws/) for the chosen region.

### Live session agenda (60 min)

| Time | Block | Content |
|---|---|---|
| 5 min | **The problem** | Connection storms in microservice fleets. Why per-instance pools do not scale with the database. |
| 10 min | **Architecture** | Three OJP nodes across AZs. Why there is no load balancer in front of OJP. How the pool is divided. Why the proxy tier does not autoscale. |
| 15 min | **A/B results** *(pre-recorded)* | Direct vs OJP vs RDS Proxy under spike and restart storm, with the data from 3 runs. Wins and losses both shown. |
| 15 min | **Live chaos** | E2 (kill an OJP node) and E5a (Aurora failover), triggered from the FIS console with the Grafana dashboard on screen. |
| 15 min | **Q&A** | Open questions, the answers from maintainers, and the repository. |

- **Why the A/B is pre-recorded:** it needs several repetitions to be credible, and one live run proves nothing.
- **Safety:** each live experiment has an abort button (stop the FIS experiment) and a fallback recording.

---

## 10. Open questions for OJP maintainers

These are the answers we need before P1. Some of them may become upstream issues or contributions.

1. **Aurora failover.** What is the expected behaviour of the server-side pool when the backend writer disappears? Are there recommended `maxLifetime`, keepalive or validation settings for Aurora?
2. **AWS Advanced JDBC Wrapper inside OJP (E5b).** Is it supported or recommended to place the wrapper in `ojp-libs` and use `jdbc:ojp[…]_aws-wrapper:postgresql://…`? Any known conflicts (driver registration, dialect detection, the wrapper's own pooling)?
3. **Credential rotation (E6).** The pool hash includes the password. After a rotation, what happens to the old pool? Is there a recommended rotation pattern?
4. **Node addressing.** Stable DNS names per node is our plan. Is DNS re-resolution on reconnect supported by the driver? Is endpoint discovery on the roadmap, which would make ASG-based scaling viable?
5. **AZ-aware routing.** Is zone-aware node preference being considered, to reduce cross-AZ latency and cost? The PoC can quantify the impact (E0).
6. **arm64 image.** Would a multi-arch (`linux/amd64,linux/arm64`) Docker build be welcome as a contribution coming out of this PoC?
7. **RDS Proxy comparison (Arm C).** Is the community comfortable with a public, side-by-side comparison? We think an AWS audience will ask "why not RDS Proxy?" first, and a fair answer with data builds credibility.
8. **Client throttling.** For storm scenarios, which `ojp.jdbc.clientThrottle.mode` should we use (`reactive`, the default, or `combined`)? Any recommended `reactiveDecreaseFactor`?
9. **Metrics.** Is there an upstream Grafana dashboard we should build on, so the PoC stays aligned with future releases?

---

## 11. Risks and mitigations

| Risk | Mitigation |
|---|---|
| Results are noisy or not reproducible | Fixed instance types (no burstable instances in the data path), warm-up, 3 repetitions, spread reported |
| The benchmark is designed to make OJP win | Same binary in all arms. `max_connections` and pool sizes published. Direct arm tuned with Hikari best practices. Maintainers and at least one outside reviewer see the results before publication |
| Aurora failover with OJP behaves worse than expected | Publish it anyway, with an upstream issue. E5b/E5c show the alternatives |
| Cost overrun | Budget alarm, `make down`, runs scheduled in blocks |
| Live demo fails | Pre-recorded fallback for every live experiment; FIS stop conditions |
| OJP version drift during the project | Pin `1.0.0` for all runs; re-run E0 before publishing if a patch release lands |
| Sensitive data in transit | mTLS app → OJP, TLS OJP → Aurora; the workload uses synthetic accounts only |
| The TLS setup behaves differently from plaintext under storms | E0-TLS establishes the baseline; E1 runs with TLS on, as it would in production |
| The test harness fails during an experiment | The harness lives in AZ-a, which is never the failed AZ; FIS stop conditions |

---

## 12. Out of scope

- **XA / distributed transactions.** A different story, possibly a follow-up PoC.
- **Read/write splitting** to the Aurora reader. OJP supports it `[DOC]`, but it adds variables that would blur this experiment.
- **Multi-region, Aurora Global Database and disaster recovery** (see the trade-offs in §4.11).
- **Kubernetes/EKS.** This PoC stays on EC2 to keep the moving parts visible.
- **Autoscaling the OJP tier.** It does not fit the current design (see §4.2).

---

## 13. References

The full list of references, grouped by component and saying what each one supports, is in **[REFERENCES.md](REFERENCES.md)**. The most important ones:

- **OJP:** [multinode guide](https://github.com/Open-J-Proxy/ojp/blob/main/documents/multinode/README.md) · [server configuration](https://github.com/Open-J-Proxy/ojp/blob/main/documents/configuration/ojp-server-configuration.md) · [client configuration](https://github.com/Open-J-Proxy/ojp/blob/main/documents/configuration/ojp-jdbc-configuration.md) · [admission control](https://github.com/Open-J-Proxy/ojp/blob/main/documents/analysis/ADMISSION_CONTROL_BACKPRESSURE_SUMMARY.md) · [telemetry](https://github.com/Open-J-Proxy/ojp/blob/main/documents/telemetry/README.md) · [mTLS](https://github.com/Open-J-Proxy/ojp/blob/main/documents/configuration/mtls-configuration-guide.md)
- **Aurora:** [fast failover with Aurora PostgreSQL](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraPostgreSQL.BestPractices.FastFailover.html) · [cluster endpoints](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.Endpoints.Cluster.html) · [RDS Proxy](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/rds-proxy.html)
- **Chaos:** [FIS actions reference](https://docs.aws.amazon.com/fis/latest/userguide/fis-actions-reference.html) · [FIS SSM documents](https://docs.aws.amazon.com/fis/latest/userguide/actions-ssm-agent.html) · [stop conditions](https://docs.aws.amazon.com/fis/latest/userguide/stop-conditions.html)
- **Architecture:** [Well-Architected pillars](https://docs.aws.amazon.com/wellarchitected/latest/framework/the-pillars-of-the-framework.html) · [Advanced Multi-AZ Resilience Patterns](https://docs.aws.amazon.com/whitepapers/latest/advanced-multi-az-resilience-patterns/advanced-multi-az-resilience-patterns.html) · [Financial Services Industry Lens](https://docs.aws.amazon.com/wellarchitected/latest/financial-services-industry-lens/financial-services-industry-lens.html)

---

*Comments, corrections and objections are welcome. Please open an issue or comment on the PR that introduces this document.*
