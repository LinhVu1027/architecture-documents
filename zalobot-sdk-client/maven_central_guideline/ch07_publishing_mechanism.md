# Chapter 7: The Publishing Mechanism — Central Portal and the Publishing Plugin

> In the old days, publishing to Maven Central required filing a JIRA ticket and waiting for manual approval. The new Central Portal is self-service. Here's how it works.

---

## Old Way vs. New Way

| | OSSRH (Legacy — `oss.sonatype.org`) | Central Portal (New — `central.sonatype.com`) |
|---|---|---|
| **Account creation** | Create JIRA ticket → wait 1-2 business days for human approval | Self-service registration → verify namespace immediately |
| **Plugin** | `nexus-staging-maven-plugin` (Sonatype-specific) | `central-publishing-maven-plugin` (official) |
| **Requires `<distributionManagement>`** | Yes — must declare staging repo URL | **No** — plugin handles everything |
| **Staging flow** | Upload to staging repo → close → release (3-step) | Upload bundle → validate → publish (automated or manual) |
| **UI** | Old Nexus Repository Manager | Modern web dashboard |
| **Status** | Deprecated for new projects (since Feb 2024) | **Required** for all new registrations |

> **Insight: No `<distributionManagement>` Needed**
>
> With the legacy OSSRH approach, you needed this in your POM:
> ```xml
> <distributionManagement>
>     <snapshotRepository>
>         <id>ossrh</id>
>         <url>https://s01.oss.sonatype.org/content/repositories/snapshots</url>
>     </snapshotRepository>
>     <repository>
>         <id>ossrh</id>
>         <url>https://s01.oss.sonatype.org/service/local/staging/deploy/maven2/</url>
>     </repository>
> </distributionManagement>
> ```
> The new `central-publishing-maven-plugin` replaces both `<distributionManagement>` and the `nexus-staging-maven-plugin`. It intercepts Maven's deploy phase directly and uploads to Central Portal. One plugin replaces two configuration blocks.

---

## Step 1: Create a Central Portal Account

1. Go to **https://central.sonatype.com**
2. Click **Sign In** → create an account (or sign in with GitHub/Google)
3. After signing in, you'll see the dashboard

---

## Step 2: Verify Your Namespace

Your `groupId` is `dev.linhvu`. Before publishing, you must prove you own this namespace.

### For domain-based groupId (`dev.linhvu`)

You own the domain `linhvu.dev`. Prove it by adding a DNS TXT record:

1. In the Central Portal, go to **Namespaces** → **Add Namespace**
2. Enter `dev.linhvu`
3. The portal gives you a verification code (e.g., `sonatype-12345abcde`)
4. Add a TXT record to your DNS:
   ```
   Host: @
   Type: TXT
   Value: sonatype-12345abcde
   ```
5. Click **Verify** in the Central Portal
6. Wait for DNS propagation (minutes to hours)

### For GitHub-based groupId (alternative)

If you used `io.github.<username>` as your groupId, verification is done through GitHub — the portal checks that the GitHub account exists and matches.

Since our groupId is `dev.linhvu` (domain-based), we use DNS verification.

---

## Step 3: Generate an API Token

The publishing plugin authenticates with the Central Portal using a token, not your password.

1. In the Central Portal, go to your **Account** → **Generate User Token**
2. You'll receive:
   - **Username**: a generated string (e.g., `aB1cD2eF`)
   - **Password**: a generated token (e.g., `xY9/zW8+vU7tS6...`)
3. Save these — you'll put them in `settings.xml` (Chapter 8)

---

## Step 4: Configure the Publishing Plugin

Add this plugin to the `release` profile in the parent `pom.xml`:

```xml
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
```

Let's break down each configuration element:

| Element | Value | Purpose |
|---|---|---|
| `<extensions>true</extensions>` | `true` | Allows the plugin to intercept the `deploy` phase — replaces default deploy behavior |
| `<publishingServerId>` | `central` | Links to `<server><id>central</id>` in `settings.xml` for credentials |
| `<autoPublish>` | `false` (start here) | `false` = upload + validate, then wait for manual "Publish" click in the UI. `true` = auto-publish if validation passes |

> **Insight: `<extensions>true</extensions>` — How the Plugin Takes Over Deploy**
>
> Normally, Maven's `deploy` phase uses the `maven-deploy-plugin` to upload artifacts to the URL specified in `<distributionManagement>`. When `central-publishing-maven-plugin` has `<extensions>true</extensions>`, it registers itself as a Maven extension that **replaces** the default deploy behavior entirely.
>
> This is why no `<distributionManagement>` is needed. The plugin doesn't look for a repository URL in your POM — it uploads directly to `central.sonatype.com` using the credentials from `settings.xml`.
>
> The `<extensions>true</extensions>` mechanism is a Maven feature called "build extensions" — plugins that modify Maven's core behavior rather than just running in a specific lifecycle phase.

---

## How the Plugin Works During `mvn deploy`

```
mvn deploy -Prelease
│
├── [Module 1: zalobot-core]
│     compile → test → package → source:jar → javadoc:jar → gpg:sign
│     deploy phase: plugin COLLECTS artifacts (does NOT upload yet)
│
├── [Module 2: zalobot-client]
│     compile → test → package → source:jar → javadoc:jar → gpg:sign
│     deploy phase: plugin COLLECTS artifacts
│
├── [Module 3: zalobot-listener]
│     ... same pattern ...
│
├── [Module 4: zalobot-spring-boot]
│     ... same pattern ...
│
├── [Module 5: zalobot-spring-boot-starter]
│     ... same pattern ...
│
└── AFTER all modules complete:
      central-publishing-maven-plugin:
      │
      ├── Bundles ALL collected artifacts into a single deployment
      │   (ZIP-like bundle with all JARs, POMs, sources, javadocs, signatures)
      │
      ├── Uploads bundle to https://central.sonatype.com
      │   using credentials from settings.xml server "central"
      │
      ├── Central Portal VALIDATES:
      │   ✓ POM metadata (name, license, developers, scm)
      │   ✓ Sources JAR present
      │   ✓ Javadoc JAR present
      │   ✓ GPG signatures valid
      │   ✓ No SNAPSHOT versions
      │   ✓ Namespace verified
      │
      └── If autoPublish=false:
            └── Deployment appears in Central Portal UI as "VALIDATED"
                └── You manually click "Publish"
                    └── Artifacts sync to Maven Central (~30 minutes)
```

---

## The Central Portal Dashboard

After a successful `mvn deploy -Prelease`, you'll see your deployment in the Central Portal UI:

```
central.sonatype.com → Deployments
┌──────────────────────────────────────────────────────────────┐
│ Deployment: dev.linhvu:zalobot-sdk-java:0.0.1                │
│ Status: VALIDATED ✓                                          │
│ Components: 6 (parent + 5 modules)                           │
│ Files: ~44                                                   │
│                                                              │
│ [Publish]  [Drop]                                            │
│                                                              │
│ Validation Results:                                          │
│ ✓ POM metadata complete                                      │
│ ✓ Sources provided                                           │
│ ✓ Javadoc provided                                           │
│ ✓ GPG signatures valid                                       │
│ ✓ Namespace dev.linhvu verified                              │
└──────────────────────────────────────────────────────────────┘
```

- **Publish**: Syncs artifacts to Maven Central. Irreversible — once published, a version can never be deleted or modified.
- **Drop**: Discards the deployment. Use this if you notice a mistake.

---

## When to Switch to `autoPublish=true`

Start with `autoPublish=false` to verify your first few deployments manually. Once you're confident:

```xml
<configuration>
    <publishingServerId>central</publishingServerId>
    <autoPublish>true</autoPublish>
</configuration>
```

With `autoPublish=true`, the plugin waits for validation to pass and then automatically publishes. No manual click needed. This is ideal for CI/CD (Chapter 11).

---

## Summary

| Concept | What It Means |
|---|---|
| Central Portal | `central.sonatype.com` — the new self-service publishing platform |
| Namespace verification | Proving you own `dev.linhvu` via DNS TXT record |
| API token | Generated credentials for the publishing plugin (not your password) |
| `central-publishing-maven-plugin` | Replaces `<distributionManagement>` + `nexus-staging-maven-plugin` |
| `<extensions>true</extensions>` | Plugin takes over Maven's deploy phase behavior |
| `<publishingServerId>central</publishingServerId>` | Links to credentials in `settings.xml` |
| `autoPublish=false` | Upload + validate, then manually click "Publish" |
| `autoPublish=true` | Upload + validate + auto-publish (for CI/CD) |

---

## What's Next?

The plugin needs credentials. In Chapter 8, we configure `settings.xml` to securely store the Central Portal token and GPG passphrase — keeping secrets out of version control.

```
Chapter 6  ──► "GPG signing"
Chapter 7  ←── YOU ARE HERE: "The Central Portal publishing mechanism"
Chapter 8  ──► "Keeping secrets out of your POM"
```
