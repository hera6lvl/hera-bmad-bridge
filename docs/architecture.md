# Architecture: Hera BMAD Bridge

## Roles

```text
BMAD = process, workflows, artifacts, navigation
bmad-help = project-state navigator and next-step recommender
Hera = operator, orchestrator, safety/acceptance gate
Coding tools = interchangeable workflow executors
Project = BMAD-native repo
```

## Runtime loop

```text
1. Hera opens a BMAD-native project.
2. Hera runs `bmad-help` through a selected tool.
3. BMAD Help recommends the next required/optional workflow.
4. Hera evaluates the recommendation against safety, mission, and user intent.
5. Hera runs the selected BMAD workflow through a coding-tool adapter.
6. Adapter returns transcript/logs and changed files.
7. Hera independently verifies the result.
8. Hera runs `bmad-help` again to continue.
```

## Bridge boundary

The bridge is intentionally thin. It does not understand product strategy better than BMAD, and it does not replace BMAD workflows.

It only standardizes how Hera invokes BMAD through different tools.

## Main components

### 1. BMAD Project

A repo with:

```text
_bmad/
_bmad-output/
.claude/skills/bmad-*
```

Future tools may install their own integration folders, but the source of truth remains BMAD artifacts.

### 2. Bridge command/tool

Proposed interface:

```text
bmad_bridge help --repo <path> --tool <auto|claude|codex|antigravity|gemini>
bmad_bridge run --repo <path> --workflow <bmad-skill> --tool <...>
bmad_bridge collect --repo <path> --run <id>
bmad_bridge verify --repo <path> --run <id>
```

Hermes may expose the same functions as tools later:

```python
bmad_bridge(repo, action="help|run|collect|verify", workflow=None, tool="auto")
```

### 3. Tool adapters

Each adapter implements one operation:

```python
run_bmad_skill(repo, skill_name, prompt=None) -> ToolRunResult
```

Adapters hide tool-specific invocation details:

- Claude Code: BMAD skills in `.claude/skills`, print/interactive mode depending on workflow.
- Codex: run equivalent instruction in `codex exec`, consuming BMAD project files.
- Gemini: same contract if authenticated and capable.
- Antigravity: only first-class if it has reliable automation/CLI; otherwise human-mediated.

### 4. Hera acceptance gate

The tool's self-report is never enough.

Hera verifies:

- git diff;
- changed artifacts;
- changed code;
- test/lint output when applicable;
- no forbidden operations;
- no credential/security/deploy changes without approval;
- scope matches the BMAD workflow/story/spec.

### 5. Run ledger

Every run should write an audit folder:

```text
.hera/bmad-runs/<timestamp-slug>/
  request.yaml
  tool.log
  result.yaml
  diff.patch
  verification.log
```

This makes agent behavior inspectable and reviewable.

## Safety model

Default restrictions:

- No `git push` unless Vova explicitly asks.
- No deploys.
- No public posting beyond requested GitHub repo work.
- No credential/security setting changes.
- Non-trivial code changes should use git worktrees.
- Hera may stop or reroute a workflow if the tool drifts.

## Success criteria

The project is useful when Hera can:

1. Ask BMAD Help in a repo.
2. Run BMAD workflows through at least one coding tool.
3. Collect normalized run evidence.
4. Verify the result independently.
5. Repeat the loop without manually inventing next steps.
