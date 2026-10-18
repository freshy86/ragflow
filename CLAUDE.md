# CLAUDE.md

This file guides Claude Code when it works in this repository. It describes the architecture, modules, and services, and it explains how to run RAGFlow from source.

Working rules come from **`AGENTS.md`** (repo root) and **`web/AGENTS.md`** (frontend), and those files take precedence. They cover the Go-only migration stance, GORM traps, test tiers, the review checklist, and the frontend conventions. Read them before you make changes.

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

## Stack at a glance

| Layer | Tech |
|---|---|
| Backend | Go 1.27 (module `ragflow`), Gin HTTP, GORM v1.25.7, zap logging, viper config, CGO + native static libs |
| Native | pdfium, pdf_oxide, office_oxide, onnxruntime (static `.a` in `~/ragflow-native-libs`), C++ tokenizer binding (`internal/binding/cpp`) |
| Frontend | React 18, TypeScript, Vite 7, shadcn/ui + Tailwind 3, Zustand, TanStack Query 5, react-router 7, react-i18next, @xyflow/react (canvas), Jest, oxlint/oxfmt |
| Infra | MySQL (or PostgreSQL-family), Elasticsearch / Infinity / OpenSearch / OceanBase / SereneDB / VastBase as doc engine, MinIO (S3/GCS/OSS also supported), Kvrocks, NATS JetStream, ClickHouse, optional OTel/Jaeger, TEI, sandbox executor |

The backend is Go-only. The Python server code (`api/`, `rag/`, `deepdoc/`, `agent/`) has already been deleted from the tree. The Python that remains is limited to `sdk/python/`, the tests under `test/`, the downloader scripts, the lefthook helper scripts, and a few things under `tools/` and `example/`, all pending removal. Don't add Python, and don't extend it. Note that AGENTS.md still lists the Python directories in its inventory, which is out of date.

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

### `internal/` packages

- **Entry and wiring**
  - `server/`: bootstrap and config per mode (`api/`, `admin/`, `ingestor/`, `syncer/`, `deepdoc/`, `config/`, `bootstrap/`, `local/`).
  - `router/`: Gin route registration. Route groups include `/api/v1/{auth,users,tenants,documents,chats,datasets,searches,files,memories,skills,providers,models,agents,connectors,mcp,system,chat-channels,...}` plus the beta bot routes (`searchbots`, `chatbots`, `agentbots`).
- **HTTP to business logic to persistence**: `handler/` → `service/` → `dao/` (GORM, migrations) with models in `entity/` (LLM provider models live in `entity/models/`).
  - `permission/`: access policy and query scoping.
  - `common/`: constants, error codes, logger, and parser config.
  - `utility/`: shared helpers (MCP client, SSRF guards, files, PDFs).
- **Ingestion**
  - `ingestion/pipeline`: DSL translation, canvas-driven execution, checkpoints, and resume.
  - `ingestion/component`: file, parser, chunker, extractor, and tokenizer stages.
  - `ingestion/service`, `ingestion/task`, `ingestion/wire`, `ingestion/registry`, and `ingestion/chunkcache`.
  - `ingestion/compilation` and `ingestion/knowledge_compile`: Knowledge Compilation.
- **Parsing**
  - `parser/parser`: typed parsers for markdown, html, pdf, docx, xlsx, text, and more.
  - `parser/chunk`: chunk operators.
  - `parser/tableutil`.
  - `deepdoc/native` and `deepdoc/parser`: ONNX layout (DLA), OCR detection and recognition, and table structure recognition. Models are located through `DEEPDOC_MODEL_DIR` or `internal/rag/res/deepdoc/` relative to the working directory.
- **Retrieval and RAG**
  - `rag/`: retrieval logic and prompts. `rag/res/` holds resources; the downloaded DeepDoc models go here.
  - `agentic_rag/`: multi-step agentic retrieval, BM25 and grep services, KG, citations, and evidence.
  - `tokenizer/`: BPE (cl100k), SPM, WordPiece, and embedding token limits.
- **Agents**
  - `agent/`: canvas runtime, DSL, components, tools, sandbox, templates, chat, audio, and `retrievalbridge`.
  - `harness/`: a generic graph engine (Pregel-style), agent kit, checkpoints on NATS, events, replay, and OTel. See its README.
- **Integrations**
  - `engine/`: backend drivers for `elasticsearch`, `infinity`, `oceanbase`, `serenedb`, `vastbase`, `kvrocks`, `nats`, and `clickhouse`.
  - `storage/`: MinIO, S3, GCS, OSS, and an in-memory test double.
  - `syncer/connector`: data-source connectors.
  - `channels/`: chat-channel bots.
  - `mcp/`: MCP server.
  - `binding/cpp`: the C++ RAG analyzer, built by `build.sh --cpp`.
- **Tooling**: `cli/` (the Go CLI).

`*_ee.go` files are enterprise-edition hooks with community-edition defaults. Keep them compiling, but don't build features around them unless asked.

### Configuration

- `conf/service_conf.yaml` is the runtime config for source launches. It has sections for `api`, `admin`, `mysql`, `minio`, `es`, `os`, `infinity`, `oceanbase`, `vastbase`, `kvrocks`, `nats`, `clickhouse`, `otel`, `ingestor`, `file_syncer`, `user_default_llm`, and `gcs`. If you change ports or credentials in `docker/.env`, edit this file too. Don't edit `docker/service_conf.yaml.template`; it is rendered for the container image.
- `kvrocks.host` is a single `host:port` value (default `localhost:6379`). The `KVROCKS_HOST` and `KVROCKS_PORT` env vars do **not** override it.
- Other files in `conf/` hold LLM factory and model catalogs (`llm_factories.json`, `all_models.json`, `models/`), doc-engine index mappings, `agentic_rag.yaml`, `system_settings.json`, and RSA keys.
- Ingestor and DeepDoc env overrides:
  - `RAGFLOW_INGESTOR_MAX_CONCURRENT_WORKERS`
  - `RAGFLOW_INGESTOR_PAGE_CONCURRENCY`
  - `RAGFLOW_DEEPDOC_INFERENCE_CONCURRENCY`
  - `RAGFLOW_DEEPDOC_INFERENCE_CPU_CORES`

  When a setting is given more than one way, the CLI flag wins over the env var, and the env var wins over the YAML.
- `RAGFLOW_DEV_MODE=true` skips the code-vs-DB version check during development. It does not migrate anything. Never use it in production.

## Frontend (`web/`)

- Vite dev server: `npm run dev` serves on **9222** (override with `PORT`). Proxies in `web/vite.config.ts`: `/api/v1/admin` → 9381, and `/api` and `/v1` → 9380. The frontend targets only the Go backend; don't add backend-variant code.
- `src/pages/` holds the feature pages: `datasets`/`dataset`, `chunk`, `document-viewer`, `next-chats`, `next-search(es)`, `agent`/`agents` (canvas), `builtin-pipeline`, `dataflow-result`, `memories`/`memory`, `skills`, `files`, `user-setting`, `admin`, `login-next`, and `home`.
- Request layering is mandatory. Hooks in `src/hooks/use-*-request.ts` call services in `src/services/*-service.ts`, and services call `src/utils/next-request.ts`, which is the single axios instance. Response types go in `src/interfaces/database/` and request types in `src/interfaces/request/`.
- Other pieces:
  - `src/components/ui/` is the locked shared UI library. Wrap its components instead of editing them.
  - `src/locales/` holds i18n. Usually only `en.ts` and `zh.ts` change, and `en.ts` uses sentence case.
  - Color tokens come from `web/tailwind.css`.
- Commands:

  ```bash
  npm install
  npm run dev
  npm run build
  npm run lint
  npm run format
  npm run type-check
  npm run test
  ```

  Tests go in `__tests__/` directories or in co-located `*.test.ts(x)` files.

See `web/AGENTS.md` for the full conventions: query key factories, no inline handlers, query colocation, and the rest.

## Services (Docker)

`docker/docker-compose-base.yml` holds the infrastructure, and `docker/docker-compose.yml` adds the `ragflow-cpu` app container. That container exposes nginx on 80 and 443, API on 9380, and Admin on 9381, and its entrypoint runs migrations and then Admin, Syncer, Ingestor, and API. `docker/.env` selects the profiles through `DOC_ENGINE` (default `elasticsearch`), `DEVICE`, and `METADATA_DB_PROFILE`.

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
# (ragflow_deps/download_deps.py is for the dependency image or CI: it adds stagehand and skips the native libs)

# 2. Build
bash build.sh --all                 # native C++ + Go server + CLI  (--strip for smaller binaries)
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
# Shortcut: RAGFLOW_DEV_MODE=true bash build.sh --run   (admin + ingestor + api; no syncer; migrate first)

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
- Build the resource image with:

  ```bash
  uv run ragflow_deps/download_deps.py
  cd ragflow_deps && docker build -f Dockerfile -t infiniflow/ragflow_deps:latest .
  ```

## Build and test commands

Always build and test Go code through `build.sh`. Raw `go build`, `go test`, and IDE Run buttons miss the CGO flags and native libraries.

```bash
bash build.sh --all | --cpp | --go          # full / C++ binding only / Go only
bash build.sh --test ./internal/handler/... # unit tier for a package (no external services)
bash build.sh --test                        # all unit tests
bash build.sh --test-integration ./...      # needs real MySQL/MinIO/ES/Infinity/LLM
bash build.sh --test-e2e                    # full pipeline against real services
bash build.sh --test-all                    # integration + e2e
bash build.sh --test-manual                 # deepdoc render/parity/bench; local only, never in CI
bash build.sh --help
```

Test tiers use build tags: none for unit, `integration`, `e2e`, and `manual`, plus the orthogonal `cgo`/`!cgo`. See AGENTS.md for the rules. After a Go build, run `--help` for every server mode you changed.

Repo-wide pre-commit hooks are in `lefthook.yml`. They cover EOF, whitespace, and line endings, `gofmt`, ruff, YAML/JSON and merge-conflict checks, and `web-oxfmt`/`web-oxlint`. Don't bypass them with `--no-verify`. CI workflows are in `.github/workflows/`.

## Other top-level directories

- `docker/`: compose files, `entrypoint.sh` (with `--disable-api-server`, `--disable-ingestor`, `--disable-syncer`, `--enable-adminserver`, `--init-superuser`, and similar flags), nginx config, init SQL for MySQL and ClickHouse, and a `docker/README.md` that explains the env vars.
- `Dockerfile*`: the app image (`Dockerfile`), base, CI, and TEI images.
- `docs/`: user and developer documentation (Docusaurus `.md`/`.mdx`). The developer docs are in `docs/develop/`.
- `conf/`: runtime config and catalogs (see above).
- `ragflow_deps/`: the downloaded runtime resources and the dependency-image Dockerfile.
- `tools/`: `scripts/download_dev_deps.py`, `hooks/` (lefthook helpers), `covsweep`, `gen-component-parity`, `migrate-canvas`, `mdprobe`, `render_diff`, and migration and integration utilities.
- `sdk/python/`, `test/` (Python API/Playwright tests), `example/`: legacy Python pending removal. Move any coverage that's still needed into Go or frontend tests.
- `bin/`: build output (`ragflow_server`, `ragflow-cli`).
