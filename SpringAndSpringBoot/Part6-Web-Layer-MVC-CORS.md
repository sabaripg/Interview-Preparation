# Part 6 — Web Layer: MVC, Filters, Interceptors, CORS & Reactive

> DispatcherServlet, @RestController vs @Controller, Filters vs Interceptors, CORS end-to-end, WebFlux. Interview Q&A at the end.

## @RestController vs @Controller

**What they are:** `@Controller` is the traditional Spring MVC controller — methods return a **view name** (resolved to a template, e.g. Thymeleaf/JSP) by default, unless individual methods are annotated with `@ResponseBody`. `@RestController` = `@Controller` + `@ResponseBody` applied to every method automatically — all methods return data (JSON/XML) directly serialized into the response body. The standard choice for REST APIs.

## The DispatcherServlet

**What it is:** the central **Front Controller** (design pattern) for Spring MVC — every incoming HTTP request first hits `DispatcherServlet`, which delegates to the appropriate `HandlerMapping` to find the right controller method, invokes it (via a `HandlerAdapter`), and delegates the result to a `ViewResolver` (traditional MVC) or directly serializes it (`@RestController`).

**Why it matters:** it's registered automatically by Spring Boot's auto-configuration when a web starter is on the classpath — you rarely configure it manually.

## Where HandlerMapping Is Stored

**What it is:** `HandlerMapping` beans (e.g. `RequestMappingHandlerMapping`, which maps `@RequestMapping`/`@GetMapping` etc. to URLs) are registered as beans in the `ApplicationContext` itself, like any other Spring-managed component. `DispatcherServlet` holds references to all registered `HandlerMapping` beans, consulting them in order at request time.

## Servlet Filter vs Interceptor

| | `Filter` (Servlet API) | `HandlerInterceptor` (Spring MVC) |
|---|---|---|
| Operates at | Servlet container level, **before** `DispatcherServlet` even sees the request | **Inside** the Spring MVC flow, after `DispatcherServlet` resolves the handler |
| Framework awareness | Spring-agnostic, raw `ServletRequest`/`ServletResponse` | Has access to the actual controller method/handler object |
| Hook points | Chained via `FilterChain` | `preHandle`, `postHandle`, `afterCompletion` |

**When to use which:** `Filter` for concerns that should apply broadly regardless of framework (encoding, CORS at the lowest level, raw request logging). `Interceptor` when you need Spring MVC-specific context (which controller/handler method is about to run) or finer-grained hooks around the MVC lifecycle.

> ⚠️ **Pitfall:** "has access to the resolved handler" is the concrete technical difference — Filters run too early in the pipeline to know which controller method will ultimately handle the request; Interceptors run after that resolution.

## Using an Interceptor in Spring Boot

```java
public class LoggingInterceptor implements HandlerInterceptor {
    public boolean preHandle(HttpServletRequest req, HttpServletResponse res, Object handler) {
        log.info("Incoming: {}", req.getRequestURI());
        return true; // false would short-circuit, controller never runs
    }
}

@Configuration
public class WebConfig implements WebMvcConfigurer {
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new LoggingInterceptor()).addPathPatterns("/api/**");
    }
}
```
Common use cases: request/response logging, authentication/authorization checks before controller invocation, adding common response headers, timing requests.

> ⚠️ **Pitfall:** returning `false` from `preHandle` short-circuits the request entirely — the controller never runs, and you're responsible for writing an appropriate response yourself (e.g. a 401/403) before returning false, or the client gets an empty/incomplete response.

## Spring WebFlux and Reactive Programming

**What it is:** Spring's **reactive**, non-blocking web framework, built on Project Reactor (`Mono<T>` for 0-or-1 results, `Flux<T>` for 0-to-N results) and running on Netty by default (instead of the traditional Servlet/Tomcat thread-per-request model).

**Why it matters:** non-blocking I/O throughout the stack means a small, fixed number of event-loop threads can handle a very large number of concurrent connections, since threads aren't blocked waiting on I/O — well-suited for high-concurrency, I/O-bound workloads (e.g. proxying/aggregating many downstream service calls).

> ⚠️ **Pitfall — the most important practical nuance:** the entire call chain (including the database driver) must be genuinely non-blocking to realize the benefit. Mixing in a blocking JDBC call inside a WebFlux pipeline defeats the purpose and can even be worse than a traditional blocking stack, since it can starve the small event-loop thread pool. Know when WebFlux is and isn't the right choice — I/O-bound high-concurrency vs typical CRUD apps, where virtual threads (see the Multithreading guide) are increasingly a simpler alternative.

---

## CORS (Cross-Origin Resource Sharing)

### What CORS Is and How It Works

**What it does:** a browser-enforced security mechanism controlling which origins (domain+protocol+port) may make cross-origin requests to your server. By default, the browser's Same-Origin Policy blocks such requests unless the server explicitly opts in via response headers.

**Flow:** the browser makes the request, checks the response for CORS headers, and only exposes the response to the calling JavaScript if the headers permit that origin.

**Key headers:** `Access-Control-Allow-Origin` (which origin(s) may access the resource), `Access-Control-Allow-Methods` (permitted HTTP methods), `Access-Control-Allow-Headers` (permitted custom request headers), `Access-Control-Allow-Credentials` (whether cookies/auth headers are allowed cross-origin).

> ⚠️ **Pitfall:** CORS is a **browser-side** protection — it does nothing to stop server-to-server requests or tools like curl/Postman. A common misconception is treating CORS as a server-side security boundary when it's specifically a browser-enforced client protection.

### Preflight Requests — When Exactly Does the Browser Send One?

**What it is:** an automatic `OPTIONS` request the browser sends *before* the real request, asking the server "is this cross-origin request allowed?" — only sent for **non-simple** requests.

**Triggers:** a non-simple HTTP method (`PUT`, `DELETE`, `PATCH`), custom headers beyond the "simple" set, or a `Content-Type` other than `application/x-www-form-urlencoded`, `multipart/form-data`, or `text/plain` — `application/json` always triggers preflight.

If the server's `OPTIONS` response includes valid `Access-Control-Allow-*` headers, the browser sends the actual request; otherwise it's blocked before reaching the server-side handler.

> ⚠️ **Pitfall:** a JSON API using `PUT`/`DELETE` or an `Authorization` header will **always** trigger preflight — this is why REST APIs commonly need explicit CORS configuration even for "simple"-seeming requests.

### Configuring CORS in Spring Boot

```java
// Option 1: method/controller level
@CrossOrigin(origins = "http://localhost:3000")
@GetMapping("/users")
public List<User> getUsers() { ... }

// Option 2: global, no Spring Security
@Configuration
public class CorsConfig implements WebMvcConfigurer {
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**").allowedOrigins("https://app.example.com")
                .allowedMethods("GET","POST","PUT","DELETE").allowCredentials(true);
    }
}

// Option 3: WITH Spring Security on the classpath (required)
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http.cors(cors -> cors.configurationSource(corsConfigurationSource()));
    return http.build();
}
```

**Why Spring Security needs its own CORS config:** if Spring Security is on the classpath, its filter chain runs **before** Spring MVC — so a `WebMvcConfigurer`-based CORS config alone never gets a chance to run, since Security's filters reject or mishandle the preflight `OPTIONS` request first. `http.cors(...)` inside the `SecurityFilterChain` is the only way to make CORS actually work once Security is present.

> ⚠️ **Pitfall:** "I configured CORS but I'm still getting CORS errors" in a Spring Security app is one of the most common real-world Spring misconfigurations — the root cause is almost always CORS configured only at the `WebMvcConfigurer` level while Security's filter chain intercepts the preflight first.

---

## Interview Q&A

**Q: Difference between @RestController and @Controller?**
Covered above.

**Q: Where is HandlerMapping stored in Spring?**
Covered above.

**Q: What is the DispatcherServlet in Spring Framework?**
Covered above.

**Q: Primary difference between a Servlet Filter and DispatcherFilter?**
"DispatcherFilter" isn't a standard Spring/Servlet term — worth naming that directly rather than inventing a confident false answer. The closest legitimate concept is `DispatcherServlet` (not "Filter") — see above.

**Q: Primary difference between a Servlet Filter and an Interceptor?**
Covered above.

**Q: How would you use an Interceptor in a Spring Boot application?**
Covered above.

**Q: What is Spring WebFlux, and how does it support reactive programming?**
Covered above.

**Q: What is CORS, how does it work, and what are the key response headers?**
Covered above.

**Q: What is a preflight request, and exactly when does the browser send one?**
Covered above.

**Q: What are the ways to configure CORS in Spring Boot, and why must Spring Security configure it separately?**
Covered above.
