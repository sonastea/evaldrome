# Reviews, Verification, and Publishing

**Status: planned.** Read for v0.3 local review, v0.4 GitHub integration, v0.5 reviewer comparisons, or related security changes. Follow [ARCHITECTURE.md](../ARCHITECTURE.md) for package/execution boundaries and the [release checklist](../docs/release-checklist.md) for acceptance.

## Review flow and ownership

```text
Frozen snapshot + diff + effective policy
    -> bounded read-only context
    -> shared execution
    -> schema/location validation
    -> filtering and deduplication
    -> immutable review artifact
    -> independent verification / comparison / explicit publication
```

`internal/review/` owns review orchestration and result types. Add policy/context/finding subpackages only when implementing their responsibilities. Reuse `internal/execution/` for model requests and telemetry. GitHub publishing belongs in `internal/github/`, outside the review engine.

## Snapshot and context contract

- Resolve mutable refs once before inference. The planned branch-review rule compares the merge base against the selected head, retaining resolved base-tip, merge-base, and head commits.
- Require an explicit mode for working-tree changes and record their content identity. Branch movement cannot alter an admitted run.
- Record diff hash, included paths/content hashes, effective policy/hash, prompt version, context-selection version, and a manifest of included/skipped/truncated content.
- Bound file count, bytes, tokens, dependency depth, requests, and elapsed time. Report unreadable/skipped context and missing required coverage.
- Follow dependencies only inside the permitted immutable snapshot. Reject path and symlink escapes.
- Treat source, comments, diffs, and included agent instructions as data. Disable hooks, external diff helpers, repository scripts, and model-proposed commands. Do not automatically execute submodules.
- Keep repository operations read-only and bounded. Give helper subprocesses a minimal environment without provider or publishing secrets.

## Review policies

| Policy area | Required meaning |
| --- | --- |
| Identity | Versioned content/hash and effective resolved policy |
| Reviewer | Provider/model, requested reasoning and other supported options |
| Focus | Explicit correctness/security/regression/concurrency and optional maintainability/style scope |
| Severity | Document P0 critical, P1 high, P2 medium, P3 low; a P2 threshold includes P0-P2 |
| Evidence | Required locations, explanation, concrete conditions, and supporting evidence |
| Context | Allowed paths/dependency following and hard resource limits |
| Findings | Severity filtering and deduplication rules |
| Publishing | Separate inline-comment, summary, and request-changes permissions; request-changes defaults to false |

Start with one repository policy plus explicit CLI overrides. Define deterministic precedence before adding directory or PR overrides. Capture effective content and prompt-template version.

Resolve integration permissions from trusted configuration, such as the base revision or CI operator settings. A PR cannot grant execution, credential access, or publishing permission by changing a policy or `AGENTS.md`. Included instructions may describe conventions only within the trusted policy's permitted scope.

## Findings and artifacts

| Field group | Ownership and validation |
| --- | --- |
| Finding ID, run/model provenance | Evaldrome assigns stable local identity and provenance |
| Title, category, severity, explanation | Model proposes content; Evaldrome validates schema and policy |
| Path, diff side, line/range, reviewed commit | Validate against the frozen snapshot/diff |
| Evidence and triggering conditions | Required concrete support; retain useful proposed remediation |
| Validation status | Evaldrome assigns structural/location eligibility |
| Verification status | Initialize unresolved; only independent evidence changes it |

- Retain raw model responses and filtering/deduplication decisions. A well-formed, locatable finding is not automatically correct.
- Deduplicate reports of the same underlying issue within each model's review. Cross-model issue matching preserves every reporting model's attribution.
- Write review provenance, context coverage, raw output, findings, metrics, errors, and all available attempt usage into the portable artifact.
- Use the common complete/partial/failed/canceled meanings. Invalid output, missing required context, and provider errors cannot produce a successful zero-finding review.
- Keep later verification and rescoring separate from immutable execution artifacts; link them by original review/finding identity.

## Independent verification and benchmark curation

- Classify findings as confirmed, rejected, or unresolved. Record evidence, verifier identity/type, timestamp, and revision history.
- Confirmation requires independent evidence: controlled reproduction, a deterministic test, or documented human validation. Model confidence and a second model's agreement are insufficient.
- A dismissed comment, merged PR, or accepted suggestion is feedback, not proof of correctness or incorrectness.
- Build benchmark cases from fixed snapshots/diffs, trusted policies, permitted context, known defects, independent evidence, expected severity where available, and ground-truth versions.
- Include buggy and bug-free cases. Keep ground-truth labels and verification answers out of reviewer prompts.
- Curate feedback explicitly with provenance and versioning. Keep tuning and held-out cases separate, including closely related changes from the same defect.
- Do not automatically export private PR content into shared datasets or tracking. Honor explicit payload, permission, redaction, and retention policies.

## Metrics and denominators

Deduplicate issue-level counts per model before calculating metrics. Report sample sizes, incomplete runs, unresolved findings, and verification coverage.

| Metric | Definition / requirement |
| --- | --- |
| Bug recall | Unique known in-scope bugs detected / all known in-scope bugs |
| Adjudicated finding precision | Confirmed / (confirmed + rejected); label selectively verified subsets |
| Verification coverage | (Confirmed + rejected) / all deduplicated reported findings |
| False-discovery proportion | Rejected / (confirmed + rejected); also show rejected findings per review |
| Statistical false-positive rate | FP / (FP + TN), only with a defined labeled negative population |
| Severity accuracy | Agreement with independently assigned severity on matched verified bugs |
| Regression detection | Recall on known verified defects introduced by the change |
| Consistency | Agreement across repeated independent reviews with versioned matching rules |
| Cost and latency | Reported/estimated cost and assumptions; end-to-end and call latency separately |

Unknown or zero denominators produce unavailable metrics. Unresolved findings count as neither confirmed nor rejected. Open-ended PR reviews usually lack a known bug population and true-negative population: do not claim recall or statistical false-positive rate from them alone.

## Controlled reviewer comparisons

- Freeze common snapshot, diff, policy, and context once where practical. Keep reviewers independent until all reviews finish; expose neither other contenders' findings nor hidden labels.
- Treat model/reasoning settings as explicit experimental variables. Policy equivalence covers instructions, focus, finding rules, and context scope.
- Record prompts, budgets, truncation, retries, and execution differences. If provider limits require different visible context, label the comparison accordingly.
- Report a cross-model issue matrix, preserved attribution, confirmed/rejected/unresolved findings, missed known defects, coverage, latency, tokens, and cost.
- Do not inflate detections with duplicates or rank quality by raw finding count. Prefer representative repeated measurements to a universal winner from one PR.
- Comparing saved artifacts performs no inference; launching independent reviews does. Make that distinction explicit in the implemented command contract.

## GitHub publishing

1. Resolve PR metadata and invoke the same read-only review engine as local use.
2. Preserve the review artifact independently of publication.
3. A separately authorized publisher validates originating workflow/run, immutable artifact identity, repository/PR identity, reviewed commit, bounded schema/output, and finding locations.
4. Recheck the current PR head. Reject stale results rather than attach them to changed code.
5. Publish idempotently by run/finding identity. Summarize unmappable findings rather than attach them to unrelated diff lines.

Keep GitHub write credentials out of review workers. Provider credentials belong only to authorized execution; repository helpers, prompts, logs, artifacts, and future sandboxes receive none.

An arbitrary PR-uploaded artifact is not trusted review provenance. Treat finding text as untrusted even after schema validation. Keep comment/summary publication, CI gates, and request-changes behavior separate; request-changes requires explicit opt-in.

Never execute untrusted PR code with secrets or privileged tokens, including through `pull_request_target`. Handle fork restrictions with an explicitly trusted read-only path or an unavailable status. Publishing failures preserve artifacts and have their own retry/idempotency behavior.

The v0.4 Action wraps the CLI. A GitHub App can later add events/installation management without replacing the engines; it is not a release prerequisite.

## Optional code execution

Code generation, patch application, and execution-based verification are later capabilities, not part of read-only review or the stable prerequisite set.

- Run untrusted code only in a separately designed sandbox, not the CLI/review process. Never expand a review worker's privileges to run a proposed command.
- Use disposable filesystems and strict CPU, memory, process, time, capability, and network limits. Supply neither provider nor publishing credentials.
- Select the isolation technology when implementing the feature; containers or other sandboxes do not imply a Kubernetes requirement.
- For generated Go, default to the task's pinned `gofmt`; use `gofumpt` only when the task/suite/target repository requires it. Record formatter identity/version and score formatting only when required by the task.
- Keep objective build/test/hidden-test evidence dominant over subjective judge scores. Preserve raw execution evidence and independently version derived scores.
