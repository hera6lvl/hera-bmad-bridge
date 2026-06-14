# Hera BMAD Bridge

Thin orchestration bridge for making Hera operate BMAD-native projects through interchangeable coding tools.

## Core idea

Hera does **not** replace BMAD and does **not** invent a parallel planner.

```text
Vova intent
  ↓
Hera asks bmad-help
  ↓
BMAD recommends next workflow
  ↓
Hera runs that BMAD workflow through a selected coding tool
  ↓
Claude Code / Codex / Antigravity / Gemini executes the workflow
  ↓
BMAD artifacts and/or code change
  ↓
Hera verifies diff, tests, safety boundaries
  ↓
Hera asks bmad-help again
```

## What this repo is

A BMAD-native project that defines and dogfoods the bridge.

Installed BMAD modules/tools:

- BMad Core
- BMad Method (`bmm`)
- BMad Builder (`bmb`)
- Test Architect (`tea`)
- Creative Innovation Suite (`cis`)
- BMad Automator (`automator`)
- Claude Code skill integration (`.claude/skills/bmad-*`)

## What we are building

A thin operational layer with commands/tools like:

- `bmad_bridge help` — run `bmad-help` in a project through a selected coding tool.
- `bmad_bridge run <workflow>` — run a BMAD workflow skill through a selected coding tool.
- `bmad_bridge collect` — collect transcript, changed artifacts, changed code, logs.
- `bmad_bridge verify` — Hera acceptance gate: diff, tests, forbidden ops, scope creep.

## Non-goals

- Do not reimplement BMAD Help.
- Do not create a custom artifact schema while BMAD artifacts exist.
- Do not make projects depend on Claude-specific concepts.
- Do not let coding tools decide business priority or safety approval.
- Do not push/deploy/change credentials without explicit human approval.

## Dogfood rule

This project must be developed by the same loop it is building:

1. Ask `bmad-help` what to do next.
2. Follow the recommended BMAD workflow.
3. Use a coding tool as workflow executor.
4. Hera verifies and accepts/rejects.
5. Repeat.

Until the bridge exists, Hera may manually operate the loop. Once a bridge capability exists, the next story should use it.
