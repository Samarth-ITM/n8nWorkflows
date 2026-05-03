Detect WooCommerce return surges in real-time with Slack alerts & Airtable logging

https://n8nworkflows.xyz/workflows/detect-woocommerce-return-surges-in-real-time-with-slack-alerts---airtable-logging-11945


# Detect WooCommerce return surges in real-time with Slack alerts & Airtable logging

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Detect WooCommerce return surges in real-time with Slack alerts & Airtable logging  
**Workflow name (in JSON):** Real-time using WooCommerce Webhook (note: it actually runs on a schedule, not a webhook)

**Purpose:**  
This workflow monitors WooCommerce refunds on a recurring schedule, detects unusual SKU-level return spikes by comparing two rolling 24-hour windows, then alerts a Slack channel and logs the alert details to Airtable for historical tracking.

**Target use cases**
- Operations / QA teams monitoring product issues (packaging defects, batch problems)
- Customer support escalation when returns spike for specific SKUs
- Building an incident history in Airtable to correlate with suppliers, fulfillment centers, or campaigns

### 1.1 Scheduling & Time Window Setup
Runs hourly and computes the current and previous 24-hour time windows.

### 1.2 WooCommerce Data Collection
Fetches orders and refunds from WooCommerce REST API.

### 1.3 Refund-to-Order Mapping (SKU extraction)
Joins refunds back to their parent orders and emits per-line-item SKU return events.

### 1.4 Aggregation & Surge Detection
Aggregates returns per SKU, computes absolute/percent change, and filters for surge thresholds.

### 1.5 Alert Enrichment, Slack Notification, and Airtable Logging
Adds metadata, sends a formatted Slack alert, parses that Slack message back into structured fields, and stores a record in Airtable.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduling & Time Window Setup
**Overview:**  
Triggers the workflow on a fixed interval and generates ISO timestamps for the current and previous rolling 24-hour windows (mainly for consistency and future query filtering).

**Nodes involved:**  
- Schedule Trigger  
- Time_window

#### Node: Schedule Trigger
- **Type / role:** `Schedule Trigger` — entry point; initiates runs automatically.
- **Configuration choices:**
  - Runs every **1 hour** (`interval` field = `hours`).
- **Inputs / outputs:**
  - **Input:** none (trigger node)
  - **Output:** single item into `Time_window`
- **Version notes:** typeVersion **1.2**
- **Failure/edge cases:**
  - If n8n instance is paused/offline, scheduled runs won’t occur.

#### Node: Time_window
- **Type / role:** `Code` — computes time window boundaries.
- **Configuration choices (interpreted):**
  - Defines `WINDOW_HOURS = 24`
  - Computes:
    - `currentStart = now - 24h`
    - `currentEnd = now`
    - `prevStart = now - 48h`
    - `prevEnd = now - 24h`
  - Formats to ISO-like UTC strings without milliseconds.
- **Key variables produced:**
  - `$json.timeWindow.currentStart/currentEnd/prevStart/prevEnd`
- **Inputs / outputs:**
  - **Input:** trigger item
  - **Output:** to `HTTP Orders`
- **Version notes:** typeVersion **2**
- **Failure/edge cases:**
  - None likely; JavaScript date operations are stable.
  - Note: these computed windows are **not used** to filter WooCommerce API calls in the current workflow (potential improvement).

---

### Block 2 — WooCommerce Data Collection
**Overview:**  
Fetches WooCommerce orders first, then refunds. These datasets are later joined by `parent_id` (refund → order id).

**Nodes involved:**  
- HTTP Orders  
- HTTP Refunds

#### Node: HTTP Orders
- **Type / role:** `HTTP Request` — retrieves WooCommerce orders.
- **Configuration choices (interpreted):**
  - GET `https://{your_wocommerce_domain}/wp-json/wc/v3/orders`
  - Uses **HTTP Basic Auth** via n8n credential.
  - `sendQuery: true` but query parameters are empty (no pagination/date filter configured).
- **Key expressions/variables:** none
- **Inputs / outputs:**
  - **Input:** from `Time_window`
  - **Output:** to `HTTP Refunds`
- **Version notes:** typeVersion **4.3**
- **Failure/edge cases:**
  - 401/403 if credentials invalid or API keys lack permissions.
  - Large stores may return many orders; without pagination (`per_page`, `page`) this can:
    - miss records (WooCommerce defaults apply), or
    - cause slow/large responses depending on server settings.
  - No date filtering: workload grows with store size.

#### Node: HTTP Refunds
- **Type / role:** `HTTP Request` — retrieves WooCommerce refunds.
- **Configuration choices (interpreted):**
  - GET `https://{your_wocommerce_domain}/wp-json/wc/v3/refunds`
  - Uses same **HTTP Basic Auth** credential type.
  - No explicit query parameters (no window filtering/pagination).
  - `retryOnFail: false`
- **Inputs / outputs:**
  - **Input:** from `HTTP Orders`
  - **Output:** to `Orders_Fetch`
- **Version notes:** typeVersion **4.3**
- **Failure/edge cases:**
  - Same auth/pagination/volume concerns as orders.
  - Refund objects must include `parent_id`, `date_created_gmt`, `reason` for downstream logic.

---

### Block 3 — Refund-to-Order Mapping (SKU extraction)
**Overview:**  
Joins refunds to orders by `refund.parent_id = order.id`, then emits “return events” per order line item with the SKU and refund metadata.

**Nodes involved:**  
- Orders_Fetch

#### Node: Orders_Fetch
- **Type / role:** `Code` — merges orders + refunds and expands into SKU-level events.
- **Configuration choices (interpreted):**
  - Reads:
    - `const orders = $items("HTTP Orders")`
    - `const refunds = $items("HTTP Refunds")`
  - Builds `orderMap[orderId] = orderJson`
  - For each refund:
    - Finds parent order
    - For each order line item:
      - outputs `{ order_id, refund_id, sku, quantity, reason, refund_date_utc }`
  - SKU fallback: `li.sku || li.name || li.product_id`
- **Inputs / outputs:**
  - **Input:** execution context (it references other nodes’ items by name)
  - **Output:** to `Refund_details`
- **Version notes:** typeVersion **2**
- **Failure/edge cases:**
  - If node names change (“HTTP Orders”, “HTTP Refunds”), `$items("...")` breaks.
  - If WooCommerce responses are paginated/partial, join results are incomplete.
  - Potential overcounting: if an order has multiple line items, a single refund becomes multiple SKU events (by design). However, the next step counts **refund events**, not quantities, which may skew SKU attribution if refunds apply to only certain items.
  - `refund.date_created_gmt` must be parseable by `Date`.

---

### Block 4 — Aggregation & Surge Detection
**Overview:**  
Aggregates refund events per SKU into two 24-hour windows, calculates increases, then filters for significant surges.

**Nodes involved:**  
- Refund_details  
- If

#### Node: Refund_details
- **Type / role:** `Code` — aggregates by SKU and computes metrics.
- **Configuration choices (interpreted):**
  - `WINDOW_HOURS = 24`
  - Defines:
    - `currentStart = now - 24h`
    - `previousStart = now - 48h`
  - Aggregates into `stats[sku]`:
    - `current` count, `previous` count
    - `reasons` frequency map
  - Counts **refund records** (not item quantity): “more reliable” per author comment.
  - Calculates:
    - `absolute_increase = current - previous`
    - `percent_increase`:
      - if `previous === 0` and `current > 0` => `null` (no baseline)
      - else computed and rounded
- **Outputs (per SKU):**
  - `sku`
  - `current_returns`
  - `previous_returns`
  - `absolute_increase`
  - `percent_increase`
  - `reasons` (object)
- **Inputs / outputs:**
  - **Input:** from `Orders_Fetch`
  - **Output:** to `If`
- **Version notes:** typeVersion **2**
- **Failure/edge cases:**
  - If `refund_date_utc` missing or invalid, events are skipped.
  - `percent_increase` can be `null` when previous is 0; downstream conditions must handle that (the IF uses numeric compare and may treat null as non-number depending on n8n coercion rules).
  - Timezone: uses JS `Date()` on `date_created_gmt` (should be UTC), generally fine.

#### Node: If
- **Type / role:** `If` — surge gate.
- **Configuration choices (interpreted):**
  - Triggers **true** if either:
    - `percent_increase >= 100`, OR
    - `current_returns >= 25`
  - Uses strict type validation.
- **Key expressions:**
  - `={{ $json.percent_increase }} >= 100`
  - `={{ $json.current_returns }} >= 25`
- **Inputs / outputs:**
  - **Input:** from `Refund_details`
  - **True output:** to `Edit Fields1`
  - **False output:** not connected (items discarded)
- **Version notes:** typeVersion **2.2**
- **Failure/edge cases:**
  - If `percent_increase` is `null`, strict numeric compare may fail validation or evaluate false depending on n8n’s internal handling. The second condition (`current_returns >= 25`) still works.
  - If values are strings instead of numbers, strict validation may cause unexpected results.

---

### Block 5 — Alert Enrichment, Slack Notification, and Airtable Logging
**Overview:**  
Enriches surge items with metadata, posts a detailed Slack message, parses the Slack message text back into structured fields, and logs the alert in Airtable.

**Nodes involved:**  
- Edit Fields1  
- Send a message  
- Code in JavaScript  
- Create a record

#### Node: Edit Fields1
- **Type / role:** `Set` — adds alert metadata while keeping original fields.
- **Configuration choices (interpreted):**
  - `includeOtherFields: true`
  - Adds:
    - `status = "alerted"`
    - `run_date_utc = new Date().toISOString()`
    - `cooldown_key = sku + "_" + YYYY-MM-DD`
- **Key expressions:**
  - `={{ new Date().toISOString() }}`
  - `={{ $json.sku + "_" + new Date().toISOString().slice(0,10) }}`
- **Inputs / outputs:**
  - **Input:** If (true branch)
  - **Output:** to `Send a message`
- **Version notes:** typeVersion **3.4**
- **Failure/edge cases:**
  - `sku` missing → cooldown key becomes `"undefined_YYYY-MM-DD"`.
  - Cooldown key is computed but **not used** to suppress duplicate alerts; it’s informational only unless you add a dedupe step.

#### Node: Send a message
- **Type / role:** `Slack` — posts alert into a channel.
- **Configuration choices (interpreted):**
  - Operation: send message to a selected channel (`n8n-demo`, id `C0A3WQEKQ58`)
  - Message is dynamically built with SKU metrics and a reasons list.
  - Workflow link inclusion disabled.
- **Key expressions:**
  - Message text uses:
    - `{{$json.sku}}`, `{{$json.current_returns}}`, `{{$json.previous_returns}}`, etc.
    - `{{$json.percent_increase ?? 'N/A'}}%`
    - Reasons formatting:
      ```js
      {{ Object.entries($json.reasons ?? {})
        .map(r => r[0] + " : " + r[1])
        .join("\n") }}
      ```
- **Inputs / outputs:**
  - **Input:** from `Edit Fields1`
  - **Output:** to `Code in JavaScript` (Slack API response item)
- **Version notes:** typeVersion **2.3**
- **Failure/edge cases:**
  - Slack auth errors (invalid token, missing scopes like `chat:write`).
  - Channel not found / bot not invited to channel.
  - If `$json.reasons` is very large, message length may exceed Slack limits.
  - The next node expects `item.json.message.text` to exist in Slack response; Slack node output shape can vary by node version/config.

#### Node: Code in JavaScript
- **Type / role:** `Code` — normalizes Slack message text into structured fields for Airtable.
- **Configuration choices (interpreted):**
  - Reads Slack output text: `item.json.message?.text || ""`
  - Extracts fields using regex:
    - SKU, Increase, Current/Previous returns, Status, Run Date, Cooldown Key
  - Extracts “Reasons” block between `Reasons:\n` and `\n\nStatus:`
  - Produces:
    - `reasons_raw` (human-readable multi-line)
    - `reasons_json` (JSON string)
- **Inputs / outputs:**
  - **Input:** from `Send a message`
  - **Output:** to `Create a record`
- **Version notes:** typeVersion **2**
- **Failure/edge cases:**
  - If Slack message formatting changes, regex parsing breaks (e.g., spacing/labels differ).
  - If Slack node returns a different structure (no `message.text`), outputs become null.
  - The Increase parsing checks for `"N/A%"`, but the Slack text prints `"N/A"` then `%` is appended, so it becomes `"N/A%"`—this matches the code’s expectation (good), but any template change can break this.

#### Node: Create a record
- **Type / role:** `Airtable` — persists alert record.
- **Configuration choices (interpreted):**
  - Base: **n8n Learning** (`app6M3NseJ4VEy2Ro`)  
    Link: https://airtable.com/app6M3NseJ4VEy2Ro
  - Table: **woocommerce** (`tblT2vWZ8en2ZRdox`)  
    Link: https://airtable.com/app6M3NseJ4VEy2Ro/tblT2vWZ8en2ZRdox
  - Operation: **Create**
  - Fields mapped:
    - `sku`
    - `resons` (typo field name; populated from `reasons_raw`)
    - `cooldown_key`
    - `run_date_utc`
    - `current_returns`
    - `percent_increase`
    - `previous_returns`
- **Inputs / outputs:**
  - **Input:** from `Code in JavaScript`
  - **Output:** none connected (end)
- **Version notes:** typeVersion **2.1**
- **Failure/edge cases:**
  - Airtable auth/token missing or insufficient permissions.
  - Field name mismatch: table field appears to be spelled `resons` (not `reasons`). If the actual Airtable field is `reasons`, this will silently fail or error depending on Airtable/node behavior.
  - Type mismatches (numbers vs strings) if Airtable schema expects a different type.
  - Rate limits for Airtable API if high alert volume.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | n8n-nodes-base.scheduleTrigger | Hourly trigger | — | Time_window | ## How it Works: This workflow automatically monitors WooCommerce refund activity on a scheduled basis... (full note content applies) |
| Time_window | n8n-nodes-base.code | Compute current/previous 24h windows | Schedule Trigger | HTTP Orders | ## How it Works: This workflow automatically monitors WooCommerce refund activity on a scheduled basis... (full note content applies) |
| HTTP Orders | n8n-nodes-base.httpRequest | Fetch WooCommerce orders | Time_window | HTTP Refunds | ## Data Collection & Return Analysis: • HTTP Orders – Fetches WooCommerce orders and line-item details... |
| HTTP Refunds | n8n-nodes-base.httpRequest | Fetch WooCommerce refunds | HTTP Orders | Orders_Fetch | ## Data Collection & Return Analysis: • HTTP Refunds – Fetches refund records with reasons and timestamps... |
| Orders_Fetch | n8n-nodes-base.code | Join refunds to orders; emit SKU-level events | HTTP Refunds | Refund_details | ## Data Collection & Return Analysis: • Orders_Fetch – Maps refunds to parent orders and extracts SKU data... |
| Refund_details | n8n-nodes-base.code | Aggregate returns per SKU; compute change | Orders_Fetch | If | ## Data Collection & Return Analysis: • Refund_details – Calculates return counts, increases, and reasons per SKU... |
| If | n8n-nodes-base.if | Filter for surge thresholds | Refund_details | Edit Fields1 (true) | ## Surge Filter & Enrichment: • IF – Detects return surges based on thresholds (≥100% increase OR ≥25 current returns). |
| Edit Fields1 | n8n-nodes-base.set | Add alert metadata | If (true) | Send a message | ## Surge Filter & Enrichment: • Set Fields – Adds alert metadata (status, run date, cooldown key)... |
| Send a message | n8n-nodes-base.slack | Send Slack alert | Edit Fields1 | Code in JavaScript | ## Alerts & Logging: • Slack – Sends return surge alerts with SKU, counts, increase %, and reasons. |
| Code in JavaScript | n8n-nodes-base.code | Parse Slack message into structured fields | Send a message | Create a record | ## Alerts & Logging: • Code in JavaScript – Normalizes Slack alert text into structured fields. |
| Create a record | n8n-nodes-base.airtable | Log alert to Airtable | Code in JavaScript | — | ## Alerts & Logging: • Airtable – Stores alert records for audit, trend analysis, and reporting. |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation | — | — | (This node is itself the note content: “How it Works” + “Setup Steps”) |
| Sticky Note1 | n8n-nodes-base.stickyNote | Documentation | — | — | (This node is itself the note content: “Data Collection & Return Analysis”) |
| Sticky Note2 | n8n-nodes-base.stickyNote | Documentation | — | — | (This node is itself the note content: scheduled monitoring summary) |
| Sticky Note3 | n8n-nodes-base.stickyNote | Documentation | — | — | (This node is itself the note content: “Alerts & Logging”) |
| Sticky Note4 | n8n-nodes-base.stickyNote | Documentation | — | — | (This node is itself the note content: “Surge Filter & Enrichment”) |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
2. **Add node: Schedule Trigger**
   - Interval: every **1 hour**.
   - Connect to the next node.
3. **Add node: Code** named **Time_window**
   - Paste logic to compute `timeWindow.currentStart/currentEnd/prevStart/prevEnd` in UTC ISO format (24h rolling window).
   - Connect to HTTP Orders.
4. **Add node: HTTP Request** named **HTTP Orders**
   - Method: GET
   - URL: `https://{your_wocommerce_domain}/wp-json/wc/v3/orders`
   - Authentication: **HTTP Basic Auth** (via Generic Credential Type)
   - Create/select an **HTTP Basic Auth credential** using your WooCommerce REST API key/secret (or WP Basic Auth if that’s how your store is configured).
   - Connect to HTTP Refunds.
5. **Add node: HTTP Request** named **HTTP Refunds**
   - Method: GET
   - URL: `https://{your_wocommerce_domain}/wp-json/wc/v3/refunds`
   - Authentication: same **HTTP Basic Auth** credential
   - Connect to Orders_Fetch.
6. **Add node: Code** named **Orders_Fetch**
   - Implement join logic:
     - Load items from `HTTP Orders` and `HTTP Refunds` using `$items("HTTP Orders")` and `$items("HTTP Refunds")`
     - Map orders by id
     - For each refund, find its parent order and emit per-line-item events including: `sku`, `reason`, and `refund_date_utc = date_created_gmt`
   - Connect to Refund_details.
7. **Add node: Code** named **Refund_details**
   - Aggregate by SKU into `current` and `previous` windows (each 24 hours).
   - Compute `absolute_increase` and `percent_increase` (null when no baseline).
   - Output per SKU: `sku, current_returns, previous_returns, absolute_increase, percent_increase, reasons`.
   - Connect to If.
8. **Add node: If** named **If**
   - Condition group OR:
     - Number: `{{$json.percent_increase}}` **gte** `100`
     - Number: `{{$json.current_returns}}` **gte** `25`
   - Connect **true** output to Edit Fields.
9. **Add node: Set** named **Edit Fields1**
   - Set `status` = `alerted`
   - Set `run_date_utc` = `{{ new Date().toISOString() }}`
   - Set `cooldown_key` = `{{ $json.sku + "_" + new Date().toISOString().slice(0,10) }}`
   - Keep “Include Other Fields” enabled.
   - Connect to Slack.
10. **Add node: Slack** named **Send a message**
   - Resource/Operation: send message to a channel
   - Choose your channel (e.g., `#n8n-demo`)
   - Configure Slack credential (Slack API token with `chat:write`; ensure bot is in the channel)
   - Message text template should include SKU, counts, increase %, reasons list, status, run date, cooldown key.
   - Connect to parsing code node.
11. **Add node: Code** named **Code in JavaScript**
   - Parse Slack response’s `message.text` using regex to extract:
     - `sku, current_returns, previous_returns, percent_increase, status, run_date_utc, cooldown_key`
     - reasons block into `reasons_raw` and `reasons_json`
   - Connect to Airtable.
12. **Add node: Airtable** named **Create a record**
   - Operation: Create
   - Configure Airtable Personal Access Token credential
   - Select Base and Table (e.g., base “n8n Learning”, table “woocommerce”)
   - Map fields:
     - `sku` ← `{{$json.sku}}`
     - `current_returns` ← `{{$json.current_returns}}`
     - `previous_returns` ← `{{$json.previous_returns}}`
     - `percent_increase` ← `{{$json.percent_increase}}`
     - `run_date_utc` ← `{{$json.run_date_utc}}`
     - `cooldown_key` ← `{{$json.cooldown_key}}`
     - `resons` (or your actual field name) ← `{{$json.reasons_raw}}`
13. **Activate** the workflow once credentials are valid and endpoints respond.

**Optional but strongly recommended to match intent (“real-time” and reduce load):**
- Add WooCommerce query parameters to both HTTP nodes:
  - `after` / `before` using values from `Time_window`
  - Pagination (`per_page`, `page`) loop until empty
- Add a deduplication/cooldown check using `cooldown_key` (e.g., store keys in Airtable and skip if already alerted today).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “How it Works” + “Setup Steps” sticky note describing the overall flow and setup sequence. | Internal sticky note (no external link). |
| Airtable base/table referenced in node selection. | Base: https://airtable.com/app6M3NseJ4VEy2Ro ; Table: https://airtable.com/app6M3NseJ4VEy2Ro/tblT2vWZ8en2ZRdox |
| Surge logic: alert when `percent_increase >= 100` OR `current_returns >= 25`. | Implemented in the `If` node. |
| Workflow name mentions “Webhook” but the entry point is a Schedule Trigger. | Consider renaming for clarity. |