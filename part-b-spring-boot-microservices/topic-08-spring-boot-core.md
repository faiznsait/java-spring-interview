# Topic 08: Spring Boot Core Concepts

## Q38. Advantages of Spring & Spring Boot — Starter Parent

### Concept
**Spring Framework** provides the IoC container, DI, AOP, and integration abstractions. **Spring Boot** eliminates XML configuration and boilerplate through auto-configuration, opinionated defaults, and embedded servers.

### Spring vs Spring Boot

| | Spring Framework | Spring Boot |
|---|---|---|
| Configuration | Manual XML / Java config | Auto-configured via `@EnableAutoConfiguration` |
| Server | Deploy WAR to external Tomcat | Embedded Tomcat/Jetty/Undertow — run as JAR |
| Dependencies | Manual version management | Starter POMs manage compatible versions |
| Setup time | Hours | Minutes |

### Spring Boot Starter Parent
```xml
<!-- pom.xml -->
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.2.0</version>
</parent>

<!-- What it gives you for FREE:
     1. Dependency version management — all Spring + 3rd party libs at compatible versions
     2. Plugin configuration — Maven compiler at Java 17, resource filtering
     3. Default encoding UTF-8
     4. sensible test config (Surefire plugin)
-->

<!-- Now you can add starters WITHOUT specifying versions: -->
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>  <!-- No version needed! -->
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
</dependencies>
```

### What a Starter POM Actually Contains
```xml
<!-- spring-boot-starter-web is itself just a POM with these deps: -->
<!-- spring-webmvc, spring-web, spring-boot-starter-tomcat,        -->
<!-- spring-boot-starter-json (Jackson), spring-boot-starter       -->
<!-- (which includes: spring-core, spring-boot-autoconfigure, slf4j)-->

<!-- spring-boot-starter-data-jpa pulls in: -->
<!-- hibernate-core, spring-data-jpa, spring-orm, HikariCP (connection pool) -->
```

### Interview Answer
> "Spring Boot's value is 'convention over configuration' — sensible defaults for 95% of use cases so you don't write XML or wire every bean manually. The Starter Parent is a parent POM that manages dependency versions, so all your starters — web, JPA, security — are guaranteed compatible. You get an embedded Tomcat, auto-configured DataSource, Jackson for JSON, all from one `<dependency>` block. If you need to override a default, you can — but you only configure deviations from the convention."
>
> *Likely follow-up: "What happens when Spring Boot starts? Walk me through the bootstrap sequence."*

### Real Project Usage
- In migration projects from Spring 4.x to Boot 3.x, Starter Parent removed version drift across teams. Earlier, one module upgraded Jackson while another stayed old, causing runtime serialization errors. With Boot starters, compatibility came from a single managed BOM.
- During onboarding, new developers shipped a first endpoint in hours instead of spending days setting XML contexts and servlet container config.

### Common Mistakes to Avoid
- Adding explicit versions for Spring starter dependencies unnecessarily. This often fights Boot's dependency management and creates transitive conflicts.
- Treating starters as magic and not checking transitive dependencies. In interviews, mention that you still inspect dependency trees for security and footprint.

---

## Q39. Spring Boot Application — End-to-End Flow

### From Code to Running Service

```
1. main() → SpringApplication.run()
      ↓
2. Create ApplicationContext (AnnotationConfigServletWebServerApplicationContext)
      ↓
3. Load @PropertySource, application.properties/yml
      ↓
4. Component Scan (@SpringBootApplication → @ComponentScan)
   → Find all @Component, @Service, @Repository, @Controller, @Configuration
      ↓
5. Process @Configuration classes → build BeanDefinitions (blueprint, not instances yet)
      ↓
6. Run BeanFactoryPostProcessors (e.g., PropertyPlaceholderConfigurer → resolve ${...})
      ↓
7. Instantiate singleton beans (constructor → @Autowired → @PostConstruct)
      ↓
8. Run ApplicationRunner / CommandLineRunner beans (startup jobs)
      ↓
9. Start embedded Tomcat → register DispatcherServlet
      ↓
10. Application READY — "Started App in 2.3 seconds"
```

### Request Flow (once running)
```
HTTP Request
    ↓
Embedded Tomcat (accepts socket connection)
    ↓
Filter Chain (Spring Security filters, CORS, logging filters)
    ↓
DispatcherServlet (Front Controller — single entry point)
    ↓
HandlerMapping (which @RequestMapping matches this URL?)
    ↓
HandlerInterceptor (preHandle — auth tokens, logging)
    ↓
@Controller / @RestController method
    ↓
@Service layer → @Repository layer → Database
    ↓
Return value → HttpMessageConverter (JSON via Jackson)
    ↓
HandlerInterceptor (postHandle, afterCompletion)
    ↓
HTTP Response
```

```java
// Minimal Spring Boot app
@SpringBootApplication  // = @Configuration + @EnableAutoConfiguration + @ComponentScan
public class MyApp {
    public static void main(String[] args) {
        SpringApplication.run(MyApp.class, args);
    }
}
```

### Real Project Usage
- This startup flow understanding is critical when debugging long cold starts in containerized deployments. Most delays are usually in external connectivity checks (database, config server, discovery), not in controller logic.
- During incident response, startup phase logs help identify whether the issue is bean creation failure, property binding, or embedded server binding/port conflicts.

### Common Interview Follow-up
"How do you diagnose startup problems quickly?"

Answer pattern:
- Run with `--debug` to get auto-configuration condition report.
- Enable startup tracing (`ApplicationStartup`) to see expensive bean creation.
- Check `Caused by` chain in first bean creation failure and fix root dependency/property issue.

---

## Q40. `@EnableAutoConfiguration` vs `@Configuration`

### Concept
- `@Configuration` marks a class as a source of **manual** bean definitions.
- `@EnableAutoConfiguration` tells Spring Boot to **automatically** configure beans based on what's on the classpath.

### How `@EnableAutoConfiguration` Works Internally
```
1. Reads META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
   (or spring.factories in older versions) from every JAR on the classpath

2. Each entry is a @Configuration class with @ConditionalOn* clauses

3. For example, DataSourceAutoConfiguration runs only IF:
   - javax.sql.DataSource is on the classpath
   - AND no DataSource bean is already manually defined
   (@ConditionalOnClass + @ConditionalOnMissingBean)

4. Spring Boot applies matching auto-configs → beans appear without you writing them
```

```java
// @Configuration: you define the beans manually
@Configuration
public class AppConfig {
    @Bean
    public ObjectMapper objectMapper() {
        return new ObjectMapper()
            .configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
    }
}
// This bean is ALWAYS created regardless of classpath

// @EnableAutoConfiguration: Spring Boot decides what to create
// JacksonAutoConfiguration creates an ObjectMapper IF:
//   - Jackson is on classpath (it is via spring-boot-starter-web)
//   - AND no ObjectMapper bean is already defined (@ConditionalOnMissingBean)
// So if you define your own @Bean ObjectMapper above, the auto-config BACKS OFF

// What library is affected if you don't use it:
// → Jackson won't be auto-configured → no automatic JSON serialization
// → DataSource won't be auto-configured → no DB connection
// → Spring MVC / DispatcherServlet won't be set up → no web layer
```

### `@SpringBootApplication` = the three combined
```java
@Target(ElementType.TYPE)
@SpringBootApplication  // Shorthand for:
// = @Configuration           → this class is a config class
// + @ComponentScan           → scan current package and sub-packages
// + @EnableAutoConfiguration → use Spring Boot magic
public class MyApp { ... }
```

### Interview Answer
> "`@Configuration` is the standard Spring annotation for defining beans manually using `@Bean` methods — it's just a source of bean definitions you fully control. `@EnableAutoConfiguration` is Spring Boot's classpath scanner — it reads auto-configuration classes from each JAR's `spring.factories` (or `AutoConfiguration.imports` in Boot 3), each gated by `@ConditionalOn*` annotations. So if Jackson is on the classpath and you haven't defined your own `ObjectMapper` bean, `JacksonAutoConfiguration` creates one for you. Your manual `@Configuration` beans take priority — defining a bean yourself causes the matching auto-config to back off."
>
> *Likely follow-up: "How do you see which auto-configurations are active?"*
> (Answer: `--debug` flag at startup prints a conditions report showing applied/not-applied auto-configs.)

### Real Project Usage
- Teams commonly override Boot defaults only at boundaries: custom `ObjectMapper`, security filter chain, CORS policy, datasource tuning. The rest should remain convention-driven to keep maintenance cost low.
- In production hardening, condition reports were used to prove why a bean was not created (`@ConditionalOnMissingBean` failing due to duplicate custom bean).

### Common Mistakes to Avoid
- Overriding too many auto-config beans early. This increases upgrade friction between Boot versions.
- Confusing `@ComponentScan` issues with auto-config issues. Missing package scan means bean is never discovered; auto-config conditions are a different layer.

### Externalized Configuration Essentials

Interview-ready precedence (high-level):
1. Command-line args
2. Environment variables
3. `application-{profile}.properties`
4. `application.properties`

```properties
# application.properties
server.port=8080
spring.profiles.active=dev
```

```properties
# application-dev.properties
server.port=8081
```

Key interview point: profiles allow environment-specific behavior without code changes.

---

