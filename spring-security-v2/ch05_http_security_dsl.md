# Chapter 5: HTTP Security DSL & Auto-Configuration

## Build Challenge

Before reading any code, try to implement this yourself:

```java
// A fluent builder that produces a SecurityFilterChain:
SimpleHttpSecurity http = new SimpleHttpSecurity();

// Register a custom configurer that adds a filter during build:
http.with(new MyLoggingConfigurer(), configurer -> configurer.setLevel("DEBUG"));

// Add some filters at explicit positions:
http.addFilterAt(securityContextFilter, 100)
    .addFilterAt(authorizationFilter, 600);

// Restrict which requests this chain handles:
http.securityMatcher("/api/**");

// Build the chain — runs init() then configure() on all configurers:
SecurityFilterChain chain = http.build();

// The chain contains sorted filters and a request matcher
assertThat(chain.getFilters()).hasSize(3);  // 2 manual + 1 from configurer
assertThat(chain.matches(apiRequest)).isTrue();
assertThat(chain.matches(webRequest)).isFalse();
```

Questions to guide your thinking:
- Why does the builder use a **two-phase** lifecycle (init then configure) instead of just one `configure()` call?
- Why is `getOrApply()` idempotent — calling `http.formLogin(c1).formLogin(c2)` reuses the *same* configurer?
- Why does the real framework make `HttpSecurity` a **prototype-scoped** Spring bean?
- Where does the `AuthenticationManager` come from, and why is it resolved *between* init and configure?

---

## The API Contract

Feature 5 introduces the **HTTP Security DSL** — the fluent builder (`SimpleHttpSecurity`)
that clients use to declare security rules, and the `@EnableSimpleWebSecurity` annotation
that bootstraps the entire security infrastructure from a Spring `ApplicationContext`.

There are three layers in the public API:

### Layer 1: The Customizer functional interface

```java
// simple/security/config/Customizer.java
@FunctionalInterface
public interface Customizer<T> {
    void customize(T t);

    static <T> Customizer<T> withDefaults() {
        return (t) -> {};
    }
}
```

`Customizer` is the lambda-friendly hook used throughout the DSL. `withDefaults()` returns
a no-op customizer, meaning "accept the defaults." This single interface enables the entire
fluent lambda API: `http.formLogin(form -> form.loginPage("/login"))`.

### Layer 2: The builder (`SimpleHttpSecurity`)

```java
// simple/security/config/annotation/web/builders/SimpleHttpSecurity.java
public final class SimpleHttpSecurity implements SecurityBuilder<SecurityFilterChain> {
    public <C extends SimpleAbstractHttpConfigurer<C>> SimpleHttpSecurity with(
            C configurer, Customizer<C> customizer) { ... }
    public SimpleHttpSecurity addFilter(Filter filter) { ... }
    public SimpleHttpSecurity addFilterBefore(Filter filter, Class<? extends Filter> beforeFilter) { ... }
    public SimpleHttpSecurity addFilterAfter(Filter filter, Class<? extends Filter> afterFilter) { ... }
    public SimpleHttpSecurity securityMatcher(String... patterns) { ... }
    public SimpleHttpSecurity authenticationManager(AuthenticationManager manager) { ... }
    public <C> void setSharedObject(Class<C> type, C object) { ... }
    public <C> C getSharedObject(Class<C> type) { ... }
    public SecurityFilterChain build() throws Exception { ... }
}
```

`SimpleHttpSecurity` is the central DSL class. Clients receive it (typically as a Spring
bean), call DSL methods to register configurers, and call `build()` to produce a
`SecurityFilterChain`. The build lifecycle runs all configurers through init → configure,
then sorts the accumulated filters.

### Layer 3: The configurer base class

```java
// simple/security/config/annotation/web/configurers/SimpleAbstractHttpConfigurer.java
public abstract class SimpleAbstractHttpConfigurer<T extends SimpleAbstractHttpConfigurer<T>>
        implements SecurityConfigurer<SecurityFilterChain, SimpleHttpSecurity> {
    public void init(SimpleHttpSecurity builder) throws Exception { }
    public void configure(SimpleHttpSecurity builder) throws Exception { }
    public SimpleHttpSecurity disable() { ... }
    protected SimpleHttpSecurity getBuilder() { ... }
}
```

`SimpleAbstractHttpConfigurer` is the base class for all DSL extensions. Each concrete
configurer (form login, HTTP basic, CSRF, etc.) extends this to add its own configuration
options and create its filter during `configure()`. The self-referential type parameter `T`
enables fluent chaining within the configurer.

### Layer 4: The bootstrap annotation

```java
// simple/security/config/annotation/web/configuration/EnableSimpleWebSecurity.java
@Import({ SimpleHttpSecurityConfiguration.class, SimpleWebSecurityConfiguration.class })
public @interface EnableSimpleWebSecurity { }
```

`@EnableSimpleWebSecurity` imports two configuration classes:
- `SimpleHttpSecurityConfiguration` — creates a prototype-scoped `SimpleHttpSecurity` bean
- `SimpleWebSecurityConfiguration` — collects all `SecurityFilterChain` beans and assembles
  them into a `SimpleFilterChainProxy`

### Client usage example

```java
@EnableSimpleWebSecurity
@Configuration
public class SecurityConfig {

    @Bean
    SecurityFilterChain filterChain(SimpleHttpSecurity http) throws Exception {
        // Future features add DSL methods like:
        // http.authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
        //     .formLogin(Customizer.withDefaults())
        //     .httpBasic(Customizer.withDefaults());

        // For now, use the lower-level configurer API:
        http.with(new MyCustomConfigurer(), c -> c.setEnabled(true));
        return http.build();
    }
}
```

---

## Client Usage & Tests

The tests in `HttpSecurityDslApiTest` define the behavioral contract from a client's
perspective. Key scenarios:

```java
@Test
void shouldBuildEmptyFilterChain() throws Exception {
    SimpleHttpSecurity http = new SimpleHttpSecurity();
    SecurityFilterChain chain = http.build();

    assertThat(chain).isNotNull();
    assertThat(chain.getFilters()).isEmpty();
    assertThat(chain.matches(new MockHttpServletRequest("GET", "/anything"))).isTrue();
}

@Test
void shouldExecuteConfigurerInitBeforeConfigure() throws Exception {
    List<String> lifecycle = new ArrayList<>();
    SimpleHttpSecurity http = new SimpleHttpSecurity();
    http.with(new LifecycleTrackingConfigurer(lifecycle), Customizer.withDefaults());

    http.build();

    assertThat(lifecycle).containsExactly("init", "configure");
}

@Test
void shouldExecuteMultipleConfigurersInRegistrationOrder() throws Exception {
    List<String> lifecycle = new ArrayList<>();
    SimpleHttpSecurity http = new SimpleHttpSecurity();
    http.with(new FirstConfigurer(lifecycle), Customizer.withDefaults());
    http.with(new SecondConfigurer(lifecycle), Customizer.withDefaults());

    http.build();

    // ALL inits run first, THEN all configures
    assertThat(lifecycle).containsExactly(
        "first-init", "second-init",
        "first-configure", "second-configure");
}

@Test
void shouldReturnExistingConfigurerOnSecondGetOrApply() {
    SimpleHttpSecurity http = new SimpleHttpSecurity();
    SettableConfigurer first = new SettableConfigurer();
    first.setValue("first");
    SettableConfigurer registered = http.getOrApply(first);

    SettableConfigurer second = new SettableConfigurer();
    second.setValue("second");
    SettableConfigurer returned = http.getOrApply(second);

    // Second call returns the EXISTING configurer, not the new one
    assertThat(returned.getValue()).isEqualTo("first");
    assertThat(returned).isSameAs(registered);
}

@Test
void configurerShouldAccessAuthenticationManagerFromSharedObjects() throws Exception {
    AuthenticationManager mockManager = authentication -> authentication;
    List<AuthenticationManager> captured = new ArrayList<>();

    SimpleHttpSecurity http = new SimpleHttpSecurity();
    http.authenticationManager(mockManager);
    http.with(new AuthManagerCapturingConfigurer(captured), Customizer.withDefaults());
    http.build();

    assertThat(captured).hasSize(1);
    assertThat(captured.get(0)).isSameAs(mockManager);
}
```

The `EnableSimpleWebSecurityIntegrationTest` tests the Spring Context bootstrap:

```java
@Test
void shouldCreateFilterChainProxyBean() {
    try (var ctx = new AnnotationConfigApplicationContext(EmptySecurityConfig.class)) {
        Filter filterChainProxy = ctx.getBean("springSecurityFilterChain", Filter.class);
        assertThat(filterChainProxy).isInstanceOf(SimpleFilterChainProxy.class);
    }
}

@Test
void shouldUseUserDefinedSecurityFilterChain() {
    try (var ctx = new AnnotationConfigApplicationContext(UserDefinedChainConfig.class)) {
        SimpleFilterChainProxy proxy = (SimpleFilterChainProxy)
            ctx.getBean("springSecurityFilterChain");
        List<SecurityFilterChain> chains = proxy.getFilterChains();

        assertThat(chains).hasSize(1);
        assertThat(chains.get(0).getFilters()).hasSize(1);
    }
}

@Test
void shouldMakeHttpSecurityPrototypeScoped() {
    try (var ctx = new AnnotationConfigApplicationContext(EmptySecurityConfig.class)) {
        SimpleHttpSecurity http1 = ctx.getBean(SimpleHttpSecurity.class);
        SimpleHttpSecurity http2 = ctx.getBean(SimpleHttpSecurity.class);
        assertThat(http1).isNotSameAs(http2);
    }
}
```

---

## Implementing the Call Chain

### Call chain recap

```
Client writes @Configuration with @EnableSimpleWebSecurity
  │
  ├─► [Bootstrap] @EnableSimpleWebSecurity
  │     @Import triggers SimpleHttpSecurityConfiguration + SimpleWebSecurityConfiguration
  │
  ├─► [Bean Factory] SimpleHttpSecurityConfiguration.simpleHttpSecurity()
  │     Creates a prototype-scoped SimpleHttpSecurity with ApplicationContext as shared object
  │
  ├─► [User Code] SecurityFilterChain filterChain(SimpleHttpSecurity http)
  │     User calls DSL methods → registers configurers → calls http.build()
  │
  ├─► [Build Lifecycle] SimpleHttpSecurity.build()
  │     │
  │     ├─► Phase 1: INITIALIZING
  │     │     configurer.init(http) for ALL configurers
  │     │     (configurers set up shared state here)
  │     │
  │     ├─► Phase 2: beforeConfigure()
  │     │     Resolve AuthenticationManager → setSharedObject
  │     │
  │     ├─► Phase 3: CONFIGURING
  │     │     configurer.configure(http) for ALL configurers
  │     │     (configurers create filters and call http.addFilter() here)
  │     │
  │     └─► Phase 4: performBuild()
  │           Sort filters by order → create SimpleDefaultSecurityFilterChain
  │
  └─► [Assembly] SimpleWebSecurityConfiguration.springSecurityFilterChain()
        Collects all SecurityFilterChain beans → wraps in SimpleFilterChainProxy
```

### 5a. Foundation: SecurityBuilder and SecurityConfigurer interfaces

The builder pattern starts with two foundational interfaces:

```java
// simple/security/config/annotation/SecurityBuilder.java
public interface SecurityBuilder<O> {
    O build() throws Exception;
}
```

```java
// simple/security/config/annotation/SecurityConfigurer.java
public interface SecurityConfigurer<O, B extends SecurityBuilder<O>> {
    default void init(B builder) throws Exception { }
    default void configure(B builder) throws Exception { }
}
```

**Why two phases?** During `init()`, configurers set up shared state (e.g., registering
an `AuthenticationEntryPoint` that other configurers need). During `configure()`, all
shared state is available, so configurers can safely create and add their filters. Without
two phases, configurer ordering would matter — whoever ran first would determine what
shared state existed. The two-phase approach makes configurers order-independent for
shared state setup.

**Why default methods?** Not every configurer needs both phases. Many configurers only
need `configure()` to add their filter. Default no-ops avoid forcing empty method
implementations.

### 5b. Configurer base: SimpleAbstractHttpConfigurer

```java
public abstract class SimpleAbstractHttpConfigurer<T extends SimpleAbstractHttpConfigurer<T>>
        implements SecurityConfigurer<SecurityFilterChain, SimpleHttpSecurity> {

    private SimpleHttpSecurity builder;

    public SimpleHttpSecurity disable() {
        getBuilder().removeConfigurer(getClass());
        return getBuilder();
    }

    protected SimpleHttpSecurity getBuilder() {
        return this.builder;
    }
}
```

**Why self-referential generics?** The `T extends SimpleAbstractHttpConfigurer<T>` pattern
(F-bounded polymorphism) enables fluent method chaining that returns the concrete type:

```java
public class FormLoginConfigurer extends SimpleAbstractHttpConfigurer<FormLoginConfigurer> {
    public FormLoginConfigurer loginPage(String page) {
        this.loginPage = page;
        return this;  // returns FormLoginConfigurer, not SimpleAbstractHttpConfigurer
    }
}
```

Without this, `loginPage()` would return the base type, and chaining
`configurer.loginPage("/login").failureUrl("/error")` would fail to compile because
`SimpleAbstractHttpConfigurer` doesn't have `failureUrl()`.

**Why `disable()`?** It allows users to opt out of a specific security feature:
`http.csrf(csrf -> csrf.disable())`. The configurer removes itself from the builder,
so it won't run during the build lifecycle.

### 5c. The DSL builder: SimpleHttpSecurity

The central class accumulates configurers and filters, then builds a `SecurityFilterChain`.

**Configurer registry** — a `LinkedHashMap` keyed by configurer class:

```java
private final LinkedHashMap<Class<?>, SecurityConfigurer<...>> configurers = new LinkedHashMap<>();
```

**Why LinkedHashMap?** Preserves insertion order. When the build lifecycle iterates
configurers, they run in the order they were registered. This is deterministic and
predictable.

**Why key by class?** Each configurer type can appear at most once. Calling
`http.formLogin(c1).formLogin(c2)` doesn't create two `FormLoginConfigurer`s — the second
call customizes the existing one. This is implemented by `getOrApply()`:

```java
public <C extends SimpleAbstractHttpConfigurer<C>> C getOrApply(C configurer) {
    SecurityConfigurer<...> existing = this.configurers.get(configurer.getClass());
    if (existing != null) {
        return (C) existing;  // reuse existing
    }
    configurer.setBuilder(this);
    this.configurers.put(configurer.getClass(), configurer);
    return configurer;
}
```

**Filter ordering** — well-known filter positions are pre-registered:

```java
private static final Map<String, Integer> FILTER_ORDER_MAP = new LinkedHashMap<>();
static {
    FILTER_ORDER_MAP.put("SecurityContextHolderFilter", 100);
    FILTER_ORDER_MAP.put("CsrfFilter", 200);
    FILTER_ORDER_MAP.put("SimpleUsernamePasswordAuthenticationFilter", 300);
    FILTER_ORDER_MAP.put("SimpleBasicAuthenticationFilter", 400);
    FILTER_ORDER_MAP.put("ExceptionTranslationFilter", 500);
    FILTER_ORDER_MAP.put("AuthorizationFilter", 600);
}
```

This mirrors the real framework's `FilterOrderRegistration` which maps ~30 filter types
to integer positions. `addFilter()` looks up the order by class name; `addFilterBefore/After`
offsets by ±1 relative to a registered position.

**The build lifecycle** — the heart of the class:

```java
public SecurityFilterChain build() throws Exception {
    // Phase 1: init all configurers
    for (SecurityConfigurer<...> configurer : getConfigurers()) {
        configurer.init(this);
    }

    // Resolve AuthenticationManager as shared object
    beforeConfigure();

    // Phase 2: configure all configurers
    for (SecurityConfigurer<...> configurer : getConfigurers()) {
        configurer.configure(this);
    }

    // Phase 3: sort and build
    return performBuild();
}

private SecurityFilterChain performBuild() {
    this.filters.sort(Comparator.comparingInt(f -> f.order));
    List<Filter> sortedFilters = new ArrayList<>();
    for (OrderedFilter of : this.filters) {
        sortedFilters.add(of.filter);
    }
    return new SimpleDefaultSecurityFilterChain(this.requestMatcher, sortedFilters);
}
```

**Why resolve AuthenticationManager between init and configure?** Configurers like
form login and HTTP basic need the `AuthenticationManager` during `configure()` to wire
it into their filters. But the `AuthenticationManager` might be set by the user *or*
built from registered `AuthenticationProvider`s. This resolution happens in
`beforeConfigure()`:

```java
private void beforeConfigure() {
    if (this.authenticationManager != null) {
        setSharedObject(AuthenticationManager.class, this.authenticationManager);
    }
    else if (getSharedObject(AuthenticationManager.class) == null
            && !this.authenticationProviders.isEmpty()) {
        setSharedObject(AuthenticationManager.class,
            new SimpleProviderManager(this.authenticationProviders));
    }
}
```

**Shared objects** — a typed `Map<Class<?>, Object>` that provides cross-configurer
communication. The `ApplicationContext` is pre-loaded as a shared object by the
configuration class. During `init()`, configurers can add more shared objects.
During `configure()`, all configurers can read any shared object.

### 5d. Bootstrap: @EnableSimpleWebSecurity and configuration classes

**`@EnableSimpleWebSecurity`** uses Spring's `@Import` mechanism to load two
configuration classes.

**`SimpleHttpSecurityConfiguration`** creates the prototype-scoped builder:

```java
@Bean
@Scope("prototype")
SimpleHttpSecurity simpleHttpSecurity() {
    Map<Class<?>, Object> sharedObjects = new HashMap<>();
    sharedObjects.put(ApplicationContext.class, this.applicationContext);
    return new SimpleHttpSecurity(sharedObjects);
}
```

**Why prototype-scoped?** Each `SecurityFilterChain` bean gets its own fresh
`SimpleHttpSecurity`. If it were singleton, all chains would share the same builder,
and `build()` could only be called once. Prototype scope means every `@Bean` method
that declares `SimpleHttpSecurity` as a parameter receives a new instance.

**`SimpleWebSecurityConfiguration`** assembles the final `FilterChainProxy`:

```java
@Bean(name = "springSecurityFilterChain")
Filter springSecurityFilterChain(ObjectProvider<SimpleHttpSecurity> httpSecurityProvider)
        throws Exception {
    if (this.securityFilterChains.isEmpty()) {
        SimpleHttpSecurity http = httpSecurityProvider.getObject();
        SecurityFilterChain defaultChain = http.build();
        return new SimpleFilterChainProxy(List.of(defaultChain));
    }
    return new SimpleFilterChainProxy(this.securityFilterChains);
}
```

If users define `SecurityFilterChain` beans, those are used. If not, a default empty
chain is created. The bean is named `"springSecurityFilterChain"` — the same name
the real framework uses, which `DelegatingFilterProxy` looks up by default.

---

## Try It Yourself

**Challenge 1**: Write a `LoggingConfigurer` that adds a filter which prints
"Processing request: {URI}" before each request. Register it using
`http.with(new LoggingConfigurer(), Customizer.withDefaults())` and verify the logging
output in a test.

**Challenge 2**: Write a configurer that uses `init()` to set a shared object
(`http.setSharedObject(MyConfig.class, config)`) and another configurer that reads
that shared object during `configure()`. Verify the two-phase lifecycle ensures
the object is available.

**Challenge 3**: Implement a `CompositeRequestMatcher` that takes multiple
`RequestMatcher`s and returns `true` if any of them match. Use it with
`http.securityMatcher(compositeRequestMatcher)`.

**Challenge 4**: What happens if you call `http.build()` twice? Why does the real
framework use an `AtomicBoolean` for its one-shot guard instead of a plain `boolean`?
(Hint: think about thread safety during application startup.)

---

## Why This Works

**The Builder pattern**: `SimpleHttpSecurity` collects configuration declaratively (via
DSL methods and configurers) and produces the final object (`SecurityFilterChain`) in a
single `build()` call. This separates *what* the user wants from *how* it's assembled.
The user never creates filters directly — they declare intent, and the builder translates
intent into a correctly ordered filter chain.

**The Strategy pattern**: Each `SimpleAbstractHttpConfigurer` is a pluggable strategy for
one aspect of security. `FormLoginConfigurer` knows how to create a login filter.
`CsrfConfigurer` knows how to create a CSRF filter. The builder doesn't need to know the
details — it just calls `init()` and `configure()` on each strategy.

**The Template Method pattern**: The `build()` method defines the algorithm skeleton:
init → beforeConfigure → configure → performBuild. Subclasses (or in our case,
configurers) fill in the variable steps. This ensures the lifecycle order is always
correct regardless of which configurers are registered.

**Shared objects as dependency injection**: The `sharedObjects` map is a lightweight
dependency injection mechanism scoped to a single build. Configurers don't depend on
each other directly — they depend on *shared objects*. This decouples configurers and
makes them independently testable.

**Prototype scope for builder isolation**: Each `SecurityFilterChain` bean gets its own
`SimpleHttpSecurity`, so configuring one chain doesn't affect another. This is critical
for applications with multiple chains (e.g., one for API endpoints, one for web pages).

---

## What We Enhanced

| Layer | Feature 1 | Feature 2 | Feature 3 | Feature 4 | Feature 5 (This Chapter) |
|-------|-----------|-----------|-----------|-----------|--------------------------|
| Crypto | `PasswordEncoder` / BCrypt | — | Used by `SimpleDaoAuthenticationProvider` | — | — |
| Identity model | — | `Authentication`, `UserDetails`, `SecurityContext` | Used as pipeline inputs/outputs | `SecurityContextHolder.clearContext()` called by `FilterChainProxy` | `AuthenticationManager` resolved as shared object during build |
| Pipeline | — | — | `AuthenticationManager`, providers, user store | — | Wired into `SimpleHttpSecurity` via `.authenticationManager()` and `.authenticationProvider()` |
| Filter dispatch | — | — | — | `SecurityFilterChain`, `SimpleFilterChainProxy`, `VirtualFilterChain` | `SimpleFilterChainProxy` assembled by `SimpleWebSecurityConfiguration` |
| Request matching | — | — | — | `RequestMatcher`, `SimplePathPatternRequestMatcher` | Used by `.securityMatcher()` DSL method |
| DSL & Config | — | — | — | — | `SimpleHttpSecurity`, `Customizer`, `@EnableSimpleWebSecurity`, configurer lifecycle |

Feature 5 is the **orchestration layer**. It doesn't add any new security filters or
request processing logic — instead, it provides the machinery that *assembles* those
components. After this chapter, clients can use a fluent DSL to build filter chains
and bootstrap the entire security infrastructure with a single annotation. Features 6-8
will add concrete DSL methods (`.formLogin()`, `.httpBasic()`, `.authorizeHttpRequests()`,
`.csrf()`) that register configurers producing real security filters.

---

## Connection to Real Framework

| Our Class | Real Framework Class | Key Difference |
|-----------|---------------------|----------------|
| `SecurityBuilder` | `o.s.s.config.annotation.SecurityBuilder` | Identical interface |
| `SecurityConfigurer` | `o.s.s.config.annotation.SecurityConfigurer` | Uses default methods instead of separate `SecurityConfigurerAdapter` base class |
| `Customizer` | `o.s.s.config.Customizer` | Identical interface |
| `SimpleAbstractHttpConfigurer` | `o.s.s.config.annotation.web.configurers.AbstractHttpConfigurer` | No `ObjectPostProcessor` support, no `SecurityContextHolderStrategy` accessor |
| `SimpleHttpSecurity` | `o.s.s.config.annotation.web.builders.HttpSecurity` | No 3-level builder hierarchy (`AbstractSecurityBuilder` → `AbstractConfiguredSecurityBuilder` → `HttpSecurity`), no `HttpSecurityBuilder` interface, no `FilterOrderRegistration` (uses simple static map), no `ObjectPostProcessor`, no `RequestMatcherConfigurer` inner class |
| `@EnableSimpleWebSecurity` | `@EnableWebSecurity` | No `@EnableGlobalAuthentication`, no `SpringWebMvcImportSelector`, no `OAuth2ImportSelector`, no `ObservationImportSelector` |
| `SimpleHttpSecurityConfiguration` | `HttpSecurityConfiguration` | No default configurers (csrf, headers, session, etc.), no `SpringFactoriesLoader` SPI, no `Customizer<HttpSecurity>` bean discovery |
| `SimpleWebSecurityConfiguration` | `WebSecurityConfiguration` | No `WebSecurity` intermediate builder, no `WebSecurityCustomizer`, no `@Order` support for chain precedence, no `ImportAware` for debug mode |

### Real framework: the three-level builder hierarchy

The real `HttpSecurity` inherits from a carefully layered builder hierarchy:

```
SecurityBuilder<O>                                   (interface: O build())
  │
AbstractSecurityBuilder<O>                           (one-shot guard via AtomicBoolean)
  │                                                   + caches built object
  │
AbstractConfiguredSecurityBuilder<O, B>              (configurer registry + lifecycle)
  │                                                   + LinkedHashMap<Class, List<Configurer>>
  │                                                   + init → configure → performBuild lifecycle
  │                                                   + shared objects
  │                                                   + BuildState enum: UNBUILT → INITIALIZING → CONFIGURING → BUILDING → BUILT
  │
HttpSecurity                                         (DSL methods + filter accumulation)
                                                      + FilterOrderRegistration (30+ filters)
                                                      + OrderedFilter inner class
                                                      + ObjectPostProcessor integration
```

Our `SimpleHttpSecurity` flattens this entire hierarchy into a single class. The key
behaviors we retain:
- One-shot build guard (simple boolean instead of `AtomicBoolean` — no thread safety)
- Configurer registry with two-phase lifecycle
- Shared objects
- Filter ordering and sorting

The key behaviors we omit:
- `BuildState` enum tracking (prevents adding configurers after build starts)
- `ObjectPostProcessor` integration (enables Spring-managed post-processing of filters)
- `configurersAddedInInitializing` list (handles configurers added during init phase)
- Thread-safe `AtomicBoolean` build guard
- `allowConfigurersOfSameType` flag (for multiple configurers of the same class)

The real framework also uses `WebSecurity` as an additional builder layer between
`WebSecurityConfiguration` and `FilterChainProxy`. `WebSecurity` collects multiple
`SecurityFilterChain` builders (each `HttpSecurity`) and assembles them into a single
`FilterChainProxy`. Our simplified version skips `WebSecurity` and has
`SimpleWebSecurityConfiguration` collect the already-built `SecurityFilterChain` beans
directly.

---

## Complete Code

### [NEW] `simple-security-config/src/main/java/simple/security/config/Customizer.java`

```java
package simple.security.config;

/**
 * Callback interface that accepts a single input argument and returns no result.
 * Used throughout the {@code SimpleHttpSecurity} DSL to let clients customize
 * individual security configurers via lambda expressions.
 *
 * <p>Simplified version of
 * {@code org.springframework.security.config.Customizer}.
 *
 * @param <T> the type of the input to the operation
 */
@FunctionalInterface
public interface Customizer<T> {

	void customize(T t);

	static <T> Customizer<T> withDefaults() {
		return (t) -> {
		};
	}

}
```

### [NEW] `simple-security-config/src/main/java/simple/security/config/annotation/SecurityBuilder.java`

```java
package simple.security.config.annotation;

/**
 * Interface for building an Object.
 *
 * <p>Simplified version of
 * {@code org.springframework.security.config.annotation.SecurityBuilder}.
 *
 * @param <O> the type of the Object being built
 */
public interface SecurityBuilder<O> {

	O build() throws Exception;

}
```

### [NEW] `simple-security-config/src/main/java/simple/security/config/annotation/SecurityConfigurer.java`

```java
package simple.security.config.annotation;

/**
 * Allows for configuring a {@link SecurityBuilder}. All configurers are first
 * initialized ({@link #init}) and then configured ({@link #configure}).
 *
 * <p>Simplified version of
 * {@code org.springframework.security.config.annotation.SecurityConfigurer}.
 *
 * @param <O> the type of Object being built
 * @param <B> the type of SecurityBuilder being configured
 */
public interface SecurityConfigurer<O, B extends SecurityBuilder<O>> {

	default void init(B builder) throws Exception { }

	default void configure(B builder) throws Exception { }

}
```

### [NEW] `simple-security-config/src/main/java/simple/security/config/annotation/web/configurers/SimpleAbstractHttpConfigurer.java`

```java
package simple.security.config.annotation.web.configurers;

import simple.security.config.annotation.SecurityConfigurer;
import simple.security.config.annotation.web.builders.SimpleHttpSecurity;
import simple.security.web.SecurityFilterChain;

/**
 * Base class for all configurers that configure {@link SimpleHttpSecurity}.
 *
 * <p>Simplified version of
 * {@code o.s.s.config.annotation.web.configurers.AbstractHttpConfigurer}.
 *
 * @param <T> the concrete configurer type (self-referential for fluent API)
 */
public abstract class SimpleAbstractHttpConfigurer<T extends SimpleAbstractHttpConfigurer<T>>
		implements SecurityConfigurer<SecurityFilterChain, SimpleHttpSecurity> {

	private SimpleHttpSecurity builder;

	@Override
	public void init(SimpleHttpSecurity builder) throws Exception { }

	@Override
	public void configure(SimpleHttpSecurity builder) throws Exception { }

	@SuppressWarnings("unchecked")
	public SimpleHttpSecurity disable() {
		getBuilder().removeConfigurer(
			(Class<? extends SecurityConfigurer<SecurityFilterChain, SimpleHttpSecurity>>) getClass());
		return getBuilder();
	}

	public void setBuilder(SimpleHttpSecurity builder) {
		this.builder = builder;
	}

	protected SimpleHttpSecurity getBuilder() {
		return this.builder;
	}

}
```

### [NEW] `simple-security-config/src/main/java/simple/security/config/annotation/web/builders/SimpleHttpSecurity.java`

See full source: `simple-security-config/src/main/java/simple/security/config/annotation/web/builders/SimpleHttpSecurity.java`

Key sections:
- Static `FILTER_ORDER_MAP` with 6 well-known filter positions (100–600)
- Configurer registry (`LinkedHashMap`) with `with()`, `getOrApply()`, `getConfigurer()`, `removeConfigurer()`
- Filter management: `addFilter()`, `addFilterBefore()`, `addFilterAfter()`, `addFilterAt()`
- Authentication: `authenticationManager()`, `authenticationProvider()`, `userDetailsService()`
- Security matcher: `securityMatcher(RequestMatcher)`, `securityMatcher(String...)`
- Shared objects: `getSharedObject()`, `setSharedObject()`, `getSharedObjects()`
- Build lifecycle: `build()` → init → beforeConfigure → configure → `performBuild()`
- Inner class: `OrderedFilter` wrapping `Filter` + `int order`

### [NEW] `simple-security-config/src/main/java/simple/security/config/annotation/web/configuration/EnableSimpleWebSecurity.java`

```java
package simple.security.config.annotation.web.configuration;

import java.lang.annotation.*;
import org.springframework.context.annotation.Import;

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Import({ SimpleHttpSecurityConfiguration.class, SimpleWebSecurityConfiguration.class })
public @interface EnableSimpleWebSecurity { }
```

### [NEW] `simple-security-config/src/main/java/simple/security/config/annotation/web/configuration/SimpleHttpSecurityConfiguration.java`

```java
package simple.security.config.annotation.web.configuration;

import java.util.HashMap;
import java.util.Map;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Scope;

import simple.security.config.annotation.web.builders.SimpleHttpSecurity;

@Configuration(proxyBeanMethods = false)
class SimpleHttpSecurityConfiguration {

	@Autowired
	private ApplicationContext applicationContext;

	@Bean
	@Scope("prototype")
	SimpleHttpSecurity simpleHttpSecurity() {
		Map<Class<?>, Object> sharedObjects = new HashMap<>();
		sharedObjects.put(ApplicationContext.class, this.applicationContext);
		return new SimpleHttpSecurity(sharedObjects);
	}

}
```

### [NEW] `simple-security-config/src/main/java/simple/security/config/annotation/web/configuration/SimpleWebSecurityConfiguration.java`

```java
package simple.security.config.annotation.web.configuration;

import java.util.Collections;
import java.util.List;

import jakarta.servlet.Filter;

import org.springframework.beans.factory.ObjectProvider;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import simple.security.config.annotation.web.builders.SimpleHttpSecurity;
import simple.security.web.SecurityFilterChain;
import simple.security.web.SimpleFilterChainProxy;

@Configuration(proxyBeanMethods = false)
class SimpleWebSecurityConfiguration {

	private List<SecurityFilterChain> securityFilterChains = Collections.emptyList();

	@Autowired(required = false)
	void setFilterChains(List<SecurityFilterChain> securityFilterChains) {
		this.securityFilterChains = securityFilterChains;
	}

	@Bean(name = "springSecurityFilterChain")
	Filter springSecurityFilterChain(
			ObjectProvider<SimpleHttpSecurity> httpSecurityProvider) throws Exception {
		if (this.securityFilterChains.isEmpty()) {
			SimpleHttpSecurity http = httpSecurityProvider.getObject();
			SecurityFilterChain defaultChain = http.build();
			return new SimpleFilterChainProxy(List.of(defaultChain));
		}
		return new SimpleFilterChainProxy(this.securityFilterChains);
	}

}
```

### Test files

- `simple-security-config/src/test/java/simple/security/config/HttpSecurityDslApiTest.java` — 20 tests covering: empty build, double-build guard, filter ordering, configurer lifecycle, multiple configurers, customizer application, configurer filter adding, getOrApply idempotence, disable, shared objects, constructor injection, AuthenticationManager shared object resolution, UserDetailsService, security matcher (single and multiple patterns), Customizer.withDefaults no-op
- `simple-security-config/src/test/java/simple/security/config/EnableSimpleWebSecurityIntegrationTest.java` — 7 integration tests covering: FilterChainProxy bean creation, default chain, user-defined chain, multiple chains, prototype scope, ApplicationContext shared object, configurer-based chain
