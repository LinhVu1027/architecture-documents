# Chapter 6: Configuring Exclusions

> **Type:** IN-PROJECT — changes to POM files

---

## Why Exclude?

Not everything in your project should be analyzed or measured for coverage:

- **The starter module** has no source code — only dependency declarations
- **Model/DTO classes** are often simple data holders with no logic to test
- **Generated code** (if any) shouldn't count against your quality metrics

SonarQube provides three levels of exclusion:

```
┌─────────────────────────────────────────────────────────────┐
│                                                              │
│  sonar.skip = true          Skip the ENTIRE module           │
│                             (no analysis at all)             │
│                                                              │
│  sonar.exclusions           Skip specific files from         │
│                             analysis (bugs, smells)          │
│                                                              │
│  sonar.coverage.exclusions  Skip specific files from         │
│                             coverage measurement only        │
│                             (still analyzed for bugs)        │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Exclusion 1: Skip the Starter Module Entirely

The `zalobot-spring-boot-starter` module contains no Java source code. It only declares dependencies. SonarQube analyzing it would produce noise.

Add to `zalobot-spring-boot-starter/pom.xml`:

```xml
<!-- Add inside <properties> -->
<sonar.skip>true</sonar.skip>
```

Combined with the `jacoco.skip` from Chapter 4, the full `<properties>` block becomes:

```xml
<properties>
    <maven.compiler.source>25</maven.compiler.source>
    <maven.compiler.target>25</maven.compiler.target>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <jacoco.skip>true</jacoco.skip>
    <sonar.skip>true</sonar.skip>
</properties>
```

---

## Exclusion 2: Source Exclusions (Optional)

If you want to exclude specific files from static analysis entirely, add this to the parent POM's `<properties>`:

```xml
<!-- Example: exclude a specific generated file -->
<sonar.exclusions>
    **/generated/**
</sonar.exclusions>
```

**For zalobot-sdk-java:** You likely don't need source exclusions right now. All 40 production files are hand-written and should be analyzed.

### Pattern syntax:

| Pattern | Matches |
|---|---|
| `**/*.java` | All Java files in all directories |
| `**/model/**` | All files in any `model` directory |
| `**/generated/**` | All files in any `generated` directory |
| `src/main/java/dev/linhvu/zalobot/core/model/*.java` | Specific model classes |

Multiple patterns are comma-separated:
```xml
<sonar.exclusions>**/generated/**,**/model/dto/**</sonar.exclusions>
```

---

## Exclusion 3: Coverage Exclusions (Optional)

Coverage exclusions are different from source exclusions. A file excluded from coverage is still analyzed for bugs and code smells — it just isn't counted in the coverage percentage.

This is useful for classes that are hard to unit test or don't benefit from coverage measurement:

```xml
<!-- Example: don't measure coverage for simple model classes -->
<sonar.coverage.exclusions>
    **/model/**,
    **/ZaloBotUrl.java
</sonar.coverage.exclusions>
```

### Should you exclude model classes?

| Argument for excluding | Argument against excluding |
|---|---|
| Records/DTOs are trivial, testing them inflates coverage artificially | Even simple records can have bugs in equals/hashCode |
| Keeps coverage metric focused on "real logic" | Excluding creates a slippery slope — "this is too simple to test" |
| Reduces noise in the dashboard | SonarQube already distinguishes line vs branch coverage |

**Recommendation for zalobot-sdk-java:** Start with **no coverage exclusions**. Your model classes in `zalobot-core` (like `SendMessage`, `GetUpdates`, etc.) already have tests. Let SonarQube show you the real coverage picture first, then decide if exclusions make sense.

---

## Understanding What Each Module Contains

To make informed exclusion decisions, here's what each module has:

```
zalobot-core (10 production files, 7 test files)
├── model/
│   ├── GetUpdates.java, GetUpdatesResult.java
│   ├── GetMe.java, GetMeResult.java
│   ├── SendMessage.java, SendMessageResult.java
│   ├── SendPhoto.java, SendSticker.java, SendChatAction.java
│   └── ZaloApiResponse.java
└── (all model classes — tests exist for most)

zalobot-client (16 production files, 0 test files)
├── ZaloBotClient.java, DefaultZaloBotClient.java
├── DefaultZaloBotClientBuilder.java, ZaloBotUrl.java
├── util/Assert.java, util/ClassUtils.java
├── http/ClientHttpRequest.java, ClientHttpResponse.java
├── http/ClientHttpRequestFactory.java, HttpMethod.java
├── http/jdk/JdkClientHttp{Request,Response,RequestFactory}.java
└── http/okhhtp3/OkClientHttp{Request,Response,RequestFactory}.java

zalobot-listener (9 production files, 0 test files)
├── UpdateListener.java, UpdateListenerContainer.java
├── AbstractUpdateListenerContainer.java
├── ZaloUpdateListenerContainer.java
├── ConcurrentUpdateListenerContainer.java
├── ContainerProperties.java, ExponentialBackOff.java
├── ErrorHandler.java, LoggingErrorHandler.java

zalobot-spring-boot (5 production files, 3 test files)
├── ZaloBotClientCustomizer.java
├── autoconfigure/ZaloBotProperties.java
├── autoconfigure/ZaloBotClientAutoConfiguration.java
├── autoconfigure/ZaloBotListenerAutoConfiguration.java
└── autoconfigure/ZaloBotListenerContainerLifecycle.java

zalobot-spring-boot-starter (0 production files) ← SKIPPED
```

---

## Summary of Changes

| File | Change |
|---|---|
| `zalobot-spring-boot-starter/pom.xml` | Add `<sonar.skip>true</sonar.skip>` property |
| `pom.xml` (parent) — optional | Add `<sonar.exclusions>` if needed |
| `pom.xml` (parent) — optional | Add `<sonar.coverage.exclusions>` if needed |

---

## Next: Chapter 7

We'll discuss whether to wrap SonarQube analysis in a Maven profile for opt-in execution.
