# Chapter 3: Central Requirements — What Maven Central Demands

> Maven Central isn't just a file server. It validates every artifact before accepting it. What exactly does it check, and why?

---

## The Complete Checklist

Maven Central (via the Central Portal) enforces these requirements on every deployment:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    CENTRAL PORTAL VALIDATION CHECKLIST                   │
│                                                                         │
│  POM Metadata                                                           │
│  ├── ✓ <groupId> present and matches verified namespace                 │
│  ├── ✓ <artifactId> present                                             │
│  ├── ✓ <version> present and NOT a SNAPSHOT                             │
│  ├── ✓ <name> present (human-readable project name)                     │
│  ├── ✓ <description> present (what the project does)                    │
│  ├── ✓ <url> present (project homepage)                                 │
│  ├── ✓ <licenses> with at least one <license> (name + url)              │
│  ├── ✓ <developers> with at least one <developer> (name or id)          │
│  └── ✓ <scm> present (connection, developerConnection, url)             │
│                                                                         │
│  Companion Artifacts                                                    │
│  ├── ✓ -sources.jar present for each artifact                           │
│  └── ✓ -javadoc.jar present for each artifact                           │
│                                                                         │
│  Signatures                                                             │
│  ├── ✓ .asc GPG signature for every file                                │
│  └── ✓ GPG public key available on a keyserver                          │
│                                                                         │
│  Namespace                                                              │
│  └── ✓ groupId namespace verified (you own dev.linhvu)                  │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Why Each Requirement Exists

Every requirement maps to a specific **consumer need**. Maven Central isn't being bureaucratic — it's protecting the millions of developers who will download your artifact:

| Requirement | Consumer Need | What Happens Without It |
|---|---|---|
| `<name>` | Search results show a readable name instead of artifactId | Search for "zalo bot" shows `zalobot-client` with no context |
| `<description>` | Consumers understand what the library does before adding it | Developers add a dependency without knowing what it provides |
| `<url>` | Link to project homepage for documentation, issues, community | No way to report bugs or read docs |
| `<licenses>` | Legal compliance — can I use this in my commercial project? | Legal teams block adoption because license is unknown |
| `<developers>` | Contact the maintainer for security issues | Security vulnerabilities have no responsible disclosure path |
| `<scm>` | IDE "Open on GitHub" navigation, finding the source | Can't verify what the JAR contains matches the claimed source |
| `-sources.jar` | IDE source navigation (Ctrl+click into library code) | Developers debug with decompiled bytecode instead of real source |
| `-javadoc.jar` | IDE hover documentation, API reference | No documentation popups in the IDE |
| `.asc` signatures | Verify the artifact came from the claimed author | Supply chain attacks — anyone could publish malicious code |
| Namespace verification | Prevent impersonation (`dev.linhvu` is really you) | Someone else publishes under your groupId |

> **Insight: Maven Central Validates Consumer Experience, Not Code Quality**
>
> Notice what's NOT on the checklist: no code review, no test coverage requirements, no performance benchmarks, no compatibility testing. Maven Central doesn't care if your code works well — it cares that your artifact is **properly documented, legally usable, and verifiably authentic**.
>
> This is a deliberate design choice. Central is a distribution platform, not a quality gate. Quality assurance is the author's responsibility.

---

## Gap Analysis: Our Project vs. Requirements

Let's map every requirement to the current state of `zalobot-sdk-java`:

### POM Metadata

| Requirement | Current `pom.xml` | Status | Fix (Chapter) |
|---|---|---|---|
| `<groupId>` | `dev.linhvu` (line 7) | ✅ Present | — |
| `<artifactId>` | `zalobot-sdk-java` (line 8) | ✅ Present | — |
| `<version>` | `0.0.1` (line 9) | ✅ Present, not SNAPSHOT | — |
| `<name>` | **Missing** | ❌ | Ch 4 |
| `<description>` | **Missing** | ❌ | Ch 4 |
| `<url>` | **Missing** | ❌ | Ch 4 |
| `<licenses>` | **Missing** | ❌ | Ch 4 |
| `<developers>` | **Missing** | ❌ | Ch 4 |
| `<scm>` | **Missing** | ❌ | Ch 4 |

### Companion Artifacts

| Requirement | Current Configuration | Status | Fix (Chapter) |
|---|---|---|---|
| `-sources.jar` | No `maven-source-plugin` configured | ❌ | Ch 5 |
| `-javadoc.jar` | No `maven-javadoc-plugin` configured | ❌ | Ch 5 |

### Signatures

| Requirement | Current Configuration | Status | Fix (Chapter) |
|---|---|---|---|
| `.asc` signatures | No `maven-gpg-plugin` configured | ❌ | Ch 6 |
| Public key on keyserver | No GPG key exists | ❌ | Ch 6 |

### Publishing Infrastructure

| Requirement | Current Configuration | Status | Fix (Chapter) |
|---|---|---|---|
| Publishing plugin | No `central-publishing-maven-plugin` | ❌ | Ch 7 |
| Central Portal account | Not created | ❌ | Ch 7 |
| Namespace verification | `dev.linhvu` not verified | ❌ | Ch 7 |
| Credentials in settings.xml | No `settings.xml` setup | ❌ | Ch 8 |

---

## How Validation Works at the Central Portal

When you run `mvn deploy -Prelease`, the `central-publishing-maven-plugin` uploads a deployment bundle to the Central Portal. The portal then validates:

```
Deployment uploaded to Central Portal
        │
        ▼
┌─────────────────────────────┐
│ AUTOMATED VALIDATION        │
│                             │
│ For each module in bundle:  │
│ ├── POM has <name>?         │──── NO ──→ FAIL: "Missing <name> in POM"
│ ├── POM has <description>?  │──── NO ──→ FAIL: "Missing <description> in POM"
│ ├── POM has <url>?          │──── NO ──→ FAIL: ...
│ ├── POM has <licenses>?     │──── NO ──→ FAIL: ...
│ ├── POM has <developers>?   │──── NO ──→ FAIL: ...
│ ├── POM has <scm>?          │──── NO ──→ FAIL: ...
│ ├── -sources.jar present?   │──── NO ──→ FAIL: "Missing sources JAR"
│ ├── -javadoc.jar present?   │──── NO ──→ FAIL: "Missing javadoc JAR"
│ ├── .asc for every file?    │──── NO ──→ FAIL: "Missing GPG signature"
│ ├── Version is SNAPSHOT?    │──── YES ─→ FAIL: "SNAPSHOT versions not allowed"
│ └── groupId namespace       │
│     verified?               │──── NO ──→ FAIL: "Namespace not verified"
│                             │
│ ALL PASS                    │
└──────────┬──────────────────┘
           │
           ▼
┌─────────────────────────────┐
│ DEPLOYMENT STATUS:          │
│ VALIDATED                   │
│                             │
│ If autoPublish=true:        │
│   → Published to Central    │
│                             │
│ If autoPublish=false:       │
│   → Awaits manual approval  │
│   → You click "Publish"     │
│   → Published to Central    │
└─────────────────────────────┘
```

The key insight: validation happens **per module**. If `zalobot-core` passes but `zalobot-client` fails, the entire deployment is rejected. All modules must pass.

---

## The Special Case: Parent POM

The parent POM (`zalobot-sdk-java`) has `<packaging>pom</packaging>`. It produces no JAR, so:
- It still needs all POM metadata (`<name>`, `<licenses>`, etc.)
- It **does not** need `-sources.jar` or `-javadoc.jar` (there's no Java code to document)
- Its POM file still needs a `.asc` GPG signature

> **Insight: POM Metadata Inheritance Saves You**
>
> Because child modules inherit from the parent POM, you only need to add `<name>`, `<licenses>`, `<developers>`, `<scm>`, and `<url>` **once** in the parent. All 5 child modules inherit these automatically.
>
> However, `<name>` and `<description>` should be overridden in each child module so they describe that specific module, not the parent. For example:
> - Parent: `<name>ZaloBot SDK for Java</name>`
> - Client: `<name>ZaloBot Client</name>`
> - Listener: `<name>ZaloBot Listener</name>`
>
> `<licenses>`, `<developers>`, and `<scm>` are the same across all modules, so they inherit cleanly.

---

## Summary

| Concept | What It Means |
|---|---|
| Central Portal validation | Automated checks on every deployment before publishing |
| POM metadata | 6 elements: name, description, url, licenses, developers, scm |
| Companion JARs | sources + javadoc for every module |
| GPG signatures | `.asc` file for every artifact, proving authorship |
| Namespace verification | You must prove you own `dev.linhvu` (DNS or GitHub verification) |
| Validation is per-module | All modules must pass; one failure rejects the entire deployment |

---

## What's Next?

In Chapter 4, we start closing the gaps — adding the required POM metadata elements to the parent `pom.xml`.

```
Chapter 2  ──► "How Maven finds artifacts"
Chapter 3  ←── YOU ARE HERE: "What Maven Central demands"
Chapter 4  ──► "Adding required POM metadata"
```
