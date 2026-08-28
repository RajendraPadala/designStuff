# Microservices System Design — Interview Question Bank
### Spring Boot · Java · GitHub · Jenkins · Kubernetes · ELK · Gloo Mesh
### Question Bank + Two Full Situational Scenarios with Trade-off Analysis

---

## HOW TO USE THIS

**Part A** is a categorized question bank — each question has a short "what a strong answer covers" note, not a full essay, so you can self-test quickly. **Part B and C** are two complete situational scenarios, worked end-to-end with explicit trade-off reasoning — these are the ones to rehearse out loud, since "design X" and "here's an incident, walk me through it" are the two question types that actually dominate real system-design interviews.

---

# PART A — QUESTION BANK BY CATEGORY

## A1. Microservices Architecture & Design Principles

1. **How do you determine service boundaries?**
   Key points: Domain-Driven Design / bounded contexts, not technical layers; a service should own its data and change for one business reason (single responsibility at the service level); avoid boundaries that force chatty cross-service calls for a single business transaction.

2. **Database per service vs. shared database — which do you default to, and why?**
   Key points: database-per-service is the default for true service autonomy (independent schema evolution, independent scaling, no hidden coupling through shared tables) — cost is you give up cross-service joins and ACID transactions across services, forcing eventual consistency patterns (Saga, outbox).

3. **How do you handle a transaction that spans multiple services?**
   Key points: **Saga pattern** — choreography (services react to each other's published events, no central coordinator) vs. orchestration (a central saga orchestrator explicitly sequences steps and compensating actions). Choreography scales better with fewer services/simpler flows; orchestration is easier to reason about/debug as the number of steps grows, at the cost of a new coupling point (the orchestrator itself).

4. **What's the Outbox pattern and why does it matter?**
   Key points: solves the dual-write problem — writing to your own DB and publishing an event must be atomic; the outbox table is written in the *same DB transaction* as the business change, then a separate relay publishes it, avoiding "DB commit succeeded but the event publish failed" (or vice versa) inconsistency.

5. **Synchronous (REST/gRPC) vs. asynchronous (events/queue) communication — how do you decide?**
   Key points: sync when the caller genuinely needs an immediate answer to proceed; async wherever eventual consistency is acceptable, because sync call chains create cascading-failure risk (a slow downstream service directly threatens every synchronous caller up the chain).

6. **What's the difference between an API Gateway and a Service Mesh, and do you need both?**
   Key points: Gateway = north-south (external client → your system), handles auth/rate-limiting/routing for external consumers. Mesh = east-west (service-to-service inside your system), handles mTLS/retries/circuit-breaking/observability transparently via sidecars. Most real systems run both — different trust boundaries, different concerns.

7. **How do you version an API without breaking existing consumers?**
   Key points: additive-only changes within a version (new optional fields); a genuinely breaking change gets a new version with an explicit deprecation window, not a silent replace; URI versioning is simplest and most cache/tooling-friendly for most enterprise contexts.

8. **What's the Strangler Fig pattern and when would you use it?**
   Key points: incremental migration off a legacy system — new functionality built on the new platform while the legacy system keeps serving existing traffic, gradually routing more traffic to the new system until legacy can be retired. Used specifically to avoid a risky big-bang cutover.

## A2. Spring Boot / Java Implementation

9. **Walk through what happens when a Spring Boot app starts up.**
   Key points: `SpringApplication.run()` → environment prep → `ApplicationContext` creation → `refresh()` (bean factory post-processors, bean post-processor registration, singleton instantiation in dependency order) → embedded server starts only after the context is fully ready.

10. **How does Spring Boot auto-configuration decide what to configure?**
    Key points: `@ConditionalOnClass`, `@ConditionalOnMissingBean` (this is how you override any default — just declare your own bean), `@ConditionalOnProperty`; debuggable via the conditions evaluation report (`--debug`).

11. **How do you prevent the N+1 query problem in a Spring Data JPA microservice?**
    Key points: `JOIN FETCH` for managed-entity needs, `@EntityGraph` for reusable fetch plans, DTO/record projections for pure read paths — pick based on whether you need managed entities afterward.

12. **How would you implement resilience (timeouts, retries, circuit breaking) for a downstream call in Spring Boot?**
    Key points: Resilience4j (`@CircuitBreaker`, `@Retry`, `@Bulkhead`) at the application layer — and articulate that this is *complementary* to, not redundant with, mesh-level (Envoy/Gloo) resilience — defense in depth, not double coverage.

13. **How do virtual threads (Java 21+) change how you'd design a Spring Boot service's concurrency model?**
    Key points: `spring.threads.virtual.enabled=true` lets existing blocking MVC code scale I/O-bound concurrency without a WebFlux rewrite; virtual threads help I/O-bound work only, not CPU-bound; doesn't replace reactive programming for genuinely streaming use cases.

14. **How do you secure service-to-service calls in a microservices architecture?**
    Key points: two layers — external clients authenticate via JWT/OAuth2 validated at the gateway; internal service-to-service trust is typically established via mTLS at the mesh layer (mutual certificate-based identity), a different trust model for a different trust boundary.

## A3. CI/CD — GitHub & Jenkins

15. **Describe a CI/CD pipeline for a microservice from commit to production.**
    Key points: build → unit test → static analysis/quality gate (SonarQube) → security scan → containerize (Docker) → push to a registry → deploy to lower environments automatically → gated (manual approval) production deploy → post-deploy smoke test → auto-rollback on failure.

16. **How do you keep pipeline logic consistent across dozens of microservice repos?**
    Key points: Jenkins **shared libraries** (or reusable GitHub Actions workflows) — common stages defined once, each service's pipeline file stays thin and just references the shared library; a platform-wide pipeline change (new scan stage) propagates everywhere without editing every repo.

17. **What's your branching/release strategy, and why?**
    Key points: trunk-based development + feature flags for continuous integration without long-lived branches; the *same built artifact* (image, tagged by commit SHA) must be promoted through every environment unchanged — never rebuilt per environment, or you can't be certain what passed QA is what's in prod.

18. **How would you automate onboarding a brand-new microservice's repo and pipeline?**
    Key points: golden-path repo templates, Terraform-driven repo/branch-protection provisioning, webhook auto-registration to Jenkins, so a new service gets a working pipeline without a manual Jenkins-job-creation step.

## A4. Kubernetes

19. **Explain the difference between liveness and readiness probes, and a real misconfiguration risk.**
    Key points: liveness = should this pod be restarted; readiness = should traffic be routed to it right now. A liveness probe checking a downstream dependency is a classic anti-pattern — an unrelated downstream outage can trigger mass pod restarts across your whole fleet, worsening the incident.

20. **How do resource requests and limits affect scheduling and stability?**
    Key points: requests drive scheduling (which node has room); limits enforce a hard ceiling (exceeding memory limit = OOMKill). Undersized limits cause instability under real load; oversized requests waste cluster capacity.

21. **How would you autoscale a Kafka-consuming microservice based on actual backlog, not just CPU?**
    Key points: Horizontal Pod Autoscaler on a **custom metric** (consumer lag / queue depth) via a metrics adapter — CPU-based scaling often doesn't correlate well with I/O-bound consumer workload.

22. **What's a PodDisruptionBudget and why does it matter for a critical service?**
    Key points: guarantees a minimum number of pods stay available during voluntary disruptions (node drains, cluster upgrades) — without it, routine maintenance can take an entire service's replica set down simultaneously.

23. **Namespace-per-team vs. namespace-per-service-per-environment — trade-offs?**
    Key points: finer-grained namespacing gives tighter blast-radius containment (ResourceQuota, NetworkPolicy, RBAC scoped narrowly) at the cost of more objects to provision/manage — should be automated, not hand-maintained, past a small team.

## A5. Observability — ELK Stack

24. **How do you make a request traceable across many microservices?**
    Key points: a correlation/trace ID generated at the edge (or gateway), propagated through every service's logs and across async boundaries (Kafka headers) — without this, debugging a slow cross-service request means manually cross-referencing timestamps across separate log streams.

25. **What's the difference between logs, metrics, and traces, and when do you reach for each?**
    Key points: logs = discrete events, good for "what exactly happened"; metrics = aggregated numeric trends, good for "is something wrong right now" (dashboards/alerts); traces = the causal chain of a single request across services, good for "where exactly did the latency/failure happen."

26. **How would you design log retention/cost trade-offs in ELK at scale?**
    Key points: hot/warm/cold tiering in Elasticsearch, shorter retention + sampling for high-volume debug logs, longer retention for audit-relevant logs — a blanket "keep everything forever at full fidelity" policy doesn't scale cost-wise.

27. **How do you avoid alert fatigue while still catching real incidents?**
    Key points: reserve paging alerts for things that need a human right now (error rate spike, SLA breach); route lower-severity signals to a dashboard/ticket queue; tune thresholds against real historical noise, not guesses.

## A6. Service Mesh — Gloo Mesh

28. **What problem does a service mesh actually solve that a well-written HTTP client library doesn't?**
    Key points: consistency across languages/teams (mTLS, retries, tracing become infrastructure properties, not something every team re-implements correctly in every language) and centralized policy (traffic splitting, security policy) without redeploying application code.

29. **How would you implement a canary release using Gloo Mesh?**
    Key points: a `VirtualService` traffic-split rule routing a small percentage of traffic to the new version subset, monitored via mesh-level metrics, gradually increased — rollback is just a routing-weight change, not a redeploy.

30. **How does mTLS in the mesh interact with JWT-based auth at the gateway?**
    Key points: two different trust boundaries — JWT authenticates the *external caller's identity* at the edge; mTLS authenticates *service identity* for internal east-west calls. They're complementary, not redundant.

31. **What's the operational cost of adopting a service mesh, and when is it not worth it?**
    Key points: real learning curve, another control plane to operate, sidecar resource overhead per pod — worth it once you have enough services/languages that consistent cross-cutting behavior can't be achieved another way; likely overkill for a handful of services on one team.

## A7. Cross-Cutting: Security, Resilience, Data Consistency

32. **How do you prevent a slow downstream dependency from taking down your whole system?**
    Key points: timeouts everywhere (never an unbounded wait), circuit breakers to fail fast once a dependency is clearly degraded, bulkheads (separate thread/connection pools per dependency) so one dependency's exhaustion doesn't starve calls to a healthy one.

33. **How do you handle idempotency for retried requests across service boundaries?**
    Key points: idempotency keys (natural key + hash comparison) checked before applying a write — critical because most async messaging (Kafka, SQS) guarantees at-least-once delivery, so redelivery *will* happen and must be safe.

34. **What's your strategy for secrets management across services on Kubernetes?**
    Key points: never plaintext in manifests/env vars in source; Kubernetes Secrets at minimum, ideally an external secrets manager (Vault, cloud KMS) with short-lived, rotated credentials rather than long-lived static ones.

35. **How would you roll back a bad production deployment safely and fast?**
    Key points: automated post-deploy smoke tests as the trigger for auto-rollback; the "same artifact promoted through environments" discipline (A17) is what makes rollback trustworthy — you're reverting to a known-good, previously-validated artifact, not guessing.

---

# PART B — SITUATIONAL SCENARIO 1: DESIGN

## "Design a microservices-based Order Management system for an e-commerce platform. It must integrate with Payment, Inventory, and Notification services, run on Kubernetes with Gloo Mesh, be built in Spring Boot, deployed via GitHub + Jenkins, and observable via ELK."

### Requirements to state out loud first
- **Functional**: place an order → reserve inventory → charge payment → confirm order → notify customer; support order cancellation/refund.
- **Non-functional**: no double-charging, no overselling inventory, traceable end-to-end, deployable independently per service, resilient to a single downstream service being temporarily degraded.

### High-level design
```
Client → Gloo Gateway → Order Service
                             │
                 ┌───────────┼────────────┐
                 ▼           ▼            ▼
          Inventory Svc  Payment Svc  (async) Kafka → Notification Svc
```

### Key design decisions, each with the trade-off named explicitly

**1. Saga pattern (orchestrated) for the order-placement flow, not a distributed transaction.**
- *Alternative rejected*: a two-phase-commit-style distributed transaction across services.
- *Why*: 2PC couples services tightly (all must be available and fast simultaneously) and doesn't fit a database-per-service architecture cleanly.
- *Trade-off accepted*: eventual consistency — for a brief window, an order might exist in "pending" state while inventory/payment steps are still resolving. The system must expose that state honestly (not pretend it's instantly consistent) and have compensating actions (release inventory reservation, refund) if a later step fails.
- *Orchestration vs. choreography*: I'd choose **orchestration** here (Order Service acts as the saga orchestrator, explicitly calling Inventory then Payment then publishing a confirmation event) over choreography, because the number of steps and compensating actions (a failed payment must trigger an inventory release, which itself needs confirmation) is complex enough that an explicit, debuggable sequence beats a web of implicit event reactions — worth being able to say *why* you didn't pick the "more decoupled-sounding" option.

**2. Synchronous calls for Inventory reservation and Payment charge; asynchronous (Kafka) for Notification.**
- *Why*: the customer needs to know *now* whether their order succeeded (inventory/payment must resolve before responding) — that's a genuine synchronous need. Notification has no such requirement; a customer doesn't need their confirmation email synchronously blocking the checkout response.
- *Trade-off*: the synchronous inventory/payment chain is exactly where cascading-failure risk lives (A32) — mitigated with strict timeouts, Resilience4j circuit breakers at the app layer, *and* Gloo Mesh-level retry/outlier-detection at the network layer (defense in depth, not redundant).

**3. Database-per-service (Order DB, Inventory DB, Payment DB — separate schemas, separate ownership).**
- *Alternative rejected*: one shared order-domain database.
- *Trade-off*: gives up cross-service SQL joins and requires the Outbox pattern (A4) for reliably publishing order-state-changed events in the same transaction as the DB write — real added complexity, accepted because it's what actually gives each team independent deployability and schema evolution.

**4. Idempotency keys on the Payment charge call specifically.**
- *Why*: a network blip causing the Order Service to retry the payment call must never double-charge — the Payment Service checks an idempotency key (e.g., order ID + attempt) before processing, per A33.

**5. Gloo Mesh for east-west traffic; Gloo Gateway for north-south.**
- STRICT mTLS between Order/Inventory/Payment (internal trust); JWT validation at the gateway for the external client.
- Canary rollout capability for Order Service releases via `VirtualService` traffic splitting (A29) — since Order Service is the highest-risk-to-change component (touches every flow), gradual rollout matters most there.

**6. CI/CD**: each service has its own thin Jenkinsfile referencing a shared pipeline library (A16); same-artifact promotion (A17) dev→staging→prod; production deploy gated behind manual approval given this touches payment/money-movement logic.

**7. Observability**: correlation ID generated at the Gloo Gateway, propagated through Order→Inventory→Payment synchronous calls and into the Kafka event heading to Notification — a single trace ID lets you reconstruct one customer's entire order journey across all four services in ELK/Kibana, per A24.

### The single trade-off worth defending if pushed
> "The biggest trade-off in this design is accepting eventual consistency and a temporary 'pending' order state in exchange for service autonomy — a distributed transaction would give stronger consistency guarantees but would couple every service's availability together, which is a worse trade for a system where Inventory and Payment genuinely need to scale, deploy, and fail independently."

---

# PART C — SITUATIONAL SCENARIO 2: PRODUCTION INCIDENT

## "During a flash sale, Order Service traffic spiked 20x. Inventory Service started timing out, and 10 minutes later Payment Service also became unresponsive, even though nothing changed in Payment's own code or infra."

Walk this the same way as any strong incident story: **Detect → Investigate → Root Cause → Contain → Recover → Prevent Recurrence.**

### Detect
- Kubernetes HPA metrics and Gloo Mesh dashboards show Order Service pod count climbing rapidly; ELK-based error-rate dashboards show a spike in `5xx` responses from Inventory Service first, then Payment Service several minutes later — the staggered timing itself is a clue.

### Investigate
- Distributed traces (A24) show Order Service requests to Inventory Service taking increasingly long, eventually timing out — but *Inventory Service's own CPU/memory metrics look normal*, ruling out "Inventory just needs more replicas" as the sole explanation.
- Checking Payment Service: its metrics also look infrastructurally healthy, yet it's failing — meaning the *cause* isn't in Payment's own resources either.

### Root cause
- **Thread pool / connection pool exhaustion cascading across services** — classic pattern: Order Service's synchronous calls to Inventory had no circuit breaker and a generous timeout; under 20x load, Inventory's *own* downstream (a database connection pool) became the actual bottleneck, causing Inventory calls to hang rather than fail fast. Order Service's threads calling Inventory piled up waiting, exhausting Order Service's own thread pool. Because Payment Service was called from the *same* Order Service request thread pool (not a separate bulkhead), Order Service ran out of capacity to make Payment calls at all — not because Payment was slow, but because **Order Service itself was starved**, and every downstream call routed through its exhausted pool looked equally broken from the outside.

### Contain
- Immediately reduce load: Gloo Mesh's rate limiting at the gateway is tightened temporarily to shed excess incoming traffic rather than let it keep queuing against an already-starved Order Service.
- Manually scale Inventory Service's *database connection pool* and add read replicas if the DB itself was the real bottleneck (confirmed via ELK logs showing DB connection wait times spiking).

### Recover
- Once Inventory's actual bottleneck (DB connection pool) is relieved, Order Service's calls to it start succeeding again, its own thread pool drains, and Payment calls resume — the cascade unwinds in reverse order from how it cascaded.

### Prevent recurrence — the trade-off-laden part of the answer
This is where a strong candidate shows judgment, not just "we fixed it":

1. **Add Resilience4j Bulkhead isolation** (A32) — separate thread/connection pools *per downstream dependency* within Order Service, so Inventory's slowness can never starve Payment calls again. *Trade-off*: more configuration complexity and slightly more total resource overhead (multiple smaller pools instead of one shared large one) — accepted because the alternative is exactly the cascading failure just experienced.

2. **Add circuit breakers with a fail-fast timeout on the Inventory call** — once Inventory's error/latency rate crosses a threshold, Order Service stops waiting and fails fast (with a clear "try again shortly" response) instead of queuing requests indefinitely. *Trade-off*: during a genuine, brief blip, some legitimate requests get an early failure they might have otherwise succeeded on with more patience — accepted because the alternative (unbounded queuing) is what caused system-wide starvation.

3. **Scale Inventory's HPA on a more predictive signal** (e.g., queue depth or request latency, not just CPU, per A21) — CPU looked fine while the real bottleneck was DB connection saturation, a case where CPU-based autoscaling gave false confidence.

4. **Load-test the *actual* dependency chain, not just each service in isolation** — the flash-sale traffic pattern should have been simulated end-to-end (Order → Inventory → its DB) before the real event, which likely would have surfaced the connection-pool ceiling in a controlled test instead of production.

5. **Reconsider whether inventory reservation needs to be fully synchronous at this volume** — a genuinely deeper architectural question worth raising: could inventory be optimistically reserved with async confirmation and a compensating cancellation on failure (leaning further into the Saga pattern from Scenario 1), reducing the synchronous dependency chain's fragility under extreme load? *Trade-off*: more complexity and a slightly worse "instant" UX guarantee (the customer's confirmation might take slightly longer / be provisional) in exchange for the system no longer having a single synchronous chain that can cascade end-to-end under load.

### The single trade-off worth defending if pushed
> "The core lesson wasn't 'Inventory needed to scale' — it's that **no service should let a slow downstream dependency consume shared, un-partitioned capacity**. Bulkheading and circuit breakers cost some complexity and occasionally reject a request that might have eventually succeeded, but that's a far better trade than one dependency's degradation being able to take down services that have no direct relationship to it."

---

# PART D — QUICK TRADE-OFF REFERENCE TABLE

| Decision | Option A | Option B | Default Lean |
|---|---|---|---|
| Cross-service data | Database per service | Shared database | Per-service, accept eventual consistency |
| Cross-service transaction | Saga (orchestrated/choreographed) | Distributed transaction (2PC) | Saga — services stay independently available |
| Order-flow coordination | Orchestration | Choreography | Orchestration once steps/compensations get complex |
| Service communication | Synchronous (REST/gRPC) | Asynchronous (events/queue) | Sync only when caller needs an immediate answer |
| External vs internal trust | JWT at gateway | mTLS in mesh | Both — different trust boundaries |
| Resilience layering | App-level (Resilience4j) | Mesh-level (Envoy) | Both — defense in depth |
| Autoscaling signal | CPU/memory | Custom metric (queue depth, latency) | Custom metric for I/O-bound/queue-driven services |
| Deployment strategy | Big-bang cutover | Canary / Strangler Fig | Incremental — lower risk, reversible |
| CI/CD artifact handling | Rebuild per environment | Promote same artifact | Promote same artifact — trustworthy rollback |
| Namespace granularity | Coarse (per team) | Fine (per service/env) | Fine, but automated — blast-radius containment |

---

## HOW TO DELIVER ANY OF THIS LIVE

For any question in Part A, answer in this shape: **the mechanic → the alternative you didn't pick → the specific cost you accepted → a one-sentence tie to a real (or realistic) scenario.** For the two full scenarios in Parts B and C, the structure to reuse for *any* new scenario an interviewer throws at you is identical: **state requirements first, draw the flow, name each decision's trade-off explicitly, and close with the one trade-off you'd defend hardest under pushback.**
