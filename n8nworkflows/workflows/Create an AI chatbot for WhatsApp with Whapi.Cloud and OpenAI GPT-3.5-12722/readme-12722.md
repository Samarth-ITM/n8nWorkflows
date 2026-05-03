Create an AI chatbot for WhatsApp with Whapi.Cloud and OpenAI GPT-3.5

https://n8nworkflows.xyz/workflows/create-an-ai-chatbot-for-whatsapp-with-whapi-cloud-and-openai-gpt-3-5-12722


# Create an AI chatbot for WhatsApp with Whapi.Cloud and OpenAI GPT-3.5

## 1. Workflow Overview

**Title:** Create an AI chatbot for WhatsApp with Whapi.Cloud and OpenAI GPT-3.5  
**Workflow name (internal):** `whatsapp-demo-bot-whapi-n8n-moderation` (inactive by default)

**Purpose:**  
This workflow implements a WhatsApp bot using **Whapi.Cloud**. Incoming messages are received via an **n8n Webhook**, parsed, and routed into one of three paths:
- **AI chat**: messages starting with `/AI ` are sent to **OpenAI (GPT‑3.5 Turbo via the LangChain OpenAI node)** and the response is sent back.
- **Numeric commands**: a single number triggers prebuilt actions (send text/image/document/video/contact/product, create group, message a group, list groups).
- **Fallback**: anything else returns a help menu.

### 1.1 Input Reception & Normalization
Receives Whapi webhook payload, extracts message fields (chat id, text, flags), and normalizes a trimmed command input.

### 1.2 Safety/Loop Prevention & Read Acknowledgement
Ignores messages sent by the bot itself (`from_me=true`) and marks valid incoming messages as “read” in Whapi.

### 1.3 Command Routing
Routes into:
- AI command (`/AI `)
- Numeric command (`^\d+$`)
- Default → help menu

### 1.4 AI Processing & Reply
Extracts AI prompt text, sends it to GPT‑3.5, formats the model response into a WhatsApp text payload, sends it via Whapi.

### 1.5 Numeric Commands (1–9)
Maps digits to specific Whapi actions (media send, contact card, product send, group operations) then sends formatted results back to the chat.

---

## 2. Block-by-Block Analysis

### Block A — Webhook Intake & Message Parsing
**Overview:** Receives WhatsApp events from Whapi.Cloud and extracts core fields into predictable variables used throughout the workflow.  
**Nodes involved:**  
- `1️⃣ Receive WhatsApp Message`
- `2️⃣ Get Message Info`

#### Node: 1️⃣ Receive WhatsApp Message
- **Type / role:** Webhook Trigger (`n8n-nodes-base.webhook`) — entry point for Whapi callbacks.
- **Configuration choices:**
  - HTTP Method: `POST`
  - Path: `8794014c-2794-418c-9432-14a27f49b964` (also used as webhookId)
- **Inputs/outputs:** No input; outputs webhook payload (expected `body.messages[0]` structure).
- **Failure/edge cases:**
  - If Whapi sends a different payload shape (no `body.messages`), downstream expressions will fail.
  - If webhook is not reachable publicly (firewall, wrong URL, not activated), Whapi delivery fails.
- **Sticky note context:** “Webhook Info” — configure webhook URL in Whapi dashboard: https://panel.whapi.cloud/dashboard

#### Node: 2️⃣ Get Message Info
- **Type / role:** Set (`n8n-nodes-base.set`) — extracts/derives fields used by routing.
- **Key assignments:**
  - `message` (object): `{{$json.body.messages[0]}}`
  - `chatId` (string): `{{$json.body.messages[0].chat_id}}`
  - `textBody` (string): `{{$json.body.messages[0].text.body}}`
  - `fromMe` (boolean): `{{$json.body.messages[0].from_me}}`
  - `commandInput` (string): `{{$json.body.messages[0].text.body.trim()}}`
- **Connections:**
  - Input: `1️⃣ Receive WhatsApp Message`
  - Output: `3️⃣ Ignore Bot's Own Messages`
- **Failure/edge cases:**
  - Non-text messages (image/video/etc.) may not have `text.body`; expressions will error unless Whapi always includes it.
  - Empty strings: `.trim()` is safe if `text.body` exists; not safe if it’s `undefined`.

---

### Block B — Ignore Self-Messages + Mark as Read
**Overview:** Prevents feedback loops by ignoring the bot’s own outgoing messages and marks incoming messages as “read” via Whapi.  
**Nodes involved:**
- `3️⃣ Ignore Bot's Own Messages`
- `✔✔ Mark message as read`

#### Node: 3️⃣ Ignore Bot's Own Messages
- **Type / role:** IF (`n8n-nodes-base.if`) — gatekeeping.
- **Condition:** `fromMe` must be `false`  
  - Left value: `={{ $json.fromMe }}`
  - Operation: boolean equals `false`
- **Connections:**
  - Input: `2️⃣ Get Message Info`
  - True output: `✔✔ Mark message as read`
  - False output: not connected (workflow stops for bot-origin messages)
- **Edge cases:**
  - If `fromMe` missing or not boolean, strict validation could cause evaluation issues.

#### Node: ✔✔ Mark message as read
- **Type / role:** HTTP Request (`n8n-nodes-base.httpRequest`) — Whapi API call to mark message as read.
- **Configuration choices:**
  - Method: `PUT`
  - URL: `https://gate.whapi.cloud/messages/{{ $('2️⃣ Get Message Info').item.json.message.id }}`
  - Headers: `accept: application/json`
  - Auth: predefined credential type `whapiChannelApi`
- **Connections:**
  - Input: `3️⃣ Ignore Bot's Own Messages` (true)
  - Output: `4️⃣ What Type of Command?`
- **Failure/edge cases:**
  - Missing `message.id` breaks URL expression.
  - Whapi auth failure (wrong/expired token) → 401/403.
  - Whapi downtime/timeout → node error halts workflow unless error handling is added.

---

### Block C — Primary Routing: AI vs Numeric vs Help
**Overview:** Routes messages to AI processing, numeric command handling, or help menu.  
**Nodes involved:**
- `4️⃣ What Type of Command?`
- `Show Help Menu`

#### Node: 4️⃣ What Type of Command?
- **Type / role:** Switch (`n8n-nodes-base.switch`) — command classifier with named outputs.
- **Rules / outputs (renamed):**
  1. **`ai`** if command starts with `/ai ` (case-insensitive):
     - Left: `{{ $('2️⃣ Get Message Info').item.json.commandInput.toLowerCase() }}`
     - Operation: `startsWith`
     - Right: `"/ai "`
  2. **`numeric`** if command is only digits:
     - Left: `{{ $('2️⃣ Get Message Info').item.json.commandInput }}`
     - Regex: `^\\d+$`
- **Fallback output:** `extra` (goes to help)
- **Connections:**
  - `ai` → `Extract AI Question`
  - `numeric` → `5️⃣ Which Number Command? (1-9)`
  - fallback (`extra`) → `Show Help Menu`
- **Edge cases:**
  - `/AI` without a space (e.g., `/AIHello`) will not match; it falls back to Help.
  - Mixed numeric text (e.g., `1 please`) fails regex and falls back.

#### Node: Show Help Menu
- **Type / role:** Set — prepares a WhatsApp text response listing commands.
- **Key fields:**
  - `to`: chat id from node 2
  - `body`: multi-line help menu including `/AI <your message>`
- **Connections:**
  - Output → `📤 Send Text Message`
- **Edge cases:**
  - None significant; depends on `chatId` existing.

Sticky note context for this block:
- “Route Command Info” — documents AI vs numeric vs default.
- “Switch Node Info” — documents numeric mapping 1–9.

---

### Block D — AI Command Handling (OpenAI → WhatsApp)
**Overview:** Extracts the user’s AI prompt from `/AI ...`, requests a GPT‑3.5 response, then formats and sends it back via Whapi.  
**Nodes involved:**
- `Extract AI Question`
- `Message a model`
- `Format AI Answer`
- `📤 Send Text Message` (shared sender)

#### Node: Extract AI Question
- **Type / role:** Set — derives `aiPrompt`.
- **Expression:**
  - `aiPrompt = {{ $('2️⃣ Get Message Info').item.json.commandInput.substring(4).trim() }}`
  - Assumes `/AI ` is exactly 4 characters (`/` `a` `i` `space`) after `.toLowerCase()` check in routing.
- **Connections:** `4️⃣ What Type of Command? (ai)` → this node → `Message a model`
- **Edge cases:**
  - If the user sends only `/AI `, the prompt becomes empty string; model may return generic output.

#### Node: Message a model
- **Type / role:** OpenAI (LangChain) (`@n8n/n8n-nodes-langchain.openAi`) — generates AI response.
- **Configuration choices (interpreted):**
  - Model: `gpt-3.5-turbo` (selected from list)
  - “Store”: disabled (`store: false`)
  - Messages:
    - User content: `{{ $json.aiPrompt }}`
    - System content: a long instruction to mirror user tone, be conversational, allow emojis/slang, avoid expressing its own personal point of view, etc.
- **Inputs/outputs:**
  - Input: item containing `aiPrompt`
  - Output: structured model response (used downstream as `$json.output[0].content[0].text`)
- **Version-specific notes:**
  - Node type indicates n8n’s LangChain OpenAI integration; requires compatible n8n version supporting `@n8n/n8n-nodes-langchain.openAi` v2.
- **Failure/edge cases:**
  - Missing OpenAI credentials / API key (commonly via env var in self-hosting) → auth error.
  - Rate limits, model unavailability, network timeouts.
  - Output shape mismatch: downstream assumes `output[0].content[0].text` exists.

#### Node: Format AI Answer
- **Type / role:** Set — converts model output into Whapi text message payload.
- **Assignments:**
  - `to = {{ $('2️⃣ Get Message Info').item.json.chatId }}`
  - `body = {{ $json.output[0].content[0].text }}`
- **Connections:** `Message a model` → `Format AI Answer` → `📤 Send Text Message`
- **Edge cases:**
  - If the OpenAI node returns a different schema (or empty content), `body` expression fails.

Sticky note context:
- “AI Integration Info” — Requires `OPENAI_API_KEY`; usage `/AI <your message>`.

---

### Block E — Numeric Command Router (1–9)
**Overview:** Maps digit commands to specialized “Prepare” nodes and Whapi API calls.  
**Nodes involved:**
- `5️⃣ Which Number Command? (1-9)`
- `Prepare: Simple Text`
- `Prepare: Image (Command 2)`
- `Prepare: Document (Command 3)`
- `Prepare: Video (Command 4)`
- `Prepare: Contact (Command 5)`
- `Prepare: Product (Command 6)`
- `Prepare: Create Group (Command 7)`
- `Prepare: Group Message (Command 8)`
- `Get Groups List (Command 9)`
- `Format Groups List`
- `Create WhatsApp Group`
- `Format Group Creation Result`
- Plus all the corresponding `📤 Send ...` HTTP nodes

#### Node: 5️⃣ Which Number Command? (1-9)
- **Type / role:** Switch — numeric dispatcher.
- **Rules:** checks `startsWith` for `"1"`..`"9"` against `commandInput`.
  - Notably: it uses `startsWith`, so `"10"` matches command `1`. (This is a functional edge case.)
- **Connections (each output):**
  - `text` → `Prepare: Simple Text`
  - `image` → `Prepare: Image (Command 2)`
  - `document` → `Prepare: Document (Command 3)`
  - `video` → `Prepare: Video (Command 4)`
  - `contact` → `Prepare: Contact (Command 5)`
  - `product` → `Prepare: Product (Command 6)`
  - `group_create` → `Prepare: Create Group (Command 7)`
  - `group_text` → `Prepare: Group Message (Command 8)`
  - `groups_ids` → `Get Groups List (Command 9)`
- **Edge cases:**
  - As noted, `startsWith` makes `10`, `11`, etc. route to command 1. If you want strict equality, use equals or regex (`^1$`, etc.).

---

### Block F — Command 1: Send Text
**Overview:** Sends a simple text message back to the same chat.  
**Nodes involved:** `Prepare: Simple Text`, `📤 Send Text Message`

#### Node: Prepare: Simple Text
- **Type / role:** Set — prepares Whapi text payload.
- **Fields:**  
  - `to = chatId`  
  - `body = "Simple text message"`
- **Connection:** from switch command 1 → to `📤 Send Text Message`

Sticky note context: “Command 1 Info”.

---

### Block G — Shared Sender: Send Text Message
**Overview:** Common sending node used by multiple paths (help menu, AI answer, command 1, group creation result, group list, group message).  
**Nodes involved:** `📤 Send Text Message`

#### Node: 📤 Send Text Message
- **Type / role:** HTTP Request — sends a WhatsApp text message via Whapi.
- **Configuration:**
  - POST `https://gate.whapi.cloud/messages/text`
  - JSON body: `{"to":"{{ $json.to }}","body":"{{ $json.body }}"}`
  - Header: `Content-Type: application/json`
  - Auth: predefined credential `whapiChannelApi`
- **Inputs:** expects `to` and `body` in incoming JSON.
- **Failure/edge cases:**
  - Missing/invalid `to` (chat id format) → Whapi rejects.
  - Body too long or unsupported characters → API-level error.
  - Credential issues → 401/403.

---

### Block H — Command 2: Send Image
**Overview:** Sends an image by public URL with an optional caption.  
**Nodes involved:** `Prepare: Image (Command 2)`, `📤 Send Image`

#### Node: Prepare: Image (Command 2)
- **Type / role:** Set — prepares `to`, `mediaPath`, `caption`.
- **Media URL:** `https://upload.wikimedia.org/wikipedia/commons/3/3f/JPEG_example_flower.jpg`
- **Edge cases:** media URL must be directly accessible and point to the final file (not an HTML page).

#### Node: 📤 Send Image
- **Type / role:** HTTP Request
- **Endpoint:** POST `https://gate.whapi.cloud/messages/image`
- **Body:** `{"to":"...","media":"...","caption":"..."}`
- **Auth:** `whapiChannelApi`

Sticky note context: “Command 2 Info”.

---

### Block I — Command 3: Send Document
**Overview:** Sends a document (PDF) by public URL with filename and caption.  
**Nodes involved:** `Prepare: Document (Command 3)`, `📤 Send Document`

#### Node: Prepare: Document (Command 3)
- **Fields:** `to`, `mediaPath`, `caption`, `filename`
- **Media URL:** `https://upload.wikimedia.org/wikipedia/commons/2/27/PDF_metadata.pdf`
- **Filename default:** `Example`

#### Node: 📤 Send Document
- **Endpoint:** POST `https://gate.whapi.cloud/messages/document`
- **Body:** includes `filename`

Sticky note context: “Command 3 Info”.

---

### Block J — Command 4: Send Video
**Overview:** Sends a video by public URL with caption.  
**Nodes involved:** `Prepare: Video (Command 4)`, `📤 Send Video`

- **Video URL:** `https://download.samplelib.com/mp4/sample-5s.mp4`
- **Whapi endpoint:** POST `https://gate.whapi.cloud/messages/video`

Sticky note context: “Command 4 Info”.

---

### Block K — Command 5: Send Contact (vCard)
**Overview:** Sends a WhatsApp contact card using vCard content.  
**Nodes involved:** `Prepare: Contact (Command 5)`, `📤 Send Contact`

#### Node: Prepare: Contact (Command 5)
- **Fields:** `to`, `name`, `vcard`
- **vCard:** embedded multi-line vCard 3.0 string.

#### Node: 📤 Send Contact
- **Endpoint:** POST `https://gate.whapi.cloud/messages/contact`
- **Body:** `{"to":"...","name":"...","vcard":"..."}`

Sticky note context includes vCard generator link: https://panel.whapi.cloud/tools/vcard-creation

---

### Block L — Command 6: Send Product
**Overview:** Sends a WhatsApp business product by product ID.  
**Nodes involved:** `Prepare: Product (Command 6)`, `📤 Send Product`

#### Node: Prepare: Product (Command 6)
- **Fields:**  
  - `to = chatId`
  - `ProductID = "YOUR_PRODUCT_ID"` (placeholder; should be replaced)
- **Edge cases:** if not replaced, product send will fail.

#### Node: 📤 Send Product
- **Type / role:** HTTP Request
- **Endpoint (dynamic):** `https://gate.whapi.cloud/business/products/{{ $json.ProductID }}`
- **Method:** POST
- **Body:** `{"to":"{{ $json.to }}"}`
- **Failure/edge cases:**
  - Invalid product ID → 404/400.
  - Requires Whapi business/catalog features enabled.

Sticky note context includes ProductID reference link: https://whapi.readme.io/reference/getproducts

---

### Block M — Command 7: Create Group + Return Group ID
**Overview:** Creates a WhatsApp group with specified participants and reports the resulting group id back to the user.  
**Nodes involved:** `Prepare: Create Group (Command 7)`, `Create WhatsApp Group`, `Format Group Creation Result`, `📤 Send Text Message`

#### Node: Prepare: Create Group (Command 7)
- **Type / role:** Set — provides group subject and participants.
- **Fields:**
  - `subject = "My new group"`
  - `participants` (string containing comma-separated quoted values):  
    - first participant derived from inbound message sender: `{{ $('2️⃣ Get Message Info').item.json.message.from }}`
    - plus two example numbers: `"15550000001", "15550000002"`
- **Edge cases:**
  - `participants` is built as a string that is injected into JSON (`[{{ $json.participants }}]`). If quoting is wrong, JSON becomes invalid.
  - Phone number formats must match Whapi expectations.

#### Node: Create WhatsApp Group
- **Type / role:** HTTP Request — group creation.
- **Endpoint:** POST `https://gate.whapi.cloud/groups`
- **Body:** `{"subject":"...","participants":[...]}`
- **Auth:** `whapiChannelApi`
- **Failure/edge cases:** permissions, invalid participants, rate limits.

#### Node: Format Group Creation Result
- **Type / role:** Set — generates human-readable result.
- **Logic:**  
  - If `$json.group_id` exists: `"Group created. Group id: ... Use it ID in node Prepare: Group Message - before executing command 8"`
  - Else `"Error"`
- **Output:** to `📤 Send Text Message`

Sticky note context: “Command 7 Info”.

---

### Block N — Command 8: Send Message to a Group
**Overview:** Sends a text message to a group chat id stored in an environment variable.  
**Nodes involved:** `Prepare: Group Message (Command 8)`, `📤 Send Text Message`

#### Node: Prepare: Group Message (Command 8)
- **Type / role:** Set
- **Fields:**
  - `to = {{ $env.GROUP_ID }}`
  - `body = "Simple text message for the group"`
- **Edge cases:**
  - If `GROUP_ID` is not set in environment variables, `to` becomes empty → send fails.

Sticky note context: “Command 8 Info”.

---

### Block O — Command 9: List Groups (Top 3) + Send Back
**Overview:** Fetches up to 3 groups from Whapi, formats their ids/names, and sends them to the user.  
**Nodes involved:** `Get Groups List (Command 9)`, `Format Groups List`, `📤 Send Text Message`

#### Node: Get Groups List (Command 9)
- **Type / role:** HTTP Request
- **Endpoint:** `GET https://gate.whapi.cloud/groups?count=3`
- **Headers:** Content-Type application/json (not strictly needed for GET, but harmless)
- **Auth:** `whapiChannelApi`
- **Failure/edge cases:** permissions, empty group list, API changes.

#### Node: Format Groups List
- **Type / role:** Set — converts groups into a text string.
- **Fields:**
  - `to = {{ $('2️⃣ Get Message Info').item.json.message.chat_id }}`
  - `body` expression:
    - If `$json.groups` exists and has items: join formatted `"id - name"` entries with `  &  `
    - Sanitizes group name newlines: `.replace(/[\r\n]+/g, ' ')`
    - Else: `'No groups'`
- **Edge cases:**
  - If Whapi returns groups under a different key, formatting fails or returns “No groups”.

Sticky note context: “Command 9 Info”.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| 1️⃣ Receive WhatsApp Message | Webhook | Entry point for Whapi inbound messages | — | 2️⃣ Get Message Info | ## Webhook Trigger  \nReceives incoming WhatsApp messages from Whapi.Cloud. Configure the webhook URL in [Whapi.Cloud Channel settings](https://panel.whapi.cloud/dashboard) to point to this endpoint. |
| 2️⃣ Get Message Info | Set | Extracts message fields and command text | 1️⃣ Receive WhatsApp Message | 3️⃣ Ignore Bot's Own Messages |  |
| 3️⃣ Ignore Bot's Own Messages | IF | Prevent loop by ignoring bot’s own messages | 2️⃣ Get Message Info | ✔✔ Mark message as read |  |
| ✔✔ Mark message as read | HTTP Request | Marks inbound message as read in Whapi | 3️⃣ Ignore Bot's Own Messages | 4️⃣ What Type of Command? |  |
| 4️⃣ What Type of Command? | Switch | Routes to AI, numeric commands, or help | ✔✔ Mark message as read | Extract AI Question; 5️⃣ Which Number Command? (1-9); Show Help Menu | Routes incoming messages to appropriate handlers:  \n- **AI output:** Messages starting with `/AI ` → OpenAI integration  \n- **Numeric output:** Messages with digits (1-9) → Command routing  \n- **Default output:** Unknown commands → Help message |
| Extract AI Question | Set | Extracts the prompt after `/AI ` | 4️⃣ What Type of Command? | Message a model | Uses OpenAI ChatGPT for AI-powered responses. Requires `OPENAI_API_KEY` environment variable.  \n\n**Usage:** Send `/AI <your message>` to chat with ChatGPT. |
| Message a model | OpenAI (LangChain) | Generates AI response from prompt | Extract AI Question | Format AI Answer | Uses OpenAI ChatGPT for AI-powered responses. Requires `OPENAI_API_KEY` environment variable.  \n\n**Usage:** Send `/AI <your message>` to chat with ChatGPT. |
| Format AI Answer | Set | Maps AI output into `{to, body}` | Message a model | 📤 Send Text Message | Uses OpenAI ChatGPT for AI-powered responses. Requires `OPENAI_API_KEY` environment variable.  \n\n**Usage:** Send `/AI <your message>` to chat with ChatGPT. |
| Show Help Menu | Set | Builds help/command list reply | 4️⃣ What Type of Command? | 📤 Send Text Message | Routes incoming messages to appropriate handlers:  \n- **AI output:** Messages starting with `/AI ` → OpenAI integration  \n- **Numeric output:** Messages with digits (1-9) → Command routing  \n- **Default output:** Unknown commands → Help message |
| 5️⃣ Which Number Command? (1-9) | Switch | Routes digit commands to handlers | 4️⃣ What Type of Command? | Prepare nodes; Get Groups List | Routes numeric commands (1-9) to specific handlers:  \n- **1:** Text message  \n- **2:** Image  \n- **3:** Document  \n- **4:** Video  \n- **5:** Contact  \n- **6:** Product  \n- **7:** Create group  \n- **8:** Group text  \n- **9:** Get groups list |
| Prepare: Simple Text | Set | Prepares command 1 payload | 5️⃣ Which Number Command? (1-9) | 📤 Send Text Message | ## Command 1: Simple Text  \n\n**User sends:** `1`  \n\n**What happens:**  \n1. Prepares text message  \n2. Sends via Whapi.Cloud  \n\n**To customize:** Edit "Prepare: Simple Text" node → Change `body` field |
| 📤 Send Text Message | HTTP Request | Sends text via Whapi | Show Help Menu; Prepare: Simple Text; Format AI Answer; Format Group Creation Result; Prepare: Group Message (Command 8); Format Groups List | — |  |
| Prepare: Image (Command 2) | Set | Prepares image payload | 5️⃣ Which Number Command? (1-9) | 📤 Send Image | ## Command 2: Send Image  \n\n**User sends:** `2`  \n\n**What happens:**  \n1. Prepares image data (URL path, caption)  \n2. Sends via Whapi.Cloud  \n\n**To customize:** Edit "Prepare: Image" node → Change `mediaPath` and `caption` |
| 📤 Send Image | HTTP Request | Sends image via Whapi | Prepare: Image (Command 2) | — | ## Command 2: Send Image  \n\n**User sends:** `2`  \n\n**What happens:**  \n1. Prepares image data (URL path, caption)  \n2. Sends via Whapi.Cloud  \n\n**To customize:** Edit "Prepare: Image" node → Change `mediaPath` and `caption` |
| Prepare: Document (Command 3) | Set | Prepares document payload | 5️⃣ Which Number Command? (1-9) | 📤 Send Document | ## Command 3: Send Document  \n\n**User sends:** `3`  \n\n**What happens:**  \n1. Prepares PDF document data (URL path, caption, filename)  \n2. Sends via Whapi.Cloud  \n\n**To customize:** Edit "Prepare: Document" node → Change `mediaPath` and `caption` and `filename` |
| 📤 Send Document | HTTP Request | Sends document via Whapi | Prepare: Document (Command 3) | — | ## Command 3: Send Document  \n\n**User sends:** `3`  \n\n**What happens:**  \n1. Prepares PDF document data (URL path, caption, filename)  \n2. Sends via Whapi.Cloud  \n\n**To customize:** Edit "Prepare: Document" node → Change `mediaPath` and `caption` and `filename` |
| Prepare: Video (Command 4) | Set | Prepares video payload | 5️⃣ Which Number Command? (1-9) | 📤 Send Video | ## Command 4: Send Video  \n\n**User sends:** `4`  \n\n**What happens:**  \n1. Prepares video data (URL path, caption)  \n2. Sends via Whapi.Cloud  \n\n**To customize:** Edit "Prepare: Video" node → Change `mediaPath` and `caption` |
| 📤 Send Video | HTTP Request | Sends video via Whapi | Prepare: Video (Command 4) | — | ## Command 4: Send Video  \n\n**User sends:** `4`  \n\n**What happens:**  \n1. Prepares video data (URL path, caption)  \n2. Sends via Whapi.Cloud  \n\n**To customize:** Edit "Prepare: Video" node → Change `mediaPath` and `caption` |
| Prepare: Contact (Command 5) | Set | Prepares vCard payload | 5️⃣ Which Number Command? (1-9) | 📤 Send Contact | ## Command 5: Send Contact  \n\n**User sends:** `5`  \n\n**What happens:**  \n1. Prepares contact data (name, vCard). You can use [the vCard generator in the dashboard](https://panel.whapi.cloud/tools/vcard-creation)  \n2. Sends contact card via JSON  \n\n**To customize:** Edit vCard file or "Prepare: Contact" node → Change `name` |
| 📤 Send Contact | HTTP Request | Sends contact card via Whapi | Prepare: Contact (Command 5) | — | ## Command 5: Send Contact  \n\n**User sends:** `5`  \n\n**What happens:**  \n1. Prepares contact data (name, vCard). You can use [the vCard generator in the dashboard](https://panel.whapi.cloud/tools/vcard-creation)  \n2. Sends contact card via JSON  \n\n**To customize:** Edit vCard file or "Prepare: Contact" node → Change `name` |
| Prepare: Product (Command 6) | Set | Prepares ProductID and target | 5️⃣ Which Number Command? (1-9) | 📤 Send Product | ## Command 6: Get Product  \n\n**User sends:** `6`  \n\nYou need [to find out your ProductID](https://whapi.readme.io/reference/getproducts) before you use this command.  \n\n**To customize:**  Edit "Prepare: Product (Command 6)" node → Change `ProductID` |
| 📤 Send Product | HTTP Request | Sends a product message via Whapi | Prepare: Product (Command 6) | — | ## Command 6: Get Product  \n\n**User sends:** `6`  \n\nYou need [to find out your ProductID](https://whapi.readme.io/reference/getproducts) before you use this command.  \n\n**To customize:**  Edit "Prepare: Product (Command 6)" node → Change `ProductID` |
| Prepare: Create Group (Command 7) | Set | Prepares group subject + participants | 5️⃣ Which Number Command? (1-9) | Create WhatsApp Group | ## Command 7: Create Group  \n\n**User sends:** `7`  \n\n**What happens:**  \n1. Extracts sender phone number  \n2. Creates new WhatsApp group  \n3. Adds the sender and several other numbers as participants  \n4. Returns group ID  \n\n**To customize:** Edit "Prepare: Create Group" node → Change `subject` (group name) and `participants` |
| Create WhatsApp Group | HTTP Request | Creates WhatsApp group via Whapi | Prepare: Create Group (Command 7) | Format Group Creation Result | ## Command 7: Create Group  \n\n**User sends:** `7`  \n\n**What happens:**  \n1. Extracts sender phone number  \n2. Creates new WhatsApp group  \n3. Adds the sender and several other numbers as participants  \n4. Returns group ID  \n\n**To customize:** Edit "Prepare: Create Group" node → Change `subject` (group name) and `participants` |
| Format Group Creation Result | Set | Builds success/error message with group_id | Create WhatsApp Group | 📤 Send Text Message | ## Command 7: Create Group  \n\n**User sends:** `7`  \n\n**What happens:**  \n1. Extracts sender phone number  \n2. Creates new WhatsApp group  \n3. Adds the sender and several other numbers as participants  \n4. Returns group ID  \n\n**To customize:** Edit "Prepare: Create Group" node → Change `subject` (group name) and `participants` |
| Prepare: Group Message (Command 8) | Set | Prepares message to group using GROUP_ID | 5️⃣ Which Number Command? (1-9) | 📤 Send Text Message | ## Command 8: Group Message  \n\n**User sends:** `8`  \n\n**What happens:**  \n1. Prepares message for group  \n2. Uses GROUP_ID from environment or provide a specific identifier  \n3. Sends message to group  \n\n**Requires:** `GROUP_ID` in environment variables  \n\n**To customize:** Set `GROUP_ID` in n8n environment variables (get from Command 9) |
| Get Groups List (Command 9) | HTTP Request | Fetches groups list from Whapi | 5️⃣ Which Number Command? (1-9) | Format Groups List | ## Command 9: List Groups  \n\n**User sends:** `9`  \n\n**What happens:**  \n1. Gets list of up to 3 groups  \n2. Formats group IDs and names  \n3. Sends list to user  \n\n**Use this:** To get GROUP_ID for Command 8 |
| Format Groups List | Set | Formats groups into a single reply string | Get Groups List (Command 9) | 📤 Send Text Message | ## Command 9: List Groups  \n\n**User sends:** `9`  \n\n**What happens:**  \n1. Gets list of up to 3 groups  \n2. Formats group IDs and names  \n3. Sends list to user  \n\n**Use this:** To get GROUP_ID for Command 8 |
| Workflow Overview | Sticky Note | Documentation | — | — | ## How it works  \n\nThis workflow is a WhatsApp bot built with the Whapi.Cloud API... (full note content in section 5) |
| Webhook Info | Sticky Note | Documentation | — | — | ## Webhook Trigger… |
| Route Command Info | Sticky Note | Documentation | — | — | Routes incoming messages… |
| AI Integration Info | Sticky Note | Documentation | — | — | Uses OpenAI ChatGPT… |
| Switch Node Info | Sticky Note | Documentation | — | — | Routes numeric commands (1-9)… |
| Command 1 Info | Sticky Note | Documentation | — | — | ## Command 1: Simple Text… |
| Command 2 Info | Sticky Note | Documentation | — | — | ## Command 2: Send Image… |
| Command 3 Info | Sticky Note | Documentation | — | — | ## Command 3: Send Document… |
| Command 4 Info | Sticky Note | Documentation | — | — | ## Command 4: Send Video… |
| Command 5 Info | Sticky Note | Documentation | — | — | ## Command 5: Send Contact… |
| Command 6 Info | Sticky Note | Documentation | — | — | ## Command 6: Get Product… |
| Command 7 Info | Sticky Note | Documentation | — | — | ## Command 7: Create Group… |
| Command 8 Info | Sticky Note | Documentation | — | — | ## Command 8: Group Message… |
| Command 9 Info | Sticky Note | Documentation | — | — | ## Command 9: List Groups… |

> Note: Sticky notes are documentation-only nodes; they do not connect to the execution graph.

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Name it (e.g.) `whatsapp-demo-bot-whapi-n8n-moderation`.
- Keep it inactive until credentials and webhook are set.

2) **Add Webhook Trigger**
- Node: **Webhook**
- Method: `POST`
- Path: generate a unique path (n8n will create one)
- Save the node.

3) **Add “Get Message Info” (Set)**
- Node: **Set**
- Add fields:
  - `message` (Object) → expression: `{{$json.body.messages[0]}}`
  - `chatId` (String) → `{{$json.body.messages[0].chat_id}}`
  - `textBody` (String) → `{{$json.body.messages[0].text.body}}`
  - `fromMe` (Boolean) → `{{$json.body.messages[0].from_me}}`
  - `commandInput` (String) → `{{$json.body.messages[0].text.body.trim()}}`
- Connect: **Webhook → Set**

4) **Add “Ignore Bot’s Own Messages” (IF)**
- Node: **IF**
- Condition: Boolean
  - Value 1: `={{$json.fromMe}}`
  - Operation: “is false”
- Connect: **Get Message Info → IF** (true path used)

5) **Add “Mark message as read” (HTTP Request)**
- Node: **HTTP Request**
- Method: `PUT`
- URL expression: `=https://gate.whapi.cloud/messages/{{ $('2️⃣ Get Message Info').item.json.message.id }}`
- Authentication: **Predefined Credential Type** → create/select **Whapi** credential (see step 14)
- Add header `accept: application/json`
- Connect: **IF (true) → Mark as read**

6) **Add main command router (Switch)**
- Node: **Switch**
- Add rule/output `ai`:
  - Condition: String `startsWith`
  - Left: `={{ $('2️⃣ Get Message Info').item.json.commandInput.toLowerCase() }}`
  - Right: `/ai `
- Add rule/output `numeric`:
  - Condition: String `regex`
  - Left: `={{ $('2️⃣ Get Message Info').item.json.commandInput }}`
  - Right: `^\\d+$`
- Fallback output: enabled (rename/label as “extra” if desired)
- Connect: **Mark as read → Switch**

7) **Add Help Menu (Set)**
- Node: **Set**
- Fields:
  - `to` → `={{ $('2️⃣ Get Message Info').item.json.chatId }}`
  - `body` → help text listing commands 1–9 and `/AI <your message>`
- Connect: **Switch (fallback) → Help Menu**

8) **Add AI branch**
- **Extract AI Question** (Set):
  - `aiPrompt` → `={{ $('2️⃣ Get Message Info').item.json.commandInput.substring(4).trim() }}`
- **OpenAI “Message a model”**:
  - Node: **OpenAI (LangChain)**
  - Model: `gpt-3.5-turbo`
  - Messages:
    - User content: `={{ $json.aiPrompt }}`
    - System content: paste the workflow’s system instruction (tone-mirroring prompt)
  - Credentials: configure OpenAI access (see step 15)
- **Format AI Answer** (Set):
  - `to` → chatId
  - `body` → `={{ $json.output[0].content[0].text }}`
- Connect: **Switch (ai) → Extract AI Question → Message a model → Format AI Answer**

9) **Add numeric command switch (1–9)**
- Node: **Switch**
- Add 9 rules with `startsWith` `"1"` … `"9"` (or replace with strict equality/regex if you want to avoid `10` matching `1`).
- Connect: **Main Switch (numeric) → Numeric Switch**

10) **Create Command 1 nodes**
- Set “Prepare: Simple Text”: fields `to`, `body`
- Connect to shared sender (created in step 13)

11) **Create Command 2–4 media nodes**
- For each command, add a **Set** node with `to`, `mediaPath`, `caption`
- Then an **HTTP Request** node:
  - Image: POST `https://gate.whapi.cloud/messages/image`
  - Document: POST `https://gate.whapi.cloud/messages/document` (+ filename)
  - Video: POST `https://gate.whapi.cloud/messages/video`
  - Auth: Whapi credential
  - Header: `Content-Type: application/json`
  - JSON bodies match the workflow (media URL placed in `media`)
- Connect each numeric output → prepare → send

12) **Create Command 5 contact nodes**
- Set with `to`, `name`, `vcard` (vCard 3.0 string)
- HTTP Request POST `https://gate.whapi.cloud/messages/contact`
- Connect numeric output → prepare → send

13) **Create the shared text sender**
- HTTP Request node “Send Text Message”
- POST `https://gate.whapi.cloud/messages/text`
- JSON body: `{"to":"{{ $json.to }}","body":"{{ $json.body }}"}`
- Header `Content-Type: application/json`
- Auth: Whapi credential
- Connect **Help Menu**, **Format AI Answer**, **Prepare: Simple Text**, **Format Group Creation Result**, **Prepare: Group Message**, **Format Groups List** into this node.

14) **Set up Whapi.Cloud credentials in n8n**
- Create credential matching `whapiChannelApi` (or equivalent HTTP header auth):
  - Either use n8n’s Whapi credential type (if available in your instance), or configure HTTP Request nodes with header:
    - `Authorization: Bearer YOUR_TOKEN`
- Token source: Whapi dashboard https://panel.whapi.cloud/dashboard

15) **Set up OpenAI credentials**
- Configure the OpenAI (LangChain) node credentials.
- If self-hosting and using env vars, ensure `OPENAI_API_KEY` is present for n8n runtime (as referenced by the workflow notes).

16) **Create Command 6 product nodes**
- Set “Prepare: Product” with `to` and `ProductID` (replace placeholder).
- HTTP Request:
  - POST `https://gate.whapi.cloud/business/products/{{ $json.ProductID }}`
  - Body: `{"to":"{{ $json.to }}"}`
- Connect numeric output 6 → prepare → send

17) **Create Command 7 group creation nodes**
- Set “Prepare: Create Group”:
  - `subject`
  - `participants` string that includes the sender and additional numbers
- HTTP Request POST `https://gate.whapi.cloud/groups`
- Set “Format Group Creation Result” to produce `{to, body}` based on `group_id`
- Connect numeric output 7 → prepare → create group → format result → shared text sender

18) **Create Command 8 group message node**
- Set “Prepare: Group Message”
  - `to = {{$env.GROUP_ID}}`
  - `body = ...`
- Connect numeric output 8 → prepare → shared text sender
- Ensure `GROUP_ID` is set in your n8n environment variables.

19) **Create Command 9 list groups nodes**
- HTTP Request GET `https://gate.whapi.cloud/groups?count=3`
- Set “Format Groups List” to map group ids/names into `body`, and `to` back to the original chat.
- Connect numeric output 9 → get groups → format → shared text sender

20) **Configure Whapi webhook**
- Copy the **Production URL** from the Webhook node (not the Test URL).
- In Whapi.Cloud channel settings → Webhooks:
  - Event: `messages`
  - Method: `POST`
  - Mode: `Body`
  - Paste n8n Production URL

21) **Activate workflow**
- Turn workflow to Active once webhook + credentials are correct.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Whapi channel token setup; can use Whapi credential or Authorization header `Bearer YOUR_TOKEN`. | Whapi dashboard: https://panel.whapi.cloud/dashboard |
| Optional env vars mentioned in workflow notes: `OPENAI_API_KEY`, `PRODUCT_ID`, `GROUP_ID` (implementation actually uses `GROUP_ID` and a `ProductID` field in node data). | Applies to AI + group/product commands |
| Webhook must use Production URL in live mode; configure in Whapi webhooks as event `messages`, POST, Body mode. | Whapi dashboard: https://panel.whapi.cloud/dashboard |
| Media URLs must be direct and publicly accessible (must resolve to the actual file). | Used by image/document/video commands |
| vCard generator referenced for contact command. | https://panel.whapi.cloud/tools/vcard-creation |
| Product ID lookup reference. | https://whapi.readme.io/reference/getproducts |

Disclaimer (as provided):  
Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.