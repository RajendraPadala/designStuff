# Design Patterns — The Complete Reference
### GoF Classics → Enterprise Patterns → Cloud-Native Patterns
### With Java Examples, When-to-Use/Avoid, and Interview Framing

---

## 0. WHAT A DESIGN PATTERN ACTUALLY IS (say this first in an interview)

A design pattern is a **named, reusable solution to a recurring design problem** — not a library you import, not a specific piece of code, but a *shape* of solution you recognize and adapt. The value of knowing them isn't reciting definitions — it's **vocabulary**: saying "we used the Strategy pattern here" instantly communicates an entire design shape to another engineer without re-explaining it from scratch. That's the single best line to open with if asked "why do design patterns matter."

They're traditionally grouped into three families from the 1994 "Gang of Four" (GoF) book:
- **Creational** — how objects get created.
- **Structural** — how objects/classes are composed into larger structures.
- **Behavioral** — how objects communicate and distribute responsibility.

Plus two families that matter enormously in modern backend work but predate/postdate GoF: **Enterprise patterns** (J2EE-era — DAO, DTO, Repository, MVC) and **Cloud-native/distributed patterns** (Circuit Breaker, Saga, Sidecar, Strangler Fig).

---

## PART A — CREATIONAL PATTERNS

### A.1 Singleton
**Intent**: ensure a class has exactly one instance, with a global access point to it.
```java
public class ConnectionPoolManager {
    private static volatile ConnectionPoolManager instance;
    private ConnectionPoolManager() { /* expensive setup */ }

    public static ConnectionPoolManager getInstance() {
        if (instance == null) {
            synchronized (ConnectionPoolManager.class) {
                if (instance == null) {          // double-checked locking
                    instance = new ConnectionPoolManager();
                }
            }
        }
        return instance;
    }
}
```
**Modern reality**: in Spring, you almost never hand-write this — every `@Component`/`@Service`/`@Bean` is a **singleton by default** within the `ApplicationContext`. Knowing *that* Spring's default bean scope is a container-managed Singleton (not the classic GoF static-instance version) is more interview-relevant than being able to write double-checked locking from memory.
**When to avoid**: Singletons introduce **global mutable state** and make unit testing harder (hidden dependencies, hard to substitute a mock) — prefer DI-managed singletons (Spring beans) over hand-rolled static ones whenever a framework is available.
**Interview angle**: be ready to explain *why* `volatile` and double-checked locking are both needed — without `volatile`, another thread could see a partially-constructed instance (a direct application of the JMM happens-before discussion from the Java internals material).

### A.2 Factory Method
**Intent**: define an interface for creating an object, but let subclasses decide which concrete class to instantiate.
```java
public interface NotificationFactory {
    Notification create(String type);
}
public class DefaultNotificationFactory implements NotificationFactory {
    public Notification create(String type) {
        return switch (type) {
            case "EMAIL" -> new EmailNotification();
            case "SMS"   -> new SmsNotification();
            default      -> throw new IllegalArgumentException("Unknown type: " + type);
        };
    }
}
```
**When to use**: when a class can't anticipate which concrete implementation it needs until runtime, and you want that decision centralized in one place rather than scattered `new` calls throughout the codebase.
**Real example from this conversation**: Spring's own `BeanFactory`/`ApplicationContext` is fundamentally a sophisticated Factory — you ask the container for a bean by type/name, and it decides how to construct it (including all its dependencies).

### A.3 Abstract Factory
**Intent**: provide an interface for creating **families of related objects** without specifying their concrete classes.
```java
public interface UIComponentFactory {
    Button createButton();
    Checkbox createCheckbox();
}
public class DarkThemeFactory implements UIComponentFactory {
    public Button createButton() { return new DarkButton(); }
    public Checkbox createCheckbox() { return new DarkCheckbox(); }
}
public class LightThemeFactory implements UIComponentFactory {
    public Button createButton() { return new LightButton(); }
    public Checkbox createCheckbox() { return new LightCheckbox(); }
}
```
**Distinction from Factory Method (a common interview trap)**: Factory Method creates **one** product; Abstract Factory creates a **family of related products** that need to stay consistent with each other (you'd never want a `DarkButton` paired with a `LightCheckbox`).

### A.4 Builder
**Intent**: construct a complex object step by step, separating construction from representation.
```java
public class HttpRequest {
    private final String url;
    private final String method;
    private final Map<String, String> headers;

    private HttpRequest(Builder b) {
        this.url = b.url; this.method = b.method; this.headers = b.headers;
    }

    public static class Builder {
        private String url;
        private String method = "GET";
        private Map<String, String> headers = new HashMap<>();

        public Builder url(String url) { this.url = url; return this; }
        public Builder method(String method) { this.method = method; return this; }
        public Builder header(String k, String v) { headers.put(k, v); return this; }
        public HttpRequest build() { return new HttpRequest(this); }
    }
}
// usage: new HttpRequest.Builder().url("...").method("POST").header("Auth", token).build();
```
**When to use**: many optional constructor parameters (avoids "telescoping constructors" — five overloaded constructors with increasing parameter counts). **Lombok's `@Builder`** generates exactly this boilerplate automatically — worth mentioning you know the codegen shortcut exists, not just the hand-written version.
**Note on Java records**: records give you a concise immutable data holder but **not** a builder — for many optional fields, you'd still reach for the Builder pattern (or Lombok `@Builder` on top of a record-like class) rather than a record's single canonical constructor.

### A.5 Prototype
**Intent**: create new objects by copying an existing instance (cloning) rather than instantiating from scratch.
```java
public class ConfigTemplate implements Cloneable {
    private Map<String, String> settings;
    @Override
    public ConfigTemplate clone() {
        ConfigTemplate copy = new ConfigTemplate();
        copy.settings = new HashMap<>(this.settings);  // deep copy the mutable field
        return copy;
    }
}
```
**When to use**: when object creation is expensive (complex initialization) and you need many similar-but-slightly-different instances — clone a pre-configured template and tweak it, rather than rebuilding from scratch each time.
**Watch for**: shallow vs. deep copy bugs — cloning a mutable field by reference (not copying its contents) means two "independent" clones actually share state, a classic subtle bug.

---

## PART B — STRUCTURAL PATTERNS

### B.1 Adapter
**Intent**: convert one interface into another that a client expects, letting incompatible interfaces work together.
```java
// Third-party library gives you this interface, you need to fit it into yours
public interface LegacyPaymentGateway {
    void makePayment(String accountId, double amountInCents);
}
public interface ModernPaymentProcessor {
    void process(PaymentRequest request);
}
public class LegacyPaymentAdapter implements ModernPaymentProcessor {
    private final LegacyPaymentGateway legacy;
    public LegacyPaymentAdapter(LegacyPaymentGateway legacy) { this.legacy = legacy; }
    public void process(PaymentRequest request) {
        legacy.makePayment(request.getAccountId(), request.getAmount().doubleValue() * 100);
    }
}
```
**Real example from this conversation**: the SOAP-to-REST migration discussion — an adapter layer let old SOAP consumers keep working while new REST endpoints were built, translating between the two contracts.

### B.2 Decorator
**Intent**: attach additional responsibilities to an object dynamically, without altering its class — an alternative to subclassing for extending behavior.
```java
public interface DataSource { void writeData(String data); }

public class FileDataSource implements DataSource {
    public void writeData(String data) { /* write to file */ }
}
public abstract class DataSourceDecorator implements DataSource {
    protected final DataSource wrapped;
    protected DataSourceDecorator(DataSource wrapped) { this.wrapped = wrapped; }
}
public class EncryptionDecorator extends DataSourceDecorator {
    public EncryptionDecorator(DataSource wrapped) { super(wrapped); }
    public void writeData(String data) { wrapped.writeData(encrypt(data)); }
    private String encrypt(String data) { return "ENC[" + data + "]"; }
}
public class CompressionDecorator extends DataSourceDecorator {
    public CompressionDecorator(DataSource wrapped) { super(wrapped); }
    public void writeData(String data) { wrapped.writeData(compress(data)); }
    private String compress(String data) { return "GZ[" + data + "]"; }
}
// usage: new EncryptionDecorator(new CompressionDecorator(new FileDataSource())).writeData("...")
```
**Real example from earlier in this conversation**: Envoy sidecar injection in the platform-engineering doc — the sidecar adds mTLS/retry/observability behavior *around* a pod's existing container, without modifying the application's own image. Java I/O streams (`new BufferedReader(new InputStreamReader(new FileInputStream(...)))`) are the textbook JDK example.

### B.3 Facade
**Intent**: provide a single, simplified interface to a complex subsystem.
```java
public class OrderFacade {
    private final InventoryService inventory;
    private final PaymentService payment;
    private final ShippingService shipping;

    public void placeOrder(Order order) {
        inventory.reserve(order.getItems());
        payment.charge(order.getPaymentDetails());
        shipping.schedule(order);
    }
}
```
**Real example from this conversation**: `RoutingRuleEngine` in the CARE deep-dive — one clean entry point hiding DAO/rule-repository complexity from both the batch layer and the UI-facing service layer.

### B.4 Proxy
**Intent**: provide a surrogate/placeholder for another object to control access to it (lazy loading, access control, logging, remote invocation).
```java
public interface ImageLoader { void display(); }

public class RealImage implements ImageLoader {
    public RealImage(String path) { loadFromDisk(path); }  // expensive
    public void display() { /* render */ }
}
public class LazyImageProxy implements ImageLoader {
    private RealImage real;
    private final String path;
    public LazyImageProxy(String path) { this.path = path; }
    public void display() {
        if (real == null) real = new RealImage(path);  // load only when actually needed
        real.display();
    }
}
```
**Real example from this conversation**: Hibernate's **lazy-loaded proxy** (`fetch="select" lazy="proxy"` from the CARE mapping example) is a textbook Proxy pattern instance — you hold a proxy object standing in for the real entity, and the real DB query only fires when you actually touch the proxy's fields.
**Also**: Spring AOP's `@Transactional`/`@PreAuthorize` are implemented via dynamically-generated **proxies** wrapping your bean — a very high-value fact to know precisely, since it's *why* calling an `@Transactional` method from **within the same class** doesn't get transaction behavior (the call bypasses the proxy entirely, going directly to the real object) — a classic, very-frequently-asked Spring gotcha.

### B.5 Composite
**Intent**: compose objects into tree structures to represent part-whole hierarchies, letting clients treat individual objects and compositions uniformly.
```java
public interface FileSystemNode { long getSize(); }

public class File implements FileSystemNode {
    private final long size;
    public long getSize() { return size; }
}
public class Directory implements FileSystemNode {
    private final List<FileSystemNode> children = new ArrayList<>();
    public long getSize() {
        return children.stream().mapToLong(FileSystemNode::getSize).sum();
    }
    public void add(FileSystemNode node) { children.add(node); }
}
```
**When to use**: whenever you have a tree of "leaf" and "container" objects that should support the same operations uniformly (an org chart, a UI component tree, a file system).

### B.6 Bridge
**Intent**: decouple an abstraction from its implementation so the two can vary independently.
```java
public interface Renderer { void renderCircle(float radius); }
public class VectorRenderer implements Renderer { public void renderCircle(float r) { /* SVG */ } }
public class RasterRenderer implements Renderer { public void renderCircle(float r) { /* pixels */ } }

public abstract class Shape {
    protected Renderer renderer;
    protected Shape(Renderer renderer) { this.renderer = renderer; }
    public abstract void draw();
}
public class Circle extends Shape {
    private float radius;
    public Circle(Renderer renderer, float radius) { super(renderer); this.radius = radius; }
    public void draw() { renderer.renderCircle(radius); }
}
```
**Distinction from Adapter (interview trap)**: Adapter is applied **after the fact** to make incompatible interfaces work together; Bridge is designed **up front** so an abstraction hierarchy and an implementation hierarchy can each grow independently without a combinatorial explosion of subclasses.

### B.7 Flyweight
**Intent**: minimize memory usage by sharing as much data as possible with similar objects, separating **intrinsic** (shared, immutable) state from **extrinsic** (context-specific) state.
```java
public class CharacterGlyph {  // intrinsic: shared, immutable
    private final char symbol;
    private final Font font;
}
public class GlyphFactory {
    private static final Map<String, CharacterGlyph> cache = new HashMap<>();
    public static CharacterGlyph get(char symbol, Font font) {
        String key = symbol + "_" + font;
        return cache.computeIfAbsent(key, k -> new CharacterGlyph(symbol, font));
    }
}
```
**Real-world example**: `Integer.valueOf()` caches small integer values (-128 to 127) internally rather than allocating a new object every time — a JDK-native Flyweight.

---

## PART C — BEHAVIORAL PATTERNS

### C.1 Strategy
**Intent**: define a family of interchangeable algorithms, encapsulated behind a common interface, selectable at runtime.
```java
public interface RoutingRule {
    boolean matches(Account account);
    RoutingOutcome apply(Account account);
}
public class BehaviorScoreThresholdRule implements RoutingRule { /* ... */ }
public class HighBalanceEscalationRule implements RoutingRule { /* ... */ }
```
**This is directly the pattern behind the CARE rule-engine deep-dive earlier in this conversation** — each rule is an interchangeable strategy, and the engine doesn't know or care about any rule's internals, only that it can call `matches()`/`apply()`.
**Distinction from State (a common interview trap)**: Strategy is chosen **by the client** based on external criteria; State is chosen **by the object itself** based on its own internal condition, and the object's behavior changes as its state changes.

### C.2 Observer
**Intent**: define a one-to-many dependency so that when one object changes state, all its dependents are notified automatically.
```java
public interface OrderObserver { void onOrderPlaced(Order order); }

public class OrderService {
    private final List<OrderObserver> observers = new ArrayList<>();
    public void subscribe(OrderObserver o) { observers.add(o); }
    public void placeOrder(Order order) {
        // ... persist order ...
        observers.forEach(o -> o.onOrderPlaced(order));
    }
}
```
**Real example from this conversation**: Spring's `ApplicationEventPublisher`/`@EventListener` is a first-class Observer implementation; at a larger scale, **Kafka pub-sub** is the distributed-systems version of the exact same idea — decouple the publisher from knowing how many/which consumers exist.

### C.3 Template Method
**Intent**: define the skeleton of an algorithm in a base class, deferring specific steps to subclasses.
```java
public abstract class DataImportJob {
    public final void run() {          // the "template" -- fixed skeleton
        List<String> raw = readSource();
        List<Record> validated = validate(raw);
        persist(validated);
        notifyCompletion();
    }
    protected abstract List<String> readSource();
    protected abstract List<Record> validate(List<String> raw);
    protected void persist(List<Record> records) { /* default impl, overridable */ }
    private void notifyCompletion() { /* shared, not overridable */ }
}
```
**This is exactly Spring Batch's chunk-oriented `Step`** (from the MCO and CARE deep-dives) — the framework owns the read→process→write→commit loop (the template), and your `ItemReader`/`ItemProcessor`/`ItemWriter` implementations supply the specific steps.

### C.4 Chain of Responsibility
**Intent**: pass a request along a chain of handlers until one handles it.
```java
public abstract class Middleware {
    protected Middleware next;
    public Middleware linkWith(Middleware next) { this.next = next; return next; }
    public abstract boolean check(Request request);
    protected boolean checkNext(Request request) {
        return next == null || next.check(request);
    }
}
public class AuthCheck extends Middleware {
    public boolean check(Request request) {
        if (!request.isAuthenticated()) return false;
        return checkNext(request);
    }
}
```
**Real example from this conversation**: **Spring Security's filter chain** and **Envoy's filter chain** (from the platform-engineering doc) are both Chain of Responsibility — an ordered sequence of handlers, each free to short-circuit and stop the chain.

### C.5 Command
**Intent**: encapsulate a request as an object, letting you parameterize clients with different requests, queue them, or support undo.
```java
public interface Command { void execute(); void undo(); }

public class AddTextCommand implements Command {
    private final Document doc; private final String text;
    public void execute() { doc.append(text); }
    public void undo() { doc.removeLast(text.length()); }
}
public class CommandInvoker {
    private final Deque<Command> history = new ArrayDeque<>();
    public void execute(Command cmd) { cmd.execute(); history.push(cmd); }
    public void undoLast() { if (!history.isEmpty()) history.pop().undo(); }
}
```
**Real-world example**: a Jenkins pipeline stage, or a Kafka message payload representing "do this specific thing" — the request itself becomes a first-class, queueable, replayable object, which is conceptually the same idea Command formalizes at the OOP level.

### C.6 State
**Intent**: allow an object to alter its behavior when its internal state changes, appearing as if it changed class.
```java
public interface OrderState {
    void next(OrderContext ctx);
    String name();
}
public class PendingState implements OrderState {
    public void next(OrderContext ctx) { ctx.setState(new ShippedState()); }
    public String name() { return "PENDING"; }
}
public class ShippedState implements OrderState {
    public void next(OrderContext ctx) { ctx.setState(new DeliveredState()); }
    public String name() { return "SHIPPED"; }
}
```
**Real example from this conversation**: the Chartink monitor's `NEW`/`ACTIVE`/`DISAPPEARED` stock states, and CARE's `NEW → ROUTED → IN-COLLECTION → RESOLVED` account lifecycle — both are State-pattern-shaped, even when implemented as a simple enum + transition logic rather than a full class-per-state hierarchy (a very common, perfectly reasonable simplification in real code).

### C.7 Iterator
**Intent**: provide a way to access elements of a collection sequentially without exposing its underlying representation.
Java's own `Iterable`/`Iterator` interfaces (powering every `for (X x : collection)` loop) **are** this pattern, built into the language — worth stating plainly that you use this pattern dozens of times a day without thinking about it, which is itself a good answer to "give me an example of a pattern you use constantly."

### C.8 Mediator
**Intent**: define an object that encapsulates how a set of objects interact, promoting loose coupling by keeping objects from referring to each other directly.
```java
public interface ChatMediator { void sendMessage(String msg, User sender); }

public class ChatRoom implements ChatMediator {
    private final List<User> users = new ArrayList<>();
    public void sendMessage(String msg, User sender) {
        users.stream().filter(u -> u != sender).forEach(u -> u.receive(msg));
    }
}
```
**Real example**: an **API Gateway** is a Mediator at the distributed-systems scale — services don't call each other directly; they go through the gateway, which centralizes the interaction logic.

### C.9 Memento
**Intent**: capture and externalize an object's internal state so it can be restored later, without violating encapsulation.
```java
public class EditorMemento {
    private final String content;
    EditorMemento(String content) { this.content = content; }
    String getContent() { return content; }
}
public class Editor {
    private String content = "";
    public EditorMemento save() { return new EditorMemento(content); }
    public void restore(EditorMemento m) { this.content = m.getContent(); }
}
```
**Real example**: a database transaction's rollback capability, or Kubernetes' declarative "desired state" reconciliation — snapshotting state to return to later is the core idea.

### C.10 Visitor
**Intent**: represent an operation to be performed on elements of an object structure, letting you define a new operation without changing the classes of the elements it operates on.
```java
public interface ShapeVisitor { void visit(Circle c); void visit(Square s); }
public interface Shape { void accept(ShapeVisitor v); }
public class Circle implements Shape {
    public void accept(ShapeVisitor v) { v.visit(this); }
}
public class AreaCalculator implements ShapeVisitor {
    public void visit(Circle c) { /* compute circle area */ }
    public void visit(Square s) { /* compute square area */ }
}
```
**When to use**: rare in typical CRUD backend work, but shows up in compilers/ASTs and serialization frameworks. **Java's pattern matching over sealed types (`switch` with exhaustiveness checking, from the Java internals material) increasingly replaces the classical Visitor pattern** for exactly this use case in modern Java — a good, precise thing to say if asked about Visitor: "modern Java's sealed classes + pattern matching largely subsume what Visitor used to be needed for."

---

## PART D — ENTERPRISE PATTERNS (pre-dating/alongside GoF, J2EE-era, still everywhere in backend Java)

### D.1 DAO (Data Access Object)
Encapsulates persistence logic behind an interface, isolating the rest of the application from the specific persistence technology. Covered in full depth (with the **Generic DAO** variant) in the CARE and 360° documents earlier in this conversation — Spring Data JPA's `JpaRepository<T, ID>` is the industrialized, auto-implemented modern descendant of this exact pattern.

### D.2 DTO (Data Transfer Object)
A plain object carrying data across a process/layer boundary (e.g., over a REST API), deliberately decoupled from the persistence entity — covered extensively throughout this conversation's REST API design sections.

### D.3 Repository
Similar in spirit to DAO but framed at a slightly higher, more domain-oriented abstraction level (Domain-Driven Design's term) — a Repository presents a collection-like interface over an aggregate root, hiding not just *how* data is persisted but framing access in domain terms (`findActiveOrdersForCustomer(customerId)` rather than a generic `find(id)`).

### D.4 MVC (Model-View-Controller)
Separates an application into Model (data/business logic), View (presentation), and Controller (coordinates between the two, handles input) — Spring MVC's `@Controller`/`@Service`/JSP-or-JSON-view structure is a direct implementation, and the layered-architecture discussion throughout this conversation (Controller → Service → Repository) is essentially MVC's principles applied to a REST API context (no literal "View" beyond the serialized JSON response).

### D.5 Dependency Injection / Inversion of Control
Technically an architectural principle more than a single "pattern," but always covered alongside these — covered in depth in the Java Ecosystem document (Section 1: Spring Core & IoC). The single most foundational pattern underlying everything Spring does.

---

## PART E — CLOUD-NATIVE / DISTRIBUTED SYSTEMS PATTERNS

These are covered in full depth with real project examples in the Platform Engineering documents from earlier in this conversation — summarized here for completeness, cross-referenced to where you can find the deep dive:

| Pattern | One-line intent | Where covered in depth |
|---|---|---|
| **Circuit Breaker** | Stop calling a failing dependency, fail fast, allow recovery time | Java Ecosystem doc, Section on Resilience4j |
| **Bulkhead** | Isolate resource pools so one failure can't exhaust shared capacity | Architecture/Patterns doc, Part B.2 |
| **Saga** | Multi-step distributed "transaction" via compensating actions, since no real cross-system atomic commit exists | Architecture/Patterns doc, Part C.6 |
| **Sidecar** | Cross-cutting infra concerns (mTLS, logging, tracing) live in a co-located helper process, not app code | Platform Engineering docs |
| **Strangler Fig** | Incrementally migrate a legacy system by routing new functionality to the new system while the old one still runs | CARE deep-dive; Architecture/Patterns doc |
| **API Gateway** | Single external entry point, decoupling internal service topology from external contract | Platform Engineering docs |
| **Outbox Pattern** | Write a DB change and its corresponding event in one local transaction, relay the event separately, avoiding the dual-write problem | Architecture/Patterns doc, Part B.3 |
| **Idempotent Receiver** | Safely handle duplicate message delivery (at-least-once guarantees) | MCO deep-dive; Java Ecosystem doc |
| **CQRS** (Command Query Responsibility Segregation) | Separate the model/path used for writes from the one used for reads, when their scaling/shape needs genuinely diverge | Worth knowing exists; not deeply used in most of your documented projects |
| **Event Sourcing** | Persist a sequence of events as the source of truth, deriving current state by replaying them, rather than only storing current state | MCO deep-dive (effective-dated history tables are a lightweight, partial application of this idea) |

---

## PART F — HOW TO ACTUALLY USE THIS IN AN INTERVIEW

1. **Never just define a pattern** — name it, then immediately point to a real (or realistic) place you used it. Every pattern above has a cross-referenced real example from this conversation's project material; use those.
2. **Know the adjacent trap-pattern distinctions** — Factory Method vs. Abstract Factory, Strategy vs. State, Adapter vs. Bridge, Decorator vs. Proxy (Decorator *adds* behavior, Proxy *controls access* — subtly different intents even though structurally similar). Interviewers love these pairs specifically because confusing them reveals memorization over understanding.
3. **Say what a pattern costs, not just what it buys** — Singleton costs testability; Decorator costs a deeper call stack and more classes; Observer costs harder-to-trace control flow (an event fires, and you have to go find every listener to understand full behavior). A senior answer always names the trade-off.
4. **Connect patterns across layers** — the best answers show the *same* pattern recurring at different scales (Chain of Responsibility as both a Java class hierarchy *and* Spring Security's filter chain *and* Envoy's filter chain) — that's the strongest possible signal of real, transferable understanding rather than memorized definitions.
