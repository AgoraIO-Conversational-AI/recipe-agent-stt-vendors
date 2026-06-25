# Docs Test Results

**Date:** 2026-06-25
**Repo:** `AgoraIO-Conversational-AI/recipe-agent-stt-vendors`
**Branch:** `docs/progressive-disclosure`

---

## Structural Checks

| Check | Result |
| ----- | ------ |
| `docs/ai/L0_repo_card.md` exists | PASS |
| `docs/ai/L0_repo_card.md` ≤ 50 lines | PASS (36 lines) |
| `docs/ai/RECIPE.md` exists | PASS |
| `docs/ai/AGENTS.md` not overwritten (AGENTS.md at root updated) | PASS |
| All 8 L1 files present (01–08) | PASS |
| `docs/ai/L1/L2/_index.md` exists | PASS |
| L2 deep dives present: `stt_vendor_matrix.md`, `session_lifecycle.md` | PASS |
| AGENTS.md contains "## How to Load" section | PASS |
| AGENTS.md contains "## Git Conventions" section | PASS |
| AGENTS.md contains "## Doc Commands" section | PASS |
| AGENTS.md stale "docs/ai not present" note removed | PASS |
| L0 Recipe Role = `base` | PASS |
| L0 Recipe Version = `1.0.0` | PASS |
| L0 Recipe Status = `experimental` | PASS |
| CLAUDE.md left unchanged (redirects to @AGENTS.md) | PASS |

**Relative-link counts (internal links, no HTTP):** 27 unique relative references checked; all targets confirmed present.

---

## pytest Results

**Command:** `pytest tests -v` in `server/` (throwaway venv `/tmp/v_stt`, Python 3.14.4)
**Install:** `pip install -r requirements.txt -r requirements-dev.txt`

| Test | Result |
| ---- | ------ |
| `tests/test_agent_config.py::test_agent_constructs` | PASS |
| `tests/test_agent_construction.py::test_start_constructs_real_agent_and_returns_shape` | PASS |
| `tests/test_vendors.py::test_every_vendor_constructs_and_emits_config` | PASS |
| `tests/test_vendors.py::test_byo_vendor_missing_creds_raises` | PASS |

**4 passed in 0.60s.** Venv removed (`rm -rf /tmp/v_stt`).

---

## Q&A — 12 Questions Across 5 Categories

### Category 1: Identity & Role

**Q1.** What is the repo identifier and recipe role?
**A:** `AgoraIO-Conversational-AI/recipe-agent-stt-vendors`, Recipe Role `base`.
**Source:** `docs/ai/L0_repo_card.md` Identity table.
**Verdict:** PASS — matches `AGENTS.md` and `RECIPE.md` frontmatter.

**Q2.** What language/runtime does the backend use?
**A:** Python 3.10+ (FastAPI + uvicorn), SDK `agora-agents>=2.3.0`.
**Source:** `server/requirements.txt` (`fastapi`, `uvicorn`, `agora-agents>=2.3.0`); `server/src/agent.py` imports.
**Verdict:** PASS

---

### Category 2: Setup & Configuration

**Q3.** What is the default STT vendor and is an API key required to run it?
**A:** `deepgram` (Agora-managed); no API key required. The recipe is zero-key by default.
**Source:** `server/src/agent.py` line 47: `self.vendor = os.getenv("STT_VENDOR", "deepgram")`; `server/src/vendors.py` `REGISTRY`: `"deepgram": (build_deepgram, [])`.
**Verdict:** PASS

**Q4.** What must you never add to `server/.env.example` and why?
**A:** `PORT`. `verify:local:fastapi` injects a random `PORT` via `load_dotenv(override=True)`; a `PORT` in `.env.example` (copied to `.env.local`) would clobber that injected port and break the smoke test.
**Source:** `web/scripts/verify-local-fastapi.ts` line 120–128 (PORT injection); `server/src/server.py` line 219 (`os.getenv("PORT", "8000")`).
**Verdict:** PASS — documented in `07_gotchas.md`.

**Q5.** What env vars does `STT_VENDOR=microsoft` require?
**A:** `AZURE_SPEECH_KEY` and `AZURE_SPEECH_REGION`.
**Source:** `server/src/vendors.py` `REGISTRY`: `"microsoft": (build_microsoft, ["AZURE_SPEECH_KEY", "AZURE_SPEECH_REGION"])`.
**Verdict:** PASS — confirmed against `REGISTRY` entry and `build_microsoft` function.

---

### Category 3: Architecture & Pipeline

**Q6.** What is the full STT→LLM→TTS pipeline (default config)?
**A:** `Deepgram STT (nova-3, Agora-managed, keyless)` → `OpenAI gpt-4o-mini (managed)` → `MiniMaxTTS (speech_2_6_turbo, English_captivating_female1)`.
**Source:** `server/src/agent.py` lines 83–86: `stt = build_vendor(selected)`, `llm = OpenAI(model="gpt-4o-mini")`, `tts = MiniMaxTTS(model="speech_2_6_turbo", voice_id="English_captivating_female1")`.
**Verdict:** PASS

**Q7.** Where does the STT vendor switchboard live and how does `build_vendor` work?
**A:** `server/src/vendors.py`. `build_vendor(name, env)` looks up `REGISTRY[name]`, checks for missing creds, and calls the matching `build_<vendor>(env)`. Raises `ValueError` naming missing vars.
**Source:** `server/src/vendors.py` lines 129–141.
**Verdict:** PASS

**Q8.** What is the `/api/vendors` route and what does it return?
**A:** `GET /vendors` (backend); rewritten from `/api/vendors` by Next.js. Returns `{ code: 0, data: { default: str, vendors: [{name, needs_key, required_env}] }, msg: "success" }`.
**Source:** `server/src/server.py` lines 144–160 (`list_vendors` handler); `web/next.config.ts` lines 36–38.
**Verdict:** PASS — correctly documented in `06_interfaces.md`.

---

### Category 4: Conventions & Invariants

**Q9.** Where should `TimelineEvent` be imported from in the web client?
**A:** `web/src/components/EventTimeline.tsx` — it is exported there directly. Do not create a duplicate in a separate types file.
**Source:** `web/src/components/EventTimeline.tsx` line 3: `export type TimelineEvent = { ... }`.
**Verdict:** PASS — documented in `03_code_map.md` and `04_conventions.md`.

**Q10.** Why are vendor credentials validated in `start()` rather than `__init__`?
**A:** So that `/get_config`, `doctor` checks, and the managed docker smoke remain key-less even when a BYO `STT_VENDOR` is configured. The server boots without BYO credentials; only an actual `startAgent` call triggers the check.
**Source:** `server/src/agent.py` lines 44–47 (comment: "No credential validation here — that happens in start()"); lines 77–83 (`build_vendor` call in `start()`).
**Verdict:** PASS

---

### Category 5: Security & Deployment

**Q11.** Which secrets must never reach the browser?
**A:** `AGORA_APP_CERTIFICATE` and all BYO STT vendor credential env vars (e.g. `ASSEMBLYAI_API_KEY`, `AZURE_SPEECH_KEY`). The browser receives only a short-lived Token007.
**Source:** `server/src/server.py` — credentials are read via `os.getenv` server-side and never serialized to client responses. `server/.env.example` and `server/.env.local` are gitignored.
**Verdict:** PASS — documented in `08_security.md`.

**Q12.** What does `POST /api/startAgent` return in its `data` field?
**A:** `{ agent_id, channel_name, vendor: str, status: "started" }`.
**Source:** `server/src/agent.py` lines 166–172: `return {"agent_id": agent_id, "channel_name": channel_name, "vendor": selected, "status": "started"}`.
**Verdict:** PASS — the `vendor` field in the response is specific to this recipe (not in the realtime recipe); documented in `06_interfaces.md`.

---

## Summary Table by Category

| Category | Questions | Passed | Failed |
| -------- | :-------: | :----: | :----: |
| Identity & Role | 2 | 2 | 0 |
| Setup & Configuration | 3 | 3 | 0 |
| Architecture & Pipeline | 3 | 3 | 0 |
| Conventions & Invariants | 2 | 2 | 0 |
| Security & Deployment | 2 | 2 | 0 |
| **Total** | **12** | **12** | **0** |

---

## Fix/Retest

No failures. No fixes required.
