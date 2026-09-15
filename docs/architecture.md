# Evaldrome Architecture

## Overview

Evaldrome is a Go-first CLI for AI model evaluation and benchmarking.

It provides a reproducible environment for running the same tasks against multiple AI models and comparing them using:

- deterministic tests
- structured scorers
- LLM-as-a-judge evaluation
- latency
- token usage
- estimated cost
- pairwise comparisons
- model rankings
- regression thresholds

Braintrust provides experiment-oriented tracing and analysis.

Evaldrome owns:

- evaluation execution
- provider orchestration
- scoring
- result comparison
- local and CI presentation
- regression decisions
- pairwise competition and rankings

Evaldrome does not require a long-running server or external application database.

This document describes both the current system and the intended direction of the project.

It is not a requirement to build the entire architecture at once.

---

# Current Milestone

## v0.1.0 — Local Evaluation Engine

This is the current implementation target.

The goal of `v0.1.0` is to prove the fundamental Evaldrome evaluation loop with the smallest useful CLI.

Do not build:

- CI regression gates
- LLM judges
- pairwise battles
- rankings
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
- Braintrust experiment integration
- useful terminal result summaries
- unit tests for meaningful evaluation logic
- README instructions for running an evaluation

### Target flow

```text
                  evaldrome CLI
                        |
                        v
               +------------------+
               | Evaluation       |
               | Runner           |
               +--------+---------+
                        |
             +----------+----------+
             |                     |
             v                     v
       Model Provider           Scorers
          Interface
             |
        +----+----+
        |         |
        v         v
      Fake     Real Provider
                   |
                   v
               Braintrust
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

When Braintrust is configured, the run should also appear as an experiment there.

### v0.1.0 is done when

- a dataset can be loaded locally
- a fake model can execute every case deterministically
- at least one real provider works
- execution uses bounded concurrency
- deterministic scores are calculated
- results are summarized locally
- Braintrust can receive the experiment
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
         +--> model providers
         +--> scorers
         +--> Braintrust
         +--> result artifacts
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

Its value is in making model evaluation reproducible, comparable, automatable, and suitable for CI/CD.

---

# Core System

The intended CLI architecture is:

```text
                   Developer / CI
                         |
                         v
                 +----------------+
                 | Evaldrome CLI  |
                 +-------+--------+
                         |
            +------------+-------------+
            |                          |
            v                          v
        Suite /                    CLI Options
        Dataset
            |
            +------------+-------------+
                         |
                         v
                +-------------------+
                | Evaluation Engine |
                +---------+---------+
                          |
          +---------------+----------------+
          |               |                |
          v               v                v
      Providers        Scorers        Experiment
                                       Tracking
          |               |                |
     +----+----+          |                v
     |    |    |          |           Braintrust
     v    v    v          |
   OpenAI ... Google      |
          |               |
          +-------+-------+
                  |
                  v
              Results
                  |
          +-------+--------+
          |                |
          v                v
       Terminal        Artifacts
                       JSON / JUnit
```

The evaluation engine is the core reusable component.

CLI rendering, Braintrust integration, and artifact generation should sit around it rather than defining it.

---

# Evaluation Engine

The evaluation engine should not depend on:

- a specific CLI framework
- GitHub Actions
- Braintrust SDK types
- SQLite
- a specific model provider

Its responsibilities are:

1. accept an evaluation definition
2. enumerate cases
3. execute each case against selected models
4. respect concurrency and cancellation
5. collect normalized model responses
6. execute configured scorers
7. collect execution metrics
8. produce structured results
9. report lifecycle events to configured tracking adapters

This allows the same engine to run from:

```text
evaldrome run
evaldrome check
tests
GitHub Actions
other CI systems
```

without duplicating evaluation logic.

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
  +-- Model runs
        |
        +-- Case
        +-- Model output
        +-- Scores
        +-- Metrics

Comparison
  |
  +-- Baseline
  +-- Candidate
  +-- Regression rules

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
- **Experiment** — execution of a suite
- **Contender** — model participating in an experiment
- **Scorer** — objective or subjective evaluator
- **Judge** — model used for subjective or pairwise scoring
- **Battle** — blind pairwise comparison
- **Baseline** — known result used for regression comparison
- **Leaderboard** — aggregate competitive ranking

Internal APIs should prioritize clarity over themed terminology.

---

# CLI Surface

The CLI should grow gradually.

Potential commands include:

```text
evaldrome run
evaldrome validate
evaldrome compare
evaldrome check
evaldrome battle
evaldrome leaderboard
```

Not all commands are required initially.

## `run`

Execute a suite or dataset and display results.

```bash
evaldrome run ./evals/support.yaml
```

## `validate`

Validate configuration and datasets without performing model inference.

```bash
evaldrome validate ./evals/support.yaml
```

## `compare`

Compare existing result artifacts or experiments.

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

Potential providers include:

```text
OpenAI
Anthropic
Google
OpenRouter
local/self-hosted providers
```

Provider SDK-specific types must remain inside provider implementations.

The evaluation engine should not know which vendor SDK produced a response.

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

Evaluation workloads naturally fan out.

For example:

```text
100 cases
× 4 models
= 400 model requests
```

Evaldrome must not translate that directly into hundreds of uncontrolled goroutines.

The evaluation engine should use bounded concurrency.

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

Evaldrome should eventually record a dataset hash with results so comparisons can detect when the underlying dataset changed.

For `v0.1.0`, a generalized suite configuration system is not required.

A straightforward local dataset format is enough.

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

# Braintrust Integration

Braintrust is Evaldrome's primary remote experiment and model-observability integration.

Braintrust may receive:

- experiment metadata
- task inputs
- model outputs
- traces
- spans
- scores
- provider/model metadata
- suite metadata

Conceptually:

```text
Evaldrome                     Braintrust

Experiment -----------------> Experiment
Evaluation -----------------> Trace
Model call ------------------> Span
Input -----------------------> Input
Output ----------------------> Output
Scores ----------------------> Scores
Metadata --------------------> Metadata
```

Core evaluation logic should not depend directly on Braintrust SDK types.

A small adapter should translate Evaldrome lifecycle events and results into Braintrust operations.

Braintrust integration should be first-class, but the core evaluation engine should still be capable of calculating and returning results when remote tracking is disabled or temporarily unavailable.

Do not build a generic observability framework before one is required.

---

# Result Artifacts

Evaldrome should eventually produce portable machine-readable results.

Examples:

```text
results.json
summary.json
junit.xml
```

A result artifact should contain enough metadata to understand how the run was produced.

Potential metadata includes:

```text
Evaldrome version
suite name
suite version
dataset hash
provider
model
execution settings
Git commit
start/end time
scores
latency
token usage
estimated cost
```

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
0   evaluation completed and thresholds passed
1   evaluation completed but quality/regression thresholds failed
2   configuration, execution, or infrastructure error
```

The exact contract should be finalized before stable CI support.

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

A future GitHub Action should wrap the CLI rather than reimplement its behavior.

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

LLM judges are planned after the deterministic evaluation system is mature.

---

# Model Comparisons

Evaldrome should support comparing several models against the same suite.

Example:

```text
MODEL              SCORE     LATENCY     COST

Claude             94.2%       2.1s      $0.82
GPT                91.8%       1.4s      $0.38
Gemini             88.4%       0.9s      $0.16
```

Comparisons should retain separate dimensions rather than immediately creating one opaque universal score.

---

# Pairwise Battles

Evaldrome will eventually support blind pairwise comparison.

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

A strong future Evaldrome use case is evaluating models against real engineering tasks.

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

Model-generated code must be treated as untrusted.

Do not execute arbitrary generated code directly inside the main Evaldrome CLI process without isolation.

Future execution environments should consider:

- disposable filesystems
- strict timeouts
- CPU limits
- memory limits
- process limits
- restricted capabilities
- disabled or restricted networking

Possible future execution mechanisms may include containers or other sandbox technologies.

The isolation mechanism should be selected when the code-evaluation feature is implemented rather than prematurely committing the project to Kubernetes or another infrastructure platform.

---

# Repository Direction

A possible early repository structure is:

```text
evaldrome/
├── cmd/
│   └── evaldrome/
│       └── main.go
│
├── internal/
│   ├── eval/
│   ├── provider/
│   ├── scorer/
│   ├── braintrust/
│   ├── dataset/
│   └── result/
│
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

## Repository Formatting

Evaldrome's own Go source uses `gofumpt` as its default formatter. Its output is `gofmt`-compatible, with additional formatting rules for consistency.

When development tooling is introduced, pin the `gofumpt` version and use the same version for local formatting and automated checks. It is a development-tool dependency, not a requirement for users running the Evaldrome CLI.

Model-generated Go code evaluated by Evaldrome follows the formatter policy in [Task and Code Execution](#task-and-code-execution).

---

# Release Milestones

Evaldrome uses semantic versioning.

Before `v1.0.0`, each minor release represents a meaningful product milestone.

---

## v0.1.0 — Local Evaluation Engine

Ship the first useful Evaldrome.

Includes:

- Go CLI
- core evaluation engine
- fake provider
- one real provider
- local dataset
- deterministic scoring
- bounded concurrency
- cancellation
- Braintrust integration
- terminal result summary

Goal:

> Run one useful model evaluation locally.

This is the current milestone.

---

## v0.2.0 — Multi-Provider Execution

Make model execution reliable across several providers.

Potential additions:

- additional providers
- running multiple models in one experiment
- provider error normalization
- retry classification
- exponential backoff
- rate-limit handling
- improved token accounting
- estimated cost
- stronger concurrency controls

Goal:

> Run the same dataset against several providers reliably.

---

## v0.3.0 — Reproducible Evaluation Suites

Turn individual evaluations into reusable benchmark definitions.

Potential additions:

- suite configuration files
- suite versioning
- dataset hashes
- richer dataset definitions
- additional deterministic scorers
- result metadata
- suite validation

Goal:

> A benchmark can be committed to source control and reproduced by another developer.

---

## v0.4.0 — Judges and Model Comparison

Add subjective evaluation and richer comparison.

Potential additions:

- LLM-as-a-judge
- structured judge output
- model comparison reports
- multiple scoring dimensions
- cost comparison
- latency comparison

Goal:

> Compare models using both objective tests and carefully constrained subjective evaluation.

---

## v0.5.0 — CI Evaluation Gates

Make Evaldrome useful as a CI command.

Potential additions:

- `evaldrome check`
- portable baseline artifacts
- regression thresholds
- deterministic exit codes
- JSON output
- JUnit output
- Git metadata
- CI-friendly terminal output

Goal:

> Fail a build when an AI change introduces an unacceptable regression.

---

## v0.6.0 — Model Arena

Introduce the competitive evaluation layer.

Potential additions:

- blind pairwise battles
- randomized A/B ordering
- judge models
- Elo rankings
- leaderboard output
- historical battle data where persistence is available

Optional SQLite may be introduced here if maintaining local battle history and ratings clearly improves the experience.

Goal:

> Rank models through repeatable head-to-head evaluation.

---

## v0.7.0 — Task and Code Evaluation

Move beyond text-only evaluations.

Potential additions:

- workspace preparation
- model-generated patches
- deterministic command execution
- compile/test scorers
- resource limits
- isolated execution
- timeouts
- disposable workspaces

Goal:

> Evaluate models against real engineering tasks rather than only textual responses.

---

## v0.8.0 — CI Integrations

Polish integration with software delivery platforms.

Potential additions:

- official GitHub Action
- PR summaries
- CI annotations
- artifact upload/download helpers
- baseline retrieval workflows
- documentation for GitLab CI and other systems

The integrations should wrap the same Evaldrome CLI.

Goal:

> Make Evaldrome easy to add to an existing repository's CI pipeline.

---

## v0.9.0 — Stabilization

Prepare for stable external use.

Focus on:

- CLI UX
- configuration format
- error messages
- documentation
- installation
- packaging
- backwards compatibility
- deterministic output
- test coverage
- provider reliability
- security review
- performance
- migration behavior if SQLite exists

Avoid major new architecture unless necessary for `v1.0.0`.

Goal:

> Make the existing feature set dependable and pleasant to use.

---

## v1.0.0 — Stable CLI Release

`v1.0.0` represents the first stable, documented release intended for external use.

At this point Evaldrome should have:

- documented installation
- stable core CLI behavior
- stable suite/configuration format
- reliable multi-provider execution
- reproducible evaluation suites
- deterministic scorers
- Braintrust integration
- LLM judges
- comparison tooling
- CI regression gates
- pairwise model evaluation
- documented task/code evaluation
- polished CI integration
- reasonable compatibility guarantees

A server, hosted service, PostgreSQL deployment, or Kubernetes control plane is not required for `v1.0.0`.

---

# Possible Post-1.0 Directions

Future features should be driven by demonstrated user needs.

Possible directions include:

- optional SQLite-backed local history
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
v0.1.0   local evaluation engine
v0.2.0   multi-provider execution
v0.3.0   reproducible suites
v0.4.0   judges and comparisons
v0.5.0   CI evaluation gates
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
- authentication
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

The preferred architecture is a small, modular Go CLI whose capabilities expand as concrete evaluation needs emerge.

---

# Architectural Philosophy

Evaldrome should demonstrate backend and systems engineering through justified architectural decisions.

Technology should exist because the problem requires it.

Examples:

```text
Go
    CLI, evaluation engine, concurrency, provider integration,
    artifact generation, and task execution

Braintrust
    specialized AI experiment tracing and analysis

Portable files
    reproducible suites, datasets, baselines, and CI artifacts

SQLite
    optional local history when persistent state becomes useful

CI systems
    automated regression enforcement
```

The architecture is a direction, not a checklist.

For the current project state, the most important path remains:

```text
v0.1.0

Dataset
   |
   v
Evaluation Runner
   |
   +--> Model Provider
   |
   +--> Deterministic Scorer
   |
   +--> Braintrust
   |
   v
Useful Result
```

Build that well first.

When implementation experience proves an assumption in this document incorrect, update the architecture rather than forcing the code to preserve an outdated plan.
