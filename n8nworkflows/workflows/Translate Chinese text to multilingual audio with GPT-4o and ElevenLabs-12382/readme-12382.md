Translate Chinese text to multilingual audio with GPT-4o and ElevenLabs

https://n8nworkflows.xyz/workflows/translate-chinese-text-to-multilingual-audio-with-gpt-4o-and-elevenlabs-12382


# Translate Chinese text to multilingual audio with GPT-4o and ElevenLabs

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** Receive Chinese text via an HTTP webhook, validate it contains Chinese characters, translate it into multiple target languages using **GPT-4o**, generate **audio (MP3)** for each translation via **ElevenLabs Text-to-Speech**, run an AI-based translation quality assessment, and return the resulting audio files via the webhook response.

**Typical use cases:** language learning platforms, content localization/marketing, multilingual voiceover generation, automated international communication.

### 1.1 Input Ingestion & Configuration
Receives POST requests and defines workflow-wide settings (target languages, ElevenLabs voice/model, QA thresholds, retry limits).

### 1.2 Input Validation (Chinese detection)
Ensures there is non-empty text and it contains Chinese characters before invoking AI.

### 1.3 Multilingual Translation (GPT-4o + structured JSON)
Uses a LangChain Agent powered by GPT-4o with a **Structured Output Parser** to return a predictable JSON payload containing translations.

### 1.4 Translation Fan-out + Audio Synthesis (ElevenLabs)
Splits the translation list into per-language items and generates speech audio for each.

### 1.5 Quality Review & Output Control (GPT-4o QA gate)
Scores each translation; if it passes the threshold, audio items are returned. If it fails, the workflow loops back to re-translate.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Ingestion & Configuration
**Overview:** Starts the workflow from an incoming webhook call and centralizes configuration values used across translation, audio generation, and QA.

**Nodes involved:**
- Webhook Trigger
- Workflow Configuration

#### Node: Webhook Trigger
- **Type / role:** `Webhook` (n8n-nodes-base.webhook) — workflow entry point.
- **Key configuration:**
  - **HTTP Method:** POST
  - **Path:** `chinese-to-speech`
  - **Response mode:** `lastNode` (response will be produced by the last executed node in the path, here intended to be **Return Audio Files**).
- **Expected input:** JSON body with `text` (or `body.text` depending on caller).
- **Outputs / connections:** → **Workflow Configuration**
- **Version notes:** typeVersion 2.1
- **Edge cases / failures:**
  - Caller sends non-JSON body or missing `text`.
  - Response expectations: since `responseMode=lastNode`, any branch that does not reach Respond node can cause timeouts for the caller.

#### Node: Workflow Configuration
- **Type / role:** `Set` — defines reusable configuration parameters.
- **Key configuration choices (interpreted):**
  - `targetLanguages` (array): `["English","Spanish","French","German","Japanese"]`
  - ElevenLabs:
    - `elevenLabsVoiceId`: placeholder (`<__PLACEHOLDER_VALUE__ElevenLabs Voice ID__>`) must be replaced
    - `elevenLabsModelId`: `eleven_multilingual_v2`
    - voice settings: `stability=0.5`, `similarityBoost=0.75`, `style=0`, `speakerBoost=true`
  - QA / control:
    - `qualityThreshold=7` (intended passing threshold)
    - `maxRetries=2` (intended loop limit, but **not enforced** in current logic)
  - **Include other fields:** enabled (preserves incoming webhook payload alongside these settings).
- **Outputs / connections:** → **Validate Chinese Text**
- **Version notes:** typeVersion 3.4
- **Edge cases / failures:**
  - Placeholder `elevenLabsVoiceId` left unchanged will cause ElevenLabs API errors (404/400).
  - `qualityThreshold` and `maxRetries` are defined here but **qualityThreshold is never actually applied** and **maxRetries is unused** (see Block 5).

---

### Block 2 — Input Validation (Chinese detection)
**Overview:** Extracts the provided text and validates it is non-empty and contains Chinese characters before proceeding.

**Nodes involved:**
- Validate Chinese Text

#### Node: Validate Chinese Text
- **Type / role:** `Code` — custom validation and normalization.
- **Key logic:**
  - Reads text from: `body.text` OR `text`:
    - `const inputText = $input.first().json.body?.text || $input.first().json.text || '';`
  - Rejects empty/whitespace-only.
  - Checks presence of Chinese characters using a Unicode-range regex.
- **Output:**
  - On success: `{ isValid: true, text, characterCount }`
  - On failure: `{ isValid: false, error, text }`
- **Outputs / connections:** → **Translation Agent** (unconditional)
- **Version notes:** typeVersion 2
- **Edge cases / failures:**
  - **Important:** Even when `isValid=false`, the workflow still proceeds to Translation Agent because there is no IF gate here. This can cause:
    - translation prompt receiving empty text,
    - inconsistent downstream behavior.
  - Regex is broad but may still miss some punctuation-only Chinese input or mixed-language inputs depending on caller intent.

**Recommended fix:** Add an `IF` node after this code node that routes invalid inputs to an immediate `Respond to Webhook` with an error.

---

### Block 3 — Multilingual Translation (GPT-4o + structured JSON)
**Overview:** Uses GPT-4o to translate Chinese text into multiple configured languages, enforcing a strict JSON structure via a structured output parser.

**Nodes involved:**
- Translation Agent
- OpenAI Chat Model
- Structured Output Parser

#### Node: Translation Agent
- **Type / role:** LangChain `Agent` — orchestrates prompt + model + output parsing.
- **Key configuration choices:**
  - **Input text:** `{{ $json.text }}`
  - **System message:** translation instructions plus dynamic list of target languages:
    - `{{ $('Workflow Configuration').first().json.targetLanguages.join(', ') }}`
  - **Has output parser:** enabled (expects parser-provided JSON schema).
- **Connections:**
  - **AI language model input:** from **OpenAI Chat Model**
  - **AI output parser input:** from **Structured Output Parser**
  - **Main output:** → **Split Translations**
- **Version notes:** typeVersion 3.1
- **Edge cases / failures:**
  - If upstream text is empty/invalid (Block 2 issue), translation may be garbage or the parser may fail.
  - If the model returns malformed JSON, parser will throw.

#### Node: OpenAI Chat Model
- **Type / role:** LangChain `lmChatOpenAi` — provides GPT-4o chat model to the agent.
- **Key configuration:**
  - **Model:** `gpt-4o`
  - Credentials: “OpenAi account”
- **Connections:** Provides `ai_languageModel` → **Translation Agent**
- **Version notes:** typeVersion 1.3
- **Edge cases / failures:**
  - Invalid/expired API key, quota exceeded, rate limits.
  - Latency/timeouts for long texts.

#### Node: Structured Output Parser
- **Type / role:** LangChain structured parser — enforces translation output schema.
- **Schema (interpreted):**
  - Root object with `translations: [{ language: string, text: string }, ...]`
- **Connections:** Provides `ai_outputParser` → **Translation Agent**
- **Version notes:** typeVersion 1.3
- **Edge cases / failures:**
  - Model output not matching schema (missing fields, wrong types) → parsing failure.
  - Agent may need stronger instructions if it tends to include extra keys.

---

### Block 4 — Translation Fan-out + Audio Synthesis (ElevenLabs)
**Overview:** Converts the translations array into individual items (one per language), then calls ElevenLabs TTS to produce MP3 audio for each.

**Nodes involved:**
- Split Translations
- Generate Speech with ElevenLabs
- Format Audio Response

#### Node: Split Translations
- **Type / role:** `Code` — fan-out items per translation.
- **Key logic:**
  - Reads: `$input.first().json.output?.translations || []`
    - Assumes the Agent output is stored under `output.translations`.
  - Throws if none: `throw new Error('No translations found in the AI output');`
  - Creates items: `{ language, text, originalText }`
- **Connections:** → **Generate Speech with ElevenLabs**
- **Version notes:** typeVersion 2
- **Edge cases / failures:**
  - If the agent output structure differs (e.g., not under `output`), this will fail.
  - If translations contain non-string text, HTTP body building later may break.

#### Node: Generate Speech with ElevenLabs
- **Type / role:** `HTTP Request` — calls ElevenLabs TTS API.
- **Key configuration:**
  - **URL:** `https://api.elevenlabs.io/v1/text-to-speech/{{ voiceId }}`
    - `voiceId` from `Workflow Configuration.elevenLabsVoiceId`
  - **Method:** POST
  - **Authentication:** predefined credential type `elevenLabsApi`
  - **Headers:** `Content-Type: application/json`
  - **Body (JSON, expression-built):**
    - `text`: translation text
    - `model_id`: from config (`eleven_multilingual_v2`)
    - `voice_settings`: stability, similarity_boost, style, use_speaker_boost
  - **Response handling:** `responseFormat=file`, stored in binary property **audio**
- **Connections:** → **Format Audio Response**
- **Version notes:** typeVersion 4.3
- **Edge cases / failures:**
  - Invalid voice ID, invalid API key, subscription/credit limitations.
  - Text too long for the chosen model/endpoint.
  - The JSON body uses expressions; if `$json.text` contains characters that aren’t properly treated as a JSON string by n8n expression rendering, request can fail. (Safer pattern is to construct body via fields rather than raw JSON template.)
  - Endpoint may return non-audio error JSON; node expects file, which can yield confusing binary output behavior.

#### Node: Format Audio Response
- **Type / role:** `Set` — enriches items with metadata while keeping binary audio.
- **Key configuration:**
  - `stripBinary: false` (keeps ElevenLabs binary `audio`)
  - Adds:
    - `language`: from item
    - `fileName`: `{{ $json.language.toLowerCase().replace(/ /g, '_') }}_audio.mp3`
    - `mimeType`: `audio/mpeg`
    - `originalText`
- **Connections:** → **Quality Review Agent**
- **Version notes:** typeVersion 3.4
- **Edge cases / failures:**
  - `language` missing leads to expression errors for `toLowerCase()`.
  - Filename collisions if languages repeat.

---

### Block 5 — Quality Review & Output Control (GPT-4o QA gate)
**Overview:** Uses GPT-4o to score translation quality and then either returns results or loops back to re-translate.

**Nodes involved:**
- Quality Review Agent
- OpenAI Chat Model1
- Structured Output Parser1
- Check Quality Score
- Return Audio Files

#### Node: Quality Review Agent
- **Type / role:** LangChain `Agent` — evaluates translation quality.
- **Input text composed as:**
  - `Original Chinese: {{ $json.originalText }}`
  - `Translated {{ $json.language }}: {{ $json.text }}`
- **System message:** QA rubric; instructs returning JSON that matches the parser schema.
- **Connections:**
  - **AI language model input:** from **OpenAI Chat Model1**
  - **AI output parser input:** from **Structured Output Parser1**
  - **Main output:** → **Check Quality Score**
- **Version notes:** typeVersion 3.1
- **Edge cases / failures:**
  - If `originalText` missing (earlier failures), evaluation becomes meaningless.
  - Parser mismatch → errors.

#### Node: OpenAI Chat Model1
- **Type / role:** GPT-4o model provider for QA.
- **Key configuration:**
  - Model: `gpt-4o`
  - Credentials: same “OpenAi account”
- **Connections:** `ai_languageModel` → **Quality Review Agent**
- **Version notes:** typeVersion 1.3
- **Edge cases / failures:** same as the translation model (rate limits, quota, latency).

#### Node: Structured Output Parser1
- **Type / role:** structured parser for QA output.
- **Schema (interpreted):**
  - `{ qualityScore: number, feedback: string, passesQuality: boolean }`
- **Connections:** `ai_outputParser` → **Quality Review Agent**
- **Version notes:** typeVersion 1.3
- **Edge cases / failures:**
  - Agent must compute `passesQuality`, but no instruction binds it to `qualityThreshold`. It may set it inconsistently.

#### Node: Check Quality Score
- **Type / role:** `IF` — gates output based on QA.
- **Condition:** `{{ $json.output.passesQuality }}` is `true`
  - Note: it checks `output.passesQuality` (agent output is assumed under `output`).
- **True branch:** → **Return Audio Files**
- **False branch:** → **Translation Agent** (re-translation loop)
- **Version notes:** typeVersion 2.3
- **Edge cases / failures / design gaps:**
  - **qualityThreshold is not used**: the decision relies entirely on AI-generated boolean `passesQuality`.
  - **maxRetries is not used**: the loop can repeat indefinitely if AI keeps failing (or if `passesQuality` is always false).
  - Looping back to **Translation Agent** from an item that currently represents a single language may not match the translation agent’s expected input shape (it expects `$json.text` to be the original Chinese). After the loop, `$json.text` may refer to the translated text, not the original, depending on item fields at that time.

**Recommended fixes:**
1. Compute `passesQuality` deterministically in n8n: `qualityScore >= qualityThreshold`.
2. Implement retry counter (e.g., `retryCount`) and stop after `maxRetries`.
3. Ensure loop reuses the original Chinese text and reprocesses translations cleanly.

#### Node: Return Audio Files
- **Type / role:** `Respond to Webhook` — returns results to the caller.
- **Key configuration:** `respondWith=allIncomingItems` (returns all items, typically each with its binary `audio` and metadata).
- **Connections:** terminal node
- **Version notes:** typeVersion 1.5
- **Edge cases / failures:**
  - Returning multiple binary items via webhook depends on client expectations; some clients may not handle multipart/batched binary n8n responses well.
  - If workflow loops and never hits this node, webhook call will time out.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook Trigger | Webhook | Entry point (POST webhook) | — | Workflow Configuration | ## Input Ingestion & Configuration<br>**What:** Webhook receives Chinese text input with configuration parameters<br>**Why:** Initiates automated processing without manual intervention from connected applications |
| Workflow Configuration | Set | Central configuration (languages, voice, thresholds) | Webhook Trigger | Validate Chinese Text | ## Input Ingestion & Configuration<br>**What:** Webhook receives Chinese text input with configuration parameters<br>**Why:** Initiates automated processing without manual intervention from connected applications |
| Validate Chinese Text | Code | Validates non-empty and contains Chinese chars | Workflow Configuration | Translation Agent | ## Input Ingestion & Configuration<br>**What:** Webhook receives Chinese text input with configuration parameters<br>**Why:** Initiates automated processing without manual intervention from connected applications |
| Translation Agent | LangChain Agent | Multilingual translation into target languages | Validate Chinese Text; (loop) Check Quality Score (false) | Split Translations | ## Context-Aware Multilingual Translation<br>**What:** Translation agent converts Chinese to target languages with context awareness<br>**Why:** Produces culturally appropriate translations while preserving original meaning and nuance |
| OpenAI Chat Model | OpenAI Chat Model (LangChain) | GPT-4o for translation | — | Translation Agent | ## Context-Aware Multilingual Translation<br>**What:** Translation agent converts Chinese to target languages with context awareness<br>**Why:** Produces culturally appropriate translations while preserving original meaning and nuance |
| Structured Output Parser | Structured Output Parser (LangChain) | Enforces translation JSON schema | — | Translation Agent | ## Context-Aware Multilingual Translation<br>**What:** Translation agent converts Chinese to target languages with context awareness<br>**Why:** Produces culturally appropriate translations while preserving original meaning and nuance |
| Split Translations | Code | Fan-out translations array to per-language items | Translation Agent | Generate Speech with ElevenLabs |  |
| Generate Speech with ElevenLabs | HTTP Request | ElevenLabs TTS → audio binary | Split Translations | Format Audio Response | ## Neural Text-to-Speech Synthesis<br>**What:** ElevenLabs generates natural speech from translated text<br>**Why:** Creates professional voiceovers with accurate pronunciation and natural intonation |
| Format Audio Response | Set | Adds metadata, preserves binary audio | Generate Speech with ElevenLabs | Quality Review Agent | ## Neural Text-to-Speech Synthesis<br>**What:** ElevenLabs generates natural speech from translated text<br>**Why:** Creates professional voiceovers with accurate pronunciation and natural intonation |
| Quality Review Agent | LangChain Agent | QA scoring for translation quality | Format Audio Response | Check Quality Score | ## Quality Scoring & Output Control<br>**What:** Quality agent scores translations on linguistic accuracy and audio fidelity<br>**Why:** Maintains output standards and filters unsuitable content before delivery |
| OpenAI Chat Model1 | OpenAI Chat Model (LangChain) | GPT-4o for QA | — | Quality Review Agent | ## Quality Scoring & Output Control<br>**What:** Quality agent scores translations on linguistic accuracy and audio fidelity<br>**Why:** Maintains output standards and filters unsuitable content before delivery |
| Structured Output Parser1 | Structured Output Parser (LangChain) | Enforces QA JSON schema | — | Quality Review Agent | ## Quality Scoring & Output Control<br>**What:** Quality agent scores translations on linguistic accuracy and audio fidelity<br>**Why:** Maintains output standards and filters unsuitable content before delivery |
| Check Quality Score | IF | Route pass → return, fail → re-translate | Quality Review Agent | Return Audio Files (true); Translation Agent (false) | ## Quality Scoring & Output Control<br>**What:** Quality agent scores translations on linguistic accuracy and audio fidelity<br>**Why:** Maintains output standards and filters unsuitable content before delivery |
| Return Audio Files | Respond to Webhook | Sends all items (incl. audio) to caller | Check Quality Score (true) | — |  |
| Sticky Note | Sticky Note | Context / prerequisites | — | — | ## Prerequisites<br>Active accounts: OpenAI API access, ElevenLabs subscription.<br>## Use Cases<br>Chinese language learning apps, international marketing content localization<br>## Customization<br>Add additional target languages, modify voice characteristics and speaking rates<br>## Benefits<br>Automates 95% of translation workflow, delivers publication-ready audio in minutes |
| Sticky Note1 | Sticky Note | Setup checklist | — | — | ## Setup Steps<br>1. Obtain OpenAI API key and configure in "Translation Agent"<br>2. Set up ElevenLabs account, generate API key<br>3. Configure webhook URL and update in source applications to trigger workflow<br>4. Customize target languages and voice settings in translation and ElevenLabs nodes<br>5. Adjust quality thresholds in "Check Quality Score"<br>6. Update output webhook endpoint in "Return Audio Files" node |
| Sticky Note2 | Sticky Note | How it works narrative | — | — | ## How It Works<br>This workflow provides automated Chinese text translation with high-quality audio synthesis for language learning platforms, content creators, and international communication teams. It addresses the challenge of converting Chinese text into accurate multilingual translations with natural-sounding voiceovers. The system receives Chinese text via webhook, validates input formatting, and processes it through an AI translation agent that generates multiple language versions. Each translation is converted to speech using ElevenLabs' neural voice models, then formatted into professional audio responses. A quality review agent evaluates translation accuracy, cultural appropriateness, and audio clarity against predefined criteria. High-scoring outputs are returned via webhook for immediate use, while low-quality results trigger review processes, ensuring consistent delivery of publication-ready multilingual audio content. |
| Sticky Note3 | Sticky Note | ElevenLabs explanation | — | — | ## Neural Text-to-Speech Synthesis<br>**What:** ElevenLabs generates natural speech from translated text<br>**Why:** Creates professional voiceovers with accurate pronunciation and natural intonation |
| Sticky Note4 | Sticky Note | Translation explanation | — | — | ## Context-Aware Multilingual Translation<br>**What:** Translation agent converts Chinese to target languages with context awareness<br>**Why:** Produces culturally appropriate translations while preserving original meaning and nuance |
| Sticky Note5 | Sticky Note | Input block explanation | — | — | ## Input Ingestion & Configuration<br>**What:** Webhook receives Chinese text input with configuration parameters<br>**Why:** Initiates automated processing without manual intervention from connected applications |
| Sticky Note6 | Sticky Note | QA block explanation | — | — | ## Quality Scoring & Output Control<br>**What:** Quality agent scores translations on linguistic accuracy and audio fidelity<br>**Why:** Maintains output standards and filters unsuitable content before delivery |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
2. **Add “Webhook” node** (name: *Webhook Trigger*):
   - Method: **POST**
   - Path: `chinese-to-speech`
   - Response mode: **Last node**
3. **Add “Set” node** (name: *Workflow Configuration*) and connect from Webhook:
   - Add fields:
     - `targetLanguages` (Array): `English, Spanish, French, German, Japanese`
     - `elevenLabsVoiceId` (String): your ElevenLabs voice ID
     - `elevenLabsModelId` (String): `eleven_multilingual_v2`
     - `stability` (Number): `0.5`
     - `similarityBoost` (Number): `0.75`
     - `style` (Number): `0`
     - `speakerBoost` (Boolean): `true`
     - `qualityThreshold` (Number): `7`
     - `maxRetries` (Number): `2`
   - Enable “Include Other Fields” so webhook payload remains available.
4. **Add “Code” node** (name: *Validate Chinese Text*) and connect from Workflow Configuration:
   - Paste logic that:
     - extracts `body.text` or `text`,
     - validates non-empty,
     - validates Chinese characters via regex,
     - outputs `{isValid, text, characterCount}`.
   - (Optional but recommended) Add an **IF** node after this to stop on invalid input and respond with an error.
5. **Add LangChain “OpenAI Chat Model” node** (name: *OpenAI Chat Model*):
   - Model: `gpt-4o`
   - Configure **OpenAI credentials** (API key) in n8n Credentials.
6. **Add LangChain “Structured Output Parser”** (name: *Structured Output Parser*):
   - Schema: object with `translations` array of `{language, text}`.
7. **Add LangChain “Agent”** (name: *Translation Agent*) and connect from Validate Chinese Text:
   - Set Text: `{{ $json.text }}`
   - System message: include translation instructions and dynamically list languages using:
     - `{{ $('Workflow Configuration').first().json.targetLanguages.join(', ') }}`
   - Enable “Use Output Parser”.
   - Connect **OpenAI Chat Model** to the agent’s **AI Language Model** input.
   - Connect **Structured Output Parser** to the agent’s **AI Output Parser** input.
8. **Add “Code” node** (name: *Split Translations*) and connect from Translation Agent:
   - Read translations from `{{$input.first().json.output.translations}}`
   - Create one item per translation with `language`, `text`, `originalText`.
9. **Add “HTTP Request” node** (name: *Generate Speech with ElevenLabs*) and connect from Split Translations:
   - Method: **POST**
   - URL: `https://api.elevenlabs.io/v1/text-to-speech/{{ $('Workflow Configuration').first().json.elevenLabsVoiceId }}`
   - Authentication: **ElevenLabs API** credential (create in n8n)
   - Headers: `Content-Type: application/json`
   - Body: JSON containing `text`, `model_id`, and `voice_settings` from Workflow Configuration.
   - Response: set to **File** and store in binary property `audio`.
10. **Add “Set” node** (name: *Format Audio Response*) and connect from ElevenLabs:
    - Keep binary data (do not strip).
    - Add:
      - `language` = `{{ $json.language }}`
      - `fileName` = `{{ $json.language.toLowerCase().replace(/ /g, '_') }}_audio.mp3`
      - `mimeType` = `audio/mpeg`
      - `originalText` = `{{ $json.originalText }}`
11. **Add LangChain “OpenAI Chat Model”** (name: *OpenAI Chat Model1*) for QA:
    - Model: `gpt-4o`
    - Same OpenAI credentials (or another).
12. **Add LangChain “Structured Output Parser”** (name: *Structured Output Parser1*) for QA:
    - Schema: `{ qualityScore:number, feedback:string, passesQuality:boolean }`
13. **Add LangChain “Agent”** (name: *Quality Review Agent*) and connect from Format Audio Response:
    - Text: `Original Chinese: {{ $json.originalText }}\nTranslated {{ $json.language }}: {{ $json.text }}`
    - System message: QA rubric and instruction to match the parser schema.
    - Connect **OpenAI Chat Model1** to AI Language Model input.
    - Connect **Structured Output Parser1** to AI Output Parser input.
14. **Add “IF” node** (name: *Check Quality Score*) and connect from Quality Review Agent:
    - Condition: `{{ $json.output.passesQuality }}` is true
    - True output: to Respond node
    - False output: back to **Translation Agent** (as in the original)
    - (Recommended) add retry counting and threshold-based pass computation here.
15. **Add “Respond to Webhook” node** (name: *Return Audio Files*) and connect from Check Quality Score (true):
    - Respond with: **All Incoming Items**

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Active accounts required: OpenAI API access, ElevenLabs subscription. | Prerequisites sticky note |
| Suggested use cases: Chinese language learning apps, international marketing content localization. | Prerequisites sticky note |
| Customization: add target languages, modify voice characteristics and speaking rates. | Prerequisites sticky note |
| Setup steps: configure OpenAI key, ElevenLabs key, webhook URL, adjust quality thresholds, and webhook response settings. | Setup Steps sticky note |
| Design note: `qualityThreshold` and `maxRetries` are configured but not enforced by the current logic; consider implementing deterministic thresholding and retry limits to avoid infinite loops. | Analysis derived from workflow structure |
