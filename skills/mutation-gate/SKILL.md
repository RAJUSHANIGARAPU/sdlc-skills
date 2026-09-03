---
name: mutation-gate
description: >
  Replace line/branch coverage thresholds with mutation-score gates that actually
  block a build. Load when working with mutation testing (PiTest, Stryker, mutmut,
  cosmic-ray), when a coverage percentage is being used as a quality signal or a
  merge condition, when a JaCoCo threshold is being raised or argued about, when
  coverage looks high but defects still ship, or when auditing a CI quality gate
  that reports failures without failing the pipeline.
version: 1.0.0
---

# Mutation gates

## The distinction

Line/branch coverage answers **"did this line execute"**. Mutation testing answers
**"would a test fail if this line were wrong"**. A line-coverage gate is satisfied by
tests that assert nothing — `service.calculate(input)` with no assertion moves the
JaCoCo number and catches zero defects. That is coverage theatre: the metric is
gameable in exactly the direction teams are pushed to game it. Mutation score is not —
a mutant survives precisely when no test observes the changed behaviour. Keep JaCoCo as
a cheap map of untested regions (PiTest consumes its data); stop using it as the gate.

## PiTest on a Maven / Spring Boot module

```xml
<plugin>
  <groupId>org.pitest</groupId>
  <artifactId>pitest-maven</artifactId>
  <version>1.17.0</version>
  <dependencies><dependency><groupId>org.pitest</groupId>
    <artifactId>pitest-junit5-plugin</artifactId><version>1.2.1</version>
  </dependency></dependencies>
  <configuration>
    <targetClasses><param>com.example.domain.*</param><param>com.example.service.*</param></targetClasses>
    <excludedClasses><param>*Config</param><param>*Application</param><param>*Dto</param></excludedClasses>
    <mutators><mutator>DEFAULTS</mutator><mutator>REMOVE_CONDITIONALS</mutator></mutators>
    <mutationThreshold>70</mutationThreshold><timeoutConstant>6000</timeoutConstant>
    <outputFormats><param>HTML</param><param>XML</param></outputFormats>
    <failWhenNoMutations>false</failWhenNoMutations>
  </configuration>
</plugin>
```

`mutationThreshold` is the gate — the build fails below it. Nothing else here gates.

**Target changed classes only.** A whole-module run takes tens of minutes, so it gets
moved to nightly, then ignored, then deleted. Scope to the diff with the SCM goal:

```bash
mvn -B test-compile org.pitest:pitest-maven:scmMutationCoverage \
  -DanalyseLastCommit=true -DoriginBranch=origin/main -DwithHistory -DmutationThreshold=80
```

`-DwithHistory` writes `target/pit-history` — cache it between CI runs and incremental
runs cost a fraction. Two knobs matter more than mutator choice: `excludedClasses`
(generated code, config, Lombok-only carriers — noise) and `timeoutConstant` (Spring
context startup inflates per-mutant time). `DEFAULTS` is the baseline;
`REMOVE_CONDITIONALS` yields the real findings — branches nothing asserts on.

**Read survivors, not the score.** Open `target/pit-reports/index.html`, sort by
surviving mutants, treat each as a line item: "negated conditional at
`PremiumCalculator:88` survived" means no test distinguishes the branches — write that
test. The score is a byproduct of clearing the list. Two survivor kinds are not bugs:
equivalent mutants and code with no observable contract — exclude those with a stated
reason, never by lowering the gate.

## Other stacks

TypeScript/JS — Stryker. Gate on `thresholds.break`; `high`/`low` are report colours:

```jsonc
// stryker.conf.json
{ "testRunner": "jest", "mutate": ["src/**/*.ts", "!src/**/*.spec.ts"],
  "thresholds": { "high": 85, "low": 70, "break": 70 }, "incremental": true }
```

Python — `mutmut run --paths-to-mutate src/`, then fail CI on survivors (`mutmut run
--CI`, or wrap `mutmut junitxml`). `cosmic-ray` only for per-operator control and a
session database; slower, needs a config file.

## Audit: is this gate real?

A gate that logs instead of failing is not a gate. Audit the pipeline definition:

| Fake gate | Look for | Fix |
|---|---|---|
| Failure swallowed inline | `\|\| true`, `; exit 0`, `set +e`, `-DskipTests` on the gating job | Remove it; let the non-zero exit propagate. |
| Failure declared non-fatal | `continue-on-error: true`, `catchError(buildResult: 'SUCCESS')`, `try {} catch(e) {}` with no rethrow | `catchError(buildResult: 'FAILURE', stageResult: 'FAILURE')`, or drop the wrapper. |
| Threshold that cannot trip | `mutationThreshold` absent or 0, JaCoCo `minimum` at `0.10`, Stryker `break` unset | Set it at the measured baseline less a small margin. |
| Skipped where it counts | `@DisabledIfEnvironmentVariable(named="CI", …)`, `if [ -z "$CI" ]`, a profile active only locally | Invert it or delete the test — one that skips on CI has no gate value. |
| Off the merge path | Runs in a nightly/cron job, or a pipeline outside branch protection | Move to the PR pipeline; mark the check required. |
| Report nobody reads | Artifact published, no threshold, no owner | Attach a threshold, or stop paying for the run. |

Two empirical checks: delete one assertion, push to a scratch branch, confirm the
pipeline goes red; then read branch protection — a red build that can still be merged
is advisory. Bitbucket has no `continue-on-error`, so a step is faked only in-script:

```yaml
- step:
    name: Mutation gate (changed classes)
    caches: [maven]
    script:
      - mvn -B test-compile org.pitest:pitest-maven:scmMutationCoverage
          -DanalyseLastCommit=true -DoriginBranch=origin/main -DmutationThreshold=80
    artifacts: [target/pit-reports/**]
```

```groovy
// Jenkins — the gate is the absence of a swallowing wrapper
stage('Mutation gate') {
  steps { sh 'mvn -B test-compile org.pitest:pitest-maven:scmMutationCoverage -DmutationThreshold=80' }
  post { always { archiveArtifacts 'target/pit-reports/**' } }
}
```

## Rollout

1. **Baseline, no gate.** `-DmutationThreshold=0`, publish the report, record the real
   score per module. Argue about numbers only once you have them.
2. **Gate on changed files.** `scmMutationCoverage` on the PR pipeline — new code meets
   the bar, legacy is not rewritten by a metric.
3. **Ratchet.** Start at the measured baseline, raise in small steps as survivors clear.
   An aspirational number set on day one gets disabled in week two.
4. **Never lower a threshold to make a build pass.** A surviving mutant is a missing
   test; if the number must drop, that is a decision with a name and a ticket on it.
5. **Whole-module runs stay nightly**, gating nothing, purely as a drift signal.

Counter-argument to have ready: mutation runs cost real CI minutes and yield some
unactionable survivors, so on modules with thin behavioural logic (mappers, plumbing,
config) they buy little over review. Gate where being wrong costs money.
