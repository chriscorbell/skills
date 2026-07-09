---
name: ci-cd-pipeline-setup
description: Set up, review, or fix CI/CD for a repository — PR validation, branch protection, release/deploy gating, tag-triggered release and publish workflows, least-privilege permissions. GitHub, GitLab, Bitbucket, or similar.
---

# CI/CD Pipeline Setup

For setting up, reviewing, or improving CI/CD automation. Prefer simple, reliable automation over complex pipelines — add what the repo needs, not every possible workflow.

On a repo that already meets the baseline, verify and report "no changes needed" — don't churn. When invoked alongside a development task (e.g. with /git-development-workflow), land pipeline changes first in their own PR so the task's PR runs against real checks.

## 1. Inspect first

Identify language, package manager, test/build commands, and deployment model from existing files: workflows (`.github/workflows/`, GitLab/Bitbucket CI files), package scripts, Makefiles, Dockerfiles, docs. Reuse existing scripts in CI rather than duplicating commands. If the remote can't be inspected, infer and state assumptions.

## 2. Baseline for an active repo

- PR + main-branch validation workflow (format → lint → types → tests → build, cheap checks first)
- Required checks before merge
- Dependabot/Renovate (group patch/minor, separate majors, run normal CI on their PRs)
- Secret scanning + dependency vulnerability alerts; CodeQL for supported languages
- Prefer the platform-native CI (GitHub Actions on GitHub) unless the project already uses another

For a **new empty repo**: minimal foundation only — README, PR workflow once real commands exist. Never invent build/test commands; document intent instead of committing placeholder steps.

For an **existing repo**: incremental improvements, don't rewrite. Don't break existing required-check names or deployment behavior.

## 3. Gating rules

- Build/release/deploy jobs `needs:` successful validation.
- Never publish packages, images, or releases from untrusted PR contexts; PRs may only deploy to isolated previews.
- Production deploys: passing validation + manual approval (protected environment) unless the project explicitly does continuous deployment.
- Tag-based or manually approved releases; document trigger and rollback.

## 4. Release and publish workflows

When the repo ships an artifact (package registry, container image, binary, GitHub Release), add a release workflow — don't for libraries/apps that only deploy or repos with nothing to publish.

- Trigger on semver tags (`v*`) or manual dispatch; never on every push to main unless the project explicitly does continuous release.
- Version lives in one place (package manifest); the tag matches it. Prefer an existing release tool (release-please, changesets, goreleaser) over hand-rolled bump scripts if the ecosystem has one — but don't introduce one into a repo that releases rarely.
- Publish job runs the full validation suite first (`needs:`), then builds from the tagged commit only.
- Use OIDC/trusted publishing (npm provenance, PyPI Trusted Publishers) over long-lived registry tokens where the registry supports it.
- Generate release notes from commits/PRs; create the GitHub Release as part of the workflow.
- Document in the README or workflow comments: how to cut a release, and how to yank/rollback one.

## 5. Permissions and secrets

```yaml
permissions:
  contents: read
```

Default read-only; grant write per-job only where needed. Environment-scoped secrets; never expose secrets to fork PRs or run untrusted code with privileged tokens. Prefer OIDC over long-lived cloud credentials.

## 6. Speed and reliability

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

Dependency caching, pinned runtime versions, lockfile installs, job timeouts, deterministic tests (quarantine flaky ones — don't require them as checks). Matrix builds only when multiple versions/platforms actually matter.

## 7. Branch protection

Attempt it with `gh api` / `gh repo edit` (or platform equivalent) before declaring it manual; if the token lacks admin rights, give exact manual steps.

Require: PRs before merge, passing status checks, conversation resolution. Require an approving human review only when humans review this repo — on solo or agent-operated repos it deadlocks autonomous merges; rely on required checks and review bots instead. Block force pushes and branch deletion. CODEOWNERS review for sensitive paths (workflows, infra, auth, migrations). Don't require flaky or experimental checks.

## 8. Bots

Only when they add signal: dependency updates, security alerts, release notes, preview deploys. Avoid overlapping bots and low-signal comment noise. Least privilege; their PRs still pass normal validation.

## 9. Validate and report

Check YAML syntax, triggers, job deps, permissions, secrets usage, branch/tag filters. Run the equivalent local commands. Trigger a real run or test PR when possible.

Report: files changed, checks included, gating structure, and — clearly separated — **remote settings you could not set via CLI** (branch protection, environments, secrets, bot installs) with exact steps. Never claim remote settings, workflow runs, or deployments are active unless actually verified.
