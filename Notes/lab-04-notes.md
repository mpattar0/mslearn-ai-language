# Lab 04 — Azure Speech SDK (Voice Mail App) | Exam Notes

## Service & SDK
- **Service**: Azure AI Speech (in Foundry Tools) — dedicated speech service
- **SDK Package**: `azure-cognitiveservices-speech`
- **Import alias**: `import azure.cognitiveservices.speech as speech_sdk`
- **Endpoint format**: `https://{resource}.cognitiveservices.azure.com/` *(trailing slash required!)*

---

## Lab Scenario
Voice mail assistant with 3 options:
1. Record a greeting → TTS (Text → `.wav` file)
2. Transcribe messages → STT (`.wav` files → Text)
3. Exit

---

## Step 1 — SpeechConfig (Central Config Object)

```python
speech_config = speech_sdk.SpeechConfig(
    token_credential=credential,   # DefaultAzureCredential (Entra ID)
    endpoint=foundry_endpoint      # https://{resource}.cognitiveservices.azure.com/
)
```

### All SpeechConfig Constructor Parameters (from Microsoft Docs)

| Parameter | Required | Description |
|-----------|----------|-------------|
| `subscription` | ❌ | Subscription key (key-based auth) |
| `region` | ❌ | Region name (e.g. `"eastus"`) — used with subscription |
| `endpoint` | ❌ | Custom endpoint URL — used in lab |
| `host` | ❌ | Host address — format `"protocol://host:port"` |
| `auth_token` | ❌ | Authorization token string |
| `speech_recognition_language` | ❌ | BCP-47 language code (e.g. `"en-US"`) |
| `token_credential` | ❌ | AAD token (DefaultAzureCredential) — used in lab |
| `key_credential` | ❌ | AzureKeyCredential object |

> **Exam Rule**: When using `token_credential`, you MUST also set a custom `endpoint`. Region alone is not enough.

### SpeechConfig Auth Methods — Which to Use When

| Auth Method | Parameters Needed | Use Case |
|-------------|-------------------|----------|
| Key-based | `subscription` + `region` | Simple/quick setup |
| Passwordless | `token_credential` + `endpoint` | Production, RBAC, no key exposure ← **used in lab** |
| Token | `auth_token` + `region` | Short-lived tokens |
| AzureKeyCredential | `key_credential` + `endpoint` | Alternative to subscription string |

### SpeechConfig Key Attributes (set after creation)
```python
speech_config.speech_synthesis_voice_name = "en-US-Serena:DragonHDLatestNeural"
speech_config.speech_recognition_language = "en-US"
speech_config.speech_synthesis_language = "en-US"
speech_config.output_format  # Simple (default) or Detailed
```

### SpeechConfig Key Methods
```python
speech_config.set_speech_synthesis_output_format(SpeechSynthesisOutputFormat.Riff16Khz16BitMonoPcm)
speech_config.set_profanity(ProfanityOption.Masked)  # Masked / Removed / Raw
speech_config.request_word_level_timestamps()         # include timestamps in STT result
speech_config.enable_audio_logging()                  # log audio to storage
speech_config.enable_dictation()                      # continuous recognition dictation mode
```

---

## Step 2 — AudioOutputConfig (TTS — Where to save audio)

```python
audio_config = speech_sdk.audio.AudioOutputConfig(filename="greeting.wav")
```

### AudioOutputConfig Options

| Parameter | What It Does |
|-----------|-------------|
| `filename="greeting.wav"` | Save output to file |
| `use_default_speaker=True` | Play directly through speakers (no file) |
| `stream=audio_stream` | Output to custom stream |
| `None` | Drop audio (performance testing only) |

> **Exam**: `filename=` saves to file. `use_default_speaker=True` plays live. `None` discards output.

---

## Step 3 — SpeechSynthesizer (TTS Engine)

```python
speech_synthesizer = speech_sdk.SpeechSynthesizer(
    speech_config=speech_config,   # required
    audio_config=audio_config      # optional — None = drop audio, omit = use default speaker
)
```

### SpeechSynthesizer Constructor Parameters (from Microsoft Docs)

| Parameter | Required | Description |
|-----------|----------|-------------|
| `speech_config` | ✅ | The SpeechConfig object |
| `audio_config` | ❌ | AudioOutputConfig — omit = default speaker, None = drop audio |
| `auto_detect_source_language_config` | ❌ | For auto language detection |

### SpeechSynthesizer Methods — ALL Options

| Method | Blocking? | Input | Use Case |
|--------|-----------|-------|----------|
| `speak_text(text)` | ✅ Sync | Plain text | Simple, waits for finish |
| `speak_text_async(text).get()` | Async + wait | Plain text | **Used in lab** |
| `speak_ssml(ssml)` | ✅ Sync | SSML XML string | Fine-grained control |
| `speak_ssml_async(ssml).get()` | Async + wait | SSML XML string | Async SSML |
| `start_speaking_text(text)` | ✅ Sync | Plain text | Starts streaming immediately |
| `start_speaking_text_async(text)` | Async | Plain text | Async streaming |
| `stop_speaking()` | Sync | — | Stop ongoing synthesis |
| `stop_speaking_async()` | Async | — | Async stop |
| `get_voices_async(locale="").get()` | Async | BCP-47 locale | List available voices |

> **Exam**: `speak_text_async().get()` = async call but blocks until done. `speak_text()` = fully synchronous.

### TTS Result Check
```python
result = speech_synthesizer.speak_text_async(greeting_message).get()

if result.reason == speech_sdk.ResultReason.SynthesizingAudioCompleted:
    print("Success!")
else:
    print("Error:", result.reason)
    # Check result.cancellation_details for why it failed
```

---

## Step 4 — AudioConfig (STT — Which audio file to read)

```python
audio_config = speech_sdk.audio.AudioConfig(filename=file_path)
```

### AudioConfig Options

| Parameter | What It Does |
|-----------|-------------|
| `filename="path.wav"` | Read from audio file ← **used in lab** |
| `use_default_microphone=True` | Record from default microphone |
| `device_name="device_id"` | Record from specific microphone |
| `stream=audio_stream` | Read from custom audio stream |

> **Exam**: `AudioConfig` (input) vs `AudioOutputConfig` (output) — different classes!

---

## Step 5 — SpeechRecognizer (STT Engine)

```python
speech_recognizer = speech_sdk.SpeechRecognizer(
    speech_config=speech_config,
    audio_config=audio_config
)
```

### SpeechRecognizer Constructor Parameters

| Parameter | Required | Description |
|-----------|----------|-------------|
| `speech_config` | ✅ | The SpeechConfig object |
| `audio_config` | ❌ | AudioConfig — omit = default microphone |
| `language` | ❌ | BCP-47 language code: `"en-US"` |
| `source_language_config` | ❌ | SourceLanguageConfig object |
| `auto_detect_source_language_config` | ❌ | AutoDetect language config |

> **Exam**: Specify ONLY ONE of: `language`, `source_language_config`, or `auto_detect_source_language_config`.

### SpeechRecognizer Methods — ALL Options

| Method | Type | Use Case |
|--------|------|----------|
| `recognize_once()` | Sync | Recognize single utterance, blocking |
| `recognize_once_async().get()` | Async+wait | **Used in lab** — single utterance |
| `start_continuous_recognition()` | Sync | Start continuous recognition |
| `start_continuous_recognition_async()` | Async | Async continuous recognition |
| `stop_continuous_recognition()` | Sync | Stop continuous recognition |
| `stop_continuous_recognition_async()` | Async | Async stop |
| `start_keyword_recognition(model)` | Sync | Keyword spotting |
| `stop_keyword_recognition()` | Sync | Stop keyword spotting |

> **Exam**: `recognize_once_async()` = single utterance (stops at silence). Continuous recognition = keeps listening until stopped.

### STT Result Check
```python
result = speech_recognizer.recognize_once_async().get()

if result.reason == speech_sdk.ResultReason.RecognizedSpeech:
    print(result.text)          # the transcribed text
elif result.reason == speech_sdk.ResultReason.NoMatch:
    print("No speech detected")
elif result.reason == speech_sdk.ResultReason.Canceled:
    details = result.cancellation_details
    print(details.reason, details.error_details)
```

---

## ResultReason Enum — ALL Values (Exam Critical)

| Value | Meaning | Which Operation |
|-------|---------|-----------------|
| `RecognizedSpeech` | STT success — speech found and transcribed | STT ✅ |
| `NoMatch` | STT ran but no speech recognized | STT |
| `Canceled` | Operation canceled (auth error, network, etc.) | Both |
| `SynthesizingAudioCompleted` | TTS success — audio generated fully | TTS ✅ |
| `SynthesizingAudio` | TTS in progress | TTS |
| `SynthesizingAudioStarted` | TTS just started | TTS |
| `RecognizingKeyword` | Keyword recognition in progress | Keyword |
| `RecognizedKeyword` | Keyword recognized | Keyword |

---

## Voice Name Format
```
{language}-{Region}-{VoiceName}:{Model}
```
Example: `"en-US-Serena:DragonHDLatestNeural"`

| Part | Value | Meaning |
|------|-------|---------|
| `en-US` | Language-Region | BCP-47 locale |
| `Serena` | Voice name | The persona/voice identity |
| `DragonHDLatestNeural` | Model | Neural model type — HD quality |

### Common Neural Voice Types
| Type | Quality |
|------|---------|
| `Neural` | Standard neural voice |
| `DragonHDLatestNeural` | High definition — higher quality |
| `TurboMultilingualNeural` | Multilingual support |

---

## Output Format Options (SpeechSynthesisOutputFormat)
```python
speech_config.set_speech_synthesis_output_format(
    speech_sdk.SpeechSynthesisOutputFormat.Riff16Khz16BitMonoPcm
)
```

| Format | Sample Rate | Bit Depth | Use Case |
|--------|-------------|-----------|----------|
| `Riff16Khz16BitMonoPcm` | 16kHz | 16-bit | Standard WAV — telephony |
| `Riff24Khz16BitMonoPcm` | 24kHz | 16-bit | Higher quality WAV |
| `Audio16Khz32KBitRateMonoMp3` | 16kHz | — | MP3 streaming |
| `Audio24Khz160KBitRateMonoMp3` | 24kHz | — | Higher quality MP3 |
| `Ogg16Khz16BitMonoOpus` | 16kHz | — | Ogg/Opus for web |

---

## Lab vs Exam — Object Summary

| Object | Class | Purpose |
|--------|-------|---------|
| `speech_config` | `SpeechConfig` | Auth + language + voice settings |
| TTS output destination | `AudioOutputConfig` | File / speaker / stream |
| STT audio source | `AudioConfig` | File / microphone / stream |
| TTS engine | `SpeechSynthesizer` | Converts text → audio |
| STT engine | `SpeechRecognizer` | Converts audio → text |

---

## Endpoint Comparison — All 4 Labs

| Lab | Client | Endpoint Format |
|-----|--------|----------------|
| 01 | `TextAnalyticsClient` | `https://{resource}.services.ai.azure.com` |
| 02 | `AIProjectClient` | `https://{resource}.services.ai.azure.com/api/projects/{name}` |
| 03 | `AzureOpenAI` | `https://{resource}.services.ai.azure.com` (Target URI) |
| **04** | **`SpeechConfig`** | **`https://{resource}.cognitiveservices.azure.com/`** |

---

## Exam Traps
1. `AudioConfig` (input) ≠ `AudioOutputConfig` (output) — different classes
2. `recognize_once_async()` stops at first silence — use continuous for ongoing speech
3. `ResultReason.RecognizedSpeech` ← STT success; `ResultReason.SynthesizingAudioCompleted` ← TTS success
4. Endpoint must end with `/` and use `cognitiveservices.azure.com` domain
5. `foundry_key` loaded in code but **never used** — `DefaultAzureCredential` handles auth
6. `speak_text_async(text).get()` — `.get()` is required to block and get the result
7. Voice name format: `"en-US-VoiceName:ModelType"` — must match exactly

---

## Profanity Options
```python
speech_config.set_profanity(speech_sdk.ProfanityOption.Masked)
```
| Option | Behavior |
|--------|----------|
| `Raw` | Return profanity as-is |
| `Masked` | Replace letters with `*` |
| `Removed` | Remove profane words entirely |
