# Testing — Behavioral Contracts and Evidence

**Status: testing specification for the planned v0.1.0 CLI.** The repository has no Go module, source, tests, or CI configuration yet. Commands below require bootstrap; this guide is not evidence that tests exist or pass.

This file owns test design, layers, fixtures, validation standards, and development evidence. [Architecture](../ARCHITECTURE.md) owns package boundaries and product behavior; the [technology specification](tech_stack.md) owns toolchain/dependency selections; the [release checklist](../docs/release-checklist.md) owns acceptance gates. Future capability documents retain their domain contracts. Resolve disagreements with those owners rather than encoding a new product decision in a test.

Read the core standards and the relevant matrix rows when implementing or fixing behavior. Read future-milestone sections only when that capability is in scope. No testing framework, coverage percentage, or additional infrastructure is required by this guide.

## Core standards

### Assert behavior with independent expectations

- Test observable outcomes and documented contracts through package APIs, CLI output/exit status, artifacts, and external effects. A test should usually survive an internal refactor unchanged.
- Prefer external test packages, such as `dataset_test`, for package API behavior. Use same-package tests for otherwise unobservable critical invariants, such as bounded worker creation or cleanup ownership; explain the invariant. Do not export helpers just for tests or assert private structures and call sequences without that reason.
- Derive expected values from requirements, specifications, hand-checked examples, or independently established fixtures. Never call the production scorer/aggregator to compute its own expected result, reproduce its algorithm in test code, or bless existing output because it is current behavior.
- For each meaningful case, identify the contract, explicit input/stimulus, observable assertion, and plausible defect it would detect. Add a short contract/oracle source comment when the expectation is not obvious. An ambiguous contract is a decision to resolve, not an opportunity to guess a golden value.
- Assert enough to distinguish correct behavior from a plausible broken implementation. A non-nil error alone is insufficient when classification, record location, retained output, or coverage matters. Use `errors.Is`/`errors.As` for contractual error categories; assert exact prose or formatting only where it is a documented output contract.
- Use table-driven cases for related inputs with distinct names and explicit expectations. Keep assertions simple: no miniature execution engine, branching to accept incompatible outcomes, or calculations that reconstruct the implementation. Small loops over cases or fields and focused builders are appropriate when expectations remain visible.
- Passing tests are evidence, not proof of correctness. Coverage measures execution, not assertion quality; use uncovered behavior to investigate gaps, never a percentage as a substitute for behavioral evidence.

### Exercise real behavior deterministically

- Use real deterministic, inexpensive implementations: dataset parsing, scoring, aggregation, serialization, temporary files, and local HTTP transport. Use small deterministic fakes at external provider boundaries and `net/http/httptest` for HTTP adapters.
- Inject clocks, random sources, filesystem operations, and external systems only where behavior needs control. Follow constructor injection and consumer-owned interfaces from the architecture; do not invent interfaces to mock private methods or hypothetical dependencies.
- Observe provider requests/responses, active requests, and externally visible attempt counts when these are contractual. Avoid assertions about internal method invocation order or a specific scheduling implementation.
- Ordinary tests run offline, without real credentials or live external services, including locally running models. Use disposable resources, explicit synchronization, and bounded cleanup. No arbitrary sleeps, shared mutable state, test-order dependencies, or assertions on exact wall-clock duration.
- Tests must work independently and repeatedly. Parallelize only independent cases; ensure failures also release blocked fakes, cancel contexts, close servers/files, and join owned goroutines.

### Reproduce defects before fixing them

For a bug fix:

1. Identify expected behavior and its contract or independent evidence. Resolve ambiguity first.
2. Add a focused regression test that exercises the defect at the appropriate boundary.
3. Run it against the unfixed behavior, uncached when needed, and inspect the failure. A compilation error, unrelated setup failure, or timeout caused by the test harness is not the required reproduction.
4. Apply the smallest appropriate fix.
5. Rerun the regression test and confirm it passes.
6. Run relevant surrounding tests and the applicable validation gates below.
7. Report the original failing command/assertion and final results. If reproduction is impossible or unsafe, explain why and state what alternative evidence was obtained.

Never weaken an assertion, delete a failing case, silently skip it, or regenerate an expectation to accommodate incorrect behavior. If a requirement legitimately changes or a test is wrong, correct the owning contract/oracle with an explicit explanation and preserve relevant regression coverage. Report unrelated existing failures rather than treating a partially passing run as a pass.

## Test layers and execution frequency

Layers describe confidence boundaries, not new packages or frameworks. A contract case may be a unit, component, or process test; do not duplicate it merely to give it another label.

| Layer | Appropriate boundary and purpose | Expected frequency |
| --- | --- | --- |
| Unit (`U`) | Package APIs for dataset validation, exact match, aggregation, error classification, status determination, artifact metadata/schema validation, and capability validation. Use explicit inputs and independently known outputs. | Affected packages during development; all in the ordinary pre-merge suite. |
| Component/integration (`I`) | Real package composition: loading through benchmark execution with a fake provider; HTTP adapter with `httptest`; injected CLI; storage serialization/staging/finalization; cancellation and partial results. Use disposable real dependencies when they add confidence. | Affected boundaries during development; deterministic v0.1 cases in the ordinary pre-merge suite. |
| End-to-end (`E`) | Built CLI process with temporary inputs/output and a clean environment: valid run, invalid input, artifacts, stdout/stderr, exit status, cancellation/incomplete run. Exercise actual wiring, not only an injected command function. | Small offline suite before merging meaningful Go changes and at milestone acceptance; targeted runs when process/wiring behavior changes. |
| Contract (`K`) | Stable provider, dataset, versioned artifact, and CLI boundaries; later service APIs. Shared provider expectations cover normalized outputs/errors, supported options, cancellation, attempt visibility, and unknown usage. Vendor wire fixtures remain adapter-specific. | When a boundary or dependency changes; retained compatibility fixtures in the ordinary suite. |
| Fuzz/property | Native Go fuzzing initially; assert meaningful invariants over bounded inputs. Seed cases run with ordinary `go test`; generated exploration is separately bounded. | Focused exploration when parsers or invariants change and periodically. |
| Mutation | Deliberately perturb high-value deterministic logic to evaluate assertions. See the deferred workflow below. | Selectively after useful tests exist; no default per-change or release mutation quota. |

Reuse provider contract cases across adapters as real implementations appear, without importing vendor types into engines or requiring a generic fake framework. Verify each real HTTP adapter against independently sourced examples from the selected API contract, including malformed/error responses; passing against a fake alone cannot establish vendor compatibility.

CLI component tests inject the engine/provider and output streams; process tests catch executable wiring, cleanup-before-exit, and signal handling that injection cannot establish. Keep test-only composition at an outer boundary and preserve production dependency direction. No browser E2E belongs in v0.1.

Real-provider smoke runs are separate from ordinary tests and require the authorization/budget rules in [AGENTS.md](../AGENTS.md#ask-first). The v0.1 release checklist requires one authorized result. Validate the chosen adapter's actual capabilities, shape, and available metrics without expecting nondeterministic model text to match a fixed golden response.

## v0.1.0 test matrix

All packages below are planned. Add cases alongside implementation, not placeholder packages. **P0** means first-slice correctness and evidence-loss priorities; **P1** means additional v0.1 boundary/compatibility cases. Both are required for v0.1 acceptance where their contract applies; priority orders work, not permission to omit cases. `U/I/E/K` refer to the layers above.

Decision references `D`, `P`, `B`, `A`, and `C` point to [first-slice contract decisions](#first-slice-contract-decisions). Test settled invariants now when implementing them; finalize dependent contracts before asserting exact values or shipping the slice. Do not silently turn a decision blocker into a skipped passing gate.

### Dataset — `internal/dataset`

| ID / priority | Behavior and observable oracle | Boundary / level | Decision dependency |
| --- | --- | --- | --- |
| DS1 / P0 | Small valid UTF-8 JSONL loads the explicit IDs, inputs, and expected strings in source order without altering string contents. | Loader API; U, K | D: accepted empty-string and parsing policies for edge fixtures. |
| DS2 / P0 | Reject empty input, missing required fields, wrong types (including `null`), malformed JSON, invalid UTF-8, and duplicate IDs. Include valid records followed by a late invalid record; the entire run makes **zero provider requests**. | Loader U, K; loader-to-runner/CLI I | D: exact error representation; rejection of these categories is already settled. |
| DS3 / P0 | Errors identify the offending record and field/reason where applicable; duplicate-ID diagnostics are actionable. Assert safe, useful locations rather than incidental decoder wording. | Loader API; U, K | D: line/record/offset conventions, empty-input location, and error ordering. |
| DS4 / P0 | Repeated identical immutable input has identical identity. Use a fixed byte fixture and SHA-256 digest established independently of production; two accepted byte-distinct fixtures with the same decoded records retain their distinct known digests. Verify the bytes identified are the bytes evaluated, including a controlled source-change scenario. | Loader U, K; loaded dataset-to-runner I | D: digest representation; hashing original bytes rather than reserialized records is settled. |
| DS5 / P1 | Add below/at/above limit cases and parsing-policy fixtures once decisions exist; never assume the standard decoder/scanner's defaults are the product contract. | Loader API; U, K | D: byte/record limits, blank lines, unknown/duplicate members, BOM/newlines, Unicode decoding. |

### Scorer — `internal/scorer`

| ID / priority | Behavior and observable oracle | Boundary / level | Decision dependency |
| --- | --- | --- | --- |
| SC1 / P0 | Exact string match and mismatch, including whitespace, case, and canonically equivalent but unequal Unicode strings. Use the literal examples below; no implicit normalization. | Scorer API; U | B: score representation, not equality semantics. |
| SC2 / P1 | Equal empty strings and empty/nonempty pairs where the contract admits them; distinguish an explicit empty string from a missing dataset field. Include non-ASCII and multibyte characters. | Scorer U; dataset-to-scoring I | D/B: empty-string admission and scorer input contract. |
| SC3 / P0 | A mismatch is scored work, not an operational error. Provider failures are unscored; exercise actual scorer-error paths if the scorer contract has them, retaining available response data. | Scorer U; benchmark I | B: scorer failure and aggregation representation. Do not invent a failure API just for tests. |

### Execution and providers — `internal/execution`, `internal/provider/*`

| ID / priority | Behavior and observable oracle | Boundary / level | Decision dependency |
| --- | --- | --- | --- |
| EX1 / P0 | Invalid/unsupported options detectable locally produce an actionable classified error and zero provider dispatches. A supported request reaches the provider with the documented options; requested/effective settings remain distinguishable. | Execution API U, I, K; adapter `httptest` I, K | P: selected provider capabilities, option/error mappings. |
| EX2 / P0 | Provider calls retain caller context values; cancellation and deadlines reach a blocking fake/HTTP request. Request timeout while the caller is still live is a timeout failure, not user cancellation; request limits never extend an earlier caller deadline. | Execution API and adapter; I, K | P: timeout defaults, parent-deadline classification, competing-cause precedence. |
| EX3 / P0 | With more cases than the injected limit, simultaneously blocked requests never exceed that limit. Cancellation stops further scheduling, releases active work, and joins workers. Exercise normal completion, errors, timeout, and cancellation without owned goroutine/resource leaks. | Execution/benchmark I; focused internal U only for otherwise invisible worker bounds | P/B: admission/cancellation race semantics. Use synchronized starts/releases, not scheduler timing or global goroutine totals. |
| EX4 / P0 | An actually dispatched v0.1 request produces one observable attempt, including error responses; no hidden SDK/transport inference retries. Count HTTP requests for a retryable-looking response and inspect retained attempt identity/results. Rejected requests make no network call. | Execution I, K; real adapter + local server I, K | P: attempt lifecycle, pre-dispatch accounting, normalized error codes, partial responses. Retries are deferred to v0.2. |
| EX5 / P0 | Available usage is retained for successes and failures; unknown usage/cost stays unavailable through aggregation and serialization. Known zero is distinguishable from unknown. | Execution U, K; execution-to-artifact I | P/A: optional usage fields and wire representation. |
| EX6 / P1 | Selected adapter correctly translates success, malformed body, documented provider error, connection failure, and canceled request; response bodies/connections are released under expected termination. | Actual HTTP adapter with local server; I, K | P: selected vendor API and response/error contract; no invented vendor codes. |

### Benchmark — `internal/benchmark`

| ID / priority | Behavior and observable oracle | Boundary / level | Decision dependency |
| --- | --- | --- | --- |
| BM1 / P0 | Complete requests in a controlled different order; results retain dataset order, case identity, raw responses, and associated scores/errors. | Runner with real loader/scorer and scripted provider; I | B: result shape; ordering invariant is settled. |
| BM2 / P0 | All required cases successfully executed and scored, including mismatches, give `complete`; mixed usable output and operational failures give `partial`; no usable result gives `failed`; explicit cancellation gives `canceled` and retains already completed output. Test provider and applicable scorer failures separately. | Status/aggregation API U; runner I | B: usable/completed result definitions and cancellation/terminal races. |
| BM3 / P0 | Assert explicit total/completed/scored/failed/canceled/unstarted counts for all-success, all-failure, mixed, and interrupted scenarios. Score denominator is scored cases, not total or successful matches; zero scored cases yields unavailable. Use hand-checked examples below. | Aggregation U; runner I; artifact/summary I | B: exact counter definitions, overlap, score representation. Do not sum overlapping counters as if they were a partition. |
| BM4 / P0 | Cancel after one result is recorded while other work is blocked/not admitted. Retain that result, identify canceled versus unstarted cases, stop dispatch, and return after active work joins. Include cancellation before dispatch. | Runner with synchronized fake; I | B/P: observable admission/completion boundaries and race precedence. |

### Storage — `internal/storage`

| ID / priority | Behavior and observable oracle | Boundary / level | Decision dependency |
| --- | --- | --- | --- |
| ST1 / P0 | Versioned JSON preserves raw results, errors, coverage, identity/provenance, requested/effective settings, scorer metadata, limits, timestamps, latency, and nullable usage/cost. Decode an independently authored fixture and inspect emitted fields against it; add round trips without relying on round trips alone. | Serializer/reader API; U, I, K | A: field names, schema version, optional/required fields, run-ID format. Use the architecture's metadata inventory. |
| ST2 / P0 | Same-filesystem staging exposes only a complete finalized artifact. Existing destination bytes are not silently overwritten, including a controlled competing-finalization case. Assert final file contents and error outcomes, not temporary filenames or syscall sequences. | Real temporary filesystem; I | A/C: overwrite opt-in and failed-staging lifecycle; default overwrite protection is settled. |
| ST3 / P0 | Serialization and filesystem failures, including write/flush/close/finalization where applicable, return errors rather than false success. Preserve existing files and retained run evidence; the CLI must not report success after artifact-write failure. | Minimal I/O failure seams plus real filesystem I; CLI I, E | A/C: error reporting, staging cleanup, partial-artifact/reporting policy. |
| ST4 / P1 | Hand-reviewed fixtures cover each supported schema version, missing/invalid metadata, unknown versions, and documented compatibility behavior. Preserve historical fixtures when adding readers. | Artifact reader/writer; U, K | A: initial compatibility/rejection policy; do not fabricate historical versions. |

### CLI — `internal/cli` and the `cmd/evaldrome` process boundary

| ID / priority | Behavior and observable oracle | Boundary / level | Decision dependency |
| --- | --- | --- | --- |
| CL1 / P0 | Invalid arguments, configuration, dataset, and locally detectable unsupported settings produce safe actionable diagnostics and no inference. Exercise required credential configuration with synthetic values only. | Injected CLI I; representative built-process E, K | C/P/D: flags/defaults, credential selection, error/exit mapping. |
| CL2 / P0 | A valid fake-provider run writes a readable artifact and accurately reports score plus coverage. All-mismatch, mixed-failure, and zero-scored runs preserve the distinction between execution status and scoring outcome. Summary and artifact agree on independently expected counts, not just with each other. | Real CLI composition I; built process E, K | C/B/A: output channels/format, exit mapping, counters, wire schema. |
| CL3 / P0 | Cancellation reaches active requests, graceful cleanup completes within its own bound, and retained results are offered for artifact finalization. Process tests verify signal/exit behavior and incomplete-run reporting on supported platforms. | Injected CLI I; built process E, K | C/P/A: signals, exit codes, cleanup bound, and artifact failure reporting. |
| CL4 / P0 | Synthetic credential sentinels reach only the authorized provider configuration/authentication boundary; they never enter prompts, exported configuration, artifacts, diagnostics, stdout/stderr, or subprocess arguments. Include failure paths. | CLI + adapter I; process-output/artifact E | C/P: credential sources and redaction behavior. No real credentials in fixtures or test output. |
| CL5 / P0 | A failed artifact write produces an operational command failure even after successful inference; diagnostics distinguish execution, scorer, and artifact errors while retaining available coverage. | Injected CLI I; disposable invalid output destination E, K | C/A/B: error/exit mapping and what can be reported when no artifact is saved. |

### Independently known examples

These are semantic expectations, not proposed API fields, serialized score units, or exit codes. String literals below use Go escape notation.

| Expected string | Response string | Exact-match outcome |
| --- | --- | --- |
| `"ok"` | `"ok"` | Match |
| `"ok"` | `"ok "` | Mismatch |
| `"ok"` | `"OK"` | Mismatch |
| `"\u00e9"` | `"e\u0301"` | Mismatch; no Unicode normalization |
| `"猫"` | `"猫"` | Match |
| `""` | `""` | Match if empty strings are admitted |
| `""` | `"x"` | Mismatch if empty strings are admitted |

| Scripted scenario | Status | Total / scored / failed | Exact-match fraction |
| --- | --- | --- | --- |
| One match, one mismatch | `complete` | 2 / 2 / 0 | 1/2 |
| One match, one mismatch, one provider failure | `partial` | 3 / 2 / 1 | 1/2, not 1/3 |
| Two provider failures, no usable responses | `failed` | 2 / 0 / 2 | Unavailable, not zero or perfect |

These runs have zero canceled/unstarted cases. Fill in the `completed` expectation after decision B; do not assume its meaning from a field name. Cancellation fixtures must deliberately establish which work finished, was admitted, and remained unstarted before cancellation rather than accept whichever totals happen to occur.

## Fixtures, fakes, and test mechanics

### Placement and independent fixtures

- Colocate `*_test.go` with the owning package. Store fixtures under that package's `testdata/`; add no empty test packages or global testing layer. Introduce shared helpers only after concrete reuse emerges.
- Keep JSONL examples small and human-readable, with one purpose per fixture. Construct invalid UTF-8 as explicit bytes or a clearly documented binary fixture so editors/decoders cannot silently repair it. Use public/synthetic data; no private prompts, real credentials, or generated runs in source control.
- Keep input and expected output separate. For a hash fixture, record the exact bytes, independently obtained SHA-256 value, and how it was checked. For aggregation, write literal case outcomes and expected counts rather than another summation implementation.
- Use golden JSON for useful versioned artifact contracts. Author/review it against the schema independently; test both reading the fixture and writing the required fields. Round-trip equality alone can hide matching reader/writer defects. Compare semantic JSON when whitespace/key order is not contractual.
- Control time, randomness, and run IDs where needed; otherwise assert their documented properties. Do not normalize away meaningful fields, strip every unknown value, or ignore errors to make a golden comparison stable.
- Update fixtures explicitly after a reviewed contract change, inspect the diff, and retain supported-version examples. Tests never silently regenerate expected output during ordinary runs. A production-generated candidate requires independent review before becoming an oracle.
- Check `.gitignore` before adding fixtures: `artifacts/`, `results/`, `*.out`, and similar patterns can hide nested test data. Prefer descriptive fixture filenames under `testdata/`; verify they are tracked rather than weakening private-data ignore rules.

### Deterministic provider scenarios

The planned runnable `internal/provider/fake` serves the local workflow. Tests may use small package-local scripted fakes through the execution-owned provider contract; do not turn either into a general-purpose simulation framework.

- Script outcomes by observable case/request identity, not the nondeterministic order concurrent calls arrive. Provide explicit responses, normalized errors, usage present/absent, and capability declarations only as each case needs them.
- For blocking, timeout, and cancellation scenarios, expose started/release/finished signals. Block on release or the supplied context. Record requests and active/peak counts with synchronization, returning observations for assertions in the test goroutine.
- To test out-of-order completion, wait for identified requests to start, release them in a chosen order, then check final result ordering. To test cancellation, wait for active work and a retained earlier result before canceling; assert observation of cancellation, return, and no further work under the finalized admission contract.
- A generous test deadline is a hang guard, not an assertion that elapsed time matches a tiny duration. Prove request-timeout classification separately from caller cancellation. Use channels/wait groups, and a controllable clock only where the implementation actually owns time-based logic.
- Register cleanup before assertions that can abort the test: release/cancel blocked work and join owned goroutines. Observe lifecycle/resource closure; process-wide goroutine counts are noisy and do not prove worker ownership. A focused internal invariant test is justified if bounded worker creation cannot otherwise be observed.
- In `httptest` handlers and background goroutines, send observations/errors back to the test goroutine; do not rely on `t.Fatal` there to terminate the test. Local server responses come from the selected provider contract, not from copying the adapter's encoder/decoder.

### Filesystem, environment, and process isolation

- Use `t.TempDir()` for inputs, artifacts, and disposable filesystem failure scenarios. Prefer portable conflicts, such as a non-directory parent, over permission tests that change meaning under different users. For otherwise unreachable write/flush/close failures, use a narrow I/O seam justified by that failure contract; keep successful staging/finalization tests on a real filesystem.
- Use `t.Setenv()` for isolated in-process configuration tests; it is incompatible with parallel tests or parallel ancestors because the environment is process-wide. Avoid changing process working directory; prefer explicit paths. Never read a developer's live credentials as test input.
- Build process-test binaries into disposable storage and supply a deliberate child environment, input/output paths, and timeout. Ensure child processes are reaped on success and failure. Use an observable readiness handshake before sending a signal, not a sleep. Keep unsupported platform signal scenarios explicitly reported and run them on a supported acceptance platform.

## Validation commands and gates

### Prerequisites

Go commands apply **only after** the module, source, `.go-version`, and pinned formatter tool entry exist. Use the exact selected versions from the [technology specification](tech_stack.md), from repository root:

```sh
export GOTOOLCHAIN=local
test "$(go env GOVERSION)" = "go$(cat .go-version)"
```

Do not silently download another toolchain, install additional test tooling, or claim an unavailable check ran. [AGENTS.md](../AGENTS.md#environment-commands) remains the command quick reference.

| Stage | Requirement | Commands / evidence |
| --- | --- | --- |
| Documentation-only | **Mandatory:** whitespace, changed Markdown/link validation, ownership/references, scope and checklist-state review. No Go checks required. | `git diff --check`; also inspect new untracked documents, local link targets/fragments, fences/tables, and the actual diff. |
| During development | **Mandatory for changed behavior:** targeted package cases and relevant boundary tests; table cases for ordinary edges; regression-first workflow for fixes. | For example, `go test ./internal/dataset ./internal/scorer` once those packages exist. Use `go test -count=1 ./internal/execution` for uncached execution evidence when appropriate. |
| Before merging meaningful Go changes | **Mandatory:** toolchain verification, full ordinary offline suite, vet, pinned formatting, and affected integration/contract/process cases. | `go test ./...`; `go vet ./...`; `files="$(go tool gofumpt -l cmd internal)" && test -z "$files"`. Format with `go tool gofumpt -w cmd internal` first when necessary. |
| Concurrency / cancellation / lifecycle changes | **Conditional mandatory:** race detector on a supported platform/C toolchain. | `go test -race ./...`. If unavailable locally, report the limitation and obtain supported-runner evidence; absence of a race report is not proof of leak freedom. |
| Executable/build wiring and milestone acceptance | **Conditional mandatory:** build the CLI and exercise the offline acceptance workflow. | `go build -o ./bin/evaldrome ./cmd/evaldrome`; use the implemented CLI contract, not invented flags. The v0.1 release checklist also requires the full race check on a supported environment. |
| Parser/invariant, scheduler, or compatibility risk | **Selective:** bounded fuzzing, additional cancellation/concurrency repetitions, and compatibility cases for the changed boundary. **Optional:** targeted mutation analysis. | Record package/target, time/parallelism/repetition budget, toolchain, seeds, and outcomes. Existing compatibility fixtures remain mandatory ordinary tests. |

Keep v0.1 deterministic component and small process tests in the ordinary suite so `go test ./...` covers them. If a later real dependency requires separate integration selection, document its exact command and prerequisites with that milestone; never hide required coverage behind an undocumented tag or silent skip. Stop broadening/repeating checks after relevant gates pass unless new changes or unresolved evidence justify it.

**Intended CI gates, when CI is introduced:** use the pinned toolchain/formatter, run ordinary offline tests, vet, formatting, and build; run race checks in a supported environment and relevant integration/contract cases. Keep bounded fuzz/stress/mutation jobs selective and real-provider smoke runs explicitly authorized. No workflow is created by this guide. The v0.4 CLI Action and v0.8 product regression gates retain their separate [milestone scope](experiments.md#ci-gates).

### Fuzzing and properties

Use native Go `testing.F` first. Bound input sizes and execution; never fuzz paid inference or unrestricted filesystem/network operations. Seed with ordinary valid, invalid, and regression examples. Minimize discovered failures, retain useful corpus entries under the package's `testdata/fuzz/`, and add a readable deterministic regression when it clarifies the defect.

Useful properties include:

- Dataset parsing terminates without panic for bounded arbitrary bytes, rejects the specified invalid categories, and preserves accepted string contents. Fixed valid byte fixtures have stable independently known identity; do not claim reformatting leaves the raw-byte hash unchanged.
- Exact match is reflexive for admissible strings; appending a nonempty suffix to only one equal string creates a mismatch. Literal whitespace/Unicode examples still provide independent anchors; do not merely reimplement equality to calculate every expected result.
- Permuting completion order leaves dataset-ordered results and aggregate scores unchanged. Adding an operational failure leaves existing scored results and their scored-case denominator intact while exposing the extra failure/coverage gap.
- Supported artifact serialization preserves all contractual values, including unavailable versus known-zero usage; fixed schema fixtures supplement round trips.

Once a fuzz target exists, substitute its actual package and target in this **command template**: `go test <package> -run='^$' -fuzz='^<target>$' -fuzztime=30s -parallel=2`. The budget is an example, not a product limit; report the actual settings. No fuzz targets exist yet. For selective scheduler stress after bootstrap, an example is `go test -race -count=20 ./internal/execution ./internal/benchmark`; repetitions supplement explicit synchronization, not replace it.

### Mutation testing — deferred tool selection

Do not install mutation tooling for this documentation milestone. Once scoring, validation, coverage accounting, and result classification exist:

1. Select a maintained tool compatible with the pinned Go version and module layout. Require package/test filtering, useful source-level diffs, reproducible configuration, bounded runtime/parallelism, and clear reporting of survived, killed, timed-out, and invalid/non-compiling mutations. Avoid a tool that requires production restructuring or live providers.
2. Establish a passing deterministic baseline. Use a disposable checkout and restrict mutations to the chosen critical logic. Start with equality/normalization, required-field/duplicate rejection, scored denominators, unknown-to-zero conversion, or status precedence.
3. Run the relevant tests against each mutation. Inspect why a test fails; a compilation failure or unrelated timeout is not evidence of a strong behavioral assertion.
4. Investigate surviving mutants: a missing assertion/case, equivalent behavior, unreachable code, or an unclear contract may explain them. Add an independent behavioral case only where justified; a survivor is not automatic proof of a product defect.
5. Record scope, tool/version/configuration, investigated survivors, and resulting improvements. Verify the original code and relevant tests afterward. Do not optimize for a universal mutation-score threshold.

## First-slice contract decisions

These refine the [architecture's unresolved contracts](../ARCHITECTURE.md#contract-decisions-required-before-the-first-cli-slice-ships); they are questions, not selected defaults. Record answers in focused dataset/benchmark, provider execution, artifact, and CLI specifications as those boundaries are implemented. Keep detailed product semantics in those owners and update this matrix with references, rather than maintaining a second specification here.

| Ref | Finalize before dependent tests / slice ships | Intended contract owner |
| --- | --- | --- |
| D | Dataset byte/record limits; blank/whitespace-only lines; unknown fields and duplicate JSON members; BOM, line endings, final newline, and Unicode escape handling; empty `id`/`input`/`expected` admission; location/error ordering; digest encoding. Raw immutable-byte SHA-256 and required string fields are already decided. | Dataset portion of the benchmark specification. |
| P | First provider/model/API and supported options; timeout/concurrency defaults; stable error classes/codes; attempt admission/accounting and partial responses/usage; caller deadline versus request timeout; competing cancellation/error causes. The provider execution specification must make shared adapter contract cases possible. | Provider execution specification; technology specification for dependency pins. |
| B | Exact meaning/overlap of completed/scored/failed/canceled/unstarted; usable output when scoring fails; score representation/rounding; cancellation versus terminal-completion precedence and observable admission boundaries. Scored-case denominators, unavailable empty scores, and retained completed work are already decided. | Benchmark/execution specifications at their respective boundaries. |
| A | Schema version/fields, required metadata and nullable values, run IDs, error serialization, supported-reader/unknown-field/version behavior, staged-file cleanup/failure guarantees, and compatibility fixtures. Atomic finalization and no silent overwrite are already decided. | Artifact specification owned by storage. |
| C | CLI flags/defaults, output paths and overwrite opt-in, stdout/stderr and machine-readable contracts, exit/error/signal mapping, bounded cancellation cleanup, credential sources, and reporting after artifact failure. | CLI contract; reuse benchmark/execution classifications. |

Recommended first implementation sequence: finalize the contracts needed for one fake-provider path and its independent fixtures; bootstrap pinned tooling; implement loader/scorer, execution/runner, storage, and CLI in small tested slices. Prioritize invalid-before-inference, exact matching, score/coverage accounting, cancellation/retained results, and artifact-failure truthfulness. Add the first real adapter with local HTTP contract tests after the offline path works; obtain the separately authorized smoke evidence required for release.

## Later milestones — testing requirements only

### v0.2: experiments and comparisons

Follow [experiments](experiments.md). Test repeatability with frozen inputs/settings/scorer versions and complete provenance, supported artifact compatibility, independently calculated score/coverage comparisons, and comparison without inference. Exercise classified retries, attempt/usage visibility, bounded cancellable backoff, and actual capabilities through controlled adapters. Test Braintrust disabled/unavailable states, payload permissions, and preservation of local results at its adapter boundary.

### v0.3–v0.5: reviews and GitHub

Follow [reviews](reviews.md). Use fixed buggy and bug-free repositories with independent defect evidence and labels kept out of reviewer prompts. Test malformed/adversarial/model/prompt-injection content, location/evidence validation, deterministic filtering/deduplication, and explicit missing coverage. Verify read-only behavior, snapshot/path boundaries, disabled hooks/scripts/model commands, and secret isolation. Valid structure or agreement between models never proves a finding correct; keep model suggestions and independent verification distinct.

At the publishing boundary, test explicit authorization, separation of publishing credentials from review workers, artifact provenance, stale-head rejection, idempotency, and failure without loss of review evidence using local GitHub fakes. Test review-comparison denominators and unresolved findings with hand-checked cases. Product contracts and publication gates remain in the review document and release checklist.

### v0.6+: hosted service

Follow [service](service.md). Use real disposable SQLite databases for migrations, constraints, transactional admission, authorization/workspace isolation at HTTP and storage boundaries, and durable job lifecycle/restart/recovery. Test HTTP API contracts, disconnect versus durable cancellation, ambiguous attempts without automatic billable replay, artifact/database reconciliation, and backup/restore. Introduce Testcontainers only if an actual containerized dependency adds necessary confidence; SQLite does not itself require it.

Once a web UI exists, persist critical login, submission/cancellation, artifact access, and workspace/role journeys as Playwright Test suites suitable for CI. Exploratory Playwright MCP sessions can discover issues and inform test cases but are not persistent suites or CI evidence. Defer browser tooling, service infrastructure, and these implementation tests to their owning milestones.

## Evidence-based completion report

Report observed results separately from assumptions. A concise development report must include:

```text
Behavior/contracts exercised:
Tests added/changed (paths and cases; or none with reason):
Commands actually executed and pass/fail outcomes:
Bug regression: original failure/reason and final result (or not applicable/exemption):
Untested behavior, skipped/blocked checks, and environmental limitations:
Unresolved risks or contract questions and next action:
```

Include relevant fixture/build/tool identities when needed to reproduce the evidence; release acceptance uses the [milestone evidence record](../docs/release-checklist.md#how-to-use-this-checklist). Distinguish executed tests from cached results, unrun command recommendations, and expected outcomes. Never claim a test passed because it was written, never silently omit a failure, and never mark a release gate complete without implementation and execution evidence.
