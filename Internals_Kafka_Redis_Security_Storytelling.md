# Spring Boot Internals, Kafka, Redis, REST/JWT Security
### Basic → Advanced, Told as Production War Stories — Interview Edition

---

## HOW TO USE THIS GUIDE

Every section opens with a **short scene** — a realistic production moment — because that's how the best interview answers actually land: not "here's a definition," but "here's the 2 AM moment that only makes sense once you understand the internals, and here's what I did about it." Read the scene, then the mechanics that explain it, then the pattern/trade-off layer, then the "how to tell this story" close.

---

## 1. SPRING BOOT INTERNALS

### The scene
> "A new engineer adds `@Service` to a class, expects it to be injectable, and it isn't — Spring throws `NoSuchBeanDefinitionException`. They're staring at the class, it's annotated correctly, and they can't figure out why. Ten minutes later we find it: the class lives in a package **outside** the component-scan base package, because someone reorganized packages six months ago and the `@SpringBootApplication` class never moved to the new root. This is a perfect small window into how much Spring Boot is doing for you invisibly — and how confusing it gets the one time an assumption it's making silently doesn't hold."

### Basic — the ApplicationContext startup lifecycle
When `SpringApplication.run()` executes, roughly, in order:
1. **Environment preparation** — property sources resolved (application.yml, environment variables, command-line args, in a defined precedence order — command-line args win over env vars win over `application.yml`, worth knowing precisely because "why did my env var override not work" is a real, common debugging moment traceable directly to this precedence order).
2. **ApplicationContext creation** — for a web app, specifically a `ServletWebServerApplicationContext` (or `ReactiveWebServerApplicationContext` for WebFlux).
3. **`refresh()`** — this single method is the heart of the Spring container, and knowing its phases at an intermediate/advanced level is what separates "I use Spring" from "I understand Spring":
   - `invokeBeanFactoryPostProcessors()` — processors that can modify **bean definitions** before any bean is instantiated (e.g., `@Configuration` classes get parsed here, `@ComponentScan` packages get scanned, `@PropertySource` gets registered).
   - `registerBeanPostProcessors()` — registers (but doesn't yet run) processors that will later intercept **bean instances** (this is the extension point `@Autowired` processing, AOP proxy creation, and validation annotation processing all hook into).
   - `finishBeanFactoryInitialization()` — **this is where singleton beans are actually instantiated**, in **dependency order** (Spring figures out the dependency graph and instantiates leaf dependencies first) — this is *why* a missing/misconfigured dependency fails loudly at startup rather than silently at first use: eager singleton instantiation is a deliberate fail-fast design choice.
4. **Embedded server starts** (`WebServerStartStoppedLifecycle` / `ServletWebServerFactory`) — **after** the ApplicationContext is otherwise ready, meaning by the time your app actually starts accepting HTTP traffic, every singleton bean is already fully wired — no request can ever hit a half-initialized bean, another deliberate ordering guarantee worth naming.

### Intermediate — how auto-configuration actually decides what to configure
This is the mechanism behind the "magic" — worth being able to explain precisely:
- `@SpringBootApplication` = `@Configuration` + `@ComponentScan` + `@EnableAutoConfiguration`.
- `@EnableAutoConfiguration` triggers Spring Boot to read a file (`META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` in modern Spring Boot 3.x, formerly `spring.factories` in 2.x) listing every candidate auto-configuration class **bundled inside every starter jar on the classpath**.
- Each candidate auto-configuration class is guarded by **`@Conditional` annotations** — `@ConditionalOnClass` (only apply if a specific class is on the classpath — e.g., Kafka auto-config only activates if `KafkaTemplate` is present), `@ConditionalOnMissingBean` (only apply if the developer hasn't already defined their own bean of that type — **this is the exact mechanism that lets you override any auto-configured bean just by defining your own** — a very common, very practical interview question: "how do you override Spring Boot's default X" — answer: define your own `@Bean` of that type; `@ConditionalOnMissingBean` on the auto-configuration means Spring Boot politely backs off), `@ConditionalOnProperty` (only apply if a specific property is set to a specific value — this is how `spring.threads.virtual.enabled=true` from the version-evolution doc actually works under the hood).

**Production scenario**: "We once had a service silently NOT get its expected DataSource auto-configured, and it took a while to realize: we'd added an explicit `@Bean DataSource` for a test-scoped in-memory database in a `@TestConfiguration` class that was accidentally being picked up by component scanning in the main application context too — `@ConditionalOnMissingBean` did exactly what it was designed to do (back off because a bean already existed), just not what *we* wanted, because the 'already existing' bean was a test artifact leaking where it shouldn't have been. Understanding the conditional mechanism is what let us diagnose it fast instead of assuming Spring Boot was 'broken.'"

### Advanced — the request-handling path (DispatcherServlet) and where cross-cutting concerns actually hook in
```
HTTP request
   │
   ▼
Servlet Container (embedded Tomcat) accepts connection
   │
   ▼
Filter chain (javax/jakarta Servlet Filters — Spring Security's filter chain lives HERE)
   │   e.g., JwtAuthenticationFilter, CORS filter, request-logging filter
   ▼
DispatcherServlet.doDispatch()
   │
   ├─► HandlerMapping — resolves which @Controller method matches this URL/method
   ├─► HandlerInterceptor.preHandle() — Spring MVC-level cross-cutting hooks (lighter than filters, Spring-context-aware)
   ├─► HandlerAdapter invokes the controller method
   │        │
   │        ├─► @Valid triggers Bean Validation
   │        ├─► HttpMessageConverter deserializes the request body (JSON -> DTO, via Jackson)
   │        └─► Method executes, returns a DTO
   │
   ├─► HttpMessageConverter serializes the response DTO back to JSON
   └─► HandlerInterceptor.postHandle() / afterCompletion()
   │
   ▼
Filter chain unwinds (response headers, logging, etc.)
```
**The precise, interview-relevant distinction to draw**: **Servlet Filters** run outside Spring's context awareness (they operate at the raw HTTP request/response level, which is *why* Spring Security's filter chain has to be a Filter, not an Interceptor — authentication needs to happen before Spring even resolves which controller to invoke); **HandlerInterceptors** run inside Spring MVC's dispatch cycle and have access to the resolved handler method (useful for things like method-level `@PreAuthorize`-style logic, though that specific example is actually AOP-based, a third mechanism); **AOP proxies** (like `@Transactional`, `@PreAuthorize`, `@Retryable`) wrap the *bean method call itself*, one layer further in than either Filters or Interceptors. Knowing which of these three layers a given cross-cutting concern actually lives in is exactly the kind of precision a staff-level interviewer is listening for.

### Interview angle
> "Explain what happens between a request hitting your load balancer and your controller method executing" is a very standard system-design-adjacent Spring question — answer it by walking the diagram above, and land specifically on the Filter-vs-Interceptor-vs-AOP distinction unprompted, because almost nobody volunteers that distinction without being pushed for it, and volunteering it is a strong signal.

---

## 2. KAFKA INTERNALS

### The scene
> "It's a Tuesday morning and the on-call dashboard shows consumer lag climbing steadily on the compliance-event topic — not spiking, *climbing*, steadily, for twenty minutes. Naively you'd think 'the consumer is too slow, scale it up.' We scaled it up — lag got *worse* for the next two minutes before finally recovering. That's not what you'd expect from 'just add more consumers' — and it's a direct, visible symptom of how Kafka's consumer group rebalancing protocol actually works, which is worth understanding precisely rather than just knowing 'Kafka has consumer groups.'"

### Basic — the log, partitions, offsets
A Kafka topic is a **partitioned, append-only log**. Each partition is an ordered, immutable sequence of records, each identified by a monotonically increasing **offset**. Producers append; consumers track (via a committed offset, itself stored in an internal Kafka topic, `__consumer_offsets`) how far they've read. **Ordering is only guaranteed within a single partition**, not across a whole topic — this is the single most important basic fact, and it's *why* partition-key selection (Section 10 of the code-snippet companion — keying by CRD number) matters: you get ordering exactly for the set of records sharing a key, and no ordering guarantee across different keys, which is usually exactly the guarantee you actually need and no more.

### Intermediate — replication, ISR, and what `acks` actually controls
Each partition has one **leader** broker (all reads/writes for that partition go through it) and N **replica** brokers that copy the leader's log. The **In-Sync Replica (ISR) set** is the subset of replicas that are caught up closely enough to the leader to be considered safe to promote if the leader fails.

- `acks=0` — producer doesn't wait for any broker acknowledgment at all — fastest, and **can silently lose data** on a broker failure between send and replication.
- `acks=1` — producer waits for the **leader** to acknowledge — a leader crash immediately after acknowledging, before replicas caught up, can still lose the record.
- `acks=all` (equivalent to `acks=-1`) — producer waits for **all current ISR members** to acknowledge — combined with `min.insync.replicas` (e.g., set to 2 on a replication-factor-3 topic), this is what actually gives you a durability guarantee: a write is only acknowledged once it's safely on at least `min.insync.replicas` brokers, so losing any single broker (up to replication-factor minus min-insync) cannot lose acknowledged data.

**Production-relevant precision**: `min.insync.replicas=2` on a replication-factor-3 topic means the topic can tolerate **one** broker being down and keep accepting writes; if a **second** broker also goes down, producers using `acks=all` will start getting `NotEnoughReplicasException` — **the system deliberately refuses to accept writes it can't durably guarantee**, rather than silently degrading durability. This is a fail-closed-on-durability design choice, directly analogous to the mTLS STRICT trade-off from the architecture doc — worth explicitly drawing that parallel if it comes up.

### Advanced — producer batching internals
Producers don't send one record per network call — they **batch**, governed by two settings that trade off latency against throughput:
- `linger.ms` — how long the producer will *wait*, accumulating records into a batch, before sending, even if the batch isn't full yet (e.g., `linger.ms=5` — wait up to 5ms hoping more records arrive to batch together).
- `batch.size` — the max batch size (bytes) per partition; a batch sends immediately once full, without waiting for `linger.ms` to elapse.

**The trade-off to articulate precisely**: `linger.ms=0` (the default) minimizes per-record latency but forfeits batching efficiency under high throughput; a small positive `linger.ms` (5–20ms is typical) sacrifices a small, usually-imperceptible amount of per-record latency in exchange for meaningfully better throughput and lower broker-side overhead, because far fewer, larger network requests are sent. For a high-volume feed-ingestion producer (the MCO-style scenario), a small nonzero `linger.ms` is almost always the right call; for a genuinely latency-sensitive single-record-at-a-time producer, `linger.ms=0` is correct.

**Idempotent producer + transactions (exactly-once semantics)**: `enable.idempotence=true` (from the earlier code-snippet companion) prevents **producer-side retries** from creating duplicate records on the broker, by having the broker track a producer ID + sequence number per partition and deduplicate retries at that level — this solves duplicate-on-retry, but **not** the separate "consumer processes a record, then crashes before committing its offset, then reprocesses on restart" problem, which is what application-level idempotent-consumer design (the natural-key/hash-comparison pattern) is *actually* for. **Kafka transactions** (`transactional.id`, used for read-process-write patterns across topics) go further, atomically committing a consumer's offset *and* its produced output records together — worth knowing this exists as the "true exactly-once" mechanism, and worth being precise that it's a heavier, more specialized tool than the idempotent producer alone, typically reached for in stream-processing (Kafka Streams) topologies rather than simple produce/consume services.

### Advanced — consumer group rebalancing, and the scene from the top of this section
When a consumer group's membership changes (an instance joins, leaves, or is considered dead because it missed a heartbeat or exceeded `max.poll.interval.ms` between polls), the **group coordinator** (a designated broker) triggers a **rebalance** — reassigning partitions across the current group members.

- **Eager rebalancing** (the historical default): **every** consumer in the group **stops consuming entirely**, revokes all its partitions, a new assignment is computed, and consumption resumes — a "stop-the-world" pause affecting the *whole group*, even for consumers whose partition assignment isn't actually changing.
- **Cooperative (incremental) rebalancing** (modern default with `CooperativeStickyAssignor`): only the partitions that are actually *moving* are revoked and reassigned; consumers keep processing their unaffected partitions throughout — a direct fix for exactly the "why did lag get worse right after we scaled up" scene above: on an eager rebalancer, adding new consumer instances **paused the entire existing group** while reassignment computed, briefly halting all consumption — the exact opposite of the intended effect — before the new, larger group resumed at higher throughput.

**Why `max.poll.interval.ms` matters specifically**: a consumer is considered "alive" partly based on calling `poll()` frequently enough; if your processing logic between polls (e.g., a slow downstream call per record, without enough internal batching/async-ing) takes longer than `max.poll.interval.ms`, the coordinator considers that consumer dead and triggers a rebalance **even though the consumer process is still healthy and working**, just slow — a very real, very common self-inflicted rebalance-storm cause worth naming precisely if asked to diagnose one.

### Interview angle
> If asked "tell me about a Kafka production issue," this section *is* that story: consumer lag rising, a naive scale-up response, and the actual explanation (eager rebalancing pausing the whole group, exacerbated if `max.poll.interval.ms` was also being exceeded) is layered, specific, and demonstrates genuine internals knowledge rather than "Kafka had lag, we restarted it."

---

## 3. REDIS

### The scene
> "A heavily-read reference-data cache entry — the kind used on every single validation check during feed ingestion — expired at the exact same moment a burst of 50,000 records hit the pipeline. Every one of those in-flight validation calls found the cache empty simultaneously, and every one of them went to the database to refetch and repopulate it, at the same instant — fifty thousand near-simultaneous queries for the identical row, hammering the database at once. That's a **cache stampede** (also called the thundering herd problem), and it's the kind of production incident you can only really prevent if you understand *why* a naive cache-aside pattern has this failure mode built in."

### Basic — data structures and why "just a key-value store" undersells Redis
Redis's core value isn't just speed — it's **purpose-built data structures**, each with O(1) or O(log n) operations for the access patterns they're built for:
- **String** — the basic case (a serialized DTO, a counter via `INCR`).
- **Hash** — a nested object with multiple fields, letting you update/read a single field without deserializing the whole object (e.g., updating just `status` on a cached advisor record without touching the rest).
- **List** — an ordered sequence, useful for simple queue-like structures.
- **Set / Sorted Set** — `Set` for unique membership checks (O(1) `SISMEMBER` — great for "has this idempotency key been seen before" checks); `Sorted Set` (`ZSET`) for anything ranked/time-ordered (a leaderboard, or a sliding-window rate limiter using timestamps as scores).
- **Bitmap / HyperLogLog** — advanced, memory-efficient structures for specific use cases (bitmap for compact boolean flags across huge ID ranges; HyperLogLog for approximate distinct-count estimation at a tiny fixed memory cost — worth knowing exists even if rarely used directly).

### Intermediate — caching patterns, precisely named
- **Cache-aside (lazy loading)** — the application checks the cache first; on a miss, reads from the DB, then populates the cache. Simplest, most common — and the pattern that produces the stampede scene above if not hardened.
- **Write-through** — writes go to the cache *and* the DB synchronously, together, keeping them always consistent, at the cost of write latency (every write pays the cache-write cost too).
- **Write-behind (write-back)** — writes go to the cache immediately, and are asynchronously flushed to the DB later — lowest write latency, but a real durability risk window (a cache failure before the async flush loses the write) — rarely the right choice for anything compliance-adjacent, worth explicitly saying so if asked.
- **Refresh-ahead** — proactively refreshes a cache entry **before** it expires (typically triggered when an entry is accessed within some threshold of its expiry), aiming to avoid ever serving a genuinely-expired entry to a real request — reduces (but doesn't eliminate on its own) stampede risk.

### Advanced — solving the cache stampede precisely
Three complementary, commonly-combined mitigations, worth naming all three:
1. **Locking / single-flight** — on a cache miss, the first request to notice the miss acquires a short-lived **distributed lock** (`SET key value NX PX 5000` — Redis's atomic "set if not exists, with expiry" is exactly built for this) and is the only one that actually queries the DB and repopulates the cache; every other concurrent request either waits briefly and re-checks the cache, or serves a slightly-stale cached value if one is available, rather than all fifty thousand hitting the DB simultaneously.
2. **Jittered TTL** — instead of every related cache entry expiring at exactly the same configured TTL (which is *why* a burst of entries populated together tends to expire together, setting up the stampede precondition), add a small random jitter to each entry's TTL (`baseTtl + random(0, jitterWindow)`), spreading expirations out over time instead of them clustering.
3. **Probabilistic early expiration (a more advanced refinement of refresh-ahead)** — as an entry approaches its expiry, probabilistically (with increasing probability the closer to expiry) trigger a background refresh *before* it actually expires, so under sustained load the cache is very rarely actually empty for a hot key at all.

**Production scenario, applied**: "After that incident, we added jittered TTLs on the reference-data cache (removing the synchronized-expiry precondition) and a short Redis-based lock around the refill path (so even if entries *did* expire together again, only one request per key would actually hit the database, and the other 49,999 would wait milliseconds for that one request's result to land in cache) — a combination, not a single fix, because either one alone reduces but doesn't eliminate the risk."

### Advanced — distributed locking caveats (worth being precise, not just "use Redis for locks")
A naive `SET key value NX PX ttl` lock has real failure modes worth naming if asked about distributed locking specifically: a process that acquires the lock, then pauses (GC pause, network partition) longer than the lock's TTL, has its lock silently expire and **taken by another process** — meaning two processes can now believe they both hold the lock simultaneously. The **Redlock algorithm** (acquiring the lock across a majority of independent Redis instances) is Redis's own proposed mitigation for higher-stakes locking, though it has known, debated edge cases (notably critiqued by Martin Kleppmann) around clock/timing assumptions — the mature, interview-safe answer is: **use Redis locks for best-effort, low-stakes coordination (like the cache-stampede lock above, where "occasionally two requests both refill the cache" is a harmless inefficiency, not a correctness bug); reach for a purpose-built consensus system (or a database-level lock with a proper fencing token) for genuinely correctness-critical distributed locking**, rather than trusting Redis alone for the highest-stakes case.

### Redis Cluster vs Sentinel — sharding vs high availability, not the same problem
- **Sentinel** — monitors a primary/replica Redis deployment and handles **automatic failover** (promoting a replica if the primary fails) — solves **availability**, doesn't shard data at all; every node still holds the full dataset.
- **Cluster** — **shards** data across multiple nodes (via hash slots, 16384 of them, each key mapped to a slot), solving **horizontal scalability** of dataset size/throughput, with its own replication-per-shard for availability layered on top.
- **The precise distinction to state**: Sentinel solves "what if my one Redis dataset's primary dies," Cluster solves "my dataset/throughput no longer fits on one node" — they address genuinely different scaling dimensions, and a system might need Cluster for scale *and* still rely on per-shard replication (conceptually similar to what Sentinel provides) for availability within each shard.

### Interview angle
> The cache-stampede story is one of the best "production scenario" answers available for a Redis question, because it naturally demonstrates you understand cache-aside's failure mode, not just its happy path — and naming three complementary mitigations (not just one) signals depth.

---

## 4. REST API — DEEPER SEMANTICS

### The scene
> "A client's network flaked mid-request during a POST to create a new disclosure-review case. Their retry logic, reasonably, resent the exact same POST. Without any special handling, that created **two** identical case records — a real, live bug, not a hypothetical — because POST, by HTTP's own definition, is not supposed to be safe to blindly retry, and nothing in our API told the client (or protected the server) otherwise."

### Basic — HTTP method semantics, precisely
- **Safe methods** (GET, HEAD, OPTIONS) — must not have side effects; safe to call any number of times, safe to prefetch/cache.
- **Idempotent methods** (GET, PUT, DELETE, and safe methods) — calling N times has the **same effect** as calling once (PUT replacing a resource with the same payload twice leaves the resource in the same end state either time; DELETE-ing an already-deleted resource is still "deleted" either way).
- **POST is neither safe nor idempotent by HTTP's own definition** — which is *exactly* the scene above: a client (or an intermediary proxy, or the client's own retry logic) resending a POST is, by spec, allowed to create a second resource, because HTTP never promised otherwise.

### Intermediate — the Idempotency-Key pattern (the actual fix for the scene above)
```
POST /arl/v1/cases
Idempotency-Key: 6f9a2e21-8f0a-4e11-9c2e-3a441b7e9d21
{ ...case payload... }
```
Server-side: on receiving a request with an `Idempotency-Key` header, check a short-lived store (Redis, naturally — Section 3) for that key. If seen before **and the request completed successfully**, return the **original** response again without re-executing the business logic. If not seen, execute normally and store the key → response mapping (with a bounded TTL — long enough to cover realistic client retry windows, not forever). This is the same pattern payment APIs (Stripe et al.) use, precisely because "don't double-charge on a network retry" is the identical problem to "don't double-create on a network retry."

### Advanced — statelessness, and HATEOAS as a "know it exists, know why most shops skip it" topic
- **Statelessness** (a core REST constraint) — each request must contain everything needed to process it; the server holds no client session state between requests. This is *why* JWT-based stateless auth (Section 5) fits REST's own architectural constraints so naturally, and *why* a session-based auth model is arguably a partial violation of REST's own statelessness principle, worth being able to say precisely if pushed on "is JWT more RESTful than sessions."
- **HATEOAS** (Hypermedia As The Engine Of Application State) — the idea that responses should include links describing *available next actions* (e.g., a case response including a link to "approve" or "escalate" only if the current state allows it), letting clients navigate the API dynamically rather than hardcoding URL structures. **The honest, senior-level answer on HATEOAS**: it's the most theoretically "pure" part of REST's original definition, and also the part almost nobody fully implements in practice, because most API consumers are internal, versioned, and coordinated closely enough that dynamically-discovered affordances add more complexity than they save — worth knowing what it is and being able to explain *why* it's commonly skipped, rather than either pretending it's used everywhere or not knowing it exists at all.

### Interview angle
> "Walk me through a subtle API design bug you fixed" — the duplicate-POST-on-retry scene, explained via safe/idempotent method semantics and fixed via the Idempotency-Key pattern, is a genuinely strong, specific answer.

---

## 5. JWT & SECURITY — DEEPER MECHANICS

### The scene
> "A stolen refresh token got used from an unfamiliar IP months after it was issued. Because we rotated refresh tokens on every use — issuing a brand-new refresh token each time one was redeemed, and immediately invalidating the one just used — the *legitimate* user's next real login attempt tried to redeem a refresh token that had **already been used** (by the attacker, earlier). That single fact — 'this already-used refresh token is being presented again' — is a strong, automatic signal of token theft, and our system flagged and force-invalidated the entire token family, logging the user out everywhere, well before anyone manually noticed anything wrong."

### Basic — JWT structure and signing, precisely
`header.payload.signature`, base64url-encoded, dot-separated.
- **Header**: algorithm (`alg`) and token type.
- **Payload (claims)**: `sub` (subject), `exp` (expiry), `iat` (issued-at), `nbf` (not-before), `aud` (intended audience), `iss` (issuer), plus custom claims (roles/scopes).
- **Signature**: computed over the header+payload using either:
  - **HS256 (HMAC, symmetric)** — the same secret both signs and verifies — simple, but means **every service that needs to verify a token must possess the same shared secret**, which is a real operational/security liability in a microservices architecture (compromise of any one verifying service exposes the signing secret itself).
  - **RS256 (RSA, asymmetric)** — the auth server signs with a **private** key; every other service verifies using only the corresponding **public** key (often fetched dynamically via a **JWKS endpoint** — JSON Web Key Set), which can be freely distributed without risk, because possessing the public key only lets you *verify*, never *forge*, a token. **This is precisely why RS256 (or ES256, its elliptic-curve equivalent) is the standard choice for a multi-service architecture**, and HS256 is more defensible only in a simple, single-service, single-secret-holder scenario.

### Intermediate — the claims-validation checklist (what "validating a JWT" actually means beyond signature check)
A correct verifier checks, at minimum: **signature validity** (tamper detection), **`exp`** (rejects expired tokens), **`nbf`** (rejects tokens presented before they're valid, relevant for tokens issued with a deliberate future activation time), **`iss`** (rejects tokens from an unexpected issuer — relevant if multiple auth servers/environments could otherwise produce structurally-valid-looking tokens), and **`aud`** (rejects a token that was issued *for* a different service — without an audience check, a token legitimately issued for Service A could be replayed against Service B if B only checks signature validity, a real, named vulnerability class: **the "confused deputy" problem**, worth naming precisely). A validator that only checks the signature and expiry, skipping `iss`/`aud`, is a common, subtle real-world security gap.

### Advanced — refresh-token rotation and reuse detection (the scene above, mechanically)
1. On login, issue an access token (short-lived, minutes) and a refresh token (longer-lived, tracked server-side as part of a "token family," often with a unique ID stored, not just a bare opaque string).
2. On refresh, **invalidate the presented refresh token and issue a brand-new one** — the old one can never be redeemed again.
3. **If an already-invalidated (already-used) refresh token is ever presented again**, that's a near-certain signal of theft (either the legitimate user's token was stolen and used by an attacker before the user's own next refresh, or vice versa) — the correct response is to **invalidate the entire token family immediately**, forcing full re-authentication, not just silently rejecting the one bad request.
4. This is meaningfully stronger than a fixed-lifetime, non-rotating refresh token, because it converts "a stolen long-lived token is quietly usable for its entire remaining lifetime" into "a stolen token is usable exactly once before the theft is detected and the whole session family is killed."

### Advanced — revocation, honestly stated as a real trade-off (not solved for free by JWT)
Stateless JWT verification (no DB lookup needed to check a token, Section 6 of the earlier 360° doc) is exactly what makes horizontal scaling trivial — and it's exactly what makes **immediate revocation** (e.g., "log this user out right now, everywhere, this instant, because we detected fraud") **not free**. The honest trade-off: either accept that a compromised-but-not-yet-expired access token remains valid until its natural (short) expiry, or add a **revocation/blocklist check** (typically a fast Redis lookup — Section 3 again — checked on every request), which reintroduces exactly the kind of shared-state check stateless JWTs were meant to avoid, just made fast and horizontally-scalable via Redis rather than a heavier DB round-trip. **Most production systems land on a middle ground**: keep access tokens genuinely short-lived (minutes) so the "damage window" without a blocklist is small, and reserve an explicit Redis-backed blocklist specifically for high-severity events (confirmed account compromise) rather than checking it on every single request as a default.

### OWASP-adjacent security patterns worth naming for a compliance-adjacent API
- **Broken Object Level Authorization (BOLA / IDOR)** — the single most common real-world API vulnerability class: checking that a caller is *authenticated*, but forgetting to check they're *authorized for this specific resource ID* (e.g., `/arl/v1/advisors/{crdNumber}` returning data for **any** CRD number to **any** authenticated caller, rather than checking the caller's scope actually covers that specific advisor/jurisdiction). Worth naming precisely — "we always check both authentication *and* per-resource authorization, not just authentication" is a strong, specific answer.
- **Excessive data exposure** — returning a full internal entity (or a DTO carrying more fields than the specific consumer needs) rather than a purpose-built response DTO — ties directly back to the DTO-mapping discipline from the earlier 360° doc, now framed explicitly as a *security* control, not just an architectural-cleanliness one.
- **Security misconfiguration** — a genuinely common, unglamorous root cause (default credentials left active, verbose stack traces leaking internals to API responses, permissive CORS configured for convenience during development and never tightened) — worth mentioning because interviewers sometimes probe specifically for "what's the boring-but-real vulnerability class," and this is it.

### Interview angle
> The refresh-token-reuse-detection story is one of the strongest security answers available precisely because it demonstrates understanding of *why* rotation matters mechanically (not just "we rotate tokens because it's best practice") — walking through exactly what reuse detection catches, and why, shows real depth.

---

## 6. TYING IT TOGETHER — DESIGN PATTERNS & TRADE-OFFS ACROSS ALL FIVE TOPICS

A quick cross-reference, since the deeper architecture/patterns/trade-offs doc already covers the general catalog — here's specifically how *this* document's five topics map onto it:

| Concept from this doc | Pattern/principle it's an instance of |
|---|---|
| `@ConditionalOnMissingBean` overriding auto-config | Convention over configuration, with an explicit escape hatch (recurring theme from the architecture doc) |
| Filter / Interceptor / AOP layering | Chain of Responsibility (Filters) + Decorator (AOP proxies) |
| Kafka `acks=all` + `min.insync.replicas` | Fail-closed-on-durability (same family as mTLS STRICT) |
| Cooperative rebalancing | A direct example of minimizing blast radius during a state-change event (only affected partitions pause) |
| Redis cache-aside + lock + jitter | Bulkhead-adjacent (containing the "stampede" blast radius) + Circuit-Breaker-adjacent (the lock briefly protects the DB from overload) |
| Idempotency-Key pattern | Idempotent Receiver, applied to synchronous REST rather than async messaging |
| RS256 + JWKS | Principle of least privilege applied to cryptographic material distribution (public key freely shareable, private key never leaves the issuer) |
| Refresh-token rotation + reuse detection | Defense in depth + fail-secure design (a detected anomaly triggers the *safer* outcome — full logout — by default) |

---

## 7. THE STORYTELLING FRAMEWORK — HOW TO DELIVER ANY OF THIS IN AN INTERVIEW

Every section above follows the same five-beat structure — internalize the *shape*, and you can improvise it for any topic, including ones not in this document:

1. **The scene** — one or two sentences, concrete, specific ("consumer lag climbing steadily, we scaled up, it got briefly worse"). This is what makes an answer memorable instead of generic.
2. **The naive expectation** — what you (or a less-experienced engineer) would reasonably assume before understanding the internals ("just add more consumers, lag should drop immediately").
3. **The mechanism** — the actual internals that explain why the naive expectation was wrong (eager rebalancing pausing the whole group).
4. **The fix** — specific and named (cooperative rebalancing, or `max.poll.interval.ms` tuning, or both).
5. **The generalizable principle** — one sentence connecting it to a broader design idea (blast-radius minimization, fail-closed-on-durability, defense in depth) — this is the sentence that shows you're not just recounting an incident, you extracted a transferable lesson from it.

**Practice prompt**: pick any three sections above and, without looking, retell them in under 90 seconds each using exactly this five-beat structure. That rehearsal — not memorizing facts — is what makes this material actually usable live in a room.
