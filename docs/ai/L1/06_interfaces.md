# 06 · Interfaces

> Boundary contracts: backend routes, the `/api/*` rewrite map, env vars, the response envelope, and the vendor registry API.

## Backend routes (port 8000)

The browser calls these as `/api/<name>`; Next rewrites to the backend `/<name>`.

### `GET /get_config`

- Query (optional): `channel?: string`, `uid?: int` (≤ 0 or missing → backend generates one).
- Returns `data`: `{ app_id, token, uid (string), channel_name, agent_uid (string) }`.
- Token is a Token007 RTC+RTM token, expiry 3600s, for a concrete non-zero UID.
- Always works key-less (no STT credentials needed for config).

### `GET /vendors`

- No parameters.
- Returns `data`: `{ default: string, vendors: [{ name, needs_key: bool, required_env: string[] }] }`.
- Used by the pre-call dropdown to list all nine STT vendors with their key requirements.

### `POST /startAgent`

- Body: `{ channelName: string, rtcUid: int, userUid: int, vendor?: string, parameters?: object }`.
  - `vendor`: optional STT vendor name override (defaults to `STT_VENDOR` / `deepgram`).
  - `parameters.output_audio_codec?: string` is the only honored parameter field.
- Returns `data`: `{ agent_id, channel_name, vendor: string, status: "started" }`.
- 400 if a BYO vendor is selected but missing its credentials, or if `channelName`/`rtcUid`/`userUid` are invalid.

### `POST /stopAgent`

- Body: `{ agentId: string }`.
- Returns `{ code: 0, msg: "success" }` (no `data`).

## Response envelope

```json
{ "code": 0, "msg": "success", "data": { } }
```

`data` omitted when the route has no payload. Non-zero `code` or missing `data` = error on the client side.

## Rewrite map (`web/next.config.ts`)

| Browser path        | Backend destination |
| ------------------- | ------------------- |
| `/api/get_config`   | `/get_config`       |
| `/api/vendors`      | `/vendors`          |
| `/api/startAgent`   | `/startAgent`       |
| `/api/stopAgent`    | `/stopAgent`        |

`rewrites()` returns `[]` when `AGENT_BACKEND_URL` is unset. The contract is asserted by `verify-api-contracts.ts` and exercised by `verify-local-proxy.ts`.

## Browser API client (`web/src/services/api.ts`)

- `getConfig({ channel?, uid? }) → GetConfigResponse`
- `getVendors() → { default: string, vendors: VendorOption[] }`
- `startAgent(channelName, rtcUid, userUid, vendor?) → agent_id`
- `stopAgent(agentId) → void`

## Environment variables

| Variable                | Scope              | Required | Default              |
| ----------------------- | ------------------ | :------: | -------------------- |
| `AGORA_APP_ID`          | backend            |    ✅    | —                    |
| `AGORA_APP_CERTIFICATE` | backend            |    ✅    | —                    |
| `STT_VENDOR`            | backend            |          | `deepgram`           |
| `STT_MODEL`             | backend            |          | per-vendor           |
| `STT_LANGUAGE`          | backend            |          | per-vendor           |
| `AGENT_GREETING`        | backend            |          | built-in line        |
| _vendor creds_          | backend            | ✅\*     | — (BYO only)         |
| `AGENT_BACKEND_URL`     | web (deploy)       |    ✅\*  | `http://localhost:8000` (dev) |
| `PORT`                  | backend (env only) |          | `8000` — do **not** put in `.env.example` |

\* `AGENT_BACKEND_URL` required wherever the web app is deployed; rewrites are empty without it.
\* Vendor creds required only for the selected BYO vendor (see `required_env(STT_VENDOR)` or [stt_vendor_matrix](L2/stt_vendor_matrix.md)).

## Vendor registry API (`server/src/vendors.py`)

| Function | Signature | Returns |
| -------- | --------- | ------- |
| `available()` | `() → List[str]` | Sorted list of all vendor names |
| `required_env(name)` | `(str) → List[str]` | Credential env vars required for that vendor |
| `needs_key(name)` | `(str) → bool` | True if the vendor requires BYO credentials |
| `build_vendor(name, env?)` | `(str, dict?) → STT vendor` | Builds and returns the vendor; raises `ValueError` if creds missing |

## Cascade fixed contracts (`server/src/agent.py`)

| Component | Fixed value |
| --------- | ----------- |
| LLM | `OpenAI(model="gpt-4o-mini")` |
| TTS | `MiniMaxTTS(model="speech_2_6_turbo", voice_id="English_captivating_female1")` |
| `audio_scenario` | `"chorus"` |
| `data_channel` | `"rtm"` |
| `enable_error_message` | `True` |
| `enable_metrics` | `True` |
| `advanced_features` | `{"enable_rtm": True}` |

## Related Deep Dives

- [stt_vendor_matrix](L2/stt_vendor_matrix.md) — every vendor's creds, defaults, and SDK constructor.
