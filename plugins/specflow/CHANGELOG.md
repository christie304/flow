# Changelog

All notable changes to Specflow are recorded here. Versions follow
informal semver: patch for prose cleanup, minor for new phases, flags or
defaults that keep prior invocations working, major for breaking renames
or removals.

## v1.0.0 — 2026-10-15 — first public release

### Added

- `/specflow:run {ticket}`, the full pipeline: pre-flight, card fetch and
  decision log, branch and spec, implementation, escalated code review,
  local commits, escalated acceptance-criteria verification, test
  documents, Playwright authoring, an opt-in code push and PR, an
  optional dev→QA handoff, and a closing run log.
- Subcommands as their own skills: `brief`, `spec`, `code`, `review`,
  `verify`, `fetch`, `test`, `playwright`, `decisions`, `datapoints`,
  `log` and `report`.
- `/specflow:report`, which reads local telemetry and renders a summary
  in the terminal, or a self-contained HTML dashboard with `--html`.
- `/specflow:datapoints`, a language-agnostic catalog of every code path
  that writes to named database tables, with a reverse lookup by file.
- Ten plugin sub-agents dispatched as `specflow:{name}`: `jira-fetch`,
  `decision-logger`, `branch-manager`, `spec-author`, `implementer`,
  `code-reviewer`, `ac-verifier`, `test-planner`, `playwright-testing`
  and `jira-commenter`.
- Guardrail hooks: a path guard that keeps spec artifacts under the spec
  root and blocks edits to the plugin cache, and a `Stop` hook that
  recovers telemetry from runs that crashed.
- A local spec root (`.spec` at the consuming repo root by default,
  configurable with `SPECFLOW_SPEC_ROOT`) for specs, decision logs and
  telemetry. Specflow never pushes spec artifacts or telemetry anywhere;
  only the code feature branch is pushed, and only when you opt in with
  `--push` or `AUTO_PUSH=true`.
