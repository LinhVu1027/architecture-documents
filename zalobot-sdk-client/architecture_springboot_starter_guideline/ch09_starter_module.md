# Chapter 9: Building the ZaloBot Starter Module

> Starters have ZERO Java code. They are pure dependency aggregators.

---

## The Key Lesson

Look at the canonical Kafka starter:

```groovy
// starter/spring-boot-starter-kafka/build.gradle:17-27
plugins {
    id "org.springframework.boot.starter"
}

description = "Starter for using Apache Kafka"

dependencies {
    api(project(":starter:spring-boot-starter"))    // base Spring Boot
    api(project(":module:spring-boot-kafka"))        // auto-config + conditions
}
```

**Source:** `starter/spring-boot-starter-kafka/build.gradle`

That's it. No Java source files. No resources. Just a build file that declares dependencies. The starter is a **shopping list**, not a recipe book.

---

## The Three-Layer Architecture

```
┌─────────────────────────────────────────────────────────┐
│                       STARTER                            │
│            spring-boot-starter-zalobot                   │
│                                                          │
│  - ZERO Java code                                        │
│  - Pure dependency aggregator                            │
│  - What the end-user adds to their pom.xml              │
│                                                          │
│  Dependencies:                                           │
│    ├── spring-boot-starter (base Spring Boot)            │
│    ├── spring-boot-zalobot (auto-config module)          │
│    ├── zalobot-client (SDK client)                       │
│    └── zalobot-listener (SDK listener)                   │
└────────────────────────┬────────────────────────────────┘
                         │ depends on
                         ▼
┌─────────────────────────────────────────────────────────┐
│                       MODULE                             │
│               spring-boot-zalobot                        │
│                                                          │
│  - Auto-configuration classes                            │
│  - @ConfigurationProperties                              │
│  - .imports registration file                            │
│  - SDK dependencies are <optional>                       │
│                                                          │
│  Contains:                                               │
│    ├── ZaloBotProperties.java                            │
│    ├── ZaloBotClientCustomizer.java                      │
│    ├── ZaloBotClientAutoConfiguration.java               │
│    ├── ZaloBotListenerAutoConfiguration.java             │
│    ├── ZaloBotListenerContainerLifecycle.java            │
│    └── META-INF/spring/...AutoConfiguration.imports      │
└────────────────────────┬────────────────────────────────┘
                         │ depends on (optional)
                         ▼
┌─────────────────────────────────────────────────────────┐
│                      CORE SDK                            │
│            zalobot-client + zalobot-listener              │
│                                                          │
│  - Pure library code, no Spring dependency               │
│  - Can be used without Spring Boot                       │
│  - ZaloBotClient, ContainerProperties, etc.              │
└─────────────────────────────────────────────────────────┘
```

| Layer | Purpose | Java Code | Spring Dependency |
|---|---|---|---|
| **Starter** (`spring-boot-starter-zalobot`) | Dependency aggregator for end users | None | Transitive only |
| **Module** (`spring-boot-zalobot`) | Auto-configuration logic | Yes — 5 classes + 1 `.imports` file | `spring-boot-autoconfigure` (optional) |
| **Core SDK** (`zalobot-client`, `zalobot-listener`) | Library implementation | Yes — pure SDK | None |

---

## Module POM: `spring-boot-zalobot/pom.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>dev.linhvu</groupId>
        <artifactId>zalobot-sdk-java</artifactId>
        <version>0.0.1</version>
    </parent>

    <artifactId>spring-boot-zalobot</artifactId>

    <properties>
        <maven.compiler.source>25</maven.compiler.source>
        <maven.compiler.target>25</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

    <dependencies>
        <!-- SDK modules -->
        <dependency>
            <groupId>dev.linhvu</groupId>
            <artifactId>zalobot-client</artifactId>
            <version>${project.parent.version}</version>
        </dependency>
        <dependency>
            <groupId>dev.linhvu</groupId>
            <artifactId>zalobot-listener</artifactId>
            <version>${project.parent.version}</version>
            <optional>true</optional>
        </dependency>

        <!-- Spring Boot (optional — starter brings them transitively) -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-autoconfigure</artifactId>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot</artifactId>
            <optional>true</optional>
        </dependency>

        <!-- Test -->
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter</artifactId>
            <version>${junit-jupiter.version}</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-test-autoconfigure</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
</project>
```

### Why `<optional>true</optional>`?

| Dependency | Optional? | Reason |
|---|---|---|
| `zalobot-client` | No | Always needed — the client auto-config requires it |
| `zalobot-listener` | **Yes** | Listener auto-config only activates if this is on classpath (`@ConditionalOnClass`). Users who only want the client don't get the listener transitively |
| `spring-boot-autoconfigure` | **Yes** | The module compiles against it, but the starter (not the module) should be the one bringing Spring Boot transitively |
| `spring-boot` | **Yes** | Same reason — provides `@ConfigurationProperties`, `ImportCandidates`, etc. |

Optional dependencies are NOT transitive. They must be explicitly pulled in by something else (in our case, the starter POM).

---

## Starter POM: `spring-boot-starter-zalobot/pom.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>dev.linhvu</groupId>
        <artifactId>zalobot-sdk-java</artifactId>
        <version>0.0.1</version>
    </parent>

    <artifactId>spring-boot-starter-zalobot</artifactId>

    <properties>
        <maven.compiler.source>25</maven.compiler.source>
        <maven.compiler.target>25</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

    <dependencies>
        <!-- Base Spring Boot starter (logging, auto-config, core) -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>

        <!-- Our auto-configuration module -->
        <dependency>
            <groupId>dev.linhvu</groupId>
            <artifactId>spring-boot-zalobot</artifactId>
            <version>${project.parent.version}</version>
        </dependency>

        <!-- SDK client (not transitive from spring-boot-zalobot since it's compile scope) -->
        <dependency>
            <groupId>dev.linhvu</groupId>
            <artifactId>zalobot-client</artifactId>
            <version>${project.parent.version}</version>
        </dependency>

        <!-- SDK listener (optional in spring-boot-zalobot, explicit here to make it transitive) -->
        <dependency>
            <groupId>dev.linhvu</groupId>
            <artifactId>zalobot-listener</artifactId>
            <version>${project.parent.version}</version>
        </dependency>
    </dependencies>
</project>
```

### Why `zalobot-listener` is Listed Explicitly

In `spring-boot-zalobot/pom.xml`, `zalobot-listener` is `<optional>true</optional>`. Optional dependencies are NOT transitive — they don't flow through to consumers. The starter must explicitly list it to make it available to end users.

This is the exact same pattern Spring Boot uses: `spring-boot-kafka` marks Kafka libraries as optional, and `spring-boot-starter-kafka` re-declares them to make them transitive.

---

## Root POM Changes

The root `pom.xml` needs two changes:

### 1. Add Spring Boot BOM to `<dependencyManagement>`

```xml
<properties>
    <!-- ... existing properties ... -->
    <spring-boot.version>4.0.2</spring-boot.version>
</properties>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-dependencies</artifactId>
            <version>${spring-boot.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

This imports the Spring Boot BOM, so child modules can reference `spring-boot-autoconfigure`, `spring-boot-starter`, etc. without specifying versions.

### 2. Fix Module Ordering

```xml
<modules>
    <module>zalobot-core</module>
    <module>zalobot-client</module>
    <module>zalobot-listener</module>
    <module>spring-boot-zalobot</module>          <!-- builds after SDK modules -->
    <module>spring-boot-starter-zalobot</module>  <!-- builds last -->
</modules>
```

`spring-boot-zalobot` depends on `zalobot-client` and `zalobot-listener`, so it must build after them. `spring-boot-starter-zalobot` depends on `spring-boot-zalobot`, so it builds last.

---

## Dependency Flow Diagram

```
End User's pom.xml
  └── spring-boot-starter-zalobot
        │
        ├── spring-boot-starter ─────────────────────┐
        │     ├── spring-boot                         │ Spring Boot
        │     ├── spring-boot-autoconfigure           │ framework
        │     ├── spring-boot-starter-logging         │
        │     └── jakarta.annotation-api              │
        │                                             │
        ├── spring-boot-zalobot ─────────────────────┤
        │     ├── (spring-boot-autoconfigure)  [opt]  │ NOT transitive
        │     ├── (spring-boot)                [opt]  │ NOT transitive
        │     ├── zalobot-client               [dep]  │
        │     └── (zalobot-listener)           [opt]  │ NOT transitive
        │                                             │
        ├── zalobot-client ──────────────────────────┤
        │     ├── zalobot-core                        │ SDK
        │     └── jackson-databind                    │
        │                                             │
        └── zalobot-listener ────────────────────────┘
              ├── zalobot-core
              └── zalobot-client
```

The end user's effective classpath includes everything needed — Spring Boot, auto-configuration, SDK client, and SDK listener. Dependencies marked `[opt]` in the module are NOT transitive, but the starter re-declares them to make them available.

---

## Package Structure Summary

```
zalobot-sdk-java/
  ├── zalobot-core/                          ← SDK models (GetUpdates, SendMessage, etc.)
  ├── zalobot-client/                        ← SDK client (ZaloBotClient, Builder)
  ├── zalobot-listener/                      ← SDK listener (Container, UpdateListener)
  │
  ├── spring-boot-zalobot/                   ← AUTO-CONFIG MODULE
  │     ├── pom.xml
  │     └── src/main/
  │           ├── java/dev/linhvu/zalobot/autoconfigure/
  │           │     ├── ZaloBotProperties.java
  │           │     ├── ZaloBotClientCustomizer.java
  │           │     ├── ZaloBotClientAutoConfiguration.java
  │           │     ├── ZaloBotListenerAutoConfiguration.java
  │           │     └── ZaloBotListenerContainerLifecycle.java
  │           └── resources/META-INF/spring/
  │                 └── org.springframework.boot.autoconfigure.AutoConfiguration.imports
  │
  └── spring-boot-starter-zalobot/           ← STARTER (zero Java code)
        └── pom.xml
```

---

## What's Next?

The module structure is complete. Chapter 10 covers testing and verification — proving that everything works end-to-end.

```
Chapter 8  ──► "Build it: auto-config"
Chapter 9  ←── YOU ARE HERE: "Build it: POM/dependencies"
Chapter 10 ──► "Verify it all works"
```
