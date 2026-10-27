# Evaldrome Architecture

## Overview

Evaldrome is a Go-first, CLI-first AI evaluation and code-review platform.

> Help developers benchmark models, compare their findings, and automate code reviews using the models they trust.

The product has three complementary capabilities:

| Capability | Responsibility |
| --- | --- |
| **Evaldrome Evaluate** | Reproducible benchmarks, deterministic scoring, experimentation, quality, latency, cost, and eventually CI gates, battles, and rankings |
| **Evaldrome Review** | Customizable local and GitHub pull-request reviews with bring-your-own-key (BYOK) providers, model selection, reasoning settings, review policies, and structured findings |
| **Evaldrome Compare** | Compare models on the same tasks or repository changes and measure the quality, cost, and consistency of their results |

These are capabilities of one product, not separate applications or microservices.

The differentiator is evidence-based model selection: measure which reviewers find verified bugs, which produce noise, and what each review costs. A model-reported finding is not automatically a confirmed bug.

The intended feedback loop is:

```text
Benchmark models
       |
       v
Select a reviewer
       |
       v
Review real pull requests
       |
       v
Independently verify findings
       |
       v
Curate new benchmark cases
       |
       +--> Benchmark models
```

Benchmark and review runs share model execution, provider configuration, metrics, portable result storage, and Braintrust tracking. They retain distinct orchestration and result types.

Evaldrome owns benchmark execution, review context and policies, findings, scoring, comparisons, GitHub publishing, local and CI presentation, and regression decisions. Braintrust provides remote experiment-oriented tracing and analysis behind an internal adapter.

Evaldrome primarily runs on a developer machine or CI runner. It does not require a long-running server or external application database.

## Implementation Status

The repository currently contains architecture and agent guidance; the Go CLI and execution engine have not been implemented. Commands, configuration schemas, interfaces, and workflows below describe the intended direction, not shipped behavior.

Build the benchmark engine first, then make code review its first major real-world application. The roadmap is incremental, not a requirement to build the whole platform at once.

---

# Current Milestone

## v0.1.0 — Reproducible Local Benchmarks

This is the current implementation target.

The goal of `v0.1.0` is to prove the fundamental benchmark loop with the smallest useful CLI and reproducible inputs and artifacts. Reproducibility does not imply identical responses from nondeterministic providers.

Do not build:

- CI regression gates
- LLM judges
- pairwise battles
- rankings
- code-review orchestration or GitHub publishing
- Braintrust integration, which is targeted for `v0.2.0`
- SQLite persistence
- code sandboxes
- GitHub Actions integration
- a server
- an HTTP API

### v0.1.0 should include

- Go CLI
- core evaluation types
- model/provider interface
- deterministic fake provider
- at least one real model provider
- local dataset loading
- basic deterministic scoring
- bounded concurrent execution
- cancellation through `context.Context`
- basic latency and token accounting where available
- portable raw results with dataset identity/hash, model, and effective execution settings
- useful terminal result summaries
- unit tests for meaningful evaluation logic
- README instructions for running an evaluation

### Target flow

```text
Dataset + CLI options
         |
         v
Benchmark Runner
         |
         +--> Shared Model Execution --> Fake / Real Provider
         |
         +--> Basic Deterministic Scorer
         |
         v
Local Summary + Portable Results
```

A successful `v0.1.0` should allow a developer to run something conceptually similar to:

```bash
evaldrome run \
  --dataset ./datasets/example.jsonl \
  --model openai/example
```

and receive a useful result:

```text
Experiment: 01K...
Model: openai/example
Cases: 20

Passed:       17/20
Score:        85.0%
Avg latency:  1.24s
Input tokens: 12,481
Output tokens: 3,102
```

The result artifact should retain raw responses, scores, failures, and execution metadata so summaries can be recomputed locally. Braintrust experiment tracking follows in `v0.2.0`.

### v0.1.0 is done when

- a dataset can be loaded locally
- a fake model can execute every case deterministically
- at least one real provider works
- execution uses bounded concurrency
- deterministic scores are calculated
- results are summarized locally
- raw results and metadata are saved in a portable artifact
- the dataset and effective model/execution settings can be identified from the artifact
- cancellation propagates to model requests
- core evaluation logic is covered by tests
- ordinary tests do not require paid model calls

Once this works reliably, tag `v0.1.0`.

---

# Product Boundary

Evaldrome is primarily:

```text
developer machine
        or
     CI runner
         |
         v
    evaldrome
         |
         +--> configuration
         +--> dataset
         +--> repository snapshot / diff
         +--> review policy
         +--> model providers
         +--> scorers
         +--> Braintrust
         +--> result artifacts
         +--> GitHub review publishing when enabled
         |
         v
      exit status
```

Evaldrome is not currently intended to be:

- a hosted SaaS
- a multi-tenant service
- a model proxy
- an inference reseller
- a long-running daemon
- an application requiring PostgreSQL
- a Kubernetes control plane
- a replacement for Braintrust

Its value is in making model evaluation and code review reproducible, comparable, automatable, and measurable. GitHub Actions and a later optional GitHub App should invoke the same underlying workflows.

---

# Core System

The intended CLI architecture is:

```text
                 Developer / CI
                        |
                        v
                  Evaldrome CLI
                        |
           +------------+------------+
           |                         |
           v                         v
     Benchmark Runner          Review Engine
     Suite / Dataset           Snapshot / Diff / Policy
     Scorers                   Context / Findings
           |                         |
           +------------+------------+
                        |
                        v
              Shared Model Execution
              Providers / Capabilities
              Concurrency / Cancellation
              Usage / Latency / Cost
                        |
           +------------+------------+
           |                         |
           v                         v
    Portable Run Artifacts    Braintrust Adapter
           |
           +--> Terminal / Comparisons / CI Gates
           |
           +--> Review Artifacts --> GitHub Publisher
```

The shared execution layer is reusable infrastructure. Benchmark and review orchestration remain separate consumers; a review does not need to masquerade as a scored benchmark case.

| Component | Shared by benchmark and review workloads? |
| --- | --- |
| Provider/model configuration and BYOK credentials | Yes |
| Reasoning-effort configuration | Yes, where supported by the selected provider/model |
| Bounded model execution, cancellation, retries, and tracing | Yes |
| Token usage, cost, and latency measurements | Yes |
| Portable storage and Braintrust integration | Yes, with workload-specific records |
| Benchmark cases, scorers, and judges | Primarily Evaluate; reused for review-quality benchmarks |
| Repository context, policies, and finding validation | Review |
| GitHub review publishing | Review integration boundary |
| Review-quality comparison | Both |

CLI rendering, GitHub APIs, tracking adapters, and artifact serialization should sit around the engines rather than defining them. Keep read-only review generation separate from write-enabled publishing.

---

# Shared Execution and Workload Engines

The shared model-execution layer should not depend on:

- a specific CLI framework
- GitHub Actions
- GitHub SDK types or publishing credentials
- Braintrust SDK types
- SQLite
- a specific model provider

Its responsibilities are:

1. accept Evaldrome-owned model requests and execution settings
2. validate provider/model capabilities, including reasoning controls
3. apply bounded concurrency, cancellation, timeouts, and classified retries
4. collect normalized responses, usage, latency, and cost information
5. emit lifecycle events to a small tracking adapter
6. return results and errors to the calling workload

Do not introduce a separate review-specific model execution or telemetry framework. Extract shared behavior as it is needed rather than designing a universal workflow framework up front.

## Benchmark Runner

The benchmark runner is the evaluation engine for suites and datasets. It should:

1. accept an evaluation definition
2. enumerate cases
3. execute each case against selected models through shared execution
4. execute configured scorers
5. aggregate scores and execution metrics
6. produce structured benchmark results

## Review Engine

The review engine should:

1. resolve a repository snapshot, diff, and effective review policy
2. gather bounded read-only context
3. request structured findings through shared execution
4. validate findings and their source locations
5. filter and deduplicate findings while retaining raw outputs
6. produce review artifacts with coverage, execution metrics, and errors

Independent verification and review-quality scoring can later consume those artifacts. GitHub publishing is a separate adapter, not a responsibility of shared model execution.

These engines serve local commands, tests, GitHub Actions, and other CI systems without duplicating workload logic.

---

# Core Domain Model

The main concepts are:

```text
Suite
  |
  +-- Dataset
  +-- Models
  +-- Scorers
  +-- Execution configuration

Experiment
  |
  +-- Suite
  +-- Benchmark runs
        |
        +-- Model + execution settings
        +-- Case
        +-- Model output
        +-- Scores
        +-- Metrics

Review run
  |
  +-- Repository snapshot + diff
  +-- Review policy + context manifest
  +-- Model + execution settings
  +-- Raw output + structured findings
  +-- Coverage + metrics + errors

Finding verification
  |
  +-- Finding / known-defect identity
  +-- Confirmed / rejected / unresolved
  +-- Evidence + verifier + timestamp

Comparison
  |
  +-- Comparable benchmark or review runs
  +-- Quality + cost + latency dimensions
  +-- Baseline / candidate + optional regression rules

Battle
  |
  +-- Response A
  +-- Response B
  +-- Judge result

Leaderboard
  |
  +-- Battles
  +-- Ratings
```

Preferred terminology:

- **Suite** — versioned benchmark definition
- **Dataset** — collection of evaluation cases
- **Case** — individual evaluation input
- **Experiment** — benchmark execution grouping cases and model runs
- **Benchmark run** — scored execution of benchmark cases for a selected model configuration
- **Review run** — analysis of a fixed repository snapshot and diff under a review policy
- **Review policy** — versioned instructions, scope, filtering, and output requirements
- **Finding** — a model-reported issue with a location and supporting evidence
- **Verification** — an independently recorded assessment of a finding or known defect
- **Contender** — model and effective settings participating in a benchmark or review comparison
- **Scorer** — objective or subjective evaluator
- **Judge** — model used for subjective or pairwise scoring
- **Battle** — blind pairwise comparison
- **Baseline** — known result used for regression comparison
- **Leaderboard** — aggregate competitive ranking

Internal APIs should prioritize clarity over themed terminology.

Benchmark and review artifacts should carry an explicit run kind and schema version. Shared metadata does not imply identical result schemas, and a review run may exist without benchmark ground truth or scores.

---

# CLI Surface

The CLI should grow gradually.

Potential commands include:

```text
evaldrome run
evaldrome review
evaldrome review compare
evaldrome validate
evaldrome compare
evaldrome check
evaldrome battle
evaldrome leaderboard
```

Not all commands are required initially.

Evaluate, Review, and Compare are product capabilities; they do not require three separate binaries or top-level commands with exactly those names.

## `run`

Execute a suite or dataset and display results.

```bash
evaldrome run ./evals/support.yaml
```

## `review`

Analyze branch changes locally using a review policy:

```bash
evaldrome review --base main
```

Override the configured provider/model and, where supported, reasoning effort:

```bash
evaldrome review --base main \
  --model openai/example-review-model \
  --reasoning high
```

The model name is a placeholder, not a claim that an API supports that identifier or reasoning setting. Validate the actual provider/model combination before execution.

Local review should produce a terminal summary and a portable review artifact. GitHub publication is an explicitly enabled integration, not a side effect of a local review.

## `review compare`

Run independent reviewers against the same frozen changes and policy:

```bash
evaldrome review compare \
  --base main \
  --config review-comparison.yaml
```

This performs model calls. Comparing existing artifacts through `evaldrome compare` should not repeat inference.

## `validate`

Validate suite/review configuration and datasets without performing model inference.

```bash
evaldrome validate ./evals/support.yaml
```

## `compare`

Compare compatible existing benchmark or review artifacts, or tracked experiments. Report incompatible datasets, policies, context, or scoring versions rather than presenting unlike runs as a controlled comparison.

```bash
evaldrome compare baseline.json candidate.json
```

## `check`

Run or compare an evaluation and enforce configured thresholds.

Designed for CI.

```bash
evaldrome check ./evals/support.yaml
```

## `battle`

Perform pairwise comparisons between model responses.

```bash
evaldrome battle ./evals/go-backend.yaml
```

The exact command syntax should evolve from actual usage.

---

# Provider Boundary

Providers should use Evaldrome-owned interfaces and types.

A conceptual interface is:

```go
type Model interface {
	ID() string
	Provider() string

	Generate(
		ctx context.Context,
		request GenerateRequest,
	) (GenerateResponse, error)
}
```

A normalized response may contain:

```go
type GenerateResponse struct {
	Content      string
	InputTokens  int
	OutputTokens int
	Latency      time.Duration
}
```

Estimated cost may be calculated separately when provider APIs do not return reliable cost information directly.

Both workloads use the same provider configuration and credential resolution. BYOK credentials should come from environment variables or CI secret systems, not committed policies, result artifacts, or prompts.

## Model Capabilities and Reasoning

Reasoning effort belongs to the effective model configuration used by both benchmarks and reviews. Its availability and accepted values depend on the provider, model, and API endpoint; `high` or `xhigh` is not a universal capability or comparable compute budget.

Provider adapters should validate model identifiers and requested options against supported capabilities where practical, and surface API rejections clearly. Reject unsupported reasoning settings instead of silently ignoring or downgrading them. An omitted setting uses the provider default and should be recorded as such if its effective value is unknown.

Record requested settings, effective settings when known, the resolved model identifier where available, and usage including reasoning tokens when reported. Label estimated costs and retain pricing assumptions; missing usage or cost is unknown, not zero.

Provider-specific mappings belong in adapters. A small capability description is preferable to exposing vendor SDK request types throughout either workload.

Potential providers include:

```text
OpenAI
Anthropic
Google
OpenRouter
local/self-hosted providers
```

Provider SDK-specific types must remain inside provider implementations.

Neither workload engine should know which vendor SDK produced a response.

---

# Fake Provider

A deterministic fake provider should exist from the first release.

It supports tests without:

- API keys
- internet access
- paid inference
- unpredictable model behavior

The fake provider may return predefined responses based on case IDs or request content.

It should make it possible to test:

- evaluation orchestration
- scoring
- concurrency
- failures
- cancellation
- timeouts
- result aggregation

without contacting a real provider.

As review support is added, deterministic fake responses should also exercise structured findings, malformed output, duplicate findings, invalid source locations, capability validation, and partial review failures. GitHub publishing should have its own fake adapter.

---

# Provider Reliability

Real model requests should support:

```text
context cancellation
request timeout
rate-limit detection
Retry-After handling
retry classification
exponential backoff
jitter
structured errors
```

Errors may eventually normalize into categories such as:

```text
RATE_LIMITED
TIMEOUT
AUTHENTICATION
INVALID_REQUEST
PROVIDER_ERROR
CONNECTION_ERROR
CANCELED
UNKNOWN
```

Not every failure is retryable.

Authentication failures and invalid requests should normally fail immediately.

Sophisticated retry behavior does not need to block `v0.1.0`.

---

# Concurrency

Benchmark and multi-model review workloads naturally fan out.

For example:

```text
100 cases
× 4 models
= 400 model requests
```

Evaldrome must not translate that directly into hundreds of uncontrolled goroutines.

Shared execution should enforce bounded concurrency across model requests, including requests from review chunks and model comparisons.

For the first release, one global concurrency limit is sufficient.

```text
Cases
  |
  v
Bounded concurrency
  |
  +--> request
  +--> request
  +--> request
  |
  v
Results
```

Later versions may support:

```text
global concurrency
provider-specific concurrency
model-specific concurrency
```

Only introduce additional scheduling complexity when actual provider behavior requires it.

---

# Suites and Datasets

Evaluation suites should eventually be reproducible configuration files.

A future suite may resemble:

```yaml
name: support-agent
version: 1

models:
  - provider: openai
    model: example-model

  - provider: anthropic
    model: example-model

dataset:
  path: ./datasets/support.jsonl

scorers:
  - type: exact_match

execution:
  concurrency: 5
  timeout: 30s
```

Material changes to suite behavior should produce a new suite version rather than silently altering historical meaning.

Datasets should remain easy to review and version in source control.

JSONL is a reasonable initial dataset format.

Record a dataset hash with results from `v0.1.0` so comparisons can detect when the underlying dataset changed. Versioned suite configuration can follow in `v0.2.0`.

For `v0.1.0`, a generalized suite configuration system is not required.

A straightforward local dataset format is enough.

---

# Code Review Workload

Code review is the first major application of the shared execution engine beyond general benchmarks. The initial reviewer analyzes code read-only; generating patches or executing repository code is a separate later capability.

## Repository and Diff Context

A review should resolve branch names once and freeze the repository snapshot before making model calls. For `--base main`, the initial branch-review design should compare the merge base with the selected head commit and record both resolved commits and the base branch tip. Working-tree changes should require an explicit mode with their content identity captured.

Record the diff hash, included paths and content hashes, context-selection version, policy hash, and any omissions or truncation. File-count, byte, token, dependency-depth, and request limits should bound context gathering and execution. Surface skipped or unreadable files and incomplete coverage in the result.

Dependency following should stay within the permitted snapshot and configured limits. Repository content, comments, diffs, and agent instruction files are untrusted review input. Reading context must not execute repository scripts, hooks, external diff helpers, or model-proposed commands. Resolve paths and symlinks within the allowed snapshot.

## Review Policies

Policies should be first-class, versioned configuration files rather than instructions hardcoded into prompts. A conceptual `evaldrome.review.yaml` could contain:

```yaml
version: 1

review:
  enabled: true
  trigger: pull_request

  model:
    provider: openai
    name: example-review-model
    reasoning: high

  focus:
    correctness: true
    security: true
    regressions: true
    concurrency: true
    maintainability: false
    style: false

  findings:
    minimum_severity: P2
    require_evidence: true
    deduplicate: true

  github:
    inline_comments: true
    summary: true
    request_changes: false

  context:
    follow_dependencies: true
    include_agent_instructions: true
```

This is a proposed schema, not a supported configuration file yet. The provider/model name is illustrative, and reasoning support must be validated. `trigger: pull_request` describes future integration behavior; it does not require a background service or prevent a manual local review.

Focus areas control review scope. With style disabled, the reviewer should prioritize functional issues over formatting preferences. Severity should have documented meanings, for example P0 critical, P1 high, P2 medium, and P3 low; a minimum severity of P2 includes P0, P1, and P2.

Start with one repository policy plus explicit CLI overrides. Later directory and pull-request overrides should have deterministic precedence and be captured in the effective policy. Record policy content/hash and prompt-template version so reviews can be reproduced.

Load integration settings and publishing permissions from trusted configuration, such as the base revision or CI operator configuration. A pull request must not grant itself permissions by editing its policy or `AGENTS.md`. Included agent instructions can describe repository conventions, but cannot override execution, credential, or publishing boundaries. Any permitted PR-level override must remain within the trusted policy's allowed scope.

## Structured Findings

Model responses should follow a validated structured schema. An Evaldrome-owned finding record should include:

- a stable local identifier and originating review/model
- title, category, severity, and explanation
- repository-relative path, diff side, line/range, and reviewed commit
- concrete evidence and the conditions under which the issue occurs
- suggested remediation when useful
- validation and verification status

Evaldrome assigns identifiers and provenance and initializes verification as unresolved. Models supply proposed issue details, not authoritative validation or verification status.

Validate schemas, source locations, and required evidence before findings are eligible for publication. Filter by policy and deduplicate repeated reports of the same underlying issue within each review. Retain the raw response and filtering/deduplication decisions so results can be audited and rescored.

Structural validation establishes that a finding is well formed and locatable; it does not establish that the bug is real. A model's confidence or an additional judge's agreement is not independent bug confirmation.

In multi-model comparisons, match findings to common issue identities while preserving attribution to every reporting model. Duplicate descriptions of one bug must not inflate that model's recall or confirmed-bug count.

Review artifacts should distinguish complete, partial, failed, and canceled runs. Invalid output, missing context, and provider errors must not appear as a successful review with zero findings.

---

# GitHub Integration

GitHub Actions integration and PR comments are targeted for `v0.4.0`, after local review works. The integration should resolve PR metadata and invoke the same review engine used locally.

Keep two responsibilities separate:

1. **Read-only inspection:** gather a fixed PR snapshot, run model requests, and produce validated review artifacts.
2. **Write-enabled publishing:** consume validated artifacts and use narrowly scoped GitHub permissions to create inline comments and a summary.

The publisher should validate repository/PR identity, originating run, reviewed commit, and finding locations. Recheck the current PR head before publishing and report stale results rather than attaching them to changed code. Repeated publication of the same run/findings should be idempotent to avoid duplicate comments. Findings that cannot be mapped to an eligible diff line should be identified in the summary rather than attached to unrelated lines.

Posting comments and requesting changes are distinct policy decisions. `request_changes` defaults to false; enabling summaries or inline comments must not implicitly enable blocking reviews. Publishing errors should be reported separately while preserving the local review artifact.

Untrusted PR code must never run with provider secrets or write-enabled GitHub tokens. In particular, privileged workflows such as `pull_request_target` must not check out and execute PR code. Fork PRs need an explicitly designed trusted execution path or a clear unavailable status when secrets cannot be safely supplied, rather than bypassing that restriction.

Keep the write token out of review workers. The shared model executor may access provider credentials, but must not expose them in repository workspaces, model context, artifacts, or logs. Any later code-verification sandbox must have neither provider credentials nor the publisher token.

Artifact provenance matters across workflow boundaries: a publisher must not accept an arbitrary PR-uploaded artifact as a trusted review result. Validate the originating workflow/run and immutable artifact identity, enforce a strict schema and bounded output, and treat all finding text as untrusted data.

A GitHub App can later provide event delivery, installation permissions, and dispatch, while reusing the same engines and publisher. It is not needed for local review or the initial Actions integration.

---

# Scoring

Evaldrome should expose independent metrics rather than immediately reducing every evaluation to one opaque score.

Potential metrics include:

```text
correctness
build success
unit-test pass rate
integration-test pass rate
instruction following
code quality
latency
input tokens
output tokens
estimated cost
provider failure rate
```

Users should be able to understand why one model performed better than another.

---

## Deterministic Scorers

Prefer deterministic scoring whenever correctness can be verified programmatically.

Examples:

- exact match
- schema validation
- JSON validity
- regular-expression matching
- compiler success
- unit tests
- integration tests
- hidden tests
- HTTP response validation
- database-state validation

A conceptual scorer interface is:

```go
type Scorer interface {
	Name() string

	Score(
		ctx context.Context,
		input ScoreInput,
	) (ScoreResult, error)
}
```

The actual interface should emerge from implementing real scorers rather than being heavily generalized before use.

For `v0.1.0`, one or two deterministic scorers are enough.

---

# Review-Quality Benchmarks

Review quality should be measured against verified bugs, deliberately seeded defects, or independently human-validated findings. A second LLM can help match findings, triage reports, or suggest checks, but must not be the sole authority on whether a bug exists.

A benchmark case should identify its fixed repository snapshot, diff, review policy, permitted context, known defects, evidence, expected severity where available, and ground-truth version. Include representative bug-free changes as well as buggy cases. Keep hidden labels and verification answers out of reviewer prompts.

Record independent verification as confirmed, rejected, or unresolved, with verifier identity/type, evidence, and timestamp. Preserve revisions to verification records rather than silently rewriting historical results. A dismissed comment, accepted suggestion, or merged PR is feedback, not by itself proof that a finding was correct or incorrect.

## Metrics and Denominators

Define matching rules and scoring versions explicitly, and deduplicate each model's findings before computing issue-level counts.

| Metric | Meaning and requirements |
| --- | --- |
| Bug recall | Unique known in-scope bugs detected / all known in-scope bugs in the benchmark |
| Finding precision | Confirmed findings / (confirmed + rejected findings), with unresolved findings and verification coverage reported separately |
| False-positive noise | Rejected findings per review, plus rejected / adjudicated findings (false discovery proportion) |
| False positive rate | FP / (FP + TN), only when the benchmark defines a meaningful labeled negative population |
| Severity accuracy | Agreement with independently assigned severity on verified, matched bugs |
| Review cost | Reported or estimated API cost per review, with pricing assumptions and missing usage disclosed |
| Review latency | End-to-end review time, with provider-call latency separately available |
| Regression detection | Recall on verified defects introduced by the reviewed change |
| Review consistency | Agreement across repeated independent reviews of identical inputs, using versioned issue-matching rules |

In open-ended real PR reviews, the total number of bugs and true negatives is usually unknown. Do not claim recall or a statistical false positive rate from those reviews alone. The rejected share of adjudicated findings is a noise metric, not FP / (FP + TN).

Unresolved findings must not count as either confirmed or rejected. Report sample counts, verification coverage, incomplete runs, and unavailable metrics; a zero denominator means unavailable rather than a perfect or zero score. Precision on a selectively verified subset should be labeled accordingly.

A reviewer finding five possible issues is not necessarily better than one finding three. Comparisons should show verified detections, missed known defects, noise, severity, cost, and latency independently. A confirmed bug should require evidence independent of the reporting model, such as a controlled reproduction, a deterministic test, or documented human validation.

## Feedback into Benchmarks

Real reviews may generate candidate benchmark cases after verification. Curate those cases explicitly, retain their provenance and verification history, and version the resulting dataset. Keep tuning examples separate from held-out evaluation cases, including closely related changes from the same defect, to reduce leakage.

Private repository content and review artifacts must not be automatically uploaded into shared datasets. Dataset export and remote payload tracking should follow explicit repository configuration, data permissions, and retention rules. Store useful raw results locally where permitted, and record when content is omitted or redacted.

---

# Braintrust Integration

Braintrust is Evaldrome's planned primary remote experiment and model-observability integration, introduced in `v0.2.0`. Prefer the Braintrust Go SDK where practical and confirm the needed SDK capabilities during implementation.

Braintrust may receive:

- experiment metadata
- task inputs
- model outputs
- traces
- spans
- scores
- provider/model metadata
- suite metadata
- run kind, review policy, and snapshot identity
- structured findings and independently derived review-quality scores

Conceptually:

```text
Evaldrome                     Braintrust

Benchmark experiment -------> Experiment
Benchmark / review run -----> Trace
Model call ------------------> Span
Input -----------------------> Input
Output ----------------------> Output
Scores ----------------------> Scores
Metadata --------------------> Metadata
```

Benchmark, review, and shared execution logic should not depend directly on Braintrust SDK types. A review does not need to become a synthetic benchmark experiment merely to be traced.

A small adapter should translate Evaldrome lifecycle events and results into Braintrust operations.

Braintrust integration should be first-class, but both workload engines should still calculate and return useful results when remote tracking is disabled or temporarily unavailable. Surface tracking failures separately from model and scoring failures.

Repository inputs and raw findings may contain private code. Remote payload capture must follow explicit configuration and redaction rules; configuring a tracking destination should not silently enable uploads of all repository content. Credentials must never be included in tracking payloads.

Do not build a generic observability framework before one is required.

---

# Result Artifacts

Evaldrome should produce a minimal portable JSON result artifact from `v0.1.0`, expanding to review artifacts and CI-oriented formats as those workflows are introduced.

Examples:

```text
results.json
review.json
summary.json
junit.xml
```

A result artifact should contain enough metadata to understand how the run was produced.

Potential metadata includes:

```text
Evaldrome version
artifact schema version
run ID and run kind
suite name
suite version
dataset hash
provider
model
requested/effective reasoning settings
execution settings
Git commit
start/end time
scores
latency
token usage
estimated cost
status, errors, and incomplete coverage
```

Review artifacts should additionally retain base/merge-base/head commits, diff hash, effective policy and hash, prompt/context-selection versions, a context manifest, raw model output, validated findings, and filtering/deduplication decisions. Include usage and cost assumptions for all attempts when available, not just successful responses.

Verification records and cross-model issue matching should reference the original review/finding identities. Derived summaries, scores, and rankings must be recomputable without modifying raw runs. Distinguish original execution results from later human feedback or rescoring.

Portable result artifacts are particularly important for CI because CI runners are often ephemeral.

---

# Baselines

CI comparisons should not require a persistent database.

A baseline may be represented by a committed or downloaded artifact.

Conceptually:

```json
{
  "experiment": "01K...",
  "suite": "support-agent",
  "suite_version": 2,
  "dataset_hash": "a8cf...",
  "git_commit": "a51f32c",
  "metrics": {
    "correctness": 0.942
  }
}
```

Then:

```text
Baseline
    |
    +----------+
               |
               v
           Compare
               ^
               |
Candidate ------+
               |
               v
        Regression Rules
               |
          +----+----+
          |         |
          v         v
        PASS       FAIL
```

Baselines should remain portable between developer machines and CI systems.

Review-quality baselines should also identify the snapshot/diff, policy, context contract, ground-truth version, and scoring/matching configuration. Treat incompatible benchmark inputs as an explicit comparison mismatch, not a model regression.

---

# CI Mode

CI/CD is a primary product use case.

Evaldrome should eventually support a command such as:

```bash
evaldrome check ./evals/support.yaml
```

A future suite may define thresholds:

```yaml
thresholds:
  correctness:
    minimum: 0.90

  regression:
    maximum: 0.02
```

Example output:

```text
Baseline       Candidate

Correctness
92.1%          94.8%      +2.7% ✓

Cost
$0.041         $0.036     -12.2% ✓

Hallucination
2.4%           5.8%       +3.4% ✗

Evaluation failed:
hallucination regression exceeded threshold.
```

The CLI should expose deterministic exit behavior.

A possible convention is:

```text
0   run completed and configured gates passed
1   run completed but quality/regression/review-policy gates failed
2   configuration, execution, or infrastructure error
```

The exact command-specific contract should be finalized before stable CI support. Finding generation, CI gate outcomes, and GitHub `request_changes` publication are separate decisions. A successful review can contain provisional findings; provider errors or incomplete required coverage must not masquerade as a clean pass.

---

# CI Provider Independence

Evaldrome should not depend on GitHub Actions internally.

The same binary should work in:

```text
GitHub Actions
GitLab CI
Jenkins
CircleCI
Buildkite
local shells
other CI systems
```

The `v0.4.0` GitHub Action should wrap the CLI rather than reimplement benchmark or review behavior. GitHub-specific PR APIs and comment publishing belong in an adapter; local review and other CI systems should not require that adapter.

---

# Optional Local Persistence

Evaldrome does not require a database.

Early versions should rely on:

```text
configuration files
dataset files
result artifacts
Braintrust
```

If local persistence becomes useful, SQLite is the preferred database.

Potential uses include:

```text
local run history
cached metadata
local baselines
review history and finding verification
pairwise battle history
ratings
```

A possible location is:

```text
.evaldrome/evaldrome.db
```

SQLite should remain optional unless a future architectural decision intentionally changes this.

Do not store:

- OpenAI API keys
- Anthropic API keys
- Google API keys
- Braintrust API keys
- GitHub tokens or App private keys
- other provider credentials

in the SQLite database.

Environment variables and CI secret systems should remain responsible for credentials.

## CI and SQLite

CI runners are commonly ephemeral.

Therefore CI correctness must not depend on a SQLite database surviving between runs.

CI should prefer:

```text
committed baselines
downloaded artifacts
Braintrust experiment identifiers
CI-native artifacts/cache when useful
```

SQLite is for local convenience, not distributed system state.

---

# LLM Judges

LLM-based scoring is intended for qualities that cannot be reliably verified deterministically.

Examples:

- instruction following
- clarity
- usefulness
- code quality
- completeness
- hallucination assessment

Judge output should preferably be structured.

```json
{
  "instruction_following": 0.95,
  "clarity": 0.82,
  "reason": "The answer is correct but unnecessarily verbose."
}
```

A model should not judge itself when a reasonable alternative exists.

Judge scores should supplement objective tests rather than override them.

For review-quality evaluation, judges can assist with issue matching and evidence assessment, but their agreement must not automatically promote a finding to a confirmed bug. Keep judge assessments separate from independent verification records.

LLM judges are planned after the deterministic evaluation system is mature.

---

# Model Comparisons

Evaldrome Compare should support comparing several models against the same suite or frozen review inputs.

Illustrative benchmark output, not actual model performance measurements:

```text
MODEL              SCORE     LATENCY     COST

Claude             94.2%       2.1s      $0.82
GPT                91.8%       1.4s      $0.38
Gemini             88.4%       0.9s      $0.16
```

Comparisons should retain separate dimensions rather than immediately creating one opaque universal score.

## Comparing Code Reviewers

Each reviewer should run independently against the same repository snapshot, diff, review policy, and context-selection contract. Policy equivalence covers instructions, focus areas, finding rules, and context scope; contender-specific model/reasoning overrides are recorded separately. Freeze common context once where practical. Do not expose another contender's findings, ground-truth labels, or verification results before its review finishes.

Model and reasoning configurations are intended experimental variables. Record other settings, prompt versions, context budgets, truncation, retries, and errors. If provider limits require different visible context or execution conditions, surface the difference and do not present the result as a controlled same-input comparison.

A comparison report should include per-model findings, a cross-model issue matrix, independent verification status, execution coverage, latency, token usage, and cost. For example:

| Reviewer | Reported findings | Independently confirmed | Rejected | Unresolved | Estimated cost |
| --- | --- | --- | --- | --- | --- |
| Model A / setting A | 3 | 2 | 1 | 0 | $0.32 |
| Model B / setting B | 5 | 3 | 1 | 1 | $0.58 |

These are hypothetical numbers, not actual benchmark results. Overlapping findings should map to shared issue identities without losing model attribution. Confirmation requires independent evidence, and comparisons should report sample size and verification coverage before drawing conclusions.

Use repeated, representative benchmark runs to guide future reviewer selection. Preserve per-dimension tradeoffs instead of declaring a universal winner from one PR or treating raw finding counts as quality.

---

# Pairwise Battles

Blind pairwise comparison is a later benchmark capability, after the initial benchmark and review workflows.

Given:

```text
Model X response
Model Y response
```

the judge receives:

```text
Task
Reference information

Response A
Response B

Choose:
A
B
TIE
```

Model identities should be hidden from the judge.

A/B placement should be randomized to reduce positional bias.

Raw battle results should be retained when practical so ranking algorithms can be changed later.

Preference battles can assess review usefulness, but preference is not proof of bug correctness. Keep preference ratings separate from verified review-quality metrics.

---

# Rankings

The first pairwise ranking implementation can use Elo because it is straightforward and understandable.

A possible initial rating is:

```text
1500
```

Raw battle results should remain the durable source of truth.

This allows later experimentation with:

- Bradley-Terry
- Glicko
- TrueSkill
- Bayesian ranking

without rerunning model inference.

If SQLite has been introduced by this point, it may be used to persist local battles and ratings.

Otherwise rankings may be generated from result artifacts.

---

# Task and Code Execution

Beyond read-only code review, a later Evaldrome capability is evaluating models against code-generation and repair tasks. This is separate from the initial review engine and is not a prerequisite for it.

For example:

> Given a partially implemented Go REST API and requirements, implement the missing behavior.

A future pipeline may resemble:

```text
Model
  |
  v
Generate patch
  |
  v
Apply patch
  |
  +--> gofmt
  +--> compile
  +--> go vet
  +--> go test
  +--> hidden tests
  |
  v
Scores
```

For model-generated Go code, the formatting step defaults to `gofmt`. Use `gofumpt` when explicitly requested or required by the evaluation task, suite, or target repository.

The formatter choice and version should be recorded with execution settings for reproducibility. Stricter `gofumpt` formatting should affect scores only when it is part of the task or suite requirements, including applicable target-repository conventions.

Potential scores include:

```text
build
unit_tests
integration_tests
hidden_tests
instruction_following
code_quality
latency
cost
```

Objective correctness should dominate subjective judging.

---

# Code Execution Security

Model-generated code and code from reviewed repositories or PRs must be treated as untrusted.

Do not execute arbitrary generated code directly inside the main Evaldrome CLI process without isolation.

Initial reviews should inspect source read-only in bounded, isolated workspaces. If later verification runs tests or builds, it must use a separate execution sandbox rather than expanding the review worker's privileges.

Future execution environments should consider:

- disposable filesystems
- strict timeouts
- CPU limits
- memory limits
- process limits
- restricted capabilities
- disabled or restricted networking
- no provider credentials or write-enabled GitHub tokens

Possible future execution mechanisms may include containers or other sandbox technologies.

The isolation mechanism should be selected when the code-evaluation feature is implemented rather than prematurely committing the project to Kubernetes or another infrastructure platform.

---

# Repository Direction

A possible repository structure as benchmark and review capabilities develop is:

```text
evaldrome/
├── cmd/
│   └── evaldrome/
│       └── main.go
│
├── internal/
│   ├── provider/        # Provider adapters and capability mapping
│   ├── execution/       # Shared model execution and metrics
│   ├── benchmark/       # Suite and benchmark orchestration
│   ├── scorer/          # Deterministic scorers and judges
│   ├── braintrust/
│   ├── dataset/
│   ├── review/
│   │   ├── engine/      # Review orchestration
│   │   ├── context/     # Repository snapshots and diff context
│   │   ├── findings/    # Validation, filtering, and deduplication
│   │   └── policy/      # Review policy resolution
│   ├── github/          # PR metadata and publishing adapter
│   └── storage/         # Portable run artifacts and verification records
│
├── configs/
├── datasets/
├── evals/
├── docs/
│   └── architecture.md
│
├── AGENTS.md
├── go.mod
└── README.md
```

This is directional rather than mandatory.

Prefer packages based on real responsibilities rather than creating directories merely to match this document.

Start with the packages needed for `v0.1.0`. Review subpackages should emerge with the review implementation, and `storage` does not imply a database or a generic persistence framework.

## Repository Formatting

Evaldrome's own Go source uses `gofumpt` as its default formatter. Its output is `gofmt`-compatible, with additional formatting rules for consistency.

When development tooling is introduced, pin the `gofumpt` version and use the same version for local formatting and automated checks. It is a development-tool dependency, not a requirement for users running the Evaldrome CLI.

Model-generated Go code evaluated by Evaldrome follows the formatter policy in [Task and Code Execution](#task-and-code-execution).

---

# Release Milestones

Evaldrome uses semantic versioning.

Before `v1.0.0`, each minor release represents a meaningful product milestone. This revised roadmap is planned work, not a description of implemented features. It supersedes the earlier benchmark-only milestone sequence.

---

## v0.1.0 — Reproducible Local Benchmarks

Ship the first useful benchmark loop, establishing the shared model-execution boundary.

Includes:

- Go CLI
- benchmark runner and shared model execution
- fake provider
- one real provider
- local dataset
- basic deterministic scoring
- bounded concurrency
- cancellation
- latency and token accounting where available
- dataset hash and effective execution metadata
- portable raw result artifacts
- terminal result summary
- deterministic tests without paid model calls

Goal:

> Run a useful local benchmark and retain enough information to reproduce its inputs and inspect its results.

This is the current milestone. Braintrust integration and richer scoring follow in `v0.2.0`; review starts in `v0.3.0`.

---

## v0.2.0 — Scoring, Tracking, and Experiment Comparison

Make benchmark results useful for choosing between model configurations.

Potential additions:

- richer deterministic scorers
- versioned suite configuration and validation
- Braintrust experiment integration through an internal adapter
- multi-model experiments and comparison of existing result artifacts
- quality, latency, token usage, and estimated-cost reporting
- additional providers and capability-aware configuration as needed
- provider error normalization, classified retries, and rate-limit handling
- supplementary judges only where deterministic scoring is insufficient

Goal:

> Compare reproducible experiments locally and in Braintrust without coupling the engine to remote tracking.

---

## v0.3.0 — Local Code Review

Introduce code review as a workload using the same model execution and telemetry.

Potential additions:

- `evaldrome review --base main`
- fixed repository snapshots and diff analysis
- bounded read-only repository/dependency context
- versioned review policies with focus and severity controls
- BYOK model selection and validated reasoning settings
- structured findings with source locations and evidence
- finding validation, filtering, and deduplication
- portable review artifacts with coverage, metrics, and explicit partial/failure states
- deterministic fake-model tests for review behavior

Goal:

> Review a local branch with a configurable model and inspect actionable findings without requiring GitHub publishing.

---

## v0.4.0 — GitHub Actions and PR Reviews

Bring the local reviewer into pull-request workflows.

Potential additions:

- a GitHub Action wrapping the CLI
- PR snapshot and metadata resolution
- inline comments and review summaries
- opt-in request-changes policy, disabled by default
- commit-aware, idempotent publishing
- separate read-only review and write-enabled publishing responsibilities
- scoped credentials and explicit handling of untrusted/fork PRs
- portable artifacts and documented CI exit behavior

Goal:

> Run the same reviewer on GitHub PRs and publish useful findings against the correct commit.

---

## v0.5.0 — Review Comparisons and Quality Measurement

Make evidence-based reviewer selection a first-class product capability.

Potential additions:

- `evaldrome review compare`
- independent model reviews on identical snapshots, diffs, and policies
- cross-model issue matching with preserved attribution
- verified or seeded-defect review benchmark datasets
- independent finding-verification records
- recall, precision, noise, severity, cost, latency, and consistency metrics with explicit denominators
- versioned scoring and matching rules
- curated feedback from real reviews into new benchmark cases

Goal:

> Explain which reviewer performs better for a workload using verified findings and transparent cost/quality tradeoffs.

---

## Later 0.x — Deepening and Stabilization

Prioritize based on experience with benchmarks and reviews rather than assigning every optional capability a fixed release in advance.

Potential additions:

- CI regression gates, portable baselines, and JUnit output
- deeper review feedback analytics and policy overrides
- blind pairwise comparisons, judges, and rankings
- code-generation/repair evaluations and isolated verification
- additional provider and CI integrations
- optional local SQLite history when useful

Before `v1.0.0`, stabilize:

- CLI UX
- suite and review-policy formats
- artifact schemas and comparison semantics
- error messages
- documentation
- installation
- packaging
- backwards compatibility
- deterministic output
- test coverage
- provider reliability
- review and publishing trust boundaries
- performance
- migration behavior if SQLite exists

Avoid major new architecture unless necessary for `v1.0.0`.

Goal:

> Make the benchmark, review, and comparison workflows dependable and pleasant to use.

---

## v1.0.0 — Stable Evaluation and Review Workflows

`v1.0.0` represents the first stable, documented release intended for external use.

At this point Evaldrome should have:

- documented installation
- stable CLI and machine-readable behavior
- stable suite, review-policy, and artifact formats
- reliable multi-provider execution
- reproducible evaluation suites
- deterministic scorers
- Braintrust integration
- configurable local reviews and GitHub PR publishing
- validated model/reasoning configuration and BYOK credentials
- benchmark and review comparison tooling
- review-quality scoring grounded in independent verification
- documented CI behavior and execution/publishing boundaries
- preserved raw results and versioned derived scores
- reasonable compatibility guarantees

GitHub Apps, distributed execution, dashboards, pairwise rankings, and code-execution sandboxes are not prerequisites for this stable release. A server, hosted service, PostgreSQL deployment, or Kubernetes control plane is not required.

---

# Optional Later Directions

Future features, before or after `v1.0.0`, should be driven by demonstrated user needs and the maturity of the local workflows.

Possible directions include:

- optional SQLite-backed local history
- GitHub App event delivery and installation management
- distributed review/benchmark execution
- richer feedback analytics
- optional dashboard for run history and comparisons
- additional CI providers
- additional result exporters
- additional model providers
- richer ranking algorithms
- container execution backends
- remote execution backends
- Kubernetes as an optional executor
- plugin systems
- shared benchmark registries

These are possibilities, not commitments.

---

# Versioning Rules

Before `v1.0.0`:

```text
v0.MINOR.0
```

represents a meaningful feature milestone.

Examples:

```text
v0.1.0   reproducible local benchmarks
v0.2.0   scoring, tracking, and experiment comparison
v0.3.0   local code review
v0.4.0   GitHub Actions and PR reviews
v0.5.0   review comparisons and quality measurement
```

Patch releases contain fixes or small compatible improvements:

```text
v0.1.1
v0.1.2
v0.2.1
```

Breaking changes are acceptable during `0.x`, but should remain intentional and documented when user-facing.

Do not force development to match the exact milestone contents if implementation experience suggests a better sequence.

Update this document when the roadmap materially changes.

---

# Non-Goals Until Needed

Avoid adding these without a demonstrated requirement:

- HTTP server
- hosted SaaS
- PostgreSQL
- hosted user authentication
- multi-tenancy
- Kubernetes control plane
- Kafka
- RabbitMQ
- Temporal
- service mesh
- custom Kubernetes operators
- CRDs
- custom schedulers
- custom tracing backend
- custom model gateway
- complex frontend architecture
- large microservice decomposition
- dozens of provider integrations

The preferred architecture is a small, modular Go CLI whose capabilities expand as concrete evaluation and review needs emerge. GitHub integration credentials are part of the planned review workflow; they do not imply a hosted identity system.

---

# Architectural Philosophy

Evaldrome should demonstrate backend and systems engineering through justified architectural decisions.

Technology should exist because the problem requires it.

Examples:

```text
Go
    CLI, shared model execution, benchmarks, code review,
    concurrency, provider integration, and artifact generation

Braintrust
    specialized AI experiment tracing and analysis

Portable files
    reproducible suites, datasets, review policies, snapshots,
    findings, verification records, baselines, and CI artifacts

SQLite
    optional local history when persistent state becomes useful

CI systems
    benchmark automation, code reviews, and regression enforcement

GitHub adapter
    PR context and narrowly scoped review publishing
```

The architecture is a direction, not a checklist.

For the current project state, the most important path remains:

```text
v0.1.0

Dataset
   |
   v
Benchmark Runner
   |
   +--> Shared Model Execution --> Model Provider
   |
   +--> Deterministic Scorer
   |
   v
Useful Local Result + Portable Artifact
```

Build that well first. Add richer scoring, Braintrust, and experiment comparison next, then apply the same infrastructure to local review, GitHub review, and measured reviewer comparisons.

The product should help developers answer: why am I using this model, how reliable has it been, and would a different model be better for this workload?

When implementation experience proves an assumption in this document incorrect, update the architecture rather than forcing the code to preserve an outdated plan.
