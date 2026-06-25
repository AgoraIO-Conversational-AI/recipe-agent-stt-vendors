# 05 · Workflows

> Step-by-step guides for the common changes in this recipe. Each ends with the narrowest verify command to run.

## Swap the active STT vendor

**Option A — via the UI** (no restart): pick from the pre-call dropdown; the selected vendor name is passed in `POST /api/startAgent` as `vendor`.

**Option B — via env**:
1. Set `STT_VENDOR=<name>` in `server/.env.local` (e.g. `STT_VENDOR=assemblyai`).
2. Add the vendor's required credential env vars (see `required_env(name)` in `vendors.py` or [stt_vendor_matrix](L2/stt_vendor_matrix.md)).
3. Restart `bun run dev`.
4. Verify: `bun run doctor:local`.

## Add or change an STT vendor

1. Add a `build_<vendor>(env)` function to `server/src/vendors.py` (use the existing ones as copy-pasteable templates).
2. Add a `REGISTRY` entry: `"<name>": (build_<vendor>, ["VAR1", "VAR2"])` — empty creds list if keyless.
3. Update `server/.env.example` with a commented-out stanza for the new vendor's credentials.
4. Run the vendor test: `cd server && pytest tests/test_vendors.py -v`.
5. Verify: `bun run verify:backend`.

## Add or change a browser-facing route

1. Add the FastAPI handler in `server/src/server.py` (return the `{ code, msg, data }` envelope).
2. Add the `/api/<name>` → `/<name>` mapping in `web/next.config.ts` `rewrites()`.
3. Add a client helper in `web/src/services/api.ts`.
4. Extend `web/scripts/verify-api-contracts.ts` with the new path + envelope assertions.
5. Verify: `bun run verify:web` (and `bun run verify:local:fastapi` if it should go through the real backend).

## Change the agent prompt / greeting

1. Set `AGENT_GREETING` (env) or edit the default in `server/src/agent.py`.
2. Verify: `bun run verify:backend`.

## Change the LLM or TTS

1. Edit `llm = OpenAI(...)` or `tts = MiniMaxTTS(...)` in `Agent.start()` (`server/src/agent.py`).
2. If a BYO key is needed, add it to `server/.env.example` and document it in `01_setup.md`.
3. Verify: `bun run verify:backend` + `cd server && pytest tests -v`.

## Change VAD config

1. Edit the `turn_detection` dict in `AgoraAgent(...)` in `Agent.start()`.
2. Verify: `bun run verify:local:fastapi`.

## Run / debug locally

```bash
bun run dev              # both processes
bun run doctor:local     # check creds + .env.local before a live call
```

## Verify before finishing

| Change touches…              | Run                                                                  |
| ---------------------------- | -------------------------------------------------------------------- |
| Web only                     | `bun run verify:web`                                                  |
| Backend logic / cascade      | `bun run verify:backend` + `cd server && pytest tests -v`             |
| Route/proxy boundary         | `bun run verify:web:proxy` and/or `bun run verify:local:fastapi`     |
| Anything end-to-end (local)  | `bun run verify:local`                                                |

## Deploy

1. Deploy `web/` as a Next.js app.
2. Deploy `server/` (or any reachable FastAPI host); the published backend-only image is `ghcr.io/AgoraIO-Conversational-AI/recipe-agent-stt-vendors` on `v*` tags.
3. Set `AGENT_BACKEND_URL` in the web deployment so rewrites reach the backend.
4. On the default `deepgram` vendor no STT key is needed in the backend image.

## Related Deep Dives

- [stt_vendor_matrix](L2/stt_vendor_matrix.md) — full vendor creds and SDK constructor details.
- [session_lifecycle](L2/session_lifecycle.md) — client-side join/renewal/teardown and EventTimeline wiring.
