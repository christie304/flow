---
name: drift
description: Flow subcommand. Compares a ticket's spec passes, decision log, test documents and acceptance-criteria checklist for drift with the flow:drift-checker agent and writes drift.md. Compares the passes you list, or the latest against the one before it. Run as /flow:drift {ticket} [p03 p04 … | --all].
argument-hint: "{ticket} [pass … | --all]"
disable-model-invocation: true
---

# /flow:drift — what has stopped agreeing

Arguments: `$ARGUMENTS` — the ticket, then optionally pass names or `--all`.

Drift is any place where a later document, or a decision, says something an
earlier document no longer matches: an AC that was dropped but still reads
as required, a decision the spec never picked up, tests built from an older
spec. This command finds it and reports it. It never fixes anything and
never blocks.

**Paths.** `${CLAUDE_PLUGIN_ROOT}` is the flow plugin root — the flow
reference files call it `{PLUGIN_ROOT}` — and
`${CLAUDE_PLUGIN_ROOT}/skills/run` is the run skill directory they
call `{SKILL_DIR}`. Honour
`${CLAUDE_PLUGIN_ROOT}/skills/run/references/context-boundaries.md`.

**1. Pre-flight** in subcommand mode per
`${CLAUDE_PLUGIN_ROOT}/skills/run/references/preflight.md`, with
`invocation: "drift"`: config, standards, memory, `run.start`, then
`CARD_ROOT` / `SPEC_REPO` and `PASS_DIR` = the latest existing pass. No new
pass folder. **Skip step 7 (source mode) entirely**: this command reads no
card, so neither MCP availability nor ticket-key validation gates it.

If `{CARD_ROOT}` has no pass folder, stop with:
`⚠️  No passes for {ticket} under {CARD_ROOT} — run /flow:run {ticket} or /flow:spec {ticket} first.`

**2. Choose the passes.** From the arguments after the ticket:

| Arguments | Passes compared |
|-----------|-----------------|
| none | the latest pass against the one before it; if only one pass exists, that pass alone (it is still checked against the decisions and its own documents) |
| `p03 p04 …` (two or more names) | those passes, in the order of their numbers, each against the next |
| one name, e.g. `p02` | that pass against the one before it; `p01` has none, so it is checked alone |
| `--all` | every pass, each against the next |

A name with no folder under `{CARD_ROOT}` → stop and list the passes that
exist. The report always goes to the **latest** pass's folder
(`{DRIFT_PATH}`, `{SPEC_ROOT}/drift.md`), whichever passes were compared; its
header names them.

**3. Check.** Dispatch the `flow:drift-checker` agent (Agent tool,
`subagent_type: "flow:drift-checker"`) with the block below as the prompt,
every `{token}` resolved to an absolute value. Its final message is its JSON
payload.

```
drift-checker \
  --card-root {CARD_ROOT} \
  --ticket {ticket} \
  --passes {comma-separated pass names, oldest first} \
  --decision-log {DECISION_PATH} \
  --repo {consuming codebase repo} \
  --write {DRIFT_PATH}
```

Omit `--decision-log` when `{DECISION_PATH}` does not exist, and `--repo`
when the command is not run inside a consuming codebase. The agent lists
each skipped check in its payload.

**4. Show** the `drift` level and the tally, then each finding as one line:
`{id} {severity} — {where}: {what}`, high first, and the report path. For
`significant` drift, end with the next step the findings point at: usually
`/flow:spec {ticket}` to bring the spec up to date, then `/flow:verify {ticket}`.
With `none`, say so in one line.

Nothing here edits the decision log. If a finding deserves a decision, the
developer records it with `/flow:decisions {ticket}`.

Telemetry follows
`${CLAUDE_PLUGIN_ROOT}/skills/run/references/emission-map.md` →
"Subcommand runs": one `phase.start` / `phase.end` pair, `phase.drift.check`
(agent `drift-checker`), around the agent dispatch with
`{drift, high, medium, low, passes_compared_count}` in `outputs`, an
`artifact` (kind: `drift_report`) for `{DRIFT_PATH}`, then `run.end` (`failed`
if the agent returned `ok: false`), then the final `.spec` publish in
`${CLAUDE_PLUGIN_ROOT}/skills/run/references/telemetry-schema.md` →
"Publish mechanics" (`--path {CARD_ROOT}`). Show the ⚠️ line if it comes
back `pending: true`.
