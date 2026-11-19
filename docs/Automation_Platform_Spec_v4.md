
# PowerShell & C# Automation Platform
## Replacement Feature Specification — v4

**Document Type:** Functional, Technical, Security, and Operational Requirements  
**Target Platform:** C#/.NET application hosted on Windows Server  
**Supported Workload Languages:** PowerShell and C#  
**Primary Database:** Microsoft SQL Server  
**Local Development Database:** SQLite in write-ahead logging (WAL) mode for a single standalone application instance  
**Deployment:** Standalone instance or multi-node cluster  
**Status:** Requirements specification

---

# 1. Overview

## 1.1 Purpose

This document defines the requirements for building an enterprise automation platform intended to replace an existing PowerShell automation, API hosting, scheduling, and execution platform.

The replacement must be implemented in C#/.NET and provide a centralized interface for creating, configuring, executing, securing, and monitoring PowerShell and C# automation workloads.

The system must support:

- PowerShell and C# script execution.
- Custom HTTP API endpoints written in PowerShell or C#.
- Scheduled and ad hoc jobs.
- Multiple runtime environments and runtime versions.
- Configurable execution identities.
- Distributed execution across clustered servers.
- Fine-grained, nested role-based access control.
- Centralized variables and encrypted secret management.
- Native HashiCorp Vault integration.
- Database-backed and file-backed source storage.
- Git-based source management and deployment.
- High availability behind a load balancer.
- A complete administrative REST API.
- A web-based administration interface and code editor.
- Fault-isolated workload execution.
- Configurable monitoring, metrics, and auditing.
- Mandatory detailed configuration and HTTP endpoint auditing.
- Customizable asynchronous logging targets with PowerShell/C# client APIs.
- EF Core database management and migration for installation and upgrades.

The platform must support a single-server deployment without requiring clustering infrastructure.

It must also support independently operated clusters containing one or more servers, including an existing four-server production configuration.

## 1.2 Technology Requirements

The platform MUST:

1. Be implemented primarily in C# using .NET.
2. Run as one or more Windows Services.
3. Support Microsoft SQL Server and offer SQLite in WAL mode for single-instance standalone local development.
4. Support Windows PowerShell 5.1.
5. Support multiple side-by-side PowerShell 7 installations.
6. Support C# execution using configurable installed .NET versions.
7. Execute PowerShell and C# workloads outside the privileged application host.
8. Support both standalone and clustered operation.
9. Provide a browser-based administration application.
10. Expose documented administrative APIs for every UI management operation.
11. Provide granular, object-level authorization.
12. Support mixed database-backed and file-backed source storage on a per-object basis.

The service itself must use a supported .NET runtime, but that runtime must not determine which PowerShell or .NET versions can be selected for user workloads.

## 1.3 Definitions

**Object:** Any independently configured resource, including a script, endpoint, schedule, trigger, variable, secret, role, runtime environment, vault, folder, token, execution record, deployment configuration, or published file resource.

**Script:** An executable PowerShell or C# source definition.

**Job:** An execution instance of a script, whether initiated manually, by API, by another script, or through automation.

**Run-As Identity:** The Windows/domain/service account used to execute a workload.

**Caller Identity:** The user or system identity requesting an operation.

**Runtime Environment:** A reusable configuration defining the language, runtime version, dependencies, and execution behavior for a workload.

**Cluster:** A logical deployment consisting of one or more application instances sharing cluster-specific operational state.

**Database-Backed Object:** An object whose authoritative editable source and metadata are stored in the configured operational database.

**File-Backed Object:** An object whose authoritative source and representable nonsecret metadata are stored in files.

**Published Revision:** A validated, immutable configuration and code version eligible for execution.

## 1.4 Requirement Classification

Unless explicitly designated optional or future functionality, requirements in this specification are mandatory.

- **MUST:** Required for feature completeness.
- **SHOULD:** Strongly recommended, but not a mandatory initial-release gate.
- **MAY:** Optional capability.
- **FUTURE:** Extension anticipated by the architecture but not required in the first release.

Where requirements describe performance or fault tolerance without numeric thresholds, representative acceptance testing will establish the supported limits.

---

# 2. Core Architecture

## 2.1 Application Components

The architecture must logically separate the following responsibilities:

1. HTTP/API hosting.
2. Administrative web application.
3. Authentication and authorization.
4. Runtime environment management.
5. Workload compilation and validation.
6. Job execution and worker management.
7. Scheduling and event triggers.
8. Cluster coordination.
9. Database persistence.
10. Source storage and Git synchronization.
11. Variable and secret management.
12. Monitoring, telemetry, and auditing.

Components may share a Windows Service or be deployed as separate services, provided the required security and workload-isolation boundaries remain intact.

The architecture must not require every component to run as an independently installed service.

## 2.2 Deployment Models

The application must support:

### Standalone

- One application instance.
- One SQL Server database, or one local SQLite database in WAL mode for local development (see §10.5).
- Local execution workers.
- All required functionality without additional application nodes.
- No dependency on distributed coordination services solely for standalone operation.

### Clustered

- One to many application instances.
- Shared cluster-specific SQL Server database.
- Distributed execution and scheduling.
- Shared configuration and source revisions.
- Load-balanced HTTP services.
- Automatic detection of unavailable nodes.
- Coordinated deployment and job ownership.

A one-node cluster must be valid. Every cluster must use SQL Server, including a one-node cluster.

Adding another node must not require redesigning the deployment.

## 2.3 Independent Clusters

Each cluster must be independently configurable.

Every cluster must support its own:

- SQL Server database.
- Git repository and branch configuration.
- Vault connections.
- Runtime environments.
- Execution identities.
- Roles and permissions.
- Schedules.
- Scripts and endpoints.
- Deployment revisions.
- Monitoring configuration.
- Administrators.
- Service configuration.

A central cross-cluster management service is not mandatory.

Clusters must not require a shared database, shared secrets, or shared Git configuration.

Cross-cluster administration or federation may be added later without compromising cluster independence.

## 2.4 Cluster Consistency

All nodes belonging to the same cluster must execute against identical published source and configuration revisions.

For file-backed resources:

- Files must be synchronized across all participating nodes.
- Source and metadata must be identical for a given revision.
- Configuration changes must not be partially deployed.
- Nodes must report their active revisions.
- Nodes with incompatible revisions must be excluded from affected workload execution until synchronized.

Compiled C# artifacts may be stored in node-local caches, but they must correspond to the same published source and dependency revision.

Runtime-specific local installation paths may differ when necessary, provided each node resolves the configured environment to an equivalent runtime and dependency set.

---

# 3. Runtime Environment Management

## 3.1 Runtime Environment Objects

**RUN-01** The platform must provide independently configurable runtime environment objects.

**RUN-02** Runtime environments must support at least two language types:

- PowerShell
- C#

**RUN-03** An environment must contain:

- Unique identifier.
- Name.
- Description.
- Language.
- Runtime version selection.
- Runtime discovery/resolution rules.
- Dependency configuration.
- Environment variables.
- Working-directory configuration.
- Execution limits.
- Optional default run-as identity.
- Module or assembly configuration.
- Applicable security policies.
- Enabled/disabled state.
- Revision history.
- Object-level RBAC.

**RUN-04** Every script and endpoint must be assignable to a runtime environment.

Schedules and triggers must support referencing or overriding a runtime environment where permitted.

**RUN-05** A default runtime environment may be configured for each language.

**RUN-06** Runtime environments must be reusable across multiple objects.

**RUN-07** Environment changes must be versioned, validated, and propagated consistently across cluster nodes.

**RUN-08** Execution history must record the exact effective environment revision and resolved runtime version.

## 3.2 PowerShell Environments

**RUN-09** Windows PowerShell 5.1 support is mandatory.

**RUN-10** PowerShell 7 environments must support any installed PowerShell 7 version that the platform can execute.

**RUN-11** Multiple PowerShell 7 versions must coexist without requiring the replacement or upgrade of other installed versions.

**RUN-12** Runtime selection must support exact installed version selection rather than only broad choices such as "PowerShell 7."

**RUN-13** The application must discover installed PowerShell versions and display them in the administration UI.

**RUN-14** Each environment must identify its actual PowerShell executable and version.

**RUN-15** Environments must support configurable:

- PowerShell modules.
- Module search paths.
- Working directories.
- Environment variables.
- Execution policy where applicable.
- Script initialization.
- Execution constraints.

**RUN-16** Different scripts and endpoints must be able to execute simultaneously under different PowerShell versions.

**RUN-17** PowerShell modules loaded in one execution environment must not inadvertently affect other environments.

**RUN-18** A missing runtime version must produce a clear validation or eligibility error rather than silently using an incompatible version.

## 3.3 C# Environments

**RUN-19** The platform must support C# runtime environments.

**RUN-20** A C# environment must support selecting any installed .NET runtime version that the platform's C# execution backend can use, including side-by-side .NET installations.

**RUN-21** The runtime version must be independent of the .NET version hosting the automation platform.

**RUN-22** A C# environment must support:

- Target .NET runtime.
- Runtime version resolution.
- Compilation configuration.
- Language version.
- Assembly references.
- Approved external dependencies.
- Dependency resolution.
- Environment variables.
- Working directory.
- Execution limits.
- Default execution identity.
- Security restrictions.

**RUN-23** Multiple C# environments must support different .NET versions simultaneously.

**RUN-24** The system must enumerate installed .NET runtimes and report their availability on each cluster node.

**RUN-25** An environment must not silently roll forward to a different runtime version unless an explicit runtime-resolution policy permits it.

**RUN-26** A C# environment may reference shared libraries and approved packages.

**RUN-27** Dependency changes must cause affected C# definitions to be revalidated and recompiled.

**RUN-28** The platform must identify unavailable or incompatible runtime dependencies before execution.

**RUN-29** A compiled C# workload must execute using its configured environment, not automatically inherit the runtime or permissions of the hosting service.

**RUN-29A** C# workload targets are limited to modern .NET (.NET Core / .NET 5+ family), including older installed versions that the execution backend can support. Legacy .NET Framework 4.x execution is outside the required scope. The supported runtime/SDK/reference-assembly compatibility matrix must be documented.

## 3.4 Runtime Availability Across Nodes

**RUN-30** Each node must report its installed and available execution runtimes.

**RUN-31** The scheduler and dispatcher must assign execution only to nodes compatible with the selected environment.

**RUN-32** Administrators must be able to inspect environment compatibility across the cluster.

**RUN-33** Runtime and dependency mismatches must appear in deployment validation, node readiness, and diagnostic views.

**RUN-34** Production deployment of a revision must verify that every node intended to execute the affected workload can use the required environment.

**RUN-35** Runtime environments must support object-level and administrative API management.

### Acceptance Demonstration

Install two different PowerShell 7 versions alongside Windows PowerShell 5.1.

Create three PowerShell environments, one for each version.

Create two C# environments targeting different installed .NET versions.

Execute workloads simultaneously under the selected environments and verify that execution records show the correct versions.

Remove or disable a runtime on one test node and demonstrate accurate node eligibility and deployment diagnostics.

---

# 4. C# Workload Compilation and Execution

## 4.1 C# as a First-Class Automation Language

**CSH-01** C# must be supported as a first-class language for both jobs and HTTP API endpoints.

**CSH-02** C# scripts must support the same relevant management operations as PowerShell scripts:

- Create.
- Edit.
- Delete.
- Version.
- Validate.
- Compile.
- Execute.
- Schedule.
- Cancel.
- Inspect output.
- Review history.
- Assign run-as identity.
- Assign runtime environment.
- Assign RBAC.
- Deploy.
- Roll back.

**CSH-03** C# API endpoints must support the same HTTP, authentication, request-context, output, and RBAC capabilities as PowerShell endpoints.

**CSH-04** C# source definitions must use a documented entry-point contract for jobs and endpoints.

**CSH-05** C# source must support asynchronous execution using appropriate .NET asynchronous programming constructs.

## 4.2 Compilation Lifecycle

**CSH-06** C# scripts and endpoints must compile when their definitions are created or changed.

**CSH-07** Compilation must occur during definition validation and before a new revision becomes eligible for execution.

**CSH-08** The system must not wait until the first incoming HTTP request or job execution to compile newly published C# source.

**CSH-09** Compilation must run against the selected runtime environment, references, and dependency versions.

**CSH-10** Compilation errors must prevent activation of the invalid revision.

**CSH-11** Compilation failures must not replace or invalidate the last successfully published revision.

**CSH-12** Compiler diagnostics must include:

- Error or warning severity.
- Diagnostic code.
- Message.
- File name.
- Line.
- Column.
- Relevant source location.

**CSH-13** Diagnostics must be displayed in the web editor and exposed through the management API.

**CSH-14** The system must support an explicit compile/validate action without deploying the result.

**CSH-15** Compilation must respect configurable timeout and resource limits.

## 4.3 Per-Node Compiled Code Cache

**CSH-16** Every eligible cluster node must maintain a local cache of compiled C# workloads.

**CSH-17** Successfully compiled revisions must remain cached for reuse rather than recompiling on each execution.

**CSH-18** Compiled cache identity must incorporate all inputs that can affect compilation, including:

- Source content/revision.
- Runtime environment revision.
- Target .NET runtime.
- Compiler configuration.
- Assembly references.
- Dependency versions.
- Relevant compilation settings.

**CSH-19** Saving a new source revision must invalidate the compiled cache for that revision's changed inputs without destroying cache entries for still-valid published revisions.

**CSH-20** When a new revision is published, each eligible node must compile or securely acquire and validate its corresponding compiled artifact before the revision is activated on that node.

**CSH-21** A node must not execute a source revision against a compiled artifact from another incompatible revision.

**CSH-22** Cache consistency must be checked using revision identifiers and content hashes.

**CSH-23** New or restarted nodes must populate their required compiled caches during synchronization or workload preparation.

**CSH-24** Compilation failures on one node must be visible to administrators and must prevent that node from advertising readiness for the affected revision.

**CSH-25** Cache cleanup, capacity limits, and obsolete revision retention must be configurable.

**CSH-26** Administrators must be able to request cache validation or forced recompilation through the UI and management API.

**CSH-27** Manual cache refresh must not cause unnecessary interruption to unrelated executing workloads.

## 4.4 C# Job Execution

**CSH-28** C# jobs must support parameterized execution.

**CSH-29** The platform must provide a documented execution context to C# jobs containing:

- Job ID.
- Script ID.
- Script revision.
- Parameters.
- Caller identity.
- Run-as identity.
- Runtime environment.
- Node information.
- Cancellation token.
- Logging/output interface.
- Authorized variable access.
- Authorized secret access.
- Correlation ID.

**CSH-30** Jobs must support structured result objects.

**CSH-31** Standard output, errors, warnings, diagnostic messages, and structured results must be captured.

**CSH-32** C# jobs must support cancellation and execution timeouts.

**CSH-33** C# jobs must follow the same configurable script/schedule retry policies as PowerShell jobs.

**CSH-34** An exception or process failure must be recorded against the execution without crashing the hosting application.

## 4.5 C# HTTP Endpoints

**CSH-35** A C# endpoint must support all configured HTTP methods and route patterns available to PowerShell endpoints.

**CSH-36** The platform must provide a documented HTTP request context containing:

- HTTP method.
- Original URL.
- Matched route.
- Route parameters.
- Query parameters.
- Headers.
- Cookies.
- Request body.
- Parsed JSON/form content.
- Authenticated principal/claims.
- Client/network information.
- Correlation ID.
- Cancellation token.
- Deadline/timeout information.

**CSH-37** Endpoint code must be able to control:

- Status code.
- Response headers.
- Cookies.
- Content type.
- JSON responses.
- Text responses.
- Binary responses.
- File downloads.

**CSH-38** C# endpoints must support dependency access through explicitly provided, authorized platform interfaces.

**CSH-39** C# endpoints must not require recompilation for every incoming request.

**CSH-40** C# endpoints must not be automatically re-executed or retried when their execution fails.

**CSH-41** An endpoint's failure must return the configured error response without destabilizing the HTTP service.

## 4.6 C# Security and Isolation

**CSH-42** User-authored C# code must not execute inside the privileged HTTP, scheduler, or administrative service process.

**CSH-43** Compiled C# must execute inside isolated, supervised worker processes.

**CSH-44** C# workloads must support configured Windows run-as identities.

**CSH-45** C# execution must respect memory, CPU, time, process, concurrency, and output constraints.

**CSH-46** Arbitrary C# code must not receive unrestricted references to privileged service objects, database connections, or administrative internals.

**CSH-47** Compilation and dependency restoration must not execute arbitrary untrusted build tasks in the privileged hosting process.

**CSH-48** Any shared compilation artifacts must have integrity validation and appropriate access controls.

### Acceptance Demonstration

Create a C# job and C# API endpoint.

Verify both compile when their definitions are saved/validated.

Verify the compiled revisions are cached on each eligible node.

Execute the endpoint repeatedly without recompilation.

Edit the source and verify cache invalidation and recompilation.

Introduce a compilation error and verify that the previously published revision remains operational.

Run the code under an alternate Windows identity and a different installed .NET version from the service host.

Kill the worker process and verify that the web service remains operational.

---

# 5. Scripts, Jobs, and Execution

## 5.1 Script Definitions

**JOB-01** Scripts must support PowerShell and C# source definitions.

**JOB-02** Every script must have:

- Stable object ID.
- Name.
- Description.
- Language.
- Runtime environment.
- Source storage mode.
- Source revision.
- Parameters.
- Default run-as identity.
- Timeout policy.
- Concurrency policy.
- Retry policy.
- Execution permissions.
- Version history.

**JOB-03** Scripts must be executable through:

- Administrative web UI.
- Management API.
- Schedules.
- Event triggers.
- Other automation scripts.
- Explicitly authorized workflows.

## 5.2 Job Execution

**JOB-04** Every execution must create a durable job record with a unique identifier.

**JOB-05** Execution records must include:

- Script and revision.
- Effective runtime environment and version.
- Caller.
- Run-as identity.
- Target node.
- Cluster.
- Start/end timestamps.
- Parameters, subject to redaction.
- Status.
- Output.
- Error information.
- Correlation ID.

**JOB-06** The system must support:

- Queued.
- Running.
- Completed.
- Failed.
- Warning.
- Cancelled.
- Timed out.
- Waiting for input where supported.

**JOB-07** PowerShell execution must capture pipeline output and applicable PowerShell information, host, verbose, warning, and error streams.

**JOB-08** C# execution must capture structured results, logs, warnings, and errors.

**JOB-09** The system must provide live execution output where supported.

**JOB-10** Historical output must remain accessible according to retention and RBAC policies.

**JOB-11** Jobs must support cancellation and configurable timeouts.

**JOB-12** Parameterized scripts must support automatically generated execution forms.

**JOB-13** Scripts must support configurable execution concurrency limits.

**JOB-14** Jobs must support execution on:

- A specified node.
- Any eligible node.
- Multiple selected nodes.
- All eligible nodes when explicitly requested.

**JOB-15** The dispatcher must enforce runtime compatibility and run-as availability when selecting nodes.

## 5.3 Run-As Identity

**JOB-16** Every script must support a configured default run-as identity.

**JOB-17** Every API endpoint, schedule, and trigger must support its own run-as configuration where applicable.

**JOB-18** When launching a script ad hoc, an authorized user must be able to select or override the run-as identity for that execution.

**JOB-19** Stored domain or service-account credentials are sufficient for the required implementation.

**JOB-20** Group Managed Service Accounts are not mandatory.

**JOB-21** An execution must never receive an alternate run-as identity solely because the caller can edit its parameters.

**JOB-22** Permission to select or use an execution identity must be separate from permission to execute a script.

**JOB-23** Effective identity selection must follow a documented precedence model:

1. Authorized explicit ad hoc execution override.
2. Endpoint/schedule/trigger-specific configuration.
3. Script default.
4. Runtime environment default.
5. Cluster default execution identity.

An HTTP client cannot supply an arbitrary run-as override unless the endpoint explicitly implements a separately authorized policy for doing so.

**JOB-24** The platform must record both the requesting principal and effective Windows execution principal.

**JOB-25** Stored credentials must be encrypted and access-controlled.

## 5.4 Retry and Rerun Policies

**JOB-26** Retry/rerun behavior must be configurable independently for each script.

**JOB-27** Schedules must support their own retry policies.

**JOB-28** A schedule-specific retry policy must override its script's default retry policy when explicitly configured.

**JOB-29** Policies must support applicable options such as:

- No automatic retry.
- Maximum retry count.
- Retry delay.
- Fixed delay.
- Exponential backoff.
- Retry on selected failure categories.
- Retry after worker termination.
- Manual review required.
- Retry only when safe/idempotent.
- Retry on another eligible node.

**JOB-30** An execution record must retain the original job ID, attempt number, retry chain, failure reason, and final outcome.

**JOB-31** Retry policies must account for scripts with non-idempotent side effects.

**JOB-32** Ad hoc execution may support an authorized per-launch retry override, but such an override must be explicitly visible and audited.

**JOB-33** API endpoint request executions must not have automatic retry/rerun behavior.

**JOB-34** An endpoint that explicitly queues an independent background job does not turn the HTTP request itself into a retryable operation. The independent job follows its own script policy.

## 5.5 Durability and Resource Protection

**JOB-35** Job queues and job state must be durable.

**JOB-36** Worker failures must not silently erase execution records.

**JOB-37** Orphaned jobs must follow their configured recovery policy.

**JOB-38** Each execution must have bounded resources and cleanup behavior.

**JOB-39** Executions must not modify other jobs' isolated runtime state.

**JOB-40** Job output must support retention, redaction, export, and size limits.

**JOB-41** The system should support configurable job priorities and workload classes.

### Acceptance Demonstration

Execute PowerShell and C# scripts under different runtimes and run-as accounts.

Launch an ad hoc job with an authorized identity override.

Demonstrate a denied unauthorized identity override.

Terminate a worker running a retry-enabled script and verify the configured retry policy.

Repeat using a script requiring manual review and verify that it is not automatically rerun.

---

# 6. HTTP API Endpoints

## 6.1 Endpoint Definitions

**API-01** The system must host custom HTTP routes backed by PowerShell or C# code.

**API-02** Endpoints must support configurable HTTP methods and parameterized routes.

**API-03** Every endpoint must support:

- Name and description.
- Route.
- HTTP methods.
- Language.
- Runtime environment.
- Source storage mode.
- Authentication policy.
- RBAC.
- Run-as identity.
- Timeout.
- Concurrency limits.
- Rate limits.
- Request size limits.
- Response configuration.
- Documentation metadata.
- Source revision.

**API-04** Endpoints must support standard REST patterns and custom HTTP routes.

## 6.2 Request Context

**API-05** Both languages must receive a complete, structured request context.

The context must include:

- Method.
- URL.
- Route template.
- Route variables.
- Query string.
- Headers.
- Cookies.
- Raw request body.
- Parsed JSON.
- Form data.
- Binary content.
- Content type.
- Authenticated user and claims.
- Caller/source information.
- Request/correlation ID.
- Cancellation information.
- Request deadline.

**API-06** PowerShell must receive request data through documented variables and/or a safe request facade.

**API-07** C# must receive equivalent information through documented context types.

**API-08** Request context must be isolated per request, including when execution workers are pooled.

## 6.3 Responses

**API-09** Endpoints must support:

- JSON.
- Text.
- Binary responses.
- Custom content types.
- Custom HTTP status codes.
- Custom headers.
- Cookies.
- File downloads.

**API-10** JSON serialization must be available by default.

**API-11** Structured validation errors must use documented response formats.

**API-12** Endpoint errors must not expose privileged internal implementation details by default.

## 6.4 Authentication and Authorization

**API-13** Endpoints must support configurable authentication and RBAC policies.

**API-14** Authentication choices must include supported user/session authentication and API tokens.

**API-15** Windows-integrated authentication must be supported where configured.

**API-16** Endpoint invocation permission and run-as identity usage permission must be evaluated separately.

## 6.5 Rate Limiting

**API-17** Rate limiting must be configurable through the web administration interface.

**API-18** Rate-limit configuration must also be available through the management API.

**API-19** Rate limits must apply independently to each node.

**API-20** The system must not require a cluster-wide rate-limit counter.

**API-21** Administrators must be able to configure limits at the applicable endpoint/route/method level.

**API-22** Per-node limits must be accurately represented in the UI and documentation.

For example, a configured limit of 100 requests per minute per node is not equivalent to a cluster-wide limit of 100 requests per minute.

**API-23** The system must return appropriate HTTP responses when rate limits are exceeded.

**API-24** Rate limiting must not require recompilation of C# endpoint source.

## 6.6 Execution Behavior

**API-25** Endpoint execution must occur outside the critical HTTP hosting process.

**API-26** Long-running endpoints must not exhaust front-end request handling capacity.

**API-27** Endpoints must support configurable timeouts and concurrency limits.

**API-28** Excess traffic must be rejected or queued according to explicit policy.

**API-29** Endpoint execution must not automatically retry after failure.

**API-30** Endpoints must support synchronous operation and configurable asynchronous/background execution where required.

**API-31** Request cancellation must be propagated to the execution worker.

**API-32** Optional capabilities should include:

- Streamed responses.
- Multipart uploads.
- Idempotency keys.
- Request/response schema validation.
- CORS configuration.
- API versioning.

### Acceptance Demonstration

Create matching PowerShell and C# endpoints.

Pass route parameters, query parameters, headers, cookies, and JSON content.

Return a custom HTTP status code and binary response.

Configure a rate limit of 20 requests per minute per node.

Verify the limit applies independently to two nodes behind a load balancer.

Verify a failed endpoint execution is not automatically retried.

---

# 7. Scheduling

## 7.1 Schedule Definitions

**SCH-01** The platform must support:

- One-time execution.
- Recurring interval execution.
- CRON expressions.
- Continuously repeated execution with delay.
- Paused/disabled schedules.

**SCH-02** Schedules must be durable database-managed objects.

**SCH-03** Each schedule must support:

- Script reference.
- Parameters.
- Runtime environment.
- Run-as identity.
- Target node selection.
- Retry policy.
- Overlap policy.
- Missed-run policy.
- Time zone.
- Enabled/disabled state.
- RBAC.
- Execution history.

**SCH-04** Schedules must support PowerShell and C# scripts.

**SCH-05** Runtime environment overrides must be validated against the script language and supported execution requirements.

## 7.2 Cluster Coordination

**SCH-06** A scheduled occurrence must be created once across a cluster unless all-node execution is explicitly configured.

**SCH-07** Scheduling must use durable shared coordination.

**SCH-08** Scheduler ownership must survive node failure or restart.

**SCH-09** Schedule history must include planned and actual execution times.

**SCH-10** Every occurrence must have a stable identifier.

## 7.3 Retry and Misfire Handling

**SCH-11** Every schedule must have a configurable retry/rerun policy.

**SCH-12** Supported behavior must include:

- No retry.
- Retry on failure.
- Retry on worker crash.
- Maximum retries.
- Delay/backoff.
- Manual review.
- Skip missed executions.
- Catch up missed executions.
- Controlled replay.

**SCH-13** Schedule-specific policy must override the script-level default.

**SCH-14** Overlapping execution must be configurable.

**SCH-15** DST and time-zone behavior must be documented and predictable.

## 7.4 Security

**SCH-16** Schedules must use configured workload identities rather than relying on an active browser session.

**SCH-17** A schedule may use an authorized secret without granting the interactive creator permission to reveal the secret.

**SCH-18** Schedule configuration and schedule history must have independently enforceable permissions.

### Acceptance Demonstration

Configure PowerShell and C# scheduled jobs.

Demonstrate CRON, interval, one-time, and recurring execution.

Terminate the scheduler during an occurrence and demonstrate the configured missed-run and retry behavior.

Verify one scheduled occurrence does not execute on every cluster node unintentionally.

---

# 8. Authentication and Identity

## 8.1 Identity Providers

**AUTH-01** Active Directory/LDAP support is mandatory.

**AUTH-02** Microsoft Entra ID authentication through OpenID Connect is mandatory.

**AUTH-03** The platform must support secure LDAP/LDAPS connections and group membership resolution.

**AUTH-04** Nested Active Directory groups must be supported as authorization inputs.

**AUTH-05** Entra OIDC must support issuer/audience validation, signing-key rollover, and claims mapping.

**AUTH-06** Identity-provider MFA and conditional-access policies must be respected.

**AUTH-07** The platform must support Windows Integrated Authentication through Kerberos/Negotiate where configured.

**AUTH-08** SAML, WS-Federation/AD FS, and compatible forms-based authentication should be supported where required.

## 8.2 API and Service Tokens

**AUTH-09** The platform must support user-bound access tokens.

**AUTH-10** User-bound tokens must be limited by the effective rights of the associated user.

**AUTH-11** The platform must support independent system/service tokens.

**AUTH-12** Service tokens must represent nonhuman security principals with independently assigned permissions.

**AUTH-13** Tokens must support:

- Scopes.
- Expiration.
- Rotation.
- Revocation.
- Ownership.
- Auditing.
- Optional client/IP restrictions.

**AUTH-14** Full bearer-token values must only be shown when issued or rotated.

**AUTH-15** Stored token material must be protected and never exposed through routine read APIs.

## 8.3 Identity Separation

**AUTH-16** The platform must distinguish between:

- Authenticating caller.
- Authorized principal.
- Executing Windows account.
- Vault authentication identity.
- Service identity.

**AUTH-17** A token must not inherit the permissions of the service or run-as account without explicit authorization.

**AUTH-18** The platform must handle group membership changes, session expiry, and directory outages securely.

### Acceptance Demonstration

Authenticate through Entra OIDC and Active Directory.

Resolve a nested AD group.

Create user-bound and service tokens with different permissions.

Revoke a token and verify that subsequent requests are rejected.

---

# 9. Authorization and RBAC

## 9.1 Object-Level Authorization

**RBAC-01** Every configurable object must support granular RBAC.

This includes:

- Scripts.
- C# definitions.
- API endpoints.
- Schedules.
- Triggers.
- Variables.
- Secrets.
- Vault registrations.
- Runtime environments.
- Run-as identities.
- Folders.
- Published files.
- Jobs and execution history.
- Roles.
- Tokens.
- Clusters and nodes.
- Deployment configurations.
- Monitoring configurations.

**RBAC-02** Authorization must use default-deny behavior.

**RBAC-03** Object discoverability must be independently controllable from permission to inspect or modify it.

## 9.2 Permission Types

**RBAC-04** The permission model must support applicable actions including:

- Discover/list.
- View metadata.
- View source.
- Create.
- Edit.
- Delete.
- Execute/invoke.
- View execution history.
- View output.
- Cancel execution.
- Retry execution.
- Modify permissions.
- Use execution identity.
- Use secret.
- Reveal secret.
- Rotate secret.
- Manage vault.
- Compile/validate code.
- Publish/deploy.
- Issue/revoke tokens.
- Manage runtime environments.
- Change system settings.

**RBAC-05** Permission to edit source must not automatically grant permission to deploy it.

**RBAC-06** Permission to execute a script must not automatically grant permission to edit its source.

**RBAC-07** Permission to use a secret must not imply permission to reveal its value.

## 9.3 Nested Roles

**RBAC-08** Custom roles must support nesting.

**RBAC-09** Nested membership and permissions must resolve transitively.

**RBAC-10** Role cycles must be detected and rejected.

**RBAC-11** Role inheritance and explicit deny/allow precedence must be deterministic and documented.

**RBAC-12** Roles must support direct principal assignment and group-derived membership.

**RBAC-13** Folder and collection inheritance must be supported.

**RBAC-14** Role nesting must not permit privilege escalation through unauthorized role modification.

## 9.4 Enforcement

**RBAC-15** Authorization rules must apply identically to the UI and management API.

**RBAC-16** Rules must also be enforced for:

- Custom endpoint execution.
- Scheduling.
- Background processing.
- Git synchronization.
- Deployment.
- Inter-node operations.
- Secret retrieval.

**RBAC-17** Client-side hiding of UI controls is not sufficient enforcement.

**RBAC-18** Authorization caches must be invalidated or refreshed safely across nodes.

**RBAC-19** The system must provide effective-permission inspection and administrative permission simulation.

**RBAC-20** All role and permission changes must be audited.

**RBAC-21** Cross-cluster role templates may be supported, but cluster-local authorization must remain independently enforceable.

### Acceptance Demonstration

Create nested custom roles and assign them through a mixture of direct grants and directory groups.

Restrict a C# script, PowerShell script, endpoint, schedule, secret, and runtime environment.

Attempt unauthorized access through both the UI and management API.

Verify effective-permission inspection identifies why access was allowed or denied.

---

# 10. Operational Persistence

## 10.1 Database Requirements

**SQL-01** Microsoft SQL Server must be the primary operational persistence backend.

**SQL-02** Each cluster must support its own SQL Server database.

**SQL-03** The database must hold applicable:

- Platform configuration.
- Object metadata.
- Database-backed source code.
- Runtime environment definitions.
- Identity and authorization records.
- Token metadata.
- Execution queues.
- Job history.
- Audit history.
- Scheduling state.
- Deployment state.
- Cluster coordination.
- Metrics and operational data.
- Database-backed variables and secrets.

**SQL-04** The service must have sufficient database permissions to perform all required normal platform operations.

**SQL-05** Normal platform database permissions must include authorized reading and writing of objects, queues, execution history, and the local encrypted secret store.

**SQL-06** The service must be able to retrieve and decrypt authorized local secrets at runtime.

**SQL-07** Runtime permissions must follow least privilege while remaining sufficient for normal operation.

**SQL-08** Database schema creation and migrations may use a separate elevated deployment identity.

The application must not require permanent SQL sysadmin privileges for routine operation.

## 10.2 Source Storage

**SQL-09** Database-backed source storage must be supported.

**SQL-10** Database-backed source and metadata must be versioned.

**SQL-11** Source modification must use concurrency/version checks.

**SQL-12** The database must preserve revision history and rollback information.

**SQL-13** File-backed source storage must coexist with database-backed storage.

**SQL-14** Source storage selection must be available per individual object.

## 10.3 Availability and Recovery

**SQL-15** SQL Server high-availability configurations must be supported.

**SQL-16** Transient SQL connectivity failures must not crash the application host.

**SQL-17** The platform must expose degraded health when required database writes cannot be performed.

**SQL-18** Database reconnection behavior must be resilient and configurable.

**SQL-19** The system must support documented database backup and restore procedures.

**SQL-20** Restore procedures must account for:

- Source revisions.
- Encryption key availability.
- Secrets.
- Vault references.
- Tokens and identities.
- RBAC.
- Queues.
- Execution history.
- Cluster configuration.

**SQL-21** Runtime and schema versions must be compatible during rolling cluster upgrades.

**SQL-22** Data retention and offloading must be configurable to control database growth.

## 10.4 Entity Framework Core and Database Migrations

**SQL-23** Entity Framework Core (EF Core) is the required ORM and schema management approach for the platform's operational database. Use the SQL Server provider for SQL Server deployments and the SQLite provider for the local development profile in §10.5.

**SQL-24** All platform-owned persistent entities, relationships, indexes, constraints, and schema changes must be represented by maintained EF Core models, mapping configurations, and version-controlled migrations. Direct SQL may be used for explicitly reviewed operations where EF Core is unsuitable, such as specialized leasing, high-throughput writes, or controlled log sink table provisioning.

**SQL-25** Development environments must support reproducible database creation and migration from a clean database. Tests must execute the checked-in migration history for the configured provider.

**SQL-26** Clean installation must create/upgrade the target SQL schema to the approved release version through a controlled installer/deployment migration step.

**SQL-27** Application upgrades must execute ordered, versioned, validated EF Core migrations with schema version tracking, error reporting, and supported rollback/recovery procedures.

**SQL-28** Database migrations must execute once per cluster deployment, using an authorized migration identity and appropriate locking/coordination. Ordinary application nodes must not race to auto-apply migrations during startup.

**SQL-29** Rolling upgrades must use compatible expand/contract schema changes or explicitly require maintenance mode when compatibility cannot be preserved.

**SQL-30** Installation and upgrade tooling must preflight SQL access, migration state, prerequisites, and backup policy; it must not silently proceed after an incompatible or failed migration.

**SQL-31** Schema migration execution, success/failure, identity, host, old/new schema versions, and correlation identifier must be audited.

**SQL-32** User-configurable logging destination columns and tables do not automatically become EF Core entity migrations. The logging subsystem must use validated typed mappings, existing destination tables, or separately authorized and documented schema-provisioning operations. It must not construct unsafe SQL from untrusted script inputs.

### Acceptance Demonstration

Deploy with a dedicated application SQL identity.

Create, read, update, and execute database-backed objects.

Store and retrieve an encrypted local secret through an authorized execution.

Restore a database backup and verify that the application recovers its objects, histories, and secret references.

## 10.5 SQLite for Local Development

**SQLITE-01** The application must offer an explicit SQLite provider option for single-instance standalone local development. Startup must reject SQLite in production or in any clustered configuration, including a one-node cluster. The application must prevent two running application instances from using the same SQLite database file.

**SQLITE-02** The SQLite database must reside in a writable directory on a local filesystem. The host must have access to the database and its `-wal` and `-shm` files. Startup must reject network and shared filesystem paths.

**SQLITE-03** The application must enable `PRAGMA journal_mode=WAL` during database initialization and verify that the database reports `wal` before serving requests or starting background services. Initialization or verification failure must produce an actionable startup error. Enable foreign-key enforcement on every SQLite connection. Keep write transactions short and bound the time spent waiting on `SQLITE_BUSY`, with observable failure if contention persists.

**SQLITE-04** Keep a separate, version-controlled EF Core migration history for SQLite. Provide a documented local command to create or upgrade the database without a SQL Server dependency. Apply migrations with the application stopped and exclusive access to the database. Startup must verify schema compatibility before serving requests.

**SQLITE-05** Preserve standalone persistence behavior for source revisions, configuration, encrypted secrets, audit and outbox records, jobs, and scheduling. Use application-managed concurrency tokens where SQL Server uses database-generated tokens. Configuration changes and their required audit records must commit atomically. Local job claims and schedule occurrence creation must remain atomic under concurrent access. SQL Server logging targets and cluster capabilities retain their own SQL Server requirements. The file logging target must be usable without SQL Server.

**SQLITE-06** Document local backup and restore using a SQLite-aware snapshot or a procedure that stops the application and checkpoints the WAL before copying the database. Preserve the key material required to restore encrypted secrets.

---

# 11. Source Storage Modes

## 11.1 Per-Object Configuration

**SRC-01** Every eligible source-backed object must support independently selecting its storage mode.

The required modes are:

1. Database-backed.
2. File-backed.

**SRC-02** Database-backed storage must be the default.

**SRC-03** The storage mode must be configurable through both the web UI and management API.

**SRC-04** Different objects of the same type may use different modes simultaneously.

Example:

- Script A: Database-backed.
- Script B: File-backed.
- Endpoint C: Database-backed.
- Endpoint D: File-backed.

**SRC-05** The selected storage mode must be visible in object details.

**SRC-06** Storage-mode changes must use explicit migration/validation rather than silently moving source data.

## 11.2 Database-Backed Mode

**SRC-07** Source code and editable metadata must be stored in the configured operational database.

**SRC-08** Database-backed source must not require an authoritative local file.

**SRC-09** Database-backed objects must support revision history, diff, rollback, and Git export.

## 11.3 File-Backed Mode

**SRC-10** File-backed source must store code and metadata in separate files.

**SRC-11** PowerShell source must use appropriate `.ps1` files.

**SRC-12** C# source must use appropriate `.cs` files or an explicitly configured supported C# source format.

**SRC-13** Metadata must not be required to be embedded in source-code headers or comments.

**SRC-14** Metadata must use structured, machine-readable files.

**SRC-15** One metadata file per object type must be supported as the standard file-backed representation.

For example:

    /automation/
      /scripts/
        Inventory.ps1
        Cleanup.ps1
        ReportJob.cs

      /endpoints/
        GetInventory.ps1
        ProcessOrder.cs

      /metadata/
        scripts.json
        endpoints.json
        schedules.json
        triggers.json

The specific file and directory naming scheme is configurable or implementation-defined, but the format must be documented.

**SRC-16** Object metadata files must reference objects using stable object IDs.

**SRC-17** Each metadata entry must associate its object with the correct source file and revision information.

**SRC-18** Metadata must represent all file-serializable nonsecret object configuration necessary for round-trip deployment.

**SRC-19** Secret values, bearer tokens, private keys, and stored credential contents must never be exported into source-controlled metadata.

**SRC-20** References to authorized secrets and execution identities may be stored in metadata.

## 11.4 Mixed-Mode Consistency

**SRC-21** File-backed and database-backed objects must coexist without ambiguous ownership.

**SRC-22** Every object's authoritative source location must be explicitly known.

**SRC-23** Editing one storage mode must not silently overwrite another authoritative source.

**SRC-24** Concurrent edits must use revision checking and conflict detection.

**SRC-25** Source and metadata must be committed or published as a logically consistent revision.

**SRC-26** Cluster nodes must resolve file-backed objects to identical published file revisions.

**SRC-27** A cluster-wide file synchronization or deployment mechanism must ensure file consistency across nodes.

**SRC-28** A new source revision must not become active on a node until the code and corresponding metadata have been validated.

**SRC-29** C# file-backed source changes must trigger compilation and cache refresh before activation.

**SRC-30** Storage-mode migration must preserve:

- Object ID.
- Permissions.
- Code revision.
- Metadata.
- Runtime environment.
- Run-as references.
- Endpoint routes.
- Schedule definitions.
- Variable references.
- Vault references.
- Deployment provenance.

**SRC-31** Rollback and export/import must support both storage modes.

### Acceptance Demonstration

Create a database-backed PowerShell script and a file-backed PowerShell script.

Create a database-backed C# endpoint and a file-backed C# endpoint.

Verify separate source and metadata files for file-backed objects.

Edit and deploy one file-backed source revision.

Verify identical source and metadata hashes across cluster nodes.

Migrate an object from database-backed to file-backed and back while preserving its identity and configuration.

---

# 12. Git Integration and Deployment

## 12.1 Deployment Modes

**GIT-01** The system must support at least two configurable workflows.

### Mode A — UI/Platform Authoritative

- Changes originate in the administration UI or management API.
- Approved changes are committed or exported to Git.
- A designated publishing node may perform Git pushes.
- Other nodes poll/pull published changes.

### Mode B — Git Authoritative

- Changes originate in Git or a CI/CD workflow.
- Published revisions are imported and validated.
- Nodes pull or receive approved deployments.
- UI changes must follow a configured review/publish workflow.

**GIT-02** The deployment model must be configurable by cluster.

**GIT-03** A deployment-mode change must not require application redevelopment.

## 12.2 Existing Publishing Topology

**GIT-04** The platform must support one designated publishing server.

**GIT-05** The publishing server may commit approved source and metadata changes to GitHub.

**GIT-06** Follower nodes must support periodic polling and pulling.

**GIT-07** Followers must not push or alter authoritative Git state unless explicitly promoted.

**GIT-08** Publisher reassignment must be configurable and audited.

## 12.3 Repository Configuration

**GIT-09** Configurable settings must include:

- Repository URL.
- Authentication method.
- Branch.
- Pull interval.
- Push permission.
- Publisher selection.
- Automatic deployment.
- Approval policy.
- Target cluster.
- Conflict resolution.
- Revision selection.

**GIT-10** Configuration must be available through the UI and administrative API.

**GIT-11** The platform must support authenticated Git access without storing plaintext credentials in repository files.

## 12.4 Deployment Integrity

**GIT-12** Source code and metadata must be versioned consistently.

**GIT-13** Deployments must support validation before activation.

**GIT-14** Deployments must support rollback.

**GIT-15** Mixed database-backed and file-backed objects must be represented consistently in deployment exports/imports.

**GIT-16** The system must not allow untracked last-writer-wins overwrites between UI, SQL, file source, and Git.

**GIT-17** Every execution must be traceable to its published source and configuration revision.

**GIT-18** Cluster deployments must support atomic or coordinated activation semantics.

**GIT-19** C# deployment validation must include compilation and cache preparation on applicable nodes.

**GIT-20** A failed C# compile must not result in an invalid revision serving requests on one node while other nodes continue serving a different unannounced revision.

**GIT-21** Git repositories must not contain secret values or private credential material.

## 12.5 CI/CD

**GIT-22** The platform should support:

- Webhook-triggered synchronization.
- Pull-request workflows.
- Deployment approvals.
- Automated testing.
- PowerShell linting.
- C# compilation validation.
- Deployment previews.
- Immutable release bundles.
- Signed or attested artifacts.
- Promotion across environments.

### Acceptance Demonstration

Configure a publishing node and three polling followers.

Publish a PowerShell and a C# change.

Verify source revision alignment and C# compiled-cache preparation.

Roll back a revision.

Repeat using Git-authoritative deployment mode.

Verify secrets do not appear in Git.

---

# 13. Variable Store and Secrets

## 13.1 Variable Store

**VAR-01** Provide a centralized variable store.

**VAR-02** Variables must support:

- Unique name.
- Identifier.
- Description.
- Type.
- Scope.
- Owner.
- Tags.
- RBAC.
- Version history.
- Optional expiration.
- Source/provider reference.

**VAR-03** Supported nonsecret types must include:

- String.
- Integer.
- Decimal.
- Boolean.
- JSON object.
- List/array.
- Structured values.

**VAR-04** Supported secret types must include:

- Secret string.
- Secure credential.
- Username/password pair.
- Secret binary data.
- Structured secret values.

**VAR-05** Variables must be usable from PowerShell and C# workloads.

**VAR-06** Variable availability must follow execution authorization rules.

**VAR-07** The system must support applicable global, cluster, environment, folder, object, and invocation scopes.

**VAR-08** Variable precedence and overrides must be deterministic.

**VAR-09** Caller-supplied parameters must not overwrite privileged injected variables.

## 13.2 Native Encrypted Database Secret Store

**VAR-10** The platform must provide its own database-backed secret store.

**VAR-11** Secret contents must be encrypted at rest.

**VAR-12** Encryption must use authenticated encryption with secure key management.

**VAR-13** Encryption keys must be protected separately from ordinary SQL data permissions.

**VAR-14** The service must have the database permissions necessary to read encrypted secrets and perform authorized secret-store operations.

**VAR-15** The application must be able to decrypt authorized local secrets when an execution needs them.

**VAR-16** Decryption must occur only after applicable authorization checks.

**VAR-17** Secret values must not be returned through ordinary metadata APIs.

**VAR-18** Secret-use permission must be distinct from secret-reveal permission.

**VAR-19** The store must support secret creation, update, rotation, deletion, and versioning.

**VAR-20** Backup/restore procedures must preserve recoverability without exposing encryption keys or plaintext secret contents.

**VAR-21** Database administrators' ability to read ciphertext must not automatically grant access to plaintext through the application.

## 13.3 HashiCorp Vault

**VAR-22** Native HashiCorp Vault integration is mandatory.

**VAR-23** HashiCorp AppRole authentication must be supported.

**VAR-24** HashiCorp KV version 2 must be supported.

**VAR-25** Vault configuration must support:

- Server address.
- Namespace where applicable.
- KV mount.
- Secret path.
- AppRole Role ID.
- Secure Secret ID handling.
- TLS validation.
- Authentication renewal/reauthentication.
- Connection health.
- Timeout policy.

**VAR-26** External secrets must be referenced rather than copied permanently into the platform database unless explicitly configured as an authorized local copy/cache.

**VAR-27** Vault access must occur under the configured authorized service/workload identity.

## 13.4 Per-Secret Vault Caching

**VAR-28** Vault-backed secrets must support individually configurable caching policies.

**VAR-29** Caching must be configurable independently for each secret.

**VAR-30** A secret must support disabling caching so that it is retrieved from Vault when requested.

**VAR-31** A secret must support a configured refresh interval.

**VAR-32** Automatic refresh must be able to retrieve an updated Vault value every configured interval without requiring a job or endpoint request to trigger refresh.

**VAR-33** Administrators with appropriate permissions must be able to force a manual refresh.

**VAR-34** The management API must expose an authorized manual refresh operation.

**VAR-35** Cache refresh status must include:

- Last successful refresh.
- Last attempted refresh.
- Next scheduled refresh.
- Current validity/staleness state.
- Last error category.
- Source secret version where available.

**VAR-36** The UI must never reveal cached secret contents without explicit secret-reveal authorization.

**VAR-37** Cache refresh must not require modifying or redeploying scripts.

**VAR-38** Cache storage must use encryption appropriate to the platform's secret-protection requirements.

**VAR-39** In clustered deployments, every node that may use a cached secret must have a valid, authorized means to resolve that secret.

**VAR-40** Cache consistency and refresh coordination must be documented.

**VAR-41** Stale-secret usage during Vault outages must be configurable.

**VAR-42** Fail-closed must be the default when no valid secret value is available.

**VAR-43** Explicit stale-cache policies must define maximum permitted staleness and failure behavior.

**VAR-44** Secret expiration, revocation, and rotation must invalidate cached values when required by the applicable security policy.

**VAR-45** Cache refresh failures must not crash the host or unrelated executions.

## 13.5 Custom Vault Providers

**VAR-46** The platform must support custom secret-vault providers.

**VAR-47** A documented provider extension interface must be supplied.

**VAR-48** Providers must support applicable operations such as:

- Read/resolve.
- List metadata.
- Create/update.
- Rotate.
- Delete.
- Validate connection.
- Report health.

**VAR-49** Provider capabilities must be discoverable.

**VAR-50** Provider timeouts and errors must be isolated from the hosting application.

**VAR-51** Additional enterprise vault integrations should be possible without modifying business script logic.

## 13.6 Secret Authorization and Auditing

**VAR-52** Secret RBAC must distinguish:

- Discover.
- View metadata.
- Use in execution.
- Reveal.
- Create.
- Modify.
- Rotate.
- Delete.
- Manage provider connection.
- Force cache refresh.

**VAR-53** Unattended schedules must be able to use secrets under approved workload identities.

**VAR-54** Secret access must be audited without recording values.

**VAR-55** Logs, diagnostics, telemetry, source history, and ordinary execution output must apply secret redaction controls.

**VAR-56** Because arbitrary scripts can deliberately transmit retrieved secrets, workload permissions and outbound network privileges must also be controlled.

### Acceptance Demonstration

Create typed variables and a local encrypted database secret.

Retrieve that secret using an authorized PowerShell job and C# job.

Configure HashiCorp Vault using AppRole and KV v2.

Create one noncached Vault secret and another cached with a timed refresh policy.

Change the Vault value and verify automatic cache refresh.

Force manual refresh through both UI and API.

Disable Vault and demonstrate configured stale-cache/fail-closed behavior.

Verify a user can use an approved secret without permission to reveal it.

---

# 14. Event Triggers

**TRG-01** The platform must support event-driven execution.

**TRG-02** Trigger sources must include applicable:

- Job started.
- Job completed.
- Job failed.
- Job cancelled.
- Job timed out.
- Job requires input.
- Server started/stopped.
- Node health changed.
- User login.
- API failure.
- Git synchronization.
- Deployment events.
- System health events.

**TRG-03** Triggers must support PowerShell or C# target scripts.

**TRG-04** Trigger configuration must include:

- Event source.
- Filter.
- Target script.
- Parameters.
- Run-as identity.
- Runtime environment where applicable.
- Enabled/disabled state.
- RBAC.
- Event-delivery failure handling.

**TRG-05** Event context must be provided to the triggered script.

**TRG-06** Recursive trigger storms must be prevented.

**TRG-07** Trigger failures must not crash the event-originating service.

**TRG-08** Event delivery must define deduplication, replay, and delivery semantics.

**TRG-09** Trigger source and execution failures must be independently diagnosable.

**TRG-10** Optional external webhook/message-bus event sources should be supported.

### Acceptance Demonstration

Create a trigger that executes C# when a PowerShell job fails.

Verify the event context and execution identity.

Create a second trigger that reacts to deployment changes.

Demonstrate recursive-trigger protection.

---

# 15. Administrative Management API

## 15.1 Complete UI/API Parity

**MGT-01** Every administrative operation available through the UI must have a corresponding authenticated management API operation.

**MGT-02** No ordinary management functionality may require direct database modification.

**MGT-03** API coverage must include:

- Scripts.
- Endpoints.
- Schedules.
- Triggers.
- Runtime environments.
- C# compilation and cache operations.
- Source storage modes.
- Variables.
- Secrets.
- Vaults.
- Roles.
- Tokens.
- Run-as identities.
- Folders.
- Published files.
- Clusters and nodes.
- Deployment/Git settings.
- Queues.
- Job history.
- Monitoring configuration.
- Service configuration.

## 15.2 API Behavior

**MGT-04** Management APIs must support applicable:

- Create/read/update/delete.
- List/search/filter.
- Pagination.
- Enable/disable.
- Version inspection.
- Diff.
- Validate.
- Compile.
- Publish.
- Rollback.
- Execute.
- Cancel.
- Permission management.

**MGT-05** APIs must use stable object identifiers.

**MGT-06** APIs must expose typed request and response schemas.

**MGT-07** APIs must support versioning and documented deprecation behavior.

**MGT-08** Writes must support concurrency protection.

**MGT-09** Appropriate operations must support idempotency.

**MGT-10** All management operations must enforce object-level RBAC.

**MGT-11** Administrative writes must be audited.

**MGT-12** API responses must include appropriate errors and correlation identifiers.

**MGT-13** Destructive bulk operations should support preview/dry-run behavior.

### Acceptance Demonstration

Create and deploy a C# endpoint entirely through the management API.

Create a PowerShell runtime environment, variable, role, and schedule using API calls.

Modify an object's storage mode.

Force a vault-secret refresh.

Invalidate a compiled C# cache and recompile.

Verify each action is also available in the web UI and uses the same authorization rules.

---

# 16. Administrative Web UI and Code Editor

## 16.1 Administration Interface

**UI-01** The platform must provide a modern browser-based administration interface.

**UI-02** The UI must support:

- Object navigation.
- Search.
- Folders.
- Tags.
- Execution history.
- Logs.
- Permission management.
- Runtime configuration.
- Source storage configuration.
- Git/deployment status.
- Cluster health.
- Vault management.
- Metrics configuration.

**UI-03** The UI must enforce the same RBAC decisions as the management API.

## 16.2 Source Editor

**UI-04** The platform must include a VS Code-like browser-based code editor.

**UI-05** The editor must support PowerShell and C#.

**UI-06** Editor features must include:

- Syntax highlighting.
- Bracket matching.
- Folding.
- Autocomplete.
- Parameter signatures.
- Hover information.
- Find/replace.
- Keyboard shortcuts.
- Multiple tabs.
- Source diff.
- Revision history.
- Save validation.
- Conflict detection.

## 16.3 PowerShell Language Intelligence

**UI-07** The PowerShell editor must provide live linting through a real PowerShell analysis/language service or equivalent.

**UI-08** Diagnostics must cover syntax, parsing, semantic issues, applicable module analysis, and configurable lint rules.

**UI-09** Lint severity and enabled rules must be configurable.

## 16.4 C# Language Intelligence

**UI-10** The C# editor must provide language-aware diagnostics and completion.

**UI-11** C# diagnostics must correspond to the selected runtime environment.

**UI-12** The editor must display compilation errors and warnings with source locations.

**UI-13** An explicit compile/validate action must be available.

**UI-14** The editor must display:

- Compilation status.
- Last successful compiled revision.
- Target .NET runtime.
- Source revision.
- Dependency validation status.
- Deployment/cache readiness.

**UI-15** Saving invalid C# source may preserve an editable draft, but it must not activate that source as a published revision.

## 16.5 Testing and Publishing

**UI-16** Scripts and endpoints must support integrated test execution.

**UI-17** Endpoint testing must support configurable request headers, parameters, body, and authentication context.

**UI-18** Run-as identity selection during testing must be authorized.

**UI-19** The UI must support database-backed and file-backed editing.

**UI-20** The UI must expose source conflicts instead of silently overwriting changes.

**UI-21** Publishing permissions must be separate from editing permissions.

### Acceptance Demonstration

Edit PowerShell and C# source.

Observe live PowerShell lint diagnostics and C# compile errors.

Test execution and inspect output.

Attempt concurrent edits from two sessions.

Publish a valid C# revision and verify cluster cache readiness.

Demonstrate a viewer who can read source but cannot edit or deploy.

---

# 17. API Documentation

**DOC-01** The platform must generate OpenAPI documentation for hosted endpoints.

**DOC-02** A Swagger-compatible interactive documentation viewer must be provided.

**DOC-03** Endpoint documentation must support:

- HTTP methods.
- Routes.
- Parameters.
- Request body schemas.
- Response schemas.
- Status codes.
- Authentication requirements.
- Descriptions.
- Version information.

**DOC-04** Documentation must support PowerShell and C# endpoints.

**DOC-05** Multiple documentation groups/definitions must be supported.

**DOC-06** OpenAPI JSON/YAML export must be supported.

**DOC-07** The management API must have complete versioned OpenAPI documentation.

**DOC-08** PowerShell and C# endpoint execution-context contracts must be documented.

**DOC-09** Documentation completeness must be validated during acceptance testing.

**DOC-10** Endpoint documentation must not reveal source or privileged metadata to unauthorized callers.

### Acceptance Demonstration

Create a documented PowerShell endpoint and C# endpoint.

Generate OpenAPI output and test authenticated requests through the documentation viewer.

Export the OpenAPI definition and verify its route and response schemas.

---

# 18. Cluster High Availability

## 18.1 Cluster Operation

**HA-01** Support one or more nodes per cluster.

**HA-02** Production must support four-node clusters.

**HA-03** The platform must support multiple independent clusters.

**HA-04** Nodes must share durable operational state within a cluster.

**HA-05** Work must be distributed to eligible execution nodes.

**HA-06** Ordinary jobs must be claimed by one worker unless explicitly configured for multiple/all-node execution.

## 18.2 HTTP High Availability

**HA-07** The web UI, administrative API, and hosted endpoints must support high availability behind a load balancer.

**HA-08** Readiness and health endpoints must be available.

**HA-09** Nodes must support HTTPS reverse-proxy operation.

**HA-10** WebSocket connections must be supported where required by UI functionality.

**HA-11** Authentication/session state must survive requests moving between healthy nodes or use a documented safe affinity strategy.

**HA-12** A failed node must be removed from request routing based on readiness/health policy.

## 18.3 Job Coordination

**HA-13** Job claims, leases, retries, and scheduler ownership must use durable coordination.

**HA-14** Node failure must not cause uncontrolled duplicate execution.

**HA-15** Worker recovery must respect the originating script/schedule retry policy.

**HA-16** Endpoint executions must not be transparently rerun by the platform after worker failure.

**HA-17** Job execution must account for runtime compatibility and available run-as identities.

## 18.4 Deployment and Revision Consistency

**HA-18** Nodes must report active source and configuration revisions.

**HA-19** File-backed objects must use identical published files and metadata across nodes.

**HA-20** C# compiled caches must be prepared on each eligible node.

**HA-21** Deployment failures must not silently introduce source-version inconsistency.

**HA-22** Nodes must not advertise readiness for workloads they cannot execute.

**HA-23** Cluster configuration revisions must be durable and auditable.

## 18.5 Maintenance

**HA-24** Nodes must support maintenance/drain mode.

**HA-25** Draining must stop new work assignments and handle existing executions gracefully or according to their recovery policies.

**HA-26** Rolling upgrades must be supported.

**HA-27** Incompatible schema or application versions must be detected before a node resumes service.

**HA-28** Node and cluster health must be exposed through UI, API, and monitoring.

### Acceptance Demonstration

Deploy four nodes behind a load balancer.

Run PowerShell and C# jobs and endpoints.

Remove a node during load.

Verify remaining nodes serve UI, API, and eligible jobs.

Restore the node and verify synchronization of source, metadata, runtime configuration, and compiled C# caches.

---

# 19. Fault Isolation and Resource Protection

## 19.1 Workload Isolation

**ISO-01** User-authored PowerShell and C# must execute outside the privileged application host process.

**ISO-02** Separate execution processes or equivalently demonstrated OS-level security boundaries must be used.

**ISO-03** In-process PowerShell runspaces alone do not meet the mandatory crash-containment requirement.

**ISO-04** Loading arbitrary user C# assemblies into the privileged ASP.NET/Windows Service process is prohibited.

**ISO-05** Worker failures must not terminate the web host, scheduler, or cluster coordinator.

## 19.2 Resource Limits

**ISO-06** The system must support:

- Execution timeouts.
- Memory limits.
- CPU limits.
- Process-tree limits.
- Concurrency limits.
- Queue limits.
- Output-size limits.
- Cancellation.
- Worker cleanup.

**ISO-07** A workload must not be able to exhaust the entire cluster's configured worker capacity without bounded admission controls.

**ISO-08** Execution limits must be configurable at applicable environment/object levels.

## 19.3 Process Lifecycle

**ISO-09** Workers must be supervised.

**ISO-10** Failed workers must be cleaned up and replaced when required.

**ISO-11** Orphaned processes must be detected and terminated according to policy.

**ISO-12** Worker failure must produce accurate execution records.

**ISO-13** Strict process-per-execution isolation must be supported.

**ISO-14** Warm/pooled execution may be supported for performance, but must not compromise required security and state isolation.

## 19.4 Security

**ISO-15** Workers must use least-privileged execution identities.

**ISO-16** Arbitrary modules and assemblies must not automatically receive access to privileged service internals.

**ISO-17** The system must protect against:

- Infinite loops.
- Hung calls.
- Excessive output.
- Fork bombs.
- Excessive memory allocation.
- Worker crashes.
- Uncontrolled recursive launches.
- Unexpected process exit.
- Malformed request data.

**ISO-18** Failed workloads must not expose secrets, tokens, or privileged stack traces to unauthorized users.

**ISO-19** The platform must acknowledge that OS administrator-level rights granted to arbitrary code can weaken isolation; runtime and identity policies must explicitly account for this.

### Acceptance Demonstration

Run PowerShell and C# workloads that:

- Throw uncaught exceptions.
- Exit the process.
- Exceed memory limits.
- Exceed timeouts.
- Produce excessive output.
- Start child processes.
- Fail credential initialization.

Verify that unrelated requests, jobs, scheduler operations, and administrative UI remain available within tested resource limits.

---

# 20. Windows Service Hosting

**WIN-01** The platform must run as an unattended Windows Service.

**WIN-02** The service must support installation, upgrade, removal, start, stop, and restart.

**WIN-03** Windows Server support must include documented supported OS versions.

**WIN-04** The service must support a configurable Windows service account.

**WIN-05** The service account must be separate from workload run-as identities.

**WIN-06** The service must have sufficient access to the configured operational database for normal platform operations, including authorized access to the encrypted local secret store. For SQLite local development, see the filesystem requirements in §10.5.

**WIN-07** Service configuration must support TLS certificates, ports, database connectivity, Git configuration, and vault configuration.

**WIN-08** Service configuration and persistent data must reside outside replaceable application binaries.

**WIN-09** Automatic recovery and restart policies must be supported.

**WIN-10** The platform must support graceful shutdown and job draining.

**WIN-11** Windows Event Log and centralized logging integration must be supported.

**WIN-12** gMSA support may be added but is not mandatory.

**WIN-13** The platform must support least-privileged domain or service accounts for workload execution using securely stored credentials.

### Acceptance Demonstration

Install the platform on a clean Windows Server machine.

Configure SQL Server and an encrypted secret store.

Restart the machine.

Verify the Windows Service starts, authenticates users, serves APIs, and executes PowerShell/C# workloads under separate accounts.

---

# 21. Published Files and Static Content

**FILE-01** The platform must support publishing configured directories through HTTP routes.

**FILE-02** Static content and file downloads must be supported.

**FILE-03** Published directories must support authentication and RBAC.

**FILE-04** Configurable default/index documents must be supported.

**FILE-05** File-serving configuration must prevent directory traversal and unauthorized access.

**FILE-06** Per-resource permissions must govern applicable listing, download, configuration, upload, and deletion operations.

**FILE-07** Files intended to remain available during node failure must use shared or synchronized storage.

**FILE-08** Published file revisions and authorization behavior must be consistent behind a load balancer.

**FILE-09** Windows identity-based file access should be supported where explicitly required.

### Acceptance Demonstration

Publish a protected directory.

Access it as an authorized user.

Verify unauthorized access is rejected.

Remove the serving node and verify availability from another node.

---

# 22. Monitoring, Metrics, and Audit

## 22.1 Dashboard

**OBS-01** The system must provide a cluster/system dashboard.

**OBS-02** Dashboard views must include applicable:

- Node health.
- Cluster status.
- Active deployment revision.
- Runtime compatibility.
- C# compilation/cache readiness.
- SQL connectivity.
- Vault connectivity.
- Git connectivity.
- Scheduler status.
- Running jobs.
- Queued jobs.
- Failed jobs.
- API request counts.
- API response errors.
- Worker resource use.
- Recent deployment/security events.

**OBS-03** Operational views must support filtering by cluster, node, object, folder, language, runtime, execution identity, and time range.

## 22.2 Optional Detailed Metrics

**OBS-04** Detailed metrics must be configurable at:

- Global default.
- Cluster.
- Resource type.
- Folder/group.
- Individual object.

**OBS-05** The platform must provide deterministic override precedence.

**OBS-06** Effective metrics settings must be visible in the UI and management API.

**OBS-07** Metrics for executable objects should include:

- Invocation count.
- Success/failure.
- Cancellation/timeout.
- Duration.
- Queue wait.
- Throughput.
- Retries.
- Concurrency.
- Node.
- Runtime.
- Revision.

**OBS-08** API-specific metrics must include request rates, status codes, and latency where enabled.

**OBS-09** C# compilation metrics should include compilation duration, failure count, cache-hit behavior, and cache-preparation state.

**OBS-10** Secret/vault metrics must report authorized access and refresh behavior without exposing values.

## 22.3 Mandatory Health and Audit

**OBS-11** Disabling detailed object telemetry must not disable:

- Security auditing.
- Durable job status.
- Required error records.
- Health checks.
- Deployment integrity monitoring.

**OBS-12** Administrative and security events must be audited.

**OBS-13** Audited events must include applicable:

- Authentication.
- Token creation/revocation.
- RBAC changes.
- Source edits.
- Compilation/publishing.
- Deployment.
- Secret access.
- Secret cache refresh.
- Run-as usage.
- Runtime configuration changes.
- Administrative API mutations.

**OBS-14** Audit records must be protected against unauthorized modification.

## 22.3A Automatic Configuration Change Audit

**AUD-01** Every successful create, update, delete, enable/disable, publish, rollback, permission change, and other persistent configuration mutation must automatically produce a durable audit event, regardless of whether it is initiated by the web UI, management API, Git synchronization, deployment process, system task, service token, or authorized workload.

**AUD-02** Denied attempts, failed administrative mutations, and security-relevant validation failures must be audited separately where applicable.

**AUD-03** Each configuration-change audit event must include, when applicable: UTC timestamp; stable audit ID; actor principal ID/name/type; user or service token identifier; effective/impersonated identity; authentication mechanism; operation; resource type/name/ID; resource scope; old and new version identifiers; sanitized before/after values and field-level diff; outcome and failure category; source (UI/API/Git/system); source IP and trusted proxy information; processing node/host; correlation and request IDs; approval/change-ticket reference if supplied.

**AUD-04** Secret plaintext, token material, passwords, private keys, credential contents, and unsanitized sensitive request payloads must never appear in audit records or diffs. Changes to protected data may be represented with redacted metadata or cryptographic fingerprints when safe.

**AUD-05** Configuration commits and corresponding durable audit/outbox records must be atomic. If durable auditing cannot be achieved, protected configuration changes must fail rather than succeed without an audit record.

**AUD-06** Audit records must be append-only from ordinary application permissions, access-controlled, retained according to separate audit policies, and protected from ordinary logging configuration changes.

**AUD-07** Audit history must be filterable/searchable by actor, object, action, time, IP, host, outcome, and correlation ID in both UI and management API; export must be permission-controlled.

## 22.3B Mandatory Hosted Endpoint Invocation Audit

**AUD-08** Every hosted custom endpoint invocation must produce an automatic execution audit, including requests rejected during authentication, authorization, rate limiting, or validation, as well as completed, failed, cancelled, and timed-out executions.

**AUD-09** Endpoint invocation audit records must include, where available: UTC start/end; audit ID; endpoint ID/name; matched route template; actual request path (sanitized); HTTP method; authenticated user/service principal or anonymous marker; authentication type; token ID rather than token value; client IP; validated proxy chain; receiving node/hostname/IP; executing worker node/hostname and process ID; effective Windows run-as identity; request/correlation ID; source/environment/deployment revisions; HTTP response status; duration; request and response sizes; execution outcome; error classification; user agent; and trace identifiers.

**AUD-10** Client IP resolution must honor trusted reverse-proxy configuration. Untrusted forwarded headers must never be accepted as authoritative client identity.

**AUD-11** Raw request bodies, response bodies, authorization headers, secret query parameters, and cookies must not be recorded by default. Optional diagnostic capture must require explicit security and privacy policy, redaction, size limits, and retention rules.

**AUD-12** Endpoint audit writes must be durably enqueued without requiring synchronous inserts into an external logging sink on the request path. Events must be idempotently delivered/recovered after transient failure. If neither the durable primary store nor an approved local encrypted spool can accept required audit records, the configured fail-closed behavior must prevent unaudited invocation.

**AUD-13** Endpoint audit history must be queryable by endpoint, caller, method, route, IP, host, response status, time period, and correlation ID; visibility must follow object-level RBAC.

## 22.4 Export and Alerting

**OBS-15** Metrics retention, sampling, cardinality, and aggregation must be configurable.

**OBS-16** Monitoring data must be accessible through authorized APIs.

**OBS-17** OpenTelemetry and Prometheus-compatible integrations should be supported.

**OBS-18** SIEM/log forwarding should be supported.

**OBS-19** Alert thresholds and destinations should be configurable.

## 22.5 Configurable Asynchronous Structured Logging Platform

**LOG-01** The platform must provide named, versioned, reusable logging targets, selectable by scripts, C# workloads, endpoints, environments, folders, and other authorized producers. Targets must have independent RBAC for management, use, and log viewing.

**LOG-02** Each target must specify its destination, format, typed record schema, mapped columns/fields, field defaults, optional computed/context values, validation policy, redaction policy, retention/rotation, batch/flush behavior, failure policy, and enabled state.

**LOG-03** An initial release must support SQL Server table targets and filesystem targets. File output must support at least structured JSON Lines and configurable delimited formats, with safe rotation and path management.

**LOG-04** SQL target configuration must support mapping log properties into a pre-existing table or an explicitly provisioned managed log table with configurable names and typed columns. All writes must be parameterized and validated against approved schemas.

**LOG-05** A target may optionally fan out to multiple destinations. Target selection and effective configuration must be inspectable through the UI and API.

**LOG-06** Log submissions must support structured named properties; standard severity levels; message template/text; UTC time; event IDs; exception information; and automatic correlation context (job/request IDs, script/endpoint, source revision, user, runtime, executing host and run-as identity where available).

**LOG-07** Logging must use asynchronous, bounded producer-to-sink transport without blocking individual workloads on slow destination I/O. Queue capacity, batching, retries, backoff, overflow response, and resource ceilings must be configurable.

**LOG-08** For targets requiring durability, accepted records must be persisted through a reliable queue or local encrypted spool before the producer receives durable acceptance. Delivery must support idempotency, replay after restart, dead-letter/error visibility, and an explicit loss/failure contract; unbounded in-memory fire-and-forget queues are insufficient.

**LOG-09** The platform must ship a PowerShell function named `Write-AutomationLog`, automatically accessible within platform-hosted PowerShell workers, with at least Target, Level, Message, Properties, and optional EventId/Exception inputs.

**LOG-10** The platform must ship a versioned C# library exposing `AutomationLogger` / `IAutomationLogger` (or compatible clearly documented public types) and `LogAsync`, obtainable through the authorized workload context and usable on supported modern .NET runtimes.

**LOG-11** Both language APIs must support explicit named target selection and a configurable inherited default target, without granting scripts direct access to privileged platform database connections or raw filesystem sink credentials.

**LOG-12** Administrators must be able to create, test, edit, clone, disable, and inspect targets, schema mappings, destination health, queue depth, failures, retention settings, and sampled log records via management API and UI.

**LOG-13** Logs must respect RBAC, secret redaction, file path restrictions, outbound destination policies, row/record size limits, and configurable log-volume quotas.

**LOG-14** Application/diagnostic logging is distinct from mandatory security/configuration/endpoint auditing: changing or disabling a custom target must never disable required audit events, and audit records cannot be rerouted to an unprotected arbitrary target.

**LOG-15** Logging target changes themselves must generate immutable configuration audit events.

### Acceptance Demonstration

Enable detailed metrics for one endpoint.

Disable detailed metrics for another.

Enable schedule metrics at resource-type level and override one schedule.

Produce successful and failed runs.

Verify dashboard reporting and exported metrics.

Disable detailed telemetry and verify security auditing remains active.

Change configuration through the UI, management API, and Git; verify redacted old/new diffs, actor, source, host, and request correlation; force audit persistence failure and verify the configuration mutation is not committed without its audit event.

Invoke a PowerShell endpoint and a C# endpoint successfully, unsuccessfully, and anonymously/without authorization; verify per-request audit details including client IP through a trusted proxy, executing worker host, route/method, principal, status, duration, and source revision.

Create two named logging targets with different custom columns, one SQL table target and one rotating file target. Emit structured asynchronous messages using `Write-AutomationLog` and `AutomationLogger.LogAsync`; verify mappings, redaction, retry/restart recovery, and permissions.

---

# 23. Cross-Cutting Security Requirements

## 23.1 Identity Boundaries

The platform must keep separate:

1. Caller authentication.
2. Object authorization.
3. Run-as identity authorization.
4. Runtime process identity.
5. Secret-use authorization.
6. Vault authentication.
7. Database service permissions.

No one decision must automatically grant another.

## 23.2 Secret Protection

Secrets must not appear in:

- Git repositories.
- Plaintext source metadata.
- Ordinary source history.
- API documentation.
- Routine logs.
- Standard telemetry.
- Compilation diagnostics.
- Cached source previews.
- Token listings.

Secret redaction and secure handling must extend to both PowerShell and C#.

## 23.3 Secure Communication

The platform must support:

- TLS.
- Certificate validation.
- Secure sessions/cookies.
- CSRF protection where applicable.
- Reverse-proxy trust configuration.
- Secure token validation.
- Input validation.
- Request limits.
- Audit correlation.

## 23.4 Source and Compilation Security

C# compilation must not provide arbitrary source code with privileged build-time execution access.

External package and assembly references must be subject to configured trust/dependency policies.

Compiled artifact integrity must be verifiable.

User-authored code must execute only in the intended worker security context.

## 23.5 Data Lifecycle

The platform must define retention and deletion policies for:

- Source revisions.
- Job history.
- Job output.
- Request logs.
- Metrics.
- Audit records.
- Tokens.
- Secret versions.
- Compiled artifacts.
- Deployment snapshots.

## 23.6 Version Provenance

Every executed workload must be traceable to:

- Exact source revision.
- Source storage mode.
- Language.
- Runtime environment revision.
- Runtime version.
- Dependency revision.
- Caller identity.
- Run-as identity.
- Cluster.
- Node.
- Deployment revision.

For C#, the compiled artifact identifier or content hash must also be traceable.

---

# 24. Platform Extensibility

**EXT-01** The application should use documented .NET extension interfaces for applicable subsystems.

**EXT-02** Custom vault providers are mandatory.

**EXT-03** Supported extension points may include:

- Authentication providers.
- Source storage providers.
- Secret providers.
- Execution backends.
- Event sources.
- Monitoring exporters.
- Deployment integrations.

**EXT-04** Extension failures must not crash the hosting service.

**EXT-05** Extension version compatibility must be documented.

**EXT-06** Future container-based workers and additional operating systems may be supported, but Windows Service hosting remains mandatory.

**EXT-07** Future runtime support must not require redesigning the core job/execution model.

---

# 25. Candidate Evaluation and Proof of Concept

Each requirement must be evaluated using:

- Requirement ID.
- Business priority.
- Implementation path.
- Evidence.
- Verification method.
- Score.
- Engineering effort.
- Risk.
- Owner.

## 25.1 Scoring

| Score | Definition |
|---|---|
| 5 | Native, verified in representative testing, operationally ready |
| 4 | Native/documented, minor configuration needed |
| 3 | Supported extension or manageable integration |
| 2 | Significant custom work or operational compromise |
| 1 | Partial functionality or major gap |
| 0 | Unsupported or not demonstrated |

A mandatory requirement cannot be considered satisfied solely through unverified documentation.

## 25.2 Mandatory Acceptance Gates

### Gate 1 — Windows Service and C#/.NET Host

- Install the service.
- Restart it unattended.
- Validate health reporting.
- Crash an execution worker.
- Verify the service remains operational.

### Gate 2 — Authentication

- Entra OIDC login.
- AD/LDAP integration.
- Nested directory group resolution.
- User-bound token.
- Independent service token.
- Token revocation.

### Gate 3 — Nested Object-Level RBAC

- Nested custom roles.
- Object-specific permissions.
- Different permissions for edit, execute, deploy, and use run-as.
- UI/API enforcement.
- Effective-permission inspection.

### Gate 4 — Four-Node HA

- Four application nodes.
- HTTP load balancing.
- Shared durable execution state.
- Node loss under load.
- Continued UI/API availability.
- Coordinated scheduling.

### Gate 5 — Runtime Environments

- Windows PowerShell 5.1.
- Multiple installed PowerShell 7 versions.
- Multiple installed .NET runtime versions.
- Reusable environment definitions.
- Per-object environment selection.
- Runtime eligibility validation.

### Gate 6 — C# Compilation and Caching

- C# job definition.
- C# endpoint definition.
- Compile at definition validation.
- Prevent activation on compiler errors.
- Compile/cache on each eligible node.
- Reuse compiled artifacts.
- Invalidate cache after source/dependency changes.
- Restore last valid revision.

### Gate 7 — Run-As Execution

- Script-specific run-as.
- Schedule-specific run-as.
- Endpoint-specific run-as.
- Authorized ad hoc override.
- Denied unauthorized override.
- Distinct caller and execution audit identity.

### Gate 8 — HTTP Request Context

- PowerShell endpoint.
- C# endpoint.
- Route/query/header/cookie/body parsing.
- Authenticated principal context.
- Custom response.
- Request isolation.

### Gate 9 — Per-Node Rate Limiting

- Configure through UI.
- Apply limit per node.
- Verify independent counters behind a load balancer.
- Verify appropriate limit-exceeded responses.

### Gate 10 — Retry Policies

- Script-specific retry options.
- Schedule-specific retry overrides.
- Crash recovery.
- Manual-review behavior.
- Retry chain audit.
- No automatic endpoint execution retry.

### Gate 11 — Isolation

- Worker crash.
- Process exit.
- Hung execution.
- Excessive memory.
- Excessive output.
- C# and PowerShell isolation.
- Continued service availability.

### Gate 12 — Variables and Secrets

- Typed variables.
- Encrypted SQL secret store.
- Service access to authorized local secrets.
- Secret-use versus secret-reveal permissions.
- HashiCorp AppRole authentication.
- KV v2.
- Custom vault provider.

### Gate 13 — Vault Cache

- Per-secret caching.
- Automatic timed refresh.
- Manual refresh.
- Cache status reporting.
- Vault outage behavior.
- Secret rotation.
- Secret-safe logs.

### Gate 14 — Mixed Storage

- Database-backed script.
- File-backed script.
- Database-backed endpoint.
- File-backed endpoint.
- Separate source and metadata files.
- Per-object selection.
- Migration preserving IDs and ACLs.
- Consistent files across nodes.

### Gate 15 — Git Deployment

- Single publisher/polling followers.
- Git-authoritative workflow.
- Revision tracking.
- Conflict detection.
- Rollback.
- Source and metadata consistency.

### Gate 16 — Management API Parity

- Create/edit/deploy/execute entirely through API.
- Manage roles and permissions.
- Manage runtime environments.
- Manage storage selection.
- Manage compilation.
- Manage vault caching.
- Equivalent UI/API authorization.

### Gate 17 — Web Editor

- PowerShell linting.
- C# language intelligence.
- Compilation diagnostics.
- Concurrent edit protection.
- Diff/history.
- Deployment permissions.
- Both storage modes.

### Gate 18 — Dashboard and Metrics

- System dashboard.
- Object-level telemetry control.
- Resource-type telemetry control.
- Folder/group overrides.
- Required audit preservation.
- Export and API access.

---

### Gate 19 — Automatic Configuration and Endpoint Auditing

- Every persisted configuration mutation creates a durable, redacted audit record.
- UI/API/Git changes include actor, before/after diff, host, origin, and correlation.
- Failed/denied changes are visible without exposing credentials.
- Custom endpoint audit includes user/token, method, route, IP, receiving/executing host, status, duration, and revision.
- Unavailable audit persistence cannot silently lose protected configuration changes.

### Gate 20 — Custom Logging Targets and Workload SDKs

- Create named SQL-table and file-based targets with different configured columns.
- Emit logs using PowerShell `Write-AutomationLog` and C# `AutomationLogger.LogAsync`.
- Select targets per call and through inherited default configuration.
- Verify nonblocking execution, bounded delivery, batch/retry, restart recovery, RBAC and redaction.
- Confirm disabling custom logging leaves mandatory audit trails enabled.

### Gate 21 — EF Core Migrations and Upgrade Safety

- Create a clean SQL Server database through checked-in EF Core migrations.
- Upgrade from a prior schema with the same migration chain using the authorized installer identity.
- Verify only one coordinated migration runs per cluster deployment.
- Confirm migration failure leaves the application in a controlled state with auditable diagnostics.
- Test supported rolling upgrade compatibility and database restore procedure.
- Create and upgrade a standalone SQLite database with its checked-in migrations and no SQL Server instance. Verify WAL mode and restart persistence for database-backed objects and encrypted secrets.
- Verify stale-write rejection, atomic audit and configuration writes, and duplicate-claim prevention for local jobs and schedule occurrences under concurrent access. Verify bounded failure under sustained write contention and restore a local snapshot with its encryption keys.
- Verify startup rejects production and clustered SQLite configurations, including a one-node cluster, a network database path, a second application instance targeting the same file, an incompatible schema, and failure to activate WAL.

---

# 26. Capacity and Performance Validation

The platform must provide configurable operating limits rather than relying on a fixed cluster-wide concurrency assumption.

## 26.1 Per-Node Capacity

Concurrency, worker capacity, and applicable API rate limits must be configurable at the node level.

Capacity testing must evaluate:

- HTTP requests per second per node.
- Burst traffic per node.
- Concurrent PowerShell workers.
- Concurrent C# workers.
- Memory consumption.
- CPU usage.
- Queue wait time.
- Job startup latency.
- Cold worker startup.
- Warm worker reuse.
- Database throughput.
- Vault retrieval latency.

## 26.2 C# Performance

Testing must explicitly measure:

- Definition-time compilation.
- Compilation cache creation.
- Compiled cache hit rate.
- Endpoint latency after compilation.
- Deployment compilation overhead.
- Cache recovery after node restart.
- Memory usage for multiple loaded runtime versions.

Compilation should not occur in the normal hot path of a successfully published, cached C# endpoint.

## 26.3 PowerShell Performance

Testing must account for:

- PowerShell 5.1 startup.
- Multiple PowerShell 7 runtime versions.
- Module initialization.
- Run-as account initialization.
- Worker process isolation.
- Long-running scripts.

## 26.4 Failover and Recovery

Acceptance testing must establish and document:

- Node-loss detection time.
- HTTP failover behavior.
- Job claim recovery.
- Retry behavior.
- Database reconnection.
- Git/source resynchronization.
- C# cache preparation.
- Vault recovery.
- Schedule misfire handling.

A universal exactly-once execution guarantee is not assumed for scripts with external side effects.

The system must instead provide durable tracking, duplicate prevention where possible, and configurable retry/manual-review policies.

---

# 27. Implementation Principles

The following principles apply throughout the implementation.

## 27.1 Language Parity

PowerShell and C# must be equal first-class workload languages wherever the underlying feature applies.

Neither language should require a separate management experience.

## 27.2 Separation of Control and Execution

Administrative services must not directly execute arbitrary user code within their privileged hosting processes.

## 27.3 Configuration Over Hardcoding

Cluster, node, runtime, script, endpoint, schedule, retry, secret-cache, and storage behavior must be configurable at the applicable resource scope.

## 27.4 Single Source of Truth Per Object

Even when database-backed and file-backed resources coexist, each object must have one unambiguous authoritative source.

## 27.5 Consistent Cluster Revisions

Every eligible node must operate against the same published source and metadata revision.

Node-local compiled C# caches are permitted but must correspond exactly to the expected revision.

## 27.6 Least Privilege

The service must have the permissions necessary to function, including required database and encrypted secret-store access, without automatically granting arbitrary user workloads those permissions.

## 27.7 Explicit Failure Behavior

Worker crashes, compilation failures, runtime mismatches, database outages, vault outages, and deployment conflicts must have explicit, observable outcomes.

## 27.8 Administrative API Completeness

The web application must not be the only supported means of administering any platform capability.

## 27.9 Operational Traceability

Every execution must be associated with its source revision, environment, runtime, execution identity, node, and caller.

---

# 28. Confirmed Architectural Decisions

The following decisions are confirmed requirements, not unresolved questions.

| Area | Decision |
|---|---|
| Application language | C#/.NET |
| Service hosting | Windows Services |
| Workload languages | PowerShell and C# |
| PowerShell versions | Windows PowerShell 5.1 plus any installed supported PowerShell 7 version |
| C# runtime versions | Modern .NET only, including older compatible installed modern .NET versions; legacy .NET Framework 4.x excluded |
| C# compilation | At definition validation, before activation |
| C# compiled code | Cached independently on each eligible node |
| Runtime selection | Configurable runtime environment per script/endpoint |
| Deployment | Standalone or one-to-many-node clusters |
| Cluster isolation | Independent SQL, Git, vault, configuration, and administration |
| Production cluster | Four servers, with other clusters of varying size |
| Web HA | Load-balanced |
| Database | SQL Server for production and all clusters. SQLite in WAL mode for single-instance standalone local development. |
| Database ORM/migrations | EF Core with separate version-controlled SQL Server and SQLite migration histories |
| Local secret store | Database-backed and encrypted |
| Service SQL access | Sufficient permissions for all required normal operations, including authorized secret retrieval |
| Primary identity | AD/LDAP and Entra OIDC |
| Tokens | User-bound and independent service tokens |
| Authorization | Nested, granular, per-object RBAC |
| Run-as | Configurable per object; authorized ad hoc override |
| Run-as credentials | Stored domain/service-account credentials sufficient |
| gMSA | Not required |
| Source storage | Per-object database-backed or file-backed |
| File metadata | Separate from code; structured metadata file per object type |
| Cluster file consistency | All nodes must share identical published source/metadata revisions |
| Git deployment | UI-to-Git and Git-to-platform modes |
| Current Git workflow | One publisher, followers polling/pulling |
| API rate limiting | Configurable in UI, enforced per node |
| Retries | Configurable per script and schedule |
| Endpoint retries | No automatic endpoint execution retries |
| HashiCorp Vault | Native AppRole and KV v2 |
| Vault caching | Configurable per secret |
| Vault refresh | Timed automatic refresh and manual forced refresh |
| Management API | Full UI feature parity |
| Web editor | PowerShell linting and C# diagnostics |
| Workload isolation | Mandatory for both PowerShell and C# |
| Monitoring | Dashboard plus optional metrics by object, group, and type |
| Mandatory auditing | Automatic, durable configuration-change and endpoint-invocation audits |
| Script logging | Asynchronous configurable named targets (SQL table/file), typed columns, PowerShell and C# APIs |
| PSU migration | Manual migration; automated import tooling is not required |

---

# 29. Remaining Implementation-Time Decisions

The following are engineering decisions to be documented during design and validation. They do not block the requirements specification.

1. Exact Windows Server and SQL Server version support matrix.
2. Exact supported modern .NET runtime/SDK/reference-assembly version matrix for C# workers; legacy .NET Framework 4.x is excluded.
3. C# source entry-point convention and approved dependency/package configuration format.
4. Exact worker process launch, credential handling, and OS resource-limiting mechanism.
5. Exact compiler service and compiled-artifact cache implementation.
6. Exact metadata JSON schema and repository directory conventions.
7. EF Core model/schema design, controlled migration packaging, and expand/contract upgrade strategy.
8. Cluster synchronization and deployment transaction mechanism.
9. Detailed secret-cache consistency/refresh coordination and stale-value policies.
10. Default execution, retry, and resource-limit settings.
11. Performance targets based on representative production load.
12. Backup/recovery objectives and retention periods.
13. Supported monitoring exporters and external alert integrations.
14. Initial supported extension/provider API versions.

These decisions must be made in accordance with the requirements above and must not weaken mandatory behavior.

---

# 30. Definition of Completion

The replacement is functionally complete when it can:

1. Run as a managed C#/.NET Windows Service.
2. Operate as a standalone deployment or a multi-node cluster.
3. Host a highly available web UI and HTTP API behind a load balancer.
4. Authenticate users through AD/LDAP and Microsoft Entra OIDC.
5. Enforce nested, fine-grained RBAC on all managed objects.
6. Execute PowerShell 5.1 and multiple PowerShell 7 versions.
7. Execute C# jobs and endpoints using selectable installed .NET runtimes.
8. Compile C# definitions before activation and cache compiled results on each eligible node.
9. Run PowerShell and C# in fault-isolated execution workers.
10. Support configured run-as identities and authorized ad hoc overrides.
11. Execute scripts manually, by API, by schedule, and by event trigger.
12. Enforce configurable per-node endpoint rate limits.
13. Apply independently configurable script and schedule retry policies without automatic endpoint retries.
14. Store platform data in SQL Server for production and clusters and in SQLite with WAL for single-instance standalone local development.
15. Maintain an encrypted, usable database-backed secret store with either supported provider.
16. Integrate natively with HashiCorp Vault AppRole and KV v2.
17. Cache and automatically/manual-refresh Vault secrets according to individual secret policies.
18. Support per-object database-backed and file-backed source storage.
19. Store file-backed source and metadata separately and synchronize their published revisions across cluster nodes.
20. Support both current publisher/follower Git workflows and Git-authoritative deployment.
21. Provide complete administration through both UI and REST API.
22. Provide a PowerShell/C# web editor with language intelligence, validation, history, and deployment controls.
23. Expose operational dashboards, configurable detailed metrics, durable automatic configuration-change and endpoint-execution audits, and an asynchronous customizable logging platform with PowerShell and C# logging APIs.
24. Survive failures of individual worker processes and application nodes without uncontrolled loss of unrelated functionality.
25. Preserve full execution provenance, authorization boundaries, and deployment consistency.

All mandatory acceptance gates must pass in a representative environment before the platform is considered production-ready.

The platform must use EF Core migrations for clean installation and upgrades. Manual migration of existing PowerShell Universal workloads is acceptable; automatic import tooling is out of scope. All 21 mandatory acceptance gates must pass.

**End of Specification — v4**
