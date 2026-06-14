# Mechanisms

## 1. BMAD installation

Every target project must be BMAD-native. BMAD owns workflows and artifacts.

## 2. BMAD Help as navigator

Hera asks `bmad-help` instead of guessing the next step.

`bmad-help` inspects:

- installed modules;
- workflow catalog;
- config;
- existing artifacts;
- project state.

It recommends the next relevant workflow.

## 3. Coding-tool adapters

Adapters run BMAD skills through external coding tools.

Common adapter contract:

```python
run_bmad_skill(repo, skill_name, prompt=None) -> ToolRunResult
```

Tool-specific details remain inside adapters.

## 4. Hera acceptance gate

Hera verifies actual outputs before accepting work.

Checks:

- diff;
- tests/lint;
- artifact updates;
- forbidden operations;
- scope;
- safety.

## 5. Worktree isolation

For non-trivial code changes, the bridge should create a git worktree and run the coding tool there.

## 6. Run ledger

Every run creates evidence files under `.hera/bmad-runs/`.

## 7. BMAD review workflows

For review, prefer BMAD workflows/tools:

- `bmad-code-review`;
- `bmad-review-adversarial-general`;
- `bmad-review-edge-case-hunter`.

Hera triages findings and decides whether to fix, defer, reject, or ask Vova.
