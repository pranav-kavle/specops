# specops

Spec-Driven Development for Claude Code — an anchor plugin that keeps Claude faithful to a written spec through an entire feature's build cycle, the way [`superpowers`](https://github.com/obra/superpowers) anchors Claude to TDD.

## The problem

When Claude implements a feature, it drifts: jumping to code before requirements are clear, quietly adding things nobody asked for, forgetting constraints stated earlier, or reinterpreting scope mid-build. No native Claude Code plugin anchors Claude to spec-driven development the way `superpowers` anchors it to test-driven development — Spec Kit is a CLI-first, cross-agent tool rather than a Claude Code plugin, and Feature-Dev has SDD-like phases without being a dedicated SDD anchor.

`specops` treats the spec as the source of truth for the duration of a feature's build, with a dedicated mechanism for each point where drift actually happens: writing a good spec, staying anchored while implementing, and making sure each phase's artifact faithfully derives from the one before it.

## The loop

```
writing-the-spec  →  planning-to-spec  →  decomposing-into-tasks  →  auditing-spec-fidelity  →  implementing-with-fidelity
(+ playbook.md,          plan.md              tasks.md                 (hard gate)                  code
 once, optional)
```

| Skill | Slash command | Artifact |
|---|---|---|
| `running-specops` | *(auto-loaded)* | — routes to the right phase below |
| `writing-the-spec` | `/specops:specify` | `specs/<feature>/spec.md` (+ `specs/playbook.md` on first use) |
| `planning-to-spec` | `/specops:plan` | `specs/<feature>/plan.md` |
| `decomposing-into-tasks` | `/specops:tasks` | `specs/<feature>/tasks.md` |
| `auditing-spec-fidelity` | `/specops:audit` | audit report (hard-blocks on objective gaps) |
| `implementing-with-fidelity` | `/specops:implement` | working code |

Full design rationale: [`docs/design/2026-07-18-specops-design.md`](docs/design/2026-07-18-specops-design.md).

## Status

Design complete. Skills are scaffolded (names + descriptions only) but not yet implemented.

## Installation

Once implemented:

```
/plugin marketplace add pranav-kavle/specops
/plugin install specops@specops
```

## License

MIT
