Build a voice & text Telegram Bot with GPT-4.1-Mini and Gemini transcription

https://n8nworkflows.xyz/workflows/build-a-voice---text-telegram-bot-with-gpt-4-1-mini-and-gemini-transcription-12003


# Build a voice & text Telegram Bot with GPT-4.1-Mini and Gemini transcription

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This n8n workflow implements a Telegram bot that can handle **both text and voice messages**.  
- **Text messages** are checked with **Guardrails** for policy violations, then answered using **GPT-4.1-mini** with conversational memory.  
- **Voice messages** (Telegram OGG) are downloaded, **transcribed with Google Gemini**, answered with **GPT-4.1-mini**, then converted to **audio (TTS)** and returned to the user.

**Target use cases:**
- Customer support bots on Telegram (text + voice)
- Personal assistant bots with voice note transcription
- Policy-aware chatbots (basic “block and apologize” behavior for text)

### Logical Blocks
**1.1 Telegram Input Reception & Routing**
- Receives all incoming Telegram messages and routes to voice vs text handling.

**1.2 Voice Message Handling (Download → Transcribe → Answer → TTS → Send)**
- Downloads the voice note, transcribes via Gemini, answers via GPT, generates audio response, and sends it.

**1.3 Text Message Handling (Guardrails → Answer → Send)**
- Validates message against guardrails; if allowed, generates a GPT reply; otherwise sends a refusal message.

**1.4 Shared AI Infrastructure (Model + Memory)**
- A shared OpenAI chat model (gpt-4.1-mini) and shared conversation memory keyed by Telegram chat ID.

---

## 2. Block-by-Block Analysis

### 2.1 Telegram Input Reception & Routing

**Overview:**  
Triggers on incoming Telegram messages and decides whether the message is a voice note (OGG) or a text message.

**Nodes Involved:**
- Telegram Trigger
- If

#### Node: Telegram Trigger
- **Type / role:** `telegramTrigger` — webhook-based entry point receiving Telegram updates.
- **Configuration (interpreted):**
  - Listens to `message` updates only.
  - Uses Telegram bot credentials (`telegramApi`).
- **Inputs/Outputs:**
  - **Output →** `If`
- **Key data used later:**
  - `message.chat.id` (chat routing + memory key)
  - `message.text` (text flow)
  - `message.voice.file_id`, `message.voice.mime_type` (voice flow)
- **Failure/edge cases:**
  - Telegram credential misconfigured (bot token invalid) → trigger won’t receive updates.
  - Users send non-text/non-voice content (photos, stickers) → `message.text` or `message.voice` may be missing, affecting downstream expressions.

#### Node: If
- **Type / role:** `if` — branches between voice vs text logic.
- **Configuration (interpreted):**
  - Condition: `{{$json.message.voice.mime_type}} == "audio/ogg"`
  - **True branch:** Voice handling
  - **False branch:** Text handling
- **Inputs/Outputs:**
  - **Input ←** `Telegram Trigger`
  - **True output →** `Get a file`
  - **False output →** `Guardrails`
- **Failure/edge cases:**
  - If `message.voice` is missing, strict type validation can cause evaluation issues depending on n8n behavior/version. Practically, ensure Telegram messages that are not voice go to the false branch; if you see errors, add a “exists” check (e.g., `message.voice` exists) before reading `mime_type`.
  - Telegram sometimes uses different audio types (e.g., `audio/mpeg` for audio files). This workflow only treats `audio/ogg` as “voice note”.

---

### 2.2 Voice Message Handling (Download → Transcribe → Answer → TTS → Send)

**Overview:**  
Downloads the Telegram voice file, transcribes it with Gemini, answers using GPT-4.1-mini, generates an audio response (OpenAI audio), and sends audio back to the user.

**Nodes Involved:**
- Get a file
- Transcribe a recording
- AI Agent1
- Generate audio
- Send an audio file
- OpenAI Chat Model (shared)
- Simple Memory (shared)

#### Node: Get a file
- **Type / role:** `telegram` (resource: file) — fetches the file contents for a Telegram `file_id`.
- **Configuration (interpreted):**
  - Operation: get file by `fileId = {{ $('Telegram Trigger').item.json.message.voice.file_id }}`
  - Uses Telegram credentials.
- **Inputs/Outputs:**
  - **Input ←** `If` (true branch)
  - **Output →** `Transcribe a recording`
- **Failure/edge cases:**
  - Missing `message.voice.file_id` (non-voice message) → expression failure.
  - Telegram API errors / file not found (expired file_id, permissions).
  - Large files may increase download time.

#### Node: Transcribe a recording
- **Type / role:** `googleGemini` (resource: audio) — transcribes binary audio input to text.
- **Configuration (interpreted):**
  - Model: `models/gemini-2.5-flash`
  - Input type: `binary` (expects audio in binary from previous node)
- **Inputs/Outputs:**
  - **Input ←** `Get a file`
  - **Output →** `AI Agent1`
- **Key output used later:**
  - Transcription referenced as: `{{$json.content.parts[0].text}}`
- **Failure/edge cases:**
  - Google AI Studio / Gemini credential issues (invalid key, permissions).
  - If binary data is not in the expected property name or format, transcription may fail.
  - Very noisy/short audio → poor transcription quality; empty transcription possible.
  - Model availability/quota limits.

#### Node: AI Agent1
- **Type / role:** `langchain.agent` — generates a response based on the transcription.
- **Configuration (interpreted):**
  - Prompt type: **define**
  - **System message:**  
    “You are an ai agent for my telegram bot. you will be given a transcription of a voice message from telegram and you have to answer that user query.”
  - **User text fed to agent:**  
    `here's the transcription: {{ $json.content.parts[0].text }}`
- **Inputs/Outputs:**
  - **Main input ←** `Transcribe a recording`
  - **AI model input (ai_languageModel) ←** `OpenAI Chat Model`
  - **Memory input (ai_memory) ←** `Simple Memory`
  - **Main output →** `Generate audio`
- **Failure/edge cases:**
  - If transcription path differs (Gemini output structure changes), `content.parts[0].text` may be undefined.
  - Model refusals or empty outputs.
  - Memory key relies on Telegram Trigger item; if node execution doesn’t have access to that item (multi-item scenarios), reference might break—here it works because it’s a single message flow.

#### Node: Generate audio
- **Type / role:** `openAi` (resource: audio) — generates an audio rendition of the AI Agent’s text output.
- **Configuration (interpreted):**
  - Input text: `{{$json.output}}` (the agent’s generated answer)
- **Inputs/Outputs:**
  - **Input ←** `AI Agent1`
  - **Output →** `Send an audio file` (binary)
- **Failure/edge cases:**
  - Requires OpenAI API access that supports audio generation for this node configuration.
  - Output binary property naming must match what the Telegram “sendAudio” expects (n8n typically handles it, but mismatches can happen).
  - Large audio responses might exceed Telegram limits or cause timeouts.

#### Node: Send an audio file
- **Type / role:** `telegram` (operation: sendAudio) — sends the generated audio back to the chat.
- **Configuration (interpreted):**
  - `chatId = {{ $('Telegram Trigger').item.json.message.chat.id }}`
  - `binaryData = true` (send audio from binary output of previous node)
- **Inputs/Outputs:**
  - **Input ←** `Generate audio`
- **Failure/edge cases:**
  - If no binary data is present, Telegram sendAudio fails.
  - Telegram file size limits may block sending.
  - Chat ID expression depends on Telegram Trigger item context.

**Sticky note covering this flow:**  
“## Handle Voice Message  
Tips: * You can edit the system prompt in the AI Agent node according to your needs”

---

### 2.3 Text Message Handling (Guardrails → Answer → Send)

**Overview:**  
Checks incoming text against guardrails; if allowed, generates a GPT response and replies; if not allowed, sends a policy-violation apology.

**Nodes Involved:**
- Guardrails
- AI Agent
- Send a text message
- Send a text message1
- OpenAI Chat Model1 (Guardrails model)
- OpenAI Chat Model (shared for answering)
- Simple Memory (shared)

#### Node: Guardrails
- **Type / role:** `langchain.guardrails` — screens text before sending it to the agent.
- **Configuration (interpreted):**
  - Input text: `{{ $json.message.text }}`
  - Guardrails: configured as `{}` (defaults). In practice, this means you likely rely on node defaults or an empty ruleset (behavior depends on node implementation/version).
  - Uses an LLM connection from `OpenAI Chat Model1`.
- **Inputs/Outputs:**
  - **Input ←** `If` (false branch)
  - **Output 0 (“allowed”) →** `AI Agent`
  - **Output 1 (“blocked”) →** `Send a text message1`
  - **AI model input (ai_languageModel) ←** `OpenAI Chat Model1`
- **Failure/edge cases:**
  - If `message.text` is missing (e.g., user sends sticker), guardrails input is undefined.
  - If guardrails are empty/default, it may allow everything or behave unexpectedly; consider explicitly defining rules/policies.
  - OpenAI credential/model errors affect guardrails decision.

#### Node: AI Agent
- **Type / role:** `langchain.agent` — generates the response for allowed text input.
- **Configuration (interpreted):**
  - Prompt type: **guardrails** (meaning it expects upstream guardrails context/format)
  - No explicit system prompt in the node parameters (defaults apply).
- **Inputs/Outputs:**
  - **Main input ←** `Guardrails` (allowed path)
  - **AI model input (ai_languageModel) ←** `OpenAI Chat Model`
  - **Memory input (ai_memory) ←** `Simple Memory`
  - **Main output →** `Send a text message`
- **Failure/edge cases:**
  - If Guardrails output schema changes, the agent may not receive user text correctly.
  - Without an explicit system prompt, behavior may be generic; add one for consistent bot persona.

#### Node: Send a text message
- **Type / role:** `telegram` — sends the agent response as a Telegram message.
- **Configuration (interpreted):**
  - Text: `{{ $json.output }}` (agent output)
  - Chat ID: `{{ $('Telegram Trigger').item.json.message.chat.id }}`
  - `appendAttribution = false`
- **Inputs/Outputs:**
  - **Input ←** `AI Agent`
- **Failure/edge cases:**
  - If `$json.output` is missing/empty, message may be blank.
  - Telegram API errors, chat unavailable, bot blocked by user.

#### Node: Send a text message1
- **Type / role:** `telegram` — sends a refusal message when text violates policies.
- **Configuration (interpreted):**
  - Text: `Sorry! your text violates our policies`
  - Chat ID: `{{ $('Telegram Trigger').item.json.message.chat.id }}`
  - `appendAttribution = false`
- **Inputs/Outputs:**
  - **Input ←** `Guardrails` (blocked path)
- **Failure/edge cases:**
  - Same Telegram delivery risks as above.

#### Node: OpenAI Chat Model1
- **Type / role:** `lmChatOpenAi` — provides the language model used by Guardrails.
- **Configuration (interpreted):**
  - Model: `gpt-4.1-mini`
- **Inputs/Outputs:**
  - **Output (ai_languageModel) →** `Guardrails`
- **Failure/edge cases:**
  - OpenAI auth/quota/model access issues.

**Sticky note covering this flow:**  
“## Handle Text Message  
Tips: * You can edit the system prompt in the AI Agent node according to your needs”

---

### 2.4 Shared AI Infrastructure (Model + Memory)

**Overview:**  
Centralizes the chat model and conversation memory so both voice and text interactions remain consistent per Telegram chat.

**Nodes Involved:**
- OpenAI Chat Model
- Simple Memory

#### Node: OpenAI Chat Model
- **Type / role:** `lmChatOpenAi` — shared LLM backend for both AI Agent (text) and AI Agent1 (voice).
- **Configuration (interpreted):**
  - Model: `gpt-4.1-mini`
  - Built-in tools: none configured
- **Inputs/Outputs:**
  - **Output (ai_languageModel) →** `AI Agent`, `AI Agent1`
- **Failure/edge cases:**
  - OpenAI credential errors, rate limits, quota exhaustion.

#### Node: Simple Memory
- **Type / role:** `memoryBufferWindow` — conversational memory store keyed by session.
- **Configuration (interpreted):**
  - `sessionIdType = customKey`
  - `sessionKey = {{ $('Telegram Trigger').item.json.message.chat.id }}`
  - (Window size not shown; defaults apply—typically a limited number of turns.)
- **Inputs/Outputs:**
  - **Output (ai_memory) →** `AI Agent`, `AI Agent1`
- **Failure/edge cases:**
  - If chat ID is missing, memory sessions may collide or fail.
  - Workflow execution concurrency: multiple simultaneous messages from same chat could cause confusing context depending on memory implementation.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Telegram Trigger | telegramTrigger | Entry point: receive Telegram messages | — | If | ## How to setup (steps 1–6) |
| If | if | Route voice vs text | Telegram Trigger | Get a file; Guardrails | ## How to setup (steps 1–6) |
| Get a file | telegram | Download voice file by file_id | If (true) | Transcribe a recording | ## Handle Voice Message / Tips: edit system prompt |
| Transcribe a recording | googleGemini | Transcribe binary audio to text | Get a file | AI Agent1 | ## Handle Voice Message / Tips: edit system prompt |
| AI Agent1 | langchain.agent | Answer based on transcription | Transcribe a recording (+ OpenAI Chat Model + Simple Memory) | Generate audio | ## Handle Voice Message / Tips: edit system prompt |
| Generate audio | openAi (audio) | TTS / audio generation from agent output | AI Agent1 | Send an audio file | ## Handle Voice Message / Tips: edit system prompt |
| Send an audio file | telegram | Send audio reply to Telegram | Generate audio | — | ## Handle Voice Message / Tips: edit system prompt |
| Guardrails | langchain.guardrails | Validate text message | If (false) (+ OpenAI Chat Model1) | AI Agent; Send a text message1 | ## Handle Text Message / Tips: edit system prompt |
| AI Agent | langchain.agent | Answer allowed text message | Guardrails (+ OpenAI Chat Model + Simple Memory) | Send a text message | ## Handle Text Message / Tips: edit system prompt |
| Send a text message | telegram | Send text reply to Telegram | AI Agent | — | ## Handle Text Message / Tips: edit system prompt |
| Send a text message1 | telegram | Send policy-violation notice | Guardrails (blocked) | — | ## Handle Text Message / Tips: edit system prompt |
| OpenAI Chat Model | lmChatOpenAi | Shared LLM for agents | — | AI Agent; AI Agent1 |  |
| OpenAI Chat Model1 | lmChatOpenAi | LLM for guardrails evaluation | — | Guardrails |  |
| Simple Memory | memoryBufferWindow | Per-chat conversational memory | — | AI Agent; AI Agent1 |  |
| Sticky Note | stickyNote | Workspace note (setup steps) | — | — | ## How to setup (contains links) |
| Sticky Note1 | stickyNote | Workspace note (voice handling) | — | — | ## Handle Voice Message |
| Sticky Note2 | stickyNote | Workspace note (text handling) | — | — | ## Handle Text Message |

---

## 4. Reproducing the Workflow from Scratch

1) **Create credentials**
   1. **Telegram Bot Token**: Create a bot with BotFather, copy the token. In n8n, create **Telegram API** credentials and paste the token.
   2. **Google Gemini (AI Studio)**: Get an API key from https://aistudio.google.com/ and create **Google PaLM/Gemini** credentials in n8n (node uses `googlePalmApi` credential type).
   3. **OpenAI**: Create an OpenAI API key from https://platform.openai.com/docs/overview and ensure billing/credits are available. Create **OpenAI API** credentials in n8n.

2) **Add trigger**
   1. Add **Telegram Trigger** node.
   2. Set **Updates** to `message`.
   3. Select your **Telegram API credentials**.
   4. Activate or test webhook as required by n8n.

3) **Add router**
   1. Add an **If** node connected from **Telegram Trigger**.
   2. Condition: String equals  
      - Left: `{{$json.message.voice.mime_type}}`  
      - Right: `audio/ogg`
   3. Keep two outputs: **true** (voice) and **false** (text).

4) **Voice flow: download voice**
   1. Add **Telegram** node named “Get a file”.
   2. Resource: **file** (get file).
   3. File ID: `{{ $('Telegram Trigger').item.json.message.voice.file_id }}`
   4. Connect **If (true)** → **Get a file**.

5) **Voice flow: transcribe**
   1. Add **Google Gemini** node named “Transcribe a recording”.
   2. Resource: **audio**
   3. Input type: **binary**
   4. Model: `models/gemini-2.5-flash`
   5. Connect **Get a file** → **Transcribe a recording**.
   6. Select **Gemini credentials**.

6) **Shared model and memory**
   1. Add **OpenAI Chat Model** node (`lmChatOpenAi`), select model `gpt-4.1-mini`, set OpenAI credentials.
   2. Add **Simple Memory** node (`memoryBufferWindow`):
      - Session ID type: **customKey**
      - Session key: `{{ $('Telegram Trigger').item.json.message.chat.id }}`

7) **Voice flow: answer with agent**
   1. Add **AI Agent** node named “AI Agent1”.
   2. Prompt type: **define**
   3. System message:  
      “You are an ai agent for my telegram bot. you will be given a transcription of a voice message from telegram and you have to answer that user query.”
   4. Text input:  
      `here's the transcription:  {{ $json.content.parts[0].text }}`
   5. Connect **Transcribe a recording** → **AI Agent1**.
   6. Connect **OpenAI Chat Model** to **AI Agent1** via the **ai_languageModel** connector.
   7. Connect **Simple Memory** to **AI Agent1** via the **ai_memory** connector.

8) **Voice flow: generate audio**
   1. Add **OpenAI** node named “Generate audio”.
   2. Resource: **audio**
   3. Input: `{{ $json.output }}`
   4. Connect **AI Agent1** → **Generate audio**.
   5. Select **OpenAI credentials**.

9) **Voice flow: send audio**
   1. Add **Telegram** node named “Send an audio file”.
   2. Operation: **sendAudio**
   3. Chat ID: `{{ $('Telegram Trigger').item.json.message.chat.id }}`
   4. Enable **Binary Data** = true.
   5. Connect **Generate audio** → **Send an audio file**.

10) **Text flow: guardrails**
   1. Add **OpenAI Chat Model** node named “OpenAI Chat Model1” with model `gpt-4.1-mini`.
   2. Add **Guardrails** node:
      - Text: `{{ $json.message.text }}`
      - Guardrails rules: leave default/empty (or configure explicitly if desired)
   3. Connect **If (false)** → **Guardrails**.
   4. Connect **OpenAI Chat Model1** → **Guardrails** via **ai_languageModel**.

11) **Text flow: answer**
   1. Add **AI Agent** node named “AI Agent”.
   2. Prompt type: **guardrails**
   3. Connect **Guardrails (allowed output)** → **AI Agent**.
   4. Connect **OpenAI Chat Model** → **AI Agent** via **ai_languageModel**.
   5. Connect **Simple Memory** → **AI Agent** via **ai_memory**.

12) **Text flow: send answer**
   1. Add **Telegram** node named “Send a text message”.
   2. Text: `{{ $json.output }}`
   3. Chat ID: `{{ $('Telegram Trigger').item.json.message.chat.id }}`
   4. Set `appendAttribution` to **false** (Additional fields).
   5. Connect **AI Agent** → **Send a text message**.

13) **Text flow: blocked message**
   1. Add **Telegram** node named “Send a text message1”.
   2. Text: `Sorry! your text violates our policies`
   3. Chat ID: `{{ $('Telegram Trigger').item.json.message.chat.id }}`
   4. Set `appendAttribution` to **false**.
   5. Connect **Guardrails (blocked output)** → **Send a text message1**.

14) **Activate**
   - Turn the workflow **active** and test by sending **text** and an **OGG voice note** to your bot.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Create a Telegram bot using BotFather, obtain API key, configure in n8n Telegram nodes | Setup instructions (Sticky Note) |
| Get a free Gemini API key | https://aistudio.google.com/ |
| Get an OpenAI API key and top up credits | https://platform.openai.com/docs/overview |
| You can edit the system prompt in the AI Agent node according to your needs | Applies to voice and text handling notes (Sticky Note1/2) |