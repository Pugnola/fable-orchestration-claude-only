---
name: fable-orchestration-claude-only
description: "Delegation policy for a Claude-only stack: a scarce Fable main loop and ONE delegate tier, Opus, for everything delegated — code writing, exploration, review, verification. No Codex lanes, no Sonnet/Haiku lanes. Merges the routing law of fable-orchestration with the lane doctrine of codex-fleet (brief contract, ownership lists, acceptance checks, worktree isolation, 'completions are claims'). Load whenever spawning sub-agents (Agent tool, Workflow agent() calls) or planning any delegation without a Codex subscription."
---

# Fable orchestration — Claude only

A routing policy for a stack with **one scarce, expensive main-loop model** (Claude
Fable) and **one unlimited delegate tier** (Claude Opus). It replaces both
`fable-orchestration` and `codex-fleet` when there is no Codex subscription. What
carried over from those two is the doctrine, not the mechanics: the lane brief as a
contract, ownership lists, acceptance checks run by the orchestrator, worktree
isolation for concurrent writers, and the rule that a lane's "done" is a claim.

Core law: **Fable's tokens buy judgment, not throughput.** Fable writes the brief,
reads the report, decides and integrates. Opus does everything a brief can describe,
including the code.

## The hard laws

1. **Fable is never a sub-agent.** Every `Agent` call, every Workflow `agent()` call
   and every `meta.phases` entry names `model: "opus"` explicitly. Omitting the model
   inherits the parent, which silently fans out the scarce tier.
2. **One delegate tier.** Opus for exploration, building, review and verification
   alike. No Sonnet or Haiku lanes "to save effort" (Opus is unlimited for this
   operator; a cheaper lane saves nothing and adds a second quality regime), no Codex.
   Lanes differ by their **agent definition and their brief**, never by model.
3. **The main loop keeps the big picture.** Architecture, contracts between modules,
   subtle state machines, conflict resolution, final synthesis, commit messages and
   every judgment call stay with Fable. Fable does not type code a brief can describe,
   and does not re-read files a lane has already summarized.
4. **Completions are claims, not evidence.** A lane's acceptance check is re-run before
   integration: by Fable when it is one command, by an `opus-verifier` lane when it
   is a batch. Nothing is committed on a report alone.

## Lane types

Four agent definitions live in `~/.claude/agents/`, all `model: opus`. The Agent tool
has no per-call effort or tool knob; both come from the definition, so the definition
carries what Codex's `high`/`xhigh` and `read-only`/`workspace-write` flags used to.

| Lane | Definition | Tools | Brief | Deliverable |
|---|---|---|---|---|
| Scout | `opus-scout` | read-only | may be loose: a goal | understanding — a map, numbers, options, flags |
| Builder | `opus-builder` | write | must be complete (see contract) | diff + verbatim acceptance output + deviations |
| Reviewer | `opus-reviewer` | read-only, fresh context | adversarial: "find what breaks" | ordered findings, file:line, failure scenario, CONFIRMED / PLAUSIBLE |
| Verifier | `opus-verifier` | read-only + shell | the acceptance list, verbatim | pass/fail per check with raw output, no fixes |

Spawn with `subagent_type: "<definition name>"` and still pass `model: "opus"`.

## Routing

The axis is **difficulty**, not "recon vs execution" — everything is Opus now, so the
question is how much scaffolding the lane needs.

- **Unknown terrain → scout first.** Never send a builder into code Fable has not had
  mapped; the scout's report becomes the builder's "read first" list.
- **Simple, well-bounded work** (a templated page, a clone of an existing pattern, a
  feature that rides an existing pipeline): one builder, a brief that names the
  reference files and the acceptance commands, no reviewer. Fable spot-checks (one
  screenshot, one `git diff --stat`) and commits.
- **Hard or precision work** (core evaluator seams, security-sensitive
  strengthen-never-relax changes, correctness-critical paths, the largest surfaces):
  builder with a complete spec **plus** a reviewer gate on the diff **plus** a verifier
  on the acceptance list. Worktree isolation when more than one builder writes.
- **Judgment / synthesis / spec writing:** Fable itself, always.
- Prefer Fable doing a two-line change hands-on over spawning a lane; delegation has
  its own overhead (a page-sized build cost 90–130k Opus tokens and 3–7 minutes in
  practice). Below that scale, just do it.

## The brief contract (every builder brief carries all of it)

A lane cannot see the conversation. The brief is its entire world.

1. **GOAL** — one paragraph, what exists when the lane is done.
2. **READ FIRST** — an ordered list of files with one line each on what to take from
   it. Include the design/contract docs the work must obey.
3. **OWNS** — the files it may create or edit.
4. **DO-NOT-TOUCH** — the files it must not edit, each with who owns it (a sibling
   lane, the user, "unchanged by decision").
5. **CONSTRAINTS** — no git, no new dependencies, no renames while a dev server
   watches the tree, no `.py` while the engine runs, no DB writes, which `.md` files
   (if any) it may edit, what it must not restart.
6. **ACCEPTANCE** — exact commands and the expected output for each (typecheck, lint,
   captures at named viewports, click-through evals with expected strings, a live-mode
   sanity check). Expected numbers go in the brief so the lane can compare, not guess.
7. **REPORT** — files with line counts, verbatim tails of every check, every eval
   result verbatim, screenshot paths, defects seen and fixed, **every deviation from
   the brief with its reason**, anything not done.

The deviation sentence is not optional. Opus improvises more readily than Codex did;
that sentence turns an improvisation into a reported decision Fable can accept or
revert. A brief without acceptance commands produces a lane that verifies nothing.

Scout and reviewer briefs are shorter: goal, read-first, what to report, a word
budget. A reviewer brief also states what the builder claimed, so the reviewer can
check claims against the diff.

## Fleet rules (more than one lane)

- **Independent lanes go out in one message.** The harness notifies on completion;
  never poll, never schedule wake-ups to check, never predict a pending result.
- **Concurrent writers get isolation.** Either `isolation: "worktree"` per builder with
  one commit per lane and Fable cherry-picking in completion order (gate at each
  pick), or a shared tree with **disjoint OWNS lists** and shared files (barrels,
  routers, token files) declared append-only and allocated to exactly one lane.
- **Read lanes cannot write.** The definition enforces it; do not tell a scout or a
  reviewer to "just fix it" — spawn a builder.
- **Fable runs the single gate** (typecheck + lint + tests + captures) after all lanes
  settle, repairs cross-lane breakage itself only when it is a two-line integration
  fix, otherwise sends it back to the owning lane with `SendMessage`.
- **Resume, don't respawn.** A lane's context survives; continuing it with
  `SendMessage` is cheaper and more accurate than a new lane re-reading the tree.
  Respawn only when the lane's context is polluted or it stopped reporting.
- **Honest failure beats a cheerful lie.** A lane reporting "blocked: X is outside my
  OWNS" is doing its job; integrate what it delivered and route X.

## Quality without a second model family

The Codex stack had one advantage this stack lacks: a reviewer from a different model
family with different blind spots. Compensate deliberately:

- Every reviewer is a **fresh context** (never the builder resumed) with an
  **adversarial** brief: assume the diff is wrong, find the input that breaks it,
  check each claim in the builder's report against the diff, list DO-NOT-TOUCH
  violations and scope creep.
- Findings carry a verdict: CONFIRMED (reproduced or traced to a concrete failure) or
  PLAUSIBLE (argued, not shown). Fable acts on CONFIRMED, decides on PLAUSIBLE.
- **The user arbitrates** on high-stakes disputes; Fable samples (~10% of items or one
  screenshot) rather than reading everything.
- For dataset labeling, two Opus judges are **not** independent the way Opus + Codex
  were. Use different prompts and sampling, report the agreement rate as a caveat,
  and route disagreements to the user, not to Fable.

## Token discipline for the main loop

Fable spends on: reading the request, writing the brief, reading the report, viewing
one artifact, deciding, committing. When the operator says the Fable quota is low,
reviews and verification also go to lanes and Fable reads reports only. Fable never
opens a lane's transcript file; it reads the hand-back report. If a report is missing,
ask the lane to resend a compact version with `SendMessage`.

## Out of scope

- **Image generation.** Claude has no equivalent of Codex's `gpt-image-2`. Keep a
  separate, API-key-based image skill if assets are still needed, or drop image work.
  This skill never promises rendered images.
- **Codex CLI mechanics** (`codex exec`, sandbox flags, `--approve-for-me`, stdin
  redirects, fast tier): gone with the subscription. Memories that describe them are
  historical.

## Exceptions

- If the operator names a model for a scoped task, honor it for that task only, then
  return to this policy.
- The operator can override any of this per session; absent that, this policy stands.
