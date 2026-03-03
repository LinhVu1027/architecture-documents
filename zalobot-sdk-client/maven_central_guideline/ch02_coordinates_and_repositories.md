# Chapter 2: Coordinates and Repositories — How Maven Finds Artifacts

> When you write `<groupId>org.springframework.boot</groupId>`, how does Maven know where to download the JAR from? The answer is a coordinate system that maps identifiers to URLs.

---

## Maven Coordinates

Every Maven artifact is uniquely identified by **five** coordinates:

```
groupId    : artifactId    : version  : packaging : classifier
─────────    ──────────      ───────    ─────────   ──────────
dev.linhvu : zalobot-client : 0.0.1   : jar       : (none)
dev.linhvu : zalobot-client : 0.0.1   : jar       : sources
dev.linhvu : zalobot-client : 0.0.1   : jar       : javadoc
```

In a `<dependency>` block, you typically specify only three (groupId, artifactId, version). Packaging defaults to `jar` and classifier is empty unless you're requesting sources or javadoc.

> **Insight: Coordinates → URL**
>
> Maven Central uses a deterministic URL structure. The coordinates map to a path:
> ```
> groupId (dots → slashes) / artifactId / version / artifactId-version[-classifier].packaging
> ```
> So `dev.linhvu:zalobot-client:0.0.1` becomes:
> ```
> https://repo.maven.apache.org/maven2/dev/linhvu/zalobot-client/0.0.1/zalobot-client-0.0.1.jar
> ```
> This is not magic — it's string manipulation. Maven converts dots to slashes in the groupId, appends the artifactId as a directory, then the version, and finally constructs the filename.

---

## Our Project's Coordinates

| Module | groupId | artifactId | version | packaging |
|---|---|---|---|---|
| Parent POM | `dev.linhvu` | `zalobot-sdk-java` | `0.0.1` | `pom` |
| Core | `dev.linhvu` | `zalobot-core` | `0.0.1` | `jar` |
| Client | `dev.linhvu` | `zalobot-client` | `0.0.1` | `jar` |
| Listener | `dev.linhvu` | `zalobot-listener` | `0.0.1` | `jar` |
| Spring Boot Auto-Config | `dev.linhvu` | `zalobot-spring-boot` | `0.0.1` | `jar` |
| Spring Boot Starter | `dev.linhvu` | `zalobot-spring-boot-starter` | `0.0.1` | `jar` |

All 6 modules (parent + 5 children) share the same `groupId` and `version`. The `artifactId` is the only thing that varies.

> **Insight: The Parent POM Gets Published Too**
>
> Many developers assume only JARs go to Maven Central. But the **parent POM** (`zalobot-sdk-java-0.0.1.pom`) is also published. Why?
>
> When Maven resolves `zalobot-client-0.0.1.pom`, it finds `<parent><artifactId>zalobot-sdk-java</artifactId></parent>`. Maven then needs to download the parent POM to resolve inherited properties like `<dependencyManagement>` versions. If the parent POM isn't on Maven Central, child modules cannot be resolved properly.
>
> This is why a parent POM with `<packaging>pom</packaging>` is also deployed — it's the root of the inheritance tree that consumers need.

---

## The Repository Hierarchy

Maven resolves dependencies by searching repositories in order:

```
┌─────────────────────────────────────────────────────────────────┐
│ STEP 1: LOCAL REPOSITORY                                        │
│                                                                 │
│ Location: ~/.m2/repository/                                     │
│ Populated by: mvn install, or cached from previous downloads    │
│ Example: ~/.m2/repository/dev/linhvu/zalobot-client/0.0.1/      │
│          └── zalobot-client-0.0.1.jar                           │
│                                                                 │
│ Found? → Use it. Done.                                          │
│ Not found? → Continue to Step 2.                                │
└──────────────────────────────┬──────────────────────────────────┘
                               │ not found
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│ STEP 2: CONFIGURED REMOTE REPOSITORIES                          │
│                                                                 │
│ Defined in: POM <repositories> or settings.xml <mirrors>        │
│ Examples: company Nexus, GitHub Packages, JitPack               │
│                                                                 │
│ Found? → Download to local repo cache. Done.                    │
│ Not found? → Continue to Step 3.                                │
└──────────────────────────────┬──────────────────────────────────┘
                               │ not found
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│ STEP 3: MAVEN CENTRAL (always searched last, implicitly)        │
│                                                                 │
│ URL: https://repo.maven.apache.org/maven2/                      │
│ The default repository — no configuration needed                │
│ Contains: ~15 million artifacts (as of 2025)                    │
│                                                                 │
│ Found? → Download to local repo cache. Done.                    │
│ Not found? → ERROR: "Could not find artifact"                   │
└─────────────────────────────────────────────────────────────────┘
```

Our goal: get all 6 modules into Step 3 — Maven Central.

---

## What Lives on Maven Central for a Published Artifact

When you successfully publish `zalobot-client:0.0.1`, Maven Central hosts these files:

```
/maven2/dev/linhvu/zalobot-client/0.0.1/
│
├── zalobot-client-0.0.1.jar               ← compiled classes (the artifact)
├── zalobot-client-0.0.1.jar.asc           ← GPG signature of the JAR
├── zalobot-client-0.0.1.jar.md5           ← MD5 checksum
├── zalobot-client-0.0.1.jar.sha1          ← SHA-1 checksum
│
├── zalobot-client-0.0.1.pom               ← project descriptor (dependencies, etc.)
├── zalobot-client-0.0.1.pom.asc           ← GPG signature of the POM
├── zalobot-client-0.0.1.pom.md5           ← MD5 checksum
├── zalobot-client-0.0.1.pom.sha1          ← SHA-1 checksum
│
├── zalobot-client-0.0.1-sources.jar       ← source code for IDE navigation
├── zalobot-client-0.0.1-sources.jar.asc   ← GPG signature
├── zalobot-client-0.0.1-sources.jar.md5   ← MD5 checksum
├── zalobot-client-0.0.1-sources.jar.sha1  ← SHA-1 checksum
│
├── zalobot-client-0.0.1-javadoc.jar       ← API documentation
├── zalobot-client-0.0.1-javadoc.jar.asc   ← GPG signature
├── zalobot-client-0.0.1-javadoc.jar.md5   ← MD5 checksum
└── zalobot-client-0.0.1-javadoc.jar.sha1  ← SHA-1 checksum
```

That's **16 files** for a single module. For our 5-module project + parent POM, that's roughly **80-90 files** total on Maven Central.

> **Insight: Why So Many Files?**
>
> Each file serves a distinct consumer need:
> - `.jar` — the actual code that gets compiled into your application
> - `.pom` — Maven reads this to resolve transitive dependencies
> - `-sources.jar` — IDEs (IntelliJ, VS Code) download this to show you source code when you Ctrl+click into a library class
> - `-javadoc.jar` — IDEs use this to show you hover documentation
> - `.asc` — verifies the artifact was published by the owner of the GPG key, not a malicious actor
> - `.md5` / `.sha1` — verifies the artifact wasn't corrupted during download
>
> Maven Central requires all of these. They're not optional extras — they're the quality standard that makes the Java ecosystem trustworthy.

---

## Coordinates in Action

Here's how coordinates flow through the entire system:

```
Your pom.xml (root)                     Consumer's pom.xml
─────────────────                       ──────────────────
<groupId>dev.linhvu</groupId>           <dependency>
<artifactId>zalobot-sdk-java</artifactId>   <groupId>dev.linhvu</groupId>
<version>0.0.1</version>                    <artifactId>zalobot-spring-boot-starter</artifactId>
                                            <version>0.0.1</version>
        │ mvn deploy                    </dependency>
        ▼                                       │ mvn compile
                                                ▼
Maven Central URL                       Maven searches:
────────────────                        ───────────────
dev/linhvu/                             1. ~/.m2/repository/dev/linhvu/...  → miss
  zalobot-sdk-java/0.0.1/              2. Maven Central → hit!
  zalobot-core/0.0.1/                     ↓ downloads
  zalobot-client/0.0.1/                3. Reads POM → finds transitive deps:
  zalobot-listener/0.0.1/                 zalobot-spring-boot → zalobot-client → zalobot-core
  zalobot-spring-boot/0.0.1/              ↓ downloads all transitively
  zalobot-spring-boot-starter/0.0.1/   4. All JARs in classpath → compilation succeeds
```

---

## Summary

| Concept | What It Means |
|---|---|
| Maven coordinates | `groupId:artifactId:version[:packaging[:classifier]]` — unique identifier |
| Coordinate → URL | Dots become slashes in groupId, then `/artifactId/version/filename` |
| Local repository | `~/.m2/repository/` — your machine only |
| Maven Central | `repo.maven.apache.org/maven2/` — searched by default by every Maven user |
| Parent POM published | The parent POM must be on Central so children can resolve inheritance |
| Files per artifact | JAR + POM + sources + javadoc + GPG signatures + checksums ≈ 16 files |

---

## What's Next?

We know our coordinates and where artifacts need to live. In Chapter 3, we learn the complete checklist of what Maven Central demands before it will accept your artifacts.

```
Chapter 1  ──► "Your JARs are trapped"
Chapter 2  ←── YOU ARE HERE: "How Maven finds artifacts"
Chapter 3  ──► "What Maven Central demands"
```
