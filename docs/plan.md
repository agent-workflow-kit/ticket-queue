# ticket-queue: implementation plan

Implements [interface.md](interface.md) (the single normative source) and [design.md](design.md). Issues are expanded into bite-sized test-first steps when they are picked up; this plan fixes the stages, the order, what each stage proves and the version roadmap. Status values: Not Started, In Progress, Complete.

## Conventions for every stage

- **Gateway boundary.** Every subprocess call (ticketctl, claude-session, the crontab, the browser opener, hooks, a custom re-rank command) goes through `src/ticket_queue/gateway.py`. Tests replace it with fakes: fake `ticketctl` and `claude-session` binaries live in `tests/fakebin/` (copied per repository, never shared) and return canned output while logging their invocations.
- **Pinned sibling schemas.** The siblings' schemas that ticket-queue parses (ticketctl's ticket object, `search`, `get` and `whoami` outputs; claude-session's `start`, `status` and outcome objects) are copied into `tests/` pinned to a version and every fake's canned output is validated against them. Nothing is imported from a sibling repository.
- **Schemas and goldens.** Every `--json` output, hook payload, log record and the re-rank documents are validated against the repository's own `schemas/`; every command output has a golden file. A test asserts that the code's own validation of the re-rank output document matches `schemas/rerank-output.json`.
- **Process tests and live tests.** Concurrency (two refreshes, two claimers, a claim during a refresh, a killed refresh, lease takeover, outbox delivery from two processes) is tested with real processes against a real SQLite file. Tests that need a real model or real credentials carry the pytest marker `live`, are deselected by default and never run in CI.
- **Spike regressions.** Each relied-on behavior of the design-time spike (design.md, Live verification) is a `live` test or a documented "still to verify" item with a conservative default.
- **Untrusted text.** Every stage that prints or stores ticket text tests the sanitizer with hostile input (control, bidi and invisible characters, very long lines, wide characters, `</untrusted`) and checks that no ticket text reaches argv, the environment, a label or a hook payload.
- **Plugin packaging.** A test asserts the installed package contains the plugin with its four sandbox placeholders, the skill, `references/marker.md` and the profile file, and that the profile denies `Agent` and `Task` and the file forms of `claude-session report` and allows only `Skill(ticket-queue:rank-tickets)`.
- **Quality gate.** `make verify` (ruff, pyright, pytest without `live`, markdown lint) before every commit; test first; one stage's issues land as stacked PRs.

## Module map

| Module | Stage | Responsibility |
|---|---|---|
| `cli.py`, `render.py` | 0, grows | argparse commands, global flags, exit codes, human output, sanitizing |
| `gateway.py` | 0 | the only process boundary; builds the ticketctl and claude-session child environments |
| `config.py` | 0 | YAML, drop-ins, Pydantic validation, precedence, env overrides, file reader rules |
| `logs.py` | 0 | `log.jsonl` with rotation, `export` |
| `store.py`, `model.py` | 1 | SQLite access, migrations, the single writer, leases, transactions, normalized objects |
| `query/` | 1 | grammar, parser, evaluator, sort keys, the field table (UI-free) |
| `rules.py` | 2 | eligibility rules and verdicts |
| `refresh.py` | 2 | fetch and commit phases, membership, watermark, departures, failure mapping |
| `scoring.py` | 3 | signals, weights, effective scores, ordering |
| `claims.py`, `manual.py` | 4 | claims, releases, auto-skip, pins, moves, skips |
| `hooks.py`, `cron.py` | 5 | hook execution, outbox delivery, the crontab block |
| `rerank/` | 6 | input and output documents, validation, the custom command runner, the bundled claude-session run, process-group runner, sweeps |
| `plugin/` | 6 | the bundled skill, marker, profile, placeholders |
| `tui/` | 7 | the Textual interface (optional extra) |

## Stage 0: scaffold, configuration, gateway

**Goal**: A package that parses every command of the interface, loads and validates configuration, runs every external call through fakeable gateways and writes the log.
**Success Criteria**: every command exists and unimplemented ones exit 12 `not_implemented`; `--version`, `boards` (from config) and `validate` (without `--probe`: config, plugin placeholders, profile file, sibling versions) work; invalid config fails fast naming the key; precedence (flags, board, env, global, defaults) and drop-in merging are tested; `log.jsonl` rotates by size under a lock; `config.example.yaml` validates against `schemas/config.json`.
**Tests**: config golden cases and bounds; precedence table; the file reader refusals (links, `/proc`, size, encoding); the YAML loader (anchors, aliases, tags, a `<<` key, `.nan`, nesting deeper than 20, every parser exception mapped to `invalid_config`); a config file, `config.d/` or drop-in owned by another user or writable by group or others; gateway fakes for both siblings with pinned schemas; log rotation under concurrent writers; the exit-code table for usage errors.
**Status**: Not Started

## Stage 1: store and query engine

**Goal**: A private SQLite store with versioned additive migrations, one writer for every ticket write, leases, and a bounded, three-valued query language shared by every surface.
**Success Criteria**: the single writer canonicalizes moved tickets (one hop, a real key beats an alias), refuses hidden reads with a tombstone, applies the newest read only (a stored `read_at` in the future counts as now) and keeps `incomplete` per the last applied read; leases heartbeat and take over a dead holder; the query engine passes the corpus (grammar, three-valued logic, durations, `me`, `LIKE` escapes, bounds and the evaluation budget, `ORDER BY` rules).
**Tests**: migration up and re-open; writer scenarios (moved-ticket collisions, hidden over visible with no pin or skip deleted, an `incomplete` labels read that keeps the stored labels, a tombstone's `read_at`, older read after newer); lease takeover with a killed holder; the query corpus of at least 15 expressions plus hostile text, deep nesting and over-length input; a property test that evaluation never exceeds the budget.
**Status**: Not Started

## Stage 2: refresh, pool and eligibility

**Goal**: `refresh` reads each board through ticketctl and keeps an accurate local pool with explained eligibility.
**Success Criteria**: structured and named-query sources fetch as specified; a fetch is complete, truncated or failed and a failed fetch commits nothing; membership rows, the watermark, `source_hash` (including `hidden.*`), departures re-reads and the per-error failure table behave per the interface; excluded and disabled boards never mix; `whoami` identity per site; eligibility verdicts per rule with reasons; `refresh --dry-run`.
**Tests**: fake `search` and `get` scenarios for every failure code; a fake `search` that rejects `description`; truncation and `max_pages`; incremental overlap and clock skew; two boards owning one ticket and a config reorder; a departure between refreshes; a claim during the fetch phase never waits on the write lock; `estimate_ambiguous` and unknown verdicts.
**Status**: Not Started

## Stage 3: scoring, ordering and the read commands

**Goal**: `list`, `show`, `why` and `next` (without a claim) over the stored pool.
**Success Criteria**: signals, weights, exact arithmetic and tie-breaks per the interface; ordering with pins first; `why` explains every term and verdict; `list --query` and `next --query` with the filter-only rule; `stale` and `refreshed_at` in every read output; `claims`-style filters on `list`; no network call in any read command.
**Tests**: a scoring table (every signal at its boundaries, unknown and none cases); cross-board order with differing weights and the `weights_differ` warning; golden outputs; a counting fake proving read commands never call ticketctl.
**Status**: Not Started

## Stage 4: claims, pins and skips

**Goal**: Reservations and manual order with the exact concurrency rules.
**Success Criteria**: `claim`, `next --claim`, `release`, `claims`, `pin`, `unpin`, `move`, `skip`, `unskip` per the interface; the claim row is the arbiter, checks re-run inside one transaction after the live read; renewal rules (holder required, `done` and `skipped` warn, `gone` refuses); lapse at read time; consecutive failures and the auto-skip; claim states and `moved`; hard and soft rules for pins; dormant pins (a pin of a ticket in no pool stays, is never handed out and becomes live again at its old position); `--force` requires a human terminal.
**Tests**: two claimers racing (no double claim, no spurious `contended`); a claim racing a refresh and a TTL lapse; `next --claim` passing over ineligible candidates within `scan_limit`; failure counting and `unskip` reset; the alias and canonicalization paths through claims and pins; dormant pins dropped only by expiry, done, gone, `unpin` or `prune` retention; exit codes and reasons per command.
**Status**: Not Started

## Stage 5: hooks, outbox, cron, export and prune

**Goal**: Delivery of events and the housekeeping that keeps the store small.
**Success Criteria**: post-hooks delivered at least once by writing commands within their budget with per-event leases, backoff and retention; `post-refresh` fires on changes with the reminder; `export` with `--since` and `--after-seq`; `prune` and the single housekeeping sweep (lapsed claims, expired, done and gone pins, past skips, the `hidden.*` tombstone sweep); `cron install|uninstall` writes the exact block with the accepted intervals and the path and env-file guards (an absolute env file, a loud failed guard, `crontab -l` failure handling, the lock and the previous-crontab copy, uninstall that refuses foreign lines).
**Tests**: two processes delivering one event; a failing hook backing off and being dropped at retention with the final record; a hanging hook cut by the budget; concurrent writers producing `log.jsonl` in `seq` order; the cron interval table and block parsing with and without the end line, `crontab -l` failures, a relative env file and a foreign line inside the block; a killed re-rank leaving nothing after the next `prune` (the sweep goes by lock files and skips a run between its lock and its directory); a `seq` that stays unique after a rolled-back log line.
**Status**: Not Started

## Stage 6: re-rank

**Goal**: The optional, advisory model re-rank, with the bundled claude-session path and the custom command path.
**Success Criteria**: input documents with per-field caps and the 64 KiB rule; strict output validation (exact key set, run id, marker, caps, duplicate keys, depth, lone surrogates); the effective-score permutation with the model's order as a sort key; triggers, input hash, minimum interval, daily limit and backoff; the bundled run sequence (`start`, guarded `exec`, `status`, one-descriptor artifact read, `forget`) with the built child environment; crash sweeps; the bundled plugin; `validate --probe`.
**Tests**: fake claude-session scenarios (missing skill, wrong shape, no artifact, an artifact outside the frozen artifact directory, a sha256 or size mismatch, wrong run id, wrong marker, key mismatch, timeout, unreported run, `sandbox_active` false), each falling back to the deterministic order; plugin directory checks on the bundled and a custom plugin and the `profile_skill_mismatch` warning; a property test that covered tickets appear in model order; a hostile-length input; a killed re-rank; a custom command with output over the cap, a hang and a non-zero exit; `live` tests of the spike behaviors against a real `claude`.
**Status**: Not Started

## Stage 7: the `browse` TUI

**Goal**: The optional terminal interface with forgectl parity for the query panel and the list, plus the local actions.
**Success Criteria**: list, sticky header, column scroll, markers, query panel, default query drop-in, history, quick presets, detail sections with lazy comments, refresh keys, the actions (pin, move, skip, claim, release, `U`, re-rank, open), `--focus`, `--board`, `--interval` and the result file; every action is also a CLI command and is run through it; cancel and quit hygiene.
**Tests**: pilot tests with hostile text; resizes through 200, 80, 30, 20 and 10 columns; deadline-based waits instead of sleeps; a test that every binding has a help row; the opener URL rules; result-file write rules; a refresh racing the detail load.
**Status**: Not Started

## Stage 8: live suite, macOS checklist, documentation

**Goal**: Evidence against real systems and a documented release path.
**Success Criteria**: the `live` suite runs the spike A behaviors, the placeholder check, the sandbox on and off, a containment attempt with a hostile ticket, the wildcard syntax of the profile's deny rules for `report --summary-file` and `--reason-file`, and the tool inventory of a `rank` session; a macOS checklist is recorded; README, `config.example.yaml` and the interface registry match the released behavior.
**Tests**: `make live` (never in CI); the checklist.
**Status**: Not Started

## Version roadmap

- **0.x**: stages 0 to 8, released as `feat` and `fix` commits (never `!` before 1.0).
- **1.0** is reached when: the live suite has passed against a real ticketctl and a real `claude`; the macOS checklist is done; ticket-work uses `next --claim`, `claim --renew` and `release --failed` end to end; and the interface has been unchanged for a release.
- **1.x** (additive): saved named views in the TUI; a `pre-claim` hook (a veto passes over that ticket for `next --claim` and exits 11 for `claim KEY`); batch renewal (`claim KEY... --renew`); a dollar cap on re-rank once claude-session reports cost reliably; board-membership and board-rank reads once ticketctl offers them; an install phase for the re-rank sandbox is not planned.
- **2.0**: another ticket source through a ticketctl provider, if ticketctl gains one; no ticket-queue change is expected beyond configuration.
