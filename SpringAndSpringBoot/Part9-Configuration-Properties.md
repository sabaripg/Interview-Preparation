# Part 9 — Configuration & Properties

> Loading values dynamically, YAML vs properties files. Interview Q&A at the end.

## Dynamically Loading Values in Spring Boot

- `@Value("${some.property}")` — injects a single property value from `application.properties`/`.yml`, environment variables, or command-line args (Spring's layered property source resolution).
- `@ConfigurationProperties(prefix = "app")` — binds a whole group of related properties to a strongly-typed Java class, preferred over many individual `@Value` fields for structured config.
- `Environment` bean — programmatic access (`environment.getProperty("key")`) when you need dynamic/conditional property lookup rather than static injection.
- For genuinely runtime-dynamic values (changing without a restart): integrate Spring Cloud Config with `@RefreshScope`, or a feature-flag service.

> ⚠️ **Pitfall:** plain `@Value`/`@ConfigurationProperties` values are resolved **once** at startup and don't automatically pick up changes afterward — don't imply they're "dynamic" in the live-reloading sense unless paired with `@RefreshScope`/Spring Cloud Config or similar.

## YAML vs Properties Files

**`.properties`** — flat key-value pairs (`app.database.url=...`), simple, no native support for lists/maps beyond comma-separated hacks, less readable for deeply nested configuration.

**`.yml`** — hierarchical/nested structure via indentation, natively supports lists and maps cleanly, generally more readable for complex configuration, supports multiple profile-specific documents in a single file (`---` separators).

**When to prefer which:** YAML for configuration with meaningful nesting/hierarchy (most real Spring Boot apps). `.properties` remains fine for simple flat configuration or when avoiding YAML's whitespace-sensitivity concerns.

> ⚠️ **Pitfall:** YAML's whitespace sensitivity is a real practical downside — a tab character or misaligned indent can cause a subtle config error that's harder to spot than a typo in a flat `.properties` file.

---

## Interview Q&A

**Q: How can you dynamically load values in a Spring Boot application?**
Covered above.

**Q: Key differences between YML and properties files — when to prefer one?**
Covered above.
