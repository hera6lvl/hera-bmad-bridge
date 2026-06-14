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

## 7. BMAD review and quality workflows

For review and quality strategy, prefer BMAD workflows/tools:

- `bmad-code-review`;
- `bmad-review-adversarial-general`;
- `bmad-review-edge-case-hunter`;
- TEA workflows for risk-based test strategy, traceability, NFR review, ATDD, automation, CI, and test review.

Hera triages findings and decides whether to fix, defer, reject, or ask Vova.

## 8. Creative and automation modules

CIS is available for brainstorming, design thinking, storytelling, innovation strategy, and creative problem solving.

Automator is available as an experimental BMAD module for story automation. It should be treated as a BMAD workflow executor aid, not as a replacement for Hera's acceptance gate.
