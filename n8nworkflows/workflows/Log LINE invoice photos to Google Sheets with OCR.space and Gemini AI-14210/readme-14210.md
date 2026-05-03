Log LINE invoice photos to Google Sheets with OCR.space and Gemini AI

https://n8nworkflows.xyz/workflows/log-line-invoice-photos-to-google-sheets-with-ocr-space-and-gemini-ai-14210


# Log LINE invoice photos to Google Sheets with OCR.space and Gemini AI

# 1. Workflow Overview

This workflow receives invoice photos sent to a LINE bot, extracts text from the image using OCR.space, uses Gemini to convert the OCR text into structured invoice data, appends the result to Google Sheets, and replies to the user in LINE with either a success or error message.

It is designed primarily for Japanese invoice intake and bookkeeping scenarios, especially for freelancers, small businesses, or accounting teams that want to avoid manual spreadsheet entry.

## 1.1 Webhook Reception and Immediate Acknowledgement

The workflow starts from a LINE Messaging API webhook. It immediately returns HTTP 200 to LINE to prevent webhook retries, then continues processing asynchronously inside the same workflow execution path.

## 1.2 Configuration and Message Type Filtering

A Set node centralizes the user-editable variables such as the LINE access token, OCR API key, spreadsheet ID, and sheet name. The workflow then checks whether the incoming LINE message is an image. Non-image messages are rejected with a LINE reply asking the user to send invoice data.

## 1.3 Image Retrieval and OCR

For image messages, the workflow downloads the binary image from LINE’s content API, converts it to Base64, and submits it to OCR.space. OCR.space returns extracted text from the invoice image.

## 1.4 AI-Based Invoice Parsing

The OCR text is passed to a Gemini-powered AI Agent. The prompt instructs the model to return only JSON with specific invoice fields such as invoice number, invoice date, due date, client name, subtotal, tax, total amount, currency, and notes. A code node then cleans and parses the model response.

## 1.5 Validation, Sheet Logging, and LINE Reply

The parsed JSON is checked for an error flag. If successful, the structured data is appended to Google Sheets and a success message is sent back via LINE. Otherwise, the user receives an error message indicating that the invoice could not be read.

---

# 2. Block-by-Block Analysis

## 2.1 Block: Workflow Guidance and Setup Notes

**Overview:**  
This block is purely informational. It contains sticky notes that explain the workflow purpose, setup steps, and the logical sections of the canvas.

**Nodes Involved:**  
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4
- Sticky Note5
- Sticky Note6

### Node Details

#### Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`; visual documentation node.
- **Configuration choices:** Contains a full explanation of how the workflow works, setup steps, and customization ideas.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version 1.
- **Edge cases or potential failure types:** None; does not affect execution.
- **Sub-workflow reference:** None.

#### Sticky Note1
- **Type and technical role:** Sticky note for the image intake block.
- **Configuration choices:** Labels the intake stage as “Image Intake”.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version 1.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky Note2
- **Type and technical role:** Sticky note for the OCR block.
- **Configuration choices:** Labels Base64 conversion and OCR extraction as “Text Extraction”.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version 1.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky Note3
- **Type and technical role:** Sticky note for the AI parsing block.
- **Configuration choices:** Labels Gemini parsing as “AI Parsing”.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version 1.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky Note4
- **Type and technical role:** Sticky note for logging and notification.
- **Configuration choices:** Labels Google Sheets append plus notification path as “Log & Notify”.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version 1.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky Note5
- **Type and technical role:** Sticky note for validation logic.
- **Configuration choices:** Labels the conditional branch as “Error Handling”.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version 1.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky Note6
- **Type and technical role:** Sticky note for LINE reply nodes.
- **Configuration choices:** Labels the user feedback block as “LINE Reply”.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version 1.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

---

## 2.2 Block: Webhook Reception and Immediate Response

**Overview:**  
This block receives webhook events from LINE and immediately acknowledges them with a JSON 200-style response. This is essential because LINE retries webhook deliveries if the endpoint does not respond quickly enough.

**Nodes Involved:**  
- Receive LINE Webhook
- Return 200 OK to LINE

### Node Details

#### Receive LINE Webhook
- **Type and technical role:** `n8n-nodes-base.webhook`; public HTTP entry point for LINE webhook events.
- **Configuration choices:**  
  - HTTP method: `POST`
  - Path: `line-invoice`
  - Response mode: `responseNode`, so the separate Respond to Webhook node controls the HTTP response.
  - Webhook ID is defined internally as `line-invoice-webhook`.
- **Key expressions or variables used:** Downstream nodes read:
  - `$('Receive LINE Webhook').item.json.body.events[0].message.type`
  - `$('Receive LINE Webhook').item.json.body.events[0].replyToken`
- **Input and output connections:**  
  - Outputs to `Return 200 OK to LINE`
  - Outputs to `Variables (API Key Management)`
- **Version-specific requirements:** Type version 2.
- **Edge cases or potential failure types:**  
  - If LINE sends a webhook payload without `events[0]`, downstream expressions will fail.
  - If the webhook URL is not correctly set in LINE Developers, no events will reach n8n.
  - If the workflow is inactive, the production webhook will not be available.
- **Sub-workflow reference:** None.

#### Return 200 OK to LINE
- **Type and technical role:** `n8n-nodes-base.respondToWebhook`; sends the HTTP response back to LINE.
- **Configuration choices:**  
  - Responds with JSON
  - Body: `{"status":"ok"}`
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Input from `Receive LINE Webhook`
  - No downstream execution dependency shown from this node
- **Version-specific requirements:** Type version 1.
- **Edge cases or potential failure types:**  
  - If omitted or delayed, LINE may retry the same event.
  - If webhook response mode is changed in the Webhook node, this node may become invalid or unused.
- **Sub-workflow reference:** None.

---

## 2.3 Block: Configuration and Image Message Filtering

**Overview:**  
This block stores reusable configuration values and checks whether the incoming LINE event contains an image message. Only image messages proceed to OCR; other message types trigger a polite rejection reply.

**Nodes Involved:**  
- Variables (API Key Management)
- Check Image Message
- LINE Reply - Not an Image

### Node Details

#### Variables (API Key Management)
- **Type and technical role:** `n8n-nodes-base.set`; injects configurable values into the execution data.
- **Configuration choices:** Assigns:
  - `LINE_CHANNEL_ACCESS_TOKEN`
  - `LINE_MESSAGE_ID`
  - `SPREADSHEET_ID`
  - `SHEET_NAME`
  - `OCR_SPACE_API_KEY`
- **Key expressions or variables used:**  
  - `LINE_MESSAGE_ID` is set to `=User Key` in the provided workflow, which appears to be a placeholder and not a working expression.
  - The node notes say this is the only node the user should configure.
- **Input and output connections:**  
  - Input from `Receive LINE Webhook`
  - Output to `Check Image Message`
- **Version-specific requirements:** Type version 3.4.
- **Edge cases or potential failure types:**  
  - As configured, `LINE_MESSAGE_ID` is not dynamically read from the webhook payload. This will break image download unless manually corrected.
  - `SPREADSHEET_ID` and `SHEET_NAME` are not actually used by the Google Sheets node in this exported version, which instead contains a hardcoded sheet URL/reference.
  - Placeholder values like `User Key` will cause authentication failures.
- **Sub-workflow reference:** None.

#### Check Image Message
- **Type and technical role:** `n8n-nodes-base.if`; checks whether the LINE message type equals `image`.
- **Configuration choices:**  
  - Condition compares `$('Receive LINE Webhook').item.json.body.events[0].message.type` to `"image"`.
  - True branch continues to image download.
  - False branch replies with a not-an-image message.
- **Key expressions or variables used:**  
  - `={{ $('Receive LINE Webhook').item.json.body.events[0].message.type }}`
- **Input and output connections:**  
  - Input from `Variables (API Key Management)`
  - True output to `Fetch Image from LINE`
  - False output to `LINE Reply - Not an Image`
- **Version-specific requirements:** Type version 2.
- **Edge cases or potential failure types:**  
  - If the webhook body shape differs, the expression can fail.
  - LINE may send multiple events in one webhook; this workflow only reads `events[0]`.
  - Non-message events such as follow/unfollow could also fail this expression if `message` is absent.
- **Sub-workflow reference:** None.

#### LINE Reply - Not an Image
- **Type and technical role:** `n8n-nodes-base.httpRequest`; sends a reply message back to LINE.
- **Configuration choices:**  
  - POST to `https://api.line.me/v2/bot/message/reply`
  - JSON body uses the webhook reply token
  - Message text: `請求書データを送ってください`
  - Headers include Bearer token and `Content-Type: application/json`
- **Key expressions or variables used:**  
  - Reply token from webhook: `{{ $('Receive LINE Webhook').item.json.body.events[0].replyToken }}`
  - Access token from Variables node: `{{ $('Variables (API Key Management)').item.json.LINE_CHANNEL_ACCESS_TOKEN }}`
- **Input and output connections:**  
  - Input from false branch of `Check Image Message`
  - No output
- **Version-specific requirements:** Type version 4.4.
- **Edge cases or potential failure types:**  
  - Invalid LINE token will return 401/403.
  - Reply tokens are single-use and time-limited; delayed execution can fail.
  - If the incoming event does not include a reply token, this request will fail.
- **Sub-workflow reference:** None.

---

## 2.4 Block: Image Retrieval and OCR

**Overview:**  
This block downloads the image content from LINE, converts it into Base64, and sends it to OCR.space for text extraction. It transforms an incoming chat image into machine-readable text.

**Nodes Involved:**  
- Fetch Image from LINE
- Convert Image to Base64
- OCR - Extract Text from Image

### Node Details

#### Fetch Image from LINE
- **Type and technical role:** `n8n-nodes-base.httpRequest`; retrieves image binary from LINE content API.
- **Configuration choices:**  
  - URL: `https://api-data.line.me/v2/bot/message/{{$json.LINE_MESSAGE_ID}}/content`
  - Sends `Authorization: Bearer <token>`
  - Response format set to `file`
- **Key expressions or variables used:**  
  - `{{$json.LINE_MESSAGE_ID}}`
  - `{{$json.LINE_CHANNEL_ACCESS_TOKEN}}`
- **Input and output connections:**  
  - Input from true branch of `Check Image Message`
  - Output to `Convert Image to Base64`
- **Version-specific requirements:** Type version 4.2.
- **Edge cases or potential failure types:**  
  - This node will fail if `LINE_MESSAGE_ID` is not correctly populated from the webhook event.
  - If the wrong LINE host is used, requests fail; note correctly says `api-data.line.me`.
  - Authorization or expired token problems will cause 401/403.
  - Large image payloads may affect execution time or memory.
- **Sub-workflow reference:** None.

#### Convert Image to Base64
- **Type and technical role:** `n8n-nodes-base.code`; converts binary file data into Base64 text.
- **Configuration choices:**  
  - Uses `this.helpers.getBinaryDataBuffer(0, 'data')`
  - Returns JSON with:
    - `data`: Base64 string
    - `mimeType`: `image/jpeg`
- **Key expressions or variables used:** JavaScript code only.
- **Input and output connections:**  
  - Input from `Fetch Image from LINE`
  - Output to `OCR - Extract Text from Image`
- **Version-specific requirements:** Type version 2.
- **Edge cases or potential failure types:**  
  - Assumes binary property name is `data`.
  - Hardcodes MIME type as `image/jpeg`, which may be inaccurate for PNG or other formats returned by LINE.
  - Missing binary data will throw an execution error.
- **Sub-workflow reference:** None.

#### OCR - Extract Text from Image
- **Type and technical role:** `n8n-nodes-base.httpRequest`; calls OCR.space API.
- **Configuration choices:**  
  - POST to `https://api.ocr.space/parse/image`
  - Multipart form-data request
  - Body parameters:
    - `base64Image = data:image/jpeg;base64,{{ $json.data }}`
    - `language = jpn`
    - `isTable = true`
    - `OCREngine = 2`
  - Header:
    - `apikey = {{ $('Variables (API Key Management)').item.json.OCR_SPACE_API_KEY }}`
- **Key expressions or variables used:**  
  - OCR API key from Variables node
  - Base64 payload from previous node
- **Input and output connections:**  
  - Input from `Convert Image to Base64`
  - Output to `AI Agent - Parse Invoice with Gemini`
- **Version-specific requirements:** Type version 4.4.
- **Edge cases or potential failure types:**  
  - Invalid OCR key or rate-limit issues.
  - OCR may return empty or poor-quality `ParsedResults`.
  - If `ParsedResults[0].ParsedText` is absent, the AI node expression will fail downstream.
  - OCR.space can struggle with low-resolution, skewed, or handwritten invoices.
- **Sub-workflow reference:** None.

---

## 2.5 Block: Gemini-Based Invoice Parsing

**Overview:**  
This block uses a Gemini chat model through an AI Agent to interpret raw OCR text and output structured invoice JSON. It defines the schema and business rules in Japanese to match the invoice use case.

**Nodes Involved:**  
- Google Gemini Chat Model
- AI Agent - Parse Invoice with Gemini

### Node Details

#### Google Gemini Chat Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`; language model provider node for the LangChain agent.
- **Configuration choices:**  
  - Minimal configuration shown in export
  - Serves as the AI language model connected to the AI Agent
- **Key expressions or variables used:** None shown.
- **Input and output connections:**  
  - AI language model connection to `AI Agent - Parse Invoice with Gemini`
- **Version-specific requirements:** Type version 1.
- **Edge cases or potential failure types:**  
  - Requires valid Gemini credentials in n8n.
  - Quota limits or model access restrictions may block execution.
  - If model configuration is incomplete, the agent cannot run.
- **Sub-workflow reference:** None.

#### AI Agent - Parse Invoice with Gemini
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`; prompts Gemini to extract structured invoice fields from OCR text.
- **Configuration choices:**  
  - Prompt type: defined text prompt
  - Prompt instructs the model to:
    - Read OCR text
    - Return JSON only
    - Follow rules for client name, numeric formatting, date formatting, and empty invoice number
    - Return `{ "error": true }` if the image is not an invoice
  - OCR input is injected from `ParsedResults[0].ParsedText`
- **Key expressions or variables used:**  
  - `{{ $('OCR - Extract Text from Image').item.json.ParsedResults[0].ParsedText }}`
- **Input and output connections:**  
  - Main input from `OCR - Extract Text from Image`
  - AI model input from `Google Gemini Chat Model`
  - Main output to `Parse & Format JSON`
- **Version-specific requirements:** Type version 3.1.
- **Edge cases or potential failure types:**  
  - Model may return Markdown fences despite instructions.
  - Model may output malformed JSON or hallucinated fields.
  - If OCR text is empty, parsing quality will be poor.
  - The agent may not reliably set `"error": true` for all non-invoice cases.
- **Sub-workflow reference:** None.

---

## 2.6 Block: JSON Parsing and Validation

**Overview:**  
This block normalizes the Gemini response into reliable JSON and checks whether the result should continue to Google Sheets or be treated as an error. It is the main data sanitation stage.

**Nodes Involved:**  
- Parse & Format JSON
- Error Check

### Node Details

#### Parse & Format JSON
- **Type and technical role:** `n8n-nodes-base.code`; strips code fences, parses JSON, normalizes numeric fields, and adds a processing timestamp in JST.
- **Configuration choices:**  
  - Reads `item.json.text || item.json.output || ''`
  - Removes ```json and ``` wrappers
  - Parses JSON
  - Converts `subtotal`, `tax_amount`, `total_amount` to numbers
  - Adds `processed_at` via `toLocaleString('ja-JP', { timeZone: 'Asia/Tokyo' })`
- **Key expressions or variables used:** JavaScript code only.
- **Input and output connections:**  
  - Input from `AI Agent - Parse Invoice with Gemini`
  - Output to `Error Check`
- **Version-specific requirements:** Type version 2.
- **Edge cases or potential failure types:**  
  - Important logic issue: on both success and catch branches, `error` is set to `false`. This means parsing failures are incorrectly treated as success by the next IF node.
  - If the AI returns `{ "error": true }`, that value is ignored and overwritten with `error: false`.
  - Missing AI response content can trigger JSON parse failures.
  - Numbers containing unexpected localized formatting may still parse incorrectly.
- **Sub-workflow reference:** None.

#### Error Check
- **Type and technical role:** `n8n-nodes-base.if`; routes success vs error branch based on `$json.error`.
- **Configuration choices:**  
  - Condition: `$json.error === false`
  - True branch goes to Google Sheets append
  - False branch goes to LINE error reply
- **Key expressions or variables used:**  
  - `={{ $json.error }}`
- **Input and output connections:**  
  - Input from `Parse & Format JSON`
  - True output to `Append to Google Sheets`
  - False output to `LINE Reply - Error`
- **Version-specific requirements:** Type version 2.3.
- **Edge cases or potential failure types:**  
  - Because the previous code node always sets `error: false`, the error branch is effectively unreachable in normal conditions.
  - If `error` is missing or non-boolean, strict type validation may affect branching.
- **Sub-workflow reference:** None.

---

## 2.7 Block: Google Sheets Logging and User Notification

**Overview:**  
This block appends the structured invoice data to Google Sheets and sends a success message back to the LINE user. It closes the main happy path of the workflow.

**Nodes Involved:**  
- Append to Google Sheets
- LINE Reply - Success
- LINE Reply - Error

### Node Details

#### Append to Google Sheets
- **Type and technical role:** `n8n-nodes-base.googleSheets`; appends a row to a Google Sheets document.
- **Configuration choices:**  
  - Operation: `append`
  - Document selected by direct URL/reference
  - Sheet selected as `gid=0` / cached name `シート1`
  - Mapping mode: define below
  - Column mapping:
    - 請求書番号 ← invoice number
    - 請求日 ← invoice date
    - 支払期限 ← due date
    - 取引先名 ← client name
    - 小計 ← subtotal
    - 消費税額 ← tax amount
    - 合計金額 ← total amount
    - 通貨 ← currency
    - 処理日時 ← processed timestamp
  - There is a schema entry for 備考, but it is marked removed and not actively mapped.
- **Key expressions or variables used:**  
  - Uses `$json` fields from `Parse & Format JSON`
- **Input and output connections:**  
  - Input from true branch of `Error Check`
  - Output to `LINE Reply - Success`
- **Version-specific requirements:** Type version 4.4.
- **Edge cases or potential failure types:**  
  - Requires Google Sheets OAuth2 credentials.
  - The Variables node includes `SPREADSHEET_ID` and `SHEET_NAME`, but they are not used here in the exported config.
  - Column names in the target sheet must match the configured schema/mapping.
  - Permission errors, missing sheet tabs, or changed headers can break append.
- **Sub-workflow reference:** None.

#### LINE Reply - Success
- **Type and technical role:** `n8n-nodes-base.httpRequest`; sends success confirmation to the LINE user.
- **Configuration choices:**  
  - POST to LINE reply endpoint
  - JSON body includes:
    - reply token from webhook
    - success message showing client name, invoice number, invoice date, and total amount
  - Authorization header uses LINE channel access token
- **Key expressions or variables used:**  
  - Reply token from `Receive LINE Webhook`
  - Fields from `Parse & Format JSON`
  - Bearer token from Variables node
- **Input and output connections:**  
  - Input from `Append to Google Sheets`
  - No output
- **Version-specific requirements:** Type version 4.4.
- **Edge cases or potential failure types:**  
  - Reply token expiry if upstream processing takes too long.
  - If Google Sheets append succeeds but reply fails, logging still happened but the user receives no confirmation.
  - Invalid LINE token or malformed JSON body will fail the request.
- **Sub-workflow reference:** None.

#### LINE Reply - Error
- **Type and technical role:** `n8n-nodes-base.httpRequest`; sends failure reply to the LINE user.
- **Configuration choices:**  
  - POST to LINE reply endpoint
  - JSON body message text: `請求書が読み取れませんんでした。`
  - Authorization header uses the LINE token from Variables node
- **Key expressions or variables used:**  
  - Reply token from webhook
  - LINE token from Variables node
- **Input and output connections:**  
  - Input from false branch of `Error Check`
  - No output
- **Version-specific requirements:** Type version 4.4.
- **Edge cases or potential failure types:**  
  - In current logic, likely never reached due to the `error: false` bug in the Code node.
  - Same reply token constraints as other LINE reply nodes.
  - The Japanese message contains a typo with duplicated ん.
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Google Gemini Chat Model | `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` | Provides Gemini model to the AI Agent |  | AI Agent - Parse Invoice with Gemini | ## AI Parsing\nGemini AI reads the raw OCR text and structures it into labelled invoice fields, returned as clean JSON. |
| Sticky Note | `n8n-nodes-base.stickyNote` | Canvas documentation and setup guidance |  |  | ### How it works\nSend a photo of any invoice to the LINE bot and the rest is handled automatically. The workflow picks up the image via LINE Messaging API, pushes it through OCR.space to pull out the raw text, then hands that text off to a Gemini AI agent. The agent identifies the key fields — invoice number, issue date, due date, vendor name, subtotal, tax, and total — and writes them as a new row in your Google Sheets ledger. A confirmation is sent back to LINE when everything is logged. If the image isn't an invoice, the bot replies with a friendly error message instead.\n### Setup steps\n1. Create a Messaging API channel in LINE Developers and grab a long-lived channel access token\n2. Get a free OCR.space API key at https://ocr.space/ocrapi/freekey (25,000 requests/month free)\n3. Get a free Gemini API key at https://aistudio.google.com (1,500 requests/day free)\n4. Connect your Google account via Google Sheets OAuth2 in n8n Credentials\n5. Open the Variables node — it's the only place you need to enter anything\n6. Paste your LINE token, OCR key, and Spreadsheet ID there\n7. Set your Webhook URL in LINE Developers to match your n8n instance URL\n8. Activate the workflow and send a test invoice photo to your bot\n### Customization\nSwap `SHEET_NAME` in the Variables node to file invoices by month or project. Edit the Gemini prompt to capture extra fields like bank details or itemized lines. Replace OCR.space with Google Cloud Vision if you need better accuracy on complex or handwritten layouts. |
| Receive LINE Webhook | `n8n-nodes-base.webhook` | Receives LINE Messaging API events |  | Return 200 OK to LINE; Variables (API Key Management) | ## Image Intake\nReceives the invoice image from LINE via Webhook, filters for image messages only, and downloads the file for processing. |
| Return 200 OK to LINE | `n8n-nodes-base.respondToWebhook` | Immediately acknowledges webhook delivery | Receive LINE Webhook |  | ## Image Intake\nReceives the invoice image from LINE via Webhook, filters for image messages only, and downloads the file for processing. |
| Check Image Message | `n8n-nodes-base.if` | Routes image vs non-image messages | Variables (API Key Management) | Fetch Image from LINE; LINE Reply - Not an Image | ## Image Intake\nReceives the invoice image from LINE via Webhook, filters for image messages only, and downloads the file for processing. |
| Variables (API Key Management) | `n8n-nodes-base.set` | Stores tokens, IDs, and user-configurable values | Receive LINE Webhook | Check Image Message | ## Image Intake\nReceives the invoice image from LINE via Webhook, filters for image messages only, and downloads the file for processing. |
| Fetch Image from LINE | `n8n-nodes-base.httpRequest` | Downloads image binary from LINE content API | Check Image Message | Convert Image to Base64 | ## Image Intake\nReceives the invoice image from LINE via Webhook, filters for image messages only, and downloads the file for processing. |
| Convert Image to Base64 | `n8n-nodes-base.code` | Converts downloaded binary into Base64 text | Fetch Image from LINE | OCR - Extract Text from Image | ## Text Extraction\nConverts the image to Base64 and sends it to OCR.space, which returns the raw text content of the invoice. |
| OCR - Extract Text from Image | `n8n-nodes-base.httpRequest` | Sends image to OCR.space and receives OCR text | Convert Image to Base64 | AI Agent - Parse Invoice with Gemini | ## Text Extraction\nConverts the image to Base64 and sends it to OCR.space, which returns the raw text content of the invoice. |
| AI Agent - Parse Invoice with Gemini | `@n8n/n8n-nodes-langchain.agent` | Extracts structured invoice JSON from OCR text | OCR - Extract Text from Image; Google Gemini Chat Model | Parse & Format JSON | ## AI Parsing\nGemini AI reads the raw OCR text and structures it into labelled invoice fields, returned as clean JSON. |
| Parse & Format JSON | `n8n-nodes-base.code` | Parses model output, normalizes fields, adds timestamp | AI Agent - Parse Invoice with Gemini | Error Check | ## Error Handling\nChecks whether the AI successfully parsed the invoice and routes to the appropriate success or error reply. |
| Error Check | `n8n-nodes-base.if` | Branches to success or error handling | Parse & Format JSON | Append to Google Sheets; LINE Reply - Error | ## Error Handling\nChecks whether the AI successfully parsed the invoice and routes to the appropriate success or error reply. |
| Append to Google Sheets | `n8n-nodes-base.googleSheets` | Appends invoice record to spreadsheet | Error Check | LINE Reply - Success | ## Log & Notify\nAppends the parsed invoice data to Google Sheets and sends a success confirmation back to the user on LINE.\n## LINE Reply\nSends a success message or a clear error message back to the user depending on the parsing result. |
| LINE Reply - Success | `n8n-nodes-base.httpRequest` | Sends success reply to LINE user | Append to Google Sheets |  | ## Log & Notify\nAppends the parsed invoice data to Google Sheets and sends a success confirmation back to the user on LINE.\n## LINE Reply\nSends a success message or a clear error message back to the user depending on the parsing result. |
| LINE Reply - Error | `n8n-nodes-base.httpRequest` | Sends parsing failure reply to LINE user | Error Check |  | ## Error Handling\nChecks whether the AI successfully parsed the invoice and routes to the appropriate success or error reply.\n## LINE Reply\nSends a success message or a clear error message back to the user depending on the parsing result. |
| LINE Reply - Not an Image | `n8n-nodes-base.httpRequest` | Sends guidance reply for non-image messages | Check Image Message |  | ## Image Intake\nReceives the invoice image from LINE via Webhook, filters for image messages only, and downloads the file for processing. |
| Sticky Note1 | `n8n-nodes-base.stickyNote` | Visual label for intake block |  |  | ## Image Intake\nReceives the invoice image from LINE via Webhook, filters for image messages only, and downloads the file for processing. |
| Sticky Note2 | `n8n-nodes-base.stickyNote` | Visual label for OCR block |  |  | ## Text Extraction\nConverts the image to Base64 and sends it to OCR.space, which returns the raw text content of the invoice. |
| Sticky Note3 | `n8n-nodes-base.stickyNote` | Visual label for AI block |  |  | ## AI Parsing\nGemini AI reads the raw OCR text and structures it into labelled invoice fields, returned as clean JSON. |
| Sticky Note4 | `n8n-nodes-base.stickyNote` | Visual label for logging block |  |  | ## Log & Notify\nAppends the parsed invoice data to Google Sheets and sends a success confirmation back to the user on LINE. |
| Sticky Note5 | `n8n-nodes-base.stickyNote` | Visual label for error-handling block |  |  | ## Error Handling\nChecks whether the AI successfully parsed the invoice and routes to the appropriate success or error reply. |
| Sticky Note6 | `n8n-nodes-base.stickyNote` | Visual label for reply block |  |  | ## LINE Reply\nSends a success message or a clear error message back to the user depending on the parsing result. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n**
   - Give it a name such as: `LINE Invoice Scanner — Auto-log to Google Sheets with OCR & Gemini AI`.

2. **Add a Webhook node**
   - Type: `Webhook`
   - Name: `Receive LINE Webhook`
   - Method: `POST`
   - Path: `line-invoice`
   - Response mode: `Using Respond to Webhook node`
   - Save the node.
   - This will become the URL you configure in LINE Developers.

3. **Add a Respond to Webhook node**
   - Type: `Respond to Webhook`
   - Name: `Return 200 OK to LINE`
   - Respond with: `JSON`
   - Response body:
     ```json
     {"status":"ok"}
     ```
   - Connect `Receive LINE Webhook` → `Return 200 OK to LINE`.

4. **Add a Set node for configuration**
   - Type: `Set`
   - Name: `Variables (API Key Management)`
   - Add these fields:
     - `LINE_CHANNEL_ACCESS_TOKEN` as string
     - `LINE_MESSAGE_ID` as string
     - `SPREADSHEET_ID` as string
     - `SHEET_NAME` as string
     - `OCR_SPACE_API_KEY` as string
   - Recommended values:
     - `LINE_CHANNEL_ACCESS_TOKEN`: your LINE Messaging API long-lived access token
     - `OCR_SPACE_API_KEY`: your OCR.space API key
     - `SPREADSHEET_ID`: the ID from the Google Sheets URL
     - `SHEET_NAME`: target sheet/tab name
   - Important correction: set `LINE_MESSAGE_ID` dynamically from the webhook:
     ```n8n
     ={{ $('Receive LINE Webhook').item.json.body.events[0].message.id }}
     ```
   - Connect `Receive LINE Webhook` → `Variables (API Key Management)`.

5. **Add an IF node to check the LINE message type**
   - Type: `IF`
   - Name: `Check Image Message`
   - Condition:
     - Left value:
       ```n8n
       ={{ $('Receive LINE Webhook').item.json.body.events[0].message.type }}
       ```
     - Operator: `equals`
     - Right value: `image`
   - Connect `Variables (API Key Management)` → `Check Image Message`.

6. **Add the non-image reply node**
   - Type: `HTTP Request`
   - Name: `LINE Reply - Not an Image`
   - Method: `POST`
   - URL: `https://api.line.me/v2/bot/message/reply`
   - Send headers: enabled
   - Headers:
     - `Authorization`:
       ```n8n
       =Bearer {{ $('Variables (API Key Management)').item.json.LINE_CHANNEL_ACCESS_TOKEN }}
       ```
     - `Content-Type`: `application/json`
   - Send body: enabled
   - Specify body: `JSON`
   - JSON body:
     ```json
     ={
       "replyToken": "{{ $('Receive LINE Webhook').item.json.body.events[0].replyToken }}",
       "messages": [
         {
           "type": "text",
           "text": "請求書データを送ってください"
         }
       ]
     }
     ```
   - Connect the **false** output of `Check Image Message` → `LINE Reply - Not an Image`.

7. **Add the LINE image download node**
   - Type: `HTTP Request`
   - Name: `Fetch Image from LINE`
   - Method: `GET`
   - URL:
     ```n8n
     =https://api-data.line.me/v2/bot/message/{{$json.LINE_MESSAGE_ID}}/content
     ```
   - Send headers: enabled
   - Header:
     - `Authorization`:
       ```n8n
       =Bearer {{$json.LINE_CHANNEL_ACCESS_TOKEN}}
       ```
   - Response format: `File`
   - Connect the **true** output of `Check Image Message` → `Fetch Image from LINE`.

8. **Add a Code node to convert binary image to Base64**
   - Type: `Code`
   - Name: `Convert Image to Base64`
   - JavaScript:
     ```javascript
     const binaryData = await this.helpers.getBinaryDataBuffer(0, 'data');
     const base64 = binaryData.toString('base64');

     return [{
       json: {
         data: base64,
         mimeType: 'image/jpeg'
       }
     }];
     ```
   - Connect `Fetch Image from LINE` → `Convert Image to Base64`.

9. **Add the OCR.space request node**
   - Type: `HTTP Request`
   - Name: `OCR - Extract Text from Image`
   - Method: `POST`
   - URL: `https://api.ocr.space/parse/image`
   - Content type: `Multipart Form-Data`
   - Send headers: enabled
   - Header:
     - `apikey`:
       ```n8n
       ={{ $('Variables (API Key Management)').item.json.OCR_SPACE_API_KEY }}
       ```
   - Send body: enabled
   - Body parameters:
     - `base64Image`:
       ```n8n
       =data:image/jpeg;base64,{{ $json.data }}
       ```
     - `language`: `jpn`
     - `isTable`: `true`
     - `OCREngine`: `2`
   - Connect `Convert Image to Base64` → `OCR - Extract Text from Image`.

10. **Create Gemini credentials in n8n**
    - Go to **Credentials**.
    - Create credentials appropriate for the Google Gemini chat model used by your n8n version.
    - Provide your Gemini API key from Google AI Studio.
    - Save the credential.

11. **Add a Google Gemini Chat Model node**
    - Type: `Google Gemini Chat Model`
    - Name: `Google Gemini Chat Model`
    - Select the Gemini credential created above.
    - Leave default options unless you want to tune model behavior.

12. **Add an AI Agent node**
    - Type: `AI Agent`
    - Name: `AI Agent - Parse Invoice with Gemini`
    - Prompt type: `Define below`
    - Prompt text:
      ```text
      以下は請求書をOCRで読み取ったテキストです。
      このテキストから請求書データを抽出してJSON形式のみで返してください。

      【OCRテキスト】
      {{ $('OCR - Extract Text from Image').item.json.ParsedResults[0].ParsedText }}

      【重要ルール】
      ・client_name: 請求書の右上に書いてある自分の会社名
      ・金額: 数字のみ（カンマ・円マーク・「円」不要）
      ・日付: YYYY-MM-DD形式
      ・請求書番号: ない場合は空文字

      【出力形式】
      {
        "error": false,
        "invoice_number": "請求書番号",
        "invoice_date": "YYYY-MM-DD",
        "due_date": "YYYY-MM-DD",
        "client_name": "右上に書いてある自分の会社名",
        "subtotal": 0,
        "tax_amount": 0,
        "total_amount": 0,
        "currency": "JPY",
        "notes": "備考欄の内容"
      }

      画像が請求書でない場合のみ:
      { "error": true }
      ```
    - Connect:
      - `OCR - Extract Text from Image` → `AI Agent - Parse Invoice with Gemini`
      - `Google Gemini Chat Model` → AI language model input of `AI Agent - Parse Invoice with Gemini`

13. **Add a Code node to parse and normalize AI output**
    - Type: `Code`
    - Name: `Parse & Format JSON`
    - Use this improved version so error handling works properly:
      ```javascript
      const items = $input.all();
      const results = [];

      for (const item of items) {
        try {
          const raw = item.json.text || item.json.output || '';

          const cleaned = raw
            .replace(/^```json\s*/i, '')
            .replace(/^```\s*/i, '')
            .replace(/\s*```$/i, '')
            .trim();

          const parsed = JSON.parse(cleaned);

          if (parsed.error === true) {
            results.push({
              json: {
                error: true,
                processed_at: new Date().toLocaleString('ja-JP', { timeZone: 'Asia/Tokyo' }),
                raw_response: raw
              }
            });
            continue;
          }

          const toNum = (val) =>
            Number(String(val || '0').replace(/[^0-9.]/g, '')) || 0;

          results.push({
            json: {
              error: false,
              invoice_number: parsed.invoice_number || '',
              invoice_date: parsed.invoice_date || '',
              due_date: parsed.due_date || '',
              client_name: parsed.client_name || '',
              subtotal: toNum(parsed.subtotal),
              tax_amount: toNum(parsed.tax_amount),
              total_amount: toNum(parsed.total_amount),
              currency: parsed.currency || 'JPY',
              notes: parsed.notes || '',
              processed_at: new Date().toLocaleString('ja-JP', { timeZone: 'Asia/Tokyo' })
            }
          });

        } catch (e) {
          results.push({
            json: {
              error: true,
              error_message: `JSON parse failed: ${e.message}`,
              raw_response: item.json.text || item.json.output || 'No response',
              processed_at: new Date().toLocaleString('ja-JP', { timeZone: 'Asia/Tokyo' })
            }
          });
        }
      }

      return results;
      ```
   - Connect `AI Agent - Parse Invoice with Gemini` → `Parse & Format JSON`.

14. **Add an IF node for result validation**
    - Type: `IF`
    - Name: `Error Check`
    - Condition:
      - Left value:
        ```n8n
        ={{ $json.error }}
        ```
      - Operator: `equals`
      - Right value: `false`
    - Connect `Parse & Format JSON` → `Error Check`.

15. **Prepare Google Sheets credentials**
    - In n8n Credentials, create `Google Sheets OAuth2 API` credentials.
    - Sign in with the Google account that can edit the destination spreadsheet.

16. **Prepare the destination spreadsheet**
    - Create a Google Sheet.
    - Add a sheet tab, for example `シート1` or your monthly tab.
    - Ensure headers exist in row 1. Based on the actual node mapping, use:
      - `請求書番号`
      - `請求日`
      - `支払期限`
      - `取引先名`
      - `小計`
      - `消費税額`
      - `合計金額`
      - `通貨`
      - `備考` optionally
      - `処理日時`

17. **Add a Google Sheets node**
    - Type: `Google Sheets`
    - Name: `Append to Google Sheets`
    - Credential: your Google Sheets OAuth2 credential
    - Operation: `Append`
    - Select the spreadsheet
    - Select the target sheet
    - Set mapping mode to manually define columns
    - Map:
      - `請求書番号` → `={{ $json.invoice_number }}`
      - `請求日` → `={{ $json.invoice_date }}`
      - `支払期限` → `={{ $json.due_date }}`
      - `取引先名` → `={{ $json.client_name }}`
      - `小計` → `={{ $json.subtotal }}`
      - `消費税額` → `={{ $json.tax_amount }}`
      - `合計金額` → `={{ $json.total_amount }}`
      - `通貨` → `={{ $json.currency }}`
      - `備考` → `={{ $json.notes }}` if you want to store notes
      - `処理日時` → `={{ $json.processed_at }}`
    - Connect the **true** output of `Error Check` → `Append to Google Sheets`.

18. **Add the success reply node**
    - Type: `HTTP Request`
    - Name: `LINE Reply - Success`
    - Method: `POST`
    - URL: `https://api.line.me/v2/bot/message/reply`
    - Send headers: enabled
    - Headers:
      - `Authorization`:
        ```n8n
        =Bearer {{ $('Variables (API Key Management)').item.json.LINE_CHANNEL_ACCESS_TOKEN }}
        ```
      - `Content-Type`: `application/json`
    - JSON body:
      ```json
      ={
        "replyToken": "{{ $('Receive LINE Webhook').item.json.body.events[0].replyToken }}",
        "messages": [
          {
            "type": "text",
            "text": "✅ スプレッドシートに登録しました！\n\n取引先: {{ $('Parse & Format JSON').item.json.client_name }}\n請求書番号: {{ $('Parse & Format JSON').item.json.invoice_number }}\n請求日: {{ $('Parse & Format JSON').item.json.invoice_date }}\n合計金額: ¥{{ $('Parse & Format JSON').item.json.total_amount }}"
          }
        ]
      }
      ```
    - Connect `Append to Google Sheets` → `LINE Reply - Success`.

19. **Add the error reply node**
    - Type: `HTTP Request`
    - Name: `LINE Reply - Error`
    - Method: `POST`
    - URL: `https://api.line.me/v2/bot/message/reply`
    - Send headers: enabled
    - Headers:
      - `Authorization`:
        ```n8n
        =Bearer {{ $('Variables (API Key Management)').item.json.LINE_CHANNEL_ACCESS_TOKEN }}
        ```
      - `Content-Type`: `application/json`
    - JSON body:
      ```json
      ={
        "replyToken": "{{ $('Receive LINE Webhook').item.json.body.events[0].replyToken }}",
        "messages": [
          {
            "type": "text",
            "text": "請求書が読み取れませんでした。"
          }
        ]
      }
      ```
    - Connect the **false** output of `Error Check` → `LINE Reply - Error`.

20. **Optionally add sticky notes**
    - Add notes for:
      - Image Intake
      - Text Extraction
      - AI Parsing
      - Error Handling
      - Log & Notify
      - LINE Reply
    - These are optional and do not affect execution.

21. **Configure LINE Developers webhook**
    - In LINE Developers, open your Messaging API channel.
    - Set the Webhook URL to your production webhook URL from n8n.
    - Enable webhook usage.

22. **Activate the workflow**
    - The workflow must be active for the production webhook URL to work.

23. **Test with a real image**
    - Send an invoice photo to the LINE bot.
    - Confirm:
      - LINE receives immediate 200 OK
      - The image is downloaded
      - OCR text is extracted
      - Gemini returns JSON
      - A row is appended to Google Sheets
      - The user receives a success reply

24. **Recommended hardening**
    - Add validation for missing `events[0]`
    - Handle multiple LINE events instead of only the first one
    - Use `message.id` directly from the webhook rather than a manually filled variable
    - Use `SPREADSHEET_ID` and `SHEET_NAME` dynamically in the Google Sheets node if you want the Variables node to fully control sheet routing
    - Add OCR response checks before calling Gemini
    - Consider pushing the final LINE message via `push` API if processing time may exceed reply token limits

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Free OCR.space API key signup | https://ocr.space/ocrapi/freekey |
| Gemini API key generation via Google AI Studio | https://aistudio.google.com |
| LINE Developers is required to create the Messaging API channel and configure the webhook URL | LINE Developers console |
| The workflow description targets Japanese invoice handling for freelancers, small businesses, and accounting teams | Project context |
| Customization idea: change `SHEET_NAME` to organize invoices by month or project | Workflow design note |
| Customization idea: edit the Gemini prompt to capture extra fields such as bank details or line items | Workflow design note |
| Alternative OCR engine suggestion: replace OCR.space with Google Cloud Vision for better handling of complex or handwritten layouts | Architecture note |

## Important implementation observations
| Note Content | Context or Link |
|---|---|
| `LINE_MESSAGE_ID` is incorrectly configured as a placeholder in the provided workflow and should be mapped from `body.events[0].message.id` | Required correction |
| `SPREADSHEET_ID` and `SHEET_NAME` are presented as configurable variables, but the exported Google Sheets node is hardcoded to a specific spreadsheet and sheet selection | Required correction |
| The current `Parse & Format JSON` code forces `error: false` even when parsing fails or Gemini says the image is not an invoice, which prevents the error branch from working correctly | Logic bug |
| The `備考` field exists in the AI output and sheet schema, but it is not actively mapped in the current Google Sheets configuration | Data mapping gap |