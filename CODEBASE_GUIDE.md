# Woodpecker Codebase Guide

This is a reading guide and file map for the Woodpecker repository. It is
organized around the execution path rather than alphabetically.

## Scope And Legend

- The repository is primarily a Go module: `go.woodpecker-ci.org/woodpecker/v3`.
- `pipeline/` is the execution kernel shared by the server, agent, and local CLI.
- `server/` creates and schedules work, persists state, and exposes HTTP and gRPC APIs.
- `agent/` receives work over gRPC and runs it through `pipeline/runtime`.
- `web/` is the Vue frontend served by the server.
- `docs/` is a Docusaurus documentation site, including historical snapshots.
- `*_test.go` files test the package represented by the matching non-test files.
- `fixtures/`, `test-files/`, images, fonts, and generated files are listed as support artifacts, not architectural entry points.
- The repository contained 1,762 tracked files when this guide was created. The exhaustive map below accounts for them through exact paths or recursive path families, plus the four currently untracked root SVG working artifacts. The guide itself is the new root file being added.

The most useful mental model is:

```text
forge webhook
    -> server/api and server/forge
    -> server/pipeline creates and persists workflows
    -> server/scheduler and server/queue
    -> server/rpc over rpc/proto
    -> agent/runner
    -> pipeline/runtime
    -> pipeline/backend/docker, kubernetes, or local
    -> agent reports state and logs over gRPC
    -> server/rpc persists state and publishes events
    -> web REST/SSE clients display the result
```

## Ranked Reading Order

Read the following in order. The rank is about architectural leverage, not
code quality or file size.

| Rank | Read | What to learn |
| ---: | --- | --- |
| 1 | `README.md` | Product purpose, deployment shape, and vocabulary. |
| 2 | `docs/docs/92-development/05-architecture.md` | The repository's own package boundaries and intended dependencies. |
| 3 | `go.mod` | Go version, module boundary, and major runtime dependencies. |
| 4 | `Makefile` | Build targets, generated code, tests, and the supported developer workflows. |
| 5 | `pipeline/backend/types/backend.go` | The backend contract: workflow setup, step lifecycle, logs, and cleanup. |
| 6 | `pipeline/backend/types/config.go` | The backend-neutral workflow configuration passed between server and agent. |
| 7 | `pipeline/backend/types/stage.go` and `pipeline/backend/types/step.go` | The stage/step execution model and the fields that survive compilation. |
| 8 | `pipeline/frontend/yaml/types/workflow.go` | The user-facing YAML model before compilation. |
| 9 | `pipeline/frontend/yaml/parse.go` | YAML decoding, normalization, and parse errors. |
| 10 | `pipeline/frontend/metadata/substitution.go` and `pipeline/frontend/metadata/environment.go` | How forge metadata and environment values enter a workflow. |
| 11 | `pipeline/frontend/yaml/linter/linter.go` | Trust, schema, backend-option, and security validation. |
| 12 | `pipeline/frontend/yaml/compiler/compiler.go` | Conversion from YAML workflow types to backend types. |
| 13 | `pipeline/frontend/yaml/compiler/dag.go` | Dependency validation and conversion of a step DAG into sequential stages with parallel steps. |
| 14 | `pipeline/frontend/builder/builder.go` | The complete frontend pipeline: matrix expansion, substitution, parse, lint, filtering, and compilation. |
| 15 | `pipeline/runtime/runtime.go` | Runtime construction and shared execution state. |
| 16 | `pipeline/runtime/workflow.go` | Sequential stage execution, parallel steps, cancellation, and workflow cleanup. |
| 17 | `pipeline/runtime/step.go` | Step skip rules, backend calls, log draining, exit-state conversion, and tracing. |
| 18 | `pipeline/backend/docker/docker.go` | The default container backend and its implementation of the backend contract. |
| 19 | `pipeline/backend/local/local.go` | The local-process backend used by local execution and development. |
| 20 | `cmd/server/main.go`, `cmd/server/app.go`, and `cmd/server/server.go` | Server startup, configuration, listeners, HTTP serving, gRPC, and shutdown. |
| 21 | `cmd/server/setup.go` and `server/services/setup.go` | Dependency wiring for store, forge, queue, scheduler, services, and logging. |
| 22 | `server/model/pipeline.go`, `server/model/workflow.go`, `server/model/step.go`, and `server/model/task.go` | What the server persists and how a pipeline relates to workflows, steps, and queue tasks. |
| 23 | `server/store/store.go` and `server/store/datastore/xorm.go` | The storage abstraction and its XORM-backed implementation. |
| 24 | `server/router/router.go` and `server/router/api.go` | HTTP route registration and middleware composition. |
| 25 | `server/forge/forge.go` and one adapter such as `server/forge/github/github.go` | The forge interface and provider-specific behavior. |
| 26 | `server/api/hook.go` | Webhook authentication, forge parsing, repository checks, and pipeline creation entry. |
| 27 | `server/pipeline/create.go`, `server/pipeline/items.go`, and `server/pipeline/start.go` | Persistence, config fetching, secret/registry injection, builder invocation, and queueing. |
| 28 | `server/queue/queue.go`, `server/queue/fifo.go`, and `server/scheduler/impl.go` | Lease-based task delivery, label filtering, pub/sub, and queue orchestration. |
| 29 | `rpc/proto/woodpecker.proto` and `rpc/types.go` | The wire contract between agents and the server. |
| 30 | `agent/runner.go` and `agent/rpc/client_grpc.go` | Agent polling, cancellation, lease renewal, runtime invocation, retries, and log batching. |
| 31 | `server/rpc/rpc.go`, `server/rpc/auth_server.go`, and `server/rpc/authorizer.go` | Server-side RPC ownership checks and state/log ingestion. |
| 32 | `server/api/pipeline.go`, `server/api/stream.go`, and `server/pubsub/pubsub.go` | How REST and event streams expose pipeline state to clients. |
| 33 | `web/src/main.ts`, `web/src/router.ts`, `web/src/lib/api/client.ts`, and `web/src/store/pipelines.ts` | Frontend bootstrap, routing, API calls, and pipeline state. |
| 34 | `cmd/agent/main.go`, `cmd/agent/core/run.go`, and `cmd/agent/core/agent.go` | Agent process configuration and lifecycle around `agent.Runner`. |
| 35 | `cmd/cli/main.go`, `cmd/cli/app.go`, `cli/exec/exec.go`, and `woodpecker-go/woodpecker/client.go` | CLI command wiring, local execution, and the typed REST client. |
| 36 | `e2e/scenarios/suite_test.go` and representative scenario tests | The intended behavior of cancellation, concurrency, matrices, dependencies, and restart. |

### Suggested Reading Checkpoints

After ranks 1-4, you should know how to build and test the project.

After ranks 5-19, you should be able to explain how a YAML step becomes a
backend operation and why stages are sequential while steps within a stage can
run concurrently.

After ranks 20-31, you should be able to trace one pipeline from webhook to
agent and back to persisted state without reading the UI.

After ranks 32-36, you should understand how users observe and control that
pipeline, and how the repository verifies the behavior.

## Core Data Flow

### 1. Configuration To Executable Workflow

1. A forge adapter parses an incoming event into a `server/model.Pipeline`.
2. `server/pipeline/create.go` persists the pipeline and fetches one or more
   configuration files from the forge.
3. `server/pipeline/items.go` loads secrets, registries, netrc, global
   environment, trust settings, and forge metadata.
4. `pipeline/frontend/builder/builder.go` expands matrices, substitutes
   variables, parses YAML, lints it, applies `when`, and calls the compiler.
5. `pipeline/frontend/yaml/compiler/compiler.go` creates
   `pipeline/backend/types.Config`.
6. `pipeline/frontend/yaml/compiler/dag.go` groups dependency-ready steps into
   stages. The runtime executes stages in order and steps in a stage together.
7. The server persists the resulting workflows and steps as server models.

### 2. Scheduling And Execution

1. `server/pipeline/start.go` turns workflows into queue tasks.
2. `server/scheduler/impl.go` publishes a UI event and pushes tasks into the
   queue.
3. `server/rpc/rpc.go` serves a matching task to an authenticated agent.
4. `agent/runner.go` starts cancellation listening and lease renewal, then
   creates `pipeline/runtime.Runtime`.
5. `pipeline/runtime/workflow.go` calls `SetupWorkflow`, runs each stage, and
   always calls `DestroyWorkflow`.
6. `pipeline/runtime/step.go` calls `StartStep`, streams `TailStep`, waits with
   `WaitStep`, and calls `DestroyStep`.
7. A concrete backend performs the process/container/pod work.

### 3. Reporting And Observation

1. The runtime tracer sends started, skipped, finished, and error states through
   the agent RPC peer.
2. The agent batches log entries in `agent/rpc/client_grpc.go` and sends them
   over the protobuf `Log` method.
3. `server/rpc/rpc.go` validates ownership, updates steps and workflows,
   stores logs, updates pipeline status, and publishes events.
4. REST handlers in `server/api/` serve snapshots. Stream handlers and pub/sub
   provide live updates consumed by the Vue application.

## Repository Map

### Root Files

| Path | Purpose |
| --- | --- |
| `.cspell.json` | Spell-check vocabulary and exclusions. |
| `.editorconfig` | Editor indentation and newline conventions. |
| `.editorconfig-checker.json` | EditorConfig checker settings. |
| `.gitattributes` | Git attributes and generated/binary handling. |
| `.gitignore` | Ignored build, dependency, coverage, and local files. |
| `.gitpod.yml` | Gitpod development environment configuration. |
| `.golangci.yaml` | Go lint configuration. |
| `.hadolint.yaml` | Dockerfile lint configuration. |
| `.lycheeignore` | Link checker exclusions. |
| `.markdownlint.yaml` | Markdown lint configuration. |
| `.mockery.yaml` | Mockery generation configuration. |
| `.pre-commit-config.yaml` | Pre-commit hooks and repository checks. |
| `.prettierignore` and `.prettierrc.json` | Prettier exclusions and formatting rules. |
| `.trivyignore` | Container/security scanner exceptions. |
| `.yamllint.yaml` | YAML lint configuration. |
| `checkmake.ini` | Makefile lint configuration. |
| `codecov.yaml` | Codecov reporting configuration. |
| `CHANGELOG.md` | Release history. |
| `LICENSE` | Apache 2.0 project license. |
| `README.md` | Product overview, resources, and high-level deployment information. |
| `CODEBASE_GUIDE.md` | This architecture reading order and repository file map. |
| `Makefile` | Canonical build, test, lint, generation, packaging, and release tasks. |
| `go.mod` and `go.sum` | Go module metadata and dependency checksums. |
| `flake.nix` and `flake.lock` | Reproducible Nix development environment and locked inputs. |
| `release-config.ts` | Release automation configuration. |
| `tools/tools.go` | Build-tagged declaration of Go tooling dependencies. |
| `version/version.go` | Version value populated or overridden at build time. |
| `docker-compose.example.yaml` | Example server/agent deployment. |
| `docker-compose.gitpod.yaml` | Gitpod-specific Compose deployment. |
| `contrib/woodpecker-test-repo/.woodpecker/demo.yaml` | Example pipeline for the test repository. |
| `contrib/woodpecker-test-repo/.woodpecker/test.yaml` | Test pipeline for the test repository. |

### CI, GitHub, And Packaging

| Path family | Purpose |
| --- | --- |
| `.woodpecker/binaries.yaml` | Builds release binaries. |
| `.woodpecker/check-feature-docs.sh` | Checks feature documentation coverage. |
| `.woodpecker/docker.yaml` | Container image pipeline. |
| `.woodpecker/docs.yaml` | Documentation build and publishing pipeline. |
| `.woodpecker/links.yaml` | Link validation pipeline. |
| `.woodpecker/release-helper.yaml` | Release helper pipeline. |
| `.woodpecker/securityscan.yaml` | Security scanning pipeline. |
| `.woodpecker/social.yaml` | Social/release announcement automation. |
| `.woodpecker/static.yaml` | Static analysis and formatting pipeline. |
| `.woodpecker/test.yaml` | Go and integration test pipeline. |
| `.woodpecker/web.yaml` | Web frontend pipeline. |
| `.github/ISSUE_TEMPLATE/*` | Bug and feature issue forms plus issue configuration. |
| `.github/pull_request_template.md` | Pull request checklist/template. |
| `.github/release_template.md` | Release notes template. |
| `.github/renovate.json` | Dependency update policy. |
| `.vscode/extensions.json` | Recommended editor extensions. |
| `.vscode/launch.json` | Debug launch configurations. |
| `.vscode/settings.json` | Workspace editor settings. |
| `docker/Dockerfile.*` | Multi-architecture images for server, agent, CLI, and build tooling. |
| `nfpm/agent.yaml`, `nfpm/cli.yaml`, `nfpm/server.yaml` | Debian/RPM package definitions. |
| `nfpm/*.env.example` | Example service environment files. |
| `nfpm/*.service` | systemd service definitions. |
| `nfpm/woodpecker-system-user.preinstall.sh` | Package preinstall user setup. |

### `pipeline/`: Shared CI/CD Engine

This is the most important subsystem after the entry points. It is deliberately
independent of the server database and HTTP API.

| Path family | Contents and responsibility |
| --- | --- |
| `pipeline/const.go` | Internal labels and constants shared by pipeline packages. |
| `pipeline/backend/backend.go` | Backend package-level contract and backend construction boundary. |
| `pipeline/backend/types/auth.go`, `config.go`, `conn.go`, `errors.go`, `network.go`, `secret.go`, `stage.go`, `state.go`, `step.go` | Backend-neutral data structures for auth, workflow configuration, connections, errors, networks, secrets, stage state, and steps. |
| `pipeline/backend/types/mocks/mock_Backend.go` | Generated mock of the backend interface. |
| `pipeline/backend/common/script.go`, `script_posix.go`, `script_win.go` | Cross-platform script generation and shell behavior. |
| `pipeline/backend/common/*_test.go` | Script generation tests on POSIX and Windows behavior. |
| `pipeline/backend/docker/backend_options.go`, `config.go`, `convert.go`, `convert_win.go`, `docker.go`, `errors.go`, `flags.go` | Docker backend options, config conversion, platform-specific command conversion, lifecycle operations, errors, and flags. |
| `pipeline/backend/docker/*_test.go` | Docker option, conversion, and behavior tests. |
| `pipeline/backend/kubernetes/backend_options.go`, `flags.go`, `kubernetes.go`, `namespace.go`, `pod.go`, `secrets.go`, `service.go`, `utils.go`, `volume.go` | Kubernetes backend options, client/lifecycle orchestration, namespaces, pods, secrets, services, utilities, and volumes. |
| `pipeline/backend/kubernetes/*_test.go` | Kubernetes resource and option tests, including node selector behavior. |
| `pipeline/backend/local/clone.go`, `cmd_unix.go`, `cmd_win.go`, `command.go`, `const.go`, `errors.go`, `flags.go`, `local.go`, `plugin.go` | Local process backend, clone behavior, platform commands, process groups, plugin execution, and local flags/errors. |
| `pipeline/backend/local/*_test.go` | Local process, command, cleanup, and process-group tests. |
| `pipeline/backend/dummy/dummy.go` and `dummy_test.go` | Deterministic backend used by tests and examples. |
| `pipeline/errors/linter.go`, `pipeline.go`, `runtime.go` | Typed diagnostics for validation, pipeline construction, and execution. |
| `pipeline/errors/*_test.go` | Error classification and formatting tests. |
| `pipeline/frontend/metadata/const.go`, `types.go`, `environment.go`, `substitution.go` | Forge metadata types, environment construction, and variable substitution. |
| `pipeline/frontend/metadata/drone_compatibility.go` | Compatibility environment values for Drone-style plugins. |
| `pipeline/frontend/metadata/*_test.go` and `fuzz_test.go` | Metadata, substitution, environment, and compatibility tests/fuzzing. |
| `pipeline/frontend/yaml/parse.go` | Entry point for parsing YAML into the user-facing workflow model. |
| `pipeline/frontend/yaml/types/workflow.go`, `concurrency.go`, `container.go`, `container_list.go`, `network.go`, `volume.go`, and `base/*` | YAML schema model and reusable scalar/list types. |
| `pipeline/frontend/yaml/types/*_test.go` | YAML type decoding and validation tests. |
| `pipeline/frontend/yaml/constraint/constraint.go`, `depends_on.go`, `list.go`, `map.go`, `path.go`, `skip.go` | `when`, dependency, list/map, path, and skip-ci constraint evaluation. |
| `pipeline/frontend/yaml/constraint/*_test.go` and `fuzz_test.go` | Constraint behavior and fuzz tests. |
| `pipeline/frontend/yaml/linter/linter.go`, `option.go`, `error.go` | Workflow lint orchestration, options/trust configuration, and lint errors. |
| `pipeline/frontend/yaml/linter/schema/schema.go`, `schema.json` | Embedded/generated schema definition used by YAML validation. |
| `pipeline/frontend/yaml/linter/schema/.woodpecker/*.yaml` | Schema/linter fixture pipelines covering valid and invalid syntax. |
| `pipeline/frontend/yaml/linter/*_test.go` and `schema/*_test.go` | Linter and schema tests/fuzzing. |
| `pipeline/frontend/yaml/matrix/matrix.go` | Matrix axis parsing and expansion. |
| `pipeline/frontend/yaml/matrix/*_test.go` and `fuzz_test.go` | Matrix tests and fuzzing. |
| `pipeline/frontend/yaml/compiler/compiler.go`, `convert.go`, `option.go`, `errors.go` | Main YAML-to-backend compiler, conversion helpers, compiler options, and compiler errors. |
| `pipeline/frontend/yaml/compiler/dag.go` | Dependency graph validation and stage construction. |
| `pipeline/frontend/yaml/compiler/settings/params.go` | Compiler settings and parameter parsing. |
| `pipeline/frontend/yaml/compiler/*_test.go` and `settings/*_test.go` | Compiler, DAG, option, conversion, and settings tests. |
| `pipeline/frontend/builder/builder.go`, `types.go`, `utils.go` | Server/CLI-facing builder that combines YAML, metadata, trust, secrets, registries, matrices, linting, and compilation. |
| `pipeline/frontend/builder/*_test.go` | Builder and item conversion tests. |
| `pipeline/runtime/runtime.go`, `option.go`, `shutdown.go`, `workflow.go`, `step.go` | Runtime construction, options, shutdown context, stage/workflow execution, and step lifecycle. |
| `pipeline/runtime/*_test.go` and `helpers_test.go` | Runtime concurrency, cancellation, state, and helper tests. |
| `pipeline/state/state.go` | Runtime state aggregate passed to tracers. |
| `pipeline/tracing/tracer.go`, `mocks/mock_Tracer.go` | State reporting interface and generated mock. |
| `pipeline/logging/logger.go` | Pipeline log stream adapter. |
| `pipeline/shared/replace_secrets.go` | Redaction of secret values in output. |
| `pipeline/shared/*_test.go` | Secret replacement tests. |
| `pipeline/utils/copy_line_by_line.go` and its test | Line-oriented stream copying utility. |

### `server/`: Control Plane

#### Server Startup And Configuration

| Path | Purpose |
| --- | --- |
| `server/config.go` | Global server configuration and service references. |
| `cmd/server/main.go` | Process entry point, dotenv loading, signal context, and app execution. |
| `cmd/server/app.go` | CLI application definition and server command registration. |
| `cmd/server/flags.go` | Server command flags and environment-backed options. |
| `cmd/server/server.go` | Store setup, service startup, HTTP/TLS listeners, web serving, metrics, and graceful shutdown. |
| `cmd/server/setup.go` | Server dependency assembly and global service setup. |
| `cmd/server/grpc_server.go` | gRPC listener and server registration. |
| `cmd/server/health.go` | Server health command/endpoint support. |
| `cmd/server/man.go` | Man-page build-tag entry. |
| `cmd/server/openapi.go`, `openapi/docs.go`, `openapi_json_gen.go` | OpenAPI registration, generated specification, and specification generator. |
| `cmd/server/*_test.go` | Server flags, setup, OpenAPI, and lifecycle tests. |
| `server/services/setup.go`, `manager.go`, `mocks/mock_Manager.go` | Service manager interfaces, construction, and generated mock. |
| `server/services/config/*` | Configuration service and forge/repository configuration selection. |
| `server/services/encryption/*` | AES/Tink encryption implementations, state, keysets, builders, wrappers, and interfaces. |
| `server/services/environment/*` | Global environment parsing and service. |
| `server/services/log/*` | Log service, file store, and addon log integration. |
| `server/services/permissions/*` | Admin, organization, and repository ownership permission checks. |
| `server/services/registry/*` | Registry lookup and composition across database, filesystem, HTTP, and extensions. |
| `server/services/secret/*` | Secret lookup and composition across database, HTTP, and extensions. |
| `server/services/utils/*` | Shared service HTTP helpers and host matching. |
| `server/services/**/mocks/*` | Generated mocks for service interfaces. |
| `server/services/**/*_test.go` | Unit tests for each service and service composition. |

#### HTTP API And Routing

| Path family | Purpose |
| --- | --- |
| `server/router/router.go` | Base Gin router and middleware composition. |
| `server/router/api.go` | REST route registration and API grouping. |
| `server/router/middleware/logger.go`, `header/header.go`, `store.go`, `version.go` | Request logging, security/response headers, store injection, and version middleware. |
| `server/router/middleware/session/*` | Repository, organization, user, agent, and pagination session loading. |
| `server/router/middleware/token/token.go` | Token authentication middleware. |
| `server/api/hook.go` | Forge webhook authentication and pipeline trigger path. |
| `server/api/pipeline.go` | Pipeline REST operations and status/control actions. |
| `server/api/stream.go` | Live event and log stream endpoints. |
| `server/api/agent.go`, `repo.go`, `org.go`, `user.go`, `users.go`, `forge.go` | Agent, repository, organization, user, user administration, and forge endpoints. |
| `server/api/cron.go`, `queue.go`, `metrics/prometheus.go` | Cron, queue, and metrics endpoints. |
| `server/api/*_secret.go`, `*_registry.go`, `global_*` and `org_*` variants | Repository, organization, and global secret/registry APIs. |
| `server/api/login.go`, `signature_public_key.go`, `badge.go`, `debug/debug.go`, `helper.go`, `z.go` | Login, signing key, badge, debug, API helper, and miscellaneous endpoint support. |
| `server/api/**/*_test.go` | API handler and SQLite helper tests. |

#### Models And Persistence

| Path family | Purpose |
| --- | --- |
| `server/model/agent.go`, `commit.go`, `config.go`, `const.go`, `cron.go`, `environ.go`, `event.go`, `feed.go`, `forge.go`, `log.go`, `netrc.go`, `org.go`, `pagination.go`, `perm.go`, `pipeline.go`, `pull_request.go`, `queue.go`, `redirection.go`, `registry.go`, `repo.go`, `secret.go`, `server_config.go`, `step.go`, `task.go`, `team.go`, `user.go`, `workflow.go` | Database/API domain models and status/value constants. |
| `server/model/*_test.go` and `fuzz_test.go` | Model parsing, pagination, persistence shape, and fuzz tests. |
| `server/store/store.go`, `common.go`, `context.go`, `types/errors.go` | Store interface, common store helpers, request context access, and store errors. |
| `server/store/datastore/*.go` | XORM datastore operations for every model and datastore initialization. |
| `server/store/datastore/*_test.go` | Datastore operation and engine tests. |
| `server/store/datastore/migration/migration.go`, `common.go`, `logger.go` | Migration runner, shared migration helpers, and migration logging. |
| `server/store/datastore/migration/000_*.go` through `029_*.go` | Ordered schema/data migrations. Read these for historical persistence constraints, not first-pass architecture. |
| `server/store/datastore/migration/*_test.go` | Migration behavior and migration helper tests. |
| `server/store/datastore/migration/test-files/postgres.sql` | PostgreSQL migration fixture. |
| `server/store/datastore/migration/test-files/sqlite.db` | SQLite binary migration fixture. |
| `server/store/datastore/migration/test-files/.gitignore` | Fixture-directory ignore rules. |
| `server/store/mocks/mock_Store.go` | Generated store mock. |

#### Pipeline Orchestration, Queue, And Events

| Path family | Purpose |
| --- | --- |
| `server/pipeline/create.go` | Creates a pipeline, fetches and persists config, builds workflows, and starts work. |
| `server/pipeline/items.go` | Injects metadata/secrets/registries and calls the shared builder; persists workflows. |
| `server/pipeline/start.go`, `queue.go` | Converts workflows into tasks and starts/cancels queue work. |
| `server/pipeline/status.go`, `pipeline_status.go`, `workflow_status.go`, `step_status.go` | Pipeline/workflow/step status transitions and aggregation. |
| `server/pipeline/approve.go`, `decline.go`, `gated.go` | Approval-gated pipeline behavior. |
| `server/pipeline/cancel.go`, `restart.go` | Cancellation and restart operations. |
| `server/pipeline/config.go`, `errors.go`, `helper.go` | Pipeline config/error/helper logic. |
| `server/pipeline/metadata/*` | Server-specific metadata construction from forge, repository, and pipeline state. |
| `server/pipeline/*_test.go` and `metadata/*_test.go` | Orchestration, status, gated, queue, restart, and metadata tests. |
| `server/queue/queue.go`, `fifo.go`, `persistent.go` | Queue interface, in-memory FIFO implementation, and persistent task/lease behavior. |
| `server/queue/mocks/mock_Queue.go` | Generated queue mock. |
| `server/queue/*_test.go` | Queue ordering, lease, persistence, and cancellation tests. |
| `server/scheduler/scheduler.go`, `impl.go`, `filter.go`, `mocks/mock_Scheduler.go` | Scheduler interface, queue/pubsub implementation, agent label filters, and mock. |
| `server/scheduler/*_test.go` | Scheduler and filter tests. |
| `server/pubsub/pubsub.go`, `memory/pub.go` | Pub/sub contract and in-memory implementation used for live UI events. |
| `server/pubsub/**/*_test.go` | Pub/sub tests. |

#### Forge Integrations

| Path family | Purpose |
| --- | --- |
| `server/forge/forge.go`, `refresh.go` | Forge abstraction, registration, repository refresh, and common lifecycle. |
| `server/forge/types/*` | Forge errors, OAuth values, and normalized metadata types. |
| `server/forge/common/*` | Shared event normalization, commit status, and forge utilities. |
| `server/forge/setup/*` | Forge service setup and tests. |
| `server/forge/addon/*` | External forge addon client/server and argument protocol. |
| `server/forge/github/*` | GitHub adapter, conversion, webhook parsing, status behavior, and fixtures. |
| `server/forge/gitlab/*` | GitLab adapter, conversion, status behavior, webhook parsing, and fixtures. |
| `server/forge/gitea/*` | Gitea adapter, webhook parsing, conversion, helpers, and fixtures. |
| `server/forge/forgejo/*` | Forgejo adapter, webhook parsing, conversion, helpers, and fixtures. |
| `server/forge/bitbucket/*` | Bitbucket adapter, conversion, parsing, internal client/types, and fixtures. |
| `server/forge/bitbucketdatacenter/*` | Bitbucket Data Center adapter, conversion, parsing, internal client/types, and fixtures. |
| `server/forge/mocks/*` | Generated forge and refresher mocks. |
| `server/forge/**/*_test.go` and `fuzz_test.go` | Adapter, conversion, status, helper, and webhook parser tests/fuzzing. |
| `server/forge/**/fixtures/*.json` | Recorded forge webhook payloads and expected responses. |
| `server/forge/**/fixtures/*.go` | Fixture loading, mock HTTP handlers, and fixture helpers. |

#### Supporting Server Packages

| Path family | Purpose |
| --- | --- |
| `server/badges/*.go` | SVG pipeline badges, colors, styles, drawing, and embedded font support. |
| `server/badges/fonts/DejaVuSans.ttf` | Embedded badge font binary. |
| `server/badges/**/*_test.go` | Badge generation tests. |
| `server/ccmenu/*.go` | CI menu XML generation. |
| `server/ccmenu/*_test.go` | CI menu tests. |
| `server/cron/cron.go` | Cron scheduling service. |
| `server/cron/*_test.go` and `fuzz_test.go` | Cron scheduling tests and fuzzing. |
| `server/cache/membership.go` | Membership cache. |
| `server/cache/*_test.go` | Membership cache tests. |
| `server/logging/LICENSE`, `log.go`, `logging.go` | Server-side streamed logging primitives and license. |
| `server/logging/*_test.go` | Logging tests. |
| `server/metric/metrics_server.go` | Server Prometheus metrics definitions. |
| `server/web/config.go`, `web.go`, `web_test.go` | Server-side web configuration and SPA/static serving. |

#### Agent RPC Server

| Path | Purpose |
| --- | --- |
| `server/rpc/server.go`, `serve.go` | gRPC server construction, registration, and serving. |
| `server/rpc/auth_server.go` | Agent authentication RPC. |
| `server/rpc/authorizer.go`, `jwt_manager.go` | Agent authorization and JWT lifecycle. |
| `server/rpc/rpc.go` | Core agent RPC implementation for poll, init, update, logs, lease, cancellation, and done. |
| `server/rpc/sanitize.go` | Validation/sanitization of agent-submitted state. |
| `server/rpc/errors.go` | RPC-specific errors. |
| `server/rpc/*_test.go` | Auth, authorization, JWT, integration, state, sanitization, and server tests. |

### `rpc/`: Shared Wire-Level Types

| Path | Purpose |
| --- | --- |
| `rpc/proto/woodpecker.proto` | Source of truth for server/agent gRPC services and messages. |
| `rpc/proto/woodpecker.pb.go` | Generated protobuf message types. |
| `rpc/proto/woodpecker_grpc.pb.go` | Generated gRPC client/server interfaces. |
| `rpc/proto/generate.go` | Protobuf generation directive/tool entry. |
| `rpc/proto/version.go` | Protocol compatibility version. |
| `rpc/peer.go` | Client/server-independent RPC peer interface used by the agent runner. |
| `rpc/types.go` | Shared workflow, filter, state, version, and log types. |
| `rpc/log_entry.go` | Log entry conversion/helpers. |
| `rpc/mocks/mock_Peer.go` | Generated RPC peer mock. |
| `rpc/*_test.go` | Log and peer contract tests. |

### `agent/`: Remote Worker

| Path family | Purpose |
| --- | --- |
| `agent/runner.go` | Main worker loop: polls, tracks timeout/cancel/lease, invokes runtime, and reports completion. |
| `agent/state.go` | Concurrent state for active workflows. |
| `agent/tracer.go` | Runtime tracer that converts execution state into RPC updates. |
| `agent/logger.go` | Runtime log adapter that sends logs to the server. |
| `agent/log/line_writer.go` | Converts byte streams into line log entries. |
| `agent/**/*_test.go` | Runner, state, tracing, logging, and line writer tests. |
| `agent/rpc/dial.go` | gRPC connection setup. |
| `agent/rpc/auth_client_grpc.go`, `auth_interceptor.go` | Agent authentication and access-token interceptor. |
| `agent/rpc/client_grpc.go` | RPC peer implementation, retries, polling, state updates, log batching, and lease calls. |
| `agent/rpc/**/*_test.go` | Agent RPC authentication, dialing, retry, and client tests. |

### `cmd/agent/`: Agent Process

| Path | Purpose |
| --- | --- |
| `cmd/agent/main.go` | Agent process entry point. |
| `cmd/agent/core/agent.go` | Agent service lifecycle and runner construction. |
| `cmd/agent/core/config.go` | Agent configuration values. |
| `cmd/agent/core/flags.go` | Agent CLI/environment flags. |
| `cmd/agent/core/run.go` | Polling loop and worker capacity management. |
| `cmd/agent/core/health.go` | Agent health endpoint/check. |
| `cmd/agent/dummy.go` | Dummy-agent build/test support. |
| `cmd/agent/man.go` | Man-page build-tag entry. |
| `cmd/agent/**/*_test.go` | Agent configuration, health, lifecycle, and runner tests. |

### `cli/` And `cmd/cli/`: Command-Line Client

| Path family | Purpose |
| --- | --- |
| `cmd/cli/main.go`, `app.go` | CLI process entry and root command assembly. |
| `cmd/cli/docs.go`, `man.go` | Generated command docs and man-page build-tag entries. |
| `cli/README.md` | CLI-specific overview. |
| `cli/common/*` | Shared command flags, hooks, pipeline helpers, and CLI logging. |
| `cli/internal/config/*` | Context/config file loading and credential persistence. |
| `cli/internal/util.go` | Internal HTTP/client helpers. |
| `cli/output/*` | Tables and output formatting. |
| `cli/context/context.go` | Manage named server contexts. |
| `cli/setup/*` | Interactive first-time setup and token fetching. |
| `cli/lint/*` | Local YAML lint command using the shared linter. |
| `cli/exec/exec.go`, `flags.go`, `metadata.go`, `line.go`, `dummy.go` | Local pipeline execution path using the shared builder/runtime and selected backend. |
| `cli/exec/*_test.go` | Local execution and metadata tests. |
| `cli/admin/*` | Server administration commands for users, organizations, secrets, registries, and log level. |
| `cli/org/*` | Organization resource commands. |
| `cli/repo/*` | Repository, cron, secret, and registry commands. |
| `cli/pipeline/*` | Pipeline creation, listing, display, logs, approval, cancellation, restart, queue, and deployment commands. |
| `cli/update/*` | CLI self-update command, archive handling, types, and updater tests. |
| `cli/**/*_test.go` | Command behavior, config, output, and updater tests. |

The repeated `admin/{secret,registry,user}`, `org/{secret,registry}`,
`repo/{secret,registry,cron}`, and `pipeline/{log,deploy}` files follow a
consistent command pattern: the directory-level file creates the command
group and the verb files implement add/list/show/set/remove or the relevant
pipeline operation.

### `woodpecker-go/`: Typed REST Client

| Path | Purpose |
| --- | --- |
| `woodpecker-go/README.md` | Client library usage and scope. |
| `woodpecker-go/LICENSE` | Client library license. |
| `woodpecker-go/woodpecker/interface.go` | Public REST client interface. |
| `woodpecker-go/woodpecker/client.go` | HTTP client, authentication, request, and response handling. |
| `woodpecker-go/woodpecker/types.go`, `const.go`, `list_options.go` | Shared API types, constants, and list query options. |
| `woodpecker-go/woodpecker/agent.go`, `global_secret.go`, `global_registry.go`, `org.go`, `pipeline.go`, `queue.go`, `repo.go`, `user.go` | Typed endpoint methods grouped by resource. |
| `woodpecker-go/woodpecker/httputil/*` | User-agent/request helper. |
| `woodpecker-go/woodpecker/mocks/mock_Client.go` | Generated client mock. |
| `woodpecker-go/woodpecker/*_test.go` | Client, resource endpoint, and list option tests. |

### `web/`: Vue Frontend

| Path family | Purpose |
| --- | --- |
| `web/package.json` | Frontend scripts and dependencies. |
| `web/pnpm-lock.yaml`, `pnpm-workspace.yaml` | Locked frontend dependencies/workspace. |
| `web/index.html` | Vite HTML entry. |
| `web/vite.config.ts`, `tsconfig.json`, `eslint.config.js` | Build, TypeScript, and lint configuration. |
| `web/.gitignore`, `.prettierignore`, `.prettierrc.js`, `.yamlignore`, `LICENSE` | Frontend repository settings and license. |
| `web/components.d.ts` | Generated component auto-registration metadata. |
| `web/web.go` | Embeds the built SPA for server use. |
| `web/web_external.go` | Serves an externally configured frontend directory. |
| `web/public/favicons/*` | Theme/status-specific PNG and SVG browser icons. |
| `web/src/main.ts` | Vue application bootstrap, plugins, and root mount. |
| `web/src/App.vue` | Root application shell. |
| `web/src/router.ts` | Route definitions and navigation guards. |
| `web/src/lib/api/client.ts`, `index.ts` | Typed API client setup and exports. |
| `web/src/lib/api/types/*.ts` | API response/request types for agents, crons, forges, organizations, pipelines, queues, registries, repositories, secrets, users, and webhooks. |
| `web/src/lib/pipeline.ts` | Pipeline display/status helpers. |
| `web/src/lib/utils/*` | Generic frontend utilities and tests. |
| `web/src/compositions/*.ts` | Reusable Composition API logic for auth, API calls, config, events, pagination, pipelines, repositories, themes, notifications, tabs, and time. `*.test.ts` files test individual compositions. |
| `web/src/store/pipelines.ts`, `repos.ts` | Pinia pipeline and repository stores. |
| `web/src/components/atomic/*` | Small reusable UI primitives, icons, buttons, warnings, markdown, and syntax highlighting. |
| `web/src/components/form/*` | Form controls, field types, list/key-value editors, and form tests. |
| `web/src/components/layout/*` | Containers, panels, popups, settings, navigation, headers, tabs, and scaffold layout. |
| `web/src/components/agent/*` | Agent list, form, and management UI. |
| `web/src/components/registry/*`, `secrets/*` | Registry and secret editing/list UI. |
| `web/src/components/repo/*` | Repository cards and pipeline list/log/status components. |
| `web/src/components/pipeline-feed/*` | Pipeline feed item/sidebar components. |
| `web/src/components/admin/settings/*` | Admin forge form and queue statistics. |
| `web/src/components/FileTree.vue` | Pipeline/config file tree display. |
| `web/src/views/Login.vue`, `NotFound.vue`, `Repos.vue`, `RepoAdd.vue`, `RouterView.vue` | Login, fallback, repository index/add, and route shell views. |
| `web/src/views/admin/*` | Admin agents, users, repositories, organizations, registries, secrets, queue, info, settings, and forge views. |
| `web/src/views/org/*` | Organization overview, repositories, redirects, and settings views. |
| `web/src/views/repo/*` | Repository overview, branches, pull requests, manual pipelines, pipeline details, and settings views. |
| `web/src/views/user/*` | User profile, agents, secrets, registries, CLI/API, and settings views. |
| `web/src/views/cli/Auth.vue` | CLI authentication view. |
| `web/src/vite-env.d.ts` | Vite-provided TypeScript environment declarations. |
| `web/src/assets/locales/*.json` | Localized strings. `en.json` is the primary reference locale; the other locale files are translations. |
| `web/src/assets/logo.svg`, `woodpecker.svg` | Frontend branding assets. |
| `web/src/style.css`, `tailwind.css`, `style/console.css`, `style/prism.css` | Global, Tailwind, console, and code highlighting styles. |

### `shared/`: Cross-Cutting Go Utilities

| Path family | Purpose |
| --- | --- |
| `shared/constant/constant.go` | Shared timing, environment, and protocol constants. |
| `shared/dot_env/dot_env.go` | Optional dotenv loading. |
| `shared/httputil/*` | HTTP errors, request helpers, and user-agent support. |
| `shared/logger/*` | Shared zerolog setup, terminal formatting, and addon logging. |
| `shared/optional/*` | Optional value type and JSON/YAML serialization. |
| `shared/token/token.go` | Token parsing/creation and token types. |
| `shared/token/*_test.go` and `fuzz_test.go` | Token tests and fuzzing. |
| `shared/utils/context.go`, `paginate.go`, `protected.go`, `slices.go`, `strings.go` | Context signals, pagination, protected values, slice, and string helpers. |
| `shared/**/*_test.go` | Utility tests. |

### `e2e/`: End-To-End Behavior

| Path family | Purpose |
| --- | --- |
| `e2e/setup/server.go`, `agent.go`, `forge.go`, `store.go`, `wait.go` | Starts and coordinates a real test server, agent, forge, store, and wait conditions. |
| `e2e/scenarios/suite_test.go` | E2E suite setup and common scenario harness. |
| `e2e/scenarios/agent_routing_test.go` | Agent label routing. |
| `e2e/scenarios/cancel_test.go` | Cancellation propagation. |
| `e2e/scenarios/concurrency_test.go` | Concurrency limits and parallel execution. |
| `e2e/scenarios/depends_on_ordering_test.go` | Dependency ordering. |
| `e2e/scenarios/matrix_test.go` | Matrix expansion behavior. |
| `e2e/scenarios/gated_test.go` | Approval-gated pipelines. |
| `e2e/scenarios/infra_test.go` | Infrastructure and service behavior. |
| `e2e/scenarios/restart_test.go` | Restart behavior. |
| `e2e/scenarios/fixtures/*.yaml`, `*.json`, and numbered fixture directories | Input pipeline configurations and expected scenario results. The numbered cases cover success, failures, ignored failures, services, parallel steps, OOM, multi-workflow dependencies, optional dependencies, and missing required dependencies. |

### `docs/`: Documentation Application

#### Active Documentation

| Path family | Purpose |
| --- | --- |
| `docs/docusaurus.config.ts`, `sidebars.js`, `versions.json` | Docusaurus site configuration, navigation, and version list. |
| `docs/package.json`, `pnpm-lock.yaml`, `pnpm-workspace.yaml`, `tsconfig.json` | Documentation site dependencies and TypeScript/workspace configuration. |
| `docs/README.md`, `LICENSE` | Documentation project overview and license. |
| `docs/docs/10-intro/*` | Introduction. |
| `docs/docs/20-usage/*` | Workflow syntax, matrices, secrets, registries, cron, environment, plugins, services, volumes, extensions, linting, local execution, settings, badges, and advanced usage. |
| `docs/docs/30-administration/*` | Installation, server/agent configuration, backends, forges, addons, and autoscaling. |
| `docs/docs/92-development/*` | Development setup, core ideas, UI, docs, architecture, conventions, guides, translations, OpenAPI, testing, packaging, addons, and deprecations. `05-architecture.md` is the most useful document for code navigation. |
| `docs/blog/*` | Release and project blog posts with their images. |
| `docs/src/components/*`, `src/css/*`, `src/pages/*` | Docusaurus homepage, static pages, components, and styles. |
| `docs/plugins/woodpecker-plugins/*` | Custom Docusaurus plugin that renders the plugin catalog. |
| `docs/static/*` | Site logos, feature art, favicons, and workflow flowchart assets. |
| `docs/woodpecker.png` | Documentation site/project image used by the root documentation. |

The active documentation files under `docs/docs/20-usage`,
`docs/docs/30-administration`, and `docs/docs/92-development` are intentionally
grouped by topic because their filenames already encode the navigation order.
Read `20-workflow-syntax.md`, `25-workflows.md`, `30-matrix-workflows.md`, and
`92-development/05-architecture.md` first if you want user-facing context.

#### Versioned Documentation

The following four trees are historical copies of the documentation structure,
not separate runtime implementations:

```text
docs/versioned_docs/version-2.8/**
docs/versioned_docs/version-3.15/**
docs/versioned_docs/version-3.16/**
docs/versioned_docs/version-3.17/**
```

Every file under those trees is a versioned Markdown page, category metadata,
diagram, screenshot, or release-era top-level page corresponding to the same
relative path in that version. The version-specific navigation files are:

```text
docs/versioned_sidebars/version-2.8-sidebars.json
docs/versioned_sidebars/version-3.15-sidebars.json
docs/versioned_sidebars/version-3.16-sidebars.json
docs/versioned_sidebars/version-3.17-sidebars.json
```

Do not read the versioned trees during the first architecture pass. Use them
when investigating compatibility, migrations, or when a current behavior was
introduced.

#### Documentation Assets And Plugin Files

| Path family | Purpose |
| --- | --- |
| `docs/docs/**/*.png`, `*.gif`, `*.svg`, `*.excalidraw`, `*.dot` | Diagrams, screenshots, and editable architecture assets. |
| `docs/plugins/woodpecker-plugins/.gitignore` | Plugin build output exclusions. |
| `docs/plugins/woodpecker-plugins/package.json` | Plugin package metadata. |
| `docs/plugins/woodpecker-plugins/plugins.json` | Plugin catalog data. |
| `docs/plugins/woodpecker-plugins/src/index.ts`, `markdown.ts`, `types.ts` | Plugin entry point, Markdown generation, and types. |
| `docs/plugins/woodpecker-plugins/src/theme/*` | React theme components and plugin catalog styling. |
| `docs/plugins/woodpecker-plugins/tsconfig*.json` | Plugin TypeScript configuration. |

### Generated And Non-Architectural Files

| Path | Why it is not a first-read file |
| --- | --- |
| `rpc/proto/woodpecker.pb.go` | Generated protobuf message implementation; read the `.proto` source instead. |
| `rpc/proto/woodpecker_grpc.pb.go` | Generated gRPC bindings; read the `.proto` source instead. |
| `cmd/server/openapi/docs.go` | Generated OpenAPI data; read handlers and `openapi.go` instead. |
| `pipeline/frontend/yaml/linter/schema/schema.json` | Generated/embedded schema data; read `schema.go` and linter code instead. |
| `web/components.d.ts` | Generated frontend component metadata. |
| `*/mocks/mock_*.go` | Generated test doubles. |
| `server/badges/fonts/DejaVuSans.ttf` | Binary font embedded in badge generation. |
| `server/store/datastore/migration/test-files/sqlite.db` | Binary migration test fixture, not the default runtime database. |
| `go.sum`, `web/pnpm-lock.yaml`, `docs/pnpm-lock.yaml`, `flake.lock` | Dependency/input locks, not application behavior. |
| `docs/versioned_docs/**` | Historical documentation snapshots. |

There is no tracked `vendor/` directory. Dependencies are downloaded through
Go and pnpm lockfiles.

## Tests By Purpose

Use tests as executable documentation after reading the corresponding
implementation package.

| Test area | Start with | Main behavior covered |
| --- | --- | --- |
| Pipeline compiler | `pipeline/frontend/yaml/compiler/dag_test.go` | Ordering, optional dependencies, missing dependencies, and cycles. |
| Pipeline builder | `pipeline/frontend/builder/builder_test.go` | Metadata, matrices, linting, filtering, and internal representation. |
| Runtime | `pipeline/runtime/workflow_test.go` and `step_test.go` | Stage concurrency, cancellation, log draining, cleanup, and state. |
| Docker/local backends | `pipeline/backend/docker/convert_test.go` and `pipeline/backend/local/command_test.go` | Backend-specific translation and process behavior. |
| Forge parsing | `server/forge/github/parse_test.go` or `server/forge/gitea/parse_test.go` | Provider payload normalization. |
| Server pipeline | `server/pipeline/create_test.go`, `items_test.go`, `status_test.go` | Creation, persistence, and status aggregation. |
| Queue/scheduler | `server/queue/fifo_test.go` and `server/scheduler/filter_test.go` | Task ordering, leases, and agent routing. |
| Agent RPC | `agent/rpc/client_grpc_test.go` and `server/rpc/rpc_test.go` | Wire behavior, retries, ownership, and state updates. |
| E2E | `e2e/scenarios/suite_test.go` | Full server-agent-backend behavior. |
| Web | `web/src/compositions/*.test.ts` and component `*.test.ts` | Frontend composition and component behavior. |

## Useful Commands While Reading

```sh
# List the exact tracked inventory.
git ls-files

# Show the package dependency and test surface.
go list ./...
go test ./...

# Run the focused architectural tests.
go test ./pipeline/frontend/... ./pipeline/runtime/...
go test ./server/pipeline/... ./server/queue/... ./server/scheduler/...

# Run the frontend tests after reading web/.
pnpm --dir web test

# Regenerate protobuf/OpenAPI/schema artifacts only when needed.
make help
```

## Current Working-Tree Artifacts

These four root SVGs were present as untracked files when this guide was
created. They appear to be before/after diagrams from a local task and are not
part of the tracked application architecture:

| Path | Purpose |
| --- | --- |
| `task1-scenarios-before.svg` | Untracked before-state scenario diagram. |
| `task1-scenarios-after.svg` | Untracked after-state scenario diagram. |
| `task1-schema-before.svg` | Untracked before-state schema diagram. |
| `task1-schema-after.svg` | Untracked after-state schema diagram. |

Ignored local binaries such as `scenarios.test` and `schema.test`, if present,
are build outputs and should not be read as source.

## One-Sentence Summary

Woodpecker has a shared pipeline compiler/runtime at its center, with a server
that turns forge events into durable queued workflows, agents that execute
those workflows through a backend, and REST/event APIs plus a Vue UI that
report and control the resulting state.
