# ticket-queue: design

Status: Seeded from awkit architecture v1.4; not yet designed.

## Role and layer

Layer 2 (composite). Answers "which ticket should I do next?" with filters, scoring and claims. Works fully standalone (architecture C2).

## Responsibilities (architecture §5.2)

"Which ticket should I do next?"

- Commands: `refresh`, `list`, `next [--claim]`, `claim KEY`, `release KEY [--failed]`, `why KEY`, `pin`, `move`, `skip`, `browse`.
- Uses: ticketctl; claude-session (default re-rank, headless, kind `rank`); ships `plugin/` with the rank-tickets skill.
- Notes: Candidate pool from configured boards/filters, refreshed periodically, ranked by the pipeline in section 11. Orders candidates not yet started (work already enqueued is ordered by jobsctl). Pins and manual moves always win. **Claims:** `next --claim` and `claim KEY` reserve a ticket so it is not handed out twice; `claim` on a ticket already claimed renews it. A claim is held until `release`; a TTL applies only to claims that are not renewed. `release KEY --failed` records a failed start; after N consecutive failed releases the ticket is auto-skipped and a hook fires (section 20).

Section numbers above refer to `../awkit-docs/architecture.md`.

## Uses

- ticketctl: binding TBD (A/B/C), decided in this design session
- claude-session (default re-rank, headless, kind `rank`): binding B (configurable re-rank command, defaulting to the bundled rank-tickets skill via claude-session), decided in architecture §11; details in this tool's design session

## Cross-cutting sections to read

In `../awkit-docs/architecture.md` (plus `../awkit-docs/design-guide.md` throughout):

- §5.5: capability slots: `rank-tickets`, kind `rank`, headless; exempt from `--mode`, skill configured in config
- §8.1: kinds (`rank`: not third-party-reading, cwd `session-dir`)
- §11: the ranking pipeline (reproduced below)
- §12: autonomy and gates (claims are reservations, not ticket writes)
- §13: security: fencing ticket text for the re-rank session; trust tiers (§13.2 rule 7)

## Ranking pipeline (architecture §11)

1. **Eligibility filters:** status, pointed, current sprint, unblocked according to linked-ticket dependencies, per-board allow/deny lists of parents/epics, unassigned or assigned to the user, and not claimed.
2. **Deterministic scoring:** configurable weights, e.g. priority, sprint membership, prioritized epics, due date, age, labels, assignee (final list decided in this tool's design session). `why KEY` explains the score.
3. **Optional model re-rank** of the top N via a configurable command (binding B, default the bundled rank-tickets skill run headless through claude-session with kind `rank`) using per-board free-text goals and prioritized epics.
4. **Manual pins and moves** always override.

Goals, prioritized epics and parent filters are per board. A ticket stays claimed while the caller works on it, which is how in-flight tickets are excluded without this tool knowing about its callers.

## Capability slot

Default skill: `ticket-queue:rank-tickets` (bundled in `plugin/skills/rank-tickets/`). The slot contract (inputs given to the skill, artifact format expected back) is specified in `docs/interface.md`.

## Constraints (decided)

From architecture §20: The claim TTL applies only to claims not renewed and is configurable; after N consecutive `release --failed` a ticket is auto-skipped and the `post-auto-skip` hook fires.

Packaging: The bundled plugin is shipped inside the package and located via `importlib.resources.files('ticket_queue') / 'plugin'`.

## Open topics

From architecture §20:

- pool definition per board
- scoring weights
- refresh cadence
- pin persistence
- re-rank contract

## Decisions

| Decision | Rationale | Date |
|---|---|---|

## Questions for the design session

1. How is the candidate pool defined per board (filters, sources, eligibility)?
2. Which scoring weights ship by default, and what does `why KEY` show?
3. What refresh cadence does `refresh` assume, and how is staleness handled?
4. How do pins and moves persist across refreshes (and when do they expire, if ever)?
5. What is the re-rank contract: inputs to the rank-tickets skill, the artifact returned, and the configurable command interface (binding B)?
