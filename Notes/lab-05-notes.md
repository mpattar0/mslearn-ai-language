# Lab 05 – Use Azure Speech in an Agent (MCP Server)

> **Exam relevance**: AI-102 / AI-103 — Azure AI Speech, Azure AI Foundry, MCP, Agent Tools, Authentication, Blob Storage SAS

---

## 1. What This Lab Builds

```
User CLI prompt
     │
     ▼
speech-client.py (Python)
     │ AIProjectClient → DefaultAzureCredential
     │ openai_client.responses.create(agent_reference)
     ▼
Microsoft Foundry Project  (speech-agent)
     │
     ├── Model: gpt-5 (reasoning)
     ├── Instructions: "use Azure AI Speech tool to transcribe and generate speech"
     └── Tool: Azure Speech MCP Server
              │
              ├── Text-to-Speech → .wav → saved to Azure Blob (SAS URL)
              └── Speech-to-Text → transcribe audio from any URL
```

**Goal**: Build a Python client that sends natural-language prompts to a Foundry agent. The agent uses the **Azure Speech MCP Server** to synthesize or transcribe audio autonomously.

---

## 2. Azure Speech Service — Deep Dive

### What is Azure Speech?
Azure Speech (part of Foundry Tools, formerly Azure AI Services) provides:
- **Speech-to-Text (STT)** — audio → text
- **Text-to-Speech (TTS)** — text → audio (neural voices)
- **Speech Translation** — audio in language A → text/audio in language B
- **Speaker Recognition** — identify or verify speakers
- **Pronunciation Assessment** — evaluate spoken language quality
- **Voice Live** — real-time conversational AI voice interface

### Speech-to-Text Modes
| Mode | Use case |
|---|---|
| **Real-time** | Live streaming audio |
| **Fast transcription** | Pre-recorded files, fastest result |
| **Batch transcription** | Large volumes, asynchronous, REST API |
| **LLM speech (preview)** | LLM-enhanced quality, contextual understanding |

### Text-to-Speech Neural Voices
- **Standard neural voices** — prebuilt, no training needed (e.g., `en-GB-SoniaNeural`, `en-US-JennyNeural`)
- **Custom neural voice** — fine-tuned on your own audio data, brand-specific
- Controlled via **SSML** (Speech Synthesis Markup Language) for pitch, rate, volume, pauses

### Key Neural Voice Name Format
```
{language}-{Locale}-{VoiceName}Neural
Example: en-GB-SoniaNeural  (British English, Sonia voice)
         en-US-JennyNeural  (US English, Jenny voice)
         es-ES-ElviraNeural (Spanish, Elvira voice)
```

### Speech Service Endpoints
| Feature | Endpoint pattern |
|---|---|
| STT (standard) | `https://{region}.stt.speech.microsoft.com` |
| TTS (neural) | `https://{region}.tts.speech.microsoft.com` |
| Custom Voice / MCP | `https://{resource-name}.cognitiveservices.azure.com/` |

### MCP Endpoint for Azure Speech in Foundry
```
https://{foundry-resource-name}.cognitiveservices.azure.com/speech/mcp?api-version=2025-11-15-preview
```

### Integration methods
- **Speech SDK** — multi-platform (Python, C#, JS, Java); most features
- **Speech CLI** (`spx`) — command-line, no code needed
- **REST APIs** — used for batch transcription, simpler scenarios

---

## 3. Model Context Protocol (MCP)

### What is MCP?
**Model Context Protocol (MCP)** is an open standard for connecting AI agents to external **tool servers** in a structured, interoperable way. Instead of building custom integrations per tool, MCP defines a uniform protocol.

### How it Works in Foundry
1. An MCP server exposes **tools** (functions the agent can call)
2. The agent's model receives tool definitions alongside the conversation
3. When needed, the model decides to call a tool → Foundry calls the MCP server
4. The MCP server returns results → the model uses them in its reply

### Azure Speech MCP Server
- Provides tools like `synthesize_speech` and `transcribe_audio`
- Connected in Foundry Portal under **Tools → Catalog → Azure Speech MCP Server**
- Requires:
  - Remote MCP endpoint URL (the Speech resource URL above)
  - **Authentication**: Key-based (`Ocp-Apim-Subscription-Key` = API key)
  - **`X-Blob-Container-Url`**: SAS URL of the blob container where TTS audio is saved

### MCP Tool Approval in Playground
When the agent first calls an MCP tool:
- The playground shows a **permission prompt**
- You select: **"Always approve all Azure Speech MCP Server tools"**
- In production code this is handled automatically via the agent configuration

### Toolbox (Foundry concept)
Foundry's **Toolbox** is a centralized, versioned collection of MCP-compatible tools that any agent can consume through a single endpoint. Versioning allows safe upgrades.

---

## 4. Microsoft Foundry — Platform Concepts

### Foundry Resource vs Project
| Concept | Description |
|---|---|
| **Foundry Resource** | Azure resource (like a "hub") — holds API keys, region, billing |
| **Foundry Project** | Logical workspace within a resource — organizes agents, models, connections |

### Foundry Project Endpoint
```
https://{resource-name}.services.ai.azure.com/api/projects/{project-name}
```
This single endpoint gives access to:
- Model inference (via Responses API)
- Platform tools (file search, code interpreter, web search, MCP servers)
- Agent management

### Foundry Portal Key Info
From the project home page you get:
- **API Key** — for key-based auth
- **Project Endpoint** — for SDK connection
- **OpenAI Endpoint** — for direct Azure OpenAI calls

### Foundry RBAC Roles
| Role | Purpose |
|---|---|
| **Foundry User** | Least privilege for development |
| **Foundry Project Manager** | Manage projects |
| **Foundry Owner** | Full control |
| **Contributor/Owner** | Subscription-level permissions |

---

## 5. Foundry Agents — Types and Architecture

### Agent = Model + Instructions + Tools
Every Foundry agent has three components:
1. **Model** — from Foundry catalog (GPT-5, GPT-4o, Llama, DeepSeek, etc.)
2. **Instructions** — defines the agent's personality, goals, constraints
3. **Tools** — gives access to data/actions (search, speech, files, APIs)

### Agent Types
| Type | Description | Best For |
|---|---|---|
| **Prompt Agent** | Configured in portal (no code). Foundry runs it. | Fast start, production agents without custom logic |
| **Hosted Agent** | Your own code (Agent Framework, LangGraph, OpenAI Agents SDK) packaged as container | Custom orchestration, multi-agent, webhooks |

### In Lab 05
The **speech-agent** is a **Prompt Agent**:
- Created in Foundry Portal with a model and instructions
- Tools connected via MCP server
- Called from client code using `agent_reference`

---

## 6. Azure AI Projects SDK

### Installation
```bash
pip install "azure-ai-projects>=2.0.0"
pip install azure-identity
pip install python-dotenv
```

Lab 05 uses version: `azure-ai-projects==2.0.0b4` (beta)

### Two Client Types

#### 1. Project Client (Foundry-native operations)
```python
from azure.ai.projects import AIProjectClient
from azure.identity import DefaultAzureCredential

project_client = AIProjectClient(
    endpoint="https://{resource}.services.ai.azure.com/api/projects/{project}",
    credential=DefaultAzureCredential(),
)
```
Used for: listing connections, retrieving project properties, enabling tracing.

#### 2. OpenAI-compatible Client (calling agents, models, tools)
```python
openai_client = project_client.get_openai_client()
```
Routes through: `{project_endpoint}/openai/v1/responses`
Gives access to: Foundry models + platform tools (including MCP servers)

### Calling an Agent via Responses API
```python
response = openai_client.responses.create(
    input=[{"role": "user", "content": prompt}],
    extra_body={
        "agent_reference": {
            "name": agent_name,   # "speech-agent"
            "type": "agent_reference"
        }
    },
)
print(response.output_text)
```

**Key parameters**:
- `input` — the conversation messages (list of role/content dicts)
- `extra_body["agent_reference"]` — tells Foundry which agent to invoke
- `response.output_text` — the agent's text reply

### Responses API vs Chat Completions API
| API | Use when |
|---|---|
| **Responses API** | Agents, platform tools, Foundry-specific features |
| **Chat Completions** | Direct model calls, OpenAI compatibility, embeddings |

---

## 7. Authentication — DefaultAzureCredential

### What is DefaultAzureCredential?
From the `azure-identity` package. It tries a **chain of credential sources** in order:
1. Environment variables (`AZURE_CLIENT_ID`, `AZURE_CLIENT_SECRET`, `AZURE_TENANT_ID`)
2. Workload identity (Kubernetes)
3. Managed identity
4. Visual Studio / VS Code credentials
5. **Azure CLI** — `az login` ← used in this lab
6. Azure PowerShell
7. Interactive browser

### Why used in Lab 05?
- More secure than storing API keys in code
- Works locally via `az login`
- Works in production via Managed Identity (no credential change needed)

### Alternative: API Key auth
```python
# Only for /openai/v1 endpoint, not recommended for production
client = openai.AzureOpenAI(api_key="your-key")
```

### Authentication Troubleshooting
```bash
az account show          # verify you are logged in
az login                 # sign in
az account set --subscription "your-subscription-id"  # switch tenant/sub
```

---

## 8. Azure Blob Storage + SAS Tokens

### Why Blob Storage in Lab 05?
The **Azure Speech MCP server generates audio files** (TTS). These files need to be:
- Stored somewhere accessible
- Linked back to the user (as a URL in the agent's response)

Azure Blob Storage with a **SAS URL** provides this.

### What is a SAS (Shared Access Signature)?
A SAS is a URI token that grants **limited, time-bound, permission-scoped** access to a storage resource — without exposing the account key.

### SAS Token Types
| Type | Secured by | Notes |
|---|---|---|
| **User Delegation SAS** | Microsoft Entra (Entra ID) | **Most secure — recommended** |
| **Service SAS** | Storage account key | Access to one service only |
| **Account SAS** | Storage account key | Access to multiple services |

### SAS URI Structure
```
https://storageaccount.blob.core.windows.net/files?sv=2022-11-02
  &ss=b&srt=sco&sp=rwdlacupiytfx&se=2025-12-31T23:59:00Z
  &st=2025-01-01T00:00:00Z&spr=https&sig=xxxxx
```

### SAS Permissions in Lab 05
The lab requires these permissions on the `files` container:
- **Read** — read generated audio file
- **Add** — add new blobs
- **Create** — create new blobs
- **Write** — overwrite blobs
- **List** — list blobs in container

### SAS Best Practices (Exam-relevant)
1. **Use HTTPS only** — never HTTP (man-in-the-middle risk)
2. **Use User Delegation SAS** when possible (Entra-secured)
3. **Short expiry times** — minimize damage if token leaks
4. **Least privilege** — only grant needed permissions
5. **Start time 15 min in the past** — avoids clock skew failures
6. **Have a revocation plan** — stored access policies allow revocation without regenerating keys
7. **Use Azure Monitor** — to detect authorization failures
8. **Set a SAS expiration policy** on the storage account

### X-Blob-Container-Url Header
In the MCP tool configuration, `X-Blob-Container-Url` is set to the **SAS URL** of the container. The speech MCP server uses this to upload generated audio files and return a link.

---

## 9. Full Architecture Flow (Lab 05)

```
Step 1: az login
        ↓
Step 2: python speech-client.py
        ↓
Step 3: load .env → FOUNDRY_ENDPOINT, AGENT_NAME
        ↓
Step 4: AIProjectClient(endpoint, DefaultAzureCredential)
        ↓
Step 5: project_client.get_openai_client()
        ↓
Step 6: User types: "Synthesize '...' as speech using voice 'en-GB-SoniaNeural'"
        ↓
Step 7: openai_client.responses.create(input, agent_reference={"name":"speech-agent"})
        ↓
Step 8: Foundry routes to speech-agent
        ↓
Step 9: GPT-5 model reasons → decides to call Azure Speech MCP tool
        ↓
Step 10: Foundry calls MCP server endpoint with Ocp-Apim-Subscription-Key
         - synthesize_speech(text, voice) → uploads .wav to Blob
         - Returns blob URL to agent
        ↓
Step 11: Agent wraps URL in response text → returned to client
        ↓
Step 12: print(f"speech-agent: {response.output_text}")  ← clickable URL
```

---

## 10. Configuration File (.env)

```ini
FOUNDRY_ENDPOINT=https://{resource}.services.ai.azure.com/api/projects/{project-name}
AGENT_NAME=speech-agent
```

Loaded with `python-dotenv`:
```python
from dotenv import load_dotenv
load_dotenv()
foundry_endpoint = os.getenv('FOUNDRY_ENDPOINT')
agent_name = os.getenv('AGENT_NAME')
```

---

## 11. Lab 05 Code — Full Annotated Version

```python
from dotenv import load_dotenv
import os
from azure.identity import DefaultAzureCredential   # Entra-based auth
from azure.ai.projects import AIProjectClient        # Foundry SDK

def main():
    os.system('cls' if os.name == 'nt' else 'clear')

    load_dotenv()
    foundry_endpoint = os.getenv('FOUNDRY_ENDPOINT')
    agent_name = os.getenv('AGENT_NAME')

    # Connect to Foundry project (no hardcoded keys!)
    project_client = AIProjectClient(
        endpoint=foundry_endpoint,
        credential=DefaultAzureCredential(),
    )

    # Get OpenAI-compatible client (routes to /openai/v1/responses)
    openai_client = project_client.get_openai_client()

    while True:
        prompt = input("User prompt (or 'quit'): ")
        if prompt == "quit" or len(prompt) == 0:
            break

        # Call the agent — extra_body routes to named prompt agent
        response = openai_client.responses.create(
            input=[{"role": "user", "content": prompt}],
            extra_body={
                "agent_reference": {
                    "name": agent_name,        # "speech-agent"
                    "type": "agent_reference"
                }
            },
        )
        print(f"{agent_name}: {response.output_text}")

if __name__ == "__main__":
    main()
```

---

## 12. Key Concepts Comparison Table

| Concept | Lab 05 Usage | Exam Key Point |
|---|---|---|
| `AIProjectClient` | Connect to Foundry project | Single endpoint for all Foundry APIs |
| `DefaultAzureCredential` | Auth via `az login` | Chains multiple credential sources |
| `get_openai_client()` | Get OpenAI-compatible client | Routes to `/openai/v1/responses` |
| `responses.create()` | Call the agent | Foundry-native agents API |
| `agent_reference` | Route to named agent | In `extra_body`, not a model param |
| `output_text` | Agent's text response | Property on Responses API result |
| MCP Server | Azure Speech tools | Protocol for agent tool integration |
| SAS Token | Blob storage access | Time-limited, scoped permissions |
| `Ocp-Apim-Subscription-Key` | Authenticate to MCP server | API key passed as HTTP header |
| `X-Blob-Container-Url` | Where TTS audio is saved | SAS URL of Azure Blob container |
| Neural voice name | `en-GB-SoniaNeural` | `{lang}-{Locale}-{Name}Neural` |

---

## 13. Exam Pitfalls & Watch-Outs

1. **Agent name is case-sensitive** — `speech-agent` ≠ `Speech-Agent` in the `.env` file
2. **`agent_reference` goes in `extra_body`**, NOT as the `model` parameter
3. **Responses API ≠ Chat Completions API** — different endpoint, different features
4. **SAS token vs API key** — SAS is for Blob Storage; API key is for cognitive/speech services
5. **`DefaultAzureCredential` requires `az login`** locally — not the project API key
6. **MCP endpoint version**: `?api-version=2025-11-15-preview` — exact version matters
7. **TTS output is a URL link** in the response, not raw audio bytes
8. **Batch transcription uses REST API**, not SDK — different from real-time STT
9. **User Delegation SAS is most secure** — uses Entra ID, not account key
10. **SAS start time** — set at least 15 minutes in the past to avoid clock skew

---

## 14. Related Services & Surrounding Topics

### Azure AI Foundry Portal URL
```
https://ai.azure.com
```

### Azure Portal URL
```
https://portal.azure.com
```

### Speech Service Region-based Endpoints (Exam)
- Real-time STT: `https://{region}.stt.speech.microsoft.com`
- Neural TTS: `https://{region}.tts.speech.microsoft.com`
- Batch/Custom: `https://{resource-name}.cognitiveservices.azure.com/`

### SSML (Speech Synthesis Markup Language)
Controls TTS voice characteristics:
```xml
<speak version="1.0" xmlns="http://www.w3.org/2001/10/synthesis" xml:lang="en-GB">
  <voice name="en-GB-SoniaNeural">
    <prosody rate="slow" pitch="high">
      To be or not to be
    </prosody>
  </voice>
</speak>
```

### Custom Speech Models
When base model is insufficient (ambient noise, domain jargon), train custom models with:
- Acoustic data
- Language data
- Pronunciation data

### Voice Live
Real-time conversational AI voice interface — humanlike two-way conversation. Different from standard TTS (batch/single request).

### Other Tools Available in Foundry Catalog (exam context)
| Tool | Purpose |
|---|---|
| Azure Speech MCP | STT / TTS via agent |
| Azure DevOps MCP | Code repos, pipelines |
| File Search | Retrieval from uploaded files |
| Code Interpreter | Execute Python code |
| Web Search (preview) | Live web results |
| SharePoint, WorkIQ, Fabric IQ | Microsoft 365 data integration |
| Memory (preview) | Persistent agent memory |

---

## 15. Lifecycle Summary for Exam

```
1. Create Azure Storage Account
   └── Create "files" container
   └── Generate SAS token (Read, Add, Create, Write, List)
   └── Copy SAS URL

2. Create Foundry Project (https://ai.azure.com)
   └── Note: API Key, Project Endpoint

3. Create Agent (speech-agent)
   └── Model: gpt-5
   └── Instructions: use Azure AI Speech tool

4. Connect Azure Speech MCP Server Tool
   └── Endpoint: https://{resource}.cognitiveservices.azure.com/speech/mcp?api-version=2025-11-15-preview
   └── Auth: Key-based (Ocp-Apim-Subscription-Key = API Key)
   └── X-Blob-Container-Url = SAS URL

5. Test in Playground
   └── TTS prompt → approve → get audio link
   └── STT prompt → approve → get transcription

6. Write Client App (speech-client.py)
   └── AIProjectClient + DefaultAzureCredential
   └── get_openai_client()
   └── responses.create(input, agent_reference)

7. Run: az login → python speech-client.py
```

---

## 16. Microsoft Learn References

- [What is Azure Speech?](https://learn.microsoft.com/azure/ai-services/speech-service/overview)
- [Speech to text overview](https://learn.microsoft.com/azure/ai-services/speech-service/speech-to-text)
- [Text to speech overview](https://learn.microsoft.com/azure/ai-services/speech-service/text-to-speech)
- [What is Foundry Agent Service?](https://learn.microsoft.com/azure/ai-foundry/agents/overview)
- [Microsoft Foundry SDKs and Endpoints](https://learn.microsoft.com/azure/foundry/how-to/develop/sdk-overview)
- [Azure Storage SAS overview](https://learn.microsoft.com/azure/storage/common/storage-sas-overview)
- [DefaultAzureCredential (azure-identity)](https://learn.microsoft.com/python/api/azure-identity/azure.identity.defaultazurecredential)
- [SSML for TTS](https://learn.microsoft.com/azure/ai-services/speech-service/speech-synthesis-markup)
