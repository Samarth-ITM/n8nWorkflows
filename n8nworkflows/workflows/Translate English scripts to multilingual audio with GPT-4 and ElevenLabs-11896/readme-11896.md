Translate English scripts to multilingual audio with GPT-4 and ElevenLabs

https://n8nworkflows.xyz/workflows/translate-english-scripts-to-multilingual-audio-with-gpt-4-and-elevenlabs-11896


# Translate English scripts to multilingual audio with GPT-4 and ElevenLabs

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow name:** Translate English scripts to multi-language audio using AI and ElevenLabs  
**Purpose:** Accept an English script via webhook, translate it into multiple languages using GPT‑4 (via the n8n LangChain Agent), then generate audio for each translation using ElevenLabs (HTTP Request). The workflow then returns an audio-related response and uploads output to Google Drive.  
**Typical use cases:** multilingual narration generation, localization pipelines, automated voiceover generation for content teams.

### 1.1 Input Reception
Receives an HTTP request and extracts/normalizes parameters needed for translation and audio generation.

### 1.2 AI Translation (GPT‑4 via LangChain Agent)
Uses an OpenAI chat model plus structured output parsing to produce a predictable translations payload (e.g., an array of language/text items).

### 1.3 Translation Fan-out / Iteration
Splits the translation list into individual items and loops over each translation.

### 1.4 Audio Synthesis (ElevenLabs)
For each translation item, calls ElevenLabs Text-to-Speech and produces audio (likely binary).

### 1.5 Response + Storage
Responds to the webhook and uploads content to Google Drive. The current connections suggest a loop that may iterate through upload + loop again.

### 1.6 Error Handling / Alerting
On workflow errors, a Slack message is sent.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception
**Overview:** Starts the workflow when a webhook is called, then prepares request fields for the AI translation step.  
**Nodes involved:** `Webhook Trigger`, `Extract Request Parameters`

#### Node: Webhook Trigger
- **Type / role:** `n8n-nodes-base.webhook` — Entry point via HTTP.
- **Configuration (interpreted):** Parameters are not specified in the JSON (blank), but it will at minimum define an HTTP method/path in the n8n UI. `typeVersion: 2.1`.
- **Key data:** Output will include `body`, `query`, `headers` depending on configuration.
- **Connections:**  
  - **Output →** `Extract Request Parameters` (main)
- **Failure modes / edge cases:**
  - Misconfigured method/path or missing authentication → unexpected public endpoint behavior.
  - Large script payloads may exceed n8n payload limits or reverse proxy limits.
  - If expecting JSON but receiving form-data/text, downstream expressions may fail.

#### Node: Extract Request Parameters
- **Type / role:** `n8n-nodes-base.set` — Normalize/rename incoming parameters.
- **Configuration (interpreted):** Empty in JSON; in practice you would map:
  - `script` (English text)
  - `targetLanguages` (list)
  - optional voice settings (voiceId, model, stability, similarity, etc.)
- **Connections:**  
  - **Input ←** `Webhook Trigger`  
  - **Output →** `Translate with GPT-4`
- **Failure modes / edge cases:**
  - Missing fields causing undefined expressions later (e.g., `{{$json.script}}`).
  - Target languages not being an array (string vs array) breaks list splitting.

---

### Block 2 — AI Translation (GPT‑4 via LangChain)
**Overview:** Uses a LangChain Agent configured with OpenAI chat model, memory, and a structured output parser to return translations in a predictable schema.  
**Nodes involved:** `Translate with GPT-4`, `OpenAI Chat Model`, `Structured Output Parser`, `Simple Memory`

#### Node: Translate with GPT-4
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — Orchestrates prompting, tool/model usage, and structured outputs.
- **Configuration (interpreted):** Empty in JSON; typically includes:
  - System/user instructions (translate script into N languages)
  - Use of `Structured Output Parser` to force JSON structure
  - Possibly referencing input fields from `Extract Request Parameters`
- **Connections:**
  - **Input ←** `Extract Request Parameters` (main)
  - **AI language model ←** `OpenAI Chat Model` (ai_languageModel)
  - **AI output parser ←** `Structured Output Parser` (ai_outputParser)
  - **AI memory ←** `Simple Memory` (ai_memory)
  - **Output →** `Split Translations`
- **Version requirements:** `typeVersion: 3` (LangChain agent node behavior can change between versions; keep n8n and node package aligned).
- **Failure modes / edge cases:**
  - OpenAI auth/quota errors; model name mismatch.
  - Output not matching the structured schema → parser failure.
  - Long scripts can hit token limits; translations truncated.

#### Node: OpenAI Chat Model
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — Provides GPT‑4-class chat completion capability.
- **Configuration (interpreted):** Empty in JSON; normally includes:
  - Credentials: OpenAI API key (or OpenAI-compatible provider)
  - Model: e.g., `gpt-4o`, `gpt-4.1`, etc.
  - Temperature/top_p, max tokens
- **Connections:**
  - **Output →** `Translate with GPT-4` (ai_languageModel)
- **Version requirements:** `typeVersion: 1.3`
- **Failure modes / edge cases:** credential invalid, rate limits, request too large, model deprecation.

#### Node: Structured Output Parser
- **Type / role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — Enforces a schema for the agent’s output.
- **Configuration (interpreted):** Empty in JSON; usually defines a JSON schema like:
  - `translations: [{ languageCode, languageName, text }]`
- **Connections:**
  - **Output →** `Translate with GPT-4` (ai_outputParser)
- **Version requirements:** `typeVersion: 1.3`
- **Failure modes / edge cases:** model returns non-JSON or schema mismatch; escaping issues; partial output.

#### Node: Simple Memory
- **Type / role:** `@n8n/n8n-nodes-langchain.memoryBufferWindow` — Keeps short window of conversation context.
- **Configuration (interpreted):** Empty in JSON; typically window size (k) set in UI.
- **Connections:**
  - **Output →** `Translate with GPT-4` (ai_memory)
- **Version requirements:** `typeVersion: 1.3`
- **Failure modes / edge cases:** memory not usually fatal; but could cause unexpected context bleed if reused incorrectly.

---

### Block 3 — Split & Loop Over Translations
**Overview:** Converts the structured translations array into individual items and iterates them to generate audio per language.  
**Nodes involved:** `Split Translations`, `Loop Over Translations`

#### Node: Split Translations
- **Type / role:** `n8n-nodes-base.itemLists` — Transforms lists/arrays into separate items.
- **Configuration (interpreted):** Empty in JSON; typically “Split Out Items” on a field like `translations`.
- **Connections:**
  - **Input ←** `Translate with GPT-4`
  - **Output →** `Loop Over Translations`
- **Version requirements:** `typeVersion: 3`
- **Failure modes / edge cases:**
  - Wrong field path (e.g., translations nested under `data.translations`) produces 0 items.
  - If translations is already itemized, you may accidentally double-split.

#### Node: Loop Over Translations
- **Type / role:** `n8n-nodes-base.splitInBatches` — Batch/loop controller.
- **Configuration (interpreted):** Empty in JSON; normally batch size = 1 to process one translation at a time.
- **Connections:**
  - **Input ←** `Split Translations`
  - **Output (main, index 1) →** `Generate Audio with ElevenLabs`  
  - **Input from loop-back ←** `Upload to Google Drive` (see connections)
- **Important connection detail:** The node has two output branches in n8n:  
  - One branch for “current batch” items (goes to ElevenLabs)  
  - Another for “no items left” (empty in this workflow)
- **Failure modes / edge cases:**
  - If batch size > 1, ElevenLabs node must handle multiple items (may increase cost and time).
  - Loop-back wiring here is unusual (see Block 5); can cause repeated processing if not configured carefully.

---

### Block 4 — Audio Synthesis (ElevenLabs)
**Overview:** Calls ElevenLabs Text-to-Speech to generate audio per translation item.  
**Nodes involved:** `Generate Audio with ElevenLabs`

#### Node: Generate Audio with ElevenLabs
- **Type / role:** `n8n-nodes-base.httpRequest` — Direct API call to ElevenLabs.
- **Configuration (interpreted):** Empty in JSON; typically:
  - Method: `POST`
  - URL: `https://api.elevenlabs.io/v1/text-to-speech/{voice_id}` (or newer endpoints)
  - Headers: `xi-api-key: <ElevenLabs API key>`, `Content-Type: application/json`
  - Body: `{ text: {{$json.text}}, model_id, voice_settings... }`
  - Response: audio binary (e.g., `audio/mpeg`) and “Download” enabled in n8n
- **Connections:**
  - **Input ←** `Loop Over Translations`
  - **Output →** `Return Audio Response`
- **Version requirements:** `typeVersion: 4.2`
- **Failure modes / edge cases:**
  - Wrong endpoint/voice ID → 404.
  - Missing/invalid API key → 401/403.
  - Large text → request rejected or long latency/timeouts.
  - Binary handling: if “Response Format” not set to file/binary, you may lose audio.

---

### Block 5 — Response + Google Drive Storage
**Overview:** Responds to the webhook and uploads content to Google Drive. The current wiring suggests an iterative pattern, but the order is atypical (respond first, then upload, then loop).  
**Nodes involved:** `Return Audio Response`, `Upload to Google Drive`

#### Node: Return Audio Response
- **Type / role:** `n8n-nodes-base.respondToWebhook` — Sends HTTP response back to the caller.
- **Configuration (interpreted):** Empty in JSON; typically:
  - Respond with binary audio OR JSON containing links/metadata per language.
- **Connections:**
  - **Input ←** `Generate Audio with ElevenLabs`
  - **Output →** `Upload to Google Drive`
- **Edge cases / failure modes:**
  - If responding with binary, ensure correct `Content-Type` and binary property mapping.
  - Responding inside a loop can be problematic: a webhook call can typically only get one response. If this node runs multiple times (once per translation), only the first response may succeed; subsequent runs can error or be ignored depending on n8n mode.

#### Node: Upload to Google Drive
- **Type / role:** `n8n-nodes-base.googleDrive` — Uploads generated audio to Drive.
- **Configuration (interpreted):** Empty in JSON; typically:
  - Operation: Upload
  - Binary Property: e.g., `data` or `audio`
  - File Name: from language code/title
  - Parent Folder ID: target folder
  - Credentials: Google OAuth2
- **Connections:**
  - **Input ←** `Return Audio Response`
  - **Output →** `Loop Over Translations` (loop-back)
- **Important behavior note:** This creates a cycle: `Loop → ElevenLabs → Respond → Drive → Loop`. This can be valid if `SplitInBatches` is used to control iteration, but the webhook response placement is risky (see above).
- **Failure modes / edge cases:**
  - OAuth scope/consent issues; expired refresh token.
  - Uploading binary property mismatch → empty/invalid files.
  - Rate limits or large files causing upload timeouts.

---

### Block 6 — Error Handling / Slack Alert
**Overview:** If the workflow errors, it triggers a Slack message.  
**Nodes involved:** `Error Handler Trigger`, `Send a message`

#### Node: Error Handler Trigger
- **Type / role:** `n8n-nodes-base.errorTrigger` — Runs when another workflow execution fails.
- **Configuration (interpreted):** Empty in JSON; in n8n this node is used in a dedicated error workflow or paired error handling.
- **Connections:**
  - **Output →** `Send a message`
- **Failure modes / edge cases:**
  - If this workflow is not configured as the global error workflow, it may not run as expected.
  - Missing error context mapping can produce generic Slack messages.

#### Node: Send a message
- **Type / role:** `n8n-nodes-base.slack` — Sends notification to Slack.
- **Configuration (interpreted):** Empty in JSON; typically:
  - Auth: Slack OAuth2 or webhook
  - Channel and message text (including execution URL, error message)
- **Connections:**
  - **Input ←** `Error Handler Trigger`
- **Version requirements:** `typeVersion: 2.3`
- **Failure modes / edge cases:** invalid Slack credentials, missing channel permissions, message formatting errors.

---

### Sticky Notes
All sticky notes exist but have **empty content**:
- `Sticky Note`, `Sticky Note1`, `Sticky Note2`, `Sticky Note3`, `Sticky Note4`, `Sticky Note7`

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook Trigger | n8n-nodes-base.webhook | Receives incoming request | — | Extract Request Parameters |  |
| Extract Request Parameters | n8n-nodes-base.set | Maps/normalizes request fields | Webhook Trigger | Translate with GPT-4 |  |
| Translate with GPT-4 | @n8n/n8n-nodes-langchain.agent | Produces structured translations | Extract Request Parameters; (AI: OpenAI Chat Model, Structured Output Parser, Simple Memory) | Split Translations |  |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM provider for agent | — | Translate with GPT-4 (ai_languageModel) |  |
| Structured Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces schema for translations | — | Translate with GPT-4 (ai_outputParser) |  |
| Simple Memory | @n8n/n8n-nodes-langchain.memoryBufferWindow | Short-term context memory | — | Translate with GPT-4 (ai_memory) |  |
| Split Translations | n8n-nodes-base.itemLists | Splits translations array into items | Translate with GPT-4 | Loop Over Translations |  |
| Loop Over Translations | n8n-nodes-base.splitInBatches | Iterates over translations | Split Translations; (loop-back from Upload to Google Drive) | Generate Audio with ElevenLabs |  |
| Generate Audio with ElevenLabs | n8n-nodes-base.httpRequest | TTS API call producing audio | Loop Over Translations | Return Audio Response |  |
| Return Audio Response | n8n-nodes-base.respondToWebhook | Responds to the webhook caller | Generate Audio with ElevenLabs | Upload to Google Drive |  |
| Upload to Google Drive | n8n-nodes-base.googleDrive | Stores audio files in Drive | Return Audio Response | Loop Over Translations |  |
| Error Handler Trigger | n8n-nodes-base.errorTrigger | Runs on execution error | — | Send a message |  |
| Send a message | n8n-nodes-base.slack | Sends Slack alert | Error Handler Trigger | — |  |
| Sticky Note | n8n-nodes-base.stickyNote | Comment container | — | — |  |
| Sticky Note1 | n8n-nodes-base.stickyNote | Comment container | — | — |  |
| Sticky Note2 | n8n-nodes-base.stickyNote | Comment container | — | — |  |
| Sticky Note3 | n8n-nodes-base.stickyNote | Comment container | — | — |  |
| Sticky Note4 | n8n-nodes-base.stickyNote | Comment container | — | — |  |
| Sticky Note7 | n8n-nodes-base.stickyNote | Comment container | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create `Webhook Trigger` (Webhook)**
   - Choose **HTTP Method** (commonly `POST`) and a **Path** (e.g., `/translate-to-audio`).
   - Enable response mode compatible with `Respond to Webhook` usage (in n8n this is typically “Using ‘Respond to Webhook’ node” behavior).

2. **Create `Extract Request Parameters` (Set)**
   - Map incoming fields from the webhook, for example:
     - `script` = `{{$json.body.script}}`
     - `languages` = `{{$json.body.languages}}` (array like `["fr","de","es"]`)
     - Optional: `voiceId`, `modelId`, `audioFormat`, etc.
   - Ensure `languages` is an array (convert if needed).

3. **Create AI nodes (LangChain)**
   1) **`OpenAI Chat Model`**
      - Add OpenAI credentials (API key).
      - Select a model (e.g., GPT‑4 class).
      - Set temperature as needed.
   2) **`Structured Output Parser`**
      - Define a schema such as:
        - `translations`: array of objects with `languageCode` and `text`
   3) **`Simple Memory`**
      - Configure memory window (optional; often small like 3–10).

4. **Create `Translate with GPT-4` (LangChain Agent)**
   - Connect:
     - `OpenAI Chat Model` → Agent as **AI Language Model**
     - `Structured Output Parser` → Agent as **AI Output Parser**
     - `Simple Memory` → Agent as **AI Memory**
   - In the Agent prompt/instructions (in node UI), instruct:
     - Translate `script` into each language from `languages`.
     - Return strictly the structured format expected by the parser.

5. **Create `Split Translations` (Item Lists)**
   - Configure to split out items from the `translations` field produced by the agent.
   - Output should be **one item per language**, each containing at least `languageCode` and `text`.

6. **Create `Loop Over Translations` (Split in Batches)**
   - Set **Batch Size = 1** for predictable per-language processing.
   - Connect `Split Translations` → `Loop Over Translations`.

7. **Create `Generate Audio with ElevenLabs` (HTTP Request)**
   - Add ElevenLabs API key in headers (either via credentials or static header).
   - Typical configuration:
     - Method: `POST`
     - URL: `https://api.elevenlabs.io/v1/text-to-speech/{{$json.voiceId || 'YOUR_DEFAULT_VOICE_ID'}}`
     - JSON body includes `text: {{$json.text}}` and optional voice settings.
   - Set response to return **binary audio** (download enabled) and store in a known binary property (e.g., `audio`).

8. **Create `Respond to Webhook` node (`Return Audio Response`)**
   - Decide response strategy:
     - **Option A (recommended):** respond once with a JSON summary (e.g., Drive links) after loop completes.
     - **Option B:** respond with first audio only (not ideal for multi-language).
   - Note: The provided workflow places this node inside the loop chain; adjust if you need a single response.

9. **Create `Upload to Google Drive` (Google Drive)**
   - Configure Google OAuth2 credentials.
   - Operation: Upload
   - Binary Property: the one produced by ElevenLabs (e.g., `audio`)
   - File name expression example: `{{$json.languageCode}}.mp3`
   - Folder: select target folder ID.

10. **Wire nodes in the same order as the workflow**
   - `Webhook Trigger` → `Extract Request Parameters` → `Translate with GPT-4` → `Split Translations` → `Loop Over Translations` → `Generate Audio with ElevenLabs` → `Return Audio Response` → `Upload to Google Drive` → back to `Loop Over Translations`
   - If you want a safer architecture:
     - Upload inside the loop, but **respond after the loop finishes** (requires using the “no items left” output of SplitInBatches or collecting results).

11. **Create error workflow path**
   - Add `Error Handler Trigger` (Error Trigger).
   - Add `Slack` node (`Send a message`) with Slack credentials and a message template containing:
     - error message, workflow name, execution URL.
   - Connect `Error Handler Trigger` → `Send a message`.
   - Ensure n8n is configured so this workflow is used for error handling if desired.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Sticky notes are present but contain no text. | No additional embedded documentation was provided inside the workflow. |
| The current graph responds to the webhook before Google Drive upload and loops afterward; this can be problematic for multi-item loops because a webhook typically expects a single response. | Consider restructuring to respond once after all uploads complete. |