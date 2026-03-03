# Chapter 6: GPG Signing — Proving You Are You

> When a developer downloads `zalobot-client-0.0.1.jar`, how do they know it was really published by you and not a malicious actor? GPG signatures solve this.

---

## The Problem: Supply Chain Trust

```
Scenario: Man-in-the-middle attack

Developer                   Attacker                    Maven Central
─────────                   ────────                    ─────────────
                            Gains access to the
                            publishing credentials
                            │
                            ├── Uploads malicious
                            │   zalobot-client-0.0.1.jar
                            │   (looks identical)
                            │
Consumer downloads JAR ←────┘
  └── Malicious code runs in their production app
```

**Without GPG signing:** No way to detect this. The JAR has the right coordinates and the right checksums (the attacker generated new ones).

**With GPG signing:** The attacker can't produce a valid `.asc` signature without your private key. The signature verification fails, and the consumer knows the artifact was tampered with.

---

## Asymmetric Cryptography in 30 Seconds

```
GPG Key Pair
├── PRIVATE KEY (secret — never leaves your machine / CI secrets)
│     └── Used to SIGN artifacts
│         artifact.jar + private key → artifact.jar.asc (signature)
│
└── PUBLIC KEY (published to a keyserver — anyone can download)
      └── Used to VERIFY signatures
          artifact.jar + artifact.jar.asc + public key → VALID or INVALID
```

The math ensures:
- Only someone with the private key can produce a valid signature
- Anyone with the public key can verify the signature
- The signature is tied to the exact content — change one byte and verification fails

---

## Step 1: Generate a GPG Key

```bash
gpg --full-generate-key
```

When prompted:

```
Please select what kind of key you want:
   (1) RSA and RSA
   (9) ECC (sign and encrypt)
   (10) ECC (sign only)
Your selection? 1

RSA keys may be between 1024 and 4096 bits long.
What keysize do you want? (3072) 4096

Please specify how long the key should be valid.
         0 = key does not expire
Key is valid for? (0) 2y

Real name: Linh Vu
Email address: your-email@example.com
Comment: (leave empty)

You need a Passphrase to protect your secret key.
Enter passphrase: (enter a strong passphrase — you'll need this later)
```

> **Insight: Key Expiration**
>
> Setting an expiration (e.g., `2y` for 2 years) is a security best practice. If your private key is ever compromised, the damage is time-limited. You can always extend the expiration before it lapses. Maven Central doesn't reject artifacts signed with expired keys — the signature is checked against the key's validity at signing time, not at download time.

After generation, verify your key:

```bash
gpg --list-keys --keyid-format short
```

Output:

```
pub   rsa4096/ABCD1234 2026-02-17 [SC] [expires: 2028-02-17]
      1234567890ABCDEF1234567890ABCDEF12345678
uid         [ultimate] Linh Vu <your-email@example.com>
sub   rsa4096/EFGH5678 2026-02-17 [E] [expires: 2028-02-17]
```

The short key ID is `ABCD1234` (the 8 characters after `rsa4096/`). The long fingerprint is the 40-character hex string.

---

## Step 2: Publish Your Public Key

Maven Central's validators need to verify your signatures. They download your public key from a keyserver:

```bash
gpg --keyserver keyserver.ubuntu.com --send-keys ABCD1234
```

Common keyservers (they sync with each other, but it can take minutes to hours):

| Keyserver | URL |
|---|---|
| Ubuntu | `keyserver.ubuntu.com` |
| MIT | `pgp.mit.edu` |
| OpenPGP | `keys.openpgp.org` |

Verify the upload:

```bash
gpg --keyserver keyserver.ubuntu.com --recv-keys ABCD1234
```

If it returns your key info, the upload succeeded.

---

## Step 3: Configure `maven-gpg-plugin`

```xml
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
```

This plugin:
1. Runs in the `verify` phase (after `package`, before `deploy`)
2. Signs **every** artifact attached to the module: main JAR, POM, sources JAR, javadoc JAR
3. Produces a `.asc` file for each

> **Insight: How Many Signatures?**
>
> For our 5-module project, the signing math is:
>
> | Module | Artifacts to Sign | Signatures |
> |---|---|---|
> | `zalobot-sdk-java` (parent) | POM only | 1 .asc |
> | `zalobot-core` | JAR + POM + sources + javadoc | 4 .asc |
> | `zalobot-client` | JAR + POM + sources + javadoc | 4 .asc |
> | `zalobot-listener` | JAR + POM + sources + javadoc | 4 .asc |
> | `zalobot-spring-boot` | JAR + POM + sources + javadoc | 4 .asc |
> | `zalobot-spring-boot-starter` | JAR + POM + sources (no javadoc) | 3 .asc |
> | **Total** | | **~20 .asc files** |
>
> The plugin handles all of this automatically. You configure it once in the parent POM, and it signs every artifact across all modules.

---

## Providing the Passphrase

The GPG plugin needs your passphrase to unlock the private key. There are several ways to provide it:

### Option A: `settings.xml` (recommended for local development)

In `~/.m2/settings.xml`:

```xml
<settings>
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
</settings>
```

### Option B: Command-line property

```bash
mvn deploy -Prelease -Dgpg.passphrase=your-passphrase
```

### Option C: Environment variable (recommended for CI/CD)

```bash
export MAVEN_GPG_PASSPHRASE=your-passphrase
mvn deploy -Prelease
```

In the plugin configuration, reference it:

```xml
<configuration>
    <passphrase>${env.MAVEN_GPG_PASSPHRASE}</passphrase>
</configuration>
```

We'll cover the CI/CD approach in detail in Chapter 11.

---

## GPG in the Build Lifecycle

Here's where signing fits in the overall build:

```
mvn deploy -Prelease

  compile     → src/main/java → target/classes
  test        → run unit tests
  package     → target/zalobot-client-0.0.1.jar
                target/zalobot-client-0.0.1-sources.jar    (from Ch 5)
                target/zalobot-client-0.0.1-javadoc.jar    (from Ch 5)
  verify      → maven-gpg-plugin signs ALL of the above:   ← THIS CHAPTER
                target/zalobot-client-0.0.1.jar.asc
                target/zalobot-client-0.0.1-sources.jar.asc
                target/zalobot-client-0.0.1-javadoc.jar.asc
                target/zalobot-client-0.0.1.pom.asc
  deploy      → uploads everything to Central Portal        (Ch 7)
```

---

## Exporting Your Key (for CI/CD Later)

In Chapter 11, you'll need to import your GPG key into a CI/CD environment. Export it now:

```bash
# Export private key (keep this SECRET)
gpg --export-secret-keys --armor ABCD1234 > private-key.asc

# View the content (you'll paste this into GitHub Secrets)
cat private-key.asc
# -----BEGIN PGP PRIVATE KEY BLOCK-----
# ... base64 content ...
# -----END PGP PRIVATE KEY BLOCK-----

# After pasting into GitHub Secrets, DELETE the file
rm private-key.asc
```

**Never commit `private-key.asc` to version control.**

---

## Summary

| Concept | What It Means |
|---|---|
| GPG key pair | Private key (signs) + public key (verifies) |
| `gpg --full-generate-key` | Creates a new key pair on your machine |
| Keyserver | Public server where anyone can download your public key |
| `maven-gpg-plugin` | Signs all Maven artifacts with your private key during `verify` phase |
| `.asc` files | ASCII-armored GPG signatures — one per artifact |
| Passphrase | Unlocks the private key; provided via settings.xml, CLI, or env var |
| ~20 signatures | Our 5-module project generates approximately 20 `.asc` files |

---

## What's Next?

We can sign artifacts. Now we need somewhere to upload them. In Chapter 7, we configure the Central Portal publishing mechanism — the pipeline that gets your artifacts from your machine to Maven Central.

```
Chapter 5  ──► "Source and Javadoc JARs"
Chapter 6  ←── YOU ARE HERE: "GPG signing"
Chapter 7  ──► "The Central Portal publishing mechanism"
```
