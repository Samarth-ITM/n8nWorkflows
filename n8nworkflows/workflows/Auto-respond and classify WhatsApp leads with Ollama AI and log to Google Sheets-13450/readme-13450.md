Auto-respond and classify WhatsApp leads with Ollama AI and log to Google Sheets

https://n8nworkflows.xyz/workflows/auto-respond-and-classify-whatsapp-leads-with-ollama-ai-and-log-to-google-sheets-13450


# Auto-respond and classify WhatsApp leads with Ollama AI and log to Google Sheets

## 1. Workflow Overview

**Workflow name:** Auto-respond and classify WhatsApp leads with AI and log to Google Sheets  
**Purpose:** Automatically handle incoming WhatsApp Business messages by (1) preventing reply loops, (2) classifying the sender intent with an LLM (Ollama chat model via LangChain Agent), (3) routing to the correct path, (4) logging all interactions to Google Sheets, (5) booking Google Calendar meetings (hot/warm), and (6) sending WhatsApp replies (meeting links, support confirmation, or qualification questions).

**Primary use cases**
- Sales lead triage on WhatsApp (hot vs warm)
- Automated support acknowledgment
- Automated qualification when initial messages are too vague
- Central logging in Google Sheets + fast follow-up via auto-booked meetings

### 1.1 Input Reception & Normalization
Receives WhatsApp webhook events and converts them into a normalized internal format, while skipping status events and the bot’s own outgoing messages to avoid infinite loops.

### 1.2 AI Classification (with conversation memory)
Uses an LLM agent to classify intent into one of four categories and (for “needs qualification”) generate a follow-up question. A memory buffer is attached to keep short conversational context per WhatsApp user.

### 1.3 Robust Extraction & Validation
Parses/normalizes the AI output into a predictable schema, with multiple fallback parsing strategies and defaulting to `needs_qualification` if parsing fails.

### 1.4 Intent Routing
A Switch node routes the flow into one of the downstream paths.

### 1.5 Hot Lead Path
Generates a “hot lead” reply text, logs to Google Sheets, books a Google Calendar meeting with Meet link, sends meeting link via WhatsApp.

### 1.6 Warm Lead Path
Generates a “warm lead” reply text, logs to Google Sheets, books a meeting later than hot leads, sends meeting link via WhatsApp.

### 1.7 Support Path
Generates support confirmation, logs to Google Sheets, sends confirmation via WhatsApp.

### 1.8 Qualification Path
Uses the AI-generated `nextQuestion` (or smart defaults) to ask a follow-up question and keep the conversation moving.

---

## 2. Block-by-Block Analysis

### Block 1.1 — Input Reception & Normalization
**Overview:** Listens for incoming WhatsApp messages and converts them into a consistent internal object. Filters out status events and bot-originated messages to prevent self-triggered loops.

**Nodes involved**
- WhatsApp Trigger
- 🔧 Parse WhatsApp Message

#### Node: WhatsApp Trigger
- **Type / role:** `n8n-nodes-base.whatsAppTrigger` — entry point webhook for Meta WhatsApp Cloud API events.
- **Key configuration (interpreted):**
  - Subscribes to update type: `messages` only.
- **Input / output:**
  - No input (trigger).
  - Output goes to **🔧 Parse WhatsApp Message**.
- **Edge cases / failures:**
  - Meta webhook verification/config errors (wrong callback URL/token).
  - Missing permissions or misconfigured WhatsApp Business account/phone number.
  - Payload differences depending on message type (text vs media) can cause downstream “messageBody” to be empty.
- **Version notes:** Node `typeVersion: 1`.

#### Node: 🔧 Parse WhatsApp Message
- **Type / role:** `n8n-nodes-base.code` — normalizes WhatsApp payload and applies loop prevention.
- **Key configuration choices:**
  - Defines `BOT_NUMBERS = ['YOUR_PHONE_NUMBER_ID', 'YOUR_DISPLAY_PHONE_NUMBER']` and checks if `message.from` matches; if yes, marks `skipProcessing: true`.
  - Handles three cases:
    1. Incoming `messages[0]` → produces normalized message fields
    2. `statuses[0]` → returns eventType `status` and `skipProcessing: true`
    3. Unknown payload → returns eventType `unknown` and `skipProcessing: true`
  - Converts WhatsApp epoch `timestamp` (seconds) into ISO `receivedAt`.
- **Key fields created (outputs):**
  - `from`, `name`, `messageBody`, `messageId`, `messageType`, `timestamp`, `waId`, `phoneNumberId`, `receivedAt`, `skipProcessing`, `eventType`
- **Connections:**
  - Input from **WhatsApp Trigger**
  - Output to **🤖 AI Intent Classifier**
- **Edge cases / failures:**
  - If message is not text (image/audio), `message.text?.body` is empty → classification may be weak.
  - `BOT_NUMBERS` must be updated; otherwise bot may reply to itself depending on Meta “from” values, causing loop spam.
  - Timestamp parsing assumes `message.timestamp` exists and is numeric.
- **Version notes:** Code node `typeVersion: 2`.

---

### Block 1.2 — AI Classification (with conversation memory)
**Overview:** Uses a LangChain agent to classify the message. Conversation memory is attached using `waId` as session key, enabling better reclassification on follow-up replies.

**Nodes involved**
- Ollama Chat Model
- Simple Memory
- 🤖 AI Intent Classifier

#### Node: Ollama Chat Model
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOllama` — provides the chat model backend to the agent.
- **Key configuration choices:**
  - Model: `minimax-m2:cloud` (as configured; can be any Ollama-available model).
- **Connections:**
  - Connected to **🤖 AI Intent Classifier** via `ai_languageModel`.
- **Edge cases / failures:**
  - Ollama endpoint unreachable / DNS / network errors.
  - Model not pulled/available or name mismatch.
  - Latency/timeouts under load.
- **Version notes:** `typeVersion: 1`.

#### Node: Simple Memory
- **Type / role:** `@n8n/n8n-nodes-langchain.memoryBufferWindow` — stores recent chat context per user.
- **Key configuration choices:**
  - `sessionIdType: customKey`
  - `sessionKey: =={{ $json.waId }}` (note the double `==`; see edge case below)
  - `contextWindowLength: 20`
- **Connections:**
  - Connected to **🤖 AI Intent Classifier** via `ai_memory`.
- **Edge cases / failures:**
  - **Potential expression bug:** `sessionKey` is set to `=={{ $json.waId }}`. In n8n, expressions typically use `={{ ... }}`. The leading extra `=` may cause the literal string to be used, resulting in all users sharing the same memory key or an invalid key. This should likely be `={{ $json.waId }}`.
  - If `waId` is empty, multiple users may collide into the same memory session.
- **Version notes:** `typeVersion: 1.3`.

#### Node: 🤖 AI Intent Classifier
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — prompts the model to classify the incoming message.
- **Key configuration choices:**
  - Prompt text includes:
    - `From: {{ $json.name }}`
    - `Phone: {{ $json.from }}`
    - `Message: {{ $json.messageBody }}`
  - System message enforces:
    - Allowed intents: `hot_lead`, `warm_lead`, `support`, `needs_qualification`
    - Strict rules: greetings/pricing-without-context → always `needs_qualification`
    - Output must be **raw JSON** only (no markdown fences), with keys:
      - `intent`, `confidence`, `reason`, `hasEnoughInfo`, `missingInfo`, `nextQuestion`
  - `hasOutputParser: true` (agent will attempt to parse structured output).
- **Connections:**
  - Input from **🔧 Parse WhatsApp Message**
  - Uses **Ollama Chat Model** via `ai_languageModel`
  - Uses **Simple Memory** via `ai_memory`
  - Output to **🎯 Extract Classification**
- **Edge cases / failures:**
  - Model may still produce non-JSON output; downstream parser compensates.
  - If `messageBody` is empty (non-text message), classification likely defaults to qualification.
- **Version notes:** `typeVersion: 2.1`.

---

### Block 1.3 — Robust Extraction & Validation
**Overview:** Normalizes the agent output to a consistent schema, attempts multiple ways to locate/parse JSON, validates intent, and applies safe defaults on failure.

**Nodes involved**
- 🎯 Extract Classification

#### Node: 🎯 Extract Classification
- **Type / role:** `n8n-nodes-base.code` — robust parser/validator and schema builder.
- **Key configuration choices:**
  - Reads original message from: `$node["🔧 Parse WhatsApp Message"].json`
  - Attempts to extract AI JSON from many possible nesting shapes (`output`, `text`, `content`, `kwargs`, `lc_kwargs`, etc.).
  - `extractJSON()` removes ```json fences and tries `JSON.parse()`, fallback regex to extract a `{ ... "intent": ... }` region.
  - Validates `intent` against:
    - `['hot_lead', 'warm_lead', 'support', 'spam', 'needs_qualification']`
    - Note: `spam` is allowed here, even though the system prompt defines four categories. (Switch includes spam rule but isn’t connected downstream; see routing.)
  - If parsing fails, defaults to:
    - `intent: needs_qualification`
    - `confidence: 0.5`
    - `missingInfo: ['parsing failed - needs manual review']`
    - `nextQuestion: 'Hi! Thanks for reaching out 😊 How can I help you today?'`
  - Outputs combined object including metadata and classification fields and sets `model: 'your-ai-model'`.
- **Connections:**
  - Input from **🤖 AI Intent Classifier**
  - Output to **🔀 Route by Intent**
- **Edge cases / failures:**
  - If `$node["🔧 Parse WhatsApp Message"]` is missing (renamed/deleted), references break.
  - If AI returns partial JSON (trailing commas, single quotes), parse fails and default triggers.
  - Accepting `spam` intent but not handling it downstream may cause drop-off (see Router block).
- **Version notes:** `typeVersion: 2`.

---

### Block 1.4 — Intent Routing
**Overview:** Directs the normalized classification to the appropriate downstream response/logging/booking path.

**Nodes involved**
- 🔀 Route by Intent

#### Node: 🔀 Route by Intent
- **Type / role:** `n8n-nodes-base.switch` — routes by `$json.intent`.
- **Key configuration choices:**
  - Rules (equals):
    - `hot_lead`
    - `warm_lead`
    - `support`
    - `needs_qualification`
    - `spam`
  - `allMatchingOutputs: false` (first matching route only).
- **Connections (as wired):**
  - Output 1 → **💬 Hot Lead Reply**
  - Output 2 → **💬 Warm Lead Reply**
  - Output 3 → **💬 Support Reply**
  - Output 4 → **🧠 Smart Qualification Reply**
  - **Spam output is not connected** (classification “spam” effectively ends here without logging/reply).
- **Edge cases / failures:**
  - If intent is `spam`, nothing happens downstream (no log, no reply) unless a connection is added.
  - If `skipProcessing` is true, routing still occurs because there is no IF guard; however Parse returns `skipProcessing: true` mainly for statuses/outgoing and also sets missing message fields, so classification may default. A production-hardening would add an IF node to stop early.
- **Version notes:** `typeVersion: 3.2`.

---

### Block 1.5 — Hot Lead Path (reply → log → book → send meet)
**Overview:** Creates a premium “hot lead” confirmation message, logs to Sheets, books a meeting soon, then sends the Meet link to the lead on WhatsApp.

**Nodes involved**
- 💬 Hot Lead Reply
- 📊 Log Hot Lead
- 📅 Book Meeting (Hot)
- 📱 Send Meeting (Hot)

#### Node: 💬 Hot Lead Reply
- **Type / role:** `n8n-nodes-base.code` — crafts a hot lead reply template.
- **Key configuration choices:**
  - Uses `name` with fallback `'there'`.
  - Outputs:
    - `autoReply` message
    - `shouldReply: true`
    - plus passes through original JSON via spread.
- **Connections:**
  - Input from **🔀 Route by Intent**
  - Output to **📊 Log Hot Lead**
- **Edge cases / failures:**
  - Reply is generated but never directly sent in this path; only the meeting invite is sent later. If you intended to send this immediate reply too, an additional WhatsApp Send node is needed here.

#### Node: 📊 Log Hot Lead
- **Type / role:** `n8n-nodes-base.googleSheets` — append-or-update lead row in Google Sheets.
- **Key configuration choices:**
  - Operation: `appendOrUpdate`
  - Matching column: `From` (acts like a unique key per phone number; repeated leads update the same row).
  - Maps many fields including:
    - `From`, `Name`, `Intent`, `Reason`, `Confidence`, `message body`, `message id`, timestamps, `Missing Info`, `Question Asked`
  - Status computed with expression:
    - URGENT for hot leads; other statuses for other intents (even though this node is on hot path).
  - Document and Sheet placeholders: `YOUR_GOOGLE_SHEET_ID`, `gid=0`.
- **Connections:**
  - Output to **📅 Book Meeting (Hot)**
- **Edge cases / failures:**
  - OAuth credential missing/expired.
  - Sheet column names must exactly match (e.g., `recived at` is misspelled; must exist in the sheet exactly as written).
  - Matching only on `From` means multiple different conversations from same number overwrite prior row instead of creating history.
- **Version notes:** `typeVersion: 4.6`.

#### Node: 📅 Book Meeting (Hot)
- **Type / role:** `n8n-nodes-base.googleCalendar` — creates an event and requests Meet conference data.
- **Key configuration choices:**
  - Start: `now + 2 hours`
  - End: `now + 3 hours`
  - Calendar: `user@example.com` (placeholder)
  - Description uses fields like `{{ $json.Intent }}`, `{{ $json.Reason }}` and `{{ $json['message body'] }}`
    - These keys come from the Google Sheets node output shape (column names), not from the original classification object.
  - Conference data UI is configured, but `conferenceSolution` is set to `={{ $json.Name }}` (likely incorrect usage; conference solution is usually a fixed enum like “hangoutsMeet” depending on node implementation).
- **Connections:**
  - Output to **📱 Send Meeting (Hot)**
- **Edge cases / failures:**
  - Calendar OAuth issues/permissions.
  - Conference/Meet creation may fail if the Calendar API settings don’t allow it.
  - Description references depend on prior node output keys; changing sheet column names can break the meeting description.
- **Version notes:** `typeVersion: 1.2`.

#### Node: 📱 Send Meeting (Hot)
- **Type / role:** `n8n-nodes-base.whatsApp` — sends meeting confirmation message with Meet link.
- **Key configuration choices:**
  - Operation: `send`
  - `phoneNumberId: YOUR_PHONE_NUMBER_ID` (must be replaced)
  - Recipient: `={{ $('🎯 Extract Classification').item.json.from }}`
  - Text includes Meet link: `{{ $json.hangoutLink }}`
  - Uses formatted time based on `$now.plus(2, 'hours')...`
- **Connections:**
  - Terminal node for hot path.
- **Edge cases / failures:**
  - WhatsApp credentials and template/policy constraints (Cloud API limits; sending free-form text typically allowed within 24h customer care window).
  - If Calendar node does not return `hangoutLink`, message will contain blank link.
- **Version notes:** `typeVersion: 1.1`.

---

### Block 1.6 — Warm Lead Path (reply → log → book → send meet)
**Overview:** Creates a warm lead informational reply, logs to Sheets, books a later meeting, and sends the Meet link.

**Nodes involved**
- 💬 Warm Lead Reply
- 📊 Log Warm Lead
- 📅 Book Meeting (Warm)
- 📱 Send Meeting (Warm)

#### Node: 💬 Warm Lead Reply
- **Type / role:** Code — builds a warm lead response template with links.
- **Key configuration choices:**
  - Includes placeholder links: `your-site.com`, `your-site.com/cases`
  - Outputs `autoReply` and `shouldReply: true` (but not sent in this path—same caveat as hot).
- **Connections:** Switch → this node → **📊 Log Warm Lead**
- **Edge cases:** Same as hot reply node (message generated but not sent unless you add a send node).

#### Node: 📊 Log Warm Lead
- **Type / role:** Google Sheets append-or-update.
- **Key configuration choices:**
  - Status hard-coded: `Follow Up`
  - Matching: `From`
  - Same column naming requirements.
- **Connections:** → **📅 Book Meeting (Warm)**
- **Edge cases:** Same as other Sheets nodes.

#### Node: 📅 Book Meeting (Warm)
- **Type / role:** Google Calendar event creation.
- **Key configuration choices:**
  - Start: `now + 4 hours`
  - End: `now + 5 hours`
  - Description uses sheet-output keys.
  - Same potential conference configuration issue.
- **Connections:** → **📱 Send Meeting (Warm)**

#### Node: 📱 Send Meeting (Warm)
- **Type / role:** WhatsApp send.
- **Key configuration choices:**
  - Recipient from Extract Classification `from`
  - Uses `{{ $json.hangoutLink }}`
  - Time formatted based on `now + 4 hours`
- **Edge cases:** Same as hot send node.

---

### Block 1.7 — Support Path
**Overview:** Acknowledges support requests, logs them, and sends a confirmation message containing a ticket ID derived from message ID.

**Nodes involved**
- 💬 Support Reply
- 📊 Log Support
- 📱 Send Support Reply

#### Node: 💬 Support Reply
- **Type / role:** Code — creates support confirmation message.
- **Key configuration choices:**
  - Ticket ID derived from message ID first 8 chars: `substring(0, 8).toUpperCase()`
  - Issue preview: first 50 chars of message body.
  - Outputs `autoReply` and `shouldReply: true`.
- **Connections:** Switch → this node → **📊 Log Support**
- **Edge cases:**
  - If `messageId` is missing/unexpected, ticket becomes `UNKNOWN`.
  - Newlines/formatting should be checked in WhatsApp rendering.

#### Node: 📊 Log Support
- **Type / role:** Google Sheets append-or-update.
- **Key configuration choices:** Status `Support Ticket`, matching on `From`.
- **Connections:** → **📱 Send Support Reply**
- **Edge cases:** same sheet-column naming strictness.

#### Node: 📱 Send Support Reply
- **Type / role:** WhatsApp send of the generated `autoReply`.
- **Key configuration choices:**
  - Text body: `={{ $('💬 Support Reply').item.json.autoReply }}`
  - Recipient: Extract Classification `from`
  - `phoneNumberId` placeholder must be updated.
- **Edge cases:** WhatsApp window/policy; credentials.

---

### Block 1.8 — Qualification Path
**Overview:** If not enough info, asks a follow-up question. Uses AI-generated `nextQuestion` when available; otherwise uses heuristic defaults based on message content.

**Nodes involved**
- 🧠 Smart Qualification Reply
- 📱 Send Qualifying Question

#### Node: 🧠 Smart Qualification Reply
- **Type / role:** Code — chooses `nextQuestion` and falls back to smart templates.
- **Key configuration choices:**
  - If `nextQuestion` missing:
    - Greeting detection → a multi-option prompt (automation/pricing/support)
    - Pricing keywords → asks industry + team size (note: this is multiple questions in one message; the system prompt asked for ONE question, but this is only a fallback path)
    - Help keywords → asks for more context
    - Default → generic “tell me more”
  - Outputs:
    - `autoReply`
    - `shouldReply: true`
    - `requiresFollowUp: true`
- **Connections:** Switch → this node → **📱 Send Qualifying Question**
- **Edge cases:**
  - The greeting fallback message asks multiple questions/options; may reduce response clarity but can be effective.
  - Classification memory depends on correct `Simple Memory.sessionKey`.

#### Node: 📱 Send Qualifying Question
- **Type / role:** WhatsApp send.
- **Key configuration choices:**
  - Text body: `={{ $('🧠 Smart Qualification Reply').item.json.autoReply }}`
  - Recipient: Extract Classification `from`
  - `phoneNumberId` placeholder must be updated.
- **Edge cases:** WhatsApp messaging policy/time window; credentials.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| WhatsApp Trigger | n8n-nodes-base.whatsAppTrigger | Entry point for incoming WhatsApp messages | — | 🔧 Parse WhatsApp Message | ## 📥 Incoming Message; WhatsApp Trigger receives and parses incoming messages |
| 🔧 Parse WhatsApp Message | n8n-nodes-base.code | Normalize payload + loop prevention | WhatsApp Trigger | 🤖 AI Intent Classifier | ## 📥 Incoming Message; WhatsApp Trigger receives and parses incoming messages / ## ⚠️ Update Bot Number; Change the `BOT_NUMBERS` array in the Parse node to match your WhatsApp Business phone number ID. This prevents infinite loops from the bot replying to itself. |
| 🤖 AI Intent Classifier | @n8n/n8n-nodes-langchain.agent | LLM-based intent classification | 🔧 Parse WhatsApp Message (+ AI model + memory) | 🎯 Extract Classification | ## 🤖 AI Classification + Extraction; AI classifies intent, then robust parser extracts the result |
| 🎯 Extract Classification | n8n-nodes-base.code | Parse/validate AI output into stable schema | 🤖 AI Intent Classifier | 🔀 Route by Intent | ## 🤖 AI Classification + Extraction; AI classifies intent, then robust parser extracts the result |
| 🔀 Route by Intent | n8n-nodes-base.switch | Route flow by intent | 🎯 Extract Classification | 💬 Hot Lead Reply; 💬 Warm Lead Reply; 💬 Support Reply; 🧠 Smart Qualification Reply | ## 🔀 Intent Routing; Routes to the correct path based on AI classification |
| 💬 Hot Lead Reply | n8n-nodes-base.code | Build hot lead message | 🔀 Route by Intent | 📊 Log Hot Lead | ## 🔥 Hot Lead Path; Reply → Log to Sheets → Book meeting → Send Google Meet invite |
| 📊 Log Hot Lead | n8n-nodes-base.googleSheets | Log/update hot lead in Sheets | 💬 Hot Lead Reply | 📅 Book Meeting (Hot) | ## 🔥 Hot Lead Path; Reply → Log to Sheets → Book meeting → Send Google Meet invite |
| 📅 Book Meeting (Hot) | n8n-nodes-base.googleCalendar | Create calendar event + Meet link | 📊 Log Hot Lead | 📱 Send Meeting (Hot) | ## 🔥 Hot Lead Path; Reply → Log to Sheets → Book meeting → Send Google Meet invite |
| 📱 Send Meeting (Hot) | n8n-nodes-base.whatsApp | Send Meet link to hot lead | 📅 Book Meeting (Hot) | — | ## 🔥 Hot Lead Path; Reply → Log to Sheets → Book meeting → Send Google Meet invite |
| 💬 Warm Lead Reply | n8n-nodes-base.code | Build warm lead message | 🔀 Route by Intent | 📊 Log Warm Lead | ## 🌡️ Warm Lead Path; Reply → Log to Sheets → Book meeting → Send Google Meet invite |
| 📊 Log Warm Lead | n8n-nodes-base.googleSheets | Log/update warm lead in Sheets | 💬 Warm Lead Reply | 📅 Book Meeting (Warm) | ## 🌡️ Warm Lead Path; Reply → Log to Sheets → Book meeting → Send Google Meet invite |
| 📅 Book Meeting (Warm) | n8n-nodes-base.googleCalendar | Create calendar event + Meet link | 📊 Log Warm Lead | 📱 Send Meeting (Warm) | ## 🌡️ Warm Lead Path; Reply → Log to Sheets → Book meeting → Send Google Meet invite |
| 📱 Send Meeting (Warm) | n8n-nodes-base.whatsApp | Send Meet link to warm lead | 📅 Book Meeting (Warm) | — | ## 🌡️ Warm Lead Path; Reply → Log to Sheets → Book meeting → Send Google Meet invite |
| 💬 Support Reply | n8n-nodes-base.code | Build support acknowledgement (ticket) | 🔀 Route by Intent | 📊 Log Support | ## 🛠️ Support Path; Reply → Log to Sheets → Send ticket confirmation via WhatsApp |
| 📊 Log Support | n8n-nodes-base.googleSheets | Log/update support request in Sheets | 💬 Support Reply | 📱 Send Support Reply | ## 🛠️ Support Path; Reply → Log to Sheets → Send ticket confirmation via WhatsApp |
| 📱 Send Support Reply | n8n-nodes-base.whatsApp | Send support confirmation via WhatsApp | 📊 Log Support | — | ## 🛠️ Support Path; Reply → Log to Sheets → Send ticket confirmation via WhatsApp |
| 🧠 Smart Qualification Reply | n8n-nodes-base.code | Choose AI follow-up question or fallback text | 🔀 Route by Intent | 📱 Send Qualifying Question | ## ❓ Qualification Path; Smart follow-up question via WhatsApp. Memory ensures re-classification on reply. |
| 📱 Send Qualifying Question | n8n-nodes-base.whatsApp | Send qualification question via WhatsApp | 🧠 Smart Qualification Reply | — | ## ❓ Qualification Path; Smart follow-up question via WhatsApp. Memory ensures re-classification on reply. |
| Simple Memory | @n8n/n8n-nodes-langchain.memoryBufferWindow | Store last 20 turns per user | (connected to agent) | 🤖 AI Intent Classifier (ai_memory) |  |
| Ollama Chat Model | @n8n/n8n-nodes-langchain.lmChatOllama | LLM backend for agent | (connected to agent) | 🤖 AI Intent Classifier (ai_languageModel) |  |
| Sticky Note - Main Overview | n8n-nodes-base.stickyNote | Canvas documentation | — | — | ## Auto-respond and classify WhatsApp leads with AI and log to Google Sheets; Automatically classify incoming WhatsApp messages using AI, route them by intent, log every interaction to Google Sheets, book Google Calendar meetings, and send smart auto-replies — all without lifting a finger. |
| Sticky Note - Incoming | n8n-nodes-base.stickyNote | Canvas documentation | — | — | ## 📥 Incoming Message; WhatsApp Trigger receives and parses incoming messages |
| Sticky Note - AI | n8n-nodes-base.stickyNote | Canvas documentation | — | — | ## 🤖 AI Classification + Extraction; AI classifies intent, then robust parser extracts the result |
| Sticky Note - Router | n8n-nodes-base.stickyNote | Canvas documentation | — | — | ## 🔀 Intent Routing; Routes to the correct path based on AI classification |
| Sticky Note - Hot | n8n-nodes-base.stickyNote | Canvas documentation | — | — | ## 🔥 Hot Lead Path; Reply → Log to Sheets → Book meeting → Send Google Meet invite |
| Sticky Note - Warm | n8n-nodes-base.stickyNote | Canvas documentation | — | — | ## 🌡️ Warm Lead Path; Reply → Log to Sheets → Book meeting → Send Google Meet invite |
| Sticky Note - Support | n8n-nodes-base.stickyNote | Canvas documentation | — | — | ## 🛠️ Support Path; Reply → Log to Sheets → Send ticket confirmation via WhatsApp |
| Sticky Note - Qualify | n8n-nodes-base.stickyNote | Canvas documentation | — | — | ## ❓ Qualification Path; Smart follow-up question via WhatsApp. Memory ensures re-classification on reply. |
| Sticky Note - Warning | n8n-nodes-base.stickyNote | Canvas documentation | — | — | ## ⚠️ Update Bot Number; Change the `BOT_NUMBERS` array in the Parse node to match your WhatsApp Business phone number ID. This prevents infinite loops from the bot replying to itself. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow** in n8n  
   - Name it: *Auto-respond and classify WhatsApp leads with AI and log to Google Sheets* (or your preferred title).

2) **Add “WhatsApp Trigger”** (`WhatsApp Trigger` node)
   - Updates: select **messages**
   - Configure Meta WhatsApp Cloud API credentials in n8n (access token / app setup as required by your n8n environment).
   - Copy the webhook URL from n8n and configure it in Meta Developer portal for WhatsApp.

3) **Add “Code” node** named **🔧 Parse WhatsApp Message**
   - Paste logic that:
     - Reads `messages[0]` and `contacts[0]`
     - Skips status updates
     - Skips bot’s own messages using a `BOT_NUMBERS` allowlist
   - **Update**:
     - `BOT_NUMBERS` to include your real `phone_number_id` and display phone number (digits-only comparison is used).
   - Connect: **WhatsApp Trigger → 🔧 Parse WhatsApp Message**

4) **Add Ollama model node** named **Ollama Chat Model**
   - Node type: **Ollama Chat Model** (`lmChatOllama`)
   - Model: set to your Ollama model (example in workflow: `minimax-m2:cloud`)
   - Configure Ollama connection/credentials if needed by your setup (base URL, auth if applicable).

5) **Add memory node** named **Simple Memory**
   - Node type: **Simple Memory / Buffer Window**
   - Session ID type: **Custom Key**
   - Session Key: **`={{ $json.waId }}`** (recommended; avoid the extra `=` so per-user memory works)
   - Context Window Length: **20**

6) **Add LangChain Agent node** named **🤖 AI Intent Classifier**
   - Node type: **AI Agent**
   - Prompt type: **Define**
   - User message text:
     - `Classify this WhatsApp message...` including:
       - `{{ $json.name }}`
       - `{{ $json.from }}`
       - `{{ $json.messageBody }}`
   - System message:
     - Define the four intents (`hot_lead`, `warm_lead`, `support`, `needs_qualification`)
     - Force raw JSON output with keys: `intent`, `confidence`, `reason`, `hasEnoughInfo`, `missingInfo`, `nextQuestion`
   - Enable output parsing if available.
   - Connect model and memory:
     - **Ollama Chat Model → 🤖 AI Intent Classifier** (AI Language Model connection)
     - **Simple Memory → 🤖 AI Intent Classifier** (AI Memory connection)
   - Connect main flow:
     - **🔧 Parse WhatsApp Message → 🤖 AI Intent Classifier**

7) **Add “Code” node** named **🎯 Extract Classification**
   - Implement robust parsing to:
     - Locate JSON in nested agent output
     - Validate allowed intents
     - Default to `needs_qualification` on parsing failure
   - Ensure final output includes:
     - Message metadata (`from`, `name`, `messageBody`, `messageId`, timestamps)
     - Classification (`intent`, `confidence`, `reason`, `missingInfo`, `nextQuestion`)
   - Connect: **🤖 AI Intent Classifier → 🎯 Extract Classification**

8) **Add “Switch” node** named **🔀 Route by Intent**
   - Switch on: `={{ $json.intent }}`
   - Create rules:
     - equals `hot_lead`
     - equals `warm_lead`
     - equals `support`
     - equals `needs_qualification`
     - (optional) equals `spam`
   - Connect: **🎯 Extract Classification → 🔀 Route by Intent**

9) **Hot path nodes**
   1. Add Code node **💬 Hot Lead Reply** (generates `autoReply`)
   2. Add Google Sheets node **📊 Log Hot Lead**
      - Credential: Google OAuth2
      - Operation: **Append or Update**
      - Document ID: your spreadsheet ID
      - Sheet tab: select the correct sheet (gid=0 / “Sheet1”)
      - Matching column: `From`
      - Map columns exactly to your sheet headers (including spelling).
   3. Add Google Calendar node **📅 Book Meeting (Hot)**
      - Credential: Google OAuth2
      - Calendar: choose the target calendar
      - Start: `={{ $now.plus(2, 'hours').toISO() }}`
      - End: `={{ $now.plus(3, 'hours').toISO() }}`
      - Request conference/Meet link (as supported by the node/version)
   4. Add WhatsApp node **📱 Send Meeting (Hot)**
      - Operation: Send
      - `phoneNumberId`: your Meta phone number ID
      - Recipient: `={{ $('🎯 Extract Classification').item.json.from }}`
      - Text: include `{{ $json.hangoutLink }}` (from Calendar output)
   5. Wire: **Switch(hot) → 💬 Hot Lead Reply → 📊 Log Hot Lead → 📅 Book Meeting (Hot) → 📱 Send Meeting (Hot)**

10) **Warm path nodes**
   - Repeat similarly:
     - **💬 Warm Lead Reply → 📊 Log Warm Lead → 📅 Book Meeting (Warm) → 📱 Send Meeting (Warm)**
   - Warm meeting times:
     - Start `now + 4 hours`
     - End `now + 5 hours`

11) **Support path nodes**
   1. Code node **💬 Support Reply**
   2. Google Sheets node **📊 Log Support** (Append or Update, match `From`)
   3. WhatsApp node **📱 Send Support Reply**
      - Text: `={{ $('💬 Support Reply').item.json.autoReply }}`
      - Recipient: `={{ $('🎯 Extract Classification').item.json.from }}`
      - `phoneNumberId`: your Meta phone number ID
   4. Wire: **Switch(support) → 💬 Support Reply → 📊 Log Support → 📱 Send Support Reply**

12) **Qualification path nodes**
   1. Code node **🧠 Smart Qualification Reply**
      - Prefer `$json.nextQuestion`
      - Fallback templates for greetings/pricing/help
   2. WhatsApp node **📱 Send Qualifying Question**
      - Text: `={{ $('🧠 Smart Qualification Reply').item.json.autoReply }}`
      - Recipient: `={{ $('🎯 Extract Classification').item.json.from }}`
      - `phoneNumberId`: your Meta phone number ID
   3. Wire: **Switch(needs_qualification) → 🧠 Smart Qualification Reply → 📱 Send Qualifying Question**

13) **(Optional but recommended) Add a guard for `skipProcessing`**
   - Insert an IF node after **🔧 Parse WhatsApp Message**:
     - If `skipProcessing` is true → end
     - Else → continue to classifier
   - This prevents AI calls on status/outgoing events.

14) **Validate external placeholders**
   - Replace:
     - `YOUR_PHONE_NUMBER_ID`
     - `YOUR_DISPLAY_PHONE_NUMBER`
     - `YOUR_GOOGLE_SHEET_ID`
     - Calendar email `user@example.com`
     - Website links in warm reply
   - Ensure your Google Sheet has headers matching the mapped fields (including `recived at` if you keep that spelling).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Automatically classify incoming WhatsApp messages using AI, route them by intent, log every interaction to Google Sheets, book Google Calendar meetings, and send smart auto-replies — all without lifting a finger. | Sticky Note - Main Overview |
| Setup includes connecting WhatsApp Business API, Google Sheets OAuth, Google Calendar OAuth, configuring LLM endpoint, updating `BOT_NUMBERS` and WhatsApp `phoneNumberId`. | Sticky Note - Main Overview |
| Change the `BOT_NUMBERS` array in the Parse node to match your WhatsApp Business phone number ID to prevent infinite loops. | Sticky Note - Warning |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.