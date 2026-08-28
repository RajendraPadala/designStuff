# Java Concurrency & Collections — Internals, Evolution & System Design
### Threads and Collections/Streams: From JDK 1.0 to Java 21+, With Performance Reasoning and Real-Time Design Scenarios

---

## HOW TO READ THIS DOCUMENT

Part 1 covers **Java Threads** end-to-end, chronologically, with internals at each stage. Part 2 covers **Collections & Streams** the same way. Part 3 distills the **before/after Java 8** performance story specifically, since that's the single biggest inflection point in both timelines. Part 4 is the **system design lens** — decision trees and trade-off tables. Part 5 gives you **two fully-worked real-time design scenarios** using everything above, the way an interviewer actually wants to see it applied, not just recited.

---

# PART 1 — JAVA THREADS: EVOLUTION & INTERNALS

## 1.1 The Beginning — Green Threads vs. Native Threads (JDK 1.0–1.1, 1996–1997)

**What happened**: early JVMs (notably on Solaris) implemented threads as **green threads** — threads scheduled entirely by the JVM itself, multiplexed onto a single OS thread, with no help from the OS scheduler. This was a portability decision: OS-level threading support was inconsistent across platforms in 1996.

**Why it mattered, and why it failed**: green threads couldn't take advantage of multiple CPU cores (everything ran on one OS thread) and had no access to OS-level I/O scheduling — meaning a blocking I/O call in one green thread could stall *every* green thread. By JDK 1.1–1.2, Sun abandoned green threads on most platforms in favor of **native threads** — a JVM thread mapped 1:1 to a real OS thread, scheduled by the OS's own preemptive scheduler.

**The lasting consequence**: this 1:1 platform-thread-to-OS-thread model became Java's threading model for the next **25 years**, and it's precisely the model **virtual threads (Section 1.8)** finally broke away from in 2023 — worth stating this arc explicitly in an interview, because it shows you understand virtual threads aren't a random new feature, they're the JVM *revisiting a 1996-era decision* with modern engineering.

## 1.2 The synchronized/Monitor Model & the Broken Early Memory Model (JDK 1.0–1.4)

**Mechanics**: every Java object has an implicit **monitor** (a mutex + wait-set), acquired via `synchronized`. Internally, this maps to the object's header **Mark Word** — a JVM-internal field that, depending on lock state, holds either a hash code, a biased-lock thread pointer, a lightweight-lock stack pointer, or a heavyweight-lock's monitor pointer (this specific escalation path — biased → lightweight → heavyweight — was itself introduced later, in Java 6, as a performance optimization; JDK 1.0–1.4 essentially only had the heavyweight (OS mutex-backed) form, meaning **every** `synchronized` block paid full OS-mutex cost even for uncontended, single-threaded-in-practice locking).

**The critical, underappreciated problem**: the **original Java Memory Model (JMM), as specified pre-2004, was formally broken**. It allowed compiler/CPU reorderings that could produce results no programmer would expect, and — most notoriously — the classic **double-checked locking idiom** was provably unsafe under the original JMM, because a reading thread could observe a **partially-constructed object** (the reference write could be observed before the constructor's field-initialization writes completed, from another thread's perspective). This wasn't a minor edge case; it invalidated a pattern that had been taught as correct for years.

## 1.3 JSR-133 — The Java Memory Model Rewrite (Java 5, 2004) — THE pivotal moment for threading correctness

This is arguably **the single most important event in Java concurrency's history**, more foundational than any single API addition, and worth being able to discuss precisely:

- **`volatile` semantics were substantially strengthened**. Pre-JSR-133, `volatile` only guaranteed that reads/writes to that *specific field* weren't reordered relative to each other. Post-JSR-133, a `volatile` write establishes a **happens-before** edge with any subsequent `volatile` read of the same field by another thread — meaning **everything the writing thread did before the volatile write becomes visible** to the reading thread after it reads that volatile field, not just the field itself. This is what made the **safe double-checked locking with `volatile`** pattern actually correct, and it's the formal foundation every concurrent Java data structure built afterward relies on.
- **`final` field semantics were fixed** — a properly-constructed object's `final` fields are guaranteed visible to any thread that obtains a reference to the object after construction completes, without needing additional synchronization, closing the "observing a partially-constructed object" hole for immutable objects specifically.
- **The formal happens-before relationship** was defined precisely for the first time: program order within a thread, monitor unlock → subsequent lock on the same monitor, volatile write → subsequent volatile read, `Thread.start()` → anything the started thread does, and everything a thread does → a successful `Thread.join()` on it by another thread.

**Why this belongs in a "before/after" performance discussion, not just a correctness one**: a *correct*, well-specified memory model is what allowed the JVM's JIT compiler and CPU to perform **aggressive reordering and optimization everywhere synchronization isn't required**, while giving programmers a precise contract for where it *is* required. A vague or broken memory model, ironically, tends to produce *more* conservative (slower) compiler behavior over time, because compiler engineers can't safely reorder around undefined semantics. JSR-133 is as much a performance enabler as a correctness fix.

## 1.4 java.util.concurrent (Java 5, 2004) — the modern concurrency toolkit arrives

Released in the same JDK as JSR-133 (not a coincidence — Doug Lea's `java.util.concurrent`, JSR 166, was built *on top of* the newly-correct memory model):

- **`ExecutorService` / thread pools** — replacing manual `new Thread(...).start()` patterns with managed, reusable worker pools (`ThreadPoolExecutor` internals: `corePoolSize`, `maximumPoolSize`, a `BlockingQueue` work queue, and — a commonly-misunderstood mechanic — the pool only grows **beyond** `corePoolSize` once the work queue is **completely full**, not merely because tasks are queuing up).
- **Lock framework (`java.util.concurrent.locks`)** — `ReentrantLock` as an explicit alternative to `synchronized`, adding capabilities `synchronized` never had: **tryLock with timeout** (avoiding indefinite blocking), **interruptible lock acquisition**, and **fairness policies** (FIFO lock granting, at a throughput cost). `ReadWriteLock`/`ReentrantReadWriteLock` — allowing many concurrent readers when no writer holds the lock, a direct throughput win for read-heavy shared state.
- **Atomic variables (`AtomicInteger`, `AtomicLong`, `AtomicReference`, etc.)** — built on **CAS (Compare-And-Swap)**, a hardware-level instruction that atomically updates a value only if it still matches an expected prior value, retrying otherwise. This is **lock-free**, not merely "uses a smaller lock" — under moderate contention, CAS-based atomics dramatically outperform `synchronized` because there's no OS-level blocking/context-switch involved at all, just a tight retry loop.
- **Concurrent collections** (`ConcurrentHashMap`, `CopyOnWriteArrayList`, `BlockingQueue` implementations) — covered in depth in Part 2, since they're as much a Collections-framework story as a threading one.
- **Coordination primitives** — `CountDownLatch` (one-shot gate), `CyclicBarrier` (reusable, resets automatically), `Semaphore` (bounded concurrent-access permits), `CompletableFuture`'s predecessor `FutureTask`.

**The performance framing worth stating explicitly**: before Java 5, achieving correct, non-crude concurrency meant either hand-rolling `wait()`/`notify()` monitor logic (error-prone, hard to reason about, and coarse-grained — `synchronized` locks the *entire* method/block, no partial concurrency) or using external, heavier concurrency libraries. `java.util.concurrent` gave the platform **fine-grained, composable, benchmarked-and-tuned concurrency primitives as a standard library**, and this is the toolkit most production Java concurrency code still runs on today, 20 years later.

## 1.5 Fork/Join Framework (Java 7, 2011)

**Mechanics**: `ForkJoinPool` implements **work-stealing** — each worker thread has its own double-ended work queue; a thread that runs out of work steals from the *tail* of another thread's queue (while the owning thread takes from its own *head*), minimizing contention on any single queue. Designed for **recursively decomposable** problems (`RecursiveTask<V>`/`RecursiveAction`) — split a problem into halves until small enough to solve directly, then combine results.

**Why it matters for the Java 8 story**: `ForkJoinPool.commonPool()`, introduced alongside this framework, became the **default execution engine for parallel streams** in Java 8 — meaning parallel streams didn't introduce a new concurrency model, they *reused* a three-years-prior addition, which is exactly why parallel streams share the common pool with any other Fork/Join-based code running in the same JVM (a real, frequently-tested interview gotcha, covered in Part 3).

## 1.6 Java 8 (2014) — CompletableFuture and the parallel-streams threading model

- **`CompletableFuture`** — composable async computation: `thenApply`/`thenCompose`/`thenCombine` for chaining, `allOf`/`anyOf` for fan-out/fan-in, `exceptionally`/`handle` for error paths. The `...Async` variants execute their continuation on a **different thread** — by default, again, the common `ForkJoinPool`, unless an explicit `Executor` is supplied — a detail that trips people up when a continuation ends up running on a thread that was never intended to do blocking work.
- **Parallel streams' threading**: `.parallelStream()` splits the source via a `Spliterator` (Part 2) and processes chunks across `ForkJoinPool.commonPool()` workers. This is powerful for large, CPU-bound, evenly-splittable in-memory computation, and a **liability** for small collections (fork/join coordination overhead exceeds the work) or I/O-bound operations (a blocked common-pool worker thread starves every other concurrent user of that shared pool in the JVM — parallel streams, `CompletableFuture.supplyAsync()` without an explicit executor, and any other Fork/Join-based code all compete for the same limited pool).

## 1.7 Reactive Streams / Flow API (Java 9, 2017)

`java.util.concurrent.Flow` standardized the **reactive streams** interfaces (`Publisher`, `Subscriber`, `Subscription`, `Processor`) already popularized by libraries like RxJava and Project Reactor — introducing **backpressure** as a first-class concept: a subscriber explicitly requests how many items it can handle (`Subscription.request(n)`), preventing a fast producer from overwhelming a slow consumer's memory. Java 9 standardized the *interfaces*, not a full reactive implementation — actual reactive programming in Java still typically leans on Reactor/RxJava/Akka Streams built against these standard interfaces.

## 1.8 Virtual Threads / Project Loom (Java 21, 2023) — the biggest leap since 2004

This directly revisits Section 1.1's 1:1 platform-thread model, finally giving the JVM its own lightweight, user-mode-scheduled threads:

- **Mechanics**: a virtual thread is a cheap, JVM-managed thread. Millions can exist simultaneously (vs. thousands at most for platform threads, each costing roughly 1MB of stack by default plus real OS scheduling overhead). Many virtual threads are **multiplexed onto a small pool of carrier (platform/OS) threads**. When a virtual thread performs a **blocking operation the JDK has made virtual-thread-aware** (most blocking I/O — sockets, files, and JDBC drivers that have been updated to cooperate), it **unmounts** from its carrier thread, freeing the carrier for other work, and **remounts** when the operation completes.
- **The concrete performance story**: pre-Loom, scaling I/O-bound concurrency to tens of thousands of simultaneous operations required either an enormous platform-thread count (expensive, eventually hits OS limits) or a rewrite into asynchronous/reactive code (Section 1.7) — correct, scalable, but much harder to write, debug, and reason about (no clean stack traces across async boundaries, callback/composition complexity). Virtual threads let you write **plain, blocking-looking, synchronous code** that scales like reactive code, because the *scheduling* trick happens transparently beneath the language, not in the code you write.
- **Real caveat worth naming precisely**: a virtual thread blocking inside a `synchronized` block historically could **pin** its carrier thread (unable to unmount during that block), defeating the scalability benefit for that code path — a genuine reason to prefer `ReentrantLock` over `synchronized` in hot, blocking-I/O-adjacent paths on earlier virtual-thread JDK versions (later JDK releases progressively narrowed this limitation).
- **Virtual threads help I/O-bound concurrency only** — they do nothing for CPU-bound work, where a bounded platform-thread pool sized near `Runtime.getRuntime().availableProcessors()` remains the correct tool.

## 1.9 Structured Concurrency & Scoped Values (Java 21 preview / Java 25 finalized)

- **Structured Concurrency** treats a set of forked concurrent subtasks as a **single unit of work** — if one subtask fails, the others are automatically cancelled, and the parent doesn't proceed until all subtasks complete or the scope is cancelled. This directly fixes the classic manual-`ExecutorService`-coordination bug: a forgotten cancellation on partial failure leaving orphaned background work running.
- **Scoped Values** replace `ThreadLocal` for context propagation (e.g., a request's trace ID through a deep call chain) in a way that's cheap and safe at virtual-thread scale — `ThreadLocal`'s mutable, per-thread-copied design doesn't fit well with potentially millions of cheap virtual threads; Scoped Values are immutable for the duration of a scope and designed for exactly this propagation pattern.

---

# PART 2 — JAVA COLLECTIONS & STREAMS: EVOLUTION & INTERNALS

## 2.1 Pre-Framework Era (JDK 1.0–1.1, 1996) — Vector, Hashtable, Enumeration

The original collection-like types: `Vector` (a growable array), `Hashtable` (a hash-based map), `Stack` (extends `Vector`), all **synchronized on every method by default** — meaning even single-threaded code paid lock-acquisition cost on every `add()`/`get()`/`put()`, a real, measurable tax with no way to opt out short of not using them. Iteration used `Enumeration`, a minimal, one-directional, non-removable iteration interface.

## 2.2 The Collections Framework (Java 1.2, 1998)

A genuine architectural rewrite: `Collection`/`List`/`Set`/`Map` interfaces, concrete implementations (`ArrayList`, `LinkedList`, `HashMap`, `HashSet`, `TreeMap`, `TreeSet`), and — critically — **unsynchronized by default** (`ArrayList` instead of the always-synchronized `Vector`, `HashMap` instead of `Hashtable`), with `Collections.synchronizedList(...)`/`synchronizedMap(...)` as an explicit opt-in wrapper for the cases that actually needed thread safety. This was a direct performance response to 1.1's "pay for synchronization whether you need it or not" design. `Iterator` replaced `Enumeration`, adding `remove()` support (safe removal during iteration) and bidirectional traversal for `ListIterator`.

## 2.3 Generics (Java 5, 2004)

**Mechanics**: `List<String>` instead of a raw `List` requiring casts everywhere. Implemented via **type erasure** — generic type information exists only at compile time (for type-checking) and is erased to raw types in the compiled bytecode; a `List<String>` and a `List<Integer>` are the *same* class at runtime, `List`. This is *why* you can't do `new T[]` or `instanceof List<String>` — the runtime genuinely doesn't retain that information.

**Performance framing**: generics themselves add **zero runtime overhead** relative to pre-generics manually-cast code (erasure means the bytecode is essentially the same, just with compiler-inserted casts and compile-time checks) — the real performance story is what generics *enabled* afterward: type-safe APIs that could be aggressively used throughout the Streams API (Part 2.5) without a wave of `ClassCastException` risk or manual casting overhead littering every call site.

## 2.4 Concurrent Collections (Java 5, 2004)

- **`ConcurrentHashMap`** — the flagship. **Pre-Java-8 internals**: divided into a fixed number of **segments** (16 by default), each independently `synchronized`-lockable — allowing up to 16 concurrent writers to *different* segments simultaneously, a massive improvement over `Collections.synchronizedMap()`'s single global lock for the entire map.
- **`CopyOnWriteArrayList`/`CopyOnWriteArraySet`** — every mutation copies the entire backing array; reads need **zero locking** (iterating a stable snapshot). Ideal for read-heavy, write-rare data (e.g., a small list of registered listeners); actively harmful for write-heavy usage (every write is an O(n) full-array copy).
- **`BlockingQueue` implementations** — `ArrayBlockingQueue` (bounded, single lock shared by put/take by default) vs. `LinkedBlockingQueue` (optionally bounded, **separate locks for put and take**, generally higher producer/consumer throughput since a producer and consumer don't contend on the same lock).

## 2.5 Java 8 (2014) — The Big Rewrite

This is the second pivotal moment (alongside Section 1.3) worth deep familiarity with:

### Default methods on interfaces
`Collection.forEach()`, `Collection.removeIf()`, `Map.computeIfAbsent()`/`merge()`/`getOrDefault()`, `List.replaceAll()`/`sort()` — added as **default methods**, a language feature specifically introduced *to make this possible* without breaking every existing implementer of `Collection`/`Map` across the ecosystem (a class implementing `Map` written in 2001 still compiles and works in Java 8+, automatically gaining `computeIfAbsent` for free via the default implementation). This is *why* default methods exist at all — retrofitting the Collections Framework was literally the motivating use case for the language feature.

### ConcurrentHashMap's Java 8 redesign
**Segments were removed entirely.** Locking became **per-bin (per-bucket)** — using `synchronized` on the bin's first node combined with **CAS operations** for cases like inserting into an empty bin — giving contention proportional to actual hash-bin collisions, not an artificial fixed count of 16 segments. This is a substantial, real internals change worth being able to describe precisely if asked "how does ConcurrentHashMap work" — the segment-based answer is now a decade-plus out of date.

### HashMap treeification
Since Java 8, a bin (bucket) accumulating **8 or more** colliding entries (`TREEIFY_THRESHOLD = 8`, only actually converting if the table has at least 64 buckets total, `MIN_TREEIFY_CAPACITY`) converts from a **linked list to a red-black tree** for that bin — changing worst-case lookup within a badly-collided bucket from O(n) to O(log n), a defense against hash-flooding (adversarial or accidental keys all colliding into one bucket). Bins **untreeify** back to a linked list below 6 entries (`UNTREEIFY_THRESHOLD`), avoiding tree-maintenance overhead once the collision pressure subsides. Load factor (default 0.75) still governs when the whole table resizes/rehashes.

### The Streams API — internals worth knowing precisely
- **Laziness**: intermediate operations (`filter`, `map`) build a pipeline description but execute **nothing** until a terminal operation (`collect`, `forEach`, `reduce`) is invoked — this is what allows an infinite stream (`Stream.iterate(0, n -> n + 1)`) to be legal, as long as `.limit(n)` bounds it before evaluation forces anything.
- **`Spliterator`** — the successor to `Iterator`, designed specifically for **parallel splitting**: `trySplit()` attempts to partition the remaining elements into two roughly-equal pieces for separate processing, `estimateSize()` informs the Fork/Join framework's decomposition strategy. A source's `Spliterator` characteristics (`SIZED`, `SUBSIZED`, `ORDERED`, `IMMUTABLE`, etc.) directly affect how well it parallelizes — an `ArrayList`'s array-backed Spliterator splits cheaply and evenly; a `LinkedList`'s does not, which is *why* `LinkedList.parallelStream()` rarely helps.
- **Short-circuiting operations** (`findFirst`, `anyMatch`, `limit`) stop processing as soon as the answer is determined, which is only meaningfully beneficial because of laziness — an eager pipeline would have already done all the work before a "stop early" operation could matter.
- **Primitive streams** (`IntStream`, `LongStream`, `DoubleStream`) exist specifically to avoid **autoboxing overhead** — a generic `Stream<Integer>` boxes every `int` into an `Integer` object, with real allocation and cache-locality costs at scale; `IntStream` operates on raw primitives throughout the pipeline, a genuinely significant performance difference for numeric-heavy processing that's a common, concrete "why would you use IntStream instead of Stream<Integer>" interview answer.

## 2.6 Java 9 (2017) — Immutable collection factories

`List.of(...)`, `Set.of(...)`, `Map.of(...)` — create genuinely **immutable** collections (throwing `UnsupportedOperationException` on any mutation attempt, unlike `Collections.unmodifiableList()`, which merely wraps a still-mutable backing list that *someone else* holding the original reference could still change out from under you). Also `Stream.takeWhile()`/`dropWhile()`, filling a gap the original Streams API left for ordered, condition-based partial consumption.

## 2.7 Java 10 (2018) — `var`, and `List.copyOf()`/etc.

Local-variable type inference (`var`) — compile-time-checked, not dynamic typing, purely a readability/boilerplate reduction with zero runtime cost (erased to the inferred static type at compile time, identical bytecode to writing the type explicitly).

## 2.8 Java 16 — `Stream.toList()`

A shorthand for the extremely common `.collect(Collectors.toList())`, additionally returning an **unmodifiable** list (unlike `Collectors.toList()`, which makes no guarantee about mutability of the returned list) — a small but real semantic difference worth knowing if asked to compare the two.

## 2.9 Java 21 — Sequenced Collections

A new interface family (`SequencedCollection`, `SequencedSet`, `SequencedMap`) retrofitted onto `List`, `LinkedHashSet`, `LinkedHashMap`, and others, giving **consistent** `getFirst()`/`getLast()`/`addFirst()`/`addLast()`/`reversed()` methods across collection types that previously had inconsistent or entirely missing ways to do this (e.g., getting the "first" and "last" element of a `LinkedHashSet` previously required manual iteration; there was no direct API for it at all).

---

# PART 3 — PERFORMANCE: BEFORE vs. AFTER JAVA 8, CONCRETELY

## 3.1 Iteration: external vs. internal, and what it means for optimization

**Before (external iteration)**:
```java
List<Order> highValue = new ArrayList<>();
for (Order o : orders) {
    if (o.getAmount() > 1000) {
        highValue.add(o);
    }
}
```
**After (internal iteration via Streams)**:
```java
List<Order> highValue = orders.stream()
    .filter(o -> o.getAmount() > 1000)
    .collect(Collectors.toList());
```
**The performance-relevant distinction**: with external iteration, *you* control the loop, and the JIT optimizes exactly the loop you wrote — no more, no less. With internal iteration, the **Streams implementation** controls traversal, which means the JVM/library authors can (and did, over subsequent JDK releases) improve the underlying execution strategy — better Spliterator-based splitting, improved short-circuit handling — **without your code changing at all**. This is a genuine, if subtle, architectural performance advantage: internal iteration decouples "what to compute" from "how to traverse," letting the *how* improve independently.

## 3.2 Concurrency: manual thread management vs. java.util.concurrent vs. parallel streams

**Before Java 5 (raw threads)**:
```java
List<Thread> threads = new ArrayList<>();
for (Chunk c : chunks) {
    Thread t = new Thread(() -> process(c));
    t.start();
    threads.add(t);
}
for (Thread t : threads) t.join(); // manual, error-prone coordination
```
**After Java 5 (ExecutorService)**:
```java
ExecutorService pool = Executors.newFixedThreadPool(4);
List<Future<Result>> futures = chunks.stream()
    .map(c -> pool.submit(() -> process(c)))
    .collect(Collectors.toList());
```
**After Java 8 (parallel streams, for CPU-bound, evenly-splittable work)**:
```java
List<Result> results = chunks.parallelStream()
    .map(this::process)
    .collect(Collectors.toList());
```
**The performance trade-off worth being precise about**: each step trades manual control for higher-level abstraction and reduced bug surface (thread-count tuning, coordination, exception handling all get progressively more managed), but each higher-level abstraction also makes **implicit resource-sharing decisions** you must understand — parallel streams silently use the shared `ForkJoinPool.commonPool()`, meaning it's easy to get *better* code with *worse* actual production behavior if you don't know that pool is shared across the entire JVM process, including unrelated code.

## 3.3 HashMap under adversarial/heavy collision load

**Before Java 8**: a bucket with many colliding entries was **always** a linked list — worst-case lookup within a bad bucket was O(n), and a maliciously (or just unluckily) crafted set of colliding keys could degrade `HashMap` performance toward that of a single linked list across the whole table.
**After Java 8**: treeification (Section 2.5) caps worst-case bucket lookup at O(log n) once collisions cross the threshold — a direct, measurable robustness and worst-case-performance improvement, not just an internals curiosity.

## 3.4 The parallel-streams overhead crossover point — a concrete rule of thumb worth stating in an interview

Parallel streams pay **fixed coordination costs** (splitting via Spliterator, task scheduling on the Fork/Join pool, result combination) that a sequential stream doesn't. For a **small collection** (a common rough heuristic: fewer than ~10,000 elements for simple per-element work, though the real threshold depends heavily on how expensive each element's processing is), that fixed overhead exceeds any parallelism benefit — sequential is faster. Parallel streams win specifically when: the collection is large, the per-element work is genuinely CPU-bound (not I/O-bound — Section 1.6's shared-pool starvation risk), and the source splits cheaply and evenly (array-backed sources like `ArrayList`/arrays split far better than `LinkedList` or I/O-sourced streams). **This is a "measure, don't assume" answer** — the correct interview response to "should I use parallel streams here" is describing this reasoning, not a blanket yes/no.

---

# PART 4 — SYSTEM DESIGN INTERVIEW LENS

## 4.1 Collection choice decision table

| Need | Choice | Why |
|---|---|---|
| Ordered, index-based access, read-heavy | `ArrayList` | O(1) random access, good cache locality (contiguous array) |
| Frequent insert/remove at known node (e.g., via iterator), rarely random-accessed | `LinkedList` / `ArrayDeque` for pure queue/stack use | O(1) insert/remove at the point of reference — but see 3.4's cache-locality caveat: `ArrayList` often still wins in practice at real-world sizes |
| Fast key lookup, no ordering needed | `HashMap` | O(1) average case |
| Fast key lookup, insertion order matters | `LinkedHashMap` | Same O(1) lookup, plus predictable iteration order; also the basis of a simple LRU cache (`removeEldestEntry` override) |
| Fast key lookup, sorted key order / range queries needed | `TreeMap` | O(log n), Red-Black tree, supports `headMap`/`tailMap`/`ceilingKey` etc. |
| Thread-safe map, high concurrent read/write | `ConcurrentHashMap` | Per-bin locking + CAS, far better than a globally-locked `synchronizedMap` |
| Thread-safe list, read-heavy/write-rare | `CopyOnWriteArrayList` | Zero-lock reads |
| Producer/consumer queue | `BlockingQueue` impl (`ArrayBlockingQueue`/`LinkedBlockingQueue`) | Built-in blocking put/take semantics |
| Immutable, defensively-shared collection | `List.of(...)`/`Set.of(...)`/`Map.of(...)` (Java 9+) | Genuinely unmodifiable, not just a wrapper |

## 4.2 Concurrency-model decision table

| Workload profile | Choice | Why |
|---|---|---|
| CPU-bound, parallelizable, in-memory, large dataset | Parallel streams / `ForkJoinPool` | Built for exactly this; avoid for small data or I/O-bound work |
| I/O-bound, many concurrent operations, simple code desired | Virtual threads (Java 21+) | Blocking-style code, scales via cheap JVM-managed threads |
| I/O-bound, need genuine backpressure / streaming composition | Reactive (Reactor/RxJava on Flow API) | Explicit flow-control the JDK's virtual threads don't provide by themselves |
| CPU-bound, fixed/small worker count | Bounded `ThreadPoolExecutor` sized near core count | Virtual threads don't help CPU-bound work; a correctly-sized platform pool is still right |
| Coordinating a fixed set of fan-out subtasks with unified cancellation | Structured Concurrency (Java 21+) | Eliminates the "forgot to cancel a sibling" bug class |
| Simple shared counter/flag under contention | `AtomicInteger`/`AtomicLong`/CAS-based types | Lock-free, faster than `synchronized` under moderate contention |
| Coarse-grained mutual exclusion, simple case | `synchronized` | Simplest correct option when you don't need `ReentrantLock`'s extra capabilities |
| Need tryLock/timeout/interruptible/fairness | `ReentrantLock` | `synchronized` can't express any of these |

## 4.3 The "why not just use X everywhere" pushback questions to prepare for

- **"Why not always use `ConcurrentHashMap` instead of `HashMap`?"** — Overhead. Every operation on `ConcurrentHashMap` pays some synchronization cost even under zero contention; a single-threaded or externally-synchronized context should just use `HashMap`.
- **"Why not always use virtual threads?"** — They help I/O-bound concurrency specifically; CPU-bound work sees no benefit, and virtual threads still have caveats (Section 1.8's pinning history, and libraries that haven't been made virtual-thread-aware for blocking calls won't unmount properly).
- **"Why not always parallelize streams?"** — Section 3.4's crossover-point reasoning, plus the shared common-pool starvation risk for I/O-bound work.

---

# PART 5 — TWO REAL-TIME DESIGN SCENARIOS

## Scenario 1: Real-Time Leaderboard / Metrics Aggregation Service

**Requirements**: ingest a high-throughput stream of scoring events (tens of thousands/sec) from multiple producers, maintain a live "top N" leaderboard queryable at low latency, and expose periodic aggregate stats (avg/percentiles) without blocking ingestion.

**Design walkthrough**:
- **Ingestion layer**: each incoming event handled by a virtual thread (Java 21+) — ingestion is I/O-bound (accepting network connections, deserializing), and the whole point of a leaderboard service is handling *many concurrent* producers without hand-tuning a platform-thread pool size against an unknown, possibly-spiky producer count.
- **Shared state**: a `ConcurrentHashMap<String, AtomicLong>` (playerId → current score) for O(1) average-case concurrent updates — chosen over `synchronizedMap(new HashMap<>())` specifically because per-bin locking scales far better under the concurrent-write load this system is defined by; chosen over `Hashtable` because it's a decades-obsolete, single-lock legacy type with no reason to still be used.
- **"Top N" maintenance**: rather than sorting the entire (potentially millions-of-entries) map on every read, maintain a bounded structure — e.g., a periodically-refreshed `TreeMap<Long, List<String>>` (score → players) rebuilt on a schedule (every 1–2 seconds) via a background virtual thread reading a **consistent snapshot** of the `ConcurrentHashMap`, rather than trying to keep a perfectly-live sorted structure under concurrent mutation (which would require far more expensive coordination for a use case that doesn't actually need sub-second-perfect ranking).
- **Aggregate stats**: computed via a scheduled task using `IntStream`/`LongStream` (Section 2.5) over a snapshot of current scores — using primitive streams specifically to avoid boxing overhead across what could be millions of entries, and using `summaryStatistics()` for a single-pass mean/min/max/count computation rather than multiple separate stream traversals.
- **Why not parallel streams for the snapshot aggregation?** — Tempting, but the snapshot itself is typically not large enough (relative to Section 3.4's crossover reasoning) to clearly benefit, and it would compete with the ingestion-side virtual threads' carrier-thread pool if not carefully isolated — a concrete, defensible "we measured it, sequential was fine" answer beats reflexively parallelizing.
- **Trade-off explicitly stated**: this design accepts **eventual, near-real-time** leaderboard consistency (1–2 second staleness on rank) in exchange for not paying the cost of a perfectly-live sorted structure under heavy concurrent write load — a deliberate CAP-adjacent trade-off (favoring availability/throughput of writes over instantaneous read consistency), worth naming explicitly as a design decision, not an oversight.

## Scenario 2: Rate Limiter + Bounded Connection Pool for a High-Traffic API Gateway

**Requirements**: protect a downstream service from being overwhelmed — enforce a per-client rate limit and a global concurrent-connection cap, under high concurrent request volume, with minimal added latency.

**Design walkthrough**:
- **Per-client rate limiting**: a `ConcurrentHashMap<String, AtomicInteger>` (clientId → request count in the current window) — `computeIfAbsent`/`merge` (Java 8 default methods, Section 2.5) used for atomic "increment or initialize" logic in one call rather than a manual get-check-put race-prone sequence. A scheduled virtual thread resets/decays counters per window (sliding or fixed, depending on requirements) — chosen over a heavier token-bucket library dependency because the core primitive (`AtomicInteger` + `ConcurrentHashMap`) already provides everything needed at this scale without extra operational surface area.
- **Global concurrency cap**: a `Semaphore` sized to the downstream service's known safe concurrent-request capacity — every request acquires a permit before proceeding and releases it in a `finally` block; a request that can't acquire a permit within a short timeout (`tryAcquire(timeout)`) is rejected fast (backpressure) rather than queuing indefinitely and risking cascading latency.
- **Connection pooling to the downstream service**: a **bounded** `BlockingQueue`-backed pool (or a library-provided pool built on the same primitives) — bounded specifically so a downstream slowdown can't cause unbounded connection/memory growth on the gateway side; `ArrayBlockingQueue` chosen over `LinkedBlockingQueue` here specifically *because* it's bounded by construction (a `LinkedBlockingQueue` can also be bounded via a constructor argument, but `ArrayBlockingQueue`'s fixed-array backing makes the bound structurally explicit and avoids any risk of accidentally constructing an unbounded one elsewhere in the codebase).
- **Handling the request-processing itself**: virtual threads for the gateway's request-handling layer (I/O-bound — waiting on the downstream call), **not** parallel streams (this isn't a batch, in-memory, CPU-bound computation problem — it's a live, concurrent-request-handling problem, exactly the profile Section 4.2's decision table routes to virtual threads, not Fork/Join-based parallelism).
- **Why `Semaphore` over just relying on the connection pool's own bound?** — Separating **rate limiting** (a policy decision, per-client, potentially independent of raw connection availability) from **connection pooling** (a resource-management concern) keeps the two concerns independently tunable — you might want to reject a client for exceeding *their own* rate limit even while the downstream service has spare connection capacity, and vice versa; conflating them into one mechanism removes that flexibility.
- **Trade-off explicitly stated**: `tryAcquire` with a short timeout **fails fast** rather than queuing every excess request — this deliberately favors overall system stability (bounded worst-case latency, predictable behavior under overload) over maximizing the number of requests eventually served, a classic backpressure trade-off worth naming as a design choice rather than leaving implicit.

---

# CLOSING — HOW TO DELIVER ANY PART OF THIS LIVE

For any topic in this document, structure the answer:
1. **What changed and when** (the historical anchor — "this was fixed/added in Java X").
2. **The internals mechanism** (Mark Word, Spliterator, CAS, per-bin locking — whatever's precise and specific).
3. **The performance "why"** (not just that it's faster, but the actual mechanical reason).
4. **A trade-off or caveat** (parallel streams' shared pool, virtual-thread pinning, `ConcurrentHashMap` overhead vs. plain `HashMap`) — this is what separates reciting a changelog from demonstrating engineering judgment.
5. **Tie it to a design scenario** if the conversation allows — Part 5's two scenarios are built specifically to be adaptable talking points for almost any "how would you design X" follow-up.
