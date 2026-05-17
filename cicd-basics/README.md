# 🚀 CI/CD Basics — A DevOps Engineer's Field Guide

> **Who this is for:** Junior engineers and anyone new to CI/CD, Jenkins, and branching strategies.  
> **Maintained by:** A DevOps engineer documenting real-world patterns for the team.

---

## 📖 Table of Contents

1. [What is CI/CD?](#1-what-is-cicd)
2. [The CI/CD Glossary](#2-the-cicd-glossary)
3. [Branch Protection — Why It Matters](#3-branch-protection--why-it-matters)
4. [Branching Strategies](#4-branching-strategies)
5. [GitFlow-Based CI/CD](#5-gitflow-based-cicd)
6. [Trunk-Based CI/CD](#6-trunk-based-cicd)
7. [Build Once, Promote Many](#7-build-once-promote-many)
8. [The Promotion PR Flow](#8-the-promotion-pr-flow)
9. [Stage vs Production — Key Differences](#9-stage-vs-production--key-differences)
10. [Jenkins — The Orchestrator](#10-jenkins--the-orchestrator)
11. [Quick Reference Cheatsheet](#11-quick-reference-cheatsheet)

---

## 1. What is CI/CD?

CI/CD stands for **Continuous Integration / Continuous Delivery (or Deployment)**. It is the practice of automatically building, testing, and releasing software every time a developer pushes code.

Think of it like a factory assembly line:

```
Developer writes code
        ↓
Code is automatically tested (CI)
        ↓
Tested code is automatically shipped to environments (CD)
        ↓
Users get working software faster and safely
```

**Why do teams use it?**

| Without CI/CD | With CI/CD |
|---|---|
| Bugs caught late (in production) | Bugs caught early (in PR/pipeline) |
| Manual, error-prone deployments | Automated, repeatable deployments |
| "It works on my machine" problems | Consistent build in a clean environment |
| Long release cycles (weeks/months) | Short, frequent releases (days/hours) |
| No audit trail | Full history of what was deployed and when |

---

## 2. The CI/CD Glossary

These terms are used everywhere. Learn them once and they will make sense across all tools (Jenkins, GitHub Actions, GitLab CI, etc.).

| Term | Full Form | What it means in plain English |
|---|---|---|
| **CI** | Continuous Integration | Automatically build and test code every time someone pushes a change. Catches bugs before they land in the main branch. |
| **CT** | Continuous Testing | Running automated tests continuously — unit tests, integration tests, e2e tests — as part of every pipeline run. |
| **CD** | Continuous Delivery | Automatically deliver tested code to an environment (like staging), but a **human approves** before it goes to production. |
| **CDp** | Continuous Deployment | Like CD, but **no human approval** needed — code goes all the way to production automatically if tests pass. |
| **CM** | Configuration Management | Managing and versioning the config of your infrastructure and apps (e.g., Helm values, Terraform files, Ansible playbooks). |

### How they connect

```
  Push code
      │
      ▼
 ┌─────────┐     ┌─────────┐     ┌─────────────────┐     ┌─────────────────────┐
 │   CI    │────▶│   CT    │────▶│  CD (approval)  │────▶│  CDp (auto to prod) │
 │ Build + │     │  Tests  │     │  Human gates    │     │  Fully automated    │
 │  Lint   │     │  pass   │     │  each env       │     │  (needs mature team)│
 └─────────┘     └─────────┘     └─────────────────┘     └─────────────────────┘
```

> 💡 **Most teams use CD (not CDp) for production.** A human still approves before prod. CDp works well for microservices teams with very strong test coverage and monitoring.

---

## 3. Branch Protection — Why It Matters

A **protected branch** is a Git branch where you cannot push code directly. All changes must come through a **Pull Request (PR)**, and that PR must pass certain checks before it can be merged.

### Why protect branches?

Without protection, anyone can:
- Force-push and overwrite history
- Break `main` / `dev` / `stage` for the whole team
- Bypass tests entirely

### What branch protection rules look like

```
Protected branch: main
──────────────────────────────────────────────────
✅ PRs only — no direct pushes allowed
✅ Must be up-to-date with base before merge
✅ Required status checks must pass:
      - build
      - unit-tests
      - lint
      - security-scan
✅ At least 1 reviewer must approve
✅ No force pushes
✅ No deletions
```

### The PR flow on a protected branch

```
You push to feature/my-change
          │
          ▼
    Open a Pull Request
          │
          ▼
   CI runs automatically ──── tests the merge result (not just your branch)
          │
    ┌─────┴─────┐
    │           │
  Green ✅    Red ❌
    │           │
  Merge       Fix your code,
  enabled     push again → CI reruns
```

> 💡 **Key insight:** CI tests the *merge result* (base branch + your branch combined), not just your branch alone. This is how it catches conflicts before they land.

---

## 4. Branching Strategies

There are two main strategies teams use. Both are valid — the right choice depends on your team size, release cadence, and risk tolerance.

```
┌───────────────────────────────────────────────────────────────────────────┐
│                    BRANCHING STRATEGY COMPARISON                          │
├────────────────────────┬──────────────────────────────────────────────────┤
│                        │                                                  │
│     GitFlow            │           Trunk-Based                            │
│  (env branches)        │         (single trunk)                           │
│                        │                                                  │
│  feature ──▶ dev       │   feature ──▶ main                               │
│  dev     ──▶ stage     │   main   ──▶ Dev env                             │
│  stage   ──▶ prod      │   main   ──▶ Stage env                           │
│                        │   main   ──▶ Prod env                            │
│  Long-lived branches   │   Short-lived feature branches only              │
│  mirror environments   │   Environments managed separately                │
│                        │                                                  │
├────────────────────────┼──────────────────────────────────────────────────┤
│ More branch merges     │ Fewer merges, faster feedback                    │
│ Risk of branch drift   │ Requires strong tests + feature flags            │
│ Great for regulated    │ Great for SaaS / microservices                   │
│ / enterprise setups    │ / high-frequency release teams                   │
│ Clear audit trail      │ Clean history on single trunk                    │
│ per environment        │                                                  │
└────────────────────────┴──────────────────────────────────────────────────┘
```

---

## 5. GitFlow-Based CI/CD

### The Model

Multiple **long-lived branches** directly mirror your environments:

```
feature/topup-loan
       │
       ▼  (PR)
      dev  ──────────────────────▶  Dev environment
       │
       ▼  (promotion PR)
     stage ─────────────────────▶  Stage environment
       │
       ▼  (promotion PR)
      prod  ─────────────────────▶  Production environment
```

Each branch represents a stage in the release pipeline. Code always flows **left to right** — never backward.

### Step-by-Step: Feature → Dev

```
Step 1: You write code on feature/topup-loan and push it to Git

Step 2: You open a PR from feature → dev
        (dev is protected; no direct pushes allowed)

Step 3: Opening the PR triggers CI automatically
        CI builds: feature + dev combined (a test merge)
        CI runs:   checkout → build → unit tests → lint → security scan
        Target time: ~10–15 minutes

Step 4: CI posts a result back to the PR
        ✅ Green → Merge button unlocks
        ❌ Red   → Merge blocked; push a fix, CI reruns

Step 5: PR is approved + CI is green → Merge into dev
        dev now points to a known-good commit

Step 6: Pipeline auto-deploys to Dev environment (Continuous Delivery)
        If this deploy fails → roll back dev; it does NOT block other PRs
```

### Flow diagram

```
feature/topup-loan
│
│  git push
│
▼
┌──────────────────┐
│  Open PR          │  ← PR to dev (protected branch)
│  feature → dev   │
└────────┬─────────┘
         │ triggers
         ▼
┌──────────────────────────────┐
│         CI Pipeline          │
│  1. Checkout (merge result)  │
│  2. Build / Compile          │
│  3. Unit Tests               │
│  4. Lint + Security Scan     │
│  Target: ~10-15 min          │
└────────┬─────────────────────┘
         │
    ┌────┴────┐
    │         │
  Green ✅  Red ❌
    │         │
    │         └──▶ Push fix → CI reruns
    │
    ▼
  Merge PR into dev
    │
    ▼
  Auto-deploy to Dev environment  ←  Continuous Delivery (CD)
```

---

## 6. Trunk-Based CI/CD

### The Model

There is **one long-lived branch: `main`** (the trunk). All feature branches are short-lived and merge directly into `main`. Environments are promoted separately — not via branch merges.

```
feature/topup-loan (short-lived)
       │
       ▼  (PR → main)
      main  ──▶  Build once → publish image + digest + manifest
                     │
                     ▼
                 Dev environment   (auto-deploy)
                     │
                     ▼
               Stage environment  (approval gate)
                     │
                     ▼
                Prod environment  (approval gate + canary rollout)
```

### Step-by-Step: Full Trunk Flow

```
Step 1: Work on feature/topup-loan, push to Git

Step 2: Open PR → main
        Branch protection requires passing checks

Step 3: PR CI runs (same as GitFlow)
        Checkout → Build → Tests → Lint → Security scan
        Tests run against: main + your feature (merge result)

Step 4: Green → Merge. Red → Fix and rerun.

Step 5: Merge PR into main
        main now points at a known-good merge commit

Step 6: Post-merge pipeline kicks off on main:
        - Build once → push image to container registry
        - Capture immutable digest  sha256:abc123...
        - Write manifest.json and publish it

Step 7: Dev — auto-deploy using the manifest (same digest, no rebuild)
        Smoke test passes → ready for promotion

Step 8: Stage — approval gate → deploy same digest with Stage config
        Run heavier tests (integration, e2e, performance smoke)

Step 9: Prod — approval gate → cautious canary rollout
        SLO guardrails monitor; auto-rollback if metrics degrade
```

### Full flow diagram

```
feature/topup-loan
│  push
▼
┌─────────────────────────────┐
│  Open PR → main             │
└──────────┬──────────────────┘
           │ triggers
           ▼
┌──────────────────────────────┐
│       PR CI Pipeline         │
│  Build + Tests + Lint + SAST │
│  (tests main + feature)      │
└──────────┬───────────────────┘
      Green ✅ / Red ❌
           │
           ▼ (merge)
         main  ←─ single trunk, protected
           │
           ▼ (post-merge pipeline)
┌──────────────────────────────────────────┐
│  Build Once & Publish                    │
│  ├── Build image                         │
│  ├── Push to registry                    │
│  ├── Capture digest  sha256:abc123...    │
│  └── Write + publish manifest.json       │
└──────────────┬───────────────────────────┘
               │
     ┌─────────▼─────────┐
     │   Dev (auto CD)   │
     │  Deploy by digest │
     │  Smoke test       │
     └─────────┬─────────┘
               │ approval
     ┌─────────▼──────────┐
     │  Stage             │
     │  Deploy by digest  │
     │  Heavy testing     │
     │  Integration + E2E │
     └─────────┬──────────┘
               │ approval
     ┌─────────▼──────────────────────┐
     │  Prod                          │
     │  Canary: 5%→20%→50%→100%      │
     │  SLO guardrails + rollback     │
     └────────────────────────────────┘
```

---

## 7. Build Once, Promote Many

> ⚠️ **The golden rule of CI/CD:** Build the artifact **once**. Promote the **same artifact** to every environment. Never rebuild.

### Why?

If you rebuild for each environment, you have no guarantee the binary that passed Stage tests is the same one you're shipping to Production. A rebuild can pull different dependency versions, apply different compiler flags, or fail silently.

```
❌ WRONG — Rebuilding per environment

  Dev build   ──▶  Dev env
  Stage build ──▶  Stage env    ← Different binary! Not what you tested.
  Prod build  ──▶  Prod env     ← Not what passed Stage tests!


✅ CORRECT — Build once, promote the same artifact

  Build once ──▶  Publish image to registry
                       │
               ┌───────┼───────┐
               ▼       ▼       ▼
             Dev     Stage    Prod
          (same digest)  (same digest)  (same digest)
```

### The manifest.json — source of truth

After building, CI writes a small `manifest.json` that acts as the **passport** for the artifact across environments.

```json
{
  "image": "ghcr.io/acme/banking-loans",
  "digest": "sha256:abc123...",
  "version": "v1.4.3",
  "commit_sha": "9f3c2e7",
  "tags": ["v1.4.3", "9f3c2e7", "9f3c2e7-v1.4.3"],
  "built_at": "2025-09-15T10:22:00Z"
}
```

| Field | Purpose |
|---|---|
| `image` | Which registry / repo |
| `digest` | The immutable fingerprint — **always deploy by this** |
| `version` | Human-readable release version |
| `commit_sha` | Trace back to exact source code |
| `tags` | Convenient labels for humans (dashboards, changelogs) |
| `built_at` | When the artifact was produced |

### Tags vs Digest — what's the difference?

```
Tag:    :v1.4.3           ← mutable, can be overwritten
Tag:    :9f3c2e7          ← mutable
Digest: sha256:abc123...  ← immutable, always this exact binary
```

> 💡 **Always deploy by digest.** Tags are for humans (dashboards, release notes). Digests are for pipelines and deployments.

### Why keep multiple tags?

| Tag format | Who uses it | Why |
|---|---|---|
| `:v1.4.3` | Product / QA / Support | Easy to map to changelog and incidents |
| `:9f3c2e7` | Engineers / SRE | Jump to exact source for debugging |
| `:9f3c2e7-v1.4.3` | Everyone | Best of both — version + commit at a glance |

---

## 8. The Promotion PR Flow

### What is a Promotion PR?

When promoting across environments (dev → stage, stage → prod), many teams open a **Promotion PR**. This is a PR that:

1. References the exact artifact digest to be promoted
2. Triggers the CI pipeline to **deploy during the PR** (not after merge)
3. Merges **only after** the environment is healthy

This keeps the branch history clean and auditable — every merge means "this artifact was validated in this environment."

### Deploy-Before-Merge vs Deploy-After-Merge

```
┌──────────────────────────────────────────────────────────────────┐
│                     Two Different Strategies                      │
├──────────────────────────────┬───────────────────────────────────┤
│  Feature → Dev               │  Dev → Stage / Stage → Prod       │
│  Deploy AFTER merge          │  Deploy DURING the PR             │
│  (speed)                     │  Merge AFTER deploy is healthy    │
│                              │  (safety)                         │
├──────────────────────────────┼───────────────────────────────────┤
│  Goal: fast integration      │  Goal: controlled promotion       │
│  Many PRs open at once       │  Only one promotion active        │
│  Dev deploy failure = fix    │  Failure = red PR, roll back,     │
│  fast, does NOT block others │  fix in dev, re-promote           │
└──────────────────────────────┴───────────────────────────────────┘
```

> 💡 **Analogy:** Feature → dev is like a daily standup — quick sync, integrate fast.  
> Dev → Stage/Prod is like a dress rehearsal — you run the whole show before you sign off.

### Stage Promotion PR flow

```
1. Open PR: dev → stage
   Include: image, digest, version, commit SHA, manifest link

2. CI runs the Promotion Pipeline on the PR:

   ┌─────────────────┐
   │   Preflight     │  ← fast fail (~seconds)
   │  - Fetch & validate manifest.json
   │  - Verify digest is pullable
   │  - Render Stage config (Helm dry-run)
   └────────┬────────┘
            │
   ┌─────────▼────────┐
   │    Deploy        │  ← no rebuild; deploy by digest
   │  helm upgrade    │
   │  --set image@digest
   └────────┬─────────┘
            │
   ┌─────────▼────────┐
   │  Post-Deploy     │  ← only what needs a live Stage
   │  - Smoke / health check
   │  - /version confirms right digest
   │  - 1-2 integration tests (happy path)
   └────────┬─────────┘
            │
     CI posts statuses to PR:
     ✅ preflight-manifest
     ✅ preflight-config
     ✅ deploy-to-stage
     ✅ stage-smoke
     ✅ stage-integration

3. All green + reviewer approved → Merge PR
   (No new deploy needed — Stage is already on that digest)

4. If anything fails:
   - Roll back Stage (helm rollback loans <rev>)
   - Keep PR red / unmerged
   - Fix in dev → produce new digest → re-promote
```

### Stage PR branch protection rules

```
Stage branch rules:
──────────────────────────────────────────────
✅ PRs only (block direct / force pushes)
✅ Must be up-to-date with base
✅ Dev build/publish green for this commit_sha
✅ Required status checks:
      preflight-manifest
      preflight-config
      deploy-to-stage
      stage-smoke
      stage-integration
✅ At least 1 reviewer (TL / QA / owner)
```

---

## 9. Stage vs Production — Key Differences

Stage and Production both use the same artifact (same digest), but they differ in **what they check** and **how strictly they gate**.

```
┌─────────────────────────────────────────────────────────────────────┐
│                   STAGE vs PRODUCTION PIPELINE                      │
├────────────────────────────┬────────────────────────────────────────┤
│         STAGE              │            PRODUCTION                  │
├────────────────────────────┼────────────────────────────────────────┤
│ PRE-DEPLOY:                │ PRE-DEPLOY (stricter):                 │
│  Lighter approval gates    │  More reviewers required               │
│  Standard config check     │  Change window / approval may apply    │
│                            │  Feature flags must default OFF        │
│                            │  Re-verify manifest + digest           │
├────────────────────────────┼────────────────────────────────────────┤
│ DEPLOY:                    │ DEPLOY:                                │
│  Same digest, Stage config │  Same digest, Prod config              │
│  Quick canary (10%→100%)   │  Careful canary: 5%→20%→50%→100%      │
│  (rehearse the mechanism)  │  or blue-green                         │
│                            │  Feature flags OFF during ramp-up      │
│                            │  SLO guardrails + auto-rollback        │
├────────────────────────────┼────────────────────────────────────────┤
│ POST-DEPLOY:               │ POST-DEPLOY (lighter):                 │
│  Heavy testing:            │  Don't re-run Stage's suites           │
│   Integration tests        │  SLO guardrails: error rate,           │
│   Contract tests           │    latency, saturation                 │
│   API / UI e2e (1-2 flows) │  Short synthetics (critical journeys)  │
│   Performance smoke        │  Brief dwell — wait for alerts         │
│   Security smoke           │                                        │
└────────────────────────────┴────────────────────────────────────────┘
```

### Canary rollout in Production

```
Prod deploy begins
       │
  ┌────▼────┐
  │   5%    │  ← small slice of traffic
  │ traffic │  Monitor SLOs: error rate, latency, saturation
  └────┬────┘
       │ SLOs healthy ✅
  ┌────▼────┐
  │  20%    │  Continue monitoring
  └────┬────┘
       │ SLOs healthy ✅
  ┌────▼────┐
  │  50%    │  Halfway — still watching
  └────┬────┘
       │ SLOs healthy ✅
  ┌────▼────┐
  │  100%   │  Full rollout complete
  └─────────┘

  If SLOs breach at ANY stage → auto-rollback to previous good digest
```

---

## 10. Jenkins — The Orchestrator

### What is Jenkins?

Jenkins is an open-source **automation server**. It listens for events (like a push to Git) and runs pipelines that build, test, and deploy your software. Think of Jenkins as the conductor of the CI/CD orchestra — it coordinates all the tools (Git, Docker, Helm, Vault, Slack) but doesn't replace any of them.

```
┌─────────────────────────────────────────────────────────────┐
│                        JENKINS                              │
│                                                             │
│  Git push / PR ──▶  Webhook  ──▶  Trigger pipeline         │
│                                                             │
│  Pipeline steps:                                            │
│    Checkout  →  Build  →  Test  →  Scan  →  Publish        │
│        →  Deploy  →  Post-deploy  →  Notify                │
│                                                             │
│  Posts status checks back to GitHub/GitLab                  │
│  Gates merge buttons (green/red)                            │
│                                                             │
│  Connects to:                                               │
│    Container Registry │ Vault (secrets) │ Helm/Kubernetes  │
│    Slack/Teams        │ JIRA            │ SonarQube        │
└─────────────────────────────────────────────────────────────┘
```

### Why teams use Jenkins

| Feature | What it means for you |
|---|---|
| **Pipeline as Code** | `Jenkinsfile` lives in your repo — versioned, reviewed, reproducible |
| **Portable** | Runs on a VM, Kubernetes, or even a laptop |
| **1,800+ plugins** | Connects to almost any tool in your stack |
| **Multi-branch pipelines** | Automatically creates a pipeline per branch/PR |
| **Ephemeral agents** | Spins up fresh build agents on Kubernetes — no stale state |
| **Credential management** | Integrates with Vault, AWS Secrets Manager, etc. |

### Jenkins fits both strategies

```
GitFlow setup:
  Multibranch pipeline per branch (dev, stage, prod)
  Feature → dev  : CI pipeline (build + test + lint)
  dev → stage    : Promotion pipeline (preflight + deploy + post-deploy)
  stage → prod   : Production gate pipeline (stricter pre-deploy + canary)

Trunk-based setup:
  PR → main      : CI pipeline (build + test + lint)
  Post-merge     : Build-once pipeline (build + publish + manifest.json)
  Dev            : Auto-deploy pipeline (reads manifest, deploys by digest)
  Stage          : Approval-gated pipeline (same digest, heavier tests)
  Prod           : Approval + canary pipeline (SLO guardrails)
```

### A minimal Jenkinsfile (PR CI)

```groovy
pipeline {
    agent { label 'docker-agent' }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Test') {
            steps {
                sh 'mvn test'
            }
            post {
                always {
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }

        stage('Lint & Security') {
            parallel {
                stage('Lint') {
                    steps { sh 'mvn checkstyle:check' }
                }
                stage('SAST') {
                    steps { sh 'trivy fs --exit-code 1 .' }
                }
            }
        }
    }

    post {
        success {
            githubSetCommitStatus context: 'ci/jenkins', state: 'success'
        }
        failure {
            githubSetCommitStatus context: 'ci/jenkins', state: 'failure'
        }
    }
}
```

### What Jenkins is NOT

- Not a Git server (use GitHub / GitLab / Bitbucket)
- Not a container registry (use GHCR / ECR / Nexus)
- Not a monitoring stack (use Prometheus / Grafana / Datadog)
- Not a secrets store (use HashiCorp Vault / AWS Secrets Manager)

Jenkins **calls** those systems — it doesn't replace them.

---

## 11. Quick Reference Cheatsheet

### CI/CD Terms at a Glance

```
CI  = Build + Test on every push/PR        → fast feedback, gate merges
CT  = Testing is continuous, part of CI    → unit, integration, e2e
CD  = Auto-promote artifact, human gates   → Dev/Stage auto, Prod approval
CDp = Auto-promote all the way to Prod     → no human approval, needs mature team
CM  = Config as code (Helm, Terraform)     → versioned, reviewed infra config
```

### The 5 Rules of Good CI/CD

```
1. Build once. Promote the same artifact (by digest) to every environment.
2. CI runs on every PR. Tests the merge result, not just your branch.
3. Protected branches everywhere. PRs only, required checks, required reviews.
4. Stage carries the heavy tests. Prod enforces the strict pre-deploy gates.
5. Always have a rollback plan. Know the last good digest. Be ready to use it.
```

### GitFlow vs Trunk-Based — When to use which

```
Use GitFlow when:                    Use Trunk-Based when:
─────────────────────                ──────────────────────────
Regulated / enterprise               SaaS / microservices
Release trains (monthly)             Frequent releases (daily)
Multiple teams, long QA              Small, autonomous teams
Need per-env branch history          Need fast feedback loops
Hard compliance audit trail          Feature flags are mature
```

### Promotion checklist (any environment)

```
Before promoting to Stage or Prod:
  □ Artifact built from a known, reviewed commit
  □ manifest.json published with correct digest
  □ Digest verified as pullable from registry
  □ Config rendered + dry-run validated (helm lint + dry-run)
  □ Feature flags default OFF
  □ Required reviewers have approved
  □ Change window met (if applicable, for Prod)
  □ Rollback plan confirmed (last good digest known)
```

---

## 📚 Further Reading

- [Jenkins Documentation](https://www.jenkins.io/doc/)
- [Helm Docs — values and templating](https://helm.sh/docs/)
- [Git branch protection rules (GitHub)](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches)
- [Trunk Based Development](https://trunkbaseddevelopment.com/)
- [GitFlow original post — Vincent Driessen](https://nvie.com/posts/a-successful-git-branching-model/)
- [The Twelve-Factor App](https://12factor.net/) — solid principles for CI/CD-friendly apps

---

## 🤝 Contributing

Found a mistake or want to add a topic? Open a PR!

- Keep language simple — write for a junior who is reading this on day 1
- Add a diagram if you're explaining a flow
- Test any code/script examples before committing

---

*Last updated: 2025 | Maintained by the DevOps team*
