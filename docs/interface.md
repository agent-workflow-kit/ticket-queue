# ticket-queue: interface

Status: 0.1.0-design. Written before implementation (architecture §18.1). This document is the single normative source for ticket-queue's rules; `docs/design.md`, the README and the architecture link to its headings and never restate its tables. The `--json` shapes, hook payloads, config, log records and the re-rank documents are specified in [schemas/](../schemas/).

ticket-queue answers "which ticket should I do next?". It keeps a local pool of candidate tickets per board, read through ticketctl, decides which are eligible, scores them deterministically, optionally lets a model re-rank the top of the list, applies the user's pins and skips, and hands out the top ticket, optionally reserving it with a claim. It never writes to the ticket system: a claim is a local reservation, and status, assignee and sprint changes belong to the caller (through ticketctl).

## Concepts

### Identifiers

A ticket is identified by `(site, key)`. A key matches `^[A-Z][A-Z0-9_]*-[0-9]+$`, is accepted in any letter case and is printed upper-case. A ticket argument is `KEY` or `<site>:KEY`, where `<site>` is a ticketctl site name (`^[A-Za-z0-9][A-Za-z0-9_-]{0,63}$`); a browse URL is not accepted (a caller passes the key). `--json` output always carries `key` and `site`.

The site of a ticket argument is resolved in this order: `site:KEY`; else `--site`; else the one site whose store holds the key (several: exit 21 `ambiguous_key` with the candidates in `details.candidates`); for a key in no store that needs a live read: `ticketctl.default_site`, else the only site any board names; else exit 2 `invalid_argument`, reason `site_required`.

When a read returns a canonical key that differs from the stored one (a moved ticket), the store merges in the same transaction. With no row for the canonical key, the row is re-keyed. Otherwise a live claim on the canonical key wins and the other claim ends (`post-release`, reason `merged`), else the claim moves; the earlier pin position is kept; one skip remains (untimed beats timed, `auto` beats `human`); `consecutive_failures` takes the maximum; member rows are united. The old key is kept as an alias that resolves on input and is never a ticket row, and a search result whose key equals a stored alias is the canonical ticket. `requested_keys` (an array of aliases) is carried by the `claim`, `pin` and `skip` objects, by `claims` entries and by ticket hook payloads, and `ticket_state` is `moved` while the claim's original key is an alias. Every writer canonicalizes through one store function in its own transaction.

Other identifier grammars: a board name matches `^[a-z][a-z0-9_-]{0,63}$`; a label key matches `^[a-z0-9][a-z0-9_.-]{0,63}$` and a label value is 0 to 256 characters from `[A-Za-z0-9._:/#@+=,-]` (architecture §8.1; a label is split at the first `=`); a holder is 1 to 256 characters from the label-value set (`--holder ''` exits 2); an actor matches `^(human|auto):[A-Za-z0-9._@-]{1,64}$`; a decision id matches `^[A-Za-z0-9._:@/-]{1,128}$`; a duration matches `^[0-9]{1,6}(ms|s|m|h|d|w)$` in flags and config. Every identifier is matched with `fullmatch`. An identifier that fails its grammar exits 2 `invalid_argument`.

### Boards, pools and candidates

A **board** is a named entry in config. It names a ticketctl **site** and a **source** (a ticketctl named query, or structured flags), plus the board's goals, prioritized epics, parent allow and deny lists, eligibility rule settings, signal weights and re-rank settings. A **pool** is the set of tickets fetched for a board and stored. A **candidate** is a stored ticket that is eligible.

A ticket is in a board's pool when that board's last fetch or departure read returned it (a membership row records `first_seen` and `last_seen`). Its owner is the first board, in current config order, whose pool holds it; ownership is computed when read, and that board's goals, weights, epics and rules apply to it. A ticket in no pool is not a candidate; it stays stored while claimed or pinned and otherwise until `prune` (`retention.tickets` after `last_seen`), so `why` can still explain it. A ticket in no pool is evaluated with the built-in default rules and weights, no goals and no prioritized epics.

### Eligibility rules

Eligibility is decided locally on the stored ticket, not by the ticket system. Each ticket has a verdict per rule, `pass`, `fail` or `unknown`, and a rule that is `unknown` excludes the ticket just as `fail` does. The reason string of an exclusion is the rule name, or the rule's unknown reason. Rules are **soft** (a pin overrides them) or **hard** (nothing overrides them).

| Rule | Kind | Passes when | Unknown reason |
|---|---|---|---|
| `done` | hard | the stored `status.category` is not `done` | none |
| `gone` | hard | `gone_at` is not set (a ticket `get` reported as not found or forbidden sets it; a later read that returns the ticket clears it) | none |
| `hidden` | hard | neither the project matches `hidden.projects` nor any label matches `hidden.labels` (globs `*` and `?`, fullmatch, ASCII case-insensitive) | none |
| `claimed` | hard | no live claim exists | none |
| `skipped` | hard | no active skip exists | none |
| `status` | soft | `status.category` is in `rules.status_category` | none |
| `pointed` | soft | `estimate` is not null and `incomplete` does not contain `estimate` | `estimate_unknown` (when `incomplete` names `estimate`) |
| `sprint` | soft | per `rules.sprint`: `open` passes when a sprint with state `active` is in `sprints`; `open_or_future` also passes a `future` sprint; `any` always passes | `sprint_unknown` (when `sprints` is null because the field was not read) |
| `unblocked` | soft | `blocked.value` is false | `blocked_unknown` (when `blocked.value` is null) |
| `assignee` | soft | per `rules.assignee`: `me_or_none` passes an unassigned ticket or one assigned to the site's account; `me`; `none`; `any` | `assignee_unknown` (when `incomplete` names `assignee`), `me_unknown` (the comparison needs the site's account and none is stored) |
| `parent` | soft | the ticket's `epic.key` and `parent.key` are in none of the board's `parent_deny` keys, and, when `parent_allow` is not empty, at least one of them is in `parent_allow` | none |

"The site's account" is the `account_id` stored from `ticketctl whoami` for the ticket's site; a null `account_id` on either side never matches. With no stored account (`whoami` failed), the comparisons that need it are unknown with reason `me_unknown`, and a `reassigned` claim state is not reported.

A rule with its setting turned off (`false` or `any`) always passes. `claimed` and `skipped` are decided from the store, not from the ticket, at the `now` of the command. A pin overrides every soft rule and never a hard rule (see [Manual order](#manual-order)). `eligible` is true when every rule passes (for a pinned ticket: every hard rule passes).

A hidden ticket is never printed by `list`, `show`, `why`, `next` or the TUI and never reaches a re-rank. `claims` lists a hidden claimed ticket with `hidden: true` and its key only. `claim`, `pin` and `skip` of a hidden key exit 20 `refused`, reason `hidden` (a renewal included). The store writer drops a hidden read, and `prune` deletes hidden rows that are not claimed.

### Store

The pool, membership, claims, pins, skips, re-rank results and the hook outbox live in ticket-queue's private state database. Its only readers are ticket-queue's own commands. `list`, `show`, `why`, `claims`, `next` without `--claim` and `browse` read the store and never contact ticketctl.

Verdicts, signals, scores and the order are computed when read, from the stored ticket, the claims, pins, skips, membership and account tables, the current config and one `now` taken at the start of the command. A claim, pin or skip past its end is not live from that instant, whether or not a writer has removed it yet. `list`, `show`, `why`, `next`, `claims` and `boards` carry `refreshed_at` (a map of board to the board's last successful refresh) and `stale`, true when a pool is older than `refresh.stale_after`; state-changing outputs do not. The files and records are specified in [Files](#files).

## Commands

Global flags where meaningful: `--json`, `--config PATH`, `--site NAME`, `--verbose`, `--quiet`, `--actor <actor>`, `--decision-id <id>` (repeatable, accepted and logged), `--dry-run` (every command that changes local state), `--version`. `--site NAME` on `list`, `next`, `refresh`, `boards` and `claims` selects the boards on that site; on a ticket argument it resolves the site. Every command that outputs data supports `--json`; errors go to stderr and, with `--json`, to stdout as `{"error": {"code": "...", "message": "...", "reason": null, "details": {...}}}`. `error.code` is a closed class per exit code and `error.reason` is a separate field, null when not used (see [Exit codes](#exit-codes)). Text that can carry ticket or agent text (`--reason-file`) is taken as a file or `-`.

The default actor is `human:<me.name>` when stdin is a TTY, else `auto:unknown` (design guide §6). `--actor` is advisory. `--force` requires `/dev/tty` to open and stderr to be a TTY, else exit 2 `invalid_argument`, reason `human_only`. The log records of `release`, `skip`, `unskip`, `pin` and `move` carry `actor_source` (`flag` or `default`) and `tty`.

| Command | Purpose |
|---|---|
| `refresh [--board B] [--full] [--rerank]` | Read each board's pool through ticketctl and update the store |
| `list [--board B] [--all] [--skipped] [--claimed] [--query Q\|--query-file F] [--default-query] [--limit N]` | Print candidates in order |
| `show KEY` | Print one stored ticket with its verdicts and state |
| `why KEY` | Explain a ticket's score, verdicts and position |
| `next [--board B] [--claim] [--holder H] [--ttl D] [--label k=v]... [--query Q\|--query-file F] [--default-query] [--refresh]` | Print the top eligible ticket, optionally claiming it |
| `claim KEY [--holder H] [--ttl D] [--label k=v]... [--renew]` | Reserve a ticket, or renew a claim |
| `release KEY [--failed] [--holder H] [--force]` | End a claim |
| `claims [--holder H] [--label k=v]... [--board B]` | List live claims with their ticket state |
| `pin KEY [--top] [--expires D]` | Append a ticket to the pin list |
| `unpin KEY` | Remove a pin |
| `move KEY (--to N\|--top\|--before KEY\|--after KEY)` | Reorder within the pin list, pinning first when needed |
| `skip KEY [--for D\|--until T] [--reason-file F]` | Remove a ticket from candidates |
| `unskip KEY` | End a skip |
| `rerank [--board B] [--dry-run]` | Run the model re-rank |
| `boards` | List the configured boards and their state |
| `validate [--probe]` | Check config, plugin, ticketctl and claude-session; `--probe` also makes one real re-rank call |
| `cron install\|uninstall [--interval D]` | Manage the crontab block |
| `export [--since T] [--after-seq N]` | Print `log.jsonl` records |
| `prune [--dry-run]` | Remove old records |
| `browse [--focus KEY] [--board B] [--interval D] [--result-file PATH]` | Terminal UI |

`--board` takes one board; an unknown board exits 10. `--version` prints `ticket-queue <semver>`. Every command that changes local state (`refresh`, `claim`, `release`, `pin`, `unpin`, `move`, `skip`, `unskip`, `rerank`, `prune`, `next --claim`) supports `--dry-run`, which reports what it would change and writes nothing; `claim --dry-run` and `next --claim --dry-run` do the live reads and print the ticket they would claim. `unpin`, `unskip` and `release` are idempotent; `next --claim` is safe to retry, because each call claims at most one ticket. No command takes an external action, so no command is confirm-by-default and none exits 13 in 0.x.

A command that needs a ticket treats a key by this table. `show` and `why` never read live.

| Command | A key not in the store |
|---|---|
| `claim`, `pin`, `skip`, `move` | read live with `ticketctl get` and stored (in no pool); not found exits 10 `not_found`; a hidden key exits 20 `refused`, reason `hidden`; `pin` of a done or gone ticket exits 20 with that reason |
| `release`, `unpin`, `unskip` | exit 0 with `unchanged: true` |
| `show`, `why` | exit 10 `not_found` with `details.boards` (each board's name, site and source) and a message saying the ticket is in no pool |

### refresh

`refresh` is the only command that reads pools from ticketctl. It processes the selected boards (all enabled boards by default, each in its own lease, see [Concurrency](#concurrency)); a board whose lease is held by another process is skipped and the command exits 0 and says "already running" for it. Each board stores a `source_hash` (sha256 of its site and normalized source). On a mismatch the next refresh of the board is a full rebuild, clears `disabled` and deletes the board's stored re-rank result. A ticketctl named query's text is not visible to ticket-queue: after editing it, run `refresh --full`. A board that is removed, renamed or has `enabled: false` loses its member rows at the next `refresh` or `prune`; its claimed and pinned tickets stay (owner none).

Once per site per refresh run it first reads the account: `ticketctl whoami --site S --json` (`--refresh` with `refresh --full`); the account (`account_id`, `display_name`) is stored per site.

For each board it then does, in order:

1. **Fetch.** A named-query source runs `ticketctl search --query <name>`; a structured source runs `ticketctl search` with `--project` (`projects`), `--type` (`types`), `--label` (`labels`), `--assignee` (`assignee`, each of `none` and `me`), `--sprint` (`sprint`, each of `open` and `future`) and `--status-category new --status-category indeterminate` (a done ticket is never a candidate, so it is not fetched). Every search passes `--site`, `--limit 500`, `--fields key,title,type,status,assignee,reporter,priority,estimate,sprints,parent,epic,labels,due_date,created,updated,blocked,url,incomplete` (the `description` is not requested: a rich-text page could fail the whole board) and pages with `--cursor`, at most `refresh.max_pages` calls (default 20). An **incremental** refresh passes `--order-by updated:asc` and `--updated-since <watermark minus refresh.overlap>`; a **full** rebuild (`--full`, the first refresh of a board, a `source_hash` mismatch, or the board's `last_full_at` older than `refresh.full_every`) does not. The fetch holds every page in memory. A fetch is **complete** when ticketctl returned `next: null`, **truncated** after `refresh.max_pages` calls without it, and **failed** on any error. A failed fetch commits nothing: rows, member rows, the watermark and `last_full_at` are unchanged. Complete and truncated fetches are committed in one `BEGIN IMMEDIATE` transaction per board (steps 2 to 4). The ticketctl output is read through a 64 MiB cap; more is `lower_failed`, reason `output_too_large`.
2. **Upsert** each returned ticket by `(site, key)` through the store writer (hidden tickets are dropped, a moved key is canonicalized), store its normalized fields, and upsert this board's member row (`first_seen`, `last_seen`). A field named in the ticket's `incomplete` keeps its stored value. A stub (a ticket ticketctl could not parse) changes nothing but `incomplete` and sets `degraded: true`, which `show` prints. A search upsert never clears the stored `description`.
3. **Re-read departures.** `ticketctl get` (grouped by site, batches of 50, the fields of step 1 plus `description`) for every claimed or pinned ticket of the board and for the 100 tickets the board owns with the highest deterministic score among those passing the hard rules, soft rules ignored (this is the only way a ticket that left the pool is noticed, because `search` never reports departures). Each key is mapped on its own: a ticket updates the row (stores `description`, cut to 8 KiB, and `description_read_at`) and clears `gone_at`; `not_found` sets `gone_at`; `forbidden` sets `gone_at` with `gone_reason: forbidden`; any other per-key error leaves the row unchanged and is counted in `departures_unchecked`. An exit 10 or 12 with `details.errors` is read per key the same way; any other failure marks the whole batch unchecked. A ticket whose status category becomes `done` is stored as such (`state_since` records when a ticket became done or gone, for `claim.dead_grace`). A ticket outside the re-read set is corrected at the next full rebuild (`refresh.full_every`).
4. **Remove.** After a complete full rebuild (not truncated, not failed), delete this board's member rows for tickets it did not return. Tickets are not deleted here; a ticket with no member row, no claim and no pin is removed by `prune`.
5. **Re-rank** when `--rerank` is given, or when `rerank.auto` is true and the rules of [Re-rank triggers](#re-rank-triggers) allow it. A failed or skipped re-rank never fails the refresh.

The **watermark** is the time taken before the board's first call, after a complete fetch; after a truncated incremental fetch it is the largest `updated` returned. The overlap also absorbs up to 5 minutes of clock difference with the ticket system. A truncated full rebuild commits its upserts, skips step 4 and leaves `last_full_at` unchanged, so the next refresh is a full rebuild again; a board larger than `max_pages × 500` tickets is truncated on every run, so narrow its source.

A ticketctl failure of one board commits nothing for that board and marks it with the status and error below; the other boards still refresh.

| ticketctl code or reason | Board status | `error.code` | Rows | Retry |
|---|---|---|---|---|
| `rate_limited` | `refresh_failed` | `lower_failed` | kept | the board is not called until `retry_after` and stays `refresh_failed` |
| `cursor_expired`, `cursor_invalid` | `refresh_failed` | `lower_failed` | kept | restart without a cursor once, then fail |
| reasons `invalid_query`, `invalid_value`, `unknown_site`, `unknown_field`, any `invalid_config`; code `ambiguous_site` | `disabled` | `invalid_argument`, reason `invalid_config` | kept | until `source_hash` or the `ticketctl` config section changes |
| `auth_required`, `bad_credentials` | `refresh_failed` | `lower_failed` | kept | the message names `details.token_env` |
| `forbidden`, `unavailable`, `timeout`, `provider_changed`, `partial`, any other | `refresh_failed` | `lower_failed` | kept | next refresh |

`lower_error` is `{exit, code, reason}` copied from ticketctl (with `tool`). A board whose tickets all have `incomplete` naming `estimate` reports the warning `estimate_ambiguous` in its `warnings[]` with the ticketctl config key to set (`fields.estimate`). A board's `boards[]` entry in the output of `refresh` carries `already_running`, `mode` (`incremental` or `full`), `fetch` (`complete`, `truncated` or `failed`), `counts` `{fetched, upserted, members_removed, departures_checked}` and `departures_unchecked`.

Exit codes: 0 when every selected board refreshed (or was already running); 12 when any board is `refresh_failed`, `truncated` or `disabled`, with the normal output attached (`partial_results`, `details.boards`), and `details.lower_error` carrying ticketctl's exit and code for a failed board. `post-refresh` fires on changes (see [Hooks](#hooks)).

### list, show, why

`list` prints candidates for the selected boards in the order of [Ordering](#ordering), after the query filter, and always exits 0 (`empty` is true when nothing matched). The default set is the candidates; `--all` is every ticket of the selected pools (eligible first in Ordering, then ineligible by deterministic score, with `position` null), with their exclusion reasons; `--skipped` and `--claimed` select those sets (both together are their union; with `--all` the flag is redundant). `--limit` is 1 to 1000 (default 50), else exit 2; `total` counts after the query and before the limit. `--query` applies [Query language](#query-language) with `ORDER BY` allowed; with no `ORDER BY` the order is `ORDER BY position`. The configured `browse.query` is applied only with `--default-query` (on `list` and `next`).

`show KEY` prints the stored ticket (the fields of the `ticket` object of the schemas, plus `description` and `description_read_at`, and `degraded` for a stub), its verdicts, state (`pinned`, `claimed`, `skipped`) and `refreshed_at`; `show` is the only command that prints the description. `why KEY` prints, for a ticket: the board, each signal's raw value, normalized value, weight and contribution (terms to two decimals for display), the total, the verdict of each rule with the exclusion reason, the pin, move and skip state with warnings for soft rules a pin overrides, the consecutive failures and `last_failure_at`, the re-rank result (position, reason, age, whether it applied), the claim state, and the position with the effective score. All text from the ticket is sanitized (control, format and invisible characters become `<U+XXXX>` markers). Both read the store only.

### next

`next` prints the top eligible ticket of the selected boards in the order of [Ordering](#ordering). It takes `--query`/`--query-file` for filters only; a query containing `ORDER BY` exits 2 (`invalid_argument`, `reason: order_by_not_allowed`), so the ranking stays the contract of `next`. `--holder`, `--ttl` and `--label` without `--claim` exit 2 `invalid_argument`, reason `claim_required`. With `--refresh` it first refreshes the selected boards that are stale and proceeds on the committed rows; a partial refresh with a candidate is exit 0 with the warning `refresh_partial` and a `refresh` member holding the per-board results. With no eligible candidate it exits 22 (see [Exit codes](#exit-codes)).

With `--claim` it takes the claim by the steps of [Claims](#claims): take the first candidate; read it live with `ticketctl get`, up to 10 candidates per call, and check at most `claim.scan_limit` candidates (default 20), storing every result; re-evaluate its eligibility on the fresh ticket with all rules; if it fails, record `claim_skipped` in the output's `passed_over` list (`{key, site, outcome, reason}`) and take the next candidate; when one passes, write the claim in one transaction and print it. `next --claim` never returns a ticket with a live claim. A refresh older than `refresh.stale_after` does not block a claim. When none passes within the limit the command exits 22 with `passed_over` listed and, when the limit was reached, the warning `scan_limit_reached`. A `get` failure (ticketctl unavailable) exits 12 with `details.lower_error` and claims nothing.

### Ordering

The order of candidates is: first the **pin list** (in pin order) restricted to pinned tickets that pass the hard rules; then every other eligible ticket by **effective score** descending, then `created` ascending (older first, null last), then site, then key by project (ASCII) and number (numeric). `position` is the 1-based position in this order for the selected set (all boards by default); it is null for an ineligible ticket and for a ticket in no pool. A pinned ticket that fails a hard rule is `pinned_blocked`: it is ineligible, listed with `--all` and by `why`, never counted in `total`, and never handed out; `boards` counts them. The **effective score** is the deterministic [score](#scoring), except for tickets covered by an applied re-rank result (see [Re-rank](#re-rank)).

### Scoring

The score of a ticket is the sum over signals of `weight × signal`, each signal normalized to 0..1, using its owner board's weights. Terms are computed exactly (decimal), summed unrounded, and the total is rounded once, half up, to two decimals; ordering compares the rounded totals. A signal whose input is unknown contributes 0 and is shown as `unknown`; an absent optional value that is known absent (no due date) is 0 and shown as `none`. The order across boards follows the weights (raising a board's weights raises the board); `validate` warns `weights_differ` when the boards' weight sums differ. Defaults and formulas:

| Signal | Input | Normalization | Unknown when / none when | Default weight |
|---|---|---|---|---|
| `priority` | `priority.name` | `signals.priority_map` (entries overlay the defaults per name, case-insensitive; defaults Highest 1, High 0.75, Medium 0.5, Low 0.25, Lowest 0); a name not in the map falls back to `max(0, 1 - priority.rank / (signals.priority_levels - 1))` (`priority_levels` default 5) | null field: none (0); a name not in the map with `rank` null: unknown | 40 |
| `sprint` | `sprints[].state` | an `active` sprint 1, only a `future` sprint 0.3, none 0 (`signals.sprint.active`, `.future`) | `[]`: none; null: unknown | 25 |
| `epic` | the higher of the values for `epic.key` and `parent.key`, against the board's `prioritized_epics` (n entries) | the key at index `i` (from 0) gives `1 - i/n`; not listed 0 | never unknown | 20 |
| `due` | `due_date` | `d` is the due date minus today's UTC date, in days (negative when overdue): `d <= 3` gives 1, `d >= 30` gives 0, else `(30 - d) / 27` | no date: none | 20 |
| `assignee` | `assignee` | assigned to the site's account 1, unassigned or another person 0 | unknown when `incomplete` names `assignee` or the account is unknown | 10 |
| `age` | `created` | `min(days / 90, 1)`, days being today's UTC date minus `created`'s UTC date | `created` null: unknown | 5 |
| `labels` | `labels` | `min(sum of signals.label_bonus[label], 1)` | never unknown | 0 |
| `size` | `estimate` | `max(0, 1 - estimate / 13)` | `estimate` null: unknown | 0 |

The weights and signal settings are per board and overridable (see [Config](#config) for the bounds). `why` prints every term.

### Re-rank

Re-rank is optional (`rerank.enabled`, default false) and advisory: a failure, an expired result or no result leaves the deterministic order. It reorders only the tickets it was given.

**Candidates given.** The top `rerank.top_n` (default 20, at most 30) tickets the board owns, by deterministic score, that pass every rule and are not pinned. A board's re-rank covers only its own tickets; with no such ticket the run is skipped with reason `no_candidates`.

**Applying a result.** A stored, valid, unexpired result is applied in five steps:

1. Covered tickets are the result's keys that the board still owns, that pass every rule and are not pinned.
2. The slots are the current deterministic scores of the covered tickets, highest first.
3. The covered tickets, in result order (`ranking`, then `excluded` in deterministic order), take the slots in order.
4. The sort key is (effective score descending, result index ascending with uncovered tickets last, `created` ascending, site, key).
5. A dropped ticket's slot disappears; a new ticket keeps its deterministic score.

`rerank.position` is the 1-based index in `ranking`; an excluded ticket has `position` null and `excluded: true`; `ranking` may be empty. A result is applied only until `rerank.ttl` (default 24 hours) after it was made, and is shown as `rerank: expired` after that. `list --json` and `why` report whether a result applied, its age, and each ticket's re-rank position and reason. A re-rank never changes eligibility, the deterministic score or the pins.

Example: A has score 50 and was created in June, B has 50 and was created in May, C has 40; the model returns A, B, C. The slots are 50, 50, 40, so A takes 50, B takes 50 and C takes 40. The result index is a sort key after the effective score, so A (index 0) sorts before B (index 1) although B is older, and the order is A, B, C.

#### Re-rank triggers

`rerank [--board B]` runs a re-rank for the selected boards now. With the global `rerank.enabled` false the command exits 20 `refused`, reason `rerank_disabled`. With it true, a board whose own `rerank.enabled` is false is skipped with `rerank_disabled` in the output unless `--board` names it. `refresh --rerank` runs it after the refresh. With `rerank.auto: true` every refresh runs it, but only when the **input hash** differs from the last valid, unexpired result's. The hash is the sha256 of the canonical JSON of the candidate keys in deterministic order, the goals text, the prioritized epics, the skill, the model, `top_n`, `description_chars`, and the plugin directory hash (sorted relative paths plus bytes) or the `rerank.command` argv; ticket text and scores are excluded on purpose (cost). An expired result never blocks a run.

Every started run counts toward `rerank.min_interval` (default 1 hour) and `rerank.daily_limit` (default 8 per local day, the machine's local day). After a failed automatic run the next automatic attempt waits `min_interval`, doubling up to 24 hours, reset by a success. A blocked automatic run is logged as `rerank_skipped` with the reason (`unchanged`, `min_interval`, `daily_limit`, `rerank_disabled`, `no_candidates`, `backoff`). `post-rerank` fires only for failed runs, and a failed run logs `rerank` with `outcome: failed`. `--dry-run` prints the input document that would be given to the command and calls nothing. `next` and `list` never run a re-rank.

#### Slot contract

The re-rank slot is run by a command. `rerank.command` is an argv list run without a shell, or null (the default), meaning ticket-queue's bundled implementation (below). A custom command receives the **input document** on stdin and prints the **output document** on stdout; exit 0 means the output is to be used, any other exit is a failure. Ticket text never travels in argv or environment. The command runs in a private empty directory with a built environment (`PATH`, `HOME`, `LC_ALL=C` and the names in `rerank.env_allowlist`), under `rerank.timeout`, in its own process group which ticket-queue kills on expiry (group signals are allowed only for groups ticket-queue created, and a group is signalled only before its leader is reaped). Its stdout is read up to 32 KiB plus one byte (more: `output_invalid`), its stderr up to 4 KiB and shown sanitized, and stdin is written while stdout is read; its timeout is `deadline`. A `skill_marker` key in its output violates the exact key set. The command should exit on stdin EOF; a SIGKILL of ticket-queue does not stop it. `validate --probe` runs it on the sample (`sandbox_active` null).

The **input document** (schema `schemas/rerank-input.json`) is a JSON object: `schema_version` (1), `run_id` (32 random hex characters), `board`, `goals` (the board's trusted free text, at most 2000 characters, or an empty string), `prioritized_epics` (keys in order) and `candidates` (the top tickets in deterministic order: `key`, `position`, `deterministic_score`, `signals` (the normalized values), and the ticket fields `title`, `description`, `type`, `status`, `priority`, `estimate`, `sprint`, `epic`, `parent`, `labels`, `due_date`, `age_days`). The mapping is: `type` is `type.name`, `status` is `status.name`, `priority` is `priority.name` or null, `sprint` is `active` when an active sprint exists, else `future`, else `closed`, else `none`, and `age_days` is null when `created` is null. Every untrusted field is cut before the size is measured: `title` 200 characters, at most 20 labels of 64 characters, `epic.title` and `parent.title` 120, `description` `rerank.description_chars` (default 1000, maximum 1000). The ticket text fields (`title`, `description`, `labels`, the titles of `epic` and `parent`) are untrusted (`x-untrusted`).

The **output document** (schema `schemas/rerank-output.json`) is a JSON object with exactly the keys `schema_version` (1), `run_id` (echoed), `ranking` (best first: objects `{key, reason}`; it may be empty), `excluded` (optional, objects `{key, reason}`) and, for the bundled implementation only, `skill_marker`. `reason` is at most 200 characters and untrusted. The document is at most 32 KiB.

**Validation.** ticket-queue checks the output in this order, and any violation discards the whole result, keeps the deterministic order, logs `rerank` with `outcome: failed` and a reason from the table below, and fires `post-rerank`:

1. Read at most 32 KiB plus one byte; larger is `output_invalid`.
2. Decode strict UTF-8.
3. Check the nesting depth, iteratively, at most 64, before parsing.
4. Parse as a JSON object, rejecting duplicate keys at every level, `NaN`, `Infinity` and overflowing numbers.
5. Require the exact key set above; `schema_version` is an integer and not a boolean; `run_id` matches the run.
6. Match keys by `fullmatch` and compare `ranking` plus `excluded` to the candidate key set exactly (no invented, duplicate or missing key).
7. Sanitize every `reason` at ingest (a lone surrogate becomes a marker) and cut it to 200 characters after sanitizing; every `reason` is a string.
8. For the bundled implementation, require `skill_marker` equal to the value ticket-queue expects.

A re-rank failure is never a failure of the command that triggered it (`rerank` itself exits 12 with `rerank_failed`, see [Exit codes](#exit-codes)).

| Reason | Meaning |
|---|---|
| `tool_missing` | claude-session is not installed |
| `tool_too_old` | claude-session is older than 0.37.0 |
| `sandbox_unavailable` | `rerank.sandbox` is `required` and the sandbox cannot start |
| `launch_failed` | claude-session refused the launch (exit 12), or a name in `required_env_names` is not present; claude's own version problems arrive here |
| `unreported` | the run ended without a report (claude-session exit 14) |
| `blocked` | the run reported `blocked` or `needs-input` (exit 13) |
| `timeout` | claude-session reported its own timeout or turn limit (exit 16) |
| `session_failed` | any other non-zero claude-session outcome |
| `artifact_missing`, `artifact_invalid` | no artifact, or it failed parsing or its size check |
| `run_id_mismatch`, `marker_mismatch` | the echoed run id or skill marker is wrong |
| `keys_mismatch` | the key sets do not equal |
| `command_failed`, `output_invalid` | a custom command exited non-zero or printed an invalid document (or more than the output cap) |
| `input_invalid` | the input document exceeded its limits (a guard; the per-field caps make it unreachable by content) |
| `deadline` | ticket-queue's own guard expired (the `exec` guard, or a custom command's timeout) |

**Bundled implementation.** It uses claude-session directly (binding A, kind `rank`, headless, no agent-spawn). ticket-queue requires claude-session 0.37.0 or later and checks `claude-session --version` before the first run (`tool_too_old` otherwise). claude-session is called with `PATH`, `HOME`, the XDG variables, `LC_ALL=C` and the names in `claude_session.pass_env` (default `[]`; list `ANTHROPIC_API_KEY` there when Claude Code has no stored login), never with ticketctl's names. Before `exec`, every name in `start`'s `required_env_names` must be present, else the run fails `launch_failed` naming it and is stopped and forgotten. `validate` warns `pass_env_overlap` when a name is in both `pass_env` lists. Durations are passed to claude-session as whole seconds.

The sequence is: write the input to a private run directory (mode 0700, files created exclusively and never following links); `claude-session start --kind rank --mode headless --skill <rerank.skill> --prompt-file <prompt> --input-file <candidates.md> --plugin-dir <plugin directory> --permission-profile <profile file> --sandbox <rerank.sandbox> --setting-sources "" --env-allowlist-only <names>... --model <rerank.model> --timeout <rerank.timeout> --max-turns <rerank.max_turns> --label owner=ticket-queue --label rerank=<run_id> --actor auto:ticket-queue --json`; then `claude-session exec --session-id <id> --run <n> --outcome-file <private path> --json`, where `<n>` is `run` from `start --json`, guarded at `rerank.timeout` plus 10 seconds (on expiry: `stop <id> --grace 2s`, then SIGTERM to the exact `exec` child, then SIGKILL after 2 seconds); then `claude-session status <id> --json` to read `run.report.artifact` (`path`, `sha256`, `size`), `run.sandbox_active` and the session's `csession_dir`. The artifact path must lie under `csession_dir`; it is opened once without following links (a regular file, one link, owned by the user, size equal to the reported `size` and at most 32 KiB), and hashed and parsed from that descriptor's bytes. The validated result is copied into ticket-queue's store, then `claude-session forget <id>` runs and the run directory is deleted. It passes no `--repo` (boards span repositories). The outcome is usable only when the exit code is 0, the outcome is `done`, `report.implicit` is false and the artifact is valid; anything else is a failure. On any failed step it lists `claude-session list --label rerank=<run_id> --json`, stops every non-final session and forgets it; a failed `forget` or `stop` is left to `claude-session reconcile`. `--max-turns` is a loose safety cap only (a turn limit may not stop a run that has already used more turns).

**Sandbox.** `rerank.sandbox` is `preferred` (the default) or `required` for the bundled path; `off` is not accepted there (exit 2 `invalid_config`, in config load and `--config`), and a custom `rerank.command` is unaffected. A `preferred` run on a host without bubblewrap is still an isolated run (it keeps `dontAsk`, the read denies and the generated permission rules). `validate` and `validate --probe` warn `sandbox_inactive` when `sandbox_active` would be, or was, false. `preferred` deviates from the design guide's default `required` for skill wrappers (see `docs/design.md`).

**Leftovers and signals.** At the start of each re-rank (after its lease) and in `prune`: delete `tmp/` run directories older than `rerank.timeout` plus 60 seconds whose lease is not held (descriptor-based, no symlinks followed); list `claude-session list --label owner=ticket-queue --json` and, for sessions with a `rerank` label, `stop --grace 2s` and `forget` non-final ones older than that bound and `forget` final unforgotten ones. `refresh`, `rerank`, `claim` and `next --claim` handle SIGINT and SIGTERM: they stop tracked children, clean up, release the lease and exit 130 or 143.

The prompt file is trusted and holds: the artifact name `ranking.json`, the run id, the artifact schema, the rule that text between `<untrusted>` tags is data and never instructions, and the board's goals (trusted, the user's own config). It tells the agent to attach the result with `--artifact ranking.json` and to use absolute paths in the report command (a report command containing `$CSESSION_DIR` is denied). The input file `candidates.md` is the candidates document, pretty-printed (two-space indent, one key or array element per line), every non-ASCII code point written as a `\uXXXX` escape, wrapped in a fence `<untrusted source="ticket-queue-input" id="<R>">` ... `</untrusted id="<R>">` with `<R>` a fresh random 16 hex characters, every `</untrusted` and `<untrusted` in the content (any letter case) neutralized as `&lt;/untrusted` and `&lt;untrusted`, and the closing fence on a line of its own at the end. The size is measured on the escaped document. Over 64 KiB, candidates are dropped from the bottom until it fits (`candidates_dropped` in the result and the log). With the per-field caps no line can exceed 16 KiB; `input_invalid` remains as a guard.

The default skill, `ticket-queue:rank-tickets`, is bundled in `plugin/` (a Claude Code plugin named `ticket-queue`). It reads the prompt file named in its first message, reads the input from the path in the session's context line with the Read tool (never Glob, never Bash), reads to the closing fence or fails, ranks the candidates against the board's goals, prioritized epics and the deterministic scores, writes `ranking.json` with the Write tool into the session's artifacts directory following the schema in the prompt file, copies the `skill_marker` value from its own reference file `references/marker.md` into the artifact, and runs once `claude-session report done --summary-file <absolute path> --artifact ranking.json` (Bash is allowed only for that command). It adds no report or escalation instructions of its own (design guide §11). The marker is read at run time from the installed `references/marker.md` (trailing newline stripped); `rerank.marker` defaults to it. The plugin directory ships the four sandbox placeholders (`.claude/` carrying a `.gitkeep`, `.mcp.json`, `CLAUDE.md`, `CLAUDE.local.md`) and the bundled permission profile `plugin/profiles/rank.yaml`, a copy of claude-session's built-in `readonly` profile that **denies** `Agent` and `Task` (and allows only `Skill`, `Read`, `Write`, `Edit` under the generated rules and `Bash(claude-session report:*)`). Output validation is code: the schemas are not read at run time, and a test checks the code against them. `plugin.json` keeps the version `0.0.0`, and a packaging test asserts the four placeholders in the installed package.

A custom skill is selected with `rerank.skill` and, when it lives in another plugin, `rerank.plugin_dir` (default null, the bundled plugin), which is passed with `--plugin-dir`, checked like the bundled one (the placeholders, read-only) and hashed into the input hash. It must follow the same contract (input, artifact name, schema, run id echo). The marker check applies only to the bundled plugin's default skill; `rerank.marker: null` disables it (documented as lowering the check to the run id). The marker detects a skill that did not run; it is not authentication.

`validate --probe` runs one real re-rank on a built-in sample of three fake tickets and reports `probe_ok`, `sandbox_active`, the duration and the cost; `validate` without `--probe` checks the config, resolves the skill, the plugin placeholders and the profile file, and runs `ticketctl --version`, `claude-session --version`, and `whoami` and `fields check` per configured site, without a model call. `--dry-run` of a launch is not a self-test: it reports a healthy sandbox for setups in which every report call still fails (a missing placeholder, an interpreter reachable only through an alias).

### Claims

A claim reserves a ticket so it is not handed out twice. It is local: it never writes to the ticket system. A claim has: `(site, key)` (unique per ticket), `claimed_at`, `renewed_at`, `expires_at`, `ttl_s`, an optional `holder` and labels; the claim's `board` is the ticket's current owner (not stored on the claim). `consecutive_failures` is kept on the ticket, not the claim, across claims.

`holder` is an explicit opaque string (`--holder`), never inferred from the caller; it matches the holder grammar. Labels follow the usual `--label key=value` rule (a multiset; adding an identical pair is a no-op). A caller holds a claim when both holders are present and equal. A claim without a holder is never renewed (`claim KEY` on it exits 20 `refused`, reason `claimed`); `release KEY` without `--holder` ends it. A claim with a holder is released only with the same `--holder` or `--force`.

**Taking a claim** (`claim KEY`, `next --claim`) follows these steps:

1. Resolve the key and site (see [Identifiers](#identifiers)).
2. Read the ticket live with `ticketctl get` (the fields of [refresh](#refresh) step 1 plus `description`). Not found: a key not stored exits 10 `not_found`; a stored key is marked gone and exits 20 `refused`, reason `gone` (`next --claim` passes over it).
3. Canonicalize a moved key.
4. In one `BEGIN IMMEDIATE` transaction: store the fresh ticket through the writer; evaluate the rules on it with the live claim, skip, pin and hidden state at `now` (the hard rules for `claim KEY` and for pinned tickets, all rules for `next --claim`; a holder's renewal is refused only for `gone` and `hidden`, see below); a live claim by another holder refuses (`claimed`); write or renew the claim; write the outbox event; commit.
5. `SQLITE_BUSY` after the busy timeout retries steps 2 to 4 once, then exits 12 `unavailable`, reason `contended`.

`claim KEY` on an explicit key refuses only on hard rules: exit 20 `refused` with reason `done`, `gone`, `skipped`, `hidden` or `claimed` (held by another holder or by a caller without a holder). Soft-rule failures are `warnings` in the result, because naming a ticket is the user's intent; a key outside every board's pool is allowed. A holder's renewal (`claim KEY` by the claim's holder, and `claim KEY --renew`) sets `expires_at` to `now` plus `--ttl`, else the claim's stored `ttl_s`, adds labels and reports `renewed: true`. A renewal succeeds when the ticket is now `done` or skipped, with the warnings `ticket_done` and `ticket_skipped`, and is refused only for `gone` (exit 20, reason `gone`); a `reassigned` ticket renews with a warning. A new claim still refuses every hard rule. `claim KEY --renew` renews only: with no live claim held by this holder it exits 20 `refused`, reason `not_held` (the idiom for periodic renewal). A claim held by a different holder exits 20 `claimed`.

**TTL.** A claim is live while `expires_at > now`; `claim.ttl` (default 72 hours; `--ttl` per claim, at most `claim.max_ttl`, default 30 days) applies to a claim that is not renewed. Writing commands (`claim`, `release`, `next --claim`, `pin`, `skip`, `refresh`, `prune`) remove lapsed claims inside their own transaction, log `release` with reason `lapsed` and enqueue `post-release`. Read commands never write. A same-holder `claim KEY` on a lapsed claim creates a new claim (`renewed: false`). `prune`'s `dead_grace` release re-checks inside its transaction.

**Release.** `release KEY [--failed] [--holder H] [--force]` is idempotent: releasing an unclaimed ticket exits 0 with `unchanged: true`. A holder that differs from the claim's exits 20 `refused`, reason `holder_mismatch`; only a human `--force` (actor `human:`, see [Commands](#commands)) overrides it. A normal release sets the ticket's `consecutive_failures` to 0. A failure is counted only when `release --failed` ends a live claim held by that holder; on an unclaimed ticket it is `unchanged` and counts nothing. It adds 1 and records `last_failure_at` (shown by `why`); the auto skip is created when the count is at least `claim.max_failures` (default 3; a lowered limit applies at the next failure), in the same transaction that ends the claim, with `source: auto` and a reason, replacing an existing skip (a ticket has at most one skip), and `post-release` is written before `post-auto-skip`. The skip lasts until `unskip`. Any `unskip` resets the count to 0; a lapse or a `prune` release leaves it unchanged.

**Claim state.** `claims` and `list --claimed` report `ticket_state` for each claim: `ok`, `done`, `gone`, `reassigned` (the assignee is neither the site's account nor empty) or `moved` (the claim's original key is an alias of the canonical key). A refresh never releases a claim. `prune` releases a claim whose ticket has been `done` or `gone` for more than `claim.dead_grace` (default 7 days, counted from `state_since`), with a log entry; this is the only automatic release besides a lapsed TTL. A claim never excludes the ticket from `show` or `why`.

`claims` lists live claims ordered by `claimed_at`, filtered by `--holder`, `--label k=v` (repeatable, all must match) and `--board`. An entry is `{site, key, requested_keys, board, holder, labels, claimed_at, renewed_at, expires_at, expires_in_s, ttl_s, ticket_state, ticket: {title, status, url}}`; `ticket` is never null.

### Manual order

**Pins.** The pin list is an ordered, global list of tickets above all scoring and re-rank; `list --board B` shows B's pinned tickets in global order. `pin KEY` appends the ticket (`--top` puts it first); `--expires D` sets an expiry. A `pin` of a pinned ticket is `unchanged`, except that `--top` moves it first and `--expires` replaces the expiry. `unpin KEY` removes it. `move KEY` takes `--to N` (1-based), `--top`, `--before KEY` or `--after KEY` (the anchor must be pinned) and reorders within the pin list, pinning the ticket first when it was not pinned; `--to N` places at N (N larger than the list: last; N below 1: exit 2), and an anchor equal to the ticket exits 2. `pin_position` is a ticket's index in the pin list (it is never called `position`, which only means the ranking), renumbered 1..n after every change. A pin has no timer unless `--expires` is given, and it is kept through a normal release. A pin past its expiry is not pinned when read. The next writer drops a pin whose ticket became `done` or `gone` or whose expiry passed (`pin_dropped`, reason `done`, `gone` or `expired`); until then a pinned ticket that fails a hard rule is `pinned_blocked`: it stays stored, is shown with its reason, and is never handed out. A pin overrides the soft rules; `why` lists each overridden soft rule as a warning (`soft_rule_overridden`).

**Skips.** `skip KEY [--for D|--until T] [--reason-file F]` removes the ticket from candidates. A skip with no time lasts until `unskip KEY`; a timed skip ends by itself (`until` is an RFC 3339 instant). `source` is `human` or `auto`; an auto skip comes from `release --failed` (see [Claims](#claims)) and never times out. `--reason-file` text is untrusted and shown sanitized. `list --skipped` lists both kinds. A skipped ticket fails the hard rule `skipped`.

Pin, move and skip changes are written to `log.jsonl` with the actor. They are not external actions and have no audit record.

### Query language

One engine, in one UI-free module, serves `list --query`, `next --query`, the TUI panel, `browse.query` and the quick presets. A query is text of at most 1000 characters (a word at most 200, nesting at most 20, at most 50 conditions, an `IN` list at most 50 entries); control characters are refused; there is no eval and no regex (`LIKE` uses an iterative matcher). It narrows the stored pool and never widens it. A syntax error exits 2 `invalid_argument` with `reason: bad_query` and the position.

Grammar: `expr := term (OR term)*`; `term := factor (AND factor)*`; `factor := NOT factor | '(' expr ')' | condition | boolean-field`; `condition := field op value | field IN '(' value (',' value)* ')' | field NOT IN (...) | field IS [NOT] NULL | field [NOT] LIKE string`; ops `= != <> < <= > >=`; `query := [expr] [ORDER BY key [ASC|DESC] (',' key [ASC|DESC])*]` with at most 8 keys. Keywords and field names are case-insensitive. A string is single-quoted with `''` for a quote. A duration is `[0-9]+w`, `d`, `h`, `m` terms each used at most once in that order (`1w2d`, `90m`, `12h30m`), and a literal may be negative (`-2d`); a number is an integer or decimal and may be negative; a number compared with a duration field exits 2 `bad_query`. `me` (unquoted) is the account id of the ticket's own site and matches `assignee` and `reporter`; text comparison of `assignee` uses `display_name`; `'me'` is text. `true` and `false` are booleans. An unquoted word that is not a keyword, field, `me`, `true` or `false` exits 2 `bad_query`.

| Field | Type | Notes |
|---|---|---|
| `key`, `project`, `title`, `status`, `category`, `type`, `priority`, `assignee`, `reporter`, `epic`, `parent`, `sprint`, `board`, `holder` | text | `category` is the status category; `epic` and `parent` compare their keys; `sprint` the names of the ticket's sprints (any); `holder` the claim's holder |
| `labels`, `claim_label` | set of text | `labels = 'x'` means the set contains `x`; `!=` means it does not; `IN (...)` means it contains any; `LIKE` matches any member; `claim_label` is the claim's labels as `k=v` |
| `text` | free text | `LIKE` only; over `title`, `key` and `labels` (matches when any matches) |
| `estimate`, `score`, `position`, `rerank_position`, `failures` | number | `score` is the effective score, `failures` the consecutive failed releases; `position` is the position in the full order of the selected boards before the query |
| `age`, `updated`, `due` | duration | `age` since created, `updated` since last update, `due` the signed time until the due date (negative when overdue) |
| `mine`, `pointed`, `blocked`, `claimed`, `pinned`, `skipped`, `eligible`, `overdue` | boolean | usable bare (`claimed`, `NOT blocked`); `mine` is `assignee = me`, `pointed` and `blocked` read the raw fields with the rule's unknown cases, `overdue` is `due < 0` |

Text `=`, `!=`, `IN` and `LIKE` are ASCII case-insensitive for every text field. `LIKE` uses `%` and `_` with `\` as the escape. Evaluation is three-valued (Kleene logic, as in SQL). A known absent value (unassigned, no due date, no epic or parent, no sprint, empty labels) is null: `=` is false, `!=` is true and `IS NULL` is true. An unknown value (named in `incomplete`, or not loaded) makes a condition on it unknown and `IS NULL` true; `NOT unknown` is unknown; a row is kept only when the whole expression is true. `sprint != 'x'` is `NOT sprint = 'x'`. `ORDER BY` accepts scalar fields only (not `labels`, `text`, `claim_label`); booleans sort false before true; a repeated key exits 2 `bad_query`; an unknown value sorts last in both directions, and ties end on `(site, key)`. `eligible` is never unknown for a stored ticket. A key that is not a field exits 2 `unknown_field`.

Bounds: each `NOT` counts toward nesting 20; a `LIKE` pattern is at most 64 characters with at most 4 `%`; compared text is its first 512 characters; an evaluation budget of 2,000,000 character comparisons applies, then exit 2 `invalid_argument` with reason `too_expensive` (the TUI shows the error and evaluates off the UI thread).

The **named order** `position` is the ranked order of [Ordering](#ordering). Quick presets of the query panel are query texts: `m` mine (`mine`), `d` due soon (`due < 7d`), `b` blocked (`blocked`) and `c` claimed (`claimed`); they work inside the panel only.

### Concurrency

Commands run by cron or the harness are safe under concurrent invocation. The store uses WAL and a 5-second busy timeout. `refresh` and `rerank` take a per-board lease (holder PID and process start time, plus an expiry); a second caller exits 0 and reports "already running" for that board. A lease expires 120 seconds after its last heartbeat (every 30 seconds); a holder is dead when its PID is gone or its start time differs, and a lease whose holder is dead may be taken over. A claim is one `BEGIN IMMEDIATE` transaction in which every check is re-run and the claim row (unique per ticket) is the arbiter. `next`, `list`, `show`, `why`, `claims` and `browse` never wait for a refresh: they read the last committed rows.

### Hook delivery

Post-hooks are delivered through an outbox: every event is written to the store in the transaction that caused it, with an `event_id`, a `seq` and one row per hook entry, and delivered at least once after the commit. Each event is delivered per hook entry. Writing commands and `refresh` deliver after printing their result: at most 10 deliveries or 5 seconds per command (`refresh`: 30 seconds). A delivery first takes the event's lease (`lease_until = now + timeout + 5 s`) in a transaction, so two processes never run the same delivery at once. A failure retries with backoff from 1 minute doubling to 6 hours. Read commands never deliver. An `async` entry is delivered when started. Events are delivered in `created_at` order, and `release --failed` writes `post-release` before `post-auto-skip`; delivery order is not guaranteed, and consumers order by `seq`. A receiver deduplicates on `event_id` and tolerates unknown payload keys. A failed hook is logged and warned and never fails a command. An event still undelivered after `retention.events` (14 days) is deleted by `prune` with a final `hook_failed` log record (`final: true`). `boards` and `validate` report `undelivered_events` meanwhile.

## Exit codes

Reserved codes follow design guide §4.2.

| Code | Meaning in ticket-queue |
|---|---|
| 0 | success (including "already running" and `unchanged`; `list` always exits 0) |
| 1 | unexpected internal error (`internal`) |
| 2 | usage error (`invalid_argument`, with a `reason` such as `bad_query`, `unknown_field`, `order_by_not_allowed`, `invalid_config`, `site_required`, `claim_required`, `human_only`, `too_expensive`, `invalid_interval`) |
| 10 | not found: a key in no store, an unknown board (`not_found`) |
| 11 | not used in 0.x (there are no pre-hooks) |
| 12 | a lower tool failed, a refresh was partial, the store was contended or the tool cannot do it (`partial_results`, `lower_failed`, `rerank_failed`, `unavailable` with reason `contended`, `tui_missing`, `tool_missing`, `tool_too_old`, `schema_newer`, `not_implemented`; `output_too_large` is a reason of `lower_failed`) |
| 13 | not used in 0.x |
| 14 to 19 | not used |
| 20 | refused (`refused`), with `reason` in {`claimed`, `done`, `gone`, `skipped`, `hidden`, `holder_mismatch`, `not_held`, `rerank_disabled`} |
| 21 | ambiguous key (`ambiguous_key`; several sites hold it); `details.candidates` lists them |
| 22 | `next` (with or without `--claim`) found no candidate; `list` never exits 22 |

Output by exit code: 0, 12 (when partial) and 22 print the normal object (`next` prints `{"empty": true, ...}` with `--json` for 22); 2, 10, 20, 21 and a total 12 print the error envelope. A tool-specific code is documented here and in the README.

| Command | Exit rules |
|---|---|
| `validate` | 0 when `ok`; 12 `partial_results` with the normal output when any check fails; 2 for an invalid config |
| `boards` | always 0 |
| `refresh` | 12 when a board is `refresh_failed`, `truncated` or `disabled` |
| `next --refresh` | a partial refresh with a candidate: 0 with the warning `refresh_partial` and a `refresh` member holding the per-board results |
| `rerank` | several boards with some failed: 12 with the normal output (`results[]`); all selected boards failed: 12 error envelope `rerank_failed` with `details.results` |
| `cron uninstall` | no block: 0 with `removed: false` |
| `export` | 0 |

## `--json` outputs

Schemas live in [schemas/cli/](../schemas/cli/). Every output is a JSON object (never a bare array; `export` prints JSON lines, exempt from this rule, each validated by `log.json`), validated against its schema in tests. `list`, `show`, `why`, `next`, `claims` and `boards` carry `refreshed_at` (a map of board to timestamp or null) and `stale`; state-changing outputs do not. Multi-board outputs carry `boards` (each with `status`: `ok`, `refresh_failed`, `truncated`, `disabled`; `disabled` means a configuration error or `enabled: false`, and `rerank_disabled` is never a board status). Outputs carry `complete` (false when a board is failed or truncated) and `errors`. Ticket text fields (`title`, `description`, `labels`, `epic.title`, `parent.title`, display names, reasons) are `x-untrusted` in the schemas and printed sanitized. Timestamps are UTC `YYYY-MM-DDTHH:MM:SSZ`.

The set of fields of a ticket object follows ticketctl's normalized ticket (`key`, `site`, `url`, `title`, `type`, `status`, `assignee`, `reporter`, `priority`, `estimate`, `sprints`, `parent`, `epic`, `labels`, `due_date`, `created`, `updated`, `blocked`, `incomplete`) plus ticket-queue's own `board`, `score`, `position`, `rerank`, `eligible`, `verdicts`, `claim`, `pinned`, `skipped` and `warnings`. `board`, `score` and `position` are null for a ticket in no pool; `rerank` is `{applied, position, excluded, reason, age_s}`; `verdicts` is an object keyed by rule name, each `{result: pass|fail|unknown, reason, overridden_by_pin}`; `claim`, `pin` and `skip` are the objects or null, and `pinned_blocked` is `pin.blocked`. `description` is printed only by `show` (with `description_read_at`). Warning codes are the enum of `defs.json`: `soft_rule_overridden`, `reassigned`, `estimate_ambiguous`, `departures_unchecked`, `sandbox_inactive`, `refresh_partial`, `scan_limit_reached`, `me_unknown`, `pinned_blocked`, `ticket_done`, `ticket_skipped`, `weights_differ`, `pass_env_overlap`, `browse_query_invalid`.

The fields of each command output, copied from the schemas:

- `list`: `tickets`, `count`, `total`, `limit`, `empty`, `query`, `rerank` (per board), `boards`; `empty` is true when nothing matched (exit 0).
- `show`: `ticket` (with description). `why`: `ticket`, `board`, `signals`, `deterministic_score`, `effective_score`, `position`, `eligible`, `verdicts`, `reasons`, `pin`, `skip`, `claim`, `claim_state`, `failures`, `rerank`, `rerank_summary`, `warnings`.
- `next`: `empty`, `ticket`, `claim`, `passed_over`, `refresh` (with `--refresh`), `warnings`, `boards`. `claim`: `ticket`, `claim`, `renewed`, `warnings`.
- `release`: `key`, `site`, `released`, `unchanged`, `failed`, `forced`, `consecutive_failures`, `auto_skip`.
- `pin`, `unpin`, `move`, `skip`, `unskip`: `key`, `site`, the record or null, `unchanged`, and for pins `pins` (the list, with `pin_position`); `move` adds `pinned_first`; `unskip` has `ended`.
- `claims`: `claims`, `count`, `boards`. `boards`: `count`, `undelivered_events`, `boards` (per board: `status`, `refreshed_at`, `stale`, `last_full_at`, `pool`, `eligible`, `claimed`, `pinned`, `pinned_blocked`, `skipped`, `source`, `error`, `retry_after`, `lower_error`, `warnings`, `rerank`).
- `validate`: `ok`, `checks[]` (`{name, ok, message}`), `claude_session_version`, `ticketctl_version`, `probe`, `undelivered_events`, `warnings`.
- `refresh`: `mode`, `boards` (see [refresh](#refresh)), `dry_run`. `prune`: counts per kind. `cron install`: `installed`, `changed`, `interval`, `block`; `cron uninstall`: `removed`. `rerank`: `results[]`, and `inputs[]` with `--dry-run`.

Additive changes (new optional fields) are allowed within a minor version; removals and renames are breaking (design guide §13).

## Hooks

Post-hooks only in 0.x (no `pre-` hooks; a `pre-claim` hook is a 1.x item). Hooks are configured in `hooks:` (design guide §8): each entry is `{cmd, timeout, async}` (timeout: integer seconds 1 to 600 or a duration, default 30 seconds), run via `sh -c` in its own process group with the JSON payload on stdin and `TICKET_QUEUE_HOOK`, `TICKET_QUEUE_ID` and `TICKET_QUEUE_OBJECT_TYPE` in the environment; the hook's environment is the tool's environment minus the names in `ticketctl.pass_env` and `claude_session.pass_env`. An unknown hook name is `invalid_config`. Payloads (schemas in `schemas/hooks/`) carry the common keys `tool`, `hook`, `event_id`, `event_ts`, `seq` (monotonic per store), `object_type` (`ticket` or `board`), `object_id` (`<site>:<KEY>` or the board name), `board`, `key`, `site`, `holder`, `labels` (array of `k=v`) and `requested_keys`, null or empty for board hooks, never ticket text. Payloads omit `repo` (ticket-queue has no repository notion), so a gate matching on a payload lands in scope `*`. Delivery is described under [Hook delivery](#hook-delivery).

| Hook | Fires | Notes |
|---|---|---|
| `post-claim` | after a new claim is written, or a renewal that adds a label (`claim`, `next --claim`); a plain renewal fires nothing (the log keeps it) | `renewed`, `claimed_at`, `expires_at`, `ttl_s` |
| `post-release` | after a claim ends by `release`, a lapsed TTL, `prune`, `--force` or a merge | `failed`, `reason` (`released`, `lapsed`, `pruned`, `forced`, `merged`), `forced`, `actor`, `claimed_at` |
| `post-auto-skip` | after `release --failed` reached `claim.max_failures` | `skip_source`, `failures`, `caused_by` (the `post-release` event id) |
| `post-refresh` | after a refresh in which a board's status or error code changed (recovery included) or a claimed ticket's state changed (either direction), and at most once per `hooks.refresh_repeat` (default `6h`) while a board stays failed | `boards[]` `{board, status, previous_status, error_code, lower_code, retry_after}`; `changed_claims[]` `{key, site, holder, labels, requested_keys, ticket_state, previous_state}` |
| `post-rerank` | after a re-rank run failed (automatic skips are log-only) | `outcome: failed`, `reason` from the table, `run_id` |

## Config

YAML at `~/.config/ticket-queue/config.yaml` (respecting `XDG_CONFIG_HOME`), plus `config.d/*.yaml` drop-ins merged in lexical order after it (maps merge; hook lists append). Boards are ordered as written in `config.yaml`, then boards first defined in drop-ins, in lexical file order; a drop-in that overrides an existing board keeps its slot. The schema is [schemas/config.json](../schemas/config.json); `config.example.yaml` documents every key and its default. Invalid config fails fast naming the key (exit 2, `invalid_config`); a board with no `site` and no `ticketctl.default_site` is `invalid_config` naming the board. Structured config (`boards`, `hooks`, maps) cannot come from the environment; only top-level and nested scalars can. Precedence: flags, then `boards.<name>`, then environment variables, then global config with drop-ins, then built-in defaults; the global `rerank.enabled` is a gate for every board. ticket-queue has no notion of a repository, so the board takes the per-repo position of the precedence order (design guide §5). Epic and parent keys are case-insensitive and stored upper-case. Structured `labels` of a source must all match; the other lists match any.

| Key | Default | Meaning |
|---|---|---|
| `me.name` | OS username | the actor name (`human:<me.name>`) and the TUI holder suffix; there is no `me.jira`: identity is the account id of `ticketctl whoami` |
| `ticketctl.config` | null | an explicit ticketctl `--config` path |
| `ticketctl.pass_env` | `[]` | names of environment variables handed to the ticketctl child (in addition to `PATH`, `HOME`, the XDG variables and `TICKETCTL_*`) |
| `ticketctl.default_site` | null | the site for a board that names none |
| `claude_session.pass_env` | `[]` | names handed to claude-session children (in addition to `PATH`, `HOME`, the XDG variables and `LC_ALL=C`); list `ANTHROPIC_API_KEY` when Claude Code has no stored login |
| `boards.<name>.enabled` | `true` | false ignores the board (its member rows go at the next refresh or prune) |
| `boards.<name>.site` | `ticketctl.default_site` | the ticketctl site |
| `boards.<name>.source` | required | `{query: <name>}` or any of `projects`, `types`, `labels`, `assignee` (`none`, `me`), `sprint` (`open`, `future`) |
| `boards.<name>.goals`, `goals_file` | empty | the board's free-text goals (trusted: the user's own text), at most 2000 characters; `goals` or `goals_file`, not both; `goals_file` is relative to the config file's directory, read at each use, at most 16 KiB |
| `boards.<name>.prioritized_epics` | `[]` | epic keys in priority order |
| `boards.<name>.parent_allow`, `parent_deny` | `[]` | parent or epic keys |
| `boards.<name>.rules` | `status_category: [new]`, `pointed: true`, `sprint: open`, `unblocked: true`, `assignee: me_or_none` | each rule can be turned off with `false` or `any`; `sprint` takes `open`, `open_or_future`, `any`; `assignee` takes `me_or_none`, `me`, `none`, `any`; `status_category` is a list of `new`, `indeterminate` |
| `boards.<name>.weights` | the defaults of [Scoring](#scoring) | per-signal weights, 0 to 1000 |
| `boards.<name>.signals` | the defaults of [Scoring](#scoring) | `priority_map` (values 0 to 1), `priority_levels` (at least 2), `sprint.active` and `.future` (0 to 1), `label_bonus` (values 0 to 1) |
| `boards.<name>.rerank` | `enabled: null`, `top_n: null` | per-board overrides; a board re-ranks when the global `rerank.enabled` is true and its own is not false; null `top_n` is the global value |
| `claim.ttl`, `claim.max_ttl` | `72h`, `30d` | claim lifetime without renewal |
| `claim.max_failures` | `3` | consecutive failed releases before an auto-skip |
| `claim.dead_grace` | `7d` | age of a done or gone claimed ticket before `prune` releases it |
| `claim.scan_limit` | `20` | candidates `next --claim` live-checks at most |
| `refresh.stale_after` | `30m` | pool age shown as stale |
| `refresh.full_every` | `24h` | full rebuild interval |
| `refresh.overlap` | `5m` | incremental refresh overlap |
| `refresh.max_pages` | `20` | ticketctl search calls per board and refresh |
| `rerank.enabled` | `false` | the global re-rank switch (a gate for every board) |
| `rerank.command` | `null` | a custom argv list; null is the bundled implementation |
| `rerank.skill`, `rerank.marker` | `ticket-queue:rank-tickets`, the bundled value | the skill and the expected marker (`null` disables the marker check) |
| `rerank.plugin_dir` | `null` | a custom plugin directory (null: the bundled plugin) |
| `rerank.model` | `haiku` | passed to claude |
| `rerank.top_n`, `rerank.description_chars` | `20`, `1000` | candidates given (at most 30) and description cut (at most 1000) |
| `rerank.ttl`, `rerank.min_interval`, `rerank.daily_limit` | `24h`, `1h`, `8` | result lifetime and automatic-run limits |
| `rerank.auto` | `false` | run after each refresh when the input changed |
| `rerank.permission_profile` | the bundled `rank.yaml` | a profile file replacing the bundled one |
| `rerank.sandbox` | `preferred` | `required` or `preferred` (`off` is rejected for the bundled path) |
| `rerank.setting_sources` | `[]` | setting sources passed with `--setting-sources` (`user`, `project`, `local`) |
| `rerank.env_allowlist` | `[PATH]` | names passed with `--env-allowlist-only` |
| `rerank.max_turns`, `rerank.timeout` | `12`, `150s` | loose turn cap and wall-clock bound |
| `hidden.projects`, `hidden.labels` | `[]` | glob lists (`*` and `?`) of projects and labels never shown; a hard rule (see [Eligibility rules](#eligibility-rules)), always on, with no reveal |
| `browse.refresh_interval` | `120s` | automatic refresh in the TUI (0 turns it off) |
| `browse.detail_collapsed` | `[]` | detail sections collapsed at start |
| `browse.query` | `null` | the default query; the TUI writes it to `config.d/90-browse-query.yaml`, which overrides `browse.query` in `config.yaml`; validated when used (invalid: warning, ignored) |
| `cron.env_file` | `null` | a user-owned env file sourced by the cron line |
| `hooks.<name>` | `[]` | hook entries (an unknown name is an error) |
| `hooks.refresh_repeat` | `6h` | minimum time between repeated `post-refresh` events while a board stays failed |
| `log.max_bytes`, `log.keep` | `10MiB`, `3` | `log.jsonl` rotation (`^[0-9]{1,9}(B\|KiB\|MiB\|GiB)$`) |
| `retention.tickets`, `retention.events` | `30d`, `14d` | `prune` ages |

A durations value is a string such as `30m`, `24h` or `7d`.

## Environment variables

Config values from the environment are named `TICKET_QUEUE_<KEY>` (upper-cased, nested keys joined with `__`, e.g. `TICKET_QUEUE_REFRESH__STALE_AFTER`). The hook variables are `TICKET_QUEUE_HOOK`, `TICKET_QUEUE_ID` and `TICKET_QUEUE_OBJECT_TYPE`; these three names are never config keys. ticket-queue reads no credentials: ticketctl's own variables are named in ticketctl's config, and ticket-queue hands them to the ticketctl child only when they are listed in `ticketctl.pass_env`; claude-session children get only `claude_session.pass_env` (never ticketctl's names). `XDG_CONFIG_HOME` and `XDG_STATE_HOME` are respected.

## Trust boundary

Allowing ticket-queue allows what its config can do: hooks, `rerank.command`, cron with `cron.env_file`, and `ticketctl.config` with `pass_env`. A constrained caller must fix `--config`, scrub the environment and allow only `list`, `show`, `why`, `next` without `--claim`, `claims`, `boards` and `export`. Hook environment stripping is a boundary only when the caller cannot choose the config. `ticketctl` and `claude-session` are resolved on `PATH`, like the siblings; there are no binary-path keys.

Files read from user-supplied paths (`--query-file`, `--reason-file`, `goals_file` and the config files) are opened without following links, must be regular files, not under `/proc`, `/sys` or `/dev`, are decoded as strict UTF-8 and size-capped (a query 4 KiB, a reason 4 KiB, goals 16 KiB, each config file 256 KiB); errors never echo content. YAML is loaded safely with bounded aliases.

## Files

State is under `~/.local/state/ticket-queue/` (respecting `XDG_STATE_HOME`).

| File | Public | Content |
|---|---|---|
| `state.db` | no | SQLite (WAL), versioned, additive-only migrations: tickets, board membership, aliases, accounts, boards, claims, pins, skips, rerank results, events outbox |
| `log.jsonl` | yes | append-only operational log, schema [schemas/log.json](../schemas/log.json) |
| `tmp/<run_id>/` | no | a re-rank run's private input and prompt files (mode 0700), deleted when the run ends |

There is no `audit.jsonl`: no command takes an external action. `--actor` and `--decision-id` are accepted and recorded in `log.jsonl`. [schemas/audit.json](../schemas/audit.json) is kept as an empty contract for tools that read every tool's audit file.

The store keeps, for each ticket, the normalized object as ticketctl returned it (sanitized by ticketctl), with `description` (from the last `get`) cut to 8 KiB, and no verdicts or scores. It holds ticket text on disk; it is mode 0600 in a 0700 directory. `prune` removes tickets with no member row, no claim and no pin after `retention.tickets` from `last_seen`, hidden rows that are not claimed, expired re-rank results, outbox events after `retention.events` (an undelivered one with a final `hook_failed` record), lapsed claims, leftovers of crashed re-ranks (see [Slot contract](#slot-contract)) and claims as described under [Claims](#claims); `log.jsonl` is kept (rotated by size under a lock, never pruned). `cron install` also schedules `prune` daily.

## Audit and log

`log.jsonl` records: `refresh` (per board: counts, status, duration), `claim`, `release`, `auto_skip`, `pin`, `unpin`, `move`, `skip`, `unskip`, `pin_dropped`, `rerank` (outcome, reason, run id, duration, cost when known, `sandbox_active`, `candidates_dropped`), `rerank_skipped`, `prune` and `hook_failed`. Every record has `seq` (monotonic per store, one counter shared with the outbox events), `ts`, `event`, `actor`, `actor_source`, `decision_ids`, and `event_id` when an outbox event exists; ticket records add `labels` and `requested_keys`. A lapse is a `release` record with reason `lapsed`; `release.reason` is `released`, `lapsed`, `pruned`, `forced` or `merged`; `pin_dropped.reason` is `done`, `gone` or `expired`; a `skip` records its reason as a byte count; `hook_failed` is `{event_id, hook, attempts, final}`; `level` is optional. None holds ticket text beyond keys. [schemas/log.json](../schemas/log.json) defines them; `decision_ids` appear only in the log.

`export [--since T] [--after-seq N]` prints JSON lines (exempt from the one-object rule, each validated by `log.json`), reading rotated files oldest first. `T` is RFC 3339, a date or `<n>m|h|d|w`.

## Cron

`cron install [--interval D]` writes one block in the crontab (default interval 15 minutes). Accepted intervals are `1m`, `2m`, `3m`, `4m`, `5m`, `6m`, `10m`, `12m`, `15m`, `20m`, `30m`, `1h`, `2h`, `3h`, `4h`, `6h`, `8h`, `12h` and `24h`; any other exits 2 `invalid_argument`, reason `invalid_interval`, listing them. Minutes write `*/N * * * *`, hours `0 */N * * *`, and `24h` writes `0 0 * * *`; `prune` runs `17 3 * * *`. The block is: the first line `# ticket-queue managed v1 every=<interval>` (recording the accepted value), a `PATH=<dirs>` line, the refresh line `<schedule> [. '<env_file>' && ]'<absolute ticket-queue>' refresh --quiet`, the daily `prune --quiet` line, and the last line `# ticket-queue managed end`. A reinstall with a different interval or path replaces the block (`changed: true`).

Install refuses a path containing any character outside `[A-Za-z0-9_@+=:,./-]`, and an env file that is not a regular file owned by the user with mode 0600 or 0400 in a directory others cannot write (`validate` repeats the check). A block starts at a line beginning `# ticket-queue managed` and ends at the end line. `cron uninstall` removes it; for a block without its end line (written by another program) it removes the header and only the directly following lines in the tool's own formats (`PATH=`, the refresh line, the prune line). The harness writes the same block.

## TUI

`ticket-queue browse` is an optional Textual interface (the `ticket-queue[tui]` extra; without it the command exits 12 `tui_missing` and names the install command). Flags: `--focus KEY`, `--board B`, `--interval D` (wins over `browse.refresh_interval`) and `--result-file PATH`.

**List.** One table over all boards or one board (`tab` switches). Columns: `key` (cut from the left), `note` (`[refreshing]`, `[stale: <code>]`, `[changed]`), `score`, `pos`, `title`, `status`, `pri`, `est`, `sprint`, `age`, `flags` (`P` pinned, `C` claimed with holder, `S` skipped, `R` re-ranked, `!` warning). `[stale: <code>]` shows the board's `lower_code` or `error.code`; `[changed]` is set when `updated` differs from the last shown value and cleared when highlighted. Every cell is cut to its width in terminal cells and by code points (at most 4 per cell), runs of more than 3 combining marks become one marker, and empty cells stay as padding. The header is sticky with a sort arrow; left and right (or `h` and `l`) scroll by column with `key` and `note` pinned; the offset survives refreshes. Sorting is by parsed value (unknown last in both directions), the header cursor (`H`) sorts on enter or space and reverses on a second press, and the ranked order is the default preset (`ORDER BY position`). At 10 columns only `key` is shown.

**Query panel (`F`).** A form and a query line kept in sync, with match counts and unknown counts and the actions Apply, Cancel, Reset and Reset to default. `D` writes the current query as the default to `~/.config/ticket-queue/config.d/90-browse-query.yaml` (written with a safe dumper, to a temp file in mode 0600 and renamed; the path is shown in the status line; the file overrides `browse.query` in `config.yaml`). History is kept in `~/.local/state/ticket-queue/browse-history` (mode 0600, valid queries only, newest first, no duplicates, at most 100 lines of 1000 characters, each re-validated on load, merged with entries other running instances saved). A bad `browse.query` is a warning and is ignored, so it never fails other commands. The panel shows each board's source and scope read-only, so a missing ticket can be explained. The presets edit the query text.

**Detail pane.** Opens at once from the stored row. Sections, toggled by the digits 1 to 6: facts, description, score and eligibility (the content of `why`), links and blockers, comments (the only section read lazily, with `ticketctl get --comments 10`, cached for 50 entries by `(site, key, updated)`, at most 8 KiB per comment and 64 KiB per pane, with "(N more characters)"), and claim and manual-order history. Text is sanitized and never parsed as markup; highlighting is bounded; link targets are shown.

**Refresh.** `r` re-reads the highlighted ticket live with `ticketctl get` (through the store writer) and never opens the pane. `R` runs the incremental `refresh`. An automatic refresh runs every `browse.refresh_interval` and reads the store first. Every refresh takes the same lease as cron; a second caller just reads the store. Selection and scroll survive a refresh; a failed board marks its rows stale and keeps the data. Reads cancel between calls, tracked child process groups are killed on quit and the timer stops on every exit path.

**Actions** (each is also a CLI command and is run through it). The TUI's holder is `browse:<me.name>` (`me.name` normalized to the label-value set: a character outside it becomes `_`), so two people's TUIs differ. The bindings, from which `?` generates its help table, are:

| Key | Context | Action |
|---|---|---|
| `up`, `down`, `enter` | list | move; `enter` opens the detail pane |
| `tab` | list | switch board |
| `left`, `right`, `h`, `l` | list | scroll by column |
| `H` | list | header cursor; sorts on enter or space |
| `1` to `6` | pane | toggle a detail section |
| `p` | list | pin or unpin |
| `K`, `J` | list | move up or down in the pin list (pin first when not pinned) |
| `x` | list | skip |
| `c` | list | claim with holder `browse:<me.name>` |
| `u` | list | release the TUI's own claim |
| `U` | list | force-release any claim after a confirmation dialog showing the holder and the claim's labels (actor `human:<me.name>`, the `--force` rules of [Commands](#commands)) |
| `r` | list | re-read the highlighted ticket |
| `R` | list | incremental refresh |
| `ctrl+r` | list | re-rank (the TUI owns the keyboard, so this is not a shell binding) |
| `o` | list | open the ticket's browse URL; the TUI stays open |
| `a` | list | toggle ineligible tickets with their reasons |
| `/` | list | filter |
| `F` | list | query panel |
| `m`, `d`, `b`, `c` | query panel only | presets mine, due soon, blocked, claimed |
| `D` | query panel | write the default query |
| `S` | list | select the highlighted ticket to start (see the result file) |
| `?` | any | help |
| `q` | any | quit |

The opener accepts only a URL matching `^https://[A-Za-z0-9.-]+(:[0-9]{1,5})?/[A-Za-z0-9._~/?=&#%:@+,;-]*$` that ends in the row's key (ticketctl's built `https` URL, no userinfo). It runs it as one argv element through `wslview` (WSL), `xdg-open` (Linux) or `open` (macOS), never `cmd.exe`, with the environment `PATH`, `HOME`, `DISPLAY`, `WAYLAND_DISPLAY`, `XDG_RUNTIME_DIR`, `DBUS_SESSION_BUS_ADDRESS`, `XDG_CURRENT_DESKTOP` and `BROWSER`, detached, with the standard streams on `/dev/null`. `S` exits at once with intent `start` and exactly one `selected`; on a ticket failing a hard rule or claimed by another holder it shows an error instead. When the ticket carries a claim made by this TUI (holder `browse:<me.name>`), `S` releases that claim when it writes the start result, so the starting tool can claim with its own holder; a cron `next --claim` could take it in that moment, in which case the starter reports `claimed`.

**Result file.** With `--result-file`, on exit 0 the TUI writes `{"intent": "start"|"open"|"back", "selected": [{"key", "site", "board"}]}` (schema `schemas/cli/browse-result.json`); `start` means the user pressed `S` on a ticket and wants it started, `open` that the user opened the browse URL (with the last opened ticket) and `back` that the user quit without either; on quit the intent is `open` if a URL was opened, else `back`. It carries ids only and no ticket text. Any existing file at the path is removed at start; the file is written as a temp file (exclusive, no links, mode 0600) and renamed; a symlink or non-regular target is refused; a signal exit leaves no file. Without the flag nothing is written.

The TUI runs correctly as a foreground child of another TUI, respects the terminal's colors and never hard-codes a background. Resizing is supported down to 10 columns.

## Upgrades

The state database is versioned; migrations are additive only, so a newer tool can read an older store, and an older tool reading a newer store exits 12 (`schema_newer`). `--json` outputs and hook payloads gain fields additively. Unimplemented commands in a release exit 12 `not_implemented`.

## Residual risks

- A prompt-injected ticket can steer the order of the tickets inside the re-rank's candidate set. The effects are bounded by validation (exact key set), by the slot rule (a re-rank only reorders the slots its covered tickets already hold), by pins and hard rules, and by the fact that a ticket is only handed out for a human or a caller to act on. No re-rank output is acted on as an instruction.
- The sandbox's containment was not attacked in the design-time spike (the injected instructions were not obeyed, so the sandbox was never exercised); a live containment test is a still-to-verify item. `rerank.sandbox: preferred` runs the session without the sandbox when `bwrap` or `socat` is missing, with the warning `sandbox_inactive` and `sandbox_active: false` in the result; such a run still keeps `dontAsk`, the read denies and the generated permission rules, and `off` is not accepted for the bundled path.
- Ticket text is stored on disk (the store, the run directory while a re-rank runs). claude-session keeps a re-rank's session directory and input until its `retention_days` after `forget`, and Claude Code keeps a transcript with the fenced input under `~/.claude/projects` indefinitely; ticket-queue sweeps its own run directories and sessions (see [Slot contract](#slot-contract)), but Claude Code transcripts remain.
- A claim is local to one machine's store. Two machines (or two state directories) can hand out the same ticket; the caller's own write to the ticket system (an assignment) is what coordinates them.
- A report command containing `$CSESSION_DIR` is denied by claude-session and the model may then give up; the trusted prompt uses absolute paths, which reduces but does not remove the chance. A run that never reports ends `unreported` and falls back to the deterministic order.
- `blocked` and `estimate` are only as good as ticketctl's reading of links and the points fields; an ambiguous points field excludes every ticket on a board until it is mapped (`estimate_ambiguous` names the key to set).
- The cron-filled store can be up to `refresh.stale_after` old when read; `claim` re-reads the ticket live, but `list` and `next` without `--claim` show stored data. A ticket outside the departure re-read set (claimed, pinned, the top 100) can be stale until the next full rebuild.
- A custom `rerank.command` is not stopped by a SIGKILL of ticket-queue.

## Platform notes

Linux and WSL (Ubuntu) are required. macOS is supported on a best-effort basis: there is no Linux-only code, the re-rank sandbox uses Seatbelt where claude-session supports it, and a macOS checklist is part of the live suite. File locking and the lease use portable primitives. Under WSL, `bubblewrap` and `socat` are needed for a sandboxed re-rank; WSL1 has neither, so with `preferred` the re-rank runs without the sandbox (warning `sandbox_inactive`) and with `required` it fails `sandbox_unavailable`.
