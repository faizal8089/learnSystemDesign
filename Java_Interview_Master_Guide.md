# Java Interview Master Guide
> Covers every concept from your company's interview question bank. Use this as your go-to reference before any round.

---

## TABLE OF CONTENTS

1. [OOP Concepts](#1-oop-concepts)
2. [Core Java & Keywords](#2-core-java--keywords)
3. [Collections & Data Structures](#3-collections--data-structures)
4. [Multi-threading & Concurrency](#4-multi-threading--concurrency)
5. [JVM Architecture, Memory & GC](#5-jvm-architecture-memory--gc)
6. [Java 8 Features](#6-java-8-features)
7. [Exception Handling](#7-exception-handling)
8. [Design Patterns & SOLID](#8-design-patterns--solid)
9. [Spring Boot & Spring Framework](#9-spring-boot--spring-framework)
10. [Microservices Architecture](#10-microservices-architecture)
11. [Kafka](#11-kafka)
12. [Database & SQL](#12-database--sql)
13. [REST & SOAP](#13-rest--soap)
14. [CI/CD, Docker & Kubernetes](#14-cicd-docker--kubernetes)
15. [Testing](#15-testing)
16. [DSA & Sorting](#16-dsa--sorting)
17. [Code Review Checklist](#17-code-review-checklist)
18. [Miscellaneous & Scenario Questions](#18-miscellaneous--scenario-questions)

---

## 1. OOP CONCEPTS

### Encapsulation
Wrapping data (fields) and methods inside a class, exposing only what is necessary via getters/setters.

```java
public class Employee {
    private String name;  // hidden
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
}
```
**Benefit:** Controls how data is accessed and modified.

---

### Abstraction
Hiding implementation details and exposing only the interface/contract.

```java
abstract class Shape {
    abstract double area(); // no implementation
}

class Circle extends Shape {
    double r;
    Circle(double r) { this.r = r; }
    double area() { return Math.PI * r * r; }
}
```
**Abstract class vs Interface:**
| Feature | Abstract Class | Interface |
|---|---|---|
| Can have constructors | Yes | No |
| Can have instance fields | Yes | No (only constants) |
| Multiple inheritance | No | Yes |
| Default methods | Yes (from Java 8) | Yes (from Java 8) |

---

### Inheritance
A child class acquires properties and behavior of a parent class.

```java
class Animal {
    void speak() { System.out.println("..."); }
}

class Dog extends Animal {
    @Override
    void speak() { System.out.println("Woof!"); }
}
```

**Multiple Inheritance via Interface:**
```java
interface Flyable { default void fly() { System.out.println("Flying"); } }
interface Swimmable { default void swim() { System.out.println("Swimming"); } }

class Duck implements Flyable, Swimmable { }

// If both interfaces have same default method, you MUST override it
```

---

### Polymorphism
**Compile-time (Overloading):** Same method name, different parameters.
```java
class Math {
    int add(int a, int b) { return a + b; }
    double add(double a, double b) { return a + b; } // overloaded
}
```

**Runtime (Overriding):** Child class redefines parent method.
```java
Animal a = new Dog(); // reference is Animal, object is Dog
a.speak(); // calls Dog's speak() — decided at runtime
```

> **Private methods CANNOT be overridden** — they are not inherited. Static methods are hidden, not overridden.

---

### Covariant Return Type
A child class can override a method and return a more specific type.
```java
class Animal { Animal get() { return new Animal(); } }
class Dog extends Animal { 
    @Override
    Dog get() { return new Dog(); } // Dog is a subtype of Animal — valid!
}
```

---

## 2. CORE JAVA & KEYWORDS

### volatile
Ensures a variable's value is always read from and written to **main memory**, not CPU cache. Used in multi-threaded environments.

```java
class Singleton {
    private static volatile Singleton instance;
    // Without volatile, partial initialization can be seen by other threads
}
```
- **Can you store an array in volatile?** Yes — `volatile int[] arr` makes the **reference** volatile, not the array elements.

---

### synchronized
Prevents multiple threads from executing a block/method simultaneously.

```java
// Method synchronization (locks on 'this')
public synchronized void increment() { count++; }

// Block synchronization (finer control — preferred)
public void increment() {
    synchronized(this) { count++; }
}
```

**Static vs Non-static synchronization:**
- Non-static synchronized → locks on **instance** (`this`)
- Static synchronized → locks on **Class object** (`MyClass.class`)

---

### transient
Marks a field to be excluded from serialization.
```java
class User implements Serializable {
    String name;
    transient String password; // won't be serialized
}
```

---

### Immutable Object
An object whose state cannot change after creation.

```java
public final class Money {          // 1. final class
    private final int amount;       // 2. final fields
    public Money(int amount) { this.amount = amount; }
    public int getAmount() { return amount; } // 3. no setters
    // 4. if field is mutable object, return a copy
}
```
**Benefits:** Thread-safe, cacheable, safe for use in HashMap keys.
**Examples in JDK:** `String`, `Integer`, `LocalDate`

---

### static keyword
- `static` field → shared across all instances
- `static` method → can be called without creating an object
- `static` block → runs once when class is loaded

**Code before `main()` using static block:**
```java
public class Demo {
    static {
        System.out.println("Runs before main!"); // ← this prints first
    }
    public static void main(String[] args) {
        System.out.println("Main method");
    }
}
```

---

### Breaking a Singleton (and how to prevent it)
```java
// 1. Via Reflection
Constructor<Singleton> c = Singleton.class.getDeclaredConstructor();
c.setAccessible(true);
Singleton s2 = c.newInstance(); // breaks singleton!

// Prevention: throw exception in constructor if instance already exists
private Singleton() {
    if (instance != null) throw new RuntimeException("Use getInstance()");
}

// 2. Via Serialization — use readResolve()
protected Object readResolve() { return instance; }

// 3. Safest approach: Enum Singleton (immune to reflection & serialization)
public enum Singleton { INSTANCE; }
```

---

### JIT (Just-In-Time Compiler)
| | Normal Compiler | JIT |
|---|---|---|
| When | Before execution | During execution (runtime) |
| Input | Source code | Bytecode |
| Output | Machine code | Native machine code |
| Benefit | — | Optimizes **hot** (frequently run) code paths |

JIT is part of the JVM. It compiles bytecode to native code at runtime for performance.

---

### What is `public static void main(String[] args)`?
| Part | Meaning |
|---|---|
| `public` | Accessible from JVM anywhere |
| `static` | JVM can call without creating an object |
| `void` | Returns nothing to JVM |
| `main` | Entry point name JVM looks for |
| `String[] args` | Command-line arguments |

---

### Transitive Dependency Conflict
When two libraries depend on different versions of the same library.

**Maven resolution:** nearest-wins (closest to your pom.xml wins).

**Force a specific version:**
```xml
<dependency>
    <groupId>com.google.guava</groupId>
    <artifactId>guava</artifactId>
    <version>31.0-jre</version> <!-- explicitly pinned -->
</dependency>
```

**Exclude a transitive dep:**
```xml
<dependency>
    <groupId>some.lib</groupId>
    <artifactId>some-lib</artifactId>
    <exclusions>
        <exclusion>
            <groupId>com.google.guava</groupId>
            <artifactId>guava</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```

---

## 3. COLLECTIONS & DATA STRUCTURES

### HashMap Internals
- Backed by an array of `Node<K,V>[]` (buckets)
- Uses `hashCode()` to find bucket index: `index = hash & (n-1)`
- Uses `equals()` to handle key collisions within the same bucket
- Default capacity: 16, load factor: 0.75 → resizes at 12 entries

**Hash Collision:** Two keys land in the same bucket → stored as a linked list (or tree if > 8 entries from Java 8).

**Time Complexity:**
| Operation | Average | Worst (all collide) |
|---|---|---|
| get / put | O(1) | O(n) |
| HashSet operations | O(1) | O(n) |

> **Why use HashSet?** O(1) lookup, no duplicates, backed by HashMap internally.

---

### HashMap vs ConcurrentHashMap
| | HashMap | ConcurrentHashMap |
|---|---|---|
| Thread-safe | No | Yes |
| Null keys/values | 1 null key allowed | No null keys/values |
| Locking | None | Segment/bucket-level locking |
| Performance | Fastest (single thread) | Fast for concurrent access |

---

### TreeMap vs HashMap vs LinkedHashMap
| | HashMap | TreeMap | LinkedHashMap |
|---|---|---|---|
| Order | None | Sorted by key | Insertion order |
| Null key | Yes (1) | No | Yes (1) |
| Time complexity | O(1) | O(log n) | O(1) |

---

### PriorityQueue
Min-heap by default. Use `Collections.reverseOrder()` for max-heap.
```java
PriorityQueue<Integer> pq = new PriorityQueue<>(); // min-heap
pq.add(5); pq.add(1); pq.add(3);
System.out.println(pq.poll()); // 1 — smallest first
```

---

### Comparable vs Comparator
Use **Comparable** when you own the class and define natural ordering.  
Use **Comparator** for external/multiple sort strategies.

```java
// Comparable — natural order
class Item implements Comparable<Item> {
    String category;
    @Override
    public int compareTo(Item other) {
        return this.category.compareTo(other.category);
    }
}

// Comparator — custom order
List<Item> items = ...;
items.sort(Comparator.comparing(i -> i.category));
// OR
items.sort((a, b) -> a.category.compareTo(b.category));
```

---

### LinkedList — Add at Beginning
```java
LinkedList<Integer> list = new LinkedList<>();
list.addFirst(10);  // adds to head
list.addFirst(20);  // 20 → 10
```

---

### Hashcode & equals Contract
- If `a.equals(b)` is true, then `a.hashCode() == b.hashCode()` **must** be true.
- Override both together — if only `equals` is overridden, HashMap/HashSet breaks.

---

## 4. MULTI-THREADING & CONCURRENCY

### Thread Creation
```java
// 1. Extend Thread
class MyThread extends Thread {
    public void run() { System.out.println("Thread running"); }
}
new MyThread().start();

// 2. Implement Runnable (preferred — allows extending other classes)
Thread t = new Thread(() -> System.out.println("Runnable thread"));
t.start();

// 3. Callable (returns a value, can throw checked exception)
Callable<Integer> task = () -> 42;
FutureTask<Integer> future = new FutureTask<>(task);
new Thread(future).start();
System.out.println(future.get()); // 42
```

---

### Runnable vs Callable vs ExecutorService
| | Runnable | Callable |
|---|---|---|
| Return value | No (`void`) | Yes (via `Future<T>`) |
| Throws checked exception | No | Yes |

```java
// ExecutorService — thread pool
ExecutorService pool = Executors.newFixedThreadPool(4);
Future<Integer> result = pool.submit(() -> 100); // Callable
pool.execute(() -> System.out.println("fire and forget")); // Runnable
pool.shutdown();
```

---

### Thread Pool — Print 1 to 100
```java
ExecutorService pool = Executors.newFixedThreadPool(5);
for (int i = 1; i <= 100; i++) {
    final int num = i;
    pool.submit(() -> System.out.println(num));
}
pool.shutdown();
```

---

### Deadlock
Two or more threads waiting for each other's locks indefinitely.

```java
Object lock1 = new Object();
Object lock2 = new Object();

Thread t1 = new Thread(() -> {
    synchronized (lock1) {
        synchronized (lock2) { System.out.println("T1"); }
    }
});

Thread t2 = new Thread(() -> {
    synchronized (lock2) {           // ← reverse order causes deadlock
        synchronized (lock1) { System.out.println("T2"); }
    }
});
t1.start(); t2.start();
```

**Resolution:**
- Always acquire locks in the same order
- Use `tryLock()` with timeout (`ReentrantLock`)
- Minimize synchronized scope

---

### Daemon Thread
Background thread that doesn't prevent JVM from exiting. GC is a daemon thread.
```java
Thread t = new Thread(() -> { while(true) {} });
t.setDaemon(true); // must be set BEFORE start()
t.start();
// JVM exits even though t is running
```

---

### Maximum Number of Threads
There's no fixed JVM limit — constrained by OS, available memory, and stack size.  
Rule of thumb: Each thread uses ~256KB–1MB stack. With 4GB heap, you might get ~4000 threads.  
**Use thread pools — never create unbounded threads.**

---

### wait/notify (wait-less locking alternative)
```java
// Producer-Consumer pattern
synchronized(queue) {
    while (queue.isEmpty()) queue.wait();   // releases lock, waits
    queue.notify(); // wakes one waiting thread
}
```

---

### Atomic Variables
Lock-free thread safety for single variables using CAS (Compare-And-Swap).
```java
AtomicInteger counter = new AtomicInteger(0);
counter.incrementAndGet(); // thread-safe, no synchronized needed
```

---

### Set Internal Implementation
- `HashSet` → backed by `HashMap` (keys are set elements, value is a dummy object)
- `TreeSet` → backed by `TreeMap` → sorted, `O(log n)`
- `LinkedHashSet` → backed by `LinkedHashMap` → insertion order

---

## 5. JVM ARCHITECTURE, MEMORY & GC

### JVM Architecture
```
Source (.java) → Compiler (javac) → Bytecode (.class) → JVM
                                                          │
                              ┌───────────────────────────┤
                              │        Class Loader        │
                              │  ┌─────────────────────┐  │
                              │  │    Runtime Memory    │  │
                              │  │  Heap | Stack | etc  │  │
                              │  └─────────────────────┘  │
                              │  Execution Engine (JIT+GC) │
                              └───────────────────────────┘
```

---

### Class Loaders (Parent Delegation Model)
| Loader | Loads |
|---|---|
| Bootstrap | `java.lang.*`, core JDK classes (from `rt.jar`) |
| Extension (Platform) | `javax.*`, security extensions (`ext/` folder) |
| Application (System) | Your app's classpath classes |
| Custom | You write it for hot-reloading, plugins, etc. |

**Parent delegation:** Child asks parent first → prevents duplicate/malicious loading.

---

### JVM Memory Areas
| Area | Per Thread? | Stores |
|---|---|---|
| Heap | No (shared) | Objects, instance variables |
| Stack | Yes | Method frames, local variables, references |
| Method Area (Metaspace) | No (shared) | Class metadata, static fields |
| PC Register | Yes | Current instruction pointer |
| Native Method Stack | Yes | Native (JNI) method calls |

---

### Heap Memory Structure (Pre Java 8)
```
Heap
├── Young Generation
│   ├── Eden Space      ← new objects born here
│   ├── Survivor S0     ← survives 1 GC
│   └── Survivor S1     ← survives 2 GC
└── Old Generation (Tenured) ← long-lived objects
```

**Java 8+:** PermGen removed → replaced by **Metaspace** (grows dynamically in native memory).

---

### Garbage Collection Types
| GC | How | Best For |
|---|---|---|
| Serial GC | Single thread, stop-the-world | Small/single-core apps |
| Parallel GC | Multiple threads, stop-the-world | Throughput-focused |
| CMS (Concurrent Mark Sweep) | Concurrent, low pause | Low-latency apps |
| G1 GC (default Java 9+) | Splits heap into regions, predictable pauses | Most modern apps |
| ZGC / Shenandoah | Ultra-low pause (<10ms) | Very large heaps |

**Out of Memory Error:**
```java
List<byte[]> list = new ArrayList<>();
while (true) list.add(new byte[1024 * 1024]); // → java.lang.OutOfMemoryError: Java heap space
```

---

## 6. JAVA 8 FEATURES

### Stream API
```java
List<Integer> nums = Arrays.asList(1, 2, 3, 4, 5);

// filter + map + collect
List<Integer> evens = nums.stream()
    .filter(n -> n % 2 == 0)
    .map(n -> n * 2)
    .collect(Collectors.toList()); // [4, 8]

// reduce
int sum = nums.stream().reduce(0, Integer::sum); // 15

// sorted, distinct, limit
nums.stream().sorted().distinct().limit(3).forEach(System.out::println);
```

---

### flatMap
Flattens a stream of collections into a single stream.
```java
List<List<Integer>> nested = Arrays.asList(
    Arrays.asList(1, 2),
    Arrays.asList(3, 4)
);
List<Integer> flat = nested.stream()
    .flatMap(Collection::stream)
    .collect(Collectors.toList()); // [1, 2, 3, 4]
```

---

### Functional Interface
Interface with exactly one abstract method. Can use `@FunctionalInterface` annotation.
```java
@FunctionalInterface
interface Greet {
    void sayHello(String name);
}

Greet g = name -> System.out.println("Hello " + name);
g.sayHello("Faizal");
```

**Built-in functional interfaces:**
| Interface | Method | Purpose |
|---|---|---|
| `Predicate<T>` | `test(T t)` | Returns boolean |
| `Function<T,R>` | `apply(T t)` | Transform T to R |
| `Consumer<T>` | `accept(T t)` | Use T, return nothing |
| `Supplier<T>` | `get()` | Produce T |

---

### Interface vs Abstract Class vs Functional Interface
| | Interface | Abstract Class | Functional Interface |
|---|---|---|---|
| Multiple inheritance | Yes | No | Yes |
| Constructor | No | Yes | No |
| State (fields) | No (only constants) | Yes | No |
| Purpose | Contract | Partial implementation | Lambda target |

---

### Advantages/Disadvantages of Java 8
**Advantages:** Lambdas reduce boilerplate, Streams for pipeline processing, Optional eliminates NPE, default methods enable backward-compatible API evolution.  
**Disadvantages:** Streams can be harder to debug, overuse of lambdas hurts readability, parallel streams have overhead.

---

## 7. EXCEPTION HANDLING

### Types
```
Throwable
├── Error           (don't catch — StackOverflowError, OutOfMemoryError)
└── Exception
    ├── Checked     (must handle — IOException, SQLException)
    └── Unchecked   (RuntimeException — NullPointerException, ArrayIndexOutOfBoundsException)
```

### Handling
```java
try {
    int result = 10 / 0;
} catch (ArithmeticException e) {
    System.out.println("Caught: " + e.getMessage());
} catch (Exception e) {
    System.out.println("General: " + e.getMessage());
} finally {
    System.out.println("Always runs");
}
```

### Custom Exception
```java
public class InsufficientFundsException extends RuntimeException {
    public InsufficientFundsException(String msg) { super(msg); }
}
```

---

## 8. DESIGN PATTERNS & SOLID

### SOLID Principles
| Principle | Meaning | Example |
|---|---|---|
| **S** - Single Responsibility | One class, one reason to change | `UserService` only handles user logic, not email sending |
| **O** - Open/Closed | Open for extension, closed for modification | Add new payment type by extending, not editing existing code |
| **L** - Liskov Substitution | Child class must replace parent without breaking behavior | `Square extends Rectangle` violates if it overrides setWidth |
| **I** - Interface Segregation | Don't force clients to implement unused methods | Split `Animal` into `Flyable`, `Swimmable` etc. |
| **D** - Dependency Inversion | Depend on abstractions, not concretions | Inject `PaymentGateway` interface, not `StripeService` directly |

---

### Singleton Pattern
```java
public class Singleton {
    private static volatile Singleton instance;
    private Singleton() {}
    
    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) {           // double-checked locking
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

---

### Factory Pattern
```java
interface Shape { void draw(); }
class Circle implements Shape { public void draw() { System.out.println("Circle"); } }
class Square implements Shape { public void draw() { System.out.println("Square"); } }

class ShapeFactory {
    public static Shape getShape(String type) {
        return switch (type) {
            case "circle" -> new Circle();
            case "square" -> new Square();
            default -> throw new IllegalArgumentException("Unknown: " + type);
        };
    }
}
```

---

### Circuit Breaker Pattern
Prevents cascading failures in distributed systems by "tripping" when failures exceed a threshold.

**States:**
- **CLOSED** → Normal. Requests pass through.
- **OPEN** → Too many failures. Requests fail fast (no downstream call).
- **HALF-OPEN** → Test request allowed. If it succeeds, go to CLOSED.

**Frameworks:** Resilience4j (recommended for Spring Boot), Hystrix (deprecated).

```java
// Resilience4j example
@CircuitBreaker(name = "inventoryService", fallbackMethod = "fallback")
public String callInventory() {
    return restTemplate.getForObject("http://inventory/items", String.class);
}
public String fallback(Exception ex) {
    return "Service temporarily unavailable";
}
```

---

### Dependency Injection & IoC
**Inversion of Control (IoC):** Framework controls object creation (not your code).  
**Dependency Injection (DI):** IoC container injects dependencies into your class.

```java
// Without DI
class OrderService {
    PaymentService p = new PaymentService(); // tightly coupled
}

// With DI (Spring)
@Service
class OrderService {
    private final PaymentService paymentService;
    
    @Autowired  // injected by Spring
    OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
```

**Constructor vs Setter injection:**
- **Constructor injection** → Mandatory dependencies, immutable, preferred
- **Setter injection** → Optional dependencies, allows changing later

---

## 9. SPRING BOOT & SPRING FRAMEWORK

### Spring vs Spring Boot
| | Spring | Spring Boot |
|---|---|---|
| Configuration | Manual XML or Java config | Auto-configuration |
| Server | External (Tomcat WAR) | Embedded Tomcat/Jetty |
| Starter POMs | No | Yes (`spring-boot-starter-*`) |
| Boot time | Slower setup | Fast start |

---

### How Auto-configuration Works
1. `@SpringBootApplication` → includes `@EnableAutoConfiguration`
2. Spring Boot scans `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`
3. Loads condition-based beans (e.g., `@ConditionalOnClass(DataSource.class)`)

---

### Key Annotations
| Annotation | Purpose |
|---|---|
| `@SpringBootApplication` | = `@Configuration` + `@EnableAutoConfiguration` + `@ComponentScan` |
| `@ComponentScan` | Scans packages for Spring components |
| `@EnableAutoConfiguration` | Triggers Spring Boot auto-config |
| `@Component` | Generic Spring-managed bean |
| `@Service` | Service layer bean |
| `@Repository` | DAO layer bean (with exception translation) |
| `@Controller` | MVC controller |
| `@RestController` | = `@Controller` + `@ResponseBody` |
| `@Autowired` | Dependency injection |
| `@Value` | Inject property values |
| `@Bean` | Declare a bean in a `@Configuration` class |
| `@Transactional` | Wrap method in DB transaction |

---

### @GetMapping vs @RequestMapping
```java
// @RequestMapping is general — supports all HTTP methods
@RequestMapping(value = "/users", method = RequestMethod.GET)

// @GetMapping is a shortcut for GET only
@GetMapping("/users")   // cleaner, preferred
```

---

### Spring Bean Scopes
| Scope | Lifecycle |
|---|---|
| **singleton** (default) | One instance per Spring context |
| **prototype** | New instance per `getBean()` call |
| **request** | One per HTTP request (web only) |
| **session** | One per HTTP session (web only) |

---

### BeanPostProcessor
Hooks into bean lifecycle to customize beans before/after initialization.
```java
@Component
public class MyBPP implements BeanPostProcessor {
    public Object postProcessBeforeInitialization(Object bean, String name) {
        System.out.println("Before init: " + name);
        return bean;
    }
}
```

---

### Custom Annotation
```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface AuditLog {
    String action() default "ACCESS";
}

// Use with AOP
@Aspect @Component
public class AuditAspect {
    @Around("@annotation(auditLog)")
    public Object log(ProceedingJoinPoint pjp, AuditLog auditLog) throws Throwable {
        System.out.println("Action: " + auditLog.action());
        return pjp.proceed();
    }
}
```

---

### AOP (Aspect-Oriented Programming)
Cross-cutting concerns (logging, security, transactions) are separated from business logic.

**Key terms:**
- **Aspect** → the cross-cutting concern class
- **Advice** → `@Before`, `@After`, `@Around`, `@AfterReturning`, `@AfterThrowing`
- **Pointcut** → expression defining where advice applies
- **JoinPoint** → the actual method being intercepted

---

### Actuator
Provides production-ready monitoring endpoints.
```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, metrics, info, env
```
Common endpoints: `/actuator/health`, `/actuator/metrics`, `/actuator/info`

---

### Cron Expression (Spring Batch / @Scheduled)
```
┌─── second (0-59)
│ ┌─── minute (0-59)
│ │ ┌─── hour (0-23)
│ │ │ ┌─── day of month (1-31)
│ │ │ │ ┌─── month (1-12)
│ │ │ │ │ ┌─── day of week (0-6, 0=Sun)
│ │ │ │ │ │
0 0 9 * * MON  → Every Monday at 9:00 AM
0 */5 * * * *  → Every 5 minutes
```

---

### Couchbase Connection (Spring Boot)
```yaml
spring:
  couchbase:
    connection-string: couchbase://localhost
    username: admin
    password: secret

spring:
  data:
    couchbase:
      bucket-name: my-bucket
```

---

## 10. MICROSERVICES ARCHITECTURE

### Monolithic vs Microservices
| | Monolithic | Microservices |
|---|---|---|
| Deployment | All-in-one | Each service independently |
| Scaling | Scale everything | Scale individual services |
| Tech stack | Single | Polyglot |
| Best for | Small teams, early stage | Large systems, independent teams |

> Use monolith when starting out. Migrate to microservices when a specific component needs independent scaling or team isolation.

---

### Service-to-Service Communication
```java
// 1. RestTemplate (traditional)
RestTemplate rt = new RestTemplate();
String result = rt.getForObject("http://order-service/orders/1", String.class);

// 2. WebClient (reactive, preferred)
WebClient.create("http://order-service")
    .get().uri("/orders/1")
    .retrieve().bodyToMono(String.class);

// 3. OpenFeign (declarative, cleanest)
@FeignClient(name = "order-service")
interface OrderClient {
    @GetMapping("/orders/{id}")
    Order getOrder(@PathVariable Long id);
}
```

---

### Circuit Breaker in Microservices
(See section 8 — Circuit Breaker Pattern)

---

### Monitoring Microservices
- **Spring Boot Actuator** → expose metrics
- **Micrometer** → metrics instrumentation
- **Prometheus** → scrapes and stores metrics
- **Grafana** → visualization dashboards
- **ELK Stack** (Elasticsearch + Logstash + Kibana) → centralized logging
- **Zipkin / Jaeger** → distributed tracing
- **Spring Cloud Sleuth** → auto-injects trace IDs

---

### Load Testing Microservices
- **Apache JMeter** — UI-based, HTTP load testing
- **Gatling** — code-based (Scala DSL), detailed reports
- **K6** — JavaScript-based, modern
- **Locust** — Python-based

CPU/memory monitoring during tests:
```bash
docker stats <container-id>   # live container metrics
kubectl top pod               # Kubernetes pod metrics
```

---

### Application Slowness Resolution
1. Check GC logs — excessive GC pauses?
2. Thread dump — deadlocks or thread starvation?
3. Heap dump — memory leaks?
4. Slow SQL queries — check `EXPLAIN PLAN`, missing index?
5. Network latency — check inter-service call times
6. Connection pool exhaustion — tune `HikariCP` settings

---

## 11. KAFKA

### Core Concepts
| Term | Meaning |
|---|---|
| **Topic** | Named stream/category of messages |
| **Partition** | Topic split into ordered, immutable logs |
| **Broker** | Kafka server that stores and serves messages |
| **Producer** | Sends messages to a topic |
| **Consumer** | Reads messages from a topic |
| **Consumer Group** | Multiple consumers sharing partition load |
| **Offset** | Position of a message in a partition |

---

### Partition & Consumer Strategy for Performance
- **More partitions** = more parallelism
- **Consumer count ≤ Partition count** (extra consumers sit idle)
- Rule: **1 consumer per partition** for max throughput

```
Topic: orders (3 partitions)
Consumer Group: order-processors (3 consumers)
→ Consumer 1 reads Partition 0
→ Consumer 2 reads Partition 1
→ Consumer 3 reads Partition 2
```

---

### Failover / Message Recovery Scenarios

**Consumer didn't receive message:**
- Kafka retains messages for a configurable duration (`log.retention.hours`)
- Consumer can reset offset: `--reset-offsets --to-earliest`

**Consumer received message but processing failed:**
```java
// Manual offset commit — only commit after successful processing
@KafkaListener(topics = "orders")
public void consume(String message, Acknowledgment ack) {
    try {
        process(message);
        ack.acknowledge(); // commit only on success
    } catch (Exception e) {
        // don't ack → message will be redelivered
    }
}
```

**Dead Letter Topic (DLT):** Failed messages after retries are routed to a DLT for inspection.

---

### Kafka Topic Partition Strategy
- **Round Robin** → default, distributes evenly
- **Key-based** → same key always goes to same partition (order guarantee)
- **Custom Partitioner** → implement `Partitioner` interface

---

## 12. DATABASE & SQL

### Indexing
Speeds up read queries by building a sorted B-Tree structure on the indexed column.  
**Downside:** Slows down writes (index must be updated).

```sql
CREATE INDEX idx_salary ON employees(salary);
SELECT * FROM employees WHERE salary > 50000; -- uses index
```

---

### Primary Key vs Unique Constraint
| | Primary Key | Unique Constraint |
|---|---|---|
| Null allowed | No | Yes (one null) |
| Count per table | One | Multiple |
| Clustered index | Yes (usually) | No |

---

### SQL Queries

**Max salary:**
```sql
SELECT MAX(salary) FROM employees;
```

**Second max salary:**
```sql
SELECT MAX(salary) FROM employees WHERE salary < (SELECT MAX(salary) FROM employees);
-- OR
SELECT DISTINCT salary FROM employees ORDER BY salary DESC LIMIT 1 OFFSET 1;
```

**Third max salary:**
```sql
SELECT DISTINCT salary FROM employees ORDER BY salary DESC LIMIT 1 OFFSET 2;
```

---

### Query Performance Tuning
- Use `EXPLAIN ANALYZE` to check execution plan
- Add indexes on `WHERE`, `JOIN`, `ORDER BY` columns
- Avoid `SELECT *` — fetch only needed columns
- Avoid functions on indexed columns in WHERE: `WHERE YEAR(created_at) = 2024` → won't use index
- Use pagination (`LIMIT`/`OFFSET`) for large datasets
- Denormalize hot read paths if needed

---

### Normalization
- **1NF** — Atomic values, no repeating groups
- **2NF** — 1NF + no partial dependency on composite key
- **3NF** — 2NF + no transitive dependency
- **BCNF** — Every determinant is a candidate key

---

### Why Non-Relational Databases?
- Flexible schema (schema changes don't require migrations)
- Horizontal scaling (sharding)
- High read/write throughput (Cassandra, MongoDB)
- Hierarchical/document data fits naturally (no JOINs needed)
- Use cases: user profiles, product catalogs, logs, real-time feeds

---

## 13. REST & SOAP

### REST vs SOAP
| | REST | SOAP |
|---|---|---|
| Protocol | HTTP | HTTP, SMTP, TCP |
| Format | JSON, XML | XML only |
| Performance | Faster, lightweight | Heavier |
| Standards | Flexible | Strict (WSDL) |
| Use case | Web/mobile APIs | Enterprise, banking, legacy |

---

### REST Best Practices
- Use nouns for endpoints: `/users`, not `/getUsers`
- Use HTTP verbs correctly: GET (read), POST (create), PUT (update), DELETE (remove)
- Return appropriate status codes: 200, 201, 400, 401, 404, 500
- Use HTTPS
- Version your API: `/api/v1/users`

**Can GET fetch billions of records?** Technically yes, but you must use **pagination** (`?page=1&size=50`). Never return unbounded datasets.

---

### Connection Timeout vs Socket Exception
| | Connection Timeout | Socket Exception |
|---|---|---|
| When | Server not reachable within time limit | Connection dropped mid-communication |
| Cause | Network issue, server down, wrong port | Server crash, network interruption |
| Fix | Retry with backoff, increase timeout | Circuit breaker, retry logic |

---

### JSON in REST
```java
// Spring Boot automatically serializes/deserializes with Jackson
@PostMapping("/users")
public ResponseEntity<User> create(@RequestBody User user) { // @RequestBody deserializes JSON
    return ResponseEntity.status(201).body(userService.save(user));
}

@GetMapping("/users/{id}")
public User get(@PathVariable Long id) {
    return userService.findById(id); // auto-serialized to JSON
}
```

Jackson iterates the JSON keys, maps to Java field names (using getters/setters or `@JsonProperty`).

---

## 14. CI/CD, DOCKER & KUBERNETES

### CI/CD Pipeline
```
Code Push → Git → CI Server (Jenkins/GitHub Actions)
              ↓
         Build (mvn package)
              ↓
         Unit Tests
              ↓
         Code Quality (SonarQube)
              ↓
         Build Docker Image
              ↓
         Push to Registry
              ↓
         Deploy to Staging (CD)
              ↓
         Integration Tests
              ↓
         Deploy to Production
```

---

### Docker
**Containers** package the app + its dependencies into an isolated, portable unit.

```dockerfile
FROM openjdk:17-jre-slim
WORKDIR /app
COPY target/app.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

```bash
docker build -t my-app:1.0 .
docker run -p 8080:8080 my-app:1.0
docker ps          # running containers
docker logs <id>   # container logs
docker stats       # live CPU/memory
```

---

### Kubernetes
Orchestrates containers across multiple machines.

**Key concepts:**
| Term | Meaning |
|---|---|
| **Pod** | Smallest unit — one or more containers |
| **Deployment** | Manages pod replicas, rolling updates |
| **Service** | Stable endpoint (DNS + load balancing) for pods |
| **ConfigMap** | External config for pods |
| **Ingress** | HTTP routing into the cluster |
| **Namespace** | Logical isolation within cluster |

```bash
kubectl get pods
kubectl describe pod <pod-name>
kubectl top pod           # CPU/memory usage
kubectl logs <pod-name>
kubectl scale deployment my-app --replicas=5
```

---

## 15. TESTING

### Testing Frameworks
| Framework | Purpose |
|---|---|
| JUnit 5 | Unit testing |
| Mockito | Mocking dependencies |
| Spring Boot Test | Integration testing |
| Testcontainers | Real DB in tests (Docker-based) |
| WireMock | Mock external HTTP services |

```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {
    @Mock UserRepository userRepo;
    @InjectMocks UserService userService;

    @Test
    void shouldReturnUser() {
        when(userRepo.findById(1L)).thenReturn(Optional.of(new User("Faizal")));
        User user = userService.getUser(1L);
        assertEquals("Faizal", user.getName());
    }
}
```

---

### Code Coverage
- **JaCoCo** → most common tool for Java code coverage
- Tracks which lines, branches, methods were executed by tests
- Aim for 80%+ coverage on business logic

---

### MQ Message Recovery (App A → MQ → App B scenario)
Node B1 consumed message and crashed before processing:

**Solution:** Use **message acknowledgment**  
- B1 only ACKs after successful processing
- If B1 crashes before ACK, MQ redelivers to B2
- **Idempotency** is critical — design consumers to handle duplicate messages safely

---

## 16. DSA & SORTING

### Time Complexities
| Algorithm | Best | Average | Worst | Space |
|---|---|---|---|---|
| Bubble Sort | O(n) | O(n²) | O(n²) | O(1) |
| Selection Sort | O(n²) | O(n²) | O(n²) | O(1) |
| Insertion Sort | O(n) | O(n²) | O(n²) | O(1) |
| Merge Sort | O(n log n) | O(n log n) | O(n log n) | O(n) |
| Quick Sort | O(n log n) | O(n log n) | O(n²) | O(log n) |
| `Arrays.sort()` | — | O(n log n) | O(n log n) | O(log n) |

> `Arrays.sort()` uses **Dual-Pivot Quicksort** for primitives, **TimSort** (merge + insertion) for objects.

---

### O(n) Sorting — Counting Sort
Works only when values are in a known, bounded integer range.
```java
int[] arr = {4, 2, 2, 1, 4, 3};
int max = Arrays.stream(arr).max().getAsInt();
int[] count = new int[max + 1];
for (int n : arr) count[n]++;
// reconstruct: [1, 2, 2, 3, 4, 4]
```

**Radix Sort** and **Bucket Sort** also achieve O(n) under specific conditions.

---

### Find Duplicates in Integer Array
```java
int[] arr = {1, 2, 3, 2, 4, 3};
Set<Integer> seen = new HashSet<>();
List<Integer> duplicates = new ArrayList<>();

for (int n : arr) {
    if (!seen.add(n)) duplicates.add(n); // add returns false if already present
}
System.out.println(duplicates); // [2, 3]
```

---

### Time & Space Complexity
- **Time Complexity** → How execution time grows with input size (Big O notation)
- **Space Complexity** → How memory usage grows with input size

```
O(1) < O(log n) < O(n) < O(n log n) < O(n²) < O(2ⁿ)
```

---

## 17. CODE REVIEW CHECKLIST

1. **Correctness** — Does the code do what's intended? Edge cases handled?
2. **Naming** — Are variables, methods, and classes clearly named?
3. **SOLID Principles** — Is there a single responsibility? Is it extensible?
4. **Error Handling** — Are exceptions caught at the right level? Custom exceptions used?
5. **Null Safety** — Are NPEs prevented? `Optional` used where appropriate?
6. **Thread Safety** — Are shared resources properly synchronized?
7. **Performance** — Any N+1 queries? Inefficient loops? Missing indexes?
8. **Security** — SQL injection? Input validation? Sensitive data in logs?
9. **Test Coverage** — Are new paths covered by unit/integration tests?
10. **Documentation** — Complex logic commented? Javadoc for public APIs?
11. **Dependencies** — No unnecessary libraries added? Version conflicts?
12. **Logging** — Appropriate log levels? No sensitive info logged?

---

## 18. MISCELLANEOUS & SCENARIO QUESTIONS

### How Typing google.com works (DNS Resolution)
1. Browser checks local cache
2. OS checks `/etc/hosts`
3. Query sent to **DNS Resolver** (ISP or 8.8.8.8)
4. Resolver queries **Root DNS** → **TLD (.com)** → **Authoritative DNS**
5. IP returned → browser opens TCP connection (TLS handshake for HTTPS)
6. HTTP GET request sent → Google's server responds

---

### How TinyURL Works
1. User submits long URL
2. System generates a short hash (e.g., base62 encoding of an auto-increment ID)
3. Stores mapping `{shortCode → longURL}` in database (Redis cache for hot URLs)
4. On access: lookup shortCode → 301/302 redirect to longURL

---

### Session Tracking in Servlets
| Method | How |
|---|---|
| **Cookies** | Server sends `Set-Cookie` header, browser returns it on each request |
| **URL Rewriting** | Session ID appended to URL: `/page?jsessionid=abc123` |
| **HttpSession** | `request.getSession()` — server-side session object |
| **Hidden Form Fields** | Session data embedded in HTML forms |

---

### JDBC Connection Steps
```java
// 1. Load driver (auto in modern JDBC)
Class.forName("com.mysql.cj.jdbc.Driver");

// 2. Get connection
Connection conn = DriverManager.getConnection(
    "jdbc:mysql://localhost:3306/mydb", "user", "pass");

// 3. Create statement
Statement stmt = conn.createStatement();           // basic
PreparedStatement ps = conn.prepareStatement(      // precompiled, safe from SQL injection
    "SELECT * FROM users WHERE id = ?");
ps.setInt(1, 42);
CallableStatement cs = conn.prepareCall("{call getUserById(?)}"); // stored procedures

// 4. Execute
ResultSet rs = ps.executeQuery();

// 5. Close resources
rs.close(); ps.close(); conn.close();
```

---

### Honking Pattern (Decorator Pattern)
Adds behavior to an object dynamically without subclassing.
```java
interface Car { void assemble(); }
class BasicCar implements Car { public void assemble() { System.out.println("Basic car"); } }

class CarDecorator implements Car {
    protected Car car;
    CarDecorator(Car c) { this.car = c; }
    public void assemble() { car.assemble(); }
}

class SportsCar extends CarDecorator {
    SportsCar(Car c) { super(c); }
    public void assemble() {
        super.assemble();
        System.out.println("+ Sports features");
    }
}

// Usage
Car car = new SportsCar(new BasicCar());
car.assemble();
```

---

### IOC vs DI
- **IoC** is the broader principle — "don't call us, we'll call you" — the framework controls flow
- **DI** is one implementation of IoC — dependencies are injected by the container (not `new`-ed by the class)

---

### Base Path for RESTful Services
```java
// Class-level @RequestMapping sets the base path
@RestController
@RequestMapping("/api/v1/users")  // base path
public class UserController {
    
    @GetMapping("/{id}")     // full: /api/v1/users/{id}
    public User getUser(@PathVariable Long id) { ... }
}
```

Or globally in `application.properties`:
```properties
server.servlet.context-path=/api
```

---

### Linked List — Add to Beginning
```java
class Node {
    int data;
    Node next;
    Node(int data) { this.data = data; }
}

class LinkedList {
    Node head;
    void addFirst(int data) {
        Node newNode = new Node(data);
        newNode.next = head;   // point to current head
        head = newNode;        // new head is the new node
    }
}
```

---

*Built for Faizal Sulthan — UST Global → Backend Developer transition. Review one section per day, run the code examples in your IDE, and you'll be interview-ready. Good luck! 🚀*
