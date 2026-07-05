# 🚀 Part 4 — Spring Boot Auto-Configuration & Starters

> Neat, point-based format with callout boxes, tables, and icons. Interview Q&A at the end.

---

## 🥾 What Is Bootstrapping in Spring Boot?

- The startup sequence triggered by `SpringApplication.run()`.
- Creates the `ApplicationContext`, triggers component scanning, applies auto-configuration based on classpath contents, processes `@Bean` definitions, publishes lifecycle events, and starts the embedded server (Tomcat/Jetty/Netty) for web apps.
- This is what turns a `main()` method into a fully configured, running application with minimal explicit configuration — Spring Boot's "convention over configuration" philosophy.

---

## 🎚️ @Profile vs @ConditionalOnXXX

| Annotation | What it's based on |
|---|---|
| `@Profile("dev")` | A specific named Spring profile being active (`spring.profiles.active`) — environment-based |
| `@ConditionalOnXXX` (`@ConditionalOnClass`, `@ConditionalOnProperty`, `@ConditionalOnMissingBean`) | Classpath contents, property values, existing bean presence — not tied to named environments |

> [!IMPORTANT]
> `@Profile` is actually itself implemented as a specialization built on the same underlying `@Conditional` infrastructure that `@ConditionalOnXXX` uses — they're the same mechanism at different levels of specificity.

---

## ⚙️ How Spring Boot Auto-Configuration Actually Works

- Based on `@Conditional`-family annotations applied to configuration classes bundled inside Spring Boot's "autoconfigure" jars.
- Each auto-configuration class conditionally registers beans based on:
  - what's present on the classpath (`@ConditionalOnClass`)
  - what beans already exist (`@ConditionalOnMissingBean`)
  - property values (`@ConditionalOnProperty`)
- Since Spring Boot 2.7+/3.0, these are registered via `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` (previously `spring.factories`).

> [!TIP]
> `@ConditionalOnMissingBean` is the key mental model — auto-configuration isn't a rigid framework taking over, it's a set of sensible fallback defaults that **defer to your own explicit configuration** whenever present. Define your own `DataSource` bean, and Boot's auto-configured default backs off automatically.

---

## 🧩 The @Conditional Family, with Concrete Examples

```java
@Service
@ConditionalOnProperty(value = "logging.enabled", havingValue = "true", matchIfMissing = true)
class LoggingService { }
```

| Annotation | Condition |
|---|---|
| `@ConditionalOnProperty` | A property equals a given value (or is absent, per `matchIfMissing`) |
| `@ConditionalOnClass` / `@ConditionalOnMissingBean` | Classpath presence or absence of a bean |
| `@ConditionalOnExpression` | A full SpEL expression combining multiple conditions |
| `@ConditionalOnJava` | Gated on JVM version |
| Custom `Condition` | Implement `matches()`, apply via `@Conditional(YourCondition.class)` |

---

## 🏷️ What @SpringBootApplication Actually Does

A meta-annotation bundling three:

| Bundled annotation | Purpose |
|---|---|
| `@SpringBootConfiguration` | Marks the class as a `@Configuration` source |
| `@EnableAutoConfiguration` | Triggers classpath/property-based auto-configuration |
| `@ComponentScan` | Scans the class's package and sub-packages for `@Component`-family beans |

> [!WARNING]
> `@ComponentScan` defaults to the package of the annotated class — placing your main application class in the wrong (too-shallow or sibling) package silently misses component scanning some beans. A common real setup mistake.

---

## 📜 How Spring Boot Knows Which Auto-Configs to Load, and Excluding One

| Boot version | Reads candidates from |
|---|---|
| ≤ 2.7 | `META-INF/spring.factories` |
| 3.0+ | `META-INF/spring/...AutoConfiguration.imports` |

**Excluding one:**
```java
@SpringBootApplication(exclude = {DataSourceAutoConfiguration.class})
```
- Or the `excludeName` attribute (fully-qualified class name string, for a class not on your compile classpath).
- Or `spring.autoconfigure.exclude=...` in `application.properties`.

> [!TIP]
> `excludeName` exists specifically for excluding a class you can't reference directly — knowing *why* that variant exists is the deeper answer.

---

## 🧰 Creating a Custom Spring Boot Starter

- A module with an `autoconfigure` package containing `@Configuration` classes guarded by `@Conditional` annotations.
- Registered in `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` so Boot discovers it automatically.
- **Conventional split into two modules:**
  - `*-spring-boot-autoconfigure` — the actual configuration logic
  - `*-spring-boot-starter` — a thin POM aggregating the autoconfigure module + required third-party dependencies
- Provide `@ConfigurationProperties`-bound classes so consumers can customize via `application.yml`.

---

## 🔍 What Happens Internally During Startup

1. `SpringApplication.run()` creates a `SpringApplication` instance, determines application type (servlet web, reactive web, or none).
2. Loads `ApplicationContextInitializers`/`ApplicationListeners`, prepares the `Environment` (property sources in defined precedence order).
3. Creates the `ApplicationContext`, applies `BeanFactoryPostProcessor`s (incl. auto-configuration resolution), instantiates and wires all singleton beans.
4. Publishes `ApplicationStartedEvent`/`ApplicationReadyEvent`.
5. For web apps, starts the embedded server and begins accepting requests.

---

## ⏱️ Reducing Startup Time

- **Lazy initialization** (`spring.main.lazy-initialization=true`) — beans created on first use.
- **Reduce component scanning scope** — narrow `@ComponentScan` base packages.
- **Exclude unused auto-configurations** explicitly.
- **AOT processing + GraalVM native image** (Spring Boot 3+) — dramatically faster startup (milliseconds instead of seconds), doing reflection/proxy resolution at build time.
- **Profile actual startup phases** rather than guessing.

> [!TIP]
> GraalVM native image / AOT is the current state-of-the-art answer — mentioning it shows up-to-date knowledge beyond Spring Boot 3's native support.

---

## 📦 The Role of Starter Dependencies

- Curated dependency bundles (`spring-boot-starter-web`, etc.) pulling in a tested, compatible set of transitive libraries.

> [!IMPORTANT]
> The real value is **version compatibility management**, not just convenience — starters exist so you don't hit subtle version-mismatch bugs between related libraries.

---

## 📦➡️🌐 Changing Packaging from JAR to WAR

- Change `<packaging>jar</packaging>` → `<packaging>war</packaging>`.
- Extend `SpringBootServletInitializer`, overriding `configure(SpringApplicationBuilder)`.
- Mark the embedded container dependency as `provided` scope.

> [!NOTE]
> Modern deployments (containers/Kubernetes) have largely moved away from WAR-to-external-app-server deployment in favor of self-contained executable JARs — this is increasingly a legacy pattern.

---

## 🔄 Switching the Embedded Server to Jetty

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

> [!WARNING]
> Forgetting the exclusion step leaves both servers on the classpath — can cause conflicts or Spring Boot picking an unintended one.

---

## 📋 Interview Q&A

| Question | Short answer |
|---|---|
| What is bootstrapping in Spring Boot? | The `SpringApplication.run()` startup sequence |
| @Profile vs @ConditionalOnXXX? | Both built on `@Conditional`; @Profile is environment-specific |
| How does auto-configuration work? | `@Conditional` classes + `@ConditionalOnMissingBean` "backs off if you define your own" |
| Changing JAR to WAR? | Change packaging, extend `SpringBootServletInitializer`, mark container `provided` |
| Switching to Jetty? | Exclude Tomcat starter, add Jetty starter |
| Creating a custom starter? | autoconfigure module + thin starter module, registered via imports file |
| What happens internally at startup? | Environment prep → context creation → bean processing → embedded server start |
| Reducing startup time? | Lazy init, narrower scanning, exclusions, AOT/native image |
| Role of starter dependencies? | Version-compatible dependency bundles |
| What does @SpringBootApplication do? | Bundles @SpringBootConfiguration + @EnableAutoConfiguration + @ComponentScan |
| Excluding an auto-configuration? | `exclude=`, `excludeName=`, or property-based exclusion |
| @Conditional family with examples? | @ConditionalOnProperty/Class/MissingBean/Expression/Java + custom Condition |
