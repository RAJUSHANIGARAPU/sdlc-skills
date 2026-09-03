---
name: design-review
description: >
  Turn a vague ask into acceptance criteria, write an architecture decision record that can
  actually be decided, and review someone else's design or proposal. Load when writing or
  reviewing an ADR, reviewing a design doc / RFC / proposal, choosing between architectural
  options, or when a requirement arrives too vague to build against.
version: 1.0.0
---

# Design review

Two jobs: make a decision recordable, and make a review useful. Both fail the same way — words that
no observation can settle.

## 1. Vague ask → acceptance criteria

Do this **before** any design. A criterion is valid only if you can name the observation that settles
it: a query, a metric with a threshold, a test, a log line, a screen.

Bad (aspiration): *"Policy search must be performant."*
Good (observation): *"p95 latency of `GET /policies?holder=` under 50 concurrent users against a
2M-row `policy` table is < 400 ms, from the gateway access log over a 10-minute soak."*

Bad: *"The migration must be safe."*
Good: *"After the changeset runs on a restored production snapshot, `SELECT count(*) FROM policy
WHERE premium_cents IS NULL` returns 0, in under 60 s, with no lock held longer than 2 s."*

Trap: aspiration criteria ("resilient", "scalable", "no regressions") survive review because
nobody can disagree with them, then decide nothing at delivery. Rewrite each as *who observes what,
where, against what threshold*; if nobody can name the observation it is a value, not a requirement
— drop it. Also pin must-have vs nice-to-have and what is explicitly **out of scope**: an unwritten
non-goal starts every scope fight.

## 2. The ADR, and what makes one decidable

```markdown
# ADR-014: Outbox table for contract-created events

## Status
Proposed | Accepted (2026-09-03) | Superseded by ADR-021

## Context
What is true today, in facts and numbers. The trigger for deciding now.

## Forces in tension
The constraints that actually conflict — not a wish list; pairs that cannot both win:
- Exactly-once delivery vs. no distributed transaction across Postgres and the broker.
- Publishing inside the JPA transaction vs. p95 write latency under 200 ms.
- Two-week deadline vs. zero operational experience with Debezium on this team.

## Options considered
### A. Transactional outbox table, polled (CHOSEN)
### B. Publish from @TransactionalEventListener(AFTER_COMMIT)
Rejected: a crash between commit and publish loses the event silently with no reconciliation
path; a lost contract event costs a manual finance correction.
### C. Debezium CDC on the WAL
Rejected: needs `wal_level=logical` and a Connect cluster nobody here operates — a second failure
domain for 40 events/minute. Reopen if volume passes ~1k/min.

## Decision
One paragraph, active voice, specific enough to implement.

## Consequences
Accepted: a poller, a table to prune, at-least-once delivery so every consumer must be idempotent
on `event_id`. Gained: no lost events, one transaction, no new infrastructure.

## Reversibility
Cheap to undo (~1 day: delete poller + table) — so this got 2 hours of deliberation, not a spike.
```

**An ADR listing only the chosen option is a press release, not a decision record.** The rejected
options are the content — they tell the next person (often you, in 18 months) which alternatives
were already priced and why, so the decision is not reopened from zero. Every rejection needs a
reason and, where relevant, a *reopen trigger*.

**Reversibility gates effort.** What does undoing this cost — an afternoon, a migration, or a
client renegotiation? One-way doors (public API shape, a wire event schema, a data model everyone
joins against, a vendor lock-in) justify days; local reversible choices do not. Effort
disproportionate to reversibility wastes in both directions.

## 3. Reviewing someone else's design

Ask these. Missing answers *are* the finding.

- **Partial failure.** Step 3 of 5 fails — what state is the system in, and who cleans up?
- **Existing data.** What is the migration path for rows already in production, including the ones
  that violate the new invariant? Backfill, or accept two shapes forever?
- **Retry.** Is it idempotent, keyed on what? Duplicate delivery: second charge, or no-op?
- **State ownership.** Where does the state live, which service is authoritative, who may write it?
  Two writers is the finding.
- **Observability.** How will we know in production that it works — not that the pod is up. Name
  the metric, log, or trace and its expected value.
- **Blast radius.** Wrong at 03:00 Sunday: how many users, silent or loud, recoverable from data
  still on disk?
- **What it makes harder.** Which future change does this foreclose or make expensive?

Spring Boot / Postgres / Kubernetes specifics, worth asking every time:

- **Transaction boundaries.** Where does `@Transactional` start and end — is an HTTP call, broker
  publish, or file write inside it? Is the read-modify-write protected against a concurrent writer
  (`@Version`, `SELECT … FOR UPDATE`) or is last-write-wins accepted? Does a self-invocation or
  `REQUIRES_NEW` quietly bypass the proxy?
- **Liquibase on a large table.** What does this changeset do to 50M rows on production hardware?
  `ADD COLUMN` with a non-volatile default is metadata-only on modern Postgres; `SET NOT NULL`, a
  table rewrite, or a plain `CREATE INDEX` takes locks that block writes. Ask for `CREATE INDEX
  CONCURRENTLY` in its own changeset with `runInTransaction:false`, timing from a restored
  snapshot, and a rollback block.
- **Rolling deploy.** Old and new pods run at once. Is the schema change backward-compatible with
  the *currently deployed* code (expand → migrate → contract, across two releases)? Can the old
  consumer read the new payload? Is a dropped column still referenced by a live replica?

## 4. Grading findings so they are actionable

Label every finding. Unlabelled feedback gets triaged as opinion.

- **BLOCKING** — names the scenario in which the design fails, plus the consequence. *"Two pods run
  the poller during a rolling deploy; nothing claims rows, so every event publishes twice and the
  finance consumer is not idempotent → duplicate bookings."* A blocking objection without a named
  scenario is a preference wearing a badge.
- **SHOULD FIX** — real cost, not a failure. *"No metric on outbox lag; we would learn about a
  stalled poller from a client call."*
- **QUESTION** — you cannot yet tell. Ask; do not pre-judge.
- **PREFERENCE** — *"I would have done it differently."* Say those words. Legitimate to state, and
  legitimate for the author to decline.

Say which one you mean, explicitly. Most review damage is a preference delivered in the register of a
defect: it costs the author a rewrite and costs you credibility the next time something genuinely
blocks. The inverse ships bugs — a real failure scenario phrased as "just a thought" gets waved through.

As the author under review: separate the objection you must fix from the one you may decline, answer
the scenario not the tone, and record every declined objection in Consequences — that is the
difference between defending a decision and winning an argument.

## 5. When NOT to write an ADR

Skip it when the decision is **reversible, cheap, and local**: a library inside one module, a package
layout, a DTO behind an internal interface, a fixture strategy. The code is the record.

Write one when any of these holds: hard to reverse; crosses a team or service boundary; someone will
ask "why is it like this?" in a year; the rejected options were genuinely plausible; it accepts a
known, deliberate cost. Ceremony on trivia is not harmless — it trains people to skim the format, and
then the one ADR that mattered gets skimmed too. Fewer, heavier ADRs beat a directory of forty.
