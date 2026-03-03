# Chapter 5: The Discovery Mechanism — `.imports` Files

> How does Spring Boot find auto-configuration classes without component scanning? Through a classpath-wide file convention.

---

## The Core Class: `ImportCandidates`

```java
// ImportCandidates.java:45-47
public final class ImportCandidates implements Iterable<String> {

    private static final String LOCATION = "META-INF/spring/%s.imports";    // ← the file pattern
    private static final String COMMENT_START = "#";
```

**Source:** `core/spring-boot/.../ImportCandidates.java:47`

The location pattern is `META-INF/spring/<fully-qualified-annotation-name>.imports`. For `@AutoConfiguration`, this resolves to:

```
META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
```

### The `load()` Method

```java
// ImportCandidates.java:81-92
public static ImportCandidates load(Class<?> annotation, @Nullable ClassLoader classLoader) {
    Assert.notNull(annotation, "'annotation' must not be null");
    ClassLoader classLoaderToUse = decideClassloader(classLoader);
    String location = String.format(LOCATION, annotation.getName());        // ← format path
    Enumeration<URL> urls = findUrlsInClasspath(classLoaderToUse, location); // ← multi-JAR!
    List<String> importCandidates = new ArrayList<>();
    while (urls.hasMoreElements()) {
        URL url = urls.nextElement();
        importCandidates.addAll(readCandidateConfigurations(url));          // ← read each file
    }
    return new ImportCandidates(importCandidates);
}
```

**Source:** `core/spring-boot/.../ImportCandidates.java:81-92`

The key line is `classLoaderToUse.getResources(location)` (line 103) — this returns URLs from **every JAR** on the classpath that contains this file. This is how multi-JAR discovery works.

---

## Multi-JAR Discovery

When your application starts, the classpath might contain dozens of JARs, each with its own `.imports` file:

```
classpath
  ├── spring-boot-kafka.jar
  │     └── META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
  │           → org.springframework.boot.kafka.autoconfigure.KafkaAutoConfiguration
  │           → org.springframework.boot.kafka.autoconfigure.metrics.KafkaMetricsAutoConfiguration
  │
  ├── spring-boot-restclient.jar
  │     └── META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
  │           → org.springframework.boot.restclient.autoconfigure.RestClientAutoConfiguration
  │           → org.springframework.boot.restclient.autoconfigure.RestClientObservationAutoConfiguration
  │           → org.springframework.boot.restclient.autoconfigure.RestTemplateAutoConfiguration
  │           → org.springframework.boot.restclient.autoconfigure.RestTemplateObservationAutoConfiguration
  │           → org.springframework.boot.restclient.autoconfigure.service.HttpServiceClientAutoConfiguration
  │
  ├── spring-boot-zalobot.jar        ← OUR MODULE
  │     └── META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
  │           → dev.linhvu.zalobot.boot.autoconfigure.ZaloBotClientAutoConfiguration
  │           → dev.linhvu.zalobot.autoconfigure.ZaloBotListenerAutoConfiguration
  │
  └── ... (~30 more modules with their own .imports files)
```

`ImportCandidates.load()` collects ALL entries from ALL JARs into a single list. This is why adding a starter dependency is enough — its module JAR contributes entries to the discovery mechanism automatically.

---

## Real Examples

### Kafka `.imports` File (2 entries)

**File:** `module/spring-boot-kafka/src/main/resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`

```
org.springframework.boot.kafka.autoconfigure.KafkaAutoConfiguration
org.springframework.boot.kafka.autoconfigure.metrics.KafkaMetricsAutoConfiguration
```

### RestClient `.imports` File (5 entries)

**File:** `module/spring-boot-restclient/src/main/resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`

```
org.springframework.boot.restclient.autoconfigure.RestClientAutoConfiguration
org.springframework.boot.restclient.autoconfigure.RestClientObservationAutoConfiguration
org.springframework.boot.restclient.autoconfigure.RestTemplateAutoConfiguration
org.springframework.boot.restclient.autoconfigure.RestTemplateObservationAutoConfiguration
org.springframework.boot.restclient.autoconfigure.service.HttpServiceClientAutoConfiguration
```

---

## Building: The ZaloBot `.imports` File

**File:** `spring-boot-zalobot/src/main/resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`

```
dev.linhvu.zalobot.boot.autoconfigure.ZaloBotClientAutoConfiguration
dev.linhvu.zalobot.autoconfigure.ZaloBotListenerAutoConfiguration
```

That's it. Two lines. Each line is a fully-qualified class name of an `@AutoConfiguration` class.

### File Format Rules

- One fully-qualified class name per line
- Lines starting with `#` are comments (stripped by `ImportCandidates.readCandidateConfigurations()`)
- Blank lines are ignored
- No special ordering needed in the file (sorting happens in Phase 5 via `@AutoConfigureBefore`/`@AutoConfigureAfter`)

---

## How Discovery Connects to the Pipeline

```
@SpringBootApplication
  └── @EnableAutoConfiguration
        └── @Import(AutoConfigurationImportSelector.class)
              │
              ▼
AutoConfigurationImportSelector.getCandidateConfigurations()
  at AutoConfigurationImportSelector.java:200-210
              │
              ▼ calls
ImportCandidates.load(AutoConfiguration.class, classLoader)
  at ImportCandidates.java:81
              │
              ▼ reads from every JAR
META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
              │
              ▼ collects
["KafkaAutoConfiguration", "RestClientAutoConfiguration", ...,
 "ZaloBotClientAutoConfiguration", "ZaloBotListenerAutoConfiguration", ...]
              │
              ▼ continues to Phase 2 (dedup), Phase 3 (exclusion), etc.
```

---

## Why `.imports` Files Instead of Component Scanning?

You might wonder: why not just `@ComponentScan` for `@AutoConfiguration` classes?

| Component Scanning | `.imports` Files |
|---|---|
| Scans packages recursively | Reads specific file paths |
| Would scan the entire classpath | Only reads from JARs that explicitly contribute |
| No control over which classes are found | Each module declares exactly which classes to register |
| No way to exclude without extra config | Users can exclude via `@SpringBootApplication(exclude=...)` |
| Slower for large classpaths | Fast — just reads text files |

The `.imports` mechanism is an **explicit opt-in**. A module must deliberately create the file and list its auto-configuration classes. This is safer and more performant than classpath scanning.

---

## What's Next?

We now understand all the individual pieces:
- Chapter 2: Properties binding (`@ConfigurationProperties`)
- Chapter 3: Auto-configuration classes (`@AutoConfiguration`)
- Chapter 4: Conditional annotations (`@ConditionalOnClass`, etc.)
- Chapter 5: Discovery mechanism (`.imports` files)

In Chapter 6, we assemble the complete picture — tracing from `@SpringBootApplication` through all 6 phases to beans being created.

```
Chapter 1  ──► "Why do we need starters?"
Chapter 2  ──► "Externalize config"
Chapter 3  ──► "Who creates beans?"
Chapter 4  ──► "When to create?"
Chapter 5  ←── YOU ARE HERE: "How does Spring find us?"
Chapter 6  ──► "Complete loading pipeline"
```
