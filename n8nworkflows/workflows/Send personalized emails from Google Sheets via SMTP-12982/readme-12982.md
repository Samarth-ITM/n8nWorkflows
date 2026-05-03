Send personalized emails from Google Sheets via SMTP

https://n8nworkflows.xyz/workflows/send-personalized-emails-from-google-sheets-via-smtp-12982


# Send personalized emails from Google Sheets via SMTP

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Send personalized emails from Google Sheets via SMTP

**Purpose:**  
This workflow runs on a weekly schedule, reads leads from a Google Sheet, filters for “No-Show” contacts who have not yet been emailed, sends a personalized follow-up email via SMTP, waits between sends to reduce risk of rate limiting/spam flags, and then updates the Google Sheet to mark each email as sent to avoid duplicates.

**Target use cases:**
- Follow-up campaigns after events/workshops (e.g., no-shows)
- Simple email outreach driven by a spreadsheet CRM
- Low-volume, controlled SMTP sending with deduplication via a “Mail”/sent flag

### 1.1 Scheduled Start
Triggers the workflow weekly at a specified day/time.

### 1.2 Data Fetch from Google Sheets
Loads all rows (leads) from a specific sheet range.

### 1.3 Iteration / Batch Processing
Processes contacts one at a time using Split In Batches, enabling per-contact branching and controlled pacing.

### 1.4 Eligibility Filtering (No-show + Not emailed)
Keeps only rows where:
- `Showed Up` is exactly `No-Show`
- `Mail` indicates the email was not sent yet (equals `0` after normalization)

### 1.5 Personalization + SMTP Send
Prepares personalized subject/body fields (placeholder replacement) and sends an HTML email via SMTP.

### 1.6 Throttling + Sheet Update (Deduplication)
Waits 10 seconds, then updates the sheet to set `Mail = 1` for the matching `Email`, then loops to the next contact.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduled Start
**Overview:** Starts the workflow automatically on a weekly schedule.  
**Nodes involved:** `Start email automation`

#### Node: Start email automation
- **Type / role:** Schedule Trigger (`n8n-nodes-base.scheduleTrigger`) — entry point.
- **Configuration (interpreted):**
  - Runs every week.
  - Triggers on **day 6** at **15:00** (n8n’s “day of week” mapping can vary by locale/version; confirm in UI whether day 6 is Saturday or Sunday in your instance).
- **Inputs / outputs:**
  - **Output →** `Fetch contacts from Google Sheets`
- **Edge cases / failures:**
  - Timezone mismatch (n8n instance timezone vs expected business timezone).
  - Misinterpreting “triggerAtDay” numeric value.

---

### Block 2 — Fetch Leads from Google Sheets
**Overview:** Reads a worksheet range containing lead/contact rows used downstream for filtering and email sending.  
**Nodes involved:** `Fetch contacts from Google Sheets`

#### Node: Fetch contacts from Google Sheets
- **Type / role:** Google Sheets node (`n8n-nodes-base.googleSheets`) — read rows.
- **Configuration (interpreted):**
  - Reads from Spreadsheet ID: `1x2gLnN8gbhG6upC76dL17K1e-XfK4wUPjeS3xg8Wdgk`
  - Range: `leads!A:Z` (tab named `leads`, columns A through Z)
  - Uses OAuth2 credential: **Google Sheets account**
- **Inputs / outputs:**
  - **Input ←** `Start email automation`
  - **Output →** `Process contacts one by one`
- **Key data expectations:**
  - Column headers referenced later must exist exactly as used:
    - `Email`, `name`, `Showed Up`, `Mail`, optionally `Subject_1`, `Body_1`
- **Edge cases / failures:**
  - Auth/consent issues or expired refresh token.
  - Tab name mismatch (`leads`) or range not present.
  - Header mismatch causing expressions like `$json['Showed Up']` to be `undefined`.

**Sticky note context:**  
“Fetches contacts from Google Sheets. Ensure column names match the workflow configuration.”

---

### Block 3 — One-by-one Processing Loop
**Overview:** Iterates through sheet rows one at a time and loops back after processing to continue with the next lead.  
**Nodes involved:** `Process contacts one by one`

#### Node: Process contacts one by one
- **Type / role:** Split In Batches (`n8n-nodes-base.splitInBatches`) — controls iteration.
- **Configuration (interpreted):**
  - `reset: false` (continues from the current run state within the execution; not resetting mid-run)
  - **Batch size is not explicitly set in the JSON**; in n8n, default is often `1`, but you must confirm/set **Batch Size = 1** in the UI to match the intended behavior.
- **Inputs / outputs:**
  - **Input ←** `Fetch contacts from Google Sheets` (initial feed)
  - **Input ←** `Mark email as sent in sheet` (loop-back to get next batch)
  - **Output (main index 1) →** `Check no-show status` (this workflow uses the “next batch” output)
  - **Output (main index 0)** is unused in connections (common pattern: index 1 = “continue”, index 0 = “done”; confirm in your n8n version’s SplitInBatches behavior).
- **Edge cases / failures:**
  - If batch size > 1, throttling and per-contact checks still run, but dedup/update timing may be less predictable.
  - Miswiring outputs can cause “no processing” or infinite looping.
  - If no items are provided, the loop ends immediately (expected).

**Sticky note context:**  
- “Common first-time issues … Workflow keeps looping → Ensure Process contacts one by one batch size is set to 1”

---

### Block 4 — Filtering: No-Show and Not Already Emailed
**Overview:** Filters contacts to only those marked as no-show, then checks whether an email was already sent (via the `Mail` flag). Non-eligible rows loop back to process the next contact.  
**Nodes involved:** `Check no-show status`, `Check if email already sent`

#### Node: Check no-show status
- **Type / role:** IF (`n8n-nodes-base.if`) — first eligibility gate.
- **Configuration (interpreted):**
  - Condition: `$json['Showed Up']` equals `"No-Show"` (string match).
- **Inputs / outputs:**
  - **Input ←** `Process contacts one by one`
  - **True →** `Check if email already sent`
  - **False →** `Process contacts one by one` (skip and move to next row)
- **Edge cases / failures:**
  - Value mismatch due to casing/whitespace (e.g., `No Show`, `no-show`, `No-Show `).
  - Missing `Showed Up` column results in `undefined` and condition fails (routes to False branch, silently skipping).

#### Node: Check if email already sent
- **Type / role:** IF (`n8n-nodes-base.if`) — deduplication gate.
- **Configuration (interpreted):**
  - Uses condition builder (v2).
  - Left value expression normalizes missing values:  
    `={{ ($json.Mail || "0").toString().trim() }}`
  - Checks equality to `"0"` (right value set as `=0` in the node; functionally a string comparison to `0` in this config).
  - Meaning: **Only proceed if Mail is unset/empty or explicitly 0.**
- **Inputs / outputs:**
  - **Input ←** `Check no-show status` (True branch)
  - **True →** `Prepare email content`
  - **False →** `Process contacts one by one` (skip already-emailed rows)
- **Edge cases / failures:**
  - If the sheet uses `"Yes"`, `"1"`, `TRUE`, etc., this condition will treat them as already sent only if they are not normalized to `"0"`. Adjust accordingly.
  - If `Mail` contains numeric 0 but parsed differently, normalization to string makes it robust.

**Sticky note context:**  
“Filters contacts based on no-show or custom status and skips already emailed leads.”

---

### Block 5 — Personalization and Email Send via SMTP
**Overview:** Builds per-contact email fields (subject/body/email/name) and sends an HTML email via an SMTP credential.  
**Nodes involved:** `Prepare email content`, `Send email via SMTP`

#### Node: Prepare email content
- **Type / role:** Set (`n8n-nodes-base.set`) — constructs normalized fields used for sending.
- **Configuration (interpreted):**
  - Creates/overwrites fields:
    - `Subject` = `Subject_1` with `{{name}}` placeholder replaced by contact `name` (case-insensitive, flexible spacing). Falls back to empty string if `Subject_1` missing.
      - Expression: `($json.Subject_1 || '').replace(/\{\{\s*name\s*\}\}/gi, $json.name || 'there')`
    - `Body` = `Body_1` with the same placeholder replacement and fallback.
    - `Email` = `$json.Email`
    - `name` = `$json.name`
- **Inputs / outputs:**
  - **Input ←** `Check if email already sent` (True branch)
  - **Output →** `Send email via SMTP`
- **Important note:**  
  Although `Subject` and `Body` are prepared here, the SMTP node in this workflow uses its own hardcoded subject and HTML body expressions, not the `Subject`/`Body` fields. If you intend to use sheet-driven templates, you must wire the SMTP node to use `$json.Subject` and `$json.Body`.
- **Edge cases / failures:**
  - Missing `Email` or invalid email format will later fail SMTP send.
  - Missing `name` results in fallback `"there"` in template replacement, but the SMTP node’s current subject/html uses `$json.name` directly (could become `undefined` in the final email).

#### Node: Send email via SMTP
- **Type / role:** Send Email (`n8n-nodes-base.emailSend`) — sends outgoing email.
- **Configuration (interpreted):**
  - Uses SMTP credential: **SMTP account**
  - `fromEmail`: `Shivam Gandhi <user@example.com>` (must typically match SMTP domain/policy)
  - `toEmail`: `={{ $json.Email }}`
  - `subject`: `={{ 'Hi ' + $json.name + ', quick follow-up after the workshop' }}`
  - `html`: a composed HTML string including name greeting, follow-up copy, and social links.
  - `appendAttribution: false`
- **Inputs / outputs:**
  - **Input ←** `Prepare email content`
  - **Output →** `Delay between emails`
- **Edge cases / failures:**
  - SMTP auth failures, blocked “From” identity, or SPF/DKIM/DMARC alignment issues.
  - Provider rate limits / throttling (mitigated partly by Wait node).
  - HTML link formatting: one anchor tag is missing a starting quote around `href` (`<a href=https://...">`), which can break rendering in some clients. Prefer `href="https://..."`.
  - If `$json.name` is empty/undefined, subject becomes `Hi , ...` or `Hi undefined, ...`.

**Sticky note context:**  
“Prepares personalized email content and sends emails using an SMTP provider.”

---

### Block 6 — Throttling and Marking as Sent (Sheet Update)
**Overview:** Waits between emails, then updates the sheet to mark the contact as emailed (Mail = 1), and loops to the next record.  
**Nodes involved:** `Delay between emails`, `Mark email as sent in sheet`

#### Node: Delay between emails
- **Type / role:** Wait (`n8n-nodes-base.wait`) — pacing/throttling.
- **Configuration (interpreted):**
  - Waits `10` (unit depends on node parameter defaults; typically seconds in n8n Wait node “amount” when no unit is shown—verify in UI).
- **Inputs / outputs:**
  - **Input ←** `Send email via SMTP`
  - **Output →** `Mark email as sent in sheet`
- **Edge cases / failures:**
  - Long waits increase execution time; on some n8n setups, very long waits require queue mode/worker configuration.
  - If the execution is stopped mid-wait, the email may be sent but sheet not updated (causing duplicates next run).

#### Node: Mark email as sent in sheet
- **Type / role:** Google Sheets (`n8n-nodes-base.googleSheets`) — updates an existing row by matching on Email.
- **Configuration (interpreted):**
  - **Operation:** Update
  - **Document:** “Data Management CRM” (same Spreadsheet ID as above)
  - **Sheet/tab:** “Paid Workshop-registeration”
  - **Matching column:** `Email`
  - **Values to set:**
    - `Mail` = `"1"`
    - `Email` = `={{ $('Fetch contacts from Google Sheets').item.json.Email }}`
      - This uses a cross-node reference to the fetched item’s `Email`. In a batching loop, it usually works, but it’s safer to use the current item: `={{ $json.Email }}`.
- **Inputs / outputs:**
  - **Input ←** `Delay between emails`
  - **Output →** `Process contacts one by one` (loop continuation)
- **Edge cases / failures:**
  - If there are duplicate `Email` entries, the update may affect the wrong row or multiple rows depending on Sheets node behavior.
  - If Email is empty, matching fails and nothing is updated → duplicates next run.
  - Sheet/tab mismatch: reading from `leads` tab but updating `Paid Workshop-registeration` tab. This is valid only if both tabs share the same contact rows keyed by Email. If not, updates will not reflect in the source list.

**Sticky note context:**  
“Adds a delay between emails and updates Google Sheets to prevent duplicate sends.”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Start email automation | Schedule Trigger | Weekly trigger / entry point | — | Fetch contacts from Google Sheets | ## Email outreach setup  \n\n## How it works  \n- Reads contacts from Google Sheets  \n- Filters contacts based on no-show or custom status  \n- Skips contacts already emailed  \n- Sends personalized emails via SMTP  \n- Updates the sheet after sending  \n\n## Setup steps  \n- Connect Google Sheets and SMTP credentials  \n- Prepare a sheet with Email, Name, Status, Email Sent columns  \n- Test the workflow  \n- Activate the schedule trigger |
| Fetch contacts from Google Sheets | Google Sheets | Read contacts/leads | Start email automation | Process contacts one by one | Fetches contacts from Google Sheets. Ensure column names match the workflow configuration. |
| Process contacts one by one | Split In Batches | Iterate through leads one-by-one | Fetch contacts from Google Sheets; Mark email as sent in sheet | Check no-show status | ## Common first-time issues  \n\n❌ No emails sent → Check SMTP credentials and From Email address  \n❌ Sheet not updating → Confirm the Mark email as sent in sheet node points to the correct spreadsheet and tab  \n❌ Workflow keeps looping → Ensure Process contacts one by one batch size is set to 1 |
| Check no-show status | IF | Filter: only “No-Show” | Process contacts one by one | Check if email already sent; Process contacts one by one | Filters contacts based on  \nno-show or custom status  \nand skips already emailed leads. |
| Check if email already sent | IF | Dedup gate via `Mail` flag | Check no-show status | Prepare email content; Process contacts one by one | Filters contacts based on  \nno-show or custom status  \nand skips already emailed leads. |
| Prepare email content | Set | Prepare per-contact fields (Subject/Body/Email/name) | Check if email already sent | Send email via SMTP | Prepares personalized email  \ncontent and sends emails  \nusing an SMTP provider. |
| Send email via SMTP | Send Email (SMTP) | Send personalized follow-up email | Prepare email content | Delay between emails | Prepares personalized email  \ncontent and sends emails  \nusing an SMTP provider. |
| Delay between emails | Wait | Throttle sending | Send email via SMTP | Mark email as sent in sheet | Adds a delay between emails  \nand updates Google Sheets  \nto prevent duplicate sends. |
| Mark email as sent in sheet | Google Sheets | Update row: set Mail=1 (sent) | Delay between emails | Process contacts one by one | Adds a delay between emails  \nand updates Google Sheets  \nto prevent duplicate sends. |
| Sticky Note | Sticky Note | Documentation | — | — | ## Email outreach setup  \n\n## How it works  \n- Reads contacts from Google Sheets  \n- Filters contacts based on no-show or custom status  \n- Skips contacts already emailed  \n- Sends personalized emails via SMTP  \n- Updates the sheet after sending  \n\n## Setup steps  \n- Connect Google Sheets and SMTP credentials  \n- Prepare a sheet with Email, Name, Status, Email Sent columns  \n- Test the workflow  \n- Activate the schedule trigger |
| Sticky Note1 | Sticky Note | Documentation | — | — | Fetches contacts from Google Sheets. Ensure column names match the workflow configuration. |
| Sticky Note2 | Sticky Note | Documentation | — | — | Filters contacts based on  \nno-show or custom status  \nand skips already emailed leads. |
| Sticky Note4 | Sticky Note | Documentation | — | — | Prepares personalized email  \ncontent and sends emails  \nusing an SMTP provider. |
| Sticky Note5 | Sticky Note | Documentation | — | — | Adds a delay between emails  \nand updates Google Sheets  \nto prevent duplicate sends. |
| Sticky Note11 | Sticky Note | Documentation / QA checklist | — | — | ## What to test before running the campaign  \n\n✅ Send 2–3 test emails to your own email address  \n✅ Verify personalization works (Name is replaced correctly)  \n✅ Check that emails do not land in the spam folder  \n✅ Confirm the Google Sheet updates Email Sent = Yes after sending  \n✅ Run the workflow again to ensure already-sent contacts are skipped  \n✅ Verify email formatting (line breaks, links, signatures)  \n✅ Test with an empty Name field (use a generic greeting like “Hi there”) |
| Sticky Note12 | Sticky Note | Documentation / Troubleshooting | — | — | ## Common first-time issues  \n\n❌ No emails sent → Check SMTP credentials and From Email address  \n❌ Sheet not updating → Confirm the Mark email as sent in sheet node points to the correct spreadsheet and tab  \n❌ Workflow keeps looping → Ensure Process contacts one by one batch size is set to 1 |
| Sticky Note13 | Sticky Note | Documentation / Provider guidance | — | — | ## 🚫 SMTP provider requirements (important)  \n\nSome consumer email providers restrict or block SMTP usage for automated workflows.  \n\n## ✅ Recommended SMTP providers  \n\nThese services are designed for automated and transactional emails:  \n\t•\tSendGrid (free tier available)  \n\t•\tWebmail  \n\t•\tCustom domain SMTP (cPanel / Plesk hosting)  \n\n## ⚠️ Not recommended for automation  \n\nThese providers may fail or block automated SMTP usage:  \n\t•\tGmail SMTP  \n\t•\tYahoo SMTP  \n\t•\tOutlook / Hotmail SMTP |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add node: “Start email automation”**
   - Type: **Schedule Trigger**
   - Configure a **weekly** schedule:
     - Set the day to match `triggerAtDay: 6` (verify in UI which weekday that is)
     - Set time to **15:00**
   - This is the workflow entry point.

3. **Add node: “Fetch contacts from Google Sheets”**
   - Type: **Google Sheets**
   - Credentials: connect **Google Sheets OAuth2**
   - Operation: **Read / Get many rows** (depends on n8n UI wording for v2)
   - Spreadsheet/Document: select the spreadsheet with ID `1x2gLnN8gbhG6upC76dL17K1e-XfK4wUPjeS3xg8Wdgk`
   - Range: `leads!A:Z`
   - Ensure the sheet has headers at least: `Email`, `name`, `Showed Up`, `Mail` (and optionally `Subject_1`, `Body_1`).

4. **Add node: “Process contacts one by one”**
   - Type: **Split In Batches**
   - Set **Batch Size = 1** (important to avoid looping issues and to throttle properly)
   - Keep `Reset` disabled (or default) to match `reset: false`.

5. **Connect**: `Start email automation` → `Fetch contacts from Google Sheets` → `Process contacts one by one`.

6. **Add node: “Check no-show status”**
   - Type: **IF**
   - Condition (String):
     - Value 1: `={{ $json['Showed Up'] }}`
     - Equals: `No-Show`

7. **Connect**: `Process contacts one by one` → `Check no-show status`.

8. **Add node: “Check if email already sent”**
   - Type: **IF**
   - Condition (String equals):
     - Left value: `={{ ($json.Mail || "0").toString().trim() }}`
     - Right value: `0`
   - Meaning: proceed only when Mail is effectively “0”.

9. **Connect**: `Check no-show status` (True) → `Check if email already sent`.

10. **Add node: “Prepare email content”**
    - Type: **Set**
    - Add fields:
      - `Subject` (String): `={{ ($json.Subject_1 || '').replace(/\{\{\s*name\s*\}\}/gi, $json.name || 'there') }}`
      - `Body` (String): `={{ ($json.Body_1 || '').replace(/\{\{\s*name\s*\}\}/gi, $json.name || 'there') }}`
      - `Email` (String): `={{ $json.Email }}`
      - `name` (String): `={{ $json.name }}`
    - (Optional improvement) Enable “Keep Only Set” depending on whether you want to drop other columns.

11. **Connect**: `Check if email already sent` (True) → `Prepare email content`.

12. **Add node: “Send email via SMTP”**
    - Type: **Send Email**
    - Credentials: configure **SMTP**
      - Host/port, username/password, TLS settings per provider
      - Prefer a provider intended for automated/transactional sends
    - Set:
      - From: `Shivam Gandhi <user@example.com>` (replace with a real, authorized sender)
      - To: `={{ $json.Email }}`
      - Subject: `={{ 'Hi ' + $json.name + ', quick follow-up after the workshop' }}`
      - HTML Body: paste the workflow’s HTML expression (or use `={{ $json.Body }}` if you want sheet-driven body)
    - Disable attribution if desired.

13. **Connect**: `Prepare email content` → `Send email via SMTP`.

14. **Add node: “Delay between emails”**
    - Type: **Wait**
    - Set wait amount to **10** (confirm unit in your UI; commonly seconds).

15. **Connect**: `Send email via SMTP` → `Delay between emails`.

16. **Add node: “Mark email as sent in sheet”**
    - Type: **Google Sheets**
    - Credentials: same **Google Sheets OAuth2**
    - Operation: **Update**
    - Document: the same spreadsheet
    - Sheet/tab: **Paid Workshop-registeration**
    - Matching:
      - Matching column: `Email`
      - Values:
        - `Email`: ideally `={{ $json.Email }}`
        - `Mail`: `1`
    - Ensure the update mapping uses the correct tab that actually contains the row to update.

17. **Connect**: `Delay between emails` → `Mark email as sent in sheet`.

18. **Loop connections (skip paths):**
    - Connect `Check no-show status` (False) → `Process contacts one by one`
    - Connect `Check if email already sent` (False) → `Process contacts one by one`
    - Connect `Mark email as sent in sheet` → `Process contacts one by one`
    - This creates the loop that processes the next contact until exhausted.

19. **Test safely:**
    - Limit the sheet to 2–3 rows or add a temporary filter.
    - Replace recipient emails with your own for initial tests.
    - Verify sheet updates occur after send.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “What to test before running the campaign” checklist (personalization, spam, sheet update, rerun skip, formatting, empty name handling) | Internal QA guidance (Sticky Note11) |
| SMTP provider guidance: recommended (SendGrid, Webmail, custom domain SMTP) and not recommended (Gmail/Yahoo/Outlook SMTP) for automation | Internal ops guidance (Sticky Note13) |
| Common first-time issues: no emails (SMTP/from), sheet not updating (tab/document), looping (batch size = 1) | Troubleshooting (Sticky Note12) |