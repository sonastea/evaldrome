# Experiments, Tracking, and Comparisons

**Status: planned.** Read for v0.2+ suite/scoring/provider/tracking/comparison work or v0.8 CI gates. Use [ARCHITECTURE.md](../ARCHITECTURE.md) for the v0.1 loop and package boundaries; use the [release checklist](../docs/release-checklist.md) for acceptance. Nothing here expands the active milestone automatically.

## Concepts and ownership

| Concept | Meaning |
| --- | --- |
| Suite | Versioned dataset, model, scorer, and execution definition |
| Case | One identifiable evaluation input and its expected result |
| Benchmark run | Scored cases for one effective model configuration |
| Experiment | Grouping of compatible benchmark runs |
| Contender | Model plus effective options participating in a comparison |
| Baseline | Portable prior result used for comparison or regression decisions |
| Judge | Model supplying supplementary subjective assessment |

Benchmark and review runs share execution and artifact conventions, not orchestration or a universal result schema. Add comparison logic under `internal/compare/`; keep provider SDKs inside their adapters and Braintrust inside `internal/braintrust/`.

## Versioned suites and datasets

- Introduce generalized suite configuration at v0.2. Define name/version, dataset identity, model configurations, scorer versions/options, and execution settings. Keep files reviewable in source control.
- Define validation and override precedence before exposing a configuration format. Record the effective configuration and hashes, not only the source file path.
- Increment suite/scorer versions when behavior materially changes. Preserve historical definitions and raw results.
- Identify the exact dataset bytes used by a run. A changed dataset, scorer, or normalization rule is a compatibility difference, not evidence of a model regression.
- Validate configurations and inputs without inference. Document the implemented CLI syntax when available; illustrative YAML or model names are not a supported API.

## Scoring

- Prefer deterministic checks: exact match, JSON/schema validity, or other task-specific checks. Build/test/code-execution scorers require the separate [sandbox boundary](reviews.md#optional-code-execution), not execution inside an ordinary benchmark/review worker.
- Keep correctness, coverage, provider reliability, latency, tokens, and cost independently visible. Do not collapse them into an unexplained universal score.
- Distinguish score mismatch from scorer failure and provider failure. Return unavailable metrics for missing denominators; preserve the raw inputs required for later rescoring where permitted.
- Version scoring and issue-matching rules. Keep derived summaries separate from immutable run evidence.
- Add LLM judges only when deterministic scoring is insufficient. Require structured assessment and retain judge identity/options/output. Prefer an independent judge over self-judging when practical.
- Judge scores supplement objective evidence. Agreement from another model cannot confirm that a reported bug exists.

## Providers and reliability

- By v0.2, support at least two adapters covering a documented cloud provider and a tested local/self-hosted endpoint. A configurable base URL does not prove compatibility with every API-compatible server.
- Validate capabilities per provider/model/API endpoint. Reasoning labels such as `high` are not universal capabilities or equivalent compute budgets.
- Reject unsupported settings. Record requested settings, effective settings when known, provider-default uncertainty, resolved model identity, and reasoning-token usage when reported.
- Normalize authentication, invalid-request, rate-limit, timeout, connection, provider, cancellation, and unknown failures. Define stable serialized codes in the execution contract rather than exposing SDK exceptions.
- Classify retries explicitly. Bound attempts and elapsed time; honor supported `Retry-After` behavior; use exponential backoff with jitter and cancellation-aware waits.
- Disable or coordinate SDK retries so attempts are not hidden or multiplied. Do not blindly retry authentication/validation failures or automatically replay ambiguous requests without a provider guarantee.
- Record successful and failed attempts, available usage, and unknown charges. Keep end-to-end latency separate from provider-call latency.
- Compute estimated costs only from known usage and pricing assumptions. Missing usage/pricing is unknown. Estimated spend controls do not guarantee an invoice cap.
- Use deterministic fake providers for success, malformed responses, errors, capability rejection, timeouts, cancellation, and concurrency tests. Real-provider smoke runs are separately authorized and budgeted.
- Test local inference with remote tracking disabled and no mandatory hosted identity/telemetry calls. Model weights, downloads, and inference hardware are operator-provisioned.

## Braintrust adapter

| Concern | Contract |
| --- | --- |
| Availability | Opt-in from v0.2; disabled/unavailable tracking does not prevent useful local or service results |
| Types | Translate Evaldrome-owned lifecycle events/results inside `internal/braintrust/`; no SDK types in engines |
| Mapping | Benchmark experiments may map to experiments; benchmark/review runs to traces; calls to spans |
| Payloads | Metadata, inputs/outputs, scores, and findings follow explicit capture/redaction settings |
| Privacy | Selecting a destination does not enable raw repository/prompt export; never export credentials |
| Service use | Workspace-authorized destination and payload policy; no implicit operator credential inheritance |
| Failure | Report tracking failure separately from execution/scoring; retain portable results |

Prefer the Braintrust Go SDK after verifying required capabilities. Do not force review runs into synthetic benchmark cases or build a generic observability framework before a concrete adapter requires it.

## Compatible comparisons and baselines

- Comparing saved artifacts does not repeat inference. Starting an experiment or independent reviewer comparison does perform model calls; keep these operations explicit.
- Require compatible dataset/suite/scorer versions and settings. For reviews, also compare snapshot/diff, policy, prompt/context contract, ground truth, and matching version.
- Record intended experimental variables, including model and reasoning options. Report context limits, truncation, retries, and other execution differences; do not label unlike inputs as a controlled comparison.
- Show sample counts, failures, coverage, quality, latency, usage, and cost. Use representative repeated runs rather than declare a universal winner from one task or PR.
- Store baselines in portable artifacts usable by local and ephemeral CI environments. Service/database history must not be required to compare them.
- Preserve original runs when applying new scoring or baseline rules; create versioned derived results.

## CI gates

- Add GitHub Actions as a CLI wrapper at v0.4; introduce reproducible regression gates at v0.8. Keep workload logic independent of CI vendor APIs.
- Define deterministic, command-specific exit codes, machine-readable output, baseline compatibility checks, and threshold semantics before release. The first CLI slice must already define its own exit contract.
- Keep execution status, quality-gate outcomes, finding verification, and GitHub request-changes decisions independent.
- Missing required coverage and provider failures cannot pass as clean evaluations. An incompatible baseline is a comparison/configuration problem, not automatically a quality regression.
- Retain raw artifacts if tracking, upload, or publication fails. Do not depend on CI runner state surviving between jobs.

## Optional later work

- Pairwise battles, leaderboards, and optional SQLite CLI history are not stable-release prerequisites.
- For a battle, hide model identity, randomize A/B placement, allow ties, and retain raw judge responses and assignment metadata.
- Elo with an initial rating of 1500 is a candidate first ranking policy, not a selected implementation. Version any adopted rating rule; recompute from raw battles when changing algorithms.
- Keep preference rankings separate from verified correctness. Judge preference for a review is not evidence that its findings are true.
- Additional exporters, providers, registries, plugins, and distributed execution require demonstrated needs and intentional architecture changes.
