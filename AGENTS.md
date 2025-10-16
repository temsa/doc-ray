# Repository Guidelines

## Project Structure & Module Organization
- `app/` — service code:
  - `server.py` FastAPI app and `ServeController` (API: `/submit`, `/status/{id}`, `/result/{id}`).
  - `main.py` Ray Serve entrypoint; binds actors and deployments.
  - `parser.py` `DocumentParser` deployment; parsing workflow.
  - `background_parsing.py` async parsing actor.
  - `state_manager.py` job state, TTL, and results.
  - `const.py` supported file types.
- `scripts/` — utilities (e.g., `client.py`).
- `build/` — Dockerfiles (`Dockerfile`, `Dockerfile-base`, `Dockerfile-models`).
- `run.py` — local runner that starts Ray + Serve.
- `serve_config.yaml` — Ray Serve config (for in-image/container usage).
- `serve_config_cluster.yaml` — Ray Serve config for remote clusters (ships code + installs deps).
- `kubernetes/` — KubeRay `RayService` manifest for containerized clusters.

## Build, Test, and Development Commands
- Env & deps (uv): `uv venv && source .venv/bin/activate && uv sync --all-extras`.
- Run locally: `python run.py` (HTTP on `:8639`, dashboard `:8265`).
- Serve via config (containerized): `serve run serve_config.yaml`.
- Serve on remote cluster (non-container): `serve run serve_config_cluster.yaml` with `RAY_ADDRESS` set.
- Docker: `make build`, `make run-standalone`, `make logs-standalone`, `make clean`.
- MinerU models: `mineru-models-download -m pipeline && cp ~/mineru.json .`.

## Coding Style & Naming Conventions
- Python 3.10+, PEP 8, 4-space indent; prefer type hints and module-level docstrings.
- Names: modules/files `snake_case.py`; functions/variables `snake_case`; classes `PascalCase`; constants `UPPER_SNAKE`.
- Keep API schemas (Pydantic models) adjacent to endpoints in `app/server.py`.
- Logging: use `logging` with informative, structured messages (no prints in server code).

## Testing Guidelines
- Framework: `pytest` (add as dev dep if introducing tests).
- Layout: `tests/` with `test_*.py`; mirror module paths when helpful.
- Run: `uv run pytest -q`.
- HTTP tests: use FastAPI `TestClient` against `serve.run(entrypoint)` or mock Ray actors for unit scope.

## Commit & Pull Request Guidelines
- Use Conventional Commits (seen in history): `feat: …`, `fix: …`, `chore: …`.
- PRs must include: clear description, rationale, test/validation steps, linked issues, and API/behavior notes; screenshots or `scripts/client.py` sample output when relevant.
- Keep changes focused; update `README.md`/`serve_config.yaml` when behavior or endpoints change.

## Security & Configuration Tips
- Key env vars: `STANDALONE_MODE`, `PARSER_FORCE_GPU_PER_REPLICA`, `JOB_TTL_SECONDS`, `JOB_CLEANUP_INTERVAL_SECONDS`.
- Ports: `8639` (Serve HTTP), `8265` (Ray dashboard). For Docker without GPUs, run `make run-standalone USE_GPUS=false`.
