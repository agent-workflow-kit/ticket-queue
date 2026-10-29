# ticket-queue

Answers "which ticket should I do next?" with filters, scoring and claims.

## Status

Pre-design: interface not yet specified; see [docs/design.md](docs/design.md).

## Layer

L2 (composite). Works fully standalone; it does not need the `awkit` harness.

## What it does

- Builds a candidate pool from configured boards/filters, refreshed periodically, ranked by eligibility filters, deterministic scoring, an optional model re-rank and manual pins/moves (which always win).
- Orders candidates not yet started (work already enqueued is ordered by jobsctl).
- Claims: `next --claim` and `claim KEY` reserve a ticket so it is not handed out twice; a claim is held until `release`, with a TTL only for claims not renewed.
- `release KEY --failed` records a failed start; after N consecutive failed releases the ticket is auto-skipped and a hook fires.

Bundles a Claude Code plugin named `ticket-queue` in `plugin/` with the default `rank-tickets` skill (`ticket-queue:rank-tickets`).

## Indicative commands

Indicative, finalized in [docs/interface.md](docs/interface.md).

- `ticket-queue refresh`
- `ticket-queue list`
- `ticket-queue next [--claim]`
- `ticket-queue claim KEY`
- `ticket-queue release KEY [--failed]`
- `ticket-queue why KEY`
- `ticket-queue pin`
- `ticket-queue move`
- `ticket-queue skip`
- `ticket-queue browse`

## Dependencies

- ticketctl: binding: TBD (A/B/C), decided in this tool's design session
- claude-session (default re-rank, headless, kind `rank`): binding B (configurable re-rank command, defaulting to the bundled rank-tickets skill via claude-session), decided in architecture §11; details in this tool's design session

## Install

```sh
uv tool install git+https://github.com/agent-workflow-kit/ticket-queue
```

## Documentation

- [awkit architecture](../awkit-docs/architecture.md) ([published](https://github.com/agent-workflow-kit/awkit-docs/blob/main/architecture.md))
- [awkit design guide](../awkit-docs/design-guide.md) ([published](https://github.com/agent-workflow-kit/awkit-docs/blob/main/design-guide.md))

## License

MIT — see LICENSE.
