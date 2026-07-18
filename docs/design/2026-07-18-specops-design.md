# SpecOps — a Spec-Driven Development plugin for Claude Code

**Date:** 2026-07-18
**Status:** Design (pre-implementation)

## Problem

When Claude implements a feature, it tends to drift from the original intent — jumping to code before requirements are clear, quietly adding things nobody asked for, forgetting constraints stated earlier, or reinterpreting scope as it goes. `superpowers` solves the analogous problem for testing by anchoring Claude to TDD. No comparable native Claude Code plugin exists for Spec-Driven Development (SDD): Spec Kit is a CLI-first, 30+-agent tool rather than a Claude Code plugin, and Feature-Dev has SDD-like phases without being a dedicated SDD anchor.

Drift shows up at three points, roughly equally:

1. **Creating a good spec** — Claude jumps to code before requirements are pinned down.
2. **Staying anchored during implementation** — Claude drifts across the many tool calls of a long build.
3. **Phase transitions** — a spec exists, but the next artifact doesn't faithfully derive from it.

`specops` treats the spec as the source of truth for the duration of a feature's build, and gives each of these three failure points its own dedicated mechanism rather than a single bolted-on reminder.

## Goals

- Anchor Claude to a spec through an entire feature's build cycle: Specify → Plan → Tasks → Implement, each phase producing an artifact the next phase consumes.
- Work for both solo developers and small teams — artifacts are plain, visible files that double as a shared source of truth in code review, not a personal scratchpad.
- Be installable and usable standalone, with no dependency on `superpowers` or any other plugin.

## Non-goals

- **Not living documentation.** Artifacts (`spec.md`, `plan.md`, `tasks.md`, `playbook.md`) are point-in-time build artifacts, analogous to a `superpowers` plan doc — authoritative during the build they govern, not a permanently maintained reference. If a feature changes later outside `specops`, the spec may go stale; that's an accepted tradeoff, not a defect. A follow-up change gets its own spec/revision cycle rather than an obligation to keep old files eternally in sync with reality.
- **Not a dependency on `superpowers`.** `writing-the-spec` reimplements a Socratic-questioning discipline directly rather than wrapping `superpowers:brainstorming`, so the plugin has zero peer-plugin requirements.
- **Not a general development-methodology framework.** `specops` is scoped to the SDD lifecycle specifically, not debugging, code review, or git workflow (the way `superpowers` covers many disciplines).
- **Not multi-agent-tool agnostic.** Built for Claude Code specifically, not a cross-agent CLI like Spec Kit.

## Skill architecture

Seven skills, chained through artifacts rather than one monolithic skill — each phase transition becomes a checkable "does skill N correctly read skill N-1's artifact" problem instead of an implicit one.

| Skill | Slash command | Artifact | Gate behavior |
|---|---|---|---|
| `running-specops` | *(none — auto-loaded)* | — | Meta/bootstrap skill; routes to the correct phase skill |
| `writing-the-playbook` | `/specops:playbook` | `playbook.md` | Soft nudge |
| `writing-the-spec` | `/specops:specify` | `spec.md` | — |
| `planning-to-spec` | `/specops:plan` | `plan.md` | — |
| `decomposing-into-tasks` | `/specops:tasks` | `tasks.md` | — |
| `auditing-spec-fidelity` | `/specops:audit` | *(audit report)* | Hard block (objective gaps only) |
| `implementing-with-fidelity` | `/specops:implement` | *(code)* | — |

### `running-specops` (meta skill)

Force-loaded via a SessionStart hook, mirroring how `superpowers:using-superpowers` is injected at the start of every session rather than left to Claude's discretion to notice. Establishes the mandate ("if a `specops` skill applies, invoke it — not optional") and routes to the correct next phase by checking which artifacts already exist for the current feature:

- No `specs/playbook.md` → suggest `writing-the-playbook` (soft nudge, not blocking)
- No `spec.md` for this feature → `writing-the-spec`
- `spec.md` exists, no `plan.md` → `planning-to-spec`
- `plan.md` exists, no `tasks.md` → `decomposing-into-tasks`
- `tasks.md` exists, not yet audited → `auditing-spec-fidelity`
- Audited clean → `implementing-with-fidelity`

### `writing-the-playbook`

One-time (or rarely revisited) project-level setup. Captures durable rules every spec/plan/task must honor: coding standards, dependency policy, architectural constraints, testing requirements. **Soft nudge, not a hard gate** — `running-specops` suggests creating one if it's missing, but a solo/prototype user can decline and proceed straight to `writing-the-spec`. A hard gate here would recreate the "jumps to code before requirements are clear" problem, just relocated to a different artifact, for users who don't yet have team-wide conventions to declare.

### `writing-the-spec`

Standalone Socratic questioning (one question at a time, propose 2-3 approaches, multiple choice preferred) — the same discipline `superpowers:brainstorming` uses, reimplemented directly rather than depended on. Produces `spec.md`:

- Summary
- Goals / Non-goals
- Requirements — each a stable, numbered `REQ-N` (acceptance criteria)
- Constraints (referencing `playbook.md` rules where relevant)
- Open questions (anything explicitly deferred)

### `planning-to-spec`

Reads `spec.md` and `playbook.md`, produces `plan.md`:

- Approach summary
- Architecture/design decisions, each referencing which `REQ-N`(s) it addresses
- Alternatives considered (where a real tradeoff existed)
- Risks / unknowns

### `decomposing-into-tasks`

Reads `plan.md`, produces `tasks.md` — a checklist of discrete, independently-executable units of work. Each task carries:

- A stable task ID
- The `REQ-N`(s) it satisfies
- `depends-on: [task IDs]` — explicit dependency edges on other tasks in this file

The dependency edges are what let `implementing-with-fidelity` batch tasks safely (see below) instead of running everything either fully sequentially or fully in parallel.

### `auditing-spec-fidelity`

Cross-artifact consistency check, run before implementation starts. **Hard block**, but narrowly scoped to objective, mechanically-checkable failures:

- **Coverage**: does every `REQ-N` in `spec.md` appear referenced in `plan.md`, and tagged on at least one task in `tasks.md`?
- **Contradiction**: does any plan decision or task violate an explicit `playbook.md` rule?

Subjective judgment calls (e.g. "is this the best architecture?") are explicitly out of scope for this gate — it exists to catch drift and scope-narrowing mechanically, not to second-guess design decisions. This is deliberately a hard block despite `writing-the-playbook` being a soft nudge: the audit is the one checkpoint that actually catches pain points B and C (drift, silently-narrowed scope) before wasted implementation work happens, and an advisory-only version is exactly the kind of check that gets waved away under time pressure.

### `implementing-with-fidelity`

The core anchor for pain point B (drift during implementation):

1. **Dependency-aware batched dispatch.** Compute which tasks are currently unblocked (all `depends-on` entries complete), dispatch all of them as parallel fresh subagents, wait for the batch, recompute the next unblocked batch, repeat. Each subagent's context is bounded to `spec.md` + `plan.md` + its single task — no accumulated conversation history to drift from, and no risk of two dependent tasks running out of order (ruled out a fully-sequential model as too slow, and a fully-parallel/no-dependency-tracking model as reintroducing the exact drift/file-conflict risk this plugin exists to prevent).
2. **Final verification gate.** Once all tasks complete, a verification pass diffs the finished implementation against `spec.md`, `plan.md`, and `tasks.md` to catch cross-task drift that no single task's subagent could see (e.g. task 3's subagent doesn't know task 1's subagent quietly added an extra field).

## Revision protocol

The pipeline is linear (`specify → plan → tasks → audit → implement`), but reality isn't — a downstream skill may discover the upstream artifact was ambiguous, incomplete, or self-contradictory. When that happens, the skill **stops and reports** exactly what's wrong and which upstream artifact needs revision, then waits for the user to either fix it (re-run the earlier skill) or explicitly instruct Claude to proceed anyway. No silent self-healing, and no automatic backward re-invocation of the upstream skill. A human has to bless any change to a source-of-truth artifact — the same principle that requires human sign-off on code review elsewhere. Letting Claude unilaterally patch the spec whenever inconvenient would turn the spec back into something Claude can quietly rewrite mid-task, which is the exact failure mode this plugin exists to prevent.

## Artifact file layout

Flat and visible — no hidden dotfolder, no numbered-folder bookkeeping:

```
specs/
├── playbook.md
└── <feature-name>/
    ├── spec.md
    ├── plan.md
    └── tasks.md
```

Chosen over a hidden `.specops/` dotfolder (less discoverable to teammates who don't know to look there) and numbered `specs/001-<feature>/` folders (bookkeeping overhead — "what's the next number?" — that mainly earns its keep at a scale of many parallel contributors that a solo dev or small team doesn't have yet).

## Plugin distribution

- **Name:** `specops`
- **Repo:** `github.com/pranav-kavle/specops` (created, public, empty)
- **Structure** (single-plugin repo, mirroring `superpowers`'s flat layout):
  ```
  specops/
  ├── .claude-plugin/
  │   ├── plugin.json
  │   └── marketplace.json
  └── skills/
      ├── running-specops/SKILL.md
      ├── writing-the-playbook/SKILL.md
      ├── writing-the-spec/SKILL.md
      ├── planning-to-spec/SKILL.md
      ├── decomposing-into-tasks/SKILL.md
      ├── auditing-spec-fidelity/SKILL.md
      └── implementing-with-fidelity/SKILL.md
  ```
- **Publishing path:** self-hosted marketplace first (`/plugin marketplace add pranav-kavle/specops`, works immediately, no approval needed) → community marketplace submission (`anthropics/claude-plugins-community`, open submission form) once stable → official marketplace (`anthropics/claude-plugins-official`) is Anthropic-curated only and requires a partner contact, not something submitted directly.
- **Versioning:** explicit semver in `plugin.json` once a stable release is cut; omit `version` during active pre-release development so every commit is picked up.

## Open items for the implementation plan

These are deliberately left for `writing-plans` rather than settled here:

- Exact `SKILL.md` content/wording for each of the 7 skills
- The `SessionStart` hook script for `running-specops`
- Eval/acceptance-test harness to verify the skills actually behave as designed (`superpowers` has its own eval harness for this — worth reusing the pattern, not the dependency)
