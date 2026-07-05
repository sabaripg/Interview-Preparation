# 📡 Part 7 — REST API Design & Error Handling

> Neat, point-based format with callout boxes, tables, and icons. Interview Q&A at the end.

---

## 📊 Handling Millions of Records Through an Endpoint

- Never return millions of records in a single response — implement **pagination** (`Pageable`/`Page<T>` in Spring Data JPA).
- For genuine bulk export: **stream** the response instead of materializing the whole result set in memory — `ResponseBodyEmitter`/`StreamingResponseBody`, or reactive `Flux<T>` with WebFlux, plus a streaming query (JDBC fetch-size tuning, cursor-based/`Stream<T>` JPA query).
- Add proper DB-side indexing/query optimization.
- Consider async/background processing with a job-status endpoint for very large exports.

> [!IMPORTANT]
> Pagination is the first-line answer, but for genuinely "millions of records" scenarios, **streaming** (not just pagination) is the more complete senior answer.

---

## 🔁 Idempotent Methods in REST

| Method | Idempotent? |
|---|---|
| `GET` | ✅ |
| `PUT` | ✅ |
| `DELETE` | ✅ |
| `POST` | ❌ |
| `PATCH` | ❌ (generally) |

- Idempotent = repeating the same request produces the **same result/end state**, no matter how many times.
- Why it matters: clients/proxies/load balancers can safely **retry** idempotent requests after a timeout without risking duplicate side effects.

> [!IMPORTANT]
> `PUT` being idempotent specifically means "replace the resource with this exact representation" — calling it 5 times with the same body leaves the same final state as calling it once. That's the precise technical meaning.

---

## 🔢 Common HTTP Status Codes

| Code | Meaning |
|---|---|
| `200` / `201` / `204` | OK / Created / No Content |
| `400` | Bad Request — malformed/invalid client request |
| `401` | Unauthorized — **who are you** (authentication problem) |
| `403` | Forbidden — **I know who you are, you're not allowed** (authorization problem) |
| `404` | Not Found |
| `409` | Conflict — e.g. optimistic locking version mismatch, duplicate creation |
| `500` | Internal Server Error |
| `503` | Service Unavailable |

> [!IMPORTANT]
> `401` vs `403` is a very commonly confused pair — memorize the exact distinction above.

---

## ✏️ PUT vs PATCH

| | PUT | PATCH |
|---|---|---|
| Scope | Replaces the **entire** resource | Partial update — only specified fields change |
| Idempotent? | ✅ Yes | ❌ Not guaranteed by spec |
| Omitted fields | Typically cleared/reset (full replace semantics) | Left unchanged |

> [!WARNING]
> A client sending `PUT` with only some fields populated (intending a partial update) can accidentally **null out the fields it left out**, if the server implements true replace semantics.

---

## 🗺️ ModelMapper — What It Is and Performance Considerations

- Automatic object-to-object mapping (Entity ↔ DTO) via reflection, reducing boilerplate.
- **Performance considerations:** reflection has real runtime overhead and poor refactoring safety — a field rename can silently break mapping without a compile error, only discovered at runtime.
- Mitigation: cache/reuse `TypeMap` configurations.

> [!TIP]
> Many teams now prefer **MapStruct** — generates mapping code at **compile time** (no runtime reflection), giving better performance and compile-time type safety. Proactively recommending it is a strong senior signal.

---

## 🛡️ Global Exception Handling

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(ResourceNotFoundException ex) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
            .body(new ErrorResponse(ex.getMessage()));
    }
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGeneric(Exception ex) {
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(new ErrorResponse("Unexpected error"));
    }
}
```
- `@RestControllerAdvice` centralizes exception-to-HTTP-response mapping across all controllers.
- More specific `@ExceptionHandler`s take precedence over general ones.

> [!WARNING]
> Always include a generic `Exception.class` fallback — without one, unhandled exceptions fall through to Boot's default error page, which can leak internal details (stack traces in dev profile) to clients in production.

### Customizing Error Responses Further
- `@RestControllerAdvice` + `@ExceptionHandler` — the primary approach.
- A custom `ErrorController` (overriding `/error`) — for errors that bypass your controllers entirely (404s for unmapped routes).
- A consistent `ErrorResponse` DTO (timestamp, status, error code, message, path) uniformly across responses.

> [!CAUTION]
> A `@RestControllerAdvice` only catches exceptions from **within controller execution** — it won't catch errors before a controller is even reached.

---

## 📖 Documenting a REST API (Swagger/OpenAPI)

```java
// build.gradle / pom.xml: springdoc-openapi-starter-webmvc-ui
@Operation(summary = "Get employee by ID")
@GetMapping("/employee/{id}")
public Employee get(@PathVariable Long id) { ... }
```
- `springdoc-openapi` auto-generates an OpenAPI spec from controllers/DTOs, exposing `/v3/api-docs` and `/swagger-ui.html`.
- The spec is machine-readable — can drive client SDK generation or contract testing.

> [!CAUTION]
> Never leave Swagger UI enabled and unauthenticated in production — it exposes your full API surface. Restrict to non-prod profiles or secure it like the rest of the app.

---

## 📦 A Common Response Wrapper Across All REST Endpoints

```java
public record ApiResponse<T>(boolean success, T data, String message, Instant timestamp) {
    public static <T> ApiResponse<T> ok(T data) {
        return new ApiResponse<>(true, data, null, Instant.now());
    }
    public static <T> ApiResponse<T> error(String message) {
        return new ApiResponse<>(false, null, message, Instant.now());
    }
}
```
- Standardizes the envelope shape across every endpoint's success/error responses.
- Pairs naturally with the global exception handler above.

> [!WARNING]
> Resist forcing every endpoint (file downloads, streaming responses, paginated results) into one rigid wrapper — works well for CRUD JSON APIs, but can become an awkward straightjacket elsewhere.

---

## 📋 Interview Q&A

| Question | Short answer |
|---|---|
| Handling millions of records through an endpoint? | Pagination first, streaming for genuine bulk export |
| Idempotent methods in REST? | GET/PUT/DELETE idempotent; POST/PATCH generally not |
| Common HTTP status codes? | See table — know 401 vs 403 precisely |
| PUT vs PATCH? | Full replace (clears omitted fields) vs partial update |
| What is ModelMapper, and its performance concerns? | Reflection-based mapping; MapStruct is the modern compile-time alternative |
| Handling exceptions globally? | `@RestControllerAdvice` + `@ExceptionHandler`, always with a generic fallback |
| Customizing error responses further? | Custom ErrorController for pre-controller errors, consistent ErrorResponse DTO |
| Documenting a REST API? | springdoc-openapi — secure Swagger UI in production |
| A common response wrapper across endpoints? | Generic `ApiResponse<T>` record — know when not to force-fit it |
