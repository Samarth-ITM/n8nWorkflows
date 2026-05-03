Process receipt images from Google Drive into Google Sheets using OpenAI

https://n8nworkflows.xyz/workflows/process-receipt-images-from-google-drive-into-google-sheets-using-openai-12954


# Process receipt images from Google Drive into Google Sheets using OpenAI

## 1. Workflow Overview

**Purpose:** Automatically process receipt images uploaded to a specific Google Drive folder: download the file, extract structured receipt fields with OpenAI (via an AI Agent with a strict JSON schema), append the extracted data into a Google Sheet, convert the result into an HTML table, and email a notification with the summary.

**Target use cases:** Personal or business expense tracking, automating bookkeeping intake, reducing manual data entry for receipts.

### 1.1 Input Reception (Google Drive monitoring)
Watches a dedicated “Receipts” folder in Google Drive and triggers on each newly created file.

### 1.2 File Retrieval (Download binary)
Downloads the newly detected Drive file so it can be sent to the AI node as an image.

### 1.3 AI Extraction (Vision + structured JSON)
Sends the receipt image to an OpenAI chat model through an AI Agent, enforcing a strict JSON structure using a structured output parser.

### 1.4 Persistence (Google Sheets logging)
Appends the extracted receipt fields (vendor/date/items/total) as a new row in a target Google Sheet.

### 1.5 Reporting (HTML + Email)
Converts the appended row data into an HTML table and emails it to a configured recipient.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception (Google Drive monitoring)
**Overview:** Triggers the workflow when a new file is created inside a specific Google Drive folder.  
**Nodes involved:** `Monitor Receipts Folder`

#### Node: Monitor Receipts Folder
- **Type / role:** `googleDriveTrigger` — event trigger (polling) for new files.
- **Configuration (interpreted):**
  - Watches a **specific folder** (`FOLDER_ID_PLACEHOLDER`, displayed name “Receipts”).
  - **Event:** `fileCreated`
  - **Polling frequency:** every minute
- **Key variables/expressions:** none (trigger emits file metadata).
- **Inputs/outputs:**
  - **Output →** `Download file` (main)
- **Credentials:** Google Drive OAuth2 (“Google Drive account”).
- **Version notes:** Node typeVersion `1` (polling trigger behavior depends on n8n trigger implementation).
- **Edge cases / failures:**
  - Polling may **miss/duplicate** events if files are rapidly added/renamed or if Drive latency occurs.
  - Permission issues if the OAuth account lacks access to the folder.
  - Non-image files may still trigger (no filtering is applied here).

---

### Block 2 — File Retrieval (Download binary)
**Overview:** Downloads the newly created file from Drive so downstream AI can process it as an image binary.  
**Nodes involved:** `Download file`

#### Node: Download file
- **Type / role:** `googleDrive` — file download operation.
- **Configuration (interpreted):**
  - **Operation:** Download
  - **File ID:** from trigger output: `={{ $json.id }}`
- **Key variables/expressions:**
  - `={{ $json.id }}` resolves to the Drive file ID emitted by the trigger.
- **Inputs/outputs:**
  - **Input ←** `Monitor Receipts Folder`
  - **Output →** `Extract Receipt Data with AI` (main)
- **Credentials:** Google Drive OAuth2 (“Google Drive account”).
- **Version notes:** typeVersion `3` (download/binary handling differs across versions; keep n8n updated for consistent binary behavior).
- **Edge cases / failures:**
  - Download fails if file is not accessible, is a shortcut, is still uploading, or exceeds size/time limits.
  - If the file is not an image (PDF, text), the AI step may fail or produce poor extraction unless the model/node supports it.

---

### Block 3 — AI Extraction (Vision + structured JSON)
**Overview:** Uses an AI Agent connected to an OpenAI chat model to extract receipt fields from the image and enforces a strict JSON output using a structured output parser.  
**Nodes involved:** `Extract Receipt Data with AI`, `OpenAI Chat Model`, `Structured Output Parser`

#### Node: Extract Receipt Data with AI
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — orchestrates LLM call(s) and applies an output parser.
- **Configuration (interpreted):**
  - **Prompt type:** “define” (custom prompt provided directly in node)
  - **Instruction:** Extract receipt data and output **strict JSON only** with schema:
    - `vendor` (string)
    - `date` (string formatted `DD/MM/YYYY`)
    - `items` (string with one line per item, exact format: `Name: ... - Qty: ... - Total: ...`, separated by newline)
    - `total` (float)
  - **Binary handling:** `passthroughBinaryImages: true` (passes downloaded image binary into the agent context for vision-capable extraction).
  - **Output parsing enabled:** `hasOutputParser: true`
- **Key variables/expressions:** none embedded; relies on upstream binary + prompt + parser.
- **Inputs/outputs:**
  - **Input ←** `Download file` (main)
  - **AI language model input ←** `OpenAI Chat Model` (ai_languageModel connection)
  - **AI output parser input ←** `Structured Output Parser` (ai_outputParser connection)
  - **Output →** `Append Receipt to Google Sheet` (main)
- **Credentials:** none directly (uses connected LLM node credentials).
- **Version notes:** typeVersion `2.2` (LangChain agent behavior and binary vision support can vary by version).
- **Edge cases / failures:**
  - If the receipt image is blurred, rotated, or partial, extracted fields may be missing/incorrect.
  - Model may return invalid JSON; the structured parser can fail the run if it cannot parse/validate.
  - Date normalization to `DD/MM/YYYY` can fail with ambiguous formats (US vs EU).
  - The `items` field is constrained to a single string; long receipts may exceed token or output limits.

#### Node: OpenAI Chat Model
- **Type / role:** `lmChatOpenAi` — provides the chat model for the agent.
- **Configuration (interpreted):**
  - **Model:** `gpt-5-mini`
  - Default options (no custom temperature/max tokens specified in the workflow JSON).
- **Inputs/outputs:**
  - **Output →** `Extract Receipt Data with AI` (ai_languageModel)
- **Credentials:** OpenAI API (“OpenAi account”).
- **Version notes:** typeVersion `1.2`; available models depend on the OpenAI account and n8n’s OpenAI node version.
- **Edge cases / failures:**
  - Invalid/expired API key, model not available to the account, rate limiting, or request size limits (image payloads can be large).

#### Node: Structured Output Parser
- **Type / role:** `outputParserStructured` — enforces/derives structured JSON output.
- **Configuration (interpreted):**
  - Provides a **JSON schema example** used to guide/validate output:
    ```json
    {
      "vendor": "TESCO metro",
      "date": "23/04/2012",
      "items": "Name: FRESH MILK - Qty: 1 - Total: 0.89\nName: MUESLI - Qty: 1 - Total: 2.29\nName: DARK CHOCOLATE - Qty: 2 - Total: 1.90",
      "total": 5.08
    }
    ```
- **Inputs/outputs:**
  - **Output →** `Extract Receipt Data with AI` (ai_outputParser)
- **Version notes:** typeVersion `1.3`.
- **Edge cases / failures:**
  - If the model returns `total` as a string or uses commas as decimal separators, parsing/validation may fail or produce mismatched types.
  - If extra keys appear or JSON is malformed, this node can cause the agent to error.

---

### Block 4 — Persistence (Google Sheets logging)
**Overview:** Appends the extracted receipt fields into a target Google Sheet as a new row.  
**Nodes involved:** `Append Receipt to Google Sheet`

#### Node: Append Receipt to Google Sheet
- **Type / role:** `googleSheets` — append row operation.
- **Configuration (interpreted):**
  - **Operation:** Append
  - **Spreadsheet:** `SPREADSHEET_ID_PLACEHOLDER`
  - **Sheet/tab name:** `Sheet1`
  - **Mapping mode:** Define below (explicit field mapping)
  - **Mapped columns:**
    - `vendor` → `={{ $json.output.vendor }}`
    - `date` → `={{ $json.output.date }}`
    - `items` → `={{ $json.output.items }}`
    - `total` → `={{ $json.output.total }}`
  - Type conversion is disabled (`attemptToConvertTypes: false`)
- **Key variables/expressions:**
  - Uses AI Agent output at `$json.output.*` (meaning the agent returns an object with an `output` property containing the parsed JSON).
- **Inputs/outputs:**
  - **Input ←** `Extract Receipt Data with AI`
  - **Output →** `Convert Receipt to HTML Table`
- **Credentials:** Google Sheets OAuth2 (“vasarmilan sheets”).
- **Version notes:** typeVersion `4.7` (Sheets node UI/mapping differs across versions; ensure compatible mapping fields).
- **Edge cases / failures:**
  - Spreadsheet/sheet name mismatch, missing header columns, or insufficient permissions.
  - If `total` is not a string and the sheet expects numeric (or vice versa), append may still work but formatting can be inconsistent.
  - If the agent failed and `output` is missing, expressions like `$json.output.vendor` will error.

---

### Block 5 — Reporting (HTML + Email)
**Overview:** Converts the appended receipt data into an HTML table and emails it to a configured recipient.  
**Nodes involved:** `Convert Receipt to HTML Table`, `Send Receipt Email Notification`

#### Node: Convert Receipt to HTML Table
- **Type / role:** `html` — transforms JSON into an HTML table representation.
- **Configuration (interpreted):**
  - **Operation:** Convert to HTML table
  - Uses the incoming JSON (from the Sheets append output) to build a table.
- **Inputs/outputs:**
  - **Input ←** `Append Receipt to Google Sheet`
  - **Output →** `Send Receipt Email Notification`
- **Version notes:** typeVersion `1.2`.
- **Edge cases / failures:**
  - The produced table depends on the incoming JSON structure; if Sheets node output is not the desired fields, the HTML may include unexpected columns.
  - Large `items` strings may render as a single cell with embedded newlines.

#### Node: Send Receipt Email Notification
- **Type / role:** `gmail` — sends an email notification.
- **Configuration (interpreted):**
  - **To:** `user@example.com`
  - **Subject:** `=New receipt Processed from {{ $('Append Receipt to Google Sheet').item.json.vendor }}`
    - Uses data from the Sheets node item (`vendor`) to personalize the subject.
  - **Message body:** `={{ $json.table }}`
    - Sends the HTML table generated by the previous node.
  - **Option:** append attribution enabled.
- **Inputs/outputs:**
  - **Input ←** `Convert Receipt to HTML Table`
  - No downstream nodes.
- **Credentials:** Gmail OAuth2 (“john.doe@example.com”).
- **Version notes:** typeVersion `2.1`.
- **Edge cases / failures:**
  - Gmail OAuth token expiration/revocation, restricted scopes, or sending limits.
  - The email body expects `$json.table`; if HTML node outputs under a different key (or multiple items), the message may be empty or error.
  - Subject expression references `Append Receipt to Google Sheet` output; if that node fails, subject rendering fails.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Monitor Receipts Folder | n8n-nodes-base.googleDriveTrigger | Trigger on new receipt files in a Drive folder | — | Download file | ## Workflow Overview This automation eliminates manual receipt data entry by automatically processing receipt images uploaded to Google Drive. When a new receipt is added to a monitored folder, the workflow extracts key information using AI, logs it to a Google Sheet, and sends you an email notification with the processed data. / First Setup + Configuration notes (Drive/OpenAI/Sheets/Gmail; model selection). |
| Download file | n8n-nodes-base.googleDrive | Download the newly created file as binary | Monitor Receipts Folder | Extract Receipt Data with AI | ## Workflow Overview This automation eliminates manual receipt data entry… (setup/config notes). |
| Extract Receipt Data with AI | @n8n/n8n-nodes-langchain.agent | Extract structured receipt fields from image via LLM + parser | Download file (+ OpenAI Chat Model, Structured Output Parser via AI ports) | Append Receipt to Google Sheet | ## Workflow Overview This automation eliminates manual receipt data entry… (setup/config notes). |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM backend for the agent | — | Extract Receipt Data with AI (ai_languageModel) | ## Workflow Overview … “AI model: The workflow uses GPT-4-mini, but you can select a different OpenAI model…” (note: actual node uses gpt-5-mini). |
| Structured Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce strict JSON structure for extracted data | — | Extract Receipt Data with AI (ai_outputParser) | ## Workflow Overview … (setup/config notes). |
| Append Receipt to Google Sheet | n8n-nodes-base.googleSheets | Append extracted receipt fields into Sheets | Extract Receipt Data with AI | Convert Receipt to HTML Table | ## Workflow Overview … (setup/config notes). |
| Convert Receipt to HTML Table | n8n-nodes-base.html | Convert result JSON to an HTML table for email | Append Receipt to Google Sheet | Send Receipt Email Notification | ## Workflow Overview … (setup/config notes). |
| Send Receipt Email Notification | n8n-nodes-base.gmail | Email the processed receipt summary | Convert Receipt to HTML Table | — | ## Workflow Overview … (setup/config notes). |
| Workflow Description | n8n-nodes-base.stickyNote | Documentation / context | — | — | ## Workflow Overview (full text as provided in the sticky note). |
| Video Walkthrough | n8n-nodes-base.stickyNote | Documentation link to video | — | — | # Video Walkthrough [https://youtu.be/Vl0wCc9pEGM](https://youtu.be/Vl0wCc9pEGM) (thumbnail image: https://vasarmilan-public.s3.us-east-1.amazonaws.com/blog_thumbnails/thumbnail_rec5dRs6sFpgUZ675.jpg) |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Trigger: Google Drive Trigger**
   - Add node: **Google Drive Trigger**
   - Set **Event** = *File Created*
   - Set **Trigger On** = *Specific Folder*
   - Select the receipts folder (use your real folder ID instead of `FOLDER_ID_PLACEHOLDER`)
   - Set polling interval to **Every Minute**
   - Configure **Google Drive OAuth2** credentials (Google account with access to the folder).

2. **Download the new file**
   - Add node: **Google Drive**
   - Operation: **Download**
   - **File ID**: set expression to `{{ $json.id }}`
   - Connect: **Monitor Receipts Folder → Download file**
   - Reuse the same **Google Drive OAuth2** credentials.

3. **Add the AI Agent for extraction**
   - Add node: **AI Agent** (LangChain Agent in n8n)
   - Configure prompt (define/custom prompt) to demand **strict JSON only** with fields:
     - `vendor` (string)
     - `date` in `DD/MM/YYYY`
     - `items` as a newline-separated string of rows: `Name: ... - Qty: ... - Total: ...`
     - `total` (float)
   - Enable **binary image passthrough** (so the downloaded image is available to the model).
   - Connect: **Download file → Extract Receipt Data with AI**

4. **Attach an OpenAI Chat Model to the Agent**
   - Add node: **OpenAI Chat Model**
   - Select model: **gpt-5-mini** (or another vision-capable/appropriate model available to your account)
   - Configure **OpenAI API** credentials
   - Connect the model to the agent using the **AI Language Model** connection:
     - **OpenAI Chat Model (ai_languageModel) → Extract Receipt Data with AI**

5. **Attach a Structured Output Parser**
   - Add node: **Structured Output Parser**
   - Provide a JSON example matching your schema (vendor/date/items/total), similar to:
     - vendor string, date `DD/MM/YYYY`, items newline-separated, total numeric
   - Connect using the **AI Output Parser** connection:
     - **Structured Output Parser (ai_outputParser) → Extract Receipt Data with AI**

6. **Append extracted data to Google Sheets**
   - Add node: **Google Sheets**
   - Operation: **Append**
   - Select your spreadsheet (replace `SPREADSHEET_ID_PLACEHOLDER`)
   - Sheet name: **Sheet1** (or your chosen tab)
   - In column mapping (define below), map:
     - `vendor` = `{{ $json.output.vendor }}`
     - `date` = `{{ $json.output.date }}`
     - `items` = `{{ $json.output.items }}`
     - `total` = `{{ $json.output.total }}`
   - Configure **Google Sheets OAuth2** credentials
   - Connect: **Extract Receipt Data with AI → Append Receipt to Google Sheet**

7. **Convert output to an HTML table**
   - Add node: **HTML**
   - Operation: **Convert to HTML Table**
   - Connect: **Append Receipt to Google Sheet → Convert Receipt to HTML Table**

8. **Send notification email via Gmail**
   - Add node: **Gmail**
   - Operation: **Send**
   - To: set your recipient email (replace `user@example.com`)
   - Subject: `New receipt Processed from {{ $('Append Receipt to Google Sheet').item.json.vendor }}`
   - Message/body: `{{ $json.table }}`
   - Configure **Gmail OAuth2** credentials (account allowed to send)
   - Connect: **Convert Receipt to HTML Table → Send Receipt Email Notification**

9. **(Optional) Add documentation sticky notes**
   - Add sticky note “Workflow Description” and paste the overview/setup/config text.
   - Add sticky note “Video Walkthrough” with:
     - https://youtu.be/Vl0wCc9pEGM

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “This automation eliminates manual receipt data entry… Monitor → Download → Extract → Log → Notify” plus setup/configuration bullets (Drive/OpenAI/Sheets/Gmail; update folder, sheet ID, recipient; model selection) | Sticky note: **Workflow Description** |
| Video walkthrough link: https://youtu.be/Vl0wCc9pEGM (thumbnail image: https://vasarmilan-public.s3.us-east-1.amazonaws.com/blog_thumbnails/thumbnail_rec5dRs6sFpgUZ675.jpg) | Sticky note: **Video Walkthrough** |
| Note: Sticky claims “GPT-4-mini”, but the actual configured model node is **gpt-5-mini** | Consistency check between documentation and node configuration |