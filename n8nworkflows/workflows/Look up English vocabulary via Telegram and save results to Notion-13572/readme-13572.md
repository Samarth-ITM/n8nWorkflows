Look up English vocabulary via Telegram and save results to Notion

https://n8nworkflows.xyz/workflows/look-up-english-vocabulary-via-telegram-and-save-results-to-notion-13572


# Look up English vocabulary via Telegram and save results to Notion

## 1. Workflow Overview

**Purpose:**  
This workflow runs a personal Telegram bot that helps you build English vocabulary. You can send **text**, **voice**, or **photo** messages to the bot; it extracts an English word (or words), looks it up with an AI “dictionary” agent, replies in Telegram with a formatted definition card, and saves the result into a **Notion database**.

**Primary use cases:**
- Quickly look up unfamiliar English words encountered while reading or browsing
- Hands-free word lookup via voice messages (Whisper transcription)
- Extract words from images/screenshots (Vision OCR) and learn them
- Create a searchable vocabulary list in Notion

### 1.1 Entry & Security
Receives Telegram updates, loads configuration values, and restricts usage to a single authorized chat ID.

### 1.2 Input Detection & Pre-processing
Routes incoming messages by type (text/voice/photo) and normalizes all paths to a single field: `chat_input`.

### 1.3 AI Lookup & Output
Uses a structured-output AI agent to spell-check and define the word, returns a Telegram reply, then stores the entry in Notion.

---

## 2. Block-by-Block Analysis

### Block 1 — Entry & Security

**Overview:**  
This block triggers on Telegram messages, sets user-editable config values, and checks whether the sender is authorized. Unauthorized users receive a rejection message and do not proceed.

**Nodes involved:**
- **Telegram Trigger**
- **Config — Edit Me**
- **Authorize User**
- **Reject Unauthorized User**

#### Node: Telegram Trigger
- **Type / role:** `telegramTrigger` — Entry point; receives Telegram updates via webhook.
- **Key config:**
  - Updates: `message`
  - Webhook ID: `tgvoc-webhook-0001`
- **Credentials:** Telegram Bot API credential: **Telegram (Blackwaltz)**
- **Outputs to:** **Config — Edit Me**
- **Edge cases / failures:**
  - Telegram credential misconfigured → webhook registration fails / no triggers
  - Bot not started by user (user never pressed “Start”) → messages may not arrive as expected
  - Telegram outages / webhook connectivity issues

#### Node: Config — Edit Me
- **Type / role:** `set` — Central configuration values (meant to be edited before activation).
- **Config values created:**
  - `TELEGRAM_CHAT_ID` (string): authorized user chat ID
  - `TELEGRAM_BOT_TOKEN` (string): bot token (used for direct Telegram file download endpoints)
  - `NOTION_VOCABULARY_DB_ID` (string): target Notion database ID
  - `TARGET_LANGUAGE` (string): translation language for AI output (e.g., “Traditional Chinese”)
- **Notes (node note):** “Edit all 4 values here before activating the workflow”
- **Outputs to:** **Authorize User**
- **Edge cases / failures:**
  - Wrong/missing token → later HTTP file calls to Telegram fail (401/404)
  - Wrong database ID → Notion create-page fails
  - `TARGET_LANGUAGE` ambiguous/unsupported → AI may produce inconsistent translations

#### Node: Authorize User
- **Type / role:** `if` — Security gate.
- **Condition logic:**  
  Compares:
  - Left: `String($('Telegram Trigger').item.json.message.chat.id)`
  - Right: `$('Config — Edit Me').item.json.TELEGRAM_CHAT_ID`
- **Outputs:**
  - **True** → **Detect Input Type**
  - **False** → **Reject Unauthorized User**
- **Edge cases / failures:**
  - Messages from channels/groups may have different chat/user semantics
  - If `message.chat.id` is missing (rare update types), condition may error or evaluate unexpectedly

#### Node: Reject Unauthorized User
- **Type / role:** `telegram` — Sends rejection message to unauthorized sender.
- **Message text:** “Sorry, you are not authorized to use this bot.”
- **Chat ID:** `$('Telegram Trigger').item.json.message.chat.id`
- **Outputs:** none (end branch)
- **Edge cases / failures:**
  - If the bot cannot message the user (blocked bot, privacy settings) → send fails

---

### Block 2 — Input Detection & Processing

**Overview:**  
Determines whether the message is text, voice, or photo. Voice messages are downloaded and transcribed; photos are downloaded and analyzed for English text; text passes through directly. All branches output a unified `chat_input`.

**Nodes involved:**
- **Detect Input Type**
- **Set Text Input**
- **Get Voice File**
- **Download Audio**
- **Transcribe Audio (Whisper)**
- **Set Voice Text**
- **Get Photo File**
- **Download Image**
- **Fix Image MIME Type**
- **Analyze Image**
- **Set Photo Text**

#### Node: Detect Input Type
- **Type / role:** `switch` — Routes messages to one of 3 typed flows, with fallback.
- **Rules (renamed outputs):**
  - `text`: message text not empty  
    `$('Telegram Trigger').item.json.message.text`
  - `voice`: voice `file_id` not empty  
    `$('Telegram Trigger').item.json.message.voice?.file_id`
  - `photo`: photo file_id not empty  
    `$('Telegram Trigger').item.json.message.photo?.[0]?.file_id`
- **Fallback output:** `none` (not connected)
- **Outputs to:**
  - `text` → **Set Text Input**
  - `voice` → **Get Voice File**
  - `photo` → **Get Photo File**
- **Edge cases / failures:**
  - Captions on photos are ignored (only OCR path used)
  - Unsupported message types (documents, stickers, etc.) go to `none` and effectively drop execution

#### Node: Set Text Input
- **Type / role:** `set` — Normalizes text input.
- **Sets:**
  - `chat_input` = `$('Telegram Trigger').item.json.message.text`
  - `type` = `"text"`
- **Outputs to:** **Dictionary Agent**
- **Edge cases:**
  - Multi-word phrases will be passed as-is; downstream agent expects “English word provided” but may still handle phrases.

#### Node: Get Voice File
- **Type / role:** `httpRequest` — Calls Telegram Bot API `getFile` to obtain file path for the voice message.
- **URL expression:**
  - `https://api.telegram.org/bot{{ TELEGRAM_BOT_TOKEN }}/getFile?file_id={{ voice.file_id }}`
- **Depends on:** `TELEGRAM_BOT_TOKEN` from **Config — Edit Me**
- **Outputs to:** **Download Audio**
- **Edge cases / failures:**
  - Invalid token → 401
  - Expired/invalid file_id → Telegram returns ok=false / missing `result.file_path`

#### Node: Download Audio
- **Type / role:** `httpRequest` — Downloads the actual audio file from Telegram’s file endpoint.
- **URL expression:**
  - `https://api.telegram.org/file/bot{{ TELEGRAM_BOT_TOKEN }}/{{ $json.result.file_path }}`
- **Response format:** file (binary)
- **Outputs to:** **Transcribe Audio (Whisper)**
- **Edge cases / failures:**
  - Large files / slow network → timeout risk
  - Missing `result.file_path` from previous step → expression failure

#### Node: Transcribe Audio (Whisper)
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi` (OpenAI Audio Transcription) — Converts audio to text.
- **Operation:** Resource `audio` → `transcribe`
- **Notes:** “Transcribes voice messages to text”
- **Credentials:** OpenAI API credential: **OpenAi (Jason)**
- **Input:** binary audio from **Download Audio**
- **Output:** JSON containing `text`
- **Outputs to:** **Set Voice Text**
- **Edge cases / failures:**
  - Unsupported audio encoding/container → transcription fails
  - OpenAI rate limiting / quota exceeded
  - If binary property name differs, node may not find the audio (depends on n8n binary field conventions)

#### Node: Set Voice Text
- **Type / role:** `set` — Normalizes transcription output.
- **Sets:**
  - `chat_input` = `$json.text`
  - `type` = `"voice"`
- **Outputs to:** **Dictionary Agent**
- **Edge cases:**
  - Whisper may transcribe extra words; agent may need to infer intended target word

#### Node: Get Photo File
- **Type / role:** `httpRequest` — Calls Telegram `getFile` for the *highest resolution* photo.
- **URL expression:**
  - Uses last photo size: `message.photo.at(-1).file_id`
  - `https://api.telegram.org/bot{{ TELEGRAM_BOT_TOKEN }}/getFile?file_id={{ ... }}`
- **Outputs to:** **Download Image**
- **Edge cases / failures:**
  - If `photo` array is empty/unavailable → expression failure
  - Invalid token/file_id → Telegram API error

#### Node: Download Image
- **Type / role:** `httpRequest` — Downloads the image file from Telegram.
- **URL expression:**
  - `https://api.telegram.org/file/bot{{ TELEGRAM_BOT_TOKEN }}/{{ $json.result.file_path }}`
- **Response format:** file (binary)
- **Outputs to:** **Fix Image MIME Type**
- **Edge cases / failures:**
  - Missing `file_path` → expression failure
  - Large images may increase memory/time

#### Node: Fix Image MIME Type
- **Type / role:** `code` — Forces binary metadata so the next node treats it as a JPEG.
- **Code behavior:**
  - For each item with `item.binary.data`, sets:
    - `mimeType = 'image/jpeg'`
    - `fileExtension = 'jpg'`
- **Outputs to:** **Analyze Image**
- **Why it exists:** Some Telegram images may not carry correct MIME metadata in n8n; vision nodes can require it.
- **Edge cases / failures:**
  - If binary property isn’t named `data`, nothing is changed (and downstream may fail)
  - If image is actually PNG/WebP, forcing JPEG metadata might be misleading (often still works, but not guaranteed)

#### Node: Analyze Image
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi` (OpenAI Vision) — Extracts English text from image.
- **Operation:** Resource `image` → `analyze`
- **Model:** `gpt-4o-mini`
- **Input type:** `base64` (node will read from binary and encode as needed)
- **Prompt text:**  
  “Extract any English words or text from this image. Output only the words, no explanations.”
- **Output:** `content` (the extracted words/text)
- **Outputs to:** **Set Photo Text**
- **Edge cases / failures:**
  - OCR quality issues (blur, stylized fonts) → partial/incorrect extraction
  - Image too large → request may fail or be slow
  - OpenAI rate limits/quota

#### Node: Set Photo Text
- **Type / role:** `set` — Normalizes OCR output.
- **Sets:**
  - `chat_input` = `$json.content`
  - `type` = `"photo"`
- **Outputs to:** **Dictionary Agent**
- **Edge cases:**
  - If OCR returns multiple words/lines, downstream dictionary agent may treat it as a phrase/list

---

### Block 3 — AI Lookup & Output (Telegram + Notion)

**Overview:**  
A structured-output AI agent spell-checks the provided input and returns dictionary fields (word, definition, translation, etc.) in a predictable JSON structure. The workflow then replies in Telegram and saves the entry to Notion.

**Nodes involved:**
- **Dictionary Agent**
- **OpenAI (Dictionary)**
- **Dictionary Output Parser**
- **Send Telegram Reply**
- **Save Vocabulary to Notion**

#### Node: Dictionary Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — Orchestrates the dictionary task with a system message and structured output.
- **Input text:** `{{$json.chat_input}}`
- **Prompt type:** `define`
- **System message (key logic):**
  - Spell-check first; if incorrect, infer intended word and look up corrected word
  - Otherwise look up directly
  - `translation` and `example_translation` must be in `TARGET_LANGUAGE` from config
  - Output *structured only*, no commentary; use `N/A` for missing fields
- **Connected AI components:**
  - Language model input: from **OpenAI (Dictionary)** via `ai_languageModel`
  - Output parser: from **Dictionary Output Parser** via `ai_outputParser`
- **Main output:** to **Send Telegram Reply**
- **Edge cases / failures:**
  - If input contains multiple words, the agent may return one combined result or choose one word unpredictably
  - If the model returns non-conformant JSON, parsing may fail (mitigated by output parser, but still possible)
  - `TARGET_LANGUAGE` injection/format issues if set to unusual strings

#### Node: OpenAI (Dictionary)
- **Type / role:** `lmChatOpenAi` — Provides the chat model used by the agent.
- **Model:** `gpt-4.1-mini`
- **Credentials:** OpenAI API credential: **OpenAi (Jason)**
- **Connection:** outputs to **Dictionary Agent** on `ai_languageModel`
- **Edge cases / failures:**
  - Model availability changes
  - Rate limits/quota

#### Node: Dictionary Output Parser
- **Type / role:** `outputParserStructured` — Enforces structured JSON output using a schema example.
- **Schema example fields:**
  - `word`, `definition`, `translation`, `part_of_speech`, `example_sentence`, `example_translation`
- **Connection:** outputs to **Dictionary Agent** on `ai_outputParser`
- **Edge cases / failures:**
  - If the model returns invalid JSON or misses required keys, parsing can error and stop the workflow

#### Node: Send Telegram Reply
- **Type / role:** `telegram` — Sends a formatted dictionary card back to the user.
- **Chat ID:** `$('Telegram Trigger').item.json.message.chat.id`
- **Message body (uses agent output):**
  - `{{ $json.output.word }}`, `part_of_speech`, `definition`, `translation`, example sentence + translation
- **Outputs to:** **Save Vocabulary to Notion**
- **Edge cases / failures:**
  - Telegram message length limits (very long definitions/examples could fail)
  - Formatting: includes a leading “📖”; otherwise plain text

#### Node: Save Vocabulary to Notion
- **Type / role:** `notion` — Creates a new page in a Notion database.
- **Resource/operation:** `databasePage` create
- **Database ID:** from config: `NOTION_VOCABULARY_DB_ID`
- **Title:** `$('Dictionary Agent').item.json.output.word`
- **Properties set:**
  - A rich text property named **Description** is filled with a multi-line block containing:
    - Definition, Translation, Part of speech, Example, Example translation, Word
- **Credentials:** Notion API credential: **Notion (Jason)**
- **Execution note:** `alwaysOutputData: false` (workflow doesn’t force output if Notion returns nothing)
- **Edge cases / failures:**
  - Notion database missing required Title property or property name mismatch (“Description” must exist and be rich_text)
  - Notion integration not shared with the database → 403
  - Rate limits or intermittent Notion API errors

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Telegram Trigger | telegramTrigger | Entry point: receives Telegram messages | — | Config — Edit Me | ## Entry & Security\n\nReceives incoming Telegram messages, loads config values, and checks the sender's chat ID. Only the authorised user continues; others receive a rejection notice. |
| Config — Edit Me | set | Stores editable config (chat id, token, Notion DB, language) | Telegram Trigger | Authorize User | ## Entry & Security\n\nReceives incoming Telegram messages, loads config values, and checks the sender's chat ID. Only the authorised user continues; others receive a rejection notice. |
| Authorize User | if | Allows only configured chat ID | Config — Edit Me | Detect Input Type (true), Reject Unauthorized User (false) | ## Entry & Security\n\nReceives incoming Telegram messages, loads config values, and checks the sender's chat ID. Only the authorised user continues; others receive a rejection notice. |
| Reject Unauthorized User | telegram | Sends rejection reply | Authorize User (false) | — | ## Entry & Security\n\nReceives incoming Telegram messages, loads config values, and checks the sender's chat ID. Only the authorised user continues; others receive a rejection notice. |
| Detect Input Type | switch | Routes to text/voice/photo pipelines | Authorize User (true) | Set Text Input, Get Voice File, Get Photo File | ## Input Detection & Processing\n\nRoutes each message by type: text passes through directly, voice is transcribed via Whisper, and photos are processed via GPT-4o-mini Vision OCR. All paths produce a unified `chat_input` value for the AI agent. |
| Set Text Input | set | Normalizes text to `chat_input` | Detect Input Type (text) | Dictionary Agent | ## Input Detection & Processing\n\nRoutes each message by type: text passes through directly, voice is transcribed via Whisper, and photos are processed via GPT-4o-mini Vision OCR. All paths produce a unified `chat_input` value for the AI agent. |
| Get Voice File | httpRequest | Calls Telegram `getFile` for voice | Detect Input Type (voice) | Download Audio | ## Input Detection & Processing\n\nRoutes each message by type: text passes through directly, voice is transcribed via Whisper, and photos are processed via GPT-4o-mini Vision OCR. All paths produce a unified `chat_input` value for the AI agent. |
| Download Audio | httpRequest | Downloads voice file as binary | Get Voice File | Transcribe Audio (Whisper) | ## Input Detection & Processing\n\nRoutes each message by type: text passes through directly, voice is transcribed via Whisper, and photos are processed via GPT-4o-mini Vision OCR. All paths produce a unified `chat_input` value for the AI agent. |
| Transcribe Audio (Whisper) | @n8n/n8n-nodes-langchain.openAi | Whisper transcription | Download Audio | Set Voice Text | ## Input Detection & Processing\n\nRoutes each message by type: text passes through directly, voice is transcribed via Whisper, and photos are processed via GPT-4o-mini Vision OCR. All paths produce a unified `chat_input` value for the AI agent. |
| Set Voice Text | set | Normalizes transcription to `chat_input` | Transcribe Audio (Whisper) | Dictionary Agent | ## Input Detection & Processing\n\nRoutes each message by type: text passes through directly, voice is transcribed via Whisper, and photos are processed via GPT-4o-mini Vision OCR. All paths produce a unified `chat_input` value for the AI agent. |
| Get Photo File | httpRequest | Calls Telegram `getFile` for best photo size | Detect Input Type (photo) | Download Image | ## Input Detection & Processing\n\nRoutes each message by type: text passes through directly, voice is transcribed via Whisper, and photos are processed via GPT-4o-mini Vision OCR. All paths produce a unified `chat_input` value for the AI agent. |
| Download Image | httpRequest | Downloads photo as binary | Get Photo File | Fix Image MIME Type | ## Input Detection & Processing\n\nRoutes each message by type: text passes through directly, voice is transcribed via Whisper, and photos are processed via GPT-4o-mini Vision OCR. All paths produce a unified `chat_input` value for the AI agent. |
| Fix Image MIME Type | code | Forces binary MIME metadata to image/jpeg | Download Image | Analyze Image | ## Input Detection & Processing\n\nRoutes each message by type: text passes through directly, voice is transcribed via Whisper, and photos are processed via GPT-4o-mini Vision OCR. All paths produce a unified `chat_input` value for the AI agent. |
| Analyze Image | @n8n/n8n-nodes-langchain.openAi | Vision OCR-style extraction of English words | Fix Image MIME Type | Set Photo Text | ## Input Detection & Processing\n\nRoutes each message by type: text passes through directly, voice is transcribed via Whisper, and photos are processed via GPT-4o-mini Vision OCR. All paths produce a unified `chat_input` value for the AI agent. |
| Set Photo Text | set | Normalizes OCR output to `chat_input` | Analyze Image | Dictionary Agent | ## Input Detection & Processing\n\nRoutes each message by type: text passes through directly, voice is transcribed via Whisper, and photos are processed via GPT-4o-mini Vision OCR. All paths produce a unified `chat_input` value for the AI agent. |
| Dictionary Agent | @n8n/n8n-nodes-langchain.agent | Spell-check + dictionary lookup with structured output | Set Text Input / Set Voice Text / Set Photo Text (+ AI model/parser inputs) | Send Telegram Reply | ## AI Lookup & Output\n\nA GPT-4.1-mini agent spell-checks the input, then returns structured vocabulary data. The result is sent back as a Telegram reply and saved to the Notion vocabulary database. |
| OpenAI (Dictionary) | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM provider for Dictionary Agent | — | Dictionary Agent (ai_languageModel) | ## AI Lookup & Output\n\nA GPT-4.1-mini agent spell-checks the input, then returns structured vocabulary data. The result is sent back as a Telegram reply and saved to the Notion vocabulary database. |
| Dictionary Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces JSON structure for agent output | — | Dictionary Agent (ai_outputParser) | ## AI Lookup & Output\n\nA GPT-4.1-mini agent spell-checks the input, then returns structured vocabulary data. The result is sent back as a Telegram reply and saved to the Notion vocabulary database. |
| Send Telegram Reply | telegram | Returns formatted vocab card | Dictionary Agent | Save Vocabulary to Notion | ## AI Lookup & Output\n\nA GPT-4.1-mini agent spell-checks the input, then returns structured vocabulary data. The result is sent back as a Telegram reply and saved to the Notion vocabulary database. |
| Save Vocabulary to Notion | notion | Persists vocab entry to Notion DB | Send Telegram Reply | — | ## AI Lookup & Output\n\nA GPT-4.1-mini agent spell-checks the input, then returns structured vocabulary data. The result is sent back as a Telegram reply and saved to the Notion vocabulary database. |
| Sticky Note - Main | stickyNote | Documentation panel | — | — | ## How it works\n\nThis workflow powers a personal Telegram bot for building English vocabulary. When you send a message — text, voice, or photo — the bot detects the input type and processes it: voice messages are transcribed via Whisper, photos are scanned via GPT-4o-mini Vision OCR to extract words. The extracted text is passed to a GPT-4.1-mini agent that checks spelling, then returns the word, definition, translation, part of speech, and an example sentence. The translation language is controlled by `TARGET_LANGUAGE` in the Config node — set it to any language you need (e.g. Traditional Chinese, Japanese, Spanish). The reply is sent back via Telegram and the entry is saved to Notion for later review.\n\n## Setup steps\n\n1. Open **Config — Edit Me** and set: `TELEGRAM_CHAT_ID`, `TELEGRAM_BOT_TOKEN`, `NOTION_VOCABULARY_DB_ID`, and `TARGET_LANGUAGE` (e.g. `Traditional Chinese`, `Japanese`, `Spanish`).\n2. Create a bot via [@BotFather](https://t.me/botfather) and copy the token.\n3. Get your chat ID by messaging [@userinfobot](https://t.me/userinfobot).\n4. Add credentials: **Telegram Bot API**, **OpenAI API**, **Notion API**.\n5. Activate the workflow — the webhook is registered automatically.\n6. Send any English word to your bot to test, for example: `phenomenon`. |
| Sticky Note - Group 1 | stickyNote | Comment: Entry & security | — | — | ## Entry & Security\n\nReceives incoming Telegram messages, loads config values, and checks the sender's chat ID. Only the authorised user continues; others receive a rejection notice. |
| Sticky Note - Group 2 | stickyNote | Comment: Input routing | — | — | ## Input Detection & Processing\n\nRoutes each message by type: text passes through directly, voice is transcribed via Whisper, and photos are processed via GPT-4o-mini Vision OCR. All paths produce a unified `chat_input` value for the AI agent. |
| Sticky Note - Group 3 | stickyNote | Comment: AI + outputs | — | — | ## AI Lookup & Output\n\nA GPT-4.1-mini agent spell-checks the input, then returns structured vocabulary data. The result is sent back as a Telegram reply and saved to the Notion vocabulary database. |
| Sticky Note - CTA | stickyNote | Promotional link | — | — | ## Want more automation templates?\n\nCheck out the **Content Automation Bundle** — 6 n8n workflows for AI content creation, social media publishing, and Threads analytics.\n\n👉 https://jasonchuang0818.gumroad.com/l/n8n-content-automation-bundle |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named: *Look up English vocabulary via Telegram and save to Notion*.

2. **Add Telegram Trigger**
   - Node type: **Telegram Trigger**
   - Updates: **message**
   - Create/select **Telegram Bot API** credentials for your bot.
   - This node becomes the entry point.

3. **Add Set node: “Config — Edit Me”**
   - Node type: **Set**
   - Add 4 string fields:
     - `TELEGRAM_CHAT_ID` = your personal chat ID (from https://t.me/userinfobot)
     - `TELEGRAM_BOT_TOKEN` = token from https://t.me/botfather
     - `NOTION_VOCABULARY_DB_ID` = the Notion database ID
     - `TARGET_LANGUAGE` = e.g., `Traditional Chinese`
   - Connect: **Telegram Trigger → Config — Edit Me**

4. **Add IF node: “Authorize User”**
   - Node type: **IF**
   - Condition (String equals):
     - Left: `{{ String($('Telegram Trigger').item.json.message.chat.id) }}`
     - Right: `{{ $('Config — Edit Me').item.json.TELEGRAM_CHAT_ID }}`
   - Connect: **Config — Edit Me → Authorize User**

5. **Add Telegram node: “Reject Unauthorized User”**
   - Node type: **Telegram**
   - Chat ID: `{{ $('Telegram Trigger').item.json.message.chat.id }}`
   - Text: `Sorry, you are not authorized to use this bot.`
   - Connect: **Authorize User (false) → Reject Unauthorized User**

6. **Add Switch node: “Detect Input Type”**
   - Node type: **Switch**
   - Create 3 rules (with renamed outputs):
     1) Output key `text` if `{{ $('Telegram Trigger').item.json.message.text }}` is not empty  
     2) Output key `voice` if `{{ $('Telegram Trigger').item.json.message.voice?.file_id }}` is not empty  
     3) Output key `photo` if `{{ $('Telegram Trigger').item.json.message.photo?.[0]?.file_id }}` is not empty  
   - Fallback output: `none` (can remain unconnected)
   - Connect: **Authorize User (true) → Detect Input Type**

7. **Text path**
   1) Add Set node: **Set Text Input**
      - `chat_input` = `{{ $('Telegram Trigger').item.json.message.text }}`
      - `type` = `text`
   2) Connect: **Detect Input Type (text) → Set Text Input**

8. **Voice path**
   1) Add HTTP Request node: **Get Voice File**
      - Method: GET
      - URL: `https://api.telegram.org/bot{{ $('Config — Edit Me').item.json.TELEGRAM_BOT_TOKEN }}/getFile?file_id={{ $('Telegram Trigger').item.json.message.voice.file_id }}`
   2) Add HTTP Request node: **Download Audio**
      - Method: GET
      - URL: `https://api.telegram.org/file/bot{{ $('Config — Edit Me').item.json.TELEGRAM_BOT_TOKEN }}/{{ $json.result.file_path }}`
      - Response: **File** (binary)
   3) Add OpenAI node: **Transcribe Audio (Whisper)**
      - Node type: **OpenAI (LangChain)**
      - Resource: **audio**
      - Operation: **transcribe**
      - Credentials: OpenAI API key
   4) Add Set node: **Set Voice Text**
      - `chat_input` = `{{ $json.text }}`
      - `type` = `voice`
   5) Connect:  
      **Detect Input Type (voice) → Get Voice File → Download Audio → Transcribe Audio (Whisper) → Set Voice Text**

9. **Photo path**
   1) Add HTTP Request node: **Get Photo File**
      - URL: `https://api.telegram.org/bot{{ $('Config — Edit Me').item.json.TELEGRAM_BOT_TOKEN }}/getFile?file_id={{ $('Telegram Trigger').item.json.message.photo.at(-1).file_id }}`
   2) Add HTTP Request node: **Download Image**
      - URL: `https://api.telegram.org/file/bot{{ $('Config — Edit Me').item.json.TELEGRAM_BOT_TOKEN }}/{{ $json.result.file_path }}`
      - Response: **File**
   3) Add Code node: **Fix Image MIME Type**
      - Paste logic that sets `item.binary.data.mimeType = 'image/jpeg'` and `fileExtension = 'jpg'`
   4) Add OpenAI node: **Analyze Image**
      - Resource: **image**
      - Operation: **analyze**
      - Model: **gpt-4o-mini**
      - Input type: **base64**
      - Prompt: “Extract any English words or text from this image. Output only the words, no explanations.”
   5) Add Set node: **Set Photo Text**
      - `chat_input` = `{{ $json.content }}`
      - `type` = `photo`
   6) Connect:  
      **Detect Input Type (photo) → Get Photo File → Download Image → Fix Image MIME Type → Analyze Image → Set Photo Text**

10. **Add AI components for dictionary lookup**
    1) Add OpenAI Chat Model node: **OpenAI (Dictionary)**
       - Node type: **OpenAI Chat Model (LangChain)**
       - Model: **gpt-4.1-mini**
       - Credentials: OpenAI API key
    2) Add Structured Output Parser node: **Dictionary Output Parser**
       - Provide a JSON schema example with keys:  
         `word`, `definition`, `translation`, `part_of_speech`, `example_sentence`, `example_translation`
    3) Add Agent node: **Dictionary Agent**
       - Prompt type: **define**
       - Text: `{{ $json.chat_input }}`
       - System message: include:
         - spell-check / correction rule
         - translation language: `{{ $('Config — Edit Me').item.json.TARGET_LANGUAGE }}`
         - structured output only, use `N/A` when missing
       - Connect the model and parser:
         - **OpenAI (Dictionary)** → **Dictionary Agent** (AI Language Model connection)
         - **Dictionary Output Parser** → **Dictionary Agent** (AI Output Parser connection)

11. **Connect inputs into Dictionary Agent**
    - **Set Text Input → Dictionary Agent**
    - **Set Voice Text → Dictionary Agent**
    - **Set Photo Text → Dictionary Agent**

12. **Add Telegram node: “Send Telegram Reply”**
    - Chat ID: `{{ $('Telegram Trigger').item.json.message.chat.id }}`
    - Text uses agent output fields, e.g.:
      - `{{ $json.output.word }}`, `part_of_speech`, `definition`, `translation`, example sentence + translation
    - Connect: **Dictionary Agent → Send Telegram Reply**

13. **Add Notion node: “Save Vocabulary to Notion”**
    - Node type: **Notion**
    - Resource: **Database Page**
    - Operation: **Create**
    - Database ID: `{{ $('Config — Edit Me').item.json.NOTION_VOCABULARY_DB_ID }}`
    - Title: `{{ $('Dictionary Agent').item.json.output.word }}`
    - Properties:
      - Ensure your Notion database has a **rich text** property named `Description`
      - Fill `Description` with a multi-line summary using the agent output fields
    - Credentials: Notion API integration that is shared with the target database
    - Connect: **Send Telegram Reply → Save Vocabulary to Notion**

14. **Activate the workflow**
    - On activation, the Telegram Trigger webhook should register automatically.
    - Test by sending: `phenomenon` to your bot.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Create a Telegram bot and get token via BotFather | https://t.me/botfather |
| Find your Telegram chat ID via userinfobot | https://t.me/userinfobot |
| Bundle link mentioned in workflow notes | https://jasonchuang0818.gumroad.com/l/n8n-content-automation-bundle |
| Config guidance: set `TARGET_LANGUAGE` to any language you need (e.g., Traditional Chinese, Japanese, Spanish) | In-workflow note “Sticky Note - Main” |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.