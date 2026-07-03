# Lab 03 — Generative AI Speech | Exam Notes

## Two Apps in This Lab
1. **generate-speech** → Text-to-Speech (TTS) using `gpt-4o-mini-tts`
2. **transcribe-speech** → Speech-to-Text (STT) using `gpt-4o-mini-transcribe`

---

## Service & SDK
- **SDK Package**: `openai` (standard OpenAI SDK)
- **Client Class**: `AzureOpenAI`
- **Endpoint**: **Target URI** from model deployment page in Foundry portal
- **Endpoint Format**: `https://{resource}.services.ai.azure.com` *(base URL, no project path)*

---

## Authentication
```python
from azure.identity import DefaultAzureCredential, get_bearer_token_provider

token_provider = get_bearer_token_provider(
    DefaultAzureCredential(), "https://ai.azure.com/.default"
)

client = AzureOpenAI(
    azure_endpoint=endpoint,
    azure_ad_token_provider=token_provider,
    api_version="2025-01-01-preview"
)
```
- Token audience: `"https://ai.azure.com/.default"` (for `services.ai.azure.com` endpoints)
- Uses `get_bearer_token_provider` — wraps credential for OpenAI SDK

---

## App 1: Text-to-Speech (TTS)

### Full API Call
```python
with client.audio.speech.with_streaming_response.create(
    model=model_deployment,
    voice="alloy",
    input="My voice is my passport!",
    instructions="Speak in a serious tone.",
) as response:
    response.stream_to_file(speech_file_path)
```

### Parameters Explained — ALL OPTIONS

| Parameter | Required | What It Does | Available Options |
|-----------|----------|--------------|-------------------|
| `model` | ✅ Yes | The TTS model deployment name | `gpt-4o-mini-tts` (or your deployment name) |
| `voice` | ✅ Yes | The voice persona/character | `alloy` `echo` `fable` `onyx` `nova` `shimmer` |
| `input` | ✅ Yes | The text to convert to speech | Any string (max 4096 chars) |
| `instructions` | ❌ Optional | Tone/style/character instructions for the model | Any string — e.g. `"Speak slowly"`, `"Use a British accent"`, `"Sound excited"` |
| `response_format` | ❌ Optional | Output audio file format | `mp3` (default) · `opus` · `aac` · `flac` · `wav` · `pcm` |
| `speed` | ❌ Optional | Playback speed of audio | `0.25` to `4.0` (default: `1.0`) |

### Voice Options — What Each Sounds Like
| Voice | Character |
|-------|-----------|
| `alloy` | Neutral, balanced — good for general use |
| `echo` | Male, deep, clear |
| `fable` | Expressive, storytelling |
| `onyx` | Deep, authoritative |
| `nova` | Warm, friendly female |
| `shimmer` | Soft, gentle female |

### Streaming vs Non-Streaming
```python
# Streaming (used in lab — memory efficient)
with client.audio.speech.with_streaming_response.create(...) as response:
    response.stream_to_file(path)

# Non-streaming (loads full audio into memory first)
response = client.audio.speech.create(...)
response.write_to_file(path)
```

### Play the Audio File
```python
from playsound3 import playsound
playsound(speech_file_path)
```

---

## App 2: Speech-to-Text (Transcription)

### Full API Call
```python
audio_file = open(file_path, "rb")   # open in binary read mode
transcription = client.audio.transcriptions.create(
    model=model_deployment,
    file=audio_file,
    response_format="text"
)
print(transcription)
```

### Parameters Explained — ALL OPTIONS

| Parameter | Required | What It Does | Available Options |
|-----------|----------|--------------|-------------------|
| `model` | ✅ Yes | The STT model deployment name | `gpt-4o-mini-transcribe` |
| `file` | ✅ Yes | Audio file to transcribe | Open file in `"rb"` (binary read) mode |
| `response_format` | ❌ Optional | Output format of transcript | `text` · `json` · `srt` · `verbose_json` · `vtt` |
| `language` | ❌ Optional | Language hint (improves accuracy) | ISO-639-1 code: `"en"`, `"fr"`, `"de"`, `"es"` |
| `prompt` | ❌ Optional | Context hint for the model | Any string — helps with proper nouns/spelling |
| `temperature` | ❌ Optional | Sampling randomness | `0` to `1` (default: `0`) |

### Response Format Options
| Format | What You Get |
|--------|-------------|
| `text` | Plain string — just the words |
| `json` | `{"text": "..."}` object |
| `srt` | Subtitle format with timestamps |
| `vtt` | WebVTT subtitle format |
| `verbose_json` | Full detail: segments, words, timestamps, language, duration |

---

## Endpoint Format — Critical Difference

| Lab | Client | Endpoint |
|-----|--------|----------|
| Lab 01 | `TextAnalyticsClient` | Resource URL (no project path) |
| Lab 02 | `AIProjectClient` | Full project URL (`/api/projects/{name}`) |
| **Lab 03** | **`AzureOpenAI`** | **Target URI from model deployment page** |

---

## Exam Key Facts

| Fact | Answer |
|------|--------|
| SDK package | `openai` |
| Client class | `AzureOpenAI` |
| TTS model | `gpt-4o-mini-tts` |
| STT model | `gpt-4o-mini-transcribe` |
| Token audience | `https://ai.azure.com/.default` |
| TTS API method | `client.audio.speech.with_streaming_response.create()` |
| STT API method | `client.audio.transcriptions.create()` |
| Default audio format | `mp3` |
| File must be opened as | `"rb"` (binary read) for transcription |
| Speed range | `0.25` to `4.0` (default `1.0`) |
| `instructions=` | Only in TTS — controls tone/style |

---

## Exam Traps
- `with_streaming_response.create()` ≠ `.create()` — streaming is memory efficient
- File for transcription must be opened in **binary** mode: `open(path, "rb")`
- `instructions=` is **TTS only** — transcription doesn't have it
- Endpoint for `AzureOpenAI` = **Target URI** from deployment page, NOT the project endpoint
- `response_format="text"` returns a plain string; `"json"` returns an object
- `get_bearer_token_provider` wraps the credential — OpenAI SDK doesn't use credential directly

---

## Models Summary

| Model | Purpose | API Method |
|-------|---------|------------|
| `gpt-4o-mini-tts` | Text → Speech | `audio.speech.create()` |
| `gpt-4o-mini-transcribe` | Speech → Text | `audio.transcriptions.create()` |
