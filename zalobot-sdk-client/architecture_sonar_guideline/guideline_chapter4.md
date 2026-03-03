# Chapter 4: Adding JaCoCo for Code Coverage

> **Type:** IN-PROJECT — changes to `pom.xml` files that you commit to Git

---

## What is JaCoCo?

JaCoCo (Java Code Coverage) is a library that measures which lines and branches of your code are executed during tests. It works by **instrumenting bytecode** — injecting tiny probes into your compiled `.class` files that record "this line was reached."

```
                    Without JaCoCo                With JaCoCo
                    ──────────────                ─────────────
Source:  foo.java ──▶ foo.class ──▶ tests run    foo.class + probes ──▶ tests run
                                       │                                    │
                                       ▼                                    ▼
                                  "tests pass"                     "tests pass"
                                                                  + jacoco.exec
                                                                  + jacoco.xml
                                                                    (coverage data)
```

JaCoCo has three Maven plugin executions we need:

| Execution | Phase | What It Does |
|---|---|---|
| `prepare-agent` | `initialize` | Injects the JaCoCo agent into Surefire's JVM arguments |
| `report` | `verify` | Generates per-module coverage report (XML) from `jacoco.exec` |
| `report-aggregate` | `verify` | Merges all module reports into one combined XML |

---

## Why Aggregation Matters for Multi-Module Projects

Our project has 5 modules, but tests are only in 2 of them (`zalobot-core` and `zalobot-spring-boot`). Without aggregation, we'd get separate coverage reports per module — and modules without tests would show 0% coverage individually.

With aggregation at the parent level, we get **one combined report** that SonarQube can read:

```
zalobot-core/target/jacoco.exec          ──┐
zalobot-spring-boot/target/jacoco.exec   ──┤──▶ target/site/jacoco-aggregate/jacoco.xml
zalobot-client/target/jacoco.exec        ──┤    (combined report at parent level)
zalobot-listener/target/jacoco.exec      ──┘
```

---

## The Change: Parent POM

Here is what to add to the **parent `pom.xml`** (`/pom.xml`). Currently, the parent POM has no `<build>` section at all. We add the entire `<build>` block:

```xml
<!-- Add this after </dependencyManagement> in the parent pom.xml -->

<build>
    <pluginManagement>
        <plugins>
            <plugin>
                <groupId>org.jacoco</groupId>
                <artifactId>jacoco-maven-plugin</artifactId>
                <version>0.8.12</version>
                <executions>
                    <!-- 1. Inject JaCoCo agent before tests run -->
                    <execution>
                        <id>prepare-agent</id>
                        <goals>
                            <goal>prepare-agent</goal>
                        </goals>
                    </execution>
                    <!-- 2. Generate per-module coverage report after tests -->
                    <execution>
                        <id>report</id>
                        <goals>
                            <goal>report</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </pluginManagement>

    <plugins>
        <!-- Activate JaCoCo for all modules -->
        <plugin>
            <groupId>org.jacoco</groupId>
            <artifactId>jacoco-maven-plugin</artifactId>
            <executions>
                <execution>
                    <id>prepare-agent</id>
                </execution>
                <execution>
                    <id>report</id>
                </execution>
                <!-- 3. Aggregate coverage from all modules (parent only) -->
                <execution>
                    <id>report-aggregate</id>
                    <phase>verify</phase>
                    <goals>
                        <goal>report-aggregate</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

### What each part does:

1. **`<pluginManagement>`** — Declares the JaCoCo plugin version and default executions. Child modules inherit this configuration but don't activate it unless they also declare the plugin in `<plugins>`.

2. **`<plugins>` (outside `<pluginManagement>`)** — Activates the plugin for all modules. The `prepare-agent` and `report` executions run in every module. The `report-aggregate` execution runs only at the parent level and combines all module reports.

---

## Excluding the Starter Module

The `zalobot-spring-boot-starter` module has no source code — it only declares dependencies. Running JaCoCo on it is wasteful and may produce warnings. Add this to `zalobot-spring-boot-starter/pom.xml`:

```xml
<!-- Add inside <properties> in zalobot-spring-boot-starter/pom.xml -->
<jacoco.skip>true</jacoco.skip>
```

The full `<properties>` block becomes:

```xml
<properties>
    <maven.compiler.source>25</maven.compiler.source>
    <maven.compiler.target>25</maven.compiler.target>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <jacoco.skip>true</jacoco.skip>
</properties>
```

---

## How It Works Under the Hood

When you run `mvn clean verify`, here's what happens in each module:

```
Phase: initialize
  └── JaCoCo prepare-agent runs
      └── Sets argLine property:
          -javaagent:/path/to/jacocoagent.jar=destfile=target/jacoco.exec
      └── Surefire picks up argLine automatically

Phase: test
  └── Surefire runs JUnit tests
      └── JaCoCo agent inside the JVM records coverage
      └── On JVM shutdown, writes target/jacoco.exec (binary format)

Phase: verify
  └── JaCoCo report runs
      └── Reads: target/jacoco.exec + target/classes/
      └── Writes: target/site/jacoco/jacoco.xml (per-module report)

  └── JaCoCo report-aggregate runs (parent only)
      └── Reads: */target/jacoco.exec + */target/classes/
      └── Writes: target/site/jacoco-aggregate/jacoco.xml (combined)
```

---

## Verifying the Change

After making the POM changes, run:

```bash
mvn clean verify
```

**Expected output** (look for these lines in the build log):

```
[INFO] --- jacoco:0.8.12:prepare-agent (prepare-agent) @ zalobot-core ---
[INFO] argLine set to -javaagent:...jacocoagent.jar=destfile=.../target/jacoco.exec
...
[INFO] --- jacoco:0.8.12:report (report) @ zalobot-core ---
[INFO] Loading execution data file .../zalobot-core/target/jacoco.exec
[INFO] Analyzed bundle 'zalobot-core' with 10 classes
...
[INFO] --- jacoco:0.8.12:report-aggregate (report-aggregate) @ zalobot-sdk-java ---
```

**Expected file:**

```bash
ls -la target/site/jacoco-aggregate/jacoco.xml
# This file should exist and contain coverage data
```

If `jacoco.xml` exists, JaCoCo is working. This is the file SonarQube will read in Chapter 5.

---

## Troubleshooting

### "No execution data file found"
The module has no tests. This is expected for `zalobot-client` and `zalobot-listener`. The `jacoco.exec` file is only created when tests run.

### "Classes in bundle do not match with execution data"
This happens when classes are recompiled between the test and report phases. Make sure you're running `mvn clean verify` (not running individual phases separately).

### Surefire's argLine conflict
If you already set a custom `<argLine>` in Surefire configuration, JaCoCo's `prepare-agent` may not inject correctly. The fix is to append: `<argLine>@{argLine} your-other-args</argLine>`. This project doesn't have custom argLine, so this isn't an issue.

---

## Summary of Changes

| File | Change |
|---|---|
| `pom.xml` (parent) | Add `<build>` section with JaCoCo plugin |
| `zalobot-spring-boot-starter/pom.xml` | Add `<jacoco.skip>true</jacoco.skip>` property |

---

## Next: Chapter 5

With coverage measurement in place, we'll add the Sonar Maven Plugin to upload analysis results to SonarQube.
