# 🧾 Part 9 — Configuration & Properties

> Neat, point-based format with callout boxes, tables, and icons. Interview Q&A at the end.

---

## 🔌 Dynamically Loading Values in Spring Boot

| Approach | Use when |
|---|---|
| `@Value("${some.property}")` | Injecting a single property value |
| `@ConfigurationProperties(prefix = "app")` | Binding a whole group of related properties to a typed class |
| `Environment` bean | Dynamic/conditional programmatic lookup (`environment.getProperty("key")`) |
| Spring Cloud Config + `@RefreshScope` | Genuinely runtime-dynamic values that change without a restart |

> [!WARNING]
> Plain `@Value`/`@ConfigurationProperties` values are resolved **once** at startup and don't automatically pick up changes afterward — don't imply they're "dynamic" in the live-reloading sense unless paired with `@RefreshScope`/Spring Cloud Config.

---

## 📄 YAML vs Properties Files

| | `.properties` | `.yml` |
|---|---|---|
| Structure | Flat key-value pairs | Hierarchical/nested via indentation |
| Lists/maps | Comma-separated hacks | Native support |
| Readability | Lower for complex config | Higher for nested config |
| Multiple profiles in one file | No | Yes (`---` separators) |

> [!CAUTION]
> YAML's whitespace sensitivity is a real downside — a tab character or misaligned indent can cause a subtle config error that's harder to spot than a typo in a flat `.properties` file.

---

## 📋 Interview Q&A

| Question | Short answer |
|---|---|
| Dynamically loading values? | @Value, @ConfigurationProperties, Environment, or @RefreshScope for true dynamism |
| YAML vs properties — when to prefer one? | YAML for nested config; properties for simple flat config or avoiding whitespace issues |
