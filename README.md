# ticket-queue

Answers "which ticket should I do next?" with filters, scoring and claims.

## Status

Designed; not implemented. The interface is specified in [docs/interface.md](docs/interface.md) (0.1.0-design) and the decisions are in [docs/design.md](docs/design.md).

## Layer

L2 (composite). Works fully standalone; it does not need the `awkit` harness, tmux or any notifier.

## What it does

- Keeps a local pool of candidate tickets per board, read through ticketctl and refreshed periodically (incrementally, with a periodic full rebuild).
- Decides eligibility locally, rule by rule (status, pointed, sprint, unblocked, assignee, parent allow and deny lists, hidden, claimed, skipped), and records why a ticket is excluded.
- Scores eligible tickets deterministically from configurable weights (priority, sprint, prioritized epics, due date, assignee, age, labels, size). `why KEY` explains every term.
- Optionally lets a model re-rank the top of the list against per-board goals, through the bundled `rank-tickets` skill run headless via claude-session (or any command you configure). A bad or missing result never changes the deterministic order.
- Applies manual pins, moves and skips, which always win over scores and re-rank (a pin never overrides a hard rule: done, gone, hidden, claimed, skipped).
- Claims: `next --claim` and `claim KEY` reserve a ticket so it is not handed out twice; a claim is held until `release`, with a TTL only for claims that are not renewed (a holder's renewal survives a ticket that became done or skipped, with a warning). `release KEY --failed` records a failed start; after N consecutive failed releases the ticket is auto-skipped and a hook fires. A claim is local: ticket-queue never writes to the ticket system.
- A query language (`list --query`, `next --query`, the TUI panel) shared by the CLI and the terminal UI.
- An optional terminal UI, `browse`, with a query panel, pins, moves, skips, claims and a detail pane.

Bundles a Claude Code plugin named `ticket-queue` in `plugin/` with the default `rank-tickets` skill (`ticket-queue:rank-tickets`).

## Commands

Specified in [docs/interface.md](docs/interface.md#commands).

- `ticket-queue refresh`, `list`, `show`, `why`, `next [--claim]`
- `ticket-queue claim`, `release [--failed]`, `claims`
- `ticket-queue pin`, `unpin`, `move`, `skip`, `unskip`
- `ticket-queue rerank`
- `ticket-queue boards`, `validate [--probe]`, `cron install|uninstall`, `export`, `prune`, `browse`

Exit codes follow the design guide. The tool-specific code is 22: `next` found no eligible candidate (`list` always exits 0 and reports `empty`). Errors carry a closed `error.code` per exit (`invalid_argument`, `not_found`, `partial_results`, `lower_failed`, `refused`, `ambiguous_key` and others) and a separate `error.reason` such as `claimed`, `done`, `gone`, `skipped`, `hidden`, `holder_mismatch`, `not_held`, `bad_query` or `contended`. See [Exit codes](docs/interface.md#exit-codes).

## Dependencies

- ticketctl: binding A (hard). Read commands only (`search`, `get`, `whoami`, `fields check`). Install: `uv tool install git+https://github.com/agent-workflow-kit/ticketctl`. Configure its sites, field mappings and named queries in ticketctl's own config.
- claude-session (version 0.37.0 or later): binding A, only for the optional re-rank. Install: `uv tool install --link-mode=copy git+https://github.com/agent-workflow-kit/claude-session`. On Linux and WSL2 the sandbox also needs `bubblewrap` and `socat`.
- the re-rank command: binding B, configured with `rerank.command` (default: the bundled implementation).

A missing sibling produces an error naming its install command.

## Configuration

`~/.config/ticket-queue/config.yaml` plus `config.d/*.yaml`; see [config.example.yaml](config.example.yaml) and [Config](docs/interface.md#config). Environment variables are `TICKET_QUEUE_<KEY>`. ticket-queue reads no credentials itself, and allowing the tool allows its config (see [Trust boundary](docs/interface.md#trust-boundary)): ticketctl's variables are named in ticketctl's config and are handed to ticketctl only when listed in `ticketctl.pass_env`.

## Hooks

`post-claim`, `post-release`, `post-auto-skip`, `post-refresh` (fires on changes), `post-rerank` (failed runs only), delivered at least once with an `event_id` and a `seq`, by the commands that write. See [Hooks](docs/interface.md#hooks).

## Install

```sh
uv tool install git+https://github.com/agent-workflow-kit/ticket-queue
```

The terminal UI needs the extra: `uv tool install "ticket-queue[tui] @ git+https://github.com/agent-workflow-kit/ticket-queue"`.

## Documentation

- [Interface](docs/interface.md)
- [Design](docs/design.md)
- [awkit architecture](../awkit-docs/architecture.md) ([published](https://github.com/agent-workflow-kit/awkit-docs/blob/main/architecture.md))
- [awkit design guide](../awkit-docs/design-guide.md) ([published](https://github.com/agent-workflow-kit/awkit-docs/blob/main/design-guide.md))

## License

MIT — see LICENSE.
