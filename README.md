# Hermes Hardware Horizon Backend

Async backend for autonomous hardware pipeline orchestration — PySpice/Ngspice (Simulation), KiCad/SKiSDL (ECAD), and Build123d (MCAD) → SSE dashboard.

## Quick Start

```bash
cd /home/rescueuser/workshop-agent
uv venv
uv pip install -e .
uvicorn app.main:app --host 0.0.0.0 --port 8000
```

Then open http://localhost:8000 in your browser.

## Architecture

```
app/
├── main.py        # FastAPI server + CORS + SSE /api/stream + static routes
├── broker.py      # asyncio.Queue-based event broker (pub/sub per SSE client)
├── pipeline.py    # Background worker queue (sequential), 4-stage pipeline
├── parsers.py     # PySpice / KiCad / Build123d data parsers → JSON
└── __init__.py

frontend/
└── index.html     # Stitch-built dashboard (Google Material 3, Tailwind)
```

## API Endpoints

| Method | Path           | Description                          |
|--------|----------------|--------------------------------------|
| GET    | `/`            | Serves frontend dashboard HTML       |
| GET    | `/api/health`  | Health check (connections, queue)    |
| POST   | `/api/pipeline`| Trigger a new build pipeline         |
| GET    | `/api/stream`  | SSE endpoint for real-time events    |
| GET    | `/static/meshes/<uuid>.glb` | GLB mesh files from Build123d |

## SSE Event Schema

All events use standard SSE `event:` + `data:` framing. The browser's `EventSource` API parses them natively.

### `sidebar_update` — Stage status
```json
{"stage": 1, "task": "planning", "status": "active"}
```

### `log_stream` — Terminal output line
```json
{"text": "  Project: sensor_node_v1"}
```

### `viewport_render` — Viewport visualization
```json
{"type": "simulation_chart", "payload": {"type": "transient", "data": [{"x": 0.0, "y": 5.0}, ...]}}
{"type": "pcb_wireframe",  "payload": {"components": [...], "traces": [...], "vias": [...]}}
{"type": "enclosure_3d",   "payload": {"url": "/static/meshes/uuid.glb", "enclosure_name": "enclosure"}}
```

### `global_status` — Footer state
```json
{"state": "running", "msg": "Build #abcd started"}
```

## Pipeline Stages

1. **PLANNING** — Parse requirements, define specs/constraints
2. **SIMULATION** — PySpice/Ngspice transient + AC analysis → numpy vectors → coordinate sets
3. **PCB_GENERATOR** — KiCad/SKiSDL layout → component X/Y, trace nodes, via data
4. **DESIGN_3D** — Build123d solid geometry → glTF mesh binaries

Each stage emits:
- `sidebar_update` (stage active → completed)
- Multiple `log_stream` events (real-time process output)
- One `viewport_render` event (visualization payload)
- `global_status` (pipeline start/complete)
