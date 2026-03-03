# Chapter 5: Source and Javadoc JARs — Companion Artifacts for Consumers

> When you Ctrl+click a Spring class in IntelliJ and see the actual source code with comments, that's because Spring published a `-sources.jar`. Maven Central requires you to do the same.

---

## What Each JAR Contains

```
zalobot-client-0.0.1.jar              ← Compiled .class files (what the JVM runs)
zalobot-client-0.0.1-sources.jar      ← Original .java files (what the developer reads)
zalobot-client-0.0.1-javadoc.jar      ← Generated HTML documentation (what the IDE displays)
```

The main JAR is what Maven puts on the classpath. The sources and javadoc JARs are **never on the classpath** — they're downloaded by IDEs on demand:

```
IntelliJ / VS Code                       Maven Central
──────────────────                        ─────────────
User Ctrl+clicks ZaloBotClient.class
  │
  ├── IDE checks: do I have sources?
  │     └── NO → downloads zalobot-client-0.0.1-sources.jar
  │               from Maven Central
  │
  └── IDE opens ZaloBotClient.java from the sources JAR
      with full comments, formatting, and navigation
```

---

## Plugin 1: `maven-source-plugin`

This plugin packages your `src/main/java` directory into a `-sources.jar`.

```xml
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
```

The execution binds to the `package` phase by default. After `mvn package`, you'll see:

```
target/
├── zalobot-client-0.0.1.jar            ← main artifact
└── zalobot-client-0.0.1-sources.jar    ← NEW: source code
```

> **Insight: `jar-no-fork` vs `jar`**
>
> The `maven-source-plugin` offers two goals:
> - `jar` — forks a new Maven lifecycle execution to compile sources before packaging them. In a multi-module build, this can cause the `compile` phase to run **twice** for every module (once for the normal build, once for the forked execution).
> - `jar-no-fork` — packages sources without forking. It assumes compilation has already happened in the current lifecycle.
>
> Always use `jar-no-fork` in multi-module projects. With 5 modules, `jar` would cause 5 unnecessary re-compilations, roughly doubling your build time.

---

## Plugin 2: `maven-javadoc-plugin`

This plugin runs the `javadoc` tool on your source code and packages the generated HTML into a `-javadoc.jar`.

```xml
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
```

After `mvn package`, you'll see:

```
target/
├── zalobot-client-0.0.1.jar            ← main artifact
├── zalobot-client-0.0.1-sources.jar    ← source code
└── zalobot-client-0.0.1-javadoc.jar    ← NEW: API documentation
```

---

## The Special Case: `zalobot-spring-boot-starter`

The starter module has **no Java source code**. It's a POM-only module that declares dependencies:

```
zalobot-spring-boot-starter/
├── pom.xml          ← dependencies only
└── src/
    └── (empty — no Java code)
```

The `maven-source-plugin` handles this gracefully — it produces an empty sources JAR. But the `maven-javadoc-plugin` will **fail** because there are no Java files to process.

**Fix:** Add this property to `zalobot-spring-boot-starter/pom.xml`:

```xml
<properties>
    <!-- No Java code in this module — skip javadoc generation -->
    <maven.javadoc.skip>true</maven.javadoc.skip>
</properties>
```

This tells the `maven-javadoc-plugin` to skip this module entirely. The sources JAR will still be generated (empty, which is fine).

> **Insight: Empty JARs Are Valid**
>
> Maven Central accepts empty sources and javadoc JARs. A starter module with only a POM and transitive dependencies is a well-established pattern (look at any `spring-boot-starter-*`). The empty JARs satisfy the Central Portal's "artifact must exist" check without requiring actual content.
>
> However, some validation tools are stricter. Skipping javadoc generation entirely (with `maven.javadoc.skip`) is cleaner than producing a JAR that fails to generate.

---

## Where These Plugins Go

These plugins will be placed inside a `<profile>` (Chapter 9), not in the default build. This way, they only run when you explicitly activate the release profile:

```
mvn install                    → compiles + tests + packages (fast, no sources/javadoc)
mvn install -Prelease          → compiles + tests + packages + sources + javadoc (slower)
mvn deploy -Prelease           → all of the above + signing + uploading
```

Here's the preview of where they'll live in the parent POM:

```xml
<profiles>
    <profile>
        <id>release</id>
        <build>
            <plugins>
                <!-- Source JAR (this chapter) -->
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

                <!-- Javadoc JAR (this chapter) -->
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

                <!-- GPG signing (Chapter 6) -->
                <!-- Central Portal publishing (Chapter 7) -->
            </plugins>
        </build>
    </profile>
</profiles>
```

---

## Artifact Inventory Per Module

After `mvn package -Prelease`, each module produces:

| Module | Main JAR | Sources JAR | Javadoc JAR | POM |
|---|---|---|---|---|
| `zalobot-sdk-java` | — (POM packaging) | — | — | ✅ |
| `zalobot-core` | ✅ | ✅ | ✅ | ✅ |
| `zalobot-client` | ✅ | ✅ | ✅ | ✅ |
| `zalobot-listener` | ✅ | ✅ | ✅ | ✅ |
| `zalobot-spring-boot` | ✅ | ✅ | ✅ | ✅ |
| `zalobot-spring-boot-starter` | ✅ (empty) | ✅ (empty) | ❌ (skipped) | ✅ |

---

## Summary

| Concept | What It Means |
|---|---|
| `-sources.jar` | Contains `.java` files; IDEs download it for Ctrl+click navigation |
| `-javadoc.jar` | Contains HTML docs; IDEs download it for hover documentation |
| `jar-no-fork` | Source plugin goal that avoids re-running compile in multi-module builds |
| `maven.javadoc.skip` | Property to skip javadoc for modules with no Java source (like starters) |
| Profile-gated | Both plugins live inside `<profile><id>release</id>` — only run with `-Prelease` |

---

## What's Next?

We can generate JARs. But Maven Central also requires that every file is cryptographically signed. In Chapter 6, we learn about GPG signing — proving that these artifacts really came from you.

```
Chapter 4  ──► "Adding required POM metadata"
Chapter 5  ←── YOU ARE HERE: "Source and Javadoc JARs"
Chapter 6  ──► "GPG signing — proving you are you"
```
