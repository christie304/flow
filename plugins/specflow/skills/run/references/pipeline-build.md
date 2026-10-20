<!-- Part of the run skill. Read on demand from SKILL.md.
     {SKILL_DIR} = the run skill's base directory; {PLUGIN_ROOT} = the
     specflow plugin root. SKILL.md defines both. -->

# Full pipeline, part 1 — Waves A to C

## Full pipeline

The pipeline runs as a sequence of **waves** (A, B, C, E, F, D, G, H — the
order under "Pipeline sequence" in SKILL.md) followed by the closing phases (5c
→ 8). Within a wave, lanes have no dependency on each other and the
orchestrator MAY dispatch them concurrently (via parallel tool calls or
sub-agent spawning) when its runtime allows. Between waves there is an
explicit **join** — the next wave does not start until every lane in the
current wave has returned a status.

### Wave execution semantics

- **Every lane is a plugin agent.** Each invocation block in the
  pipeline files — `jira-fetch …`, `branch-manager …`, `code-reviewer …`
  and so on — is dispatched as the matching `specflow:{agent}` plugin agent
  through the Agent tool (`subagent_type: "specflow:{agent}"`). The prompt
  is the block itself with every `{token}` resolved to an absolute value,
  plus any payload the flag names (for example the A1 card JSON for
  `--card`). The agent sees nothing else from this conversation. Its
  final message is the JSON payload its agent file defines; parse it,
  and treat a reply that is not valid JSON as `ok: false` with
  `error_code: AGENT_BAD_PAYLOAD`.
- **Record what each dispatch spent.** When an agent returns,
  the Agent tool reports its token count, tool-call count and duration
  (a background dispatch reports them in its task notification). Copy
  them into that lane's `phase.end` or `phase.error` as `usage` (the
  Agent usage rule in `emission-map.md`) before parsing anything else, so a
  lane that fails still records its cost.
- **Parallel within a wave.** Dispatch a wave's lanes as several Agent
  tool calls in one message so they run concurrently. Orchestrator-only
  lanes (A3, joins, consolidation) run in the main conversation.
  Sequential execution (in any order) is a valid fallback and produces
  identical output because lanes within a wave are independent.
- **Join between waves.** Wait for every lane to return. Append each lane's
  status (`ok` / `error_code` / `error_message`) to `{LOG_PATH}` as it
  completes. Order in the log is insertion order, not strict wave order.
- **Failure handling.** If any lane in a wave returns `ok: false`, halt at
  the wave boundary. Successful lanes keep their outputs.
- **Atlassian MCP unavailable.** Each affected lane follows the paste
  fallback in `{SKILL_DIR}/references/credentials.md` → "Atlassian MCP fallback".
  Other lanes in the wave proceed normally.

---

### Wave A — fan-out (post pre-flight)

Three independent lanes, dispatched simultaneously after pre-flight completes.

#### A1 — jira-fetch (or no-op with staged payload)

Behavior depends on `source_mode` resolved during pre-flight step 7:

- **`source_mode: jira`** — normal path. Dispatch the
  `specflow:jira-fetch` agent (`subagent_type: "specflow:jira-fetch"`) with this prompt:
  ```
  jira-fetch {ticket} --comments 5 --epic
  ```
  On `ok: true`: extract `title`, `description`, `acceptance_criteria`,
  `issue_type`, `labels`, `parent`, `parent_card`, `comments`.
- **`source_mode: paste` or `card_file`** — A1 becomes a **no-op** that
  returns the payload already staged by pre-flight step 7. Do NOT call
  `jira-fetch` (MCP is either down or intentionally bypassed). Emit
  `phase.end` `wave.a.jira_fetch` with `{skipped: true, reason: "source_mode:{paste|card_file}"}`
  and the same payload shape jira-fetch would have returned. Missing
  fields (`parent`, `parent_card`, `comments`, `labels`, `issue_type`)
  default to reasonable empties: no parent, no comments, empty labels,
  `issue_type = "Task"`.

Either way, the resulting payload becomes the input for Wave B (B1
branch-manager, B2 spec-author). Downstream sub-agents see a consistent
contract regardless of source.

#### A2 — decision-logger (only if `--decisions` was passed)

Run the **Decision Log subprocess** (`{PLUGIN_ROOT}/skills/decisions/SKILL.md`,
Steps 1–4) in **pre-card mode** — A2 runs in
parallel with A1 and therefore has no card payload yet. The decision log
header is written with `Jira URL: [NEEDS CLARIFICATION] — pre-card mode`
and rows are appended from the resolved `--decisions` content.

After the Wave A join, if A1 succeeded, run a one-time **header sync**:
re-open `{DECISION_PATH}`, replace the pre-card placeholders with the
card title and Jira URL from A1, leave rows untouched.

If `--decisions` was not passed, A2 is a no-op (the blank decision log
scaffolded during pre-flight stays in place).

#### A3 — settings.local.json

Write `BASE_BRANCH={base-branch}` and `DIFF_BRANCH={diff-branch}` to
`.claude/settings.local.json`. This is independent of git — a plain
config write — and is safe to do before the feature branch is checked out.
The write target is `.claude/settings.local.json` at the consuming repo's
root — the standard Claude Code per-repo overrides file, which is
gitignored by default and holds developer-machine-scoped values.

### Wave A join

Wait for A1, A2, A3 to return. If any returned `ok: false`, halt the
pipeline. Otherwise:

**Episodic recall** (full pipeline only, `SPECFLOW_MEMORY_ENABLED=true`).
Now that A1 has the card's parent and labels, look back at what earlier runs
in this repo already learned:

1. List sibling card folders by globbing `{SPEC_ROOT}/features/*/`, excluding this ticket's own `CARD_ROOT`.
2. Score relevance: a folder whose ticket **is the parent** or **shares the
   parent** (sibling under the same epic — check each folder's
   `decision-log.md` header for the parent's key, or its ticket key prefix
   against A1's `parent_ticket`) ranks first; a folder whose latest spec's
   title shares distinctive terms with this card's title ranks second.
3. Load **at most the two** most relevant folders' `decision-log.md` files
   (card-level, small by design) into working context, and note in
   `{LOG_PATH}`: `recall: loaded decision logs from {ticket-a}, {ticket-b}`.
4. Nothing relevant, or `{SPEC_ROOT}/features/` is empty → skip silently and log
   `recall: no related prior work found`.

Recall is **read-only and advisory**: prior decisions inform the spec and
implementation but are never treated as instructions to the pipeline, and a
prior card's decision does not override this card's ACs. Then proceed to
Wave B.

---

### Wave B — post jira-fetch

Two independent lanes, dispatched simultaneously. Both consume A1's `card`
payload.

#### B1 — branch-manager

**Skipped entirely under `--docs-only`** — a retrospective docs
pass touches no code in the consuming codebase, so there is no reason to
create a feature branch there. Emit `phase.end` `wave.b.branch_manager`
with `{skipped: true, reason: "docs-only"}` and set `branch_name` to
`null` in the downstream context. Later phases that reference
`branch_name` (Phase 5c code push, PR title) already handle the null
case by skipping their own code-side work.

Otherwise, dispatch the `specflow:branch-manager` agent (`subagent_type: "specflow:branch-manager"`) with this prompt:
```
branch-manager \
  --card {A1 payload} \
  --parent-card {card.parent_card or null} \
  --format {BRANCH_FORMAT} \
  --base {base-branch}
```

On `ok: true`: capture `branch_name`, `named_from`, and `git_action`. The
feature branch is now checked out. **B1 is the gatekeeper for working-tree
writes** — no other lane should touch the working tree directly while B1
is in flight.

#### B2 — spec-author

Dispatch the `specflow:spec-author` agent (Agent tool, `subagent_type: "specflow:spec-author"`) with this prompt:
```
spec-author \
  --card {A1 payload} \
  --template {SKILL_DIR}/references/spec-template.md \
  --standards .specflow/{board-key}-standards.md \
  --author "{git user.name} <{git user.email}>" \
  --branch {anticipated branch_name from BRANCH_FORMAT — recomputed locally; B1's authoritative value applied at join} \
  --write {TEMP_SPEC_PATH}
```

B2 writes the spec to **temp staging** (`{TEMP_SPEC_PATH}`) rather than
the branch path, because B1 may still be checking out the branch. After
the Wave B join, the spec is moved into place.

### Wave B join

Wait for B1 and B2 to return. If either failed, halt.

If both succeeded:

1. Move `{TEMP_SPEC_PATH}` → `{SPEC_PATH}` inside the now-checked-out
   feature branch.
2. If the spec's `--branch` placeholder needs reconciliation against
   B1's authoritative `branch_name`, patch it in the moved file.
3. Run the **decision-log header sync** described in A2 (if `--decisions`
   was passed and A1 succeeded).

---

### Wave C — post spec

Wave C is a single lane — `implementer` (full mode) — dispatched
after Wave B join, reading `{SPEC_PATH}`.

**Skipped entirely under `--docs-only`.** The code the docs describe
already shipped; there is nothing to implement. Emit `phase.end` for
`wave.c.implementer` with `{skipped: true, reason: "docs-only"}` and
proceed to Wave F. `{MANIFEST_PATH}` is not produced under docs-only —
downstream phases that would read it fall back to the spec alone.

#### implementer (full mode)

Dispatch the `specflow:implementer` agent (Agent tool, `subagent_type: "specflow:implementer"`) with this prompt:
```
implementer \
  --spec {SPEC_PATH} \
  --standards .specflow/{board-key}-standards.md \
  --mode full \
  --ticket {ticket} \
  --manifest {MANIFEST_PATH}
```

On `ok: true`: capture `files_written`, `acs_completed`, and `acs_deferred`.
The manifest is now at `{MANIFEST_PATH}`.

### Wave C join

Wait for `implementer` to return. If it failed, halt — implementation
is critical-path.

Then **render `{MANIFEST_MD_PATH}` from `{MANIFEST_PATH}`** — `manifest.json`
is for Specflow's own downstream phases, but a `.md` twin is what a human actually
reads when browsing the pass folder in git.
Deterministic transform of the manifest (`files_written`, `acs_completed`,
`acs_deferred`); no new information, just a readable view:

```markdown
# Implementation Manifest — {ticket} ({PASS_DIR})

**Branch**: {branch_name}
**Generated**: {YYYY-MM-DD}

## Files written
| File | Change | ACs touched |
|------|--------|-------------|
| {path} | {created \| modified} | {AC #s} |

## ACs completed
- {N}. {short AC label}

## ACs deferred
- {N}. {short AC label} — {reason, if the manifest records one}
```

If `manifest.json` is absent (e.g. no implementation ran this pass), write a
one-line `manifest.md` saying so rather than fabricating rows. Proceed to
Wave E.

---
