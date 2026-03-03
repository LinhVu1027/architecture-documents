# Chapter 3: Creating Your SonarQube Project & Token

> **Type:** OUTSIDE-PROJECT — no code changes; all steps happen on the SonarQube UI and your local Maven settings

---

## Why You Need a Token

When you run `mvn sonar:sonar`, the Sonar Scanner plugin uploads an analysis report to the SonarQube server via HTTP. The server needs to know *who* is uploading — that's what the token does. It's like an API key.

```
Your machine                          SonarQube Server
─────────────                         ────────────────
mvn sonar:sonar
    │
    └──── HTTP POST ────────────────▶  "Who are you?"
          Authorization: Bearer          │
          sqa_xxxxxxxxxxxxx              ▼
                                      "Token valid.
                                       Processing report."
```

---

## Generating a Token

### SonarCloud

1. Go to `https://sonarcloud.io`
2. Click your avatar (top-right) → "My Account"
3. Go to the "Security" tab
4. Under "Generate Tokens":
   - Name: `zalobot-sdk-java-local` (or any descriptive name)
   - Type: **Project Analysis Token**
   - Project: select your `zalobot-sdk-java` project
   - Expires in: 90 days (or "No expiration" for simplicity)
5. Click "Generate"
6. **Copy the token immediately** — it won't be shown again

The token looks like: `sqa_a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6`

### Self-Hosted

1. Go to `http://localhost:9000`
2. Click your avatar (top-right) → "My Account"
3. Go to the "Security" tab
4. Under "Generate Tokens":
   - Name: `zalobot-sdk-java-local`
   - Type: **Project Analysis Token**
   - Project: select your project
5. Click "Generate"
6. **Copy the token immediately**

---

## Storing the Token Safely

You need the token available to Maven, but you must **never commit it to Git**. There are two approaches:

### Approach 1: Maven settings.xml (for local development)

Edit `~/.m2/settings.xml` (create it if it doesn't exist):

```xml
<settings>
  <profiles>
    <profile>
      <id>sonar</id>
      <properties>
        <sonar.token>sqa_YOUR_TOKEN_HERE</sonar.token>
      </properties>
    </profile>
  </profiles>
  <activeProfiles>
    <activeProfile>sonar</activeProfile>
  </activeProfiles>
</settings>
```

With this in place, you can run `mvn sonar:sonar` without passing `-Dsonar.token=xxx` every time.

**Security note:** `~/.m2/settings.xml` is on your local machine and is never committed to Git. This is a standard Maven practice for storing credentials.

### Approach 2: Command-line argument (simpler, explicit)

Pass the token directly when running Maven:

```bash
mvn clean verify sonar:sonar -Dsonar.token=sqa_YOUR_TOKEN_HERE
```

This is simpler but means you have to provide the token every time. Fine for occasional use.

### Approach 3: GitHub Secrets (for CI/CD — covered in Chapter 9)

For automated analysis in GitHub Actions, the token is stored as a repository secret:

```
GitHub → Repository Settings → Secrets → Actions → New repository secret
  Name:  SONAR_TOKEN
  Value: sqa_YOUR_TOKEN_HERE
```

---

## Token Types Explained

SonarQube offers three token types:

| Token Type | Scope | When to Use |
|---|---|---|
| **User Token** | All projects the user can access | Personal use across multiple projects |
| **Project Analysis Token** | Single project only | Recommended for CI/CD — least privilege |
| **Global Analysis Token** | All projects on the server | Admin-level, avoid unless necessary |

**Recommendation:** Use a **Project Analysis Token** for `zalobot-sdk-java`. It follows the principle of least privilege — if the token is ever leaked, it can only affect this one project.

---

## Verifying Your Setup

At this point, you should have these values ready:

```
┌─────────────────────────────────────────────────────────┐
│ From Chapter 2:                                          │
│   Server URL:     https://sonarcloud.io                  │
│                   (or http://localhost:9000)              │
│   Project key:    linhvu_zalobot-sdk-java                │
│   Organization:   linhvu (SonarCloud only)               │
│                                                          │
│ From Chapter 3:                                          │
│   Token:          sqa_xxxxxxxxxxxxxxxxxxxxxxxx           │
│   Stored in:      ~/.m2/settings.xml                     │
│                   (or you'll pass it via -D flag)         │
└─────────────────────────────────────────────────────────┘
```

You can verify the token works with a simple curl (optional):

```bash
# SonarCloud
curl -s -u sqa_YOUR_TOKEN: https://sonarcloud.io/api/authentication/validate
# Expected: {"valid":true}

# Self-hosted
curl -s -u sqa_YOUR_TOKEN: http://localhost:9000/api/authentication/validate
# Expected: {"valid":true}
```

---

## What's Next

The outside-project setup is done. Starting in Chapter 4, we make our first code changes: adding JaCoCo to the project to measure test coverage.
