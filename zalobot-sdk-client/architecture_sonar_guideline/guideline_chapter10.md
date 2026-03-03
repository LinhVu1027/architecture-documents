# Chapter 10: Complete Configuration Reference

> **Type:** IN-PROJECT — consolidated reference of all changes from Chapters 4-9

---

## Final Parent POM (`pom.xml`)

This is the complete parent `pom.xml` with all additions from Chapters 4, 5, 6, and 7 (profile optional). Changes from the original are marked with comments.

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

        <!-- ========== NEW: SonarQube properties (Chapter 5) ========== -->
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

    <!-- ========== NEW: Build section (Chapters 4 + 5) ========== -->
    <build>
        <pluginManagement>
            <plugins>
                <!-- Chapter 4: JaCoCo for code coverage -->
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
                <!-- Chapter 5: Sonar Maven Plugin -->
                <plugin>
                    <groupId>org.sonarsource.scanner.maven</groupId>
                    <artifactId>sonar-maven-plugin</artifactId>
                    <version>4.0.0.4121</version>
                </plugin>
            </plugins>
        </pluginManagement>

        <plugins>
            <!-- Chapter 4: Activate JaCoCo for all modules -->
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

## Final Starter Module POM (`zalobot-spring-boot-starter/pom.xml`)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>dev.linhvu</groupId>
        <artifactId>zalobot-sdk-java</artifactId>
        <version>0.0.1</version>
    </parent>

    <artifactId>zalobot-spring-boot-starter</artifactId>

    <properties>
        <maven.compiler.source>25</maven.compiler.source>
        <maven.compiler.target>25</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <!-- ========== NEW: Skip JaCoCo and Sonar (Chapters 4 + 6) ========== -->
        <jacoco.skip>true</jacoco.skip>
        <sonar.skip>true</sonar.skip>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>

        <dependency>
            <groupId>dev.linhvu</groupId>
            <artifactId>zalobot-spring-boot</artifactId>
            <version>${project.parent.version}</version>
        </dependency>
    </dependencies>
</project>
```

---

## GitHub Actions Workflow (`.github/workflows/sonar.yml`)

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
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Set up JDK 25
        uses: actions/setup-java@v4
        with:
          java-version: '25'
          distribution: 'temurin'

      - name: Cache Maven packages
        uses: actions/cache@v4
        with:
          path: ~/.m2/repository
          key: ${{ runner.os }}-maven-${{ hashFiles('**/pom.xml') }}
          restore-keys: |
            ${{ runner.os }}-maven-

      - name: Cache SonarQube packages
        uses: actions/cache@v4
        with:
          path: ~/.sonar/cache
          key: ${{ runner.os }}-sonar
          restore-keys: |
            ${{ runner.os }}-sonar

      - name: Build and Analyze
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
        run: >
          mvn -B clean verify sonar:sonar
          -Dsonar.token=${{ secrets.SONAR_TOKEN }}
```

---

## Complete Checklist

### In-Project Steps (committed to Git)

- [ ] **Ch4:** Add JaCoCo plugin to parent POM `<pluginManagement>` with `prepare-agent` and `report` executions
- [ ] **Ch4:** Add JaCoCo to parent POM `<plugins>` with `prepare-agent`, `report`, and `report-aggregate` executions
- [ ] **Ch4:** Add `<jacoco.skip>true</jacoco.skip>` to `zalobot-spring-boot-starter/pom.xml`
- [ ] **Ch5:** Add Sonar properties to parent POM `<properties>` (`sonar.projectKey`, `sonar.organization`, `sonar.host.url`, `sonar.coverage.jacoco.xmlReportPaths`)
- [ ] **Ch5:** Add sonar-maven-plugin to parent POM `<pluginManagement>`
- [ ] **Ch6:** Add `<sonar.skip>true</sonar.skip>` to `zalobot-spring-boot-starter/pom.xml`
- [ ] **Ch9:** Create `.github/workflows/sonar.yml`

### Outside-Project Steps (server/UI configuration)

- [ ] **Ch2:** Choose SonarCloud or self-hosted
- [ ] **Ch2:** Create account / start server
- [ ] **Ch3:** Create SonarQube project
- [ ] **Ch3:** Generate authentication token
- [ ] **Ch3:** Store token in `~/.m2/settings.xml` (local dev)
- [ ] **Ch8:** Review/customize Quality Gate (consider 60% coverage threshold for new code)
- [ ] **Ch8:** Set New Code Period to "Reference Branch" = `main`
- [ ] **Ch8:** Review Quality Profile (start with "Sonar way")
- [ ] **Ch9:** Add `SONAR_TOKEN` to GitHub repository secrets
- [ ] **Ch9:** Configure SonarCloud GitHub integration for PR decoration

---

## Common Commands Quick Reference

```bash
# Build and test (no Sonar analysis)
mvn clean verify

# Build, test, and analyze (with token from command line)
mvn clean verify sonar:sonar -Dsonar.token=sqa_YOUR_TOKEN

# Build, test, and analyze (with token from ~/.m2/settings.xml)
mvn clean verify sonar:sonar

# Check if JaCoCo report was generated
ls target/site/jacoco-aggregate/jacoco.xml

# Verify Sonar token is valid
curl -s -u sqa_YOUR_TOKEN: https://sonarcloud.io/api/authentication/validate

# View project status via API
curl -s "https://sonarcloud.io/api/qualitygates/project_status?projectKey=linhvu_zalobot-sdk-java" \
  -H "Authorization: Bearer sqa_YOUR_TOKEN"
```

---

## Files Changed Summary

| File | Chapter | Type of Change |
|---|---|---|
| `pom.xml` | Ch4, Ch5 | Add `<build>` section + sonar properties |
| `zalobot-spring-boot-starter/pom.xml` | Ch4, Ch6 | Add `jacoco.skip` + `sonar.skip` properties |
| `.github/workflows/sonar.yml` | Ch9 | New file — CI/CD workflow |

Total: **2 files modified, 1 file created.**

---

## Verification Steps

1. **JaCoCo works:**
   ```bash
   mvn clean verify
   # Check: target/site/jacoco-aggregate/jacoco.xml exists
   ```

2. **Sonar analysis uploads:**
   ```bash
   mvn clean verify sonar:sonar -Dsonar.token=sqa_YOUR_TOKEN
   # Check: "ANALYSIS SUCCESSFUL" in output
   # Check: dashboard at sonarcloud.io shows data
   ```

3. **CI/CD works:**
   ```
   Push to main → GitHub Actions runs → SonarCloud dashboard updates
   Open PR → GitHub Actions runs → PR gets Quality Gate status check
   ```

4. **Quality Gate enforced:**
   ```
   PR with insufficient coverage → Quality Gate fails → status check shows ✗
   PR with good quality → Quality Gate passes → status check shows ✓
   ```
