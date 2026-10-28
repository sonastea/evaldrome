# Evaldrome Architecture

## Overview

Evaldrome is a Go-first, CLI-first, self-hostable AI evaluation and code-review platform.

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

Benchmark and review runs share model execution, provider configuration, metrics, portable result storage, and optional Braintrust tracking. They retain distinct orchestration and result types.

Evaldrome owns benchmark execution, review context and policies, findings, scoring, comparisons, GitHub publishing, presentation, and regression decisions. The optional self-hosted service adds user access, workspaces, durable jobs, and run history. Braintrust provides optional remote experiment-oriented tracing and analysis behind an internal adapter.

Evaldrome supports two planned deployment modes: standalone execution on a developer machine or CI runner, and a single-host service operated for one person or multiple invited users. Standalone commands require no server, account, or database. The service uses the same engines and adds SQLite-backed application state; neither mode requires an Evaldrome-operated cloud service.

## Concrete Product Contract

The first audience is a developer choosing models for their own repositories. The next audience is an operator offering that same workflow to invited developers or small teams on infrastructure they control.

| User need | Input | Useful output |
| --- | --- | --- |
| Choose a model for a task | Versioned dataset, scorer, and model configurations | Comparable scores, raw responses, failures, latency, and usage/cost |
| Review a change | Fixed repository snapshot/diff, policy, and reviewer configuration | Located findings with evidence, coverage, and explicit execution status |
| Check reviewer quality | Identical review inputs and independently verified defects/findings | Verified detections, misses, rejected/unresolved findings, and cost/latency tradeoffs |
| Share a self-hosted instance | Invited users, workspace membership, authorized inputs and provider configurations | Private run submission, progress, history, comparisons, and downloadable artifacts |

The core workflow is: register inputs, validate configuration, execute bounded work, retain raw evidence, inspect results, and compare compatible runs. A review can then be explicitly published to GitHub or independently verified. Model-reported findings remain provisional until verified.

For `v1.0.0`, a self-hosted user must be able to sign in, select an authorized workspace/provider, upload a dataset or select a registered repository, submit a benchmark or review, cancel it, inspect results, and compare existing runs. Workspace admins can invite members and manage workspace access; operators provision credentials and instance limits. The basic web UI covers these tasks rather than requiring every user to have shell access to the host.

Release success is demonstrated by the end-to-end scenarios and evidence in [the release checklist](release-checklist.md), not by a feature name appearing in this document.

## Implementation Status

The repository currently contains planning documents and agent guidance; the Go CLI, execution engines, and self-hosted service have not been implemented. Commands, configuration schemas, interfaces, and workflows below describe the intended direction, not shipped behavior.

Build the benchmark engine first, then make code review its first major real-world application, then expose those proven workflows through the optional service. The roadmap is incremental, not a requirement to build the whole platform at once. [Release Milestones](#release-milestones) defines scope; [the release checklist](release-checklist.md) tracks completion.

For deployment decisions, start with [Product Boundary](#product-boundary), [Self-Hosted Service](#self-hosted-service), and [Persistence and Recovery](#persistence-and-recovery).

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

The first vertical slice is deliberately narrow: a UTF-8 JSONL dataset whose records have a unique string `id` and string `input` and `expected` fields; one selected model; an exact-match scorer; and one JSON artifact. Reject empty/malformed datasets and duplicate IDs before inference. Exact match compares the returned text to `expected` without implicit trimming or case folding; any later normalization is explicit, versioned scorer configuration.

Illustrative dataset record:

```json
{"id":"capital-france","input":"Return only the capital of France.","expected":"Paris"}
```

Record failed or canceled cases separately from scored responses, and show completed/scored coverage alongside the score. Preserve available responses and error metadata on partial runs. Finalize the artifact schema and exit-code contract with this slice rather than postponing them until hosting.

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

Standalone mode runs as:

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
         +--> Braintrust when enabled
         +--> result artifacts
         +--> GitHub review publishing when enabled
         |
         v
      exit status
```

Self-hosted mode runs as:

```text
browser / authenticated API client
               |
               v
TLS endpoint --> evaldrome serve
                 authentication + workspaces
                 durable queue + bounded workers
                         |
                         +--> same benchmark / review / comparison engines
                         +--> SQLite + private artifact storage
                         +--> authorized model providers
                         +--> optional Braintrust / GitHub integrations
```

### Deployment commitments

| Mode | Intended use | Required state/services |
| --- | --- | --- |
| Standalone CLI / CI | Individual local use and automation | Input/configuration files and portable output artifacts; provider access for real inference |
| Personal self-hosted service | One operator using a browser/API on their own host | One Go server, local SQLite database, artifact volume, authentication, and provider access |
| Shared self-hosted service | An operator serving invited users in private or shared workspaces | The same deployment plus membership, resource authorization, scoped credentials, quotas, and audit records |

Self-hosting means the operator controls Evaldrome, its application data, and deployment. It does not automatically make cloud model calls private to that host. Selected prompts and context go to the configured provider; Braintrust payload export is separately opt-in. A tested local-provider configuration with remote tracking disabled must support operation without a mandatory external application service. Model weights, inference hardware, and provider terms remain separate from running Evaldrome.

The repository's [Apache-2.0 license](../LICENSE) permits operating and modifying Evaldrome for personal use or serving other users, subject to its terms. It does not grant rights to third-party datasets, repository content, or model services.

The initial stable service targets one active application instance on a Linux host with local persistent storage. SQLite and the local artifact store do not imply multi-node support. Users are authenticated and isolated at the application/workspace boundary; the host operator remains trusted and can access host data. Separate deployments are appropriate when users require isolation from one another's operators.

The stable target does not include:

- an Evaldrome-operated hosted SaaS, public self-signup, subscriptions, or billing
- a model proxy
- an inference reseller
- a highly available or distributed service requiring PostgreSQL or an external queue
- a Kubernetes control plane
- a replacement for Braintrust

Its value is in making model evaluation and code review reproducible, comparable, automatable, and measurable. The CLI, service, GitHub Actions, and a later optional GitHub App invoke the same underlying workflows.

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

CLI rendering, HTTP handlers, service scheduling/storage, GitHub APIs, tracking adapters, and artifact serialization should sit around the engines rather than defining them. Keep read-only review generation separate from write-enabled publishing. The service calls Go engines directly through bounded workers; it must not shell out to the CLI or duplicate workload logic.

---

# Self-Hosted Service

Self-hosting is planned for `v0.6.0` onward, with multi-user operation in `v0.7.0` and operational acceptance before `v1.0.0`. It is an optional way to run the product, but a required capability of the stable release.

## Runtime and Deployment

Start with one Go application: `evaldrome serve`, HTTP handlers, embedded web assets, a SQLite-backed job queue, and bounded in-process workers. Prefer server-rendered HTML with small progressive enhancements over a separate frontend deployment. Use standard Go HTTP facilities and small, maintained dependencies for authentication and SQLite.

```text
Browser / API client
         |
         v
TLS reverse proxy
         |
         v
HTTP adapter --> authentication / workspace authorization
         |
         +--> SQLite: accounts, membership, input metadata, jobs, audit
         +--> Private files: datasets, snapshots, raw results, exports
         |
         v
Durable queue --> bounded workers --> existing workload engines
                                         |
                                         +--> authorized provider configuration
                                         +--> optional tracking adapter
                                         +--> separate publishing adapter
```

Support a Linux binary managed by a service manager and a non-root OCI container with a documented Compose example. Use one persistent data directory and separately mounted secrets. The deployment needs neither PostgreSQL, Redis, a managed identity service, nor Kubernetes. Bind to loopback by default; remote use requires an explicit bind configuration and HTTPS, normally terminated by a reverse proxy. Only configured proxies may supply trusted forwarding headers.

Support one active server per data directory, with SQLite on a local filesystem. Reject a second active process using that directory. Network-mounted SQLite databases, multiple replicas, and distributed workers are outside the supported topology. Document supported CPU architectures, resource requirements, maximum tested workload, volume permissions, and network destinations with the actual release measurements.

## Accounts, Workspaces, and Authorization

Use local accounts and invite-only enrollment so the service can be fully operator-hosted. Bootstrap the first operator through a local administrative command or single-use local setup mechanism, never a public first-user-wins endpoint. Store passwords with a maintained Argon2id implementation and unique salts; use opaque server-side sessions, secure/HttpOnly/SameSite cookies, CSRF protection for cookie-authenticated mutations, login throttling, and session expiry/revocation.

Invitations are single-use, expiring tokens bound to an intended identity, workspace, and role. The operator can deliver an invitation link out of band, so email infrastructure is not required. Provide a documented operator-assisted account recovery flow; password resets revoke prior sessions. Keep invitation/session/API token values out of logs and store only their hashes. API tokens are revocable, expiring, workspace-scoped, and permission-limited; checks also enforce the issuing user's current membership and role.

Every dataset, repository registration, policy, provider grant, run, artifact, comparison, verification record, and associated audit event belongs to a workspace. Instance-level account/bootstrap audit records are separate and operator-only. A person can have a private workspace and belong to shared workspaces. Initial roles are:

| Role | Permissions |
| --- | --- |
| Viewer | Read permitted workspace inputs, runs, findings, comparisons, and exports |
| Member | Viewer permissions plus submit/cancel own runs and add inputs, comparisons, and verification records |
| Workspace admin | Member permissions plus manage workspace membership, cancel any workspace run, and manage workspace retention within operator limits |
| Instance operator | Bootstrap accounts, create workspaces, provision integrations/secrets, set instance ceilings, and administer the host |

Publishing additionally requires an explicit workspace publishing grant from the operator and an authorized role/token; membership alone must not grant GitHub write access. Workspace admins cannot raise operator ceilings or grant access to unassigned credentials. Removing membership immediately revokes resource access and prevents new work under that membership; cancel its queued work and request cancellation of active work.

Authorize every object lookup and download, not only list pages. Derive identity from authentication and verify membership rather than trusting a supplied user/workspace ID. Scope database queries and storage keys by workspace; apply the same checks to logs, pagination, comparison inputs, and verification history. Cross-workspace object references must not reveal whether the referenced object exists. Avoid cross-workspace content caches in the initial service. The basic UI must provide run submission, status/cancellation, findings, comparisons, exports, and membership administration.

## Inputs, Credentials, and Outbound Access

Local CLI paths remain valid locally. A remote request instead names an authorized, immutable uploaded dataset/policy or a registered repository and requested revision. Resolve mutable repository refs into pinned commit IDs before admitting a review job; workers materialize only those commits using the same fixed snapshot/diff contract as local reviews. Authorize all referenced resources and record the accepted configuration/input versions; a later edit or branch movement must not mutate a queued job's meaning.

Stage uploads with byte/count limits and publish them atomically under server-generated IDs. Reject path traversal and unsafe archive/symlink entries. Store inputs and artifacts outside the web root and expose downloads only through authorized handlers, with safe attachment/content-type headers and content sniffing disabled. Render repository/model text as escaped content, never trusted HTML. Use per-run work directories and bounded read-only Git operations with hooks, external helpers, and automatic submodule execution disabled; subprocesses receive a minimal environment without provider or publishing secrets. Uploaded content never grants command execution or access to host paths.

Operators register permitted repository origins and provider/tracking endpoints. Users select workspace-authorized entries, not arbitrary fetch URLs or environment-variable names. Enforce outbound destination rules during connection and redirects, including protection against DNS rebinding and access to instance metadata or unrelated internal services. Local/private inference endpoints are allowed only through explicit operator configuration. Repository-read credentials are narrowly scoped and separate from provider and publishing secrets.

For the first stable service, credentials are operator-provisioned environment variables or read-only secret files, mapped to named configurations with explicit workspace grants. This supports personal keys, separately provisioned user/team BYOK keys, or an explicitly shared operator key with limits. It does not provide self-service secret upload/storage in the UI. The operator is trusted with provisioned keys; users must never implicitly inherit the operator's default credentials. SQLite contains only authorized references, not the raw secrets, and exports contain no secret values or host secret paths.

Resolve credentials only when executing an authorized operation. Recheck membership, provider grants, repository access, and limits at dispatch and before additional provider requests; fail explicitly if authorization was revoked. Model inputs and repository workers must not receive publishing credentials. Tracking destinations and payload policies are workspace-scoped, with raw-content export disabled unless explicitly enabled.

## Job Lifecycle and Failure Semantics

Persist the accepted immutable run specification, actor/workspace identity, and queued job transactionally before acknowledging submission. Workers invoke the engines with job-scoped contexts independent of the submitting HTTP connection; closing a browser does not cancel a run.

```text
queued --> running --> complete | partial | failed | canceled
   |
   +-----------------> canceled
```

`complete` means required work finished, not that a quality gate passed or all findings were confirmed. `partial` retains usable output but records missing required work. `failed` has no usable completed result; `canceled` records user/operator cancellation even if some output is retained. Cancellation requests are durable and propagated to queued and active work. Quality gate outcomes and publishing status are separate fields.

Claim jobs transactionally and checkpoint completed units and attempt metadata. Publish final artifact files atomically before marking their job terminal; detect missing/corrupt files and reconcile staged/orphaned files after a crash. Queued jobs survive restart. Running jobs interrupted by a crash become partial or failed with an explicit interruption reason unless a fully finalized result can be recovered. They do not silently become successful or restart inference. A user-requested rerun has a new run ID linked to the original.

Submission idempotency prevents duplicate jobs; it cannot make a provider request exactly-once. A crash after dispatch may leave an unknown response or charge. Record ambiguous attempts and unknown usage, and never automatically replay them without a provider guarantee or explicit user-requested rerun. Classified retries for known retryable responses remain bounded and recorded. Publishing has its own idempotency and provenance checks.

On graceful shutdown, stop accepting/claiming work, allow a bounded drain, then cancel remaining engine contexts and persist interruption status. On storage failure or exhausted disk, stop admitting work and expose degraded readiness rather than returning success for an unpersisted job.

## HTTP API Contract

Use a versioned REST API under `/api/v1`, shared by the basic UI and API clients where useful. Document request/response schemas in OpenAPI as endpoints are implemented. The proposed resource surface is:

| Endpoint | Behavior |
| --- | --- |
| `POST /api/v1/workspaces/{workspace}/datasets` | Upload and validate an immutable dataset; return `201` with its ID |
| `GET /api/v1/workspaces/{workspace}/runs` | List authorized runs with bounded cursor pagination |
| `POST /api/v1/workspaces/{workspace}/runs` | Validate references/options and durably enqueue a benchmark/review; return `202` with run ID and `Location` |
| `GET /api/v1/workspaces/{workspace}/runs/{run}` | Read status, coverage, errors, gate outcome, and artifact references |
| `POST /api/v1/workspaces/{workspace}/runs/{run}/cancellation` | Idempotently request cancellation; report terminal state if already finished |
| `GET /api/v1/workspaces/{workspace}/runs/{run}/artifacts/{artifact}` | Download an authorized artifact |
| `POST /api/v1/workspaces/{workspace}/comparisons` | Compare compatible existing workspace runs without repeating inference |

Membership, repository/policy registration, verification, and explicit publishing endpoints follow the same authorization/versioning rules. Polling is sufficient initially; streaming and a remote-CLI client are not required for `v1.0.0`.

Require an `Idempotency-Key` for run submission, scoped to actor/workspace/operation and a canonical request hash. An identical retry returns the same run; reuse with a different request returns `409`. Persist keys at least as long as their run metadata and document expiry. Apply pagination to all collections and bound bodies, query limits, and downloads. Use stable machine-readable error codes, a safe message, and a request ID; distinguish unauthenticated (`401`), forbidden (`403` or non-disclosing `404`), invalid configuration (`422`), excessive payload (`413`), quota/rate limits (`429`), and temporary unavailability (`503`). Do not expose secrets or raw internal errors.

## Resource and Cost Controls

Enforce instance-wide and per-workspace limits on queued/running jobs, model requests, attempts, context size, tokens, elapsed time, uploads, and retained storage. The scheduler must give eligible workspaces a turn instead of letting one workspace monopolize the queue. All workers share the instance-wide provider concurrency ceiling; a per-job limiter alone is insufficient.

Reserve capacity transactionally at admission/dispatch and reconcile usage across concurrent jobs. If an estimated-cost guard is required, reject configurations without usable pricing/output bounds rather than treating unknown cost as zero. Report all known attempt usage and uncertainty. Estimated spend limits are admission controls, not guaranteed billing caps; provider-enforced hard limits are needed for that guarantee. Never charge another workspace's configured key through a fallback.

## Operations and Support Boundary

Provide bounded, secret-redacted structured logs, authenticated operational metrics, minimal liveness/readiness endpoints, and audit records for membership changes, credential grants, submissions, cancellations, verification revisions, deletions, and publishing. Audit metadata records actor, workspace, action, target, timestamp, and outcome without duplicating prompts or keys. Display tracking/publishing failures separately from completed inference.

Before stable release, document and exercise installation, HTTPS, bootstrap/invitations, provider setup, data export/deletion, storage limits, backup/restore, upgrades, and recovery. Use a deterministic load scenario of at least 100 mixed benchmark/review jobs from 10 clients across three workspaces, with four global and two per-workspace in-flight model requests, injected rate limits/timeouts, cancellation, and a server restart. Record CPU/RAM, dataset/context sizes, queue latency, peak memory/disk, and outcomes; verify bounds and workspace progress. This is a validation workload, not an unmeasured production capacity claim.

Run a multi-user pilot before `v1.0.0` using at least two non-operator accounts, private workspaces, and a shared workspace. A clean install, isolated user access, successful restore onto a fresh host, and upgrade from the release candidate are release gates. Detailed evidence requirements live in [the release checklist](release-checklist.md).

---

# Shared Execution and Workload Engines

The shared model-execution layer should not depend on:

- a specific CLI framework
- HTTP handlers, sessions, or a service job queue
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

These engines serve local commands, service workers, tests, GitHub Actions, and other CI systems without duplicating workload logic. Authentication, workspace authorization, scheduling, and durable queue state belong to the service adapter, not provider or scorer interfaces.

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

Service-only concepts are **Account**, **Workspace**, **Membership**, **Credential grant**, and **Job**. A job schedules a workload run and links its actor, immutable inputs, attempts, status, and artifacts. These concepts provide access control and durable orchestration around the engines; a standalone run does not require a synthetic account or workspace.

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
evaldrome serve
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

## `serve`

Start the optional service using operator-controlled configuration, from `v0.6.0` onward:

```bash
evaldrome serve --config /etc/evaldrome/server.yaml
```

This is planned syntax, not an available command. Server configuration selects the bind address, persistent data directory, secret references, permitted endpoints, and resource limits. It must not alter standalone command behavior. Administrative bootstrap, backup, and migration commands should be defined with the service implementation and documented before release.

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
	InputTokens  *int
	OutputTokens *int
	Latency      time.Duration
}
```

Nil usage fields represent unknown usage, not zero. Estimated cost may be calculated separately when provider APIs do not return reliable cost information directly.

Both workloads use the same provider configuration and credential resolution. BYOK credentials should come from environment variables, CI secret systems, or operator-mounted secret files, not committed policies, result artifacts, or prompts. Service workers resolve only workspace-authorized configurations; standalone credential defaults must not leak into service requests.

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

By the stable release, document and test at least one cloud provider and one local/self-hosted inference endpoint, with their actual supported models/options and endpoint configuration. A configurable base URL alone is not proof of compatibility with every API-compatible server. Local inference and disabled remote tracking must be a usable deployment path; tests should verify there is no mandatory cloud tracking or identity call. Model downloads and inference hardware are separately provisioned by the operator.

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

Only introduce provider/model-specific scheduling when actual provider behavior requires it. Shared hosting additionally requires instance-wide and per-workspace limits and fair scheduling, as described in [Self-Hosted Service](#self-hosted-service).

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

Braintrust is Evaldrome's planned primary remote experiment and model-observability integration, introduced in `v0.2.0`. It is opt-in and never required to authenticate, run workloads, inspect history, or compare local/service artifacts. Prefer the Braintrust Go SDK where practical and confirm the needed SDK capabilities during implementation.

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

The service stores the same workload artifact schemas, with workspace ownership and authorization held in its own trusted metadata. Downloading or importing an artifact must not grant access to its claimed workspace, secrets, or publishing permissions. Exports remain readable by standalone tools without the service database. Service verification revisions reference the original artifact/finding IDs and authenticated actor while preserving prior assessments.

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

# Persistence and Recovery

## Standalone and CI

Standalone benchmark, review, and comparison commands do not require a database. Configuration files, datasets, snapshots, and portable artifacts remain sufficient. Braintrust is an optional remote view of results, not their required source of truth.

If local history becomes useful, an optional SQLite index at a location such as `.evaldrome/evaldrome.db` can hold run history, cached metadata, baselines, or verification references. That index must not become a hidden prerequisite for exporting or comparing artifacts. CI correctness must not depend on state surviving between runs; use portable baselines and CI-native artifacts.

## Required Service State

The single-host service uses SQLite for accounts, password/token hashes, workspaces/membership, authorized secret references, input versions, jobs/attempts, idempotency records, artifact metadata, verification history, and audit records. Store large immutable inputs and results as private files, scoped by workspace and server-generated IDs. Braintrust does not replace this state.

An illustrative data directory is:

```text
/var/lib/evaldrome/
  evaldrome.db
  workspaces/<workspace-id>/inputs/
  workspaces/<workspace-id>/artifacts/
  workspaces/<workspace-id>/staging/
```

Use foreign keys, transactions, schema migrations, bounded database access, and an explicitly configured SQLite journal/busy-timeout policy. WAL is appropriate for the single-host design; include its checkpoint/backup behavior in operations tests. Do not hold write transactions open during model calls. Enforce relationships between workspace IDs and all referenced objects, including at storage boundaries.

Artifact writes use staging and atomic finalization with hashes; database references become visible only after the file is durable. Recovery must detect missing, incomplete, and orphaned files without inventing successful results. Input/artifact readers verify recorded integrity before use where corruption could affect execution or publishing.

Raw provider keys, Braintrust keys, repository-read credentials, GitHub publishing tokens/private keys, account passwords, and bearer tokens must not be stored in the database. Passwords and service-issued authentication tokens are stored as appropriate hashes; external credentials remain in environment variables, CI secret systems, or operator-controlled secret files outside the data directory. Back up required secret material separately under operator control; never bundle it into ordinary user exports.

## Backups, Upgrades, and Retention

The initial supported backup procedure may use a maintenance window: stop admission, drain or explicitly interrupt jobs, stop the server cleanly, then back up the database and referenced input/artifact files as one consistent set. A live backup is supported only if it coordinates SQLite's backup mechanism with file consistency; copying a live `.db` alone is insufficient. Document what is included, secret dependencies, permissions, and backup encryption under operator control.

Restore onto a fresh host and verify accounts/access rules, queued/terminal job states, artifact hashes, findings, and comparisons. Disable dispatch and outbound integrations during restoration until the operator explicitly re-enables them, so a restore cannot unexpectedly repeat billable work or publish comments. Record the achieved restore time and the backup's recovery point rather than promising untested availability.

Version service migrations independently of artifact schemas. Before upgrading, create a consistent backup; reject unsupported schema versions and test the documented upgrade path. Initial rollback may restore the entire pre-upgrade backup, including matching artifact state, rather than attempt unsafe downgrade migrations. State explicitly that this loses writes after that backup; never silently rewrite historical artifacts to satisfy a new schema.

Set operator storage ceilings and configurable retention for inputs, runs, logs, and audit history, with workspace admins able to shorten retention within those limits. Cleanup must respect active jobs and references used by retained results, comparisons, baselines, or verification records. Explicit deletion removes the selected data and marks dependent views unavailable without silently altering historical scores. Explain that deletion from the active instance does not erase separately retained backups or opted-in remote tracking copies; document their retention and deletion procedures too.

SQLite is durable application state for a single host, not distributed coordination. A future multi-node design would need an intentional migration of queue, storage, and consistency boundaries, not simply a second container pointed at the same directory.

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
│   ├── server/          # HTTP, identity, workspaces, jobs, SQLite (v0.6+)
│   └── storage/         # Portable run artifacts and verification records
│
├── web/                 # Embedded service UI assets (v0.6+)
├── deploy/              # Service-manager and Compose examples (v0.6+)
├── configs/
├── datasets/
├── evals/
├── docs/
│   ├── architecture.md
│   └── release-checklist.md
│
├── AGENTS.md
├── go.mod
└── README.md
```

This is directional rather than mandatory.

Prefer packages based on real responsibilities rather than creating directories merely to match this document.

Start with the packages needed for `v0.1.0`. Review subpackages should emerge with the review implementation, and `storage` does not imply a database or a generic persistence framework. Add service packages/assets when hosting is implemented; keep its SQLite/session/job dependencies out of the core engines.

## Repository Formatting

Evaldrome's own Go source uses `gofumpt` as its default formatter. Its output is `gofmt`-compatible, with additional formatting rules for consistency.

When development tooling is introduced, pin the `gofumpt` version and use the same version for local formatting and automated checks. It is a development-tool dependency, not a requirement for users running the Evaldrome CLI.

Model-generated Go code evaluated by Evaldrome follows the formatter policy in [Task and Code Execution](#task-and-code-execution).

---

# Release Milestones

Evaldrome uses semantic versioning.

Before `v1.0.0`, each minor release represents a meaningful product milestone. This revised roadmap is planned work, not a description of implemented features. It includes standalone and self-hosted delivery. The actionable acceptance gates are maintained in [the release checklist](release-checklist.md); all implementation milestones are currently incomplete.

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

Required scope:

- richer deterministic scorers
- versioned suite configuration and validation
- Braintrust experiment integration through an internal adapter
- multi-model experiments and comparison of existing result artifacts
- quality, latency, token usage, and estimated-cost reporting
- a second provider, with documented cloud and local/self-hosted execution paths and capability-aware configuration
- provider error normalization, classified retries, and rate-limit handling

Supplementary judges can follow when deterministic scoring is insufficient; they are not an acceptance requirement for this milestone.

Goal:

> Compare reproducible experiments locally and in Braintrust without coupling the engine to remote tracking.

---

## v0.3.0 — Local Code Review

Introduce code review as a workload using the same model execution and telemetry.

Required scope:

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

Required scope:

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

Required scope:

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

## v0.6.0 — Personal Self-Hosting Alpha

Expose the proven benchmark loop to one authenticated operator through `evaldrome serve`.

Required scope:

- single-host Go service with local account bootstrap and one private workspace
- versioned API and basic UI for dataset upload, benchmark submission, progress, cancellation, history, and artifact download
- SQLite-backed durable queue, attempt metadata, idempotency, and explicit restart/interruption behavior
- operator-configured provider credentials and bounded workers using the existing engines
- Linux binary and non-root container packaging with persistent storage and HTTPS deployment instructions
- initial consistent backup/restore procedure; useful operation with Braintrust disabled

Goal:

> Install Evaldrome on a personal host, run a benchmark through a browser, restart the service, and retain inspectable results.

---

## v0.7.0 — Multi-User Self-Hosting Beta

Make the instance usable by invited users and small teams, with private or shared workspaces.

Required scope:

- invitations, account recovery, workspace admin/member/viewer roles, and scoped revocable API tokens
- authorization for every resource, comparison, log, and download
- explicit workspace credential/repository grants without implicit access to operator secrets
- instance/workspace quotas, fair scheduling, and audit records
- benchmark, review, artifact comparison, verification, and explicitly authorized GitHub publication through the service
- private-repository read access and frozen snapshots with the same local review contract
- adversarial two-workspace fixtures for data, credential, path, and outbound-network boundaries

Goal:

> The operator and other users can complete the core workflows independently or collaborate in a shared workspace without accessing another workspace's private resources.

---

## v0.8.0 — Operational Readiness and CI Gates

Make the service recoverable and the local/CI workflows suitable for automated decisions.

Required scope:

- tested backup/restore onto a fresh host, migrations, upgrade/rollback procedure, and retention/deletion
- bounded logs, operational metrics, audit visibility, health/readiness, and graceful shutdown
- deterministic restart, storage-failure, quota, and concurrent-load scenarios
- portable baselines, deterministic CI regression thresholds, and documented exit behavior
- measured resource requirements and supported deployment/provider matrix

Goal:

> Recover and upgrade a shared instance predictably, and enforce reproducible quality gates without requiring the service in CI.

---

## v0.9.0 — Release Candidate

Freeze the intended `v1` public contracts and validate the complete operator/user experience.

Required scope:

- CLI flags/exit codes, API schemas, configuration, policies, artifacts, and compatibility rules documented
- clean installation and deployment instructions exercised by someone other than their author
- at least seven consecutive days of a multi-user pilot, including private/shared workspaces and representative benchmark/review/comparison jobs
- fake-provider load/failure tests, opt-in real-provider smoke evidence, and offline/local-provider path checked
- successful backup/restore and release-candidate-to-stable migration rehearsal
- all release-blocking correctness, isolation, data-loss, and recovery issues resolved; remaining limitations documented

Goal:

> Demonstrate the stable workflows on an actual self-hosted instance and retain reproducible release evidence.

---

## v1.0.0 — Stable Local, CI, and Self-Hosted Workflows

`v1.0.0` represents the first stable, documented release intended for external use.

At this point Evaldrome should have:

- documented installation
- stable CLI and machine-readable behavior
- stable `/api/v1` behavior and a usable basic web UI
- stable suite, review-policy, and artifact formats
- reliable multi-provider execution
- documented cloud and local/self-hosted inference configurations
- reproducible evaluation suites
- deterministic scorers
- optional Braintrust integration, with standalone and service operation independent of it
- configurable local reviews and GitHub PR publishing
- validated model/reasoning configuration and BYOK credentials
- benchmark and review comparison tooling
- review-quality scoring grounded in independent verification
- documented CI behavior and execution/publishing boundaries
- preserved raw results and versioned derived scores
- single-host self-hosting for the operator and invited users, including workspace isolation and credential grants
- durable jobs, bounded resource use, migrations, tested backup/restore, and retention controls
- reasonable compatibility guarantees

All stable-release gates in [the release checklist](release-checklist.md) must pass with evidence. GitHub Apps, public signup/billing, distributed execution, advanced dashboards, pairwise rankings, and code-execution sandboxes are not prerequisites. A single-host server is included; running it remains optional for CLI/CI users. PostgreSQL and Kubernetes are not required.

---

# Optional Later Directions

Future features, before or after `v1.0.0`, should be driven by demonstrated user needs and the maturity of the local workflows.

Possible directions include:

- optional SQLite-backed local history
- GitHub App event delivery and installation management
- distributed review/benchmark execution
- richer feedback analytics
- advanced dashboards and feedback visualization beyond the basic service UI
- public registration, hosted-service billing, and external identity federation
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
v0.6.0   personal self-hosting alpha
v0.7.0   multi-user self-hosting beta
v0.8.0   operational readiness and CI gates
v0.9.0   release candidate
v1.0.0   stable local, CI, and self-hosted workflows
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

From `v1.0.0`, compatible patch/minor releases preserve documented CLI behavior, `/api/v1` contracts, and supported configuration/artifact readers. Breaking changes require a new major contract and a documented migration path; artifact schemas carry their own versions and historical raw artifacts are not silently rewritten. Test upgrades from `v0.9.0` to `v1.0.0`, then from the previous supported stable minor to each new minor. Publish the support/deprecation window and backup-based rollback procedure with each release. These guarantees apply to documented contracts, not internal Go packages or nondeterministic model output.

---

# Non-Goals Until Needed

Avoid adding these without a demonstrated requirement:

- an Evaldrome-operated hosted SaaS
- PostgreSQL
- mandatory hosted identity or public self-registration
- distributed multi-tenant infrastructure beyond the planned single-host workspaces
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

The preferred architecture is a modular Go application with standalone CLI/CI use and an optional, single-host service. HTTP, local authentication, workspace authorization, and durable jobs are justified by serving invited users; they do not require an external identity platform or distributed infrastructure.

---

# Architectural Philosophy

Evaldrome should demonstrate backend and systems engineering through justified architectural decisions.

Technology should exist because the problem requires it.

Examples:

```text
Go
    CLI, shared model execution, benchmarks, code review,
    concurrency, provider integration, artifact generation,
    and the optional HTTP service with bounded workers

Braintrust
    optional specialized AI experiment tracing and analysis

Portable files
    reproducible suites, datasets, review policies, snapshots,
    findings, verification records, baselines, and CI artifacts

SQLite
    required single-host service state and durable jobs;
    optional standalone history when useful

CI systems
    benchmark automation, code reviews, and regression enforcement

GitHub adapter
    PR context and narrowly scoped review publishing
```

This architecture defines the intended design. [The release checklist](release-checklist.md) defines the executable path and acceptance evidence for shipping it.

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

Build that well first. Add richer scoring, optional Braintrust, and experiment comparison next, then apply the same infrastructure to local review, GitHub review, and measured reviewer comparisons. Wrap those engines in the personal and multi-user service milestones, then prove operation and compatibility before declaring the complete product stable.

The product should help developers answer: why am I using this model, how reliable has it been, and would a different model be better for this workload?

When implementation experience proves an assumption in this document incorrect, update the architecture rather than forcing the code to preserve an outdated plan.
