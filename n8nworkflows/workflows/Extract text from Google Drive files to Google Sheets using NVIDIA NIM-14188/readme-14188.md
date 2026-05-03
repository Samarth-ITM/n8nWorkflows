Extract text from Google Drive files to Google Sheets using NVIDIA NIM

https://n8nworkflows.xyz/workflows/extract-text-from-google-drive-files-to-google-sheets-using-nvidia-nim-14188


# Extract text from Google Drive files to Google Sheets using NVIDIA NIM

# 1. Workflow Overview

This workflow monitors a specific Google Drive folder for newly created files, extracts text depending on the file type, structures that text with NVIDIA NIM, logs the result into Google Sheets, and sends a Telegram confirmation.

Its main use cases are:
- automated document intake from a shared Google Drive folder
- lightweight OCR and text extraction for mixed file types
- structured logging of document content for downstream reporting or review
- operational notifications after processing

The workflow supports these input types:
- PDF
- image files
- Google Docs
- plain text files
- CSV files

Unsupported file types are notified through Telegram and then stopped without writing to Google Sheets.

## 1.1 Input Reception and Normalization
The workflow starts with a Google Drive Trigger that watches a specific folder every minute. A Code node then standardizes metadata such as file ID, MIME type, route, Telegram chat ID, and execution run ID.

## 1.2 File Routing and Fetching
A Switch node routes incoming items by normalized file type. Binary-based files are downloaded from Google Drive, while Google Docs are fetched through the Google Docs node.

## 1.3 Extraction Branches
Each supported format follows its own extraction path:
- PDFs use the native Extract from File node in PDF mode
- text files use the native Extract from File node in text mode
- CSV files are parsed from file contents into rows and then serialized as JSON text
- images are converted to a data URL and sent to an NVIDIA vision model for OCR-like extraction
- Google Docs are fetched directly and normalized into a single content string

## 1.4 AI Structuring
All supported branches converge into a common structuring step. The extracted text is trimmed, wrapped into a strict prompt with a guided JSON schema, and sent to NVIDIA NIM using `nvidia/llama-3.3-nemotron-super-49b-v1.5`.

## 1.5 Output Logging and Notification
The structured result is parsed, fallback values are applied if the model output is incomplete or invalid, and the final data is:
- appended to the `Extract_Log` tab in Google Sheets
- summarized into a Telegram message and sent to the configured chat

## 1.6 Fallback and Stop Paths
There are two special paths:
- `raw_text` exists in routing logic but is not reachable from the current Google Drive trigger normalization, since `Normalize Input` never assigns `raw_text`
- unsupported files trigger a Telegram alert and stop, with no Google Sheets write

---

# 2. Block-by-Block Analysis

## 2.1 Block: Trigger and Input Normalization

### Overview
This block detects new files in a target Google Drive folder and converts the trigger payload into a consistent internal format. It also decides which processing route should be used downstream.

### Nodes Involved
- Google Drive Trigger
- Normalize Input
- Route Input

### Node Details

#### Google Drive Trigger
- **Type and role:** `n8n-nodes-base.googleDriveTrigger`; entry point that polls Google Drive for newly created files.
- **Configuration choices:**
  - event: `fileCreated`
  - file type filter: `all`
  - polling schedule: every minute
  - trigger scope: specific folder
  - folder ID must be manually replaced from placeholder
- **Key expressions or variables used:**
  - folder ID is stored directly in the node as `REPLACE_WITH_GOOGLE_DRIVE_FOLDER_ID`
- **Input and output connections:**
  - no upstream node
  - outputs to `Normalize Input`
- **Version-specific requirements:**
  - typeVersion `1`
- **Edge cases / failure types:**
  - invalid or inaccessible folder ID
  - expired Google Drive OAuth2 credential
  - polling delays or missed expectations due to polling behavior
  - insufficient Drive permissions on shared folders
- **Sub-workflow reference:** none

#### Normalize Input
- **Type and role:** `n8n-nodes-base.code`; normalizes the incoming Google Drive metadata and computes the route.
- **Configuration choices:**
  - extracts:
    - `fileId`
    - `mimeType`
    - `fileName`
    - Google Doc-specific fields
    - `runId` from `$execution.id`
  - initializes:
    - `rawText` as empty string
    - `chatId` as placeholder
    - `messageId` as empty string
    - `binaryKey` as empty string
  - determines `route` using MIME type and filename
- **Key expressions or variables used:**
  - `const fileId = $json.id || $json.fileId || ''`
  - `const rawMimeType = ($json.mimeType || '').toString()`
  - route selection:
    - `application/pdf` → `pdf`
    - `image/*` → `image`
    - `application/vnd.google-apps.document` → `google_doc`
    - `text/csv` or `.csv` filename → `csv`
    - `text/*` → `text_file`
    - otherwise `unsupported`
  - `chatId: 'REPLACE_WITH_TELEGRAM_CHAT_ID'`
  - `runId: $execution.id`
- **Input and output connections:**
  - input from `Google Drive Trigger`
  - output to `Route Input`
- **Version-specific requirements:**
  - typeVersion `2`
  - uses modern Code node syntax
- **Edge cases / failure types:**
  - trigger payload missing `id`, `mimeType`, or `name`
  - Google-native file types other than Docs are marked unsupported
  - raw text route is not assigned here, so that branch is dormant
  - placeholder chat ID will cause Telegram send failures later
- **Sub-workflow reference:** none

#### Route Input
- **Type and role:** `n8n-nodes-base.switch`; dispatches items to the correct high-level branch.
- **Configuration choices:**
  - compares `{{ $json.route }}`
  - configured outputs:
    - `pdf`
    - `image`
    - `google_doc`
    - `text_file`
    - `csv`
    - `raw_text`
- **Key expressions or variables used:**
  - `value1 = {{ $json.route }}`
- **Input and output connections:**
  - input from `Normalize Input`
  - outputs to:
    - `Download Drive File` for `pdf`
    - `Download Drive File` for `image`
    - `Get Google Doc` for `google_doc`
    - `Download Drive File` for `text_file`
    - `Download Drive File` for `csv`
    - `Send Unsupported Message` on unmatched/unsupported effective path
- **Version-specific requirements:**
  - typeVersion `2`
- **Edge cases / failure types:**
  - if a route is introduced in code but not in switch, item may fall through unexpectedly
  - `raw_text` output exists but is not currently fed by the current normalization logic
- **Sub-workflow reference:** none

---

## 2.2 Block: Binary File Fetching and Google Doc Retrieval

### Overview
This block acquires the actual document content source required for extraction. Binary files are downloaded from Google Drive, while Google Docs are retrieved through the Google Docs integration.

### Nodes Involved
- Download Drive File
- Route Downloaded File
- Get Google Doc
- Normalize Google Doc

### Node Details

#### Download Drive File
- **Type and role:** `n8n-nodes-base.googleDrive`; downloads file binary from Google Drive.
- **Configuration choices:**
  - operation: `download`
  - file ID comes from normalized input
  - downloaded file name is set to the original file name
- **Key expressions or variables used:**
  - file ID: `{{ $json.fileId }}`
  - file name option: `{{ $json.fileName }}`
- **Input and output connections:**
  - input from `Route Input` for `pdf`, `image`, `text_file`, `csv`
  - output to `Route Downloaded File`
- **Version-specific requirements:**
  - typeVersion `3`
- **Edge cases / failure types:**
  - Google Drive auth issues
  - insufficient access rights
  - downloading Google-native files would fail, but those are routed elsewhere
  - binary output may use an unexpected key; later image logic attempts to detect it dynamically
- **Sub-workflow reference:** none

#### Route Downloaded File
- **Type and role:** `n8n-nodes-base.switch`; routes downloaded binary files into the correct extraction branch.
- **Configuration choices:**
  - evaluates the route from `Normalize Input` rather than current item JSON
  - outputs:
    - `pdf`
    - `image`
    - `text_file`
    - `csv`
- **Key expressions or variables used:**
  - `{{ $('Normalize Input').item.json.route }}`
- **Input and output connections:**
  - input from `Download Drive File`
  - outputs to:
    - `Extract from PDF`
    - `Prepare Image Data URL`
    - `Extract from Text File`
    - `Extract from CSV`
- **Version-specific requirements:**
  - typeVersion `2`
- **Edge cases / failure types:**
  - expression references another node’s item; if item pairing breaks in future edits, routing could misalign
  - no default output for unexpected values
- **Sub-workflow reference:** none

#### Get Google Doc
- **Type and role:** `n8n-nodes-base.googleDocs`; fetches a Google Doc directly instead of downloading a binary file.
- **Configuration choices:**
  - operation: `get`
  - document identifier is passed via the normalized Google Doc ID
- **Key expressions or variables used:**
  - `{{ $('Normalize Input').first().json.googleDocId }}`
- **Input and output connections:**
  - input from `Route Input`
  - output to `Normalize Google Doc`
- **Version-specific requirements:**
  - typeVersion `2`
  - requires a Google Docs credential/account selection in n8n even though the exported JSON does not show credentials inline
- **Edge cases / failure types:**
  - wrong document ID or inaccessible document
  - missing Google Docs credential
  - using a document URL field with an ID expression may still work depending on node behavior, but naming suggests this should be validated carefully
- **Sub-workflow reference:** none

#### Normalize Google Doc
- **Type and role:** `n8n-nodes-base.code`; extracts a best-effort text content string from the Google Docs API response.
- **Configuration choices:**
  - recursively searches the response for the longest string
  - falls back to full JSON stringification if no string is found
  - standardizes output fields to `content`, `fileType`, `fileName`, `chatId`, `sourceType`, `runId`
- **Key expressions or variables used:**
  - helper `pickLongestString(value)`
  - content source: `$input.first().json`
  - metadata from `$('Normalize Input').first().json`
- **Input and output connections:**
  - input from `Get Google Doc`
  - output to `Build Structuring Payload`
- **Version-specific requirements:**
  - typeVersion `2`
- **Edge cases / failure types:**
  - longest-string heuristic may capture metadata rather than the full text body if response structure changes
  - complex Docs formatting may be flattened poorly
- **Sub-workflow reference:** none

---

## 2.3 Block: PDF Extraction

### Overview
This branch handles PDF files using n8n’s native file extraction and normalizes the result into the common `content` schema expected downstream.

### Nodes Involved
- Extract from PDF
- Normalize PDF

### Node Details

#### Extract from PDF
- **Type and role:** `n8n-nodes-base.extractFromFile`; extracts text from PDF binary.
- **Configuration choices:**
  - operation: `pdf`
  - default options
- **Key expressions or variables used:**
  - none
- **Input and output connections:**
  - input from `Route Downloaded File`
  - output to `Normalize PDF`
- **Version-specific requirements:**
  - typeVersion `1`
- **Edge cases / failure types:**
  - scanned PDFs without embedded text may produce weak or empty output
  - malformed or encrypted PDFs may fail
  - large PDFs may stress memory or processing time
- **Sub-workflow reference:** none

#### Normalize PDF
- **Type and role:** `n8n-nodes-base.code`; consolidates PDF extraction output into a single `content` field.
- **Configuration choices:**
  - uses the same recursive longest-string helper pattern
  - emits standardized metadata
- **Key expressions or variables used:**
  - `pickLongestString($input.first().json)`
  - metadata from `$('Normalize Input').first().json`
- **Input and output connections:**
  - input from `Extract from PDF`
  - output to `Build Structuring Payload`
- **Version-specific requirements:**
  - typeVersion `2`
- **Edge cases / failure types:**
  - extracted structure may differ depending on n8n version or PDF parser behavior
  - longest-string heuristic may omit multi-part text if split unusually
- **Sub-workflow reference:** none

---

## 2.4 Block: Image OCR via NVIDIA NIM

### Overview
This branch converts image binary into a base64 data URL, sends it to an NVIDIA vision model, and normalizes the OCR-like response into plain text.

### Nodes Involved
- Prepare Image Data URL
- Build Image Payload
- Analyze Image with NVIDIA
- Normalize Image

### Node Details

#### Prepare Image Data URL
- **Type and role:** `n8n-nodes-base.code`; reads binary image content and creates a `data:` URL for API submission.
- **Configuration choices:**
  - tries `binaryKey` from normalized input first
  - otherwise uses the first available binary key
  - loads binary buffer via helper
  - base64-encodes it into `imageDataUrl`
- **Key expressions or variables used:**
  - `$('Normalize Input').first().json.binaryKey || Object.keys(item.binary || {})[0] || 'data'`
  - `await this.helpers.getBinaryDataBuffer(0, binaryKey)`
- **Input and output connections:**
  - input from `Route Downloaded File`
  - output to `Build Image Payload`
- **Version-specific requirements:**
  - typeVersion `2`
  - depends on Code node binary helper availability
- **Edge cases / failure types:**
  - no binary present
  - unsupported or missing MIME type
  - very large images may create oversized payloads
  - binary key mismatch
- **Sub-workflow reference:** none

#### Build Image Payload
- **Type and role:** `n8n-nodes-base.code`; builds the NVIDIA chat completion request payload for OCR extraction.
- **Configuration choices:**
  - model: `meta/llama-3.2-11b-vision-instruct`
  - system prompt forces plain text extraction only
  - user content includes extraction instruction and `image_url`
  - deterministic generation with `temperature: 0`, `top_p: 1`
  - `max_tokens: 1800`
  - stores payload as stringified JSON in `nimPayload`
- **Key expressions or variables used:**
  - `$json.imageDataUrl`
- **Input and output connections:**
  - input from `Prepare Image Data URL`
  - output to `Analyze Image with NVIDIA`
- **Version-specific requirements:**
  - typeVersion `2`
- **Edge cases / failure types:**
  - data URLs may exceed API size limits
  - model availability may differ by NVIDIA account region or plan
- **Sub-workflow reference:** none

#### Analyze Image with NVIDIA
- **Type and role:** `n8n-nodes-base.httpRequest`; sends the image OCR request to NVIDIA NIM.
- **Configuration choices:**
  - method: `POST`
  - URL: `https://integrate.api.nvidia.com/v1/chat/completions`
  - raw JSON body from `nimPayload`
  - content type explicitly set to `application/json`
  - authentication uses generic credential type with HTTP Header Auth
- **Key expressions or variables used:**
  - body: `{{ $json.nimPayload }}`
- **Input and output connections:**
  - input from `Build Image Payload`
  - output to `Normalize Image`
- **Version-specific requirements:**
  - typeVersion `4.2`
  - requires HTTP Header Auth credential with `Authorization: Bearer YOUR_API_KEY`
- **Edge cases / failure types:**
  - invalid API key
  - quota/rate limit issues
  - timeout on large images
  - non-JSON error responses
  - no `continueOnFail` here, so OCR branch failures stop this path
- **Sub-workflow reference:** none

#### Normalize Image
- **Type and role:** `n8n-nodes-base.code`; extracts the text content from the NVIDIA response and normalizes it.
- **Configuration choices:**
  - supports either string or array-based message content
  - removes code fences
  - converts empty or `NO_TEXT_EXTRACTED` response into `No text extracted`
- **Key expressions or variables used:**
  - `$json.choices?.[0]?.message?.content`
  - metadata from `$('Build Image Payload').first().json`
- **Input and output connections:**
  - input from `Analyze Image with NVIDIA`
  - output to `Build Structuring Payload`
- **Version-specific requirements:**
  - typeVersion `2`
- **Edge cases / failure types:**
  - unexpected NVIDIA response format
  - empty OCR result
  - hallucinated formatting or wrapper text from the model
- **Sub-workflow reference:** none

---

## 2.5 Block: Text File Extraction

### Overview
This branch extracts plain text from downloaded text files and maps the output into the workflow’s common content structure.

### Nodes Involved
- Extract from Text File
- Normalize Text File

### Node Details

#### Extract from Text File
- **Type and role:** `n8n-nodes-base.extractFromFile`; reads text content from a binary text file.
- **Configuration choices:**
  - operation: `text`
- **Key expressions or variables used:** none
- **Input and output connections:**
  - input from `Route Downloaded File`
  - output to `Normalize Text File`
- **Version-specific requirements:**
  - typeVersion `1`
- **Edge cases / failure types:**
  - encoding issues for non-UTF text files
  - large files may affect memory
- **Sub-workflow reference:** none

#### Normalize Text File
- **Type and role:** `n8n-nodes-base.code`; standardizes extracted text file output.
- **Configuration choices:**
  - uses recursive longest-string extraction helper
  - adds normalized metadata
- **Key expressions or variables used:**
  - `pickLongestString($input.first().json)`
  - metadata from `$('Normalize Input').first().json`
- **Input and output connections:**
  - input from `Extract from Text File`
  - output to `Build Structuring Payload`
- **Version-specific requirements:**
  - typeVersion `2`
- **Edge cases / failure types:**
  - parser output changes could affect longest-string heuristic
- **Sub-workflow reference:** none

---

## 2.6 Block: CSV Extraction

### Overview
This branch parses CSV content into row objects, then converts the rows into a JSON string so the structuring model can process tabular data as text.

### Nodes Involved
- Extract from CSV
- Normalize CSV

### Node Details

#### Extract from CSV
- **Type and role:** `n8n-nodes-base.extractFromFile`; parses CSV from binary input.
- **Configuration choices:**
  - default extraction behavior
  - no explicit operation set, so node uses its default file extraction mode for CSV-like input
- **Key expressions or variables used:** none
- **Input and output connections:**
  - input from `Route Downloaded File`
  - output to `Normalize CSV`
- **Version-specific requirements:**
  - typeVersion `1`
- **Edge cases / failure types:**
  - malformed delimiters, quoting, or headers
  - ambiguous encoding
  - very large CSVs can produce many items and large serialized content
- **Sub-workflow reference:** none

#### Normalize CSV
- **Type and role:** `n8n-nodes-base.code`; aggregates all parsed row items and turns them into one formatted string.
- **Configuration choices:**
  - collects `$input.all().map(item => item.json)`
  - serializes rows with `JSON.stringify(rows, null, 2)`
  - standardizes metadata
- **Key expressions or variables used:**
  - `$input.all()`
  - metadata from `$('Normalize Input').first().json`
- **Input and output connections:**
  - input from `Extract from CSV`
  - output to `Build Structuring Payload`
- **Version-specific requirements:**
  - typeVersion `2`
- **Edge cases / failure types:**
  - very large CSVs can generate oversized text that later gets truncated to 12,000 chars in structuring payload
- **Sub-workflow reference:** none

---

## 2.7 Block: Raw Text and Unsupported Fallbacks

### Overview
This block contains fallback handling. One path normalizes direct raw text for future reuse, while the other sends an unsupported-file Telegram message and exits.

### Nodes Involved
- Normalize Raw Text
- Send Unsupported Message

### Node Details

#### Normalize Raw Text
- **Type and role:** `n8n-nodes-base.code`; prepares already-available raw text into standard downstream structure.
- **Configuration choices:**
  - content comes from `Normalize Input.rawText`
  - file name is hardcoded to `telegram-message`
  - source type is `google_drive_text`
- **Key expressions or variables used:**
  - `$('Normalize Input').first().json.rawText`
- **Input and output connections:**
  - not connected from any current active route in this workflow
  - output to `Build Structuring Payload`
- **Version-specific requirements:**
  - typeVersion `2`
- **Edge cases / failure types:**
  - currently unreachable because `Normalize Input` never sets route to `raw_text`
  - if manually wired later, empty `rawText` would still pass through
- **Sub-workflow reference:** none

#### Send Unsupported Message
- **Type and role:** `n8n-nodes-base.telegram`; notifies the configured Telegram chat when an unsupported file type is detected.
- **Configuration choices:**
  - sends a fixed text explaining supported types
  - attribution disabled
  - chat ID comes from normalized input
- **Key expressions or variables used:**
  - chat ID: `{{ $('Normalize Input').first().json.chatId }}`
- **Input and output connections:**
  - input from `Route Input` unsupported/fallback path
  - no downstream node
- **Version-specific requirements:**
  - typeVersion `1.2`
  - requires Telegram credential
- **Edge cases / failure types:**
  - invalid Telegram bot token
  - bot not allowed to message the target chat
  - placeholder chat ID not replaced
- **Sub-workflow reference:** none

---

## 2.8 Block: AI Structuring with NVIDIA NIM

### Overview
This block converts extracted raw text into a consistent structured schema. It uses a guided JSON schema so downstream logging can rely on stable fields even when document types vary.

### Nodes Involved
- Build Structuring Payload
- Structure Output with NVIDIA
- Parse Structured Output

### Node Details

#### Build Structuring Payload
- **Type and role:** `n8n-nodes-base.code`; prepares the prompt and JSON schema for the structuring model.
- **Configuration choices:**
  - trims source text to 12,000 characters before sending
  - schema fields:
    - `title`
    - `summary`
    - `category`
    - `language`
    - `key_points`
    - `confidence_notes`
  - model: `nvidia/llama-3.3-nemotron-super-49b-v1.5`
  - deterministic settings: `temperature 0`, `top_p 1`
  - `max_tokens 600`
  - uses `guided_json`
- **Key expressions or variables used:**
  - `const rawContent = (input.content || '').toString().trim()`
  - `const trimmedContent = rawContent.slice(0, 12000)`
- **Input and output connections:**
  - inputs from:
    - `Normalize Google Doc`
    - `Normalize PDF`
    - `Normalize Image`
    - `Normalize Text File`
    - `Normalize CSV`
    - `Normalize Raw Text`
  - output to `Structure Output with NVIDIA`
- **Version-specific requirements:**
  - typeVersion `2`
- **Edge cases / failure types:**
  - prompt truncation may omit relevant content for long files
  - if `content` is empty, model still runs with little value
  - `guided_json` support depends on NVIDIA endpoint compatibility
- **Sub-workflow reference:** none

#### Structure Output with NVIDIA
- **Type and role:** `n8n-nodes-base.httpRequest`; sends the structuring request to NVIDIA NIM.
- **Configuration choices:**
  - POST to NVIDIA chat completions endpoint
  - raw JSON body from `nimPayload`
  - HTTP Header Auth credential
  - `continueOnFail: true`
- **Key expressions or variables used:**
  - body: `{{ $json.nimPayload }}`
- **Input and output connections:**
  - input from `Build Structuring Payload`
  - output to `Parse Structured Output`
- **Version-specific requirements:**
  - typeVersion `4.2`
- **Edge cases / failure types:**
  - API auth errors
  - model unavailability
  - quota/rate limits
  - request timeout
  - malformed or empty model response
  - because `continueOnFail` is enabled, downstream parsing may receive an error-shaped item instead of a normal response
- **Sub-workflow reference:** none

#### Parse Structured Output
- **Type and role:** `n8n-nodes-base.code`; parses the model response safely and applies fallback values if needed.
- **Configuration choices:**
  - attempts direct `JSON.parse`
  - if needed, extracts the first `{ ... }` block and parses that
  - removes `<think>...</think>` tags
  - fallback strategy:
    - title → file name or `Untitled`
    - summary → first 300 chars of original extracted text
    - category/language/confidence_notes → empty string
    - key_points → empty array
  - passes original extracted text downstream as `extracted_text`
- **Key expressions or variables used:**
  - `$json.choices?.[0]?.message?.content?.trim()`
  - `$('Build Structuring Payload').first().json.originalContent`
- **Input and output connections:**
  - input from `Structure Output with NVIDIA`
  - outputs to:
    - `Append Row in Sheet`
    - `Build Telegram Reply`
- **Version-specific requirements:**
  - typeVersion `2`
- **Edge cases / failure types:**
  - if upstream HTTP Request failed under `continueOnFail`, expected response fields may be absent
  - parsing regex can extract wrong JSON block if response contains multiple objects
  - fallback summary may be noisy for CSV JSON strings
- **Sub-workflow reference:** none

---

## 2.9 Block: Logging and Notification

### Overview
This final block persists the structured output into Google Sheets and sends a concise Telegram success message.

### Nodes Involved
- Append Row in Sheet
- Build Telegram Reply
- Send Reply

### Node Details

#### Append Row in Sheet
- **Type and role:** `n8n-nodes-base.googleSheets`; appends one structured result row to the `Extract_Log` tab.
- **Configuration choices:**
  - operation: `append`
  - spreadsheet ID must be replaced from placeholder
  - sheet name: `Extract_Log`
  - explicit field mapping in define-below mode
  - includes:
    - Title
    - Run ID
    - Summary
    - Category
    - Language
    - File Name
    - File Type
    - Timestamp
    - Key Points
    - Source Type
    - Extracted Text
    - Confidence Notes
  - timestamp generated with current ISO datetime
  - `continueOnFail: true`
- **Key expressions or variables used:**
  - `{{ $json.title }}`
  - `{{ new Date().toISOString() }}`
  - `{{ ($json.key_points || []).join(' | ') }}`
  - spreadsheet ID placeholder: `REPLACE_WITH_GOOGLE_SHEET_ID`
- **Input and output connections:**
  - input from `Parse Structured Output`
  - no downstream node
- **Version-specific requirements:**
  - typeVersion `4.5`
  - requires matching columns in target sheet
- **Edge cases / failure types:**
  - wrong spreadsheet ID
  - missing `Extract_Log` tab
  - column names do not match configured mappings
  - permission issues on the spreadsheet
  - row still proceeds to Telegram branch because this node is parallel, not blocking
- **Sub-workflow reference:** none

#### Build Telegram Reply
- **Type and role:** `n8n-nodes-base.code`; formats a compact success message for Telegram.
- **Configuration choices:**
  - message includes:
    - fixed header `FILE PROCESSED`
    - file name
    - file type
    - title
    - category
    - status line
  - trims message to 3500 chars max
- **Key expressions or variables used:**
  - values taken from current JSON fields
- **Input and output connections:**
  - input from `Parse Structured Output`
  - output to `Send Reply`
- **Version-specific requirements:**
  - typeVersion `2`
- **Edge cases / failure types:**
  - if upstream parsing produced blanks, message still sends but may be sparse
- **Sub-workflow reference:** none

#### Send Reply
- **Type and role:** `n8n-nodes-base.telegram`; sends the success notification to Telegram.
- **Configuration choices:**
  - text from `replyText`
  - chat ID from parsed item
  - attribution disabled
- **Key expressions or variables used:**
  - `{{ $json.replyText }}`
  - `{{ $json.chatId }}`
- **Input and output connections:**
  - input from `Build Telegram Reply`
  - no downstream node
- **Version-specific requirements:**
  - typeVersion `1.2`
  - requires Telegram bot credential
- **Edge cases / failure types:**
  - invalid bot credentials
  - wrong chat ID
  - bot blocked or not present in target chat
- **Sub-workflow reference:** none

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Google Drive Trigger | googleDriveTrigger | Polls a specific Google Drive folder for newly created files |  | Normalize Input | # Extract Text From Google Drive Files<br>**Who it's for:** Teams and individuals who need to automatically capture, structure, and log content from files dropped into a shared Google Drive folder.<br>**What it does:** Monitors a Drive folder for new files, extracts text by file type, structures the result with NVIDIA NIM, logs it to Google Sheets, and sends a Telegram notification.<br>**How it works:** 1. Google Drive Trigger detects a new file 2. File type is identified and routed (PDF, image, Google Docs, TXT, CSV) 3. Text is extracted via the appropriate method for each type 4. NVIDIA NIM structures the extracted text into a consistent schema 5. Result is appended to a Google Sheets log 6. Telegram notification confirms completion<br>**Required setup:** - Google Drive OAuth2 credential - NVIDIA NIM API key (HTTP Header Auth) - Telegram bot credential - Google Sheets OAuth2 credential<br>Built by Cordexa Technologies<br>https://cordexa.tech<br>cordexatech@gmail.com<br>## Step 1 — Config<br>This workflow watches a specific Google Drive folder every minute, normalizes file metadata, and routes each file by MIME type.<br>Before testing: - Replace `REPLACE_WITH_GOOGLE_DRIVE_FOLDER_ID` in `Google Drive Trigger` with your Drive folder ID. - Replace `REPLACE_WITH_TELEGRAM_CHAT_ID` in `Normalize Input` with your destination Telegram chat ID.<br>Supported routes: - PDF - image - Google Doc - text file - CSV - unsupported fallback |
| Normalize Input | code | Standardizes trigger metadata and determines processing route | Google Drive Trigger | Route Input | # Extract Text From Google Drive Files<br>**Who it's for:** Teams and individuals who need to automatically capture, structure, and log content from files dropped into a shared Google Drive folder.<br>**What it does:** Monitors a Drive folder for new files, extracts text by file type, structures the result with NVIDIA NIM, logs it to Google Sheets, and sends a Telegram notification.<br>**How it works:** 1. Google Drive Trigger detects a new file 2. File type is identified and routed (PDF, image, Google Docs, TXT, CSV) 3. Text is extracted via the appropriate method for each type 4. NVIDIA NIM structures the extracted text into a consistent schema 5. Result is appended to a Google Sheets log 6. Telegram notification confirms completion<br>**Required setup:** - Google Drive OAuth2 credential - NVIDIA NIM API key (HTTP Header Auth) - Telegram bot credential - Google Sheets OAuth2 credential<br>Built by Cordexa Technologies<br>https://cordexa.tech<br>cordexatech@gmail.com<br>## Step 1 — Config<br>This workflow watches a specific Google Drive folder every minute, normalizes file metadata, and routes each file by MIME type.<br>Before testing: - Replace `REPLACE_WITH_GOOGLE_DRIVE_FOLDER_ID` in `Google Drive Trigger` with your Drive folder ID. - Replace `REPLACE_WITH_TELEGRAM_CHAT_ID` in `Normalize Input` with your destination Telegram chat ID.<br>Supported routes: - PDF - image - Google Doc - text file - CSV - unsupported fallback |
| Route Input | switch | Routes normalized input by file type | Normalize Input | Download Drive File; Get Google Doc; Send Unsupported Message | # Extract Text From Google Drive Files<br>**Who it's for:** Teams and individuals who need to automatically capture, structure, and log content from files dropped into a shared Google Drive folder.<br>**What it does:** Monitors a Drive folder for new files, extracts text by file type, structures the result with NVIDIA NIM, logs it to Google Sheets, and sends a Telegram notification.<br>**How it works:** 1. Google Drive Trigger detects a new file 2. File type is identified and routed (PDF, image, Google Docs, TXT, CSV) 3. Text is extracted via the appropriate method for each type 4. NVIDIA NIM structures the extracted text into a consistent schema 5. Result is appended to a Google Sheets log 6. Telegram notification confirms completion<br>**Required setup:** - Google Drive OAuth2 credential - NVIDIA NIM API key (HTTP Header Auth) - Telegram bot credential - Google Sheets OAuth2 credential<br>Built by Cordexa Technologies<br>https://cordexa.tech<br>cordexatech@gmail.com<br>## Step 1 — Config<br>This workflow watches a specific Google Drive folder every minute, normalizes file metadata, and routes each file by MIME type.<br>Before testing: - Replace `REPLACE_WITH_GOOGLE_DRIVE_FOLDER_ID` in `Google Drive Trigger` with your Drive folder ID. - Replace `REPLACE_WITH_TELEGRAM_CHAT_ID` in `Normalize Input` with your destination Telegram chat ID.<br>Supported routes: - PDF - image - Google Doc - text file - CSV - unsupported fallback |
| Download Drive File | googleDrive | Downloads supported binary files from Drive | Route Input | Route Downloaded File | ## File Fetching<br>Downloads the file binary from Drive for local extraction (PDF, image, TXT, CSV) or fetches Google Doc content directly via the Docs API. |
| Route Downloaded File | switch | Routes downloaded binary content to the correct extractor | Download Drive File | Extract from PDF; Prepare Image Data URL; Extract from Text File; Extract from CSV | ## File Fetching<br>Downloads the file binary from Drive for local extraction (PDF, image, TXT, CSV) or fetches Google Doc content directly via the Docs API.<br>## Extraction Branches<br>Extracts raw text from each supported file type. - **PDF** — native Extract from File node - **TXT** — native Extract from File node - **CSV** — parsed to JSON rows - **Image** — NVIDIA Llama 3.2 vision model via HTTP - **Google Doc** — pre-fetched upstream, normalised here<br>All paths converge on a unified `content` field passed downstream. |
| Get Google Doc | googleDocs | Retrieves Google Docs content directly | Route Input | Normalize Google Doc | ## File Fetching<br>Downloads the file binary from Drive for local extraction (PDF, image, TXT, CSV) or fetches Google Doc content directly via the Docs API. |
| Normalize Google Doc | code | Flattens Google Docs response into a single content field | Get Google Doc | Build Structuring Payload | ## Extraction Branches<br>Extracts raw text from each supported file type. - **PDF** — native Extract from File node - **TXT** — native Extract from File node - **CSV** — parsed to JSON rows - **Image** — NVIDIA Llama 3.2 vision model via HTTP - **Google Doc** — pre-fetched upstream, normalised here<br>All paths converge on a unified `content` field passed downstream. |
| Extract from PDF | extractFromFile | Extracts text from PDF binary | Route Downloaded File | Normalize PDF | ## Extraction Branches<br>Extracts raw text from each supported file type. - **PDF** — native Extract from File node - **TXT** — native Extract from File node - **CSV** — parsed to JSON rows - **Image** — NVIDIA Llama 3.2 vision model via HTTP - **Google Doc** — pre-fetched upstream, normalised here<br>All paths converge on a unified `content` field passed downstream. |
| Normalize PDF | code | Normalizes PDF extraction output into common content schema | Extract from PDF | Build Structuring Payload | ## Extraction Branches<br>Extracts raw text from each supported file type. - **PDF** — native Extract from File node - **TXT** — native Extract from File node - **CSV** — parsed to JSON rows - **Image** — NVIDIA Llama 3.2 vision model via HTTP - **Google Doc** — pre-fetched upstream, normalised here<br>All paths converge on a unified `content` field passed downstream. |
| Prepare Image Data URL | code | Converts downloaded image binary into a base64 data URL | Route Downloaded File | Build Image Payload | ## Extraction Branches<br>Extracts raw text from each supported file type. - **PDF** — native Extract from File node - **TXT** — native Extract from File node - **CSV** — parsed to JSON rows - **Image** — NVIDIA Llama 3.2 vision model via HTTP - **Google Doc** — pre-fetched upstream, normalised here<br>All paths converge on a unified `content` field passed downstream. |
| Build Image Payload | code | Builds NVIDIA vision API request payload | Prepare Image Data URL | Analyze Image with NVIDIA | ## Extraction Branches<br>Extracts raw text from each supported file type. - **PDF** — native Extract from File node - **TXT** — native Extract from File node - **CSV** — parsed to JSON rows - **Image** — NVIDIA Llama 3.2 vision model via HTTP - **Google Doc** — pre-fetched upstream, normalised here<br>All paths converge on a unified `content` field passed downstream. |
| Analyze Image with NVIDIA | httpRequest | Calls NVIDIA NIM vision model for OCR-like text extraction | Build Image Payload | Normalize Image | ## Extraction Branches<br>Extracts raw text from each supported file type. - **PDF** — native Extract from File node - **TXT** — native Extract from File node - **CSV** — parsed to JSON rows - **Image** — NVIDIA Llama 3.2 vision model via HTTP - **Google Doc** — pre-fetched upstream, normalised here<br>All paths converge on a unified `content` field passed downstream.<br>## AI Structuring with NVIDIA NIM<br>Sends extracted text to `nvidia/llama-3.3-nemotron-super-49b-v1.5` with a guided JSON schema. Returns: title, summary, category, language, key points, and confidence notes. Falls back gracefully if the model call fails (`continueOnFail: true`).<br>**Setup:** Add your NVIDIA NIM API key as an HTTP Header Auth credential (`Authorization: Bearer YOUR_KEY`). Apply it to both NVIDIA HTTP Request nodes. |
| Normalize Image | code | Extracts usable text from NVIDIA vision response | Analyze Image with NVIDIA | Build Structuring Payload | ## AI Structuring with NVIDIA NIM<br>Sends extracted text to `nvidia/llama-3.3-nemotron-super-49b-v1.5` with a guided JSON schema. Returns: title, summary, category, language, key points, and confidence notes. Falls back gracefully if the model call fails (`continueOnFail: true`).<br>**Setup:** Add your NVIDIA NIM API key as an HTTP Header Auth credential (`Authorization: Bearer YOUR_KEY`). Apply it to both NVIDIA HTTP Request nodes. |
| Extract from Text File | extractFromFile | Extracts plain text from a downloaded text file | Route Downloaded File | Normalize Text File | ## Extraction Branches<br>Extracts raw text from each supported file type. - **PDF** — native Extract from File node - **TXT** — native Extract from File node - **CSV** — parsed to JSON rows - **Image** — NVIDIA Llama 3.2 vision model via HTTP - **Google Doc** — pre-fetched upstream, normalised here<br>All paths converge on a unified `content` field passed downstream. |
| Normalize Text File | code | Normalizes text file extraction output | Extract from Text File | Build Structuring Payload | ## Extraction Branches<br>Extracts raw text from each supported file type. - **PDF** — native Extract from File node - **TXT** — native Extract from File node - **CSV** — parsed to JSON rows - **Image** — NVIDIA Llama 3.2 vision model via HTTP - **Google Doc** — pre-fetched upstream, normalised here<br>All paths converge on a unified `content` field passed downstream. |
| Extract from CSV | extractFromFile | Parses CSV file content into row items | Route Downloaded File | Normalize CSV | ## Extraction Branches<br>Extracts raw text from each supported file type. - **PDF** — native Extract from File node - **TXT** — native Extract from File node - **CSV** — parsed to JSON rows - **Image** — NVIDIA Llama 3.2 vision model via HTTP - **Google Doc** — pre-fetched upstream, normalised here<br>All paths converge on a unified `content` field passed downstream. |
| Normalize CSV | code | Aggregates CSV row items into a single JSON text blob | Extract from CSV | Build Structuring Payload | ## Extraction Branches<br>Extracts raw text from each supported file type. - **PDF** — native Extract from File node - **TXT** — native Extract from File node - **CSV** — parsed to JSON rows - **Image** — NVIDIA Llama 3.2 vision model via HTTP - **Google Doc** — pre-fetched upstream, normalised here<br>All paths converge on a unified `content` field passed downstream. |
| Normalize Raw Text | code | Normalizes direct raw text into common schema |  | Build Structuring Payload | ## Fallback Paths<br>Raw text inputs bypass extraction entirely.<br>Unsupported MIME types send a Telegram notification and stop — no Sheets row is written. |
| Send Unsupported Message | telegram | Sends Telegram notification for unsupported files | Route Input |  | ## Fallback Paths<br>Raw text inputs bypass extraction entirely.<br>Unsupported MIME types send a Telegram notification and stop — no Sheets row is written. |
| Build Structuring Payload | code | Builds guided JSON structuring request for NVIDIA | Normalize Google Doc; Normalize PDF; Normalize Image; Normalize Text File; Normalize CSV; Normalize Raw Text | Structure Output with NVIDIA | ## AI Structuring with NVIDIA NIM<br>Sends extracted text to `nvidia/llama-3.3-nemotron-super-49b-v1.5` with a guided JSON schema. Returns: title, summary, category, language, key points, and confidence notes. Falls back gracefully if the model call fails (`continueOnFail: true`).<br>**Setup:** Add your NVIDIA NIM API key as an HTTP Header Auth credential (`Authorization: Bearer YOUR_KEY`). Apply it to both NVIDIA HTTP Request nodes. |
| Structure Output with NVIDIA | httpRequest | Sends extracted text to NVIDIA for structured JSON output | Build Structuring Payload | Parse Structured Output | ## AI Structuring with NVIDIA NIM<br>Sends extracted text to `nvidia/llama-3.3-nemotron-super-49b-v1.5` with a guided JSON schema. Returns: title, summary, category, language, key points, and confidence notes. Falls back gracefully if the model call fails (`continueOnFail: true`).<br>**Setup:** Add your NVIDIA NIM API key as an HTTP Header Auth credential (`Authorization: Bearer YOUR_KEY`). Apply it to both NVIDIA HTTP Request nodes. |
| Parse Structured Output | code | Safely parses NVIDIA output and applies fallback defaults | Structure Output with NVIDIA | Append Row in Sheet; Build Telegram Reply | ## AI Structuring with NVIDIA NIM<br>Sends extracted text to `nvidia/llama-3.3-nemotron-super-49b-v1.5` with a guided JSON schema. Returns: title, summary, category, language, key points, and confidence notes. Falls back gracefully if the model call fails (`continueOnFail: true`).<br>**Setup:** Add your NVIDIA NIM API key as an HTTP Header Auth credential (`Authorization: Bearer YOUR_KEY`). Apply it to both NVIDIA HTTP Request nodes. |
| Append Row in Sheet | googleSheets | Appends structured extraction results to Google Sheets | Parse Structured Output |  | ## Step 3 — Delivery<br>This workflow delivers output in two places:<br>- `Append Row in Sheet` writes a structured row to the `Extract_Log` tab - `Send Reply` sends a short Telegram confirmation message<br>Before going live: - Replace `REPLACE_WITH_GOOGLE_SHEET_ID` in `Append Row in Sheet` - Confirm the target sheet contains an `Extract_Log` tab - Confirm the sheet headers match the mapped fields in the node - Confirm Telegram replies go to the configured chat ID |
| Build Telegram Reply | code | Formats success notification text | Parse Structured Output | Send Reply | ## Step 3 — Delivery<br>This workflow delivers output in two places:<br>- `Append Row in Sheet` writes a structured row to the `Extract_Log` tab - `Send Reply` sends a short Telegram confirmation message<br>Before going live: - Replace `REPLACE_WITH_GOOGLE_SHEET_ID` in `Append Row in Sheet` - Confirm the target sheet contains an `Extract_Log` tab - Confirm the sheet headers match the mapped fields in the node - Confirm Telegram replies go to the configured chat ID |
| Send Reply | telegram | Sends final Telegram success message | Build Telegram Reply |  | ## Step 3 — Delivery<br>This workflow delivers output in two places:<br>- `Append Row in Sheet` writes a structured row to the `Extract_Log` tab - `Send Reply` sends a short Telegram confirmation message<br>Before going live: - Replace `REPLACE_WITH_GOOGLE_SHEET_ID` in `Append Row in Sheet` - Confirm the target sheet contains an `Extract_Log` tab - Confirm the sheet headers match the mapped fields in the node - Confirm Telegram replies go to the configured chat ID |
| Overview | stickyNote | Canvas documentation note |  |  | # Extract Text From Google Drive Files<br>**Who it's for:** Teams and individuals who need to automatically capture, structure, and log content from files dropped into a shared Google Drive folder.<br>**What it does:** Monitors a Drive folder for new files, extracts text by file type, structures the result with NVIDIA NIM, logs it to Google Sheets, and sends a Telegram notification.<br>**How it works:** 1. Google Drive Trigger detects a new file 2. File type is identified and routed (PDF, image, Google Docs, TXT, CSV) 3. Text is extracted via the appropriate method for each type 4. NVIDIA NIM structures the extracted text into a consistent schema 5. Result is appended to a Google Sheets log 6. Telegram notification confirms completion<br>**Required setup:** - Google Drive OAuth2 credential - NVIDIA NIM API key (HTTP Header Auth) - Telegram bot credential - Google Sheets OAuth2 credential<br>Built by Cordexa Technologies<br>https://cordexa.tech<br>cordexatech@gmail.com |
| Section - Trigger and Input | stickyNote | Canvas section note for input configuration |  |  | ## Step 1 — Config<br>This workflow watches a specific Google Drive folder every minute, normalizes file metadata, and routes each file by MIME type.<br>Before testing: - Replace `REPLACE_WITH_GOOGLE_DRIVE_FOLDER_ID` in `Google Drive Trigger` with your Drive folder ID. - Replace `REPLACE_WITH_TELEGRAM_CHAT_ID` in `Normalize Input` with your destination Telegram chat ID.<br>Supported routes: - PDF - image - Google Doc - text file - CSV - unsupported fallback |
| Section - File Fetching | stickyNote | Canvas section note for file acquisition |  |  | ## File Fetching<br>Downloads the file binary from Drive for local extraction (PDF, image, TXT, CSV) or fetches Google Doc content directly via the Docs API. |
| Section - Extraction Branches | stickyNote | Canvas section note for extraction paths |  |  | ## Extraction Branches<br>Extracts raw text from each supported file type. - **PDF** — native Extract from File node - **TXT** — native Extract from File node - **CSV** — parsed to JSON rows - **Image** — NVIDIA Llama 3.2 vision model via HTTP - **Google Doc** — pre-fetched upstream, normalised here<br>All paths converge on a unified `content` field passed downstream. |
| Section - Fallback Paths | stickyNote | Canvas section note for fallback handling |  |  | ## Fallback Paths<br>Raw text inputs bypass extraction entirely.<br>Unsupported MIME types send a Telegram notification and stop — no Sheets row is written. |
| Section - AI Structuring | stickyNote | Canvas section note for NVIDIA structuring stage |  |  | ## AI Structuring with NVIDIA NIM<br>Sends extracted text to `nvidia/llama-3.3-nemotron-super-49b-v1.5` with a guided JSON schema. Returns: title, summary, category, language, key points, and confidence notes. Falls back gracefully if the model call fails (`continueOnFail: true`).<br>**Setup:** Add your NVIDIA NIM API key as an HTTP Header Auth credential (`Authorization: Bearer YOUR_KEY`). Apply it to both NVIDIA HTTP Request nodes. |
| Section - Log and Notify | stickyNote | Canvas section note for output delivery |  |  | ## Step 3 — Delivery<br>This workflow delivers output in two places:<br>- `Append Row in Sheet` writes a structured row to the `Extract_Log` tab - `Send Reply` sends a short Telegram confirmation message<br>Before going live: - Replace `REPLACE_WITH_GOOGLE_SHEET_ID` in `Append Row in Sheet` - Confirm the target sheet contains an `Extract_Log` tab - Confirm the sheet headers match the mapped fields in the node - Confirm Telegram replies go to the configured chat ID |
| Sticky Note | stickyNote | Canvas credentials note |  |  | ## Step 2 — Credentials<br>Create and connect these credentials before testing:<br>- Google Drive OAuth2 account — used by `Google Drive Trigger` and `Download Drive File` - Google Docs OAuth2 account — used by `Get Google Doc` - Google Sheets OAuth2 account — used by `Append Row in Sheet` - Telegram credential — used by `Send Reply` and `Send Unsupported Message` - HTTP Header Auth credential for NVIDIA NIM — set `Authorization: Bearer YOUR_API_KEY` and apply it to both NVIDIA HTTP Request nodes<br>After connecting credentials, re-open each node once and confirm the correct account is selected. |
| Sticky Note1 | stickyNote | Canvas activation checklist note |  |  | ## Step 4 — Activate<br>Before activating the workflow, test one file at a time in this order:<br>- PDF - text file - CSV - image - Google Doc<br>For each test, confirm: - the correct branch ran - extracted text was produced - structured output was returned - a row was appended to `Extract_Log` - the Telegram reply was sent successfully<br>Activate the workflow only after all supported file paths pass at least one full end-to-end test. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it something like:  
   `Extract text from Google Drive files to Google Sheets using NVIDIA NIM`.

2. **Add a Google Drive Trigger node** named `Google Drive Trigger`.
   - Set **Event** to `File Created`.
   - Set **Trigger On** to `Specific Folder`.
   - Set the folder ID to your target Google Drive folder.
   - Set polling to **Every Minute**.
   - Connect a **Google Drive OAuth2** credential with access to that folder.

3. **Add a Code node** named `Normalize Input` after the trigger.
   - Use JavaScript mode.
   - Configure it to:
     - read `id`, `mimeType`, and `name` from the trigger payload
     - compute `route`
     - set `googleDocId` if the MIME type is Google Docs
     - initialize `rawText`, `messageId`, and `binaryKey` as empty strings
     - set `chatId` to your Telegram destination
     - set `runId` to `$execution.id`
   - Use this route logic:
     - PDF → `pdf`
     - image MIME types → `image`
     - Google Docs MIME type → `google_doc`
     - CSV MIME type or `.csv` extension → `csv`
     - other `text/*` MIME types → `text_file`
     - otherwise → `unsupported`

4. **Add a Switch node** named `Route Input`.
   - Match on `{{ $json.route }}`
   - Create outputs for:
     - `pdf`
     - `image`
     - `google_doc`
     - `text_file`
     - `csv`
     - `raw_text`
   - Wire:
     - `pdf` → `Download Drive File`
     - `image` → `Download Drive File`
     - `google_doc` → `Get Google Doc`
     - `text_file` → `Download Drive File`
     - `csv` → `Download Drive File`
     - fallback/unsupported → `Send Unsupported Message`

5. **Add a Google Drive node** named `Download Drive File`.
   - Operation: `Download`
   - File ID: `{{ $json.fileId }}`
   - In options, set file name to `{{ $json.fileName }}`
   - Reuse the **Google Drive OAuth2** credential.

6. **Add a second Switch node** named `Route Downloaded File`.
   - Match on `{{ $('Normalize Input').item.json.route }}`
   - Create outputs:
     - `pdf`
     - `image`
     - `text_file`
     - `csv`
   - Wire outputs to:
     - `pdf` → `Extract from PDF`
     - `image` → `Prepare Image Data URL`
     - `text_file` → `Extract from Text File`
     - `csv` → `Extract from CSV`

7. **Add a Google Docs node** named `Get Google Doc`.
   - Operation: `Get`
   - Set the document field using the normalized Google Doc ID.
   - In this workflow JSON, the expression is `{{ $('Normalize Input').first().json.googleDocId }}`
   - Connect a **Google Docs OAuth2** credential.
   - Wire it from the `google_doc` output of `Route Input`.

8. **Add a Code node** named `Normalize Google Doc`.
   - Implement a helper that recursively searches for the longest string in the Google Docs response.
   - Set output fields:
     - `content`
     - `fileType: google_doc`
     - `fileName`
     - `chatId`
     - `messageId`
     - `sourceType: google_drive_google_doc`
     - `runId`
   - Wire to `Build Structuring Payload`.

9. **Add an Extract from File node** named `Extract from PDF`.
   - Set operation to `PDF`.
   - Wire from the `pdf` output of `Route Downloaded File`.

10. **Add a Code node** named `Normalize PDF`.
    - Reuse the same longest-string helper pattern.
    - Output:
      - `content`
      - `fileType: pdf`
      - `fileName`
      - `chatId`
      - `messageId`
      - `sourceType: google_drive_pdf`
      - `runId`
    - Wire to `Build Structuring Payload`.

11. **Add a Code node** named `Prepare Image Data URL`.
    - Read binary content from the downloaded file.
    - Determine the binary key from:
      - `Normalize Input.binaryKey`, or
      - the first available binary key
    - Load the buffer with `this.helpers.getBinaryDataBuffer(...)`
    - Convert to base64
    - Build a `data:<mime>;base64,...` URL
    - Output:
      - `imageDataUrl`
      - `fileType: image`
      - `fileName`
      - `chatId`
      - `messageId`
      - `sourceType: google_drive_image`
      - `runId`

12. **Add a Code node** named `Build Image Payload`.
    - Build a JSON payload for NVIDIA vision OCR.
    - Use model: `meta/llama-3.2-11b-vision-instruct`
    - Add:
      - a strict system prompt telling the model to return plain text only
      - a user message containing instruction text plus the `image_url`
    - Set:
      - `temperature: 0`
      - `top_p: 1`
      - `max_tokens: 1800`
      - `stream: false`
    - Store the payload as `nimPayload` using `JSON.stringify(...)`

13. **Create an HTTP Header Auth credential** for NVIDIA NIM.
    - Header name: `Authorization`
    - Header value: `Bearer YOUR_API_KEY`

14. **Add an HTTP Request node** named `Analyze Image with NVIDIA`.
    - Method: `POST`
    - URL: `https://integrate.api.nvidia.com/v1/chat/completions`
    - Authentication: `Generic Credential Type`
    - Generic Auth Type: `HTTP Header Auth`
    - Select the NVIDIA credential from step 13
    - Send headers: enabled
    - Add header `Content-Type: application/json`
    - Send body: enabled
    - Content type: `Raw`
    - Raw content type: `application/json`
    - Body: `{{ $json.nimPayload }}`
    - Wire to `Normalize Image`

15. **Add a Code node** named `Normalize Image`.
    - Extract text from `choices[0].message.content`
    - Support both:
      - plain string content
      - array content with text parts
    - Remove code fences
    - If response is empty or `NO_TEXT_EXTRACTED`, replace with `No text extracted`
    - Output normalized fields and wire to `Build Structuring Payload`

16. **Add an Extract from File node** named `Extract from Text File`.
    - Set operation to `Text`
    - Wire from the `text_file` output of `Route Downloaded File`

17. **Add a Code node** named `Normalize Text File`.
    - Reuse the longest-string helper pattern
    - Output:
      - `content`
      - `fileType: text_file`
      - `fileName`
      - `chatId`
      - `messageId`
      - `sourceType: google_drive_text_file`
      - `runId`
    - Wire to `Build Structuring Payload`

18. **Add an Extract from File node** named `Extract from CSV`.
    - Leave default extraction settings, or configure CSV parsing if your n8n version requires it explicitly.
    - Wire from the `csv` output of `Route Downloaded File`

19. **Add a Code node** named `Normalize CSV`.
    - Gather all CSV row items with `$input.all()`
    - Map rows to `item.json`
    - Serialize with `JSON.stringify(rows, null, 2)`
    - Output:
      - `content`
      - `fileType: csv`
      - `fileName`
      - `chatId`
      - `messageId`
      - `sourceType: google_drive_csv`
      - `runId`
    - Wire to `Build Structuring Payload`

20. **Add a Code node** named `Normalize Raw Text`.
    - This is optional for future extension.
    - Set:
      - `content` from `Normalize Input.rawText`
      - `fileType: raw_text`
      - `fileName: telegram-message`
      - `chatId`
      - `messageId`
      - `sourceType: google_drive_text`
      - `runId`
    - Wire to `Build Structuring Payload`
    - Note: in the current design, this node is not actively reached.

21. **Add a Telegram node** named `Send Unsupported Message`.
    - Operation: send message
    - Chat ID: `{{ $('Normalize Input').first().json.chatId }}`
    - Text:
      - `Unsupported Google Drive file.`
      - blank line
      - `Supported types: PDF, image, Google Docs, text file, and CSV.`
    - Disable attribution
    - Connect your **Telegram bot credential**

22. **Add a Code node** named `Build Structuring Payload`.
    - Accept input from all normalized branches.
    - Read `input.content`
    - Trim it to `12000` characters before sending to the model
    - Build a guided JSON schema with fields:
      - `title`
      - `summary`
      - `category`
      - `language`
      - `key_points` as array of strings
      - `confidence_notes`
    - Use model `nvidia/llama-3.3-nemotron-super-49b-v1.5`
    - Add a system prompt instructing the model to:
      - structure text only
      - return schema fields only
      - avoid reasoning
      - avoid markdown
      - avoid hallucinations
      - use empty values when unknown
    - Store the stringified payload in `nimPayload`
    - Preserve original extracted text as `originalContent`

23. **Add an HTTP Request node** named `Structure Output with NVIDIA`.
    - Method: `POST`
    - URL: `https://integrate.api.nvidia.com/v1/chat/completions`
    - Use the same **HTTP Header Auth** NVIDIA credential
    - Set `Content-Type: application/json`
    - Body: `{{ $json.nimPayload }}`
    - Enable `Continue On Fail`
    - Wire to `Parse Structured Output`

24. **Add a Code node** named `Parse Structured Output`.
    - Implement a safe parser that:
      - tries direct `JSON.parse`
      - if needed, extracts the first JSON object from a text response
    - Remove `<think>...</think>` tags before parsing
    - Populate fallback values:
      - `title` = parsed title or file name or `Untitled`
      - `summary` = parsed summary or first 300 chars of original content
      - `category` = parsed category or empty string
      - `language` = parsed language or empty string
      - `key_points` = parsed array or `[]`
      - `confidence_notes` = parsed notes or empty string
    - Also output:
      - `extracted_text`
      - `sourceType`
      - `fileType`
      - `fileName`
      - `chatId`
      - `messageId`
      - `runId`

25. **Prepare the Google Sheet** before creating the Sheets node.
    - Create a spreadsheet.
    - Add a tab named `Extract_Log`.
    - Create these headers exactly:
      - `Timestamp`
      - `Source Type`
      - `File Type`
      - `File Name`
      - `Title`
      - `Summary`
      - `Category`
      - `Language`
      - `Key Points`
      - `Confidence Notes`
      - `Extracted Text`
      - `Run ID`

26. **Add a Google Sheets node** named `Append Row in Sheet`.
    - Operation: `Append`
    - Spreadsheet ID: your Google Sheet ID
    - Sheet name: `Extract_Log`
    - Mapping mode: define below
    - Map:
      - `Title` ← `{{ $json.title }}`
      - `Run ID` ← `{{ $json.runId }}`
      - `Summary` ← `{{ $json.summary }}`
      - `Category` ← `{{ $json.category }}`
      - `Language` ← `{{ $json.language }}`
      - `File Name` ← `{{ $json.fileName }}`
      - `File Type` ← `{{ $json.fileType }}`
      - `Timestamp` ← `{{ new Date().toISOString() }}`
      - `Key Points` ← `{{ ($json.key_points || []).join(' | ') }}`
      - `Source Type` ← `{{ $json.sourceType }}`
      - `Extracted Text` ← `{{ $json.extracted_text }}`
      - `Confidence Notes` ← `{{ $json.confidence_notes }}`
    - Connect a **Google Sheets OAuth2** credential
    - Enable `Continue On Fail`

27. **Add a Code node** named `Build Telegram Reply`.
    - Build a short message with:
      - `FILE PROCESSED`
      - file name
      - file type
      - title
      - category
      - final status
    - Truncate to 3500 characters if needed
    - Output:
      - `replyText`
      - `chatId`
      - `messageId`

28. **Add a Telegram node** named `Send Reply`.
    - Text: `{{ $json.replyText }}`
    - Chat ID: `{{ $json.chatId }}`
    - Disable attribution
    - Use the same **Telegram bot credential**

29. **Connect the final output paths** from `Parse Structured Output`.
    - `Parse Structured Output` → `Append Row in Sheet`
    - `Parse Structured Output` → `Build Telegram Reply` → `Send Reply`

30. **Reconnect all branches to the structuring node**.
    - `Normalize Google Doc` → `Build Structuring Payload`
    - `Normalize PDF` → `Build Structuring Payload`
    - `Normalize Image` → `Build Structuring Payload`
    - `Normalize Text File` → `Build Structuring Payload`
    - `Normalize CSV` → `Build Structuring Payload`
    - `Normalize Raw Text` → `Build Structuring Payload`

31. **Replace all placeholders**.
    - Google Drive folder ID
    - Telegram chat ID
    - Google Sheet ID
    - NVIDIA API key in the HTTP Header Auth credential

32. **Validate credentials**.
    - Google Drive OAuth2 for:
      - `Google Drive Trigger`
      - `Download Drive File`
    - Google Docs OAuth2 for:
      - `Get Google Doc`
    - Google Sheets OAuth2 for:
      - `Append Row in Sheet`
    - Telegram credential for:
      - `Send Reply`
      - `Send Unsupported Message`
    - HTTP Header Auth for:
      - `Analyze Image with NVIDIA`
      - `Structure Output with NVIDIA`

33. **Test supported file types one by one**.
    - PDF
    - text file
    - CSV
    - image
    - Google Doc

34. **Verify expected outcomes for each test**.
    - correct route selected
    - extracted text present
    - structured JSON fields populated or safely defaulted
    - row appended to `Extract_Log`
    - Telegram success message sent

35. **Activate the workflow** only after all supported branches pass end-to-end.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Built by Cordexa Technologies | https://cordexa.tech |
| Contact email from workflow note | cordexatech@gmail.com |
| Folder ID placeholder must be replaced in `Google Drive Trigger` | Workflow setup |
| Telegram chat ID placeholder must be replaced in `Normalize Input` | Workflow setup |
| Google Sheet ID placeholder must be replaced in `Append Row in Sheet` | Workflow setup |
| NVIDIA NIM auth must be configured as HTTP Header Auth with `Authorization: Bearer YOUR_API_KEY` | Used by both NVIDIA HTTP Request nodes |
| The workflow includes a `raw_text` branch scaffold, but the current Google Drive trigger path never sets `route = raw_text` | Design note |
| Unsupported MIME types send Telegram only and do not write to Google Sheets | Runtime behavior |
| Recommended test order before activation: PDF, text file, CSV, image, Google Doc | Activation checklist |