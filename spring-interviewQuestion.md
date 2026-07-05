# Spring Interview Questions — Topic-wise

> Source: Spring Interview Question (Medium) — reorganized by topic, question order corrected within each topic. Content unchanged from original.

---

## 1. Configuration & Externalized Properties

**1. How can you read properties in Spring Boot? How many ways exist?**

- **@Value** annotation
- we can also **auto-wire** the `org.springframework.core.env.Environment` bean and access the values defined in the `application.properties`
Example : **env.getProperty("email.smtp.server")**;
- The `@ConfigurationProperties` bind values from `application.properties` to a Java object.
Example: @Configuration Properties("email.smtp") *// prefix*

**@ConfigurationProperties** maps properties from application.properties or application.yml to Java objects. This provides organized access to your application's configuration properties.

### Example

```
@ConfigurationProperties(prefix = "app")  
public class AppConfig {  
    private String name;  
    private String description;  
    // getters and setters  
}
```

**2. If a scheduler is already running at a particular time every day, how can you change its property without restarting the application?**

- **@RefreshScope** annotation is used to enable dynamic configuration properties reloading.
- This means that if you modify the values of a property in the configuration server, Spring Boot will automatically reload the new values in your application without needing to restart it.
- The @RefreshScope annotation is part of the **Spring Cloud Config project**, which provides externalized configuration for distributed systems
- To use **@RefreshScope** in a Spring Boot application, you need to follow these steps:
- Add the Spring Cloud Config Client dependency to your pom.xml file
- add @RefreshScope annotation where reading value from propertie file.
- management.endpoints.web.exposure.include=refresh
- /actuator/refresh

```
import org.springframework.cloud.context.config.annotation.RefreshScope;  
import org.springframework.scheduling.annotation.Scheduled;  
import org.springframework.stereotype.Component;  
import org.springframework.beans.factory.annotation.Value;  

@RefreshScope  
@Component  
public class MyScheduledTask {  

    @Value("${my.task.rate}")  
    private long rate;  

    @Scheduled(fixedRateString = "${my.task.rate}")  
    public void myTask() {  
        System.out.println("Task running at rate: " + rate);  
    }  
}
```

So, changing the property on the config server **alone is not enough** — `@RefreshScope` just makes the bean reloadable when a refresh is triggered.

**27. What is the difference between `application.properties` and `application.yml`?**

```
server.port: 8085  
server.servlet.context-path: /api/v1  
server.servlet.session.cookie.name: acctk  
server.servlet.session.cookie.http-only: true  
server.servlet.session.path: /api/v1  
server.servlet.session.max-age: 30d
```

```
server:  
  port: 8085  
  servlet:  
    context-path: /api/v1  
    session:  
      cookie:  
        name: acctk  
        http-only: true  
  path: /api/v1  
        max-age: 30d
```

**41. How does Spring Boot handle external configuration?**

- **Spring Boot** supports a variety of external configuration options.
- It supports property files (**application.properties or application.yml**) that can be stored in a variety of locations, including the classpath, file system, and external directories.
- Spring Boot also allows environment variables, command-line arguments, and the creation of profiles for various deployment contexts.
- The configuration values can be obtained using the **@Value annotation** or bound to Java objects using the **@ConfigurationProperties annotation**.

---

## 2. Spring Boot Actuator

**3. What are common use cases of Spring Boot Actuator?**

Spring Boot Actuator provides a variety of built-in endpoints. Below are some of the most commonly used ones:

- **/actuator/health**: Displays the health status of the application.
- **/actuator/info**: Displays arbitrary application information.
- **/actuator/metrics**: Shows 'metrics' information for the current application.
- **/actuator/env**: Displays properties from the `Environment`.
- **/actuator/beans**: Displays a complete list of all Spring beans in your application.

---

## 3. Spring Boot Starters

**4. What is the role of Spring Boot Starter dependencies?**

> Spring Developers used to spend a lot of time on Dependency management
>
> Spring Boot Starters were introduced to solve this problem so that the developers can spend more time on actual code than Dependencies.
>
> **Advantage:**
>
> Increase productivity by decreasing the Configuration time for developers.
>
> Managing the POM is easier since the number of dependencies to be added is decreased.
>
> No need to remember the name and version of the dependencies.

---

## 4. Auto-Configuration

**5. What is Auto-Configuration in Spring Boot?**

Spring Boot tries to **automatically configure** beans based on:

The libraries present in your classpath

- Your custom configuration
- Default sensible settings

Example: If you add `spring-boot-starter-web`, Spring Boot automatically configures:

- `DispatcherServlet`
- Jackson JSON support
- Embedded Tomcat
without you explicitly defining them.

**6. How does Spring Boot know which configuration classes to load?**

- Older Versions (Spring Boot ≤ 2.7) ?
- Auto-configuration classes are listed in:

```
META-INF/spring.factories
```

- Inside that file, you'll see something like

```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\  
  org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration,\  
  org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
```

- When Spring Boot starts, it **reads** `spring.factories` to find all classes under `EnableAutoConfiguration` and imports them.

Note :

Spring does not write `spring.factories` dynamically; it is manually written by framework or library authors at build time and packaged inside JARs. Spring only reads it during application startup.

**7. What happens when you exclude an auto-configuration class?**

- When you exclude an auto-configuration class using:

```
@SpringBootApplication(exclude = { DataSourceAutoConfiguration.class })  

OR  

@EnableAutoConfiguration(exclude = { DataSourceAutoConfiguration.class })
```

> you are telling Spring Boot: "When you read that `spring.factories` or `AutoConfiguration.imports(spring boot 3 version)` file, **ignore** this class — don't load it into the application context."

```
Start Application  
   ↓  
Check classpath for auto-config files  
   ↓  
Spring Boot 2.x → spring.factories  
Spring Boot 3.x → AutoConfiguration.imports  
   ↓  
List all config classes  
   ↓  
Remove excluded classes  
   ↓  
Load remaining configurations into Spring context
```

**Key takeaway:**

- The `.factories` / `.imports` file is like a **menu** of possible auto-configurations.
- `exclude` is like saying, "I don't want this dish from the menu."
- In Spring Boot 3.x, the **menu moved from** `spring.factories` **to** `AutoConfiguration.imports` for performance and maintainability.

**8. How can we disable a specific auto-configuration class? Name different approaches.**

- Using `exclude` Attribute in `@EnableAutoConfiguration`

```
import org.springframework.boot.autoconfigure.EnableAutoConfiguration;  
import org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration;  
import org.springframework.boot.SpringApplication;  
import org.springframework.boot.autoconfigure.SpringBootApplication;  

@SpringBootApplication  
@EnableAutoConfiguration(exclude = { DataSourceAutoConfiguration.class })  
public class MyApp {  
    public static void main(String[] args) {  
        SpringApplication.run(MyApp.class, args);  
    }  
}
```

**Note:** `@SpringBootApplication` already has `@EnableAutoConfiguration`, so you can just put `exclude` there:

```
@SpringBootApplication(exclude = { DataSourceAutoConfiguration.class })
```

- Using `excludeName` Attribute

```
@SpringBootApplication(excludeName = { "org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration" })
```

- Using `spring.autoconfigure.exclude` in `application.properties`

```
spring.autoconfigure.exclude=org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
```

This approach keeps the code clean — useful when you want to change configs per environment.

**25. Can you explain auto-configuration in Spring Boot?**

- Spring Boot Auto configuration looks at the jars available on the CLASSPATH and configuration provided by us in the application.properties file and it tries to auto-configure those classes

---

## 5. Embedded Server (Tomcat / Jetty)

**9. Can we override and replace the embedded Tomcat server in Spring Boot? How?**

Option 1️⃣ Replace Tomcat with Jetty ✅ (Most common)

Step 1: Exclude Tomcat

```
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
```

### Step 2: Add Jetty

```
<dependency>  
  <groupId>org.springframework.boot</groupId>  
  <artifactId>spring-boot-starter-jetty</artifactId>  
</dependency>
```

🚀 Result:

- Tomcat removed
- Jetty embedded server used automatically
- we can override and replace the Tomcat embedded server in Spring Boot. To do this, we need to create a bean of type `EmbeddedServletContainerFactory` and configure it in the `@Configuration` class.

---

## 6. @SpringBootApplication Annotation

**10. What does the `@SpringBootApplication` annotation do internally?**

- The `@SpringBootApplication` annotation is a combination of three annotations: `@Configuration`, `@EnableAutoConfiguration`, and `@ComponentScan`.
- `@SpringBootConfiguration` marks the class as a source of Spring bean definitions (like `@Configuration`
- `@EnableAutoConfiguration` triggers Spring Boot's **auto-configuration mechanism**, which scans the classpath and application properties to automatically configure beans
- `@ComponentScan` enables scanning of the package where the annotated class is located, including its sub-packages, to detect Spring-managed components (`@Component`, `@Service`, `@Repository`, etc.).
- It provides a convenient way to configure Spring Boot applications.

---

## 7. CORS (Cross-Origin Resource Sharing)

> Note: These questions were listed in the original article's outline but no detailed answers were written in the source content. Preserved here as-is for completeness.

**11. What is Cross-Origin Resource Sharing (CORS)?**

**12. How does CORS work?**

**13. How can you solve CORS issues in Spring Boot with Spring Security?**

**14. What is a Pre-flight Request and when does it happen?**

**15. What are pre-flight requests in CORS, and how are they handled?**

**16. How can you configure CORS globally in Spring Boot 2.4+ besides using Spring Security?**

**17. Why is configuring CORS in Spring Security essential if Spring Security is on the classpath?**

---

## 8. Dependency Injection & Bean Definition

**18. What is the difference between `@Bean` and stereotype annotations (@Component, @Service, etc.)?**

***Using the @Bean annotation*** to add instances to Spring context enables us to add any kind of object instance as a bean and multiple instances of the same kind to the Spring Context.

***Using stereotype annotation you can create beans*** only for the application class with a specific annotation like @Component. This is preferred for the classes you define in an application and can annotate.

Example: **@Service, @Repository, @Controller**

summary : Use `@Bean` when you need explicit programmatic control or want to register external classes as beans. Use stereotypes for your own application components to leverage automatic detection and semantic clarity.

**19. Will the following code compile? Why or why not?**

```
@Component  
public class Server {  
    private String type= "classicServer";  
    @Autowired  
    private final WebServer webServer;  
}
```

No, this code will **not compile** because:

- In Java, `final` **fields must be initialized either at declaration or inside a constructor**.
- Here, `webServer` is `final` but **no constructor initializes it**, and it's not assigned a value inline.
- Field injection (`@Autowired` on the field) **does not work with final fields** because Spring injects after object construction, and Java requires final fields to be initialized during construction.

**20. How do you fix the problem above? What is Constructor Injection?**

- **Constructor Injection** is the recommended way to inject dependencies by passing them via the class constructor.
- It allows `final` fields to be initialized during object construction, satisfying Java's final variable rules.
- Spring automatically uses the constructor annotated with `@Autowired` (or the single constructor if only one is present) to resolve and inject dependencies.

```
@Component  
public class Server {  

    private String type = "classicServer";  
    private final WebServer webServer;  

    @Autowired  // optional from Spring 4.3+ if there is only one constructor  
    public Server(WebServer webServer) {  
        this.webServer = webServer;  
    }  
}
```

**21. Why prefer Constructor Injection over Field Injection?**

- Enables immutable dependencies (`final` fields).
- Easier to write unit tests (constructor args can be mocked).
- Makes dependencies explicit and required at object creation time.
- Avoids issues related to field injection lifecycle (like injection happening after constructor execution).

---

## 9. Bean Resolution — @Primary & @Qualifier

**22. Explain the use case of `@Primary` annotation in Spring.**

- `@Primary` annotation is used to designate **one bean among multiple candidates of the same type** as the **default bean** to be injected when Spring encounters ambiguity during dependency injection.
- Without `@Primary`, Spring throws a `NoUniqueBeanDefinitionException` because it doesn't know which bean to inject.
- Marking one implementation as `@Primary` resolves this by instructing Spring to prefer that bean **unless another is explicitly specified by** `@Qualifier` **or other means**.

```
@Component  
@Primary  
public class PrimaryPaymentService implements PaymentService {  
    // default implementation  
}  

@Component  
public class SecondaryPaymentService implements PaymentService {  
    // alternate implementation  
}
```

- When injecting `PaymentService` without `@Qualifier`, Spring will inject `PrimaryPaymentService` because it is marked as `@Primary`.

**23. What is the difference between `@Primary` and `@Qualifier`, and how do they work together?**

### Important precedence rules (INTERVIEW GOLD ⭐)

Spring resolves dependency in this order:

1️⃣ `@Qualifier` (highest priority)  
2️⃣ `@Primary`  
3️⃣ **Bean name match**  
4️⃣ ❌ Ambiguous → Exception

- `@Primary` marks a bean as the **default candidate** for injection when multiple beans of the same type exist.
- `@Qualifier` is used to **explicitly specify which bean to inject** by its name or qualifier value, overriding the default selection
- If a bean is marked `@Primary`, it is injected **unless** another bean is selected using `@Qualifier`

```
@Component  
@Primary  
public class PrimaryPaymentService implements PaymentService { ... }  

@Component  
@Qualifier("secondary")  
public class SecondaryPaymentService implements PaymentService { ... }
```

Injection examples:

```
@Autowired  
private PaymentService paymentService; // Injects PrimaryPaymentService  

@Autowired  
@Qualifier("secondary")  
private PaymentService paymentService; // Injects SecondaryPaymentService
```

---

## 10. Bean Scopes

**24. Can you define a prototype bean inside a singleton bean? How?**

- By injecting an ApplicationContext in Singleton bean and then getting the new instance of prototyped scoped bean from this ApplicationContext
- Lookup method injection using @Lookup
- Using scoped proxy

**Example:**

- To inject the ApplicationContext in singleton bean, we can either use @Autowired annotation or we can implement ApplicationContextAware interface

- **Using @LookUp method Injection**

- Spring overrides this method(getPrototypeBean()) at runtime (using CGLIB proxy) to return a new prototype instance on each call.
- Spring dynamically creates a subclass that overrides `getPrototypeBean()` to return a new prototype bean each time.

**Using Scoped Proxy**

- Define the prototype bean with a **scoped proxy**, which Spring injects as a proxy object inside the singleton.
- The proxy delegates each method call to a new prototype instance, ensuring fresh instances per use

**28. Are singleton beans thread-safe?**

- No, singleton beans are not thread-safe, as thread safety is about execution, whereas the singleton is a design pattern focusing on creation.
- Thread safety depends only on the bean implementation itself.

---

## 11. IoC Container — BeanFactory vs ApplicationContext

**26. What is the difference between BeanFactory and ApplicationContext?**

- **BeanFactory** is the most basic version of IOC containers which should be preferred when memory consumption is critical, whereas **ApplicationContext** extends BeanFactory, so we get all benefits of BeanFactory plus advanced features of enterprise applications.
- **BeanFactory** instantiates beans on-demand i.e. when the method getBean(beanName) is called, it is also called **Lazy initializer** whereas **ApplicationContext** instantiates beans at the time of creating the container where bean scope is Singleton, so it is an Eager initializer.
- **BeanFactory** only supports 2 bean scopes, singleton and prototype whereas **ApplicationContext** supports all bean scopes.
- Annotation based dependency injection is not supported by **BeanFactory** whereas **ApplicationContext** supports it.
- **ApplicationContext** provides additional features like MessageSource access (i18n or Internationalization) and Event Publication.

---

## 12. Design Patterns in Spring

**29. Name some design patterns used in the Spring Framework.**

- Singleton Pattern — singleton-scoped beans
- Factory Pattern — Bean Factory classes
- Prototype Pattern — prototype-scoped beans
- Adapter Pattern — Spring Web and Spring MVC
- Proxy Pattern — Spring Aspect-Oriented Programming support
- Template Method Pattern — JdbcTemplate, HibernateTemplate, etc.
- Front Controller — Spring MVC DispatcherServlet
- Data Access Object — Spring DAO support
- Model View Controller — Spring MVC

---

## 13. Asynchronous Processing & Scheduling

**30. How can you enable asynchronous processing in a Spring Boot application?**

- To enable asynchronous processing, you need to use the `@EnableAsync` annotation on a configuration class and the `@Async` annotation on methods that should run asynchronously.

**31. What is the difference between `@Async` and `@Scheduled` in Spring Boot?**

- `@Async` is used for executing methods asynchronously, i.e., in a separate thread without blocking the main thread.
- `@Scheduled` is used for executing methods at specific intervals or schedules, i.e., periodic execution.

**32. How do you schedule a task in Spring Boot?**

- You can schedule a task using the `@Scheduled` annotation. You also need to enable scheduling by using the `@EnableScheduling` annotation on a configuration class.

```
@Service  
public class ScheduledService {  
    @Scheduled(fixedRate = 5000)  
    public void performTask() {  
        System.out.println("Scheduled task executed at: " + System.currentTimeMillis());  
    }  
}
```

**33. How do you configure a thread pool for scheduling tasks in Spring Boot?**

- You can configure a thread pool for scheduling tasks by defining a `TaskScheduler` bean.

```
@Configuration  
@EnableScheduling  
public class SchedulingConfig {  
    @Bean  
    public ThreadPoolTaskScheduler taskScheduler() {  
        ThreadPoolTaskScheduler scheduler = new ThreadPoolTaskScheduler();  
        scheduler.setPoolSize(10);  
        scheduler.setThreadNamePrefix("ScheduledTask-");  
        return scheduler;  
    }  
}
```

**34. How can you handle errors in asynchronous methods?**

- Errors in asynchronous methods can be handled by using a custom `AsyncUncaughtExceptionHandler`.

```
import org.springframework.aop.interceptor.AsyncUncaughtExceptionHandler;  
import org.springframework.context.annotation.Configuration;  
import org.springframework.scheduling.annotation.AsyncConfigurer;  
import org.springframework.scheduling.annotation.EnableAsync;  

import java.lang.reflect.Method;  
import java.util.concurrent.Executor;  

@Configuration  
@EnableAsync  
public class AsyncConfig implements AsyncConfigurer {  
    @Override  
    public Executor getAsyncExecutor() {  
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();  
        executor.setCorePoolSize(2);  
        executor.setMaxPoolSize(2);  
        executor.setQueueCapacity(500);  
        executor.setThreadNamePrefix("AsyncThread-");  
        executor.initialize();  
        return executor;  
    }  

    @Override  
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {  
        return new CustomAsyncExceptionHandler();  
    }  
}  

class CustomAsyncExceptionHandler implements AsyncUncaughtExceptionHandler {  
    @Override  
    public void handleUncaughtException(Throwable throwable, Method method, Object... obj) {  
        System.err.println("Exception message - " + throwable.getMessage());  
        System.err.println("Method name - " + method.getName());  
        for (Object param : obj) {  
            System.err.println("Parameter value - " + param);  
        }  
    }  
}
```

**35. How can you handle errors in scheduled methods?**

- Errors in scheduled methods can be handled by wrapping the method body with try-catch blocks.

```
import org.springframework.scheduling.annotation.Scheduled;  
import org.springframework.stereotype.Service;  

@Service  
public class ScheduledService {  
    @Scheduled(fixedRate = 5000)  
    public void performTask() {  
        try {  
            // Task execution logic  
            System.out.println("Scheduled task executed at: " + System.currentTimeMillis());  
        } catch (Exception e) {  
            // Error handling logic  
            System.err.println("Error occurred: " + e.getMessage());  
        }  
    }  
}
```

**36. What are the different types of scheduling options available in `@Scheduled`?**

Answer: The `@Scheduled` annotation supports several scheduling options:

- `fixedRate`: Executes the task at a fixed interval, specified in milliseconds.
- `fixedDelay`: Executes the task with a fixed delay between the completion of the last invocation and the start of the next.
- `cron`: Executes the task based on a cron expression.

---

## 14. Interceptors & Filters

**42. What is a Spring Boot Interceptor? Give example methods.**

**Request Interceptor :**

- Request Interceptor is an additional component class that intercepts all the incoming and outgoing requests (before any action is performed).
- It has the following 3 methods :
- ***preHandle() :***
- ***postHandle() :***
- ***afterCompletion() :***

```
import jakarta.servlet.http.HttpServletRequest;  
import jakarta.servlet.http.HttpServletResponse;  
import org.springframework.stereotype.Component;  
import org.springframework.web.servlet.HandlerInterceptor;  
import org.springframework.web.servlet.ModelAndView;  

@Component  
public class RequestInterceptor implements HandlerInterceptor {  

    // Request is intercepted by this method before reaching the Controller  
    @Override  
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {  

        //* Business logic just when the request is received and intercepted by this interceptor before reaching the controller  
        try {  
            System.out.println("1 - preHandle() : Before sending request to the Controller");  
            System.out.println("Method Type: " + request.getMethod());  
            System.out.println("Request URL: " + request.getRequestURI());  
        }  
        //* If the Exception is caught, this method will return false  
        catch (Exception e) {  
            e.printStackTrace();  
            return false;  
        }  
        return true;  
    }  

    // Response is intercepted by this method before reaching the client  
    @Override  
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {  
        //* Business logic just before the response reaches the client and the request is served  
        try {  
            System.out.println("2 - postHandle() : After the Controller serves the request (before returning back response to the client)");  
        }  
        catch (Exception e) {  
            e.printStackTrace();  
        }  
    }  

    // This method is called after request & response HTTP communication is done.  
    @Override  
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {  
        //* Business logic after request and response is Completed  
        try {  
            System.out.println("3 - afterCompletion() : After the request and Response is completed");  
        }  
        catch (Exception e) {  
            e.printStackTrace();  
        }  
    }  
}
```

**43. What is a Spring Boot Filter? Give an example.**

- Spring boot Servlet Filter is a component used to intercept & manipulate HTTP requests and responses.
- Servlet filters help in performing pre-processing & post-processing tasks such as:
**Logging and Auditing, Authentication and Authorization, Caching, Routing.**

```
// Servlet Filter Class  
@Component  
public class MyFilter implements Filter {  
  
      // doFilter() Method - To apply Filter's Business logic.  
    @Override  
    public void doFilter(ServletRequest servletRequest,  
                         ServletResponse servletResponse,  
                         FilterChain filterChain)  
        throws IOException, ServletException{  
  
        System.out.println("This is a Servlet doFilter() Method !");  
  
        // Get remote data  
        System.out.println("Remote Host : "+ servletRequest.getRemoteHost());  
        System.out.println("Remote Address : "+ servletRequest.getRemoteAddr());  
  
        // Invoke filterChain to execute the next filter inorder.  
        filterChain.doFilter(servletRequest,servletResponse);  
    }  
}
```

**44. Filters vs HandlerInterceptors — what's the difference?**

- *Filter* is related to the Servlet API and *HandlerIntercepter* is a Spring specific concept.
- *Interceptors* will only execute after *Filters*.
- Fine-grained pre-processing tasks are suitable for *HandlerInterceptors* (authorization checks, etc.)
- Content handling related or generic flows are well-suited for *Filters* (such as multipart forms, zip compression, image handling, logging requests, authentication etc.)
- *Interceptor's* *postHandle* method will allow you to add more model objects to the view but you can not change the HttpServletResponse since it's already committed.
- *Filter's* *doFilter* method is much more versatile than *Interceptor's* *postHandle*. You can change the request or response and pass it to the chain or even block the request processing.
- A *HandlerInterceptor* gives more fine-grained control than a filter because you have access to the actual target "handler". You can even check if the handler method has a specific annotation.

---

## 15. Conditional Annotations

**45. What are Spring Conditional annotations? Give examples.**

- It's used to indicate whether a given Bean is eligible for registration based on a defined condition.

```
@Configuration  
class DevEnvLoggingConfiguration {  
  
    @Bean  
    @Conditional(IsDevEnvCondition.class)  
    LoggingService loggingService() {  
        return new LoggingService();  
    }  
}
```

Predefined Conditional Annotations :

ConditionalOnProperty :

```
@Service  
@ConditionalOnProperty(  
  value="logging.enabled",  
  havingValue = "true",  
  matchIfMissing = true)  
class LoggingService {  
    // ...  
}
```

ConditionalOnExpression:

```
@Service  
@ConditionalOnExpression(  
  "${logging.enabled:true} and '${logging.level}'.equals('DEBUG')"  
)  
class LoggingService {  
    // ...  
}
```

- Now Spring will create the *LoggingService* only when both the *logging.enabled* configuration property is set to *true,* and the *logging.level* is set to *DEBUG.*

ConditionalOnJava:

```
@Service  
@ConditionalOnJava(JavaVersion.EIGHT)  
class LoggingService {  
    // ...  
}
```

CustomCondtion:

```
class Java8Condition implements Condition {  
    @Override  
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {  
        return JavaVersion.getJavaVersion().equals(JavaVersion.EIGHT);  
    }  
}
```

```
@Service  
@Conditional(Java8Condition.class)  
public class Java8DependedService {  
    // ...  
}
```

---

## 16. JPA — Composite Keys

**46. What is a composite key in JPA? How do you map it in an entity?**

```
@Data  
@Embeddable  
public class ProductRatingKey implements Serializable {  
    @Column(name = "product_id")  
    private Long productId;  
  
    @Column(name = "customer_id")  
    private Long customerId;  

  
}
```

Entity :

```
@Data  
@Entity  
public class ProductRating {  

 @EmbeddedId  
 private ProductRatingKey id;  
  
 @ManyToOne  
 @MapsId("product_id")  
 @JoinColumn(name="product_id")  
 private Product product;  
  
 @ManyToOne  
 @MapsId("customer_id")  
 @JoinColumn(name="customer_id")  
 private Customer customer;  
  
  
 private Integer rating;  
}
```
