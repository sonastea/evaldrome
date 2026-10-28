# Technology Stack — Bootstrap Specification

**Status: specification, not an installed or locked repository toolchain.** The repository has no `go.mod`, `go.sum`, or `.go-version` yet. Read this file when bootstrapping Go, changing dependencies, or updating development tooling.

## Verified bootstrap baseline

- **Go: `1.27.3`.** Observed locally with `go version` during this audit on 2026-10-27 (`darwin/arm64`). Adopt this exact development/CI version at bootstrap; this is not a claim about the latest upstream release.
- **Module path: `github.com/sonastea/evaldrome`.** Derived from the configured Git origin. Use one module at repository root.
- **Dependency manager: Go modules.** Commit `go.mod` and `go.sum`. Do not introduce a workspace or another package manager for the initial CLI.
- **Repository formatter: `mvdan.cc/gofumpt v0.12.0`.** Verified with `gofumpt -version` and executable module metadata. The installed binary was built with Go `1.27.1`; that build provenance is not a second project runtime requirement.
- **Test/vet/standard library: Go `1.27.3`.** Use `testing`, `go test`, the race detector, `go vet`, and `gofmt` from the selected toolchain. Race tests require a supported platform and C toolchain.
- **Initial CLI/HTTP/serialization/logging libraries: standard library.** Prefer `flag`, `net/http`, `encoding/json`, `context`, and `log/slog`. No external CLI, web, ORM, or dependency-injection framework is selected.
- **Initial data formats: UTF-8 JSONL inputs and versioned JSON outputs.** Use SHA-256 for dataset byte identity. YAML suite configuration belongs to v0.2; no YAML library is selected.
- **Current dependencies beyond development tooling: none selected.** A real-provider SDK is selected and pinned when that adapter is implemented.

## Bootstrap pinning requirements

- Create `.go-version` containing `1.27.3`. Align the initial `go` directive and `toolchain go1.27.3` in `go.mod` with that baseline.
- Treat the `go` directive as a minimum requirement, not an exact toolchain lock. Set `GOTOOLCHAIN=local` for repository checks and verify `go env GOVERSION` against `.go-version`; do not rely on silent toolchain downloads.
- Register the formatter with `go get -tool mvdan.cc/gofumpt@v0.12.0` after module initialization. Commit its tool entry, requirement, and checksums. Invoke it through `go tool gofumpt`, not an arbitrary global executable.
- Verify the resolved formatter version with `go list -m mvdan.cc/gofumpt`. Dependency changes can affect module version selection; the tool entry alone is not a version pin.
- Use the same Go/formatter versions locally and in automated checks. Update this specification, manifests, and check configuration together when intentionally upgrading.
- Use exact module versions when adding dependencies. Never use `@latest` in reproducible setup instructions. Retain `go.sum`; do not treat checksums as a substitute for compatibility/security review.
- Check the commands in [AGENTS.md](../AGENTS.md) after bootstrap. Add the implemented quickstart to `README.md`. Do not mark the release tooling gate complete until these checks pass.

## Deferred selections

- **Real provider, v0.1:** vendor, SDK/module version, API endpoint/version, model identifier, and supported options are unselected. Verify against the chosen provider before implementation; example model names are not valid defaults.
- **Braintrust, v0.2:** use the Go SDK where practical behind `internal/braintrust`. Exact module/version and required capabilities remain unselected. Remote tracking stays optional.
- **SQLite, v0.6:** required only for service state. Driver, embedded/system SQLite version, CGO policy, and migration runner remain unselected. Pin and test these together with backup/WAL behavior before service implementation.
- **Authentication, v0.6-v0.7:** local accounts, Argon2id password hashing, opaque sessions, and scoped tokens are planned. Select maintained Go dependencies and exact versions at implementation. No hosted identity framework is selected.
- **Web UI, v0.6:** prefer Go templates, embedded assets, and small progressive enhancements. No Node.js runtime, TypeScript compiler, frontend framework, or JavaScript package manager is selected.
- **Deployment, v0.6:** Linux binary and non-root OCI container are planned. Pin base-image digests and document supported architectures when packaging exists. Compose/service-manager versions require release validation.
- **Migrations:** no command or executable exists. Specify the service migration command with its runner, schema policy, and recovery tests; do not install a migration framework during v0.1.

## Selection rules

- An unselected item is deferred, not permission to install a convenient default. Resolve only the choices needed by the active task and record the decision.
- Prefer the standard library and small dependencies with a concrete implementation benefit. Introduce TypeScript only for a documented advantage or a required hosted integration feature.
- Ignore patterns for npm, pnpm, Kubernetes, or Terraform do not select those technologies.
- Evaluated model-generated Go code defaults to the task's pinned Go toolchain and `gofmt`. Apply `gofumpt` only when the task/suite/target repository requires it; record formatter identity/version in execution metadata.
