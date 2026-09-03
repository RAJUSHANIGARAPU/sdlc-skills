---
name: incident-postmortem
description: >
  Run a live incident and write the postmortem that prevents recurrence. Load when something is broken in production right now (outage, elevated error rate, degraded latency, bad deploy), when writing or reviewing a postmortem or incident report, when the same failure has happened more than once, or when authoring a runbook for an operation a tired person must perform correctly at 3am.
version: 1.0.0
---

# Incident and Postmortem

## 1. During the incident

**Stabilise first, diagnose second.** Wanting root cause before service is restored is the main reason outages run long. Roll back, scale out, flip the flag off, fail over — then investigate. Rollback destroys no evidence: artifact, logs, metrics and traces all persist.

Exception: actions that mutate state (corrective SQL, replaying a queue, clearing a sole-copy cache). Capture first — dump affected rows, snapshot the volume, save trace IDs — that evidence does not survive.

**One named coordinator.** Say it explicitly: "I am coordinating." The coordinator does not debug. They hold the timeline, decide the next action, post updates. If they start reading logs, hand coordination off by name.

**Status update — one line each, no narrative:**

```
[14:32] IMPACT: policy creation failing, ~40% of requests, since 14:06
        STATUS: rolling back v2.14.1
        NEXT UPDATE: 14:45
```

Every 15 min while unmitigated, 30 min while degraded, once at resolution. Post on schedule even with nothing new — "no change, rollback still running" is an update. Silence makes people join and ask, which costs responders more than the update did.

**Write a timestamped log as it happens** — actions, observations, and negative results:

```
14:06 first 500s in api-gateway (alert fired 14:19 - 13m gap)
14:11 deploy billing v2.14.1 completed (candidate)
14:14 pool 12/50 - NOT saturation
14:22 rollback started    14:29 error rate at baseline
```

Memory reconstructs incidents wrongly and confidently: it compresses gaps, reverses cause and effect, and drops the theories that were wrong. This log is the postmortem's only real evidence, and the ruled-out lines are what stop the next responder re-spending 20 minutes there.

## 2. Query the data before theorising

The failure mode to name: a plausible theory formed in minute three, then evidence gathered to **support** it rather than **test** it. Write the theory down plus what you would expect to see if it were false, then query that. A query that can only confirm is the wrong query.

**Prometheus — what changed, what is saturated:**

```promql
sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m])) by (service)
  / sum(rate(http_server_requests_seconds_count[5m])) by (service)   # ratio, not raw count
histogram_quantile(0.99, sum(rate(http_server_requests_seconds_bucket[5m])) by (le, uri))
hikaricp_connections_active / hikaricp_connections_max                # saturation
jvm_memory_used_bytes{area="heap"} / jvm_memory_max_bytes{area="heap"}
rate(kube_pod_container_status_restarts_total[15m]) > 0
changes(kube_deployment_status_observed_generation[1h]) > 0           # what changed
```

Always widen to 7d and overlay: a spike that also happened last Tuesday is not about today's deploy.

**Loki — volume and shape before individual lines:**

```logql
sum(rate({namespace="prod"} |= "ERROR" [1m])) by (app)                       # where volume jumped
sum by (exception) (count_over_time({app="billing"} | json | level="ERROR" [10m]))
{app="billing"} | json | level="ERROR" | line_format "{{.trace_id}} {{.message}}"
```

Grouping first matters — one exception class is usually 90% of the noise. Check volume *before* the errors too: a drop to zero means an instance stopped serving, which reads nothing like a spike.

**Tempo — one bad trace beats a hundred log lines.** Take a `trace_id` from a Loki error line, read which span holds the time: DB call, downstream HTTP, or wall-clock gap inside your own service (GC, lock, thread-pool wait). Diff it against a successful trace for the same endpoint from before the incident — that diff usually *is* the answer.

**PostgreSQL, when traces point at the DB:**

```sql
select pid, state, wait_event_type, wait_event, now()-query_start as age, left(query,120)
from pg_stat_activity where state <> 'idle' order by age desc limit 20;
select * from pg_locks where not granted;   -- blocked, not slow
```

Long `idle in transaction` is an application bug (transaction held open across an external call), not a database problem.

## 3. The postmortem document

Write it inside 48 hours, while the log is still legible.

- **Impact** — user terms plus duration. Not "billing returned 500s" but "for 43 minutes (14:06–14:49) ~40% of policy creation attempts failed with an error page; 212 requests affected; no data loss." Unknown count → say so and make counting an action item.
- **Timeline** — from the live log. Mark when it *started* vs *detected* vs *mitigated*; those two gaps are the most useful numbers in the document.
- **Contributing conditions** (plural, deliberately). "Root cause" invites one story and one fix. Real incidents need conditions to line up: a change landed, something failed to catch it, a default was unsafe, a limit was untuned, a retry amplified it. Each one is a separate place the chain could have broken.
- **Why detection was slow** — what fired, when, what should have fired earlier.
- **Why recovery was slow** — hunting for the runbook, missing access, untested rollback, a coordinator who was also debugging.
- **Action items** — owner, ticket, and a done-condition someone else can verify.

| Bad | Good |
|---|---|
| Improve monitoring | Page on `hikaricp_connections_active/max > 0.8 for 5m` for billing, proven by a load test that trips it — owner A, OPS-441 |
| Add better tests | Integration test asserting policy creation returns 201 when pricing returns 503, in `PolicyCreationIT` — owner B, OPS-442 |
| Be careful with migrations | Pipeline fails any changeset adding a non-nullable column without a default — owner C, OPS-443 |
| Document the rollback | `runbooks/billing-rollback.md`, dry-run verified on staging by D — OPS-444 |

Bad items share a shape: no owner, no observable end state, no way to fail. "Improve monitoring" is never done and never not-done, which is why it appears in three consecutive postmortems for the same service. Cap the list at three to five that will genuinely be done; ten items is a way of doing none.

## 4. Blameless with teeth

Blameless means **no individual named as a cause**. It does not mean reaching no conclusion. The common failure is a postmortem so soft it concludes nothing: everyone agrees it was unfortunate, nothing changes, it recurs. These are system findings and they belong in the document, stated plainly:

- "The integration-test stage was configured `continueOnError` and had been failing for 11 days; the gate did not gate."
- "The PR was approved 4 minutes after opening, 340 lines changed; the process does not create time to review."
- "This alert fired 62 times in 30 days, 61 non-actionable; the team had reasonably learned to ignore it."
- "Deployed Friday 17:40 with no rollback tested; nothing in the process prevents this."

Rule for wording: attack the mechanism, name no person. "The gate did not gate" is blameless *and* damning, and it yields an action item. "We should all be more careful" is neither.

## 5. The detection question

For every incident: **what would have caught this earlier, and does that check exist?** Cheapest outward — unit test → integration/contract test → pipeline gate (schema, migration lint, dependency diff) → staging smoke or canary → alert on a symptom users feel → nothing, a customer told us (the most expensive kind).

**An incident a test could have caught is a test gap, and that is the action item most often missing.** Postmortems say "rolled back, added an alert" — an alert only shortens the *next* occurrence; the test prevents it. Write the failing test, watch it fail against the broken version, then confirm it passes against the fix. That sequence is the only proof it would have caught anything.

Also answer: how long did detection take, and is that acceptable for this failure class? A 13-minute gap on a total outage is a finding on its own.

## 6. Runbooks

Write one when the action item is "next time, do X" and X is more than one obvious command — rollback, failover, cache invalidation, replaying a dead-letter queue, manual data correction. A runbook is an admission that a human must stay in the loop, so justify it rather than automating, or automate instead.

Usable at 3am by someone who neither wrote it nor owns the service:

- **Preconditions and blast radius up top** — what it affects, what it will not fix, when *not* to run it.
- **Exact copy-pasteable commands** with `<PLACEHOLDERS>` marked. Not "restart the pods" — `kubectl -n prod rollout restart deploy/billing`.
- **Expected output after each step.** Without it the reader cannot tell a working step from a silently failing one.
- **Decision points as explicit branches** — "if `READY` is not `3/3` within 120s → stop, escalate to #platform-oncall, do not retry."
- **How to know it worked** — a named check with a threshold (`error ratio < 0.001 for 5m` on panel X), never "verify the service is healthy".
- **How to undo it**, whenever the step is not idempotent.

Keep runbooks in the service repo beside the code so they are reviewed when the code changes. A runbook not executed or dry-run since it was written is a hypothesis: run it against staging once and date it at the top.
