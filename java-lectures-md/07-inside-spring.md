# Inside Spring

_spring framework · container, proxies, boot_

> Spring has a reputation for magic. It isn't magic — it's _a container that builds your objects_ and _a proxy that wraps them_. Almost every surprising behaviour, and every annotation that mysteriously does nothing, comes back to one of those two.

---

## Contents

1. [Inversion of control](#inversion-of-control)
2. [The container](#the-container)
3. [Dependency injection](#dependency-injection)
4. [Scopes](#scopes)
5. [Declaring beans](#declaring-beans)
6. [The bean lifecycle](#the-bean-lifecycle)
7. [Proxies and AOP](#proxies-and-aop)
8. [@Transactional](#transactional)
9. [Spring Boot](#spring-boot)
10. [Spring MVC](#spring-mvc)
11. [Data access](#data-access)
12. [Testing](#testing)
13. [Spring 6 and Boot 3](#spring-6-and-boot-3)
14. [Ways to get hurt](#ways-to-get-hurt)

---

## Inversion of control

Normally your code creates what it needs. With Spring, something else creates it and hands it to you — so your class states _what_ it depends on and never decides _which_ implementation or _how_ to build it.

![Without a container a service constructs its own dependencies and is welded to them; with a container the service declares what it needs and the container supplies it](diagrams/inside-spring-01.svg)

**The payoff is testability, not cleverness.** A class that receives its dependencies can be unit-tested with two lines and no framework at all — which is the strongest argument for constructor injection later on.

## The container

The `ApplicationContext` is a registry of objects Spring built and manages. Those objects are **beans** — an ordinary Java object plus a definition telling Spring how to make it.

| Property          | Value                        |
| ----------------- | ---------------------------- |
| **Interface**     | ApplicationContext           |
| **Holds**         | Bean definitions + instances |
| **Default scope** | Singleton                    |
| **Created**       | Eagerly, at startup          |
| **A bean is**     | A normal object              |

```java
            // A bean is nothing special — no interface, no base class.
@Service
public class OrderService {
    private final OrderRepository repo;

    // Since Spring 4.3, a single constructor needs no @Autowired.
    public OrderService(OrderRepository repo) { this.repo = repo; }
}

// Spring finds it, builds it, wires it, and keeps one instance.
ApplicationContext ctx = SpringApplication.run(App.class, args);
OrderService svc = ctx.getBean(OrderService.class);

```

Beans are created **eagerly at startup**, not on first use. That's deliberate: a missing dependency or a bad config fails the moment the app boots, rather than at 3am on the first request that touches it.

## Dependency injection

Three ways to receive dependencies. One is right, one is situational, and one is a habit worth breaking.

| Style         | Looks like              | Verdict                                                                                     |
| ------------- | ----------------------- | ------------------------------------------------------------------------------------------- |
| `Constructor` | `Svc(Repo r)`           | **Use this.** Fields can be final, object is never half-built, and it's testable with `new` |
| `Setter`      | `@Autowired setRepo(…)` | Only for genuinely optional or reconfigurable dependencies                                  |
| `Field`       | `@Autowired Repo repo;` | Avoid. Can't be final, can't construct in a test, hides how many deps you have              |

```java
            // ✓ Constructor — the object is valid the instant it exists
@Service
class OrderService {
    private final Repo repo;
    OrderService(Repo repo) { this.repo = repo; }
}
// test: new OrderService(new FakeRepo());   — no Spring needed

// ✗ Field — compiles, runs, and quietly hurts
@Service
class OrderService {
    @Autowired private Repo repo;    // not final; null in a plain unit test
}

// Several candidates for one type? Disambiguate.
@Primary                        // the default winner
@Qualifier("fast")              // or pick by name at the injection point

```

> **Constructor injection kills circular dependencies at compile time**
>
> Two beans that need each other in their constructors _cannot both be built first_, so Spring fails at startup with a clear message. Field injection hides the cycle by building both empty and filling them in afterwards. Spring Boot 2.6+ rejects circular references by default — and the right fix is almost always to extract the shared logic into a third bean, not to set `spring.main.allow-circular-references=true`.

## Scopes

How many instances the container keeps. The default is **one**, and forgetting that is the source of a whole family of bugs.

| Scope       | Instances                       | Use for                                    |
| ----------- | ------------------------------- | ------------------------------------------ |
| `singleton` | One per container **(default)** | Nearly everything — services, repositories |
| `prototype` | A new one per injection/lookup  | Stateful helpers                           |
| `request`   | One per HTTP request            | Per-request context                        |
| `session`   | One per HTTP session            | Per-user state                             |

> ⚠️ **Singletons are shared by every thread**
>
> One instance serves every concurrent request, so a mutable field on a `@Service` is shared mutable state across threads — the exact hazard from _Inside Java Threads_. Keep beans stateless; put per-request data in method parameters and locals, which live on the calling thread's own stack.

```java
            // ✗ One instance, many threads, one field. A classic race.
@Service
class ReportService {
    private String currentUser;        // shared by everyone at once
    Report build(String user) { this.currentUser = user; … }
}

// ✓ Stateless — the parameter lives on the caller's stack
@Service
class ReportService {
    Report build(String user) { … }
}

// Injecting a prototype into a singleton gives you ONE instance forever —
// it is injected once, at startup. Ask the container each time instead:
private final ObjectProvider<Job> jobs;
Job j = jobs.getObject();      // a genuinely fresh one

```

## Declaring beans

Two styles in modern Spring: annotate your own classes, or write a factory method for things you don't own.

```java
            // 1 — Stereotypes on your own classes, found by component scanning.
@Component     // the generic one
@Service       // business logic      — same behaviour, better intent
@Repository    // data access         — also translates SQL exceptions
@Controller    // web endpoint

// 2 — @Bean methods, for third-party classes you can't annotate.
@Configuration
class AppConfig {
    @Bean
    RestClient restClient(Builder b) {
        return b.baseUrl("https://api.example.com").build();
    }

    @Bean
    @ConditionalOnMissingBean       // only if nobody else defined one
    Clock clock() { return Clock.systemUTC(); }
}

// Profiles switch whole sets of beans per environment.
@Profile("!prod")
@Bean Mailer loggingMailer() { return System.out::println; }

```

Scanning starts at the package of your `@SpringBootApplication` class and walks downward. A bean in a sibling package simply isn't found — which is why the main class belongs at the root of your package tree.

## The bean lifecycle

Between "constructor returns" and "your code runs" there are several steps — and knowing them explains why a field is still null in a constructor.

![The bean lifecycle from instantiation through dependency injection, post-processors, initialisation callbacks, use, and destruction](diagrams/inside-spring-02.svg)

**Step 6 is where the magic is installed.** `@Transactional`, `@Cacheable` and `@Async` don't modify your class — a post-processor swaps your bean for a proxy that wraps it. Everything in the next section follows from that.

## Proxies and AOP

This is the single most useful thing to understand about Spring. Annotations like `@Transactional` work by **wrapping your bean in another object** — and anything that bypasses that wrapper silently does nothing.

![An external caller goes through the proxy and gets a transaction, while an internal this-call goes straight to the target and is not intercepted](diagrams/inside-spring-03.svg)

**Self-invocation is the most common Spring bug there is.** The fix is to move the annotated method onto a different bean, so the call crosses a proxy boundary. Injecting the bean into itself also works, and reads like an apology.

### Which proxy you get

| Kind          | Requires                                    | Can't intercept                  |
| ------------- | ------------------------------------------- | -------------------------------- |
| `JDK dynamic` | An interface                                | Anything not on the interface    |
| `CGLIB`       | A non-final class with a usable constructor | `final, static, private methods` |

Spring Boot defaults to CGLIB (`proxyTargetClass=true`), so you usually get a generated subclass. That's why a `final` method or a `private` method can never be advised — the subclass has no way to override it.

```java
            // Your own aspect — the same mechanism the built-ins use.
@Aspect @Component
class TimingAspect {

    @Around("@annotation(Timed)")
    Object time(ProceedingJoinPoint pjp) throws Throwable {
        long t0 = System.nanoTime();
        try { return pjp.proceed(); }
        finally { log.info("{} took {}ns", pjp.getSignature(), System.nanoTime()-t0); }
    }
}

```

## @Transactional

A proxy that opens a transaction before your method and commits after — or rolls back. Its defaults surprise people, and two of them cause real data loss.

| Property                | Value                   |
| ----------------------- | ----------------------- |
| **Mechanism**           | Proxy, not bytecode     |
| **Default propagation** | REQUIRED                |
| **Rolls back on**       | RuntimeException, Error |
| **Checked exceptions**  | Commits anyway          |
| **Non-public methods**  | Ignored                 |
| **Self-invocation**     | Ignored                 |

```java
            // ✗ Checked exception → Spring COMMITS. This is the default.
@Transactional
void transfer() throws InsufficientFunds {   // a checked exception
    debit(); throw new InsufficientFunds();     // the debit is kept!
}

// ✓ Say what you mean
@Transactional(rollbackFor = Exception.class)

// ✗ Not public → the proxy cannot advise it → silently no transaction
@Transactional private void save() { … }

// Read-only is a real optimisation, not a hint
@Transactional(readOnly = true)   // skips dirty-checking; flags the JDBC connection

```

### Propagation — what happens when a transaction already exists

| Propagation    | Existing transaction          | None            |
| -------------- | ----------------------------- | --------------- |
| `REQUIRED`     | Join it **(default)**         | Start one       |
| `REQUIRES_NEW` | Suspend it, start a fresh one | Start one       |
| `MANDATORY`    | Join it                       | Throw           |
| `SUPPORTS`     | Join it                       | Run without one |
| `NEVER`        | Throw                         | Run without one |
| `NESTED`       | Savepoint inside it           | Start one       |

> ⚠️ **REQUIRED means one rollback poisons everything**
>
> Because the inner call _joins_ the outer transaction, an exception inside it marks the whole thing rollback-only — even if you catch the exception in the caller. The commit then fails with `UnexpectedRollbackException`, which looks like it came from nowhere. If a sub-operation must be able to fail independently (an audit log, say), it needs `REQUIRES_NEW` _and_ its own bean so the call crosses a proxy.

## Spring Boot

Boot is not a different framework. It's Spring plus opinionated defaults — auto-configuration, starter dependencies, and an embedded server.

```java
            @SpringBootApplication   // = @Configuration + @EnableAutoConfiguration + @ComponentScan
public class App {
    public static void main(String[] args) { SpringApplication.run(App.class, args); }
}

```

![Auto-configuration checks whether a class is on the classpath and whether you already defined the bean, and backs off if you did](diagrams/inside-spring-04.svg)

**Auto-configuration is conditional, not magic.** When something unexpected appears in your context, the auto-configuration report tells you which condition fired — it's the fastest way out of "where did this bean come from?".

### Configuration and profiles

```java
            # application.yml — later sources override earlier ones
spring.datasource.url: jdbc:postgresql://localhost/app

# Precedence, roughly: command line > env vars > profile-specific file
#                     > application.yml > defaults in code

// Typed, validated config beats scattered @Value
@ConfigurationProperties(prefix = "billing")
record BillingProps(String apiKey, Duration timeout) { }

// @Value is fine for one-offs, awkward for groups
@Value("${billing.timeout:30s}") Duration timeout;

```

## Spring MVC

One servlet receives every request and routes it. Knowing the path through it makes stack traces legible.

![A request passing through filters, the dispatcher servlet, handler mapping, the controller, and a message converter on the way back](diagrams/inside-spring-05.svg)

**Where to put cross-cutting logic:** a `Filter` for anything that must wrap the raw request (security, correlation ids), `@ControllerAdvice` for exception handling and shared model attributes.

```java
            @RestController                     // = @Controller + @ResponseBody
@RequestMapping("/orders")
class OrderController {

    @GetMapping("/{id}")
    OrderDto get(@PathVariable long id) { … }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    OrderDto create(@Valid @RequestBody CreateOrder body) { … }
}

// One place for error mapping across every controller
@RestControllerAdvice
class Errors {
    @ExceptionHandler(NotFound.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    Problem notFound(NotFound e) { return new Problem(e.getMessage()); }
}

```

## Data access

Spring Data writes your repository implementations from method names. Convenient, and it hides two performance traps worth knowing by name.

```java
            interface OrderRepository extends JpaRepository<Order, Long> {

    // Parsed from the name — no implementation needed
    List<Order> findByCustomerIdAndStatus(long id, Status s);

    // When the name would get silly, write the query
    @Query("select o from Order o join fetch o.items where o.id = :id")
    Optional<Order> findWithItems(@Param("id") long id);
}

```

> ⚠️ **Two traps that only appear under real data**
>
> **N+1 queries.** Loading 100 orders and touching `order.getItems()` on each fires 101 queries. Fix with `join fetch` or an entity graph — and turn on SQL logging in development so you can see it happen.
>
> **LazyInitializationException.** Touching a lazy association after the session closed. Boot's `spring.jpa.open-in-view` defaults to `true`, which hides this by holding the connection for the whole request — convenient in development, a connection-pool problem in production. Turn it off and fetch what you need explicitly.

## Testing

The main decision is how much of the container to start. Most tests need none of it.

| Approach          | Starts                | Use for                                                  |
| ----------------- | --------------------- | -------------------------------------------------------- |
| `Plain JUnit`     | Nothing               | **Most tests.** Constructor injection makes this trivial |
| `@WebMvcTest`     | Web layer only        | Controllers, JSON, validation                            |
| `@DataJpaTest`    | JPA + a test database | Repositories and queries                                 |
| `@SpringBootTest` | The whole context     | A few end-to-end paths — it's slow                       |

```java
            // No Spring at all — fast, and it proves the class is well-designed
@Test void appliesDiscount() {
    var svc = new OrderService(new InMemoryRepo(), Clock.fixed(…));
    assertThat(svc.total(order)).isEqualTo("90.00");
}

// A slice — only the web layer, with the service replaced
@WebMvcTest(OrderController.class)
class OrderControllerTest {
    @Autowired MockMvc mvc;
    @MockitoBean OrderService service;    // @MockBean in older Boot
}

// A real database in Docker, rather than H2 pretending to be Postgres
@Testcontainers @SpringBootTest

```

Context caching matters more than it looks: Spring reuses a context across test classes with identical configuration, so every distinct combination of `@MockitoBean` and properties creates another full startup. Keeping test configurations uniform is often the single biggest win in suite runtime.

## Spring 6 and Boot 3

The 2022 jump was the largest breaking change in Spring's history, and it's the one you'll meet in any migration.

- **Java 17 is the floor.** Spring 6 and Boot 3 will not run on 8 or 11.
- **`javax.*` became `jakarta.*`.** Every import of `javax.persistence`, `javax.servlet` or `javax.validation` has to change — this is the bulk of the migration work, and it's mechanical.
- **AOT and native images.** Boot 3 can pre-compute the context at build time for GraalVM, trading dynamic flexibility for near-instant startup.
- **Observability built in.** Micrometer tracing replaces Sleuth.
- **`RestClient`** arrived in Spring 6.1 as the modern synchronous HTTP client — `RestTemplate` is in maintenance.

> **Check versions rather than trusting any guide**
>
> Spring's release cadence is fast and defaults do shift between minor versions — `@MockBean` giving way to `@MockitoBean`, circular references flipping to rejected-by-default. Pin the documentation to _your_ Boot version; answers written for 2.x are often wrong for 3.x in small, expensive ways.

## Ways to get hurt

### Calling an annotated method on `this`

The number one Spring bug. `@Transactional`, `@Cacheable`, `@Async`, `@Retryable` — all of them are proxy-based, and an internal call never reaches the proxy. There is no warning; the annotation simply has no effect. Move the method to another bean.

### Expecting a checked exception to roll back

It won't. Spring rolls back on `RuntimeException` and `Error` only. If your domain uses checked exceptions, every `@Transactional` needs `rollbackFor`, or you will commit half-finished work.

### Mutable state on a singleton bean

One instance, every thread. A field that looks like per-call scratch space is shared across concurrent requests. Beans should be stateless; if a bean genuinely needs state, it needs the same treatment as any other shared object.

### Field injection

```java
            // Compiles. Runs. Can't be constructed in a test, can't be final,
// and lets a class quietly accumulate twelve dependencies
// without anyone noticing the constructor getting longer.
@Autowired private Repo repo;

```

### `@SpringBootTest` for everything

Starting the full context for a test that exercises one method makes suites take minutes instead of seconds. Reach for plain JUnit first, a slice second, and the full context only for genuine end-to-end coverage.

### Component scanning from the wrong place

Scanning begins at the main class's package. Put `@SpringBootApplication` somewhere deep and beans in sibling packages are invisible — producing a `NoSuchBeanDefinitionException` that looks like a wiring problem but is really a layout one.

### Trusting auto-configuration you haven't inspected

When a bean you never declared turns up, or one you expected doesn't, run with `--debug`. The condition evaluation report says exactly which auto-configuration matched and why — it turns guesswork into reading.

> **The short version**
>
> A container builds your objects, and a proxy wraps them. Take dependencies through the constructor, keep beans stateless, remember that annotations only work when the call crosses the proxy, and check the condition report when Spring does something you didn't ask for.
