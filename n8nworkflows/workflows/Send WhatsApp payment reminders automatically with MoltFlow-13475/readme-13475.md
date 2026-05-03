Send WhatsApp payment reminders automatically with MoltFlow

https://n8nworkflows.xyz/workflows/send-whatsapp-payment-reminders-automatically-with-moltflow-13475


# Send WhatsApp payment reminders automatically with MoltFlow

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow name:** *WhatsApp Payment Reminder via MoltFlow*  
**Purpose:** Send personalized WhatsApp payment reminders automatically every day at **9:00 AM** using the **MoltFlow (Waiflow) API**.  
**Primary use cases:** Small business invoicing follow-ups, recurring payment reminders, collections workflows, or daily dunning messages.

### 1.1 Scheduled Execution (Entry Point)
Runs daily at 9 AM and starts the workflow.

### 1.2 Contact & Message Preparation
Builds a list of contacts and generates a personalized message per contact (templating with name, amount, due date).

### 1.3 WhatsApp Delivery via MoltFlow API
Sends each reminder through an HTTP POST to MoltFlow’s message endpoint using an API key.

### 1.4 Result Classification & Logging
Checks if the API response indicates success (HTTP 201). Routes to success or failure logging blocks and aggregates a summary output.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduled Execution (Entry Point)

**Overview:** Triggers the workflow every day at 9 AM. This is the only entry point.  
**Nodes involved:** `Daily 9 AM Trigger`

#### Node: Daily 9 AM Trigger
- **Type / role:** `Schedule Trigger` — time-based trigger to start the workflow.
- **Configuration (interpreted):**
  - Runs on an interval rule set to fire at **hour = 9** (daily at 09:00).
- **Input / output:**
  - **Input:** None (trigger node).
  - **Output:** One item emitted at schedule time → `Prepare Contacts`.
- **Version-specific notes:** `typeVersion 1.2` (standard schedule trigger behavior).
- **Potential failures / edge cases:**
  - Timezone considerations: schedule uses the workflow/server timezone (n8n instance setting). If you expect 9 AM in a specific locale, ensure instance timezone is correct.

---

### Block 2 — Contact & Message Preparation

**Overview:** Generates multiple items (one per contact) with `session_id`, WhatsApp `chat_id`, and a personalized reminder message.  
**Nodes involved:** `Prepare Contacts`

#### Node: Prepare Contacts
- **Type / role:** `Code` — constructs outbound message payloads.
- **Configuration (interpreted):**
  - Runs in **runOnceForAllItems** mode (it ignores incoming items and returns a new array of items).
  - Hardcoded variables:
    - `SESSION_ID = 'YOUR_SESSION_ID'`
    - `contacts = [...]` list of objects with `phone`, `name`, `amount`, `due_date`
  - Message templating:
    - `messageTemplate` contains placeholders `{{name}}`, `{{amount}}`, `{{due_date}}`
    - Replaces placeholders via chained `.replace()`.
  - Produces items shaped like:
    - `session_id`
    - `chat_id` = `phone + '@c.us'` (WhatsApp ID format used by many providers)
    - `message`
    - `contact_name`, `contact_phone` (for later logging)
- **Key variables/expressions:**
  - `chat_id: contact.phone + '@c.us'`
- **Input / output:**
  - **Input:** From trigger (not used).
  - **Output:** Multiple items → `Send WhatsApp Reminder` (one HTTP call per item).
- **Version-specific notes:** `typeVersion 2` Code node (supports `$input`, `$json`, etc.).
- **Potential failures / edge cases:**
  - `SESSION_ID` left as `YOUR_SESSION_ID` will cause API failures.
  - Phone format: if MoltFlow expects E.164 (e.g., `+33...`) or requires country code, sending raw digits may fail.
  - Template replacement is single-pass per placeholder. If placeholders appear multiple times, chained `.replace()` only replaces the first occurrence of each token; use regex or `replaceAll` if needed.
  - Hardcoded contacts: in real use, you likely want a DB/Sheet/CRM source; large lists increase runtime and API load.

---

### Block 3 — WhatsApp Delivery via MoltFlow API

**Overview:** Sends each prepared message to MoltFlow using HTTP Header Authentication (API key). Continues execution even if a request errors.  
**Nodes involved:** `Send WhatsApp Reminder`

#### Node: Send WhatsApp Reminder
- **Type / role:** `HTTP Request` — calls MoltFlow message send endpoint.
- **Configuration (interpreted):**
  - **Method:** POST
  - **URL:** `https://apiv2.waiflow.app/api/v2/messages/send`
  - **Authentication:** Generic credential type → `httpHeaderAuth`
    - Credential expected: header `X-API-Key: <your_key>` (per sticky note)
  - **Body:** JSON body built via expression:
    - `session_id`, `chat_id`, `message`
  - **Response handling:**
    - Requests full response (`fullResponse: true`) so output includes status code and body.
  - **Error handling:** `onError: continueRegularOutput`
    - The node won’t stop the workflow on HTTP errors; it will output error-like responses for downstream handling.
- **Key expressions/variables:**
  - Body is built as:
    - `={{ JSON.stringify({ session_id: $json.session_id, chat_id: $json.chat_id, message: $json.message }) }}`
- **Input / output:**
  - **Input:** Items from `Prepare Contacts` (one per contact).
  - **Output:** To `Message Sent?` with response metadata (notably `statusCode` and `body`).
- **Version-specific notes:** `typeVersion 4.2` HTTP Request node (supports full response option and modern auth patterns).
- **Potential failures / edge cases:**
  - Invalid API key → 401/403.
  - Invalid/expired `session_id` → provider-specific error (often 400/404).
  - Wrong `chat_id` format → message not delivered / API rejects.
  - Rate limiting / throttling → 429.
  - Provider downtime / network timeout.
  - The body is manually stringified; if n8n expects an object in JSON mode, stringifying can sometimes lead to double-encoding depending on node behavior. If you see API receiving a quoted JSON string instead of JSON, switch to passing an object directly (no `JSON.stringify`).

---

### Block 4 — Result Classification & Logging

**Overview:** Splits items into success vs failure based on HTTP status code, then aggregates results into a summary output per branch.  
**Nodes involved:** `Message Sent?`, `Log Success`, `Log Failure`

#### Node: Message Sent?
- **Type / role:** `IF` — conditionally routes execution based on HTTP status.
- **Configuration (interpreted):**
  - Condition: `$json.statusCode == 201`
  - Strict type validation enabled (number equals).
- **Key expressions/variables:**
  - `leftValue: ={{ $json.statusCode }}`
- **Input / output:**
  - **Input:** Output from `Send WhatsApp Reminder`.
  - **Output (true):** → `Log Success`
  - **Output (false):** → `Log Failure`
- **Version-specific notes:** `typeVersion 2` IF node.
- **Potential failures / edge cases:**
  - If `statusCode` is missing (some error modes), strict validation may behave unexpectedly; typically it will evaluate false and go to failure branch.
  - Some APIs return 200 instead of 201 on success; if MoltFlow changes behavior, successes may be misclassified.

#### Node: Log Success
- **Type / role:** `Code` — aggregates all successful items into a single summary.
- **Configuration (interpreted):**
  - Runs **runOnceForAllItems** to aggregate.
  - Extracts `data` from `item.json.body || item.json`.
  - Produces result objects with:
    - `status: 'sent'`
    - `contact` from `item.json.contact_name`
    - `message_id` from `data.id`
    - `wa_status` from `data.status`
  - Returns one item:
    - `summary: "Successfully sent X payment reminder(s)"`
    - `results: [...]`
- **Input / output:**
  - **Input:** True branch items from `Message Sent?`.
  - **Output:** Final aggregated output for successes (no further nodes).
- **Potential failures / edge cases:**
  - If API response body schema differs, `data.id`/`data.status` may be missing → outputs `N/A`.
  - If items are empty (no successes), this node may not run (depending on n8n behavior when a branch has zero items).

#### Node: Log Failure
- **Type / role:** `Code` — aggregates failed items into a single summary.
- **Configuration (interpreted):**
  - Runs **runOnceForAllItems** to aggregate.
  - Extracts `data` from `item.json.body || item.json`.
  - Produces failure objects with:
    - `status: 'failed'`
    - `contact` from `item.json.contact_name`
    - `error` from `data.detail || data.message || ('HTTP ' + item.json.statusCode)`
    - `status_code` from `item.json.statusCode`
  - Returns one item with:
    - `summary: "Failed to send X reminder(s)"`
    - `failures: [...]`
- **Input / output:**
  - **Input:** False branch items from `Message Sent?`.
  - **Output:** Final aggregated output for failures (no further nodes).
- **Potential failures / edge cases:**
  - Some HTTP failures may not include `body.detail` or `body.message`; fallback error becomes `HTTP undefined` if statusCode is absent.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | Sticky Note | Documentation / inline instructions | — | — | ## WhatsApp Payment Reminder; Automatically sends personalized WhatsApp payment reminders every morning at 9 AM using the [MoltFlow API](https://molt.waiflow.app).; **How it works:**; 1. Schedule fires daily at 9 AM; 2. Code node builds a contact list with payment details; 3. Each contact gets a personalized WhatsApp message; 4. Results are logged as success or failure |
| Sticky Note1 | Sticky Note | Setup instructions | — | — | ## Setup (3 min); 1. Create a [MoltFlow account](https://molt.waiflow.app); 2. Connect WhatsApp (scan QR); 3. Generate API key > Sessions > API Keys tab; 4. Add credential: Header Auth > Name: X-API-Key > Value: your key; 5. Edit Prepare Contacts with your data; 6. Activate! |
| Daily 9 AM Trigger | Schedule Trigger | Daily scheduled entry point | — | Prepare Contacts |  |
| Prepare Contacts | Code | Build contact items + personalized messages | Daily 9 AM Trigger | Send WhatsApp Reminder |  |
| Send WhatsApp Reminder | HTTP Request | Send WhatsApp message via MoltFlow API | Prepare Contacts | Message Sent? |  |
| Message Sent? | IF | Classify success vs failure by HTTP status | Send WhatsApp Reminder | Log Success (true), Log Failure (false) |  |
| Log Success | Code | Aggregate successful sends into a summary | Message Sent? (true) | — |  |
| Log Failure | Code | Aggregate failed sends into a summary | Message Sent? (false) | — |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **WhatsApp Payment Reminder via MoltFlow**
   - (Optional) Add tags: `whatsapp`, `automation`, `moltflow`, `payments`, `reminders`

2. **Add Sticky Note (overview)**
   - Node: **Sticky Note**
   - Content (paste):
     - “WhatsApp Payment Reminder…” including the link to `https://molt.waiflow.app`

3. **Add Sticky Note (setup)**
   - Node: **Sticky Note**
   - Content (paste):
     - Setup steps including header auth name `X-API-Key`

4. **Add trigger**
   - Node: **Schedule Trigger**
   - Name: **Daily 9 AM Trigger**
   - Configure rule to run **daily at 09:00** (ensure timezone matches your requirement).

5. **Add contact preparation**
   - Node: **Code**
   - Name: **Prepare Contacts**
   - Mode: **Run Once for All Items**
   - Paste/implement logic:
     - Set `SESSION_ID` (replace `YOUR_SESSION_ID` with your real session id from MoltFlow)
     - Create `contacts` array (phone, name, amount, due_date)
     - Build `message` from template
     - Return items with:
       - `session_id`
       - `chat_id = phone + '@c.us'`
       - `message`
       - `contact_name`, `contact_phone`

6. **Add MoltFlow HTTP request**
   - Node: **HTTP Request**
   - Name: **Send WhatsApp Reminder**
   - Method: **POST**
   - URL: `https://apiv2.waiflow.app/api/v2/messages/send`
   - Authentication: **Generic Credential Type** → **HTTP Header Auth**
   - Create credential (in n8n Credentials):
     - Type: **HTTP Header Auth**
     - Header name: **X-API-Key**
     - Value: **your MoltFlow API key**
   - Body:
     - Set **Send Body = true**
     - Body content type = **JSON**
     - Include fields: `session_id`, `chat_id`, `message`
   - Options:
     - Enable **Full Response** so `statusCode` is available for the IF node.
   - Error handling:
     - Set node setting **On Error** to **Continue (regular output)**.

7. **Add status check**
   - Node: **IF**
   - Name: **Message Sent?**
   - Condition:
     - Left: expression `{{$json.statusCode}}`
     - Operator: **equals**
     - Right: `201` (number)

8. **Add success logger**
   - Node: **Code**
   - Name: **Log Success**
   - Mode: **Run Once for All Items**
   - Implement aggregation to output one item with:
     - `summary`
     - `results[]` containing contact name and any id/status from response body.

9. **Add failure logger**
   - Node: **Code**
   - Name: **Log Failure**
   - Mode: **Run Once for All Items**
   - Implement aggregation to output one item with:
     - `summary`
     - `failures[]` containing contact name, status code, and best-effort error message.

10. **Connect nodes in order**
    - `Daily 9 AM Trigger` → `Prepare Contacts`
    - `Prepare Contacts` → `Send WhatsApp Reminder`
    - `Send WhatsApp Reminder` → `Message Sent?`
    - `Message Sent?` (true) → `Log Success`
    - `Message Sent?` (false) → `Log Failure`

11. **Activate workflow**
    - Ensure:
      - MoltFlow session is valid and WhatsApp is connected
      - API key credential is selected on the HTTP node
      - Contacts list and phone formats are correct

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| MoltFlow API / product site | https://molt.waiflow.app |
| Message send endpoint used by this workflow | `https://apiv2.waiflow.app/api/v2/messages/send` |
| Authentication requirement (header) | `X-API-Key: <your_key>` (from sticky note instructions) |
| Operational behavior summary | Daily schedule → build contact items → send WhatsApp message per item → route by HTTP 201 → aggregate success/failure summaries |