# Part 4 — Spring Boot Auto-Configuration & Starters

> Bootstrapping, @Conditional, how auto-configuration actually works, custom starters, startup internals, reducing startup time, switching packaging/webserver. Interview Q&A at the end.

## What Is Bootstrapping in Spring Boot?

**What it is:** the startup sequence triggered by `SpringApplication.run()` — creates the `ApplicationContext`, triggers component scanning, applies auto-configuration based on classpath contents, processes `@Bean` definitions, publishes application lifecycle events, and finally starts the embedded server (Tomcat/Jetty/Netty) if it's a web application.

**Why it matters:** this is what turns a `main()` method into a fully configured, running application with minimal explicit configuration — the mechanism behind Spring Boot's "convention over configuration" philosophy.

## @Profile vs @ConditionalOnXXX

**What they do:** `@Profile("dev")` activates a bean only when a specific named Spring profile is active (set via `spring.profiles.active`) — environment-based conditional bean registration. `@ConditionalOnXXX` (e.g. `@ConditionalOnClass`, `@ConditionalOnProperty`, `@ConditionalOnMissingBean`) is a broader family of conditions evaluated on classpath contents, property values, or existing bean presence — not tied to named environments. This is the mechanism Spring Boot's own **auto-configuration** is built on internally.

**Why they're related:** `@Profile` is actually itself implemented as a specialization built on the same underlying `@Conditional` infrastructure that `@ConditionalOnXXX` annotations use.

> ⚠️ **Pitfall:** knowing that `@Profile` is built on top of the same `@Conditional` mechanism (not a separate, unrelated system) is the detail that shows real depth here.

## How Spring Boot Auto-Configuration Actually Works

**What it does:** based on the `@Conditional`-family annotations applied to configuration classes bundled inside Spring Boot's "autoconfigure" jars — each auto-configuration class conditionally registers beans based on what's present on the classpath (`@ConditionalOnClass`), what beans already exist (`@ConditionalOnMissingBean`), and property values (`@ConditionalOnProperty`).

**Where the list lives:** since Spring Boot 2.7+/3.0, these are registered via a `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` file (previously `spring.factories`) that Spring Boot scans at startup to discover candidate classes.

**The key mental model:** `@ConditionalOnMissingBean` is what makes auto-configuration "opinionated but overridable" — define your own `DataSource` bean, and Spring Boot's auto-configured default backs off automatically.

> ⚠️ **Pitfall:** the "backs off if you define your own" behavior is essential — auto-configuration isn't a rigid framework taking over, it's a set of sensible fallback defaults that defer to explicit application configuration whenever present.

## The @Conditional Family, with Concrete Examples

```java
@Service
@ConditionalOnProperty(value = "logging.enabled", havingValue = "true", matchIfMissing = true)
class LoggingService { }
```
- `@ConditionalOnProperty` — bean registered only if a specific property equals a given value (or is absent, per `matchIfMissing`).
- `@ConditionalOnClass`/`@ConditionalOnMissingBean` — based on classpath presence or absence of another bean.
- `@ConditionalOnExpression` — a full SpEL expression combining multiple conditions.
- `@ConditionalOnJava` — gated on the running JVM version.
- Custom conditions: implement `Condition`'s `matches()` method, apply via `@Conditional(YourCondition.class)` for logic none of the built-in variants cover.

## What @SpringBootApplication Actually Does

**What it is:** a meta-annotation bundling three annotations: `@SpringBootConfiguration` (marks the class as a `@Configuration` source), `@EnableAutoConfiguration` (triggers the classpath/property-based auto-configuration mechanism), and `@ComponentScan` (scans the annotated class's package and sub-packages for `@Component`-family beans).

> ⚠️ **Pitfall:** because `@ComponentScan` defaults to the package of the annotated class, placing your main application class in the wrong (too-shallow or sibling) package silently misses component scanning some beans — a common real setup mistake.

## How Spring Boot Knows Which Auto-Configurations to Load, and Excluding One

**Where it reads from:** Spring Boot ≤2.7 reads candidate classes from `META-INF/spring.factories`; 3.0+ uses `META-INF/spring/...AutoConfiguration.imports` — either way, a list of candidate classes written by library authors, filtered by each class's own `@Conditional` guards.

**Excluding one:**
```java
@SpringBootApplication(exclude = {DataSourceAutoConfiguration.class})
```
Or the `excludeName` attribute (fully-qualified class name string, for a class not on your compile classpath), or `spring.autoconfigure.exclude=...` in `application.properties`.

> ⚠️ **Pitfall:** `excludeName` exists specifically for excluding a class you can't reference directly — knowing why that variant exists, not just that it does, is the deeper answer.

## Creating a Custom Spring Boot Starter

**What it involves:** a module with an `autoconfigure` package containing `@Configuration` classes guarded by `@Conditional` annotations (mirroring built-in starters), registered in `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` so Boot discovers it automatically when the starter JAR is on the classpath.

**Conventional structure:** split into two modules — the `*-spring-boot-autoconfigure` module (actual configuration logic) and a thin `*-spring-boot-starter` module (just a POM aggregating the autoconfigure module plus any required third-party dependencies) — mirrors the structure of official Spring Boot starters. Provide `@ConfigurationProperties`-bound configuration classes so consumers can customize behavior via `application.yml`.

## What Happens Internally During Spring Boot Startup

1. `SpringApplication.run()` creates a `SpringApplication` instance, determines the application type (servlet web, reactive web, or none) based on classpath.
2. Loads `ApplicationContextInitializers` and `ApplicationListeners`, prepares the `Environment` (property sources: command-line args, `application.yml`, environment variables, in defined precedence order).
3. Creates the appropriate `ApplicationContext`, applies `BeanFactoryPostProcessor`s and `BeanDefinitionRegistryPostProcessor`s (including auto-configuration resolution), then instantiates and wires all singleton beans (DI, `@PostConstruct`).
4. Publishes `ApplicationStartedEvent`/`ApplicationReadyEvent`.
5. For web applications, starts the embedded server (Tomcat/Jetty/Netty) and begins accepting requests.

> ⚠️ **Pitfall:** structure this as a clear sequence (environment prep → context creation → bean processing → embedded server start) rather than a jumbled list — interviewers check for a coherent mental model.

## Reducing Startup Time

- **Lazy initialization** (`spring.main.lazy-initialization=true`) — beans created on first use rather than eagerly at startup; trades startup speed for slightly delayed first-request latency.
- **Reduce component scanning scope** — narrow `@ComponentScan` base packages.
- **Exclude unused auto-configurations** explicitly (`@SpringBootApplication(exclude = {...})`).
- **AOT (Ahead-of-Time) processing + GraalVM native image** (Spring Boot 3+) — dramatically faster startup (often milliseconds instead of seconds) by doing reflection/proxy resolution at build time instead of runtime — the most impactful modern option, especially for serverless/scale-to-zero deployments.
- **Profile actual startup phases** (Spring Boot logs a startup breakdown, or `--debug`) to find what's actually slow rather than guessing.

> ⚠️ **Pitfall:** GraalVM native image / AOT is the current state-of-the-art answer and shows up-to-date knowledge — candidates whose knowledge predates Spring Boot 3's native support miss the most impactful modern option available.

## The Role of Starter Dependencies

**What they do:** curated dependency bundles (`spring-boot-starter-web`, `spring-boot-starter-data-jpa`, etc.) that pull in a tested, compatible set of transitive libraries for a given concern, instead of you hand-picking and version-matching each one.

> ⚠️ **Pitfall:** the real value is **version compatibility management**, not just convenience — starters exist specifically so you don't hit subtle version-mismatch bugs between related libraries.

## Changing Packaging from JAR to WAR

- Change `<packaging>jar</packaging>` to `<packaging>war</packaging>`.
- Extend `SpringBootServletInitializer` in the main application class, overriding `configure(SpringApplicationBuilder)`.
- Mark the embedded container dependency (e.g. `spring-boot-starter-tomcat`) as `provided` scope.

> ⚠️ **Pitfall:** modern deployments (containers/Kubernetes) have largely moved away from WAR-to-external-app-server deployment in favor of self-contained executable JARs — this is increasingly a legacy pattern.

## Switching the Embedded Web Server to Jetty

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jetty</artifactId>
</dependency>
```
Exclude the default `spring-boot-starter-tomcat` transitive dependency, then add `spring-boot-starter-jetty` — Spring Boot's auto-configuration detects whichever embedded server is on the classpath.

> ⚠️ **Pitfall:** forgetting the exclusion step leaves both servers on the classpath, which can cause conflicts or Spring Boot picking an unintended one.

---

## Interview Q&A

**Q: What is bootstrapping in Spring Boot? Explain its importance.**
Covered above.

**Q: Difference between @Profile and @ConditionalOnXXX?**
Covered above.

**Q: How does Spring Boot auto-configuration work?**
Covered above.

**Q: How to change the packaging from JAR to WAR in a Spring Boot app?**
Covered above.

**Q: How to change the webserver to Jetty in a Spring Boot app?**
Covered above.

**Q: How to create a custom Spring Boot starter?**
Covered above.

**Q: What happens internally during Spring Boot app startup?**
Covered above.

**Q: How do you reduce startup time in Spring Boot apps?**
Covered above.

**Q: What is the role of Spring Boot Starter dependencies?**
Covered above.

**Q: What does @SpringBootApplication actually do internally?**
Covered above.

**Q: How does Spring Boot know which auto-configuration classes to load, and how do you exclude one?**
Covered above.

**Q: What are Spring's @Conditional-family annotations, with concrete examples?**
Covered above.
