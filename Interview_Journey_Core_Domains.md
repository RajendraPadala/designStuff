# Interview Journey — Core Domain Reference
### Java Ecosystem · Spring Boot · Data & Persistence · Testing · DevOps & Cloud
### Basic → Advanced, with Functional Scenarios & Trade-offs

---

## HOW THIS DOCUMENT IS ORGANIZED

Each topic follows the same crisp structure:
- **Basic** — the mechanic, in plain terms.
- **Intermediate** — how it's actually used/configured in a real service.
- **Advanced** — the internals, edge cases, and performance/scale implications.
- **Trade-off table** — the options and when each wins.
- **Scenario** — a short, concrete "here's when this mattered" example, interview-ready.

Skim top-to-bottom for a first pass; use it as a lookup table before a specific interview.

---

# DOMAIN 1 — JAVA ECOSYSTEM

## 1.1 Java 17+ — Records, Sealed Classes, Pattern Matching

**Basic**
- **Record**: `record Point(int x, int y) {}` — immutable data carrier; compiler generates constructor, accessors (`x()`, `y()`), `equals()`, `hashCode()`, `toString()`.
- **Sealed class/interface**: `sealed interface Shape permits Circle, Square {}` — restricts which types may extend/implement it; the compiler knows the *complete* set of subtypes.
- **Pattern matching**: `if (obj instanceof String s)` binds `s` automatically, no manual cast; extended to `switch` for exhaustive, type-safe branching.

**Intermediate**
- Records used as **API DTOs** (immutable by construction — safe to pass across threads/queues) and as **compact value objects** in the domain model (e.g., `record Money(BigDecimal amount, Currency currency) {}` with a **compact constructor** for validation: `record Money(...) { Money { if (amount.signum() < 0) throw new IllegalArgumentException(...); } }`).
- Sealed types model **closed domain hierarchies** precisely — e.g., a payment result that can only ever be `Approved`, `Declined`, or `Pending` — the compiler enforces every consumer handles all three, forever, even as the codebase grows.
- Pattern matching in `switch` with **record deconstruction** (Java 21+): `case Circle(var radius) -> ...` — destructures a record's components directly in the case label.

**Advanced**
- Records **cannot** extend another class (they implicitly extend `Record`), can implement interfaces, and are implicitly `final` — meaning a record can never be a Mockito mock via subclassing (Mockito's inline mock maker, default since Mockito 5, handles this via bytecode manipulation instead of subclassing).
- Sealed + pattern matching gives **exhaustiveness checking at compile time** — removing a `default` branch is *safe* only when every permitted subtype is explicitly handled; adding a new permitted subtype later **breaks the build** everywhere it's not handled — this is a deliberate trade of a small amount of switch-statement verbosity for a strong compile-time correctness guarantee.
- Records interact with Hibernate 6+ as native **DTO projection targets** (constructor expressions map directly), removing the need for a manual mapper class purely to shape a query result.

**Trade-offs**

| Choice | Use when | Avoid when |
|---|---|---|
| Record vs. class | Immutable data, DTOs, value objects | Need mutability, builders with many optional fields, or JPA entities (entities need a no-arg constructor and mutable fields per the JPA spec) |
| Sealed hierarchy vs. open hierarchy | Domain concept has a known, closed set of variants (state machine states, event types) | Domain concept is meant to be extended by unknown future consumers (e.g., a plugin API) |
| Pattern-matching switch vs. instanceof chain | Dispatching over a sealed hierarchy | Dispatching over an open/unknown type set |

**Scenario**
> A feed-processing pipeline models incoming records as a sealed `FeedRecord` hierarchy (`StatusChange`, `Disclosure`, `Exam`). Six months later, a new record type (`Reinstatement`) is added to the feed spec. Adding it to the `permits` clause causes every existing `switch` over `FeedRecord` across the codebase to **fail to compile** until handled — catching every place that needed updating at build time, not as a silent runtime gap discovered in production three weeks later.

---

## 1.2 Virtual Threads (Java 21+)

**Basic**
- A **virtual thread** is a lightweight thread managed by the JVM, not the OS. Millions can exist concurrently. Blocking code (`Thread.sleep`, a blocking JDBC call, a blocking HTTP client call) still *looks* the same — you write ordinary synchronous code.

**Intermediate**
```java
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    tasks.forEach(t -> executor.submit(() -> process(t)));
}
```
- Under the hood: a virtual thread runs on a small pool of **carrier threads** (platform/OS threads). When a virtual thread performs a **blocking operation that's been made virtual-thread-aware** (most JDK I/O — sockets, files, JDBC via compatible drivers), it **unmounts** from its carrier thread, freeing the carrier to run another virtual thread, and **remounts** when the I/O completes.
- Spring Boot 3.2+: `spring.threads.virtual.enabled=true` runs the embedded Tomcat request-handling on virtual threads — existing blocking Spring MVC code gains I/O-bound scalability with **zero code changes**.

**Advanced**
- **Pinning**: a virtual thread that's inside a `synchronized` block (pre-Java 24 — this was improved in later releases) **cannot unmount** during a blocking call within that block — it pins its carrier thread, defeating the scalability benefit for that section. The fix historically: replace `synchronized` with `java.util.concurrent.locks.ReentrantLock` in hot, blocking-I/O-adjacent code paths, or upgrade to a JDK version where synchronized-block pinning was resolved.
- Virtual threads **do not help CPU-bound work** — no I/O wait means no opportunity to unmount; a CPU-bound task occupies its carrier thread exactly like a platform thread would.
- **`ThreadLocal` cost at virtual-thread scale**: `ThreadLocal` was designed assuming a bounded, small number of threads; with potentially millions of virtual threads, naive heavy `ThreadLocal` usage (e.g., large per-thread caches) can balloon memory. **Scoped Values** (finalized Java 25) are the purpose-built, cheaper, immutable-for-the-scope replacement for context propagation (e.g., a request's trace ID) in virtual-thread-heavy code.
- **Structured Concurrency** (finalized Java 25) — fan out to N virtual-thread subtasks as a single unit, with automatic cancellation of siblings on first failure — fixes the "forgot to cancel the other future" bug class from manual `ExecutorService` coordination.

**Trade-offs**

| Choice | Use when | Avoid when |
|---|---|---|
| Virtual threads | I/O-bound workloads: many concurrent blocking calls (DB, HTTP, downstream lookups) | Pure CPU-bound computation (use a bounded platform-thread pool sized to core count) |
| Virtual threads vs. WebFlux/reactive | Want simple, debuggable, synchronous-looking code that still scales | Need genuine backpressure control, streaming, or are already deeply reactive-invested |
| `synchronized` vs. `ReentrantLock` in hot paths | Simple, uncontended, short critical sections | Blocking I/O inside the critical section on a virtual thread (pinning risk pre-fix) |

**Scenario**
> A Kafka consumer processing feed records made a blocking downstream reference-data lookup per record. On a fixed thread pool (size 200, empirically tuned), throughput plateaued under load. Switching the consumer's processing executor to `newVirtualThreadPerTaskExecutor()` removed the pool-sizing guesswork entirely — the JVM scaled concurrency to the actual I/O-wait profile of the lookup calls, and the "what's the right pool size" tuning question simply stopped being relevant.

---

## 1.3 JVM Internals & Performance Tuning

**Basic — memory areas**
- **Heap**: object storage, garbage collected. Split conceptually into **Young Generation** (Eden + two Survivor spaces — most objects die young here) and **Old Generation** (long-lived objects, promoted after surviving enough young-gen collections).
- **Metaspace** (replaced PermGen since Java 8): class metadata — grows dynamically, off-heap, bounded by `-XX:MaxMetaspaceSize` if set (unbounded by default, a real leak risk with dynamic classloading — e.g., a misbehaving plugin system repeatedly loading classes without unloading).
- **Stack**: per-thread, method call frames, local variables — a `StackOverflowError` is this filling up (deep/infinite recursion).

**Intermediate — garbage collectors**
| GC | Model | Best for |
|---|---|---|
| **G1GC** (default since Java 9) | Region-based, splits heap into many regions, collects the most garbage-dense regions first | General-purpose default; balances throughput and pause time for most microservices |
| **ZGC / Generational ZGC** (Java 17/21+) | Concurrent, colored-pointer-based; pause times sub-millisecond, largely independent of heap size | Latency-sensitive services with large heaps where G1's tens-of-ms pauses aren't acceptable |
| **Parallel GC** | Stop-the-world, multi-threaded, throughput-optimized | Batch/offline jobs where pause time doesn't matter, only total throughput |
| **Shenandoah** | Concurrent, low-pause (alternative to ZGC, OpenJDK/Red Hat lineage) | Similar niche to ZGC; choice sometimes driven by JDK distribution availability |

**Advanced — container-aware sizing (the most commonly-missed production detail)**
```
-XX:+UseContainerSupport         # default true since Java 10 — reads cgroup limits, not host totals
-XX:MaxRAMPercentage=75.0        # heap as % of the CONTAINER's memory limit
-XX:InitialRAMPercentage=50.0
```
Without this awareness (pre-Java 10, or misconfigured), a JVM in a 512Mi-limited container can default heap sizing based on the **host node's** total memory, guaranteeing an OOMKill under real load — one of the most common "worked locally, crashed in Kubernetes" root causes.

**Advanced — diagnosing a production memory/GC issue**
- **Heap dump analysis** (`jmap -dump`, or automatic via `-XX:+HeapDumpOnOutOfMemoryError`) opened in Eclipse MAT/VisualVM — look for **dominator trees** (which object retains the most memory) and suspiciously large collection instances (a classic leak: an ever-growing static `Map` used as an ad-hoc cache with no eviction).
- **GC logs** (`-Xlog:gc*`) — a rising **old-generation occupancy after every full GC** (garbage collection not reclaiming memory back down to baseline) is the signature of a real leak, distinct from healthy sawtooth young-gen collection patterns.
- **Thread dumps** (`jstack`, or `kill -3`) — for diagnosing deadlocks, thread-pool exhaustion, or (in virtual-thread code) pinning.

**Trade-offs**

| Decision | Gain | Cost |
|---|---|---|
| G1 (default) vs ZGC | ZGC: near-zero pause | ZGC: higher memory overhead, somewhat lower peak throughput |
| Larger heap | Fewer GC cycles | Longer individual GC pauses (mitigated by ZGC), more memory cost |
| Native image (GraalVM) | Millisecond startup, low memory | Longer build times, some reflection-library friction, gives up further JIT optimization beyond AOT |
| `MaxRAMPercentage` tuning | Correct, container-safe heap sizing | Requires understanding the relationship between container limits and JVM defaults — easy to get wrong once |

**Scenario**
> A service passed load testing on a dev laptop (32GB RAM) but OOMKilled within minutes in Kubernetes (512Mi limit). Root cause: the base image's JVM defaulted heap sizing off the **node's** total memory (an older JDK, `UseContainerSupport` not effective for this metric), not the container's cgroup limit. Fix: explicit `-XX:MaxRAMPercentage=75.0` in the container's JVM flags, aligned with the Kubernetes `resources.limits.memory`, plus a load test actually run **inside** a correctly resource-constrained container, not on an unconstrained laptop.

---

# DOMAIN 2 — SPRING BOOT

## 2.1 Auto-Configuration

**Basic**: `@SpringBootApplication` = `@Configuration` + `@ComponentScan` + `@EnableAutoConfiguration`. Auto-configuration classes bundled in each "starter" jar are conditionally applied based on what's on the classpath and what properties are set.

**Intermediate**: the conditional annotations that drive it —
- `@ConditionalOnClass` — apply only if a specific class exists on the classpath (e.g., Kafka auto-config only activates if `KafkaTemplate` is present).
- `@ConditionalOnMissingBean` — apply only if the developer hasn't defined their own bean of that type — **this is how you override any default**: just declare your own `@Bean`.
- `@ConditionalOnProperty` — apply only if a property matches (e.g., `spring.threads.virtual.enabled=true`).

**Advanced**: auto-configuration classes are ordered (`@AutoConfigureOrder`, `@AutoConfigureBefore/After`) because some configurations must exist before others can sensibly apply. Debugging *why* an auto-configuration didn't apply: run with `--debug` (or `management.endpoint.conditions`) to get the **conditions evaluation report** — shows every auto-config class and exactly which condition matched or failed, turning "why isn't my DataSource configured" from guesswork into a direct answer.

**Trade-off**: auto-configuration accelerates getting a working service, but a developer who doesn't understand the underlying mechanism can be genuinely stuck when it *doesn't* do what they expect — the conditions evaluation report is the escape hatch worth knowing exists.

**Scenario**
> A `DataSource` unexpectedly wasn't auto-configured. Running with `--debug` showed `DataSourceAutoConfiguration` was **not applied** because `@ConditionalOnMissingBean` found an existing `DataSource` bean — one accidentally leaking in from a `@TestConfiguration` class picked up by an overly broad component scan. The conditions report turned a confusing "Spring is being weird" moment into a two-minute fix.

## 2.2 Production-Ready Features — Actuator & Metrics

**Basic**: `/actuator/health`, `/actuator/info`, `/actuator/metrics` — exposed endpoints for operational visibility, secured/exposed selectively (`management.endpoints.web.exposure.include`).

**Intermediate**:
- **Liveness vs readiness probes** — `/actuator/health/liveness` (should Kubernetes restart this pod?) vs `/actuator/health/readiness` (should traffic be routed to this pod right now?) — genuinely different questions: a pod might be alive but not ready (still warming up its connection pool), and conflating the two probes is a common, real misconfiguration causing traffic to hit a not-yet-ready pod.
- **Micrometer** as the metrics facade — `MeterRegistry` abstracts the actual backend (Prometheus format, Datadog, CloudWatch) so application code never depends on a specific vendor's metrics API — a direct application of coding to an abstraction.

**Advanced**: **custom `HealthIndicator`** implementations composing into overall health (e.g., a Kafka connectivity check, a downstream dependency check) — but deliberately **not** including slow/expensive checks in the *liveness* probe (a liveness check calling a flaky downstream dependency can cause Kubernetes to restart an otherwise-healthy pod because of someone else's outage — a classic cascading-failure-amplifier misconfiguration). Custom **`@Timed`/`Counter`/`Gauge`** Micrometer metrics for business-relevant signals (records processed, feed-run duration) alongside framework-level metrics.

**Trade-off**: exposing more Actuator endpoints gives more operational visibility, but is a real attack surface if not secured — `management.endpoints.web.exposure.include` should be scoped deliberately, and sensitive endpoints (`/actuator/env`, `/actuator/heapdump`) restricted to internal-only access or disabled entirely in production.

**Scenario**
> A liveness probe was implemented checking downstream Kafka connectivity. When Kafka had a brief, unrelated blip, **every** consuming pod's liveness probe failed simultaneously, and Kubernetes restarted them all at once — worsening the outage by dropping in-flight processing on every pod during the very incident recovery was needed most. Fix: move the Kafka check to the **readiness** probe only (stop routing new traffic, but don't kill the process), keep liveness limited to "is this JVM process itself responsive," a precise, deliberate distinction.

## 2.3 Spring MVC — Request Handling Deep Dive

**Basic**: `@RestController` + `@GetMapping`/`@PostMapping` — `DispatcherServlet` routes to the matching handler method via `HandlerMapping`.

**Intermediate**: **Filters** (Servlet-level, outside Spring's context — where Spring Security's chain lives, because auth must happen before controller dispatch) vs. **HandlerInterceptors** (Spring-MVC-level, `preHandle`/`postHandle`/`afterCompletion`, access to the resolved handler) vs. **`@ControllerAdvice`** (centralized exception handling, response-body advice).

**Advanced**: **`HttpMessageConverter`** chain (Jackson for JSON by default) — advanced customization (custom serializers/deserializers for specific types, e.g., a `Money` value object serializing as a structured `{amount, currency}` object rather than relying on default reflection-based serialization). **Content negotiation** (`Accept` header driving converter selection) for APIs supporting multiple representations. **Async request handling** (`Callable`/`DeferredResult`/`CompletableFuture` return types) — releases the request-handling thread while a long-running operation completes, relevant *before* virtual threads made this less necessary for the common case.

**Trade-off**: `@ControllerAdvice` centralizes exception-to-HTTP-status mapping (consistency), but can obscure *where* an exception actually originated if error responses aren't given enough context (a `traceId` in every error response, correlatable to logs, is the mitigation).

## 2.4 Spring Data

**Basic**: `JpaRepository<T, ID>` — CRUD auto-implemented from an interface; derived query methods (`findByStatusAndJurisdiction(...)`) generated from method name parsing.

**Intermediate**: `@Query` for JPQL/native SQL when derived-method naming gets unwieldy or a query needs specific joins/fetch behavior; `Pageable`/`Page<T>` for pagination; `Specification<T>` (Criteria API wrapper) for dynamically composable query predicates (search endpoints with many optional filters).

**Advanced**: **projection interfaces/DTO projections** to avoid loading full entities for read-only paths; **`@Modifying` + `@Query`** for bulk update/delete operations that bypass the persistence context (must manually consider stale first-level-cache state afterward — `clearAutomatically = true` flushes the persistence context); **auditing** (`@CreatedDate`/`@LastModifiedDate` via `@EnableJpaAuditing`) for automatic audit-column population, directly relevant to compliance-style systems needing "who/when" tracked without manual boilerplate in every service method.

**Trade-off**: derived query methods are fast to write but become unreadable past 3–4 conditions in the method name; `Specification`/Criteria API is more verbose but scales to genuinely dynamic, many-optional-filter search endpoints.

## 2.5 Spring Security

**Basic**: authentication (who) vs authorization (what they can do); `SecurityFilterChain` bean (Spring Security 6 lambda DSL, replacing the deprecated `WebSecurityConfigurerAdapter`).

**Intermediate**: `@PreAuthorize("hasAuthority('arl.read')")` method-level authorization (`@EnableMethodSecurity`); JWT resource-server support (`oauth2ResourceServer().jwt()`) — stateless, signature-validated-against-JWKS authentication requiring no session store.

**Advanced**: **custom `AuthenticationConverter`** for non-standard claim structures; **CORS configuration precision** (a permissive `allowedOrigins("*")` combined with `allowCredentials(true)` is actually rejected by browsers and is a common, confusing misconfiguration to know how to diagnose); **CSRF** — correctly disabled for stateless, token-authenticated APIs (no cookie-based session to forge against) but a real requirement to keep enabled for any cookie-session-based endpoints coexisting in the same application; **method security vs URL security layering** (URL-level `authorizeHttpRequests` as a coarse first gate, method-level `@PreAuthorize` as fine-grained per-operation control — deliberately layered, not either/or).

**Trade-off**

| Choice | Gain | Cost |
|---|---|---|
| JWT stateless auth | Horizontal scaling without shared session store | Revocation isn't free (needs a blocklist or short expiry + rotation) |
| URL-level only authorization | Simple, fast to reason about | Coarse — can't express "read own records only" style rules |
| Method-level `@PreAuthorize` | Fine-grained, close to the business logic it protects | Scattered across the codebase, harder to audit as a single list vs. a centralized URL config |

**Scenario**
> A `/arl/v1/advisors/{crdNumber}` endpoint correctly required authentication but had no check that the caller's scope actually covered that specific CRD number/jurisdiction — any authenticated caller could fetch **any** advisor's record. This is a textbook **Broken Object Level Authorization (BOLA)** finding — the fix layered a method-level `@PreAuthorize` (or a service-layer check) validating the resource-specific claim, not just "are you logged in."

---

# DOMAIN 3 — DATA & PERSISTENCE

## 3.1 JPA/Hibernate — N+1 and Entity Graphs

**Basic**: the **N+1 problem** — fetching N parent entities, then lazily triggering one additional query per parent to fetch a related collection = 1 + N queries instead of 1.

**Intermediate — fixes, in order of when to reach for each**:
1. **`JOIN FETCH`** in JPQL — one query, eager for this specific call site only, still returns managed entities.
2. **`@EntityGraph`** — declaratively defines a reusable fetch plan, applicable across multiple repository methods without repeating JPQL joins.
3. **DTO/record projection** — bypasses entity loading entirely for pure read paths; fastest, but no managed entities (can't be mutated and saved).

**Advanced**:
- **Batch fetching** (`@BatchSize` on a collection, or Hibernate 6's improved default batching) — instead of N individual queries for related collections, Hibernate issues **fewer, larger `IN (...)`-clause queries** — a middle ground between full N+1 and a full join, useful when a join would produce a large, wasteful Cartesian-product result set.
- **Second-level cache** (`@Cacheable`, backed by Ehcache/Infinispan/etc.) — caches entities **across sessions/transactions** (distinct from the first-level cache, which is per-persistence-context/transaction-scoped and always on) — genuinely useful for slow-changing reference data, dangerous for anything that must always reflect the latest write (stale second-level cache serving outdated compliance data is a real risk worth naming explicitly).
- **Detecting N+1 in practice**: enable `logging.level.org.hibernate.SQL=DEBUG` (or a dedicated tool like Hibernate's own statistics/`SessionFactory.getStatistics()`, or a library like `datasource-proxy`/p6spy) and look for a burst of near-identical queries differing only by a bound parameter, immediately following a single parent-fetch query.

**Trade-off**

| Fix | Best for |
|---|---|
| `JOIN FETCH` | Need managed entities, moderate related-collection size, single call site |
| `@EntityGraph` | Same fetch plan reused across multiple repository methods |
| DTO projection | Pure read/reporting path, no mutation needed — fastest |
| `@BatchSize` | Multiple different collections need loading, a full join would over-fetch |
| Second-level cache | Slow-changing, read-heavy reference data — never for volatile/compliance-critical current state |

**Scenario**
> A `GET /advisors?jurisdiction=NY` endpoint returning 500 advisors, each with its disclosure list rendered in the response, silently fired 501 queries. Detected via SQL debug logging showing 500 near-identical `SELECT ... FROM disclosure WHERE advisor_id = ?` queries. Fixed with a DTO projection (`AdvisorSummary(crdNumber, name, disclosureCount)`) for the list endpoint, reserving full entity + `JOIN FETCH` only for the single-advisor detail endpoint where the full disclosure list is actually needed.

## 3.2 Flyway / Liquibase — Schema Migrations

**Basic**: version-controlled, incrementally-applied SQL (Flyway) or XML/YAML/SQL changesets (Liquibase) — the schema's history becomes an auditable, replayable sequence rather than manually-run, undocumented DBA scripts.

**Intermediate**: Flyway — plain, numbered SQL files (`V1__init.sql`, `V2__add_column.sql`), applied in order, checksummed (a modified already-applied migration file is detected and fails the build — protecting against silent drift). Liquibase — changeset-based, supports **rollback** definitions per changeset and database-agnostic abstractions (useful if the same schema must support multiple DB engines, e.g., PostgreSQL in prod and H2 in tests).

**Advanced**:
- **Migrations in CI/CD**: run automatically as a pipeline stage before the application deploys, against the target environment — with a **read-only dry-run/validate step** in earlier environments before the same migration is promoted to production, exactly mirroring the "same artifact promoted through environments" principle from the Helm/CI discussion in the earlier docs.
- **Backward-compatible migration strategy for zero-downtime deploys**: a genuinely advanced, frequently-tested topic — adding a column must be **nullable or have a default** so the *old* application version (still running during a rolling deploy) doesn't break on an unexpected required column; renaming a column is done as **add-new, dual-write, backfill, remove-old** across multiple deploys, never as a single atomic rename, specifically because a rolling Kubernetes deployment means old and new application versions run **simultaneously** against the same database for a window of time.
- **Out-of-band schema changes** (a DBA manually altering a table outside the migration tool) break the tool's checksum/history tracking — Flyway's `baseline`/repair commands exist specifically to recover from this, worth knowing exists even if rarely needed.

**Trade-off**

| Tool | Strength | Consideration |
|---|---|---|
| Flyway | Simple, plain SQL, widely understood | Less built-in rollback tooling (rollback is typically a new forward migration) |
| Liquibase | Native rollback definitions, DB-agnostic changesets | More abstraction/tooling overhead for simple SQL-only shops |

**Scenario**
> A migration added a `NOT NULL` column with no default during a rolling deployment. Old pods (previous version) were still inserting rows without that column populated mid-rollout, and every one of those inserts failed until the rollout finished. Fix, going forward: **every** migration adding a required column ships as two steps across two deploys — first nullable-with-backfill, then a later migration adding the `NOT NULL` constraint only after all pods are confirmed on the new version.

## 3.3 NoSQL — MongoDB & Cassandra

**Basic**: MongoDB — document store, flexible schema, general-purpose. Cassandra — wide-column store, partition-key-centric, built for very high write throughput and horizontal scale.

**Intermediate**:
- MongoDB: **when to reach for it** — semi-structured/evolving-shape data (varying disclosure-event detail fields per type), or where PostgreSQL's native `JSONB` column type doesn't cleanly fit the access pattern needed.
- Cassandra: **partition key design is the single most important decision** — all efficient queries filter by partition key (or partition key + clustering columns); a poorly chosen partition key (e.g., one with very uneven data distribution, a "hot partition") creates a scaling bottleneck no amount of adding nodes fixes, because that one partition still lives on a bounded set of replica nodes.

**Advanced**:
- MongoDB: multi-document **ACID transactions** (matured since 4.0) — narrows the historical "NoSQL means no transactions" gap; **time-series collections** (native since 5.0) purpose-built for timestamped data, a direct modern alternative to reaching for Cassandra purely for that access pattern.
- Cassandra: **tunable consistency** per query (`QUORUM`, `ONE`, `ALL` on both read and write) — the classic trade-off `R + W > N` (replication factor) gives strong consistency; lower consistency levels trade correctness guarantees for lower latency/higher availability during partial outages — a direct, explicit CAP-theorem dial the application controls per-query, not a fixed system-wide setting.
- Cassandra: **Storage-Attached Indexes (SAI, Cassandra 5)** — meaningfully soften the historical "can basically only query by partition key" limitation, enabling more flexible secondary filtering without the older secondary-index model's severe performance cliffs.

**Trade-off**

| Choice | Use when | Avoid when |
|---|---|---|
| PostgreSQL JSONB | Mostly-relational data with a semi-structured subset | The *entire* dataset's shape varies significantly per record type |
| MongoDB | Flexible schema, document-shaped domain, need ACID within a document (and increasingly across a few) | Extremely high-throughput, simple key-based write workload (Cassandra's sweet spot) |
| Cassandra | Very high write throughput, time-series/append-heavy, access pattern is genuinely partition-key-driven | Need ad-hoc relational joins or flexible query patterns beyond the partition key (even with SAI, still a real constraint) |

**Scenario**
> A timecard-punch ingestion system needed to sustain very high write volume from a large workforce with a simple "write and later aggregate" access pattern. Cassandra, partitioned by `(employee_id, date_bucket)`, absorbed the write load with predictable latency; the trade-off accepted was no ad-hoc relational querying against the raw punch data — reporting/aggregation ran through a separate downstream pipeline instead, which was an acceptable split given the access pattern was never "ad-hoc joins," only "append, then batch-aggregate."

---

# DOMAIN 4 — TESTING

## 4.1 JUnit 5

**Basic**: `@Test`, assertions (`assertEquals`, `assertThrows`), lifecycle annotations (`@BeforeEach`/`@AfterEach`).

**Intermediate**: `@ParameterizedTest` (`@ValueSource`, `@CsvSource`, `@MethodSource`) for a visible matrix of edge cases in one test method rather than N near-duplicate methods; `@Nested` classes for organizing related test groups with shared setup; `@Tag` for selectively running test subsets (e.g., excluding slow integration tests from a fast local dev loop).

**Advanced**: the **extension model** (`@ExtendWith`) — composable, multiple extensions per test class (combine `MockitoExtension` with a Spring context extension, or a custom `Testcontainers`-lifecycle extension) — replacing JUnit 4's rigid single-`@RunWith` model. **JUnit 5's Vintage engine** enables running old JUnit 4 tests unmodified on the JUnit 5 platform — the mechanism that makes **incremental migration** possible instead of a risky big-bang test-suite rewrite.

## 4.2 Mockito

**Basic**: `@Mock` (a fake with no real behavior, returns defaults unless stubbed), `when(...).thenReturn(...)` (stubbing), `verify(...)` (interaction verification).

**Intermediate**: `@InjectMocks` (constructor/field injection of mocks into the class under test); `ArgumentCaptor` (capturing and asserting on the actual argument a mocked method was called with, not just that it was called); `@Spy` (a real object with selectively-overridden behavior — useful when you want most of the real logic but need to stub one specific method).

**Advanced**: **test double taxonomy precision** — a *stub* returns canned data; a *mock* additionally supports interaction verification; a *fake* is a lightweight working implementation (e.g., an in-memory repository) preferable when Mockito-style interaction verification gets unwieldy for complex, stateful collaborators. **Mockito 5's inline mock maker** (default) mocks final classes/methods (including implicitly-final records) via bytecode manipulation rather than subclass-based proxying, which is *why* mocking a record or a `final` utility method now works without extra configuration.

**Trade-off**: over-reliance on `verify()` for every interaction produces brittle tests coupled to *implementation* rather than *behavior* — the mature guideline is to verify interactions that represent genuine side effects/contracts (e.g., "the repository's save was called"), and prefer state-based assertions (checking the returned/resulting value) wherever behavior can be observed that way instead.

## 4.3 Testcontainers

**Basic**: spins up real, ephemeral Docker containers (Postgres, Kafka, Redis, LocalStack for AWS) for integration tests — testing against the **real** technology instead of a mock or an in-memory substitute (like H2 standing in for Postgres, which can silently diverge in SQL dialect behavior).

**Intermediate**:
```java
@Testcontainers
class FeedRepositoryIT {
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16");

    @DynamicPropertySource
    static void props(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
    }
    // tests run against a real, throwaway Postgres instance
}
```

**Advanced**: **container reuse across test classes** (`.withReuse(true)` + a Testcontainers config flag) to avoid the startup-cost penalty of spinning up a fresh container per test class in a large suite — a real, deliberate trade-off between test isolation purity and CI runtime; **`Network`** for multi-container test scenarios (a service container talking to a Kafka container over a shared Testcontainers network, testing genuine cross-container integration); placing Testcontainers-based tests specifically in the **integration-test stage** of the pipeline (not the fast unit-test stage), consistent with the test-pyramid principle — slower but far higher-fidelity than mocking the persistence/messaging layer entirely, catching real schema/serialization/query issues a pure-mock suite would miss.

**Trade-off**

| Approach | Fidelity | Speed |
|---|---|---|
| Mocked repository | Low (assumes the mock's behavior matches reality) | Fastest |
| In-memory DB (H2) | Medium (can diverge from real DB dialect/behavior) | Fast |
| Testcontainers (real DB/broker in Docker) | High (genuine behavior) | Slower — reserved for integration-test stage |

## 4.4 TDD

**Basic**: red (write a failing test for the next small behavior) → green (write the minimum code to pass) → refactor (clean up with the safety net of a passing test).

**Intermediate**: TDD works best when applied at the **unit level for genuinely non-trivial logic** (validation rules, state-transition logic, calculation logic) — the discipline of writing the test first forces precise thinking about the actual contract/edge cases *before* the implementation biases what you consider "correct."

**Advanced**: the honest, senior-level nuance — TDD is a **design technique first, a testing technique second**; writing the test first for a poorly-understood or exploratory piece of design can lead to over-fitting tests to an implementation that hasn't found its right shape yet. A pragmatic middle ground many senior engineers actually practice: spike/explore first for genuinely uncertain design, then apply strict TDD once the shape of the solution is clear, rather than treating "always write the test first, no exceptions" as dogma.

**Scenario**
> Writing the feed-record validation logic (correction vs. duplicate vs. new-record branching) test-first forced explicit enumeration of every edge case (same key/same hash, same key/different hash, new key) **before** the implementation existed — surfacing the "what happens if the natural key matches but the hash doesn't" question as a design decision made deliberately, rather than as a bug found later because the implementation was written first and the edge case simply wasn't considered.

---

# DOMAIN 5 — DEVOPS & CLOUD

## 5.1 CI/CD — Jenkins & GitHub Actions

**Basic**: pipeline-as-code (`Jenkinsfile` / `.github/workflows/*.yml`) — build → test → package → deploy, triggered on push/PR.

**Intermediate**: **quality gates as blocking pipeline stages** (SonarQube coverage/quality threshold, security scan) — a build **fails automatically** below a bar rather than relying on manual review discipline; **shared/reusable pipeline logic** (Jenkins shared libraries; GitHub Actions **reusable workflows**/composite actions) so a platform-wide pipeline change (e.g., adding a new scan stage) is a one-place edit propagating to every consuming repo.

**Advanced**: **environment promotion discipline** — the *same* built artifact (Docker image/Helm chart, tagged by commit SHA) is promoted dev → qa → prod, never rebuilt per environment, which is the only way to be certain "what passed qa is what's running in prod"; **gated production deploys** (manual approval + change-ticket linkage) as a deliberate, regulated-environment-appropriate friction point; **auto-rollback** wired to a post-deploy smoke-test failure, rather than relying on a human noticing and manually rolling back.

**Trade-off**

| Choice | Jenkins | GitHub Actions |
|---|---|---|
| Control | Full self-hosted control, huge plugin ecosystem | Managed, less operational overhead |
| Fit | Regulated environments wanting full infra control | Teams already fully on GitHub, wanting less CI infra to maintain |
| Cost dimension | Self-hosted runner/infra cost + maintenance burden | Usage-based hosted runner cost, less maintenance |

## 5.2 Docker

**Basic**: image = layered filesystem + metadata; container = a running instance of an image. `Dockerfile` defines the build.

**Intermediate**: **multi-stage builds** — build stage with the full JDK/build tools, final stage copies only the built artifact onto a minimal JRE base image, shrinking final image size and attack surface:
```dockerfile
FROM eclipse-temurin:21-jdk AS build
WORKDIR /app
COPY . .
RUN ./mvnw clean package -DskipTests

FROM eclipse-temurin:21-jre-alpine
COPY --from=build /app/target/app.jar /app.jar
USER 1000:1000
ENTRYPOINT ["java", "-XX:MaxRAMPercentage=75.0", "-jar", "/app.jar"]
```

**Advanced**: **layer caching strategy** — ordering `Dockerfile` instructions so rarely-changing layers (dependency resolution) come **before** frequently-changing layers (application source copy), so a source-only change doesn't invalidate the (slow) dependency-download layer cache on every build; **BuildKit** as the modern default builder (parallel stage execution, better cache export/import for CI); **non-root user by default** (security-scanning requirement, and a real defense-in-depth measure — a container-escape vulnerability is meaningfully less dangerous from a non-root process).

## 5.3 Kubernetes

**Basic**: Pod (smallest deployable unit) → Deployment (manages ReplicaSets/rolling updates) → Service (stable network identity/load balancing across pods) → Ingress (external HTTP routing).

**Intermediate**: **liveness vs. readiness probes** (Section 2.2's distinction, enforced here); **resource `requests`/`limits`** — requests drive scheduling (which node has room), limits enforce a hard ceiling (a container exceeding its memory limit is OOMKilled); **ConfigMaps/Secrets** for externalized configuration (12-Factor App principle).

**Advanced**:
- **Horizontal Pod Autoscaler (HPA)** — basic: scale on CPU/memory; advanced: scale on **custom metrics** (Kafka consumer lag, queue depth) via a metrics adapter — directly relevant to scaling event-consumer pods based on actual backlog rather than a CPU proxy that may not correlate well with I/O-bound consumer workload.
- **PodDisruptionBudget (PDB)** — guarantees a minimum number of pods stay available during voluntary disruptions (node drains, cluster upgrades) — critical for tier-1 services to avoid an entire service's replica set being drained simultaneously during routine cluster maintenance.
- **readiness gates + rolling update strategy tuning** (`maxUnavailable`/`maxSurge`) — controls how aggressively old pods are replaced by new ones during a deploy, trading deploy speed against the risk of too many pods being simultaneously unready.

**Trade-off**

| Decision | Gain | Cost |
|---|---|---|
| Aggressive HPA scale-down | Cost savings during low traffic | Risk of scaling too aggressively into a traffic spike (cold-start latency, Section 1.2's JVM startup discussion) |
| Strict resource limits | Predictable, fair multi-tenant node usage | A legitimately spiky service gets OOMKilled/throttled if limits are set too tight |
| Custom-metric HPA | Scales on the metric that actually matters (queue depth) | More setup complexity (metrics adapter, correct metric selection) than default CPU-based scaling |

## 5.4 AWS

**Basic**: EC2 (VMs), S3 (object storage), RDS (managed relational DB), Lambda (serverless functions), SQS/SNS (queue/pub-sub).

**Intermediate**: **EC2 vs Lambda decision** — EC2/EKS for steady, long-running, connection-pool-benefiting workloads; Lambda for event-triggered, short-duration, bursty workloads where paying for idle capacity is wasteful — and knowing Lambda's execution-time limits and cold-start characteristics matter for *why* a long batch job doesn't fit there.

**Advanced**:
- **IAM least-privilege**: scoping roles/policies to the minimum required actions/resources (a Lambda's execution role should be able to read *one specific* S3 bucket/prefix, not `s3:*` on `*`) — the cloud-native equivalent of the Kubernetes RBAC/NetworkPolicy least-privilege discussion from the platform-architecture doc.
- **VPC design** — public vs private subnets, NAT gateways for outbound-only internet access from private resources, security groups (stateful, instance-level) vs. NACLs (stateless, subnet-level) — a genuinely common "explain the difference" interview question, worth having crisp: security groups evaluate **all rules** and allow if *any* rule matches (implicit deny otherwise); NACLs evaluate rules **in numbered order** and stop at the first match, and require explicit rules for both directions since they're stateless.
- **AWS SDK v2**: async-first clients (`S3AsyncClient`), modular dependencies, immutable request/response objects — pairs naturally with virtual threads/reactive patterns (Section 1.2) for high-throughput S3/SQS interaction without tying up threads on I/O wait.

**Trade-off**

| Choice | Use when | Avoid when |
|---|---|---|
| EC2/EKS | Steady load, needs persistent connections/warm state | Bursty, infrequent, short-duration triggers |
| Lambda | Event-driven, short execution, want zero idle cost | Long-running batch jobs (execution time limits), workloads needing a warm connection pool |
| RDS | Relational, transactional, joins-heavy data | Extremely high write throughput better suited to a purpose-built NoSQL store |

**Scenario**
> A Lambda function's IAM role was initially granted `s3:*` on `*` "to get it working quickly." A security review flagged it — the fix scoped the role to `s3:GetObject`/`s3:PutObject` on exactly the one bucket/prefix the function actually used, closing a real, avoidable blast-radius risk (a compromised function credential could otherwise touch every S3 bucket in the account) for effectively zero functional cost.

---

# CROSS-DOMAIN INTERVIEW CHEAT SHEET

For any topic in this document, structure your answer:
1. **Basic mechanic** — one or two sentences, precise.
2. **The option space** — name at least two alternatives and when each wins (the trade-off table format above).
3. **A concrete scenario** — something that went wrong or had to be decided, and why the choice was made.
4. **The generalizable principle** — connect it to a broader theme (fail-closed vs fail-open, blast-radius containment, defense in depth, single source of truth) so the answer reads as transferable judgment, not memorized trivia.

This four-beat structure works for every row in every trade-off table above — practice producing it live for at least one topic per domain before the interview.
