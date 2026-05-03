Automate WhatsApp lead capture and replies with Whapi, Ollama and Sheets

https://n8nworkflows.xyz/workflows/automate-whatsapp-lead-capture-and-replies-with-whapi--ollama-and-sheets-15044


# Automate WhatsApp lead capture and replies with Whapi, Ollama and Sheets

---

## 1. Workflow Overview

**WhatsApp AI CRM — Whapi + Ollama** is a fully self-contained n8n workflow that turns inbound WhatsApp messages into AI-powered conversations, persistent conversation logs, and automatic lead capture — all using a **locally-hosted AI model** (Ollama) so no data leaves your infrastructure and no cloud LLM costs are incurred.

**Purpose & Target Use Cases:**  
Instant, 24/7 AI replies to WhatsApp leads for real estate agencies, service businesses, consultants, appointment-based companies, and customer-support teams. The workflow also builds a lightweight CRM inside Google Sheets.

**Logical Blocks:**

| Block | Name | Nodes | Purpose |
|-------|------|-------|---------|
| 1.1 | Input Reception | Receive WhatsApp Message → Extract Message Data → Is Text From Customer? | Receive the Whapi webhook, extract sender/message fields, and gate out non-text or outbound messages |
| 1.2 | Context Retrieval | Read Conversation History | Pull all previous messages for this phone number from the History sheet to give the AI memory |
| 1.3 | AI Processing | Build AI Prompt → Generate AI Reply ← Ollama Model | Assemble a full prompt (system instructions + history + new message) and generate a contextual reply with a local Ollama model |
| 1.4 | Logging & CRM | Log User Message → Log AI Reply → Upsert Lead in CRM | Append both sides of the conversation to the History sheet and upsert the lead record in the Leads sheet |
| 1.5 | Response Delivery | Send Reply via Whapi | POST the AI reply back to the customer through the Whapi.cloud API |

---

## 2. Block-by-Block Analysis

---

### Block 1.1 — Input Reception

**Overview:** This block is the entry point of the workflow. A Webhook node listens for POST requests from Whapi.cloud containing inbound WhatsApp events. A Code node normalises and extracts the essential fields, and an IF node filters so that only genuine inbound text messages from customers proceed.

**Nodes Involved:**
- Receive WhatsApp Message
- Extract Message Data
- Is Text From Customer?

---

#### Node: Receive WhatsApp Message

| Property | Value |
|----------|-------|
| **Type** | Webhook (`n8n-nodes-base.webhook`) |
| **Technical Role** | HTTP endpoint that receives Whapi.cloud event payloads via POST |
| **HTTP Method** | POST |
| **Path** | `whapi-crm` (produces URL: `<n8n-base-url>/webhook/whapi-crm`) |
| **Authentication** | None (Whapi does not sign webhooks by default; consider adding header-based validation if needed) |
| **Response Mode** | Default (responds immediately with 200) |
| **Version** | 2 |
| **Input** | None (entry node) |
| **Output** | Extract Message Data |

**Edge Cases & Failure Types:**
- If Whapi sends an empty body or non-JSON payload, downstream extraction will fall back to defaults.
- If the webhook URL is incorrect or the workflow is inactive, Whapi will receive connection errors or timeouts.
- No authentication on the webhook means any POST to the URL is accepted — a potential abuse vector in production.

---

#### Node: Extract Message Data

| Property | Value |
|----------|-------|
| **Type** | Code (`n8n-nodes-base.code`) |
| **Technical Role** | Parses the Whapi payload, normalises the phone number, and outputs a structured object |
| **Language** | JavaScript |
| **Version** | 2 |

**Key Logic:**
1. Reads `body` from the incoming JSON (supports both `body` wrapper and direct payload).
2. Looks for `body.messages` array; if empty, returns a `skip: true` sentinel object.
3. Takes the first message in the array.
4. Strips `@s.whatsapp.net` and `@c.us` suffixes from the chat ID to produce a clean phone number.
5. Extracts: `phone`, `name` (from `pushName` or `senderName`, defaulting to "Unknown"), `messageText`, `fromMe` (boolean), `messageType`, `messageId`, and a current ISO `timestamp`.

**Output Structure:**
```
{ skip, phone, name, messageText, fromMe, messageType, messageId, timestamp }
```

**Input** | Receive WhatsApp Message  
**Output** | Is Text From Customer?

**Edge Cases & Failure Types:**
- If the payload structure changes (e.g., Whapi changes field names), extraction silently defaults to empty/unknown values.
- Group messages may include additional metadata not captured here.
- Media messages will have `messageType` other than "text" and empty `messageText`, which the IF node will filter out.

---

#### Node: Is Text From Customer?

| Property | Value |
|----------|-------|
| **Type** | IF (`n8n-nodes-base.if`) |
| **Technical Role** | Gate node that only passes through genuine inbound text messages |
| **Version** | 2 |
| **Combinator** | AND (all conditions must be true) |

**Conditions:**

| # | Left Value | Operator | Right Value | Purpose |
|---|-----------|----------|-------------|---------|
| 1 | `{{ $json.fromMe }}` | boolean equals | `false` | Exclude messages sent by the business |
| 2 | `{{ $json.messageType }}` | string equals | `text` | Exclude media, stickers, voice, etc. |
| 3 | `{{ $json.skip }}` | boolean equals | `false` | Exclude empty/invalid payloads from the Code node |

**Input** | Extract Message Data  
**Output (true branch)** | Read Conversation History  
**Output (false branch)** | Workflow ends (no further processing)

**Edge Cases & Failure Types:**
- Messages with `fromMe: undefined` will not match `false` — they will be filtered out (safe default).
- Any future Whapi event types (e.g., status updates) that lack `fromMe` will also be filtered out.
- The false branch has no connected node, so filtered-out items simply terminate.

---

### Block 1.2 — Context Retrieval

**Overview:** Before the AI generates a reply, this block fetches the full conversation history for the current phone number from the History tab in Google Sheets. This gives the model memory of past interactions so replies are contextual rather than stateless.

**Nodes Involved:**
- Read Conversation History

---

#### Node: Read Conversation History

| Property | Value |
|----------|-------|
| **Type** | Google Sheets (`n8n-nodes-base.googleSheets`) |
| **Technical Role** | Reads rows from the History tab filtered by the sender's phone number |
| **Version** | 4.5 |
| **Operation** | Read |
| **Document ID** | `YOUR_GOOGLE_SHEET_ID` (must be replaced) |
| **Sheet Name** | `History` |
| **Filter** | Column `Phone` equals `{{ $('Is Text From Customer?').first().json.phone }}` |
| **Always Output Data** | Enabled (returns empty array if no rows match — critical for first-time contacts) |
| **Credential** | Google Sheets OAuth2 (must be configured in n8n) |

**Input** | Is Text From Customer? (true branch)  
**Output** | Build AI Prompt

**Edge Cases & Failure Types:**
- If the Sheet ID is invalid or the credential lacks access, the node throws a permissions error and the workflow halts.
- For first-time contacts, the node returns zero rows; `alwaysOutputData: true` ensures the empty array still flows forward.
- If the History tab contains malformed rows (missing Role or Message columns), the Build AI Prompt node gracefully handles missing fields by skipping them.

---

### Block 1.3 — AI Processing

**Overview:** This block assembles the full prompt — system instructions, conversation history, and the new message — then invokes a local Ollama model via the LangChain LLM Chain node to generate a contextual reply.

**Nodes Involved:**
- Build AI Prompt
- Generate AI Reply
- Ollama Model

---

#### Node: Build AI Prompt

| Property | Value |
|----------|-------|
| **Type** | Code (`n8n-nodes-base.code`) |
| **Technical Role** | Constructs the full text prompt for the AI model |
| **Language** | JavaScript |
| **Version** | 2 |

**Key Logic:**

1. Retrieves message data from the `Is Text From Customer?` node (phone, name, messageText).
2. Iterates over all incoming rows from Read Conversation History; for each row with a `Role` and `Message`, formats it as `Name: Message` (using the contact's name for "user" role and "Assistant" for "assistant" role).
3. Defines a **system prompt** (default: real estate assistant persona, 2–4 sentence responses, same-language replies, includes customer name and phone).
4. Concatenates: system prompt → `--- Conversation History ---` → history text (or "(This is the first message)") → separator → `CustomerName: newMessage` → `Assistant:`.

**Output Structure (added to existing msgData fields):**
```
{ ...msgData, prompt: fullPrompt }
```

**Customisation Point:** The `systemPrompt` constant is the primary place to change the AI's persona, industry focus, response length, and behaviour.

**Input** | Read Conversation History  
**Output** | Generate AI Reply

**Edge Cases & Failure Types:**
- If conversation history contains very long transcripts, the prompt may exceed the model's context window, causing truncation or Ollama errors.
- The system prompt uses template literals with `msgData.name` and `msgData.phone` — if those are empty, the prompt will contain empty placeholders.
- No explicit token counting is performed; users with large histories should consider limiting rows read.

---

#### Node: Generate AI Reply

| Property | Value |
|----------|-------|
| **Type** | LangChain LLM Chain (`@n8n/n8n-nodes-langchain.chainLlm`) |
| **Technical Role** | Invokes the connected LLM (Ollama) with the constructed prompt and returns the AI's text response |
| **Version** | 1.4 |
| **Prompt Type** | Define (text prompt passed directly) |
| **Text Parameter** | `{{ $json.prompt }}` (the full prompt from Build AI Prompt) |

**Sub-node Connection:** Receives its language model from the **Ollama Model** node via the `ai_languageModel` input.

**Input (main)** | Build AI Prompt  
**Input (ai_languageModel)** | Ollama Model  
**Output** | Log User Message

**Edge Cases & Failure Types:**
- If Ollama is unreachable (server down, wrong URL), the node will timeout or throw a connection error.
- If the model name is not pulled/available in Ollama, the node will throw a "model not found" error.
- If the prompt is empty or malformed, the model may return empty or nonsensical output.
- The output is available as `{{ $json.text }}` (the `text` field of the chain output).

---

#### Node: Ollama Model

| Property | Value |
|----------|-------|
| **Type** | LangChain Ollama Chat Model (`@n8n/n8n-nodes-langchain.lmChatOllama`) |
| **Technical Role** | Configures the local Ollama LLM that powers the chain |
| **Version** | 1 |
| **Model** | `gemma3:1b` |
| **Options** | Default (no temperature, top-p, etc. overrides) |
| **Credential** | Ollama API (requires base URL of the Ollama server, e.g., `http://localhost:11434`) |

**Connection:** Connected to **Generate AI Reply** via the `ai_languageModel` input (not a main flow connection).

**Input** | None (sub-node, connected to Generate AI Reply)  
**Output** | Generate AI Reply (ai_languageModel)

**Edge Cases & Failure Types:**
- Default model `gemma3:1b` must be pulled with `ollama pull gemma3:1b` before the workflow runs.
- If the Ollama server is on a remote host, the base URL credential must point to the correct host and port.
- No model options are set — defaults from Ollama apply (temperature ≈ 0.8, etc.). For deterministic responses, configure temperature in the options.
- Changing the model name to an unavailable model will cause runtime errors.

---

### Block 1.4 — Logging & CRM

**Overview:** After the AI generates a reply, this block logs both the user's message and the AI's reply to the History sheet for conversation continuity, then upserts the lead record in the Leads sheet to maintain a simple CRM.

**Nodes Involved:**
- Log User Message
- Log AI Reply
- Upsert Lead in CRM

---

#### Node: Log User Message

| Property | Value |
|----------|-------|
| **Type** | Google Sheets (`n8n-nodes-base.googleSheets`) |
| **Technical Role** | Appends the customer's inbound message to the History tab |
| **Version** | 4.5 |
| **Operation** | Append |
| **Document ID** | `YOUR_GOOGLE_SHEET_ID` |
| **Sheet Name** | `History` |
| **Mapping Mode** | Auto Map Input Data |
| **Column Schema** | Phone (string), Name (string), Role (string), Message (string), Timestamp (string) |

Because the mapping mode is `autoMapInputData`, the node will automatically map any incoming JSON fields that match the column names. The incoming data from Generate AI Reply still carries the fields from Build AI Prompt (`phone`, `name`, `messageText`, `timestamp`) plus the AI's `text` field.

**Note:** The `Role` field and proper `Message` mapping for the user message rely on the incoming data containing a `Role` field set to `user` and a `Message` field. Since Build AI Prompt does not explicitly set `Role: "user"` or rename `messageText` to `Message`, the auto-map may not populate `Role` and `Message` correctly for this row. **Users should verify the output** and may need to switch to `defineBelow` mode with explicit mappings:
- `Phone` = `{{ $('Build AI Prompt').first().json.phone }}`
- `Name` = `{{ $('Build AI Prompt').first().json.name }}`
- `Role` = `user`
- `Message` = `{{ $('Build AI Prompt').first().json.messageText }}`
- `Timestamp` = `{{ $('Build AI Prompt').first().json.timestamp }}`

**Input** | Generate AI Reply  
**Output** | Log AI Reply

**Edge Cases & Failure Types:**
- If auto-mapping fails to match column names, the row will be appended with empty cells for unmatched columns.
- If the Google Sheet quota is exceeded (API limits), append operations may fail.
- Concurrent messages from the same number could cause race conditions in reading/writing, but Google Sheets API handles row-level appends sequentially.

---

#### Node: Log AI Reply

| Property | Value |
|----------|-------|
| **Type** | Google Sheets (`n8n-nodes-base.googleSheets`) |
| **Technical Role** | Appends the AI's reply to the History tab |
| **Version** | 4.5 |
| **Operation** | Append |
| **Document ID** | `YOUR_GOOGLE_SHEET_ID` |
| **Sheet Name** | `History` |
| **Mapping Mode** | Define Below (explicit) |

**Explicit Column Mappings:**

| Column | Expression |
|--------|-----------|
| Phone | `{{ $('Build AI Prompt').first().json.phone }}` |
| Name | `{{ $('Build AI Prompt').first().json.name }}` |
| Role | `assistant` (hard-coded) |
| Message | `{{ $('Generate AI Reply').first().json.text }}` |
| Timestamp | `{{ $now.toISO() }}` |

**Input** | Log User Message  
**Output** | Upsert Lead in CRM

**Edge Cases & Failure Types:**
- If `Generate AI Reply` outputs an empty `text` field, the Message column will be empty.
- Uses `$now.toISO()` for the timestamp (time of logging, not original message time).

---

#### Node: Upsert Lead in CRM

| Property | Value |
|----------|-------|
| **Type** | Google Sheets (`n8n-nodes-base.googleSheets`) |
| **Technical Role** | Creates or updates a lead record in the Leads tab, matched by phone number |
| **Version** | 4.5 |
| **Operation** | Append or Update (upsert) |
| **Document ID** | `YOUR_GOOGLE_SHEET_ID` |
| **Sheet Name** | `Leads` |
| **Matching Column** | `Phone` |
| **Mapping Mode** | Define Below (explicit) |

**Explicit Column Mappings:**

| Column | Expression |
|--------|-----------|
| Phone | `{{ $('Build AI Prompt').first().json.phone }}` |
| Name | `{{ $('Build AI Prompt').first().json.name }}` |
| Last Message | `{{ $('Build AI Prompt').first().json.messageText }}` |
| Last Contact | `{{ $now.toISO() }}` |
| Status | `New Lead` (hard-coded) |

**Upsert Behaviour:** If a row with the same `Phone` value already exists, it updates the matching columns. If not, it appends a new row.

**Input** | Log AI Reply  
**Output** | Send Reply via Whapi

**Edge Cases & Failure Types:**
- The `Status` field is always overwritten with "New Lead" on every upsert. To preserve status changes (e.g., "Contacted", "Qualified"), the expression should reference an existing status or use conditional logic.
- If multiple leads share the same phone number, only the first match is updated.
- Phone number format must be consistent between History and Leads tabs for matching to work correctly.

---

### Block 1.5 — Response Delivery

**Overview:** The final block sends the AI-generated reply back to the customer via the Whapi.cloud HTTP API.

**Nodes Involved:**
- Send Reply via Whapi

---

#### Node: Send Reply via Whapi

| Property | Value |
|----------|-------|
| **Type** | HTTP Request (`n8n-nodes-base.httpRequest`) |
| **Technical Role** | POSTs the AI reply to Whapi's message endpoint |
| **Version** | 4.2 |
| **Method** | POST |
| **URL** | `https://gate.whapi.cloud/messages/text` |
| **Body Specification** | JSON |
| **JSON Body** | `{{ JSON.stringify({ to: $('Build AI Prompt').first().json.phone, body: $('Generate AI Reply').first().json.text }) }}` |
| **Headers** | `Authorization: Bearer YOUR_TOKEN_HERE`, `Content-Type: application/json` |

**Input** | Upsert Lead in CRM  
**Output** | None (end of workflow)

**Edge Cases & Failure Types:**
- `YOUR_TOKEN_HERE` must be replaced with the actual Whapi API token; otherwise, the API returns 401 Unauthorized.
- If the phone number format is not in E.164 or Whapi's expected format, the message may fail to deliver.
- If the AI reply is empty, Whapi may reject the request.
- Whapi rate limits may apply; high-volume use could hit throttling.
- No error handling branch exists — if the HTTP call fails, the workflow ends with an error status and the AI reply will already be logged in Sheets but not delivered.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|-----------|-----------|----------------|---------------|----------------|-------------|
| Sticky Note — Overview | Sticky Note | Visual documentation — workflow overview and setup checklist | — | — | ## 📱 WhatsApp AI CRM — Whapi + Ollama · Automatically respond to WhatsApp messages with a **local AI model (Ollama)**, log every conversation in **Google Sheets**, and capture leads — all without paying for cloud LLMs. **Use cases:** Real estate, service businesses, agencies, appointment booking, customer support. · ⚙️ Setup Checklist: 1. Connect Google Sheets OAuth2 credentials · 2. Replace the Google Sheet ID in all Sheets nodes · 3. Add your Whapi API token to the Send Reply node · 4. Add your Ollama credentials (base URL of your Ollama server) · 5. Point your Whapi webhook to this workflow's webhook URL · 6. Create the two sheets: History and Leads (see sticky notes below) · 7. ✏️ Edit the system prompt in Build AI Prompt to match your business |
| Sticky Note — Sheets Setup | Sticky Note | Visual documentation — Google Sheet structure | — | — | 📊 Google Sheet Structure · Create **one Google Sheet** with two tabs: · Tab 1 — History (conversation log): Phone, Name, Role, Message, Timestamp · Tab 2 — Leads (CRM): Phone, Name, Last Message, Last Contact, Status · Copy the Sheet ID from the URL: docs.google.com/spreadsheets/d/**[SHEET_ID]**/edit · Paste it into all 4 Google Sheets nodes. |
| Sticky Note — Webhook | Sticky Note | Visual documentation — webhook setup instructions | — | — | 🔗 Step 1 — Receive WhatsApp Message · This webhook receives all incoming events from **Whapi.cloud**. · To connect: 1. Copy this node's **Production URL** · 2. In your Whapi dashboard → Settings → Webhook URL → paste it · 3. Enable the `messages` event type |
| Sticky Note — Filter | Sticky Note | Visual documentation — filter logic explanation | — | — | 🔍 Step 2 — Filter · Only processes **inbound text messages** from customers. Ignores: Messages sent by you (fromMe = true), Media/voice/sticker messages, Empty payloads |
| Sticky Note — AI | Sticky Note | Visual documentation — AI prompt building and model info | — | — | 🧠 Step 3 — Build Prompt + AI Reply · Fetches past conversation history for this contact, then builds a full prompt with: System instructions (your business persona), Conversation history, The new message · ✏️ Customise the system prompt inside the Build AI Prompt code node to match your industry (real estate, bookings, support, etc.) · The Ollama node runs a local model — default is gemma3:1b. Change it to llama3, mistral, or any model you have pulled. |
| Sticky Note — CRM & Send | Sticky Note | Visual documentation — logging, CRM, and reply sending | — | — | 💾 Step 4 — Log + CRM + Send · Log User Message — appends the customer message to History sheet · Log AI Reply — appends the AI response to History sheet · Upsert Lead — creates or updates the lead record in the Leads sheet (matched by phone number) · Send Reply via Whapi — delivers the AI reply back to the customer on WhatsApp · ⚠️ Replace the Bearer token in the Send Reply HTTP node with your Whapi API key. |
| Receive WhatsApp Message | Webhook | Entry point — receives POST from Whapi.cloud | None (entry) | Extract Message Data | 🔗 Step 1 — Receive WhatsApp Message · This webhook receives all incoming events from **Whapi.cloud**. · To connect: 1. Copy this node's **Production URL** · 2. In your Whapi dashboard → Settings → Webhook URL → paste it · 3. Enable the `messages` event type |
| Extract Message Data | Code | Parses Whapi payload, extracts phone/name/message/metadata | Receive WhatsApp Message | Is Text From Customer? | 🔍 Step 2 — Filter · Only processes **inbound text messages** from customers. Ignores: Messages sent by you (fromMe = true), Media/voice/sticker messages, Empty payloads |
| Is Text From Customer? | IF | Gates processing to only inbound text messages from customers | Extract Message Data | Read Conversation History (true branch) | 🔍 Step 2 — Filter · Only processes **inbound text messages** from customers. Ignores: Messages sent by you (fromMe = true), Media/voice/sticker messages, Empty payloads |
| Read Conversation History | Google Sheets | Reads all past messages for this phone number from the History sheet | Is Text From Customer? | Build AI Prompt | 🧠 Step 3 — Build Prompt + AI Reply · Fetches past conversation history for this contact, then builds a full prompt with: System instructions (your business persona), Conversation history, The new message · ✏️ Customise the system prompt inside the Build AI Prompt code node to match your industry (real estate, bookings, support, etc.) · The Ollama node runs a local model — default is gemma3:1b. Change it to llama3, mistral, or any model you have pulled. |
| Build AI Prompt | Code | Assembles full prompt (system instructions + history + new message) | Read Conversation History | Generate AI Reply | 🧠 Step 3 — Build Prompt + AI Reply · Fetches past conversation history for this contact, then builds a full prompt with: System instructions (your business persona), Conversation history, The new message · ✏️ Customise the system prompt inside the Build AI Prompt code node to match your industry (real estate, bookings, support, etc.) · The Ollama node runs a local model — default is gemma3:1b. Change it to llama3, mistral, or any model you have pulled. |
| Generate AI Reply | LangChain LLM Chain | Invokes Ollama model with the full prompt and returns AI text | Build AI Prompt (main), Ollama Model (ai_languageModel) | Log User Message | 🧠 Step 3 — Build Prompt + AI Reply · Fetches past conversation history for this contact, then builds a full prompt with: System instructions (your business persona), Conversation history, The new message · ✏️ Customise the system prompt inside the Build AI Prompt code node to match your industry (real estate, bookings, support, etc.) · The Ollama node runs a local model — default is gemma3:1b. Change it to llama3, mistral, or any model you have pulled. |
| Ollama Model | LangChain Ollama Chat Model | Configures the local Ollama LLM (gemma3:1b) as the AI backend | None (sub-node) | Generate AI Reply (ai_languageModel) | 🧠 Step 3 — Build Prompt + AI Reply · Fetches past conversation history for this contact, then builds a full prompt with: System instructions (your business persona), Conversation history, The new message · ✏️ Customise the system prompt inside the Build AI Prompt code node to match your industry (real estate, bookings, support, etc.) · The Ollama node runs a local model — default is gemma3:1b. Change it to llama3, mistral, or any model you have pulled. |
| Log User Message | Google Sheets | Appends customer's message to the History sheet | Generate AI Reply | Log AI Reply | 💾 Step 4 — Log + CRM + Send · Log User Message — appends the customer message to History sheet · Log AI Reply — appends the AI response to History sheet · Upsert Lead — creates or updates the lead record in the Leads sheet (matched by phone number) · Send Reply via Whapi — delivers the AI reply back to the customer on WhatsApp · ⚠️ Replace the Bearer token in the Send Reply HTTP node with your Whapi API key. |
| Log AI Reply | Google Sheets | Appends AI's reply to the History sheet | Log User Message | Upsert Lead in CRM | 💾 Step 4 — Log + CRM + Send · Log User Message — appends the customer message to History sheet · Log AI Reply — appends the AI response to History sheet · Upsert Lead — creates or updates the lead record in the Leads sheet (matched by phone number) · Send Reply via Whapi — delivers the AI reply back to the customer on WhatsApp · ⚠️ Replace the Bearer token in the Send Reply HTTP node with your Whapi API key. |
| Upsert Lead in CRM | Google Sheets | Creates or updates a lead record in the Leads sheet by phone number | Log AI Reply | Send Reply via Whapi | 💾 Step 4 — Log + CRM + Send · Log User Message — appends the customer message to History sheet · Log AI Reply — appends the AI response to History sheet · Upsert Lead — creates or updates the lead record in the Leads sheet (matched by phone number) · Send Reply via Whapi — delivers the AI reply back to the customer on WhatsApp · ⚠️ Replace the Bearer token in the Send Reply HTTP node with your Whapi API key. |
| Send Reply via Whapi | HTTP Request | Sends the AI reply to the customer via Whapi.cloud API | Upsert Lead in CRM | None (end) | 💾 Step 4 — Log + CRM + Send · Log User Message — appends the customer message to History sheet · Log AI Reply — appends the AI response to History sheet · Upsert Lead — creates or updates the lead record in the Leads sheet (matched by phone number) · Send Reply via Whapi — delivers the AI reply back to the customer on WhatsApp · ⚠️ Replace the Bearer token in the Send Reply HTTP node with your Whapi API key. |

---

## 4. Reproducing the Workflow from Scratch

### Prerequisites

1. **n8n instance** (self-hosted or cloud) with the LangChain nodes package installed (`@n8n/n8n-nodes-langchain`).
2. **Google account** with access to Google Sheets.
3. **Whapi.cloud account** with a connected WhatsApp number and an API token.
4. **Ollama server** running locally or on a reachable host, with the desired model pulled (e.g., `ollama pull gemma3:1b`).

### Google Sheet Preparation

1. Create a new Google Sheet.
2. Create tab **History** with header row: `Phone` | `Name` | `Role` | `Message` | `Timestamp`
3. Create tab **Leads** with header row: `Phone` | `Name` | `Last Message` | `Last Contact` | `Status`
4. Copy the **Sheet ID** from the URL: `docs.google.com/spreadsheets/d/[SHEET_ID]/edit`
5. In n8n, configure a **Google Sheets OAuth2** credential and ensure it has edit access to this sheet.

### Step-by-Step Node Creation

1. **Create the Webhook node**
   - Type: Webhook
   - HTTP Method: POST
   - Path: `whapi-crm`
   - Name it: `Receive WhatsApp Message`

2. **Create the Extract Message Data node**
   - Type: Code
   - Paste the JavaScript code that:
     - Reads `body.messages` from the incoming payload
     - Extracts phone (stripping `@s.whatsapp.net` / `@c.us`), name, messageText, fromMe, messageType, messageId, timestamp
     - Returns `{ skip, phone, name, messageText, fromMe, messageType, messageId, timestamp }`
   - Connect: **Receive WhatsApp Message** → **Extract Message Data**

3. **Create the Is Text From Customer? node**
   - Type: IF
   - Combinator: AND
   - Condition 1: `$json.fromMe` (boolean) equals `false`
   - Condition 2: `$json.messageType` (string) equals `text`
   - Condition 3: `$json.skip` (boolean) equals `false`
   - Connect: **Extract Message Data** → **Is Text From Customer?** (main output → true branch)

4. **Create the Read Conversation History node**
   - Type: Google Sheets
   - Operation: Read
   - Document ID: Paste your Google Sheet ID
   - Sheet Name: `History`
   - Filter: Column `Phone` equals `{{ $('Is Text From Customer?').first().json.phone }}`
   - Toggle **Always Output Data** to ON
   - Credential: Select your Google Sheets OAuth2 credential
   - Connect: **Is Text From Customer?** (true) → **Read Conversation History**

5. **Create the Build AI Prompt node**
   - Type: Code
   - Paste the JavaScript code that:
     - Reads message data from the `Is Text From Customer?` node
     - Reads all history rows from the input
     - Builds conversation history text with role labels
     - Defines the system prompt (customise for your business)
     - Concatenates system prompt + history + new message into `fullPrompt`
     - Returns `{ ...msgData, prompt: fullPrompt }`
   - Connect: **Read Conversation History** → **Build AI Prompt**

6. **Create the Ollama Model node**
   - Type: LangChain Ollama Chat Model
   - Model: `gemma3:1b` (or your preferred model)
   - Options: Leave default (or configure temperature, etc.)
   - Credential: Create/select Ollama API credential with your Ollama server base URL (e.g., `http://localhost:11434`)
   - Do NOT connect this to the main flow; it will be connected to the LLM Chain node as a sub-node

7. **Create the Generate AI Reply node**
   - Type: LangChain LLM Chain
   - Prompt Type: Define
   - Text: `{{ $json.prompt }}`
   - Connect main input: **Build AI Prompt** → **Generate AI Reply**
   - Connect AI sub-node: **Ollama Model** → **Generate AI Reply** (drag from Ollama Model's `ai_languageModel` output to Generate AI Reply's `ai_languageModel` input)

8. **Create the Log User Message node**
   - Type: Google Sheets
   - Operation: Append
   - Document ID: Paste your Google Sheet ID
   - Sheet Name: `History`
   - Mapping Mode: **Define Below** (recommended for reliability)
   - Column mappings:
     - `Phone` = `{{ $('Build AI Prompt').first().json.phone }}`
     - `Name` = `{{ $('Build AI Prompt').first().json.name }}`
     - `Role` = `user`
     - `Message` = `{{ $('Build AI Prompt').first().json.messageText }}`
     - `Timestamp` = `{{ $('Build AI Prompt').first().json.timestamp }}`
   - Credential: Select your Google Sheets OAuth2 credential
   - Connect: **Generate AI Reply** → **Log User Message**

9. **Create the Log AI Reply node**
   - Type: Google Sheets
   - Operation: Append
   - Document ID: Paste your Google Sheet ID
   - Sheet Name: `History`
   - Mapping Mode: Define Below
   - Column mappings:
     - `Phone` = `{{ $('Build AI Prompt').first().json.phone }}`
     - `Name` = `{{ $('Build AI Prompt').first().json.name }}`
     - `Role` = `assistant`
     - `Message` = `{{ $('Generate AI Reply').first().json.text }}`
     - `Timestamp` = `{{ $now.toISO() }}`
   - Credential: Select your Google Sheets OAuth2 credential
   - Connect: **Log User Message** → **Log AI Reply**

10. **Create the Upsert Lead in CRM node**
    - Type: Google Sheets
    - Operation: Append or Update
    - Document ID: Paste your Google Sheet ID
    - Sheet Name: `Leads`
    - Matching Column: `Phone`
    - Mapping Mode: Define Below
    - Column mappings:
      - `Phone` = `{{ $('Build AI Prompt').first().json.phone }}`
      - `Name` = `{{ $('Build AI Prompt').first().json.name }}`
      - `Last Message` = `{{ $('Build AI Prompt').first().json.messageText }}`
      - `Last Contact` = `{{ $now.toISO() }}`
      - `Status` = `New Lead`
    - Credential: Select your Google Sheets OAuth2 credential
    - Connect: **Log AI Reply** → **Upsert Lead in CRM**

11. **Create the Send Reply via Whapi node**
    - Type: HTTP Request
    - Method: POST
    - URL: `https://gate.whapi.cloud/messages/text`
    - Body Specification: JSON
    - JSON Body: `{{ JSON.stringify({ to: $('Build AI Prompt').first().json.phone, body: $('Generate AI Reply').first().json.text }) }}`
    - Headers:
      - `Authorization` = `Bearer YOUR_TOKEN_HERE` (replace with your Whapi API token)
      - `Content-Type` = `application/json`
    - Connect: **Upsert Lead in CRM** → **Send Reply via Whapi**

12. **(Optional) Add Sticky Notes** for documentation — replicate the six sticky notes from the original workflow to annotate each section.

### Webhook Configuration in Whapi

1. Activate the workflow in n8n.
2. Copy the **Production Webhook URL** from the Receive WhatsApp Message node (format: `https://<your-n8n-domain>/webhook/whapi-crm`).
3. In your Whapi.cloud dashboard, navigate to Settings → Webhook URL and paste the URL.
4. Enable the `messages` event type.

### Testing

1. Send a WhatsApp text message to the connected Whapi number.
2. Verify the execution appears in n8n with all nodes completing.
3. Check the Google Sheet: the History tab should have two new rows (user + assistant), and the Leads tab should have one upserted row.
4. Verify you received the AI reply on WhatsApp.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|-------------|-----------------|
| Whapi.cloud is a WhatsApp Business API provider; webhook events are sent as POST JSON payloads with a `messages` array | https://whapi.cloud |
| Ollama must be running and the model pulled before the workflow executes; use `ollama pull gemma3:1b` or your preferred model | https://ollama.com |
| The Google Sheets nodes use OAuth2 credentials — ensure the Google Cloud project has the Sheets API enabled and the n8n OAuth client is authorised | https://developers.google.com/sheets/api |
| The `Log User Message` node originally uses `autoMapInputData` mode which may not correctly map `Role` and `Message` fields; switching to `defineBelow` with explicit expressions is recommended | Internal workflow observation |
| The `Upsert Lead in CRM` node always sets Status to "New Lead", which will overwrite any manually updated status; consider conditional logic if you track lead progression | Internal workflow observation |
| No error-handling branch exists after the HTTP Request node — if the Whapi API call fails, the AI reply is logged in Sheets but not delivered to the customer | Internal workflow observation |
| The workflow uses LangChain nodes which require the `@n8n/n8n-nodes-langchain` community package to be installed in n8n | n8n community nodes |
| To add human handoff logic, insert an IF node after Generate AI Reply that checks for keywords like "speak to someone" or "human" and routes to a different action (e.g., notify via email or Slack) | Workflow customisation suggestion |
| To use a different CRM, replace the Google Sheets nodes with Airtable, Notion, HubSpot, or Pipedrive nodes while preserving the same field mapping | Workflow customisation suggestion |
| For multi-language support, the system prompt already instructs the AI to reply in the customer's language — no additional configuration is required | System prompt design |
| Consider adding a conversation history limit (e.g., last 20 messages) in the Build AI Prompt node to avoid exceeding the Ollama model's context window | Performance consideration |