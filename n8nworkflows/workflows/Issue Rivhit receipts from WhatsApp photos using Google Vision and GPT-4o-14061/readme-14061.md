Issue Rivhit receipts from WhatsApp photos using Google Vision and GPT-4o

https://n8nworkflows.xyz/workflows/issue-rivhit-receipts-from-whatsapp-photos-using-google-vision-and-gpt-4o-14061


# Issue Rivhit receipts from WhatsApp photos using Google Vision and GPT-4o

# 1. Workflow Overview

This workflow receives WhatsApp messages through WAHA, extracts text from image messages with Google Vision OCR, sends the conversation context to a GPT-4o-based AI agent, lets the agent decide whether to ask for clarification, request approval, or issue accounting actions through the Rivhit API, and finally sends the response back to WhatsApp as either text or a PDF attachment.

Its main use case is field collection/accounting operations for small businesses: a courier sends an invoice photo, then a payment proof photo, confirms issuance, and the workflow creates a receipt in Rivhit, closes the invoice, and returns the receipt PDF to WhatsApp.

## 1.1 Input Reception and Source Filtering
The workflow starts from an HTTP webhook connected to WAHA. It accepts incoming WhatsApp events and filters them so only messages from a designated WhatsApp group are processed.

## 1.2 User Feedback and Media Detection
Once a message passes filtering, the workflow triggers a WhatsApp “typing” indicator and checks whether the incoming message contains media.

## 1.3 OCR and Message Assembly
If media is present, the workflow sends the image URL to Google Vision for OCR. It then combines the original WhatsApp message text with the OCR result into one consolidated user prompt for the AI agent.

## 1.4 AI Decisioning with Memory and Rivhit Tooling
The assembled message is enriched with a stored Rivhit API token and passed to a LangChain AI Agent using GPT-4o. The agent uses short-term memory keyed by sender identity and can call a custom Rivhit API tool for receipt creation, invoice closing, searches, customer operations, cancellations, and document sending.

## 1.5 Response Parsing and WhatsApp Delivery
The AI agent is expected to return JSON. The workflow parses that JSON, checks whether a `document_url` exists, and sends either a PDF file with a caption or a plain text WhatsApp reply.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception and Group Filtering

### Overview
This block receives incoming WAHA webhook requests and ensures the workflow only handles messages from the configured bot group. It prevents accidental processing of unrelated chats.

### Nodes Involved
- Webhook
- Only if it's from the bot's group

### Node Details

#### Webhook
- **Type and technical role:** `n8n-nodes-base.webhook`; entry point for incoming HTTP POST requests.
- **Configuration choices:** Configured with POST method and path `whatsapp-receipt-agent`. WAHA should be set to call this endpoint.
- **Key expressions or variables used:** None in the node itself; downstream nodes rely on fields like:
  - `body.payload.from`
  - `body.payload.body`
  - `body.payload.hasMedia`
  - `body.payload.media.url`
  - `body.payload._data.Info.*`
  - `body.session`
- **Input and output connections:** Entry node; outputs to **Only if it's from the bot's group**.
- **Version-specific requirements:** Type version `2.1`.
- **Edge cases or potential failure types:**
  - WAHA webhook payload shape may differ by version/configuration.
  - Missing `body` or `payload` keys will break downstream expressions.
  - Incorrect webhook URL in WAHA means no execution reaches n8n.
- **Sub-workflow reference:** None.

#### Only if it's from the bot's group
- **Type and technical role:** `n8n-nodes-base.filter`; hard filter on sender/group chat ID.
- **Configuration choices:** Compares `{{$json.body.payload.from}}` to a fixed string placeholder currently set to `user@example.com`, which must be replaced with a WhatsApp group ID such as `120363XXXXXXXXXX@g.us`.
- **Key expressions or variables used:** `={{ $json.body.payload.from }}`
- **Input and output connections:** Input from **Webhook**; output to **Start Typing**.
- **Version-specific requirements:** Type version `2.3`.
- **Edge cases or potential failure types:**
  - If not updated, the workflow will reject all real group messages.
  - If payload field `from` is absent, the condition evaluates incorrectly and items are dropped.
  - If WAHA sends direct messages instead of group IDs, filtering may exclude intended traffic.
- **Sub-workflow reference:** None.

---

## 2.2 User Feedback and Media Detection

### Overview
This block immediately signals activity to the WhatsApp user and determines whether the message includes media. That branch controls whether OCR is performed.

### Nodes Involved
- Start Typing
- Is there a picture or not?

### Node Details

#### Start Typing
- **Type and technical role:** `@devlikeapro/n8n-nodes-waha.WAHA`; sends a typing indicator into the chat.
- **Configuration choices:** Uses WAHA “Chatting → Start Typing”. Pulls `chatId` and `session` from the filtered webhook payload.
- **Key expressions or variables used:**
  - `={{ $item("0").$node["Only if it's from the bot's group"].json["body"]["payload"]["from"] }}`
  - `={{ $item("0").$node["Only if it's from the bot's group"].json["body"]["session"] }}`
- **Input and output connections:** Input from **Only if it's from the bot's group**; output to **Is there a picture or not?**
- **Version-specific requirements:** Type version `202502`; requires the installed devlikeapro WAHA node package version compatible with this operation.
- **Edge cases or potential failure types:**
  - WAHA credentials/session issues can prevent typing indicator delivery.
  - If the payload path changes, expressions will fail.
- **Sub-workflow reference:** None.

#### Is there a picture or not?
- **Type and technical role:** `n8n-nodes-base.if`; checks whether media exists.
- **Configuration choices:** Tests boolean `true` against `{{$('Webhook').item.json.body.payload.hasMedia}}`.
- **Key expressions or variables used:** `={{ $('Webhook').item.json.body.payload.hasMedia }}`
- **Input and output connections:** Input from **Start Typing**; true branch goes to **Extracting text from images**, false branch goes directly to **Preparing a message**.
- **Version-specific requirements:** Type version `2.3`.
- **Edge cases or potential failure types:**
  - If `hasMedia` is missing or not a strict boolean, the branch may misroute.
  - Non-image media may still produce `hasMedia = true`, but OCR node assumes a usable image URL.
- **Sub-workflow reference:** None.

---

## 2.3 OCR and Message Preparation

### Overview
This block extracts text from image messages using Google Vision and merges it with the courier’s original text message. It creates the final user input consumed by the AI agent.

### Nodes Involved
- Extracting text from images
- Preparing a message

### Node Details

#### Extracting text from images
- **Type and technical role:** `n8n-nodes-base.httpRequest`; performs a POST request to Google Cloud Vision `images:annotate`.
- **Configuration choices:**
  - URL: `https://vision.googleapis.com/v1/images:annotate`
  - Sends query parameter `key` for the Google API key
  - Sends JSON body with `DOCUMENT_TEXT_DETECTION`
  - Uses the WhatsApp media URL from the webhook payload as `imageUri`
- **Key expressions or variables used:**
  - `{{ $('Webhook').item.json.body.payload.media.url }}`
- **Input and output connections:** Input from true branch of **Is there a picture or not?**; output to **Preparing a message**.
- **Version-specific requirements:** Type version `4.4`.
- **Edge cases or potential failure types:**
  - Missing or invalid Google Vision API key.
  - Vision API not enabled in Google Cloud.
  - Media URL inaccessible from Google servers.
  - WAHA media URL may be temporary or require auth.
  - OCR response may lack `responses[0].textAnnotations[0].description`.
- **Sub-workflow reference:** None.

#### Preparing a message
- **Type and technical role:** `n8n-nodes-base.code`; consolidates text and OCR into one `userMessage`.
- **Configuration choices:** JavaScript code:
  - Reads original WhatsApp message from `Webhook`
  - Defaults to `"I sent an image"` if no text body is present
  - Reads OCR result from **Extracting text from images**
  - Appends OCR result under `[Text extracted from the image I sent:]`
- **Key expressions or variables used inside code:**
  - `$('Webhook').item.json.body.payload.body`
  - `$('Extracting text from images').item.json.responses[0].textAnnotations[0].description`
- **Input and output connections:** Receives from both branches:
  - direct false branch from **Is there a picture or not?**
  - OCR output from **Extracting text from images**
  Then outputs to **api key**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - If OCR node did not run, the code safely catches errors and continues.
  - If webhook payload body is empty and no OCR exists, the AI gets a minimal placeholder.
  - Since the node references another node by name, renaming nodes without updating code will break behavior.
- **Sub-workflow reference:** None.

---

## 2.4 AI Agent, Memory, and Rivhit API Tool

### Overview
This is the core business logic block. It supplies the AI with the user message, persistent conversation memory, a GPT-4o model, and a custom executable tool that maps agent actions into Rivhit API calls.

### Nodes Involved
- api key
- AI Agent
- OpenAI Chat Model
- Simple Memory
- rivhit_api

### Node Details

#### api key
- **Type and technical role:** `n8n-nodes-base.set`; injects the Rivhit API token into the working item.
- **Configuration choices:** Creates a field `api_token` with an empty string placeholder. This must be filled manually.
- **Key expressions or variables used:** Field name `api_token`.
- **Input and output connections:** Input from **Preparing a message**; output to **AI Agent**.
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**
  - If left blank, all Rivhit tool operations will fail.
  - If token is invalid/expired, API errors will be returned from the tool.
- **Sub-workflow reference:** None.

#### OpenAI Chat Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; LLM backend for the agent.
- **Configuration choices:** Uses model `gpt-4o`; requires OpenAI credentials attached in n8n.
- **Key expressions or variables used:** None beyond model selection.
- **Input and output connections:** Connected to **AI Agent** as `ai_languageModel`.
- **Version-specific requirements:** Type version `1.3`; requires compatible LangChain nodes in n8n and valid OpenAI credentials.
- **Edge cases or potential failure types:**
  - Missing credentials.
  - Model access not enabled on the OpenAI account.
  - Rate limits or timeout under heavy usage.
- **Sub-workflow reference:** None.

#### Simple Memory
- **Type and technical role:** `@n8n/n8n-nodes-langchain.memoryBufferWindow`; stores recent messages for each sender.
- **Configuration choices:**
  - Session key is derived from `{{$('Webhook').item.json.body.payload._data.Info.SenderAlt}}`
  - Context window length is 15
- **Key expressions or variables used:** `={{ $('Webhook').item.json.body.payload._data.Info.SenderAlt }}`
- **Input and output connections:** Connected to **AI Agent** as `ai_memory`.
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases or potential failure types:**
  - If `SenderAlt` is missing, conversations may collapse into one shared or invalid session.
  - Memory window of 15 may truncate long operational threads.
  - In multi-user group contexts, this keying strategy assumes sender-based continuity, not group-thread continuity.
- **Sub-workflow reference:** None.

#### rivhit_api
- **Type and technical role:** `@n8n/n8n-nodes-langchain.toolCode`; custom tool the AI can call to interact with Rivhit.
- **Configuration choices:**
  - Accepts `action` and `params` from the AI
  - Reads `api_token` from upstream data or parsed tool payload
  - Builds POST requests to `https://api.rivhit.co.il/online/RivhitOnlineAPI.svc`
  - Supports many actions: receipt creation, invoice closing, reopening/canceling, search, details, email sending, customer creation/update, VAT/bank info
  - Converts/normalizes dates and numeric fields
  - Adds reference/comments automatically for receipt creation
  - Can auto-send email if `email` is supplied
- **Key expressions or variables used:**
  - `rawData.some_input`
  - `rawData.api_token`
  - `action`
  - `params`
  - helper functions such as `parseDocNum`, `formatDate`, `buildCheckPayment`, `buildTransferPayment`, `buildCreditPayment`
- **Input and output connections:** Connected to **AI Agent** as `ai_tool`.
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases or potential failure types:**
  - AI may send malformed JSON in `some_input`.
  - Unsupported `action` returns an “Unrecognized action” string.
  - Rivhit API validation errors are returned as strings, not structured exceptions.
  - `create_receipt_multiple` assumes `params.payments` exists.
  - Credit card flows may succeed while still appending warnings for missing voucher/card info.
  - Tool returns stringified JSON or error strings, so the agent must interpret them correctly.
- **Sub-workflow reference:** None.

#### AI Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`; orchestrates LLM reasoning, memory, and tool usage.
- **Configuration choices:**
  - Input text is `{{$('Preparing a message').item.json.userMessage}}`
  - System prompt is highly specialized for receipt issuance logic, approval gating, invoice-first requirement, formatting, customer identification, and Rivhit usage
  - Expected output format is JSON with at least `message`, optionally `document_url` and `document_name`
- **Key expressions or variables used:**
  - `={{ $('Preparing a message').item.json.userMessage }}`
- **Input and output connections:** Main input from **api key**; receives model from **OpenAI Chat Model**, memory from **Simple Memory**, tool from **rivhit_api**; outputs to **Separation between the message and document link**.
- **Version-specific requirements:** Type version `3.1`.
- **Edge cases or potential failure types:**
  - The system prompt is long and complex; smaller context limits could affect compliance.
  - The next code node assumes valid JSON in `output`. If the agent responds in plain text, parsing fails.
  - Approval logic is prompt-enforced, not technically enforced. The model could theoretically call the tool early if it ignores instructions.
  - Tool error strings may require the model to reason through failures correctly.
- **Sub-workflow reference:** None.

---

## 2.5 Response Parsing and WhatsApp Delivery

### Overview
This block parses the AI response, determines whether a PDF link is present, and sends either a file attachment or a text reply back to the WhatsApp chat.

### Nodes Involved
- Separation between the message and document link
- Only if there is an attachment
- Send a file
- Send a text message

### Node Details

#### Separation between the message and document link
- **Type and technical role:** `n8n-nodes-base.code`; parses the AI agent’s JSON string output.
- **Configuration choices:** Reads `$input.first().json.output`, runs `JSON.parse`, and returns the parsed object.
- **Key expressions or variables used:**
  - `$input.first().json.output`
- **Input and output connections:** Input from **AI Agent**; output to **Only if there is an attachment**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - If `output` is missing or not valid JSON, node execution fails.
  - If the agent returns JSON wrapped in markdown fences, parsing also fails unless cleaned first.
- **Sub-workflow reference:** None.

#### Only if there is an attachment
- **Type and technical role:** `n8n-nodes-base.if`; routes output based on presence of `document_url`.
- **Configuration choices:** Checks whether `$json.document_url` is not empty.
- **Key expressions or variables used:** `={{ $json.document_url }}`
- **Input and output connections:** Input from **Separation between the message and document link**; true branch to **Send a file**, false branch to **Send a text message**.
- **Version-specific requirements:** Type version `2.3`.
- **Edge cases or potential failure types:**
  - If `document_url` exists but is invalid or expired, the file send will fail.
  - If AI returns `document_url` as non-empty garbage, the workflow still attempts file delivery.
- **Sub-workflow reference:** None.

#### Send a file
- **Type and technical role:** `@devlikeapro/n8n-nodes-waha.WAHA`; sends a WhatsApp file attachment.
- **Configuration choices:**
  - WAHA operation: `Chatting → Send File`
  - Sends a PDF using:
    - `filename` from `document_name`
    - `url` from `document_url`
    - `mimetype` fixed to `application/pdf`
  - Uses original chat/session/message ID for reply context
  - Caption is set to AI `message`
- **Key expressions or variables used:**
  - `{{ $json.document_name }}`
  - `{{ $json.document_url }}`
  - `={{ $node["Webhook"].json["body"]["payload"]["from"] }}`
  - `={{ $node["Webhook"].json["body"]["session"] }}`
  - `={{ $node["Webhook"].json["body"]["payload"]["_data"]["Info"]["ID"] }}`
  - `={{ $json.message }}`
- **Input and output connections:** True branch from **Only if there is an attachment**.
- **Version-specific requirements:** Type version `202502`.
- **Edge cases or potential failure types:**
  - WAHA instance may not be able to fetch the remote file URL.
  - URL may require authentication or may expire.
  - If `document_name` is missing, filename may be blank.
- **Sub-workflow reference:** None.

#### Send a text message
- **Type and technical role:** `n8n-nodes-waha.WAHA`; sends plain WhatsApp text.
- **Configuration choices:**
  - WAHA operation: `Chatting → Send Text`
  - Uses original chat/session/message ID for reply threading
  - Sends `{{$json.message}}`
- **Key expressions or variables used:**
  - `={{ $json.message }}`
  - `={{ $node["Webhook"].json["body"]["payload"]["from"] }}`
  - `={{ $node["Webhook"].json["body"]["session"] }}`
  - `={{ $node["Webhook"].json["body"]["payload"]["_data"]["Info"]["ID"] }}`
- **Input and output connections:** False branch from **Only if there is an attachment**.
- **Version-specific requirements:** Type version `202411`; note that this uses a different WAHA package/version naming than the file node.
- **Edge cases or potential failure types:**
  - Credential/session mismatch with WAHA.
  - Missing `message` field leads to blank or rejected sends.
  - Mixed WAHA node packages may require separate installation/compatibility checks.
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook | n8n-nodes-base.webhook | Receives WAHA webhook POST events from WhatsApp |  | Only if it's from the bot's group | ## WhatsApp AI Receipt Agent<br>**Automatically issue receipts from invoice & payment photos**<br><br>### Who is it for<br>Small businesses using Rivhit accounting + WhatsApp for field payment collection.<br><br>### How it works<br>1. Courier sends an **invoice photo** → AI extracts all details via OCR<br>2. Courier sends a **payment photo** (check / bank transfer / credit card voucher) → AI matches it to the invoice<br>3. AI asks for **confirmation** → Courier approves<br>4. **Receipt is created** in Rivhit, invoice is closed, PDF is sent back to WhatsApp<br><br>### Supported payment types<br>Cash, checks, credit cards, bank transfers, and split (multiple) payments.<br><br>### Setup<br>1. Connect **WAHA** to your WhatsApp & set the webhook URL to this workflow<br>2. Add your **Google Vision API key** to the HTTP Request node<br>3. Add your **Rivhit API token** to the `api key` Set node<br>4. Update the **Filter node** with your WhatsApp group ID<br>5. Connect your **OpenAI credentials** to the Chat Model node<br>6. **Activate** the workflow and start sending photos! |
| Only if it's from the bot's group | n8n-nodes-base.filter | Restricts processing to one WhatsApp group/chat | Webhook | Start Typing | ### 1. Webhook & Filtering<br>Receives incoming WhatsApp messages via WAHA and filters to only process messages from the designated bot group.<br>⚠️ **Replace with your WhatsApp group ID**<br>Format: `120363XXXXXXXXXX@g.us`<br>You can find it in the webhook payload under `payload.from`. |
| Start Typing | @devlikeapro/n8n-nodes-waha.WAHA | Sends typing indicator to WhatsApp | Only if it's from the bot's group | Is there a picture or not? |  |
| Is there a picture or not? | n8n-nodes-base.if | Detects whether the incoming message contains media | Start Typing | Extracting text from images; Preparing a message | ### 2. OCR & Message Preparation<br>If the message contains an image → extract text using Google Vision OCR.<br>The extracted text is then appended to the courier's message so the AI Agent can read it. |
| Extracting text from images | n8n-nodes-base.httpRequest | Calls Google Vision OCR on the WhatsApp media URL | Is there a picture or not? | Preparing a message | ### 2. OCR & Message Preparation<br>If the message contains an image → extract text using Google Vision OCR.<br>The extracted text is then appended to the courier's message so the AI Agent can read it.<br>⚠️ **Add your Google Cloud Vision API key**<br>Enable the Vision API in your Google Cloud Console and paste the key in the query parameter below. |
| Preparing a message | n8n-nodes-base.code | Combines user text and OCR result into one message | Is there a picture or not?; Extracting text from images | api key | ### 2. OCR & Message Preparation<br>If the message contains an image → extract text using Google Vision OCR.<br>The extracted text is then appended to the courier's message so the AI Agent can read it. |
| api key | n8n-nodes-base.set | Stores the Rivhit API token for downstream tool use | Preparing a message | AI Agent | ### 3. AI Agent + Rivhit API Tool<br>The AI Agent reads conversation history, extracts invoice/payment details, and calls the Rivhit API tool to:<br>- Issue receipts (cash, check, credit, transfer, multiple)<br>- Close invoices<br>- Search documents<br>- Manage customers<br><br>It asks for confirmation before issuing any receipt.<br>⚠️ **Add your Rivhit API token here**<br>Get it from your Rivhit account settings → API section. |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provides GPT-4o model to the AI agent |  | AI Agent | ### 3. AI Agent + Rivhit API Tool<br>The AI Agent reads conversation history, extracts invoice/payment details, and calls the Rivhit API tool to:<br>- Issue receipts (cash, check, credit, transfer, multiple)<br>- Close invoices<br>- Search documents<br>- Manage customers<br><br>It asks for confirmation before issuing any receipt. |
| Simple Memory | @n8n/n8n-nodes-langchain.memoryBufferWindow | Stores recent conversation context per sender |  | AI Agent | ### 3. AI Agent + Rivhit API Tool<br>The AI Agent reads conversation history, extracts invoice/payment details, and calls the Rivhit API tool to:<br>- Issue receipts (cash, check, credit, transfer, multiple)<br>- Close invoices<br>- Search documents<br>- Manage customers<br><br>It asks for confirmation before issuing any receipt. |
| AI Agent | @n8n/n8n-nodes-langchain.agent | Interprets messages, maintains flow, calls Rivhit tool, formats final JSON | api key | Separation between the message and document link | ### 3. AI Agent + Rivhit API Tool<br>The AI Agent reads conversation history, extracts invoice/payment details, and calls the Rivhit API tool to:<br>- Issue receipts (cash, check, credit, transfer, multiple)<br>- Close invoices<br>- Search documents<br>- Manage customers<br><br>It asks for confirmation before issuing any receipt. |
| rivhit_api | @n8n/n8n-nodes-langchain.toolCode | Exposes Rivhit API actions as an AI-callable tool |  | AI Agent | ### 3. AI Agent + Rivhit API Tool<br>The AI Agent reads conversation history, extracts invoice/payment details, and calls the Rivhit API tool to:<br>- Issue receipts (cash, check, credit, transfer, multiple)<br>- Close invoices<br>- Search documents<br>- Manage customers<br><br>It asks for confirmation before issuing any receipt. |
| Separation between the message and document link | n8n-nodes-base.code | Parses AI JSON output into fields | AI Agent | Only if there is an attachment | ### 4. Response Delivery<br>If a document (PDF) was generated → send it as a file attachment with a caption.<br>Otherwise → send a text-only reply. |
| Only if there is an attachment | n8n-nodes-base.if | Chooses file send vs text send based on document_url | Separation between the message and document link | Send a file; Send a text message | ### 4. Response Delivery<br>If a document (PDF) was generated → send it as a file attachment with a caption.<br>Otherwise → send a text-only reply. |
| Send a file | @devlikeapro/n8n-nodes-waha.WAHA | Sends generated PDF receipt back to WhatsApp | Only if there is an attachment |  | ### 4. Response Delivery<br>If a document (PDF) was generated → send it as a file attachment with a caption.<br>Otherwise → send a text-only reply. |
| Send a text message | n8n-nodes-waha.WAHA | Sends text-only reply back to WhatsApp | Only if there is an attachment |  | ### 4. Response Delivery<br>If a document (PDF) was generated → send it as a file attachment with a caption.<br>Otherwise → send a text-only reply. |
| Sticky Note | n8n-nodes-base.stickyNote | Visual documentation note |  |  | ## WhatsApp AI Receipt Agent<br>**Automatically issue receipts from invoice & payment photos**<br><br>### Who is it for<br>Small businesses using Rivhit accounting + WhatsApp for field payment collection.<br><br>### How it works<br>1. Courier sends an **invoice photo** → AI extracts all details via OCR<br>2. Courier sends a **payment photo** (check / bank transfer / credit card voucher) → AI matches it to the invoice<br>3. AI asks for **confirmation** → Courier approves<br>4. **Receipt is created** in Rivhit, invoice is closed, PDF is sent back to WhatsApp<br><br>### Supported payment types<br>Cash, checks, credit cards, bank transfers, and split (multiple) payments.<br><br>### Setup<br>1. Connect **WAHA** to your WhatsApp & set the webhook URL to this workflow<br>2. Add your **Google Vision API key** to the HTTP Request node<br>3. Add your **Rivhit API token** to the `api key` Set node<br>4. Update the **Filter node** with your WhatsApp group ID<br>5. Connect your **OpenAI credentials** to the Chat Model node<br>6. **Activate** the workflow and start sending photos! |
| Sticky Note1 | n8n-nodes-base.stickyNote | Visual documentation note for webhook/filter block |  |  | ### 1. Webhook & Filtering<br>Receives incoming WhatsApp messages via WAHA and filters to only process messages from the designated bot group. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Visual documentation note for OCR block |  |  | ### 2. OCR & Message Preparation<br>If the message contains an image → extract text using Google Vision OCR.<br>The extracted text is then appended to the courier's message so the AI Agent can read it. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Visual documentation note for AI and Rivhit block |  |  | ### 3. AI Agent + Rivhit API Tool<br>The AI Agent reads conversation history, extracts invoice/payment details, and calls the Rivhit API tool to:<br>- Issue receipts (cash, check, credit, transfer, multiple)<br>- Close invoices<br>- Search documents<br>- Manage customers<br><br>It asks for confirmation before issuing any receipt. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Visual documentation note for response delivery block |  |  | ### 4. Response Delivery<br>If a document (PDF) was generated → send it as a file attachment with a caption.<br>Otherwise → send a text-only reply. |
| Sticky Note5 | n8n-nodes-base.stickyNote | Visual reminder for Rivhit token setup |  |  | ⚠️ **Add your Rivhit API token here**<br>Get it from your Rivhit account settings → API section. |
| Sticky Note6 | n8n-nodes-base.stickyNote | Visual reminder for WhatsApp group ID setup |  |  | ⚠️ **Replace with your WhatsApp group ID**<br>Format: `120363XXXXXXXXXX@g.us`<br>You can find it in the webhook payload under `payload.from`. |
| Sticky Note7 | n8n-nodes-base.stickyNote | Visual reminder for Google Vision API key setup |  |  | ⚠️ **Add your Google Cloud Vision API key**<br>Enable the Vision API in your Google Cloud Console and paste the key in the query parameter below. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it something like:  
   `Issue Rivhit receipts from WhatsApp photos using Google Vision and GPT-4o`.

2. **Add a Webhook node** named **Webhook**.
   - Type: `Webhook`
   - HTTP Method: `POST`
   - Path: `whatsapp-receipt-agent`
   - Save the node.
   - Copy the test/production webhook URL for WAHA configuration later.

3. **Add a Filter node** named **Only if it's from the bot's group**.
   - Type: `Filter`
   - Condition:
     - Left value: `{{$json.body.payload.from}}`
     - Operator: equals
     - Right value: your WhatsApp group ID, e.g. `120363XXXXXXXXXX@g.us`
   - Connect **Webhook → Only if it's from the bot's group**.

4. **Add a WAHA node** named **Start Typing**.
   - Type: `@devlikeapro/n8n-nodes-waha.WAHA`
   - Resource: `Chatting`
   - Operation: `Start Typing`
   - `chatId`: `={{ $item("0").$node["Only if it's from the bot's group"].json["body"]["payload"]["from"] }}`
   - `session`: `={{ $item("0").$node["Only if it's from the bot's group"].json["body"]["session"] }}`
   - Configure WAHA credentials or base connection as required by your installed WAHA node.
   - Connect **Only if it's from the bot's group → Start Typing**.

5. **Add an IF node** named **Is there a picture or not?**
   - Type: `If`
   - Condition:
     - Left value: `={{ $('Webhook').item.json.body.payload.hasMedia }}`
     - Operator: boolean is true
   - Connect **Start Typing → Is there a picture or not?**

6. **Add an HTTP Request node** named **Extracting text from images**.
   - Type: `HTTP Request`
   - Method: `POST`
   - URL: `https://vision.googleapis.com/v1/images:annotate`
   - Enable **Send Query Parameters**
   - Add query parameter:
     - `key` = your Google Cloud Vision API key
   - Enable **Send Body**
   - Body type: JSON
   - JSON body:
     ```json
     {
       "requests": [
         {
           "image": {
             "source": {
               "imageUri": "{{ $('Webhook').item.json.body.payload.media.url }}"
             }
           },
           "features": [
             {
               "type": "DOCUMENT_TEXT_DETECTION"
             }
           ]
         }
       ]
     }
     ```
   - Connect the **true** output of **Is there a picture or not?** to **Extracting text from images**.

7. **Add a Code node** named **Preparing a message**.
   - Type: `Code`
   - Language: JavaScript
   - Paste this logic:
     - Read original WhatsApp body from `Webhook`
     - Default to `"I sent an image"`
     - Try to read OCR text from **Extracting text from images**
     - Append OCR text under a label
     - Return `{ userMessage }`
   - Use this code:
     ```javascript
     var webhookBody = "";
     try {
       webhookBody = $('Webhook').item.json.body.payload.body || "";
     } catch(e) {}
     if (!webhookBody) webhookBody = "I sent an image";
     var ocrText = "";
     try {
       ocrText = $('Extracting text from images').item.json.responses[0].textAnnotations[0].description || "";
     } catch(e) {}
     var userMessage = webhookBody;
     if (ocrText) {
       userMessage += "\n\n[Text extracted from the image I sent:]\n" + ocrText;
     }
     return { json: { userMessage: userMessage } };
     ```
   - Connect:
     - **false** output of **Is there a picture or not?** → **Preparing a message**
     - **Extracting text from images** → **Preparing a message**

8. **Add a Set node** named **api key**.
   - Type: `Set`
   - Add field:
     - `api_token` = your Rivhit API token
   - Keep it as a string field.
   - Connect **Preparing a message → api key**.

9. **Add an OpenAI Chat Model node** named **OpenAI Chat Model**.
   - Type: `OpenAI Chat Model`
   - Model: `gpt-4o`
   - Attach valid OpenAI credentials.
   - No special options are required.

10. **Add a Memory node** named **Simple Memory**.
    - Type: `Memory Buffer Window`
    - Session ID type: custom key
    - Session key:
      `={{ $('Webhook').item.json.body.payload._data.Info.SenderAlt }}`
    - Context window length: `15`

11. **Add an AI Agent node** named **AI Agent**.
    - Type: `LangChain Agent`
    - Prompt/input text:
      `={{ $('Preparing a message').item.json.userMessage }}`
    - System message: paste the full system instructions from the original workflow.
    - This system message is essential because it contains:
      - invoice-first logic
      - approval-before-issuance rule
      - required output JSON format
      - payment-type handling
      - customer lookup policy
      - invoice closing behavior
      - response style constraints

12. **Connect the AI dependencies**:
    - **api key → AI Agent** as normal main input
    - **OpenAI Chat Model → AI Agent** as language model connection
    - **Simple Memory → AI Agent** as memory connection

13. **Add a Tool Code node** named **rivhit_api**.
    - Type: `Tool Code`
    - Description: paste the original descriptive schema/instructions so the agent knows available actions and payloads.
    - Input schema mode: manual/specify input schema enabled
    - Paste the full JavaScript tool implementation from the workflow.
    - This code:
      - interprets `action` and `params`
      - adds Rivhit token
      - normalizes dates/document numbers
      - calls Rivhit endpoints
      - returns result strings to the agent
    - Connect **rivhit_api → AI Agent** as tool connection.

14. **Add a Code node** named **Separation between the message and document link**.
    - Type: `Code`
    - JavaScript:
      ```javascript
      const output = $input.first().json.output;
      const parsed = JSON.parse(output);

      return [{ json: parsed }];
      ```
    - Connect **AI Agent → Separation between the message and document link**.

15. **Add an IF node** named **Only if there is an attachment**.
    - Type: `If`
    - Condition:
      - Left value: `={{ $json.document_url }}`
      - Operator: string not empty
    - Connect **Separation between the message and document link → Only if there is an attachment**.

16. **Add a WAHA file-send node** named **Send a file**.
    - Type: `@devlikeapro/n8n-nodes-waha.WAHA`
    - Resource: `Chatting`
    - Operation: `Send File`
    - `file`:
      ```json
      {
        "mimetype": "application/pdf",
        "filename": "{{ $json.document_name }}",
        "url": "{{ $json.document_url }}"
      }
      ```
    - `chatId`: `={{ $node["Webhook"].json["body"]["payload"]["from"] }}`
    - `session`: `={{ $node["Webhook"].json["body"]["session"] }}`
    - `reply_to`: `={{ $node["Webhook"].json["body"]["payload"]["_data"]["Info"]["ID"] }}`
    - `caption`: `={{ $json.message }}`
    - Connect the **true** output of **Only if there is an attachment** to **Send a file**.

17. **Add a WAHA text-send node** named **Send a text message**.
    - Type: `n8n-nodes-waha.WAHA`
    - Resource: `Chatting`
    - Operation: `Send Text`
    - `text`: `={{ $json.message }}`
    - `chatId`: `={{ $node["Webhook"].json["body"]["payload"]["from"] }}`
    - `session`: `={{ $node["Webhook"].json["body"]["session"] }}`
    - `reply_to`: `={{ $node["Webhook"].json["body"]["payload"]["_data"]["Info"]["ID"] }}`
    - Connect the **false** output of **Only if there is an attachment** to **Send a text message**.

18. **Configure credentials and external systems**:
    - **WAHA**
      - Connect your WhatsApp session in WAHA.
      - Set WAHA webhook URL to the n8n webhook endpoint.
      - Make sure WAHA payloads include:
        - `payload.from`
        - `payload.body`
        - `payload.hasMedia`
        - `payload.media.url`
        - `_data.Info.ID`
        - `_data.Info.SenderAlt`
        - `session`
    - **OpenAI**
      - Add valid OpenAI API credentials to the **OpenAI Chat Model** node.
      - Ensure the account can access `gpt-4o`.
    - **Google Vision**
      - Enable Cloud Vision API.
      - Create an API key.
      - Put that key in the HTTP Request node query parameter.
    - **Rivhit**
      - Generate or retrieve an API token from Rivhit account settings.
      - Put it in the **api key** Set node.

19. **Add optional sticky notes/documentation** matching the original layout:
    - Intro note with purpose and setup
    - Webhook/filter block note
    - OCR block note
    - AI/Rivhit block note
    - Response delivery note
    - Setup warnings for:
      - Rivhit API token
      - WhatsApp group ID
      - Google Vision API key

20. **Test the workflow in stages**:
    - First test with a plain text message from the target group.
    - Then test with an image containing readable invoice text.
    - Then test a payment image.
    - Confirm the AI asks for approval before issuing.
    - Confirm approved issuance creates receipt and closes invoice.
    - Confirm PDF path leads to file delivery.

21. **Harden the workflow before production**:
    - Consider adding error handling after Google Vision, AI parsing, and WAHA send nodes.
    - Consider validating the AI output before `JSON.parse`.
    - Consider replacing the plain Set node token with environment variables or credentials for safer secret handling.

### Sub-workflow setup
This workflow does **not** invoke any n8n sub-workflow node.  
The `rivhit_api` node is an inline AI tool, not a separate workflow.

### Input/output expectations
- **Inbound webhook input:** WAHA WhatsApp event payload with sender, session, text, media flag, and optional media URL.
- **AI output expected shape:**
  - Text-only:
    ```json
    { "message": "..." }
    ```
  - Document response:
    ```json
    {
      "message": "...",
      "document_url": "https://...",
      "document_name": "receipt_12345.pdf"
    }
    ```

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| WhatsApp AI Receipt Agent: automatically issue receipts from invoice and payment photos | Workflow branding/purpose |
| Target audience: small businesses using Rivhit accounting and WhatsApp for field payment collection | Project context |
| Supported payment types: cash, checks, credit cards, bank transfers, split payments | Functional scope |
| WAHA must be connected to WhatsApp and configured to send webhooks to this n8n workflow | WAHA integration requirement |
| Google Cloud Vision API must be enabled and an API key inserted into the OCR HTTP Request node | Google Vision setup |
| Rivhit API token must be added to the `api key` Set node | Rivhit setup |
| The filter node must be updated with the real WhatsApp group ID in the format `120363XXXXXXXXXX@g.us` | WhatsApp group restriction |
| OpenAI credentials must be connected to the Chat Model node | AI model setup |
| Important design assumption: the AI is instructed not to issue receipts before explicit approval, but that rule is prompt-enforced rather than programmatically enforced | Operational risk note |
| Important implementation detail: response parsing assumes the AI always returns valid raw JSON, without markdown formatting | Parsing constraint |
| Important compatibility note: the workflow uses two WAHA node package identifiers/versions (`@devlikeapro/...` and `n8n-nodes-waha...`) | Package compatibility consideration |

Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.