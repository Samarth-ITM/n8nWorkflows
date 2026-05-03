Send weekly AI summaries of Google Sheets data via Gmail using OpenAI

https://n8nworkflows.xyz/workflows/send-weekly-ai-summaries-of-google-sheets-data-via-gmail-using-openai-13404


# Send weekly AI summaries of Google Sheets data via Gmail using OpenAI

## 1. Workflow Overview

**Purpose:**  
This workflow automatically generates a **weekly business-style summary** from a Google Sheets dataset using **OpenAI (Chat model)**, sends the summary by **Gmail**, and then **logs** the send event into a separate Google Sheet (“Report Log”).

**Primary use cases:**
- Weekly KPI/status email from a spreadsheet (sales, support tickets, marketing metrics, ops metrics)
- Automated stakeholder reporting without manual copy/paste
- Maintaining an audit trail of sent reports

### 1.1 Schedule & Data Extraction
Runs on a weekly schedule (Monday 9:00), reads all rows from a specified Google Sheet, and aggregates them into one payload.

### 1.2 AI Summarization
Sends the aggregated rows to an AI Agent powered by an OpenAI chat model, with instructions to produce a concise, bullet-point weekly report.

### 1.3 Formatting, Email Delivery & Logging
Formats the AI output into email-ready fields (subject/date/body), sends the email via Gmail, then appends a log entry to a “Report Log” sheet.

---

## 2. Block-by-Block Analysis

### Block 1 — Schedule & Read Data
**Overview:**  
Triggers weekly, pulls spreadsheet rows from Google Sheets, then consolidates all row items into a single JSON field for downstream AI processing.

**Nodes involved:**
- Schedule Trigger
- Read Google Sheets Data
- Aggregate Rows

#### Node: Schedule Trigger
- **Type / role:** `n8n-nodes-base.scheduleTrigger` — time-based entry point.
- **Configuration (interpreted):**
  - Runs **every week** on **Monday** at **09:00** (server/workflow timezone applies).
- **Inputs/outputs:**
  - **Output →** Read Google Sheets Data (main)
- **Version notes:** typeVersion **1.2**
- **Edge cases / failures:**
  - Timezone mismatch (n8n instance timezone vs. expected business timezone).
  - If workflow is inactive (`"active": false`), it will never run.

#### Node: Read Google Sheets Data
- **Type / role:** `n8n-nodes-base.googleSheets` — reads rows from a sheet.
- **Configuration (interpreted):**
  - Uses a **Document ID** placeholder: `<Google Sheet ID with your data>`
  - Uses a **Sheet name** placeholder: `<Sheet name e.g. Sales Data>`
  - Operation is implied as “read/get rows” (default for this node when not explicitly set in parameters).
- **Key variables/expressions:** none (placeholders must be replaced).
- **Inputs/outputs:**
  - **Input ←** Schedule Trigger
  - **Output →** Aggregate Rows
- **Version notes:** typeVersion **4.7**
- **Edge cases / failures:**
  - Google auth/permissions (403) if the connected account lacks access.
  - Wrong sheet name or document ID → “not found”.
  - Large sheets may cause performance issues or pagination/limits depending on node settings.

#### Node: Aggregate Rows
- **Type / role:** `n8n-nodes-base.aggregate` — combines multiple incoming items into one item.
- **Configuration (interpreted):**
  - Mode: **Aggregate All Item Data**
  - Stores the combined dataset in field: **`all_rows`**
- **Inputs/outputs:**
  - **Input ←** Read Google Sheets Data
  - **Output →** Generate Summary with AI
- **Version notes:** typeVersion **1**
- **Edge cases / failures:**
  - Empty sheet (0 rows) → downstream AI gets `[]` which may produce an unhelpful summary unless handled.
  - Very large combined payload can exceed AI model input limits.

---

### Block 2 — AI Summarization (OpenAI via LangChain Agent)
**Overview:**  
Takes the aggregated rows and prompts an AI agent to produce a short, professional weekly report (bullet points, <300 words, no greeting/sign-off).

**Nodes involved:**
- Generate Summary with AI
- OpenAI Chat Model

#### Node: OpenAI Chat Model
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — provides the language model to the agent.
- **Configuration (interpreted):**
  - Model: **`gpt-4.1-mini`**
  - No special options or built-in tools configured.
- **Inputs/outputs:**
  - Connected via **AI language model port** to Generate Summary with AI.
- **Version notes:** typeVersion **1.3**
- **Edge cases / failures:**
  - Missing/invalid OpenAI credentials.
  - Model unavailable in the account/region.
  - Rate limits or transient 5xx errors.
  - Token/context overflow if `all_rows` is too large.

#### Node: Generate Summary with AI
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — agent that generates the summary text.
- **Configuration (interpreted):**
  - Prompt type: **Define** (custom prompt)
  - **User prompt** (key expression):
    - Embeds aggregated data: `{{ JSON.stringify($json.all_rows) }}`
    - Asks for “brief weekly summary report highlighting key trends, totals, and notable changes… formatted as clean email body with bullet points.”
  - **System message**:
    - “Business analyst assistant”, concise/professional, bullet points, **under 300 words**, **no greetings/sign-offs**.
- **Inputs/outputs:**
  - **Input ←** Aggregate Rows
  - **AI Model input ←** OpenAI Chat Model (ai_languageModel connection)
  - **Output →** Format Report
- **Version notes:** typeVersion **3**
- **Edge cases / failures:**
  - If `all_rows` contains complex/large objects, `JSON.stringify` can bloat the prompt.
  - If sheet columns are inconsistent, the model may misinterpret data types.
  - Output format variability: the next node expects `{{$json.output}}` to exist.

---

### Block 3 — Format, Email, Log
**Overview:**  
Normalizes the AI output into predictable fields, sends the email via Gmail, then appends a row to a logging spreadsheet.

**Nodes involved:**
- Format Report
- Send Report Email
- Log Report Sent

#### Node: Format Report
- **Type / role:** `n8n-nodes-base.set` — creates/overwrites fields for downstream usage.
- **Configuration (interpreted):**
  - Creates fields:
    - `report_body` = `{{ $json.output }}`
    - `report_date` = `{{ $now.format('yyyy-MM-dd') }}`
    - `subject_line` = `Weekly Report - {{ $now.format('yyyy-MM-dd') }}`
- **Inputs/outputs:**
  - **Input ←** Generate Summary with AI
  - **Output →** Send Report Email
- **Version notes:** typeVersion **3.4**
- **Edge cases / failures:**
  - If the Agent node output key is not `output` (varies by configuration/version), `report_body` becomes empty/undefined.
  - Date formatting uses `$now`; timezone can affect the date stamp near midnight boundaries.

#### Node: Send Report Email
- **Type / role:** `n8n-nodes-base.gmail` — sends an email through Gmail OAuth2.
- **Configuration (interpreted):**
  - To: placeholder `<Recipient email address>`
  - Subject: `{{ $json.subject_line }}`
  - Message body: `{{ $json.report_body }}`
  - Credentials: **Gmail OAuth2** (named “Gmail account” in the workflow)
- **Inputs/outputs:**
  - **Input ←** Format Report
  - **Output →** Log Report Sent
- **Version notes:** typeVersion **2.1**
- **Edge cases / failures:**
  - OAuth token expired/revoked → authentication failure.
  - Gmail sending limits/quota issues.
  - If `report_body` is empty, email will be blank unless validated.

#### Node: Log Report Sent
- **Type / role:** `n8n-nodes-base.googleSheets` — appends a row to a log sheet.
- **Configuration (interpreted):**
  - Operation: **Append**
  - Document ID placeholder: `<Google Sheet ID for report log>`
  - Sheet name: **Report Log**
  - Columns mapped (define below):
    - `date` = `{{ $json.report_date }}`
    - `sent_at` = `{{ $now.toISO() }}`
    - `subject` = `{{ $json.subject_line }}`
- **Inputs/outputs:**
  - **Input ←** Send Report Email
  - No downstream nodes.
- **Version notes:** typeVersion **4.7**
- **Edge cases / failures:**
  - “Report Log” sheet missing or columns don’t match expected headers.
  - Permission/auth issues on the logging spreadsheet.
  - Partial failure risk: email could send successfully but logging fails (no compensating handling in this workflow).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | n8n-nodes-base.scheduleTrigger | Weekly trigger (Mon 09:00) | — | Read Google Sheets Data | ## Schedule & Read<br>Runs every Monday at 9am, reads all rows from the data spreadsheet, and aggregates them. |
| Read Google Sheets Data | n8n-nodes-base.googleSheets | Read source spreadsheet rows | Schedule Trigger | Aggregate Rows | ## Schedule & Read<br>Runs every Monday at 9am, reads all rows from the data spreadsheet, and aggregates them. |
| Aggregate Rows | n8n-nodes-base.aggregate | Combine all rows into one payload (`all_rows`) | Read Google Sheets Data | Generate Summary with AI | ## Schedule & Read<br>Runs every Monday at 9am, reads all rows from the data spreadsheet, and aggregates them. |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provides OpenAI chat model to agent | — (AI port connection only) | Generate Summary with AI (ai_languageModel) | ## Summarize, Send & Log<br>AI generates a summary report, emails it via Gmail, and logs the send to a separate spreadsheet. |
| Generate Summary with AI | @n8n/n8n-nodes-langchain.agent | Generate weekly summary from aggregated rows | Aggregate Rows + OpenAI Chat Model (AI port) | Format Report | ## Summarize, Send & Log<br>AI generates a summary report, emails it via Gmail, and logs the send to a separate spreadsheet. |
| Format Report | n8n-nodes-base.set | Build report fields (body/date/subject) | Generate Summary with AI | Send Report Email | ## Summarize, Send & Log<br>AI generates a summary report, emails it via Gmail, and logs the send to a separate spreadsheet. |
| Send Report Email | n8n-nodes-base.gmail | Send report via Gmail | Format Report | Log Report Sent | ## Summarize, Send & Log<br>AI generates a summary report, emails it via Gmail, and logs the send to a separate spreadsheet. |
| Log Report Sent | n8n-nodes-base.googleSheets | Append log row (date/sent_at/subject) | Send Report Email | — | ## Summarize, Send & Log<br>AI generates a summary report, emails it via Gmail, and logs the send to a separate spreadsheet. |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation note (not executed) | — | — | ## How it works<br>This workflow runs automatically every Monday at 9am. It reads data from a Google Sheets spreadsheet, aggregates all the rows into a single payload, and sends it to OpenAI to generate a concise weekly summary report with key trends and highlights. The AI-generated report is formatted with a date-stamped subject line, emailed to your chosen recipient via Gmail, and logged to a separate spreadsheet so you have a record of every report sent.<br><br>## Setup steps<br>1. Adjust the Schedule Trigger if you want a different day or time<br>2. Connect your Google Sheets account and set the spreadsheet ID and sheet name containing your data<br>3. Connect your OpenAI credentials to the OpenAI Chat Model node<br>4. Set the recipient email address in the Send Report Email node<br>5. Connect your Gmail account for sending the report<br>6. Create a "Report Log" sheet with columns: date, subject, sent_at<br>7. Set the Report Log spreadsheet ID in the Log Report Sent node<br>8. Activate the workflow |
| Sticky Note1 | n8n-nodes-base.stickyNote | Documentation note (not executed) | — | — |  |
| Sticky Note2 | n8n-nodes-base.stickyNote | Documentation note (not executed) | — | — |  |

> Note: Sticky notes are non-executing annotation nodes; they appear as rows above for completeness because they are nodes in the workflow JSON.

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow** in n8n  
   - Name it similar to: *Send weekly AI-summarized Google Sheets reports via Gmail using OpenAI*

2) **Add “Schedule Trigger” (Schedule Trigger node)**
   - Set interval to **Weeks**
   - Choose **Monday**
   - Set time to **09:00**
   - (Optional) Confirm n8n instance timezone aligns with your reporting timezone

3) **Add “Read Google Sheets Data” (Google Sheets node)**
   - Connect **Schedule Trigger → Read Google Sheets Data**
   - Authenticate Google Sheets (Google OAuth2 in n8n)
   - Set:
     - **Document**: your source Google Sheet ID
     - **Sheet name**: the tab containing your data (e.g., “Sales Data”)
   - Configure the node to read rows (default “read/get” behavior; keep defaults unless you need filtering)

4) **Add “Aggregate Rows” (Aggregate node)**
   - Connect **Read Google Sheets Data → Aggregate Rows**
   - Set **Aggregate** mode to **Aggregate All Item Data**
   - Set **Destination Field Name** to: `all_rows`

5) **Add “OpenAI Chat Model” (OpenAI Chat Model node)**
   - Choose model: **gpt-4.1-mini**
   - Add/attach OpenAI credentials (API key / OpenAI credential in n8n)
   - Leave options default unless you need temperature/max tokens

6) **Add “Generate Summary with AI” (AI Agent node)**
   - Connect **Aggregate Rows → Generate Summary with AI**
   - Connect **OpenAI Chat Model → Generate Summary with AI** using the **AI language model** connection type (not main)
   - Set Prompt type to **Define**
   - System message:
     - “You are a business analyst assistant. Write concise, professional weekly summary reports. Use bullet points and keep it under 300 words. Do not include greetings or sign-offs, just the report content.”
   - User prompt text (include expression for the data):
     - Provide instruction + embed: `{{ JSON.stringify($json.all_rows) }}`

7) **Add “Format Report” (Set node)**
   - Connect **Generate Summary with AI → Format Report**
   - Add fields:
     - `report_body` = expression referencing the agent output (as in the workflow): `{{ $json.output }}`
     - `report_date` = `{{ $now.format('yyyy-MM-dd') }}`
     - `subject_line` = `Weekly Report - {{ $now.format('yyyy-MM-dd') }}`
   - If your Agent node returns a different field name than `output`, adjust `report_body` accordingly.

8) **Add “Send Report Email” (Gmail node)**
   - Connect **Format Report → Send Report Email**
   - Configure Gmail OAuth2 credentials in n8n (connect the Gmail account authorized to send)
   - Set:
     - **To**: recipient email address
     - **Subject**: `{{ $json.subject_line }}`
     - **Message**: `{{ $json.report_body }}`

9) **Add “Log Report Sent” (Google Sheets node)**
   - Connect **Send Report Email → Log Report Sent**
   - Authenticate Google Sheets (can be same or different Google credential)
   - Set **Operation**: **Append**
   - Set **Document**: the Google Sheet ID used for logging
   - Set **Sheet name**: `Report Log`
   - Ensure the “Report Log” tab has headers/columns: **date**, **subject**, **sent_at**
   - Map columns:
     - `date` = `{{ $json.report_date }}`
     - `subject` = `{{ $json.subject_line }}`
     - `sent_at` = `{{ $now.toISO() }}`

10) **Test and activate**
   - Manually execute once (or temporarily change schedule to “every minute”) to validate:
     - Sheets read returns expected rows
     - Aggregate output looks correct
     - AI returns text in the expected field
     - Gmail sends successfully
     - Log row appends properly
   - Activate the workflow.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow runs automatically every Monday at 9am. It reads data from a Google Sheets spreadsheet, aggregates all the rows into a single payload, and sends it to OpenAI to generate a concise weekly summary report with key trends and highlights. The AI-generated report is formatted with a date-stamped subject line, emailed to your chosen recipient via Gmail, and logged to a separate spreadsheet so you have a record of every report sent. | From sticky note “How it works” |
| Setup steps: 1) Adjust schedule 2) Connect Google Sheets + set spreadsheet ID & sheet name 3) Connect OpenAI credentials 4) Set recipient email 5) Connect Gmail 6) Create “Report Log” with columns: date, subject, sent_at 7) Set Report Log spreadsheet ID 8) Activate workflow | From sticky note “Setup steps” |
| “Schedule & Read: Runs every Monday at 9am, reads all rows from the data spreadsheet, and aggregates them.” | From sticky note “Schedule & Read” |
| “Summarize, Send & Log: AI generates a summary report, emails it via Gmail, and logs the send to a separate spreadsheet.” | From sticky note “Summarize, Send & Log” |