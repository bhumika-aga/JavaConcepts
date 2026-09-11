# Inside Spring Boot

_spring boot · auto-configuration to fat jar_

> Spring gives you a container. Boot gives you _everything around it_ — sensible defaults that retreat the moment you disagree, one file you can actually run, and the endpoints ops will ask for. None of it is magic; all of it is conditions and ordering.

---

## Contents

1. [What Boot actually adds](#what-boot-actually-adds)
2. [Starters](#starters)
3. [How auto-configuration works](#how-auto-configuration-works)
4. [The conditions](#the-conditions)
5. [Overriding it](#overriding-it)
6. [Property sources](#property-sources)
7. [Binding and profiles](#binding-and-profiles)
8. [The startup sequence](#the-startup-sequence)
9. [The fat jar](#the-fat-jar)
10. [Embedded servers](#embedded-servers)
11. [Actuator](#actuator)
12. [Logging and DevTools](#logging-and-devtools)
13. [AOT and native images](#aot-and-native-images)
14. [Ways to get hurt](#ways-to-get-hurt)

---

## What Boot actually adds

Boot is not a fork of Spring or a replacement for it. It's four additions on top, and they're separable — you could adopt any one without the others.

| Addition             | Replaces                          | The point                                                 |
| -------------------- | --------------------------------- | --------------------------------------------------------- |
| `Auto-configuration` | Pages of `@Bean` boilerplate      | Configure what's on the classpath, unless you already did |
| `Starters`           | Hand-picking compatible versions  | One dependency pulls a tested, coherent set               |
| `Executable jar`     | WAR files and an installed server | `java -jar app.jar` and it runs                           |
| `Actuator`           | Writing your own health endpoint  | Health, metrics and diagnostics as standard               |

> **This page assumes the container**
>
> Beans, dependency injection, scopes, the lifecycle and proxy-based annotations belong to Spring itself, and are covered in [**Inside Spring**](07-inside-spring.html). Everything here sits on top of that — if `@Transactional` mysteriously does nothing, the answer is in that lecture, not this one.

## Starters

A starter is a dependency with no code in it. It exists purely to drag in a curated set of other dependencies at versions known to work together.

```java
            <!-- One line. Roughly thirty jars, all version-matched. -->
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-web</artifactId>
</dependency>

// what it actually brings:
//   spring-boot-starter        core + auto-config + logging
//   spring-webmvc              the MVC framework
//   spring-boot-starter-json   Jackson
//   spring-boot-starter-tomcat embedded server

// Notice there is no <version>. The parent manages it.
<parent>
  <artifactId>spring-boot-starter-parent</artifactId>
  <version>3.3.0</version>
</parent>

```

The parent does two things: it imports the `spring-boot-dependencies` BOM, which pins versions for hundreds of libraries, and it preconfigures plugins. If you already have a corporate parent POM, import the BOM directly instead — you get the version management without the plugin opinions.

> **Swapping a managed version**
>
> Override a BOM-managed version with a property, not a `<version>` tag: `<properties><jackson-bom.version>2.17.1</…>`. Setting the version on the dependency works but silently diverges from the tested set, which is exactly the problem starters exist to solve.

## How auto-configuration works

At startup Boot reads a list of candidate configuration classes shipped inside the jars on your classpath, then evaluates each one's conditions. Most are discarded.

![Candidate auto-configurations are loaded from a file in each jar, filtered by conditions, and only the matching ones register beans](diagrams/inside-spring-boot-01.svg)

**Candidates are names, not classes.** Boot deliberately checks the cheap conditions before loading anything, so 150 candidates costs milliseconds rather than 150 class loads.

```java
            // A real auto-configuration, simplified
@AutoConfiguration(after = DataSourceAutoConfiguration.class)
@ConditionalOnClass(JdbcTemplate.class)          // is the library present?
@ConditionalOnSingleCandidate(DataSource.class)   // exactly one DataSource?
public class JdbcTemplateAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean(JdbcOperations.class)   // did YOU define one?
    JdbcTemplate jdbcTemplate(DataSource ds) { return new JdbcTemplate(ds); }
}

```

## The conditions

Everything Boot decides comes from this family of annotations. They're available to you too, and they're the correct way to write a library that plays nicely.

| Condition                      | Matches when                   | Typical use                            |
| ------------------------------ | ------------------------------ | -------------------------------------- |
| `@ConditionalOnClass`          | A class is on the classpath    | "They added the Redis driver"          |
| `@ConditionalOnMissingBean`    | Nobody defined that bean       | Back off if the user configured it     |
| `@ConditionalOnBean`           | Some other bean exists         | Only configure if there's a DataSource |
| `@ConditionalOnProperty`       | A property has a value         | Feature flags                          |
| `@ConditionalOnWebApplication` | It's a servlet or reactive app | Web-only beans                         |
| `@ConditionalOnResource`       | A file exists                  | Legacy config files                    |
| `@ConditionalOnExpression`     | A SpEL expression is true      | Anything more complex                  |

> ⚠️ **@ConditionalOnMissingBean is unreliable in your own config**
>
> It works in auto-configuration _because auto-configuration runs last_. Put it in your own `@Configuration` class and the result depends on the order two of your classes happen to be processed in — which is not something you control. Use it for library code and auto-configuration; use `@Profile` or `@ConditionalOnProperty` for your own application.

```java
            // Seeing what actually happened — the single most useful Boot command
java -jar app.jar --debug

// prints a report:
//   Positive matches:  DataSourceAutoConfiguration matched because …
//   Negative matches:  RedisAutoConfiguration did not match because
//                      @ConditionalOnClass did not find RedisOperations

// Or at runtime, if actuator is on:   GET /actuator/conditions

```

## Overriding it

Three levers, in the order you should reach for them.

1. **Set a property** — Most auto-configuration is tunable without code. Check the reference before writing a bean.
2. **Define the bean yourself** — `@ConditionalOnMissingBean` means yours simply wins. This is the intended extension point.
3. **Exclude the auto-configuration** — A blunt instrument, and occasionally the right one — usually when a starter arrived transitively.

```java
            // 1 — a property
spring.jpa.hibernate.ddl-auto: validate

// 2 — your bean wins, no annotation needed on your side
@Bean ObjectMapper objectMapper() { return new ObjectMapper().findAndRegisterModules(); }

// 3 — exclude entirely
@SpringBootApplication(exclude = DataSourceAutoConfiguration.class)
// or:  spring.autoconfigure.exclude=org.springframework…DataSourceAutoConfiguration

```

Excluding `DataSourceAutoConfiguration` is the standard fix for the classic startup failure where a library dragged in JDBC and Boot now insists on a database URL you don't have.

## Property sources

Boot merges configuration from many places into one view. When a value isn't what you expected, it's nearly always because something higher on this ladder overrode it.

![A precedence ladder of configuration sources from command line arguments at the top down to defaults set in code at the bottom](diagrams/inside-spring-boot-02.svg)

**This ladder explains most "but I set that!" moments.** An environment variable set by your platform silently beats the value you committed — which is the intended design, and confusing exactly once.

```java
            # See the resolved value and where it came from:
GET /actuator/env/server.port

# Import extra config explicitly (Boot 2.4+)
spring.config.import: optional:file:./local.yml, configtree:/run/secrets/

```

## Binding and profiles

Boot maps loosely-spelled property names onto typed objects. The looseness is deliberate, because environment variables can't contain dots.

```java
            // All four of these bind to the same property — "relaxed binding"
app.mail-server.host          // kebab-case — the canonical form
app.mailServer.host
app.mail_server.host
APP_MAILSERVER_HOST           // how you set it in Docker or Kubernetes

// Typed, immutable, validated — the modern shape
@ConfigurationProperties(prefix = "app.mail-server")
@Validated
record MailProps(@NotBlank String host, int port, Duration timeout) { }

// enable with @ConfigurationPropertiesScan on the main class

// Durations and sizes parse from friendly strings
app.timeout: 30s        // → Duration
app.max-upload: 10MB    // → DataSize

```

> ⚠️ **@Value does not do relaxed binding**
>
> `@Value("${app.mailServer.host}")` will not pick up `APP_MAILSERVER_HOST`; it looks for that exact key. `@ConfigurationProperties` does the relaxed matching. This is the usual reason a value works locally in `application.yml` and is null in a container.

### Profiles

```java
            # application.yml — shared defaults
# application-prod.yml — merged over the top when prod is active

@Profile("!prod")  @Bean Mailer fake() { … }

# activate: --spring.profiles.active=prod   or  SPRING_PROFILES_ACTIVE=prod
# and inside a document, gate a whole block:
spring.config.activate.on-profile: prod

```

Profile-specific files **supplement** the base file rather than replacing it — `application.yml` is always loaded, then `application-prod.yml` layers on top. A key absent from the profile file keeps its base value, which is usually what you want and occasionally not.

## The startup sequence

What `SpringApplication.run()` actually does, in order. Worth knowing because the failure message tells you which step you died in.

![The Spring Boot startup sequence from environment preparation through context refresh to runners and the ready event](diagrams/inside-spring-boot-03.svg)

**The port opens at step 5, before the runners.** If a `CommandLineRunner` does slow warm-up work, traffic can arrive before it finishes — use the readiness probe rather than assuming startup order protects you.

## The fat jar

One file containing your code, every dependency, and a small launcher that knows how to read jars nested inside a jar — which the JVM cannot do on its own.

![The internal layout of an executable Spring Boot jar showing the loader, application classes, nested dependency jars and the manifest entries](diagrams/inside-spring-boot-04.svg)

**This is why `java -cp app.jar com.example.App` fails.** Your class is under `BOOT-INF/classes`, and its dependencies are jars the default classloader can't open. Always `java -jar`.

```java
            # Layered jars — one Docker layer per rate of change, so a code-only
# change re-uploads megabytes instead of hundreds.
java -Djarmode=tools -jar app.jar extract --layers --launcher

# Or skip the Dockerfile entirely — Boot builds an OCI image
mvn spring-boot:build-image

```

## Embedded servers

The server is a dependency, not an installation. Swapping it is two lines of build config.

```java
            <!-- Tomcat is the default. To use Undertow instead: -->
<exclusion>spring-boot-starter-tomcat</exclusion>
<dependency>spring-boot-starter-undertow</dependency>

# Tuning that actually matters under load
server.port: 8080
server.tomcat.threads.max: 200          # the real concurrency limit
server.tomcat.accept-count: 100         # queue depth once threads are busy
server.shutdown: graceful               # finish in-flight requests
spring.lifecycle.timeout-per-shutdown-phase: 30s

```

`server.shutdown=graceful` is worth setting in any containerised deployment: on SIGTERM the server stops accepting new connections but lets in-flight requests finish, instead of severing them mid-response during a rolling deploy.

## Actuator

Production endpoints you'd otherwise write yourself. Only `/health` is exposed over HTTP by default — deliberately, because several of the others leak secrets.

| Endpoint      | Gives you                           | Exposure              |
| ------------- | ----------------------------------- | --------------------- |
| `/health`     | Up/down, plus per-component detail  | on by default         |
| `/metrics`    | Micrometer meters                   | opt in                |
| `/prometheus` | Scrape format                       | opt in                |
| `/conditions` | Why auto-config matched             | opt in                |
| `/beans`      | Everything in the context           | opt in                |
| `/loggers`    | Read **and change** log levels live | opt in — write access |
| `/env`        | Every resolved property             | leaks secrets         |
| `/heapdump`   | A full heap dump                    | leaks everything      |

```java
            management.endpoints.web.exposure.include: health,info,prometheus
management.endpoint.health.show-details: when-authorized

# Kubernetes probes, split so a slow dependency doesn't cause a restart
management.endpoint.health.probes.enabled: true
#   /actuator/health/liveness   — am I broken? (restart me)
#   /actuator/health/readiness  — can I serve? (stop routing to me)

// A custom health check is one method
@Component
class QueueHealth implements HealthIndicator {
    public Health health() {
        return depth() < 1000 ? Health.up().build()
                            : Health.down().withDetail("depth", depth()).build();
    }
}

```

> ⚠️ **Never expose `*`**
>
> `management.endpoints.web.exposure.include=*` appears in countless tutorials. It publishes `/env` and `/heapdump` — your database password and a complete dump of process memory — to anyone who can reach the port. Expose the specific endpoints you need, and put the management port behind your network policy.

## Logging and DevTools

```java
            # Logback is the default; SLF4J is the API you code against
logging.level.root: INFO
logging.level.org.hibernate.SQL: DEBUG
logging.group.web: org.springframework.web, org.springframework.http
logging.level.web: DEBUG                 # set the whole group at once
logging.file.name: app.log

# Structured JSON logging is built in from Boot 3.4
logging.structured.format.console: ecs

```

**DevTools** adds an automatic restart on classpath change, using two classloaders — your code in a throwaway one, libraries in a permanent one, so a restart is far faster than a cold boot. It disables itself when running from a fat jar, so it can't reach production by accident. It's a development convenience and nothing more.

## AOT and native images

Boot 3 can do at build time what it normally does at startup — evaluate conditions, resolve beans — and emit code with the answers baked in.

|                 | JVM                   | Native image         |
| --------------- | --------------------- | -------------------- |
| Startup         | 1–3 seconds           | Tens of milliseconds |
| Memory          | Higher baseline       | Much lower           |
| Build time      | Seconds               | Minutes              |
| Peak throughput | Higher — JIT warms up | Lower                |
| Reflection      | Free                  | Must be registered   |

Native suits short-lived and scale-to-zero workloads — functions, CLI tools, anything billed per-millisecond. For a long-running server handling steady traffic, the JVM usually still wins on throughput once it has warmed up. The AOT step also runs on the JVM alone, where it trims a little startup time without giving up dynamic behaviour.

## Ways to get hurt

### Exposing all actuator endpoints

The single most damaging Boot misconfiguration. `include=*` puts `/env` and `/heapdump` on the network. Treat the management port as privileged infrastructure.

### `@Value` with a camelCase key in a container

Relaxed binding is a `@ConfigurationProperties` feature. `@Value` matches the literal key, so an environment variable that works for one silently fails for the other.

### Assuming your `application.yml` wins

It's near the bottom of the ladder. Environment variables injected by your platform beat it, which is by design — but it means the committed file is a default, not a decision. Check `/actuator/env` before arguing with the config.

### `@ConditionalOnMissingBean` in your own configuration

Only dependable in auto-configuration, which is ordered to run last. In application code the outcome depends on processing order you don't control.

### Unpacking the fat jar and running it by classpath

```java
            # ✗ Your class is under BOOT-INF/classes, deps are nested jars
java -cp app.jar com.example.App

# ✓ Let the launcher do it
java -jar app.jar

# ✓ Or extract properly first, for a leaner container image
java -Djarmode=tools -jar app.jar extract --launcher

```

### Slow work in a `CommandLineRunner`

The HTTP port opens before runners execute, so traffic can arrive while warm-up is still going. Gate real availability on the readiness probe instead.

### Reaching for `@SpringBootTest` everywhere

Every distinct test configuration builds and caches its own context. A suite with a dozen different `@MockitoBean` combinations pays a dozen full startups. Keep configurations uniform and prefer slices.

### Guessing instead of reading the report

When a bean appears you didn't declare, or vanishes when you expected it, `--debug` or `/actuator/conditions` tells you precisely which condition decided. This turns most Boot mysteries into a two-minute read.

> **The short version**
>
> Boot is conditions plus ordering. It configures what it finds on the classpath, backs off the moment you define the bean yourself, and merges configuration from a strict precedence ladder. When something surprises you, the condition report and `/actuator/env` answer nearly every question — and both are faster than reasoning about it.
