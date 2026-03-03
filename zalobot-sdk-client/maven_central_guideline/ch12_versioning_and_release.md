# Chapter 12: Versioning and Release Workflow — From Development to Published Release

> You've tagged `v0.0.1` and pushed it. GitHub Actions builds, signs, and deploys. 30 minutes later, any developer on Earth can add your SDK as a dependency. What happens next? How do you manage versions going forward?

---

## Semantic Versioning

Maven Central artifacts follow **Semantic Versioning** (SemVer):

```
MAJOR . MINOR . PATCH
  │       │       │
  │       │       └── Bug fixes, no API changes
  │       └────────── New features, backward-compatible
  └────────────────── Breaking changes, not backward-compatible
```

For `zalobot-sdk-java`:

| Version Bump | When | Example |
|---|---|---|
| `0.0.1` → `0.0.2` | Fix a bug in `ZaloBotClient` | Patch |
| `0.0.1` → `0.1.0` | Add support for file message types | Minor |
| `0.0.1` → `1.0.0` | Redesign the `UpdateListener` interface | Major |

> **Insight: The `0.x.y` Convention**
>
> Versions starting with `0.` (like `0.0.1`) signal "initial development — anything may change at any time." In the SemVer spec, major version zero is for initial development, and the public API should not be considered stable.
>
> This means consumers of `0.0.1` understand that `0.1.0` might contain breaking changes. Once you release `1.0.0`, you're making a stability promise: minor and patch updates will be backward-compatible.
>
> For a new SDK, starting at `0.0.1` is the right choice. Move to `1.0.0` when you're confident in the API design.

---

## SNAPSHOT vs Release Versions

```
0.0.2-SNAPSHOT                         0.0.2
──────────────                         ─────
Development version                    Release version
  - Mutable (can be overwritten)         - IMMUTABLE (can never change)
  - Not allowed on Maven Central         - Published to Maven Central permanently
  - Used during active development       - Used when tagging a release
  - Maven re-downloads on each build     - Maven caches after first download
```

The typical development cycle:

```
pom.xml version         Activity
───────────────         ────────
0.0.1                   Initial release → published to Maven Central
0.0.2-SNAPSHOT          Development of next version (local only)
0.0.2-SNAPSHOT          More development...
0.0.2                   Release → published to Maven Central
0.0.3-SNAPSHOT          Development continues...
```

> **Insight: Published Versions Are Immutable Forever**
>
> Once `0.0.1` is published to Maven Central, it can **never** be deleted, modified, or overwritten. This is by design:
> - Build reproducibility: if someone builds their project with your `0.0.1` today and again in 5 years, they should get the same artifact
> - Supply chain security: artifacts can't be silently replaced with malicious versions
>
> If you publish a broken version, you must publish a new version with the fix. You cannot "recall" a Maven Central release. This is why `autoPublish=false` is recommended for your first few deployments — it gives you a chance to inspect before the irreversible publish.

---

## Updating Version in a Multi-Module Project

**Never edit POM version numbers by hand.** Use the `versions-maven-plugin`:

```bash
# Set version across all 6 POMs (parent + 5 children)
mvn versions:set -DnewVersion=0.0.2-SNAPSHOT
```

This updates:
- `pom.xml` (root): `<version>0.0.2-SNAPSHOT</version>`
- `zalobot-core/pom.xml`: `<parent><version>0.0.2-SNAPSHOT</version></parent>`
- `zalobot-client/pom.xml`: `<parent><version>0.0.2-SNAPSHOT</version></parent>`
- `zalobot-listener/pom.xml`: `<parent><version>0.0.2-SNAPSHOT</version></parent>`
- `zalobot-spring-boot/pom.xml`: `<parent><version>0.0.2-SNAPSHOT</version></parent>`
- `zalobot-spring-boot-starter/pom.xml`: `<parent><version>0.0.2-SNAPSHOT</version></parent>`

The plugin also handles inter-module references like `${project.parent.version}`.

If something goes wrong:

```bash
# Revert version changes
mvn versions:revert

# Commit version changes (removes backup POM files)
mvn versions:commit
```

---

## The Complete Release Checklist

Here is the step-by-step release process for `zalobot-sdk-java`:

### Prerequisites (one-time setup)

- [ ] Central Portal account created at `central.sonatype.com`
- [ ] Namespace `dev.linhvu` verified via DNS
- [ ] API token generated and stored in GitHub Secrets
- [ ] GPG key generated, published to keyserver, private key in GitHub Secrets
- [ ] GitHub Actions workflow `.github/workflows/publish.yml` committed
- [ ] Maven Wrapper (`mvnw`) committed
- [ ] All POM metadata (name, license, developers, scm) in parent POM
- [ ] Release profile with source, javadoc, GPG, and central-publishing plugins

### Per-Release Checklist

```
1. PREPARE
   ├── Ensure all code changes are merged to main
   ├── Ensure all tests pass: ./mvnw verify
   ├── Update CHANGELOG.md (if you maintain one)
   └── Decide on the new version number (SemVer)

2. VERSION
   ├── Set release version: mvn versions:set -DnewVersion=X.Y.Z
   ├── Commit: git commit -am "Release vX.Y.Z"
   └── Verify: ./mvnw verify -Prelease (dry run — builds everything, doesn't upload)

3. TAG + PUSH
   ├── Tag: git tag vX.Y.Z
   ├── Push commit: git push origin main
   └── Push tag: git push origin vX.Y.Z
       └── This triggers GitHub Actions

4. MONITOR
   ├── Watch GitHub Actions: Repository → Actions tab
   ├── If CI fails: fix the issue, delete the tag, start over
   └── If CI succeeds: check Central Portal

5. PUBLISH (if autoPublish=false)
   ├── Go to central.sonatype.com → Deployments
   ├── Verify all validation checks pass
   ├── Click "Publish"
   └── Wait ~30 minutes for Maven Central sync

6. VERIFY
   ├── Check https://central.sonatype.com for your artifacts
   ├── Check https://repo.maven.apache.org/maven2/dev/linhvu/
   ├── Test in a fresh project:
   │     <dependency>
   │       <groupId>dev.linhvu</groupId>
   │       <artifactId>zalobot-spring-boot-starter</artifactId>
   │       <version>X.Y.Z</version>
   │     </dependency>
   └── Verify ./mvnw compile works in the test project

7. POST-RELEASE
   ├── Bump to next SNAPSHOT: mvn versions:set -DnewVersion=X.Y.(Z+1)-SNAPSHOT
   ├── Commit: git commit -am "Prepare for next development iteration"
   ├── Push: git push origin main
   └── Create GitHub Release (optional): gh release create vX.Y.Z --generate-notes
```

---

## Example: Releasing Version 0.0.1

```bash
# 1. Verify everything builds and tests pass
./mvnw verify -Prelease

# 2. Commit release version (already 0.0.1 in our case)
git add -A
git commit -m "Release v0.0.1"

# 3. Tag and push
git tag v0.0.1
git push origin main
git push origin v0.0.1

# 4. GitHub Actions triggers automatically
#    Monitor at: https://github.com/<YOUR_GITHUB_USERNAME>/zalobot-sdk-java/actions

# 5. After CI succeeds, go to central.sonatype.com
#    Click "Publish" on the validated deployment

# 6. Wait ~30 minutes, then verify:
#    https://repo.maven.apache.org/maven2/dev/linhvu/zalobot-client/0.0.1/

# 7. Bump to next development version
mvn versions:set -DnewVersion=0.0.2-SNAPSHOT
git commit -am "Prepare for next development iteration"
git push origin main
```

---

## Recovering from Mistakes

| Situation | Recovery |
|---|---|
| Tagged wrong commit | `git tag -d vX.Y.Z && git push origin --delete vX.Y.Z` then re-tag |
| CI failed but tag exists | Fix the issue, delete and re-create the tag |
| Published with a bug | **Cannot undo.** Publish `X.Y.(Z+1)` with the fix |
| Wrong version number published | **Cannot undo.** Skip to the correct version in the next release |
| `autoPublish=false` and you see an issue | Click "Drop" in Central Portal before clicking "Publish" |

---

## Version History Visualization

```
Time →

main branch:  ─── 0.0.1-SNAPSHOT ─── 0.0.1 ─── 0.0.2-SNAPSHOT ─── 0.0.2 ─── 0.1.0-SNAPSHOT ───
                                       │                            │
                                       └── tag: v0.0.1              └── tag: v0.0.2
                                       └── published                └── published

Maven Central:    ─────────────────── 0.0.1 ──────────────────── 0.0.2 ──────────────────────────
                                    (immutable)                 (immutable)
```

---

## Summary

| Concept | What It Means |
|---|---|
| Semantic Versioning | MAJOR.MINOR.PATCH — communicates the nature of changes |
| SNAPSHOT | Development version, mutable, not publishable to Central |
| Release version | Immutable, published forever on Maven Central |
| `versions:set` | Updates version across all 6 POMs in one command |
| Release checklist | Prepare → Version → Tag → Monitor → Publish → Verify → Post-release |
| Immutability | Published versions can never be deleted or modified |
| `0.x.y` | Initial development — API stability not guaranteed |

---

## Congratulations!

You've completed the entire Maven Central publishing guide. Here's what you've learned:

```
Chapter 1  ── The Problem: JARs trapped locally
Chapter 2  ── How Maven finds artifacts (coordinates + repositories)
Chapter 3  ── What Maven Central demands (complete checklist)
Chapter 4  ── POM metadata (name, license, developers, scm)
Chapter 5  ── Source and Javadoc JARs (companion artifacts)
Chapter 6  ── GPG signing (cryptographic proof of authorship)
Chapter 7  ── Central Portal (the publishing platform)
Chapter 8  ── settings.xml (credential management)
Chapter 9  ── The Deploy (release profile consolidation)
Chapter 10 ── Maven Wrapper (reproducible builds)
Chapter 11 ── GitHub Actions CI/CD (automated publishing)
Chapter 12 ── Versioning and Release (SemVer + workflow)
```

From `mvn install` (local only) to `mvn deploy -Prelease` (global distribution) — your JARs are no longer trapped.
