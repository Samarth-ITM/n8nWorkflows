Generate verified job offer letters with OpenAI, Gmail and Slack

https://n8nworkflows.xyz/workflows/generate-verified-job-offer-letters-with-openai--gmail-and-slack-11732


# Generate verified job offer letters with OpenAI, Gmail and Slack

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Generate verified job offer letters with OpenAI, Gmail and Slack  
**Workflow name (internal):** Verified Job Offer Letter Generator  
**Purpose:** Automates end-to-end creation and delivery of a “verified” job offer letter: it receives candidate/job data via webhook, validates the candidate email, generates offer letter body text using OpenAI (GPT‑4), renders a branded HTML letter with a digital signature + document ID, converts it to PDF, emails it via Gmail, and notifies HR via Slack.

### 1.1 Logical Blocks
1. **Input Reception & Email Verification**  
   Webhook intake → VerifiEmail validation → decision gate.
2. **Data Preparation & AI Letter Generation**  
   Normalize fields (salary formatting, dates, HR identity) → GPT‑4 creates letter body text.
3. **Document Formatting & Conversion**  
   Merge AI text + candidate/job metadata into a branded HTML letter, generate SVG signature + Document ID → convert HTML to PDF.
4. **Delivery & HR Notifications**  
   Send email to candidate with PDF attachment → post Slack confirmation to HR channel.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception & Email Verification
**Overview:** Accepts candidate/job details via HTTP POST and verifies the candidate email deliverability before any AI generation or delivery occurs.

**Nodes involved:** Webhook, Email Verification, Check Email Validity

#### Node: Webhook
- **Type / role:** `n8n-nodes-base.webhook` — workflow trigger (HTTP endpoint).
- **Config (interpreted):**
  - **Method:** POST
  - **Path:** `job-offer-generator` (final URL depends on n8n base URL)
- **Key data expected:** `body.candidateName`, `body.candidateEmail`, `body.position`, `body.department`, `body.salary`, `body.joiningDate`, `body.reportingTo`, `body.workLocation`
- **Outputs:** Passes incoming request JSON to **Email Verification**.
- **Potential failures / edge cases:**
  - Missing fields (later nodes reference them via expressions; missing data can cause blank values or formatting errors).
  - Wrong content-type or structure (e.g., `body` absent).
  - Security: no authentication/verification configured (anyone could POST unless protected externally).

#### Node: Email Verification
- **Type / role:** `n8n-nodes-verifiemail.verifiEmail` — validates email address via VerifiEmail API.
- **Config:**
  - Email input: `{{$json.body.candidateEmail}}` (from Webhook payload)
  - Requires **VerifiEmail API credentials**
- **Outputs:** Provides verification result, including a boolean field used later: `valid`.
- **Connections:** Input from **Webhook** → output to **Check Email Validity**.
- **Potential failures / edge cases:**
  - Credential/auth errors (invalid API key).
  - API rate limits/timeouts.
  - Unexpected response schema (if `valid` missing, the IF node may not behave as intended).

#### Node: Check Email Validity
- **Type / role:** `n8n-nodes-base.if` — gate to proceed only if email is valid.
- **Config:**
  - Condition checks boolean: `{{$json.valid}}` is **true**.
- **Connections:** Input from **Email Verification** → **true** path to **Prepare Offer Data**.
- **Important behavior:** The workflow has **no “false” branch handling**. Invalid emails effectively stop processing silently.
- **Potential failures / edge cases:**
  - If `valid` is not strictly boolean, strict validation may evaluate unexpectedly.
  - Lack of rejection response: webhook caller receives no explicit “invalid email” handling unless n8n’s default behavior is relied upon.

---

### Block 2 — Data Preparation & AI Letter Generation
**Overview:** Maps webhook fields into a clean internal schema, formats salary/date, injects company/HR defaults, and generates the offer letter body via GPT‑4.

**Nodes involved:** Prepare Offer Data, Generate Offer Letter Content

#### Node: Prepare Offer Data
- **Type / role:** `n8n-nodes-base.set` — field mapping and normalization.
- **Config highlights:**
  - Pulls source data via cross-node expressions from **Webhook**:
    - `candidateName`, `candidateEmail`, `position`, `department`, `salary`, `joiningDate`, `reportingTo`, `workLocation`
  - Derived fields:
    - `formattedSalary`: `"$" + Number(salary).toLocaleString()`
    - `offerDate`: `$now.format('MMMM DD, YYYY')`
  - Static defaults:
    - `companyName`: `TechCorp Inc. `
    - `hrName`: `Sarah Johnson`
    - `hrTitle`: `VP of Human Resources`
- **Connections:** Input from **Check Email Validity (true)** → output to **Generate Offer Letter Content**.
- **Potential failures / edge cases:**
  - `salary` not numeric: `Number(...)` becomes `NaN`, resulting in `$NaN`.
  - `joiningDate` format not validated; used as plain string later.
  - Company identity hardcoded; must be updated for real usage.

#### Node: Generate Offer Letter Content
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi` — OpenAI chat completion via n8n’s LangChain-based node.
- **Config highlights:**
  - **Model:** `gpt-4`
  - **Temperature:** `0.7`
  - **Messages:**
    - System prompt defines HR writing style, constraints (400–600 words, no signature/date/address blocks, etc.).
    - User prompt injects dynamic fields from `$json` (coming from Prepare Offer Data) such as name, email, role, department, salary, start date, etc.
- **Connections:** Input from **Prepare Offer Data** → output to **Build HTML with SVG Signature**.
- **Version-specific considerations:**
  - Output schema differs across OpenAI nodes and versions; downstream code assumes a particular structure (see next block).
- **Potential failures / edge cases:**
  - OpenAI credential/auth issues, rate limits, or timeouts.
  - Model availability (GPT‑4 access not enabled).
  - Output format mismatch with downstream extraction logic (most critical risk): the Code node expects `openAiData.output[0].content[0].text`.

---

### Block 3 — Document Formatting & Conversion
**Overview:** Converts AI text into a fully branded HTML document, adds SVG “signature” and a unique Document ID, then renders it into a PDF.

**Nodes involved:** Build HTML with SVG Signature, Convert to PDF

#### Node: Build HTML with SVG Signature
- **Type / role:** `n8n-nodes-base.code` — custom JavaScript to assemble HTML + metadata.
- **Config highlights (logic):**
  - Reads candidate/job/company data from: `$('Prepare Offer Data').item.json`
  - Reads OpenAI output from: `$input.item.json`
  - Extracts AI text into `offerLetterText` with try/catch:
    - Expected path: `openAiData.output[0].content[0].text`
    - On failure: sets `offerLetterText = 'Error extracting text: ...'`
  - Generates a unique ID:
    - Format: `OFFER-${timestamp}-${RANDOM}`
  - Generates SVG signature:
    - Uses HR name (`candidateData.hrName`) rendered in cursive-like font.
  - Builds large branded HTML with:
    - Letterhead + gradient styling
    - Recipient block
    - AI letter content inserted into `<div class="content">`
    - Position details table
    - Signature section + “Digitally Signed & Verified Document” block
    - Footer with “contingent upon background verification” notice
  - Returns JSON:
    - `html`, `documentId`, plus some key fields (candidateName/email/position/formattedSalary)
- **Connections:** Input from **Generate Offer Letter Content** → output to **Convert to PDF**.
- **Potential failures / edge cases:**
  - **OpenAI output parsing mismatch**: very likely across node versions; could yield “Error extracting text…” inside the letter.
  - HTML injection risk: AI text inserted directly into HTML; if it contains unexpected markup, PDF rendering could break (less likely with typical LLM output, but not guaranteed).
  - “Legally binding” language is included; may be inappropriate without legal review (business risk rather than technical).
  - Hardcoded verification contact: `user@example.com` appears in multiple places.

#### Node: Convert to PDF
- **Type / role:** `n8n-nodes-htmlcsstopdf.htmlcsstopdf` — external HTML-to-PDF rendering service.
- **Config:**
  - `html_content`: `{{$json.html}}` (from Build HTML node)
  - `output_format`: `file`
  - `output_filename`: `data`
  - Requires HTMLCSS to PDF credentials.
- **Connections:** Input from **Build HTML with SVG Signature** → output to **Deliver Offer Letter**.
- **Potential failures / edge cases:**
  - Credential/auth errors, rate limits, rendering timeouts.
  - Large HTML/CSS may exceed service limits.
  - PDF returned as binary must be correctly mapped to Gmail attachment (see next block).

---

### Block 4 — Delivery & Notifications
**Overview:** Emails the offer letter PDF to the candidate using Gmail and sends a Slack message to HR with the document ID and key details.

**Nodes involved:** Deliver Offer Letter, Notify HR Team

#### Node: Deliver Offer Letter
- **Type / role:** `n8n-nodes-base.gmail` — send email with HTML body and attachment.
- **Config highlights:**
  - **To:** `{{$('Prepare Offer Data').item.json.candidateEmail}}`
  - **Subject:** `Job Offer - {{position}} at {{companyName}}`
  - **Message body:** Large HTML email template (congratulatory, next steps, contact info).
  - **Attachments:** configured via UI as “attachmentsBinary”, but the provided configuration is effectively empty (`attachmentsBinary: [ {} ]`).
- **Connections:** Input from **Convert to PDF** → output to **Notify HR Team**.
- **Potential failures / edge cases (important):**
  - **Attachment wiring likely incomplete**: to attach the generated PDF, Gmail node usually needs a binary property name (e.g., `data`) that matches the incoming binary from the PDF node. Current configuration shows an empty attachment entry, which may result in **no PDF attached**.
  - Gmail OAuth issues (expired token, missing scopes).
  - Sending limits, blocked attachments, or invalid recipient address (despite verification, delivery can still fail).
  - HTML email rendering differences across clients.

#### Node: Notify HR Team
- **Type / role:** `n8n-nodes-base.slack` — posts a confirmation message to HR channel.
- **Config highlights:**
  - Posts structured text including:
    - Candidate name/email, position, salary, start date
    - Document ID: `{{ $('Build HTML with SVG Signature').item.json.documentId }}`
  - Channel selected by `channelId` (placeholder `YOUR_SLACK_CHANNEL_ID`)
- **Connections:** Input from **Deliver Offer Letter** → end.
- **Potential failures / edge cases:**
  - Slack credential/auth errors.
  - Channel ID not valid or bot not invited to channel.
  - Slack API rate limits.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook | n8n-nodes-base.webhook | Entry point (HTTP POST intake) | — | Email Verification | ## Input & Email Verification; Receives candidate details via webhook and verifies email deliverability before processing. |
| Email Verification | n8n-nodes-verifiemail.verifiEmail | Validate candidate email deliverability | Webhook | Check Email Validity | ## Input & Email Verification; Receives candidate details via webhook and verifies email deliverability before processing. |
| Check Email Validity | n8n-nodes-base.if | Conditional gate (proceed only if email valid) | Email Verification | Prepare Offer Data (true branch) | ## Input & Email Verification; Receives candidate details via webhook and verifies email deliverability before processing. |
| Prepare Offer Data | n8n-nodes-base.set | Normalize fields, format salary/date, set company/HR defaults | Check Email Validity | Generate Offer Letter Content | ## Data Preparation & AI Generation; Structures candidate data and uses OpenAI to generate a professional, personalized offer letter. |
| Generate Offer Letter Content | @n8n/n8n-nodes-langchain.openAi | Generate offer letter body text using GPT‑4 | Prepare Offer Data | Build HTML with SVG Signature | ## Data Preparation & AI Generation; Structures candidate data and uses OpenAI to generate a professional, personalized offer letter. |
| Build HTML with SVG Signature | n8n-nodes-base.code | Merge AI text into branded HTML, create SVG signature + Document ID | Generate Offer Letter Content | Convert to PDF | ## Document Formatting & Conversion; Builds branded HTML offer letter with digital signature and converts it to PDF format. |
| Convert to PDF | n8n-nodes-htmlcsstopdf.htmlcsstopdf | Render HTML into PDF (binary) | Build HTML with SVG Signature | Deliver Offer Letter | ## Document Formatting & Conversion; Builds branded HTML offer letter with digital signature and converts it to PDF format. |
| Deliver Offer Letter | n8n-nodes-base.gmail | Email candidate (HTML body + PDF attachment) | Convert to PDF | Notify HR Team | ## Delivery & Notifications; Emails PDF to candidate and sends confirmation notification to HR team via Slack. |
| Notify HR Team | n8n-nodes-base.slack | Post Slack confirmation with Document ID | Deliver Offer Letter | — | ## Delivery & Notifications; Emails PDF to candidate and sends confirmation notification to HR team via Slack. |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation/comment | — | — |  |
| Sticky Note2 | n8n-nodes-base.stickyNote | Documentation/comment | — | — |  |
| Sticky Note3 | n8n-nodes-base.stickyNote | Documentation/comment | — | — |  |
| Sticky Note4 | n8n-nodes-base.stickyNote | Documentation/comment | — | — |  |
| Sticky Note5 | n8n-nodes-base.stickyNote | Documentation/comment | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **Verified Job Offer Letter Generator**
   - (Optional) Set execution order to **v1** (workflow settings).

2. **Add “Webhook” trigger**
   - Node type: **Webhook**
   - HTTP Method: **POST**
   - Path: **job-offer-generator**
   - Save and copy the generated webhook URL for later testing/integration.

3. **Add “Email Verification”**
   - Node type: **VerifiEmail**
   - Email field: `={{ $json.body.candidateEmail }}`
   - Credentials: create/select **VerifiEmail account** (API key).

4. **Add “Check Email Validity” (IF)**
   - Node type: **IF**
   - Condition: Boolean `={{ $json.valid }}` is **true**
   - Connect:
     - Webhook → Email Verification → Check Email Validity
     - Use the **true** output only (optional: add a false path response node if you want explicit failure handling).

5. **Add “Prepare Offer Data” (Set)**
   - Node type: **Set**
   - Add fields (as “Assignments”):
     - `candidateName` = `={{ $('Webhook').item.json.body.candidateName }}`
     - `candidateEmail` = `={{ $('Webhook').item.json.body.candidateEmail }}`
     - `position` = `={{ $('Webhook').item.json.body.position }}`
     - `department` = `={{ $('Webhook').item.json.body.department }}`
     - `salary` (number) = `={{ $('Webhook').item.json.body.salary }}`
     - `formattedSalary` = `={{ "$" + Number($('Webhook').item.json.body.salary).toLocaleString() }}`
     - `joiningDate` = `={{ $('Webhook').item.json.body.joiningDate }}`
     - `reportingTo` = `={{ $('Webhook').item.json.body.reportingTo }}`
     - `workLocation` = `={{ $('Webhook').item.json.body.workLocation }}`
     - `offerDate` = `={{ $now.format('MMMM DD, YYYY') }}`
     - `companyName` = `TechCorp Inc. ` (replace with your company)
     - `hrName` = `Sarah Johnson` (replace)
     - `hrTitle` = `VP of Human Resources` (replace)
   - Connect IF (true) → Prepare Offer Data.

6. **Add “Generate Offer Letter Content” (OpenAI / LangChain)**
   - Node type: **OpenAI (LangChain)**
   - Model: **gpt-4**
   - Temperature: **0.7**
   - Add a **System** message describing tone/format constraints (as in workflow).
   - Add a **User** message that injects variables like `{{ $json.candidateName }}`, `{{ $json.position }}`, etc.
   - Credentials: create/select **OpenAI API** credential.
   - Connect Prepare Offer Data → Generate Offer Letter Content.

7. **Add “Build HTML with SVG Signature” (Code)**
   - Node type: **Code**
   - Paste the JavaScript that:
     - Reads `$('Prepare Offer Data').item.json`
     - Extracts the OpenAI text
     - Generates `documentId`
     - Builds the HTML string
     - Returns `{ html, documentId, ... }`
   - Connect Generate Offer Letter Content → Build HTML with SVG Signature.
   - **Important:** If your OpenAI node outputs text in a different JSON path, adjust the extraction logic accordingly.

8. **Add “Convert to PDF”**
   - Node type: **HTMLCSS to PDF**
   - Input HTML: `={{ $json.html }}`
   - Output format: **file**
   - Output filename: `data` (or your preferred name)
   - Credentials: create/select **HTML-to-PDF service** credential.
   - Connect Build HTML with SVG Signature → Convert to PDF.

9. **Add “Deliver Offer Letter” (Gmail)**
   - Node type: **Gmail**
   - Operation: **Send**
   - To: `={{ $('Prepare Offer Data').item.json.candidateEmail }}`
   - Subject: `=Job Offer - {{ $('Prepare Offer Data').item.json.position }} at {{ $('Prepare Offer Data').item.json.companyName }}`
   - Message: use the provided HTML email body (customize branding/contact data).
   - Credentials: create/select **Gmail OAuth2** credential with send-mail permissions.
   - **Attach the generated PDF correctly:**
     - Ensure the PDF node outputs a binary property (commonly `data`).
     - In Gmail node attachments, set the binary property name to that value (e.g., `data`).
   - Connect Convert to PDF → Deliver Offer Letter.

10. **Add “Notify HR Team” (Slack)**
   - Node type: **Slack**
   - Resource/operation: **Post message**
   - Channel: pick your HR channel (set `channelId`)
   - Text: include candidate/job fields and `documentId` from Build HTML node.
   - Credentials: create/select **Slack API** credential; ensure the app/bot is in the channel.
   - Connect Deliver Offer Letter → Notify HR Team.

11. **Test end-to-end**
   - POST JSON to the webhook, for example:
     - candidateName, candidateEmail, position, department, salary, joiningDate, reportingTo, workLocation
   - Verify:
     - IF gate passes only for valid emails
     - PDF is actually attached in Gmail
     - Slack message posts with a real Document ID

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow automates the complete job offer letter generation and delivery process: webhook intake → email verification → OpenAI letter body → branded HTML + SVG signature + document ID → PDF conversion → Gmail delivery → Slack confirmation. | From sticky note “How It Works” |
| Setup steps: connect OpenAI, VerifiEmail, HTML-to-PDF, Gmail, Slack; update branding; customize HR signature; set Slack channel; adjust prompt; integrate webhook with ATS; test with sample data. | From sticky note “Setup Steps” |
| Customization ideas: adjust system prompt for values/culture; update HTML styling/colors; add more fields (bonus/equity/benefits); customize email template. | From sticky note “Customization” |