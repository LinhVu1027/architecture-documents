# Chapter 8: Quality Gates & Quality Profiles

> **Type:** OUTSIDE-PROJECT — all configuration happens on the SonarQube/SonarCloud web UI

---

## Two Concepts, Often Confused

```
┌──────────────────────────────────────────────────────────────────┐
│                                                                   │
│  Quality PROFILE                    Quality GATE                  │
│  ────────────────                   ─────────────                 │
│  "Which rules to check"            "What thresholds to enforce"  │
│                                                                   │
│  Example:                           Example:                      │
│  - "Use diamond operator"           - Coverage on new code ≥ 60% │
│  - "Don't use raw types"            - No new bugs                │
│  - "Close resources in finally"     - No new vulnerabilities     │
│                                     - Duplication on new ≤ 3%    │
│                                                                   │
│  Think: "the rulebook"              Think: "the pass/fail exam"  │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

- **Quality Profile** = the set of rules used to analyze your code (e.g., "Sonar way" for Java has ~600 rules)
- **Quality Gate** = the pass/fail conditions applied to analysis results (e.g., "coverage must be ≥ 60%")

Both are configured on the server, not in your Maven POM.

---

## Quality Gates

### What is a Quality Gate?

A Quality Gate is a set of conditions that an analysis must satisfy. If any condition fails, the gate status is "Failed" — visible on the dashboard and optionally blocking PRs.

### The "Sonar way" Default Gate

SonarQube ships with a default Quality Gate called **"Sonar way"**. Its conditions focus on **new code** (code changed since your baseline):

| Condition | Threshold | What It Means |
|---|---|---|
| Coverage on new code | ≥ 80% | New/changed lines must be 80%+ covered by tests |
| Duplicated lines on new code | ≤ 3% | No more than 3% copy-paste in new code |
| Reliability rating on new code | A | No new bugs |
| Security rating on new code | A | No new vulnerabilities |
| Maintainability rating on new code | A | No new blocker/critical code smells |

### Recommended Adjustments for zalobot-sdk-java

The default 80% coverage on new code is aggressive for a project starting from zero. You'll likely want to adjust:

| Condition | Default | Recommended Initial | Why |
|---|---|---|---|
| Coverage on new code | ≥ 80% | **≥ 60%** | Achievable while building test habits |
| Duplicated lines on new code | ≤ 3% | ≤ 3% (keep default) | 3% is reasonable |
| No new bugs | A | A (keep default) | Non-negotiable |
| No new vulnerabilities | A | A (keep default) | Non-negotiable |

### How to Configure (SonarCloud)

1. Go to your project on SonarCloud
2. Click "Project Settings" → "Quality Gate"
3. You can use the default "Sonar way" or create a custom gate:
   - Click "Create" → name it `zalobot-sdk-java-gate`
   - Add conditions:
     - Coverage on New Code ≥ 60%
     - Duplicated Lines on New Code ≤ 3%
     - Reliability Rating on New Code is A
     - Security Rating on New Code is A
4. Assign this gate to your project

### How to Configure (Self-Hosted)

1. Go to `http://localhost:9000`
2. Navigate to "Quality Gates" in the top menu
3. Click "Create" → name it `zalobot-sdk-java-gate`
4. Add the same conditions as above
5. Click "Set as Default" or assign to your project specifically

---

## The "New Code" Concept

This is one of SonarQube's most important features: **Clean as You Code**.

Instead of demanding you fix all existing issues (overwhelming for an existing project), SonarQube focuses on **new code** — code changed or added since a reference point.

```
┌─────────────────────────────────────────────────────┐
│                                                      │
│  Overall Code          New Code                      │
│  ────────────          ────────                      │
│  All 40 files          Only files changed since      │
│  Historical issues     the "New Code Period"          │
│  May have low coverage Must meet gate thresholds     │
│  Informational only    Enforced by Quality Gate      │
│                                                      │
└─────────────────────────────────────────────────────┘
```

### New Code Period Options

| Option | How It Works | Best For |
|---|---|---|
| **Previous version** | Compares to the last tagged version | Projects with regular releases |
| **Reference branch** | Compares to `main` branch | PR-based workflows (recommended) |
| **Number of days** | Last N days of changes | Time-based development cycles |
| **Specific analysis** | Compared to a specific past analysis | One-time baseline reset |

**Recommendation:** Use **"Reference branch" = `main`** for PR analysis. This means every PR is compared against `main`, and the Quality Gate applies only to changes in that PR.

### How to Set (SonarCloud)

1. Go to project → "Administration" → "New Code"
2. Select "Reference Branch"
3. Set branch to `main`

---

## Quality Profiles

### What is a Quality Profile?

A Quality Profile is the set of rules SonarQube uses to analyze your code. Each language has its own profile. For Java, the default "Sonar way" profile includes ~600 rules covering:

- Bug detection (null dereferences, resource leaks, infinite loops)
- Vulnerability detection (SQL injection, XSS, hardcoded secrets)
- Code smell detection (naming conventions, complexity, dead code)

### Should You Customize the Profile?

For most projects, **"Sonar way" is an excellent starting point.** It's maintained by Sonar's team and updated with each release.

You might customize if:
- A rule produces too many false positives for your coding style
- You want to enable additional rules that "Sonar way" leaves off
- You want rules specific to Java 25 features (records, sealed classes, pattern matching)

### How to Customize (if needed)

1. Go to "Quality Profiles" in the top menu
2. Find the "Sonar way" profile for Java
3. Click "Copy" → name it `zalobot-sdk-java-profile`
4. In your copy, you can:
   - **Activate** additional rules (click "Activate More Rules")
   - **Deactivate** noisy rules (search → click → "Deactivate")
   - **Change severity** of rules (Info → Minor → Major → Critical → Blocker)
5. Assign the profile to your project

### Rules Relevant to Your Project

Given that `zalobot-sdk-java` uses Java 25 features, these rules may be particularly relevant:

| Rule Category | Examples |
|---|---|
| Records | Suggest using records for simple data carriers |
| Sealed classes | Exhaustive switch/pattern matching |
| Resource management | Ensure HTTP clients/connections are closed |
| Null safety | Null checks on API responses (`ZaloApiResponse`) |
| Exception handling | Proper exception propagation in listener containers |

---

## Viewing Results

After your first analysis (from Chapter 5), the dashboard shows:

```
┌──────────────────────────────────────────────────────────────┐
│  zalobot-sdk-java                          Quality Gate: ???  │
│                                                               │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────────────┐ │
│  │ Bugs    │  │ Vulns   │  │ Smells  │  │ Coverage        │ │
│  │   ??    │  │   ??    │  │   ??    │  │   ??%           │ │
│  └─────────┘  └─────────┘  └─────────┘  └─────────────────┘ │
│                                                               │
│  ┌──────────────────┐  ┌──────────────────────────────────┐  │
│  │ Duplication      │  │ Lines of Code                    │  │
│  │   ??%            │  │   ~2,000                         │  │
│  └──────────────────┘  └──────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

After the first analysis, actual numbers replace the question marks. Use these numbers to:
1. Establish your baseline
2. Decide if the default Quality Gate thresholds are appropriate
3. Identify which modules need the most attention

---

## Summary

| What | Where | Action |
|---|---|---|
| Quality Gate | SonarQube/SonarCloud UI | Create or customize pass/fail thresholds |
| New Code Period | SonarQube/SonarCloud UI | Set to "Reference branch" = `main` |
| Quality Profile | SonarQube/SonarCloud UI | Start with "Sonar way", customize later |

No code changes in this chapter — all configuration is on the server.

---

## Next: Chapter 9

We bring it all together with GitHub Actions: automated analysis on every push and PR, combining both in-project (workflow file) and outside-project (secrets) configuration.
