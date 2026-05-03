Send automated invoice payment reminders with Gmail, Google Sheets and Twilio

https://n8nworkflows.xyz/workflows/send-automated-invoice-payment-reminders-with-gmail--google-sheets-and-twilio-11915


# Send automated invoice payment reminders with Gmail, Google Sheets and Twilio

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Send automated invoice payment reminders with Gmail, Google Sheets and Twilio  
**Purpose:** Automate invoice follow-ups after an invoice is submitted through an n8n Form. The workflow waits fixed intervals (7 days, then +2 days, then +3 days) and—before each reminder—re-checks a Google Sheet to avoid messaging customers who have already paid. Reminders can be sent by **Gmail** (emails) and optionally **Twilio** (SMS nodes are present but disabled).

### 1.1 Input Reception (Invoice submission)
- Captures invoice/contact details via an n8n Form trigger.
- Downstream logic uses the submitted **Email** as the identifier to find/check invoice status in Google Sheets.

### 1.2 Follow-up Cycle 1 (Day 7 check + first reminder)
- Waits 7 days.
- Checks Google Sheet record(s).
- If still unpaid (per IF conditions), sends first reminder email (and optional SMS).

### 1.3 Follow-up Cycle 2 (Day 9 check + second reminder)
- Waits an additional 2 days (total ~9 days from submission).
- Re-checks Google Sheet.
- If still unpaid, sends second reminder email (and optional SMS).

### 1.4 Follow-up Cycle 3 (Day 12 check + final reminder)
- Waits an additional 3 days (total ~12 days from submission).
- Re-checks Google Sheet.
- If still unpaid, sends final reminder email (and optional SMS).
- Otherwise, ends.

### 1.5 Termination / Exit
- A NoOp node marks completion for paths where reminders should not be sent (e.g., already paid) and also receives the “unpaid” branch (as currently wired) for one IF node.

---

## 2. Block-by-Block Analysis

### Block A — Input Reception (Invoice creation event)
**Overview:** Collects invoice information via an n8n Form and starts the reminder schedule.  
**Nodes involved:**  
- Invoice Form Submission Trigger

#### Node: Invoice Form Submission Trigger
- **Type / role:** `Form Trigger` — entry point; generates a new workflow execution per form submission.
- **Configuration (interpreted):**
  - Form title: **Invoice**
  - Description: “Enter Details for invoice”
  - Fields (notable): First Name (required), Email (required, email type), Phone (required), Payment Link (required), Name Of Bank (required), etc.
- **Key variables used later:**
  - `$('Invoice Form Submission Trigger').item.json.Email` is referenced by later nodes as the recipient email and for sheet matching.
- **Connections:**
  - **Output →** Wait 7 Days for First Follow-up
- **Version-specific notes:** Node typeVersion **2.2** (form trigger behavior and UI fields depend on n8n version; ensure your n8n supports Form Trigger v2+).
- **Edge cases / failures:**
  - Missing/invalid email: the form enforces required email, but if modified, downstream Gmail nodes may fail validation.
  - Phone required but Twilio SMS nodes are disabled; if enabled later, ensure phone is stored and formatted to E.164.

---

### Block B — First Follow-up Cycle (Day 7)
**Overview:** After 7 days, reads Google Sheets to confirm payment status and sends the first reminder if unpaid.  
**Nodes involved:**  
- Wait 7 Days for First Follow-up  
- Check Payment Status in Sheet - Day 7  
- If Still Unpaid - First Check"  
- Send First Payment Reminder  
- Send First SMS Reminder (Optional) *(disabled)*

#### Node: Wait 7 Days for First Follow-up
- **Type / role:** `Wait` — delays execution by a fixed duration.
- **Configuration:** 7 days.
- **Connections:**
  - **Input ←** Invoice Form Submission Trigger
  - **Output →** Check Payment Status in Sheet - Day 7
- **Version-specific notes:** typeVersion **1.1** (standard wait functionality).
- **Edge cases / failures:**
  - If n8n is restarted and using non-persistent execution data without proper settings, delayed executions may be lost depending on your hosting mode/config.

#### Node: Check Payment Status in Sheet - Day 7
- **Type / role:** `Google Sheets` — reads sheet data to validate whether the invoice is still unpaid.
- **Configuration (as provided):**
  - Document and Sheet are placeholders: `=[SELECT YOUR SHEET]` / `[SELECT YOUR SHEET]`
  - Operation is not explicitly visible in the JSON excerpt (likely a “Read” operation).
- **Connections:**
  - **Input ←** Wait 7 Days for First Follow-up
  - **Output →** If Still Unpaid - First Check"
- **Credentials:** Google Sheets OAuth2 (`CTK Sheets 12/9/25`)
- **Version-specific notes:** typeVersion **4.6**
- **Edge cases / failures:**
  - Spreadsheet not selected → node fails at runtime.
  - Permissions/auth issues → OAuth token expired or missing access.
  - Returned rows: if multiple rows match the same email, the IF node will evaluate per item; you may send multiple reminders unless you limit results or filter uniquely.

#### Node: If Still Unpaid - First Check"
- **Type / role:** `IF` — decides whether to send reminder or end.
- **Configuration (interpreted):**
  - Combines conditions with **AND**
  - Condition 1 (string equals): submitted email equals sheet column `EMAIL `  
    - Left: `$('Invoice Form Submission Trigger').item.json.Email`
    - Right: `{{$json['EMAIL ']}}`
    - **Important:** the sheet key is `EMAIL ` (with a trailing space). This must match the column header exactly as returned by the Sheets node.
  - Condition 2 (date afterOrEquals): `{{$json.DATE}}` after or equals `{{$today.minus({Days: 9}).format("yyyy-MM-dd")}}`
    - This compares a row date to “today minus 9 days”.
    - **Important:** This logic is inconsistent with a “Day 7” check (it uses 9 days). It may cause invoices newer than 9 days to fail the condition and thus stop reminders.
- **Connections (n8n IF semantics):**
  - Output **true** (condition met) → **Process Complete**
  - Output **false** → **Send First Payment Reminder**
  - **This wiring appears reversed** for the stated intent. Normally “still unpaid” should go to sending, not “Process Complete”.
- **Version-specific notes:** typeVersion **2.2**
- **Edge cases / failures:**
  - Date parsing: if `DATE` in the sheet is not ISO-like or is stored as a formatted string, comparisons may behave unexpectedly.
  - Column mismatch: sticky note says columns `DATE`, `EMAIL`, `Payment Status`, but IF references `EMAIL ` (with trailing space) and does not check `Payment Status` at all.
  - If the Sheets node returns no items, the IF node won’t run on the expected record; behavior depends on n8n item flow (may result in no emails sent).

#### Node: Send First Payment Reminder
- **Type / role:** `Gmail` — sends the first reminder email.
- **Configuration:**
  - To: `={{ $('Invoice Form Submission Trigger').item.json.Email }}`
  - Subject: `(Company Name) Invoice Due`
  - Message: `=First Message` (a placeholder; not dynamically built here)
  - Option: `appendAttribution: false`
  - `executeOnce: true` (only runs once in manual executions; in production each execution still sends once)
  - `alwaysOutputData: true` to pass data forward even if Gmail doesn’t return much.
- **Connections:**
  - **Input ←** If Still Unpaid - First Check" (false branch as wired)
  - **Output →** Send First SMS Reminder (Optional)
- **Credentials:** Gmail OAuth2 (`CTKTruck Gmail 12/9/25`)
- **Version-specific notes:** typeVersion **2.1**
- **Edge cases / failures:**
  - Gmail API auth scopes insufficient, token revoked, or daily sending limits.
  - Placeholder body `First Message` may be too minimal; consider including invoice details from form fields.

#### Node: Send First SMS Reminder (Optional)
- **Type / role:** `Twilio` — sends SMS reminder (currently disabled).
- **Configuration:** message placeholder `[ADD YOUR MESSAGE]`
- **Connections:**
  - **Input ←** Send First Payment Reminder
  - **Output →** Wait 2 Days for Second Follow-up
- **Credentials:** Not shown (must be configured if enabled).
- **Edge cases / failures (if enabled):**
  - Missing “To” phone number configuration (not set in parameters).
  - Phone formatting; Twilio requires E.164.
  - SMS deliverability/geo restrictions.

---

### Block C — Second Follow-up Cycle (Day 9)
**Overview:** Two days after the first reminder, checks the sheet again and sends a stronger reminder if unpaid.  
**Nodes involved:**  
- Wait 2 Days for Second Follow-up  
- Check Payment Status in Sheet - Day 9  
- If Still Unpaid - Second Check  
- Send Second Payment Reminder Email  
- Send Second SMS Reminder (Optional) *(disabled)*

#### Node: Wait 2 Days for Second Follow-up
- **Type / role:** `Wait` — delays the flow by 2 days.
- **Connections:**
  - **Input ←** Send First SMS Reminder (Optional)
  - **Output →** Check Payment Status in Sheet - Day 9
- **Edge cases:** same persistence considerations as other Wait nodes.

#### Node: Check Payment Status in Sheet - Day 9
- **Type / role:** `Google Sheets` — re-reads payment status.
- **Configuration:** placeholders for documentId/sheetName again.
- **Connections:**
  - **Input ←** Wait 2 Days for Second Follow-up
  - **Output →** If Still Unpaid - Second Check
- **Credentials:** Google Sheets OAuth2 (`CTK Sheets 12/9/25`)
- **Edge cases:** same as Day 7 sheets node (duplicates, missing sheet selection, auth).

#### Node: If Still Unpaid - Second Check
- **Type / role:** `IF`
- **Configuration (interpreted):**
  - Same two conditions as first check:
    - Email equals `EMAIL ` (trailing space)
    - DATE afterOrEquals today minus 9 days
  - Again, no explicit “Payment Status == unpaid/paid” condition.
- **Connections:**
  - **True →** Process Complete
  - **False →** Send Second Payment Reminder Email
  - Likely reversed relative to the node name.
- **Edge cases:** same as Day 7 IF.

#### Node: Send Second Payment Reminder Email
- **Type / role:** `Gmail`
- **Configuration:**
  - To: submitted Email
  - Subject: `Friendly Reminder: Invoice Still Outstanding with <Name>` (placeholder `<Name>`)
  - Message: `=Follow up message` (placeholder)
  - `appendAttribution: false`
- **Connections:**
  - **Input ←** If Still Unpaid - Second Check (false branch as wired)
  - **Output →** Send Second SMS Reminder (Optional)
- **Credentials:** Gmail OAuth2 (`CTKTruck Gmail 12/9/25`)
- **Edge cases:** same as other Gmail send.

#### Node: Send Second SMS Reminder (Optional)
- **Type / role:** `Twilio` (disabled)
- **Connections:**
  - **Input ←** Send Second Payment Reminder Email
  - **Output →** Wait 3 Days for Final Follow-up

---

### Block D — Final Follow-up Cycle (Day 12)
**Overview:** Three days later, performs a final sheet check and sends a last urgent reminder if still unpaid.  
**Nodes involved:**  
- Wait 3 Days for Final Follow-up  
- Check Payment Status in Sheet - Day 12  
- If Still Unpaid - Third Check  
- Send Final Payment Reminder Email  
- Send Final SMS Reminder (Optional) *(disabled)*  
- Process Complete

#### Node: Wait 3 Days for Final Follow-up
- **Type / role:** `Wait` — delays by 3 days.
- **Connections:**
  - **Input ←** Send Second SMS Reminder (Optional)
  - **Output →** Check Payment Status in Sheet - Day 12

#### Node: Check Payment Status in Sheet - Day 12
- **Type / role:** `Google Sheets`
- **Configuration:** placeholders for documentId/sheetName.
- **Connections:**
  - **Input ←** Wait 3 Days for Final Follow-up
  - **Output →** If Still Unpaid - Third Check
- **Credentials:** Google Sheets OAuth2 (`CTK Sheets 12/9/25`)

#### Node: If Still Unpaid - Third Check
- **Type / role:** `IF`
- **Configuration:** same as other IF nodes (email equals `EMAIL ` + DATE afterOrEquals today minus 9 days).
- **Connections:**
  - **True →** Process Complete
  - **False →** Send Final Payment Reminder Email
  - Again likely reversed for the intended naming.
- **Edge cases:** same as prior IF nodes.

#### Node: Send Final Payment Reminder Email
- **Type / role:** `Gmail`
- **Configuration:**
  - Subject: `Final Reminder: Action Needed — Outstanding Invoice`
  - Message: `=Final Message` (placeholder)
  - To: submitted Email
- **Connections:**
  - **Input ←** If Still Unpaid - Third Check (false branch as wired)
  - **Output →** Send Final SMS Reminder (Optional)
- **Credentials:** Gmail OAuth2 (`CTKTruck Gmail 12/9/25`)

#### Node: Send Final SMS Reminder (Optional)
- **Type / role:** `Twilio` (disabled)
- **Connections:**
  - **Input ←** Send Final Payment Reminder Email
  - **Output:** none (workflow ends after this node in the current design)

#### Node: Process Complete
- **Type / role:** `NoOp` — explicit terminator/marker node.
- **Connections:**
  - **Inputs from:** all three IF nodes (their “true” branches) and also the third IF’s true branch explicitly.
- **Edge cases:** none; used for clarity and end-of-path.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note1 | Sticky Note | Overall workflow description / setup notes | — | — | # Automated Invoice Payment Reminder; Setup steps; — Neal, CTK Industries |
| Invoice Form Submission Trigger | Form Trigger | Entry point: collect invoice data | — | Wait 7 Days for First Follow-up | ## First Follow-up Cycle: Waits 7 days… checks sheet… sends initial reminder… |
| Wait 7 Days for First Follow-up | Wait | Delay 7 days | Invoice Form Submission Trigger | Check Payment Status in Sheet - Day 7 | ## First Follow-up Cycle: Waits 7 days… checks sheet… sends initial reminder… |
| Check Payment Status in Sheet - Day 7 | Google Sheets | Read invoice/payment status from sheet | Wait 7 Days for First Follow-up | If Still Unpaid - First Check" | ## First Follow-up Cycle: Waits 7 days… checks sheet… sends initial reminder… |
| If Still Unpaid - First Check" | IF | Decide send vs stop (first cycle) | Check Payment Status in Sheet - Day 7 | Process Complete; Send First Payment Reminder | ## First Follow-up Cycle: Waits 7 days… checks sheet… sends initial reminder… |
| Send First Payment Reminder | Gmail | Send first reminder email | If Still Unpaid - First Check" | Send First SMS Reminder (Optional) | ## First Follow-up Cycle: Waits 7 days… checks sheet… sends initial reminder… |
| Send First SMS Reminder (Optional) | Twilio | Optional first SMS | Send First Payment Reminder | Wait 2 Days for Second Follow-up | ## First Follow-up Cycle: Waits 7 days… checks sheet… sends initial reminder… |
| Wait 2 Days for Second Follow-up | Wait | Delay 2 days | Send First SMS Reminder (Optional) | Check Payment Status in Sheet - Day 9 | ## Second Follow-up Cycle: Waits an additional 2 days… re-checks… escalates… |
| Check Payment Status in Sheet - Day 9 | Google Sheets | Re-check status | Wait 2 Days for Second Follow-up | If Still Unpaid - Second Check | ## Second Follow-up Cycle: Waits an additional 2 days… re-checks… escalates… |
| If Still Unpaid - Second Check | IF | Decide send vs stop (second cycle) | Check Payment Status in Sheet - Day 9 | Process Complete; Send Second Payment Reminder Email | ## Second Follow-up Cycle: Waits an additional 2 days… re-checks… escalates… |
| Send Second Payment Reminder Email | Gmail | Send second reminder email | If Still Unpaid - Second Check | Send Second SMS Reminder (Optional) | ## Second Follow-up Cycle: Waits an additional 2 days… re-checks… escalates… |
| Send Second SMS Reminder (Optional) | Twilio | Optional second SMS | Send Second Payment Reminder Email | Wait 3 Days for Final Follow-up | ## Second Follow-up Cycle: Waits an additional 2 days… re-checks… escalates… |
| Wait 3 Days for Final Follow-up | Wait | Delay 3 days | Send Second SMS Reminder (Optional) | Check Payment Status in Sheet - Day 12 | ## Final Follow-up Cycle: After 3 more days… final status check… last urgent reminder… |
| Check Payment Status in Sheet - Day 12 | Google Sheets | Final re-check status | Wait 3 Days for Final Follow-up | If Still Unpaid - Third Check | ## Final Follow-up Cycle: After 3 more days… final status check… last urgent reminder… |
| If Still Unpaid - Third Check | IF | Decide send vs stop (final cycle) | Check Payment Status in Sheet - Day 12 | Process Complete; Send Final Payment Reminder Email | ## Final Follow-up Cycle: After 3 more days… final status check… last urgent reminder… |
| Send Final Payment Reminder Email | Gmail | Send final reminder email | If Still Unpaid - Third Check | Send Final SMS Reminder (Optional) | ## Final Follow-up Cycle: After 3 more days… final status check… last urgent reminder… |
| Send Final SMS Reminder (Optional) | Twilio | Optional final SMS | Send Final Payment Reminder Email | — | ## Final Follow-up Cycle: After 3 more days… final status check… last urgent reminder… |
| Sticky Note | Sticky Note | Comment block for first cycle | — | — | ## First Follow-up Cycle… |
| Sticky Note4 | Sticky Note | Comment block for second cycle | — | — | ## Second Follow-up Cycle… |
| Sticky Note5 | Sticky Note | Comment block for final cycle | — | — | ## Final Follow-up Cycle… |
| Process Complete | NoOp | End marker | If Still Unpaid - First Check"; If Still Unpaid - Second Check; If Still Unpaid - Third Check | — | (Covered by Final Follow-up Cycle note visually; if not, leave blank) |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it:  
   **Send automated invoice payment reminders with Gmail, Google Sheets and Twilio**

2. **Add a Form Trigger node**
   - Node type: **Form Trigger**
   - Form title: `Invoice`
   - Description: `Enter Details for invoice`
   - Add fields (at minimum to match downstream references):
     - `First Name` (required)
     - `Last Name` (optional)
     - `Email` (type: email, required)
     - `Phone` (required)
     - `Payment Link` (required)
     - `Invoice Amount` (optional)
     - `Name Of Bank` (required)

3. **Add Wait node (7 days)**
   - Node type: **Wait**
   - Unit: `days`
   - Amount: `7`
   - Connect: **Form Trigger → Wait (7 days)**

4. **Add Google Sheets node (Day 7 check)**
   - Node type: **Google Sheets**
   - Configure credentials: **Google Sheets OAuth2**
   - Select your **Spreadsheet (Document)** and **Sheet**
   - Configure the operation to **read rows** (commonly “Read” or “Get Many”, depending on your n8n version).
   - Ensure your sheet contains headers. The sticky note requires: `DATE`, `EMAIL`, `Payment Status`.
   - Connect: **Wait (7 days) → Sheets (Day 7)**

5. **Add IF node (Day 7 decision)**
   - Node type: **IF**
   - Add conditions (replicating current workflow):
     - String: `$('Invoice Form Submission Trigger').item.json.Email` **equals** `{{$json['EMAIL ']}}`
     - DateTime: `{{$json.DATE}}` **afterOrEquals** `{{$today.minus({Days: 9}).format("yyyy-MM-dd")}}`
   - Connect: **Sheets (Day 7) → IF (Day 7)**
   - Connect IF outputs as in JSON:
     - **True →** NoOp “Process Complete”
     - **False →** Gmail “Send First Payment Reminder”
   - (If you want intended behavior, swap these branches and add a `Payment Status != Paid` condition.)

6. **Add Gmail node (First reminder)**
   - Node type: **Gmail** (Send Email)
   - Credentials: **Gmail OAuth2**
   - To: `={{ $('Invoice Form Submission Trigger').item.json.Email }}`
   - Subject: `(Company Name) Invoice Due`
   - Body: `First Message` (replace with your actual template)
   - Option: disable attribution if desired.
   - Connect: **IF (false branch) → Gmail (First)**

7. **(Optional) Add Twilio node (First SMS)**
   - Node type: **Twilio**
   - Enable credentials: Twilio Account SID/Auth Token
   - Configure **To** using the form `Phone` field (you must add it; current JSON doesn’t show it configured)
   - Set message text.
   - Keep node disabled if not using SMS (matches provided workflow).
   - Connect: **Gmail (First) → Twilio (First SMS)**

8. **Add Wait node (2 days)**
   - Unit: `days`, Amount: `2`
   - Connect: **Twilio (First SMS) → Wait (2 days)**  
     (If Twilio is disabled, consider connecting Gmail directly to this Wait.)

9. **Add Google Sheets node (Day 9 check)**
   - Same spreadsheet/sheet selection as Day 7.
   - Connect: **Wait (2 days) → Sheets (Day 9)**

10. **Add IF node (Day 9 decision)**
   - Same conditions as Day 7 IF (to replicate).
   - Connect: **Sheets (Day 9) → IF (Day 9)**
   - True → Process Complete; False → Gmail (Second reminder)

11. **Add Gmail node (Second reminder)**
   - To: submitted email
   - Subject: `Friendly Reminder: Invoice Still Outstanding with <Name>`
   - Body: `Follow up message`
   - Connect: **IF (Day 9 false) → Gmail (Second)**

12. **(Optional) Add Twilio node (Second SMS)** (disabled by default)
   - Connect: **Gmail (Second) → Twilio (Second)**

13. **Add Wait node (3 days)**
   - Unit: days, Amount: 3
   - Connect: **Twilio (Second) → Wait (3 days)**  
     (Or Gmail → Wait if Twilio disabled.)

14. **Add Google Sheets node (Day 12 check)**
   - Same spreadsheet/sheet selection.
   - Connect: **Wait (3 days) → Sheets (Day 12)**

15. **Add IF node (Day 12 decision)**
   - Same conditions as prior IFs (to replicate).
   - Connect: **Sheets (Day 12) → IF (Day 12)**

16. **Add Gmail node (Final reminder)**
   - Subject: `Final Reminder: Action Needed — Outstanding Invoice`
   - Body: `Final Message`
   - To: submitted email
   - Connect: **IF (Day 12 false) → Gmail (Final)**

17. **(Optional) Add Twilio node (Final SMS)** (disabled)
   - Connect: **Gmail (Final) → Twilio (Final)**

18. **Add NoOp node “Process Complete”**
   - Node type: **NoOp**
   - Connect **IF true branches** (Day 7, Day 9, Day 12) → **Process Complete**

19. **Credentials checklist**
   - Gmail OAuth2 credential with permission to send email.
   - Google Sheets OAuth2 credential with access to the selected spreadsheet.
   - Twilio credential (only if enabling SMS nodes).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| # Automated Invoice Payment Reminder — describes schedule (Day 7, 9, 12), safety re-check, and setup steps (required sheet columns `DATE`, `EMAIL`, `Payment Status`) — “— Neal, CTK Industries” | Embedded in workflow sticky note (overall documentation) |
| Twilio is optional; SMS nodes are present but disabled and require enabling + proper “To” phone mapping. | Embedded in sticky note + node configuration |
| Potential configuration mismatch: IF nodes reference `EMAIL ` (with trailing space) and do not check `Payment Status`; date threshold uses `today - 9 days` even in Day 7 and Day 12 checks. | Derived from node logic; review before production use |