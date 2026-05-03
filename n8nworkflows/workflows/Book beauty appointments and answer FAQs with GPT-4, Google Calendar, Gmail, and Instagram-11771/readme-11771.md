Book beauty appointments and answer FAQs with GPT-4, Google Calendar, Gmail, and Instagram

https://n8nworkflows.xyz/workflows/book-beauty-appointments-and-answer-faqs-with-gpt-4--google-calendar--gmail--and-instagram-11771


# Book beauty appointments and answer FAQs with GPT-4, Google Calendar, Gmail, and Instagram

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
An Instagram DM assistant for a beauty business that can (a) answer customer FAQs using a Google Docs document as the only source of truth and (b) guide appointment booking by checking Google Calendar availability, creating a reservation, and sending a confirmation email via Gmail. It supports **text and voice messages** (voice is transcribed with OpenAI first), then routes everything through a **LangChain Agent** powered by an OpenAI chat model and memory.

**Target use cases:**
- “What are your prices / services / address / hours?” → answered strictly from the FAQ Google Doc.
- “I want to book lashes tomorrow at 3pm” → availability check → calendar event creation → confirmation email → Instagram reply.

### Logical blocks
1. **1.1 Input Reception & Filtering (Instagram webhook + echo protection)**  
2. **1.2 Voice Handling (audio detection → download → transcription)**  
3. **1.3 Message Normalization (unify text & transcribed voice to `userMessage`)**  
4. **1.4 AI Orchestration (agent + memory + tools: Docs/Calendar/DateTime/Gmail)**  
5. **1.5 Outbound Reply (send message back to Instagram Graph API)**

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception & Filtering
**Overview:** Receives Instagram messages via webhook, then attempts to ignore echo/system messages so the bot does not reply to itself.

**Nodes involved:**
- Instagram Webhook Trigger
- Extract Sender & Text
- Check is_ecco

#### Node: Instagram Webhook Trigger
- **Type / role:** `Webhook` trigger; entry point for Instagram events.
- **Key configuration:**
  - Method: **POST**
  - Path: `instagram-message`
- **Inputs/Outputs:** No inputs; output goes to **Extract Sender & Text**.
- **Failure/edge cases:**
  - Instagram signature validation is not configured here (no verification/security options shown).
  - Payload shape assumptions (`body.entry[0].messaging[0]...`) can break if Instagram changes format or sends other event types.

#### Node: Extract Sender & Text
- **Type / role:** `Set` node; intended to extract sender id and message text into normalized fields.
- **Configuration choices (interpreted):**
  - Currently **no fields are configured** (empty Set node).
  - Intended outputs likely:
    - `sender`: `{{$json.body.entry[0].messaging[0].sender.id}}`
    - `text` (or message text): `{{$json.body.entry[0].messaging[0].message.text}}`
- **Inputs/Outputs:** From webhook; output to **Check is_ecco**.
- **Failure/edge cases:**
  - As configured, it does not actually extract anything; downstream nodes reference `$('Extract Sender & Text').item.json.sender` and may be **undefined**.
  - If the incoming message is voice-only, `message.text` may be missing.

#### Node: Check is_ecco
- **Type / role:** `IF` node; filters out messages that are echoes (`is_echo === true`).
- **Key expression/variables:**
  - Left value: `={{ $('Webhook').item.json.body.entry[0].messaging[0].message.is_echo }}`
  - Condition: boolean **true**
- **Connections:**
  - **True branch**: no outgoing nodes (message is dropped).
  - **False branch**: goes to **Is Voice Message?**
- **Version requirements:** `typeVersion 2.2` uses the newer condition builder.
- **Important issue (likely bug):**
  - It references a node named **`Webhook`**, but the trigger node is named **`Instagram Webhook Trigger`**. This will cause expression resolution errors.
  - Should likely be:  
    `={{ $('Instagram Webhook Trigger').item.json.body.entry[0].messaging[0].message.is_echo }}`
- **Failure/edge cases:**
  - If `message.is_echo` is absent, strict boolean evaluation may fail; you may want a safer expression (default false).

---

### 2.2 Voice Handling
**Overview:** Detects whether the incoming message includes an audio attachment, downloads it, and transcribes it using OpenAI’s audio transcription operation.

**Nodes involved:**
- Is Voice Message?
- Download Voice Audio
- Transcribe Voice Message

#### Node: Is Voice Message?
- **Type / role:** `IF` node; checks attachment type equals `"audio"`.
- **Key expression:**
  - `={{ $json.body.entry[0].messaging[0].message.attachments[0].type }} == "audio"`
- **Connections:**
  - **True branch** → Download Voice Audio
  - **False branch** → Normalize User Message (text path)
- **Failure/edge cases:**
  - If `attachments` is missing, the expression can throw. A safer check would guard for existence.

#### Node: Download Voice Audio
- **Type / role:** `HTTP Request` to fetch the audio file from the attachment URL.
- **Key configuration:**
  - URL: `={{ $json.body.entry[0].messaging[0].message.attachments[0].payload.url }}`
  - Method not specified (defaults to GET).
- **Connections:** Output → Transcribe Voice Message
- **Failure/edge cases:**
  - Instagram-hosted URLs may expire or require headers; request may return 401/403.
  - Response is not explicitly configured as “file/binary”; transcription nodes typically require binary audio input.

#### Node: Transcribe Voice Message
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi` (OpenAI) audio transcription.
- **Key configuration:**
  - Resource: **audio**
  - Operation: **transcribe**
  - Credentials: OpenAI (not shown on this node, but required)
- **Connections:** Output → Normalize User Message
- **Failure/edge cases:**
  - If Download Voice Audio doesn’t provide binary in the format the node expects, transcription fails.
  - Model/format constraints for transcription (file size, encoding) can cause errors.

---

### 2.3 Message Normalization
**Overview:** Ensures both text messages and transcribed voice become a single field `userMessage` for the AI agent.

**Nodes involved:**
- Normalize User Message

#### Node: Normalize User Message
- **Type / role:** `Code` node; maps incoming item(s) to `{ userMessage }`.
- **Key logic:**
  - Uses `item.json.text || item.json.message || ""`
  - Trims and outputs `json.userMessage`
- **Connections:** Output → Instagram AI Agent
- **Failure/edge cases:**
  - If transcription output does not populate `text` or `message` fields, `userMessage` becomes empty.
  - You may need to adapt to actual transcription output schema (often something like `text` or `transcription` depending on node).

---

### 2.4 AI Orchestration (Agent + Tools + Memory)
**Overview:** A LangChain AI Agent receives `userMessage`, uses memory for conversation context, and can call tools to: read FAQs from Google Docs, get current date/time, check calendar availability, create a booking, and send a confirmation email.

**Nodes involved:**
- Instagram AI Agent
- OpenAI Chat Model
- Simple Memory
- FAQs (Google Docs)
- Get Current Date & Time
- Check Date Availability
- Create Calendar Reservation
- Send Confirmation Email

#### Node: Instagram AI Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` (LangChain Agent orchestrator).
- **Key configuration:**
  - Prompt type: `define`
  - User text: `={{ $json.userMessage }}`
  - System message: `"Instagram Beauty Assistant system message (sanitized template)"`
- **Tools connected (AI tool ports):**
  - Google Docs tool (FAQ source)
  - DateTime tool (current time context)
  - Google Calendar getAll (availability)
  - Google Calendar create (reservation)
  - Gmail send (confirmation)
- **Model + memory:**
  - Receives model from **OpenAI Chat Model** via `ai_languageModel`.
  - Receives memory from **Simple Memory** via `ai_memory`.
- **Output:** `main` output → Send Instagram Reply (expects something like `$json.output` to exist).
- **Failure/edge cases:**
  - If system message is too vague, the agent may call tools incorrectly.
  - If `$json.output` is not the actual agent output field, the reply node will send empty text. Verify agent output schema in executions.

#### Node: OpenAI Chat Model
- **Type / role:** `lmChatOpenAi` chat model provider for the agent.
- **Key configuration:**
  - Model: `gpt-4.1-mini`
  - Credentials: OpenAI account
- **Connection:** `ai_languageModel` → Instagram AI Agent
- **Failure/edge cases:**
  - Rate limits, invalid API key, model not available in the account/region.

#### Node: Simple Memory
- **Type / role:** `memoryBufferWindow`; maintains last N turns.
- **Key configuration:**
  - Context window length: `10`
  - SessionIdType: `customKey` (but **no custom key is shown** in parameters)
- **Connection:** `ai_memory` → Instagram AI Agent
- **Failure/edge cases:**
  - Without a stable session key per Instagram sender, conversations may mix across users or reset every message. Typically session key should be the sender id.

#### Node: FAQs (Google Docs)
- **Type / role:** `googleDocsTool` used as an agent tool to fetch FAQ content.
- **Key configuration:**
  - Operation: `get`
  - Document URL: placeholder `https://docs.google.com/document/d/YOUR_DOC_ID/edit`
  - Credentials: Google Docs OAuth2
- **Connection:** `ai_tool` → Instagram AI Agent
- **Failure/edge cases:**
  - Doc permissions (must be accessible to the OAuth user).
  - Large doc size may exceed context; consider extracting only relevant sections or using chunking/search.

#### Node: Get Current Date & Time
- **Type / role:** `dateTimeTool` for the agent to anchor booking dates/times.
- **Connection:** `ai_tool` → Instagram AI Agent
- **Failure/edge cases:**
  - Business timezone depends on n8n instance timezone settings (not per-node here).

#### Node: Check Date Availability
- **Type / role:** `googleCalendarTool` (agent tool) to list events (availability checking).
- **Key configuration:**
  - Operation: `getAll`
  - Calendar: set to “list mode” but **no calendar selected** (empty value)
- **Connection:** `ai_tool` → Instagram AI Agent
- **Failure/edge cases:**
  - Missing calendar selection will fail at runtime.
  - For real availability, you must constrain timeMin/timeMax; not shown here, so the agent must supply or defaults may be unusable.

#### Node: Create Calendar Reservation
- **Type / role:** `googleCalendarTool` (agent tool) to create the booking event.
- **Key configuration:**
  - Calendar: not selected (empty)
  - Additional fields: empty (so title/attendees/time likely must be provided by agent/tool call)
- **Connection:** `ai_tool` → Instagram AI Agent
- **Failure/edge cases:**
  - Missing required event fields (start/end/summary).
  - Timezone mismatches create wrong slot bookings.

#### Node: Send Confirmation Email
- **Type / role:** `gmailTool` (agent tool) to send confirmation emails.
- **Key configuration:** Options empty; agent must provide recipients/subject/body.
- **Connection:** `ai_tool` → Instagram AI Agent
- **Failure/edge cases:**
  - Gmail OAuth scopes/consent issues.
  - Sending to an address not collected from the user (agent must ask for email first).

---

### 2.5 Outbound Reply (Instagram)
**Overview:** Sends the agent’s final text reply back to Instagram using the Graph API.

**Nodes involved:**
- Send Instagram Reply

#### Node: Send Instagram Reply
- **Type / role:** `HTTP Request` (Graph API call) to send a DM reply.
- **Key configuration:**
  - Method: POST
  - URL: `https://graph.instagram.com/v21.0/me/messages`
  - Headers:
    - `Authorization: Bearer YOUR_TOKEN_HERE`
    - `Content-Type: application/json`
  - JSON body expression:
    - Recipient id: `$('Extract Sender & Text').item.json.sender`
    - Message text: `$json.output`
- **Connections:** Receives from Instagram AI Agent; no further outputs.
- **Failure/edge cases:**
  - Token expiration / missing permissions.
  - Wrong endpoint/versioning for Instagram messaging (often requires Graph API host `graph.facebook.com` and Instagram Messaging setup depending on account type).
  - If `sender` was never extracted (Set node empty), recipient id is undefined → API error.
  - If agent output field is not `output`, text may be empty.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Instagram Webhook Trigger | Webhook | Receives incoming Instagram DM events | — | Extract Sender & Text | Preparing incoming messages; How it works / Setup steps |
| Extract Sender & Text | Set | Intended to extract sender + message text | Instagram Webhook Trigger | Check is_ecco | Preparing incoming messages; How it works / Setup steps |
| Check is_ecco | If | Drops echo/system messages | Extract Sender & Text | Is Voice Message? (false path only) | Preparing incoming messages; How it works / Setup steps |
| Is Voice Message? | If | Routes audio vs text messages | Check is_ecco | Download Voice Audio (true), Normalize User Message (false) | Handling Voice messages; How it works / Setup steps |
| Download Voice Audio | HTTP Request | Downloads audio file from attachment URL | Is Voice Message? (true) | Transcribe Voice Message | Handling Voice messages; How it works / Setup steps |
| Transcribe Voice Message | OpenAI (LangChain) | Speech-to-text transcription | Download Voice Audio | Normalize User Message | Handling Voice messages; How it works / Setup steps |
| Normalize User Message | Code | Produces unified `userMessage` field | Is Voice Message? (false) OR Transcribe Voice Message | Instagram AI Agent | Preparing final message; How it works / Setup steps |
| Instagram AI Agent | LangChain Agent | Decides response; calls tools for FAQ/booking | Normalize User Message + AI model/memory/tools | Send Instagram Reply | Main message processing; How it works / Setup steps |
| OpenAI Chat Model | LangChain Chat Model | Provides GPT model to agent | — | Instagram AI Agent (ai_languageModel) | Main message processing; How it works / Setup steps |
| Simple Memory | LangChain Memory | Stores last 10 turns for context | — | Instagram AI Agent (ai_memory) | Main message processing; How it works / Setup steps |
| FAQs (Google Docs) | Google Docs Tool | Tool: fetch FAQ doc content | — | Instagram AI Agent (ai_tool) | Main message processing; How it works / Setup steps |
| Get Current Date & Time | DateTime Tool | Tool: current date/time reference | — | Instagram AI Agent (ai_tool) | Main message processing; How it works / Setup steps |
| Check Date Availability | Google Calendar Tool | Tool: list events to assess availability | — | Instagram AI Agent (ai_tool) | Main message processing; How it works / Setup steps |
| Create Calendar Reservation | Google Calendar Tool | Tool: create booking event | — | Instagram AI Agent (ai_tool) | Main message processing; How it works / Setup steps |
| Send Confirmation Email | Gmail Tool | Tool: email booking confirmation | — | Instagram AI Agent (ai_tool) | Main message processing; How it works / Setup steps |
| Send Instagram Reply | HTTP Request | Sends final reply to Instagram API | Instagram AI Agent | — | Main message processing; How it works / Setup steps |
| Sticky Note | Sticky Note | Comment block | — | — | How it works / Setup steps |
| Sticky Note2 | Sticky Note | Comment block | — | — | Preparing incoming messages |
| Sticky Note3 | Sticky Note | Comment block | — | — | Handling Voice messages |
| Sticky Note4 | Sticky Note | Comment block | — | — | Preparing final message |
| Sticky Note5 | Sticky Note | Comment block | — | — | Main message processing |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n (keep execution order default `v1` unless you know you need `v2`).

2. **Add Trigger: Webhook**
   - Node type: **Webhook**
   - Name: `Instagram Webhook Trigger`
   - HTTP Method: **POST**
   - Path: `instagram-message`
   - (Recommended) Enable and configure request validation/security according to your Instagram webhook approach.

3. **Add Set node to extract sender + text**
   - Node type: **Set**
   - Name: `Extract Sender & Text`
   - Add fields (recommended):
     - `sender` = `={{ $json.body.entry[0].messaging[0].sender.id }}`
     - `text` = `={{ $json.body.entry[0].messaging[0].message.text }}`
     - (Optional) keep original payload in a separate field if needed.
   - Connect: `Instagram Webhook Trigger` → `Extract Sender & Text`

4. **Add IF node to ignore echo messages**
   - Node type: **If**
   - Name: `Check is_ecco`
   - Condition: boolean true on:
     - `={{ $('Instagram Webhook Trigger').item.json.body.entry[0].messaging[0].message.is_echo }}`
   - Connect: `Extract Sender & Text` → `Check is_ecco`
   - Leave **true branch** unconnected (drop), connect **false branch** forward.

5. **Add IF node for voice detection**
   - Node type: **If**
   - Name: `Is Voice Message?`
   - Condition (string equals):
     - Left: `={{ $json.body.entry[0].messaging[0].message.attachments[0].type }}`
     - Right: `audio`
   - Connect: `Check is_ecco` (false output) → `Is Voice Message?`

6. **Voice path: download audio**
   - Node type: **HTTP Request**
   - Name: `Download Voice Audio`
   - Method: GET
   - URL: `={{ $json.body.entry[0].messaging[0].message.attachments[0].payload.url }}`
   - Configure response to be **binary/file** if your transcription node requires binary input.
   - Connect: `Is Voice Message?` (true) → `Download Voice Audio`

7. **Voice path: transcribe**
   - Node type: **OpenAI (LangChain)** audio node
   - Name: `Transcribe Voice Message`
   - Resource: **audio**
   - Operation: **transcribe**
   - Credentials: configure **OpenAI API** credential in n8n and select it here.
   - Connect: `Download Voice Audio` → `Transcribe Voice Message`

8. **Add Code node to normalize message**
   - Node type: **Code**
   - Name: `Normalize User Message`
   - Use logic equivalent to:
     - Read from `text` (text messages) or transcription output field (adjust as needed).
     - Output `{ userMessage }`.
   - Connect:
     - `Is Voice Message?` (false) → `Normalize User Message`
     - `Transcribe Voice Message` → `Normalize User Message`

9. **Add AI components**
   1) **Chat model**
   - Node type: `OpenAI Chat Model (LangChain)`
   - Name: `OpenAI Chat Model`
   - Model: `gpt-4.1-mini` (or your preferred)
   - Credentials: OpenAI API credential

   2) **Memory**
   - Node type: `Simple Memory` (buffer window)
   - Name: `Simple Memory`
   - Context window length: `10`
   - Session key: set it to a stable identifier (recommended: Instagram sender id). If the node requires a specific field, map it from `sender`.

   3) **Agent**
   - Node type: `AI Agent (LangChain)`
   - Name: `Instagram AI Agent`
   - Prompt type: `define`
   - Text input: `={{ $json.userMessage }}`
   - System message: define strict rules, e.g.:
     - Answer only from FAQ doc.
     - For bookings: collect required info (service, date/time, email), check availability, create event, then email.
     - If not found in FAQ: say you don’t know / ask to contact.
   - Connect:
     - `Normalize User Message` → `Instagram AI Agent` (main)
     - `OpenAI Chat Model` → `Instagram AI Agent` (ai_languageModel)
     - `Simple Memory` → `Instagram AI Agent` (ai_memory)

10. **Add agent tools**
    1) **Google Docs tool**
    - Node type: `Google Docs Tool`
    - Name: `FAQs (Google Docs)`
    - Operation: `get`
    - Document URL: your FAQ doc URL
    - Credentials: **Google Docs OAuth2**
    - Connect tool output: `FAQs (Google Docs)` → `Instagram AI Agent` (ai_tool)

    2) **Date/Time tool**
    - Node type: `Date & Time Tool`
    - Name: `Get Current Date & Time`
    - Connect: → `Instagram AI Agent` (ai_tool)

    3) **Google Calendar tool: availability**
    - Node type: `Google Calendar Tool`
    - Name: `Check Date Availability`
    - Operation: `getAll`
    - Select calendar (required)
    - Ensure the tool can accept time boundaries (timeMin/timeMax) via agent parameters or preset defaults.
    - Connect: → `Instagram AI Agent` (ai_tool)

    4) **Google Calendar tool: create event**
    - Node type: `Google Calendar Tool`
    - Name: `Create Calendar Reservation`
    - Operation: create (configure as needed)
    - Select calendar (required)
    - Map required fields: summary/title, start, end, timezone, description, etc. Either fixed defaults or agent-supplied.
    - Connect: → `Instagram AI Agent` (ai_tool)

    5) **Gmail tool**
    - Node type: `Gmail Tool`
    - Name: `Send Confirmation Email`
    - Credentials: **Gmail OAuth2**
    - Ensure scopes allow sending mail.
    - Connect: → `Instagram AI Agent` (ai_tool)

11. **Add HTTP Request to reply on Instagram**
    - Node type: **HTTP Request**
    - Name: `Send Instagram Reply`
    - Method: POST
    - URL: `https://graph.instagram.com/v21.0/me/messages`
    - Headers:
      - `Authorization: Bearer <YOUR_TOKEN>`
      - `Content-Type: application/json`
    - Body (JSON):
      - recipient id from your extracted `sender`
      - message text from the agent output field (verify actual field; update expression accordingly)
    - Connect: `Instagram AI Agent` → `Send Instagram Reply`

12. **Credentials & platform setup checklist**
    - OpenAI API key configured in n8n credentials.
    - Google OAuth2 credentials for **Docs** and **Calendar** (with correct scopes).
    - Gmail OAuth2 credentials (send mail scope).
    - Instagram messaging setup:
      - Webhook subscription to your endpoint.
      - Proper access token with messaging permissions.
      - Confirm correct Graph endpoint for your account type and messaging product.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “This workflow creates an AI assistant for Instagram direct messages… If information is not found in the FAQ document, the agent does not guess or generate answers.” | Sticky note “How it works” (workflow intent/behavior rules) |
| Setup steps: connect Instagram webhook + token; OpenAI credentials; replace Google Docs URL; connect Calendar + Gmail; set business timezone; test with text/voice | Sticky note “How it works” (operational setup guidance) |
| Preparing incoming messages: extract message; ensure not an echo/system response | Sticky note “Preparing incoming messages” |
| Handling Voice messages: filter voice, download audio, transcribe to text | Sticky note “Handling Voice messages” |
| Preparing final message: normalize text + voice into same format before AI | Sticky note “Preparing final message” |
| Main message processing: FAQ answers + booking flow via Calendar + email | Sticky note “Main message processing” |