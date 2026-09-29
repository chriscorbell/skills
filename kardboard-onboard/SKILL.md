---
name: kardboard-onboard
description: Prepare a GitHub repository for kardboard, the kanban at kardboard.cc, for a board that runs Sessions or one the user's own agent works through an access token. Use when the user wants to connect, onboard, or set up a repo or project for kardboard, connect their agent to a kardboard board, or asks why Sessions on a board cannot push or merge.
---

# Onboard a repository for kardboard

A kardboard board is one of two kinds, and the kind decides the whole setup:

- **Sessions on.** Every change people make on the board starts a **Session**: a disposable container that clones this repository on a branch named `kardboard/<card id>-<slug>`, reads `AGENTS.md`, does the work or asks about it, runs the acceptance command, and opens a pull request with `gh`. A member presses Approve, and kardboard merges through a second GitHub App that bypasses a branch ruleset. This kind suits client work.
- **Sessions off.** The board is a shared record of the work. The user's own agent, running on their machine, reads and changes it over MCP with an **access token**, acting as the board's agent, and merges its own pull requests. Nothing on the board starts by itself, and kardboard merges nothing. A new board starts this way.

Work from the repository root. Find the repository with `gh repo view --json nameWithOwner,defaultBranchRef -q '.nameWithOwner + " " + .defaultBranchRef.name'`. Onboarding is done when every item in the chosen kind's final checklist is verified, not merely written.

## Choose the kind

1. Look for signs that the repository is already on kardboard:
   - `claude mcp list` in this folder lists `kardboard`: a board with Sessions off.
   - `gh api "/repos/OWNER/REPO/rulesets" -q '.[].name'` lists `kardboard`, or `git ls-remote --heads origin 'kardboard/*'` finds branches: a board with Sessions on.
   - `AGENTS.md` or `CLAUDE.md` mentions kardboard.
2. With no sign, or signs of both kinds, ask the user whether the board will run Sessions, offering the two kinds above as the choices, and wait for the answer. With signs of one kind only, tell the user which kind you found and continue with it.
3. Follow that kind's section below. For a repository already on kardboard, check each item of its final checklist first and do only the steps whose items fail.

Done when: the user's answer, or the one kind you found, names the section you follow.

## Sessions on

### 1. Write `AGENTS.md` for a Session

A Session reads `AGENTS.md` once and follows it. Reuse an existing `AGENTS.md` or `CLAUDE.md`; create one only if none exists, and keep `CLAUDE.md` a relative symlink to `AGENTS.md`.

The file must give a Session what it cannot discover by looking:

- The **acceptance command** to run before opening a pull request, and what passing looks like. Verify the command yourself by running it; a command you have not run does not go in the file.
- Setup that is not obvious from the manifest: env files to copy, services to start, generated code to build.
- Conventions the code does not enforce: where new pages or modules go, naming, what never to touch.

Leave out anything the environment already states (scripts in `package.json`, the directory tree). A Session's container has Node with pnpm and Bun, Python with uv, Go, git, and `gh`, no Docker, no browser, and a 45-minute wall clock; an acceptance command that needs more than that must be replaced or scoped down here.

Done when: `AGENTS.md` names one acceptance command you ran successfully in this checkout.

### 2. Make pull requests from Session branches usable

- CI must run on pull requests from branches in this repository, since Session branches are same-repo branches. Check the workflow triggers include `pull_request`.
- If the project deploys pull-request previews on its own (Cloudflare Pages, Vercel), note the preview URL pattern in `AGENTS.md` so the Session can link it on the card. Without one, the board runs in external preview mode with no preview link.

Done when: a pull request from a `kardboard/*` branch would get the same checks as any other.

### 3. Install both GitHub Apps on the repository

Only the repository owner can do this; open the two links for the user and wait:

- https://github.com/apps/kardboard-sessions/installations/new
- https://github.com/apps/kardboard-merge/installations/new

Choose "Only select repositories" and pick this repository. On a client-owned repository the client installs them from the same links.

Done when: the board's settings dialog in kardboard admin shows both apps as installed (step 5 creates the board).

### 4. Protect the default branch with the `kardboard` ruleset

The ruleset requires a pull request with one approval. Its bypass actors are the merge app and repository admins, so a Session's token cannot merge while the owner keeps pushing directly. The approval is what stops a Session merging its own pull request, so keep it at one. Create the ruleset with one API call, feeding the JSON below on stdin; it already carries the merge app's actor id:

```bash
gh api -X POST "/repos/$(gh repo view --json nameWithOwner -q .nameWithOwner)/rulesets" --input - <<'JSON'
{
  "name": "kardboard",
  "target": "branch",
  "enforcement": "active",
  "conditions": {
    "ref_name": {
      "include": [
        "~DEFAULT_BRANCH"
      ],
      "exclude": []
    }
  },
  "bypass_actors": [
    {
      "actor_id": 4943268,
      "actor_type": "Integration",
      "bypass_mode": "always"
    },
    {
      "actor_id": 5,
      "actor_type": "RepositoryRole",
      "bypass_mode": "always"
    }
  ],
  "rules": [
    {
      "type": "deletion"
    },
    {
      "type": "non_fast_forward"
    },
    {
      "type": "pull_request",
      "parameters": {
        "required_approving_review_count": 1,
        "dismiss_stale_reviews_on_push": true,
        "require_code_owner_review": false,
        "require_last_push_approval": false,
        "required_review_thread_resolution": false,
        "allowed_merge_methods": [
          "squash"
        ]
      }
    }
  ]
}
JSON
```

If the repository already has a ruleset named `kardboard`, leave it, and tell the user if its required approvals are below one. Requires admin on the repository; on a client-owned repository, hand the JSON and the command to the client.

Done when: `gh api /repos/OWNER/REPO/rulesets -q '.[].name'` lists `kardboard` and the ruleset page shows enforcement Active.

### 5. Create the board

The user does this at https://kardboard.cc/admin/boards: name, slug, and this repository's URL; **Sessions** ticked, which reveals provider and model, preview mode (`external` unless the repository has a Dockerfile and runner previews are enabled), and the rest; and the members who may open it. Members must sign up at kardboard.cc with the invited email before they can sign in.

Done when: the board exists with Sessions on and its settings dialog shows both apps installed.

### 6. Prove the loop

Have the user create one small, concrete card on the board, for example a one-line copy change. Within about a minute a Session starts. Done when the card reaches Review with a pull request link and a passing acceptance command in the pull request, and pressing Approve merges it and moves the card to Done.

### Final checklist

- `AGENTS.md` names a verified acceptance command.
- CI runs on pull requests from same-repo branches.
- Both apps show as installed on the board.
- The `kardboard` ruleset is active on the default branch, with one required approval.
- One card has gone Inbox to Review to Done through a merged pull request.

## Sessions off

The user's agent does the work and merges with the user's own GitHub access, so this repository needs neither GitHub App nor the `kardboard` ruleset. A `kardboard` ruleset left from an earlier Sessions setup, while it requires an approval, blocks merges by anyone but a repository admin; tell the user if you find one.

### 1. Create the board

The user does this at https://kardboard.cc/admin/boards: name, slug, and this repository's URL, with **Sessions** left unticked. The URL is optional; with it, kardboard refuses a pull request link from any other repository. Members are optional too.

Done when: the board exists and its row in the admin board list says Sessions off.

### 2. Connect the agent

The user saves the board, opens its settings again, and under **Access tokens** creates a token named after the machine the agent runs on. kardboard shows the token once, beside a `claude mcp add` command for it. The user runs that command in this repository's folder, which keeps the token in their own Claude Code config and out of your transcript. If they hand you the command instead, run it unchanged and keep the token out of everything you write.

The command adds the server in Claude Code's local scope, private to the user and this folder. Keep it there: project scope writes the token into `.mcp.json`, which is committed with the repository. Another agent connects the same way through its own user-level MCP config: an HTTP server at `https://kardboard.cc/mcp` with the header `Authorization: Bearer <token>`.

Done when: `claude mcp get kardboard | grep -E 'Scope|Status'`, run in this folder, shows local scope and Connected. The filter matters: `claude mcp get` alone prints the token.

### 3. Point `AGENTS.md` at the board

An agent calls the board's tools only when something tells it the board exists. Reuse `AGENTS.md` or `CLAUDE.md`; create `AGENTS.md` only if neither exists, and keep `CLAUDE.md` a relative symlink to it. Add this section, with the board's slug:

```markdown
## Work tracking

Work on this project is tracked on the kardboard board `<slug>`, through the `kardboard` MCP server. Keep the board true to the work: find or create a task's card before starting it, and move the card as the work moves. The server's instructions say what each column is for.
```

Done when: the section names this board's slug.

### 4. Let the agent use the board without asking

Claude Code asks before every MCP tool call until a rule allows it, which turns each card move into a prompt. Add `mcp__kardboard` to `permissions.allow` in `.claude/settings.json`, keeping whatever the file already holds; the rule covers every tool of the `kardboard` server. The file holds no secret, so commit it with the `AGENTS.md` change, and every checkout of the repository gets the rule once the user accepts Claude Code's trust prompt for the folder.

Done when: `.claude/settings.json` parses as JSON and its `permissions.allow` lists `mcp__kardboard`.

### 5. Prove the connection

An agent loads its MCP tools when its session starts, so the session that ran the setup may lack them. Read the board from a fresh one:

```bash
claude -p "Call the kardboard get_board tool and reply with the board's name." --allowedTools mcp__kardboard__get_board
```

Done when: the reply names this board.

### Final checklist

- The board exists with Sessions off.
- `kardboard` is Connected in local scope for this folder.
- `AGENTS.md` names the board's slug in a Work tracking section.
- `.claude/settings.json` allows `mcp__kardboard`.
- A fresh agent session read the board through `get_board`.
