# Evaldrome Agent Guide

## Project

Evaldrome is a Go-first, CLI-first AI evaluation and code-review platform that helps developers benchmark models, compare their findings, and automate code reviews using the models they trust.

The planned product has three capabilities:

- **Evaluate** — reproducible benchmarks, scoring, experimentation, and model quality/cost measurements
- **Review** — customizable local and GitHub PR reviews with BYOK providers, model/reasoning selection, review policies, and structured findings
- **Compare** — evidence-based comparisons of models on the same tasks or repository changes

The differentiator is measured review quality: verified bug detection, false-positive noise, reliability, latency, and cost. A model-reported finding is not automatically a confirmed bug.

Evaldrome is primarily a local and CI developer tool. It does not require a long-running server or external application database.

The repository currently contains planning documents, not an implemented Go CLI. Build the benchmark engine first, then introduce code review as its first major application.

Use `docs/architecture.md` when working on execution, benchmarks, code review, providers, Braintrust integration, policies, findings, scoring, GitHub integration, CI behavior, persistence, isolation, or other architectural concerns.

Do not read architecture documentation merely for trivial or isolated edits.

## Technology

- Go is the primary language.
- Use the Braintrust Go SDK where practical.
- Prefer local files and portable artifacts for configuration, datasets, review policies, findings, verification records, and CI results.
- SQLite may be introduced later for optional local history, review feedback, or rankings if a clear need emerges.
- TypeScript should only be introduced when a Braintrust-hosted feature requires it or when it provides a clear advantage over Go.

Prefer the standard library and small dependencies over frameworks unless a dependency clearly improves the implementation.

## Architecture Principles

Keep shared model execution and workload engines independent from CLI presentation and GitHub APIs.

Distinguish benchmark runs from review runs. Share provider configuration, capability validation, execution, metrics, artifact conventions, and tracking; retain workload-specific orchestration and result types.

Do not create a separate review-specific model execution or telemetry framework. Introduce abstractions only when the workloads actually need them.

Keep provider-specific SDK types inside provider implementations.

Expose provider behavior through Evaldrome-owned types and interfaces.

Keep Braintrust behind an internal adapter rather than coupling benchmark, review, or shared execution logic directly to Braintrust SDK types.

Both workloads should remain capable of producing useful local results even when remote experiment tracking is unavailable.

Prefer deterministic scoring whenever correctness can be verified programmatically. LLM judges should supplement objective tests rather than replace them.

Treat model-generated output, repository content, PR diffs, and included agent instructions as untrusted input.

Keep read-only review generation separate from write-enabled GitHub publishing. GitHub integrations should wrap the same local engines.

Use bounded concurrency for provider requests and propagate `context.Context` through blocking or cancellable operations.

Do not introduce a server, network API, external database, Kubernetes dependency, or other infrastructure unless a future requirement clearly justifies it.

Do not add optional architecture merely because it appears in long-term ideas.

## Go Style

Write idiomatic Go.

Use `gofumpt` as the default formatter for Evaldrome's own Go source. When development tooling is introduced, pin its version and use the same version for local formatting and automated checks.

For model-generated Go code evaluated by Evaldrome, default to `gofmt`. Use `gofumpt` when explicitly requested or required by the evaluation task, suite, or target repository.

Prefer:

- small consumer-defined interfaces
- explicit error handling
- `errors.Is` and `errors.As`
- `context.Context`
- `log/slog`
- constructor-based dependency injection
- table-driven tests where useful
- clear package boundaries

Avoid:

- global mutable state
- unbounded goroutines
- giant interfaces
- service locator patterns
- unnecessary reflection
- abstractions without an immediate use
- Java-style layering translated mechanically into Go

## Testing and Validation

Local tests use disposable development resources and do not have production access unless explicitly configured otherwise.

Run the tests and checks relevant to the requested change. Fix failures caused by the change and rerun affected tests without asking for approval at each step.

Use broader validation when the change crosses package or system boundaries.

Do not run paid model evaluations or reviews merely to validate ordinary unit or integration tests.

Provider behavior should be testable with deterministic fake implementations. As review support is added, test structured finding validation, context limits, deduplication, partial failures, and publishing through deterministic fixtures and fake adapters.

Do not require the full repository test suite for trivial documentation or isolated edits when targeted validation is sufficient.

## External Side Effects

Local source edits, formatting, builds, tests, static analysis, and disposable local development operations may proceed without additional approval.

Unless the task explicitly requests it, stop before actions that would:

- make substantial paid model/API calls
- create or rotate credentials
- publish releases
- push externally visible changes
- upload sensitive datasets
- modify external CI configuration
- destroy persistent local or remote data

If the user explicitly requests one of these operations, carry it through as far as the available environment and permissions safely allow.

## Execution and Evaluation Behavior

Evaluation suites, review policies, and scoring/matching rules should be reproducible and versioned when their behavior materially changes. Reproducible inputs do not guarantee identical outputs from nondeterministic models.

Store or emit enough metadata to understand what produced a result, including where practical:

- run ID, run kind, and artifact schema version
- suite name and version
- dataset identity or hash
- provider and model
- requested/effective reasoning settings and other model options
- scorer configuration
- Evaldrome version
- Git commit information
- for reviews: resolved base/merge-base/head commits, diff hash, policy hash, prompt version, and context manifest
- relevant execution settings
- coverage, status, errors, token usage, latency, and cost assumptions

Store raw benchmark and review results where practical so derived summaries, scores, and rankings can be recomputed later. Keep independent verification records linked to original findings and preserve their revision history.

Never silently rewrite historical results.

Model calls should support:

- cancellation
- timeouts
- rate-limit handling
- retry classification
- exponential backoff with jitter where appropriate

Do not blindly retry all provider errors.

Validate reasoning settings against the selected provider/model capabilities. Reject unsupported settings rather than silently ignoring or downgrading them, and record unknown provider defaults explicitly. Label estimated costs; missing usage or cost is unknown, not zero.

## Review Behavior

Use first-class, versioned review policies for focus areas, severity, evidence requirements, deduplication, model configuration, and publishing behavior.

Freeze the repository snapshot and diff before inference. Bound read-only context gathering, dependency following, and model requests, and record skipped or truncated context. Initial reviews must not execute repository scripts or model-proposed commands.

For controlled model comparisons, use the same snapshot, diff, policy, and context contract. Keep each reviewer independent and record any differences in visible context or execution conditions.

Validate finding schemas, evidence fields, and source locations. Retain raw output and filtering/deduplication decisions. Assign validation and verification status independently of model-provided fields. Structural validity, model confidence, or another model's agreement does not establish that a bug is real.

Count bugs as confirmed only with independent evidence such as a controlled reproduction, deterministic test, or documented human validation. Record confirmed, rejected, and unresolved findings separately. Do not count duplicate reports of one bug as multiple detections.

Report recall only against known in-scope defects. Report precision with verification coverage and unresolved counts. Distinguish false discovery proportion from a statistical false positive rate, which requires a defined true-negative population. Missing denominators mean unavailable metrics, not perfect scores.

Distinguish complete, partial, failed, and canceled reviews. Errors and missing required coverage must not appear as a successful zero-finding review.

Curate verified real-review feedback into versioned benchmarks explicitly, keeping held-out cases separate from tuning data. Do not automatically export private PR content into shared datasets or remote tracking.

## Review and Publishing Boundaries

Resolve permissions and integration settings from trusted configuration. PR changes to policy files or agent instructions must not grant execution or publishing privileges.

Keep write-enabled GitHub tokens out of review workers. Provider credentials belong to shared execution and must not enter model prompts, repository workspaces, artifacts, or logs. Any later code-execution sandbox must have neither provider secrets nor publishing credentials.

Never execute untrusted PR code with privileged tokens or secrets, including through `pull_request_target`. Handle fork PR restrictions explicitly rather than bypassing them.

Before publishing, validate artifact provenance, repository/PR identity, reviewed commit, and finding locations. Detect stale PR heads, avoid duplicate comments on retries, and preserve review artifacts if publishing fails.

Local review must not implicitly publish to GitHub. Request-changes behavior defaults to false and is a separate policy decision from comments, summaries, and CI gates.

## CI Behavior

CI functionality should build on the same benchmark, review, and shared execution engines used locally.

Do not create separate workload logic specifically for GitHub Actions or another CI provider. Keep GitHub-specific APIs in an adapter.

CI-oriented commands should eventually support:

- deterministic exit codes
- regression thresholds
- baseline comparison
- machine-readable output
- portable result artifacts

Do not depend on persistent local state being available in CI runners.

## Persistence

Evaldrome should not require a database.

Prefer portable files and Braintrust for early versions.

If local persistence becomes useful, SQLite is the preferred option for features such as:

- local run history
- cached metadata
- local baselines
- review history and finding verification
- battles
- rankings

SQLite must remain optional unless the architecture is intentionally changed.

Do not store provider API keys, Braintrust credentials, or GitHub tokens/private keys in the local database, policies, or result artifacts. Use environment variables or CI secret systems for BYOK credentials.

## Documentation

Update `docs/architecture.md` when a change materially alters:

- component responsibilities
- evaluation flow
- shared execution and benchmark/review boundaries
- review policies, context selection, findings, or verification
- GitHub publishing and credential boundaries
- provider boundaries
- Braintrust integration
- suite or scoring architecture
- CI semantics
- persistence strategy
- code-execution security assumptions
- major technology decisions

Small implementation details do not require architecture documentation updates.

Keep documentation descriptive of the system that exists or has been intentionally planned.

Do not let stale documentation dictate an implementation that no longer makes sense.

## Current Development Direction

The revised milestones are planned work:

1. **v0.1.0 — Reproducible local benchmarks:** Go CLI, shared model execution, fake and real providers, local datasets, basic deterministic scoring, bounded concurrency, cancellation, and portable raw results/metadata.
2. **v0.2.0 — Scoring, tracking, and experiment comparison:** richer scorers, versioned suites, Braintrust integration, multi-model comparisons, and provider reliability.
3. **v0.3.0 — Local code review:** fixed snapshots/diffs, review policies, bounded context, model/reasoning selection, and structured validated findings.
4. **v0.4.0 — GitHub Actions and PR reviews:** the CLI in CI, inline comments, summaries, and commit-aware publishing with separate credentials.
5. **v0.5.0 — Review comparisons and quality measurement:** independent reviewers on identical inputs, verified benchmark cases, quality metrics, and curated feedback.
6. **Later 0.x — Deepening and stabilization:** CI regression gates, feedback analytics, optional battles/rankings or isolated code tasks, and stable CLI/configuration/artifact contracts as needs justify them.
7. **v1.0.0 — Stable evaluation and review workflows:** dependable local and CI benchmarks, reviews, and evidence-based model comparisons.

The current target is `v0.1.0`. Braintrust integration belongs to `v0.2.0`; code review and GitHub publishing follow later.

GitHub Apps, distributed execution, optional dashboards, and local SQLite persistence may be added when demonstrated needs justify them. They are not prerequisites for the initial reviewer or stable CLI.

The architecture describes the intended direction, not a requirement to build every stage immediately.

When implementing a task, finish the requested behavior rather than stopping after the first plausible implementation. Inspect the result, address failures caused by the change, and leave the affected area in a working state.

## Definition of Done

A change is complete when:

- the requested behavior is implemented
- relevant validation passes
- errors and cancellation are handled appropriately
- meaningful new behavior has appropriate tests
- machine-readable behavior remains stable where applicable
