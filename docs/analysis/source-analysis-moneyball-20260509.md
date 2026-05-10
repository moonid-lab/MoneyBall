# MoneyBall Source Analysis Report

Date: 2026-05-09
Mode: `$ralph` static-only source analysis
Plan: `.omx/plans/source-analysis-strategy-20260509T103412Z.md`

## Scope and verification

Static-only source analysis completed against the approved plan.

- No source files were edited.
- No app/server/tests/build/scripts were run.
- Source-related `git status --short` was checked clean for `frontend`, `backend`, package/docs/ops files.
- Final architect verification: **APPROVED**.

## 1) Project shape

MoneyBall is a Vue + Flask app for a 5-step workflow:

1. Upload docs + simulation requirement
2. Generate ontology/project
3. Build Zep graph
4. Create/prepare/run simulation
5. Generate report, then chat/interview with report/simulation agents

Key entrypoints:

| Area | Evidence |
|---|---|
| Backend app factory | `backend/app/__init__.py:65-74` registers `/api/graph`, `/api/simulation`, `/api/report`, `/health` |
| Backend runtime | `backend/run.py` validates config then starts Flask |
| Frontend router | `frontend/src/router/index.js:9-44` maps `/`, `/process/:projectId`, `/simulation/:simulationId`, `/simulation/:simulationId/start`, `/report/:reportId`, `/interaction/:reportId` |
| Frontend API base | `frontend/src/api/index.js:5-36` uses `VITE_API_BASE_URL || http://localhost:5001`, 5-min timeout, rejects `{ success:false }` |

## 2) Bounded inventory

| Surface | Classification | Notes |
|---|---|---|
| `backend/app/api/graph.py` | known workflow | Ontology, graph build, graph data, task polling |
| `backend/app/api/simulation.py` | known workflow | Create/prepare/start/stop/status/profiles/interview/env control |
| `backend/app/api/report.py` | known workflow | Report generation, status, logs, chat, tools |
| `backend/app/models/project.py` | persistence | Project JSON/files/extracted text |
| `backend/app/models/task.py` | volatile state | In-memory task manager |
| `backend/app/services/simulation_manager.py` | persistence/orchestration | `state.json`, profiles, config |
| `backend/app/services/simulation_runner.py` | runtime lifecycle | subprocesses, `run_state.json`, logs |
| `backend/app/services/report_agent.py` | report engine/persistence | `meta.json`, `progress.json`, sections, logs, full report |
| `frontend/src/views/MainView.vue` | active workflow | Router imports it as `Process` |
| `frontend/src/views/Process.vue` | likely orphan | Duplicate/legacy; router does not import it |
| `frontend/src/store/pendingUpload.js` | active transient bridge | In-memory Home -> Process upload handoff |
| `docker-compose.yml`, `.github/workflows/*`, `docs/github/*` | ops/naming | MiroFish/MoneyBall upstream/deploy references |

## 3) Frontend -> backend contract matrix

| Step | Frontend caller | Backend endpoint | Payload / expected response |
|---|---|---|---|
| Upload/ontology | `Home.vue:297-310`, `MainView.vue:194-221` | `POST /api/graph/ontology/generate` | multipart `files[]`, `simulation_requirement`; expects `project_id` |
| Graph build | `MainView.vue:276-356` | `POST /api/graph/build`, `GET /api/graph/task/:taskId`, `GET /api/graph/data/:graphId` | expects `task_id`, then `completed/failed`, then graph nodes/edges |
| Create simulation | `Step1GraphBuild.vue:213-235` | `POST /api/simulation/create` | sends `project_id`, `graph_id`, `enable_twitter:true`, `enable_reddit:true`; expects `simulation_id` |
| Prepare simulation | `Step2EnvSetup.vue:771-905` | `POST /api/simulation/prepare`, `POST /api/simulation/prepare/status` | sends `simulation_id`, LLM/profile settings; polls `task_id` + `simulation_id` |
| Start run | `Step3Simulation.vue:400-430` | `POST /api/simulation/start` | sends `platform:'parallel'`, `force:true`, graph memory update; expects run state/PID |
| Generate report | `Step3Simulation.vue:644-678` | `POST /api/report/generate` | sends `simulation_id`, `force_regenerate:true`; expects `report_id` and routes immediately |
| Report display | `Step4Report.vue:2024-2165` | `GET /api/report/:id/agent-log`, `/console-log` | polls logs; completion only on `report_complete` |
| Interaction | `Step5Interaction.vue:682-773` | `POST /api/report/chat`, `POST /api/simulation/interview/batch` | expects `response` or `answer` |

## 4) Ranked findings

### 1. Confirmed / High — async task state is volatile

`TaskManager` stores tasks only in process memory: `backend/app/models/task.py:62-106`.

Impact:

- Graph build persists `graph_build_task_id` to project JSON, but `/api/graph/task/:taskId` only reads in-memory `TaskManager`: `backend/app/api/graph.py:364-372`, `534-550`.
- Report generation creates an in-memory task: `backend/app/api/report.py:109-122`.
- Report status falls back to `TaskManager().get_task(task_id)`: `backend/app/api/report.py:203-265`.

Risk: server restart during graph/report generation can strand the UI with stale task IDs.

### 2. Confirmed / High path risk — stale `run_state.json` can block restart

Runner state is split:

- In-memory process registries: `backend/app/services/simulation_runner.py:219-225`
- File-backed `run_state.json` reload: `backend/app/services/simulation_runner.py:230-310`
- Start rejects existing `RUNNING` / `STARTING`: `backend/app/services/simulation_runner.py:313-338`

API force cleanup happens only inside `state.status != READY`: `backend/app/api/simulation.py:1540-1577`.

Boundary: runtime trigger was not reproduced, but the stale-file rejection path is evident statically.

### 3. Confirmed / Medium — report status frontend/backend mismatch

Frontend wrapper:

- `frontend/src/api/report.js:15-17` calls `GET /api/report/generate/status?report_id=...`

Backend route:

- `backend/app/api/report.py:203-229` only supports `POST /api/report/generate/status` with JSON `task_id` or `simulation_id`.

Primary UI impact is low because Step4 does not use this wrapper; it polls logs instead. Risk remains medium for shared/API consumers.

### 4. Confirmed / Medium-High — Twitter profile format mismatch

Preparation writes:

- Reddit: `reddit_profiles.json`
- Twitter: `twitter_profiles.csv`

Evidence: `backend/app/services/simulation_manager.py:329-374`.

But generic profile reader always looks for JSON:

- `backend/app/services/simulation_manager.py:481-494` uses `f"{platform}_profiles.json"`

Realtime endpoint handles the split correctly, and observed UI paths mostly use realtime Reddit profiles. Non-realtime `/profiles?platform=twitter` is the risky path.

### 5. Confirmed / Medium — prepared check ignores platform toggles

`_check_simulation_prepared()` always requires:

- `state.json`
- `simulation_config.json`
- `reddit_profiles.json`
- `twitter_profiles.csv`

Evidence: `backend/app/api/simulation.py:240-270`.

But create supports `enable_twitter` / `enable_reddit`: `backend/app/api/simulation.py:218-224`.

Single-platform simulations can be falsely treated as not prepared.

### 6. Likely / Medium — `state.json` and `run_state.json` can diverge

Start updates simulation state to running: `backend/app/api/simulation.py:1612-1614`.

Runner monitor completion updates only run state in the inspected path: `backend/app/services/simulation_runner.py:524-547`.

Boundary: `/close-env` is an exception that writes completed state, but natural runner completion/failure may leave stale `state.json`.

## 5) Orphans / unknowns

| Item | Status |
|---|---|
| `frontend/src/views/Process.vue` | likely orphan; router imports `MainView.vue` as `Process` |
| `getReportStatus` wrapper | exported but not observed in active primary UI |
| `getSimulationProfiles` non-realtime | defined; active UI mostly uses realtime Reddit profile loading |
| Optional backend endpoints | graph list/delete/tasks, simulation timeline/stats/comments/download/single-interview, report download/delete/tools/progress/sections are candidate/admin/optional surfaces |

## 6) MoneyBall / MiroFish naming

Confirmed naming split:

- Repo/workspace is `MoneyBall`
- Product/upstream/deployment still uses `MiroFish`
- Backend health/logging also says MiroFish: `backend/app/__init__.py:61-77`
- Docs/workflows reference upstream `666ghj/MiroFish`

Assessment: operational/documentation risk, not a direct runtime bug. Runtime paths are mostly relative/config-driven.

## 7) Recommended fix order

1. Make status endpoints file-first or persist task records.
2. Add stale-run reconciliation before simulation start.
3. Fix report status contract: either frontend POSTs `task_id/simulation_id`, or backend supports GET/report-id from durable progress.
4. Fix Twitter profile reader to support CSV.
5. Make prepared-file checks platform-aware.
6. Consolidate or remove orphan/duplicate frontend workflow surfaces.

## Verification evidence

- Static source reads only.
- Source-related `git status --short` checked clean for `frontend`, `backend`, package/docs/ops files before saving this report artifact.
- Architect verifier approved with minor wording corrections.
- No source edits; no runtime execution; no build/test execution by approved scope.
