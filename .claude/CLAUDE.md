# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Telusko Workflow Engine — a Kanban board for tracking video production through a fixed pipeline. Two parts:

- `frontend/` — static vanilla-JS SPA (`index.html`, `app.js`, `styles.css`). No build step.
- `backend/` — FastAPI app (async SQLAlchemy + SQLite, JWT auth) that serves the JSON API **and** mounts `../frontend` as static files at `/`.
- `specs/` — `api-contract.md` (request/response shapes the frontend expects), `seed-data.json` (consumed by `backend/seed.py`), `features/`, `security/`.

## Tech Stack

- FastAPI, Pydantic v2 (+ pydantic-settings), SQLAlchemy 2.0 async with aiosqlite
- Alembic for migrations
- python-jose (JWT), passlib[bcrypt] (password hashing)
- pytest + pytest-asyncio + httpx for tests
- Python >= 3.12, managed with **uv**

## Running locally

All commands from `backend/`:

```sh
uv sync                        # install deps
uv run alembic upgrade head    # apply migrations
uv run python seed.py          # idempotent seed of tasks + users from specs/seed-data.json
uv run python start.py         # uvicorn on 127.0.0.1:${PORT:-8000} with reload
uv run pytest                  # tests (asyncio_mode=auto, testpaths=tests)
```

Then open http://localhost:8000 — the backend serves the frontend, and `app.js` uses relative URLs (`/api/...`). `/health` is a liveness probe. The `/run` skill launches everything.

Docker (`Dockerfile` at repo root): runs `alembic upgrade head && python seed.py && uvicorn ... --port ${PORT:-8080}` — targeted at Cloud Run.

Config comes from `backend/app/core/config.py` (env / `.env`): `DATABASE_URL`, `PORT`, `JWT_SECRET_KEY`, `JWT_ALGORITHM`, `JWT_ACCESS_TOKEN_EXPIRE_MINUTES`. The default secret is a placeholder — set a real one outside local dev.

## Architecture

### Pipeline & roles (shared concepts)

- **States** — ordered: `code_ready` → `recorded` → `editing` → `uploaded` → `published`. Defined in `frontend/app.js` (`STATES`), `backend/app/models/task.py` (`TaskState`, `STATES_ORDER`), and the `<select id="task-state">` in `index.html`. All three must stay in sync.
- **Roles** — two different enums exist, don't confuse them:
  - `UserRole` (`models/user.py`, in JWT `role` claim): `admin`, `content_team`, `video_editor`, `uploader`.
  - `AssignedRole` (`models/task.py`, a task's owner field): `Admin`, `Content`, `Editor`, `Uploader`.
  - Frontend `ROLE_MAP` maps `UserRole` → `ROLE_CONFIG` keys (which match `AssignedRole`).
- **Advance permissions** — which stages a role may move a card forward *from*:
  - admin: code_ready, recorded, editing, uploaded
  - content_team: code_ready
  - video_editor: recorded, editing
  - uploader: uploaded

  Mirrored in frontend `ROLE_CONFIG.canAdvance` and backend `_ADVANCE_ALLOWED` (`app/api/v1/tasks.py`). Keep them in sync.

### Backend (`backend/app/`)

- `main.py` — app factory: CORS (localhost 8000/8080/5500 + `null` for `file://`), routers, `StaticFiles` mount of `../frontend` (mounted last so `/api/*` wins), `/health`.
- `api/v1/auth.py` — `/api/auth`: `POST /login` (OAuth2 form → `{access_token, token_type}`), `POST /users` (admin only), `GET /me`.
- `api/v1/tasks.py` — `/api/tasks`: `GET` (optional `state` / `assigned_role` filters), `POST` (admin, content_team), `PUT /{id}` (partial; RBAC check when `state` changes), `DELETE /{id}` (admin, 204). Responses use camelCase aliases (`assignedRole`, `createdAt`) via `response_model_by_alias=True`.
- `services/task_service.py` — DB access/CRUD for tasks; routes delegate here.
- `security/` — `jwt.py` (encode/decode), `password.py` (bcrypt), `dependencies.py` (`get_current_user`, `require_role(*roles)` → 401/403).
- `models/`, `schemas/` — SQLAlchemy models and Pydantic schemas.
- `core/` — `config.py` (settings), `database.py` (async engine, `AsyncSessionLocal`, `get_db`).
- `alembic/versions/` — `create_tasks_table`, `add_users_table`.
- `routes/tasks.py` — legacy stub (comments only); the live router is `api/v1/tasks.py`. `backend/main.py` is a uv-generated placeholder — the real entry is `start.py` / `app.main:app`.

Known deviation: the advance-permission check (`_ADVANCE_ALLOWED`) currently lives in the route, not the service layer. Move it into `task_service` when touching that code.

### Frontend (`frontend/app.js`)

- Login form posts to `/api/auth/login`; token stored in `localStorage` (`telusko_token`); role is read from the JWT payload for UI only.
- `apiFetch` adds `Authorization: Bearer`; any 401 clears the token and shows the login form.
- `api.{getTasks,createTask,updateTask,deleteTask}` wraps all server calls. `getTasks` degrades gracefully to the local `tasks` copy; others surface errors via `showToast(..., 'error')`.
- `renderBoard()` is the single re-render entry point, rebuilt from scratch after each awaited API call. Drag-and-drop is Admin-only; other roles use "Move Forward →".
- Frontend permission checks are UX only — the backend is the trust boundary.

### Tests

`backend/tests/` currently contains only placeholders (`conftest.py` lists planned fixtures: in-memory aiosqlite engine, per-test session, httpx `AsyncClient` with `get_db` overridden). No real tests yet.

## UV package manager

Use uv as the package manager. All Python commands run through uv — `uv pip install` / `uv add` for packages, `uv run pytest` for tests, `uv run python` for scripts. Never use pip or python directly.

## Code Style

- Python: snake_case for functions and variables
- Always use type hints
- Never put business logic in routes — use the service layer
- Schemas in `schemas/`, models in `models/`

## Rules

- NEVER modify files in /frontend
- NEVER commit .env files
- Always create an Alembic migration when changing models
