# OJP on AWS: Surviving the Connection Storm (summary)

> Short version of [PROPOSAL.md](PROPOSAL.md). References: [REFERENCES.md](REFERENCES.md). **Status: draft, nothing measured yet.**

## Goal

Test whether **3 OJP nodes (one per AZ)** protect **Aurora PostgreSQL** from connection storms and failures, compared with the application connecting directly (and, optionally, through RDS Proxy).

**Hypothesis:** under overload or failure, OJP turns a database collapse into bounded, observable load shedding, and recovers without a reconnection storm. OJP is framed as a database control plane that caps connections centrally; it does **not** create database capacity.

**Rule:** every result gets published, whether it favours OJP or not, and every chart is generated from the raw data in the repository.

## Architecture

- **App tier:** k6 → internal ALB → 12 JVMs (3 EC2 × 4) running the settlement service.
- **OJP tier:** the app reaches OJP through the multinode URL `jdbc:ojp[ojp-a,ojp-b,ojp-c]_postgresql://…`. There is **no load balancer in front of OJP**. The driver balances load itself, keeps each session on one node and divides the pool among healthy nodes.
- **OJP nodes:** 3 ASGs of size 1, one per AZ, with stable Route 53 names. **No autoscaling**: the nodes are listed in the URL, and adding nodes does not add database capacity.
- **Runtime:** `c7g.large` Graviton instances running the OJP JAR on Corretto 25, because the official Docker image is amd64 only. JVM flags: G1, UTC timezone, DNS TTL of 5 s.
- **Pool:** `maximumPoolSize=60` is declared by the client and divided by the servers: 20 per node with 3 nodes, 30 with 2. The app keeps no local pool.
- **Admission control:** fast rejection with `RESOURCE_EXHAUSTED`, which the app returns as HTTP 503.
- **Aurora:** `db.r7g.large` writer (AZ-b) and reader (AZ-c), with `max_connections=200` set on purpose to represent a database sized for steady state.
- **Secure by default:**
  - no NAT, SSM for access, Secrets Manager for credentials, GitHub OIDC for CI;
  - **mTLS** from app to OJP (OJP is plaintext by default), **TLS `verify-full`** from OJP to Aurora;
  - OJP `allowedIps` restricted (default is `0.0.0.0/0`);
  - KMS at rest, IMDSv2, CloudTrail and Flow Logs.
- **Static stability:** 2 OJP nodes and 8 JVMs must carry 100% of the steady load **without launching new capacity**. Checked in E0 (N-1 run), E2 and E3.
- **Health checks:** the ALB uses a shallow check with no database call, so one database problem does not mark every target unhealthy. Deep checks feed alarms only.
- **Control AZ:** AZ-a hosts the load generator and observability, and is never the AZ we fail.

**Declared trade-offs (Well-Architected)**

- About 2/3 of calls cross AZs, which goes against the AZ-independence pattern. We measure it and ask maintainers.
- The test harness runs in a single AZ.
- No disaster recovery: backups (7-day PITR) only, and deletion protection is off.

## Comparison

| | Direct | OJP | RDS Proxy (optional) |
|---|---|---|---|
| Pool | Hikari 20 per JVM | OJP, 60 globally | Hikari into the proxy, capped at 30% |
| Max DB connections | **240** (restart storm: 120 at once) | **60** | ~60 |

Load comes from k6 in open model (arrival rate, not fixed VUs):

- **steady:** 150 tx/s
- **spike:** up to 1,500 tx/s
- **restart storm**
- **retry storm**

Each run is repeated 3 times, and we report the median and the spread.

## Experiments (AWS FIS, stop conditions on CloudWatch alarms)

| ID | What | What we expect / measure |
|---|---|---|
| E0 | Baseline, no faults | OJP p99 overhead; cost of calls crossing AZs; TLS cost; N-1 capacity. **SLOs are fixed here**, before any storm |
| E1 | Spike + restart storm + retry storm | DB connections ≤ 60 with OJP vs ~240 direct. Rejections within 2 s |
| E2 | Stop an OJP node | Only that node's sessions fail (by design). Pool goes 20 → 30 on the others, then rebalances |
| E3 | Isolate AZ-c (network ACLs) | a) writer outside the AZ; b) writer inside the AZ plus explicit Aurora failover |
| E4 | +200 ms between OJP and Aurora | Little's law: ceiling ~75 tx/s, so ~50% shed. Slow Query Segregation tested in a separate run |
| E5 | Aurora failover | a) native pgjdbc: in-flight transactions fail; b) AWS JDBC Wrapper inside OJP (unvalidated); c) RDS Proxy |
| E6 | Rotate the DB password | The pool hash includes the password, so a new pool opens next to the old one. What happens to the old pool is unknown |

## Observability

One Grafana dashboard with:

- **OJP `:9159`:** pool state, acquisition time, SQL timings, circuit breaker.
- **JMX exporter:** heap and GC.
- **App:** Micrometer metrics.
- **k6:** client-side latency.
- **Aurora:** CloudWatch metrics.
- **Traces:** OTLP to Jaeger.
- **FIS:** start/stop annotations on the graphs.

## Questions for OJP maintainers

1. What happens to the server-side pool during an Aurora failover? Any recommended settings?
2. Is it supported to run the AWS Advanced JDBC Wrapper inside `ojp-libs`?
3. After a password rotation, what happens to the old pool?
4. Does the driver re-resolve node DNS names? Is node discovery on the roadmap?
5. Is AZ-aware routing planned?
6. Would a multi-arch (amd64 + arm64) Docker image be welcome as a contribution?
7. Is the community comfortable with a public comparison against RDS Proxy?
8. Which client throttle mode should we use for storms?
9. Is there an upstream Grafana dashboard we should build on?

## Risks

- **Noise or a biased benchmark.** Mitigation: no burstable instances in the data path, the same binary in all arms, all configuration published, and outside review of the results.
- **Bad results for OJP.** They get published anyway, with an upstream issue.
- **Cost.** Estimated at ~US$1.5–2/h (unconfirmed), with a budget alarm and `make down`.
- **Live demo failure.** Every live experiment has a recorded fallback.

## Out of scope

XA, read/write splitting, multi-region, EKS, and autoscaling the OJP tier.

## Plan

| Phase | What |
|---|---|
| P0 | Validate this proposal with the maintainers |
| P1 | Build the infrastructure and run the baseline (E0) |
| P2 | Storm experiments (E1) |
| P3 | Chaos experiments (E2–E6) |
| P4 | Article, tagged release and live session |

**Live session (60 min):** problem 5 · architecture 10 · A/B results (pre-recorded) 15 · live chaos with E2 and E5 15 · Q&A 15.
