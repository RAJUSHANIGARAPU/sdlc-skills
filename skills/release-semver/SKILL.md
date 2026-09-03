---
name: release-semver
description: >
  Judgment for cutting releases and versioning. Load when deciding a version bump,
  judging whether a change is breaking, writing release notes or a changelog, planning
  a deprecation and its removal, coordinating bumps across multi-module Maven or
  dependent published packages, or shipping a migration under a rolling deploy.
version: 1.0.0
---

# Release & semver judgment

Semver is a contract about **consumer breakage**, not about how big the diff felt.

## 1. The asymmetry rule

Direction decides everything. Ask: who supplies this value, me or my consumer?

| Surface | Safe (MINOR) | Breaking (MAJOR) |
|---|---|---|
| **Response you return** | ADDING a field | REMOVING, renaming, narrowing, or making a field newly absent |
| **Request you accept** | ACCEPTING more (new optional field, wider range) | REQUIRING more, or accepting less |
| **Enum you emit** | — | adding a value (consumers switch on it) |
| **Enum you accept** | adding a value | removing a value |

Corollary: **consumers decide whether you broke them.** If real traffic depends on
it, it is part of the contract regardless of what the docs, the OpenAPI spec, or
your intent said. "It was never guaranteed" is a postmortem line, not a defence.

## 2. What actually counts as breaking

Each of these is MAJOR (or a new API version) even though none removes anything:

- **Adding a required request field** — every existing caller's payload is now invalid; they 400 on the next deploy.
- **Adding an enum value to a response** — clients with an exhaustive `switch`/`when` hit the default branch, throw, or silently mis-route.
- **Narrowing an accepted range** (max 1000 → 100) — payloads that worked yesterday are rejected today.
- **Changing a default** — callers who omitted the field get different behaviour without changing a line of code. The most under-reported break there is.
- **Nullable response field → non-nullable** — *not* safe: consumers who tested `!= null` are fine, but anyone who used the null case as a signal ("absent means not yet calculated") loses that signal. Non-nullable → nullable **is** breaking: deserialisation into a non-null Kotlin/record type or a `@NotNull` DTO blows up.
- **Tightening validation** (regex, length, cross-field rule) — same shape as narrowing a range: previously-accepted real data now fails. Check production data against the new rule before calling it a PATCH.
- **Changing an error code, or a message clients match on** — retry logic, alert rules, and `if (msg.contains(...))` all key off these. Error codes are API. Messages become API the moment someone greps them.
- **Renaming a JSON property** — a rename is a remove plus an add. If both are emitted for a window, it is MINOR; if only the new name ships, MAJOR.
- **Changing ordering that was never guaranteed** — an unsorted list that happened to come back insertion-ordered, now paged or parallel-fetched. Consumers index `[0]`. Breaking in practice; the fix is to *guarantee* an order, not to argue.
- **A migration older pods cannot tolerate** — see §6. During a rolling deploy the old and new code run against one schema; a drop or rename breaks the old pods mid-release.

Genuinely MINOR/PATCH: new optional request field, new response field, new endpoint,
relaxed validation, performance work, and a bug fix where **no** consumer could
reasonably depend on the wrong behaviour — if any could, it is breaking; ship it
behind a flag or a new version.

## 3. Deprecation that works

Four steps, in order. Skipping step 3 is the common failure.

1. **Announce** — mark it (`@Deprecated(since, forRemoval)`, `DeprecationWarning`, `Deprecation`/`Sunset` headers), name the replacement and a target release in the same breath. A deprecation with no successor is just a complaint.
2. **Keep both paths working** — both must be tested. An untested deprecated path rots and breaks before you remove it.
3. **Instrument** — count calls per consumer (metric with a caller/client-id label, gateway access-log query). This is the step people skip.
4. **Remove when the data says zero** — over a full business cycle, not a quiet week. Monthly and quarterly batch jobs are exactly the callers your two-week window misses.

**Failure mode:** a deprecation notice with no telemetry. Removal then rests on
"nobody complained", which is indistinguishable from "nobody read the notice" — you
find out during an incident, from the consumer you did not know existed.

## 4. Multi-module Maven and dependent packages

Deciding what to bump when one module changed:

- **Bump the changed module by what its own consumers see.** Internal refactor with an unchanged public surface: PATCH.
- **Transitive breaking change is the trap.** If `A` goes MAJOR and `B` depends on `A` *and re-exposes A's types* in its own signatures, `B` is also MAJOR — B's consumers must move. If `B` merely uses `A` internally and absorbs the change, `B` is PATCH. Test: does an A type appear in B's public API? `mvn dependency:tree` shows the edge; only reading B's signatures answers the question.
- **Uniform-version release** (one version for the whole reactor, `revision` property) is right when the modules ship and deploy as one unit — nobody consumes a single module standalone. It **hides breaks** as soon as a module is published independently: everything reads MAJOR, so the one real break is invisible and consumers stop reading bumps.
- **Independent versions** are right for anything published with its own consumers (artifact repo, PyPI). Cost: a version matrix plus a BOM/`dependencyManagement` to keep it coherent. Python follows the same re-export test via `project.dependencies`.

## 5. Release notes people read

Organise by **what the reader must do**, never by commit type. `feat:`/`fix:` is
input to the bump decision, not an output section.

**Bad** — commit-type sections, no action, no consumer view:
```
### Features
- feat(api): add validation to POST /contracts (#412)
### Fixes
- fix(core): correct null handling in ContractMapper (#418)
```

**Good:**
```
## 4.0.0

### Action required
- `POST /contracts` now requires `startDate` (ISO-8601). **Action:** add the field
  to every caller before upgrading; requests without it return 400 `FIELD_REQUIRED`.
- Error code `VALIDATION_ERROR` split into `FIELD_REQUIRED` / `FIELD_FORMAT`.
  **Action:** update code or alerts matching on the old code.

### Behaviour changed
- `GET /contracts` is now sorted by `createdAt` desc (previously unspecified).
- `retryCount` default 3 → 1. Set it explicitly to keep the old behaviour.

### Deprecated
- `GET /v1/contracts/search` — replaced by `POST /v1/contracts/query`. Removal in 5.0.0.

### Fixed
- `ContractMapper` no longer returns a partially-populated result when `endDate` is absent.

### Internal
- Dependency and build changes; no API impact.
```
Rules: every "action required" entry states the action imperatively. Name the
symbol/endpoint, not the PR. Note removed defaults explicitly. Anything with no
consumer impact goes under Internal or is omitted.

## 6. Rolling deploys and Liquibase (expand/contract)

Under a rolling deploy old and new pods serve the same schema simultaneously. Every
migration must be tolerable by **both** versions of the code.

**Hazard:** a single changeset that renames or drops a column. The moment it runs,
old pods still in the rotation `SELECT` a column that no longer exists — every
request they serve fails until they are replaced, and rollback is worse because the
new pods then break. This looks green in CI, where only the new code exists.

Split across releases:

1. **Expand** (release N) — additive only: `addColumn` nullable, no default backfill in the same transaction, new table/index. Old code ignores it.
2. **Tolerate** (release N) — new code writes both old and new shapes and reads new-with-fallback-to-old.
3. **Backfill** (release N, separate changeset or job) — batched, restartable, idempotent.
4. **Contract** (release N+1, after every pod runs N) — drop the old column, add `NOT NULL`, drop the fallback read.

Also: every changeset gets a written `rollback`; never edit a merged changeset
(checksum failure on every environment that already ran it) — add a new one.

## 7. Pre-release checklist

Verification, not ceremony. Each line has a command that can fail.

| Claim | Proof |
|---|---|
| No unintended API break | `mvn verify` with japicmp/revapi bound to the build; Python: `griffe check <pkg> -a <last-tag>` |
| Consumers still satisfied | contract/Pact verification against the last released consumer versions, not only HEAD |
| Bump matches the diff | `git log --oneline <last-tag>..HEAD` — any `!`/`BREAKING CHANGE` present ⇒ MAJOR, or justify in the PR |
| Migration is rolling-safe | run N-1 code's test suite against the migrated schema; `liquibase updateSQL` read for DROP/RENAME/NOT NULL |
| Migration reversible | `liquibase update && liquibase rollbackCount <n>` on a throwaway database |
| Deprecated paths still work | tests exist and pass for both old and new path |
| Removals are safe | usage metric for the removed path reads zero over a full business cycle |
| Version metadata coherent | `mvn versions:display-dependency-updates`, BOM aligned, no `SNAPSHOT` in the release tree |
| Notes cover every break | each MAJOR item in §2 that applies has an "Action required" entry |
| Artifact is the tested one | tag equals the commit CI built; checksum/digest matches what deploys |
