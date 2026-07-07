# AI-102 / AI-103 Lab Quiz — Reusable Prompt

Copy this into any chat session to start the same quiz activity on any lab set.

---

## PROMPT TO PASTE

```
I am preparing for the Microsoft AI-102 / AI-103 exam.
I have completed hands-on labs covering:
- Azure AI Language (TextAnalyticsClient)
- Azure AI Foundry Agents (AIProjectClient, Responses API)
- Azure OpenAI speech (TTS + STT with Whisper)
- Azure Speech SDK (SpeechConfig, SpeechSynthesizer, SpeechRecognizer, SpeechTranslationConfig)
- Azure Speech Voice Live API (real-time WebSocket voice agent)
- Azure Translator (TextTranslationClient)

Please quiz me on these topics one question at a time.

Rules:
1. Ask ONE question at a time — wait for my answer before the next
2. Mix question types: Multiple Choice, Hotspot (fill-in-blank), True/False, Scenario
3. Cover: endpoints, SDK methods, parameter names, credential patterns, class names, result properties, audio formats, event types, VAD types, pricing rules, language codes
4. Make all answer options equally plausible — avoid obvious distractors
5. After my answer: tell me RIGHT or WRONG + brief explanation of why, including the traps
6. Keep score and show it after each answer
7. At the end of all questions give me a summary of wrong answers for review
8. Include THEORY questions too (not just code) — e.g. "What does Voice Live use instead of chaining STT+LLM+TTS?"

Start with Question 1.
```

---

## Key Topics to Cover Per Lab

| Lab | Must-Know Topics |
|---|---|
| Lab 01 | TextAnalyticsClient, detect_language, recognize_entities, recognize_pii_entities, redacted_text, analyze_sentiment, endpoint format |
| Lab 02 | AIProjectClient, get_openai_client(), responses.create(), input=, output_text, agent_reference, extra_body, case-sensitivity |
| Lab 03 | AzureOpenAI, azure_endpoint=, get_bearer_token_provider, scope string, audio.speech (TTS), audio.transcriptions (STT), voice names (alloy/nova/etc), response_format |
| Lab 04 | SpeechConfig, token_credential=, SpeechSynthesizer, SpeechRecognizer, AudioConfig, AudioOutputConfig, speak_text_async, recognize_once_async, ResultReason, voice name format |
| Lab 05 | AIProjectClient + MCP, agent_reference, SAS token, X-Blob-Container-Url, Ocp-Apim-Subscription-Key, MCP endpoint format |
| Lab 06 | Voice Live, connect(), AgentConfig, RequestSession, AzureSemanticVadMultilingual, ServerEventType, SESSION_UPDATED, RESPONSE_AUDIO_DELTA, base endpoint (no /api/projects), AzureCliCredential aio |
| Lab 07 | TextTranslationClient, InputTextItem, translate(body, to_language), detected_language, SpeechTranslationConfig, add_target_language, short codes vs locale codes, recognize_once_async, translations dict |

## Theory Questions to Include

1. What 3 components does Voice Live unify into one WebSocket API?
2. What is the difference between Responses API and Chat Completions API?
3. What type of SAS is most secure and why?
4. What happens when `INPUT_AUDIO_BUFFER_SPEECH_STARTED` fires in Voice Live?
5. What VAD type supports all models AND multilingual detection?
6. What pricing tier does gpt-5 fall into for Voice Live?
7. What is the 3rd+ target language pricing rule for speech translation?
8. What is the difference between `redacted_text` and `entities` in PII detection?
9. What does `recognize_once_async()` do vs continuous recognition?
10. Why does TextTranslationClient not need `from_language=` parameter?
