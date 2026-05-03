Run A/B-tested email campaigns using Gmail, Google Sheets, and Slack

https://n8nworkflows.xyz/workflows/run-a-b-tested-email-campaigns-using-gmail--google-sheets--and-slack-12080


# Run A/B-tested email campaigns using Gmail, Google Sheets, and Slack

## 1. Workflow Overview

**Purpose:** This workflow runs A/B-tested email campaigns using **Gmail** for delivery, **Google Sheets** as the contact/campaign database, and **Slack** for operational notifications. It supports three entry points (manual, scheduled, webhook), validates/segments contacts, assigns A/B variants, sends in controlled batches with delays, logs outcomes back to Sheets, and posts start/completion/error alerts to Slack.

**Target use cases:**
- Newsletter or product update campaigns from a Google Sheet contact list
- Simple A/B subject line testing (50/50 split)
- Lightweight campaign tracking (sent/error flags + send logs) without a dedicated ESP

### 1.1 Triggers (Entry points)
Starts the campaign via Manual Trigger, weekly schedule, or an external POST webhook.

### 1.2 Campaign configuration + contact loading
Builds a campaign configuration object (including IDs and throttling settings), then loads contacts from Google Sheets.

### 1.3 Validation, segmentation, and A/B assignment
Validates emails, excludes already-sent/unsubscribed, applies optional segment targeting, assigns variant A/B and subject.

### 1.4 Campaign start logging + Slack notification
Appends a “Campaigns” row (stats + metadata) and notifies Slack that the campaign began.

### 1.5 Delivery pipeline (batching, personalization, Gmail send)
Splits valid contacts into batches, prepares personalized HTML, sends via Gmail with error continuation.

### 1.6 Per-email tracking (Sheet updates + send log) + anti-spam delays
Updates the contact row as sent or error, appends to “Send Log”, waits between sends, then continues.

### 1.7 Completion reporting
Generates a final summary, updates the campaign row as completed, and notifies Slack.

### 1.8 Global error handling
If the workflow run errors (outside per-email errors), it formats the error and alerts Slack.

---

## 2. Block-by-Block Analysis

### Block 1 — Triggers (Entry points)
**Overview:** Provides three ways to start the same campaign pipeline: manual, scheduled weekly, or webhook-driven. All triggers converge into “Configure Campaign”.

**Nodes involved:**
- Manual Trigger
- Scheduled Trigger
- Webhook Trigger

#### Node: Manual Trigger
- **Type / role:** `n8n-nodes-base.manualTrigger` — manual entry point for testing or ad-hoc runs.
- **Config choices:** No parameters.
- **Connections:** Outputs to **Configure Campaign**.
- **Edge cases:** None (manual execution only).

#### Node: Scheduled Trigger
- **Type / role:** `n8n-nodes-base.scheduleTrigger` — time-based entry point.
- **Config choices:** Cron `0 9 * * 1` → every Monday at 09:00.
- **Connections:** Outputs to **Configure Campaign**.
- **Edge cases:** Timezone is n8n instance timezone; DST changes may shift local expectation.

#### Node: Webhook Trigger
- **Type / role:** `n8n-nodes-base.webhook` — external entry point to start a campaign.
- **Config choices:**
  - Method: `POST`
  - Path: `email-campaign`
  - Webhook ID placeholder: `YOUR_WEBHOOK_ID`
- **Key variables used later:** `Configure Campaign` reads `body?.user` from this trigger payload.
- **Connections:** Outputs to **Configure Campaign**.
- **Edge cases / failures:**
  - Wrong HTTP method/path, missing activation, or firewall prevents calls.
  - If payload has no `body.user`, starter will be set to `manual`.

---

### Block 2 — Campaign configuration + contact loading
**Overview:** Creates a campaign configuration object (including throttling and A/B settings) and loads contacts from Google Sheets “Contacts”.

**Nodes involved:**
- Configure Campaign
- Read Contacts

#### Node: Configure Campaign
- **Type / role:** `n8n-nodes-base.code` — central configuration and campaign metadata creation.
- **Configuration choices (interpreted):**
  - Defines `CONFIG` with:
    - Company branding: `companyName`, `senderEmail`, colors, optional `logoUrl`
    - Throttling: `batchSize`, `delayBetweenEmails` (seconds), `delayBetweenBatches` (seconds, but not actually used elsewhere), `maxRetries` (defined but not used elsewhere)
    - Campaign: `campaignName`, `emailSubject`, `alternativeSubject`, `enableABTest`
    - Segmentation: `targetSegment` (empty means all)
    - Tracking flags: `trackOpens`, `trackClicks` (declared but not used in later nodes)
  - Generates `campaignId` like `CAMP-<timestamp>-<random>`
  - Adds `startedAt` ISO timestamp
  - Adds `startedBy` from webhook body user if present
- **Key expressions/variables:**
  - Uses `$input.item.json.body?.user` (exists only for webhook runs).
- **Connections:** Outputs to **Read Contacts**.
- **Edge cases / failures:**
  - Code node exceptions (syntax errors) stop the run.
  - Some config fields are unused (senderEmail, delayBetweenBatches, maxRetries, trackOpens/trackClicks), which may mislead operators.

#### Node: Read Contacts
- **Type / role:** `n8n-nodes-base.googleSheets` — reads contacts to send to.
- **Configuration choices:**
  - Document: `YOUR_DOCUMENT_ID`
  - Sheet/tab: `Contacts`
  - Operation: implied “read/get all rows” (node configured as a read by virtue of lacking an explicit write operation).
- **Connections:** Outputs rows to **Filter and Validate**.
- **Edge cases / failures:**
  - OAuth/permissions errors (Sheets credential)
  - Wrong document ID or missing “Contacts” sheet
  - Column naming mismatches with downstream code expectations (`email`, `sent`, `unsubscribed`, `segment`, `name`, `row_number`)

---

### Block 3 — Validation, segmentation, and A/B assignment
**Overview:** Filters contacts into categories (valid/invalid/already sent/unsubscribed/other segment), assigns A/B variants, and produces campaign stats plus a list of valid contacts.

**Nodes involved:**
- Filter and Validate

#### Node: Filter and Validate
- **Type / role:** `n8n-nodes-base.code` — validates and prepares contact dataset.
- **Configuration choices:**
  - Reads config from `$('Configure Campaign').first().json`
  - Iterates over all incoming rows via `$input.all()`
  - Email validation with regex `^[^\s@]+@[^\s@]+\.[^\s@]+$`
  - Exclusion rules (in order):
    1. invalid email → `invalid`
    2. `sent` is `yes/true` → `alreadySent`
    3. `unsubscribed` is `yes/true` → `unsubscribed`
    4. if `targetSegment` set and row’s `segment` doesn’t match → `otherSegment`
    5. else → `valid`
  - A/B assignment:
    - If `enableABTest` true: random 50/50 to variant `A` or `B`
    - Subject assigned based on variant
- **Key outputs:**
  - `stats`: totals and counts incl. `variantA`, `variantB`
  - `validContacts`: array of prepared contacts `{..., email, abVariant, subject}`
  - `invalidContacts`: array with `{..., reason}`
- **Connections:** Outputs to **Log Campaign Start**.
- **Edge cases / failures:**
  - If the sheet doesn’t provide expected fields, many rows may be flagged invalid (e.g., missing `email`).
  - Random A/B assignment is not stable across reruns; re-running can change variant assignment for the same contact (unless already marked `sent`).
  - `sent`/`unsubscribed` comparisons assume text values; if sheet uses booleans or other markers, adjust logic.

---

### Block 4 — Campaign start logging + Slack notification
**Overview:** Records the campaign start and initial stats in a “Campaigns” sheet, then posts a “started” message to Slack.

**Nodes involved:**
- Log Campaign Start
- Slack - Campaign Started
- Has Contacts?

#### Node: Log Campaign Start
- **Type / role:** `n8n-nodes-base.googleSheets` — appends one row per campaign run.
- **Configuration choices:**
  - Document: `YOUR_DOCUMENT_ID`
  - Sheet/tab: `Campaigns`
  - Operation: `append`
  - Mapped columns include:
    - `Name` = campaign name
    - `Status` = “In Progress”
    - counts: `Valid`, `Invalid`, `Variant A`, `Variant B`, `Already Sent`, `Unsubscribed`, `Total Contacts`
    - `Start Date`, `Campaign ID`
- **Key expressions:**
  - Pulls config via `$('Configure Campaign').item.json...`
  - Pulls stats via `$json.stats...`
- **Connections:** Outputs to **Slack - Campaign Started**.
- **Edge cases / failures:**
  - If “Campaigns” tab lacks these headers, append may fail or create misaligned columns.
  - This node does not store the appended row ID for later update; later “Update Campaign” attempts to update without an explicit row reference (see Completion block).

#### Node: Slack - Campaign Started
- **Type / role:** `n8n-nodes-base.slack` — posts a start notification.
- **Configuration choices:**
  - Sends text `:rocket: *Email campaign started*`
  - `includeLinkToWorkflow: false`
- **Connections:** Outputs to **Has Contacts?**
- **Edge cases / failures:**
  - Slack credential/webhook issues; channel routing depends on Slack node credential configuration.

#### Node: Has Contacts?
- **Type / role:** `n8n-nodes-base.if` — gates sending vs. “no contacts” completion path.
- **Configuration choices:**
  - Condition: `$json.stats.valid` **larger** than (implicit) `0`
- **Connections:**
  - **True** → Extract Contacts
  - **False** → Calculate Results
- **Edge cases / failures:**
  - If `stats.valid` is missing/not numeric, the condition may behave unexpectedly.

---

### Block 5 — Delivery pipeline (batching, personalization, Gmail send)
**Overview:** Converts the valid contacts array into items, processes them in batches, builds personalized HTML emails, and sends them via Gmail. The Gmail node continues on error so individual failures don’t stop the run.

**Nodes involved:**
- Extract Contacts
- Process in Batches
- Prepare Email
- Send Email

#### Node: Extract Contacts
- **Type / role:** `n8n-nodes-base.code` — transforms `validContacts[]` into individual items.
- **Configuration choices:**
  - Reads `$('Filter and Validate').first().json.validContacts`
  - Returns `validContacts.map(c => ({ json: c }))`
- **Connections:** Outputs to **Process in Batches**.
- **Edge cases / failures:**
  - If `validContacts` is undefined, code throws (but should only run when Has Contacts? is true).

#### Node: Process in Batches
- **Type / role:** `n8n-nodes-base.splitInBatches` — throttles processing by batching items.
- **Configuration choices:**
  - `batchSize` from config: `$('Configure Campaign').first().json.batchSize`
- **Connections:**
  - Main output (per batch item) → **Prepare Email**
  - “No more items” output → **Final Summary**
  - Loop-back: **Anti-Spam Delay** connects back into this node to continue.
- **Edge cases / failures:**
  - `delayBetweenBatches` exists in config but is not applied; only per-email delay is used.
  - If `batchSize` is a string that can’t coerce to number, batching may fail.

#### Node: Prepare Email
- **Type / role:** `n8n-nodes-base.code` — builds subject + HTML body per contact.
- **Configuration choices:**
  - Defines replacement variables:
    - `{{name}}`, `{{email}}`, `{{company}}`, `{{date}}`, `{{campaignId}}`
  - Subject templating: replaces variables in `contact.subject` (already assigned A/B)
  - HTML template:
    - Header with optional logo
    - Content with “Learn More” button including UTM `utm_campaign={{campaignId}}`
    - Footer with unsubscribe/preferences links including `email` and `campaignId`
  - Output fields:
    - `rowNumber: contact.row_number` (used later for Sheets update, but update nodes don’t explicitly reference it)
    - `to`, `name`, `abVariant`, `subject`, `htmlContent`, `campaignId`, `sentAt`
- **Connections:** Outputs to **Send Email**.
- **Edge cases / failures:**
  - If the incoming contact does not have `name` or `subject`, fallback is applied for name but subject may become incorrect/empty.
  - `row_number` must exist if you want deterministic row updates in Sheets (but current update nodes don’t use it).
  - Unsubscribe links are placeholders (`yourcompany.com`)—ensure they match your actual system.

#### Node: Send Email
- **Type / role:** `n8n-nodes-base.gmail` — sends the email through Gmail.
- **Configuration choices:**
  - `sendTo` = `$json.to`
  - `subject` = `$json.subject`
  - `message` = `$json.htmlContent`
  - `appendAttribution: false`
  - **onError:** `continueErrorOutput` (critical: routes failures to an error output instead of stopping run)
- **Connections:**
  - Success output → **Mark as Sent**
  - Error output → **Mark as Error**
- **Edge cases / failures:**
  - Gmail OAuth auth errors, quota limits, rate limiting, or daily send caps.
  - Gmail may sanitize HTML; ensure “message” field supports HTML in your n8n Gmail node version.
  - Large HTML may exceed provider limits.

---

### Block 6 — Per-email tracking (Sheet updates + send log) + anti-spam delay
**Overview:** After each send attempt, updates the contact status in “Contacts”, appends a row in “Send Log”, then waits before continuing the batch loop.

**Nodes involved:**
- Mark as Sent
- Log Success
- Mark as Error
- Log Error
- Anti-Spam Delay

#### Node: Mark as Sent
- **Type / role:** `n8n-nodes-base.googleSheets` — updates a contact row to indicate successful send.
- **Configuration choices:**
  - Document: `YOUR_DOCUMENT_ID`
  - Sheet/tab: `Contacts`
  - Operation: `update`
  - Sets columns:
    - `sent = "yes"`
    - `variant = $json.abVariant`
    - `campaign = $json.campaignId`
    - `sent_date = $json.sentAt`
- **Connections:** Outputs to **Log Success**.
- **Edge cases / failures:**
  - **Missing row reference:** Update operations typically require a row ID/row number/key. This node does not show how the row is selected (no explicit “Row Number” parameter visible). If not configured in UI, it may fail or update the wrong row.
  - If sheet headers differ (`sent_date` vs `sentDate`), update fails.

#### Node: Log Success
- **Type / role:** `n8n-nodes-base.googleSheets` — appends a send log entry for successful sends.
- **Configuration choices:**
  - Sheet/tab: `Send Log`
  - Operation: `append`
  - Columns: Name, Email, Status=Sent, Subject, Variant, Timestamp, Campaign ID
- **Connections:** Outputs to **Anti-Spam Delay**.
- **Edge cases / failures:** “Send Log” tab/headers must exist.

#### Node: Mark as Error
- **Type / role:** `n8n-nodes-base.googleSheets` — updates a contact row to indicate an error.
- **Configuration choices:**
  - Sheet/tab: `Contacts`
  - Operation: `update`
  - Sets:
    - `sent="error"`
    - `error_msg = $json.error?.message || 'Send error'`
    - `sent_date = now`
- **Connections:** Outputs to **Log Error**.
- **Edge cases / failures:**
  - Same **row selection** risk as “Mark as Sent”.
  - `$json.error` exists only on Gmail error output; if structure differs, message may be blank.

#### Node: Log Error
- **Type / role:** `n8n-nodes-base.googleSheets` — appends a send log entry for failures.
- **Configuration choices:**
  - Sheet/tab: `Send Log`
  - Operation: `append`
  - Columns similar to success, but `Status=Error`, timestamp is “now”, and Campaign ID pulled from Configure Campaign.
- **Connections:** Outputs to **Anti-Spam Delay**.
- **Edge cases / failures:** Same logging sheet requirements.

#### Node: Anti-Spam Delay
- **Type / role:** `n8n-nodes-base.wait` — throttles sending to reduce spam/rate-limit risk.
- **Configuration choices:**
  - Wait amount = `$('Configure Campaign').first().json.delayBetweenEmails`
  - (Unit is determined by the node’s UI configuration; here only “amount” is provided in JSON.)
- **Connections:** Outputs back to **Process in Batches** to continue loop.
- **Edge cases / failures:**
  - If the Wait node interprets the amount in minutes by default, you may unintentionally wait too long. Ensure the unit matches “seconds” intent.
  - Very low delays may still hit Gmail rate limits.

---

### Block 7 — Completion reporting
**Overview:** Handles both “no valid contacts” and “finished sending” paths, builds a final summary, attempts to update the campaign record, and notifies Slack.

**Nodes involved:**
- Calculate Results
- Final Summary
- Update Campaign
- Slack - Campaign Completed
- Done

#### Node: Calculate Results
- **Type / role:** `n8n-nodes-base.code` — prepares a message when there are no valid contacts.
- **Configuration choices:**
  - Reads stats from `$('Filter and Validate').first().json.stats`
  - Outputs `{ message: 'No valid contacts to send', stats }`
- **Connections:** Outputs to **Final Summary**.
- **Edge cases:** None significant.

#### Node: Final Summary
- **Type / role:** `n8n-nodes-base.code` — creates final campaign summary and duration.
- **Configuration choices:**
  - Reads config and initial stats
  - Computes duration in minutes (rounded), returns:
    - campaign identifiers
    - started/finished timestamps
    - human-readable `duration`
    - processed counts (based on initial valid count, not actual successful sends)
    - `summary` string
- **Connections:** Outputs to **Update Campaign**.
- **Edge cases / limitations:**
  - “totalProcessed” equals initial valid contacts, not “sent successfully”. Errors are not subtracted.
  - Duration rounding: runs under 30 seconds show “Less than 1 minute”.

#### Node: Update Campaign
- **Type / role:** `n8n-nodes-base.googleSheets` — marks the campaign as completed in “Campaigns”.
- **Configuration choices:**
  - Sheet/tab: `Campaigns`
  - Operation: `update`
  - Sets `Status=Completed`, `Duration`, `End Date`
- **Connections:** Outputs to **Slack - Campaign Completed**.
- **Edge cases / failures:**
  - **Likely incomplete configuration:** Updating a specific campaign row typically requires row identification (row number or lookup by Campaign ID). The workflow does not pass/store the appended row reference from “Log Campaign Start”, so this update may fail or update an unintended row unless the node is configured in the UI to find the row by “Campaign ID”.

#### Node: Slack - Campaign Completed
- **Type / role:** `n8n-nodes-base.slack` — posts completion message.
- **Configuration choices:** Text `:white_check_mark: *Campaign completed*`, link-to-workflow disabled.
- **Connections:** Outputs to **Done**.
- **Edge cases:** Slack credential/channel issues.

#### Node: Done
- **Type / role:** `n8n-nodes-base.noOp` — terminator node for clarity.
- **Connections:** None.

---

### Block 8 — Global error handling
**Overview:** If the overall workflow run errors (beyond the Gmail node’s per-item continue-on-error), it formats a concise error payload and posts to Slack with a workflow link.

**Nodes involved:**
- Error Trigger
- Format Error
- Slack - Error Alert

#### Node: Error Trigger
- **Type / role:** `n8n-nodes-base.errorTrigger` — runs when workflow execution fails.
- **Config choices:** None.
- **Connections:** Outputs to **Format Error**.
- **Edge cases:** Only fires for execution errors; it will not necessarily capture “business logic” failures if nodes continue-on-fail.

#### Node: Format Error
- **Type / role:** `n8n-nodes-base.code` — normalizes error info into a message object.
- **Configuration choices:**
  - Reads `$input.item.json` as error object
  - Outputs: `errorTime`, `errorMessage`, `errorNode`, hard-coded `workflow: 'Email Marketing PRO'`
- **Connections:** Outputs to **Slack - Error Alert**.
- **Edge cases:** If error object structure differs, `error.node?.name` may be undefined.

#### Node: Slack - Error Alert
- **Type / role:** `n8n-nodes-base.slack` — posts error alert to Slack.
- **Configuration choices:**
  - Text `:rotating_light: *Workflow Error*`
  - `includeLinkToWorkflow: true`
- **Connections:** None.
- **Edge cases:** Slack failures could hide critical errors; consider also email/pager escalation.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Manual Trigger | Manual Trigger | Manual start entry point | — | Configure Campaign | ## Campaign triggers<br><br>Manual, scheduled, or webhook. |
| Scheduled Trigger | Schedule Trigger | Weekly automated entry point | — | Configure Campaign | ## Campaign triggers<br><br>Manual, scheduled, or webhook. |
| Webhook Trigger | Webhook | External start entry point (POST) | — | Configure Campaign | ## Campaign triggers<br><br>Manual, scheduled, or webhook. |
| Configure Campaign | Code | Defines campaign config + IDs | Manual Trigger; Scheduled Trigger; Webhook Trigger | Read Contacts | ## Campaign setup<br><br>Loads and validates contacts. |
| Read Contacts | Google Sheets | Load contacts from “Contacts” tab | Configure Campaign | Filter and Validate | ## Campaign setup<br><br>Loads and validates contacts. |
| Filter and Validate | Code | Validate, segment, A/B assign, compute stats | Read Contacts | Log Campaign Start | ## Campaign setup<br><br>Loads and validates contacts. |
| Log Campaign Start | Google Sheets | Append campaign start row in “Campaigns” | Filter and Validate | Slack - Campaign Started | ## Logging<br><br>Logs start to Sheets and Slack. |
| Slack - Campaign Started | Slack | Notify Slack campaign started | Log Campaign Start | Has Contacts? | ## Logging<br><br>Logs start to Sheets and Slack. |
| Has Contacts? | IF | Branch if any valid contacts exist | Slack - Campaign Started | Extract Contacts; Calculate Results | ## Validation<br><br>Checks for valid contacts. |
| Extract Contacts | Code | Convert validContacts array into items | Has Contacts? (true) | Process in Batches | ## Email delivery<br><br>Sends in batches with delays. |
| Process in Batches | Split In Batches | Batch iteration + completion output | Extract Contacts; Anti-Spam Delay | Prepare Email; Final Summary | ## Email delivery<br><br>Sends in batches with delays. |
| Prepare Email | Code | Personalize subject + build HTML | Process in Batches | Send Email | ## Email delivery<br><br>Sends in batches with delays. |
| Send Email | Gmail | Send HTML emails; continue on error | Prepare Email | Mark as Sent; Mark as Error | ## Email delivery<br><br>Sends in batches with delays. |
| Mark as Sent | Google Sheets | Update contact row as sent | Send Email (success) | Log Success | ## Send tracking<br><br>Tracks results and handles delays. |
| Log Success | Google Sheets | Append “Sent” entry to “Send Log” | Mark as Sent | Anti-Spam Delay | ## Send tracking<br><br>Tracks results and handles delays. |
| Mark as Error | Google Sheets | Update contact row as error | Send Email (error) | Log Error | ## Send tracking<br><br>Tracks results and handles delays. |
| Log Error | Google Sheets | Append “Error” entry to “Send Log” | Mark as Error | Anti-Spam Delay | ## Send tracking<br><br>Tracks results and handles delays. |
| Anti-Spam Delay | Wait | Delay between sends; loop control | Log Success; Log Error | Process in Batches | ## Send tracking<br><br>Tracks results and handles delays. |
| Calculate Results | Code | “No valid contacts” message path | Has Contacts? (false) | Final Summary | ## Campaign completion<br><br>Calculates stats and notifies team. |
| Final Summary | Code | Create duration + summary payload | Process in Batches (done); Calculate Results | Update Campaign | ## Campaign completion<br><br>Calculates stats and notifies team. |
| Update Campaign | Google Sheets | Update campaign status to completed | Final Summary | Slack - Campaign Completed | ## Campaign completion<br><br>Calculates stats and notifies team. |
| Slack - Campaign Completed | Slack | Notify Slack campaign completed | Update Campaign | Done | ## Campaign completion<br><br>Calculates stats and notifies team. |
| Done | No Operation | Visual end node | Slack - Campaign Completed | — | ## Campaign completion<br><br>Calculates stats and notifies team. |
| Error Trigger | Error Trigger | Catch workflow execution errors | — | Format Error | ## Error handling<br><br>Alerts team on Slack. |
| Format Error | Code | Normalize error payload | Error Trigger | Slack - Error Alert | ## Error handling<br><br>Alerts team on Slack. |
| Slack - Error Alert | Slack | Post error alert + workflow link | Format Error | — | ## Error handling<br><br>Alerts team on Slack. |
| Overview | Sticky Note | Documentation panel | — | — | ## Send email campaigns with A/B testing using Gmail and Google Sheets<br><br>Email marketing with contact management, delivery tracking, and analytics.<br><br>### How it works<br><br>1. **Trigger** - Start manually, scheduled, or via webhook<br>2. **Load** - Reads contacts from Google Sheets<br>3. **Validate** - Filters invalid emails and duplicates<br>4. **Send** - Delivers in batches with anti-spam delay<br>5. **Track** - Logs successes and errors to Sheets<br>6. **Report** - Sends Slack summary with stats<br><br>### Setup steps<br><br>1. **Connect:** Gmail, Google Sheets, Slack<br>2. **Create Sheet:** Tabs for Contacts, Campaigns, Logs<br>3. **Update IDs:** Replace YOUR_DOCUMENT_ID<br>4. **Slack:** #marketing and #errors channels<br>5. **Test:** Run with test contacts first |
| Triggers | Sticky Note | Section label | — | — | ## Campaign triggers<br><br>Manual, scheduled, or webhook. |
| Setup | Sticky Note | Section label | — | — | ## Campaign setup<br><br>Loads and validates contacts. |
| Logging | Sticky Note | Section label | — | — | ## Logging<br><br>Logs start to Sheets and Slack. |
| Validation | Sticky Note | Section label | — | — | ## Validation<br><br>Checks for valid contacts. |
| Delivery | Sticky Note | Section label | — | — | ## Email delivery<br><br>Sends in batches with delays. |
| Tracking | Sticky Note | Section label | — | — | ## Send tracking<br><br>Tracks results and handles delays. |
| Completion | Sticky Note | Section label | — | — | ## Campaign completion<br><br>Calculates stats and notifies team. |
| Errors | Sticky Note | Section label | — | — | ## Error handling<br><br>Alerts team on Slack. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add triggers (3 nodes) and connect each to the same next node:**
   1) Add **Manual Trigger** → connect to “Configure Campaign” (later).  
   2) Add **Schedule Trigger**:
      - Set **Cron** to: `0 9 * * 1`
      - Connect to “Configure Campaign”.  
   3) Add **Webhook Trigger**:
      - Method: **POST**
      - Path: `email-campaign`
      - Connect to “Configure Campaign”.

3. **Add “Configure Campaign” (Code node):**
   - Paste the configuration code that:
     - Defines company + throttling + campaign + segmentation settings
     - Generates a `campaignId`
     - Sets `startedAt`
     - Sets `startedBy` from webhook payload (optional)
   - Connect it to “Read Contacts”.

4. **Add “Read Contacts” (Google Sheets node):**
   - Credentials: Google Sheets OAuth2 with access to your spreadsheet
   - Document ID: set your spreadsheet ID
   - Sheet name: `Contacts`
   - Operation: read/get rows (default read behavior)
   - Connect to “Filter and Validate”.

5. **Add “Filter and Validate” (Code node):**
   - Paste logic to:
     - Validate emails
     - Skip sent/unsubscribed
     - Apply optional `targetSegment`
     - Assign `abVariant` and `subject`
     - Output `stats`, `validContacts`, `invalidContacts`
   - Connect to “Log Campaign Start”.

6. **Add “Log Campaign Start” (Google Sheets, append):**
   - Sheet: `Campaigns`
   - Operation: **Append**
   - Map columns (must exist as headers) such as:
     - Name, Status, Start Date, Campaign ID, Total Contacts, Valid, Invalid, Variant A, Variant B, Already Sent, Unsubscribed
   - Connect to “Slack - Campaign Started”.

7. **Add “Slack - Campaign Started” (Slack node):**
   - Credentials: Slack (bot token) or webhook (depending on node configuration)
   - Message text: `:rocket: *Email campaign started*`
   - Connect to “Has Contacts?”.

8. **Add “Has Contacts?” (IF node):**
   - Condition: Number → `{{$json.stats.valid}}` is **larger than** `0`
   - True output → “Extract Contacts”
   - False output → “Calculate Results”

9. **True branch (sending path):**
   1) Add “Extract Contacts” (Code):
      - Convert `validContacts` array into item list.
      - Connect to “Process in Batches”.
   2) Add “Process in Batches” (Split In Batches):
      - Batch size: `{{$('Configure Campaign').first().json.batchSize}}`
      - Connect main output to “Prepare Email”
      - Connect “No more items” output to “Final Summary”
   3) Add “Prepare Email” (Code):
      - Build subject replacement + HTML template
      - Output: `to`, `subject`, `htmlContent`, `abVariant`, `campaignId`, `sentAt`, etc.
      - Connect to “Send Email”.
   4) Add “Send Email” (Gmail node):
      - Credentials: Gmail OAuth2
      - To: `{{$json.to}}`
      - Subject: `{{$json.subject}}`
      - Message/body: `{{$json.htmlContent}}`
      - Disable attribution if desired
      - Set node error handling to **Continue on Fail / Continue Error Output** (so you get separate success/error outputs)

10. **Per-email tracking (success path):**
    1) Add “Mark as Sent” (Google Sheets, Update):
       - Sheet: `Contacts`
       - Operation: **Update**
       - Configure how the row is selected (critical): use Row Number / Row ID / lookup by email (implementation depends on your Sheet structure).
       - Set `sent=yes`, `variant`, `campaign`, `sent_date`.
       - Connect to “Log Success”.
    2) Add “Log Success” (Google Sheets, Append):
       - Sheet: `Send Log`
       - Append fields including Name, Email, Status=Sent, Subject, Variant, Timestamp, Campaign ID
       - Connect to “Anti-Spam Delay”.

11. **Per-email tracking (error path):**
    1) Add “Mark as Error” (Google Sheets, Update):
       - Same row selection approach as “Mark as Sent”
       - Set `sent=error`, `error_msg`, `sent_date=now`
       - Connect to “Log Error”.
    2) Add “Log Error” (Google Sheets, Append):
       - Sheet: `Send Log`
       - Append Status=Error, etc.
       - Connect to “Anti-Spam Delay”.

12. **Add “Anti-Spam Delay” (Wait node) and loop:**
    - Wait amount: `{{$('Configure Campaign').first().json.delayBetweenEmails}}`
    - Ensure the unit matches your intent (seconds).
    - Connect “Anti-Spam Delay” back to “Process in Batches” to continue the loop.

13. **False branch (no valid contacts):**
    - Add “Calculate Results” (Code) to produce `{message, stats}`
    - Connect to “Final Summary”.

14. **Completion path:**
    1) Add “Final Summary” (Code):
       - Compute duration
       - Output final `finishedAt`, `duration`, and summary fields
    2) Add “Update Campaign” (Google Sheets, Update):
       - Sheet: `Campaigns`
       - Operation: **Update**
       - **Configure row selection** (recommended: lookup by `Campaign ID`)
       - Set `Status=Completed`, `Duration`, `End Date`
    3) Add “Slack - Campaign Completed” (Slack):
       - Text: `:white_check_mark: *Campaign completed*`
    4) Add “Done” (NoOp) as an end marker.

15. **Global error handling:**
    1) Add “Error Trigger” node.
    2) Add “Format Error” (Code) to extract message, node name, and timestamp.
    3) Add “Slack - Error Alert” (Slack) with:
       - Text: `:rotating_light: *Workflow Error*`
       - Enable “Include link to workflow”.

**Required credentials:**
- **Gmail OAuth2** (Gmail node)
- **Google Sheets OAuth2** (all Sheets nodes)
- **Slack** (Slack nodes; bot token or webhook as configured)

**Required Google Sheets tabs (recommended headers):**
- `Contacts`: `email`, `name`, `sent`, `unsubscribed`, `segment`, plus columns used for updates (`variant`, `campaign`, `sent_date`, `error_msg`)
- `Campaigns`: `Name`, `Campaign ID`, `Status`, `Start Date`, `End Date`, `Duration`, and counters columns
- `Send Log`: `Name`, `Email`, `Status`, `Subject`, `Variant`, `Timestamp`, `Campaign ID`

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Replace `YOUR_DOCUMENT_ID` with your Google Sheets document ID in all Google Sheets nodes. | Workflow setup (Sheets integration) |
| Webhook trigger path is `POST /email-campaign`; ensure the workflow is active to receive calls. | Triggering externally |
| Slack channels are implied as `#marketing` and `#errors` in the overview note; actual channel depends on Slack credential/node configuration. | Operational notifications |
| Config includes `delayBetweenBatches`, `maxRetries`, `trackOpens`, `trackClicks` but they are not implemented elsewhere in the workflow. | Potential enhancements / avoid confusion |
| Contact “update” nodes likely require explicit row selection (row number/id or lookup key). Ensure the update criteria is configured in the Google Sheets Update nodes. | Data integrity / correctness risk |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.