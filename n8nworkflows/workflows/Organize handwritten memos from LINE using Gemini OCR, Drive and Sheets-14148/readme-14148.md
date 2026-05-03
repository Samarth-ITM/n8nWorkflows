Organize handwritten memos from LINE using Gemini OCR, Drive and Sheets

https://n8nworkflows.xyz/workflows/organize-handwritten-memos-from-line-using-gemini-ocr--drive-and-sheets-14148


# Organize handwritten memos from LINE using Gemini OCR, Drive and Sheets

# 1. Workflow Overview

This workflow receives messages from LINE, accepts only image messages, downloads the image, stores it in Google Drive, sends it to Gemini for OCR-style extraction and summarization, parses the AI output into structured JSON, checks whether the OCR failed, then stores the result in Google Sheets under a category-specific sheet. Finally, it sends a completion message back to the LINE user.

Typical use cases:
- Organizing handwritten memos sent from a mobile phone
- Converting handwritten notes into searchable structured records
- Automatically categorizing and archiving memo images with metadata

## 1.1 LINE Input Reception and Validation
The workflow starts with a webhook that receives LINE events. It injects environment-like configuration values, validates whether the incoming LINE message is an image, and replies immediately either with a processing message or with guidance if the message is not an image.

## 1.2 Image Retrieval and Persistence
If the incoming message is an image, the workflow downloads the image binary from the LINE content API, uploads the file to Google Drive, and restores the binary payload so it can still be used downstream by the AI node.

## 1.3 OCR and AI Structuring
The restored image is passed to a Gemini chat model through an LLM chain node configured with an OCR-oriented prompt. The AI is instructed to output only JSON containing `title`, `category`, `summary`, and `tags`.

## 1.4 JSON Parsing and OCR Failure Handling
Because LLM outputs can be malformed, a Code node removes markdown wrappers, extracts the JSON object, parses it safely, applies defaults, and sanitizes fields. A conditional check then detects OCR failure based on the summary text. If OCR failed, the workflow stops storage and notifies the user.

## 1.5 Google Sheets Category Management
If OCR succeeded, the workflow queries Google Sheets metadata via the Sheets API, checks whether a sheet named after the detected category already exists, and branches accordingly. If the category sheet does not exist, it creates it.

## 1.6 Data Preparation, Storage, and User Notification
The workflow prepares row data including date, title, summary, Drive URL, and tags, appends the row to the correct category sheet, and sends a final LINE push notification confirming the save.

---

# 2. Block-by-Block Analysis

## 2.1 LINE Input Reception and Validation

**Overview:**  
This block receives webhook calls from LINE, loads token and spreadsheet configuration, verifies that the incoming message is an image, and responds immediately. It prevents unsupported message types from entering the expensive OCR path.

**Nodes Involved:**  
- LINE_Receive_Webhook
- Config_Set_Environment
- LINE_Check_MessageType
- LINE_Reply_Processing
- LINE_Reply_NoImage

### Node: LINE_Receive_Webhook
- **Type and technical role:** `n8n-nodes-base.webhook`; public HTTP entry point for LINE webhook events.
- **Configuration choices:**  
  - HTTP method: `POST`
  - Path is dynamically set but resolves to a fixed UUID-like string.
- **Key expressions or variables used:**  
  - Webhook body is later accessed through `body.events[0]`.
- **Input and output connections:**  
  - No input; entry point
  - Outputs to `Config_Set_Environment`
- **Version-specific requirements:**  
  - Uses `typeVersion` 2.1; standard webhook behavior for recent n8n versions.
- **Edge cases or potential failure types:**  
  - LINE signature validation is not implemented in this workflow
  - If LINE sends multiple events in one webhook, only `events[0]` is used
  - Missing `body.events[0]` will break downstream expressions
- **Sub-workflow reference:** None

### Node: Config_Set_Environment
- **Type and technical role:** `n8n-nodes-base.set`; injects static configuration values into the item.
- **Configuration choices:**  
  - Sets:
    - `LINE_ACCESS_TOKEN`
    - `GOOGLE_SHEETS_ID`
  - Placeholder values must be replaced manually.
- **Key expressions or variables used:**  
  - `LINE_ACCESS_TOKEN = YOUR_ACCESS_TOKEN`
  - `GOOGLE_SHEETS_ID = YOUR_GOOGLE_SHEETS_ID`
- **Input and output connections:**  
  - Input from `LINE_Receive_Webhook`
  - Output to `LINE_Check_MessageType`
- **Version-specific requirements:**  
  - `typeVersion` 3.4
- **Edge cases or potential failure types:**  
  - If placeholders are not replaced, all LINE and Sheets calls will fail
  - Storing secrets in a Set node is functional but not ideal for production security
- **Sub-workflow reference:** None

### Node: LINE_Check_MessageType
- **Type and technical role:** `n8n-nodes-base.if`; validates whether the LINE message type is `image`.
- **Configuration choices:**  
  - Main condition checks:
    - `$('LINE_Receive_Webhook').item.json.body.events[0].message.type == "image"`
  - There is a second empty equality condition (`"" == ""`) that is always true and functionally unnecessary.
- **Key expressions or variables used:**  
  - `{{ $('LINE_Receive_Webhook').item.json.body.events[0].message.type }}`
- **Input and output connections:**  
  - Input from `Config_Set_Environment`
  - True output to `LINE_Reply_Processing`
  - False output to `LINE_Reply_NoImage`
- **Version-specific requirements:**  
  - `typeVersion` 2.3, conditions format version 3
- **Edge cases or potential failure types:**  
  - If `message` is absent, expression evaluation may fail
  - If LINE sends non-message events, the workflow may error before reaching the false branch
- **Sub-workflow reference:** None

### Node: LINE_Reply_Processing
- **Type and technical role:** `n8n-nodes-base.httpRequest`; sends an immediate reply to LINE indicating processing has started.
- **Configuration choices:**  
  - POST to `https://api.line.me/v2/bot/message/reply`
  - JSON body includes `replyToken` and text `"Processing…"`
  - Headers include Bearer token and `Content-type: application/json`
- **Key expressions or variables used:**  
  - Reply token: `{{$node['LINE_Receive_Webhook'].json.body.events[0].replyToken }}`
  - Access token: `{{$node["Config_Set_Environment"].json.LINE_ACCESS_TOKEN }}`
- **Input and output connections:**  
  - Input from `LINE_Check_MessageType` true branch
  - Output to `LINE_Download_ImageContent`
- **Version-specific requirements:**  
  - `typeVersion` 4.3
- **Edge cases or potential failure types:**  
  - LINE reply tokens expire quickly; delays before this node can cause failure
  - Invalid or missing token results in 401/403
  - If the same reply token is reused, LINE returns an error
- **Sub-workflow reference:** None

### Node: LINE_Reply_NoImage
- **Type and technical role:** `n8n-nodes-base.httpRequest`; replies when the incoming message is not an image.
- **Configuration choices:**  
  - POST to LINE reply endpoint
  - Sends text: `"If you send a handwritten memo, I will summarize its contents."`
- **Key expressions or variables used:**  
  - Same `replyToken` and access token pattern as above
- **Input and output connections:**  
  - Input from `LINE_Check_MessageType` false branch
  - No downstream nodes
- **Version-specific requirements:**  
  - `typeVersion` 4.3
- **Edge cases or potential failure types:**  
  - Same token-expiry and auth concerns as `LINE_Reply_Processing`
- **Sub-workflow reference:** None

---

## 2.2 Image Retrieval and Persistence

**Overview:**  
This block retrieves the uploaded image from LINE, stores it in Drive for persistence, then restores the binary content so the same image can be passed to the AI OCR stage.

**Nodes Involved:**  
- LINE_Download_ImageContent
- Drive_Upload_Image
- Binary_Restore_From_LINE

### Node: LINE_Download_ImageContent
- **Type and technical role:** `n8n-nodes-base.httpRequest`; downloads the raw image file from LINE content API.
- **Configuration choices:**  
  - GET request to:
    `https://api-data.line.me/v2/bot/message/{messageId}/content`
  - Response format: file/binary
  - Authorization header with LINE bearer token
- **Key expressions or variables used:**  
  - Message ID: `{{ $('LINE_Receive_Webhook').item.json.body.events[0].message.id }}`
  - Access token: `{{ $('Config_Set_Environment').item.json.LINE_ACCESS_TOKEN }}`
- **Input and output connections:**  
  - Input from `LINE_Reply_Processing`
  - Output to `Drive_Upload_Image`
- **Version-specific requirements:**  
  - `typeVersion` 4.3
- **Edge cases or potential failure types:**  
  - Invalid token or message ID causes 401/404
  - Binary response may fail if LINE content is unavailable or expired
  - Large images may increase execution time
- **Sub-workflow reference:** None

### Node: Drive_Upload_Image
- **Type and technical role:** `n8n-nodes-base.googleDrive`; uploads the image to Google Drive.
- **Configuration choices:**  
  - File name: `{LINE message id}.jpg`
  - Drive: `My Drive`
  - Folder: `LINE_PIC`
  - Uses Google Drive OAuth2 credentials
- **Key expressions or variables used:**  
  - Name: `={{ $('LINE_Receive_Webhook').item.json.body.events[0].message.id }}.jpg`
- **Input and output connections:**  
  - Input from `LINE_Download_ImageContent`
  - Output to `Binary_Restore_From_LINE`
- **Version-specific requirements:**  
  - `typeVersion` 3
  - Requires valid Google Drive OAuth2 credentials and existing target folder
- **Edge cases or potential failure types:**  
  - OAuth scopes may be insufficient
  - Folder may not exist or be inaccessible
  - If binary property is missing, upload will fail
  - MIME type mismatch may still upload but with weaker metadata quality
- **Sub-workflow reference:** None

### Node: Binary_Restore_From_LINE
- **Type and technical role:** `n8n-nodes-base.code`; reattaches the original binary data after the Google Drive node, which otherwise passes mainly JSON metadata.
- **Configuration choices:**  
  - Copies binary from `LINE_Download_ImageContent` into current item
- **Key expressions or variables used:**  
  - `item.binary = $node["LINE_Download_ImageContent"].binary;`
- **Input and output connections:**  
  - Input from `Drive_Upload_Image`
  - Output to `AI_OCR_Analyze_Image`
- **Version-specific requirements:**  
  - `typeVersion` 2
- **Edge cases or potential failure types:**  
  - If `LINE_Download_ImageContent` did not produce binary data, downstream AI image input will fail
  - Assumes single-item execution context
- **Sub-workflow reference:** None

---

## 2.3 OCR and AI Structuring

**Overview:**  
This block submits the image to Gemini with a tightly constrained OCR-and-summarization prompt. The AI is asked to extract handwritten content and return only valid JSON.

**Nodes Involved:**  
- AI_Model_Gemini
- AI_OCR_Analyze_Image

### Node: AI_Model_Gemini
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`; language model connector used by the LLM chain.
- **Configuration choices:**  
  - Temperature: `0`
  - Top P: `1`
  - Max output tokens: `2048`
  - Uses Google Gemini (PaLM) API credentials
- **Key expressions or variables used:**  
  - No dynamic expression in parameters
- **Input and output connections:**  
  - No main input/output
  - Connected to `AI_OCR_Analyze_Image` through `ai_languageModel`
- **Version-specific requirements:**  
  - `typeVersion` 1
  - Requires n8n LangChain-compatible node support and valid Gemini credential setup
- **Edge cases or potential failure types:**  
  - Quota/rate limits
  - Vision capability depends on the selected model behind the credential/node version
  - Authentication issues or model deprecation can break execution
- **Sub-workflow reference:** None

### Node: AI_OCR_Analyze_Image
- **Type and technical role:** `@n8n/n8n-nodes-langchain.chainLlm`; orchestrates the prompt and sends the binary image to the Gemini model.
- **Configuration choices:**  
  - Prompt instructs model to:
    - act as a high-precision OCR engine
    - return a JSON object with `title`, `category`, `summary`, `tags`
    - emit a specific fallback JSON when unreadable
    - output pure JSON only
  - Uses `HumanMessagePromptTemplate` with `imageBinary`
- **Key expressions or variables used:**  
  - Prompt is static, but crucially defines the failure marker:
    - `"summary": "The text could not be recognized."`
- **Input and output connections:**  
  - Input from `Binary_Restore_From_LINE`
  - Output to `Data_Parse_OCR_JSON`
  - Receives model connector from `AI_Model_Gemini`
- **Version-specific requirements:**  
  - `typeVersion` 1.7
  - Requires image-capable model support in the linked Gemini node
- **Edge cases or potential failure types:**  
  - LLM may still output markdown code fences or explanatory text despite instructions
  - Handwriting quality, blur, low light, or non-supported language may reduce OCR quality
  - If binary is missing or malformed, model input may fail
- **Sub-workflow reference:** None

---

## 2.4 JSON Parsing and OCR Failure Handling

**Overview:**  
This block normalizes the AI output into reliable structured fields and decides whether processing should continue. It acts as the main guardrail against malformed LLM output and unreadable images.

**Nodes Involved:**  
- Data_Parse_OCR_JSON
- AI_Check_OCR_Failure
- LINE_Push_NoSummary

### Node: Data_Parse_OCR_JSON
- **Type and technical role:** `n8n-nodes-base.code`; parses the model response, applies fallback defaults, sanitizes tags, and preserves binary data.
- **Configuration choices:**  
  - Reads `$json.text`
  - Removes `````json` wrappers and plain ``` fences
  - Extracts the first JSON object found via regex
  - Attempts `JSON.parse`
  - On parse failure, builds fallback object:
    - empty title/category
    - summary = first 100 chars of raw output
    - empty tags
  - Applies defaults:
    - `title = "No title"`
    - `category = "Uncategorized"`
    - `summary = "No summary"`
    - `tags = []`
  - Limits tags to string values and first 3 entries
  - Merges parsed fields into existing item
  - Restores binary from input
- **Key expressions or variables used:**  
  - Source text: `$json["text"]`
- **Input and output connections:**  
  - Input from `AI_OCR_Analyze_Image`
  - Output to `AI_Check_OCR_Failure`
- **Version-specific requirements:**  
  - `typeVersion` 2
- **Edge cases or potential failure types:**  
  - Regex may capture unintended braces if the model returns complex extra text
  - Invalid Unicode or malformed JSON still falls back rather than crashing
  - If AI returns nested/nonstandard structures, some data may be discarded
- **Sub-workflow reference:** None

### Node: AI_Check_OCR_Failure
- **Type and technical role:** `n8n-nodes-base.if`; detects OCR failure by comparing summary text to the exact failure string.
- **Configuration choices:**  
  - Condition:
    - `$json.summary == "The text could not be recognized."`
- **Key expressions or variables used:**  
  - `={{ $json.summary }}`
- **Input and output connections:**  
  - Input from `Data_Parse_OCR_JSON`
  - True output to `LINE_Push_NoSummary`
  - False output to `Sheets_Get_Metadata`
- **Version-specific requirements:**  
  - `typeVersion` 2.3
- **Edge cases or potential failure types:**  
  - Detection is exact-string-based, so any variation from the AI prompt may bypass failure handling
  - Parse fallback summaries will not trigger this node unless they match exactly
- **Sub-workflow reference:** None

### Node: LINE_Push_NoSummary
- **Type and technical role:** `n8n-nodes-base.httpRequest`; sends a push message to the user when OCR could not read the image.
- **Configuration choices:**  
  - POST to `https://api.line.me/v2/bot/message/push`
  - Message explains that the image could not be read and suggests retaking in better conditions
- **Key expressions or variables used:**  
  - User ID: `{{ $('LINE_Receive_Webhook').item.json.body.events[0].source.userId }}`
  - Access token: `{{ $('Config_Set_Environment').item.json.LINE_ACCESS_TOKEN }}`
- **Input and output connections:**  
  - Input from `AI_Check_OCR_Failure` true branch
  - No downstream node
- **Version-specific requirements:**  
  - `typeVersion` 4.3
- **Edge cases or potential failure types:**  
  - Push messaging may require proper LINE bot permissions
  - If `source.userId` is unavailable for the event type, push fails
- **Sub-workflow reference:** None

---

## 2.5 Google Sheets Category Management

**Overview:**  
This block determines whether the OCR-generated category already has a corresponding sheet tab in the target spreadsheet. If not, it creates one so data can be stored in a category-specific location.

**Nodes Involved:**  
- Sheets_Get_Metadata
- Sheets_Check_Category_Exists
- Sheets_Branch_Category
- Sheets_Create_Category

### Node: Sheets_Get_Metadata
- **Type and technical role:** `n8n-nodes-base.httpRequest`; fetches spreadsheet metadata directly from the Google Sheets REST API.
- **Configuration choices:**  
  - GET request to:
    `https://sheets.googleapis.com/v4/spreadsheets/{GOOGLE_SHEETS_ID}`
  - Uses predefined Google OAuth2 credential
- **Key expressions or variables used:**  
  - Spreadsheet ID from config:
    `{{ $('Config_Set_Environment').item.json.GOOGLE_SHEETS_ID }}`
- **Input and output connections:**  
  - Input from `AI_Check_OCR_Failure` false branch
  - Output to `Sheets_Check_Category_Exists`
- **Version-specific requirements:**  
  - `typeVersion` 4.3
  - Requires Sheets metadata access scope
- **Edge cases or potential failure types:**  
  - Invalid spreadsheet ID
  - Insufficient OAuth scopes
  - Spreadsheet inaccessible to the credential owner
- **Sub-workflow reference:** None

### Node: Sheets_Check_Category_Exists
- **Type and technical role:** `n8n-nodes-base.code`; checks if a sheet tab name matches the OCR category.
- **Configuration choices:**  
  - Reads category from `Data_Parse_OCR_JSON`
  - Builds array of existing sheet names from API metadata
  - Outputs:
    - `category`
    - `sheetExists`
    - `sheetNames`
- **Key expressions or variables used:**  
  - `const category = $('Data_Parse_OCR_JSON').first().json.category;`
- **Input and output connections:**  
  - Input from `Sheets_Get_Metadata`
  - Output to `Sheets_Branch_Category`
- **Version-specific requirements:**  
  - `typeVersion` 2
- **Edge cases or potential failure types:**  
  - Category may contain characters invalid for Google Sheets tab names
  - Category may be too long
  - Case-sensitive matching means near-duplicates can create multiple tabs
- **Sub-workflow reference:** None

### Node: Sheets_Branch_Category
- **Type and technical role:** `n8n-nodes-base.if`; branches depending on whether the category sheet already exists.
- **Configuration choices:**  
  - Condition:
    - `$json.sheetExists == true`
- **Key expressions or variables used:**  
  - `={{ $json.sheetExists }}`
- **Input and output connections:**  
  - Input from `Sheets_Check_Category_Exists`
  - True output to `Sheets_Append_Row`
  - False output to `Sheets_Create_Category`
- **Version-specific requirements:**  
  - `typeVersion` 2.3
- **Edge cases or potential failure types:**  
  - If the previous node returns malformed output, branch logic can fail
- **Sub-workflow reference:** None

### Node: Sheets_Create_Category
- **Type and technical role:** `n8n-nodes-base.googleSheets`; creates a new sheet tab in the spreadsheet.
- **Configuration choices:**  
  - Operation: `create`
  - Document: fixed spreadsheet `OCR_data`
  - Title: OCR-derived category
- **Key expressions or variables used:**  
  - `={{ $('Sheets_Check_Category_Exists').item.json.category }}`
- **Input and output connections:**  
  - Input from `Sheets_Branch_Category` false branch
  - Output to `Data_Prepare_Sheet_Row`
- **Version-specific requirements:**  
  - `typeVersion` 4.7
  - Requires Google Sheets OAuth2 credential with edit access
- **Edge cases or potential failure types:**  
  - Invalid sheet title characters
  - Duplicate title race condition if two executions create the same category simultaneously
  - Spreadsheet permissions issues
- **Sub-workflow reference:** None

---

## 2.6 Data Preparation, Storage, and User Notification

**Overview:**  
This block builds the final row payload, appends it to the appropriate category sheet, and notifies the user that the memo has been stored successfully.

**Nodes Involved:**  
- Data_Prepare_Sheet_Row
- Sheets_Append_Row
- LINE_Push_Completion_Message

### Node: Data_Prepare_Sheet_Row
- **Type and technical role:** `n8n-nodes-base.set`; prepares normalized row values after a new sheet was created.
- **Configuration choices:**  
  - Sets:
    - `date`
    - `title`
    - `summary`
    - `webUrl`
    - `tags`
- **Key expressions or variables used:**  
  - `{{$now.format("yyyy-MM-dd HH:mm:ss")}}`
  - `{{ $node['Data_Parse_OCR_JSON'].json.title }}`
  - `{{ $node['Data_Parse_OCR_JSON'].json.summary }}`
  - `{{ $node['Drive_Upload_Image'].json.webViewLink }}`
  - `{{ $node['Data_Parse_OCR_JSON'].json.tags.join(',') }}`
- **Input and output connections:**  
  - Input from `Sheets_Create_Category`
  - Output to `Sheets_Append_Row`
- **Version-specific requirements:**  
  - `typeVersion` 3.4
- **Edge cases or potential failure types:**  
  - This node is only used after sheet creation; if category already exists, append uses direct expressions instead
  - Missing `webViewLink` or missing tags array can affect output quality
- **Sub-workflow reference:** None

### Node: Sheets_Append_Row
- **Type and technical role:** `n8n-nodes-base.googleSheets`; appends memo metadata into the category sheet.
- **Configuration choices:**  
  - Operation: `append`
  - Document: fixed spreadsheet `OCR_data`
  - Sheet name: category from `Sheets_Check_Category_Exists`
  - Explicit mapping for columns:
    - `date`
    - `title`
    - `summary`
    - `webUrl`
    - `tags`
  - Uses expressions directly rather than relying on incoming item shape
- **Key expressions or variables used:**  
  - Date: `{{$now.format("yyyy-MM-dd HH:mm:ss")}}`
  - Tags: `{{ $node['Data_Parse_OCR_JSON'].json.tags.join(',') }}`
  - Title: `{{ $node['Data_Parse_OCR_JSON'].json.title }}`
  - Web URL: `{{ $('Drive_Upload_Image').item.json.webViewLink }}`
  - Summary: `{{ $node['Data_Parse_OCR_JSON'].json.summary }}`
  - Sheet name: `={{ $('Sheets_Check_Category_Exists').item.json.category }}`
- **Input and output connections:**  
  - Inputs from:
    - `Sheets_Branch_Category` true branch
    - `Data_Prepare_Sheet_Row`
  - Output to `LINE_Push_Completion_Message`
- **Version-specific requirements:**  
  - `typeVersion` 4.7
  - Spreadsheet should contain headers matching the mapped schema
- **Edge cases or potential failure types:**  
  - Newly created sheet may not yet have expected headers depending on Google Sheets node behavior
  - Invalid sheet name or missing permissions will fail append
  - Mixed upstream shapes are tolerated because expressions reference other nodes directly
- **Sub-workflow reference:** None

### Node: LINE_Push_Completion_Message
- **Type and technical role:** `n8n-nodes-base.httpRequest`; sends a final success push message to the LINE user.
- **Configuration choices:**  
  - POST to `https://api.line.me/v2/bot/message/push`
  - Text includes:
    - sheet name/category
    - title
    - tags
- **Key expressions or variables used:**  
  - User ID: `{{ $('LINE_Receive_Webhook').item.json.body.events[0].source.userId }}`
  - Category: `{{$node['Sheets_Check_Category_Exists'].json.category}}`
  - Title: `{{$json.title}}`
  - Tags: `{{ $json.tags }}`
  - Access token: `{{ $('Config_Set_Environment').item.json.LINE_ACCESS_TOKEN }}`
- **Input and output connections:**  
  - Input from `Sheets_Append_Row`
  - No downstream node
- **Version-specific requirements:**  
  - `typeVersion` 4.3
- **Edge cases or potential failure types:**  
  - `$json.title` and `$json.tags` depend on the item shape coming from `Sheets_Append_Row`; in some configurations these may no longer be present
  - Safer design would reference `Data_Parse_OCR_JSON` directly for title/tags
  - Push API requires valid user ID and channel permissions
- **Sub-workflow reference:** None

---

## 2.7 Documentation / Annotation Nodes

**Overview:**  
These nodes do not affect execution. They document intent, setup steps, and logical grouping for human readers in the n8n canvas.

**Nodes Involved:**  
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4
- Sticky Note5
- Sticky Note6
- Sticky Note7
- Sticky Note8

### Node: Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`; visual annotation for image handling.
- **Configuration choices:** Documents download, Drive upload, and binary restore.
- **Input and output connections:** None
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

### Node: Sticky Note1
- **Type and technical role:** `n8n-nodes-base.stickyNote`; high-level overview and setup notes.
- **Configuration choices:** Describes workflow behavior, setup steps, and limitations.
- **Input and output connections:** None
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

### Node: Sticky Note2
- **Type and technical role:** `n8n-nodes-base.stickyNote`; visual annotation for LINE intake and validation.
- **Configuration choices:** Explains validation and immediate response behavior.
- **Input and output connections:** None
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

### Node: Sticky Note3
- **Type and technical role:** `n8n-nodes-base.stickyNote`; annotation for OCR and AI processing.
- **Configuration choices:** Explains title/category/summary/tags generation.
- **Input and output connections:** None
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

### Node: Sticky Note4
- **Type and technical role:** `n8n-nodes-base.stickyNote`; annotation for JSON parsing.
- **Configuration choices:** Describes cleanup and fallback handling.
- **Input and output connections:** None
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

### Node: Sticky Note5
- **Type and technical role:** `n8n-nodes-base.stickyNote`; annotation for OCR failure logic.
- **Configuration choices:** Notes stop conditions to prevent incorrect saves.
- **Input and output connections:** None
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

### Node: Sticky Note6
- **Type and technical role:** `n8n-nodes-base.stickyNote`; annotation for sheet management.
- **Configuration choices:** Notes sheet existence check and sheet creation.
- **Input and output connections:** None
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

### Node: Sticky Note7
- **Type and technical role:** `n8n-nodes-base.stickyNote`; annotation for data storage.
- **Configuration choices:** Lists stored fields.
- **Input and output connections:** None
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

### Node: Sticky Note8
- **Type and technical role:** `n8n-nodes-base.stickyNote`; annotation for completion response.
- **Configuration choices:** Notes final user notification behavior.
- **Input and output connections:** None
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| LINE_Receive_Webhook | Webhook | Receives LINE webhook POST events |  | Config_Set_Environment | ## AI Handwritten Memo Organizer – Overview<br>This workflow receives handwritten memo images sent via LINE and automatically extracts, summarizes, and organizes the content using AI.<br><br>## Step-by-step process:<br>1. User sends a handwritten memo image via LINE<br>2. Webhook receives the image<br>3. Immediate reply is sent: “Processing…”<br>4. Image is saved to Google Drive<br>5. AI performs OCR and generates structured data (title, category, summary, tags)<br>6. The JSON response is safely parsed with error handling<br>7. OCR failure is detected if text cannot be properly extracted<br>If OCR fails:<br>→ User is notified with guidance for retaking the image<br>If OCR succeeds:<br>→ Check if the category sheet exists in Google Sheets<br>→ If not, create a new sheet<br>→ Save the data (title, summary, tags, date, image URL)<br>8. Completion message is sent to the user via LINE<br><br>## Setup Steps<br>1. Create a LINE Messaging API channel and obtain the Channel Access Token<br>2. Create a Google Spreadsheet for storing memo data<br>3. Create a Google Drive folder to store uploaded images<br>4. Set the following values in the Config node:<br>o LINE_ACCESS_TOKEN<br>o GOOGLE_SHEETS_ID<br>5. Set the Webhook URL in the LINE Developers Console<br><br>## Key points:<br>• Immediate response improves user experience<br>• JSON parsing includes fallback handling for malformed AI output<br>• OCR failure is detected and handled safely<br>• Data is automatically categorized into separate sheets<br>• Tags are generated to enable future search functionality<br><br>## Notes:<br>• If a non-image message is sent, the user is prompted to send an image<br>• If text cannot be extracted, the data will not be saved<br>• The AI output may not always be perfectly accurate<br>• This workflow is designed for memo organization, not critical data processing |
| Config_Set_Environment | Set | Stores LINE token and Google Sheets ID placeholders | LINE_Receive_Webhook | LINE_Check_MessageType | ## AI Handwritten Memo Organizer – Overview<br>This workflow receives handwritten memo images sent via LINE and automatically extracts, summarizes, and organizes the content using AI.<br><br>## Step-by-step process:<br>1. User sends a handwritten memo image via LINE<br>2. Webhook receives the image<br>3. Immediate reply is sent: “Processing…”<br>4. Image is saved to Google Drive<br>5. AI performs OCR and generates structured data (title, category, summary, tags)<br>6. The JSON response is safely parsed with error handling<br>7. OCR failure is detected if text cannot be properly extracted<br>If OCR fails:<br>→ User is notified with guidance for retaking the image<br>If OCR succeeds:<br>→ Check if the category sheet exists in Google Sheets<br>→ If not, create a new sheet<br>→ Save the data (title, summary, tags, date, image URL)<br>8. Completion message is sent to the user via LINE<br><br>## Setup Steps<br>1. Create a LINE Messaging API channel and obtain the Channel Access Token<br>2. Create a Google Spreadsheet for storing memo data<br>3. Create a Google Drive folder to store uploaded images<br>4. Set the following values in the Config node:<br>o LINE_ACCESS_TOKEN<br>o GOOGLE_SHEETS_ID<br>5. Set the Webhook URL in the LINE Developers Console<br><br>## Key points:<br>• Immediate response improves user experience<br>• JSON parsing includes fallback handling for malformed AI output<br>• OCR failure is detected and handled safely<br>• Data is automatically categorized into separate sheets<br>• Tags are generated to enable future search functionality<br><br>## Notes:<br>• If a non-image message is sent, the user is prompted to send an image<br>• If text cannot be extracted, the data will not be saved<br>• The AI output may not always be perfectly accurate<br>• This workflow is designed for memo organization, not critical data processing |
| LINE_Check_MessageType | If | Validates that the incoming LINE message is an image | Config_Set_Environment | LINE_Reply_Processing; LINE_Reply_NoImage | ## LINE Input & Validation<br><br>Receives messages via LINE Webhook.<br><br>• Validates that the input is an image<br>• Non-image messages are rejected with guidance<br>• Ensures only valid data enters the workflow<br>• Sends an instant reply: “Processing…”<br><br>This step prevents invalid processing early. |
| LINE_Reply_Processing | HTTP Request | Sends immediate “Processing…” reply to LINE | LINE_Check_MessageType | LINE_Download_ImageContent | ## LINE Input & Validation<br><br>Receives messages via LINE Webhook.<br><br>• Validates that the input is an image<br>• Non-image messages are rejected with guidance<br>• Ensures only valid data enters the workflow<br>• Sends an instant reply: “Processing…”<br><br>This step prevents invalid processing early. |
| LINE_Reply_NoImage | HTTP Request | Replies to non-image messages with guidance | LINE_Check_MessageType |  | ## LINE Input & Validation<br><br>Receives messages via LINE Webhook.<br><br>• Validates that the input is an image<br>• Non-image messages are rejected with guidance<br>• Ensures only valid data enters the workflow<br>• Sends an instant reply: “Processing…”<br><br>This step prevents invalid processing early. |
| LINE_Download_ImageContent | HTTP Request | Downloads image binary from LINE content API | LINE_Reply_Processing | Drive_Upload_Image | ## Image Handling<br><br>Processes the incoming image:<br><br>• Downloads image from LINE API<br>• Uploads image to Google Drive<br>• Restores binary data<br><br>Ensures persistent storage for later use. |
| Drive_Upload_Image | Google Drive | Uploads the image file to Google Drive | LINE_Download_ImageContent | Binary_Restore_From_LINE | ## Image Handling<br><br>Processes the incoming image:<br><br>• Downloads image from LINE API<br>• Uploads image to Google Drive<br>• Restores binary data<br><br>Ensures persistent storage for later use. |
| Binary_Restore_From_LINE | Code | Restores binary data after Drive upload | Drive_Upload_Image | AI_OCR_Analyze_Image | ## Image Handling<br><br>Processes the incoming image:<br><br>• Downloads image from LINE API<br>• Uploads image to Google Drive<br>• Restores binary data<br><br>Ensures persistent storage for later use. |
| AI_Model_Gemini | Google Gemini Chat Model | Supplies the Gemini model to the OCR chain |  | AI_OCR_Analyze_Image | ## OCR & AI Processing<br><br>Extracts and structures data from the image:<br><br>• Performs OCR on handwritten content<br>• Generates structured JSON:<br>  - title<br>  - category<br>  - summary<br>  - tags<br><br>Transforms unstructured data into usable format. |
| AI_OCR_Analyze_Image | LLM Chain | Performs OCR-like extraction and structured summarization from image input | Binary_Restore_From_LINE; AI_Model_Gemini | Data_Parse_OCR_JSON | ## OCR & AI Processing<br><br>Extracts and structures data from the image:<br><br>• Performs OCR on handwritten content<br>• Generates structured JSON:<br>  - title<br>  - category<br>  - summary<br>  - tags<br><br>Transforms unstructured data into usable format. |
| Data_Parse_OCR_JSON | Code | Cleans and parses AI JSON output with fallback defaults | AI_OCR_Analyze_Image | AI_Check_OCR_Failure | ## JSON Parsing<br><br>Safely parses AI output:<br><br>• Removes invalid wrappers (e.g. ```json)<br>• Extracts JSON content<br>• Handles malformed responses with fallback<br><br>Prevents workflow crashes from AI inconsistencies. |
| AI_Check_OCR_Failure | If | Detects OCR failure using the fallback summary text | Data_Parse_OCR_JSON | LINE_Push_NoSummary; Sheets_Get_Metadata | ## OCR Failure Detection<br><br>Validates OCR result quality:<br><br>• Detects missing or invalid summary<br>• Stops workflow if text is not recognized<br><br>Prevents saving incorrect data. |
| LINE_Push_NoSummary | HTTP Request | Pushes failure guidance to the LINE user | AI_Check_OCR_Failure |  | ## OCR Failure Detection<br><br>Validates OCR result quality:<br><br>• Detects missing or invalid summary<br>• Stops workflow if text is not recognized<br><br>Prevents saving incorrect data. |
| Sheets_Get_Metadata | HTTP Request | Retrieves spreadsheet metadata to inspect existing sheet tabs | AI_Check_OCR_Failure | Sheets_Check_Category_Exists | ## Sheet Management<br><br>Manages Google Sheets structure:<br><br>• Checks if category sheet exists<br>• Creates new sheet if needed<br><br>Enables scalable data organization. |
| Sheets_Check_Category_Exists | Code | Checks whether the OCR category already exists as a sheet tab | Sheets_Get_Metadata | Sheets_Branch_Category | ## Sheet Management<br><br>Manages Google Sheets structure:<br><br>• Checks if category sheet exists<br>• Creates new sheet if needed<br><br>Enables scalable data organization. |
| Sheets_Branch_Category | If | Branches based on whether the category sheet exists | Sheets_Check_Category_Exists | Sheets_Append_Row; Sheets_Create_Category | ## Sheet Management<br><br>Manages Google Sheets structure:<br><br>• Checks if category sheet exists<br>• Creates new sheet if needed<br><br>Enables scalable data organization. |
| Sheets_Create_Category | Google Sheets | Creates a new sheet tab for a new OCR category | Sheets_Branch_Category | Data_Prepare_Sheet_Row | ## Sheet Management<br><br>Manages Google Sheets structure:<br><br>• Checks if category sheet exists<br>• Creates new sheet if needed<br><br>Enables scalable data organization. |
| Data_Prepare_Sheet_Row | Set | Prepares row fields after creating a new category sheet | Sheets_Create_Category | Sheets_Append_Row | ## Sheet Management<br><br>Manages Google Sheets structure:<br><br>• Checks if category sheet exists<br>• Creates new sheet if needed<br><br>Enables scalable data organization. |
| Sheets_Append_Row | Google Sheets | Appends memo metadata to the category sheet | Sheets_Branch_Category; Data_Prepare_Sheet_Row | LINE_Push_Completion_Message | ## Data Storage<br><br>Stores structured data:<br><br>• Title<br>• Summary<br>• Tags<br>• Date<br>• Image URL<br><br>All data is appended to the correct sheet. |
| LINE_Push_Completion_Message | HTTP Request | Pushes a success confirmation to the LINE user | Sheets_Append_Row |  | ## Completion Response<br><br>Sends results back to user:<br><br>• Confirms successful processing<br>• Provides organized output<br><br>Closes the workflow clearly. |
| Sticky Note | Sticky Note | Canvas documentation for image handling |  |  |  |
| Sticky Note1 | Sticky Note | Canvas documentation for global overview and setup |  |  |  |
| Sticky Note2 | Sticky Note | Canvas documentation for LINE validation block |  |  |  |
| Sticky Note3 | Sticky Note | Canvas documentation for OCR and AI block |  |  |  |
| Sticky Note4 | Sticky Note | Canvas documentation for JSON parsing block |  |  |  |
| Sticky Note5 | Sticky Note | Canvas documentation for OCR failure block |  |  |  |
| Sticky Note6 | Sticky Note | Canvas documentation for sheet management block |  |  |  |
| Sticky Note7 | Sticky Note | Canvas documentation for data storage block |  |  |  |
| Sticky Note8 | Sticky Note | Canvas documentation for completion response block |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n**
   - Name it something like: `AI Handwritten Memo Organizer`.
   - Keep execution order as default `v1` if you want parity with the original.

2. **Add a Webhook node**
   - Name: `LINE_Receive_Webhook`
   - Method: `POST`
   - Path: choose a unique path string
   - Save the workflow so the production URL is available later.
   - This will be the endpoint registered in LINE Developers Console.

3. **Add a Set node after the webhook**
   - Name: `Config_Set_Environment`
   - Add two string fields:
     - `LINE_ACCESS_TOKEN`
     - `GOOGLE_SHEETS_ID`
   - Fill them with your real values or placeholders during setup.
   - Connect `LINE_Receive_Webhook -> Config_Set_Environment`

4. **Add an If node to validate message type**
   - Name: `LINE_Check_MessageType`
   - Condition: String equals
   - Left value:
     `{{ $('LINE_Receive_Webhook').item.json.body.events[0].message.type }}`
   - Right value: `image`
   - Connect `Config_Set_Environment -> LINE_Check_MessageType`
   - You do not need the extra empty condition present in the source workflow.

5. **Add an HTTP Request node for the immediate reply**
   - Name: `LINE_Reply_Processing`
   - Method: `POST`
   - URL: `https://api.line.me/v2/bot/message/reply`
   - Send headers: enabled
   - Headers:
     - `Authorization` = `Bearer {{$node["Config_Set_Environment"].json.LINE_ACCESS_TOKEN }}`
     - `Content-type` = `application/json`
   - Body type: JSON
   - JSON body:
     ```json
     {
       "replyToken": "{{$node['LINE_Receive_Webhook'].json.body.events[0].replyToken }}",
       "messages": [
         {
           "type": "text",
           "text": "Processing…"
         }
       ]
     }
     ```
   - Connect the **true** output of `LINE_Check_MessageType` to this node.

6. **Add an HTTP Request node for non-image messages**
   - Name: `LINE_Reply_NoImage`
   - Method: `POST`
   - URL: `https://api.line.me/v2/bot/message/reply`
   - Same headers as above
   - JSON body:
     ```json
     {
       "replyToken": "{{$node['LINE_Receive_Webhook'].json.body.events[0].replyToken }}",
       "messages": [
         {
           "type": "text",
           "text": "If you send a handwritten memo, I will summarize its contents."
         }
       ]
     }
     ```
   - Connect the **false** output of `LINE_Check_MessageType` to this node.

7. **Add an HTTP Request node to download image content from LINE**
   - Name: `LINE_Download_ImageContent`
   - Method: `GET`
   - URL:
     `https://api-data.line.me/v2/bot/message/{{ $('LINE_Receive_Webhook').item.json.body.events[0].message.id }}/content`
   - Response format: File
   - Send headers: enabled
   - Header:
     - `Authorization` = `Bearer {{ $('Config_Set_Environment').item.json.LINE_ACCESS_TOKEN }}`
   - Connect `LINE_Reply_Processing -> LINE_Download_ImageContent`

8. **Create Google Drive credentials**
   - Add Google Drive OAuth2 credentials in n8n.
   - Make sure the authorized Google account has access to the destination Drive folder.

9. **Add a Google Drive node to upload the image**
   - Name: `Drive_Upload_Image`
   - Operation: Upload file
   - File name:
     `{{ $('LINE_Receive_Webhook').item.json.body.events[0].message.id }}.jpg`
   - Drive: `My Drive`
   - Folder: choose or paste the target folder ID
   - Use the Google Drive OAuth2 credential
   - Connect `LINE_Download_ImageContent -> Drive_Upload_Image`

10. **Add a Code node to restore binary data**
    - Name: `Binary_Restore_From_LINE`
    - Code:
      ```javascript
      const item = $input.first();

      item.binary = $node["LINE_Download_ImageContent"].binary;

      return [item];
      ```
    - Connect `Drive_Upload_Image -> Binary_Restore_From_LINE`

11. **Create Gemini credentials**
    - Add Google Gemini / PaLM API credentials in n8n.
    - Confirm the selected model supports image input in your n8n version.

12. **Add a Gemini chat model node**
    - Name: `AI_Model_Gemini`
    - Configure:
      - Temperature: `0`
      - Top P: `1`
      - Max output tokens: `2048`
    - Attach your Gemini credential.

13. **Add an LLM Chain node**
    - Name: `AI_OCR_Analyze_Image`
    - Prompt type: Define manually
    - Add a human message using `imageBinary`
    - Use this prompt text:
      ```text
      You are a high-precision OCR engine.

      Please read the text written in the image as accurately as possible.
      If the text cannot be read, you must output the following JSON and terminate the process:
      {
      "title": "",
      "category": "",
      "summary": "The text could not be recognized.",
      "tags":[]
      }

      If the text can be read, generate the following based on the summarized content:

      Title
      Category
      Summary
      Tags (up to 3 important keywords)

      【Rules】

      The category should be a short, easy-to-group term.

      Tags must be single words with no duplicates.

      Avoid overly abstract tags (e.g., important, memo).

      【Important】

      Output must be pure JSON only

      No explanations or preface text

      Do not output anything other than JSON

      Output the following fields in JSON format:

      {
      "title": "",
      "category": "",
      "summary": "",
      "tags":[]
      }
      ```
    - Connect:
      - `Binary_Restore_From_LINE -> AI_OCR_Analyze_Image`
      - `AI_Model_Gemini` to the chain’s model input

14. **Add a Code node to parse AI JSON**
    - Name: `Data_Parse_OCR_JSON`
    - Paste:
      ```javascript
      let raw = $json["text"] || "";

      raw = raw.replace(/```json\s*/i, "").replace(/```/g, "").trim();

      const match = raw.match(/\{[\s\S]*\}/);
      if (match) {
        raw = match[0];
      }

      let parsed;
      try {
        parsed = JSON.parse(raw);
      } catch (e) {
        parsed = {
          title: "",
          category: "",
          summary: raw.slice(0, 100),
          tags: []
        };
      }

      if (!parsed.title || typeof parsed.title !== "string") {
        parsed.title = "No title";
      }

      if (!parsed.category || typeof parsed.category !== "string") {
        parsed.category = "Uncategorized";
      }

      if (!parsed.summary || typeof parsed.summary !== "string") {
        parsed.summary = "No summary";
      }

      if (!Array.isArray(parsed.tags)) {
        parsed.tags = [];
      }

      parsed.tags = parsed.tags
        .filter(tag => typeof tag === "string")
        .slice(0, 3);

      const item = $input.first();

      item.json = {
        ...item.json,
        ...parsed
      };

      item.binary = $input.first().binary;

      return [item];
      ```
    - Connect `AI_OCR_Analyze_Image -> Data_Parse_OCR_JSON`

15. **Add an If node for OCR failure detection**
    - Name: `AI_Check_OCR_Failure`
    - Condition:
      - Left value: `{{ $json.summary }}`
      - Operation: equals
      - Right value: `The text could not be recognized.`
    - Connect `Data_Parse_OCR_JSON -> AI_Check_OCR_Failure`

16. **Add an HTTP Request node for OCR failure push**
    - Name: `LINE_Push_NoSummary`
    - Method: `POST`
    - URL: `https://api.line.me/v2/bot/message/push`
    - Headers:
      - `Authorization` = `Bearer {{ $('Config_Set_Environment').item.json.LINE_ACCESS_TOKEN }}`
      - `Content-type` = `application/json`
    - JSON body:
      ```json
      {
        "to":"{{ $('LINE_Receive_Webhook').item.json.body.events[0].source.userId }}",
        "messages": [
          {
            "type": "text",
            "text": "The image could not be read.\n• The text may be too small.\n• Please retake the photo in a well-lit environment."
          }
        ]
      }
      ```
    - Connect the **true** output of `AI_Check_OCR_Failure` to this node.

17. **Create Google Sheets credentials**
    - Add Google Sheets OAuth2 credentials with edit access to the target spreadsheet.
    - Also add a Google OAuth2 credential usable for raw Sheets API metadata requests if you want to reproduce the original node design exactly.

18. **Prepare the target spreadsheet**
    - Create a Google Spreadsheet.
    - Copy its spreadsheet ID into `Config_Set_Environment` as `GOOGLE_SHEETS_ID`.
    - Recommended: create at least one tab manually for testing.
    - For each category sheet you plan to append into, ensure the following headers exist in row 1:
      - `date`
      - `title`
      - `summary`
      - `webUrl`
      - `tags`
    - Important: newly created sheets may need header initialization depending on your preferred design. The source workflow does not explicitly add headers.

19. **Add an HTTP Request node to fetch spreadsheet metadata**
    - Name: `Sheets_Get_Metadata`
    - Method: `GET`
    - URL:
      `https://sheets.googleapis.com/v4/spreadsheets/{{ $('Config_Set_Environment').item.json.GOOGLE_SHEETS_ID }}`
    - Authentication: predefined credential
    - Credential type: Google OAuth2
    - Connect the **false** output of `AI_Check_OCR_Failure` to this node.

20. **Add a Code node to check category existence**
    - Name: `Sheets_Check_Category_Exists`
    - Code:
      ```javascript
      const category = $('Data_Parse_OCR_JSON').first().json.category;
      const sheets = $json.sheets || [];
      const sheetNames = sheets.map(s => s.properties.title);
      const exists = sheetNames.includes(category);

      return [
        {
          json: {
            category: category,
            sheetExists: exists,
            sheetNames: sheetNames
          }
        }
      ];
      ```
    - Connect `Sheets_Get_Metadata -> Sheets_Check_Category_Exists`

21. **Add an If node to branch on category existence**
    - Name: `Sheets_Branch_Category`
    - Condition:
      - Left value: `{{ $json.sheetExists }}`
      - Operation: equals
      - Right value: `true` as boolean
    - Connect `Sheets_Check_Category_Exists -> Sheets_Branch_Category`

22. **Add a Google Sheets node to create a new category sheet**
    - Name: `Sheets_Create_Category`
    - Operation: `Create`
    - Document: select the target spreadsheet
    - Title:
      `{{ $('Sheets_Check_Category_Exists').item.json.category }}`
    - Connect the **false** output of `Sheets_Branch_Category` to this node.

23. **Add a Set node to prepare row data**
    - Name: `Data_Prepare_Sheet_Row`
    - Add string fields:
      - `date` = `{{ $now.format("yyyy-MM-dd HH:mm:ss") }}`
      - `title` = `{{ $node['Data_Parse_OCR_JSON'].json.title }}`
      - `summary` = `{{ $node['Data_Parse_OCR_JSON'].json.summary }}`
      - `webUrl` = `{{ $node['Drive_Upload_Image'].json.webViewLink }}`
      - `tags` = `{{ $node['Data_Parse_OCR_JSON'].json.tags.join(',') }}`
    - Connect `Sheets_Create_Category -> Data_Prepare_Sheet_Row`

24. **Add a Google Sheets node to append the row**
    - Name: `Sheets_Append_Row`
    - Operation: `Append`
    - Document: select the target spreadsheet
    - Sheet name:
      `{{ $('Sheets_Check_Category_Exists').item.json.category }}`
    - Define columns manually:
      - `date` = `{{ $now.format("yyyy-MM-dd HH:mm:ss") }}`
      - `title` = `{{ $node['Data_Parse_OCR_JSON'].json.title }}`
      - `summary` = `{{ $node['Data_Parse_OCR_JSON'].json.summary }}`
      - `webUrl` = `{{ $('Drive_Upload_Image').item.json.webViewLink }}`
      - `tags` = `{{ $node['Data_Parse_OCR_JSON'].json.tags.join(',') }}`
    - Connect:
      - **true** output of `Sheets_Branch_Category -> Sheets_Append_Row`
      - `Data_Prepare_Sheet_Row -> Sheets_Append_Row`
    - Note: the node uses direct expressions from earlier nodes, so the incoming item payload is less important.

25. **Add an HTTP Request node for success push**
    - Name: `LINE_Push_Completion_Message`
    - Method: `POST`
    - URL: `https://api.line.me/v2/bot/message/push`
    - Headers:
      - `Authorization` = `Bearer {{ $('Config_Set_Environment').item.json.LINE_ACCESS_TOKEN }}`
      - `Content-type` = `application/json`
    - JSON body:
      ```json
      {
        "to":"{{ $('LINE_Receive_Webhook').item.json.body.events[0].source.userId }}",
        "messages": [
          {
            "type": "text",
            "text": "The text has been summarized and saved.\n SheetsName「{{$node['Sheets_Check_Category_Exists'].json.category}}」\nTitle「{{$json.title}}」\n Tags「{{ $json.tags }}」"
          }
        ]
      }
      ```
    - Connect `Sheets_Append_Row -> LINE_Push_Completion_Message`

26. **Optionally add sticky notes**
    - Add notes for:
      - overview/setup
      - LINE validation
      - image handling
      - OCR and AI
      - JSON parsing
      - OCR failure detection
      - sheet management
      - data storage
      - completion response

27. **Configure LINE Developers Console**
    - Create a Messaging API channel.
    - Enable webhook usage.
    - Paste the production webhook URL from `LINE_Receive_Webhook`.
    - Use the Channel Access Token as `LINE_ACCESS_TOKEN`.

28. **Test with a handwritten image**
    - Send an image to the LINE bot.
    - Expected behavior:
      - Immediate reply: `Processing…`
      - Image stored in Drive
      - OCR result parsed
      - Category sheet checked or created
      - Row appended to Sheets
      - Final success push sent to user

29. **Test failure paths**
    - Send a text message instead of an image.
      - Expected: guidance reply from `LINE_Reply_NoImage`
    - Send a blurry or unreadable image.
      - Expected: failure push from `LINE_Push_NoSummary`

30. **Recommended hardening improvements**
    - Replace `Config_Set_Environment` secrets with credentials or environment variables
    - Add LINE signature verification
    - Handle multiple events instead of only `events[0]`
    - Sanitize category names for Google Sheets tab restrictions
    - Initialize headers automatically when creating a new sheet
    - In the final push message, reference `Data_Parse_OCR_JSON` directly for title and tags instead of relying on `$json`

### Sub-workflow setup
This workflow does **not** invoke any sub-workflows and has only one entry point:
- `LINE_Receive_Webhook`

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Create a LINE Messaging API channel and obtain the Channel Access Token | LINE Developers Console |
| Create a Google Spreadsheet for storing memo data | Google Sheets |
| Create a Google Drive folder to store uploaded images | Google Drive |
| Set the Webhook URL in the LINE Developers Console | LINE webhook configuration |
| Immediate response improves user experience | Workflow design note |
| JSON parsing includes fallback handling for malformed AI output | Reliability note |
| OCR failure is detected and handled safely | Reliability note |
| Data is automatically categorized into separate sheets | Data organization note |
| Tags are generated to enable future search functionality | Searchability note |
| If a non-image message is sent, the user is prompted to send an image | Behavior note |
| If text cannot be extracted, the data will not be saved | Behavior note |
| The AI output may not always be perfectly accurate | AI limitation note |
| This workflow is designed for memo organization, not critical data processing | Scope / limitation note |