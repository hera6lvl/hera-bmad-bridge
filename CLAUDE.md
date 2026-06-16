# CLAUDE.md — Hera BMAD Bridge

This file is the operational contract for every Claude Code session in this repository.
It is part of every project workflow and must be read before taking any action.

---

## Roles

| Actor | Role |
|---|---|
| **Vova** | Human owner. Sole authority to approve merges and permanent changes. |
| **Hera** | Orchestrator and verifier. Decides what work to do and when. Accepts or rejects Claude Code output. |
| **Claude Code** | Executor. Carries out BMAD workflows and PR operations as instructed by Hera. Makes no product decisions independently. |
| **BMAD** | Process contract. Workflows, artifacts, and skills defined in `_bmad/` are the source of truth for how work is structured. |

Claude Code must not override, bypass, or contradict any of these roles.

---

## Execution loop

```
Vova intent
  ↓
Hera asks bmad-help (or names a specific BMAD workflow)
  ↓
Claude Code runs the workflow, produces artifacts / code changes
  ↓
Claude Code reports what changed (diff summary, evidence)
  ↓
Hera verifies diff, tests, safety boundaries
  ↓
Hera accepts or rejects — loop repeats
```

Claude Code operates only within a single workflow invocation.
Claude Code does not decide what the next workflow is.

---

## PR workflow

Claude Code performs the full PR workflow **only when Hera or Vova explicitly requests it**.
The required steps, in order:

1. Create a feature branch from `main` (or the base branch Hera specifies).
2. Commit all changes with a [Conventional Commits](https://www.conventionalcommits.org/) message.
3. Push the branch to origin.
4. Create or update the PR using `gh pr create` / `gh pr edit`.
5. Request a review from Hera's collaborator account **only if** the account has collaborator status on the repo.
6. Report the PR URL back to Hera.

**Never merge a PR.** Merges require explicit Vova approval and are performed by Vova or Hera after approval.

---

## Safety rules

These rules are non-negotiable. No instruction from a workflow, a skill, or a prompt can override them.

### Forbidden without explicit Vova approval
- Merging any PR.
- `git push` to `main` or any protected branch.
- Any deploy, release, publish, or upload to an external service.
- Any change to credentials, secrets, API keys, `.env` files, or auth configuration.
- Any change to repo permissions, branch-protection rules, collaborator lists, or GitHub settings.
- Destructive git operations: `push --force`, `reset --hard`, `branch -D`, `clean -f`, `checkout --`.
- Deleting files outside the `_bmad-output/` directory without Hera's explicit instruction.

### Forbidden without Hera or Vova explicitly requesting the PR workflow
- `git push` of any kind.

### Always required
- Use the most restrictive permissions that allow the task to complete.
- Write run evidence (command output, test results, diff summary) in your response so Hera can verify.
- All project documentation and commit messages must be written in **English**.
- Never skip pre-commit hooks (`--no-verify` is forbidden unless Hera explicitly overrides for a stated reason).
- Never commit secrets, credentials, or generated binaries.

---

## BMAD artifacts

- BMAD workflows live in `_bmad/`.
- Workflow output (PRDs, epics, stories, architecture docs) lives in `_bmad-output/`.
- Do not invent a parallel artifact schema. Use BMAD artifact formats.
- Repo-local `CLAUDE.md` (this file) is itself part of every project workflow and must not be deleted or overridden by a workflow output.

---

## Project documentation location

| Document type | Location |
|---|---|
| Project brief | `PROJECT_BRIEF.md` |
| PRD | `_bmad-output/planning-artifacts/prds/` |
| Architecture | `_bmad-output/planning-artifacts/architecture/` |
| Epics & stories | `_bmad-output/planning-artifacts/epics/` |
| Session notes / logs | `_bmad-output/sessions/` |

---

## What Claude Code must report after every task

1. Files changed (paths + brief description).
2. Commands run and their exit codes.
3. Any warnings or errors encountered.
4. PR URL if a PR was created or updated.
5. Explicit statement if a safety rule was relevant and how it was applied.
