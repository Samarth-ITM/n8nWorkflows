Send employee leave alerts from Google Sheets via Slack and Gmail

https://n8nworkflows.xyz/workflows/send-employee-leave-alerts-from-google-sheets-via-slack-and-gmail-13521


# Send employee leave alerts from Google Sheets via Slack and Gmail

## 1. Workflow Overview

**Title:** Send employee leave alerts from Google Sheets via Slack and Gmail  
**Workflow name (in JSON):** Transform Your Employee Leave Sheet into a Smart Alert System

This workflow runs daily, reads employee leave rows from a Google Sheet, determines whether each employee’s leave should **activate** (starts/ongoing today but not yet marked Active) or **reset** (ended and still marked Active), then sends notifications via **Slack** and **Gmail**, and finally updates the sheet status to prevent duplicate alerts.

### 1.1 Daily Trigger + Data Ingestion & Normalization
Runs on a schedule, fetches rows from Google Sheets, validates required fields, normalizes dates to midnight, and computes boolean flags used for routing.

### 1.2 Leave Activation Path (Start / In-Progress Leave)
For employees whose leave is currently active (today within start/end) and not yet marked “Active”, it processes them sequentially, sends a Slack message to HR, emails the employee a reminder, and updates the sheet status to “Active”.

### 1.3 Leave Reset Path (Return From Leave)
For employees whose leave has ended and is still marked “Active”, it processes them sequentially, sends a Slack update, emails the employee a welcome-back message, and updates the sheet status back to “Inactive”.

---

## 2. Block-by-Block Analysis

### Block 1.1 — Daily Trigger + Data Ingestion & Normalization
**Overview:** Triggers once per day, reads leave rows, filters invalid rows, and computes activation/reset flags based on today’s date and the sheet “Status”.

**Nodes involved:**
- Daily Leave Check Trigger
- Fetch Leave Records
- Validate & Normalize Leave Data
- Check Needs Activation
- Check Needs Reset

#### Node: Daily Leave Check Trigger
- **Type / role:** Schedule Trigger — entry point; fires on a time-based schedule.
- **Config (interpreted):** Runs every day at **08:00** (server/workflow timezone).
- **Connections:**  
  - Output → **Fetch Leave Records**
- **Edge cases / failures:**
  - Timezone mismatch can cause “today” comparisons to shift a day (especially if sheet dates are in a different timezone than n8n server).

#### Node: Fetch Leave Records
- **Type / role:** Google Sheets — reads rows from a worksheet containing leave data.
- **Config (interpreted):**
  - Document: “Employee Leave Records1” (Google Spreadsheet ID in node)
  - Sheet/tab: “dummy leave data” (gid 273133283)
  - Operation: read (default read/list behavior)
  - **Retry on fail:** enabled
- **Credentials:** Google Sheets OAuth2 (`automations@techdome.ai`)
- **Connections:**  
  - Output → **Validate & Normalize Leave Data**
- **Edge cases / failures:**
  - OAuth token expiry / revoked permissions.
  - Sheet schema changes (renamed columns like “Start Date”) causing downstream code to treat fields as missing.
  - Date cells returned in unexpected formats depending on sheet locale.

#### Node: Validate & Normalize Leave Data
- **Type / role:** Code node — validates required fields, parses/normalizes dates, computes routing flags.
- **Config (interpreted):**
  - Sets `today` to midnight (00:00:00.000).
  - For each row:
    - Reads: `Employee Email`, `Name`, `Start Date`, `End Date`, `Status` (defaults to `"Inactive"` if missing)
    - Parses start/end to Date, normalizes to midnight
    - Drops invalid rows (missing email/start/end or invalid date parse)
    - Computes:
      - `isWithinLeave`: today between start and end inclusive
      - `isLeaveEnded`: today > end
      - `needsActivation`: isWithinLeave AND status != "Active"
      - `needsReset`: isLeaveEnded AND status == "Active"
  - Outputs normalized fields: `email`, `name`, `startDate`, `endDate`, `today`, etc.
- **Connections:**  
  - Output → **Check Needs Activation**
- **Edge cases / failures:**
  - `new Date(startRaw)` is sensitive to input format:
    - ISO (YYYY-MM-DD) is safest; locale formats (DD/MM/YYYY) can parse incorrectly.
  - End date inclusion: leave is considered active **through** endDate (inclusive).
  - `returnDate` is set equal to `endDate` (not next business day); name suggests it might be intended as a “back at work” date.

#### Node: Check Needs Activation
- **Type / role:** IF — routes items that require activation.
- **Config (interpreted):** True branch when `{{$json.needsActivation}}` is true.
- **Connections:**
  - **True output** → **Process Activations Sequentially**
  - **False output** → **Check Needs Reset**
- **Edge cases / failures:**
  - If normalization produced no items, nothing proceeds (expected).
  - Loose type validation enabled in node options; unexpected types could still pass.

#### Node: Check Needs Reset
- **Type / role:** IF — routes items that require reset after activation check fails.
- **Config (interpreted):** True branch when `{{$json.needsReset}}` is true.
- **Connections:**
  - **True output** → **Process Resets Sequentially**
- **Edge cases / failures:**
  - Items that are neither `needsActivation` nor `needsReset` are dropped (no false-branch continuation).

---

### Block 1.2 — Leave Activation Path (Start / In-Progress Leave)
**Overview:** For each employee who should become Active, the workflow processes one item at a time, notifies HR on Slack, emails the employee, and updates the Google Sheet status to Active.

**Nodes involved:**
- Process Activations Sequentially
- Notify HR Channel
- Send OOO Reminder Email
- Mark Leave as Active

#### Node: Process Activations Sequentially
- **Type / role:** Split in Batches — throttling/sequential processing.
- **Config (interpreted):** Default batch behavior (batch size not explicitly set; n8n defaults commonly to 1 unless configured).
- **Connections:**
  - Output (items) → **Notify HR Channel**
  - Output (no items / done) → unused
- **Edge cases / failures:**
  - If batch size is >1, Slack/Gmail/Sheets will run per batch item set; however downstream nodes appear to assume single-item context.
  - If execution stops mid-way, some employees may have been notified but not marked Active.

#### Node: Notify HR Channel
- **Type / role:** Slack — posts activation notice to a channel.
- **Config (interpreted):**
  - Authentication: OAuth2
  - Channel: `alerting-channel` (ID: C09L70CVDL3)
  - Message template uses:
    - `{{$json.name}}`, `{{$json.email}}`, `{{$json.startDate}}`, `{{$json.endDate}}`
  - **Retry on fail:** enabled
- **Connections:**  
  - Output → **Send OOO Reminder Email**
- **Edge cases / failures:**
  - Slack OAuth scopes missing (e.g., `chat:write`).
  - Channel ID invalid or bot not in channel.

#### Node: Send OOO Reminder Email
- **Type / role:** Gmail — sends reminder email to employee.
- **Config (interpreted):**
  - Subject: “Leave Reminder – Please Enable Out of Office”
  - Body uses data from the split-batch context:
    - `{{$('Process Activations Sequentially').item.json.name}}`, dates, etc.
  - **SendTo expression (important):**
    - `{{ "forreferal04@gmail.com" || !$('Process Activations Sequentially').item.json.email }}`
- **Credentials:** Gmail OAuth2
- **Connections:**  
  - Output → **Mark Leave as Active**
- **Edge cases / failures:**
  - **SendTo logic is effectively hardcoded**: in JavaScript, non-empty string is truthy, so it will always send to `forreferal04@gmail.com` and never to the employee.
  - The expression also uses `!email` (boolean negation) instead of `email`, which would yield `true/false` if evaluated.
  - Gmail sending limits / blocked “from” identity / restricted scopes.

#### Node: Mark Leave as Active
- **Type / role:** Google Sheets (Update) — marks record as Active to prevent duplicate activations.
- **Config (interpreted):**
  - Operation: **Update**
  - Match row by: **Employee Email**
  - Sets:
    - `Status` = `"Active"`
    - `Last Updated` = `{{$('Process Activations Sequentially').item.json.today}}`
    - `Employee Email` = current item email (also used for matching)
  - **Retry on fail:** enabled
- **Connections:**  
  - Output → **Process Activations Sequentially** (to continue next batch)
- **Edge cases / failures:**
  - If multiple rows share the same Employee Email, update behavior may be ambiguous (which row is updated).
  - If “Employee Email” differs in sheet due to whitespace/case, matching may fail.
  - Updating a protected sheet/range will fail.

---

### Block 1.3 — Leave Reset Path (Return From Leave)
**Overview:** For employees whose leave ended and is still marked Active, the workflow processes sequentially, posts a Slack update, emails a welcome-back message, and sets status to Inactive.

**Nodes involved:**
- Process Resets Sequentially
- Notify HR Leave Reset
- Send Welcome Back Email
- Mark Leave As Inactive

#### Node: Process Resets Sequentially
- **Type / role:** Split in Batches — sequential processing for resets.
- **Config (interpreted):** Default batch settings.
- **Connections:**
  - Output (items) → **Notify HR Leave Reset**
- **Edge cases / failures:**
  - Same batch-size caveat as activation path.
  - Partial completion can leave sheet status inconsistent with sent notifications.

#### Node: Notify HR Leave Reset
- **Type / role:** Slack — posts leave-completed notice.
- **Config (interpreted):**
  - OAuth2 auth
  - Channel: `alerting-channel` (C09L70CVDL3)
  - Message uses `{{$json.name}}`, `{{$json.email}}`, `{{$json.startDate}}`, `{{$json.endDate}}`
  - **Retry on fail:** enabled
- **Connections:**  
  - Output → **Send Welcome Back Email**
- **Edge cases / failures:** Same Slack scope/channel membership issues as activation.

#### Node: Send Welcome Back Email
- **Type / role:** Gmail — sends return email to employee.
- **Config (interpreted):**
  - Subject: “Welcome Back – Leave Completed”
  - Uses batch context `Process Resets Sequentially` for name/endDate.
  - **SendTo expression (important):**
    - `{{ "forreferal04@gmail.com" || !$('Process Resets Sequentially').item.json.email }}`
- **Connections:**  
  - Output → **Mark Leave As Inactive**
- **Edge cases / failures:**
  - Same issue: will always send to `forreferal04@gmail.com`, and the fallback is also incorrect (`!email`).

#### Node: Mark Leave As Inactive
- **Type / role:** Google Sheets (Update) — resets status to Inactive.
- **Config (interpreted):**
  - Operation: **Update**
  - Match by: **Employee Email**
  - Sets:
    - `Status` = `"Inactive"`
    - `Last Updated` = today from reset batch item
    - `Employee Email` = current item email
  - **Retry on fail:** enabled
- **Connections:**  
  - Output → **Process Resets Sequentially** (continue loop)
- **Edge cases / failures:** Same matching/protection issues as “Mark Leave as Active”.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Holiday Workflow Overview | Sticky Note | Documentation / on-canvas notes | — | — | ## Workflow Overview (includes setup instructions for Google Sheets/Slack/Gmail OAuth2) |
| Daily Leave Check Trigger | Schedule Trigger | Daily entry point | — | Fetch Leave Records | Triggers daily execution, loads configuration values, reads leave records from Google Sheets, validates inputs, and calculates activation and reset conditions. |
| Fetch Leave Records | Google Sheets | Read leave rows | Daily Leave Check Trigger | Validate & Normalize Leave Data | Triggers daily execution, loads configuration values, reads leave records from Google Sheets, validates inputs, and calculates activation and reset conditions. |
| Validate & Normalize Leave Data | Code | Validate/normalize rows and compute flags | Fetch Leave Records | Check Needs Activation | Triggers daily execution, loads configuration values, reads leave records from Google Sheets, validates inputs, and calculates activation and reset conditions. |
| Check Needs Activation | IF | Route activation candidates | Validate & Normalize Leave Data | Process Activations Sequentially; Check Needs Reset | Triggers daily execution, loads configuration values, reads leave records from Google Sheets, validates inputs, and calculates activation and reset conditions. |
| Process Activations Sequentially | Split in Batches | Sequential processing (activation) | Check Needs Activation (true) | Notify HR Channel | Handles employees whose leave starts today. Sends Slack notification, emails reminder, updates sheet status, and prevents duplicate activations. |
| Notify HR Channel | Slack | Post activation message to HR channel | Process Activations Sequentially | Send OOO Reminder Email | Handles employees whose leave starts today. Sends Slack notification, emails reminder, updates sheet status, and prevents duplicate activations. |
| Send OOO Reminder Email | Gmail | Email reminder to employee | Notify HR Channel | Mark Leave as Active | Handles employees whose leave starts today. Sends Slack notification, emails reminder, updates sheet status, and prevents duplicate activations. |
| Mark Leave as Active | Google Sheets | Update sheet: set Status=Active | Send OOO Reminder Email | Process Activations Sequentially | Handles employees whose leave starts today. Sends Slack notification, emails reminder, updates sheet status, and prevents duplicate activations. |
| Check Needs Reset | IF | Route reset candidates | Check Needs Activation (false) | Process Resets Sequentially | Triggers daily execution, loads configuration values, reads leave records from Google Sheets, validates inputs, and calculates activation and reset conditions. |
| Process Resets Sequentially | Split in Batches | Sequential processing (reset) | Check Needs Reset (true) | Notify HR Leave Reset | Handles employees whose leave has ended. Sends return notification, emails reminder, and resets status to inactive in the sheet. |
| Notify HR Leave Reset | Slack | Post reset/return message to HR channel | Process Resets Sequentially | Send Welcome Back Email | Handles employees whose leave has ended. Sends return notification, emails reminder, and resets status to inactive in the sheet. |
| Send Welcome Back Email | Gmail | Email welcome-back to employee | Notify HR Leave Reset | Mark Leave As Inactive | Handles employees whose leave has ended. Sends return notification, emails reminder, and resets status to inactive in the sheet. |
| Mark Leave As Inactive | Google Sheets | Update sheet: set Status=Inactive | Send Welcome Back Email | Process Resets Sequentially | Handles employees whose leave has ended. Sends return notification, emails reminder, and resets status to inactive in the sheet. |
| Sticky Note | Sticky Note | Documentation / block label | — | — | Handles employees whose leave starts today. Sends Slack notification, emails reminder, updates sheet status, and prevents duplicate activations. |
| Sticky Note1 | Sticky Note | Documentation / block label | — | — | Handles employees whose leave has ended. Sends return notification, emails reminder, and resets status to inactive in the sheet. |
| Sticky Note2 | Sticky Note | Documentation / block label | — | — | Triggers daily execution, loads configuration values, reads leave records from Google Sheets, validates inputs, and calculates activation and reset conditions. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
2. **Add node: “Schedule Trigger”**
   - Name: `Daily Leave Check Trigger`
   - Configure: run **daily at 08:00** (set timezone appropriately in n8n/workflow settings if needed).
3. **Add node: “Google Sheets”** (read)
   - Name: `Fetch Leave Records`
   - Credentials: **Google Sheets OAuth2**
   - Select the target Spreadsheet and the sheet/tab containing leave rows.
   - Ensure the sheet has (at least) these columns:  
     - `Employee Email`, `Name`, `Start Date`, `End Date`, `Status`, `Last Updated`
4. **Connect:** `Daily Leave Check Trigger` → `Fetch Leave Records`
5. **Add node: “Code”**
   - Name: `Validate & Normalize Leave Data`
   - Paste logic that:
     - Normalizes today/start/end to midnight
     - Drops invalid rows (missing email/start/end; invalid date)
     - Outputs: `email`, `name`, `startDate`, `endDate`, `today`, `needsActivation`, `needsReset`
6. **Connect:** `Fetch Leave Records` → `Validate & Normalize Leave Data`
7. **Add node: “IF”**
   - Name: `Check Needs Activation`
   - Condition: Boolean `{{$json.needsActivation}}` is **true**
8. **Connect:** `Validate & Normalize Leave Data` → `Check Needs Activation`

### Activation branch
9. **Add node: “Split In Batches”**
   - Name: `Process Activations Sequentially`
   - Set **Batch Size = 1** (recommended to truly process sequentially).
10. **Connect (true branch):** `Check Needs Activation` (true) → `Process Activations Sequentially`
11. **Add node: “Slack”** (post message)
   - Name: `Notify HR Channel`
   - Credentials: **Slack OAuth2** (scopes typically include `chat:write`)
   - Select: `channel`
   - Choose the HR/alerts channel
   - Message text templated with `{{$json.name}}`, `{{$json.email}}`, `{{$json.startDate}}`, `{{$json.endDate}}`
12. **Connect:** `Process Activations Sequentially` → `Notify HR Channel`
13. **Add node: “Gmail”** (send email)
   - Name: `Send OOO Reminder Email`
   - Credentials: **Gmail OAuth2**
   - Send To: use the employee email from the batch item, e.g.  
     - `{{$('Process Activations Sequentially').item.json.email}}`
   - Subject/body: reference `name/startDate/endDate` from the same item.
14. **Connect:** `Notify HR Channel` → `Send OOO Reminder Email`
15. **Add node: “Google Sheets”** (Update)
   - Name: `Mark Leave as Active`
   - Credentials: Google Sheets OAuth2
   - Operation: **Update**
   - Matching column: `Employee Email`
   - Values to set:
     - `Status` = `Active`
     - `Last Updated` = `{{$('Process Activations Sequentially').item.json.today}}`
     - `Employee Email` = `{{$('Process Activations Sequentially').item.json.email}}`
16. **Connect:** `Send OOO Reminder Email` → `Mark Leave as Active`
17. **Loop batches:** Connect `Mark Leave as Active` → `Process Activations Sequentially` (so it continues with the next item).

### Reset branch
18. **Add node: “IF”**
   - Name: `Check Needs Reset`
   - Condition: Boolean `{{$json.needsReset}}` is **true**
19. **Connect (false branch):** `Check Needs Activation` (false) → `Check Needs Reset`
20. **Add node: “Split In Batches”**
   - Name: `Process Resets Sequentially`
   - Set **Batch Size = 1**
21. **Connect (true branch):** `Check Needs Reset` (true) → `Process Resets Sequentially`
22. **Add node: “Slack”**
   - Name: `Notify HR Leave Reset`
   - Same Slack credential/channel as activation
   - Message indicates leave completed
23. **Connect:** `Process Resets Sequentially` → `Notify HR Leave Reset`
24. **Add node: “Gmail”**
   - Name: `Send Welcome Back Email`
   - Send To: `{{$('Process Resets Sequentially').item.json.email}}`
   - Subject/body referencing `name/endDate`
25. **Connect:** `Notify HR Leave Reset` → `Send Welcome Back Email`
26. **Add node: “Google Sheets”** (Update)
   - Name: `Mark Leave As Inactive`
   - Operation: **Update**
   - Matching column: `Employee Email`
   - Set:
     - `Status` = `Inactive`
     - `Last Updated` = `{{$('Process Resets Sequentially').item.json.today}}`
     - `Employee Email` = `{{$('Process Resets Sequentially').item.json.email}}`
27. **Connect:** `Send Welcome Back Email` → `Mark Leave As Inactive`
28. **Loop batches:** Connect `Mark Leave As Inactive` → `Process Resets Sequentially`.

### Credentials checklist
29. Configure and test:
   - **Google Sheets OAuth2**: access to the spreadsheet
   - **Slack OAuth2**: bot/user authorized; bot invited to the channel
   - **Gmail OAuth2**: permission to send mail as the connected account

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Connect your Google Sheets OAuth2 credential. Connect your Slack OAuth2 credential. Connect your Gmail OAuth2 credential.” | From “Holiday Workflow Overview” sticky note |
| Workflow intentionally sends reminders and HR notifications; it does not modify Slack profiles or Gmail settings directly. | From “Holiday Workflow Overview” sticky note |

**Important implementation note (from analysis):** Both Gmail nodes’ **Send To** expressions are currently written in a way that will always send to `forreferal04@gmail.com` and not to the employee. If the intent is to email the employee, replace the expression with the employee email field from the current batch item.