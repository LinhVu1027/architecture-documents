# Chapter 6: The Full Loading Pipeline

> Everything connects here. From `@SpringBootApplication` to beans registered — the complete sequence.

---

## The Entry Point: `@EnableAutoConfiguration`

```java
// EnableAutoConfiguration.java:72-78
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)    // ← THIS triggers everything
public @interface EnableAutoConfiguration { ... }
```

**Source:** `core/spring-boot-autoconfigure/.../EnableAutoConfiguration.java:77`

`@SpringBootApplication` is meta-annotated with `@EnableAutoConfiguration`, which in turn has `@Import(AutoConfigurationImportSelector.class)`. This single `@Import` triggers the entire auto-configuration pipeline.

---

## Why `DeferredImportSelector`?

```java
// AutoConfigurationImportSelector.java:78-79
public class AutoConfigurationImportSelector implements DeferredImportSelector,
        BeanClassLoaderAware, ResourceLoaderAware, BeanFactoryAware, EnvironmentAware, Ordered {
```

`DeferredImportSelector` (a Spring Framework interface) is processed **after all regular `@Configuration` classes**. This is the critical guarantee:

```
Processing Order:
  1. User's @Configuration classes        ← user beans registered FIRST
  2. DeferredImportSelector               ← auto-config runs SECOND
       └── AutoConfigurationImportSelector
             └── evaluates @ConditionalOnMissingBean
                   └── can see user beans → backs away if needed
```

If `AutoConfigurationImportSelector` were a regular `ImportSelector`, it would run in arbitrary order — sometimes before user beans, sometimes after. `@ConditionalOnMissingBean` wouldn't work reliably.

---

## The Main Pipeline: `getAutoConfigurationEntry()`

```java
// AutoConfigurationImportSelector.java:142-155
protected AutoConfigurationEntry getAutoConfigurationEntry(AnnotationMetadata annotationMetadata) {
    if (!isEnabled(annotationMetadata)) {                              // check kill switch
        return EMPTY_ENTRY;
    }
    AnnotationAttributes attributes = getAttributes(annotationMetadata);
    List<String> configurations = getCandidateConfigurations(          // Phase 1: DISCOVERY
            annotationMetadata, attributes);
    configurations = removeDuplicates(configurations);                 // Phase 2: DEDUP
    Set<String> exclusions = getExclusions(                            // Phase 3: EXCLUSION
            annotationMetadata, attributes);
    checkExcludedClasses(configurations, exclusions);
    configurations.removeAll(exclusions);
    configurations = getConfigurationClassFilter()                     // Phase 4: EARLY FILTER
            .filter(configurations);
    fireAutoConfigurationImportEvents(configurations, exclusions);     // notify listeners
    return new AutoConfigurationEntry(configurations, exclusions);
}
```

**Source:** `AutoConfigurationImportSelector.java:142-155`

---

## Full Sequence Diagram

```
@SpringBootApplication                  Spring's ConfigurationClassParser
  │                                       │
  │ @EnableAutoConfiguration              │
  │   @Import(AutoConfigurationImport     │
  │           Selector.class)             │
  │                                       │
  ▼                                       ▼
╔═══════════════════════════════════════════════════════════════╗
║  1. Process all user @Configuration classes                   ║
║     └── User beans registered in BeanFactory                  ║
╚═══════════════════════════════════════════════════════════════╝
                          │
                          ▼ THEN (deferred)
╔═══════════════════════════════════════════════════════════════╗
║  2. AutoConfigurationImportSelector.getAutoConfigurationEntry ║
║                                                               ║
║  ┌─────────────────────────────────────────────────────────┐  ║
║  │ Phase 1: DISCOVERY                                      │  ║
║  │ ImportCandidates.load(AutoConfiguration.class, cl)      │  ║
║  │   reads META-INF/spring/...AutoConfiguration.imports    │  ║
║  │   from EVERY JAR on classpath                           │  ║
║  │   → ~150 candidate class names                          │  ║
║  └─────────────────────────────────────┬───────────────────┘  ║
║                                        ▼                      ║
║  ┌─────────────────────────────────────────────────────────┐  ║
║  │ Phase 2: DEDUP                                          │  ║
║  │ removeDuplicates() → LinkedHashSet                      │  ║
║  │   → ~150 unique candidates                              │  ║
║  └─────────────────────────────────────┬───────────────────┘  ║
║                                        ▼                      ║
║  ┌─────────────────────────────────────────────────────────┐  ║
║  │ Phase 3: EXCLUSION                                      │  ║
║  │ getExclusions() collects from:                          │  ║
║  │   - @EnableAutoConfiguration(exclude=...)               │  ║
║  │   - @SpringBootApplication(exclude=...)                 │  ║
║  │   - spring.autoconfigure.exclude property               │  ║
║  │ configurations.removeAll(exclusions)                    │  ║
║  │   → ~150 candidates (few exclusions typically)          │  ║
║  └─────────────────────────────────────┬───────────────────┘  ║
║                                        ▼                      ║
║  ┌─────────────────────────────────────────────────────────┐  ║
║  │ Phase 4: EARLY FILTERING                                │  ║
║  │ ConfigurationClassFilter.filter(configurations)         │  ║
║  │                                                         │  ║
║  │ For each filter in [OnClassCondition, OnBeanCondition]: │  ║
║  │   filter.match(candidates, metadataCache)               │  ║
║  │     └── checks annotation metadata WITHOUT loading      │  ║
║  │         the auto-configuration classes                  │  ║
║  │     └── OnClassCondition: "KafkaAutoConfiguration       │  ║
║  │         requires KafkaTemplate — is it present?"        │  ║
║  │                                                         │  ║
║  │   → ~50 candidates survive (100 eliminated!)            │  ║
║  └─────────────────────────────────────┬───────────────────┘  ║
╚════════════════════════════════════════╪═══════════════════════╝
                                         ▼
╔═══════════════════════════════════════════════════════════════╗
║  3. AutoConfigurationGroup.selectImports()                    ║
║                                                               ║
║  ┌─────────────────────────────────────────────────────────┐  ║
║  │ Phase 5: SORTING                                        │  ║
║  │ AutoConfigurationSorter.getInPriorityOrder()            │  ║
║  │                                                         │  ║
║  │ Reads @AutoConfiguration(before=..., after=...)         │  ║
║  │ Topological sort:                                       │  ║
║  │   ZaloBotClientAutoConfiguration                        │  ║
║  │   ZaloBotListenerAutoConfiguration (after=Client)       │  ║
║  │                                                         │  ║
║  │ → ordered list of ~50 candidates                        │  ║
║  └─────────────────────────────────────┬───────────────────┘  ║
╚════════════════════════════════════════╪═══════════════════════╝
                                         ▼
╔═══════════════════════════════════════════════════════════════╗
║  4. ConfigurationClassParser processes each candidate         ║
║                                                               ║
║  ┌─────────────────────────────────────────────────────────┐  ║
║  │ Phase 6: FULL CONDITION EVALUATION                      │  ║
║  │                                                         │  ║
║  │ For each auto-config class:                             │  ║
║  │   SpringBootCondition.matches() evaluates               │  ║
║  │     class-level conditions:                             │  ║
║  │       @ConditionalOnClass                               │  ║
║  │       @ConditionalOnProperty                            │  ║
║  │                                                         │  ║
║  │   If class passes → for each @Bean method:              │  ║
║  │     SpringBootCondition.matches() evaluates             │  ║
║  │       method-level conditions:                          │  ║
║  │         @ConditionalOnMissingBean                       │  ║
║  │         @ConditionalOnBean                              │  ║
║  │         @ConditionalOnProperty                          │  ║
║  │                                                         │  ║
║  │   If method passes → register bean definition           │  ║
║  │                                                         │  ║
║  │ → actual beans created in BeanFactory                   │  ║
║  └─────────────────────────────────────────────────────────┘  ║
╚═══════════════════════════════════════════════════════════════╝
```

---

## Complete Source Mapping

| Pipeline Step | Method/Class | Source File:Line |
|---|---|---|
| Entry point | `@EnableAutoConfiguration` | `EnableAutoConfiguration.java:77` |
| Import trigger | `@Import(AutoConfigurationImportSelector.class)` | `EnableAutoConfiguration.java:77` |
| Selector class | `AutoConfigurationImportSelector` | `AutoConfigurationImportSelector.java:78` |
| Main method | `getAutoConfigurationEntry()` | `AutoConfigurationImportSelector.java:142` |
| Kill switch check | `isEnabled()` | `AutoConfigurationImportSelector.java:162-167` |
| Phase 1: Discovery | `getCandidateConfigurations()` | `AutoConfigurationImportSelector.java:200-210` |
| Phase 1: File loading | `ImportCandidates.load()` | `ImportCandidates.java:81-92` |
| Phase 1: File location | `LOCATION = "META-INF/spring/%s.imports"` | `ImportCandidates.java:47` |
| Phase 1: Multi-JAR scan | `classLoader.getResources(location)` | `ImportCandidates.java:103` |
| Phase 1: Line reading | `readCandidateConfigurations()` | `ImportCandidates.java:110-128` |
| Phase 2: Dedup | `removeDuplicates()` | `AutoConfigurationImportSelector.java:148,305-307` |
| Phase 3: Exclusion collection | `getExclusions()` | `AutoConfigurationImportSelector.java:149,247-255` |
| Phase 3: Exclusion removal | `configurations.removeAll(exclusions)` | `AutoConfigurationImportSelector.java:151` |
| Phase 4: Filter creation | `getConfigurationClassFilter()` | `AutoConfigurationImportSelector.java:282-293` |
| Phase 4: Filter execution | `ConfigurationClassFilter.filter()` | `AutoConfigurationImportSelector.java:399-427` |
| Phase 4: Batch matching | `FilteringSpringBootCondition.match()` | `FilteringSpringBootCondition.java:52-69` |
| Phase 4: Class-check impl | `OnClassCondition.getOutcomes()` | `OnClassCondition.java:49-62` |
| Phase 5: Sort delegation | `AutoConfigurationGroup.selectImports()` | `AutoConfigurationImportSelector.java:489-505` |
| Phase 5: Sort execution | `sortAutoConfigurations()` | `AutoConfigurationImportSelector.java:520-527` |
| Phase 6: Condition base | `SpringBootCondition.matches()` | `SpringBootCondition.java:44-62` |
| Phase 6: Abstract dispatch | `getMatchOutcome()` | `SpringBootCondition.java:115` |
| Phase 6: Class condition | `OnClassCondition.getMatchOutcome()` | `OnClassCondition.java:87-115` |
| Phase 6: Bean condition | `OnBeanCondition.getMatchOutcome()` | `OnBeanCondition.java:88` |
| Phase 6: Property condition | `OnPropertyCondition.getMatchOutcome()` | `OnPropertyCondition.java:56-69` |

---

## The Kill Switch

There's a global kill switch for auto-configuration:

```java
// AutoConfigurationImportSelector.java:162-167
protected boolean isEnabled(AnnotationMetadata metadata) {
    if (getClass() == AutoConfigurationImportSelector.class) {
        return getEnvironment().getProperty(
                EnableAutoConfiguration.ENABLED_OVERRIDE_PROPERTY,    // "spring.boot.enableautoconfiguration"
                Boolean.class, true);
    }
    return true;
}
```

Setting `spring.boot.enableautoconfiguration=false` disables ALL auto-configuration. This is rarely used in practice but exists as an escape hatch.

---

## What Happens in Each Phase — By the Numbers

For a typical Spring Boot web app with Kafka and ZaloBot starters:

| Phase | Input | Output | Eliminated | Time |
|---|---|---|---|---|
| 1. Discovery | 0 | ~150 | — | ~5ms |
| 2. Dedup | ~150 | ~150 | ~0 | <1ms |
| 3. Exclusion | ~150 | ~148 | ~2 | <1ms |
| 4. Early filter | ~148 | ~50 | ~98 | ~10ms |
| 5. Sorting | ~50 | ~50 | 0 | ~2ms |
| 6. Full eval | ~50 | ~25 beans | ~25 classes skipped | ~50ms |

Phase 4 is the key optimization: it eliminates two-thirds of candidates without loading any classes.

---

## What's Next?

We've completed the conceptual foundation (Chapters 1-6). Now we build:

```
Chapters 1-6: Conceptual ← COMPLETE
  Chapter 1: The Problem
  Chapter 2: @ConfigurationProperties
  Chapter 3: @AutoConfiguration
  Chapter 4: Conditional Annotations
  Chapter 5: .imports Discovery
  Chapter 6: Full Pipeline ← YOU ARE HERE

Chapters 7-10: Build Phase → STARTS NEXT
  Chapter 7:  Build ZaloBotProperties (complete, compilable)
  Chapter 8:  Build ZaloBot Auto-Configuration classes
  Chapter 9:  Build the Starter Module (POM/dependencies)
  Chapter 10: Testing and Verification
```
