# Part 3 — Load Balancing

> Client-side vs server-side load balancing, algorithms, Ribbon (legacy) vs Spring Cloud LoadBalancer (modern), and where server-side load balancers fit. Interview Q&A at the end.

## What It Is

**What it does:** distributes incoming requests across multiple instances of a service (a "server pool" or "server farm") instead of sending all traffic to one instance — spreading load evenly (or by some rule) so no single instance gets overwhelmed while others sit idle.

**Common algorithms:**
- **Round Robin** — cycle through instances in order, one request each, wrapping around. Simple, assumes all requests and all instances are roughly equal cost.
- **Least Connections** — send the next request to whichever instance currently has the fewest active connections. Better than round robin when request costs vary a lot.
- **Weighted Response Time** — favor instances that have been responding faster recently — adapts to instances that are under more load or running on weaker hardware.
- **Zone Avoidance** — (Netflix Ribbon-specific) avoids routing to instances in an availability zone showing symptoms of a problem, factoring in Amazon AWS zone health.

## Client-Side vs Server-Side Load Balancing

**Client-side load balancing:** the calling service itself decides which instance to route each request to — typically working directly off the list of instances from service discovery (Part 1), with no separate load-balancer component in between.

**Server-side load balancing:** a dedicated component (a reverse proxy, hardware/cloud load balancer, or Kubernetes Service) sits in front of the instance pool; the client always calls one stable address, and the load balancer distributes requests behind the scenes.

| | Client-side | Server-side |
|---|---|---|
| Examples | Ribbon, Spring Cloud LoadBalancer | Nginx, AWS ELB/ALB, Kubernetes Service, HAProxy |
| Extra hop? | No | Yes — through the LB |
| Where does the decision happen? | Inside the calling application | In dedicated infrastructure |
| Coupling | Client needs the instance list + balancing logic | Client just needs one stable name/IP |

> ⚠️ **Pitfall:** this is the exact same client-side/server-side distinction from Part 1's service discovery — that's not a coincidence. Client-side load balancing and client-side service discovery are almost always paired (the client fetches the instance list from the registry, *then* applies a balancing algorithm to pick one) — Ribbon and Eureka were designed to work together for exactly this reason.

## Ribbon (Legacy)

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-ribbon</artifactId>
</dependency>
```
```java
@LoadBalanced
@Bean
RestTemplate restTemplate() {
    return new RestTemplate();
}
```
`@LoadBalanced` makes this `RestTemplate` resolve logical service names (`http://order-service/orders`) through the discovery client and Ribbon's balancing algorithm, instead of requiring a literal IP:port.

> ⚠️ **Pitfall — say this proactively, don't wait to be asked:** Netflix Ribbon is in **maintenance mode** (Spring Cloud officially dropped it from the default stack starting with the 2020.0 release train) — it still works in older codebases, but it's not the answer for a new project. The modern default is **Spring Cloud LoadBalancer**.

## Spring Cloud LoadBalancer (Modern)

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-loadbalancer</artifactId>
</dependency>
```
```java
@LoadBalanced
@Bean
RestTemplate restTemplate() {
    return new RestTemplate();
}
```
**The API surface looks nearly identical on purpose** — `@LoadBalanced` still works the same way — but underneath, Spring Cloud LoadBalancer is a lighter-weight, reactive-friendly (works cleanly with `WebClient` as well as `RestTemplate`), pluggable-algorithm replacement that isn't tied to the (now-unmaintained) Netflix OSS stack.

```java
// WebClient variant — the reactive-stack equivalent
@Bean
@LoadBalanced
WebClient.Builder loadBalancedWebClientBuilder() {
    return WebClient.builder();
}
```

> ⚠️ **Pitfall:** because the migration is mostly a dependency swap with the same `@LoadBalanced` annotation, teams sometimes don't realize they're still pulling in the deprecated Ribbon starter transitively from an old parent POM. If asked "how would you verify which load balancer a service is actually using," the answer is checking the actual resolved dependency tree (`mvn dependency:tree` / `./gradlew dependencies`), not just assuming based on which annotation is in the code.

## Where Server-Side Load Balancers Fit

Client-side balancing (Ribbon/Spring Cloud LoadBalancer) only works well **within** a service mesh where every caller is itself a Spring Cloud application that can run the balancing logic. For traffic **entering** the system from outside (browsers, mobile apps, external partners), you need a server-side load balancer in front — this is exactly the role an **API Gateway** (Part 4) or a cloud load balancer (AWS ALB, GCP Load Balancer) plays, and it's common to see both layers in the same architecture: a server-side LB/gateway at the edge, client-side balancing for internal service-to-service calls.

> ⚠️ **Pitfall:** conflating "load balancer" with "API Gateway" — a load balancer's only job is distributing traffic across instances of the *same* service; a gateway (Part 4) does that plus routing between *different* services, auth, rate limiting, and request transformation. A load balancer is often one small piece of what a gateway does internally, not a replacement for it.

---

## Interview Q&A

**Q: Client-side vs server-side load balancing — what's the actual difference, and which does Ribbon implement?**
Covered above — Ribbon (and its replacement, Spring Cloud LoadBalancer) are client-side: the calling application itself picks which instance to call, typically using the instance list from service discovery.

**Q: Ribbon vs Spring Cloud LoadBalancer — which would you use today, and why?**
Spring Cloud LoadBalancer. Ribbon has been out of the default Spring Cloud stack since the 2020.0 release train and isn't actively developed; Spring Cloud LoadBalancer is the maintained replacement, works with both `RestTemplate` and reactive `WebClient`, and uses the same `@LoadBalanced` annotation so the migration is largely mechanical.

**Q: Name the common load-balancing algorithms and when each is appropriate.**
Covered above — Round Robin (simple, equal-cost assumption), Least Connections (better under variable request cost), Weighted Response Time (adapts to instance performance), Zone Avoidance (Ribbon-specific, AWS-zone-aware).

**Q: Is a load balancer the same thing as an API Gateway?**
No — covered above. A load balancer distributes traffic across instances of the *same* service. A gateway routes between *different* services and adds cross-cutting concerns (auth, rate limiting, transformation) on top — a gateway may use load balancing internally, but it's a superset of responsibility, not the same thing.
