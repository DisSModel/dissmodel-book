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

*TODO: expand with the full architecture diagram (Researcher → Browser →
JupyterLab → API → Workers → MinIO), citing `docs/architecture.md` and
`docs/platform/*` from the original repository.*

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

*TODO: expand with the full convention table and real examples
(`brmangue_raster.toml` as the canonical reference).*

## 9.6 Usage example: triggering experiments from QGIS

The [`brmangue-qgis`](https://github.com/DisSModel/brmangue-qgis) plugin is
a concrete client of the platform's REST API described above — it lets a
researcher select layers directly in QGIS and submit an experiment without
touching the command line.

*TODO: this example depends on documentation that doesn't exist yet in the
source repository (`brmangue-qgis` currently has no README). Observed file
structure: `dialog.py`, `experiment_panel.py`, `payload_builder.py`,
`plugin.py`, `symbology.py`, `task.py`, `metadata.txt` — suggesting the
plugin builds a payload (`payload_builder.py`) and submits it to the
platform's API (Section 9.3).*

*TODO: document the end-to-end flow: install plugin → select layers →
configure experiment panel → submit → monitor via JupyterLab/API.*

## Exercises

*TODO: write exercises.*

## Summary

*TODO: summarize the chapter's key takeaways.*
