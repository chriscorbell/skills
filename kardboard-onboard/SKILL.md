---
name: kardboard-onboard
description: Connect a coding agent to kardboard, the user's self-hosted kanban, and onboard a project onto it. Use when the user wants to connect their agent to kardboard, onboard or set up a repo or project for kardboard, or when an agent cannot see a project's board.
---

# Onboard onto kardboard

kardboard keeps every project the user works on as a board of cards, kept up to date by the user and their coding agents. It runs on the user's home server as a plain port with no sign-in, reached from their home network or tailnet and never from the public internet, at `http://minicore.saanen-monitor.ts.net:3071` for Chris, so this machine must be on one of them. Use the full tailnet name: the bare host name can resolve through another search domain first. Nothing on a board starts by itself. An agent reaches every board through one access token and the `kardboard` MCP server, finds the board for the repository it is in from `git remote get-url origin`, and files its side-findings as Backlog cards. kardboard never reads or merges pull requests: the agent merges with the user's own GitHub access, so a repository installs nothing from kardboard.

Onboarding has two halves: connecting the agent, once per machine, and onboarding the project, once per repository. Check each step's completion criterion first and do only the steps that fail.

## 1. Connect the agent

Check with `claude mcp get kardboard 2>/dev/null | grep -E 'Scope|Status'`. The filter matters: `claude mcp get` alone prints the token. For Codex, `codex mcp list` lists `kardboard`.

When it is missing, the user makes a token in kardboard's **Settings → Agent**, under **Connect an agent**, named after this machine. kardboard shows the token once, with the install command for Claude Code or for Codex. The user runs it, which keeps the token out of your transcript; if they hand you the command, run it unchanged and keep the token out of everything you write. The Claude Code command installs at user scope, so every project on the machine has the server; project scope would write the token into `.mcp.json`, which is committed.

An older `kardboard` entry in local scope, from when a token reached one board, still works, since every token now reaches every board. Remove it with `claude mcp remove kardboard -s local`, run in that project's folder, so the machine has one install.

Then let Claude Code use the board without a prompt per tool call: add `mcp__kardboard` to `permissions.allow` in `~/.claude/settings.json`, keeping whatever the file already holds.

Done when: `kardboard` shows Connected in user scope, and `~/.claude/settings.json` parses as JSON with `mcp__kardboard` in `permissions.allow`.

## 2. Find or create the project's board

An agent loads its MCP tools when its session starts, so the session that ran step 1 may lack them. Ask a fresh one, from the repository root:

```bash
claude -p "Call the kardboard get_board tool with board set to \"$(git remote get-url origin)\", and reply with the board's name, or with the error it returns." --allowedTools mcp__kardboard__get_board
```

When it names a board, that is this project's board. When no board matches, ask the user whether to create one. On yes, call `create_board` with the project's name and the repository's address, in this session if it has the tools or through `claude -p` as above with `--allowedTools mcp__kardboard__create_board`. A project with no GitHub repository gets a board too; create it without one, and agents name it by its slug.

Done when: `get_board` with this repository's remote, or the slug for a project without one, returns this project's board.

## 3. Point `AGENTS.md` at the board

The server's instructions tell a connected agent how to work a board; this section tells it, in this repository, to reach for it unprompted. Reuse `AGENTS.md` or `CLAUDE.md`; create `AGENTS.md` only if neither exists, and keep `CLAUDE.md` a relative symlink to it. Add:

```markdown
## Work tracking

Work on this project is tracked on its kardboard board, through the `kardboard` MCP server. Keep the board true to the work: find or create a task's card before starting it, move the card as the work moves, and file what you notice along the way as Backlog cards.
```

Done when: `AGENTS.md` carries the Work tracking section.

## When an agent cannot see its board

- No `kardboard` tools: the session started before the server was added. Start a new one.
- `401 unauthorized`: the token was revoked. Make a new one, as in step 1.
- A connection error: this machine cannot reach the server, or the MCP URL still names an old address, such as `https://minicore.saanen-monitor.ts.net/mcp` from before kardboard moved to its own port. `curl <address>/healthz` checks the first, and `claude mcp get kardboard | grep URL` the second. For an old address, remove the server with `claude mcp remove kardboard -s user` and install it again as in step 1.
- `no board matches`: the board has no repository, or a different one. Set it in kardboard's **Settings → Boards**, or name the board by its slug.

## Final checklist

- `kardboard` is Connected in user scope, and `~/.claude/settings.json` allows `mcp__kardboard`.
- A fresh session's `get_board` returns this project's board.
- `AGENTS.md` has the Work tracking section.
