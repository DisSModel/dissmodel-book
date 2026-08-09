# Chapter 9: The DisSModel Platform — Execution & Registry

*Part IV — Data & Infrastructure*

Covers the [`dissmodel-platform`](https://github.com/DisSModel/dissmodel-platform)
and [`dissmodel-configs`](https://github.com/DisSModel/dissmodel-configs) packages.

## Learning objectives

- Bring the platform up locally via Docker Compose
- Understand the service architecture (JupyterLab, REST API, MinIO, workers)
- Register a new executor via `dissmodel-configs`
- See a full usage example: triggering an experiment from QGIS

## 9.1 Status disclaimer

> ⚠️ **MVP phase.** While the DisSModel ecosystem is steadily growing,
> the platform is currently a Minimum Viable Product. Security hardening
> and extreme performance efficiency are not the primary focus at this
> stage. Use caution if deploying in production environments.

## 9.2 Where it runs

- ✅ Local server (desktop/laptop)
- ✅ On-premise cluster (INPE, universities)
- ✅ Private cloud (OpenStack, etc.)
- ✅ Public cloud (AWS, GCP, Azure) — optional

## 9.3 Installation

```bash
git clone https://github.com/DisSModel/dissmodel-platform.git
cd dissmodel-platform
cp .env.example .env
docker compose up --build
```

Available services:

- JupyterLab: `http://localhost:8888`
- API Docs: `http://localhost:8000/docs`
- MinIO: `http://localhost:9001`

```bash
docker compose down       # stop
docker compose down -v    # stop and remove volumes (use with caution!)
```

## 9.4 Architecture

The platform is a containerized microservice architecture
(`docs/architecture.md`) built around one data flow: a researcher's
browser talks to JupyterLab, which either imports DisSModel directly for
light local work or submits heavy jobs to a REST API:

```
Researcher → Browser → JupyterLab (container)
                          │
                          ├── direct Python imports (light models)
                          └── REST API (heavy jobs)
                                   │
                                   ▼
                  Docker host: API (FastAPI) · Worker · MinIO · Redis
```

- **JupyterLab (frontend)** — interactive development; runs light models
  locally, submits heavy jobs via the API.
- **API Gateway (FastAPI)** — receives job requests, validates
  parameters, enqueues tasks in Redis.
- **Workers** — consume the Redis queue, execute DisSModel models
  (through the `ModelExecutor` contract from Chapter 2), save results to
  MinIO.
- **Storage (MinIO)** — S3-compatible, stores inputs/outputs, persisted
  via Docker volumes.
- **Queue (Redis)** — message queue with priority support (high, normal,
  low), also caches job metadata.

**Data flow:** a researcher uploads data to `data/inputs/` → it becomes
reachable via MinIO (`s3://dissmodel-inputs/`) → the model is developed
in JupyterLab → heavy jobs are submitted through the API → workers
process them and write results to `data/outputs/`.

**Scaling.** Workers scale horizontally with
`docker compose up --scale worker=5`; a later phase is planned to move
production deployments to Kubernetes + Dask, with Prometheus metrics —
both explicitly marked as "Fase 2" (not yet implemented), consistent with
the MVP disclaimer in 9.1.

## 9.5 The executor registry (`dissmodel-configs`)

`dissmodel-configs` decouples model parameterization from both the
executor packages and the platform itself. The platform reads each TOML
at experiment submission time, resolves the executor class from the
registered Python package, and injects the spec into the experiment
record.

A TOML file merged into this registry means the executor is available on
the platform. New executors enter the platform only through a PR to this
repository.

### TOML structure

```toml
[model]
executor_module = "<python.module.path>"
name            = "<executor_name>"
class           = "<executor_name>"
description     = "..."
package         = "git+https://github.com/DisSModel/<repo>@<ref>"
dissmodel       = ">=0.6.0,<0.7.0"

[model.parameters]
# Runtime parameters read via params.get("key")
```

Fields under `[model]` (excluding `parameters`) are exposed as
`record.resolved_spec["model"]`, and `[model.parameters]` is exposed as
`record.parameters`. Placing a field in the wrong section silently falls
back to its default.

### File naming convention

```
<model>_<substrate>.toml
```

Real examples from `dissmodel-configs/models/`:

| File | `name`/`class` | Substrate |
|---|---|---|
| `brmangue_raster.toml` | `brmangue_raster` | RasterBackend/NumPy |
| `coastal_raster.toml` | `coastal_raster` | RasterBackend/NumPy |
| `coastal_vector.toml` | `coastal_vector` | GeoDataFrame |
| `disslucc_raster.toml` | `lucc_raster` | RasterBackend/NumPy |
| `disslucc_vector.toml` | `lucc_vector` | GeoDataFrame |

`brmangue_raster.toml` is the canonical reference for the convention:

```toml
# dissmodel-configs/models/brmangue_raster.toml
[model]
executor_module = "brmangue.executors"
name            = "brmangue_raster"
class           = "brmangue_raster"
description     = "BR-MANGUE raster simulation (mangrove + flood dynamics)"
package         = "git+https://github.com/DisSModel/brmangue-dissmodel@refactoring"

[model.parameters]
end_time      = 88
taxa_elevacao = 0.5
altura_mare   = 6.0
acrecao_ativa = false
resolution    = 100.0
crs           = "EPSG:31983"
```

Beyond `[model]` and `[model.parameters]`, the registry format
(`docs/platform/registry.md`) also supports `[model.bands]` /
`[model.columns]` — a **canonical vocabulary** mapping generic names
(e.g. `elevation`) to the actual dataset column/band names (e.g.
`SRTM_B1`), resolved at request time via `band_map`/`column_map` so the
same executor code runs unmodified across differently-named datasets —
and arbitrarily nested tables like `[[model.potential]]` for structured
parameters such as regression betas (Chapter 7).

**Synchronization.** The platform doesn't read `dissmodel-configs`
directly on every request — a background `APScheduler` job runs `git
pull` on the worker/API nodes every 15 minutes (configurable), and
invalidates the `lru_cache` of `load_model_spec()` when it detects
changes, so a merged PR becomes visible via `GET /models` within that
window without a redeploy. `POST /admin/sync` (with an API key) forces
an immediate sync, and `POST /submit_job_inline` lets a researcher bypass
the registry entirely for rapid exploration in Jupyter — at the cost of
reproducibility (`model_commit` is recorded as `local-inline`, not tied
to a `dissmodel-configs` git commit).

## 9.6 Usage example: triggering experiments from QGIS

The [`brmangue-qgis`](https://github.com/DisSModel/brmangue-qgis) plugin is
a concrete client of the platform's REST API described above — it lets a
researcher select layers directly in QGIS and submit an experiment without
touching the command line.

> **Note on sourcing.** `brmangue-qgis` has no README yet, so this section
> is grounded directly in its source files (`plugin.py`, `dialog.py`,
> `task.py`, `payload_builder.py`, `symbology.py`, `experiment_panel.py`,
> `metadata.txt`) rather than in prose documentation. Per `metadata.txt`
> (v0.2.0): "No local installation of `dissmodel` or `brmangue-dissmodel`
> is required — the science runs on the server."

**End-to-end flow:**

1. **Install & open.** The plugin registers a toolbar button and a
   Plugins → Raster menu entry (`plugin.py: BrmanguePlugin.initGui`).
   Triggering it opens `BrmangueDialog`.
2. **Configure.** The dialog collects a platform `server_url` (default
   `http://127.0.0.1:8000`), an `X-API-Key`, an `input_uri` (e.g.
   `s3://dissmodel-inputs/ilha_maranhao.tif`), model parameters
   (`end_time`, `resolution`, ...), and which output bands to load
   (default `["uso", "solo", "alt"]`).
3. **Submit.** `payload_builder.build_payload()` assembles a JSON body —
   `model_name: "brmangue_raster"`, `input_dataset`, `input_format`,
   `parameters`, and an optional `band_map` — matching the platform's
   `POST /submit_job` contract from 9.4/9.5 exactly.
4. **Poll (background).** `BrmangueTask` (a `QgsTask`) runs the submit →
   poll loop off the UI thread: `POST /submit_job` returns a `job_id`,
   then `GET /job/{experiment_id}` is polled every ~4 seconds until
   `status == "completed"` (or `"failed"`, surfacing the last server log
   line as the error message).
5. **Load & style.** On completion, `symbology.py: load_result()` opens
   the result's `uso`/`solo`/`alt` bands as separate QGIS raster layers
   and applies automatic styling per band — a paletted (categorical)
   renderer for land use (`uso`) and soil (`solo`), and a continuous
   gray, contrast-stretched renderer for elevation (`alt`) — so the
   researcher never has to build a QML style by hand.
6. **Inspect provenance.** The completed job's FAIR metadata
   (`model_commit`, `code_version`, `output_sha256` — the same fields
   that make up an `ExperimentRecord`, Chapter 2) is captured by
   `BrmangueTask` and made available through `ExperimentPanel`, which a
   researcher can open from the result log to inspect or copy a
   citation.

This makes `brmangue-qgis` a thin, server-only client: every field it
sends maps directly onto the `ModelExecutor`/platform contract already
documented in this chapter, and no part of the simulation itself runs on
the researcher's machine.

## Exercises

1. Bring the platform up locally (9.3) and submit `brmangue_raster`
   through the API directly with `curl`, then repeat the same submission
   through the `brmangue-qgis` plugin (9.6). Compare the two JSON
   payloads — they should be structurally identical, since
   `payload_builder.build_payload()` targets the same `POST /submit_job`
   contract.
2. Every request except health checks must carry a valid `X-API-Key`
   (`docs/platform/security.md`), checked against `API_KEYS` from the
   environment. Submit a job with a wrong key and confirm you get an
   HTTP 403. Where in `.env` would you add a second researcher's key
   without invalidating the first?
3. Add a new TOML file to a local clone of `dissmodel-configs` for a
   model you haven't seen registered yet (pick any package from Chapter
   7 or 8), following the `<model>_<substrate>.toml` convention and the
   `brmangue_raster.toml` structure. What three fields under `[model]`
   does the platform need at minimum to resolve and run your executor?
4. `POST /submit_job_inline` bypasses the registry for rapid Jupyter
   exploration, at the cost of `model_commit` being recorded as
   `"local-inline"`. Explain why that specific field is what breaks
   reproducibility, referencing the `ExperimentRecord` fields from
   Chapter 2.
5. Trace `brmangue-qgis`'s `BrmangueTask._poll()` loop and identify what
   happens if the platform returns a transient HTTP error mid-poll
   (hint: look at the `except` branch — does it fail the job or retry?).

## Summary

The DisSModel Platform turns the local `ModelExecutor` contract from
Chapter 2 into a shared, multi-user service without changing a single
line of model code: JupyterLab, a FastAPI gateway, Redis-backed workers,
and MinIO storage form a containerized MVP (explicitly flagged as such —
security hardening and extreme performance are "Fase 2" concerns, not
today's). `dissmodel-configs` is the piece that keeps executors
pluggable — a TOML file per `<model>_<substrate>` combination declares
the package, the executor class, default parameters, and a canonical
band/column vocabulary, git-synced into the platform every 15 minutes and
snapshotted into every `ExperimentRecord` for reproducibility.
`brmangue-qgis` shows what a thin client on top of this looks like in
practice: it never runs `dissmodel` or `brmangue-dissmodel` locally, only
submits a payload, polls `/job/{id}`, and renders whatever comes back —
proof that the API contract, not any particular client, is the real
product surface of the platform.
