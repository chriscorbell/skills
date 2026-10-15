---
name: hygiene
description: Hygiene pass on the current repository. Inventories stale worktrees, branches, stashes, temp files, outputs, leftover processes and outdated docs, then cleans up only what the user approves. Use when the user asks for a hygiene pass or a cleanup of leftovers.
---

# Hygiene pass

Inventory the leftovers, prove each one stale with evidence, clean up exactly what the user approves in a way that can be undone, and hand back the undo.

**Scope** is the current repository: its working tree, every worktree attached to it wherever that lives, its local and remote branches, its stashes, and processes running from its paths. Widen it only to what the user names, such as another repository, every repository under a folder, or machine caches.

## 1. Inventory

Read the project's instruction files (`AGENTS.md`, `CLAUDE.md`, nested ones included) and READMEs for paths it keeps on purpose (an untracked `.env`, local data), for files it treats as append-only, and for any procedure it has for maintaining its own docs or memory.

Sweep every category under [Categories](#categories). For each candidate, collect its evidence: path, size, age (last commit or modification), merge state, what references it, which tool created it, and whether it can be rebuilt. Everything in this step is read-only, except `git fetch --prune`, which branch state depends on.

A candidate needs two independent stale signals, and age counts as one at most. A candidate with a keep signal goes in the Keep tier.

Done when every category has been swept and every candidate has its evidence and a tier.

## 2. Report and get approval

Send one numbered report, grouped by category, one line per item: what it is, the evidence, the proposed action, its undo, and its tier.

- **Safe**: rebuildable, or fully recoverable from Git.
- **Review**: holds work that exists nowhere else, acts outside this machine (deleting a remote branch), or edits a doc.
- **Keep**: has a keep signal. List these briefly, unnumbered, so the user sees they were considered.

```text
Worktrees
1. [Safe] ~/.t3/worktrees/app/t3code-90bd (t3code/fix-login): merged in #42, clean, untouched 16 days, thread finished → remove worktree and branch. Undo: git branch t3code/fix-login b358ae9 && git worktree add ~/.t3/worktrees/app/t3code-90bd t3code/fix-login
```

End with the total space reclaimable and how to answer, for example "all safe, 7, 9" or "none". Wait for the answer. When part of it is ambiguous, ask about those items only.

## 3. Clean up

Act on exactly the approved items, using each category's method, and record each undo before acting. Use the non-forcing form of every command (`git worktree remove`, `git branch -d`). A refusal means it found something the inventory missed: report it and ask again rather than forcing.

Done when every approved item is done or reported as failed with its reason, and nothing unapproved has changed.

## 4. Final report

List what was removed, the space freed, the undo for each item with its SHA where there is one, and what was skipped or failed and why. Commit doc edits through the repository's normal workflow, with a branch and pull request when the repository uses them.

## Categories

### Worktrees

- **Look**: `git worktree list --porcelain`, which includes worktrees agent harnesses create outside the repository (T3 Code under `~/.t3/worktrees/`, Claude Code under `.claude/worktrees/`).
- **Stale**: its branch is merged or its pull request is closed, it has no uncommitted changes and no unpushed commits, and it hasn't been touched in a while. An entry marked `prunable` has lost its directory.
- **Keep**: changes or commits that exist only there, a process whose working directory is inside it, or a live agent thread or session bound to it. Check binding with the harness's own tools: when no live thread or session claims the worktree, it is unbound. Name the owning thread in the report, or say the owner couldn't be checked and make it Review tier.
- **Clean**: `git worktree remove <path>`, then its branch as below. `git worktree prune` for prunable entries.
- **Undo**: `git branch <branch> <sha> && git worktree add <path> <branch>`.

### Branches and stashes

- **Look**: `git branch -vv` for `[gone]` upstreams, `git branch --merged <default>`, `git branch -r --merged <default>`, `git stash list --date=relative`.
- **Stale**: merged into the default branch, or its upstream is gone. A branch never committed to (its reflog holds only `branch: Created from`) is the usual agent-harness leftover and holds nothing. A squash-merged branch isn't an ancestor of the default branch, so prove it merged through its pull request (`gh pr list --head <branch> --state merged`). A stash is stale once its contents (`git stash show -p`) are already in the default branch.
- **Keep**: commits found nowhere else, or an open pull request.
- **Clean**: `git branch -d`, or `git branch -D` for a branch approved as squash-merged or unmerged. `git push origin --delete <branch>` for a remote branch. `git stash drop stash@{n}`.
- **Undo**: record `<name> <sha>` first, then `git branch <name> <sha>`, `git push origin <sha>:refs/heads/<name>`, or `git stash store -m <message> <sha>`.

### Temp files and outputs

- **Look**: `git status --porcelain --ignored`, plus `find . -type d -empty -not -path './.git/*'` for empty folders, which Git can't see. Typical leftovers are logs, screenshots, `*.orig` and `*.rej` files, scratch scripts, eval and research dumps, `.DS_Store`, and build output or dependencies in abandoned worktrees.
- **Stale**: nothing references it (search the repository for its name), its name or location marks it as throwaway, and it is old. An untracked file that isn't ignored may be work never committed, so it is always Review tier. Rebuildable output in the active checkout comes straight back, so list it only when it is large or nothing builds it anymore.
- **Keep**: anything the project's instructions name, local data, and secrets (`.env`, keys, credentials). List secret files by path only, leaving their contents unread.
- **Clean**: delete rebuildable output (build directories, dependency folders, caches) outright. Move everything else to the system trash: `trash` on macOS, `gio trash` on Linux. Where neither exists, ask before deleting permanently.
- **Undo**: rebuild, or restore from the trash.

### Docs

- **Look**, in this order: instruction files and READMEs, then guides and runbooks, then plans, specs and TODO lists. Check what the repository itself can confirm; a claim about another machine or a live service stays unchecked unless the user widens the scope.
- **Stale**: it names a path, command, flag, script or environment variable that no longer exists (verify each with a search or `--help`), a relative link is broken, it plans work that shipped or was abandoned, or it repeats a fact another doc owns.
- **Keep**: history. Dated logs, research reports, decision records and history folders record their time and keep their content as written; a superseded ADR gets marked superseded. Files covered by the project's own procedure for maintaining docs or memory become one item: run that procedure's review.
- **Clean**: a small, exact edit, or a deletion with inbound links fixed. Show changes to instruction files (`AGENTS.md`, `CLAUDE.md`) as a diff in the report. A rewrite is a follow-up item for the user, outside the pass.
- **Undo**: revert the commit.

### Leftover processes

- **Look**: `lsof -nP -d cwd -Fpn` lists every process's working directory; narrow it to the repository and its worktrees, leaving out this session and its child processes. `lsof -nP -iTCP -sTCP:LISTEN` shows which of those serve a port.
- **Stale**: started for a session that has ended, with nothing depending on it. The user may have started it, so it is always Review tier.
- **Clean**: kill by PID. When going by port, include `-sTCP:LISTEN` (`lsof -ti tcp:<port> -sTCP:LISTEN | xargs kill`); without it, lsof also returns every client connected to that port.
- **Undo**: rerun its command, recorded first from `ps -o command= -p <pid>`.
