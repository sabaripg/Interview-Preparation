# Part 7 — REST API Design & Error Handling

> Handling large datasets, idempotency, HTTP status codes, PUT vs PATCH, object mapping, global exception handling, API documentation, response wrappers. Interview Q&A at the end.

## Handling Millions of Records Through an Endpoint

**What to do:** never return millions of records in a single response — implement **pagination** (`Pageable`/`Page<T>` in Spring Data JPA) so the client fetches manageable chunks.

**For genuine bulk export** (not typical API consumption): stream the response instead of materializing the whole result set in memory — `ResponseBodyEmitter`/`StreamingResponseBody`, or a reactive `Flux<T>` with WebFlux, combined with a streaming query (JDBC fetch-size tuning, or a cursor-based/`Stream<T>` JPA query) so records are read and written incrementally.

Also: add appropriate database-side indexing/query optimization for whatever filter/sort the endpoint applies. Consider async/background processing with a job-status endpoint for very large exports ("start export, poll for completion, download when ready").

> ⚠️ **Pitfall:** pagination is the first-line answer, but for genuinely "millions of records" bulk scenarios, streaming (not just pagination) is the more complete senior answer — loading a huge `List<T>` into memory, even if paginated at the DB layer but assembled into one giant in-memory response, defeats the purpose.

## Idempotent Methods in REST

**What it means:** an idempotent operation produces the **same result/end state** no matter how many times it's repeated with the same input — `GET`, `PUT`, `DELETE` are idempotent by HTTP spec convention; `POST` and `PATCH` are generally **not**.

**Why it matters:** clients/proxies/load balancers can safely **retry** idempotent requests after a timeout or ambiguous failure without risking duplicate side effects — retrying a non-idempotent `POST` ("create an order") could create duplicate orders.

> ⚠️ **Pitfall:** `PUT` being idempotent specifically means "replace the resource with this exact representation" — calling it 5 times with the same body leaves the resource in the same final state as calling it once. That's the precise technical meaning, not just "same effect roughly."

## Common HTTP Status Codes

- `200 OK`, `201 Created`, `204 No Content` — successful outcomes.
- `400 Bad Request` (malformed/invalid client request), `401 Unauthorized` (missing/invalid authentication), `403 Forbidden` (authenticated but not permitted), `404 Not Found`, `409 Conflict` (e.g. optimistic locking version mismatch, or duplicate resource creation).
- `500 Internal Server Error`, `503 Service Unavailable`.

> ⚠️ **Pitfall:** know the precise distinction between `401` (who are you — authentication problem) and `403` (I know who you are, you're just not allowed — authorization problem) — a very commonly confused pair.

## PUT vs PATCH

**PUT** — replaces the **entire** resource with the provided representation; idempotent; fields omitted from the request body are typically treated as being cleared/reset (full replacement semantics). **PATCH** — applies a **partial** update, modifying only specified fields, leaving the rest unchanged; not guaranteed idempotent by spec (though simple field-level patches usually are in practice).

> ⚠️ **Pitfall:** the "omitted fields get cleared" behavior of `PUT` is a common real-world gotcha — a client sending a `PUT` with only some fields populated (intending a partial update) can accidentally null out the fields it left out, if the server implements true replace semantics rather than lenient merge behavior.

## ModelMapper — What It Is and Performance Considerations

**What it does:** a library for automatic object-to-object mapping (Entity → DTO, DTO → Entity) based on matching field names/types via reflection, reducing hand-written boilerplate.

**Performance considerations:** reflection-based mapping has real runtime overhead per call, and worse refactoring safety, compared to hand-written or compile-time-generated mapping — a field rename on either side can silently break mapping without a compile error, only discovered at runtime or in tests. Mitigation: cache/reuse `TypeMap` configurations rather than reconfiguring per call.

> ⚠️ **Pitfall:** many teams now prefer **MapStruct** instead, which generates actual mapping code at **compile time** (no runtime reflection) — better performance and compile-time type safety (a typo/mismatch is caught at build time). Proactively recommending MapStruct as the modern alternative is a strong senior signal, rather than just describing ModelMapper's limitations in isolation.

## Global Exception Handling

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
**What it does:** `@RestControllerAdvice` (or `@ControllerAdvice` + `@ResponseBody`) applies globally across all controllers — centralizes exception-to-HTTP-response mapping instead of repeating try/catch in every controller. `@ExceptionHandler` methods can target specific exception types, with more specific handlers taking precedence over general ones.

> ⚠️ **Pitfall:** always include a generic `Exception.class` fallback handler — without one, unhandled exceptions fall through to Spring Boot's default error page/response, which can leak internal details (stack traces in dev profile) that shouldn't reach clients in production.

## Customizing Error Responses Further

- `@RestControllerAdvice` + `@ExceptionHandler` — the primary, most flexible approach.
- A custom `ErrorController` (overriding the default `/error` mapping) for lower-level control over Boot's built-in fallback error handling — useful for errors that bypass your controllers entirely (e.g. 404s for non-existent routes).
- A consistent `ErrorResponse` DTO structure (timestamp, status, error code, message, path) applied uniformly across all error responses.

> ⚠️ **Pitfall:** a `@RestControllerAdvice` only catches exceptions thrown from within controller method execution — it won't catch errors that happen *before* a controller is even reached (a 404 for an unmapped URL, or a filter-level failure). Those need the `ErrorController` override or filter-level handling instead.

## Documenting a REST API (Swagger/OpenAPI)

```java
// build.gradle / pom.xml: springdoc-openapi-starter-webmvc-ui
@Operation(summary = "Get employee by ID")
@GetMapping("/employee/{id}")
public Employee get(@PathVariable Long id) { ... }
```
**What it does:** `springdoc-openapi` (the modern successor to the older SpringFox) auto-generates an OpenAPI spec by scanning controllers, `@RequestMapping` annotations, and DTOs — exposing both the raw spec (`/v3/api-docs`) and an interactive UI (`/swagger-ui.html`). Annotations like `@Operation`, `@Parameter`, `@Schema` enrich the auto-generated docs. The generated spec is also machine-readable, so it can drive client SDK generation or contract testing.

> ⚠️ **Pitfall:** never leave Swagger UI enabled and unauthenticated in production — it exposes your full API surface (including inadvertently unsecured internal/admin endpoints) to anyone who finds the URL. Restrict it to non-prod profiles or put it behind the same authentication as the rest of the app.

## A Common Response Wrapper Across All REST Endpoints

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
**What it does:** standardizes the envelope shape (success flag, payload, message, timestamp) across every endpoint's success/error responses, so consumers write one consistent parsing path. Pairs naturally with the global exception handler above — `@ExceptionHandler` methods return `ApiResponse.error(...)` uniformly.

> ⚠️ **Pitfall:** resist forcing every possible endpoint (file downloads, streaming responses, paginated results with their own envelope needs) into one rigid wrapper shape — it works well for simple CRUD JSON APIs but can become an awkward straightjacket for endpoints with genuinely different response semantics.

---

## Interview Q&A

**Q: Delay fetching millions of records through an endpoint — how do you handle it?**
Covered above.

**Q: What are idempotent methods in REST, and why are they important?**
Covered above.

**Q: Tell me some common HTTP status codes.**
Covered above.

**Q: Difference between PUT and PATCH?**
Covered above.

**Q: What is ModelMapper in Java, and why is it used?**
Covered above.

**Q: Performance considerations when using ModelMapper in large-scale applications?**
Covered above.

**Q: How to handle exceptions globally in a Spring Boot app?**
Covered above.

**Q: How do you customize error responses in Spring Boot?**
Covered above.

**Q: How do you document a Spring Boot REST API (Swagger/OpenAPI)?**
Covered above.

**Q: How would you generalize a common response wrapper across all your REST endpoints?**
Covered above.
