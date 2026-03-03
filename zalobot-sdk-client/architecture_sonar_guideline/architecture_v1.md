# SonarQube Integration — Architecture Overview

## 1. Overview: The Restaurant Health Inspector Analogy

Imagine you run a chain of restaurants (your 5 Maven modules). Every day, your chefs cook meals (write code), and customers eat them (users run the software). Everything seems fine — until someone gets food poisoning.

**SonarQube is your health inspector.** It doesn't cook food. It doesn't serve customers. It walks through your kitchen and checks:

| Restaurant Inspection | SonarQube Analysis |
|---|---|
| Are surfaces clean? | Are there code smells? |
| Is food stored at safe temperatures? | Are there security vulnerabilities? |
| Are expiration dates respected? | Is there dead/unreachable code? |
| Do staff wash their hands? | Are coding standards followed? |
| What percentage of food was tested? | What percentage of code is covered by tests? |

Without an inspector, you *think* your kitchen is clean. With one, you *know* — and you have a report to prove it.

**Your project today:**
- 40 production Java files across 5 modules
- 10 test files (all in `zalobot-core`)
- Zero quality metrics, zero coverage measurement, zero static analysis
- No way to know if a PR makes quality better or worse

**After SonarQube integration:**
- Every build measures test coverage via JaCoCo
- Every analysis detects bugs, vulnerabilities, and code smells
- A Quality Gate blocks merges that degrade quality
- A dashboard tracks quality trends over time

---

## 2. Component Hierarchy

```
┌─────────────────────────────────────────────────────────────────┐
│                    SonarQube Server                              │
│               (SonarCloud or self-hosted)                        │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────────────┐  │
│  │ Quality Gate  │  │ Quality      │  │ Dashboard & Reports   │  │
│  │ (pass/fail    │  │ Profile      │  │ (bugs, smells,        │  │
│  │  thresholds)  │  │ (rule set)   │  │  coverage, duplication│  │
│  └──────────────┘  └──────────────┘  └───────────────────────┘  │
└──────────────────────────────────┬──────────────────────────────┘
                                   │ receives analysis report
                                   │
┌──────────────────────────────────┴──────────────────────────────┐
│                   Your Maven Build (local or CI)                 │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │ mvn clean verify sonar:sonar                               │  │
│  │                                                            │  │
│  │  Phase: test          Phase: verify       Goal: sonar      │  │
│  │  ┌─────────────┐     ┌──────────────┐    ┌─────────────┐  │  │
│  │  │ Surefire    │     │ JaCoCo       │    │ Sonar       │  │  │
│  │  │ Plugin      │────▶│ Plugin       │───▶│ Scanner     │  │  │
│  │  │             │     │              │    │ Plugin      │  │  │
│  │  │ runs tests  │     │ collects     │    │ uploads     │  │  │
│  │  │ via JUnit   │     │ coverage     │    │ analysis    │  │  │
│  │  └─────────────┘     └──────────────┘    └─────────────┘  │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                  │
│  Modules:                                                        │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐           │
│  │zalobot-  │ │zalobot-  │ │zalobot-  │ │zalobot-  │           │
│  │core      │ │client    │ │listener  │ │spring-   │           │
│  │          │ │          │ │          │ │boot      │           │
│  │(has tests│ │(no tests)│ │(no tests)│ │(has tests│           │
│  │ 10 files)│ │          │ │          │ │ 3 files) │           │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘           │
│                                                                  │
│  Excluded:  ┌────────────────────┐                               │
│             │zalobot-spring-boot-│  (no code, only dependencies) │
│             │starter             │                               │
│             └────────────────────┘                               │
└──────────────────────────────────────────────────────────────────┘
```

---

## 3. ASCII Class Diagram — Maven Plugin Wiring

```
┌─────────────────────────────────────────────────────────────────┐
│                      Parent POM (pom.xml)                        │
│                                                                  │
│  <build>                                                         │
│    <pluginManagement>                                            │
│    │                                                             │
│    │  ┌──────────────────────────────────┐                       │
│    │  │ jacoco-maven-plugin (0.8.12)     │                       │
│    │  │                                  │                       │
│    │  │ Execution: prepare-agent         │──▶ instruments        │
│    │  │   phase: initialize              │    bytecode before    │
│    │  │                                  │    tests run          │
│    │  │ Execution: report                │──▶ generates per-     │
│    │  │   phase: verify                  │    module jacoco.xml  │
│    │  │                                  │                       │
│    │  │ Execution: report-aggregate      │──▶ merges all module  │
│    │  │   phase: verify                  │    reports into one   │
│    │  │   output: target/site/jacoco-    │    jacoco.xml         │
│    │  │           aggregate/jacoco.xml   │                       │
│    │  └──────────────────────────────────┘                       │
│    │                                                             │
│    │  ┌──────────────────────────────────┐                       │
│    │  │ sonar-maven-plugin (4.0.0.4121)  │                       │
│    │  │                                  │                       │
│    │  │ reads: source files              │                       │
│    │  │ reads: compiled classes          │                       │
│    │  │ reads: jacoco.xml (coverage)     │                       │
│    │  │ sends: analysis report to server │                       │
│    │  └──────────────────────────────────┘                       │
│    │                                                             │
│    │  ┌──────────────────────────────────┐                       │
│    │  │ maven-surefire-plugin            │                       │
│    │  │  (inherited from Spring Boot)    │                       │
│    │  │                                  │                       │
│    │  │ JaCoCo agent is injected via     │                       │
│    │  │ argLine property automatically   │                       │
│    │  └──────────────────────────────────┘                       │
│    │                                                             │
│    </pluginManagement>                                           │
│  </build>                                                        │
│                                                                  │
│  <properties>                                                    │
│    sonar.projectKey = dev.linhvu:zalobot-sdk-java                │
│    sonar.organization = linhvu                                   │
│    sonar.host.url = https://sonarcloud.io                        │
│    sonar.coverage.jacoco.xmlReportPaths =                        │
│      ${project.basedir}/../target/site/jacoco-aggregate/...      │
│  </properties>                                                   │
└──────────────────────────────────────────────────────────────────┘
```

---

## 4. State Diagram — Analysis Lifecycle

```
                    ┌───────┐
                    │ IDLE  │
                    └───┬───┘
                        │ mvn clean verify sonar:sonar
                        ▼
                   ┌─────────┐
                   │CLEANING │  (deletes target/)
                   └────┬────┘
                        ▼
                 ┌────────────┐
                 │ COMPILING  │  (javac compiles sources)
                 └─────┬──────┘
                       ▼
              ┌──────────────────┐
              │ PREPARING AGENT  │  (JaCoCo instruments bytecode)
              │ (initialize)     │
              └────────┬─────────┘
                       ▼
                ┌────────────┐
                │  TESTING   │  (Surefire runs JUnit tests)
                │  (test)    │  (JaCoCo agent records coverage)
                └─────┬──────┘
                      ▼
           ┌───────────────────┐
           │ COVERAGE REPORT   │  (JaCoCo generates per-module XML)
           │ (verify)          │
           └────────┬──────────┘
                    ▼
          ┌─────────────────────┐
          │ COVERAGE AGGREGATE  │  (JaCoCo merges all module XMLs)
          │ (verify)            │
          └─────────┬───────────┘
                    ▼
            ┌──────────────┐
            │  SCANNING    │  (Sonar plugin reads sources +
            │  (sonar)     │   classes + coverage XML)
            └──────┬───────┘
                   ▼
           ┌───────────────┐
           │  UPLOADING    │  (sends analysis report to server)
           └───────┬───────┘
                   ▼
        ┌────────────────────┐
        │ SERVER PROCESSING  │  (SonarQube computes metrics,
        │                    │   applies rules, evaluates gates)
        └─────────┬──────────┘
                  │
           ┌──────┴──────┐
           ▼             ▼
     ┌──────────┐  ┌──────────┐
     │  GATE    │  │  GATE    │
     │  PASSED  │  │  FAILED  │
     │  ✓       │  │  ✗       │
     └──────────┘  └──────────┘
```

---

## 5. Flow: Local Analysis

```
Developer machine
─────────────────

$ mvn clean verify sonar:sonar -Dsonar.token=sqa_xxxxx

Step 1: clean
  └── deletes target/ in all 5 modules

Step 2: initialize (JaCoCo prepare-agent)
  └── for each module (except starter with jacoco.skip=true):
      └── injects JaCoCo Java agent into Surefire's argLine
          argLine = "-javaagent:~/.m2/.../jacocoagent.jar=destfile=target/jacoco.exec"

Step 3: compile
  └── javac compiles *.java → target/classes/*.class

Step 4: test (Surefire)
  └── runs JUnit tests in zalobot-core and zalobot-spring-boot
      └── JaCoCo agent records which lines/branches were hit
      └── writes binary data to target/jacoco.exec

Step 5: verify (JaCoCo report)
  └── for each module:
      └── reads target/jacoco.exec
      └── writes target/site/jacoco/jacoco.xml

Step 6: verify (JaCoCo report-aggregate) — parent POM only
  └── reads all module jacoco.exec files
  └── writes target/site/jacoco-aggregate/jacoco.xml
      (single combined coverage report)

Step 7: sonar:sonar (Sonar Scanner)
  └── reads: all *.java source files
  └── reads: all *.class compiled files
  └── reads: target/site/jacoco-aggregate/jacoco.xml
  └── performs static analysis (bug detection, smell detection)
  └── packages everything into an analysis report
  └── HTTP POST report → https://sonarcloud.io/api/...

Step 8: Server-side (SonarQube)
  └── processes report
  └── computes metrics (coverage %, duplication %, etc.)
  └── evaluates Quality Gate conditions
  └── updates dashboard
```

---

## 6. Flow: CI/CD Analysis (GitHub Actions)

```
GitHub
──────
  push/PR to main
       │
       ▼
  ┌──────────────────────────────────────────────────┐
  │ .github/workflows/sonar.yml                       │
  │                                                    │
  │  1. checkout code                                  │
  │  2. setup JDK 25 (temurin)                         │
  │  3. cache Maven dependencies (~/.m2/repository)    │
  │  4. cache SonarQube scanner data (~/.sonar/cache)  │
  │  5. mvn clean verify sonar:sonar                   │
  │       -Dsonar.token=${{ secrets.SONAR_TOKEN }}      │
  │                                                    │
  │  (steps 1-7 from local flow execute identically)   │
  └──────────────────────┬─────────────────────────────┘
                         │
                         ▼
  ┌──────────────────────────────────────────────────┐
  │ SonarCloud                                        │
  │                                                    │
  │  • processes analysis report                       │
  │  • evaluates Quality Gate                          │
  │  • if PR: posts decoration comment on PR           │
  │  • updates dashboard at sonarcloud.io              │
  └──────────────────────────────────────────────────┘
```

---

## 7. Design Patterns in this Integration

### Pipeline Pattern (Maven Lifecycle)
Each Maven phase is a pipeline stage. Data flows forward: source → compiled classes → test results → coverage → analysis report. You can't skip stages — coverage requires tests, analysis requires coverage.

```
clean → initialize → compile → test → verify → sonar:sonar
  │         │           │        │        │          │
  │    JaCoCo agent  javac    Surefire  JaCoCo    Sonar
  │    injected                runs     reports   uploads
  ▼                            tests
delete
target/
```

### Strategy Pattern (HTTP Client Coverage)
Your `zalobot-client` module has two HTTP client implementations: `JdkClientHttpRequestFactory` and `OkClientHttpRequestFactory`. SonarQube's coverage report will show which implementation paths are tested — revealing whether your tests exercise both strategies or only one.

### Observer Pattern (Quality Gate Webhooks)
SonarQube can notify external systems when analysis completes. GitHub PR decoration uses this: SonarCloud observes analysis completion → posts a status check on the PR → developers see pass/fail without visiting the dashboard.

---

## 8. Architectural Decisions

| Decision | Options | Chosen | Rationale |
|---|---|---|---|
| Platform | SonarCloud vs self-hosted | SonarCloud (recommended) | Zero infrastructure, free for open source, built-in GitHub integration |
| Coverage tool | JaCoCo vs Cobertura | JaCoCo | Industry standard, active development, SonarQube native support |
| Aggregation | Per-module reports vs aggregate | Aggregate at parent | Single coverage report simplifies Sonar configuration |
| Starter module | Analyze vs skip | Skip | No source code — only dependency declarations |
| Quality Gate | Sonar way vs custom | Sonar way (initially) | Good defaults; customize after establishing baseline |
| Coverage target | 80% vs 60% (new code) | 60% initially | Realistic for a project starting from zero; increase over time |
| Sonar execution | Always vs opt-in profile | Explicit `sonar:sonar` goal | Simpler; analysis only when intended |

---

## 9. Mapping Table — Simplified Concepts → Real Components

| Simplified Concept | Real Component | Documentation |
|---|---|---|
| "The inspector" | SonarQube Server / SonarCloud | [docs.sonarsource.com/sonarqube](https://docs.sonarsource.com/sonarqube/latest/) |
| "The inspection report" | Analysis report (uploaded via HTTP) | [Scanner documentation](https://docs.sonarsource.com/sonarqube/latest/analyzing-source-code/overview/) |
| "Coverage measurement" | JaCoCo Maven Plugin | [jacoco.org/jacoco/trunk/doc](https://www.jacoco.org/jacoco/trunk/doc/) |
| "Test runner" | Maven Surefire Plugin | [maven.apache.org/surefire](https://maven.apache.org/surefire/maven-surefire-plugin/) |
| "The scanner" | sonar-maven-plugin | [docs.sonarsource.com/sonarqube/.../maven](https://docs.sonarsource.com/sonarqube/latest/analyzing-source-code/scanners/sonarscanner-for-maven/) |
| "Pass/fail criteria" | Quality Gate | [docs.sonarsource.com/sonarqube/.../quality-gates](https://docs.sonarsource.com/sonarqube/latest/instance-administration/quality-gates/) |
| "Rule book" | Quality Profile | [docs.sonarsource.com/sonarqube/.../quality-profiles](https://docs.sonarsource.com/sonarqube/latest/instance-administration/quality-profiles/) |
| "New code" vs "overall" | New Code Period | [docs.sonarsource.com/sonarqube/.../new-code-definition](https://docs.sonarsource.com/sonarqube/latest/project-administration/clean-as-you-code-settings/defining-new-code/) |

---

## 10. What We Simplified

This guide intentionally omits several advanced SonarQube features to keep focus:

1. **Branch analysis** — SonarCloud supports analyzing feature branches and comparing them to the main branch. We only cover main branch analysis.

2. **Monorepo support** — SonarQube can analyze multiple projects in a single repository with separate project keys. We treat the entire multi-module project as one SonarQube project.

3. **Custom rules** — You can write custom SonarQube rules in Java. We use the built-in "Sonar way" profile.

4. **Security hotspot review workflow** — SonarQube flags potential security issues for manual review. We mention detection but not the review workflow.

5. **Pull request decoration details** — SonarCloud can post inline comments on specific code lines in PRs. We cover the basic setup but not fine-tuning.

6. **SonarLint IDE integration** — SonarLint is an IDE plugin that runs SonarQube rules locally in real-time. It's valuable but outside the scope of Maven/CI integration.

7. **External issue reports** — SonarQube can import reports from other tools (SpotBugs, PMD, Checkstyle). We focus on SonarQube's built-in analyzer.

8. **Test execution data** — SonarQube can import detailed test execution reports (which tests passed/failed). We only import coverage data, not test results.
