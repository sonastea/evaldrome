# Evaldrome Release Checklist

## Status and Release Contract

**Current state: planning only. Current implementation target: `v0.1.0`.** No CLI, engine, or service has shipped from this repository. Every implementation/release item below is intentionally unchecked.

The stable product lets a developer benchmark models, review fixed code changes, and compare evidence-based results locally or in CI. An operator can also deploy the same product on one Linux host for themselves and invited users, with private/shared workspaces and a basic web UI/API. Standalone use needs no server or account; hosting needs no Evaldrome cloud account or mandatory Braintrust service.

[Architecture](architecture.md) owns design decisions and scope. [AGENTS.md](../AGENTS.md) owns implementation guidance. This file owns acceptance gates. Public signup/billing, distributed execution, advanced dashboards, battles/rankings, and code-execution sandboxes are outside the initial stable-release requirement.

### How to use this checklist

- Complete milestones in dependency order; keep each implementation change small enough to validate.
- Check an item only after its behavior exists and the stated validation passes. A design paragraph or an unrun test is not completion.
- Link evidence beside completed items or in a milestone evidence record: implementation commit/PR, commands and outcomes, fixtures/artifact identities, environment, and documentation exercised.
- Ordinary tests use deterministic fake providers/adapters. Real-provider smoke tests are opt-in, budgeted, and explicitly authorized; record provider/model/options and observed usage without exposing keys or private inputs.
- Record blockers and remaining limitations. A blocked or skipped gate is not a pass. If scope changes intentionally, update the architecture and this checklist together.
- Assign an owner to each milestone before starting it. Release dates follow working acceptance evidence, not the version number alone.

Suggested evidence record for each milestone:

```text
Milestone / owner:
Commit / build version:
Platform / Go / formatter / provider versions:
Checks run and outcomes:
Fixture and result artifact locations / hashes:
Installation or operator walkthrough:
Known limitations / unresolved blockers:
```

## v0.1.0 — Reproducible Local Benchmarks

**Acceptance demo:** from a clean checkout, run a bundled JSONL dataset through the fake provider, inspect the summary and saved raw artifact, then repeat with identical inputs and matching deterministic scores.

- [ ] Create the Go module and CLI entry point; pin the Go toolchain/development tools and `gofumpt` version; document build, format, test, and vet commands.
- [ ] Define Evaldrome-owned request/response, case, score, status, and artifact types without provider SDK or presentation dependencies.
- [ ] Load UTF-8 JSONL records with a unique string `id` and string `input` and `expected` fields; reject empty/malformed datasets and duplicate IDs before inference, with actionable locations.
- [ ] Implement a deterministic fake provider and one real provider; validate configuration and requested options before dispatch where possible.
- [ ] Implement exact-match scoring with documented string semantics; distinguish mismatches, provider failures, scorer failures, and unscored/canceled cases.
- [ ] Bound concurrent provider calls; test timeouts and cancellation with a blocking fake, including propagation to active requests and no leaked workers.
- [ ] Save a versioned artifact containing raw responses, scores, failures, dataset hash, run/model/settings identity, timestamps, latency, and nullable usage/cost. Preserve completed work on interruption.
- [ ] Define CLI exit behavior, output locations, overwrite policy, and stdout/stderr behavior; show coverage alongside scores and errors without requiring Braintrust.
- [ ] Pass meaningful unit/integration tests, `go test -race ./...`, `go vet ./...`, and the pinned formatting check without paid calls.
- [ ] Add a runnable example dataset, README quickstart, build/install instructions, and one authorized real-provider smoke result with secrets excluded.

## v0.2.0 — Scoring, Tracking, and Experiment Comparison

**Acceptance demo:** run two model configurations against one versioned suite, compare saved artifacts without new inference, and repeat the workflow with tracking disabled.

- [ ] Introduce versioned suite configuration, deterministic validation, documented precedence, and input/scorer/model configuration hashes.
- [ ] Add useful deterministic scorers with boundary/error fixtures; record scorer versions and make derived summaries recomputable from preserved raw results.
- [ ] Support at least two provider adapters, covering a documented cloud provider and a local/self-hosted inference endpoint. Validate actual supported options rather than assuming endpoint compatibility.
- [ ] Implement capability-aware reasoning validation, classified retries, rate-limit/Retry-After handling, bounded backoff, and cancellation during waits.
- [ ] Record all attempts and available usage; label pricing assumptions and estimated cost; display missing usage/pricing as unknown.
- [ ] Compare compatible saved runs by score, coverage, latency, usage, and cost; reject or explicitly label dataset/scorer/configuration mismatches.
- [ ] Add the optional Braintrust adapter; test disabled/unavailable tracking and explicit payload-export settings without losing local results.
- [ ] Document and exercise a local-inference workflow with remote tracking disabled and no mandatory hosted identity or telemetry dependency.

## v0.3.0 — Local Code Review

**Acceptance demo:** review a fixture branch at a fixed commit, inspect located findings, and reproduce the same selected context after the branch name moves.

- [ ] Resolve base/merge-base/head and freeze the snapshot/diff before inference; record content hashes, context manifest, policy hash, and prompt/context-selection versions.
- [ ] Bound files, bytes, tokens, dependency depth, requests, and elapsed time; expose skipped/unreadable/truncated context and required-coverage failures.
- [ ] Keep context read-only and inside the snapshot; test path/symlink escapes and disabled hooks, external diff helpers, repository scripts, and model-proposed commands.
- [ ] Implement versioned policies with documented focus, severity, evidence, model/reasoning controls, and override precedence; resolve permissions from trusted configuration.
- [ ] Validate structured findings, evidence, source locations, and provenance; retain raw output plus filtering/deduplication decisions.
- [ ] Assign verification status independently of the model and distinguish complete, partial, failed, and canceled reviews, including malformed and empty outputs.
- [ ] Test fixtures for valid/invalid findings, duplicate reports, missing coverage, provider failures, cancellation, and secret-free artifacts/logs.
- [ ] Document `review --base` behavior and a useful local workflow that produces portable findings without implicit publication.

## v0.4.0 — GitHub Actions and PR Reviews

**Acceptance demo:** a fixture PR is reviewed through the same engine, published against the reviewed commit, and retried without duplicate comments; a changed PR head is rejected as stale.

- [ ] Wrap the CLI in a documented Action; keep GitHub metadata and publishing APIs outside the workload engines.
- [ ] Separate read-only review from write-enabled publishing and test that review workers never receive publishing credentials.
- [ ] Validate artifact origin, repository/PR identity, immutable reviewed commit, schema/output bounds, and eligible diff locations before posting.
- [ ] Implement commit-aware, idempotent comments/summaries and explicit opt-in request-changes behavior, disabled by default.
- [ ] Preserve artifacts and report publishing failure separately from successful inference; cover retries, rate limits, and stale heads using a fake GitHub adapter.
- [ ] Document and test fork/untrusted-PR behavior without executing PR code with secrets or using privileged workflow shortcuts.
- [ ] Publish deterministic command-specific exit behavior and a portable-artifact CI example; ordinary tests do not contact GitHub or paid providers.

## v0.5.0 — Review Comparisons and Quality Measurement

**Acceptance demo:** independent fake reviewers analyze identical fixtures with known defects; calculated recall, adjudicated precision, noise, and unresolved counts match hand-checked expectations.

- [ ] Freeze common snapshot/diff/policy/context once for comparison and keep reviewers independent; record contender-specific model/reasoning settings and context differences.
- [ ] Build versioned buggy and bug-free fixtures with independent defect evidence, hidden labels, and separate tuning/held-out cases.
- [ ] Record confirmed/rejected/unresolved verification with evidence, verifier identity, timestamp, and revision history linked to original finding IDs.
- [ ] Implement versioned issue matching/deduplication with per-model attribution; duplicate reports do not inflate detections.
- [ ] Test metric denominators, zero/unknown populations, partial runs, selective verification, and unresolved findings; never equate another model's agreement with confirmed correctness.
- [ ] Produce a per-model issue matrix plus quality, coverage, latency, and cost comparisons without one opaque winner score.
- [ ] Document explicit curation of verified feedback into new benchmark versions without automatic export of private code.

## v0.6.0 — Personal Self-Hosting Alpha

**Acceptance demo:** install on a clean Linux host, bootstrap an account locally, submit/cancel a benchmark in the browser, restart, and retrieve the preserved artifact with Braintrust disabled.

- [ ] Implement `evaldrome serve` as an adapter around existing engines with embedded UI assets; standalone commands still run without its database or accounts.
- [ ] Add local operator bootstrap, one private workspace, hashed passwords/tokens, protected sessions, login throttling, expiry/revocation, and CSRF protection. No public unauthenticated setup endpoint exists.
- [ ] Provide authenticated dataset upload, run submission/status/cancellation/history, and authorized artifact download through the UI and documented `/api/v1` API.
- [ ] Add SQLite migrations, foreign keys, transactional queue/attempt metadata, immutable resource versions, atomic file finalization, and integrity checks.
- [ ] Persist submission idempotency with actor/workspace/request-hash scope; test identical retries and conflicting reuse. HTTP disconnects do not terminate jobs.
- [ ] Test restart at enqueue, claim, provider dispatch, result write, and terminal-state boundaries. Preserve queued jobs and completed work; mark ambiguous interrupted requests explicitly without automatic billable replay.
- [ ] Add bounded workers, durable cancellation, graceful shutdown, storage-failure handling, and loopback-by-default binding.
- [ ] Resolve only operator-authorized secret references/endpoints; exclude secrets and host secret paths from logs/exports, and expose private payloads only through authorized workspace artifacts rather than logs.
- [ ] Ship a Linux binary and non-root container, persistent-volume/service-manager/Compose examples, HTTPS/proxy configuration, and a tested maintenance-window backup/restore walkthrough.

## v0.7.0 — Multi-User Self-Hosting Beta

**Acceptance demo:** the operator invites two users into separate private workspaces and one shared workspace; each completes a run, collaborates where permitted, and cannot access the other's private resources.

- [ ] Implement single-use expiring invitations, operator-assisted account recovery, workspace admin/member/viewer roles, and scoped expiring/revocable API tokens without requiring email or external login services.
- [ ] Authorize inputs, provider/repository grants, jobs, results, logs, downloads, comparisons, verification, and publishing server-side. Cross-workspace references do not disclose private object existence.
- [ ] Test role changes, membership removal, token expiry/revocation, and credential-grant removal for both queued and active work.
- [ ] Provision separate per-workspace BYOK credentials or explicitly shared operator credentials with limits; no fallback selects another workspace's key or arbitrary environment variable.
- [ ] Add workspace-authorized repository registration/read credentials and review commits pinned before admission. Test upload/path traversal, hostile Git configuration, sanitized subprocess environments, network redirects/DNS rebinding, and denied outbound destinations with deterministic fixtures.
- [ ] Enforce global and per-workspace queue/concurrency/request/token/time/storage limits and fair scheduling across concurrent jobs. Unknown prices cannot bypass required cost admission checks.
- [ ] Expose benchmarks, reviews, compatible artifact comparisons, verification history, and explicitly authorized GitHub publishing through the service; results remain exportable to standalone tools.
- [ ] Add audit records for membership/grants, submissions, cancellations, verification revisions, deletions, and publication without logging secrets or raw private payloads.
- [ ] Test cross-workspace access at both HTTP and storage boundaries, including guessed IDs, pagination, artifacts, caches, and comparison references.
- [ ] Exercise usable browser flows for each role, including keyboard navigation, validation messages, queued/running/partial/error states, and membership administration; test that malicious model/repository text is escaped and downloads cannot execute as trusted page content.

## v0.8.0 — Operational Readiness and CI Gates

**Acceptance demo:** back up a populated instance, restore it onto a fresh host with outbound work paused, verify its data/access rules, then upgrade and rehearse the documented rollback.

- [ ] Test consistent database-plus-file backups and restoration of accounts, memberships, jobs, artifacts, hashes, verification history, and comparisons. Document separate secret recovery and keep restored dispatch/publishing disabled until explicitly enabled.
- [ ] Implement supported-version migration checks and test upgrade failures. Rehearse whole-backup rollback and document loss of post-backup writes.
- [ ] Implement storage ceilings, retention, and explicit deletion with active-job/reference handling; document effects on dependent comparisons, remote copies, and backups.
- [ ] Provide minimal liveness/readiness checks, secret-redacted bounded logs, authenticated metrics/audit access, and actionable degraded-state errors.
- [ ] Run at least 100 mixed fake-provider benchmark/review jobs from 10 clients across three workspaces with four global/two per-workspace model requests maximum. Inject rate limits, timeouts, cancellations, and restart; verify limits, workspace progress, and explicit terminal outcomes.
- [ ] Record hardware, input/context sizes, queue latency, peak memory/disk, and recovery behavior; document tested capacity rather than claiming unmeasured scale.
- [ ] Exercise disk-full, missing/corrupt artifact, database-busy, provider outage, and tracking outage scenarios without false success or lost acknowledged queued jobs.
- [ ] Implement portable baselines and reproducible CI quality/regression gates with stable exit codes; incomplete required coverage cannot pass as a clean result.
- [ ] Publish an operator guide for install, HTTPS, invitations, provider grants, quotas, upgrades, backup/restore, retention/deletion, and troubleshooting, plus the supported platform/provider matrix.

## v0.9.0 — Release Candidate

**Acceptance demo:** a second person follows only the documentation to deploy the candidate, invite users, run the core workflows, and recover the instance from backup.

- [ ] Freeze and document intended `v1` CLI flags/exit codes, API/OpenAPI schemas, suite/policy/configuration formats, artifact schemas, error codes, and comparison semantics.
- [ ] Add compatibility fixtures for documented CLI/API/artifact behavior and supported schema readers; publish upgrade/deprecation and backup-based rollback policies.
- [ ] Complete a clean-install walkthrough by someone other than the documentation author using release binaries/container images.
- [ ] Complete at least seven consecutive days of a self-hosted pilot with two non-operator accounts, private/shared workspaces, representative benchmark/review/comparison jobs, and recorded issues/outcomes.
- [ ] Pass deterministic workload, integration, race, HTTP authorization, queue/restart, load, and backup/restore checks on the release build; record opt-in cloud/local-provider smoke evidence separately.
- [ ] Verify operation with Braintrust disabled and the documented local inference path; no Evaldrome cloud login, mandatory external tracking, or service database is required for standalone use.
- [ ] Rehearse the `v0.9.0`-to-`v1.0.0` upgrade against the candidate stable build and retain artifact/access-control integrity evidence.
- [ ] Resolve all release-blocking core-workflow, isolation, credential-exposure, data-loss, and recovery defects; record remaining non-blocking limitations explicitly.

## v1.0.0 — Stable Release Gate

All milestone acceptance requirements above must be implemented and validated, even if intermediate releases were combined. A feature remains optional to use when listed as such; its committed implementation cannot be silently omitted from the stable scope.

- [ ] **Local/CI:** a clean environment runs benchmarks, local reviews, comparisons, and regression gates with portable artifacts and documented errors/cancellation, without a service or database.
- [ ] **Personal hosting:** a clean host runs the packaged service with authentication and persistent results using only the published setup instructions.
- [ ] **Serving other users:** the operator can invite users, assign private/shared workspaces and credentials, enforce quotas, and demonstrate isolation and collaboration through both UI and API.
- [ ] **Review quality:** independent verification, deduplication, partial coverage, unresolved findings, and metric denominators remain visible and correct in representative results.
- [ ] **Recovery:** consistent backup/restore, restart behavior, migration, rollback, retention, and corruption/storage-failure handling pass on the release build.
- [ ] **Compatibility:** documented CLI/API/configuration/artifact contracts have passing fixtures; supported platforms/providers, limitations, and upgrade/support policies are published.
- [ ] **Distribution:** versioned binaries and container images are built from the tagged source with checksums/digests, license/third-party notices, release notes, and tested installation commands. Review dependency vulnerability results and resolve release-blocking findings.
- [ ] **Release evidence:** a named release owner records the build/commit, completed gates, test and pilot outcomes, restore/upgrade evidence, and known limitations; no unresolved release blockers remain.
- [ ] **Publication:** after the release is explicitly authorized, publish `v1.0.0` artifacts and smoke-test their install/version/quickstart paths; document the issue-reporting and patch-release process.

## Immediate Next Work

Start with the `v0.1.0` vertical slice: Go module and pinned tooling, dataset validation, fake provider, exact-match scorer, bounded runner, raw artifact, and CLI summary. Prove it with deterministic tests before adding the first real-provider adapter. Keep the service boundary in mind while implementing the engines; begin service implementation at `v0.6.0`.
