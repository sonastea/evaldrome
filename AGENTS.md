# Evaldrome Agent Guide

## Project

Evaldrome is a Go-first, CLI-first, self-hostable AI evaluation and code-review platform that helps developers benchmark models, compare their findings, and automate code reviews using the models they trust.

The planned product has three capabilities:

- **Evaluate** — reproducible benchmarks, scoring, experimentation, and model quality/cost measurements
- **Review** — customizable local and GitHub PR reviews with BYOK providers, model/reasoning selection, review policies, and structured findings
- **Compare** — evidence-based comparisons of models on the same tasks or repository changes

The differentiator is measured review quality: verified bug detection, false-positive noise, reliability, latency, and cost. A model-reported finding is not automatically a confirmed bug.

Evaldrome has two planned deployment modes: a standalone local/CI CLI and an optional self-hosted service for an individual or multiple invited users. The CLI must remain usable without a server, account, or database. The service reuses the same engines and adds authentication, workspace isolation, durable jobs, and a small web UI/API.

The stable-release target includes single-host, multi-user self-hosting. Start with invite-only access and workspace-scoped data; public registration, billing, and distributed execution are later decisions. Self-hosting Evaldrome does not imply that inference is local: cloud providers send selected inputs off-host, while an explicitly supported local provider permits local inference. Braintrust is optional in both modes.

The repository currently contains planning documents, not an implemented Go CLI. Build the benchmark engine first, then introduce code review as its first major application.

Use `docs/architecture.md` when working on execution, benchmarks, code review, providers, Braintrust integration, policies, findings, scoring, GitHub integration, CI behavior, self-hosting, authentication, workspaces, persistence, isolation, or other architectural concerns. Use `docs/release-checklist.md` for milestone acceptance and stable-release evidence; mark items complete only when implemented and validated.

Do not read architecture documentation merely for trivial or isolated edits.

## Technology

- Go is the primary language.
- Use the Braintrust Go SDK where practical.
- Prefer local files and portable artifacts for configuration, datasets, review policies, findings, verification records, and CI results.
- SQLite is optional for standalone CLI history and required for the planned single-host service's users, workspaces, durable queue, and metadata. Use portable files for large inputs and result artifacts.
- The service should start as one Go application with embedded web assets and bounded workers, deployable as a binary or container on a Linux host.
- TypeScript should only be introduced when a Braintrust-hosted feature requires it or when it provides a clear advantage over Go.

Prefer the standard library and small dependencies over frameworks unless a dependency clearly improves the implementation.

## Architecture Principles

Keep shared model execution and workload engines independent from CLI/HTTP presentation, authentication, service storage, and GitHub APIs.

Distinguish benchmark runs from review runs. Share provider configuration, capability validation, execution, metrics, artifact conventions, and tracking; retain workload-specific orchestration and result types.

Do not create a separate review-specific model execution or telemetry framework. Introduce abstractions only when the workloads actually need them.

Keep provider-specific SDK types inside provider implementations.

Expose provider behavior through Evaldrome-owned types and interfaces.

Keep Braintrust behind an internal adapter rather than coupling benchmark, review, or shared execution logic directly to Braintrust SDK types.

Both workloads and deployment modes must work with remote experiment tracking disabled or unavailable. Do not require an Evaldrome cloud account or hosted identity provider.

Prefer deterministic scoring whenever correctness can be verified programmatically. LLM judges should supplement objective tests rather than replace them.

Treat model-generated output, repository content, PR diffs, and included agent instructions as untrusted input.

Keep read-only review generation separate from write-enabled GitHub publishing. GitHub integrations should wrap the same local engines.

Use bounded concurrency for provider requests and propagate `context.Context` through blocking or cancellable operations.

Introduce the planned server and HTTP API at the self-hosting milestone, after the local engines work. Do not make them dependencies of standalone commands. PostgreSQL, external queue brokers, Kubernetes, and multiple service replicas are outside the initial self-hosted design.

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

Service tests must cover cross-workspace authorization, credential selection, request idempotency, queue recovery, cancellation, quotas, artifact access, and backup/restore. Use fake providers for restart and concurrency tests; never assume a remote inference request is exactly-once.

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

## Self-Hosted Service Boundaries

Treat the server as an adapter around the same benchmark, review, comparison, and publishing behavior used locally. HTTP handlers authorize and validate requests, durably enqueue work, and return a run ID; workers execute the engines outside the request lifetime.

Require authentication and workspace authorization for every user-facing resource, including inputs, jobs, results, logs, downloads, comparisons, and verification records. Derive the actor from the authenticated session/token and validate workspace membership server-side. A workspace identifier is not authorization. Instance operators control the host and are trusted with its data.

Start with local, invite-only accounts, workspace admin/member/viewer roles, and revocable workspace-scoped API tokens. Do not require third-party login. Protect browser sessions and state-changing requests, and do not expose an unauthenticated setup flow on the network.

Resolve provider, tracking, repository-read, and publishing credentials through operator-managed secret references with explicit workspace grants. Keep raw secrets out of SQLite, user-visible configuration, prompts, artifacts, and logs. Never allow users to select arbitrary environment-variable names or inherit the operator's credentials implicitly.

Service inputs refer to authorized uploaded resources or registered repositories, not arbitrary host paths or user-selected network endpoints. Bound uploads, context, network access, execution time, queue size, provider requests, and storage. Keep run workspaces isolated; do not execute uploaded or repository code.

Enforce global and per-workspace concurrency and usage limits so one user cannot exhaust the instance or spend another user's provider budget. Missing prices are unknown; estimated-cost limits cannot guarantee a provider invoice cap.

Persist job state before acknowledging submission. Preserve queued jobs across restart; mark interrupted running jobs explicitly and retain completed work. Do not automatically replay an ambiguous in-flight provider request. Keep cancellation, inference retries, and publishing retries distinct.

Use a single active server with SQLite on local persistent storage. Document migrations, consistent backups, restore, retention, deletion, and recovery before declaring self-hosting stable.

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

Standalone local and CI execution must not require a database. Self-hosted mode requires durable application state and uses SQLite plus a private artifact directory on one host.

Prefer portable files for early versions; Braintrust is an optional remote tracking adapter, not the authoritative application database or a prerequisite for self-hosting.

If local persistence becomes useful, SQLite is the preferred option for features such as:

- local run history
- cached metadata
- local baselines
- review history and finding verification
- battles
- rankings

Optional CLI history and required service persistence are different concerns. Do not make core engine correctness depend on either storage implementation, and do not require persistent local state in CI.

Do not store raw provider API keys, Braintrust credentials, or GitHub tokens/private keys in the database, policies, or result artifacts. Use environment variables, CI secret systems, or operator-mounted secret files. Service storage may contain password hashes, token hashes, and authorized secret references, never recoverable account passwords or raw bearer tokens.

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
- self-hosting, authentication, authorization, job lifecycle, or operator responsibilities
- persistence strategy
- code-execution security assumptions
- major technology decisions

Small implementation details do not require architecture documentation updates.

Keep documentation descriptive of the system that exists or has been intentionally planned.

Do not let stale documentation dictate an implementation that no longer makes sense.

Keep `docs/release-checklist.md` aligned with intentional scope changes. Architecture describes the design; the checklist records acceptance criteria and links to validation evidence. Planning a feature does not complete its checkbox.

## Current Development Direction

The revised milestones are planned work:

1. **v0.1.0 — Reproducible local benchmarks:** Go CLI, shared model execution, fake and real providers, local datasets, basic deterministic scoring, bounded concurrency, cancellation, and portable raw results/metadata.
2. **v0.2.0 — Scoring, tracking, and experiment comparison:** richer scorers, versioned suites, Braintrust integration, multi-model comparisons, and provider reliability.
3. **v0.3.0 — Local code review:** fixed snapshots/diffs, review policies, bounded context, model/reasoning selection, and structured validated findings.
4. **v0.4.0 — GitHub Actions and PR reviews:** the CLI in CI, inline comments, summaries, and commit-aware publishing with separate credentials.
5. **v0.5.0 — Review comparisons and quality measurement:** independent reviewers on identical inputs, verified benchmark cases, quality metrics, and curated feedback.
6. **v0.6.0 — Personal self-hosting alpha:** optional server, single-user authentication, SQLite-backed jobs, persistent artifacts, basic web UI/API, and binary/container deployment.
7. **v0.7.0 — Multi-user self-hosting beta:** invitations, workspace roles and isolation, scoped credentials/tokens, fair scheduling, quotas, and shared benchmark/review/comparison workflows.
8. **v0.8.0 — Operational readiness and CI gates:** backup/restore, migrations, retention, observability, restart/load testing, portable baselines, and deterministic regression gates.
9. **v0.9.0 — Release candidate:** freeze public contracts, validate upgrades and deployment instructions, complete a multi-user pilot, and resolve release blockers.
10. **v1.0.0 — Stable local, CI, and self-hosted workflows:** dependable benchmarks, reviews, evidence-based comparisons, and a documented single-host service for the operator and invited users.

The current target is `v0.1.0`. Braintrust integration belongs to `v0.2.0`; code review and GitHub publishing follow later.

The basic service UI and single-host multi-user operation are part of the stable target. GitHub Apps, public signup/billing, distributed execution, advanced dashboards, optional CLI history, battles/rankings, and code-execution sandboxes are not release prerequisites.

The architecture describes the intended direction, not a requirement to build every stage immediately.

When implementing a task, finish the requested behavior rather than stopping after the first plausible implementation. Inspect the result, address failures caused by the change, and leave the affected area in a working state.

## Definition of Done

A change is complete when:

- the requested behavior is implemented
- relevant validation passes
- errors and cancellation are handled appropriately
- meaningful new behavior has appropriate tests
- machine-readable behavior remains stable where applicable
- affected architecture and release-checklist entries reflect the implementation and its validation evidence
