# Lab 02 — Language Agent | Exam Notes

## What's New vs Lab 01
- Lab 01 = **direct SDK call** to Azure Language
- Lab 02 = **AI Agent** that uses Azure Language as a tool via **MCP** (Model Context Protocol)
- Your code talks to an agent → agent decides when to call Azure Language

---

## Architecture
```
Python App
    ↓ AIProjectClient
Foundry Agent (Text-Analysis-Agent)
    ↓ MCP Tool
Azure Language Service
    ↓
NER / PII / Language Detection
```

---

## Service & SDK
- **SDK Package**: `azure-ai-projects`
- **Client Class**: `AIProjectClient`
- **Endpoint Format**: `https://{resource}.services.ai.azure.com/api/projects/{project-name}` *(includes project path!)*

---

## Authentication
```python
from azure.identity import DefaultAzureCredential
from azure.ai.projects import AIProjectClient

project_client = AIProjectClient(
    endpoint=foundry_endpoint,       # FULL project endpoint
    credential=DefaultAzureCredential(),
)
```

---

## Core Code Pattern

### Step 1 — Get Project Client
```python
project_client = AIProjectClient(
    endpoint=foundry_endpoint,
    credential=DefaultAzureCredential(),
)
```

### Step 2 — Get OpenAI Client (scoped to Foundry project)
```python
openai_client = project_client.get_openai_client()
```
- This is NOT a raw OpenAI client — it's scoped to your Foundry project

### Step 3 — Call Agent by Name
```python
response = openai_client.responses.create(
    input=[{"role": "user", "content": prompt}],
    extra_body={"agent_reference": {"name": agent_name, "type": "agent_reference"}},
)
print(response.output_text)
```
- `agent_name` = `"Text-Analysis-Agent"` — **CASE SENSITIVE**
- `output_text` = the agent's final natural language response

---

## MCP — Model Context Protocol

| What | Detail |
|------|--------|
| What is MCP | Standard protocol for agents to connect external tools |
| Azure Language MCP endpoint | `https://{resource}.cognitiveservices.azure.com/language/mcp?api-version=2025-11-15-preview` |
| Authentication options | Key-based (`Ocp-Apim-Subscription-Key`) or Entra ID |
| Who calls MCP | The **agent** — your Python code does NOT call it directly |
| Tool approval | Can be set to "Always auto-approve" in agent config |

---

## Endpoint Differences — Critical!

| Lab | Client | Endpoint |
|-----|--------|----------|
| Lab 01 | `TextAnalyticsClient` | `https://{resource}.services.ai.azure.com` |
| Lab 02 | `AIProjectClient` | `https://{resource}.services.ai.azure.com/api/projects/{name}` |

---

## Exam Key Facts

| Fact | Answer |
|------|--------|
| SDK package | `azure-ai-projects` |
| Client class | `AIProjectClient` |
| Endpoint format | Must include `/api/projects/{project-name}` |
| Agent name | Case-sensitive in `agent_reference` |
| `output_text` | Property to get agent's text response |
| `get_openai_client()` | Returns Foundry-scoped client, NOT raw OpenAI |
| Input format | `[{"role": "user", "content": prompt}]` — list of message dicts |
| MCP caller | The agent — not your Python code |

---

## Direct SDK vs Agent Comparison

| | Direct (Lab 01) | Agent (Lab 02) |
|--|--|--|
| Input | Raw text string | Natural language prompt |
| Who calls NER | Your code | The agent (LLM decides) |
| Output | Structured JSON | Natural language response |
| Flexibility | Fixed operations | Agent chains multiple tools |
| User experience | Programmatic | Conversational |

---

## Exam Traps
- `AIProjectClient` endpoint ≠ `TextAnalyticsClient` endpoint (different formats)
- Agent name is **case-sensitive**: `Text-Analysis-Agent` ≠ `text-analysis-agent`
- `get_openai_client()` ≠ raw OpenAI — it's Foundry project scoped
- MCP is called by the **agent**, not your application code
- Tool auto-approve must be configured in Foundry portal per agent

---

## Exam Scenario
> *"Build a conversational app where users ask in plain English to analyze text for PII"*

**Answer**: Create Foundry agent → connect Azure Language MCP tool → use `AIProjectClient` + `responses.create()` with `agent_reference`
