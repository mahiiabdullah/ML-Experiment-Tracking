# ML Experiment Tracking Platform

> **Author:** Mahi Abdullah
> **Project-12** ML Experiment Tracking Platform - A local ML experimentation system.
> A local-first Flask + SQLite experiment tracker. Log hyperparameters, per-epoch metrics, and final results from your training scripts and explore them in a Tailwind dashboard — no cloud, no accounts, no Docker.

![Dashboard](docs/screenshot-dashboard.png)

## Screenshots

| Dashboard | Detail | Compare |
| :-------: | :----: | :-----: |
| ![Dashboard](docs/screenshot-dashboard.png) | ![Detail](docs/screenshot-detail.png) | ![Compare](docs/screenshot-compare.png) |

---

## Table of Contents

1. [Overview](#overview)
2. [Features](#features)
3. [Stack](#stack)
4. [Project Layout](#project-layout)
5. [Setup](#setup)
6. [SDK Usage](#sdk-usage)
7. [API](#api)
8. [How It Works](#how-it-works)
9. [Troubleshooting](#troubleshooting)
10. [Roadmap](#roadmap)
11. [License](#license)

---

## Overview

`ML Experiment Tracking Platform` is a self-contained Python app for ML practitioners who run training on their own machine and want clean visualizations without spinning up a cloud service. You wrap a section of your training code in a `with start_run(...) as run:` block, log params and metrics, and the run appears on a local dashboard with sortable columns, a tags filter, per-epoch line charts, and a side-by-side compare view.

The whole stack is intentionally tiny: a small Python SDK, a Flask server serving both HTML and JSON, and one SQLite file. No build step, no JS framework, no auth, no external services.

---

## Features

- One-line SDK: `with start_run("name", project="p") as run: ...`
- Per-epoch metrics with `run.log_metric(name, value, step)`
- Final metrics with `run.log_metrics({...})` and inline notes
- Dashboard with project filter, search, and status badges
- Detail page with Chart.js line chart of every metric
- A/B compare with diff-highlighted params and grouped bar chart
- JSON API for every page; PATCH endpoint for inline notes editing
- Single-file portability — back up or share by copying `experiments.db`

---

## Stack

| Layer    | Tech                                  | Role                                                          |
| -------- | ------------------------------------- | ------------------------------------------------------------- |
| SDK      | Python stdlib + `sqlite3`             | `start_run()` / `Run` writes directly to SQLite                |
| Server   | Flask 3.x                             | HTML pages and JSON API on `127.0.0.1:5000`                    |
| Storage  | SQLite (stdlib)                       | One file; schema + `end_time` migration on boot                |
| UI       | Jinja2 + Tailwind (CDN)               | `base.html`, `dashboard.html`, `detail.html`, `compare.html`  |

| Charts   | Chart.js 4.4.1 via jsdelivr           | Line chart on detail; grouped bar on compare                   |

---

## Project Layout

```text
.
├── app.py                 # Flask app factory, routes, error handlers
├── database.py            # SQLite connection, schema, migrations
├── tracker.py             # SDK: start_run() / Run / context manager
├── sample_run.py          # Seeds 3 demo runs into "Demo Project"
├── requirements.txt       # flask>=3.0,<4.0
├── templates/             # base, dashboard, detail, compare, 404
├── static/main.js         # Reserved for shared client helpers
└── docs/                  # screenshot-dashboard/detail/compare.png
```

---

## Setup

**Prerequisites:** Python 3.9+, pip, a modern browser. No other tooling.

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt
python sample_run.py        # optional: seed 3 demo runs
python app.py               # serves on http://127.0.0.1:5000
```

Open <http://127.0.0.1:5000/> in your browser. Flask debug mode is on, so template edits reload automatically.

**Environment variables (all optional):**

| Var                | Default       | Purpose                                |
| ------------------ | ------------- | -------------------------------------- |
| `FLASK_RUN_HOST`   | `127.0.0.1`   | Bind address                           |
| `FLASK_RUN_PORT`   | `5000`        | TCP port                               |
| `FLASK_DEBUG`      | `True`        | Set `0` in production                  |

---

## SDK Usage

```python
from tracker import start_run

with start_run("resnet50-v3", project="cv", tags=["imagenet", "baseline"]) as run:
    run.log_params({"lr": 1e-3, "batch_size": 64, "optimizer": "adamw"})

    for epoch in range(10):
        train_loss = train_one_epoch(...)
        val_acc = evaluate(...)
        run.log_metric("train_loss", train_loss, step=epoch)
        run.log_metric("val_acc", val_acc, step=epoch)

    run.log_metrics({"final_acc": val_acc, "final_loss": train_loss})
    run.set_notes("Best run so far.")
```

If the block raises, the run is auto-marked `failed`. If it exits normally, it is `completed`.

---

## API

Base URL: `http://127.0.0.1:5000`. Errors return `{"error": str, "detail": str|null}`.

### `GET /api/experiments`

Query: `project=<exact>`, `search=<substring>`. Returns runs newest first.

```http
GET /api/experiments?project=Demo%20Project
```

```json
[
  {
    "id": 2,
    "name": "Demo · big-model run",
    "project": "Demo Project",
    "tags": ["demo", "transformer"],
    "params": {"lr": 0.001, "epochs": 10, "batch_size": 64, "model": "transformer-xl"},
    "metrics": {"final_loss": 0.182, "final_acc": 0.94},
    "notes": "Best run so far.",
    "status": "completed",
    "created_at": "2026-06-17T10:21:02",
    "end_time": "2026-06-17T10:21:03"
  }
]
```

### `GET /api/experiment/<id>`

Single run plus its `epoch_metrics` array. `404` if id is unknown.

### `GET /api/compare?ids=<id1>,<id2>`

Two runs in the requested order. `400` on fewer than two or duplicate ids.

### `PATCH /api/experiment/<id>/notes`

Body: `{"notes": "..."}`. `400` on missing key or non-JSON body.

```http
PATCH /api/experiment/2/notes
Content-Type: application/json

{"notes": "Switched to cosine LR schedule."}
```

```json
{"id": 2, "notes": "Switched to cosine LR schedule."}
```

---

## How It Works

```python
# app.py — create_app()
def create_app():
    app = Flask(__name__)
    init_db()                              # create tables + end_time migration
    app.register_blueprint(routes)         # /, /experiment/<id>, /compare, /api/...
    app.register_error_handler(404, render_404)
    app.register_error_handler(500, json_500)
    return app
```

1. **`init_db()`** — opens `experiments.db` (creates if missing), creates `experiments` and `epoch_metrics` tables, runs an idempotent `ALTER TABLE … ADD COLUMN end_time TEXT` so older DBs upgrade in place.
2. **`start_run(name, project, tags)`** — inserts a row with `status="running"`, `created_at=now`, and returns a `Run` bound to the new id.
3. **`run.log_params({...})` / `log_metrics({...})`** — `json.dumps` the dict and `UPDATE` the column. Later calls merge into earlier ones.
4. **`run.log_metric(name, value, step)`** — inserts one row into `epoch_metrics` per call. The detail page groups by `metric_name` for Chart.js.
5. **`Run.__exit__`** — calls `finish()` (idempotent), which sets `end_time` and `status="completed"`. An exception in flight triggers `status="failed"`.
6. **Flask routes** — every `GET /api/...` is wrapped in a `_with_db` decorator that opens a per-request SQLite connection and returns stable JSON.
7. **Notes PATCH** — the detail page's `<textarea>` fires `fetch("/api/experiment/<id>/notes", { method: "PATCH", body: ... })` on blur.

---

## Troubleshooting

| Problem                                              | Cause                                              | Fix                                                          |
| ---------------------------------------------------- | -------------------------------------------------- | ------------------------------------------------------------ |
| `ModuleNotFoundError: No module named 'flask'`       | Dependencies not installed                         | `pip install -r requirements.txt`                         |
| `sqlite3.OperationalError: no such table: experiments` | `experiments.db` deleted without re-init          | `python -c "import database; database.init_db()"`        |
| Dashboard shows zero rows after `sample_run.py`      | Stale `app.py` process reopened the DB             | Stop old `python app.py`, then re-run `sample_run.py`     |
| Port 5000 already in use                             | Another process bound to 5000                      | `FLASK_RUN_PORT=5001 python app.py`                       |
| Tailwind / Chart.js unstyled on first load           | No internet access on first request                | Refresh once online, or vendor the CDN files locally      |
| `notes` PATCH returns 400                            | Body not JSON or missing `notes` key               | Send `Content-Type: application/json` with `{"notes": ""}`|
| Compare shows "Nothing to compare yet"               | One dropdown unset, or only one run exists         | Pick two distinct runs, or run `sample_run.py`            |
| `werkzeug` reloader complains                        | Non-ASCII path or symlink loop on Windows          | Move project to an ASCII path without spaces              |

---

## Roadmap

- [ ] Async-friendly `start_run` for `asyncio` training loops
- [ ] `run.log_artifact(path)` for checkpoints and config files
- [ ] Nested runs for hyperparameter sweeps
- [ ] CSV / JSON export of `/api/experiments`
- [ ] WebSocket push so the dashboard updates without refresh
- [ ] `ML_TRACKER_DB` env var to support multiple projects
- [ ] Pagination on `/api/experiments`
- [ ] Sparkline previews on dashboard cards
- [ ] Dark mode toggle persisted to `localStorage`
- [ ] `pytest` suite + GitHub Actions CI

---

## License

MIT — add a `LICENSE` file with the MIT template if it isn't present yet.
