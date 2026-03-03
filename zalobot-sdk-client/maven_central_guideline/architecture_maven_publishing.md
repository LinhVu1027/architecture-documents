# Maven Central Publishing Architecture — From Local JARs to Global Distribution

> How does running `mvn deploy -Prelease` transform your local Java project into artifacts that any developer in the world can pull with a single `<dependency>` block?

This document dissects the complete publishing pipeline for the `zalobot-sdk-java` project — a 5-module Maven project with zero publishing infrastructure — and maps every concept to the real configuration changes needed.

---

## 1. Overview — The Book Publishing Analogy

Publishing a Maven artifact is like publishing a book:

| Book Publishing | Maven Central Publishing | Our Project |
|---|---|---|
| **Manuscript** (your writing) | **JAR files** (compiled code) | `zalobot-core-0.0.1.jar`, `zalobot-client-0.0.1.jar`, etc. |
| **ISBN** (unique identifier) | **Maven coordinates** (groupId:artifactId:version) | `dev.linhvu:zalobot-client:0.0.1` |
| **Copyright page** (author, license, publisher) | **POM metadata** (name, license, developers, SCM) | Currently missing — Chapter 4 adds it |
| **Author's signature** (proves authenticity) | **GPG signatures** (.asc files) | Currently missing — Chapter 6 adds it |
| **Publisher** (handles distribution) | **Central Portal** (validates + publishes) | `central.sonatype.com` — Chapter 7 sets it up |
| **Bookstores** (where readers find the book) | **Maven Central** (where developers find artifacts) | `repo.maven.apache.org/maven2/` |
| **Reader** (the consumer) | **Downstream developer** (adds `<dependency>`) | Anyone with `<dependency><groupId>dev.linhvu</groupId>...` |

**The flow:**
1. You write code and build JARs (manuscript)
2. You add metadata: license, developers, SCM URL (copyright page)
3. You generate source + javadoc JARs (appendices for readers)
4. You sign everything with GPG (author's signature)
5. You upload to the Central Portal (send to publisher)
6. Central Portal validates all requirements (editorial review)
7. Artifacts appear on Maven Central (books on shelves worldwide)

---

## 2. Component Hierarchy — The Publishing Pipeline

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         DEVELOPER MACHINE                                   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                     Parent POM (pom.xml)                            │    │
│  │                     dev.linhvu:zalobot-sdk-java:0.0.1               │    │
│  │                                                                     │    │
│  │  ┌─────────────┐ ┌──────────────┐ ┌───────────────┐                │    │
│  │  │ zalobot-core│ │zalobot-client│ │zalobot-listener│               │    │
│  │  └─────────────┘ └──────────────┘ └───────────────┘                │    │
│  │  ┌──────────────────┐ ┌──────────────────────────┐                  │    │
│  │  │zalobot-spring-boot│ │zalobot-spring-boot-starter│                │    │
│  │  └──────────────────┘ └──────────────────────────┘                  │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                              │                                              │
│                              │ mvn deploy -Prelease                         │
│                              ▼                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                    Maven Build Pipeline                              │    │
│  │                                                                     │    │
│  │  compile → test → package → [source-jar] → [javadoc-jar] → [sign]  │    │
│  │                                                                     │    │
│  │  For EACH module produces:                                          │    │
│  │    artifact-0.0.1.jar          (compiled classes)                   │    │
│  │    artifact-0.0.1.pom          (project descriptor)                 │    │
│  │    artifact-0.0.1-sources.jar  (source code)                        │    │
│  │    artifact-0.0.1-javadoc.jar  (API documentation)                  │    │
│  │    *.asc                       (GPG signature for each above)       │    │
│  └───────────────────────────────┬─────────────────────────────────────┘    │
│                                  │                                          │
│  ┌───────────────────────┐       │                                          │
│  │ ~/.m2/settings.xml    │───────┤  (provides credentials)                  │
│  │  - Central Portal token│       │                                          │
│  │  - GPG passphrase     │       │                                          │
│  └───────────────────────┘       │                                          │
└──────────────────────────────────┼──────────────────────────────────────────┘
                                   │ HTTPS upload
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                      CENTRAL PORTAL (central.sonatype.com)                  │
│                                                                             │
│  Receives deployment bundle → validates:                                    │
│    ✓ POM has name, description, url, license, developers, scm              │
│    ✓ Sources JAR present                                                    │
│    ✓ Javadoc JAR present                                                    │
│    ✓ All artifacts have .asc GPG signatures                                 │
│    ✓ No SNAPSHOT versions                                                   │
│    ✓ GPG public key on a keyserver                                          │
│                                                                             │
│  If valid → publishes to Maven Central                                      │
│  If invalid → rejects with specific error messages                          │
└───────────────────────────────────┬─────────────────────────────────────────┘
                                    │ sync
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                      MAVEN CENTRAL (repo.maven.apache.org)                  │
│                                                                             │
│  /maven2/dev/linhvu/zalobot-client/0.0.1/                                  │
│    ├── zalobot-client-0.0.1.jar                                             │
│    ├── zalobot-client-0.0.1.jar.asc                                         │
│    ├── zalobot-client-0.0.1.pom                                             │
│    ├── zalobot-client-0.0.1.pom.asc                                         │
│    ├── zalobot-client-0.0.1-sources.jar                                     │
│    ├── zalobot-client-0.0.1-sources.jar.asc                                 │
│    ├── zalobot-client-0.0.1-javadoc.jar                                     │
│    ├── zalobot-client-0.0.1-javadoc.jar.asc                                 │
│    ├── zalobot-client-0.0.1.jar.md5                                         │
│    └── zalobot-client-0.0.1.jar.sha1                                        │
│  (repeated for all 6 modules: parent + 5 children)                          │
└───────────────────────────────────┬─────────────────────────────────────────┘
                                    │ resolve dependency
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                      CONSUMER APPLICATION                                   │
│                                                                             │
│  <dependency>                                                               │
│      <groupId>dev.linhvu</groupId>                                          │
│      <artifactId>zalobot-spring-boot-starter</artifactId>                   │
│      <version>0.0.1</version>                                               │
│  </dependency>                                                              │
│                                                                             │
│  mvn compile → downloads from Maven Central → works!                        │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. POM Anatomy — Required Sections for Publishing

```
pom.xml (parent)
├── <modelVersion>4.0.0</modelVersion>
├── <groupId>dev.linhvu</groupId>                    ─┐
├── <artifactId>zalobot-sdk-java</artifactId>         │ COORDINATES
├── <version>0.0.1</version>                          │ (already have)
├── <packaging>pom</packaging>                        ─┘
│
├── <name>ZaloBot SDK for Java</name>                 ─┐
├── <description>...</description>                     │
├── <url>https://github.com/...</url>                  │ METADATA
├── <licenses>                                         │ (Chapter 4)
│     └── MIT License                                  │
├── <developers>                                       │
│     └── Linh Vu                                      │
├── <scm>                                              │
│     └── GitHub repository URLs                       ─┘
│
├── <modules>                                          ─┐
│     ├── zalobot-core                                 │
│     ├── zalobot-client                               │ MODULE LIST
│     ├── zalobot-listener                             │ (already have)
│     ├── zalobot-spring-boot                          │
│     └── zalobot-spring-boot-starter                  ─┘
│
├── <properties>...</properties>                        (already have)
├── <dependencyManagement>...</dependencyManagement>    (already have)
│
└── <profiles>
      └── <profile>
            <id>release</id>                           ─┐
            <build>                                     │
              <plugins>                                 │ RELEASE PROFILE
                ├── maven-source-plugin                 │ (Chapter 9)
                │     → generates -sources.jar          │
                ├── maven-javadoc-plugin                │
                │     → generates -javadoc.jar          │
                ├── maven-gpg-plugin                    │
                │     → generates .asc signatures       │
                └── central-publishing-maven-plugin     │
                      → uploads to Central Portal       ─┘
```

---

## 4. State Diagram — The 7-Phase Publishing Lifecycle

```
  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
  │          │    │          │    │  BUILD   │    │          │    │          │    │          │    │          │
  │ DEVELOP  ├───►│ METADATA ├───►│ARTIFACTS ├───►│  SIGN    ├───►│  DEPLOY  ├───►│ VALIDATE ├───►│ RELEASE  │
  │          │    │          │    │          │    │          │    │          │    │          │    │          │
  └──────────┘    └──────────┘    └──────────┘    └──────────┘    └──────────┘    └──────────┘    └──────────┘
       │               │               │               │               │               │               │
       ▼               ▼               ▼               ▼               ▼               ▼               ▼
   Write code     Add to POM:     maven-source    maven-gpg       central-        Central         Artifacts
   Run tests      - name          -plugin         -plugin         publishing-     Portal          appear on
   mvn install    - license       maven-javadoc   signs each      maven-plugin    checks:         Maven Central
                  - developers    -plugin         artifact        uploads         ✓ metadata       immutably
                  - scm           generates       with GPG        bundle to       ✓ sources
                  - url           companion       private key     Central         ✓ javadoc
                                  JARs                            Portal          ✓ signatures
       │               │               │               │               │               │               │
       ▼               ▼               ▼               ▼               ▼               ▼               ▼
   Chapter 1       Chapter 4      Chapter 5       Chapter 6       Chapter 7      Chapter 3       Chapter 12
                                                                  Chapter 8
                                                                  Chapter 9
```

**Key property:** Each phase is additive. Your normal development workflow (`mvn install`) is untouched — the publishing phases only activate with `-Prelease`.

---

## 5. Flow Diagram A — Manual `mvn deploy -Prelease` Trace

```
$ mvn deploy -Prelease

Maven reads pom.xml
  │
  ├── Profile "release" is ACTIVE → source, javadoc, GPG, central-publishing plugins loaded
  │
  ├── Maven reads ~/.m2/settings.xml
  │     ├── <server id="central"> → Central Portal username + token
  │     └── <server id="gpg.passphrase"> → GPG passphrase (or via property)
  │
  └── Reactor build order (from <modules>):
        │
        ├── [1/6] zalobot-sdk-java (parent POM)
        │     └── deploy: uploads zalobot-sdk-java-0.0.1.pom + .asc
        │
        ├── [2/6] zalobot-core
        │     ├── compile: src/main/java → target/classes
        │     ├── test: run unit tests
        │     ├── package: target/zalobot-core-0.0.1.jar
        │     ├── source:jar-no-fork: target/zalobot-core-0.0.1-sources.jar
        │     ├── javadoc:jar: target/zalobot-core-0.0.1-javadoc.jar
        │     ├── gpg:sign: creates .asc for JAR + POM + sources + javadoc (4 signatures)
        │     └── deploy: uploads all 8 files (4 artifacts + 4 .asc)
        │
        ├── [3/6] zalobot-client
        │     └── (same as zalobot-core: 4 artifacts + 4 signatures = 8 files)
        │
        ├── [4/6] zalobot-listener
        │     └── (same: 8 files)
        │
        ├── [5/6] zalobot-spring-boot
        │     └── (same: 8 files)
        │
        └── [6/6] zalobot-spring-boot-starter
              ├── package: zalobot-spring-boot-starter-0.0.1.jar (empty — no Java code)
              ├── source:jar-no-fork: -sources.jar (empty)
              ├── javadoc:jar: -javadoc.jar (empty — skip content generation)
              ├── gpg:sign: 4 signatures
              └── deploy: uploads 8 files

TOTAL uploaded: ~44 files (parent POM + 5 modules × ~8 files each + checksums)

central-publishing-maven-plugin bundles all into a single deployment
  → uploads to Central Portal
  → Central Portal validates
  → if autoPublish=true: immediately released to Maven Central
  → if autoPublish=false: awaits manual approval at central.sonatype.com
```

---

## 6. Flow Diagram B — CI/CD: Push Tag → Published Artifacts

```
Developer                    GitHub                        GitHub Actions Runner
──────────                   ──────                        ─────────────────────

$ git tag v0.0.1             │                             │
$ git push origin v0.0.1 ───►│ tag push event              │
                             │ matches: 'v*'               │
                             │────────────────────────────►│
                             │                             │
                             │                             ├── Checkout code
                             │                             │
                             │                             ├── Setup JDK 25
                             │                             │     (actions/setup-java)
                             │                             │     generates settings.xml with:
                             │                             │       server id=central
                             │                             │       username=${{ secrets.MAVEN_CENTRAL_USERNAME }}
                             │                             │       password=${{ secrets.MAVEN_CENTRAL_PASSWORD }}
                             │                             │
                             │                             ├── Import GPG key
                             │                             │     echo "$GPG_PRIVATE_KEY" | gpg --import
                             │                             │
                             │                             ├── Set version from tag
                             │                             │     VERSION=${GITHUB_REF#refs/tags/v}
                             │                             │     mvn versions:set -DnewVersion=$VERSION
                             │                             │
                             │                             ├── Build + Deploy
                             │                             │     ./mvnw deploy -Prelease --batch-mode
                             │                             │
                             │                             │     compile → test → package
                             │                             │       → source:jar → javadoc:jar
                             │                             │       → gpg:sign → deploy
                             │                             │
                             │                             └── central-publishing-maven-plugin
                             │                                   → uploads to Central Portal
                             │                                   → autoPublish=true → released
                             │
                             │                             Maven Central
                             │                             ─────────────
                             │                             dev.linhvu:zalobot-*:0.0.1
                             │                             available worldwide
```

---

## 7. Design Patterns

| Pattern | Where Used | Configuration Location | Why |
|---|---|---|---|
| **POM Inheritance** | All metadata + plugins in parent, inherited by children | `pom.xml` (root) | Define once, apply to all 5 modules. Children add nothing for publishing |
| **Maven Profiles** | Release plugins gated behind `<profile><id>release</id>` | `pom.xml` (root) `<profiles>` section | Fast `mvn install` in dev, full pipeline only with `-Prelease` |
| **Credential Separation** | Secrets in `settings.xml`, never in POM | `~/.m2/settings.xml` | POM is committed to git; credentials must never be in source control |
| **Server ID Bridging** | `<server><id>central</id>` in settings links to plugin's `<publishingServerId>central</publishingServerId>` | `settings.xml` ↔ `pom.xml` | Decouples "what to deploy to" from "how to authenticate" |
| **Plugin Management** | (optional) Plugins in `<pluginManagement>` for version control | `pom.xml` (root) `<build>` | Ensures consistent plugin versions across all modules |

---

## 8. Architectural Decisions

### Central Portal vs OSSRH (Legacy Nexus)

| | OSSRH (Legacy) | Central Portal (New) |
|---|---|---|
| **URL** | `oss.sonatype.org` | `central.sonatype.com` |
| **Plugin** | `nexus-staging-maven-plugin` | `central-publishing-maven-plugin` |
| **Requires `<distributionManagement>`** | Yes | No |
| **Account creation** | JIRA ticket → wait for approval | Self-service → verify namespace |
| **Status** | Deprecated for new projects | **Recommended** — all new registrations use this |

**Decision:** Use Central Portal. It's simpler (no `<distributionManagement>` needed), self-service, and the officially recommended path for new projects as of 2024+.

### Profile-Gated Deploy vs Always-On Plugins

**Decision:** Wrap all publishing plugins in a `release` profile. This means:
- `mvn install` during development: compiles, tests, packages. Fast.
- `mvn deploy -Prelease`: compiles, tests, packages, generates sources/javadoc, signs, uploads. Full pipeline.

The alternative (always-on plugins with `<skip>` properties) is more complex and error-prone.

### Manual Deploy vs maven-release-plugin

**Decision:** Manual `mvn versions:set` + `mvn deploy -Prelease`. The `maven-release-plugin`:
- Creates two commits (prepare + perform)
- Modifies POMs automatically
- Can leave the project in a broken state if the release fails mid-way

Manual version setting + CI/CD on tag push is simpler, more predictable, and the modern best practice.

### autoPublish: true vs false

**Decision:** Start with `autoPublish=false` (manual approval at Central Portal UI). Switch to `true` once confident in the pipeline. This gives you a safety net — you can inspect the deployment before it becomes permanent.

---

## 9. Mapping Table — Concept to Real Configuration

| Concept | Real Component | Configuration Location | Chapter |
|---|---|---|---|
| Maven coordinates | `<groupId>`, `<artifactId>`, `<version>` | `pom.xml` (root, line 7-9) | Ch 2 |
| Project metadata | `<name>`, `<description>`, `<url>` | `pom.xml` (root) — to be added | Ch 4 |
| License declaration | `<licenses><license>` MIT | `pom.xml` (root) — to be added | Ch 4 |
| Developer info | `<developers><developer>` | `pom.xml` (root) — to be added | Ch 4 |
| Source control | `<scm>` GitHub URLs | `pom.xml` (root) — to be added | Ch 4 |
| Source JAR generation | `maven-source-plugin` 3.3.1 | `pom.xml` release profile — to be added | Ch 5 |
| Javadoc JAR generation | `maven-javadoc-plugin` 3.11.2 | `pom.xml` release profile — to be added | Ch 5 |
| Javadoc skip (starter) | `<maven.javadoc.skip>true</maven.javadoc.skip>` | `zalobot-spring-boot-starter/pom.xml` | Ch 5 |
| GPG signing | `maven-gpg-plugin` 3.2.7 | `pom.xml` release profile — to be added | Ch 6 |
| GPG key management | `gpg --full-generate-key` | Developer machine | Ch 6 |
| Central Portal account | `central.sonatype.com` registration | Browser | Ch 7 |
| Namespace verification | DNS TXT record for `dev.linhvu` | DNS provider | Ch 7 |
| Upload plugin | `central-publishing-maven-plugin` 0.7.0 | `pom.xml` release profile — to be added | Ch 7 |
| Central Portal credentials | `<server><id>central</id>` | `~/.m2/settings.xml` | Ch 8 |
| GPG passphrase | `<server><id>gpg.passphrase</id>` or property | `~/.m2/settings.xml` | Ch 8 |
| Release profile | `<profile><id>release</id>` | `pom.xml` (root) — to be added | Ch 9 |
| Maven Wrapper | `mvnw`, `.mvn/wrapper/` | Project root — to be added | Ch 10 |
| CI/CD pipeline | `.github/workflows/publish.yml` | Project root — to be added | Ch 11 |
| Version management | `mvn versions:set -DnewVersion=X.Y.Z` | CLI command | Ch 12 |

---

## 10. What We Simplified Away

| Topic | What It Does | Why We Skip It |
|---|---|---|
| **SNAPSHOT repositories** | Allows publishing pre-release versions (`0.0.1-SNAPSHOT`) to a snapshot repo | Central Portal doesn't support SNAPSHOTs; use local `mvn install` or GitHub Packages for pre-release |
| **OSGi metadata** | `Bundle-SymbolicName`, `Bundle-Version` in MANIFEST.MF | Only needed if consumers run in OSGi containers (Eclipse RCP, Apache Karaf) |
| **module-info.java** | Java Platform Module System (JPMS) descriptors | Not required for Maven Central; can be added later as a separate concern |
| **Reproducible builds** | `project.build.outputTimestamp` property ensuring byte-identical rebuilds | Best practice but not a Central requirement; orthogonal to publishing |
| **Multi-repository deploy** | Publishing to both Maven Central and GitHub Packages simultaneously | Adds complexity; start with one target |
| **maven-release-plugin** | Automates version bumping, tagging, and deploying in two commits | We use the simpler manual approach: `versions:set` + CI/CD on tag |
| **Gradle publishing** | Publishing via Gradle instead of Maven | This is a Maven project |
| **PGP Web of Trust** | Having other developers sign your GPG key for trust chain | Maven Central only requires that your key is on a keyserver, not that it's in a web of trust |
| **Bill of Materials (BOM)** | A separate POM with `<dependencyManagement>` for consumers | Useful for large SDKs with many modules; can be added later |
| **Relocation POMs** | Telling Maven "this artifact moved to a new groupId/artifactId" | Only needed if you rename your coordinates after initial publication |
