---
name: code
description: Specflow subcommand. Implements the latest pass's spec.md in the consuming codebase with the specflow:implementer agent (full mode), or fixes a named list of ACs (targeted mode), and writes manifest.json and manifest.md. No review, verification or commit. Run as /specflow:code {ticket} [--target-acs N,N].
argument-hint: "{ticket} [--target-acs {N,N}]"
disable-model-invocation: true
---

# /specflow:code — implement the latest pass's spec

The arguments are `$ARGUMENTS`: a ticket, optionally followed by
`--target-acs {comma-separated AC numbers}`. With `--target-acs` the
agent runs in targeted mode against those ACs; without it, full mode.

**Paths.** `${CLAUDE_PLUGIN_ROOT}` is the specflow plugin root — the specflow
reference files call it `{PLUGIN_ROOT}` — and
`${CLAUDE_PLUGIN_ROOT}/skills/run` is the run skill directory they
call `{SKILL_DIR}`. Honour
`${CLAUDE_PLUGIN_ROOT}/skills/run/references/context-boundaries.md`.

**1. Pre-flight.** Read
`${CLAUDE_PLUGIN_ROOT}/skills/run/references/preflight.md` and run it in
**subcommand mode** with `invocation: "code"`: config,
standards, memory, `run.start`, source mode, then `SPEC_ROOT`,
`CARD_ROOT` and `PASS_DIR` = the latest existing pass. No new pass
folder.

If `{SPEC_PATH}` does not exist, stop with:
`⚠️  No spec.md for {ticket} in {PASS_DIR} — run /specflow:run {ticket} or /specflow:spec {ticket} first.`

If the consuming repo is on its base branch rather than the ticket's
feature branch, stop and say so — this subcommand never creates or
switches branches. `/specflow:run {ticket}` does that through
`specflow:branch-manager`.

**2. Implement.** Dispatch the `specflow:implementer` agent (Agent
tool, `subagent_type: "specflow:implementer"`) with the block below as
the prompt, every `{token}` resolved to an absolute value. Its final
message is its JSON payload.

Full mode (no `--target-acs`):

```
implementer \
  --spec {SPEC_PATH} \
  --standards {consuming repo}/.specflow/{board-key}-standards.md \
  --mode full \
  --ticket {ticket} \
  --manifest {MANIFEST_PATH}
```

Targeted mode:

```
implementer \
  --spec {SPEC_PATH} \
  --standards {consuming repo}/.specflow/{board-key}-standards.md \
  --mode targeted \
  --target-acs {comma-separated AC numbers} \
  --ticket {ticket} \
  --manifest {MANIFEST_PATH}
```

If `{MANIFEST_PATH}` already exists, copy it to
`{CARD_ROOT}/{PASS_DIR}/manifest-{YYYYMMDD-HHMM}.json` first, so a re-run never
silently overwrites the previous manifest.

**3. Manifest.** On `ok: true`, render `{MANIFEST_MD_PATH}` from
`{MANIFEST_PATH}` exactly as Wave C join does in
`${CLAUDE_PLUGIN_ROOT}/skills/run/references/pipeline-build.md`.

**4. Show** the files written, the ACs completed and the ACs deferred
(with reasons). No code is committed — suggest `/specflow:review {ticket}`
and `/specflow:verify {ticket}` once the developer has reviewed and committed
it. The manifest stays uncommitted under `{CARD_ROOT}` — Specflow never
commits spec artifacts outside the full pipeline's Wave F.

Telemetry follows
`${CLAUDE_PLUGIN_ROOT}/skills/run/references/emission-map.md` →
"Subcommand runs": one `phase.start` / `phase.end` pair for
`wave.c.implementer`, then `run.end` (`failed` if the agent
returned `ok: false`), then move the run's telemetry buffer into the local telemetry file per
`${CLAUDE_PLUGIN_ROOT}/skills/run/references/telemetry-schema.md` →
"Where events go". Specflow does not commit or push anything here.
