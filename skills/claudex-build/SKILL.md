---
name: claudex-build
description: Run the full multi-model feature loop in one thread - Claude plans, reviews and triages; Codex (via the codex plugin) critiques the plan and implements; dual parallel review with Codex as background edge-case grinder; Fable escalation for architectural work (relaunch the loop on Fable for planning, read-only Fable subagent for high-risk review). Every run logs rework metrics to docs/reviews.md. Use when the user invokes /claudex-build, asks for "the full loop", or wants a feature built with cross-model critique, implementation and review.
---

# Claudex Build

One feature end-to-end in a single thread, with role separation and empirical
tracking:

- **Claude** (this session, normally Opus): plans, reviews, triages findings,
  applies taste fixes, commits.
- **Codex** (via the `codex-plugin-cc` plugin commands, fallback `codex exec`):
  critiques the plan, implements, grinds edge cases in review, applies
  mechanical fixes.
- **Fable** (escalation, not default): for architectural, niche, or
  design-driven work. Planning escalates by relaunching the skill with Fable
  as the session model; high-risk review uses a read-only Fable subagent,
  skipped when the session already runs on Fable.

Every run measures how much Claude had to rework Codex's code, so the log in
`docs/reviews.md` eventually answers empirically whether Codex-as-implementer
holds up on quality and taste.

## Arguments

The skill argument is the feature description. Modifier flags anywhere in it:

- `--fast` — skip the plan critique (phase 2).
- `--auto` — no checkpoints; run all phases without waiting for approval.
- `--high-risk` — force the Fable proposal and the extra adversarial pass.
- `--claude-impl` — Claude implements instead of Codex (quality-first override).
- `--throwaway` — code that will not be maintained: skip planning and review,
  delegate implementation to Codex, verify minimally, stop. No metrics log.

Auto-detect high-risk even without the flag when the change touches auth,
payments, DB migrations, or widely shared core modules.

## Codex invocation

Prefer the plugin commands when available in the session:
`/codex:adversarial-review`, `/codex:review`, `/codex:rescue`,
`/codex:status`, `/codex:result`. Pass the loop's constraint prompts as free
text arguments to the commands.

If the plugin is unavailable, fall back to raw `codex exec` (inherit model and
effort from the user's `~/.codex/config.toml`, never override):

```bash
ARTIFACT_DIR="$(mktemp -d "${TMPDIR:-/tmp}/claudex-build.XXXXXX")"

# Read-only leg (plan critique, review):
codex exec -C "$PWD" --ephemeral -s read-only \
  -o "$ARTIFACT_DIR/report.md" - < "$ARTIFACT_DIR/prompt.md"

# Write leg (implementation, mechanical fixes):
codex exec -C "$PWD" --ephemeral -s workspace-write \
  -o "$ARTIFACT_DIR/report.md" - < "$ARTIFACT_DIR/prompt.md"
```

If `codex` itself is missing or broken, report it and offer the degraded loop:
Claude implements, a fresh subagent reviews.

## Phase 0 — Setup

1. `git status --short`. If the tree is dirty: **warn the user, list the dirty
   files, and proceed anyway** — stopping is the user's call. Keep the dirty
   file list; it is needed later to flag polluted metrics.
2. No branch by default. For medium/large or risky tasks, propose a branch and
   ask before creating it. Never create one silently.
3. If the scope is genuinely ambiguous (materially different readings), ask
   now via AskUserQuestion.

## Phase 1 — Plan (Claude)

Explore the codebase enough to plan concretely, then write the plan to
`docs/plans/YYYY-MM-DD-<slug>.md` with: goal, acceptance criteria, file-level
steps, files to leave alone, verification commands, open questions.

If a plan file for this feature already exists — typically after a Fable
escalation relaunch — pick it up and refine it instead of starting over.

The bar is higher than a plan Claude would implement itself: Codex implements
only what the plan specifies, so vague steps become vague code.

While exploring and drafting, surface doubts instead of assuming: whenever the
goal, the scope, or a design choice admits materially different readings, or
the plan needs information only the user has (expected behavior, priorities,
constraints not visible in the code), ask via AskUserQuestion before writing
the affected steps — grouped in one call when the questions are related, not
one interruption each. A wrong assumption here gets implemented mechanically
by Codex. With `--auto`, don't ask: make the most reasonable assumption and
record it under the plan's open questions.

**Fable rubric** — when any of these hold, or when `--high-risk` is set, stop
and propose escalating the whole loop to Fable:

- real architectural decision (new module boundaries, data model, public API)
- niche domain or obscure platform knowledge
- UI/design direction from scratch
- second failed attempt on the same bug

Escalating means Fable becomes the session model, not a subagent: save the
exploration notes and draft plan gathered so far to the plan file, then tell
the user to relaunch `/claudex-build` with the same arguments after switching
this thread to Fable via `/model fable` — or in a fresh Fable thread. The
relaunched run finds the existing plan file and continues from it. If the
user declines, keep planning with the current model.

## Phase 2 — Plan critique (Codex, read-only)

Skip with `--fast`. Use `/codex:adversarial-review` pointed at the plan file
(fallback: read-only `codex exec`), with this stance:

```text
Read docs/plans/<file>.md. Do not implement anything.
Challenge the plan: edge cases it ignores, assumptions about this codebase
that are wrong (verify against the actual code), steps too vague to implement
mechanically, missing verification. List only concrete problems with file
references. Do not rewrite the plan.
```

Fold valid findings into the plan file; note rejected ones with a one-line
reason at the bottom.

## Checkpoint A

Unless `--auto`: show the plan summary, what the critique changed, the branch
decision, and any Fable proposal. Wait for approval before implementing.

## Phase 3 — Implement (Codex)

Skip to direct implementation by Claude only with `--claude-impl`.

Delegate via `/codex:rescue` (fallback: `codex exec -s workspace-write`) with
a self-contained prompt: goal and acceptance criteria from the plan, files in
scope and files not to touch, preserve unrelated changes, **do not commit,
push, deploy, or edit global config**, run the plan's verification commands,
end with a concise report (files changed, verification, open questions).

Then Claude:

1. Inspects `git status` and the diff; runs the project's check commands
   (typecheck, lint, focused tests — not build or dev servers).
2. **Commits Codex's code as-is** — the measurable baseline — staging only the
   files Codex touched. If Codex modified a file that was already dirty before
   the run, flag it: metrics for that file are polluted.
3. Does not fix anything yet; fixes belong to later commits.

## Phase 4 — Dual review, in parallel

Start (b) in the background, do (a) meanwhile, then collect (b).

- **(a) Claude main — primary reviewer** (it is not the author): read the full
  diff for bugs, taste, and simplification opportunities. Judge against the
  plan's acceptance criteria and the project conventions.
- **(b) Codex background — edge-case grinder**: `/codex:review --background`
  on the diff (fallback: read-only `codex exec` via background Bash), with a
  coverage-first instruction:

```text
Report every issue you find, including low-confidence and low-severity ones —
a separate verification step filters them. For each finding: severity,
file:line, concrete failure mode, fix direction. Do not edit files.
```

Declare in the report that leg (b) is the same model as the author in a fresh
session — a grinder, not an independent judge.

With `--high-risk` (or on request): add `/codex:adversarial-review` on the
diff (challenges design choices) and a Fable pass scoped to architecture and
simplification only — bugs were already hunted. If the session already runs
on Fable (post-escalation), fold this pass into leg (a) instead of spawning
a subagent; otherwise use a read-only Fable subagent (Task/Agent with
`model: fable`; if that model is unavailable, say so and skip the pass).

## Phase 5 — Triage and fix (Claude decides, fixes are routed)

For each finding, read the cited code and classify: real, false positive
(discard with reason), out of scope (note it). Codex output is evidence, not
authority.

Route the fixes:

- **Mechanical bug fixes** → back to Codex (`/codex:rescue` or write-leg
  `exec`) with the finding list; Claude approves the resulting diff.
- **Taste and simplification fixes** → Claude applies them directly.

All fixes go in **separate commits** from the baseline — this is what makes
rework measurable. Re-run checks. If fixes were substantial, one more quick
Codex pass on the new diff; maximum two review rounds total.

## Phase 6 — Report and metrics log

1. Final message: what shipped, findings fixed / discarded (with reasons),
   verification commands and results, who implemented and who fixed what,
   anything left open.
2. Append one row to `docs/reviews.md` (create with header if missing):

```markdown
| date | feature | impl lines (codex) | rework lines (claude) | rework % | bug findings | taste findings | false positives | notes |
```

   - impl lines: `git diff --stat` of the baseline commit
   - rework lines: combined stats of the fix commits touching the same files
   - rework % = rework / impl
   - note polluted files (pre-dirty) and any `--claude-impl` runs in notes
3. Do not push or open a PR unless the user asked.

## Rules

- The author never reviews its own code. The one declared exception is leg
  (b): Codex grinding over Codex-authored code, fresh session, labeled as such.
- Codex is read-only in every review leg; write access only in implementation
  and mechanical-fix legs.
- Preserve unrelated user changes at every step; never sweep pre-existing
  dirty files into the loop's commits.
- Verify Codex's claims against the code before acting on them or relaying
  them to the user.
