# ticket-queue: design

Status: designed (2026-11-12, review passes 1 and 2 applied 2026-11-13), interface 0.1.0-design. The rules are in [interface.md](interface.md), the single normative source; this document records decisions, live verification, structure and test strategy, and links to the interface's headings instead of restating them.

## Role and layer

Layer 2 (composite). Answers "which ticket should I do next?": a local pool of candidate tickets per board read through ticketctl, local eligibility rules, a deterministic score, an optional model re-rank of the top of the list, the user's pins and skips, and claims. It works fully standalone (architecture C2): it needs ticketctl for tickets and, only for the optional re-rank, claude-session; it needs no harness, no tmux and no notifier. It never writes to the ticket system. Section numbers below refer to `../awkit-docs/architecture.md`.

## Uses

| Tool | Binding | Used for |
|---|---|---|
| ticketctl | A (hard) | `search` and `get`; `whoami` at refresh; `fields check` in `validate`; read commands only, with `--json`; every call passes `--site` (no `sprints list`: sprint state comes with every read) |
| claude-session | A (hard), the agent session role (§3.3) | the bundled re-rank only: `start`, `exec --session-id`, `status`, `forget`, `stop`, `list --label`; kind `rank`, headless, called directly (no agent-spawn) |
| the re-rank command | B (configurable command) | `rerank.command`, an argv list reading one JSON document on stdin and printing one on stdout; the default (null) is the bundled implementation that uses claude-session |

ticket-queue has no upward awareness: it does not know ticket-work or the harness. A caller's identity is never an input; behavior differs only by explicit arguments such as `--holder` and `--label` (an opaque holder string and labels that ticket-queue stores and filters on and gives no meaning).

## Decisions

| ID | Decision | Rationale | Interface |
|---|---|---|---|
| D-1 | A board is a named config entry with a ticketctl site and a source (a named query or structured flags); the pool is fetched broadly and eligibility is decided locally per rule, each with a recorded verdict and reason | `why KEY` can explain a ticket that is missing, which JQL alone cannot; `pointed` and `unblocked` need the normalized fields | [Eligibility rules](interface.md#eligibility-rules), [refresh](interface.md#refresh) |
| D-2 | A structured source fetches status categories `new` and `indeterminate` (never done) and applies the rest locally | a done ticket is never a candidate; an in-progress ticket is fetched so `why` can name the rule | [refresh](interface.md#refresh) |
| D-3 | A ticket in several boards belongs to the first enabled board in config order whose pool holds it; ownership is computed when read | one set of goals and weights per ticket, recomputed when config order changes | [Boards, pools and candidates](interface.md#boards-pools-and-candidates) |
| D-4 | Eligibility is three-valued per rule (pass, fail, unknown); unknown excludes with its own reason | ticketctl's honest nulls (`blocked`, `estimate`) must not read as "not blocked" or "unpointed" | [Eligibility rules](interface.md#eligibility-rules) |
| D-5 | The score is a weighted sum of normalized signals with defaults priority 40, sprint 25, epic 20, due 20, assignee 10, age 5, labels 0, size 0; unknown contributes 0 and is shown; the order across boards uses raw sums, so a board's weights raise or lower the board | explainable, configurable per board, no model needed; `validate` warns when boards' weight sums differ | [Scoring](interface.md#scoring) |
| D-6 | `refresh` is the only command that reads pools from ticketctl; the rest read the store and report `stale` | cron fills the cache, interactive commands never wait on the network | [Store](interface.md#store), [refresh](interface.md#refresh) |
| D-7 | Refresh is incremental (`--updated-since` with an overlap) with a periodic full rebuild; a board's refresh has a fetch phase (every ticketctl call) and a commit phase (one transaction, no call), so a failed fetch commits nothing; departures are caught by re-reading claimed, pinned and the top 100 candidates, and pool membership is corrected by the full rebuild | `search` never reports a ticket that left the pool and a `get` cannot see membership; `updated` is not a complete change signal; a half-written pool must never be visible | [refresh](interface.md#refresh) |
| D-8 | A claim is checked live (`get` plus the rules) and the rules are re-run inside the writing transaction; the claim row, unique per ticket, is the arbiter; a stale pool never blocks a claim | the pool may be 15 minutes old; the claim is the moment it must be true, and a ticket revision did not stop two claimers | [next](interface.md#next), [Claims](interface.md#claims) |
| D-9 | One ordered pin list above scoring and re-rank; `move` reorders within it; a pin beats soft rules, never hard rules; a pin is dropped when it expires or its ticket is done or gone (or by `unpin`, or by `prune` after `retention.tickets` in no pool); a pin of a ticket in no enabled board's pool is dormant (kept, never handed out); no default pin timer | anchors relative to the natural order break on refresh; silent expiry changes behavior | [Manual order](interface.md#manual-order) |
| D-10 | Skips are timed or until `unskip`; an auto skip after N consecutive failed releases lasts until `unskip`; a failure counts when `release --failed` ends a live claim, and an `unskip` that ends a skip resets the count | the architecture's auto-skip constraint, with a human way back | [Claims](interface.md#claims), [Manual order](interface.md#manual-order) |
| D-11 | Re-rank is optional, off by default and advisory; it permutes the effective scores the covered tickets already hold, and the model's order is a sort key after the effective score | stays well defined as the pool changes, never changes eligibility or pins, and a tie in score cannot override the model | [Re-rank](interface.md#re-rank) |
| D-12 | The re-rank contract is a stdin/stdout JSON command with strict validation; a violation discards the whole result and keeps the deterministic order | spike A: claude-session catches none of a missing skill, a wrong-shape artifact or a missing `--artifact`; partial acceptance would let a malformed or steered output partly apply | [Slot contract](interface.md#slot-contract) |
| D-13 | The bundled skill echoes a marker read from a plugin reference file, in addition to the run id; the marker detects a skill that did not run and is not authentication | the run id sits in the prompt file and does not prove the skill ran | [Slot contract](interface.md#slot-contract) |
| D-14 | The bundled path calls claude-session directly (`start` then `exec --session-id`, guarded, `status`, `forget`) with kind `rank`, sandbox `preferred` (only `required` or `preferred` are accepted), setting sources none, an env allowlist, `--max-turns 12` and `--timeout 150s`, each replaceable in config; `preferred` deviates from the design guide's default `required` (see Deviations) | the design guide's skill-wrapper rule; `start` gives a handle for crash cleanup; `rank` is the built-in kind for this slot | [Slot contract](interface.md#slot-contract), [Config](interface.md#config) |
| D-15 | Re-rank runs on request, on `--rerank`, or automatically only when the input hash changed, with a minimum interval and a daily limit; a failed automatic run counts toward the limit and backs off, an expired result never blocks a run; manual runs are not blocked by the interval, limit or backoff; `next` and `list` never run it | a re-rank takes 20 to 50 seconds and costs money; unchanged input must not be paid for twice; a failing run must not retry every minute | [Re-rank triggers](interface.md#re-rank-triggers) |
| D-16 | A claim has an optional opaque holder, labels, a TTL that applies only to unrenewed claims (default 72 hours) and a failure counter; a holder's renewal survives `done` and skipped (with warnings) and is refused only for `gone` and `hidden`; holderless claims are not renewable; release is idempotent; claims never write to the ticket system | the architecture's claim contract; a weekend without a caller must not hand out a worked ticket; an "in review" status in category done must not strip a running agent's claim | [Claims](interface.md#claims) |
| D-17 | A refresh never releases a claim; claim state is reported, the first that holds of `gone`, `done`, `reassigned`, `moved` (the claim's `claimed_key` is now an alias), `ok`; a lapse is a read-time fact that the housekeeping sweep of writing commands cleans up; `prune` drops claims on dead tickets after a grace period | the caller knows when work ended; an automatic release can fire under a caller still finishing | [Claims](interface.md#claims) |
| D-18 | One UI-free query engine with a bounded grammar, three-valued evaluation and `ORDER BY` serves `list`, `next` (filters only), the TUI, the default query and the presets | forgectl extracted its engine from the TUI later (#139); a flags-only design would need separate filter code for each surface | [Query language](interface.md#query-language) |
| D-19 | State is one SQLite database; `log.jsonl` is the only public file; there is no audit log; post-hooks are delivered through an outbox, at least once, by the writing commands after their output is closed (the budget bounds the whole delivery phase, including a running hook; a per-event lease, backoff); log lines are appended inside the writing transaction with a `seq` past the log's tail, so file order equals `seq` order; an event undelivered after 14 days is deleted by `prune` with a final log record | no command takes an external action; an auto-skip notification must not be lost, yet a dead receiver must not grow the store forever | [Files](interface.md#files), [Hook delivery](interface.md#hook-delivery) |
| D-20 | Post-hooks only in 0.x; no `pre-claim` | a veto inside `next --claim` raises "take the next ticket or give up?"; gating belongs to the caller's own `pre-` hooks | [Hooks](interface.md#hooks) |
| D-21 | ticket-queue holds no credentials: hooks, the ticketctl child and the claude-session children get separate built environments (the ticketctl child an allowlist plus the names in `ticketctl.pass_env`, the claude-session children `claude_session.pass_env`); the re-rank session and hooks never see ticketctl's names | the credential belongs to ticketctl; a permission profile wrapper can grant read-only ticket access; allowing the tool allows its config ([Trust boundary](interface.md#trust-boundary)) | [Config](interface.md#config), [Environment variables](interface.md#environment-variables) |
| D-22 | The `browse` TUI includes the local actions from the first release, modelled on forgectl's `browse` (query panel, default query, history, sticky header, column scroll, lazy detail, markers, result file); its holder is `browse:<me.name>`, TUI actions run in process, `S` releases a claim the TUI itself made when it writes the start result, and `U` force-releases any claim after a confirmation dialog that shows the holder and labels and stands in for the `--force` TTY test | the user asked for parity; pins and moves only touch local state; the fixed holder `browse` left a human unable to release a stale claim and made `S` refuse | [TUI](interface.md#tui) |
| D-23 | Exit 22 is `empty` and belongs to `next` only; `list` exits 0 with `empty` true | a cron caller must tell "nothing to do" from an error | [Exit codes](interface.md#exit-codes) |
| D-24 | The board takes the per-repo position in the precedence order; hook payloads omit `repo` | ticket-queue has no repository notion (design guide §5, as for jobsctl) | [Config](interface.md#config), [Hooks](interface.md#hooks) |
| D-25 | Board membership is a table (`board_members`); a ticket's owner is the first enabled board in config order with a member row (ownership counts enabled boards only); a refresh's full rebuild removes member rows, not tickets | ownership by "first board that stored it" would depend on refresh order; a ticket that left every pool must stay explainable by `why` | [Boards, pools and candidates](interface.md#boards-pools-and-candidates), [refresh](interface.md#refresh) |
| D-26 | Verdicts, signals, scores and the order are computed on every read from the stored ticket, claims, pins, skips, membership, accounts, the current config and one `now`; nothing derived is stored | time, config and claims change verdicts without the ticket changing; a stored verdict goes stale silently | [Store](interface.md#store) |
| D-27 | Identity is the `account_id` from `ticketctl whoami`, per site, read once per site per refresh (a failed `whoami` uses the stored account); ticket-queue has no `me.jira` key | a display name is not unique; ticketctl already owns the account | [Eligibility rules](interface.md#eligibility-rules), [Config](interface.md#config) |
| D-28 | One store writer: every path that stores a ticket (search, `get`, claim, pin, TUI `r`, canonicalization of a moved key) goes through one function: the newest read wins (each row stores `read_at`), a hidden read replaces the row with a tombstone (it deletes no pin or skip, and a read whose `incomplete` names `labels` keeps the stored labels), aliases are one hop and a real key beats an alias | two writers would each need the same two rules, and one would eventually miss them | [Identifiers](interface.md#identifiers), [Eligibility rules](interface.md#eligibility-rules) |

## Resolved open topics

Architecture §20 listed five; all are resolved here.

- **Pool definition per board:** D-1, D-2, D-3 and [Eligibility rules](interface.md#eligibility-rules).
- **Scoring weights:** D-5 and [Scoring](interface.md#scoring).
- **Refresh cadence:** D-6, D-7, `refresh.stale_after` (30 minutes), `refresh.full_every` (24 hours) and a cron entry every 15 minutes ([Cron](interface.md#cron)).
- **Pin persistence:** D-9, D-10 and [Manual order](interface.md#manual-order).
- **Re-rank contract:** D-11 to D-15 and [Slot contract](interface.md#slot-contract).

The seeded text of this document (written from architecture v1.4) is stale in one respect: it left the ticketctl binding open. It is binding A, now that ticketctl is designed.

## Deviations from the design guide

- **Re-rank sandbox default `preferred`, not `required` (D-14).** The design guide's default for skill wrappers is `required`. The `rank` kind's built-in default is `preferred`, ticket text is semi-trusted board content, and a `preferred` run on a host without bubblewrap is still an isolated run: claude-session keeps `dontAsk` and the generated read denies, so nothing is lost without the sandbox. `rerank.sandbox` therefore accepts only `required` or `preferred` (any other value exits 2 `invalid_argument`, reason `invalid_config`), and `validate` warns `sandbox_inactive` when the sandbox would be or was inactive. A custom `rerank.command` does not use the key.

## Left to 1.x

Batch claim renewal (`claim KEY... --renew`), saved named views, a `pre-claim` hook, a dollar cap on re-rank, board-membership and board-rank reads once ticketctl offers them (a membership check through `search --key` is declined for now; the full rebuild corrects membership), an idempotency token for `next --claim`, a `claim --new` flag and a per-holder claim cap.

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
- `search --order-by updated:asc` combined with `--updated-since` against a real site, and that `incomplete` and `reporter` are valid `--fields` names that print when requested.
- That claude-session's generated rules for the `rank` kind allow `Bash(claude-session report:*)` with a file permission profile that lists only `Skill(ticket-queue:rank-tickets)`, and that the context line names the artifacts directory as an absolute path.
- The wildcard syntax of the bundled profile's deny rules for the file forms of `report` (`Bash(claude-session report * --summary-file *)`, `Bash(claude-session report * --reason-file *)` and the absolute-path forms): that deny beats the generated allow and that a form with `=` is also caught.
- That a missing or hidden issue is a `not_found` and a `forbidden` is a site-level error on a real site (read from ticketctl's source; `get` returns the ticket only and cannot show pool membership).
- That claude-session accepts `--label rerank_until=<unix seconds>` and filters `list --kind rank --label owner=ticket-queue`, and that its context line carries the input path relative to the session directory.
- macOS.
- That a changed `SKILL.md` or plugin content changes the input hash (the hash includes the plugin directory hash by design; the live test asserts it).

## Modules

| Module | Responsibility |
|---|---|
| `cli.py` | argparse commands, global flags, exit codes, output by exit code |
| `gateway.py` | the only process boundary: ticketctl and claude-session calls; builds both child environments; output caps; fakes substitute here |
| `config.py` | YAML and drop-ins, Pydantic validation, precedence, env overrides |
| `store.py` | SQLite access, migrations, transactions, leases, the hook outbox, the single ticket writer (newest read wins, hidden tombstone, alias merge) |
| `model.py` | normalized ticket and result objects, verdicts, claim and skip records |
| `rules.py` | the eligibility rules and verdicts |
| `scoring.py` | signals, weights, effective scores, ordering |
| `query/` | grammar, parser, evaluator, sort keys, the field table (UI-free; shared by the CLI and the TUI) |
| `refresh.py` | board fetch, incremental and full, departures, the one failure-mapping table |
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

Tables (private; the shapes are an implementation detail, only the rules in the interface are contracts). Nothing derived is stored (D-26).

- `tickets(site, key, ticket_json, description, description_read_at, degraded, read_at, gone_at, hidden_at, state_since, first_seen, last_seen, updated_remote)`, keyed `(site, key)`; no board, verdicts, score or revision.
- `board_members(site, key, board, first_seen, last_seen)`; the owner is computed from config order (D-25).
- `aliases(site, alias_key, key)`; an alias is never a ticket row.
- `accounts(site, account_id, display_name, fetched_at)` from `whoami` (D-27).
- `boards(name, source_hash, last_refresh_at, last_full_at, last_notified_at, watermark, status, status_since, error_json, retry_after, truncated)` (the hash of the normalized `hidden.*` lists, for the housekeeping sweep, is kept in a `meta` row); `last_notified_at` is the last `post-refresh` per board (for `hooks.refresh_repeat`), and `status_since` with `status` gives `previous_status`.
- `claims(site, key, claimed_key, holder, labels_json, claimed_at, renewed_at, expires_at, ttl_s, last_state)`, primary key `(site, key)`: one row per ticket, the arbiter of a claim race; the owning board is not stored; `last_state` is written by `refresh` at commit only to detect `post-refresh` changes (outputs always compute the state, and `tickets.state_since` is the one source for `dead_grace`).
- `ticket_state(site, key, consecutive_failures, last_failure_at)`, kept across claims.
- `pins(site, key, pin_position, created_at, expires_at)`; `skips(site, key, source, reason, until, created_at, actor)`.
- `rerank_results(board, input_hash, run_id, result_json, validated_at, sandbox_active, duration_ms, cost_usd, expires_at)` and `rerank_runs` (a count of started runs per local day, for `rerank.daily_limit` and the backoff).
- `events(event_id, seq, hook, entry_cmd_sha256, payload_json, created_at, attempts, next_attempt_at, lease_until, delivered_at)`.
- `leases(object, holder_pid, holder_start, expires_at)`.

Migrations are additive only.

## Concurrency

Per-board leases for `refresh` and `rerank`; one `BEGIN IMMEDIATE` transaction per claim, with every check re-run inside it and the claim row unique per ticket; read commands never wait ([Concurrency](interface.md#concurrency)). The store uses WAL and a 5-second busy timeout. A lease expires 120 seconds after its last heartbeat (sent every 30 seconds), and a holder is dead when its PID is gone or its start time differs, so a long refresh (several pages and batches of `get`) is not taken over. A refresh holds its write transaction only for the commit, after the fetch is complete; no ticketctl or claude-session call is made inside a write transaction; an outbox event has its own lease so two processes never run one delivery at once. The re-rank run holds its board's lease for its whole duration, including the `exec` guard; the process-group runner kills the exact child group on expiry. `refresh`, `rerank`, `claim` and `next --claim` handle SIGINT and SIGTERM by stopping tracked children, cleaning up and releasing the lease.

## Error handling

Every ticketctl and claude-session failure is mapped once, in `gateway.py` and the one table in `refresh.py` / `rerank/`, to the codes and reasons in the interface; nothing guesses. A failing board keeps its data and is marked stale, a failed fetch commits nothing, and a board that failed with a configuration error stays `disabled` and is retried on every refresh until one succeeds (a board with `enabled: false` is `excluded` and never affects an exit code); a stub ticket never overwrites stored fields; a failing re-rank keeps the deterministic order; a failing hook is retried with backoff and logged; an unreadable store exits 12 with a hint. ticketctl output is read through a 64 MiB cap and ticketctl children are killed after 300 s. Untrusted text is sanitized wherever it is printed, with markers for control, format and invisible characters, and never reaches argv, environment or labels; every file the tool reads from a user path follows ticketctl's reader rules, opened without following links and size-capped ([Trust boundary](interface.md#trust-boundary)).

## Dependencies

| Package | Why |
|---|---|
| `pyyaml` | YAML config (the stdlib cannot read it; design guide §3) |
| `pydantic` | config validation (design guide §3) |
| `textual` (extra `tui`) | the browse interface (design guide §3) |
| `jsonschema` (dev only) | validating every `--json` output against its schema in tests |

Everything else is the standard library: `argparse`, `sqlite3`, `subprocess` (inside the gateway), `json`, `hashlib`. The query engine is hand-written (a small bounded recursive-descent parser) so that no query text reaches an evaluator that could run code.

## Test strategy

- Unit tests for each module; every `--json` output validated against its schema and a golden file; the query engine tested with a corpus (including hostile text, deep nesting, over-length, three-valued, number, duration, `ORDER BY` and `me` cases).
- Every subprocess call goes through the gateway; tests use fake `ticketctl` and `claude-session` binaries in `tests/fakebin/` (copied per repository) that return canned output and log invocations, and pinned copies of the siblings' schemas in `tests/`.
- Fake-claude process tests for the re-rank: missing skill, wrong shape, no artifact, a wrong run id, a wrong marker, a key mismatch, a timeout and an unreported run, each falling back to the deterministic order.
- A Textual pilot suite for the TUI, with hostile text, resizes through 200, 80, 30, 20 and 10 columns, deadline-based waits instead of sleeps, and a test that every key binding has a help row.
- Process tests for concurrency: two `refresh` calls on one board, two claimers of one ticket, a claim during a refresh (including during its fetch phase, which never waits on the write lock), a killed refresh, a lease takeover, concurrent writers producing `log.jsonl` in `seq` order (a rolled-back orphan line never causes a reused `seq`), and a re-rank sweep by lock files during another board's live run, including a run between its lock and its directory, that deletes nothing.
- Scenario tests for the store rules: board membership (two boards, a departure from the first, a reorder of config), a moved-ticket collision (a fake `get` returning `requested_key` onto an existing row), a claim racing a refresh and two claimers (no double claim, no spurious `contended`), a failed fetch that commits nothing, an older search read that never overwrites a newer claim read, a per-key `forbidden` that leaves the row, an excluded board that never changes an exit code, a disabled board that recovers on the next successful refresh, a hidden read over a visible row that leaves a tombstone on every writer path and deletes no pin or skip, an `incomplete` labels read that keeps the stored labels, a tombstone's `read_at`, alias repoint with a real key beating an alias and no cycle, the sweep's pin drop reasons (`expired`, `done`, `gone`, `retention`) and dormant pins, `ticket_state` precedence with `moved` ending on renewal, and `requested_keys` present in every listed object.
- A fake `search` that rejects `description` in `--fields`; a fake `get` that answers per key.
- Outbox tests: a lease with two processes, a failing hook with backoff, an event deleted after 14 days with its final log record, and the delivery budget cutting a hanging hook.
- Re-rank tests: a property test that covered tickets appear in model order, hostile-length input, validation traps (duplicate keys, a lone surrogate, a boolean `schema_version`), and a killed re-rank that leaves nothing after the next `prune`.
- A query corpus of at least 15 expressions covering the three-valued, escape, duration and cost-bound rules, a cron interval table with the `PATH` and env-guard rules (a `crontab -l` failure, a relative env file, a failed guard that is loud, an uninstall refused for a foreign line), a YAML loader that refuses an alias, an anchor, a tag, a `<<` key, `.nan` and deep nesting, every CLI schema with `warnings` validating, a packaging test for the four placeholders, and every error envelope validated against `defs.json`.
- `live` tests (never in CI): the spike A behaviors above against a real `claude`, the placeholder check, the sandbox on and off, and the macOS checklist.
- `make verify` (ruff, pyright, pytest without `live`, markdown lint) before every commit.
