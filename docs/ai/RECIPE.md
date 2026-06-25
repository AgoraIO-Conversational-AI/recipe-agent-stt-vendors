---
recipe_version: 1.0.0
recipe_status: experimental
extension_points:
  - id: stt.vendor-registry
    name: STT vendor REGISTRY in vendors.py (add/remove vendors)
  - id: api.routes
    name: Browser-facing API routes
  - id: agent.cascade-config
    name: LLM model, TTS model/voice, greeting, VAD, and session parameters
  - id: web.conversation-ui
    name: Conversation UI panels, EventTimeline, and vendor dropdown
  - id: verification.contracts
    name: Contract, proxy, and local FastAPI smoke verification
invariants:
  - id: api.rewrite-boundary
    summary: Browser calls stay on /api/* and Next rewrites to FastAPI; no Route Handlers for agent/token logic.
  - id: secrets.server-only
    summary: Agora App Certificate and BYO STT vendor credentials stay in the Python backend.
  - id: stt.registry-driven
    summary: The STT leg is built from the REGISTRY in vendors.py via build_vendor(); never hardcoded in agent.py.
  - id: stt.creds-at-start
    summary: Vendor credentials are validated in start() (not __init__) so /get_config stays key-less.
  - id: cascade.fixed-llm-tts
    summary: Only the STT leg is swappable; LLM (OpenAI gpt-4o-mini) and TTS (MiniMaxTTS) stay on keyless configs.
  - id: token.uid-concrete
    summary: Backend resolves missing, zero, or negative UIDs before issuing an RTC+RTM token.
stable_contracts:
  - id: env.required
    summary: AGORA_APP_ID and AGORA_APP_CERTIFICATE are always required; AGENT_BACKEND_URL is required by deployed web rewrites.
  - id: api.core-routes
    summary: GET /api/get_config, GET /api/vendors, POST /api/startAgent, and POST /api/stopAgent remain the browser-facing contract.
  - id: api.vendors-route
    summary: GET /api/vendors returns { default, vendors: [{name, needs_key, required_env}] } for the in-UI switcher.
  - id: response.envelope
    summary: Successful backend responses use { code, msg, data }.
---

# Recipe Contract

This base recipe defines the reusable surface for a Python-backed Agora Conversational AI **STT vendors** quickstart: a cascading STT→LLM→TTS pipeline where the STT leg is a data-driven switchboard behind a Next.js web client.

## Recipe Role

- Role: `base` recipe (self-contained, clone-and-run; no `Extends` pin).
- Target audience: developers who want to explore or compare Agora-supported STT vendors in a working voice agent, or who need to wire a specific BYO STT provider.
- Reuse model: clone, bind project, run on default keyless Deepgram STT, then swap `STT_VENDOR` + credentials to test any of the nine vendors.

## Recipe Scope

- Python FastAPI token generation and managed agent lifecycle.
- A data-driven STT vendor switchboard (`vendors.py`) covering nine A4.1 STT providers, selected via `STT_VENDOR`.
- Fixed LLM (`OpenAI gpt-4o-mini`, managed) and TTS (`MiniMaxTTS speech_2_6_turbo`, managed) — only the STT leg changes.
- Next.js browser UI with RTC audio, RTM transcript, EventTimeline (state/metric/error/turn events), and an in-UI STT vendor dropdown.
- Rewrite-only `/api/*` browser facade hiding backend placement.
- Contract, proxy, and local FastAPI smoke verification that need no live Agora calls.

## Baseline Implementation Guidance

Use this repo's source and progressive disclosure docs as the starting point, then customize. Do not recreate the Agora ConvoAI integration from memory — vendor schemas, SDK builder fields, token behavior, and RTM details drift. Copy verified patterns from this repo.

## Extension Points

| ID | Surface | How to extend | Required follow-up |
| -- | ------- | ------------- | ------------------ |
| `stt.vendor-registry` | `server/src/vendors.py` `REGISTRY` + per-vendor `build_<vendor>` | Add a `build_<vendor>(env)` function and a `REGISTRY` entry with `(builder, [required_env_vars])`. | Update `server/.env.example`; run `pytest tests/test_vendors.py`. |
| `api.routes` | `server/src/server.py`, `web/next.config.ts`, `web/src/services/api.ts` | Add FastAPI route, add rewrite, add browser fetch helper. | Extend `web/scripts/verify-api-contracts.ts`; add proxy/fastapi coverage if it belongs in local verification. |
| `agent.cascade-config` | `server/src/agent.py` | Change `OpenAI` model, `MiniMaxTTS` model/voice, `greeting`, `turn_detection`, or session `parameters`. | Run `bun run verify:backend` + `pytest tests`; document new env in `server/.env.example` (never add `PORT`). |
| `web.conversation-ui` | `web/src/components/*`, `web/src/lib/conversation.ts` | Customize pre-call, EventTimeline, transcript, vendor dropdown, or connection status. | Preserve RTC/RTM lifecycle ownership and transcript UID normalization. |
| `verification.contracts` | `web/scripts/*.ts`, root `package.json` | Add checks for new browser/backend boundaries. | Keep checks runnable without live Agora credentials. |

## Invariants

- Browser code calls only `/api/get_config`, `/api/vendors`, `/api/startAgent`, and `/api/stopAgent`.
- Next.js owns `/api/*` through rewrites only; no `web/app/api/**/route.ts` for agent/token logic.
- FastAPI owns token generation, `AGORA_APP_CERTIFICATE`, BYO STT credentials, and agent lifecycle.
- The STT vendor is built from `REGISTRY` via `build_vendor(name)` — never constructed inline in `agent.py`.
- Vendor credentials are validated in `start()`, not `__init__` — `/get_config` and doctor checks stay key-less.
- The backend issues one RTC+RTM-capable token for a concrete non-zero UID.

## Stable Contracts

| Contract | Stable shape |
| -------- | ------------ |
| Required backend env | `AGORA_APP_ID`, `AGORA_APP_CERTIFICATE` |
| Optional backend env | `STT_VENDOR` (default `deepgram`), `STT_MODEL`, `STT_LANGUAGE`, `AGENT_GREETING` |
| Required web deploy env | `AGENT_BACKEND_URL` |
| `GET /api/get_config` | Query `channel?`, `uid?`; returns `data.app_id`, `data.token`, `data.uid`, `data.channel_name`, `data.agent_uid`. |
| `GET /api/vendors` | Returns `data.default` (string) and `data.vendors` (array of `{name, needs_key, required_env}`). |
| `POST /api/startAgent` | Body `{ channelName, rtcUid, userUid, vendor?, parameters? }`; returns `data.agent_id`, `data.channel_name`, `data.vendor`, `data.status`. |
| `POST /api/stopAgent` | Body `{ agentId }`; returns `{ code: 0, msg: "success" }`. |
| Success envelope | `{ "code": 0, "msg": "success", "data": ... }` where the route has data. |
| Verification entry points | `bun run verify:web`, `bun run verify:backend`, `bun run verify:web:proxy`, `bun run verify:local:fastapi`, `bun run verify:local`. |

## Internal / Subject to Change

- Visual layout, component composition, Tailwind classes, and assets under `web/src/components/`.
- Exact LLM model name (`gpt-4o-mini`), TTS model/voice, and greeting text, as long as they stay documented extension points.
- VAD config (`speech_threshold`, `interrupt_duration_ms`, `prefix_padding_ms`, `silence_duration_ms`).
- In-memory `Agent._sessions` details; the stable behavior is start by channel/user and stop by returned `agent_id`.
- Verification internals under `web/scripts/`; the stable surface is the root script names and what they assert.
- `agora-agents` SDK minor-version behavior; this recipe lower-bounds `>=2.3.0` but does not freeze every field.

## Related Progressive Disclosure Docs

- `L1/01_setup.md` — setup, env, and commands.
- `L1/02_architecture.md` — request flow, pipeline, and vendor switchboard topology.
- `L1/05_workflows.md` — common modification workflows.
- `L1/06_interfaces.md` — route, rewrite, env, vendor registry, and cascade contracts.
- `L1/L2/stt_vendor_matrix.md` — full nine-vendor matrix with creds, defaults, and SDK constructors.
- `L1/L2/session_lifecycle.md` — RTC/RTM/session orchestration and EventTimeline wiring.
