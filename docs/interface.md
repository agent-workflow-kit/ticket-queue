# ticket-queue: interface

Status: 0.1.0-design. Written before implementation (architecture §18.1). This document is the single normative source for ticket-queue's rules; `docs/design.md`, the README and the architecture link to its headings and never restate its tables. The `--json` shapes, hook payloads, config, log records and the re-rank documents are specified in [schemas/](../schemas/).

ticket-queue answers "which ticket should I do next?". It keeps a local pool of candidate tickets per board, read through ticketctl, decides which are eligible, scores them deterministically, optionally lets a model re-rank the top of the list, applies the user's pins and skips, and hands out the top ticket, optionally reserving it with a claim. It never writes to the ticket system: a claim is a local reservation, and status, assignee and sprint changes belong to the caller (through ticketctl).

## Concepts

### Identifiers

A ticket is identified by `(site, key)`. A key matches `^[A-Z][A-Z0-9_]*-[0-9]+$`, is accepted in any letter case and is printed upper-case. A ticket argument is `KEY` or `<site>:KEY`, where `<site>` is a ticketctl site name (`^[A-Za-z0-9][A-Za-z0-9_-]{0,63}$`); a browse URL is not accepted (a caller passes the key). A bare `KEY` that is in the store of more than one site exits 21 `ambiguous_key` with the candidates; `--site <name>` resolves it. `--json` output always carries `key` and `site`. A ticket whose canonical key differs from the key a claim or pin was made for (a moved ticket) is followed by the store: the canonical key is stored and the old key is kept as `requested_key`, an alias that resolves to the canonical entry.

Other identifier grammars: a board name matches `^[a-z][a-z0-9_-]{0,63}$`; a holder and a label value match the claim label-value grammar below; a label key matches `^[a-z0-9][a-z0-9_.-]{0,63}$` and a label value is 0 to 256 characters from `[A-Za-z0-9._:/#@+=,-]` (architecture §8.1; a label is split at the first `=`); an actor matches `^(human|auto):[A-Za-z0-9._@-]{1,64}$`; a decision id matches `^[A-Za-z0-9._:@/-]{1,128}$`; a duration matches `^[0-9]{1,6}(ms|s|m|h|d|w)$` in flags and config. Every identifier is matched with `fullmatch`. An identifier that fails its grammar exits 2 `invalid_argument`.

### Boards, pools and candidates

A **board** is a named entry in config. It names a ticketctl **site** and a **source** (a ticketctl named query, or structured flags), plus the board's goals, prioritized epics, parent allow and deny lists, eligibility rule settings, signal weights and re-rank settings. A **pool** is the set of tickets fetched for a board and stored. A **candidate** is a stored ticket that is eligible. Boards are listed in config in order; a ticket that is in the pool of several boards belongs to the **first** board, in config order, that contains it, and that board's goals, weights, epics and rules apply to it. A ticket in no board's pool is stored only while it is claimed or pinned.

### Eligibility rules

Eligibility is decided locally on the stored ticket, not by the ticket system. Each ticket has a verdict per rule, `pass`, `fail` or `unknown`, and a rule that is `unknown` excludes the ticket just as `fail` does. The reason string of an exclusion is the rule name, or the rule's unknown reason. Rules are **soft** (a pin overrides them) or **hard** (nothing overrides them).

| Rule | Kind | Passes when | Unknown reason |
|---|---|---|---|
| `done` | hard | `status.category` is not `done` (fails: the ticket is done) | none (a ticket with no status is `gone`) |
| `gone` | hard | the ticket was read in the last full rebuild and the last `get` of it did not return not found | none |
| `claimed` | hard | no live claim exists (a claim by the asking caller does not exclude the ticket from that caller's `claim` renewal, see [Claims](#claims)) | none |
| `skipped` | hard | no active skip exists | none |
| `status` | soft | `status.category` is in `rules.status_category` | none |
| `pointed` | soft | `estimate` is not null and `incomplete` does not contain `estimate` | `estimate_unknown` (when `incomplete` names `estimate`) |
| `sprint` | soft | per `rules.sprint`: `open` passes when a sprint with state `active` is in `sprints`; `open_or_future` also passes a `future` sprint; `any` always passes | `sprint_unknown` (when `sprints` is null because the field was not read) |
| `unblocked` | soft | `blocked.value` is false | `blocked_unknown` (when `blocked.value` is null) |
| `assignee` | soft | per `rules.assignee`: `me_or_none` passes an unassigned ticket or one assigned to the ticketctl account; `me`; `none`; `any` | `assignee_unknown` (when `incomplete` names `assignee`) |
| `parent` | soft | the ticket's `epic.key` and `parent.key` are in none of the board's `parent_deny` keys, and, when `parent_allow` is not empty, at least one of them is in `parent_allow` | none |

A rule with its setting turned off (`false` or `any`) always passes. `done` also applies to a ticket whose `status.category` is unknown (the ticket is then not stored as a candidate). `claimed` and `skipped` are decided from the store, not from the ticket. A pin overrides every soft rule and never a hard rule (see [Manual order](#manual-order)). `eligible` is true when every rule passes (for a pinned ticket: every hard rule passes).

### Store

The pool, claims, pins, skips, re-rank results and the hook outbox live in ticket-queue's private state database. Its only readers are ticket-queue's own commands. `list`, `show`, `why`, `claims`, `next` without `--claim` and `browse` read the store and never contact ticketctl. Every output that reports ticket data carries `refreshed_at` (the board's last successful refresh) and `stale`, true when the pool is older than `refresh.stale_after`. The files and records are specified in [Files](#files).

## Commands

Global flags where meaningful: `--json`, `--config PATH`, `--site NAME`, `--verbose`, `--quiet`, `--actor <actor>`, `--decision-id <id>` (repeatable, accepted and logged), `--dry-run` (every command that changes local state), `--version`. Every command that outputs data supports `--json`; errors go to stderr and, with `--json`, to stdout as `{"error": {"code": "...", "message": "...", "details": {...}}}`. Text that can carry ticket or agent text (`--reason-file`) is taken as a file or `-`.

| Command | Purpose |
|---|---|
| `refresh [--board B] [--full] [--rerank]` | Read each board's pool through ticketctl and update the store |
| `list [--board B] [--all] [--skipped] [--claimed] [--query Q\|--query-file F] [--limit N]` | Print candidates in order |
| `show KEY` | Print one stored ticket with its verdicts and state |
| `why KEY` | Explain a ticket's score, verdicts and position |
| `next [--board B] [--claim] [--holder H] [--ttl D] [--label k=v]... [--query Q\|--query-file F] [--refresh]` | Print the top eligible ticket, optionally claiming it |
| `claim KEY [--holder H] [--ttl D] [--label k=v]...` | Reserve a ticket, or renew a claim |
| `release KEY [--failed] [--holder H] [--force]` | End a claim |
| `claims` | List live claims with their ticket state |
| `pin KEY [--top] [--expires D]` | Append a ticket to the pin list |
| `unpin KEY` | Remove a pin |
| `move KEY (--to N\|--top\|--before KEY\|--after KEY)` | Reorder within the pin list, pinning first when needed |
| `skip KEY [--for D\|--until T] [--reason-file F]` | Remove a ticket from candidates |
| `unskip KEY` | End a skip |
| `rerank [--board B] [--dry-run]` | Run the model re-rank |
| `boards` | List the configured boards and their state |
| `validate [--probe]` | Check config, plugin and ticketctl; `--probe` also makes one real re-rank call |
| `cron install\|uninstall [--interval D]` | Manage the crontab block |
| `export [--since T]` | Print `log.jsonl` records |
| `prune [--dry-run]` | Remove old records |
| `browse [--focus KEY] [--board B] [--interval D] [--result-file PATH]` | Terminal UI |

`--version` prints `ticket-queue <semver>`. Every command that changes local state (`refresh`, `claim`, `release`, `pin`, `unpin`, `move`, `skip`, `unskip`, `rerank`, `prune`, `next --claim`) is idempotent and supports `--dry-run`, which reports what it would change and writes nothing. No command takes an external action, so no command is confirm-by-default and none exits 13 in 0.x.

A key that is in no store exits 10 `not_found`. A command that needs a ticket in the store (`show`, `why`, `claim` on a key not stored) reads it live with ticketctl `get` only where stated; `show` and `why` never do.

### refresh

`refresh` is the only command that reads pools from ticketctl. It processes the selected boards (all by default, each in its own lease, see [Concurrency](#concurrency)); a board whose lease is held by another process is skipped and the command exits 0 and says "already running" for it.

For each board it does, in order:

1. **Fetch.** A named-query source runs `ticketctl search --query <name>`; a structured source runs `ticketctl search` with `--project` (`projects`), `--type` (`types`), `--label` (`labels`), `--assignee` (`assignee`, each of `none` and `me`), `--sprint` (`sprint`, each of `open` and `future`) and `--status-category new --status-category indeterminate` (a done ticket is never a candidate, so it is not fetched). Every search passes `--site`, `--limit 500` and `--fields` for exactly the properties ticket-queue reads (`key,title,type,status,assignee,priority,estimate,sprints,parent,epic,labels,due_date,created,updated,blocked,description,url`) and pages with `--cursor`, at most `refresh.max_pages` pages (default 20). An **incremental** refresh adds `--updated-since <last successful refresh minus refresh.overlap>`; a **full** rebuild (`--full`, the first refresh of a board, or the board's `last_full_at` older than `refresh.full_every`) does not. A page limit reached makes the board `truncated` (reported, rows kept).
2. **Upsert** each returned ticket by `(site, key)`, store its normalized fields, evaluate its eligibility verdicts and score, and record `first_seen` and `last_seen`.
3. **Re-read departures.** Every stored ticket that is claimed or pinned, and every one in the top 100 of the board's order, is re-read with `ticketctl get` in batches of 50 (this is the only way a ticket that left the pool is noticed, because `search` never reports departures). A ticket `get` reports as not found is marked `gone`; one whose status category is `done` is marked `done`; any other result updates the row.
4. **Remove.** After a full rebuild, a ticket of this board that was not returned and is not claimed or pinned is removed from the pool (a claimed or pinned ticket stays, marked by the result of step 3).
5. **Re-rank** when `--rerank` is given, or when `rerank.auto` is true and the rules of [Re-rank triggers](#re-rank-triggers) allow it. A failed or skipped re-rank never fails the refresh.

A ticketctl failure of one board leaves that board's rows unchanged and marks the board `refresh_failed` with the error; the other boards still refresh. Mapping of ticketctl's errors: `rate_limited` records `retry_after` and does no in-process waiting; `cursor_expired` and `cursor_invalid` restart that board's incremental search without a cursor, once; `invalid_query` is a configuration error naming the board, and the board stays disabled until the next config change; `auth_required` and `bad_credentials` name the variable ticketctl reported; `unavailable`, `timeout` and `provider_changed` keep the old rows. A `get` for a batch that fails as a whole keeps the old rows and is reported as `departures_unchecked`. A board whose tickets all have `incomplete` naming `estimate` reports `estimate_ambiguous` with the ticketctl config key to set (`fields.estimate`).

Exit codes: 0 when every selected board refreshed (or was already running); 12 when any board failed or was truncated, with the normal output attached (`partial_results`, `details.boards`), and `details.lower_error` carrying ticketctl's exit and code for a failed board. `post-refresh` fires when a board failed, was truncated, or a claimed ticket's state changed to something other than `ok`.

### list, show, why

`list` prints candidates for the selected boards in the order of [Ordering](#ordering), after the query filter. `--all` includes ineligible tickets with their exclusion reasons; `--skipped` and `--claimed` select those sets; `--limit` defaults to 50. `--query` applies [Query language](#query-language) with `ORDER BY` allowed; with no `ORDER BY` the order is `ORDER BY position`. The configured `browse.query` is applied only with `--default-query`.

`show KEY` prints the stored ticket (the fields of the `ticket` object of the schemas), its verdicts, state (`pinned`, `claimed`, `skipped`), and `refreshed_at`. `why KEY` prints, for a ticket: the board, each signal's raw value, normalized value, weight and contribution, the total, the verdict of each rule with the exclusion reason, the pin, move and skip state with warnings for soft rules a pin overrides, the re-rank result (position, reason, age, whether it applied), the claim state, and the position with the effective score. All text from the ticket is sanitized (control, format and invisible characters become `<U+XXXX>` markers). Both read the store only.

### next

`next` prints the top eligible ticket of the selected boards in the order of [Ordering](#ordering). It takes `--query`/`--query-file` for filters only; a query containing `ORDER BY` exits 2 (`invalid_argument`, `reason: order_by_not_allowed`), so the ranking stays the contract of `next`. With `--refresh` it first runs a refresh of the selected boards when the pool is stale. With no eligible candidate it exits 22 `empty` (see [Exit codes](#exit-codes)).

With `--claim` it takes the claim in this order: take the first candidate; read it live with `ticketctl get`; re-evaluate its eligibility on the fresh ticket (hard rules only for a pinned ticket, all rules otherwise); if it fails, record the result in the store, mark `claim_skipped` in the output's `passed_over` list with the reason and take the next candidate; when one passes, write the claim in one transaction (see [Claims](#claims)) and print it. A refresh older than `refresh.stale_after` does not block a claim. If no candidate passes, exit 22 `empty` with `passed_over` listed. A `get` failure (ticketctl unavailable) exits 12 with `details.lower_error` and claims nothing.

### Ordering

The order of candidates is: first the **pin list** (in pin order) restricted to pinned tickets that pass the hard rules; then every other eligible ticket by **effective score** descending, then `created` ascending (older first), then `(site, key)`. `position` is the 1-based position in this order for the selected set (all boards by default). A pinned ticket that fails a hard rule is listed with state `pinned_blocked` and is never handed out. The **effective score** is the deterministic [score](#scoring), except for tickets covered by an applied re-rank result (see [Re-rank](#re-rank)).

### Scoring

The score of a ticket is the sum over signals of `weight × signal`, each signal normalized to 0..1, using its board's weights. A signal whose input is missing contributes 0 and is shown as `unknown`; an absent optional value that is known absent (no due date) is 0 and shown as `none`. Scores are rounded to two decimals. Defaults and formulas:

| Signal | Input | Normalization | Default weight |
|---|---|---|---|
| `priority` | `priority.name` | `signals.priority_map` (case-insensitive name to 0..1; defaults Highest 1, High 0.75, Medium 0.5, Low 0.25, Lowest 0); a name not in the map falls back to `max(0, 1 - priority.rank / (signals.priority_levels - 1))` (`priority_levels` default 5), and to unknown when `rank` is null | 40 |
| `sprint` | `sprints[].state` | an `active` sprint 1, only a `future` sprint 0.3, none 0 (`signals.sprint.active`, `.future`) | 25 |
| `epic` | `epic.key`, else `parent.key`, against the board's `prioritized_epics` (n entries) | the key at index `i` (from 0) gives `1 - i/n`; not listed 0 | 20 |
| `due` | `due_date` | days until due `d` (negative when overdue): `d <= 3` gives 1, `d >= 30` gives 0, else `(30 - d) / 27`; no due date is 0 | 20 |
| `assignee` | `assignee` | assigned to the ticketctl account 1, unassigned 0 | 10 |
| `age` | `created` | `min(days since created / 90, 1)` | 5 |
| `labels` | `labels` | `min(sum of signals.label_bonus[label], 1)` | 0 |
| `size` | `estimate` | `max(0, 1 - estimate / 13)`; unknown when `estimate` is null | 0 |

Days use the UTC calendar day of the evaluation time. The weights and signal settings are per board and overridable (see [Config](#config)). `why` prints every term.

### Re-rank

Re-rank is optional (`rerank.enabled`, default false) and advisory: a failure, an expired result or no result leaves the deterministic order. It reorders only the tickets it was given.

**Candidates given.** The top `rerank.top_n` (default 20, at most 30) eligible tickets of one board by deterministic score, excluding pinned and skipped tickets. A board's re-rank covers only its own tickets.

**Applying a result.** A stored, valid, unexpired result permutes the **effective scores** of the tickets it covers: the covered tickets keep the set of deterministic scores they hold, and the highest of those scores goes to the ticket ranked first by the result, the next to the second, and so on; tickets listed as `excluded` by the model take the lowest scores of the set, in their deterministic relative order. A covered ticket that has since left the pool or become ineligible is dropped from the result and the rest keep their relative order; a ticket that is new keeps its deterministic score. Tie-breaks are those of [Ordering](#ordering). A result is applied only until `rerank.ttl` (default 24 hours) after it was made, and is shown as `rerank: expired` after that. `list --json` and `why` report whether a result applied, its age, and each ticket's re-rank position and reason. A re-rank never changes eligibility, the deterministic score or the pins.

#### Re-rank triggers

`rerank [--board B]` runs a re-rank for the selected boards now. With the global `rerank.enabled` false the command exits 20 (`reason: rerank_disabled`). With it true, a board whose own `rerank.enabled` is false is skipped with `rerank_disabled` in the output unless `--board` names it. `refresh --rerank` runs it after the refresh. With `rerank.auto: true` every refresh runs it, but only when the **input hash** differs from the last valid result's (the hash covers the candidate key set, the order, the board's goals and prioritized epics, the skill name, the model and the plugin hash), not more than once per `rerank.min_interval` (default 1 hour) and at most `rerank.daily_limit` times per local day (default 8). A blocked automatic run is logged as `rerank_skipped` with the reason (`unchanged`, `min_interval`, `daily_limit`, `disabled`) and fires no hook except `post-rerank` for a failure. `--dry-run` prints the input document that would be given to the command and calls nothing. `next` and `list` never run a re-rank.

#### Slot contract

The re-rank slot is run by a command. `rerank.command` is an argv list run without a shell, or null (the default), meaning ticket-queue's bundled implementation (below). A custom command receives the **input document** on stdin and prints the **output document** on stdout; exit 0 means the output is to be used, any other exit is a failure. Ticket text never travels in argv or environment. The command runs in a private empty directory with a built environment (`PATH`, `HOME`, `LC_ALL=C` and the names in `rerank.env_allowlist`), under `rerank.timeout`, in its own process group which ticket-queue kills on expiry (group signals are allowed only for groups ticket-queue created).

The **input document** (schema `schemas/rerank-input.json`) is a JSON object: `schema_version` (1), `run_id` (32 random hex characters), `board`, `goals` (the board's trusted free text, or an empty string), `prioritized_epics` (keys in order) and `candidates` (the top tickets in deterministic order: `key`, `position`, `deterministic_score`, `signals` (the normalized values), and the ticket fields `title`, `description` (cut to `rerank.description_chars`, default 1000 characters), `type`, `status`, `priority`, `estimate`, `sprint` (the best state), `epic`, `parent`, `labels`, `due_date`, `age_days`). The ticket text fields (`title`, `description`, `labels`, the titles of `epic` and `parent`) are untrusted (`x-untrusted`).

The **output document** (schema `schemas/rerank-output.json`) is a JSON object with exactly the keys `schema_version` (1), `run_id` (echoed), `ranking` (best first: objects `{key, reason}`), `excluded` (optional, objects `{key, reason}`) and, for the bundled implementation only, `skill_marker`. `reason` is at most 200 characters and untrusted. The document is at most 32 KiB.

**Validation.** ticket-queue parses the output strictly (strict UTF-8, no `NaN` or `Infinity`, no duplicate keys, a JSON object, no more than the size cap, no more than 64 nesting levels) and requires: the exact key set above, a matching `run_id`, `ranking` plus `excluded` equal to the candidate key set exactly (no invented, duplicate or missing key), every `reason` a string within its cap and, for the bundled implementation, `skill_marker` equal to the value ticket-queue expects. Any violation discards the whole result, keeps the deterministic order, records `rerank_failed` with a reason from the table below and fires `post-rerank`; a re-rank failure is never a failure of the command that triggered it (`rerank` itself exits 12 `rerank_failed`).

| Reason | Meaning |
|---|---|
| `tool_missing`, `tool_too_old` | claude-session is not installed or older than the minimum |
| `sandbox_unavailable` | `rerank.sandbox` is `required` and the sandbox cannot start |
| `launch_failed` | claude-session refused the launch (exit 12) |
| `unreported` | the run ended without a report (claude-session exit 14) |
| `blocked` | the run reported `blocked` or `needs-input` (exit 13) |
| `timeout` | the run hit its timeout or turn limit (exit 16) |
| `session_failed` | any other non-zero claude-session outcome |
| `artifact_missing`, `artifact_invalid` | no artifact, or it failed parsing or its size check |
| `run_id_mismatch`, `marker_mismatch` | the echoed run id or skill marker is wrong |
| `keys_mismatch` | the key sets do not equal |
| `command_failed`, `output_invalid` | a custom command exited non-zero or printed an invalid document |
| `deadline` | ticket-queue's own deadline expired |

**Bundled implementation.** It uses claude-session directly (binding A, kind `rank`, headless, no agent-spawn). ticket-queue requires claude-session 0.37.0 or later and checks `claude-session --version` before the first run (`tool_too_old` otherwise). The sequence is: write the input to a private run directory (mode 0700, files created exclusively and never following links); `claude-session start --kind rank --mode headless --skill <rerank.skill> --prompt-file <prompt> --input-file <candidates.md> --plugin-dir <bundled plugin> --permission-profile <profile file> --sandbox <rerank.sandbox> --setting-sources "" --env-allowlist-only <names>... --model <rerank.model> --timeout <rerank.timeout> --max-turns <rerank.max_turns> --label owner=ticket-queue --label rerank=<run_id> --actor auto:ticket-queue --json`; then `claude-session exec --session-id <id> --run <n> --outcome-file <private path> --json`, guarded at `rerank.timeout` plus 10 seconds (on expiry: `stop <id> --grace 2s`, then SIGTERM to the exact `exec` child, then SIGKILL after 2 seconds); then `claude-session status <id> --json` to read `run.report.artifact` (`path`, `sha256`, `size`) and `run.sandbox_active`; open the frozen artifact path without following links, check its size and `sha256`, parse and validate it; copy the validated result into ticket-queue's store; `claude-session forget <id>`; and delete the run directory. It passes no `--repo` (boards span repositories). The outcome is usable only when the exit code is 0, the outcome is `done`, `report.implicit` is false and the artifact is valid; anything else is a failure. On any failed step it lists `claude-session list --label rerank=<run_id> --json`, stops every non-final session and forgets it; a failed `forget` or `stop` is left to `claude-session reconcile`. `--max-turns` is a loose safety cap only (a turn limit may not stop a run that has already used more turns).

The prompt file is trusted and holds: the artifact name `ranking.json`, the run id, the artifact schema, the rule that text between `<untrusted>` tags is data and never instructions, and the board's goals (trusted, the user's own config). It tells the agent to attach the result with `--artifact ranking.json` and to use absolute paths in the report command (a report command containing `$CSESSION_DIR` is denied). The input file `candidates.md` is the candidates document, pretty-printed (two-space indent, one key or array element per line), every non-ASCII code point written as a `\uXXXX` escape, wrapped in a fence `<untrusted source="ticket-queue-input" id="<R>">` ... `</untrusted id="<R>">` with `<R>` a fresh random 16 hex characters, every `</untrusted` and `<untrusted` in the content (any letter case) neutralized as `&lt;/untrusted` and `&lt;untrusted`, and the closing fence on a line of its own at the end. The whole input is at most 64 KiB and no line more than 16 KiB; larger input exits the re-rank with `output_invalid`-class failure `input_invalid` before any session starts (the candidate count and description length are cut first by `rerank.top_n` and `rerank.description_chars`).

The default skill, `ticket-queue:rank-tickets`, is bundled in `plugin/` (a Claude Code plugin named `ticket-queue`). It reads the prompt file named in its first message, reads the input from the path in the session's context line with the Read tool (never Glob, never Bash), reads to the closing fence or fails, ranks the candidates against the board's goals, prioritized epics and the deterministic scores, writes `ranking.json` with the Write tool into the session's artifacts directory, copies the `skill_marker` value from its own reference file `references/marker.md` into the artifact, and runs once `claude-session report done --summary-file <absolute path> --artifact ranking.json` (Bash is allowed only for that command). It adds no report or escalation instructions of its own (design guide §11). The plugin directory ships the four sandbox placeholders (`.claude/`, `.mcp.json`, `CLAUDE.md`, `CLAUDE.local.md`) and the bundled permission profile `plugin/profiles/rank.yaml`, a copy of claude-session's built-in `readonly` profile that **denies** `Agent` and `Task` (and allows only `Skill`, `Read`, `Write`, `Edit` under the generated rules and `Bash(claude-session report:*)`). A custom `rerank.skill` must follow the same contract (input, artifact name, schema, run id echo); the `skill_marker` check applies only to the bundled skill, so a custom skill that does not echo it needs `rerank.marker: null` (documented as lowering the check to the run id).

`validate --probe` runs one real re-rank on a built-in sample of three fake tickets and reports `probe_ok`, `sandbox_active`, the duration and the cost; `validate` without `--probe` resolves the skill, the plugin placeholders, the profile file and the claude-session version without a model call. `--dry-run` of a launch is not a self-test: it reports a healthy sandbox for setups in which every report call still fails (a missing placeholder, an interpreter reachable only through an alias).

### Claims

A claim reserves a ticket so it is not handed out twice. It is local: it never writes to the ticket system. A claim has: `(site, key)`, `board`, `claimed_at`, `renewed_at`, `expires_at`, an optional `holder`, labels and `consecutive_failures` (kept on the ticket, not the claim, across claims).

`holder` is an explicit opaque string (`--holder`), never inferred from the caller; it matches the label-value grammar. Labels follow the usual `--label key=value` rule (a multiset; adding an identical pair is a no-op).

**Taking a claim** (`claim KEY`, `next --claim`) follows this order: read the ticket live with `ticketctl get` (for `claim KEY` on a key not in the store this creates its row with board null); evaluate the hard rules on the fresh ticket (and, for `next --claim`, all rules); refuse or pass over a failing ticket; then in one transaction (`BEGIN IMMEDIATE`, comparing the ticket row's revision read before the live read) write the claim. A concurrent change of the row makes the command retry the whole sequence once and then exit 12 `unavailable` (`reason: contended`).

`claim KEY` on an explicit key refuses only on hard rules: exit 20 with `reason` `done`, `gone`, `skipped` or `claimed` (held by another holder). Soft-rule failures are `warnings` in the result, because naming a ticket is the user's intent; a key outside every board's pool is allowed. `claim KEY` on a ticket the caller already holds renews it: `expires_at` is extended, `renewed: true` is reported and labels are added. A caller holds a claim when its `--holder` equals the claim's holder exactly (both absent counts as equal). A claim held by a different holder exits 20 `claimed`. A renewal of a ticket that is now `done` or `gone` exits 20 with that reason; a `reassigned` ticket renews with a warning.

**TTL.** A claim lives until `release`, except that a claim that is not renewed within its TTL lapses: `claim.ttl` (default 72 hours; `--ttl` per claim, at most `claim.max_ttl`, default 30 days). A lapsed claim is released at the next command that reads claims, with a log entry, and does not count as a failure.

**Release.** `release KEY [--failed] [--holder H]` is idempotent: releasing an unclaimed ticket exits 0 with `unchanged: true`. A holder that differs from the claim's exits 20 `holder_mismatch`; only a human `--force` (actor `human:`) overrides it. A normal release sets the ticket's `consecutive_failures` to 0. `release --failed` adds 1 and records `last_failure_at`; when the count reaches `claim.max_failures` (default 3) the same transaction ends the claim and creates a skip with `source: auto` and a reason, and `post-auto-skip` fires. The skip lasts until `unskip`.

**Claim state.** `claims` and `list --claimed` report `ticket_state` for each claim: `ok`, `done`, `gone`, `reassigned` (the assignee is neither the ticketctl account nor empty) or `moved` (the key changed; the claim follows the canonical key). A refresh never releases a claim. `prune` releases a claim whose ticket has been `done` or `gone` for more than `claim.dead_grace` (default 7 days), with a log entry; this is the only automatic release besides a lapsed TTL. A claim never excludes the ticket from `show` or `why`.

### Manual order

**Pins.** The pin list is an ordered list of tickets above all scoring and re-rank. `pin KEY` appends the ticket (`--top` puts it first); `--expires D` sets an expiry. `unpin KEY` removes it. `move KEY` takes `--to N` (1-based), `--top`, `--before KEY` or `--after KEY` (the anchor must be pinned) and reorders within the pin list, pinning the ticket first when it was not pinned. A pin has no timer unless `--expires` is given; it is dropped automatically, with a log entry, when the ticket reaches the `done` status category or disappears in a full rebuild, and it is kept through a normal release. A pinned ticket that fails a hard rule is `pinned_blocked`: it stays stored, is shown with its reason, and is never handed out. A pin overrides the soft rules; `why` lists each overridden soft rule as a warning.

**Skips.** `skip KEY [--for D|--until T] [--reason-file F]` removes the ticket from candidates. A skip with no time lasts until `unskip KEY`; a timed skip ends by itself (`until` is an RFC 3339 instant). `source` is `human` or `auto`; an auto skip comes from `release --failed` (see [Claims](#claims)) and never times out. `--reason-file` text is untrusted and shown sanitized. `list --skipped` lists both kinds. A skipped ticket fails the hard rule `skipped`.

Pin, move and skip changes are written to `log.jsonl` with the actor. They are not external actions and have no audit record.

### Query language

One engine, in one UI-free module, serves `list --query`, `next --query`, the TUI panel, `browse.query` and the quick presets. A query is text of at most 1000 characters (a word at most 200, nesting at most 20, at most 50 conditions, an `IN` list at most 50 entries); control characters are refused; there is no eval and no regex (`LIKE` uses an iterative matcher). It narrows the stored pool and never widens it. A syntax error exits 2 `invalid_argument` with `reason: bad_query` and the position.

Grammar: `expr := term (OR term)*`; `term := factor (AND factor)*`; `factor := NOT factor | '(' expr ')' | condition | boolean-field`; `condition := field op value | field IN '(' value (',' value)* ')' | field NOT IN (...) | field IS [NOT] NULL | field [NOT] LIKE string`; ops `= != <> < <= > >=`; `query := [expr] [ORDER BY key [ASC|DESC] (',' key [ASC|DESC])*]` with at most 8 keys, each at most once. Keywords are case-insensitive. A string is single-quoted with `''` for a quote. A duration is `[0-9]+w`, `d`, `h`, `m` terms each used at most once in that order (`1w2d`, `90m`, `12h30m`). `me` (unquoted) is the ticketctl account's display identity and matches `assignee` and `reporter` only; `'me'` is text. A number is an integer or decimal. `true` and `false` are booleans.

| Field | Type | Notes |
|---|---|---|
| `key`, `project`, `title`, `status`, `type`, `priority`, `assignee`, `reporter`, `epic`, `parent`, `sprint`, `board`, `holder` | text | `epic` and `parent` compare their keys; `sprint` the names of the ticket's sprints (any); `holder` the claim's holder |
| `labels` | set of text | `labels = 'x'` means the set contains `x`; `!=` means it does not; `IN (...)` means it contains any; `LIKE` matches any member |
| `text` | free text | `LIKE` only; over `title`, `key` and `labels` |
| `estimate`, `score`, `position`, `rerank_position`, `failures` | number | `score` is the effective score, `failures` the consecutive failed releases |
| `age`, `updated`, `due` | duration | `age` since created, `updated` since last update, `due` the time until the due date (negative when overdue); compared with unsigned durations |
| `mine`, `pointed`, `blocked`, `claimed`, `pinned`, `skipped`, `eligible`, `overdue` | boolean | usable bare (`claimed`, `NOT blocked`) |

Evaluation is three-valued (Kleene logic, as in SQL): a condition on a value that is unknown or not loaded is unknown; `NOT unknown` is unknown; a row is kept only when the whole expression is true; `IS NULL` is true for unknown. In `ORDER BY`, an unknown value sorts last in both directions, and ties end on `(site, key)`. `eligible` is never unknown for a stored ticket. A key that is not a field exits 2 `unknown_field`.

The **named order** `position` is the ranked order of [Ordering](#ordering). Quick presets of the TUI are query texts: `m` mine (`mine`), `d` due soon (`due < 7d`), `b` blocked (`blocked`), `c` claimed (`claimed`), and `a` shows ineligible tickets (a view switch, not a query).

### Concurrency

Commands run by cron or the harness are safe under concurrent invocation. `refresh` and `rerank` take a per-board lease (holder PID and process start time, plus an expiry); a second caller exits 0 and reports "already running" for that board. A lease whose holder is dead may be taken over. Claims and pins use compare-and-set transactions. `next`, `list`, `show`, `why`, `claims` and `browse` never wait for a refresh: they read the last committed rows.

### Hook delivery

Post-hooks are delivered through an outbox: every event is written to the store in the transaction that caused it, with an `event_id`, and delivered at least once after the commit. The next command (and every `refresh`) redelivers undelivered events. A receiver deduplicates on `event_id`. A failed hook is logged and warned and never fails a command.

## Exit codes

Reserved codes follow design guide §4.2.

| Code | Meaning in ticket-queue |
|---|---|
| 0 | success (including "already running" and `unchanged`) |
| 1 | unexpected internal error |
| 2 | usage error (`invalid_argument`, with a `reason` such as `bad_query`, `unknown_field`, `order_by_not_allowed`, `invalid_config`) |
| 10 | not found: a key in no store, an unknown board, an unknown claim |
| 11 | not used in 0.x (there are no pre-hooks) |
| 12 | a lower tool failed, a refresh was partial, or the store was contended (`partial_results`, `lower_failed`, `rerank_failed`, `unavailable`, `tui_missing`, `tool_missing`, `tool_too_old`) |
| 13 | not used in 0.x |
| 14 to 19 | not used |
| 20 | refused, with `reason` in {`claimed`, `done`, `gone`, `skipped`, `holder_mismatch`, `rerank_disabled`} |
| 21 | ambiguous key (several sites hold it); `details.candidates` lists them |
| 22 | `empty`: `next` found no eligible candidate |

Output by exit code: 0, 12 (when partial) and 22 print the normal object (`list` and `next` print `{"empty": true, ...}` with `--json` for 22); 2, 10, 20, 21 and a total 12 print the error envelope. A tool-specific code is documented here and in the README.

## `--json` outputs

Schemas live in [schemas/cli/](../schemas/cli/). Every output is a JSON object (never a bare array), validated against its schema in tests. Every output that reads the store carries `refreshed_at` (a map of board to timestamp for several boards), `stale` and, for multi-board outputs, `boards` (each with `status`: `ok`, `refresh_failed`, `truncated`, `disabled`). Outputs carry `complete` (false when a board is failed or truncated) and `errors`. Ticket text fields (`title`, `description`, `labels`, `epic.title`, `parent.title`, reasons) are `x-untrusted` in the schemas and printed sanitized. The set of fields of a ticket object follows ticketctl's normalized ticket (`key`, `site`, `url`, `title`, `type`, `status`, `assignee`, `priority`, `estimate`, `sprints`, `parent`, `epic`, `labels`, `due_date`, `created`, `updated`, `blocked`, `incomplete`) plus ticket-queue's own `board`, `score`, `position`, `rerank` (`applied`, `position`, `reason`, `age_s`), `eligible`, `verdicts`, `claim`, `pinned`, `skipped` and `warnings`. Additive changes (new optional fields) are allowed within a minor version; removals and renames are breaking (design guide §13).

## Hooks

Post-hooks only in 0.x (no `pre-` hooks; a `pre-claim` hook is a 1.x item). Hooks are configured in `hooks:` (design guide §8): each entry is `{cmd, timeout, async}`, run via `sh -c` in its own process group with the JSON payload on stdin and `TICKET_QUEUE_HOOK`, `TICKET_QUEUE_ID` and `TICKET_QUEUE_OBJECT_TYPE` in the environment; the default timeout is 30 seconds; the hook's environment is the tool's environment minus the names in `ticketctl.pass_env`. Payloads (schemas in `schemas/hooks/`) carry `tool`, `hook`, `event_id`, `object_type`, `object_id`, `board`, `key`, `site`, `holder`, `labels` (the claim's labels where one exists) and counts, never ticket text. Payloads omit `repo` (ticket-queue has no repository notion), so a gate matching on a payload lands in scope `*`.

| Hook | Fires | Notes |
|---|---|---|
| `post-claim` | after a claim is written (`claim`, `next --claim`) | `renewed` true for a renewal |
| `post-release` | after a claim ends by `release` or a lapsed TTL | `failed`, `reason` (`released`, `lapsed`, `pruned`) |
| `post-auto-skip` | after `release --failed` reached `claim.max_failures` | the skip's `source` and `failures` |
| `post-refresh` | after a refresh in which a board failed or was truncated, or a claimed ticket's state changed to non-`ok` | per-board status, error codes, the keys whose claim state changed |
| `post-rerank` | after a re-rank run failed or was skipped for a failure reason | `reason` from the table, `board`, `run_id` |

## Config

YAML at `~/.config/ticket-queue/config.yaml` (respecting `XDG_CONFIG_HOME`), plus `config.d/*.yaml` drop-ins merged in lexical order after it (maps merge; hook lists append). The schema is [schemas/config.json](../schemas/config.json); `config.example.yaml` documents every key and its default. Invalid config fails fast naming the key (exit 2, `invalid_config`). Structured config (`boards`, `hooks`, maps) cannot come from the environment; only top-level and nested scalars can. Precedence: flags, then `boards.<name>` overrides, then environment variables, then global config (with drop-ins), then built-in defaults. ticket-queue has no notion of a repository, so the board takes the per-repo position of the precedence order (design guide §5).

| Key | Default | Meaning |
|---|---|---|
| `me.name`, `me.jira` | OS username, unset | the actor name; an expected ticketctl account display name (a mismatch warns) |
| `ticketctl.config` | null | an explicit ticketctl `--config` path |
| `ticketctl.pass_env` | `[]` | names of environment variables handed to the ticketctl child (in addition to `PATH`, `HOME`, the XDG variables and `TICKETCTL_*`) |
| `ticketctl.default_site` | null | the site for a board that names none |
| `boards.<name>.site` | `ticketctl.default_site` | the ticketctl site |
| `boards.<name>.source` | required | `{query: <name>}` or any of `projects`, `types`, `labels`, `assignee` (`none`, `me`), `sprint` (`open`, `future`) |
| `boards.<name>.goals`, `goals_file` | empty | the board's free-text goals (trusted: the user's own text) |
| `boards.<name>.prioritized_epics` | `[]` | epic keys in priority order |
| `boards.<name>.parent_allow`, `parent_deny` | `[]` | parent or epic keys |
| `boards.<name>.rules` | `status_category: [new]`, `pointed: true`, `sprint: open`, `unblocked: true`, `assignee: me_or_none` | each rule can be turned off with `false` or `any`; `sprint` takes `open`, `open_or_future`, `any`; `assignee` takes `me_or_none`, `me`, `none`, `any`; `status_category` is a list of `new`, `indeterminate` |
| `boards.<name>.weights` | the defaults of [Scoring](#scoring) | per-signal weights |
| `boards.<name>.signals` | the defaults of [Scoring](#scoring) | `priority_map`, `priority_levels`, `sprint`, `label_bonus` |
| `boards.<name>.rerank` | `enabled: false`, `top_n: 20` | per-board re-rank switch and size |
| `claim.ttl`, `claim.max_ttl` | `72h`, `30d` | claim lifetime without renewal |
| `claim.max_failures` | `3` | consecutive failed releases before an auto-skip |
| `claim.dead_grace` | `7d` | age of a done or gone claimed ticket before `prune` releases it |
| `refresh.stale_after` | `30m` | pool age shown as stale |
| `refresh.full_every` | `24h` | full rebuild interval |
| `refresh.overlap` | `5m` | incremental refresh overlap |
| `refresh.max_pages` | `20` | pages per board and refresh |
| `rerank.enabled` | `false` | the global re-rank switch |
| `rerank.command` | `null` | a custom argv list; null is the bundled implementation |
| `rerank.skill`, `rerank.marker` | `ticket-queue:rank-tickets`, the bundled value | the skill and the expected marker (`null` disables the marker check) |
| `rerank.model` | `haiku` | passed to claude |
| `rerank.top_n`, `rerank.description_chars` | `20`, `1000` | candidates given and description cut |
| `rerank.ttl`, `rerank.min_interval`, `rerank.daily_limit` | `24h`, `1h`, `8` | result lifetime and automatic-run limits |
| `rerank.auto` | `false` | run after each refresh when the input changed |
| `rerank.permission_profile` | the bundled `rank.yaml` | a profile file replacing the bundled one |
| `rerank.sandbox` | `preferred` | `required`, `preferred` or `off` |
| `rerank.setting_sources` | `[]` | setting sources passed with `--setting-sources` |
| `rerank.env_allowlist` | `[PATH]` | names passed with `--env-allowlist-only` |
| `rerank.max_turns`, `rerank.timeout` | `12`, `150s` | loose turn cap and wall-clock bound |
| `hidden.projects`, `hidden.labels` | `[]` | glob lists (`*` and `?`) of projects and labels never shown; applied when a pool is stored, always on, with no reveal |
| `browse.refresh_interval` | `120s` | automatic refresh in the TUI (0 turns it off) |
| `browse.detail_collapsed` | `[]` | detail sections collapsed at start |
| `browse.query` | `null` | the default query (written by the TUI to `config.d/90-browse-query.yaml`) |
| `cron.env_file` | `null` | a user-owned env file sourced by the cron line |
| `hooks.<name>` | `[]` | hook entries |
| `log.max_bytes`, `log.keep` | `10MiB`, `3` | `log.jsonl` rotation |
| `retention.tickets`, `retention.events` | `30d`, `14d` | `prune` ages |

A durations value is a string such as `30m`, `24h` or `7d`.

## Environment variables

Config values from the environment are named `TICKET_QUEUE_<KEY>` (upper-cased, nested keys joined with `__`, e.g. `TICKET_QUEUE_REFRESH__STALE_AFTER`). The hook variables are `TICKET_QUEUE_HOOK`, `TICKET_QUEUE_ID` and `TICKET_QUEUE_OBJECT_TYPE`; these three names are never config keys. ticket-queue reads no credentials: ticketctl's own variables are named in ticketctl's config, and ticket-queue hands them to the ticketctl child only when they are listed in `ticketctl.pass_env`. `XDG_CONFIG_HOME` and `XDG_STATE_HOME` are respected.

## Files

State is under `~/.local/state/ticket-queue/` (respecting `XDG_STATE_HOME`).

| File | Public | Content |
|---|---|---|
| `state.db` | no | SQLite (WAL), versioned, additive-only migrations: tickets, boards, claims, pins, skips, rerank results, events outbox |
| `log.jsonl` | yes | append-only operational log, schema [schemas/log.json](../schemas/log.json) |
| `tmp/<run_id>/` | no | a re-rank run's private input and prompt files (mode 0700), deleted when the run ends |

There is no `audit.jsonl`: no command takes an external action. `--actor` and `--decision-id` are accepted and recorded in `log.jsonl`. [schemas/audit.json](../schemas/audit.json) is kept as an empty contract for tools that read every tool's audit file.

The store keeps, for each ticket, the normalized object as ticketctl returned it (sanitized by ticketctl), with `description` cut to 8 KiB, and the verdicts. It holds ticket text on disk; it is mode 0600 in a 0700 directory. `prune` removes tickets in no pool that are not claimed or pinned after `retention.tickets`, expired re-rank results, delivered events after `retention.events` and claims as described under [Claims](#claims); `log.jsonl` is kept (rotated by size under a lock, never pruned). `cron install` also schedules `prune` daily.

## Audit and log

`log.jsonl` records: `refresh` (per board: counts, status, duration), `claim`, `release`, `claim_lapsed`, `auto_skip`, `pin`, `unpin`, `move`, `skip`, `unskip`, `pin_dropped`, `rerank` (outcome, reason, run id, duration, cost when known, `sandbox_active`), `rerank_skipped`, `prune` and `hook_failed`. Each record has `ts`, `event`, `actor`, `decision_ids` and the event's fields; none holds ticket text beyond keys. [schemas/log.json](../schemas/log.json) defines them. `export [--since T]` prints the records.

## Cron

`cron install [--interval D]` writes one block in the crontab (default interval 15 minutes): the first line `# ticket-queue managed v1 every=<interval>`, a `PATH=<dirs>` line, `*/15 * * * * [. '<env_file>' && ]'<absolute ticket-queue>' refresh --quiet`, a daily `prune --quiet` line, and the last line `# ticket-queue managed end`. A block starts at a line beginning `# ticket-queue managed` and ends at the end line, or, when the end line is absent (a block written by another program), at the next blank line or the next `# ... managed` line. `cron uninstall` removes it. The harness writes the same block.

## TUI

`ticket-queue browse` is an optional Textual interface (the `ticket-queue[tui]` extra; without it the command exits 12 `tui_missing` and names the install command). Flags: `--focus KEY`, `--board B`, `--interval D` (wins over `browse.refresh_interval`) and `--result-file PATH`.

**List.** One table over all boards or one board (`tab` switches). Columns: `key` (cut from the left), `note` (`[refreshing]`, `[stale: <code>]`, `[changed]`), `score`, `pos`, `title`, `status`, `pri`, `est`, `sprint`, `age`, `flags` (`P` pinned, `C` claimed with holder, `S` skipped, `R` re-ranked, `!` warning). Every cell is cut to its width in terminal cells; empty cells stay as padding. The header is sticky with a sort arrow; left and right (or `h` and `l`) scroll by column with `key` and `note` pinned; the offset survives refreshes. Sorting is by parsed value (unknown last in both directions), the header cursor (`H`) sorts on enter or space and reverses on a second press, and the ranked order is the default preset (`ORDER BY position`).

**Query panel (`F`).** A form and a query line kept in sync, with match counts and unknown counts and the actions Apply, Cancel, Reset and Reset to default. `D` writes the current query as the default to `~/.config/ticket-queue/config.d/90-browse-query.yaml` (atomic, mode 0600; the path is shown in the status line). History is kept in `~/.local/state/ticket-queue/browse-history` (mode 0600, valid queries only, newest first, no duplicates, at most 100, merged with entries other running instances saved). The panel shows each board's source and scope read-only, so a missing ticket can be explained. Quick presets edit the query text.

**Detail pane.** Opens at once from the stored row. Sections, toggled by the digits 1 to 6: facts, description, score and eligibility (the content of `why`), links and blockers, comments (the only section read lazily, with `ticketctl get --comments N`, cached by `(site, key, updated)` with bounded entries), and claim and manual-order history. Text is sanitized and never parsed as markup; highlighting is bounded; link targets are shown.

**Refresh.** `r` re-reads the highlighted ticket live with `ticketctl get` and never opens the pane. `R` runs the incremental `refresh`. An automatic refresh runs every `browse.refresh_interval` and reads the store first. Every refresh takes the same lease as cron; a second caller just reads the store. Selection and scroll survive a refresh; a failed board marks its rows stale and keeps the data. Reads cancel between calls, tracked child process groups are killed on quit and the timer stops on every exit path.

**Actions** (each is also a CLI command and is run through it): `p` pin or unpin, `K` and `J` move, `x` skip, `c` claim and `u` release (both with `--holder browse`), `R` is refresh and `ctrl+r` re-ranks, `o` opens the ticket's browse URL (the built `https` URL from ticketctl's `url` only, with no userinfo, from a stripped environment, detached), `a` toggles ineligible tickets with their reasons, `/` filters, `?` shows help (a table generated from the bindings), `q` quits, `enter` opens the pane, and `S` selects the highlighted ticket to start (see below). Keys and help rows are tested to match.

**Result file.** With `--result-file`, on exit 0 the TUI writes `{"intent": "start"|"open"|"back", "selected": [{"key", "site", "board"}]}` (schema `schemas/cli/browse-result.json`); `start` means the user pressed `S` on a ticket and wants it started, `open` that the user opened the browse URL and `back` that the user quit. It carries ids only and no ticket text. Without the flag nothing is written.

The TUI runs correctly as a foreground child of another TUI, respects the terminal's colors and never hard-codes a background. Resizing is supported down to 10 columns.

## Upgrades

The state database is versioned; migrations are additive only, so a newer tool can read an older store, and an older tool reading a newer store exits 12 (`schema_newer`). `--json` outputs and hook payloads gain fields additively. Unimplemented commands in a release exit 12 `not_implemented`.

## Residual risks

- A prompt-injected ticket can steer the order of the tickets inside the re-rank's candidate set. The effects are bounded by validation (exact key set), by the permutation rule (a re-rank only reorders slots), by pins and hard rules, and by the fact that a ticket is only handed out for a human or a caller to act on. No re-rank output is acted on as an instruction.
- The sandbox's containment was not attacked in the design-time spike (the injected instructions were not obeyed, so the sandbox was never exercised); a live containment test is a still-to-verify item. `rerank.sandbox: preferred` runs the session without the sandbox when `bwrap` or `socat` is missing, with a warning and `sandbox_active: false` in the result, and relies on the permission layer.
- Ticket text is stored on disk (the store, the run directory while a re-rank runs). claude-session keeps a re-rank's session directory and input until its `retention_days` after `forget`, and Claude Code keeps a transcript with the fenced input under `~/.claude/projects` indefinitely; neither is removed by ticket-queue.
- A claim is local to one machine's store. Two machines (or two state directories) can hand out the same ticket; the caller's own write to the ticket system (an assignment) is what coordinates them.
- A report command containing `$CSESSION_DIR` is denied by claude-session and the model may then give up; the trusted prompt uses absolute paths, which reduces but does not remove the chance. A run that never reports ends `unreported` and falls back to the deterministic order.
- `blocked` and `estimate` are only as good as ticketctl's reading of links and the points fields; an ambiguous points field excludes every ticket on a board until it is mapped (`estimate_ambiguous` names the key to set).
- The cron-filled store can be up to `refresh.stale_after` old when read; `claim` re-reads the ticket live, but `list` and `next` without `--claim` show stored data.

## Platform notes

Linux and WSL (Ubuntu) are required. macOS is supported on a best-effort basis: there is no Linux-only code, the re-rank sandbox uses Seatbelt where claude-session supports it, and a macOS checklist is part of the live suite. File locking and the lease use portable primitives. Under WSL, `bubblewrap` and `socat` are needed for a sandboxed re-rank; WSL1 has neither, so with `preferred` the re-rank runs unsandboxed and with `required` it fails `sandbox_unavailable`.
