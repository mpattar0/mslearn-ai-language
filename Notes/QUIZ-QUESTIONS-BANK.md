# QUIZ QUESTIONS BANK — Labs 1–7 (Exact Exam Format)
# All questions stored as-asked, with options, correct answer, and explanation.
# Use this file to re-run the quiz yourself anytime.

---

## HOW TO USE
- Cover the "> ANSWER:" section, attempt the question
- Check your answer and read the explanation
- Focus on the WRONG ANSWERS SUMMARY at the end

---

## Q1 — Lab 01 | Multiple Choice

You are building a Python app that analyzes hotel reviews using Azure AI Language.
Which **client class** and **package** do you use?

**A)**
```python
from azure.ai.projects import AIProjectClient
client = AIProjectClient(endpoint=..., credential=...)
```
**B)**
```python
from azure.ai.textanalytics import TextAnalyticsClient
client = TextAnalyticsClient(endpoint=..., credential=...)
```
**C)**
```python
from openai import AzureOpenAI
client = AzureOpenAI(azure_endpoint=..., api_version=...)
```
**D)**
```python
from azure.ai.translation.text import TextTranslationClient
client = TextTranslationClient(endpoint=..., credential=...)
```

> **ANSWER: B** ✅ (User: B)
> `TextAnalyticsClient` from `azure-ai-textanalytics==5.3.0` is the client for Azure AI Language.
> - A = Foundry agents (Lab 02/05)
> - C = Azure OpenAI (Lab 03)
> - D = Azure Translator (Lab 07)

---

## Q2 — Lab 01 | Hotspot (2 blanks)

Fill in the **two blanks** to detect the language of a review:

```python
result = ai_client._______________(documents=[text])[0]
#                  ^^^BLANK1^^^

print(result.primary_language._______________)   # prints "English"
#                              ^^^BLANK2^^^
```

**Blank 1:**
- A) `analyze_sentiment`
- B) `detect_language`
- C) `recognize_entities`
- D) `extract_key_phrases`

**Blank 2:**
- W) `text`
- X) `iso6391_name`
- Y) `name`
- Z) `language`

> **ANSWER: B-Y** ✅ (User: B-Y)
> - `detect_language` is the method
> - `.primary_language.name` → `"English"` (full name)
> - `.primary_language.iso6391_name` → `"en"` (short code)
> - `.primary_language.confidence_score` → `0.99`

---

## Q3 — Lab 01 | Hotspot (2 blanks)

Fill in the **two blanks** to detect PII and print the masked version of the text:

```python
pii_result = ai_client._______________(documents=[text])[0]
#                       ^^^BLANK1^^^

print(pii_result._______________)   # prints "My name is ****"
#                ^^^BLANK2^^^
```

**Blank 1:**
- A) `recognize_entities`
- B) `recognize_linked_entities`
- C) `recognize_pii_entities`
- D) `extract_key_phrases`

**Blank 2:**
- W) `masked_text`
- X) `redacted_text`
- Y) `filtered_text`
- Z) `sanitized_text`

> **ANSWER: C-X** ✅ (User: C-X)
> - `recognize_pii_entities` detects Personal Identifiable Information
> - `redacted_text` = full text with PII replaced by `****`
> - `pii_result.entities` = list of individual PII items found

---

## Q4 — Lab 01 | Multiple Choice ❌ USER GOT WRONG

What is the **correct endpoint format** for `TextAnalyticsClient` in Lab 01?

**A)**
```
https://{resource}.services.ai.azure.com/api/projects/{project}
```
**B)**
```
https://{resource}.cognitiveservices.azure.com/
```
**C)**
```
https://{resource}.services.ai.azure.com
```
**D)**
```
https://api.cognitive.microsofttranslator.com/
```

> **ANSWER: C** ❌ (User: A — WRONG)
> - A = AIProjectClient (agents, Labs 02/05) — WRONG
> - **C = TextAnalyticsClient, AzureOpenAI, Voice Live base** ← CORRECT
> - B = Speech SDK, Translator SDK (Labs 04, 07)
> - D = Global Translator REST endpoint

---

## Q5 — Lab 02 | Hotspot (3 blanks)

Fill in the **three blanks** to call a Foundry agent:

```python
project_client = AIProjectClient(
    endpoint=foundry_endpoint,
    credential=DefaultAzureCredential()
)
openai_client = project_client._______________()
#                               ^^^BLANK1^^^

response = openai_client.responses.create(
    _______________=[{"role": "user", "content": prompt}],
#   ^^^BLANK2^^^
    extra_body={"agent_reference": {"name": agent_name, "type": "agent_reference"}}
)
print(response._______________)
#              ^^^BLANK3^^^
```

**Blank 1:** A) `get_chat_client`  B) `get_openai_client`  C) `get_inference_client`
**Blank 2:** W) `messages`  X) `prompt`  Y) `input`
**Blank 3:** P) `choices[0].message.content`  Q) `output_text`  R) `text`

> **ANSWER: B-Y-Q** ✅ (User: B-Y-Q)
> - `get_openai_client()` → AzureOpenAI pointed at Responses API
> - `input=` is the Responses API parameter (NOT `messages=`)
> - `response.output_text` (NOT `choices[0].message.content`)

---

## Q6 — Lab 02 | Multiple Choice

An agent named `language-helper` is configured in Foundry portal. Your `.env` has `AGENT_NAME=Language-Helper`. The app runs but the agent is **not found**. What is the most likely cause?

**A)** The `AIProjectClient` endpoint is missing the `/api/projects/` suffix
**B)** `agent_reference` must go in `messages=` not `extra_body=`
**C)** Agent names are case-sensitive — `Language-Helper` ≠ `language-helper`
**D)** `get_openai_client()` does not support the Responses API

> **ANSWER: C** ✅ (User: C)
> Agent names in Foundry are case-sensitive. Must match exactly what is in Foundry portal.

---

## Q7 — Lab 03 | Hotspot (2 blanks) ❌ USER GOT HALF WRONG

Fill in the **two blanks** to create an `AzureOpenAI` client using Entra ID authentication:

```python
token_provider = get_bearer_token_provider(
    DefaultAzureCredential(),
    "_______________"
#    ^^^BLANK1^^^
)

client = AzureOpenAI(
    _______________=endpoint,
#   ^^^BLANK2^^^
    azure_ad_token_provider=token_provider,
    api_version="2025-01-01-preview"
)
```

**Blank 1:**
- A) `"https://cognitiveservices.azure.com/.default"`
- B) `"https://management.azure.com/.default"`
- C) `"https://ai.azure.com/.default"`
- D) `"https://openai.azure.com/.default"`

**Blank 2:**
- W) `endpoint`
- X) `azure_endpoint`
- Y) `base_url`
- Z) `resource_endpoint`

> **ANSWER: C-X** ❌ (User: C-Z — Blank1 ✅, Blank2 ❌)
> - Scope: `"https://ai.azure.com/.default"` for Foundry/Azure OpenAI ✅
> - `azure_endpoint=` is the AzureOpenAI parameter ← KEY TRAP
> - `TextAnalyticsClient` → `endpoint=`  |  `AzureOpenAI` → `azure_endpoint=`

---

## Q8 — Lab 03 | Multiple Choice ❌ USER GOT WRONG

You want to generate speech from text using Azure OpenAI and save it as an MP3 file. Which code is correct?

**A)**
```python
result = client.audio.transcriptions.create(
    model=model_deployment, voice="alloy", input="Hello world"
)
result.stream_to_file("speech.mp3")
```
**B)**
```python
with client.audio.speech.with_streaming_response.create(
    model=model_deployment, voice="alloy", input="Hello world"
) as response:
    response.stream_to_file("speech.mp3")
```
**C)**
```python
result = client.audio.speech.create(
    model=model_deployment, voice="alloy",
    messages=[{"role": "user", "content": "Hello world"}]
)
result.stream_to_file("speech.mp3")
```
**D)**
```python
with client.audio.synthesis.with_streaming_response.create(
    model=model_deployment, voice="alloy", text="Hello world"
) as response:
    response.stream_to_file("speech.mp3")
```

> **ANSWER: B** ❌ (User: D — WRONG)
> - A) `client.audio.transcriptions` = STT not TTS
> - **B) CORRECT** — correct path, parameter, and streaming pattern
> - C) `messages=` is Chat Completions; TTS uses `input=`
> - D) `client.audio.synthesis` does not exist; `text=` is wrong param

---

## Q9 — Lab 04 | Hotspot (2 blanks)

Fill in the **two blanks** to create `SpeechConfig` with Entra ID and set the output voice:

```python
speech_config = speech_sdk.SpeechConfig(
    _______________=credential,
#   ^^^BLANK1^^^
    endpoint=foundry_endpoint
)

speech_config._______________ = "en-US-JennyNeural"
#             ^^^BLANK2^^^
```

**Blank 1:** A) `credential`  B) `auth_token`  C) `token_credential`  D) `azure_credential`
**Blank 2:** W) `voice_name`  X) `synthesis_voice`  Y) `speech_synthesis_voice_name`  Z) `output_voice_name`

> **ANSWER: C-Y**
> - `token_credential=` is the only valid Entra ID param for all Speech SDK classes
> - `speech_config.speech_synthesis_voice_name` sets the TTS voice

---

## Q10 — Lab 04 | Multiple Choice

You want to **transcribe a WAV file** to text using Azure Speech SDK. Which combination is correct?

**A)**
```python
audio_config = speech_sdk.audio.AudioOutputConfig(filename=file_path)
recognizer = speech_sdk.SpeechRecognizer(speech_config=speech_config, audio_config=audio_config)
result = recognizer.recognize_once_async().get()
```
**B)**
```python
audio_config = speech_sdk.AudioConfig(filename=file_path)
recognizer = speech_sdk.SpeechRecognizer(speech_config=speech_config, audio_config=audio_config)
result = recognizer.recognize_once_async().get()
print(result.text)
```
**C)**
```python
audio_config = speech_sdk.AudioConfig(filename=file_path)
synthesizer = speech_sdk.SpeechSynthesizer(speech_config=speech_config, audio_config=audio_config)
result = synthesizer.recognize_once_async().get()
print(result.text)
```
**D)**
```python
audio_config = speech_sdk.AudioConfig(use_default_microphone=True)
recognizer = speech_sdk.SpeechRecognizer(speech_config=speech_config, audio_config=audio_config)
result = recognizer.transcribe_async().get()
print(result.text)
```

> **ANSWER: B**
> - A) `AudioOutputConfig` = speaker OUTPUT, wrong for file input
> - **B) CORRECT** — `AudioConfig(filename=)` for input, `SpeechRecognizer`, `recognize_once_async()`
> - C) `SpeechSynthesizer` is TTS not STT; no `.recognize_once_async()` on it
> - D) `transcribe_async()` does not exist — correct is `recognize_once_async()`

---

## Q11 — Lab 04 | True or False

**"When using Azure Speech SDK with Entra ID authentication, you should pass `credential=` to `SpeechConfig`."**

- A) True
- B) False

> **ANSWER: B — False**
> Correct parameter is `token_credential=`, not `credential=`.

---

## Q12 — Lab 05 | Multiple Choice

What **two values** must be configured when connecting Azure Speech MCP Server in Foundry Portal?

**A)** `AZURE_CLIENT_SECRET` and `AZURE_TENANT_ID`
**B)** `Ocp-Apim-Subscription-Key` and `X-Blob-Container-Url`
**C)** `FOUNDRY_KEY` and `AZURE_STORAGE_CONNECTION_STRING`
**D)** `api-key` and `storage-account-name`

> **ANSWER: B**
> - `Ocp-Apim-Subscription-Key` = project API key → authenticates to Speech service
> - `X-Blob-Container-Url` = SAS URL of blob container → where TTS audio files are saved

---

## Q13 — Lab 06 | Hotspot (2 blanks)

Fill in the **two blanks** for the Voice Live async connection:

```python
from azure.identity.___ import AzureCliCredential
#                   ^^^BLANK1^^^

async with connect(
    endpoint=endpoint,
    credential=AzureCliCredential(),
    api_version="2026-01-01-preview",
    agent_config=agent_config
) as connection:
    await connection.session.update(session=session_config)
    await self._____________()
#                ^^^BLANK2^^^
```

**Blank 1:** A) `.sync`  B) `.aio`  C) `.async_`  D) (no submodule)
**Blank 2:** W) `start()`  X) `listen()`  Y) `process_events()`  Z) `run_loop()`

> **ANSWER: B-Y**
> - `azure.identity.aio` = async submodule — required for async Voice Live code
> - `process_events()` runs `async for event in connection` — the main event loop

---

## Q14 — Lab 06 | Multiple Choice

Which `ServerEventType` fires **first** after a successful Voice Live connection and signals that the microphone should start?

**A)** `RESPONSE_AUDIO_DELTA`
**B)** `INPUT_AUDIO_BUFFER_SPEECH_STARTED`
**C)** `SESSION_UPDATED`
**D)** `RESPONSE_AUDIO_DONE`

> **ANSWER: C**
> `SESSION_UPDATED` = session is ready and agent is connected.
> Microphone should ONLY start INSIDE this event handler — not before.

---

## Q15 — Lab 06 | Multiple Choice

A user interrupts the Voice Live agent mid-sentence. Which event fires and what should the app do?

**A)** `RESPONSE_AUDIO_DONE` → stop playback
**B)** `INPUT_AUDIO_BUFFER_SPEECH_STARTED` → call `clear_playback_queue()`
**C)** `SESSION_UPDATED` → restart the session
**D)** `INPUT_AUDIO_BUFFER_SPEECH_STOPPED` → call `clear_playback_queue()`

> **ANSWER: B**
> Barge-in: `INPUT_AUDIO_BUFFER_SPEECH_STARTED` fires → `clear_playback_queue()` immediately stops agent audio.

---

## Q16 — Lab 06 | Theory

What three components does Voice Live unify into a single WebSocket API?

**A)** Language detection + Translation + TTS
**B)** STT + NER + TTS
**C)** STT + Generative AI (LLM) + TTS
**D)** STT + Translation + Speaker Recognition

> **ANSWER: C**
> Voice Live = Speech-to-Text + LLM reasoning + Text-to-Speech — in one WebSocket.

---

## Q17 — Lab 06 | Multiple Choice

Your `.env` has:
```
AZURE_VOICELIVE_ENDPOINT="https://myresource.services.ai.azure.com/api/projects/proj-default"
```
The app throws `400 Invalid response status` on WebSocket connect. What is wrong?

**A)** The API version is incorrect
**B)** The endpoint must end with `/voice-live/realtime`
**C)** The endpoint must be the resource BASE URL — remove the `/api/projects/...` suffix
**D)** `AzureCliCredential` is not supported for Voice Live

> **ANSWER: C**
> The SDK appends `/voice-live/realtime` itself. Adding `/api/projects/...` creates a double-path → 400.
> Correct: `https://myresource.services.ai.azure.com`

---

## Q18 — Lab 07 | Hotspot (2 blanks)

Fill in the **two blanks** to translate text using Azure Translator SDK:

```python
response = client.translate(
    _______________=[InputTextItem(text="Hello")],
#   ^^^BLANK1^^^
    _______________=["fr", "kn"]
#   ^^^BLANK2^^^
)
translation = response[0]
print(translation.detected_language.language)   # "en"
```

**Blank 1:** A) `documents`  B) `texts`  C) `body`  D) `input`
**Blank 2:** W) `target_languages`  X) `to_language`  Y) `languages`  Z) `translate_to`

> **ANSWER: C-X**
> - `body=` wraps the list of `InputTextItem`
> - `to_language=` specifies target codes
> - Source language auto-detected — `from_language=` is optional

---

## Q19 — Lab 07 | Hotspot (2 blanks)

Fill in the **two blanks** for speech translation configuration:

```python
translation_cfg = speech_sdk.translation.SpeechTranslationConfig(
    token_credential=credential,
    endpoint=foundry_endpoint
)
translation_cfg._______________ = 'en-US'
#               ^^^BLANK1^^^

translation_cfg._______________('fr')
#               ^^^BLANK2^^^
```

**Blank 1:** A) `source_language`  B) `recognition_language`  C) `speech_recognition_language`  D) `input_language`
**Blank 2:** W) `set_target_language`  X) `append_target_language`  Y) `add_target_language`  Z) `push_target_language`

> **ANSWER: C-Y**
> - `speech_recognition_language` = source in locale format (`'en-US'`)
> - `add_target_language('fr')` = target in SHORT code (not `'fr-FR'`)

---

## Q20 — Lab 07 | Theory

How many target languages in speech translation are included at no extra cost, and what is charged for additional ones?

**A)** 1 free; 2+ billed per audio minute
**B)** 2 free; 3rd+ billed per character at Translator text rates
**C)** 3 free; 4+ require a separate Translator resource
**D)** Unlimited; all included in Speech service price

> **ANSWER: B**
> First 2 target languages = included. 3rd+ = billed per character at Azure Translator rates.

---

## Q21 — Theory | Multiple Choice

Which statement correctly describes the difference between Responses API and Chat Completions API?

**A)** Responses API supports embeddings; Chat Completions does not
**B)** Chat Completions API supports Foundry agents and MCP tools; Responses API is for models only
**C)** Responses API uses `input=` and supports agents + MCP tools; Chat Completions uses `messages=` for direct model calls
**D)** They are identical — both use `messages=` and `choices[0].message.content`

> **ANSWER: C**
> Responses API: `input=`, `response.output_text`, supports agents/MCP
> Chat Completions: `messages=`, `choices[0].message.content`, no agents

---

## Q22 — Theory | Multiple Choice

Which `DefaultAzureCredential` source is used in **all 7 labs** for local development?

**A)** Environment variables (`AZURE_CLIENT_ID`, `AZURE_CLIENT_SECRET`)
**B)** Managed Identity
**C)** Azure CLI (`az login`)
**D)** Visual Studio Code token cache

> **ANSWER: C** — All labs use `az login` for local development.

---

## Q23 — Theory | Multiple Choice

What is the most secure type of SAS token for Azure Blob Storage?

**A)** Account SAS — signed with the storage account key
**B)** Service SAS — signed with the storage account key
**C)** User Delegation SAS — signed with Microsoft Entra ID credentials
**D)** Anonymous SAS — no signing required

> **ANSWER: C**
> User Delegation SAS uses Entra ID — account key is never exposed.

---

## Q24 — Lab 04 | Multiple Choice

Which `ResultReason` value indicates **successful TTS synthesis**?

**A)** `speech_sdk.ResultReason.RecognizedSpeech`
**B)** `speech_sdk.ResultReason.SynthesizingAudio`
**C)** `speech_sdk.ResultReason.SynthesizingAudioCompleted`
**D)** `speech_sdk.ResultReason.AudioSynthesized`

> **ANSWER: C**
> - `RecognizedSpeech` = STT success
> - `SynthesizingAudio` = audio chunk during streaming (not final)
> - **`SynthesizingAudioCompleted`** = TTS fully done ← CORRECT
> - `AudioSynthesized` does not exist

---

## Q25 — Lab 07 | True or False

**"When using `TextTranslationClient.translate()`, you must always specify `from_language=` to indicate the source language."**

- A) True
- B) False

> **ANSWER: B — False**
> Azure Translator auto-detects source language. `from_language=` is optional.

---

## Q26 — Lab 06 | Multiple Choice

What audio format and sample rate does the Voice Live lab use?

**A)** MP3, 44100 Hz, stereo
**B)** PCM16, 16000 Hz, mono
**C)** PCM16, 24000 Hz, mono
**D)** WAV, 22050 Hz, stereo

> **ANSWER: C** — PCM16, 24kHz, Mono. Chunk size = 1200 samples (50ms @ 24kHz).

---

## Q27 — Lab 03 | Multiple Choice

You want to **transcribe an audio file** using Azure OpenAI (Whisper). Which is correct?

**A)**
```python
transcription = client.audio.speech.create(
    model=model_deployment, file=open(file_path, "rb")
)
```
**B)**
```python
transcription = client.audio.transcriptions.create(
    model=model_deployment,
    file=open(file_path, "rb"),
    response_format="text"
)
```
**C)**
```python
transcription = client.audio.transcriptions.create(
    model=model_deployment,
    input=open(file_path, "r"),
    response_format="json"
)
```
**D)**
```python
transcription = client.responses.create(
    model=model_deployment,
    input=open(file_path, "rb")
)
```

> **ANSWER: B**
> - A) `client.audio.speech` = TTS, not STT
> - **B) CORRECT** — `client.audio.transcriptions.create(file=open(...,"rb"), response_format="text")`
> - C) `input=` is wrong param (should be `file=`); must open in binary `"rb"`
> - D) `responses.create` is the agent API, not audio transcription

---

## Q28 — Lab 01 | Multiple Choice

You call `ai_client.analyze_sentiment(documents=[text])`. The result shows `sentiment = "mixed"`. What does this mean?

**A)** The model could not determine the sentiment
**B)** The document contains both positive and negative sentences
**C)** The confidence score is exactly 0.5
**D)** The text is in an unsupported language

> **ANSWER: B**
> `"mixed"` = document has both positive and negative sentences.
> Four values: `"positive"`, `"negative"`, `"neutral"`, `"mixed"`

---

## Q29 — Lab 06 | Theory

What VAD type supports **all models** AND detects end-of-turn based on **semantic meaning** in **multiple languages**?

**A)** `server_vad`
**B)** `semantic_vad`
**C)** `azure_semantic_vad`
**D)** `azure_semantic_vad_multilingual`

> **ANSWER: D**
> | Type | All models | Semantic | Multilingual |
> |---|---|---|---|
> | `server_vad` | ✅ | ❌ | ❌ |
> | `semantic_vad` | ❌ (realtime only) | ✅ | ❌ |
> | `azure_semantic_vad` | ✅ | ✅ | ❌ |
> | `azure_semantic_vad_multilingual` | ✅ | ✅ | ✅ |

---

## Q30 — Theory | Multiple Choice

You need to translate a **large batch of Word and PDF documents** while preserving formatting. Which feature?

**A)** Text Translation v3 with `client.translate()`
**B)** Synchronous Document Translation (single file, no blob needed)
**C)** Asynchronous (Batch) Document Translation with Azure Blob Storage
**D)** Custom Translator portal

> **ANSWER: C**
> - A) Text Translation = strings only, no document structure
> - B) Sync = single file on-demand only
> - **C) CORRECT** — Async Batch handles many files, preserves formatting, uses Blob Storage
> - D) Custom Translator = trains domain-specific models, not batch jobs

---

## Q31 — Lab 04 | Hotspot (2 blanks)

You want to synthesize text to a WAV **file** (not speaker). Fill in the blanks:

```python
audio_config = speech_sdk.audio.AudioOutputConfig(
    _______________="greeting.wav"
#   ^^^BLANK1^^^
)
speech_synthesizer = speech_sdk.SpeechSynthesizer(
    speech_config=speech_config,
    audio_config=audio_config
)
result = speech_synthesizer._______________.get()
#                            ^^^BLANK2^^^
```

**Blank 1:** A) `output_file`  B) `filename`  C) `filepath`  D) `file`
**Blank 2:** W) `speak_async("Hello")`  X) `synthesize_text_async("Hello")`  Y) `speak_text_async("Hello")`  Z) `generate_speech_async("Hello")`

> **ANSWER: B-Y**
> - `AudioOutputConfig(filename="greeting.wav")` — parameter is `filename=`
> - `speak_text_async("Hello").get()` — method is `speak_text_async`
> - `synthesize_text_async` and `speak_async` do not exist

---

## Q32 — Lab 01 | Multiple Choice

You call `ai_client.recognize_entities(documents=[text])[0]`. Which properties does each entity have?

**A)** `.word`, `.type`, `.score`
**B)** `.text`, `.category`, `.confidence_score`
**C)** `.name`, `.label`, `.probability`
**D)** `.value`, `.entity_type`, `.confidence`

> **ANSWER: B**
> - `entity.text` — the actual text found (e.g., "Microsoft")
> - `entity.category` — type (e.g., "Organization")
> - `entity.confidence_score` — 0.0–1.0

---

## Q33 — Lab 02 | True or False

**"The `extra_body` parameter in `responses.create()` is specific to the Azure Foundry SDK and has no equivalent in the standard OpenAI SDK."**

- A) True
- B) False

> **ANSWER: A — True**
> `extra_body` is used to pass Azure-specific extensions (like `agent_reference`) that are not part of the standard OpenAI API specification. The standard OpenAI SDK does not use `agent_reference`.

---

## Q34 — Lab 03 | Multiple Choice

Which is the correct `response_format` value when you want Azure OpenAI Whisper to return a **plain text string** (not JSON)?

**A)** `"plain"`
**B)** `"string"`
**C)** `"text"`
**D)** `"raw"`

> **ANSWER: C**
> Valid values: `"text"` | `"json"` | `"verbose_json"` | `"srt"` | `"vtt"`
> `"text"` returns a plain string.

---

## Q35 — Lab 04 | Multiple Choice

What is the correct endpoint format for `SpeechConfig` when using a Foundry resource?

**A)** `https://{resource}.services.ai.azure.com/api/projects/{project}`
**B)** `https://{resource}.services.ai.azure.com`
**C)** `https://{resource}.cognitiveservices.azure.com/`
**D)** `https://{region}.stt.speech.microsoft.com`

> **ANSWER: C**
> - Speech SDK (SpeechConfig, SpeechTranslationConfig) always uses `.cognitiveservices.azure.com/`
> - D is the region-based STT endpoint used only with raw REST calls, not the SDK
> - A = AIProjectClient endpoint
> - B = TextAnalyticsClient, AzureOpenAI endpoint

---

## Q36 — Lab 05 | True or False

**"In Lab 05, the agent's TTS response audio is returned as raw PCM bytes in `response.output_text`."**

- A) True
- B) False

> **ANSWER: B — False**
> The TTS audio is saved to Azure Blob Storage by the MCP server. `response.output_text` contains a **clickable URL link** to the `.wav` file in blob storage — not the raw audio bytes.

---

## Q37 — Lab 06 | Hotspot (2 blanks)

Fill in the blanks to send microphone audio to the Voice Live service:

```python
def capture_callback(in_data, frame_count, time_info, status):
    audio_base64 = base64._______________(in_data).decode("utf-8")
    #                      ^^^BLANK1^^^
    asyncio.run_coroutine_threadsafe(
        self.connection.input_audio_buffer._______________( audio=audio_base64 ),
        #                                  ^^^BLANK2^^^
        self.loop
    )
    return (None, pyaudio.paContinue)
```

**Blank 1:** A) `encode`  B) `b64encode`  C) `urlsafe_b64encode`  D) `encodebytes`
**Blank 2:** W) `send`  X) `write`  Y) `append`  Z) `push`

> **ANSWER: B-Y**
> - `base64.b64encode(in_data)` converts raw PCM bytes to base64
> - `connection.input_audio_buffer.append(audio=audio_base64)` sends it to Voice Live

---

## Q38 — Lab 06 | Multiple Choice

What is the chunk size and why in the Voice Live lab?

**A)** 4096 samples — standard FFT buffer size
**B)** 1200 samples — represents 50ms of audio at 24kHz
**C)** 2400 samples — represents 100ms of audio at 24kHz
**D)** 960 samples — standard WebRTC frame size

> **ANSWER: B**
> `chunk_size = 1200` = 50ms at 24000Hz (1200 / 24000 = 0.05 seconds)
> 50ms chunks give low latency while keeping network overhead reasonable.

---

## Q39 — Lab 07 | Multiple Choice

You need to convert Arabic text to Latin script (romanization) **without translating the meaning**. Which method do you use?

**A)** `client.translate(body=[...], to_language=["en"])`
**B)** `client.detect(body=[...])`
**C)** `client.transliterate(body=[...], language="ar", from_script="Arab", to_script="Latn")`
**D)** `client.break_sentence(body=[...], language="ar")`

> **ANSWER: C**
> `transliterate()` converts script without changing the language meaning.
> e.g., `"مرحبا"` → `"marhabaan"` (still Arabic meaning, just in Latin letters)

---

## Q40 — Lab 07 | Hotspot (2 blanks)

After translating speech, you want to speak the French translation aloud. Fill in the blanks:

```python
speech_cfg._______________ = voices.get("fr")
#           ^^^BLANK1^^^

speech_synthesizer = speech_sdk.SpeechSynthesizer(speech_cfg, audio_out_cfg)
speak = speech_synthesizer.speak_text_async(translations["fr"])._______________
#                                                               ^^^BLANK2^^^
```

**Blank 1:** A) `voice`  B) `tts_voice`  C) `speech_synthesis_voice_name`  D) `synthesis_voice_name`
**Blank 2:** W) `result()`  X) `wait()`  Y) `get()`  Z) `run()`

> **ANSWER: C-Y**
> - `speech_cfg.speech_synthesis_voice_name` sets the voice
> - `.get()` makes the async call blocking (waits for result)

---

## Q41 — Theory | Multiple Choice

Which Azure service would you use to **train a custom translation model** using your own industry-specific terminology (e.g., legal or medical documents)?

**A)** Azure Translator Text API with `category=` parameter and Custom Translator portal
**B)** Azure AI Language with custom NER training
**C)** Azure Speech with custom acoustic model
**D)** Azure OpenAI fine-tuning with parallel document pairs

> **ANSWER: A**
> Custom Translator portal allows uploading parallel document pairs to train domain-specific NMT models. The trained model is referenced via the `category=` parameter in the Translator API.

---

## Q42 — Theory | Multiple Choice

You are building a contact center solution that needs to handle customer calls in **real-time** with voice responses. The solution must support **barge-in** (customer interrupts the agent). Which service is most appropriate?

**A)** Azure AI Language + Azure Speech SDK (chained STT → NLP → TTS)
**B)** Azure Speech Voice Live API with `azure_semantic_vad` turn detection
**C)** Azure Translator with speech-to-speech translation
**D)** Azure OpenAI with audio.transcriptions and audio.speech endpoints

> **ANSWER: B**
> Voice Live provides built-in barge-in detection, low latency, noise suppression, echo cancellation — all in one WebSocket. Chaining services (A, D) adds latency and complexity. C is for translation not conversation.

---

## Q43 — Lab 01 | Hotspot (2 blanks)

Fill in the blanks to extract the **sentiment** and its **positive confidence score**:

```python
result = ai_client.analyze_sentiment(documents=[text])[0]

print(result._______________)             # "positive"
#            ^^^BLANK1^^^
print(result.confidence_scores._______________)   # 0.95
#                               ^^^BLANK2^^^
```

**Blank 1:** A) `label`  B) `polarity`  C) `sentiment`  D) `feeling`
**Blank 2:** W) `score`  X) `positive`  Y) `positive_score`  Z) `confidence`

> **ANSWER: C-X**
> - `result.sentiment` → `"positive"` | `"negative"` | `"neutral"` | `"mixed"`
> - `result.confidence_scores.positive` → float 0.0–1.0

---

## Q44 — Lab 02 | Multiple Choice

What is the purpose of `project_client.get_openai_client()` in the Foundry SDK?

**A)** Creates a connection directly to the Azure OpenAI resource (bypassing the project)
**B)** Returns an AzureOpenAI client that routes through the project endpoint to `/openai/v1/responses`, enabling access to Foundry models and platform tools
**C)** Creates an async HTTP client for calling the Responses API directly
**D)** Returns a new `AIProjectClient` scoped to the OpenAI sub-resource

> **ANSWER: B**
> `get_openai_client()` returns an `AzureOpenAI` instance configured to route through the **project endpoint** (`/openai/v1/responses`). This gives access to Foundry-specific features like agents, MCP tools, platform tools — not available through the resource-level OpenAI endpoint.

---

## Q45 — Lab 06 | Multiple Choice

The Voice Live pricing tier is determined by which factor?

**A)** The number of concurrent WebSocket connections
**B)** The Azure region of the Foundry resource
**C)** The generative AI model selected
**D)** The number of target languages configured

> **ANSWER: C**
> | Tier | Models |
> |---|---|
> | Pro | gpt-5, gpt-4o, gpt-4.1, gpt-realtime |
> | Basic | gpt-5-mini, gpt-4o-mini, gpt-4.1-mini |
> | Lite | gpt-5-nano, phi4-mm-realtime, phi4-mini |
> You choose a model — the tier is automatic.

---

## Q46 — Lab 04 | Multiple Choice

Which class and method do you use for **continuous** (ongoing) speech recognition from a microphone?

**A)**
```python
recognizer = speech_sdk.SpeechRecognizer(speech_config, audio_config)
recognizer.recognize_once_async().get()
```
**B)**
```python
recognizer = speech_sdk.SpeechRecognizer(speech_config, audio_config)
recognizer.recognized.connect(callback)
recognizer.start_continuous_recognition_async()
```
**C)**
```python
recognizer = speech_sdk.SpeechSynthesizer(speech_config, audio_config)
recognizer.start_recognition_async()
```
**D)**
```python
recognizer = speech_sdk.SpeechRecognizer(speech_config, audio_config)
recognizer.transcribe_continuous_async()
```

> **ANSWER: B**
> - A = single utterance only
> - **B = CORRECT** — wire `recognized` event, then call `start_continuous_recognition_async()`
> - C = SpeechSynthesizer is for TTS, not STT
> - D = `transcribe_continuous_async()` does not exist

---

## Q47 — Theory | True or False

**"Azure Translator Text Translation supports 138 languages, but Azure Speech Translation supports far fewer source languages."**

- A) True
- B) False

> **ANSWER: B — False**
> Both Azure Translator and Azure Speech Translation support a broad range of languages. Speech Translation supports 140+ locales for STT input. The number is comparable, not "far fewer."

---

## Q48 — Lab 05 | Multiple Choice

What is the correct format for the Azure Speech MCP Server endpoint configured in Foundry Portal?

**A)** `https://{resource}.services.ai.azure.com/speech/mcp`
**B)** `https://{resource}.cognitiveservices.azure.com/speech/mcp?api-version=2025-11-15-preview`
**C)** `wss://{resource}.cognitiveservices.azure.com/speech/realtime`
**D)** `https://{resource}.speech.microsoft.com/mcp/v1`

> **ANSWER: B**
> Full format: `https://{foundry-resource-name}.cognitiveservices.azure.com/speech/mcp?api-version=2025-11-15-preview`
> - Uses HTTPS (not WSS — that's Voice Live)
> - Uses `.cognitiveservices.azure.com` (not `.services.ai.azure.com`)
> - Requires the `api-version` query parameter

---

## Q49 — Lab 07 | Multiple Choice

You call `translation_cfg.add_target_language('fr-FR')`. The code runs but translations are not returned for French. What is wrong?

**A)** French is not supported by Azure Speech Translation
**B)** `add_target_language()` requires the locale format — use `'fr_FR'` instead
**C)** `add_target_language()` requires the short language code — use `'fr'` not `'fr-FR'`
**D)** You must call `set_target_language()` not `add_target_language()`

> **ANSWER: C**
> `add_target_language()` takes **short codes** (`'fr'`, `'es'`, `'hi'`), NOT locale format (`'fr-FR'`).
> Compare: `speech_recognition_language = 'en-US'` (source uses locale) vs `add_target_language('fr')` (target uses short code).

---

## Q50 — Theory | Multiple Choice

What is the main difference between **Asynchronous (Batch) Document Translation** and **Synchronous Document Translation** in Azure Translator?

**A)** Batch requires a Premium tier; Synchronous works on Free tier
**B)** Batch processes multiple files via Azure Blob Storage; Synchronous handles a single file and returns the result directly
**C)** Batch supports only PDF; Synchronous supports all file types
**D)** Batch uses REST API; Synchronous uses the Python SDK only

> **ANSWER: B**
> | | Async Batch | Sync Single |
> |---|---|---|
> | Files | Many | One |
> | Storage | Azure Blob required | No blob needed |
> | Response | Polling / callback | Returned directly |
> | Use case | Large batch jobs | On-demand single document |

---

## WRONG ANSWERS SUMMARY (Update as you go)

| Q# | Topic | User Answer | Correct | Key Lesson |
|---|---|---|---|---|
| Q4 | TextAnalyticsClient endpoint | A | C | `.services.ai.azure.com` NOT `/api/projects/...` |
| Q7 | AzureOpenAI parameter name | C-Z | C-X | `azure_endpoint=` not `resource_endpoint=` |
| Q8 | TTS code path + param | D | B | `client.audio.speech` NOT `.synthesis`; `input=` not `text=` |
| Q9 | SpeechConfig credential param | D-Y | C-Y | `token_credential=` — ONLY valid Entra param in all Speech SDK classes |

---

## RESUME STATUS
- **Last question asked**: Q9
- **Next question**: Q10
- **Score when paused**: 5/9 (56%)
- **Questions completed**: Q1–Q9 (from Q1–Q30 bank + Q31–Q50 additional bank)
- **Total available**: 50 questions
