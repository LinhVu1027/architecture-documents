# Spring Boot Starter Architecture — From `@SpringBootApplication` to Bean Creation

> How does adding a single dependency like `spring-boot-starter-kafka` cause dozens of beans to appear in your application context — without a single line of configuration code?

This document dissects the internal mechanism, using real Spring Boot source code with exact file:line references, and maps every concept to our custom ZaloBot starter.

---

## 1. Overview — The Restaurant Franchise Analogy

Think of Spring Boot's auto-configuration system as a **restaurant franchise**:

| Franchise Concept | Spring Boot Concept | Example |
|---|---|---|
| **Franchise kit** (shopping list of ingredients, zero recipes) | **Starter** — a POM/Gradle file with dependencies, zero Java code | `starter/spring-boot-starter-kafka/build.gradle` |
| **Operations manual** (recipes + conditional logic: "if you have a pizza oven, open the pizza station") | **Module** — auto-configuration classes with `@Conditional` annotations | `module/spring-boot-kafka/` containing `KafkaAutoConfiguration.java` |
| **Franchise HQ** that reads every operations manual and decides which restaurants to open | **Core engine** — `AutoConfigurationImportSelector` + `ImportCandidates` | `AutoConfigurationImportSelector.java:78` |

**The flow:**
1. You add a franchise kit (starter dependency) to your project
2. The kit brings in the operations manual (module with auto-config classes) and the raw ingredients (library JARs)
3. At startup, franchise HQ reads all operations manuals it can find on the classpath
4. For each manual, HQ checks the conditions ("do we have the right equipment?")
5. If conditions pass, HQ follows the recipes to set up the restaurant stations (create beans)

---

## 2. Interface Hierarchy — The Players

```
Spring Framework
================
Condition                                          (interface — single method: matches())
  |
DeferredImportSelector                             (interface — extends ImportSelector)
  |                                                   processed AFTER all @Configuration classes


Spring Boot
===========
DeferredImportSelector
  └── AutoConfigurationImportSelector               ← The engine (line 78)
        implements: BeanClassLoaderAware, ResourceLoaderAware,
                    BeanFactoryAware, EnvironmentAware, Ordered

Condition
  └── SpringBootCondition (abstract)                 ← Template Method pattern
        │   matches() is final → delegates to abstract getMatchOutcome()
        │
        ├── FilteringSpringBootCondition (abstract)  ← Dual role: Condition + ImportFilter
        │     also implements AutoConfigurationImportFilter
        │     │
        │     ├── OnClassCondition                   @Order(HIGHEST_PRECEDENCE)
        │     │     checks: @ConditionalOnClass, @ConditionalOnMissingClass
        │     │
        │     └── OnBeanCondition                    @Order(LOWEST_PRECEDENCE)
        │           checks: @ConditionalOnBean, @ConditionalOnMissingBean,
        │                   @ConditionalOnSingleCandidate
        │
        └── OnPropertyCondition                      @Order(HIGHEST_PRECEDENCE + 40)
              checks: @ConditionalOnProperty, @ConditionalOnBooleanProperty
```

**Key insight:** `OnClassCondition` and `OnBeanCondition` extend `FilteringSpringBootCondition`, which implements **both** `Condition` (for full evaluation in Phase 6) **and** `AutoConfigurationImportFilter` (for fast early filtering in Phase 4). This dual role is the performance optimization that makes Spring Boot start fast.

---

## 3. ASCII Class Diagram — Core Engine

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    @EnableAutoConfiguration (line 77)                    │
│                    @Import(AutoConfigurationImportSelector.class)        │
└───────────────────────────────────┬─────────────────────────────────────┘
                                    │ triggers
                                    v
┌─────────────────────────────────────────────────────────────────────────┐
│              AutoConfigurationImportSelector (line 78)                   │
│              implements DeferredImportSelector                           │
│                                                                         │
│  getAutoConfigurationEntry(metadata)        ← main pipeline (line 142)  │
│    │                                                                    │
│    ├── getCandidateConfigurations()         ← calls ImportCandidates    │
│    │     └── ImportCandidates.load(AutoConfiguration.class, cl)         │
│    │           reads: META-INF/spring/<annotation>.imports              │
│    │                                                                    │
│    ├── removeDuplicates(configurations)     ← LinkedHashSet (line 148)  │
│    │                                                                    │
│    ├── getExclusions(metadata, attributes)  ← exclude=, excludeName=   │
│    │                                           spring.autoconfigure     │
│    │                                           .exclude (line 149-151)  │
│    │                                                                    │
│    ├── getConfigurationClassFilter()        ← early filter (line 152)   │
│    │     └── ConfigurationClassFilter                                   │
│    │           ├── AutoConfigurationMetadataLoader (annotation cache)    │
│    │           └── filters: [OnClassCondition, OnBeanCondition]         │
│    │                                                                    │
│    └── fireAutoConfigurationImportEvents()  ← listeners (line 153)      │
│                                                                         │
│  getImportGroup() → AutoConfigurationGroup (line 158-159)               │
│    └── selectImports()                      ← SORTING happens here      │
│          └── AutoConfigurationSorter        ← @AutoConfigureBefore/     │
│                                                @AutoConfigureAfter      │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    v reads from
┌─────────────────────────────────────────────────────────────────────────┐
│                   ImportCandidates (line 45)                             │
│                                                                         │
│  LOCATION = "META-INF/spring/%s.imports"         (line 47)              │
│                                                                         │
│  load(annotation, classLoader)                   (line 81)              │
│    ├── format location with annotation.getName()                        │
│    ├── classLoader.getResources(location)        ← multi-JAR discovery  │
│    └── read each URL line by line                                       │
│          └── strip comments (#), trim, collect                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 4. State Diagram — The 6-Phase Loading Lifecycle

```
  ┌──────────┐     ┌──────────┐     ┌───────────┐     ┌──────────────┐     ┌──────────┐     ┌────────────────────┐
  │          │     │          │     │           │     │              │     │          │     │                    │
  │ DISCOVERY├────►│  DEDUP   ├────►│ EXCLUSION ├────►│EARLY FILTER  ├────►│ SORTING  ├────►│ FULL CONDITION     │
  │          │     │          │     │           │     │              │     │          │     │ EVALUATION         │
  └──────────┘     └──────────┘     └───────────┘     └──────────────┘     └──────────┘     └────────────────────┘
       │                │                 │                  │                   │                    │
       v                v                 v                  v                   v                    v
  ImportCandidates   removeDuplicates  getExclusions    ConfigurationClass    AutoConfiguration   SpringBootCondition
  .load()            (LinkedHashSet)   + removeAll()    Filter.filter()       Sorter               .matches()
                                                                              .getInPriorityOrder   → getMatchOutcome()
  ─────────────────────────────────────────────────────────────────────────────────────────────────────────────
  Source References:
  Phase 1: ImportCandidates.java:81-92         — load .imports files from all JARs
  Phase 2: AutoConfigurationImportSelector     — new ArrayList<>(new LinkedHashSet<>(list))
           .java:148,305-307
  Phase 3: AutoConfigurationImportSelector     — getExclusions() + configurations.removeAll()
           .java:149-151,247-255
  Phase 4: AutoConfigurationImportSelector     — getConfigurationClassFilter().filter()
           .java:152,282-293,399-427           — FilteringSpringBootCondition.java:52-69
  Phase 5: AutoConfigurationGroup              — sortAutoConfigurations() via
           .java:502,520-527                     AutoConfigurationSorter using @Before/@After
  Phase 6: SpringBootCondition.java:44-62      — matches() → getMatchOutcome() per class and per @Bean
```

### Phase Details

**Phase 1 — DISCOVERY:** `ImportCandidates.load(AutoConfiguration.class, classLoader)` scans all JARs on the classpath for files at `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`. Each JAR contributes its own file. Lines are read, comments stripped, and all candidates collected into a single list.

**Phase 2 — DEDUP:** Since multiple JARs might list the same class, `removeDuplicates()` wraps the list in a `LinkedHashSet` to eliminate duplicates while preserving order.

**Phase 3 — EXCLUSION:** User-specified exclusions from `@EnableAutoConfiguration(exclude=...)`, `@SpringBootApplication(exclude=...)`, and the `spring.autoconfigure.exclude` property are collected and removed.

**Phase 4 — EARLY FILTERING:** This is the **performance-critical** phase. The `ConfigurationClassFilter` uses `AutoConfigurationMetadataLoader` (an annotation-processor-generated cache) plus `AutoConfigurationImportFilter` implementations (`OnClassCondition`, `OnBeanCondition`) to quickly eliminate candidates **without loading their classes**. A typical Spring Boot app starts with ~150 candidates and early filtering eliminates ~100 of them in milliseconds.

**Phase 5 — SORTING:** The surviving candidates are topologically sorted using `@AutoConfigureBefore` and `@AutoConfigureAfter` annotations, ensuring that if `ZaloBotListenerAutoConfiguration` declares `after = ZaloBotClientAutoConfiguration.class`, the client config is processed first.

**Phase 6 — FULL CONDITION EVALUATION:** Spring's `ConfigurationClassParser` processes each surviving class. For each class and each `@Bean` method, all `@Conditional` annotations are evaluated via `SpringBootCondition.matches()`. This is where `@ConditionalOnMissingBean`, `@ConditionalOnProperty`, and fine-grained conditions take effect.

---

## 5. Flow Diagrams

### Flow A — Kafka Starter Loading Trace

```
User's pom.xml
  └── spring-boot-starter-kafka (build.gradle:23-27)
        ├── spring-boot-starter (base Spring Boot)
        └── spring-boot-kafka (module with auto-config)

[Application Starts]
  │
  ▼
@SpringBootApplication
  └── @EnableAutoConfiguration
        └── @Import(AutoConfigurationImportSelector.class)
              │
              ▼ Phase 1: DISCOVERY
              ImportCandidates.load(AutoConfiguration.class, classLoader)
              reads META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
              from EVERY JAR → finds ~150 candidates total including:
                - org.springframework.boot.kafka.autoconfigure.KafkaAutoConfiguration
                - org.springframework.boot.kafka.autoconfigure.metrics.KafkaMetricsAutoConfiguration
              │
              ▼ Phase 2-3: DEDUP + EXCLUSION
              ~150 → ~150 (no duplicates, no user exclusions)
              │
              ▼ Phase 4: EARLY FILTERING
              OnClassCondition checks annotation metadata cache:
                KafkaAutoConfiguration requires KafkaTemplate.class
                  → KafkaTemplate on classpath? YES (kafka-clients JAR present)
                  → PASS
              ~150 → ~50 (classes missing from classpath eliminated)
              │
              ▼ Phase 5: SORTING
              KafkaAutoConfiguration has no @AutoConfigureBefore/@After
              → default order maintained
              │
              ▼ Phase 6: FULL CONDITION EVALUATION
              KafkaAutoConfiguration.class:
                @ConditionalOnClass(KafkaTemplate.class)  → MATCH
                │
                @EnableConfigurationProperties(KafkaProperties.class)
                → binds spring.kafka.* from application.yml
                │
                Each @Bean method:
                  kafkaTemplate()           → @ConditionalOnMissingBean(KafkaTemplate.class) → no user bean → CREATE
                  kafkaConsumerFactory()     → @ConditionalOnMissingBean(ConsumerFactory.class) → CREATE
                  kafkaProducerFactory()     → @ConditionalOnMissingBean(ProducerFactory.class) → CREATE
                  kafkaAdmin()              → @ConditionalOnMissingBean → CREATE
                  kafkaTransactionManager() → @ConditionalOnProperty("spring.kafka.producer.transaction-id-prefix")
                                              → property not set → SKIP
```

### Flow B — ZaloBot Starter Loading Trace

```
User's pom.xml
  └── spring-boot-starter-zalobot
        ├── spring-boot-starter (base Spring Boot)
        ├── spring-boot-zalobot (module with auto-config)
        ├── zalobot-client
        └── zalobot-listener

application.yml:
  zalobot:
    bot-token: "abc123"
    listener:
      concurrency: 2

[Application Starts]
  │
  ▼
@SpringBootApplication
  └── @EnableAutoConfiguration
        └── @Import(AutoConfigurationImportSelector.class)
              │
              ▼ Phase 1: DISCOVERY
              reads .imports from spring-boot-zalobot JAR:
                - dev.linhvu.zalobot.boot.autoconfigure.ZaloBotClientAutoConfiguration
                - dev.linhvu.zalobot.autoconfigure.ZaloBotListenerAutoConfiguration
              │
              ▼ Phase 4: EARLY FILTERING
              OnClassCondition:
                ZaloBotClientAutoConfiguration requires ZaloBotClient.class → PRESENT → PASS
                ZaloBotListenerAutoConfiguration requires UpdateListenerContainer.class → PRESENT → PASS
              │
              ▼ Phase 5: SORTING
              ZaloBotListenerAutoConfiguration declares: after = ZaloBotClientAutoConfiguration.class
              → Order: Client first, then Listener
              │
              ▼ Phase 6: FULL CONDITION EVALUATION
              │
              ├── ZaloBotClientAutoConfiguration:
              │     @ConditionalOnClass(ZaloBotClient.class)       → MATCH
              │     @ConditionalOnProperty("zalobot.bot-token")    → "abc123" → MATCH
              │     @EnableConfigurationProperties(ZaloBotProperties.class) → binds zalobot.*
              │     │
              │     zaloBotClientBuilder()  → @ConditionalOnMissingBean → CREATE (prototype)
              │     zaloBotClient()         → @ConditionalOnMissingBean → CREATE
              │
              └── ZaloBotListenerAutoConfiguration:
                    @ConditionalOnClass(UpdateListenerContainer.class) → MATCH
                    @ConditionalOnBean(ZaloBotClient.class)            → just created above → MATCH
                    @ConditionalOnProperty("zalobot.listener.enabled", matchIfMissing=true) → MATCH
                    │
                    zaloBotContainerProperties() → @ConditionalOnMissingBean → CREATE
                    │                               (wires user's UpdateListener + ErrorHandler beans)
                    │
                    zaloBotListenerContainer()   → @ConditionalOnMissingBean(UpdateListenerContainer)
                    │                              @ConditionalOnBean(UpdateListener.class)
                    │                              → user defined UpdateListener bean? YES → CREATE
                    │                              → concurrency=2 → ConcurrentUpdateListenerContainer
                    │
                    zaloBotListenerContainerLifecycle() → @ConditionalOnBean(UpdateListenerContainer)
                                                         → container exists → CREATE
                                                         → SmartLifecycle.start() called by Spring
                                                         → container starts polling Zalo API
```

---

## 6. Design Patterns

| Pattern | Where Used | Source Reference | Why |
|---|---|---|---|
| **Factory Method** | `ZaloBotClient.botToken(token)` / `ZaloBotClient.builder()` | `ZaloBotClient.java:30-31` | Hides `DefaultZaloBotClientBuilder` from the user; provides a clean API entry point |
| **Strategy** | `ClientHttpRequestFactory` swappable in builder | `DefaultZaloBotClientBuilder.java:85-97` | JDK HttpClient vs OkHttp — selected at build time based on classpath |
| **Template Method** | `SpringBootCondition.matches()` calls abstract `getMatchOutcome()` | `SpringBootCondition.java:44,115` | `matches()` is `final` — handles logging, error handling, and reporting. Subclasses only implement the matching logic |
| **Back-Away** | `@ConditionalOnMissingBean` on every `@Bean` | `KafkaAutoConfiguration.java:98,104,121,127,138,154,161,176` | If user defines their own bean, auto-config backs away. Per-bean, not per-class — granular control |
| **Deferred Import** | `DeferredImportSelector` interface | `AutoConfigurationImportSelector.java:78` | User `@Configuration` classes are always processed first, so `@ConditionalOnMissingBean` can detect user beans |
| **Adapter** | `ZaloBotListenerContainerLifecycle` wraps `UpdateListenerContainer` as `SmartLifecycle` | `ZaloBotListenerContainerLifecycle.java` (new) | SDK container doesn't know about Spring lifecycle; adapter bridges the two |
| **Customizer** | `ZaloBotClientCustomizer` functional interface | `ZaloBotClientCustomizer.java` (new) | Allows multiple beans to customize the builder without replacing it. Same pattern as `RestClientCustomizer` in Spring Boot |

---

## 7. Architectural Decisions

### Why `proxyBeanMethods = false`?
**Source:** `AutoConfiguration.java:58` — `@Configuration(proxyBeanMethods = false)`

With `proxyBeanMethods = true` (default for `@Configuration`), Spring creates a CGLIB subclass to intercept `@Bean` method calls so that calling `beanA()` from `beanB()` returns the same singleton. This adds startup overhead (proxy creation) and memory cost. Auto-configuration classes rarely call one `@Bean` method from another, so `proxyBeanMethods = false` is safe and faster. The `@AutoConfiguration` annotation enforces this.

### Why `DeferredImportSelector` instead of regular `ImportSelector`?
**Source:** `AutoConfigurationImportSelector.java:78`

A regular `ImportSelector` would be processed immediately when the `@EnableAutoConfiguration` annotation is parsed. A `DeferredImportSelector` is processed **after all other `@Configuration` classes**. This is critical: it means user-defined beans are registered before auto-configuration runs, so `@ConditionalOnMissingBean` can detect them and back away.

### Why early filtering via `AutoConfigurationImportFilter`?
**Source:** `ConfigurationClassFilter` in `AutoConfigurationImportSelector.java:388-429`

Without early filtering, Spring would have to load all ~150 auto-configuration classes (plus their imports) just to evaluate `@ConditionalOnClass`. The early filter uses pre-computed annotation metadata (from `AutoConfigurationMetadataLoader`) to check class presence **without loading the auto-configuration class itself**. This eliminates ~100 candidates in milliseconds.

### Why `.imports` files replaced `spring.factories`?
**Since:** Spring Boot 2.7

The old `spring.factories` file mixed auto-configurations with other factory types and was a single file per JAR. The new `.imports` approach:
- Uses a dedicated file per annotation type (`AutoConfiguration.imports`)
- One class name per line (easier to read, merge-friendly in version control)
- Supports comments with `#`
- Loaded by `ImportCandidates.java:47` — `LOCATION = "META-INF/spring/%s.imports"`

### Why `@ConditionalOnMissingBean` per `@Bean`, not per class?
**Source:** `KafkaAutoConfiguration.java` — every `@Bean` method has its own `@ConditionalOnMissingBean`

Putting `@ConditionalOnMissingBean` on the class would mean: "if the user defines ANY Kafka bean, skip ALL auto-configuration." But a user might want to customize just the `KafkaTemplate` while keeping the auto-configured `ConsumerFactory`, `ProducerFactory`, and `KafkaAdmin`. Per-bean conditionals allow granular back-away.

---

## 8. Mapping Table — Simplified Concept to Real Source

| Concept | Real Class | File:Line | What It Does |
|---|---|---|---|
| "Find all auto-config candidates" | `ImportCandidates.load()` | `ImportCandidates.java:81` | Reads `.imports` files from all JARs |
| "File location pattern" | `ImportCandidates.LOCATION` | `ImportCandidates.java:47` | `META-INF/spring/%s.imports` |
| "The loading engine" | `AutoConfigurationImportSelector` | `AutoConfigurationImportSelector.java:78` | Implements `DeferredImportSelector` |
| "Main pipeline method" | `getAutoConfigurationEntry()` | `AutoConfigurationImportSelector.java:142` | Orchestrates all 6 phases |
| "Remove duplicates" | `removeDuplicates()` | `AutoConfigurationImportSelector.java:148,305-307` | `LinkedHashSet` |
| "Collect exclusions" | `getExclusions()` | `AutoConfigurationImportSelector.java:149,247-255` | From annotation + property |
| "Early class-presence filter" | `ConfigurationClassFilter.filter()` | `AutoConfigurationImportSelector.java:152,399-427` | Fast metadata-based check |
| "Sort by @Before/@After" | `AutoConfigurationGroup.sortAutoConfigurations()` | `AutoConfigurationImportSelector.java:502,520-527` | Topological sort |
| "The entry point annotation" | `@EnableAutoConfiguration` | `EnableAutoConfiguration.java:77` | `@Import(AutoConfigurationImportSelector.class)` |
| "Mark a class as auto-config" | `@AutoConfiguration` | `AutoConfiguration.java:55-61` | Meta-annotated with `@Configuration(proxyBeanMethods=false)` |
| "Class-presence condition (annotation)" | `@ConditionalOnClass` | `ConditionalOnClass.java:65` | `@Conditional(OnClassCondition.class)` |
| "Class-presence condition (impl)" | `OnClassCondition` | `OnClassCondition.java:45-46` | `@Order(HIGHEST_PRECEDENCE)`, extends `FilteringSpringBootCondition` |
| "Bean-absence condition (annotation)" | `@ConditionalOnMissingBean` | `ConditionalOnMissingBean.java:69` | `@Conditional(OnBeanCondition.class)` |
| "Bean-absence condition (impl)" | `OnBeanCondition` | `OnBeanCondition.java:88` | `@Order(LOWEST_PRECEDENCE)`, extends `FilteringSpringBootCondition` |
| "Property condition (annotation)" | `@ConditionalOnProperty` | `ConditionalOnProperty.java:97` | `@Conditional(OnPropertyCondition.class)` |
| "Property condition (impl)" | `OnPropertyCondition` | `OnPropertyCondition.java:52` | `@Order(HIGHEST_PRECEDENCE + 40)` |
| "Base condition (template method)" | `SpringBootCondition` | `SpringBootCondition.java:39,44,115` | `matches()` is final, calls abstract `getMatchOutcome()` |
| "Dual-role condition+filter" | `FilteringSpringBootCondition` | `FilteringSpringBootCondition.java:42-43` | Extends `SpringBootCondition`, implements `AutoConfigurationImportFilter` |
| "Bind properties to POJO" | `@ConfigurationProperties` | `ConfigurationProperties.java:52` | `@ConfigurationProperties("spring.kafka")` on `KafkaProperties` |
| "Register properties class" | `@EnableConfigurationProperties` | `EnableConfigurationProperties.java:40-41` | `@Import(EnableConfigurationPropertiesRegistrar.class)` |
| "Reference: Kafka auto-config" | `KafkaAutoConfiguration` | `KafkaAutoConfiguration.java:84-89` | Complete example with all patterns |
| "Reference: Kafka properties" | `KafkaProperties` | `KafkaProperties.java:64` | `@ConfigurationProperties("spring.kafka")` |
| "Reference: Kafka starter" | `build.gradle` | `spring-boot-starter-kafka/build.gradle:23-27` | Zero code — just `api(project(...))` |

---

## 9. What We Simplified Away

This document focuses on the core flow. The following are real parts of the system that we intentionally omit for clarity:

| Topic | What It Does | Why We Skip It |
|---|---|---|
| `AutoConfigurationMetadataLoader` | Annotation processor that pre-computes condition metadata at compile time into `META-INF/spring-autoconfigure-metadata.properties` | Internal optimization — users never interact with it |
| `ConditionEvaluationReport` | Collects all condition match/no-match results for `--debug` output | Debugging tool — not part of the loading flow |
| `AutoConfigurationReplacements` | Allows renaming auto-configuration classes across versions | Migration concern — not relevant for new starters |
| AOT / GraalVM hints | `@ImportRuntimeHints`, `RuntimeHintsRegistrar` | Native image optimization — orthogonal to the auto-config mechanism |
| `ConfigurationPropertiesBindingPostProcessor` | The actual machinery that binds YAML/properties to `@ConfigurationProperties` POJOs | Deep framework internals — we use it as a black box |
| `AutoConfigurationPackage` | Registers the base package for entity scanning | Tangential to auto-configuration loading |
| `spring.factories` (legacy) | Old registration mechanism, still partially supported | Deprecated since 2.7 — we only cover `.imports` |
| `SharedMetadataReaderFactoryContextInitializer` | Shares a `MetadataReaderFactory` for performance | Implementation optimization detail |
