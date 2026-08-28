# Spring Boot Interview Questions — Basic → Tough
### With Business Scenarios, Code Examples, and Tricky EM / PSE Questions

---

## HOW TO USE THIS GUIDE

Questions are grouped by difficulty, then by audience (Engineering Manager vs. Principal/Senior Software Engineer). For every non-trivial question, answer in this order out loud: **the mechanic → a code/config example → a business scenario where it actually mattered → the trade-off or gotcha**. That structure is what turns a textbook answer into an experienced one.

---

# PART 1 — BASIC

**Q1. What's the actual difference between the Spring Framework and Spring Boot?**
Spring Framework is the underlying IoC/DI container and ecosystem (Core, MVC, Data, Security, etc.) — powerful but historically required substantial manual XML/Java configuration to wire together. Spring Boot is an **opinionated layer on top** that auto-configures sensible defaults, bundles an embedded server, and packages dependencies via "starters" — the goal being a production-ready app with minimal boilerplate.
> **Business scenario**: A startup needs to ship an MVP API in two weeks. Spring Boot's auto-configuration and embedded Tomcat mean a working, deployable REST service exists on day one, versus days spent hand-wiring `DispatcherServlet`, view resolvers, and a `web.xml` in plain Spring MVC.

**Q2. What is auto-configuration and how does it decide what to configure?**
`@EnableAutoConfiguration` (bundled inside `@SpringBootApplication`) scans a list of candidate configuration classes (declared in each starter jar's `META-INF/spring/...AutoConfiguration.imports`), each guarded by `@Conditional` annotations — `@ConditionalOnClass` (only if a specific class is on the classpath), `@ConditionalOnMissingBean` (only if you haven't defined your own bean of that type), `@ConditionalOnProperty`. This is also **how you override any default** — just declare your own `@Bean`.
```java
@Bean
public ObjectMapper objectMapper() {
    return new ObjectMapper().registerModule(new JavaTimeModule());
}
// Spring Boot's default ObjectMapper auto-configuration backs off automatically
// because of @ConditionalOnMissingBean.
```

**Q3. What are Spring Boot "starters"?**
Curated dependency bundles (`spring-boot-starter-web`, `spring-boot-starter-data-jpa`) that pull in a tested-together set of libraries and trigger the relevant auto-configuration, so you declare intent ("I want a web app") rather than hand-picking a dozen individual dependency versions.

**Q4. `@Component` vs `@Service` vs `@Repository` vs `@Controller` — functionally different, or just naming?**
All four are `@Component` specializations and are functionally interchangeable for basic bean registration — the difference is **semantic clarity plus a few real behavioral extras**: `@Repository` adds automatic translation of persistence-technology-specific exceptions into Spring's `DataAccessException` hierarchy (so a DAO doesn't leak a raw `SQLException` up to a service layer that shouldn't know or care which database driver threw it); `@Controller`/`@RestController` trigger Spring MVC's request-mapping machinery. `@Service` has no extra framework behavior — it's purely semantic, signaling "business logic lives here."
> **Business scenario**: A team wraps a DAO's Hibernate calls in `@Repository`, then swaps the underlying JPA provider during a migration — application code that catches `DataAccessException` doesn't need to change at all, because the vendor-specific exception got translated away at the `@Repository` boundary.

**Q5. What is dependency injection, and why constructor injection over field injection?**
DI means a bean declares what it needs and the container supplies it, rather than the bean constructing its own collaborators. Constructor injection is preferred because: dependencies are explicit in the signature, the object is immutable (`final` fields) once constructed, the container **fails fast at startup** if a required dependency is missing, and it's trivially unit-testable without a Spring context (`new MyService(mockDep)`).
```java
@Service
public class OrderService {
    private final PaymentGateway paymentGateway; // final, constructor-injected

    public OrderService(PaymentGateway paymentGateway) {
        this.paymentGateway = paymentGateway;
    }
}
```

**Q6. What does `@SpringBootApplication` actually expand to?**
`@Configuration` + `@ComponentScan` + `@EnableAutoConfiguration` — a single annotation is convenience, not magic; every piece is a real, separately-usable Spring annotation.

**Q7. What's the embedded server, and why does it matter operationally?**
Tomcat (default), Jetty, or Undertow bundled inside the application jar rather than deployed as a WAR to an external server — the app *is* the deployable unit (`java -jar app.jar`), which is what makes containerization (Docker) and cloud-native deployment straightforward without a separately-managed app server.

**Q8. What is Spring Boot Actuator?**
Production-readiness endpoints (`/actuator/health`, `/actuator/metrics`, `/actuator/env`) exposed with minimal setup — the mechanism Kubernetes liveness/readiness probes and monitoring tools (Prometheus, Datadog) hook into.
> **Business scenario**: Without Actuator's `/health` endpoint, Kubernetes has no reliable signal to know a pod is actually ready to receive traffic after a deploy — it would either route traffic to a still-initializing pod (errors) or never route at all (false negative).

---

# PART 2 — INTERMEDIATE

**Q9. Walk through the bean lifecycle.**
Bean definition registered → instantiation (constructor called) → dependency injection → `@PostConstruct` callback → bean ready for use → (on shutdown) `@PreDestroy` callback. Default scope is **singleton** — one instance per container, instantiated eagerly at startup in dependency order (which is *why* a missing bean fails loudly at startup, not at first request).

**Q10. How do Spring profiles work, and when would you use them?**
`@Profile("prod")` on a bean, or `application-{profile}.yml` files, activated via `spring.profiles.active`. Lets the same codebase behave differently per environment (a mock payment gateway bean in `dev`, the real one in `prod`) without conditional code littering the business logic.
```java
@Profile("dev")
@Bean
public PaymentGateway mockPaymentGateway() { return new MockPaymentGateway(); }

@Profile("prod")
@Bean
public PaymentGateway stripePaymentGateway() { return new StripePaymentGateway(apiKey); }
```
> **Business scenario**: QA needs to run full checkout-flow tests without actually charging real cards — a `test` profile swaps in a stub payment gateway, letting the *exact same* order-processing code run against a fake integration.

**Q11. How does centralized exception handling work?**
`@RestControllerAdvice` + `@ExceptionHandler` maps exception types to HTTP responses in one place, instead of try/catch scattered through every controller.
```java
@RestControllerAdvice
public class ApiExceptionHandler {
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(ResourceNotFoundException ex) {
        return ResponseEntity.status(404).body(new ErrorResponse("NOT_FOUND", ex.getMessage()));
    }
}
```
> **Business scenario**: A customer support team keeps getting confused, inconsistent error payloads from different endpoints (one returns a stack trace, another a bare 500). Centralizing exception handling gives every API consumer the same predictable error envelope, directly reducing support escalations caused by ambiguous error responses.

**Q12. `@Transactional` — what does it actually do, and where does it break?**
It's implemented via **AOP proxying** — Spring wraps the bean in a proxy that opens a transaction before the method executes and commits/rolls back after. Two classic gotchas worth knowing cold:
- **Self-invocation doesn't trigger the proxy**: calling a `@Transactional` method from *another method in the same class* bypasses the proxy entirely (it's a direct `this.method()` call, not a call through the Spring-managed proxy), so no transaction is applied.
```java
@Service
public class OrderService {
    public void placeOrder() {
        saveOrder(); // NOT transactional -- this is a direct call, bypasses the proxy
    }
    @Transactional
    public void saveOrder() { ... }
}
```
- **Default rollback only on unchecked exceptions**: a checked exception thrown from a `@Transactional` method does **not** roll back the transaction by default — you must explicitly declare `@Transactional(rollbackFor = Exception.class)` if checked exceptions should also trigger rollback.
> **Business scenario**: A real production bug pattern — an order-processing method throws a checked `InventoryException` mid-transaction, and because the default rollback rule only covers unchecked exceptions, the order gets partially committed (payment charged, inventory not decremented) — a data-integrity incident traceable directly to this exact default.

**Q13. How does Bean Validation (`@Valid`) work end to end?**
`@Valid` on a controller method parameter triggers Bean Validation annotations (`@NotNull`, `@Size`, `@Email`) on the request DTO; a failure throws `MethodArgumentNotValidException`, caught centrally via `@ExceptionHandler` to produce a structured 400 response with field-level errors.

**Q14. What's the difference between `@SpringBootTest`, `@WebMvcTest`, and `@DataJpaTest`?**
`@SpringBootTest` loads the **full application context** (slow, but tests real integration); `@WebMvcTest` loads only the web layer (controllers, filters, `MockMvc`) with everything else mocked — fast, focused on request/response behavior; `@DataJpaTest` loads only the JPA layer against an embedded/test database — fast, focused on repository/query correctness. Choosing the narrowest slice that actually exercises what you're testing is the difference between a CI suite that runs in 2 minutes vs. 20.

**Q15. How does caching (`@Cacheable`) work, and what's the most common way it silently fails?**
`@Cacheable("orders")` caches a method's return value keyed by its arguments, backed by a configurable cache provider (Caffeine, Redis, EhCache). Most common silent failure: **the same self-invocation problem as `@Transactional`** — since caching is also AOP-proxy-based, calling a `@Cacheable` method from within the same class bypasses caching entirely, with no error or warning, just quietly worse performance than expected.

**Q16. When would you use `@Async`, and what has to be true for it to actually run asynchronously?**
`@Async` methods run on a separate thread pool rather than blocking the caller — useful for fire-and-forget work (sending a notification email after an order completes) that shouldn't hold up the response. Requirements: `@EnableAsync` on a configuration class, the method must be `public`, and — same proxy caveat again — **must be called from a different bean**, not self-invoked.
> **Business scenario**: A checkout endpoint was taking 800ms partly because it synchronously sent a confirmation email inline. Moving the email send to an `@Async` method dropped perceived checkout latency to under 200ms — the email still sends, just off the critical response path.

---

# PART 3 — TOUGH / ADVANCED

**Q17. Explain the full `ApplicationContext.refresh()` lifecycle and why it matters for troubleshooting.**
`invokeBeanFactoryPostProcessors()` (parses `@Configuration` classes, registers bean definitions) → `registerBeanPostProcessors()` (registers, but doesn't yet run, the processors that will later intercept bean *instances* — this is the hook AOP proxy creation and `@Autowired` processing use) → `finishBeanFactoryInitialization()` (singleton beans actually instantiated, in dependency order) → embedded server starts *last*, after every singleton is fully wired — meaning no request can ever hit a half-initialized bean.
> Why this matters practically: knowing bean instantiation happens in **dependency order, before the server starts**, is exactly what explains why a misconfigured required bean fails the whole app at startup rather than surfacing as an intermittent runtime error — a deliberate fail-fast design worth being able to explain when asked "why does my app crash on startup instead of on first use."

**Q18. How does Spring resolve (or fail to resolve) circular dependencies?**
For singleton beans using **setter/field injection**, Spring can resolve simple circular dependencies via early-reference caching (a bean is exposed as an "early" reference to satisfy a circular need before it's fully initialized). For **constructor injection**, Spring **cannot** resolve a true circular dependency — `BeanCurrentlyInCreationException` at startup — because a constructor requires the fully-built dependency to exist before the object itself can be constructed at all. This is actually a *feature*, not a limitation to work around: a circular dependency between two services is almost always a sign of a genuine design problem (unclear ownership/responsibility boundary), and constructor injection forcing it to fail loudly is better than setter injection silently papering over bad design with a partially-initialized bean.
> **EM-relevant framing**: if a team hits this error and "fixes" it by switching to field/setter injection, that's treating a symptom, not the design smell — the better fix is usually extracting the shared logic into a third bean both original services depend on, breaking the cycle structurally.

**Q19. What's the actual limitation of proxy-based AOP (`@Transactional`, `@Cacheable`, `@Async`), and how does it differ between JDK dynamic proxies and CGLIB?**
Spring's default proxying for classes **implementing an interface** uses **JDK dynamic proxies** (interface-based, reflection-driven); for classes with no interface, Spring falls back to **CGLIB** (subclass-based bytecode generation). Both share the self-invocation limitation from Q12/Q15/Q16. CGLIB specifically **cannot proxy `final` classes or `final` methods** (can't subclass/override a final method) — a real, concrete gotcha: marking a `@Service` class or one of its `@Transactional` methods `final` silently breaks the proxy's ability to intercept it.

**Q20. Bean scopes — what happens when you inject a `prototype`-scoped bean into a `singleton`-scoped bean, naively?**
A singleton is instantiated once; if it holds a direct reference to a prototype bean injected at construction time, it gets **one instance of that prototype, forever** — defeating the entire purpose of prototype scope (a fresh instance per use). The fix: inject an `ObjectProvider<T>` (or `ObjectFactory<T>`/a scoped proxy) instead of the bean directly, so a fresh prototype instance is fetched **on each actual use**, not once at singleton construction time.
```java
@Service
public class ReportGenerator {
    private final ObjectProvider<ReportBuilder> builderProvider;
    public ReportGenerator(ObjectProvider<ReportBuilder> builderProvider) {
        this.builderProvider = builderProvider;
    }
    public Report generate() {
        ReportBuilder builder = builderProvider.getObject(); // fresh instance every call
        return builder.build();
    }
}
```

**Q21. How would you design idempotent REST endpoints in Spring Boot for a payment-adjacent API?**
An `Idempotency-Key` header, checked against a fast store (Redis) before executing business logic — if the key was already processed successfully, return the **stored original response** rather than re-executing; if not seen, execute and store the result keyed by that value with a bounded TTL. Critical because HTTP's own semantics don't protect you here: **POST is neither safe nor idempotent** by spec, so a client's network-retry logic resending an identical POST is fully spec-compliant behavior your API must defend against, not an edge case to dismiss.
> **Business scenario**: A flaky mobile network causes a payment-confirmation POST to be silently retried by the client. Without an idempotency key, this double-charges the customer — a directly quantifiable, real financial-harm bug, not a theoretical one.

**Q22. How do you handle safe, zero-downtime database migrations in a Spring Boot service using Flyway/Liquibase under a rolling deployment?**
Because a rolling Kubernetes deployment runs **old and new application versions simultaneously** against the same database for a window of time, a migration must be backward-compatible with the *previous* app version, not just forward-compatible with the new one. Adding a required column must ship as nullable-with-a-default first (old pods keep inserting rows without it, which is fine); a column rename is add-new → dual-write → backfill → remove-old across multiple deploys, never a single atomic rename.

**Q23. How would you diagnose a production connection-pool-exhaustion incident in a Spring Boot service using HikariCP?**
Check Actuator's `/actuator/metrics/hikaricp.connections.active` vs. `hikaricp.connections.max` trend leading up to the incident; correlate with thread dumps (`jstack`) showing many threads blocked waiting to acquire a connection; the actual root cause is almost always one of: a slow/leaking query holding connections far longer than expected (missing a `LIMIT`, an N+1 storm), a pool sized smaller than the actual concurrent-request load, or a connection leak (a code path that acquires a connection/transaction but doesn't release it on an exception path). The fix targets the *specific* cause — resizing the pool blindly without finding the actual leak just delays the next occurrence.

---

# PART 4 — BUSINESS-SCENARIO-DRIVEN QUESTIONS

**Q24. "We're running a Black Friday flash sale and expect 50x normal traffic for 2 hours. Walk me through what you'd check/harden in a Spring Boot checkout service beforehand."**
A strong answer touches: connection pool sizing relative to expected concurrency (and the corresponding DB-side max connections), caching read-heavy, slow-changing data (product catalog) via `@Cacheable`/Redis to take load off the DB, rate limiting to protect against both legitimate traffic spikes and bot/scalper abuse, circuit breakers (Resilience4j) around any downstream dependency (inventory, payment) so a struggling dependency degrades gracefully instead of cascading, load-testing beforehand against realistic concurrency (not just functional testing), and — a detail many miss — **JVM warm-up/startup time** if autoscaling adds new pods mid-spike (Section on virtual threads/native image from the Java-ecosystem material is directly relevant here).

**Q25. "A multi-tenant SaaS product needs each customer's data isolated. How would you architect this in Spring Boot?"**
Trade-off framing, not one "correct" answer: **schema-per-tenant** (strong isolation, more operational overhead managing N schemas/migrations) vs. **shared schema with a tenant-ID discriminator column** (simpler operationally, requires disciplined enforcement — e.g., a Hibernate filter or a base repository that *always* scopes by tenant ID, since a forgotten `WHERE tenant_id = ?` is a serious cross-tenant data leak) vs. **database-per-tenant** (strongest isolation, highest operational cost, often reserved for enterprise/compliance-sensitive tenants). Spring's `AbstractRoutingDataSource` is the mechanism for dynamically routing to the right schema/database per request based on the resolved tenant context.

**Q26. "Our compliance team needs a full audit trail of every change to a sensitive entity — who changed what, when. How do you implement this without littering every service method with manual logging?"**
Spring Data's auditing support (`@CreatedDate`, `@LastModifiedDate`, `@CreatedBy`, `@LastModifiedBy` + `@EnableJpaAuditing`) handles the common case declaratively. For a genuine full audit *history* (not just last-modified metadata), Hibernate Envers (tracks every revision of an entity) or an event-sourcing-style approach (publish a domain event on every state change, persisted to an append-only audit log/Kafka topic) are the two standard patterns — Envers for "give me the full history of this specific entity," event sourcing for "the audit log itself is a first-class, independently useful data stream," a trade-off worth stating explicitly when asked which you'd pick.

**Q27. "A legacy monolith is being split into microservices. One critical business transaction currently spans three domains that will become three separate services. How do you preserve transactional integrity?"**
This is the classic **distributed transaction** problem, and the honest answer starts with: **you generally can't have the same ACID guarantee across service boundaries that you had inside one database transaction** — reaching for a naive two-phase-commit-style solution across microservices is a well-known anti-pattern (poor availability, doesn't scale). The standard pattern is the **Saga** — a sequence of local transactions, each publishing an event that triggers the next step, with explicit **compensating actions** defined for each step to undo its effect if a later step fails (choreography via events, or orchestration via a dedicated saga coordinator). This is also exactly where **idempotent consumers** (Q21's pattern, applied to async messages instead of REST) become mandatory, since most messaging systems guarantee at-least-once delivery.

---

# PART 5 — TRICKY QUESTIONS FROM AN ENGINEERING MANAGER'S LENS

These probe judgment, trade-off ownership, and how you'd lead a team through a decision — not just technical correctness.

**Q28. "A team member wants to switch the whole platform from Spring MVC to WebFlux because 'reactive is faster.' How do you evaluate that proposal?"**
A good answer resists the binary framing immediately: WebFlux's non-blocking model genuinely helps for high-concurrency, I/O-bound workloads, but it comes with a steeper learning curve, harder debugging (stack traces across reactive chains are notoriously unfriendly), and — critically, post-Java-21 — **virtual threads have narrowed the practical gap** for the *majority* of standard CRUD/request-response services, letting existing blocking MVC code scale similarly with far less complexity (`spring.threads.virtual.enabled=true`). The EM-level answer: ask for the actual bottleneck data first (is the current service genuinely thread-pool-starved under real load, or is this solving a problem that doesn't exist yet), and treat "rewrite the whole platform" as a much higher bar than "let's pilot virtual threads on the specific service showing the actual symptom."

**Q29. "You inherit a team where every service method is wrapped in `@Transactional` 'just to be safe,' regardless of whether it actually writes anything. How do you address this?"**
This is a real, common anti-pattern worth having a direct answer for: blanket `@Transactional` on read-only methods holds a database connection/transaction open unnecessarily (real cost under load — connection pool pressure), and worse, it signals the team doesn't actually understand *why* they're using it, which is a bigger risk than the immediate performance cost (Q12's self-invocation and checked-exception rollback gotchas are exactly the kind of subtle bug this team is unlikely to have considered). The management response isn't just "fix the code" — it's recognizing a **knowledge gap** and addressing it with a lunch-and-learn or paired code-review pass explaining the proxy mechanics, so the fix sticks instead of becoming a one-time cleanup that regresses next sprint.

**Q30. "How would you make the business case for upgrading 40 microservices from Spring Boot 2 to Spring Boot 3?"**
A strong EM answer doesn't lead with "it's the newer version" — it leads with concrete, business-relevant costs of *not* upgrading: Spring Boot 2.x is approaching/past end of active support (security patch risk), the JDK-17-floor requirement is a forcing function that also unlocks virtual threads (a real performance/cost lever, tie to Q28), and the migration cost itself is mostly mechanical (`javax`→`jakarta` namespace, tooling like OpenRewrite can automate much of it) rather than a risky rewrite — sequencing the case as **risk of inaction + a genuine capability unlock + a bounded, mostly-automatable cost** is what gets budget approved, versus "let's modernize" as a vague aspiration.

**Q31. "One of your senior engineers insists on hand-rolling a custom Spring Boot starter for internal use across teams instead of just sharing a utility library. Good idea or scope creep?"**
Genuinely depends, and a good EM probes rather than immediately vetoing: a custom starter is worth the investment when the shared capability needs **auto-configuration behavior** (not just shared code — e.g., automatically wiring a properly-configured `RestTemplate`/`WebClient` with the org's standard retry/timeout/tracing defaults the moment the starter's on the classpath), and when enough teams will adopt it to amortize the maintenance cost of a versioned, org-owned artifact. If it's just shared utility *code* with no auto-configuration behavior needed, a plain library is simpler and lower-maintenance — the EM's job here is making sure the decision is driven by actual reuse-and-adoption math, not "starters are the cool Spring pattern."

**Q32. "Your team ships a Spring Boot service that OOMKills in production despite passing all load tests. What's your process, not just your guess at the cause?"**
This is testing process discipline, not a trivia answer. Good shape: confirm the load test environment actually matched production's **container resource limits** (a very common gap — Section on JVM container-awareness from earlier material: a load test run on an unconstrained laptop/CI runner won't surface a heap-sizing-vs-container-limit mismatch); pull a heap dump (`-XX:+HeapDumpOnOutOfMemoryError`) from the actual incident, not a reproduction guess; distinguish "genuinely too little memory for real traffic" from "a real leak" by checking whether old-gen occupancy trends upward and never comes back down after GC (leak signature) vs. simply being undersized. The EM-level point: **the process of diagnosing precisely, not the fastest plausible-sounding fix**, is what prevents a recurring, expensive incident pattern.

---

# PART 6 — TRICKY QUESTIONS FROM A PRINCIPAL / SENIOR SOFTWARE ENGINEER'S LENS

These probe deep mechanics, system design judgment, and the ability to defend a specific technical trade-off under scrutiny.

**Q33. "Design a distributed rate limiter for a Spring Boot API gateway, shared across multiple service instances."**
A per-instance in-memory rate limiter (e.g., Guava `RateLimiter`) doesn't work correctly once you have N instances behind a load balancer — each instance enforces its own independent limit, so the *effective* aggregate limit becomes N× the intended one. The correct design centralizes state in **Redis**, using an atomic operation (`INCR` + `EXPIRE`, or a sliding-window/token-bucket algorithm implemented via a Lua script for atomicity across the check-and-increment) so every instance shares one true count. Worth being able to compare **fixed window** (simple, but allows a burst right at the window boundary — up to 2x the limit in a short span) vs. **sliding window log/counter** (more accurate, more storage/compute cost) vs. **token bucket** (naturally allows brief bursts while enforcing a steady-state rate) — and to state which you'd pick for a given API's actual traffic shape, not just name all three.

**Q34. "Explain exactly what happens, at the proxy level, when a `@Transactional` public method calls another `@Transactional` public method on a *different* bean, with `Propagation.REQUIRES_NEW` on the second."**
The outer method's proxy opens transaction A. The call crosses into a different bean, so it *does* go through that bean's own proxy (no self-invocation issue here, unlike Q12) — the proxy sees `REQUIRES_NEW`, **suspends** transaction A, opens a completely independent transaction B, commits or rolls back B independently of A's eventual outcome, then resumes A. This is the mechanism behind a common real pattern: writing an audit log entry that must persist **even if the main business transaction later rolls back** — putting the audit-log write in a `REQUIRES_NEW`-annotated method on a separate bean guarantees it survives independently.

**Q35. "You're asked to add distributed tracing across 20 Spring Boot microservices with minimal code changes. Walk through the actual mechanics of how a trace ID propagates."**
Micrometer Tracing (Spring Boot 3.x, replacing the older Sleuth) auto-instruments common entry/exit points (incoming HTTP requests, `RestTemplate`/`WebClient` calls, Kafka producers/consumers) to generate spans and propagate the trace context — concretely, an incoming request without an existing trace context gets a new trace ID generated; the same ID is injected into outbound headers (`traceparent`, W3C Trace Context standard) on any downstream call, and into Kafka message headers for async hops, so every service in the chain, without individual code changes beyond the dependency being on the classpath, contributes spans to the same trace. The genuinely hard part worth naming: propagation across **async boundaries not covered by auto-instrumentation** — a custom `ExecutorService`, a hand-rolled thread pool, or a message queue library without built-in Micrometer support requires manually propagating the trace context (or using Scoped Values/`ThreadLocal`-aware wrappers), or you get broken/orphaned traces at exactly those hops.

**Q36. "Critique this design: a Spring Boot service stores JWT refresh tokens in the database and checks them on every single API request, in addition to the short-lived access token."**
This is deliberately checking whether the candidate understands *why* stateless JWT auth exists at all: checking a refresh token on every request reintroduces a database round-trip per request, which defeats the entire scalability rationale for using JWTs (avoiding a shared session store lookup) — the access token's whole purpose is to be independently, statelessly verifiable via signature. The refresh token should only be consulted at the *token-refresh* endpoint, not on every protected-resource request. If the actual requirement is "we need the ability to instantly revoke access mid-session," the correct tool is a lightweight, fast **revocation blocklist check** (Redis, O(1) lookup) consulted only for that specific concern, kept separate from routine request authorization — a good answer distinguishes "what problem are we actually solving" from "let's just check everything against the database to be safe."

**Q37. "A team wants to migrate a Spring Boot monolith's synchronous, chained service-to-service REST calls (A calls B calls C) to an event-driven model. What do you actually need to prove works before calling the migration done?"**
Beyond "the events fire and get consumed," a rigorous answer requires: **idempotent consumers** at every hop (Q21/Q27's pattern — at-least-once delivery is the default assumption), a defined **ordering guarantee** where it actually matters (partition-key strategy if using Kafka, and an explicit decision about whether cross-entity ordering is even required), a **reconciliation mechanism** proving no events were silently lost (source-count vs. processed-count vs. dead-lettered-count balancing, echoing the reconciliation discipline from data-pipeline design), and **observable, traceable async flows** (Q35) so a support engineer debugging "why didn't C ever get updated" isn't reduced to manually correlating logs across three services and a message broker by timestamp.

**Q38. "How would you version a Spring Boot REST API to avoid breaking existing consumers, and what's the trade-off of each approach you'd mention?"**
URI versioning (`/v1/`, `/v2/`) — simplest, most cache-friendly, easiest for consumers to test manually, but "pollutes" the resource URI with a non-resource concept. Header/media-type versioning (`Accept: application/vnd.company.v2+json`) — more "RESTfully pure," but harder for consumers to test/debug casually and less visible in logs/traces. The advanced point worth adding unprompted: **whichever scheme you pick, the real discipline is an explicit deprecation policy** (a stated sunset timeline, `Deprecation`/`Sunset` HTTP headers) for the old version — versioning without a retirement plan just accumulates permanent parallel-maintenance cost indefinitely.

---

## CLOSING — THE ANSWER STRUCTURE THAT WORKS ACROSS ALL OF THESE

For any Spring Boot question, regardless of difficulty:
1. **State the mechanic precisely** (what actually happens, not just the annotation name).
2. **Show a minimal code example** if the question is mechanics-based.
3. **Tie it to a business scenario** — a real cost (money, downtime, compliance risk) the mechanic prevents or causes when misunderstood.
4. **Name the gotcha/trade-off explicitly** — this is the single highest-signal addition to any answer, because it's what separates "read the docs" from "operated this in production."
