# Lab 07 – Translate Text and Speech

> **Exam relevance**: AI-102 / AI-103 — Azure Translator, Speech Translation SDK, Language Detection, Neural Machine Translation, SpeechTranslationConfig, multi-target translation

---

## 1. What This Lab Builds

Two separate Python apps using the same Foundry resource endpoint:

```
App 1: translate-text.py
  User types text (any language)
       │
       ▼
  Azure Translator SDK (TextTranslationClient)
  - Auto-detects source language
  - Translates to chosen target language (e.g., kn = Kannada)
       │
       ▼
  Prints: "'Hello' was translated from en to kn as 'ಹಲೋ'"

─────────────────────────────────────────────────

App 2: translate-speech.py
  User speaks English into microphone
       │
       ▼
  Speech SDK – SpeechTranslationConfig
  - STT: en-US → text
  - Translates to: French + Spanish + Hindi (3 targets simultaneously)
       │
       ▼
  SpeechSynthesizer plays each translation aloud in corresponding neural voice
```

---

## 2. Azure Translator — Deep Dive

### What is Azure Translator?
Cloud-based **neural machine translation (NMT)** service. Part of Azure AI Foundry Tools (formerly Azure AI Services / Cognitive Services).

Powers Microsoft products: Office 365, Teams, Edge, Bing.

### Translator Features (Exam Table)

| Feature | Description | API |
|---|---|---|
| **Text Translation v3** (GA) | Real-time text translation between 138+ languages | REST + SDK |
| **Text Translation v2026-06-06** (GA) | Newest NMT — LLM-enhanced, adaptive custom translation | REST |
| **Document Translation (Async)** | Batch translate documents; preserves formatting; uses Azure Blob | REST + SDK |
| **Document Translation (Sync)** | Single file translation; no blob storage needed | REST |
| **Custom Translator** | Train domain-specific models (legal, medical, etc.) | Custom Translator portal |
| **Adaptive Custom Translation** | Upload 5–10,000 aligned sentence pairs → adapts LLMs (GPT-5.1) in minutes | REST |

### Translator Endpoints
| Use case | Endpoint |
|---|---|
| Global text translation | `https://api.cognitive.microsofttranslator.com/` |
| Resource-specific (custom domain / Foundry) | `https://{resource-name}.cognitiveservices.azure.com/` |
| Document translation | `https://{resource-name}.cognitiveservices.azure.com/` |

> **IMPORTANT for exam**: In Lab 07 the `.env` uses the **cognitiveservices.azure.com** endpoint format (not the project endpoint). This is the Foundry resource's "AI Services" endpoint.

---

## 3. Azure Translator Python SDK

### Package
```bash
pip install azure-ai-translation-text==1.0.1
```

### Imports
```python
from azure.ai.translation.text import *
from azure.ai.translation.text.models import InputTextItem
```

### Create Client
```python
from azure.identity import DefaultAzureCredential
from azure.ai.translation.text import TextTranslationClient

credential = DefaultAzureCredential()
client = TextTranslationClient(
    credential=credential,
    endpoint="https://{resource}.cognitiveservices.azure.com/"
)
```

### Get Supported Languages
```python
languagesResponse = client.get_supported_languages(scope="translation")
# Returns a dict of language codes → language info
# languagesResponse.translation.keys() = ['af', 'ar', 'bg', 'bn', 'kn', ...]
print(f"{len(languagesResponse.translation)} languages supported.")
```
- `scope="translation"` — only translation languages
- Other scopes: `"transliteration"`, `"dictionary"`

### Translate Text
```python
input_text_elements = [InputTextItem(text="Hello world")]

# Returns a list — one result per input item
translationResponse = client.translate(
    body=input_text_elements,
    to_language=["fr", "es"]    # can specify multiple target languages
)

translation = translationResponse[0]
# Auto-detected source language
print(translation.detected_language.language)   # "en"
print(translation.detected_language.score)       # confidence 0.0–1.0

# Each target language
for translated_text in translation.translations:
    print(f"{translated_text.to}: {translated_text.text}")
    # "fr: 'Bonjour le monde'"
    # "es: 'Hola mundo'"
```

### Key Objects

| Object | Description |
|---|---|
| `TextTranslationClient` | Main client for the Azure Translator |
| `InputTextItem(text=...)` | Wraps one string for translation input |
| `client.get_supported_languages(scope=)` | Returns all supported language codes |
| `client.translate(body, to_language)` | Translate list of texts to one or more languages |
| `translation.detected_language.language` | ISO 639-1 code of auto-detected source (e.g., `"en"`) |
| `translation.detected_language.score` | Confidence of language detection (0.0–1.0) |
| `translation.translations` | List of translated outputs, one per target language |
| `translated_text.to` | Target language code (`"fr"`) |
| `translated_text.text` | Translated string |

---

## 4. Azure Speech Translation — Deep Dive

### What is Speech Translation?
Azure Speech service feature that provides **real-time audio → text translation**. Part of the same Speech SDK used for STT/TTS.

### Key Capabilities
| Feature | Description |
|---|---|
| **Speech-to-text translation** | Audio in language A → text in language B |
| **Speech-to-speech translation** | Audio in language A → audio in language B (combines STT + TTS) |
| **Multi-lingual translation** | No specified input language — auto-detects; handles language switching mid-session |
| **Live Interpreter** | Continuous identification + low-latency speech-to-speech translation (e.g., Teams meetings) |
| **Multiple target languages** | Translate to 2 languages free; additional languages charged per-character via Translator pricing |

### Multi-target language pricing rule (Exam)
- **First 2 target languages**: included in Speech Translation price
- **3rd+ target languages**: charged separately per character at Azure Translator text translation rates
- Formula: `speech_hours × rate + characters × translator_rate × (n_languages - 2)`

---

## 5. Speech SDK — Translation Classes

### Package
```bash
pip install azure-cognitiveservices-speech==1.48.2
```

### Imports
```python
import azure.cognitiveservices.speech as speech_sdk
```

### Key Classes for Translation

| Class | Purpose |
|---|---|
| `speech_sdk.translation.SpeechTranslationConfig` | Config for STT + translation |
| `speech_sdk.translation.TranslationRecognizer` | Performs STT and translation |
| `speech_sdk.SpeechConfig` | Config for TTS synthesis |
| `speech_sdk.SpeechSynthesizer` | Synthesizes text to speech (plays audio) |
| `speech_sdk.AudioConfig` | Audio input config (microphone or file) |
| `speech_sdk.audio.AudioOutputConfig` | Audio output config (speaker or file) |

---

## 6. Speech Translation — Annotated Code (translate-speech.py)

### Step 1: Configure translation input
```python
credential = DefaultAzureCredential()

translation_cfg = speech_sdk.translation.SpeechTranslationConfig(
    token_credential=credential,
    endpoint=foundry_endpoint      # https://{resource}.cognitiveservices.azure.com/
)

# Source language: what the user speaks
translation_cfg.speech_recognition_language = 'en-US'

# Target languages: translate into these
translation_cfg.add_target_language('fr')   # French
translation_cfg.add_target_language('es')   # Spanish
translation_cfg.add_target_language('hi')   # Hindi

# Input: default microphone
audio_in_cfg = speech_sdk.AudioConfig(use_default_microphone=True)

# Recognizer: does STT + translation
translator = speech_sdk.translation.TranslationRecognizer(
    translation_config=translation_cfg,
    audio_config=audio_in_cfg
)
```

### Step 2: Configure speech synthesis (TTS)
```python
# One SpeechConfig for TTS (same endpoint, same credential)
speech_cfg = speech_sdk.SpeechConfig(
    token_credential=credential,
    endpoint=foundry_endpoint
)

# Map language code → neural voice name
voices = {
    "fr": "fr-FR-HenriNeural",    # French male
    "es": "es-ES-ElviraNeural",   # Spanish female
    "hi": "hi-IN-MadhurNeural"    # Hindi male
}
```

### Step 3: Recognize + translate (single utterance)
```python
print("Speak now...")
translation_results = translator.recognize_once_async().get()
# recognize_once_async() → single utterance (stops at silence)
print(f"Translating '{translation_results.text}'")
# translation_results.text = source transcription
```

### Step 4: Print + speak each translation
```python
translations = translation_results.translations  # dict: {"fr": "Bonjour", "es": "Hola", ...}

for translation_language in translations:
    print(f"{translation_language}: '{translations[translation_language]}'")

    # Set the voice for this language
    speech_cfg.speech_synthesis_voice_name = voices.get(translation_language)

    audio_out_cfg = speech_sdk.audio.AudioOutputConfig(use_default_speaker=True)
    speech_synthesizer = speech_sdk.SpeechSynthesizer(speech_cfg, audio_out_cfg)

    # Synthesize and play the translated text
    speak = speech_synthesizer.speak_text_async(translations[translation_language]).get()

    if speak.reason != speech_sdk.ResultReason.SynthesizingAudioCompleted:
        print(speak.reason)  # log any error
```

---

## 7. Key SDK Methods Comparison

### Text Translator (azure-ai-translation-text)
| Method | What it does |
|---|---|
| `client.get_supported_languages(scope="translation")` | List all supported language codes |
| `client.translate(body=[InputTextItem(text=...)], to_language=[...])` | Translate text |
| `client.transliterate(...)` | Convert script (e.g., Arabic text → Latin letters) |
| `client.detect(...)` | Detect language without translating |
| `client.lookup_dictionary_entries(...)` | Alternative word/phrase suggestions |

### Speech SDK (azure-cognitiveservices-speech) — Translation
| Method | What it does |
|---|---|
| `SpeechTranslationConfig(token_credential, endpoint)` | Configure STT + translation |
| `.speech_recognition_language = 'en-US'` | Set the source (input) language |
| `.add_target_language('fr')` | Add a target language |
| `TranslationRecognizer(translation_config, audio_config)` | Create recognizer |
| `.recognize_once_async().get()` | Single-utterance recognition + translation (blocking) |
| `.start_continuous_recognition_async()` | Continuous recognition (event-driven) |
| `result.text` | Source language transcription |
| `result.translations` | Dict of target language → translated text |

### Speech SDK — Synthesis
| Method | What it does |
|---|---|
| `SpeechConfig(token_credential, endpoint)` | Configure TTS |
| `speech_cfg.speech_synthesis_voice_name = "fr-FR-HenriNeural"` | Set the output voice |
| `SpeechSynthesizer(speech_cfg, audio_out_cfg)` | Create synthesizer |
| `.speak_text_async(text).get()` | Synthesize text to speech (blocking) |
| `result.reason == ResultReason.SynthesizingAudioCompleted` | Success check |

---

## 8. Configuration (.env)

```ini
FOUNDRY_ENDPOINT=https://{resource-name}.cognitiveservices.azure.com/
```

> **Critical exam difference**:
> - Lab 05/06 used: `https://{resource}.services.ai.azure.com/api/projects/{project}`
> - Lab 07 uses: `https://{resource}.cognitiveservices.azure.com/` (AI Services endpoint)

Both endpoints come from the same Foundry resource but serve different APIs:
- `.services.ai.azure.com/api/projects/...` → Foundry SDK (agents, models)
- `.cognitiveservices.azure.com/` → Foundry Tools (Translator, Speech, Language, Vision)

---

## 9. Language Codes (Exam Examples)

| Language | Code | Neural Voice example |
|---|---|---|
| English (US) | `en-US` | `en-US-JennyNeural` |
| English (GB) | `en-GB` | `en-GB-SoniaNeural` |
| French | `fr` / `fr-FR` | `fr-FR-HenriNeural` |
| Spanish (Spain) | `es` / `es-ES` | `es-ES-ElviraNeural` |
| Hindi | `hi` / `hi-IN` | `hi-IN-MadhurNeural` |
| Kannada | `kn` | — |
| Arabic | `ar` | `ar-SA-HamedNeural` |
| Chinese (Simplified) | `zh-Hans` | `zh-CN-XiaoxiaoNeural` |
| Japanese | `ja` | `ja-JP-NanamiNeural` |

> Text Translator uses short codes (`"fr"`, `"es"`).  
> Speech SDK recognition language uses locale format (`"en-US"`, `"fr-FR"`).

---

## 10. Recognition Modes Comparison

| Mode | Method | Use case |
|---|---|---|
| **Single utterance** (Lab 07) | `recognize_once_async()` | One sentence/command, stops after silence |
| **Continuous** | `start_continuous_recognition_async()` + events | Long conversations, real-time transcription |
| **Push stream** | `PushAudioInputStream` | Feed audio data programmatically |
| **Pull stream** | `PullAudioInputStream` | Custom audio source |

### Continuous recognition pattern (not in Lab 07 but exam-relevant)
```python
# Wire up event handlers
translator.recognized.connect(lambda e: handle_result(e))
translator.start_continuous_recognition_async()
# ... run until stopped
translator.stop_continuous_recognition_async()
```

---

## 11. Full Architecture Comparison — Text vs Speech Translation

```
TEXT TRANSLATION (translate-text.py)
──────────────────────────────────────────────────────
User types: "こんにちは"   ← any language, auto-detected
      │
      ▼ InputTextItem(text=...)
TextTranslationClient.translate(body, to_language=["kn"])
      │
      ▼ HTTP POST to Azure Translator REST API
      Returns: detected_language="ja", translations=["ಹಲೋ"]
      │
      ▼
Print result

SPEECH TRANSLATION (translate-speech.py)
──────────────────────────────────────────────────────
User speaks English into microphone
      │
      ▼ AudioConfig(use_default_microphone=True)
SpeechTranslationConfig:
  - speech_recognition_language = "en-US"
  - target_languages = ["fr", "es", "hi"]
      │
      ▼ WebSocket to Azure Speech Service
TranslationRecognizer.recognize_once_async()
      │
      ▼ Returns: TranslationRecognitionResult
        .text = "How are you?"           (en-US transcription)
        .translations = {
          "fr": "Comment allez-vous ?",
          "es": "¿Cómo estás?",
          "hi": "आप कैसे हैं?"
        }
      │
      ▼ For each target language:
SpeechSynthesizer.speak_text_async(translation)
  → plays audio through speakers in that language's neural voice
```

---

## 12. ResultReason Values (Exam)

| Value | Meaning |
|---|---|
| `ResultReason.RecognizedSpeech` | Speech recognized successfully (STT) |
| `ResultReason.TranslatedSpeech` | Speech translated successfully |
| `ResultReason.NoMatch` | No speech detected |
| `ResultReason.Canceled` | Recognition canceled (error or end of audio) |
| `ResultReason.SynthesizingAudioCompleted` | TTS synthesis completed successfully |
| `ResultReason.SynthesizingAudio` | Audio chunk synthesized (streaming) |

---

## 13. Authentication — Same Pattern as Other Labs

Both apps use `DefaultAzureCredential` which resolves via `az login` locally.

```python
from azure.identity import DefaultAzureCredential
credential = DefaultAzureCredential()

# For Text Translator:
client = TextTranslationClient(credential=credential, endpoint=...)

# For Speech Translation (uses token_credential, not credential):
translation_cfg = speech_sdk.translation.SpeechTranslationConfig(
    token_credential=credential,  # ← note: token_credential not credential
    endpoint=...
)
```

> **Exam pitfall**: Speech SDK uses `token_credential=` parameter name, not `credential=`.

---

## 14. Code Bug Fixed in This Lab

The original `translate-text.py` had the **"Translate text"** comment section replaced with a duplicate of the language selection code. The fixed version replaces it with:

```python
# Translate text
inputText = ""
while inputText.lower() != "quit":
    inputText = input("Enter text to translate ('quit' to exit):")
    if inputText != "quit":
        input_text_elements = [InputTextItem(text=inputText)]
        translationResponse = client.translate(body=input_text_elements, to_language=[targetLanguage])
        translation = translationResponse[0] if translationResponse else None
        if translation:
            sourceLanguage = translation.detected_language
            for translated_text in translation.translations:
                print(f"'{inputText}' was translated from {sourceLanguage.language} to {translated_text.to} as '{translated_text.text}'.")
```

---

## 15. Surrounding Topics — Azure Translator Full Feature Set

### Custom Translator
- Train domain-specific NMT models (legal, medical, technical)
- Use the **Custom Translator portal** (`customtranslator.azure.com`)
- Upload parallel document pairs (source + translated)
- Deploy as a custom model, call via same Translator API with `category` parameter

### Adaptive Custom Translation (new in 2026)
- Upload **5–10,000 prealigned sentence pairs** (≤500 chars per pair)
- Adapts existing LLMs (e.g., GPT-5.1) in **minutes**
- No full model training required
- Called via Azure Translator APIs

### Document Translation
| Type | Storage | Best for |
|---|---|---|
| **Async (batch)** | Azure Blob Storage required | Many files, large documents |
| **Sync (single file)** | No blob needed, result returned directly | Single documents on-demand |

### Transliteration
Convert script without translation:
```python
# Arabic text → Latin letters (without translating meaning)
client.transliterate(
    body=[InputTextItem(text="مرحبا")],
    language="ar",
    from_script="Arab",
    to_script="Latn"
)
# → "marhabaan"
```

---

## 16. Key Concepts Comparison Table

| Concept | Text Translation | Speech Translation |
|---|---|---|
| Package | `azure-ai-translation-text` | `azure-cognitiveservices-speech` |
| Main class | `TextTranslationClient` | `SpeechTranslationConfig` + `TranslationRecognizer` |
| Input | String (`InputTextItem`) | Audio (microphone/file) |
| Output | Translated string | Translated text + optional TTS audio |
| Language detection | **Automatic** (no need to specify) | Source language **must be specified** |
| Multiple targets | Yes (`to_language=["fr","es","hi"]`) | Yes (`add_target_language()` × N) |
| Real-time | No (request-response) | **Yes** (streaming WebSocket) |
| 138 languages | Yes | Yes (STT input) |
| Free targets | All included | First 2 free, 3rd+ charged by characters |

---

## 17. Exam Pitfalls & Watch-Outs

1. **Text Translator endpoint** = `.cognitiveservices.azure.com/` NOT the project endpoint
2. **Speech SDK uses `token_credential=`** not `credential=` for Entra auth
3. **Language code format differs**:
   - Text Translator: `"fr"`, `"es"`, `"hi"` (short)
   - Speech recognition: `"en-US"`, `"fr-FR"` (locale format)
4. **`add_target_language()` uses short codes**: `"fr"`, NOT `"fr-FR"`
5. **`result.text`** = source language transcription (not translation)
6. **`result.translations`** = dict of `{language_code: translated_text}`
7. **`recognize_once_async()`** = blocking, single utterance; not for continuous use
8. **138 languages** supported for text translation; more limited for speech translation
9. **Text Translator auto-detects source language** — you don't need to specify it
10. **Speech translation source must be specified** — `speech_recognition_language = 'en-US'`
11. **3rd+ target languages are billed per character** via Translator pricing

---

## 18. Microsoft Learn References

- [What is Azure Translator?](https://learn.microsoft.com/azure/ai-services/translator/translator-overview)
- [Text Translation overview](https://learn.microsoft.com/azure/ai-services/translator/text-translation/overview)
- [Document Translation overview](https://learn.microsoft.com/azure/ai-services/translator/document-translation/overview)
- [What is speech translation?](https://learn.microsoft.com/azure/ai-services/speech-service/speech-translation)
- [Speech translation quickstart](https://learn.microsoft.com/azure/ai-services/speech-service/get-started-speech-translation)
- [Translator language support (138 languages)](https://learn.microsoft.com/azure/ai-services/translator/language-support)
- [Speech translation language support](https://learn.microsoft.com/azure/ai-services/speech-service/language-support?tabs=speech-translation)
- [Custom Translator overview](https://learn.microsoft.com/azure/ai-services/translator/custom-translator/overview)
- [azure-ai-translation-text PyPI](https://pypi.org/project/azure-ai-translation-text/)
