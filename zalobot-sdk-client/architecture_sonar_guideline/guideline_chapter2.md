# Chapter 2: Choosing Your Platform

> **Type:** OUTSIDE-PROJECT — no code changes; this is a decision + infrastructure step

---

## Two Options

SonarQube comes in two flavors:

```
┌──────────────────────────────────────────────────────────────────┐
│                                                                   │
│  Option A: SonarCloud              Option B: Self-Hosted          │
│  ─────────────────                 ──────────────────             │
│                                                                   │
│  ☁  Hosted by Sonar               🖥  You run the server          │
│  URL: sonarcloud.io               URL: localhost:9000             │
│  Free for open source             Free Community Edition          │
│  Zero maintenance                 You maintain it                 │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

---

## Decision Table

| Criteria | SonarCloud | Self-Hosted |
|---|---|---|
| **Cost** | Free (open-source projects) | Free (Community Edition) |
| **Setup time** | 5 minutes (sign up + import repo) | 30-60 minutes (Docker + PostgreSQL) |
| **Maintenance** | Zero — Sonar handles updates | You update server + database |
| **PR decoration** | Built-in GitHub integration | Requires Developer Edition ($) |
| **Branch analysis** | Included | Requires Developer Edition ($) |
| **Privacy** | Code metadata sent to sonarcloud.io | Everything stays on your machine |
| **Availability** | Depends on sonarcloud.io uptime | Depends on your infrastructure |
| **Java 25 support** | Always up to date | Depends on your server version |

**Recommendation for zalobot-sdk-java:** **SonarCloud.** The project is open-source, so it's free. Zero infrastructure to manage. PR decoration works out of the box. The rest of this guide uses SonarCloud URLs, but all Maven configurations work identically with self-hosted.

---

## Option A: SonarCloud Setup

### Step 1: Create an account
1. Go to `https://sonarcloud.io`
2. Click "Log in" → "Sign in with GitHub"
3. Authorize SonarCloud to access your GitHub account

### Step 2: Create an organization
1. Click "+" → "Create new organization"
2. Choose "Import from GitHub"
3. Select your GitHub account or organization
4. Organization key will be something like `linhvu` (your GitHub username)

### Step 3: Import your project
1. Click "+" → "Analyze new project"
2. Select `zalobot-sdk-java` from your repositories
3. SonarCloud creates a project with key `dev.linhvu:zalobot-sdk-java` (or similar)
4. Note down the **project key** and **organization** — you'll need them in Chapter 5

After this step you'll have:
```
Project key:    linhvu_zalobot-sdk-java   (or your chosen key)
Organization:   linhvu                     (your GitHub username)
Server URL:     https://sonarcloud.io
```

---

## Option B: Self-Hosted Setup (Docker Compose)

If you prefer to keep everything local, here's a Docker Compose setup.

### Prerequisites
- Docker and Docker Compose installed
- At least 4GB RAM available for the SonarQube container

### docker-compose.yml

```yaml
# This file lives OUTSIDE your project (e.g., ~/sonarqube/)
# Do NOT commit this to zalobot-sdk-java

version: "3.8"

services:
  sonarqube:
    image: sonarqube:10-community
    container_name: sonarqube
    ports:
      - "9000:9000"
    environment:
      - SONAR_JDBC_URL=jdbc:postgresql://db:5432/sonarqube
      - SONAR_JDBC_USERNAME=sonar
      - SONAR_JDBC_PASSWORD=sonar
    volumes:
      - sonarqube_data:/opt/sonarqube/data
      - sonarqube_logs:/opt/sonarqube/logs
      - sonarqube_extensions:/opt/sonarqube/extensions
    depends_on:
      - db

  db:
    image: postgres:15
    container_name: sonarqube-db
    environment:
      - POSTGRES_USER=sonar
      - POSTGRES_PASSWORD=sonar
      - POSTGRES_DB=sonarqube
    volumes:
      - postgresql_data:/var/lib/postgresql/data

volumes:
  sonarqube_data:
  sonarqube_logs:
  sonarqube_extensions:
  postgresql_data:
```

### Start the server

```bash
cd ~/sonarqube/    # wherever you saved docker-compose.yml
docker compose up -d
```

### First login
1. Wait ~1-2 minutes for SonarQube to start
2. Open `http://localhost:9000`
3. Login with `admin` / `admin`
4. You'll be forced to change the password on first login

### Create a project
1. Click "Create Project" → "Manually"
2. Project key: `dev.linhvu:zalobot-sdk-java`
3. Display name: `zalobot-sdk-java`
4. Click "Set Up"

After this step you'll have:
```
Project key:    dev.linhvu:zalobot-sdk-java
Organization:   (not applicable for self-hosted)
Server URL:     http://localhost:9000
```

---

## What You Have After This Chapter

Regardless of which option you chose:

```
✓ A SonarQube server running (cloud or local)
✓ A project created on that server
✓ Project key noted down
✓ Organization noted down (SonarCloud only)
✓ Server URL noted down
```

These three values (project key, organization, server URL) will be used in Chapter 5 when we configure the Maven plugin.

---

## Next: Chapter 3

Before we touch any code, we need one more outside-project step: generating an authentication token so Maven can upload analysis reports to the server.
