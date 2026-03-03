# Chapter 8: Settings XML — Keeping Secrets Out of Your POM

> Your `pom.xml` is committed to git. Your passwords, tokens, and passphrases must never be. Maven's `settings.xml` is the bridge between public configuration and private credentials.

---

## The Two Worlds

```
┌──────────────────────────────────┐     ┌──────────────────────────────────┐
│         pom.xml (PUBLIC)         │     │     settings.xml (PRIVATE)       │
│                                  │     │                                  │
│  ✓ Committed to git              │     │  ✗ NEVER committed to git        │
│  ✓ Shared with all developers    │     │  ✗ Lives in ~/.m2/ only          │
│  ✓ Defines WHAT to do            │     │  ✓ Defines HOW to authenticate   │
│                                  │     │                                  │
│  Plugin config:                  │     │  Credentials:                    │
│  <publishingServerId>            │     │  <server>                        │
│    central ────────────┐         │     │    <id>central</id> ◄────────────┘
│  </publishingServerId> │         │     │    <username>aB1cD2eF</username>
│                        │ links   │     │    <password>xY9/zW8+...</password>
│                        └─────────┼────►│  </server>
│                                  │     │                                  │
└──────────────────────────────────┘     └──────────────────────────────────┘
```

> **Insight: The `<id>` Bridge**
>
> The `<id>` string is the critical link between POM and settings.xml:
> - In `pom.xml`: `<publishingServerId>central</publishingServerId>` — says "use credentials with id `central`"
> - In `settings.xml`: `<server><id>central</id>...</server>` — provides those credentials
>
> This is the same mechanism used for all Maven server authentication: private registries, Nexus servers, GitHub Packages, etc. The pattern is always: POM declares the server ID, settings.xml provides the credentials.

---

## The Complete `settings.xml`

Create or edit `~/.m2/settings.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<settings xmlns="http://maven.apache.org/SETTINGS/1.2.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.2.0
          https://maven.apache.org/xsd/settings-1.2.0.xsd">

    <servers>
        <!-- Central Portal credentials (from Chapter 7, Step 3) -->
        <server>
            <id>central</id>
            <username>aB1cD2eF</username>
            <password>xY9/zW8+vU7tS6rQ5pO4nM3lK2jI1hG0</password>
        </server>
    </servers>

    <profiles>
        <profile>
            <id>gpg</id>
            <properties>
                <!-- GPG passphrase (from Chapter 6) -->
                <gpg.passphrase>your-gpg-passphrase-here</gpg.passphrase>
            </properties>
        </profile>
    </profiles>

    <activeProfiles>
        <activeProfile>gpg</activeProfile>
    </activeProfiles>

</settings>
```

---

## Section-by-Section Breakdown

### `<servers>` — Authentication Credentials

```xml
<servers>
    <server>
        <id>central</id>
        <username>aB1cD2eF</username>
        <password>xY9/zW8+vU7tS6rQ5pO4nM3lK2jI1hG0</password>
    </server>
</servers>
```

| Field | Value | Source |
|---|---|---|
| `<id>` | `central` | Must match `<publishingServerId>central</publishingServerId>` in pom.xml |
| `<username>` | Central Portal generated username | From Central Portal → Account → Generate User Token |
| `<password>` | Central Portal generated token | From Central Portal → Account → Generate User Token |

**These are NOT your Central Portal login credentials.** They are a generated token pair specifically for the API.

### `<profiles>` — GPG Passphrase

```xml
<profiles>
    <profile>
        <id>gpg</id>
        <properties>
            <gpg.passphrase>your-gpg-passphrase-here</gpg.passphrase>
        </properties>
    </profile>
</profiles>

<activeProfiles>
    <activeProfile>gpg</activeProfile>
</activeProfiles>
```

The `maven-gpg-plugin` reads the `gpg.passphrase` property to unlock your private key. By putting it in a profile that's always active, you don't need to pass it on the command line.

---

## How It All Connects

```
pom.xml                           settings.xml                    External Service
───────                           ────────────                    ────────────────

central-publishing-maven-plugin    <server>
  <publishingServerId>              <id>central</id>──────────────► central.sonatype.com
    central ────────────────────►   <username>aB1cD2eF</username>    authenticates with
  </publishingServerId>             <password>xY9/zW8+</password>    these credentials
                                  </server>

maven-gpg-plugin                  <profiles>
  runs gpg --sign                   <profile><id>gpg</id>
  needs passphrase ◄──────────────   <gpg.passphrase>...           ► gpg command
                                    </gpg.passphrase>                unlocks private key
                                  </profile>
```

---

## Security Checklist

| Check | Status |
|---|---|
| `settings.xml` is in `~/.m2/` (not in project directory) | ✅ |
| `settings.xml` is NOT in `.gitignore` (it's outside the repo) | ✅ |
| `pom.xml` contains NO passwords, tokens, or passphrases | ✅ |
| Central Portal token is generated (not your login password) | ✅ |
| File permissions: `chmod 600 ~/.m2/settings.xml` | ✅ |

```bash
# Set restrictive permissions on settings.xml
chmod 600 ~/.m2/settings.xml
# Only your user can read/write it
```

> **Insight: Encrypted Passwords in settings.xml**
>
> Maven supports encrypting passwords in `settings.xml` using `mvn --encrypt-password`. This stores an encrypted string instead of plaintext:
> ```xml
> <password>{jSMOWnoPFgsHVpMvz5VrIt5kRbzGpI8u+9EF1iFQyJQ=}</password>
> ```
> While better than plaintext, this is "security through obscurity" — the master password is stored in `~/.m2/settings-security.xml`, which is also on disk. For local development, the practical security benefit is marginal. For CI/CD, environment variables (Chapter 11) are the proper solution.

---

## Verifying Your Setup

Before attempting a real deploy, verify that Maven can read your settings:

```bash
# Check that Maven sees your settings.xml
mvn help:effective-settings

# Look for:
# <server>
#   <id>central</id>
#   <username>aB1cD2eF</username>
#   <password>***</password>  (masked in output)
# </server>
```

If the server with id `central` doesn't appear, Maven can't find your `settings.xml`. Check the file location (`~/.m2/settings.xml`) and XML validity.

---

## CI/CD: A Different Approach to Credentials

In CI/CD (Chapter 11), you don't use a `settings.xml` file on disk. Instead:
- GitHub Actions' `setup-java` action generates `settings.xml` from secrets
- GPG passphrase comes from `${{ secrets.GPG_PASSPHRASE }}`
- Central Portal credentials come from `${{ secrets.MAVEN_CENTRAL_USERNAME }}` and `${{ secrets.MAVEN_CENTRAL_PASSWORD }}`

The `<id>` bridging mechanism stays the same — only the source of credentials changes.

---

## Summary

| Concept | What It Means |
|---|---|
| `settings.xml` | Private configuration file at `~/.m2/settings.xml` — never committed |
| `<server>` | Provides credentials for a specific server ID |
| `<id>central</id>` | Links to `<publishingServerId>central</publishingServerId>` in pom.xml |
| API token | Generated username/password pair from Central Portal (not your login) |
| `gpg.passphrase` | Property that `maven-gpg-plugin` reads to unlock your private key |
| `chmod 600` | Restrict settings.xml so only your user can read it |

---

## What's Next?

We have metadata (Ch 4), companion JARs (Ch 5), signing (Ch 6), the publishing plugin (Ch 7), and credentials (Ch 8). In Chapter 9, we wire it all together into a single `release` profile in the parent POM.

```
Chapter 7  ──► "The Central Portal publishing mechanism"
Chapter 8  ←── YOU ARE HERE: "Keeping secrets out of your POM"
Chapter 9  ──► "Putting it all together"
```
