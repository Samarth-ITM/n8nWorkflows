Sync inventory updates between systems via webhook and Google Sheets logging

https://n8nworkflows.xyz/workflows/sync-inventory-updates-between-systems-via-webhook-and-google-sheets-logging-12871


# Sync inventory updates between systems via webhook and Google Sheets logging

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Sync inventory updates between systems via webhook and Google Sheets logging  
**Workflow name (internal):** Retail - Transfer Inventory Updates Across Systems  
**Purpose:** Receive inventory update events from a source system via a webhook, prevent sync loops, forward the normalized update to a secondary system via an API call, and log the synchronization outcome/details to Google Sheets for auditing.

### 1.1 Trigger & Input Reception
- Entry point: a **POST Webhook** that receives inventory update payloads.

### 1.2 Data Normalization
- Converts/ensures expected field types and exposes a consistent `body.*` structure for downstream nodes.

### 1.3 Loop Prevention + Sync to Secondary System
- Checks the `source` field to avoid circular updates (e.g., secondary system echoing changes back).
- Builds a clean payload and sends it to the secondary system via HTTP.

### 1.4 Logging & Auditing
- Appends or updates a Google Sheets log with key fields (ID/SKU/qty/source/status/etc.).

---

## 2. Block-by-Block Analysis

### Block 1 — Trigger & Data Preparation
**Overview:** Accepts inbound inventory update events and standardizes incoming fields into a predictable schema (`body.id`, `body.sku`, etc.) with explicit types for later processing.  
**Nodes involved:** `Inventory Webhook`, `Normalize Inventory Data`  
**Sticky note context:** “Trigger & Data Preparation”

#### Node: Inventory Webhook
- **Type / role:** `Webhook` (n8n-nodes-base.webhook) — workflow entry point.
- **Configuration (interpreted):**
  - **HTTP Method:** POST
  - **Path:** `inventory-update` (endpoint becomes something like `https://<n8n-host>/webhook/inventory-update` for test/production depending on n8n mode)
  - No special options configured.
- **Key data expectations:**
  - Expects a request body containing fields used later: `id`, `sku`, `stock_quantity`, `date_modified`, `manage_stock`, `type`, `status`, `source`.
  - In n8n, the incoming body is typically available at `$json.body` for Webhook nodes.
- **Connections:**
  - **Output:** to `Normalize Inventory Data`
- **Version-specific notes:** typeVersion **2.1**.
- **Edge cases / failures:**
  - Missing/invalid JSON body (client sends wrong Content-Type or malformed JSON).
  - Unexpected payload shape (no `body` object or missing keys) leading to downstream expression issues.
  - If used in production, ensure the correct webhook URL (test vs production) is configured in the source system.

#### Node: Normalize Inventory Data
- **Type / role:** `Set` (n8n-nodes-base.set) — normalizes and type-casts inbound fields.
- **Configuration (interpreted):**
  - Creates/overwrites the following fields (as *new assignments*) with explicit types:
    - `body.id` (number) = `{{ $json.body.id }}`
    - `body.sku` (string) = `{{ $json.body.sku }}`
    - `body.stock_quantity` (number) = `{{ $json.body.stock_quantity }}`
    - `body.date_modified` (string) = `{{ $json.body.date_modified }}`
    - `body.manage_stock` (boolean) = `{{ $json.body.manage_stock }}`
    - `body.type` (string) = `{{ $json.body.type }}`
    - `body.status` (string) = `{{ $json.body.status }}`
    - `body.source` (string) = `{{ $json.body.source }}`
- **Key expressions / variables:**
  - Direct reads from `$json.body.<field>`
- **Connections:**
  - **Input:** from `Inventory Webhook`
  - **Output:** to `Check Sync Origin`
- **Version-specific notes:** typeVersion **3.4**.
- **Edge cases / failures:**
  - If `$json.body` is absent, every assignment can evaluate to `undefined`, and strict type conversion may yield unexpected values.
  - If `stock_quantity` is a non-numeric string, “number” typing can produce `NaN` downstream (especially when later wrapped by `Number(...)`).
  - If `manage_stock` arrives as `"true"` (string) instead of boolean, behavior depends on n8n’s conversion rules; validate upstream payloads if you rely on this.

---

### Block 2 — Inventory Sync (Loop Prevention + API Push)
**Overview:** Prevents circular synchronization by filtering events that originated from the secondary system, then prepares a clean payload and sends it to the secondary API.  
**Nodes involved:** `Check Sync Origin`, `Prepare Inventory Payload`, `Send Inventory To Secondary API`  
**Sticky note context:** “Inventory Sync”

#### Node: Check Sync Origin
- **Type / role:** `If` (n8n-nodes-base.if) — conditional gate to prevent sync loops.
- **Configuration (interpreted):**
  - Condition: `{{ $json.body.source }}` **notEquals** `"secondary-system"`
  - If true, workflow continues to payload prep and API send.
- **Important detail (possible misconfiguration):**
  - The **right value** is stored as `"\"secondary-system\""` in JSON, which evaluates as the literal string `"secondary-system"` including quotes if not interpreted correctly by the UI. In practice, in n8n UI it typically represents the string `secondary-system`, but you should verify in the node UI that the comparison is against `secondary-system` (without extra quotes).
- **Connections:**
  - **Input:** from `Normalize Inventory Data`
  - **True output:** to `Prepare Inventory Payload`
  - **False output:** not connected (events from `secondary-system` will be dropped silently).
- **Version-specific notes:** typeVersion **2.2** (conditions “version 2” format).
- **Edge cases / failures:**
  - If `body.source` is missing, condition becomes `undefined != secondary-system` which is generally true → may unintentionally sync.
  - Case-sensitivity is enabled; `Secondary-System` won’t match. Standardize upstream casing or adjust condition.
  - No explicit handling/logging for the “false” branch (skipped updates are not logged).

#### Node: Prepare Inventory Payload
- **Type / role:** `Code` (n8n-nodes-base.code) — transforms normalized data into an API-ready structure.
- **Configuration (interpreted):**
  - JavaScript builds a single output item:
    - `id`: from `Normalize Inventory Data` `body.id`
    - `sku`: from `body.sku`
    - `qty`: `Number(body.stock_quantity)`
    - `dateModified`: from `body.date_modified`
    - `type`: from `body.type`
    - `origin`: from `body.source`
    - `status`: from `body.status`
  - Uses node selector: `$('Normalize Inventory Data').first().json...`
- **Key expressions / variables:**
  - `$('Normalize Inventory Data').first()` (explicitly takes the first item from that node)
- **Connections:**
  - **Input:** from `Check Sync Origin` (true branch)
  - **Output:** to `Send Inventory To Secondary API`
- **Version-specific notes:** typeVersion **2**.
- **Edge cases / failures:**
  - If multiple webhook items were ever batched (uncommon for Webhook), `.first()` would ignore all but the first.
  - `Number(...)` can produce `NaN` if `stock_quantity` is empty/non-numeric; secondary API may reject.
  - If `Normalize Inventory Data` is renamed, this node breaks due to hard-coded node name reference.

#### Node: Send Inventory To Secondary API
- **Type / role:** `HTTP Request` (n8n-nodes-base.httpRequest) — pushes the inventory update to the secondary system.
- **Configuration (interpreted):**
  - **Method:** POST
  - **Send Body:** enabled
  - **Body parameters:** `id`, `sku`, `qty`, `dateModified`, `type`, `origin`, `status` from the current item (`$json.*`)
  - **Response handling:** “Full Response” enabled (captures status code/headers/body), although the response is not explicitly used downstream.
  - **Critical missing configuration:** **No URL/endpoint is defined in the provided JSON.** You must set the target API URL in the node (e.g., `https://secondary.example.com/api/inventory`).
- **Connections:**
  - **Input:** from `Prepare Inventory Payload`
  - **Output:** to `Log Inventory Sync To Google Sheet`
- **Version-specific notes:** typeVersion **4.3**.
- **Edge cases / failures:**
  - Missing URL will cause the node to fail at runtime.
  - Authentication not configured: most APIs require headers (Bearer token), API key, or OAuth2.
  - Timeouts, non-2xx responses, schema validation errors.
  - If the API expects JSON body but the node is configured to send form-encoded parameters, you may need to switch to JSON body mode (depends on n8n UI settings for “Body Content Type” / “JSON/RAW”).

---

### Block 3 — Logging
**Overview:** Writes a record of the sync to Google Sheets using an “append or update” strategy keyed on `Id`.  
**Nodes involved:** `Log Inventory Sync To Google Sheet`  
**Sticky note context:** “Logging”

#### Node: Log Inventory Sync To Google Sheet
- **Type / role:** `Google Sheets` (n8n-nodes-base.googleSheets) — audit log storage.
- **Configuration (interpreted):**
  - **Operation:** Append or Update
  - **Document:** `Inventory_Sync_Log` (Spreadsheet ID: `1tZp-X-3ZLwkj53h7BKhM5DbZKf3zVOtg-kcOIjmrA4w`)
  - **Sheet tab:** `Sheet1` (gid=0)
  - **Matching column:** `Id` (used to update existing rows)
  - **Columns mapped (Define Below):**
    - `Id` = `{{ $('Prepare Inventory Payload').item.json.id }}`
    - `SKU` = `{{ $('Prepare Inventory Payload').item.json.sku }}`
    - `Type` = `{{ $('Prepare Inventory Payload').item.json.type }}`
    - `Quantity` = `{{ $('Prepare Inventory Payload').item.json.qty }}`
    - `Sync Status` = `{{ $('Prepare Inventory Payload').item.json.status }}`
    - `Date Modified` = `{{ $('Prepare Inventory Payload').item.json.status }}`
    - `Source System` = `{{ $('Prepare Inventory Payload').item.json.origin }}`
  - **Credential:** Google Sheets OAuth2 (`Google Sheets account 36`)
- **Connections:**
  - **Input:** from `Send Inventory To Secondary API`
  - **Output:** none
- **Version-specific notes:** typeVersion **4.7**.
- **Edge cases / failures:**
  - **Bug in mapping:** “Date Modified” is mapped to `status` instead of `dateModified`. This will log incorrect data; should likely be `{{ $('Prepare Inventory Payload').item.json.dateModified }}`.
  - Append/Update depends on the `Id` column existing and matching exactly; if blank/missing, updates may behave like appends or fail depending on node behavior.
  - OAuth token expiration/permission issues, missing access to the spreadsheet, rate limits.
  - If the sheet headers differ (e.g., `SKU` vs `Sku`), mapping can fail or create unexpected columns depending on mode.

---

### Block 4 — Documentation / Notes (Sticky Notes)
**Overview:** Non-executing nodes that document intent, setup, and block roles.  
**Nodes involved:** `Inventory Sync Overview`, `Trigger & Data Preparation`, `Inventory Sync`, `Logging`

#### Node: Inventory Sync Overview (Sticky Note)
- **Type / role:** `Sticky Note` — describes purpose, flow, and setup steps.
- **Key content (preserved):**
  - Describes listening via webhook, normalizing fields, preventing loops, sending to secondary system, logging to Google Sheets.
  - Setup steps include: configure webhook URL, ensure payload fields exist, set HTTP Request endpoint, prep Google Sheet headers, activate workflow.

#### Node: Trigger & Data Preparation (Sticky Note)
- Documents the trigger and normalization intent.

#### Node: Inventory Sync (Sticky Note)
- Documents loop prevention and API sync intent.

#### Node: Logging (Sticky Note)
- Documents audit logging intent.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Inventory Webhook | Webhook | Receives inventory updates via POST endpoint | — | Normalize Inventory Data | ## Trigger & Data Preparation<br><br>Detects inventory update events from the source system and prepares a standardized data structure by extracting essential fields such as SKU, quantity, source system and modification timestamp for downstream processing. |
| Normalize Inventory Data | Set | Normalizes/typed mapping of incoming payload fields | Inventory Webhook | Check Sync Origin | ## Trigger & Data Preparation<br><br>Detects inventory update events from the source system and prepares a standardized data structure by extracting essential fields such as SKU, quantity, source system and modification timestamp for downstream processing. |
| Check Sync Origin | If | Prevents circular sync by filtering by source system | Normalize Inventory Data | Prepare Inventory Payload (true) | ## Inventory Sync<br><br>Validates the origin of the inventory update to prevent circular sync loops, prepares a clean and API-ready payload and sends the inventory update to the secondary system for synchronization. |
| Prepare Inventory Payload | Code | Builds clean payload (id/sku/qty/date/type/origin/status) | Check Sync Origin (true) | Send Inventory To Secondary API | ## Inventory Sync<br><br>Validates the origin of the inventory update to prevent circular sync loops, prepares a clean and API-ready payload and sends the inventory update to the secondary system for synchronization. |
| Send Inventory To Secondary API | HTTP Request | POST inventory update to secondary system API | Prepare Inventory Payload | Log Inventory Sync To Google Sheet | ## Inventory Sync<br><br>Validates the origin of the inventory update to prevent circular sync loops, prepares a clean and API-ready payload and sends the inventory update to the secondary system for synchronization. |
| Log Inventory Sync To Google Sheet | Google Sheets | Append/update audit log row keyed by Id | Send Inventory To Secondary API | — | ## Logging<br><br>Records inventory sync details such as SKU, quantity, source system, status into Google Sheets for auditing, tracking and operational visibility. |
| Inventory Sync Overview | Sticky Note | Documentation (purpose/setup) | — | — | ## Retail – Transfer Inventory Updates Across Systems<br><br>### Purpose<br>Automatically synchronize inventory quantity updates between systems such as online stores, POS, ERP or warehouse platforms. This workflow ensures that stock changes remain consistent across systems and are logged for tracking and auditing.<br><br>### How it works<br>The workflow listens for inventory update events via a webhook. Incoming data is normalized to extract key fields such as product ID, SKU, quantity, source system and modification timestamp. A validation step checks the origin of the update to prevent circular sync loops. Valid updates are transformed into a clean API-ready payload and sent to a secondary system. After a successful API call, the update is logged into Google Sheets for visibility and reference.<br><br>### Setup steps<br>1. Configure the Webhook URL in your source system to send inventory updates.<br>2. Ensure SKU, stock quantity, source and date modified fields are included in the payload.<br>3. Update the HTTP Request node with your secondary system API endpoint.<br>4. Connect Google Sheets and prepare the sheet with required headers.<br>5. Activate the workflow to start processing inventory updates. |
| Trigger & Data Preparation | Sticky Note | Documentation (block label) | — | — |  |
| Inventory Sync | Sticky Note | Documentation (block label) | — | — |  |
| Logging | Sticky Note | Documentation (block label) | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **Retail - Transfer Inventory Updates Across Systems** (or your preferred name).
   - (Optional) Set workflow setting **Execution Order** to `v1` to match the provided workflow.

2. **Add Webhook node**
   - Node type: **Webhook**
   - HTTP Method: **POST**
   - Path: **inventory-update**
   - Save the node and copy the **Production** webhook URL (and/or Test URL).
   - Configure your source inventory system to POST updates to this URL with JSON including:
     - `id`, `sku`, `stock_quantity`, `date_modified`, `manage_stock`, `type`, `status`, `source`

3. **Add “Normalize Inventory Data” (Set node)**
   - Node type: **Set**
   - Add fields (use the “Add Assignment” UI and set types accordingly):
     - `body.id` (Number) = `{{$json.body.id}}`
     - `body.sku` (String) = `{{$json.body.sku}}`
     - `body.stock_quantity` (Number) = `{{$json.body.stock_quantity}}`
     - `body.date_modified` (String) = `{{$json.body.date_modified}}`
     - `body.manage_stock` (Boolean) = `{{$json.body.manage_stock}}`
     - `body.type` (String) = `{{$json.body.type}}`
     - `body.status` (String) = `{{$json.body.status}}`
     - `body.source` (String) = `{{$json.body.source}}`
   - Connect: **Inventory Webhook → Normalize Inventory Data**

4. **Add “Check Sync Origin” (If node)**
   - Node type: **If**
   - Condition (String):
     - Left value: `{{$json.body.source}}`
     - Operation: **Not equals**
     - Right value: `secondary-system`
   - Connect: **Normalize Inventory Data → Check Sync Origin**
   - Use the **true** output for the next step (leave false unconnected unless you add logging for skipped events).

5. **Add “Prepare Inventory Payload” (Code node)**
   - Node type: **Code**
   - Paste code:
     ```js
     return [{
       json: {
         id: $('Normalize Inventory Data').first().json.body.id,
         sku: $('Normalize Inventory Data').first().json.body.sku,
         qty: Number($('Normalize Inventory Data').first().json.body.stock_quantity),
         dateModified: $('Normalize Inventory Data').first().json.body.date_modified,
         type: $('Normalize Inventory Data').first().json.body.type,
         origin: $('Normalize Inventory Data').first().json.body.source,
         status: $('Normalize Inventory Data').first().json.body.status
       }
     }];
     ```
   - Connect: **Check Sync Origin (true) → Prepare Inventory Payload**

6. **Add “Send Inventory To Secondary API” (HTTP Request node)**
   - Node type: **HTTP Request**
   - Method: **POST**
   - **Set the URL** to your secondary system endpoint (required).
   - Configure authentication as required by your API, for example:
     - **Header Auth**: `Authorization: Bearer <token>`
     - or **API Key** header/query
     - or OAuth2 (if supported by the API)
   - Body:
     - Enable **Send Body**
     - Add body parameters:
       - `id` = `{{$json.id}}`
       - `sku` = `{{$json.sku}}`
       - `qty` = `{{$json.qty}}`
       - `dateModified` = `{{$json.dateModified}}`
       - `type` = `{{$json.type}}`
       - `origin` = `{{$json.origin}}`
       - `status` = `{{$json.status}}`
   - Response:
     - Enable “Full Response” if you want status codes/headers available for later branching.
   - Connect: **Prepare Inventory Payload → Send Inventory To Secondary API**

7. **Add “Log Inventory Sync To Google Sheet” (Google Sheets node)**
   - Node type: **Google Sheets**
   - Credentials: create/select **Google Sheets OAuth2** credential with access to the target spreadsheet.
   - Operation: **Append or Update**
   - Document: select your spreadsheet (e.g., `Inventory_Sync_Log`)
   - Sheet: select tab (e.g., `Sheet1`)
   - Mapping mode: **Define Below**
   - Matching columns: **Id**
   - Map columns:
     - `Id` = `{{$json.id}}` (or reference the payload node explicitly)
     - `SKU` = `{{$json.sku}}`
     - `Type` = `{{$json.type}}`
     - `Quantity` = `{{$json.qty}}`
     - `Sync Status` = `{{$json.status}}`
     - `Date Modified` = `{{$json.dateModified}}` (**recommended fix**; the provided workflow maps this incorrectly)
     - `Source System` = `{{$json.origin}}`
   - Connect: **Send Inventory To Secondary API → Log Inventory Sync To Google Sheet**
   - Ensure the Google Sheet has headers matching the column names exactly: `Id, SKU, Type, Quantity, Sync Status, Date Modified, Source System`.

8. **(Optional) Add handling for filtered/skipped events**
   - Add a second Google Sheets log or a NoOp path on the **false** output of `Check Sync Origin` to record that the event was ignored due to origin.

9. **Activate workflow**
   - Switch workflow to **Active**.
   - Send a test POST request to the webhook and verify:
     - If `source != secondary-system`: API call occurs and row is logged/updated.
     - If `source == secondary-system`: flow stops (unless you add false-branch logging).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The HTTP Request node is missing its target URL in the provided workflow; you must set the secondary system API endpoint before activation. | Applies to “Send Inventory To Secondary API”. |
| “Date Modified” is currently mapped to `status` in Google Sheets; update it to `dateModified` to log correct timestamps. | Applies to “Log Inventory Sync To Google Sheet”. |
| The workflow intentionally drops events originating from `secondary-system` to prevent circular synchronization loops. | Implemented in “Check Sync Origin”. |
| Google Sheets file referenced: `Inventory_Sync_Log` | https://docs.google.com/spreadsheets/d/1tZp-X-3ZLwkj53h7BKhM5DbZKf3zVOtg-kcOIjmrA4w/edit?usp=drivesdk |