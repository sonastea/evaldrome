# Self-Hosted Service and Operations

**Status: planned.** Read for v0.6 personal hosting, v0.7 multi-user hosting, v0.8 operations, or service-related boundary changes. The service is required for the stable product but optional to run. Do not introduce it into v0.1.

[ARCHITECTURE.md](../ARCHITECTURE.md) owns package boundaries and common outcome semantics. The [release checklist](../docs/release-checklist.md) owns milestone gates and evidence.

## Deployment and execution boundary

```text
Browser / authenticated API client
    -> HTTPS endpoint
    -> HTTP authentication + workspace authorization
    -> SQLite durable queue
    -> bounded workers calling Go engines directly
    -> private artifacts + durable terminal metadata
```

- Use one active Go server per persistent data directory on a Linux host. Reject a second active process. SQLite must use local storage; network-mounted databases, multiple replicas, and distributed workers are unsupported.
- Provide a Linux binary/service-manager example and a non-root OCI container/Compose example. Keep data and separately mounted operator secrets distinct.
- Bind to loopback by default. Require explicit remote binding and HTTPS, normally through a reverse proxy. Trust forwarding headers only from configured proxies.
- Prefer Go-rendered HTML and embedded assets with small progressive enhancements. Keep the embed owner, templates, and assets under `internal/server/web/`.
- Keep HTTP, authentication, jobs, quotas, and SQLite outside workload/provider/scorer contracts. Workers call Go engines directly; do not shell out to the CLI or duplicate its workload behavior.
- Document supported CPU architectures, resource requirements, volume permissions, endpoints, and tested capacity with release measurements. No PostgreSQL, Redis, external identity, or Kubernetes is required.
- Instance operators control the host and can access its data. Application workspace isolation does not isolate users from the operator; use separate deployments when that trust is unacceptable.

## Identity, sessions, and roles

- Start with local accounts and one private workspace at v0.6. Add invite-only membership, roles, and scoped API tokens at v0.7.
- Bootstrap the first operator locally with an administrative command or single-use local mechanism. Never expose a public first-user-wins setup endpoint.
- Hash passwords using maintained Argon2id with unique salts. Store opaque session/token hashes, not recoverable passwords or bearer values.
- Use Secure/HttpOnly/SameSite cookies, session expiry/revocation, CSRF protection for cookie-authenticated mutations, and login throttling.
- Invitations are single-use, expiring, and bound to intended identity/workspace/role. Permit out-of-band link delivery without email infrastructure. Define operator-assisted account recovery; password resets revoke prior sessions.
- API tokens are revocable, expiring, workspace-scoped, and permission-limited. Enforce the issuing user's current membership/role as well as token scope.

| Role | Permissions |
| --- | --- |
| Viewer | Read authorized workspace inputs, runs, findings, comparisons, and exports |
| Member | Viewer rights; add inputs/comparisons/verifications; submit and cancel own runs |
| Workspace admin | Member rights; manage membership, cancel workspace runs, and manage retention within operator limits |
| Instance operator | Bootstrap accounts/workspaces, provision secrets/integrations, set ceilings, and administer the host |

Publishing requires an explicit operator workspace grant plus an authorized role/token. Workspace admins cannot raise operator ceilings or grant unassigned credentials.

## Authorization and resource ownership

- Scope datasets, repository registrations, policies, provider grants, runs, artifacts, comparisons, verification history, and associated audit events to a workspace. Keep instance account/bootstrap audits operator-only.
- Derive the actor from authentication. Validate membership for every object lookup, log, page, comparison reference, and download; a supplied workspace ID is not authorization.
- Scope queries and file keys at storage boundaries, not only HTTP handlers. Cross-workspace references must not disclose whether private objects exist. Avoid cross-workspace content caches initially.
- Membership removal immediately revokes access, cancels queued work, and requests cancellation of active work. Recheck membership/grants/limits at dispatch and before additional provider requests.
- The basic UI covers authorized submission, status/cancellation, results/findings, comparisons, exports, verification, and membership administration as their milestones arrive.
- Imported artifacts cannot grant access to their claimed workspace or credentials. Trusted service metadata owns authorization; exported workload artifacts remain readable without the database.

## Inputs, credentials, and outbound access

| Boundary | Required behavior |
| --- | --- |
| Input references | Remote requests select authorized immutable uploads or registered repositories, never arbitrary host paths |
| Review admission | Resolve requested refs to pinned commits before enqueue; record accepted input/configuration versions |
| Uploads | Bound bytes/count; stage and publish atomically under server-generated IDs; reject traversal and unsafe archive/symlink entries |
| Work directories | Isolate each run; bound read-only Git operations; disable hooks/helpers/automatic submodule execution |
| Rendering/downloads | Escape repository/model text; keep files outside web root; authorize downloads and use safe attachment/type headers with sniffing disabled |
| Endpoints | Operator registers allowed repositories/provider/tracking endpoints; users select authorized configurations |
| Network policy | Validate destinations at connection and redirects; protect against DNS rebinding, instance metadata, and unrelated internal services |
| Local inference | Permit private endpoints only through explicit operator configuration |
| Secrets | Operator environment variables or read-only files, resolved through named workspace grants; no UI secret upload/storage in the first stable service |

Keep repository-read, provider, tracking, and publishing credentials distinct. Subprocess environments contain no provider/publishing secrets. Users cannot select arbitrary environment-variable names, inherit operator defaults, or fall back to another workspace's key.

Resolve credentials only for authorized operations. SQLite stores authorized secret references; logs, exported configuration, and artifacts contain neither raw secrets nor host secret paths. Raw-content tracking export is disabled unless explicitly enabled by the workspace's permitted policy.

Cloud inference leaves the host. A separately tested local-provider path with remote tracking disabled must work without mandatory hosted identity or application telemetry.

## Durable jobs and cancellation

1. Authenticate/authorize every referenced resource; freeze accepted inputs/options.
2. Persist the run specification, actor/workspace, idempotency record, and queued job transactionally before acknowledging submission.
3. Claim work transactionally and create a job-scoped engine context independent of the HTTP connection.
4. Checkpoint completed units and attempt metadata. Recheck grants and limits before additional requests.
5. Finalize durable artifact files before exposing their references and terminal job state. Reconcile files and metadata after crashes.

Queued jobs survive restart. Closing a browser does not cancel a job. Cancellation requests are durable: cancel queued work, propagate cancellation to active requests/waits, and retain completed output.

Interrupted running jobs become explicitly partial or failed unless a fully finalized result can be recovered; honor recorded cancellation where applicable. Never silently restart inference or invent success. A user-requested rerun receives a new run ID linked to its predecessor.

Submission idempotency does not make provider execution exactly-once. A crash after dispatch can leave an unknown response/charge. Record ambiguous attempts and unknown usage; do not automatically replay without a provider guarantee or explicit rerun. Keep inference retries, cancellation, and publishing retries separate.

On graceful shutdown, stop admission/claiming, drain for a bounded interval, then cancel remaining contexts and persist interruption status. Storage failure or exhausted disk stops admission and degrades readiness; do not acknowledge unpersisted work.

## Planned HTTP contract

Use `/api/v1` and document schemas in OpenAPI as endpoints are implemented. These are planned resources, not available commands or routes.

| Endpoint | Behavior |
| --- | --- |
| `POST /api/v1/workspaces/{workspace}/datasets` | Validate immutable upload; return `201` and ID |
| `GET /api/v1/workspaces/{workspace}/runs` | Authorized, bounded cursor-paginated history |
| `POST /api/v1/workspaces/{workspace}/runs` | Validate and durably enqueue; return `202`, run ID, and `Location` |
| `GET /api/v1/workspaces/{workspace}/runs/{run}` | Status, coverage, errors, gate outcome, artifact references |
| `POST /api/v1/workspaces/{workspace}/runs/{run}/cancellation` | Idempotent cancellation request; report existing terminal state |
| `GET /api/v1/workspaces/{workspace}/runs/{run}/artifacts/{artifact}` | Authorized download |
| `POST /api/v1/workspaces/{workspace}/comparisons` | Compare compatible saved runs without inference |

- Apply the same rules to membership, registration, verification, and explicit publication endpoints. Polling suffices initially; streaming and remote CLI support are not stable prerequisites.
- Require `Idempotency-Key` for run submission, scoped to actor/workspace/operation and canonical request hash. Identical retries return the same run; conflicting reuse returns `409`. Retain keys at least as long as run metadata and document expiry.
- Bound request bodies, queries, downloads, and all collections. Use stable error codes, safe messages, and request IDs.
- Distinguish unauthenticated `401`, forbidden `403` or non-disclosing `404`, invalid configuration `422`, payload-too-large `413`, quota/rate-limit `429`, and temporary-unavailable `503` responses. Do not expose raw internal errors or secrets.

## Fairness, quotas, and cost

- Enforce instance-wide and per-workspace limits for queued/running jobs, in-flight requests, attempts, tokens, context, elapsed time, uploads, and storage.
- Share the instance-wide provider limiter across workers. A per-job semaphore alone is insufficient. Give eligible workspaces progress rather than allowing one to monopolize the queue.
- Reserve capacity transactionally at admission/dispatch and reconcile concurrent usage. Workspace limits cannot exceed operator ceilings.
- If estimated-cost admission is required, reject configurations without usable pricing/output bounds. Unknown cost cannot bypass a required limit.
- Report all known attempt usage and uncertainty. Estimated spend guards are not guaranteed invoice caps; provider-enforced hard limits provide that guarantee.

## Persistence, migrations, and recovery

| Storage | Owns |
| --- | --- |
| SQLite in `internal/server/store/` | Accounts/hashes, membership/grants, input metadata, jobs/attempts, idempotency, artifact metadata, verification revisions, audit |
| Private workspace files | Large immutable datasets, snapshots, raw results, exports, staged writes |
| Operator-controlled secret storage | Provider/tracking/repository/publishing secrets outside ordinary exports and database |

- Use foreign keys, transactions, versioned migrations, bounded access, and explicit journal/busy-timeout settings. WAL is appropriate for one host; test its checkpoint and backup behavior.
- Never hold database write transactions during model calls. Enforce workspace relationships at storage boundaries.
- Stage and atomically finalize artifacts with hashes; expose database references only after file durability. Detect missing/corrupt/staged/orphaned files and reconcile without inventing successful results.
- Verify integrity before execution or publishing where corruption affects correctness. Hash integrity does not replace trusted authorization or publishing provenance.
- Keep external secrets out of SQLite. Store account/password/session/invitation/API-token hashes as appropriate, never plaintext passwords or bearer values.
- Version database migrations independently from portable artifact schemas. Define the migration command when implementing the service, not during CLI bootstrap.

### Backup and upgrade

- Initially support a consistent maintenance-window backup: stop admission, drain or explicitly interrupt jobs, stop cleanly, and back up the database plus referenced files together.
- Support live backup only with coordinated SQLite backup and file consistency. Copying a live `.db` alone is insufficient. Document permissions, operator-managed encryption, and separately recovered secrets.
- Restore onto a fresh host and verify accounts/access, job states, hashes, findings, and comparisons. Keep dispatch and outbound integrations disabled until the operator re-enables them.
- Record observed recovery time and backup recovery point. Do not promise untested availability.
- Back up before upgrading, reject unsupported schemas, and test the supported migration path. Initial rollback may restore the full matching pre-upgrade backup; explicitly document loss of subsequent writes.
- Test release-candidate-to-stable upgrades, then each supported stable-minor path. Publish compatibility/support windows and rollback procedures.

### Retention and deletion

- Set operator ceilings and retention for inputs, results, logs, and audit history. Workspace admins may shorten retention within those limits.
- Protect active jobs and references from retained comparisons, baselines, and verifications during cleanup.
- Explicit deletion marks dependent views unavailable rather than rewriting historical scores. Document separately retained backups and opted-in tracking copies and their deletion procedures.
- Optional local SQLite history remains a separate future concern; standalone/CI correctness never depends on service or local-history databases.

## Operations and release evidence

- Provide bounded redacted structured logs, authenticated metrics, minimal health/readiness signals, and authorized audit access. Health responses disclose no private data.
- Audit actor/workspace/action/target/time/outcome for memberships, grants, submissions, cancellations, verification revisions, deletions, and publication. Do not duplicate private prompts or keys in audit records.
- Test two-workspace isolation, credential selection/revocation, input/path/network boundaries, idempotency, restart, cancellation, quotas, artifacts, and backup/restore with fake providers.
- The operational scenario uses at least 100 mixed benchmark/review jobs, 10 clients, three workspaces, four global and two per-workspace in-flight requests maximum, injected rate limits/timeouts, cancellation, and restart.
- Record hardware, input/context sizes, queue latency, peak memory/disk, fairness, and terminal outcomes. Exercise disk-full, database-busy, corrupt/missing artifacts, provider outage, and tracking outage.
- Run at least seven consecutive pilot days with two non-operator accounts and private/shared workspaces before stable release. Require an independent clean-install walkthrough, fresh-host restore, and upgrade rehearsal.
- Document installation, HTTPS, bootstrap/invitations, provider grants, exports/deletion, quotas, migrations, backup/recovery, and troubleshooting. Mark acceptance only with the evidence required by the release checklist.
