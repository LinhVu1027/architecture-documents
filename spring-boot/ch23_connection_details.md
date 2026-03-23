# Chapter 23: ConnectionDetails

## Build Challenge

| Dimension | State |
|-----------|-------|
| **Current State** | Auto-configuration creates beans (DataSource, Redis template) by reading connection info directly from `@ConfigurationProperties` — hardwiring the source |
| **Limitation** | External sources (Testcontainers, Docker Compose, cloud service brokers) cannot override connection info without duplicating property names or intercepting the binding pipeline |
| **Objective** | Decouple *what* connection information a service needs from *where* it comes from, using a single `@ConditionalOnMissingBean`-based override pattern |

---

## 23.1 The Integration Point: Auto-Configuration + @ConditionalOnMissingBean

The integration point for ConnectionDetails is **inside auto-configuration classes** — specifically, the `@Bean` methods that create connection details beans. This is where the new abstraction plugs into the existing system:

**`DataSourceAutoConfiguration.java`** [NEW]

```java
@AutoConfiguration
@ConditionalOnProperty(name = "spring.datasource.url")
@EnableConfigurationProperties(DataSourceProperties.class)
public class DataSourceAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean(JdbcConnectionDetails.class)
    public JdbcConnectionDetails jdbcConnectionDetails(DataSourceProperties properties) {
        return new PropertiesJdbcConnectionDetails(properties);
    }
}
```

**Direction:** This auto-configuration connects **the ConnectionDetails abstraction** (new) with **the auto-configuration engine** (Feature 13) and **conditional beans** (Feature 12). The `@ConditionalOnMissingBean` is the pivot — it makes the properties-backed default yield to any externally-provided bean. From this integration point, we need to build:

1. The `ConnectionDetails` marker interface and `ConnectionDetailsFactory` SPI
2. Technology-specific interfaces (`JdbcConnectionDetails`, `RedisConnectionDetails`)
3. Properties classes (`DataSourceProperties`, `RedisProperties`)
4. Properties-backed adapter implementations
5. A second auto-configuration (`RedisAutoConfiguration`) demonstrating the same pattern

**Why `@ConditionalOnProperty` on the auto-config class?** Without it, the auto-config would create a `JdbcConnectionDetails` bean with all-null values when no datasource properties are configured. The property gate ensures the auto-config only activates when the user has actually configured a datasource.

---

## 23.2 Core Interfaces: The Abstraction Layer

### ConnectionDetails — The Marker

**New file:** `iris-boot-core/src/main/java/com/iris/boot/autoconfigure/service/connection/ConnectionDetails.java`

```java
package com.iris.boot.autoconfigure.service.connection;

public interface ConnectionDetails {
}
```

An empty marker interface. Each technology defines its own sub-interface with the relevant methods. The marker enables the `ConnectionDetailsFactory` to use a bounded type parameter.

### ConnectionDetailsFactory — The SPI

**New file:** `iris-boot-core/src/main/java/com/iris/boot/autoconfigure/service/connection/ConnectionDetailsFactory.java`

```java
package com.iris.boot.autoconfigure.service.connection;

@FunctionalInterface
public interface ConnectionDetailsFactory<S, D extends ConnectionDetails> {

    D getConnectionDetails(S source);
}
```

The SPI that bridges external sources to ConnectionDetails. The source type `S` determines which factories apply:
- `ContainerConnectionSource` for Testcontainers
- `DockerComposeConnectionSource` for Docker Compose
- Any custom type for custom integrations

---

## 23.3 Technology-Specific Interfaces and Properties

### JDBC Connection Details

**New file:** `iris-boot-core/src/main/java/com/iris/boot/autoconfigure/jdbc/JdbcConnectionDetails.java`

```java
package com.iris.boot.autoconfigure.jdbc;

import com.iris.boot.autoconfigure.service.connection.ConnectionDetails;

public interface JdbcConnectionDetails extends ConnectionDetails {

    String getJdbcUrl();

    String getUsername();

    String getPassword();
}
```

**New file:** `iris-boot-core/src/main/java/com/iris/boot/autoconfigure/jdbc/DataSourceProperties.java`

```java
package com.iris.boot.autoconfigure.jdbc;

import com.iris.boot.context.properties.ConfigurationProperties;

@ConfigurationProperties(prefix = "spring.datasource")
public class DataSourceProperties {

    private String url;
    private String username;
    private String password;
    private String driverClassName;

    // getters and setters...
    public String getUrl() { return url; }
    public void setUrl(String url) { this.url = url; }
    public String getUsername() { return username; }
    public void setUsername(String username) { this.username = username; }
    public String getPassword() { return password; }
    public void setPassword(String password) { this.password = password; }
    public String getDriverClassName() { return driverClassName; }
    public void setDriverClassName(String driverClassName) { this.driverClassName = driverClassName; }
}
```

### Redis Connection Details

**New file:** `iris-boot-core/src/main/java/com/iris/boot/autoconfigure/data/redis/RedisConnectionDetails.java`

```java
package com.iris.boot.autoconfigure.data.redis;

import com.iris.boot.autoconfigure.service.connection.ConnectionDetails;

public interface RedisConnectionDetails extends ConnectionDetails {

    String getHost();

    int getPort();

    String getPassword();
}
```

**New file:** `iris-boot-core/src/main/java/com/iris/boot/autoconfigure/data/redis/RedisProperties.java`

```java
package com.iris.boot.autoconfigure.data.redis;

import com.iris.boot.context.properties.ConfigurationProperties;

@ConfigurationProperties(prefix = "spring.data.redis")
public class RedisProperties {

    private String host = "localhost";
    private int port = 6379;
    private String password;

    // getters and setters...
    public String getHost() { return host; }
    public void setHost(String host) { this.host = host; }
    public int getPort() { return port; }
    public void setPort(int port) { this.port = port; }
    public String getPassword() { return password; }
    public void setPassword(String password) { this.password = password; }
}
```

---

## 23.4 Properties-Backed Implementations and Auto-Configuration

### The Adapter Classes

**New file:** `iris-boot-core/src/main/java/com/iris/boot/autoconfigure/jdbc/PropertiesJdbcConnectionDetails.java`

```java
package com.iris.boot.autoconfigure.jdbc;

class PropertiesJdbcConnectionDetails implements JdbcConnectionDetails {

    private final DataSourceProperties properties;

    PropertiesJdbcConnectionDetails(DataSourceProperties properties) {
        this.properties = properties;
    }

    @Override
    public String getJdbcUrl() { return this.properties.getUrl(); }

    @Override
    public String getUsername() { return this.properties.getUsername(); }

    @Override
    public String getPassword() { return this.properties.getPassword(); }
}
```

**New file:** `iris-boot-core/src/main/java/com/iris/boot/autoconfigure/data/redis/PropertiesRedisConnectionDetails.java`

```java
package com.iris.boot.autoconfigure.data.redis;

class PropertiesRedisConnectionDetails implements RedisConnectionDetails {

    private final RedisProperties properties;

    PropertiesRedisConnectionDetails(RedisProperties properties) {
        this.properties = properties;
    }

    @Override
    public String getHost() { return this.properties.getHost(); }

    @Override
    public int getPort() { return this.properties.getPort(); }

    @Override
    public String getPassword() { return this.properties.getPassword(); }
}
```

Note that both adapter classes are **package-private**. They're implementation details of the auto-configuration — consumers always work with the interface.

### The Auto-Configurations

**New file:** `iris-boot-core/src/main/java/com/iris/boot/autoconfigure/jdbc/DataSourceAutoConfiguration.java`

```java
package com.iris.boot.autoconfigure.jdbc;

import com.iris.boot.autoconfigure.AutoConfiguration;
import com.iris.boot.autoconfigure.condition.ConditionalOnMissingBean;
import com.iris.boot.autoconfigure.condition.ConditionalOnProperty;
import com.iris.boot.context.properties.EnableConfigurationProperties;
import com.iris.framework.context.annotation.Bean;

@AutoConfiguration
@ConditionalOnProperty(name = "spring.datasource.url")
@EnableConfigurationProperties(DataSourceProperties.class)
public class DataSourceAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean(JdbcConnectionDetails.class)
    public JdbcConnectionDetails jdbcConnectionDetails(DataSourceProperties properties) {
        return new PropertiesJdbcConnectionDetails(properties);
    }
}
```

**New file:** `iris-boot-core/src/main/java/com/iris/boot/autoconfigure/data/redis/RedisAutoConfiguration.java`

```java
package com.iris.boot.autoconfigure.data.redis;

import com.iris.boot.autoconfigure.AutoConfiguration;
import com.iris.boot.autoconfigure.condition.ConditionalOnMissingBean;
import com.iris.boot.autoconfigure.condition.ConditionalOnProperty;
import com.iris.boot.context.properties.EnableConfigurationProperties;
import com.iris.framework.context.annotation.Bean;

@AutoConfiguration
@ConditionalOnProperty(name = "spring.data.redis.host")
@EnableConfigurationProperties(RedisProperties.class)
public class RedisAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean(RedisConnectionDetails.class)
    public RedisConnectionDetails redisConnectionDetails(RedisProperties properties) {
        return new PropertiesRedisConnectionDetails(properties);
    }
}
```

### Registration in AutoConfiguration.imports

**Modifying:** `iris-boot-core/src/main/resources/META-INF/iris/com.iris.boot.autoconfigure.AutoConfiguration.imports`
**Change:** Add two new auto-configuration class names at the end

```diff
 com.iris.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration
+com.iris.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
+com.iris.boot.autoconfigure.data.redis.RedisAutoConfiguration
```

---

## 23.5 Try It Yourself

### Challenge 1: Implement the ConnectionDetails marker interface

<details><summary>Think about it: Why is ConnectionDetails an empty marker interface instead of having common methods like <code>getHost()</code>?</summary>

Because different services need fundamentally different connection information:
- JDBC needs a URL, username, password
- Redis needs host, port, password
- Kafka needs bootstrap servers, security protocol

There's no meaningful common API. The marker's purpose is to enable the `ConnectionDetailsFactory<S, D extends ConnectionDetails>` bounded type parameter, creating a type-safe SPI. It's the same pattern as `java.io.Serializable` — a type marker, not a behavioral contract.

</details>

### Challenge 2: Write a Testcontainers-style override

<details><summary>Write a <code>@Configuration</code> class that provides a <code>JdbcConnectionDetails</code> bean pointing to a hypothetical running container.</summary>

```java
@Configuration
public class TestcontainersJdbcConfig {

    @Bean
    public JdbcConnectionDetails jdbcConnectionDetails() {
        // In real Testcontainers, these values come from the running container
        return new JdbcConnectionDetails() {
            @Override public String getJdbcUrl() {
                return "jdbc:postgresql://localhost:55432/testdb";
            }
            @Override public String getUsername() { return "test"; }
            @Override public String getPassword() { return "test"; }
        };
    }
}
```

When this configuration is registered in the context (whether via `@ServiceConnection` annotation or manually), the `DataSourceAutoConfiguration`'s `@ConditionalOnMissingBean(JdbcConnectionDetails.class)` detects it and backs off. The container-provided details win.

</details>

### Challenge 3: What happens if you remove `@ConditionalOnMissingBean`?

<details><summary>Think through the failure mode.</summary>

Without `@ConditionalOnMissingBean`, the auto-configuration would *always* create a `PropertiesJdbcConnectionDetails` bean. If a Testcontainers config also provides a `JdbcConnectionDetails` bean, you'd get a `NoUniqueBeanDefinitionException` — two beans of the same type with no way to disambiguate.

`@ConditionalOnMissingBean` is what makes the entire override mechanism work. It's the "back-off" switch.

</details>

---

## 23.6 Tests

### Unit Tests

**`ConnectionDetailsTest.java`** — Tests the marker interface and factory SPI:

```java
@Test
@DisplayName("should implement ConnectionDetails marker interface")
void shouldImplementConnectionDetailsMarker() {
    record TestConnectionDetails(String host, int port) implements ConnectionDetails {}
    TestConnectionDetails details = new TestConnectionDetails("localhost", 5432);

    assertThat(details).isInstanceOf(ConnectionDetails.class);
}

@Test
@DisplayName("should create ConnectionDetails from factory")
void shouldCreateConnectionDetailsFromFactory() {
    ConnectionDetailsFactory<String, TestConnectionDetails> factory =
            source -> new TestConnectionDetails(source, 5432);

    TestConnectionDetails details = factory.getConnectionDetails("my-host");
    assertThat(details.host()).isEqualTo("my-host");
}
```

**`JdbcConnectionDetailsTest.java`** — Tests the JDBC adapter:

```java
@Test
@DisplayName("should create PropertiesJdbcConnectionDetails from DataSourceProperties")
void shouldCreateFromProperties() {
    DataSourceProperties props = new DataSourceProperties();
    props.setUrl("jdbc:postgresql://db-host:5432/mydb");
    props.setUsername("admin");
    props.setPassword("secret");

    JdbcConnectionDetails details = new DataSourceAutoConfiguration()
            .jdbcConnectionDetails(props);

    assertThat(details.getJdbcUrl()).isEqualTo("jdbc:postgresql://db-host:5432/mydb");
    assertThat(details.getUsername()).isEqualTo("admin");
    assertThat(details.getPassword()).isEqualTo("secret");
}
```

**`RedisConnectionDetailsTest.java`** — Tests the Redis adapter:

```java
@Test
@DisplayName("should use default values from RedisProperties")
void shouldUseDefaultValues() {
    RedisProperties props = new RedisProperties();

    RedisConnectionDetails details = new RedisAutoConfiguration()
            .redisConnectionDetails(props);

    assertThat(details.getHost()).isEqualTo("localhost");
    assertThat(details.getPort()).isEqualTo(6379);
    assertThat(details.getPassword()).isNull();
}
```

### Integration Tests

**`ConnectionDetailsIntegrationTest.java`** — Tests the full auto-configuration back-off pattern:

```java
@Test
@DisplayName("should create properties-backed JdbcConnectionDetails when no override exists")
void shouldCreatePropertiesJdbcConnectionDetails_WhenNoOverride() {
    var ctx = createContext(Map.of(
            "spring.datasource.url", "jdbc:mysql://prod:3306/app",
            "spring.datasource.username", "produser",
            "spring.datasource.password", "prodpass"
    ));

    JdbcConnectionDetails details = ctx.getBean(JdbcConnectionDetails.class);
    assertThat(details.getJdbcUrl()).isEqualTo("jdbc:mysql://prod:3306/app");

    ctx.close();
}

@Test
@DisplayName("should back off when Testcontainers provides JdbcConnectionDetails")
void shouldBackOff_WhenTestcontainersProvidesJdbcConnectionDetails() {
    var ctx = createContext(
            Map.of("spring.datasource.url", "jdbc:mysql://prod:3306/app",
                   "spring.datasource.username", "produser",
                   "spring.datasource.password", "prodpass"),
            TestcontainersJdbcConfig.class
    );

    JdbcConnectionDetails details = ctx.getBean(JdbcConnectionDetails.class);
    // Testcontainers' connection details win — properties-backed backs off
    assertThat(details.getJdbcUrl())
            .isEqualTo("jdbc:postgresql://localhost:55432/testdb");

    ctx.close();
}

@Test
@DisplayName("should support mixed: external JDBC + properties-backed Redis")
void shouldSupportMixedSources() {
    var ctx = createContext(
            Map.of("spring.datasource.url", "jdbc:mysql://prod:3306/app",
                   "spring.datasource.username", "produser",
                   "spring.datasource.password", "prodpass",
                   "spring.data.redis.host", "redis.prod.internal"),
            TestcontainersJdbcConfig.class
    );

    // JDBC: overridden by Testcontainers
    assertThat(ctx.getBean(JdbcConnectionDetails.class).getJdbcUrl())
            .isEqualTo("jdbc:postgresql://localhost:55432/testdb");

    // Redis: properties-backed (no override)
    assertThat(ctx.getBean(RedisConnectionDetails.class).getHost())
            .isEqualTo("redis.prod.internal");

    ctx.close();
}
```

---

## 23.7 Why This Works

`★ Insight ─────────────────────────────────────`
**The three-layer strategy pattern.** ConnectionDetails creates a clean separation of concerns using three layers: (1) the **interface** defines *what* information a service needs, (2) the **properties-backed default** adapts from `application.properties`, and (3) **external sources** override the default via `@ConditionalOnMissingBean`. The beauty is that layers 1 and 3 are completely independent of Spring's property system — a `JdbcConnectionDetails` from Testcontainers knows nothing about `DataSourceProperties`. This is the Dependency Inversion Principle applied to configuration: high-level auto-configuration depends on the `JdbcConnectionDetails` abstraction, not on the concrete properties source.
`─────────────────────────────────────────────────`

`★ Insight ─────────────────────────────────────`
**Why package-private adapters?** `PropertiesJdbcConnectionDetails` and `PropertiesRedisConnectionDetails` are package-private, not public. This is deliberate: consumers should *always* depend on the `JdbcConnectionDetails` or `RedisConnectionDetails` interface, never on the properties-backed implementation. The adapter is an implementation detail of the auto-configuration module. If it were public, developers might be tempted to reference it directly, defeating the entire decoupling purpose. This matches the real Spring Boot, where these classes are also package-private.
`─────────────────────────────────────────────────`

`★ Insight ─────────────────────────────────────`
**ConnectionDetailsFactory is the extensibility hinge.** While our simplified implementation demonstrates the pattern via manual `@Bean` overrides, the real Spring Boot uses `ConnectionDetailsFactory<S, D>` as a Service Provider Interface (SPI). When Testcontainers sees a `@ServiceConnection` annotation on a container bean, it creates a `ContainerConnectionSource` and passes it through all registered `ConnectionDetailsFactory` implementations (discovered via `META-INF/spring.factories`). The matching factory produces a `JdbcConnectionDetails` that reads from the running container. The same `@ConditionalOnMissingBean` on the auto-config causes the properties-backed default to yield — the override is completely transparent to the rest of the application.
`─────────────────────────────────────────────────`

---

## 23.8 What We Enhanced

| Component | Before (Ch22) | After (Ch23) | Why |
|-----------|--------------|--------------|-----|
| Auto-configuration imports | 4 entries (Jackson, SSL, Tomcat, WebMvc) | 6 entries (+DataSource, +Redis) | New auto-configs demonstrating the ConnectionDetails pattern |
| `AutoConfigurationImportSelectorTest` | Excluded 7 auto-configs in "exclude all" test | Excluded 9 auto-configs | Test must account for the 2 new auto-config classes in the imports file |

---

## 23.9 Connection to Real Spring Boot

| Iris Class | Real Spring Boot Class | File:Line (commit `5922311a95a`) |
|-----------|----------------------|----------------------------------|
| `ConnectionDetails` | `ConnectionDetails` | `core/spring-boot-autoconfigure/src/main/java/org/springframework/boot/autoconfigure/service/connection/ConnectionDetails.java:30` |
| `ConnectionDetailsFactory` | `ConnectionDetailsFactory` | `core/spring-boot-autoconfigure/src/main/java/org/springframework/boot/autoconfigure/service/connection/ConnectionDetailsFactory.java:33` |
| `JdbcConnectionDetails` | `JdbcConnectionDetails` | `module/spring-boot-jdbc/src/main/java/org/springframework/boot/jdbc/autoconfigure/JdbcConnectionDetails.java:28` |
| `DataSourceProperties` | `DataSourceProperties` | `module/spring-boot-jdbc/src/main/java/org/springframework/boot/jdbc/autoconfigure/DataSourceProperties.java:42` |
| `PropertiesJdbcConnectionDetails` | `PropertiesJdbcConnectionDetails` | `module/spring-boot-jdbc/src/main/java/org/springframework/boot/jdbc/autoconfigure/PropertiesJdbcConnectionDetails.java:26` |
| `DataSourceAutoConfiguration` | `DataSourceAutoConfiguration` | `module/spring-boot-jdbc/src/main/java/org/springframework/boot/jdbc/autoconfigure/DataSourceAutoConfiguration.java:48` |
| `RedisConnectionDetails` | `DataRedisConnectionDetails` | `module/spring-boot-data-redis/src/main/java/org/springframework/boot/data/redis/autoconfigure/DataRedisConnectionDetails.java:28` |
| `RedisProperties` | `DataRedisProperties` | `module/spring-boot-data-redis/src/main/java/org/springframework/boot/data/redis/autoconfigure/DataRedisProperties.java:35` |
| `PropertiesRedisConnectionDetails` | `PropertiesDataRedisConnectionDetails` | `module/spring-boot-data-redis/src/main/java/org/springframework/boot/data/redis/autoconfigure/PropertiesDataRedisConnectionDetails.java:30` |
| `RedisAutoConfiguration` | `DataRedisAutoConfiguration` | `module/spring-boot-data-redis/src/main/java/org/springframework/boot/data/redis/autoconfigure/DataRedisAutoConfiguration.java:45` |

### Key Simplifications

| What we simplified | What Spring Boot does | Why it matters |
|-------------------|----------------------|----------------|
| No `ConnectionDetailsFactories` resolver class | Uses `SpringFactoriesLoader` + `ResolvableType` to discover and match factories by generic type parameters | The full resolver is needed for automatic Testcontainers/Docker Compose discovery — beyond our scope |
| Manual `@Bean` overrides only | `@ServiceConnection` triggers automatic `ContainerConnectionDetailsFactory` discovery | We demonstrate the *pattern* (back-off via `@ConditionalOnMissingBean`); the SPI machinery adds automation |
| Standalone Redis only | `DataRedisConnectionDetails` supports Sentinel, Cluster, and MasterReplica modes with nested interfaces | Each mode is a separate concern; standalone demonstrates the ConnectionDetails pattern fully |
| No `JdbcConnectionDetailsBeanPostProcessor` | Re-applies connection values after `@ConfigurationProperties` binding for non-properties-backed details | Only needed when connection pool properties (Hikari, Tomcat DBCP) bind from `@ConfigurationProperties` first |
| Properties class has basic fields only | `DataSourceProperties` auto-detects embedded databases, resolves driver class from URL, supports JNDI | Fallback/detection logic is orthogonal to the ConnectionDetails abstraction pattern |

---

## 23.10 Complete Code

### Production Code

#### `ConnectionDetails.java` [NEW]

```java
package com.iris.boot.autoconfigure.service.connection;

/**
 * Marker interface for types that provide connection details for a remote service.
 *
 * <p>ConnectionDetails decouples <i>what</i> information is needed to connect to a
 * service (e.g., JDBC URL, username, password) from <i>where</i> that information
 * comes from (e.g., {@code application.properties}, Testcontainers, Docker Compose,
 * a cloud service broker).
 *
 * <p>The pattern works in three layers:
 * <ol>
 *   <li><b>Interface layer</b> — technology-specific sub-interfaces define the
 *       contract (e.g., {@code JdbcConnectionDetails.getJdbcUrl()})</li>
 *   <li><b>Default layer</b> — auto-configuration creates a properties-backed
 *       implementation as a {@code @Bean @ConditionalOnMissingBean}, adapting
 *       from {@code @ConfigurationProperties}</li>
 *   <li><b>Override layer</b> — external sources register their own
 *       {@code ConnectionDetails} bean, causing the properties-backed default
 *       to back off via {@code @ConditionalOnMissingBean}</li>
 * </ol>
 *
 * <p>This is the simplified equivalent of Spring Boot's {@code ConnectionDetails}.
 * The real interface additionally extends nothing (it's also a pure marker), but
 * implementations can optionally implement {@code OriginProvider} to indicate
 * where the connection information originated.
 *
 * @since Feature 23
 * @see ConnectionDetailsFactory
 * @see org.springframework.boot.autoconfigure.service.connection.ConnectionDetails
 */
public interface ConnectionDetails {
}
```

#### `ConnectionDetailsFactory.java` [NEW]

```java
package com.iris.boot.autoconfigure.service.connection;

/**
 * SPI for creating {@link ConnectionDetails} from an external source.
 *
 * <p>A {@code ConnectionDetailsFactory} bridges an external service source
 * (e.g., a running Testcontainer, a Docker Compose service, or a cloud
 * service broker) to the {@code ConnectionDetails} interface that
 * auto-configuration classes consume.
 *
 * <p>The type parameters create a type-safe contract:
 * <ul>
 *   <li>{@code S} — the source type (e.g., a container or service descriptor)</li>
 *   <li>{@code D} — the {@code ConnectionDetails} type this factory produces</li>
 * </ul>
 *
 * <p>In the real Spring Boot, implementations are registered in
 * {@code META-INF/spring.factories} and discovered by
 * {@code ConnectionDetailsFactories}, which uses {@code ResolvableType}
 * to match each factory's generic parameters against the source being
 * processed. Matching factories are invoked to produce connection details,
 * which are then registered as beans in the application context.
 *
 * <p>We include this interface to demonstrate the SPI design, even though
 * our simplified version doesn't implement the full Testcontainers/Docker
 * Compose integration. The key insight is the <i>pattern</i>: any source
 * that can create a {@code ConnectionDetails} instance can participate
 * in the override mechanism.
 *
 * <h3>Example: A hypothetical Testcontainers factory</h3>
 * <pre>{@code
 * public class PostgresContainerConnectionDetailsFactory
 *         implements ConnectionDetailsFactory<PostgreSQLContainer<?>, JdbcConnectionDetails> {
 *
 *     @Override
 *     public JdbcConnectionDetails getConnectionDetails(PostgreSQLContainer<?> container) {
 *         return new JdbcConnectionDetails() {
 *             public String getJdbcUrl() { return container.getJdbcUrl(); }
 *             public String getUsername() { return container.getUsername(); }
 *             public String getPassword() { return container.getPassword(); }
 *         };
 *     }
 * }
 * }</pre>
 *
 * @param <S> the type of source this factory can process
 * @param <D> the type of {@link ConnectionDetails} this factory produces
 * @since Feature 23
 * @see ConnectionDetails
 * @see org.springframework.boot.autoconfigure.service.connection.ConnectionDetailsFactory
 */
@FunctionalInterface
public interface ConnectionDetailsFactory<S, D extends ConnectionDetails> {

    /**
     * Get the {@link ConnectionDetails} from the given {@code source}.
     *
     * @param source the source to extract connection information from
     * @return the connection details, or {@code null} if this factory cannot
     *         handle the given source
     */
    D getConnectionDetails(S source);
}
```

#### `JdbcConnectionDetails.java` [NEW]

```java
package com.iris.boot.autoconfigure.jdbc;

import com.iris.boot.autoconfigure.service.connection.ConnectionDetails;

/**
 * {@link ConnectionDetails} for a JDBC connection.
 *
 * <p>Provides the information needed to connect to a relational database:
 * the JDBC URL, username, and password. Any auto-configuration that creates
 * a {@code DataSource} should consume this interface rather than reading
 * properties directly.
 *
 * <p>This decoupling means that the connection information can come from:
 * <ul>
 *   <li>{@code application.properties} (the default, via
 *       {@link PropertiesJdbcConnectionDetails})</li>
 *   <li>A running Testcontainer (registered by a
 *       {@code ConnectionDetailsFactory})</li>
 *   <li>A Docker Compose service</li>
 *   <li>A cloud service broker (e.g., Cloud Foundry, Kubernetes service bindings)</li>
 * </ul>
 *
 * <p>In the real Spring Boot, this interface also provides:
 * <ul>
 *   <li>{@code getDriverClassName()} — with a default that auto-detects from the URL</li>
 *   <li>{@code getXaDataSourceClassName()} — for XA/distributed transactions</li>
 * </ul>
 * We simplify to the three essential methods.
 *
 * @since Feature 23
 * @see ConnectionDetails
 * @see PropertiesJdbcConnectionDetails
 * @see DataSourceAutoConfiguration
 * @see org.springframework.boot.jdbc.autoconfigure.JdbcConnectionDetails
 */
public interface JdbcConnectionDetails extends ConnectionDetails {

    String getJdbcUrl();

    String getUsername();

    String getPassword();
}
```

#### `DataSourceProperties.java` [NEW]

```java
package com.iris.boot.autoconfigure.jdbc;

import com.iris.boot.context.properties.ConfigurationProperties;

/**
 * {@link ConfigurationProperties @ConfigurationProperties} for configuring a
 * JDBC DataSource connection.
 *
 * <p>Binds properties with the {@code spring.datasource} prefix from
 * {@code application.properties}:
 * <pre>
 * spring.datasource.url=jdbc:postgresql://localhost:5432/mydb
 * spring.datasource.username=admin
 * spring.datasource.password=secret
 * spring.datasource.driver-class-name=org.postgresql.Driver
 * </pre>
 *
 * <p>In the real Spring Boot, {@code DataSourceProperties} (now in the
 * {@code spring-boot-jdbc} module) is much more sophisticated:
 * <ul>
 *   <li>Auto-detects embedded databases (H2, HSQL, Derby)</li>
 *   <li>Has {@code determineUrl()}/{@code determineUsername()}/{@code determinePassword()}
 *       methods with fallback logic</li>
 *   <li>Supports JNDI lookup via {@code jndi-name}</li>
 *   <li>Supports connection pool type selection</li>
 * </ul>
 * We simplify to the four core properties.
 *
 * @since Feature 23
 * @see JdbcConnectionDetails
 * @see DataSourceAutoConfiguration
 * @see org.springframework.boot.jdbc.autoconfigure.DataSourceProperties
 */
@ConfigurationProperties(prefix = "spring.datasource")
public class DataSourceProperties {

    private String url;
    private String username;
    private String password;
    private String driverClassName;

    public String getUrl() { return url; }
    public void setUrl(String url) { this.url = url; }

    public String getUsername() { return username; }
    public void setUsername(String username) { this.username = username; }

    public String getPassword() { return password; }
    public void setPassword(String password) { this.password = password; }

    public String getDriverClassName() { return driverClassName; }
    public void setDriverClassName(String driverClassName) { this.driverClassName = driverClassName; }
}
```

#### `PropertiesJdbcConnectionDetails.java` [NEW]

```java
package com.iris.boot.autoconfigure.jdbc;

/**
 * {@link JdbcConnectionDetails} backed by {@link DataSourceProperties}.
 *
 * <p>This is the default implementation that adapts property values from
 * {@code application.properties} (via {@code @ConfigurationProperties}) into
 * the {@code JdbcConnectionDetails} interface. It is created by
 * {@link DataSourceAutoConfiguration} as a {@code @ConditionalOnMissingBean}
 * — meaning it backs off when an external source (e.g., Testcontainers)
 * provides its own {@code JdbcConnectionDetails} bean.
 *
 * <p>In the real Spring Boot, this class is package-private in the
 * {@code spring-boot-jdbc} module and delegates to
 * {@code DataSourceProperties.determineUrl()}, etc. for auto-detection
 * logic. We delegate directly to the properties' getters.
 *
 * @since Feature 23
 * @see JdbcConnectionDetails
 * @see DataSourceProperties
 * @see org.springframework.boot.jdbc.autoconfigure.PropertiesJdbcConnectionDetails
 */
class PropertiesJdbcConnectionDetails implements JdbcConnectionDetails {

    private final DataSourceProperties properties;

    PropertiesJdbcConnectionDetails(DataSourceProperties properties) {
        this.properties = properties;
    }

    @Override
    public String getJdbcUrl() { return this.properties.getUrl(); }

    @Override
    public String getUsername() { return this.properties.getUsername(); }

    @Override
    public String getPassword() { return this.properties.getPassword(); }
}
```

#### `DataSourceAutoConfiguration.java` [NEW]

```java
package com.iris.boot.autoconfigure.jdbc;

import com.iris.boot.autoconfigure.AutoConfiguration;
import com.iris.boot.autoconfigure.condition.ConditionalOnMissingBean;
import com.iris.boot.autoconfigure.condition.ConditionalOnProperty;
import com.iris.boot.context.properties.EnableConfigurationProperties;
import com.iris.framework.context.annotation.Bean;

/**
 * Auto-configuration for JDBC {@link JdbcConnectionDetails}.
 *
 * <p>This is the canonical demonstration of the ConnectionDetails pattern:
 * <ol>
 *   <li>Enable {@link DataSourceProperties} to bind from
 *       {@code spring.datasource.*}</li>
 *   <li>Create a {@link PropertiesJdbcConnectionDetails} bean (the
 *       properties-backed default) — but only if no other
 *       {@code JdbcConnectionDetails} bean exists</li>
 * </ol>
 *
 * <p>The {@code @ConditionalOnMissingBean(JdbcConnectionDetails.class)} is
 * the key: if an external source (Testcontainers, Docker Compose, or any
 * custom bean) has already registered a {@code JdbcConnectionDetails} bean,
 * this auto-configuration backs off and the external source's connection
 * details are used instead.
 *
 * <p>In the real Spring Boot, the {@code DataSourceAutoConfiguration} is
 * split across multiple classes:
 * <ul>
 *   <li>{@code DataSourceAutoConfiguration} — outer class with
 *       embedded/pooled inner configurations</li>
 *   <li>{@code DataSourceConfiguration} — creates pool-specific DataSource
 *       beans (Hikari, Tomcat, DBCP2) using {@code JdbcConnectionDetails}</li>
 *   <li>{@code JdbcConnectionDetailsBeanPostProcessor} — re-applies
 *       connection values for non-properties-backed details</li>
 * </ul>
 * We simplify to a single class that creates the connection details bean.
 *
 * <p><b>Note:</b> This auto-configuration does NOT actually create a
 * {@code DataSource} bean — that would require a connection pool library
 * on the classpath. Instead, it demonstrates the ConnectionDetails pattern:
 * how connection information is abstracted and how the default backs off.
 * A real application would have additional auto-configuration that consumes
 * the {@code JdbcConnectionDetails} to create the actual {@code DataSource}.
 *
 * @since Feature 23
 * @see JdbcConnectionDetails
 * @see PropertiesJdbcConnectionDetails
 * @see DataSourceProperties
 * @see org.springframework.boot.jdbc.autoconfigure.DataSourceAutoConfiguration
 */
@AutoConfiguration
@ConditionalOnProperty(name = "spring.datasource.url")
@EnableConfigurationProperties(DataSourceProperties.class)
public class DataSourceAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean(JdbcConnectionDetails.class)
    public JdbcConnectionDetails jdbcConnectionDetails(DataSourceProperties properties) {
        return new PropertiesJdbcConnectionDetails(properties);
    }
}
```

#### `RedisConnectionDetails.java` [NEW]

```java
package com.iris.boot.autoconfigure.data.redis;

import com.iris.boot.autoconfigure.service.connection.ConnectionDetails;

/**
 * {@link ConnectionDetails} for a Redis connection.
 *
 * <p>Provides the information needed to connect to a Redis server: host,
 * port, and password. Like {@link com.iris.boot.autoconfigure.jdbc.JdbcConnectionDetails
 * JdbcConnectionDetails}, the actual source of this information is abstracted —
 * it may come from properties, a Testcontainer, Docker Compose, or any other source.
 *
 * <p>In the real Spring Boot, {@code DataRedisConnectionDetails} is significantly
 * more complex, supporting:
 * <ul>
 *   <li>Standalone mode (host/port)</li>
 *   <li>Sentinel mode (master name + sentinel nodes)</li>
 *   <li>Cluster mode (cluster nodes)</li>
 *   <li>Master-Replica mode</li>
 *   <li>SSL bundle reference</li>
 *   <li>Username (ACL-based auth in Redis 6+)</li>
 * </ul>
 * We simplify to standalone mode with host, port, and password.
 *
 * @since Feature 23
 * @see ConnectionDetails
 * @see PropertiesRedisConnectionDetails
 * @see RedisAutoConfiguration
 * @see org.springframework.boot.data.redis.autoconfigure.DataRedisConnectionDetails
 */
public interface RedisConnectionDetails extends ConnectionDetails {

    String getHost();

    int getPort();

    String getPassword();
}
```

#### `RedisProperties.java` [NEW]

```java
package com.iris.boot.autoconfigure.data.redis;

import com.iris.boot.context.properties.ConfigurationProperties;

/**
 * {@link ConfigurationProperties @ConfigurationProperties} for configuring
 * a Redis connection.
 *
 * <p>Binds properties with the {@code spring.data.redis} prefix from
 * {@code application.properties}:
 * <pre>
 * spring.data.redis.host=redis.example.com
 * spring.data.redis.port=6380
 * spring.data.redis.password=secret
 * </pre>
 *
 * <p>In the real Spring Boot, {@code DataRedisProperties} (previously
 * {@code RedisProperties}) supports sentinel, cluster, SSL, connection
 * pooling (Lettuce/Jedis), timeouts, and more. We simplify to the three
 * essential standalone properties.
 *
 * @since Feature 23
 * @see RedisConnectionDetails
 * @see RedisAutoConfiguration
 * @see org.springframework.boot.data.redis.autoconfigure.DataRedisProperties
 */
@ConfigurationProperties(prefix = "spring.data.redis")
public class RedisProperties {

    private String host = "localhost";
    private int port = 6379;
    private String password;

    public String getHost() { return host; }
    public void setHost(String host) { this.host = host; }

    public int getPort() { return port; }
    public void setPort(int port) { this.port = port; }

    public String getPassword() { return password; }
    public void setPassword(String password) { this.password = password; }
}
```

#### `PropertiesRedisConnectionDetails.java` [NEW]

```java
package com.iris.boot.autoconfigure.data.redis;

/**
 * {@link RedisConnectionDetails} backed by {@link RedisProperties}.
 *
 * <p>This is the default implementation that adapts property values from
 * {@code application.properties} (via {@code @ConfigurationProperties}) into
 * the {@code RedisConnectionDetails} interface. Created by
 * {@link RedisAutoConfiguration} as a {@code @ConditionalOnMissingBean}.
 *
 * <p>In the real Spring Boot, {@code PropertiesDataRedisConnectionDetails}
 * handles URL parsing, sentinel/cluster/master-replica modes, and SSL
 * bundles. We delegate directly to the properties' getters for the
 * standalone case.
 *
 * @since Feature 23
 * @see RedisConnectionDetails
 * @see RedisProperties
 * @see org.springframework.boot.data.redis.autoconfigure.PropertiesDataRedisConnectionDetails
 */
class PropertiesRedisConnectionDetails implements RedisConnectionDetails {

    private final RedisProperties properties;

    PropertiesRedisConnectionDetails(RedisProperties properties) {
        this.properties = properties;
    }

    @Override
    public String getHost() { return this.properties.getHost(); }

    @Override
    public int getPort() { return this.properties.getPort(); }

    @Override
    public String getPassword() { return this.properties.getPassword(); }
}
```

#### `RedisAutoConfiguration.java` [NEW]

```java
package com.iris.boot.autoconfigure.data.redis;

import com.iris.boot.autoconfigure.AutoConfiguration;
import com.iris.boot.autoconfigure.condition.ConditionalOnMissingBean;
import com.iris.boot.autoconfigure.condition.ConditionalOnProperty;
import com.iris.boot.context.properties.EnableConfigurationProperties;
import com.iris.framework.context.annotation.Bean;

/**
 * Auto-configuration for Redis {@link RedisConnectionDetails}.
 *
 * <p>Follows the same ConnectionDetails pattern as
 * {@link com.iris.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
 * DataSourceAutoConfiguration}:
 * <ol>
 *   <li>Enable {@link RedisProperties} to bind from
 *       {@code spring.data.redis.*}</li>
 *   <li>Create a {@link PropertiesRedisConnectionDetails} bean —
 *       but only if no other {@code RedisConnectionDetails} bean exists</li>
 * </ol>
 *
 * <p>In the real Spring Boot, {@code DataRedisAutoConfiguration} also creates
 * a {@code RedisConnectionFactory} (Lettuce or Jedis) using the connection
 * details. We focus on the ConnectionDetails layer to demonstrate the pattern.
 *
 * @since Feature 23
 * @see RedisConnectionDetails
 * @see PropertiesRedisConnectionDetails
 * @see RedisProperties
 * @see org.springframework.boot.data.redis.autoconfigure.DataRedisAutoConfiguration
 */
@AutoConfiguration
@ConditionalOnProperty(name = "spring.data.redis.host")
@EnableConfigurationProperties(RedisProperties.class)
public class RedisAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean(RedisConnectionDetails.class)
    public RedisConnectionDetails redisConnectionDetails(RedisProperties properties) {
        return new PropertiesRedisConnectionDetails(properties);
    }
}
```

#### `AutoConfiguration.imports` [MODIFIED]

```
# Iris Boot Auto-Configuration Candidates
#
# Listed in dependency order:
# 1. JacksonAutoConfiguration — provides ObjectMapper (no ordering constraints)
# 2. SslAutoConfiguration — provides SslBundles registry (before Tomcat)
# 3. TomcatAutoConfiguration — provides ServletWebServerFactory (after SSL)
# 4. WebMvcAutoConfiguration — provides HttpMessageConverter (after Tomcat)
# 5. DataSourceAutoConfiguration — provides JdbcConnectionDetails
# 6. RedisAutoConfiguration — provides RedisConnectionDetails
com.iris.boot.autoconfigure.jackson.JacksonAutoConfiguration
com.iris.boot.autoconfigure.ssl.SslAutoConfiguration
com.iris.boot.autoconfigure.web.servlet.TomcatAutoConfiguration
com.iris.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration
com.iris.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
com.iris.boot.autoconfigure.data.redis.RedisAutoConfiguration
```

---

### Test Code

#### `ConnectionDetailsTest.java` [NEW]

```java
package com.iris.boot.autoconfigure.service.connection;

import static org.assertj.core.api.Assertions.assertThat;

import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;

class ConnectionDetailsTest {

    record TestConnectionDetails(String host, int port) implements ConnectionDetails {}
    record AnotherConnectionDetails(String url) implements ConnectionDetails {}

    @Test
    @DisplayName("should implement ConnectionDetails marker interface")
    void shouldImplementConnectionDetailsMarker() {
        TestConnectionDetails details = new TestConnectionDetails("localhost", 5432);
        assertThat(details).isInstanceOf(ConnectionDetails.class);
        assertThat(details.host()).isEqualTo("localhost");
        assertThat(details.port()).isEqualTo(5432);
    }

    @Test
    @DisplayName("should distinguish different ConnectionDetails types")
    void shouldDistinguishDifferentTypes() {
        ConnectionDetails jdbc = new TestConnectionDetails("db-host", 5432);
        ConnectionDetails redis = new AnotherConnectionDetails("redis://cache:6379");
        assertThat(jdbc).isNotInstanceOf(AnotherConnectionDetails.class);
        assertThat(redis).isNotInstanceOf(TestConnectionDetails.class);
    }

    @Test
    @DisplayName("should create ConnectionDetails from factory")
    void shouldCreateConnectionDetailsFromFactory() {
        ConnectionDetailsFactory<String, TestConnectionDetails> factory =
                source -> new TestConnectionDetails(source, 5432);
        TestConnectionDetails details = factory.getConnectionDetails("my-host");
        assertThat(details).isNotNull();
        assertThat(details.host()).isEqualTo("my-host");
    }

    @Test
    @DisplayName("should return null from factory when source cannot be handled")
    void shouldReturnNull_WhenSourceCannotBeHandled() {
        ConnectionDetailsFactory<String, TestConnectionDetails> factory =
                source -> source.startsWith("valid:") ?
                        new TestConnectionDetails(source, 5432) : null;
        assertThat(factory.getConnectionDetails("valid:host")).isNotNull();
        assertThat(factory.getConnectionDetails("invalid")).isNull();
    }

    @Test
    @DisplayName("should use factory as functional interface with lambda")
    void shouldUseLambdaSyntax() {
        ConnectionDetailsFactory<Integer, TestConnectionDetails> portFactory =
                port -> new TestConnectionDetails("localhost", port);
        TestConnectionDetails details = portFactory.getConnectionDetails(9999);
        assertThat(details.port()).isEqualTo(9999);
    }
}
```

#### `JdbcConnectionDetailsTest.java` [NEW]

```java
package com.iris.boot.autoconfigure.jdbc;

import static org.assertj.core.api.Assertions.assertThat;

import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;

import com.iris.boot.autoconfigure.service.connection.ConnectionDetails;

class JdbcConnectionDetailsTest {

    @Test
    @DisplayName("should extend ConnectionDetails marker")
    void shouldExtendConnectionDetails() {
        JdbcConnectionDetails details = createDetails("jdbc:h2:mem:test", "sa", "");
        assertThat(details).isInstanceOf(ConnectionDetails.class);
    }

    @Test
    @DisplayName("should create PropertiesJdbcConnectionDetails from DataSourceProperties")
    void shouldCreateFromProperties() {
        DataSourceProperties props = new DataSourceProperties();
        props.setUrl("jdbc:postgresql://db-host:5432/mydb");
        props.setUsername("admin");
        props.setPassword("secret");

        JdbcConnectionDetails details = new DataSourceAutoConfiguration()
                .jdbcConnectionDetails(props);
        assertThat(details.getJdbcUrl()).isEqualTo("jdbc:postgresql://db-host:5432/mydb");
        assertThat(details.getUsername()).isEqualTo("admin");
        assertThat(details.getPassword()).isEqualTo("secret");
    }

    @Test
    @DisplayName("should reflect property changes (adapter pattern)")
    void shouldReflectPropertyChanges() {
        DataSourceProperties props = new DataSourceProperties();
        props.setUrl("jdbc:h2:mem:initial");

        JdbcConnectionDetails details = new DataSourceAutoConfiguration()
                .jdbcConnectionDetails(props);
        assertThat(details.getJdbcUrl()).isEqualTo("jdbc:h2:mem:initial");

        props.setUrl("jdbc:h2:mem:updated");
        assertThat(details.getJdbcUrl()).isEqualTo("jdbc:h2:mem:updated");
    }

    @Test
    @DisplayName("should handle null properties gracefully")
    void shouldHandleNullProperties() {
        DataSourceProperties props = new DataSourceProperties();
        JdbcConnectionDetails details = new DataSourceAutoConfiguration()
                .jdbcConnectionDetails(props);

        assertThat(details.getJdbcUrl()).isNull();
        assertThat(details.getUsername()).isNull();
        assertThat(details.getPassword()).isNull();
    }

    @Test
    @DisplayName("should bind DataSourceProperties fields")
    void shouldBindDataSourceProperties() {
        DataSourceProperties props = new DataSourceProperties();
        props.setUrl("jdbc:mysql://localhost:3306/shop");
        props.setUsername("root");
        props.setPassword("rootpw");
        props.setDriverClassName("com.mysql.cj.jdbc.Driver");

        assertThat(props.getUrl()).isEqualTo("jdbc:mysql://localhost:3306/shop");
        assertThat(props.getDriverClassName()).isEqualTo("com.mysql.cj.jdbc.Driver");
    }

    private JdbcConnectionDetails createDetails(String url, String user, String password) {
        return new JdbcConnectionDetails() {
            @Override public String getJdbcUrl() { return url; }
            @Override public String getUsername() { return user; }
            @Override public String getPassword() { return password; }
        };
    }
}
```

#### `RedisConnectionDetailsTest.java` [NEW]

```java
package com.iris.boot.autoconfigure.data.redis;

import static org.assertj.core.api.Assertions.assertThat;

import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;

import com.iris.boot.autoconfigure.service.connection.ConnectionDetails;

class RedisConnectionDetailsTest {

    @Test
    @DisplayName("should extend ConnectionDetails marker")
    void shouldExtendConnectionDetails() {
        RedisConnectionDetails details = createDetails("localhost", 6379, null);
        assertThat(details).isInstanceOf(ConnectionDetails.class);
    }

    @Test
    @DisplayName("should create PropertiesRedisConnectionDetails from RedisProperties")
    void shouldCreateFromProperties() {
        RedisProperties props = new RedisProperties();
        props.setHost("redis.example.com");
        props.setPort(6380);
        props.setPassword("redis-secret");

        RedisConnectionDetails details = new RedisAutoConfiguration()
                .redisConnectionDetails(props);
        assertThat(details.getHost()).isEqualTo("redis.example.com");
        assertThat(details.getPort()).isEqualTo(6380);
        assertThat(details.getPassword()).isEqualTo("redis-secret");
    }

    @Test
    @DisplayName("should use default values from RedisProperties")
    void shouldUseDefaultValues() {
        RedisProperties props = new RedisProperties();
        RedisConnectionDetails details = new RedisAutoConfiguration()
                .redisConnectionDetails(props);

        assertThat(details.getHost()).isEqualTo("localhost");
        assertThat(details.getPort()).isEqualTo(6379);
        assertThat(details.getPassword()).isNull();
    }

    @Test
    @DisplayName("should reflect property changes (adapter pattern)")
    void shouldReflectPropertyChanges() {
        RedisProperties props = new RedisProperties();
        props.setHost("initial-host");

        RedisConnectionDetails details = new RedisAutoConfiguration()
                .redisConnectionDetails(props);
        assertThat(details.getHost()).isEqualTo("initial-host");

        props.setHost("updated-host");
        assertThat(details.getHost()).isEqualTo("updated-host");
    }

    @Test
    @DisplayName("should bind RedisProperties fields")
    void shouldBindRedisProperties() {
        RedisProperties props = new RedisProperties();
        props.setHost("cache-01.internal");
        props.setPort(6380);
        props.setPassword("s3cret");

        assertThat(props.getHost()).isEqualTo("cache-01.internal");
        assertThat(props.getPort()).isEqualTo(6380);
        assertThat(props.getPassword()).isEqualTo("s3cret");
    }

    private RedisConnectionDetails createDetails(String host, int port, String password) {
        return new RedisConnectionDetails() {
            @Override public String getHost() { return host; }
            @Override public int getPort() { return port; }
            @Override public String getPassword() { return password; }
        };
    }
}
```

#### `ConnectionDetailsIntegrationTest.java` [NEW]

```java
package com.iris.boot.autoconfigure.service.connection.integration;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.Map;

import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;

import com.iris.boot.autoconfigure.AutoConfigurationImportSelector;
import com.iris.boot.autoconfigure.jdbc.DataSourceProperties;
import com.iris.boot.autoconfigure.jdbc.JdbcConnectionDetails;
import com.iris.boot.autoconfigure.data.redis.RedisConnectionDetails;
import com.iris.boot.autoconfigure.data.redis.RedisProperties;
import com.iris.boot.context.properties.ConfigurationPropertiesBindingPostProcessor;
import com.iris.framework.context.AnnotationConfigApplicationContext;
import com.iris.framework.context.annotation.Bean;
import com.iris.framework.context.annotation.Configuration;
import com.iris.framework.core.env.MapPropertySource;
import com.iris.framework.core.env.StandardEnvironment;

class ConnectionDetailsIntegrationTest {

    @Configuration
    static class TestcontainersJdbcConfig {
        @Bean
        public JdbcConnectionDetails jdbcConnectionDetails() {
            return new JdbcConnectionDetails() {
                @Override public String getJdbcUrl() {
                    return "jdbc:postgresql://localhost:55432/testdb";
                }
                @Override public String getUsername() { return "test"; }
                @Override public String getPassword() { return "test"; }
            };
        }
    }

    @Configuration
    static class DockerComposeRedisConfig {
        @Bean
        public RedisConnectionDetails redisConnectionDetails() {
            return new RedisConnectionDetails() {
                @Override public String getHost() { return "localhost"; }
                @Override public int getPort() { return 63790; }
                @Override public String getPassword() { return "docker-secret"; }
            };
        }
    }

    @SuppressWarnings("unchecked")
    private AnnotationConfigApplicationContext createContext(
            Map<String, ?> properties, Class<?>... userConfigs) {
        StandardEnvironment env = new StandardEnvironment();
        env.getPropertySources().addLast(new MapPropertySource("test",
                (Map<String, Object>) (Map<?, ?>) properties));

        var ctx = new AnnotationConfigApplicationContext();
        ctx.setEnvironment(env);
        ctx.addInternalBeanPostProcessor(
                new ConfigurationPropertiesBindingPostProcessor(env));
        for (Class<?> cfg : userConfigs) {
            ctx.register(cfg);
        }
        ctx.setDeferredConfigurationLoader(new AutoConfigurationImportSelector());
        ctx.refresh();
        return ctx;
    }

    @Test
    @DisplayName("should create properties-backed JdbcConnectionDetails when no override exists")
    void shouldCreatePropertiesJdbcConnectionDetails_WhenNoOverride() {
        var ctx = createContext(Map.of(
                "spring.datasource.url", "jdbc:mysql://prod:3306/app",
                "spring.datasource.username", "produser",
                "spring.datasource.password", "prodpass"
        ));

        JdbcConnectionDetails details = ctx.getBean(JdbcConnectionDetails.class);
        assertThat(details.getJdbcUrl()).isEqualTo("jdbc:mysql://prod:3306/app");
        assertThat(details.getUsername()).isEqualTo("produser");
        assertThat(details.getPassword()).isEqualTo("prodpass");

        ctx.close();
    }

    @Test
    @DisplayName("should back off when Testcontainers provides JdbcConnectionDetails")
    void shouldBackOff_WhenTestcontainersProvidesJdbcConnectionDetails() {
        var ctx = createContext(
                Map.of("spring.datasource.url", "jdbc:mysql://prod:3306/app",
                       "spring.datasource.username", "produser",
                       "spring.datasource.password", "prodpass"),
                TestcontainersJdbcConfig.class
        );

        JdbcConnectionDetails details = ctx.getBean(JdbcConnectionDetails.class);
        assertThat(details.getJdbcUrl())
                .isEqualTo("jdbc:postgresql://localhost:55432/testdb");
        assertThat(details.getUsername()).isEqualTo("test");

        ctx.close();
    }

    @Test
    @DisplayName("should create properties-backed RedisConnectionDetails when no override exists")
    void shouldCreatePropertiesRedisConnectionDetails_WhenNoOverride() {
        var ctx = createContext(Map.of(
                "spring.data.redis.host", "redis.prod.internal",
                "spring.data.redis.port", "6380",
                "spring.data.redis.password", "prod-redis-pw"
        ));

        RedisConnectionDetails details = ctx.getBean(RedisConnectionDetails.class);
        assertThat(details.getHost()).isEqualTo("redis.prod.internal");
        assertThat(details.getPort()).isEqualTo(6380);
        assertThat(details.getPassword()).isEqualTo("prod-redis-pw");

        ctx.close();
    }

    @Test
    @DisplayName("should back off when Docker Compose provides RedisConnectionDetails")
    void shouldBackOff_WhenDockerComposeProvidesRedisConnectionDetails() {
        var ctx = createContext(
                Map.of("spring.data.redis.host", "redis.prod.internal",
                       "spring.data.redis.port", "6380"),
                DockerComposeRedisConfig.class
        );

        RedisConnectionDetails details = ctx.getBean(RedisConnectionDetails.class);
        assertThat(details.getHost()).isEqualTo("localhost");
        assertThat(details.getPort()).isEqualTo(63790);
        assertThat(details.getPassword()).isEqualTo("docker-secret");

        ctx.close();
    }

    @Test
    @DisplayName("should support mixed: external JDBC + properties-backed Redis")
    void shouldSupportMixedSources() {
        var ctx = createContext(
                Map.of("spring.datasource.url", "jdbc:mysql://prod:3306/app",
                       "spring.datasource.username", "produser",
                       "spring.datasource.password", "prodpass",
                       "spring.data.redis.host", "redis.prod.internal"),
                TestcontainersJdbcConfig.class
        );

        assertThat(ctx.getBean(JdbcConnectionDetails.class).getJdbcUrl())
                .isEqualTo("jdbc:postgresql://localhost:55432/testdb");
        assertThat(ctx.getBean(RedisConnectionDetails.class).getHost())
                .isEqualTo("redis.prod.internal");

        ctx.close();
    }

    @Test
    @DisplayName("should not create ConnectionDetails when properties are absent")
    void shouldNotCreateConnectionDetails_WhenPropertiesAbsent() {
        var ctx = createContext(Map.of());
        assertThat(ctx.containsBean("jdbcConnectionDetails")).isFalse();
        assertThat(ctx.containsBean("redisConnectionDetails")).isFalse();
        ctx.close();
    }

    @Test
    @DisplayName("should override both JDBC and Redis simultaneously")
    void shouldOverrideBothSimultaneously() {
        var ctx = createContext(
                Map.of("spring.datasource.url", "jdbc:h2:mem:ignored",
                       "spring.data.redis.host", "ignored-host"),
                TestcontainersJdbcConfig.class,
                DockerComposeRedisConfig.class
        );

        assertThat(ctx.getBean(JdbcConnectionDetails.class).getJdbcUrl())
                .isEqualTo("jdbc:postgresql://localhost:55432/testdb");
        assertThat(ctx.getBean(RedisConnectionDetails.class).getPort())
                .isEqualTo(63790);

        ctx.close();
    }
}
```

---

## Summary

| Concept | What you built | Real Spring Boot equivalent |
|---------|---------------|---------------------------|
| Marker interface | `ConnectionDetails` | `ConnectionDetails` |
| SPI | `ConnectionDetailsFactory<S, D>` | `ConnectionDetailsFactory<S, D>` |
| JDBC interface | `JdbcConnectionDetails` | `JdbcConnectionDetails` |
| JDBC properties | `DataSourceProperties` | `DataSourceProperties` |
| JDBC default | `PropertiesJdbcConnectionDetails` | `PropertiesJdbcConnectionDetails` |
| JDBC auto-config | `DataSourceAutoConfiguration` | `DataSourceAutoConfiguration` |
| Redis interface | `RedisConnectionDetails` | `DataRedisConnectionDetails` |
| Redis properties | `RedisProperties` | `DataRedisProperties` |
| Redis default | `PropertiesRedisConnectionDetails` | `PropertiesDataRedisConnectionDetails` |
| Redis auto-config | `RedisAutoConfiguration` | `DataRedisAutoConfiguration` |

**Files created:** 10 production files + 4 test files
**Files modified:** 2 (AutoConfiguration.imports, AutoConfigurationImportSelectorTest)

**Next chapter:** [Chapter 24: Actuator & Health](ch24_actuator_and_health.md) — expose `/actuator/health`, `/actuator/info`, and `/actuator/metrics` endpoints by scanning for `@Endpoint` beans and mapping `@ReadOperation`/`@WriteOperation` methods to HTTP paths.
