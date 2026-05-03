Verify webinar signups and send personalized GPT-4 welcome kits via Gmail and Google Sheets

https://n8nworkflows.xyz/workflows/verify-webinar-signups-and-send-personalized-gpt-4-welcome-kits-via-gmail-and-google-sheets-11730


# Verify webinar signups and send personalized GPT-4 welcome kits via Gmail and Google Sheets

Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow receives webinar signup submissions via a webhook, verifies the attendee’s email, stores verified signups in Google Sheets, generates a personalized GPT‑4 welcome message, creates a “welcome kit” document in Google Drive, downloads it as a PDF, and emails it via Gmail. It also notifies organizers in Slack. If the attendee requested extra materials, it generates additional GPT‑4 content, creates a second document, downloads it, and sends a follow‑up email with that PDF attached.

**Typical use cases:**
- Webinar signup intake with anti-fraud / typo protection (email validation)
- Automatic attendee CRM logging (Google Sheets)
- High-touch onboarding at scale (AI-generated personalization)
- Conditional post-signup enrichment (optional learning materials)

### Logical blocks
**1.1 Signup Intake & Email Validation**  
Webhook receives POST signup data → VerifiEmail validates the email → IF branches valid vs invalid.

**1.2 Invalid Signup Handling**  
If invalid: send Slack alert → stop workflow with an error.

**1.3 Verified Attendee Storage**  
If valid: append attendee record to Google Sheets.

**1.4 Personalized Welcome Content**  
Generate GPT‑4 welcome message → create a text-based Google Drive document → download (intended as PDF) for emailing.

**1.5 Welcome Email Delivery & Organizer Notification**  
Send welcome email via Gmail with attachment → Slack notify organizers.

**1.6 Optional Extra Materials Flow**  
If extra materials requested: generate GPT‑4 materials → create document → download → send follow-up email with attachment.

---

## 2. Block-by-Block Analysis

### 2.1 Signup Intake & Email Validation

**Overview:**  
Captures signup payload from an external form/system and validates the provided email using VerifiEmail. Routes execution based on validity.

**Nodes involved:**
- Webhook
- VerifiEmail - Verify Email
- Check Email Valid

#### Node: Webhook
- **Type / role:** `n8n-nodes-base.webhook` — entry point receiving HTTP requests.
- **Configuration choices:**
  - Method: `POST`
  - Path: `webinar-signup` (endpoint will be `/webhook/webinar-signup` or `/webhook-test/webinar-signup` depending on environment)
- **Key data used:** Expects `body` fields like:
  - `body.name`, `body.email`, `body.company`, `body.session_preference`, `body.extra_materials`
- **Outputs / connections:**
  - Outputs to **VerifiEmail - Verify Email**
- **Version notes:** TypeVersion `2.1` (Webhook node UI differs slightly across n8n versions; path/method concepts stable).
- **Edge cases / failures:**
  - Missing/invalid JSON payload (no `body.email`, etc.) will break downstream expressions.
  - If your form posts as `application/x-www-form-urlencoded`, body parsing may differ.
  - Public webhook exposure: consider rate-limiting / auth if needed.

#### Node: VerifiEmail - Verify Email
- **Type / role:** `n8n-nodes-verifiemail.verifiEmail` — external email verification service call.
- **Configuration choices:**
  - Email input: `{{ $json.body.email }}`
- **Credentials:** `verifiEmailApi` (API key required)
- **Outputs / connections:**
  - Outputs to **Check Email Valid**
- **Version notes:** TypeVersion `1`.
- **Edge cases / failures:**
  - API key invalid/expired → auth error.
  - Service timeouts/rate limits.
  - If webhook payload lacks `body.email`, this node will send an empty value or fail.
  - The workflow assumes the response includes a boolean field `valid` (used next).

#### Node: Check Email Valid
- **Type / role:** `n8n-nodes-base.if` — branching based on validation result.
- **Configuration choices:**
  - Condition: `{{ $json.valid }} equals true` (boolean strict validation)
- **Outputs / connections:**
  - **True branch** → Store Attendee Data
  - **False branch** → Alert - Invalid Email
- **Version notes:** TypeVersion `2.3` (IF node “version 3” condition schema used).
- **Edge cases / failures:**
  - If VerifiEmail returns `valid` as a string (`"true"`) instead of boolean, strict comparison fails (routes to invalid).
  - If `valid` is missing, condition evaluates false.

**Sticky note coverage (applies to nodes in this block):**  
“## Signup Intake & Email Validation … Captures webinar signup data and verifies email authenticity…”

---

### 2.2 Invalid Signup Handling

**Overview:**  
Alerts organizers in Slack about invalid signups and halts execution to prevent polluted data and unnecessary processing.

**Nodes involved:**
- Alert - Invalid Email
- Stop and Error

#### Node: Alert - Invalid Email
- **Type / role:** `n8n-nodes-base.slack` — send a message to a Slack channel.
- **Configuration choices:**
  - Sends a formatted message including name/email from the **Webhook** node and a timestamp.
  - Channel selection via `channelId` (list mode).
- **Key expressions:**
  - Name: `{{ $('Webhook').item.json.body.name }}`
  - Email: `{{ $('Webhook').item.json.body.email }}`
  - Time: `{{$now.format("yyyy-MM-dd HH:mm:ss")}}`
- **Outputs / connections:**
  - Outputs to **Stop and Error**
- **Credentials:** `slackApi` (bot token)
- **Version notes:** TypeVersion `2.4`
- **Edge cases / failures:**
  - Slack auth errors (invalid token/scopes).
  - Channel ID mismatch or bot not in the channel.
  - If webhook body fields are missing, message may contain blanks or expression errors.

#### Node: Stop and Error
- **Type / role:** `n8n-nodes-base.stopAndError` — terminates the workflow with a controlled error.
- **Configuration choices:** Error message: `"Invalid Email"`
- **Outputs / connections:** Terminal node (no outputs).
- **Version notes:** TypeVersion `1`
- **Edge cases / failures:**
  - This intentionally marks executions as failed; if you rely on “success-only” metrics, account for this behavior.

**Sticky note coverage:**  
“## Invalid Signup Handling … Stops fake or invalid registrations and alerts organizers…”

---

### 2.3 Verified Attendee Storage

**Overview:**  
Appends verified attendee information into a Google Sheet to serve as the system of record.

**Nodes involved:**
- Store Attendee Data

#### Node: Store Attendee Data
- **Type / role:** `n8n-nodes-base.googleSheets` — append row to Google Sheets.
- **Configuration choices:**
  - Operation: `append`
  - Document: `YOUR_GOOGLE_SHEETS_DOCUMENT_ID`
  - Sheet: `gid=0` (selected sheet tab)
  - Mapping mode: “Define below” using schema fields
- **Mapped columns / expressions:**
  - Name: `{{ $('Webhook').item.json.body.name }}`
  - Email: `{{ $('Webhook').item.json.body.email }}`
  - Company: `{{ $('Webhook').item.json.body.company }}`
  - Session: `{{ $('Webhook').item.json.body.session_preference }}`
  - Verified: `"Yes"`
  - Timestamp: `{{$now.format("yyyy-MM-dd HH:mm:ss")}}`
  - Extra Materials: `{{ $('Webhook').item.json.body.extra_materials }}`
- **Outputs / connections:**
  - Outputs to **Generate Welcome Message**
- **Credentials:** `googleSheetsOAuth2Api`
- **Version notes:** TypeVersion `4.7` (recent Google Sheets node UI/behavior).
- **Edge cases / failures:**
  - OAuth token expired / missing scopes (Sheets + Drive access).
  - Sheet column headers must match schema; if renamed, append may misplace data.
  - `extra_materials` might be boolean, string, or missing; consider normalizing.

**Sticky note coverage:**  
“## Verified Attendee Storage … Stores verified attendee details securely in Google Sheets…”

---

### 2.4 Personalized Welcome Content

**Overview:**  
Uses GPT‑4 to craft a personalized welcome message, then builds a “welcome kit” document in Google Drive using both static content and the AI output.

**Nodes involved:**
- Generate Welcome Message
- Create Welcome Doc
- Convert to PDF

#### Node: Generate Welcome Message
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi` — LLM call via the LangChain-based OpenAI node.
- **Configuration choices:**
  - Model: `gpt-4`
  - Max tokens: `300`
  - Temperature: `0.7`
  - Prompt instructs: 3–4 paragraphs, professional-friendly, no subject line, uses name/company/session preference.
- **Key expressions / variables:**
  - Injects: `$('Webhook').item.json.body.name`, `.company`, `.session_preference`
- **Outputs / connections:**
  - Outputs to **Create Welcome Doc**
- **Credentials:** `openAiApi`
- **Version notes:** TypeVersion `2.1` (LangChain OpenAI node; output structure can differ from classic OpenAI node).
- **Edge cases / failures:**
  - Model access (GPT‑4 may require account eligibility).
  - Token limits: long inputs could truncate.
  - Output parsing: downstream nodes reference a nested path; if node output format changes, expressions break.

#### Node: Create Welcome Doc
- **Type / role:** `n8n-nodes-base.googleDrive` — creates a file from text in Google Drive.
- **Configuration choices:**
  - Operation: `createFromText`
  - Name pattern: `Webinar_Welcome_Kit_{Name}_{YYYY-MM-DD}` (spaces replaced once via `.replace(" ", "_")`)
  - Folder: `root` (configured as “YOUR_GOOGLE_DRIVE_FOLDER_NAME” but value shown is `root`)
  - Content includes attendee details, schedules, tips, plus AI-generated message inserted.
- **Key expressions:**
  - File name: `{{ $('Webhook').item.json.body.name.replace(" ", "_")}}` (note: replaces only first space)
  - AI insertion: `{{$node["Generate Welcome Message"].json["output"][0]["content"][0]["text"]}}`
- **Outputs / connections:**
  - Outputs to **Convert to PDF**
- **Credentials:** `googleDriveOAuth2Api`
- **Version notes:** TypeVersion `3`
- **Edge cases / failures:**
  - Folder permission issues.
  - If AI output path differs, the content expression fails.
  - Name sanitization: multiple spaces or special characters may create awkward filenames.

#### Node: Convert to PDF
- **Type / role:** `n8n-nodes-base.googleDrive` — downloads the created file.
- **Configuration choices:**
  - Operation: `download`
  - File ID: `{{ $json.id }}` (expects previous node output includes `id`)
- **Important note:** Despite being named “Convert to PDF”, this node is configured as **download**, not explicit PDF export. In Google Drive, downloading a Google Doc as PDF usually requires an “export”/conversion option. As configured, it may download the original format (or a default binary) depending on node behavior and file type.
- **Outputs / connections:**
  - Outputs to **Send Welcome Email**
- **Credentials:** `googleDriveOAuth2Api`
- **Version notes:** TypeVersion `3`
- **Edge cases / failures:**
  - If it downloads as a non-PDF format, Gmail attachment may not be the intended PDF.
  - Binary property naming must match what Gmail expects (see next block).

**Sticky note coverage:**  
“## Personalized Welcome Content … Uses AI to generate a custom welcome message and creates a downloadable welcome kit.”

---

### 2.5 Welcome Email Delivery & Organizer Notification

**Overview:**  
Emails the attendee the personalized welcome message and attaches the downloaded file; then posts a Slack confirmation to organizers.

**Nodes involved:**
- Send Welcome Email
- Notify Organizers

#### Node: Send Welcome Email
- **Type / role:** `n8n-nodes-base.gmail` — sends an email via Gmail.
- **Configuration choices:**
  - To: `{{ $('Webhook').item.json.body.email }}`
  - Subject: `Welcome to [Webinar Name] - Your Session Details Inside!`
  - Message: HTML body built from AI text with `\n` replaced by `<br>`
  - Attachments: uses UI field `attachmentsBinary` with an empty object placeholder
- **Key expressions:**
  - Body: `{{$node["Generate Welcome Message"].json["output"][0]["content"][0]["text"].replace(/\n/g, "<br>")}}`
- **Inputs / connections:**
  - Receives main data from **Convert to PDF**
  - Sends to **Notify Organizers**
- **Credentials:** `gmailOAuth2`
- **Version notes:** TypeVersion `2.2`
- **Edge cases / failures (important):**
  - **Attachment mapping is incomplete as shown.** Gmail node requires selecting the binary property name (e.g., `data`) produced by the Drive download node. With `{}` as placeholder, the email may send **without attachment**.
  - If Drive node outputs binary under a different key, configure `attachmentsBinary` accordingly.
  - Gmail OAuth scopes and “From” address constraints.
  - HTML rendering: ensure AI output doesn’t contain unsafe HTML (it’s plain text here).

#### Node: Notify Organizers
- **Type / role:** `n8n-nodes-base.slack` — notify Slack channel of verified attendee.
- **Configuration choices:**
  - Posts attendee details + whether extra materials requested.
  - Extra Materials display: `{{ $('Webhook').item.json.body.extra_materials ? "Yes" : "No"}}`
- **Outputs / connections:**
  - Outputs to **Check Extra Materials Requested**
- **Credentials:** `slackApi`
- **Version notes:** TypeVersion `2.4`
- **Edge cases / failures:**
  - Slack auth/scopes/channel membership issues.
  - If `extra_materials` is not boolean but `"true"`/`"yes"`, the ternary may behave unexpectedly.

**Sticky note coverage:**  
“## Welcome Email Delivery & Notification … Sends the welcome email with the PDF attached and notifies organizers in Slack.”

---

### 2.6 Optional Extra Materials Flow

**Overview:**  
Conditionally generates additional AI-powered materials for attendees who requested them, creates a second document, downloads it, and emails it.

**Nodes involved:**
- Check Extra Materials Requested
- Generate Extra Materials
- Create Extra Materials PDF
- Convert to PDF (Extra Materials)
- Send Follow-up Email

#### Node: Check Extra Materials Requested
- **Type / role:** `n8n-nodes-base.if` — conditional branch.
- **Configuration choices:**
  - Condition: `{{ $('Webhook').item.json.body.extra_materials }} equals true` (strict boolean)
- **Outputs / connections:**
  - **True branch** → Generate Extra Materials
  - No explicit false branch destination (flow ends if not requested)
- **Version notes:** TypeVersion `2.3`
- **Edge cases / failures:**
  - If the form sends `"true"` as a string, strict boolean comparison fails and the block won’t run.

#### Node: Generate Extra Materials
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi` — GPT‑4 content generation.
- **Configuration choices:**
  - Model: `gpt-4`
  - Prompt requests:
    - Pre-webinar reading list (5 articles)
    - Key terminology
    - Advanced questions
    - Tailored to session preference
- **Outputs / connections:**
  - Outputs to **Create Extra Materials PDF**
- **Credentials:** `openAiApi`
- **Version notes:** TypeVersion `2.1`
- **Edge cases / failures:**
  - Same as other LLM node: model access, token limits, output structure variability.

#### Node: Create Extra Materials PDF
- **Type / role:** `n8n-nodes-base.googleDrive` — create file from text.
- **Configuration choices:**
  - Operation: `createFromText`
  - Name: `Extra_Materials_{attendee name}`
  - Content: `{{ $json.output[0].content[0].text }}`
  - Folder: `root`
- **Outputs / connections:**
  - Outputs to **Convert to PDF (Extra Materials)**
- **Credentials:** `googleDriveOAuth2Api`
- **Version notes:** TypeVersion `3`
- **Edge cases / failures:**
  - Same Drive permissions/content-expression issues.

#### Node: Convert to PDF (Extra Materials)
- **Type / role:** `n8n-nodes-base.googleDrive` — downloads created file.
- **Configuration choices:**
  - Operation: `download`
  - File ID: `{{ $json.id }}`
- **Outputs / connections:**
  - Outputs to **Send Follow-up Email**
- **Credentials:** `googleDriveOAuth2Api`
- **Version notes:** TypeVersion `3`
- **Edge cases / failures:**
  - Same “download vs export to PDF” concern as earlier.

#### Node: Send Follow-up Email
- **Type / role:** `n8n-nodes-base.gmail` — send follow-up email with extra materials.
- **Configuration choices:**
  - Email type: `text`
  - To: `{{ $('Webhook').item.json.body.email }}`
  - Subject: `Extra Materials for [Webinar Name]`
  - Body references sheet data for name: `{{ $('Store Attendee Data').item.json.Name }}`
  - Attachments: `attachmentsBinary` placeholder is empty object `{}` again
- **Inputs / connections:**
  - Receives from **Convert to PDF (Extra Materials)**
- **Credentials:** `gmailOAuth2`
- **Version notes:** TypeVersion `2.2`
- **Edge cases / failures:**
  - Attachment not configured (likely sends without the PDF).
  - If Google Sheets append returns a different structure (or multiple items), `$('Store Attendee Data').item.json.Name` could be missing.

**Sticky note coverage:**  
- “## Optional Extra Materials Flow … Checks if extra materials were requested…”
- “## Extra Materials Creation & Follow-up … Creates a PDF … and sends a follow-up email…”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook | n8n-nodes-base.webhook | Receive webinar signup payload | — | VerifiEmail - Verify Email | ## Signup Intake & Email Validation  Captures webinar signup data and verifies email authenticity before allowing the user into the system. |
| VerifiEmail - Verify Email | n8n-nodes-verifiemail.verifiEmail | Validate email via VerifiEmail API | Webhook | Check Email Valid | ## Signup Intake & Email Validation  Captures webinar signup data and verifies email authenticity before allowing the user into the system. |
| Check Email Valid | n8n-nodes-base.if | Branch valid vs invalid email | VerifiEmail - Verify Email | Store Attendee Data; Alert - Invalid Email | ## Signup Intake & Email Validation  Captures webinar signup data and verifies email authenticity before allowing the user into the system. |
| Alert - Invalid Email | n8n-nodes-base.slack | Alert organizers about invalid signup | Check Email Valid (false) | Stop and Error | ## Invalid Signup Handling  Stops fake or invalid registrations and alerts organizers instantly to protect data quality. |
| Stop and Error | n8n-nodes-base.stopAndError | Stop execution for invalid signup | Alert - Invalid Email | — | ## Invalid Signup Handling  Stops fake or invalid registrations and alerts organizers instantly to protect data quality. |
| Store Attendee Data | n8n-nodes-base.googleSheets | Append verified attendee to sheet | Check Email Valid (true) | Generate Welcome Message | ## Verified Attendee Storage  Stores verified attendee details securely in Google Sheets for tracking and reporting. |
| Generate Welcome Message | @n8n/n8n-nodes-langchain.openAi | Create personalized welcome message | Store Attendee Data | Create Welcome Doc | ## Personalized Welcome Content  Uses AI to generate a custom welcome message and creates a downloadable welcome kit. |
| Create Welcome Doc | n8n-nodes-base.googleDrive | Create welcome kit file from text | Generate Welcome Message | Convert to PDF | ## Personalized Welcome Content  Uses AI to generate a custom welcome message and creates a downloadable welcome kit. |
| Convert to PDF | n8n-nodes-base.googleDrive | Download created file (intended as PDF) | Create Welcome Doc | Send Welcome Email | ## Personalized Welcome Content  Uses AI to generate a custom welcome message and creates a downloadable welcome kit. |
| Send Welcome Email | n8n-nodes-base.gmail | Email welcome + attachment | Convert to PDF | Notify Organizers | ## Welcome Email Delivery & Notification  Sends the welcome email with the PDF attached and notifies organizers in Slack. |
| Notify Organizers | n8n-nodes-base.slack | Slack notification for verified attendee | Send Welcome Email | Check Extra Materials Requested | ## Welcome Email Delivery & Notification  Sends the welcome email with the PDF attached and notifies organizers in Slack. |
| Check Extra Materials Requested | n8n-nodes-base.if | Conditional branch for extra materials | Notify Organizers | Generate Extra Materials | ## Optional Extra Materials Flow  Checks if extra materials were requested and generates advanced session-specific content if needed. |
| Generate Extra Materials | @n8n/n8n-nodes-langchain.openAi | Generate additional tailored resources | Check Extra Materials Requested (true) | Create Extra Materials PDF | ## Extra Materials Creation & Follow-up  Creates a PDF of extra learning materials and sends a follow-up email to the attendee. |
| Create Extra Materials PDF | n8n-nodes-base.googleDrive | Create extra materials file from text | Generate Extra Materials | Convert to PDF (Extra Materials) | ## Extra Materials Creation & Follow-up  Creates a PDF of extra learning materials and sends a follow-up email to the attendee. |
| Convert to PDF (Extra Materials) | n8n-nodes-base.googleDrive | Download extra materials file | Create Extra Materials PDF | Send Follow-up Email | ## Extra Materials Creation & Follow-up  Creates a PDF of extra learning materials and sends a follow-up email to the attendee. |
| Send Follow-up Email | n8n-nodes-base.gmail | Send follow-up email + attachment | Convert to PDF (Extra Materials) | — | ## Extra Materials Creation & Follow-up  Creates a PDF of extra learning materials and sends a follow-up email to the attendee. |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation (canvas note) | — | — | ## **Automated Webinar Signup Verification & Personalized Welcome Kit** This workflow automates the complete lifecycle of a webinar signup — from email verification to personalized onboarding and follow-up delivery... (full note content) |
| Sticky Note1 | n8n-nodes-base.stickyNote | Documentation (canvas note) | — | — | ## Signup Intake & Email Validation  Captures webinar signup data and verifies email authenticity before allowing the user into the system. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Documentation (canvas note) | — | — | ## Invalid Signup Handling  Stops fake or invalid registrations and alerts organizers instantly to protect data quality. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Documentation (canvas note) | — | — | ## Verified Attendee Storage  Stores verified attendee details securely in Google Sheets for tracking and reporting. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Documentation (canvas note) | — | — | ## Personalized Welcome Content  Uses AI to generate a custom welcome message and creates a downloadable welcome kit. |
| Sticky Note5 | n8n-nodes-base.stickyNote | Documentation (canvas note) | — | — | ## Welcome Email Delivery & Notification  Sends the welcome email with the PDF attached and notifies organizers in Slack. |
| Sticky Note6 | n8n-nodes-base.stickyNote | Documentation (canvas note) | — | — | ## Extra Materials Creation & Follow-up  Creates a PDF of extra learning materials and sends a follow-up email to the attendee. |
| Sticky Note7 | n8n-nodes-base.stickyNote | Documentation (canvas note) | — | — | ## Optional Extra Materials Flow  Checks if extra materials were requested and generates advanced session-specific content if needed. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Name it: **Automated Webinar Signup Verification & Personalized Welcome Kit**
- Keep execution order default (`v1`)

2) **Add Webhook node**
- Node type: **Webhook**
- HTTP Method: **POST**
- Path: **webinar-signup**
- Save; copy the test/production webhook URL for your signup form.

3) **Add VerifiEmail verification node**
- Node type: **VerifiEmail**
- Operation: verify email (as provided by the node)
- Email field expression: `{{ $json.body.email }}`
- Set **VerifiEmail API credentials** (API key).

4) **Add IF node for email validity**
- Node type: **IF**
- Condition (boolean): Left: `{{ $json.valid }}` → equals → Right: `true`
- Connect:
  - Webhook → VerifiEmail
  - VerifiEmail → IF

5) **Invalid path: Slack alert + stop**
- Add **Slack** node (send message)
  - Select channel (set Slack OAuth/bot token credential; pick channel ID)
  - Message text includes:
    - `{{ $('Webhook').item.json.body.name }}`
    - `{{ $('Webhook').item.json.body.email }}`
    - `{{$now.format("yyyy-MM-dd HH:mm:ss")}}`
- Add **Stop and Error** node with message: `Invalid Email`
- Connect:
  - IF (false) → Slack alert → Stop and Error

6) **Valid path: Google Sheets append**
- Add **Google Sheets** node
  - Operation: **Append**
  - Choose Document (Spreadsheet) and Sheet (tab)
  - Map columns:
    - Name = `{{ $('Webhook').item.json.body.name }}`
    - Email = `{{ $('Webhook').item.json.body.email }}`
    - Company = `{{ $('Webhook').item.json.body.company }}`
    - Session = `{{ $('Webhook').item.json.body.session_preference }}`
    - Verified = `Yes`
    - Timestamp = `{{$now.format("yyyy-MM-dd HH:mm:ss")}}`
    - Extra Materials = `{{ $('Webhook').item.json.body.extra_materials }}`
  - Configure **Google Sheets OAuth2** credential.
- Connect: IF (true) → Google Sheets

7) **Generate welcome message (GPT‑4)**
- Add node: **OpenAI (LangChain)** (`@n8n/n8n-nodes-langchain.openAi`)
- Model: **gpt‑4**
- Temperature: `0.7`
- Max tokens: `300`
- Prompt: instructs 3–4 paragraphs, uses attendee details, message body only.
- Configure **OpenAI API credential**.
- Connect: Google Sheets → OpenAI node

8) **Create welcome kit file in Google Drive**
- Add node: **Google Drive**
- Operation: **Create from text**
- Name expression: `Webinar_Welcome_Kit_{{ $('Webhook').item.json.body.name.replace(" ", "_")}}_{{$now.format("yyyy-MM-dd")}}`
- Folder: choose target folder (or root)
- Content: include static schedule/tips + insert AI message from previous node output.
- Configure **Google Drive OAuth2** credential.
- Connect: OpenAI welcome → Drive create-from-text

9) **Download file (intended PDF)**
- Add node: **Google Drive**
- Operation: **Download**
- File ID: `{{ $json.id }}`
- Connect: Drive create → Drive download
- **If you truly need a PDF:** ensure the Drive node supports exporting Google Docs to PDF (or add a dedicated export/conversion step). As built, it only “downloads”.

10) **Send welcome email via Gmail**
- Add node: **Gmail**
- To: `{{ $('Webhook').item.json.body.email }}`
- Subject: `Welcome to [Webinar Name] - Your Session Details Inside!`
- Body (HTML): use the OpenAI output and replace newlines with `<br>`.
- Attachments:
  - Select the binary property produced by the Drive download node (commonly something like `data`).
  - In the Gmail node, set **Attachments → Binary property** to that key.
- Configure **Gmail OAuth2** credential.
- Connect: Drive download → Gmail send

11) **Notify organizers in Slack**
- Add **Slack** node to post success message to the channel.
- Include attendee details + computed extra-materials flag.
- Connect: Gmail welcome → Slack notify

12) **Optional extra materials IF**
- Add **IF** node:
  - Condition: `{{ $('Webhook').item.json.body.extra_materials }}` equals `true`
- Connect: Slack notify → IF extra materials

13) **Generate extra materials (GPT‑4)**
- Add **OpenAI (LangChain)** node (GPT‑4) with prompt for reading list, terminology, advanced questions tailored to session.
- Connect: IF (true) → OpenAI extra materials

14) **Create extra materials file in Drive + download**
- Add **Google Drive** create-from-text (name `Extra_Materials_{name}`, content from AI output)
- Add **Google Drive** download node referencing created file id
- Connect: OpenAI extra → Drive create → Drive download

15) **Send follow-up email with extra materials**
- Add **Gmail** node:
  - Email type: text
  - To: `{{ $('Webhook').item.json.body.email }}`
  - Subject: `Extra Materials for [Webinar Name]`
  - Body can reference `{{ $('Store Attendee Data').item.json.Name }}` (or directly webhook name)
  - Attachments: set correct binary property from the extra-materials Drive download node
- Connect: Drive download (extra) → Gmail follow-up

16) **Activate and test**
- Send a sample POST payload:
  - Ensure `email` is valid to test the full happy path.
  - Test invalid email to confirm Slack alert + Stop and Error.
  - Test `extra_materials=true` to validate optional branch.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| **Automated Webinar Signup Verification & Personalized Welcome Kit** — end-to-end flow description, setup steps, and logic outline included in the workflow canvas note. | Sticky note content (overall workflow documentation) |
| Key setup tasks mentioned: configure webhook endpoint, add VerifiEmail/OpenAI/Google/Gmail/Slack credentials, replace spreadsheet + Slack channel IDs, activate and test. | Sticky note content (setup guidance) |
| Important implementation caution: both Gmail nodes show attachment UI placeholders without selecting a binary property; to actually attach PDFs/files, map the Drive download binary field in Gmail “attachmentsBinary”. | Integration reliability note based on node configuration |
| PDF conversion caution: nodes labeled “Convert to PDF” are configured as Google Drive **download**; if you require true PDF export, add/adjust a Drive export/conversion step. | Integration behavior note (Drive export vs download) |