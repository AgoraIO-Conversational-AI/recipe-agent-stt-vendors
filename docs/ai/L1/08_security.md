# 08 · Security

> Trust boundaries, secret handling, and auth for the STT vendors recipe.

## Trust boundaries

| Hop                              | Auth                                                                              |
| -------------------------------- | --------------------------------------------------------------------------------- |
| Browser → agent backend          | None in local dev (the `/api/*` rewrite is same-origin).                          |
| Agent backend → Agora cloud      | Token007, generated from `AGORA_APP_ID` + `AGORA_APP_CERTIFICATE`.                |
| Agora cloud → STT vendor         | Keyless for `deepgram`/`ares` (Agora-managed). BYO creds for all other vendors — passed from the backend environment, never from the browser. |
| Agora cloud → OpenAI (LLM/TTS)   | Managed by Agora (no BYO key for the fixed LLM or TTS legs in this recipe).       |

## Secret handling

- **Server-only secrets:** `AGORA_APP_CERTIFICATE` and all BYO STT vendor credentials (e.g. `ASSEMBLYAI_API_KEY`, `AZURE_SPEECH_KEY`) live only in `server/.env.local` and never reach the browser.
- The browser receives a short-lived Token007 from `/get_config`, never the certificate or vendor API keys.
- `server/.env.local` is gitignored; `server/.env.example` ships placeholder stanzas only.
- Tokens (`generate_convo_ai_token`) expire after 3600s and are minted per `get_config` call for a concrete non-zero UID.

## Keyless default

The default `deepgram` vendor is Agora-managed: no STT API key is needed. The entire recipe runs key-less for evaluation. When a BYO vendor is selected and its credentials are absent, the server returns a clear 400 error naming the missing env vars — it does not fail silently or leak partial credentials.

## CORS

The backend sets `CORSMiddleware` with `allow_origins=["*"]` — open by design for a local/dev recipe. **Lock this down to known origins before any production deployment.**

## Validation

- `Agent.start()` rejects empty `channel_name` and non-positive `agent_uid`/`user_uid` before issuing tokens or starting a session.
- `build_vendor(name, env)` raises `ValueError` listing missing credential env vars; the route maps this to HTTP 400.
- Route errors are sanitized: `_log_route_error` logs only non-`None` context; SDK exceptions map to 400/500 without leaking internals to the client beyond the message.

## Deployment notes

- Set `AGENT_BACKEND_URL` only to a backend you control; the rewrite forwards browser requests there verbatim.
- The published Docker image is **backend-only** (`:8000`); it does not bundle secrets.
- On the default `deepgram` vendor no extra credentials are needed in the deployed backend image.

## Related Deep Dives

- None.
