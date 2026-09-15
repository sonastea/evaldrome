# Evaldrome Agent Guide

## Project

Evaldrome is a Go-first CLI for AI model evaluation and benchmarking.

It runs reproducible evaluation suites against multiple AI models, scores their outputs, records experiments in Braintrust, compares model performance, and can enforce evaluation thresholds in CI/CD.

Evaldrome is primarily a local and CI developer tool. It does not require a long-running server or external application database.

Use `docs/architecture.md` when working on evaluation execution, providers, Braintrust integration, suites, scoring, CI behavior, persistence, isolation, or other architectural concerns.

Do not read architecture documentation merely for trivial or isolated edits.

## Technology

- Go is the primary language.
- Use the Braintrust Go SDK where practical.
- Prefer local files and portable artifacts for configuration, datasets, and CI results.
- SQLite may be introduced later for optional local history or rankings if a clear need emerges.
- TypeScript should only be introduced when a Braintrust-hosted feature requires it or when it provides a clear advantage over Go.

Prefer the standard library and small dependencies over frameworks unless a dependency clearly improves the implementation.

## Architecture Principles

Keep the evaluation engine independent from CLI presentation.

Keep provider-specific SDK types inside provider implementations.

Expose provider behavior through Evaldrome-owned types and interfaces.

Keep Braintrust behind an internal adapter rather than coupling core evaluation logic directly to Braintrust SDK types.

The evaluation engine should remain capable of producing useful results even when remote experiment tracking is unavailable.

Prefer deterministic scoring whenever correctness can be verified programmatically. LLM judges should supplement objective tests rather than replace them.

Treat model-generated output as untrusted input.

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

Do not run paid model evaluations merely to validate ordinary unit or integration tests.

Provider behavior should be testable with deterministic fake implementations.

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

## Evaluation Behavior

Evaluation suites should be reproducible and versioned when their behavior materially changes.

Store or emit enough metadata to understand what produced a result, including where practical:

- suite name and version
- dataset identity or hash
- provider and model
- scorer configuration
- Evaldrome version
- Git commit information
- relevant execution settings

Store raw evaluation results where practical so derived summaries and rankings can be recomputed later.

Never silently rewrite historical results.

Model calls should support:

- cancellation
- timeouts
- rate-limit handling
- retry classification
- exponential backoff with jitter where appropriate

Do not blindly retry all provider errors.

## CI Behavior

CI functionality should build on the same evaluation engine used locally.

Do not create separate evaluation logic specifically for GitHub Actions or another CI provider.

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
- battles
- rankings

SQLite must remain optional unless the architecture is intentionally changed.

Do not store model-provider API keys or Braintrust credentials in the local database.

## Documentation

Update `docs/architecture.md` when a change materially alters:

- component responsibilities
- evaluation flow
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

The project should evolve incrementally:

1. local Go evaluation engine
2. reliable multi-provider execution
3. versioned suites and richer deterministic scoring
4. LLM judges and model comparisons
5. CI regression checks and portable result artifacts
6. pairwise battles and rankings
7. task/code execution with appropriate isolation
8. polished CI integrations
9. CLI/configuration stabilization
10. stable `v1.0.0`

Optional local persistence may be added when history, baselines, or rankings demonstrate a real need.

The architecture describes the intended direction, not a requirement to build every stage immediately.

When implementing a task, finish the requested behavior rather than stopping after the first plausible implementation. Inspect the result, address failures caused by the change, and leave the affected area in a working state.

## Definition of Done

A change is complete when:

- the requested behavior is implemented
- relevant validation passes
- errors and cancellation are handled appropriately
- meaningful new behavior has appropriate tests
- machine-readable behavior remains stable where applicable
