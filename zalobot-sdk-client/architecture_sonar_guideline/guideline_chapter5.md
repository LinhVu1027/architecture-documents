# Chapter 5: Adding the Sonar Maven Plugin

> **Type:** IN-PROJECT — changes to the parent `pom.xml`

---

## What is the Sonar Maven Plugin?

The Sonar Maven Plugin is the bridge between your Maven build and the SonarQube server. It:

1. Reads your source files, compiled classes, and JaCoCo coverage reports
2. Runs static analysis (bug detection, vulnerability scanning, code smell detection)
3. Packages everything into an analysis report
4. Uploads the report to SonarQube via HTTP

```
┌─────────────────────────────────────────────┐
│            Your Maven Build                  │
│                                              │
│  src/main/java/**/*.java  ──┐                │
│  target/classes/**/*.class ──┤──▶ sonar-     │──── HTTP POST ──▶ SonarQube
│  target/site/jacoco-        │    maven-      │                   Server
│    aggregate/jacoco.xml  ───┘    plugin      │
│                                              │
└─────────────────────────────────────────────┘
```

---

## The Change: Parent POM

Add two things to the parent `pom.xml`:

### 1. Sonar properties (in `<properties>`)

```xml
<!-- Add these inside the existing <properties> block -->
<sonar.projectKey>linhvu_zalobot-sdk-java</sonar.projectKey>
<sonar.organization>linhvu</sonar.organization>
<sonar.host.url>https://sonarcloud.io</sonar.host.url>
<sonar.coverage.jacoco.xmlReportPaths>
    ${project.basedir}/target/site/jacoco-aggregate/jacoco.xml
</sonar.coverage.jacoco.xmlReportPaths>
```

> **Note:** Replace `linhvu_zalobot-sdk-java` and `linhvu` with the values you noted in Chapter 2-3. If using self-hosted, change `sonar.host.url` to `http://localhost:9000` and remove the `sonar.organization` property.

### 2. Sonar plugin declaration (in `<pluginManagement>`)

Add this inside the existing `<pluginManagement><plugins>` block (alongside JaCoCo from Chapter 4):

```xml
<plugin>
    <groupId>org.sonarsource.scanner.maven</groupId>
    <artifactId>sonar-maven-plugin</artifactId>
    <version>4.0.0.4121</version>
</plugin>
```

---

## What Each Property Means

| Property | Value | Purpose |
|---|---|---|
| `sonar.projectKey` | `linhvu_zalobot-sdk-java` | Unique identifier for your project on the server |
| `sonar.organization` | `linhvu` | SonarCloud organization (not needed for self-hosted) |
| `sonar.host.url` | `https://sonarcloud.io` | Where to upload the analysis report |
| `sonar.coverage.jacoco.xmlReportPaths` | `${project.basedir}/target/site/jacoco-aggregate/jacoco.xml` | Tells Sonar where to find the combined coverage report from Chapter 4 |

### Why `${project.basedir}` in the coverage path?

In a multi-module build, each module has its own `${project.basedir}`. But the aggregated report lives at the **parent** level. Using `${project.basedir}` from a child module would point to the wrong directory. For the aggregate report, you may need to adjust this path.

A more robust approach for multi-module projects:

```xml
<sonar.coverage.jacoco.xmlReportPaths>
    ${maven.multiModuleProjectDirectory}/target/site/jacoco-aggregate/jacoco.xml
</sonar.coverage.jacoco.xmlReportPaths>
```

`${maven.multiModuleProjectDirectory}` always resolves to the root of the multi-module project, regardless of which module is being processed.

---

## The Complete Parent POM After Chapters 4 + 5

For reference, here's what the full parent `pom.xml` looks like with both JaCoCo and Sonar:

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

    <modules>
        <module>zalobot-core</module>
        <module>zalobot-client</module>
        <module>zalobot-listener</module>
        <module>zalobot-spring-boot</module>
        <module>zalobot-spring-boot-starter</module>
    </modules>

    <properties>
        <maven.compiler.source>25</maven.compiler.source>
        <maven.compiler.target>25</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>

        <spring-boot.version>4.0.2</spring-boot.version>

        <okhttp.version>4.12.0</okhttp.version>
        <jackson-databind.version>3.0.4</jackson-databind.version>
        <junit-jupiter.version>6.0.2</junit-jupiter.version>

        <!-- SonarQube properties -->
        <sonar.projectKey>linhvu_zalobot-sdk-java</sonar.projectKey>
        <sonar.organization>linhvu</sonar.organization>
        <sonar.host.url>https://sonarcloud.io</sonar.host.url>
        <sonar.coverage.jacoco.xmlReportPaths>
            ${maven.multiModuleProjectDirectory}/target/site/jacoco-aggregate/jacoco.xml
        </sonar.coverage.jacoco.xmlReportPaths>
    </properties>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-dependencies</artifactId>
                <version>${spring-boot.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <build>
        <pluginManagement>
            <plugins>
                <plugin>
                    <groupId>org.jacoco</groupId>
                    <artifactId>jacoco-maven-plugin</artifactId>
                    <version>0.8.12</version>
                    <executions>
                        <execution>
                            <id>prepare-agent</id>
                            <goals>
                                <goal>prepare-agent</goal>
                            </goals>
                        </execution>
                        <execution>
                            <id>report</id>
                            <goals>
                                <goal>report</goal>
                            </goals>
                        </execution>
                    </executions>
                </plugin>
                <plugin>
                    <groupId>org.sonarsource.scanner.maven</groupId>
                    <artifactId>sonar-maven-plugin</artifactId>
                    <version>4.0.0.4121</version>
                </plugin>
            </plugins>
        </pluginManagement>

        <plugins>
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

</project>
```

---

## Running Your First Analysis

```bash
mvn clean verify sonar:sonar -Dsonar.token=sqp_595b032f4e2f9f2b35bcff36e2f3d3435bfbba8d
```

**What happens:**

1. `clean` — deletes all `target/` directories
2. `verify` — compiles, tests, generates JaCoCo coverage (all from Chapter 4)
3. `sonar:sonar` — reads everything and uploads to SonarQube

**Expected output** (look for these lines):

```
[INFO] --- sonar:4.0.0.4121:sonar (default-cli) @ zalobot-sdk-java ---
[INFO] User cache: /Users/you/.sonar/cache
[INFO] SonarQube version: ...
[INFO] Default locale: "en_US", source code encoding: "UTF-8"
...
[INFO] ANALYSIS SUCCESSFUL, you can find the results at:
[INFO]   https://sonarcloud.io/dashboard?id=linhvu_zalobot-sdk-java
[INFO] Note that you will be able to access the updated dashboard
       once the server has processed the submitted analysis report.
```

### First Time? Expect Some Issues

Your first analysis will likely show:
- **Low coverage** — only `zalobot-core` and `zalobot-spring-boot` have tests
- **Code smells** — SonarQube's default rules are opinionated
- **Possibly some bugs** — static analysis may find patterns you missed

This is normal and expected. The value of SonarQube is making these visible so you can decide what to address.

---

## Why Not `<plugins>` for Sonar?

Notice we put the Sonar plugin in `<pluginManagement>` only, not in `<plugins>`. This is intentional:

- **JaCoCo** is in both `<pluginManagement>` and `<plugins>` because it needs to run automatically during `mvn verify`
- **Sonar** is only in `<pluginManagement>` because you invoke it explicitly via `mvn sonar:sonar` — it never runs as part of the normal build lifecycle

This means `mvn clean verify` does NOT trigger Sonar analysis. You must explicitly add `sonar:sonar` to the command. This is the recommended approach — you don't want analysis running on every local build.

---

## Summary of Changes

| File | Change |
|---|---|
| `pom.xml` (parent) | Add 4 sonar properties to `<properties>` |
| `pom.xml` (parent) | Add sonar-maven-plugin to `<pluginManagement>` |

---

## Next: Chapter 6

We'll configure exclusions to skip analysis for modules and files that don't need it.
