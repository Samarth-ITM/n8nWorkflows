Track monthly OpenAI token usage with Google Sheets and Gmail reports

https://n8nworkflows.xyz/workflows/track-monthly-openai-token-usage-with-google-sheets-and-gmail-reports-12646


# Track monthly OpenAI token usage with Google Sheets and Gmail reports

## 1. Workflow Overview

**Purpose:**  
This workflow tracks your **previous month’s OpenAI completions token usage**, writes a **daily per-model breakdown** into a **new Google Sheet created from a template**, then **exports** the finished sheet to **PDF (for email)** and **Excel (for archiving)**. It runs automatically **on the 5th of each month**.

**Target use cases:**
- Monthly cost/usage reporting for engineering, finance, or operations
- Auditing usage by model (grouped by day)
- Creating an archived monthly record in Drive plus a stakeholder email report

### 1.1 Scheduling & Report Initialization
Runs monthly and creates a new monthly report sheet by copying a template.

### 1.2 OpenAI Usage Collection
Calls the OpenAI Organization Usage API for the previous month (daily buckets, grouped by model).

### 1.3 Data Transformation & Sheet Population
Flattens OpenAI’s bucketed response into rows and appends them into the report sheet.

### 1.4 Report Export, Email Distribution, and Archival
Exports the filled sheet to PDF (email attachment) and Excel (Drive archive).

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduling & Report Initialization

**Overview:**  
Triggers on a monthly schedule (day 5) and copies a Google Sheets template to create a new report file named for the previous month.

**Nodes Involved:**
- Monthly Report Trigger
- Create Monthly Report from Template

#### Node: **Monthly Report Trigger**
- **Type / Role:** `Schedule Trigger` — workflow entrypoint; runs automatically.
- **Configuration (interpreted):**
  - Runs **monthly**, **on the 5th day** of the month.
- **Inputs / Outputs:**
  - **Output →** Create Monthly Report from Template
- **Edge cases / failure modes:**
  - Timezone effects: schedule uses the n8n instance timezone; if reporting month boundaries matter, confirm timezone alignment with OpenAI billing expectations.
  - If the instance is down on the 5th, the run may be missed unless you implement catch-up logic.

#### Node: **Create Monthly Report from Template**
- **Type / Role:** `Google Drive` (operation: Copy) — creates a new spreadsheet file from a template.
- **Configuration (interpreted):**
  - Copies the file identified by **template file ID** (`your-template-file-id` placeholder).
  - Sets the new file name to:  
    `Token_Tracking_<PreviousMonthName>_<PreviousYear>`  
    using n8n date handling:  
    - `{{$now.minus({months: 1}).toFormat('LLLL')}}` (month name)  
    - `{{$now.minus({months: 1}).toFormat('yyyy')}}` (year)
- **Key expressions / variables:**
  - Uses `$now.minus({months: 1})` to label the report as the previous month.
- **Inputs / Outputs:**
  - **Input ←** Monthly Report Trigger
  - **Output →** Fetch OpenAI Usage Data
  - Produces a file object including `id` and `name` used downstream.
- **Credentials:**
  - Google Drive OAuth2 (`WFANMain`)
- **Edge cases / failure modes:**
  - Invalid/insufficient Drive permissions to copy the template.
  - Template file ID not accessible (wrong ID, moved, deleted).
  - If the template is not a Google Sheet (or not convertible/usable as a sheet), downstream Sheets append may fail.

---

### Block 2 — OpenAI Usage Collection

**Overview:**  
Fetches OpenAI usage data for the previous month in daily buckets and grouped by model.

**Nodes Involved:**
- Fetch OpenAI Usage Data

#### Node: **Fetch OpenAI Usage Data**
- **Type / Role:** `HTTP Request` — calls OpenAI Admin/Org usage endpoint.
- **Configuration (interpreted):**
  - **GET** `https://api.openai.com/v1/organization/usage/completions`
  - Sends query parameters including:
    - `start_time`: Unix seconds for **UTC first day of previous month at 00:00:00**
    - `end_time`: Unix seconds for **UTC last day of previous month at 23:59:59**
    - `api_key_ids[]`: placeholder `your-api-key-id` (must be replaced)
    - `limit=31`
    - `bucket_width=1d`
    - `group_by[]=model`
  - Authentication: predefined credential type `openAiApi`
- **Key expressions / variables:**
  - `start_time`:
    - `{{ Math.floor(Date.UTC(new Date().getFullYear(), new Date().getMonth() - 1, 1) / 1000) }}`
  - `end_time`:
    - `{{ Math.floor(Date.UTC(new Date().getFullYear(), new Date().getMonth(), 0, 23, 59, 59) / 1000) }}`
- **Inputs / Outputs:**
  - **Input ←** Create Monthly Report from Template (sequence dependency; output not directly used here)
  - **Output →** Transform Usage to Daily Breakdown
- **Credentials:**
  - OpenAI API credential (`WFAN OpenAI`)
- **Version-specific requirements:**
  - Uses HTTP Request node `typeVersion 4.3` features for credential-based auth.
- **Edge cases / failure modes:**
  - **403/401** if the OpenAI key lacks **organization/admin usage** access.
  - Wrong `api_key_ids[]` value → empty or filtered results.
  - API response shape changes or missing fields (e.g., `start_time_iso`) will break transformation.
  - Rate limiting / transient failures (no retry configured explicitly here).
  - Month boundary logic is UTC; if you expect local-time months, adjust calculations accordingly.

---

### Block 3 — Data Transformation & Sheet Population

**Overview:**  
Transforms OpenAI’s bucketed response (`data[]` with daily buckets and `results[]`) into a flat row set and appends it into the newly created monthly Google Sheet.

**Nodes Involved:**
- Transform Usage to Daily Breakdown
- Append Data to Google Sheet

#### Node: **Transform Usage to Daily Breakdown**
- **Type / Role:** `Code` — converts nested usage buckets into row records.
- **Configuration (interpreted):**
  - Reads `rawData = $input.all()[0].json.data`
  - For each bucket:
    - Extracts `date` from `bucket.start_time_iso` (YYYY-MM-DD)
    - If bucket has `results[]`, emits one row per model:
      - `model`, `input_tokens`, `output_tokens`
    - If no results, emits a “No usage” row with zero tokens
- **Key expressions / variables:**
  - `bucket.start_time_iso.split('T')[0]`
  - Defaults:
    - `result.model || 'N/A'`
    - tokens default to `0`
- **Inputs / Outputs:**
  - **Input ←** Fetch OpenAI Usage Data
  - **Output →** Append Data to Google Sheet
- **Version-specific requirements:**
  - Code node `typeVersion 2`
- **Edge cases / failure modes:**
  - If `data` is missing or not an array → runtime error.
  - If `start_time_iso` is missing → `.split` fails.
  - Multiple API keys or unexpected grouping can increase row volume (Sheets append limits/time).

#### Node: **Append Data to Google Sheet**
- **Type / Role:** `Google Sheets` (operation: Append) — writes the transformed rows into the monthly report file.
- **Configuration (interpreted):**
  - Appends into `Sheet1` of the spreadsheet whose **Document ID** is the copied template’s `id`:
    - `documentId = {{ $('Create Monthly Report from Template').item.json.id }}`
  - Column mapping:
    - `Date` = `{{$json.date}}`
    - `Model` = `{{$json.model}}`
    - `Token Usage In` = `{{$json.input_tokens}}`
    - `Token Usage Out` = `{{$json.output_tokens}}`
- **Inputs / Outputs:**
  - **Input ←** Transform Usage to Daily Breakdown
  - **Outputs →**
    - Export Sheet as Excel
    - Export Sheet as PDF for Email
- **Credentials:**
  - Google Sheets OAuth2 (`WFAN`)
- **Version-specific requirements:**
  - Google Sheets node `typeVersion 4.7`
- **Edge cases / failure modes:**
  - Template must contain `Sheet1` (or update the node).
  - Column headers must match (Date/Model/Token Usage In/Out); otherwise append may misalign.
  - If the template uses protected ranges or permissions, append can fail.
  - Potential duplication: each monthly run appends all days again into a newly created file (fine). If re-run for the same month, it will append duplicates into that same monthly file unless you create idempotency checks.

---

### Block 4 — Report Export, Email Distribution, and Archival

**Overview:**  
Exports the filled Google Sheet as PDF and Excel. Emails the PDF to a stakeholder and archives the Excel file into a Drive folder.

**Nodes Involved:**
- Export Sheet as PDF for Email
- Email Report to Stakeholder
- Export Sheet as Excel
- Archive Report to Drive

#### Node: **Export Sheet as PDF for Email**
- **Type / Role:** `Google Drive` (operation: Download/export) — downloads the sheet converted to PDF.
- **Configuration (interpreted):**
  - `fileId = {{ $('Create Monthly Report from Template').item.json.id }}`
  - Converts Google Sheets → `application/pdf`
  - Outputs binary to property `data`
  - File name: `{{ $('Create Monthly Report from Template').item.json.name }}.pdf`
- **Inputs / Outputs:**
  - **Input ←** Append Data to Google Sheet
  - **Output →** Email Report to Stakeholder
- **Credentials:**
  - Google Drive OAuth2 (`WFANMain`)
- **Notable setting:**
  - `executeOnce: true` (node-level) — prevents multiple executions per run if multiple input items arrive.
- **Edge cases / failure modes:**
  - If multiple rows flow from Sheets append, `executeOnce` avoids multiple downloads—good. Without it, you’d create many downloads.
  - Export permissions/conversion errors if the file isn’t a Google Sheet or Drive token lacks rights.

#### Node: **Email Report to Stakeholder**
- **Type / Role:** `Gmail` (send email) — sends the monthly PDF report.
- **Configuration (interpreted):**
  - To: `user@example.com` (placeholder)
  - Subject: `Monthly OpenAI Token Usage Report`
  - Body: “Please see attached token usage data for the previous month.”
  - Attachments: configured for binary attachments, expecting the PDF binary (from previous node) in property `data`.
  - Attribution disabled.
- **Inputs / Outputs:**
  - **Input ←** Export Sheet as PDF for Email (binary)
  - **Output:** none
- **Credentials:**
  - Gmail OAuth2 (`PC Personal`)
- **Edge cases / failure modes:**
  - If attachment mapping is not correctly set to the `data` binary property, email may send without attachment.
  - Gmail API scopes insufficient, OAuth expired, or user mailbox restrictions.

#### Node: **Export Sheet as Excel**
- **Type / Role:** `Google Drive` (operation: Download/export) — downloads the sheet converted to Excel.
- **Configuration (interpreted):**
  - `fileId = {{ $('Create Monthly Report from Template').item.json.id }}`
  - Converts Google Sheets → Excel MIME:  
    `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`
  - Outputs binary to property `data`
  - File name: `{{ $('Create Monthly Report from Template').item.json.name }}.xlsx`
- **Inputs / Outputs:**
  - **Input ←** Append Data to Google Sheet
  - **Output →** Archive Report to Drive
- **Credentials:**
  - Google Drive OAuth2 (`WFANMain`)
- **Notable setting:**
  - `executeOnce: true`
- **Edge cases / failure modes:**
  - Same conversion/permission risks as PDF export.
  - Large sheets may increase export time.

#### Node: **Archive Report to Drive**
- **Type / Role:** `Google Drive` (upload/create file) — stores the Excel output into an archive folder.
- **Configuration (interpreted):**
  - Target drive: “My Drive”
  - Destination folder: `your-archive-folder-id` (placeholder)
  - Name: `{{ $('Create Monthly Report from Template').item.json.name }}.xlsx`
  - **Important:** As configured, this node does not explicitly map the incoming binary to an upload field in the visible parameters. In n8n, archiving an exported file typically requires an **Upload** operation (or “Create: File” with binary property specified). Here it is set with `name`, `driveId`, `folderId` only—so you may need to adjust this node to actually upload the binary from `Export Sheet as Excel`.
- **Inputs / Outputs:**
  - **Input ←** Export Sheet as Excel
  - **Output:** none
- **Credentials:**
  - Google Drive OAuth2 (`WFANMain`)
- **Notable setting:**
  - `executeOnce: true`
- **Edge cases / failure modes:**
  - If not configured as an upload using the binary `data`, it may create an empty/incorrect file or fail depending on node defaults.
  - Folder ID invalid or not accessible.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Monthly Report Trigger | Schedule Trigger | Monthly scheduler (day 5) | — | Create Monthly Report from Template | ## Step 1: Initialize Monthly Report<br>Triggers on the 5th of each month and copies the Google Sheets template with proper naming convention (e.g., 'Token_Tracking_December_2025').<br><br>**Update:** Set your template file ID in the Google Drive node. |
| Create Monthly Report from Template | Google Drive | Copy template sheet into a new monthly report | Monthly Report Trigger | Fetch OpenAI Usage Data | ## Step 1: Initialize Monthly Report<br>Triggers on the 5th of each month and copies the Google Sheets template with proper naming convention (e.g., 'Token_Tracking_December_2025').<br><br>**Update:** Set your template file ID in the Google Drive node. |
| Fetch OpenAI Usage Data | HTTP Request | Pull prior-month OpenAI completion usage grouped by model/day | Create Monthly Report from Template | Transform Usage to Daily Breakdown | ## Step 2: Fetch and Process Usage Data<br><br>Retrieves OpenAI usage data for the previous month via API, transforms it into a daily breakdown by model (input/output tokens), and populates the Google Sheet.<br><br>**Note:** Make sure to replace 'your-api-key-id' with your actual OpenAI API key ID in the HTTP Request node. |
| Transform Usage to Daily Breakdown | Code | Flatten bucketed usage into rows | Fetch OpenAI Usage Data | Append Data to Google Sheet | ## Step 2: Fetch and Process Usage Data<br><br>Retrieves OpenAI usage data for the previous month via API, transforms it into a daily breakdown by model (input/output tokens), and populates the Google Sheet.<br><br>**Note:** Make sure to replace 'your-api-key-id' with your actual OpenAI API key ID in the HTTP Request node. |
| Append Data to Google Sheet | Google Sheets | Append daily model/token rows to the report spreadsheet | Transform Usage to Daily Breakdown | Export Sheet as Excel; Export Sheet as PDF for Email | ## Step 2: Fetch and Process Usage Data<br><br>Retrieves OpenAI usage data for the previous month via API, transforms it into a daily breakdown by model (input/output tokens), and populates the Google Sheet.<br><br>**Note:** Make sure to replace 'your-api-key-id' with your actual OpenAI API key ID in the HTTP Request node. |
| Export Sheet as Excel | Google Drive | Export Google Sheet to XLSX (binary) | Append Data to Google Sheet | Archive Report to Drive | ## Step 3: Generate and Distribute Report<br><br>Downloads the completed sheet as both PDF and Excel formats, emails the PDF to stakeholders with usage summary, and archives the Excel file in Google Drive for long-term storage.<br><br>**Remember to update:** Email recipient address and Google Drive archive folder ID. |
| Archive Report to Drive | Google Drive | Store XLSX in archive folder | Export Sheet as Excel | — | ## Step 3: Generate and Distribute Report<br><br>Downloads the completed sheet as both PDF and Excel formats, emails the PDF to stakeholders with usage summary, and archives the Excel file in Google Drive for long-term storage.<br><br>**Remember to update:** Email recipient address and Google Drive archive folder ID. |
| Export Sheet as PDF for Email | Google Drive | Export Google Sheet to PDF (binary) | Append Data to Google Sheet | Email Report to Stakeholder | ## Step 3: Generate and Distribute Report<br><br>Downloads the completed sheet as both PDF and Excel formats, emails the PDF to stakeholders with usage summary, and archives the Excel file in Google Drive for long-term storage.<br><br>**Remember to update:** Email recipient address and Google Drive archive folder ID. |
| Email Report to Stakeholder | Gmail | Email the PDF report | Export Sheet as PDF for Email | — | ## Step 3: Generate and Distribute Report<br><br>Downloads the completed sheet as both PDF and Excel formats, emails the PDF to stakeholders with usage summary, and archives the Excel file in Google Drive for long-term storage.<br><br>**Remember to update:** Email recipient address and Google Drive archive folder ID. |
| Workflow Overview | Sticky Note | Documentation / requirements and setup guidance | — | — | ## Track OpenAI Token Usage with Automated Monthly Reports<br><br>This workflow automatically monitors your OpenAI API usage and costs on a monthly basis. Perfect for finance teams and developers who need to track AI spending across different models and projects.<br><br>### How it works<br>* Runs automatically on the 5th of each month via Schedule Trigger<br>* Creates a new Google Sheet from your template for the reporting period (template can be found [HERE](https://github.com/WorkFlowAutomationNetwork/Youtube-Resources/tree/main/N8NTemplate) )<br>* Fetches previous month's usage data from OpenAI API<br>* Transforms raw API data into daily breakdown by model<br>* Appends usage data to Google Sheets (input/output tokens per model)<br>* Your sheet calculates costs automatically using formulas<br>* Generates PDF report and Excel file<br>* Emails the PDF report to stakeholders<br>* Archives Excel file in Google Drive for recordkeeping<br><br>### Requirements<br>* OpenAI Admin API access (to access organization usage endpoints)<br>* Google Sheets template with cost calculation formulas<br>* Google Drive for storage<br>* Gmail account for email notifications<br><br>### Setup Instructions<br>1. **Create Google Sheets template** with columns: Date, Model, Token Usage In, Token Usage Out, Token Cost Input, Token Cost Output, Total Cost USD (add formulas for automatic calculations) (note a template is provided if you don't want to create your own)<br>2. **Configure credentials** in n8n for OpenAI API, Google Sheets, Google Drive, and Gmail<br>3. **Update the HTTP Request node** with your OpenAI API key ID<br>4. **Replace placeholder email** (your-email@example.com) with recipient address<br>5. **Set your Google Drive folders** for template source and archive destination<br>6. **Test the workflow** using the manual trigger before enabling schedule<br><br>### Customization Ideas<br>* Track multiple API keys by adding more `api_key_ids[]` parameters<br>* Change reporting frequency by modifying the Schedule Trigger<br>* Add Slack notifications instead of or in addition to email<br>* Include budget threshold alerts using IF nodes<br>* Add cost forecasting based on trends<br>* Track additional metrics like average tokens per request<br>* Support multiple currencies with conversion rates |
| Sticky Note | Sticky Note | Step label / guidance | — | — | ## Step 2: Fetch and Process Usage Data<br><br>Retrieves OpenAI usage data for the previous month via API, transforms it into a daily breakdown by model (input/output tokens), and populates the Google Sheet.<br><br>**Note:** Make sure to replace 'your-api-key-id' with your actual OpenAI API key ID in the HTTP Request node. |
| Sticky Note4 | Sticky Note | Step label / guidance | — | — | ## Step 3: Generate and Distribute Report<br><br>Downloads the completed sheet as both PDF and Excel formats, emails the PDF to stakeholders with usage summary, and archives the Excel file in Google Drive for long-term storage.<br><br>**Remember to update:** Email recipient address and Google Drive archive folder ID. |
| Sticky Note5 | Sticky Note | Step label / guidance | — | — | ## Step 1: Initialize Monthly Report<br><br>Triggers on the 5th of each month and copies the Google Sheets template with proper naming convention (e.g., 'Token_Tracking_December_2025').<br><br>**Update:** Set your template file ID in the Google Drive node. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named:  
   `Track OpenAI token usage to Google Sheets with monthly email reports`

2. **Add node: Schedule Trigger**
   - Type: **Schedule Trigger**
   - Set rule: **Every month**, **day of month = 5**
   - Name: `Monthly Report Trigger`

3. **Add node: Google Drive (Copy)**
   - Type: **Google Drive**
   - Operation: **Copy**
   - File to copy: set **Template File ID** (your Google Sheets template)
   - Name (new file):  
     `Token_Tracking_{{ $now.minus({months: 1}).toFormat('LLLL') }}_{{ $now.minus({months: 1}).toFormat('yyyy') }}`
   - Credentials: Google Drive OAuth2 (account with access to the template)
   - Name node: `Create Monthly Report from Template`
   - **Connect:** Monthly Report Trigger → Create Monthly Report from Template

4. **Add node: HTTP Request (OpenAI usage)**
   - Type: **HTTP Request**
   - Method: GET
   - URL: `https://api.openai.com/v1/organization/usage/completions`
   - Authentication: **OpenAI API** credential (predefined credential type)
   - Enable “Send Query Parameters”
   - Add query parameters:
     - `start_time` = `{{ Math.floor(Date.UTC(new Date().getFullYear(), new Date().getMonth() - 1, 1) / 1000) }}`
     - `end_time` = `{{ Math.floor(Date.UTC(new Date().getFullYear(), new Date().getMonth(), 0, 23, 59, 59) / 1000) }}`
     - `api_key_ids[]` = your OpenAI API key id (replace placeholder)
     - `limit` = `31`
     - `bucket_width` = `1d`
     - `group_by[]` = `model`
   - Name node: `Fetch OpenAI Usage Data`
   - **Connect:** Create Monthly Report from Template → Fetch OpenAI Usage Data

5. **Add node: Code (transform buckets to rows)**
   - Type: **Code**
   - Paste the transformation logic that:
     - Reads `json.data`
     - Produces `{date, model, input_tokens, output_tokens}` per day/model
     - Emits “No usage” rows for empty days
   - Name node: `Transform Usage to Daily Breakdown`
   - **Connect:** Fetch OpenAI Usage Data → Transform Usage to Daily Breakdown

6. **Add node: Google Sheets (Append)**
   - Type: **Google Sheets**
   - Operation: **Append**
   - Document ID: expression pointing to the copied file:  
     `{{ $('Create Monthly Report from Template').item.json.id }}`
   - Sheet name: `Sheet1` (or your template’s sheet tab name)
   - Map columns:
     - Date → `{{$json.date}}`
     - Model → `{{$json.model}}`
     - Token Usage In → `{{$json.input_tokens}}`
     - Token Usage Out → `{{$json.output_tokens}}`
   - Credentials: Google Sheets OAuth2
   - Name node: `Append Data to Google Sheet`
   - **Connect:** Transform Usage to Daily Breakdown → Append Data to Google Sheet

7. **Add node: Google Drive (Download as PDF)**
   - Type: **Google Drive**
   - Operation: **Download**
   - File ID: `{{ $('Create Monthly Report from Template').item.json.id }}`
   - Enable Google file conversion to: **PDF**
   - Output binary property name: `data`
   - File name: `{{ $('Create Monthly Report from Template').item.json.name }}.pdf`
   - (Optional but recommended) enable **Execute Once**
   - Name: `Export Sheet as PDF for Email`
   - **Connect:** Append Data to Google Sheet → Export Sheet as PDF for Email

8. **Add node: Gmail (Send)**
   - Type: **Gmail**
   - Operation: **Send**
   - To: replace with stakeholder email
   - Subject: `Monthly OpenAI Token Usage Report`
   - Message: your text
   - Attachments: attach binary property **`data`** from the previous node
   - Credentials: Gmail OAuth2
   - Name: `Email Report to Stakeholder`
   - **Connect:** Export Sheet as PDF for Email → Email Report to Stakeholder

9. **Add node: Google Drive (Download as Excel)**
   - Type: **Google Drive**
   - Operation: **Download**
   - File ID: `{{ $('Create Monthly Report from Template').item.json.id }}`
   - Convert to: **XLSX**
   - Binary property: `data`
   - File name: `{{ $('Create Monthly Report from Template').item.json.name }}.xlsx`
   - (Optional but recommended) enable **Execute Once**
   - Name: `Export Sheet as Excel`
   - **Connect:** Append Data to Google Sheet → Export Sheet as Excel

10. **Add node: Google Drive (Archive upload)**
   - Type: **Google Drive**
   - Configure it to **upload the binary** created by “Export Sheet as Excel” into a folder:
     - Destination folder: your archive folder ID
     - File name: `{{ $('Create Monthly Report from Template').item.json.name }}.xlsx`
     - Binary property: `data`
   - Name: `Archive Report to Drive`
   - **Connect:** Export Sheet as Excel → Archive Report to Drive  
   - **Note:** Ensure the operation is an upload/create-file operation that consumes the `data` binary; otherwise the archive step will not store the exported Excel.

11. **Create the Google Sheets template**
   - Include headers used by the append step:
     - Date, Model, Token Usage In, Token Usage Out
   - Add computed columns/formulas (optional but referenced by the workflow notes):
     - Token Cost Input, Token Cost Output, Total Cost USD

12. **Set credentials**
   - OpenAI API credential with access to **organization usage** endpoints
   - Google Drive OAuth2 (template copy + exports + archive)
   - Google Sheets OAuth2 (append rows)
   - Gmail OAuth2 (send email)

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Template can be found here: [https://github.com/WorkFlowAutomationNetwork/Youtube-Resources/tree/main/N8NTemplate](https://github.com/WorkFlowAutomationNetwork/Youtube-Resources/tree/main/N8NTemplate) | Referenced in “Workflow Overview” sticky note |
| Workflow runs on the 5th of each month; adjust if you need earlier/later reporting. | Schedule Trigger behavior |
| Replace placeholders: `your-api-key-id`, `your-template-file-id`, `your-archive-folder-id`, and `user@example.com`. | Required for a working deployment |
| Ensure the archive step is configured to upload the XLSX binary (`data`) to Drive; the provided configuration may require adjustment. | Prevents missing/empty archived file |