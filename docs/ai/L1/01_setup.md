# 01 · Setup

> Install dependencies, configure env, and run the STT vendors recipe locally. Zero-key by default: the `deepgram` STT vendor is Agora-managed and requires no API key.

## Prerequisites

- Python 3.10+ (backend runs on 3.10 and 3.13 in CI)
- [Bun](https://bun.sh/) (runs the web app and orchestration scripts)
- [Agora CLI](https://github.com/AgoraIO/cli) (optional; easiest way to mint App ID + Certificate)
- No STT key required on the default `deepgram` vendor

## Install

```bash
bun run setup            # installs web deps + creates server/ venv from requirements.txt
```

`setup` runs `setup:env` (copies `server/.env.example` → `server/.env.local` if missing), `setup:server` (recreates `server/venv`, installs `requirements.txt`), and `setup:web` (`bun install`).

## Configure env

Backend env file is `server/.env.local` (template: `server/.env.example`).

| Variable                | Required | Default                    | Notes                                                                      |
| ----------------------- | :------: | -------------------------- | -------------------------------------------------------------------------- |
| `AGORA_APP_ID`          |    ✅    | —                          | Agora Console → Project → App ID                                           |
| `AGORA_APP_CERTIFICATE` |    ✅    | —                          | Agora Console → Project → App Certificate                                  |
| `STT_VENDOR`            |          | `deepgram`                 | Which STT vendor to build; see [06_interfaces](06_interfaces.md) for all 9 |
| `STT_MODEL`             |          | per-vendor                 | Optional model override (vendors with a model field)                       |
| `STT_LANGUAGE`          |          | per-vendor                 | Optional language hint (documented per vendor)                             |
| `AGENT_GREETING`        |          | built-in line              | Optional opening utterance override                                        |
| _vendor creds_          |          | —                          | Required only for BYO vendors; see `required_env(STT_VENDOR)` in vendors.py |

Fill Agora credentials via the CLI or by hand:

```bash
agora login
agora project use <your-project>
agora project env write server/.env.local   # writes App ID + Certificate
# For a BYO STT vendor, also add e.g. ASSEMBLYAI_API_KEY=... to server/.env.local
```

> Do **not** add `PORT` to `server/.env.example` — see [07_gotchas](07_gotchas.md).

## Run

```bash
bun run dev              # backend (:8000) + web (:3000) via concurrently
```

Open <http://localhost:3000> → pick STT vendor from the dropdown (default `deepgram`, keyless) → **Start Conversation** → speak. Watch the **Event Timeline** panel. Backend API docs at <http://localhost:8000/docs>.

## Quick commands

```bash
bun run doctor           # shared prereqs (bun + node_modules); no creds needed
bun run doctor:local     # + .env.local + AGORA_APP_ID/CERTIFICATE present
bun run verify           # web-only gate (doctor + api contracts + web build)
bun run verify:local     # full local gate: backend compile + fastapi smoke + proxy + web build
bun run clean            # remove venvs and build artifacts
```

Backend unit tests run standalone (no cloud, no creds):

```bash
cd server && pytest tests -v
```

## Related Deep Dives

- [stt_vendor_matrix](L2/stt_vendor_matrix.md) — all nine vendors with creds, defaults, and SDK constructors.
