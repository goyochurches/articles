# gradle-llm-test-reviewer

A Gradle task that uses an LLM to review the **quality** of unit tests, not just their coverage. It flags tautological assertions, over-mocking, vague test names, and missing edge cases — before they reach a human code review.

## The problem

A test can have 100% line coverage and protect nothing:

```kotlin
@Test
fun `should process refund`() {
    val service = RefundService(mockGateway, mockLedger)
    val result = service.processRefund(refundRequest)
    assertNotNull(result)
}
```

This test "passes" and "covers" the line, but `processRefund` never returns `null` by signature, so the assertion can never fail. Coverage tools like JaCoCo won't catch this. Mutation testing (PIT) would catch it, but it only tells you *that* the mutant survived — not *why* the test is weak or how to fix it.

## What it does

For each test file changed in a PR, the task:

1. Reads the test file and the class it tests.
2. Sends both to an LLM (Claude, via the Anthropic API) along with a fixed rubric.
3. Gets back a JSON verdict: a 0-100 score, a list of issues with severity, and a summary.
4. Fails the build if the score falls below a configurable threshold.

## Real example

Against the same `RefundService` class, we compared a weak test and one with three behavioural cases (rejection on negative amount, success with ledger verification, gateway failure). This is Claude's actual output, unedited:

**Weak test → 8/100**

> "The single test provides almost no value: its only assertion is untriggerable, no behaviour or return-value properties are verified, collaborator interactions are unchecked, and none of the three distinct code paths (invalid amount, gateway failure, success) are deliberately targeted."

High-severity issues: tautological assertion, zero verification of mock interactions, a test name that describes the method rather than a behaviour, zero edge cases covered.

**Robust test → 72/100**

> "The three tests cover the main happy path and the two primary failure branches with appropriate mock usage and meaningful assertions, but miss the zero-amount boundary, a ledger-never-called assertion on rejection, transaction ID propagation on success, and gateway exception handling."

What's interesting about this second case: the model flagged that **no test covers what happens if `gateway.refund()` throws an exception** instead of returning a controlled failure — a case that wasn't in the explicit rubric and one a human reviewer would easily miss on a quick pass.

## Usage

### 1. Set the API key

```bash
export ANTHROPIC_API_KEY="your-key-here"
```

### 2. Run the review

```bash
./gradlew reviewTestQuality
```

Optional: adjust the minimum passing score (defaults to 70):

```bash
./gradlew reviewTestQuality -PtestReviewMinScore=60
```

### 3. Integrate into CI

Add it as a pipeline step, scoped to test files changed relative to the target branch:

```yaml
- name: Review test quality
  run: ./gradlew reviewTestQuality
  env:
    ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
```

## How it works

```
libs.versions.toml          →  build.gradle.kts          →  Prompt + rubric
(centralised versions)         (reviewTestQuality task)     (src/test/resources/prompts/)
                                        │
                                        ▼
                              Anthropic API (Claude)
                                        │
                                        ▼
                         JSON: { score, issues[], summary }
                                        │
                                        ▼
                           Fails the build if score < threshold
```

The rubric lives in `src/test/resources/prompts/test-quality-review.txt`, versioned alongside the code it evaluates — not buried in a build script:

```
Score the test file from 0-100 on these criteria:
- Assertions are behavioural, not tautological
- Collaborators are mocked only where necessary
- Test names describe the specific behaviour under test
- Edge cases relevant to the method signature are covered
```

## Limitations

- **It's a heuristic, not ground truth.** It can misjudge an intentionally simple smoke test. Treat it as a prompt for review, not a gate that overrides human judgment on its own.
- **It doesn't replace mutation testing.** PIT tells you objectively whether a mutant survived. This tool explains likely reasons faster, but the two are complementary.
- **Cost and latency.** It's meant to run only against test files changed in a PR, not the full suite on every commit.

## Stack

- Kotlin + Gradle Version Catalogs
- Spring Boot 4 (example project)
- Anthropic API (Claude)

## Related article

This repo accompanies the Parser blog article: *"Can AI Tell If Your Tests Are Actually Good? Using LLMs to Review Test Quality"*.