Extract Indonesian stock (IDX) trades from Telegram invoices with Gemini AI

https://n8nworkflows.xyz/workflows/extract-indonesian-stock--idx--trades-from-telegram-invoices-with-gemini-ai-14252


# Extract Indonesian stock (IDX) trades from Telegram invoices with Gemini AI

# 1. Workflow Overview

This workflow receives Telegram updates, detects uploaded broker invoices or trade confirmations, downloads the attached file, converts it to base64, sends it to OpenRouter using a Gemini model, parses the extracted stock transactions, and asks the user to confirm saving the batch.

Its main use case is structured extraction of Indonesian stock exchange (IDX) trades from broker PDFs or images sent through Telegram. The workflow is designed to normalize AI output into transaction records that can later be routed to storage or downstream systems.

## 1.1 Message Intake and Routing

The workflow starts from a Telegram webhook trigger. Incoming updates are normalized into a consistent structure, then routed depending on whether they are callback button events or file uploads.

## 1.2 File Download and Preparation

If the Telegram message contains a document or photo, the workflow extracts file metadata, downloads the binary file from Telegram, and converts the binary content into base64 for AI submission.

## 1.3 AI Request Construction and Extraction

The workflow builds a multimodal request for OpenRouter. It formats PDFs differently from images, sends the request to Gemini through OpenRouter, and expects raw JSON describing all detected trades.

## 1.4 Response Parsing and Error Handling

The AI response is cleaned, parsed as JSON, normalized into one transaction per item, and checked for parsing or response errors. Errors are sent back to the user via Telegram.

## 1.5 Confirmation and Batch Staging

Successful transaction items are grouped into a human-readable summary. The batch is stored temporarily in workflow static data, and a Telegram confirmation message is sent so the user can later approve or cancel the full batch.

---

# 2. Block-by-Block Analysis

## 2.1 Message Intake and Routing

### Overview

This block receives Telegram updates and standardizes them into a simplified message format. It distinguishes normal messages from callback queries and routes file uploads toward the extraction branch.

### Nodes Involved

- Telegram Trigger
- Parse Message
- Route Type1

### Node Details

#### Telegram Trigger

- **Type and technical role:** `n8n-nodes-base.telegramTrigger`; webhook-based entry point for Telegram bot updates.
- **Configuration choices:** Configured to receive `message` and `callback_query` updates.
- **Key expressions or variables used:** None in parameters.
- **Input and output connections:** Entry node; outputs to **Parse Message**.
- **Version-specific requirements:** Type version `1.1`. Requires Telegram bot credentials and a publicly reachable HTTPS webhook URL.
- **Edge cases or potential failure types:**
  - Telegram credential or webhook registration failure
  - Public URL not reachable by Telegram
  - Bot not activated or webhook blocked
  - Unsupported update structure
- **Sub-workflow reference:** None.

#### Parse Message

- **Type and technical role:** `n8n-nodes-base.code`; converts raw Telegram update payloads into a normalized JSON structure.
- **Configuration choices:**
  - Detects `callback_query` and outputs fields such as `is_callback`, `callback_id`, `callback_data`, `chat_id`, `message_id`, and `username`.
  - Detects standard messages and determines whether they contain a file via `document` or `photo`.
  - Parses slash commands if no file is present.
  - Extracts Telegram `file_id`, `filename`, and `mime_type`.
- **Key expressions or variables used:**
  - `update.callback_query`
  - `update.message`
  - `msg.document?.file_id`
  - `msg.photo[msg.photo.length - 1].file_id`
- **Input and output connections:** Input from **Telegram Trigger**; output to **Route Type1**.
- **Version-specific requirements:** Code node version `2`.
- **Edge cases or potential failure types:**
  - Unexpected Telegram payload shape
  - `message` missing on non-callback updates
  - Photo uploads default to `image/jpeg`
  - Non-file messages are accepted but not further processed in current routing
- **Sub-workflow reference:** None.

#### Route Type1

- **Type and technical role:** `n8n-nodes-base.switch`; routes normalized input by event type.
- **Configuration choices:**
  - Output `callback` when `is_callback` is true
  - Output `file` when `has_file` is true
  - Fallback output named `extra`
- **Key expressions or variables used:**
  - `={{ $json.is_callback }}`
  - `={{ $json.has_file }}`
- **Input and output connections:**
  - Input from **Parse Message**
  - `file` output goes to **Extract File Info**
  - `callback` and `extra` outputs are currently unconnected
- **Version-specific requirements:** Switch node version `3`.
- **Edge cases or potential failure types:**
  - Callback queries are detected but not processed further in this exported workflow
  - Text commands and miscellaneous messages drop into fallback with no response
- **Sub-workflow reference:** None.

---

## 2.2 File Download and Preparation

### Overview

This block isolates file metadata, downloads the binary from Telegram servers, and transforms it into base64. The result is a clean file payload ready for multimodal AI processing.

### Nodes Involved

- Extract File Info
- Download File
- Prepare File

### Node Details

#### Extract File Info

- **Type and technical role:** `n8n-nodes-base.code`; reduces the parsed message payload to only the fields needed downstream.
- **Configuration choices:** Outputs `file_id`, `filename`, `mime_type`, `chat_id`, and `username`.
- **Key expressions or variables used:** Reads `const data = $input.first().json`.
- **Input and output connections:** Input from **Route Type1**; output to **Download File**.
- **Version-specific requirements:** Code node version `2`.
- **Edge cases or potential failure types:**
  - Missing `file_id`
  - Inaccurate filename defaults for photos and unnamed files
- **Sub-workflow reference:** None.

#### Download File

- **Type and technical role:** `n8n-nodes-base.telegram`; downloads a Telegram file as binary data.
- **Configuration choices:**
  - Resource is `file`
  - `fileId` is taken from `={{ $json.file_id }}`
- **Key expressions or variables used:**
  - `={{ $json.file_id }}`
- **Input and output connections:** Input from **Extract File Info**; output to **Prepare File**.
- **Version-specific requirements:** Telegram node version `1.2`.
- **Edge cases or potential failure types:**
  - Invalid or expired Telegram file ID
  - Telegram API/network failure
  - Binary download issues
- **Sub-workflow reference:** None.

#### Prepare File

- **Type and technical role:** `n8n-nodes-base.code`; reads downloaded binary data and converts it to base64.
- **Configuration choices:**
  - Retrieves metadata from **Extract File Info**
  - Uses `this.helpers.getBinaryDataBuffer(0, 'data')`
  - Returns `file_base64`, file metadata, and `received_at`
- **Key expressions or variables used:**
  - `$('Extract File Info').first().json`
  - `await this.helpers.getBinaryDataBuffer(0, 'data')`
- **Input and output connections:** Input from **Download File**; output to **Build Request**.
- **Version-specific requirements:**
  - Code node version `2`
  - Workflow setting `binaryMode` is `separate`
  - The code comment explicitly assumes filesystem-style binary handling
- **Edge cases or potential failure types:**
  - Binary property name `data` missing
  - Large files causing memory pressure during base64 conversion
  - Binary mode differences between environments
- **Sub-workflow reference:** None.

---

## 2.3 AI Request Construction and Extraction

### Overview

This block creates the OpenRouter request payload, selecting the correct content format for PDF versus image input. It then sends the request to Gemini and retrieves the model response.

### Nodes Involved

- Build Request
- OpenRouter Extract

### Node Details

#### Build Request

- **Type and technical role:** `n8n-nodes-base.code`; builds a multimodal OpenRouter chat-completions payload.
- **Configuration choices:**
  - Detects PDFs using `mime_type === 'application/pdf'`
  - For PDFs, uses a `file` content object with a data URL
  - For images, uses `image_url` with a data URL
  - Creates a detailed extraction prompt tailored to Indonesian broker trade confirmations
  - Sets:
    - model: `google/gemini-2.5-flash-lite`
    - temperature: `0`
    - max_tokens: `2048`
  - Adds plugin configuration for PDFs:
    - `file-parser`
    - OCR engine `mistral-ocr`
- **Key expressions or variables used:**
  - `const isPdf = d.mime_type === 'application/pdf'`
  - `data:${d.mime_type};base64,${d.file_base64}`
  - `Document: ${d.filename}`
- **Input and output connections:** Input from **Prepare File**; output to **OpenRouter Extract**.
- **Version-specific requirements:** Code node version `2`.
- **Edge cases or potential failure types:**
  - Unsupported MIME type
  - Payload too large for model/API limits
  - Prompt/output mismatch if the model ignores “raw JSON only”
  - PDF OCR plugin availability may depend on OpenRouter support
- **Sub-workflow reference:** None.

#### OpenRouter Extract

- **Type and technical role:** `n8n-nodes-base.httpRequest`; sends the extraction request to OpenRouter.
- **Configuration choices:**
  - Method: `POST`
  - URL: `https://openrouter.ai/api/v1/chat/completions`
  - Body sent as JSON from `={{ $json.payload }}`
  - Generic credential auth using HTTP Header Auth
  - Additional headers:
    - `HTTP-Referer: https://n8n.io`
    - `X-Title: IDX Invoice Reader`
  - Timeout: 30000 ms
- **Key expressions or variables used:**
  - `={{ $json.payload }}`
- **Input and output connections:** Input from **Build Request**; output to **Parse Invoice**.
- **Version-specific requirements:** HTTP Request version `4.2`.
- **Edge cases or potential failure types:**
  - Missing or invalid OpenRouter API key
  - Rate limiting
  - Timeout on large PDFs or slow OCR/model execution
  - Non-200 responses or malformed API payloads
- **Sub-workflow reference:** None.

---

## 2.4 Response Parsing and Error Handling

### Overview

This block extracts the model text response, strips code fences if present, parses it as JSON, and normalizes it into transaction items. It also catches response or parsing failures and routes them to a Telegram error reply.

### Nodes Involved

- Parse Invoice
- Invoice Error?
- Reply Invoice Error

### Node Details

#### Parse Invoice

- **Type and technical role:** `n8n-nodes-base.code`; validates and parses the LLM response into normalized trade records.
- **Configuration choices:**
  - Reads the text from `resp.choices[0].message.content`
  - Strips possible markdown code blocks
  - Parses JSON
  - Wraps a single object into an array if needed
  - Emits one output item per transaction
  - Normalizes:
    - ticker to uppercase without `.JK`
    - company name default `unknown`
    - default type `buy`
    - numeric coercion for quantity, price, fee, total_amount, confidence
  - Adds `chat_id`, `filename`, and `total_in_batch`
- **Key expressions or variables used:**
  - `resp.choices[0].message.content.trim()`
  - `$('Build Request').first().json.chat_id`
  - `JSON.parse(raw)`
- **Input and output connections:** Input from **OpenRouter Extract**; output to **Invoice Error?**.
- **Version-specific requirements:** Code node version `2`.
- **Edge cases or potential failure types:**
  - OpenRouter response missing `choices`
  - Model returning markdown, commentary, or invalid JSON
  - Numeric conversion collapsing invalid values to `0`
  - If the model returns nested structures instead of direct array/object, parsing logic will not handle it
- **Sub-workflow reference:** None.

#### Invoice Error?

- **Type and technical role:** `n8n-nodes-base.if`; separates parser errors from successful extraction items.
- **Configuration choices:** Tests whether `$json.error` exists.
- **Key expressions or variables used:**
  - `={{ $json.error }}`
- **Input and output connections:**
  - Input from **Parse Invoice**
  - True output goes to **Reply Invoice Error**
  - False output goes to **Format Confirmation**
- **Version-specific requirements:** IF node version `2`.
- **Edge cases or potential failure types:**
  - Assumes error items contain `error`; all normal transactions must omit it
  - Mixed item batches with both success and error are not expected
- **Sub-workflow reference:** None.

#### Reply Invoice Error

- **Type and technical role:** `n8n-nodes-base.telegram`; sends a human-readable error message back to Telegram.
- **Configuration choices:**
  - Sends a message containing `{{ $json.error_message }}`
  - Targets `={{ $json.chat_id }}`
- **Key expressions or variables used:**
  - `{{ $json.error_message }}`
  - `={{ $json.chat_id }}`
- **Input and output connections:** Input from **Invoice Error?** true branch; no downstream node.
- **Version-specific requirements:** Telegram node version `1.2`.
- **Edge cases or potential failure types:**
  - Telegram send failure
  - Missing `chat_id`
  - User sees only generic parse failure if upstream HTTP node itself throws and stops execution before this point
- **Sub-workflow reference:** None.

---

## 2.5 Confirmation and Batch Staging

### Overview

This block aggregates all extracted transactions into one summary message, stores the batch in workflow static data, and sends confirmation buttons to Telegram. It is the staging point before any permanent save action is implemented.

### Nodes Involved

- Format Confirmation
- Send Confirmation

### Node Details

#### Format Confirmation

- **Type and technical role:** `n8n-nodes-base.code`; aggregates transaction items into a single batch confirmation payload.
- **Configuration choices:**
  - Uses all incoming items from successful parse output
  - Reads `chat_id`, `filename`, `broker`, `transaction_date` from the first transaction
  - Stores the full batch in workflow global static data under a generated key `batch_<id>`
  - Builds a Markdown-style summary line for each transaction
  - Computes total amount and total fee
  - Creates callback payloads:
    - `confirm_batch|<batchId>`
    - `cancel_batch|<batchId>`
  - Outputs `text`, `chat_id`, and a `reply_markup` array
- **Key expressions or variables used:**
  - `$input.all()`
  - `$getWorkflowStaticData('global')`
  - `Date.now().toString(36) + Math.random().toString(36).slice(2, 6)`
- **Input and output connections:** Input from **Invoice Error?** false branch; output to **Send Confirmation**.
- **Version-specific requirements:** Code node version `2`.
- **Edge cases or potential failure types:**
  - Workflow static data may grow indefinitely without cleanup
  - Generated `reply_markup` is not used by the next node as currently configured
  - Assumes all items belong to one invoice/batch
  - Large transaction lists may exceed Telegram message length limits
- **Sub-workflow reference:** None.

#### Send Confirmation

- **Type and technical role:** `n8n-nodes-base.telegram`; sends the confirmation summary to the Telegram chat.
- **Configuration choices:**
  - Sends `={{ $json.text }}`
  - Targets `={{ $json.chat_id }}`
  - No inline keyboard or additional fields are currently configured in the node
- **Key expressions or variables used:**
  - `={{ $json.text }}`
  - `={{ $json.chat_id }}`
- **Input and output connections:** Input from **Format Confirmation**; no downstream node.
- **Version-specific requirements:** Telegram node version `1.2`.
- **Edge cases or potential failure types:**
  - The workflow description claims confirmation buttons are sent, but `reply_markup` is not actually mapped in this Telegram node
  - Markdown-like formatting in text may not render as intended unless parse mode is configured
  - Telegram message length limits may be exceeded for large batches
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note - Overview | stickyNote | Visual documentation for workflow purpose, setup, and customization |  |  | ## 🇮🇩 Indonesian Stock (IDX) Invoice Reader<br>Automatically extract stock transactions from Indonesian broker trade confirmations sent via Telegram.<br><br>### How it works<br>1. Send a broker PDF or image to your Telegram bot<br>2. The file is downloaded and encoded as base64<br>3. OpenRouter (Gemini) extracts all transactions as structured JSON<br>4. A single batch confirmation is sent with ✅ / ❌ buttons<br>5. On confirm, the structured data is output — connect to any destination node<br><br>### Setup<br>1. Create a bot via BotFather → add **Telegram account** credential<br>2. Get an OpenRouter API key → add **Header Auth** credential named **OpenRouter API** with value `Authorization: Bearer sk-or-...`<br>3. Expose n8n via HTTPS (Cloudflare Tunnel, ngrok, or a public server)<br>4. Activate the workflow — Telegram webhook registers automatically<br><br>### Customization<br>Swap the **Send Confirmation** output to any destination (HTTP Request, Google Sheets, Airtable, etc.). Change the model in **Build Request** — default is `google/gemini-2.5-flash-lite`. |
| Sticky Note - Message Intake | stickyNote | Visual note for Telegram intake and routing |  |  | ## 📨 Message Intake<br>Receives all Telegram updates (messages and button callbacks), parses the incoming data, and routes file uploads to the processing branch. |
| Sticky Note - File Processing | stickyNote | Visual note for file download and base64 preparation |  |  | ## 📁 File Processing<br>Downloads the file from Telegram servers, reads the binary data, and encodes it as base64 ready for the AI request. |
| Sticky Note - AI Extraction | stickyNote | Visual note for AI request creation and parsing |  |  | ## 🤖 AI Extraction<br>Builds the OpenRouter request (PDF or image format), calls Gemini, and parses the JSON array of transactions from the response. |
| Sticky Note - Confirmation | stickyNote | Visual note for staging and confirmation |  |  | ## ✅ Confirmation<br>Groups all extracted transactions into a batch, stores them temporarily in workflow static data, and sends a confirmation message with Save All / Cancel buttons. |
| Telegram Trigger | telegramTrigger | Receives Telegram bot updates via webhook |  | Parse Message | ## 📨 Message Intake<br>Receives all Telegram updates (messages and button callbacks), parses the incoming data, and routes file uploads to the processing branch. |
| Parse Message | code | Normalizes Telegram updates into a simplified internal structure | Telegram Trigger | Route Type1 | ## 📨 Message Intake<br>Receives all Telegram updates (messages and button callbacks), parses the incoming data, and routes file uploads to the processing branch. |
| Route Type1 | switch | Routes callback events and file uploads into different branches | Parse Message | Extract File Info | ## 📨 Message Intake<br>Receives all Telegram updates (messages and button callbacks), parses the incoming data, and routes file uploads to the processing branch. |
| Extract File Info | code | Keeps only the file metadata required for download and later processing | Route Type1 | Download File | ## 📁 File Processing<br>Downloads the file from Telegram servers, reads the binary data, and encodes it as base64 ready for the AI request. |
| Download File | telegram | Downloads the Telegram document/photo as binary data | Extract File Info | Prepare File | ## 📁 File Processing<br>Downloads the file from Telegram servers, reads the binary data, and encodes it as base64 ready for the AI request. |
| Prepare File | code | Converts binary file data to base64 and restores metadata | Download File | Build Request | ## 📁 File Processing<br>Downloads the file from Telegram servers, reads the binary data, and encodes it as base64 ready for the AI request. |
| Build Request | code | Creates the OpenRouter multimodal extraction payload | Prepare File | OpenRouter Extract | ## 🤖 AI Extraction<br>Builds the OpenRouter request (PDF or image format), calls Gemini, and parses the JSON array of transactions from the response. |
| OpenRouter Extract | httpRequest | Sends the multimodal extraction request to OpenRouter | Build Request | Parse Invoice | ## 🤖 AI Extraction<br>Builds the OpenRouter request (PDF or image format), calls Gemini, and parses the JSON array of transactions from the response. |
| Parse Invoice | code | Parses and normalizes AI output into one transaction per item | OpenRouter Extract | Invoice Error? | ## 🤖 AI Extraction<br>Builds the OpenRouter request (PDF or image format), calls Gemini, and parses the JSON array of transactions from the response. |
| Invoice Error? | if | Routes failed parsing results versus valid transaction items | Parse Invoice | Reply Invoice Error; Format Confirmation | ## ✅ Confirmation<br>Groups all extracted transactions into a batch, stores them temporarily in workflow static data, and sends a confirmation message with Save All / Cancel buttons. |
| Format Confirmation | code | Aggregates extracted trades into one batch confirmation message and stores batch data | Invoice Error? | Send Confirmation | ## ✅ Confirmation<br>Groups all extracted transactions into a batch, stores them temporarily in workflow static data, and sends a confirmation message with Save All / Cancel buttons. |
| Send Confirmation | telegram | Sends the summary confirmation message to Telegram | Format Confirmation |  | ## ✅ Confirmation<br>Groups all extracted transactions into a batch, stores them temporarily in workflow static data, and sends a confirmation message with Save All / Cancel buttons. |
| Reply Invoice Error | telegram | Sends a failure message when invoice parsing fails | Invoice Error? |  | ## ✅ Confirmation<br>Groups all extracted transactions into a batch, stores them temporarily in workflow static data, and sends a confirmation message with Save All / Cancel buttons. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like `Indonesian Stock (IDX) Invoice Reader`.
   - In workflow settings:
     - Set **Execution Order** to `v1`.
     - Set **Binary Data Mode** to `separate`.

2. **Create Telegram credentials**
   - Create a Telegram bot through BotFather.
   - In n8n, add a **Telegram account** credential for both trigger and send/download nodes.
   - Ensure your n8n instance is exposed through a valid public HTTPS URL so Telegram webhooks can register.

3. **Create OpenRouter credentials**
   - Add a **HTTP Header Auth** credential.
   - Store the header as:
     - `Authorization: Bearer sk-or-...`
   - Use a recognizable name such as `OpenRouter API`.

4. **Add `Telegram Trigger`**
   - Node type: **Telegram Trigger**.
   - Configure update types:
     - `message`
     - `callback_query`
   - Connect it to the next node.

5. **Add `Parse Message`**
   - Node type: **Code**.
   - Paste logic that:
     - detects callback queries
     - detects whether a standard message contains a file
     - extracts chat metadata
     - extracts `file_id`, `filename`, and `mime_type`
     - parses slash commands for non-file messages
   - Connect `Telegram Trigger -> Parse Message`.

6. **Add `Route Type1`**
   - Node type: **Switch**.
   - Add output rule `callback` where `{{$json.is_callback}}` is true.
   - Add output rule `file` where `{{$json.has_file}}` is true.
   - Set fallback output to `extra`.
   - Connect `Parse Message -> Route Type1`.

7. **Add `Extract File Info`**
   - Node type: **Code**.
   - Return only:
     - `file_id`
     - `filename`
     - `mime_type`
     - `chat_id`
     - `username`
   - Connect the `file` output of `Route Type1` to `Extract File Info`.

8. **Add `Download File`**
   - Node type: **Telegram**.
   - Resource: **File**.
   - Set **File ID** to `{{$json.file_id}}`.
   - Use the Telegram credential.
   - Connect `Extract File Info -> Download File`.

9. **Add `Prepare File`**
   - Node type: **Code**.
   - In the code:
     - fetch metadata from `Extract File Info`
     - read binary buffer from property `data`
     - convert the buffer to base64
     - return:
       - `file_base64`
       - `mime_type`
       - `filename`
       - `chat_id`
       - `username`
       - `received_at`
   - Connect `Download File -> Prepare File`.

10. **Add `Build Request`**
    - Node type: **Code**.
    - Build a payload for `https://openrouter.ai/api/v1/chat/completions`.
    - Logic should:
      - detect whether the file is PDF
      - for PDF, use a `file` object with `file_data: data:<mime>;base64,<content>`
      - for images, use `image_url.url: data:<mime>;base64,<content>`
      - write a prompt instructing the model to return only a raw JSON array of transactions
      - include normalization rules:
        - uppercase ticker without `.JK`
        - lots, not shares
        - IDR numeric values
        - `transaction_date` as `YYYY-MM-DD`
        - confidence score
      - set:
        - `model = google/gemini-2.5-flash-lite`
        - `temperature = 0`
        - `max_tokens = 2048`
      - if PDF, add `plugins: [{ id: 'file-parser', pdf: { engine: 'mistral-ocr' } }]`
    - Return `chat_id`, `filename`, `mime_type`, and `payload`.
    - Connect `Prepare File -> Build Request`.

11. **Add `OpenRouter Extract`**
    - Node type: **HTTP Request**.
    - Configure:
      - Method: `POST`
      - URL: `https://openrouter.ai/api/v1/chat/completions`
      - Authentication: **Generic Credential Type**
      - Generic auth type: **HTTP Header Auth**
      - Credential: your OpenRouter credential
      - Send headers: enabled
      - Add headers:
        - `HTTP-Referer = https://n8n.io`
        - `X-Title = IDX Invoice Reader`
      - Send body: enabled
      - Specify body: `JSON`
      - JSON body: `{{$json.payload}}`
      - Timeout: `30000`
    - Connect `Build Request -> OpenRouter Extract`.

12. **Add `Parse Invoice`**
    - Node type: **Code**.
    - Logic should:
      - read `choices[0].message.content`
      - trim it
      - remove code fences such as ```json
      - parse JSON
      - if a single object is returned, wrap it in an array
      - output one item per transaction
      - normalize fields:
        - `ticker`
        - `company_name`
        - `type`
        - `quantity`
        - `price`
        - `fee`
        - `total_amount`
        - `transaction_date`
        - `broker`
        - `confidence`
      - add `chat_id`, `filename`, and `total_in_batch`
      - on failure, emit one item with:
        - `error: true`
        - `error_message`
        - `chat_id`
        - `filename`
    - Connect `OpenRouter Extract -> Parse Invoice`.

13. **Add `Invoice Error?`**
    - Node type: **If**.
    - Condition: field `{{$json.error}}` **exists**.
    - Connect `Parse Invoice -> Invoice Error?`.

14. **Add `Reply Invoice Error`**
    - Node type: **Telegram**.
    - Configure:
      - Chat ID: `{{$json.chat_id}}`
      - Text: `❌ Failed to read invoice: {{$json.error_message}}` plus a retry hint
    - Connect the **true** output of `Invoice Error?` to `Reply Invoice Error`.

15. **Add `Format Confirmation`**
    - Node type: **Code**.
    - Use `$input.all()` to read all transaction items.
    - Build one summary message containing:
      - trade date
      - broker
      - source filename
      - one line per transaction
      - total amount
      - total fee
    - Generate a unique batch ID.
    - Store the batch in workflow static data:
      - `$getWorkflowStaticData('global')`
      - save under a key like `batch_<batchId>`
    - Build callback values:
      - `confirm_batch|<batchId>`
      - `cancel_batch|<batchId>`
    - Return:
      - `chat_id`
      - summary `text`
      - `reply_markup` button definitions
    - Connect the **false** output of `Invoice Error?` to `Format Confirmation`.

16. **Add `Send Confirmation`**
    - Node type: **Telegram**.
    - Configure:
      - Chat ID: `{{$json.chat_id}}`
      - Text: `{{$json.text}}`
    - Connect `Format Confirmation -> Send Confirmation`.

17. **Optionally add inline keyboard support**
    - The current exported workflow creates `reply_markup` in the code but does not map it into the Telegram send node.
    - To fully match the intended behavior, update `Send Confirmation` so it sends inline keyboard buttons using the generated callback data.
    - Without this addition, the confirmation text is sent, but the Save/Cancel buttons are not actually delivered.

18. **Optionally implement callback handling**
    - The `callback` branch from `Route Type1` is present but not wired.
    - To complete the approval flow, add downstream logic that:
      - reads `callback_data`
      - extracts `batchId`
      - loads `batch_<batchId>` from workflow static data
      - either persists all transactions on confirm or deletes/ignores them on cancel
      - answers the Telegram callback query
    - This is necessary if you want the buttons to perform actual actions.

19. **Add sticky notes if desired**
    - Add notes for:
      - overview/setup
      - message intake
      - file processing
      - AI extraction
      - confirmation

20. **Test the workflow**
    - Activate the workflow.
    - Send a PDF or photo of an Indonesian broker trade confirmation to the Telegram bot.
    - Confirm:
      - file is detected
      - file is downloaded
      - OpenRouter responds
      - transactions are summarized
      - Telegram receives either an error or a confirmation message

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Create a Telegram bot via BotFather, then use that bot token in an n8n Telegram credential. | Telegram / Bot setup |
| OpenRouter should be configured as an HTTP Header Auth credential using `Authorization: Bearer sk-or-...`. | OpenRouter API setup |
| n8n must be publicly reachable through HTTPS for Telegram webhooks to work correctly. Cloudflare Tunnel, ngrok, or a public server are suggested. | Deployment requirement |
| The default AI model is `google/gemini-2.5-flash-lite`, defined in the `Build Request` node. | Model customization |
| The exported workflow description says confirmation buttons are sent, but the actual `Send Confirmation` node does not currently transmit the generated button payload. | Functional gap to address |
| The callback branch exists in routing, but no callback-processing nodes are included in this workflow export. | Incomplete approval flow |
| Suggested downstream customization: replace or extend the output after confirmation with HTTP Request, Google Sheets, Airtable, or another destination node. | Storage/integration extension |