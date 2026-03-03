# Chapter 1: The Problem — Your JARs Are Trapped

> You've built a 5-module SDK. You run `mvn install`. It works perfectly on your machine. But no one else in the world can use it. Why?

---

## The Current State

After running `mvn install`, Maven places your artifacts in the **local repository**:

```
~/.m2/repository/dev/linhvu/
├── zalobot-sdk-java/0.0.1/
│     └── zalobot-sdk-java-0.0.1.pom
├── zalobot-core/0.0.1/
│     ├── zalobot-core-0.0.1.jar
│     └── zalobot-core-0.0.1.pom
├── zalobot-client/0.0.1/
│     ├── zalobot-client-0.0.1.jar
│     └── zalobot-client-0.0.1.pom
├── zalobot-listener/0.0.1/
│     ├── zalobot-listener-0.0.1.jar
│     └── zalobot-listener-0.0.1.pom
├── zalobot-spring-boot/0.0.1/
│     ├── zalobot-spring-boot-0.0.1.jar
│     └── zalobot-spring-boot-0.0.1.pom
└── zalobot-spring-boot-starter/0.0.1/
      ├── zalobot-spring-boot-starter-0.0.1.jar
      └── zalobot-spring-boot-starter-0.0.1.pom
```

This is **your** `~/.m2` directory. It lives on **your** machine only.

---

## What a Consumer Sees

A developer on another machine tries to use your SDK:

```xml
<!-- Their pom.xml -->
<dependency>
    <groupId>dev.linhvu</groupId>
    <artifactId>zalobot-spring-boot-starter</artifactId>
    <version>0.0.1</version>
</dependency>
```

They run `mvn compile` and get:

```
[ERROR] Failed to execute goal on project their-app:
  Could not find artifact dev.linhvu:zalobot-spring-boot-starter:jar:0.0.1
  in central (https://repo.maven.apache.org/maven2)
```

Maven searched everywhere it knows and found nothing.

---

## Where Maven Searches

When Maven encounters a `<dependency>`, it searches in this exact order:

```
Step 1: LOCAL REPOSITORY (~/.m2/repository/)
        └── Does dev/linhvu/zalobot-spring-boot-starter/0.0.1/ exist here?
            └── On YOUR machine: YES (you ran mvn install)
            └── On THEIR machine: NO → continue searching

Step 2: CONFIGURED REMOTE REPOSITORIES (from <repositories> in POM or settings.xml)
        └── Any custom repos configured?
            └── Typically NO for most projects → continue searching

Step 3: MAVEN CENTRAL (https://repo.maven.apache.org/maven2/)
        └── Does this artifact exist on Central?
            └── NO — you never published it
            └── ERROR: "Could not find artifact"
```

> **Insight: `mvn install` vs `mvn deploy`**
>
> These two commands sound similar but do fundamentally different things:
> - `mvn install` — copies artifacts to your **local** `~/.m2/repository/`. Only you can use them. It's like saving a document to your hard drive.
> - `mvn deploy` — uploads artifacts to a **remote** repository (like Maven Central). Anyone can use them. It's like publishing a document to the internet.
>
> The gap between `install` and `deploy` is what this entire guide bridges.

---

## The Gap Analysis

Here's everything our project currently has vs. what Maven Central requires:

| Requirement | Current State | Status |
|---|---|---|
| Maven coordinates (groupId, artifactId, version) | `dev.linhvu:zalobot-*:0.0.1` | ✅ Have it |
| `<packaging>` declared | `pom` for parent, `jar` (default) for children | ✅ Have it |
| `<name>` element | Missing | ❌ Need to add |
| `<description>` element | Missing | ❌ Need to add |
| `<url>` element | Missing | ❌ Need to add |
| `<licenses>` element | Missing | ❌ Need to add |
| `<developers>` element | Missing | ❌ Need to add |
| `<scm>` element | Missing | ❌ Need to add |
| Source JAR (`-sources.jar`) | Not generated | ❌ Need to add |
| Javadoc JAR (`-javadoc.jar`) | Not generated | ❌ Need to add |
| GPG signatures (`.asc`) | Not generated | ❌ Need to add |
| Publishing plugin | Not configured | ❌ Need to add |
| Credentials management | No `settings.xml` setup | ❌ Need to add |
| Central Portal account | Not created | ❌ Need to create |
| No SNAPSHOT in version | `0.0.1` (no SNAPSHOT) | ✅ Have it |

That's **10 gaps** to close. The next 11 chapters close them one by one.

---

## The Journey Ahead

```
YOU ARE HERE                                                      GOAL
     │                                                              │
     ▼                                                              ▼
  mvn install                                                   mvn deploy -Prelease
  JARs in ~/.m2 only                                           JARs on Maven Central
  Only you can use them                                        Anyone can use them
     │                                                              │
     └── ch02: Understand coordinates ──────────────────────────────┘
         ch03: Learn Central's requirements
         ch04: Add POM metadata
         ch05: Generate source + javadoc JARs
         ch06: Sign with GPG
         ch07: Configure Central Portal plugin
         ch08: Set up credentials in settings.xml
         ch09: Wire it all together with a release profile
         ch10: Add Maven Wrapper for reproducibility
         ch11: Automate with GitHub Actions
         ch12: Version management and release workflow
```

---

## Summary

| Concept | What It Means |
|---|---|
| `mvn install` | Copies artifacts to `~/.m2/repository/` — local only |
| `mvn deploy` | Uploads artifacts to a remote repository — globally accessible |
| Maven Central | The default remote repository Maven searches |
| The problem | Your JARs exist locally but aren't published to any remote repository |
| The solution | Configure metadata, generate companion JARs, sign, and deploy to Maven Central |

---

## What's Next?

In Chapter 2, we learn how Maven actually finds artifacts — the coordinate system that turns `dev.linhvu:zalobot-client:0.0.1` into a specific URL on Maven Central.

```
Chapter 1  ←── YOU ARE HERE: "Your JARs are trapped"
Chapter 2  ──► "How Maven finds artifacts"
```
