# Chapter 1: Why SonarQube — The Problem of Invisible Code Quality

> **Type:** CONCEPTUAL — no code changes, no infrastructure setup

---

## The Current State of zalobot-sdk-java

Let's look at what we have today:

```
zalobot-sdk-java/
├── zalobot-core/           10 production files, 7 test files
├── zalobot-client/         16 production files, 0 test files
├── zalobot-listener/        9 production files, 0 test files
├── zalobot-spring-boot/     5 production files, 3 test files
└── zalobot-spring-boot-starter/  0 production files (dependency-only)
                            ─────────────────────────────────
                            40 production files, 10 test files
```

Right now, if someone asks:
- "What percentage of your code is covered by tests?" — **We don't know.**
- "Are there any security vulnerabilities?" — **We don't know.**
- "Did this PR introduce any bugs?" — **We don't know.**
- "Is code quality getting better or worse over time?" — **We don't know.**

We're flying blind.

---

## What SonarQube Detects

SonarQube analyzes your source code and classifies issues into five categories:

### 1. Bugs
Code that is demonstrably wrong or will cause unexpected behavior at runtime.

```java
// Example: null dereference
String name = null;
System.out.println(name.length());  // SonarQube flags this
```

### 2. Vulnerabilities
Code that could be exploited by an attacker.

```java
// Example: hardcoded credential
String token = "sqa_abc123def456";  // SonarQube flags this
```

### 3. Code Smells
Code that works but is confusing, hard to maintain, or violates conventions.

```java
// Example: empty catch block
try {
    riskyOperation();
} catch (Exception e) {
    // SonarQube flags: swallowing exception silently
}
```

### 4. Coverage Gaps
Lines of code that no test ever executes. SonarQube doesn't measure coverage itself — it reads coverage reports from JaCoCo — but it displays and gates on coverage metrics.

### 5. Duplication
Copy-pasted code blocks. SonarQube detects blocks of code that are identical or near-identical across files, which signals a refactoring opportunity.

---

## The Quality Pyramid

Think of code quality as a pyramid — each layer supports the ones above:

```
            ┌─────────────────┐
            │  Quality Gates  │  ← automated pass/fail on every PR
            │  (Chapter 8)    │
            ├─────────────────┤
            │ Static Analysis │  ← bugs, vulnerabilities, smells
            │ (Chapters 5-6)  │
            ├─────────────────┤
            │   Coverage      │  ← "how much code do tests touch?"
            │  (Chapter 4)    │
            ├─────────────────┤
            │     Tests       │  ← JUnit tests (already exist)
            │  (you have 10)  │
            └─────────────────┘
```

**You already have the bottom layer** — tests exist in `zalobot-core` and `zalobot-spring-boot`. This guide builds each layer on top:

1. **Chapter 4** adds JaCoCo to measure *coverage* of your existing tests
2. **Chapters 5-6** add the Sonar Scanner to perform *static analysis*
3. **Chapter 8** configures *Quality Gates* to enforce thresholds

---

## What SonarQube Does NOT Do

It's important to understand the boundaries:

| SonarQube does | SonarQube does NOT |
|---|---|
| Analyze source code for issues | Write tests for you |
| Measure test coverage (via JaCoCo) | Run your tests (Maven/Surefire does that) |
| Track quality trends over time | Fix issues automatically |
| Block PRs that fail Quality Gates | Deploy your application |
| Detect common vulnerability patterns | Replace a security audit |

SonarQube is a **measurement and reporting tool**. It tells you where problems are. You still have to fix them.

---

## The Two Worlds: In-Project vs Outside-Project

A key concept throughout this guide is the separation between:

### In-Project (changes to your codebase)
Things you commit to Git:
- Maven plugin configurations in `pom.xml`
- Sonar properties
- GitHub Actions workflow files
- Exclusion rules

### Outside-Project (infrastructure and UI)
Things that exist on a server or web UI:
- SonarQube/SonarCloud server setup
- Project creation on the dashboard
- Authentication tokens
- Quality Gate configuration
- Quality Profile customization

Most tutorials mix these together, making it hard to know "what do I actually commit?" vs "what do I click in a UI?" This guide labels every chapter clearly.

---

## What's Next

| Chapter | What | Type |
|---|---|---|
| **Ch2** | Choose SonarCloud or self-hosted | Outside-project |
| **Ch3** | Create your SonarQube project and generate a token | Outside-project |
| **Ch4** | Add JaCoCo to measure coverage | In-project |
| **Ch5** | Add the Sonar Maven plugin | In-project |
| **Ch6** | Configure exclusions | In-project |
| **Ch7** | Maven profiles (optional) | In-project |
| **Ch8** | Configure Quality Gates and Profiles | Outside-project |
| **Ch9** | GitHub Actions CI/CD | Both |
| **Ch10** | Complete reference | In-project |

Let's start with choosing your platform.
