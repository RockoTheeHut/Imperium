# PowerShell & C# Automation Platform
## Codex Implementation Plan — v2.0

**Source of truth:** `Automation_Platform_Spec_v4.md`
**Target:** Windows Server, SQL Server for production and clusters, and SQLite in WAL mode for single-instance standalone local development
**Implementation:** C#/.NET platform, PowerShell 5.1/7 and modern .NET C# workloads
**Approach:** API-first modular monolith, isolated workers, vertical features, explicit acceptance gates
**Scope decision:** Manual migration of existing PowerShell Universal workloads; no automatic importer

---

## 1. Confirmed decisions and architectural invariants

1. **C# workloads use modern .NET only.** Older installed modern .NET runtimes may be selected where compatible; .NET Framework 4.x is excluded. The platform host's .NET version is independent of workload versions.
2. **EF Core with SQL Server and SQLite.** Use SQL Server for production and every cluster. Offer SQLite in WAL mode as an explicit provider for single-instance standalone local development. Maintain separate version-controlled EF Core migration histories for the two providers.
3. **Controlled upgrades.** Apply migrations using an authorized install/upgrade identity and a single coordinated migration process per cluster. Do not let every service node attempt `Database.Migrate()` on startup. Use expand/contract schema migrations or maintenance mode for incompatible changes.
4. **Automatic mandatory auditing.** All persistent configuration changes and every hosted endpoint invocation must generate protected audit events. This also covers denied/failed attempts where applicable.
5. **Asynchronous customizable logging.** Named, selectable logging targets have configurable schemas/columns and destinations. Required sinks are SQL tables and files; extensions and multi-destination fan-out are designed in.
6. **Script-friendly logging SDKs.** Hosted PowerShell gets `Write-AutomationLog`; C# gets an approved `AutomationLogger`/`IAutomationLogger` library with `LogAsync`.
7. **Audit is not optional telemetry.** Turning off metrics or a user-configured log target cannot turn off required configuration, authorization, secret, or endpoint audits.
8. **Everything management-related is API-first.** The web UI calls the same authorized application services as the REST API; all UI management operations have API equivalents.
9. **Workloads never run inside the privileged host process.** Both PowerShell and C# execute under supervised, isolated processes and authorized Windows identities.
10. **Immutable execution provenance.** Each execution or audit record identifies its source/environment revision, request/caller, effective execution identity, and node.
11. **Standalone first, cluster-safe foundations.** Both providers preserve durable state and atomic claims for standalone execution. SQL Server also implements distributed leases and fencing for four-node deployments.
12. **No automatic PowerShell Universal importer.** Migration documentation and manual validation are required, not importer code.

### Suggested components

- `Platform.Host`: .NET Generic Host + Windows Service + ASP.NET Core.
- `Platform.Domain` / `Platform.Application`: resource objects, commands, policy, events and authorization abstractions.
- `Platform.Persistence`: EF Core model, SQL Server and SQLite mappings and migrations, provider-specific claim operations, transactions, durable outbox.
- `Platform.Security`: AD/LDAP, Entra OIDC, Kerberos, tokens, RBAC, secret authorization.
- `Platform.Runtime`: runtime discovery, environment compatibility, worker contract.
- `Platform.WorkerSupervisor`: launch, IPC, credentials, job objects/resource limits, process lifecycle.
- `Platform.PowerShellWorker`, `Platform.CSharpWorker`: language-specific user-code processes.
- `Platform.Compiler`: Roslyn, dependencies, cache manifests, validation.
- `Platform.Jobs`, `Platform.Scheduling`, `Platform.Events`: durable execution, scheduling, triggers.
- `Platform.Endpoints`: published custom HTTP routing and invocation auditing.
- `Platform.Logging`: target definitions, routers, serializers, sinks, spool, auditing boundary.
- `Platform.PowerShellSdk`, `Platform.CSharpSdk`: ergonomic in-script logging and execution context APIs.
- `Platform.Sources`, `Platform.Deployments`, `Platform.Cluster`: source ownership, Git, immutable rollout, node coordination.
- `Platform.AdminUi`: browser UI and editor.
- `Platform.Observability`: dashboards, metrics, alerting, audit queries, export.

These are logical projects/modules; they need not be independent Windows services.

---

## 2. Cross-cutting engineering contracts (apply from the beginning)

**Persisted entities:** stable IDs, concurrency tokens, UTC times, audit metadata, soft-delete or explicit deletion rules, source revisions, and RBAC scope. Use shared EF Core entity definitions with provider-specific mappings as needed. Use application-managed concurrency tokens on SQLite and preserve the same stale-write semantics on both providers.

**Mutation path:** authenticate -> authorize -> validate -> execute transaction -> change state + append audit/outbox event in the same transaction -> commit -> asynchronously deliver to authorized destinations. For attempted/denied changes that have no state transaction, persist the security audit separately. If the protected change cannot be durably audited, reject it.

**Endpoint audit:** capture an immutable request execution event for every matched hosted custom endpoint call, including authentication/rate-limit failures. Record client IP only through validated trusted-proxy handling; do not trust arbitrary `X-Forwarded-For` headers.

**Async user logging:** producer -> bounded local authenticated worker IPC -> target routing -> durable queue/spool as required -> background batch writer -> destination(s). Provide retry with bounded backoff, at-least-once/idempotent delivery, drop/failure policies, backpressure and dead-letter visibility. A producer is told whether the event was accepted; do not promise durability for an in-memory-only acknowledgment.

**Sensitive fields:** default-deny body capture; no credentials, secret values, token values, raw authorization headers or unredacted payloads in logs/audits. Log access uses RBAC. Sensitive operations use separate mandatory audit retention.

**Worker identity and isolation:** the caller identity, resource authorization, and executing Windows account are different concepts. Access to named logging targets and secrets must be validated by the host; workload code never receives the privileged database connection or sink credentials.

**Testing:** each phase closes with unit tests, Windows integration tests against the applicable database providers, security-negative tests, and explicit documented acceptance demonstrations. Track all v4 requirement IDs and 21 acceptance gates.

---

## 3. Phased implementation

### Phase 0 — Architecture and high-risk proof of concept

**Objective:** Prove the risky execution, persistence, authentication, auditing, and logging boundaries before building production features.

**Steps**

- **P0.1 Repository:** establish solution projects, branch/PR rules, `SPEC.md`, `IMPLEMENTATION_PLAN.md`, `AGENTS.md`, architecture decisions, requirement matrix, Windows CI test fixtures.
- **P0.2 Domain boundaries:** design versioned resource IDs, configuration mutation path, audit outbox, immutable published source, execution context, runtime environment contract, API versioning and RBAC policy engine.
- **P0.3 Runtime prototype:** execute PowerShell 5.1 and two installed PowerShell 7 versions in separate processes; compile C# for two supported modern .NET targets and execute them independently of the host runtime.
- **P0.4 Windows identity prototype:** start an unprivileged worker with a stored run-as credential in a protected proof-of-concept store, capture identity, enforce timeouts, kill worker process tree, and show the host survives.
- **P0.5 Persistence prototype:** model two entities with EF Core and apply initial migrations for both providers. Verify SQLite enters WAL mode and test concurrent version updates, atomic configuration and audit writes, and job claim contention on both providers.
- **P0.6 Logging prototype:** send a structured worker log through authenticated IPC, write to a parameterized SQL table sink and a JSONL file sink asynchronously, then test destination outage/retry.
- **P0.7 Language tooling prototype:** verify Monaco/PowerShell language-service integration and C# diagnostics for the supported runtime matrix.
- **P0.8 Risk and threat analysis:** document privileged boundaries, key management, log destination controls, SQL outage behavior, identity handling and cluster consistency design.

**Deliverables:** solution architecture, decisions, security threat model, functioning prototypes, Windows test environment with SQL Server and local SQLite fixtures, requirement traceability.

**Exit criteria:** cross-runtime execution, host/worker isolation, EF migrations, durable audit linkage, and multi-sink logging are all demonstrated. No unresolved blocker to the chosen architecture.

### Phase 1 — Windows host, EF Core database, and management foundations

**Objective:** Run a secure, persistent, API-based Windows application with a reproducible database lifecycle.

**Steps**

- **P1.1 Host:** .NET Generic Host, Windows Service, ASP.NET Core, configuration, HTTPS, shutdown/restart, service health and readiness.
- **P1.2 EF Core model:** shared entities, indexes, relationships and test seeding, with provider-specific mappings, value converters, and concurrency tokens where required. Keep a separate migrations assembly and model snapshot for each provider.
- **P1.3 Migrations:** check in both migration histories. Provide a SQL Server install/upgrade bundle or command for the authorized deployment identity and a local SQLite setup, reset, and upgrade command. Apply SQLite migrations with the application stopped and exclusive access to the file. Validate schema compatibility at host startup and document failure and recovery for each provider.
- **P1.4 Cluster migration safety:** single coordinator/installer applies changes, nodes verify required schema version; no unconditional EF migration calls by ordinary production nodes.
- **P1.5 Resource foundation:** object IDs, folders, tags, draft/published state, validation, immutable revision IDs, concurrency and per-object storage-mode placeholders.
- **P1.6 Management API:** versioned route conventions, typed DTOs, OpenAPI, CRUD, pagination, filtering, structured errors, correlation IDs, authorization hooks.
- **P1.7 Audit foundation:** immutable audit/event entity models, transactional outbox, actor/operation/object context, redaction framework, append-only write permissions and basic audit query API.
- **P1.8 Admin UI shell:** minimal layout, API integration, system health screen, configuration views; no dev-only bypass in production packaging.
- **P1.9 SQLite local development (§10.5):** require explicit provider selection and a writable local database path. Reject SQLite for production and any cluster configuration, including one-node clusters. Hold an exclusive file lock for the lifetime of the application instance or migration command, using a consistent database path. Initialize and verify WAL before starting HTTP or background services, enable foreign keys on every connection, and use short write transactions with a bounded busy timeout and actionable diagnostics. Document a SQL Server-free quickstart with file logging and WAL-safe backup and restore.

**Exit criteria:** create and upgrade databases on both providers, modify an object via API, observe a durable audit event, and restart without data loss. Verify that the local profile runs without SQL Server and rejects incompatible schemas, invalid deployment or filesystem configurations, a second instance targeting the same file, and WAL activation failure.

### Phase 2 — Authentication, granular RBAC, and mandatory configuration auditing

**Objective:** Enforce security and record every state-changing action before exposing arbitrary code execution.

**Steps**

- **P2.1 Authentication:** AD/LDAP, nested groups, Entra OIDC, Windows Negotiate/Kerberos, identity-provider health and outage behavior.
- **P2.2 Token principals:** user-bound tokens, independent service principals, scoped tokens, rotation/revocation, hashed protected token material.
- **P2.3 RBAC:** nested roles, cycle detection, explicit deny/allow precedence, folders, per-object discover/view/edit/execute/deploy/use-identity/use-secret/view-audit permissions.
- **P2.4 Security boundary:** same effective-authorization engine used by UI, REST endpoints, Git/deploy/background code and workers; reject impersonation/identity escalation.
- **P2.5 Comprehensive mutation audit middleware and domain hooks:** automatically capture all successful and attempted/denied create/update/delete/enable/disable/publish/rollback/RBAC/token/runtime/Git/vault/logging-target changes from all origins.
- **P2.6 Audit schema:** actor ID/name/type, original/effective identity, method/origin, object ID/type/name, change reason, timestamp UTC, before/after redacted diff, revision IDs, host/node, client/proxy IP, request and correlation IDs, outcome and failure category.
- **P2.7 Audit guarantees:** commit business mutation and audit event atomically; fail protected changes when durability unavailable; independently audit denied attempts; append-only permissions, read/export RBAC and retention policy.
- **P2.8 Secret foundation:** encryption service, key separation, encrypted database-backed secret storage and authorized resolver; never record secret plaintext in audit diffs.

**Exit criteria:** changing any managed configuration through UI/API generates an accurate redacted audit; denied changes are visible; permission simulation explains authorization; audit persistence failure cannot silently permit unaudited protected mutation.

### Phase 3 — Runtime discovery, environments, isolated PowerShell workers

**Objective:** Run arbitrary PowerShell code under selected versions and Windows identities without risking the privileged host.

**Steps**

- **P3.1 Discovery:** enumerate WinPS 5.1, side-by-side PowerShell 7 executables and installed modern .NET runtimes; report exact versions and paths.
- **P3.2 Environments:** language, exact version, dependency/module paths, variables, working directory, default run-as, limits, revision and RBAC.
- **P3.3 Supervisor:** authenticated IPC, process launch, Windows job objects/resource limits, cancellation, timeouts, heartbeat, crash detection and tree cleanup.
- **P3.4 PowerShell workers:** parameter transport and output/error/warning/verbose/information stream capture, working directory/module isolation, per-execution context.
- **P3.5 Run-as:** encrypted credential resolution, Windows logon/profile handling, identity precedence, separate permission to use a run-as account, audit actual identity.
- **P3.6 Eligibility:** fail clearly for missing runtimes, missing run-as accounts, incompatible environment revisions or disabled nodes.

**Exit criteria:** multiple PowerShell versions run concurrently as distinct authorized accounts, worker failure is contained, output is captured, and environment compatibility errors are accurate.

### Phase 4 — C# compilation, runtime selection and compiled cache

**Objective:** Support C# jobs and endpoints with definition-time compilation and per-node reuse.

**Steps**

- **P4.1 Public contracts:** documented C# job and endpoint entry points, async cancellation, request/job context, explicit capabilities.
- **P4.2 Compiler:** Roslyn-based compilation, target frameworks/reference assemblies, compatible modern .NET target selection, language version, assemblies/packages, diagnostics.
- **P4.3 Compile on definition:** compile on create/edit/validate before publish; reject invalid revisions while preserving last published revision.
- **P4.4 Immutable cache:** source/environment/compiler/dependency-hashed artifact keys, local store, validity checks, cleanup, forced rebuild and capacity limits.
- **P4.5 Execution worker:** launch compiled code using configured runtime, authorized run-as identity, bounded resources, output/structured results and cancellation.
- **P4.6 Node readiness contract:** report runtime, dependency and compiled artifact eligibility by revision; only ready nodes may serve that revision.

**Exit criteria:** create C# job and endpoint, compile at definition time, reuse cache on repeated execution, reject an invalid update, run under two modern .NET target versions, survive worker crash.

### Phase 5 — Script definitions, durable jobs and initial execution UI

**Objective:** Deliver the first usable automation product: scripts with history, permissions, parameters and durable execution.

**Steps**

- **P5.1 Scripts:** CRUD, source/draft revisions, language/environment, parameters, run-as, concurrency/retry settings, RBAC and audit hooks.
- **P5.2 Queue:** durable job records on both providers with atomic claims, a state machine, unique IDs, runtime/run-as matching, and attempt and correlation tracking. Use provider-specific claim operations where needed and verify SQLite claim contention and restart recovery.
- **P5.3 Execution:** ad hoc UI/API, parameter forms, target-node policy, authorization of execution and run-as separately.
- **P5.4 Output:** PowerShell streams, C# structured results, live output, historical search, output limits, redaction and retention.
- **P5.5 Reliability:** timeouts, cancellation, idempotency decisions, exponential/fixed retry, orphan cleanup, manual review for unsafe retries.
- **P5.6 UI:** script list/editor, execute dialog, job details, live/history output, retries and permissions.

**Exit criteria:** launch authorized PowerShell/C# jobs, capture correct provenance, recover state after restart, block unauthorized identity overrides, follow retry policy after worker loss.

**Milestone A:** first functional internal automation runner; not yet complete production replacement.

### Phase 6 — Custom asynchronous logging platform and language SDKs

**Objective:** Let script authors choose reusable logging configurations with customizable fields/columns and destinations, without sacrificing security or performance.

**Steps**

- **P6.1 Logging domain:** versioned `LoggingTarget` objects with stable IDs/names, RBAC, owner, enable state, default target inheritance by cluster/environment/folder/object and per-call override.
- **P6.2 Target configuration:** sink type, connection/file identity, schema/typed columns, mapping expressions, required fields, defaults, format, severity filters, redaction, rotation, retention, batch size, retry/backoff, durable/non-durable acceptance and overflow/failure policy.
- **P6.3 SQL sink:** configurable destination database/table, pre-existing table mapping or explicitly managed table creation, validated column names/types and parameterized inserts; constrain destinations and DDL privileges. Platform-owned tables stay under EF Core migrations; configurable user target tables use separate controlled provisioning or validated existing schemas.
- **P6.4 File sink:** append/rotate JSONL and CSV/delimited files, safe paths, permissions, encoding, file size/time rotation, retention, simultaneous-writer safety and cluster-local/shared destination semantics.
- **P6.5 Async transport:** authenticated worker-to-host logging IPC; bounded channels and spool; background batching; retries, circuit breaking, idempotent delivery IDs, dead letter and restart recovery; clear accepted/durable/persisted delivery semantics.
- **P6.6 PowerShell SDK:** create and automatically load `Write-AutomationLog`, accepting `-Target`, `-Level`, `-Message`, `-Properties`, optional `-EventId`, `-Exception` and an optional durability acknowledgment mode.
- **P6.7 C# SDK:** create versioned library with `IAutomationLogger` and `AutomationLogger.LogAsync`; inject logger via workload context; support structured fields and cancellation tokens on all selected modern .NET targets.
- **P6.8 Automatic context:** attach UTC timestamp, request/job/correlation ID, user/service principal, run-as, endpoint/script IDs, revision, runtime, node, severity and event ID unless explicitly restricted.
- **P6.9 Management:** CRUD/test/clone/disable targets; schema mapping designer/API; query log records, health, queue backlog, failed deliveries, retention, and target-use audits.
- **P6.10 Security:** separate target-use permission from management/view; prevent raw sink credentials and privileged DB access from escaping to workers; redact secrets; audit all logging target changes.

**Exit criteria:** two workers can log to different named SQL/file targets with distinct columns; PowerShell and C# APIs function; logging is asynchronous and bounded; sink outages are visible/recoverable; custom target disablement does not affect mandatory audits.

**Proposed PowerShell experience**

```powershell
Write-AutomationLog -Target 'InventorySql' -Level Information `
  -Message 'Inventory refresh completed' `
  -Properties @{ RowsUpdated = 125; Warehouse = 'East'; DurationMs = 410 }
```

**Proposed C# experience**

```csharp
await context.Logger.LogAsync(
    target: "InventorySql",
    level: AutomationLogLevel.Information,
    message: "Inventory refresh completed",
    properties: new Dictionary<string, object?>
    {
        ["RowsUpdated"] = 125,
        ["Warehouse"] = "East",
        ["DurationMs"] = 410
    },
    cancellationToken: context.CancellationToken);
```

The exact API signature is decided and documented before implementation; these examples express the desired developer ergonomics. `LogAsync` must not mean unsafe unobserved fire-and-forget. It should asynchronously acknowledge the selected acceptance policy.

### Phase 7 — Custom HTTP endpoints and complete endpoint auditing

**Objective:** Host PowerShell and C# HTTP endpoints with safe execution and automatic per-request records.

**Steps**

- **P7.1 Endpoint CRUD:** methods, parameterized routes, auth/RBAC, execution identity, environment/revision, rate limits, payload limits and response policy.
- **P7.2 Request/response contracts:** method, headers, cookies, route/query/body, user claims, cancellation, custom status/headers, JSON/text/binary/file return.
- **P7.3 Supervised execution:** resolve compiled C# cache, PowerShell worker launch, bounded concurrency/queue/reject, deadline propagation; no automatic HTTP invocation retry.
- **P7.4 Endpoint audit lifecycle:** automatically record one immutable invocation event for successful, denied, invalid, rate-limited, failed, cancelled and timed-out requests, including cases that never reach a worker.
- **P7.5 Endpoint audit fields:** UTC start/end, invocation ID, endpoint name/ID, route template and sanitized path, HTTP method, user/service principal or anonymous, auth scheme/token ID, client IP, trusted proxy chain, receiving server/IP, executing worker server/process, run-as Windows identity, source/env revisions, status code, duration, bytes, user agent, error category, correlation/trace IDs.
- **P7.6 IP and privacy:** trusted proxy configuration; exclude raw credentials, query secrets, bodies and cookies by default; RBAC and retention for queries/exports.
- **P7.7 Durable audit pipeline:** outbox/spool, idempotent writer and recoverable delivery; fail-closed on inability to accept required audit record under configured security policy.
- **P7.8 API docs/test UI:** custom endpoint OpenAPI, Swagger-compatible test page, management validation and request simulator.

**Exit criteria:** matching PowerShell/C# endpoints serve authenticated requests, return custom responses, obey per-node limits, log complete per-invocation audit including denial and client IP behind trusted proxy, and survive worker crashes.

**Milestone B:** scripting + custom API hosting + custom logging.

### Phase 8 — Scheduling and event triggers

**Objective:** Automate jobs independently of interactive users while remaining durable and cluster-ready.

**Steps**

- **P8.1 Schedule definitions:** one-time, interval, CRON, continuous loop, pause, time zone, params, identity/environment overrides.
- **P8.2 Durable occurrences:** database-backed schedule state, stable occurrence IDs, atomic claims, and singleton occurrence semantics. Use unique occurrence constraints and test concurrent scheduling on both providers.
- **P8.3 Policy:** retries/backoff, misfires, DST rules, missed runs, overlap, manual review, node selection, execution audits.
- **P8.4 Events:** job lifecycle, system/node health, login, API failure, deployment/Git events, durable delivery contract.
- **P8.5 Triggers:** filters, target scripts, context, identities, permissions, deduplication, recursion/storm guards, audits.

**Exit criteria:** scheduled jobs run unattended, duplicate claims are prevented, missed runs and trigger failures follow policy, event storms are bounded.

### Phase 9 — Variables, encrypted secrets and Vault providers

**Objective:** Complete typed configuration/secret services with secure retrieval by scheduled and interactive workloads.

**Steps**

- **P9.1 Typed variables:** JSON, list, scalar, scope hierarchy, deterministic resolution, revisions and RBAC.
- **P9.2 Local secrets:** encryption at rest, key management separated from routine database access, secret versions/rotation, use/reveal permissions, secure audit events.
- **P9.3 HashiCorp Vault:** AppRole, KV v2, TLS, namespaces/mounts, auth renewal, health, reference-only secrets.
- **P9.4 Per-secret caching:** no-cache mode, periodic background refresh, manual refresh, encrypted cache, stale/max-age policy, fail-closed and revocation behavior.
- **P9.5 Providers:** documented custom vault adapter interface, capabilities, timeouts, errors and health.
- **P9.6 Runtime APIs:** secure PowerShell and C# variable/secret access via host capabilities without leaking unrestricted DB/Vault handles.

**Exit criteria:** both languages access authorized scoped variables/secrets, Vault refresh and fail-closed behavior work, and all secret operations are safely audited/redacted.

### Phase 10 — Source storage, Git and immutable deployments

**Objective:** Support per-object database/file ownership, Git workflows, validation, rollback, and revision integrity.

**Steps**

- **P10.1 Source abstraction:** database and file-backed authoritative mode per object with revision history and concurrency checks on both providers.
- **P10.2 File-backed format:** `.ps1`, `.cs`, separate structured metadata per type, stable IDs, no secrets in repository, file/hash manifests.
- **P10.3 Migration between modes:** explicit source-mode conversion preserving identities, permissions, metadata and provenance.
- **P10.4 Git:** platform-to-Git and Git-authoritative modes, publisher/followers, credentials, pull/push, conflict detection, audit.
- **P10.5 Deployment lifecycle:** validated immutable manifest; precompile C# on eligible nodes; stage, coordinate activation, rollback and revision tracking.
- **P10.6 Audit:** publishing, approval, mode changes, Git integration and revision changes produce immutable redacted audit events.

**Exit criteria:** mixed source modes coexist, two Git workflows function, invalid revision cannot replace working one, rollback restores a consistent prior version.

**Milestone C:** source-controlled automation with release lifecycle.

### Phase 11 — Clustering, distributed execution and high availability

**Objective:** Scale to four Windows servers on a shared cluster-specific SQL Server database while keeping workload, source and configuration consistency.

**Steps**

- **P11.1 Node registration/health:** authenticated nodes, identity, availability, runtime/credential readiness, heartbeats and maintenance mode.
- **P11.2 Distributed jobs:** SQL leases/fencing, selection any/specific/all nodes, retry routing, worker loss recovery.
- **P11.3 Singleton scheduling:** shared SQL coordination, failover and duplicate occurrence prevention.
- **P11.4 HTTP HA:** load balancer, auth/session coherence, readiness, trusted proxy/IP audit consistency, graceful drain.
- **P11.5 Revision deployment:** hashes, local source validation, precompiled C# readiness, coordinated activation and node exclusion.
- **P11.6 Cross-node logging:** aggregate records preserving originating node/worker; safe spool and sink ownership; duplicate/idempotency behavior; independent cluster policies.
- **P11.7 Upgrades:** installer-controlled single EF migration, schema compatibility and rolling upgrade/maintenance handling.
- **P11.8 Failure testing:** kill scheduler/node/worker during jobs and deployments; reconcile state and verify audits.

**Exit criteria:** four-node cluster passes job ownership, schedule, HTTP failover, revision consistency, EF upgrade coordination and logging/audit recovery tests.

### Phase 12 — Complete administration UI and code editor

**Objective:** Achieve UI/API parity with a capable PowerShell/C# browser editor.

**Steps**

- **P12.1 Complete navigation:** scripts, endpoints, schedules, triggers, environments, variables/secrets, vaults, identities, roles, clusters, Git, deployments, logs, settings.
- **P12.2 Monaco editor:** tabs, syntax, folding, shortcuts, revision diff, history, optimistic conflict controls.
- **P12.3 PowerShell intelligence:** PowerShell language service, lint, modules, completion, signatures and version-specific validation.
- **P12.4 C# intelligence:** Roslyn completion/diagnostics, source locations, runtime-aware compile status and cache readiness.
- **P12.5 Test workflows:** script/job execution, endpoint request builder, live output, publish/rollback and permissions.
- **P12.6 Audit viewer:** configuration history with redacted field diffs and actor/origin; endpoint audit search by user, route/method/IP/host/status/time/correlation.
- **P12.7 Logging target designer:** configure destination, typed columns/property mapping, rotation/batching/failure policy, test records, health/queue/errors, use permissions and default target inheritance.
- **P12.8 Parity tests:** map every UI action to authenticated, authorized, documented API and verify audit generation.

**Exit criteria:** all management tasks work from both UI/API, logs/audits can be inspected safely, language diagnostics work and concurrency/RBAC are enforced.

### Phase 13 — Monitoring, published files and operations

**Objective:** Complete production observability and operations without weakening mandatory audits.

**Steps**

- **P13.1 Dashboards:** nodes, database health, Vault, Git, scheduler, job counts/errors, endpoints, runtime compatibility, compilation/cache, active revision.
- **P13.2 Configurable telemetry:** global/type/folder/object settings, API and job metrics, retention/sampling/aggregation; explicit audit separation.
- **P13.3 Logging operations:** target health, throughput, queue depth, spool usage, delivery delay, dead letters, sink errors and search/export permissions.
- **P13.4 Audit operations:** immutable audit queries/export, retention, tamper-evidence or protected write constraints, detection of delivery gaps, secure SIEM forwarding.
- **P13.5 Published files:** safe routes, static file access, index/list policies, RBAC, synchronized/available content across nodes.
- **P13.6 Alerts/exports:** health notifications, OpenTelemetry/Prometheus where selected, Windows Event Log, log forwarding.

**Exit criteria:** monitoring views reflect reality; disabling optional metrics/custom log targets does not disable mandatory audit; protected file publication operates behind load balancer.

### Phase 14 — Production hardening, installation/upgrade validation, and manual cutover

**Objective:** Verify all requirements, install/upgrade paths, recovery and performance before moving real workloads.

**Steps**

- **P14.1 Packaging:** install/upgrade Windows Service, certificate and account setup, EF Core migration bundle, database version preflight, logging spool paths and permissions.
- **P14.2 Upgrade testing:** fresh install, existing-schema upgrade, migration failure and restore, single migration per cluster, expand/contract compatibility, service rollback.
- **P14.3 Security:** secrets, run-as, RBAC, token lifecycle, auditing completeness, logger sink injection/path traversal, isolation, proxy spoofing, API abuse and redaction.
- **P14.4 Durability:** SQL/Vault/Git/node failure, endpoint audit spool exhaustion, logging target outage/recovery, job/schedule deduplication, source revision consistency.
- **P14.5 Performance:** PowerShell/C# throughput, API latency, compilation reuse, async logging throughput/backpressure and SQL audit growth under realistic loads.
- **P14.6 Manual PSU migration:** inventory and manually recreate scripts, endpoints, schedules, roles, run-as configurations, secrets and Git layouts; verify equivalent outputs and document differences. **Do not create an automated importer.**
- **P14.7 Four-node staged rollout:** dev -> staging -> production parallel operations where possible, rollback playbook, operator runbooks and sign-off.
- **P14.8 Final traceability:** close every MUST requirement in v4 and all **21 mandatory acceptance gates** with recorded evidence.

**Exit criteria:** clean install and upgrade are tested, audit/logging/security/recovery tests pass, performance limits documented, migrated pilot workloads validated manually and production rollback rehearsed.

**Milestone D:** production-ready replacement.

---

## 4. Phase dependency and milestone summary

| Phase | Deliverable | Depends on |
|---|---|---|
| 0 | Prototypes, architecture, threat model | — |
| 1 | Windows host + EF Core install/migrations + audit outbox | 0 |
| 2 | AD/Entra, RBAC, durable configuration audits | 1 |
| 3 | Runtime environments + isolated PowerShell workers | 2 |
| 4 | C# compile/cache/isolated execution | 3 |
| 5 | Durable script jobs and history | 4 |
| 6 | Async custom logging + PowerShell/C# SDKs | 5 |
| 7 | PowerShell/C# HTTP endpoints + automatic invocation audit | 6 |
| 8 | Scheduling and triggers | 5, 7 |
| 9 | Variables, full secrets and Vault | 2, 5 |
| 10 | Database/file source, Git and deployments | 4, 7, 9 |
| 11 | Cluster HA and distributed coordination | 7, 8, 10 |
| 12 | Complete UI, editor, logging/audit designers | 6–11 |
| 13 | Metrics, operations and published files | 11, 12 |
| 14 | Production validation and **manual** PSU cutover | All prior phases |

**Pilot checkpoints:** Phase 5 usable script execution; Phase 7 script + API host with custom logging; Phase 10 deployable Git-backed platform; Phase 11 four-node HA; Phase 14 production-ready.

---

## 5. Required audit data contracts

### 5.1 ConfigurationChangeAudit (mandatory)

| Category | Required fields |
|---|---|
| Identity | `AuditId`, `ActorPrincipalId`, `ActorDisplayName`, `ActorType`, `AuthScheme`, `ServiceTokenId` (never token secret), `EffectiveIdentity` |
| Resource | `ResourceId`, `ResourceName`, `ResourceType`, `Scope`, `Operation`, `OldRevision`, `NewRevision` |
| Change | `ChangedFields`, `BeforeRedacted`, `AfterRedacted`, `Outcome`, `ErrorCategory`, `Reason`, `ApprovalReference` |
| Request/source | `Origin` (UI/API/Git/system/script), `HttpMethod` when applicable, `RequestPath`, `ClientIp`, `TrustedProxyChain`, `RequestId`, `CorrelationId` |
| Execution | `OccurredAtUtc`, `ProcessingNodeId`, `HostName`, `ClusterId`, `TraceId` |

Capture configuration modifications from every supported entry point. Never rely solely on UI event handlers; persistence/application commands must enforce audit emission. For a successful protected change, write an audit/outbox row in the same database transaction.

### 5.2 EndpointInvocationAudit (mandatory)

| Category | Required fields |
|---|---|
| Request | `AuditId`, `StartUtc`, `EndUtc`, `HttpMethod`, `RouteTemplate`, `SanitizedRequestPath`, `EndpointId`, `EndpointName`, `RequestSizeBytes`, `UserAgent` |
| Caller | `PrincipalId`, `PrincipalDisplayName`, `PrincipalType`, `AuthScheme`, `TokenId`, `ClientIp`, `TrustedProxyChain` |
| Execution | `ReceivingNodeId`, `ReceivingHost`, `ReceivingIp`, `ExecutingNodeId`, `ExecutingHost`, `WorkerProcessId`, `WindowsRunAs`, `RuntimeVersion` |
| Provenance | `SourceRevision`, `EnvironmentRevision`, `DeploymentRevision`, `RequestId`, `CorrelationId`, `TraceId` |
| Result | `StatusCode`, `DurationMs`, `ResponseSizeBytes`, `Outcome`, `ErrorCategory`, `WorkerCrash`, `Cancelled`, `RateLimited` |

Record successful requests, application failures, authorization rejections, rate limits, and timeouts; preserve a clear absence marker for fields unavailable before a worker launches. Do not store raw request/response bodies, cookies or authorization headers by default. Treat client IP reported from untrusted forwarding headers as untrusted data, not identity.

---

## 6. Named logging targets: required model and example

A logging target is a **resource object**, with object-level RBAC, revision/version, audit history and a selected sink configuration. Scripts choose targets by name or stable ID. An environment/folder/script/endpoint may provide an inherited default target.

Example target: **`InventorySql`**

| Property | Example |
|---|---|
| Destination | SQL Server, pre-existing `ops.InventoryLogs` table or approved managed table |
| Columns | `OccurredUtc datetime2`, `Level nvarchar(20)`, `Message nvarchar(max)`, `RowsUpdated int`, `Warehouse nvarchar(100)`, `DurationMs int`, `JobId uniqueidentifier`, `NodeName nvarchar(128)` |
| Mappings | fixed context fields + structured script-provided properties with validation |
| Delivery | durable local spool, background batch, retry/backoff, idempotent record ID |
| Security | allow `Use` to selected scripts/roles, restrict `Manage` and `View` |

Example target: **`InventoryFile`**

- Destination: approved `D:\AutomationLogs\Inventory\` directory.
- Format: JSON Lines (or configurable CSV/TSV).
- Fields: `OccurredUtc`, `Level`, `Message`, `ScriptId`, `CorrelationId`, `RowsUpdated`.
- Policies: daily/size rotation, max disk budget, retention, redaction and failure behavior.

A target may optionally fan out to both destinations. The system should validate schema compatibility, avoid dynamic SQL injection, and use parameterized writes. EF Core manages **platform-owned schema**; runtime-configurable destination columns use safe mapping into existing tables or a separately governed DDL provisioning mechanism rather than changing EF migrations at every target edit.

### Required log SDK contracts

PowerShell: `Write-AutomationLog -Target <name> -Level <level> -Message <text> -Properties <hashtable> [-EventId <id>] [-Exception <error>]`.

C#: `AutomationLogger` implementing `IAutomationLogger`, with `Task<LogReceipt> LogAsync(string target, AutomationLogLevel level, string message, IReadOnlyDictionary<string, object?> properties, CancellationToken cancellationToken = default)` (final signature may be refined, but typed async invocation and named target selection are mandatory).

Both must attach execution context automatically, resolve only authorized target IDs, and surface enqueue/delivery failures according to documented policy. Endpoint audit records are generated by the platform and do not depend on scripts calling the logger.

---

## 7. EF Core installation and upgrade workflow

### SQL Server

1. Develop entity models and mapping configuration in version control; generate checked-in EF Core migrations, reviewed with each PR.
2. Build the application and an authorized migration bundle as part of CI. Validate migration scripts on a clean DB and a previous-release DB.
3. For clean installation, preflight SQL/permissions/backup/key availability, then use the migration identity to bring the DB to the exact approved schema version.
4. For upgrades, place incompatible nodes in maintenance/drain mode as needed, apply schema changes once with a coordinator lock, verify resulting migration history and audit the schema transition.
5. Start/upgrade application nodes only after their expected schema compatibility is verified; support rolling upgrade with expand/contract migrations when possible.
6. On failure, report the migration and schema versions, preserve logs and audit evidence, and follow a documented restore/roll-forward procedure. Do not assume EF migrations are automatically reversible without data risk.
7. Keep the normal application identity limited to DML/procedure permissions needed for operation; migration DDL rights should be separate.

### SQLite local development

1. Select the SQLite provider in the standalone local development profile and configure a writable local database path. Create or upgrade the file with the documented command while the host is stopped. Apply only the SQLite migrations for the selected release.
2. Start the host and verify exclusive instance ownership, schema compatibility, and active WAL mode. Report the failing prerequisite if startup cannot proceed.
3. Back up using a SQLite-aware snapshot or stop the host and checkpoint the WAL successfully before copying the database. Retain encryption keys needed to recover local secrets and verify the restored database before resuming work.
4. In CI, apply the checked-in SQLite migrations to clean and previous-release databases. Exercise restart and restore, concurrent updates, atomic audit writes, job and schedule deduplication, bounded busy handling, and rejection of unsupported startup configurations. Run SQL Server cluster and installer checks separately.

For queue leasing and other specialized hot paths, reviewed parameterized SQL or stored procedures may complement EF Core; this does not replace EF Core as the platform's schema management foundation.

---

## 8. Codex workflow and definition of done

Use `SPEC.md` as requirements truth and `IMPLEMENTATION_PLAN.md` as execution order. Create small tickets such as `P6.3a Validate SQL log target mappings` rather than one giant phase-wide prompt. Every task should specify: **requirement IDs, affected modules, API contracts, schema/migration changes, authorization, audit events, tests, failure modes, and acceptance criteria**.

Recommended repository files:

- `SPEC.md` (v4) and `IMPLEMENTATION_PLAN.md` (this file).
- `AGENTS.md`: code style, interfaces, scope boundaries, security, testing and no-silent-skips rules.
- `ARCHITECTURE.md` and `docs/adr/`: authoritative system design and decisions.
- `docs/contracts/`: job, endpoint, audit, logging, runtime, SDK and deployment schemas.
- `docs/requirements-matrix.md`: requirement ID -> task(s) -> test(s) -> completion evidence.
- `docs/progress.md`: completed/in progress/blocked; dependency and regression tracking.
- `tests/`: unit, Windows integration, SQL Server and SQLite, security-negative, failure injection and 4-node load tests.

**Mandatory task lifecycle:** inspect requirements -> propose focused implementation -> implement -> tests/security validation -> documentation and requirement mapping -> commit. No phase is complete with a failing mandatory acceptance test. Keep temporary development-auth shortcuts out of production builds.

## 9. Final production acceptance

All **21 mandatory v4 gates** must be demonstrated, including the three new gates:

- **Gate 19:** automatic configuration audit and hosted endpoint invocation audit, with actor/method/route/IP/host/revisions and durable fail-closed semantics.
- **Gate 20:** reusable configurable async logging targets (SQL/file), `Write-AutomationLog`, `AutomationLogger.LogAsync`, schema mapping, RBAC, backpressure and recovery.
- **Gate 21:** validate EF Core clean installation, upgrade, concurrency protection, schema compatibility, and recovery for both providers. Verify SQL Server cluster migration coordination and the standalone SQLite WAL startup and rejection cases in the spec.

The platform is production-ready only when security, recoverability, revision consistency, full management API parity and the four-node deployment are validated. Manual migration of existing PSU objects is sufficient; an automatic migration tool is not required.

**End — Codex Implementation Plan v2.0**
