Track receipt expenses via LINE with OpenAI GPT-4o-mini and Google Sheets

https://n8nworkflows.xyz/workflows/track-receipt-expenses-via-line-with-openai-gpt-4o-mini-and-google-sheets-12680


# Track receipt expenses via LINE with OpenAI GPT-4o-mini and Google Sheets

## 1. Workflow Overview

**Purpose:**  
This workflow lets a user send a **receipt photo via LINE**, uses **OpenAI (GPT-4o-mini)** to extract structured expense data (date, store, amount, category), prevents **duplicate registrations** in **Google Sheets**, stores the receipt image in **Google Drive**, logs the expense to **Google Sheets**, then calculates the **current month total** and returns a short **AI financial advice** via LINE.

**Target use cases:**
- Personal expense tracking by chatting with a LINE bot
- Lightweight receipt OCR + categorization
- Simple bookkeeping pipeline with anti-duplicate check and monthly summary

### 1.1 Input Reception (LINE Webhook)
Receives LINE webhook events and routes only **image messages** to OCR.

### 1.2 OCR & Structured Extraction (OpenAI Agent)
Downloads the image from LINE, sends it to an AI Agent (backed by GPT-4o-mini) with a structured output parser to extract the receipt fields.

### 1.3 Duplicate Check (“Security”)
Checks Google Sheets for duplicates (intended: same date + amount), and branches:
- Duplicate → notify user in LINE
- Not duplicate → store proof + write record

### 1.4 Storage & Intelligence
Saves receipt image to Google Drive, appends a new row in Google Sheets, fetches history, calculates monthly totals, asks AI for brief advice, and replies to the user with confirmation + totals + advice.

---

## 2. Block-by-Block Analysis

### Block 1 — Ingestion & OCR
**Overview:** Receives a LINE event, confirms it’s an image message, downloads the image file, and extracts structured receipt data using an OpenAI-powered agent with schema parsing.

**Nodes involved:**
- LINE Webhook1
- Filter: Is Image Message?
- Download LINE Image File
- AI Agent: Extract Receipt Data
- OpenAI Model: OCR
- Output Parser: Structured Data

#### 2.1 LINE Webhook1
- **Type / role:** Webhook trigger; entry point for LINE Messaging API callbacks.
- **Configuration (interpreted):**
  - HTTP Method: `POST`
  - Path: `line-receipt-tracker`
- **Key data used later:** `body.events[0].message`, `body.events[0].replyToken`, `body.events[0].message.id`
- **Outputs:** Main output to **Filter: Is Image Message?**
- **Edge cases / failures:**
  - LINE signature verification is not implemented here (no X-Line-Signature validation); endpoint may accept spoofed requests unless handled elsewhere (e.g., reverse proxy).
  - LINE can send multiple events; workflow assumes `events[0]` only.

#### 2.2 Filter: Is Image Message?
- **Type / role:** IF node to route only image messages.
- **Configuration:**
  - Condition: `{{$json.body.events[0].message.type}} == "image"`
- **Inputs:** From LINE Webhook1
- **Outputs:** True branch to **Download LINE Image File** (no false-branch handling configured)
- **Edge cases:**
  - Non-image messages are dropped silently (no user feedback).
  - If `events[0]` or `message.type` is missing, strict type validation may cause evaluation failure.

#### 2.3 Download LINE Image File
- **Type / role:** HTTP Request; downloads binary content of the LINE image.
- **Configuration:**
  - URL: `https://api-data.line.me/v2/bot/message/{{ $json.body.events[0].message.id }}/content`
  - Response format: **file** (binary)
  - Header: `Authorization: Bearer YOUR_TOKEN_HERE`
- **Inputs:** From Filter: Is Image Message?
- **Outputs:** Main output to **AI Agent: Extract Receipt Data** (includes binary at `binary.data`)
- **Version-specific notes:** Uses HTTP Request node v4.3 response file handling.
- **Edge cases / failures:**
  - Invalid/expired LINE token → 401/403.
  - LINE message content may expire or be unavailable → 404.
  - Large images or network timeouts.

#### 2.4 AI Agent: Extract Receipt Data
- **Type / role:** LangChain Agent; orchestrates OCR/extraction using a chat model and outputs structured JSON via output parser.
- **Configuration:**
  - User instruction: “Analyze this receipt image and extract the data.”
  - System message: accounting assistant; must return:
    - `date` in `YYYY/MM/DD`
    - `store`
    - `amount` (number only)
    - `category` ∈ `[Food, Daily Needs, Transport, Entertainment, Others]`
  - Output parser enabled (`hasOutputParser: true`)
- **Connections:**
  - **Language model input:** from **OpenAI Model: OCR** via `ai_languageModel`
  - **Output parser:** from **Output Parser: Structured Data** via `ai_outputParser`
  - **Main output:** to **Google Sheets: Check Duplicate**
- **Edge cases / failures:**
  - If the agent is not provided the binary image correctly (depends on how the Agent node interprets incoming binary), extraction may be unreliable. In many designs, a dedicated “vision” input mapping is required; validate that this agent actually uses the downloaded binary.
  - Model may return malformed fields; parser can fail if output deviates from schema.
  - Ambiguous receipts: missing date, multiple totals, handwritten text.

#### 2.5 OpenAI Model: OCR
- **Type / role:** OpenAI Chat Model connector for the agent.
- **Configuration:**
  - Model: `gpt-4o-mini`
- **Credentials:** `OpenAi account 3`
- **Connections:** Feeds **AI Agent: Extract Receipt Data** on `ai_languageModel`
- **Edge cases:**
  - OpenAI auth / quota errors.
  - If vision capability requires specific settings, ensure the node supports image/binary input in your n8n version.

#### 2.6 Output Parser: Structured Data
- **Type / role:** Structured output parser enforcing a JSON schema example.
- **Configuration:**
  - Schema example:
    ```json
    { "date":"2023/10/25", "store":"Store Name", "amount":1250, "category":"Food" }
    ```
- **Connections:** Feeds **AI Agent: Extract Receipt Data** on `ai_outputParser`
- **Edge cases:**
  - Parser failures if the model returns strings for amount, different keys, or extra nesting.

---

### Block 2 — Duplicate Check (“Security”)
**Overview:** Checks the sheet for existing entries and branches to either notify the user or continue storing the new receipt.

**Nodes involved:**
- Google Sheets: Check Duplicate
- Filter: Duplicate Found?
- LINE: Notify Duplicate

#### 2.7 Google Sheets: Check Duplicate
- **Type / role:** Google Sheets node; intended to search/check duplicates.
- **Configuration (as provided):**
  - Document ID: `YOUR_SHEET_ID`
  - Sheet name: **blank** in JSON (must be set)
  - Operation not specified (node defaults vary by version); as-is, this is likely incomplete.
  - `alwaysOutputData: true` enabled
- **Inputs:** From AI Agent: Extract Receipt Data
- **Outputs:** To Filter: Duplicate Found?
- **Edge cases / failures:**
  - Missing `sheetName` will cause runtime failure.
  - Duplicate detection logic is not actually defined here (no filter/search conditions shown). The downstream IF checks `$json.row_number`, which only exists for certain operations (e.g., lookup/read with metadata). You’ll need to configure this node properly (see Section 4).

#### 2.8 Filter: Duplicate Found?
- **Type / role:** IF node; determines whether a duplicate exists.
- **Configuration:**
  - Condition: `{{$json.row_number}} > 0`
- **Inputs:** From Google Sheets: Check Duplicate
- **Outputs:**
  - True → LINE: Notify Duplicate
  - False → Prepare Image Data
- **Edge cases:**
  - If `row_number` is missing/undefined, numeric comparison may fail or evaluate false depending on n8n coercion; current settings use strict validation.

#### 2.9 LINE: Notify Duplicate
- **Type / role:** HTTP Request; replies to the LINE conversation with a duplicate warning.
- **Configuration:**
  - POST `https://api.line.me/v2/bot/message/reply`
  - Authorization: `Bearer YOUR_TOKEN_HERE`
  - JSON body uses expressions:
    - replyToken from `$('LINE Webhook1').item.json.body.events[0].replyToken`
    - date/amount from `$('AI Agent: Extract Receipt Data').first().json.output.date` and `.amount`
- **Inputs:** From Filter: Duplicate Found? (true path)
- **Outputs:** None
- **Edge cases / failures:**
  - Reply token expires quickly; delays in upstream OCR can cause LINE reply to fail.
  - If the agent output is missing `.output.date` or `.output.amount`, message formatting fails.

---

### Block 3 — Storage & Intelligence
**Overview:** If not duplicate, stores the image in Drive, appends the expense row to Sheets, fetches full history, computes current month total, asks AI for advice, and replies in LINE with summary.

**Nodes involved:**
- Prepare Image Data
- Google Drive: Save Proof
- Google Sheets: Add Row
- Google Sheets: Fetch History
- Calculate Monthly Totals
- AI Agent: Business Advice
- OpenAI Model: Advice
- LINE: Final Confirmation

#### 2.10 Prepare Image Data
- **Type / role:** Set node; ensures the downloaded binary is in field `binary.data` for downstream nodes.
- **Configuration:**
  - Adds/overwrites binary field `data` with:
    - `={{ $('Download LINE Image File').item.binary.data }}`
  - Keeps other fields (`includeOtherFields: true`)
- **Inputs:** From Filter: Duplicate Found? (false path)
- **Outputs:** To Google Drive: Save Proof
- **Edge cases:**
  - If Download LINE Image File did not produce `binary.data`, this assignment fails.

#### 2.11 Google Drive: Save Proof
- **Type / role:** Uploads the receipt image to Google Drive (proof storage).
- **Configuration:**
  - Drive: `My Drive`
  - Folder ID: `YOUR_FOLDER_ID`
  - (File naming is not specified in the JSON; default behavior may create a generic name unless configured in UI.)
- **Credentials:** Google Drive OAuth2
- **Inputs:** From Prepare Image Data (expects binary)
- **Outputs:** To Google Sheets: Add Row
- **Edge cases / failures:**
  - Missing file name/mime type can produce unexpected Drive file metadata.
  - OAuth scope/auth issues; folder permission issues.

#### 2.12 Google Sheets: Add Row
- **Type / role:** Appends a new expense row.
- **Configuration:**
  - Operation: `append`
  - Document ID: `YOUR_SHEET_ID`
  - Sheet name: **blank** in JSON (must be set)
  - Column mapping is not shown; must map extracted fields to `Date`, `Amount`, `Store`, `Category` (and optionally Drive file URL/id).
- **Inputs:** From Google Drive: Save Proof
- **Outputs:** To Google Sheets: Fetch History
- **Edge cases:**
  - If columns don’t match header names exactly, append may misalign or fail.
  - Date format differences can break later monthly calculation.

#### 2.13 Google Sheets: Fetch History
- **Type / role:** Reads sheet data for computing monthly totals.
- **Configuration:**
  - Document ID: `YOUR_SHEET_ID`
  - Sheet name: **blank** in JSON (must be set)
  - Operation not shown; assumed “read/get all rows”.
- **Inputs:** From Google Sheets: Add Row
- **Outputs:** To Calculate Monthly Totals
- **Edge cases:**
  - Large sheets can be slow; may hit API quotas or execution timeouts.

#### 2.14 Calculate Monthly Totals
- **Type / role:** Code node; sums amounts in the current month and prepares summary fields.
- **Key logic:**
  - Determines current month/year from `new Date()`
  - Reads current extracted receipt from: `$('AI Agent: Extract Receipt Data').item.json.output`
  - Iterates through all input rows (`$input.all()`):
    - Expects columns `Date` and `Amount`
    - Parses date by replacing `/` with `-`
    - Sums `Amount` for rows in current month/year
  - Returns:
    - `header: "✅ Entry Successful"`
    - `monthlyTotal`
    - `store`, `amount` (from current receipt)
- **Inputs:** From Google Sheets: Fetch History
- **Outputs:** To AI Agent: Business Advice
- **Edge cases / failures:**
  - `new Date("YYYY/MM/DD")` parsing is locale/engine dependent; the code normalizes to `YYYY-MM-DD` which is safer, but still verify consistency.
  - Amount values like `"1,250"` will convert to `NaN` with `Number()` unless cleaned.
  - If Sheet columns are named differently (e.g., lowercase), totals will be zero.

#### 2.15 AI Agent: Business Advice
- **Type / role:** LangChain Agent; generates one brief financial insight.
- **Configuration:**
  - Prompt (uses values from previous code node):  
    `Current: {{ $json.amount }} JPY at {{ $json.store }}. Total this month: {{ $json.monthlyTotal }} JPY. Provide one brief financial advice.`
  - System: “professional business consultant… one sharp, short advice”
- **Connections:**
  - **Language model:** from **OpenAI Model: Advice**
  - **Main output:** to LINE: Final Confirmation
- **Edge cases:**
  - If monthlyTotal is very large, keep response short to avoid LINE message limits.

#### 2.16 OpenAI Model: Advice
- **Type / role:** OpenAI Chat Model connector for advice generation.
- **Configuration:** Model `gpt-4o-mini`
- **Credentials:** `OpenAi account 3`
- **Connections:** Feeds AI Agent: Business Advice on `ai_languageModel`

#### 2.17 LINE: Final Confirmation
- **Type / role:** HTTP Request; replies to LINE with confirmation, monthly total, and AI advice.
- **Configuration:**
  - POST `https://api.line.me/v2/bot/message/reply`
  - Authorization: `Bearer YOUR_TOKEN_HERE`
  - Body expressions:
    - replyToken from webhook
    - header/monthlyTotal from `Calculate Monthly Totals`
    - AI advice text from current node input: `{{ $json.output }}`
  - Uses `toLocaleString()` for monthly total formatting.
- **Inputs:** From AI Agent: Business Advice
- **Edge cases / failures:**
  - If advice agent output key is not `output`, message will be blank or expression fails.
  - Reply token expiration remains a risk if workflow is slow.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| LINE Webhook1 | Webhook | Receive LINE events | — | Filter: Is Image Message? | ### Phase 1: Ingestion & OCR |
| Filter: Is Image Message? | IF | Allow only image messages | LINE Webhook1 | Download LINE Image File | ### Phase 1: Ingestion & OCR |
| Download LINE Image File | HTTP Request | Download receipt image binary from LINE | Filter: Is Image Message? | AI Agent: Extract Receipt Data | ### Phase 1: Ingestion & OCR |
| AI Agent: Extract Receipt Data | LangChain Agent | Extract date/store/amount/category | Download LINE Image File | Google Sheets: Check Duplicate | ### Phase 1: Ingestion & OCR |
| OpenAI Model: OCR | OpenAI Chat Model | LLM backend for receipt extraction | — (AI connection) | AI Agent: Extract Receipt Data | ### Phase 1: Ingestion & OCR |
| Output Parser: Structured Data | Structured Output Parser | Enforce structured JSON output | — (AI connection) | AI Agent: Extract Receipt Data | ### Phase 1: Ingestion & OCR |
| Google Sheets: Check Duplicate | Google Sheets | Check for existing entry | AI Agent: Extract Receipt Data | Filter: Duplicate Found? | ### Phase 2: Security |
| Filter: Duplicate Found? | IF | Branch if duplicate exists | Google Sheets: Check Duplicate | LINE: Notify Duplicate / Prepare Image Data | ### Phase 2: Security |
| LINE: Notify Duplicate | HTTP Request | Reply duplicate warning via LINE | Filter: Duplicate Found? (true) | — | ### Phase 2: Security |
| Prepare Image Data | Set | Ensure binary field for Drive upload | Filter: Duplicate Found? (false) | Google Drive: Save Proof | ### Phase 3: Storage & Intelligence |
| Google Drive: Save Proof | Google Drive | Archive receipt image | Prepare Image Data | Google Sheets: Add Row | ### Phase 3: Storage & Intelligence |
| Google Sheets: Add Row | Google Sheets | Append expense record | Google Drive: Save Proof | Google Sheets: Fetch History | ### Phase 3: Storage & Intelligence |
| Google Sheets: Fetch History | Google Sheets | Read all rows for monthly totals | Google Sheets: Add Row | Calculate Monthly Totals | ### Phase 3: Storage & Intelligence |
| Calculate Monthly Totals | Code | Sum current month expenses + prep summary | Google Sheets: Fetch History | AI Agent: Business Advice | ### Phase 3: Storage & Intelligence |
| AI Agent: Business Advice | LangChain Agent | Generate one brief spending advice | Calculate Monthly Totals | LINE: Final Confirmation | ### Phase 3: Storage & Intelligence |
| OpenAI Model: Advice | OpenAI Chat Model | LLM backend for advice | — (AI connection) | AI Agent: Business Advice | ### Phase 3: Storage & Intelligence |
| LINE: Final Confirmation | HTTP Request | Reply success + monthly total + advice | AI Agent: Business Advice | — | ### Phase 3: Storage & Intelligence |
| Sticky Note | Sticky Note | Comment | — | — | ### Phase 1: Ingestion & OCR |
| Sticky Note1 | Sticky Note | Comment | — | — | ### Phase 2: Security |
| Sticky Note2 | Sticky Note | Comment | — | — | ### Phase 3: Storage & Intelligence |
| Workflow Documentation1 | Sticky Note | Comment / embedded project notes | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Webhook Trigger**
   1. Add **Webhook** node named `LINE Webhook1`
   2. Set:
      - Method: `POST`
      - Path: `line-receipt-tracker`
   3. Deploy/activate workflow and copy the production URL into LINE Messaging API webhook settings.

2) **Filter only image messages**
   1. Add **IF** node `Filter: Is Image Message?`
   2. Condition (String → Equals):
      - Left: `{{$json.body.events[0].message.type}}`
      - Right: `image`
   3. Connect: `LINE Webhook1` → `Filter: Is Image Message?` (main)

3) **Download image from LINE**
   1. Add **HTTP Request** node `Download LINE Image File`
   2. Set:
      - Method: `GET`
      - URL: `https://api-data.line.me/v2/bot/message/{{$json.body.events[0].message.id}}/content`
      - Response: **File** (binary)
      - Header: `Authorization: Bearer <YOUR_LINE_CHANNEL_ACCESS_TOKEN>`
   3. Connect: `Filter: Is Image Message? (true)` → `Download LINE Image File`

4) **Configure OpenAI receipt extraction agent**
   1. Add **OpenAI Chat Model** node `OpenAI Model: OCR`
      - Model: `gpt-4o-mini`
      - Credential: OpenAI API key credential
   2. Add **Structured Output Parser** node `Output Parser: Structured Data`
      - Provide schema example:
        - `date` (YYYY/MM/DD), `store` (string), `amount` (number), `category` (enum)
   3. Add **AI Agent** node `AI Agent: Extract Receipt Data`
      - Prompt: “Analyze this receipt image and extract the data.”
      - System message: enforce fields + category list
      - Enable output parser
   4. Wire AI connections:
      - `OpenAI Model: OCR` → `AI Agent: Extract Receipt Data` via **ai_languageModel**
      - `Output Parser: Structured Data` → `AI Agent: Extract Receipt Data` via **ai_outputParser**
   5. Connect main flow:
      - `Download LINE Image File` → `AI Agent: Extract Receipt Data`

   **Important:** Ensure the Agent node is actually receiving the image (binary) as context. Depending on your n8n/LangChain node version, you may need to map binary/image explicitly (e.g., use a node that converts binary to an “image” message part, or use an agent/tool that supports vision inputs). Validate by testing with one receipt.

5) **Set up Google Sheets duplicate check**
   1. Add **Google Sheets** node `Google Sheets: Check Duplicate`
      - Credential: Google Sheets OAuth2
      - Document ID: your spreadsheet ID
      - Sheet name: your target tab (e.g., `Expenses`)
      - Configure it to **lookup/search** for existing rows using extracted data:
        - Date equals `{{$('AI Agent: Extract Receipt Data').item.json.output.date}}`
        - Amount equals `{{$('AI Agent: Extract Receipt Data').item.json.output.amount}}`
      - Ensure the node outputs a field you can test (commonly: matched rows length, or row number if using “Lookup”).
   2. Connect: `AI Agent: Extract Receipt Data` → `Google Sheets: Check Duplicate`

6) **Branch on duplicate**
   1. Add **IF** node `Filter: Duplicate Found?`
      - Condition should match what your lookup returns. To mirror the provided workflow:
        - Number > 0: Left `{{$json.row_number}}` Right `0`
      - If your lookup returns an array instead, use e.g. `{{$json.length}} > 0` (adjust accordingly).
   2. Connect: `Google Sheets: Check Duplicate` → `Filter: Duplicate Found?`

7) **If duplicate: reply on LINE**
   1. Add **HTTP Request** node `LINE: Notify Duplicate`
      - Method: POST
      - URL: `https://api.line.me/v2/bot/message/reply`
      - Headers: `Authorization: Bearer <LINE_TOKEN>`
      - Body (JSON):
        - replyToken: `{{$('LINE Webhook1').item.json.body.events[0].replyToken}}`
        - message text referencing extracted `date` and `amount`
   2. Connect: `Filter: Duplicate Found? (true)` → `LINE: Notify Duplicate`

8) **If not duplicate: prepare binary for Drive**
   1. Add **Set** node `Prepare Image Data`
      - Add binary field `data`:
        - Value: `{{$('Download LINE Image File').item.binary.data}}`
      - Keep other fields
   2. Connect: `Filter: Duplicate Found? (false)` → `Prepare Image Data`

9) **Save receipt image to Google Drive**
   1. Add **Google Drive** node `Google Drive: Save Proof`
      - Credential: Google Drive OAuth2
      - Drive: My Drive (or select)
      - Folder ID: target folder for receipts
      - Configure upload from binary property `data`
      - (Recommended) Set file name expression (e.g., `{{$('AI Agent: Extract Receipt Data').item.json.output.date}}_{{$('AI Agent: Extract Receipt Data').item.json.output.amount}}.jpg`)
   2. Connect: `Prepare Image Data` → `Google Drive: Save Proof`

10) **Append row to Google Sheets**
   1. Add **Google Sheets** node `Google Sheets: Add Row`
      - Operation: Append
      - Document ID + Sheet name set
      - Map columns:
        - `Date` ← extracted `date`
        - `Amount` ← extracted `amount`
        - `Store` ← extracted `store`
        - `Category` ← extracted `category`
        - (Optional) `DriveFileId/Url` from Drive node output
   2. Connect: `Google Drive: Save Proof` → `Google Sheets: Add Row`

11) **Fetch history**
   1. Add **Google Sheets** node `Google Sheets: Fetch History`
      - Operation: Read/Get All (entire sheet)
      - Document ID + Sheet name
   2. Connect: `Google Sheets: Add Row` → `Google Sheets: Fetch History`

12) **Compute current month totals**
   1. Add **Code** node `Calculate Monthly Totals`
   2. Paste the provided logic (optionally enhance amount cleaning for commas)
   3. Connect: `Google Sheets: Fetch History` → `Calculate Monthly Totals`

13) **Generate advice with OpenAI**
   1. Add **OpenAI Chat Model** node `OpenAI Model: Advice` (model `gpt-4o-mini`, same credential)
   2. Add **AI Agent** node `AI Agent: Business Advice`
      - Prompt uses `amount`, `store`, `monthlyTotal`
      - System message: short, sharp advice
   3. Connect:
      - `Calculate Monthly Totals` → `AI Agent: Business Advice` (main)
      - `OpenAI Model: Advice` → `AI Agent: Business Advice` via **ai_languageModel**

14) **Reply final confirmation to LINE**
   1. Add **HTTP Request** node `LINE: Final Confirmation`
      - POST `https://api.line.me/v2/bot/message/reply`
      - Header Authorization bearer token
      - JSON body:
        - replyToken from webhook
        - text includes:
          - header/monthlyTotal from `Calculate Monthly Totals`
          - advice from `AI Agent: Business Advice` output
   2. Connect: `AI Agent: Business Advice` → `LINE: Final Confirmation`

15) **Credentials required**
   - **OpenAI**: API key with access to `gpt-4o-mini`
   - **Google Sheets OAuth2**: access to the spreadsheet
   - **Google Drive OAuth2**: access to receipt folder
   - **LINE Channel Access Token**: used as Bearer token in HTTP nodes

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “## 🚀 Manage expenses with AI insights through LINE … Setup steps … Replace placeholder IDs in the Google Sheets, Google Drive, and HTTP Request nodes.” | From sticky note “Workflow Documentation1” (embedded project notes) |
| LINE setup: Messaging API Channel + Channel Access Token; enable webhook and point it to `/line-receipt-tracker` | Mentioned in Workflow Documentation1 |
| Google Sheets required columns: `Date`, `Amount`, `Store`, `Category` | Mentioned in Workflow Documentation1 |
| Google Drive: create a folder and place its ID in the Drive node | Mentioned in Workflow Documentation1 |

**Disclaimer (provided by user):**  
Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.