---
name: log
description: Specflow subcommand. Shows the specflow-log.md run log for a ticket (latest pass, or a named pass), or lists every run log with its status. Run as /specflow:log [ticket] [p0N].
argument-hint: "[ticket] [p0N]"
disable-model-invocation: true
---

# /specflow:log — run logs

Arguments: `$ARGUMENTS`. Also reached through the main skill as
`/specflow:run {ticket} --log` and `/specflow:run --log`.

**Paths.** `${CLAUDE_PLUGIN_ROOT}` is the specflow plugin root — the
specflow reference files call it `{PLUGIN_ROOT}` — and
`${CLAUDE_PLUGIN_ROOT}/skills/run` is the run skill directory they call
`{SKILL_DIR}`. Honour
`${CLAUDE_PLUGIN_ROOT}/skills/run/references/context-boundaries.md`.

Resolve config and `SPEC_ROOT` as pre-flight step 2 and "Spec path
resolution" in `${CLAUDE_PLUGIN_ROOT}/skills/run/references/preflight.md`
describe; nothing else from pre-flight runs. Emit `run.start`
(`invocation: "log"`) and `run.end` only, then the finalize step in
`${CLAUDE_PLUGIN_ROOT}/skills/run/references/telemetry-schema.md` →
"Where events go" — the local telemetry file is the only thing this
command writes.

## With a ticket — `/specflow:log {ticket} [p0N]`

1. Resolve the log path: `{LOG_PATH}` for the **latest** pass
   (`p{HIGHEST}` per "Pass-index resolution" in `preflight.md`) — that
   is, `{SPEC_ROOT}/features/{ticket}/p{NN}/specflow-log.md`. To read an
   earlier pass, name it: `/specflow:log {ticket} p02`.
2. If the file exists → read it and display the full contents in the
   conversation.
3. If not found → report clearly:
   ```
   ⚠️  No run log found for {ticket}.
   Expected: {LOG_PATH}
   Has /specflow:run {ticket} been run yet?
   ```

## Without a ticket — `/specflow:log`

1. Glob `{SPEC_ROOT}/features/*/p*/specflow-log.md`. When a ticket has
   several passes, key on the **latest** pass (highest `p{NN}`) for its
   summary row.
2. For each match, extract the ticket key and the `**Overall status**`
   and `**Completed**` lines from the log's summary block.
3. Display a summary list:

```
╔══ Specflow logs ════════════════════════════════════════════════╗

  PROJ-123   p03  ✅ Success    2026-04-21 14:03
  WEB-42     p01  ⚠️  Partial    2026-04-20 11:47
  OPS-7      p02  ❌ Failed     2026-04-19 09:12

  /specflow:log {ticket} to view the latest pass · add p0N for an earlier one
  /specflow:report for run statistics across every command

╚═════════════════════════════════════════════════════════════╝
```

4. If no logs are found:
   ```
   ⚠️  No specflow logs found under {SPEC_ROOT}/features/.
   Run /specflow:run {ticket} to generate one.
   ```
