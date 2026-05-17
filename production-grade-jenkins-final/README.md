# Production CI/CD with Jenkins Multi-Branch Pipelines & ArgoCD

> A complete reference for building a production-grade GitOps pipeline:
> developer pushes a branch → Jenkins MBP builds and tags the image →
> PR gates the merge → ArgoCD deploys from the updated manifest → cleanup is automatic.

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Repository Layout](#2-repository-layout)
3. [The Full Developer Flow](#3-the-full-developer-flow)
4. [Image Tagging Strategy](#4-image-tagging-strategy)
5. [Jenkins Multi-Branch Pipeline — Setup](#5-jenkins-multi-branch-pipeline--setup)
6. [The Jenkinsfile](#6-the-jenkinsfile)
7. [Branch Protection Rules](#7-branch-protection-rules)
8. [ArgoCD Application — Setup & Config](#8-argocd-application--setup--config)
9. [Manifest Repository & Auto-Update](#9-manifest-repository--auto-update)
10. [Secrets Management](#10-secrets-management)
11. [Cleanup — Branches, Jobs, Images](#11-cleanup--branches-jobs-images)
12. [Quick Reference Cheatsheet](#12-quick-reference-cheatsheet)

---

## 1. Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        PRODUCTION CI/CD ARCHITECTURE                            │
│                                                                                 │
│  DEVELOPER                APP REPO                JENKINS (CI)                 │
│  ──────────               ──────────               ────────────                │
│                                                                                 │
│  git switch -c            feature/<name>           MBP detects branch          │
│  feature/<name>           │                        → branch job created        │
│       │                   │                        → build runs on push        │
│       │  git push         ▼                                │                   │
│       └────────────▶  branch push                  ┌───────▼──────────┐       │
│                                                     │  Branch Job      │       │
│                                                     │  lint → build    │       │
│                                                     │  test → image    │       │
│                                                     │  tag: branch-N   │       │
│                                                     └───────┬──────────┘       │
│                                                             │                  │
│  git open PR                                         image pushed to           │
│       │                                              registry (branch tag)     │
│       ▼                                                     │                  │
│   PR opened            PR-N branch                  ┌───────▼──────────┐       │
│   feature → main       created in MBP               │   PR Job         │       │
│                                                     │  lint → build    │       │
│                                                     │  test → image    │       │
│                                                     │  scan → deploy   │       │
│                                                     │  smoke test      │       │
│                                                     │  tag: pr-N-sha   │       │
│                                                     └───────┬──────────┘       │
│                                                             │                  │
│  ✅ checks green                                     status posted to PR       │
│  ✅ review approved                                         │                  │
│  merge PR                                                   │                  │
│       │                                                     │                  │
│       ▼                                                     ▼                  │
│   main updated         main job                     ┌──────────────────┐       │
│                        triggered                    │   Main Job       │       │
│                                                     │  build once      │       │
│                                                     │  tag: sha + ver  │       │
│                                                     │  push :latest    │       │
│                                                     │  write digest    │       │
│                                                     │  update manifest │ ◀─┐  │
│                                                     └──────────────────┘   │  │
│                                                                             │  │
│                                                                             │  │
│  MANIFEST REPO          ARGOCD (CD)                                        │  │
│  ────────────           ───────────                                        │  │
│                                                                             │  │
│  manifests/             watches manifest repo                               │  │
│  └── deploy/            ← image tag updated by Jenkins ────────────────────┘  │
│       └── values.yaml   ← ArgoCD detects drift                                │
│                                ↓                                               │
│                         syncs Kubernetes cluster                               │
│                                ↓                                               │
│                         new pods roll out                                      │
│                                ↓                                               │
│                         health checks pass                                     │
│                                ↓                                               │
│                         deployment complete ✅                                 │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Component responsibilities

| Component | Responsibility |
|---|---|
| **Jenkins MBP** | Detect branches/PRs, build images, run tests, update manifests |
| **Container Registry** | Store immutable image artifacts (Docker Hub, GHCR, ECR) |
| **App Repo** | Application code + `Dockerfile` + `Jenkinsfile` |
| **Manifest Repo** | Kubernetes manifests / Helm values (separate Git repo) |
| **ArgoCD** | Watch manifest repo, sync cluster to declared state |
| **Branch protection** | Gate merges on green CI + review approval |

---

## 2. Repository Layout

Two repositories. Keep them separate — mixing CI config and deployment config in one repo creates a circular dependency (the CI job that updates the manifest would trigger itself).

### App Repository

```
app-repo/
├── .github/
│   └── CODEOWNERS                   ← who reviews which paths
├── src/                             ← application source
├── tests/
│   ├── unit/
│   └── integration/
├── Dockerfile                       ← multi-stage, pinned base image
├── .dockerignore
├── Jenkinsfile                      ← pipeline definition (see Section 6)
└── scripts/
    └── update-manifest.sh           ← helper: bumps image tag in manifest repo
```

### Manifest Repository

```
manifest-repo/
├── apps/
│   └── <service-name>/
│       ├── base/
│       │   ├── deployment.yaml
│       │   ├── service.yaml
│       │   └── kustomization.yaml
│       └── overlays/
│           ├── dev/
│           │   └── kustomization.yaml   ← image: <registry>/<image>:<branch-tag>
│           ├── stage/
│           │   └── kustomization.yaml   ← image: <registry>/<image>:<sha-version>
│           └── prod/
│               └── kustomization.yaml   ← image: <registry>/<image>:<sha-version>
└── argocd/
    ├── app-dev.yaml                 ← ArgoCD Application manifest for dev
    ├── app-stage.yaml
    └── app-prod.yaml
```

---

## 3. The Full Developer Flow

```
Step 1 ── Create a feature branch

  git switch main && git pull
  git switch -c feature/<ticket-id>-<short-description>

  Naming convention:
    feature/<id>-<description>     new functionality
    bugfix/<id>-<description>      fixes
    hotfix/<id>-<description>      urgent production fixes
    chore/<id>-<description>       non-functional changes

──────────────────────────────────────────────────────────────────────

Step 2 ── Push → MBP creates a Branch Job

  git add .
  git commit -m "feat(<scope>): <what and why>"
  git push origin feature/<ticket-id>-<short-description>

  What Jenkins does automatically:
    - MBP detects the new branch (webhook) or next scheduled scan
    - Creates child job: feature/<ticket-id>-<short-description>
    - Runs the branch pipeline:
        lint → unit tests → build image → push with branch tag
    - Target: ≤ 5 minutes
    - Image tag: <branch-slug>-<build-number>
        e.g. feature-123-login-fix-7

  Subsequent pushes re-trigger the branch job.
  Once a PR is opened, the branch job pauses (strategy: Exclude PRs).

──────────────────────────────────────────────────────────────────────

Step 3 ── Open PR → MBP creates a PR Job

  Open PR:  base = main,  compare = feature/<ticket-id>-...

  What Jenkins does:
    - MBP detects the PR (webhook or scan)
    - Creates child job: PR-<N>
    - Runs the PR pipeline (Head + Merge):
        lint → unit tests → build image → image scan
        → deploy ephemeral preview → integration + smoke tests
        → post status checks to the PR
    - Image tag: pr-<N>-<short-sha>
        e.g.  pr-42-9f3c2e7

  Status checks posted back to GitHub:
    ci/lint
    ci/unit-tests
    ci/image-scan
    ci/integration-tests
    ci/smoke-test

  All five must be green before the merge button unlocks.

──────────────────────────────────────────────────────────────────────

Step 4 ── Merge → MBP runs Main Job → ArgoCD deploys

  After merge:
    - MBP detects new commit on main
    - Runs the main pipeline:
        build once → push immutable image
        tag: <short-sha>-<version>   AND   :latest
        capture digest sha256:...
        write deploy-info artifact
        commit new image tag to manifest repo (dev overlay)
    - ArgoCD detects diff in manifest repo
    - ArgoCD syncs cluster → new pods roll out
    - Health checks pass → deployment complete

──────────────────────────────────────────────────────────────────────

Step 5 ── Promote to Stage / Prod via Promotion PR

  Promote to Stage:
    - Open PR in manifest repo:  dev overlay → stage overlay
    - CI verifies manifest integrity
    - Deploy to Stage by digest (same image, different config)
    - Integration + E2E tests run against Stage
    - All checks green + TL approval → merge
    - ArgoCD syncs Stage cluster

  Promote to Prod (same pattern, stricter gates):
    - Open PR:  stage overlay → prod overlay
    - Requires 2 approvals + change window
    - Canary rollout: 5% → 20% → 50% → 100%
    - SLO guardrails; auto-rollback on breach

──────────────────────────────────────────────────────────────────────

Step 6 ── Automatic Cleanup

  - Git host deletes feature branch after merge (auto-delete enabled)
  - MBP orphan-cleanup retires the branch/PR job and workspace
  - Preview environment torn down by pipeline post-step
  - Old branch-tagged images pruned by registry retention policy
```

---

## 4. Image Tagging Strategy

Every image gets multiple tags that serve different audiences. **Always deploy by digest** — tags are for humans.

```
┌─────────────────────────────────────────────────────────────────────┐
│                     IMAGE TAGGING MATRIX                            │
├─────────────────┬──────────────────┬───────────────────────────────┤
│  Stage          │  Tag format      │  Example                      │
├─────────────────┼──────────────────┼───────────────────────────────┤
│ Branch build    │ <branch>-<build> │ feature-login-fix-7           │
│                 │                  │ (mutable, dev preview only)   │
├─────────────────┼──────────────────┼───────────────────────────────┤
│ PR build        │ pr-<N>-<sha7>    │ pr-42-9f3c2e7                 │
│                 │                  │ (traceability to PR + commit) │
├─────────────────┼──────────────────┼───────────────────────────────┤
│ Main build      │ <sha7>-<version> │ 9f3c2e7-v1.4.3               │
│  (primary)      │                  │ (immutable production tag)    │
├─────────────────┼──────────────────┼───────────────────────────────┤
│ Main build      │ <version>        │ v1.4.3                        │
│  (semver)       │                  │ (mutable; changelog anchor)   │
├─────────────────┼──────────────────┼───────────────────────────────┤
│ Main build      │ latest           │ latest                        │
│  (latest)       │                  │ (mutable; never deploy this)  │
├─────────────────┼──────────────────┼───────────────────────────────┤
│ Immutable ref   │ sha256:<digest>  │ sha256:abc123...              │
│  (deploy this)  │                  │ (what actually goes to K8s)   │
└─────────────────┴──────────────────┴───────────────────────────────┘
```

### The deploy-info artifact

Every successful main build writes and archives this file. It is the passport for the artifact as it moves through environments.

```json
{
  "image":      "<registry>/<repo>/<service>",
  "digest":     "sha256:abc123...",
  "tags":       ["9f3c2e7-v1.4.3", "v1.4.3", "latest"],
  "version":    "v1.4.3",
  "commit_sha": "9f3c2e7abc123...",
  "branch":     "main",
  "build":      "42",
  "build_url":  "https://jenkins.example.com/job/my-service/42/",
  "built_at":   "2025-09-15T10:22:00Z"
}
```

---

## 5. Jenkins Multi-Branch Pipeline — Setup

### 5.1 Prerequisites

- Jenkins LTS with the following plugins:
  - Pipeline
  - GitHub Branch Source (or equivalent for your Git host)
  - Credentials Binding
  - Docker Pipeline
  - Git
  - Workspace Cleanup
  - Blue Ocean (optional, better UI)
- GitHub credentials stored in Jenkins (PAT or GitHub App)
- Docker registry credentials stored in Jenkins
- A deploy key or PAT with write access to the **manifest repo** stored in Jenkins

### 5.2 Create the MBP Job

```
New Item
  → Name: <service-name>
  → Type: Multibranch Pipeline
  → OK
```

### 5.3 Branch Sources Configuration

```
Branch Sources → Add source → GitHub

  Credentials:      <your-github-credential-id>
  Repository URL:   https://github.com/<org>/<app-repo>.git

  Behaviors (add all of these):
    ✅ Discover branches
    ✅ Discover pull requests from origin
    ✅ Discover pull requests from forks
```

### 5.4 Branch Build Strategy

```
Filter by name (with regular expression):
  Include: ^(main|release/.+|feature/.+|bugfix/.+|hotfix/.+|chore/.+)$
  (adjust to your naming convention)

Build strategies — select:
  ✅ Exclude branches that are also filed as PRs
```

This means:
- A branch job builds on every push until a PR is opened
- Once a PR is open, only the PR job builds
- Branch job resumes if the PR is closed without merging

### 5.5 PR Discovery Strategy

```
Discover pull requests from origin:
  Strategy: Both the current pull request revision and the pull request
            merged with the current target branch revision
            (Head AND Merge — gives you both signals)

Trust:
  Collaborators and members of the organization with write access
  (limits secret exposure for fork PRs)
```

### 5.6 Orphan Job Cleanup

```
Orphaned Item Strategy:
  ✅ Discard old items
    Days to keep old items: 7
    Max # of old items to keep: 20
```

### 5.7 Scan Triggers

```
Scan Repository Triggers:
  ✅ Periodically if not otherwise run
    Interval: 1 hour
    (safety net if webhook misses an event)
```

Webhook is the primary trigger — configure it in your GitHub repo:

```
GitHub repo → Settings → Webhooks → Add webhook
  Payload URL:  https://<jenkins-domain>/github-webhook/
  Content type: application/json
  Secret:       <webhook-secret stored in Jenkins credentials>
  Events:       Pushes + Pull requests
```

---

## 6. The Jenkinsfile

Place this at the root of your app repository. It handles all three pipeline stages: branch builds, PR builds, and main builds.

```groovy
// ─────────────────────────────────────────────────────────────────────────────
// Jenkinsfile — Production Multi-Branch Pipeline
//
// Stages:
//   Branch job  → lint + unit tests + image build (branch tag, no deploy)
//   PR job      → lint + tests + image build + scan + preview + integration
//   Main job    → build once + tag + push + update manifest → ArgoCD deploys
// ─────────────────────────────────────────────────────────────────────────────

// ── Pipeline-level settings ──────────────────────────────────────────────────
def IMAGE_REGISTRY  = "docker.io"                     // or ghcr.io / <ecr-url>
def IMAGE_REPO      = "<your-dockerhub-org>/<service-name>"
def MANIFEST_REPO   = "https://github.com/<org>/<manifest-repo>.git"
def MANIFEST_BRANCH = "main"
def SERVICE_NAME    = "<service-name>"                // used for container names

// Derived at runtime
def IMAGE_FULL      = "${IMAGE_REGISTRY}/${IMAGE_REPO}"
def SHORT_SHA       = ""
def IMAGE_TAG       = ""
def IMAGE_DIGEST    = ""

pipeline {

    // ── Agent ─────────────────────────────────────────────────────────────────
    // Use a label that matches your Docker-capable agent(s).
    // For Kubernetes: replace with a kubernetes { yaml '...' } block.
    agent { label 'docker-agent' }

    // ── Options ───────────────────────────────────────────────────────────────
    options {
        buildDiscarder(logRotator(numToKeepStr: '20', artifactNumToKeepStr: '10'))
        disableConcurrentBuilds()          // one build at a time per branch
        timestamps()                       // prefix every log line with time
        timeout(time: 30, unit: 'MINUTES') // kill runaway builds
    }

    // ── Triggers ──────────────────────────────────────────────────────────────
    triggers {
        // Primary: GitHub webhook fires this automatically.
        // This line is the Jenkinsfile-side declaration of that trigger.
        githubPush()
    }

    // ── Environment ───────────────────────────────────────────────────────────
    environment {
        // Registry credentials injected from Jenkins Credentials store.
        // The binding creates REGISTRY_CREDS_USR and REGISTRY_CREDS_PSW.
        REGISTRY_CREDS    = credentials('registry-credentials')

        // PAT with write access to the manifest repo (for tag updates).
        MANIFEST_PAT      = credentials('manifest-repo-pat')

        // Used by trivy image scanning; set to 'HIGH,CRITICAL' to block on those.
        TRIVY_SEVERITY    = "HIGH,CRITICAL"
    }

    // ── Stages ────────────────────────────────────────────────────────────────
    stages {

        // ── Stage 0: Context & Setup ──────────────────────────────────────────
        stage('Setup') {
            steps {
                script {
                    SHORT_SHA = sh(
                        script: "git rev-parse --short=7 HEAD",
                        returnStdout: true
                    ).trim()

                    // Tag logic differs per pipeline type
                    if (env.CHANGE_ID) {
                        // PR job  →  pr-<PR_number>-<sha>
                        IMAGE_TAG = "pr-${env.CHANGE_ID}-${SHORT_SHA}"
                    } else if (env.BRANCH_NAME == 'main') {
                        // Main job  →  <sha>-<version>  (version from git tag or BUILD_NUMBER)
                        def version = sh(
                            script: "git describe --tags --abbrev=0 2>/dev/null || echo v0.${BUILD_NUMBER}",
                            returnStdout: true
                        ).trim()
                        IMAGE_TAG = "${SHORT_SHA}-${version}"
                    } else {
                        // Branch job  →  <branch-slug>-<build>
                        def branchSlug = env.BRANCH_NAME.replaceAll('[^a-zA-Z0-9]', '-').toLowerCase()
                        IMAGE_TAG = "${branchSlug}-${BUILD_NUMBER}"
                    }

                    echo "=== Build Context ==="
                    echo "Job:        ${JOB_NAME}"
                    echo "Build:      #${BUILD_NUMBER}"
                    echo "Branch:     ${env.BRANCH_NAME}"
                    echo "PR:         ${env.CHANGE_ID ?: 'N/A'}"
                    echo "Commit:     ${SHORT_SHA}"
                    echo "Image tag:  ${IMAGE_TAG}"
                    echo "Pipeline:   ${env.CHANGE_ID ? 'PR' : (env.BRANCH_NAME == 'main' ? 'MAIN' : 'BRANCH')}"
                    echo "===================="
                }
            }
        }

        // ── Stage 1: Lint ─────────────────────────────────────────────────────
        // Runs on: branch, PR, main
        stage('Lint') {
            steps {
                // Replace with your linter. Examples:
                //   Python:     sh 'flake8 src/ && black --check src/'
                //   Node.js:    sh 'npm run lint'
                //   Go:         sh 'golangci-lint run ./...'
                //   Java/Maven: sh 'mvn checkstyle:check'
                sh '''
                    echo "Running linter..."
                    # INSERT YOUR LINT COMMAND HERE
                    echo "Lint passed ✅"
                '''
            }
        }

        // ── Stage 2: Unit Tests ───────────────────────────────────────────────
        // Runs on: branch, PR, main
        stage('Unit Tests') {
            steps {
                // Replace with your test runner. Examples:
                //   Python: sh 'pytest tests/unit/ --junitxml=reports/unit.xml'
                //   Node:   sh 'npm test -- --reporter junit'
                //   Java:   sh 'mvn test'
                sh '''
                    echo "Running unit tests..."
                    mkdir -p reports
                    # INSERT YOUR TEST COMMAND HERE
                    echo "Tests passed ✅"
                '''
            }
            post {
                always {
                    // Publish JUnit results if your test runner produces them
                    // junit 'reports/*.xml'
                    echo "Test results published"
                }
            }
        }

        // ── Stage 3: Build Image ──────────────────────────────────────────────
        // Runs on: branch, PR, main
        stage('Build Image') {
            steps {
                script {
                    echo "Building image: ${IMAGE_FULL}:${IMAGE_TAG}"

                    sh """
                        docker build \\
                          --label "git.commit=${SHORT_SHA}" \\
                          --label "git.branch=${env.BRANCH_NAME}" \\
                          --label "jenkins.build=${BUILD_NUMBER}" \\
                          --label "jenkins.url=${BUILD_URL}" \\
                          -t "${IMAGE_FULL}:${IMAGE_TAG}" \\
                          .
                    """

                    // Tag :latest only on main builds
                    if (env.BRANCH_NAME == 'main') {
                        sh "docker tag ${IMAGE_FULL}:${IMAGE_TAG} ${IMAGE_FULL}:latest"
                    }
                }
            }
        }

        // ── Stage 4: Image Scan ───────────────────────────────────────────────
        // Runs on: PR, main  (skip on short-lived branch builds for speed)
        stage('Image Scan') {
            when {
                anyOf {
                    expression { env.CHANGE_ID != null }    // PR
                    branch 'main'
                }
            }
            steps {
                script {
                    // Trivy: https://github.com/aquasecurity/trivy
                    // Fails the build if HIGH or CRITICAL CVEs are found.
                    sh """
                        trivy image \\
                          --severity ${TRIVY_SEVERITY} \\
                          --exit-code 1 \\
                          --no-progress \\
                          --format table \\
                          "${IMAGE_FULL}:${IMAGE_TAG}"
                    """
                }
            }
        }

        // ── Stage 5: Push Image ───────────────────────────────────────────────
        // Branch: push branch tag (for preview use)
        // PR:     push pr tag
        // Main:   push sha-version tag + :latest + capture digest
        stage('Push Image') {
            steps {
                script {
                    sh "echo ${REGISTRY_CREDS_PSW} | docker login ${IMAGE_REGISTRY} -u ${REGISTRY_CREDS_USR} --password-stdin"

                    sh "docker push ${IMAGE_FULL}:${IMAGE_TAG}"

                    if (env.BRANCH_NAME == 'main') {
                        sh "docker push ${IMAGE_FULL}:latest"

                        // Capture the immutable digest — this is what ArgoCD will pin
                        IMAGE_DIGEST = sh(
                            script: "docker inspect --format='{{index .RepoDigests 0}}' ${IMAGE_FULL}:${IMAGE_TAG} | cut -d@ -f2",
                            returnStdout: true
                        ).trim()

                        echo "Image digest: ${IMAGE_DIGEST}"
                    }

                    sh "docker logout ${IMAGE_REGISTRY}"
                }
            }
        }

        // ── Stage 6: Deploy Preview (PR only) ────────────────────────────────
        // Spin up an ephemeral environment for integration testing.
        // On real infrastructure replace the docker run with a helm install
        // into a per-PR namespace.
        stage('Deploy Preview') {
            when {
                expression { env.CHANGE_ID != null }    // PR only
            }
            steps {
                script {
                    def previewName = "${SERVICE_NAME}-pr-${env.CHANGE_ID}"

                    echo "Deploying preview: ${previewName}"

                    sh """
                        docker rm -f ${previewName} 2>/dev/null || true
                        docker run -d \\
                          --name ${previewName} \\
                          --restart unless-stopped \\
                          -p 0:8080 \\
                          "${IMAGE_FULL}:${IMAGE_TAG}"

                        # Wait for the container to be healthy
                        sleep 5

                        # Resolve the dynamically assigned host port
                        HOST_PORT=\$(docker inspect ${previewName} \\
                          --format='{{(index (index .NetworkSettings.Ports "8080/tcp") 0).HostPort}}')
                        echo "Preview running on port \${HOST_PORT}"
                        echo "\${HOST_PORT}" > preview-port.txt
                    """
                }
            }
        }

        // ── Stage 7: Integration & Smoke Tests (PR only) ─────────────────────
        stage('Integration Tests') {
            when {
                expression { env.CHANGE_ID != null }    // PR only
            }
            steps {
                script {
                    def port = readFile('preview-port.txt').trim()

                    sh """
                        echo "Running smoke test against preview (port ${port})..."

                        # Health check
                        HTTP_STATUS=\$(curl -s -o /dev/null -w "%{http_code}" \\
                          --retry 5 --retry-delay 2 \\
                          http://localhost:${port}/health)

                        if [ "\$HTTP_STATUS" = "200" ]; then
                          echo "Health check PASSED ✅ (HTTP \$HTTP_STATUS)"
                        else
                          echo "Health check FAILED ❌ (HTTP \$HTTP_STATUS)"
                          exit 1
                        fi

                        # INSERT YOUR INTEGRATION TEST SUITE HERE
                        # Examples:
                        #   pytest tests/integration/ -k "smoke"
                        #   newman run collection.json --env-var base_url=http://localhost:PORT
                    """
                }
            }
        }

        // ── Stage 8: Update Manifest (main only) ─────────────────────────────
        // After a successful main build, write the new image tag (by digest)
        // into the manifest repo's dev overlay. ArgoCD picks this up and
        // deploys automatically.
        stage('Update Manifest') {
            when {
                branch 'main'
            }
            steps {
                script {
                    echo "Updating manifest repo with digest: ${IMAGE_DIGEST}"

                    withCredentials([string(credentialsId: 'manifest-repo-pat', variable: 'MANIFEST_TOKEN')]) {
                        sh """
                            # Clone the manifest repo
                            git clone --depth 1 \\
                              https://x-access-token:${MANIFEST_TOKEN}@${MANIFEST_REPO.replace('https://', '')} \\
                              /tmp/manifest-repo

                            cd /tmp/manifest-repo

                            # Update the dev overlay image tag
                            # Using kustomize edit — adjust path to your overlay
                            cd apps/${SERVICE_NAME}/overlays/dev

                            kustomize edit set image \\
                              ${IMAGE_FULL}=${IMAGE_FULL}@${IMAGE_DIGEST}

                            # Commit and push
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

        // ── Stage 9: Write Deploy Info (main only) ────────────────────────────
        stage('Deploy Info') {
            when {
                branch 'main'
            }
            steps {
                script {
                    def deployInfo = """
{
  "image":      "${IMAGE_FULL}",
  "digest":     "${IMAGE_DIGEST}",
  "tags":       ["${IMAGE_TAG}", "latest"],
  "version":    "${IMAGE_TAG}",
  "commit_sha": "${SHORT_SHA}",
  "branch":     "${env.BRANCH_NAME}",
  "build":      "${BUILD_NUMBER}",
  "build_url":  "${BUILD_URL}",
  "built_at":   "\$(date -u +%Y-%m-%dT%H:%M:%SZ)"
}
"""
                    writeFile file: "deploy-info-${BUILD_NUMBER}.json", text: deployInfo
                    archiveArtifacts artifacts: "deploy-info-${BUILD_NUMBER}.json"
                    echo "Deploy info archived ✅"
                }
            }
        }

    }
    // ── End of stages ──────────────────────────────────────────────────────────

    // ── Post-build actions ─────────────────────────────────────────────────────
    post {

        always {
            script {
                // Tear down the PR preview container
                if (env.CHANGE_ID) {
                    def previewName = "${SERVICE_NAME}-pr-${env.CHANGE_ID}"
                    sh "docker rm -f ${previewName} 2>/dev/null || true"
                    echo "Preview environment torn down ✅"
                }

                // Clean up local images to avoid disk bloat on the agent
                sh "docker rmi ${IMAGE_FULL}:${IMAGE_TAG} 2>/dev/null || true"
                sh "docker rmi ${IMAGE_FULL}:latest 2>/dev/null || true"
            }

            // Clean workspace after build
            cleanWs()
        }

        success {
            echo "✅ Build ${BUILD_NUMBER} succeeded on ${env.BRANCH_NAME}"
            // Add Slack/Teams notification here if needed:
            // slackSend channel: '#ci-alerts', color: 'good',
            //   message: "✅ ${JOB_NAME} #${BUILD_NUMBER} passed — ${BUILD_URL}"
        }

        failure {
            echo "❌ Build ${BUILD_NUMBER} FAILED on ${env.BRANCH_NAME}"
            // slackSend channel: '#ci-alerts', color: 'danger',
            //   message: "❌ ${JOB_NAME} #${BUILD_NUMBER} FAILED — ${BUILD_URL}"
        }

    }

}
```

---

## 7. Branch Protection Rules

Configure these on the `main` branch of your **app repo**.

### GitHub — Classic branch protection rule

```
Pattern: main
────────────────────────────────────────────────────────────────────

✅ Require a pull request before merging
   ✅ Require approvals: 1
   ✅ Dismiss stale pull request approvals when new commits are pushed
   ✅ Require review from Code Owners (if CODEOWNERS file exists)

✅ Require status checks to pass before merging
   ✅ Require branches to be up to date before merging
   Required status checks (add these — exact names must match what Jenkins posts):
     ci/lint
     ci/unit-tests
     ci/image-scan
     ci/integration-tests
     ci/smoke-test

✅ Require linear history  (squash or rebase merges only)

✅ Do not allow bypassing the above settings
   (applies to admins too — prevents emergency direct pushes)

────────────────────────────────────────────────────────────────────
Optional for regulated environments:
  ✅ Require signed commits
  ✅ Restrict who can push to matching branches
     → add only the CI service account and senior engineers
```

### Manifest Repo — same protection on `main`

The manifest repo's `main` branch should have equivalent protection. The Jenkins service account needs write access to push manifest updates, but human direct-push should be blocked.

```
Pattern: main (manifest repo)
────────────────────────────────────────────────────────────────────

✅ Require a pull request before merging
   (Exception: Jenkins service account can push directly for automated
    manifest updates — configure this via a bypass list or a dedicated
    manifests/auto-update branch that PRs into main.)

✅ Require status checks:
     manifest/validate    (Helm lint or kustomize build)
     manifest/dry-run     (kubectl apply --dry-run=server)
```

---

## 8. ArgoCD Application — Setup & Config

ArgoCD watches the manifest repo and syncs the cluster to whatever is declared there. Jenkins only writes to the manifest repo — it never touches the cluster directly.

### 8.1 Install ArgoCD

```bash
kubectl create namespace argocd

kubectl apply -n argocd -f \
  https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for pods to be ready
kubectl wait --for=condition=available deployment/argocd-server \
  -n argocd --timeout=120s

# Expose the UI (for local access — use Ingress in production)
kubectl port-forward svc/argocd-server -n argocd 8443:443
```

### 8.2 Connect ArgoCD to the Manifest Repo

```bash
# Get initial admin password
argocd admin initial-password -n argocd

# Log in
argocd login localhost:8443 --username admin --password <password> --insecure

# Add the manifest repo (use deploy key for private repo)
argocd repo add https://github.com/<org>/<manifest-repo>.git \
  --username <service-account> \
  --password <pat-or-deploy-key>
```

### 8.3 ArgoCD Application Manifests

Store these in `manifest-repo/argocd/`. Apply them with `kubectl apply -f`.

**Dev Application (`app-dev.yaml`):**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: <service-name>-dev
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io  # cascade delete on app removal
spec:
  project: default

  source:
    repoURL: https://github.com/<org>/<manifest-repo>.git
    targetRevision: main
    path: apps/<service-name>/overlays/dev

  destination:
    server: https://kubernetes.default.svc
    namespace: <service-name>-dev

  syncPolicy:
    automated:
      prune: true      # remove resources deleted from manifests
      selfHeal: true   # revert manual cluster changes
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
      - ApplyOutOfSyncOnly=true
    retry:
      limit: 3
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m

  revisionHistoryLimit: 10
```

**Stage Application (`app-stage.yaml`):**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: <service-name>-stage
  namespace: argocd
spec:
  project: default

  source:
    repoURL: https://github.com/<org>/<manifest-repo>.git
    targetRevision: main
    path: apps/<service-name>/overlays/stage

  destination:
    server: https://kubernetes.default.svc
    namespace: <service-name>-stage

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
    retry:
      limit: 2
      backoff:
        duration: 10s
        factor: 2
        maxDuration: 5m

  revisionHistoryLimit: 10
```

**Prod Application (`app-prod.yaml`) — manual sync, no auto:**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: <service-name>-prod
  namespace: argocd
spec:
  project: default

  source:
    repoURL: https://github.com/<org>/<manifest-repo>.git
    targetRevision: main
    path: apps/<service-name>/overlays/prod

  destination:
    server: https://kubernetes.default.svc
    namespace: <service-name>-prod

  # No automated sync for production.
  # A human (or a promotion pipeline with approval) triggers sync:
  #   argocd app sync <service-name>-prod
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
    retry:
      limit: 1
      backoff:
        duration: 30s
        factor: 2
        maxDuration: 5m

  revisionHistoryLimit: 20
```

### 8.4 Kustomize Overlay Structure

**Dev overlay (`apps/<service-name>/overlays/dev/kustomization.yaml`):**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: <service-name>-dev

resources:
  - ../../base

# Jenkins updates this image line automatically via:
#   kustomize edit set image <image>=<image>@sha256:<digest>
images:
  - name: <registry>/<repo>/<service-name>
    newTag: <sha>-<version>    # e.g. 9f3c2e7-v1.4.3

# Dev-specific config
patchesStrategicMerge:
  - replica-patch.yaml         # replicas: 1 for dev
```

**Prod overlay (`apps/<service-name>/overlays/prod/kustomization.yaml`):**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: <service-name>-prod

resources:
  - ../../base

images:
  - name: <registry>/<repo>/<service-name>
    newTag: <sha>-<version>    # same tag as stage — promoted, not rebuilt

patchesStrategicMerge:
  - replica-patch.yaml         # replicas: 3+ for prod
  - resources-patch.yaml       # higher CPU/memory limits
```

---

## 9. Manifest Repository & Auto-Update

### How Jenkins updates the manifest

The `Update Manifest` stage in the Jenkinsfile does this automatically after every successful main build:

```bash
# 1. Clone manifest repo
git clone --depth 1 \
  https://x-access-token:<PAT>@github.com/<org>/<manifest-repo>.git \
  /tmp/manifest-repo

cd /tmp/manifest-repo/apps/<service-name>/overlays/dev

# 2. Update the image to the new digest (immutable reference)
kustomize edit set image \
  <registry>/<repo>/<service-name>=<registry>/<repo>/<service-name>@sha256:<digest>

# 3. Commit and push
git config user.email "jenkins@ci.example.com"
git config user.name  "Jenkins CI"
git add kustomization.yaml
git commit -m "ci(<service>): update dev image to <tag> [<sha>]"
git push origin main

# 4. ArgoCD detects the diff in ~3 minutes (or immediately via webhook)
#    and syncs the cluster
```

### ArgoCD webhook (optional but recommended)

Configure a webhook from the manifest repo to ArgoCD so it syncs in seconds, not on a polling interval:

```
Manifest repo → Settings → Webhooks → Add webhook
  Payload URL:  https://<argocd-server>/api/webhook
  Content type: application/json
  Secret:       <argocd-webhook-secret>
  Events:       Push events only
```

---

## 10. Secrets Management

Never store credentials in the Jenkinsfile or in any Git repo. All secrets live in Jenkins Credentials and are injected at runtime.

### Required Jenkins credentials

| Credential ID | Kind | What it is |
|---|---|---|
| `registry-credentials` | Username + password | Container registry login (Docker Hub / GHCR / ECR) |
| `github-app` or `github-pat` | GitHub App or Username+password | Checkout from private app repo |
| `manifest-repo-pat` | Secret text | PAT with `contents: write` on the manifest repo |
| `webhook-secret` | Secret text | Shared secret for GitHub → Jenkins webhook validation |
| `trivy-db-token` | Secret text (optional) | If using a private Trivy DB mirror |

### Adding credentials in Jenkins

```
Manage Jenkins → Credentials → System
  → Global credentials (unrestricted)
    → Add Credentials

For registry login:
  Kind:        Username with password
  Scope:       Global
  Username:    <registry-username>
  Password:    <registry-token>   (use a token, not a password)
  ID:          registry-credentials
  Description: Container registry push credentials

For manifest PAT:
  Kind:        Secret text
  Scope:       Global
  Secret:      <github-pat>
  ID:          manifest-repo-pat
  Description: GitHub PAT for manifest repo write access
```

### Using credentials in the Jenkinsfile

```groovy
// Username + password → creates _USR and _PSW env vars (masked in logs)
environment {
    REGISTRY_CREDS = credentials('registry-credentials')
}
// Available as: REGISTRY_CREDS_USR, REGISTRY_CREDS_PSW

// Secret text → single variable
withCredentials([string(credentialsId: 'manifest-repo-pat', variable: 'TOKEN')]) {
    sh "git clone https://x-access-token:${TOKEN}@..."
}
```

---

## 11. Cleanup — Branches, Jobs, Images

### Branch cleanup (GitHub)

Enable in repository settings:

```
GitHub repo → Settings → General → Pull Requests
  ✅ Automatically delete head branches
     (deletes the feature branch after PR merge)
```

### Jenkins MBP orphan cleanup

Already configured in Section 5.6. When a branch or PR is deleted, MBP marks the child job as orphaned and removes it after the configured retention period (7 days / 20 jobs).

To check orphaned jobs:
```
MBP job → Manage → Orphaned Item Strategy
```

### Preview environment cleanup

The Jenkinsfile `post { always { ... } }` block tears down the preview container after every PR build — pass or fail. For Kubernetes-based previews, add a `helm uninstall` or `kubectl delete namespace` there.

For TTL-based safety net cleanup (catches previews that were orphaned by a failed post-step):

```groovy
// Add this as a separate nightly Jenkins job
stage('Cleanup Stale Previews') {
    steps {
        sh '''
            docker ps --filter "name=<service>-pr-" \
              --format "{{.Names}} {{.RunningFor}}" | \\
            awk '$2 > 4 {print $1}' | \\   # older than 4 hours
            xargs -r docker rm -f
        '''
    }
}
```

### Container registry cleanup

Configure a retention policy in your registry:

```
Docker Hub:
  Repo → Settings → Image Retention Policies
  Rule: delete untagged images older than 7 days
  Rule: keep only the last 20 tagged images with prefix "pr-"
  Rule: never delete images tagged with a semantic version (v*)

GHCR (GitHub Actions):
  Org Settings → Packages → Container → Retention policy
  (or use: ghcr.io cleanup GitHub Action)

ECR:
  ECR repo → Lifecycle Policies → Add rule
  Priority 1: filter pr-* tags older than 7 days → expire
  Priority 2: keep last 30 tagged images
  Priority 3: expire untagged images older than 1 day
```

---

## 12. Quick Reference Cheatsheet

### Pipeline type matrix

```
┌─────────────────┬──────────────────────────────────────────────────────────┐
│  Pipeline type  │  What runs                                               │
├─────────────────┼──────────────────────────────────────────────────────────┤
│  Branch job     │  lint → unit tests → build image → push branch tag       │
│  (feature/*)    │  Target: ≤ 5 min. No deploy. No scan.                    │
│                 │  Image tag: <branch-slug>-<build#>                       │
├─────────────────┼──────────────────────────────────────────────────────────┤
│  PR job         │  lint → unit tests → build → scan → push pr tag          │
│  (PR-N)         │  → deploy preview → integration tests → post checks      │
│                 │  Image tag: pr-<N>-<sha7>                                │
├─────────────────┼──────────────────────────────────────────────────────────┤
│  Main job       │  lint → unit tests → build once → scan → push sha+ver   │
│                 │  → push :latest → capture digest → update manifest       │
│                 │  → ArgoCD syncs dev cluster                              │
│                 │  Image tag: <sha7>-<version>                             │
└─────────────────┴──────────────────────────────────────────────────────────┘
```

### Image tag reference

```
Branch job:   <branch-slug>-<build#>    feature-login-fix-7
PR job:       pr-<N>-<sha7>             pr-42-9f3c2e7
Main (primary): <sha7>-<version>        9f3c2e7-v1.4.3
Main (semver):  <version>               v1.4.3
Main (latest):  latest                  latest         ← never deploy this
Deploy by:      sha256:<digest>         sha256:abc123...
```

### Jenkins credentials required

```
registry-credentials    Username+password   push/pull from container registry
github-app              GitHub App          checkout private app repo
manifest-repo-pat       Secret text         write to manifest repo
webhook-secret          Secret text         validate GitHub → Jenkins webhooks
```

### ArgoCD sync modes

```
Dev:    automated  (prune=true, selfHeal=true)  → syncs on every manifest push
Stage:  automated  (prune=true, selfHeal=true)  → syncs on promotion PR merge
Prod:   manual     → human or approval pipeline runs: argocd app sync <app>
```

### Rollback procedure

```
1. Find last known-good digest in deploy-info artifact or manifest repo history

2. Update the manifest overlay to the previous digest:
   cd apps/<service>/overlays/<env>
   kustomize edit set image <image>=<image>@sha256:<prev-digest>
   git commit -m "rollback: revert <service> to <prev-tag>"
   git push

3. ArgoCD detects the change and syncs (auto for dev/stage, manual for prod)

4. Or for immediate prod rollback:
   argocd app set <service>-prod \
     --kustomize-image <image>=<image>@sha256:<prev-digest>
   argocd app sync <service>-prod
```

### The 5 invariants of this pipeline

```
1. Build once.  The same image (same digest) flows from PR → dev → stage → prod.
               Never rebuild for an environment.

2. Deploy by digest.  Tags are mutable and can be overwritten.
                      sha256:<digest> is immutable. Always pin to it.

3. Manifest repo is the source of truth.  Jenkins writes to it.
                                          ArgoCD reads from it.
                                          Nobody deploys directly to the cluster.

4. main is always releasable.  Branch protection + required checks enforce this.
                               If main is red, fix it before anything else.

5. Every deployment is auditable.  The deploy-info artifact, the manifest
                                   git history, and the Jenkins build record
                                   together tell you: what is running, where,
                                   since when, and from which commit.
```

---

*Maintained by the platform team. Open a PR to propose changes — the same CI rules apply.*