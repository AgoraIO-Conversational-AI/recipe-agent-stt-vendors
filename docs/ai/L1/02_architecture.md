# 02 · Architecture

> Two co-located processes. The browser talks only to Next.js `/api/*`, which rewrites to the FastAPI agent backend. The backend owns Agora tokens, the agent session, and the STT vendor switchboard. The LLM and TTS legs stay on keyless managed configs.

## Topology

```
Browser (localhost:3000)
  │  fetch /api/*
  ▼
Next.js (web/)  ──rewrite──▶  Agent backend (server/, :8000)
                                 │  builds cascading pipeline:
                                 │    stt = build_vendor(STT_VENDOR)   ← switchboard
                                 │    llm = OpenAI(gpt-4o-mini)        ← fixed, managed
                                 │    tts = MiniMaxTTS(speech_2_6_turbo) ← fixed, managed
                                 ▼
                              Agora ConvoAI Cloud
                                 │  user speech → <STT_VENDOR> (default Deepgram, keyless)
                                 │  text → OpenAI gpt-4o-mini
                                 │  reply → MiniMaxTTS → user's channel
                                 │  RTM events → browser:
                                 │    AGENT_STATE_CHANGED, AGENT_METRICS, AGENT_ERROR,
                                 │    MESSAGE_ERROR, TRANSCRIPT_UPDATED
                                 ▼
                              EventTimeline + annotated transcript in the web UI
```

- **`web/`** — Next.js 16 / React 19 / TypeScript. Owns UI, RTC/RTM client lifecycle, EventTimeline, and vendor dropdown. Calls only `/api/*`.
- **`server/`** — Python FastAPI (:8000). Owns Agora token generation, agent session lifecycle, and the STT vendor registry. SDK: `agora-agents>=2.3.0` (`import agora_agent`).
- No `llm/` service — LLM and TTS are Agora-managed (no extra API keys on default config).

## Request lifecycle

1. Browser `GET /api/get_config` → Next rewrites to backend `/get_config`; backend mints a Token007 from `AGORA_APP_ID` + `AGORA_APP_CERTIFICATE` and returns channel + UIDs. Always works key-less.
2. Browser `GET /api/vendors` → returns the nine-vendor list with `needs_key` flags. The pre-call dropdown uses this.
3. Browser joins the RTC channel, then `POST /api/startAgent` with optional `vendor` field; backend calls `build_vendor(selected)` — validates BYO creds here — and starts the cascading agent session.
4. Agora routes user audio to the selected STT; text goes to OpenAI gpt-4o-mini; reply TTS goes back into the channel.
5. RTM delivers state, metric, error, and transcript events to the web UI. The EventTimeline appends a `TimelineEvent` for each (capped at 50).
6. `POST /api/stopAgent { agentId }` ends the session.

## The vendor switchboard

`server/src/vendors.py` is the core of this recipe:

- `REGISTRY` maps each `STT_VENDOR` string to `(builder_fn, [required_env_vars])` — nine entries covering all A4.1 STT providers.
- `build_vendor(name, env)` looks up the builder, checks for missing creds (raises a clear `ValueError` naming each missing var), and calls the builder. Construction never fails on a missing SDK field.
- `available()` / `required_env(name)` / `needs_key(name)` expose the registry for the `/vendors` route and tests.
- Entries with empty creds (`deepgram`, `ares`) are keyless. All others require BYO API keys.
- The framework code is identical across sibling vendor recipes; only `CATEGORY` and `REGISTRY` differ.

## Key abstractions

- **`Agent`** (`server/src/agent.py`) — async wrapper around `AgoraAgent`. Reads `STT_VENDOR` in `__init__` (no validation). Calls `build_vendor` in `start()`, builds the full cascade, and owns the in-memory `_sessions` map keyed by `agent_id`.
- **`REGISTRY`** (`server/src/vendors.py`) — the switchboard. Each `build_<vendor>(env)` is a standalone, copy-pasteable sample of the real SDK constructor.
- **Rewrite proxy** (`web/next.config.ts`) — the only browser→backend boundary; no Next Route Handlers exist for agent/token logic.

## VAD config

The cascading recipe uses a top-level `turn_detection` on `AgoraAgent(...)` (unlike the realtime MLLM recipe). Config is:
- `speech_threshold: 0.5`
- `start_of_speech: mode=vad` with `interrupt_duration_ms=160`, `prefix_padding_ms=300`
- `end_of_speech: mode=vad` with `silence_duration_ms=480`

## Tech decisions

- **Rewrites, not Route Handlers** — hides backend placement behind `/api/*` so the same client works locally and deployed (set `AGENT_BACKEND_URL`).
- **Creds validated at `start()`** — server boots without BYO creds; `/startAgent` returns 400 listing the missing env vars only when a BYO vendor is selected without its credentials.
- **Zero-key default** — the `deepgram` vendor is Agora-managed; no STT key is needed to try the recipe.
- **In-UI vendor switcher** — the pre-call dropdown calls `/api/vendors` and passes the chosen vendor in `POST /api/startAgent`; no restart needed.

## Related Deep Dives

- [stt_vendor_matrix](L2/stt_vendor_matrix.md) — full nine-vendor matrix with SDK constructors and cred details.
- [session_lifecycle](L2/session_lifecycle.md) — browser orchestration of config + start/stop, RTC/RTM, EventTimeline wiring.
