# 04 · Conventions

> Coding patterns shared across `server/` and `web/`. Follow these to keep local and deployed modes aligned and the vendor registry consistent with sibling recipes.

## Boundary ownership

- Browser code calls only `/api/*`. Backend placement is hidden behind Next rewrites (`web/next.config.ts`).
- **Never** add `web/app/api/**/route.ts` for agent/token logic — `verify-api-contracts.ts` fails the build if a `route.ts` appears under `app/api`.
- Token generation, the App Certificate, and BYO STT credentials stay in `server/`.

## Vendor registry pattern

- Add or change a vendor by editing its `build_<vendor>(env)` function in `vendors.py` **and** the corresponding `REGISTRY` line — never both places separately.
- Each builder is a standalone, copy-pasteable sample: it shows the real SDK constructor and exactly which env vars it needs.
- `build_vendor(name, env)` is the only call site; `agent.py` does not construct vendor objects directly.
- Empty `creds` list (`"deepgram"`, `"ares"`) = Agora-managed / keyless. Non-empty = BYO.
- The framework (`build_vendor` / `required_env` / `available` / `needs_key`) is shared across sibling recipes. Keep its interface identical when porting.

## Backend (Python / FastAPI)

- Async throughout: route handlers are `async def`; the agent uses `AsyncAgora` and `create_async_session`.
- Request bodies are Pydantic models (`StartAgentRequest`, `StopAgentRequest`). Field names are **camelCase** (`channelName`, `rtcUid`, `userUid`) to match the browser client.
- Error mapping is centralized: `_to_http_error()` maps `ValueError → 400`, `RuntimeError → 500`, else 500. `_log_route_error()` logs with safe context + traceback. Raise plain `ValueError`/`RuntimeError`; let the route convert.
- Logging via `logging.getLogger("uvicorn.error")`.
- Env read with `os.getenv`; `.env.local` then `.env` loaded with `override=True`.

## Response envelope

All backend JSON responses use:

```json
{ "code": 0, "msg": "success", "data": { } }
```

`data` is present only when the route returns a payload. The browser client treats `code !== 0` (or missing `data`) as an error.

## Cascade configuration

- The cascade is built in `Agent.start()` (`agent.py`): `stt = build_vendor(selected)`, `llm = OpenAI(model="gpt-4o-mini")`, `tts = MiniMaxTTS(...)`, then `.with_stt(stt).with_llm(llm).with_tts(tts)`.
- `turn_detection` is set as a top-level dict on `AgoraAgent(...)` — **not** on an individual vendor. This is the opposite of the realtime MLLM recipe.
- Do not validate BYO vendor credentials in `__init__` — they belong in `start()` via `build_vendor`.

## Web (TypeScript / Next.js)

- Lint/format with Biome (`bun run lint`, `bun run lint:fix` in `web/`).
- RTC client creation must be StrictMode-safe (strict mode is on).
- Transcript speaker mapping uses real UIDs (`normalizeTranscript` maps `uid === '0'` to the local UID); do not heuristically guess speakers.
- API client lives in `src/services/api.ts`; UI never calls `fetch` to the backend directly.
- `TimelineEvent` is imported from `src/components/EventTimeline.tsx`; do not duplicate this type.

## Testing approach

- Backend: `pytest` in `server/`, standalone — `conftest.py` fakes env and the SDK session, so no cloud or real creds are needed.
- Web: contract/proxy/fastapi smoke scripts under `web/scripts/` run without live Agora calls.
- Run the **narrowest** relevant verify command before finishing (see [05_workflows](05_workflows.md)).

## Doc upkeep

When you change request/response contracts, env vars, vendor entries, or workflow, update the web client, backend, contract checks, README, **and** the matching `docs/ai/L1/` file together, then bump `Last Reviewed` in [L0](../L0_repo_card.md).

## Related Deep Dives

- [stt_vendor_matrix](L2/stt_vendor_matrix.md) — vendor-by-vendor SDK constructor details.
