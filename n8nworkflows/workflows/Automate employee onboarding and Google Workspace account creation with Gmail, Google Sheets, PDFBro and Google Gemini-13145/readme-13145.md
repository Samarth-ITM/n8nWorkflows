Automate employee onboarding and Google Workspace account creation with Gmail, Google Sheets, PDFBro and Google Gemini

https://n8nworkflows.xyz/workflows/automate-employee-onboarding-and-google-workspace-account-creation-with-gmail--google-sheets--pdfbro-and-google-gemini-13145


# Automate employee onboarding and Google Workspace account creation with Gmail, Google Sheets, PDFBro and Google Gemini

## 1. Workflow Overview

**Title:** Automate employee onboarding and Google Workspace account creation with Gmail, Google Sheets, PDFBro and Google Gemini

This workflow automates two major onboarding phases:

### 1.1 Candidate intake → Offer letter generation and delivery (Webhook-driven)
- Receives candidate data via a secured Webhook (Basic Auth).
- Normalizes/rehydrates the JSON fields for predictable mapping.
- Stores candidate details in a Google Sheets “Employee DB”.
- Generates a personalized offer letter PDF using PDFBro.
- Emails the offer letter to the candidate through Gmail.

### 1.2 Reply monitoring → Acceptance verification → Google Workspace provisioning (Email-driven)
- Polls Gmail for incoming candidate replies.
- Finds the candidate record in Google Sheets via sender email.
- Ensures the candidate hasn’t already been provisioned (“workplace_acc_created?” is not “yes”).
- Uses Google Gemini to classify whether the email is an offer acceptance.
- If accepted, generates a temporary password, creates the Google Workspace account, emails credentials, then marks the DB row as provisioned.

**Entry points**
- **Webhook** node (HTTP POST) for candidate submission.
- **Gmail Trigger** node (polling) for processing acceptance replies.

---

## 2. Block-by-Block Analysis

### Block A — Candidate Intake & Normalization (Webhook)
**Overview:** Accepts structured candidate payloads and maps required fields into consistent keys used by downstream nodes.

**Nodes involved**
- Webhook
- Edit Fields

#### Node: Webhook
- **Type / role:** `n8n-nodes-base.webhook` — inbound HTTP entry point for candidate data.
- **Configuration (interpreted):**
  - Method: **POST**
  - Path: auto-generated UUID-like string (used as endpoint path)
  - Auth: **Basic Auth** enabled (protects endpoint)
- **Credentials:** HTTP Basic Auth credential named `hrr`.
- **Inputs/Outputs:** No input; output goes to **Edit Fields**.
- **Edge cases / failures:**
  - 401 Unauthorized if Basic Auth incorrect/missing.
  - Payload not matching expected JSON structure will cause expression failures downstream (e.g., missing `body.personal_info.*`).

#### Node: Edit Fields
- **Type / role:** `n8n-nodes-base.set` — copies/normalizes nested values into explicit fields (still nested keys, but explicitly assigned).
- **Configuration choices:**
  - Uses **Assignments** to map values like:
    - `body.personal_info.full_name = {{$json.body.personal_info.full_name}}`
    - `body.personal_info.address` as an **object**
    - many other nested fields (bank, emergency contact, job info)
- **Key expressions/variables:** references `$json.body.personal_info...` from Webhook payload.
- **Inputs/Outputs:** Input from Webhook; output to **Save Candidates Details to DB**.
- **Edge cases / failures:**
  - If `body.personal_info.address` is missing or not an object, later address string concatenation (PDF node) may fail.
  - If any nested properties are undefined, Gmail/PDF mapping can produce blank values or expression errors depending on n8n settings.

---

### Block B — Persist Candidate to Google Sheets (“Employee DB”)
**Overview:** Stores candidate details into a central sheet and marks the candidate as not yet provisioned.

**Nodes involved**
- Save Candidates Details to DB

#### Node: Save Candidates Details to DB
- **Type / role:** `n8n-nodes-base.googleSheets` — appends or updates the candidate record in Sheets.
- **Configuration choices:**
  - Operation: **Append or Update**
  - Auth: **Service Account**
  - Document: “Employee DB” (Spreadsheet ID provided)
  - Sheet tab: “Sheet1”
  - Writes multiple columns, including:
    - Personal info, address, job info, national id, bank info, emergency contact
    - `workplace_acc_created? = "no"`
  - Matching columns: **contact_number** (used to decide update vs append)
- **Credentials:** Google Service Account credential `Google Service Account account`.
- **Inputs/Outputs:** Input from **Edit Fields**; output to **Generate Offer Letter**.
- **Edge cases / failures:**
  - If `contact_number` is missing/duplicated, the append/update behavior can update the wrong row or create duplicates.
  - Service account must have access to the spreadsheet; otherwise 403 errors.
  - Column schema mismatches (renamed columns in sheet) can cause write failures.

---

### Block C — Offer Letter PDF Generation & Email Delivery
**Overview:** Creates a customized offer letter PDF and sends it to the candidate via Gmail.

**Nodes involved**
- Generate Offer Letter
- Send Offer letter via mail

#### Node: Generate Offer Letter
- **Type / role:** `n8n-nodes-pdfbro.pdfBro` — community node used to generate an offer letter PDF.
- **Configuration choices:**
  - Operation: **offerLetter**
  - Populates fields using expressions from **Edit Fields** output:
    - Candidate name, preferred name, email
    - Job title, department, manager name
    - Address composed from street/city/state/zip/country
  - Payment frequency: **monthly**
- **Inputs/Outputs:** Input from Google Sheets write node; output to **Send Offer letter via mail**.
- **Version-specific requirements:**
  - Requires community node: **`n8n-nodes-pdfbro`** installed on the n8n instance.
- **Edge cases / failures:**
  - Missing address subfields (street/city/state/zip/country) can produce malformed address strings.
  - PDFBro template/operation misconfiguration can cause runtime errors or missing attachment binary.

#### Node: Send Offer letter via mail
- **Type / role:** `n8n-nodes-base.gmail` — sends the offer letter email.
- **Configuration choices:**
  - To: candidate personal email from **Edit Fields**
  - Subject: “Offer Letter”
  - Body: HTML email with placeholders for candidate name and job title.
  - Attachments: configured via `attachmentsBinary`, but the node configuration shows an empty binary reference (`attachmentsBinary: [{}]`).
    - This implies the node expects a binary property from upstream (PDFBro) and may require selecting the actual binary field name in the UI.
  - Attribution disabled (`appendAttribution: false`)
- **Credentials:** Gmail OAuth2 credential `Abhiram.bvb`.
- **Inputs/Outputs:** Input from PDFBro; terminal node for this branch.
- **Edge cases / failures:**
  - If the PDF binary is not correctly referenced, the email may send without attachment or fail validation.
  - Gmail API quota/auth issues (invalid refresh token, revoked consent).
  - The “View Offer Letter” link is a placeholder (`YOUR_OFFER_LETTER_LINK_HERE`) and does not point to an actual document.

---

### Block D — Incoming Email Monitoring & Candidate Lookup
**Overview:** Polls Gmail for new emails and checks whether the sender exists in the Employee DB and still requires provisioning.

**Nodes involved**
- Gmail Trigger
- Get row(s) in sheet
- Filter Only Candidate Emails

#### Node: Gmail Trigger
- **Type / role:** `n8n-nodes-base.gmailTrigger` — polling trigger for new emails.
- **Configuration choices:**
  - Polling frequency: **every minute**
  - No filters defined (will react to all messages it detects as “new” per node behavior)
- **Credentials:** Gmail OAuth2 credential `Abhiram.bvb`.
- **Outputs:** To **Get row(s) in sheet**.
- **Edge cases / failures:**
  - Without filters, non-candidate emails may trigger unnecessary processing.
  - Polling can miss/duplicate events depending on Gmail trigger semantics and mailbox volume.

#### Node: Get row(s) in sheet
- **Type / role:** `n8n-nodes-base.googleSheets` — fetch candidate row by email.
- **Configuration choices:**
  - Operation: “Get row(s)”
  - Filter:
    - `lookupColumn = personal_email`
    - `lookupValue` extracts email from `From` header:
      - If `From` includes `<...>`, uses regex to extract email inside angle brackets.
      - Otherwise uses `From` as-is.
- **Credentials:** Google Service Account credential.
- **Outputs:** To **Filter Only Candidate Emails**.
- **Edge cases / failures:**
  - If multiple rows match the same personal email, downstream nodes may run once per matched row (potential multiple account creations).
  - If `From` contains unexpected formatting, regex may fail, producing wrong lookup value and zero rows.

#### Node: Filter Only Candidate Emails
- **Type / role:** `n8n-nodes-base.if` — prevents duplicate account provisioning.
- **Configuration choices:**
  - Condition: `workplace_acc_created? != "yes"`
  - True branch continues; false branch stops.
- **Inputs/Outputs:** Input from sheet lookup; true → **check OL status**.
- **Edge cases / failures:**
  - If the sheet column name changes (e.g., `workplace_acc_created?` renamed), condition will evaluate incorrectly.
  - Values like “Yes”/“YES” won’t match strict comparison (case-sensitive).

---

### Block E — AI Acceptance Classification (Gemini + Agent)
**Overview:** Uses Google Gemini to classify whether the incoming email is an offer acceptance.

**Nodes involved**
- Google Gemini Chat Model
- check OL status
- check for acceptance of OL

#### Node: Google Gemini Chat Model
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` — provides an LLM backend for the agent.
- **Configuration choices:**
  - Default options (no explicit temperature/model parameters shown).
- **Credentials:** `Google Gemini(PaLM) Api account 3`.
- **Connections:** Connected to agent node via **ai_languageModel** port.
- **Edge cases / failures:**
  - API key/permission issues.
  - Rate limits or transient 5xx errors.
  - Model output variability (may not strictly return “yes”/“no” unless constrained well).

#### Node: check OL status
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — LLM agent that produces a classification output.
- **Configuration choices:**
  - Prompt instructs: respond **only** with `"yes"` if acceptance, else `"no"`.
  - Injects:
    - Email Subject: from `Gmail Trigger` → `Subject`
    - Email Body: from `Gmail Trigger` → `snippet`
- **Inputs/Outputs:** Input from **Filter Only Candidate Emails**; output to **check for acceptance of OL**.
- **Edge cases / failures:**
  - Using only Gmail `snippet` may omit the actual acceptance text; classification may be wrong.
  - If the agent outputs variants like “Yes.” or “yes\n”, the downstream “contains yes” condition will still pass, but other variants may cause false negatives.

#### Node: check for acceptance of OL
- **Type / role:** `n8n-nodes-base.if` — branches if acceptance confirmed.
- **Configuration choices:**
  - Condition: `$json.output` **contains** `"yes"`
  - True → provisioning pipeline; False → stop.
- **Edge cases / failures:**
  - If agent returns JSON in a different shape (not `output`), condition fails.
  - False positives possible if the model output includes “yesterday” etc. (contains “yes”). Safer would be equals/regex match.

---

### Block F — Password Generation → Workspace User Creation → Credential Email → DB Update
**Overview:** Generates a temporary password, creates a Google Workspace user, emails credentials to candidate, and marks the sheet row as provisioned.

**Nodes involved**
- Code in JavaScript
- Create a user
- Send a message1
- Update row in sheet1

#### Node: Code in JavaScript
- **Type / role:** `n8n-nodes-base.code` — generates a 16-character temporary password/token.
- **Configuration choices:**
  - Combines:
    - `Date.now().toString(36)` (timestamp base36)
    - `Math.random().toString(36).substring(2)` (random base36)
  - Trims to 16 characters
  - Outputs `{ json: { id: finalValue } }`
- **Inputs/Outputs:** Input from acceptance IF; output to **Create a user**.
- **Edge cases / failures:**
  - Not cryptographically secure; should not be used as a high-security password generator.
  - Output key is `id`; downstream nodes must reference it correctly.

#### Node: Create a user
- **Type / role:** `n8n-nodes-base.gSuiteAdmin` — creates a Google Workspace user.
- **Configuration choices:**
  - Domain: `blankarray.com`
  - First/Last name derived from `full_name` in sheet:
    - `full_name.split(' - ')[0].split(' ')[0]` (first token as first name)
    - `full_name.split(' - ')[0].split(' ')[1]` (second token as last name)
  - Username: hard-coded `vaar1`
  - Password: hard-coded placeholder `YOUR_CREDENTIAL_HERE`
  - Force password change at next login: enabled
- **Credentials:** Workspace Admin OAuth2 credential `Google Workspace Admin account`.
- **Inputs/Outputs:** Input from Code node; output to **Send a message1**.
- **Major implementation gaps (important):**
  - The generated password (`Code in JavaScript` → `id`) is **not used** here; password remains the placeholder.
  - Username is **not dynamically generated** and will cause conflicts after first run.
- **Edge cases / failures:**
  - User already exists (409 conflict).
  - Insufficient admin scopes/privileges (403).
  - Name parsing breaks for multi-part names (e.g., “Mary Jane Watson” → last name becomes “Jane” only; if only one word, lastName becomes undefined).

#### Node: Send a message1
- **Type / role:** `n8n-nodes-base.gmail` — sends credentials email to candidate.
- **Configuration choices:**
  - To: `Get row(s) in sheet` → `personal_email`
  - Subject: “Inviting you to our Workspace”
  - Body: rich HTML template including:
    - Work Email: `{{ $json.primaryEmail }}` (expects from “Create a user” output)
    - Temporary Password: `{{ $('Code in JavaScript').item.json.id }}`
- **Credentials:** Gmail OAuth2 `Abhiram.bvb`.
- **Inputs/Outputs:** Input from **Create a user**; output to **Update row in sheet1**.
- **Edge cases / failures:**
  - If Create user fails or doesn’t return `primaryEmail`, email will have blank work email.
  - Sending passwords by email is a security risk; consider secure delivery methods.

#### Node: Update row in sheet1
- **Type / role:** `n8n-nodes-base.googleSheets` — marks candidate as provisioned.
- **Configuration choices:**
  - Operation: **Update**
  - Matching column: `personal_email`
  - Values updated:
    - `workplace_acc_created? = "yes"`
    - `personal_email` retained from lookup row
- **Credentials:** Google Service Account.
- **Inputs/Outputs:** Input from Gmail send node; terminal.
- **Edge cases / failures:**
  - If multiple rows share same personal_email, multiple updates may occur unpredictably.
  - If candidate row wasn’t found earlier (no items), this node may not run or may error depending on execution mode.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook | Webhook | Candidate intake endpoint (secured POST) | — | Edit Fields | ## 1. Automated Offer Letter Creation and Sending / Intake engine description + “Note: This workflow needs a community node named "n8n-nodes-pdfbro"” |
| Edit Fields | Set | Normalize/map incoming candidate payload fields | Webhook | Save Candidates Details to DB | ## 1. Automated Offer Letter Creation and Sending / Intake engine description + “Note: This workflow needs a community node named "n8n-nodes-pdfbro"” |
| Save Candidates Details to DB | Google Sheets | Persist candidate record; set workplace_acc_created? = no | Edit Fields | Generate Offer Letter | ## 1. Automated Offer Letter Creation and Sending / Intake engine description + “Note: This workflow needs a community node named "n8n-nodes-pdfbro"” |
| Generate Offer Letter | PDFBro | Generate offer letter PDF from candidate/job fields | Save Candidates Details to DB | Send Offer letter via mail | ## 1. Automated Offer Letter Creation and Sending / Intake engine description + “Note: This workflow needs a community node named "n8n-nodes-pdfbro"” |
| Send Offer letter via mail | Gmail | Email offer letter (HTML + attachment) | Generate Offer Letter | — | ## 1. Automated Offer Letter Creation and Sending / Intake engine description + “Note: This workflow needs a community node named "n8n-nodes-pdfbro"” |
| Gmail Trigger | Gmail Trigger | Poll inbox for replies | — | Get row(s) in sheet | ## 2. Offer Letter Acceptance Verifier & Work Account Creator / monitoring + provisioning description |
| Get row(s) in sheet | Google Sheets | Find candidate row by sender email | Gmail Trigger | Filter Only Candidate Emails | ## 2. Offer Letter Acceptance Verifier & Work Account Creator / monitoring + provisioning description |
| Filter Only Candidate Emails | IF | Block candidates already provisioned | Get row(s) in sheet | check OL status (true) | ## 2. Offer Letter Acceptance Verifier & Work Account Creator / monitoring + provisioning description |
| Google Gemini Chat Model | LangChain LLM (Gemini) | Provides LLM backend | — | check OL status (ai_languageModel) | ## 2. Offer Letter Acceptance Verifier & Work Account Creator / monitoring + provisioning description |
| check OL status | LangChain Agent | Classify acceptance from subject/snippet | Filter Only Candidate Emails + Gemini model | check for acceptance of OL | ## 2. Offer Letter Acceptance Verifier & Work Account Creator / monitoring + provisioning description |
| check for acceptance of OL | IF | Branch on “yes” acceptance | check OL status | Code in JavaScript (true) | ## 2. Offer Letter Acceptance Verifier & Work Account Creator / monitoring + provisioning description |
| Code in JavaScript | Code | Generate temporary 16-char password | check for acceptance of OL (true) | Create a user | ## 2. Offer Letter Acceptance Verifier & Work Account Creator / monitoring + provisioning description |
| Create a user | Google Workspace Admin | Provision Workspace user | Code in JavaScript | Send a message1 | ## 2. Offer Letter Acceptance Verifier & Work Account Creator / monitoring + provisioning description |
| Send a message1 | Gmail | Send credentials email | Create a user | Update row in sheet1 | ## 2. Offer Letter Acceptance Verifier & Work Account Creator / monitoring + provisioning description |
| Update row in sheet1 | Google Sheets | Mark workplace account created in DB | Send a message1 | — | ## 2. Offer Letter Acceptance Verifier & Work Account Creator / monitoring + provisioning description |
| Sticky Note | Sticky Note | Documentation / grouping | — | — | (same content as node) |
| Sticky Note1 | Sticky Note | Documentation / grouping | — | — | (same content as node) |
| Sticky Note2 | Sticky Note | Visual header: Candidate Details Procurement | — | — | ## Candidate Details Procurement |
| Sticky Note3 | Sticky Note | Visual header: Offer Letter Creation and sending email | — | — | ## Offer Letter Creation and sending email |
| Sticky Note4 | Sticky Note | Visual header: Strict Verification for candidates email | — | — | ## Strict Verification for candidates email |
| Sticky Note5 | Sticky Note | Visual header: AI based Offer Letter Acceptance checker | — | — | ## AI based Offer Letter Acceptance checker |
| Sticky Note6 | Sticky Note | Visual header: Creates temporary password and internal account + email | — | — | ## Creates temporary passsword and create internal work account and send those details via email |
| Sticky Note7 | Sticky Note | Visual header: Change the status of work account delivery | — | — | ## Change the status of work account delivery |

---

## 4. Reproducing the Workflow from Scratch

### Prerequisites (credentials, apps, and sheet)
1. **Google Sheet**
   - Create a spreadsheet named “Employee DB”.
   - Create a sheet/tab “Sheet1” with columns matching the workflow fields, including:
     - `personal_email`, `full_name`, `contact_number`, `job_info.job_title`, `job_info.department`, `job_info.manager_name`, address fields, etc.
     - `workplace_acc_created?` (string)
2. **Google Service Account**
   - Create a Google Cloud service account with access to Google Sheets API.
   - Share the “Employee DB” spreadsheet with the service account email.
   - Create an n8n credential: **Google Sheets (Service Account)**.
3. **Gmail OAuth2**
   - Connect HR Gmail account to n8n as **Gmail OAuth2** credential.
   - Ensure it can send emails and read mailbox for trigger.
4. **Google Workspace Admin OAuth2**
   - Connect an admin account with permissions to create users (Directory API scopes).
   - Create n8n credential: **Google Workspace Admin**.
5. **Google Gemini API**
   - Create Gemini/PaLM API key credential in n8n used by the Gemini Chat Model node.
6. **Community node**
   - Install **`n8n-nodes-pdfbro`** on your n8n instance and restart n8n.

---

### Build Branch 1: Webhook → Sheet → PDF → Offer email
1. **Add node: Webhook**
   - Type: *Webhook*
   - Method: **POST**
   - Authentication: **Basic Auth**
   - Path: generate a unique path (n8n will create one); keep it for the caller.
   - Select/attach HTTP Basic Auth credential.
2. **Add node: Edit Fields**
   - Type: *Set*
   - Create fields mirroring the candidate payload keys you expect (e.g. `body.personal_info.full_name`, etc.).
   - Map each value using expressions from `$json.body.personal_info...`.
3. **Connect:** Webhook → Edit Fields
4. **Add node: Save Candidates Details to DB**
   - Type: *Google Sheets*
   - Auth: **Service Account**
   - Operation: **Append or Update**
   - Document: select “Employee DB”
   - Sheet: select “Sheet1”
   - Mapping: define columns to write (full name, email, etc.)
   - Set `workplace_acc_created?` to `"no"`.
   - Matching columns: choose your dedupe key (in the provided workflow it is `contact_number`).
5. **Connect:** Edit Fields → Save Candidates Details to DB
6. **Add node: Generate Offer Letter**
   - Type: *PDFBro* (offer letter operation)
   - Operation: **offerLetter**
   - Map required placeholders (job title, department, manager name, candidate info).
   - Ensure the node outputs a PDF as **binary** (verify binary property name in node output).
7. **Connect:** Save Candidates Details to DB → Generate Offer Letter
8. **Add node: Send Offer letter via mail**
   - Type: *Gmail (Send)*
   - To: candidate personal email (expression from Edit Fields)
   - Subject: “Offer Letter”
   - Body: HTML content (as desired)
   - Attachments: select the binary property produced by PDFBro (this is essential; don’t leave it unselected).
9. **Connect:** Generate Offer Letter → Send Offer letter via mail

---

### Build Branch 2: Gmail Trigger → Sheet lookup → AI check → Provision account
10. **Add node: Gmail Trigger**
   - Type: *Gmail Trigger*
   - Poll time: **Every minute**
   - (Recommended) Add filters/labels to limit to candidate replies if possible.
11. **Add node: Get row(s) in sheet**
   - Type: *Google Sheets*
   - Auth: **Service Account**
   - Operation: **Get row(s)**
   - Document: “Employee DB”, Sheet: “Sheet1”
   - Filter:
     - Lookup column: `personal_email`
     - Lookup value expression: extract email from `From` header (same logic as workflow).
12. **Connect:** Gmail Trigger → Get row(s) in sheet
13. **Add node: Filter Only Candidate Emails**
   - Type: *IF*
   - Condition: `workplace_acc_created?` **not equals** `"yes"` (case-sensitive).
14. **Connect:** Get row(s) in sheet → Filter Only Candidate Emails (true path continues)

---

### AI acceptance checker (Gemini)
15. **Add node: Google Gemini Chat Model**
   - Type: *LangChain Chat Model (Google Gemini)*
   - Select Gemini API credential.
16. **Add node: check OL status**
   - Type: *LangChain Agent*
   - Prompt: instruct output to be exactly `"yes"` for acceptance else `"no"`.
   - Include email subject/snippet using expressions from Gmail Trigger.
   - Connect the **Gemini Chat Model** to the agent’s **AI Language Model** input.
17. **Connect:** Filter Only Candidate Emails (true) → check OL status
18. **Add node: check for acceptance of OL**
   - Type: *IF*
   - Condition: `$json.output` contains `"yes"` (or safer: equals `"yes"` after trimming).
19. **Connect:** check OL status → check for acceptance of OL (true path continues)

---

### Provisioning + notifications + DB update
20. **Add node: Code in JavaScript**
   - Type: *Code*
   - Generate a 16-character password and output it as `json.id` (same as workflow).
21. **Connect:** check for acceptance of OL (true) → Code in JavaScript
22. **Add node: Create a user**
   - Type: *Google Workspace Admin* (GSuite Admin)
   - Operation: create user
   - Domain: your Workspace domain
   - First/Last name: map from sheet fields (or better: store first/last separately instead of splitting)
   - Username: generate dynamically (recommended) to avoid collisions
   - Password: map to the generated password (recommended): use `{{$('Code in JavaScript').item.json.id}}`
   - Enable: “Change password at next login”
23. **Connect:** Code in JavaScript → Create a user
24. **Add node: Send a message1**
   - Type: *Gmail (Send)*
   - To: candidate `personal_email` from sheet
   - Include:
     - Work email: from Create a user output (`primaryEmail`)
     - Temporary password: from Code node (`id`)
25. **Connect:** Create a user → Send a message1
26. **Add node: Update row in sheet1**
   - Type: *Google Sheets*
   - Operation: **Update**
   - Match by: `personal_email`
   - Update `workplace_acc_created?` to `"yes"`
27. **Connect:** Send a message1 → Update row in sheet1

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Note: This workflow needs a community node named ‘n8n-nodes-pdfbro’” | Required to run the PDF generation step (PDFBro node). |
| “YOUR_OFFER_LETTER_LINK_HERE” is a placeholder | Replace with an actual hosted offer letter link if you want a “View Offer Letter” CTA in email. |
| Current Workspace provisioning config is not fully wired | The generated password is not used in “Create a user”, and the username is hard-coded (`vaar1`), causing conflicts and security issues. |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.