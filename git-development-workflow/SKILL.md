---
name: git-development-workflow
description: Single-agent development workflow for any task in a git repository — isolate work on a branch/worktree, plan, implement, self-review, get the PR green, merge, and cut a release tag when the repo publishes on tags.
---

# Git Development Workflow

For any development task in a git repository. Complete the entire workflow yourself in one agent context: isolate, plan, implement, review, integrate, and verify. Do not spawn or delegate to sub-agents.

## 1. Understand the task

Restate the goal in one sentence. Ask only if the request is ambiguous or unsafe; otherwise proceed.

## 2. Isolate

1. Run `git status`. Note modified/staged/untracked files that aren't yours — preserve them untouched.
2. Never work directly on the default branch: create a branch (`feature/…`, `fix/…`, `refactor/…`, `docs/…`, `chore/…`) or a worktree if the main checkout must stay usable.

## 3. Rails check

Confirm the repo has PR validation and — if it publishes an artifact — a tag-triggered release workflow. Missing, or the prompt asks to set them up (/ci-cd-pipeline-setup): do that first on its own branch → PR → merge, so the feature PR runs against real checks. Already adequate: change nothing here.

## 4. Plan

Inspect the repository and write the smallest implementation path: files to touch, tests to add or update, and risks. Trim anything speculative before implementation.

## 5. Implement

Implement the approved plan directly with these constraints:

- Smallest change that satisfies the request. Follow existing patterns; no unrelated refactors, formatting churn, or public-API changes.
- Add/update tests for behavior changes; update docs when commands, config, or public usage change.
- Never run destructive git commands (`reset --hard`, `clean -fd`, force-push, history rewrite, branch deletion).
- Commit on the working branch with clear messages.

## 6. Self-review

Review the resulting diff for correctness, scope creep, test coverage, and convention adherence. Revert anything outside the plan, apply necessary cleanups directly, and record findings fixed and findings remaining.

## 7. Integrate and go green

While any PR review bot runs, you:

1. Review the final diff once more; revert anything unrelated that survived self-review.
2. Run the repo's validation (tests, lint, types, build — infer from package scripts, CI config, Makefile, README). Fix failures the change caused.
3. Open the PR if one doesn't exist: clear title, what/why, testing performed, risks.
4. Address CI and review-bot feedback until the PR is green. If a check can't run, say which one and why.

## 8. Merge and release

1. When the PR is green, merge it (squash unless the repo convention differs) and delete the working branch. If branch protection blocks you (required reviewer you can't satisfy), stop and report — never bypass protection.
2. Check whether the repo publishes on tags: a release workflow triggered by `v*` tags, or a release tool (release-please, changesets, goreleaser). No release setup → done, skip to Report.
3. Cut the release the way the repo already does it:
   - **Release tool**: follow its flow (merge the release-please PR, add a changeset, etc.) — don't hand-tag around it.
   - **Plain tag-triggered workflow**: bump the version in its single source (package manifest) if the merge didn't, then tag the merge commit on the default branch — patch for fixes, minor for features, major for breaking changes — and push the tag.
4. Watch the triggered release workflow (`gh run watch`) until it succeeds. A failed publish is your failure: fix and re-run, or report exactly what failed.

Only release changes that ship user-visible behavior; skip tagging for docs/CI/chore-only merges unless asked.

## Report

What changed and why, files touched, validation run and its actual status, remaining risks or skipped checks, PR link, and — if released — the tag and release-workflow result.

Never fabricate test results, CI status, PR links, tags, or benchmarks. Not done while relevant checks are failing or a triggered release workflow is red.
