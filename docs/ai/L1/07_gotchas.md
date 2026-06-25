# 07 · Gotchas

> Non-obvious pitfalls specific to the STT vendors recipe. Read before changing the vendor registry, agent, env, or verify scripts.

## Vendor credentials are validated in `start()`, not `__init__`

`Agent.__init__` reads `STT_VENDOR` but does **not** check credentials (no call to `build_vendor`). Credential validation happens in `start()` when `build_vendor(selected)` is called. This keeps `/get_config`, `doctor`, and contract checks key-less even when a BYO `STT_VENDOR` is configured. Do not move credential validation into `__init__`.

## Missing BYO creds → 400 with a clear error, not a silent failure

`build_vendor(name, {})` raises `ValueError` naming every missing env var. The route maps this to HTTP 400. The error message lists exactly which env vars are absent — this is the intended behavior; do not swallow it.

## Do not put `PORT` in `server/.env.example`

`verify:local:fastapi` injects a random `PORT` and loads env with `load_dotenv(override=True)`. A `PORT` line in `.env.example` (copied to `.env.local`) would clobble the injected port and break the smoke test.

## Keep `/api/*` ownership in rewrites

Adding `web/app/api/**/route.ts` for agent/token logic breaks the boundary — `verify-api-contracts.ts` explicitly fails if a `route.ts` exists under `app/api`. Token and vendor logic belongs in `server/`.

## `STT_VENDOR` is read at `__init__` but the in-UI dropdown overrides it per-call

`Agent.start()` accepts an optional `vendor` argument. The pre-call dropdown sends this as `vendor` in `POST /api/startAgent`. The `selected = (vendor or self.vendor).strip()` line means the per-call `vendor` parameter takes priority over the env default. If you change the startup logic, preserve this override path.

## `vendor` field in `startAgent` response

`POST /startAgent` returns `data.vendor` (the name of the vendor actually used). The browser does not currently display this, but tests may assert it. Do not remove `"vendor": selected` from the return dict in `agent.py`.

## camelCase request fields

`StartAgentRequest` uses `channelName`, `rtcUid`, `userUid` (camelCase) to match the browser client. Renaming one side without the other breaks the contract tests.

## VAD is top-level on `AgoraAgent`, not on the STT vendor

Unlike the realtime MLLM recipe, this cascading recipe sets `turn_detection` directly on `AgoraAgent(...)`, not on the STT vendor object. Do not set `turn_detection` on the vendor.

## UID normalization in transcripts

`normalizeTranscript` maps `uid === '0'` to the local UID. Token issuance also rejects zero/negative UIDs. Preserve both — speaker mapping and tokens depend on concrete UIDs.

## `TimelineEvent` must be imported from `EventTimeline.tsx`

The type is exported from `web/src/components/EventTimeline.tsx`. Do not create a duplicate in `web/src/types/` — the AGENTS.md pattern note is explicit about this.

## Local calls under a global proxy

Global proxies (Clash, etc.) can break `localhost`/RFC-1918 traffic. Configure the proxy to send `127.0.0.1`, `localhost`, and private ranges DIRECT, or `socksio` (in `requirements.txt`) plus `all_proxy` to route the backend through SOCKS.

## Related Deep Dives

- [stt_vendor_matrix](L2/stt_vendor_matrix.md) — vendor cred details and build patterns.
