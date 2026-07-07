# MASTER EXAM REFERENCE — AI-102 / AI-103 (Labs 1–7)
# Azure AI Language · Speech · Translator · Foundry Agents

> Covers: Endpoints, SDKs, Client constructors, Key methods, Parameters, Hotspot traps

---

## SECTION 1 — ENDPOINT MAP (All Labs)

This is the #1 source of confusion. Know this cold.

### Endpoint Types

| Endpoint Pattern | Used For | Labs |
|---|---|---|
| `https://{resource}.services.ai.azure.com` | Azure Language (TextAnalytics), Azure OpenAI (via AzureOpenAI SDK) | Lab 01, Lab 03 |
| `https://{resource}.services.ai.azure.com/api/projects/{project}` | Foundry SDK (agents, Responses API, AIProjectClient) | Lab 02, Lab 05 |
| `https://{resource}.services.ai.azure.com` (base, no path) | Voice Live SDK (`azure-ai-voicelive`) — SDK appends `/voice-live/realtime` itself | Lab 06 |
| `https://{resource}.cognitiveservices.azure.com/` | Azure Speech SDK (SpeechConfig), Azure Translator SDK (TextTranslationClient), Speech Translation | Labs 04, 07 |

### Real Values from Your Labs

| Lab | .env var | Actual endpoint |
|---|---|---|
| Lab 01 (Text Analytics) | `FOUNDRY_ENDPOINT` | `https://{resource}.services.ai.azure.com` |
| Lab 02 (Language Agent) | `FOUNDRY_ENDPOINT` | `https://{resource}.services.ai.azure.com/api/projects/{project}` |
| Lab 03 (Gen AI Speech) | `MODEL_ENDPOINT` | `https://{resource}.services.ai.azure.com` |
| Lab 04 (Azure Speech SDK) | `FOUNDRY_ENDPOINT` | `https://{resource}.cognitiveservices.azure.com/` |
| Lab 05 (Speech MCP Agent) | `FOUNDRY_ENDPOINT` | `https://{resource}.services.ai.azure.com/api/projects/{project}` |
| Lab 06 (Voice Live) | `AZURE_VOICELIVE_ENDPOINT` | `https://{resource}.services.ai.azure.com` ← BASE only |
| Lab 07 (Translator + Speech) | `FOUNDRY_ENDPOINT` | `https://{resource}.cognitiveservices.azure.com/` |

### Quick Rule for Exam
```
Need agents / Foundry models?     → .services.ai.azure.com/api/projects/{project}
Need Azure Speech SDK / Translator? → .cognitiveservices.azure.com/
Need Azure OpenAI / Language AI?  → .services.ai.azure.com  (no /api/projects)
Voice Live SDK?                    → .services.ai.azure.com  (no suffix — SDK adds it)
```

---

## SECTION 2 — SDK & PACKAGE MAP

| Service | Python Package | Version (lab) | Primary Client Class |
|---|---|---|---|
| Azure AI Language | `azure-ai-textanalytics` | `5.3.0` | `TextAnalyticsClient` |
| Azure AI Projects (Foundry) | `azure-ai-projects` | `2.0.0b4` | `AIProjectClient` |
| Azure OpenAI | `openai` | latest | `AzureOpenAI` |
| Azure Speech SDK | `azure-cognitiveservices-speech` | `1.48.2` | `SpeechConfig` |
| Azure AI Voice Live | `azure-ai-voicelive` | `1.2.0b4` | `connect()` (async) |
| Azure Translator | `azure-ai-translation-text` | `1.0.1` | `TextTranslationClient` |
| Azure Identity (sync) | `azure-identity` | latest | `DefaultAzureCredential` |
| Azure Identity (async) | `azure-identity` | latest | `AzureCliCredential` (aio) |

---

## SECTION 3 — LAB-BY-LAB CODE HOTSPOTS

---

### LAB 01 — Azure AI Language (Text Analytics)

**Endpoint**: `.services.ai.azure.com` (no `/api/projects`)

#### Client Construction ← HOTSPOT
```python
from azure.ai.textanalytics import TextAnalyticsClient
from azure.identity import DefaultAzureCredential

credential = DefaultAzureCredential()
ai_client = TextAnalyticsClient(
    endpoint=foundry_endpoint,   # ← parameter name: endpoint (NOT azure_endpoint)
    credential=credential        # ← parameter name: credential
)
```

#### Detect Language ← HOTSPOT
```python
detectedLanguage = ai_client.detect_language(documents=[text])[0]
# ↑ documents= takes a list of strings
# ↑ [0] → first document result

print(detectedLanguage.primary_language.name)    # "English"
print(detectedLanguage.primary_language.iso6391_name)  # "en"
print(detectedLanguage.primary_language.confidence_score)  # 0.99
```

#### Recognize Entities ← HOTSPOT
```python
entities = ai_client.recognize_entities(documents=[text])[0].entities
for entity in entities:
    print(entity.text)      # "Microsoft"
    print(entity.category)  # "Organization"
    print(entity.confidence_score)  # 0.98
```

#### Recognize PII ← HOTSPOT
```python
pii_result = ai_client.recognize_pii_entities(documents=[text])[0]
pii_entities = pii_result.entities
for pii_entity in pii_entities:
    print(pii_entity.text)      # "John Smith"
    print(pii_entity.category)  # "Person"
print(pii_result.redacted_text) # "Hello ****"
```

#### Other TextAnalyticsClient methods (exam list)
```python
ai_client.analyze_sentiment(documents=[text])        # positive/negative/neutral/mixed
ai_client.extract_key_phrases(documents=[text])      # key phrase list
ai_client.recognize_linked_entities(documents=[text]) # Wikipedia-linked entities
ai_client.extract_summary(documents=[...])           # extractive summarization
ai_client.abstract_summary(documents=[...])          # abstractive summarization
```

#### Sentiment result shape
```python
sentiment_result = ai_client.analyze_sentiment(documents=[text])[0]
print(sentiment_result.sentiment)          # "positive" | "negative" | "neutral" | "mixed"
print(sentiment_result.confidence_scores.positive)   # 0.95
print(sentiment_result.confidence_scores.negative)   # 0.02
print(sentiment_result.confidence_scores.neutral)    # 0.03
```

---

### LAB 02 — Language Agent (Foundry Agent via Responses API)

**Endpoint**: `.services.ai.azure.com/api/projects/{project}`

#### Client Construction ← HOTSPOT
```python
from azure.ai.projects import AIProjectClient
from azure.identity import DefaultAzureCredential

project_client = AIProjectClient(
    endpoint=foundry_endpoint,        # full project endpoint
    credential=DefaultAzureCredential()
)
```

#### Get OpenAI-compatible client ← HOTSPOT
```python
openai_client = project_client.get_openai_client()
# Returns an openai.AzureOpenAI instance pointed at /openai/v1/responses
```

#### Call Agent via Responses API ← HOTSPOT
```python
response = openai_client.responses.create(
    input=[{"role": "user", "content": prompt}],  # ← input= (NOT messages=)
    extra_body={
        "agent_reference": {
            "name": agent_name,       # "text-agent" — CASE SENSITIVE
            "type": "agent_reference"
        }
    },
)
print(response.output_text)   # ← output_text (NOT choices[0].message.content)
```

#### Exam traps — Responses API vs Chat Completions
| Feature | Responses API (`responses.create`) | Chat Completions (`chat.completions.create`) |
|---|---|---|
| Input param | `input=` | `messages=` |
| Output | `response.output_text` | `response.choices[0].message.content` |
| Agents | ✅ via `extra_body` | ❌ |
| MCP tools | ✅ | ❌ |
| Embeddings | ❌ | ✅ via separate call |

---

### LAB 03 — Gen AI Speech (Azure OpenAI TTS + Transcription)

**Endpoint**: `.services.ai.azure.com` (resource-level, NOT project endpoint)

#### Client Construction with Entra ID ← HOTSPOT
```python
from openai import AzureOpenAI
from azure.identity import DefaultAzureCredential, get_bearer_token_provider

token_provider = get_bearer_token_provider(
    DefaultAzureCredential(),
    "https://ai.azure.com/.default"   # ← scope string
)

client = AzureOpenAI(
    azure_endpoint=endpoint,                    # ← azure_endpoint (NOT endpoint)
    azure_ad_token_provider=token_provider,     # ← Entra auth
    api_version="2025-01-01-preview"
)
```

#### Generate Speech (TTS) ← HOTSPOT
```python
with client.audio.speech.with_streaming_response.create(
    model=model_deployment,        # e.g., "tts-1" or "tts-1-hd"
    voice="alloy",                 # alloy | echo | fable | onyx | nova | shimmer
    input="Hello world",           # ← text to speak
    instructions="Speak softly."   # ← style instruction (optional)
) as response:
    response.stream_to_file(speech_file_path)  # saves as .mp3
```

**Voices**: `alloy`, `echo`, `fable`, `onyx`, `nova`, `shimmer`

#### Transcribe Audio (STT) ← HOTSPOT
```python
audio_file = open(file_path, "rb")   # ← open in binary read mode

transcription = client.audio.transcriptions.create(
    model=model_deployment,           # e.g., "whisper-1"
    file=audio_file,
    response_format="text"            # "text" | "json" | "verbose_json" | "srt" | "vtt"
)
print(transcription.text if hasattr(transcription, 'text') else transcription)
```

---

### LAB 04 — Azure Speech SDK (STT + TTS)

**Endpoint**: `.cognitiveservices.azure.com/`

#### SpeechConfig Construction ← HOTSPOT
```python
import azure.cognitiveservices.speech as speech_sdk
from azure.identity import DefaultAzureCredential

credential = DefaultAzureCredential()
speech_config = speech_sdk.SpeechConfig(
    token_credential=credential,    # ← token_credential (NOT credential)
    endpoint=foundry_endpoint       # ← .cognitiveservices.azure.com/
)
```

> **CRITICAL**: Speech SDK uses `token_credential=` not `credential=`

#### Text-to-Speech (synthesize to file) ← HOTSPOT
```python
speech_config.speech_synthesis_voice_name = "en-US-Serena:DragonHDLatestNeural"

audio_config = speech_sdk.audio.AudioOutputConfig(
    filename="output.wav"           # ← to file
    # OR use_default_speaker=True   # ← to speaker
)

speech_synthesizer = speech_sdk.SpeechSynthesizer(
    speech_config=speech_config,
    audio_config=audio_config
)

result = speech_synthesizer.speak_text_async(greeting_message).get()

if result.reason == speech_sdk.ResultReason.SynthesizingAudioCompleted:
    print("Success")
```

#### Speech-to-Text (transcribe from file) ← HOTSPOT
```python
audio_config = speech_sdk.audio.AudioConfig(
    filename=file_path              # ← from file
    # OR use_default_microphone=True # ← from mic
)

speech_recognizer = speech_sdk.SpeechRecognizer(
    speech_config=speech_config,
    audio_config=audio_config
)

result = speech_recognizer.recognize_once_async().get()

if result.reason == speech_sdk.ResultReason.RecognizedSpeech:
    print(result.text)
```

#### AudioConfig vs AudioOutputConfig ← HOTSPOT
| Class | Purpose | Key params |
|---|---|---|
| `AudioConfig` | Input (microphone/file) | `filename=` or `use_default_microphone=True` |
| `AudioOutputConfig` | Output (speaker/file) | `filename=` or `use_default_speaker=True` |

#### ResultReason values ← HOTSPOT
```python
speech_sdk.ResultReason.RecognizedSpeech          # STT success
speech_sdk.ResultReason.NoMatch                    # No speech detected
speech_sdk.ResultReason.Canceled                   # Error / end of audio
speech_sdk.ResultReason.SynthesizingAudioCompleted # TTS success
speech_sdk.ResultReason.SynthesizingAudio          # TTS chunk (streaming)
speech_sdk.ResultReason.TranslatedSpeech           # Translation success
```

---

### LAB 05 — Speech Agent via MCP (AIProjectClient + Responses API)

**Endpoint**: `.services.ai.azure.com/api/projects/{project}`

Same client pattern as Lab 02. Key additions:
- Agent has Azure Speech MCP Server connected in Foundry Portal
- TTS output is a **blob URL link** in `response.output_text`
- SAS token on blob container grants the MCP server write access

#### Full pattern (same as Lab 02)
```python
project_client = AIProjectClient(
    endpoint=foundry_endpoint,
    credential=DefaultAzureCredential()
)
openai_client = project_client.get_openai_client()

response = openai_client.responses.create(
    input=[{"role": "user", "content": prompt}],
    extra_body={"agent_reference": {"name": agent_name, "type": "agent_reference"}}
)
print(response.output_text)   # contains clickable URL to .wav file
```

#### MCP Server connection (Foundry Portal config) ← HOTSPOT
| Header / Param | Value |
|---|---|
| Remote MCP endpoint | `https://{resource}.cognitiveservices.azure.com/speech/mcp?api-version=2025-11-15-preview` |
| `Ocp-Apim-Subscription-Key` | API key from Foundry project |
| `X-Blob-Container-Url` | SAS URL of the `files` blob container |

---

### LAB 06 — Voice Live Agent (WebSocket real-time speech)

**Endpoint**: `.services.ai.azure.com` (BASE ONLY — no suffix)

#### Connect (async context manager) ← HOTSPOT
```python
from azure.ai.voicelive.aio import connect
from azure.ai.voicelive.models import AgentConfig
from azure.identity.aio import AzureCliCredential    # ← async version

agent_config = AgentConfig({
    "agent_name": "chat-agent",     # Foundry agent name
    "project_name": "proj-default"  # Foundry project name
})

async with connect(
    endpoint=endpoint,               # BASE URL only — no /api/projects
    credential=AzureCliCredential(),
    api_version="2026-01-01-preview",
    agent_config=agent_config
) as connection:
    ...
```

#### Session Update ← HOTSPOT
```python
from azure.ai.voicelive.models import (
    RequestSession, Modality,
    InputAudioFormat, OutputAudioFormat,
    AzureSemanticVadMultilingual,
    AudioEchoCancellation, AudioNoiseReduction
)

session_config = RequestSession(
    modalities=[Modality.TEXT, Modality.AUDIO],
    input_audio_format=InputAudioFormat.PCM16,
    output_audio_format=OutputAudioFormat.PCM16,
    turn_detection=AzureSemanticVadMultilingual(),
    input_audio_echo_cancellation=AudioEchoCancellation(),
    input_audio_noise_reduction=AudioNoiseReduction(type="azure_deep_noise_suppression")
)
await connection.session.update(session=session_config)
```

#### Server Event Types ← HOTSPOT
```python
from azure.ai.voicelive.models import ServerEventType

ServerEventType.SESSION_UPDATED          # → start microphone capture
ServerEventType.INPUT_AUDIO_BUFFER_SPEECH_STARTED  # → barge-in (clear queue)
ServerEventType.INPUT_AUDIO_BUFFER_SPEECH_STOPPED  # → user stopped speaking
ServerEventType.CONVERSATION_ITEM_INPUT_AUDIO_TRANSCRIPTION_COMPLETED  # → user transcript
ServerEventType.RESPONSE_AUDIO_DELTA     # → audio chunk from agent (queue for playback)
ServerEventType.RESPONSE_AUDIO_TRANSCRIPT_DONE   # → agent full text response
ServerEventType.RESPONSE_AUDIO_DONE     # → audio complete
ServerEventType.ERROR                    # → service error
```

#### Send microphone audio ← HOTSPOT
```python
audio_base64 = base64.b64encode(in_data).decode("utf-8")
await connection.input_audio_buffer.append(audio=audio_base64)
# in_data = raw PCM16 bytes from PyAudio callback
```

#### VAD Types ← HOTSPOT
| Type | Works with | Feature |
|---|---|---|
| `server_vad` | All models | Silence-based |
| `azure_semantic_vad` | All models | Understands sentence meaning |
| `azure_semantic_vad_multilingual` | All models | Semantic + 10 languages |
| `semantic_vad` | gpt-realtime only | OpenAI semantic |

---

### LAB 07 — Azure Translator + Speech Translation

**Endpoint**: `.cognitiveservices.azure.com/`

#### TEXT TRANSLATOR — Client Construction ← HOTSPOT
```python
from azure.ai.translation.text import TextTranslationClient
from azure.ai.translation.text.models import InputTextItem
from azure.identity import DefaultAzureCredential

credential = DefaultAzureCredential()
client = TextTranslationClient(
    credential=credential,
    endpoint=foundry_endpoint    # .cognitiveservices.azure.com/
)
```

#### Get Supported Languages ← HOTSPOT
```python
languagesResponse = client.get_supported_languages(scope="translation")
# .translation → dict of language code → info
# .transliteration → transliteration scripts
# .dictionary → dictionary lookup languages

len(languagesResponse.translation)     # 138
"kn" in languagesResponse.translation.keys()  # True (Kannada)
```

#### Translate Text ← HOTSPOT
```python
input_text_elements = [InputTextItem(text="Hello world")]

translationResponse = client.translate(
    body=input_text_elements,        # ← body= (list of InputTextItem)
    to_language=["fr", "es", "kn"]  # ← to_language= (list of target codes)
    # from_language="en"             # ← optional; auto-detected if omitted
)

translation = translationResponse[0]   # first document

# Source language (auto-detected)
translation.detected_language.language      # "en"
translation.detected_language.score         # 0.99 (confidence)

# Each translation
for t in translation.translations:
    t.to    # "fr"
    t.text  # "Bonjour le monde"
```

#### SPEECH TRANSLATION — Config ← HOTSPOT
```python
import azure.cognitiveservices.speech as speech_sdk

credential = DefaultAzureCredential()

translation_cfg = speech_sdk.translation.SpeechTranslationConfig(
    token_credential=credential,         # ← token_credential= (NOT credential=)
    endpoint=foundry_endpoint            # .cognitiveservices.azure.com/
)

translation_cfg.speech_recognition_language = 'en-US'  # source lang (locale format)
translation_cfg.add_target_language('fr')   # target (short code, NOT fr-FR)
translation_cfg.add_target_language('es')
translation_cfg.add_target_language('hi')
```

#### Speech TranslationRecognizer ← HOTSPOT
```python
audio_in_cfg = speech_sdk.AudioConfig(use_default_microphone=True)

translator = speech_sdk.translation.TranslationRecognizer(
    translation_config=translation_cfg,
    audio_config=audio_in_cfg
)

translation_results = translator.recognize_once_async().get()

translation_results.text                  # source transcription ("How are you?")
translation_results.translations          # dict {"fr": "Comment...", "es": "¿Cómo..."}
translation_results.translations["fr"]   # access by language code
```

#### Speech Synthesis (TTS playback of translations) ← HOTSPOT
```python
speech_cfg = speech_sdk.SpeechConfig(
    token_credential=credential,
    endpoint=foundry_endpoint
)

# Voice map: language code → neural voice name
voices = {
    "fr": "fr-FR-HenriNeural",
    "es": "es-ES-ElviraNeural",
    "hi": "hi-IN-MadhurNeural"
}

for lang in translation_results.translations:
    speech_cfg.speech_synthesis_voice_name = voices.get(lang)   # ← set voice per language
    audio_out_cfg = speech_sdk.audio.AudioOutputConfig(use_default_speaker=True)
    synthesizer = speech_sdk.SpeechSynthesizer(speech_cfg, audio_out_cfg)
    speak = synthesizer.speak_text_async(translation_results.translations[lang]).get()

    if speak.reason != speech_sdk.ResultReason.SynthesizingAudioCompleted:
        print(speak.reason)
```

---

## SECTION 4 — AUTHENTICATION DEEP DIVE

### DefaultAzureCredential Chain (sync)
Tries in order — first success wins:
1. `AZURE_CLIENT_ID` + `AZURE_CLIENT_SECRET` + `AZURE_TENANT_ID` env vars
2. Workload Identity
3. Managed Identity
4. Azure CLI (`az login`) ← **used in all labs**
5. Azure Developer CLI (`azd auth login`)
6. Visual Studio / VS Code
7. Interactive browser

### Token Scope Strings ← HOTSPOT
| Service | Scope |
|---|---|
| All Azure AI / Foundry | `"https://ai.azure.com/.default"` |
| Cognitive Services | `"https://cognitiveservices.azure.com/.default"` |

### get_bearer_token_provider (Lab 03 pattern) ← HOTSPOT
```python
from azure.identity import DefaultAzureCredential, get_bearer_token_provider

token_provider = get_bearer_token_provider(
    DefaultAzureCredential(),
    "https://ai.azure.com/.default"
)
# Used with AzureOpenAI(azure_ad_token_provider=token_provider)
```

### AzureCliCredential async (Lab 06 only) ← HOTSPOT
```python
from azure.identity.aio import AzureCliCredential   # ← .aio submodule
# Required for async code (Voice Live SDK)
```

### Speech SDK credential parameter name ← HOTSPOT
```python
# ALL Speech SDK classes use:  token_credential=  NOT  credential=
speech_sdk.SpeechConfig(token_credential=credential, endpoint=...)
speech_sdk.translation.SpeechTranslationConfig(token_credential=credential, endpoint=...)
```

---

## SECTION 5 — SPEECH SDK COMPLETE PARAMETER REFERENCE

### SpeechConfig Parameters
```python
speech_sdk.SpeechConfig(
    subscription="key",            # API key auth (not recommended)
    region="eastus",               # used with subscription key
    # OR
    token_credential=credential,   # Entra ID auth (recommended)
    endpoint="https://..."         # resource endpoint
)
```

### SpeechSynthesizer — All key properties
```python
speech_config.speech_synthesis_voice_name = "en-US-JennyNeural"   # voice
speech_config.speech_synthesis_language = "en-US"                   # language

synthesizer = speech_sdk.SpeechSynthesizer(
    speech_config=speech_config,
    audio_config=audio_config     # None = no audio output (text only)
)

# Methods:
synthesizer.speak_text_async("Hello").get()      # plain text
synthesizer.speak_ssml_async("<speak>...</speak>").get()  # SSML
```

### SpeechRecognizer — All key methods
```python
recognizer = speech_sdk.SpeechRecognizer(
    speech_config=speech_config,
    audio_config=audio_config,
    language="en-US"              # optional override
)

recognizer.recognize_once_async().get()        # single utterance (blocks)
recognizer.start_continuous_recognition_async()  # continuous (event-driven)
recognizer.stop_continuous_recognition_async()

# Event handlers (continuous mode):
recognizer.recognized.connect(callback)   # final result
recognizer.recognizing.connect(callback) # interim result
recognizer.canceled.connect(callback)    # cancellation/error
```

### SpeechTranslationConfig — All key methods
```python
translation_cfg = speech_sdk.translation.SpeechTranslationConfig(
    token_credential=credential,
    endpoint=endpoint
)

translation_cfg.speech_recognition_language = 'en-US'  # source (locale)
translation_cfg.add_target_language('fr')               # target (short code)
translation_cfg.voice_name = 'fr-FR-HenriNeural'        # synthesis voice

translator = speech_sdk.translation.TranslationRecognizer(
    translation_config=translation_cfg,
    audio_config=audio_in_cfg
)
result = translator.recognize_once_async().get()
result.text                     # source transcript
result.translations             # dict {lang_code: translated_text}
result.translations.get("fr")  # safe access
```

### SSML Example ← HOTSPOT
```xml
<speak version="1.0" xmlns="http://www.w3.org/2001/10/synthesis" xml:lang="en-US">
    <voice name="en-US-JennyNeural">
        <prosody rate="slow" pitch="high" volume="loud">
            Hello world!
        </prosody>
    </voice>
</speak>
```

SSML controls: `rate`, `pitch`, `volume`, `emphasis`, `break`, `say-as`

---

## SECTION 6 — AZURE TRANSLATOR COMPLETE PARAMETER REFERENCE

### TextTranslationClient methods

#### translate()
```python
client.translate(
    body=[InputTextItem(text="Hello")],
    to_language=["fr", "es"],       # required — list of target codes
    from_language="en",             # optional — auto-detected if omitted
    text_type="plain",              # "plain" | "html"
    profanity_action="NoAction",    # "NoAction" | "Marked" | "Deleted"
    include_sentence_length=False,  # include sentence boundaries
    category="general"             # custom translator category
)
```

#### detect()
```python
detect_response = client.detect(body=[InputTextItem(text="Bonjour")])
detect_response[0].language          # "fr"
detect_response[0].confidence        # 1.0
detect_response[0].is_translation_supported  # True
```

#### transliterate()
```python
client.transliterate(
    body=[InputTextItem(text="مرحبا")],
    language="ar",
    from_script="Arab",    # source script
    to_script="Latn"       # target script
)
# → "marhabaan"  (same language, different script)
```

#### break_sentence()
```python
client.break_sentence(
    body=[InputTextItem(text="Hello world. How are you?")],
    language="en"
)
# Returns sentence boundary positions
```

#### lookup_dictionary_entries()
```python
client.lookup_dictionary_entries(
    body=[InputTextItem(text="fly")],
    from_language="en",
    to_language="es"
)
# Returns alternative translations (dictionary lookup)
```

---

## SECTION 7 — AZURE AI LANGUAGE COMPLETE REFERENCE

### TextAnalyticsClient — All key methods

```python
# Language detection
client.detect_language(documents=["text..."])
  → result[0].primary_language.name         # "English"
  → result[0].primary_language.iso6391_name # "en"
  → result[0].primary_language.confidence_score # 0.99

# Sentiment
client.analyze_sentiment(documents=["text..."])
  → result[0].sentiment                     # "positive"|"negative"|"neutral"|"mixed"
  → result[0].confidence_scores.positive    # 0.0–1.0
  → result[0].sentences[i].sentiment        # per-sentence
  → result[0].sentences[i].mined_opinions   # aspect-based sentiment

# Key phrases
client.extract_key_phrases(documents=["text..."])
  → result[0].key_phrases                   # ["Azure", "machine learning"]

# Named Entity Recognition
client.recognize_entities(documents=["text..."])
  → result[0].entities[i].text             # "Microsoft"
  → result[0].entities[i].category        # "Organization"
  → result[0].entities[i].subcategory     # "Commerce" (optional)
  → result[0].entities[i].confidence_score # 0.98

# PII Detection
client.recognize_pii_entities(documents=["text..."])
  → result[0].entities[i].text            # "John Smith"
  → result[0].entities[i].category       # "Person"
  → result[0].redacted_text              # "Hello ****"

# Entity Linking
client.recognize_linked_entities(documents=["text..."])
  → result[0].entities[i].name           # "Microsoft"
  → result[0].entities[i].url            # Wikipedia URL
  → result[0].entities[i].data_source    # "Wikipedia"

# Extractive Summarization
client.begin_extract_summary(documents=["text..."])  # long-running
  → result[0].sentences[i].text          # summary sentence
  → result[0].sentences[i].rank_score    # importance score

# Custom Named Entity Recognition
client.begin_recognize_custom_entities(
    documents=["text..."],
    project_name="myProject",
    deployment_name="myDeployment"
)
```

### NER Entity Categories (Exam) ← HOTSPOT
| Category | Examples |
|---|---|
| Person | "John Smith", "CEO" |
| Organization | "Microsoft", "NHS" |
| Location | "London", "Azure region" |
| DateTime | "Monday", "2026-01-01" |
| Quantity | "10 miles", "$500" |
| Email | "user@domain.com" |
| PhoneNumber | "+1-555-0100" |
| URL | "https://example.com" |

---

## SECTION 8 — FOUNDRY AGENT SERVICE REFERENCE

### Agent = Model + Instructions + Tools

#### Prompt Agent (Labs 02, 05)
- Created in Foundry Portal
- No code to maintain — Foundry runs it
- Called via Responses API with `agent_reference`

#### Create agent_reference call ← HOTSPOT
```python
response = openai_client.responses.create(
    input=[{"role": "user", "content": prompt}],
    extra_body={
        "agent_reference": {
            "name": "speech-agent",    # CASE SENSITIVE
            "type": "agent_reference"  # literal string
        }
    }
)
response.output_text   # agent's reply
```

### AIProjectClient — All useful methods
```python
project_client = AIProjectClient(endpoint=..., credential=...)

project_client.get_openai_client()         # → AzureOpenAI (for Responses API)
project_client.connections.list()          # list connections
project_client.agents.list_agents()        # list agents
project_client.tracing                     # tracing configuration
```

### Foundry RBAC Roles ← HOTSPOT
| Role | Can do |
|---|---|
| **Foundry User** | Call agents, use models (least privilege) |
| **Foundry Project Manager** | Manage projects and connections |
| **Foundry Owner** | Full resource control |
| **Contributor / Owner** | Subscription-level Azure permissions |

---

## SECTION 9 — VOICE NEURAL VOICE NAMES FORMAT

```
Pattern: {language}-{Locale}-{Name}Neural
         OR
         {language}-{Locale}-{Name}:{Style}

Examples:
  en-US-JennyNeural              Standard neural
  en-GB-SoniaNeural              British English
  en-US-Serena:DragonHDLatestNeural   HD voice (Lab 04)
  fr-FR-HenriNeural              French male
  es-ES-ElviraNeural             Spanish female
  hi-IN-MadhurNeural             Hindi male
  fr-FR-HenriNeural              French male (Lab 07)
```

### OpenAI TTS voices (Lab 03)
`alloy`, `echo`, `fable`, `onyx`, `nova`, `shimmer`

---

## SECTION 10 — KEY LANGUAGE CODES

| Language | Text Translator | Speech locale | Speech target |
|---|---|---|---|
| English (US) | `en` | `en-US` | `en` |
| English (UK) | `en` | `en-GB` | `en` |
| French | `fr` | `fr-FR` | `fr` |
| Spanish | `es` | `es-ES` | `es` |
| Hindi | `hi` | `hi-IN` | `hi` |
| Kannada | `kn` | — | — |
| Arabic | `ar` | `ar-SA` | `ar` |
| Chinese (Simplified) | `zh-Hans` | `zh-CN` | `zh-Hans` |
| Japanese | `ja` | `ja-JP` | `ja` |

> Text Translator `add_target_language()` uses SHORT codes (`"fr"`, `"es"`)  
> `speech_recognition_language` uses LOCALE format (`"en-US"`, `"fr-FR"`)

---

## SECTION 11 — AUDIO FORMAT REFERENCE

| Parameter | Value | Meaning |
|---|---|---|
| `PCM16` | 16-bit PCM | Standard for microphone/speaker |
| Sample rate | 24000 Hz | Voice Live default |
| Sample rate | 16000 Hz | Alternative (Speech SDK default) |
| Chunk size | 1200 samples | 50ms chunks @ 24kHz |
| Channels | 1 (Mono) | All speech APIs use mono |
| File format | `.wav` | Speech SDK output |
| File format | `.mp3` | Azure OpenAI TTS output |

---

## SECTION 12 — TOP 25 EXAM TRAPS

| # | Trap | Correct |
|---|---|---|
| 1 | Using `credential=` in Speech SDK | Use `token_credential=` |
| 2 | Using `messages=` in Responses API | Use `input=` |
| 3 | Using `choices[0].message.content` | Use `response.output_text` |
| 4 | Using project URL for Voice Live | Use resource BASE URL only |
| 5 | Using `fr-FR` in `add_target_language()` | Use `fr` (short code) |
| 6 | Forgetting `documents=` is a list | `documents=[text]` not `documents=text` |
| 7 | Forgetting `[0]` after TextAnalytics call | Returns list of results |
| 8 | Using `azure_endpoint=` for TextAnalyticsClient | Use `endpoint=` |
| 9 | Using sync `AzureCliCredential` in async code | Use `from azure.identity.aio import AzureCliCredential` |
| 10 | Agent name case mismatch | "speech-agent" ≠ "Speech-Agent" |
| 11 | Using `.cognitiveservices.azure.com` for AIProjectClient | Must be `.services.ai.azure.com/api/projects/...` |
| 12 | Using `.services.ai.azure.com/api/projects/...` for TextTranslationClient | Must be `.cognitiveservices.azure.com/` |
| 13 | Using `.services.ai.azure.com/api/projects/...` for Voice Live | Must be resource base URL |
| 14 | Forgetting `extra_body` for agent_reference | Goes in `extra_body`, not a top-level param |
| 15 | `speech_sdk.audio.AudioConfig` vs `AudioOutputConfig` | Input = `AudioConfig`, Output = `AudioOutputConfig` |
| 16 | Text Translator auto-detects language | Don't need `from_language=` (but can specify) |
| 17 | Speech translation source must be specified | `speech_recognition_language = 'en-US'` |
| 18 | 3rd+ translation target languages are billed | First 2 free, 3+ charged per character |
| 19 | `recognize_once_async()` is for single utterance | Use continuous recognition for ongoing speech |
| 20 | TTS output from MCP agent is a URL | Not raw audio bytes — it's a blob link |
| 21 | `get_bearer_token_provider` scope | `"https://ai.azure.com/.default"` for Foundry |
| 22 | `AzureOpenAI` parameter name | `azure_endpoint=` (not `endpoint=`) |
| 23 | `InputTextItem` needed for translator | Can't pass raw string to `client.translate()` |
| 24 | `response.audio.speech` vs `response.audio.transcriptions` | TTS = `.speech`, STT = `.transcriptions` |
| 25 | SAS token vs API key | SAS = Blob Storage; API key = Speech/Translator service |

---

## SECTION 13 — COMPLETE PACKAGE INSTALLATION COMMANDS

```bash
# Lab 01 — Text Analytics
pip install python-dotenv azure-identity azure-ai-textanalytics==5.3.0

# Lab 02 — Language Agent
pip install python-dotenv azure-identity azure-ai-projects==2.0.0b4

# Lab 03 — Gen AI Speech (OpenAI TTS + Whisper)
pip install python-dotenv azure-identity openai playsound3

# Lab 04 — Azure Speech SDK
pip install python-dotenv azure-identity azure-cognitiveservices-speech==1.48.2 playsound3

# Lab 05 — Speech MCP Agent
pip install python-dotenv azure-identity azure-ai-projects==2.0.0b4

# Lab 06 — Voice Live
pip install azure-identity azure-ai-voicelive==1.2.0b4 --pre azure-ai-projects==2.0.0b4 pyaudio

# Lab 07 — Translation
pip install python-dotenv azure-identity azure-ai-translation-text==1.0.1 azure-cognitiveservices-speech==1.48.2
```

---

## SECTION 14 — HOTSPOT QUESTION PATTERNS (fill-in-the-blank)

These follow Microsoft exam hotspot format where you pick from a dropdown per blank.

### Q1: Create a TextAnalyticsClient
```python
ai_client = TextAnalyticsClient(
    endpoint=foundry_endpoint,
    credential=_______________   # ← DefaultAzureCredential()
)
```
**Answer**: `DefaultAzureCredential()`

### Q2: Detect language from text
```python
result = ai_client._______________( documents=[text] )[0]
print(result.primary_language.name)
```
**Answer**: `detect_language`

### Q3: Get PII entities
```python
pii_result = ai_client._______________( documents=[text] )[0]
print(pii_result._______________)   # masked text
```
**Answer**: `recognize_pii_entities` / `redacted_text`

### Q4: Call a Foundry agent
```python
response = openai_client.responses.create(
    _______________=[{"role": "user", "content": prompt}],   # ← input=
    extra_body={"agent_reference": {"name": agent_name, "type": "agent_reference"}}
)
print(response._______________)  # ← output_text
```
**Answer**: `input` / `output_text`

### Q5: Create SpeechConfig with Entra ID
```python
speech_config = speech_sdk.SpeechConfig(
    _______________=credential,    # ← token_credential
    endpoint=foundry_endpoint
)
```
**Answer**: `token_credential`

### Q6: Set output voice
```python
speech_config._______________  = "en-US-JennyNeural"
```
**Answer**: `speech_synthesis_voice_name`

### Q7: Output audio to speaker
```python
audio_config = speech_sdk.audio.AudioOutputConfig(
    _______________=True
)
```
**Answer**: `use_default_speaker`

### Q8: Translate text
```python
response = client.translate(
    body=[InputTextItem(text="Hello")],
    _______________=["fr"]         # ← to_language
)
detected = response[0].detected_language.___________  # ← language
```
**Answer**: `to_language` / `language`

### Q9: Speech translation - add target
```python
translation_cfg.speech_recognition_language = 'en-US'
translation_cfg._______________('fr')   # ← add_target_language
```
**Answer**: `add_target_language`

### Q10: Check TTS success
```python
if result.reason == speech_sdk.ResultReason._______________:
    print("Success")
```
**Answer**: `SynthesizingAudioCompleted`

### Q11: Voice Live base endpoint (fill in the blank)
```
AZURE_VOICELIVE_ENDPOINT = "https://{resource}._______________"
```
**Answer**: `services.ai.azure.com`  (no `/api/projects/...` suffix)

### Q12: Get OpenAI client from project
```python
openai_client = project_client._______________()
```
**Answer**: `get_openai_client`

### Q13: Connect Voice Live (async)
```python
async with _______________(
    endpoint=endpoint,
    credential=credential,
    api_version="2026-01-01-preview",
    agent_config=agent_config
) as connection:
```
**Answer**: `connect`

### Q14: Session event that triggers microphone start
```
ServerEventType._______________
```
**Answer**: `SESSION_UPDATED`

### Q15: Create bearer token for AzureOpenAI
```python
token_provider = get_bearer_token_provider(
    DefaultAzureCredential(),
    "_______________"    # scope
)
```
**Answer**: `https://ai.azure.com/.default`

---

## SECTION 15 — FEATURE MATRIX (exam choose-the-service questions)

| Requirement | Service | SDK |
|---|---|---|
| Detect what language text is in | Azure AI Language | `TextAnalyticsClient.detect_language()` |
| Find people/places/org in text | Azure AI Language | `.recognize_entities()` |
| Find credit card / SSN in text | Azure AI Language | `.recognize_pii_entities()` |
| Summarize a long document | Azure AI Language | `.begin_extract_summary()` |
| Is this review positive? | Azure AI Language | `.analyze_sentiment()` |
| Convert text to speech (OpenAI voices) | Azure OpenAI | `client.audio.speech.create()` |
| Transcribe audio (Whisper) | Azure OpenAI | `client.audio.transcriptions.create()` |
| TTS to file / speaker (Azure voices) | Azure Speech SDK | `SpeechSynthesizer` |
| STT from mic / file (Azure) | Azure Speech SDK | `SpeechRecognizer` |
| Translate text between 138 languages | Azure Translator | `TextTranslationClient.translate()` |
| Translate speech to multiple languages | Speech SDK | `SpeechTranslationConfig` + `TranslationRecognizer` |
| Real-time voice conversation agent | Azure Voice Live | `azure-ai-voicelive.connect()` |
| Chat agent with instructions | Foundry Agent Service | `AIProjectClient` + `responses.create()` |
| Agent that can call speech tools | Foundry Agent + MCP | Azure Speech MCP Server |
| Convert Arabic text → Latin letters | Azure Translator | `client.transliterate()` |
| Train domain-specific translation model | Custom Translator | Custom Translator portal |
| Translate large batch of documents | Azure Translator | Document Translation (async) |
