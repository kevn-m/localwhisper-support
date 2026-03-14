# LocalWhisper Server API Contract

The app's "Server" STT mode sends audio to a user-hosted transcription server. The contract follows the **OpenAI Audio Transcriptions API** so any compatible backend works out of the box (faster-whisper, mlx-whisper, Groq, OpenAI, etc.).

An optional **LLM enhancement** step post-processes transcriptions via the **OpenAI Chat Completions API** (Ollama, Groq, OpenAI, any compatible endpoint).

---

## STT — Speech-to-Text

### Endpoint

```
POST {serverURL}/v1/audio/transcriptions
```

`serverURL` defaults to `http://localhost:8000`. The user configures this in Settings → Server.

### Request

- **Content-Type:** `multipart/form-data`
- **Fields:**

| Field   | Type          | Required | Description                                   |
|---------|---------------|----------|-----------------------------------------------|
| `file`  | binary (WAV)  | Yes      | 16 kHz, 16-bit, mono PCM WAV                  |
| `model` | string        | No       | Sent as `whisper-1` (ignored by most backends) |

### Response

```json
{
  "text": "The transcribed text."
}
```

- **Status 200** — success, body contains `text` (string).
- **Status 503** — model not loaded.
- **Status 400** — invalid WAV (wrong bit depth, wrong channel count).
- **Status 500** — transcription failed.

### Audio format details

The app builds WAV in-memory from raw Float32 PCM frames captured at the device's native sample rate (typically 16 kHz). The WAV is:

- RIFF/WAVE container
- PCM format (format tag 1)
- 1 channel (mono)
- 16-bit signed integer samples
- Sample rate matches capture (16 kHz expected by Whisper models)

### Health check

```
GET {serverURL}/health
```

```json
{
  "status": "ok",
  "model": "mlx-community/whisper-small-mlx"
}
```

### Legacy endpoint

The reference server also exposes `POST /transcribe` (returns `{"text": "...", "language": "en"}`). The iOS client does **not** use this — it exists for backwards compatibility and direct testing.

---

## LLM — Post-Transcription Enhancement (Optional)

### Endpoint

```
POST {llmBaseURL}/chat/completions
```

`llmBaseURL` defaults to `https://api.groq.com/openai/v1`. Configurable in Settings → Server. Works with Ollama (`http://<host>:11434/v1`), Groq, OpenAI, or any OpenAI-compatible API.

### Request

```json
{
  "model": "<llmModel>",
  "messages": [
    {
      "role": "system",
      "content": "<system prompt from enhancement style>"
    },
    {
      "role": "user",
      "content": "```\n<raw transcription>\n```"
    }
  ]
}
```

- **Content-Type:** `application/json`
- **Authorization:** `Bearer <llmAPIKey>` (stored in Keychain, configured in Settings)
- **Timeout:** 15s request / 60s resource

### Response

Standard OpenAI chat completion:

```json
{
  "choices": [
    {
      "message": {
        "role": "assistant",
        "content": "The cleaned-up text."
      }
    }
  ]
}
```

The app uses `choices[0].message.content` as the final result. On any failure, the raw transcription is returned unchanged (graceful degradation).

### User-configurable fields

| Setting        | Key in Settings | Default                              |
|----------------|-----------------|--------------------------------------|
| Server URL     | `serverURL`     | `http://localhost:8000`              |
| LLM Base URL   | `llmBaseURL`    | `https://api.groq.com/openai/v1`    |
| LLM Model      | `llmModel`      | `llama-3.3-70b-versatile`           |
| LLM API Key    | `llmAPIKey`     | *(empty — stored in Keychain)*       |

---

## Reference Server Implementations

Both live in `Server/` in the repo:

| File                | Backend          | Hardware     | Notes                                |
|---------------------|------------------|-------------|--------------------------------------|
| `server_native.py`  | mlx-whisper      | Apple Metal  | Daily driver. `WHISPER_MODEL` env var. |
| `server.py`         | faster-whisper   | CPU (Docker) | `docker compose up --build`          |

Both implement the same API contract above. Any server that accepts multipart WAV at `/v1/audio/transcriptions` and returns `{"text": "..."}` is compatible.

---

## Pipeline

```
Mic → Float32 PCM → WAV (in-memory) → POST /v1/audio/transcriptions → raw text
                                                                         ↓
                                                              [User Dictionary]
                                                                         ↓
                                                              [LLM Enhancement]
                                                                         ↓
                                                                   final text
```
