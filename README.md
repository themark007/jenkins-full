protected brnached devsecops
waht is cicd ci ct cd cdp cm
branching strategies
gitflow based cicd : env brnached-->multiple long lived brnaches dev -> stg-> uat->prod  build one promote artifact
GitFlow-Based CI/CD: Environment Branches

Model: Multiple long-lived branches mirror environments (dev → stage → prod).
Flow: Devs merge feature branches into dev, then raise promotion PRs dev→stage and stage→prod.
CI: Runs on every PR (feature→dev and env→env) to gate merges with green/red checks.
Build: Build once when the change is approved into dev; publish image + digest.
Promotion: Use PRs dev→stage and stage→prod that deploy the same digest; CI on those PRs gates the merge.
Governance: Strong branch protection, required reviews/approvals, clear audit trail at each hop.
Trade-offs / Fit: More merges and coordination; risk of branch drift. Common in regulated/enterprise and release-train setups.
Trunk-Based CI/CD: Environment Promotions

Model: Single long-lived trunk (main) as source of truth; short-lived feature branches.
Flow: Small PRs → CI on PR into main (tests the merge result) → build once after merge → promote same artifact through Dev → Staging → Prod (no env-branch merges).
Build: Build once on merge to main; publish image + immutable digest.
Promotion: Update the same digest in Dev → Staging → Prod via pipeline stages or GitOps PRs in an env-config repo (no rebuild; config only).
Key point: Both strategies should “build once, promote many.” Trunk-based differs in branching (single main, no env branches in the app repo), not in the artifact policy.
Governance: Lean process with protected main, required checks; Prod uses approvals, progressive rollout, and rollback.
Trade-offs / Fit: Requires discipline (small PRs, solid tests); rewards with fewer merges and faster feedback, ideal for microservices/SaaS.

tep-by-step (numbers match the diagram)

Commit & push (feature branch) You work on feature/topup-loan, commit locally, and push to Git. This publishes your change so it can be reviewed and tested. Output: a feature branch with new commits.

Open PR to dev (protected) The developer manually opens a PR from feature → dev. Because dev is protected: no direct commits, PRs only, and passing status checks + required reviews. The dotted arrow means a PR proposes a merge; dev hasn’t changed yet.

Note: A feature branch contains only your feature work—not the full, up-to-date app. To validate your change against everything else, you must raise a PR so CI can test the merge result (dev + feature) before it lands. Output: an open PR.
CI Pipeline (Checkout • Build • Test) Opening the PR triggers CI. CI builds the merge result of dev + feature in a clean workspace (so dev itself stays unchanged), then runs quick scripted steps—checkout, build/compile, unit tests, and basic lint/security checks. Target runtime: ≈10–15 min. Output: a pass/fail result.

PR Status (green/red) CI posts status back to the PR:

Green → Merge enabled. All checks passed; the Merge button unlocks.
Red → Merge blocked. Something failed; you push a fix, CI reruns, and the status updates. Output: a decision to merge or iterate.
Merge PR into dev With green checks (and any required reviews), the PR merges. dev advances to a known-good merge SHA. Many teams tie the built artifact (image/JAR) to that SHA for traceability. Output: clean dev pointing at the merge commit.

Auto-deploy to Dev (CD) After the merge, the pipeline may automatically deploy dev’s new version to the Dev environment (Kubernetes/VM/Serverless) for quick smoke checks. This is Continuous Delivery (CD) because it’s an automated non-prod promotion/deploy.

Clarifier: If you automatically deploy to Production with no human approval, that’s Continuous Deployment (CDp). If Production still requires a manual approval/gate, it remains CD.
Output: the change running in Dev, ready for further testing.

Key idea: CI runs on the PR to test the merge result before anything lands in dev. Green enables merge; red blocks it until you push a fix. After the merge, you can optionally auto-deploy to the Dev environment.
CI runs on the PR to test the tentative merge (dev + feature). If it’s green, you merge. After the merge, a pipeline can optionally auto-deploy the updated dev to the Dev environment—that’s Continuous Delivery (CD). (Auto-deploying to Production without approval would be Continuous Deployment (CDp), which we’re not doing here.)
Promotion PR: dev → stage (Deploy-Before-Merge Flow)

This flow promotes a release candidate from dev to Stage. The promotion happens during the PR: CI does Preflight → Deploy → Post-deploy. The PR turns green only if Stage is healthy, and then you merge. That keeps the stage history clean and auditable.

What happened earlier on dev (input to promotion)

Build once & push (single image). CI builds one image and may attach human-friendly tags (e.g., :v1.4.3, :9f3c2e7, :9f3c2e7-v1.4.3). All of these tags point to the same immutable digest.

Record the immutable digest (source of truth). The registry returns a content address like sha256:abc123…; this never changes and is what you deploy.

Publish a promotion manifest. CI writes manifest.json (repo, digest, commit, version, built_at, optional tags) to a read-only location Stage/Prod can read (e.g., GitHub Release, CI artifact, S3).

Deploy by digest (not tags). Dev/Stage/Prod jobs read the manifest, verify the digest, and deploy that digest with env-specific config—no rebuilds (digest preferred even if tags are immutable).

Example manifest.json:

{
  "image": "ghcr.io/acme/banking-loans",
  "digest": "sha256:abc123...",
  "version": "v1.4.3",
  "commit_sha": "9f3c2e7",
  "tags": ["v1.4.3", "9f3c2e7", "9f3c2e7-v1.4.3"],
  "built_at": "2025-09-15T10:22:00Z"
}
Why keep multiple tags (even though you deploy by digest)?

:v1.4.3 (release): easy for Product/QA/Support to map incidents and read changelogs/dashboards.
:9f3c2e7 (commit): engineers/SREs can jump to the exact source for bisects/hotfixes.
:9f3c2e7-v1.4.3 (combined): best of both at a glance in registry UIs.
TL;DR: Build once, tag for humans, deploy by digest everywhere.
Stage PR gates (repo rules)

Configure these on the stage branch so a PR cannot merge until the right things are true:

PRs only (block direct/force pushes).
Up-to-date with base (PR includes the latest stage tip; use Update branch).
Recent Dev run green for the promoted commit_sha (status like dev-build-publish).
Require these statuses to pass (produced by the pipeline below): preflight-manifest, preflight-config, deploy-to-stage, stage-smoke, stage-integration.
Required review (≥1) (TL/QA/owner).
1 Open PR: dev → stage

Open a promotion PR that references the exact release candidate by its immutable digest (tags are optional, for humans). Stage CI will read the manifest and deploy by digest—no rebuilds.

Good PR description template:

Promote to Stage

image: ghcr.io/acme/banking-loans
digest: sha256:abc123…          # deployments use this (immutable)
version: v1.4.3                 # optional, human-friendly
commit_sha: 9f3c2e7             # source traceability
tags: [v1.4.3, 9f3c2e7, 9f3c2e7-v1.4.3]  # optional, for discoverability
manifest: <read-only link>      # where Stage reads image+digest
Output: PR is open; the stage branch remains unchanged. (Promotion pipeline will validate and deploy by digest during this PR; merge only when all checks are green.)

2 Stage Promotion Pipeline (runs on the PR)

2.1 Preflight (fast fail in seconds) CI proves the release can be deployed before touching Stage:

Fetch manifest.json (S3 / repo / PR attachment) and validate schema (image, digest, commit).
Resolve & verify digest is pullable (repo@sha256:…).
Render & validate Stage config: helm lint → helm template | kubectl apply --dry-run=server -f -.
2.2 Deploy (no rebuild; by digest with Stage config only)

IMAGE=$(jq -r '.image'  manifest.json)
DIGEST=$(jq -r '.digest' manifest.json)

helm upgrade --install loans ./charts/loans \
  --set image.repository="$IMAGE" \
  --set image.digest="$DIGEST" \
  -f charts/loans/values-stage.yaml
# Chart should build: image: {{ .Values.image.repository }}@{{ .Values.image.digest }}
Deployed image: ghcr.io/acme/banking-loans@sha256:abc123… (immutable; not a tag)

2.3 Post-deploy (only what needs a live Stage)

Smoke/health: readiness/liveness; /version shows v1.4.3 (9f3c2e7).
Small integration/e2e: 1–2 happy paths (e.g., create loan → fetch).
Output: CI posts the required commit statuses back to the PR: preflight-manifest, preflight-config, deploy-to-stage, stage-smoke, stage-integration.

If all statuses are green and review is approved, the PR can merge. If any fail, the PR stays red; fix in dev, produce a new digest, and re-promote.
3 Stage environment (what’s running)

The exact digest from the manifest is now running in Stage, configured with Stage values/secrets. The stage branch is still unchanged; this is a dress rehearsal of the release.

4 PR status & merge decision

Green → Merge enabled. All required checks passed and ≥1 reviewer approved. Merge to record the promotion in stage. (No new deploy is needed; Stage is already on that digest.)

Red → Merge blocked. Something failed. Roll back Stage (e.g., helm rollback loans <rev>), fix on dev, produce a new digest, and re-promote with a fresh PR or an updated one.

Required checks to configure: preflight-manifest, preflight-config, deploy-to-stage, stage-smoke, stage-integration, plus “Up-to-date with base” and “Required review”.
Why the strategies differ: speed vs. safety

1) Feature → dev = Deploy after merge (speed)

Goal: fast, frequent integration. Dev should always equal dev@HEAD.
Why not pre-merge deploy? Many PRs are open; pre-merge deploys would overwrite each other and Dev would no longer reflect the branch.
Failure model: if the Dev deploy fails (even due to infra), fix fast or roll back; don’t block all other PRs. Dev is low-risk and shared.
2) dev → Stage/Prod = Merge after deploy (safety)

Goal: controlled promotion. Treat Stage/Prod as gates.
How: deploy the exact artifact during the PR, run Stage/Prod checks, merge only if green—so the branch history contains only proven promotions.
Benefit: clean audit trail and high confidence the thing that ran in Stage is exactly what you ship.
Analogy

Feature → dev = daily stand-up. Quick sync: you share progress, integrate fast, and handle issues without holding up the team.
dev → Stage/Prod = dress rehearsal. You perform the whole show first; only when everyone’s satisfied do you sign off and publish.
Promotion PR: stage → prod (Production Gate)

Alt text

This flow mirrors dev → stage: you promote the same artifact by digest, run checks during the PR, and merge only when the environment is healthy. The differences at Production are (1) stricter pre-deploy gates (approvals/change windows, prod config validation, flags default OFF) and (2) focused post-deploy verification (SLO guardrails + short synthetics, not Stage’s heavy suites). Below are the key deltas to highlight.

Open a promotion PR that references the exact Stage-proven artifact. Include the image and immutable digest, the commit/version, and a link to the manifest. Production branch rules are stricter: PRs only, up-to-date with base, manifest reachable, config renders cleanly, Stage run is green for this digest, and more reviewers (e.g., TL + on-call/QA or change approver/maintenance window if required).

CI runs Prod preflight and promotes the same artifact—no rebuild. The pipeline re-validates the manifest and resolves the digest, renders Prod config, checks that feature flags default OFF, confirms any change window/approval, and only then begins rollout of that same digest. (Production emphasizes stricter pre-deploy gates.)

Deploy with cautious rollout and instant rollback wired. Use the same mechanism in Stage and Prod, but different policy: Stage can do a quick canary (10% → 100%) to rehearse the flow; Prod uses a stricter canary (5% → 20% → 50% → 100%) or blue-green with SLO guardrails, flags OFF during ramp, and auto-rollback to the last good digest. “Flags OFF” = new code paths are disabled by default for users; you gradually enable them only after health looks good.

Post-deploy verification in Prod is focused (lighter than Stage). Don’t re-run Stage’s heavy test suites; rely on health/SLO guardrails (error rate, latency, saturation), short synthetic/smoke tests for critical journeys, and a brief dwell period with alerts quiet. (Stage already carried the deepest post-deploy checks.)

Merge only when Production is green, then record the release. Approvals + green checks enable merge; merging records the promotion (Prod is already on that digest). Add a release tag (e.g., v1.4.3) annotated with digest/commit, and update your change log/ticket.

If anything fails, roll back and re-promote. Auto-rollback (or flag-off), keep the PR unmerged/red, fix in dev, build a new digest, and repeat dev → stage → prod. Key idea: promotions deploy during the PR, merge only when Prod is healthy—with Prod prioritizing strict pre-deploy checks and Stage carrying heavier post-deploy testing.

Trunk-Based CI/CD: Environment Promotions

Overview (why / how):

Single trunk (main) is the source of truth; developers branch off it and send PR → main.
CI tests the tentative merge on the PR; when green, you merge.
A single immutable artifact is built once on main and then promoted (not rebuilt) through Dev → Staging → Prod.
Heavier checks happen in Staging; Production ships via CD (manual approval) or CDp (auto) with guarded rollout.
This cuts merge pain and speeds feedback; it works best with small PRs, strong tests, and feature flags.
Diagram at a glance

Alt text

One source of truth (main); Feature → main → Dev → Staging → Prod. CI validates the PR’s merge result; the post-merge pipeline builds once and publishes an image + digest + manifest.json. Promotions read the manifest and deploy by digest to each env.

Guardrails on main (protected branch)

PRs only: no direct pushes to main.
Required status checks gate the Merge button: build, unit tests, lint/format, quick security scans, and “up-to-date with main.”
(Many teams also require ≥1 review.) Keep reviews lightweight—1 CODEOWNERS approval, auto-merge via merge queue when checks pass. Allow trivial/docs-only PRs to auto-merge on green CI and treat pair/mob programming as the approval to preserve speed.
End-to-end flow (numbers match the diagram)

Alt text

1) Commit & push (feature branch)

Work on feature/topup-loan, commit locally, push to publish your change.

2) Open PR → main

Raise a PR from feature → main. Protection rules require passing checks; opening the PR triggers CI.

3) PR CI (Checkout • Build • Test)

CI tests a tentative merge: main + feature (main unchanged). Keep it fast (~10–15 min):

Checkout & build/package (Maven/Gradle/npm).
Fast tests (unit + a couple light integration).
Lint/format + quick static scans (SAST/SCA).
4) PR status gates merge

Green → merge enabled.
Red → merge blocked; push a fix and CI re-runs.
5) Merge PR into main

With green checks (and any required review), merge. main now points at a known-good merge commit.

6) Post-merge on main: build once & publish

The trunk pipeline creates the release candidate:

Build/package once, push the image to the registry.
Capture the immutable digest (e.g., ghcr.io/acme/loans@sha256:abc123…).
Publish a small manifest.json (e.g., GH Release / S3) with: image, digest, commit_sha, version, built_at. (You can add human-friendly tags like :v1.4.3, :9f3c2e7, or :9f3c2e7-v1.4.3 for discoverability, but deployments should use the digest.)
7) Dev: auto-deploy (CD) — same digest, no rebuild

A Dev job reads manifest.json, verifies the digest is pullable, renders Dev config (Helm/Kustomize), deploys by digest, and runs smoke/health checks. If red: roll back Dev to the last good digest; fix via a new PR → repeat.

8) Stage: promotion job / approval — same digest

When Dev is healthy, promote with an approval gate (often TL or QA):

Deploy the same digest with Stage values/secrets (no rebuild).
Run heavier checks: integration/contract, API/UI e2e (1–2 key flows), perf/security smoke. If anything fails: roll back Stage; fix on main via a new PR (produces a new digest in Step 6) and re-promote.
9) Prod: approval or CDp — same digest

With Stage green, promote to Prod:

Cautious rollout—canary (5% → 20% → 50% → 100%) or blue-green.
SLO guardrails (error rate, latency, saturation) + quick synthetics; ready rollback to the last good digest if guardrails trip.
Optionally tag the release (e.g., v1.4.3) and annotate with the digest + commit.
Jenkins in this picture (why & what)

What it is: Jenkins is an open-source automation server. It watches Git for changes and runs pipelines (as code) to build, test, package, and promote your software. It’s the glue between your Git host, build tools, scanners, registries, and environments.

Why teams use it:

Portable & extensible: Runs on a laptop, VM, or Kubernetes; 1,800+ plugins for build tools, scanners, clouds, and chat.

Pipeline-as-Code: A Jenkinsfile lives in the repo so CI/CD is versioned, reviewed, and reproducible.

Fits both strategies:

GitFlow: multibranch pipelines run CI on feature→dev PRs, then run promotion PR jobs (dev→stage→prod) that deploy the same digest.
Trunk-based: CI runs on PR→main; a post-merge pipeline builds once and publishes image + manifest.json; downstream stages (or GitOps PRs) promote that digest to Dev → Stage → Prod.
Orchestration end-to-end: One place to gate merges, sign artifacts, attach SBOMs, publish to a registry, create release notes/tags, request approvals, kick off canaries, and post status back to Git/Slack.

How it usually runs:

Controller + ephemeral agents (Kubernetes or cloud VMs) for isolation and scale.
Webhooks from your Git host trigger PR/merge builds; Jenkins posts green/red status checks that gate protected branches.
Credentials & secrets via Jenkins Credentials (or external managers like Vault).
Minimal, curated plugins + shared libraries for common steps (build once, write manifest.json, deploy by digest, open GitOps PR, etc.).
What it isn’t: Not a Git server, registry, or monitoring stack. Jenkins calls those systems; it doesn’t replace them.

Alternatives / complements: GitHub Actions or GitLab CI can do similar CI/CD. Argo CD/Flux excel at GitOps promotions (Jenkins can open the PRs they sync). Jenkins remains popular when you want maximum portability and customization across many tools and environments.

Conclusion

Key takeaways

CI runs on every push/PR to build + test and gate the merge; keep it fast.
CD/CDp promote the same immutable artifact (by digest) across Dev → Stage → Prod—no rebuilds.
Branch protection (PRs only, required checks, up-to-date with base, reviews) keeps mainlines deployable.
GitFlow vs Trunk-based: both should build once & promote; they differ in how code/branches are organized and how promotions are expressed.
Stage vs Prod: Stage carries heavier post-deploy tests; Prod enforces stricter pre-deploy gates and guarded rollout (canary/blue-green, SLO guardrails, ready rollback).
Jenkins is the orchestrator: pipelines as code to test, package, publish, and promote; it posts statuses back to Git and drives the same-digest promotions.
