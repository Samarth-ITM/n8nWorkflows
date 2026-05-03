Notify on menu orders via ntfy and Home Assistant TTS with daily BAC tracking

https://n8nworkflows.xyz/workflows/notify-on-menu-orders-via-ntfy-and-home-assistant-tts-with-daily-bac-tracking-14487


# Notify on menu orders via ntfy and Home Assistant TTS with daily BAC tracking

Now I have all the information I need. Let me write the comprehensive reference document. 1. Workflow Overview

**Notify on menu orders via ntfy and Home Assistant TTS with daily BAC tracking** is a single-entry-point automation that receives food/drink orders through a webhook, persists them to a DataTable, computes a cumulative daily Blood Alcohol Content (BAC) estimate for the ordering person using the Widmark formula, and delivers two types of real-time notifications: a push notification via **ntfy** and a spoken announcement via **Home Assistant TTS** (through a script service call).

**Target use cases:**
- A restaurant, bar, or private event system that collects menu orders from a front-end client.
- Monitoring cumulative alcohol intake per person per day, with threshold-based visual indicators.
- Broadcasting orders audibly in a smart-home environment (e.g., kitchen speaker) while simultaneously pushing them to mobile/desktop devices.

**Logical blocks:**

| Block | Name | Purpose |
|-------|------|---------|
| 1 | **Receive and Prepare Orders** | Accepts the incoming POST webhook, normalises the customer name, aggregates alcohol grams from the order items, and formats a human-readable order string. |
| 2 | **Log Order and Read Data** | Persists the current order into a DataTable and then queries all of today's orders for that person to feed the BAC calculator. |
| 3 | **Calculate BAC and Notify** | Calculates the cumulative BAC from today's accumulated orders, builds a rich notification message (with emoji indicators and order history), pushes it via ntfy, and triggers a Home Assistant script for TTS announcement. |

---

### 2. Block-by-Block Analysis

---

#### Block 1 — Receive and Prepare Orders

**Overview:** This block serves as the entry point of the workflow. A webhook listens for POST requests containing a JSON body with the customer's name, order time, and an array of items (each with `name`, `quantity`, optional `detail`, and optional `alcohol_grams`). A Code node then normalises the name, sums the alcohol content, and formats a readable order string for downstream consumption.

**Nodes Involved:**
1. When Order Received
2. Calculate Alcohol and Format Order

---

**Node: When Order Received**

| Property | Detail |
|----------|--------|
| **Type** | Webhook (n8n-nodes-base.webhook, v2.1) |
| **Technical Role** | HTTP entry point; exposes a POST endpoint at `/menu` and passes the request body to the next node. |
| **Configuration** | HTTP Method: `POST`; Path: `menu`; No authentication on the webhook itself; default response mode (Respond to Webhook). |
| **Key Expressions / Variables** | None in the node itself; downstream nodes read `$json.body`. |
| **Input Connections** | None (entry node). |
| **Output Connections** | → Calculate Alcohol and Format Order |
| **Version-specific Requirements** | Webhook v2 uses `body` nested under the output JSON (accessed as `$json.body`). |
| **Edge Cases / Potential Failures** | — Malformed or missing JSON body → downstream Code node falls back to defaults (`'Anonymous'`, current time, empty items). <br>— Extremely large payloads may hit n8n body size limits. <br>— No auth means any caller can post orders; consider adding Basic Auth or API-key validation if exposed publicly. |

---

**Node: Calculate Alcohol and Format Order**

| Property | Detail |
|----------|--------|
| **Type** | Code (n8n-nodes-base.code, v2) |
| **Technical Role** | Transforms and enriches the raw webhook payload into a structured object used by all downstream nodes. |
| **Configuration** | Language: JavaScript (Run Once for All Items mode by default for single-item output). |
| **Key Expressions / Variables** | Reads `$input.first().json.body`. Outputs: `name` (capitalised), `orderTime`, `items` (raw array), `orderStr` (formatted string, e.g. "2x Beer, 1x Wine red"), `alcoholGrams` (sum rounded to 1 decimal), `today` (ISO date YYYY-MM-DD). |
| **Input Connections** | ← When Order Received |
| **Output Connections** | → Log Order to Database |
| **Version-specific Requirements** | Code node v2 requires ES-module-style return (`return [{ json: { … } }]`). |
| **Edge Cases / Potential Failures** | — `body.name` is null/undefined → defaults to `'Anonymous'`. <br>— `body.items` is empty or missing → `alcoholGrams` = 0, `orderStr` = `''` (empty string). <br>— Items with missing `alcohol_grams` → treated as 0; missing `quantity` → defaults to 1. <br>— Very long item lists could produce an `orderStr` exceeding notification character limits. |

**Key logic excerpt (summary):**
- Name normalisation: first character uppercase, rest lowercase, trimmed.
- Alcohol sum: `items.reduce((s, i) => s + (i.alcohol_grams || 0) * (i.quantity || 1), 0)`.
- Order string: each item rendered as `<quantity>x <name> <detail>` (detail appended only if present), joined by `, `.

---

#### Block 2 — Log Order and Read Data

**Overview:** This block first inserts the current order as a new row into a DataTable, then immediately reads back **all** rows for the same person on the same date. This ensures the subsequent BAC calculation works on the full daily picture (including the order just inserted).

**Nodes Involved:**
1. Log Order to Database
2. Read Today's Orders

---

**Node: Log Order to Database**

| Property | Detail |
|----------|--------|
| **Type** | DataTable (n8n-nodes-base.dataTable, v1.1) |
| **Technical Role** | Inserts one row per order into the persistent DataTable. |
| **Configuration** | Operation: Insert (default). Columns mapped explicitly (`defineBelow`): <br>• `item` ← `$json.orderStr` (string) <br>• `person` ← `$json.name` (string) <br>• `alcohol_grams` ← `$json.alcoholGrams` (number) <br>• `date` ← `$json.today` (string, ISO date) <br>• `order_time` ← `$json.orderTime` (string). |
| **Key Expressions / Variables** | All values reference the output of "Calculate Alcohol and Format Order" via `$json`. |
| **Input Connections** | ← Calculate Alcohol and Format Order |
| **Output Connections** | → Read Today's Orders |
| **Version-specific Requirements** | A DataTable must already exist with columns: `item` (string), `person` (string), `alcohol_grams` (number), `date` (string), `order_time` (string). The node's `dataTableId` must be set to that table's ID. |
| **Edge Cases / Potential Failures** | — DataTable ID not configured or table deleted → node fails with "Table not found". <br>— Schema mismatch (e.g. column type wrong) → insertion error. <br>— Concurrent inserts from rapid webhook calls could lead to a race condition where the subsequent read may not yet reflect the latest row (depends on storage backend consistency). |

---

**Node: Read Today's Orders**

| Property | Detail |
|----------|--------|
| **Type** | DataTable (n8n-nodes-base.dataTable, v1.1) |
| **Technical Role** | Retrieves all rows matching the person and today's date, returning them as individual items for the next Code node. |
| **Configuration** | Operation: `get`; `returnAll`: `true`; Filter match type: `allConditions`; Two filter conditions: <br>1. `person` = `{{ $('Calculate Alcohol and Format Order').item.json.name }}` <br>2. `date` = `{{ $('Calculate Alcohol and Format Order').item.json.today }}`. |
| **Key Expressions / Variables** | References "Calculate Alcohol and Format Order" node for the person name and date. |
| **Input Connections** | ← Log Order to Database |
| **Output Connections** | → Calculate Cumulative BAC |
| **Version-specific Requirements** | Same DataTable must be referenced by `dataTableId`. `returnAll: true` ensures all matching rows are output; if `false`, only the first page (typically 100 rows) is returned, which could silently truncate data. |
| **Edge Cases / Potential Failures** | — DataTable ID not configured → same as above. <br>— No rows found for the person/date → the Code node receives an empty input array, total alcohol = 0, BAC = 0. <br>— If the insert in the previous node failed silently or was delayed, the row just inserted might not appear, leading to under-counting. <br>— Very high order volumes per person per day could produce many rows; `returnAll` loads all into memory. |

---

#### Block 3 — Calculate BAC and Notify

**Overview:** This block computes the cumulative BAC using the Widmark formula on today's accumulated alcohol grams, builds a richly formatted notification message with emoji indicators and order history, pushes it to an ntfy topic via HTTP POST, and finally calls a Home Assistant script to announce the order over TTS.

**Nodes Involved:**
1. Calculate Cumulative BAC
2. Send Ntfy Notification
3. Announce Order with Home Assistant

---

**Node: Calculate Cumulative BAC**

| Property | Detail |
|----------|--------|
| **Type** | Code (n8n-nodes-base.code, v2) |
| **Technical Role** | Aggregates all today's order rows, calculates BAC, and assembles the notification message string. |
| **Configuration** | Language: JavaScript; processes all input items. |
| **Key Expressions / Variables** | — `$input.all()` → all rows from "Read Today's Orders". <br>— References "Calculate Alcohol and Format Order" for `name`, `orderTime`, `orderStr`, `alcoholGrams`, `today`. <br>— **Widmark formula constants:** Body weight = **70 kg**, Widmark factor = **0.68** (male average). <br>— BAC = `totalAlcohol / (weight × factor)`. <br>— Thresholds: BAC = 0 → ⬜; BAC < 0.2 → 🟢; BAC < 0.5 → 🟡; BAC ≥ 0.5 → 🔴. |
| **Output Fields** | `name`, `orderTime`, `orderStr`, `message` (full notification text), `bac` (string, 2 decimal places), `bacLevel` (emoji), `totalAlcohol`, `alcoholGrams` (current order), `orderCount`, `today`. |
| **Input Connections** | ← Read Today's Orders |
| **Output Connections** | → Send Ntfy Notification |
| **Version-specific Requirements** | Code v2. |
| **Edge Cases / Potential Failures** | — Zero rows → `totalAlcohol = 0`, BAC = 0, `bacLevel` = ⬜. <br>— Very high alcohol intake could produce BAC > 1.0; no upper-bound capping is applied. <br>— The formula uses fixed weight/factor; if the actual person differs significantly, the BAC estimate will be inaccurate. <br>— Unicode emoji codepoints rely on the runtime environment supporting them (modern Node.js does). <br>— `rows.slice(0, -1)` for "previous orders" assumes the last row is the one just inserted; this may not hold if rows are not returned in insertion order. |

**Message format (example):**
```
Alice: 2x Beer, 1x Wine red
🕐 14:30:00
📋 Already ordered today (2 orders):
  - 1x Beer  🕒 12:00:00
  - 1x Beer  🕒 13:00:00
🍷 BAC ~0.35 g/L  🟡
```
Or, if no alcohol:
```
Alice: 1x Water
🕐 14:30:00
🥤 no alcohol
```

---

**Node: Send Ntfy Notification**

| Property | Detail |
|----------|--------|
| **Type** | HTTP Request (n8n-nodes-base.httpRequest, v4.4) |
| **Technical Role** | Pushes the formatted message as a plain-text notification to an ntfy topic. |
| **Configuration** | Method: `POST`; URL: `http://YOUR_NTFY_SERVER/YOUR_TOPIC` (must be replaced with actual ntfy server and topic); Body: `={{ $json.message }}`; Content-Type: `text/plain` (raw); Authentication: `genericCredentialType` → `httpHeaderAuth` (an HTTP Header Auth credential storing the ntfy token/header). |
| **Key Expressions / Variables** | `$json.message` from "Calculate Cumulative BAC". |
| **Input Connections** | ← Calculate Cumulative BAC |
| **Output Connections** | → Announce Order with Home Assistant |
| **Version-specific Requirements** | HTTP Request v4.4; requires a pre-configured HTTP Header Auth credential. |
| **Edge Cases / Potential Failures** | — URL placeholder not replaced → request will fail (DNS/connection error). <br>— Credential not configured → 401/403 authentication error. <br>— ntfy server unreachable → timeout or connection refused. <br>— Message too long for ntfy (theoretical, usually 4KB+ limit). <br>— ntfy returns non-2xx (e.g. rate limiting 429) → node fails, stopping the workflow before TTS announcement. |

---

**Node: Announce Order with Home Assistant**

| Property | Detail |
|----------|--------|
| **Type** | Home Assistant (n8n-nodes-base.homeAssistant, v1) |
| **Technical Role** | Calls a Home Assistant script service (`script.announce_order`) with a `message` attribute, triggering a TTS announcement. |
| **Configuration** | Domain: `script`; Service: `announce_order`; Operation: `call`; Resource: `service`; Service attributes: one attribute named `message` with value `={{ $json.name }} HAS ORDERED {{ $('When Order Received').item.json.body.items[0].name }}`. |
| **Key Expressions / Variables** | — `$json.name` from "Calculate Cumulative BAC" output. <br>— `$('When Order Received').item.json.body.items[0].name` → the **first item name** from the original webhook payload. |
| **Input Connections** | ← Send Ntfy Notification |
| **Output Connections** | None (terminal node). |
| **Version-specific Requirements** | Home Assistant node v1; requires configured Home Assistant credentials (long-lived access token or OAuth). The script `announce_order` must exist in the Home Assistant instance. |
| **Edge Cases / Potential Failures** | — Credential not configured or token expired → 401 error. <br>— Home Assistant instance unreachable → connection timeout. <br>— Script `announce_order` does not exist → 404/service not found. <br>— If the order has zero items, `items[0].name` is `undefined`, resulting in "undefined" being announced. <br>— Only the **first** item name is announced; multi-item orders are not fully spoken. <br>— This node is sequential after ntfy; if the ntfy node fails, this node never executes. |

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|-----------|-----------|----------------|---------------|-----------------|-------------|
| When Order Received | Webhook (v2.1) | HTTP entry point accepting POST orders | — | Calculate Alcohol and Format Order | ## Receive and prepare orders — Receives the menu order via a webhook and prepares the data by normalizing and calculating necessary order information. |
| Calculate Alcohol and Format Order | Code (v2) | Normalise name, sum alcohol grams, format order string | When Order Received | Log Order to Database | ## Receive and prepare orders — Receives the menu order via a webhook and prepares the data by normalizing and calculating necessary order information. |
| Log Order to Database | DataTable (v1.1) | Insert current order row into persistent table | Calculate Alcohol and Format Order | Read Today's Orders | ## Log order and read data — Logs the prepared order into the database and reads the accumulated orders for the day. |
| Read Today's Orders | DataTable (v1.1) | Retrieve all of today's orders for the person | Log Order to Database | Calculate Cumulative BAC | ## Log order and read data — Logs the prepared order into the database and reads the accumulated orders for the day. |
| Calculate Cumulative BAC | Code (v2) | Aggregate daily alcohol, compute BAC via Widmark, build notification message | Read Today's Orders | Send Ntfy Notification | ## Calculate BAC and notify — Calculates the cumulative BAC from today's orders and sends notifications followed by TTS announcements. |
| Send Ntfy Notification | HTTP Request (v4.4) | Push notification to ntfy topic | Calculate Cumulative BAC | Announce Order with Home Assistant | ## Calculate BAC and notify — Calculates the cumulative BAC from today's orders and sends notifications followed by TTS announcements. |
| Announce Order with Home Assistant | Home Assistant (v1) | Call HA script to announce order via TTS | Send Ntfy Notification | — | ## Calculate BAC and notify — Calculates the cumulative BAC from today's orders and sends notifications followed by TTS announcements. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it "Notify on menu orders via ntfy and Home Assistant TTS with daily BAC tracking".

2. **Create a DataTable** (via n8n's DataTable UI or an external store) with the following columns:
   - `item` — type: string
   - `person` — type: string
   - `alcohol_grams` — type: number
   - `date` — type: string (ISO date YYYY-MM-DD)
   - `order_time` — type: string
   
   Note the DataTable ID after creation.

3. **Add node: "When Order Received"**
   - Type: Webhook
   - HTTP Method: `POST`
   - Path: `menu`
   - Options: defaults (no authentication)
   - This creates the webhook URL; note it for your client application.

4. **Add node: "Calculate Alcohol and Format Order"**
   - Type: Code
   - Language: JavaScript
   - Paste the following logic (summarised):
     - Read `$input.first().json.body`.
     - Normalise `body.name` (trim, capitalise first letter, lowercase rest; default `'Anonymous'`).
     - Set `orderTime` from `body.time` or current locale string.
     - Extract `body.items` array (default empty).
     - Compute `alcoholGrams` = `items.reduce((sum, i) => sum + (i.alcohol_grams || 0) * (i.quantity || 1), 0)` rounded to 1 decimal.
     - Build `orderStr` = items mapped to `"<quantity>x <name> <detail>"` joined by `, `.
     - Set `today` = `new Date().toISOString().slice(0, 10)`.
     - Return `{ name, orderTime, items, orderStr, alcoholGrams, today }`.
   - Connect: When Order Received → Calculate Alcohol and Format Order.

5. **Add node: "Log Order to Database"**
   - Type: DataTable
   - Operation: Insert
   - DataTable ID: select the table created in step 2.
   - Mapping mode: `defineBelow`
   - Column mappings:
     - `item` = `={{ $json.orderStr }}`
     - `person` = `={{ $json.name }}`
     - `alcohol_grams` = `={{ $json.alcoholGrams }}`
     - `date` = `={{ $json.today }}`
     - `order_time` = `={{ $json.orderTime }}`
   - Connect: Calculate Alcohol and Format Order → Log Order to Database.

6. **Add node: "Read Today's Orders"**
   - Type: DataTable
   - Operation: `get`
   - Return All: `true`
   - Match Type: `allConditions`
   - Filter conditions:
     - Column `person` = `={{ $('Calculate Alcohol and Format Order').item.json.name }}`
     - Column `date` = `={{ $('Calculate Alcohol and Format Order').item.json.today }}`
   - DataTable ID: same table as step 2.
   - Connect: Log Order to Database → Read Today's Orders.

7. **Add node: "Calculate Cumulative BAC"**
   - Type: Code
   - Language: JavaScript
   - Paste the following logic (summarised):
     - `rows = $input.all().map(i => i.json)`.
     - Reference "Calculate Alcohol and Format Order" for `name`, `orderTime`, `orderStr`.
     - `totalAlcohol` = sum of `r.alcohol_grams` across all rows, rounded to 1 decimal.
     - `orderCount` = `rows.length`.
     - `bac = totalAlcohol / (70 * 0.68)` (weight 70 kg, factor 0.68).
     - `bacStr = bac.toFixed(2)`.
     - Determine `bacLevel` emoji based on thresholds (0 → ⬜, <0.2 → 🟢, <0.5 → 🟡, ≥0.5 → 🔴).
     - Build `historyStr` from `rows.slice(0, -1)` (previous orders).
     - Build `bacRow` showing BAC or "no alcohol".
     - Assemble `message` combining name, orderStr, time, history, and BAC.
     - Return all computed fields.
   - Connect: Read Today's Orders → Calculate Cumulative BAC.

8. **Add node: "Send Ntfy Notification"**
   - Type: HTTP Request
   - Method: `POST`
   - URL: Replace `http://YOUR_NTFY_SERVER/YOUR_TOPIC` with your actual ntfy server address and topic (e.g. `https://ntfy.sh/my-bar-orders`).
   - Body: `={{ $json.message }}`
   - Content Type: `raw` → sub-type: `text/plain`
   - Authentication: `genericCredentialType` → `httpHeaderAuth`
   - Create an **HTTP Header Auth** credential:
     - Header Name: `Authorization` (or whatever your ntfy instance expects, commonly `Authorization: Bearer <token>`)
     - Header Value: your ntfy access token or password.
   - Connect: Calculate Cumulative BAC → Send Ntfy Notification.

9. **Add node: "Announce Order with Home Assistant"**
   - Type: Home Assistant
   - Domain: `script`
   - Service: `announce_order`
   - Operation: `call`
   - Resource: `service`
   - Service attributes:
     - Attribute name: `message`
     - Attribute value: `={{ $json.name }} HAS ORDERED {{ $('When Order Received').item.json.body.items[0].name }}`
   - Create a **Home Assistant** credential:
     - Enter your Home Assistant base URL.
     - Provide a long-lived access token (generated in HA profile settings).
   - Ensure the script `announce_order` exists in Home Assistant (must accept a `message` parameter and typically call `tts.google_translate_say` or similar).
   - Connect: Send Ntfy Notification → Announce Order with Home Assistant.

10. **Add Sticky Notes (optional, for documentation):**
    - Place a main documentation note on the canvas summarising the workflow, setup checklist, and link: https://paoloronco.notion.site/Documentation-Menu-Order-Push-Notifications-Home-Assistant-TTS-BAC-32ff0ba27c328075a886d89ebfbf5ce5
    - Place a "Receive and prepare orders" note covering nodes 3–4.
    - Place a "Log order and read data" note covering nodes 5–6.
    - Place a "Calculate BAC and notify" note covering nodes 7–9.

11. **Activate the workflow** by toggling it to Active. Copy the webhook URL and configure your client application to POST orders in the following JSON format:
    ```json
    {
      "name": "alice",
      "time": "14:30",
      "items": [
        { "name": "Beer", "quantity": 2, "alcohol_grams": 14, "detail": "blonde" },
        { "name": "Wine", "quantity": 1, "alcohol_grams": 12, "detail": "red" }
      ]
    }
    ```

12. **Customisation points:**
    - **Weight and Widmark factor**: In "Calculate Cumulative BAC", change `70` (kg) and `0.68` (factor) to match the target person. For females, a factor of `0.55` is commonly used.
    - **BAC thresholds**: Adjust the `< 0.2` and `< 0.5` comparisons to your desired levels.
    - **Notification message format**: Modify the string-building logic in "Calculate Cumulative BAC" to change emojis, add/remove sections, or translate text.
    - **Home Assistant script name**: Change `announce_order` to any other script name in the HA node.
    - **ntfy URL and topic**: Update in the HTTP Request node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Full documentation with detailed setup instructions and explanations | https://paoloronco.notion.site/Documentation-Menu-Order-Push-Notifications-Home-Assistant-TTS-BAC-32ff0ba27c328075a886d89ebfbf5ce5 |
| Webhook expects a POST with JSON body containing `name` (string), `time` (string, optional), and `items` (array of objects with `name`, `quantity`, optional `alcohol_grams`, optional `detail`). | Input contract for the webhook endpoint |
| The Widmark BAC formula used: BAC (g/L) = total_alcohol_grams / (body_weight_kg × widmark_factor). Default: 70 kg × 0.68. | Scientific reference for the BAC calculation |
| The Home Assistant TTS announcement only speaks the **first item** of the order. If multi-item verbalisation is needed, the expression must be updated. | Known limitation |
| The DataTable read assumes rows are returned in insertion order and treats the last row as the current order. If ordering is not guaranteed, the history string may include the current order. | Known limitation |
| If the ntfy HTTP request fails, the Home Assistant announcement will not execute because they are sequentially chained. Consider adding an error-branch or setting the ntfy node to "Continue on Fail" to ensure TTS always fires. | Reliability recommendation |
| The webhook has no authentication. For production use, consider enabling Basic Auth or API-key validation. | Security recommendation |
| The BAC estimate is an approximation for educational/fun purposes and should not be used for legal or medical decisions. | Disclaimer |