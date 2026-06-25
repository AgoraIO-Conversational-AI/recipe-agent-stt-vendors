# 03 · Code Map

> Where things live. Two top-level modules: `web/` (Next.js client) and `server/` (FastAPI backend). Orchestration is in the root `package.json`.

## Root

| Path                  | Responsibility                                                          |
| --------------------- | ----------------------------------------------------------------------- |
| `package.json`        | Bun workspace; `setup`, `dev`, `doctor*`, `verify*`, `clean` scripts.  |
| `README.md`           | Setup, vendors table, run modes, env, troubleshooting.                  |
| `ARCHITECTURE.md`     | System shape, vendor switchboard, event surface, component boundaries.  |
| `AGENTS.md`           | Coding-agent handbook + Git Conventions + Doc Commands.                 |
| `Dockerfile`          | Backend-only image (`:8000`).                                           |
| `.github/workflows/`  | `ci.yml` (backend pytest matrix + web verify), `docker.yml`, `nightly.yml`. |

## `server/` — FastAPI backend (:8000)

| Path                               | Responsibility                                                                      |
| ---------------------------------- | ----------------------------------------------------------------------------------- |
| `src/server.py`                    | FastAPI app, CORS, route handlers (`/get_config`, `/vendors`, `/startAgent`, `/stopAgent`), error mapping, uvicorn entrypoint. |
| `src/agent.py`                     | `Agent` class: `AsyncAgora` client, `start()`/`stop()`, cascading pipeline build, `_sessions`. |
| `src/vendors.py`                   | `REGISTRY` + nine `build_<vendor>(env)` functions + `build_vendor`, `available`, `required_env`, `needs_key`. |
| `scripts/run_fake_server.py`       | Boots `server.app` with a `FakeAgent` for the local FastAPI smoke test.             |
| `tests/test_vendors.py`            | Asserts all nine vendors construct and emit config; asserts BYO missing-cred raises. |
| `tests/test_agent_construction.py` | Builds real `AgoraAgent` with faked SDK session, asserts result shape.              |
| `tests/test_agent_config.py`       | Asserts `Agent.__init__` sets default vendor to `deepgram`.                         |
| `tests/conftest.py`                | `fake_env` fixture; no cloud, no real creds.                                        |
| `.env.example`                     | Env template with all nine vendor credential stanzas commented out.                 |
| `requirements*.txt`                | Runtime + dev (pytest) deps.                                                        |

## `server/src/server.py` routes

- `GET /get_config` — token + channel/UID config (always key-less).
- `GET /vendors` — nine-vendor list with `needs_key` and `required_env` for the in-UI dropdown.
- `POST /startAgent` — start the cascading agent session with the selected STT.
- `POST /stopAgent` — stop by `agent_id`.

## `web/` — Next.js client (:3000)

| Path                                        | Responsibility                                                          |
| ------------------------------------------- | ----------------------------------------------------------------------- |
| `next.config.ts`                            | `/api/*` rewrites to `AGENT_BACKEND_URL` (4 routes); strict mode; Turbopack root. |
| `src/services/api.ts`                       | Browser API client: `getConfig`, `getVendors`, `startAgent`, `stopAgent`. |
| `src/lib/conversation.ts`                   | Transcript normalization, timestamp/UID mapping, visualizer state.      |
| `src/lib/agora.ts`                          | `DEFAULT_AGENT_UID` constant.                                           |
| `src/components/EventTimeline.tsx`          | `TimelineEvent` type export + EventTimeline UI (state/metric/error/turn, capped at 50). |
| `src/components/LandingPage.tsx`            | Fetches vendors, calls `getConfig`/`startAgent`, owns RTM login + teardown. |
| `src/components/ConversationComponent.tsx`  | RTC join, mic publish, transcript/metrics/state listeners.              |
| `src/components/QuickstartPreCallCard.tsx`  | Pre-call screen with vendor dropdown and start button.                  |
| `src/components/QuickstartPipelineMetrics.tsx` | Pipeline metrics panel.                                              |
| `src/components/QuickstartTranscriptPanel.tsx` | Annotated transcript panel with agent state header.                  |
| `scripts/verify-api-contracts.ts`           | Asserts rewrites + client paths + response envelope (no network).       |
| `scripts/verify-local-proxy.ts`             | Stub backend; proxies `/api/*` through the rewrite map.                 |
| `scripts/verify-local-fastapi.ts`           | Spawns real FastAPI with `FakeAgent`; proxies routes end-to-end.        |
| `scripts/verify-local-llm.ts`               | LLM service smoke (unused by default pipeline — no `llm/` service).    |
| `scripts/doctor.ts`                         | Web prerequisite check.                                                 |
| `.claude/skill-*.md`                        | Contributor reference notes for RTC/RTM/ConvoAI integration.            |

## `TimelineEvent` import rule

`TimelineEvent` is exported from `web/src/components/EventTimeline.tsx`. Import it from there — do not create a separate types file for it.

## Related Deep Dives

- None. For runtime flow see [02_architecture](02_architecture.md); for contracts see [06_interfaces](06_interfaces.md).
