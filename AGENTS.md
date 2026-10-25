# RAGFlow Instructions

Use this file as the local operating guide for the current codebase. Prefer the code and the current AGENTS.md over any older convention or remembered project shape.

## Core Stance
- Treat legacy code as liability, not as a compatibility target.
- Prefer deletion over shims, deprecated branches, wrapper APIs, and dual-track migration notes.
- If old and new implementations coexist, converge to one path unless an external contract forces compatibility.
- Remove dead tests, commented-out code, stale docs, and "move later" notes instead of preserving them.
- Reduce public surface area when a helper can be made private or internal.
- Keep refactors centered on the owning abstraction, not on adjacent compatibility layers.

## What RAGFlow is
RAGFlow is an open-source Retrieval-Augmented Generation engine with agent capabilities. Users upload documents or connect external data sources. DeepDoc parses the documents (layout analysis, OCR, and table recognition, using CPU ONNX inference). A template-based chunker splits the result, and the chunks are embedded and indexed into a document engine. Users then work with that knowledge through:

- **Chats**: RAG chat assistants with grounded citations.
- **Searches**: search apps.
- **Agents**: a visual canvas or workflow builder.
- **Agentic RAG**: multi-step retrieval with Low, Medium, High, or Ultra thinking modes.
- **Knowledge Compilation**: generates Wikis, Graphs, Trees, PageIndex, Mind Maps, Timelines, and Skills per document or dataset.
- **Memories**.
- **MCP**.
- **Chat channels**: DingTalk, Feishu, Discord, Telegram, WeCom, WhatsApp, LINE, and QQ.

## Current stack
- Backend: Go 1.27 (module `ragflow`), Gin HTTP, GORM v1.25.7, zap logging, viper config, CGO linked against native static libs. Go implements the API, admin, ingestor, and syncer modes in `cmd/ragflow_server.go`; the CLI runs from `cmd/ragflow-cli.go`. The Docker entrypoint starts the Go services according to its command-line flags, and the frontend development server uses fixed Go backend proxy routes. The Python server code (`api/`, `rag/`, `deepdoc/`, `agent/`) has been deleted from the tree; the remaining Python is limited to `sdk/python/`, the tests under `test/`, the downloader scripts, the lefthook helper scripts, and a few things under `tools/` and `example/`, all pending removal. Don't add Python, and don't extend it.
- Native: pdfium, pdf_oxide, office_oxide, onnxruntime (static `.a` in `~/ragflow-native-libs`, or `/opt/ragflow-native-libs` when pre-seeded in CI), C++ tokenizer binding in `internal/binding/cpp` built by `build.sh --cpp`.
- Frontend: React 18, TypeScript, Vite 7, shadcn/ui + Tailwind 3, Zustand, TanStack Query 5, react-router 7, react-i18next, @xyflow/react (canvas), Jest, oxlint/oxfmt. When working under `web/`, read and follow `web/AGENTS.md` for frontend conventions.
- Infra: MySQL (or PostgreSQL-family), Elasticsearch / Infinity / OpenSearch / OceanBase / SereneDB / VastBase as doc engine, MinIO (S3/GCS/OSS also supported), Kvrocks, NATS JetStream, ClickHouse, optional OTel/Jaeger, TEI, sandbox executor.

## Go-only target
- Remove all remaining Python code and Python-specific dependencies, scripts, containers, CI jobs, documentation, SDKs, and tests from the repository. Do not add new Python code while this removal is in progress. Replace Python-based preparation of Go native libraries, DeepDoc models, and the BPE table before removing the existing download scripts.
- Move any still-required behavior and test coverage into the Go backend or the frontend before deleting the old implementation. Do not preserve a Python server, compatibility layer, fallback, or dual-backend path.
- `sdk/python/`, `test/` (Python API/Playwright tests), and `example/`: legacy Python pending removal. Migrate needed coverage to Go or frontend tests, then delete them; keep only tests for supported code.
- Update Docker's default entrypoint, dependency image, and CI workflows when removing their remaining Python paths. Keep the frontend development proxy routes aligned with the Go API and Admin ports in `web/vite.config.ts`.

## Backend: one binary, several modes
`cmd/ragflow_server.go` builds `bin/ragflow_server`. Its mode flag picks which service runs:

| Mode | Port | Role |
|---|---|---|
| `--migrate` | none | Runs DB schema migrations (`internal/dao/migration.go`) and exits. Run it once before starting any server. |
| `--admin` | 9381 | Admin server (`internal/admin`): service registry and heartbeats, users, roles, and licensing. Start it **first**, because the other modes report heartbeats to it. `--init-superuser` initializes the superuser account. |
| `--api` (default) | 9380 | Main HTTP API (`/api/v1/...`, `/v1/...`) for the Web UI, SDKs, and bots (`internal/router`, `internal/handler`, `internal/service`). An optional MCP listener runs on 9382 (`mcp:` in config). |
| `--ingestor` | none | Ingestion worker. It consumes tasks from NATS JetStream and runs the parse → chunk → embed → index pipeline (`internal/ingestion`). It also runs knowledge compilation. Tuning flags: `--ingestor-max-concurrent-workers` (1–256) and `--ingestor-page-concurrency` (1–16). |
| `--syncer` | none | Data-source synchronization (`internal/syncer`). It turns `sync_logs` rows into connector calls, using NATS for wake-up and delivery, with the DB as the source of truth. |
| `--deepdoc` | none | Standalone DeepDoc server. By default the ingestor runs DeepDoc **in-process**. |

Common flags: `-f/--config`, `-p/--port`, `--admin-host ip:port`, `--name`, `--log-level debug|info|warn|error` (default: warn), and `--profile` (pprof). Run `./bin/ragflow_server --<mode> --help` to see each mode's options.

`cmd/ragflow-cli.go` builds `bin/ragflow-cli`, a SQL-like interactive CLI with a virtual filesystem for datasets. See `internal/cli/README.md`.

### Request / data flow

```
web (Vite :9222) ──/api,/v1──▶ API :9380 ──GORM──▶ MySQL (metadata)
                  └/api/v1/admin▶ Admin :9381       │
                                                    ├─▶ MinIO (raw files, images)
API ──publish task──▶ NATS JetStream ──▶ Ingestor ──▶ DeepDoc (ONNX, in-process)
                                          │            parser → chunker → embed (LLM provider)
                                          └──────────▶ Doc engine (ES/Infinity/...) index
Syncer ◀── NATS / sync_logs ──▶ external connectors ──▶ documents into datasets
Kvrocks: Redis-protocol cache/locks/sessions   ClickHouse: operation/analytics logs
```

### Configuration
- `conf/service_conf.yaml` is the runtime config for source launches. It has sections for `api`, `admin`, `mysql`, `minio`, `es`, `os`, `infinity`, `oceanbase`, `vastbase`, `kvrocks`, `nats`, `clickhouse`, `otel`, `ingestor`, `file_syncer`, `user_default_llm`, and `gcs`. If you change ports or credentials in `docker/.env`, edit this file too. Don't edit `docker/service_conf.yaml.template`; it is rendered for the container image.
- `kvrocks.host` is a single `host:port` value (default `localhost:6379`). The `KVROCKS_HOST` and `KVROCKS_PORT` env vars do **not** override it.
- Other files in `conf/` hold LLM factory and model catalogs (`llm_factories.json`, `all_models.json`, `models/`), doc-engine index mappings, `agentic_rag.yaml`, `system_settings.json`, and RSA keys.
- Ingestor and DeepDoc env overrides: `RAGFLOW_INGESTOR_MAX_CONCURRENT_WORKERS`, `RAGFLOW_INGESTOR_PAGE_CONCURRENCY`, `RAGFLOW_DEEPDOC_INFERENCE_CONCURRENCY`, `RAGFLOW_DEEPDOC_INFERENCE_CPU_CORES`. When a setting is given more than one way, the CLI flag wins over the env var, and the env var wins over the YAML.
- `RAGFLOW_DEV_MODE=true` skips the code-vs-DB version check during development. It does not migrate anything. Never use it in production.

## Code Layout to Expect
- `cmd/ragflow_server.go`: Go entrypoint for the API, admin, migrate, ingestor, syncer, and deepdoc modes (`--api`, `--admin`, `--migrate`, `--ingestor`, `--syncer`, `--deepdoc`).
- `cmd/ragflow-cli.go`: Go CLI entrypoint.
- `internal/`: main Go application code. Important subtrees:
- `internal/server/`: bootstrap and config per mode (`api/`, `admin/`, `ingestor/`, `syncer/`, `deepdoc/`, `config/`, `bootstrap/`, `local/`).
- `internal/router/`: Gin route registration. Route groups include `/api/v1/{auth,users,tenants,documents,chats,datasets,searches,files,memories,skills,providers,models,agents,connectors,mcp,system,chat-channels,...}` plus the beta bot routes (`searchbots`, `chatbots`, `agentbots`).
- `internal/handler/` → `internal/service/` → `internal/dao/`: HTTP to business logic to persistence (GORM, migrations), with models in `internal/entity/` (LLM provider models live in `entity/models/`).
- `internal/permission/`: access policy and query scoping.
- `internal/common/`: constants, error codes, logger, and parser config.
- `internal/utility/`: shared helpers (MCP client, SSRF guards, files, PDFs).
- `internal/agent/`: Go agent runtime, canvas execution, components, tool bindings, sandbox, templates, chat, audio, and `retrievalbridge`.
- `internal/admin/`: Go admin routes, handlers, and services.
- `internal/harness/`: generic graph engine (Pregel-style), agent kit, checkpoints on NATS, events, replay, and OTel. See its README.
- `internal/ingestion/`: Go ingestion pipeline, canvas adapter, components, wiring, service orchestration.
- `internal/ingestion/component/`: stage implementations such as file/parser/chunker/tokenizer/extractor.
- `internal/ingestion/pipeline/`: DSL translation, canvas-driven execution, checkpoints, resume/run logic.
- `internal/ingestion/service`, `ingestion/task`, `ingestion/wire`, `ingestion/registry`, `ingestion/chunkcache`.
- `internal/ingestion/compilation` and `ingestion/knowledge_compile`: Knowledge Compilation.
- `internal/parser/parser/`: typed parse-result parsers for markdown/html/pdf/docx/xlsx/text and related families.
- `internal/parser/chunk/`: chunk operator library and DSL/typed execution helpers.
- `internal/parser/tableutil/`: table helpers.
- `internal/deepdoc/`: Go DeepDOC integrations, especially native-backed PDF/DOCX parsing. `deepdoc/native` and `deepdoc/parser` cover ONNX layout (DLA), OCR detection and recognition, and table structure recognition. Models are located through `DEEPDOC_MODEL_DIR` or `internal/rag/res/deepdoc/` relative to the working directory.
- `internal/rag/`: Go retrieval and RAG logic and prompts. `rag/res/` holds resources; the downloaded DeepDoc models go into `internal/rag/res/deepdoc/`.
- `internal/agentic_rag/`: multi-step agentic retrieval, BM25 and grep services, KG, citations, and evidence.
- `internal/tokenizer/`: BPE (cl100k), SPM, WordPiece, and embedding token limits.
- `internal/engine/`: backend drivers for `elasticsearch`, `infinity`, `oceanbase`, `serenedb`, `vastbase`, `kvrocks`, `nats`, and `clickhouse`.
- `internal/storage/`: MinIO, S3, GCS, OSS, and in-memory test doubles.
- `internal/syncer/`: Go file synchronization service and `syncer/connector` data-source connectors.
- `internal/channels/`: chat-channel bots.
- `internal/mcp/`: MCP server.
- `internal/binding/cpp/`: the C++ RAG analyzer, built by `build.sh --cpp`.
- `internal/cli/`: CLI parsing, HTTP transport, command execution, response formatting.
- `internal/dao/`: Go data-access layer and persistence-facing helpers.
- `internal/entity/`: shared Go entities and model definitions.
- `internal/service/`: higher-level business services used by handlers and server flows.
- `*_ee.go` files are enterprise-edition hooks with community-edition defaults. Keep them compiling, but don't build features around them unless asked.
- `web/`: frontend application.
- `docker/`: local and production compose files.
- `sdk/python/`: legacy Python client SDK pending deletion.
- `tools/scripts/download_dev_deps.py` and `ragflow_deps/download_deps.py`: Python downloaders for the Go build's dependencies. `download_dev_deps.py` is the local-dev one-stop: it pulls the native static libraries (pdfium, pdf_oxide, office_oxide, onnxruntime) into `~/ragflow-native-libs` and the runtime resources (DeepDoc `.ort` models and `ocr.res` into `internal/rag/res/deepdoc/`, `cl100k_base.tiktoken` and embedding tokenizer assets into `ragflow_deps/`). `download_deps.py` provisions the same runtime resources for the dependency image or CI (it also installs the stagehand server into the SDK cache) and skips the native `.a` libraries. Versions and URLs are defined inline in each downloader (`tools/scripts/download_dev_deps.py` for native static libraries, `ragflow_deps/download_deps.py` for runtime resources); the tokenizer assets are inlined as `TOKENIZER_ASSETS` in each downloader. Replace these with non-Python preparation before removing Python dependencies.
- `test/`: contains Python tests, including tests that import the legacy Python implementation directly. Migrate needed coverage to Go tests or frontend tests, then remove the Python tests; keep only tests for supported code.

## Go-Specific Rules
- Treat `internal/ingestion`, `internal/parser`, and `internal/deepdoc` as actively refactored code. Prefer collapsing duplicate paths over preserving transitional wrappers.
- Do not add or preserve deprecated Go APIs just to ease migration inside the repo.
- Remove commented-out Go code instead of leaving recovery notes in place.
- Keep package comments and doc comments aligned with the current runtime path, not with migration history.

## GORM query traps
- **Never pass a bare string ID as the second argument to `Take`/`First`** (e.g. `Take(&x, id)`). Under GORM v1.25.7 a non-numeric string is treated as a raw SQL condition and concatenated into the `WHERE` clause: `Take(&x, "88b17c7a...")` emits `WHERE 88b17c7a... LIMIT 1`, which MySQL rejects with `Error 1054 (42S22): Unknown column '...' in 'where clause'`. Only strings that parse via `strconv.Atoi` are treated as primary-key values, so this bug is invisible for integer IDs and catastrophic for UUID primary keys. Always bind explicitly: `Take(&x, "id = ?", id)` or `Where("id = ?", id).Take(&x)`.
- When you change a query form across many files, verify the generated SQL with a `Session(&gorm.Session{DryRun: true})` probe before relying on it. A passing unit test with no rows and a broken `WHERE` clause look identical unless the SQL is inspected.

## Database schema ownership (Go migrations)
The legacy Python API that once co-owned the MySQL/PostgreSQL schema has been removed. Go now owns schema migration end to end through GORM and `internal/dao/migration.go`.

- GORM derives `idx_<table>_<column>` for indexes and `uni_<table>_<column>` for unique constraints. Tags in this repo name unique indexes explicitly, e.g. `uniqueIndex:idx_commit_file`.
- `hasUniqueIndex` (`internal/dao/migration.go`) reconciles indexes by **column set, not by name**: it accepts an existing index only when its full column set equals the requested one. Divergent names covering the same columns are expected, not a defect. Do not "fix" this by renaming inside a migration; it rewrites live databases for no functional gain.
- Do not add a second index over a column set an existing one already covers.
- Declare single-column uniqueness with a named `uniqueIndex:`, not a bare `unique` tag. A bare `unique` becomes a table-level constraint in DDL, which SQLite materializes as an implicit index that cannot be dropped by name, and which GORM's `MigrateColumnUnique` then tries to drop under its own `uni_<table>_<column>` name — MySQL answers 1091 every startup. `namedIndexMigrator` (`internal/dao/database.go`) suppresses exactly that phantom drop; see `TestNamedIndexMigratorSkipsPhantomUniqueDrop`.
- Not every column or index comes from a model tag: `pipeline_operation_log` has no `run_count` column and no unique index in the model; both are added only by `internal/dao/migration.go`.

## Go Test Tiers
Go tests are classified by build tag. Tag a test file with `//go:build <tier>` placed before the `package` clause. Run them through `build.sh` so the required CGO configuration and native libraries are available.

| Tier | Build tag | Runs by default? | Needs |
|---|---|---|---|
| Unit | (none) | Yes (`bash build.sh --test`) | Native CGO static libs; no external services — uses in-memory SQLite, miniredis, or `httptest` stubs. |
| Integration | `integration` | No (`-tags integration`) | A real service: MySQL/MinIO/Elasticsearch/Infinity/LLM. Single component, reasonably fast. |
| E2E | `e2e` | No (`-tags e2e`) | Full cross-component pipeline (ingest → index → retrieve) against real services; heavy/slow. |
| Manual | `manual` | No (`-tags manual`) | Very slow/expensive (deepdoc render/parity/snapshot/bench). **Local opt-in ONLY — never run in CI.** |
| Native (orthogonal) | `cgo` / `!cgo` | `cgo` auto-satisfies under CGO_ENABLED=1 | Native static libs (`office_oxide`/`pdfium`/`pdf_oxide`). Combine with tiers, e.g. `//go:build cgo && integration`. |

Run tiers locally via `build.sh`:
```bash
bash build.sh --test                      # unit tier (no tags)
bash build.sh --test-integration ./...    # integration tier
bash build.sh --test-e2e                  # e2e tier
bash build.sh --test-manual               # manual tier (very slow)
bash build.sh --test-all                  # integration + e2e (never includes manual)
```
Rules:
- New tests that touch a real external service MUST carry `integration`/`e2e`/`manual` — do not rely on `t.Skip` + env vars to soft-isolate them in the default unit run. Keep an env guard as a harmless secondary safety net if desired.
- `manual` is never wired into CI or any automated pipeline.
- `unit` (no tag) must stay free of external-service dependencies so `bash build.sh --test` runs without MySQL/MinIO/ES/Infinity/LLM. The native CGO static libraries (`office_oxide`/`pdfium`/`pdf_oxide`) are still required at build time and are wired by `build.sh --test`; that is expected, not an external service.

## Working Rules
- When reviewing documentation or code, inspect the full affected path and report all verifiable findings in one review; do not return after only a few findings and expose further issues in later rounds.
- When handling review comments, independently verify each substantive claim against the current code or tests before accepting, rejecting, or acting on it.
- Before editing, inspect the nearest code path that actually owns the behavior.
- For server changes, work in the Go implementation. If a Python behavior is still needed, port it to Go and test the Go path before removing the Python code.
- Keep changes small and local unless the task is explicitly a broader refactor.
- Prefer one implementation path instead of preserving old and new versions side by side.
- Preserve behavior with focused tests when the behavior is still valid; do not keep tests that protect obsolete behavior.
- If a surface is only there for compatibility, remove it unless the user asks to keep it.
- Do not add new compatibility wording in comments or docs.
- When a maintainer takes over a community PR, a new commit generated by rewriting history (e.g. `merge`, `rebase -i`) must preserve the original author and add the maintainer as co-author (via a `Co-authored-by:` trailer) instead of overwriting the author with the maintainer alone.

## Commands
### Go backend
```bash
bash build.sh --all                         # native C++ + Go server + CLI (--strip for smaller binaries)
bash build.sh --cpp                         # C++ binding only
bash build.sh --go                          # build Go server with native library already built
bash build.sh --test ./internal/handler/... # targeted unit tests
bash build.sh --test                        # all unit tests
bash build.sh --test-integration ./...      # integration tests; requires real services
bash build.sh --test-e2e                    # full-pipeline tests; requires real services
bash build.sh --test-manual                 # local opt-in only
bash build.sh --test-all                    # integration + e2e; excludes manual
bash build.sh --run                         # build and run (admin + ingestor + api; migrates first)
bash build.sh --help
```

Native libraries, Go DeepDoc models, tokenizer assets, and `ragflow_deps/cl100k_base.tiktoken` come from `tools/scripts/download_dev_deps.py` (native static libs; run with `uv run`) and `ragflow_deps/download_deps.py` (models, cl100k, tokenizer assets). Build the Go resource image with `uv run ragflow_deps/download_deps.py` then `cd ragflow_deps && docker build -f Dockerfile -t infiniflow/ragflow_deps:latest .` after downloading the resources.

Repo-wide pre-commit hooks are in `lefthook.yml`: EOF, whitespace, and line endings, `gofmt`, ruff, YAML/JSON and merge-conflict checks, and `web-oxfmt`/`web-oxlint`. Don't bypass them with `--no-verify`. CI workflows are in `.github/workflows/`.

### Frontend
```bash
cd web
npm install
npm run dev          # Vite dev server on 9222 (override with PORT)
npm run build
npm run lint
npm run format
npm run test
npm run type-check
```

The dev server proxies `/api/v1/admin` → 9381, and `/api` and `/v1` → 9380 (`web/vite.config.ts`). The frontend targets only the Go backend; don't add backend-variant code. See `web/AGENTS.md` for the full conventions.

## Launch from source (Ubuntu 24.04 x86_64; macOS is not supported by the Go backend)
The upstream guide is `docs/develop/launch_ragflow_from_source.md` (also at https://ragflow.io/docs/launch_ragflow_from_source). Detailed toolchain notes are in `internal/development.md`.

**Prerequisites:**
- About 4 CPU cores, 16 GB RAM, and 50 GB of free disk.
- Docker ≥ 24.0.0 and Compose ≥ v2.26.1.
- Go 1.27 (see `go.mod`).
- CMake ≥ 4.0, Clang 20, and **LLD 20**.
- `libpcre2-dev` and binutils (`objcopy`, `nm`).
- Node.js ≥ 18.20.4 with npm.
- Python 3.10+ and `uv`, used only to run the downloaders.

Two toolchain warnings:
- An older LLD (such as 18) links a binary that hangs or crashes before Go `main` runs. Verify with `ld.lld --version`.
- A missing `objcopy` skips RE2 symbol renaming, which causes runtime crashes.

```bash
# 1. Dependencies (once; re-runs skip existing files)
uv run tools/scripts/download_dev_deps.py
#   -> native static libs into ~/ragflow-native-libs
#   -> DeepDoc models (det.ort, layout.ort, tsr.ort, rec.ort, ocr.res) into internal/rag/res/deepdoc/
#   -> cl100k_base.tiktoken + embedding tokenizer assets into ragflow_deps/

# 2. Build
bash build.sh --all                 # native C++ + Go server + CLI
./bin/ragflow_server --api --help   # sanity check that the binary loads (may exit with status 1)

# 3. Infrastructure: list services explicitly
sudo sysctl -w vm.max_map_count=262144   # required for Elasticsearch
docker compose --env-file docker/.env -f docker/docker-compose-base.yml up -d --wait \
  es01 mysql minio nats kvrocks clickhouse

# 4. Migrate, then start each mode in its own terminal from the repo root (Admin first)
./bin/ragflow_server --migrate
RAGFLOW_DEV_MODE=true ./bin/ragflow_server --admin      # :9381
RAGFLOW_DEV_MODE=true ./bin/ragflow_server --api        # :9380
RAGFLOW_DEV_MODE=true ./bin/ragflow_server --ingestor
RAGFLOW_DEV_MODE=true ./bin/ragflow_server --syncer

# 5. Frontend
cd web && npm install && npm run dev    # http://127.0.0.1:9222/

# 6. Verify
curl -fsS http://127.0.0.1:9380/api/v1/system/healthz      # HTTP 200
curl -fsS http://127.0.0.1:9222/api/v1/system/version      # JSON with "code":0
curl -fsS http://127.0.0.1:8123/ping                       # ClickHouse: Ok.

# Stop infrastructure (stop keeps the containers; down removes them, volumes are kept)
docker compose --env-file docker/.env -f docker/docker-compose-base.yml stop \
  es01 mysql minio nats kvrocks clickhouse
```

Notes:
- Run the server from the repo root, or set `DEEPDOC_MODEL_DIR` to the absolute path of `internal/rag/res/deepdoc`.
- The log message "no in-process DeepDoc backend serving" means one of the five model files is missing. Re-run the downloader and rebuild.
- If `cl100k_base.tiktoken` is missing:

  ```bash
  curl -fsSL -o ragflow_deps/cl100k_base.tiktoken https://openaipublic.blob.core.windows.net/encodings/cl100k_base.tiktoken
  ```
- A warning that `REDIS_PORT` is unset during compose startup is harmless.

## Services (Docker)
`docker/docker-compose-base.yml` holds the infrastructure, and `docker/docker-compose.yml` adds the `ragflow-cpu` app container. That container exposes nginx on 80 and 443, API on 9380, and Admin on 9381, and its entrypoint runs migrations and then Admin, Syncer, Ingestor, and API (`docker/entrypoint.sh` flags: `--disable-api-server`, `--disable-ingestor`, `--disable-syncer`, `--enable-adminserver`, `--init-superuser`, and similar). `docker/.env` selects the profiles through `DOC_ENGINE` (default `elasticsearch`), `DEVICE`, and `METADATA_DB_PROFILE`; its README explains the env vars.

| Service | Host port (default) | Purpose |
|---|---|---|
| `mysql` | 3306 | Metadata DB (`rag_flow`, password `infini_rag_flow`) |
| `es01` | 1200 | Elasticsearch doc engine (default) |
| `infinity` / `opensearch01` / `oceanbase` / `seekdb` / `serenedb` / `vastbase` | various | Alternative doc engines (profiles) |
| `minio` | 9000 (console 9001) | Object storage |
| `kvrocks` | 6379 | Redis-compatible KV store (cache, locks) |
| `nats` | 4222 (monitor 8222) | JetStream task queue (ingestion, syncer, compilation, harness checkpoints) |
| `clickhouse` | 9900 TCP, 8123 HTTP | Analytics and operation logs |
| `jaeger`, `kibana`, `tei-cpu`/`tei-gpu`, `sandbox-executor-manager` | optional | Tracing, ES UI, local embeddings, agent code sandbox |

## Other top-level directories
- `Dockerfile*`: the app image (`Dockerfile`), base, CI, and TEI images.
- `docs/`: user and developer documentation (Docusaurus `.md`/`.mdx`). The developer docs are in `docs/develop/`.
- `conf/`: runtime config and catalogs (see Configuration).
- `ragflow_deps/`: the downloaded runtime resources and the dependency-image Dockerfile.
- `tools/`: `scripts/download_dev_deps.py`, `hooks/` (lefthook helpers), `covsweep`, `gen-component-parity`, `migrate-canvas`, `mdprobe`, `render_diff`, and migration and integration utilities.
- `bin/`: build output (`ragflow_server`, `ragflow-cli`).

## Validation Preference
- Run the narrowest relevant test, lint, or build command after a change.
- For backend changes, first run package-scoped `bash build.sh --test ./path/to/package/...`. Run the integration or e2e tier when the changed behavior needs real services or a full pipeline.
- For frontend changes, prefer the touched-package lint, type-check, or test command.
- Do not default to raw `go test`, `go build`, or IDE Run/Debug for Go in this repo. They often miss the required CGO flags and native static libraries (`office_oxide`, `pdfium-static`, `pdf_oxide`) that `build.sh` wires correctly.
- After a Go build, execute the resulting binary's help command for each changed server mode; a successful link alone does not prove the binary starts. On this host, the original `build.sh` selected default LLD 18 and produced binaries that crashed before Go `main`; linking with LLD 20 started successfully. Verify the linker actually selected on Linux before trusting a new binary.
- If Go native builds fail, inspect `build.sh` and `internal/development.md` before changing code. Common environment issues are missing downloaded native dependencies or an incompatible `ld.lld` on Linux.

## Default review checklist
- Remove instead of retaining `deprecated`, `legacy`, or compatibility-only code.
- Collapse duplicate implementations to one path.
- Drop stale comments and documentation that describe a superseded design.
- Keep exported APIs only when the current code actually needs them.
