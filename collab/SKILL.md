---
name: collab
description: Work a problem jointly with the other coding agent — Claude and Codex each draft blind, challenge each other, and converge on one plan both sign off on. Use when the user invokes `/collab`, or asks to bring in / cross-check with the other model.
---

# Collab

Your **peer** is the other agent. You both answer the same brief, then argue to a single plan. The peer is not a sub-agent doing your bidding — it is an equal whose disagreement is the whole point, so the plans that matter are the ones neither of you could have written alone.

Deliverable is a plan, not an implementation. Implement only if the user asks afterwards.

## 1. Open the collab

Identify the peer and its call. You are Claude → peer is Codex. You are Codex → peer is Claude.

```sh
# calling Codex
codex exec -s read-only --skip-git-repo-check -o "$COLLAB/<reply>.md" "<prompt>" >/dev/null 2>&1
# calling Claude
claude -p --permission-mode plan "<prompt>" > "$COLLAB/<reply>.md" 2>&1
```

Both are read-only and stateless: the peer starts cold every call, keeps no memory of the last one, and writes no files. So every prompt states the task, names the absolute paths to read, and asks for the whole reply as markdown on stdout. Peer CLI missing or erroring → report it and stop; there is no collab with one agent.

Make the shared directory: `COLLAB=$(mktemp -d -t collab-XXXX)`. All files below live there.

## 2. Brief

Write `$COLLAB/brief.md`: the task in the user's words, the deliverable, hard constraints, the repo path and the files that matter, and the open questions a plan must answer. Resolve genuine ambiguity with the user now — a vague brief produces two plans that disagree about the question rather than the answer.

**Done when** a cold agent could plan from `brief.md` alone.

## 3. Blind drafts

Launch the peer's draft first with its output redirected to `$COLLAB/<peer>-plan.md`, then write `$COLLAB/<you>-plan.md` yourself. Yours is written blind — the peer's file stays closed until yours is on disk.

Both drafts check the brief's ground truth against the machine rather than inheriting it — say so in the peer's prompt. A fact the brief got wrong otherwise lands in both plans and survives the challenge unexamined, because neither side ever looked.

Same brief, same shape for both:

**Approach** (a paragraph) · **Steps** · **Key decisions and why** · **Risks and unknowns** · **Alternatives rejected, and why**

**Done when** two plans exist and yours was written without sight of the peer's.

## 4. Challenge

Send the peer both plan paths and ask for its challenge; write yours blind against the same pair. A challenge quotes the specific claim, says why it is wrong or unsupported, and states what it would do instead. "Looks reasonable" is not a challenge — if you find nothing to challenge, you have not read closely enough. Note the concessions too: where the other plan is plainly better, say so.

**Done when** every substantive difference between the two plans appears in one of the challenge files with a position on it.

## 5. Converge

Work the open disagreements. Each one ends when a side concedes with a reason, or both of you land on a third option neither drafted. Fold the settled ones into `$COLLAB/consensus.md`, send it to the peer, and take its next round. Three exchanges is the budget; anything still open after that is a standoff, not a discussion.

Hold your position while your argument survives the peer's. Conceding to close the round is how two agents produce a plan neither believes.

**Done when** the peer, given `consensus.md`, replies with explicit approval — or `consensus.md` ends with an **Unresolved** section stating both positions on each standoff.

## Report

Give the user the consensus plan, the disagreements and how each resolved, any standoff needing their call, and `$COLLAB`. The disagreements are the payoff — lead with them, not with the plan's agreeable parts.
