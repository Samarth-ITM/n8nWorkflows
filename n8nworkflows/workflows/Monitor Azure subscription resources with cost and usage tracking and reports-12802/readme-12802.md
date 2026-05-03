Monitor Azure subscription resources with cost and usage tracking and reports

https://n8nworkflows.xyz/workflows/monitor-azure-subscription-resources-with-cost-and-usage-tracking-and-reports-12802


# Monitor Azure subscription resources with cost and usage tracking and reports

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Monitor Azure subscription resources with cost and usage tracking and reports  
**Purpose:** Collect the full inventory of resources in an Azure subscription, pull cost data for a configurable date range, merge both datasets, compute summaries (total cost, cost by type), identify top-cost resources, and generate human-friendly **text + HTML** reports. Optional (currently disabled) outputs include **Excel export**, **Power BI push**, and **webhook response**.

### 1.1 Entry & Configuration
Starts manually, then sets core identifiers (subscription/tenant) and the billing period window (defaults to current month).

### 1.2 Parallel Data Collection (Azure APIs)
Runs two HTTP calls in parallel:
- **Azure Resource Graph**: all resources and metadata.
- **Cost Management Query API**: cost grouped by resource dimensions for the chosen period.

### 1.3 Merge, Compute Metrics, and Format Reports
Transforms both API responses into row/column objects, merges costs into resources, sorts by spend, aggregates totals and cost-by-type, then renders a text and an HTML report.

### 1.4 Output Routing (Data present vs not)
If a total cost exists, outputs prepared report fields and (optionally) exports/sends. Otherwise returns an error message.

---

## 2. Block-by-Block Analysis

### Block 1 — Entry & Configuration
**Overview:** Provides a manual entry point and defines all runtime parameters used downstream (subscription scope and time window).  
**Nodes involved:** `Manual Trigger`, `Set Configuration`

#### Node: Manual Trigger
- **Type / role:** `Manual Trigger` — workflow entry point for ad-hoc runs.
- **Configuration choices:** No parameters.
- **Inputs/outputs:** No inputs. Outputs one empty item to start execution.
- **Edge cases:** None (only manual execution).
- **Sticky note:** “How it works” (overall explanation + setup steps).

#### Node: Set Configuration
- **Type / role:** `Set` — defines subscription/timeframe variables.
- **Configuration choices (interpreted):**
  - `subscriptionId` (string): placeholder `YOUR_SUBSCRIPTION_ID`
  - `tenantId` (string): placeholder `YOUR_TENANT_ID`
  - `billingPeriod` (string): `{{$now.format('yyyyMM')}}`
  - `startDate` (string): `{{$now.startOf('month').format('yyyy-MM-dd')}}`
  - `endDate` (string): `{{$now.endOf('day').format('yyyy-MM-dd')}}`
- **Key expressions/variables:**
  - Uses `$now` (n8n date/time helper) to default to “current month to today end-of-day”.
- **Connections:**
  - **Input:** `Manual Trigger`
  - **Outputs:** `Query Azure Resources`, `Get Cost Data` (parallel fan-out)
- **Version specifics:** Set node v3.3 (supports “assignments” structure).
- **Edge cases / failures:**
  - If placeholders aren’t replaced, downstream API calls will fail (404/401 depending on endpoint).
  - Date range must be valid for Cost Management “Custom” timeframe; malformed strings cause API 400.
- **Sticky note:** “Configuration”.

---

### Block 2 — Parallel Azure Data Collection
**Overview:** Queries Azure management endpoints using OAuth2 (client credentials) to fetch resource inventory and cost data concurrently.  
**Nodes involved:** `Query Azure Resources`, `Get Cost Data`

#### Node: Query Azure Resources
- **Type / role:** `HTTP Request` — calls Azure Resource Graph.
- **Configuration choices (interpreted):**
  - **Method:** implicit POST (because a JSON body is sent; in n8n HTTP Request this typically defaults to GET unless method specified—here `sendBody: true` and JSON body are set, so you should explicitly set **POST** when rebuilding to match Azure API expectations).
  - **URL:** `https://management.azure.com/subscriptions/{{ $json.subscriptionId }}/providers/Microsoft.ResourceGraph/resources?api-version=2021-03-01`
  - **Body (JSON):** Resource Graph query:
    - `query`: `Resources | project name, type, location, resourceGroup, tags, sku, properties, id`
    - `subscriptions`: `[subscriptionId]`
  - **Auth:** Predefined credential type `oAuth2Api` (Azure OAuth2 API)
- **Inputs/outputs:**
  - **Input:** `Set Configuration` (uses `$json.subscriptionId`)
  - **Output:** `Merge and Process Data`
- **Expected response shape:** Resource Graph typically returns `{ data: { columns: [...], rows: [...] }, ... }`
- **Version specifics:** HTTP Request v4.2.
- **Edge cases / failures:**
  - OAuth token scope/roles missing → 401/403.
  - Resource Graph provider not registered or wrong API version → 404/400.
  - Large subscriptions may return paginated/limited results; this workflow does not handle pagination/continuation tokens.
- **Sticky note:** “Data Collection”.

#### Node: Get Cost Data
- **Type / role:** `HTTP Request` — calls Azure Cost Management Query API.
- **Configuration choices (interpreted):**
  - **Method:** same caveat as above; Azure Cost query expects **POST** with JSON body.
  - **URL:** `https://management.azure.com/subscriptions/{{ subscriptionId }}/providers/Microsoft.CostManagement/query?api-version=2023-11-01`
    - Uses `{{ $('Set Configuration').item.json.subscriptionId }}`
  - **Body (JSON):**
    - `type`: `ActualCost`
    - `timeframe`: `Custom`
    - `timePeriod.from/to`: from `startDate` to `endDate`
    - `dataset.granularity`: `Daily`
    - `aggregation`: Sum of `Cost` and `CostUSD`
    - `grouping` dimensions:
      - `ResourceId`
      - `ServiceName`
      - `ResourceType`
- **Inputs/outputs:**
  - **Input:** `Set Configuration` (references it explicitly with `$('Set Configuration')...`)
  - **Output:** `Merge and Process Data`
- **Expected response shape:** `{ properties: { columns: [...], rows: [...] } }`
- **Version specifics:** HTTP Request v4.2.
- **Edge cases / failures:**
  - Missing `Cost Management Reader` role → 403.
  - Cost data latency: some costs may not be available for “today”.
  - Large date ranges can produce large result sets; no paging/continuation handling.
  - If API returns empty rows, downstream processing may output totalCost “0.00” (still “not empty”).
- **Sticky note:** “Data Collection”.

---

### Block 3 — Merge, Compute, and Report Formatting
**Overview:** Converts both API outputs into arrays of objects, attempts to match costs to resources, computes totals and breakdowns, then renders text and HTML reports.  
**Nodes involved:** `Merge and Process Data`, `Format Report`

#### Node: Merge and Process Data
- **Type / role:** `Code` — data transformation + analytics.
- **Configuration choices (interpreted):**
  - Reads **first input** as Resource Graph response (`$input.first().json.data`)
  - Reads **last input** as Cost Management response (`$input.last().json.properties`)
  - Converts `columns + rows` to object arrays for both datasets.
  - Merges by searching cost `ResourceId` string for `resource.name` (case-insensitive).
  - Produces:
    - `allResources`: merged, sorted desc by cost
    - `summary`: totalCost, resourceCount, period, topCostResources (top 10), costByType
- **Key expressions/variables:**
  - `$('Set Configuration').item.json.startDate/endDate` used to build `summary.period`.
- **Connections:**
  - **Inputs:** from both `Query Azure Resources` and `Get Cost Data` (two incoming connections)
  - **Output:** `Format Report`
- **Version specifics:** Code node v2.
- **Edge cases / failures (important):**
  - **Fragile join logic:** `cost.ResourceId.toLowerCase().includes(resource.name.toLowerCase())`
    - This can mismatch (resource name substring collisions), miss matches (resource name not in ID as expected), or match multiple resources incorrectly.
    - More reliable approach would match by full `resource.id` equals `ResourceId` (normalized) rather than name-contains.
  - If either API response schema differs (no `.data` or no `.properties`) → runtime error.
  - If costs contain strings instead of numbers, `(cost.Cost || 0)` may concatenate/NaN depending on type.
  - Memory/time: large subscriptions + long ranges may be heavy in Code node.

#### Node: Format Report
- **Type / role:** `Code` — produces formatted reports (Markdown-like text + HTML).
- **Configuration choices (interpreted):**
  - Builds `textReport` with:
    - Period, total cost, total resources
    - Top 10 resources (name/type/cost)
    - Top 10 resource types by cost (type/count/cost)
  - Builds `htmlReport` with styled tables for the same data.
  - Outputs: `{ textReport, htmlReport, summary, allData }`
- **Connections:**
  - **Input:** `Merge and Process Data`
  - **Output:** `Check If Data Exists`
- **Version specifics:** Code node v2.
- **Edge cases / failures:**
  - If `summary` missing or malformed, report generation fails.
  - Assumes currency is USD (`$` sign) though cost query may return subscription currency; potential semantic mismatch.

---

### Block 4 — Conditional Routing & Outputs
**Overview:** Checks whether cost data exists, then either emits report fields and optional exports, or returns a “no data” message.  
**Nodes involved:** `Check If Data Exists`, `Output Report`, `No Data Found`, plus disabled optional nodes: `Export to Excel`, `Prepare Power BI Data`, `Send to Power BI`, `Respond to Webhook`

#### Node: Check If Data Exists
- **Type / role:** `IF` — validates presence of total cost.
- **Condition:** `{{$json.summary.totalCost}}` **isNotEmpty**
- **Connections:**
  - **Input:** `Format Report`
  - **True output:** `Output Report`
  - **False output:** `No Data Found`
- **Edge cases:**
  - If totalCost is `"0.00"`, it is still “not empty” → will route to **Output Report** even when effectively zero spend.
  - If `summary` is missing, condition evaluation may error or evaluate empty depending on runtime.

#### Node: Output Report
- **Type / role:** `Set` — normalizes final output fields for downstream consumers.
- **Fields set:**
  - `report` = `textReport`
  - `htmlReport` = `htmlReport`
  - `totalCost` = `summary.totalCost` (string)
  - `resourceCount` = `summary.resourceCount` (number)
- **Connections:**
  - **Input:** `Check If Data Exists` (true branch)
  - **Outputs:** `Export to Excel` (disabled), `Prepare Power BI Data` (disabled), `Respond to Webhook` (disabled)
- **Version specifics:** Set node v3.3.
- **Edge cases:** None beyond upstream data integrity.
- **Sticky note:** “Output Options”.

#### Node: Export to Excel (disabled)
- **Type / role:** `Spreadsheet File` — converts JSON array to XLSX.
- **Configuration choices:**
  - Mode: JSON → Spreadsheet
  - Input JSON: `{{$json.allData.allResources}}`
  - Filename: `azure-cost-report-{{$now.format('yyyy-MM-dd')}}.xlsx`
  - Header row enabled
- **Connections:**
  - **Input:** `Output Report`
  - **Output:** none downstream (not connected further)
- **Edge cases:**
  - If `allResources` is large, file generation may be slow/large.
  - Node config shows `operation: toJson` while mode is `jsonToSpreadsheet`; when rebuilding, ensure operation/mode are consistent with current n8n UI.
- **Sticky note:** “Output Options”.

#### Node: Prepare Power BI Data (disabled)
- **Type / role:** `Set` — shapes payload for Power BI push.
- **Fields set:**
  - `summary` (object) = `allData.summary`
  - `resources` (array) = `allData.allResources`
  - `timestamp` = `{{$now.toISO()}}`
  - `reportType` = `azure-cost-analysis`
- **Connections:**
  - **Input:** `Output Report`
  - **Output:** `Send to Power BI`
- **Edge cases:** Power BI datasets usually need a schema; sending nested objects/arrays may not match expected row format.

#### Node: Send to Power BI (disabled)
- **Type / role:** `HTTP Request` — pushes rows to a Power BI dataset endpoint.
- **Configuration choices:**
  - POST to placeholder URL: `https://api.powerbi.com/beta/YOUR_WORKSPACE/datasets/YOUR_DATASET/rows?key=YOUR_TOKEN_HERE`
  - Body: entire current JSON `{{$json}}`
- **Connections:**
  - **Input:** `Prepare Power BI Data`
- **Edge cases:**
  - Auth: URL uses a key in querystring (push dataset key). Incorrect key → 401/403.
  - Payload format likely incorrect for “add rows” API (often expects array of rows).

#### Node: Respond to Webhook (disabled)
- **Type / role:** `Respond to Webhook` — returns a JSON response to an incoming webhook trigger (not present/enabled in this workflow).
- **Configuration choices:**
  - Status 200
  - Headers: `Content-Type: application/json`, `Access-Control-Allow-Origin: *`
  - Response JSON includes `summary`, `totalCost`, `resourceCount`, `report`, and `timestamp`.
- **Connections:**
  - **Input:** `Output Report`
- **Edge cases:**
  - Without a **Webhook Trigger** node, this node has no practical effect; enabling it alone won’t make the workflow callable via HTTP.

#### Node: No Data Found
- **Type / role:** `Set` — error output when IF condition fails.
- **Fields set:** `error` = `No cost data found for the specified period`
- **Connections:**
  - **Input:** `Check If Data Exists` (false branch)
- **Edge cases:** If IF condition never routes false due to “0.00 is not empty”, this branch may be unused.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | Sticky Note | Documentation / overview | — | — | ## How it works; Setup steps (Service Principal, OAuth2 client credentials, update Set Configuration) |
| Sticky Note - Config | Sticky Note | Documentation / config guidance | — | — | ## Configuration: set subscription/tenant and billing period dates |
| Sticky Note - Data Collection | Sticky Note | Documentation / data collection guidance | — | — | ## Data Collection: Resource Graph + Cost Management in parallel |
| Sticky Note - Processing | Sticky Note | Documentation / processing guidance | — | — | ## Processing & Reporting: merge, totals, top resources, text/HTML |
| Sticky Note - Outputs | Sticky Note | Documentation / output options | — | — | ## Output Options: Excel, Power BI, webhook JSON (nodes disabled) |
| Manual Trigger | Manual Trigger | Manual entry point | — | Set Configuration | ## How it works; Setup steps (Service Principal, OAuth2 client credentials, update Set Configuration) |
| Set Configuration | Set | Define subscription/time window variables | Manual Trigger | Query Azure Resources; Get Cost Data | ## Configuration: set subscription/tenant and billing period dates |
| Query Azure Resources | HTTP Request | Fetch resource inventory via Resource Graph | Set Configuration | Merge and Process Data | ## Data Collection: Resource Graph + Cost Management in parallel |
| Get Cost Data | HTTP Request | Fetch cost data via Cost Management Query API | Set Configuration | Merge and Process Data | ## Data Collection: Resource Graph + Cost Management in parallel |
| Merge and Process Data | Code | Normalize responses, merge cost into resources, aggregate | Query Azure Resources; Get Cost Data | Format Report | ## Processing & Reporting: merge, totals, top resources, text/HTML |
| Format Report | Code | Render text and HTML reports | Merge and Process Data | Check If Data Exists | ## Processing & Reporting: merge, totals, top resources, text/HTML |
| Check If Data Exists | IF | Route based on presence of total cost | Format Report | Output Report (true); No Data Found (false) | ## Processing & Reporting: merge, totals, top resources, text/HTML |
| Output Report | Set | Finalize key output fields | Check If Data Exists (true) | Export to Excel; Prepare Power BI Data; Respond to Webhook | ## Output Options: Excel, Power BI, webhook JSON (nodes disabled) |
| Export to Excel | Spreadsheet File | Optional XLSX export (disabled) | Output Report | — | ## Output Options: Excel, Power BI, webhook JSON (nodes disabled) |
| Prepare Power BI Data | Set | Optional Power BI payload shaping (disabled) | Output Report | Send to Power BI | ## Output Options: Excel, Power BI, webhook JSON (nodes disabled) |
| Send to Power BI | HTTP Request | Optional Power BI push (disabled) | Prepare Power BI Data | — | ## Output Options: Excel, Power BI, webhook JSON (nodes disabled) |
| Respond to Webhook | Respond to Webhook | Optional HTTP response (disabled; no webhook trigger) | Output Report | — | ## Output Options: Excel, Power BI, webhook JSON (nodes disabled) |
| No Data Found | Set | Error output when no data | Check If Data Exists (false) | — | ## Processing & Reporting: merge, totals, top resources, text/HTML |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and set the workflow name to:  
   `Monitor Azure subscription resources with cost and usage tracking`

2. **Add “Manual Trigger”** node
   - Node type: **Manual Trigger**
   - No configuration needed.

3. **Add “Set Configuration”** node
   - Node type: **Set**
   - Add fields:
     - `subscriptionId` (String) = `YOUR_SUBSCRIPTION_ID`
     - `tenantId` (String) = `YOUR_TENANT_ID`
     - `billingPeriod` (String) = `{{$now.format('yyyyMM')}}`
     - `startDate` (String) = `{{$now.startOf('month').format('yyyy-MM-dd')}}`
     - `endDate` (String) = `{{$now.endOf('day').format('yyyy-MM-dd')}}`
   - Connect: **Manual Trigger → Set Configuration**

4. **Create Azure OAuth2 credentials (client credentials)**
   - In **Azure Portal**:
     1) Create an **App Registration**  
     2) Create a **Client Secret**  
     3) Assign roles to the Service Principal at subscription scope:
        - **Reader**
        - **Cost Management Reader**
   - In **n8n Credentials**:
     - Credential type: **OAuth2 API**
     - Grant type: **Client Credentials**
     - Token URL: `https://login.microsoftonline.com/<TENANT_ID>/oauth2/v2.0/token`
     - Scope: `https://management.azure.com/.default`
     - Client ID / Client Secret: from App Registration

5. **Add “Query Azure Resources”** node
   - Node type: **HTTP Request**
   - Authentication: **Predefined Credential Type** → select your **OAuth2 API** credential
   - Method: **POST**
   - URL:  
     `https://management.azure.com/subscriptions/{{$json.subscriptionId}}/providers/Microsoft.ResourceGraph/resources?api-version=2021-03-01`
   - Body content type: **JSON**
   - JSON body:
     - `query`: `Resources | project name, type, location, resourceGroup, tags, sku, properties, id`
     - `subscriptions`: array containing `{{$json.subscriptionId}}`
   - Connect: **Set Configuration → Query Azure Resources**

6. **Add “Get Cost Data”** node
   - Node type: **HTTP Request**
   - Authentication: **Predefined Credential Type** → same Azure OAuth2 credential
   - Method: **POST**
   - URL:  
     `https://management.azure.com/subscriptions/{{ $('Set Configuration').item.json.subscriptionId }}/providers/Microsoft.CostManagement/query?api-version=2023-11-01`
   - Body content type: **JSON**
   - JSON body with:
     - `type`: `ActualCost`
     - `timeframe`: `Custom`
     - `timePeriod.from`: `{{ $('Set Configuration').item.json.startDate }}`
     - `timePeriod.to`: `{{ $('Set Configuration').item.json.endDate }}`
     - `dataset.granularity`: `Daily`
     - `dataset.aggregation.totalCost`: Sum of `Cost`
     - `dataset.aggregation.totalCostUSD`: Sum of `CostUSD`
     - `dataset.grouping`: `ResourceId`, `ServiceName`, `ResourceType`
   - Connect: **Set Configuration → Get Cost Data**

7. **Add “Merge and Process Data”** node
   - Node type: **Code**
   - Paste the logic (adapted from workflow):
     - Convert Resource Graph `data.columns/rows` into objects
     - Convert Cost query `properties.columns/rows` into objects
     - Merge: for each resource, select matching cost rows and sum costs
     - Sort by cost desc
     - Build summary: totalCost, resourceCount, period, top 10, costByType
   - Connect both:
     - **Query Azure Resources → Merge and Process Data**
     - **Get Cost Data → Merge and Process Data**

8. **Add “Format Report”** node
   - Node type: **Code**
   - Generate:
     - `textReport` (Markdown-like)
     - `htmlReport` (tables + CSS)
     - Include `summary` and `allData`
   - Connect: **Merge and Process Data → Format Report**

9. **Add “Check If Data Exists”** node
   - Node type: **IF**
   - Condition: String → Value1 `{{$json.summary.totalCost}}` → **isNotEmpty**
   - Connect: **Format Report → Check If Data Exists**

10. **Add “Output Report”** node (true branch)
   - Node type: **Set**
   - Fields:
     - `report` (String) = `{{$json.textReport}}`
     - `htmlReport` (String) = `{{$json.htmlReport}}`
     - `totalCost` (String) = `{{$json.summary.totalCost}}`
     - `resourceCount` (Number) = `{{$json.summary.resourceCount}}`
   - Connect: **Check If Data Exists (true) → Output Report**

11. **Add “No Data Found”** node (false branch)
   - Node type: **Set**
   - Field: `error` (String) = `No cost data found for the specified period`
   - Connect: **Check If Data Exists (false) → No Data Found**

12. **Optional outputs (disabled in the provided workflow)**
   - **Export to Excel** (Spreadsheet File):
     - Mode: JSON → Spreadsheet
     - JSON input: `{{$json.allData.allResources}}`
     - File name: `azure-cost-report-{{$now.format('yyyy-MM-dd')}}.xlsx`
     - Connect: **Output Report → Export to Excel**
   - **Power BI push**:
     - Add **Prepare Power BI Data** (Set) shaping summary/resources/timestamp/reportType
     - Add **Send to Power BI** (HTTP Request) POST to your push dataset endpoint
     - Connect: **Output Report → Prepare Power BI Data → Send to Power BI**
   - **Webhook response**:
     - Add a **Webhook Trigger** (not present in this workflow) if you want HTTP invocation
     - Then connect to processing and end with **Respond to Webhook**

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Create a Service Principal (App Registration) with **Reader** + **Cost Management Reader** roles; use Client Credentials OAuth2; scope `https://management.azure.com/.default` | From “How it works” sticky note setup steps |
| Update `Set Configuration` with subscription/tenant IDs and adjust start/end dates as needed (defaults to current month) | From “How it works” and “Configuration” sticky notes |
| Data collection uses **Azure Resource Graph** + **Cost Management API** in parallel | From “Data Collection” sticky note |
| Output nodes for Excel/Power BI/Webhook are present but disabled; enable only those needed | From “Output Options” sticky note |