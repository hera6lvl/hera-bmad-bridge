---
title: Hera BMAD Bridge
status: draft
created: 2026-06-16
updated: 2026-06-16
---

# PRD: Hera BMAD Bridge

## 0. Document Purpose

This PRD defines requirements for the Hera BMAD Bridge — a thin operational layer that lets Hera (an AI orchestrator) operate BMAD-native projects by asking `bmad-help`, delegating recommended BMAD workflows to interchangeable coding tool adapters, collecting run evidence, and independently verifying results. The primary audience is Hera (operator executing against these requirements), Vova (human owner and final safety approver), and downstream workflow owners running architecture, epics, and story-level work from this PRD. The document uses Glossary-anchored vocabulary throughout; terms defined in §3 are used verbatim in §4 FRs, §2 UJs, and §7 SMs. `[ASSUMPTION]` tags mark inferred decisions; all assumptions are indexed in §9.

---

## 1. Vision

The Hera BMAD Bridge exists to remove the manual gap between Hera and BMAD. Without it, Hera must hand-craft prompts, guess next steps, and manage tool invocations ad hoc — all work that BMAD is already designed to do. With the bridge, Hera asks `bmad-help` for the next workflow recommendation, hands the work to whichever coding tool is available, waits for the result, verifies it independently, and loops back. This makes Hera a reliable BMAD operator, not a parallel planner.

The bridge is intentionally thin. It does not understand product strategy better than BMAD, does not replace any BMAD workflow, and does not let coding tools make business-priority or safety decisions. Its entire value is standardizing *how* Hera invokes BMAD through different tools and *how* every run produces inspectable, auditable evidence.

The project is self-dogfooding: the loop being built is the loop used to build it. This means every capability that is delivered immediately reduces the manual overhead of the next delivery, and every gap in the bridge is felt directly by Hera as the operator.

---

## 2. Target User

### 2.1 Jobs To Be Done

- **Hera (AI orchestrator / primary operator)**
  - Operate a BMAD-native project without inventing a parallel project plan.
  - Ask `bmad-help` once and receive a concrete, actionable workflow recommendation.
  - Invoke a BMAD skill through a chosen coding tool without knowing tool-specific invocation details.
  - Collect normalized evidence from every run so verification is deterministic.
  - Verify independently that a coding tool's self-report matches actual diff, tests, and scope.
  - Detect and halt forbidden operations (push, deploy, credential changes) before they happen.
  - Loop back to `bmad-help` without manually tracking project state.

- **Vova (human project owner / safety approver)**
  - Review run ledger entries to understand what happened and why.
  - Approve or reject gate decisions when Hera escalates.
  - Authorize safety-sensitive actions (push, deploy, credential changes) explicitly and individually.
  - Trust that Hera will not act beyond its gate-approved scope.

### 2.2 Non-Users (v1)

- End users of projects built through the bridge (the bridge is an internal operator tool, not a user-facing product).
- Developers who want to build on the bridge as a library without Hera as the operator [ASSUMPTION: v1 targets Hera + Vova only; a public API/SDK is out of scope].

### 2.3 Key User Journeys

**UJ-1. Hera opens a new BMAD-native repo and gets oriented.**
Hera, operating a repo that has BMAD installed but no prior bridge runs, calls `bmad_bridge help --repo <path> --tool auto`. The bridge selects the available adapter (Claude Code), invokes `bmad-help` in the repo context, receives the recommendation, and returns a normalized result that tells Hera: which workflow is recommended, why, and what state the project is in. Hera reads the recommendation and decides whether to proceed, ask Vova, or defer.

**UJ-2. Hera runs one full loop iteration.**
Following a `bmad-help` recommendation to run `bmad-create-prd`, Hera calls `bmad_bridge run --repo <path> --workflow bmad-create-prd --tool claude`. The bridge creates a worktree [ASSUMPTION: for PRD creation, worktree is used], starts the adapter, streams the run, and writes a ledger entry when done. Hera calls `bmad_bridge verify --repo <path> --run <id>` and the gate checks diff, artifacts, tests, forbidden ops, and scope. Gate passes; Hera calls `bmad_bridge help` again for the next recommendation. The run ledger at `.hera/bmad-runs/<timestamp-slug>/` contains the full audit trail.

**UJ-3. Hera catches and rejects a forbidden operation.**
Mid-run, a coding tool attempts a `git push`. Claude Code's restrictive permission settings block the operation at the tool level. The gate (triggered post-run, reading `tool.log`) detects any attempted forbidden operation, marks the run `rejected`, writes the finding to `verification.log`, and escalates to Vova rather than silently proceeding. Vova reviews the ledger entry and either authorizes a retry with explicit push permission or closes the run as invalid.

---

## 3. Glossary

- **Hera** — AI orchestrator and primary operator of the bridge. Issues bridge commands, evaluates BMAD recommendations, runs the acceptance gate, and escalates to Vova for safety-sensitive decisions. Never makes business-priority or safety-approval decisions unilaterally.
- **Vova** — Human project owner and final approver. Authorizes safety-sensitive actions explicitly; reviews run ledger entries; may direct or redirect Hera at any time.
- **BMAD** — Process framework providing workflows, skills, artifacts, navigation, and project-state tracking. The authoritative source for what to do next and how.
- **bmad-help** — BMAD navigator skill. Inspects a BMAD-native repo's installed modules, workflow catalog, config, existing artifacts, and project state; recommends the next relevant workflow. Hera's primary steering input — never replaced or duplicated by the bridge.
- **Bridge** — The `bmad_bridge` command-line tool and/or Python API this project builds. Thin operational layer; does not contain product or process logic.
- **Coding Tool** — Interchangeable external workflow executor (Claude Code, Codex, Gemini, Antigravity). Runs BMAD skills on request. Does not set business priority, approve safety decisions, or own project state.
- **Adapter** — Tool-specific wrapper that hides invocation details and exposes the common `run_bmad_skill` contract. One adapter per Coding Tool.
- **Run** — A single, bounded invocation of a BMAD skill through an Adapter. Produces a ToolRunResult and a Run Ledger entry.
- **ToolRunResult** — Normalized return value from an Adapter: exit status, transcript/logs, list of changed files, error summary if any.
- **Run Ledger** — Audit folder under `.hera/bmad-runs/<timestamp-slug>/` containing `request.yaml`, `tool.log`, `result.yaml`, `diff.patch`, and `verification.log` for a single Run.
- **Acceptance Gate** — Hera's independent post-run verification step. Checks diff, changed artifacts, changed code, test/lint output, forbidden operations, scope compliance, and safety. The gate's verdict is independent of the Coding Tool's self-report.
- **Worktree** — A `git worktree` branch used to isolate a Coding Tool run from the main working tree. Cleaned up after verification completes.
- **BMAD-native repo** — A repository with `_bmad/`, `_bmad-output/`, and relevant BMAD skill integrations installed.
- **Forbidden Operation** — An operation the bridge blocks by default regardless of Coding Tool request: `git push`, any deploy, any credential or security-setting change.

---

## 4. Features

### 4.1 BMAD Help Invocation (`bmad_bridge help`)

**Description:** The bridge invokes `bmad-help` in a target BMAD-native repo through a selected Adapter and returns a normalized recommendation. Hera uses this recommendation as the sole input for deciding what workflow to run next. The bridge never interprets or overrides the recommendation; it only delivers it. Realizes UJ-1 and UJ-2. In v1, `auto` tool selection resolves to Claude Code.

**Functional Requirements:**

#### FR-1: Help command

Hera can call `bmad_bridge help --repo <path> --tool <selector>` and receive the `bmad-help` output as a structured recommendation. Tool selector accepts `auto`, `claude`, `codex`, `antigravity`, `gemini`. In v1, only `claude` and `auto` (→ claude) are implemented; others return an unsupported-adapter error.

**Consequences (testable):**
- When `--tool auto` is used and Claude Code adapter is available, the run uses the Claude Code adapter.
- When `--tool <unsupported>` is used in v1, the command exits with a non-zero status and a message naming the unsupported adapter.
- The return value includes: recommended workflow name, reasoning summary, project state summary.

#### FR-2: Recommendation normalization

The bridge normalizes `bmad-help` output into a ToolRunResult regardless of which Adapter executed the run.

**Consequences (testable):**
- ToolRunResult contains: `status` (ok/error), `recommended_workflow` (string or null), `reasoning` (string), `raw_transcript` (string), `changed_files` (list).
- When `bmad-help` recommends no workflow (project complete or blocked), `recommended_workflow` is null and `reasoning` explains why.

---

### 4.2 BMAD Workflow Execution (`bmad_bridge run`)

**Description:** The bridge executes a named BMAD workflow skill in a target repo through a selected Adapter. The bridge creates a worktree for non-trivial runs, hands off to the Adapter, collects the ToolRunResult, and triggers the Run Ledger write. The bridge does not interpret, alter, or summarize the workflow's output before handing it to the Acceptance Gate. Realizes UJ-2.

**Functional Requirements:**

#### FR-3: Run command

Hera can call `bmad_bridge run --repo <path> --workflow <bmad-skill-name> --tool <selector>` to execute a BMAD skill through the selected Adapter.

**Consequences (testable):**
- The run writes a Run Ledger entry before returning, even on failure.
- The run returns a unique `run-id` (timestamp-slug) that Hera can pass to `collect` and `verify`.
- When the skill name does not correspond to an installed BMAD skill in the repo, the run exits with an error before invoking the Adapter.

#### FR-4: Common Adapter contract

Every Adapter implements `run_bmad_skill(repo: str, skill_name: str, prompt: str | None) -> ToolRunResult`. Tool-specific invocation details are entirely inside the Adapter. `ToolRunResult` is a Python dataclass; its fields are serialized to `result.yaml` in the Run Ledger. JSON Schema for cross-tool validation is deferred to v2.

**Consequences (testable):**
- Swapping the `--tool` flag with the same `--workflow` produces the same ToolRunResult schema regardless of which Adapter ran.
- Adding a new Adapter does not require changes to `bmad_bridge run` logic.

#### FR-5: Claude Code adapter

The Claude Code Adapter invokes BMAD skills using `.claude/skills/bmad-*` in the target repo, in print/non-interactive mode where the workflow permits it.

**Consequences (testable):**
- The adapter can run skills available in the repo's `.claude/skills/` directory.
- The adapter captures stdout/stderr to `tool.log`.

#### FR-6: Worktree isolation

For non-trivial runs [ASSUMPTION: "non-trivial" means any run that writes or modifies code or implementation artifacts, not documentation-only runs], the bridge creates a git worktree before invoking the Adapter and runs the Adapter in the worktree.

**Consequences (testable):**
- A worktree is created at a deterministic path (e.g. `.hera/worktrees/<run-id>/`) before the Adapter starts.
- The worktree is removed after `verify` completes (pass or fail).
- If worktree creation fails, the run aborts with an error before invoking the Adapter.

---

### 4.3 Evidence Collection (`bmad_bridge collect`)

**Description:** The bridge collects the full run transcript, changed artifacts, changed code, and logs into a normalized Run Ledger entry. Collection is triggered automatically at the end of every `run` invocation, but Hera can also call `collect` explicitly against a run-id to re-collect or inspect. Realizes UJ-2 and UJ-3.

**Functional Requirements:**

#### FR-7: Run Ledger write

Every Run produces a Run Ledger entry at `.hera/bmad-runs/<timestamp-slug>/` containing: `request.yaml`, `tool.log`, `result.yaml`, `diff.patch`, `verification.log`.

**Consequences (testable):**
- `request.yaml` contains: run-id, repo path, workflow name, tool selector, timestamp, Hera-supplied prompt if any.
- `result.yaml` contains the ToolRunResult fields.
- `diff.patch` is a `git diff` of the working tree (or worktree) captured immediately after the Adapter finishes.
- `tool.log` contains the raw Adapter stdout/stderr.
- `verification.log` is initially empty and populated by `verify`.

#### FR-8: Collect command

Hera can call `bmad_bridge collect --repo <path> --run <id>` to re-collect evidence for a prior run.

**Consequences (testable):**
- Re-collection does not overwrite an existing `verification.log`.
- Re-collection updates `diff.patch` from the current state of the worktree (if still present) or the main working tree.

---

### 4.4 Acceptance Gate (`bmad_bridge verify`)

**Description:** The Acceptance Gate is Hera's independent verification step. It reads the Run Ledger and inspects actual outputs — git diff, changed artifacts, tests/lint, forbidden operation checks, scope compliance — without relying on the Coding Tool's self-report. The gate produces a `verification.log` with a pass/fail verdict and findings. The gate's verdict is the authoritative result of the run; the Coding Tool's reported success is never sufficient on its own. Realizes UJ-2 and UJ-3.

**Functional Requirements:**

#### FR-9: Verify command

Hera can call `bmad_bridge verify --repo <path> --run <id>` to run the Acceptance Gate against a completed run.

**Consequences (testable):**
- `verification.log` is written with a top-level `verdict: pass | fail | escalate`.
- `escalate` is used when a finding requires Vova's explicit decision (e.g., a Forbidden Operation was attempted).
- `verification.log` lists each check performed, its result, and any findings.

#### FR-10: Diff inspection

The gate reads `diff.patch` from the Run Ledger and checks that changed files are consistent with the requested workflow's expected scope.

**Consequences (testable):**
- If `diff.patch` shows changes outside the workflow's expected scope, the finding is recorded in `verification.log` as a scope-creep warning.

#### FR-11: Forbidden operation check

The gate checks `tool.log` and `diff.patch` for evidence of Forbidden Operations (git push, deploy commands, credential or security-setting changes). Detection is post-hoc: the gate reads the completed `tool.log` after the Adapter finishes. Claude Code's restrictive permission settings serve as the primary enforcement layer that prevents Forbidden Operations from completing; post-hoc gate inspection is the independent verification layer.

**Consequences (testable):**
- Detection of a Forbidden Operation sets `verdict: escalate` and names the operation in `verification.log`.
- The escalation is surfaced to Hera immediately; Hera escalates to Vova before any further action.

#### FR-12: Test and lint check

When the workflow type is expected to produce or modify code, the gate runs available test and lint commands and records results in `verification.log`. [ASSUMPTION: the set of test/lint commands is configured per-repo or derived from the repo's package manifest.]

**Consequences (testable):**
- Test/lint output (pass/fail + summary) is captured in `verification.log`.
- A test failure does not automatically fail the gate verdict; it is recorded as a finding for Hera to evaluate.

#### FR-13: Scope check

The gate verifies that changed artifacts and code match what the BMAD workflow/story/spec authorized.

**Consequences (testable):**
- The gate records which files changed and whether each was in-scope for the workflow.
- Out-of-scope changes are findings; sufficiently broad out-of-scope changes set `verdict: fail`.

---

### 4.5 Dogfood Compliance

**Description:** This project's own development must use the loop the bridge is building. Once `bmad_bridge help` and `bmad_bridge run` are functional, the next story must use them rather than manual invocation. This is a process requirement, not a code requirement — but it is enforced as a team norm between Hera and Vova.

**Functional Requirements:**

#### FR-14: Bootstrap exception boundary

The manual bootstrap loop (Hera manually invokes `bmad-help` via a coding tool, reads the result, manually runs the recommended skill) is in effect only until `bmad_bridge help` and `bmad_bridge run` both exist and pass their own acceptance gate.

**Consequences (testable):**
- After the bridge's own first end-to-end test passes, all subsequent stories in this repo are run through the bridge.
- A story that uses manual invocation after the boundary is met is recorded as a process finding in Hera's run log.

---

## 5. Non-Goals (Explicit)

- Do not reimplement or override `bmad-help` logic. The bridge calls it; it does not duplicate it.
- Do not create a custom artifact schema while BMAD artifact schemas exist.
- Do not make target projects depend on Claude-specific concepts or imports.
- Do not let Coding Tools set business priority, approve safety, or decide scope.
- Do not push to remotes, deploy, or change credentials without Vova's explicit, per-action authorization.
- Do not build a multi-project orchestration layer (v1 targets one repo at a time).
- Do not build a Hermes Python API surface in v1 [ASSUMPTION: Hermes integration is v2].
- Do not support Codex, Gemini, or Antigravity adapters in v1 (confirmed v2; approved by Vova 2026-06-16).
- Do not build a UI, dashboard, or notification system in v1.

---

## 6. MVP Scope

### 6.1 In Scope

- `bmad_bridge help` command with Claude Code adapter and `auto` selector.
- `bmad_bridge run` command with Claude Code adapter and worktree isolation for non-trivial runs.
- `bmad_bridge collect` command writing the full Run Ledger structure.
- `bmad_bridge verify` command implementing all five gate checks (diff, forbidden ops, test/lint, scope, artifact).
- Run Ledger at `.hera/bmad-runs/<timestamp-slug>/` with all five files.
- Dogfood compliance: bridge is used to develop itself once `help` and `run` are operational.

### 6.2 Out of Scope for MVP

- Codex, Gemini, Antigravity adapters. [NOTE FOR PM: Codex is the most likely v2 addition; worth designing the adapter interface to make it a 1–2 day lift.]
- Hermes Python API (`bmad_bridge(repo, action, ...)`) — v2.
- Multi-repo orchestration — v2.
- Notification/alerting surface (Slack, email, desktop) — v2.
- Web UI or dashboard — not planned.
- Automated retry on gate failure — Hera decides manually in v1.

---

## 7. Success Metrics

**Primary**

- **SM-1: Full loop completability** — Hera can complete a `help → run → collect → verify → help` loop on this repo without any manual step. Target: ≥ 1 full loop passing the gate before v1 is considered done. Validates FR-1, FR-3, FR-7, FR-9.
- **SM-2: Gate reliability** — Every Forbidden Operation inserted into a test run is caught. Target: 100% detection on the test suite. Validates FR-11.

**Secondary**

- **SM-3: Ledger completeness** — Every run, including failures, produces a complete Run Ledger with all five files. Target: 100% of runs. Validates FR-7.
- **SM-4: Adapter replaceability** — Swapping `--tool claude` for a second adapter (when available) requires no changes to the bridge core. Validates FR-4.

**Counter-metrics (do not optimize)**

- **SM-C1: Do not optimize run count as a proxy for correctness.** A high volume of runs with low gate pass rates signals drift, not productivity. Track gate pass rate alongside run count.
- **SM-C2: Do not optimize transcript length.** Shorter tool logs are not better; they may indicate the Coding Tool is truncating output. Gate verification depends on complete `tool.log`.

---

## 8. Cross-Cutting NFRs

- **Auditability:** Every Run must produce a complete Run Ledger entry. No run may exit without writing at least `request.yaml` and `tool.log`, even on partial failure.
- **Isolation:** Coding Tools run in a Worktree for non-trivial runs. The main working tree is never modified by an unverified run.
- **Interoperability:** All Adapters expose the same `run_bmad_skill` contract. Adding a new Adapter must not require changes to bridge core commands.
- **Safety default:** All Forbidden Operations are blocked by default. Overrides require per-action Vova authorization recorded in the Run Ledger.
- **Observability:** `tool.log` captures complete Adapter stdout/stderr. `verification.log` captures all gate findings with machine-readable verdict.

---

## 9. Constraints and Guardrails

**Safety**
- No `git push` unless Vova explicitly authorizes the specific push in the specific run.
- No deploys of any kind.
- No public posting beyond GitHub repo work Vova has authorized.
- No credential or security-setting changes.
- Claude Code adapter invocations use restrictive permission settings that prevent Forbidden Operations from completing at the tool level.
- Non-trivial code changes must use git worktrees.
- Hera may stop or reroute a workflow if the Coding Tool drifts from the requested workflow's scope.

**Privacy / Data**
- Run Ledger entries may contain LLM transcripts. These are stored locally in the repo under `.hera/`; no external upload unless Vova explicitly requests it. [ASSUMPTION: `.hera/` is gitignored by default to avoid committing potentially large or sensitive transcripts.]

**Cost**
- Each `bmad_bridge run` consumes Coding Tool API tokens or compute. [ASSUMPTION: v1 has no per-run cost guard; Hera and Vova monitor usage externally. A cost-cap mechanism is a v2 concern.]

---

## 10. Integration and Dependencies

- **BMAD** (BMad Core, BMad Method, BMad Builder, TEA, CIS, Automator): must be installed in the target repo. Bridge has no fallback if BMAD is absent.
- **Claude Code CLI**: required for the v1 Claude Code Adapter. [ASSUMPTION: authenticated and available in the execution environment.]
- **Git**: required for Worktree creation and `diff.patch` generation.
- **`bmad-help` skill**: must be installed in the target repo's `.claude/skills/` (or equivalent for future adapters).
- **No other runtime dependencies assumed in v1** — the bridge is a Python CLI.

---

## 11. Open Questions

1. **Non-trivial threshold**: What precise rule defines "non-trivial" for Worktree isolation? Suggested rule: any run that writes to implementation artifacts (`.py`, `.ts`, etc.) is non-trivial; documentation-only runs (`.md`, BMAD artifacts) are trivial. Vova to confirm.
2. **Test/lint command discovery**: How does the gate discover which test/lint commands to run in an arbitrary repo? Strategies: `package.json` scripts, `Makefile`, config file in `.hera/`, or a convention-based lookup.
3. **`.hera/` gitignore default**: Should the bridge write a `.gitignore` entry for `.hera/bmad-runs/` on first use, or leave it to the repo owner?
4. **Gate escalation UX**: When `verdict: escalate`, how does Hera surface this to Vova? Options: write to a known file Hera monitors, raise an exception in the calling context, or log to a channel. Needs a decision for UJ-3 to be testable.

---

## 12. Assumptions Index

- **A-1 (§0):** Project name is "Hera BMAD Bridge" / slug "hera-bmad-bridge", derived from README and PROJECT_BRIEF.
- **A-2 (§2.2):** v1 targets Hera + Vova only; a public developer API/SDK is out of scope.
- **A-3 (§2.3 UJ-1, FR-1):** `auto` tool selection in v1 resolves to Claude Code. *Confirmed by Vova, 2026-06-16.*
- **A-4 (§2.3 UJ-2, FR-6):** Worktree is used for PRD creation and all non-trivial runs; doc-only runs may skip the worktree.
- **A-5 (§4.1 FR-1):** Codex, Gemini, Antigravity adapters are not implemented in v1; they return an unsupported-adapter error. *Confirmed by Vova, 2026-06-16.*
- **A-6 (§4.2 FR-6):** "Non-trivial" = any run modifying implementation artifacts (.py, .ts, etc.); documentation-only runs are trivial.
- **A-7 (§4.4 FR-12):** Test/lint command set is configured per-repo or derived from the repo's package manifest; no bridge-level default list.
- **A-8 (§5):** Hermes Python API integration is v2; not in MVP.
- **A-9 (§5):** Codex, Gemini, Antigravity adapters are v2. *Confirmed by Vova, 2026-06-16.*
- **A-10 (§9, Privacy):** `.hera/` is gitignored by default to prevent committing large/sensitive run transcripts.
- **A-11 (§9, Cost):** No per-run cost guard in v1; cost monitoring is Hera + Vova's external responsibility.
- **A-12 (§10):** Claude Code CLI is authenticated and available in the execution environment at bridge invocation time.
- **A-13 (§10):** Bridge form factor is a Python CLI. *Confirmed by Vova, 2026-06-16.*
- **A-14 (§4.2 FR-4):** ToolRunResult is a Python dataclass; fields are serialized to YAML (`result.yaml`) in the Run Ledger. JSON Schema for cross-tool validation is deferred to v2. *Confirmed by Vova, 2026-06-16.*
- **A-15 (§4.4 FR-11, §9):** Forbidden Operation detection is post-hoc (reading `tool.log` after the run completes). Claude Code restrictive permissions serve as the primary enforcement mechanism; in-flight adapter hooks are deferred to v2. *Confirmed by Vova, 2026-06-16.*
