# Evaldrome Architecture

## Status and document ownership

**State: planning only. Active target: `v0.1.0`, reproducible local benchmarks.** No Go module, CLI, engine, or service exists yet. The package map below is the implementation convention, not an inventory of implemented features.

| Document | Owns |
| --- | --- |
| [AGENTS.md](AGENTS.md) | Agent routing, development commands, behavioral guardrails |
| This file | System boundaries, package placement, core flows, error and state rules |
| [Technology specification](.agent_docs/tech_stack.md) | Bootstrap versions, dependency policy, unselected technologies |
| [Testing strategy](.agent_docs/testing.md) | Test design, layers, fixtures, validation standards, development evidence |
| Topic files linked below | Detailed rules for the named future capability |
| [Release checklist](docs/release-checklist.md) | Milestone acceptance and implementation evidence |

Read the relevant topic, not every linked document. Code, manifests, and validation evidence establish what exists; design documents establish intended behavior. Resolve discrepancies explicitly. Keep one canonical owner for each detailed contract and update links when moving it.

---

## Product and system boundaries

Evaldrome is one modular Go application for benchmarking models, reviewing code, and comparing evidence-based results. Build the benchmark engine first. A model finding is provisional until independently verified.

| Boundary | Responsibility | Constraint |
| --- | --- | --- |
| Standalone CLI / CI | Load local inputs, invoke engines, write portable results, return exit status | Requires no account, server, database, or remote tracking |
| Workload engines | Benchmark orchestration; later review and comparison behavior | Own domain rules; remain independent of presentation and infrastructure adapters |
| Shared execution | Provider contracts, capability validation, bounded requests, cancellation, attempt metrics | Uses Evaldrome-owned types; contains no vendor SDK, HTTP-handler, SQLite, or GitHub dependencies |
| Integration adapters | Provider APIs, artifact serialization; later Braintrust and GitHub publishing | Translate external formats at the boundary; do not redefine workload logic |
| Optional service | Authentication, workspaces, durable jobs, quotas, API/UI, service persistence | Calls engines directly; never shells out to the CLI |

The stable target includes one active Linux service instance with SQLite, private artifact files, invited users, and a basic web UI/API. Running the service remains optional. The host operator is trusted with host data. Cloud inference sends selected inputs to the provider; self-hosting does not imply local inference.

PostgreSQL, external queues, multiple replicas, Kubernetes, public signup/billing, advanced dashboards, battles/rankings, optional CLI history, and code-execution sandboxes are outside the initial stable requirement.

## Active implementation scope

| Implement for v0.1.0 | Defer |
| --- | --- |
| One Go CLI; UTF-8 JSONL dataset loading | Generalized YAML suites and multi-model experiments |
| Deterministic fake provider and one real provider | Additional provider integrations |
| Exact-match scoring; coverage and error reporting | LLM judges, battles, rankings, regression gates |
| Bounded requests, deadlines, cancellation, basic attempt metrics | Advanced retry/rate-limit scheduling |
| One versioned JSON result artifact and terminal summary | Braintrust, reviews, GitHub publishing, SQLite, HTTP service |

The first dataset has unique string `id` values and string `input` and `expected` fields. Reject empty datasets, invalid UTF-8/JSON, missing or incorrectly typed required fields, and duplicate IDs before inference. Report record locations. Exact match compares strings without trimming, case folding, or other implicit normalization.

## Directory and package convention

Use **one Go module**, responsibility-based packages under `internal/`, and a thin executable under `cmd/`. Do not create generic `utils`, `common`, `types`, `models`, or mechanical controller/service/repository layers. Add a package only when implementing its responsibility.

All Go/build paths below are planned. Documentation paths already form the repository foundation.

```text
evaldrome/
├── AGENTS.md
├── ARCHITECTURE.md
├── .agent_docs/
│   ├── tech_stack.md
│   ├── testing.md
│   ├── experiments.md
│   ├── reviews.md
│   └── service.md
├── docs/
│   ├── architecture.md          # Compatibility pointer only
│   └── release-checklist.md
├── cmd/evaldrome/main.go        # Process entry point
├── internal/
│   ├── cli/                    # Flags, wiring, rendering, exit mapping
│   ├── dataset/                # JSONL validation and dataset identity
│   ├── benchmark/              # Benchmark orchestration and results
│   ├── scorer/                 # Deterministic scoring
│   ├── execution/              # Model contracts, execution, metrics
│   ├── provider/
│   │   ├── fake/               # Deterministic provider
│   │   └── <vendor>/           # Selected real-provider adapter
│   └── storage/                # Portable artifact schemas and file I/O
├── datasets/                   # Small, public runnable examples
├── .go-version                 # Exact development/CI Go version
├── go.mod
├── go.sum
└── README.md                   # Implemented installation and usage
```

| Future destination | Introduce when |
| --- | --- |
| `internal/braintrust/`, `internal/compare/`, `evals/` | v0.2: tracking, artifact comparison, versioned suites |
| `internal/review/` | v0.3: review orchestration; add `policy/`, `context/`, `findings/` only as concrete responsibilities require |
| `internal/github/` | v0.4: PR metadata and separately authorized publishing |
| `internal/server/` | v0.6: HTTP, identity, workspace authorization, queue, workers |
| `internal/server/store/`, `internal/server/store/migrations/` | v0.6: service-only SQLite state and versioned SQL migrations |
| `internal/server/web/` | v0.6: Go embed owner, templates, and assets together; avoid embedding files outside the package tree |
| `deploy/` | v0.6: Linux service-manager and container deployment examples |

Use the [testing strategy](.agent_docs/testing.md) for test placement, fixtures, and validation. Keep test composition at the appropriate boundary and preserve the production dependency direction below.

## Dependency direction and type ownership

An arrow means a permitted import direction, not a requirement to create an interface.

```text
cmd/evaldrome -> cli -> benchmark -> dataset, scorer, execution
                    -> provider/<vendor> -> execution
                    -> storage -> benchmark, execution

future server -> the same engines and adapters
future review -> execution
```

| Package | Owns | Must not depend on |
| --- | --- | --- |
| `execution` | Normalized request/response, capabilities, attempt metrics, provider interface, executor | Workloads, concrete providers, artifact I/O, CLI, server, vendor SDKs |
| `provider/<vendor>` | SDK/HTTP translation, vendor capabilities, normalized provider errors | Benchmark/review orchestration, service storage |
| `dataset` | Cases, validation, input identity | Providers, execution scheduling, presentation |
| `scorer` | Scoring inputs/results and deterministic rules | Provider adapters, presentation, persistence |
| `benchmark` | Case orchestration, coverage, aggregation, benchmark result types | Concrete providers, CLI/server, storage, Braintrust/GitHub SDKs |
| `storage` | Versioned wire schemas, serialization, artifact file lifecycle | CLI/server state or provider credentials |
| `cli` | Argument validation, constructor wiring, credential selection, output and exit mapping | Business rules duplicated from engines |

Keep review result types in `review`, comparison types in `compare`, and account/workspace/job types in `server` when introduced. Standalone runs require no synthetic workspace or account. Share metadata deliberately; do not force benchmark and review results into one universal result type.

Define small interfaces in the consuming package when substitution is needed. Provider adapters implement the interface owned by `execution`. Inject dependencies through constructors. Do not introduce an interface or framework for a hypothetical future implementation.

## Core data flows

### Benchmark: v0.1.0

1. CLI parses options and freezes the effective run configuration.
2. Dataset loader validates the entire bounded input before any provider request. Hash the same immutable input bytes that will be evaluated with SHA-256.
3. CLI selects and configures a provider; credentials stay inside the execution/provider boundary, outside the run specification and prompts.
4. Benchmark runner dispatches cases through shared execution with bounded workers and in-flight requests. Execution validates options and applies deadlines.
5. Scorer evaluates successful responses. A mismatch is a scored result; provider/scorer failures and canceled/unstarted cases remain separately identifiable.
6. Runner returns accumulated raw results, metrics, coverage, and operational errors. Preserve completed work when cancellation interrupts a run.
7. CLI writes the versioned artifact through `storage`, renders the summary, and maps the outcome to the documented exit contract. An artifact-write failure cannot produce a successful command exit.

### Later workloads

| Flow | Required sequence | Detailed contract |
| --- | --- | --- |
| Experiment comparison | Validate compatibility -> read immutable artifacts -> compute derived dimensions; no implicit inference | [Experiments](.agent_docs/experiments.md) |
| Review | Freeze snapshot/diff/policy -> gather bounded read-only context -> execute -> validate/filter/deduplicate -> write artifact | [Reviews](.agent_docs/reviews.md) |
| Publication | Validate trusted artifact provenance and current PR head -> explicitly publish with separate credentials | [Reviews](.agent_docs/reviews.md#github-publishing) |
| Hosted job | Authenticate/authorize -> freeze input references -> transactionally enqueue -> acknowledge -> worker executes engine -> finalize artifact and state | [Service](.agent_docs/service.md) |

## Errors, cancellation, and logging

| Condition | Required behavior |
| --- | --- |
| Invalid input/configuration or unsupported model option | Reject before inference where detectable; return an actionable, safe error |
| Exact-match mismatch | Record a valid score; do not classify it as an execution error |
| Provider/scorer failure | Preserve case/attempt identity and available output; expose missing coverage |
| User cancellation | Stop scheduling, cancel active requests/waits, join workers, retain completed output |
| Request deadline | Record a timeout failure; do not mislabel it as user cancellation |
| Artifact failure | Return an operational failure even if inference completed |
| Optional tracking/publishing failure | Report separately; preserve workload results |

Use explicit errors, wrap causes with `%w`, and classify with `errors.Is`/`errors.As`. Use typed errors or sentinels only when callers need to branch. Do not use panic for expected input/provider failures. Only the executable boundary calls `os.Exit`; perform cleanup before returning its exit code.

Pass `context.Context` first through blocking/cancellable calls. Do not replace caller contexts with `context.Background()` inside engines. The CLI owns the signal-derived run context; service workers later own durable job contexts independent of HTTP connections. Any cancellation-finalization work needs a separate bounded cleanup lifetime.

Each goroutine has an owner, cancellation path, and join point. Bound workers as well as network concurrency. Avoid a goroutine per dataset case waiting on a semaphore. Inject shared execution limits; do not use mutable package globals.

For v0.1, make request attempts explicit and disable hidden SDK retries. Add classified, bounded retries and cancellable backoff at v0.2. Never blindly retry authentication/invalid requests or replay ambiguous in-flight work. Record every attempt and available usage.

Return errors from libraries; log once at the handling boundary using `log/slog`. Include safe run/case/request identifiers. Exclude credentials and raw private prompts/responses from diagnostic logs.

## State, artifacts, and outcome semantics

| State | Rule |
| --- | --- |
| Configuration and inputs | Immutable for the run; record effective settings and input identity |
| In-memory work | Run-scoped state with explicit synchronization/ownership; no mutable globals |
| Result ordering | Preserve dataset order in benchmark output regardless of completion order |
| Raw artifacts | Immutable execution evidence; serialize independently of vendor types |
| Derived scores/verification | Version separately and reference original run/finding IDs; preserve revisions |
| Usage and cost | Unknown is nullable/unavailable, never zero; label estimates and pricing assumptions |
| Service state, later | SQLite owns authorization/jobs; private files own large inputs/results; Braintrust owns neither |

| Run status | Meaning |
| --- | --- |
| `complete` | All required work completed successfully; a scored mismatch is still completed work |
| `partial` | Usable completed output exists, but required work is missing or failed |
| `failed` | No usable completed result exists |
| `canceled` | Explicit cancellation occurred; completed output may still exist |

Keep run status, quality-gate outcome, verification state, and publication status separate. Report total/completed/scored/failed/canceled/unstarted coverage as applicable. Compute scores over scored cases and report their denominator; zero scored cases yields unavailable, not a perfect or zero score.

Artifacts carry schema version, run ID/kind, application version, dataset identity/hash, provider/model, requested/effective options, scorer identity/version, execution limits, timestamps, raw responses, scores, errors, coverage, latency, and nullable usage/cost. Record Git identity when available and explicit unknowns otherwise. Add suite/review provenance when those workloads exist.

Write artifacts through same-filesystem staging and atomic finalization; do not silently overwrite existing results. Handle flush/close/finalization failures. On graceful cancellation, attempt to save retained results with bounded cleanup; abrupt termination recovery is a later service concern. Historical artifacts remain readable without SQLite or remote tracking.

## Contract decisions required before the first CLI slice ships

The following are **unresolved implementation contracts**, not permission to invent undocumented defaults:

- Dataset byte/record limits, blank-line/unknown-field policy, and hash representation.
- Artifact field names, schema version, run-ID format, error codes, and compatibility fixtures.
- CLI flags/defaults, output location, overwrite opt-in, stdout/stderr rules, and exact exit/signal mapping.
- First real provider/model/API endpoint, supported options, timeout/concurrency defaults, and attempt reporting.

Finalize these with the first vertical slice, record them in focused context documents, and add deterministic fixtures. Do not publish placeholder commands or numeric exit-code proposals as implemented contracts.

## Progressive milestone routing

| Milestone | Design context |
| --- | --- |
| v0.1: local benchmark loop | This file and [technology specification](.agent_docs/tech_stack.md) |
| v0.2: scorers, suites, Braintrust, providers, comparisons | [Experiments](.agent_docs/experiments.md) |
| v0.3-v0.5: local review, GitHub, verified review comparisons | [Reviews](.agent_docs/reviews.md) |
| v0.6-v0.7: personal and multi-user service | [Service](.agent_docs/service.md) |
| v0.8: operational readiness and CI gates | [Service](.agent_docs/service.md), [experiments](.agent_docs/experiments.md) |
| v0.9-v1.0: compatibility, pilot, stable release | [Release checklist](docs/release-checklist.md) |

Keep milestone scope and evidence in the checklist. Before v1, document intentional breaking changes. From v1, preserve documented CLI/API/configuration/artifact contracts in compatible releases; breaking contracts require a major version and migration path. Version artifact schemas independently, retain raw history, and publish supported upgrade/deprecation and backup-based rollback policies.
