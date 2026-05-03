Track Azure API and Service Bus failures with Application Insights correlation

https://n8nworkflows.xyz/workflows/track-azure-api-and-service-bus-failures-with-application-insights-correlation-12803


# Track Azure API and Service Bus failures with Application Insights correlation

## 1. Workflow Overview

**Purpose:**  
This workflow queries **Azure Application Insights** to identify failed (or optionally all) API requests and **correlates** them with **Service Bus traces** and **exceptions** using the shared `operationId` (Application Insights `operation_Id`). It then produces a **Markdown** and **HTML** failure analysis report, and optionally exports results (Excel) or returns them via webhook (both disabled by default).

**Target use cases:**
- Investigating API failures end-to-end across **APIM → backend → Service Bus**
- Building an on-demand operational report for incident review
- Correlating Application Insights telemetry streams (requests/traces/exceptions)

### 1.1 Entry & Configuration
Receives a manual start and sets App Insights/AAD parameters (app id, tenant, client credentials, time range, filters).

### 1.2 Data Collection (Application Insights API)
Authenticates to Azure AD using client credentials and runs **three KQL queries** (requests, traces, exceptions) against the App Insights Query API.

### 1.3 Correlation & Analysis
Transforms App Insights table format into JSON objects, groups traces/exceptions by `operationId`, correlates them to request records, and computes summary stats + top lists.

### 1.4 Report Generation & Output Routing
Builds Markdown + HTML reports, checks if there is any data, then outputs either a structured “report payload” or a “no data” message. Optional output nodes are present but disabled.

---

## 2. Block-by-Block Analysis

### Block 1 — Entry & Configuration

**Overview:**  
Starts the workflow manually and defines the parameters needed for Azure AD authentication and App Insights queries (including the time range and whether to include successful requests).

**Nodes involved:**
- Manual Trigger
- Set Configuration

#### Node: **Manual Trigger**
- **Type / role:** `n8n-nodes-base.manualTrigger` — interactive entry point for testing/running on demand.
- **Configuration choices:** No parameters.
- **Input/Output:**  
  - **Output →** Set Configuration
- **Edge cases / failures:** None (only runs when manually executed).

#### Node: **Set Configuration**
- **Type / role:** `n8n-nodes-base.set` — creates a config object used by later Code nodes.
- **Key configuration (interpreted):**
  - `appInsightsAppId`: Application Insights **Application ID** (not the resource ID).
  - `tenantId`: Azure AD tenant GUID.
  - `clientId`: service principal/app registration client id.
  - `clientSecret`: service principal secret.
  - `timeRange`: KQL `ago()` value as string (examples: `24h`, `7d`, `30d`).
  - `includeSuccessful`: boolean; when `false`, requests query filters to `resultCode >= 400`.
- **Key expressions / variables:** none (static placeholders in JSON).
- **Input/Output:**  
  - **Input ←** Manual Trigger  
  - **Output →** Query Application Insights
- **Version-specific notes:** Node typeVersion `3.3` (modern Set node with “assignments” model).
- **Edge cases / failures:**
  - Incorrect/missing values will later cause OAuth failures (AAD) or App Insights query failures.
  - Storing `clientSecret` directly in node parameters is functional but risky; credentials/variables are safer.

**Sticky note coverage (context):**
- “Configuration” note: reminds to set App ID and time range, and ensure OAuth2 credentials are configured—however, this workflow actually performs OAuth internally in Code (not via n8n OAuth credentials).

---

### Block 2 — Data Collection (Application Insights API)

**Overview:**  
Fetches an Azure AD token using the client credentials grant, then issues three POST requests to the Application Insights Query API using KQL to retrieve: APIM requests, Service Bus-related traces, and exceptions.

**Nodes involved:**
- Query Application Insights

#### Node: **Query Application Insights**
- **Type / role:** `n8n-nodes-base.code` — performs OAuth token acquisition + three App Insights KQL queries.
- **Configuration choices (interpreted):**
  - Reads config from the incoming item (`$input.item.json`).
  - **AAD token endpoint:** `https://login.microsoftonline.com/${tenantId}/oauth2/v2.0/token`
  - **Grant:** `client_credentials`
  - **Scope:** `https://api.applicationinsights.io/.default`
  - **App Insights Query API endpoint:** `https://api.applicationinsights.io/v1/apps/${appId}/query`
  - Executes three KQL queries (each `take 500` and ordered by timestamp desc).
- **Key logic / expressions:**
  - Throws if `access_token` missing:
    - `throw new Error('Failed to obtain access token: ' + JSON.stringify(tokenData))`
  - Returns a single item with:
    - `apimData: apimData.tables?.[0] || {}`
    - `sbData: sbData.tables?.[0] || {}`
    - `exceptionsData: exceptionsData.tables?.[0] || {}`
- **KQL query intent (summarized):**
  1. **APIM Requests** (`requests` table)
     - Filter: `timestamp > ago(timeRange)`
     - Optional filter: if `includeSuccessful` is false → `resultCode >= 400`
     - Extract custom dimensions:
       - `apim-service-name`, `apim-operation-name`, `apim-subscription-id`
     - Projects client geo/type fields and other request fields
  2. **Service Bus Traces** (`traces` table)
     - Filter by time range
     - Filter: message contains `ServiceBus` OR customDimensions has `MessageId`
     - Extract custom dimensions: `MessageId`, `EntityName`, `Endpoint`
  3. **Exceptions** (`exceptions` table)
     - Filter by time range
     - Projects messages/methods and `details`
- **Input/Output:**
  - **Input ←** Set Configuration
  - **Output →** Correlate and Analyze Data
- **Version-specific notes:** Code node typeVersion `2` (supports `fetch` in n8n’s runtime).
- **Edge cases / failure types:**
  - **Auth failures:** invalid tenant/client/secret, missing API permissions, expired secret → token request returns error JSON; workflow throws.
  - **Authorization:** service principal lacking access to the App Insights app → 403/401 from query API.
  - **API throttling/timeouts:** App Insights Query API can throttle; no retry/backoff implemented.
  - **Unexpected response shape:** if `tables[0]` missing, downstream sees `{}` and parses to empty arrays (graceful).
  - **KQL assumptions:** expected customDimensions keys may not exist (handled as `tostring(...)` which yields empty).
  - **Row limit:** `take 500` may hide additional failures in high-volume environments.

**Sticky note coverage (context):**
- “Data Collection” note states “Handles OAuth2 authentication internally.” This matches the Code implementation.

---

### Block 3 — Correlation & Analysis

**Overview:**  
Converts the App Insights “table” responses into object arrays, groups traces and exceptions by `operationId`, correlates them to request records, and computes summary metrics plus top errors and slowest requests.

**Nodes involved:**
- Correlate and Analyze Data

#### Node: **Correlate and Analyze Data**
- **Type / role:** `n8n-nodes-base.code` — transforms and correlates telemetry; calculates analytics.
- **Configuration choices (interpreted):**
  - `parseAppInsightsTable(table)`:
    - Converts `{ columns: [{name...}], rows: [...] }` into `[ {colName: value, ...}, ... ]`
    - Returns `[]` if structure missing.
  - Builds maps:
    - `sbByOperationId[operationId] = [trace, ...]`
    - `exceptionsByOperationId[operationId] = [exception, ...]`
  - Correlates by iterating over `apimRequests` and attaching:
    - `serviceBusTraces`, `serviceBusMessageIds`
    - `exceptions`, `exceptionMessages`
    - Flags: `hasException`, `hasServiceBusTrace`, `isFailure`
- **Key expressions / variables:**
  - References config timeRange via node lookup:
    - `timeRange: $('Set Configuration').item.json.timeRange`
- **Computed outputs:**
  - `summary`:
    - totals, failed/successful counts, counts with exceptions/traces
    - average duration (see edge case below)
    - analysis timestamp
  - `topErrors`:
    - Count by `${resultCode} - ${apiName}`, top 10
  - `topSlowRequests`:
    - Sort by duration desc, top 10
  - `correlatedData`: enriched per-request objects
  - `rawData`: counts of raw rows per source
- **Input/Output:**
  - **Input ←** Query Application Insights
  - **Output →** Generate Report
- **Version-specific notes:** Code node typeVersion `2`.
- **Edge cases / failure types:**
  - **Division by zero:** `averageDuration` does `... / correlatedData.length`. If there are **zero** APIM requests, this becomes `NaN` (and later `.toFixed()` can throw). The workflow tries to guard later with “Check If Data Exists”, but report generation happens **before** the IF check, so this can still break execution.
  - **Type coercion:** `resultCode` from App Insights can be string; comparisons like `>= 400` may behave unexpectedly if not numeric. (KQL `resultCode` in `requests` is often string.)
  - **OperationId missing:** items without `operationId` will not correlate; they still appear but without linked traces/exceptions.

---

### Block 4 — Report Generation & Output Routing

**Overview:**  
Generates Markdown and HTML reports from the analytics, checks whether any correlated data exists, then emits either the report payload or a “no data” message. Optional export/webhook response nodes exist but are disabled.

**Nodes involved:**
- Generate Report
- Check If Data Exists
- Output Report
- No Data Found
- Export to Excel (disabled)
- Respond to Webhook (disabled)

#### Node: **Generate Report**
- **Type / role:** `n8n-nodes-base.code` — builds human-friendly Markdown and HTML strings.
- **Configuration choices (interpreted):**
  - Builds `markdownReport` with summary, top errors, and slow requests.
  - Builds `htmlReport` with styled cards and lists.
- **Key expressions / variables:**
  - Failure rate: `((failedRequests/totalRequests)*100).toFixed(1)`
  - Average duration: `summary.averageDuration.toFixed(2)`
  - Slow request duration formatting: `r.duration.toFixed(2)`
- **Input/Output:**
  - **Input ←** Correlate and Analyze Data
  - **Output →** Check If Data Exists
- **Edge cases / failure types:**
  - If `summary.totalRequests` is `0`, several `.toFixed()` calls and division will produce `Infinity/NaN` and may throw (depending on the value).
  - If any `duration` is null/undefined, `toFixed()` can throw. (You partially guard earlier with `(r.duration || 0)` for sorting, but not when formatting in the report.)
  - Large HTML/Markdown could exceed downstream size limits if later sent via webhook/email (not present here but common).

#### Node: **Check If Data Exists**
- **Type / role:** `n8n-nodes-base.if` — routes flow depending on whether there are correlated records.
- **Condition (interpreted):**
  - If `{{$json.correlatedData.length}} > 0` → “true” branch
  - Else → “false” branch
- **Input/Output:**
  - **Input ←** Generate Report
  - **True →** Output Report
  - **False →** No Data Found
- **Version-specific notes:** IF node typeVersion `2` with strict validation enabled.
- **Edge cases / failure types:**
  - If `correlatedData` is missing/not an array, expression may error. In normal flow it exists.

#### Node: **Output Report**
- **Type / role:** `n8n-nodes-base.set` — prepares a clean output payload for downstream use.
- **Configuration choices (interpreted):**
  - `report`: Markdown report string
  - `htmlReport`: HTML report string
  - `summary`: summary object
  - `topErrors`: array
  - `failedRequests`: filters correlatedData to failures via inline JS:
    - `={{ $json.correlatedData.filter(r => r.isFailure) }}`
- **Input/Output:**
  - **Input ←** Check If Data Exists (true)
  - **Output →** Export to Excel (disabled) and Respond to Webhook (disabled)
- **Version-specific notes:** Set node typeVersion `3.3`.
- **Edge cases / failure types:**
  - If `correlatedData` is huge, filtering inline is fine but could increase memory usage.

#### Node: **No Data Found**
- **Type / role:** `n8n-nodes-base.set` — emits a message when nothing matched.
- **Configuration choices (interpreted):**
  - Sets `message` explaining no data found.
- **Input/Output:**
  - **Input ←** Check If Data Exists (false)
  - No outgoing connections.
- **Version-specific notes:** Set node typeVersion `3.3`.

#### Node: **Export to Excel** (disabled)
- **Type / role:** `n8n-nodes-base.spreadsheetFile` — would export items to an `.xlsx` file.
- **Configuration choices (interpreted):**
  - Operation: create file (`toFile`)
  - Format: `xlsx`
  - File name pattern: `azure-api-failures-{{ $now.format('yyyy-MM-dd-HHmmss') }}.xlsx`
  - Includes header row.
- **Input/Output:**
  - **Input ←** Output Report
  - No defined output in connections (node is present as optional).
- **Edge cases / failure types:**
  - Disabled by default; if enabled, ensure the incoming data is shaped as rows you want exported (currently the workflow output is a single item with nested arrays/objects; exporting that directly may not yield a useful spreadsheet without additional flattening).

#### Node: **Respond to Webhook** (disabled)
- **Type / role:** `n8n-nodes-base.respondToWebhook` — would return a JSON response to an upstream Webhook trigger.
- **Configuration choices (interpreted):**
  - Responds with JSON body:
    - `{ status: 'success', data: { summary, topErrors, failedRequests, report }, timestamp }`
- **Input/Output:**
  - **Input ←** Output Report
- **Edge cases / failure types:**
  - This workflow has **no Webhook Trigger** node. Enabling this node alone won’t make it callable via HTTP; you’d need to add a Webhook trigger and route into this flow.

**Sticky note coverage (context):**
- “Processing & Reporting” note matches correlation + reporting nodes.
- “Output Options” note correctly states Excel and webhook response are disabled.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | n8n-nodes-base.stickyNote | Documentation / visual guidance |  |  | ## How it works\n\nThis workflow queries Azure Application Insights to track failed API calls across APIM, Service Bus, and exceptions. It makes three KQL queries to the Application Insights API, then correlates the results using operationId to show which API calls failed, what exceptions occurred, and which Service Bus messages were involved.\n\n## Setup steps\n\n1. **Create service principal**: Run `az ad sp create-for-rbac --name "n8n-appinsights" --role "Monitoring Reader" --scopes /subscriptions/{subscription-id}`\n\n2. **Configure workflow**: Update 'Set Configuration' with your Application Insights Application ID and Azure AD tenant ID.\n\n3. **Add credentials**: The workflow uses Azure credentials which must be configured in your n8n instance. |
| Sticky Note - Config | n8n-nodes-base.stickyNote | Documentation / visual guidance |  |  | ## Configuration\n\nSet your Application Insights Application ID and time range filter. Ensure OAuth2 credentials are configured in n8n. Time range options: 24h, 7d, 30d. |
| Sticky Note - Processing | n8n-nodes-base.stickyNote | Documentation / visual guidance |  |  | ## Processing & Reporting\n\nCorrelates data using operationId, calculates statistics, identifies top errors and slow requests, then generates Markdown and HTML reports. |
| Sticky Note - Data Collection | n8n-nodes-base.stickyNote | Documentation / visual guidance |  |  | ## Data Collection\n\nMakes three Application Insights API calls in one node: APIM requests, Service Bus traces, and exceptions. Handles OAuth2 authentication internally. |
| Sticky Note - Outputs | n8n-nodes-base.stickyNote | Documentation / visual guidance |  |  | ## Output Options\n\nExport to Excel or return JSON via webhook. Both nodes are disabled by default—enable as needed. |
| Manual Trigger | n8n-nodes-base.manualTrigger | Manual entry point |  | Set Configuration |  |
| Set Configuration | n8n-nodes-base.set | Define App Insights + AAD parameters | Manual Trigger | Query Application Insights | ## Configuration\n\nSet your Application Insights Application ID and time range filter. Ensure OAuth2 credentials are configured in n8n. Time range options: 24h, 7d, 30d. |
| Query Application Insights | n8n-nodes-base.code | OAuth token + 3 KQL queries to App Insights | Set Configuration | Correlate and Analyze Data | ## Data Collection\n\nMakes three Application Insights API calls in one node: APIM requests, Service Bus traces, and exceptions. Handles OAuth2 authentication internally. |
| Correlate and Analyze Data | n8n-nodes-base.code | Correlate requests/traces/exceptions by operationId; compute stats | Query Application Insights | Generate Report | ## Processing & Reporting\n\nCorrelates data using operationId, calculates statistics, identifies top errors and slow requests, then generates Markdown and HTML reports. |
| Generate Report | n8n-nodes-base.code | Build Markdown + HTML report | Correlate and Analyze Data | Check If Data Exists | ## Processing & Reporting\n\nCorrelates data using operationId, calculates statistics, identifies top errors and slow requests, then generates Markdown and HTML reports. |
| Check If Data Exists | n8n-nodes-base.if | Branch on correlatedData length | Generate Report | Output Report; No Data Found |  |
| Output Report | n8n-nodes-base.set | Prepare final structured output payload | Check If Data Exists (true) | Export to Excel; Respond to Webhook | ## Output Options\n\nExport to Excel or return JSON via webhook. Both nodes are disabled by default—enable as needed. |
| No Data Found | n8n-nodes-base.set | Output “no data” message | Check If Data Exists (false) |  |  |
| Export to Excel | n8n-nodes-base.spreadsheetFile | Optional XLSX export (disabled) | Output Report |  | ## Output Options\n\nExport to Excel or return JSON via webhook. Both nodes are disabled by default—enable as needed. |
| Respond to Webhook | n8n-nodes-base.respondToWebhook | Optional HTTP response (disabled; requires webhook trigger) | Output Report |  | ## Output Options\n\nExport to Excel or return JSON via webhook. Both nodes are disabled by default—enable as needed. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**  
   - Name: *Track API failures with Application Insights correlation* (or your preferred name).

2. **Add node: Manual Trigger**
   - Node type: **Manual Trigger**
   - No configuration needed.

3. **Add node: Set Configuration**
   - Node type: **Set**
   - Add fields (assignments):
     - `appInsightsAppId` (String) = your App Insights **Application ID**
     - `tenantId` (String) = Azure AD tenant ID
     - `clientId` (String) = service principal client ID
     - `clientSecret` (String) = service principal secret
     - `timeRange` (String) = `24h` (or `7d`, `30d`)
     - `includeSuccessful` (Boolean) = `false`
   - Connect: **Manual Trigger → Set Configuration**

4. **(Azure setup) Create/ensure service principal permissions**
   - Create SP (example from note):  
     `az ad sp create-for-rbac --name "n8n-appinsights" --role "Monitoring Reader" --scopes /subscriptions/{subscription-id}`
   - Ensure this principal can query the target Application Insights resource (at minimum, Reader/Monitoring Reader typically suffices; some environments may require explicit access to the resource).

5. **Add node: Query Application Insights (Code)**
   - Node type: **Code**
   - Paste logic that:
     - Reads `appInsightsAppId`, `tenantId`, `clientId`, `clientSecret`, `timeRange`, `includeSuccessful` from input JSON
     - Requests token from AAD v2 token endpoint using `client_credentials` and scope `https://api.applicationinsights.io/.default`
     - Calls `POST https://api.applicationinsights.io/v1/apps/{appId}/query` three times with KQL for:
       - `requests` (APIM requests; optionally failures only)
       - `traces` (Service Bus-related)
       - `exceptions`
     - Returns a single item containing `apimData`, `sbData`, `exceptionsData` set to the first table or `{}`.
   - Connect: **Set Configuration → Query Application Insights**

6. **Add node: Correlate and Analyze Data (Code)**
   - Node type: **Code**
   - Implement logic to:
     - Convert App Insights table format to object arrays
     - Group traces/exceptions by `operationId`
     - Enrich each request with matching traces/exceptions
     - Compute `summary`, `topErrors`, `topSlowRequests`, include `correlatedData`
     - Reference `timeRange` from the configuration node if desired.
   - Connect: **Query Application Insights → Correlate and Analyze Data**

7. **Add node: Generate Report (Code)**
   - Node type: **Code**
   - Build:
     - `markdownReport` string
     - `htmlReport` string
   - Return original data plus the two reports.
   - Connect: **Correlate and Analyze Data → Generate Report**

8. **Add node: Check If Data Exists (IF)**
   - Node type: **IF**
   - Condition:
     - Left value: `={{ $json.correlatedData.length }}`
     - Operator: **greater than**
     - Right value: `0`
   - Connect: **Generate Report → Check If Data Exists**

9. **Add node: Output Report (Set)**
   - Node type: **Set**
   - Add fields:
     - `report` (String) = `={{ $json.markdownReport }}`
     - `htmlReport` (String) = `={{ $json.htmlReport }}`
     - `summary` (Object) = `={{ $json.summary }}`
     - `topErrors` (Array) = `={{ $json.topErrors }}`
     - `failedRequests` (Array) = `={{ $json.correlatedData.filter(r => r.isFailure) }}`
   - Connect IF **true** output → **Output Report**

10. **Add node: No Data Found (Set)**
    - Node type: **Set**
    - Add field:
      - `message` (String) = “No data found for the specified time range and filters. Check your App Insights configuration.”
    - Connect IF **false** output → **No Data Found**

11. **(Optional) Add node: Export to Excel** (disabled by default)
    - Node type: **Spreadsheet File**
    - Operation: **To File**
    - File format: `xlsx`
    - File name: `azure-api-failures-{{ $now.format('yyyy-MM-dd-HHmmss') }}.xlsx`
    - Connect: **Output Report → Export to Excel**
    - Note: you may want to add a “flatten” step first to export a table of `failedRequests`.

12. **(Optional) Add node: Respond to Webhook** (disabled by default)
    - Node type: **Respond to Webhook**
    - Respond with: **JSON**
    - Response body (expression) building:
      - status, summary, topErrors, failedRequests, report, timestamp
    - Connect: **Output Report → Respond to Webhook**
    - Important: Add a **Webhook Trigger** node at the start if you want this to be callable via HTTP.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Create service principal: `az ad sp create-for-rbac --name "n8n-appinsights" --role "Monitoring Reader" --scopes /subscriptions/{subscription-id}` | Sticky note “How it works” |
| Time range options: `24h`, `7d`, `30d` | Sticky note “Configuration” |
| Output nodes “Export to Excel” and “Respond to Webhook” are disabled by default | Sticky note “Output Options” |
| Correlation key is `operationId` derived from `operation_Id` across requests/traces/exceptions | Workflow design (Data Processing) |

