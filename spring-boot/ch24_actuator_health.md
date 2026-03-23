# Chapter 24: Actuator & Health

## Build Challenge

| Dimension | State |
|-----------|-------|
| **Current State** | The application exposes user-defined `@Controller` endpoints but has no built-in observability -- there is no way to ask the running application "are you healthy?", "what is your version?", or "what are your metrics?" |
| **Limitation** | Without a management endpoint subsystem, operators must build ad-hoc health checks, and monitoring tools have no standard HTTP paths to probe |
| **Objective** | Build an actuator subsystem that discovers `@Endpoint` beans, introspects their `@ReadOperation`/`@WriteOperation` methods, and maps them to `/actuator/*` HTTP paths -- delivering health, info, and metrics endpoints through auto-configuration |

---

## 24.1 The Integration Point: Multiple HandlerMappings + prepareBeanFactory

The integration point for Actuator is **the DispatcherServlet gaining support for multiple HandlerMappings**. This connects the actuator endpoint discovery subsystem to the MVC request dispatch pipeline.

Two existing files are modified to make this work:

**`DispatcherServlet.java`** [MODIFIED] -- `List<HandlerMapping>` + bean discovery

```java
public class DispatcherServlet extends HttpServlet {

    private final List<HandlerMapping> handlerMappings = new ArrayList<>();
    // was: private HandlerMapping handlerMapping;

    private void initHandlerMappings() {
        // Default handler mapping for @Controller beans
        this.handlerMappings.add(new RequestMappingHandlerMapping(applicationContext));

        // Discover additional HandlerMapping beans from the ApplicationContext
        try {
            DefaultBeanFactory factory =
                    ((AnnotationConfigApplicationContext) applicationContext).getBeanFactory();
            Map<String, HandlerMapping> mappingBeans =
                    factory.getBeansOfType(HandlerMapping.class);
            for (HandlerMapping mapping : mappingBeans.values()) {
                if (!this.handlerMappings.contains(mapping)) {
                    this.handlerMappings.add(mapping);
                }
            }
        } catch (Exception e) {
            // No additional handler mappings found -- that's fine
        }
    }
}
```

The dispatch loop now iterates all mappings, first match wins:

```java
private void doDispatch(HttpServletRequest request, HttpServletResponse response)
        throws Exception {
    HandlerMethod handler = null;
    for (HandlerMapping mapping : handlerMappings) {
        handler = mapping.getHandler(request);
        if (handler != null) {
            break;
        }
    }
    // ...
}
```

**`AnnotationConfigApplicationContext.java`** [MODIFIED] -- `prepareBeanFactory()`

```java
@Override
public void refresh() {
    try {
        // Step 0: Register infrastructure beans (context itself)
        prepareBeanFactory();

        // Step 1: Process @Configuration classes ...
        invokeBeanFactoryPostProcessors();
        // ...
    }
}

private void prepareBeanFactory() {
    this.beanFactory.registerSingleton("applicationContext", this);
}
```

**Direction:** This integration point connects **the actuator endpoint discovery** (new) with **the DispatcherServlet dispatch pipeline** (Feature 10) and **auto-configuration** (Feature 13). The `List<HandlerMapping>` is the pivot -- it lets the `ActuatorHandlerMapping` sit alongside the `RequestMappingHandlerMapping` without modifying either. From this integration point, we need to build:

1. Endpoint annotations (`@Endpoint`, `@ReadOperation`, `@WriteOperation`, `@Selector`)
2. Health system (`Status`, `Health`, `HealthIndicator`, `StatusAggregator`, `HealthEndpoint`)
3. Info and Metrics endpoints
4. Endpoint discovery (`EndpointDiscoverer`) and web mapping (`ActuatorHandlerMapping`)
5. Auto-configuration that wires everything together

**Why `prepareBeanFactory()`?** Auto-configuration `@Bean` methods need the `ApplicationContext` as a parameter to discover health indicators and info contributors by type. By registering the context itself as a singleton bean, we enable `@Bean` methods to declare `ApplicationContext` as a dependency. This mirrors the real Spring Framework's `AbstractApplicationContext.prepareBeanFactory()` (line 715), which registers the context, environment, system properties, and system environment as resolvable dependencies.

---

## 24.2 Endpoint Annotations

### @Endpoint -- Marking a Management Bean

**New file:** `iris-boot-actuator/src/main/java/com/iris/boot/actuate/endpoint/annotation/Endpoint.java`

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface Endpoint {

    /**
     * The unique identifier for this endpoint (e.g., "health", "info", "metrics").
     * Used as the URL path segment under /actuator/.
     */
    String id();
}
```

The `id()` attribute drives both bean discovery and URL routing. `@Endpoint(id = "health")` maps to `GET /actuator/health`.

### @ReadOperation -- HTTP GET

**New file:** `iris-boot-actuator/src/main/java/com/iris/boot/actuate/endpoint/annotation/ReadOperation.java`

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface ReadOperation {
}
```

Marks a method as idempotent and safe. The return value is serialized to JSON. If the method returns `null`, the response is 404.

### @WriteOperation -- HTTP POST

**New file:** `iris-boot-actuator/src/main/java/com/iris/boot/actuate/endpoint/annotation/WriteOperation.java`

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface WriteOperation {
}
```

Marks a method as state-modifying. If the method returns `null`, the response is 204 (No Content).

### @Selector -- Path Variable

**New file:** `iris-boot-actuator/src/main/java/com/iris/boot/actuate/endpoint/annotation/Selector.java`

```java
@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
public @interface Selector {
}
```

Marks a parameter as a path variable extracted from the URL. For example, `@ReadOperation public MetricDescriptor metric(@Selector String metricName)` maps to `GET /actuator/metrics/{metricName}`.

> **Insight -- Annotation-driven metadata:** These four annotations form a declarative metadata system separate from `@RequestMapping`. The endpoint author says *what* each method does (`@ReadOperation`) and *what* parameters come from the path (`@Selector`), while the infrastructure decides *how* to expose them over HTTP. This separation is why the same `@Endpoint` class can be exposed over both HTTP and JMX in the real framework.

---

## 24.3 Health System

### Status -- The Status Code

**New file:** `iris-boot-actuator/src/main/java/com/iris/boot/actuate/health/Status.java`

```java
public final class Status {

    public static final Status UP = new Status("UP");
    public static final Status DOWN = new Status("DOWN");
    public static final Status OUT_OF_SERVICE = new Status("OUT_OF_SERVICE");
    public static final Status UNKNOWN = new Status("UNKNOWN");

    private final String code;

    public Status(String code) {
        Objects.requireNonNull(code, "Status code must not be null");
        this.code = code;
    }

    public String getCode() { return this.code; }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Status other)) return false;
        return this.code.equals(other.code);
    }

    @Override
    public int hashCode() { return this.code.hashCode(); }

    @Override
    public String toString() { return this.code; }
}
```

Four well-known statuses are predefined as constants. Custom statuses like `new Status("DEGRADED")` are also supported.

### Health -- Status + Details

**New file:** `iris-boot-actuator/src/main/java/com/iris/boot/actuate/health/Health.java`

```java
public final class Health {

    private final Status status;
    private final Map<String, Object> details;

    private Health(Status status, Map<String, Object> details) {
        this.status = status;
        this.details = Collections.unmodifiableMap(details);
    }

    public Status getStatus() { return this.status; }
    public Map<String, Object> getDetails() { return this.details; }

    // Static factory methods
    public static Builder up() { return new Builder(Status.UP); }
    public static Builder down() { return new Builder(Status.DOWN); }
    public static Builder down(Throwable ex) { return down().withException(ex); }
    public static Builder unknown() { return new Builder(Status.UNKNOWN); }
    public static Builder outOfService() { return new Builder(Status.OUT_OF_SERVICE); }
    public static Builder status(Status status) { return new Builder(status); }

    public static class Builder {
        private final Status status;
        private final Map<String, Object> details = new LinkedHashMap<>();

        private Builder(Status status) { this.status = status; }

        public Builder withDetail(String key, Object value) {
            this.details.put(key, value);
            return this;
        }

        public Builder withException(Throwable ex) {
            this.details.put("error", ex.getClass().getName() + ": " + ex.getMessage());
            return this;
        }

        public Health build() { return new Health(this.status, this.details); }
    }
}
```

Usage: `Health.up().withDetail("database", "PostgreSQL").withDetail("responseTime", "12ms").build()`.

### HealthIndicator -- The Strategy Interface

**New file:** `iris-boot-actuator/src/main/java/com/iris/boot/actuate/health/HealthIndicator.java`

```java
@FunctionalInterface
public interface HealthIndicator {
    Health health();
}
```

Each `HealthIndicator` bean reports the health of one subsystem (database, disk space, external service). Being a `@FunctionalInterface`, simple indicators can be lambdas:

```java
@Bean
public HealthIndicator diskSpaceHealth() {
    return () -> Health.up().withDetail("free", "120GB").build();
}
```

### StatusAggregator -- Worst Status Wins

**New file:** `iris-boot-actuator/src/main/java/com/iris/boot/actuate/health/StatusAggregator.java`

```java
public class StatusAggregator {

    private static final List<String> DEFAULT_ORDER = List.of(
            "DOWN", "OUT_OF_SERVICE", "UNKNOWN", "UP"
    );

    public Status getAggregateStatus(Set<Status> statuses) {
        if (statuses.isEmpty()) {
            return Status.UNKNOWN;
        }

        int worstIndex = Integer.MAX_VALUE;
        Status worstStatus = null;

        for (Status status : statuses) {
            int index = DEFAULT_ORDER.indexOf(status.getCode());
            if (index == -1) {
                index = 2; // Unknown codes treated as between OUT_OF_SERVICE and UNKNOWN
            }
            if (index < worstIndex) {
                worstIndex = index;
                worstStatus = status;
            }
        }

        return worstStatus != null ? worstStatus : Status.UNKNOWN;
    }
}
```

The aggregation uses a fixed severity ordering: DOWN > OUT_OF_SERVICE > UNKNOWN > UP. The lowest-index status (worst) wins. If one indicator reports UP and another reports DOWN, the aggregate is DOWN.

### HealthEndpoint -- The @Endpoint Bean

**New file:** `iris-boot-actuator/src/main/java/com/iris/boot/actuate/health/HealthEndpoint.java`

```java
@Endpoint(id = "health")
public class HealthEndpoint {

    private final Map<String, HealthIndicator> healthIndicators;
    private final StatusAggregator statusAggregator;

    public HealthEndpoint(Map<String, HealthIndicator> healthIndicators,
                          StatusAggregator statusAggregator) {
        this.healthIndicators = healthIndicators;
        this.statusAggregator = statusAggregator;
    }

    @ReadOperation
    public Map<String, Object> health() {
        Map<String, Object> result = new LinkedHashMap<>();

        if (healthIndicators.isEmpty()) {
            result.put("status", Status.UP.getCode());
            return result;
        }

        // Collect health from each indicator
        Map<String, Map<String, Object>> components = new LinkedHashMap<>();
        for (Map.Entry<String, HealthIndicator> entry : healthIndicators.entrySet()) {
            try {
                Health health = entry.getValue().health();
                components.put(entry.getKey(), healthToMap(health));
            } catch (Exception ex) {
                Health errorHealth = Health.down(ex).build();
                components.put(entry.getKey(), healthToMap(errorHealth));
            }
        }

        // Aggregate statuses
        Set<Status> statuses = components.values().stream()
                .map(m -> new Status((String) m.get("status")))
                .collect(Collectors.toSet());
        Status aggregateStatus = statusAggregator.getAggregateStatus(statuses);

        result.put("status", aggregateStatus.getCode());
        result.put("components", components);
        return result;
    }

    private Map<String, Object> healthToMap(Health health) {
        Map<String, Object> map = new LinkedHashMap<>();
        map.put("status", health.getStatus().getCode());
        if (!health.getDetails().isEmpty()) {
            map.put("details", health.getDetails());
        }
        return map;
    }
}
```

Key design choices:
- **Exception safety:** If a `HealthIndicator` throws, it is caught and reported as DOWN with an error detail -- the endpoint never crashes.
- **Aggregation:** All individual statuses are collected and fed to the `StatusAggregator` to compute the overall status.
- **Map-based response:** The `health()` method returns a `Map` rather than a typed DTO, keeping the JSON serialization simple.

Example response at `GET /actuator/health`:
```json
{
  "status": "UP",
  "components": {
    "diskSpace": { "status": "UP", "details": { "free": "120GB" } },
    "db": { "status": "UP", "details": { "database": "PostgreSQL" } }
  }
}
```

---

## 24.4 Info & Metrics Endpoints

### InfoContributor and InfoEndpoint

**New file:** `iris-boot-actuator/src/main/java/com/iris/boot/actuate/info/InfoContributor.java`

```java
@FunctionalInterface
public interface InfoContributor {
    void contribute(Map<String, Object> info);
}
```

Each contributor adds its own section to the info response. Contributors are `@FunctionalInterface` -- expressible as lambdas:

```java
@Bean
public InfoContributor appInfoContributor() {
    return builder -> builder.put("app", Map.of("name", "my-service", "version", "1.2.3"));
}
```

**New file:** `iris-boot-actuator/src/main/java/com/iris/boot/actuate/info/InfoEndpoint.java`

```java
@Endpoint(id = "info")
public class InfoEndpoint {

    private final List<InfoContributor> contributors;

    public InfoEndpoint(List<InfoContributor> contributors) {
        this.contributors = contributors;
    }

    @ReadOperation
    public Map<String, Object> info() {
        Map<String, Object> info = new LinkedHashMap<>();
        for (InfoContributor contributor : contributors) {
            contributor.contribute(info);
        }
        return info;
    }
}
```

Each contributor writes into a shared mutable map. The merged result is the response.

### MetricsEndpoint -- Bridging Micrometer to HTTP

**New file:** `iris-boot-actuator/src/main/java/com/iris/boot/actuate/metrics/MetricsEndpoint.java`

```java
@Endpoint(id = "metrics")
public class MetricsEndpoint {

    private final MeterRegistry registry;

    public MetricsEndpoint(MeterRegistry registry) {
        this.registry = registry;
    }

    @ReadOperation
    public Map<String, Object> listNames() {
        TreeSet<String> names = new TreeSet<>();
        for (Meter meter : registry.getMeters()) {
            names.add(meter.getId().getName());
        }
        return Map.of("names", names);
    }

    @ReadOperation
    public Map<String, Object> metric(@Selector String metricName) {
        List<Meter> matchingMeters = new ArrayList<>();
        for (Meter meter : registry.getMeters()) {
            if (meter.getId().getName().equals(metricName)) {
                matchingMeters.add(meter);
            }
        }

        if (matchingMeters.isEmpty()) {
            return null; // → 404
        }

        // Measurements from the first matching meter
        Meter firstMeter = matchingMeters.get(0);
        List<Map<String, Object>> measurements = getMeasurements(firstMeter);

        // Available tags across all matching meters
        Map<String, TreeSet<String>> tagValues = new LinkedHashMap<>();
        for (Meter meter : matchingMeters) {
            for (Tag tag : meter.getId().getTags()) {
                tagValues.computeIfAbsent(tag.getKey(), k -> new TreeSet<>())
                        .add(tag.getValue());
            }
        }
        // ...build result map with name, measurements, availableTags...
        return result;
    }
}
```

The `MetricsEndpoint` has **two `@ReadOperation` methods** -- one without `@Selector` (list all names) and one with `@Selector` (query a specific metric). The `EndpointDiscoverer` distinguishes them by the presence of `@Selector` parameters, and the `ActuatorHandlerMapping` routes accordingly:

- `GET /actuator/metrics` calls `listNames()`
- `GET /actuator/metrics/http.server.requests` calls `metric("http.server.requests")`

This is how the `@Selector` annotation creates sub-resource routing without traditional `@RequestMapping`.

---

## 24.5 Endpoint Discovery & Web Mapping

### EndpointDiscoverer -- Scanning for @Endpoint Beans

**New file:** `iris-boot-actuator/src/main/java/com/iris/boot/actuate/endpoint/EndpointDiscoverer.java`

```java
public class EndpointDiscoverer {

    public Map<String, DiscoveredEndpoint> discoverEndpoints(ApplicationContext context) {
        Map<String, DiscoveredEndpoint> endpoints = new LinkedHashMap<>();
        DefaultBeanFactory factory =
                ((AnnotationConfigApplicationContext) context).getBeanFactory();

        for (String name : factory.getBeanDefinitionNames()) {
            Object bean;
            try {
                bean = factory.getBean(name);
            } catch (Exception e) {
                continue;
            }

            Endpoint endpoint = bean.getClass().getAnnotation(Endpoint.class);
            if (endpoint != null) {
                DiscoveredEndpoint discovered = introspect(endpoint.id(), bean);
                endpoints.put(endpoint.id(), discovered);
            }
        }

        return endpoints;
    }
}
```

The discovery process:
1. Iterate all bean definitions in the bean factory
2. For each bean whose class is annotated with `@Endpoint`, extract the endpoint ID
3. Scan the class's methods for `@ReadOperation` and `@WriteOperation`
4. For each operation, check if it has a `@Selector` parameter
5. Build a `DiscoveredEndpoint` with all its operations

The introspection creates `EndpointOperation` records:

```java
public record EndpointOperation(
        Object bean,
        Method method,
        String httpMethod,    // "GET" or "POST"
        boolean hasSelector   // true if any parameter has @Selector
) {}

public record DiscoveredEndpoint(
        String id,
        Object bean,
        List<EndpointOperation> operations
) {
    public EndpointOperation findOperation(String httpMethod, boolean hasSelector) {
        for (EndpointOperation op : operations) {
            if (op.httpMethod().equals(httpMethod) && op.hasSelector() == hasSelector) {
                return op;
            }
        }
        return null;
    }
}
```

### ActuatorHandlerMapping -- Self-Dispatching Pattern

**New file:** `iris-boot-actuator/src/main/java/com/iris/boot/actuate/endpoint/web/ActuatorHandlerMapping.java`

```java
public class ActuatorHandlerMapping implements HandlerMapping {

    private static final String MATCHED_OPERATION_ATTR =
            ActuatorHandlerMapping.class.getName() + ".matchedOperation";

    private final String basePath;
    private final Map<String, DiscoveredEndpoint> endpoints;
    private final ObjectMapper objectMapper;
    private final HandlerMethod dispatchHandlerMethod;

    public ActuatorHandlerMapping(ApplicationContext context, ObjectMapper objectMapper) {
        this.basePath = "/actuator";
        this.objectMapper = objectMapper;
        this.endpoints = new EndpointDiscoverer().discoverEndpoints(context);

        // Create the self-dispatching handler method
        Method handler = getClass().getMethod("handleActuatorRequest",
                HttpServletRequest.class, HttpServletResponse.class);
        this.dispatchHandlerMethod = new HandlerMethod(this, handler);
    }
}
```

**The self-dispatching pattern explained:**

When the `DispatcherServlet` calls `getHandler(request)`:

1. The mapping checks if the path starts with `/actuator`
2. It parses the remaining path to find the endpoint ID and optional selector
3. It looks up the `DiscoveredEndpoint` and finds the matching operation
4. It stores the matched operation as a **request attribute**
5. It returns a `HandlerMethod` pointing to **its own `handleActuatorRequest` method**

```java
@Override
public HandlerMethod getHandler(HttpServletRequest request) {
    String path = getRequestPath(request);
    if (!path.startsWith(basePath)) {
        return null;
    }

    String remaining = path.substring(basePath.length());
    if (remaining.startsWith("/")) remaining = remaining.substring(1);

    MatchedOperation matched = null;

    if (remaining.isEmpty()) {
        // /actuator -- links endpoint
        if ("GET".equals(request.getMethod())) {
            matched = new MatchedOperation(null, null, null);
        }
    } else {
        String[] parts = remaining.split("/", 2);
        String endpointId = parts[0];
        String selectorValue = parts.length > 1 ? parts[1] : null;

        DiscoveredEndpoint endpoint = endpoints.get(endpointId);
        if (endpoint != null) {
            boolean hasSelector = selectorValue != null && !selectorValue.isEmpty();
            EndpointOperation operation = endpoint.findOperation(
                    request.getMethod(), hasSelector);
            if (operation != null) {
                matched = new MatchedOperation(endpoint, operation, selectorValue);
            }
        }
    }

    if (matched != null) {
        request.setAttribute(MATCHED_OPERATION_ATTR, matched);
        return dispatchHandlerMethod;
    }
    return null;
}
```

When the `HandlerAdapter` invokes the handler method, `handleActuatorRequest` retrieves the stored operation from the request attribute and invokes it via reflection:

```java
public void handleActuatorRequest(HttpServletRequest request,
                                   HttpServletResponse response) throws Exception {
    MatchedOperation matched =
            (MatchedOperation) request.getAttribute(MATCHED_OPERATION_ATTR);
    request.removeAttribute(MATCHED_OPERATION_ATTR);

    if (matched.endpoint == null) {
        writeLinksResponse(response);  // /actuator root -- list endpoints
        return;
    }

    Object result = invokeOperation(matched);

    if (result == null) {
        if ("GET".equals(request.getMethod())) {
            response.sendError(HttpServletResponse.SC_NOT_FOUND);
        } else {
            response.setStatus(HttpServletResponse.SC_NO_CONTENT);
        }
        return;
    }

    response.setContentType("application/json;charset=UTF-8");
    objectMapper.writeValue(response.getWriter(), result);
}
```

> **Insight -- Why self-dispatching?** The `ActuatorHandlerMapping` acts as both matcher and handler. This avoids modifying the `HandlerAdapter` -- the adapter sees a normal `HandlerMethod` with `HttpServletRequest`/`HttpServletResponse` parameters, which it already knows how to resolve. The actuator internals (endpoint discovery, operation dispatch, JSON serialization) are completely hidden from the MVC pipeline.

The URL structure:

```
GET  /actuator                     -> links (list of available endpoints)
GET  /actuator/{endpointId}        -> endpoint @ReadOperation (no selector)
GET  /actuator/{endpointId}/{sel}  -> endpoint @ReadOperation (with @Selector)
POST /actuator/{endpointId}        -> endpoint @WriteOperation
```

---

## 24.6 Auto-Configuration

**New file:** `iris-boot-actuator/src/main/java/com/iris/boot/actuate/autoconfigure/ActuatorAutoConfiguration.java`

```java
@AutoConfiguration
@ConditionalOnClass(name = "com.iris.boot.actuate.endpoint.annotation.Endpoint")
public class ActuatorAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public StatusAggregator statusAggregator() {
        return new StatusAggregator();
    }

    @Bean
    @ConditionalOnMissingBean
    public MeterRegistry meterRegistry() {
        return new SimpleMeterRegistry();
    }

    @Bean
    @ConditionalOnMissingBean
    public HealthEndpoint healthEndpoint(ApplicationContext context,
                                          StatusAggregator statusAggregator) {
        DefaultBeanFactory factory =
                ((AnnotationConfigApplicationContext) context).getBeanFactory();
        Map<String, HealthIndicator> rawIndicators =
                factory.getBeansOfType(HealthIndicator.class);

        // Derive friendly component names from bean names
        Map<String, HealthIndicator> indicators = new LinkedHashMap<>();
        for (Map.Entry<String, HealthIndicator> entry : rawIndicators.entrySet()) {
            String name = deriveComponentName(entry.getKey());
            indicators.put(name, entry.getValue());
        }

        return new HealthEndpoint(indicators, statusAggregator);
    }

    @Bean
    @ConditionalOnMissingBean
    public InfoEndpoint infoEndpoint(ApplicationContext context) {
        DefaultBeanFactory factory =
                ((AnnotationConfigApplicationContext) context).getBeanFactory();
        Map<String, InfoContributor> contributors =
                factory.getBeansOfType(InfoContributor.class);
        return new InfoEndpoint(new ArrayList<>(contributors.values()));
    }

    @Bean
    @ConditionalOnMissingBean
    public MetricsEndpoint metricsEndpoint(MeterRegistry registry) {
        return new MetricsEndpoint(registry);
    }

    @Bean
    public ActuatorHandlerMapping actuatorHandlerMapping(ApplicationContext context,
                                                          ObjectMapper objectMapper) {
        return new ActuatorHandlerMapping(context, objectMapper);
    }

    static String deriveComponentName(String beanName) {
        if (beanName.endsWith("HealthIndicator")) {
            return beanName.substring(0, beanName.length() - "HealthIndicator".length());
        }
        if (beanName.endsWith("Indicator")) {
            return beanName.substring(0, beanName.length() - "Indicator".length());
        }
        return beanName;
    }
}
```

**Registration via imports file:**

**New file:** `iris-boot-actuator/src/main/resources/META-INF/iris/com.iris.boot.autoconfigure.AutoConfiguration.imports`

```
# Iris Boot Actuator Auto-Configuration Candidates
com.iris.boot.actuate.autoconfigure.ActuatorAutoConfiguration
```

Key design points:

1. **`@ConditionalOnClass`** gates the entire auto-configuration on the `@Endpoint` annotation being on the classpath -- if the actuator module is not a dependency, nothing activates.
2. **`@ConditionalOnMissingBean`** on each endpoint bean lets users provide their own implementation that takes precedence.
3. **`HealthEndpoint` creation** uses `ApplicationContext` to discover all `HealthIndicator` beans by type, then strips common suffixes from bean names (`diskSpaceHealthIndicator` becomes `diskSpace`).
4. **`ActuatorHandlerMapping`** is **not** `@ConditionalOnMissingBean` -- it is always created because it needs to discover all `@Endpoint` beans in the context.

> **Insight -- The `ApplicationContext` as a bean.** The `healthEndpoint` and `infoEndpoint` `@Bean` methods take `ApplicationContext` as a parameter. This works because `prepareBeanFactory()` registered the context itself as a singleton. Without that registration, the bean factory would fail to resolve the `ApplicationContext` dependency. This is the same mechanism the real Spring Framework uses -- `AbstractApplicationContext.prepareBeanFactory()` registers the context, environment, and system properties as resolvable dependencies.

---

## 24.7 Try It Yourself

Start the application, then:

```bash
# Health check -- aggregate status with component details
curl http://localhost:8080/actuator/health
# { "status": "UP", "components": { "diskSpace": { "status": "UP", ... } } }

# Info endpoint
curl http://localhost:8080/actuator/info
# { "app": { "name": "my-service", "version": "1.0.0" } }

# List all metrics
curl http://localhost:8080/actuator/metrics
# { "names": ["http.server.requests", "jvm.memory.used"] }

# Query a specific metric
curl http://localhost:8080/actuator/metrics/http.server.requests
# { "name": "http.server.requests", "measurements": [...], "availableTags": [...] }

# Actuator root -- HATEOAS-style links
curl http://localhost:8080/actuator
# { "_links": { "self": { "href": "/actuator" }, "health": { "href": "/actuator/health" }, ... } }
```

To add a custom health indicator, define a `HealthIndicator` bean in your configuration:

```java
@Bean
public HealthIndicator dbHealthIndicator() {
    return () -> Health.up()
            .withDetail("database", "PostgreSQL")
            .withDetail("responseTime", "12ms")
            .build();
}
```

The auto-configuration will discover it and include it in the health endpoint response as a component named `db` (the `HealthIndicator` suffix is stripped).

---

## 24.8 Tests

### HealthTest -- 10 tests for Health, Status, and Builder

```java
class HealthTest {

    @Test
    void shouldCreateHealthWithUpStatus_WhenUsingUpFactory() {
        Health health = Health.up().build();
        assertThat(health.getStatus()).isEqualTo(Status.UP);
        assertThat(health.getDetails()).isEmpty();
    }

    @Test
    void shouldIncludeDetails_WhenAddedViaBuilder() {
        Health health = Health.up()
                .withDetail("database", "PostgreSQL")
                .withDetail("responseTime", "12ms")
                .build();
        assertThat(health.getDetails()).containsEntry("database", "PostgreSQL");
        assertThat(health.getDetails()).containsEntry("responseTime", "12ms");
    }

    @Test
    void shouldIncludeExceptionDetails_WhenUsingWithException() {
        RuntimeException ex = new RuntimeException("Connection refused");
        Health health = Health.down(ex).build();
        assertThat(health.getStatus()).isEqualTo(Status.DOWN);
        assertThat((String) health.getDetails().get("error"))
                .contains("RuntimeException").contains("Connection refused");
    }

    @Test
    void shouldHaveImmutableDetails() {
        Health health = Health.up().withDetail("key", "value").build();
        assertThatThrownBy(() -> health.getDetails().put("new", "entry"))
                .isInstanceOf(UnsupportedOperationException.class);
    }
    // ... plus tests for custom status, unknown, outOfService, Status equality, toString
}
```

### StatusAggregatorTest -- 6 tests for severity ordering

```java
class StatusAggregatorTest {

    private final StatusAggregator aggregator = new StatusAggregator();

    @Test
    void shouldReturnDown_WhenAnyIndicatorIsDown() {
        Status result = aggregator.getAggregateStatus(Set.of(Status.UP, Status.DOWN));
        assertThat(result).isEqualTo(Status.DOWN);
    }

    @Test
    void shouldReturnUp_WhenAllIndicatorsAreUp() {
        Status result = aggregator.getAggregateStatus(Set.of(Status.UP));
        assertThat(result).isEqualTo(Status.UP);
    }

    @Test
    void shouldReturnUnknown_WhenNoStatuses() {
        Status result = aggregator.getAggregateStatus(Set.of());
        assertThat(result).isEqualTo(Status.UNKNOWN);
    }
    // ... plus tests for OUT_OF_SERVICE, DOWN+OUT_OF_SERVICE, UNKNOWN+UP
}
```

### EndpointDiscovererTest -- 5 tests for endpoint discovery

```java
class EndpointDiscovererTest {

    @Test
    void shouldDiscoverEndpointBeans_WhenAnnotatedWithEndpoint() {
        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext();
        context.getBeanFactory().registerSingleton("testEndpoint", new TestEndpoint());
        context.register(EmptyConfig.class);
        context.refresh();

        EndpointDiscoverer discoverer = new EndpointDiscoverer();
        Map<String, EndpointDiscoverer.DiscoveredEndpoint> endpoints =
                discoverer.discoverEndpoints(context);

        assertThat(endpoints).containsKey("test");
        context.close();
    }

    @Test
    void shouldDetectSelectorParameters_WhenAnnotated() {
        // Registers a SelectorEndpoint with @ReadOperation + @Selector
        // Verifies findOperation("GET", true) returns a non-null operation
    }

    @Test
    void shouldReturnEmptyMap_WhenNoEndpoints() {
        // Context with no @Endpoint beans -> empty map
    }

    @Endpoint(id = "test")
    static class TestEndpoint {
        @ReadOperation
        public Map<String, String> read() { return Map.of("key", "value"); }

        @WriteOperation
        public void write() { }
    }

    @Endpoint(id = "selector")
    static class SelectorEndpoint {
        @ReadOperation
        public String readWithSelector(@Selector String name) { return "value-" + name; }
    }
}
```

### ActuatorIntegrationTest -- 8 tests for full integration

```java
class ActuatorIntegrationTest {

    @Test
    void shouldDiscoverHealthEndpoint_WhenConfigured() {
        // Registers HealthIndicator, StatusAggregator, HealthEndpoint
        // Verifies health() returns status "UP" with components
    }

    @Test
    void shouldAggregateDownStatus_WhenOneIndicatorIsDown() {
        // One UP + one DOWN indicator -> aggregate is "DOWN"
    }

    @Test
    void shouldHandleExceptionInHealthIndicator_WhenIndicatorFails() {
        // HealthIndicator that throws -> aggregate is "DOWN" (not crash)
    }

    @Test
    void shouldCollectInfoFromContributors_WhenConfigured() {
        // InfoContributor adds "app" section -> info() contains it
    }

    @Test
    void shouldExposeMetrics_WhenRegistryHasMeters() {
        // Counter with 5 increments -> metric() returns value 5.0
    }

    @Test
    void shouldReturnNull_WhenMetricNotFound() {
        // Querying "nonexistent" -> null (which maps to 404)
    }

    @Test
    void shouldCreateActuatorHandlerMapping_WhenEndpointsExist() {
        // Full config with health/info/metrics -> mapping has >= 3 endpoints
    }

    @Test
    void shouldHandleSelectorEndpoints_WhenMetricQueried() {
        // Counter with tags -> availableTags in response
    }
}
```

---

## 24.9 Why This Works

### The DispatcherServlet List Pattern

```
Request: GET /actuator/health

DispatcherServlet.doDispatch()
  |
  +-- handlerMappings.get(0): RequestMappingHandlerMapping
  |     getHandler() -> null  (no @Controller maps to /actuator/health)
  |
  +-- handlerMappings.get(1): ActuatorHandlerMapping
        getHandler() -> HandlerMethod(this, "handleActuatorRequest")
        |
        +-- Stores MatchedOperation as request attribute
        |
        +-- HandlerAdapter invokes handleActuatorRequest()
              |
              +-- Retrieves MatchedOperation from attribute
              +-- Invokes HealthEndpoint.health() via reflection
              +-- Writes JSON response
```

The key insight: the DispatcherServlet does not know anything about actuator endpoints. It simply iterates its list of HandlerMappings and takes the first match. The ActuatorHandlerMapping handles everything internally.

> **Insight -- Request attributes as a communication channel.** The `MATCHED_OPERATION_ATTR` pattern is how the mapping phase communicates with the handling phase without violating the HandlerMapping/HandlerAdapter contract. This is a standard Servlet API idiom -- `request.setAttribute()` passes contextual data between components in the same request without coupling their interfaces.

### The Self-Dispatching Pattern

```
ActuatorHandlerMapping
  |
  +-- implements HandlerMapping (matching)
  |     getHandler() -> returns HandlerMethod pointing to itself
  |
  +-- implements the handler (dispatch)
        handleActuatorRequest() -> invokes endpoint operation, writes JSON
```

> **Insight -- Avoiding adapter modification.** If the ActuatorHandlerMapping returned an endpoint bean directly, the HandlerAdapter would need to understand `@ReadOperation` and `@Selector`. By wrapping everything in a `handleActuatorRequest(HttpServletRequest, HttpServletResponse)` method, the adapter treats it as a normal handler method. The actuator-specific logic is entirely encapsulated.

### The Endpoint Discovery Flow

```
ActuatorAutoConfiguration
  |
  +-- Creates HealthEndpoint, InfoEndpoint, MetricsEndpoint beans
  |
  +-- Creates ActuatorHandlerMapping bean
        |
        +-- EndpointDiscoverer.discoverEndpoints(context)
              |
              +-- Iterates all beans in BeanFactory
              +-- Finds beans with @Endpoint annotation
              +-- Introspects @ReadOperation/@WriteOperation methods
              +-- Records @Selector parameters
              +-- Returns Map<endpointId, DiscoveredEndpoint>
```

> **Insight -- Discovery at construction time.** The `EndpointDiscoverer` runs during `ActuatorHandlerMapping` construction, which happens during `finishBeanFactoryInitialization()`. By this time, all endpoint beans have been created. The discovered operations are cached in the mapping, so no reflection happens during request processing -- only during startup.

---

## 24.10 What We Enhanced

| Component | Before | After |
|-----------|--------|-------|
| `DispatcherServlet` | Single `HandlerMapping` field | `List<HandlerMapping>` with bean discovery |
| `AnnotationConfigApplicationContext` | No `prepareBeanFactory()` | Registers context as singleton bean |
| Handler discovery | Only `@Controller` beans | Also discovers `HandlerMapping` beans from context |
| Dispatch loop | Direct `handlerMapping.getHandler()` | Iterates list, first match wins |
| Actuator support | None | Full endpoint subsystem with health/info/metrics |

**Files created:** 16 production files + 4 test files in the `iris-boot-actuator` module
**Files modified:** 2 (`DispatcherServlet.java`, `AnnotationConfigApplicationContext.java`)

---

## 24.11 Connection to Real Framework

| Iris Class | Spring Boot Class | Source Location |
|-----------|------------------|-----------------|
| `@Endpoint` | `@Endpoint` | `Endpoint.java` |
| `@ReadOperation` | `@ReadOperation` | `ReadOperation.java` |
| `@WriteOperation` | `@WriteOperation` | `WriteOperation.java` |
| `@Selector` | `@Selector` | `Selector.java` |
| `EndpointDiscoverer` | `EndpointDiscoverer` | `EndpointDiscoverer.java` |
| `ActuatorHandlerMapping` | `WebMvcEndpointHandlerMapping` | `AbstractWebMvcEndpointHandlerMapping.java` |
| `HealthIndicator` | `HealthIndicator` | `HealthIndicator.java` |
| `Health` | `Health` | `Health.java` |
| `Status` | `Status` | `Status.java` |
| `StatusAggregator` | `SimpleStatusAggregator` | `SimpleStatusAggregator.java` |
| `HealthEndpoint` | `HealthEndpoint` | `HealthEndpoint.java` |
| `InfoContributor` | `InfoContributor` | `InfoContributor.java` |
| `InfoEndpoint` | `InfoEndpoint` | `InfoEndpoint.java` |
| `MetricsEndpoint` | `MetricsEndpoint` | `MetricsEndpoint.java` |

### Key Simplifications

| What we simplified | What Spring Boot does | Why it matters |
|-------------------|----------------------|----------------|
| Single `EndpointDiscoverer` class | Abstract `EndpointDiscoverer` with `WebEndpointDiscoverer` and `JmxEndpointDiscoverer` subclasses | The real framework exposes endpoints over both HTTP and JMX with technology-specific adapters |
| Self-dispatching `ActuatorHandlerMapping` | `WebMvcEndpointHandlerMapping` extends `RequestMappingInfoHandlerMapping`, registering each operation as its own `RequestMappingInfo` | The real framework integrates fully with Spring MVC's handler mapping infrastructure |
| Flat list of `HealthIndicator` beans | Sealed `HealthContributor` hierarchy with `CompositeHealthContributor` for tree structures | Supports hierarchical health groups (e.g., liveness/readiness probes for Kubernetes) |
| Fixed severity ordering in `StatusAggregator` | Configurable via `management.endpoint.health.status.order` property | Allows applications to define custom severity rankings |
| No endpoint exposure filtering | `include`/`exclude` configuration for endpoint visibility | Production systems expose only specific endpoints over HTTP vs. JMX |
| No health groups | `management.endpoint.health.group.*` for liveness and readiness | Kubernetes readiness/liveness probes query different subsets of health indicators |

---

## 24.12 Complete Code

### Production Code

#### `Endpoint.java` [NEW]

```java
package com.iris.boot.actuate.endpoint.annotation;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Marks a class as an actuator endpoint — a management bean whose operations
 * are exposed over HTTP under {@code /actuator/{id}}.
 *
 * <p>The {@link #id()} attribute provides the endpoint's identity, which
 * determines both its bean registration name and its HTTP path segment.
 * For example, {@code @Endpoint(id = "health")} maps to
 * {@code GET /actuator/health}.
 *
 * <p>Methods on the endpoint class are annotated with {@link ReadOperation}
 * (HTTP GET) or {@link WriteOperation} (HTTP POST) to define the operations.
 *
 * <p>In the real Spring Boot, {@code @Endpoint} also has:
 * <ul>
 *   <li>{@code defaultAccess} — controls read-only vs. unrestricted access</li>
 *   <li>{@code enableByDefault} — whether the endpoint is enabled by default</li>
 *   <li>Support for technology-specific extensions ({@code @WebEndpoint},
 *       {@code @JmxEndpoint})</li>
 * </ul>
 *
 * @see ReadOperation
 * @see WriteOperation
 * @see org.springframework.boot.actuate.endpoint.annotation.Endpoint
 */
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface Endpoint {

    /**
     * The unique identifier for this endpoint. Must be a simple, lowercase
     * string (e.g., "health", "info", "metrics").
     *
     * <p>This ID is used as the URL path segment under {@code /actuator/}.
     *
     * @return the endpoint id
     */
    String id();
}
```

#### `ReadOperation.java` [NEW]

```java
package com.iris.boot.actuate.endpoint.annotation;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Marks a method on an {@link Endpoint} class as a read operation,
 * mapped to HTTP GET.
 *
 * <p>Read operations are idempotent and safe — they return data without
 * modifying state. The return value is serialized to JSON automatically.
 *
 * <p>If the method returns {@code null}, the HTTP response is 404 (Not Found).
 *
 * <p>In the real Spring Boot, {@code @ReadOperation} also supports:
 * <ul>
 *   <li>{@code produces} — custom media types</li>
 *   <li>{@code producesFrom} — dynamic media type negotiation via {@code Producible}</li>
 * </ul>
 *
 * @see Endpoint
 * @see WriteOperation
 * @see org.springframework.boot.actuate.endpoint.annotation.ReadOperation
 */
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface ReadOperation {
}
```

#### `WriteOperation.java` [NEW]

```java
package com.iris.boot.actuate.endpoint.annotation;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Marks a method on an {@link Endpoint} class as a write operation,
 * mapped to HTTP POST.
 *
 * <p>Write operations modify state. If the method returns {@code null},
 * the HTTP response is 204 (No Content). Otherwise, the return value
 * is serialized to JSON.
 *
 * <p>In the real Spring Boot, {@code @WriteOperation} also supports
 * custom media types via {@code produces}/{@code producesFrom}.
 *
 * @see Endpoint
 * @see ReadOperation
 * @see org.springframework.boot.actuate.endpoint.annotation.WriteOperation
 */
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface WriteOperation {
}
```

#### `Selector.java` [NEW]

```java
package com.iris.boot.actuate.endpoint.annotation;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * Marks a parameter on an {@link Endpoint} operation method as a path
 * selector — a value extracted from the URL path that narrows the
 * operation's scope.
 *
 * <p>For example, in the metrics endpoint:
 * <pre>{@code
 * @ReadOperation
 * public MetricDescriptor metric(@Selector String metricName) { ... }
 * }</pre>
 *
 * <p>This maps to {@code GET /actuator/metrics/{metricName}}, where
 * {@code metricName} is extracted from the URL path.
 *
 * <p>In the real Spring Boot, {@code @Selector} also supports:
 * <ul>
 *   <li>{@code Match.ALL_REMAINING} — captures multiple path segments
 *       as a {@code String[]} (used by the health endpoint for
 *       component-level health)</li>
 * </ul>
 *
 * @see Endpoint
 * @see ReadOperation
 * @see org.springframework.boot.actuate.endpoint.annotation.Selector
 */
@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
public @interface Selector {
}
```

#### `Status.java` [NEW]

```java
package com.iris.boot.actuate.health;

import java.util.Objects;

/**
 * Value object representing the status of a component's health.
 *
 * <p>Four well-known statuses are predefined as constants:
 * <ul>
 *   <li>{@link #UP} — component is functioning normally</li>
 *   <li>{@link #DOWN} — component has failed</li>
 *   <li>{@link #OUT_OF_SERVICE} — component is intentionally taken offline</li>
 *   <li>{@link #UNKNOWN} — component status cannot be determined</li>
 * </ul>
 *
 * <p>Custom statuses can be created via {@code new Status("DEGRADED")} for
 * application-specific health states.
 *
 * <p>In the real Spring Boot, {@code Status} also carries an optional
 * {@code description} field explaining the status. We omit it for simplicity.
 *
 * @see Health
 * @see HealthIndicator
 * @see org.springframework.boot.health.contributor.Status
 */
public final class Status {

    /** Health status indicating the component is functioning normally. */
    public static final Status UP = new Status("UP");

    /** Health status indicating the component has failed. */
    public static final Status DOWN = new Status("DOWN");

    /** Health status indicating the component is intentionally offline. */
    public static final Status OUT_OF_SERVICE = new Status("OUT_OF_SERVICE");

    /** Health status indicating the component's status cannot be determined. */
    public static final Status UNKNOWN = new Status("UNKNOWN");

    private final String code;

    /**
     * Create a new {@code Status} with the given code.
     *
     * @param code the status code (e.g., "UP", "DOWN", "DEGRADED")
     */
    public Status(String code) {
        Objects.requireNonNull(code, "Status code must not be null");
        this.code = code;
    }

    /**
     * Return the status code.
     */
    public String getCode() {
        return this.code;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Status other)) return false;
        return this.code.equals(other.code);
    }

    @Override
    public int hashCode() {
        return this.code.hashCode();
    }

    @Override
    public String toString() {
        return this.code;
    }
}
```

#### `Health.java` [NEW]

```java
package com.iris.boot.actuate.health;

import java.util.Collections;
import java.util.LinkedHashMap;
import java.util.Map;

/**
 * Carries information about the health of a component or the overall system.
 *
 * <p>A {@code Health} consists of a {@link Status} and an optional map of
 * details providing additional context (e.g., database response time,
 * disk space remaining, error messages).
 *
 * <p>Built using the {@link Builder} pattern for a fluent API:
 * <pre>{@code
 * Health health = Health.up()
 *     .withDetail("database", "PostgreSQL")
 *     .withDetail("responseTime", "12ms")
 *     .build();
 * }</pre>
 *
 * <p>In the real Spring Boot, {@code Health} also supports:
 * <ul>
 *   <li>{@code @JsonUnwrapped} on status — status fields inline into the JSON</li>
 *   <li>{@code @JsonInclude(NON_EMPTY)} on details — omit empty details</li>
 *   <li>{@code withoutDetails()} — strip details for security/privacy</li>
 * </ul>
 *
 * @see Status
 * @see HealthIndicator
 * @see org.springframework.boot.health.contributor.Health
 */
public final class Health {

    private final Status status;
    private final Map<String, Object> details;

    private Health(Status status, Map<String, Object> details) {
        this.status = status;
        this.details = Collections.unmodifiableMap(details);
    }

    /**
     * Return the status of this health.
     */
    public Status getStatus() {
        return this.status;
    }

    /**
     * Return the health details (unmodifiable).
     */
    public Map<String, Object> getDetails() {
        return this.details;
    }

    @Override
    public String toString() {
        return "Health{status=" + status + ", details=" + details + "}";
    }

    // --- Static factory methods for common statuses ---

    /** Create a {@link Builder} with {@link Status#UP}. */
    public static Builder up() {
        return new Builder(Status.UP);
    }

    /** Create a {@link Builder} with {@link Status#DOWN}. */
    public static Builder down() {
        return new Builder(Status.DOWN);
    }

    /** Create a {@link Builder} with {@link Status#DOWN} and exception details. */
    public static Builder down(Throwable ex) {
        return down().withException(ex);
    }

    /** Create a {@link Builder} with {@link Status#UNKNOWN}. */
    public static Builder unknown() {
        return new Builder(Status.UNKNOWN);
    }

    /** Create a {@link Builder} with {@link Status#OUT_OF_SERVICE}. */
    public static Builder outOfService() {
        return new Builder(Status.OUT_OF_SERVICE);
    }

    /** Create a {@link Builder} with the given {@link Status}. */
    public static Builder status(Status status) {
        return new Builder(status);
    }

    /**
     * Fluent builder for constructing {@link Health} instances.
     *
     * <p>In the real Spring Boot, the builder is an inner class of {@code Health}
     * and also supports {@code withDetails(Map)} for bulk detail addition.
     */
    public static class Builder {

        private final Status status;
        private final Map<String, Object> details = new LinkedHashMap<>();

        private Builder(Status status) {
            this.status = status;
        }

        /**
         * Add a detail entry.
         *
         * @param key   the detail key
         * @param value the detail value
         * @return this builder
         */
        public Builder withDetail(String key, Object value) {
            this.details.put(key, value);
            return this;
        }

        /**
         * Add exception details (class name and message).
         *
         * @param ex the exception
         * @return this builder
         */
        public Builder withException(Throwable ex) {
            this.details.put("error", ex.getClass().getName() + ": " + ex.getMessage());
            return this;
        }

        /**
         * Build the immutable {@link Health} instance.
         */
        public Health build() {
            return new Health(this.status, this.details);
        }
    }
}
```

#### `HealthIndicator.java` [NEW]

```java
package com.iris.boot.actuate.health;

/**
 * Strategy interface for providing the health of a component.
 *
 * <p>Implementations report the health of a specific subsystem — a database,
 * a message broker, a disk, an external service. The actuator's
 * {@link HealthEndpoint} aggregates all indicators into a composite health
 * response.
 *
 * <p>Implemented as a {@code @FunctionalInterface} so that simple indicators
 * can be expressed as lambdas:
 * <pre>{@code
 * @Bean
 * public HealthIndicator diskSpaceHealth() {
 *     return () -> Health.up()
 *         .withDetail("free", getFreeSpace())
 *         .build();
 * }
 * }</pre>
 *
 * <p>In the real Spring Boot, the health contributor hierarchy is:
 * <pre>
 * HealthContributor (sealed)
 *   ├── HealthIndicator (non-sealed, @FunctionalInterface)
 *   └── CompositeHealthContributor (non-sealed, tree structure)
 * </pre>
 * We simplify to just {@code HealthIndicator} — no composite tree.
 *
 * @see Health
 * @see HealthEndpoint
 * @see org.springframework.boot.health.contributor.HealthIndicator
 */
@FunctionalInterface
public interface HealthIndicator {

    /**
     * Return an indication of health.
     *
     * @return the health (never {@code null})
     */
    Health health();
}
```

#### `StatusAggregator.java` [NEW]

```java
package com.iris.boot.actuate.health;

import java.util.List;
import java.util.Set;

/**
 * Aggregates multiple {@link Status} values into a single overall status.
 *
 * <p>The aggregation follows a severity ordering: the "worst" status wins.
 * The default ordering (from worst to best) is:
 * <ol>
 *   <li>{@code DOWN}</li>
 *   <li>{@code OUT_OF_SERVICE}</li>
 *   <li>{@code UNKNOWN}</li>
 *   <li>{@code UP}</li>
 * </ol>
 *
 * <p>For example, if one indicator reports {@code UP} and another reports
 * {@code DOWN}, the aggregate status is {@code DOWN}.
 *
 * <p>In the real Spring Boot, this is {@code SimpleStatusAggregator} which
 * implements a {@code StatusAggregator} interface. The ordering is
 * configurable via {@code management.endpoint.health.status.order}.
 * We use a fixed ordering for simplicity.
 *
 * @see Status
 * @see HealthEndpoint
 * @see org.springframework.boot.health.actuate.endpoint.SimpleStatusAggregator
 */
public class StatusAggregator {

    /**
     * Default status ordering — worst (highest severity) first.
     */
    private static final List<String> DEFAULT_ORDER = List.of(
            "DOWN", "OUT_OF_SERVICE", "UNKNOWN", "UP"
    );

    /**
     * Aggregate the given statuses into a single status.
     *
     * <p>Returns the status with the highest severity (lowest index in the
     * ordering). If no statuses match the ordering, returns {@link Status#UNKNOWN}.
     *
     * @param statuses the statuses to aggregate
     * @return the aggregate status
     */
    public Status getAggregateStatus(Set<Status> statuses) {
        if (statuses.isEmpty()) {
            return Status.UNKNOWN;
        }

        // Find the status with the lowest index (worst severity) in the ordering
        int worstIndex = Integer.MAX_VALUE;
        Status worstStatus = null;

        for (Status status : statuses) {
            int index = DEFAULT_ORDER.indexOf(status.getCode());
            if (index == -1) {
                // Unknown status code — treat as between OUT_OF_SERVICE and UNKNOWN
                index = 2;
            }
            if (index < worstIndex) {
                worstIndex = index;
                worstStatus = status;
            }
        }

        return worstStatus != null ? worstStatus : Status.UNKNOWN;
    }
}
```

#### `HealthEndpoint.java` [NEW]

```java
package com.iris.boot.actuate.health;

import java.util.LinkedHashMap;
import java.util.Map;
import java.util.Set;
import java.util.stream.Collectors;

import com.iris.boot.actuate.endpoint.annotation.Endpoint;
import com.iris.boot.actuate.endpoint.annotation.ReadOperation;

/**
 * Actuator endpoint that exposes the health of the application by aggregating
 * all registered {@link HealthIndicator} beans.
 *
 * <p>The health endpoint provides:
 * <ul>
 *   <li>{@code GET /actuator/health} — aggregate health with component details</li>
 * </ul>
 *
 * <p>The response includes an overall {@link Status} (aggregated from all
 * indicators) and a {@code components} map showing each indicator's individual
 * health. Example response:
 * <pre>{@code
 * {
 *   "status": "UP",
 *   "components": {
 *     "diskSpace": { "status": "UP", "details": { "free": "120GB" } },
 *     "db": { "status": "UP", "details": { "database": "PostgreSQL" } }
 *   }
 * }
 * }</pre>
 *
 * <p>In the real Spring Boot, {@code HealthEndpoint} extends
 * {@code HealthEndpointSupport} which handles:
 * <ul>
 *   <li>Recursive traversal of composite health contributor trees</li>
 *   <li>Health groups (e.g., "liveness", "readiness" for Kubernetes)</li>
 *   <li>Path-based component drilldown ({@code /actuator/health/db})</li>
 *   <li>Slow indicator logging</li>
 *   <li>Access control (showing/hiding details based on security)</li>
 * </ul>
 *
 * @see HealthIndicator
 * @see StatusAggregator
 * @see org.springframework.boot.health.actuate.endpoint.HealthEndpoint
 */
@Endpoint(id = "health")
public class HealthEndpoint {

    private final Map<String, HealthIndicator> healthIndicators;
    private final StatusAggregator statusAggregator;

    /**
     * Create a new HealthEndpoint.
     *
     * @param healthIndicators map of component name → health indicator
     * @param statusAggregator the aggregator for computing overall status
     */
    public HealthEndpoint(Map<String, HealthIndicator> healthIndicators,
                          StatusAggregator statusAggregator) {
        this.healthIndicators = healthIndicators;
        this.statusAggregator = statusAggregator;
    }

    /**
     * Return the overall health of the application.
     *
     * <p>Calls each registered {@link HealthIndicator}, collects their
     * individual statuses, and aggregates them into an overall status
     * via the {@link StatusAggregator}.
     *
     * @return a map containing "status" and "components" keys
     */
    @ReadOperation
    public Map<String, Object> health() {
        Map<String, Object> result = new LinkedHashMap<>();

        if (healthIndicators.isEmpty()) {
            result.put("status", Status.UP.getCode());
            return result;
        }

        // Collect health from each indicator
        Map<String, Map<String, Object>> components = new LinkedHashMap<>();
        for (Map.Entry<String, HealthIndicator> entry : healthIndicators.entrySet()) {
            String name = entry.getKey();
            HealthIndicator indicator = entry.getValue();
            try {
                Health health = indicator.health();
                components.put(name, healthToMap(health));
            } catch (Exception ex) {
                Health errorHealth = Health.down(ex).build();
                components.put(name, healthToMap(errorHealth));
            }
        }

        // Aggregate statuses
        Set<Status> statuses = components.values().stream()
                .map(m -> new Status((String) m.get("status")))
                .collect(Collectors.toSet());
        Status aggregateStatus = statusAggregator.getAggregateStatus(statuses);

        result.put("status", aggregateStatus.getCode());
        result.put("components", components);
        return result;
    }

    /**
     * Convert a {@link Health} to a serializable map.
     */
    private Map<String, Object> healthToMap(Health health) {
        Map<String, Object> map = new LinkedHashMap<>();
        map.put("status", health.getStatus().getCode());
        if (!health.getDetails().isEmpty()) {
            map.put("details", health.getDetails());
        }
        return map;
    }
}
```

#### `InfoContributor.java` [NEW]

```java
package com.iris.boot.actuate.info;

import java.util.Map;

/**
 * Strategy interface for contributing information to the {@link InfoEndpoint}.
 *
 * <p>Implementations add key-value pairs to the info response. For example,
 * an application info contributor might add the application name and version:
 * <pre>{@code
 * @Bean
 * public InfoContributor appInfoContributor() {
 *     return builder -> builder.put("app", Map.of(
 *         "name", "my-service",
 *         "version", "1.2.3"
 *     ));
 * }
 * }</pre>
 *
 * <p>In the real Spring Boot, {@code InfoContributor} uses an
 * {@code Info.Builder} pattern. We simplify to direct map manipulation.
 *
 * @see InfoEndpoint
 * @see org.springframework.boot.actuate.info.InfoContributor
 */
@FunctionalInterface
public interface InfoContributor {

    /**
     * Contribute information to the given map.
     *
     * @param info the mutable info map to add entries to
     */
    void contribute(Map<String, Object> info);
}
```

#### `InfoEndpoint.java` [NEW]

```java
package com.iris.boot.actuate.info;

import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

import com.iris.boot.actuate.endpoint.annotation.Endpoint;
import com.iris.boot.actuate.endpoint.annotation.ReadOperation;

/**
 * Actuator endpoint that exposes application information collected from
 * all registered {@link InfoContributor} beans.
 *
 * <p>The info endpoint provides:
 * <ul>
 *   <li>{@code GET /actuator/info} — aggregated application information</li>
 * </ul>
 *
 * <p>Each contributor adds its own section to the response. Example:
 * <pre>{@code
 * {
 *   "app": { "name": "my-service", "version": "1.2.3" },
 *   "java": { "version": "17.0.8" }
 * }
 * }</pre>
 *
 * <p>In the real Spring Boot, built-in contributors include:
 * <ul>
 *   <li>{@code EnvironmentInfoContributor} — exposes {@code info.*} properties</li>
 *   <li>{@code GitInfoContributor} — exposes git commit info from {@code git.properties}</li>
 *   <li>{@code BuildInfoContributor} — exposes build info from {@code META-INF/build-info.properties}</li>
 *   <li>{@code JavaInfoContributor} — exposes Java runtime info</li>
 *   <li>{@code OsInfoContributor} — exposes OS info</li>
 * </ul>
 *
 * @see InfoContributor
 * @see org.springframework.boot.actuate.info.InfoEndpoint
 */
@Endpoint(id = "info")
public class InfoEndpoint {

    private final List<InfoContributor> contributors;

    /**
     * Create a new InfoEndpoint.
     *
     * @param contributors the info contributors to aggregate
     */
    public InfoEndpoint(List<InfoContributor> contributors) {
        this.contributors = contributors;
    }

    /**
     * Return aggregated application information.
     *
     * <p>Iterates all registered {@link InfoContributor}s and lets each
     * contribute to a shared map. The result is the merged map.
     *
     * @return the info map
     */
    @ReadOperation
    public Map<String, Object> info() {
        Map<String, Object> info = new LinkedHashMap<>();
        for (InfoContributor contributor : contributors) {
            contributor.contribute(info);
        }
        return info;
    }
}
```

#### `MetricsEndpoint.java` [NEW]

```java
package com.iris.boot.actuate.metrics;

import java.util.ArrayList;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.TreeSet;
import java.util.concurrent.TimeUnit;

import com.iris.boot.actuate.endpoint.annotation.Endpoint;
import com.iris.boot.actuate.endpoint.annotation.ReadOperation;
import com.iris.boot.actuate.endpoint.annotation.Selector;
import com.iris.micrometer.core.Counter;
import com.iris.micrometer.core.Gauge;
import com.iris.micrometer.core.Meter;
import com.iris.micrometer.core.MeterRegistry;
import com.iris.micrometer.core.Tag;
import com.iris.micrometer.core.Timer;

/**
 * Actuator endpoint that exposes metrics from the {@link MeterRegistry}.
 *
 * <p>Two operations:
 * <ul>
 *   <li>{@code GET /actuator/metrics} — list all meter names</li>
 *   <li>{@code GET /actuator/metrics/{metricName}} — detailed view of a
 *       specific meter (measurements + available tags)</li>
 * </ul>
 *
 * <p>Example listing response:
 * <pre>{@code
 * { "names": ["http.server.requests", "jvm.memory.used", "system.cpu.usage"] }
 * }</pre>
 *
 * <p>Example detail response:
 * <pre>{@code
 * {
 *   "name": "http.server.requests",
 *   "measurements": [
 *     { "statistic": "COUNT", "value": 42.0 },
 *     { "statistic": "TOTAL_TIME", "value": 1.234 }
 *   ],
 *   "availableTags": [
 *     { "tag": "method", "values": ["GET", "POST"] },
 *     { "tag": "status", "values": ["200", "404"] }
 *   ]
 * }
 * }</pre>
 *
 * <p>In the real Spring Boot, {@code MetricsEndpoint} also supports:
 * <ul>
 *   <li>Tag-based filtering via query parameters ({@code ?tag=method:GET})</li>
 *   <li>{@code CompositeMeterRegistry} traversal</li>
 *   <li>Base unit and description in the response</li>
 * </ul>
 *
 * @see MeterRegistry
 * @see org.springframework.boot.micrometer.metrics.actuate.endpoint.MetricsEndpoint
 */
@Endpoint(id = "metrics")
public class MetricsEndpoint {

    private final MeterRegistry registry;

    /**
     * Create a new MetricsEndpoint.
     *
     * @param registry the meter registry to expose
     */
    public MetricsEndpoint(MeterRegistry registry) {
        this.registry = registry;
    }

    /**
     * List all registered meter names.
     *
     * @return a map with a "names" key containing a sorted set of meter names
     */
    @ReadOperation
    public Map<String, Object> listNames() {
        TreeSet<String> names = new TreeSet<>();
        for (Meter meter : registry.getMeters()) {
            names.add(meter.getId().getName());
        }
        return Map.of("names", names);
    }

    /**
     * Return detailed information about a specific meter.
     *
     * @param metricName the name of the metric to query
     * @return metric details, or {@code null} if not found
     */
    @ReadOperation
    public Map<String, Object> metric(@Selector String metricName) {
        // Find all meters with this name (may have different tag combinations)
        List<Meter> matchingMeters = new ArrayList<>();
        for (Meter meter : registry.getMeters()) {
            if (meter.getId().getName().equals(metricName)) {
                matchingMeters.add(meter);
            }
        }

        if (matchingMeters.isEmpty()) {
            return null;
        }

        // Collect measurements from the first matching meter
        Meter firstMeter = matchingMeters.get(0);
        List<Map<String, Object>> measurements = getMeasurements(firstMeter);

        // Collect available tag keys and values across all matching meters
        Map<String, TreeSet<String>> tagValues = new LinkedHashMap<>();
        for (Meter meter : matchingMeters) {
            for (Tag tag : meter.getId().getTags()) {
                tagValues.computeIfAbsent(tag.getKey(), k -> new TreeSet<>())
                        .add(tag.getValue());
            }
        }
        List<Map<String, Object>> availableTags = new ArrayList<>();
        for (Map.Entry<String, TreeSet<String>> entry : tagValues.entrySet()) {
            availableTags.add(Map.of(
                    "tag", entry.getKey(),
                    "values", new ArrayList<>(entry.getValue())
            ));
        }

        Map<String, Object> result = new LinkedHashMap<>();
        result.put("name", metricName);
        result.put("measurements", measurements);
        result.put("availableTags", availableTags);

        // Add description and base unit if available
        Meter.Id id = firstMeter.getId();
        if (id.getDescription() != null) {
            result.put("description", id.getDescription());
        }
        if (id.getBaseUnit() != null) {
            result.put("baseUnit", id.getBaseUnit());
        }

        return result;
    }

    /**
     * Extract measurements from a meter based on its type.
     */
    private List<Map<String, Object>> getMeasurements(Meter meter) {
        List<Map<String, Object>> measurements = new ArrayList<>();

        if (meter instanceof Counter counter) {
            measurements.add(Map.of("statistic", "COUNT", "value", counter.count()));
        } else if (meter instanceof Timer timer) {
            measurements.add(Map.of("statistic", "COUNT", "value", (double) timer.count()));
            measurements.add(Map.of("statistic", "TOTAL_TIME",
                    "value", timer.totalTime(TimeUnit.SECONDS)));
            measurements.add(Map.of("statistic", "MAX",
                    "value", timer.max(TimeUnit.SECONDS)));
        } else if (meter instanceof Gauge gauge) {
            measurements.add(Map.of("statistic", "VALUE", "value", gauge.value()));
        }

        return measurements;
    }
}
```

#### `EndpointDiscoverer.java` [NEW]

```java
package com.iris.boot.actuate.endpoint;

import java.lang.annotation.Annotation;
import java.lang.reflect.Method;
import java.lang.reflect.Parameter;
import java.util.ArrayList;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

import com.iris.boot.actuate.endpoint.annotation.Endpoint;
import com.iris.boot.actuate.endpoint.annotation.ReadOperation;
import com.iris.boot.actuate.endpoint.annotation.Selector;
import com.iris.boot.actuate.endpoint.annotation.WriteOperation;
import com.iris.framework.beans.factory.support.DefaultBeanFactory;
import com.iris.framework.context.AnnotationConfigApplicationContext;
import com.iris.framework.context.ApplicationContext;

/**
 * Discovers {@link Endpoint @Endpoint} beans in the {@link ApplicationContext}
 * and introspects their operations ({@link ReadOperation}, {@link WriteOperation}).
 *
 * <p>The discovery process:
 * <ol>
 *   <li>Iterate all bean definitions in the bean factory</li>
 *   <li>For each bean whose class is annotated with {@code @Endpoint},
 *       extract the endpoint ID</li>
 *   <li>Scan the class's methods for {@code @ReadOperation} and
 *       {@code @WriteOperation} annotations</li>
 *   <li>For each operation, determine if it has a {@code @Selector}
 *       parameter (which maps to a path segment)</li>
 *   <li>Build a {@link DiscoveredEndpoint} with all its operations</li>
 * </ol>
 *
 * <p>In the real Spring Boot, endpoint discovery is much more sophisticated:
 * <ul>
 *   <li>{@code EndpointDiscoverer} is an abstract template class with
 *       concrete subclasses ({@code WebEndpointDiscoverer},
 *       {@code JmxEndpointDiscoverer})</li>
 *   <li>Supports endpoint extensions ({@code @EndpointExtension})</li>
 *   <li>Applies {@code EndpointFilter} and {@code OperationFilter} chains</li>
 *   <li>Handles {@code ParameterValueMapper} for type conversion</li>
 *   <li>Caches discovered endpoints</li>
 * </ul>
 *
 * @see Endpoint
 * @see DiscoveredEndpoint
 * @see org.springframework.boot.actuate.endpoint.annotation.EndpointDiscoverer
 */
public class EndpointDiscoverer {

    /**
     * Discover all {@code @Endpoint} beans in the given context.
     *
     * @param context the application context to scan
     * @return map of endpoint ID → discovered endpoint
     */
    public Map<String, DiscoveredEndpoint> discoverEndpoints(ApplicationContext context) {
        Map<String, DiscoveredEndpoint> endpoints = new LinkedHashMap<>();
        DefaultBeanFactory factory =
                ((AnnotationConfigApplicationContext) context).getBeanFactory();

        for (String name : factory.getBeanDefinitionNames()) {
            Object bean;
            try {
                bean = factory.getBean(name);
            } catch (Exception e) {
                continue;
            }

            Endpoint endpoint = bean.getClass().getAnnotation(Endpoint.class);
            if (endpoint != null) {
                DiscoveredEndpoint discovered = introspect(endpoint.id(), bean);
                endpoints.put(endpoint.id(), discovered);
            }
        }

        return endpoints;
    }

    /**
     * Introspect an endpoint bean to discover its operations.
     */
    private DiscoveredEndpoint introspect(String id, Object bean) {
        List<EndpointOperation> operations = new ArrayList<>();

        for (Method method : bean.getClass().getMethods()) {
            if (method.getDeclaringClass() == Object.class) {
                continue;
            }

            ReadOperation readOp = method.getAnnotation(ReadOperation.class);
            if (readOp != null) {
                boolean hasSelector = hasSelector(method);
                operations.add(new EndpointOperation(bean, method, "GET", hasSelector));
            }

            WriteOperation writeOp = method.getAnnotation(WriteOperation.class);
            if (writeOp != null) {
                boolean hasSelector = hasSelector(method);
                operations.add(new EndpointOperation(bean, method, "POST", hasSelector));
            }
        }

        return new DiscoveredEndpoint(id, bean, operations);
    }

    /**
     * Check if a method has a parameter annotated with {@link Selector}.
     */
    private boolean hasSelector(Method method) {
        for (Parameter param : method.getParameters()) {
            if (param.isAnnotationPresent(Selector.class)) {
                return true;
            }
        }
        return false;
    }

    // -----------------------------------------------------------------------
    // Inner classes for discovered endpoints and operations
    // -----------------------------------------------------------------------

    /**
     * A discovered endpoint with its operations.
     */
    public record DiscoveredEndpoint(
            String id,
            Object bean,
            List<EndpointOperation> operations
    ) {
        /**
         * Find the operation matching the given HTTP method and whether
         * a selector is present.
         */
        public EndpointOperation findOperation(String httpMethod, boolean hasSelector) {
            for (EndpointOperation op : operations) {
                if (op.httpMethod().equals(httpMethod) && op.hasSelector() == hasSelector) {
                    return op;
                }
            }
            return null;
        }
    }

    /**
     * A single operation on an endpoint.
     */
    public record EndpointOperation(
            Object bean,
            Method method,
            String httpMethod,
            boolean hasSelector
    ) {
    }
}
```

#### `ActuatorHandlerMapping.java` [NEW]

```java
package com.iris.boot.actuate.endpoint.web;

import java.lang.reflect.Method;
import java.lang.reflect.Parameter;
import java.util.LinkedHashMap;
import java.util.Map;

import com.fasterxml.jackson.databind.ObjectMapper;

import com.iris.boot.actuate.endpoint.EndpointDiscoverer;
import com.iris.boot.actuate.endpoint.EndpointDiscoverer.DiscoveredEndpoint;
import com.iris.boot.actuate.endpoint.EndpointDiscoverer.EndpointOperation;
import com.iris.boot.actuate.endpoint.annotation.Selector;
import com.iris.framework.context.ApplicationContext;
import com.iris.framework.web.servlet.HandlerMapping;
import com.iris.framework.web.servlet.HandlerMethod;

import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

/**
 * A {@link HandlerMapping} that exposes actuator endpoints as HTTP resources
 * under a configurable base path (default: {@code /actuator}).
 *
 * <p>This handler mapping uses a self-dispatching pattern: when it matches
 * a request, it returns a {@link HandlerMethod} pointing to its own
 * {@link #handleActuatorRequest(HttpServletRequest, HttpServletResponse)}
 * method. This method then internally routes to the correct endpoint
 * operation and serializes the result as JSON.
 *
 * <h3>URL Structure</h3>
 * <pre>
 * GET  /actuator                     → links (list of available endpoints)
 * GET  /actuator/{endpointId}        → endpoint @ReadOperation (no selector)
 * GET  /actuator/{endpointId}/{sel}  → endpoint @ReadOperation (with @Selector)
 * POST /actuator/{endpointId}        → endpoint @WriteOperation
 * </pre>
 *
 * <h3>Integration with DispatcherServlet</h3>
 * <p>This mapping is registered as a bean by {@code ActuatorAutoConfiguration}.
 * The {@code DispatcherServlet} discovers it during {@code initHandlerMappings()}
 * alongside the default {@code RequestMappingHandlerMapping}.
 *
 * <p>When a request arrives:
 * <ol>
 *   <li>DispatcherServlet iterates its handler mappings</li>
 *   <li>The default mapping returns null for {@code /actuator/*} paths</li>
 *   <li>This mapping matches and returns the self-dispatching handler method</li>
 *   <li>The adapter invokes the handler, which writes JSON directly</li>
 * </ol>
 *
 * <p>The matched operation context is passed via a request attribute
 * ({@code actuator.matched.operation}) — the standard Servlet API mechanism
 * for passing data between components in the same request.
 *
 * <p>In the real Spring Boot:
 * <ul>
 *   <li>{@code WebMvcEndpointHandlerMapping} extends Spring MVC's
 *       {@code RequestMappingInfoHandlerMapping}</li>
 *   <li>Each operation gets its own {@code RequestMappingInfo} registration</li>
 *   <li>Uses {@code ServletWebOperationAdapter} for request/response translation</li>
 *   <li>Supports HATEOAS-style links at the actuator root</li>
 * </ul>
 *
 * @see HandlerMapping
 * @see EndpointDiscoverer
 * @see org.springframework.boot.webmvc.actuate.endpoint.web.WebMvcEndpointHandlerMapping
 */
public class ActuatorHandlerMapping implements HandlerMapping {

    /** Request attribute key for passing the matched operation. */
    private static final String MATCHED_OPERATION_ATTR =
            ActuatorHandlerMapping.class.getName() + ".matchedOperation";

    private final String basePath;
    private final Map<String, DiscoveredEndpoint> endpoints;
    private final ObjectMapper objectMapper;
    private final HandlerMethod dispatchHandlerMethod;

    /**
     * Create a new ActuatorHandlerMapping.
     *
     * @param context      the application context for endpoint discovery
     * @param objectMapper the ObjectMapper for JSON serialization
     */
    public ActuatorHandlerMapping(ApplicationContext context, ObjectMapper objectMapper) {
        this(context, objectMapper, "/actuator");
    }

    /**
     * Create a new ActuatorHandlerMapping with a custom base path.
     *
     * @param context      the application context for endpoint discovery
     * @param objectMapper the ObjectMapper for JSON serialization
     * @param basePath     the base path for actuator endpoints
     */
    public ActuatorHandlerMapping(ApplicationContext context, ObjectMapper objectMapper,
                                   String basePath) {
        this.basePath = basePath;
        this.objectMapper = objectMapper;
        this.endpoints = new EndpointDiscoverer().discoverEndpoints(context);

        // Create the self-dispatching handler method
        try {
            Method handler = getClass().getMethod("handleActuatorRequest",
                    HttpServletRequest.class, HttpServletResponse.class);
            this.dispatchHandlerMethod = new HandlerMethod(this, handler);
        } catch (NoSuchMethodException e) {
            throw new RuntimeException("Cannot find handleActuatorRequest method", e);
        }
    }

    // -----------------------------------------------------------------------
    // HandlerMapping implementation
    // -----------------------------------------------------------------------

    /**
     * Match the request to an actuator endpoint operation.
     *
     * <p>If matched, stores the matched operation as a request attribute
     * and returns the self-dispatching handler method.
     */
    @Override
    public HandlerMethod getHandler(HttpServletRequest request) {
        String path = getRequestPath(request);
        String httpMethod = request.getMethod();

        if (!path.startsWith(basePath)) {
            return null;
        }

        // Extract the path after the base path
        String remaining = path.substring(basePath.length());
        if (remaining.startsWith("/")) {
            remaining = remaining.substring(1);
        }

        MatchedOperation matched = null;

        if (remaining.isEmpty()) {
            // /actuator — links endpoint
            if ("GET".equals(httpMethod)) {
                matched = new MatchedOperation(null, null, null);
            }
        } else {
            // /actuator/{endpointId} or /actuator/{endpointId}/{selector}
            String[] parts = remaining.split("/", 2);
            String endpointId = parts[0];
            String selectorValue = parts.length > 1 ? parts[1] : null;

            DiscoveredEndpoint endpoint = endpoints.get(endpointId);
            if (endpoint != null) {
                boolean hasSelector = selectorValue != null && !selectorValue.isEmpty();
                EndpointOperation operation = endpoint.findOperation(httpMethod, hasSelector);
                if (operation != null) {
                    matched = new MatchedOperation(endpoint, operation, selectorValue);
                }
            }
        }

        if (matched != null) {
            request.setAttribute(MATCHED_OPERATION_ATTR, matched);
            return dispatchHandlerMethod;
        }

        return null;
    }

    // -----------------------------------------------------------------------
    // Self-dispatching handler method
    // -----------------------------------------------------------------------

    /**
     * Handle an actuator request by invoking the matched endpoint operation
     * and writing the result as JSON.
     *
     * <p>This method is invoked by the {@code RequestMappingHandlerAdapter}
     * which resolves the {@code HttpServletRequest} and
     * {@code HttpServletResponse} parameters. The matched operation is
     * retrieved from the request attribute set by {@link #getHandler}.
     *
     * <p>The method writes the response directly and returns void, so the
     * adapter does not attempt any further response handling.
     */
    public void handleActuatorRequest(HttpServletRequest request,
                                       HttpServletResponse response) throws Exception {
        MatchedOperation matched = (MatchedOperation) request.getAttribute(MATCHED_OPERATION_ATTR);
        request.removeAttribute(MATCHED_OPERATION_ATTR);

        if (matched == null) {
            response.sendError(HttpServletResponse.SC_NOT_FOUND);
            return;
        }

        // Handle the links endpoint (/actuator)
        if (matched.endpoint == null) {
            writeLinksResponse(response);
            return;
        }

        // Invoke the endpoint operation
        Object result = invokeOperation(matched);

        // Write the response
        if (result == null) {
            if ("GET".equals(request.getMethod())) {
                response.sendError(HttpServletResponse.SC_NOT_FOUND);
            } else {
                response.setStatus(HttpServletResponse.SC_NO_CONTENT);
            }
            return;
        }

        response.setContentType("application/json;charset=UTF-8");
        objectMapper.writeValue(response.getWriter(), result);
    }

    // -----------------------------------------------------------------------
    // Internal helpers
    // -----------------------------------------------------------------------

    /**
     * Write the actuator links response — a map of endpoint IDs to their URLs.
     */
    private void writeLinksResponse(HttpServletResponse response) throws Exception {
        Map<String, Object> links = new LinkedHashMap<>();
        Map<String, Object> linkMap = new LinkedHashMap<>();

        // Self link
        linkMap.put("self", Map.of("href", basePath));

        // Links for each discovered endpoint
        for (String endpointId : endpoints.keySet()) {
            linkMap.put(endpointId, Map.of("href", basePath + "/" + endpointId));
        }

        links.put("_links", linkMap);

        response.setContentType("application/json;charset=UTF-8");
        objectMapper.writeValue(response.getWriter(), links);
    }

    /**
     * Invoke the matched endpoint operation, resolving @Selector parameters.
     */
    private Object invokeOperation(MatchedOperation matched) throws Exception {
        EndpointOperation operation = matched.operation;
        Method method = operation.method();
        method.setAccessible(true);

        // Resolve parameters
        Parameter[] params = method.getParameters();
        Object[] args = new Object[params.length];

        for (int i = 0; i < params.length; i++) {
            if (params[i].isAnnotationPresent(Selector.class)) {
                args[i] = matched.selectorValue;
            } else {
                args[i] = null;
            }
        }

        return method.invoke(operation.bean(), args);
    }

    /**
     * Extract the request path from the request URI, stripping any context path.
     */
    private String getRequestPath(HttpServletRequest request) {
        String uri = request.getRequestURI();
        String contextPath = request.getContextPath();
        if (contextPath != null && !contextPath.isEmpty() && uri.startsWith(contextPath)) {
            uri = uri.substring(contextPath.length());
        }
        // Remove trailing slash
        if (uri.length() > 1 && uri.endsWith("/")) {
            uri = uri.substring(0, uri.length() - 1);
        }
        return uri;
    }

    /**
     * Return the number of discovered endpoints (useful for testing).
     */
    public int getEndpointCount() {
        return endpoints.size();
    }

    // -----------------------------------------------------------------------
    // Internal types
    // -----------------------------------------------------------------------

    /**
     * Holds the matched endpoint, operation, and selector value for a request.
     */
    private record MatchedOperation(
            DiscoveredEndpoint endpoint,
            EndpointOperation operation,
            String selectorValue
    ) {
    }
}
```

#### `ActuatorAutoConfiguration.java` [NEW]

```java
package com.iris.boot.actuate.autoconfigure;

import java.util.ArrayList;
import java.util.LinkedHashMap;
import java.util.Map;

import com.fasterxml.jackson.databind.ObjectMapper;

import com.iris.boot.actuate.endpoint.web.ActuatorHandlerMapping;
import com.iris.boot.actuate.health.HealthEndpoint;
import com.iris.boot.actuate.health.HealthIndicator;
import com.iris.boot.actuate.health.StatusAggregator;
import com.iris.boot.actuate.info.InfoContributor;
import com.iris.boot.actuate.info.InfoEndpoint;
import com.iris.boot.actuate.metrics.MetricsEndpoint;
import com.iris.boot.autoconfigure.AutoConfiguration;
import com.iris.boot.autoconfigure.condition.ConditionalOnClass;
import com.iris.boot.autoconfigure.condition.ConditionalOnMissingBean;
import com.iris.framework.beans.factory.support.DefaultBeanFactory;
import com.iris.framework.context.AnnotationConfigApplicationContext;
import com.iris.framework.context.ApplicationContext;
import com.iris.framework.context.annotation.Bean;
import com.iris.micrometer.core.MeterRegistry;
import com.iris.micrometer.core.simple.SimpleMeterRegistry;

/**
 * Auto-configuration for Spring Boot Actuator — discovers endpoints and
 * exposes them over HTTP.
 *
 * <p>This auto-configuration:
 * <ol>
 *   <li>Creates default endpoint beans ({@link HealthEndpoint},
 *       {@link InfoEndpoint}, {@link MetricsEndpoint}) if not already defined</li>
 *   <li>Creates a default {@link StatusAggregator} and {@link MeterRegistry}
 *       if not already defined</li>
 *   <li>Creates the {@link ActuatorHandlerMapping} that exposes all
 *       {@code @Endpoint} beans over HTTP under {@code /actuator/}</li>
 * </ol>
 *
 * <p>The auto-configuration pattern ensures that user-defined beans take
 * precedence over defaults: each {@code @Bean} method is annotated with
 * {@code @ConditionalOnMissingBean} so it backs off when the user provides
 * their own implementation.
 *
 * <p>In the real Spring Boot, actuator auto-configuration is split across
 * many classes:
 * <ul>
 *   <li>{@code EndpointAutoConfiguration} — parameter mapper, caching</li>
 *   <li>{@code WebEndpointAutoConfiguration} — endpoint discoverer,
 *       path mapping, exposure filtering</li>
 *   <li>{@code HealthEndpointAutoConfiguration} — health contributors,
 *       groups, access control</li>
 *   <li>{@code WebMvcEndpointManagementContextConfiguration} — the MVC
 *       handler mapping that exposes endpoints over HTTP</li>
 * </ul>
 *
 * <p>We collapse all of this into a single auto-configuration class
 * for simplicity.
 *
 * @see ActuatorHandlerMapping
 * @see HealthEndpoint
 * @see InfoEndpoint
 * @see MetricsEndpoint
 */
@AutoConfiguration
@ConditionalOnClass(name = "com.iris.boot.actuate.endpoint.annotation.Endpoint")
public class ActuatorAutoConfiguration {

    // -----------------------------------------------------------------------
    // Infrastructure beans
    // -----------------------------------------------------------------------

    /**
     * Create a default {@link StatusAggregator} if none is defined.
     */
    @Bean
    @ConditionalOnMissingBean
    public StatusAggregator statusAggregator() {
        return new StatusAggregator();
    }

    /**
     * Create a default {@link SimpleMeterRegistry} if no {@link MeterRegistry}
     * is defined.
     *
     * <p>This ensures the metrics endpoint always has a registry to read from.
     * Users can override this by defining their own {@code MeterRegistry} bean.
     */
    @Bean
    @ConditionalOnMissingBean
    public MeterRegistry meterRegistry() {
        return new SimpleMeterRegistry();
    }

    // -----------------------------------------------------------------------
    // Endpoint beans
    // -----------------------------------------------------------------------

    /**
     * Create the {@link HealthEndpoint} with all discovered
     * {@link HealthIndicator} beans.
     *
     * <p>Takes the {@link ApplicationContext} to discover health indicators
     * by type (since there may be multiple). Strips "HealthIndicator" or
     * "healthIndicator" suffix from bean names to derive component names.
     */
    @Bean
    @ConditionalOnMissingBean
    public HealthEndpoint healthEndpoint(ApplicationContext context,
                                          StatusAggregator statusAggregator) {
        DefaultBeanFactory factory =
                ((AnnotationConfigApplicationContext) context).getBeanFactory();
        Map<String, HealthIndicator> rawIndicators =
                factory.getBeansOfType(HealthIndicator.class);

        // Derive friendly component names from bean names
        Map<String, HealthIndicator> indicators = new LinkedHashMap<>();
        for (Map.Entry<String, HealthIndicator> entry : rawIndicators.entrySet()) {
            String name = deriveComponentName(entry.getKey());
            indicators.put(name, entry.getValue());
        }

        return new HealthEndpoint(indicators, statusAggregator);
    }

    /**
     * Create the {@link InfoEndpoint} with all discovered
     * {@link InfoContributor} beans.
     */
    @Bean
    @ConditionalOnMissingBean
    public InfoEndpoint infoEndpoint(ApplicationContext context) {
        DefaultBeanFactory factory =
                ((AnnotationConfigApplicationContext) context).getBeanFactory();
        Map<String, InfoContributor> contributors =
                factory.getBeansOfType(InfoContributor.class);
        return new InfoEndpoint(new ArrayList<>(contributors.values()));
    }

    /**
     * Create the {@link MetricsEndpoint} with the {@link MeterRegistry}.
     */
    @Bean
    @ConditionalOnMissingBean
    public MetricsEndpoint metricsEndpoint(MeterRegistry registry) {
        return new MetricsEndpoint(registry);
    }

    // -----------------------------------------------------------------------
    // Handler mapping (HTTP exposure)
    // -----------------------------------------------------------------------

    /**
     * Create the {@link ActuatorHandlerMapping} that discovers all
     * {@code @Endpoint} beans and maps their operations to HTTP paths.
     *
     * <p>This mapping is registered as a bean, which the
     * {@code DispatcherServlet} discovers during its
     * {@code initHandlerMappings()} phase.
     */
    @Bean
    public ActuatorHandlerMapping actuatorHandlerMapping(ApplicationContext context,
                                                          ObjectMapper objectMapper) {
        return new ActuatorHandlerMapping(context, objectMapper);
    }

    // -----------------------------------------------------------------------
    // Helpers
    // -----------------------------------------------------------------------

    /**
     * Derive a friendly component name from a bean name.
     *
     * <p>Strips common suffixes like "HealthIndicator" and converts
     * to a simple lowercase name. For example:
     * <ul>
     *   <li>{@code "diskSpaceHealthIndicator"} → {@code "diskSpace"}</li>
     *   <li>{@code "dbHealth"} → {@code "dbHealth"}</li>
     *   <li>{@code "myIndicator"} → {@code "myIndicator"}</li>
     * </ul>
     */
    static String deriveComponentName(String beanName) {
        if (beanName.endsWith("HealthIndicator")) {
            return beanName.substring(0, beanName.length() - "HealthIndicator".length());
        }
        if (beanName.endsWith("Indicator")) {
            return beanName.substring(0, beanName.length() - "Indicator".length());
        }
        return beanName;
    }
}
```

#### `com.iris.boot.autoconfigure.AutoConfiguration.imports` [NEW]

```
# Iris Boot Actuator Auto-Configuration Candidates
com.iris.boot.actuate.autoconfigure.ActuatorAutoConfiguration
```

### Modified Code

#### `DispatcherServlet.java` [MODIFIED]

```java
package com.iris.framework.web.servlet;

import java.io.IOException;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;

import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServlet;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import com.iris.framework.beans.factory.support.DefaultBeanFactory;
import com.iris.framework.context.AnnotationConfigApplicationContext;
import com.iris.framework.context.ApplicationContext;
import com.iris.framework.web.http.converter.HttpMessageConverter;

/**
 * Central dispatcher for HTTP request handling — the <strong>front controller</strong>
 * of the Iris MVC framework.
 *
 * <p>The DispatcherServlet is the single entry point for all HTTP requests.
 * It receives each request, finds the appropriate handler (via {@link HandlerMapping}),
 * invokes it (via {@link HandlerAdapter}), and the handler writes the response.
 *
 * <h3>Initialization</h3>
 * <p>The DispatcherServlet is created by {@code ServletWebServerApplicationContext}
 * during {@code onRefresh()} and registered with Tomcat. When Tomcat starts
 * (after all beans are instantiated), it calls {@link #init()} which triggers
 * strategy initialization:
 * <ul>
 *   <li>{@link #initHandlerMappings()} — creates a {@link RequestMappingHandlerMapping}
 *       that scans all {@code @Controller} beans</li>
 *   <li>{@link #initHandlerAdapters()} — creates a {@link RequestMappingHandlerAdapter}
 *       that knows how to invoke handler methods</li>
 * </ul>
 *
 * <h3>Request Processing</h3>
 * <p>For each request, {@link #service(HttpServletRequest, HttpServletResponse)}
 * delegates to {@link #doDispatch(HttpServletRequest, HttpServletResponse)}:
 * <ol>
 *   <li>Find handler via HandlerMapping</li>
 *   <li>Find adapter via HandlerAdapter.supports()</li>
 *   <li>Invoke handler via HandlerAdapter.handle()</li>
 *   <li>If no handler found, return 404</li>
 * </ol>
 *
 * <h3>Simplifications from Real Spring</h3>
 * <ul>
 *   <li>No {@code HandlerExecutionChain} / interceptors</li>
 *   <li>No {@code ModelAndView} / view resolution</li>
 *   <li>No {@code HandlerExceptionResolver} — exceptions become 500</li>
 *   <li>No multipart resolution, locale resolution, theme resolution</li>
 *   <li>No {@code FlashMap} support</li>
 *   <li>Single HandlerMapping and HandlerAdapter (not lists)</li>
 * </ul>
 *
 * @see org.springframework.web.servlet.DispatcherServlet
 */
public class DispatcherServlet extends HttpServlet {

    private final ApplicationContext applicationContext;

    private final List<HandlerMapping> handlerMappings = new ArrayList<>();
    private HandlerAdapter handlerAdapter;

    /**
     * Create a new DispatcherServlet backed by the given ApplicationContext.
     *
     * <p>The context is stored but not used until {@link #init()} is called
     * by the servlet container (Tomcat). By that time, all beans have been
     * instantiated by the context's {@code refresh()} method.
     */
    public DispatcherServlet(ApplicationContext applicationContext) {
        this.applicationContext = applicationContext;
    }

    // -----------------------------------------------------------------------
    // Initialization — called by Tomcat on startup
    // -----------------------------------------------------------------------

    /**
     * Initialize the servlet's strategies (HandlerMapping, HandlerAdapter).
     *
     * <p>In the real Spring Framework, {@code DispatcherServlet.init()} calls
     * {@code initWebApplicationContext()} → {@code onRefresh()} →
     * {@code initStrategies()}, which discovers strategy beans from the
     * context. We create them directly since we have one of each.
     */
    @Override
    public void init() throws ServletException {
        initHandlerMappings();
        initHandlerAdapters();
    }

    /**
     * Initialize the HandlerMapping strategies.
     *
     * <p>Creates the default {@link RequestMappingHandlerMapping} which scans
     * the ApplicationContext for {@code @Controller} beans, then discovers
     * any additional {@link HandlerMapping} beans registered in the context
     * (e.g., by actuator auto-configuration).
     *
     * <p>In the real Spring Framework, {@code DispatcherServlet.initHandlerMappings()}
     * discovers all {@code HandlerMapping} beans from the ApplicationContext and
     * orders them by priority. We always put the default mapping first, then
     * append any additional bean-registered mappings.
     *
     * <p>This enhancement (from Feature 24) enables the actuator to register
     * its own {@code ActuatorHandlerMapping} bean that handles {@code /actuator/*}
     * requests separately from the application's {@code @Controller} mappings.
     */
    private void initHandlerMappings() {
        // Default handler mapping for @Controller beans
        this.handlerMappings.add(new RequestMappingHandlerMapping(applicationContext));

        // Discover additional HandlerMapping beans from the ApplicationContext
        try {
            DefaultBeanFactory factory =
                    ((AnnotationConfigApplicationContext) applicationContext).getBeanFactory();
            Map<String, HandlerMapping> mappingBeans =
                    factory.getBeansOfType(HandlerMapping.class);
            for (HandlerMapping mapping : mappingBeans.values()) {
                if (!this.handlerMappings.contains(mapping)) {
                    this.handlerMappings.add(mapping);
                }
            }
        } catch (Exception e) {
            // No additional handler mappings found — that's fine
        }
    }

    /**
     * Initialize the HandlerAdapter strategy.
     *
     * <p>Creates a {@link RequestMappingHandlerAdapter} with any
     * {@link HttpMessageConverter} beans found in the ApplicationContext.
     * These converters enable {@code @ResponseBody} serialization
     * (e.g., Jackson for JSON output).
     */
    private void initHandlerAdapters() {
        List<HttpMessageConverter> converters = findMessageConverters();
        this.handlerAdapter = new RequestMappingHandlerAdapter(converters);
    }

    /**
     * Discover {@link HttpMessageConverter} beans registered in the
     * ApplicationContext.
     *
     * <p>In the real Spring Framework, the message converters are configured
     * by {@code WebMvcAutoConfiguration} and can be customized via
     * {@code WebMvcConfigurer.configureMessageConverters()}. Our simplified
     * version simply looks up all converter beans by type.
     */
    private List<HttpMessageConverter> findMessageConverters() {
        try {
            DefaultBeanFactory factory =
                    ((AnnotationConfigApplicationContext) applicationContext).getBeanFactory();
            Map<String, HttpMessageConverter> converterBeans =
                    factory.getBeansOfType(HttpMessageConverter.class);
            return new ArrayList<>(converterBeans.values());
        } catch (Exception e) {
            return List.of();
        }
    }

    // -----------------------------------------------------------------------
    // Request processing
    // -----------------------------------------------------------------------

    /**
     * Process an incoming HTTP request.
     *
     * <p>Overrides {@code HttpServlet.service()} to handle ALL HTTP methods
     * through our dispatch pipeline. In the real Spring Framework,
     * {@code FrameworkServlet.service()} overrides this and dispatches to
     * {@code processRequest()} → {@code doService()} → {@code doDispatch()}.
     */
    @Override
    protected void service(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException {
        try {
            doDispatch(request, response);
        } catch (Exception ex) {
            if (ex instanceof ServletException se) {
                throw se;
            }
            if (ex instanceof IOException ioe) {
                throw ioe;
            }
            throw new ServletException("Request processing failed", ex);
        }
    }

    /**
     * The central dispatch method — the heart of Spring MVC.
     *
     * <p>This is the simplified version of {@code DispatcherServlet.doDispatch()}
     * (line 935 in the real Spring Framework). The real method has 9 steps
     * including multipart handling, interceptors, async support, and exception
     * resolution. We implement the core 3-step flow:
     * <ol>
     *   <li>Find handler (HandlerMapping)</li>
     *   <li>Find adapter (HandlerAdapter)</li>
     *   <li>Invoke handler (HandlerAdapter.handle())</li>
     * </ol>
     */
    private void doDispatch(HttpServletRequest request, HttpServletResponse response)
            throws Exception {
        // Step 1: Find handler — iterate all HandlerMappings, first match wins
        HandlerMethod handler = null;
        for (HandlerMapping mapping : handlerMappings) {
            handler = mapping.getHandler(request);
            if (handler != null) {
                break;
            }
        }
        if (handler == null) {
            response.sendError(HttpServletResponse.SC_NOT_FOUND,
                    "No handler found for " + request.getMethod() + " " + request.getRequestURI());
            return;
        }

        // Step 2: Find adapter (we have only one, but follow the pattern)
        if (!handlerAdapter.supports(handler)) {
            response.sendError(HttpServletResponse.SC_INTERNAL_SERVER_ERROR,
                    "No adapter for handler: " + handler);
            return;
        }

        // Step 3: Invoke handler
        handlerAdapter.handle(request, response, handler);
    }

    /**
     * Return the ApplicationContext this servlet is associated with.
     */
    public ApplicationContext getApplicationContext() {
        return applicationContext;
    }
}
```

#### `AnnotationConfigApplicationContext.java` [MODIFIED]

The key change is the addition of `prepareBeanFactory()` as Step 0 in `refresh()`:

```java
@Override
public void refresh() {
    try {
        // Step 0: Register infrastructure beans (context itself)
        prepareBeanFactory();

        // Step 1: Process @Configuration classes — component scanning + @Bean methods
        invokeBeanFactoryPostProcessors();

        // ... remaining steps unchanged ...
    }
}

/**
 * Configure the bean factory with standard infrastructure beans.
 *
 * <p>Registers the ApplicationContext itself as a singleton so that
 * other beans (e.g., auto-configuration beans) can depend on it.
 * This mirrors the real Spring Framework's
 * {@code AbstractApplicationContext.prepareBeanFactory()} (line 715),
 * which registers the context, environment, system properties, and
 * system environment as resolvable dependencies.
 *
 * <p>We register only the context itself — sufficient for our
 * simplified framework where beans that need context access can
 * declare {@code ApplicationContext} as a {@code @Bean} method parameter.
 */
private void prepareBeanFactory() {
    this.beanFactory.registerSingleton("applicationContext", this);
}
```

### Test Code

#### `HealthTest.java` [NEW]

```java
package com.iris.boot.actuate.health;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.*;

/**
 * Tests for {@link Health}, {@link Status}, and the {@link Health.Builder}.
 */
class HealthTest {

    @Test
    void shouldCreateHealthWithUpStatus_WhenUsingUpFactory() {
        Health health = Health.up().build();

        assertThat(health.getStatus()).isEqualTo(Status.UP);
        assertThat(health.getDetails()).isEmpty();
    }

    @Test
    void shouldCreateHealthWithDownStatus_WhenUsingDownFactory() {
        Health health = Health.down().build();

        assertThat(health.getStatus()).isEqualTo(Status.DOWN);
    }

    @Test
    void shouldIncludeDetails_WhenAddedViaBuilder() {
        Health health = Health.up()
                .withDetail("database", "PostgreSQL")
                .withDetail("responseTime", "12ms")
                .build();

        assertThat(health.getStatus()).isEqualTo(Status.UP);
        assertThat(health.getDetails()).containsEntry("database", "PostgreSQL");
        assertThat(health.getDetails()).containsEntry("responseTime", "12ms");
    }

    @Test
    void shouldIncludeExceptionDetails_WhenUsingWithException() {
        RuntimeException ex = new RuntimeException("Connection refused");
        Health health = Health.down(ex).build();

        assertThat(health.getStatus()).isEqualTo(Status.DOWN);
        assertThat(health.getDetails()).containsKey("error");
        assertThat((String) health.getDetails().get("error"))
                .contains("RuntimeException")
                .contains("Connection refused");
    }

    @Test
    void shouldCreateHealthWithCustomStatus() {
        Status degraded = new Status("DEGRADED");
        Health health = Health.status(degraded)
                .withDetail("reason", "High latency")
                .build();

        assertThat(health.getStatus().getCode()).isEqualTo("DEGRADED");
        assertThat(health.getDetails()).containsEntry("reason", "High latency");
    }

    @Test
    void shouldHaveImmutableDetails() {
        Health health = Health.up()
                .withDetail("key", "value")
                .build();

        assertThatThrownBy(() -> health.getDetails().put("new", "entry"))
                .isInstanceOf(UnsupportedOperationException.class);
    }

    @Test
    void shouldCreateUnknownHealth() {
        Health health = Health.unknown().build();
        assertThat(health.getStatus()).isEqualTo(Status.UNKNOWN);
    }

    @Test
    void shouldCreateOutOfServiceHealth() {
        Health health = Health.outOfService().build();
        assertThat(health.getStatus()).isEqualTo(Status.OUT_OF_SERVICE);
    }

    // --- Status tests ---

    @Test
    void shouldCompareStatusesByCode() {
        assertThat(Status.UP).isEqualTo(new Status("UP"));
        assertThat(Status.DOWN).isNotEqualTo(Status.UP);
    }

    @Test
    void shouldReturnCodeAsToString() {
        assertThat(Status.UP.toString()).isEqualTo("UP");
        assertThat(Status.DOWN.toString()).isEqualTo("DOWN");
    }
}
```

#### `StatusAggregatorTest.java` [NEW]

```java
package com.iris.boot.actuate.health;

import java.util.Set;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.*;

/**
 * Tests for {@link StatusAggregator}.
 */
class StatusAggregatorTest {

    private final StatusAggregator aggregator = new StatusAggregator();

    @Test
    void shouldReturnDown_WhenAnyIndicatorIsDown() {
        Status result = aggregator.getAggregateStatus(
                Set.of(Status.UP, Status.DOWN));

        assertThat(result).isEqualTo(Status.DOWN);
    }

    @Test
    void shouldReturnUp_WhenAllIndicatorsAreUp() {
        Status result = aggregator.getAggregateStatus(
                Set.of(Status.UP));

        assertThat(result).isEqualTo(Status.UP);
    }

    @Test
    void shouldReturnOutOfService_WhenOutOfServiceAndUp() {
        Status result = aggregator.getAggregateStatus(
                Set.of(Status.UP, Status.OUT_OF_SERVICE));

        assertThat(result).isEqualTo(Status.OUT_OF_SERVICE);
    }

    @Test
    void shouldReturnUnknown_WhenNoStatuses() {
        Status result = aggregator.getAggregateStatus(Set.of());

        assertThat(result).isEqualTo(Status.UNKNOWN);
    }

    @Test
    void shouldReturnDown_WhenDownAndOutOfService() {
        // DOWN is worse than OUT_OF_SERVICE
        Status result = aggregator.getAggregateStatus(
                Set.of(Status.DOWN, Status.OUT_OF_SERVICE));

        assertThat(result).isEqualTo(Status.DOWN);
    }

    @Test
    void shouldReturnUnknown_WhenUnknownAndUp() {
        Status result = aggregator.getAggregateStatus(
                Set.of(Status.UNKNOWN, Status.UP));

        assertThat(result).isEqualTo(Status.UNKNOWN);
    }
}
```

#### `EndpointDiscovererTest.java` [NEW]

```java
package com.iris.boot.actuate.endpoint;

import java.util.Map;

import com.iris.boot.actuate.endpoint.annotation.Endpoint;
import com.iris.boot.actuate.endpoint.annotation.ReadOperation;
import com.iris.boot.actuate.endpoint.annotation.Selector;
import com.iris.boot.actuate.endpoint.annotation.WriteOperation;
import com.iris.framework.beans.factory.config.BeanDefinition;
import com.iris.framework.context.AnnotationConfigApplicationContext;
import com.iris.framework.context.annotation.Bean;
import com.iris.framework.context.annotation.Configuration;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.*;

/**
 * Tests for {@link EndpointDiscoverer}.
 */
class EndpointDiscovererTest {

    @Test
    void shouldDiscoverEndpointBeans_WhenAnnotatedWithEndpoint() {
        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext();
        context.getBeanFactory().registerSingleton("testEndpoint", new TestEndpoint());
        context.register(EmptyConfig.class);
        context.refresh();

        EndpointDiscoverer discoverer = new EndpointDiscoverer();
        Map<String, EndpointDiscoverer.DiscoveredEndpoint> endpoints =
                discoverer.discoverEndpoints(context);

        assertThat(endpoints).containsKey("test");
        EndpointDiscoverer.DiscoveredEndpoint endpoint = endpoints.get("test");
        assertThat(endpoint.id()).isEqualTo("test");
        assertThat(endpoint.operations()).isNotEmpty();

        context.close();
    }

    @Test
    void shouldDiscoverReadOperations_WhenAnnotated() {
        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext();
        context.getBeanFactory().registerSingleton("testEndpoint", new TestEndpoint());
        context.register(EmptyConfig.class);
        context.refresh();

        EndpointDiscoverer discoverer = new EndpointDiscoverer();
        Map<String, EndpointDiscoverer.DiscoveredEndpoint> endpoints =
                discoverer.discoverEndpoints(context);

        EndpointDiscoverer.DiscoveredEndpoint endpoint = endpoints.get("test");
        EndpointDiscoverer.EndpointOperation readOp =
                endpoint.findOperation("GET", false);
        assertThat(readOp).isNotNull();
        assertThat(readOp.httpMethod()).isEqualTo("GET");
        assertThat(readOp.hasSelector()).isFalse();

        context.close();
    }

    @Test
    void shouldDiscoverWriteOperations_WhenAnnotated() {
        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext();
        context.getBeanFactory().registerSingleton("testEndpoint", new TestEndpoint());
        context.register(EmptyConfig.class);
        context.refresh();

        EndpointDiscoverer discoverer = new EndpointDiscoverer();
        Map<String, EndpointDiscoverer.DiscoveredEndpoint> endpoints =
                discoverer.discoverEndpoints(context);

        EndpointDiscoverer.DiscoveredEndpoint endpoint = endpoints.get("test");
        EndpointDiscoverer.EndpointOperation writeOp =
                endpoint.findOperation("POST", false);
        assertThat(writeOp).isNotNull();

        context.close();
    }

    @Test
    void shouldDetectSelectorParameters_WhenAnnotated() {
        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext();
        context.getBeanFactory().registerSingleton("selectorEndpoint", new SelectorEndpoint());
        context.register(EmptyConfig.class);
        context.refresh();

        EndpointDiscoverer discoverer = new EndpointDiscoverer();
        Map<String, EndpointDiscoverer.DiscoveredEndpoint> endpoints =
                discoverer.discoverEndpoints(context);

        EndpointDiscoverer.DiscoveredEndpoint endpoint = endpoints.get("selector");
        EndpointDiscoverer.EndpointOperation selectorOp =
                endpoint.findOperation("GET", true);
        assertThat(selectorOp).isNotNull();
        assertThat(selectorOp.hasSelector()).isTrue();

        context.close();
    }

    @Test
    void shouldReturnEmptyMap_WhenNoEndpoints() {
        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext();
        context.register(EmptyConfig.class);
        context.refresh();

        EndpointDiscoverer discoverer = new EndpointDiscoverer();
        Map<String, EndpointDiscoverer.DiscoveredEndpoint> endpoints =
                discoverer.discoverEndpoints(context);

        assertThat(endpoints).isEmpty();

        context.close();
    }

    // --- Test endpoint classes ---

    @Endpoint(id = "test")
    static class TestEndpoint {
        @ReadOperation
        public Map<String, String> read() {
            return Map.of("key", "value");
        }

        @WriteOperation
        public void write() {
        }
    }

    @Endpoint(id = "selector")
    static class SelectorEndpoint {
        @ReadOperation
        public String readWithSelector(@Selector String name) {
            return "value-" + name;
        }
    }

    @Configuration
    static class EmptyConfig {
    }
}
```

#### `ActuatorIntegrationTest.java` [NEW]

```java
package com.iris.boot.actuate.integration;

import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

import com.fasterxml.jackson.databind.ObjectMapper;

import com.iris.boot.actuate.endpoint.annotation.Endpoint;
import com.iris.boot.actuate.endpoint.annotation.ReadOperation;
import com.iris.boot.actuate.endpoint.annotation.Selector;
import com.iris.boot.actuate.endpoint.web.ActuatorHandlerMapping;
import com.iris.boot.actuate.health.Health;
import com.iris.boot.actuate.health.HealthEndpoint;
import com.iris.boot.actuate.health.HealthIndicator;
import com.iris.boot.actuate.health.StatusAggregator;
import com.iris.boot.actuate.info.InfoContributor;
import com.iris.boot.actuate.info.InfoEndpoint;
import com.iris.boot.actuate.metrics.MetricsEndpoint;
import com.iris.framework.context.AnnotationConfigApplicationContext;
import com.iris.framework.context.annotation.Bean;
import com.iris.framework.context.annotation.Configuration;
import com.iris.micrometer.core.MeterRegistry;
import com.iris.micrometer.core.simple.SimpleMeterRegistry;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.*;

/**
 * Integration tests for the actuator feature — verifies endpoint discovery,
 * handler mapping, and health/info/metrics endpoints working together.
 */
class ActuatorIntegrationTest {

    private AnnotationConfigApplicationContext context;

    @BeforeEach
    void setUp() {
        context = new AnnotationConfigApplicationContext();
    }

    @AfterEach
    void tearDown() {
        if (context != null) {
            context.close();
        }
    }

    @Test
    void shouldDiscoverHealthEndpoint_WhenConfigured() {
        context.register(HealthConfig.class);
        context.refresh();

        HealthEndpoint healthEndpoint = context.getBean(HealthEndpoint.class);
        Map<String, Object> health = healthEndpoint.health();

        assertThat(health).containsKey("status");
        assertThat(health.get("status")).isEqualTo("UP");
        assertThat(health).containsKey("components");

        @SuppressWarnings("unchecked")
        Map<String, Object> components = (Map<String, Object>) health.get("components");
        assertThat(components).containsKey("db");
    }

    @Test
    void shouldAggregateDownStatus_WhenOneIndicatorIsDown() {
        context.register(MixedHealthConfig.class);
        context.refresh();

        HealthEndpoint healthEndpoint = context.getBean(HealthEndpoint.class);
        Map<String, Object> health = healthEndpoint.health();

        assertThat(health.get("status")).isEqualTo("DOWN");
    }

    @Test
    void shouldCollectInfoFromContributors_WhenConfigured() {
        context.register(InfoConfig.class);
        context.refresh();

        InfoEndpoint infoEndpoint = context.getBean(InfoEndpoint.class);
        Map<String, Object> info = infoEndpoint.info();

        assertThat(info).containsKey("app");
        @SuppressWarnings("unchecked")
        Map<String, Object> appInfo = (Map<String, Object>) info.get("app");
        assertThat(appInfo).containsEntry("name", "test-app");
    }

    @Test
    void shouldExposeMetrics_WhenRegistryHasMeters() {
        context.register(MetricsConfig.class);
        context.refresh();

        // Register some metrics
        MeterRegistry registry = context.getBean(MeterRegistry.class);
        registry.counter("test.counter", "method", "GET").increment(5);

        MetricsEndpoint metricsEndpoint = context.getBean(MetricsEndpoint.class);
        Map<String, Object> listing = metricsEndpoint.listNames();

        @SuppressWarnings("unchecked")
        Iterable<String> names = (Iterable<String>) listing.get("names");
        assertThat(names).contains("test.counter");

        // Query specific metric
        Map<String, Object> detail = metricsEndpoint.metric("test.counter");
        assertThat(detail).containsEntry("name", "test.counter");

        @SuppressWarnings("unchecked")
        List<Map<String, Object>> measurements =
                (List<Map<String, Object>>) detail.get("measurements");
        assertThat(measurements).isNotEmpty();
        assertThat(measurements.get(0).get("value")).isEqualTo(5.0);
    }

    @Test
    void shouldReturnNull_WhenMetricNotFound() {
        context.register(MetricsConfig.class);
        context.refresh();

        MetricsEndpoint metricsEndpoint = context.getBean(MetricsEndpoint.class);
        Map<String, Object> detail = metricsEndpoint.metric("nonexistent");

        assertThat(detail).isNull();
    }

    @Test
    void shouldCreateActuatorHandlerMapping_WhenEndpointsExist() {
        context.register(FullActuatorConfig.class);
        context.refresh();

        ActuatorHandlerMapping mapping = context.getBean(ActuatorHandlerMapping.class);
        assertThat(mapping.getEndpointCount()).isGreaterThanOrEqualTo(3);
    }

    @Test
    void shouldHandleSelectorEndpoints_WhenMetricQueried() {
        context.register(MetricsConfig.class);
        context.refresh();

        MeterRegistry registry = context.getBean(MeterRegistry.class);
        registry.counter("http.requests", "method", "GET").increment(10);
        registry.counter("http.requests", "method", "POST").increment(3);

        MetricsEndpoint metricsEndpoint = context.getBean(MetricsEndpoint.class);
        Map<String, Object> detail = metricsEndpoint.metric("http.requests");

        assertThat(detail).isNotNull();

        @SuppressWarnings("unchecked")
        List<Map<String, Object>> tags =
                (List<Map<String, Object>>) detail.get("availableTags");
        assertThat(tags).isNotEmpty();
        assertThat(tags.get(0).get("tag")).isEqualTo("method");
    }

    @Test
    void shouldHandleExceptionInHealthIndicator_WhenIndicatorFails() {
        context.register(FailingHealthConfig.class);
        context.refresh();

        HealthEndpoint healthEndpoint = context.getBean(HealthEndpoint.class);
        Map<String, Object> health = healthEndpoint.health();

        assertThat(health.get("status")).isEqualTo("DOWN");
    }

    // -----------------------------------------------------------------------
    // Test configurations
    // -----------------------------------------------------------------------

    @Configuration
    static class HealthConfig {
        @Bean
        public HealthIndicator dbHealthIndicator() {
            return () -> Health.up()
                    .withDetail("database", "H2")
                    .build();
        }

        @Bean
        public StatusAggregator statusAggregator() {
            return new StatusAggregator();
        }

        @Bean
        public HealthEndpoint healthEndpoint(
                com.iris.framework.context.ApplicationContext context,
                StatusAggregator aggregator) {
            var factory = ((AnnotationConfigApplicationContext) context).getBeanFactory();
            var indicators = factory.getBeansOfType(HealthIndicator.class);
            Map<String, HealthIndicator> named = new LinkedHashMap<>();
            for (var entry : indicators.entrySet()) {
                String name = entry.getKey().replace("HealthIndicator", "")
                        .replace("Indicator", "");
                named.put(name.isEmpty() ? entry.getKey() : name, entry.getValue());
            }
            return new HealthEndpoint(named, aggregator);
        }
    }

    @Configuration
    static class MixedHealthConfig {
        @Bean
        public HealthIndicator upIndicator() {
            return () -> Health.up().build();
        }

        @Bean
        public HealthIndicator downIndicator() {
            return () -> Health.down()
                    .withDetail("reason", "Connection refused")
                    .build();
        }

        @Bean
        public StatusAggregator statusAggregator() {
            return new StatusAggregator();
        }

        @Bean
        public HealthEndpoint healthEndpoint(
                com.iris.framework.context.ApplicationContext context,
                StatusAggregator aggregator) {
            var factory = ((AnnotationConfigApplicationContext) context).getBeanFactory();
            var indicators = factory.getBeansOfType(HealthIndicator.class);
            Map<String, HealthIndicator> named = new LinkedHashMap<>();
            for (var entry : indicators.entrySet()) {
                named.put(entry.getKey(), entry.getValue());
            }
            return new HealthEndpoint(named, aggregator);
        }
    }

    @Configuration
    static class FailingHealthConfig {
        @Bean
        public HealthIndicator failingIndicator() {
            return () -> { throw new RuntimeException("Connection timeout"); };
        }

        @Bean
        public StatusAggregator statusAggregator() {
            return new StatusAggregator();
        }

        @Bean
        public HealthEndpoint healthEndpoint(
                com.iris.framework.context.ApplicationContext context,
                StatusAggregator aggregator) {
            var factory = ((AnnotationConfigApplicationContext) context).getBeanFactory();
            var indicators = factory.getBeansOfType(HealthIndicator.class);
            Map<String, HealthIndicator> named = new LinkedHashMap<>();
            for (var entry : indicators.entrySet()) {
                named.put(entry.getKey(), entry.getValue());
            }
            return new HealthEndpoint(named, aggregator);
        }
    }

    @Configuration
    static class InfoConfig {
        @Bean
        public InfoContributor appInfoContributor() {
            return info -> info.put("app", Map.of(
                    "name", "test-app",
                    "version", "1.0.0"
            ));
        }

        @Bean
        public InfoEndpoint infoEndpoint(
                com.iris.framework.context.ApplicationContext context) {
            var factory = ((AnnotationConfigApplicationContext) context).getBeanFactory();
            var contributors = factory.getBeansOfType(InfoContributor.class);
            return new InfoEndpoint(new java.util.ArrayList<>(contributors.values()));
        }
    }

    @Configuration
    static class MetricsConfig {
        @Bean
        public MeterRegistry meterRegistry() {
            return new SimpleMeterRegistry();
        }

        @Bean
        public MetricsEndpoint metricsEndpoint(MeterRegistry registry) {
            return new MetricsEndpoint(registry);
        }
    }

    @Configuration
    static class FullActuatorConfig {
        @Bean
        public ObjectMapper objectMapper() {
            return new ObjectMapper();
        }

        @Bean
        public StatusAggregator statusAggregator() {
            return new StatusAggregator();
        }

        @Bean
        public MeterRegistry meterRegistry() {
            return new SimpleMeterRegistry();
        }

        @Bean
        public HealthIndicator diskSpaceHealthIndicator() {
            return () -> Health.up().withDetail("free", "100GB").build();
        }

        @Bean
        public HealthEndpoint healthEndpoint(
                com.iris.framework.context.ApplicationContext context,
                StatusAggregator aggregator) {
            var factory = ((AnnotationConfigApplicationContext) context).getBeanFactory();
            var indicators = factory.getBeansOfType(HealthIndicator.class);
            Map<String, HealthIndicator> named = new LinkedHashMap<>();
            for (var entry : indicators.entrySet()) {
                String name = entry.getKey().replace("HealthIndicator", "");
                named.put(name, entry.getValue());
            }
            return new HealthEndpoint(named, aggregator);
        }

        @Bean
        public InfoContributor appInfoContributor() {
            return info -> info.put("app", Map.of("name", "test"));
        }

        @Bean
        public InfoEndpoint infoEndpoint(
                com.iris.framework.context.ApplicationContext context) {
            var factory = ((AnnotationConfigApplicationContext) context).getBeanFactory();
            var contributors = factory.getBeansOfType(InfoContributor.class);
            return new InfoEndpoint(new java.util.ArrayList<>(contributors.values()));
        }

        @Bean
        public MetricsEndpoint metricsEndpoint(MeterRegistry registry) {
            return new MetricsEndpoint(registry);
        }

        @Bean
        public ActuatorHandlerMapping actuatorHandlerMapping(
                com.iris.framework.context.ApplicationContext context,
                ObjectMapper objectMapper) {
            return new ActuatorHandlerMapping(context, objectMapper);
        }
    }
}
```

---

## Summary

| Concept | What you built | Real Spring Boot equivalent |
|---------|---------------|---------------------------|
| Endpoint annotation | `@Endpoint(id="...")` | `@Endpoint` |
| Read operation | `@ReadOperation` | `@ReadOperation` |
| Write operation | `@WriteOperation` | `@WriteOperation` |
| Path selector | `@Selector` | `@Selector` |
| Health status | `Status` (UP, DOWN, OUT_OF_SERVICE, UNKNOWN) | `Status` |
| Health result | `Health` with fluent Builder | `Health` |
| Health strategy | `HealthIndicator` (@FunctionalInterface) | `HealthIndicator` |
| Status aggregation | `StatusAggregator` (fixed severity ordering) | `SimpleStatusAggregator` |
| Health endpoint | `HealthEndpoint` (@Endpoint) | `HealthEndpoint` |
| Info contributor | `InfoContributor` (@FunctionalInterface) | `InfoContributor` |
| Info endpoint | `InfoEndpoint` (@Endpoint) | `InfoEndpoint` |
| Metrics endpoint | `MetricsEndpoint` (@Endpoint with @Selector) | `MetricsEndpoint` |
| Endpoint discovery | `EndpointDiscoverer` | `EndpointDiscoverer` hierarchy |
| HTTP mapping | `ActuatorHandlerMapping` (self-dispatching) | `WebMvcEndpointHandlerMapping` |
| Auto-configuration | `ActuatorAutoConfiguration` | Multiple auto-config classes |

**Files created:** 16 production files + 4 test files in the `iris-boot-actuator` module
**Files modified:** 2 (`DispatcherServlet.java`, `AnnotationConfigApplicationContext.java`)

---

## Final Note

This is the final chapter of the Iris Spring Lifecycle project. Over 24 chapters, we have built -- from scratch -- a working subset of the Spring Framework and Spring Boot:

1. **Chapters 1-7:** The core container -- bean definitions, dependency injection, lifecycle callbacks, annotation-based configuration, component scanning, the application context, and the environment/property system.

2. **Chapters 8-11:** The web layer -- application bootstrap, an embedded Tomcat server, the DispatcherServlet with handler mapping and handler adapters, and JSON serialization via `@ResponseBody`.

3. **Chapters 12-15:** The auto-configuration engine -- conditional beans (`@ConditionalOnClass`, `@ConditionalOnMissingBean`, `@ConditionalOnProperty`), the two-pass auto-configuration pipeline, `@ConfigurationProperties` binding, and the `@IrisBootApplication` entry point.

4. **Chapters 16-19:** Operational features -- a logging system, profiles and config data, failure analyzers for startup diagnostics, and graceful shutdown with `SmartLifecycle`.

5. **Chapters 20-24:** Observability -- Micrometer metrics (counters, gauges, timers), the Observation API with tracing, SSL bundles, connection details abstraction, and finally this chapter: actuator endpoints that tie observability data to HTTP.

Each feature was built by identifying a single **integration point** -- the precise location where the new code plugs into the existing system. This pattern mirrors how the real Spring Framework evolves: new capabilities are added by extending existing abstractions rather than rewriting them.

The complete codebase demonstrates that Spring Boot is not magic. It is a carefully layered system of interfaces, annotations, and discovery mechanisms. Every `@Autowired` injection, every `@ConditionalOnMissingBean` back-off, every `/actuator/health` response is the result of concrete code that you have now written and understood.
