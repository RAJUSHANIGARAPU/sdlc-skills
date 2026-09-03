---
name: false-green-audit
description: >
  Audit existing tests for false green — assertions that cannot fail, hollow
  checks (isNotNull as the only assertion), echo mocks that return the value
  being asserted, expected values that re-implement the production formula,
  swallowed exceptions, misleading skips/@Disabled, and coverage that overstates
  what is actually verified. Load when asked whether a test really tests
  anything, whether a suite would catch a regression, why a bug shipped through
  a green pipeline, before trusting a passing suite as evidence a feature works,
  when reviewing a diff whose tests look too easy, or when a coverage or
  quality gate is passing but the product is broken.
version: 1.0.0
---

# False-Green Audit

Goal: for each assertion in scope, decide **can this fail?** and prove the answer.
No verdict without a mutation or a reproduction. Suspicion is not a finding.

## Scope first

```bash
git diff --name-only origin/main...HEAD | grep -Ei '(test|spec)'   # changed tests
git diff origin/main...HEAD --stat -- '*/src/main/*' '*/src/*.py'  # changed prod code
```
Audit the tests covering the changed prod code, not just the changed tests. A test
untouched by the diff is the most likely one to have silently stopped asserting.

## Protocol — apply to every assertion, in order

| # | Question | Fails when |
|---|---|---|
| P1 | Does it execute unconditionally? | inside `if`/`for` that can be empty, after `return`, behind a try, in a helper never called |
| P2 | Is the expected value sourced independently? | expected read back from the SUT, from a mock's return, or from the same call |
| P3 | Does it exercise the real unit? | the class under test is itself a mock/spy; only `verify()` on a stub, no behaviour run |
| P4 | Is verification sufficient to tell right from wrong? | passes for both correct and incorrect output (`isNotNull`, `size() > 0`, `status < 500`, `contains("")`) |
| P5 | Is it decoupled from implementation detail? | asserts call order/private state, so a refactor reds it but a wrong result does not |
| P6 | Does it pass in isolation AND in suite order? | needs a neighbour's fixture, or only passes because a neighbour seeded data |

P6 both ways: `mvn -q test -Dtest='Cls#method'` then `-Dsurefire.runOrder=reversealphabetical`;
`pytest path::test -p no:randomly` then `pytest -p randomly`; `playwright test f.spec.ts --repeat-each=3 --workers=1`.

## Anti-pattern families to hunt

```bash
T=--glob='*[Tt]est*'  # or '*spec*'
rg -n 'assert(True|Equals\(true|NotNull)|\.isNotNull\(\);\s*$|assert .*is not None$' $T   # hollow
rg -n -A3 'catch *\((Exception|Throwable|AssertionError)|except (Exception|AssertionError)' $T # swallowed
rg -n 'catch *\{\s*\}' $T
rg -n '@Disabled|@Ignore|pytest.mark.(skip|xfail)|test\.(skip|fixme)|\.only\('              # hidden breaks
rg -no 'thenReturn\((\w+)\)|return_value *= *(\w+)' -r '$1$2' $T | sort -u                 # echo-mock candidates
rg -n 'assert\w*\(.*[*/+%-].*,.*[*/+%-]' $T                                                  # formula mirror
```

Specifics worth naming in a verdict:
- **dead assertion** — unreachable, or in a loop over a collection that is empty at runtime (P1).
- **echo mock** — `when(repo.save(any())).thenReturn(dto)` then `assertThat(result).isEqualTo(dto)`. Proves the stub, not the mapper (P2/P3).
- **formula mirror** — `assertEquals(base * rate / 100, service.premium(base, rate))`. A wrong formula in prod, copied into the test, is green forever (P2).
- **round-trip / assert-on-own-output** — write then read through the same layer, or assert back a value the test itself supplied; both-ends-broken cancels out (P2).
- **smoke posing as verification** — asserts 200/no-exception on an endpoint whose payload is wrong (P4).
- **misleading skip** — `@Disabled("flaky")` on a test failing because the feature is broken. Verify every disable reason against current code.
- **coverage without oracle** — JaCoCo counts executed lines, not verified ones. Never quote coverage as correctness evidence.

## Mechanical proof — break the code, watch it go red

The only verdict that stands. Automated, on changed classes only:

```bash
# Java
mvn -q org.pitest:pitest-maven:mutationCoverage \
  -DtargetClasses='com.example.pricing.*' -DtargetTests='com.example.pricing.*Test' \
  -DmutationThreshold=0 -DoutputFormats=HTML,XML
rg -o 'status="SURVIVED"|<mutatedClass>[^<]+|<lineNumber>[0-9]+' target/pit-reports/*/mutations.xml

npx stryker run --mutate 'src/pricing/**/*.ts'                          # Jest / Vitest
mutmut run --paths-to-mutate src/pricing/ && mutmut results             # pytest
```

Cheap manual mutation (30 seconds, works everywhere, use for UI/E2E where mutation tools do not reach) —
apply **one** to the prod code, run the single test, revert:

1. invert a boundary: `>=` → `>`, `&&` → `||`
2. change a constant: `100` → `101`, a rate, a rounding mode, a date offset
3. `return x` → `return null` / `return None` / `return []`
4. delete a side effect: comment out the `save()` / the field assignment / the event publish
5. swap two arguments of the same type at the call site

```bash
mvn -q test -Dtest='PricingServiceTest#appliesDiscount'; git checkout -- src/main
```

Stayed green → the test does not test that behaviour. Went red → record which mutation
killed it; that is the proof the assertion is live.

## Verdict format

One block per finding. No block without evidence.

```
FALSE GREEN  PricingServiceTest.java:142
  P2 (expected value not independent)
  Test asserts result equals the object stubbed at :131 (thenReturn(expectedDto)).
  Mutation: PriceMapper.toDto -> return null. Test still PASSES.
  Fix: assert field-by-field against values from the spec/fixture, not the stub.

LIVE        RestitutionTest.java:88
  Mutation: proration divisor 365 -> 360. Test FAILED as expected.
```

Close with counts: `N assertions audited · X false green · Y live · Z unproven`.
`unproven` is honest and required when the environment blocked the mutation run —
say which run was blocked and why, never upgrade it to a pass.

Rules:
- One mutation at a time; always `git checkout --` the prod file afterwards. Never commit a mutation.
- A live test asserting the wrong requirement is a **spec** finding, not a false green — report it separately.
- Do not rewrite tests in the same pass unless asked. The audit is the deliverable; fixes follow with their own proof.

## When NOT to load this

Writing new tests or features; debugging a test that is already **failing** (triage, not
an honesty audit); flake investigation where it does fail sometimes; raising a coverage
percentage with no correctness question attached; performance or security review.
