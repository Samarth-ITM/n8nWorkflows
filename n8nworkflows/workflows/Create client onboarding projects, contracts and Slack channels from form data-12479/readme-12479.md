Create client onboarding projects, contracts and Slack channels from form data

https://n8nworkflows.xyz/workflows/create-client-onboarding-projects--contracts-and-slack-channels-from-form-data-12479


# Create client onboarding projects, contracts and Slack channels from form data

## 1. Workflow Overview

**Workflow name (in JSON):** *Automate client onboarding: form to Asana projects, contracts & Slack*  
**Provided title:** *Create client onboarding projects, contracts and Slack channels from form data*

**Purpose:**  
Automates a service-business client onboarding pipeline from a submitted intake form: it receives form data via webhook, creates and structures a new Asana onboarding project based on a template project, generates a customized contract from a Google Docs template (then downloads it as PDF), emails the contract to the client via Gmail, logs all intake details to a Google Sheet, and creates a Slack channel for internal coordination.

### 1.1 Input Reception & Normalization
Receives a webhook POST from an intake form and normalizes 70+ fields into a predictable JSON structure.

### 1.2 Asana Project Creation & Template Replication
Creates a new Asana project, copies sections from a template project, pulls tasks per template section, and adds tasks into the new project/section.

### 1.3 Contract Generation (Google Docs/Drive)
Loads a Google Docs template, replaces placeholder strings with client-specific values, and downloads a PDF version.

### 1.4 Client Email Delivery (Gmail)
Sends a welcome email to the primary contact with the generated PDF attached.

### 1.5 Post-processing: Reset Template, Log to Sheets, Create Slack Channel
Resets the template placeholders back to blanks, appends/updates a row in Google Sheets, then creates a Slack channel.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception & Data Processing

**Overview:**  
Captures intake form submissions and maps the incoming `body` fields to top-level workflow fields used downstream (Asana naming, contract placeholders, tracking).

**Nodes involved:**  
- Receive Client Form  
- Parse Client Data

#### Node: Receive Client Form
- **Type / role:** Webhook trigger (`n8n-nodes-base.webhook`) — entry point.
- **Configuration (interpreted):**
  - HTTP Method: `POST`
  - Path: `client-onboard` (endpoint becomes `/webhook/client-onboard` or `/webhook-test/client-onboard` depending on mode)
- **Key variables/expressions:** none; relies on `{{$json.body...}}` downstream.
- **Outputs:** Sends webhook payload (expected to include `body` object) to **Parse Client Data**.
- **Version notes:** `typeVersion 2.1`
- **Edge cases / failures:**
  - If the form doesn’t POST JSON or n8n does not parse body as expected, `body.*` fields will be undefined.
  - Missing required fields (e.g., `primary_contact_email`, `primary_contact_name`) will break later expressions (email, Asana project name).

#### Node: Parse Client Data
- **Type / role:** Set node (`n8n-nodes-base.set`) — schema normalization and field extraction.
- **Configuration choices:**
  - Creates many fields like `legal_business_name`, `brand_name`, `primary_contact_email`, arrays like `services`, `api_integrations`, etc.
  - Values are pulled from the webhook payload: `={{ $json.body.<field> }}`
- **Key expressions/variables:**
  - Heavy usage of `$json.body.*`
- **Connections:**
  - **Input:** Receive Client Form
  - **Output:** Create Asana Project
- **Version notes:** `typeVersion 3.4`
- **Edge cases / failures:**
  - Array fields depend on the form sending arrays (not comma-separated strings). If the form sends strings, the downstream Google Sheets node may store unexpected format.
  - No validation is performed; empty strings/undefined values flow into contract replacements and Asana naming.

**Sticky notes applying:**
- “Section: Intake” (Webhook & Data Processing)
- “Workflow Overview” (global)
- “Credentials Note” (global)

---

### Block 2 — Asana Project Setup & Template Replication

**Overview:**  
Creates a new client project in Asana and attempts to replicate the structure of a template project by creating corresponding sections and associating template tasks to those sections.

**Nodes involved:**  
- Create Asana Project  
- Get Template Sections  
- Split Sections  
- Loop Through Sections  
- Create Section in New Project  
- Get Tasks from Template Section  
- Add Task to New Project  
- Wait for All Tasks

#### Node: Create Asana Project
- **Type / role:** Asana node (`n8n-nodes-base.asana`) — project creation.
- **Configuration choices:**
  - Resource: `project`, Operation: `create`
  - Name: `={{ $json.primary_contact_name }} Onboarding`
  - Team: `[YOUR_TEAM_ID]` (placeholder)
  - Workspace: `[YOUR_WORKSPACE_ID]` (placeholder)
  - Authentication: OAuth2
- **Connections:**
  - **Input:** Parse Client Data
  - **Output:** Get Template Sections
- **Edge cases / failures:**
  - Invalid Team/Workspace IDs or missing OAuth scopes cause authorization errors.
  - Name uses `primary_contact_name`; if missing, project name becomes `" Onboarding"`.

#### Node: Get Template Sections
- **Type / role:** HTTP Request (`n8n-nodes-base.httpRequest`) — calls Asana API directly.
- **Configuration choices:**
  - GET `https://app.asana.com/api/1.0/projects/[TEMPLATE_PROJECT_ID]/sections`
  - Auth: predefined credential type `asanaOAuth2Api`
- **Connections:**
  - **Input:** Create Asana Project
  - **Output:** Split Sections
- **Edge cases / failures:**
  - `[TEMPLATE_PROJECT_ID]` must be replaced.
  - If Asana API rate limits or returns non-200, downstream split fails (`data` missing).

#### Node: Split Sections
- **Type / role:** Split Out (`n8n-nodes-base.splitOut`) — iterates sections list.
- **Configuration choices:**
  - Field to split: `data` (expects Asana response format `{ data: [...] }`)
- **Connections:**
  - **Input:** Get Template Sections
  - **Output:** Loop Through Sections
- **Edge cases / failures:**
  - If API response is not in `{data: []}`, node outputs nothing or errors.

#### Node: Loop Through Sections
- **Type / role:** Split In Batches (`n8n-nodes-base.splitInBatches`) — controls iteration.
- **Configuration choices:**
  - Uses default batch options (not set explicitly).
- **Connections (important):**
  - **Input:** Split Sections
  - **Outputs:**
    - To **Create Section in New Project** (main output index 1 in JSON ordering, but effectively a branch)
    - To **Wait for All Tasks** (main output index 0) — intended as “done” path
  - Receives loop-back from **Add Task to New Project**
- **Edge cases / failures:**
  - Because of how connections are wired, **Wait for All Tasks** may run before all tasks are actually processed, depending on batch behavior and item counts.
  - If batch size is default and tasks are many, execution time and rate limits are likely.

#### Node: Create Section in New Project
- **Type / role:** HTTP Request — creates a section in the newly created Asana project.
- **Configuration choices:**
  - POST `https://app.asana.com/api/1.0/projects/{{ $('Create Asana Project').item.json.gid }}/sections`
  - JSON body: `{ "data": { "name": "{{ $json.name }}" } }`
  - Uses `asanaOAuth2Api` predefined credential
- **Key expressions:**
  - References prior node’s created project gid: `$('Create Asana Project').item.json.gid`
  - Uses template section name: `$json.name`
- **Connections:**
  - **Input:** Loop Through Sections (each template section)
  - **Output:** Get Tasks from Template Section
- **Edge cases / failures:**
  - If project gid is not available (failed project creation), request fails.
  - If a section name includes unsupported characters (rare), Asana may reject.

#### Node: Get Tasks from Template Section
- **Type / role:** Asana node — fetch tasks for a given template section.
- **Configuration choices:**
  - Operation: `getAll`
  - Filter: `section = {{ $('Loop Through Sections').item.json.gid }}`
  - Auth: OAuth2
- **Connections:**
  - **Input:** Create Section in New Project
  - **Output:** Add Task to New Project
- **Edge cases / failures:**
  - This retrieves tasks from the **template** section gid. If the template project is private/inaccessible, it fails.
  - Pagination/large task counts may increase runtime.

#### Node: Add Task to New Project
- **Type / role:** Asana node — associates an existing task to a project/section.
- **Configuration choices:**
  - Resource: `taskProject` (Asana “add task to project” relationship)
  - Task ID: `={{ $json.gid }}` (task from template section listing)
  - Project: `={{ $('Create Section in New Project').item.json.data.project.gid }}`
  - Additional field `section`: `={{ $('Create Section in New Project').item.json.data.gid }}`
- **Connections:**
  - **Input:** Get Tasks from Template Section
  - **Output:** Loop Through Sections (loop-back)
- **Edge cases / failures (critical design issue):**
  - This node appears to **add the template tasks themselves** into the new project, rather than duplicating them. In Asana, tasks can belong to multiple projects; this may unintentionally link template tasks to client projects (polluting the template).
  - If your intent is “duplicate tasks,” you need “create task” (or Asana API task duplication) rather than “taskProject”.
  - Rate limiting likely if sections/tasks are many.

#### Node: Wait for All Tasks
- **Type / role:** Aggregate (`n8n-nodes-base.aggregate`) — attempts to wait/collect before continuing.
- **Configuration choices:**
  - Aggregates field `success` (but upstream nodes do not clearly set `success`).
- **Connections:**
  - **Input:** Loop Through Sections (the “done” output)
  - **Output:** Get Contract Template
- **Edge cases / failures:**
  - If no `success` field exists, aggregation may produce empty results or unexpected structure.
  - This does not truly guarantee all asynchronous API calls are complete; it only aggregates incoming items.

**Sticky notes applying:**
- “Section: Project” (Asana Project Setup)
- “Credentials Note” (global)

---

### Block 3 — Contract Generation (Google Docs + Drive)

**Overview:**  
Loads a Google Docs template, performs multiple text replacements, then downloads the resulting document as a PDF.

**Nodes involved:**  
- Get Contract Template  
- Populate Contract with Client Data  
- Download Contract as PDF

#### Node: Get Contract Template
- **Type / role:** Google Docs (`n8n-nodes-base.googleDocs`) — fetches a document.
- **Configuration choices:**
  - Operation: `get`
  - Document URL/ID: `[YOUR_TEMPLATE_DOCUMENT_ID]` (placeholder)
  - `simple: false` (returns fuller metadata)
- **Connections:**
  - **Input:** Wait for All Tasks
  - **Output:** Populate Contract with Client Data
- **Edge cases / failures:**
  - Wrong document ID, missing permissions, or Google OAuth scope issues.

#### Node: Populate Contract with Client Data
- **Type / role:** Google Docs — updates a document via replace operations.
- **Configuration choices:**
  - Operation: `update` with multiple `replaceAll` actions.
  - Replaces template placeholders like `Effective Date- `, `Client-`, `Title-`, etc.
- **Key expressions/variables:**
  - Uses `$now.format('DD')` for dates (day-of-month only).
  - Uses `$('Parse Client Data').item.json.*` for client fields.
  - **Potentially broken references:** uses fields that do not exist in Parse Client Data:
    - `payment`, `pharmacy`, `asynchronous`, `synchronous_visit`, `synchronous`
- **Connections:**
  - **Input:** Get Contract Template
  - **Output:** Download Contract as PDF
- **Edge cases / failures:**
  - If placeholder text in the Google Doc does not exactly match `text` fields, replacements do nothing silently (contract remains unpopulated).
  - Undefined variables in expressions may become empty strings, producing malformed pricing lines.
  - Using only `DD` likely not desired for full effective date (missing month/year).

#### Node: Download Contract as PDF
- **Type / role:** Google Drive (`n8n-nodes-base.googleDrive`) — downloads file content.
- **Configuration choices:**
  - Operation: `download`
  - File ID: `={{ $json.documentId }}` (expects documentId from previous Google Docs node output)
- **Connections:**
  - **Input:** Populate Contract with Client Data
  - **Output:** Send Welcome Email with Contract
- **Edge cases / failures:**
  - Depending on Google Docs node output, the identifier might be `documentId` or `id`. If mismatched, download fails.
  - Download may return binary; must be wired to Gmail attachment correctly (see email block).

**Sticky notes applying:**
- “Section: Contract” (Contract Generation)
- “Credentials Note” (global)

---

### Block 4 — Client Communication (Gmail)

**Overview:**  
Sends a personalized welcome email to the primary contact and attempts to attach the generated contract.

**Nodes involved:**  
- Send Welcome Email with Contract

#### Node: Send Welcome Email with Contract
- **Type / role:** Gmail (`n8n-nodes-base.gmail`) — outbound email.
- **Configuration choices:**
  - To: `={{ $('Parse Client Data').item.json.primary_contact_email }}`
  - Subject contains contact name
  - Email body is plain text (`emailType: text`) and includes placeholders like `[YOUR_CALENDLY_LINK]`, `[YOUR_FORM_LINK]`
  - Attachments configured via UI: `attachmentsBinary` is present but not mapped to a named binary property
- **Connections:**
  - **Input:** Download Contract as PDF
  - **Output:** Reset Template to Blanks
- **Edge cases / failures (important):**
  - Attachment likely **not actually attached** unless the Google Drive download binary property name matches what Gmail expects and it is referenced in `attachmentsBinary`.
  - Subject/message include “🎉” which is fine, but keep encoding in mind.
  - Gmail OAuth must permit sending email; may fail with restricted scopes or from-address constraints.

**Sticky notes applying:**
- “Section: Email” (Client Communication)
- “Credentials Note” (global)

---

### Block 5 — Reset, Logging & Slack Setup

**Overview:**  
Resets the contract template placeholders back to the original blank markers, writes the full intake record into Google Sheets, and creates a Slack channel.

**Nodes involved:**  
- Reset Template to Blanks  
- Log to Tracking Sheet  
- Create Slack Channel

#### Node: Reset Template to Blanks
- **Type / role:** Google Docs — reverses replacements to “clean” the template for next run.
- **Configuration choices:**
  - Operation: `update` with multiple `replaceAll` actions that replace the populated strings back to placeholders.
- **Key expressions/variables (critical issues):**
  - Several “text” values include expressions referencing fields that **do not exist** in Parse Client Data (and some names differ):
    - `Client_name`, `Title `, `Signature`, `Date`, `onboarding`, `wizlo_platform`, etc.
  - Reset logic assumes the populated text exactly equals the evaluated strings, which is brittle.
- **Connections:**
  - **Input:** Send Welcome Email with Contract
  - **Output:** Log to Tracking Sheet
- **Edge cases / failures:**
  - If populated values differ (extra spaces, different formatting, different date), reset won’t match and template remains “dirty”.
  - Consider duplicating the template per client instead (safer) rather than mutating a single shared template.

#### Node: Log to Tracking Sheet
- **Type / role:** Google Sheets (`n8n-nodes-base.googleSheets`) — persists client onboarding record.
- **Configuration choices:**
  - Operation: `appendOrUpdate`
  - Spreadsheet ID: `[YOUR_TRACKING_SPREADSHEET_ID]` (placeholder)
  - Sheet: `Sheet1` (gid=0)
  - Matching column: `legal_business_name`
  - Mapping mode: “define below” with a very large column mapping
  - `attemptToConvertTypes: false` (keeps data as-is)
- **Connections:**
  - **Input:** Reset Template to Blanks
  - **Output:** Create Slack Channel
- **Edge cases / failures:**
  - If `legal_business_name` is empty, appendOrUpdate matching may behave unexpectedly (duplicates, failed match).
  - Several mapped fields appear inconsistent (example: `additional_notes` mapped to `operations_lead_name`), which may be a mistake.
  - Arrays may be written as `[object Object]` or comma-joined depending on node behavior; consider explicit joining.

#### Node: Create Slack Channel
- **Type / role:** Slack (`n8n-nodes-base.slack`) — channel operation.
- **Configuration choices:**
  - Resource: `channel`
  - **Only `channelId: onboarding` is set**; operation is not explicit in JSON.
- **Connections:**
  - **Input:** Log to Tracking Sheet
  - **Output:** none (end)
- **Edge cases / failures (likely misconfigured):**
  - To create a channel, Slack node typically needs an operation like “create” and a channel name. `channelId: onboarding` suggests it might be targeting an existing channel rather than creating a new one.
  - Slack API token must have `channels:manage` / `conversations:write` depending on workspace settings.
  - If the intent is “dedicated client channel”, the workflow currently doesn’t build a unique name from client data (e.g., `client-<brand>`).

**Sticky notes applying:**
- “Section: Logging” (Data Logging & Team Setup)
- “Credentials Note” (global)

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Workflow Overview | Sticky Note | Documentation / overview | — | — | ## 🚀 Client Onboarding Automation … (setup steps and description) |
| Section: Intake | Sticky Note | Visual section header | — | — | ## 📥 Webhook & Data Processing … |
| Section: Project | Sticky Note | Visual section header | — | — | ## 📋 Asana Project Setup … |
| Section: Contract | Sticky Note | Visual section header | — | — | ## 📄 Contract Generation … |
| Section: Email | Sticky Note | Visual section header | — | — | ## 📧 Client Communication … |
| Section: Logging | Sticky Note | Visual section header | — | — | ## 💾 Data Logging & Team Setup … |
| Credentials Note | Sticky Note | Credential/security reminder | — | — | ## 🔐 Credentials & Security … |
| Receive Client Form | Webhook | Intake entrypoint | — | Parse Client Data | ## 📥 Webhook & Data Processing … / ## 🔐 Credentials & Security … |
| Parse Client Data | Set | Normalize webhook payload | Receive Client Form | Create Asana Project | ## 📥 Webhook & Data Processing … |
| Create Asana Project | Asana | Create client project | Parse Client Data | Get Template Sections | ## 📋 Asana Project Setup … |
| Get Template Sections | HTTP Request | Fetch template project sections | Create Asana Project | Split Sections | ## 📋 Asana Project Setup … |
| Split Sections | Split Out | Explode sections array into items | Get Template Sections | Loop Through Sections | ## 📋 Asana Project Setup … |
| Loop Through Sections | Split In Batches | Iterate sections / manage loop | Split Sections; Add Task to New Project | Create Section in New Project; Wait for All Tasks | ## 📋 Asana Project Setup … |
| Create Section in New Project | HTTP Request | Create corresponding section in new project | Loop Through Sections | Get Tasks from Template Section | ## 📋 Asana Project Setup … |
| Get Tasks from Template Section | Asana | List tasks in a template section | Create Section in New Project | Add Task to New Project | ## 📋 Asana Project Setup … |
| Add Task to New Project | Asana | Associate task to new project+section | Get Tasks from Template Section | Loop Through Sections | ## 📋 Asana Project Setup … |
| Wait for All Tasks | Aggregate | Aggregate before contract generation | Loop Through Sections | Get Contract Template | ## 📋 Asana Project Setup … |
| Get Contract Template | Google Docs | Retrieve template doc | Wait for All Tasks | Populate Contract with Client Data | ## 📄 Contract Generation … |
| Populate Contract with Client Data | Google Docs | Replace placeholders with client data | Get Contract Template | Download Contract as PDF | ## 📄 Contract Generation … |
| Download Contract as PDF | Google Drive | Download document binary | Populate Contract with Client Data | Send Welcome Email with Contract | ## 📄 Contract Generation … |
| Send Welcome Email with Contract | Gmail | Email client + attach contract | Download Contract as PDF | Reset Template to Blanks | ## 📧 Client Communication … |
| Reset Template to Blanks | Google Docs | Revert placeholders in template | Send Welcome Email with Contract | Log to Tracking Sheet | ## 📄 Contract Generation … / (also affects logging flow) |
| Log to Tracking Sheet | Google Sheets | Append/update intake record | Reset Template to Blanks | Create Slack Channel | ## 💾 Data Logging & Team Setup … |
| Create Slack Channel | Slack | Create/target onboarding channel | Log to Tracking Sheet | — | ## 💾 Data Logging & Team Setup … |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
2. **Add Webhook node** named **Receive Client Form**
   - Method: `POST`
   - Path: `client-onboard`
   - Use your form tool to POST JSON to the webhook test URL while building.
3. **Add Set node** named **Parse Client Data**
   - Add fields mapping from `{{$json.body.<field>}}` into top-level fields (at minimum):
     - `primary_contact_name`, `primary_contact_email`, `primary_contact_role`
     - `brand_name`, `legal_business_name`, `terms_fees`, `status`, etc.
   - Connect: **Receive Client Form → Parse Client Data**
4. **Add Asana node** named **Create Asana Project**
   - Auth: Asana OAuth2 credentials
   - Resource: Project, Operation: Create
   - Workspace: your workspace ID
   - Team: your team ID
   - Name: `{{$json.primary_contact_name}} Onboarding`
   - Connect: **Parse Client Data → Create Asana Project**
5. **Add HTTP Request node** named **Get Template Sections**
   - Method: GET
   - URL: `https://app.asana.com/api/1.0/projects/<TEMPLATE_PROJECT_ID>/sections`
   - Auth: Asana OAuth2 credential (predefined credential type)
   - Connect: **Create Asana Project → Get Template Sections**
6. **Add Split Out node** named **Split Sections**
   - Field to split out: `data`
   - Connect: **Get Template Sections → Split Sections**
7. **Add Split In Batches node** named **Loop Through Sections**
   - Keep defaults (or set batch size = 1 for clarity)
   - Connect: **Split Sections → Loop Through Sections**
8. **Add HTTP Request node** named **Create Section in New Project**
   - Method: POST
   - URL: `https://app.asana.com/api/1.0/projects/{{ $('Create Asana Project').item.json.gid }}/sections`
   - Body type: JSON
   - Body: `{ "data": { "name": "{{ $json.name }}" } }`
   - Auth: Asana OAuth2
   - Connect: **Loop Through Sections → Create Section in New Project**
9. **Add Asana node** named **Get Tasks from Template Section**
   - Operation: Get Many / Get All tasks
   - Filter by Section: `{{ $('Loop Through Sections').item.json.gid }}`
   - Connect: **Create Section in New Project → Get Tasks from Template Section**
10. **Add Asana node** named **Add Task to New Project**
    - Resource: Task-Project relationship (often “Add task to project”)
    - Task ID: `{{$json.gid}}`
    - Project ID: `{{ $('Create Section in New Project').item.json.data.project.gid }}`
    - Section ID (additional field): `{{ $('Create Section in New Project').item.json.data.gid }}`
    - Connect: **Get Tasks from Template Section → Add Task to New Project**
11. **Loop back**
    - Connect: **Add Task to New Project → Loop Through Sections** (to continue batches)
12. **Add Aggregate node** named **Wait for All Tasks**
    - Configure it to aggregate items in a way that matches your loop output (current workflow aggregates a `success` field; you may instead aggregate any field or simply use “Merge” patterns).
    - Connect: **Loop Through Sections (done output) → Wait for All Tasks**
13. **Add Google Docs node** named **Get Contract Template**
    - Auth: Google OAuth2 (Docs)
    - Operation: Get
    - Document ID/URL: your template document
    - Connect: **Wait for All Tasks → Get Contract Template**
14. **Add Google Docs node** named **Populate Contract with Client Data**
    - Operation: Update
    - Add multiple “Replace all” actions to replace your placeholders with expressions from **Parse Client Data**
    - Connect: **Get Contract Template → Populate Contract with Client Data**
15. **Add Google Drive node** named **Download Contract as PDF**
    - Auth: Google OAuth2 (Drive)
    - Operation: Download
    - File ID: map from the Google Docs output (ensure it’s the correct property name)
    - Connect: **Populate Contract with Client Data → Download Contract as PDF**
16. **Add Gmail node** named **Send Welcome Email with Contract**
    - Auth: Google OAuth2 (Gmail)
    - To: `{{ $('Parse Client Data').item.json.primary_contact_email }}`
    - Subject/body as desired (include Calendly/form links)
    - Attach the binary output from **Download Contract as PDF** by selecting the correct binary property name.
    - Connect: **Download Contract as PDF → Send Welcome Email with Contract**
17. **Add Google Docs node** named **Reset Template to Blanks**
    - Operation: Update
    - Add reverse “Replace all” actions to restore placeholders (or preferably: change design to copy template per client).
    - Connect: **Send Welcome Email with Contract → Reset Template to Blanks**
18. **Add Google Sheets node** named **Log to Tracking Sheet**
    - Auth: Google OAuth2 (Sheets)
    - Operation: Append or Update
    - Spreadsheet: your tracking sheet
    - Sheet tab: your target sheet
    - Matching column: choose a stable key (email or a generated client ID is safer than legal name)
    - Map columns from **Parse Client Data**
    - Connect: **Reset Template to Blanks → Log to Tracking Sheet**
19. **Add Slack node** named **Create Slack Channel**
    - Auth: Slack token / OAuth2 (depending on node setup)
    - Operation: Create Channel (ensure operation is explicitly set)
    - Channel name: build from client, e.g. `onboarding-{{ $json.brand_name.toLowerCase().replaceAll(' ', '-') }}`
    - Connect: **Log to Tracking Sheet → Create Slack Channel**
20. **Credentials**
    - Configure: Asana OAuth2, Google Docs/Drive/Sheets OAuth2, Gmail OAuth2, Slack token.
    - Replace all placeholders: `[YOUR_TEAM_ID]`, `[YOUR_WORKSPACE_ID]`, `[TEMPLATE_PROJECT_ID]`, `[YOUR_TEMPLATE_DOCUMENT_ID]`, `[YOUR_TRACKING_SPREADSHEET_ID]`, link placeholders in email.
21. **Test end-to-end**
    - Use webhook test URL + sample payload.
    - Verify: Asana project/sections/tasks, PDF generation, email attachment, sheets row, Slack channel.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “This workflow automates the complete client onboarding process… creates a project in Asana… generates a customized service agreement… emails the contract… sets up communication channels… logs to Google Sheets.” | From sticky note “Workflow Overview” |
| Setup steps: connect Asana/Google/Slack; create Asana template project; create Google Docs template with placeholders; configure webhook in intake form; update template IDs/workspace refs; test with sample data. | From sticky note “Workflow Overview” |
| Required credentials: Asana OAuth2, Google Docs/Drive/Sheets OAuth2, Gmail OAuth2, Slack API token. Replace template IDs, workspace numbers, and email addresses. | From sticky note “Credentials Note” |
| Disclaimer: *Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n…* | Provided by user (applies globally) |