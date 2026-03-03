# Chapter 9: GitHub Actions CI/CD

> **Type:** BOTH — in-project (workflow file) + outside-project (GitHub Secrets, SonarCloud integration)

---

## Overview

Until now, SonarQube analysis has been manual: you run `mvn sonar:sonar` on your machine. In this chapter, we automate it so every push and pull request is analyzed automatically.

```
Developer pushes code
        │
        ▼
GitHub Actions triggers ──▶ mvn clean verify sonar:sonar
        │
        ▼
SonarCloud receives report
        │
        ├──▶ Updates dashboard
        └──▶ Posts status check on PR (pass/fail)
```

---

## Outside-Project: GitHub Secrets

### Step 1: Add SONAR_TOKEN secret

1. Go to your GitHub repository
2. Navigate to **Settings** → **Secrets and variables** → **Actions**
3. Click **"New repository secret"**
4. Name: `SONAR_TOKEN`
5. Value: paste the token from Chapter 3
6. Click **"Add secret"**

### Step 2: (SonarCloud only) Enable GitHub Integration

1. On SonarCloud, go to your project → **Administration** → **General Settings** → **Pull Requests**
2. Under "Integration with GitHub", ensure it shows "Configured"
3. If not configured:
   - Go to your SonarCloud organization settings
   - Click **"GitHub"** integration
   - Follow the prompts to install the SonarCloud GitHub App
4. This enables:
   - PR decoration (status checks on pull requests)
   - PR analysis (analyzing only the changed code)

---

## In-Project: GitHub Actions Workflow File

Create the file `.github/workflows/sonar.yml`:

```yaml
name: SonarQube Analysis

on:
  push:
    branches:
      - main
  pull_request:
    types: [opened, synchronize, reopened]

jobs:
  sonar:
    name: SonarQube Analysis
    runs-on: ubuntu-latest

    steps:
      # 1. Check out the code
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Full history needed for New Code Period

      # 2. Set up JDK 25
      - name: Set up JDK 25
        uses: actions/setup-java@v4
        with:
          java-version: '25'
          distribution: 'temurin'

      # 3. Cache Maven dependencies
      - name: Cache Maven packages
        uses: actions/cache@v4
        with:
          path: ~/.m2/repository
          key: ${{ runner.os }}-maven-${{ hashFiles('**/pom.xml') }}
          restore-keys: |
            ${{ runner.os }}-maven-

      # 4. Cache SonarQube scanner data
      - name: Cache SonarQube packages
        uses: actions/cache@v4
        with:
          path: ~/.sonar/cache
          key: ${{ runner.os }}-sonar
          restore-keys: |
            ${{ runner.os }}-sonar

      # 5. Build, test, collect coverage, and analyze
      - name: Build and Analyze
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
        run: >
          mvn -B clean verify sonar:sonar
          -Dsonar.token=${{ secrets.SONAR_TOKEN }}
```

---

## What Each Step Does

### `fetch-depth: 0`
SonarQube needs the full Git history to compute the "New Code Period" (which lines are new vs existing). A shallow clone (`fetch-depth: 1`, the default) would make all code appear as "new."

### `java-version: '25'`
Matches your project's `<maven.compiler.source>25</maven.compiler.source>`. Using a different JDK version than your project targets can cause compilation failures or incorrect analysis.

### Maven cache
Caches `~/.m2/repository` between workflow runs. Without this, every run downloads all dependencies from scratch (~100MB+ for this project). The cache key is based on the hash of all `pom.xml` files — if dependencies change, the cache is invalidated.

### SonarQube cache
Caches `~/.sonar/cache` which stores plugin data and analysis metadata. Reduces the time SonarQube spends downloading its internal plugins on each run.

### `-B` flag
Runs Maven in **batch mode** (non-interactive). Suppresses download progress messages that clutter CI logs.

---

## Workflow Execution Flow

```
Push to main or PR opened/updated
         │
         ▼
┌─────────────────────────────────────────────┐
│ GitHub Actions Runner (ubuntu-latest)        │
│                                              │
│  1. git clone (full history)                 │
│  2. install JDK 25                           │
│  3. restore Maven cache                      │
│  4. restore Sonar cache                      │
│  5. mvn -B clean verify sonar:sonar          │
│     │                                        │
│     ├── clean: delete target/                │
│     ├── compile: javac                       │
│     ├── test: Surefire + JaCoCo agent        │
│     ├── verify: JaCoCo report + aggregate    │
│     └── sonar: analyze + upload              │
│                                              │
│  6. save Maven cache                         │
│  7. save Sonar cache                         │
└──────────────────┬──────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────┐
│ SonarCloud                                    │
│                                               │
│  • Processes analysis report                  │
│  • Evaluates Quality Gate                     │
│  • Updates dashboard                          │
│  • If PR: posts status check                  │
│    ┌─────────────────────────────────────┐    │
│    │ ✓ SonarCloud Quality Gate passed    │    │
│    │   Coverage: 65% (≥60% required)     │    │
│    │   0 new bugs, 0 new vulnerabilities │    │
│    └─────────────────────────────────────┘    │
└──────────────────────────────────────────────┘
```

---

## PR Decoration

When SonarCloud's GitHub integration is configured, PRs get:

1. **A status check** — "SonarCloud Quality Gate" appears in the PR's checks section with pass/fail
2. **A summary comment** — SonarCloud posts a comment showing coverage, bugs, smells, and the Quality Gate result
3. **In-line annotations** (optional) — SonarCloud can mark specific lines with issues directly in the PR diff

This happens automatically — no additional configuration needed in the workflow file.

---

## Handling Pull Requests vs Push

The workflow triggers on both `push` to `main` and `pull_request`. The behavior differs:

| Trigger | Analysis Type | New Code Baseline |
|---|---|---|
| Push to `main` | Full branch analysis | Previous analysis of `main` |
| Pull request | PR analysis | Comparison against target branch (`main`) |

For PRs, SonarCloud automatically detects it's a PR analysis and compares only the changed code against the target branch. This is why `fetch-depth: 0` is important — SonarQube needs to determine the merge base.

---

## Security: Token Handling

The `SONAR_TOKEN` secret is:
- **Encrypted at rest** in GitHub
- **Masked in logs** — GitHub automatically redacts it from workflow output
- **Not available to fork PRs** — for security, secrets are not exposed to PRs from forks

If your repository accepts contributions from forks, the SonarQube step will fail on fork PRs because `SONAR_TOKEN` is empty. You can handle this by making the analysis step conditional:

```yaml
- name: Build and Analyze
  if: env.SONAR_TOKEN != ''
  env:
    SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
  run: >
    mvn -B clean verify sonar:sonar
    -Dsonar.token=${{ secrets.SONAR_TOKEN }}

- name: Build only (no analysis)
  if: env.SONAR_TOKEN == ''
  env:
    SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
  run: mvn -B clean verify
```

---

## Estimated CI Time

For `zalobot-sdk-java` with 40 production files and 10 test files:

| Step | Estimated Time |
|---|---|
| Checkout | ~5 seconds |
| JDK setup | ~10 seconds (cached) |
| Maven build + test | ~30-60 seconds |
| Sonar analysis + upload | ~20-30 seconds |
| **Total** | **~1-2 minutes** |

This is fast because the project is small. The Maven and Sonar caches keep it fast as the project grows.

---

## Summary of Changes

### In-Project
| File | Change |
|---|---|
| `.github/workflows/sonar.yml` | New file — GitHub Actions workflow |

### Outside-Project
| Where | Change |
|---|---|
| GitHub → Repository Settings → Secrets | Add `SONAR_TOKEN` secret |
| SonarCloud → Organization Settings | Configure GitHub integration (for PR decoration) |

---

## Next: Chapter 10

The complete configuration reference — all POM changes consolidated, all commands in one place, and a step-by-step checklist.
