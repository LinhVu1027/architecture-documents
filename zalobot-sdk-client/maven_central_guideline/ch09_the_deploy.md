# Chapter 9: The Deploy — Putting It All Together

> Chapters 4-8 added individual pieces. Now we consolidate everything into one parent POM with a `release` profile that transforms `mvn deploy -Prelease` into a full publishing pipeline.

---

## The Release Profile Strategy

```
Without -Prelease (normal development):
  mvn install
    compile → test → package → install
    Time: fast
    Output: JAR in ~/.m2/repository/

With -Prelease (publishing):
  mvn deploy -Prelease
    compile → test → package → source:jar → javadoc:jar → gpg:sign → deploy
    Time: slower (javadoc generation, signing, uploading)
    Output: artifacts on Maven Central
```

> **Insight: Profile-Gated Publishing**
>
> By wrapping all publishing plugins in a Maven profile, you get two completely different build experiences from the same POM:
> - Developers run `mvn install` dozens of times a day — fast, no signing, no uploading
> - The release process runs `mvn deploy -Prelease` once per release — full pipeline
>
> Without profiles, every `mvn install` would try to generate javadoc, sign artifacts, and fail because GPG isn't configured. Profiles are the standard pattern used by Spring Boot, Apache projects, and virtually every library published to Maven Central.

---

## The Complete Parent POM

Here is the full `pom.xml` with ALL changes from Chapters 4-8 consolidated:

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

    <!-- ==================== METADATA (Chapter 4) ==================== -->
    <name>ZaloBot SDK for Java</name>
    <description>Java SDK for the Zalo Bot API — client, listener, and Spring Boot integration</description>
    <url>https://github.com/<YOUR_GITHUB_USERNAME>/zalobot-sdk-java</url>

    <licenses>
        <license>
            <name>MIT License</name>
            <url>https://opensource.org/licenses/MIT</url>
            <distribution>repo</distribution>
        </license>
    </licenses>

    <developers>
        <developer>
            <name>Linh Vu</name>
            <url>https://github.com/<YOUR_GITHUB_USERNAME></url>
        </developer>
    </developers>

    <scm>
        <connection>scm:git:git://github.com/<YOUR_GITHUB_USERNAME>/zalobot-sdk-java.git</connection>
        <developerConnection>scm:git:ssh://github.com/<YOUR_GITHUB_USERNAME>/zalobot-sdk-java.git</developerConnection>
        <url>https://github.com/<YOUR_GITHUB_USERNAME>/zalobot-sdk-java</url>
    </scm>
    <!-- ==================== END METADATA ==================== -->

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
        <assertj-core.version>3.27.7</assertj-core.version>
        <mockito-core.version>5.21.0</mockito-core.version>
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

    <!-- ==================== RELEASE PROFILE (Chapters 5-8) ==================== -->
    <profiles>
        <profile>
            <id>release</id>
            <build>
                <plugins>
                    <!-- Source JAR (Chapter 5) -->
                    <plugin>
                        <groupId>org.apache.maven.plugins</groupId>
                        <artifactId>maven-source-plugin</artifactId>
                        <version>3.3.1</version>
                        <executions>
                            <execution>
                                <id>attach-sources</id>
                                <goals>
                                    <goal>jar-no-fork</goal>
                                </goals>
                            </execution>
                        </executions>
                    </plugin>

                    <!-- Javadoc JAR (Chapter 5) -->
                    <plugin>
                        <groupId>org.apache.maven.plugins</groupId>
                        <artifactId>maven-javadoc-plugin</artifactId>
                        <version>3.11.2</version>
                        <executions>
                            <execution>
                                <id>attach-javadocs</id>
                                <goals>
                                    <goal>jar</goal>
                                </goals>
                            </execution>
                        </executions>
                    </plugin>

                    <!-- GPG Signing (Chapter 6) -->
                    <plugin>
                        <groupId>org.apache.maven.plugins</groupId>
                        <artifactId>maven-gpg-plugin</artifactId>
                        <version>3.2.7</version>
                        <executions>
                            <execution>
                                <id>sign-artifacts</id>
                                <phase>verify</phase>
                                <goals>
                                    <goal>sign</goal>
                                </goals>
                            </execution>
                        </executions>
                    </plugin>

                    <!-- Central Portal Publishing (Chapter 7) -->
                    <plugin>
                        <groupId>org.sonatype.central</groupId>
                        <artifactId>central-publishing-maven-plugin</artifactId>
                        <version>0.7.0</version>
                        <extensions>true</extensions>
                        <configuration>
                            <publishingServerId>central</publishingServerId>
                            <autoPublish>false</autoPublish>
                        </configuration>
                    </plugin>
                </plugins>
            </build>
        </profile>
    </profiles>
    <!-- ==================== END RELEASE PROFILE ==================== -->

</project>
```

---

## The Starter Module Override

Remember from Chapter 5: `zalobot-spring-boot-starter` has no Java code. Add this to `zalobot-spring-boot-starter/pom.xml`:

```xml
<properties>
    <maven.compiler.source>25</maven.compiler.source>
    <maven.compiler.target>25</maven.compiler.target>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <!-- No Java code in this module — skip javadoc generation -->
    <maven.javadoc.skip>true</maven.javadoc.skip>
</properties>
```

---

## The Complete Lifecycle Trace

Here's exactly what happens when you run `mvn deploy -Prelease`:

```
$ mvn deploy -Prelease

[INFO] Reactor Build Order:
[INFO]   zalobot-sdk-java (parent)
[INFO]   zalobot-core
[INFO]   zalobot-client
[INFO]   zalobot-listener
[INFO]   zalobot-spring-boot
[INFO]   zalobot-spring-boot-starter

═══════════════════════════════════════════
[1/6] zalobot-sdk-java (parent POM)
═══════════════════════════════════════════
  No compile/test/package (packaging=pom)
  gpg:sign     → zalobot-sdk-java-0.0.1.pom.asc
  deploy       → collected for bundle

═══════════════════════════════════════════
[2/6] zalobot-core
═══════════════════════════════════════════
  compile      → src/main/java → target/classes
  test         → run unit tests
  package      → target/zalobot-core-0.0.1.jar
  source:jar   → target/zalobot-core-0.0.1-sources.jar
  javadoc:jar  → target/zalobot-core-0.0.1-javadoc.jar
  gpg:sign     → 4 .asc files (jar, pom, sources, javadoc)
  deploy       → collected for bundle

═══════════════════════════════════════════
[3/6] zalobot-client
═══════════════════════════════════════════
  compile      → target/classes
  test         → run unit tests
  package      → target/zalobot-client-0.0.1.jar
  source:jar   → target/zalobot-client-0.0.1-sources.jar
  javadoc:jar  → target/zalobot-client-0.0.1-javadoc.jar
  gpg:sign     → 4 .asc files
  deploy       → collected for bundle

═══════════════════════════════════════════
[4/6] zalobot-listener
═══════════════════════════════════════════
  (same as zalobot-client)

═══════════════════════════════════════════
[5/6] zalobot-spring-boot
═══════════════════════════════════════════
  (same as zalobot-client)

═══════════════════════════════════════════
[6/6] zalobot-spring-boot-starter
═══════════════════════════════════════════
  package      → target/zalobot-spring-boot-starter-0.0.1.jar (nearly empty)
  source:jar   → target/...-sources.jar (empty)
  javadoc:jar  → SKIPPED (maven.javadoc.skip=true)
  gpg:sign     → 3 .asc files (jar, pom, sources — no javadoc)
  deploy       → collected for bundle

═══════════════════════════════════════════
AFTER ALL MODULES:
═══════════════════════════════════════════
  central-publishing-maven-plugin:
    → Bundles ~44 files
    → Uploads to central.sonatype.com
    → Validation runs
    → Status: VALIDATED (awaiting manual publish)
```

---

## Dry Run: Verify Before Deploying

Before running `mvn deploy`, do a dry run with `mvn verify`:

```bash
# Dry run: everything except the actual upload
mvn verify -Prelease
```

This runs the **entire pipeline** (compile, test, package, sources, javadoc, GPG sign) but stops before the `deploy` phase. If `verify` succeeds, `deploy` will produce the same artifacts and upload them.

Check the output:
- Are all tests passing?
- Are sources JARs generated?
- Are javadoc JARs generated (except starter)?
- Are `.asc` files present in each module's `target/`?

```bash
# Check that signatures were created
find . -name "*.asc" -path "*/target/*"

# Expected output:
# ./zalobot-core/target/zalobot-core-0.0.1.jar.asc
# ./zalobot-core/target/zalobot-core-0.0.1.pom.asc
# ./zalobot-core/target/zalobot-core-0.0.1-sources.jar.asc
# ./zalobot-core/target/zalobot-core-0.0.1-javadoc.jar.asc
# ... (similar for other modules)
```

---

## Mapping: Chapter → POM Location

| Chapter | What Was Added | POM Location |
|---|---|---|
| Ch 4 | `<name>`, `<description>`, `<url>`, `<licenses>`, `<developers>`, `<scm>` | Root POM, after `<packaging>` |
| Ch 5 | `maven-source-plugin`, `maven-javadoc-plugin` | Root POM, inside `<profiles><profile><id>release</id>` |
| Ch 5 | `<maven.javadoc.skip>true</maven.javadoc.skip>` | `zalobot-spring-boot-starter/pom.xml` `<properties>` |
| Ch 6 | `maven-gpg-plugin` | Root POM, inside `<profiles><profile><id>release</id>` |
| Ch 7 | `central-publishing-maven-plugin` | Root POM, inside `<profiles><profile><id>release</id>` |
| Ch 8 | Credentials | `~/.m2/settings.xml` (not in POM) |

---

## Summary

| Concept | What It Means |
|---|---|
| `release` profile | Maven profile containing all publishing plugins — activated with `-Prelease` |
| `mvn verify -Prelease` | Dry run — full pipeline without uploading |
| `mvn deploy -Prelease` | Full pipeline + upload to Central Portal |
| Profile-gated | Normal `mvn install` is unaffected — publishing only happens explicitly |
| Single parent POM | All 4 plugins configured once; all 5 child modules inherit |
| `autoPublish=false` | First deployments require manual approval at central.sonatype.com |

---

## What's Next?

The publishing pipeline is complete. In Chapter 10, we add the Maven Wrapper — ensuring every developer and CI/CD system uses the same Maven version for reproducible builds.

```
Chapter 8  ──► "Keeping secrets out of your POM"
Chapter 9  ←── YOU ARE HERE: "Putting it all together"
Chapter 10 ──► "Maven Wrapper for reproducible builds"
```
