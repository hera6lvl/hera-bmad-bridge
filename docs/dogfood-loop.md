# Dogfood Loop

This project must use the operating model it is building.

## Correct loop

```text
Hera → bmad-help → recommended BMAD workflow → coding tool executes workflow → Hera verifies → bmad-help again
```

## What Hera must not do

- Do not invent a parallel project plan when BMAD is installed.
- Do not write custom schemas instead of using BMAD artifacts.
- Do not decide the next workflow from memory when `bmad-help` can be asked.
- Do not treat Claude/Codex/Antigravity as project-specific dependencies.

## Bootstrap exception

Before the bridge can run BMAD workflows automatically, Hera may manually operate the loop:

1. Open the repo.
2. Ask the selected coding tool to invoke `bmad-help`.
3. Read the recommendation.
4. Ask the same or another tool to run the recommended BMAD skill.
5. Verify outputs.

The bootstrap exception ends as soon as `bmad_bridge help` and `bmad_bridge run` exist.

## First BMAD prompt

Use this as the first operator prompt inside a BMAD-aware coding tool:

```text
Invoke bmad-help for this BMAD-native project.

Project goal:
Build a thin Hera BMAD Bridge so Hera can operate BMAD-native projects by asking bmad-help, running recommended BMAD workflows through interchangeable coding tools, collecting results, verifying outputs, and looping back to bmad-help.

Important constraints:
- Do not replace BMAD Help.
- Do not invent a parallel planner.
- Use BMAD workflows and artifacts as the process/source of truth.
- Hera is the operator and acceptance gate.
- Coding tools are workflow executors.

Tell us the next recommended BMAD workflow and why.
```

## Review expectation

After every workflow run, Hera should report:

- what BMAD recommended;
- what workflow ran;
- which tool executed it;
- changed files/artifacts;
- verification results;
- next BMAD recommendation.
