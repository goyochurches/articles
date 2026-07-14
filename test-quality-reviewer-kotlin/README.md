Kotlin · Spring Boot · Testing · AI · DevOps

# Can AI Tell If Your Tests Are Actually Good? Using LLMs to Review Test Quality

## By Gregorio Iglesias

## Abstract

Code coverage is one of the most widely used metrics for assessing test quality, yet it often provides a false sense of confidence. A test suite may achieve high coverage while still failing to detect regressions due to weak assertions, excessive mocking, or missing edge cases. This article explores how Large Language Models (LLMs) can complement traditional quality assurance techniques by acting as automated reviewers for unit tests. Using practical Kotlin and Spring Boot examples, we compare LLM-based review with mutation testing and static analysis, present multiple integration strategies for CI pipelines, and discuss the strengths and limitations of this approach.

**GitHub Repository:** https://github.com/goyochurches/articles/tree/main/test-quality-reviewer-kotlin

## Introduction

A test suite can report 95% coverage and still be worthless. A test that calls a method and does `assertTrue(true == true)`, or one that mocks every collaborator so thoroughly that the only thing under test is the mock itself, still counts as "covered" in a JaCoCo report. Coverage tells you which lines ran. It says nothing about whether that test would actually catch a regression.

In this article we look at using an LLM as a second reviewer for test quality — not to replace mutation testing or code review, but to sit alongside both in CI and flag the specific patterns that quietly erode a test suite's value over time.

While reviewing a legacy Kotlin/Spring Boot service at a client, we noticed a very familiar pattern: high line coverage, low confidence. Tests existed for almost every class, but many followed the same structure: instantiate the object, call a method, check that the returned value wasn't null. No behavioural assertions, no edge cases, no verification of interactions with collaborators.

Static analysis tools like SonarQube catch some of this (duplicate code, cyclomatic complexity), and mutation testing tools like [PIT](https://pitest.org/) catch more of it by actually measuring whether tests fail when the code is deliberately broken. But mutation testing is slow on large suites, and its output — a list of surviving mutants — still needs a person to read the test and judge *why* it's weak.

We wanted something capable of reading a test file the way a senior engineer would during a code review: does this assertion make sense, is this mock hiding real behaviour, does the test name describe what's actually being verified. An LLM turned out to be a reasonable fit for that specific, judgment-heavy step.

## 1. The tools

1. **Mutation Testing (PIT / pitest-kotlin)**

- **The problem:** line coverage doesn't prove a test would catch a bug.
- **The solution:** deliberately mutate the source code (flip a condition, change a return value) and check that the test suite fails. Surviving mutants point to the weak spots.

2. **An LLM as a test reviewer**

- **The problem:** even with the list of surviving mutants, someone still has to read the test and decide exactly what's wrong — is an assertion missing, is there over-mocking, does the test not match what its own name says?
- **The solution:** pass the test file and the class under test to an LLM with a structured rubric, and get back a machine-readable verdict a Gradle task can act on.

Neither tool replaces a human reviewer. Mutation testing tells you *where* to look; the LLM review gives a first pass on *why* it's weak, in the same language a reviewer would use in a pull request comment.

### Why use an LLM instead of custom static-analysis rules?

A natural question is why not simply implement these checks as custom SonarQube rules or static-analysis plugins.

The answer is that many of the weaknesses discussed here are semantic rather than syntactic. Determining whether an assertion is meaningful, whether mocks hide the behaviour under test, or whether a test name accurately reflects the scenario requires reasoning across the entire test rather than matching predefined patterns.

Static-analysis rules are excellent at detecting deterministic issues, while LLMs are better suited to contextual judgement. Rather than replacing existing tools, the two approaches complement each other: static analysis identifies objective violations, mutation testing measures whether tests actually detect faults, and an LLM provides reviewer-like feedback on aspects that are difficult to encode as fixed rules.

## 2. Why coverage numbers hide weak tests

A few patterns show up again and again in suites with high coverage but low real confidence. They're worth seeing in code, because in a quick review all four go unnoticed.

**Tautological assertions** — the value checked could never fail, given the method's signature:

```kotlin
@Test
fun `should process refund`() {
    val result = service.processRefund(refundRequest)
    assertNotNull(result) // processRefund never returns null
}
```

**Over-mocking** — even the collaborator whose behaviour the test is supposed to verify gets mocked, so the test only proves the wiring:

```kotlin
@Test
fun `should calculate discount`() {
    val discountEngine = mock<DiscountEngine>()
    whenever(discountEngine.calculate(any())).thenReturn(10.0)

    val result = pricingService.applyDiscount(order, discountEngine)

    assertEquals(10.0, result) // only checks that the mock returned what we told it to
}
```

**Assertion-free tests** — the only failure mode is an uncaught exception:

```kotlin
@Test
fun `should not throw when processing valid order`() {
    orderService.process(validOrder) // if it doesn't throw, the test "passes" — nothing else is checked
}
```

**Copy-pasted test bodies** — ten variations of the same happy path while a real edge case goes uncovered:

```kotlin
@Test fun `test order 1`() { assertTrue(service.validate(order1)) }
@Test fun `test order 2`() { assertTrue(service.validate(order2)) }
@Test fun `test order 3`() { assertTrue(service.validate(order3)) }
// ...seven more variations of the same happy path.
// None tests an empty list, a negative amount, or an expired token.
```

None of this fails a build. All of it would fail a mutation test — just later, and only after someone runs one and interprets the result.

### Review workflow

```text
                 Developer
                     │
                     ▼
              Push Pull Request
                     │
                     ▼
         Static Analysis (SonarQube)
                     │
                     ▼
        Mutation Testing (PIT)
                     │
                     ▼
      LLM Test Quality Review
                     │
                     ▼
        JSON Review Report
      (score + detected issues)
                     │
                     ▼
      CI passes or fails based on
         the configured threshold
```

The LLM review is intentionally positioned alongside existing quality verification techniques rather than replacing them. Static analysis identifies deterministic issues, mutation testing measures the effectiveness of the tests, and the LLM provides contextual feedback similar to what a senior engineer would write during a code review.

## 3. Building an AI test reviewer

There's more than one way to wire this into the workflow, depending on how much you want it integrated into the build. We look at three, from least to most integration.

### 3.1. Standalone script

The fastest way to try the idea out: a script with no dependency on Gradle or any plugin, meant to run locally before opening a PR.

```python
import json, sys, urllib.request

PROMPT = open("prompts/test-quality-review.txt").read()

def review(test_path: str, source_path: str) -> dict:
    message = f"{PROMPT}\n\nCLASS UNDER TEST:\n```kotlin\n{open(source_path).read()}\n```\n\n"
              f"TEST FILE:\n```kotlin\n{open(test_path).read()}\n```"

    req = urllib.request.Request(
        "https://api.anthropic.com/v1/messages",
        data=json.dumps({
            "model": "claude-sonnet-4-6",
            "max_tokens": 1000,
            "messages": [{"role": "user", "content": message}]
        }).encode(),
        headers={"x-api-key": "$ANTHROPIC_API_KEY", "anthropic-version": "2023-06-01", "Content-Type": "application/json"}
    )
    with urllib.request.urlopen(req) as resp:
        data = json.loads(resp.read())
    return json.loads(data["content"][0]["text"])

if __name__ == "__main__":
    result = review(sys.argv[1], sys.argv[2])
    print(f"Score: {result['score']} — {result['summary']}")
```

```
python review.py ReactiveIntegrationTest.kt RefundService.kt
Score: 8 — The single test provides almost no value...
```

No build integration, no new dependencies in the project. Ideal for validating the rubric before committing to anything more.

### 3.2. Gradle task

The form that actually integrates the review into the build: it fails compilation if any changed test doesn't reach the threshold. This is the one we use for the rest of the article.

**`gradle/libs.versions.toml`** (extending the catalog from our [previous article](https://medium.com/@parserdigital/using-gradle-version-catalogs-to-modernise-gradle-projects-6cd8217958e3) on version catalogs):

```
[versions]
kotlin = "2.2.21"
springBoot = "4.0.2"
okhttp = "5.0.0"
kotlinxSerialization = "1.8.0"

[libraries]
okhttp = { module = "com.squareup.okhttp3:okhttp", version.ref = "okhttp" }
kotlinx-serialization-json = { module = "org.jetbrains.kotlinx:kotlinx-serialization-json", version.ref = "kotlinxSerialization" }
```

**The review prompt** — kept in a resource file so it's versioned alongside the code it judges, rather than buried in a build script:

```
src/test/resources/prompts/test-quality-review.txt
```

```
You are reviewing a single Kotlin unit test file against the class it tests.

Score the test file from 0-100 on these criteria:
- Assertions are behavioural, not tautological (checking a value that can
  never fail)
- Collaborators are mocked only where necessary; the class's own logic is
  exercised
- Test names describe the specific behaviour under test
- Edge cases relevant to the method signature are covered (nulls, empty
  collections, boundary values, error paths)

Respond ONLY with JSON, no markdown fences, matching:
{
  "score": <0-100>,
  "issues": [
    { "line": <int>, "severity": "high|medium|low", "description": "<string>" }
  ],
  "summary": "<one sentence>"
}
```

**The Gradle task**, calling the API and failing the build below a threshold:

```kotlin
import kotlinx.serialization.json.Json
import kotlinx.serialization.Serializable
import okhttp3.MediaType.Companion.toMediaType
import okhttp3.OkHttpClient
import okhttp3.Request
import okhttp3.RequestBody.Companion.toRequestBody

@Serializable
data class ReviewIssue(val line: Int, val severity: String, val description: String)

@Serializable
data class ReviewResult(val score: Int, val issues: List<ReviewIssue>, val summary: String)

tasks.register("reviewTestQuality") {
    group = "verification"
    description = "Uses an LLM to flag weak assertions, over-mocking, and missing edge cases in changed test files"

    doLast {
        val minScore = (project.findProperty("testReviewMinScore") as String? ?: "70").toInt()
        val client = OkHttpClient()
        val prompt = layout.projectDirectory
            .file("src/test/resources/prompts/test-quality-review.txt")
            .asFile.readText()

        val changedTests = getChangedTestFiles() // git diff against the target branch
        var failed = false

        changedTests.forEach { testFile ->
            val sourceFile = resolveClassUnderTest(testFile)
            val body = buildRequestBody(prompt, testFile.readText(), sourceFile.readText())

            val request = Request.Builder()
                .url("https://api.anthropic.com/v1/messages")
                .addHeader("x-api-key", System.getenv("ANTHROPIC_API_KEY"))
                .addHeader("anthropic-version", "2023-06-01")
                .post(body.toRequestBody("application/json".toMediaType()))
                .build()

            client.newCall(request).execute().use { response ->
                val result = parseReviewResult(response.body!!.string())
                println("${testFile.name}: score=${result.score} — ${result.summary}")
                result.issues.filter { it.severity == "high" }
                    .forEach { println("  [HIGH] line ${it.line}: ${it.description}") }

                if (result.score < minScore) failed = true
            }
        }

        if (failed) throw GradleException("One or more test files scored below $minScore. See output above.")
    }
}
```

### 3.3. A direct CI step, bypassing Gradle

For polyglot repos where not every service uses the same build tool, it sometimes makes more sense to run this as an independent pipeline step with a direct `curl` call:

```yaml
- name: Review changed test files
  run: |
    for f in $(git diff --name-only origin/main -- '*Test.kt'); do
      source_file=$(echo "$f" | sed 's/src\/test/src\/main/; s/Test\.kt/.kt/')
      curl -s https://api.anthropic.com/v1/messages \
        -H "x-api-key: $ANTHROPIC_API_KEY" \
        -H "anthropic-version: 2023-06-01" \
        -H "Content-Type: application/json" \
        -d "$(jq -n --arg p "$(cat prompts/test-quality-review.txt)" \
                    --arg t "$(cat "$f")" \
                    --arg s "$(cat "$source_file")" \
              '{model:"claude-sonnet-4-6", max_tokens:1000, messages:[{role:"user", content:($p+"\n\n"+$t+"\n\n"+$s)}]}')" \
        | jq -r '.content[0].text'
    done
  env:
    ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
```

Less elegant than the Gradle task, but zero JVM dependencies, and it works just as well in front of a Go or Python repo.

## 4. What it caught in practice

We ran the review against two tests for the same `RefundService` class: one deliberately weak, and one with three behavioural cases. This is Claude's actual output via the Anthropic API, unedited.

**Weak test:**

```kotlin
@Test
fun `should process refund`() {
    val service = RefundService(mockGateway, mockLedger)
    val result = service.processRefund(refundRequest)
    assertNotNull(result)
}
```

```json
{
  "score": 8,
  "issues": [
    { "line": 4, "severity": "high", "description": "Assertion is tautological: assertNotNull(result) can never fail since processRefund always returns a non-null RefundResult regardless of input or logic path." },
    { "line": 3, "severity": "high", "description": "No verification that gateway.refund() or ledger.record() are called with the correct arguments, so the collaborator interaction logic is entirely untested." },
    { "line": 2, "severity": "high", "description": "Test name 'should process refund' describes the method, not a specific behaviour or outcome; it gives no information about which code path is exercised." },
    { "line": 1, "severity": "high", "description": "Zero edge cases are covered: no test for amount <= 0 (rejection path), no test for a failed gateway response, no test for amount == 0 boundary, and no test for the happy-path result being RefundResult.completed with the correct transactionId and amount." },
    { "line": 4, "severity": "medium", "description": "The happy-path result is not inspected for its type (completed vs rejected/failed) or its field values (transactionId, amount), so even a refund that returns RefundResult.rejected would pass this test." },
    { "line": 3, "severity": "medium", "description": "mockGateway and mockLedger are referenced but their stub behaviour (e.g. what gateway.refund returns) is not shown, making it unclear whether the success branch or failure branch is being exercised." }
  ],
  "summary": "The single test provides almost no value: its only assertion is untriggerable, no behaviour or return-value properties are verified, collaborator interactions are unchecked, and none of the three distinct code paths (invalid amount, gateway failure, success) are deliberately targeted."
}
```

**Robust test** (rejection on negative amount, success with ledger verification, gateway failure):

```json
{
  "score": 72,
  "issues": [
    { "line": 10, "severity": "medium", "description": "Zero amount (boundary value) is not tested. The guard condition is amount <= 0, so 0.0 is a distinct rejection case from negative values and should be covered explicitly." },
    { "line": 3, "severity": "low", "description": "Only a negative amount (-10.0) is used to test rejection; a zero amount and a very large negative value would round out boundary coverage without much extra cost." },
    { "line": 7, "severity": "medium", "description": "The rejection test does not assert that the ledger is also never called (verify(mockLedger, never()).record(...)). The class could theoretically write to the ledger before returning, and this test would not catch it." },
    { "line": 21, "severity": "low", "description": "The success-path test does not assert that result.transactionId equals \"tx-2\". The RefundResult.completed factory receives the transaction ID, and verifying it here would prevent a regression where the wrong ID is returned." },
    { "line": 35, "severity": "medium", "description": "No test covers the case where the gateway throws an exception rather than returning a failure response. If gateway.refund throws, the current tests give no signal about whether the service handles or propagates it." },
    { "line": 1, "severity": "low", "description": "There is no test for a very small positive amount just above the boundary (e.g., 0.001) to confirm it is accepted and flows through to the gateway, complementing the rejection boundary tests." }
  ],
  "summary": "The three tests cover the main happy path and the two primary failure branches with appropriate mock usage and meaningful assertions, but miss the zero-amount boundary, a ledger-never-called assertion on rejection, transaction ID propagation on success, and gateway exception handling."
}
```

The score gap (8 vs. 72) matches what any human reviewer would say looking at both files, but the interesting part is in the detail of the second case. The "robust" test already covers the happy path and the two main failures with reasonable mocks — and the model still found six real improvement points that weren't in our explicit rubric: the exact boundary at `amount == 0` (distinct from a simply negative amount), the missing check that the ledger is *not* touched on rejection, the absence of a check on the returned `transactionId`, and — the most valuable of the six — that no test covers what happens if `gateway.refund()` throws an exception instead of returning a controlled failure.

That last one is exactly the kind of comment a senior reviewer would leave on a PR after thinking about it for a while, not something that falls out of mechanically applying the rubric.

## 5. Where this breaks down

Some honest limitations worth stating up front:

- **It's a heuristic, not ground truth.** The model can misjudge intentionally simple tests (a smoke test that only checks "does this throw?") as weak. Treat the score as a prompt for review, not a gate that overrides human judgment on its own.
- **Cost and latency add up.** Reviewing every test file on every commit is unnecessary; scoping the task to changed files in a PR keeps it fast and cheap.
- **It doesn't replace mutation testing.** PIT tells you objectively whether a mutant survived. The LLM review explains likely reasons faster than reading raw mutant output, but the two are complementary, not interchangeable.

## Repository

The complete source code used throughout this article, including the Gradle task, prompts, sample services, and supporting scripts, is available in the accompanying GitHub repository:

https://github.com/goyochurches/articles/tree/main/test-quality-reviewer-kotlin

Readers can use the repository to reproduce the examples presented in this article, adapt the prompts to their own projects, and experiment with different LLM providers or review criteria.

## Conclusion

Coverage percentage answers **"Did this line run?"** Mutation testing answers **"Would this test catch a bug here?"** An LLM review sits between the two: it provides contextual feedback on *why* a test may be weak, highlighting issues such as tautological assertions, excessive mocking, poor behavioural verification, and missing edge cases using language familiar to code reviewers.

This approach is not intended to replace mutation testing or human review. Instead, it complements existing quality assurance techniques by reducing the effort required to identify low-value tests during continuous integration. Used as an advisory quality gate rather than an absolute authority, LLM-based review offers a practical way to improve test quality without significantly increasing build complexity or execution time.

As LLM capabilities continue to improve, they are likely to become another standard tool in the software quality toolbox, helping development teams focus their attention on the tests that matter most while leaving deterministic checks to traditional analysis tools.

## References

1. Jia, Y., & Harman, M. (2011). *An Analysis and Survey of the Development of Mutation Testing*. IEEE Transactions on Software Engineering, 37(5), 649–678.
2. PIT Mutation Testing Documentation. https://pitest.org/
3. JaCoCo Java Code Coverage Library. https://www.jacoco.org/jacoco/
4. SonarQube Documentation. https://docs.sonarsource.com/
5. Anthropic API Documentation. https://docs.anthropic.com/
6. Martin Fowler. *The Practical Test Pyramid*. https://martinfowler.com/articles/practical-test-pyramid.html
7. Google. *Software Engineering at Google*. https://abseil.io/resources/swe-book/
8. OpenAI. *SWE-Lancer: Can Frontier LLMs Earn $1 Million from Real-World Freelance Software Engineering?* https://arxiv.org/abs/2502.12115