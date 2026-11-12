# ticket-queue: design

Status: designed (2026-11-12), interface 0.1.0-design. The rules are in [interface.md](interface.md), the single normative source; this document records decisions, live verification, structure and test strategy, and links to the interface's headings instead of restating them.

## Role and layer

Layer 2 (composite). Answers "which ticket should I do next?": a local pool of candidate tickets per board read through ticketctl, local eligibility rules, a deterministic score, an optional model re-rank of the top of the list, the user's pins and skips, and claims. It works fully standalone (architecture C2): it needs ticketctl for tickets and, only for the optional re-rank, claude-session; it needs no harness, no tmux and no notifier. It never writes to the ticket system. Section numbers below refer to `../awkit-docs/architecture.md`.

## Uses

| Tool | Binding | Used for |
|---|---|---|
| ticketctl | A (hard) | `search`, `get` and `sprints list`, plus `whoami` and `fields check` in `validate --probe`; read commands only, with `--json`; every call passes `--site` |
| claude-session | A (hard), the agent session role (§3.3) | the bundled re-rank only: `start`, `exec --session-id`, `status`, `forget`, `stop`, `list --label`; kind `rank`, headless, called directly (no agent-spawn) |
| the re-rank command | B (configurable command) | `rerank.command`, an argv list reading one JSON document on stdin and printing one on stdout; the default (null) is the bundled implementation that uses claude-session |

ticket-queue has no upward awareness: it does not know ticket-work or the harness. A caller's identity is never an input; behavior differs only by explicit arguments such as `--holder` and `--label` (an opaque holder string and labels that ticket-queue stores and filters on and gives no meaning).

## Decisions

| ID | Decision | Rationale | Interface |
|---|---|---|---|
| D-1 | A board is a named config entry with a ticketctl site and a source (a named query or structured flags); the pool is fetched broadly and eligibility is decided locally per rule, each with a recorded verdict and reason | `why KEY` can explain a ticket that is missing, which JQL alone cannot; `pointed` and `unblocked` need the normalized fields | [Eligibility rules](interface.md#eligibility-rules), [refresh](interface.md#refresh) |
| D-2 | A structured source fetches status categories `new` and `indeterminate` (never done) and applies the rest locally | a done ticket is never a candidate; an in-progress ticket is fetched so `why` can name the rule | [refresh](interface.md#refresh) |
| D-3 | A ticket in several boards belongs to the first board in config order | one set of goals and weights per ticket; no cross-board scoring ambiguity | [Boards, pools and candidates](interface.md#boards-pools-and-candidates) |
| D-4 | Eligibility is three-valued per rule (pass, fail, unknown); unknown excludes with its own reason | ticketctl's honest nulls (`blocked`, `estimate`) must not read as "not blocked" or "unpointed" | [Eligibility rules](interface.md#eligibility-rules) |
| D-5 | The score is a weighted sum of normalized signals with defaults priority 40, sprint 25, epic 20, due 20, assignee 10, age 5, labels 0, size 0; unknown contributes 0 and is shown | explainable, configurable per board, no model needed | [Scoring](interface.md#scoring) |
| D-6 | `refresh` is the only command that reads pools from ticketctl; the rest read the store and report `stale` | cron fills the cache, interactive commands never wait on the network | [Store](interface.md#store), [refresh](interface.md#refresh) |
| D-7 | Refresh is incremental (`--updated-since` with an overlap) with a periodic full rebuild; departures are caught by re-reading claimed, pinned and top-100 tickets with `get` | `search` never reports a ticket that left the pool; `updated` is not a complete change signal | [refresh](interface.md#refresh) |
| D-8 | A claim is checked live (`get` plus the rules) before it is written; a stale pool never blocks a claim | the pool may be 15 minutes old; the claim is the moment it must be true | [next](interface.md#next), [Claims](interface.md#claims) |
| D-9 | One ordered pin list above scoring and re-rank; `move` reorders within it; a pin beats soft rules, never hard rules; no default pin timer | anchors relative to the natural order break on refresh; silent expiry changes behavior | [Manual order](interface.md#manual-order) |
| D-10 | Skips are timed or until `unskip`; an auto skip after N consecutive failed releases lasts until `unskip` | the architecture's auto-skip constraint, with a human way back | [Claims](interface.md#claims), [Manual order](interface.md#manual-order) |
| D-11 | Re-rank is optional, off by default and advisory; it permutes the effective scores the covered tickets already hold | stays well defined as the pool changes; never changes eligibility or pins | [Re-rank](interface.md#re-rank) |
| D-12 | The re-rank contract is a stdin/stdout JSON command with strict validation; a violation discards the whole result and keeps the deterministic order | spike A: claude-session catches none of a missing skill, a wrong-shape artifact or a missing `--artifact`; partial acceptance would let a malformed or steered output partly apply | [Slot contract](interface.md#slot-contract) |
| D-13 | The bundled skill echoes a marker read from a plugin reference file, in addition to the run id | the run id sits in the prompt file and does not prove the skill ran | [Slot contract](interface.md#slot-contract) |
| D-14 | The bundled path calls claude-session directly (`start` then `exec --session-id`, guarded, `status`, `forget`) with kind `rank`, sandbox `preferred`, setting sources none, an env allowlist, `--max-turns 12` and `--timeout 150s`, each replaceable in config | the design guide's skill-wrapper rule; `start` gives a handle for crash cleanup; `rank` is the built-in kind for this slot | [Slot contract](interface.md#slot-contract), [Config](interface.md#config) |
| D-15 | Re-rank runs on request, on `--rerank`, or automatically only when the input hash changed, with a minimum interval and a daily limit; `next` and `list` never run it | a re-rank takes 20 to 50 seconds and costs money; unchanged input must not be paid for twice | [Re-rank triggers](interface.md#re-rank-triggers) |
| D-16 | A claim has an optional opaque holder, labels, a TTL that applies only to unrenewed claims (default 72 hours) and a failure counter; release is idempotent; claims never write to the ticket system | the architecture's claim contract; a weekend without a caller must not hand out a worked ticket | [Claims](interface.md#claims) |
| D-17 | A refresh never releases a claim; claim state (`ok`, `done`, `gone`, `reassigned`, `moved`) is reported; `prune` drops claims on dead tickets after a grace period | the caller knows when work ended; an automatic release can fire under a caller still finishing | [Claims](interface.md#claims) |
| D-18 | One UI-free query engine with a bounded grammar, three-valued evaluation and `ORDER BY` serves `list`, `next` (filters only), the TUI, the default query and the presets | forgectl extracted its engine from the TUI later (#139); a flags-only design would need separate filter code for each surface | [Query language](interface.md#query-language) |
| D-19 | State is one SQLite database; `log.jsonl` is the only public file; there is no audit log; post-hooks are delivered through an outbox, at least once | no command takes an external action; an auto-skip notification must not be lost | [Files](interface.md#files), [Hook delivery](interface.md#hook-delivery) |
| D-20 | Post-hooks only in 0.x; no `pre-claim` | a veto inside `next --claim` raises "take the next ticket or give up?"; gating belongs to the caller's own `pre-` hooks | [Hooks](interface.md#hooks) |
| D-21 | ticket-queue holds no credentials: the ticketctl child gets an allowlisted environment plus the names in `ticketctl.pass_env`; hooks and the re-rank session never see them | the credential belongs to ticketctl; a permission profile wrapper can grant read-only ticket access | [Config](interface.md#config), [Environment variables](interface.md#environment-variables) |
| D-22 | The `browse` TUI includes the local actions from the first release, modelled on forgectl's `browse` (query panel, default query, history, sticky header, column scroll, lazy detail, markers, result file) | the user asked for parity; pins and moves only touch local state | [TUI](interface.md#tui) |
| D-23 | Exit 22 is `empty` | a cron caller must tell "nothing to do" from an error | [Exit codes](interface.md#exit-codes) |
| D-24 | The board takes the per-repo position in the precedence order; hook payloads omit `repo` | ticket-queue has no repository notion (design guide §5, as for jobsctl) | [Config](interface.md#config), [Hooks](interface.md#hooks) |

## Resolved open topics

Architecture §20 listed five; all are resolved here.

- **Pool definition per board:** D-1, D-2, D-3 and [Eligibility rules](interface.md#eligibility-rules).
- **Scoring weights:** D-5 and [Scoring](interface.md#scoring).
- **Refresh cadence:** D-6, D-7, `refresh.stale_after` (30 minutes), `refresh.full_every` (24 hours) and a cron entry every 15 minutes ([Cron](interface.md#cron)).
- **Pin persistence:** D-9, D-10 and [Manual order](interface.md#manual-order).
- **Re-rank contract:** D-11 to D-15 and [Slot contract](interface.md#slot-contract).

The seeded text of this document (written from architecture v1.4) is stale in one respect: it left the ticketctl binding open. It is binding A, now that ticketctl is designed.

## Live verification

Spike A (2026-11-12, claude 2.1.322, claude-session 0.37.0, haiku, 23 real runs on fake tickets, private files under `docs/design/spikes/ticket-queue/` in the umbrella folder, not in this repository). Each relied-on behavior is a `live` regression test in the plan.

| Behavior | Result | Consequence in the design |
|---|---|---|
| A headless `rank` run through `start` and `exec --session-id` with the sandbox on | exit 0, outcome `done`, a valid frozen artifact, 36 to 47 seconds, 6 to 7 turns, about $0.005 for 10 tickets | the bundled path works as designed |
| A skill that only says "write ranking.json" | 2 of 3 runs filed `done` with no `--artifact`: the artifact was null and the exit still 0 | the prompt tells the agent to attach the result with `--artifact`; a null artifact is a failure |
| A nonexistent skill | the run improvised a valid artifact and exited 0 | strict validation plus the marker; claude-session is not a check |
| A wrong-shape artifact | `done`, exit 0 | strict validation by ticket-queue |
| The nonce | it sits in the prompt file and proves nothing about the skill | the marker from a reference file in the plugin |
| Plugin dir without the four placeholders (`.claude`, `.mcp.json`, `CLAUDE.md`, `CLAUDE.local.md`) | every sandboxed Bash call fails; the run ends exit 14 | the plugin ships them; `validate` checks them |
| `sandbox_active` | true normally; with `bwrap` off `PATH` the run finishes with `sandbox_active: false` and a warning | recorded in the re-rank result |
| Completed rankings | 17 of 17 had the exact key set, correct run id, no invented or duplicate keys | strict validation accepts the normal case |
| Repeated identical input | 4 of 5 identical orderings; one differed by 5 of 45 pair inversions | an order can differ between runs; the stored result is what applies |
| Prompt injection in a ticket (3 runs) | not obeyed; no steering; no tool use beyond a clean run | the sandbox was never attacked: containment stays a still-to-verify item |
| No goals | the output equals the deterministic order exactly (2 runs) | a re-rank with no goals is a no-op by construction |
| With goals | matching tickets moved from positions 6, 8 and 9 to 1 to 3 in 4 of 5 runs; reasons short and usable | the re-rank has an effect only when goals say something |
| 30 candidates | valid in 2 of 3 runs, 40 to 48 seconds, about $0.007; the third failed on a Bash denial | top_n default 20, maximum 30 |
| A report command containing `$CSESSION_DIR` | denied; the agent gave up in 2 of 23 runs | the trusted prompt uses absolute paths |
| `--max-turns` | 1 gives exit 16; 3 did not stop a 6-turn run | a loose cap only (12) |
| `--input-file` size | cap 10 MiB per file; one byte over exits 2 | ticket-queue's own cap (64 KiB) is far below |
| `forget` | works; the session and private directories stay until `reconcile` (30 days) | residual risk: ticket text on disk |
| `--implicit-done` and `--no-implicit-done` | not in claude-session 0.37.0; the `rank` kind has `implicit_done: false` | the flag is not passed |

Still to verify (conservative defaults in place):

- A live containment attack against the sandbox with a hostile ticket (the model never tried one).
- The tool inventory of a `rank` session with the bundled profile (the profile denies `Agent` and `Task`; not live-checked).
- True same-second parallel re-ranks (the per-board lease prevents them in a single store).
- The behavior of ticketctl's `search --order-by` and `--status-category` against a real site (ticketctl's own to-verify list); ticket-queue does not depend on either: it orders locally and reads `status.category`.
- macOS.
- That a changed `SKILL.md` or plugin content changes the input hash (the hash includes the plugin directory hash by design; the live test asserts it).

## Modules

| Module | Responsibility |
|---|---|
| `cli.py` | argparse commands, global flags, exit codes, output by exit code |
| `gateway.py` | the only process boundary: ticketctl and claude-session calls; builds the child environment; fakes substitute here |
| `config.py` | YAML and drop-ins, Pydantic validation, precedence, env overrides |
| `store.py` | SQLite access, migrations, transactions, leases, the hook outbox |
| `model.py` | normalized ticket and result objects, verdicts, claim and skip records |
| `rules.py` | the eligibility rules and verdicts |
| `scoring.py` | signals, weights, effective scores, ordering |
| `query/` | grammar, parser, evaluator, sort keys, the field table (UI-free; shared by the CLI and the TUI) |
| `refresh.py` | board fetch, incremental and full, departures, failure mapping |
| `claims.py` | claims, releases, auto-skip, claim state |
| `manual.py` | pins, moves, skips |
| `rerank/` | the input and output documents, validation, the custom-command runner, the bundled claude-session run, the process-group runner |
| `hooks.py` | hook execution and outbox delivery |
| `logs.py` | `log.jsonl` with rotation; `export` |
| `cron.py` | the managed crontab block |
| `render.py` | human output, sanitizing of untrusted text |
| `tui/` | the Textual interface (optional extra) |
| `plugin/` | the bundled plugin: skill, references, profile, placeholders |

The bundled plugin is shipped inside the package and located with `importlib.resources.files('ticket_queue') / 'plugin'` (design guide §11); the distribution maps `plugin` to `ticket_queue/plugin` as package data.

## Data model

Tables (private; the shapes are an implementation detail, only the rules in the interface are contracts):

- `tickets(site, key, board, requested_key, ticket_json, verdicts_json, score, state, first_seen, last_seen, updated_remote, revision)`, keyed `(site, key)`.
- `boards(name, last_refresh_at, last_full_at, status, error_json, truncated)`.
- `claims(site, key, board, holder, labels_json, claimed_at, renewed_at, expires_at)`, one row per live claim.
- `ticket_state(site, key, consecutive_failures, last_failure_at)`, kept across claims.
- `pins(site, key, position, created_at, expires_at)`; `skips(site, key, source, reason, until, created_at, actor)`.
- `rerank_results(board, input_hash, run_id, result_json, validated_at, sandbox_active, duration_ms, cost_usd, expires_at)` and `rerank_runs` (a day count for `rerank.daily_limit`).
- `events(event_id, hook, payload_json, created_at, delivered_at)`.
- `leases(object, holder_pid, holder_start, expires_at)`.

Migrations are additive only. A row's `revision` is the compare-and-set value for claims.

## Concurrency

Per-board leases for `refresh` and `rerank`; compare-and-set claims; read commands never wait ([Concurrency](interface.md#concurrency)). A lease is renewed from a heartbeat while its holder lives, so a long refresh (several pages and batches of `get`) is not taken over. The re-rank run holds its board's lease for its whole duration, including the `exec` guard; the process-group runner kills the exact child group on expiry.

## Error handling

Every ticketctl and claude-session failure is mapped once, in `gateway.py` and `refresh.py` / `rerank/`, to the codes and reasons in the interface; nothing guesses. A failing board keeps its data and is marked stale; a failing re-rank keeps the deterministic order; a failing hook is logged; an unreadable store exits 12 with a hint. Untrusted text is sanitized wherever it is printed, with markers for control, format and invisible characters, and never reaches argv, environment or labels.

## Dependencies

| Package | Why |
|---|---|
| `pyyaml` | YAML config (the stdlib cannot read it; design guide §3) |
| `pydantic` | config validation (design guide §3) |
| `textual` (extra `tui`) | the browse interface (design guide §3) |
| `jsonschema` (dev only) | validating every `--json` output against its schema in tests |

Everything else is the standard library: `argparse`, `sqlite3`, `subprocess` (inside the gateway), `json`, `hashlib`. The query engine is hand-written (a small bounded recursive-descent parser) so that no query text reaches an evaluator that could run code.

## Test strategy

- Unit tests for each module; every `--json` output validated against its schema and a golden file; the query engine tested with a corpus (including hostile text, deep nesting, over-length and three-valued cases).
- Every subprocess call goes through the gateway; tests use fake `ticketctl` and `claude-session` binaries in `tests/fakebin/` (copied per repository) that return canned output and log invocations, and pinned copies of the siblings' schemas in `tests/`.
- Fake-claude process tests for the re-rank: missing skill, wrong shape, no artifact, a wrong run id, a wrong marker, a key mismatch, a timeout and an unreported run, each falling back to the deterministic order.
- A Textual pilot suite for the TUI, with hostile text, resizes through 200, 80, 30, 20 and 10 columns, deadline-based waits instead of sleeps, and a test that every key binding has a help row.
- Process tests for concurrency: two `refresh` calls on one board, two claimers of one ticket, a claim during a refresh, a killed refresh and a lease takeover.
- `live` tests (never in CI): the spike A behaviors above against a real `claude`, the placeholder check, the sandbox on and off, and the macOS checklist.
- `make verify` (ruff, pyright, pytest without `live`, markdown lint) before every commit.
