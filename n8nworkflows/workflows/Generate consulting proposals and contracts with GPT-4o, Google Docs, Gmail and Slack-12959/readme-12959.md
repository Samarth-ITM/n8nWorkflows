Generate consulting proposals and contracts with GPT-4o, Google Docs, Gmail and Slack

https://n8nworkflows.xyz/workflows/generate-consulting-proposals-and-contracts-with-gpt-4o--google-docs--gmail-and-slack-12959


# Generate consulting proposals and contracts with GPT-4o, Google Docs, Gmail and Slack

## 1. Workflow Overview

**Workflow name:** Proposal & Contract Generation Automation for Consulting Firms  
**Given title:** Generate consulting proposals and contracts with GPT-4o, Google Docs, Gmail and Slack

**Purpose:**  
Automates the creation and delivery of consulting proposals (and initiates signature collection) when a new client entry appears in Google Sheets. It uses GPT-4o to draft proposal content, generates a Google Doc, converts it to PDF, emails it via Gmail, sends it to DocuSign (via HTTP), logs the activity back to Google Sheets, and notifies the team in Slack—adding a priority notification path for high-value clients. An error trigger sends a Slack alert to an admin channel.

### 1.1 Intake & Configuration
Triggered by a new Google Sheets entry, then applies workflow-wide configuration and prepares the data.

### 1.2 Data Normalization
Normalizes/standardizes client fields to make downstream prompts and templates reliable.

### 1.3 AI Proposal Drafting
Uses OpenAI (LangChain node) to generate proposal text based on normalized client data.

### 1.4 Document Creation & PDF Conversion
Creates a Google Doc proposal and converts it to PDF via an HTTP request (likely Google Drive export endpoint or a document conversion service).

### 1.5 Delivery & Signature
Emails the PDF proposal via Gmail, then sends it to DocuSign for signature via HTTP.

### 1.6 Logging & Notifications (Standard + Priority)
Logs the proposal creation in Google Sheets, posts a standard Slack notification, checks “high-value” status, and optionally posts a priority Slack message and schedules a follow-up reminder email.

### 1.7 Error Handling
Any workflow error triggers a Slack alert to admin.

---

## 2. Block-by-Block Analysis

### Block A — Intake & Configuration
**Overview:** Captures a new client row from Google Sheets and applies initial workflow configuration values used later.  
**Nodes involved:** `New Client Entry Trigger`, `Workflow Configuration`

#### Node: New Client Entry Trigger
- **Type / role:** Google Sheets Trigger (`n8n-nodes-base.googleSheetsTrigger`) — entry point.
- **Configuration (interpreted):** Not specified in JSON export (parameters are empty). In practice, it must be configured with:
  - Spreadsheet + sheet/tab
  - Trigger event (e.g., “new row”)
  - Polling/drive trigger behavior depending on n8n version
- **Inputs / outputs:** No inputs; outputs items representing the new row’s columns.
- **Connections:** Outputs to `Workflow Configuration`.
- **Version considerations:** `typeVersion: 1` (older trigger). Some UI fields differ from newer versions.
- **Edge cases / failures:**
  - Missing/expired Google credentials
  - Sheet permissions changed, sheet renamed, columns moved
  - Duplicate triggers if rows are edited or if polling picks up historical rows depending on configuration

#### Node: Workflow Configuration
- **Type / role:** Set node (`n8n-nodes-base.set`) — initializes/overrides fields for downstream use.
- **Configuration (interpreted):** Empty in JSON; typically used to:
  - Set constants (proposal template IDs, admin Slack channel, DocuSign endpoints)
  - Map incoming sheet columns to canonical names
- **Inputs / outputs:** Input from trigger; output to `Normalize Client Data`.
- **Connections:** Receives from `New Client Entry Trigger`, sends to `Normalize Client Data`.
- **Version considerations:** `typeVersion: 3.4`.
- **Edge cases / failures:** If left empty, downstream nodes may reference missing fields, causing expression errors.

---

### Block B — Data Normalization
**Overview:** Cleans and standardizes client data (names, emails, project scope, budget) to ensure consistent prompting and document creation.  
**Nodes involved:** `Normalize Client Data`

#### Node: Normalize Client Data
- **Type / role:** Set node (`n8n-nodes-base.set`) — normalization/mapping step.
- **Configuration (interpreted):** Empty in JSON; commonly used to:
  - Trim whitespace, ensure email casing, ensure numeric budget, default missing fields
  - Create derived fields like `clientFullName`, `isHighValueCandidate`, `proposalSubject`
- **Inputs / outputs:** Input from `Workflow Configuration`; output to `Generate Proposal Content`.
- **Connections:** Sends to `Generate Proposal Content`.
- **Version considerations:** `typeVersion: 3.4`.
- **Edge cases / failures:**
  - Missing columns from Sheets leading to null/undefined values
  - Budget parsing issues (e.g., “$10k”, “10,000”, “10k EUR”) unless normalized explicitly

---

### Block C — AI Proposal Drafting
**Overview:** Generates proposal text using GPT-4o (OpenAI via LangChain node) based on normalized client data.  
**Nodes involved:** `Generate Proposal Content`

#### Node: Generate Proposal Content
- **Type / role:** OpenAI (LangChain) node (`@n8n/n8n-nodes-langchain.openAi`) — LLM content generation.
- **Configuration (interpreted):** Empty in JSON; typically includes:
  - Model selection (e.g., GPT-4o)
  - System/user prompt that references normalized fields (client name, scope, timeline, pricing)
  - Output formatting instructions (sections, bullet points, headings)
- **Key expressions / variables:** Not shown; expected to reference `$json` fields from normalization.
- **Inputs / outputs:** Input is normalized client data; output should include generated text (often in `$json.text` or a similar field depending on node settings).
- **Connections:** Outputs to `Create Google Doc Proposal`.
- **Version considerations:** `typeVersion: 2.1` (LangChain node behavior differs from legacy OpenAI node).
- **Edge cases / failures:**
  - Missing OpenAI credentials / quota exceeded
  - Prompt producing overly long output causing downstream doc insertion limits
  - Non-deterministic formatting unless constrained (risk for template-driven doc creation)

---

### Block D — Document Creation & PDF Conversion
**Overview:** Creates a proposal in Google Docs, then converts it to a PDF via HTTP.  
**Nodes involved:** `Create Google Doc Proposal`, `Convert Doc to PDF`

#### Node: Create Google Doc Proposal
- **Type / role:** Google Docs node (`n8n-nodes-base.googleDocs`) — creates/populates a Doc.
- **Configuration (interpreted):** Empty in JSON; typically:
  - Create document action (new doc) OR copy from template
  - Insert generated proposal content into the doc body
  - Name the document using client/project fields (e.g., “Proposal - {{clientName}}”)
- **Inputs / outputs:** Input from OpenAI; output should include Doc ID / URL.
- **Connections:** Outputs to `Convert Doc to PDF`.
- **Version considerations:** `typeVersion: 2`.
- **Edge cases / failures:**
  - Google Docs permissions / template not accessible
  - Rate limits on Google APIs
  - If the LLM output includes unsupported formatting, insertion can be messy without preprocessing

#### Node: Convert Doc to PDF
- **Type / role:** HTTP Request node (`n8n-nodes-base.httpRequest`) — calls an external endpoint to export/convert.
- **Configuration (interpreted):** Empty in JSON; most common patterns:
  - Google Drive export endpoint: `GET https://www.googleapis.com/drive/v3/files/{docId}/export?mimeType=application/pdf`
  - Or Google Docs export link using OAuth2
  - Response expected as **binary** (PDF) for Gmail attachment
- **Inputs / outputs:** Input contains Doc ID/URL; output should contain binary PDF data (e.g., `data` property).
- **Connections:** Outputs to `Send Proposal via Email`.
- **Version considerations:** `typeVersion: 4.3` supports newer auth/binary handling.
- **Edge cases / failures:**
  - Auth mismatch: Google Docs node may work but HTTP conversion fails if not using the same OAuth scope/credential
  - Not setting “Download”/binary response properly leads to corrupted attachments
  - Large documents can time out

---

### Block E — Delivery & Signature
**Overview:** Emails the proposal PDF to the client, then initiates DocuSign signature flow via API.  
**Nodes involved:** `Send Proposal via Email`, `Send to DocuSign for Signature`

#### Node: Send Proposal via Email
- **Type / role:** Gmail node (`n8n-nodes-base.gmail`) — sends proposal email with PDF attachment.
- **Configuration (interpreted):** Empty in JSON; typically:
  - To: client email from normalized data
  - Subject/body: includes proposal highlights + doc link
  - Attachment: binary PDF from previous step
- **Inputs / outputs:** Input includes PDF binary; output is Gmail send result (messageId, threadId).
- **Connections:** Outputs to `Send to DocuSign for Signature`.
- **Version considerations:** `typeVersion: 2.2`.
- **Edge cases / failures:**
  - Gmail OAuth expired / insufficient scopes for sending
  - Attachment missing if binary property name mismatches
  - Deliverability: SPF/DKIM/“send as” restrictions if using aliases

#### Node: Send to DocuSign for Signature
- **Type / role:** HTTP Request node (`n8n-nodes-base.httpRequest`) — calls DocuSign API to create an envelope.
- **Configuration (interpreted):** Empty in JSON; expected:
  - DocuSign REST endpoint (e.g., `/envelopes`)
  - Auth (OAuth2/JWT) and headers
  - Payload includes recipient, subject, and the PDF (base64 or remote document reference)
- **Inputs / outputs:** Input from Gmail step; output should include envelopeId/status.
- **Connections:** Outputs to `Log Proposal Creation`.
- **Version considerations:** `typeVersion: 4.3`.
- **Edge cases / failures:**
  - DocuSign auth is commonly the hardest part (token refresh/JWT consent)
  - Incorrect document encoding/base64 causes envelope creation failure
  - Rate limiting and account-specific envelope constraints

---

### Block F — Logging & Notifications (Standard + Priority + Follow-up)
**Overview:** Logs proposal creation to Sheets, posts Slack notification, checks for high-value clients, optionally posts priority alert, then schedules a follow-up reminder email.  
**Nodes involved:** `Log Proposal Creation`, `Notify Team - Standard`, `Check if High-Value Client`, `Priority Slack Notification`, `Schedule Follow-up Reminder`

#### Node: Log Proposal Creation
- **Type / role:** Google Sheets node (`n8n-nodes-base.googleSheets`) — appends/updates a log row.
- **Configuration (interpreted):** Empty in JSON; typically:
  - Append a row to a “Proposals Log” sheet with client, date, doc URL, PDF status, envelopeId
- **Inputs / outputs:** Input includes DocuSign result; output is Sheets write result.
- **Connections:** Outputs to `Notify Team - Standard`.
- **Version considerations:** `typeVersion: 4.7`.
- **Edge cases / failures:**
  - Writing to protected ranges
  - Columns changed causing mapping mismatch
  - API quotas

#### Node: Notify Team - Standard
- **Type / role:** Slack node (`n8n-nodes-base.slack`) — standard team notification.
- **Configuration (interpreted):** Empty in JSON; expected:
  - Channel + message text summarizing proposal created/sent + links
- **Inputs / outputs:** Input from log step; output is Slack API response.
- **Connections:** Outputs to `Check if High-Value Client`.
- **Version considerations:** `typeVersion: 2.4`.
- **Edge cases / failures:** Invalid bot token/scopes, channel not found, message formatting errors.

#### Node: Check if High-Value Client
- **Type / role:** IF node (`n8n-nodes-base.if`) — branches based on client value.
- **Configuration (interpreted):** Empty in JSON; should compare something like:
  - Budget ≥ threshold, or client tier == “Enterprise”, or expected contract value
- **Inputs / outputs:** Input from Slack standard notification; outputs **true** path to priority flow. (False path is not connected in this workflow.)
- **Connections:** True output goes to `Priority Slack Notification`.
- **Version considerations:** `typeVersion: 2.3`.
- **Edge cases / failures:**
  - If condition not configured, behavior may default or always-false/always-true depending on UI state
  - Missing budget field leads to comparison failures

#### Node: Priority Slack Notification
- **Type / role:** Slack node (`n8n-nodes-base.slack`) — escalated alert for high-value clients.
- **Configuration (interpreted):** Empty in JSON; likely posts to a leadership channel or uses @here/@channel.
- **Inputs / outputs:** Input from IF true branch; output to follow-up scheduling.
- **Connections:** Outputs to `Schedule Follow-up Reminder`.
- **Version considerations:** `typeVersion: 2.4`.
- **Edge cases / failures:** Same as Slack standard; additionally risk of noisy mentions.

#### Node: Schedule Follow-up Reminder
- **Type / role:** Gmail node (`n8n-nodes-base.gmail`) — sends a follow-up reminder (despite the name “Schedule”, it is still Gmail).
- **Configuration (interpreted):** Empty in JSON; possibilities:
  - Sends an internal reminder email to account manager
  - Or sends a delayed reminder to the client (note: Gmail node itself does not “delay”; delay would require Wait/Cron)
- **Inputs / outputs:** Input from priority Slack node; output is Gmail send result.
- **Connections:** No outgoing connections (end of main flow).
- **Version considerations:** `typeVersion: 2.2`.
- **Edge cases / failures:**
  - If actual scheduling is desired, a `Wait` node or separate scheduled workflow is needed
  - Risk of sending reminders immediately instead of later

---

### Block G — Error Handling
**Overview:** Captures any workflow runtime error and alerts an admin via Slack.  
**Nodes involved:** `On Workflow Error`, `Error Alert to Admin`

#### Node: On Workflow Error
- **Type / role:** Error Trigger (`n8n-nodes-base.errorTrigger`) — secondary entry point that fires on workflow failure.
- **Configuration (interpreted):** Default: triggers when the workflow errors.
- **Inputs / outputs:** No inputs; output contains error context (failing node name, message, stack, execution id).
- **Connections:** Outputs to `Error Alert to Admin`.
- **Version considerations:** `typeVersion: 1`.
- **Edge cases / failures:** If Slack credentials fail too, errors may go unreported unless n8n-level alerting is enabled.

#### Node: Error Alert to Admin
- **Type / role:** Slack node (`n8n-nodes-base.slack`) — posts error details to admin channel.
- **Configuration (interpreted):** Empty in JSON; should map error fields into message.
- **Inputs / outputs:** Input from error trigger; outputs Slack API response.
- **Connections:** None after it.
- **Version considerations:** `typeVersion: 2.4`.
- **Edge cases / failures:** Slack auth/scope issues prevent visibility into failures.

---

### Sticky Notes
All sticky notes in the JSON have **empty content**, so there are no embedded comments/links to preserve:
- `Sticky Note`, `Sticky Note1`, `Sticky Note2`, `Sticky Note3`, `Sticky Note4`, `Sticky Note5`

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| New Client Entry Trigger | googleSheetsTrigger | Entry point: detects new client row | — | Workflow Configuration |  |
| Workflow Configuration | set | Initialize constants and base mappings | New Client Entry Trigger | Normalize Client Data |  |
| Normalize Client Data | set | Clean/standardize fields for downstream steps | Workflow Configuration | Generate Proposal Content |  |
| Generate Proposal Content | openAi (LangChain) | Generate proposal text with GPT-4o | Normalize Client Data | Create Google Doc Proposal |  |
| Create Google Doc Proposal | googleDocs | Create/populate Google Doc proposal | Generate Proposal Content | Convert Doc to PDF |  |
| Convert Doc to PDF | httpRequest | Export/convert Google Doc to PDF (binary) | Create Google Doc Proposal | Send Proposal via Email |  |
| Send Proposal via Email | gmail | Email proposal PDF to client | Convert Doc to PDF | Send to DocuSign for Signature |  |
| Send to DocuSign for Signature | httpRequest | Create DocuSign envelope for signature | Send Proposal via Email | Log Proposal Creation |  |
| Log Proposal Creation | googleSheets | Write proposal/envelope metadata to log sheet | Send to DocuSign for Signature | Notify Team - Standard |  |
| Notify Team - Standard | slack | Post standard notification to team | Log Proposal Creation | Check if High-Value Client |  |
| Check if High-Value Client | if | Branch on high-value criteria | Notify Team - Standard | Priority Slack Notification |  |
| Priority Slack Notification | slack | Escalated Slack alert for high-value clients | Check if High-Value Client | Schedule Follow-up Reminder |  |
| Schedule Follow-up Reminder | gmail | Send internal/client follow-up reminder email | Priority Slack Notification | — |  |
| On Workflow Error | errorTrigger | Entry point for runtime failures | — | Error Alert to Admin |  |
| Error Alert to Admin | slack | Send admin alert with error context | On Workflow Error | — |  |
| Sticky Note | stickyNote | Canvas annotation (empty) | — | — |  |
| Sticky Note1 | stickyNote | Canvas annotation (empty) | — | — |  |
| Sticky Note2 | stickyNote | Canvas annotation (empty) | — | — |  |
| Sticky Note3 | stickyNote | Canvas annotation (empty) | — | — |  |
| Sticky Note4 | stickyNote | Canvas annotation (empty) | — | — |  |
| Sticky Note5 | stickyNote | Canvas annotation (empty) | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named: *Proposal & Contract Generation Automation for Consulting Firms*.

2. **Add trigger: Google Sheets Trigger**
   - Node: **Google Sheets Trigger** named **New Client Entry Trigger**
   - Configure Google OAuth2 credentials (Google account with access to the sheet).
   - Select Spreadsheet and Sheet (tab) where new client rows are added.
   - Configure trigger event/polling so it fires only on new rows (avoid reprocessing existing rows).

3. **Add Set node: Workflow Configuration**
   - Node: **Set** named **Workflow Configuration**
   - Add fields you want as constants, for example:
     - `proposalTemplateDocId` (if you copy from a template)
     - `highValueThreshold` (number)
     - `internalOwnerEmail`, `slackChannelStandard`, `slackChannelPriority`
     - `docusignApiBaseUrl`
   - Connect: `New Client Entry Trigger → Workflow Configuration`.

4. **Add Set node: Normalize Client Data**
   - Node: **Set** named **Normalize Client Data**
   - Map sheet columns into canonical names, e.g.:
     - `clientName`, `clientEmail`, `company`, `projectScope`, `timeline`, `budget`
   - Optionally add computed fields:
     - numeric budget: parse to number
     - `proposalTitle` / `emailSubject`
   - Connect: `Workflow Configuration → Normalize Client Data`.

5. **Add OpenAI (LangChain) node: Generate Proposal Content**
   - Node: **OpenAI** (LangChain) named **Generate Proposal Content**
   - Configure OpenAI credentials (API key).
   - Select model **GPT-4o** (or equivalent available).
   - Prompt structure (example intent):
     - System: you are a consulting proposals assistant.
     - User: include `{{$json.clientName}}`, `{{$json.projectScope}}`, `{{$json.timeline}}`, `{{$json.budget}}`, output sections (Objectives, Approach, Deliverables, Timeline, Pricing, Terms).
   - Ensure the node outputs the proposal text in a predictable field (configure output key if available).
   - Connect: `Normalize Client Data → Generate Proposal Content`.

6. **Add Google Docs node: Create Google Doc Proposal**
   - Node: **Google Docs** named **Create Google Doc Proposal**
   - Configure Google credentials (same or another Google OAuth2 with Docs access).
   - Choose an operation:
     - **Create** a document and insert generated content, or
     - **Copy from template** then replace placeholders with client fields + AI text.
   - Ensure the output includes the created document ID and URL.
   - Connect: `Generate Proposal Content → Create Google Doc Proposal`.

7. **Add HTTP Request node: Convert Doc to PDF**
   - Node: **HTTP Request** named **Convert Doc to PDF**
   - Configure to export the Doc as PDF (common approach: Google Drive API export).
     - Method: `GET`
     - URL: Drive export endpoint using the doc/file ID from previous node
     - Authentication: Google OAuth2 with Drive scopes
     - Response: **File/Binary**
     - Binary property name (example): `proposalPdf`
   - Connect: `Create Google Doc Proposal → Convert Doc to PDF`.

8. **Add Gmail node: Send Proposal via Email**
   - Node: **Gmail** named **Send Proposal via Email**
   - Configure Gmail OAuth2 credentials.
   - Operation: Send email
   - To: `{{$json.clientEmail}}`
   - Subject/body: reference client/project fields; optionally include Google Doc link.
   - Attach the PDF using the binary property from the previous step (e.g., `proposalPdf`).
   - Connect: `Convert Doc to PDF → Send Proposal via Email`.

9. **Add HTTP Request node: Send to DocuSign for Signature**
   - Node: **HTTP Request** named **Send to DocuSign for Signature**
   - Configure DocuSign authentication (OAuth2/JWT depending on your setup).
   - Operation: Create envelope with document + recipient(s).
   - Provide payload including:
     - recipient name/email
     - document (base64-encoded PDF) or a supported document reference method
   - Connect: `Send Proposal via Email → Send to DocuSign for Signature`.

10. **Add Google Sheets node: Log Proposal Creation**
    - Node: **Google Sheets** named **Log Proposal Creation**
    - Configure to append a row to a logging sheet.
    - Log fields such as: timestamp, client, doc URL, email messageId, DocuSign envelopeId/status.
    - Connect: `Send to DocuSign for Signature → Log Proposal Creation`.

11. **Add Slack node: Notify Team - Standard**
    - Node: **Slack** named **Notify Team - Standard**
    - Configure Slack credentials (bot token) and pick channel.
    - Message should include key metadata + links.
    - Connect: `Log Proposal Creation → Notify Team - Standard`.

12. **Add IF node: Check if High-Value Client**
    - Node: **IF** named **Check if High-Value Client**
    - Configure condition, e.g. `budget >= highValueThreshold` or `tier == 'Enterprise'`.
    - Connect: `Notify Team - Standard → Check if High-Value Client`.
    - Connect **true** output to next priority Slack node (false path can end or go to a different node if desired).

13. **Add Slack node: Priority Slack Notification**
    - Node: **Slack** named **Priority Slack Notification**
    - Configure channel/message for escalation (optionally mention stakeholders).
    - Connect: `Check if High-Value Client (true) → Priority Slack Notification`.

14. **Add Gmail node: Schedule Follow-up Reminder**
    - Node: **Gmail** named **Schedule Follow-up Reminder**
    - Configure recipient (internal owner) and reminder content.
    - If you truly need delayed sending, add a **Wait** node before this step (not present in the provided workflow).
    - Connect: `Priority Slack Notification → Schedule Follow-up Reminder`.

15. **Add Error Trigger + Slack alert**
    - Node: **Error Trigger** named **On Workflow Error**
    - Node: **Slack** named **Error Alert to Admin**
    - Configure Slack channel for admin alerts and map error info into message text.
    - Connect: `On Workflow Error → Error Alert to Admin`.

16. **Activate workflow** after testing with a sample row in Google Sheets.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Disclaimer (provided): “Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n… Toutes les données manipulées sont légales et publiques.” | Project/content compliance context |
| Sticky notes exist but contain no content. | No additional embedded documentation available in the workflow canvas |

