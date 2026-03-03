# Chapter 7: Maven Profiles for Sonar

> **Type:** IN-PROJECT — optional change to the parent `pom.xml`

---

## The Question: Profile or Explicit Goal?

There are two ways to make SonarQube analysis opt-in:

### Approach A: Explicit Goal (Current Setup)

```bash
# Regular build — no Sonar
mvn clean verify

# Build with Sonar analysis
mvn clean verify sonar:sonar -Dsonar.token=xxx
```

The `sonar:sonar` goal is only invoked when you explicitly add it to the command. This is what we set up in Chapter 5.

### Approach B: Maven Profile

```bash
# Regular build — no Sonar
mvn clean verify

# Build with Sonar analysis via profile
mvn clean verify -Psonar -Dsonar.token=xxx
```

A Maven profile wraps the Sonar plugin activation so you use `-Psonar` instead of adding `sonar:sonar` as a goal.

---

## Comparison

| Criteria | Explicit Goal | Maven Profile |
|---|---|---|
| Simplicity | Simpler POM (no profile needed) | More POM configuration |
| Discoverability | Must know to add `sonar:sonar` | `mvn help:all-profiles` shows it |
| CI/CD | Straightforward | Slightly cleaner in workflow files |
| Flexibility | Can't add extra configuration per-profile | Can override properties, add plugins |
| Community convention | Most common approach | Used in larger organizations |

**Recommendation:** For `zalobot-sdk-java`, the explicit goal approach (Approach A) is sufficient. The profile approach is shown below for educational purposes, but it adds configuration without clear benefit for a project this size.

---

## If You Want a Profile: Here's How

Add this to the parent `pom.xml`, as a sibling of `<build>`:

```xml
<profiles>
    <profile>
        <id>sonar</id>
        <build>
            <plugins>
                <plugin>
                    <groupId>org.sonarsource.scanner.maven</groupId>
                    <artifactId>sonar-maven-plugin</artifactId>
                    <executions>
                        <execution>
                            <id>sonar-analysis</id>
                            <phase>verify</phase>
                            <goals>
                                <goal>sonar</goal>
                            </goals>
                        </execution>
                    </executions>
                </plugin>
            </plugins>
        </build>
    </profile>
</profiles>
```

### What this does:

1. Defines a profile named `sonar`
2. When activated (`-Psonar`), it binds the `sonar:sonar` goal to the `verify` phase
3. This means `mvn clean verify -Psonar` automatically runs Sonar analysis at the end of `verify`

### Usage:

```bash
# Activate the profile
mvn clean verify -Psonar -Dsonar.token=xxx

# The profile can also be activated by an environment variable
# (useful in CI/CD — covered in Chapter 9)
```

---

## Profile with Property Override

A more advanced use case: the profile can override Sonar properties for different environments.

```xml
<profiles>
    <profile>
        <id>sonar-cloud</id>
        <properties>
            <sonar.host.url>https://sonarcloud.io</sonar.host.url>
            <sonar.organization>linhvu</sonar.organization>
        </properties>
    </profile>
    <profile>
        <id>sonar-local</id>
        <properties>
            <sonar.host.url>http://localhost:9000</sonar.host.url>
            <!-- No organization needed for self-hosted -->
            <sonar.organization></sonar.organization>
        </properties>
    </profile>
</profiles>
```

Usage:
```bash
# Analyze against SonarCloud
mvn clean verify sonar:sonar -Psonar-cloud -Dsonar.token=xxx

# Analyze against local server
mvn clean verify sonar:sonar -Psonar-local -Dsonar.token=xxx
```

This is useful if you run both SonarCloud (for CI) and a local SonarQube instance (for development).

---

## When Profiles Make Sense

Profiles become valuable when:

1. **Multiple environments** — You analyze against different servers (dev, staging, production)
2. **Different configurations per environment** — Different exclusions, different quality gates
3. **Complex plugin chains** — The profile activates multiple plugins together (e.g., SpotBugs + SonarQube)
4. **Team convention** — The team expects `-P` flags for optional build features

For a single-server setup like `zalobot-sdk-java`, the explicit `sonar:sonar` goal is cleaner.

---

## Summary

| Approach | When to Use | Command |
|---|---|---|
| Explicit goal (recommended) | Single server, simple setup | `mvn clean verify sonar:sonar` |
| Maven profile | Multiple servers, team conventions | `mvn clean verify -Psonar` |

No changes are required for this chapter if you stick with the explicit goal approach from Chapter 5.

---

## Next: Chapter 8

We switch back to outside-project work: configuring Quality Gates and Quality Profiles on the SonarQube server.
