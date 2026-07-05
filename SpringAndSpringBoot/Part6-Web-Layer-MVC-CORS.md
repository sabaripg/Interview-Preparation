# 🌐 Part 6 — Web Layer: MVC, Filters, Interceptors, CORS & Reactive

> Neat, point-based format with callout boxes, tables, and icons. Interview Q&A at the end.

---

## 🎯 @RestController vs @Controller

| | `@Controller` | `@RestController` |
|---|---|---|
| Return value | A **view name**, resolved to a template (Thymeleaf/JSP) | JSON/XML data, serialized directly into the response body |
| Needs `@ResponseBody`? | Yes, per method, to return raw data | No — it's `@Controller` + `@ResponseBody` on every method automatically |
| Standard use | Traditional server-rendered MVC | REST APIs |

---

## 🚦 The DispatcherServlet

- The central **Front Controller** for Spring MVC.
- Every incoming HTTP request hits it first → delegates to `HandlerMapping` → invokes the controller (via `HandlerAdapter`) → delegates to a `ViewResolver` (or serializes directly for `@RestController`).
- Registered automatically by Boot's auto-configuration when a web starter is on the classpath.

## 📍 Where HandlerMapping Is Stored

- `HandlerMapping` beans (e.g. `RequestMappingHandlerMapping`) are registered as beans in the `ApplicationContext` — like any other component.
- `DispatcherServlet` holds references to all of them, consulting them in order at request time.

---

## 🧱 Servlet Filter vs Interceptor

| | `Filter` (Servlet API) | `HandlerInterceptor` (Spring MVC) |
|---|---|---|
| Operates at | Servlet container level, **before** `DispatcherServlet` sees the request | **Inside** the MVC flow, after the handler is resolved |
| Framework awareness | Spring-agnostic, raw `ServletRequest`/`ServletResponse` | Has access to the actual controller method/handler |
| Hook points | Chained via `FilterChain` | `preHandle`, `postHandle`, `afterCompletion` |
| Use for | Encoding, low-level CORS, raw request logging | Auth checks needing handler context, finer MVC lifecycle hooks |

> [!IMPORTANT]
> "Has access to the resolved handler" is the concrete technical difference — Filters run too early to know which controller method will ultimately handle the request; Interceptors run after that resolution.

### Using an Interceptor in Spring Boot

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

> [!WARNING]
> Returning `false` from `preHandle` short-circuits the request entirely — the controller never runs, and **you're** responsible for writing an appropriate response (e.g. 401/403) before returning `false`, or the client gets an empty/incomplete response.

---

## 🌊 Spring WebFlux and Reactive Programming

- Spring's **reactive**, non-blocking web framework — built on Project Reactor (`Mono<T>` for 0-or-1, `Flux<T>` for 0-to-N), running on Netty by default.
- Non-blocking I/O means a small, fixed number of event-loop threads handle huge numbers of concurrent connections — great for high-concurrency, I/O-bound workloads.

> [!CAUTION]
> The **entire call chain (including the database driver) must be genuinely non-blocking** to realize the benefit. Mixing a blocking JDBC call into a WebFlux pipeline defeats the purpose and can starve the small event-loop thread pool — a genuine anti-pattern, not just suboptimal. Know when WebFlux is right (I/O-bound high-concurrency) vs a typical CRUD app (virtual threads are increasingly a simpler alternative).

---

## 🚧 CORS (Cross-Origin Resource Sharing)

### What It Is and How It Works
- A **browser-enforced** security mechanism controlling which origins may make cross-origin requests to your server.
- Flow: browser makes the request → checks response headers → only exposes the response to calling JavaScript if headers permit that origin.

| Header | Purpose |
|---|---|
| `Access-Control-Allow-Origin` | Which origin(s) may access the resource |
| `Access-Control-Allow-Methods` | Permitted HTTP methods |
| `Access-Control-Allow-Headers` | Permitted custom request headers |
| `Access-Control-Allow-Credentials` | Whether cookies/auth headers are allowed cross-origin |

> [!CAUTION]
> CORS is a **browser-side** protection — it does nothing to stop server-to-server requests or tools like curl/Postman. Don't treat it as a server-side security boundary.

### Preflight Requests — Exactly When Does the Browser Send One?

- An automatic `OPTIONS` request sent **before** the real request, for **non-simple** requests only.

**Triggers:**
- Non-simple HTTP method (`PUT`, `DELETE`, `PATCH`).
- Custom headers beyond the "simple" set.
- `Content-Type` other than `application/x-www-form-urlencoded`, `multipart/form-data`, or `text/plain` — **`application/json` always triggers preflight**.

> [!IMPORTANT]
> A JSON API using `PUT`/`DELETE` or an `Authorization` header will **always** trigger preflight — this is why REST APIs commonly need explicit CORS configuration even for "simple"-seeming requests.

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

> [!CAUTION]
> If Spring Security is on the classpath, its filter chain runs **before** Spring MVC — a `WebMvcConfigurer`-based CORS config alone never gets a chance to run, since Security's filters reject the preflight `OPTIONS` first. `http.cors(...)` inside the `SecurityFilterChain` is the only way to make CORS work once Security is present. "I configured CORS but still get CORS errors" is one of the most common real-world Spring misconfigurations.

---

## 📋 Interview Q&A

| Question | Short answer |
|---|---|
| @RestController vs @Controller? | `@RestController` = `@Controller` + `@ResponseBody` on every method |
| Where is HandlerMapping stored? | As a bean in the ApplicationContext, referenced by DispatcherServlet |
| What is DispatcherServlet? | The Front Controller for Spring MVC |
| Filter vs Interceptor? | Filter runs before DispatcherServlet, no handler context; Interceptor runs inside MVC flow, has handler context |
| Using an Interceptor? | Implement HandlerInterceptor, register via WebMvcConfigurer |
| What is Spring WebFlux? | Reactive, non-blocking web framework on Project Reactor + Netty |
| What is CORS and its key headers? | Browser-enforced origin control; Allow-Origin/Methods/Headers/Credentials |
| What triggers a preflight request? | Non-simple method, custom headers, or non-simple Content-Type (JSON always) |
| Configuring CORS with Spring Security present? | Must configure via `http.cors()` in the SecurityFilterChain, not just WebMvcConfigurer |
