# Lab 06 – Develop a Voice Live Agent

> **Exam relevance**: AI-102 / AI-103 — Azure Speech Voice Live API, Real-time speech-to-speech, WebSocket, PyAudio, Foundry Agents, Async Python, Voice Mode

---

## 1. What This Lab Builds

```
Microphone (user speaks)
     │  PCM16 audio chunks (base64 encoded)
     ▼
chat-client.py  (Python, async)
  azure.ai.voicelive.aio.connect()  — WebSocket to Voice Live API
     │
     ▼
Voice Live API (WSS endpoint)
  ├── Azure Speech STT  → transcript text
  ├── GPT-5 (gpt-5 model)  → reasoning / response text
  └── Azure Speech TTS (neural voice)  → audio response
     │
     ▼
Speakers (agent speaks back)
```

**Goal**: A real-time, conversational voice agent — speak to it, it speaks back. Fully bidirectional, interrupt-capable.

---

## 2. What is the Voice Live API?

### Definition
Voice Live is a **fully managed, low-latency speech-to-speech API** that integrates:
- **Azure Speech STT** (input)
- **Generative AI model** (reasoning)
- **Azure Speech TTS** (output)

…all into a **single WebSocket interface**, eliminating the need to chain those components manually.

### Why Voice Live vs DIY pipeline?
| Aspect | DIY (STT → LLM → TTS) | Voice Live |
|---|---|---|
| Latency (user-perceived) | High (sequential processing) | Low (streaming, pipelined) |
| Engineering cost | High (3 systems to connect) | Low (one API) |
| Interruption handling | Manual (hard) | Built-in |
| Echo cancellation | Client-side (hard) | Server-side (built-in) |
| Noise suppression | Manual | Built-in |
| End-of-turn detection | Manual (VAD) | Advanced semantic VAD |

### Protocol
- Communicates over **WebSockets** (`wss://`)
- Event-driven: client sends events, server sends events
- **Compatible with Azure OpenAI Realtime API events** — same event schema with Voice Live extras

---

## 3. WebSocket Endpoint Structure

### Base endpoint (resource-level, NOT project-level)
```
wss://{resource-name}.services.ai.azure.com/voice-live/realtime?api-version=2026-01-01-preview
```

### With direct model (no agent)
```
wss://{resource}.services.ai.azure.com/voice-live/realtime?api-version=2026-01-01-preview&model=gpt-5
```

### With Foundry agent
```
wss://{resource}.services.ai.azure.com/voice-live/realtime?api-version=2026-01-01-preview
    &agent-name=chat-agent
    &agent-project-name=proj-default
```

> **CRITICAL EXAM POINT**: The endpoint is the **Foundry resource base URL**, NOT the project endpoint (`/api/projects/...`). This is the bug fixed in Lab 06 — the `.env` had the full project path.

---

## 4. Authentication for Voice Live

### Two methods
| Method | How | Security level |
|---|---|---|
| **Microsoft Entra (recommended)** | `Bearer` token in `Authorization` header | Best — no keys in code |
| **API key** | `api-key` connection header OR `?api-key=` query param | Simpler but less secure |

### Required RBAC roles for Entra auth
- `Cognitive Services User` — to call the Speech service
- `Foundry User` — to access the Foundry project/agent

### In Lab 06: `AzureCliCredential`
```python
from azure.identity.aio import AzureCliCredential
credential = AzureCliCredential()
```
The **async version** (`azure.identity.aio`) is required because the Voice Live SDK is fully async.

---

## 5. Azure AI Voice Live SDK (`azure-ai-voicelive`)

### Installation (Lab 06 requires the beta)
```bash
pip install azure-ai-voicelive==1.2.0b4 --pre
pip install azure-identity azure-ai-projects==2.0.0b4
```

### Core connection pattern
```python
from azure.ai.voicelive.aio import connect
from azure.ai.voicelive.models import AgentConfig

agent_config = AgentConfig({
    "agent_name": "chat-agent",
    "project_name": "proj-default"
})

async with connect(
    endpoint="https://{resource}.services.ai.azure.com",   # BASE URL only!
    credential=AzureCliCredential(),
    api_version="2026-01-01-preview",
    agent_config=agent_config
) as connection:
    # connection is an async context manager (WebSocket session)
    ...
```

### Two modes of connection
| Mode | Parameter | Use case |
|---|---|---|
| **Agent mode** (Lab 06) | `agent_config=AgentConfig(...)` | Named Foundry agent — instructions in portal |
| **Direct model mode** | `model="gpt-5"` | Custom per-session instructions |

---

## 6. The 5-Step Conversation Flow (Lab 06)

```python
async with connect(...) as connection:

    # STEP 1: Connection established
    self.connection = connection

    # STEP 2: Attach audio hardware handler
    self.audio_processor = AudioProcessor(connection)

    # STEP 3: Configure session (audio format, VAD, noise, echo)
    await self.setup_session()

    # STEP 4: Start audio playback thread (speakers)
    self.audio_processor.start_playback()

    # STEP 5: Event loop — process all incoming server events
    await self.process_events()
```

---

## 7. Session Configuration (`session.update`)

Sent right after connection opens to configure the session behavior:

```python
from azure.ai.voicelive.models import (
    RequestSession,
    Modality,
    InputAudioFormat,
    OutputAudioFormat,
    AzureSemanticVadMultilingual,
    AudioEchoCancellation,
    AudioNoiseReduction,
)

session_config = RequestSession(
    modalities=[Modality.TEXT, Modality.AUDIO],       # both text + audio
    input_audio_format=InputAudioFormat.PCM16,         # 16-bit PCM input
    output_audio_format=OutputAudioFormat.PCM16,       # 16-bit PCM output
    turn_detection=AzureSemanticVadMultilingual(),     # semantic VAD
    input_audio_echo_cancellation=AudioEchoCancellation(),  # server echo cancel
    input_audio_noise_reduction=AudioNoiseReduction(
        type="azure_deep_noise_suppression"            # background noise removal
    )
)

await connection.session.update(session=session_config)
```

### Key session properties explained

| Property | What it does |
|---|---|
| `modalities` | `[TEXT, AUDIO]` = respond with both text transcript and audio |
| `input_audio_format` | `PCM16` = 16-bit PCM (standard for microphone capture) |
| `output_audio_format` | `PCM16` = 16-bit PCM (standard for speaker playback) |
| `turn_detection` | When does the user stop speaking? (triggers model response) |
| `echo_cancellation` | Prevents mic from picking up the agent's own speaker output |
| `noise_reduction` | Removes background noise to improve STT accuracy |

---

## 8. Turn Detection (VAD) — Deep Dive

**Voice Activity Detection (VAD)** determines when the user has stopped speaking so the agent can respond.

### VAD Types (exam-critical)

| Type | Works with | Description |
|---|---|---|
| `server_vad` | All models | Volume-based — detects silence after sound |
| `semantic_vad` | gpt-realtime, gpt-realtime-mini only | Semantic meaning — waits for sentence completion |
| `azure_semantic_vad` | **All models** | Azure's semantic VAD — understands natural pauses |
| `azure_semantic_vad_multilingual` | **All models** | Same + supports 10+ languages (used in Lab 06) |

### Key VAD parameters
| Parameter | Default | Description |
|---|---|---|
| `silence_duration_ms` | 500 | Silence duration to trigger end-of-turn |
| `speech_duration_ms` | 80ms (semantic) | Minimum speech to detect speech start |
| `threshold` | 0.5 | Confidence required (0.0–1.0) |
| `remove_filler_words` | false | Remove "uh", "hmm", "ah" etc. to reduce false interrupts |
| `interrupt_response` | true | Allow barge-in (user interrupts agent) |
| `create_response` | true | Auto-generate response when turn ends |
| `eagerness` | auto | How quickly to respond: `low`, `auto`, `high` |

---

## 9. Server Events — All Event Types

The event loop `async for event in connection` receives these events:

### Server → Client events (ServerEventType enum)

| Event | Meaning | Lab 06 handling |
|---|---|---|
| `SESSION_UPDATED` | Session configured, agent connected | `start_capture()` — microphone ON |
| `CONVERSATION_ITEM_INPUT_AUDIO_TRANSCRIPTION_COMPLETED` | User speech transcribed | Print `👤 You: {transcript}` |
| `RESPONSE_AUDIO_TRANSCRIPT_DONE` | Agent's text response complete | Print `🤖 Agent: {transcript}` |
| `INPUT_AUDIO_BUFFER_SPEECH_STARTED` | User started speaking (barge-in) | `clear_playback_queue()` — stop agent audio |
| `INPUT_AUDIO_BUFFER_SPEECH_STOPPED` | User stopped speaking | Print `🤔 Thinking...` |
| `RESPONSE_AUDIO_DELTA` | Chunk of agent's audio response | `queue_audio(event.delta)` — buffer for playback |
| `RESPONSE_AUDIO_DONE` | Agent's full audio response received | Print `✓ Response complete` |
| `ERROR` | Error from service | Print error message |

### Client → Server events (sent by AudioProcessor)
```python
# Send microphone audio to Voice Live service
await connection.input_audio_buffer.append(audio=audio_base64)
```
Audio is **base64-encoded PCM16** sent continuously from the microphone callback.

---

## 10. Audio Subsystem — PyAudio Details

### Audio settings in Lab 06
```python
self.format = pyaudio.paInt16   # 16-bit PCM
self.channels = 1               # Mono
self.rate = 24000               # 24kHz sample rate
self.chunk_size = 1200          # 50ms chunks (1200 samples @ 24kHz)
```

### Why 24kHz?
- Azure TTS neural voices output at 24kHz
- 24kHz is the default for Voice Live PCM16 format
- 16kHz is also supported (`input_audio_sampling_rate=16000`)

### Microphone capture (async-safe pattern)
```python
def capture_callback(in_data, frame_count, time_info, status):
    audio_base64 = base64.b64encode(in_data).decode("utf-8")
    # PyAudio callback runs in separate thread — use run_coroutine_threadsafe
    asyncio.run_coroutine_threadsafe(
        self.connection.input_audio_buffer.append(audio=audio_base64),
        self.loop   # the asyncio event loop from the main async context
    )
    return (None, pyaudio.paContinue)
```

**Key pattern**: PyAudio callbacks run in threads, but Voice Live SDK is `asyncio`. Use `asyncio.run_coroutine_threadsafe()` to bridge them.

### Playback queue (speaker output)
```python
# Agent audio arrives in chunks (RESPONSE_AUDIO_DELTA events)
def queue_audio(self, audio_data):
    self.playback_queue.put(audio_data)   # thread-safe queue

# Playback callback drains the queue continuously
def playback_callback(in_data, frame_count, time_info, status):
    # Pull from queue, pad with silence if empty
    output = ...from playback_queue...
    return (output, pyaudio.paContinue)
```

### Barge-in (interrupt) handling
When `INPUT_AUDIO_BUFFER_SPEECH_STARTED` fires:
```python
self.audio_processor.clear_playback_queue()   # immediately stop current audio
```
This empties the queue so the agent stops talking and starts listening again.

---

## 11. Full Annotated Code (chat-client.py)

```python
import asyncio, base64, queue, os
from dotenv import load_dotenv
import pyaudio

# Async credential — AzureCliCredential (NOT the sync version)
from azure.identity.aio import AzureCliCredential
# Voice Live async SDK
from azure.ai.voicelive.aio import connect
from azure.ai.voicelive.models import (
    InputAudioFormat, Modality, OutputAudioFormat,
    RequestSession, ServerEventType,
    AudioNoiseReduction, AudioEchoCancellation,
    AzureSemanticVadMultilingual, AgentConfig
)

def main():
    load_dotenv()
    endpoint = os.environ.get("AZURE_VOICELIVE_ENDPOINT")    # BASE URL only
    agent_name = os.environ.get("AZURE_VOICELIVE_AGENT_ID")
    project_name = os.environ.get("AZURE_VOICELIVE_PROJECT_NAME")

    # AgentConfig = tells Voice Live which Foundry agent to use
    agent_config = AgentConfig({
        "agent_name": agent_name,
        "project_name": project_name
    })

    credential = AzureCliCredential()   # async credential

    assistant = VoiceAssistant(
        endpoint=endpoint,
        credential=credential,
        agent_config=agent_config
    )

    try:
        asyncio.run(assistant.start())   # run entire async app
    except KeyboardInterrupt:
        print("\n Goodbye!")


class VoiceAssistant:
    def __init__(self, endpoint, credential, agent_config):
        self.endpoint = endpoint
        self.credential = credential
        self.agent_config = agent_config

    async def start(self):
        try:
            # STEP 1: Open WebSocket to Voice Live
            async with connect(
                endpoint=self.endpoint,
                credential=self.credential,
                api_version="2026-01-01-preview",
                agent_config=self.agent_config
            ) as connection:
                self.connection = connection

                # STEP 2: Attach audio I/O handler
                self.audio_processor = AudioProcessor(connection)

                # STEP 3: Send session.update with audio settings
                await self.setup_session()

                # STEP 4: Start speaker playback thread
                self.audio_processor.start_playback()

                print("✅ Ready! Start speaking...")

                # STEP 5: Main event loop
                await self.process_events()

        finally:
            if hasattr(self, 'audio_processor'):
                self.audio_processor.shutdown()

    async def setup_session(self):
        session_config = RequestSession(
            modalities=[Modality.TEXT, Modality.AUDIO],
            input_audio_format=InputAudioFormat.PCM16,
            output_audio_format=OutputAudioFormat.PCM16,
            turn_detection=AzureSemanticVadMultilingual(),
            input_audio_echo_cancellation=AudioEchoCancellation(),
            input_audio_noise_reduction=AudioNoiseReduction(
                type="azure_deep_noise_suppression"
            )
        )
        await self.connection.session.update(session=session_config)

    async def process_events(self):
        async for event in self.connection:
            await self.handle_event(event)

    async def handle_event(self, event):
        if event.type == ServerEventType.SESSION_UPDATED:
            # Session ready → start microphone
            self.audio_processor.start_capture()

        elif event.type == ServerEventType.CONVERSATION_ITEM_INPUT_AUDIO_TRANSCRIPTION_COMPLETED:
            print(f'👤 You: {event.get("transcript", "")}')

        elif event.type == ServerEventType.RESPONSE_AUDIO_TRANSCRIPT_DONE:
            print(f'🤖 Agent: {event.get("transcript", "")}')

        elif event.type == ServerEventType.INPUT_AUDIO_BUFFER_SPEECH_STARTED:
            # Barge-in: user spoke → clear queued agent audio
            self.audio_processor.clear_playback_queue()

        elif event.type == ServerEventType.RESPONSE_AUDIO_DELTA:
            # Audio chunk arrived → buffer for playback
            self.audio_processor.queue_audio(event.delta)

        elif event.type == ServerEventType.ERROR:
            print(f"❌ Error: {event.error.message}")
```

---

## 12. .env Configuration

```ini
AZURE_VOICELIVE_ENDPOINT="https://{resource-name}.services.ai.azure.com"
AZURE_VOICELIVE_PROJECT_NAME="proj-default"
AZURE_VOICELIVE_AGENT_ID="chat-agent"
```

**CRITICAL**: `AZURE_VOICELIVE_ENDPOINT` must be the **resource base URL** (no `/api/projects/...` suffix). The SDK appends `/voice-live/realtime` itself.

---

## 13. Enabling Voice Mode in Foundry Portal

1. Go to the agent in the Foundry portal (`https://ai.azure.com`)
2. In the left pane under the model selector → enable **Voice mode**
3. **Configuration pane** (cog icon) → **Voice Live** section:
   - Select output voice (preview different voices)
   - Configure speech input/output options
4. Save the agent
5. Test in playground with **Start session** → speaks directly in browser

---

## 14. Supported Models for Voice Live

### Pricing tiers (exam-relevant)
| Tier | Models | Description |
|---|---|---|
| **Voice Live Pro** | `gpt-realtime`, `gpt-4o`, `gpt-4.1`, `gpt-5`, `gpt-5-chat` | Highest capability |
| **Voice Live Basic** | `gpt-realtime-mini`, `gpt-4o-mini`, `gpt-4.1-mini`, `gpt-5-mini` | Cost-balanced |
| **Voice Live Lite** | `gpt-5-nano`, `phi4-mm-realtime`, `phi4-mini` | Lowest cost |

> You don't choose a tier — **the tier is determined by the model you select**.

### Audio input transcription model options
| Model | Works with | Notes |
|---|---|---|
| `azure-speech` | All non-multimodal models | Default for non-realtime models |
| `mai-transcribe` (preview) | All non-multimodal models + agents | Microsoft AI transcription |
| `whisper-1` | `gpt-realtime`, `gpt-realtime-mini` only | OpenAI Whisper |
| `gpt-4o-transcribe` | `gpt-realtime`, `gpt-realtime-mini` only | GPT-4o based |

### Audio output voices
- **Standard neural voices**: `en-US-AvaNeural`, `en-GB-SoniaNeural`, etc.
- **HD voices**: `en-US-Ava:DragonHDLatestNeural` — higher quality, limited regions
- **Azure-realtime native voices**: `ava`, `andrew`, `elsa` etc. — used with `azure-realtime` model
- **Custom voice**: your own trained voice (limited access, separate billing)

---

## 15. Key Comparison: Lab 05 vs Lab 06

| Aspect | Lab 05 (Speech MCP) | Lab 06 (Voice Live) |
|---|---|---|
| Interaction mode | **Text** (type/read) | **Voice** (speak/listen) |
| Protocol | REST (HTTPS) | **WebSocket** (WSS) |
| Speech feature | STT + TTS separately via MCP | STT + LLM + TTS **unified** |
| Latency model | Batch (request-response) | **Real-time streaming** |
| SDK | `azure-ai-projects` | `azure-ai-voicelive` |
| Client method | `openai_client.responses.create()` | `connect()` async context manager |
| Audio handling | No audio code (MCP saves to blob) | `PyAudio` (microphone + speakers) |
| Agent type | Prompt agent via Responses API | Prompt agent via Voice Live WebSocket |
| Interrupt support | No | **Yes** (barge-in detection) |
| Endpoint used | Project endpoint (`/api/projects/...`) | Resource base URL (`/voice-live/realtime`) |

---

## 16. Key Scenarios for Voice Live (Exam Examples)

| Scenario | Why Voice Live |
|---|---|
| Contact center voice bots | Real-time, low latency, interrupt handling |
| Automotive in-car assistants | Hands-free, offline-capable with containers |
| Education / language tutoring | Pronunciation feedback, real-time responses |
| HR interview tools | Natural conversation flow |
| Public service kiosks | Multilingual VAD, accessibility |

---

## 17. Voice Live Features Summary (Exam Checklist)

- **Noise suppression** (`azure_deep_noise_suppression`) — clears background noise
- **Echo cancellation** (`server_echo_cancellation`) — prevents mic from hearing speaker output
- **Semantic VAD** — end-of-turn by meaning, not just silence
- **Barge-in / interruption** — user can cut off agent mid-sentence
- **Multilingual** — 140+ locales for STT, 600+ voices in 150+ locales
- **Avatar integration** — text-to-speech avatar (standard or custom, video output)
- **Function calling** — agent can call external APIs/tools
- **Viseme output** — facial animation data synchronized with audio
- **Audio timestamps** — word-level timing data for captions

---

## 18. Lifecycle of One Conversation Turn

```
1. Microphone captures audio → PyAudio callback fires (50ms chunks)
2. Chunk encoded to base64 → sent to Voice Live via WebSocket
   (connection.input_audio_buffer.append)

3. Voice Live detects speech start
   → SERVER EVENT: INPUT_AUDIO_BUFFER_SPEECH_STARTED → 🎤 Listening...

4. User finishes speaking (semantic VAD detects end-of-turn)
   → SERVER EVENT: INPUT_AUDIO_BUFFER_SPEECH_STOPPED → 🤔 Thinking...

5. Voice Live transcribes audio (Azure Speech STT)
   → SERVER EVENT: CONVERSATION_ITEM_INPUT_AUDIO_TRANSCRIPTION_COMPLETED
   → 👤 You: "How does speech recognition work?"

6. GPT-5 generates response

7. TTS generates audio chunks
   → SERVER EVENT: RESPONSE_AUDIO_DELTA (repeated) → audio queued for playback

8. Speakers play queued audio continuously via PyAudio

9. Full response done
   → SERVER EVENT: RESPONSE_AUDIO_TRANSCRIPT_DONE → 🤖 Agent: "..."
   → SERVER EVENT: RESPONSE_AUDIO_DONE → ✓ Response complete

10. If user speaks during step 8 (barge-in):
    → SERVER EVENT: INPUT_AUDIO_BUFFER_SPEECH_STARTED
    → clear_playback_queue() → agent stops speaking
    → go back to step 3
```

---

## 19. Common Errors & Fixes

| Error | Cause | Fix |
|---|---|---|
| `400 Invalid response status` on WebSocket | Full project URL in endpoint | Use resource BASE URL only (remove `/api/projects/...`) |
| `Failed to establish WebSocket connection` | Wrong API version | Use `api-version=2026-01-01-preview` |
| `DefaultAzureCredential failed` | Not logged into Azure CLI | Run `az login` |
| `ModuleNotFoundError: azure.ai.voicelive` | Package not installed | `pip install azure-ai-voicelive==1.2.0b4 --pre` |
| Echo / agent hears itself | Echo cancellation not enabled | Add `AudioEchoCancellation()` to session config |
| Agent responds too fast (cuts off user) | VAD threshold too low | Use `AzureSemanticVadMultilingual()` instead of `server_vad` |
| No audio captured | Microphone started before SESSION_UPDATED | Microphone must start INSIDE the `SESSION_UPDATED` event handler |

---

## 20. Microsoft Learn References

- [Voice Live API overview](https://learn.microsoft.com/azure/ai-services/speech-service/voice-live)
- [How to use Voice Live API](https://learn.microsoft.com/azure/ai-services/speech-service/voice-live-how-to)
- [Voice Live agents quickstart](https://learn.microsoft.com/azure/ai-services/speech-service/voice-live-agents-quickstart)
- [Voice Live API reference (2026-04-10)](https://learn.microsoft.com/azure/ai-services/speech-service/voice-live-api-reference-2026-04-10)
- [Azure OpenAI Realtime API events reference](https://learn.microsoft.com/azure/ai-foundry/openai/realtime-audio-reference)
- [AzureCliCredential (async)](https://learn.microsoft.com/python/api/azure-identity/azure.identity.aio.azureclicredential)
- [Voice Live language support](https://learn.microsoft.com/azure/ai-services/speech-service/voice-live-language-support)
