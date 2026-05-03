Generate AI incident reports with GPT-4, Slack alerts, Gmail and PDF export

https://n8nworkflows.xyz/workflows/generate-ai-incident-reports-with-gpt-4--slack-alerts--gmail-and-pdf-export-11733


# Generate AI incident reports with GPT-4, Slack alerts, Gmail and PDF export

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow name:** AI Incident Report Generator  
**Purpose:** Accept incident data via an HTTP webhook, normalize it, have GPT‑4 generate structured analysis (root cause, impact, actions), optionally alert Slack for high severity, then generate a branded PDF report, upload it to Google Drive, and email it to the operations team.

**Typical use cases**
- Automated incident intake from monitoring/ITSM tools (custom integrations, alert managers, internal forms).
- Rapid triage for critical incidents (Slack ping for “high” severity).
- Consistent post-incident documentation (PDF + Drive archival + email distribution).

### Logical blocks
**1.1 Input Reception & Data Normalization**  
Webhook intake → normalize/standardize incident fields.

**1.2 AI Analysis (Root Cause / Impact / Recommendations)**  
Send normalized incident to GPT‑4 → parse JSON response → merge with original data.

**1.3 Severity Routing & Slack Alerting**  
If severity = `high` → send Slack alert, then continue; else stop with an error.

**1.4 Report Generation & Distribution**  
Render HTML report → convert to PDF → upload to Google Drive + send Gmail email with attachment.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception & Data Normalization
**Overview:** Receives incident payload via POST webhook and reshapes it into a consistent schema (including severity normalization and timestamps) for downstream AI and reporting.

**Nodes involved:** `Webhook`, `Normalize Data`

#### Node: Webhook
- **Type / role:** `n8n-nodes-base.webhook` — Entry point; accepts inbound HTTP requests.
- **Key configuration:**
  - **HTTP Method:** POST
  - **Path:** `incident-report` (final URL depends on n8n base URL and webhook configuration)
- **Input/Output:**
  - **Output:** One item containing webhook data, typically in `$json.body` for JSON requests.
  - **Next node:** `Normalize Data`
- **Edge cases / failures:**
  - Missing/invalid JSON body (depends on caller content-type).
  - Large payloads (n8n instance limits).
  - If callers send fields not matching expected keys, later expressions may fail.

#### Node: Normalize Data
- **Type / role:** `n8n-nodes-base.set` — Creates a flattened, normalized incident object.
- **Key configuration choices:**
  - Uses **Set node “assignments”** to map from `$json.body.*` into top-level keys.
  - `ignoreConversionErrors: true` reduces hard failures when types mismatch.
- **Important expressions / variables:**
  - `incident_id = {{$json.body.incident_id}}`
  - `severity = {{$json.body.severity.toLowerCase()}}` (forces lower-case)
  - `affected_systems` is declared as **array** but value is `{{$json.body.affected_systems.join(', ')}}` which actually returns a **string**.
  - `report_generated_at = {{$now.toFormat('yyyy-MM-dd HH:mm:ss')}}`
- **Connections:**
  - **Input:** `Webhook`
  - **Output:** `Root Cause + Impact Analysis`
- **Edge cases / failures:**
  - If `severity` is missing or not a string → `.toLowerCase()` will throw.
  - If `affected_systems` is not an array → `.join()` will throw.
  - Declared type “array” but providing a string can cause confusion later; downstream nodes treat it like a string (works for display), but it is inconsistent.
- **Version notes:** Set node v3.4 supports the assignments UI and conversion options used here.

---

### 2.2 AI Analysis (Root Cause / Impact / Recommendations)
**Overview:** Sends incident context to GPT‑4 with instructions to return strict JSON. Then parses and merges AI output with the normalized incident data.

**Nodes involved:** `Root Cause + Impact Analysis`, `Parse AI Response`

#### Node: Root Cause + Impact Analysis
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi` — Calls OpenAI chat model via LangChain integration.
- **Key configuration choices:**
  - **Model:** GPT‑4 (`modelId: gpt-4`)
  - **Temperature:** 0.7 (more creative; may increase risk of invalid JSON)
  - **Prompting strategy:** System message forces a JSON object with exact keys:
    - `root_cause` (string)
    - `impact_assessment` (string)
    - `immediate_actions` (array of strings)
    - `long_term_actions` (array of strings)
  - User content injects incident fields via expressions.
- **Connections:**
  - **Input:** `Normalize Data`
  - **Output:** `Parse AI Response`
- **Edge cases / failures:**
  - OpenAI auth/quota/model availability errors.
  - Model returning non-JSON or extra commentary (common with higher temperature).
  - If `affected_systems` is stored as a string, the prompt will still work, but semantics differ.
- **Version notes:** Node v2.1; output structure is assumed later in the Code node (see below), so changes in node output schema can break parsing.

#### Node: Parse AI Response
- **Type / role:** `n8n-nodes-base.code` — Extracts text from the OpenAI node response, parses JSON, merges with original incident object.
- **Key implementation details (interpreted):**
  - Reads the first input item: `const openAIOutput = $input.first().json;`
  - Extracts AI text at: `openAIOutput.output[0].content[0].text`
  - `JSON.parse(aiResponseText)` to convert the model output string into an object.
  - Fetches normalized data directly from another node using: `$('Normalize Data').first().json;`
  - Returns merged object (original + AI keys).
- **Connections:**
  - **Input:** `Root Cause + Impact Analysis`
  - **Output:** `Check Severity`
- **Edge cases / failures:**
  - **Parsing failure:** if GPT output is not valid JSON → `JSON.parse` throws and node fails.
  - **Schema mismatch:** if OpenAI node output structure differs (e.g., path to text changes) → `undefined` errors.
  - **Cross-node reference risk:** `$('Normalize Data')...` assumes that node executed and has an item; in split/merge scenarios this can be unsafe (here it’s linear, so fine).
- **Version notes:** Code node v2 uses the modern runtime and `$input` helpers.

---

### 2.3 Severity Routing & Slack Alerting
**Overview:** If severity is exactly `high`, sends an immediate Slack message with key details and first two immediate actions; then restores the incident JSON (because Slack returns its own response). If not high, the workflow is halted with a custom error.

**Nodes involved:** `Check Severity`, `Alerts`, `Data Recovery`, `Stop and Error`

#### Node: Check Severity
- **Type / role:** `n8n-nodes-base.if` — Branching based on severity.
- **Condition logic:**
  - Checks `{{$json.severity}}` **equals** `"high"` (case sensitive match, but severity is already lowercased upstream).
- **Connections:**
  - **Input:** `Parse AI Response`
  - **True output:** `Alerts`
  - **False output:** `Stop and Error`
- **Edge cases / failures:**
  - If severity is missing/empty → condition false → stops the workflow (current design).
  - Only “high” proceeds; “medium/low” are treated as errors, even though the sticky note suggests “others proceed to reporting” (the actual implementation stops).

#### Node: Alerts
- **Type / role:** `n8n-nodes-base.slack` — Posts Slack message to a channel.
- **Key configuration choices:**
  - Operation: send message (via Slack API credentials).
  - Channel selected by `channelId` (placeholder: `YOUR_CHANNEL_ID`)
  - Message text includes incident data and:
    - `- {{ $json.immediate_actions[0] }}`
    - `- {{ $json.immediate_actions[1] }}`
- **Connections:**
  - **Input:** `Check Severity` (true branch)
  - **Output:** `Data Recovery`
- **Edge cases / failures:**
  - Slack auth/token scope issues (`chat:write` etc.).
  - If `immediate_actions` has fewer than 2 items → `[1]` renders `undefined` in message.
  - Message length limits for Slack (very large root cause text could be truncated/rejected).
- **Version notes:** Slack node v2.4.

#### Node: Data Recovery
- **Type / role:** `n8n-nodes-base.code` — Restores original incident JSON after Slack node changes the output.
- **Key implementation detail:**
  - Ignores Slack response and re-reads the incident data from `Check Severity`:
    - `const incidentData = $('Check Severity').first().json;`
- **Connections:**
  - **Input:** `Alerts`
  - **Output:** `Convert to PDF`
- **Edge cases / failures:**
  - If `Check Severity` item indexing differs in multi-item executions, `.first()` may not match the Slack-posted item.

#### Node: Stop and Error
- **Type / role:** `n8n-nodes-base.stopAndError` — Hard stop for non-high incidents.
- **Configuration:**
  - Error message: `Severity is not HIGH!`
- **Connections:**
  - **Input:** `Check Severity` (false branch)
  - **Output:** none
- **Edge cases / failures:**
  - This design prevents report generation for non-high incidents. If you intended to generate reports for all severities, remove/rewire this.

---

### 2.4 Report Generation & Distribution
**Overview:** Builds a styled HTML incident report, converts it to a PDF file, uploads it to Google Drive, and emails it to ops.

**Nodes involved:** `Convert to PDF`, `Upload file`, `Send Email`

#### Node: Convert to PDF
- **Type / role:** `n8n-nodes-htmlcsstopdf.htmlcsstopdf` — Third-party HTML-to-PDF conversion.
- **Key configuration choices:**
  - **HTML template** embedded with templating expressions such as:
    - `{{ $json.severity }}`, `{{ $json.title }}`, etc.
    - Lists rendered via JS-in-template:
      - `{{ $json.immediate_actions.map(action => '<li>' + action + '</li>').join('') }}`
  - **Output format:** file (binary)
  - **Output filename:** `data` (binary property name typically becomes `data`)
  - Uses Google Fonts import and extensive CSS.
- **Connections:**
  - **Input:** `Data Recovery`
  - **Outputs:** `Upload file` and `Send Email` (parallel)
- **Edge cases / failures:**
  - HTML-to-PDF API key invalid / quota / service downtime.
  - Remote font loading may fail in some PDF renderers; consider embedding fonts or using system fonts.
  - If `immediate_actions` / `long_term_actions` are not arrays → `.map()` will throw at template rendering time.
  - Severity class expects `severity-high|medium|low`; it uses `severity-{{ $json.severity }}` so severity must be one of those words.
- **Version notes:** Custom/community node v1; ensure it is installed on your n8n instance.

#### Node: Upload file
- **Type / role:** `n8n-nodes-base.googleDrive` — Uploads the generated PDF to Drive.
- **Key configuration choices:**
  - **File name expression:**
    - `Incident_{{ $('Data Recovery').item.json.incident_id }}_{{ $('Data Recovery').item.json.severity }}_{{ $now.format('YYYY-MM-DD_HHmmss') }}.pdf`
  - **Drive:** My Drive
  - **Folder:** `root` (placeholder cached name `YOUR_FOLDER_NAME`)
  - **Binary input:** implied from previous node’s file output (must be set in node UI if required; JSON suggests standard upload usage).
- **Connections:**
  - **Input:** `Convert to PDF`
  - **Output:** none
- **Edge cases / failures:**
  - OAuth scopes / token expiry / missing permissions.
  - Folder ID incorrect.
  - If the PDF binary property name is not what the node expects, upload fails (verify binary field selection in UI).
- **Version notes:** Google Drive node v3.

#### Node: Send Email
- **Type / role:** `n8n-nodes-base.gmail` — Sends an HTML email to ops with the PDF attached.
- **Key configuration choices:**
  - **To:** `ops-team@yourcompany.com` (must be updated)
  - **Subject expression:** `[SEVERITY] Incident Report: Title - ID`
  - **HTML body:** branded template referencing `$('Data Recovery').item.json.*`
  - **Attachments:** configured via `attachmentsBinary` (expects the PDF binary from `Convert to PDF`)
- **Connections:**
  - **Input:** `Convert to PDF`
  - **Output:** none
- **Edge cases / failures:**
  - Gmail OAuth token/scopes; Gmail API sending limits.
  - Attachment mapping: ensure the binary property (likely `data`) is selected in the node UI; otherwise email sends without attachment or fails.
  - HTML references to arrays assume `immediate_actions`/`long_term_actions` exist and have `.length` and `.slice()`.
- **Version notes:** Gmail node v2.2.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook | n8n-nodes-base.webhook | Inbound incident intake (HTTP POST) | — | Normalize Data | ## Input & Data Normalization<br>Receives incident details via webhook and structures data for AI analysis. |
| Normalize Data | n8n-nodes-base.set | Map/normalize webhook payload fields | Webhook | Root Cause + Impact Analysis | ## Input & Data Normalization<br>Receives incident details via webhook and structures data for AI analysis. |
| Root Cause + Impact Analysis | @n8n/n8n-nodes-langchain.openAi | GPT‑4 analysis producing structured JSON | Normalize Data | Parse AI Response | ## AI Analysis & Risk Assessment<br>Uses OpenAI to analyze root cause, assess impact, and generate actionable recommendations. |
| Parse AI Response | n8n-nodes-base.code | Parse GPT output JSON and merge with incident | Root Cause + Impact Analysis | Check Severity | ## AI Analysis & Risk Assessment<br>Uses OpenAI to analyze root cause, assess impact, and generate actionable recommendations. |
| Check Severity | n8n-nodes-base.if | Route only “high” severity forward | Parse AI Response | Alerts (true), Stop and Error (false) | ## Severity Routing & Alerts<br>Routes high-severity incidents to immediate Slack notifications, others proceed to reporting. |
| Alerts | n8n-nodes-base.slack | Post high-severity alert to Slack channel | Check Severity (true) | Data Recovery | ## Severity Routing & Alerts<br>Routes high-severity incidents to immediate Slack notifications, others proceed to reporting. |
| Data Recovery | n8n-nodes-base.code | Restore incident JSON after Slack node | Alerts | Convert to PDF | ## Severity Routing & Alerts<br>Routes high-severity incidents to immediate Slack notifications, others proceed to reporting. |
| Stop and Error | n8n-nodes-base.stopAndError | Stop workflow if not high severity | Check Severity (false) | — | ## Severity Routing & Alerts<br>Routes high-severity incidents to immediate Slack notifications, others proceed to reporting. |
| Convert to PDF | n8n-nodes-htmlcsstopdf.htmlcsstopdf | Render HTML report → PDF binary | Data Recovery | Upload file, Send Email | ## Report Generation & Distribution<br>Creates formatted PDF report and delivers via email and Google Drive storage. |
| Upload file | n8n-nodes-base.googleDrive | Upload PDF to Drive for archiving | Convert to PDF | — | ## Report Generation & Distribution<br>Creates formatted PDF report and delivers via email and Google Drive storage. |
| Send Email | n8n-nodes-base.gmail | Email ops team with HTML summary + PDF attachment | Convert to PDF | — | ## Report Generation & Distribution<br>Creates formatted PDF report and delivers via email and Google Drive storage. |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation / usage notes | — | — | ### How It Works<br>This workflow automates incident report generation and crisis response management for operations teams. When an incident is reported via webhook, the system normalizes and validates the incoming data including incident ID, title, description, severity level, affected systems, and initial impact assessment. The structured data is sent to OpenAI GPT-4 for intelligent analysis, which performs root cause analysis, assesses business and technical impact, and generates actionable recommendations split into immediate and long-term actions. The AI response is parsed and merged with the original incident details to create a comprehensive report dataset. For high-severity incidents, the workflow automatically triggers a Slack alert to notify the operations team immediately with critical details and preliminary recommendations. All incidents proceed to document generation where a beautifully formatted HTML report is created with severity badges, color-coded risk indicators, structured sections, and actionable items. This HTML is converted to a professional PDF that's simultaneously emailed to the operations team with an executive summary and uploaded to Google Drive for permanent record-keeping and compliance documentation.<br><br>### Setup Steps<br>1. Connect required credentials: OpenAI API, HTML-to-PDF service, Gmail, Google Drive, and Slack.<br>2. Configure the Slack channel where high-severity incident alerts should be posted.<br>3. Update the email recipient address in the "Send Email" node (currently ops-team@yourcompany.com).<br>4. Customize the Google Drive folder where incident reports should be stored.<br>5. Adjust the severity threshold in the "Check Severity" node if you want alerts for medium-severity incidents too.<br>6. Copy the webhook URL and integrate it into your monitoring system or incident reporting tool.<br>7. Test with sample incidents of varying severity levels to verify routing and notifications.<br><br>### Customization<br>- Modify the AI system prompt to include industry-specific analysis frameworks or compliance requirements.<br>- Adjust the severity color scheme and badges in the HTML template to match your brand guidelines.<br>- Add additional notification channels (PagerDuty, Microsoft Teams) for critical incidents.<br>- Customize the email template styling and branding to align with your organization's visual identity.<br>- Expand the AI analysis to include cost impact calculations or SLA breach assessments. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Section label | — | — | ## Input & Data Normalization<br>Receives incident details via webhook and structures data for AI analysis. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Section label | — | — | ## AI Analysis & Risk Assessment<br>Uses OpenAI to analyze root cause, assess impact, and generate actionable recommendations. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Section label | — | — | ## Severity Routing & Alerts<br>Routes high-severity incidents to immediate Slack notifications, others proceed to reporting. |
| Sticky Note5 | n8n-nodes-base.stickyNote | Section label | — | — | ## Report Generation & Distribution<br>Creates formatted PDF report and delivers via email and Google Drive storage. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Name it: **AI Incident Report Generator**
- Ensure workflow setting **Execution Order** is the default (`v1` in this workflow).

2) **Add “Webhook” node**
- Node type: **Webhook**
- Method: **POST**
- Path: **incident-report**
- Save the node to generate a test URL.
- Expected incoming JSON body fields (minimum):
  - `incident_id`, `title`, `description`, `severity`, `reported_by`, `timestamp`, `affected_systems` (array), `initial_impact`

3) **Add “Normalize Data” node (Set)**
- Node type: **Set**
- Enable “Keep Only Set” (optional; recommended for clean schema—this workflow effectively creates a new schema).
- Add fields:
  - `incident_id` = `{{$json.body.incident_id}}`
  - `title` = `{{$json.body.title}}`
  - `description` = `{{$json.body.description}}`
  - `severity` = `{{$json.body.severity.toLowerCase()}}`
  - `reported_by` = `{{$json.body.reported_by}}`
  - `timestamp` = `{{$json.body.timestamp}}`
  - `affected_systems` = `{{$json.body.affected_systems.join(', ')}}` (note: outputs string)
  - `initial_impact` = `{{$json.body.initial_impact}}`
  - `report_generated_at` = `{{$now.toFormat('yyyy-MM-dd HH:mm:ss')}}`
- Connect: **Webhook → Normalize Data**

4) **Add “Root Cause + Impact Analysis” node (OpenAI via LangChain)**
- Node type: **OpenAI (LangChain)** (`@n8n/n8n-nodes-langchain.openAi`)
- Credentials: create/connect **OpenAI API** credential.
- Model: **gpt-4**
- Temperature: **0.7**
- Messages:
  - System: instruct strict JSON output with keys `root_cause`, `impact_assessment`, `immediate_actions`, `long_term_actions` (as in the workflow).
  - User: include incident fields using expressions from the normalized item.
- Connect: **Normalize Data → Root Cause + Impact Analysis**

5) **Add “Parse AI Response” node (Code)**
- Node type: **Code**
- Paste logic to:
  - Extract response text from OpenAI node output (path used here: `output[0].content[0].text`)
  - `JSON.parse()` it
  - Merge with `$('Normalize Data').first().json`
- Connect: **Root Cause + Impact Analysis → Parse AI Response**

6) **Add “Check Severity” node (IF)**
- Node type: **IF**
- Condition: `{{$json.severity}}` **equals** `high`
- Connect: **Parse AI Response → Check Severity**

7) **Add Slack alert path**
- Add node: **Slack** named `Alerts`
  - Credentials: Slack API (token with permission to post)
  - Channel: select your ops channel
  - Message: include incident details + first two immediate actions.
- Connect: **Check Severity (true) → Alerts**

8) **Add “Data Recovery” node (Code)**
- Node type: **Code**
- Code returns the JSON from `Check Severity` to ignore Slack output:
  - `const incidentData = $('Check Severity').first().json; return [{ json: incidentData }];`
- Connect: **Alerts → Data Recovery**

9) **Add “Stop and Error” node**
- Node type: **Stop and Error**
- Error message: `Severity is not HIGH!`
- Connect: **Check Severity (false) → Stop and Error**

10) **Add “Convert to PDF” node**
- Node type: **HTML CSS to PDF** (`n8n-nodes-htmlcsstopdf.htmlcsstopdf`)
- Credentials: set your HTML-to-PDF provider API key/credential.
- HTML content: paste the provided HTML template that references `$json.*` fields.
- Output format: **file**
- Output filename/binary property: keep as `data` (or ensure downstream nodes match).
- Connect: **Data Recovery → Convert to PDF**

11) **Add “Upload file” node (Google Drive)**
- Node type: **Google Drive**
- Credentials: Google Drive OAuth2.
- Operation: Upload (file from binary)
- File name expression (as used):
  - `Incident_{{ $('Data Recovery').item.json.incident_id }}_{{ $('Data Recovery').item.json.severity }}_{{ $now.format('YYYY-MM-DD_HHmmss') }}.pdf`
- Drive: My Drive
- Folder: choose target folder
- Ensure the node is configured to use the PDF binary from `Convert to PDF` (commonly property `data`).
- Connect: **Convert to PDF → Upload file**

12) **Add “Send Email” node (Gmail)**
- Node type: **Gmail**
- Credentials: Gmail OAuth2.
- To: set your ops distribution list (replace `ops-team@yourcompany.com`)
- Subject: severity/title/incident id expression
- Body: paste the HTML template (it references `$('Data Recovery').item.json.*`)
- Attachments: configure to attach the PDF binary from `Convert to PDF` (binary property typically `data`).
- Connect: **Convert to PDF → Send Email**

13) **Add sticky notes (optional, for documentation)**
- Add sticky notes with the provided section headers and the “How it Works / Setup / Customization” content.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The workflow currently **stops** for non-high incidents (`Stop and Error`). The comment suggests “others proceed to reporting”, but implementation does not. | Update `Check Severity` routing if you want to generate reports for all severities (e.g., connect the false branch to `Convert to PDF` instead of `Stop and Error`). |
| `Normalize Data` sets `affected_systems` as an “array” field but assigns a joined string. | Consider making it a string field explicitly, or keep it as an array and join only in templates. |
| GPT output must be strict JSON or parsing will fail in `Parse AI Response`. | Consider lowering temperature, adding retry/fallback parsing, or using a “JSON output” enforced response format if available in your OpenAI node version. |
| Slack message assumes at least two immediate actions. | Add guards or render dynamically to avoid `undefined`. |
| HTML-to-PDF node is a community/custom integration. | Ensure `n8n-nodes-htmlcsstopdf` is installed and credentials are configured on your instance. |