# Attractor Pipeline Dashboard — Implementation Plan

> **Execution:** Use the subagent-driven-development workflow to implement this plan.

**Goal:** Build a real-time pipeline monitoring dashboard (FastAPI backend + React frontend) that visualizes Attractor pipeline execution graphs using CXDB as the single source of truth.

**Architecture:** Standalone web app — FastAPI backend queries CXDB's HTTP API and serves JSON to a React frontend. React Flow renders pipeline DAGs with ELK.js layout. The backend has zero dependency on amplifier-core. A `--mock` flag provides hardcoded sample data so the frontend can be developed before CXDB pipeline events (PR #18) land.

**Tech Stack:** Python 3.11+ / FastAPI / httpx / uvicorn (backend); React 18 / Vite / TypeScript / React Flow v12 / ELK.js / ts-graphviz / React Router v7 (frontend)

**Design Doc:** `docs/plans/2026-02-24-pipeline-dashboard-design.md`

---

## Task 1: Scaffold the Repo

Create the `amplifier-dashboard-attractor/` directory at `/home/bkrabach/dev/attractor-next/amplifier-dashboard-attractor/`. Initialize git. Create the backend Python package structure with `pyproject.toml`. Create the frontend directory stub. Add `.gitignore`. Add as a git submodule to the attractor-next workspace.

**Files:**
- Create: `amplifier-dashboard-attractor/.gitignore`
- Create: `amplifier-dashboard-attractor/README.md`
- Create: `amplifier-dashboard-attractor/pyproject.toml`
- Create: `amplifier-dashboard-attractor/amplifier_dashboard_attractor/__init__.py`
- Create: `amplifier-dashboard-attractor/amplifier_dashboard_attractor/server.py` (placeholder)
- Create: `amplifier-dashboard-attractor/amplifier_dashboard_attractor/routes/__init__.py`
- Create: `amplifier-dashboard-attractor/tests/__init__.py`

**Step 1: Create the repo directory and initialize git**

```bash
cd /home/bkrabach/dev/attractor-next
mkdir -p amplifier-dashboard-attractor
cd amplifier-dashboard-attractor
git init
```

**Step 2: Create `.gitignore`**

Write `amplifier-dashboard-attractor/.gitignore`:

```gitignore
# Python
__pycache__/
*.py[cod]
*.egg-info/
dist/
build/
.eggs/
*.egg
.venv/
.ruff_cache/
.pytest_cache/
.mypy_cache/
.pyright/

# uv
uv.lock

# Node / Frontend
frontend/node_modules/
frontend/dist/
frontend/.vite/

# IDE
.vscode/
.idea/

# OS
.DS_Store
```

**Step 3: Create `pyproject.toml`**

Write `amplifier-dashboard-attractor/pyproject.toml`:

```toml
[project]
name = "amplifier-dashboard-attractor"
version = "0.1.0"
description = "Pipeline monitoring dashboard for Attractor — visualizes pipeline execution graphs via CXDB"
license = "MIT"
requires-python = ">=3.11"
authors = [
    { name = "Microsoft MADE:Explorations Team" },
]
dependencies = [
    "fastapi>=0.115",
    "uvicorn[standard]>=0.34",
    "httpx>=0.24",
]

[project.scripts]
dashboard = "amplifier_dashboard_attractor.server:main"

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.uv]
package = true

[tool.hatch.build.targets.wheel]
packages = ["amplifier_dashboard_attractor"]

[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "--import-mode=importlib"
asyncio_mode = "auto"

[dependency-groups]
dev = [
    "pytest>=8.0",
    "pytest-asyncio>=0.24",
    "httpx>=0.24",
]
```

**Step 4: Create the Python package skeleton**

Write `amplifier-dashboard-attractor/amplifier_dashboard_attractor/__init__.py`:

```python
"""Attractor Pipeline Dashboard — monitors pipeline execution via CXDB."""
```

Write `amplifier-dashboard-attractor/amplifier_dashboard_attractor/server.py`:

```python
"""FastAPI server — placeholder, implemented in Task 2."""
```

Write `amplifier-dashboard-attractor/amplifier_dashboard_attractor/routes/__init__.py`:

```python
```

Write `amplifier-dashboard-attractor/tests/__init__.py`:

```python
```

**Step 5: Create `README.md`**

Write `amplifier-dashboard-attractor/README.md`:

```markdown
# Attractor Pipeline Dashboard

Real-time pipeline monitoring dashboard for the Attractor pipeline orchestration system.
Visualizes directed pipeline graphs with execution state overlaid.

## Quick Start

Backend:
    cd amplifier-dashboard-attractor
    uv sync
    uv run dashboard --mock

Frontend (dev):
    cd frontend
    npm install
    npm run dev

## Architecture

- **Backend:** FastAPI server querying CXDB HTTP API, serving REST + static SPA
- **Frontend:** React + React Flow + ELK.js for DAG visualization
- **Data source:** CXDB (single source of truth) — no amplifier-core dependency
```

**Step 6: Install dependencies and verify the package installs**

```bash
cd /home/bkrabach/dev/attractor-next/amplifier-dashboard-attractor
uv sync
```

Expected: resolves and installs fastapi, uvicorn, httpx without errors.

**Step 7: Add as submodule to attractor-next workspace**

```bash
cd /home/bkrabach/dev/attractor-next
git submodule add ./amplifier-dashboard-attractor amplifier-dashboard-attractor
```

Note: since the dashboard repo is local-only (not pushed anywhere), the submodule URL will be a local path. This is fine for development.

**Step 8: Initial commit**

```bash
cd /home/bkrabach/dev/attractor-next/amplifier-dashboard-attractor
git add -A
git commit -m "chore: scaffold dashboard repo (FastAPI + React project structure)"
```

---

## Task 2: Backend — FastAPI Server Skeleton

Create the FastAPI application with a health check endpoint, CORS configuration, a `--mock` CLI flag, and a placeholder for static file serving.

**Files:**
- Create: `amplifier-dashboard-attractor/amplifier_dashboard_attractor/server.py`
- Create: `amplifier-dashboard-attractor/tests/test_server.py`

**Step 1: Write the failing test**

Write `amplifier-dashboard-attractor/tests/test_server.py`:

```python
"""Tests for the FastAPI server skeleton."""

import pytest
from httpx import ASGITransport, AsyncClient

from amplifier_dashboard_attractor.server import create_app


@pytest.mark.asyncio
async def test_health_endpoint():
    app = create_app(mock=True)
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as client:
        resp = await client.get("/api/health")
    assert resp.status_code == 200
    body = resp.json()
    assert body["status"] == "ok"
    assert "mock" in body


@pytest.mark.asyncio
async def test_health_shows_mock_mode():
    app = create_app(mock=True)
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as client:
        resp = await client.get("/api/health")
    assert resp.json()["mock"] is True


@pytest.mark.asyncio
async def test_cors_headers_present():
    app = create_app(mock=True)
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as client:
        resp = await client.options(
            "/api/health",
            headers={
                "Origin": "http://localhost:5173",
                "Access-Control-Request-Method": "GET",
            },
        )
    assert resp.status_code == 200
    assert "access-control-allow-origin" in resp.headers
```

**Step 2: Run test to verify it fails**

```bash
cd /home/bkrabach/dev/attractor-next/amplifier-dashboard-attractor
uv run pytest tests/test_server.py -q --tb=short
```

Expected: FAIL — `create_app` not defined (server.py is a placeholder).

**Step 3: Write the implementation**

Write `amplifier-dashboard-attractor/amplifier_dashboard_attractor/server.py`:

```python
"""FastAPI server for the Attractor Pipeline Dashboard.

Thin stateless server that queries CXDB and serves JSON + static SPA files.
Supports a --mock flag for development without a live CXDB instance.
"""

from __future__ import annotations

import argparse
from pathlib import Path

import uvicorn
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from fastapi.staticfiles import StaticFiles


def create_app(*, mock: bool = False) -> FastAPI:
    """Create and configure the FastAPI application."""
    app = FastAPI(title="Attractor Pipeline Dashboard", version="0.1.0")

    # Store mock flag in app state so routes can access it
    app.state.mock = mock

    # CORS — allow Vite dev server during development
    app.add_middleware(
        CORSMiddleware,
        allow_origins=["http://localhost:5173", "http://localhost:3000"],
        allow_credentials=True,
        allow_methods=["*"],
        allow_headers=["*"],
    )

    @app.get("/api/health")
    async def health():
        return {"status": "ok", "mock": app.state.mock}

    # Serve frontend static files if the dist/ directory exists
    frontend_dist = Path(__file__).parent.parent / "frontend" / "dist"
    if frontend_dist.is_dir():
        app.mount("/", StaticFiles(directory=str(frontend_dist), html=True), name="spa")

    return app


def main():
    """CLI entry point: `dashboard [--mock] [--port PORT]`."""
    parser = argparse.ArgumentParser(description="Attractor Pipeline Dashboard")
    parser.add_argument("--mock", action="store_true", help="Use mock data instead of CXDB")
    parser.add_argument("--port", type=int, default=8050, help="Server port (default: 8050)")
    parser.add_argument("--host", default="127.0.0.1", help="Bind address (default: 127.0.0.1)")
    args = parser.parse_args()

    app = create_app(mock=args.mock)
    uvicorn.run(app, host=args.host, port=args.port)


if __name__ == "__main__":
    main()
```

**Step 4: Run test to verify it passes**

```bash
cd /home/bkrabach/dev/attractor-next/amplifier-dashboard-attractor
uv run pytest tests/test_server.py -q --tb=short
```

Expected: 3 passed.

**Step 5: Commit**

```bash
cd /home/bkrabach/dev/attractor-next/amplifier-dashboard-attractor
git add -A && git commit -m "feat: FastAPI server skeleton with health check and CORS"
```

---

## Task 3: Backend — Mock Data

Create realistic mock `PipelineRunState` data that the `--mock` flag returns. This data drives all frontend development. The mock represents a 6-node pipeline where 3 nodes have completed, 1 is running, 1 is pending, and 1 has failed — exercising every visual state.

**Reference data model:** The `PipelineRunState` dataclass is defined at `amplifier-bundle-attractor/modules/hooks-pipeline-observability/amplifier_module_hooks_pipeline_observability/models.py`. The mock data matches `PipelineRunState.to_dict()` output format — all datetimes are ISO strings, all dataclasses are plain dicts.

**Files:**
- Create: `amplifier-dashboard-attractor/amplifier_dashboard_attractor/mock_data.py`
- Create: `amplifier-dashboard-attractor/tests/test_mock_data.py`

**Step 1: Write the failing test**

Write `amplifier-dashboard-attractor/tests/test_mock_data.py`:

```python
"""Tests for mock pipeline data."""

from amplifier_dashboard_attractor.mock_data import MOCK_PIPELINES, get_mock_pipeline


def test_mock_pipelines_is_nonempty_list():
    assert isinstance(MOCK_PIPELINES, list)
    assert len(MOCK_PIPELINES) >= 2


def test_mock_pipeline_has_required_fields():
    p = MOCK_PIPELINES[0]
    assert "pipeline_id" in p
    assert "dot_source" in p
    assert "status" in p
    assert "nodes" in p
    assert "node_runs" in p
    assert "total_tokens_in" in p
    assert "nodes_completed" in p
    assert "nodes_total" in p


def test_mock_pipeline_dot_source_is_valid_dot():
    p = MOCK_PIPELINES[0]
    dot = p["dot_source"]
    assert "digraph" in dot
    assert "->" in dot


def test_mock_pipeline_has_multiple_node_states():
    """The mock should exercise multiple visual states for frontend development."""
    p = MOCK_PIPELINES[0]
    statuses = set()
    for node_id, runs in p["node_runs"].items():
        if runs:
            statuses.add(runs[-1]["status"])
    assert len(statuses) >= 3, f"Expected >=3 distinct node statuses, got {statuses}"


def test_get_mock_pipeline_by_context_id():
    """Mock pipelines are keyed by a fake context_id (integer)."""
    result = get_mock_pipeline(1001)
    assert result is not None
    assert result["pipeline_id"] == MOCK_PIPELINES[0]["pipeline_id"]


def test_get_mock_pipeline_unknown_id_returns_none():
    assert get_mock_pipeline(9999) is None
```

**Step 2: Run test to verify it fails**

```bash
cd /home/bkrabach/dev/attractor-next/amplifier-dashboard-attractor
uv run pytest tests/test_mock_data.py -q --tb=short
```

Expected: FAIL — `mock_data` module does not exist.

**Step 3: Write the implementation**

Write `amplifier-dashboard-attractor/amplifier_dashboard_attractor/mock_data.py`:

```python
"""Mock PipelineRunState data for --mock mode.

Provides realistic pipeline state dicts matching PipelineRunState.to_dict() output.
This enables full frontend development without a live CXDB instance.

Data model reference:
  amplifier-bundle-attractor/modules/hooks-pipeline-observability/
  amplifier_module_hooks_pipeline_observability/models.py
"""

from __future__ import annotations

# Each mock pipeline is keyed by a fake context_id for URL routing.
# context_id -> pipeline state dict
_MOCK_CONTEXT_IDS: list[int] = [1001, 1002, 1003]

MOCK_PIPELINES: list[dict] = [
    # Pipeline 1: "Research and Summarize" — mixed states (the main dev pipeline)
    {
        "pipeline_id": "research-summarize-001",
        "dot_source": (
            'digraph "Research and Summarize" {\n'
            "  rankdir=LR;\n"
            '  start [label="Start" shape=ellipse];\n'
            '  gather [label="Gather Sources" shape=box];\n'
            '  analyze [label="Analyze Content" shape=box];\n'
            '  synthesize [label="Synthesize" shape=box];\n'
            '  review [label="Review Quality" shape=diamond];\n'
            '  publish [label="Publish" shape=box];\n'
            "  start -> gather;\n"
            "  gather -> analyze;\n"
            "  analyze -> synthesize;\n"
            "  synthesize -> review;\n"
            '  review -> publish [label="pass"];\n'
            '  review -> synthesize [label="retry"];\n'
            "}\n"
        ),
        "goal": "Research the topic and produce a comprehensive summary",
        "nodes": {
            "start": {"id": "start", "label": "Start", "shape": "ellipse", "type": "", "prompt": ""},
            "gather": {
                "id": "gather",
                "label": "Gather Sources",
                "shape": "box",
                "type": "llm",
                "prompt": "Find and collect relevant sources on the given topic.",
            },
            "analyze": {
                "id": "analyze",
                "label": "Analyze Content",
                "shape": "box",
                "type": "llm",
                "prompt": "Analyze the collected sources for key themes.",
            },
            "synthesize": {
                "id": "synthesize",
                "label": "Synthesize",
                "shape": "box",
                "type": "llm",
                "prompt": "Combine analysis into a coherent summary.",
            },
            "review": {
                "id": "review",
                "label": "Review Quality",
                "shape": "diamond",
                "type": "llm",
                "prompt": "Evaluate whether the summary meets quality standards.",
            },
            "publish": {"id": "publish", "label": "Publish", "shape": "box", "type": "tool", "prompt": ""},
        },
        "edges": [
            {"from_node": "start", "to_node": "gather", "label": "", "condition": "", "weight": 0},
            {"from_node": "gather", "to_node": "analyze", "label": "", "condition": "", "weight": 0},
            {"from_node": "analyze", "to_node": "synthesize", "label": "", "condition": "", "weight": 0},
            {"from_node": "synthesize", "to_node": "review", "label": "", "condition": "", "weight": 0},
            {"from_node": "review", "to_node": "publish", "label": "pass", "condition": "quality >= 0.8", "weight": 1},
            {"from_node": "review", "to_node": "synthesize", "label": "retry", "condition": "quality < 0.8", "weight": 0},
        ],
        "status": "running",
        "current_node": "synthesize",
        "execution_path": ["start", "gather", "analyze", "synthesize"],
        "branches_taken": [],
        "node_runs": {
            "start": [
                {
                    "status": "success",
                    "attempt": 1,
                    "started_at": "2026-02-24T02:30:00",
                    "completed_at": "2026-02-24T02:30:00",
                    "duration_ms": 50,
                    "outcome_notes": "Pipeline initialized",
                    "llm_calls": 0,
                    "tokens_in": 0,
                    "tokens_out": 0,
                    "tokens_cached": 0,
                },
            ],
            "gather": [
                {
                    "status": "success",
                    "attempt": 1,
                    "started_at": "2026-02-24T02:30:01",
                    "completed_at": "2026-02-24T02:30:12",
                    "duration_ms": 11200,
                    "outcome_notes": "Found 8 relevant sources",
                    "llm_calls": 3,
                    "tokens_in": 4200,
                    "tokens_out": 1800,
                    "tokens_cached": 500,
                },
            ],
            "analyze": [
                {
                    "status": "success",
                    "attempt": 1,
                    "started_at": "2026-02-24T02:30:13",
                    "completed_at": "2026-02-24T02:30:28",
                    "duration_ms": 15400,
                    "outcome_notes": "Identified 5 key themes across sources",
                    "llm_calls": 2,
                    "tokens_in": 6100,
                    "tokens_out": 2400,
                    "tokens_cached": 1200,
                },
            ],
            "synthesize": [
                {
                    "status": "running",
                    "attempt": 1,
                    "started_at": "2026-02-24T02:30:29",
                    "completed_at": None,
                    "duration_ms": 0,
                    "outcome_notes": None,
                    "llm_calls": 1,
                    "tokens_in": 3200,
                    "tokens_out": 0,
                    "tokens_cached": 0,
                },
            ],
        },
        "edge_decisions": [
            {
                "from_node": "gather",
                "evaluated_edges": [],
                "selected_edge": {"from_node": "gather", "to_node": "analyze", "label": "", "condition": "", "weight": 0},
                "reason": "default",
            },
        ],
        "loop_iterations": {},
        "goal_gate_checks": [],
        "parallel_branches": {},
        "subgraph_runs": {},
        "human_interactions": [],
        "supervisor_cycles": {},
        "total_elapsed_ms": 29500,
        "total_llm_calls": 6,
        "total_tokens_in": 13500,
        "total_tokens_out": 4200,
        "total_tokens_cached": 1700,
        "total_tokens_reasoning": 0,
        "nodes_completed": 3,
        "nodes_total": 6,
        "timing": {"start": 50, "gather": 11200, "analyze": 15400},
        "errors": [],
    },
    # Pipeline 2: "Code Review" — completed successfully
    {
        "pipeline_id": "code-review-042",
        "dot_source": (
            'digraph "Code Review" {\n'
            "  rankdir=LR;\n"
            '  fetch [label="Fetch PR"];\n'
            '  lint [label="Lint Check"];\n'
            '  review [label="AI Review"];\n'
            '  report [label="Write Report"];\n'
            "  fetch -> lint;\n"
            "  lint -> review;\n"
            "  review -> report;\n"
            "}\n"
        ),
        "goal": "Review pull request #187 for code quality issues",
        "nodes": {
            "fetch": {"id": "fetch", "label": "Fetch PR", "shape": "box", "type": "tool", "prompt": ""},
            "lint": {"id": "lint", "label": "Lint Check", "shape": "box", "type": "tool", "prompt": ""},
            "review": {"id": "review", "label": "AI Review", "shape": "box", "type": "llm", "prompt": "Review code for quality."},
            "report": {"id": "report", "label": "Write Report", "shape": "box", "type": "llm", "prompt": "Summarize findings."},
        },
        "edges": [
            {"from_node": "fetch", "to_node": "lint", "label": "", "condition": "", "weight": 0},
            {"from_node": "lint", "to_node": "review", "label": "", "condition": "", "weight": 0},
            {"from_node": "review", "to_node": "report", "label": "", "condition": "", "weight": 0},
        ],
        "status": "complete",
        "current_node": None,
        "execution_path": ["fetch", "lint", "review", "report"],
        "branches_taken": [],
        "node_runs": {
            "fetch": [{"status": "success", "attempt": 1, "started_at": "2026-02-24T02:00:00", "completed_at": "2026-02-24T02:00:02", "duration_ms": 2100, "outcome_notes": "Fetched 12 files", "llm_calls": 0, "tokens_in": 0, "tokens_out": 0, "tokens_cached": 0}],
            "lint": [{"status": "success", "attempt": 1, "started_at": "2026-02-24T02:00:03", "completed_at": "2026-02-24T02:00:05", "duration_ms": 1800, "outcome_notes": "3 warnings, 0 errors", "llm_calls": 0, "tokens_in": 0, "tokens_out": 0, "tokens_cached": 0}],
            "review": [{"status": "success", "attempt": 1, "started_at": "2026-02-24T02:00:06", "completed_at": "2026-02-24T02:00:22", "duration_ms": 16200, "outcome_notes": "Found 2 issues", "llm_calls": 4, "tokens_in": 8500, "tokens_out": 3200, "tokens_cached": 2000}],
            "report": [{"status": "success", "attempt": 1, "started_at": "2026-02-24T02:00:23", "completed_at": "2026-02-24T02:00:31", "duration_ms": 7800, "outcome_notes": "Report generated", "llm_calls": 1, "tokens_in": 4200, "tokens_out": 1800, "tokens_cached": 0}],
        },
        "edge_decisions": [],
        "loop_iterations": {},
        "goal_gate_checks": [],
        "parallel_branches": {},
        "subgraph_runs": {},
        "human_interactions": [],
        "supervisor_cycles": {},
        "total_elapsed_ms": 27900,
        "total_llm_calls": 5,
        "total_tokens_in": 12700,
        "total_tokens_out": 5000,
        "total_tokens_cached": 2000,
        "total_tokens_reasoning": 0,
        "nodes_completed": 4,
        "nodes_total": 4,
        "timing": {"fetch": 2100, "lint": 1800, "review": 16200, "report": 7800},
        "errors": [],
    },
    # Pipeline 3: "Data Pipeline" — failed at node 3
    {
        "pipeline_id": "data-pipeline-007",
        "dot_source": (
            'digraph "Data Pipeline" {\n'
            "  rankdir=LR;\n"
            '  ingest [label="Ingest Data"];\n'
            '  validate [label="Validate Schema"];\n'
            '  transform [label="Transform"];\n'
            '  load [label="Load to DB"];\n'
            "  ingest -> validate;\n"
            "  validate -> transform;\n"
            "  transform -> load;\n"
            "}\n"
        ),
        "goal": "Process and load the Q4 dataset",
        "nodes": {
            "ingest": {"id": "ingest", "label": "Ingest Data", "shape": "box", "type": "tool", "prompt": ""},
            "validate": {"id": "validate", "label": "Validate Schema", "shape": "box", "type": "tool", "prompt": ""},
            "transform": {"id": "transform", "label": "Transform", "shape": "box", "type": "llm", "prompt": "Transform data."},
            "load": {"id": "load", "label": "Load to DB", "shape": "box", "type": "tool", "prompt": ""},
        },
        "edges": [
            {"from_node": "ingest", "to_node": "validate", "label": "", "condition": "", "weight": 0},
            {"from_node": "validate", "to_node": "transform", "label": "", "condition": "", "weight": 0},
            {"from_node": "transform", "to_node": "load", "label": "", "condition": "", "weight": 0},
        ],
        "status": "failed",
        "current_node": "transform",
        "execution_path": ["ingest", "validate", "transform"],
        "branches_taken": [],
        "node_runs": {
            "ingest": [{"status": "success", "attempt": 1, "started_at": "2026-02-24T01:45:00", "completed_at": "2026-02-24T01:45:05", "duration_ms": 4800, "outcome_notes": "Ingested 2.4GB", "llm_calls": 0, "tokens_in": 0, "tokens_out": 0, "tokens_cached": 0}],
            "validate": [{"status": "success", "attempt": 1, "started_at": "2026-02-24T01:45:06", "completed_at": "2026-02-24T01:45:08", "duration_ms": 2100, "outcome_notes": "Schema valid", "llm_calls": 0, "tokens_in": 0, "tokens_out": 0, "tokens_cached": 0}],
            "transform": [
                {"status": "fail", "attempt": 1, "started_at": "2026-02-24T01:45:09", "completed_at": "2026-02-24T01:45:30", "duration_ms": 21000, "outcome_notes": "Rate limit exceeded on OpenAI API", "llm_calls": 2, "tokens_in": 15000, "tokens_out": 800, "tokens_cached": 0},
                {"status": "fail", "attempt": 2, "started_at": "2026-02-24T01:45:45", "completed_at": "2026-02-24T01:46:02", "duration_ms": 17000, "outcome_notes": "Rate limit still active", "llm_calls": 1, "tokens_in": 15000, "tokens_out": 0, "tokens_cached": 0},
            ],
        },
        "edge_decisions": [],
        "loop_iterations": {},
        "goal_gate_checks": [],
        "parallel_branches": {},
        "subgraph_runs": {},
        "human_interactions": [],
        "supervisor_cycles": {},
        "total_elapsed_ms": 62000,
        "total_llm_calls": 3,
        "total_tokens_in": 30000,
        "total_tokens_out": 800,
        "total_tokens_cached": 0,
        "total_tokens_reasoning": 0,
        "nodes_completed": 2,
        "nodes_total": 4,
        "timing": {"ingest": 4800, "validate": 2100, "transform": 38000},
        "errors": [
            {"node": "transform", "message": "Rate limit exceeded after 2 retries", "timestamp": "2026-02-24T01:46:02"},
        ],
    },
]


def get_mock_pipeline(context_id: int) -> dict | None:
    """Get a mock pipeline state by its fake context_id."""
    try:
        idx = _MOCK_CONTEXT_IDS.index(context_id)
        return MOCK_PIPELINES[idx]
    except (ValueError, IndexError):
        return None


def get_mock_fleet() -> list[dict]:
    """Return fleet summary for all mock pipelines.

    Each item contains the fields needed by the fleet view table.
    """
    fleet = []
    for i, p in enumerate(MOCK_PIPELINES):
        fleet.append(
            {
                "context_id": _MOCK_CONTEXT_IDS[i],
                "pipeline_id": p["pipeline_id"],
                "status": p["status"],
                "nodes_completed": p["nodes_completed"],
                "nodes_total": p["nodes_total"],
                "total_elapsed_ms": p["total_elapsed_ms"],
                "total_tokens_in": p["total_tokens_in"],
                "total_tokens_out": p["total_tokens_out"],
                "goal": p["goal"],
                "errors": p["errors"],
            }
        )
    return fleet
```

**Step 4: Run test to verify it passes**

```bash
cd /home/bkrabach/dev/attractor-next/amplifier-dashboard-attractor
uv run pytest tests/test_mock_data.py -q --tb=short
```

Expected: 6 passed.

**Step 5: Commit**

```bash
cd /home/bkrabach/dev/attractor-next/amplifier-dashboard-attractor
git add -A && git commit -m "feat: mock PipelineRunState data for --mock mode"
```

---

## Task 4: Backend — CXDB HTTP Client

Create a thin async CXDB client that wraps the HTTP API endpoints the dashboard needs. Three methods: search for pipeline contexts (fleet), get turns for a context (pipeline detail), get metrics. Test against mock httpx responses.

**CXDB HTTP API reference** (from `fix-cxdb-pipeline/modules/cxdb-session-storage/amplifier_module_cxdb_session_storage/cxdb/http/client.py`):
- `GET /v1/contexts/search?q={CQL}&limit={N}` — CQL context search
- `GET /v1/contexts/{id}/turns?limit={N}` — turns with decoded payloads
- Response for search: `{"contexts": [...], "total_count": N}`
- Response for turns: `{"turns": [...]}`
- Each turn has: `turn_id`, `depth`, `data` (with `item_type`, `status`, `system` sub-object)
- System turns have: `data.system.kind`, `data.system.title`, `data.system.content`

**Files:**
- Create: `amplifier-dashboard-attractor/amplifier_dashboard_attractor/cxdb_client.py`
- Create: `amplifier-dashboard-attractor/tests/test_cxdb_client.py`

**Step 1: Write the failing test**

Write `amplifier-dashboard-attractor/tests/test_cxdb_client.py`:

```python
"""Tests for the CXDB HTTP client."""

import json
from unittest.mock import AsyncMock, patch

import pytest

from amplifier_dashboard_attractor.cxdb_client import CxdbClient


@pytest.fixture
def client():
    return CxdbClient(base_url="http://localhost:8080")


@pytest.mark.asyncio
async def test_search_pipelines(client):
    """search_pipelines should query CXDB with a label CQL query."""
    mock_response = AsyncMock()
    mock_response.status_code = 200
    mock_response.raise_for_status = lambda: None
    mock_response.json.return_value = {
        "contexts": [
            {
                "context_id": "42",
                "client_tag": "amplifier",
                "title": "",
                "head_turn_id": "100",
                "head_depth": 5,
                "is_live": True,
                "created_at_unix_ms": 1708732200000,
                "labels": ["pipeline_id:test-001", "pipeline_status:running"],
            },
        ],
        "total_count": 1,
    }

    with patch.object(client._http, "get", return_value=mock_response) as mock_get:
        results = await client.search_pipelines()

    mock_get.assert_called_once()
    call_args = mock_get.call_args
    assert "pipeline_status" in call_args.kwargs.get("params", {}).get("q", "")
    assert len(results) == 1
    assert results[0]["context_id"] == "42"
    assert "pipeline_status:running" in results[0]["labels"]


@pytest.mark.asyncio
async def test_get_pipeline_state(client):
    """get_pipeline_state should find the latest pipeline_state_snapshot turn."""
    snapshot_content = json.dumps({"pipeline_id": "test-001", "status": "running"})
    mock_response = AsyncMock()
    mock_response.status_code = 200
    mock_response.raise_for_status = lambda: None
    mock_response.json.return_value = {
        "turns": [
            {
                "turn_id": 50,
                "data": {
                    "item_type": "system",
                    "status": "complete",
                    "system": {
                        "kind": "info",
                        "title": "Pipeline started: test goal (4 nodes)",
                        "content": "test-001",
                    },
                },
            },
            {
                "turn_id": 55,
                "data": {
                    "item_type": "system",
                    "status": "complete",
                    "system": {
                        "kind": "info",
                        "title": "pipeline_state_snapshot",
                        "content": snapshot_content,
                    },
                },
            },
        ],
    }

    with patch.object(client._http, "get", return_value=mock_response):
        state = await client.get_pipeline_state(context_id=42)

    assert state is not None
    assert state["pipeline_id"] == "test-001"
    assert state["status"] == "running"


@pytest.mark.asyncio
async def test_get_pipeline_state_no_snapshot(client):
    """Returns None when no pipeline_state_snapshot turn exists."""
    mock_response = AsyncMock()
    mock_response.status_code = 200
    mock_response.raise_for_status = lambda: None
    mock_response.json.return_value = {
        "turns": [
            {
                "turn_id": 50,
                "data": {
                    "item_type": "system",
                    "status": "complete",
                    "system": {"kind": "info", "title": "Some other turn", "content": ""},
                },
            },
        ],
    }

    with patch.object(client._http, "get", return_value=mock_response):
        state = await client.get_pipeline_state(context_id=42)

    assert state is None


@pytest.mark.asyncio
async def test_get_node_events(client):
    """get_node_events should filter system turns for a specific node."""
    mock_response = AsyncMock()
    mock_response.status_code = 200
    mock_response.raise_for_status = lambda: None
    mock_response.json.return_value = {
        "turns": [
            {
                "turn_id": 60,
                "data": {
                    "item_type": "system",
                    "status": "complete",
                    "system": {"kind": "info", "title": "Pipeline node started: gather", "content": "gather"},
                },
            },
            {
                "turn_id": 61,
                "data": {
                    "item_type": "system",
                    "status": "complete",
                    "system": {"kind": "info", "title": "Pipeline node completed: gather", "content": "gather"},
                },
            },
            {
                "turn_id": 62,
                "data": {
                    "item_type": "system",
                    "status": "complete",
                    "system": {"kind": "info", "title": "Pipeline node started: analyze", "content": "analyze"},
                },
            },
        ],
    }

    with patch.object(client._http, "get", return_value=mock_response):
        events = await client.get_node_events(context_id=42, node_id="gather")

    assert len(events) == 2
    assert all("gather" in e["data"]["system"]["content"] for e in events)


@pytest.mark.asyncio
async def test_client_close():
    """Client should close its httpx client cleanly."""
    client = CxdbClient(base_url="http://localhost:8080")
    await client.close()
    assert client._http.is_closed
```

**Step 2: Run test to verify it fails**

```bash
cd /home/bkrabach/dev/attractor-next/amplifier-dashboard-attractor
uv run pytest tests/test_cxdb_client.py -q --tb=short
```

Expected: FAIL — `cxdb_client` module does not exist.

**Step 3: Write the implementation**

Write `amplifier-dashboard-attractor/amplifier_dashboard_attractor/cxdb_client.py`:

```python
"""Thin async CXDB HTTP client for the dashboard.

Wraps the three CXDB HTTP API endpoints the dashboard needs.
No dependency on amplifier-core — uses httpx directly.

CXDB HTTP API reference:
  GET /v1/contexts/search?q={CQL}&limit={N}  — CQL context search
  GET /v1/contexts/{id}/turns?limit={N}       — turns with decoded payloads
"""

from __future__ import annotations

import json
import logging

import httpx

logger = logging.getLogger(__name__)


class CxdbClient:
    """Async HTTP client for querying CXDB.

    Usage:
        client = CxdbClient(base_url="http://localhost:8080")
        results = await client.search_pipelines()
        await client.close()
    """

    def __init__(self, base_url: str = "http://localhost:8080", *, timeout: float = 30.0):
        self._http = httpx.AsyncClient(base_url=base_url, timeout=timeout)

    async def close(self):
        """Close the underlying HTTP client."""
        await self._http.aclose()

    async def search_pipelines(self, *, status: str | None = None, limit: int = 50) -> list[dict]:
        """Search CXDB for pipeline contexts.

        Uses CQL label search: `label = "pipeline_status:*"` for all pipelines,
        or `label = "pipeline_status:{status}"` for a specific status.

        Returns a list of context summary dicts.
        """
        if status:
            cql = f'label = "pipeline_status:{status}"'
        else:
            # Match any context with a pipeline_status label
            cql = 'label = "pipeline_status"'

        resp = await self._http.get("/v1/contexts/search", params={"q": cql, "limit": limit})
        resp.raise_for_status()
        data = resp.json()
        return data.get("contexts", [])

    async def get_pipeline_state(self, context_id: int) -> dict | None:
        """Get the latest PipelineRunState snapshot for a pipeline context.

        Fetches system turns and finds the most recent one with
        title="pipeline_state_snapshot". The content field contains the
        JSON-serialized PipelineRunState dict.

        Returns the parsed state dict, or None if no snapshot exists.
        """
        resp = await self._http.get(
            f"/v1/contexts/{context_id}/turns",
            params={"limit": 200, "include_unknown": 1, "bytes_render": "hex"},
        )
        resp.raise_for_status()
        turns = resp.json().get("turns", [])

        # Walk turns in reverse to find the latest snapshot
        snapshot_turn = None
        for turn in reversed(turns):
            data = turn.get("data", {})
            if data.get("item_type") != "system":
                continue
            system = data.get("system", {})
            if system.get("title") == "pipeline_state_snapshot":
                snapshot_turn = turn
                break

        if snapshot_turn is None:
            return None

        content = snapshot_turn["data"]["system"]["content"]
        try:
            return json.loads(content)
        except (json.JSONDecodeError, TypeError):
            logger.warning("Failed to parse pipeline_state_snapshot content for context %s", context_id)
            return None

    async def get_node_events(self, context_id: int, node_id: str) -> list[dict]:
        """Get system turns related to a specific node.

        Filters turns where the system content field contains the node_id.
        Returns matching turn dicts (most recent first).
        """
        resp = await self._http.get(
            f"/v1/contexts/{context_id}/turns",
            params={"limit": 200, "include_unknown": 1, "bytes_render": "hex"},
        )
        resp.raise_for_status()
        turns = resp.json().get("turns", [])

        node_turns = []
        for turn in turns:
            data = turn.get("data", {})
            if data.get("item_type") != "system":
                continue
            system = data.get("system", {})
            # Match turns where the content references this node
            if node_id in (system.get("content", "")):
                node_turns.append(turn)

        return node_turns
```

**Step 4: Run test to verify it passes**

```bash
cd /home/bkrabach/dev/attractor-next/amplifier-dashboard-attractor
uv run pytest tests/test_cxdb_client.py -q --tb=short
```

Expected: 5 passed.

**Step 5: Commit**

```bash
cd /home/bkrabach/dev/attractor-next/amplifier-dashboard-attractor
git add -A && git commit -m "feat: CXDB HTTP client (search, pipeline state, node events)"
```

---

## Task 5: Backend — REST Endpoints

Implement the three REST endpoints. Each endpoint checks `app.state.mock` — if true, returns mock data; otherwise queries CXDB.

**Endpoints:**
- `GET /api/pipelines` — fleet summary
- `GET /api/pipelines/{context_id}` — full pipeline state with DOT source
- `GET /api/pipelines/{context_id}/nodes/{node_id}` — node event history

**Files:**
- Create: `amplifier-dashboard-attractor/amplifier_dashboard_attractor/routes/pipelines.py`
- Create: `amplifier-dashboard-attractor/tests/test_routes.py`
- Modify: `amplifier-dashboard-attractor/amplifier_dashboard_attractor/server.py` (register router)

**Step 1: Write the failing test**

Write `amplifier-dashboard-attractor/tests/test_routes.py`:

```python
"""Tests for the pipeline REST endpoints (mock mode)."""

import pytest
from httpx import ASGITransport, AsyncClient

from amplifier_dashboard_attractor.server import create_app


@pytest.fixture
def app():
    return create_app(mock=True)


@pytest.fixture
async def client(app):
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as c:
        yield c


@pytest.mark.asyncio
async def test_get_pipelines_returns_list(client):
    resp = await client.get("/api/pipelines")
    assert resp.status_code == 200
    body = resp.json()
    assert isinstance(body, list)
    assert len(body) >= 2


@pytest.mark.asyncio
async def test_get_pipelines_item_has_required_fields(client):
    resp = await client.get("/api/pipelines")
    item = resp.json()[0]
    for field in ["context_id", "pipeline_id", "status", "nodes_completed", "nodes_total", "total_elapsed_ms"]:
        assert field in item, f"Missing field: {field}"


@pytest.mark.asyncio
async def test_get_pipeline_detail(client):
    resp = await client.get("/api/pipelines/1001")
    assert resp.status_code == 200
    body = resp.json()
    assert body["pipeline_id"] == "research-summarize-001"
    assert "dot_source" in body
    assert "digraph" in body["dot_source"]
    assert "nodes" in body
    assert "node_runs" in body


@pytest.mark.asyncio
async def test_get_pipeline_detail_not_found(client):
    resp = await client.get("/api/pipelines/9999")
    assert resp.status_code == 404


@pytest.mark.asyncio
async def test_get_node_detail(client):
    resp = await client.get("/api/pipelines/1001/nodes/gather")
    assert resp.status_code == 200
    body = resp.json()
    assert body["node_id"] == "gather"
    assert "info" in body
    assert "runs" in body
    assert len(body["runs"]) >= 1


@pytest.mark.asyncio
async def test_get_node_detail_not_found(client):
    resp = await client.get("/api/pipelines/1001/nodes/nonexistent")
    assert resp.status_code == 404
```

**Step 2: Run test to verify it fails**

```bash
cd /home/bkrabach/dev/attractor-next/amplifier-dashboard-attractor
uv run pytest tests/test_routes.py -q --tb=short
```

Expected: FAIL — routes not registered, endpoints return 404/405.

**Step 3: Write the route implementation**

Write `amplifier-dashboard-attractor/amplifier_dashboard_attractor/routes/pipelines.py`:

```python
"""Pipeline REST endpoints.

GET /api/pipelines                           — fleet summary
GET /api/pipelines/{context_id}              — full pipeline state
GET /api/pipelines/{context_id}/nodes/{node_id} — node detail
"""

from __future__ import annotations

from fastapi import APIRouter, HTTPException, Request

from amplifier_dashboard_attractor.mock_data import get_mock_fleet, get_mock_pipeline

router = APIRouter(prefix="/api/pipelines", tags=["pipelines"])


@router.get("")
async def list_pipelines(request: Request):
    """Fleet view: list all pipeline instances with summary data."""
    if request.app.state.mock:
        return get_mock_fleet()

    # Live CXDB path (implemented when CXDB integration is ready)
    cxdb = request.app.state.cxdb_client
    contexts = await cxdb.search_pipelines()
    # TODO: enrich each context with metrics from state snapshots
    return contexts


@router.get("/{context_id}")
async def get_pipeline(request: Request, context_id: int):
    """Pipeline detail: full PipelineRunState including DOT source."""
    if request.app.state.mock:
        state = get_mock_pipeline(context_id)
        if state is None:
            raise HTTPException(status_code=404, detail=f"Pipeline {context_id} not found")
        return state

    cxdb = request.app.state.cxdb_client
    state = await cxdb.get_pipeline_state(context_id)
    if state is None:
        raise HTTPException(status_code=404, detail=f"Pipeline {context_id} not found")
    return state


@router.get("/{context_id}/nodes/{node_id}")
async def get_node(request: Request, context_id: int, node_id: str):
    """Node detail: node info + all run attempts."""
    if request.app.state.mock:
        pipeline = get_mock_pipeline(context_id)
        if pipeline is None:
            raise HTTPException(status_code=404, detail=f"Pipeline {context_id} not found")
        node_info = pipeline.get("nodes", {}).get(node_id)
        if node_info is None:
            raise HTTPException(status_code=404, detail=f"Node {node_id} not found")
        runs = pipeline.get("node_runs", {}).get(node_id, [])
        return {
            "node_id": node_id,
            "info": node_info,
            "runs": runs,
            "edge_decisions": [
                d for d in pipeline.get("edge_decisions", []) if d["from_node"] == node_id
            ],
        }

    cxdb = request.app.state.cxdb_client
    events = await cxdb.get_node_events(context_id, node_id)
    if not events:
        raise HTTPException(status_code=404, detail=f"Node {node_id} not found")
    return {"node_id": node_id, "events": events}
```

**Step 4: Register the router in server.py**

Edit `amplifier-dashboard-attractor/amplifier_dashboard_attractor/server.py`. Replace the entire file:

```python
"""FastAPI server for the Attractor Pipeline Dashboard.

Thin stateless server that queries CXDB and serves JSON + static SPA files.
Supports a --mock flag for development without a live CXDB instance.
"""

from __future__ import annotations

import argparse
from pathlib import Path

import uvicorn
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from fastapi.staticfiles import StaticFiles

from amplifier_dashboard_attractor.routes.pipelines import router as pipelines_router


def create_app(*, mock: bool = False, cxdb_url: str = "http://localhost:8080") -> FastAPI:
    """Create and configure the FastAPI application."""
    app = FastAPI(title="Attractor Pipeline Dashboard", version="0.1.0")

    # Store config in app state so routes can access it
    app.state.mock = mock

    # CORS — allow Vite dev server during development
    app.add_middleware(
        CORSMiddleware,
        allow_origins=["http://localhost:5173", "http://localhost:3000"],
        allow_credentials=True,
        allow_methods=["*"],
        allow_headers=["*"],
    )

    @app.get("/api/health")
    async def health():
        return {"status": "ok", "mock": app.state.mock}

    # Register route modules
    app.include_router(pipelines_router)

    # Initialize CXDB client for live mode
    if not mock:
        from amplifier_dashboard_attractor.cxdb_client import CxdbClient

        cxdb = CxdbClient(base_url=cxdb_url)
        app.state.cxdb_client = cxdb

        @app.on_event("shutdown")
        async def shutdown_cxdb():
            await cxdb.close()

    # Serve frontend static files if the dist/ directory exists
    frontend_dist = Path(__file__).parent.parent / "frontend" / "dist"
    if frontend_dist.is_dir():
        app.mount("/", StaticFiles(directory=str(frontend_dist), html=True), name="spa")

    return app


def main():
    """CLI entry point: `dashboard [--mock] [--port PORT]`."""
    parser = argparse.ArgumentParser(description="Attractor Pipeline Dashboard")
    parser.add_argument("--mock", action="store_true", help="Use mock data instead of CXDB")
    parser.add_argument("--port", type=int, default=8050, help="Server port (default: 8050)")
    parser.add_argument("--host", default="127.0.0.1", help="Bind address (default: 127.0.0.1)")
    parser.add_argument("--cxdb-url", default="http://localhost:8080", help="CXDB HTTP API base URL")
    args = parser.parse_args()

    app = create_app(mock=args.mock, cxdb_url=args.cxdb_url)
    uvicorn.run(app, host=args.host, port=args.port)


if __name__ == "__main__":
    main()
```

**Step 5: Run ALL backend tests to verify everything passes**

```bash
cd /home/bkrabach/dev/attractor-next/amplifier-dashboard-attractor
uv run pytest tests/ -q --tb=short
```

Expected: 14 passed (3 server + 6 mock + 5 cxdb_client = 14... but test_routes has 6 tests too — let me recount). Actually: 3 (test_server) + 6 (test_mock_data) + 5 (test_cxdb_client) + 6 (test_routes) = 20 passed.

**Step 6: Verify the server starts in mock mode**

```bash
cd /home/bkrabach/dev/attractor-next/amplifier-dashboard-attractor
uv run dashboard --mock &
sleep 2
curl -s http://127.0.0.1:8050/api/health | python3 -m json.tool
curl -s http://127.0.0.1:8050/api/pipelines | python3 -m json.tool | head -20
curl -s http://127.0.0.1:8050/api/pipelines/1001 | python3 -m json.tool | head -20
kill %1
```

Expected: health returns `{"status": "ok", "mock": true}`, pipelines returns the fleet array, pipeline detail returns full state with `dot_source`.

**Step 7: Commit**

```bash
cd /home/bkrabach/dev/attractor-next/amplifier-dashboard-attractor
git add -A && git commit -m "feat: REST endpoints for fleet, pipeline detail, and node detail"
```

---

## Task 6: Frontend — Vite + React Scaffold

Initialize the React app with Vite in the `frontend/` subdirectory. Install all frontend dependencies. Set up the dark theme CSS variables from the design doc. Create three route stubs. Configure Vite to proxy `/api` requests to the FastAPI backend during development.

**Files:**
- Create: `amplifier-dashboard-attractor/frontend/package.json`
- Create: `amplifier-dashboard-attractor/frontend/vite.config.ts`
- Create: `amplifier-dashboard-attractor/frontend/tsconfig.json`
- Create: `amplifier-dashboard-attractor/frontend/tsconfig.node.json`
- Create: `amplifier-dashboard-attractor/frontend/index.html`
- Create: `amplifier-dashboard-attractor/frontend/src/main.tsx`
- Create: `amplifier-dashboard-attractor/frontend/src/App.tsx`
- Create: `amplifier-dashboard-attractor/frontend/src/theme.css`
- Create: `amplifier-dashboard-attractor/frontend/src/views/FleetView.tsx`
- Create: `amplifier-dashboard-attractor/frontend/src/views/PipelineView.tsx`
- Create: `amplifier-dashboard-attractor/frontend/src/views/NodeView.tsx`
- Create: `amplifier-dashboard-attractor/frontend/src/lib/types.ts`
- Create: `amplifier-dashboard-attractor/frontend/src/lib/api.ts`
- Create: `amplifier-dashboard-attractor/frontend/src/vite-env.d.ts`

**Step 1: Create `package.json`**

Write `amplifier-dashboard-attractor/frontend/package.json`:

```json
{
  "name": "attractor-dashboard",
  "private": true,
  "version": "0.1.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "tsc -b && vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "@xyflow/react": "^12.0.0",
    "elkjs": "^0.9.0",
    "react": "^18.3.0",
    "react-dom": "^18.3.0",
    "react-router-dom": "^7.0.0",
    "ts-graphviz": "^2.1.0"
  },
  "devDependencies": {
    "@types/react": "^18.3.0",
    "@types/react-dom": "^18.3.0",
    "@vitejs/plugin-react": "^4.3.0",
    "typescript": "~5.6.0",
    "vite": "^6.0.0"
  }
}
```

**Step 2: Create `vite.config.ts`**

Write `amplifier-dashboard-attractor/frontend/vite.config.ts`:

```typescript
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [react()],
  server: {
    port: 5173,
    proxy: {
      "/api": {
        target: "http://127.0.0.1:8050",
        changeOrigin: true,
      },
    },
  },
  build: {
    outDir: "dist",
  },
});
```

**Step 3: Create `tsconfig.json`**

Write `amplifier-dashboard-attractor/frontend/tsconfig.json`:

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "useDefineForClassFields": true,
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "skipLibCheck": true,
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "isolatedModules": true,
    "moduleDetection": "force",
    "noEmit": true,
    "jsx": "react-jsx",
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true,
    "noUncheckedSideEffectImports": true
  },
  "include": ["src"]
}
```

Write `amplifier-dashboard-attractor/frontend/tsconfig.node.json`:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["ES2023"],
    "module": "ESNext",
    "skipLibCheck": true,
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "isolatedModules": true,
    "moduleDetection": "force",
    "noEmit": true,
    "strict": true
  },
  "include": ["vite.config.ts"]
}
```

**Step 4: Create `index.html`**

Write `amplifier-dashboard-attractor/frontend/index.html`:

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Attractor Pipeline Dashboard</title>
    <link rel="preconnect" href="https://fonts.googleapis.com" />
    <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin />
    <link
      href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&family=JetBrains+Mono:wght@400;500&display=swap"
      rel="stylesheet"
    />
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
```

**Step 5: Create the theme CSS**

Write `amplifier-dashboard-attractor/frontend/src/theme.css`:

```css
/*
 * Attractor Dashboard — Dark Theme
 *
 * Palette from the design doc (art director approved):
 * - Cool hues (teal, blue-gray) for normal operation
 * - Warm hues (coral, amber) for attention-requiring states
 * - Saturation 45-70% to prevent eye strain
 * - Triple-channel encoding: color + border + motion
 */

:root {
  /* Surface system */
  --surface-base: #0f1117;
  --surface-raised: #161922;
  --surface-overlay: #1c1f2b;
  --surface-hover: #252836;

  /* Text hierarchy (alpha-based white) */
  --text-primary: rgba(255, 255, 255, 0.92);
  --text-secondary: rgba(255, 255, 255, 0.64);
  --text-tertiary: rgba(255, 255, 255, 0.40);

  /* Border */
  --border-default: rgba(255, 255, 255, 0.08);
  --border-strong: rgba(255, 255, 255, 0.16);

  /* Pipeline state colors (from design doc HSL values) */
  --state-pending: hsl(220, 10%, 45%);
  --state-running: hsl(175, 65%, 55%);
  --state-success: hsl(145, 45%, 55%);
  --state-failed: hsl(0, 55%, 62%);
  --state-retrying: hsl(35, 70%, 60%);
  --state-skipped: hsl(220, 8%, 38%);

  /* State glow (for running node) */
  --glow-running: hsla(175, 65%, 55%, 0.3);
  --glow-failed: hsla(0, 55%, 62%, 0.2);

  /* Typography */
  --font-ui: "Inter", -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
  --font-mono: "JetBrains Mono", "Fira Code", "Cascadia Code", monospace;

  /* Spacing */
  --space-xs: 4px;
  --space-sm: 8px;
  --space-md: 16px;
  --space-lg: 24px;
  --space-xl: 32px;

  /* Sizes */
  --metrics-bar-height: 48px;
  --detail-panel-width: 360px;
  --node-min-width: 160px;
  --node-border-radius: 8px;

  /* Motion */
  --transition-fast: 150ms ease;
  --transition-normal: 300ms ease;
  --pulse-duration: 2s;
}

/* Reset */
*,
*::before,
*::after {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}

html {
  font-size: 14px;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

body {
  font-family: var(--font-ui);
  background: var(--surface-base);
  color: var(--text-primary);
  line-height: 1.5;
  min-height: 100vh;
}

code, pre, .mono {
  font-family: var(--font-mono);
}

/* Scrollbar styling */
::-webkit-scrollbar {
  width: 8px;
  height: 8px;
}

::-webkit-scrollbar-track {
  background: transparent;
}

::-webkit-scrollbar-thumb {
  background: var(--border-strong);
  border-radius: 4px;
}

/* Breathing pulse animation for running nodes */
@keyframes breathe {
  0%, 100% { box-shadow: 0 0 0 0 var(--glow-running); }
  50% { box-shadow: 0 0 12px 4px var(--glow-running); }
}

/* Shake animation for failed nodes */
@keyframes shake {
  0%, 100% { transform: translateX(0); }
  20% { transform: translateX(-3px); }
  40% { transform: translateX(3px); }
  60% { transform: translateX(-2px); }
  80% { transform: translateX(2px); }
}
```

**Step 6: Create TypeScript types**

Write `amplifier-dashboard-attractor/frontend/src/lib/types.ts`:

```typescript
/**
 * TypeScript types matching PipelineRunState.to_dict() output.
 *
 * Data model reference:
 *   amplifier-bundle-attractor/modules/hooks-pipeline-observability/
 *   amplifier_module_hooks_pipeline_observability/models.py
 */

export interface NodeInfo {
  id: string;
  label: string;
  shape: string;
  type: string;
  prompt: string;
}

export interface EdgeInfo {
  from_node: string;
  to_node: string;
  label: string;
  condition: string;
  weight: number;
}

export interface NodeRun {
  status: "running" | "success" | "fail" | "timeout" | "partial_success";
  attempt: number;
  started_at: string;
  completed_at: string | null;
  duration_ms: number;
  outcome_notes: string | null;
  llm_calls: number;
  tokens_in: number;
  tokens_out: number;
  tokens_cached: number;
}

export interface EdgeDecision {
  from_node: string;
  evaluated_edges: unknown[];
  selected_edge: EdgeInfo;
  reason: string;
}

export interface PipelineRunState {
  pipeline_id: string;
  dot_source: string;
  goal: string;
  nodes: Record<string, NodeInfo>;
  edges: EdgeInfo[];
  status: "pending" | "running" | "complete" | "failed";
  current_node: string | null;
  execution_path: string[];
  branches_taken: EdgeInfo[];
  node_runs: Record<string, NodeRun[]>;
  edge_decisions: EdgeDecision[];
  loop_iterations: Record<string, number>;
  goal_gate_checks: unknown[];
  parallel_branches: Record<string, unknown[]>;
  subgraph_runs: Record<string, unknown>;
  human_interactions: unknown[];
  supervisor_cycles: Record<string, unknown[]>;
  total_elapsed_ms: number;
  total_llm_calls: number;
  total_tokens_in: number;
  total_tokens_out: number;
  total_tokens_cached: number;
  total_tokens_reasoning: number;
  nodes_completed: number;
  nodes_total: number;
  timing: Record<string, number>;
  errors: Array<{ node: string; message: string; timestamp: string }>;
}

export interface PipelineFleetItem {
  context_id: number;
  pipeline_id: string;
  status: string;
  nodes_completed: number;
  nodes_total: number;
  total_elapsed_ms: number;
  total_tokens_in: number;
  total_tokens_out: number;
  goal: string;
  errors: Array<{ node: string; message: string; timestamp: string }>;
}

export interface NodeDetail {
  node_id: string;
  info: NodeInfo;
  runs: NodeRun[];
  edge_decisions: EdgeDecision[];
}

/**
 * Resolve the effective state for a node based on its runs.
 * Used by the graph renderer to determine CSS class.
 */
export type NodeState =
  | "pending"
  | "running"
  | "success"
  | "failed"
  | "retrying"
  | "skipped";

export function getNodeState(
  nodeId: string,
  state: PipelineRunState
): NodeState {
  const runs = state.node_runs[nodeId];
  if (!runs || runs.length === 0) return "pending";
  const lastRun = runs[runs.length - 1];
  if (lastRun.status === "running") return "running";
  if (lastRun.status === "success") return "success";
  if (lastRun.status === "fail" || lastRun.status === "timeout") {
    // If there were multiple attempts and the last one failed, it's "failed"
    // (not retrying — retrying would mean a new attempt is in progress)
    return "failed";
  }
  return "pending";
}
```

**Step 7: Create the API client**

Write `amplifier-dashboard-attractor/frontend/src/lib/api.ts`:

```typescript
/**
 * REST API client for the dashboard backend.
 *
 * During development, Vite proxies /api/* to the FastAPI backend.
 * In production, the FastAPI server serves the SPA and the API is same-origin.
 */

import type { PipelineFleetItem, PipelineRunState, NodeDetail } from "./types";

const API_BASE = "/api";

async function fetchJSON<T>(path: string): Promise<T> {
  const resp = await fetch(`${API_BASE}${path}`);
  if (!resp.ok) {
    throw new Error(`API error ${resp.status}: ${resp.statusText}`);
  }
  return resp.json();
}

export async function getPipelines(): Promise<PipelineFleetItem[]> {
  return fetchJSON<PipelineFleetItem[]>("/pipelines");
}

export async function getPipeline(
  contextId: number
): Promise<PipelineRunState> {
  return fetchJSON<PipelineRunState>(`/pipelines/${contextId}`);
}

export async function getNode(
  contextId: number,
  nodeId: string
): Promise<NodeDetail> {
  return fetchJSON<NodeDetail>(`/pipelines/${contextId}/nodes/${nodeId}`);
}
```

**Step 8: Create route stubs and App shell**

Write `amplifier-dashboard-attractor/frontend/src/vite-env.d.ts`:

```typescript
/// <reference types="vite/client" />
```

Write `amplifier-dashboard-attractor/frontend/src/views/FleetView.tsx`:

```tsx
/**
 * Fleet View — Level 1: scannable list of all pipeline instances.
 * Polls /api/pipelines every 5 seconds.
 */
export default function FleetView() {
  return (
    <div style={{ padding: "var(--space-lg)" }}>
      <h1>Pipeline Fleet</h1>
      <p style={{ color: "var(--text-secondary)" }}>
        Fleet table — implemented in Task 7.
      </p>
    </div>
  );
}
```

Write `amplifier-dashboard-attractor/frontend/src/views/PipelineView.tsx`:

```tsx
/**
 * Pipeline Detail View — Level 2: graph + detail panel.
 */
import { useParams } from "react-router-dom";

export default function PipelineView() {
  const { contextId } = useParams<{ contextId: string }>();
  return (
    <div style={{ padding: "var(--space-lg)" }}>
      <h1>Pipeline Detail</h1>
      <p style={{ color: "var(--text-secondary)" }}>
        Context ID: {contextId} — implemented in Task 9.
      </p>
    </div>
  );
}
```

Write `amplifier-dashboard-attractor/frontend/src/views/NodeView.tsx`:

```tsx
/**
 * Node Detail View — Level 3: node forensics (deferred to v1.1).
 */
import { useParams } from "react-router-dom";

export default function NodeView() {
  const { contextId, nodeId } = useParams<{
    contextId: string;
    nodeId: string;
  }>();
  return (
    <div style={{ padding: "var(--space-lg)" }}>
      <h1>Node Detail</h1>
      <p style={{ color: "var(--text-secondary)" }}>
        Context {contextId}, Node {nodeId} — deferred to v1.1.
      </p>
    </div>
  );
}
```

Write `amplifier-dashboard-attractor/frontend/src/App.tsx`:

```tsx
import { BrowserRouter, Routes, Route, Link, Navigate } from "react-router-dom";
import FleetView from "./views/FleetView";
import PipelineView from "./views/PipelineView";
import NodeView from "./views/NodeView";

export default function App() {
  return (
    <BrowserRouter>
      {/* Application shell — persistent nav */}
      <header
        style={{
          display: "flex",
          alignItems: "center",
          gap: "var(--space-md)",
          padding: "var(--space-sm) var(--space-lg)",
          borderBottom: "1px solid var(--border-default)",
          background: "var(--surface-raised)",
        }}
      >
        <Link
          to="/pipelines"
          style={{
            color: "var(--text-primary)",
            textDecoration: "none",
            fontWeight: 600,
            fontSize: "1rem",
          }}
        >
          Attractor Dashboard
        </Link>
      </header>

      <main>
        <Routes>
          <Route path="/" element={<Navigate to="/pipelines" replace />} />
          <Route path="/pipelines" element={<FleetView />} />
          <Route path="/pipelines/:contextId" element={<PipelineView />} />
          <Route
            path="/pipelines/:contextId/nodes/:nodeId"
            element={<NodeView />}
          />
        </Routes>
      </main>
    </BrowserRouter>
  );
}
```

Write `amplifier-dashboard-attractor/frontend/src/main.tsx`:

```tsx
import { StrictMode } from "react";
import { createRoot } from "react-dom/client";
import App from "./App";
import "./theme.css";

createRoot(document.getElementById("root")!).render(
  <StrictMode>
    <App />
  </StrictMode>
);
```

**Step 9: Install dependencies and verify the frontend builds**

```bash
cd /home/bkrabach/dev/attractor-next/amplifier-dashboard-attractor/frontend
npm install
npm run build
```

Expected: build succeeds, creates `frontend/dist/` directory with `index.html` and JS bundle.

**Step 10: Verify Vite dev server starts and routes work**

```bash
# Terminal 1: start backend
cd /home/bkrabach/dev/attractor-next/amplifier-dashboard-attractor
uv run dashboard --mock &

# Terminal 2: start frontend
cd /home/bkrabach/dev/attractor-next/amplifier-dashboard-attractor/frontend
npm run dev &

sleep 3

# Verify API proxy works through Vite
curl -s http://localhost:5173/api/health | python3 -m json.tool

kill %1 %2
```

Expected: health endpoint returns `{"status": "ok", "mock": true}` through the Vite proxy.

**Step 11: Commit**

```bash
cd /home/bkrabach/dev/attractor-next/amplifier-dashboard-attractor
git add -A && git commit -m "feat: React + Vite scaffold with dark theme, routing, and API client"
```

---

## Task 7: Frontend — Fleet View

Implement the fleet table showing all pipeline instances. Auto-polls the backend every 5 seconds. Each row shows status indicator, pipeline name, progress, elapsed time, token count. Click navigates to pipeline detail.

**Files:**
- Modify: `amplifier-dashboard-attractor/frontend/src/views/FleetView.tsx`
- Create: `amplifier-dashboard-attractor/frontend/src/hooks/usePolling.ts`

**Step 1: Create the polling hook**

Write `amplifier-dashboard-attractor/frontend/src/hooks/usePolling.ts`:

```typescript
/**
 * usePolling — fetches data at a regular interval.
 *
 * Returns { data, error, loading }.
 * Stops polling when the component unmounts.
 */

import { useState, useEffect, useRef, useCallback } from "react";

export function usePolling<T>(
  fetcher: () => Promise<T>,
  intervalMs: number = 5000
): { data: T | null; error: string | null; loading: boolean } {
  const [data, setData] = useState<T | null>(null);
  const [error, setError] = useState<string | null>(null);
  const [loading, setLoading] = useState(true);
  const timerRef = useRef<ReturnType<typeof setInterval> | null>(null);

  const doFetch = useCallback(async () => {
    try {
      const result = await fetcher();
      setData(result);
      setError(null);
    } catch (err) {
      setError(err instanceof Error ? err.message : "Unknown error");
    } finally {
      setLoading(false);
    }
  }, [fetcher]);

  useEffect(() => {
    doFetch();
    timerRef.current = setInterval(doFetch, intervalMs);
    return () => {
      if (timerRef.current) clearInterval(timerRef.current);
    };
  }, [doFetch, intervalMs]);

  return { data, error, loading };
}
```

**Step 2: Implement the fleet view**

Replace `amplifier-dashboard-attractor/frontend/src/views/FleetView.tsx`:

```tsx
/**
 * Fleet View — Level 1: scannable table of all pipeline instances.
 *
 * Polls GET /api/pipelines every 5 seconds.
 * Click a row to navigate to pipeline detail.
 */

import { useCallback } from "react";
import { useNavigate } from "react-router-dom";
import { getPipelines } from "../lib/api";
import { usePolling } from "../hooks/usePolling";
import type { PipelineFleetItem } from "../lib/types";

/** Format milliseconds to human-readable duration. */
function formatDuration(ms: number): string {
  if (ms < 1000) return `${ms}ms`;
  const secs = ms / 1000;
  if (secs < 60) return `${secs.toFixed(1)}s`;
  const mins = Math.floor(secs / 60);
  const remainSecs = Math.floor(secs % 60);
  return `${mins}m ${remainSecs}s`;
}

/** Format token count to compact form. */
function formatTokens(n: number): string {
  if (n < 1000) return String(n);
  return `${(n / 1000).toFixed(1)}k`;
}

const STATUS_COLORS: Record<string, string> = {
  running: "var(--state-running)",
  complete: "var(--state-success)",
  failed: "var(--state-failed)",
  pending: "var(--state-pending)",
};

export default function FleetView() {
  const fetcher = useCallback(() => getPipelines(), []);
  const { data: pipelines, error, loading } = usePolling<PipelineFleetItem[]>(fetcher);
  const navigate = useNavigate();

  if (loading && !pipelines) {
    return (
      <div style={{ padding: "var(--space-lg)", color: "var(--text-secondary)" }}>
        Loading pipelines...
      </div>
    );
  }

  if (error) {
    return (
      <div style={{ padding: "var(--space-lg)", color: "var(--state-failed)" }}>
        Error: {error}
      </div>
    );
  }

  return (
    <div style={{ padding: "var(--space-lg)" }}>
      <h1 style={{ marginBottom: "var(--space-lg)", fontWeight: 600 }}>
        Pipeline Fleet
      </h1>

      <table
        style={{
          width: "100%",
          borderCollapse: "collapse",
          fontFamily: "var(--font-ui)",
        }}
      >
        <thead>
          <tr
            style={{
              borderBottom: "1px solid var(--border-strong)",
              color: "var(--text-secondary)",
              fontSize: "0.85rem",
              textAlign: "left",
            }}
          >
            <th style={{ padding: "var(--space-sm) var(--space-md)" }}>Status</th>
            <th style={{ padding: "var(--space-sm) var(--space-md)" }}>Pipeline</th>
            <th style={{ padding: "var(--space-sm) var(--space-md)" }}>Progress</th>
            <th style={{ padding: "var(--space-sm) var(--space-md)" }}>Elapsed</th>
            <th style={{ padding: "var(--space-sm) var(--space-md)" }}>Tokens</th>
            <th style={{ padding: "var(--space-sm) var(--space-md)" }}>Goal</th>
          </tr>
        </thead>
        <tbody>
          {pipelines?.map((p) => (
            <tr
              key={p.context_id}
              onClick={() => navigate(`/pipelines/${p.context_id}`)}
              style={{
                cursor: "pointer",
                borderBottom: "1px solid var(--border-default)",
                transition: "background var(--transition-fast)",
              }}
              onMouseEnter={(e) =>
                (e.currentTarget.style.background = "var(--surface-hover)")
              }
              onMouseLeave={(e) =>
                (e.currentTarget.style.background = "transparent")
              }
            >
              <td style={{ padding: "var(--space-sm) var(--space-md)" }}>
                <span
                  style={{
                    display: "inline-block",
                    width: 10,
                    height: 10,
                    borderRadius: "50%",
                    background: STATUS_COLORS[p.status] ?? "var(--state-pending)",
                    marginRight: "var(--space-sm)",
                  }}
                />
                <span style={{ fontSize: "0.85rem" }}>{p.status}</span>
              </td>
              <td
                style={{
                  padding: "var(--space-sm) var(--space-md)",
                  fontFamily: "var(--font-mono)",
                  fontSize: "0.85rem",
                }}
              >
                {p.pipeline_id}
              </td>
              <td
                style={{
                  padding: "var(--space-sm) var(--space-md)",
                  fontFamily: "var(--font-mono)",
                  fontSize: "0.85rem",
                }}
              >
                {p.nodes_completed}/{p.nodes_total}
              </td>
              <td
                style={{
                  padding: "var(--space-sm) var(--space-md)",
                  fontFamily: "var(--font-mono)",
                  fontSize: "0.85rem",
                }}
              >
                {formatDuration(p.total_elapsed_ms)}
              </td>
              <td
                style={{
                  padding: "var(--space-sm) var(--space-md)",
                  fontFamily: "var(--font-mono)",
                  fontSize: "0.85rem",
                }}
              >
                {formatTokens(p.total_tokens_in + p.total_tokens_out)}
              </td>
              <td
                style={{
                  padding: "var(--space-sm) var(--space-md)",
                  color: "var(--text-secondary)",
                  fontSize: "0.85rem",
                  maxWidth: 300,
                  overflow: "hidden",
                  textOverflow: "ellipsis",
                  whiteSpace: "nowrap",
                }}
              >
                {p.goal}
              </td>
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  );
}
```

**Step 3: Verify it renders**

Start the backend and frontend:

```bash
cd /home/bkrabach/dev/attractor-next/amplifier-dashboard-attractor
uv run dashboard --mock &
cd frontend && npm run dev &
sleep 3
```

Open `http://localhost:5173/pipelines` in a browser. You should see a dark-themed table with 3 pipeline rows (running, complete, failed) with colored status indicators, pipeline IDs in monospace, progress fractions, and durations.

```bash
kill %1 %2
```

**Step 4: Verify the build still works**

```bash
cd /home/bkrabach/dev/attractor-next/amplifier-dashboard-attractor/frontend
npm run build
```

Expected: build succeeds.

**Step 5: Commit**

```bash
cd /home/bkrabach/dev/attractor-next/amplifier-dashboard-attractor
git add -A && git commit -m "feat: fleet view with polling table and status colors"
```

---

## Task 8: Frontend — Graph Rendering Library

Build the DOT → ELK layout → React Flow rendering pipeline. This is the core visualization engine. Parse DOT source with ts-graphviz, compute positions with ELK.js (layered algorithm, left-to-right), and convert to React Flow nodes/edges.

**Files:**
- Create: `amplifier-dashboard-attractor/frontend/src/lib/dotLayout.ts`
- Create: `amplifier-dashboard-attractor/frontend/src/components/PipelineNode.tsx`
- Create: `amplifier-dashboard-attractor/frontend/src/components/PipelineEdge.tsx`

**Step 1: Create the DOT → ELK → React Flow layout engine**

Write `amplifier-dashboard-attractor/frontend/src/lib/dotLayout.ts`:

```typescript
/**
 * DOT → ELK → React Flow layout pipeline.
 *
 * 1. Parse DOT source with ts-graphviz to extract topology (nodes, edges)
 * 2. Layout with ELK.js using the "layered" algorithm (designed for DAGs)
 * 3. Convert ELK positions to React Flow node/edge definitions
 *
 * The graph is laid out ONCE. Subsequent state changes only mutate CSS classes
 * on existing nodes — no layout recalculation during execution.
 */

import ELK, { type ElkNode, type ElkExtendedEdge } from "elkjs/lib/elk.bundled.js";
import { fromDot } from "ts-graphviz";
import type { Node, Edge } from "@xyflow/react";
import type { NodeInfo, PipelineRunState } from "./types";
import { getNodeState, type NodeState } from "./types";

const elk = new ELK();

/** Node dimensions (from design doc: 160px min-width) */
const NODE_WIDTH = 180;
const NODE_HEIGHT = 64;

export interface LayoutResult {
  nodes: Node<PipelineNodeData>[];
  edges: Edge[];
}

export interface PipelineNodeData {
  label: string;
  nodeId: string;
  nodeInfo: NodeInfo;
  state: NodeState;
  durationMs: number;
  tokensIn: number;
  tokensOut: number;
  [key: string]: unknown;
}

/**
 * Parse DOT source and lay out with ELK, returning React Flow nodes and edges.
 */
export async function layoutPipeline(
  pipelineState: PipelineRunState
): Promise<LayoutResult> {
  // Step 1: Parse DOT source
  const dotGraph = fromDot(pipelineState.dot_source);

  // Extract node IDs and edge definitions from the parsed DOT
  const dotNodes: string[] = [];
  const dotEdges: Array<{ source: string; target: string; label: string }> = [];

  for (const node of dotGraph.nodes) {
    dotNodes.push(node.id);
  }

  for (const edge of dotGraph.edges) {
    const targets = edge.targets;
    // ts-graphviz edges have targets array: [from, to]
    if (targets.length >= 2) {
      const fromId =
        typeof targets[0] === "string"
          ? targets[0]
          : "id" in targets[0]
            ? targets[0].id
            : String(targets[0]);
      const toId =
        typeof targets[1] === "string"
          ? targets[1]
          : "id" in targets[1]
            ? targets[1].id
            : String(targets[1]);
      dotEdges.push({
        source: fromId,
        target: toId,
        label: (edge.attributes.get("label") as string) ?? "",
      });
    }
  }

  // Step 2: Build ELK graph
  const elkGraph: ElkNode = {
    id: "root",
    layoutOptions: {
      "elk.algorithm": "layered",
      "elk.direction": "RIGHT",
      "elk.spacing.nodeNode": "40",
      "elk.layered.spacing.nodeNodeBetweenLayers": "60",
      "elk.padding": "[top=20,left=20,bottom=20,right=20]",
    },
    children: dotNodes.map((id) => ({
      id,
      width: NODE_WIDTH,
      height: NODE_HEIGHT,
    })),
    edges: dotEdges.map((e, i) => ({
      id: `e${i}-${e.source}-${e.target}`,
      sources: [e.source],
      targets: [e.target],
    })) as ElkExtendedEdge[],
  };

  const layout = await elk.layout(elkGraph);

  // Step 3: Convert to React Flow format
  const nodes: Node<PipelineNodeData>[] = (layout.children ?? []).map(
    (elkNode) => {
      const nodeId = elkNode.id;
      const nodeInfo = pipelineState.nodes[nodeId] ?? {
        id: nodeId,
        label: nodeId,
        shape: "box",
        type: "",
        prompt: "",
      };
      const state = getNodeState(nodeId, pipelineState);
      const runs = pipelineState.node_runs[nodeId] ?? [];
      const lastRun = runs.length > 0 ? runs[runs.length - 1] : null;

      return {
        id: nodeId,
        type: "pipelineNode",
        position: { x: elkNode.x ?? 0, y: elkNode.y ?? 0 },
        data: {
          label: nodeInfo.label || nodeId,
          nodeId,
          nodeInfo,
          state,
          durationMs: lastRun?.duration_ms ?? 0,
          tokensIn: lastRun?.tokens_in ?? 0,
          tokensOut: lastRun?.tokens_out ?? 0,
        },
      };
    }
  );

  const edges: Edge[] = dotEdges.map((e, i) => {
    // Determine edge visual state based on execution path
    const sourceIdx = pipelineState.execution_path.indexOf(e.source);
    const targetIdx = pipelineState.execution_path.indexOf(e.target);
    const isTraversed = sourceIdx >= 0 && targetIdx >= 0 && targetIdx > sourceIdx;

    return {
      id: `e${i}-${e.source}-${e.target}`,
      source: e.source,
      target: e.target,
      type: "pipelineEdge",
      label: e.label || undefined,
      data: { traversed: isTraversed },
    };
  });

  return { nodes, edges };
}
```

**Step 2: Create the custom `PipelineNode` component**

Write `amplifier-dashboard-attractor/frontend/src/components/PipelineNode.tsx`:

```tsx
/**
 * Custom React Flow node for pipeline graph visualization.
 *
 * Displays: node label, duration, and state via CSS class.
 * State colors, borders, and animations are driven by CSS variables.
 *
 * Design reference (from design doc):
 *   +----------------------+
 *   |  [icon] Node Name    |  <- Inter 13px/500
 *   |  model-name . 2.3s   |  <- JetBrains Mono 11px/400
 *   +----------------------+
 */

import { Handle, Position, type NodeProps } from "@xyflow/react";
import type { PipelineNodeData } from "../lib/dotLayout";
import type { NodeState } from "../lib/types";

const STATE_STYLES: Record<
  NodeState,
  { bg: string; border: string; borderStyle: string; animation?: string }
> = {
  pending: {
    bg: "var(--surface-raised)",
    border: "var(--state-pending)",
    borderStyle: "dashed",
  },
  running: {
    bg: "var(--surface-raised)",
    border: "var(--state-running)",
    borderStyle: "solid",
    animation: "breathe var(--pulse-duration) ease-in-out infinite",
  },
  success: {
    bg: "var(--surface-raised)",
    border: "var(--state-success)",
    borderStyle: "solid",
  },
  failed: {
    bg: "var(--surface-raised)",
    border: "var(--state-failed)",
    borderStyle: "solid",
  },
  retrying: {
    bg: "var(--surface-raised)",
    border: "var(--state-retrying)",
    borderStyle: "dashed",
  },
  skipped: {
    bg: "var(--surface-raised)",
    border: "var(--state-skipped)",
    borderStyle: "dotted",
  },
};

function formatDuration(ms: number): string {
  if (ms === 0) return "";
  if (ms < 1000) return `${ms}ms`;
  return `${(ms / 1000).toFixed(1)}s`;
}

export default function PipelineNode({ data }: NodeProps) {
  const nodeData = data as unknown as PipelineNodeData;
  const style = STATE_STYLES[nodeData.state] ?? STATE_STYLES.pending;

  return (
    <>
      <Handle type="target" position={Position.Left} style={{ visibility: "hidden" }} />
      <div
        style={{
          minWidth: "var(--node-min-width)",
          background: style.bg,
          border: `2px ${style.borderStyle} ${style.border}`,
          borderRadius: "var(--node-border-radius)",
          padding: "var(--space-sm) var(--space-md)",
          animation: style.animation,
          transition: "border-color var(--transition-normal), background var(--transition-normal)",
        }}
      >
        <div
          style={{
            fontFamily: "var(--font-ui)",
            fontSize: "13px",
            fontWeight: 500,
            color: "var(--text-primary)",
            marginBottom: "2px",
          }}
        >
          {nodeData.label}
        </div>
        <div
          style={{
            fontFamily: "var(--font-mono)",
            fontSize: "11px",
            fontWeight: 400,
            color: "var(--text-secondary)",
            display: "flex",
            gap: "var(--space-sm)",
          }}
        >
          {nodeData.nodeInfo.type && <span>{nodeData.nodeInfo.type}</span>}
          {nodeData.durationMs > 0 && (
            <>
              <span style={{ color: "var(--text-tertiary)" }}>·</span>
              <span>{formatDuration(nodeData.durationMs)}</span>
            </>
          )}
          {nodeData.state === "running" && (
            <span style={{ color: "var(--state-running)" }}>running...</span>
          )}
        </div>
      </div>
      <Handle type="source" position={Position.Right} style={{ visibility: "hidden" }} />
    </>
  );
}
```

**Step 3: Create the custom `PipelineEdge` component**

Write `amplifier-dashboard-attractor/frontend/src/components/PipelineEdge.tsx`:

```tsx
/**
 * Custom React Flow edge for pipeline graph visualization.
 *
 * Edge states (from design doc):
 *   Not yet reached: thin, dashed, dim gray
 *   Traversed:       solid, slightly brighter
 */

import {
  BaseEdge,
  getSmoothStepPath,
  type EdgeProps,
} from "@xyflow/react";

export default function PipelineEdge({
  id,
  sourceX,
  sourceY,
  targetX,
  targetY,
  sourcePosition,
  targetPosition,
  label,
  data,
}: EdgeProps) {
  const traversed = (data as Record<string, unknown>)?.traversed === true;

  const [edgePath, labelX, labelY] = getSmoothStepPath({
    sourceX,
    sourceY,
    targetX,
    targetY,
    sourcePosition,
    targetPosition,
  });

  return (
    <>
      <BaseEdge
        id={id}
        path={edgePath}
        style={{
          stroke: traversed ? "var(--text-secondary)" : "var(--text-tertiary)",
          strokeWidth: traversed ? 2 : 1,
          strokeDasharray: traversed ? "none" : "6 4",
          transition: "stroke var(--transition-normal), stroke-width var(--transition-normal)",
        }}
      />
      {label && (
        <text
          x={labelX}
          y={labelY}
          style={{
            fontSize: "10px",
            fill: "var(--text-tertiary)",
            fontFamily: "var(--font-mono)",
          }}
          textAnchor="middle"
          dominantBaseline="central"
        >
          {String(label)}
        </text>
      )}
    </>
  );
}
```

**Step 4: Verify the build compiles**

```bash
cd /home/bkrabach/dev/attractor-next/amplifier-dashboard-attractor/frontend
npm run build
```

Expected: build succeeds (these components are created but not yet wired into a view — that's Task 9).

**Step 5: Commit**

```bash
cd /home/bkrabach/dev/attractor-next/amplifier-dashboard-attractor
git add -A && git commit -m "feat: DOT layout engine + PipelineNode/PipelineEdge components"
```

---

## Task 9: Frontend — Pipeline Detail View

Wire the graph rendering into the pipeline detail view. Fetch pipeline state, layout with ELK, render with React Flow. Add a metrics bar at top and a detail panel on the right.

**Files:**
- Modify: `amplifier-dashboard-attractor/frontend/src/views/PipelineView.tsx`
- Create: `amplifier-dashboard-attractor/frontend/src/components/MetricsBar.tsx`
- Create: `amplifier-dashboard-attractor/frontend/src/components/DetailPanel.tsx`

**Step 1: Create the MetricsBar component**

Write `amplifier-dashboard-attractor/frontend/src/components/MetricsBar.tsx`:

```tsx
/**
 * Aggregate metrics bar — 48px height, sits at top of pipeline detail view.
 * Shows: status, progress, elapsed, tokens, LLM calls, error count.
 */

import type { PipelineRunState } from "../lib/types";

function formatDuration(ms: number): string {
  if (ms < 1000) return `${ms}ms`;
  const secs = ms / 1000;
  if (secs < 60) return `${secs.toFixed(1)}s`;
  const mins = Math.floor(secs / 60);
  const remainSecs = Math.floor(secs % 60);
  return `${mins}m ${remainSecs}s`;
}

function formatTokens(n: number): string {
  if (n < 1000) return String(n);
  return `${(n / 1000).toFixed(1)}k`;
}

const STATUS_COLORS: Record<string, string> = {
  running: "var(--state-running)",
  complete: "var(--state-success)",
  failed: "var(--state-failed)",
  pending: "var(--state-pending)",
};

interface MetricsBarProps {
  state: PipelineRunState;
}

export default function MetricsBar({ state }: MetricsBarProps) {
  return (
    <div
      style={{
        height: "var(--metrics-bar-height)",
        display: "flex",
        alignItems: "center",
        gap: "var(--space-xl)",
        padding: "0 var(--space-lg)",
        borderBottom: "1px solid var(--border-default)",
        background: "var(--surface-raised)",
        fontSize: "0.85rem",
      }}
    >
      {/* Status badge */}
      <div style={{ display: "flex", alignItems: "center", gap: "var(--space-sm)" }}>
        <span
          style={{
            width: 8,
            height: 8,
            borderRadius: "50%",
            background: STATUS_COLORS[state.status] ?? "var(--state-pending)",
            display: "inline-block",
          }}
        />
        <span style={{ fontWeight: 600 }}>{state.status}</span>
      </div>

      {/* Metrics */}
      <Metric label="Progress" value={`${state.nodes_completed}/${state.nodes_total}`} />
      <Metric label="Elapsed" value={formatDuration(state.total_elapsed_ms)} />
      <Metric label="Tokens" value={formatTokens(state.total_tokens_in + state.total_tokens_out)} />
      <Metric label="LLM Calls" value={String(state.total_llm_calls)} />
      {state.errors.length > 0 && (
        <Metric label="Errors" value={String(state.errors.length)} color="var(--state-failed)" />
      )}

      {/* Goal text */}
      <div
        style={{
          marginLeft: "auto",
          color: "var(--text-secondary)",
          maxWidth: 400,
          overflow: "hidden",
          textOverflow: "ellipsis",
          whiteSpace: "nowrap",
        }}
      >
        {state.goal}
      </div>
    </div>
  );
}

function Metric({
  label,
  value,
  color,
}: {
  label: string;
  value: string;
  color?: string;
}) {
  return (
    <div style={{ display: "flex", flexDirection: "column", alignItems: "center" }}>
      <span
        style={{
          fontFamily: "var(--font-mono)",
          fontSize: "0.9rem",
          fontWeight: 500,
          color: color ?? "var(--text-primary)",
        }}
      >
        {value}
      </span>
      <span style={{ fontSize: "0.7rem", color: "var(--text-tertiary)" }}>
        {label}
      </span>
    </div>
  );
}
```

**Step 2: Create the DetailPanel component**

Write `amplifier-dashboard-attractor/frontend/src/components/DetailPanel.tsx`:

```tsx
/**
 * Detail panel — right sidebar (360px) showing selected node info and run history.
 * Becomes an overlay drawer below 1280px viewport (deferred to v1.1).
 */

import type { NodeInfo, NodeRun } from "../lib/types";

interface DetailPanelProps {
  nodeId: string | null;
  nodeInfo: NodeInfo | null;
  runs: NodeRun[];
}

function formatDuration(ms: number): string {
  if (ms === 0) return "-";
  if (ms < 1000) return `${ms}ms`;
  return `${(ms / 1000).toFixed(1)}s`;
}

export default function DetailPanel({ nodeId, nodeInfo, runs }: DetailPanelProps) {
  if (!nodeId || !nodeInfo) {
    return (
      <div
        style={{
          width: "var(--detail-panel-width)",
          borderLeft: "1px solid var(--border-default)",
          padding: "var(--space-lg)",
          color: "var(--text-tertiary)",
          display: "flex",
          alignItems: "center",
          justifyContent: "center",
        }}
      >
        Click a node to view details
      </div>
    );
  }

  return (
    <div
      style={{
        width: "var(--detail-panel-width)",
        borderLeft: "1px solid var(--border-default)",
        padding: "var(--space-lg)",
        overflow: "auto",
      }}
    >
      {/* Node identity */}
      <h2
        style={{
          fontSize: "1rem",
          fontWeight: 600,
          marginBottom: "var(--space-xs)",
        }}
      >
        {nodeInfo.label || nodeId}
      </h2>
      <div
        style={{
          fontFamily: "var(--font-mono)",
          fontSize: "0.8rem",
          color: "var(--text-secondary)",
          marginBottom: "var(--space-lg)",
          display: "flex",
          gap: "var(--space-sm)",
        }}
      >
        <span>{nodeId}</span>
        {nodeInfo.type && (
          <>
            <span style={{ color: "var(--text-tertiary)" }}>·</span>
            <span>{nodeInfo.type}</span>
          </>
        )}
        {nodeInfo.shape && (
          <>
            <span style={{ color: "var(--text-tertiary)" }}>·</span>
            <span>{nodeInfo.shape}</span>
          </>
        )}
      </div>

      {/* Run history */}
      <h3
        style={{
          fontSize: "0.85rem",
          fontWeight: 600,
          color: "var(--text-secondary)",
          marginBottom: "var(--space-sm)",
        }}
      >
        Run History ({runs.length} attempt{runs.length !== 1 ? "s" : ""})
      </h3>

      {runs.map((run, i) => (
        <div
          key={i}
          style={{
            background: "var(--surface-overlay)",
            borderRadius: 6,
            padding: "var(--space-sm) var(--space-md)",
            marginBottom: "var(--space-sm)",
            fontSize: "0.8rem",
          }}
        >
          <div
            style={{
              display: "flex",
              justifyContent: "space-between",
              marginBottom: "var(--space-xs)",
            }}
          >
            <span style={{ fontWeight: 500 }}>Attempt {run.attempt}</span>
            <span
              style={{
                fontFamily: "var(--font-mono)",
                color:
                  run.status === "success"
                    ? "var(--state-success)"
                    : run.status === "running"
                      ? "var(--state-running)"
                      : "var(--state-failed)",
              }}
            >
              {run.status}
            </span>
          </div>
          <div
            style={{
              fontFamily: "var(--font-mono)",
              color: "var(--text-secondary)",
              display: "flex",
              gap: "var(--space-md)",
              flexWrap: "wrap",
            }}
          >
            <span>{formatDuration(run.duration_ms)}</span>
            {run.llm_calls > 0 && <span>{run.llm_calls} calls</span>}
            {run.tokens_in > 0 && <span>{run.tokens_in} in</span>}
            {run.tokens_out > 0 && <span>{run.tokens_out} out</span>}
          </div>
          {run.outcome_notes && (
            <div style={{ color: "var(--text-tertiary)", marginTop: "var(--space-xs)" }}>
              {run.outcome_notes}
            </div>
          )}
        </div>
      ))}
    </div>
  );
}
```

**Step 3: Implement the PipelineView**

Replace `amplifier-dashboard-attractor/frontend/src/views/PipelineView.tsx`:

```tsx
/**
 * Pipeline Detail View — Level 2: graph + detail panel.
 *
 * Layout: CSS Grid
 *   Top: MetricsBar (48px)
 *   Main: Graph (flexible) + DetailPanel (360px)
 *
 * Fetches pipeline state, lays out with ELK, renders with React Flow.
 * Click a node to populate the detail panel.
 */

import { useParams, Link } from "react-router-dom";
import { useState, useEffect, useCallback, useMemo } from "react";
import {
  ReactFlow,
  Background,
  BackgroundVariant,
  type Node,
  type Edge,
  type NodeTypes,
  type EdgeTypes,
} from "@xyflow/react";
import "@xyflow/react/dist/style.css";

import { getPipeline } from "../lib/api";
import { layoutPipeline } from "../lib/dotLayout";
import type { PipelineNodeData } from "../lib/dotLayout";
import type { PipelineRunState, NodeInfo, NodeRun } from "../lib/types";
import PipelineNode from "../components/PipelineNode";
import PipelineEdge from "../components/PipelineEdge";
import MetricsBar from "../components/MetricsBar";
import DetailPanel from "../components/DetailPanel";

const nodeTypes: NodeTypes = { pipelineNode: PipelineNode };
const edgeTypes: EdgeTypes = { pipelineEdge: PipelineEdge };

export default function PipelineView() {
  const { contextId } = useParams<{ contextId: string }>();
  const [state, setState] = useState<PipelineRunState | null>(null);
  const [nodes, setNodes] = useState<Node<PipelineNodeData>[]>([]);
  const [edges, setEdges] = useState<Edge[]>([]);
  const [error, setError] = useState<string | null>(null);
  const [loading, setLoading] = useState(true);

  // Selected node for detail panel
  const [selectedNodeId, setSelectedNodeId] = useState<string | null>(null);

  // Fetch pipeline state and compute layout
  useEffect(() => {
    if (!contextId) return;

    async function load() {
      try {
        const pipelineState = await getPipeline(Number(contextId));
        setState(pipelineState);

        const layout = await layoutPipeline(pipelineState);
        setNodes(layout.nodes);
        setEdges(layout.edges);
        setError(null);
      } catch (err) {
        setError(err instanceof Error ? err.message : "Failed to load pipeline");
      } finally {
        setLoading(false);
      }
    }

    load();
  }, [contextId]);

  // Handle node click
  const onNodeClick = useCallback(
    (_event: React.MouseEvent, node: Node) => {
      setSelectedNodeId(node.id);
    },
    []
  );

  // Derive selected node info
  const selectedNodeInfo: NodeInfo | null = useMemo(() => {
    if (!selectedNodeId || !state) return null;
    return state.nodes[selectedNodeId] ?? null;
  }, [selectedNodeId, state]);

  const selectedNodeRuns: NodeRun[] = useMemo(() => {
    if (!selectedNodeId || !state) return [];
    return state.node_runs[selectedNodeId] ?? [];
  }, [selectedNodeId, state]);

  if (loading) {
    return (
      <div style={{ padding: "var(--space-lg)", color: "var(--text-secondary)" }}>
        Loading pipeline...
      </div>
    );
  }

  if (error || !state) {
    return (
      <div style={{ padding: "var(--space-lg)", color: "var(--state-failed)" }}>
        {error ?? "Pipeline not found"}
      </div>
    );
  }

  return (
    <div
      style={{
        display: "grid",
        gridTemplateRows: "auto 1fr",
        height: "calc(100vh - 41px)", /* Subtract header height */
      }}
    >
      {/* Breadcrumb + MetricsBar */}
      <div>
        <div
          style={{
            padding: "var(--space-xs) var(--space-lg)",
            fontSize: "0.8rem",
            color: "var(--text-tertiary)",
          }}
        >
          <Link
            to="/pipelines"
            style={{ color: "var(--text-secondary)", textDecoration: "none" }}
          >
            Fleet
          </Link>
          {" / "}
          <span style={{ fontFamily: "var(--font-mono)" }}>
            {state.pipeline_id}
          </span>
        </div>
        <MetricsBar state={state} />
      </div>

      {/* Graph + Detail Panel */}
      <div style={{ display: "grid", gridTemplateColumns: "1fr var(--detail-panel-width)" }}>
        <div style={{ position: "relative" }}>
          <ReactFlow
            nodes={nodes}
            edges={edges}
            nodeTypes={nodeTypes}
            edgeTypes={edgeTypes}
            onNodeClick={onNodeClick}
            fitView
            fitViewOptions={{ padding: 0.2 }}
            proOptions={{ hideAttribution: true }}
            style={{ background: "var(--surface-base)" }}
          >
            <Background variant={BackgroundVariant.Dots} color="var(--border-default)" gap={20} />
          </ReactFlow>
        </div>

        <DetailPanel
          nodeId={selectedNodeId}
          nodeInfo={selectedNodeInfo}
          runs={selectedNodeRuns}
        />
      </div>
    </div>
  );
}
```

**Step 4: Verify it renders end-to-end**

```bash
# Start backend in mock mode
cd /home/bkrabach/dev/attractor-next/amplifier-dashboard-attractor
uv run dashboard --mock &

# Start frontend
cd frontend && npm run dev &
sleep 3
```

Open `http://localhost:5173/pipelines` in a browser:
1. Fleet view shows 3 pipeline rows.
2. Click "research-summarize-001" row → navigates to `/pipelines/1001`.
3. Pipeline detail shows: breadcrumb, metrics bar (running, 3/6, 29.5s), graph with 6 nodes laid out left-to-right.
4. Node colors: Start/Gather/Analyze = green (success), Synthesize = teal with pulse (running), Review/Publish = gray dashed (pending).
5. Click a node → detail panel populates with run history.

```bash
kill %1 %2
```

**Step 5: Verify the production build works**

```bash
cd /home/bkrabach/dev/attractor-next/amplifier-dashboard-attractor/frontend
npm run build
```

Expected: build succeeds.

**Step 6: Commit**

```bash
cd /home/bkrabach/dev/attractor-next/amplifier-dashboard-attractor
git add -A && git commit -m "feat: pipeline detail view with graph rendering and detail panel"
```

---

## Summary

| Task | Description | Files Created/Modified |
|------|-------------|----------------------|
| 1 | Scaffold repo | 7 files + git init + submodule |
| 2 | FastAPI server skeleton | `server.py`, `test_server.py` |
| 3 | Mock data | `mock_data.py`, `test_mock_data.py` |
| 4 | CXDB client | `cxdb_client.py`, `test_cxdb_client.py` |
| 5 | REST endpoints | `routes/pipelines.py`, `test_routes.py`, update `server.py` |
| 6 | Frontend scaffold | 14 files (package.json, vite config, theme, routes, types, api) |
| 7 | Fleet view | `FleetView.tsx`, `usePolling.ts` |
| 8 | Graph rendering | `dotLayout.ts`, `PipelineNode.tsx`, `PipelineEdge.tsx` |
| 9 | Pipeline detail | `PipelineView.tsx`, `MetricsBar.tsx`, `DetailPanel.tsx` |

**Total:** 9 tasks, ~30 files. Each task is independently testable and committable.

### v1.1 Deferred Items

- WebSocket real-time updates (`/ws/pipelines/:id`)
- Running node animations (marching ants on edges)
- Node drill-down view (Level 3 forensics)
- Responsive overlay drawer for detail panel below 1280px
- Minimap for large graphs
- Graph thumbnails in fleet view
- Toast notifications for pipeline completion/failure
