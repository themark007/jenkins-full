# Jenkins Multi-Branch Pipelines: Trunk-Based CI/CD in Practice

## What Is a Multi-Branch Pipeline?

A **Multi-Branch Pipeline (MBP)** is a Jenkins job type that scans your source control repository and automatically creates a child pipeline for every branch, pull request, or tag that contains a `Jenkinsfile`. There is no manual job creation — Jenkins discovers, builds, and retires jobs as your branches come and go.

This is the natural CI model for trunk-based development: `main` is the single source of truth, feature branches are short-lived, and every PR is automatically tested before it can merge.

---

## Why MBP Over a Single Pipeline With Many Branches?

| Concern | Single Pipeline | Multi-Branch Pipeline |
|---|---|---|
| Branch discovery | Manual | Automatic |
| PR awareness | None | Head and/or Merge builds |
| Build history | Shared | Per-branch / per-PR |
| Job lifecycle | Manual cleanup | Auto-created, auto-retired |
| Quality gates | Hard to enforce | Branch protection + required checks |

**Key idea:** You never list branches. MBP finds them and runs the pipeline wherever a `Jenkinsfile` exists.

---

## How Jenkins Discovers and Builds Branches

1. **Scan or webhook** — Jenkins scans on schedule and on webhook events (push, PR open/sync).
2. **Filters and traits** — Include/exclude rules, branch build strategies, PR strategies, and fork-trust policies decide what gets built.
3. **Per-branch or per-PR job** — Jenkins reads the `Jenkinsfile` for that ref and runs the pipeline.
4. **Status reporting** — Build status is posted back to the Git host as a commit/PR check. Branch protection can require green checks before merge.

---

## Trunk-Based CI/CD: The Full MBP Flow

This is the end-to-end lifecycle — from creating a branch to cleaning up after a merge.

### 1. Create a Feature Branch

Branch off `main`. Keep branches short-lived and consistently named (e.g. `feature/*`, `bugfix/*`) so that CI filters behave predictably.

> **Good practice:** Rebase or merge with `main` frequently. Enable *Require branch to be up to date before merging* in branch protection to enforce this.

---

### 2. Push → Branch Job Runs (Target: ≤ 5 Minutes)

**Trigger:** Every push to the feature branch (webhook preferred; scheduled scan as a backup).

With the *Exclude branches that are also filed as PRs* strategy, the branch job runs on every push until a PR is opened — then the PR job takes over.

**What runs:**
- Lint and format checks
- Compile / build
- Unit tests and coverage gate
- Package and build the container image (don't run it yet)
- Publish artifacts and metadata (image digest, build info)

**Why:** Fast feedback for obvious breakages before you even open a PR. Cache dependencies and use `disableConcurrentBuilds()` + `milestone()` so the newest push wins on concurrent builds.

---

### 3. Open a PR → PR Job Runs (Integration Build)

**Trigger:** PR open and each subsequent push to the PR branch.

Prefer the *Head + Merge* strategy: build the PR tip (Head) and a synthetic merge of your branch + current `main` (Merge). This tells you both whether your code alone works, and whether it will still work after landing on `main`.

**What runs (shift-left order):**
1. Light SAST and dependency scan
2. Build image → image scan (block on High/Critical findings)
3. Deploy an ephemeral preview environment (per-PR namespace or stack)
4. Integration and end-to-end smoke tests against the running app
5. Teardown of the preview on PR close or via a TTL safety net

**Why:** Unit tests cannot prove your change runs and integrates correctly. The PR job does that, before the merge.

> **Forks and secrets:** Use trust policies to limit which PRs can access secrets. Run reduced checks for untrusted forks.

---

### 4. Gate the Merge on Green Checks

Configure branch protection on `main` to require:
- All status checks green (e.g. *Tests*, *Coverage ≥ 80%*, *Deploy preview: succeeded*)
- Branch must be up to date with `main`
- At least one approving review

If any check fails: fix the code → push → the PR job re-runs automatically. Only when everything is green can the PR be merged. Optionally enable auto-merge or a merge queue.

---

### 5. Merge to Main → Promote the Artifact

**Trigger:** The merge commit lands on `main`; MBP runs the `main` job.

**What runs:**
- Reuse the image built and scanned in the PR job (preferred — promotes the same digest, eliminates build drift)
- Deploy to dev/staging, run smoke checks
- Promote onward with approvals or change control as needed
- Run full authenticated DAST here, not in the PR job (block on High/Critical)

**Why reuse the artifact:** Promoting the same image digest from PR → staging → production improves auditability and removes "works on my build" drift.

---

### 6. Cleanup

- Git host auto-deletes the source branch after merge (configure under repo settings)
- MBP's orphan-cleanup policy retires the branch/PR job and removes its workspace
- A preview-environment TTL or garbage collector reaps stale namespaces and volumes

---

## Demo Walkthrough

### Prerequisites

- A private Git repository containing your application code, a `Dockerfile`, and a `Jenkinsfile`
- Jenkins credentials (PAT or GitHub App) configured
- *(Recommended)* A webhook configured on the repository for immediate build triggers

---

### Step 0 — Create the MBP Job

1. **New Item** → give it a name → choose **Multibranch Pipeline**
2. **Branch Sources** → **Add source** → **GitHub** (or the plugin matching your Git host)

> Choosing **GitHub** (vs plain Git) gives you PR awareness, auto-webhook registration, and commit status checks back to the repo. Equivalent plugins exist for GitLab, Bitbucket, and Gitea.

3. **Credentials** — select your configured PAT or GitHub App
4. **Repository URL** — paste your repository URL

---

### Branch Build Strategy

Choose **one** of these for how branch jobs behave:

| Strategy | Behavior |
|---|---|
| **Exclude branches that are also filed as PRs** *(recommended)* | Branch job builds on push until a PR opens; then only the PR job runs. Branch job resumes if the PR closes. |
| **Only branches that are also filed as PRs** | Branch job only runs if a PR is open for that branch. Long-lived branches like `main` won't build here unless included separately. |
| **All branches** | Every branch builds on every push, even when a PR also exists. Likely causes duplicate builds. |

---

### PR Discovery Strategy (Origin PRs)

Choose **one or both** of these for how PR jobs build:

| Strategy | What It Builds | What It Tells You |
|---|---|---|
| **Head** | PR tip as-is, no merge with `main` | Does your code alone build and pass? |
| **Merge** | Synthetic merge of PR + current `main` | Would the post-merge build succeed? Catches integration drift. |
| **Both** *(recommended)* | Two jobs per PR event | Full signal: PR-only regressions AND after-merge outcome |

> **Rule of thumb:** If you must choose one, pick **Merge** — it reflects what `main` will look like after the PR lands. Require the Merge status check in branch protection.

---

### Fork PR Strategy

| Option | Use Case |
|---|---|
| **Head only** | Safest default. Run the fork's code without exposing secrets. |
| **Merge** | Internal forks where secrets are permitted and integration matters. |
| **Both** | High-trust forks only — use with care. |

**Trust policy:** Set to *Admin or Write* so external contributors get reduced checks (no secrets), while team members with write access get the full pipeline.

---

### Quick Reference: Recommended Settings

```
Branch strategy:     Exclude branches that are also filed as PRs
Origin PRs:          Both (Head + Merge) — require Merge status as a check
Fork PRs:            Head only + Trust "Admin or Write" for full runs
```

Click **Save**. Jenkins immediately scans the repo, discovers branches that contain a `Jenkinsfile`, and creates child jobs. Check **Scan Repository Log** to see what was discovered and what was skipped.

---

### Step 1 — Create a Feature Branch and Push

```bash
# Clone the repository
git clone --origin <remote-name> https://<YOUR-PAT>@github.com/<YOUR-ORG>/<YOUR-REPO>.git
cd <YOUR-REPO>

git remote -v
git branch -l

# Create and switch to a new feature branch
git switch -c feature/<your-feature>

# Make your change
# Edit the relevant source file, then:
git status
git add <changed-file>
git commit -m "feature(<scope>): <short description>"

# Verify main is unchanged
git switch main
cat <changed-file>   # main still has the old content

# Push the feature branch
git push <remote-name> feature/<your-feature>
```

Your repository now has at least two branches: `main` (your releasable trunk) and `feature/<your-feature>` (work in progress).

---

### Step 2 — Let MBP Discover the Branch

If webhooks are configured, Jenkins picks up `feature/<your-feature>` automatically. If not:

1. Open the MBP job in Jenkins
2. Click **Scan Repository Now**

You should now see two child jobs: `main` and `feature/<your-feature>`.

Because of the *Exclude branches that are also filed as PRs* strategy, the branch job builds on every push until you open a PR.

---

### Step 3 — Open a PR → PR Job Runs

1. Go to your repository on GitHub (or your Git host)
2. Click **Compare & pull request** (or **New pull request**)
3. Set **base: main**, **compare: feature/\<your-feature\>**
4. Add a title and description → **Create pull request**
5. *(Optional)* Enable **Automatically delete head branches** under repo Settings → General → Pull Requests

Back in Jenkins:
- If no webhook: click **Scan Repository Now** in the MBP job
- A new **PR-N** child job appears
- Open it → **Build Now** (or it starts automatically)

The PR job runs your full `Jenkinsfile`: builds and pushes the container image, deploys it, archives deployment metadata, and posts status back to the PR in GitHub.

---

### Step 4 — Gate the Merge on Success

Go to **Settings → Branches → Add rule** (or Add ruleset) in your repository.

Set **Branch name pattern** to `main` and enable:

- **Require a pull request before merging** — blocks direct pushes to `main`
- **Require status checks to pass** — add the Jenkins PR check name(s)
- **Require branches to be up to date** — forces rebase/merge with latest `main` before merging
- **Require approvals** — at least one reviewer; dismiss stale reviews on new commits

> **Note:** Some protections may not be enforced on personal private repos without a GitHub Team or Enterprise plan.

Once the Jenkins check on the PR shows green, the PR is ready to merge.

---

### Step 5 — Merge to Main → Run the Main Job

1. Click **Merge** on the PR in GitHub. If auto-delete is enabled, the feature branch is removed immediately.
2. In Jenkins MBP, click **Scan Repository Now** so the `main` job picks up the new commit.
3. The `main` job runs: build → push → deploy → archive → smoke test.

**Verify the deployment:**

```bash
# Check the container is running
docker ps
# Expect a container exposing your application port

# Confirm the app responds
curl -s http://localhost:<PORT>
# Expect the updated response from your change

# Inspect the archived deployment artifact in Jenkins:
# MBP job → main → Build #N → Artifacts → deploy-info-N.txt
```

**Confirm the code landed on main:**
Open `main` on GitHub and verify your changed file contains the new content.

---

### Step 6 — Cleanup

| What | How |
|---|---|
| Feature branch | Auto-deleted by Git host after merge (if configured) |
| Branch/PR jobs | MBP orphan-cleanup policy retires jobs and removes workspaces |
| Preview environments | TTL or GC job reaps stale namespaces, volumes, and databases |

---
