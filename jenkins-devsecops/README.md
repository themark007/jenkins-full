# DevSecOps in Jenkins — Security at Every Stage

> Security is not a separate phase. It runs inline, in the same pipeline,
> on every branch and every PR. This document covers the complete
> shift-left security stack: static analysis, dependency scanning,
> container scanning, secrets detection, DAST, and quality gates —
> all wired into a Jenkins Multi-Branch Pipeline.

---

## Table of Contents

1. [The DevSecOps Mental Model](#1-the-devsecops-mental-model)
2. [Security Tool Stack](#2-security-tool-stack)
3. [Where Each Tool Runs in the Pipeline](#3-where-each-tool-runs-in-the-pipeline)
4. [SonarQube — Static Analysis & Quality Gates](#4-sonarqube--static-analysis--quality-gates)
5. [Trivy — Dependency & Container Scanning](#5-trivy--dependency--container-scanning)
6. [OWASP Dependency-Check — CVE Scanning](#6-owasp-dependency-check--cve-scanning)
7. [Gitleaks — Secrets Detection](#7-gitleaks--secrets-detection)
8. [OWASP ZAP — Dynamic Application Security Testing](#8-owasp-zap--dynamic-application-security-testing)
9. [The Full Secure Jenkinsfile](#9-the-full-secure-jenkinsfile)
10. [Jenkins Configuration & Plugin Setup](#10-jenkins-configuration--plugin-setup)
11. [Quality Gates — What Blocks a Merge](#11-quality-gates--what-blocks-a-merge)
12. [Security Findings — How to Read and Triage](#12-security-findings--how-to-read-and-triage)
13. [Quick Reference Cheatsheet](#13-quick-reference-cheatsheet)

---

## 1. The DevSecOps Mental Model

Traditional security runs at the end: code ships, then security scans it.
By that point, fixing a finding means touching code that is already in production.

DevSecOps flips this. Security runs where the code is — in the pipeline — so
findings surface in minutes, not weeks, and the developer who introduced the
issue is still context-loaded and can fix it immediately.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      SHIFT-LEFT SECURITY MODEL                              │
│                                                                             │
│  TRADITIONAL (shift-right):                                                 │
│                                                                             │
│  Code → Build → Test → Deploy → ███ SECURITY SCAN ███ → Fix → Redeploy    │
│                                   (weeks later, in prod)                   │
│                                                                             │
│  DEVSECOPS (shift-left):                                                    │
│                                                                             │
│  Code                                                                       │
│   │                                                                         │
│   ├─ Secrets scan          ← catches leaked keys before push               │
│   ├─ SAST / SonarQube      ← catches code bugs on every PR                 │
│   ├─ Dependency CVE scan   ← catches known vulnerabilities in packages     │
│   ├─ Container image scan  ← catches OS/package CVEs in the built image    │
│   ├─ Quality gate          ← blocks the merge if thresholds aren't met     │
│   └─ DAST (on main)        ← tests the running app for runtime issues      │
│                                                                             │
│  Fix time: minutes (developer still has context)                           │
│  Fix cost: low (no prod rollback, no incident, no audit finding)           │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### The four security layers

```
Layer 1 — CODE          What you write
  Tools: SonarQube (SAST), Gitleaks (secrets)
  When:  Every push, every PR

Layer 2 — DEPENDENCIES  What you import
  Tools: Trivy fs, OWASP Dependency-Check
  When:  Every PR (fast), nightly (full)

Layer 3 — CONTAINER     What you ship
  Tools: Trivy image
  When:  Every PR (block on HIGH/CRITICAL), every main build

Layer 4 — RUNTIME       What you run
  Tools: OWASP ZAP (DAST)
  When:  Main builds only (needs a live app)
```

---

## 2. Security Tool Stack

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                        TOOL STACK OVERVIEW                                   │
├────────────────────┬───────────────┬─────────────────────────────────────────┤
│  Tool              │  Category     │  What it finds                          │
├────────────────────┼───────────────┼─────────────────────────────────────────┤
│  SonarQube         │  SAST         │  Code bugs, code smells, security       │
│                    │  Code Quality │  hotspots, coverage gaps, duplications  │
├────────────────────┼───────────────┼─────────────────────────────────────────┤
│  Trivy (fs)        │  SCA          │  CVEs in dependency manifests           │
│                    │               │  (requirements.txt, package.json, etc.) │
├────────────────────┼───────────────┼─────────────────────────────────────────┤
│  Trivy (image)     │  Container    │  CVEs in OS packages and language       │
│                    │  Scanning     │  packages inside a built image          │
├────────────────────┼───────────────┼─────────────────────────────────────────┤
│  OWASP             │  SCA          │  CVEs in Java/JS/Python/Ruby deps       │
│  Dependency-Check  │               │  using NVD + OSS Index databases        │
├────────────────────┼───────────────┼─────────────────────────────────────────┤
│  Gitleaks          │  Secrets      │  Hardcoded API keys, tokens, passwords, │
│                    │  Detection    │  private keys in source code            │
├────────────────────┼───────────────┼─────────────────────────────────────────┤
│  OWASP ZAP         │  DAST         │  Runtime vulnerabilities: XSS, SQLi,    │
│                    │               │  CSRF, auth issues, misconfigurations   │
└────────────────────┴───────────────┴─────────────────────────────────────────┘

SAST  = Static Application Security Testing   (reads source code)
SCA   = Software Composition Analysis         (reads dependency metadata)
DAST  = Dynamic Application Security Testing  (hits the running app)
```

---

## 3. Where Each Tool Runs in the Pipeline

This is the most important decision. Running everything everywhere makes the pipeline slow and noisy. The table below shows the right placement.

```
┌───────────────────────┬──────────────┬──────────────┬──────────────────────┐
│  Tool                 │  Branch Job  │  PR Job      │  Main Job            │
├───────────────────────┼──────────────┼──────────────┼──────────────────────┤
│  Gitleaks             │  ✅ fast     │  ✅          │  ✅                  │
│  (secrets scan)       │  blocks push │  blocks merge│  blocks build        │
├───────────────────────┼──────────────┼──────────────┼──────────────────────┤
│  SonarQube            │  ❌          │  ✅          │  ✅                  │
│  (SAST + quality)     │  skip        │  quality gate│  full analysis       │
├───────────────────────┼──────────────┼──────────────┼──────────────────────┤
│  Trivy fs             │  ❌          │  ✅          │  ✅                  │
│  (dep CVE scan)       │  skip        │  HIGH/CRIT   │  block on HIGH/CRIT  │
├───────────────────────┼──────────────┼──────────────┼──────────────────────┤
│  OWASP Dep-Check      │  ❌          │  ❌          │  ✅ or nightly       │
│  (deep CVE scan)      │  too slow    │  too slow    │  full report         │
├───────────────────────┼──────────────┼──────────────┼──────────────────────┤
│  Trivy image          │  ❌          │  ✅          │  ✅                  │
│  (container scan)     │  no image    │  HIGH/CRIT   │  block on HIGH/CRIT  │
├───────────────────────┼──────────────┼──────────────┼──────────────────────┤
│  OWASP ZAP            │  ❌          │  ❌          │  ✅                  │
│  (DAST)               │  no live app │  optional    │  needs running app   │
└───────────────────────┴──────────────┴──────────────┴──────────────────────┘

Key:
  ✅  runs and can block the build
  ❌  intentionally skipped (too slow, no artifact yet, or not meaningful here)
```

### Why OWASP Dependency-Check is not on every PR

OWASP Dependency-Check downloads the full NVD database on first run (~300 MB) and takes 5–15 minutes. Running it on every PR branch build would make PRs painfully slow. The right pattern is:

- Trivy `fs` scan on every PR (fast, ~30 seconds, catches most HIGH/CRIT CVEs)
- OWASP Dependency-Check on every main build + as a nightly scheduled job (deep scan, HTML report, CVSS threshold gate)

---

## 4. SonarQube — Static Analysis & Quality Gates

SonarQube analyses source code for bugs, vulnerabilities, code smells, test coverage, and duplication. It enforces a **Quality Gate**: a pass/fail threshold that can block a merge.

### 4.1 How SonarQube integrates with Jenkins

```
Jenkins runs sonar-scanner
        │
        ▼
SonarQube server analyses the code
        │
        ▼
SonarQube posts analysis results back
        │
        ▼
Jenkins waits for Quality Gate result (waitForQualityGate())
        │
    ┌───┴───┐
  PASS ✅  FAIL ❌
    │         │
  continue  abort build
            (merge blocked)
```

### 4.2 SonarQube server setup (Docker, for local/lab)

```bash
docker run -d \
  --name sonarqube \
  --restart unless-stopped \
  -p 9000:9000 \
  -v sonarqube_data:/opt/sonarqube/data \
  -v sonarqube_logs:/opt/sonarqube/logs \
  -v sonarqube_extensions:/opt/sonarqube/extensions \
  sonarqube:lts-community

# Default login: admin / admin  (change immediately)
# Access at: http://localhost:9000
```

### 4.3 Create a SonarQube project and token

```
SonarQube UI → Projects → Create Project → Manually
  Project key:     <your-service-name>
  Display name:    <Your Service Name>
  → Set Up

Generate a token:
  My Account → Security → Generate Token
  Name: jenkins-ci
  Type: Project Analysis Token
  Project: <your-service-name>
  → Generate → copy the token (shown once)
```

### 4.4 Store the token in Jenkins

```
Manage Jenkins → Credentials → System → Global
  Kind:        Secret text
  Secret:      <sonarqube-token>
  ID:          sonarqube-token
  Description: SonarQube project analysis token
```

### 4.5 Configure SonarQube server in Jenkins

```
Manage Jenkins → System → SonarQube servers
  ✅ Enable injection of SonarQube server configuration as build environment variables

  Add SonarQube:
    Name:             SonarQube
    Server URL:       http://localhost:9000   (or your server URL)
    Server auth token: sonarqube-token        (credential ID from above)
```

### 4.6 sonar-project.properties

Place this at the root of your app repo. Adjust to your language and structure.

```properties
# ── Project identity ──────────────────────────────────────────────────────────
sonar.projectKey=<your-service-name>
sonar.projectName=<Your Service Name>
sonar.projectVersion=1.0

# ── Source & test paths ───────────────────────────────────────────────────────
# Python
sonar.sources=src
sonar.tests=tests
sonar.python.coverage.reportPaths=reports/coverage.xml
sonar.python.xunit.reportPath=reports/unit.xml

# Java / Maven (uncomment and remove Python lines above if using Java)
# sonar.sources=src/main/java
# sonar.tests=src/test/java
# sonar.java.binaries=target/classes
# sonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml

# Node.js (uncomment if using JS/TS)
# sonar.sources=src
# sonar.tests=tests
# sonar.javascript.lcov.reportPaths=coverage/lcov.info

# ── Exclusions ────────────────────────────────────────────────────────────────
sonar.exclusions=**/node_modules/**,**/vendor/**,**/*.min.js,**/migrations/**

# ── Quality Gate ──────────────────────────────────────────────────────────────
# The gate is configured in the SonarQube UI, not here.
# See Section 4.7 below.
```

### 4.7 Configuring the Quality Gate in SonarQube

```
SonarQube UI → Quality Gates → Create

Name: Production Gate

Conditions — On New Code:
  ┌──────────────────────────────────┬────────────────┬──────────────┐
  │  Metric                          │  Operator      │  Threshold   │
  ├──────────────────────────────────┼────────────────┼──────────────┤
  │  Coverage on new code            │  is less than  │  80%         │
  │  Duplicated lines on new code    │  is greater    │  3%          │
  │  Maintainability rating new code │  is worse than │  A           │
  │  Reliability rating new code     │  is worse than │  A           │
  │  Security rating new code        │  is worse than │  A           │
  │  Security hotspots reviewed      │  is less than  │  100%        │
  └──────────────────────────────────┴────────────────┴──────────────┘

Set as Default → assign to your project
```

### 4.8 Jenkinsfile snippet — SonarQube stage

```groovy
stage('SonarQube Analysis') {
    // Only on PRs and main — skip on fast branch builds
    when {
        anyOf {
            expression { env.CHANGE_ID != null }
            branch 'main'
        }
    }
    steps {
        script {
            // withSonarQubeEnv injects SONAR_HOST_URL and auth token
            withSonarQubeEnv('SonarQube') {

                // For Maven projects:
                // sh 'mvn sonar:sonar -Dsonar.pullrequest.key=${CHANGE_ID} ...'

                // For Python / Node / Go (using sonar-scanner CLI):
                sh """
                    sonar-scanner \\
                      -Dsonar.projectKey=<your-service-name> \\
                      -Dsonar.sources=. \\
                      -Dsonar.host.url=${SONAR_HOST_URL} \\
                      -Dsonar.login=${SONAR_AUTH_TOKEN} \\
                      ${env.CHANGE_ID ? "-Dsonar.pullrequest.key=${env.CHANGE_ID} -Dsonar.pullrequest.branch=${env.CHANGE_BRANCH} -Dsonar.pullrequest.base=${env.CHANGE_TARGET}" : ""}
                """
            }
        }
    }
}

stage('Quality Gate') {
    when {
        anyOf {
            expression { env.CHANGE_ID != null }
            branch 'main'
        }
    }
    steps {
        // Wait up to 5 minutes for SonarQube to finish analysis
        // and return the Quality Gate result.
        // abortPipeline: true → fails the build if the gate fails.
        timeout(time: 5, unit: 'MINUTES') {
            waitForQualityGate abortPipeline: true
        }
    }
}
```

---

## 5. Trivy — Dependency & Container Scanning

Trivy is a fast, comprehensive vulnerability scanner. It can scan filesystem dependency manifests (SCA) and built container images (container scanning) in one tool.

### 5.1 Install Trivy on the Jenkins agent

```bash
# On the agent (or baked into your agent Docker image)
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh \
  | sh -s -- -b /usr/local/bin

trivy --version
```

### 5.2 Trivy filesystem scan (dependency CVEs)

Scans `requirements.txt`, `package.json`, `go.sum`, `pom.xml`, and similar manifests for known CVEs — without building the image first. Fast (~30 seconds).

```groovy
stage('Trivy — Filesystem Scan') {
    when {
        anyOf {
            expression { env.CHANGE_ID != null }
            branch 'main'
        }
    }
    steps {
        sh """
            trivy fs \\
              --severity HIGH,CRITICAL \\
              --exit-code 1 \\
              --no-progress \\
              --format table \\
              --output reports/trivy-fs-report.txt \\
              .

            # Also produce a JSON report for archiving
            trivy fs \\
              --severity HIGH,CRITICAL \\
              --format json \\
              --output reports/trivy-fs-report.json \\
              .
        """
    }
    post {
        always {
            archiveArtifacts artifacts: 'reports/trivy-fs-*.txt, reports/trivy-fs-*.json',
                             allowEmptyArchive: true
        }
    }
}
```

### 5.3 Trivy image scan (container CVEs)

Scans the built Docker image for CVEs in OS packages and language runtime packages. Run this after `docker build` — before `docker push`.

```groovy
stage('Trivy — Image Scan') {
    when {
        anyOf {
            expression { env.CHANGE_ID != null }
            branch 'main'
        }
    }
    steps {
        script {
            // Table output to console
            sh """
                trivy image \\
                  --severity HIGH,CRITICAL \\
                  --exit-code 1 \\
                  --no-progress \\
                  --format table \\
                  "${IMAGE_FULL}:${IMAGE_TAG}"
            """

            // JSON report for archiving and later parsing
            sh """
                trivy image \\
                  --severity HIGH,CRITICAL \\
                  --exit-code 0 \\
                  --format json \\
                  --output reports/trivy-image-report.json \\
                  "${IMAGE_FULL}:${IMAGE_TAG}"
            """
        }
    }
    post {
        always {
            archiveArtifacts artifacts: 'reports/trivy-image-report.json',
                             allowEmptyArchive: true
        }
    }
}
```

### 5.4 Trivy severity levels explained

```
CRITICAL   CVSS 9.0–10.0   Remote code execution, auth bypass, data exposure
           → Always block. No exceptions.

HIGH       CVSS 7.0–8.9    Privilege escalation, significant data exposure
           → Block on PRs and main builds.
           → Existing findings in base image: accept with a ticket, set a deadline.

MEDIUM     CVSS 4.0–6.9    Limited impact, requires specific conditions
           → Report but do not block. Review weekly.

LOW        CVSS 0.1–3.9    Minimal impact
           → Report only. Review monthly.

UNKNOWN    No CVSS score yet
           → Treat as MEDIUM until scored.
```

### 5.5 Handling false positives and accepted risks

When a finding is a known false positive or an accepted risk with a tracked ticket, use a Trivy `.trivyignore` file at the repo root:

```
# .trivyignore
# Format: CVE-ID  (one per line, comments with #)

# False positive — this CVE affects a code path we do not use
CVE-2023-12345

# Accepted risk — tracked in JIRA-789, fix scheduled for v2.1
# Expires: 2025-12-01
CVE-2024-67890
```

---

## 6. OWASP Dependency-Check — CVE Scanning

OWASP Dependency-Check performs a deeper SCA scan than Trivy. It checks your dependencies against the NIST NVD database and can catch CVEs that Trivy's database hasn't yet indexed.

### 6.1 When to run it

Run OWASP Dependency-Check on main builds and as a nightly scheduled job. Do not run it on every PR — it is too slow (5–15 minutes on first run, 2–5 minutes with a cached NVD database).

### 6.2 Install the Jenkins plugin

```
Manage Jenkins → Plugins → Available
  → Search: OWASP Dependency-Check
  → Install and restart
```

### 6.3 Configure the NVD API key (strongly recommended)

Without an NVD API key, Dependency-Check is heavily rate-limited and downloads can take 30+ minutes. Get a free key at https://nvd.nist.gov/developers/request-an-api-key and store it:

```
Manage Jenkins → Credentials → System → Global → Add Credentials:
  Kind:        Secret text
  Secret:      <your-nvd-api-key>
  ID:          nvd-api-key
  Description: NVD API key for OWASP Dependency-Check
```

### 6.4 Jenkinsfile snippet — OWASP Dependency-Check stage

```groovy
stage('OWASP Dependency-Check') {
    when {
        anyOf {
            branch 'main'
            // Uncomment to also run on PRs (slower):
            // expression { env.CHANGE_ID != null }
        }
    }
    steps {
        withCredentials([string(credentialsId: 'nvd-api-key', variable: 'NVD_KEY')]) {
            dependencyCheck(
                additionalArguments: """
                    --scan .
                    --format HTML
                    --format JSON
                    --out reports/
                    --nvdApiKey ${NVD_KEY}
                    --failOnCVSS 7
                    --enableRetired
                    --enableExperimental
                    --disableAssembly
                """,
                odcInstallation: 'dependency-check'
            )
        }
    }
    post {
        always {
            dependencyCheckPublisher(
                pattern: 'reports/dependency-check-report.xml',
                failedTotalCritical: 0,
                failedTotalHigh: 0,
                unstableTotalMedium: 5
            )
            archiveArtifacts artifacts: 'reports/dependency-check-report.*',
                             allowEmptyArchive: true
        }
    }
}
```

### 6.5 Understanding the report

```
OWASP Dependency-Check report columns:

  Dependency     The package file where the vulnerable library was found
  CVE            CVE identifier (links to NVD entry)
  CVSS           Score 0.0 – 10.0 (higher = worse)
  Severity       Critical / High / Medium / Low
  CWE            Common Weakness Enumeration category
  Description    What the vulnerability allows an attacker to do
  Evidence       How Dependency-Check identified this library version

Key actions:
  CRITICAL / HIGH  → Update the package. If not possible, open a risk ticket.
  MEDIUM           → Schedule an update in the next sprint.
  LOW              → Log it, review monthly.
  FALSE POSITIVE   → Add a suppression:
                     <suppress>
                       <notes>Not affected — we don't use this code path</notes>
                       <cve>CVE-2023-12345</cve>
                     </suppress>
                     (in dependency-check-suppressions.xml at repo root)
```

---

## 7. Gitleaks — Secrets Detection

Gitleaks scans the repository history and working tree for hardcoded secrets: API keys, tokens, passwords, private keys, connection strings. A secret committed to Git — even briefly — must be considered compromised and rotated.

### 7.1 Why secrets scanning matters

```
Common patterns Gitleaks catches:
  AWS_SECRET_ACCESS_KEY = "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
  GITHUB_TOKEN          = "ghp_xxxxxxxxxxxxxxxxxxxx"
  password              = "MyS3cr3tP@ss"
  -----BEGIN RSA PRIVATE KEY-----
  postgresql://user:password@host/db
  Authorization: Bearer eyJhbGci...
```

Scanning on every push means:
- A secret committed locally → Gitleaks catches it before it reaches GitHub
- A secret accidentally merged → it is flagged immediately on the PR/main job

### 7.2 Install Gitleaks on the Jenkins agent

```bash
# Download the latest release binary
GITLEAKS_VERSION=8.18.2
curl -sSfL \
  https://github.com/gitleaks/gitleaks/releases/download/v${GITLEAKS_VERSION}/gitleaks_${GITLEAKS_VERSION}_linux_x64.tar.gz \
  | tar -xz -C /usr/local/bin gitleaks

gitleaks version
```

### 7.3 Jenkinsfile snippet — Gitleaks stage

```groovy
stage('Gitleaks — Secrets Scan') {
    // Run on everything: branches, PRs, and main.
    // This is fast (~5 seconds) and should never be skipped.
    steps {
        sh """
            mkdir -p reports

            gitleaks detect \\
              --source . \\
              --report-format json \\
              --report-path reports/gitleaks-report.json \\
              --exit-code 1 \\
              --redact \\
              --no-banner
        """
    }
    post {
        always {
            archiveArtifacts artifacts: 'reports/gitleaks-report.json',
                             allowEmptyArchive: true
        }
        failure {
            echo """
╔══════════════════════════════════════════════════════════╗
║  🚨 SECRETS DETECTED IN SOURCE CODE 🚨                  ║
║                                                          ║
║  Gitleaks found potential secrets in this commit.        ║
║  ACTION REQUIRED:                                        ║
║  1. Remove the secret from the code immediately          ║
║  2. Rotate/revoke the exposed credential NOW             ║
║  3. Check git history — if the secret was pushed to      ║
║     GitHub, assume it is compromised even if the         ║
║     commit is deleted                                    ║
║  4. Use Jenkins Credentials store instead                ║
╚══════════════════════════════════════════════════════════╝
"""
        }
    }
}
```

### 7.4 Gitleaks configuration file

Place `.gitleaks.toml` at the repo root to tune rules for your codebase:

```toml
# .gitleaks.toml

title = "Gitleaks config"

[extend]
# Start from the official default ruleset
useDefault = true

[[allowlists]]
description = "Global allowlist"
regexTarget = "match"

# Allow known test fixtures that look like secrets but aren't
paths = [
    "tests/fixtures/.*",
    "docs/examples/.*"
]

[[allowlists]]
description = "Allow placeholder values in config templates"
regexes = [
    '''REPLACE_ME''',
    '''<YOUR_.*_HERE>''',
    '''example_secret_do_not_use'''
]
```

---

## 8. OWASP ZAP — Dynamic Application Security Testing

OWASP ZAP (Zed Attack Proxy) tests the **running application** by sending crafted HTTP requests and looking for common web vulnerabilities: XSS, SQL injection, CSRF, insecure headers, exposed error messages, and more.

### 8.1 When to run ZAP

DAST needs a live app to test. Run it on main builds after the app is deployed to dev/staging. Do not run it on every PR branch — it is slow (3–15 minutes) and requires a deployed environment.

```
Branch job:  ❌  no live app yet
PR job:      ⚠️   optional, on a preview environment (keep to baseline scan only)
Main job:    ✅  full active scan against the dev/stage environment
```

### 8.2 ZAP scan types

```
Baseline Scan (--spider + passive analysis):
  Duration:   ~1–3 minutes
  What:       Crawls the app, passively checks responses for obvious issues
              (insecure headers, exposed info, weak config)
  No active:  Does not send attack payloads
  Use for:    PR previews — safe to run against shared environments

Full Scan (--spider + active attack):
  Duration:   3–15 minutes depending on app surface
  What:       Actively probes for XSS, SQLi, path traversal, etc.
  WARNING:    Sends attack payloads. Only run against dedicated test environments.
              NEVER run against production.
  Use for:    Main/stage builds against isolated environments
```

### 8.3 Jenkinsfile snippet — OWASP ZAP stage

```groovy
stage('OWASP ZAP — DAST') {
    when {
        branch 'main'    // Main only — needs a deployed environment
    }
    steps {
        script {
            def TARGET_URL = "http://localhost:8080"   // URL of deployed dev/stage app
            def ZAP_REPORT_DIR = "${WORKSPACE}/reports/zap"

            sh "mkdir -p ${ZAP_REPORT_DIR}"

            // Run ZAP as a Docker container — no installation needed on agent
            // Replace 'full' with 'baseline' for a faster, passive-only scan
            sh """
                docker run --rm \\
                  --network host \\
                  -v "${ZAP_REPORT_DIR}:/zap/wrk:rw" \\
                  ghcr.io/zaproxy/zaproxy:stable \\
                  zap-full-scan.py \\
                    -t "${TARGET_URL}" \\
                    -r zap-report.html \\
                    -J zap-report.json \\
                    -x zap-report.xml \\
                    -l WARN \\
                    -I \\
                    --hook=/zap/auth_hook.py 2>/dev/null || true
            """

            // Parse JSON report to fail build on HIGH/CRITICAL alerts
            def zapResult = sh(
                script: """
                    python3 -c "
import json, sys
with open('${ZAP_REPORT_DIR}/zap-report.json') as f:
    report = json.load(f)
high_or_crit = [
    a for site in report.get('site', [])
    for a in site.get('alerts', [])
    if a.get('riskcode') in ('3', '4')   # 3=High, 4=Critical
]
if high_or_crit:
    print('HIGH/CRITICAL alerts found:')
    for a in high_or_crit:
        print(f\"  [{a['riskdesc']}] {a['alert']} — {a['url']}\")
    sys.exit(1)
else:
    print('No HIGH or CRITICAL alerts found ✅')
    sys.exit(0)
"
                """,
                returnStatus: true
            )

            if (zapResult != 0) {
                error "ZAP found HIGH or CRITICAL vulnerabilities. See ZAP report in artifacts."
            }
        }
    }
    post {
        always {
            publishHTML(target: [
                allowMissing:          true,
                alwaysLinkToLastBuild: true,
                keepAll:               true,
                reportDir:             'reports/zap',
                reportFiles:           'zap-report.html',
                reportName:            'ZAP Security Report'
            ])
            archiveArtifacts artifacts: 'reports/zap/zap-report.*',
                             allowEmptyArchive: true
        }
    }
}
```

### 8.4 Common ZAP findings and what they mean

```
Risk Level   Alert                           What to check
──────────   ──────────────────────────────  ──────────────────────────────────
High         SQL Injection                   Parameterize all DB queries
High         Cross-Site Scripting (XSS)      Encode output, use CSP headers
High         Path Traversal                  Validate and sanitize file paths
Medium       Missing Anti-CSRF Tokens        Implement CSRF protection on forms
Medium       X-Content-Type-Options Missing  Add header: nosniff
Medium       CSP Header Not Set              Configure Content-Security-Policy
Low          Cookie Without HttpOnly Flag    Set HttpOnly on session cookies
Low          Server Banner Disclosure        Remove version from Server header
Informational Application Error Disclosure  Don't expose stack traces in prod
```

---

## 9. The Full Secure Jenkinsfile

A complete `Jenkinsfile` with all security stages wired in, in the correct order, with the correct `when` conditions.

```groovy
// ─────────────────────────────────────────────────────────────────────────────
// Secure Jenkinsfile — DevSecOps Multi-Branch Pipeline
//
// Security stages:
//   1. Gitleaks       (all pipelines — fast, critical, always first)
//   2. SonarQube      (PR + main — SAST + quality gate)
//   3. Trivy fs       (PR + main — dependency CVE scan)
//   4. Build image
//   5. Trivy image    (PR + main — container CVE scan)
//   6. Push image
//   7. Deploy preview (PR only)
//   8. Integration tests (PR only)
//   9. OWASP Dep-Check (main only)
//  10. OWASP ZAP      (main only — DAST against live app)
//  11. Update manifest (main only)
// ─────────────────────────────────────────────────────────────────────────────

def IMAGE_REGISTRY = "docker.io"
def IMAGE_REPO     = "<your-org>/<service-name>"
def SERVICE_NAME   = "<service-name>"
def IMAGE_FULL     = "${IMAGE_REGISTRY}/${IMAGE_REPO}"

def SHORT_SHA      = ""
def IMAGE_TAG      = ""
def IMAGE_DIGEST   = ""

pipeline {

    agent { label 'docker-agent' }

    options {
        buildDiscarder(logRotator(numToKeepStr: '20', artifactNumToKeepStr: '10'))
        disableConcurrentBuilds()
        timestamps()
        timeout(time: 45, unit: 'MINUTES')
    }

    environment {
        REGISTRY_CREDS = credentials('registry-credentials')
        TRIVY_SEVERITY = "HIGH,CRITICAL"
    }

    stages {

        // ── 0. Setup ──────────────────────────────────────────────────────────
        stage('Setup') {
            steps {
                script {
                    SHORT_SHA = sh(script: "git rev-parse --short=7 HEAD", returnStdout: true).trim()

                    if (env.CHANGE_ID) {
                        IMAGE_TAG = "pr-${env.CHANGE_ID}-${SHORT_SHA}"
                    } else if (env.BRANCH_NAME == 'main') {
                        def version = sh(
                            script: "git describe --tags --abbrev=0 2>/dev/null || echo v0.${BUILD_NUMBER}",
                            returnStdout: true
                        ).trim()
                        IMAGE_TAG = "${SHORT_SHA}-${version}"
                    } else {
                        def branchSlug = env.BRANCH_NAME.replaceAll('[^a-zA-Z0-9]', '-').toLowerCase()
                        IMAGE_TAG = "${branchSlug}-${BUILD_NUMBER}"
                    }

                    sh "mkdir -p reports"

                    echo "=== Build Context ==="
                    echo "Branch:    ${env.BRANCH_NAME}"
                    echo "PR:        ${env.CHANGE_ID ?: 'N/A'}"
                    echo "Commit:    ${SHORT_SHA}"
                    echo "Image tag: ${IMAGE_TAG}"
                }
            }
        }

        // ── 1. Gitleaks — Secrets Detection ─────────────────────────────────
        // Runs on EVERYTHING. Fast. Catches leaked keys before they leave the repo.
        stage('Secrets Scan') {
            steps {
                sh """
                    gitleaks detect \\
                      --source . \\
                      --report-format json \\
                      --report-path reports/gitleaks-report.json \\
                      --exit-code 1 \\
                      --redact \\
                      --no-banner
                """
            }
            post {
                always {
                    archiveArtifacts artifacts: 'reports/gitleaks-report.json',
                                     allowEmptyArchive: true
                }
                failure {
                    echo "🚨 SECRETS DETECTED — rotate the exposed credential immediately and remove it from git history"
                }
            }
        }

        // ── 2. Lint ───────────────────────────────────────────────────────────
        stage('Lint') {
            steps {
                sh '''
                    # Replace with your linter:
                    # Python:  flake8 src/ && black --check src/
                    # Node:    npm run lint
                    # Java:    mvn checkstyle:check
                    echo "Lint passed ✅"
                '''
            }
        }

        // ── 3. Unit Tests ─────────────────────────────────────────────────────
        stage('Unit Tests') {
            steps {
                sh '''
                    # Replace with your test runner:
                    # Python:  pytest tests/unit/ --junitxml=reports/unit.xml --cov=src --cov-report=xml:reports/coverage.xml
                    # Node:    npm test
                    # Java:    mvn test
                    echo "Tests passed ✅"
                '''
            }
            post {
                always {
                    // junit 'reports/unit.xml'
                    echo "Test report published"
                }
            }
        }

        // ── 4. SonarQube — SAST + Quality Gate ───────────────────────────────
        stage('SonarQube Analysis') {
            when {
                anyOf {
                    expression { env.CHANGE_ID != null }
                    branch 'main'
                }
            }
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh """
                        sonar-scanner \\
                          -Dsonar.projectKey=${SERVICE_NAME} \\
                          -Dsonar.sources=. \\
                          -Dsonar.host.url=${SONAR_HOST_URL} \\
                          -Dsonar.login=${SONAR_AUTH_TOKEN} \\
                          ${env.CHANGE_ID ?
                            "-Dsonar.pullrequest.key=${env.CHANGE_ID} -Dsonar.pullrequest.branch=${env.CHANGE_BRANCH} -Dsonar.pullrequest.base=${env.CHANGE_TARGET}" :
                            ""}
                    """
                }
            }
        }

        stage('Quality Gate') {
            when {
                anyOf {
                    expression { env.CHANGE_ID != null }
                    branch 'main'
                }
            }
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        // ── 5. Trivy — Filesystem (Dependency) Scan ──────────────────────────
        stage('Trivy — Dependency Scan') {
            when {
                anyOf {
                    expression { env.CHANGE_ID != null }
                    branch 'main'
                }
            }
            steps {
                sh """
                    trivy fs \\
                      --severity ${TRIVY_SEVERITY} \\
                      --exit-code 1 \\
                      --no-progress \\
                      --format table .

                    trivy fs \\
                      --severity ${TRIVY_SEVERITY} \\
                      --format json \\
                      --output reports/trivy-fs-report.json .
                """
            }
            post {
                always {
                    archiveArtifacts artifacts: 'reports/trivy-fs-report.json',
                                     allowEmptyArchive: true
                }
            }
        }

        // ── 6. Build Image ────────────────────────────────────────────────────
        stage('Build Image') {
            steps {
                sh """
                    docker build \\
                      --label "git.commit=${SHORT_SHA}" \\
                      --label "git.branch=${env.BRANCH_NAME}" \\
                      --label "jenkins.build=${BUILD_NUMBER}" \\
                      -t "${IMAGE_FULL}:${IMAGE_TAG}" \\
                      .
                """
            }
        }

        // ── 7. Trivy — Image Scan ─────────────────────────────────────────────
        stage('Trivy — Image Scan') {
            when {
                anyOf {
                    expression { env.CHANGE_ID != null }
                    branch 'main'
                }
            }
            steps {
                sh """
                    trivy image \\
                      --severity ${TRIVY_SEVERITY} \\
                      --exit-code 1 \\
                      --no-progress \\
                      --format table \\
                      "${IMAGE_FULL}:${IMAGE_TAG}"

                    trivy image \\
                      --severity ${TRIVY_SEVERITY} \\
                      --format json \\
                      --output reports/trivy-image-report.json \\
                      "${IMAGE_FULL}:${IMAGE_TAG}"
                """
            }
            post {
                always {
                    archiveArtifacts artifacts: 'reports/trivy-image-report.json',
                                     allowEmptyArchive: true
                }
            }
        }

        // ── 8. Push Image ─────────────────────────────────────────────────────
        stage('Push Image') {
            steps {
                script {
                    sh "echo ${REGISTRY_CREDS_PSW} | docker login ${IMAGE_REGISTRY} -u ${REGISTRY_CREDS_USR} --password-stdin"
                    sh "docker push ${IMAGE_FULL}:${IMAGE_TAG}"

                    if (env.BRANCH_NAME == 'main') {
                        sh "docker tag ${IMAGE_FULL}:${IMAGE_TAG} ${IMAGE_FULL}:latest"
                        sh "docker push ${IMAGE_FULL}:latest"

                        IMAGE_DIGEST = sh(
                            script: "docker inspect --format='{{index .RepoDigests 0}}' ${IMAGE_FULL}:${IMAGE_TAG} | cut -d@ -f2",
                            returnStdout: true
                        ).trim()
                        echo "Digest: ${IMAGE_DIGEST}"
                    }

                    sh "docker logout ${IMAGE_REGISTRY}"
                }
            }
        }

        // ── 9. Deploy Preview (PR only) ───────────────────────────────────────
        stage('Deploy Preview') {
            when {
                expression { env.CHANGE_ID != null }
            }
            steps {
                script {
                    def preview = "${SERVICE_NAME}-pr-${env.CHANGE_ID}"
                    sh """
                        docker rm -f ${preview} 2>/dev/null || true
                        docker run -d --name ${preview} -p 0:8080 "${IMAGE_FULL}:${IMAGE_TAG}"
                        sleep 5
                        docker inspect ${preview} \\
                          --format='{{(index (index .NetworkSettings.Ports "8080/tcp") 0).HostPort}}' \\
                          > preview-port.txt
                    """
                }
            }
        }

        // ── 10. Integration Tests (PR only) ───────────────────────────────────
        stage('Integration Tests') {
            when {
                expression { env.CHANGE_ID != null }
            }
            steps {
                script {
                    def port = readFile('preview-port.txt').trim()
                    sh """
                        HTTP=\$(curl -s -o /dev/null -w "%{http_code}" --retry 5 --retry-delay 2 http://localhost:${port}/health)
                        [ "\$HTTP" = "200" ] || { echo "Health check failed: HTTP \$HTTP"; exit 1; }
                        echo "Integration smoke test PASSED ✅"
                    """
                }
            }
        }

        // ── 11. OWASP Dependency-Check (main only) ────────────────────────────
        stage('OWASP Dependency-Check') {
            when {
                branch 'main'
            }
            steps {
                withCredentials([string(credentialsId: 'nvd-api-key', variable: 'NVD_KEY')]) {
                    dependencyCheck(
                        additionalArguments: """
                            --scan .
                            --format HTML --format JSON
                            --out reports/
                            --nvdApiKey ${NVD_KEY}
                            --failOnCVSS 7
                        """,
                        odcInstallation: 'dependency-check'
                    )
                }
            }
            post {
                always {
                    dependencyCheckPublisher(
                        pattern:              'reports/dependency-check-report.xml',
                        failedTotalCritical:  0,
                        failedTotalHigh:      0,
                        unstableTotalMedium:  5
                    )
                    archiveArtifacts artifacts: 'reports/dependency-check-report.*',
                                     allowEmptyArchive: true
                }
            }
        }

        // ── 12. OWASP ZAP — DAST (main only) ─────────────────────────────────
        stage('OWASP ZAP — DAST') {
            when {
                branch 'main'
            }
            steps {
                script {
                    def TARGET = "http://localhost:8080"
                    def ZAP_DIR = "${WORKSPACE}/reports/zap"
                    sh "mkdir -p ${ZAP_DIR}"

                    sh """
                        docker run --rm --network host \\
                          -v "${ZAP_DIR}:/zap/wrk:rw" \\
                          ghcr.io/zaproxy/zaproxy:stable \\
                          zap-full-scan.py \\
                            -t "${TARGET}" \\
                            -r zap-report.html \\
                            -J zap-report.json \\
                            -l WARN -I || true
                    """

                    def rc = sh(
                        script: """
                            python3 -c "
import json, sys
with open('${ZAP_DIR}/zap-report.json') as f:
    r = json.load(f)
high = [a for s in r.get('site',[]) for a in s.get('alerts',[]) if a.get('riskcode') in ('3','4')]
if high:
    [print(f'  [{a[\\\"riskdesc\\\"]}] {a[\\\"alert\\\"]}') for a in high]
    sys.exit(1)
"
                        """,
                        returnStatus: true
                    )
                    if (rc != 0) error("ZAP found HIGH/CRITICAL vulnerabilities — see ZAP report artifact")
                }
            }
            post {
                always {
                    publishHTML(target: [
                        reportDir:             'reports/zap',
                        reportFiles:           'zap-report.html',
                        reportName:            'ZAP Security Report',
                        allowMissing:          true,
                        alwaysLinkToLastBuild: true,
                        keepAll:               true
                    ])
                    archiveArtifacts artifacts: 'reports/zap/zap-report.*',
                                     allowEmptyArchive: true
                }
            }
        }

        // ── 13. Update Manifest (main only) ───────────────────────────────────
        stage('Update Manifest') {
            when {
                branch 'main'
            }
            steps {
                withCredentials([string(credentialsId: 'manifest-repo-pat', variable: 'TOKEN')]) {
                    sh """
                        git clone --depth 1 \\
                          https://x-access-token:${TOKEN}@github.com/<org>/<manifest-repo>.git \\
                          /tmp/manifest-repo

                        cd /tmp/manifest-repo/apps/${SERVICE_NAME}/overlays/dev

                        kustomize edit set image \\
                          ${IMAGE_FULL}=${IMAGE_FULL}@${IMAGE_DIGEST}

                        git config user.email "jenkins@ci.example.com"
                        git config user.name  "Jenkins CI"
                        git add kustomization.yaml
                        git commit -m "ci(${SERVICE_NAME}): update dev image to ${IMAGE_TAG} [${SHORT_SHA}]"
                        git push origin main

                        cd /tmp && rm -rf manifest-repo
                    """
                }
            }
        }

    }
    // ── End of stages ──────────────────────────────────────────────────────────

    post {
        always {
            script {
                if (env.CHANGE_ID) {
                    sh "docker rm -f ${SERVICE_NAME}-pr-${env.CHANGE_ID} 2>/dev/null || true"
                }
                sh "docker rmi ${IMAGE_FULL}:${IMAGE_TAG} 2>/dev/null || true"
            }
            cleanWs()
        }
        success { echo "✅ Build ${BUILD_NUMBER} passed — ${env.BRANCH_NAME}" }
        failure { echo "❌ Build ${BUILD_NUMBER} FAILED — ${env.BRANCH_NAME}" }
    }
}
```

---

## 10. Jenkins Configuration & Plugin Setup

### Required plugins

Install all of these before running the pipeline:

```
Manage Jenkins → Plugins → Available plugins

Security tools:
  SonarQube Scanner             sonar
  OWASP Dependency-Check        dependency-check-jenkins-plugin
  HTML Publisher                htmlpublisher     (ZAP report UI)

Pipeline essentials:
  Pipeline                      workflow-aggregator
  GitHub Branch Source          github-branch-source
  Credentials Binding           credentials-binding
  Workspace Cleanup             ws-cleanup
  Timestamper                   timestamper
  Blue Ocean (optional)         blueocean
```

### Required tools — install on agent or bake into agent image

```bash
# Trivy
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh \
  | sh -s -- -b /usr/local/bin

# Gitleaks
GITLEAKS_VERSION=8.18.2
curl -sSfL \
  https://github.com/gitleaks/gitleaks/releases/download/v${GITLEAKS_VERSION}/gitleaks_${GITLEAKS_VERSION}_linux_x64.tar.gz \
  | tar -xz -C /usr/local/bin gitleaks

# SonarScanner CLI
SONAR_VERSION=5.0.1.3006
curl -sSfL \
  https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-cli-${SONAR_VERSION}-linux.zip \
  -o /tmp/sonar-scanner.zip
unzip /tmp/sonar-scanner.zip -d /opt/
ln -s /opt/sonar-scanner-${SONAR_VERSION}-linux/bin/sonar-scanner /usr/local/bin/sonar-scanner

# Kustomize (for manifest updates)
curl -sSfL https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh \
  | bash -s -- /usr/local/bin

# Docker CLI (if using DooD)
curl -fsSL https://get.docker.com | sh
```

### Required Jenkins credentials

```
ID                       Kind                   Used by
───────────────────────  ─────────────────────  ────────────────────────────────
registry-credentials     Username + password     docker login (push image)
sonarqube-token          Secret text             sonar-scanner auth
nvd-api-key              Secret text             OWASP Dependency-Check NVD API
manifest-repo-pat        Secret text             git push to manifest repo
github-app               GitHub App / PAT        checkout private app repo
webhook-secret           Secret text             validate GitHub webhooks
```

### SonarQube server configuration in Jenkins

```
Manage Jenkins → System → SonarQube servers
  ✅ Enable injection of SonarQube server configuration...

  Name:              SonarQube
  Server URL:        http://<sonarqube-host>:9000
  Server auth token: sonarqube-token   ← credential ID
```

### OWASP Dependency-Check tool configuration

```
Manage Jenkins → Tools → Dependency-Check installations
  Name:                   dependency-check
  Install automatically:  ✅
  Version:                (select latest)
```

---

## 11. Quality Gates — What Blocks a Merge

### Branch protection required checks

Configure these in GitHub → Repository → Settings → Branches → main:

```
Required status checks (names must match exactly what Jenkins posts):
  ci/gitleaks           → blocks if secrets found
  ci/lint               → blocks if lint fails
  ci/unit-tests         → blocks if tests fail
  ci/sonarqube          → blocks if quality gate fails
  ci/trivy-fs           → blocks if HIGH/CRITICAL dependency CVE found
  ci/trivy-image        → blocks if HIGH/CRITICAL container CVE found
  ci/integration-tests  → blocks if smoke test fails
```

### Gate summary by tool

```
┌──────────────────────┬─────────────────────────────────────────────────────┐
│  Tool                │  Gate condition                                     │
├──────────────────────┼─────────────────────────────────────────────────────┤
│  Gitleaks            │  Any secret found → FAIL. No exceptions.            │
├──────────────────────┼─────────────────────────────────────────────────────┤
│  SonarQube           │  Quality Gate result = FAILED → FAIL                │
│                      │  Quality Gate: coverage < 80%, rating worse than A  │
├──────────────────────┼─────────────────────────────────────────────────────┤
│  Trivy fs            │  Any HIGH or CRITICAL CVE → FAIL                    │
│  (dependency)        │  MEDIUM and below → report only                     │
├──────────────────────┼─────────────────────────────────────────────────────┤
│  Trivy image         │  Any HIGH or CRITICAL CVE → FAIL                    │
│  (container)         │  MEDIUM and below → report only                     │
├──────────────────────┼─────────────────────────────────────────────────────┤
│  OWASP Dep-Check     │  CVSS ≥ 7 (HIGH) → FAIL (main builds only)         │
│                      │  CVSS 4–6 (MEDIUM) ≥ 5 findings → UNSTABLE         │
├──────────────────────┼─────────────────────────────────────────────────────┤
│  OWASP ZAP           │  Any HIGH or CRITICAL alert → FAIL (main only)      │
│                      │  MEDIUM and below → reported in HTML artifact       │
└──────────────────────┴─────────────────────────────────────────────────────┘
```

---

## 12. Security Findings — How to Read and Triage

### Triage decision tree

```
Finding arrives in a scan report
          │
          ▼
  Is it a FALSE POSITIVE?
  (the code path is not reachable, or the scan
   misidentified the library version)
          │
    ┌─────┴─────┐
   YES           NO
    │             │
  Document it   What is the severity?
  Add to         │
  .trivyignore   ├── CRITICAL / HIGH
  or suppression │      │
  file           │   Can it be fixed now?
  Open a ticket  │      ├── YES → Fix it. This sprint.
  with evidence  │      └── NO  → Open a risk ticket with CVSS,
                 │               impact, mitigation, and a deadline.
                 │               Get security team sign-off.
                 │               Accepted risks do NOT suppress the
                 │               scan indefinitely — set an expiry date.
                 │
                 ├── MEDIUM
                 │      Schedule in next sprint backlog.
                 │      Do not block the PR for MEDIUM findings
                 │      unless they are in actively exploited paths.
                 │
                 └── LOW / INFO
                        Log it. Review monthly.
                        Batch fixes into a dependency update PR.
```

### Common CVE fix patterns

```
Pattern: outdated base image OS packages
  Finding:  CVE in libssl / glibc / curl inside the container
  Fix:      Rebuild from a newer base image tag
            FROM python:3.11-slim  →  FROM python:3.12-slim
            Or: add  RUN apt-get upgrade -y  in Dockerfile (short term)

Pattern: outdated direct dependency
  Finding:  CVE in requests==2.28.0 (requirements.txt)
  Fix:      Update to the patched version: requests>=2.31.0
            Run tests. Merge.

Pattern: transitive dependency CVE
  Finding:  CVE in a library you didn't install directly
  Fix:      Pin the patched version as a direct dependency
            Or use pip's --constraint flag / npm overrides

Pattern: no fix available yet
  Finding:  CVE filed but maintainer hasn't released a patch
  Action:   Document the risk. Consider alternative libraries.
            Monitor the CVE for patch release. Set a 30-day review.
```

---

## 13. Quick Reference Cheatsheet

### Tool placement summary

```
ALWAYS (branch + PR + main):   Gitleaks (fast, ~5s, catches secrets)
PR + main only:                SonarQube, Trivy fs, Trivy image
Main only:                     OWASP Dependency-Check, OWASP ZAP
```

### Severity thresholds

```
CRITICAL   Block everywhere. Rotate if it's a secret. Fix immediately.
HIGH       Block on PR and main. Fix within the sprint.
MEDIUM     Report. Do not block. Fix next sprint.
LOW        Report. Batch monthly.
```

### Useful Trivy commands

```bash
# Scan filesystem for CVEs in dependency manifests
trivy fs --severity HIGH,CRITICAL --exit-code 1 .

# Scan a built image
trivy image --severity HIGH,CRITICAL --exit-code 1 <image>:<tag>

# Scan with JSON output for archiving
trivy image --format json --output report.json <image>:<tag>

# Update Trivy's vulnerability database
trivy image --download-db-only

# Show only fixable CVEs
trivy image --ignore-unfixed <image>:<tag>
```

### Useful Gitleaks commands

```bash
# Scan working directory for secrets
gitleaks detect --source . --exit-code 1

# Scan git history (catches old commits too)
gitleaks detect --source . --log-opts="HEAD~50..HEAD" --exit-code 1

# Generate a baseline to suppress existing findings
gitleaks detect --source . --baseline-path .gitleaks-baseline.json --report-path .gitleaks-baseline.json
```

### Useful SonarQube commands

```bash
# Run sonar-scanner (sonar-project.properties must exist)
sonar-scanner

# Run for a PR
sonar-scanner \
  -Dsonar.pullrequest.key=42 \
  -Dsonar.pullrequest.branch=feature/my-feature \
  -Dsonar.pullrequest.base=main
```

### The DevSecOps invariants

```
1. Secrets never reach Git.   Gitleaks runs before lint. No exceptions.

2. HIGH/CRITICAL CVEs block.  In deps (Trivy fs), in the image (Trivy image),
                               and at runtime (ZAP). Not "noted for later".

3. Quality gates are binary.  SonarQube gate: PASS or FAIL.
                               No "we'll fix it in the next PR".

4. Findings are actionable.   Every blocked build has a linked report artifact.
                               The developer knows exactly what to fix and where.

5. Accepted risks are tracked. Suppressions and .trivyignore entries require
                               a ticket ID, an owner, and an expiry date.
                               Security debt is visible debt.
```

---

