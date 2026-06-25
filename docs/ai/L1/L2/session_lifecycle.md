# Deep Dive — Session Lifecycle

**When to Read This:** You are touching client-side join, vendor selection, token renewal, RTC/RTM wiring, transcript handling, EventTimeline wiring, or mid-call control. For the contracts these calls hit, see [06_interfaces](../06_interfaces.md).

The browser owns the full RTC/RTM client lifecycle; the backend owns tokens, the STT vendor selection, and the agent session. The two meet only at `/api/*`.

## End-to-end flow

1. **Vendor list** — `LandingPage.tsx` calls `getVendors()` on mount → `GET /api/vendors`. Backend returns `{ default, vendors: [{name, needs_key, required_env}] }`. The pre-call dropdown is populated from this.
2. **Config** — `handleStartConversation()` calls `getConfig()` → `GET /api/get_config`. Backend mints a Token007 (RTC+RTM, 3600s) for a concrete non-zero UID and returns `{ app_id, token, uid, channel_name, agent_uid }`.
3. **Start agent + RTM login in parallel** — `Promise.all` runs `startAgent(channelName, agentUid, userUid, selectedVendor)` (→ `POST /api/startAgent`) concurrently with RTM client login and channel subscribe.
4. **Join** — `ConversationComponent.tsx` joins the RTC channel with the returned token/UID and publishes the microphone.
5. **Converse** — user audio flows to the selected STT vendor; text goes to OpenAI gpt-4o-mini; reply TTS goes back into the channel. RTM delivers all events.
6. **EventTimeline wiring** — the web client uses `AgoraVoiceAI` from `agora-agent-client-toolkit` to subscribe to RTM events. Each event is mapped to a `TimelineEvent` and appended to the buffer (capped at 50). `EventTimeline.tsx` renders them reverse-chronologically.
7. **Stop** — `stopAgent(agentId)` → `POST /api/stopAgent`. The client also releases RTC/RTM media on end-call via `handleEndConversation`.

## EventTimeline events

| SDK event | `TimelineEvent.kind` | What it carries |
| --- | --- | --- |
| `AGENT_STATE_CHANGED` | `state` | `listening`, `thinking`, `speaking`, `idle` |
| `AGENT_METRICS` | `metric` | Stage type, metric name, value (ms) |
| `AGENT_ERROR` | `error` | Error type + message |
| `MESSAGE_ERROR` | `error` | RTM message error code + message |
| `TRANSCRIPT_UPDATED` | `turn` | Role (agent/user) + text snippet |

The `TimelineEvent` type is exported from `web/src/components/EventTimeline.tsx`. Import it from there.

## Backend session bookkeeping

`Agent` (`server/src/agent.py`) keeps an in-memory map `self._sessions[agent_id] = session`.

- `stop(agent_id)` pops the session and calls `session.stop()`.
- If the session is missing (e.g. process restarted), it falls back to `self.client.stop_agent(agent_id)` — the stateless cloud path. This makes stop robust across restarts but `_sessions` itself is **not** a durable store.

## Vendor selection per call

The per-call `vendor` parameter (from the UI dropdown or the `POST /startAgent` body) overrides the `STT_VENDOR` env default: `selected = (vendor or self.vendor).strip()`. The response includes `data.vendor` confirming which vendor was used.

## Transcript handling (`web/src/lib/conversation.ts`)

- `normalizeTranscript(transcript, localUid)` — maps `uid === '0'` to the local UID and runs `normalizeTranscriptSpacing` on text.
- `normalizeTimestampMs(ts)` — promotes second-precision timestamps to ms.
- `getMessageList` / `getCurrentInProgressMessage` — split finalized vs in-progress turns (by `TurnStatus.IN_PROGRESS`).
- `mapAgentVisualizerState(agentState, isConnected, connectionState)` — maps SDK state → UIKit visualizer state (`joining`, `listening`, `analyzing`, `talking`, `ambient`, `disconnected`).

## Token renewal

Tokens expire at 3600s. `handleTokenWillExpire` in `LandingPage.tsx` calls `getConfig` twice in parallel (once for the RTC UID, once for the RTM UID) and returns `{ rtcToken, rtmToken }`. Keep renewal client-side — the backend stays stateless about who is connected.

## What stays where

- **Client owns:** vendor dropdown state, RTC join, mic publish, RTM login, transcript/metrics/state listeners, EventTimeline buffer, token renewal, explicit end-call media release.
- **Backend owns:** token minting, vendor registry, STT build and validation, session start/stop.
- Do not move token or vendor logic into the web app or add Route Handlers for it (see [07_gotchas](../07_gotchas.md)).

## Related L1

- [02_architecture](../02_architecture.md) · [03_code_map](../03_code_map.md) · [06_interfaces](../06_interfaces.md)
