# Chapter 10: Maven Wrapper — Reproducible Builds

> "Works on my machine" doesn't just apply to code — it applies to build tools. If you use Maven 3.9.9 and a contributor uses Maven 3.8.1, you might get different build results. The Maven Wrapper eliminates this.

---

## The Problem: Version Inconsistency

```
Your machine                          Contributor's machine
──────────────                        ────────────────────
Maven 3.9.9                           Maven 3.8.1 (or maybe not installed at all)
  │                                     │
  mvn deploy -Prelease                  mvn deploy -Prelease
  │                                     │
  └── Works ✓                           └── Different plugin resolution behavior
                                            Possibly different results ✗
```

Maven versions can differ in:
- Default plugin versions (each Maven version ships with different plugin defaults)
- Dependency resolution behavior
- Plugin compatibility

The fix: **ship Maven with your project**.

---

## The Solution: Maven Wrapper

The Maven Wrapper is a set of files committed to your repository that:
1. Checks if the correct Maven version is installed
2. If not, downloads it automatically
3. Runs Maven with that exact version

```
Instead of:  mvn deploy -Prelease          (uses whatever Maven is installed)
You use:     ./mvnw deploy -Prelease       (uses the exact Maven version you specified)
```

---

## Generating the Wrapper

```bash
mvn wrapper:wrapper -Dmaven=3.9.9
```

This creates:

```
project-root/
├── mvnw                    ← Unix/Mac shell script (executable)
├── mvnw.cmd                ← Windows batch script
└── .mvn/
    └── wrapper/
        └── maven-wrapper.properties    ← specifies Maven version + download URL
```

The `maven-wrapper.properties` file contains:

```properties
distributionUrl=https://repo.maven.apache.org/maven2/org/apache/maven/apache-maven/3.9.9/apache-maven-3.9.9-bin.zip
wrapperUrl=https://repo.maven.apache.org/maven2/org/apache/maven/wrapper/maven-wrapper/3.3.2/maven-wrapper-3.3.2.jar
```

> **Insight: No Maven Required**
>
> The beauty of the Maven Wrapper is that a contributor doesn't even need Maven installed. Running `./mvnw` will:
> 1. Check `~/.m2/wrapper/dists/` for the specified Maven version
> 2. If not found, download it from the `distributionUrl`
> 3. Cache it for future use
> 4. Run the build with that exact version
>
> This is why many open-source projects (including Spring Boot itself) use `./mvnw` in their build instructions instead of `mvn`.

---

## What to Commit

| File | Commit? | Why |
|---|---|---|
| `mvnw` | ✅ Yes | Unix/Mac launcher script |
| `mvnw.cmd` | ✅ Yes | Windows launcher script |
| `.mvn/wrapper/maven-wrapper.properties` | ✅ Yes | Specifies which Maven version to download |
| `.mvn/wrapper/maven-wrapper.jar` | ✅ Yes (if generated) | Older wrapper versions generate this; newer ones download it on-the-fly |

**Make `mvnw` executable:**

```bash
chmod +x mvnw
```

---

## Usage

Once committed, replace `mvn` with `./mvnw` everywhere:

| Before | After |
|---|---|
| `mvn install` | `./mvnw install` |
| `mvn test` | `./mvnw test` |
| `mvn verify -Prelease` | `./mvnw verify -Prelease` |
| `mvn deploy -Prelease` | `./mvnw deploy -Prelease` |

In CI/CD (Chapter 11), the workflow uses `./mvnw` instead of `mvn`:

```yaml
- name: Build and Deploy
  run: ./mvnw deploy -Prelease --batch-mode
```

---

## `.gitattributes` for Cross-Platform Line Endings

The `mvnw` script must have Unix line endings (LF) on Unix/Mac and the `mvnw.cmd` file needs Windows line endings (CRLF). Add a `.gitattributes` entry to prevent Git from changing line endings:

```
# Ensure wrapper scripts have correct line endings
mvnw text eol=lf
mvnw.cmd text eol=crlf
```

---

## Is This Required for Maven Central?

**No.** The Maven Wrapper is not a Central Portal requirement. You can publish with `mvn deploy -Prelease` using your system-installed Maven.

However, it's a strong best practice because:
- Contributors don't need to install a specific Maven version
- CI/CD uses the same Maven version you tested with locally
- Builds are reproducible across environments

> **Insight: Wrapper vs System Maven**
>
> The Maven Wrapper is similar to Gradle's wrapper (`gradlew`), Node's `nvm`, or Python's `pyenv`. The pattern is the same: version-lock your build tool alongside your code.
>
> For a project publishing to Maven Central, this is especially important because Maven plugin behavior can vary between versions. A plugin that works with Maven 3.9.9 might not resolve the same way with Maven 3.8.1.

---

## Summary

| Concept | What It Means |
|---|---|
| Maven Wrapper | Scripts (`mvnw`, `mvnw.cmd`) + properties file that pin a specific Maven version |
| `mvn wrapper:wrapper` | Command to generate wrapper files |
| `./mvnw` | Replaces `mvn` — downloads and uses the exact specified Maven version |
| `.mvn/wrapper/maven-wrapper.properties` | Specifies which Maven version to download |
| Not required for Central | Best practice for reproducibility, not a publishing requirement |

---

## What's Next?

We can build and deploy locally. In Chapter 11, we automate the entire process with GitHub Actions — so pushing a git tag triggers a deployment to Maven Central automatically.

```
Chapter 9  ──► "Putting it all together"
Chapter 10 ←── YOU ARE HERE: "Maven Wrapper for reproducible builds"
Chapter 11 ──► "Automated publishing with GitHub Actions"
```
