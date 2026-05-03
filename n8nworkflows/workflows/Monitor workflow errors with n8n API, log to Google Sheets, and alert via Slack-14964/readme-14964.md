Monitor workflow errors with n8n API, log to Google Sheets, and alert via Slack

https://n8nworkflows.xyz/workflows/monitor-workflow-errors-with-n8n-api--log-to-google-sheets--and-alert-via-slack-14964


# Monitor workflow errors with n8n API, log to Google Sheets, and alert via Slack

Now I have a complete understanding of the workflow. Let me write the comprehensive documentation.### 1. Workflow Overview

**N8n Error Monitor Auto Logger & Alerter** is a monitoring workflow designed for n8n administrators, automation agencies, and operations teams running multiple production workflows. It continuously polls an n8n instance's internal API for failed executions, classifies each error into one of seven categories with an assigned severity level, appends a structured log row to a Google Sheets spreadsheet, and dispatches a color-coded Slack alert containing the error details, a suggested fix, and a direct link to the failed execution.

The workflow contains two entry points — a recurring schedule trigger (every 1 minute by default) and an on-demand webhook — and proceeds through the following logical blocks:

| Block | Name | Purpose |
|-------|------|---------|
| 1 | **Triggers** | Two entry points: scheduled polling and manual webhook invocation |
| 2 | **Fetch & Filter** | Retrieve the last 50 error executions from the n8n API, then narrow to only those within the last 5 minutes, excluding the monitor's own executions |
| 3 | **Detail Retrieval** | Fetch the full execution payload (including run data and error objects) for each filtered failure |
| 4 | **Classification** | Parse the execution detail, extract the failed node and error message, classify into 7 error categories with severity, generate a formatted Slack message and a Google Sheets row payload |
| 5 | **Output** | Append the structured row to Google Sheets and send the formatted Slack alert in parallel |

---

### 2. Block-by-Block Analysis

---

#### Block 1 — Triggers

**Overview:** This block provides two independent entry points into the workflow. The Schedule trigger runs the workflow automatically every minute for continuous monitoring. The Webhook trigger allows on-demand manual execution, useful for ad-hoc checks or integration with external systems.

**Nodes Involved:** Schedule, Webhook (Manual)

| Attribute | Schedule | Webhook (Manual) |
|-----------|----------|-------------------|
| **Type** | `n8n-nodes-base.scheduleTrigger` (v1.2) | `n8n-nodes-base.webhook` (v2) |
| **Technical Role** | Time-based automatic trigger | HTTP-based manual trigger |
| **Configuration** | Interval: every 1 minute (field = "minutes", no count specified so default = 1) | Path: `error-monitor-trigger-k7m2`; Response mode: "lastNode" (responds to caller with output of the last executed node); HTTP method: default (GET for test, POST for production) |
| **Key Expressions / Variables** | None | Webhook ID: `error-monitor`; Path used in the generated webhook URL |
| **Input Connections** | None (entry point) | None (entry point) |
| **Output Connections** | → Get Failures | → Get Failures |
| **Version-specific Requirements** | v1.2 is the current schedule trigger; no special setup | v2 webhook requires n8n ≥ 0.171.0 for `responseMode: lastNode` |
| **Edge Cases / Failure Types** | If the n8n instance is under heavy load, scheduled executions may queue or be skipped. Timezone of the n8n server affects interval alignment. | If the webhook path is not unique across the instance, a conflict will prevent activation. If no authentication is configured on the webhook, anyone with the URL can trigger the workflow. |

---

#### Block 2 — Fetch & Filter

**Overview:** This block calls the n8n internal API to retrieve the most recent failed executions (up to 50), then uses a Code node to filter that list down to only failures that occurred within the last 5 minutes and were not produced by this monitoring workflow itself, preventing alert loops.

**Nodes Involved:** Get Failures, Filter Recent

**Node: Get Failures**

| Attribute | Detail |
|-----------|--------|
| **Type** | `n8n-nodes-base.httpRequest` (v4.2) |
| **Technical Role** | HTTP GET request to the n8n REST API to list failed executions |
| **Configuration** | URL: `https://YOUR-N8N-INSTANCE-URL/api/v1/executions?status=error&limit=50`; Authentication: Generic Credential Type → HTTP Header Auth; Timeout: 30 000 ms |
| **Key Expressions / Variables** | The URL contains a placeholder `YOUR-N8N-INSTANCE-URL` that must be replaced with the actual n8n domain (e.g., `n8n.mycompany.com`). The query parameters `status=error` and `limit=50` are static. |
| **Input Connections** | ← Schedule OR ← Webhook (Manual) |
| **Output Connections** | → Filter Recent |
| **Version-specific Requirements** | v4.2 HTTP Request node requires n8n ≥ 1.0. HTTP Header Auth must have the header name set to `X-N8N-API-KEY` and the value set to a valid n8n API key. |
| **Edge Cases / Failure Types** | ① **401 Unauthorized**: API key missing, invalid, or not an admin key. ② **Network error / timeout**: n8n instance unreachable or slow (30 s timeout). ③ **Empty response**: No failed executions in the instance — the downstream Code node must handle this gracefully (it does: returns `[]`). ④ **Rate limiting**: n8n API may rate-limit very frequent calls; the 1-minute schedule is generally safe. |

**Node: Filter Recent**

| Attribute | Detail |
|-----------|--------|
| **Type** | `n8n-nodes-base.code` (v2) |
| **Technical Role** | JavaScript filter that narrows the API response to only recent and non-self failures |
| **Configuration** | Single JS code block (no external packages). Runs in the n8n sandbox. |
| **Key Expressions / Variables** | `$input.first().json` — the full API response body. `$workflow.id` — the ID of this monitoring workflow, used to exclude self-executions. `fiveMinAgo` — computed as `now - 5 minutes`. Outputs items with `executionId`, `workflowId`, `startedAt`, `status`. |
| **Input Connections** | ← Get Failures |
| **Output Connections** | → Get Execution Detail |
| **Version-specific Requirements** | Code node v2 supports `$input.first()`, `$workflow.id`. Requires n8n ≥ 0.166.0 for Code node v2. |
| **Edge Cases / Failure Types** | ① If the API response structure changes (e.g., `data` is not a top-level key), the filter returns `[]` silently — no error is thrown. ② If `startedAt` is null or malformed, `new Date(0)` is used, so the item will be filtered out. ③ If all items are self-executions, returns `[]` and downstream nodes will not execute (the workflow ends gracefully for that run). ④ Time skew between the n8n server clock and the execution timestamps could cause false negatives/positives — the 5-minute window mitigates this. |

**Filter Recent — Logic Summary:**

```
1. Parse the API response to get the executions array (data.data || []).
2. Compute fiveMinAgo = now - 5 minutes.
3. Filter: startedAt >= fiveMinAgo AND workflowId !== thisWorkflowId.
4. If no items remain, return [] (stops execution path).
5. Otherwise, return one item per qualifying execution with fields:
   executionId, workflowId, startedAt, status.
```

---

#### Block 3 — Detail Retrieval

**Overview:** For each filtered failure, this block makes an individual API call to fetch the complete execution record, including all node run data and error objects. This is necessary because the list endpoint only returns summary information.

**Nodes Involved:** Get Execution Detail

**Node: Get Execution Detail**

| Attribute | Detail |
|-----------|--------|
| **Type** | `n8n-nodes-base.httpRequest` (v4.2) |
| **Technical Role** | HTTP GET request to the n8n REST API to fetch a single execution's full data |
| **Configuration** | URL (expression): `https://YOUR-N8N-INSTANCE-URL/api/v1/executions/{{ $json.executionId }}?includeData=true`; Authentication: Generic Credential Type → HTTP Header Auth (same as Get Failures); Timeout: 30 000 ms |
| **Key Expressions / Variables** | `{{ $json.executionId }}` — injected from each item produced by Filter Recent. `includeData=true` ensures the response contains `workflowData` and `data.resultData.runData`. |
| **Input Connections** | ← Filter Recent |
| **Output Connections** | → Classify Error |
| **Version-specific Requirements** | Same as Get Failures: HTTP Header Auth with `X-N8N-API-KEY`. |
| **Edge Cases / Failure Types** | ① **Execution not found (404)**: The execution may have been purged between the list and detail calls. ② **Timeout**: Large executions with many nodes may take longer to serialize; 30 s timeout may be insufficient. ③ **Rate limiting**: If many failures occur simultaneously, this node makes one request per failure, potentially creating a burst of API calls. ④ **Permission**: The API key must have admin-level access to read execution data. |

---

#### Block 4 — Classification

**Overview:** This is the core intelligence block. It parses the full execution detail, locates the failed node and error message, classifies the error into one of seven categories (Auth Error, Rate Limit, Network, Data/Config Error, Not Found, Server Error, Permission, or Unclassified), assigns a severity level (Critical, High, Medium), generates a suggested fix, determines whether the error is retryable, formats a rich Slack message with color-coded emoji, and builds a JSON payload for Google Sheets.

**Nodes Involved:** Classify Error

**Node: Classify Error**

| Attribute | Detail |
|-----------|--------|
| **Type** | `n8n-nodes-base.code` (v2) |
| **Technical Role** | Error classification, enrichment, and output formatting engine |
| **Configuration** | Single JS code block. No external packages. |
| **Key Expressions / Variables** | `$input.first().json` — the full execution detail from the API. Output fields: `_slack` (formatted Slack message), `_body` (JSON string for Google Sheets append), `_retry` (boolean), `_exId` (execution ID). |
| **Input Connections** | ← Get Execution Detail |
| **Output Connections** | → Log to Sheet, → Slack Alert (fan-out) |
| **Version-specific Requirements** | Code node v2. Requires n8n ≥ 0.166.0. |
| **Edge Cases / Failure Types** | ① If the execution detail has an unexpected structure (e.g., `data.resultData` is null), the code falls back to `Unknown` for node and `Unknown error` for message. ② Very long error messages are truncated to 300 chars for Slack and 500 chars for Sheets. ③ The classification uses simple substring matching on the lowercased error message — sophisticated or non-standard error messages may fall into "Unclassified". ④ If both `topErr` and `runData` errors are present, the code prioritizes `topErr` for message/node but also scans `runData` as a fallback. |

**Classification Logic — 7 Categories:**

| Category | Trigger Keywords (in lowercased error message) | Severity | Default Fix | Retryable |
|----------|-----------------------------------------------|----------|-------------|-----------|
| Auth Error | `401`, `unauthorized`, `credential` | Critical | Re-authenticate: {failedNode} | No |
| Rate Limit | `429`, `rate limit` | Medium | Wait and retry | Yes |
| Network | `timeout`, `econn`, `network` | Medium | Check service | Yes |
| Data/Config Error | `validation`, `required`, `undefined` | High | Check config: {failedNode} | No |
| Not Found | `404`, `not found` | High | Check resource: {failedNode} | No |
| Server Error | `500`, `internal server` | Medium | Service error | Yes |
| Permission | `permission`, `access denied` | Critical | Check scopes: {failedNode} | No |
| Unclassified | (none of the above) | Medium | Check execution log | No |

**Slack Message Format:**

```
{emoji} *n8n Error*
*Workflow:* {workflowName}
*Node:* `{failedNode}`
*Type:* {category} ({severity})
```{truncatedErrorMessage}```
*Fix:* {suggestedFix}
```

- Emoji mapping: Critical → 🔴 (`:red_circle:`), High → 🟠 (`:large_orange_circle:`), Medium → 🟡 (`:large_yellow_circle:`)

**Google Sheets Row Format (columns A–L):**

| Column | Content |
|--------|---------|
| A | ISO timestamp |
| B | Workflow name |
| C | Workflow ID |
| D | Execution ID |
| E | Failed node name |
| F | Error category |
| G | Severity |
| H | Error message (truncated to 500 chars) |
| I | Suggested fix |
| J | Retryable? ("Yes" / "No") |
| K | Status ("Pending" if retryable, "Manual Fix" otherwise) |
| L | Empty (reserved, possibly for a direct link or follow-up note) |

---

#### Block 5 — Output

**Overview:** This block performs two actions in parallel after classification: it appends a row to a Google Sheets spreadsheet for persistent logging, and it sends a formatted Slack message to an alerts channel for real-time notification. Both actions happen regardless of each other's success.

**Nodes Involved:** Log to Sheet, Slack Alert

**Node: Log to Sheet**

| Attribute | Detail |
|-----------|--------|
| **Type** | `n8n-nodes-base.httpRequest` (v4.2) |
| **Technical Role** | Appends a row to a Google Sheets spreadsheet via the Google Sheets API v4 |
| **Configuration** | Method: POST; URL: `https://sheets.googleapis.com/v4/spreadsheets/YOUR_SPREADSHEET_ID/values/Error%20Log!A:L:append?valueInputOption=USER_ENTERED&insertDataOption=INSERT_ROWS`; Body: JSON expression `={{ $json._body }}`; Authentication: Predefined Credential Type → `googleSheetsOAuth2Api`; Send Body: true; Specify Body: json |
| **Key Expressions / Variables** | `{{ $json._body }}` — the pre-built JSON string from Classify Error containing the `values` array. `YOUR_SPREADSHEET_ID` must be replaced. Sheet name: `Error Log`. Range: `A:L`. |
| **Input Connections** | ← Classify Error |
| **Output Connections** | None (terminal node) |
| **Version-specific Requirements** | Google Sheets OAuth2 credential must be configured in n8n with scope `https://www.googleapis.com/auth/spreadsheets`. The spreadsheet must contain a sheet named "Error Log" with at least 12 columns (A–L). |
| **Edge Cases / Failure Types** | ① **403 Forbidden**: OAuth2 credential lacks write permission on the spreadsheet. ② **404 Not Found**: Spreadsheet ID or sheet name is incorrect. ③ **400 Bad Request**: Malformed `_body` JSON (e.g., if Classify Error produced invalid JSON). ④ **Rate limiting**: Google Sheets API has a limit of ~300 requests per minute per project. ⑤ **Column mismatch**: If the sheet has fewer than 12 columns, the append will fail. |

**Node: Slack Alert**

| Attribute | Detail |
|-----------|--------|
| **Type** | `n8n-nodes-base.slack` (v2.2) |
| **Technical Role** | Sends a formatted message to a Slack channel |
| **Configuration** | Text (expression): `={{ $('Classify Error').item.json._slack }}`; Channel selection: by list, channel ID: `YOUR_CHANNEL_ID` (cached result name: "your-alerts-channel"); Authentication: OAuth2 |
| **Key Expressions / Variables** | `$('Classify Error').item.json._slack` — references the _slack field from the Classify Error output. `YOUR_CHANNEL_ID` must be replaced with the actual Slack channel ID (e.g., `C01ABCDEF`). |
| **Input Connections** | ← Classify Error |
| **Output Connections** | None (terminal node) |
| **Version-specific Requirements** | Slack OAuth2 credential must be configured in n8n with the `chat:write` and `channels:read` bot token scopes. The Slack app must be a member of the target channel. |
| **Edge Cases / Failure Types** | ① **channel_not_found**: The bot is not in the channel or the channel ID is wrong. ② **not_in_channel**: The bot hasn't been invited to the channel. ③ **rate_limited**: Slack limits posting to ~1 message per second per channel. ④ **token revoked**: OAuth2 token expired or app was uninstalled. ⑤ **Formatting**: The `_slack` field uses Slack markdown (bold, code, code block); if the error message contains unescaped backticks or special characters, the formatting may break. |

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|-----------|-----------|----------------|---------------|----------------|-------------|
| Schedule | scheduleTrigger (v1.2) | Time-based automatic trigger running every 1 minute | — | Get Failures | ⏱️ Triggers — Runs on a schedule (every minute) for continuous monitoring, or manually via webhook for on-demand checks. |
| Webhook (Manual) | webhook (v2) | HTTP-based manual trigger for on-demand execution | — | Get Failures | ⏱️ Triggers — Runs on a schedule (every minute) for continuous monitoring, or manually via webhook for on-demand checks. |
| Get Failures | httpRequest (v4.2) | Fetches up to 50 failed executions from the n8n API | Schedule, Webhook (Manual) | Filter Recent | 🔍 Fetch & Filter — Calls the n8n API for failed executions, then filters to only recent failures (last 5 minutes). Excludes this workflow's own executions to prevent alert loops. |
| Filter Recent | code (v2) | Filters API results to failures within the last 5 minutes, excluding self-executions | Get Failures | Get Execution Detail | 🔍 Fetch & Filter — Calls the n8n API for failed executions, then filters to only recent failures (last 5 minutes). Excludes this workflow's own executions to prevent alert loops. |
| Get Execution Detail | httpRequest (v4.2) | Fetches full execution data (including run data and errors) for each filtered failure | Filter Recent | Classify Error | 🏷️ Classify, Log & Alert — Fetches full execution details, classifies the error into 7 categories with severity levels, logs to Google Sheets, and sends a color-coded Slack alert with suggested fixes and direct links. |
| Classify Error | code (v2) | Parses execution detail, classifies error into 7 categories, assigns severity, formats Slack message and Sheets row | Get Execution Detail | Log to Sheet, Slack Alert | 🏷️ Classify, Log & Alert — Fetches full execution details, classifies the error into 7 categories with severity levels, logs to Google Sheets, and sends a color-coded Slack alert with suggested fixes and direct links. |
| Log to Sheet | httpRequest (v4.2) | Appends a structured error row to a Google Sheets spreadsheet via the Sheets API v4 | Classify Error | — | 🏷️ Classify, Log & Alert — Fetches full execution details, classifies the error into 7 categories with severity levels, logs to Google Sheets, and sends a color-coded Slack alert with suggested fixes and direct links. |
| Slack Alert | slack (v2.2) | Sends a color-coded formatted alert message to a Slack channel | Classify Error | — | 🏷️ Classify, Log & Alert — Fetches full execution details, classifies the error into 7 categories with severity levels, logs to Google Sheets, and sends a color-coded Slack alert with suggested fixes and direct links. |
| Sticky Note | stickyNote (v1) | Documentation — main workflow description and setup instructions | — | — | # 🚨 N8n Error Monitor Auto Logger & Alerter — **Who it's for:** n8n admins, automation agencies, and ops teams running multiple production workflows. **What it does:** Monitors your entire n8n instance for failed executions, classifies each error into 7 categories with severity levels, logs everything to Google Sheets, and sends color-coded Slack alerts with suggested fixes and direct links. How it works: 1. Runs every minute via schedule (or on-demand via webhook) 2. Fetches failed executions from the n8n internal API 3. Filters to recent failures (last 5 min) and excludes its own executions 4. Fetches full execution details for each failure 5. Classifies error type (7 categories) + severity (Critical/High/Medium) 6. Logs to Google Sheets and sends color-coded Slack alert. Setup: 1. Create an n8n API key (Settings → API) → add as HTTP Header Auth with header `X-N8N-API-KEY` 2. Replace `YOUR-N8N-INSTANCE-URL` in the HTTP Request nodes with your n8n domain 3. Connect Google Sheets OAuth → set your spreadsheet ID in the Log to Sheet node 4. Connect Slack OAuth → set your alerts channel in the Slack Alert node 5. Adjust the schedule interval if needed (default: every 1 minute). Created by [Ahmad Bukhari](https://www.linkedin.com/in/bukhariahmad) \| [ahmadbukhari.com](https://ahmadbukhari.com) |
| Sticky Note1 | stickyNote (v1) | Documentation — triggers area description | — | — | ⏱️ Triggers — Runs on a schedule (every minute) for continuous monitoring, or manually via webhook for on-demand checks. |
| Sticky Note2 | stickyNote (v1) | Documentation — fetch & filter area description | — | — | 🔍 Fetch & Filter — Calls the n8n API for failed executions, then filters to only recent failures (last 5 minutes). Excludes this workflow's own executions to prevent alert loops. |
| Sticky Note3 | stickyNote (v1) | Documentation — classify, log & alert area description | — | — | 🏷️ Classify, Log & Alert — Fetches full execution details, classifies the error into 7 categories with severity levels, logs to Google Sheets, and sends a color-coded Slack alert with suggested fixes and direct links. |

---

### 4. Reproducing the Workflow from Scratch

Follow these steps to recreate the workflow in a new n8n instance:

1. **Create a new workflow** and name it `N8n Error Monitor Auto Logger & Alerter`.

2. **Add a Sticky Note** for documentation. Position it on the left canvas. Set content to the full setup and description text (refer to the Sticky Note content in section 3). Color: 4 (green).

3. **Add the Schedule trigger node:**
   - Type: `Schedule Trigger`
   - Version: 1.2
   - Configuration: Rule → Interval → Field: `minutes` (default count = 1, meaning every 1 minute)
   - Position it in the upper-left trigger area.

4. **Add the Webhook trigger node:**
   - Type: `Webhook`
   - Version: 2
   - Configuration: Path: `error-monitor-trigger-k7m2`; Response Mode: `Last Node`; HTTP Method: default (GET for testing, POST for production)
   - Position it below the Schedule node in the trigger area.

5. **Add a Sticky Note** covering both triggers. Color: 6 (orange). Content: `⏱️ Triggers — Runs on a schedule (every minute) for continuous monitoring, or manually via webhook for on-demand checks.`

6. **Create the HTTP Header Auth credential for the n8n API:**
   - Go to Credentials → New Credential → Header Auth
   - Header Name: `X-N8N-API-KEY`
   - Header Value: *(your n8n API key — generate one in n8n Settings → API)*

7. **Add the Get Failures node:**
   - Type: `HTTP Request`
   - Version: 4.2
   - Method: GET
   - URL: `https://YOUR-N8N-INSTANCE-URL/api/v1/executions?status=error&limit=50`
   - Authentication: Generic Credential Type → Header Auth → select the credential from step 6
   - Timeout: 30000 ms
   - Connect: Schedule → Get Failures, and Webhook (Manual) → Get Failures

8. **Add the Filter Recent node:**
   - Type: `Code`
   - Version: 2
   - Language: JavaScript
   - Code: Paste the Filter Recent code (see below)
   - Connect: Get Failures → Filter Recent

   **Filter Recent code:**
   ```javascript
   const data = $input.first().json;
   const executions = data.data || [];
   const now = new Date();
   const fiveMinAgo = new Date(now.getTime() - 5 * 60 * 1000);
   const thisWorkflowId = $workflow.id;
   const recent = executions.filter(ex => {
     const started = new Date(ex.startedAt || 0);
     return started >= fiveMinAgo && ex.workflowId !== thisWorkflowId;
   });
   if (recent.length === 0) return [];
   const entries = [];
   for (const ex of recent) {
     entries.push({ json: { executionId: ex.id, workflowId: ex.workflowId, startedAt: ex.startedAt, status: ex.status || 'error' } });
   }
   return entries;
   ```

9. **Add a Sticky Note** covering the Get Failures and Filter Recent nodes. Color: 3 (blue). Content: `🔍 Fetch & Filter — Calls the n8n API for failed executions, then filters to only recent failures (last 5 minutes). Excludes this workflow's own executions to prevent alert loops.`

10. **Add the Get Execution Detail node:**
    - Type: `HTTP Request`
    - Version: 4.2
    - Method: GET
    - URL (expression): `https://YOUR-N8N-INSTANCE-URL/api/v1/executions/{{ $json.executionId }}?includeData=true`
    - Authentication: Generic Credential Type → Header Auth → select the same credential from step 6
    - Timeout: 30000 ms
    - Connect: Filter Recent → Get Execution Detail

11. **Add the Classify Error node:**
    - Type: `Code`
    - Version: 2
    - Language: JavaScript
    - Code: Paste the Classify Error code (see below)
    - Connect: Get Execution Detail → Classify Error

    **Classify Error code:**
    ```javascript
    const ex = $input.first().json;
    const ts = new Date().toISOString();
    const wfData = ex.workflowData || {};
    const wfName = wfData.name || 'Unknown';
    const wfId = ex.workflowId || '';
    const exId = ex.id || '';
    const rd = (ex.data || {}).resultData || {};
    const topErr = rd.error || {};
    let errorMessage = topErr.message || '';
    let failedNode = topErr.node?.name || '';
    const runData = rd.runData || {};
    for (const [nodeName, runs] of Object.entries(runData)) {
      for (const run of runs) {
        if (run.error) {
          if (!failedNode) failedNode = nodeName;
          if (!errorMessage) errorMessage = run.error.message || '';
        }
      }
    }
    if (!errorMessage) errorMessage = 'Unknown error';
    if (!failedNode) failedNode = 'Unknown';
    const m = errorMessage.toLowerCase();
    let cat='Unclassified',sev='Medium',sol='Check execution log.',retry=false;
    if(m.includes('401')||m.includes('unauthorized')||m.includes('credential')){cat='Auth Error';sev='Critical';sol='Re-authenticate: '+failedNode;}
    else if(m.includes('429')||m.includes('rate limit')){cat='Rate Limit';sev='Medium';sol='Wait and retry.';retry=true;}
    else if(m.includes('timeout')||m.includes('econn')||m.includes('network')){cat='Network';sev='Medium';sol='Check service.';retry=true;}
    else if(m.includes('validation')||m.includes('required')||m.includes('undefined')){cat='Data/Config Error';sev='High';sol='Check config: '+failedNode;}
    else if(m.includes('404')||m.includes('not found')){cat='Not Found';sev='High';sol='Check resource: '+failedNode;}
    else if(m.includes('500')||m.includes('internal server')){cat='Server Error';sev='Medium';sol='Service error.';retry=true;}
    else if(m.includes('permission')||m.includes('access denied')){cat='Permission';sev='Critical';sol='Check scopes: '+failedNode;}
    const emoji=sev==='Critical'?':red_circle:':sev==='High'?':large_orange_circle:':':large_yellow_circle:';
    const slack=emoji+' *n8n Error*\n*Workflow:* '+wfName+'\n*Node:* `'+failedNode+'`\n*Type:* '+cat+' ('+sev+')\n```'+(errorMessage||'').substring(0,300)+'```\n*Fix:* '+sol;
    const sheetRow=JSON.stringify({values:[[ts,wfName,wfId,exId,failedNode,cat,sev,(errorMessage||'').substring(0,500),sol,retry?'Yes':'No',retry?'Pending':'Manual Fix','']]});
    return [{json:{_slack:slack,_body:sheetRow,_retry:retry,_exId:exId}}];
    ```

12. **Create the Google Sheets OAuth2 credential:**
    - Go to Credentials → New Credential → Google Sheets OAuth2 API
    - Follow the n8n OAuth2 setup flow (requires a Google Cloud project with the Google Sheets API enabled and OAuth consent screen configured)
    - Required scope: `https://www.googleapis.com/auth/spreadsheets`

13. **Prepare the Google Sheets spreadsheet:**
    - Create a new Google Spreadsheet (or use an existing one)
    - Ensure it has a sheet tab named exactly `Error Log`
    - Ensure columns A through L are present (headers recommended: Timestamp, Workflow Name, Workflow ID, Execution ID, Failed Node, Category, Severity, Error Message, Suggested Fix, Retryable, Status, Notes)
    - Copy the spreadsheet ID from the URL (the long string between `/d/` and `/edit`)

14. **Add the Log to Sheet node:**
    - Type: `HTTP Request`
    - Version: 4.2
    - Method: POST
    - URL: `https://sheets.googleapis.com/v4/spreadsheets/YOUR_SPREADSHEET_ID/values/Error%20Log!A:L:append?valueInputOption=USER_ENTERED&insertDataOption=INSERT_ROWS`
    - Replace `YOUR_SPREADSHEET_ID` with the actual spreadsheet ID from step 13
    - Send Body: true
    - Specify Body: JSON
    - Body (expression): `={{ $json._body }}`
    - Authentication: Predefined Credential Type → Google Sheets OAuth2 API → select the credential from step 12
    - Connect: Classify Error → Log to Sheet

15. **Create the Slack OAuth2 credential:**
    - Go to Credentials → New Credential → Slack OAuth2
    - In your Slack app configuration, add the `chat:write` and `channels:read` bot token scopes
    - Install the app to your workspace and complete the OAuth flow in n8n
    - Ensure the bot is a member of the target alerts channel

16. **Add the Slack Alert node:**
    - Type: `Slack`
    - Version: 2.2
    - Operation: Send Message
    - Channel selection: By list → select the target channel (replace `YOUR_CHANNEL_ID` with the actual channel ID, e.g., `C01ABCDEF`)
    - Text (expression): `={{ $('Classify Error').item.json._slack }}`
    - Authentication: OAuth2 → select the Slack credential from step 15
    - Connect: Classify Error → Slack Alert

17. **Add a Sticky Note** covering the Get Execution Detail, Classify Error, Log to Sheet, and Slack Alert nodes. Content: `🏷️ Classify, Log & Alert — Fetches full execution details, classifies the error into 7 categories with severity levels, logs to Google Sheets, and sends a color-coded Slack alert with suggested fixes and direct links.`

18. **Verify all connections:**
    - Schedule → Get Failures
    - Webhook (Manual) → Get Failures
    - Get Failures → Filter Recent
    - Filter Recent → Get Execution Detail
    - Get Execution Detail → Classify Error
    - Classify Error → Log to Sheet (first output)
    - Classify Error → Slack Alert (second output)

19. **Replace all placeholders:**
    - `YOUR-N8N-INSTANCE-URL` in Get Failures and Get Execution Detail with your n8n instance URL (e.g., `https://n8n.mycompany.com`)
    - `YOUR_SPREADSHEET_ID` in Log to Sheet with your Google Sheets spreadsheet ID
    - `YOUR_CHANNEL_ID` in Slack Alert with your Slack channel ID

20. **Activate the workflow** by toggling the Active switch. The Schedule trigger will begin executing every minute.

21. **Test manually** by clicking "Execute Workflow" on the Schedule trigger or by sending a request to the Webhook URL (available in the Webhook node after activation).

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|-------------|----------------|
| The n8n API key must have admin-level permissions to read execution data. Generate it under Settings → API in your n8n instance. | n8n API documentation |
| The 5-minute filter window in the Filter Recent node is hardcoded. To change it, modify the `5 * 60 * 1000` value in the code. A larger window may result in duplicate alerts across runs if the same execution is still within the window on successive polls. | Filter Recent node |
| Error classification is keyword-based and may not capture all error types. Errors that do not match any keyword pattern are classified as "Unclassified" with Medium severity. Extend the `if/else if` chain in the Classify Error code to add new categories. | Classify Error node |
| The Google Sheets append uses `INSERT_ROWS`, so new rows are always added at the bottom. The sheet does not auto-sort; consider adding a SORT formula in a separate view. | Log to Sheet node |
| The Slack message uses Slack's proprietary markdown format (bold with `*`, code with backticks, code blocks with triple backticks). Verify rendering in your Slack client. | Slack Alert node |
| This workflow excludes its own executions from alerts by comparing `workflowId` against `$workflow.id`. If you duplicate this workflow, the duplicate will also exclude its own executions but may alert on the original's failures and vice versa. | Filter Recent node |
| If the n8n instance has many concurrent failures (>50 within the polling interval), only the most recent 50 will be fetched. Increase the `limit` parameter in the Get Failures URL to raise this ceiling. | Get Failures node |
| Created by Ahmad Bukhari | [LinkedIn](https://www.linkedin.com/in/bukhariahmad) \| [ahmadbukhari.com](https://ahmadbukhari.com) |