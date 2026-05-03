Send subscription renewal reminders with Email and ClickUp

https://n8nworkflows.xyz/workflows/send-subscription-renewal-reminders-with-email-and-clickup-13259


# Send subscription renewal reminders with Email and ClickUp

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Subscription Renewal Reminder – Email & ClickUp  
**Purpose:** Triggered via an HTTP webhook, this workflow evaluates ClickUp “subscription” tasks by due date, sends renewal reminder emails to customers whose subscriptions are nearing expiry (within a configurable threshold, default **7 days**), and then notifies an administrator with either a summary (if reminders were sent) or a “nothing due” confirmation.

**Primary use cases**
- Automated subscription renewal outreach driven by ClickUp task due dates
- Centralized reporting to an ops/admin mailbox after each run
- External scheduling (cron, cloud scheduler, ClickUp automation, etc.) via webhook call

### Logical blocks
**1.1 Input Reception (Webhook trigger)**  
Starts execution whenever the webhook endpoint is called.

**1.2 Defaults & Configuration Injection**  
Defines reusable parameters (ClickUp List ID, admin email, sender email, threshold days). (Note: in the provided JSON, the *Set Defaults* node exists but appears not yet configured.)

**1.3 Processing & Filtering (compute days left + split outputs)**  
Converts ClickUp due dates to days remaining and emits:
- Output 0: items that need customer reminders
- Output 1: a single summary item for admin decision-making

**1.4 Reminder Dispatch Loop (batch, personalize, throttle, send)**  
Sends one email per expiring subscription, with a delay to avoid rate limits.

**1.5 Admin Reporting + Completion**  
If at least one reminder was sent: send a summary email to admin.  
Else: send a “no expiries” email.  
Finally: return a “completed” status item.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception
**Overview:** Receives an inbound HTTP request to start the workflow.

**Nodes involved**
- **Incoming Request** (Webhook)

**Node details**
- **Incoming Request**
  - **Type / role:** `Webhook` — entry point; triggers the workflow on HTTP POST.
  - **Configuration (interpreted):**
    - Method: **POST**
    - Path: **/subscription-renewal**
  - **Inputs/Outputs:** No inputs; outputs one item representing the webhook request payload/metadata to **Set Defaults**.
  - **Edge cases / failures:**
    - Wrong method (GET vs POST) or wrong path → no trigger.
    - If upstream scheduler expects a synchronous response, ensure n8n webhook response behavior matches expectations (here, no explicit response node is used).

---

### 2.2 Defaults & Configuration Injection
**Overview:** Intended to centralize configuration values for list ID, sender/admin email, and threshold, so downstream nodes can rely on consistent variables.

**Nodes involved**
- **Set Defaults** (Set)

**Node details**
- **Set Defaults**
  - **Type / role:** `Set` — enriches/overrides incoming JSON with defaults.
  - **Configuration (interpreted):**
    - In the provided JSON, **no fields are defined** yet (empty parameters). However, downstream code expects fields like:
      - `daysThreshold`
      - `adminEmail`
      - `fromEmail`
      - (and likely a `clickupListId`, although the workflow currently contains **no ClickUp fetch node**)
  - **Inputs/Outputs:** Receives webhook item from **Incoming Request**, outputs to **Filter Expiring & Build Data**.
  - **Edge cases / failures:**
    - Missing `adminEmail` / `fromEmail` will propagate into email nodes, causing send failures or fallbacks.
    - The code node references `$json[0].daysThreshold` and `$json[0].adminEmail`, which is unusual for a Set/Webhook single item flow; this can lead to expression/logic issues (details in block 2.3).

---

### 2.3 Processing & Filtering
**Overview:** Computes days remaining until each subscription’s due date and builds two streams: (a) customer reminders and (b) run summary for admin logic.

**Nodes involved**
- **Filter Expiring & Build Data** (Code)
- **Split per Subscription** (SplitInBatches)
- **Any Reminders Sent?** (IF)

**Node details**
- **Filter Expiring & Build Data**
  - **Type / role:** `Code` — transforms items; filters expiring subscriptions; creates summary output.
  - **Key configuration choices:**
    - Threshold default: `7` days if not provided.
    - Uses `task.due_date` (expects a UNIX timestamp in ms as a string/number).
    - Looks for a ClickUp custom field named **"Email"** to extract the customer email.
  - **Key variables / expressions:**
    - `const threshold = $json[0].daysThreshold || 7;`
    - `const adminEmail = $json[0].adminEmail;`
    - `daysLeft = Math.ceil((dueTs - now) / MS_IN_DAY)`
    - Custom field lookup: `(task.custom_fields || []).find(f => f.name === 'Email')`
    - Fallback email: `'user@example.com'`
  - **Outputs:**
    - **Output 0 (“reminders”)**: array of items with:
      - `taskId`, `subscriptionName`, `dueDateISO`, `daysLeft`, `customerEmail`, `fromEmail`, `adminEmail`, `threshold`
    - **Output 1 (“summary”)**: single item with:
      - `runAt`, `totalReminders`, `adminEmail`, `threshold`
  - **Connections:**
    - Output 0 → **Split per Subscription**
    - Output 1 is *intended* to go to decision logic, but **is not connected** in the provided workflow (see “Important wiring issue” below).
  - **Edge cases / failures:**
    - **Important wiring issue:** In the JSON connections, only Output 0 is connected to **Split per Subscription**. Output 1 (summary) is not connected, so **Any Reminders Sent?** will not receive the summary item from this node.
    - **Data source issue:** There is **no ClickUp node** fetching tasks. This code expects ClickUp-like task items (`id`, `name`, `due_date`, `custom_fields`) but will actually receive only what the webhook/Set node provides.
    - `$json[0]` usage: In a code node, `$json` is the current item’s JSON object (not an array). Using `$json[0]` will usually yield `undefined`. As written, `threshold` will fall back to 7, but `adminEmail` may become `undefined`.
    - Tasks missing `due_date` are skipped.
    - Past-due tasks (daysLeft negative) still pass `daysLeft <= threshold` and will be emailed unless you explicitly exclude `< 0`.
    - ClickUp due dates are often stored in **milliseconds** as strings; if in seconds, calculations will be wrong.

- **Split per Subscription**
  - **Type / role:** `Split In Batches` — processes reminder items sequentially/in controllable batch size.
  - **Configuration (interpreted):**
    - Batch size not explicitly set in JSON (defaults apply).
  - **Inputs/Outputs:**
    - Input from **Filter Expiring & Build Data** output 0
    - Output 0 → **Prepare Reminder Email** (per item)
    - Output 1 → **Any Reminders Sent?** (after batches complete)
  - **Edge cases / failures:**
    - With zero items, behavior depends on n8n version and SplitInBatches semantics; the “done” output may fire immediately (often desired).
    - If batch size is too large, you may hit email provider rate limits without sufficient throttling.

- **Any Reminders Sent?**
  - **Type / role:** `IF` — chooses admin summary vs. “no expiries” message.
  - **Configuration (interpreted):**
    - Condition: `totalReminders > 0`
    - Expression: `={{ $json.totalReminders }} larger than 0`
  - **Inputs/Outputs:**
    - Receives data from **Split per Subscription** “done” output (index 1).
    - TRUE → **Build Admin Summary**
    - FALSE → **Prepare No-Expiry Email**
  - **Edge cases / failures:**
    - If it receives a reminder item instead of the summary item, `$json.totalReminders` will be undefined → condition evaluation may fail or always be false.
    - As wired, it is not guaranteed to receive the summary output from the code node (see wiring issue above).

---

### 2.4 Reminder Dispatch Loop
**Overview:** For each expiring subscription, the workflow prepares a personalized email, waits briefly, sends the email, then logs status.

**Nodes involved**
- **Prepare Reminder Email** (Set)
- **Throttle Reminders** (Wait)
- **Send Reminder Email** (Email Send)
- **Log Reminder Status** (Set)

**Node details**
- **Prepare Reminder Email**
  - **Type / role:** `Set` — constructs fields required by Email Send node (subject, toEmail, body, etc.).
  - **Configuration (interpreted):**
    - In provided JSON, no fields are defined, but downstream **Send Reminder Email** expects:
      - `subject`
      - `toEmail`
      - `fromEmail`
  - **Inputs/Outputs:** Input from **Split per Subscription** (per reminder item) → output to **Throttle Reminders**
  - **Edge cases / failures:**
    - If it doesn’t set `toEmail`/`subject`, the send node will fail.
    - Should map `customerEmail` → `toEmail`, and include `daysLeft`, `dueDateISO`, `subscriptionName` in subject/body.

- **Throttle Reminders**
  - **Type / role:** `Wait` — introduces delay between sends to reduce rate limiting.
  - **Configuration (interpreted):**
    - Unit: **seconds**
    - The actual duration value is not visible in JSON (likely not set).
  - **Inputs/Outputs:** From **Prepare Reminder Email** → to **Send Reminder Email**
  - **Edge cases / failures:**
    - If duration is blank/zero, it may not throttle.
    - Very long waits can cause executions to remain running; consider n8n execution time limits.

- **Send Reminder Email**
  - **Type / role:** `Email Send` — sends customer reminder via SMTP.
  - **Configuration (interpreted):**
    - `toEmail`: `={{ $json.toEmail }}`
    - `fromEmail`: `={{ $json.fromEmail }}`
    - `subject`: `={{ $json.subject }}`
    - Body not explicitly configured in JSON excerpt (likely plain text/HTML defaults not set).
    - Requires SMTP credentials configured in n8n.
  - **Inputs/Outputs:** From **Throttle Reminders** → to **Log Reminder Status**
  - **Edge cases / failures:**
    - SMTP auth errors, TLS issues, rejected sender domain, missing body/subject.
    - Invalid recipient address from ClickUp custom field.
    - Provider rate limiting if throttle is insufficient.

- **Log Reminder Status**
  - **Type / role:** `Set` — placeholder to store/send status elsewhere (not currently configured).
  - **Configuration:** Empty in JSON.
  - **Inputs/Outputs:** From **Send Reminder Email**; no outgoing connections shown (end of reminder branch).
  - **Edge cases / failures:**
    - As-is, it doesn’t persist logs; execution history is the only record.

---

### 2.5 Admin Reporting + Completion
**Overview:** Sends either an admin summary when reminders were sent, or a “no expiries” email when none were required, then outputs a final completion payload.

**Nodes involved**
- **Build Admin Summary** (Code)
- **Send Admin Summary** (Email Send)
- **Prepare No-Expiry Email** (Set)
- **Send No-Expiry Email** (Email Send)
- **Cleanup Execution** (Code)

**Node details**
- **Build Admin Summary**
  - **Type / role:** `Code` — creates a simple HTML report for admin.
  - **Key variables:**
    - `total = $json.totalReminders`
    - `threshold = $json.threshold`
    - `runAt = $json.runAt`
  - **Output fields produced:**
    - `adminEmail`
    - `fromEmail` (fallback `'user@example.com'` if missing)
    - `summarySubject`
    - `summaryHtml` (HTML string)
  - **Connections:** → **Send Admin Summary**
  - **Edge cases / failures:**
    - If `totalReminders` is undefined (wrong input), subject/body becomes incorrect.
    - Uses `$json.fromEmail` but the summary input item (from Filter node output 1) doesn’t include it; you must ensure `fromEmail` is present or set in defaults.

- **Send Admin Summary**
  - **Type / role:** `Email Send` — sends report to admin.
  - **Configuration:**
    - `toEmail`: `={{ $json.adminEmail }}`
    - `fromEmail`: `={{ $json.fromEmail }}`
    - `subject`: `={{ $json.summarySubject }}`
    - Body should use `summaryHtml`, but JSON doesn’t show body mapping; you should ensure HTML/text body is configured.
  - **Connections:** → **Cleanup Execution**
  - **Edge cases / failures:** Same SMTP considerations as reminder email.

- **Prepare No-Expiry Email**
  - **Type / role:** `Set` — prepares “nothing due” notification.
  - **Configuration:** Empty in JSON; should set `toEmail`, `fromEmail`, `subject`, and body.
  - **Connections:** → **Send No-Expiry Email**
  - **Edge cases:** If admin email not defined, this cannot be delivered.

- **Send No-Expiry Email**
  - **Type / role:** `Email Send` — sends “no expiries” notice.
  - **Configuration:**
    - `toEmail`: `={{ $json.toEmail }}`
    - `fromEmail`: `={{ $json.fromEmail }}`
    - `subject`: `={{ $json.subject }}`
  - **Connections:** → **Cleanup Execution**
  - **Edge cases:** Same SMTP considerations.

- **Cleanup Execution**
  - **Type / role:** `Code` — final “execution completed” marker.
  - **Logic:** outputs `{ status: 'completed', finishedAt: ISOString }`
  - **Connections:** None (end)
  - **Edge cases:** None significant.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| 📒 Workflow Overview | Sticky Note | Documentation / operator guidance | — | — | ## How it works … (includes setup steps; ClickUp List with **due_date** + custom field **Email**; configure SMTP; call webhook /subscription-renewal) |
| Trigger & Data Fetch | Sticky Note | Documentation of trigger + ClickUp fetch block | — | — | ## Trigger & Data Fetch … mentions ClickUp list retrieval and List-ID in Set Defaults |
| Processing & Filtering | Sticky Note | Documentation of code filtering + summary split | — | — | ## Processing & Filtering … describes two outputs: reminders + summary |
| Notifications & Reporting | Sticky Note | Documentation of admin reporting paths | — | — | ## Notifications & Reporting … summary vs nothing due; final cleanup |
| Incoming Request | Webhook | Entry trigger endpoint | — | Set Defaults | Trigger & Data Fetch … |
| Set Defaults | Set | Define list/admin/sender/threshold defaults | Incoming Request | Filter Expiring & Build Data | Trigger & Data Fetch … |
| Filter Expiring & Build Data | Code | Compute daysLeft; filter expiring; build summary output | Set Defaults | Split per Subscription | Processing & Filtering … |
| Split per Subscription | SplitInBatches | Iterate reminders; provide “done” signal | Filter Expiring & Build Data | Prepare Reminder Email; Any Reminders Sent? | Processing & Filtering … |
| Prepare Reminder Email | Set | Create per-customer email fields | Split per Subscription | Throttle Reminders | Processing & Filtering … |
| Throttle Reminders | Wait | Delay between reminder sends | Prepare Reminder Email | Send Reminder Email | Processing & Filtering … |
| Send Reminder Email | Email Send | Send customer renewal reminder | Throttle Reminders | Log Reminder Status | Processing & Filtering … |
| Log Reminder Status | Set | Placeholder for logging outcome | Send Reminder Email | — | Processing & Filtering … |
| Any Reminders Sent? | IF | Branch: summary vs nothing due | Split per Subscription | Build Admin Summary; Prepare No-Expiry Email | Notifications & Reporting … |
| Build Admin Summary | Code | Build admin HTML summary | Any Reminders Sent? | Send Admin Summary | Notifications & Reporting … |
| Send Admin Summary | Email Send | Email report to admin | Build Admin Summary | Cleanup Execution | Notifications & Reporting … |
| Prepare No-Expiry Email | Set | Create “nothing due” email fields | Any Reminders Sent? | Send No-Expiry Email | Notifications & Reporting … |
| Send No-Expiry Email | Email Send | Email “nothing due” to admin | Prepare No-Expiry Email | Cleanup Execution | Notifications & Reporting … |
| Cleanup Execution | Code | End marker / final output | Send Admin Summary; Send No-Expiry Email | — | Notifications & Reporting … |

---

## 4. Reproducing the Workflow from Scratch

1) **Create the Webhook trigger**
   - Add node: **Webhook**
   - Name: **Incoming Request**
   - Method: **POST**
   - Path: `subscription-renewal`

2) **Add defaults/config node**
   - Add node: **Set**
   - Name: **Set Defaults**
   - Add fields (recommended to match the sticky note intent and downstream usage):
     - `daysThreshold` (Number) = `7`
     - `adminEmail` (String) = `ops@yourdomain.com`
     - `fromEmail` (String) = `billing@yourdomain.com`
     - `clickupListId` (String/Number) = `<YOUR_CLICKUP_LIST_ID>`
   - Connect: **Incoming Request → Set Defaults**

3) **(Missing in provided JSON) Add ClickUp task retrieval**
   - Add node: **ClickUp**
   - Operation: **Task → Get Many** (or “List tasks” depending on node version)
   - Configure credentials: ClickUp API token/OAuth
   - Use List ID: `={{ $json.clickupListId }}`
   - Ensure returned fields include `id`, `name`, `due_date`, `custom_fields`
   - Connect: **Set Defaults → ClickUp (Get Many)**
   - Note: The provided workflow’s notes assume this exists, but the JSON does not include it.

4) **Add filtering/processing code**
   - Add node: **Code**
   - Name: **Filter Expiring & Build Data**
   - Implement logic equivalent to:
     - Read `daysThreshold`, `adminEmail`, `fromEmail`
     - For each ClickUp task item:
       - Skip if no `due_date`
       - Compute `daysLeft`
       - Include if `0 <= daysLeft <= threshold` (recommended to avoid past-due spam)
       - Extract custom field **Email**
     - Return two outputs: reminders array + single summary item
   - Connect: **ClickUp (Get Many) → Filter Expiring & Build Data**
   - In the node, ensure **two outputs** are used.

5) **Wire outputs correctly (important)**
   - Connect **Filter Expiring & Build Data (Output 0)** → **Split In Batches** (Split per Subscription)
   - Connect **Filter Expiring & Build Data (Output 1)** → **Any Reminders Sent?** (recommended), OR store summary for later merge.
   - If you want to keep the “Any Reminders Sent?” driven by SplitInBatches completion, then ensure SplitInBatches “done” branch receives/has access to the summary (typically requires a merge strategy). The provided wiring does not guarantee that.

6) **Add reminder batching**
   - Add node: **Split In Batches**
   - Name: **Split per Subscription**
   - Set Batch Size (recommended): `1`
   - Connect: **Filter Expiring & Build Data (reminders output) → Split per Subscription**

7) **Prepare per-customer email**
   - Add node: **Set**
   - Name: **Prepare Reminder Email**
   - Set fields (example):
     - `toEmail` = `={{ $json.customerEmail }}`
     - `fromEmail` = `={{ $json.fromEmail }}`
     - `subject` = `={{ 'Renewal reminder: ' + $json.subscriptionName + ' (' + $json.daysLeft + ' days left)' }}`
     - `bodyHtml` = build HTML with `subscriptionName`, `dueDateISO`, `daysLeft`
   - Connect: **Split per Subscription → Prepare Reminder Email**

8) **Throttle**
   - Add node: **Wait**
   - Name: **Throttle Reminders**
   - Mode: wait **X seconds** (e.g., `2`)
   - Connect: **Prepare Reminder Email → Throttle Reminders**

9) **Send reminder emails**
   - Add node: **Email Send**
   - Name: **Send Reminder Email**
   - Credentials: configure **SMTP** in n8n
   - Map:
     - To: `={{ $json.toEmail }}`
     - From: `={{ $json.fromEmail }}`
     - Subject: `={{ $json.subject }}`
     - HTML/Text body: map from your prepared field (e.g., `bodyHtml`)
   - Connect: **Throttle Reminders → Send Reminder Email**

10) **Optional logging**
   - Add node: **Set**
   - Name: **Log Reminder Status**
   - (Optional) store `taskId`, `toEmail`, `sentAt`
   - Connect: **Send Reminder Email → Log Reminder Status**

11) **Decision: summary vs no-expiry**
   - Add node: **IF**
   - Name: **Any Reminders Sent?**
   - Condition: `totalReminders > 0`
   - Ensure it receives the **summary** item (either directly from output 1 of the filter code node, or via a merge design).
   - Connect TRUE → **Build Admin Summary**
   - Connect FALSE → **Prepare No-Expiry Email**

12) **Build and send admin summary**
   - Add node: **Code** (Build Admin Summary) to produce `summarySubject`, `summaryHtml`, `adminEmail`, `fromEmail`
   - Add node: **Email Send** (Send Admin Summary) mapped to those fields
   - Connect: **Build Admin Summary → Send Admin Summary**

13) **Build and send “no expiry” notice**
   - Add node: **Set** (Prepare No-Expiry Email) to set:
     - `toEmail` = admin address
     - `fromEmail`
     - `subject` = “No subscriptions due for renewal”
     - body
   - Add node: **Email Send** (Send No-Expiry Email)
   - Connect: **Prepare No-Expiry Email → Send No-Expiry Email**

14) **Finalize**
   - Add node: **Code**
   - Name: **Cleanup Execution**
   - Output a completion payload
   - Connect both:
     - **Send Admin Summary → Cleanup Execution**
     - **Send No-Expiry Email → Cleanup Execution**

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Create a ClickUp List with tasks including `due_date` and a custom field named **Email** | From sticky note “📒 Workflow Overview” |
| Store List-ID and email addresses in “Set Defaults” for easy maintenance | From sticky notes |
| Configure SMTP credentials in n8n and use them in both Email Send nodes | From sticky note “📒 Workflow Overview” |
| Webhook endpoint `/subscription-renewal` can be called by any scheduler/external system | From sticky note “📒 Workflow Overview” |

### Integrity notes (important discrepancies in provided JSON)
- The sticky notes describe a **ClickUp list fetch**, but **no ClickUp node exists** in the workflow JSON.
- The filtering code uses `$json[0]....`, which typically won’t work as intended; defaults/admin email should be read from the current item (e.g., `$json.daysThreshold`) or from a known merged config item.
- The code node produces **two outputs**, but only output 0 is wired; summary output is not connected, so admin decision logic may not function as described without rewiring/merge adjustments.