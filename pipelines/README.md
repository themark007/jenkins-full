# 🏠 Jenkins Internals, Demos & Docker CI/CD — A DevOps Engineer's Field Guide

> **Who this is for:** Junior engineers who have Jenkins running and want to understand how it works inside, plus build a real end-to-end CI/CD pipeline that ships a containerized app.  
> **Maintained by:** A DevOps engineer documenting real-world patterns for the team.

---

## 📖 Table of Contents

1. [Inside `$JENKINS_HOME` — The Directory Map](#1-inside-jenkins_home--the-directory-map)
2. [Controller vs Agents — Where Things Live vs Where They Run](#2-controller-vs-agents--where-things-live-vs-where-they-run)
3. [What to Back Up](#3-what-to-back-up)
4. [Mini Demo 1 — Parameters (Boolean + Choice)](#4-mini-demo-1--parameters-boolean--choice)
5. [Mini Demo 2 — Git SCM + Branch as a Parameter](#5-mini-demo-2--git-scm--branch-as-a-parameter)
6. [Mini Demo 3 — Poll SCM Trigger](#6-mini-demo-3--poll-scm-trigger)
7. [Mini Demo 4 — Webhook Trigger (GitHub)](#7-mini-demo-4--webhook-trigger-github)
8. [Mini Demo 5 — Scheduled Nightly Job + Workspace Cleanup](#8-mini-demo-5--scheduled-nightly-job--workspace-cleanup)
9. [Docker-outside-of-Docker (DooD) — Giving Jenkins Docker Access](#9-docker-outside-of-docker-dood--giving-jenkins-docker-access)
10. [End-to-End Demo — Build, Push & Deploy a Flask App](#10-end-to-end-demo--build-push--deploy-a-flask-app)
11. [Quick Reference Cheatsheet](#11-quick-reference-cheatsheet)

---

## 1. Inside `$JENKINS_HOME` — The Directory Map

`$JENKINS_HOME` is the single directory where Jenkins stores **everything** — your jobs, build history, plugins, credentials, and config. On a Docker install this is `/var/jenkins_home`. On a VM install it is typically `/var/lib/jenkins`.

Understanding this directory is essential for backups, debugging, and disaster recovery.

### Full directory map

```
$JENKINS_HOME/
│
├── config.xml                          ← Global Jenkins configuration
├── jenkins.model.JenkinsLocationConfiguration.xml  ← Base URL + admin email
│
├── hudson.tasks.*.xml                  ← Tool configs (Maven, Gradle, Ant, Git)
├── hudson.plugins.*.xml                ← Plugin settings
├── org.jenkinsci.*.xml                 ← Plugin settings (newer style)
├── jenkins.mvn.GlobalMavenConfig.xml   ← Maven global config
├── nodeMonitors.xml                    ← Node health monitor config
├── jenkins.install.*                   ← Install wizard state
├── copy_reference_file.log             ← First-run setup log
│
├── secrets/                            ← 🔐 CRITICAL — Encrypted credential blobs
│     ├── secret.key                    ← Primary encryption key
│     ├── secret.key.not-so-secret      ← Legacy key (keep it)
│     └── identity.key.enc              ← Jenkins instance identity
│
├── jobs/                               ← 📁 ALL your jobs and build history
│     └── my-app/
│           ├── config.xml              ← Job configuration
│           └── builds/
│                 ├── 1/               ← Build #1
│                 │     ├── log        ← Console output
│                 │     └── archive/   ← Archived artifacts
│                 └── 2/              ← Build #2
│
├── plugins/                            ← Installed plugins (.jpi / .hpi files)
│     ├── git.jpi
│     ├── pipeline.jpi
│     └── ...
│
├── users/                              ← Local Jenkins user accounts
│     └── admin_1234/
│           └── config.xml
│
├── tools/                              ← Auto-downloaded tool caches
│     └── hudson.tasks.Maven_MavenInstallation/
│           └── M3/                     ← Maven 3.x downloaded by Jenkins
│
├── workspace/                          ← Build workspaces (controller-run jobs)
│     └── my-app/                       ← Only exists if controller has executors
│
├── logs/                               ← Controller log files (rotated)
├── updates/                            ← Update center cache (safe to recreate)
├── userContent/                        ← Static files Jenkins can serve
└── war/                                ← Jenkins webapp files (the running app)
```

### Why you still see "hudson" everywhere

```
Jenkins history (quick version):

2004  Hudson created at Sun Microsystems
2011  Oracle/community conflict → community forked as "Jenkins"
Now   Jenkins is the active project, but core classes, file names,
      and plugin APIs kept the hudson.* namespace for backward
      compatibility with thousands of existing plugins.

That is why you see:
  hudson.tasks.Maven.xml
  hudson.plugins.git.xml
  hudson.model.Hudson  (in Java code)

It is normal. It is not a mistake.
```

### Jenkins is a Java web app (WAR)

```
Jenkins boots like this:

java -jar jenkins.war
        │
        ▼
  Embedded web server (Winstone / Jetty)
  listens on port 8080 by default
        │
        ▼
  You access Jenkins at http://your-server:8080
        │
        ▼
  Plugins are loaded from $JENKINS_HOME/plugins/
  as .jpi or .hpi files

Jenkins can also sit behind:
  - nginx / traefik reverse proxy (recommended for HTTPS)
  - Kubernetes Ingress (for K8s installs)
  - External servlet container like Tomcat (less common)
```

---

## 2. Controller vs Agents — Where Things Live vs Where They Run

This is one of the most important mental models in Jenkins. **Configuration and history live on the controller. Builds run on agents.**

```
┌──────────────────────────────────────────────────────────────────────────┐
│                     CONTROLLER ($JENKINS_HOME)                           │
│                                                                          │
│  STORES (permanently):                                                   │
│  ├── Job configs          jobs/*/config.xml                              │
│  ├── Build records        jobs/*/builds/*/log + archive/                 │
│  ├── Credentials          secrets/ + credential XML files                │
│  ├── Plugin files         plugins/*.jpi                                  │
│  └── User accounts        users/*/config.xml                             │
│                                                                          │
│  DOES NOT run builds (in production — set executors = 0)                 │
│                                                                          │
└──────────────────────┬───────────────────────────────────────────────────┘
                       │ assigns work via SSH / JNLP / Kubernetes API
          ┌────────────┼─────────────────┐
          ▼            ▼                 ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│   Agent 1    │ │   Agent 2    │ │   Agent 3    │
│   (VM)       │ │  (Docker)    │ │  (K8s Pod)   │
│              │ │              │ │              │
│  HAS:        │ │  HAS:        │ │  HAS:        │
│  Remote root │ │  Remote root │ │  /home/agent │
│  workspace/  │ │  workspace/  │ │  workspace/  │
│  tools/      │ │  tools/      │ │  tools/      │
│              │ │              │ │              │
│  Builds run  │ │  Builds run  │ │  Builds run  │
│  HERE        │ │  HERE        │ │  HERE        │
└──────────────┘ └──────────────┘ └──────────────┘
```

### Key rules to remember

```
✅ Build RECORDS (logs, artifacts) → stored on controller under jobs/
✅ Build EXECUTION (shell commands) → run on the agent
✅ Workspaces → live on the AGENT (its remote root directory)
✅ Controller executors = 0 → controller will not run any builds
   (old workspace/ entries may linger until cleaned)
✅ Agents do NOT have $JENKINS_HOME — they have their own dir
```

### What lives on an agent (not the controller)

```
Agent's remote root (e.g., /home/jenkins/agent/ or /var/jenkins/)
│
├── workspace/
│     ├── my-app/             ← active workspace for "my-app" job
│     ├── my-app@2/           ← second parallel build of same job
│     └── another-job/
│
└── tools/
      └── hudson.tasks.Maven.../  ← Maven downloaded by Jenkins Tools
```

---

## 3. What to Back Up

```
┌──────────────────────────────────────────────────────────────────┐
│                      BACKUP GUIDE                                │
├───────────────────────────────┬──────────────────────────────────┤
│  MUST BACK UP                 │  CAN SKIP (recreated on boot)    │
├───────────────────────────────┼──────────────────────────────────┤
│  jobs/                        │  workspace/                      │
│  plugins/                     │  updates/                        │
│  secrets/                     │  war/                            │
│  users/                       │  tools/ (re-downloaded)          │
│  secret.key                   │  logs/ (optional)                │
│  identity.key.enc             │  copy_reference_file.log         │
│  config.xml                   │                                  │
│  All global *.xml files       │                                  │
└───────────────────────────────┴──────────────────────────────────┘

Restore tips:
  - Keep Jenkins version and plugin versions compatible
  - secret.key MUST match — it decrypts all stored credentials
  - Without secret.key, credential blobs in secrets/ are unreadable
```

### Quick backup command (Docker)

```bash
# Stop Jenkins cleanly before backing up
docker stop jenkins

# Copy the entire jenkins_home volume to a tar archive
docker run --rm \
  -v jenkins_home:/data \
  -v $(pwd):/backup \
  alpine \
  tar czf /backup/jenkins-backup-$(date +%Y%m%d).tar.gz -C /data .

# Restart Jenkins
docker start jenkins
```

---

## 4. Mini Demo 1 — Parameters (Boolean + Choice)

**Goal:** Make a job reusable by adding inputs. Let users toggle tests on/off and pick a target environment. Parameters appear as environment variables inside your build steps.

### Why parameters matter in real projects

```
Without parameters:             With parameters:
───────────────────             ─────────────────
One job = one hardcoded env     One job = handles dev / stage / prod
Need separate jobs per env      Build history shows: WHO ran it,
Messy, hard to audit             WHAT env, WHICH flags
                                 Perfect audit trail for compliance
```

### Setup

In your job config:

```
This project is parameterized → Add Parameter:

1. Boolean Parameter
   Name:    RUN_TESTS
   Default: true (checked)
   Description: Uncheck to skip unit tests (use for urgent hotfixes only)

2. Choice Parameter
   Name:    ENV
   Choices: dev
            stage
            prod
   Description: Target environment for this build
```

### Build step (Execute shell)

```bash
echo "=== Build Parameters ==="
echo "RUN_TESTS = $RUN_TESTS"
echo "ENV       = $ENV"
echo "Build     = #$BUILD_NUMBER"
echo "========================="

# Create a named report file using the env parameter
echo "Report for build #$BUILD_NUMBER (env=$ENV)" > report-${ENV}.txt

if [ "$RUN_TESTS" = "true" ]; then
  echo "Running tests..."
  sleep 1
  echo "All tests passed ✅"
else
  echo "⚠️  Tests skipped — RUN_TESTS=false"
  echo "REASON: Hotfix path; tests must pass on next dev build"
fi
```

### Post-build action

```
Archive the artifacts: report-*.txt
```

### What this looks like in practice

```
Hotfix scenario:
  → Prod is down. Fix is ready. Tests take 20 minutes. SLA is 5 minutes.
  → Set RUN_TESTS=false, ENV=prod → deploy the fix
  → The build record shows: build #47, RUN_TESTS=false, by john.doe, 14:32
  → Immediately trigger build #48 with RUN_TESTS=true on dev to validate
  → Auditors can see exactly which build skipped tests and why
```

---

## 5. Mini Demo 2 — Git SCM + Branch as a Parameter

**Goal:** Let users specify which Git branch to build. Useful for building release branches, hotfix branches, or specific tags without creating duplicate jobs.

### Setup

```
This project is parameterized → Add Parameter:
  String Parameter
    Name:          BRANCH
    Default value: main
    Description:   Git branch, tag, or commit SHA to build

Source Code Management → Git:
  Repository URL: https://github.com/themark/my-app.git
  Credentials:    github-pat  (add if private repo)
  Branches to build: ${BRANCH}     ← uses the parameter
```

### Build step (Execute shell)

```bash
echo "=== Git Build Info ==="
echo "Parameter BRANCH: $BRANCH"
echo "Actual GIT_BRANCH: $GIT_BRANCH"
echo "Commit SHA: $GIT_COMMIT"
echo "Repo URL:   $GIT_URL"
echo "======================"

git --version
echo "Latest 3 commits:"
git log --oneline -n 3 || true
```

### Real-world use cases

```
Release backport:
  Security fix needs to ship on release/2.1
  → Set BRANCH=release/2.1 and run
  → GIT_COMMIT is captured in build record
  → Prove exactly which commit went to which env

Tag-pinned deployments:
  Set BRANCH=refs/tags/v2.1.3
  → Reproducible: same tag always = same commit
  → Easy to diff between releases, easy to roll back

Ops handover:
  SRE can deploy any branch/tag by changing the parameter
  → No need to edit job XML or clone jobs
  → One job handles dev, stage, and prod promotions
```

---

## 6. Mini Demo 3 — Poll SCM Trigger

**Goal:** Jenkins automatically checks the Git repo for new commits and triggers a build if it finds any. This is the firewall-friendly alternative to webhooks.

### Setup

```
Build Triggers → Poll SCM:
  Schedule: H/2 * * * *
```

### Cron syntax quick reference

```
┌─────── minute (0-59)
│ ┌───── hour (0-23)
│ │ ┌─── day of month (1-31)
│ │ │ ┌─ month (1-12)
│ │ │ │ ┌ day of week (0-7, 0=Sun)
│ │ │ │ │
H/2 * * * *    = every ~2 minutes (H adds a random offset)
H   0 * * *    = once a day at midnight (jittered per job)
H  10 * * 1-5  = 10 AM on weekdays
```

### What `H` means

```
Without H:   */2 * * * *
  → Every job polls at minute 0, 2, 4, 6...
  → 50 jobs all hit GitHub at the same second
  → "Thundering herd" — GitHub rate-limits you

With H:      H/2 * * * *
  → Jenkins picks a stable random offset per job
  → job-A polls at :01, :03, :05...
  → job-B polls at :02, :04, :06...
  → Load is spread evenly across the VCS
```

### Build step (Execute shell)

```bash
echo "Build triggered by SCM polling"
echo "Branch: $GIT_BRANCH"
echo "Recent commits:"
git log --oneline -n 3 || true
```

### When to use polling vs webhooks

```
Use POLLING when:                    Use WEBHOOKS when:
──────────────────                   ─────────────────────
Jenkins is behind a firewall         Jenkins has a public URL
Inbound connections are blocked      You want instant builds (seconds)
Working with on-prem Git servers     You want to save compute/API calls
Learning / labs                      Production CI for active teams
```

---

## 7. Mini Demo 4 — Webhook Trigger (GitHub)

**Goal:** GitHub notifies Jenkins the moment code is pushed. Build starts in seconds instead of waiting for the next poll interval.

### Setup — Jenkins side

```
Build Triggers → ✅ GitHub hook trigger for GITScm polling
```

> ⚠️ Also set your Jenkins URL first:  
> `Manage Jenkins → System → Jenkins URL → http://your-public-domain:8080/`  
> Jenkins must be reachable from GitHub's servers for webhooks to work.

### Setup — GitHub side

```
Your GitHub repo → Settings → Webhooks → Add webhook:

  Payload URL:   http://your-jenkins-domain/github-webhook/
  Content type:  application/json
  Secret:        (set a secret and store it in Jenkins credentials)
  Events:        ✅ Just the push event
                 (or add Pull requests for PR builds)

Click: Add webhook
GitHub will send a test ping → you should see a green checkmark ✅
```

### Webhook flow diagram

```
Developer runs: git push
        │
        ▼
GitHub receives the push
        │
        │  HTTP POST to your webhook URL
        │  Payload: branch, commit SHA, pusher, files changed
        ▼
Jenkins receives the webhook at /github-webhook/
        │
        ▼
Jenkins matches the repo + branch to a configured job
        │
        ▼
Build is triggered immediately (seconds, not minutes)
        │
        ▼
Console output: "Started by GitHub push by themark"
```

### Build step (Execute shell)

```bash
echo "=== Webhook-triggered Build ==="
echo "Branch:  $GIT_BRANCH"
echo "Commit:  $GIT_COMMIT"
echo "Build:   #$BUILD_NUMBER"
echo "==============================="
git log -n 1 || true
```

### Polling vs Webhook comparison

```
Polling (H/2 * * * *):  Max 2 min delay, works behind firewall
Webhook:                 Seconds delay, requires public Jenkins URL
```

---

## 8. Mini Demo 5 — Scheduled Nightly Job + Workspace Cleanup

**Goal:** Run a maintenance job at a fixed time every night. Clean up the workspace after to prevent disk bloat over time.

### Setup

```
Build Triggers → Build periodically:
  Schedule: 30 1 * * *
  (runs at 01:30 AM every day)
```

### Build step (Execute shell)

```bash
echo "=== Nightly Maintenance Job ==="
echo "Node:      $NODE_NAME"
echo "Workspace: $WORKSPACE"
echo "Time:      $(date)"

# Write a maintenance report
echo "Nightly run at $(date)" > nightly-report.txt
echo "Node: $NODE_NAME" >> nightly-report.txt
echo "Workspace: $WORKSPACE" >> nightly-report.txt

echo "Simulating nightly tasks..."
echo "  → Generating dependency report..."
sleep 1
echo "  → Refreshing caches..."
sleep 1
echo "  → Done."

# Clean up workspace at the end
echo "Cleaning workspace..."
rm -rf "$WORKSPACE"/* 2>/dev/null || true
echo "Workspace cleaned ✅"
```

### Post-build action (with Workspace Cleanup plugin)

```
Post-build Actions → Delete workspace when build is done
```

Or in a Jenkinsfile:

```groovy
post {
    always {
        cleanWs()    // cleans workspace regardless of build result
    }
}
```

### Why nightly jobs matter

```
Use case                     Why it helps
──────────────────────────   ──────────────────────────────────────
License / SBOM reports       Generate at night, push to S3
                             Keeps release-day builds fast

Dependency cache refresh     Re-warm Maven/npm caches after
                             weekly base image updates

Disk hygiene                 Multibranch workspaces balloon fast
                             Scheduled cleanup prevents agents
                             going read-only during work hours

Integration test suites      Too slow for every PR; run nightly
                             on main for a full regression check
```

---

## 9. Docker-outside-of-Docker (DooD) — Giving Jenkins Docker Access

To build and push container images from Jenkins, Jenkins needs access to a Docker engine. The simplest lab approach is **Docker-outside-of-Docker (DooD)**: mount the host's Docker socket into the Jenkins container so Jenkins can talk to the host's Docker daemon directly.

### DooD vs DinD

```
┌─────────────────────────────────────────────────────────────────────┐
│                    DooD vs DinD                                     │
├─────────────────────────────┬───────────────────────────────────────┤
│  DooD                       │  DinD                                 │
│  (Docker-outside-of-Docker) │  (Docker-in-Docker)                  │
├─────────────────────────────┼───────────────────────────────────────┤
│  Mount host socket:         │  Run a full Docker daemon INSIDE      │
│  /var/run/docker.sock       │  the Jenkins container                │
│                             │                                       │
│  Jenkins uses HOST daemon   │  Jenkins has its OWN daemon           │
│  Containers appear on host  │  Containers are isolated              │
│                             │                                       │
│  ✅ Simple, fast            │  ✅ Fully isolated                    │
│  ✅ Good for labs           │  ❌ Needs --privileged flag           │
│  ❌ Jenkins controls host   │  ❌ Slower (nested virtualization)    │
│     Docker (broad access)   │  ❌ Cache not shared with host        │
└─────────────────────────────┴───────────────────────────────────────┘

Production recommendation:
  Use Docker-enabled agent VMs or daemonless builders:
  Kaniko, BuildKit, or Buildx with a remote driver
  These don't need socket access to the host at all.
```

### What is `/var/run/docker.sock`?

```
On Linux, processes communicate using Unix domain sockets —
special files that act as two-way communication channels.

/var/run/docker.sock is where the Docker daemon listens.

Normal flow:
  docker build .
       │
       │  sends API request through /var/run/docker.sock
       ▼
  Docker daemon (on host)
  → builds the image
  → returns result

DooD flow:
  Jenkins container runs: docker build .
       │
       │  request goes through mounted socket
       │  /var/run/docker.sock → same file on host
       ▼
  Docker daemon (HOST, not inside Jenkins)
  → builds the image on the HOST
  → image appears in host docker images

Result: Jenkins container "controls" the host Docker engine.
Powerful — use with care in labs.
```

### Step 1 — Recreate Jenkins with the Docker socket mounted

Your existing Jenkins data (jobs, plugins, config) is safe because it lives in the `jenkins_home` volume. Recreating the container does not delete the volume.

```bash
# Remove the old container (data is safe in the volume)
docker rm -f jenkins 2>/dev/null || true

# Start Jenkins again with the Docker socket mounted
docker run -d \
  --name jenkins \
  --restart unless-stopped \
  -p 8080:8080 \
  -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -e TZ=Asia/Kolkata \
  jenkins/jenkins:lts
```

**What changed:** The new line `-v /var/run/docker.sock:/var/run/docker.sock` mounts the host socket into the container.

### Step 2 — Allow Jenkins to use the socket

By default the socket is owned by root and not readable by other users:

```bash
# Check socket permissions (you will see something like srw-rw----)
docker exec jenkins ls -l /var/run/docker.sock

# Lab fix: make the socket world-readable/writable
docker exec -u root jenkins chmod 666 /var/run/docker.sock
```

> ⚠️ **Note:** Docker recreates `/var/run/docker.sock` every time the Docker daemon restarts or the machine reboots, so this `chmod` is reset each time. For a more permanent lab setup, add the `jenkins` user to the `docker` group on the host.

### Step 3 — Install the Docker CLI inside Jenkins

We are installing only the **CLI client**, not a daemon. The daemon stays on the host.

```bash
docker exec -u root -it jenkins bash -c \
  'curl -fsSL https://get.docker.com -o /tmp/get-docker.sh && sh /tmp/get-docker.sh'
```

### Step 4 — Verify it works

```bash
docker exec jenkins bash -c 'docker version && echo "---" && docker ps'
```

Expected output:

```
Client: Docker Engine - Community
 Version: 24.x.x
 ...
Server: Docker Engine - Community   ← this is the HOST daemon
 ...
---
CONTAINER ID   IMAGE              ...
abc123         jenkins/jenkins:lts ...   ← your Jenkins container on the host
```

> 💡 If you see the Jenkins container listed in `docker ps` output from inside Jenkins, DooD is working. You are looking at the host's container list.

---

## 10. End-to-End Demo — Build, Push & Deploy a Flask App

**What we will build:**

```
Code pushed to private GitHub repo
        │
        ▼
Jenkins Freestyle job triggers
        │
        ▼
Checks out code from private repo (using PAT credential)
        │
        ▼
Builds Docker image (using DooD → host Docker daemon)
        │
        ▼
Pushes image to private Docker Hub repo
        │
        ▼
Deploys the container on the host (via Docker socket)
        │
        ▼
App is live at http://localhost:5000 ✅
```

### The app — a tiny Flask API

**`app.py`**

```python
from flask import Flask, jsonify

app = Flask(__name__)

@app.get("/")
def hello():
    return jsonify(
        message="Welcome to themark's Jenkins CI/CD demo",
        tip="Built with Flask, shipped by Jenkins, running in Docker."
    )

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

**`requirements.txt`**

```
flask==3.0.3
```

**`Dockerfile`**

```dockerfile
# Small Python base — keeps the final image light
FROM python:3.11-slim

# Working directory inside the container
WORKDIR /app

# Copy dependencies first — Docker caches this layer
# so rebuilds are fast if requirements.txt hasn't changed
COPY requirements.txt .

# Install dependencies (no pip cache stored in the image)
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY app.py .

# Document the port (does not actually publish it)
EXPOSE 5000

# Start the Flask app
CMD ["python", "app.py"]
```

**`.dockerignore`**

```
__pycache__/
*.pyc
.git
*.md
```

> 💡 `.dockerignore` works like `.gitignore` — it tells Docker what NOT to copy into the build context. Smaller context = faster builds.

Upload all four files to the root of your private GitHub repo.

### Setting up credentials in Jenkins

Never paste passwords or tokens into the Execute shell. Use the Jenkins Credentials store. The values are encrypted and masked in logs.

**Step 1 — Create a GitHub Personal Access Token (PAT)**

```
GitHub → Settings → Developer settings → Personal access tokens
  → Fine-grained tokens (recommended):
      Repository access: your private repo only
      Permissions: Contents → Read-only

  → OR Classic token:
      Scopes: repo (for private repo checkout)

Copy the token — you only see it once.
```

**Step 2 — Add GitHub credential in Jenkins**

```
Manage Jenkins → Credentials → System → Global → Add Credentials:
  Kind:     Username with password
  Username: your-github-username
  Password: (paste your PAT here)
  ID:       github-pat
  Description: GitHub PAT for private repo checkout
  → Save
```

**Step 3 — Add Docker Hub credential in Jenkins**

Create a private repo on Docker Hub first (e.g., `themark/flask-demo`).

```
Manage Jenkins → Credentials → System → Global → Add Credentials:
  Kind:     Username with password
  Username: your-dockerhub-username
  Password: your-dockerhub-password-or-token
  ID:       dockerhub
  Description: Docker Hub push credentials
  → Save
```

### Creating the Jenkins job

**New item → `flask-cicd-demo` → Freestyle project → OK**

#### A) Source Code Management → Git

```
Repository URL: https://github.com/themark/your-private-repo.git
Credentials:    github-pat
Branches:       */main
```

#### B) Build Environment

```
✅ Use secret text(s) or file(s)
  → Add → Username and password (separated)
    Credentials:       dockerhub
    Username variable: DOCKERHUB_USER
    Password variable: DOCKERHUB_PWD
```

> This injects Docker Hub credentials as environment variables. The values are masked as `****` in console logs automatically.

#### C) Parameters (optional — makes it cleaner)

```
This project is parameterized → String Parameter:
  Name:    IMAGE_TAG
  Default: ${BUILD_NUMBER}
  Description: Docker image tag (defaults to Jenkins build number)
```

#### D) Build Steps → Execute shell

```bash
#!/bin/bash
set -e    # exit immediately if any command fails

echo "=== Build Context ==="
echo "Job:       $JOB_NAME"
echo "Build:     #$BUILD_NUMBER"
echo "Node:      $NODE_NAME"
echo "Workspace: $WORKSPACE"
echo "Branch:    $GIT_BRANCH"
echo "Commit:    $GIT_COMMIT"
echo "====================="

# ── 1. Docker login ──────────────────────────────────────
echo "Logging in to Docker Hub..."
echo "$DOCKERHUB_PWD" | docker login -u "$DOCKERHUB_USER" --password-stdin
echo "Login successful ✅"

# ── 2. Image naming ───────────────────────────────────────
REGISTRY="docker.io"
REPO="${DOCKERHUB_USER}/flask-demo"
TAG="${BUILD_NUMBER}"
IMAGE="${REGISTRY}/${REPO}"

echo "Image: ${IMAGE}:${TAG}"

# ── 3. Build the image ────────────────────────────────────
echo "Building Docker image..."
docker build \
  --label "git.commit=${GIT_COMMIT}" \
  --label "jenkins.build=${BUILD_NUMBER}" \
  -t "${IMAGE}:${TAG}" \
  -t "${IMAGE}:latest" \
  .
echo "Build complete ✅"

# ── 4. Push to Docker Hub ─────────────────────────────────
echo "Pushing image to Docker Hub..."
docker push "${IMAGE}:${TAG}"
docker push "${IMAGE}:latest"
echo "Push complete ✅"

# ── 5. Deploy on the Docker host (via DooD) ───────────────
echo "Deploying container..."
docker pull "${IMAGE}:${TAG}"
docker rm -f flask-demo 2>/dev/null || true
docker run -d \
  --name flask-demo \
  --restart unless-stopped \
  -p 5000:5000 \
  "${IMAGE}:${TAG}"
echo "Deployed ✅"

# ── 6. Smoke test ─────────────────────────────────────────
echo "Waiting for app to start..."
sleep 3

HTTP_STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:5000)
if [ "$HTTP_STATUS" = "200" ]; then
  echo "Smoke test PASSED ✅ (HTTP $HTTP_STATUS)"
else
  echo "Smoke test FAILED ❌ (HTTP $HTTP_STATUS)"
  exit 1
fi

echo "App is live at http://localhost:5000"
```

#### E) Post-build Actions

```
✅ Delete workspace when build is done   (Workspace Cleanup plugin)
```

### Running it end-to-end

```
Step 1: Commit and push all four files to your private GitHub repo

Step 2: In Jenkins → flask-cicd-demo → Build Now

Step 3: Click the build number → Console Output
        Watch for:
          Login successful ✅
          Build complete ✅
          Push complete ✅
          Deployed ✅
          Smoke test PASSED ✅

Step 4: Open your browser → http://localhost:5000

        Expected response:
        {
          "message": "Welcome to themark's Jenkins CI/CD demo",
          "tip": "Built with Flask, shipped by Jenkins, running in Docker."
        }

Step 5: Verify on the host
        docker ps | grep flask-demo
        docker images | grep flask-demo
```

### What actually happened (full flow)

```
git push (your machine)
        │
        ▼
GitHub stores the code in private repo
        │
        ▼
Jenkins (Build Now or webhook)
  ├── Checkout: git clone using github-pat credential
  ├── Injects: DOCKERHUB_USER + DOCKERHUB_PWD (masked)
  │
  ├── docker build   → sent to HOST daemon via docker.sock
  │                    image built on host
  │
  ├── docker push    → CLI sends to Docker Hub
  │                    image now in your private registry
  │
  ├── docker run     → sent to HOST daemon via docker.sock
  │                    container starts ON THE HOST
  │
  └── curl localhost:5000  → smoke test inside Jenkins container
                             but localhost = the HOST (because DooD)
                             so it hits the running flask-demo container ✅
```

### Troubleshooting common issues

```
Problem: docker: command not found
Fix:     docker exec -u root -it jenkins bash -c \
           'curl -fsSL https://get.docker.com -o /tmp/get-docker.sh && sh /tmp/get-docker.sh'

Problem: permission denied /var/run/docker.sock
Fix:     docker exec -u root jenkins chmod 666 /var/run/docker.sock

Problem: denied: requested access to the resource is denied (push)
Fix:     Check DOCKERHUB_USER and DOCKERHUB_PWD are correct
         Verify Docker Hub repo exists and user has push access

Problem: Smoke test fails (connection refused)
Fix:     Increase sleep before curl (app may need more startup time)
         Check: docker logs flask-demo  for errors

Problem: Cannot checkout private repo
Fix:     Verify github-pat credential has correct username + PAT
         Check PAT has "Contents: Read" permission on the repo
```

---

## 11. Quick Reference Cheatsheet

### `$JENKINS_HOME` at a Glance

```
MUST BACK UP:          jobs/, plugins/, secrets/, users/,
                       secret.key, identity.key.enc, config.xml, *.xml

CAN SKIP:              workspace/, updates/, war/, tools/, logs/

Find home (Docker):    docker exec jenkins sh -c 'echo $JENKINS_HOME'
Find home (VM):        echo $JENKINS_HOME  (usually /var/lib/jenkins)
```

### DooD Setup (3 commands)

```bash
# 1. Recreate Jenkins with socket mounted
docker rm -f jenkins && docker run -d --name jenkins \
  --restart unless-stopped -p 8080:8080 -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -e TZ=Asia/Kolkata jenkins/jenkins:lts

# 2. Fix socket permissions
docker exec -u root jenkins chmod 666 /var/run/docker.sock

# 3. Install Docker CLI
docker exec -u root -it jenkins bash -c \
  'curl -fsSL https://get.docker.com -o /tmp/get-docker.sh && sh /tmp/get-docker.sh'
```

### Trigger types

```
Manual:       Build Now button in the UI
Poll SCM:     H/2 * * * *  → Jenkins checks Git every ~2 min
Webhook:      GitHub pushes to /github-webhook/ → instant
Scheduled:    30 1 * * *  → runs at 01:30 AM nightly
Upstream:     another job finishes → triggers this one
```

### Cron quick reference

```
H/2  * * * *    every ~2 minutes
H    0 * * *    nightly at midnight (jittered)
30   1 * * *    daily at 01:30 AM
H   10 * * 1-5  weekday mornings around 10 AM
0    9 * * 1    every Monday at 09:00 AM
```

### Credential types in Jenkins

```
Secret text           → API tokens, webhook secrets
Username + password   → Docker Hub, GitHub, Nexus logins
SSH private key       → SSH agent connections, Git over SSH
Secret file           → kubeconfig, certificates, .env files
Certificate           → PKI / mutual TLS setups
```

### Security rules

```
✅ Store ALL secrets in Credentials — never in shell steps or Jenkinsfiles
✅ Use "Use secret text(s) or file(s)" in Build Environment to inject creds
✅ Jenkins masks credential values as **** in console logs automatically
✅ Use docker login --password-stdin (not -p in plain text)
✅ Set executors = 0 on controller in production
✅ Keep Jenkins and plugins updated
```

---

## 📚 Further Reading

- [Jenkins Pipeline Syntax Reference](https://www.jenkins.io/doc/book/pipeline/syntax/)
- [Jenkins Credentials Plugin](https://plugins.jenkins.io/credentials/)
- [Jenkins Workspace Cleanup Plugin](https://plugins.jenkins.io/ws-cleanup/)
- [Docker Socket Security — Best Practices](https://docs.docker.com/engine/security/)
- [Kaniko — Daemonless Docker Builds in Kubernetes](https://github.com/GoogleContainerTools/kaniko)
- [BuildKit — Advanced Docker Build Backend](https://docs.docker.com/build/buildkit/)

---

## 🤝 Contributing

Found a mistake or want to add a topic? Open a PR!

- Keep language simple — write for a junior reading this on day 1
- Add a diagram if you are explaining a flow
- Test every command before committing

---

*Last updated: 2025 | Maintained by the DevOps team*