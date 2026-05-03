Get multimodal AI dating advice on Telegram with Google Gemini and Sheets

https://n8nworkflows.xyz/workflows/get-multimodal-ai-dating-advice-on-telegram-with-google-gemini-and-sheets-13446


# Get multimodal AI dating advice on Telegram with Google Gemini and Sheets

## 1. Workflow Overview

**Title:** Get multimodal AI dating advice on Telegram with Google Gemini and Sheets

**Purpose / use cases**
This workflow turns a Telegram bot into a “dating wingman” that can:
- Accept **text**, **voice notes**, and **screenshots/photos** from users
- Use **Google Gemini** for **vision** (image analysis) and **audio transcription**
- Maintain per-user context via **conversation memory**
- Store and retrieve user and lead data in **Google Sheets** (light CRM)
- Return advice to Telegram with **Telegram MarkdownV2-safe formatting** and auto-chunking

### 1.1 Auth & user check (gatekeeper)
- Entry point via Telegram Trigger
- Looks up a “Rizzler Profile” in Google Sheets by Telegram chat ID
- Routes to either onboarding (new user) or main agent (registered)

### 1.2 Input processing (text / image / voice)
- A Switch node detects message type
- Voice and image are downloaded from Telegram, then analyzed by Gemini
- Each input type is normalized into a `{ message, chat_id }` payload for the agent

### 1.3 Onboarding agent (new users)
- A Gemini-powered agent collects 4 profile fields (Name, Dating_Style, Goals, Language)
- Writes the new user profile to Google Sheets via a tool node
- Sends responses back to Telegram after MarkdownV2-safe formatting

### 1.4 Main “Rizz AI” agent (registered users)
- A Gemini chat model + buffer memory
- Uses Google Sheets “tools” to search/create/update leads and append logs
- Produces advice + a “Command Center” menu
- Output is passed through MarkdownV2-safe formatter + chunker, then sent to Telegram

### 1.5 Output formatting & delivery
- Code nodes convert agent output into Telegram-safe MarkdownV2 and split into 4000-char chunks
- Telegram sendMessage nodes deliver all chunks

---

## 2. Block-by-Block Analysis

### Block A — Auth & User Check

**Overview**
Receives Telegram messages, shows a typing indicator, checks whether the user exists in the “Rizzler Profile” sheet, then routes flow to onboarding or main logic.

**Nodes involved**
- Telegram Trigger
- Typing…
- Get Rizzler Profile
- Registered?
- (Downstream: Input Message Router / get_message (register))

#### A1) Telegram Trigger
- **Type / role:** `telegramTrigger` — workflow entry point
- **Config choices:**
  - Watches `updates: ["message"]` (any message type)
- **Key data used:** message payload at `$('Telegram Trigger').item.json.message`
- **Outputs:** to **Typing…** and **Get Rizzler Profile**
- **Failure modes / edge cases:**
  - Telegram credential invalid / bot token revoked
  - Webhook not set / wrong n8n public URL
  - Users sending non-`message` updates (not captured)

#### A2) Typing…
- **Type / role:** `telegram` — UX: sends “typing” chat action
- **Config choices:**
  - `operation: sendChatAction`
  - `chatId: {{$json.message.chat.id}}` (relies on trigger item structure)
- **Connections:** Trigger → Typing…
- **Failure modes:**
  - If input item doesn’t include `message.chat.id` (e.g., different update type), expression fails

#### A3) Get Rizzler Profile
- **Type / role:** `googleSheets` — lookup user profile row by Telegram chat ID
- **Config choices:**
  - `documentId`: the CRM spreadsheet (`Rizz_CRM`)
  - `sheetName`: “Rizzler Profile”
  - Filter: `ID == {{$('Telegram Trigger').item.json.message.chat.id}}`
  - `onError: continueRegularOutput` + `alwaysOutputData: true` (so workflow continues even if lookup fails)
- **Outputs:** to **Registered?**
- **Failure modes / edge cases:**
  - Google OAuth expired / missing permissions
  - Sheet renamed / wrong gid
  - Filter returns 0 rows → `$json.ID` missing in next node (expected)

#### A4) Registered?
- **Type / role:** `if` — determines whether the user exists
- **Config choices:**
  - Condition: “number exists” on `{{$json.ID}}`
  - This assumes `ID` is numeric; Telegram IDs are numeric but often stored as strings. If the sheet stores ID as text, “number exists” can behave unexpectedly.
- **Connections (as built):**
  - Output 0 → **Input Message Router**
  - Output 1 → **get_message (register)**
- **Important logic note**
  - The wiring suggests **registered users go to Input Message Router** and **unregistered users go to registration**, but you must verify which IF output corresponds to “true” in your n8n UI. If inverted, users may be onboarded incorrectly.
- **Failure modes:**
  - Type mismatch (string vs number) causing false negatives

---

### Block B — Input Processing (Text / Voice / Image)

**Overview**
Routes by message type, downloads media from Telegram when needed, runs Gemini analysis for image/audio, and normalizes the output into a consistent `message` string for the main agent.

**Nodes involved**
- Input Message Router
- get_message (text)
- Download Voice Message
- Analyze voice message
- get_message (Audio/Video message)
- Download Image
- Analyze image
- get_message (Media message)
- get_error_message1

#### B1) Input Message Router
- **Type / role:** `switch` — routes by message type
- **Config choices:**
  - Rule “Text”: `message.text exists`
  - Rule “Voice Message”: `message.voice exists`
  - Rule “Image”: `message.photo[0].file_id exists`
  - `fallbackOutput: "extra"`, `allMatchingOutputs: true`
- **Connections:**
  - Text → **get_message (text)**
  - Voice → **Download Voice Message**
  - Image → **Download Image**
  - Fallback (“extra”) → **get_error_message1**
- **Edge cases:**
  - Messages that include both text and photo (caption) can match multiple routes due to `allMatchingOutputs: true` → may invoke multiple downstream paths.
  - Some images may only have fewer photo sizes; index selection later must handle this.

#### B2) get_message (text)
- **Type / role:** `set` — normalizes incoming text
- **Config choices:**
  - `message = {{$('Telegram Trigger').item.json.message.text}}`
  - `chat_id = {{$('Telegram Trigger').item.json.message.chat.id}}`
- **Output:** to **Rizz AI**
- **Failure modes:**
  - If triggered by non-text, expression returns `undefined`, but this node only runs on text route.

#### B3) Download Voice Message
- **Type / role:** `telegram` — fetches binary file from Telegram servers
- **Config choices:**
  - `resource: file`
  - `fileId = {{$('Telegram Trigger').item.json.message.voice.file_id}}`
- **Output:** binary to **Analyze voice message**
- **Failure modes:**
  - File no longer available / Telegram API errors
  - Large files causing download timeout

#### B4) Analyze voice message
- **Type / role:** `googleGemini` (LangChain node) — audio understanding/transcription
- **Config choices:**
  - `resource: audio`, `operation: analyze`, `inputType: binary`
  - Prompt: “What’s in this audio message from telegram user?”
  - `model: models/gemini-3-flash-preview`
- **Output:** to **get_message (Audio/Video message)**
- **Failure modes:**
  - Gemini API key invalid / quota exceeded
  - Unsupported audio encoding
  - Output schema differences (handled partially downstream)

#### B5) get_message (Audio/Video message)
- **Type / role:** `set` — converts Gemini audio output to `message`
- **Config choices:**
  - `message = "Voice message description:" + ( $json.candidates?.[0]?.content?.parts?.[0]?.text || $json.content?.parts?.[0]?.text )`
  - `chat_id` from Telegram Trigger
- **Output:** to **Rizz AI**
- **Edge cases:**
  - Gemini node may return a different shape → message becomes empty; consider fallback text like “(transcription unavailable)”.

#### B6) Download Image
- **Type / role:** `telegram` — downloads the best available photo size
- **Config choices:**
  - `fileId = message.photo[3]?.file_id || photo[2] || photo[1]`
  - (Note: no fallback to `photo[0]`)
- **Output:** binary to **Analyze image**
- **Edge cases:**
  - If Telegram only provides `photo[0]`, this expression yields `undefined` and download fails.

#### B7) Analyze image
- **Type / role:** `googleGemini` (LangChain node) — screenshot/profile/chat analysis
- **Config choices:**
  - `resource: image`, `operation: analyze`, `inputType: binary`
  - Large structured prompt with strict “plain text fields” output requirements:
    - Context, Name, Details, Content, Visuals, Sentiment, Suggested Angle
  - `model: models/gemini-3-flash-preview`
  - `maxTries: 3`, `retryOnFail: true`, `waitBetweenTries: 2000ms`
- **Output:** to **get_message (Media message)**
- **Failure modes:**
  - Vision model refusal / safety filters depending on content
  - OCR inaccuracies
  - If prompt format is not respected, the agent later may have less reliable structured context

#### B8) get_message (Media message)
- **Type / role:** `set` — normalizes Gemini vision result
- **Config choices:**
  - `message = "Content:\n" + {{$json.content.parts[0].text}}`
  - `chat_id` from Telegram Trigger
- **Output:** to **Rizz AI**
- **Edge cases:**
  - If Gemini response is in `candidates[0]...` rather than `content.parts[0]...`, this fails (unlike the audio set node which checks both).

#### B9) get_error_message1
- **Type / role:** `set` — fallback error message for unsupported file types
- **Config choices:**
  - `message = "It was not possible to process the file.File type not supported."`
  - `chat_id` from Telegram Trigger
- **Output:** to **Rizz AI**
- **Note:** This still calls the agent; if you want a direct Telegram error reply, connect this to Markdown/Telegram send nodes instead.

---

### Block C — Onboarding Agent (New User Registration)

**Overview**
Handles new users by collecting required profile fields conversationally and writing them into Google Sheets via an AI tool call.

**Nodes involved**
- get_message (register)
- Register Agent
- Google Gemini Chat Model3
- Simple Memory2
- create_rizzler_profile
- MarkdownV1
- Send a text message3

#### C1) get_message (register)
- **Type / role:** `set` — normalizes initial registration text
- **Config choices:** same pattern:
  - `message = Telegram message.text`
  - `chat_id = Telegram chat.id`
- **Output:** to **Register Agent**
- **Edge cases:** If user starts onboarding by sending voice/image, this node won’t capture it.

#### C2) Register Agent
- **Type / role:** `langchain.agent` — collects profile slots and triggers tool when complete
- **Config choices:**
  - User input: `text = {{$json.message}}`
  - System instruction defines persona + required fields + tool usage rule:
    - Must call `create_rizzler_profile` only when all 4 fields are collected
  - Emphasizes post-tool “pivot” response format
- **Connections:**
  - Uses **Google Gemini Chat Model3** as language model (ai_languageModel)
  - Uses **Simple Memory2** (ai_memory)
  - Calls **create_rizzler_profile** (ai_tool)
  - Main output → **MarkdownV1**
- **Failure modes:**
  - Model not calling tool reliably (prompting mitigates but not guaranteed)
  - User gives ambiguous answers → agent may fill incorrectly
  - If tool errors, agent may continue without persistence

#### C3) Google Gemini Chat Model3
- **Type / role:** `lmChatGoogleGemini` — chat LLM backing the onboarding agent
- **Config choices:** `modelName: models/gemini-3-flash-preview`
- **Failure modes:** API key/quota/model availability

#### C4) Simple Memory2
- **Type / role:** `memoryBufferWindow` — stores short conversational context
- **Config choices:**
  - `sessionKey = {{$json.chat_id}}`
  - `sessionIdType: customKey`
  - (No explicit context window length shown; defaults apply)
- **Edge cases:** If `chat_id` missing, memory sessions collide or fail.

#### C5) create_rizzler_profile (Tool)
- **Type / role:** `googleSheetsTool` — appends a new row to “Rizzler Profile”
- **Config choices:**
  - Operation: `append`
  - Mapping uses `$fromAI()` fields:
    - `Name`, `Goals`, `Language`, `Daiting_Style` (note spelling: **Daiting_Style**)
  - `ID = {{$json.chat_id}}`
  - Matching columns configured as `["ID"]` but operation is append; matching is not used for append.
- **Failure modes / edge cases:**
  - Duplicates: append can create multiple rows for same ID if onboarding is repeated
  - Column name mismatch in Sheet (must match exactly, including misspelling)

#### C6) MarkdownV1 (Code)
- **Type / role:** `code` — Telegram MarkdownV2 sanitization + chunking
- **Config choices:**
  - Escapes special MarkdownV2 characters
  - Preserves limited formatting: `*bold*`, `_italic_`, `||spoiler||`, `[label](url)`
  - Normalizes headings `# Title` → `*Title*`
  - Splits into chunks using ~4000 char budget, outputs one item per chunk
  - Recommended mode: “Run Once for All Items”
- **Output:** to **Send a text message3**
- **Failure modes:**
  - If node is run “per item” while receiving multiple items, still works but chunking behavior may differ
  - Extremely long tokens (giant URLs) are hard-split, may still break Telegram formatting in rare cases

#### C7) Send a text message3
- **Type / role:** `telegram` — sends onboarding agent output
- **Config choices:**
  - `parse_mode: MarkdownV2`
  - `text = {{$json.message}}`
  - `chatId` from Telegram Trigger
  - `appendAttribution: false`
- **Edge cases:**
  - If Markdown escaping fails, Telegram rejects with “can’t parse entities”

---

### Block D — Main Rizz AI Agent + CRM Tools

**Overview**
For registered users, the main agent produces advice and optionally uses Google Sheets tools to manage leads and logs, with memory keyed to the Telegram chat.

**Nodes involved**
- Rizz AI
- Google Gemini Chat Model
- Simple Memory
- search_leads (tool)
- get_lead_history (tool)
- create_new_lead (tool)
- update_lead (tool)
- update_profile (tool)
- Append Log (tool)
- MarkdownV2
- Send a text message

#### D1) Rizz AI
- **Type / role:** `langchain.agent` — main decision-maker
- **Config choices:**
  - Input: `text = {{$json.message}}` (from normalized text/image/audio set nodes or error set)
  - System message:
    - Injects user context from **Get Rizzler Profile**: Name, Daiting_Style, Goals, Language
    - Enforces: “STRICTLY REPLY IN THIS LANGUAGE”
    - Human-in-the-loop directive: “NEVER touch the database without the user clicking/typing a command from the Menu”
    - Output format includes optional “3 reply options” and always a “COMMAND CENTER”
- **Connections:**
  - LLM: **Google Gemini Chat Model** (ai_languageModel)
  - Memory: **Simple Memory** (ai_memory)
  - Tools: **search_leads**, **get_lead_history**, **create_new_lead**, **update_lead**, **update_profile**, **Append Log** (ai_tool)
  - Main output → **MarkdownV2**
- **Important design gap**
  - The workflow does not include a dedicated “menu command parser” (e.g., an IF/switch that detects “1 Log This”, “2 Update Stage”, etc.). The agent is instructed not to modify DB without explicit command, but there’s no deterministic enforcement layer besides the instruction.
- **Failure modes / edge cases:**
  - Profile fields missing (newly registered but sheet not found) → system message includes empty values
  - Tool calls may fail if `$fromAI()` fields aren’t produced in correct format
  - Language enforcement depends on model compliance

#### D2) Google Gemini Chat Model
- **Type / role:** `lmChatGoogleGemini`
- **Config choices:** `modelName: models/gemini-3-flash-preview`
- **Failure modes:** quota/model availability, safety filters

#### D3) Simple Memory
- **Type / role:** `memoryBufferWindow`
- **Config choices:**
  - `sessionKey = {{$json.chat_id}}`
  - `contextWindowLength: 10`
- **Edge cases:** if chat_id missing/incorrect, memory cross-contaminates users.

#### D4) search_leads (Tool)
- **Type / role:** `googleSheetsTool` — find a lead by name under the current Rizzler ID
- **Config choices:**
  - Filters:
    - `Name == $fromAI('values0_Value', ...)` (the searched name)
    - `ID == {{$json.chat_id}}` (this appears to filter on column “ID”; your Leads sheet must have “ID” representing the rizzler/user, but later nodes use “Rizzler ID”. This mismatch is a likely bug.)
  - Sheet: “Leads” (gid=0)
  - Tool description indicates it returns the lead object
- **High-risk mismatch**
  - Leads schema in other nodes uses **“Rizzler ID”** column, but this tool filters on **“ID”** column. If your Leads sheet doesn’t have “ID” as Rizzler ID, searches will fail.
- **Failure modes:** no results due to column mismatch, sheet permissions, name spelling variance

#### D5) get_lead_history (Tool)
- **Type / role:** `googleSheetsTool` — fetch conversation logs for a Lead ID
- **Config choices:**
  - Filter: `Lead ID == $fromAI('values0_Value', ...)` (expects tool input to provide the lead’s ID)
  - Sheet: “Conversation Logs”
- **Edge cases:** if “Lead ID” is wrong or logs are large, may return many rows.

#### D6) create_new_lead (Tool)
- **Type / role:** `googleSheetsTool` — append a new lead row
- **Config choices:**
  - `ID = {{ Math.random().toString(16).substring(8) }}` (short random hex; collision possible)
  - `Stage = 📥 New Match`
  - `Rizzler ID = {{$json.chat_id}}`
  - `Last Contact = {{$now}}`
  - `Interest Score = 50`
  - `Profile Summary = $fromAI('Profile_Summary', ...)`
- **Edge cases:**
  - Random ID collisions in high volume
  - Sheet column names must match exactly (including emoji in Stage)

#### D7) update_lead (Tool)
- **Type / role:** `googleSheetsTool` — update an existing lead row by ID
- **Config choices:**
  - `matchingColumns: ["ID"]`
  - Requires full row update fields; tool description warns to resend name and profile summary even if unchanged.
  - Uses `$fromAI()` fields for ID, Name, Stage (enum), Next Action, Interest Score, Profile Summary
  - `Rizzler ID = {{$json.chat_id}}`, `Last Contact = {{$now}}`
- **Edge cases:**
  - If agent omits a required field, update may blank columns or fail
  - If Stage not exactly matching enum string (including emoji), downstream analytics may break

#### D8) update_profile (Tool)
- **Type / role:** `googleSheetsTool` — update user profile row in “Rizzler Profile”
- **Config choices:**
  - `matchingColumns: ["ID"]`
  - `$fromAI()` supplies Name/Goals/Language/Daiting_Style
  - Tool description warns it performs full row update and must resend all values
- **Edge cases:** partial updates can overwrite with empty values if agent fails to provide them.

#### D9) Append Log (Tool)
- **Type / role:** `googleSheetsTool` — append to “Conversation Logs”
- **Config choices:**
  - `Date = {{$now}}`
  - `Lead ID = $fromAI('Lead_ID', ...)`
  - `Summary = $fromAI('Summary', ...)`
- **Edge cases:** agent may log under wrong lead ID if not constrained by prior search.

#### D10) MarkdownV2 (Code)
- **Type / role:** same MarkdownV2-safe formatter/chunker as MarkdownV1
- **Output:** to **Send a text message**
- **Note:** This duplication is intentional for separate branches but you can reuse one code node if you unify routing.

#### D11) Send a text message
- **Type / role:** `telegram` — final send for main agent outputs
- **Config choices:**
  - `parse_mode: MarkdownV2`
  - `text = {{$json.message}}`
  - `chatId` from Telegram Trigger
- **Edge cases:** Telegram rejects invalid Markdown entities; formatter mitigates this.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Telegram Trigger | telegramTrigger | Entry point: receive Telegram messages | — | Typing…, Get Rizzler Profile | ## 🚦 Auth & User Check … |
| Typing… | telegram | Send “typing” action | Telegram Trigger | — | ## 🚦 Auth & User Check … |
| Get Rizzler Profile | googleSheets | Lookup user profile by Telegram ID | Telegram Trigger | Registered? | ## 🚦 Auth & User Check … |
| Registered? | if | Gate registered vs new user | Get Rizzler Profile | Input Message Router, get_message (register) | ## 🚦 Auth & User Check … |
| Input Message Router | switch | Route Text vs Voice vs Image vs fallback | Registered? | get_message (text), Download Voice Message, Download Image, get_error_message1 | ## 👁️ Input Processing … |
| get_message (text) | set | Normalize text into message/chat_id | Input Message Router | Rizz AI | ## 👁️ Input Processing … |
| Download Voice Message | telegram | Download voice binary | Input Message Router | Analyze voice message | ## 👁️ Input Processing … |
| Analyze voice message | googleGemini (audio analyze) | Transcribe/understand audio | Download Voice Message | get_message (Audio/Video message) | ## 👁️ Input Processing … |
| get_message (Audio/Video message) | set | Normalize audio analysis into message/chat_id | Analyze voice message | Rizz AI | ## 👁️ Input Processing … |
| Download Image | telegram | Download image binary | Input Message Router | Analyze image | ## 👁️ Input Processing … |
| Analyze image | googleGemini (image analyze) | Analyze screenshot/profile/chat | Download Image | get_message (Media  message) | ## 👁️ Input Processing … |
| get_message (Media  message) | set | Normalize image analysis into message/chat_id | Analyze image | Rizz AI | ## 👁️ Input Processing … |
| get_error_message1 | set | Fallback error message | Input Message Router | Rizz AI | ## 👁️ Input Processing … |
| Rizz AI | langchain.agent | Main agent: advice + tool usage | get_message (text) / media/audio set / get_error_message1 | MarkdownV2 | ## 🧠 The Rizz Agent (Brain) … |
| Google Gemini Chat Model | lmChatGoogleGemini | LLM for main agent | — | Rizz AI (ai_languageModel) | ## 🧠 The Rizz Agent (Brain) … |
| Simple Memory | memoryBufferWindow | Per-user memory (contextWindow=10) | — | Rizz AI (ai_memory) | ## 🧠 The Rizz Agent (Brain) … |
| search_leads | googleSheetsTool | Tool: search leads by name | — | Rizz AI (ai_tool) | ## 🧠 The Rizz Agent (Brain) … |
| get_lead_history | googleSheetsTool | Tool: fetch conversation logs | — | Rizz AI (ai_tool) | ## 🧠 The Rizz Agent (Brain) … |
| create_new_lead | googleSheetsTool | Tool: append new lead | — | Rizz AI (ai_tool) | ## 🧠 The Rizz Agent (Brain) … |
| update_lead | googleSheetsTool | Tool: update lead row | — | Rizz AI (ai_tool) | ## 🧠 The Rizz Agent (Brain) … |
| update_profile | googleSheetsTool | Tool: update user profile row | — | Rizz AI (ai_tool) | ## 🧠 The Rizz Agent (Brain) … |
| Append Log | googleSheetsTool | Tool: append conversation log | — | Rizz AI (ai_tool) | ## 🧠 The Rizz Agent (Brain) … |
| MarkdownV2 | code | Telegram MarkdownV2 escaping + chunking | Rizz AI | Send a text message | ## 🧠 The Rizz Agent (Brain) … |
| Send a text message | telegram | Send final advice to Telegram | MarkdownV2 | — | ## 🧠 The Rizz Agent (Brain) … |
| get_message (register) | set | Normalize onboarding text | Registered? | Register Agent | ## 👶 Onboarding Agent … |
| Register Agent | langchain.agent | Onboarding slot-fill + create profile tool | get_message (register) | MarkdownV1 | ## 👶 Onboarding Agent … |
| Google Gemini Chat Model3 | lmChatGoogleGemini | LLM for onboarding agent | — | Register Agent (ai_languageModel) | ## 👶 Onboarding Agent … |
| Simple Memory2 | memoryBufferWindow | Memory for onboarding | — | Register Agent (ai_memory) | ## 👶 Onboarding Agent … |
| create_rizzler_profile | googleSheetsTool | Tool: append new user profile | — | Register Agent (ai_tool) | ## 👶 Onboarding Agent … |
| MarkdownV1 | code | Telegram MarkdownV2 escaping + chunking (onboarding) | Register Agent | Send a text message3 | ## 👶 Onboarding Agent … |
| Send a text message3 | telegram | Send onboarding messages to Telegram | MarkdownV1 | — | ## 👶 Onboarding Agent … |
| Sticky Note | stickyNote | Comment | — | — | (content is the note itself) |
| Sticky Note1 | stickyNote | Comment | — | — | (content is the note itself) |
| Sticky Note2 | stickyNote | Comment | — | — | (content is the note itself) |
| Sticky Note3 | stickyNote | Comment | — | — | (content is the note itself) |
| Sticky Note4 | stickyNote | Branding/setup note | — | — | (content is the note itself) |
| Sticky Note5 | stickyNote | Image note | — | — | (content is the note itself) |
| Sticky Note6 | stickyNote | Image note | — | — | (content is the note itself) |
| Sticky Note7 | stickyNote | Image note | — | — | (content is the note itself) |
| Sticky Note8 | stickyNote | Image note | — | — | (content is the note itself) |
| Sticky Note15 | stickyNote | Contact note | — | — | (content is the note itself) |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Telegram bot + credentials**
   1. Create a bot via **@BotFather** and copy the bot token.
   2. In n8n: create **Telegram API** credentials using the token.

2) **Create / copy Google Sheets CRM + credentials**
   1. Copy the provided sheet template:  
      https://docs.google.com/spreadsheets/d/1JxoahgYNHc6nuWJ-VOsHlEzaKYxnksGFeYA0TKE4lWo/edit?usp=sharing
   2. Ensure sheets/tabs exist (names must match nodes):
      - `Rizzler Profile`
      - `Leads`
      - `Conversation Logs`
   3. Ensure column headers match exactly (including the misspelling used by nodes):  
      `Daiting_Style`, `Rizzler ID`, `Interest Score`, etc.
   4. In n8n: create **Google Sheets OAuth2** credentials with access to that spreadsheet.

3) **Create Gemini (Google AI Studio) credentials**
   1. Get an API key from Google AI Studio / Gemini.
   2. In n8n: create **Google PaLM / Gemini** credentials (node uses `googlePalmApi`).

4) **Build Block A (Trigger + profile check)**
   1. Add **Telegram Trigger**
      - Updates: `message`
   2. Add **Telegram node** “Typing…”
      - Operation: `Send Chat Action`
      - Chat ID: `{{$json.message.chat.id}}`
   3. Add **Google Sheets** node “Get Rizzler Profile”
      - Operation: read with filter UI
      - Document: your copied CRM sheet
      - Sheet: `Rizzler Profile`
      - Filter: column `ID` equals `{{$('Telegram Trigger').item.json.message.chat.id}}`
      - Enable: continue on fail / always output data
   4. Add **IF** node “Registered?”
      - Condition: check if `$json.ID` exists
      - Prefer a robust check: “exists” (string) rather than “number exists” if your IDs are stored as text.

5) **Build Block B (Router + multimodal preprocessing)**
   1. Add **Switch** node “Input Message Router”
      - Rules:
        - Text exists: `{{$('Telegram Trigger').item.json.message.text}}`
        - Voice exists: `{{$('Telegram Trigger').item.json.message.voice}}`
        - Photo exists: `{{$('Telegram Trigger').item.json.message.photo[0].file_id}}`
      - Fallback output: `extra`
      - Allow all matching outputs: decide based on your preference (safer: false).
   2. Add **Set** node “get_message (text)”
      - `message = {{$('Telegram Trigger').item.json.message.text}}`
      - `chat_id = {{$('Telegram Trigger').item.json.message.chat.id}}`
   3. Add **Telegram** node “Download Voice Message”
      - Resource: `File`
      - File ID: `{{$('Telegram Trigger').item.json.message.voice.file_id}}`
   4. Add **Google Gemini** node “Analyze voice message”
      - Resource: `audio`
      - Operation: `analyze`
      - Input: `binary`
      - Model: `models/gemini-3-flash-preview`
   5. Add **Set** node “get_message (Audio/Video message)”
      - `message = Voice message description:{{ $json.candidates?.[0]?.content?.parts?.[0]?.text || $json.content?.parts?.[0]?.text }}`
      - `chat_id = {{$('Telegram Trigger').item.json.message.chat.id}}`
   6. Add **Telegram** node “Download Image”
      - Resource: `File`
      - File ID: `{{$('Telegram Trigger').item.json.message.photo[3]?.file_id || ...photo[0]?.file_id}}` (recommended to include `[0]` fallback)
   7. Add **Google Gemini** node “Analyze image”
      - Resource: `image`
      - Operation: `analyze`
      - Input: `binary`
      - Model: `models/gemini-3-flash-preview`
      - Prompt: the structured “Dating Forensics & Social Dynamics Expert…” prompt
      - Retries: enable if desired
   8. Add **Set** node “get_message (Media message)”
      - `message = Content:\n{{ $json.content.parts[0].text }}`
      - `chat_id = {{$('Telegram Trigger').item.json.message.chat.id}}`
   9. Add **Set** node “get_error_message1”
      - `message = It was not possible to process the file.File type not supported.`
      - `chat_id = {{$('Telegram Trigger').item.json.message.chat.id}}`

6) **Build Block C (Onboarding path)**
   1. Add **Set** node “get_message (register)” (same as text normalizer).
   2. Add **Gemini Chat Model** node “Google Gemini Chat Model3”
      - Model: `models/gemini-3-flash-preview`
   3. Add **Memory Buffer Window** “Simple Memory2”
      - Session key: `{{$json.chat_id}}`
      - Session ID type: custom key
   4. Add **AI Agent** node “Register Agent”
      - Input text: `{{$json.message}}`
      - System message: onboarding persona + slot filling + tool call rule
      - Connect:
        - ai_languageModel ← Gemini Chat Model3
        - ai_memory ← Simple Memory2
   5. Add **Google Sheets Tool** node “create_rizzler_profile”
      - Operation: `append`
      - Sheet: `Rizzler Profile`
      - Columns:
        - ID = `{{$json.chat_id}}`
        - Name/Goals/Language/Daiting_Style from `$fromAI(...)`
      - Connect as ai_tool to Register Agent
   6. Add **Code** node “MarkdownV1”
      - Paste the MarkdownV2-safe formatter/chunker code
      - Set to “Run Once for All Items” (recommended)
   7. Add **Telegram** node “Send a text message3”
      - Operation: `Send Message`
      - Parse mode: `MarkdownV2`
      - Text: `{{$json.message}}`

7) **Build Block D (Main agent + tools)**
   1. Add **Gemini Chat Model** “Google Gemini Chat Model”
   2. Add **Memory Buffer Window** “Simple Memory”
      - Session key: `{{$json.chat_id}}`
      - Context window length: `10`
   3. Add **AI Agent** “Rizz AI”
      - Input text: `{{$json.message}}`
      - System message: the main persona + language enforcement + Command Center formatting
      - Connect ai_languageModel and ai_memory
   4. Add Google Sheets **Tool nodes** and connect as ai_tool to “Rizz AI”:
      - `search_leads` (Leads sheet filter by lead name and rizzler ID; ensure column names match your sheet)
      - `get_lead_history` (Conversation Logs by Lead ID)
      - `create_new_lead` (append to Leads)
      - `update_lead` (update Leads by ID)
      - `Append Log` (append Conversation Logs)
      - `update_profile` (update Rizzler Profile by ID)
   5. Add **Code** node “MarkdownV2”
      - Same MarkdownV2-safe formatter/chunker code
   6. Add **Telegram** node “Send a text message”
      - Parse mode: MarkdownV2
      - Text: `{{$json.message}}`

8) **Wire the control flow**
   1. Telegram Trigger → Typing…
   2. Telegram Trigger → Get Rizzler Profile → Registered?
   3. Registered? (registered/true branch) → Input Message Router
   4. Registered? (new/false branch) → get_message (register) → Register Agent → MarkdownV1 → Send a text message3
   5. Input Message Router:
      - Text → get_message (text) → Rizz AI
      - Voice → Download Voice Message → Analyze voice message → get_message (Audio/Video message) → Rizz AI
      - Image → Download Image → Analyze image → get_message (Media message) → Rizz AI
      - Extra → get_error_message1 → Rizz AI
   6. Rizz AI → MarkdownV2 → Send a text message

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Google Sheets template to copy | https://docs.google.com/spreadsheets/d/1JxoahgYNHc6nuWJ-VOsHlEzaKYxnksGFeYA0TKE4lWo/edit?usp=sharing |
| Workflow description and setup checklist (as provided in canvas) | “Rizz AI: Your AI Dating Assistant on Telegram 💘” sticky note content |
| Onboarding workflow image | https://raw.githubusercontent.com/AlejandroSIlvaRodriguez/Assets/main/Assets/Onboarding.png |
| Chat response example image | https://raw.githubusercontent.com/AlejandroSIlvaRodriguez/Assets/main/Assets/ChatResponse.png |
| Menu response example image | https://raw.githubusercontent.com/AlejandroSIlvaRodriguez/Assets/main/Assets/MenuResponse.png |
| Profile response example image (note: filename has a typo “Resoponce”) | https://raw.githubusercontent.com/AlejandroSIlvaRodriguez/Assets/main/Assets/ProfileResoponce.png |
| Contact / assistance | Email: [johnsilva11031@gmail.com](mailto:johnsilva11031@gmail.com) — LinkedIn: https://www.linkedin.com/in/john-alejandro-silva-rodriguez-48093526b/ |

--- 

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.