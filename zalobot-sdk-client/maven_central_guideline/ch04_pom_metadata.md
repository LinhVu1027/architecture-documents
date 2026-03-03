# Chapter 4: POM Metadata — Adding Name, License, Developers, and SCM

> Maven Central requires six metadata elements in your POM. All six go in the parent POM once, and all child modules inherit them.

---

## The Six Required Elements

```
pom.xml (root)
│
├── <name>              ← Human-readable project name
├── <description>       ← What the project does
├── <url>               ← Project homepage (GitHub page)
├── <licenses>          ← Legal terms (MIT)
├── <developers>        ← Who maintains the project
└── <scm>               ← Source code management (GitHub repo URLs)
```

Each element serves a different consumer of your artifact:

```
<name>         → Maven Central search results, IDE dependency view
<description>  → Maven Central search results, project listings
<url>          → "Visit Homepage" links in Maven Central, IDEs
<licenses>     → Legal compliance tools (FOSSA, Snyk, license-maven-plugin)
<developers>   → Security vulnerability disclosure
<scm>          → IDE "Open on GitHub", contributor workflows
```

---

## Exact XML to Add to `pom.xml` (Root)

Add the following block after the `<packaging>pom</packaging>` line in the root `pom.xml`:

```xml
<name>ZaloBot SDK for Java</name>
<description>Java SDK for the Zalo Bot API — client, listener, and Spring Boot integration</description>
<url>https://github.com/<YOUR_GITHUB_USERNAME>/zalobot-sdk-java</url>

<licenses>
    <license>
        <name>MIT License</name>
        <url>https://opensource.org/licenses/MIT</url>
        <distribution>repo</distribution>
    </license>
</licenses>

<developers>
    <developer>
        <name>Linh Vu</name>
        <url>https://github.com/<YOUR_GITHUB_USERNAME></url>
    </developer>
</developers>

<scm>
    <connection>scm:git:git://github.com/<YOUR_GITHUB_USERNAME>/zalobot-sdk-java.git</connection>
    <developerConnection>scm:git:ssh://github.com/<YOUR_GITHUB_USERNAME>/zalobot-sdk-java.git</developerConnection>
    <url>https://github.com/<YOUR_GITHUB_USERNAME>/zalobot-sdk-java</url>
</scm>
```

Replace `<YOUR_GITHUB_USERNAME>` with your actual GitHub username once you create the repository.

---

## Element-by-Element Breakdown

### `<name>` — Human-Readable Project Name

```xml
<name>ZaloBot SDK for Java</name>
```

This appears in Maven Central search results and IDE dependency trees. It should be short and descriptive — not a copy of the artifactId.

For child modules, override `<name>` to be module-specific:

| Module | `<name>` |
|---|---|
| Parent | `ZaloBot SDK for Java` |
| `zalobot-core` | `ZaloBot Core` |
| `zalobot-client` | `ZaloBot Client` |
| `zalobot-listener` | `ZaloBot Listener` |
| `zalobot-spring-boot` | `ZaloBot Spring Boot Auto-Configuration` |
| `zalobot-spring-boot-starter` | `ZaloBot Spring Boot Starter` |

### `<description>` — What the Project Does

```xml
<description>Java SDK for the Zalo Bot API — client, listener, and Spring Boot integration</description>
```

One sentence. Appears in search results. The parent description applies to the entire project.

### `<url>` — Project Homepage

```xml
<url>https://github.com/<YOUR_GITHUB_USERNAME>/zalobot-sdk-java</url>
```

Links to where users can find documentation, file issues, and contribute. All child modules inherit this — they all point to the same repository.

### `<licenses>` — Legal Terms

```xml
<licenses>
    <license>
        <name>MIT License</name>
        <url>https://opensource.org/licenses/MIT</url>
        <distribution>repo</distribution>
    </license>
</licenses>
```

- `<name>`: The SPDX license name. MIT is one of the most permissive — allows commercial use, modification, distribution.
- `<url>`: Points to the license text. Use the SPDX canonical URL.
- `<distribution>repo</distribution>`: Means "this artifact can be distributed from a Maven repository." The alternative is `manual` (must be hand-delivered), which is rare.

> **Insight: License Choice Matters for Adoption**
>
> MIT License is the most adoption-friendly choice. Companies with strict legal policies can use MIT-licensed libraries without legal review. Copyleft licenses (GPL, AGPL) require that any project using the library also be open-sourced under the same license — which blocks most commercial adoption. Apache 2.0 is another popular permissive choice that also includes an explicit patent grant.

### `<developers>` — Who Maintains This

```xml
<developers>
    <developer>
        <name>Linh Vu</name>
        <url>https://github.com/<YOUR_GITHUB_USERNAME></url>
    </developer>
</developers>
```

At minimum, Maven Central requires `<name>` or `<id>` inside `<developer>`. Adding `<url>` is optional but helpful for users who want to contact you.

### `<scm>` — Source Code Management

```xml
<scm>
    <connection>scm:git:git://github.com/<YOUR_GITHUB_USERNAME>/zalobot-sdk-java.git</connection>
    <developerConnection>scm:git:ssh://github.com/<YOUR_GITHUB_USERNAME>/zalobot-sdk-java.git</developerConnection>
    <url>https://github.com/<YOUR_GITHUB_USERNAME>/zalobot-sdk-java</url>
</scm>
```

- `<connection>`: Read-only access URL (anonymous clone). Prefix: `scm:git:`.
- `<developerConnection>`: Read-write access URL (authenticated clone). Prefix: `scm:git:ssh://`.
- `<url>`: Browsable web interface (GitHub's HTML page).

> **Insight: The `scm:git:` Prefix**
>
> The `scm:git:` prefix isn't just decoration. It follows Maven's SCM URL format: `scm:<provider>:<provider-specific-url>`. Maven's `maven-scm` API uses this to support multiple version control systems (git, svn, hg) through a single interface. The `maven-release-plugin` (which we don't use, but others might) parses this to know which VCS commands to run.

---

## Where These Go in the POM

Here's how the parent POM looks with metadata added:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>dev.linhvu</groupId>
    <artifactId>zalobot-sdk-java</artifactId>
    <version>0.0.1</version>
    <packaging>pom</packaging>

    <!-- ======== NEW: Required metadata for Maven Central ======== -->
    <name>ZaloBot SDK for Java</name>
    <description>Java SDK for the Zalo Bot API — client, listener, and Spring Boot integration</description>
    <url>https://github.com/<YOUR_GITHUB_USERNAME>/zalobot-sdk-java</url>

    <licenses>
        <license>
            <name>MIT License</name>
            <url>https://opensource.org/licenses/MIT</url>
            <distribution>repo</distribution>
        </license>
    </licenses>

    <developers>
        <developer>
            <name>Linh Vu</name>
            <url>https://github.com/<YOUR_GITHUB_USERNAME></url>
        </developer>
    </developers>

    <scm>
        <connection>scm:git:git://github.com/<YOUR_GITHUB_USERNAME>/zalobot-sdk-java.git</connection>
        <developerConnection>scm:git:ssh://github.com/<YOUR_GITHUB_USERNAME>/zalobot-sdk-java.git</developerConnection>
        <url>https://github.com/<YOUR_GITHUB_USERNAME>/zalobot-sdk-java</url>
    </scm>
    <!-- ======== END: Required metadata ======== -->

    <modules>
        <module>zalobot-core</module>
        <module>zalobot-client</module>
        <module>zalobot-listener</module>
        <module>zalobot-spring-boot</module>
        <module>zalobot-spring-boot-starter</module>
    </modules>

    <!-- ... rest of existing POM (properties, dependencyManagement) ... -->
</project>
```

---

## Child Module Overrides

Each child module should override `<name>` and optionally `<description>`. Example for `zalobot-client/pom.xml`:

```xml
<parent>
    <groupId>dev.linhvu</groupId>
    <artifactId>zalobot-sdk-java</artifactId>
    <version>0.0.1</version>
</parent>

<artifactId>zalobot-client</artifactId>
<name>ZaloBot Client</name>
<description>HTTP client for the Zalo Bot API</description>

<!-- licenses, developers, scm → all inherited from parent. Nothing to add. -->
```

> **Insight: POM Inheritance — Define Once, Inherit Everywhere**
>
> In Maven's inheritance model, child POMs inherit ALL elements from the parent unless explicitly overridden. For metadata:
> - `<licenses>`, `<developers>`, `<scm>` are the same for all modules → define in parent only
> - `<name>` and `<description>` should be specific to each module → override in each child
> - `<url>` can be inherited (all modules live in the same repo) or overridden per module
>
> This is why multi-module Maven projects with a parent POM are the standard structure for publishing to Central — you configure publishing once.

---

## Summary

| Element | Value | Where Defined | Inherited by Children? |
|---|---|---|---|
| `<name>` | `ZaloBot SDK for Java` | Parent, overridden in each child | Yes, but should override |
| `<description>` | Project description | Parent, optionally overridden | Yes, but should override |
| `<url>` | GitHub repository URL | Parent only | Yes |
| `<licenses>` | MIT License | Parent only | Yes |
| `<developers>` | Linh Vu | Parent only | Yes |
| `<scm>` | GitHub URLs | Parent only | Yes |

---

## What's Next?

Metadata is done. In Chapter 5, we generate the two companion JARs that Maven Central requires: the sources JAR and the javadoc JAR.

```
Chapter 3  ──► "What Maven Central demands"
Chapter 4  ←── YOU ARE HERE: "Adding required POM metadata"
Chapter 5  ──► "Source and Javadoc JARs"
```
