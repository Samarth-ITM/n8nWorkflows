Send a daily sales report to WhatsApp from Google Sheets with MoltFlow

https://n8nworkflows.xyz/workflows/send-a-daily-sales-report-to-whatsapp-from-google-sheets-with-moltflow-13486


# Send a daily sales report to WhatsApp from Google Sheets with MoltFlow

Disclaimer: Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** Send a daily WhatsApp message summarizing today’s sales stored in a Google Sheet. The workflow runs automatically every day at **18:00**, reads rows from a **Sales** sheet, computes key KPIs (order count, revenue, top product), formats a report message, and sends it through **MoltFlow (waiflow.app)** via HTTP API.

**Target use cases**
- Daily sales digest for small businesses/shops
- WhatsApp-based reporting for teams without dashboards
- Lightweight reporting from Google Sheets without additional BI tools

### 1.1 Scheduling / Trigger
Runs on a cron schedule (every day at 6 PM).

### 1.2 Data Retrieval (Google Sheets)
Reads all rows from a given Google Sheet tab (“Sales”).

### 1.3 Report Computation & Formatting
Filters rows for “today”, computes totals, determines the top product, and constructs a WhatsApp-ready message payload.

### 1.4 WhatsApp Delivery via MoltFlow API
Posts the message payload to MoltFlow’s message-send endpoint using an API key.

### 1.5 Completion Marker
Outputs a final status object (useful for testing/logging).

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduling / Trigger

**Overview:** Starts the workflow automatically every day at 18:00 using a cron expression.

**Nodes involved**
- **Every Day 6 PM** (Schedule Trigger)

#### Node: Every Day 6 PM
- **Type / role:** `n8n-nodes-base.scheduleTrigger` — time-based workflow entry point.
- **Configuration (interpreted):**
  - Runs with cron: **`0 18 * * *`** (daily at 18:00, server time).
- **Inputs / outputs:**
  - **Input:** none (trigger node).
  - **Output:** triggers one execution to **Read Sales Data**.
- **Version notes:** Node typeVersion **1.2**; cron field is standard for Schedule Trigger.
- **Edge cases / failures:**
  - Timezone mismatch (n8n instance timezone vs your expected business timezone).
  - If n8n is down at 18:00, the run may be skipped unless you implement catch-up logic (not present here).

---

### Block 2 — Data Retrieval (Google Sheets)

**Overview:** Reads sales rows from a Google Sheet tab named “Sales”.

**Nodes involved**
- **Read Sales Data** (Google Sheets)

#### Node: Read Sales Data
- **Type / role:** `n8n-nodes-base.googleSheets` — fetches spreadsheet data.
- **Configuration (interpreted):**
  - **Operation:** Read
  - **Document:** Provided by **URL** (`YOUR_GOOGLE_SHEET_URL`)
  - **Sheet/tab name:** `Sales`
  - **Options:** default/empty
- **Key expressions / variables:** none
- **Inputs / outputs:**
  - **Input:** from **Every Day 6 PM**
  - **Output:** list of rows (each row becomes an item) to **Build Report**
- **Credentials:**
  - Uses **Google Sheets OAuth2** credential named “Google Sheets”
- **Version notes:** typeVersion **4.5** (modern Google Sheets node behavior).
- **Edge cases / failures:**
  - OAuth2 not connected or expired → authentication error.
  - Wrong Sheet URL / access denied → 404/403.
  - Tab name “Sales” missing → runtime error.
  - Data types: dates and amounts can arrive as strings; downstream code assumes parseable formats.

**Sticky note context (applies to this area):**
- “Setup (5 min)… Connect Google Sheets OAuth2 credential… Paste your Sheet URL in the Google Sheets node…”

---

### Block 3 — Report Computation & Formatting

**Overview:** Filters sheet rows to only those dated “today”, calculates revenue and top product, and builds the exact JSON payload required by MoltFlow.

**Nodes involved**
- **Build Report** (Code)

#### Node: Build Report
- **Type / role:** `n8n-nodes-base.code` — transforms and aggregates rows into a single report item.
- **Configuration (interpreted):**
  - **Mode:** `runOnceForAllItems` (processes all incoming rows at once)
  - Hardcoded constants:
    - `SESSION_ID = 'YOUR_SESSION_ID'`
    - `MY_PHONE = 'YOUR_PHONE'`
  - Determines “today” as: `new Date().toISOString().split('T')[0]` (UTC date string `YYYY-MM-DD`)
  - Filters today’s rows by checking if the row’s `Date` (or `date`) **startsWith(today)**.
  - If no sales today → message “No sales recorded today.”
  - Otherwise:
    - Sums `Amount` (or `amount`) via `parseFloat(...)`
    - Counts products from `Product` (or `product`)
    - Selects top product by highest count
  - Outputs a single item with:
    - `session_id`
    - `chat_id` = `MY_PHONE + '@c.us'`
    - `message` (multi-line text)
- **Key expressions / variables used:**
  - `$input.all()` (reads all items)
  - `$json` is not used inside the code; it builds a new JSON output.
- **Inputs / outputs:**
  - **Input:** items from **Read Sales Data**
  - **Output:** one item to **Send Report via WhatsApp**
- **Version notes:** typeVersion **2** for Code node.
- **Edge cases / failures:**
  - **Timezone mismatch:** `toISOString()` uses UTC, but your sheet “Date” may be local; filtering may miss late-day sales depending on timezone.
  - **Date formatting mismatch:** If the sheet stores dates like `MM/DD/YYYY`, `DD/MM/YYYY`, or as Google serial numbers, `startsWith('YYYY-MM-DD')` may fail and yield “No sales recorded today.”
  - **Amount parsing:** currency symbols or commas (e.g., `€1.234,50`) can break `parseFloat`.
  - **Top product when counts empty:** protected by the earlier `todaySales.length === 0` check, so it won’t sort an empty array.
  - **Phone format:** MoltFlow/WhatsApp often expects an international format; missing country code may fail delivery.
- **Sticky note context:**
  - “Set `YOUR_SESSION_ID` and `YOUR_PHONE` in the Build Report node”

---

### Block 4 — WhatsApp Delivery via MoltFlow API

**Overview:** Sends the prepared report to MoltFlow’s API endpoint using HTTP Header authentication.

**Nodes involved**
- **Send Report via WhatsApp** (HTTP Request)

#### Node: Send Report via WhatsApp
- **Type / role:** `n8n-nodes-base.httpRequest` — calls MoltFlow’s message-send API.
- **Configuration (interpreted):**
  - **Method:** POST
  - **URL:** `https://apiv2.waiflow.app/api/v2/messages/send`
  - **Authentication:** Generic credential → **HTTP Header Auth**
  - **Body:** JSON (sent as body)
  - **JSON body expression:**
    - `={{ JSON.stringify({ session_id: $json.session_id, chat_id: $json.chat_id, message: $json.message }) }}`
- **Inputs / outputs:**
  - **Input:** single report item from **Build Report**
  - **Output:** API response to **Done**
- **Credentials:**
  - `httpHeaderAuth` credential named **“MoltFlow API Key”**
  - Sticky note indicates header key should be: **`X-API-Key`**
- **Version notes:** typeVersion **4.2**
- **Edge cases / failures:**
  - Wrong/missing API key → 401/403.
  - Invalid `session_id` (not connected session) → API error.
  - Invalid `chat_id` format → message not delivered.
  - Payload formatting risk: because the body is built with `JSON.stringify(...)`, ensure the node is truly sending raw JSON (it is configured as JSON). If MoltFlow expects an object and n8n sends a string, it can fail depending on node behavior; if you see issues, replace with a direct JSON object body mapping instead of stringifying.
  - Network timeouts / transient 5xx errors (no retry logic configured here).

---

### Block 5 — Completion Marker

**Overview:** Produces a simple “report_sent” status output (useful for execution logs and testing).

**Nodes involved**
- **Done** (Code)

#### Node: Done
- **Type / role:** `n8n-nodes-base.code` — emits a final status item.
- **Configuration (interpreted):**
  - **Mode:** `runOnceForAllItems`
  - Returns:
    - `status: 'report_sent'`
    - `timestamp: <ISO datetime>`
- **Inputs / outputs:**
  - **Input:** from **Send Report via WhatsApp**
  - **Output:** end of workflow
- **Version notes:** typeVersion **2**
- **Edge cases / failures:**
  - None significant; only fails if the Code node itself errors (rare with this simple code).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | Sticky Note | Documentation / on-canvas description | — | — | ## Daily Sales Report → WhatsApp; Get a daily WhatsApp summary of your sales data from Google Sheets, powered by [MoltFlow](https://molt.waiflow.app).; **How it works:**; 1. Runs every day at 6 PM; 2. Reads today's sales from Google Sheets; 3. Calculates total revenue, order count, and top product; 4. Sends a formatted summary to your WhatsApp |
| Sticky Note1 | Sticky Note | Documentation / setup checklist | — | — | ## Setup (5 min); 1. Create a [MoltFlow account](https://molt.waiflow.app) and connect WhatsApp; 2. Create a Google Sheet with columns: **Date**, **Product**, **Amount**, **Customer**; 3. Connect Google Sheets OAuth2 credential; 4. Paste your Sheet URL in the Google Sheets node; 5. Set `YOUR_SESSION_ID` and `YOUR_PHONE` in the Build Report node; 6. Add MoltFlow API Key: Header Auth → `X-API-Key` |
| Every Day 6 PM | Schedule Trigger | Daily scheduled start | — | Read Sales Data |  |
| Read Sales Data | Google Sheets | Read sales rows from “Sales” tab | Every Day 6 PM | Build Report |  |
| Build Report | Code | Filter today’s rows; compute KPIs; build WhatsApp payload | Read Sales Data | Send Report via WhatsApp |  |
| Send Report via WhatsApp | HTTP Request | Send message via MoltFlow API | Build Report | Done |  |
| Done | Code | Emit completion status | Send Report via WhatsApp | — |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
2. **Add Schedule Trigger**
   - Node: **Schedule Trigger**
   - Name: `Every Day 6 PM`
   - Set **Cron** expression to: `0 18 * * *`
3. **Add Google Sheets node**
   - Node: **Google Sheets**
   - Name: `Read Sales Data`
   - **Credentials:** create/select **Google Sheets OAuth2** credential (sign in, grant access).
   - **Operation:** `Read`
   - **Document:** choose input mode **URL**, paste your Google Sheet URL (replace `YOUR_GOOGLE_SHEET_URL`)
   - **Sheet name:** `Sales`
4. **Connect** `Every Day 6 PM` → `Read Sales Data`.
5. **Add Code node (report builder)**
   - Node: **Code**
   - Name: `Build Report`
   - **Mode:** `Run Once for All Items`
   - Paste and adjust:
     - Set `SESSION_ID` to your MoltFlow session id (from MoltFlow/waiflow).
     - Set `MY_PHONE` to the destination phone number (preferably in international format, digits only).
     - Keep `chat_id` formatting as `MY_PHONE + '@c.us'` unless MoltFlow instructs otherwise.
   - Keep the logic that:
     - filters on `Date`/`date`
     - reads `Amount`/`amount`
     - reads `Product`/`product`
6. **Connect** `Read Sales Data` → `Build Report`.
7. **Add HTTP Request node (MoltFlow send)**
   - Node: **HTTP Request**
   - Name: `Send Report via WhatsApp`
   - **Method:** `POST`
   - **URL:** `https://apiv2.waiflow.app/api/v2/messages/send`
   - **Authentication:** `Generic Credential Type` → `HTTP Header Auth`
   - Create credential named (example) **MoltFlow API Key**:
     - Header name: `X-API-Key`
     - Value: your MoltFlow API key
   - **Send Body:** enabled
   - **Body Content Type:** JSON
   - **JSON body:** map fields from the previous node:
     - `session_id` = `{{$json.session_id}}`
     - `chat_id` = `{{$json.chat_id}}`
     - `message` = `{{$json.message}}`
     - (If you keep the original workflow exactly, it uses a `JSON.stringify(...)` expression; if you experience API issues, switch to direct JSON field mapping as above.)
8. **Connect** `Build Report` → `Send Report via WhatsApp`.
9. **Add final Code node**
   - Node: **Code**
   - Name: `Done`
   - **Mode:** `Run Once for All Items`
   - Code: return a simple status object with timestamp (as in the workflow).
10. **Connect** `Send Report via WhatsApp` → `Done`.
11. **Test**
   - Run manually once (with sample sheet rows for today).
   - Check n8n execution data:
     - `Build Report` output contains `session_id`, `chat_id`, `message`.
     - HTTP node returns success from MoltFlow.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| MoltFlow service referenced in the workflow notes | https://molt.waiflow.app |
| MoltFlow message send endpoint used by the HTTP Request node | `https://apiv2.waiflow.app/api/v2/messages/send` |
| Expected Google Sheet columns | `Date`, `Product`, `Amount`, `Customer` |
| Important setup parameters to replace | `YOUR_GOOGLE_SHEET_URL`, `YOUR_SESSION_ID`, `YOUR_PHONE`, and the `X-API-Key` credential |