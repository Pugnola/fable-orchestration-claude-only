# fable-orchestration-claude-only

A [Claude Code](https://claude.com/claude-code) skill for running a whole agent stack on **Claude alone**: a scarce, expensive main loop (Claude Fable) that owns judgment, and **one** delegate tier (Claude Opus) that does everything a brief can describe: exploration, code, review, verification.

No Codex, no Gemini, no Sonnet/Haiku lanes. One model family, four lane types, and a contract that keeps the delegates honest.

**Core law: the main loop's tokens buy judgment, not throughput.** Fable writes the brief, reads the report, decides and integrates. Opus writes the code.

---

## Why this exists

The usual way to get good output from a multi-agent stack is to mix model families: a Codex lane builds, a Claude lane reviews, and their blind spots don't overlap. Without a Codex subscription, that half of the stack disappears.

This skill keeps what made that setup work (the doctrine, not the mechanics) and rebuilds it on Opus:

- the lane brief as a **contract** (goal, read-first, owned files, fences, acceptance commands, report shape),
- **ownership lists**, so concurrent writers never touch the same file,
- **acceptance checks re-run by the orchestrator**, because a lane's "done" is a claim, not evidence,
- **worktree isolation** when more than one builder writes,
- a deliberately **adversarial, fresh-context reviewer** to stand in for the second model family.

## The four lanes

Each lane is a Claude Code agent definition (`agents/*.md`), all `model: opus`. Lanes differ by **definition and brief**, never by model. The definition fixes the tools and effort, so a read-only lane physically cannot write.

| Lane | Tools | Brief | Hands back |
|---|---|---|---|
| `opus-scout` | read-only | a goal; can be loose | a map: numbers with the query that produced them, paths with line numbers, flags |
| `opus-builder` | read + write | complete contract (see below) | diff, verbatim acceptance output, **every deviation with its reason** |
| `opus-reviewer` | read-only, fresh context | "assume this diff is wrong" | ranked findings: `file:line`, failure scenario, CONFIRMED / PLAUSIBLE |
| `opus-verifier` | read-only + shell | the acceptance list, verbatim | PASS / FAIL / COULD NOT RUN per check, raw output, no fixes |

## Routing by difficulty

Every lane is Opus, so the question is how much scaffolding a job needs, not which model gets it.

- **Unknown terrain:** scout first. The scout's report becomes the builder's read-first list.
- **Simple, bounded work** (a templated page, a clone of an existing pattern): one builder, then the orchestrator spot-checks and commits.
- **Hard or precision work** (core seams, security-sensitive changes, correctness-critical paths): builder, then reviewer gate, then verifier.
- **Judgment, specs, synthesis, commit messages:** the main loop, always.
- **Two-line changes:** the main loop does them directly. A page-sized lane cost 90–130k Opus tokens and 3–7 minutes in practice, so small work isn't worth a lane.

## The brief contract

A lane can't see the conversation, so the brief is its entire world. Every builder brief carries:

1. **GOAL**: what exists when the lane is done.
2. **READ FIRST**: ordered files, one line each on what to take from them.
3. **OWNS**: files it may create or edit.
4. **DO-NOT-TOUCH**: files it must not edit, each with its owner.
5. **CONSTRAINTS**: no git, no new deps, no DB writes, what must not be restarted.
6. **ACCEPTANCE**: exact commands with **expected output**, so the lane compares instead of guessing.
7. **REPORT**: line counts, verbatim check tails, and **every deviation from the brief with its reason**.

The deviation sentence matters most. Opus improvises readily, and that sentence turns a silent improvisation into a reported decision the orchestrator can accept or revert.

## Install

```bash
# 1. The skill
git clone https://github.com/Pugnola/fable-orchestration-claude-only.git \
  ~/.claude/skills/fable-orchestration-claude-only

# 2. The four lane definitions
mkdir -p ~/.claude/agents
cp ~/.claude/skills/fable-orchestration-claude-only/agents/*.md ~/.claude/agents/
```

Start a new Claude Code session. The agents appear as `opus-scout`, `opus-builder`, `opus-reviewer`, `opus-verifier`; check with `/agents`.

## Use

Load the skill at the start of a session where you plan to delegate:

```
/fable-orchestration-claude-only
```

Then work normally. The main loop follows the policy: it scouts before building, writes full briefs, sends independent lanes out in one message, re-runs acceptance checks before committing, and continues a lane with `SendMessage` instead of respawning it.

## Know the limits

- **One model family has one set of blind spots.** Adversarial briefs and fresh-context reviewers help, but they are not a second family. On high-stakes disputes, **you** arbitrate.
- **Two Opus judges are not independent.** For dataset labeling, use different prompts, report the agreement rate as a caveat, and route disagreements to a human.
- **No image generation.** Claude has no built-in image tool; keep a separate image skill if you need assets.
- **Model names.** The policy assumes a Fable main loop. On a different top model, the rules still hold: whatever you run in the main loop is the scarce tier, and Opus is the delegate.
