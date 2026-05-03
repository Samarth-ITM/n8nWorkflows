Qualify mortgage leads and collect documents with Gemini, Gmail, Drive, Telegram, and Supabase

https://n8nworkflows.xyz/workflows/qualify-mortgage-leads-and-collect-documents-with-gemini--gmail--drive--telegram--and-supabase-13387


# Qualify mortgage leads and collect documents with Gemini, Gmail, Drive, Telegram, and Supabase

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow qualifies mortgage leads, enforces a human approval step for high-value leads, drafts a personalized outreach email using Gemini, provisions a dedicated Google Drive folder for document collection, and then collects/validates client documents via an upload form. It logs lead and upload outcomes to Supabase and notifies the broker via Telegram.

**Target use cases:**
- Mortgage brokers who want fast lead triage (e.g., by income threshold).
- Preventing accidental outreach by requiring Telegram approval before drafting/sending anything.
- Collecting required documents (ID, payslip, bank statement) into client-specific Drive folders with real-time broker notifications.
- Maintaining a lead + activity audit trail in Supabase.

### Logical Blocks
**1.1 Lead Capture & Normalization**  
Form intake → normalize name fields → income threshold gating.

**1.2 Human Approval (Telegram) & Lead Logging (Decline path)**  
Send lead summary to Telegram for approve/decline → branch on approval → log declined leads.

**1.3 AI Email Draft + Drive Folder Provisioning (Approved path)**  
Generate Gemini email body → create Drive folder → share folder → create Gmail draft containing upload link → Telegram confirmation.

**1.4 Activity/Error Aggregation + Supabase Storage (Approved path)**  
Aggregate execution status/errors → insert lead record into Supabase including links and error details.

**1.5 Document Upload Portal + Validation**  
Second form intake (files + hidden fields) → validate required inputs/files count.

**1.6 Drive Uploads + Results Evaluation + Notifications + Supabase Update**  
Upload 3 files to Drive (continue on fail) → compute success/failure summary → notify broker → update Supabase with upload error details.

---

## 2. Block-by-Block Analysis

### 2.1 Lead Capture & Normalization
**Overview:**  
Captures a new mortgage inquiry via an n8n form, then normalizes the lead’s name into first/last name fields to be reused across the workflow, and gates execution using an income threshold.

**Nodes Involved:**
- On form submission
- Normalize lead name
- Check income threshold

#### Node: **On form submission**
- **Type / Role:** `Form Trigger` — entry point for lead intake.
- **Configuration:**
  - Form title: “Provide Your details”
  - Fields (required): `name`, `email`, `loan_type` (radio: Home Loan/Commercial/Refinance), `income` (number), `residency_status` (radio: Yes/No)
- **Inputs/Outputs:** No inputs (trigger). Outputs one item with submitted fields.
- **Edge cases / failures:**
  - Invalid email format or missing required fields (blocked by form itself).
  - Income may come in as string depending on form handling; downstream IF uses numeric comparison with strict validation.
- **Version notes:** Node typeVersion `2.5` (formTrigger behavior/field schema depends on this version).

#### Node: **Normalize lead name**
- **Type / Role:** `Code` — parses/cleans the `name` field into structured fields.
- **Configuration choices (interpreted):**
  - Trims and lowercases raw name, then converts to Title Case (first letter of each word uppercased).
  - Splits on whitespace:
    - `first_name = parts[0]`
    - `last_name = rest joined`
  - Adds `full_name_clean`
- **Key variables/expressions:** Uses `$input.first().json.name`
- **Inputs/Outputs:**
  - Input: lead item from form trigger
  - Output: same JSON plus `first_name`, `last_name`, `full_name_clean`
- **Edge cases / failures:**
  - Single-word name → `last_name` becomes empty string (handled).
  - Empty/whitespace name should be prevented by required field, but if it happens `.trim()` on undefined would throw; ensure field always exists.

#### Node: **Check income threshold**
- **Type / Role:** `IF` — lead qualification gate.
- **Configuration:**
  - Condition: `income > 80000` (strict type validation)
- **Inputs/Outputs:**
  - True → Request Email Approval1
  - False → no connection (lead is effectively dropped; no logging for low-income path in this workflow)
- **Edge cases / failures:**
  - If `income` is not numeric, strict validation can fail or evaluate unexpectedly; consider casting (`Number($json.income)`) if needed.
  - Low-income leads are not recorded anywhere—could be intentional, but it’s a common audit gap.

---

### 2.2 Human Approval (Telegram) & Decline Logging
**Overview:**  
For qualified leads, sends a Telegram “send and wait” approval request. Approved leads proceed to AI drafting; declined leads are logged in Supabase.

**Nodes Involved:**
- Request Email Approval1
- Check approval response
- Log declined lead

#### Node: **Request Email Approval1**
- **Type / Role:** `Telegram` (`sendAndWait`) — human-in-the-loop approval gate.
- **Configuration:**
  - Sends lead summary (name/email/income/loan type/residency).
  - Approval options: `approvalType: double` (Telegram inline approval UX).
  - Chat ID hard-coded: `123456789`
- **Key expressions:**
  - Uses normalized fields: `$('Normalize lead name').item.json.full_name_clean`, etc.
- **Inputs/Outputs:**
  - Input: qualified lead item
  - Output: Telegram response payload including approval decision (later read as `$json.data.approved`)
- **Edge cases / failures:**
  - Telegram credential invalid, chat ID wrong, bot not in chat, or user blocks bot.
  - Waiting for approval can time out depending on node settings/platform; consider operational expectations.
  - If multiple leads are processed concurrently, “sendAndWait” responses must map correctly per execution (n8n generally handles this within an execution).

#### Node: **Check approval response**
- **Type / Role:** `IF` — routes based on Telegram approval.
- **Configuration:**
  - Condition: `={{ $json.data.approved }} == true`
- **Inputs/Outputs:**
  - True → Generate Personalized Email
  - False → Log declined lead
- **Edge cases / failures:**
  - If Telegram output schema differs (e.g., `approved` not under `data`), condition will be false/undefined. This is a common break when changing node versions or approval modes.

#### Node: **Log declined lead**
- **Type / Role:** `Supabase` (insert) — records declined leads.
- **Configuration:**
  - Table: `leads_consolidated`
  - Fields inserted: first/last name, email, loan_type, income, created_at (`$now`), residency_status, email_approved = “Declined”
  - Note: residency_status is taken from `On form submission` node directly.
- **Inputs/Outputs:** Input: declined lead; Output: Supabase insert response (unused).
- **Edge cases / failures:**
  - Supabase auth/permissions (RLS) can block inserts.
  - Schema mismatch (column names/types).
  - `created_at` expects a timestamp type; `$now` format must match Supabase expectations.

---

### 2.3 AI Email Draft + Drive Folder Provisioning (Approved path)
**Overview:**  
Generates a personalized email body with Gemini, creates and shares a Drive folder for the client, then creates a Gmail draft containing an upload button linking to the document upload form with folder/client parameters.

**Nodes Involved:**
- Generate Personalized Email
- Create Drive Folder
- Share Drive Folder
- Create Gmail Draft
- Send Email Created Confirmation

#### Node: **Generate Personalized Email**
- **Type / Role:** `Google Gemini (LangChain)` — generates email body text.
- **Configuration:**
  - Model: `models/gemini-2.5-flash-lite`
  - Temperature: `0.4`
  - Prompt instructs persona (“Daniel, mortgage broker at Lendium in Australia”) and includes lead variables (first name, loan type, income).
  - Output constraint: “Return ONLY the email body text, no subject line.”
- **Key expressions:**
  - `{{ $('Normalize lead name').item.json.first_name }}`
  - `{{ $('Normalize lead name').item.json.loan_type }}`
  - `{{ $('Normalize lead name').item.json.income }}`
- **Inputs/Outputs:**
  - Input: approved lead
  - Output: Gemini message object; downstream uses `...content.parts[0].text`
- **Edge cases / failures:**
  - Gemini API auth/quotas, model availability changes, latency/timeouts.
  - Output structure may differ across node versions/models; reliance on `content.parts[0].text` is brittle if schema changes.

#### Node: **Create Drive Folder**
- **Type / Role:** `Google Drive` (create folder) — provisions client folder under a parent folder.
- **Configuration:**
  - Folder name: `FirstName (LoanType)`
  - Parent folder: “Lendium Files” (`1vleObg6ZBzUflpAVT2Joc5D7oBNYWNUw`)
  - Drive: “My Drive”
- **Inputs/Outputs:**
  - Input: Gemini output item (same execution item chain)
  - Output: folder metadata including `id` (used widely)
- **Edge cases / failures:**
  - Permission issues on parent folder.
  - Duplicate folder names (Drive allows duplicates; could create confusion).
  - If the authenticated user’s Drive differs (Shared Drive vs My Drive), folder creation behavior changes.

#### Node: **Share Drive Folder**
- **Type / Role:** `Google Drive` (share folder) — makes folder accessible.
- **Configuration:**
  - Operation: share
  - Permission: `role=writer`, `type=anyone` (public link with write access)
  - **Important:** The node’s `folderNoRootId` is hard-coded to `1YJ026YLa4IT-rk-AqCUlHcdWaxvhDPLE`, not the newly created folder ID.
- **Inputs/Outputs:**
  - Input: Create Drive Folder
  - Output: share/permission response
- **Critical integration issue:**
  - As configured, it likely shares a fixed folder, not the per-lead folder. If the intent is to share the newly created folder, `folderNoRootId` should reference `{{$node["Create Drive Folder"].json.id}}`.
- **Security note:** `anyone` + `writer` means anyone with the link can upload/modify; consider `reader` or restricting to the client’s Google account if needed.

#### Node: **Create Gmail Draft**
- **Type / Role:** `Gmail` (draft create) — creates an HTML draft email to the lead.
- **Configuration:**
  - To: lead email (`Normalize lead name` → `email`)
  - Subject: `Re: Your {{loan_type}} Inquiry - Lendium Mortgage Services`
  - Body:
    - Gemini-generated email body
    - HTML “Upload Folder” button linking to:
      - `https://13.211.164.241.nip.io/form/df9aebf7-3e4f-4114-bc43-424c3df3511e?folder_id={{ Create Drive Folder.id }}&client_name={{ first_name }}`
- **Key expressions:**
  - Email body: `$('Generate Personalized Email').item.json.content.parts[0].text`
  - Link uses `Create Drive Folder` id and lead first name.
- **Inputs/Outputs:**
  - Input: Share Drive Folder
  - Output: draft metadata
- **Edge cases / failures:**
  - Gmail OAuth scopes insufficient, token expired.
  - HTML may be sanitized by Gmail; button styling may change.
  - If `Create Drive Folder` fails, link will be malformed.
  - The form URL points to a specific host; if that endpoint changes, the upload portal breaks.

#### Node: **Send Email Created Confirmation**
- **Type / Role:** `Telegram` — broker notification that the draft is ready.
- **Configuration:**
  - Sends the drafted email body and a Drive folder view link.
  - Chat ID: `123456789`
- **Inputs/Outputs:** Input: Create Gmail Draft; Output unused.
- **Edge cases / failures:** Telegram delivery/auth issues.

---

### 2.4 Activity/Error Aggregation + Supabase Storage (Approved path)
**Overview:**  
Aggregates workflow execution signals into a structured log payload, then inserts a record into Supabase including links and error detail JSON.

**Nodes Involved:**
- Aggregate Log Data
- Store in Database

#### Node: **Aggregate Log Data**
- **Type / Role:** `Code` — compiles approval status, checks for node-level error indicators, and extracts lead basics.
- **Configuration choices (interpreted):**
  - Initializes:
    - `email_approved` (boolean-like)
    - `lead_activity_error_details` (JSON string)
    - `lead_first_name`, `email`
  - Attempts to read approval outcome from a node named **`Request Email Approval`** (but actual node name is **`Request Email Approval1`**).
  - Checks for errors by inspecting outputs of:
    - `Generate Strategy Insights` (does not exist)
    - `Generate Personalized Email`
    - `Create Drive Folder`
    - `Share Drive Folder`
    - `Store in Database`
    - `Create Gmail Draft`
  - Tries to read original webhook body from `Typebot Webhook` (does not exist); actual intake node is `On form submission`.
- **Inputs/Outputs:**
  - Input: Send Email Created Confirmation
  - Output: aggregated JSON for Supabase insertion
- **Critical integration issues:**
  - References to **non-existent nodes** (`Request Email Approval`, `Generate Strategy Insights`, `Typebot Webhook`) mean the aggregation will not capture approval/lead data as intended. It will still produce output, but with nulls/defaults and possibly missing context.
- **Edge cases / failures:**
  - `$()` calls can throw if node/item context isn’t present; the code wraps many accesses in try/catch, reducing hard failures.
  - Error detection by searching for `.message.includes('error')` is heuristic and may miss real errors or create false positives.

#### Node: **Store in Database**
- **Type / Role:** `Supabase` (insert) — stores consolidated lead record for approved leads.
- **Configuration:**
  - Table: `leads_consolidated`
  - Fields include:
    - lead name, email, loan_type, income, residency_status
    - `folder_link`: `https://drive.google.com/drive/u/0/folders/{{ Create Drive Folder.id }}`
    - `upload_form_link`: points to `https://n8n.io/form/...` (note: differs from Gmail draft host `13.211.164.241.nip.io`)
    - `email_approved`: hard-coded “Approved”
    - `lead_activity_error_details`: from aggregator `$json.lead_activity_error_details`
    - `created_at`: `$now`
- **Inputs/Outputs:** Input: Aggregate Log Data; Output unused.
- **Edge cases / failures:**
  - Host inconsistency for upload form link (may confuse recipients or break functionality if only one host is valid).
  - Supabase RLS, schema mismatches, timestamp formatting.

---

### 2.5 Document Upload Portal + Validation
**Overview:**  
Provides a dedicated upload form that receives 3 required files plus hidden identifiers linking the submission to the correct Drive folder and client. Validates that all inputs are present and that exactly 3 files were uploaded.

**Nodes Involved:**
- Upload portal trigger
- Validate Input
- Send Validation Error

#### Node: **Upload portal trigger**
- **Type / Role:** `Form Trigger` — entry point for document upload.
- **Configuration:**
  - Form title: “Document Upload Portal”
  - Description: “Please upload your documents here. All fields are required.”
  - Required file fields: ID, Payslip, Bank_Statement
  - Hidden fields: `folder_id`, `client_name` (populated via query parameters from the email link)
- **Inputs/Outputs:** Outputs item including `formQueryParameters` and binary file fields.
- **Edge cases / failures:**
  - If user opens form without query parameters, hidden fields may be empty.
  - Field labels vs binary keys: downstream expects binary keys `ID`, `Payslip`, and `Bank_Statement`.

#### Node: **Validate Input**
- **Type / Role:** `IF` — validates upload submission.
- **Configuration:**
  - `folder_id` not empty
  - `client_name` not empty
  - `Object.keys($binary).length == 3`
- **Inputs/Outputs:**
  - True → Upload ID1
  - False → Send Validation Error
- **Edge cases / failures:**
  - Extra binary fields (e.g., if form adds metadata) will fail count check.
  - Missing one file fails count check (intended).
  - If binary field names differ from expectations, count might still be 3 but uploads later fail.

#### Node: **Send Validation Error**
- **Type / Role:** `Telegram` — alerts broker of validation failure.
- **Configuration:** Sends time and likely reasons; chatId hard-coded.
- **Edge cases:** Telegram delivery/auth issues.

---

### 2.6 Drive Uploads + Results Evaluation + Notifications + Supabase Update
**Overview:**  
Uploads the three documents to the specified Drive folder while allowing partial failure, evaluates results, notifies broker on success/partial failure, and updates Supabase with upload error details.

**Nodes Involved:**
- Upload ID1
- Upload Payslip1
- Upload Bank Statement1
- Evaluate upload results
- Check all uploads successful
- Send Success Notification
- Send Partial Failure Notification
- Update upload error log

#### Node: **Upload ID1**
- **Type / Role:** `Google Drive` (upload) — uploads ID file.
- **Configuration:**
  - File name from binary: `Upload portal trigger.binary.ID.fileName`
  - Folder ID: `Upload portal trigger.json.folder_id`
  - `continueOnFail: true` (important for partial failure handling)
- **Inputs/Outputs:** Input: validated submission; Output: upload metadata or `{ error: ... }` when failed.
- **Edge cases / failures:**
  - Missing binary key `ID` (form field rename) causes expression failure.
  - Drive permission issues on folder.
  - Large files/timeouts.

#### Node: **Upload Payslip1**
- **Type / Role:** `Google Drive` (upload) — uploads payslip.
- **Configuration:** Similar; uses binary key `Payslip`; `continueOnFail: true`.
- **Connection:** Runs after Upload ID1.
- **Edge cases:** Same as above.

#### Node: **Upload Bank Statement1**
- **Type / Role:** `Google Drive` (upload) — uploads bank statement.
- **Configuration:** Uses binary key `Bank_Statement`; `continueOnFail: true`.
- **Connection:** Runs after Upload Payslip1.

#### Node: **Evaluate upload results**
- **Type / Role:** `Code` — aggregates the 3 upload node results.
- **Configuration choices (interpreted):**
  - Reads `$input.all()` (expects the three upload results)
  - Determines doc type by index: 0=ID, 1=Payslip, 2=Bank_Statement
  - If `upload.json.error` exists → mark failed; else mark successful with `id` and `webViewLink`
  - Produces:
    - `all_success` boolean
    - `document_upload_error_details` JSON string or null
    - counts and arrays of successful/failed uploads
- **Inputs/Outputs:**
  - Input: output chain from Upload Bank Statement1 (and prior nodes as items)
  - Output: one summary item used by IF + Supabase update
- **Edge cases / failures:**
  - If upstream outputs aren’t all present as separate items, index-based mapping can mislabel documents.
  - Google Drive node error shape may differ; may not always populate `json.error`.

#### Node: **Check all uploads successful**
- **Type / Role:** `IF` — branches on `all_success`.
- **Configuration:** `all_success == true`
- **Outputs:**
  - True → Send Success Notification
  - False → Send Partial Failure Notification

#### Node: **Send Success Notification**
- **Type / Role:** `Telegram` — informs broker all documents uploaded.
- **Configuration:**
  - References `Upload portal trigger.item.json.formQueryParameters.client_name`
  - Drive folder link uses `Upload portal trigger.item.json.folder_id`
- **Edge cases:**
  - If `formQueryParameters` not present (depending on n8n form trigger behavior), client name may be blank.

#### Node: **Send Partial Failure Notification**
- **Type / Role:** `Telegram` — provides detailed failure report.
- **Configuration:**
  - Uses `$json.*` from Evaluate upload results for counts and lists.
  - References `$json.folder_link` in message, but **Evaluate upload results does not output `folder_link`**; this link will render blank unless added elsewhere.
- **Edge cases / failures:**
  - Message formatting assumes `failed_uploads` includes `name` and `filename`, but Evaluate upload results uses `{document, error}` (no `name`/`filename`). This will produce “undefined” fields in the Telegram message unless corrected.

#### Node: **Update upload error log**
- **Type / Role:** `Supabase` (update) — writes document upload errors back to the lead record.
- **Configuration:**
  - Table: `leads_consolidated`
  - Filter: `lead_first_name == client_name` (from upload trigger)
  - Updates: `document_upload_error_details`
- **Edge cases / failures:**
  - Matching by `lead_first_name` is not reliable (duplicates). Prefer a unique lead ID or folder_id/email.
  - If no row matches, update will affect 0 rows (silent unless monitored).
  - RLS policies may block update.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| On form submission | Form Trigger | Capture new mortgage inquiry | — | Normalize lead name | Captures new mortgage inquiries and cleans name data. / Applies income threshold before continuing. / Prevents low quality leads from triggering AI and admin steps. |
| Normalize lead name | Code | Parse full name into first/last; normalize casing | On form submission | Check income threshold | Captures new mortgage inquiries and cleans name data. / Applies income threshold before continuing. / Prevents low quality leads from triggering AI and admin steps. |
| Check income threshold | IF | Qualify lead by income | Normalize lead name | Request Email Approval1 (true) | Captures new mortgage inquiries and cleans name data. / Applies income threshold before continuing. / Prevents low quality leads from triggering AI and admin steps. |
| Request Email Approval1 | Telegram (sendAndWait) | Human approval gate | Check income threshold (true) | Check approval response | Sends high value leads to Telegram for review. / Broker must approve before AI email drafting. / Declined leads are logged in Supabase. |
| Check approval response | IF | Branch on approval | Request Email Approval1 | Generate Personalized Email (true); Log declined lead (false) | Sends high value leads to Telegram for review. / Broker must approve before AI email drafting. / Declined leads are logged in Supabase. |
| Log declined lead | Supabase | Insert declined lead record | Check approval response (false) | — | Sends high value leads to Telegram for review. / Broker must approve before AI email drafting. / Declined leads are logged in Supabase. |
| Generate Personalized Email | Google Gemini (LangChain) | Generate email body | Check approval response (true) | Create Drive Folder | Generates personalized mortgage email using Gemini. / Creates client specific Drive folder. / Prepares Gmail draft with secure upload link. |
| Create Drive Folder | Google Drive | Create per-lead folder | Generate Personalized Email | Share Drive Folder | Generates personalized mortgage email using Gemini. / Creates client specific Drive folder. / Prepares Gmail draft with secure upload link. |
| Share Drive Folder | Google Drive | Share folder permissions | Create Drive Folder | Create Gmail Draft | Generates personalized mortgage email using Gemini. / Creates client specific Drive folder. / Prepares Gmail draft with secure upload link. |
| Create Gmail Draft | Gmail | Draft outreach email with upload link | Share Drive Folder | Send Email Created Confirmation | Generates personalized mortgage email using Gemini. / Creates client specific Drive folder. / Prepares Gmail draft with secure upload link. |
| Send Email Created Confirmation | Telegram | Notify broker draft is ready | Create Gmail Draft | Aggregate Log Data | Sends Telegram notifications for success or failure. / Aggregates errors and activity data. / Stores structured results in Supabase. |
| Aggregate Log Data | Code | Aggregate approval + errors into log payload | Send Email Created Confirmation | Store in Database | Sends Telegram notifications for success or failure. / Aggregates errors and activity data. / Stores structured results in Supabase. |
| Store in Database | Supabase | Insert approved lead record + links + activity log | Aggregate Log Data | — | Sends Telegram notifications for success or failure. / Aggregates errors and activity data. / Stores structured results in Supabase. |
| Upload portal trigger | Form Trigger | Collect 3 required docs + hidden identifiers | — | Validate Input | Client uploads required documents through secure form. / Hidden fields link uploads to correct Drive folder. / Input validation ensures required data is present. |
| Validate Input | IF | Validate folder_id/client_name and file count | Upload portal trigger | Upload ID1 (true); Send Validation Error (false) | Client uploads required documents through secure form. / Hidden fields link uploads to correct Drive folder. / Input validation ensures required data is present. |
| Send Validation Error | Telegram | Notify broker validation failed | Validate Input (false) | — | Client uploads required documents through secure form. / Hidden fields link uploads to correct Drive folder. / Input validation ensures required data is present. |
| Upload ID1 | Google Drive | Upload ID file (continue on fail) | Validate Input (true) | Upload Payslip1 | Uploads ID, payslip, and bank statement to Drive. / Handles partial failures without breaking workflow. / Evaluates overall upload status. |
| Upload Payslip1 | Google Drive | Upload payslip file (continue on fail) | Upload ID1 | Upload Bank Statement1 | Uploads ID, payslip, and bank statement to Drive. / Handles partial failures without breaking workflow. / Evaluates overall upload status. |
| Upload Bank Statement1 | Google Drive | Upload bank statement file (continue on fail) | Upload Payslip1 | Evaluate upload results | Uploads ID, payslip, and bank statement to Drive. / Handles partial failures without breaking workflow. / Evaluates overall upload status. |
| Evaluate upload results | Code | Compute upload success/failure summary | Upload Bank Statement1 | Check all uploads successful; Update upload error log | Uploads ID, payslip, and bank statement to Drive. / Handles partial failures without breaking workflow. / Evaluates overall upload status. |
| Check all uploads successful | IF | Branch on all_success | Evaluate upload results | Send Success Notification (true); Send Partial Failure Notification (false) | Sends Telegram notifications for success or failure. / Aggregates errors and activity data. / Stores structured results in Supabase. |
| Send Success Notification | Telegram | Notify broker all docs uploaded | Check all uploads successful (true) | — | Sends Telegram notifications for success or failure. / Aggregates errors and activity data. / Stores structured results in Supabase. |
| Send Partial Failure Notification | Telegram | Notify broker partial failure with details | Check all uploads successful (false) | — | Sends Telegram notifications for success or failure. / Aggregates errors and activity data. / Stores structured results in Supabase. |
| Update upload error log | Supabase | Update lead record with upload error details | Evaluate upload results | — | Sends Telegram notifications for success or failure. / Aggregates errors and activity data. / Stores structured results in Supabase. |

---

## 4. Reproducing the Workflow from Scratch

### A) Lead qualification + approved outreach flow

1. **Create a Form Trigger** node named **“On form submission”**
   - Form title: “Provide Your details”
   - Add required fields:
     - `name` (text)
     - `email` (email)
     - `loan_type` (radio): Home Loan, Commercial, Refinance
     - `income` (number)
     - `residency_status` (radio): Yes, No

2. **Add a Code** node named **“Normalize lead name”** connected from the form trigger
   - Implement:
     - Title-case the name
     - Split into `first_name`, `last_name`
     - Add `full_name_clean`
   - Ensure it returns a single item with merged original fields.

3. **Add an IF** node named **“Check income threshold”**
   - Condition: `{{$json.income}} > 80000`
   - True output continues; False output can be left unconnected (or optionally add logging later).

4. **Add a Telegram node** named **“Request Email Approval1”**
   - Operation: **sendAndWait**
   - Chat ID: set your broker/admin chat ID
   - Message: include lead details (name/email/income/loan/residency)
   - Approval options: set approval type to a double confirmation (as in the workflow)
   - **Credentials:** Configure Telegram Bot API credential (bot token) and ensure the bot can message the chat.

5. **Add an IF** node named **“Check approval response”**
   - Condition: `{{$json.data.approved}} == true`
   - True → approved path; False → declined path

6. **Declined path:** Add a Supabase node named **“Log declined lead”**
   - Operation: Insert row into table `leads_consolidated`
   - Map fields from normalized lead:
     - `lead_first_name`, `lead_last_name`, `email`, `loan_type`, `income`, `residency_status`
     - `created_at` = `{{$now}}`
     - `email_approved` = `Declined`
   - **Credentials:** Supabase API credential (URL + API key/service role as appropriate). Ensure RLS allows insert.

7. **Approved path:** Add a Gemini node named **“Generate Personalized Email”**
   - Node type: Google Gemini (LangChain)
   - Model: `models/gemini-2.5-flash-lite`
   - Temperature: `0.4`
   - Prompt: generate a 3–4 sentence professional email body; interpolate `first_name`, `loan_type`, `income`; return body only.
   - **Credentials:** Google Gemini/PaLM API key credential.

8. **Add a Google Drive node** named **“Create Drive Folder”**
   - Operation: Create → Folder
   - Parent folder: choose your broker parent folder (e.g., “Lendium Files”)
   - Name: `{{$('Normalize lead name').item.json.first_name}} ({{$('Normalize lead name').item.json.loan_type}})`
   - **Credentials:** Google Drive OAuth2

9. **Add a Google Drive node** named **“Share Drive Folder”**
   - Operation: Share folder
   - Folder ID should be: `{{$node["Create Drive Folder"].json.id}}` (recommended to avoid sharing the wrong folder)
   - Permission: set according to policy (the provided workflow uses `anyone` + `writer`)
   - **Credentials:** Google Drive OAuth2

10. **Add a Gmail node** named **“Create Gmail Draft”**
   - Resource: Draft
   - Email type: HTML
   - To: `{{$('Normalize lead name').item.json.email}}`
   - Subject: `Re: Your {{$('Normalize lead name').item.json.loan_type}} Inquiry - Lendium Mortgage Services`
   - Body:
     - Insert Gemini text from the Gemini output (ensure you reference the correct output field for your Gemini node version)
     - Add an HTML link/button to your upload form URL, passing:
       - `folder_id={{Create Drive Folder.id}}`
       - `client_name={{first_name}}`
   - **Credentials:** Gmail OAuth2 (ensure draft scope)

11. **Add a Telegram node** named **“Send Email Created Confirmation”**
   - Send broker a message including:
     - Lead first name
     - Email body (Gemini output)
     - Drive folder link: `https://drive.google.com/drive/u/0/folders/{{Create Drive Folder.id}}`

12. **Add a Code node** named **“Aggregate Log Data”**
   - Build a JSON object with:
     - approval status
     - workflow error details (if any)
     - lead identifiers
   - **Important:** Reference the actual node names you used (e.g., `Request Email Approval1`, `On form submission`). Avoid references to non-existent nodes.

13. **Add a Supabase node** named **“Store in Database”**
   - Insert into `leads_consolidated`
   - Store:
     - Lead fields
     - `folder_link`
     - `upload_form_link` (use the same host you email to clients)
     - `email_approved` = “Approved”
     - `lead_activity_error_details` from Aggregate Log Data
     - `created_at` = `{{$now}}`

---

### B) Document upload flow

14. **Create a second Form Trigger** node named **“Upload portal trigger”**
   - Form title: “Document Upload Portal”
   - Description: “Please upload your documents here. All fields are required.”
   - Required file fields: ID, Payslip, Bank_Statement
   - Hidden fields: `folder_id`, `client_name`
   - Ensure your emailed link passes those as query parameters.

15. **Add an IF node** named **“Validate Input”**
   - Conditions:
     - `{{$json.folder_id}}` not empty
     - `{{$json.client_name}}` not empty
     - `Object.keys($binary).length == 3`

16. **On validation failure:** Add Telegram node **“Send Validation Error”**
   - Notify broker with timestamp and troubleshooting hints.

17. **On validation success:** Add Google Drive upload nodes in sequence (all with `continueOnFail: true`)
   - **Upload ID1**
     - Folder ID: `{{$('Upload portal trigger').item.json.folder_id}}`
     - Binary property: `ID`
   - **Upload Payslip1**
     - Binary property: `Payslip`
   - **Upload Bank Statement1**
     - Binary property: `Bank_Statement`

18. **Add a Code node** named **“Evaluate upload results”**
   - Read `$input.all()`
   - Produce:
     - `all_success`
     - `document_upload_error_details` (JSON string or null)
     - lists and counts of successful/failed

19. **Add an IF node** named **“Check all uploads successful”**
   - Condition: `{{$json.all_success}} == true`

20. **Success path:** Telegram node **“Send Success Notification”**
   - Include client name and Drive folder link.

21. **Failure path:** Telegram node **“Send Partial Failure Notification”**
   - Ensure your message uses the actual structure produced by “Evaluate upload results” (document names, errors).
   - Include folder link by explicitly adding it in Evaluate upload results or reconstructing from `folder_id`.

22. **Add Supabase node** named **“Update upload error log”**
   - Operation: update `leads_consolidated`
   - Prefer matching by a stable unique key (email or folder_id). If using `lead_first_name`, accept collision risk.
   - Update `document_upload_error_details`.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Mortgage Lead Automation with AI and Human Approval” and the described 6-step flow and setup checklist | Sticky note covering overall workflow concept and required setup (Gmail/Drive/Telegram/Supabase/Gemini, chat ID, Drive parent folder ID, income threshold testing) |
| “Captures new mortgage inquiries and cleans name data… Prevents low quality leads…” | Sticky note for lead capture + income threshold block |
| “Sends high value leads to Telegram for review… Declined leads are logged…” | Sticky note for approval + decline logging block |
| “Generates personalized mortgage email using Gemini… secure upload link.” | Sticky note for Gemini + Drive folder + Gmail draft block |
| “Client uploads required documents through secure form… Input validation…” | Sticky note for upload portal + validation block |
| “Uploads ID, payslip, and bank statement to Drive… Handles partial failures…” | Sticky note for Drive upload + evaluation block |
| “Sends Telegram notifications… Aggregates errors… Stores structured results in Supabase.” | Sticky note for notification + logging + Supabase update block |

