# MoneyBall Copilot Instructions

## Build, run, and test commands

Root commands:

- `npm run setup` - install root and frontend Node dependencies
- `npm run setup:backend` - install backend Python dependencies with `uv`
- `npm run setup:all` - install everything
- `npm run dev` - run Flask backend and Vite frontend together
- `npm run backend` - run only the backend (`backend/run.py`)
- `npm run frontend` - run only the frontend
- `npm run build` - production build for the frontend

Backend test runner:

- `cd backend && uv run pytest`
- Single test: `cd backend && uv run pytest path/to/test_file.py -k test_name`

There is no committed repo-level lint or typecheck command today, so do not invent one in responses or automation.

## High-level architecture

This repo is a customized fork of MiroFish. Keep repo-maintenance guidance aligned with `README.md` and `docs/github/GIT_UPSTREAM_SYNC_WORKFLOW.md`: upstream changes land through `upstream-main -> main` PRs, while feature work is meant to flow from `develop`.

The product is a Vue 3 + Vite frontend (`frontend/src`) talking to a Flask backend (`backend/app`). The frontend route flow is:

- `/` -> `Home.vue` for file selection and simulation requirement input
- `/process/:projectId` -> `MainView.vue`, which drives the first two workflow stages and the live graph panel
- `/simulation/:simulationId` and `/simulation/:simulationId/start` -> simulation preparation and execution views
- `/report/:reportId` and `/interaction/:reportId` -> report review and follow-up interaction

The important workflow is end-to-end, not per-file:

1. **Graph build stage**: `Home.vue` stores selected files in `frontend/src/store/pendingUpload.js`, then `MainView.vue` performs the first real upload. Backend `app/api/graph.py` creates a `Project`, extracts text, asks the LLM for an ontology, and then `GraphBuilderService` chunks the text and builds a Zep graph.
2. **Simulation preparation and run stage**: `app/api/simulation.py` creates a simulation from the built graph. `SimulationManager` reads filtered Zep entities, generates OASIS agent profiles plus simulation config, and writes everything under `backend/uploads/simulations/<simulation_id>`. `SimulationRunner` then launches the scripts in `backend/scripts` and tracks live run state and agent actions.
3. **Report and interaction stage**: `app/api/report.py` starts asynchronous report generation. `ReportAgent` plans sections, uses Zep-backed tools to gather evidence, writes report artifacts under `backend/uploads/reports/<report_id>`, and later serves chat-style follow-up interaction on top of the same simulation/report context.

The backend app factory in `backend/app/__init__.py` registers three blueprints: `/api/graph`, `/api/simulation`, and `/api/report`. Configuration is loaded from the repository root `.env`, not a backend-local env file.

## Key conventions

- **Localization is shared across frontend and backend.** Frontend `vue-i18n` loads messages from the root `locales/` directory, saves the locale in `localStorage`, and sends it as `Accept-Language` on every API request. Backend responses should use `app/utils/locale.py` helpers such as `t(...)` instead of hard-coded user-facing strings.
- **Background work must preserve locale context.** Long-running graph/report tasks spawn background threads; existing code captures the current locale before spawning and calls `set_locale(...)` inside the worker. Keep that pattern when adding threaded work.
- **Long-running operations are poll-driven.** Graph build, simulation preparation, and report generation return IDs immediately and are polled for status from the frontend. Prefer extending the existing task/status pattern over introducing a second async mechanism.
- **Persistence is file-backed, not database-backed.** Projects, simulations, extracted text, run state, and report artifacts are written under `backend/uploads/` as JSON/text/files. Before introducing a database or changing on-disk layout, inspect `ProjectManager`, `SimulationManager`, and `ReportManager`.
- **Frontend API calls assume a uniform response envelope.** `frontend/src/api/index.js` rejects any payload with `success: false`, so backend endpoints should keep returning `{ success, data }` or `{ success, error }` consistently.
- **ID prefixes and directory contracts matter.** Existing flows rely on generated IDs such as `proj_...`, `sim_...`, `report_...`, and `mirofish_...`. Preserve those patterns unless you update every dependent surface.
- **Environment variables are shared by the Flask app and simulation scripts.** The root `.env` is the source of truth for `LLM_*`, `ZEP_API_KEY`, and optional boost-model settings; backend scripts load from that same root file.
