# Deep Dive — STT Vendor Matrix

**When to Read This:** You are adding or changing an STT vendor, choosing which vendor to wire in a fork, or verifying that an SDK constructor matches the `REGISTRY` entry. For the high-level switchboard picture, start at [02_architecture](../02_architecture.md).

All nine vendors in `server/src/vendors.py` are listed here with their SDK class, required credentials, default config, and the copy-pasteable builder from the source.

## Registry overview

```
CATEGORY = "STT"

REGISTRY: Dict[str, Tuple[Callable, List[str]]] = {
    "deepgram":     (build_deepgram,     []),                          # 🟢 keyless
    "ares":         (build_ares,         []),                          # 🟢 keyless
    "assemblyai":   (build_assemblyai,   ["ASSEMBLYAI_API_KEY"]),
    "speechmatics": (build_speechmatics, ["SPEECHMATICS_API_KEY"]),
    "openai":       (build_openai,       ["OPENAI_STT_API_KEY"]),
    "microsoft":    (build_microsoft,    ["AZURE_SPEECH_KEY", "AZURE_SPEECH_REGION"]),
    "google":       (build_google,       ["GOOGLE_APPLICATION_CREDENTIALS_JSON",
                                          "GOOGLE_PROJECT_ID", "GOOGLE_LOCATION"]),
    "amazon":       (build_amazon,       ["AWS_ACCESS_KEY_ID", "AWS_SECRET_ACCESS_KEY",
                                          "AWS_REGION"]),
    "sarvam":       (build_sarvam,       ["SARVAM_API_KEY"]),
}
```

`STT_MODEL` overrides the model field on vendors that expose one (all except `ares`).

## Vendor-by-vendor reference

### `deepgram` — Agora-managed, keyless

| Field | Value |
| ----- | ----- |
| SDK class | `DeepgramSTT` |
| Required creds | _none_ (Agora-managed) |
| Default model | `nova-3` |
| Default language | `en` |
| `STT_MODEL` applies | yes |

```python
DeepgramSTT(model=_model(env, "nova-3"), language="en")
```

### `ares` — Agora-managed, keyless

| Field | Value |
| ----- | ----- |
| SDK class | `AresSTT` |
| Required creds | _none_ (Agora-managed) |
| Default model | SDK default |
| `STT_MODEL` applies | no (no model field) |

```python
AresSTT()
```

### `assemblyai`

| Field | Value |
| ----- | ----- |
| SDK class | `AssemblyAISTT` |
| Required creds | `ASSEMBLYAI_API_KEY` |
| Default language | `en` |
| `STT_MODEL` applies | no |

```python
AssemblyAISTT(
    api_key=env["ASSEMBLYAI_API_KEY"],
    language="en",
)
```

### `speechmatics`

| Field | Value |
| ----- | ----- |
| SDK class | `SpeechmaticsSTT` |
| Required creds | `SPEECHMATICS_API_KEY` |
| Default language | `en` |
| `STT_MODEL` applies | no |

```python
SpeechmaticsSTT(
    api_key=env["SPEECHMATICS_API_KEY"],
    language="en",
)
```

### `openai`

| Field | Value |
| ----- | ----- |
| SDK class | `OpenAISTT` |
| Required creds | `OPENAI_STT_API_KEY` |
| Default model | `gpt-4o-transcribe` |
| Default language | `en` |
| `STT_MODEL` applies | yes |

> Note: `OPENAI_STT_API_KEY` is distinct from `OPENAI_API_KEY` used by the realtime recipe. They may share a value but are separate env vars here.

```python
OpenAISTT(
    api_key=env["OPENAI_STT_API_KEY"],
    model=_model(env, "gpt-4o-transcribe"),
    prompt="Transcribe the audio.",
    language="en",
)
```

### `microsoft`

| Field | Value |
| ----- | ----- |
| SDK class | `MicrosoftSTT` |
| Required creds | `AZURE_SPEECH_KEY`, `AZURE_SPEECH_REGION` |
| Default language | `en-US` |
| `STT_MODEL` applies | no |

```python
MicrosoftSTT(
    key=env["AZURE_SPEECH_KEY"],
    region=env["AZURE_SPEECH_REGION"],
    language="en-US",
)
```

### `google`

| Field | Value |
| ----- | ----- |
| SDK class | `GoogleSTT` |
| Required creds | `GOOGLE_APPLICATION_CREDENTIALS_JSON`, `GOOGLE_PROJECT_ID`, `GOOGLE_LOCATION` |
| Default language | `en-US` |
| `STT_MODEL` applies | no |

> `GOOGLE_APPLICATION_CREDENTIALS_JSON` is the full service account JSON string (not a file path).

```python
GoogleSTT(
    adc_credentials_string=env["GOOGLE_APPLICATION_CREDENTIALS_JSON"],
    project_id=env["GOOGLE_PROJECT_ID"],
    location=env["GOOGLE_LOCATION"],
    language="en-US",
)
```

### `amazon`

| Field | Value |
| ----- | ----- |
| SDK class | `AmazonSTT` |
| Required creds | `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION` |
| Default language | `en-US` |
| `STT_MODEL` applies | no |

```python
AmazonSTT(
    access_key=env["AWS_ACCESS_KEY_ID"],
    secret_key=env["AWS_SECRET_ACCESS_KEY"],
    region=env["AWS_REGION"],
    language="en-US",
)
```

### `sarvam`

| Field | Value |
| ----- | ----- |
| SDK class | `SarvamSTT` |
| Required creds | `SARVAM_API_KEY` |
| Default language | `en-IN` |
| `STT_MODEL` applies | no |

```python
SarvamSTT(
    api_key=env["SARVAM_API_KEY"],
    language="en-IN",
)
```

## Summary table

| `STT_VENDOR` | SDK class | Creds required | Keyless | `STT_MODEL` |
| ------------ | --------- | -------------- | :-----: | :---------: |
| `deepgram` | `DeepgramSTT` | — | 🟢 | yes |
| `ares` | `AresSTT` | — | 🟢 | no |
| `assemblyai` | `AssemblyAISTT` | `ASSEMBLYAI_API_KEY` | | no |
| `speechmatics` | `SpeechmaticsSTT` | `SPEECHMATICS_API_KEY` | | no |
| `openai` | `OpenAISTT` | `OPENAI_STT_API_KEY` | | yes |
| `microsoft` | `MicrosoftSTT` | `AZURE_SPEECH_KEY`, `AZURE_SPEECH_REGION` | | no |
| `google` | `GoogleSTT` | `GOOGLE_APPLICATION_CREDENTIALS_JSON`, `GOOGLE_PROJECT_ID`, `GOOGLE_LOCATION` | | no |
| `amazon` | `AmazonSTT` | `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION` | | no |
| `sarvam` | `SarvamSTT` | `SARVAM_API_KEY` | | no |

## How `build_vendor` works

```python
def build_vendor(name: str, env: Optional[Dict[str, str]] = None):
    env = env if env is not None else os.environ
    if name not in REGISTRY:
        raise ValueError(f"unknown STT vendor '{name}'; choose one of {available()}")
    builder, required = REGISTRY[name]
    missing = [var for var in required if not env.get(var)]
    if missing:
        raise ValueError(
            f"STT vendor '{name}' requires environment variable(s): {', '.join(missing)}"
        )
    return builder(env)
```

The error message names every missing var — this is the UX for the 400 error that surfaces in the browser.

## Related L1

- [02_architecture](../02_architecture.md) · [04_conventions](../04_conventions.md) · [06_interfaces](../06_interfaces.md)
