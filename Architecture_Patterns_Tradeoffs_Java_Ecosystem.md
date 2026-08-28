# Architecture, Design Patterns, System Trade-offs & Java Ecosystem
### Deep Dive — Self-Service Onboarding & Multi-Cluster Deployment Platform, Basic → Advanced

This document assumes you've already got the "what/why" (project narrative) and the "how to build it" (design & implementation doc). This one goes one layer deeper on four things interviewers specifically probe once they believe you built the thing: **the architectural style choices, the named design patterns underneath them, the trade-offs you'd defend under pushback, and the Java/JVM ecosystem specifics that make the Spring Boot side of this platform actually perform at scale.**

---

## PART A — SYSTEM DESIGN ARCHITECTURE, BASIC → ADVANCED

### A.1 Architectural style — what this system *is*, named precisely

At the top level this is a **control-plane / data-plane split architecture** — a distinction worth naming explicitly because it's exactly how Kubernetes, service meshes, and this platform's own onboarding system are all structured, and recognizing the pattern once means recognizing it everywhere:

- **Control plane** = the systems that make *decisions* and *declare desired state*: the `service-catalog` descriptor, the Terraform/Jenkins onboarding automation, the Kubernetes API server, Gloo Mesh's config plane, Strimzi's Kafka operator.
- **Data plane** = the systems that *do the actual work* based on that declared state: the running Spring Boot pods, the Envoy sidecars proxying real traffic, the Kafka brokers moving real messages.

**Basic level understanding**: know the split exists.
**Advanced level understanding**: know *why* it's separated — the control plane can be slow, complex, even briefly unavailable, and the data plane keeps working (an Envoy sidecar continues proxying traffic using its last-received config even if the mesh control plane is temporarily unreachable; a running pod keeps serving traffic even if the Kubernetes API server is down). This is a **resilience property that falls out of the architectural split itself**, not something anyone had to explicitly code — a very good thing to point out unprompted, because it shows you understand *emergent* system properties, not just component inventories.

### A.2 The platform as a Reconciliation / Control-Loop system

This is the single most important architectural concept to name precisely, because it's what Kubernetes itself is built on, and this platform's onboarding automation deliberately mirrors it:

**The pattern**: observe current state → compare to desired state (the descriptor) → take action to close the gap → repeat. This is the **Kubernetes Operator pattern** (also called the **reconciliation loop** or **control loop pattern**), and it's why the onboarding design in the companion doc uses Terraform (which is fundamentally a reconciliation tool — "make reality match this declared state") rather than imperative scripts ("do these steps in order").

**Basic**: a script that creates a repo once.
**Advanced**: a system that can be **run again safely at any time** and will only make the changes needed to close the gap between current and desired state — meaning a partial failure (Section on Challenge Scenarios in the earlier doc) is recoverable by simply re-running reconciliation, not by manually figuring out which of the five systems got provisioned and which didn't.

**Interview angle**: if asked "how would you make onboarding resilient to partial failure," naming **idempotent reconciliation over imperative scripting**, and explicitly citing the Kubernetes Operator pattern as the inspiration, is a strong, precise answer.

### A.3 Layered view of the whole system (basic → advanced read)

```
Layer 5: Developer Experience     service-catalog descriptor, PR-based self-service
Layer 4: Control Plane            Terraform, Jenkins onboarding jobs, Gloo Mesh config, Strimzi operator
Layer 3: Orchestration            Kubernetes/OpenShift/EKS scheduling, Helm releases
Layer 2: Data Plane / Mesh        Envoy sidecars, mTLS, traffic routing, Kafka brokers
Layer 1: Application              Spring Boot services (business logic)
Layer 0: Observability (cross-cutting, touches every layer)   Datadog, logs, traces
```
**Basic reading**: each layer does one job.
**Advanced reading**: notice that **Layer 0 (observability) and security are cross-cutting concerns that intentionally live partly in every other layer**, not as a separate layer bolted on top — this is a direct architectural expression of **separation of cross-cutting concerns**, implemented via the sidecar pattern (Section B) rather than application code, which is *why* a Go service and a Java service onboarded onto this platform get identical mTLS/tracing/logging behavior without either language's code implementing it directly.

### A.4 Multi-region / multi-cluster topology, basic → advanced

**Basic**: "we have clusters in multiple regions for resilience."

**Intermediate**: distinguish the topology models by name —
- **Active-active**: multiple regions simultaneously serve live traffic (load-balanced/geo-routed), each capable of absorbing the other's load on failover. Higher complexity (data replication/consistency questions), better resource utilization and lower failover time.
- **Active-passive**: one region serves traffic, another stands by, promoted only on failure. Simpler to reason about (no split-brain/dual-write concerns), but the standby capacity sits mostly idle, and failover itself takes time and carries risk (a rarely-exercised failover path is a classically under-tested code path).

**Advanced**: articulate *which model applies to which workload class in this platform, and why they're not uniform*:
- The **EU cluster is active for EU-resident data specifically because of data-residency requirements** (regulatory, not purely technical) — it isn't a failover target for US traffic at all; it's a genuinely separate active region serving its own traffic for legal reasons, a nuance worth stating because "multi-region" often gets treated as a single undifferentiated concept when in this platform it actually serves **two different purposes simultaneously** (US-East/US-West could be active-active for resilience/latency; EU is active for residency, not resilience).
- **CAP theorem trade-off, stated precisely per data flow**: cross-region Kafka-based event propagation (Section 9 of the design doc) is explicitly **AP-leaning** (available, eventually consistent) — a design choice, not an oversight, because a US-East outage should not make EU services unable to process new events, only temporarily behind on receiving cross-region ones. Meanwhile, within a single region/cluster, the mesh's synchronous service-to-service calls lean **CP** — a caller would rather get a fast, clear failure (circuit breaker open) than a slow, possibly-stale response, for most of these compliance-adjacent business calls.

**Interview angle**: being able to say "this system doesn't have *one* CAP trade-off, it has different trade-offs *per data flow*, and here's why each one was chosen deliberately" is a distinctly senior/staff-level answer versus a junior answer that treats CAP as a single, whole-system dial.

---

## PART B — DESIGN PATTERNS CATALOG, MAPPED TO THIS SYSTEM

Organized basic (classic OOP/GoF) → intermediate (enterprise) → advanced (distributed systems/cloud-native). For each: the pattern, where it's used here, and the one-sentence "why this and not the naive alternative."

### B.1 Basic — Classic OOP / GoF patterns

| Pattern | Where in this system | Why |
|---|---|---|
| **Template Method** | Jenkins shared pipeline (`springBootServicePipeline`) | The shared library owns the stage skeleton (build→scan→deploy); each service only supplies its descriptor — the "hook" data, not new steps |
| **Strategy** | Helm values-per-environment; deploy behavior (canary vs blue-green) selectable per service | Swap the deployment/traffic-splitting algorithm without changing the deploying code itself |
| **Builder** | Terraform HCL resource composition; Helm chart values composition | Complex object (a fully-configured namespace, a fully-configured deployment) assembled from composable, named parts rather than one giant constructor call |
| **Facade** | The API Gateway (Section 8 of design doc) | External consumers get one simple, stable interface hiding N internal services' complexity/churn |
| **Chain of Responsibility** | Envoy filter chain (Gloo Mesh); Spring Security filter chain within each service | Ordered, composable request-processing steps (auth, rate-limit, routing), each free to short-circuit the chain |
| **Adapter** | Legacy SOAP-to-REST migration (companion resume doc); Kafka Connect connectors bridging external systems to the event backbone | Let an old/foreign interface participate in the new architecture without rewriting the old side |
| **Decorator** | Envoy sidecar injection itself (`sidecar.istio.io/inject: "true"`) | Adds behavior (mTLS, retries, tracing) around an existing pod without modifying the pod's own container image |

### B.2 Intermediate — Enterprise patterns

| Pattern | Where | Why |
|---|---|---|
| **Repository / DTO / DAO** | Every Spring Boot service's persistence layer | Covered in depth in the companion 360° doc — decouples persistence model from wire contract |
| **Unit of Work** | JPA persistence context / `@Transactional` boundaries per service | Batches related changes into one atomic commit |
| **Circuit Breaker** | Resilience4j (app layer) + Envoy outlier detection (mesh layer) — deliberately at **both** layers | Defense in depth: app-level circuit breaking protects against a slow dependency even if mesh-level detection hasn't tripped yet, and vice versa |
| **Bulkhead** | Namespace ResourceQuotas; separate thread/connection pools per downstream dependency in Resilience4j | Isolates failure — one overwhelmed dependency's thread/connection exhaustion can't starve calls to a healthy dependency, exactly like a ship's bulkheads containing flooding to one compartment |
| **Retry (with backoff)** | Resilience4j `@Retry`; Kafka producer retry config | Absorbs transient failures without manual intervention, bounded so it doesn't become a retry storm |

### B.3 Advanced — Distributed systems / cloud-native patterns

| Pattern | Where | Why |
|---|---|---|
| **Sidecar** | Envoy (mesh), Datadog log/trace agent | Cross-cutting infra concerns live outside the app process, language-agnostic, upgraded independently of app deploys |
| **Ambassador** | Envoy acting as an outbound proxy for a service's egress calls (routing, retry, mTLS origination) | The app makes a plain local call; the sidecar handles the network complexity — app code stays simple |
| **Service Mesh** (a composition of Sidecar + Ambassador + centralized control plane) | Gloo Mesh across all clusters | The named, industry-standard term for "sidecar-based cross-cutting network concerns at scale" — say this term explicitly, interviewers listen for it |
| **API Gateway** | Gloo Gateway | Single north-south entry point; decouples external contract from internal service topology |
| **Backend for Frontend (BFF)** | (if/where a dedicated aggregation service composes multiple backend calls for a specific external consumer) | Tailors a composed API shape per consumer type without forcing internal services to change their own contracts |
| **Publish-Subscribe** | Kafka topics (registration.status.changed, etc.) | Decouples publisher from consumer count/lifecycle — new consumers can be added without publisher changes |
| **Competing Consumers** | Kafka consumer groups scaling horizontally | Multiple instances share the work of one logical stream, scaling throughput linearly with instance count |
| **Idempotent Receiver** | Every Kafka consumer (idempotency-key + hash pattern from the companion docs) | Required because Kafka's at-least-once delivery guarantees mean redelivery *will* happen |
| **Outbox Pattern** (advanced — worth knowing even if not literally implemented everywhere) | Where a service must atomically update its own DB *and* publish an event | Writes the event to an "outbox" table in the same DB transaction as the business change, then a separate relay publishes it to Kafka — avoiding the classic **dual-write problem** (DB commit succeeds, Kafka publish fails or vice versa, leaving state and events inconsistent) |
| **Strangler Fig** | SOAP-to-REST migration; more generally, any golden-path template rollout to *existing* pre-platform services | Incremental migration with a working system at every intermediate step, versus a risky big-bang cutover |
| **GitOps / Declarative Reconciliation** | The entire onboarding chain (Section A.2) | Desired state in Git, actual state continuously reconciled toward it — auditable (every change is a diff-able commit) and recoverable (re-apply is always safe) |
| **Leader Election** | Kafka consumer group coordination; Kubernetes controller-manager itself | Ensures exactly one active coordinator for a given resource/partition at a time in a distributed, multi-instance system, without a single hardcoded "the leader is always instance X" assumption |
| **Health Endpoint Monitoring** | Spring Boot Actuator `/actuator/health/liveness` and `/readiness`, consumed by Kubernetes probes | A standardized, machine-checkable contract for "is this instance able to do work" — distinct liveness (should I be restarted) vs. readiness (should traffic be routed to me) semantics matter and are a common interview gotcha |

### B.4 The single most important pattern-selection insight to voice

> "None of these patterns exist in isolation — the platform's resilience comes from **layering complementary patterns at different levels of the stack**: Bulkhead + Circuit Breaker at the app layer, Circuit Breaker/outlier detection again at the mesh layer, ResourceQuota-based bulkheading at the namespace layer, and cluster/region isolation at the infrastructure layer. No single pattern is trusted to be sufficient alone — that's a deliberate defense-in-depth architectural stance, not redundant duplication."

This sentence, or something like it, is worth having memorized verbatim — it's the kind of line that makes an interviewer sit up.

---

## PART C — SYSTEM DESIGN TRADE-OFFS, BASIC → ADVANCED

Structured as **decision → alternative → what you gain → what you give up** — this is the format that reads as judgment rather than a features list.

### C.1 Descriptor-driven fan-out automation vs. manual per-system provisioning
- **Gain**: consistency, speed, auditability, single source of truth.
- **Give up**: flexibility for genuinely one-off/unusual services that don't fit the golden path well — these either need an "escape hatch" (a deliberately supported way to deviate, tracked as tech debt) or friction until the platform's template evolves to support the new case. **Advanced point**: a platform that has *zero* escape hatch tends to accumulate shadow-IT workarounds; a platform with *too easy* an escape hatch loses the consistency benefit entirely — the trade-off is in calibrating friction, not eliminating it.

### C.2 Two Kubernetes distributions (OpenShift + EKS) vs. one
- **Gain**: each workload class runs on infrastructure actually suited to its constraints (regulatory posture vs. cloud-native elasticity).
- **Give up**: operational complexity — two sets of platform-team expertise, two upgrade cadences, two sets of edge-case quirks to know, and the golden-path templates/Helm charts must work correctly on both (an OpenShift-specific SecurityContextConstraints requirement that doesn't exist on vanilla EKS is a real, concrete example of this cost).
- **Advanced framing**: this is fundamentally a **build-vs-buy-vs-consolidate** decision made at the infrastructure level, and the right answer changes as the org's workload mix changes — a platform team should periodically revisit "do we still need both," not treat it as permanently settled.

### C.3 Shared Helm library chart vs. chart-per-service autonomy
- **Gain**: platform-wide consistency and one-line rollout of standards (Section 4 of design doc).
- **Give up**: individual service teams' ability to deviate from the base template for a genuinely unusual requirement, and a **coupling risk**: a bug in the shared library chart is a bug in *every* service simultaneously — worth stating explicitly, because it's the direct cost of the DRY benefit, and it's why the library chart itself needs its own rigorous versioning/testing/staged-rollout discipline (never `latest`, always a pinned version bumped deliberately per service).

### C.4 Synchronous mesh calls vs. Kafka for service-to-service communication
- Already covered in depth in the design doc (Section 9) — the advanced framing worth adding here: this isn't a one-time architectural choice, it's a **per-interaction decision** made by whoever designs each new integration, and the platform's job is to make the *async* path easy enough (auto-provisioned topics, schema registry, idempotent-consumer scaffolding in the golden-path template) that engineers don't default to synchronous calls purely because they're more familiar, even when async is the architecturally correct choice.

### C.5 Centralized platform team vs. federated/embedded ownership (Conway's Law angle)
- **Gain of centralization** (a dedicated platform team owning the golden path, shared charts, onboarding automation): consistency, deep expertise concentrated where it matters, less duplicated effort across product teams.
- **Give up**: the platform team can become a bottleneck/single point of prioritization contention if every product team's request routes through it; **Conway's Law** predicts the platform's own architecture will mirror this org structure — a centralized team tends to produce a genuinely centralized, consistent control plane (which is what this design has), whereas a federated model would more likely produce inconsistent, per-team-owned automation, trading consistency for team autonomy and reduced central bottleneck risk.
- **Advanced point worth making**: the "Platform as a Product" principle (companion doc, Section 17) is specifically the mitigation for the centralization bottleneck risk — treating internal teams as customers with a genuine feedback loop (not a ticket queue) is what keeps a centralized platform team from becoming purely a blocker.

### C.6 Idempotent, retryable, multi-step onboarding vs. one big transactional script
- **Gain**: partial failures are recoverable by simple re-run; each step (GitHub, namespace, mesh route, Kafka topics, dashboard) can fail and be retried independently, which is also individually debuggable.
- **Give up**: there's no true cross-system "transaction" — you cannot atomically roll back a GitHub repo creation and a Kubernetes namespace creation together as one unit the way a database transaction would. **Advanced framing**: this is the **Saga pattern's** fundamental trade-off, applied at infrastructure-provisioning scale rather than the more commonly-cited business-transaction scale — each step needs its own compensating action (or, as designed here, needs to simply be safely re-runnable/idempotent, which is a lighter-weight alternative to writing explicit compensating actions for every step).

### C.7 mTLS STRICT mesh-wide vs. permissive/opt-in
- **Gain**: no possibility of accidental plaintext east-west traffic, even from a misconfigured or forgotten service.
- **Give up**: **every** workload in the mesh must have a sidecar correctly injected and functioning, or it simply cannot communicate at all — meaning a sidecar-injection failure becomes an availability incident, not a silent security gap. This is a real trade-off (fail-closed security vs. fail-open availability) worth naming precisely: **the design deliberately prefers failing closed on a security boundary**, and compensates for the availability risk with the pre-provisioning synthetic reachability check from the design doc's Section 10 rather than by loosening the security posture.

---

## PART D — JAVA & JVM ECOSYSTEM, BASIC → ADVANCED (this platform's Spring Boot workloads specifically)

### D.1 Basic — the JVM inside a container, and why it used to lie to itself
**The historical problem** (pre Java 8u191 / fully fixed by Java 10+): the JVM used to read the **host machine's** total CPU/memory when deciding default heap size and default parallelism (`-XX:+UseParallelGC` thread counts, default `-Xmx`), completely ignoring the **cgroup/container resource limits** Kubernetes actually enforced on the pod. A pod given a 1Gi memory *limit* running on a 64GB host node could have a JVM default to sizing its heap as if it had access to a large fraction of that 64GB — guaranteeing an OOMKill the moment real traffic arrived.

**The fix, and what to actually configure**:
```
-XX:+UseContainerSupport          # default true since Java 10 -- JVM reads cgroup limits, not host totals
-XX:MaxRAMPercentage=75.0         # heap sized as a % of the CONTAINER's memory limit, not a fixed -Xmx
-XX:InitialRAMPercentage=50.0
```
**Why this matters for this platform specifically**: the golden-path Dockerfile/Helm chart (Section 4/5 of the design doc) bakes these flags in by default — the ResourceQuota and per-container `resources.limits.memory` from Kubernetes and the JVM's own heap-sizing must agree, or you get either wasted memory (heap sized far below the container limit, container's memory allocation mostly unused) or OOMKills (heap sized above the container's actual enforced limit). **This is a very concrete, very real "what could go wrong" interview answer** — "a service that worked fine locally OOMKilled immediately in Kubernetes because the container memory limit was 512Mi but the JVM's default heap sizing assumed it had access to the node's full memory" is a genuinely common real-world incident, worth having ready.

### D.2 Intermediate — GC selection under container constraints
- **G1GC** (default since Java 9) — a solid general-purpose default, balances throughput and pause time, works reasonably at most heap sizes typical of a microservice (hundreds of MB to a few GB).
- **ZGC / Generational ZGC** (Java 17/21, from the earlier version-evolution doc) — worth reaching for specifically on **latency-sensitive services with larger heaps** where G1's pause times (still tens of milliseconds under load) aren't acceptable — sub-millisecond pause times largely independent of heap size. **Advanced trade-off to state**: ZGC trades some throughput and uses more memory overhead (for its colored-pointers/multi-mapping technique) in exchange for that pause-time guarantee — not a strictly-better default, a deliberate choice for a specific latency SLA.
- **Practical platform decision**: the golden-path template defaults to G1GC (safe, well-understood, fine for the vast majority of services), with ZGC as an **opt-in override** in a service's `values.yaml` for the specific tier-1, latency-SLA-bound services that need it — again the "sane default, explicit opt-in escape hatch" pattern from Section C.1.

### D.3 Basic → Advanced — JVM startup time and Kubernetes autoscaling

**The problem, stated precisely**: Kubernetes' Horizontal Pod Autoscaler can decide to scale up in seconds, but a JVM-based service takes meaningfully longer than that to be *actually useful* — class loading, Spring context initialization/bean wiring (Section 1 of the companion 360° doc), and JIT warm-up (the JVM interprets bytecode initially and only compiles hot paths to native code after enough invocations) all add up to several seconds of "technically running but not yet performing well" time. During a sudden traffic spike, new pods might be scheduled, pass their readiness probe, and start receiving traffic — while still running cold/interpreted and therefore slower than steady-state, briefly amplifying rather than relieving the load spike.

**Mitigations, basic → advanced**:
- **Basic**: tune the readiness probe's `initialDelaySeconds`/success threshold conservatively so Kubernetes doesn't route traffic to a pod that's merely "started" but not yet warmed up.
- **Intermediate**: **CDS (Class Data Sharing) / AppCDS** — pre-computes and shares parsed class metadata across JVM invocations, meaningfully cutting startup time, applicable even on standard Java 17/21 without switching runtime models.
- **Advanced**: **GraalVM Native Image** (via Spring Boot 3's AOT/native support, from the version-evolution doc) — ahead-of-time compiles the whole application to a native executable, cutting startup from seconds to **milliseconds** and memory footprint dramatically — the strongest fix for this problem, at the cost of longer build times, some reflection/dynamic-class-loading library incompatibilities needing explicit configuration, and losing the JIT's ability to *further* optimize hot paths beyond what AOT compilation already did (native image trades peak-throughput ceiling for startup-time floor — worth stating as an explicit trade-off, not a strictly-better option).
- **Most advanced (worth knowing exists)**: **Project CRaC (Coordinated Restore at Checkpoint)** — snapshots a fully warmed-up, already-JIT-optimized JVM process to disk, and restores from that checkpoint on a new instance in a fraction of a second, aiming to get startup-time benefits *without* native image's AOT-compilation throughput trade-off. Newer and less universally adopted than native image, but the right "cutting edge, here's where this is heading" answer if asked what's next after native image.

**Interview angle**: "how would you make a Spring Boot service scale fast enough under Kubernetes HPA during a traffic spike" is a great, specific, very current question, and walking through this basic→advanced ladder (readiness-probe tuning → AppCDS → native image → CRaC) is a strong, layered answer showing you know there isn't one silver bullet.

### D.4 Java concurrency across this platform, tied back to the version-evolution doc
- Every **Kafka consumer** processing records with a per-record downstream lookup (Section 6 of the design doc) is a textbook fit for **Java 21 virtual threads** (from the earlier code-snippet companion) — I/O-bound, high-fan-out, exactly the profile virtual threads scale well without manual thread-pool tuning.
- Every **fan-out call to multiple backend services behind the gateway/BFF** (Section B.3's BFF pattern) is a textbook fit for **Structured Concurrency** (Java 21 preview / 25 finalized) — the parent-child cancellation-on-failure semantics map directly onto "if one of the three calls I fanned out fails, cancel the other two and fail the composed response cleanly," which is exactly the coordination bug class structured concurrency eliminates.
- **Advanced point connecting Java's evolution to the platform's own architecture**: the platform's use of **Envoy sidecars for retries/circuit-breaking (Section B.3)** and Java's own **virtual threads/Resilience4j at the app layer** are solving *overlapping but not identical* problems — the sidecar handles it at the network layer regardless of language, virtual threads make the *application's own* concurrency simpler and cheaper regardless of whether a mesh is present at all. A service without a mesh sidecar still benefits fully from virtual threads; a service with a mesh sidecar still benefits from not writing thread-pool-constrained blocking code. **They're complementary, not redundant** — a good, precise answer if asked "doesn't the mesh already handle resilience, why bother with Resilience4j/virtual threads too" (echoing the defense-in-depth point from Part B.4).

### D.5 Spring ecosystem components actually in play, mapped to platform layers

| Spring component | Role in this platform |
|---|---|
| **Spring Boot** | The application runtime itself — auto-config, embedded Tomcat, Actuator (health probes) |
| **Spring MVC** (or WebFlux for specific streaming/BFF cases) | REST controller layer behind the gateway |
| **Spring Data JPA / Spring Data MongoDB/Cassandra** | Persistence layer, per the polyglot-persistence discussion in the companion 360° doc |
| **Spring Security (OAuth2 Resource Server)** | Validates the JWT the gateway/edge already authenticated, at the app layer — second layer of the two-trust-model design (Part A/gateway JWT + mesh mTLS) |
| **Spring Kafka** | Producer/consumer abstraction over the raw Kafka client, with Spring-idiomatic `@KafkaListener` and container-factory-level configuration for concurrency, error handling, and dead-letter routing |
| **Spring Cloud (selectively)** | Where genuinely still useful post-Kubernetes-nativization (from the version-evolution doc's Section 7 discussion) — e.g., `@HttpExchange`-based typed clients for service-to-service calls, rather than the older Netflix-stack components largely superseded by mesh + Kubernetes-native discovery |
| **Micrometer** | The metrics-facade layer feeding both Actuator/Prometheus-format scraping *and* Datadog's APM — a good example of **coding to an abstraction (Micrometer's `MeterRegistry`)** rather than a vendor-specific metrics API, so swapping or adding an observability backend later doesn't touch application code — a direct, concrete application of the Dependency Inversion Principle from the companion doc's Section 1, now at the observability-integration level rather than the object-wiring level |

### D.6 Build tooling and dependency governance
- **Maven with a shared parent BOM** (Bill of Materials) — every golden-path service inherits a centrally-managed, tested-together set of dependency versions (Spring Boot's own BOM, plus the org's internal shared-library versions) rather than each service pinning versions independently and risking incompatible combinations — directly mirrors the Helm-library-chart consistency principle (Part C.3) at the *build* layer instead of the *deployment* layer. **Same underlying principle, applied twice, at two different layers of the stack** — a strong thing to point out explicitly if asked to compare them.
- **Testcontainers** in the CI pipeline (Section 8 of design doc's quality gates) — integration tests spin up real ephemeral Kafka/Postgres/Mongo instances in Docker during the Jenkins build, rather than mocking the persistence/messaging layer entirely — catching real serialization/schema/query issues that a pure-mock test suite would miss, at the cost of slower CI runs (a deliberate trade-off: this platform's pipeline reserves Testcontainers-based tests for the integration-test stage specifically, keeping the unit-test stage fast and fully mocked, per the test-pyramid principle from the companion 360° doc's Section 7).

---

## PART E — HOW THE FOUR THEMES TIE TOGETHER (the answer to "give me the 360° view")

If asked to synthesize everything in this document into one closing statement, this is the shape of a strong answer:

> "The architecture is a control-plane/data-plane split implementing a reconciliation/GitOps model, which is itself the Kubernetes Operator pattern applied to developer onboarding rather than just workload scheduling. That architecture is built from a deliberately layered stack of named design patterns — Sidecar and Ambassador for cross-cutting network concerns, Circuit Breaker and Bulkhead repeated at both the mesh and application layers for defense in depth, Saga-style idempotent multi-step provisioning instead of an impossible cross-system transaction. Every major decision in that stack — two Kubernetes distros, sync-vs-async communication, centralized platform ownership, mesh-wide strict mTLS — was a deliberate trade-off with a stated cost, not a default. And underneath the Spring Boot services specifically, the same 'sane default, explicit opt-in override' philosophy shows up again at the JVM level: G1GC by default with ZGC available for latency-sensitive tier-1 services, blocking code by default now that virtual threads make it scale, and a Maven BOM enforcing the same dependency-consistency discipline the Helm library chart enforces one layer up. It's the same small set of architectural principles — single source of truth, layered defense, sane-default-with-escape-hatch — applied consistently from the Git repo all the way down to the JVM heap flag."

Memorize the *shape* of that paragraph (four themes, each with one concrete example), not the exact words — being able to produce something like it unprompted is what "360°" actually looks like to an interviewer.

---

## PART F — QUICK-REFERENCE LADDER (basic → advanced, all four themes)

| Level | Architecture | Patterns | Trade-offs | Java/JVM |
|---|---|---|---|---|
| Basic | Know control plane vs data plane exists | Name Facade/Strategy/Template Method in the pipeline | Know "sync vs async" is a choice | Know `-Xmx` and container limits must agree |
| Intermediate | Explain why the split gives resilience for free | Name Sidecar/Circuit Breaker/Bulkhead and where each sits | State one gain and one cost per major decision | Know G1 vs ZGC trade-off; know AppCDS |
| Advanced | Explain per-data-flow CAP trade-offs, not one whole-system dial | Explain *why* patterns are layered (defense in depth) not redundant | Frame trade-offs as ongoing, revisitable decisions (Conway's Law, escape-hatch calibration) | Explain native image vs CRaC vs virtual threads as complementary, not competing, fixes to different problems |
