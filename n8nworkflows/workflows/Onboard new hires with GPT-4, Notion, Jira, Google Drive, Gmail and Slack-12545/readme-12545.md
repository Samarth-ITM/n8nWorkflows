Onboard new hires with GPT-4, Notion, Jira, Google Drive, Gmail and Slack

https://n8nworkflows.xyz/workflows/onboard-new-hires-with-gpt-4--notion--jira--google-drive--gmail-and-slack-12545


# Onboard new hires with GPT-4, Notion, Jira, Google Drive, Gmail and Slack

## 1. Workflow Overview

**Title:** Onboard new hires with GPT-4, Notion, Jira, Google Drive, Gmail and Slack

**Purpose:** Automate end-to-end employee onboarding from an incoming HRIS-style webhook. The workflow validates and enriches employee data, generates a personalized welcome message using GPT-4.1-mini, assembles a role-based PDF onboarding package from Google Drive templates, archives artifacts, opens an IT provisioning Jira ticket, emails the new hire an HTML welcome message, and announces the hire in Slack.

### 1.1 Intake & Enrichment
Receives new hire payload via webhook, validates required fields, classifies role/department, and computes required system access + metadata.

### 1.2 Tracking & AI Content
Creates a Notion onboarding tracker entry and generates a personalized welcome message using an AI Agent powered by OpenAI Chat Model.

### 1.3 Document Processing & Consolidation
Lists role-based policy/templates from Google Drive, downloads each file, merges them into one PDF, and merges that PDF output with the AI-generated content stream.

### 1.4 Delivery & Notifications
Archives onboarding artifacts to the employee folder on Drive, generates an IT provisioning checklist, creates a Jira ticket, sends the welcome email via Gmail, and posts a Slack announcement.

---

## 2. Block-by-Block Analysis

### Block 1.1 — Intake & Enrichment
**Overview:** Accepts the onboarding request, ensures minimum fields exist, and enriches the payload with derived fields used by downstream systems (Notion, Drive, Jira, email, Slack).

**Nodes involved:**
- Trigger: New Hire Webhook1
- Validate & Enrich Employee Data1

#### Node: Trigger: New Hire Webhook1
- **Type / role:** `Webhook` — entry point (HTTP POST).
- **Configuration:**  
  - Method: `POST`  
  - Path: `onboard-employee` (so endpoint is typically `/webhook/onboard-employee` or `/webhook-test/onboard-employee` depending on mode)
- **Outputs:** Passes request payload to validation node.
- **Failure/edge cases:**
  - Missing/invalid JSON body from caller.
  - Caller not sending `Content-Type: application/json`.
  - Production vs test webhook URL mismatch.

#### Node: Validate & Enrich Employee Data1
- **Type / role:** `Code` — validation + enrichment (business logic).
- **Key logic (interpreted):**
  - Reads `item = input.json.body || input.json` to support both raw JSON and webhook body.
  - Validates required fields: `firstName`, `lastName`, `email` (throws error if missing).
  - Derives:
    - `fullName`, `generatedAt`, `startDate` default to today (YYYY-MM-DD)
    - Role flags by substring match on `jobTitle`: `isTechRole`, `isManager`, `isSales`, `isFinance`
    - `department` inferred from flags, else uses provided `department` or defaults to `General`
    - `employeeId` default: `EMP-######` from current time
    - `requiredSystems`: role-based plus baseline: `Slack`, `Google Workspace`, `1Password`
    - `checklistItems` numeric count
    - `packageType`: `Leadership` / `Technical` / `Standard`
- **Inputs/outputs:**
  - Input: webhook payload
  - Output: enriched employee JSON consumed by Notion + Drive listing
- **Failure/edge cases:**
  - `jobTitle` casing/wording differences can misclassify roles (logic is case-sensitive as written because it uses `includes()` without `.toLowerCase()`).
  - Missing `jobTitle` results in all flags false (safe due to optional chaining).
  - `employeeId` derived from timestamp can collide under extreme concurrency.
  - Department mapping prioritization: tech > sales > finance > management > fallback.

---

### Block 1.2 — Tracking & AI Content
**Overview:** Creates a tracking record in Notion and generates a personalized welcome message via an AI Agent using GPT-4.1-mini.

**Nodes involved:**
- Notion: Create Onboarding Tracker
- OpenAI Chat Model
- AI Agent

#### Node: Notion: Create Onboarding Tracker
- **Type / role:** `Notion` — creates a database page for onboarding tracking.
- **Configuration choices:**
  - Resource: `database`
  - Operation: `createPage`
  - (Database ID + property mapping are not shown in the provided JSON; they must be configured in the node UI.)
- **Inputs/outputs:**
  - Input: enriched employee JSON from validation node.
  - Output: Not connected downstream (terminal branch). It writes to Notion only.
- **Failure/edge cases:**
  - Notion auth/token issues; missing integration access to the target database.
  - Property schema mismatch (e.g., Notion expects select/date/email but values are strings).
  - Rate limiting if onboarding events burst.

#### Node: OpenAI Chat Model
- **Type / role:** LangChain `lmChatOpenAi` — provides the chat model to the agent.
- **Configuration:**
  - Model: `gpt-4.1-mini`
  - Credentials: OpenAI API
  - Built-in tools: none enabled
- **Connections:**
  - Output (ai_languageModel) → AI Agent
- **Failure/edge cases:**
  - OpenAI credential missing/expired.
  - Model availability / org permission issues.
  - Cost/latency spikes.

#### Node: AI Agent
- **Type / role:** LangChain `agent` — generates AI content (intended: personalized welcome text).
- **Configuration:**
  - Options are empty in JSON; **no explicit prompt/instructions are included**.
- **Connections:**
  - Output → Merge: Combine AI & PDF Data (main index 0)
  - Receives language model from OpenAI Chat Model.
- **Important note:** As provided, the agent has no system/user prompt or input mapping in parameters. Unless configured in the UI (not represented here) or relying on defaults, it may output nothing useful or behave unpredictably.
- **Failure/edge cases:**
  - If agent output does not contain the expected structure, the Gmail node later references:
    - `{{ $node["AI: Generate Personal Welcome"].json.content[0].text }}`
    - But there is **no node named “AI: Generate Personal Welcome”** in this workflow. This is a likely runtime expression error in the email template unless the node was renamed and the template not updated.

---

### Block 1.3 — Document Processing & Consolidation
**Overview:** Pulls role-based templates from Google Drive, downloads each policy file as binary, merges them into a single PDF, then merges this document stream with AI data.

**Nodes involved:**
- Fetch Role-Based Templates1
- Download Policy Binaries1
- Merge Multiple PDFs into One1
- Merge: Combine AI & PDF Data

#### Node: Fetch Role-Based Templates1
- **Type / role:** `Google Drive` — lists folders.
- **Configuration:**
  - Resource: `folder`
  - Operation: `list`
  - (Folder query/parent folder selection is not visible in the JSON snippet; must be set in node UI. Intended behavior per sticky notes: choose “Technical/Leadership/Standard” folder based on `packageType`.)
- **Connections:**
  - Input: enriched employee JSON
  - Output → Download Policy Binaries1
- **Failure/edge cases:**
  - If the listing returns folders rather than files, the next node (download) may fail because it expects a file ID.
  - Incorrect parent folder selection may return empty results (leading to merge PDF failure).

#### Node: Download Policy Binaries1
- **Type / role:** `Google Drive` — downloads each file as binary.
- **Configuration:**
  - Operation: `download`
  - File ID: `{{ $json.id }}` from previous list output
- **Connections:**
  - Output → Merge Multiple PDFs into One1
- **Failure/edge cases:**
  - Non-PDF files will still download but may break PDF merge downstream.
  - Google Drive permissions/404 for listed items.
  - Large files → timeouts/memory constraints.

#### Node: Merge Multiple PDFs into One1
- **Type / role:** `HTMLCSS to PDF` (pdf manipulation) — merges multiple PDFs.
- **Configuration:**
  - Resource: `pdfManipulation`
  - Operation: `mergePdf`
  - Credential: htmlcsstopdfApi
- **Connections:**
  - Output → Merge: Combine AI & PDF Data (main index 1)
- **Failure/edge cases:**
  - Requires multiple binary PDF inputs; if only one file, behavior depends on the service (usually still returns a PDF, but may error).
  - Invalid PDF binaries or mixed file types.
  - API quota / auth failure.

#### Node: Merge: Combine AI & PDF Data
- **Type / role:** `Merge` — consolidates two streams: AI output and merged PDF output.
- **Configuration:**
  - Mode: `combine`
  - Combination: `mergeByPosition` (pairs item 0 from input 0 with item 0 from input 1, etc.)
- **Connections:**
  - Input 0: AI Agent
  - Input 1: Merge Multiple PDFs into One1
  - Output → Archive to Employee Folder1
- **Failure/edge cases:**
  - Item count mismatch between AI branch and PDF branch can drop or mis-pair data.
  - If AI branch produces 1 item but PDF branch produces N items (or vice versa), merged output may not contain expected fields/binaries.

---

### Block 1.4 — Delivery & Notifications
**Overview:** Archives onboarding artifacts, generates an IT provisioning ticket payload, creates Jira ticket, emails the new hire, and announces in Slack.

**Nodes involved:**
- Archive to Employee Folder1
- Code: Generate IT Provisioning
- Jira: Create IT Provisioning Ticket
- Deliver Welcome Email (Gmail)1
- Slack: Announce New Hire

#### Node: Archive to Employee Folder1
- **Type / role:** `Google Drive` — intended to archive to an employee folder.
- **Configuration (as shown):**
  - Drive: “My Drive”
  - Folder ID: `{{ $json.employeeId }}`
  - **Operation is not present in parameters** in the provided JSON (only driveId/folderId/options). In n8n, Drive nodes usually require an explicit operation (upload/create/copy/move…). This suggests the node may be incomplete or configured in UI fields not present here.
- **Connections:**
  - Input: merged AI+PDF data
  - Output → Code: Generate IT Provisioning AND Deliver Welcome Email (Gmail)1 (two parallel branches)
- **Failure/edge cases:**
  - Using `employeeId` as a Drive `folderId` will fail unless you actually created folders whose IDs equal the employeeId string (unlikely). Typically you would:
    - Search/create folder named employeeId, then use returned Drive folder ID.
  - If the node is meant to upload the merged PDF, missing operation/field mapping will prevent archiving.

#### Node: Code: Generate IT Provisioning
- **Type / role:** `Code` — constructs a detailed IT provisioning “ticket” object.
- **Key logic:**
  - Builds `accessRequests` from `requiredSystems` with metadata (priority based on `isTechRole`).
  - Builds `hardwareNeeds` based on `isTechRole` / `isManager` / default.
  - Adds desk setup items for everyone.
  - Creates `itProvisioningTicket` object with:
    - `ticketId` like `IT-########` from timestamp
    - counts, estimated completion days, assignee, createdAt
- **Connections:**
  - Output → Jira: Create IT Provisioning Ticket
- **Failure/edge cases:**
  - If `requiredSystems` is missing/not an array, `.map()` will throw.
  - Ticket ID generation may collide under concurrency.
  - Hardware logic assumes boolean flags exist.

#### Node: Jira: Create IT Provisioning Ticket
- **Type / role:** `Jira` — creates a task in IT project.
- **Configuration:**
  - Project: `IT`
  - Issue type: `Task`
  - Summary: `New Hire IT Setup - {{ fullName }}`
  - Additional fields:
    - Labels: `onboarding`, `new-hire`, and dynamic department label `{{ department.toLowerCase() }}`
    - Priority: `High` if tech role else `Medium`
- **Connections:**
  - Output → Slack: Announce New Hire
- **Failure/edge cases:**
  - Jira auth errors, insufficient permissions to create issues in project IT.
  - Priority field values must match Jira instance configuration (e.g., “High” may be “Highest”).
  - Labels must be valid; expression fails if `department` is not a string.

#### Node: Deliver Welcome Email (Gmail)1
- **Type / role:** `Gmail` — sends HTML welcome email.
- **Configuration:**
  - To: `{{ email }}`
  - Subject: `🎉 Welcome to the team, {{ firstName }}!`
  - Message: large HTML template including:
    - AI text reference: `{{ $node["AI: Generate Personal Welcome"].json.content[0].text }}`
    - Required systems rendered with `map().join('')`
    - Estimated completion days from `itProvisioningTicket`
- **Connections:**
  - Output → Slack: Announce New Hire
- **Critical issue:** The referenced node name `AI: Generate Personal Welcome` does not exist. The AI node is named `AI Agent`. Unless renamed or a missing node exists, this expression will error.
- **Other edge cases:**
  - If `requiredSystems` is undefined, the `.map()` expression will fail.
  - Email attachments are mentioned in the body, but **no attachment configuration is shown**. If you intend to attach the merged PDF, you must map the binary property from the PDF merge node.

#### Node: Slack: Announce New Hire
- **Type / role:** `Slack` — posts onboarding announcement to a Slack channel.
- **Configuration:**
  - OAuth2 Slack credential.
  - Text includes fullName, jobTitle, department, startDate, and ticket ID.
- **Inputs:**
  - Receives from **both** Jira node and Gmail node. This can cause:
    - **Duplicate Slack posts** (one after Jira, one after Gmail), unless execution paths are controlled.
- **Failure/edge cases:**
  - Slack OAuth token revoked/scopes missing.
  - Channel selection not shown; must be set in node UI.
  - If `itProvisioningTicket.ticketId` not present on the Gmail→Slack path, message will contain blank or error depending on Slack node expression evaluation.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Trigger: New Hire Webhook1 | Webhook | Entry point (HRIS POST) | — | Validate & Enrich Employee Data1 | ### 📥 INTAKE & ENRICHMENT; Webhook receives data, validates fields, classifies role, determines systems access |
| Validate & Enrich Employee Data1 | Code | Validate/enrich payload; role/system derivation | Trigger: New Hire Webhook1 | Notion: Create Onboarding Tracker; Fetch Role-Based Templates1 | ### 📥 INTAKE & ENRICHMENT; Webhook receives data, validates fields, classifies role, determines systems access |
| Notion: Create Onboarding Tracker | Notion | Create onboarding tracker page | Validate & Enrich Employee Data1 | — | ### 🤖 AI & TRACKING; Notion tracker + AI-generated personalized welcome message |
| OpenAI Chat Model | LangChain Chat Model (OpenAI) | LLM powering agent | — | AI Agent | ### 🤖 AI & TRACKING; Notion tracker + AI-generated personalized welcome message |
| AI Agent | LangChain Agent | Generate personalized welcome content | OpenAI Chat Model | Merge: Combine AI & PDF Data | ### 🤖 AI & TRACKING; Notion tracker + AI-generated personalized welcome message |
| Fetch Role-Based Templates1 | Google Drive | List role-based templates/policies | Validate & Enrich Employee Data1 | Download Policy Binaries1 | ### 📄 DOCUMENT PROCESSING; Fetch role-based templates, download policies, merge into single PDF package |
| Download Policy Binaries1 | Google Drive | Download templates as binaries | Fetch Role-Based Templates1 | Merge Multiple PDFs into One1 | ### 📄 DOCUMENT PROCESSING; Fetch role-based templates, download policies, merge into single PDF package |
| Merge Multiple PDFs into One1 | HTMLCSS to PDF (PDF merge) | Merge multiple PDF binaries into one | Download Policy Binaries1 | Merge: Combine AI & PDF Data | ### 🔄 DATA CONSOLIDATION; Merge AI content with PDF package, archive to Drive |
| Merge: Combine AI & PDF Data | Merge | Combine AI output with merged PDF | AI Agent; Merge Multiple PDFs into One1 | Archive to Employee Folder1 | ### 🔄 DATA CONSOLIDATION; Merge AI content with PDF package, archive to Drive |
| Archive to Employee Folder1 | Google Drive | Archive artifacts to employee folder | Merge: Combine AI & PDF Data | Code: Generate IT Provisioning; Deliver Welcome Email (Gmail)1 | ### ✅ DELIVERY & NOTIFICATIONS; IT provisioning, welcome email, Slack announcement |
| Code: Generate IT Provisioning | Code | Build IT access + hardware checklist payload | Archive to Employee Folder1 | Jira: Create IT Provisioning Ticket | ### ✅ DELIVERY & NOTIFICATIONS; IT provisioning, welcome email, Slack announcement |
| Jira: Create IT Provisioning Ticket | Jira | Create IT onboarding task | Code: Generate IT Provisioning | Slack: Announce New Hire | ### ✅ DELIVERY & NOTIFICATIONS; IT provisioning, welcome email, Slack announcement |
| Deliver Welcome Email (Gmail)1 | Gmail | Send HTML welcome email | Archive to Employee Folder1 | Slack: Announce New Hire | ### ✅ DELIVERY & NOTIFICATIONS; IT provisioning, welcome email, Slack announcement |
| Slack: Announce New Hire | Slack | Announce in Slack | Jira: Create IT Provisioning Ticket; Deliver Welcome Email (Gmail)1 | — | ### ✅ DELIVERY & NOTIFICATIONS; IT provisioning, welcome email, Slack announcement |
| Sticky Note: Intake | Sticky Note | Comment block | — | — |  |
| Sticky Note: Intelligence | Sticky Note | Comment block | — | — |  |
| Processing Sticky1 | Sticky Note | Comment block | — | — |  |
| Sticky Note: Merge | Sticky Note | Comment block | — | — |  |
| Sticky Note: Delivery | Sticky Note | Comment block | — | — |  |
| Main Documentation1 | Sticky Note | Global documentation | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Webhook trigger**
   - Add node: **Webhook**
   - Method: **POST**
   - Path: **onboard-employee**
   - Save and copy the **test** and **production** URLs for your HRIS.

2) **Add validation/enrichment logic**
   - Add node: **Code** named “Validate & Enrich Employee Data”
   - Paste the validation/enrichment JavaScript (role flags, department inference, requiredSystems, employeeId).
   - Connect: Webhook → Code

3) **Create Notion onboarding tracker**
   - Add node: **Notion** (Database → Create Page)
   - Create a Notion database with properties at least:
     - Employee Name, Email, Job Title, Department, Start Date, Employee ID, Onboarding Status
   - Map values from the enriched JSON (e.g., Employee Name = `fullName`, Email = `email`, etc.)
   - Connect: Validate Code → Notion

4) **Configure AI generation (LangChain)**
   - Add node: **OpenAI Chat Model** (LangChain)
     - Model: `gpt-4.1-mini`
     - Configure OpenAI credentials
   - Add node: **AI Agent**
     - Set the Chat Model to the OpenAI Chat Model node
     - **Add explicit instructions/prompt**, for example:
       - “Write a short, warm, role-aware welcome paragraph for {{fullName}} joining as {{jobTitle}} in {{department}}. Mention first-day excitement and support.”
     - Ensure the agent outputs a consistent field (e.g., `welcomeText`)
   - Connect: OpenAI Chat Model → AI Agent

5) **List role-based templates in Google Drive**
   - Add node: **Google Drive** “Fetch Role-Based Templates”
     - Operation: list files (or list folder contents) inside a parent folder that contains subfolders `Technical`, `Leadership`, `Standard`.
     - Use `packageType` to choose the correct folder.
   - Connect: Validate Code → Fetch Role-Based Templates

6) **Download each template**
   - Add node: **Google Drive** “Download Policy Binaries”
     - Operation: **Download**
     - File ID: `{{$json.id}}`
   - Connect: Fetch Role-Based Templates → Download Policy Binaries

7) **Merge PDFs into one package**
   - Add node: **HTMLCSS to PDF** (PDF Manipulation → Merge PDF)
   - Configure API credentials for the PDF service.
   - Connect: Download Policy Binaries → Merge PDFs

8) **Merge AI output and PDF output**
   - Add node: **Merge**
     - Mode: **Combine**
     - Combination mode: **Merge by Position**
   - Connect:
     - AI Agent → Merge (Input 1)
     - Merge PDFs → Merge (Input 2)

9) **Archive to Google Drive employee folder**
   - Add node: **Google Drive** “Archive to Employee Folder”
   - Recommended implementation:
     - Search or create folder named `employeeId` under a known parent, then upload merged PDF into it.
   - Connect: Merge → Archive
   - Ensure the merged PDF binary is uploaded (map correct binary property).

10) **Generate IT provisioning payload**
   - Add node: **Code** “Generate IT Provisioning”
   - Paste the provided provisioning JavaScript.
   - Connect: Archive → IT Code

11) **Create Jira issue**
   - Add node: **Jira** “Create IT Provisioning Ticket”
     - Project: `IT`
     - Issue type: `Task`
     - Summary: `New Hire IT Setup - {{$json.fullName}}`
     - Labels: `onboarding`, `new-hire`, `{{$json.department.toLowerCase()}}`
     - Priority: `{{$json.isTechRole ? 'High' : 'Medium'}}` (ensure it matches your Jira priority names)
   - Connect: IT Code → Jira

12) **Send welcome email via Gmail**
   - Add node: **Gmail** “Deliver Welcome Email”
     - To: `{{$json.email}}`
     - Subject: `🎉 Welcome to the team, {{$json.firstName}}!`
     - Body: HTML template
     - Fix AI reference: replace `AI: Generate Personal Welcome` with your actual AI node name and output field (e.g., `{{$node["AI Agent"].json.welcomeText}}`).
     - Attach the merged PDF by mapping the binary property from the PDF merge/archive step.
   - Connect: Archive → Gmail

13) **Post Slack announcement**
   - Add node: **Slack** “Announce New Hire”
     - Choose target channel (e.g., `#new-hires`)
     - Message text similar to provided template.
   - To avoid duplicates, choose **one** trigger path:
     - Either connect **Jira → Slack** (announce after ticket created), **or**
     - Connect **Gmail → Slack** (announce after email sent), **or**
     - Use an additional **Merge (wait)** to join both before a single Slack post.
   - Configure Slack OAuth2 credentials.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “This intelligent onboarding workflow automates the entire new hire process…” | From “Main Documentation1” sticky note (overall description). |
| Setup steps include: Webhook HRIS → Notion DB → Drive folders (“Technical/Leadership/Standard”) → AI API key → PDF merge API → Jira → Gmail → Slack channel | From “Main Documentation1” sticky note (implementation checklist). |
| Key intelligence: role detection, system access requirements, hardware provisioning lists, timelines, AI welcome message | From “Main Documentation1” sticky note (design intent). |

