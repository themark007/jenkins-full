# 🔧 Jenkins Basics & Installation — A DevOps Engineer's Field Guide

> **Who this is for:** Junior engineers setting up Jenkins for the first time and anyone who wants to understand how Jenkins works under the hood.  
> **Maintained by:** A DevOps engineer documenting real-world patterns for the team.

---

## 📖 Table of Contents

1. [What is Jenkins?](#1-what-is-jenkins)
2. [Jenkins Architecture — The Big Picture](#2-jenkins-architecture--the-big-picture)
3. [Installation Methods](#3-installation-methods)
4. [Installing Jenkins on Docker (Lab Setup)](#4-installing-jenkins-on-docker-lab-setup)
5. [Installing Jenkins on a VM (Ubuntu)](#5-installing-jenkins-on-a-vm-ubuntu)
6. [Installing Jenkins on Kubernetes](#6-installing-jenkins-on-kubernetes)
7. [Jenkins Plugins — The Power-Ups](#7-jenkins-plugins--the-power-ups)
8. [Jenkins Agents — Where Work Actually Runs](#8-jenkins-agents--where-work-actually-runs)
9. [Jobs — What Jenkins Runs](#9-jobs--what-jenkins-runs)
10. [Workspaces — Where Files Live](#10-workspaces--where-files-live)
11. [Environment Variables — What Jenkins Knows About a Build](#11-environment-variables--what-jenkins-knows-about-a-build)
12. [Artifacts — Saving Build Outputs](#12-artifacts--saving-build-outputs)
13. [Tools — Maven, Git, Helm, and More](#13-tools--maven-git-helm-and-more)
14. [Users and Security](#14-users-and-security)
15. [Quick Reference Cheatsheet](#15-quick-reference-cheatsheet)

---

## 1. What is Jenkins?

Jenkins is an open-source **automation server**. You give it a script (called a pipeline), it runs that script every time something happens (like a code push), and it reports back whether things passed or failed.

Think of Jenkins like a **robot assistant** that sits and watches your Git repo. The moment you push code, the robot wakes up, runs all your checks, and either gives you a green thumbs up ✅ or a red thumbs down ❌.

```
Developer pushes code
         │
         ▼
    Jenkins wakes up  ←── watching Git via webhook
         │
         ▼
    Runs your pipeline:
    Build → Test → Lint → Scan → Deploy
         │
         ▼
    Reports result back
    ✅ Green = all good
    ❌ Red   = something failed
         │
         ▼
    Notifies Slack / Email / GitHub PR status
```

**What Jenkins does — and what it does NOT do:**

| Jenkins DOES | Jenkins does NOT |
|---|---|
| Run pipelines (build, test, deploy) | Store your code (that's GitHub/GitLab) |
| Post status checks to PRs | Host container images (that's Docker Registry) |
| Connect to Vault/AWS for secrets | Store secrets itself (use Vault) |
| Trigger on Git events via webhook | Monitor your app in production (use Grafana) |
| Coordinate all your CI/CD tools | Replace any of those tools |

> 💡 Jenkins is the **conductor** of the orchestra. It tells every other tool when to play, but it doesn't play any instrument itself.

---

## 2. Jenkins Architecture — The Big Picture

Jenkins has two main parts: the **Controller** and the **Agents**.

```
┌─────────────────────────────────────────────────────────────────────┐
│                        JENKINS ARCHITECTURE                         │
│                                                                     │
│  ┌──────────────────────────────────────────────────┐              │
│  │              CONTROLLER (the brain)              │              │
│  │                                                  │              │
│  │  - Web UI (you configure jobs here)              │              │
│  │  - Job scheduler (decides what runs when)        │              │
│  │  - Plugin manager                                │              │
│  │  - Credential store                              │              │
│  │  - Build history and logs                        │              │
│  │                                                  │              │
│  │  ⚠️  Do NOT run heavy builds here                │              │
│  └────────────────────┬─────────────────────────────┘              │
│                       │ assigns work via                            │
│                       │ SSH / JNLP / Kubernetes                     │
│          ┌────────────┼────────────┐                               │
│          ▼            ▼            ▼                               │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                         │
│  │ Agent 1  │  │ Agent 2  │  │ Agent 3  │  ← where builds RUN     │
│  │ (Linux)  │  │ (Docker) │  │ (K8s Pod)│                         │
│  │          │  │          │  │          │                         │
│  │ Has:     │  │ Has:     │  │ Has:     │                         │
│  │ Java     │  │ Maven    │  │ Node.js  │                         │
│  │ Git      │  │ Docker   │  │ Helm     │                         │
│  └──────────┘  └──────────┘  └──────────┘                         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Controller

- The **brain** of Jenkins
- Hosts the web UI, schedules jobs, stores build history
- Should be kept **lean** — do not run builds directly on the controller in production
- Communicates with agents to hand off work

### Agents (also called Nodes)

- The **muscles** — this is where your build commands actually run
- Each agent has its own tools installed (Java, Maven, Docker, etc.)
- Can be a VM, a Docker container, or a Kubernetes Pod
- Agents connect to the controller via SSH or JNLP (Java agent protocol)

> 💡 **In a lab setup** (like running Jenkins on Docker locally), you will run builds on the controller itself because you only have one machine. In real projects, always use separate agents.

---

## 3. Installation Methods

There are three common ways to run Jenkins. Each has its use case:

```
┌────────────────────────────────────────────────────────────────────────┐
│                    JENKINS INSTALLATION OPTIONS                        │
├────────────────┬─────────────────────┬────────────────────────────────┤
│   Method       │   Best For          │   Notes                        │
├────────────────┼─────────────────────┼────────────────────────────────┤
│ Docker         │ Local labs,         │ Fastest to start. Single       │
│                │ quick experiments   │ container. Data in a volume.   │
├────────────────┼─────────────────────┼────────────────────────────────┤
│ VM / Bare      │ Small teams,        │ More control. Direct OS        │
│ metal (Ubuntu) │ on-prem setups      │ access. You manage upgrades.   │
├────────────────┼─────────────────────┼────────────────────────────────┤
│ Kubernetes     │ Large teams,        │ Auto-scaling agents as Pods.   │
│ (Helm chart)   │ production CI/CD    │ High availability. Preferred   │
│                │ at scale            │ for modern setups.             │
└────────────────┴─────────────────────┴────────────────────────────────┘
```

---

## 4. Installing Jenkins on Docker (Lab Setup)

This is the fastest way to get Jenkins running. Perfect for learning and local experiments.

### What you need

- Docker installed on your machine
- At least 2 GB of free RAM
- Port 8080 free (Jenkins UI) and port 50000 free (agent connections)

### The command

```bash
docker run -d \
  --name jenkins \
  --restart unless-stopped \
  -p 8080:8080 \
  -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -e TZ=Asia/Kolkata \
  jenkins/jenkins:lts
```

**What each flag means:**

| Flag | What it does |
|---|---|
| `-d` | Run in background (detached mode) |
| `--name jenkins` | Give the container a friendly name |
| `--restart unless-stopped` | Auto-restart on crash or machine reboot |
| `-p 8080:8080` | Map port 8080 → Jenkins web UI |
| `-p 50000:50000` | Map port 50000 → Agents connect here (JNLP) |
| `-v jenkins_home:/var/jenkins_home` | Persist all Jenkins data in a Docker volume |
| `-e TZ=Asia/Kolkata` | Set timezone (change to yours if needed) |
| `jenkins/jenkins:lts` | Use the Long-Term Support (stable) image |

### How the data is stored

```
Your Machine
│
├── Docker volume: jenkins_home
│       │
│       └── /var/jenkins_home/  (inside container)
│               ├── config.xml          ← Jenkins config
│               ├── jobs/               ← All your jobs
│               │     ├── my-job/
│               │     │     ├── config.xml
│               │     │     └── builds/
│               │     │           ├── 1/   ← Build #1
│               │     │           └── 2/   ← Build #2
│               ├── plugins/            ← Installed plugins
│               ├── secrets/            ← Credentials
│               └── workspace/          ← Where builds run
```

> 💡 The volume `jenkins_home` is the key. Even if the container is deleted, your data survives. Always use a volume — never store data inside the container itself.

### First-time setup

```
1. Run the docker run command above

2. Wait ~30 seconds for Jenkins to start

3. Open your browser → http://localhost:8080

4. Get the initial admin password:
   docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword

5. Paste the password in the browser

6. Choose: "Install suggested plugins"  ← do this for a lab

7. Create your admin user

8. Jenkins is ready ✅
```

### Useful Docker commands for Jenkins

```bash
# Check if Jenkins is running
docker ps | grep jenkins

# View Jenkins logs (live)
docker logs -f jenkins

# Stop Jenkins
docker stop jenkins

# Start Jenkins again
docker start jenkins

# Get inside the container (for installing tools)
docker exec -it jenkins bash

# Check Jenkins version
docker exec jenkins jenkins --version
```

---

## 5. Installing Jenkins on a VM (Ubuntu)

Use this when you want Jenkins running directly on an Ubuntu server (no Docker).

### Install Java first (Jenkins needs it)

```bash
# Update package list
sudo apt-get update

# Install Java 17 (required for Jenkins LTS 2.x)
sudo apt-get install -y fontconfig openjdk-17-jre

# Verify Java
java -version
# Should output: openjdk version "17..."
```

### Install Jenkins

```bash
# Add Jenkins repository key
sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key

# Add Jenkins apt repo
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc]" \
  https://pkg.jenkins.io/debian-stable binary/ | \
  sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

# Update and install Jenkins
sudo apt-get update
sudo apt-get install -y jenkins

# Start Jenkins service
sudo systemctl start jenkins
sudo systemctl enable jenkins   # auto-start on reboot

# Check status
sudo systemctl status jenkins
```

### Where Jenkins files live on a VM

```
/var/jenkins_home/          ← All Jenkins data (jobs, builds, config)
/etc/default/jenkins        ← Jenkins startup config (port, heap, etc.)
/var/log/jenkins/           ← Log files
/usr/share/jenkins/         ← Jenkins WAR file (the app itself)
```

### Change the Jenkins port (if 8080 is taken)

```bash
# Edit the config file
sudo nano /etc/default/jenkins

# Find this line and change the port:
HTTP_PORT=8080
# Change to:
HTTP_PORT=9090

# Restart Jenkins
sudo systemctl restart jenkins
```

---

## 6. Installing Jenkins on Kubernetes

This is the **production-grade** setup. Jenkins runs as a Pod in Kubernetes, and build agents are also Pods that spin up on demand and die after the build finishes.

### Why Kubernetes for Jenkins?

```
Traditional VM agents:               Kubernetes Pod agents:
────────────────────                 ──────────────────────────
Always running (costs money)         Spin up on demand (cost-efficient)
Fixed capacity                       Auto-scale with load
Shared state between builds          Fresh Pod per build (clean state)
Manual to add more agents            Kubernetes adds Pods automatically
```

### Install with Helm (recommended)

```bash
# Add the Jenkins Helm chart repo
helm repo add jenkinsci https://charts.jenkins.io
helm repo update

# Create a namespace for Jenkins
kubectl create namespace jenkins

# Install Jenkins
helm install jenkins jenkinsci/jenkins \
  --namespace jenkins \
  --set controller.serviceType=LoadBalancer \
  --set controller.resources.requests.memory=2Gi \
  --set controller.resources.requests.cpu=1

# Check the pod is running
kubectl get pods -n jenkins

# Get the initial admin password
kubectl exec -n jenkins -it \
  $(kubectl get pods -n jenkins -l app.kubernetes.io/component=jenkins-controller -o name) \
  -- cat /run/secrets/additional/chart-admin-password
```

### How Kubernetes agents work

```
Developer pushes code
        │
        ▼
  Jenkins Controller (Pod)
  receives the build trigger
        │
        ▼
  Kubernetes Plugin tells K8s:
  "Spin up a new agent Pod"
        │
        ▼
  ┌─────────────────────────────┐
  │  Agent Pod spins up         │
  │  (takes ~15-30 seconds)     │
  │                             │
  │  Image has your tools:      │
  │  - Maven / Gradle           │
  │  - Docker (if needed)       │
  │  - Helm / kubectl           │
  └─────────┬───────────────────┘
            │
            ▼
      Build runs inside the Pod
            │
            ▼
      Build finishes
            │
            ▼
      Pod is deleted automatically  ← no leftover state
```

### A basic Kubernetes agent Pod template (in Jenkinsfile)

```groovy
pipeline {
    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: maven
    image: maven:3.9-eclipse-temurin-17
    command: ["sleep"]
    args: ["infinity"]
  - name: docker
    image: docker:24-dind
    securityContext:
      privileged: true
'''
            defaultContainer 'maven'
        }
    }
    stages {
        stage('Build') {
            steps {
                sh 'mvn -B clean package'
            }
        }
    }
}
```

---

## 7. Jenkins Plugins — The Power-Ups

Plugins are how Jenkins connects to everything else. Without plugins, Jenkins is a blank runner. With plugins, it connects to Git, Docker, Kubernetes, Slack, SonarQube, and hundreds more.

```
Jenkins Core (bare)
        │
        │ + Plugins
        ▼
┌────────────────────────────────────────────────┐
│  Jenkins (with plugins)                        │
│                                                │
│  Git Plugin        ← checkout code from Git   │
│  Pipeline Plugin   ← run Jenkinsfiles          │
│  Docker Plugin     ← build/push images         │
│  Kubernetes Plugin ← spin up Pod agents        │
│  Slack Plugin      ← send notifications        │
│  SonarQube Plugin  ← run code quality scans   │
│  Credentials Plugin← store secrets safely     │
│  Blue Ocean        ← prettier UI              │
└────────────────────────────────────────────────┘
```

### Essential plugins to install (recommended for every setup)

| Plugin | Why you need it |
|---|---|
| **Pipeline** | Enables Jenkinsfile (Pipeline-as-Code). Non-negotiable. |
| **Git** | Checkout code from GitHub/GitLab/Bitbucket |
| **GitHub Branch Source** | Auto-discover PRs and branches from GitHub |
| **Credentials Binding** | Use secrets (tokens, passwords) safely in pipelines |
| **Docker Pipeline** | Build and push Docker images inside pipelines |
| **Kubernetes** | Spin up ephemeral Pod agents on Kubernetes |
| **Slack Notification** | Send build results to Slack channels |
| **Blue Ocean** | A much cleaner, visual pipeline UI |
| **OWASP Dependency-Check** | Scan for known CVEs in your dependencies |
| **Timestamper** | Add timestamps to every line in console output |

### How to install plugins

```
Manage Jenkins
    └── Plugins
          └── Available plugins
                └── Search for plugin name
                      └── Install (restart may be required)
```

> ⚠️ **Tip for production:** Keep plugins minimal and curated. Every plugin is a potential security surface and upgrade headache. Only install what you actually use.

---

## 8. Jenkins Agents — Where Work Actually Runs

This is one of the most important things to understand. **Jobs don't run on the controller — they run on agents (nodes).**

### Types of agents

```
┌──────────────────────────────────────────────────────────────────┐
│                      JENKINS AGENT TYPES                         │
├───────────────────┬──────────────────────────────────────────────┤
│  Type             │  How it works                                │
├───────────────────┼──────────────────────────────────────────────┤
│ Built-in (controller) │ Runs on the controller itself.          │
│                   │ Fine for labs. Avoid in production.          │
├───────────────────┼──────────────────────────────────────────────┤
│ SSH Agent         │ Jenkins SSHes into a remote VM and runs      │
│ (permanent)       │ commands there. Always running.              │
│                   │ Good for dedicated build servers.            │
├───────────────────┼──────────────────────────────────────────────┤
│ JNLP / Inbound   │ Agent starts itself and connects to the       │
│ Agent             │ controller. Works behind firewalls.          │
├───────────────────┼──────────────────────────────────────────────┤
│ Docker Agent      │ Jenkins spins up a Docker container for      │
│                   │ the build, runs the job inside, removes it.  │
│                   │ Clean state per build.                        │
├───────────────────┼──────────────────────────────────────────────┤
│ Kubernetes Pod    │ Jenkins creates a K8s Pod, runs the job,     │
│ Agent             │ deletes the Pod. Auto-scales. Best for        │
│                   │ production at scale.                          │
└───────────────────┴──────────────────────────────────────────────┘
```

### How a job finds its agent

In a Jenkinsfile, you declare where you want the job to run using the `agent` directive:

```groovy
// Run on any available agent
pipeline {
    agent any
    ...
}

// Run on an agent with a specific label
pipeline {
    agent { label 'linux-maven' }
    ...
}

// Run inside a specific Docker image
pipeline {
    agent {
        docker { image 'maven:3.9-eclipse-temurin-17' }
    }
    ...
}

// Run on a Kubernetes Pod
pipeline {
    agent {
        kubernetes { ... }
    }
    ...
}
```

### Setting up a permanent SSH agent (VM)

```
In Jenkins UI:

Manage Jenkins
    └── Nodes
          └── New Node
                ├── Name: build-agent-01
                ├── Type: Permanent Agent
                ├── Remote root directory: /home/jenkins/agent
                ├── Labels: linux maven docker
                ├── Launch method: SSH
                │       ├── Host: 192.168.1.50
                │       └── Credentials: (add SSH key)
                └── Save
```

Then on the agent VM:

```bash
# Create jenkins user
sudo useradd -m -d /home/jenkins jenkins

# Java must be installed (same version as controller)
sudo apt-get install -y openjdk-17-jre

# Jenkins will SSH in and start the agent automatically
```

### Agent connection diagram

```
Jenkins Controller
│
│  assigns job to agent
│
├──── SSH ────────▶  VM Agent (permanent)
│                    runs build, sends logs back
│
├──── JNLP ───────▶  Agent behind firewall
│                    agent polls controller
│
└──── K8s API ───▶  Pod Agent (ephemeral)
                     Pod created → build runs → Pod deleted
```

---

## 9. Jobs — What Jenkins Runs

A **job** (also called a project) is the unit of work in Jenkins. It defines: what to run, where to run it, when to trigger it, and what to do after.

### Types of jobs

```
┌──────────────────────────────────────────────────────────────┐
│                     JENKINS JOB TYPES                        │
├───────────────────┬──────────────────────────────────────────┤
│  Job Type         │  What it is                              │
├───────────────────┼──────────────────────────────────────────┤
│ Freestyle         │ Click-through UI configuration.          │
│                   │ Good for learning. Limited power.         │
│                   │ Not versioned in Git.                     │
├───────────────────┼──────────────────────────────────────────┤
│ Pipeline          │ Defined by a Jenkinsfile in your repo.   │
│                   │ Versioned, reviewed, reproducible.        │
│                   │ The standard for real projects.           │
├───────────────────┼──────────────────────────────────────────┤
│ Multibranch       │ Jenkins auto-creates a pipeline per       │
│ Pipeline          │ branch/PR in your repo. Best for teams.  │
├───────────────────┼──────────────────────────────────────────┤
│ Organization      │ Scans all repos in a GitHub/GitLab org.  │
│ Folder            │ Auto-creates Multibranch pipelines.      │
│                   │ Enterprise-scale setup.                   │
└───────────────────┴──────────────────────────────────────────┘
```

### What's inside a Freestyle job

```
Freestyle Job
│
├── General
│     ├── Description
│     ├── Parameters  ← make the job reusable
│     └── Discard old builds (set a limit!)
│
├── Source Code Management
│     └── Git → repo URL + credentials
│
├── Build Triggers
│     ├── Poll SCM  (old way — Jenkins polls Git)
│     └── GitHub hook  (new way — Git pushes to Jenkins via webhook)
│
├── Build Environment
│     └── Inject environment variables, add timestamps, etc.
│
├── Build Steps
│     ├── Execute shell  ← run bash commands
│     ├── Invoke Maven   ← run mvn goals
│     └── Execute Windows batch (if Windows agent)
│
└── Post-build Actions
      ├── Archive artifacts
      ├── Publish test results (JUnit)
      └── Send Slack notification
```

### What's inside a Pipeline job (Jenkinsfile)

```groovy
pipeline {
    agent any                          // where to run

    triggers {
        githubPush()                   // when to run
    }

    environment {
        IMAGE = "myapp"                // shared variables
        REGISTRY = "ghcr.io/acme"
    }

    stages {
        stage('Checkout') {            // step 1
            steps {
                checkout scm
            }
        }

        stage('Build') {               // step 2
            steps {
                sh 'mvn -B clean package -DskipTests'
            }
        }

        stage('Test') {                // step 3
            steps {
                sh 'mvn test'
            }
            post {
                always {
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }

        stage('Deploy to Dev') {       // step 4
            when {
                branch 'main'          // only run on main branch
            }
            steps {
                sh 'helm upgrade --install myapp ./charts/myapp'
            }
        }
    }

    post {
        success { slackSend message: "✅ Build passed: ${env.BUILD_URL}" }
        failure { slackSend message: "❌ Build failed: ${env.BUILD_URL}" }
    }
}
```

### Build lifecycle — what happens when you click "Build Now"

```
Click "Build Now"  (or webhook / schedule triggers)
         │
         ▼
Job is added to the build queue
         │
         ▼
Jenkins finds a free agent matching the job's label
         │
         ▼
Jenkins creates a workspace on the agent
(copies nothing yet — workspace starts empty)
         │
         ▼
Pipeline stages run one by one:
  Stage 1: Checkout → Git clone into workspace
  Stage 2: Build    → mvn compile
  Stage 3: Test     → mvn test
  Stage 4: Deploy   → helm upgrade
         │
         ▼
Post-build actions run (archive, notify, etc.)
         │
         ▼
Build record saved with:
  - Console log
  - Status (SUCCESS / FAILURE / ABORTED)
  - Archived artifacts
  - Duration + timestamp
```

---

## 10. Workspaces — Where Files Live

When a Jenkins job runs, it gets a **workspace** — a folder on the agent where all the files for that build live.

### Where is the workspace?

```
On the agent machine:

/var/jenkins_home/workspace/          ← default root
    │
    ├── my-app/                       ← workspace for job "my-app"
    │     ├── src/                    ← checked-out source code
    │     ├── target/                 ← build output (Maven)
    │     ├── build-report.txt        ← anything your script creates
    │     └── ...
    │
    ├── my-app@2/                     ← second parallel build of same job
    │
    └── another-job/                  ← workspace for a different job
```

### The `$WORKSPACE` environment variable

Inside any build step, `$WORKSPACE` always points to the current job's workspace directory:

```bash
echo "I am running from: $WORKSPACE"
# Output: I am running from: /var/jenkins_home/workspace/my-app

ls $WORKSPACE
# Shows all files in the workspace

# Create a file in the workspace
echo "Build #$BUILD_NUMBER" > $WORKSPACE/build-info.txt
```

### Important things to know about workspaces

```
┌──────────────────────────────────────────────────────────────┐
│                   WORKSPACE FACTS                            │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Workspaces PERSIST between builds (by default)           │
│     → Old files from previous builds may still be there      │
│     → Always clean workspace if you need a fresh start       │
│     → Use: deleteDir() in pipeline or "Clean workspace"      │
│       in Freestyle post-build actions                        │
│                                                              │
│  2. Multiple concurrent builds get separate workspaces       │
│     → my-app/ and my-app@2/ run in parallel safely          │
│                                                              │
│  3. Artifacts are COPIED from workspace to build record      │
│     → The workspace file may be cleaned; artifact is kept    │
│                                                              │
│  4. For Kubernetes Pod agents:                               │
│     → Workspace lives inside the Pod                         │
│     → When Pod is deleted, workspace is gone                 │
│     → Archive anything you need before the Pod dies          │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Cleaning the workspace

```groovy
// In a Jenkinsfile — clean before checkout
pipeline {
    agent any
    stages {
        stage('Clean') {
            steps {
                cleanWs()    // deletes everything in workspace
            }
        }
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
    }
}
```

---

## 11. Environment Variables — What Jenkins Knows About a Build

Jenkins automatically sets a bunch of environment variables for every build. These tell your scripts everything about the current context.

### How to see all environment variables

Two ways:

```bash
# Method 1: From a build step (Execute Shell in Freestyle or sh in Pipeline)
echo "---- ALL ENV VARS ----"
printenv | sort

# Method 2: Visit this URL in your browser
http://localhost:8080/env-vars.html
```

### The most important built-in variables

| Variable | What it contains | Example value |
|---|---|---|
| `$WORKSPACE` | Full path to workspace directory | `/var/jenkins_home/workspace/my-app` |
| `$JOB_NAME` | Name of the job | `my-app` |
| `$BUILD_NUMBER` | Build count (increments each run) | `42` |
| `$BUILD_ID` | Same as build number | `42` |
| `$BUILD_URL` | Full URL to this build | `http://jenkins:8080/job/my-app/42/` |
| `$NODE_NAME` | Which agent ran this build | `build-agent-01` or `built-in` |
| `$JENKINS_URL` | Base URL of Jenkins | `http://jenkins:8080/` |
| `$GIT_COMMIT` | Full SHA of the commit | `9f3c2e7abc123...` |
| `$GIT_BRANCH` | Branch that was checked out | `origin/main` |
| `$GIT_URL` | URL of the Git repo | `https://github.com/acme/loans` |

> 💡 Git variables (`GIT_COMMIT`, `GIT_BRANCH`, `GIT_URL`) are only available after an SCM checkout step.

### A useful "print build context" script

Put this at the start of every job while learning. It tells you exactly what is running where:

```bash
echo "=== BUILD CONTEXT ==="
echo "Job:       $JOB_NAME"
echo "Build:     #$BUILD_NUMBER"
echo "Node:      $NODE_NAME"
echo "Workspace: $WORKSPACE"
echo "Branch:    $GIT_BRANCH"
echo "Commit:    $GIT_COMMIT"
echo "OS:        $(uname -a)"
echo "====================="
```

### Why print build context? Three reasons:

```
1. TRACEABILITY
   Link every log line to the exact build, commit, and agent.
   "Which build shipped v1.4.3?" → JOB_NAME + BUILD_NUMBER + GIT_COMMIT

2. DEBUGGING
   "Why did this fail on agent-02 but not agent-01?"
   → NODE_NAME tells you which agent ran it
   → WORKSPACE tells you where to look for files

3. REPRODUCIBILITY / AUDIT
   Prove exactly what code was built and where.
   Required in regulated environments.
```

### Defining your own environment variables

```groovy
pipeline {
    agent any

    environment {
        // Static values
        APP_NAME = "banking-loans"
        REGISTRY = "ghcr.io/acme"

        // From Jenkins credentials (secret)
        DOCKER_TOKEN = credentials('docker-registry-token')
    }

    stages {
        stage('Info') {
            steps {
                sh 'echo "Building $APP_NAME and pushing to $REGISTRY"'
            }
        }
    }
}
```

---

## 12. Artifacts — Saving Build Outputs

An **artifact** is any file produced by a build that you want to save and access later. Examples: compiled JARs, test reports, coverage reports, Docker image tarballs, release manifests.

### How artifacts work

```
Build runs
    │
    ▼
Files created in $WORKSPACE:
    ├── target/my-app-1.0.jar   ← compiled JAR
    ├── target/surefire-reports/ ← test results
    └── build-report.txt        ← custom report
    │
    ▼
"Archive artifacts" step copies files
from workspace → build record
    │
    ▼
Build record stores them permanently
(even if workspace is cleaned later)
    │
    ▼
Available at:
  Jenkins UI → Job → Build #42 → Artifacts tab
  URL: http://jenkins:8080/job/my-app/42/artifact/
```

### Archiving artifacts in Freestyle

```
Post-build Actions
    └── Archive the artifacts
          └── Files to archive: target/*.jar, build-report.txt
              (use comma-separated glob patterns)
```

### Archiving artifacts in Pipeline

```groovy
stage('Archive') {
    steps {
        // Archive the JAR
        archiveArtifacts artifacts: 'target/*.jar', fingerprint: true

        // Publish JUnit test results
        junit 'target/surefire-reports/*.xml'
    }
}
```

### A simple example — create and archive a report

```bash
# In Execute Shell (Freestyle) or sh step (Pipeline):

# Create a report
echo "=== Build Report ===" > build-report.txt
echo "Job:    $JOB_NAME" >> build-report.txt
echo "Build:  #$BUILD_NUMBER" >> build-report.txt
echo "Commit: $GIT_COMMIT" >> build-report.txt
echo "Node:   $NODE_NAME" >> build-report.txt
echo "Time:   $(date)" >> build-report.txt

# Show in console log too
cat build-report.txt

# List workspace contents
ls -lh
```

Then archive `build-report.txt` and it will appear in the **Artifacts** tab of the build.

---

## 13. Tools — Maven, Git, Helm, and More

"Tools" in Jenkins refers to the CLI programs your build steps need. If the tool is not installed on the agent, the command fails.

### Two ways to provide tools

```
┌──────────────────────────────────────────────────────────────────┐
│                    TWO APPROACHES TO TOOLS                       │
├─────────────────────────────┬────────────────────────────────────┤
│  A) Manage Jenkins → Tools  │  B) Install on the OS directly     │
│  (Jenkins-managed)          │  (manual)                          │
├─────────────────────────────┼────────────────────────────────────┤
│  Jenkins downloads and      │  You apt-get / yum install the     │
│  manages the tool version   │  tool on the agent VM/container    │
│  per job configuration      │                                    │
├─────────────────────────────┼────────────────────────────────────┤
│ ✅ Versioned per job        │ ✅ Works with plain shell steps     │
│ ✅ No OS root needed        │ ✅ Quick for local labs             │
│ ✅ Reproducible across nodes│ ❌ Version drift between nodes      │
│ ❌ Slightly more config     │ ❌ Manual to update                 │
│    (need a bridge plugin    │                                    │
│    for plain shell steps)   │                                    │
└─────────────────────────────┴────────────────────────────────────┘
```

### Approach A — Using Manage Jenkins → Tools (Maven example)

**Step 1: Configure the tool once in Jenkins**

```
Manage Jenkins
    └── Tools
          └── Maven installations
                └── Add Maven
                      ├── Name: M3
                      ├── Install automatically: ✅
                      └── Version: 3.9.6
```

**Step 2: Use in a Freestyle job**

```
Build Steps → Add build step → Invoke top-level Maven targets
    ├── Maven Version: M3
    └── Goals: -B clean test package
```

**Step 3: Use in a Jenkinsfile**

```groovy
pipeline {
    agent any
    tools {
        maven 'M3'    // matches the name you set in Tools
        jdk 'JDK17'
    }
    stages {
        stage('Build') {
            steps {
                sh 'mvn -B clean package'   // works because Tools added mvn to PATH
            }
        }
    }
}
```

### Approach B — Installing directly on the node (quick lab hack)

```bash
# Get inside the Jenkins container (for Docker lab setup)
docker exec -it jenkins bash

# Install Git
apt-get update && apt-get install -y git
git --version

# Install Maven
apt-get install -y maven
mvn -v

# Install Helm (example)
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

Then your Execute Shell / sh steps can use these commands directly.

### Which approach to pick?

```
Learning / quick lab demo
    → OS install (B) — fast, no configuration needed

Real team projects / multiple agents
    → Manage Jenkins → Tools (A) — versioned, reproducible, scales

Docker / Kubernetes agents (modern approach)
    → Bake tools into the agent Docker image
      (neither A nor B — the image has everything pre-installed)
```

### Tool check script — always run this to debug missing tools

```bash
echo "=== Tool Versions ==="
echo "Java:   $(java -version 2>&1 | head -1)"
echo "Maven:  $(mvn -v 2>/dev/null | head -1 || echo NOT FOUND)"
echo "Git:    $(git --version 2>/dev/null || echo NOT FOUND)"
echo "Docker: $(docker --version 2>/dev/null || echo NOT FOUND)"
echo "Helm:   $(helm version --short 2>/dev/null || echo NOT FOUND)"
echo "Node:   $(node --version 2>/dev/null || echo NOT FOUND)"
echo "====================="
```

---

## 14. Users and Security

Jenkins has its own user management system. By default, after setup, only you (the admin) exist. You should create proper users before sharing Jenkins with your team.

### User roles — who can do what

```
┌──────────────────────────────────────────────────────────────────┐
│                      JENKINS USER ROLES                          │
├─────────────────┬────────────────────────────────────────────────┤
│  Role           │  What they can do                              │
├─────────────────┼────────────────────────────────────────────────┤
│ Admin           │ Everything — manage nodes, plugins,            │
│                 │ system config, all jobs, all users             │
├─────────────────┼────────────────────────────────────────────────┤
│ Developer       │ Create/run/configure their own jobs.           │
│                 │ View build logs. Cannot change system config.  │
├─────────────────┼────────────────────────────────────────────────┤
│ Read-Only       │ View builds and logs. Cannot run or modify.    │
│ (viewer)        │ Good for stakeholders / QA.                    │
├─────────────────┼────────────────────────────────────────────────┤
│ Service Account │ A machine user for automation (GitHub → Jenkins│
│                 │ webhook triggers, API calls). No UI login.     │
└─────────────────┴────────────────────────────────────────────────┘
```

### Creating a user

```
Manage Jenkins
    └── Users
          └── Create User
                ├── Username: john.doe
                ├── Password: (strong password)
                ├── Full Name: John Doe
                └── Email: john@acme.com
```

### Setting permissions (with Matrix Authorization plugin)

```
Manage Jenkins
    └── Security
          └── Authorization: Matrix-based security
                ├── Anonymous users: Read only (or none)
                ├── john.doe: Job/Build, Job/Read, Job/Workspace
                └── admin: Overall/Administer (full access)
```

### Credentials — storing secrets safely

Never hardcode passwords, tokens, or keys in a Jenkinsfile. Use the Credentials store:

```
Manage Jenkins
    └── Credentials
          └── System → Global credentials → Add Credentials
                ├── Kind: Secret text (for API tokens)
                ├── Kind: Username with password (for registries, Git)
                ├── Kind: SSH Username with private key (for SSH agents)
                └── Kind: Secret file (for kubeconfig, certificates)
```

**Using a credential in a Jenkinsfile:**

```groovy
pipeline {
    agent any
    environment {
        // Binds credential to env var (value is masked in logs)
        DOCKER_PASSWORD = credentials('docker-registry-password')
        GITHUB_TOKEN    = credentials('github-pat-token')
    }
    stages {
        stage('Push Image') {
            steps {
                sh '''
                    echo $DOCKER_PASSWORD | docker login ghcr.io -u myuser --password-stdin
                    docker push ghcr.io/acme/myapp:latest
                '''
            }
        }
    }
}
```

> ⚠️ Jenkins automatically **masks** credentials in logs — they appear as `****`. Never `echo` a secret directly or it will be masked but still visible in log context.

### Security checklist for Jenkins

```
✅ Do not allow anonymous users to do anything
✅ Use Matrix Authorization or Role-Based Authorization plugin
✅ Store all secrets in Credentials (never in Jenkinsfile)
✅ Use HTTPS (put Jenkins behind nginx/traefik with TLS)
✅ Keep Jenkins and plugins updated (security patches)
✅ Do NOT run builds on the controller in production
✅ Restrict which users can install plugins (admins only)
✅ Set up LDAP/SSO for teams (avoids managing 50 local users)
```

---

## 15. Quick Reference Cheatsheet

### Installation at a Glance

```
Docker (lab)     →  docker run -d -p 8080:8080 -v jenkins_home:/var/jenkins_home jenkins/jenkins:lts
VM (Ubuntu)      →  apt-get install jenkins (after adding repo + Java 17)
Kubernetes       →  helm install jenkins jenkinsci/jenkins --namespace jenkins
```

### Where Things Live

```
Jenkins home:    /var/jenkins_home/
Jobs:            /var/jenkins_home/jobs/<job-name>/
Builds:          /var/jenkins_home/jobs/<job-name>/builds/<number>/
Workspace:       /var/jenkins_home/workspace/<job-name>/
Plugins:         /var/jenkins_home/plugins/
Credentials:     /var/jenkins_home/credentials.xml
Logs (Docker):   docker logs -f jenkins
Logs (VM):       /var/log/jenkins/jenkins.log
```

### Key Environment Variables

```
$WORKSPACE      → workspace directory path
$JOB_NAME       → job name
$BUILD_NUMBER   → build number
$BUILD_URL      → full URL to this build
$NODE_NAME      → agent that ran the build
$GIT_COMMIT     → git commit SHA (after checkout)
$GIT_BRANCH     → git branch (after checkout)
```

### The 5 Rules for a Clean Jenkins Setup

```
1. Never run builds on the controller in production — use agents.
2. Store all secrets in Credentials — never hardcode in Jenkinsfiles.
3. Use Jenkinsfiles in your repo — pipeline as code, versioned and reviewed.
4. Keep plugins minimal — only install what you actively use.
5. Archive anything you need — workspace files may be cleaned after a build.
```

### Job types — when to use which

```
Freestyle      → Learning, simple one-off tasks, quick experiments
Pipeline       → Any real project with a Jenkinsfile in the repo
Multibranch    → Teams with multiple feature branches and PRs
Org Folder     → Enterprise with many repos in one GitHub/GitLab org
```

---

## 📚 Further Reading

- [Jenkins Official Documentation](https://www.jenkins.io/doc/)
- [Jenkins Pipeline Syntax Reference](https://www.jenkins.io/doc/book/pipeline/syntax/)
- [Jenkins Plugins Index](https://plugins.jenkins.io/)
- [Kubernetes Plugin for Jenkins](https://plugins.jenkins.io/kubernetes/)
- [Jenkins on Kubernetes — Official Guide](https://www.jenkins.io/doc/book/installing/kubernetes/)
- [Jenkins Security — Best Practices](https://www.jenkins.io/doc/book/security/)

---
