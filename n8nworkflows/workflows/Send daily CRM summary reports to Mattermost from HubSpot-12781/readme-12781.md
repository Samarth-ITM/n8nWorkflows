Send daily CRM summary reports to Mattermost from HubSpot

https://n8nworkflows.xyz/workflows/send-daily-crm-summary-reports-to-mattermost-from-hubspot-12781


# Send daily CRM summary reports to Mattermost from HubSpot

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** Generate a daily CRM-style summary by pulling **Sales** and **Support** metrics from two external REST endpoints, aggregating them into KPIs, generating a “PDF-like” binary artifact, uploading it to **HubSpot Files**, then notifying a **Mattermost** channel with key stats and the HubSpot file identifier. If anything fails, it posts an error alert to Mattermost and responds to the webhook caller.

**Primary use cases**
- Daily business KPI snapshot for revenue/support teams
- Centralized storage of daily reports in HubSpot
- Team notification in Mattermost with fast, readable metrics

**Entry point:** Authenticated **Webhook Trigger** (POST `/daily-report`) with response handled by **Respond to Webhook** node.

### Logical blocks
1.1 **Input Reception & Configuration**: Webhook Trigger → Prepare Config  
1.2 **Data Collection & Validation (parallel branches)**: Fetch Sales Data + Fetch Support Data → IF checks  
1.3 **Merge & KPI Aggregation**: Combine Results → Aggregate Metrics  
1.4 **Document Generation**: Generate PDF (binary attachment)  
1.5 **Storage & Success Notification**: Upload to HubSpot → Validate upload → Craft message → Send to Mattermost → Respond  
1.6 **Centralized Error Handling**: Any failure → Compose Error Message → Send Mattermost Error → Respond

> Note: The provided human title (“Send daily CRM summary reports to Mattermost from HubSpot”) does not exactly match the implemented flow. The workflow **uploads to HubSpot** after building the report from external APIs; it does not read “from HubSpot”.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception & Configuration
**Overview:** Receives a POST call and injects runtime configuration (endpoints, channel, report date parameters) used downstream by both data collection branches.

**Nodes involved**
- Webhook Trigger
- Prepare Config

#### Node: Webhook Trigger
- **Type / role:** `Webhook` — entry point, HTTP trigger.
- **Key configuration:**
  - **Path:** `daily-report`
  - **Method:** `POST`
  - **Response mode:** `responseNode` (workflow will end with a Respond to Webhook node)
- **Inputs / outputs:**
  - **Output:** to **Prepare Config**
- **Edge cases / failures:**
  - If authentication is expected “externally” (reverse proxy/API gateway) but not configured in node options, unauthorized calls may still reach workflow.
  - If the caller expects immediate response but flow is slow (API delays), ensure the caller timeout accommodates runtime.

#### Node: Prepare Config
- **Type / role:** `Set` — intended to centralize variables like `salesEndpoint`, `supportEndpoint`, `mattermostChannel`, date overrides.
- **Configuration choices (as implemented):**
  - Currently **no fields are set** (node has only default `options`).
  - Downstream nodes reference `{{$json.salesEndpoint}}` and `{{$json.supportEndpoint}}`; without Set fields or webhook payload containing these keys, calls may fail.
- **Inputs / outputs:**
  - **Input:** Webhook Trigger
  - **Outputs:** splits into parallel:
    - Fetch Sales Data
    - Fetch Support Data
- **Edge cases / failures:**
  - Missing config keys results in HTTP Request nodes receiving an empty/invalid URL expression.
- **Recommendation:** Add fields:
  - `salesEndpoint` (string URL)
  - `supportEndpoint` (string URL)
  - `mattermostChannel` (string, e.g. `DAILY_REPORTS`)
  - optionally `reportDate` (YYYY-MM-DD) if you want date control from caller

---

### 2.2 Data Collection & Validation (parallel)
**Overview:** Calls two REST endpoints in parallel (sales, support). Each branch validates the response “status code” and routes failures to error handling.

**Nodes involved**
- Fetch Sales Data
- Sales Data OK?
- Fetch Support Data
- Support Data OK?

#### Node: Fetch Sales Data
- **Type / role:** `HTTP Request` — fetch sales stats JSON.
- **Key configuration:**
  - **URL:** `={{ $json.salesEndpoint }}`
  - Auth is not shown in parameters; expected to be configured via node credentials/options in editor.
- **Inputs / outputs:**
  - **Input:** Prepare Config
  - **Output:** Sales Data OK?
- **Edge cases / failures:**
  - Invalid/missing URL from config.
  - Auth/401, DNS, TLS, timeout, non-JSON response.
  - By default, many HTTP Request configurations place the HTTP status in metadata, not in `$json.statusCode` unless explicitly enabled (see IF node note below).

#### Node: Sales Data OK?
- **Type / role:** `IF` — validates sales response.
- **Key configuration:**
  - Condition: number `value1 = {{ $json.statusCode || 200 }}` equals `200`
  - This effectively treats missing `statusCode` as success (defaults to 200).
- **Routing:**
  - **True:** Combine Results (Merge input 0)
  - **False:** Compose Error Message
- **Edge cases / failures:**
  - Because of `|| 200`, **errors can be missed** if `statusCode` is not present. A failing response that still returns JSON without `statusCode` could pass validation.
  - If HTTP Request node is configured to “Continue On Fail”, the response shape may differ (error fields instead of data).

#### Node: Fetch Support Data
- **Type / role:** `HTTP Request` — fetch support ticket stats JSON.
- **Key configuration:**
  - **URL:** `={{ $json.supportEndpoint }}`
- **Inputs / outputs:**
  - **Input:** Prepare Config
  - **Output:** Support Data OK?
- **Edge cases / failures:** Same as sales branch (missing URL, auth, timeouts, response format).

#### Node: Support Data OK?
- **Type / role:** `IF` — validates support response.
- **Key configuration:**
  - Condition: number `value1 = {{ $json.statusCode || 200 }}` equals `200`
- **Routing:**
  - **True:** Combine Results (Merge input 1)
  - **False:** Compose Error Message
- **Edge cases / failures:** Same validation weakness as Sales Data OK?

---

### 2.3 Merge & KPI Aggregation
**Overview:** Merges the two datasets into one item and computes consolidated KPIs (revenue + open tickets), while preserving raw source payloads.

**Nodes involved**
- Combine Results
- Aggregate Metrics

#### Node: Combine Results
- **Type / role:** `Merge` — combines parallel branch outputs.
- **Key configuration:**
  - **Mode:** `mergeByIndex`
  - This expects both branches to output items in matching order/index.
- **Inputs / outputs:**
  - **Input 0:** Sales Data OK? (true path)
  - **Input 1:** Support Data OK? (true path)
  - **Output:** Aggregate Metrics
- **Edge cases / failures:**
  - If one branch produces 0 items (e.g., filtered out or failed), merge may produce no output or mismatched structure.
  - If multiple items are produced in either branch, merge-by-index pairing may not reflect intended semantics.

#### Node: Aggregate Metrics
- **Type / role:** `Code` — compute normalized KPIs and a report date.
- **Key logic:**
  - Assumes:
    - `items[0].json` = sales data
    - `items[1].json` = support data
  - Revenue normalization:
    - `salesData.totalRevenue || salesData.total_revenue || 0`
  - Ticket normalization:
    - `supportData.open_tickets || supportData.ticketsOpen || 0`
  - Output JSON:
    - `reportDate`: today in `YYYY-MM-DD` via `new Date().toISOString().slice(0,10)`
    - `totalRevenue`, `ticketsOpen`, plus full `salesData`, `supportData`
- **Inputs / outputs:**
  - **Input:** Combine Results
  - **Output:** Generate PDF
- **Edge cases / failures:**
  - If merge output is not two items as expected, `items[1]` may be undefined and crash.
  - Timezone: `toISOString()` uses UTC date; if you want local date, you must adjust.

---

### 2.4 Document Generation
**Overview:** Creates a simple “PDF” binary attachment from the KPI text and attaches it to the item as `binary.data`.

**Nodes involved**
- Generate PDF

#### Node: Generate PDF
- **Type / role:** `Code` — generate binary content (base64) and set PDF metadata.
- **Key logic:**
  - Creates a plain text string (`pdfContent`) with:
    - Daily report header
    - Total revenue
    - Open tickets
  - Converts to Buffer and stores as base64
  - Writes binary:
    - `mimeType: 'application/pdf'`
    - `fileName: daily_report_<date>.pdf`
- **Inputs / outputs:**
  - **Input:** Aggregate Metrics
  - **Output:** Upload Report to HubSpot
- **Important technical note:**
  - This is **not a real PDF render** (no PDF structure). Many systems may still store it, but PDF viewers may not open it reliably.
- **Edge cases / failures:**
  - HubSpot may reject malformed PDFs depending on validation.
  - File size generally small here, but binary handling requires n8n binary data to be enabled (default).

---

### 2.5 Storage & Success Notification
**Overview:** Uploads the generated file to HubSpot, validates upload success, crafts a Markdown message, posts to Mattermost, then responds to the webhook caller.

**Nodes involved**
- Upload Report to HubSpot
- HubSpot Upload OK?
- Craft Mattermost Message
- Send Mattermost Notification
- Respond to Caller

#### Node: Upload Report to HubSpot
- **Type / role:** `HubSpot` — file resource operation (upload).
- **Key configuration:**
  - `resource: file`
  - Operation is not explicit in JSON; in UI this must be set to the relevant **file upload** operation and mapped to the incoming binary property (commonly `binary.data`).
  - Requires HubSpot credentials (private app token or OAuth2 depending on node support).
- **Inputs / outputs:**
  - **Input:** Generate PDF
  - **Output:** HubSpot Upload OK?
- **Edge cases / failures:**
  - Missing credentials, insufficient scopes (files access), invalid binary field mapping.
  - HubSpot API limits and file size limits.
  - If the node is not configured to use `binary.data`, upload will fail or upload empty content.

#### Node: HubSpot Upload OK?
- **Type / role:** `IF` — validates that upload returned an ID.
- **Key configuration:**
  - String condition: `={{ $json.id || '' }}` is `notEmpty`
- **Routing:**
  - **True:** Craft Mattermost Message
  - **False:** Compose Error Message
- **Edge cases / failures:**
  - HubSpot file responses may use a different identifier field depending on operation/version (e.g., `fileId`); check actual output.

#### Node: Craft Mattermost Message
- **Type / role:** `Code` — format message text and select channel.
- **Key logic:**
  - `fileId = $json.id || 'unknown'`
  - Message stored in `mattermostText` (Markdown):
    - Date
    - Total revenue
    - Open tickets
    - HubSpot File ID
  - `channel = $json.mattermostChannel || 'DAILY_REPORTS'`
- **Inputs / outputs:**
  - **Input:** HubSpot Upload OK? (true)
  - **Output:** Send Mattermost Notification
- **Edge cases / failures:**
  - `mattermostChannel` is not created earlier (Prepare Config empty), so it will default to `DAILY_REPORTS`.

#### Node: Send Mattermost Notification
- **Type / role:** `Mattermost` — posts message to a channel.
- **Key configuration:**
  - `operation: create` (create a post/message)
  - Needs Mattermost credentials + mapping of:
    - Channel (from `json.channel`)
    - Message text (from `json.mattermostText`)
- **Inputs / outputs:**
  - **Input:** Craft Mattermost Message
  - **Output:** Respond to Caller
- **Edge cases / failures:**
  - Channel identifier format varies: some Mattermost nodes expect **channel ID**, not name. If you pass `DAILY_REPORTS` and it expects an ID, posting will fail.
  - Permission issues: bot/user not in channel.

#### Node: Respond to Caller
- **Type / role:** `Respond to Webhook` — sends HTTP response back to original webhook caller.
- **Key configuration:**
  - Options empty; default response payload will be whatever the node is configured to send in UI (often “last node data” or a fixed message).
- **Inputs / outputs:**
  - **Inputs:** from Send Mattermost Notification (success) and Send Mattermost Error (failure)
- **Edge cases / failures:**
  - If no explicit response body configured, callers may receive unexpected structure.
  - If the workflow errors before reaching this node, webhook caller may time out (but this design tries to route errors here).

---

### 2.6 Centralized Error Handling
**Overview:** Consolidates all failures into a single Mattermost alert and responds to the webhook caller.

**Nodes involved**
- Compose Error Message
- Send Mattermost Error
- Respond to Caller

#### Node: Compose Error Message
- **Type / role:** `Code` — builds an error notification payload.
- **Key logic (as implemented):**
  - Static message:
    - `❌ Daily Report workflow failed in one of the data or storage steps. Please review the execution logs.`
  - Static channel: `'DAILY_REPORTS'`
- **Inputs / outputs:**
  - **Inputs:** from Sales Data OK? (false), Support Data OK? (false), HubSpot Upload OK? (false)
  - **Output:** Send Mattermost Error
- **Edge cases / failures:**
  - Does not include actual failing node, status code, or error details (despite sticky note describing truncation logic). Troubleshooting requires checking n8n execution logs.
  - If Mattermost channel must be dynamic, this node currently ignores `mattermostChannel`.

#### Node: Send Mattermost Error
- **Type / role:** `Mattermost` — posts the error message.
- **Key configuration:** `operation: create`
- **Inputs / outputs:**
  - **Input:** Compose Error Message
  - **Output:** Respond to Caller
- **Edge cases / failures:**
  - Same channel ID vs name risk as the success notification.
  - If Mattermost is down, caller still gets response only if node is configured to continue on fail (not shown).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| 📝 Workflow Overview | Sticky Note | Documentation block | — | — | # Daily Report Generator; How it works; Setup steps (Sales/Support creds, HubSpot, Mattermost, endpoints, channel, webhook deploy/test) |
| ⚙️ Trigger & Config | Sticky Note | Documentation block | — | — | ## Trigger & Configuration (webhook + config Set; centralize endpoints/channel/date; test config carefully) |
| 📡 Data Collection | Sticky Note | Documentation block | — | — | ## Data Collection (parallel HTTP requests; validate status; route to error handling; merge on success) |
| 🛠️ Data Processing | Sticky Note | Documentation block | — | — | ## Processing & PDF Generation (aggregate KPIs; generate PDF via Buffer; keep metrics + binary) |
| 💾 Storage & Notification | Sticky Note | Documentation block | — | — | ## Storage & Notifications (upload to HubSpot; validate; craft Markdown; post to Mattermost; upload failure routes to error) |
| 🚨 Error Handling | Sticky Note | Documentation block | — | — | ## Error Handling (converge failures; compose concise alert; post to Mattermost; respond to webhook caller) |
| Webhook Trigger | Webhook | Entry point (POST trigger) | — | Prepare Config | Trigger & Configuration note applies |
| Prepare Config | Set | Inject endpoints/channel/date variables | Webhook Trigger | Fetch Sales Data; Fetch Support Data | Trigger & Configuration note applies |
| Fetch Sales Data | HTTP Request | Call sales stats endpoint | Prepare Config | Sales Data OK? | Data Collection note applies |
| Sales Data OK? | IF | Validate sales response | Fetch Sales Data | Combine Results (true); Compose Error Message (false) | Data Collection note applies |
| Fetch Support Data | HTTP Request | Call support stats endpoint | Prepare Config | Support Data OK? | Data Collection note applies |
| Support Data OK? | IF | Validate support response | Fetch Support Data | Combine Results (true); Compose Error Message (false) | Data Collection note applies |
| Combine Results | Merge | Merge sales + support payloads | Sales Data OK? (true); Support Data OK? (true) | Aggregate Metrics | Data Collection note applies |
| Aggregate Metrics | Code | Compute KPIs and normalized output | Combine Results | Generate PDF | Processing & PDF Generation note applies |
| Generate PDF | Code | Create binary “PDF” artifact | Aggregate Metrics | Upload Report to HubSpot | Processing & PDF Generation note applies |
| Upload Report to HubSpot | HubSpot | Upload binary file to HubSpot Files | Generate PDF | HubSpot Upload OK? | Storage & Notifications note applies |
| HubSpot Upload OK? | IF | Validate HubSpot upload returned ID | Upload Report to HubSpot | Craft Mattermost Message (true); Compose Error Message (false) | Storage & Notifications note applies |
| Craft Mattermost Message | Code | Build Markdown message + pick channel | HubSpot Upload OK? (true) | Send Mattermost Notification | Storage & Notifications note applies |
| Send Mattermost Notification | Mattermost | Post success message to Mattermost | Craft Mattermost Message | Respond to Caller | Storage & Notifications note applies |
| Compose Error Message | Code | Build error notification payload | Sales Data OK? (false); Support Data OK? (false); HubSpot Upload OK? (false) | Send Mattermost Error | Error Handling note applies |
| Send Mattermost Error | Mattermost | Post error message to Mattermost | Compose Error Message | Respond to Caller | Error Handling note applies |
| Respond to Caller | Respond to Webhook | Return HTTP response to webhook caller | Send Mattermost Notification; Send Mattermost Error | — | Trigger & Configuration note applies |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Name it e.g. `Daily Report Generator with Mattermost and HubSpot`.

2) **Add Webhook Trigger**
- Node: **Webhook**
- Method: `POST`
- Path: `daily-report`
- Response Mode: **Respond to Webhook node**
- (Optional) Add authentication in Webhook options (recommended).

3) **Add Prepare Config (Set node)**
- Node: **Set**
- Add fields (example):
  - `salesEndpoint` (String) e.g. `https://api.example.com/sales/daily?date={{$now.minus({days:1}).toISODate()}}`
  - `supportEndpoint` (String) e.g. `https://api.example.com/support/daily?date={{$now.minus({days:1}).toISODate()}}`
  - `mattermostChannel` (String) e.g. `DAILY_REPORTS` (or a Mattermost Channel ID, depending on your node)
- Connect: **Webhook Trigger → Prepare Config**

4) **Add the two HTTP Request nodes (parallel)**
- Node A: **HTTP Request** named `Fetch Sales Data`
  - URL: `{{$json.salesEndpoint}}`
  - Configure **Authentication** (Header/API Key/OAuth2) as required by your sales API.
- Node B: **HTTP Request** named `Fetch Support Data`
  - URL: `{{$json.supportEndpoint}}`
  - Configure authentication for your support API.
- Connect: **Prepare Config → Fetch Sales Data**
- Connect: **Prepare Config → Fetch Support Data**

5) **Add validation IF nodes**
- Add IF node `Sales Data OK?`
  - Prefer a robust check:
    - Either check an explicit field from your API (`success == true`)
    - Or configure HTTP Request to expose status code reliably, then test it
  - Connect: **Fetch Sales Data → Sales Data OK?**
- Add IF node `Support Data OK?`
  - Same validation strategy
  - Connect: **Fetch Support Data → Support Data OK?**

6) **Add Merge node**
- Node: **Merge** named `Combine Results`
- Mode: `Merge By Index`
- Connect:
  - **Sales Data OK? (true) → Combine Results (Input 1 / index 0)**
  - **Support Data OK? (true) → Combine Results (Input 2 / index 1)**

7) **Add KPI aggregation Code node**
- Node: **Code** named `Aggregate Metrics`
- Paste/adapt logic:
  - Read `items[0]` sales and `items[1]` support
  - Compute `totalRevenue`, `ticketsOpen`
  - Output a single item containing KPIs and raw payloads
- Connect: **Combine Results → Aggregate Metrics**

8) **Add PDF generation Code node**
- Node: **Code** named `Generate PDF`
- Create binary data at `binary.data` with `fileName` and `mimeType`.
- Connect: **Aggregate Metrics → Generate PDF**
- (Recommended improvement) Use a real PDF generation approach if HubSpot/viewers require valid PDFs.

9) **Add HubSpot upload**
- Node: **HubSpot** named `Upload Report to HubSpot`
- Resource: `File`
- Operation: select the **upload/create file** operation in UI
- Map the incoming binary field (commonly `data`)
- **Credentials:** set HubSpot OAuth2 or Private App token with files scope.
- Connect: **Generate PDF → Upload Report to HubSpot**

10) **Validate upload**
- Node: **IF** named `HubSpot Upload OK?`
- Condition: uploaded file identifier exists (e.g., `id` not empty, or the correct field from HubSpot output).
- Connect: **Upload Report to HubSpot → HubSpot Upload OK?**

11) **Craft Mattermost message**
- Node: **Code** named `Craft Mattermost Message`
- Output JSON with:
  - `mattermostText` (Markdown)
  - `channel` (from config or constant)
- Connect: **HubSpot Upload OK? (true) → Craft Mattermost Message**

12) **Send Mattermost success post**
- Node: **Mattermost** named `Send Mattermost Notification`
- Operation: `Create` (post message)
- **Credentials:** configure Mattermost (server URL, token/OAuth depending on node)
- Map:
  - Channel (ID/name as required)
  - Message = `{{$json.mattermostText}}`
- Connect: **Craft Mattermost Message → Send Mattermost Notification**

13) **Add error handling branch**
- Node: **Code** named `Compose Error Message`
  - Build `mattermostText` and `channel`
  - (Recommended) include actual error details (node name, HTTP status, message)
- Connect failure outputs:
  - **Sales Data OK? (false) → Compose Error Message**
  - **Support Data OK? (false) → Compose Error Message**
  - **HubSpot Upload OK? (false) → Compose Error Message**

14) **Send Mattermost error post**
- Node: **Mattermost** named `Send Mattermost Error`
- Operation: `Create`
- Use same credential as success node
- Connect: **Compose Error Message → Send Mattermost Error**

15) **Respond to webhook caller**
- Node: **Respond to Webhook** named `Respond to Caller`
- Configure response body (suggestion):
  - On success: include `reportDate`, `totalRevenue`, `ticketsOpen`, and HubSpot file ID.
  - On error: include a structured error object.
- Connect:
  - **Send Mattermost Notification → Respond to Caller**
  - **Send Mattermost Error → Respond to Caller**

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The workflow documentation claims “authenticated webhook”, but the Webhook node shows no explicit auth options configured. | Review Webhook node options / protect endpoint at gateway/reverse proxy. |
| The “Prepare Config” node is empty in the provided JSON, but downstream nodes require `salesEndpoint` and `supportEndpoint`. | Add those fields or pass them in webhook body. |
| The “Generate PDF” step writes plain text while labeling it `application/pdf`. | Consider generating a valid PDF if HubSpot or users need to open it reliably. |
| IF checks default to success when `statusCode` is missing (`$json.statusCode || 200`). | Prefer strict validation to avoid silent failures. |