# sdlc-skills

Five Claude Code skills for the parts of the software lifecycle that are easy to fake
and hard to verify: whether a test can actually fail, whether a quality gate actually
gates, whether an incident is actually understood, whether a design has actually been
decided, and whether a release is actually safe to cut.

They are deliberately few. A skill's description is loaded into context in every
session, so a large collection taxes every conversation whether or not you use it.
These five were written because nothing published covered them well for a
JVM/Python/CI stack — not to be comprehensive.

## The skills

| Skill | Answers |
|---|---|
| [`false-green-audit`](skills/false-green-audit/SKILL.md) | Can this test fail? Prove it. |
| [`mutation-gate`](skills/mutation-gate/SKILL.md) | Does this coverage number mean anything? |
| [`incident-postmortem`](skills/incident-postmortem/SKILL.md) | What actually happened, and what stops it recurring? |
| [`design-review`](skills/design-review/SKILL.md) | Has a decision been made, or just written down? |
| [`release-semver`](skills/release-semver/SKILL.md) | Is this change breaking, and for whom? |

### false-green-audit

A passing suite is not evidence a feature works. This applies six ordered questions to
every assertion — does it execute unconditionally, is the expected value sourced
independently of the code under test, does it exercise the real unit rather than its own
mock, can it distinguish right from wrong output, is it decoupled from implementation
detail, does it pass both in isolation and in suite order — and then demands the only
verdict that stands: break the code and watch the test go red.

It names the specific shapes that pass forever. An *echo mock*, where
`when(repo.save(any())).thenReturn(dto)` is followed by asserting `dto` back, proves the
stub and not the mapper. A *formula mirror*, where the expected value re-implements the
production calculation, stays green through a wrong formula. A `@Disabled("flaky")` on a
test that is failing because the feature is broken.

### mutation-gate

Line coverage answers "did this line run". Mutation testing answers "would a test fail
if this line were wrong". A coverage threshold can be satisfied by tests that assert
nothing, which is why coverage gates produce coverage theatre.

Covers PiTest on changed classes only (whole-module runs are too slow to gate on, so
they get disabled), the equivalents for TypeScript and Python, and an audit checklist for
gates that only look like gates: `continue-on-error`, `|| true`, a threshold set too low
to trip, a check that skips in the environment where it matters, a scheduled job nobody
reads.

### incident-postmortem

Stabilise before diagnosing, because the instinct to find root cause first is what
extends outages. Query the data before theorising, because a theory formed early collects
supporting evidence rather than testing itself. Write the timeline as it happens, because
memory reconstructs incidents wrongly.

Then the postmortem: contributing conditions rather than a single root cause, what made
detection slow, what made recovery slow, and action items specific enough that someone
can verify they were done. Blameless means no individual named as cause; it does not mean
concluding nothing. If a gate did not gate or an alert was routinely ignored, that is a
finding.

### design-review

An ADR listing only the chosen option is a press release. The rejected options, and why
they were rejected, are the content. Includes the reversibility question — how expensive
is this to undo, and does that justify the deliberation being spent on it.

For reviewing someone else's design, a concrete question set: what happens under partial
failure, what is the migration path for data that already exists, what does this make
harder later, is it idempotent on retry, how will we know in production that it works,
what is the blast radius. And a grading rule: a blocking objection must name the scenario
in which the design fails, not a preference. Say which one you mean.

### release-semver

Most published tooling in this area generates changelogs. The hard part is judgment: a
change is breaking if a consumer breaks, and consumers decide that, not you.

The asymmetry rule — for a response you return, adding a field is usually safe and
removing or narrowing one is breaking; for a request you accept, accepting more is safe
and requiring more is breaking. Plus the cases people get wrong: adding a value to an
enum clients switch on, tightening validation, changing a default, changing error codes
clients match on, and a migration that a rolling deploy's older pods cannot tolerate.

## Install

Copy the ones you want into your personal skills directory:

```bash
git clone https://github.com/RAJUSHANIGARAPU/sdlc-skills
cp -R sdlc-skills/skills/false-green-audit ~/.claude/skills/
```

Or per-project, into `.claude/skills/` in the repo.

**Read a skill before you install it.** A skill is instructions injected into an agent's
context, and installing one from anywhere — including here — means trusting it with
whatever that agent can reach. These are plain Markdown with no scripts, no hooks and no
network access, which is checkable in a minute and worth checking. Apply the same
standard to every collection you install from, especially the large ones.

## Scope

Written against Java 21 / Spring Boot / Maven / JUnit 5 / PiTest, Python / pytest,
TypeScript / Jest / Playwright, and Prometheus-style observability. The judgment
transfers; the commands assume that stack.

## Licence

MIT.
