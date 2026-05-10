# DevOps Practices

## Deployment Strategies

How you ship a new version determines how much risk and downtime users see. Common strategies, in roughly increasing safety / increasing complexity:

### Recreate

Stop all old instances → start new ones. Simple but causes **downtime**. Acceptable only for non-production / single-instance dev systems.

### Rolling Update

Replace instances **gradually**, one batch at a time. New instances come up before old ones go down → zero downtime. Default in Kubernetes Deployments.

- Pros: simple, no extra infra, gradual exposure.
- Cons: rollback is also gradual (slow when something is wrong); old + new versions run simultaneously, so they must be backwards-compatible.

### Blue-Green

Two identical production environments: **blue** (live) and **green** (staging). Deploy new version to green, smoke-test it, then flip traffic from blue to green at the load balancer. Old environment stays around for instant rollback.

- Pros: instant cutover, instant rollback, full pre-release validation.
- Cons: 2× infra cost during the switch; database / shared-state changes still need expand-contract migrations.

### Canary

Route a small slice of traffic (1–5%) to the new version. Watch metrics (errors, latency). If healthy, increase the slice (10% → 25% → 50% → 100%). If not, roll back having hurt only a small % of users.

- Pros: real production validation with bounded blast radius.
- Cons: need traffic-splitting infra (service mesh, ingress weights); needs solid monitoring to detect regressions early.

### Shadow

New version runs alongside the old, receives a **mirror** of real traffic, but its responses are **discarded**. Users only see the old version. Lets you test under real load + real data without user impact.

- Pros: zero user risk; great for performance / load comparison.
- Cons: side-effects (DB writes, downstream calls) need careful handling — shadow workloads must not double-charge customers, send duplicate emails, etc.

### Feature Flags

Deploy code dark; toggle features on for specific users / cohorts via a runtime flag. Decouples **deploy** from **release**.

- Combine with canary: deploy to all servers, enable feature for 1% of users, ramp up.
- Pros: instant kill switch, A/B testing, per-user rollout, gradual feature adoption.
- Cons: feature-flag debt — flags must be cleaned up; tons of flags = combinatorial test surface.

### Quick comparison

| Strategy | Downtime | Rollback speed | Infra cost | Complexity |
| --- | --- | --- | --- | --- |
| Recreate | Yes | Slow | 1× | Low |
| Rolling | No | Slow | 1× | Low |
| Blue-Green | No | Instant | 2× during switch | Medium |
| Canary | No | Fast | ~1× + routing | Medium |
| Shadow | No | N/A (validation only) | 2× during test | High |
| Feature Flags | No | Instant (toggle) | 1× | Medium (flag system) |

In practice teams combine these — e.g. rolling update at the K8s level + feature flags at the application level + canary routing for a subset of users.

**Articles:**

- [Intro to deployment strategies: blue-green, canary, and more — dev.to](https://dev.to/mostlyjason/intro-to-deployment-strategies-blue-green-canary-and-more-3a3)
- [Deployment Strategies (Rolling, Blue-Green, Canary) — dev.to](https://dev.to/godofgeeks/deployment-strategies-rolling-blue-green-canary-4ob0)
- [The Ultimate Guide to Deployment Strategies — Medium](https://medium.com/@nikoo.asadnejad.work/the-ultimate-guide-to-deployment-strategies-blue-green-canary-rolling-and-beyond-a5b916aed693)

---

## Distributed Tracing

In a microservices system, one user request fans out into dozens of internal calls. **Distributed tracing** stitches them back into a single end-to-end picture.

Core concepts:

- **Trace** — the full journey of one request across all services. Identified by a **trace ID**.
- **Span** — one unit of work within a trace (an HTTP handler, a DB query, a cache lookup). A span has a start time, end time, attributes, status, and a parent span.
- **Parent–child relationship** — when service A calls service B, B's spans are children of A's. The graph forms a tree.
- **Context propagation** — the trace ID + parent span ID must travel across the network (HTTP headers like `traceparent`, message metadata) so distant spans can be linked back.

```text
trace_id: abc123
└─ Span: GET /checkout                (frontend)
   ├─ Span: POST /orders               (order service)
   │  ├─ Span: SELECT … FROM users     (db)
   │  └─ Span: POST /payments          (payment service)
   │     └─ Span: stripe.charge         (external)
   └─ Span: render template             (frontend)
```

### OpenTelemetry (OTel)

The de facto open standard. Vendor-neutral SDKs (Python, Go, Node, Java, …) for instrumenting code; the **OTel Collector** receives and exports to any backend (Jaeger, Tempo, Honeycomb, Datadog).

Two ways to instrument:

- **Auto-instrumentation** — drop-in libraries instrument popular frameworks (FastAPI, Django, requests, SQLAlchemy) without code changes.
- **Manual** — `tracer.start_as_current_span("name")` for custom business operations.

### Sampling

Tracing every request is expensive. Common strategies:

- **Head-based** — decide at the start (e.g. 1% of requests). Cheap, but you may miss the rare slow/error trace.
- **Tail-based** — buffer all spans of a trace, decide after the request finishes. Keep all errors / slow traces, sample only fast ones. More expensive but more useful.

### Why it matters

- Find the slow service among 30 hops.
- Pinpoint root cause of an error across services.
- Quantify the cost of each step (DB vs network vs CPU).
- Correlate logs, metrics, and traces by `trace_id`.

**Articles:**

- [What is Distributed Tracing? How It Works with OpenTelemetry — Uptrace](https://uptrace.dev/opentelemetry/distributed-tracing)
- [OpenTelemetry Spans Explained: Deconstructing Distributed Tracing — Last9](https://last9.io/blog/opentelemetry-spans-events/)
- [Understanding OpenTelemetry Spans in Detail — SigNoz](https://signoz.io/blog/opentelemetry-spans/)

---

## Observability & Monitoring

**Monitoring** = "is the system meeting known thresholds?" — predefined dashboards and alerts for known failure modes.
**Observability** = "can I ask new questions of the system without redeploying?" — rich enough telemetry that you can investigate failures you didn't anticipate.

You need both. Monitoring is for known unknowns; observability is for unknown unknowns.

### Three pillars (MELT)

- **Metrics** — aggregated numerical time series (CPU, RPS, p99 latency). Cheap, great for alerts and dashboards. Bad for high-cardinality questions.
- **Logs** — structured event records. High-cardinality and rich, but expensive to store and search at scale.
- **Traces** — request flow across services (see above). Best for understanding latency and errors across boundaries.
- (**E**vents — sometimes added: deploys, config changes, incidents — annotate the timeline.)

Modern best practice: **structured logs** (JSON) + **trace IDs** in every log line + metric labels for high-level dimensions only.

### The Four Golden Signals (Google SRE)

For every user-facing service, monitor:

1. **Latency** — how long requests take. Track percentiles (p50/p95/p99), not averages. Distinguish successful and failed latency.
2. **Traffic** — how many requests (RPS / QPS).
3. **Errors** — rate of failed requests. Both `5xx` and silent failures (wrong response).
4. **Saturation** — how "full" the service is (CPU, memory, queue depth, connection pool usage). Saturation predicts future latency/errors.

If you only have 4 alerts, make them these 4.

### USE method (for resources)

For every resource (CPU, disk, network): **U**tilization, **S**aturation, **E**rrors. Complement to golden signals — golden signals are user-facing, USE is resource-facing.

### SLI / SLO / SLA

- **SLI** (Service Level **Indicator**) — the actual metric you measure. e.g. "fraction of requests served in <200ms".
- **SLO** (Service Level **Objective**) — internal target for an SLI. e.g. "99.9% of requests in <200ms over a 30-day window".
- **SLA** (Service Level **Agreement**) — contractual promise to a customer, with penalties. SLA targets are *looser* than SLOs (you want internal headroom).

**Error budget** = `100% - SLO`. If your SLO is 99.9%, you have 0.1% of "allowed" failures per window. As long as you're under budget, ship features fast. When you burn the budget, freeze risky changes and invest in reliability.

### Tools landscape

- **Metrics**: Prometheus + Grafana, Datadog, Cloudwatch.
- **Logs**: Loki, Elasticsearch/OpenSearch, Splunk.
- **Traces**: Jaeger, Tempo, Honeycomb.
- **All-in-one**: Datadog, New Relic, Honeycomb, Grafana Stack.
- **Standard**: OpenTelemetry to instrument once, export anywhere.

**Articles:**

- [Distributed Monitoring 101: the "Four Golden Signals" — Medium](https://medium.com/forepaas/distributed-monitoring-101-the-four-golden-signals-305bbbc33d35)
- [What are the 'Golden Signals' that SRE teams use to detect issues? — Cisco DevNet](https://developer.cisco.com/articles/what-are-the-golden-signals/what-are-the-golden-signals-that-sre-teams-use-to-detect-issues/)
- [Setting better SLOs using Google's Golden Signals — Gremlin](https://www.gremlin.com/blog/setting-better-slos-using-googles-golden-signals)
- [Observability vs Monitoring — Datadog Knowledge Center](https://www.datadoghq.com/knowledge-center/distributed-tracing/)

---

## CI/CD Pipelines

**CI** (Continuous Integration) — every commit is automatically built and tested against the main branch. Catches integration issues early.
**CD** — overloaded acronym:

- **Continuous Delivery** — every passing build is *ready* to ship; deploy is one button-press.
- **Continuous Deployment** — every passing build *automatically* ships to production.

Senior-level expectation: you can describe a real pipeline end-to-end and reason about its trade-offs.

### Typical pipeline stages

```text
Commit → Lint/Format → Unit tests → Build artifact → Security scan
       → Integration tests → Deploy to staging → E2E tests
       → Deploy to production (canary → full)
```

Each stage is a quality gate. Failing a gate stops the pipeline.

### Branching strategy

- **Trunk-Based Development** — short-lived branches (hours, not days), all merging into `main`. Pairs with feature flags. Default for high-velocity teams.
- **GitFlow** — long-lived `develop` / `release` / `feature` branches. Heavyweight; suited to versioned releases (e.g. on-prem software).
- **GitHub Flow** — `main` + feature branches + PRs + deploy on merge. Pragmatic middle ground.

### Pipeline-as-Code

Pipeline definition lives in the repo (`.github/workflows/*.yml`, `.gitlab-ci.yml`, `Jenkinsfile`). Versioned alongside code, reviewed via PR, reproducible.

### Build once, deploy many

Build the artifact (Docker image, jar, binary) **once**, then promote that exact same artifact through staging → production. The artifact is immutable; only configuration differs per environment. Avoids "works in staging but not in prod" caused by re-builds.

### Other senior topics

- **Infrastructure as Code (IaC)** — Terraform, Pulumi, CloudFormation. Infra changes go through PR review like code.
- **Caching** — cache dependencies (`pip`, `npm`, Docker layers) to keep build times bounded.
- **Parallelism** — split test suites by file/marker; run in matrix jobs.
- **Security gates in pipeline** — SAST (e.g. Semgrep), DAST, dependency scan (Dependabot, Snyk), container scan (Trivy), secret scan (GitLeaks).
- **Artifact registry** — versioned, immutable storage for built images (ECR, GHCR, Artifactory). Deploy by digest, not by `:latest` tag.

### Common interview gotchas

- "How do you roll back a deploy?" → redeploy the previous artifact by digest. Don't rebuild from a previous commit.
- "How do you handle DB schema changes in CI/CD?" → expand-contract migrations, deployed in two PRs (see databases.md).
- "What if tests are flaky?" → quarantine them, fix the root cause; never auto-retry green out of red.

**Articles:**

- [Most Asked CI/CD Interview Questions for DevOps and Developers — Medium](https://medium.com/@learncloud/most-asked-ci-cd-interview-questions-for-devops-and-developers-629310f1713e)
- [80 CI/CD Interview Questions — MentorCruise](https://mentorcruise.com/questions/cicd/)
- [25 Essential CI/CD Interview Questions and Best Practices — FinalRoundAI](https://www.finalroundai.com/blog/ci-cd-interview-questions)

---

## Secret Management

**Secrets** = credentials that grant access (DB passwords, API tokens, signing keys, TLS private keys, service-account credentials). The biggest senior-level mistake is treating them as configuration.

### Three rules

1. **Never in code or Git.** Even private repos leak. Use a secret scanner (GitLeaks, TruffleHog) in CI.
2. **Never in plain env files committed anywhere.** `.env` is for local dev only — and must be in `.gitignore`.
3. **Never the same secret in dev / staging / prod.** Compromise of one mustn't compromise all.

### Storage options (least → most secure)

| Option | Encryption | Rotation | Audit | When to use |
| --- | --- | --- | --- | --- |
| Plain env vars | No | Manual | None | Local dev only |
| Kubernetes Secrets | etcd encryption (opt-in) | Manual | Limited | OK for low-stakes, prefer with KMS-encrypted etcd |
| Cloud KMS-backed (AWS Secrets Manager, GCP Secret Manager, Azure Key Vault) | Yes | Built-in | Yes | Default for cloud-native; tied to one cloud |
| HashiCorp Vault | Yes | Yes (incl. dynamic) | Yes | Multi-cloud, advanced needs (dynamic secrets, PKI) |

### Static vs Dynamic secrets

- **Static** — value lives in the vault, app reads it. Risk: if leaked, valid until rotated.
- **Dynamic** — vault generates on demand, scoped to the requester, with short TTL (e.g. 60 min). Vault itself revokes when the lease expires. Massively limits blast radius.

Vault dynamic secrets shine for: per-pod DB users, short-lived AWS IAM credentials, on-demand SSH/PKI certificates. The credential leaked from logs at 3am is dead by 4am.

### Secret rotation

- Vault auto-rotates static secrets on a schedule.
- AWS Secrets Manager calls a Lambda to perform the actual rotation.
- Apps must re-read the secret periodically (or get notified) — caching for hours after start defeats rotation.

### How apps get the secret

- **Sidecar injection** (Vault Agent, Vault Secrets Operator) — secret is mounted into the pod as a file or env var; app sees only that.
- **CSI Secret Store driver** — secrets mount as files via a CSI volume.
- **SDK call** — app authenticates with the vault and fetches secrets directly. Most flexible but most code.
- **K8s Secrets synced from a vault** — the vault is the source of truth; Kubernetes Secret is a derived view.

### Authentication into the vault

Apps must authenticate before reading. Common patterns:

- **Kubernetes auth** — pod's service-account token is exchanged for a Vault token. No long-lived credential ever sits in the pod.
- **AWS IAM auth** — instance / IRSA role is presented to the vault.
- **JWT/OIDC** — for CI/CD systems and external workloads.

### Principles of least privilege

- One role per service, scoped to the secrets *it* needs.
- Audit logging on every access; alert on anomalies.
- Break-glass procedures for emergency access — logged, reviewed, time-bounded.

### Common interview red flags to know

- **Hardcoded keys in source** — instant fail.
- **K8s Secrets without etcd encryption-at-rest** — they're base64, not encrypted.
- **Secret in CI logs** — masking must be configured per variable; logs must be redacted.
- **Long-lived AWS access keys** — prefer IAM roles + STS, or Vault dynamic AWS creds.

**Articles:**

- [Secrets Management in Kubernetes: Best Practices for Security — dev.to](https://dev.to/rubixkube/secrets-management-in-kubernetes-best-practices-for-security-1df0)
- [Integrating HashiCorp Vault with Kubernetes for Secure Secrets Management — dev.to](https://dev.to/mark_mwendia_0298dd9c0aad/integrating-hashicorp-vault-with-kubernetes-for-secure-secrets-management-1gn9)
- [A Hands-On Guide to Vault in Kubernetes — Medium](https://medium.com/@muppedaanvesh/a-hand-on-guide-to-vault-in-kubernetes-%EF%B8%8F-1daf73f331bd)
- [AWS KMS + Secrets Manager VS HashiCorp Vault — Medium](https://medium.com/@shukhrat.ismailov05/aws-key-management-service-kms-aws-secrets-manager-vs-hashicorp-vault-312d73b8da9c)

---

## 12-Factor App

A methodology (Heroku, 2011) for building cloud-native services. Twelve principles; the deploy-relevant ones come up constantly in senior interviews.

| # | Factor | TL;DR |
| --- | --- | --- |
| 1 | Codebase | One codebase, many deploys (one repo per service). |
| 2 | Dependencies | Declare and isolate (`requirements.txt`, `pyproject.toml`). No "system" deps. |
| 3 | **Config** | **Store config (incl. secrets) in the environment**, never in code. |
| 4 | Backing services | Treat DBs, queues, caches as attached resources accessed via URL. |
| 5 | **Build, release, run** | **Strict separation**: build artifact → combine with config (release) → run. |
| 6 | Processes | App is one or more **stateless** processes. State goes to a backing service. |
| 7 | Port binding | Service exposes itself via a port; no app-server-of-the-OS dependency. |
| 8 | Concurrency | Scale out via the **process model** (more workers, not bigger ones). |
| 9 | Disposability | Fast startup, graceful shutdown on `SIGTERM`. Crash-only design. |
| 10 | Dev/prod parity | Keep dev, staging, prod as similar as possible. |
| 11 | Logs | Treat logs as event streams written to stdout. The platform aggregates. |
| 12 | Admin processes | One-off tasks (migrations, REPL) run in identical environment to app processes. |

### Most-asked-about factors at senior interviews

- **Config (#3)** — Why env vars and not config files? Because env vars are language-agnostic, easy to set per-environment, and don't get accidentally committed. Secrets are config, but need a vault on top.
- **Build/Release/Run (#5)** — Build is deterministic and immutable; release combines build + env config and gets a unique ID; run is starting that release. Rolling back = running an older release. No mutating SSH-into-prod fixes.
- **Processes (#6)** — A stateless app can be killed and replaced freely → makes deploys, scaling, and self-healing trivial. State means a DB / cache, not in-process.
- **Disposability (#9)** — Containers come and go; your app must accept that. `SIGTERM` → finish in-flight requests → close connections → exit. Critical for rolling updates.

The 12-factor model is the foundation that makes Kubernetes, Heroku, ECS, etc. work. If you violate it (state in process memory, mutating prod, config in code), the platform's deploy/scale features break.

**Articles:**

- [12 Factor App Methodology — dev.to](https://dev.to/adeleke123/12-factor-app-methodology-1e57)
- [The Twelve-Factor App: Modern Principles for Cloud-native Development — Medium](https://medium.com/vertisystem-platform-services/the-twelve-factor-app-modern-principles-for-cloud-native-development-298faae28775)
- [The Twelve-Factor App — official site](https://12factor.net/)
