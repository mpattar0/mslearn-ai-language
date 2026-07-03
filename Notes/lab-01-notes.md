# Lab 01 — Analyze Text | Exam Notes

## Service & SDK
- **Service**: Azure AI Language (part of Azure AI Services)
- **SDK Package**: `azure-ai-textanalytics`
- **Client Class**: `TextAnalyticsClient`
- **Endpoint Format**: `https://{resource-name}.services.ai.azure.com` *(NO project path)*

---

## Authentication
```python
from azure.identity import DefaultAzureCredential
credential = DefaultAzureCredential()
ai_client = TextAnalyticsClient(endpoint=foundry_endpoint, credential=credential)
```
- Uses `DefaultAzureCredential` — relies on `az login` session
- Never hardcode keys — use `.env` + `load_dotenv()`

---

## 3 Core API Calls

### 1. Language Detection
```python
detectedLanguage = ai_client.detect_language(documents=[text])[0]
detectedLanguage.primary_language.name          # → "English"
detectedLanguage.primary_language.iso6391_name  # → "en"
detectedLanguage.primary_language.confidence_score  # → 1.0
```

### 2. Named Entity Recognition (NER)
```python
entities = ai_client.recognize_entities(documents=[text])[0].entities
entity.text        # → "London"
entity.category    # → "Location"
entity.subcategory # → "City"
entity.confidence_score  # → 0.99
entity.offset      # char position in text
entity.length      # char length of entity
```
**Entity Categories**: `Person`, `Location`, `Organization`, `DateTime`, `Event`, `Quantity`, `URL`, `Email`, `PhoneNumber`

### 3. PII Detection & Redaction
```python
pii_result = ai_client.recognize_pii_entities(documents=[text])[0]
pii_result.entities      # list of PII entities found
pii_result.redacted_text # text with PII replaced by ***
```
**PII Categories**: `Email`, `PhoneNumber`, `Address`, `Person`, `SSN`, `CreditCardNumber`, `IPAddress`

---

## Key Pattern — Response is ALWAYS a List
```python
ai_client.detect_language(documents=[text])[0]  # always index [0] for single doc
```
- API accepts a **list** of documents → returns a **list** of results
- For single doc: pass `[text]` → get result at `[0]`
- For batch: pass `["text1", "text2"]` → iterate results

---

## Exam Key Facts

| Fact | Answer |
|------|--------|
| SDK package | `azure-ai-textanalytics` |
| Client class | `TextAnalyticsClient` |
| Endpoint format | Resource URL only (no `/api/projects/`) |
| Response type | Always a list — use `[0]` for single doc |
| `redacted_text` available in | `recognize_pii_entities()` ONLY (not NER) |
| PII vs NER | NER = general entities; PII = privacy-sensitive only |
| Language detection returns | Name, ISO code, confidence score |
| Free tier doc limit | Up to 5 documents per batch |

---

## Exam Traps
- `recognize_entities()` ≠ `recognize_pii_entities()` — different methods
- Endpoint must NOT include `/api/projects/{name}` — strip it
- `DefaultAzureCredential` needs `az login` first in dev environment
- PII redaction (`redacted_text`) only comes from `recognize_pii_entities()`
- Always `[0]` after the call for single document results

---

## Real-World Scenario
Travel agency processes multilingual hotel reviews:
1. **Detect language** → know what language each review is in
2. **Extract entities** → find mentioned hotels, cities, landmarks
3. **Redact PII** → remove guest names/emails before publishing
