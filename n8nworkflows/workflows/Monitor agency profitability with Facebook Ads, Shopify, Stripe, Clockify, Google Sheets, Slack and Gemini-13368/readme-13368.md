Monitor agency profitability with Facebook Ads, Shopify, Stripe, Clockify, Google Sheets, Slack and Gemini

https://n8nworkflows.xyz/workflows/monitor-agency-profitability-with-facebook-ads--shopify--stripe--clockify--google-sheets--slack-and-gemini-13368


# Monitor agency profitability with Facebook Ads, Shopify, Stripe, Clockify, Google Sheets, Slack and Gemini

## 1. Workflow Overview

**Purpose:** Monitor weekly agency/client profitability by pulling **ad spend** (Facebook Ads + Google Ads), **revenue** (Shopify + optional Stripe reference), and **labor cost** (Clockify). It then computes profitability KPIs (profit margin, ROAS, CAC, time-to-revenue ratio), stores a weekly snapshot in **Google Sheets**, and posts an executive summary plus AI insights to **Slack** using **Gemini**.

**Target use cases:**
- Agency leadership weekly review of client health (profitable vs at-risk).
- Operational visibility into whether labor/time costs are drifting vs revenue.
- Automated reporting with historical tracking in Sheets and an action-focused Slack message.

### 1.1 Scheduling & Orchestration
Runs on a weekly schedule and fans out to multiple data sources.

### 1.2 Data Ingestion (APIs + Mock Layer)
Calls Facebook/Google/Shopify/Stripe/Clockify. In the provided workflow, real API nodes are followed by **Set** nodes that output **mock/sample payloads** (intended to be removed when going live).

### 1.3 Data Consolidation & Profitability Computation
Merges all streams and calculates agency-level + client-level metrics, risk flags, and a Slack-ready summary line.

### 1.4 Storage (Google Sheets)
Appends one row per run to a sheet for historical tracking.

### 1.5 AI Analysis (Gemini Agent) & Notification (Slack)
Gemini agent receives structured metrics and produces Slack-formatted insights; Slack node posts the final message.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduling & Fan-out Execution
**Overview:** Triggers the workflow weekly and starts all data retrieval branches in parallel.  
**Nodes involved:** `Schedule Trigger1`

#### Node: Schedule Trigger1
- **Type / role:** Schedule Trigger (`n8n-nodes-base.scheduleTrigger`) — entry point.
- **Configuration (interpreted):** Cron schedule `0 8 * * 1` → every Monday at 08:00 (server timezone).
- **Outputs:** Connected to five branches: Facebook, Google Ads, Shopify, Clockify→Stripe mock chain, Stripe→Clockify mock chain.
- **Edge cases / failures:**
  - Timezone mismatch (n8n instance timezone vs business timezone).
  - Overlapping runs if execution time exceeds schedule frequency (unlikely weekly, but possible if APIs hang).

---

### Block 2 — Data Ingestion (Real APIs)
**Overview:** Pulls data from external services. Each API node has `onError: continueRegularOutput` (for several nodes), allowing downstream execution even if one source fails—this affects data quality.  
**Nodes involved:** `Facebook Graph API`, `Get a campaign`, `Get many orders`, `Get a time entry`, `Get a balance`

#### Node: Facebook Graph API
- **Type / role:** Facebook Graph API (`n8n-nodes-base.facebookGraphApi`) — fetch ad insights.
- **Configuration:**
  - **Edge:** `insights`
  - **Node ID:** expression `={{$json.account_id}}` (expects an `account_id` in incoming JSON)
  - **Query parameters:** `fields=spend`, `time_range={"since":"2025-02-01","until":"2025-02-10"}`, `level=account`
  - **Graph API version:** v23.0
- **Inputs/Outputs:**
  - **Input:** From schedule trigger. However, the schedule trigger does not naturally provide `account_id`, so in “go-live” you must provide it (e.g., via a Set node or by listing ad accounts).
  - **Output:** To `Facebook ads1` (mock Set).
- **Edge cases / failures:**
  - Missing `account_id` → expression resolves empty → API error.
  - Permissions/scopes (ads_read/ads_management) and account access issues.
  - Time range is hard-coded; will not roll weekly without modification.

#### Node: Get a campaign
- **Type / role:** Google Ads (`n8n-nodes-base.googleAds`) — fetch a campaign entity (not a spend report).
- **Configuration:**
  - Operation: `get`
  - Campaign ID: `12345678`
  - Client Customer ID: `3650538801`
  - Manager Customer ID: `12345678`
- **Output:** To `Google ads1` (mock Set).
- **Edge cases / failures:**
  - OAuth scope/approval and MCC permissions.
  - This node returns campaign metadata; it does **not** necessarily provide spend metrics in the shape expected by the code (which expects `results[].metrics.costMicros`, etc.). You will likely replace this with a “report/query” style Google Ads node (or adjust the code).

#### Node: Get many orders
- **Type / role:** Shopify (`n8n-nodes-base.shopify`) — fetch orders.
- **Configuration:** Operation `getAll` (no additional filters in the provided config).
- **Output:** To `Shopify1` (mock Set).
- **Edge cases / failures:**
  - Large order volumes → pagination/timeouts if not constrained by date.
  - API limits; need `created_at_min/max` or similar filters for weekly runs.
  - Returns may not include `total_refunds` in the exact field used later depending on API version; validate.

#### Node: Get a time entry
- **Type / role:** Clockify (`n8n-nodes-base.clockify`) — fetch a single time entry.
- **Configuration:** Resource `timeEntry`, operation `get` (but no ID shown in parameters → typically required).
- **Output:** To `Stripe1` (mock Set).
- **Edge cases / failures:**
  - Missing time entry ID (common misconfiguration).
  - Workspace scope/permissions.
  - Output shape likely differs from the mock used in the code.

#### Node: Get a balance
- **Type / role:** Stripe (`n8n-nodes-base.stripe`) — get balance.
- **Configuration:** Default “Get a balance” (no parameters shown).
- **Output:** To `Clockify1` (mock Set).
- **Edge cases / failures:**
  - Stripe auth errors, restricted keys, test vs live mode.
  - Returned balance is not used by the code as written (code doesn’t parse Stripe balance).

---

### Block 3 — Mock Data Layer (Set Nodes)
**Overview:** These Set nodes output sample payloads and currently feed the merge/calculation. Sticky notes explicitly instruct removing them and connecting real API nodes directly to the merge before production.  
**Nodes involved:** `Facebook ads1`, `Google ads1`, `Shopify1`, `Stripe1`, `Clockify1`

#### Node: Facebook ads1
- **Type / role:** Set (`n8n-nodes-base.set`) — mock Facebook insights payload.
- **Configuration:** “Raw” JSON output containing `data[]` with `spend`, `purchase_roas`, `conversions`, etc.
- **Output:** To `Merge1` input index 0.
- **Edge cases:** In production, remove this node and ensure real Facebook response matches the code’s expected fields (`data[]`, `spend`, `account_name`, etc.).

#### Node: Google ads1
- **Type / role:** Set — mock Google Ads payload.
- **Configuration:** “Raw” JSON output with `results[]` containing `metrics.costMicros`, clicks, impressions, etc.
- **Output:** To `Merge1` input index 1.
- **Edge cases:** Real Google Ads nodes may not output this exact structure unless you use a reporting/query node. Adjust either the node or the code.

#### Node: Shopify1
- **Type / role:** Set — mock Shopify orders payload.
- **Configuration:** “Raw” JSON output with `orders[]` including `total_price` and `total_refunds`.
- **Output:** To `Merge1` input index 2.
- **Edge cases:** Real Shopify node output may be `[]` of orders directly rather than wrapped in `{ orders: [...] }` depending on node settings/version. Align before go-live.

#### Node: Stripe1
- **Type / role:** Set — mock Stripe payload.
- **Configuration:** Sample “revenue recognition transaction” object.
- **Output:** To `Merge1` input index 3.
- **Edge cases:** The code does not parse this payload currently; it will be effectively ignored unless extended.

#### Node: Clockify1
- **Type / role:** Set — mock Clockify time entries payload.
- **Configuration:** Outputs `json: [ ...time entries... ]` (note: it uses a property literally named `json` containing an array).
- **Output:** To `Merge1` input index 4.
- **Edge cases:** Real Clockify node output format must match what the code checks: `if (json.json && Array.isArray(json.json))`.

---

### Block 4 — Consolidation & Profitability Computation
**Overview:** Merges up to five inputs and computes agency + client profitability, cost allocations, risk flags, and validation warnings/errors.  
**Nodes involved:** `Merge1`, `Code in JavaScript`

#### Node: Merge1
- **Type / role:** Merge (`n8n-nodes-base.merge`) — multi-input aggregator.
- **Configuration:** `numberInputs: 5` (expects up to five incoming streams).
- **Inputs:** From the five mock Set nodes (Facebook ads1, Google ads1, Shopify1, Stripe1, Clockify1).
- **Output:** To `Code in JavaScript`.
- **Edge cases / failures:**
  - If you remove mock nodes and wire real APIs, ensure you still supply all required inputs or adjust `numberInputs`.
  - Data arriving as multiple items vs one item per branch can change `$input.all()` results.

#### Node: Code in JavaScript
- **Type / role:** Code (`n8n-nodes-base.code`) — core financial model and formatting.
- **Configuration (interpreted):**
  - Defines `CONFIG`:
    - `platformFeeRate = 0.029`
    - `monthlySoftwareCosts = 500`
    - `overheadRate = 0.30` (applied on time cost)
    - `minProfitMargin = 20` (%)
    - `maxTimeRatio = 40` (% of revenue)
  - Reads all merged items: `const allInputs = $input.all();`
  - Detects data sources by checking presence of:
    - Facebook: `json.data[]`
    - Google: `json.results[]`
    - Shopify: `json.orders[]`
    - Time tracking: `json.json[]` (Clockify mock shape)
  - Calculates:
    - Total ad spend (Facebook + Google)
    - Revenue (from Facebook ROAS * spend, then possibly replaced by Shopify if higher, plus time-entry “revenueAttributed”)
    - Labor/time cost (sum of `costAmount`)
    - Platform fees, overhead allocation, total costs
    - KPIs: CAC, ROAS, gross profit, profit margin, time ratio
  - Builds per-client breakdown and assigns `riskLevel` using thresholds.
  - Returns:
    - `summary_text` (Slack header line)
    - `clientDetails` array
    - `summary`, `breakdown`, `dataValidation`
- **Outputs:** To `Append row in sheet1` (then AI + Slack).
- **Key variables/fields produced:**
  - `summary_text`
  - `clientDetails[]` (profitMargin, timeRatio, roas, costs, riskLevel, riskReasons)
  - `dataValidation.errors`, `dataValidation.warnings`, `dataValidation.sourcesFound`
- **Edge cases / failure types:**
  - **Schema mismatches** (most likely): real API responses won’t match the mock keys (`orders`, `results`, `json.json`).
  - **Double counting revenue:** the code adds Facebook-derived revenue + Shopify revenue adjustment + time-entry `revenueAttributed`. Ensure you intend to add `revenueAttributed` on top of Shopify revenue.
  - **Google Ads revenue/conversions missing:** code warns if `conversionValue` missing; the mock data has none, so warnings will appear.
  - **Client naming consistency:** Facebook uses `account_name`, time tracking uses `clientName`, Google uses hardcoded `Google Ads Client`. This can fragment clients across sources.

---

### Block 5 — Storage (Google Sheets)
**Overview:** Writes one appended row per run for trend/history.  
**Nodes involved:** `Append row in sheet1`

#### Node: Append row in sheet1
- **Type / role:** Google Sheets (`n8n-nodes-base.googleSheets`) — append weekly metrics.
- **Configuration:**
  - Operation: `append`
  - Document: “Autonomous Client Profit Guardian”
  - Sheet: “Sheet1” (gid=0)
  - Column schema includes: `timestamp`, `date`, `totalRevenue`, `totalAdSpend`, `facebookAdSpend`, `googleAdSpend`, `totalTimeCost`, `platformFees`, `softwareCosts`, `overheadAllocation`, `totalCosts`, `grossProfit`, `profitMargin`, `cac`, `roas`, `conversions`, `billableHours`, `isProfitable`, `meetsMinROAS`, etc.
  - Mapping mode: auto-map input data
- **Input:** From `Code in JavaScript`.
- **Output:** To `AI Profit Analyzer1`.
- **Edge cases / failures:**
  - Auto-mapping may fail if your incoming JSON doesn’t have keys matching column IDs.
  - Some columns (`isProfitable`, `meetsMinROAS`, `timestamp`, etc.) are not produced by the code currently; they may remain blank unless added.
  - Permissions: Sheets OAuth must have access to the document.

---

### Block 6 — AI Analysis (Gemini Agent) & Slack Notification
**Overview:** Sends computed metrics to a LangChain agent backed by Gemini; posts a combined summary to Slack.  
**Nodes involved:** `Google Gemini Chat Model1`, `AI Profit Analyzer1`, `Send a message`

#### Node: Google Gemini Chat Model1
- **Type / role:** Gemini chat model (`@n8n/n8n-nodes-langchain.lmChatGoogleGemini`) — LLM provider.
- **Configuration:** Default options.
- **Connection:** Feeds the agent via the `ai_languageModel` connection.
- **Edge cases / failures:**
  - Invalid/expired API key, quota/rate limits.
  - Region/model availability changes.

#### Node: AI Profit Analyzer1
- **Type / role:** LangChain Agent (`@n8n/n8n-nodes-langchain.agent`) — generates executive insights.
- **Configuration:**
  - Prompt includes:
    - `clientDetails` (stringified JSON from Code node)
    - `summary` + `breakdown` + `dataValidation`
  - Strict formatting requirements for Slack readability (headers, bullet symbol •, emojis, one-line bullets).
  - System message defines rules: use `riskLevel`, focus on `profitMargin` and `timeRatio`, include $ and %.
- **Input:** From `Append row in sheet1` (so agent runs after storage).
- **Output:** To `Send a message`.
- **Edge cases:**
  - Large payload may exceed model context limits if many clients.
  - If `Code in JavaScript` changes output keys, prompt expressions can break.

#### Node: Send a message
- **Type / role:** Slack (`n8n-nodes-base.slack`) — sends report to a channel.
- **Configuration:**
  - Channel: `estateline-ai` (by ID)
  - Message combines:
    - Current date (JS `new Date().toLocaleDateString(...)`)
    - `summary_text` from Code node
    - Agent output from `AI Profit Analyzer1`
  - `includeLinkToWorkflow: false`
- **Edge cases / failures:**
  - Bot not invited to channel.
  - Slack token scopes missing (`chat:write`, channel access).
  - If the agent fails, the Slack message expression referencing its output may be empty.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger1 | Schedule Trigger | Weekly trigger / entry point | — | Facebook Graph API; Get a campaign; Get many orders; Get a time entry; Get a balance | # Agency profitability monitor (description + setup steps) |
| Facebook Graph API | Facebook Graph API | Pull Facebook Ads insights (spend) | Schedule Trigger1 | Facebook ads1 | ## Data ingestion … Disable mock Set nodes and connect real API nodes directly to the Merge node before going live. |
| Get a campaign | Google Ads | Fetch Google Ads campaign (currently not a spend report) | Schedule Trigger1 | Google ads1 | ## Data ingestion … Disable mock Set nodes and connect real API nodes directly to the Merge node before going live. |
| Get many orders | Shopify | Fetch Shopify orders | Schedule Trigger1 | Shopify1 | ## Data ingestion … Disable mock Set nodes and connect real API nodes directly to the Merge node before going live. |
| Get a time entry | Clockify | Fetch a time entry (labor) | Schedule Trigger1 | Stripe1 | ## Data ingestion … Disable mock Set nodes and connect real API nodes directly to the Merge node before going live. |
| Get a balance | Stripe | Fetch Stripe balance (not used by calculator) | Schedule Trigger1 | Clockify1 | ## Data ingestion … Disable mock Set nodes and connect real API nodes directly to the Merge node before going live. |
| Facebook ads1 | Set | Mock Facebook insights payload | Facebook Graph API | Merge1 | ## Data ingestion … Disable mock Set nodes and connect real API nodes directly to the Merge node before going live. |
| Google ads1 | Set | Mock Google Ads metrics payload | Get a campaign | Merge1 | ## Data ingestion … Disable mock Set nodes and connect real API nodes directly to the Merge node before going live. |
| Shopify1 | Set | Mock Shopify orders payload | Get many orders | Merge1 | ## Data ingestion … Disable mock Set nodes and connect real API nodes directly to the Merge node before going live. |
| Stripe1 | Set | Mock Stripe payload | Get a time entry | Merge1 | ## Data ingestion … Disable mock Set nodes and connect real API nodes directly to the Merge node before going live. |
| Clockify1 | Set | Mock Clockify time entries payload | Get a balance | Merge1 | ## Data ingestion … Disable mock Set nodes and connect real API nodes directly to the Merge node before going live. |
| Merge1 | Merge | Combine all sources (5 inputs) | Facebook ads1; Google ads1; Shopify1; Stripe1; Clockify1 | Code in JavaScript | ## Processing logic … Update the CONFIG object in the code node… |
| Code in JavaScript | Code | Profitability calculation + risk flags + Slack summary | Merge1 | Append row in sheet1 | ## Processing logic … Update the CONFIG object in the code node… |
| Append row in sheet1 | Google Sheets | Append weekly metrics snapshot | Code in JavaScript | AI Profit Analyzer1 | ## Storage and notifications … Append weekly metrics… Post … to Slack… |
| Google Gemini Chat Model1 | Gemini Chat Model | LLM provider for the agent | — | AI Profit Analyzer1 (ai_languageModel) | ## AI analysis … Send structured financial data to the AI agent… |
| AI Profit Analyzer1 | LangChain Agent | Generate executive insights for Slack | Append row in sheet1; Google Gemini Chat Model1 | Send a message | ## AI analysis … Adjust the system message… |
| Send a message | Slack | Post report to Slack channel | AI Profit Analyzer1 | — | ## Storage and notifications … Ensure the Slack bot is invited… / ## Go live checklist … Remove all mock Set nodes… |

**Sticky notes (not attached to a single node):**
- **Go live checklist:** “Remove all mock Set nodes; confirm client naming consistency; validate Google Sheets mapping; review config; manual test before schedule.”

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** named: *Monitor agency profitability with Facebook Ads, Shopify, Stripe, Clockify, Google Sheets, Slack and Gemini*.

2. **Add trigger**
   - Add **Schedule Trigger** node.
   - Set cron to `0 8 * * 1` (Mondays 08:00).
   - Confirm instance timezone.

3. **Add Facebook Ads ingestion**
   - Add **Facebook Graph API** node.
   - Credentials: connect **Facebook Graph API** (with Ads insights permissions).
   - Configure:
     - Edge: `insights`
     - Graph API version: `v23.0`
     - Query parameters: `fields=spend`, `level=account`, and a `time_range`.
   - Important: ensure an `account_id` is available:
     - Option A: Add a **Set** node before it that sets `account_id`.
     - Option B: Iterate over multiple accounts (e.g., with an accounts list node) and pass `account_id`.

4. **Add Google Ads ingestion**
   - Add **Google Ads** node with OAuth2 credentials.
   - Replace the provided “get campaign” approach with a reporting/query that returns:
     - spend (micros), conversions, conversion value (optional)
   - Ensure the output can be normalized to a structure your code expects (either adjust node output or adjust the code).

5. **Add Shopify ingestion**
   - Add **Shopify** node with credentials.
   - Operation: **Get Many Orders / Get All**.
   - Add date filtering for the week (recommended) to avoid large pulls.
   - Ensure revenue fields you need are present (total, refunds).

6. **Add Clockify ingestion**
   - Add **Clockify** node with API key.
   - Use an operation that returns **all time entries for the period** (weekly range), not a single entry.
   - Ensure each entry includes (or can be mapped to):
     - `clientName`, `hours`, `costAmount`, `billable` (+ optional `revenueAttributed`)

7. **(Optional) Add Stripe ingestion**
   - Add **Stripe** node with API key.
   - If you intend to include Stripe-derived revenue/fees, decide what Stripe endpoint is needed (balance alone is not used by current code).
   - Otherwise, you can omit Stripe entirely and reduce merge inputs.

8. **Remove mock Set nodes (recommended for production)**
   - In the provided workflow, API nodes feed **Set mock nodes** (`Facebook ads1`, `Google ads1`, `Shopify1`, `Stripe1`, `Clockify1`).
   - For a real build, either:
     - **A. Skip creating mock Set nodes**, and connect real API nodes directly into the merge (and adjust code to their schemas), or
     - **B. Keep Set nodes but use them only to transform real outputs into the expected schema** (acting as mappers, not mock data).

9. **Add Merge**
   - Add **Merge** node.
   - Set **Number of Inputs = 5** (or fewer if you drop sources).
   - Connect each source branch into a separate merge input.

10. **Add Code node (profitability calculator)**
   - Add **Code** node (JavaScript).
   - Implement logic to:
     - Aggregate ad spend, revenue, conversions, time cost
     - Allocate platform fees, overhead, software cost per client
     - Compute ROAS, CAC, profit, margin, time ratio
     - Emit `summary_text`, `clientDetails`, `summary`, `breakdown`, `dataValidation`
   - Update CONFIG to match your agency:
     - Overhead rate, software costs, thresholds.

11. **Add Google Sheets storage**
   - Add **Google Sheets** node (OAuth2 credentials).
   - Operation: **Append** row.
   - Select Spreadsheet and Sheet.
   - Create headers to match the fields you will map.
   - Map from the Code node output. (If using automap, ensure keys exactly match columns.)

12. **Add Gemini model**
   - Add **Google Gemini Chat Model** node.
   - Credentials: connect Gemini/PaLM API key.
   - Keep defaults unless you need a specific model/temperature.

13. **Add AI Agent**
   - Add **LangChain Agent** node.
   - Connect the **Gemini model** into the agent’s **Language Model** input.
   - Prompt:
     - Provide `clientDetails`, `summary`, `breakdown`, `dataValidation`
     - Require Slack-friendly formatting
     - In system message: enforce using `riskLevel`, include $ and %, concise bullets.

14. **Add Slack notification**
   - Add **Slack** node → “Send Message”.
   - Credentials: Slack bot token with `chat:write` and channel access.
   - Select the channel.
   - Message body:
     - Include a date stamp
     - Include `summary_text` from the Code node
     - Include the agent output
   - Invite bot to channel.

15. **Test and go live**
   - Run manually once with real data.
   - Verify:
     - Client naming consistency across sources
     - Revenue correctness (Shopify vs ad-platform derived)
     - Sheet column mapping
     - Slack formatting
   - Enable the schedule.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Fetch ad spend from Facebook and Google, revenue from Shopify and Stripe, and time tracking data from Clockify. Disable mock Set nodes and connect real API nodes directly to the Merge node before going live. | Sticky note: **Data ingestion** |
| Merge all sources and calculate revenue, costs, ROAS, CAC, profit margin, and time ratio per client. Update the CONFIG object in the code node to match your real overhead rate, software costs, and minimum margin thresholds. | Sticky note: **Processing logic** |
| Send structured financial data to the AI agent to generate a concise executive summary. Adjust the system message if your definition of healthy performance changes. | Sticky note: **AI analysis** |
| Append weekly metrics to Google Sheets for historical tracking. Post a formatted profitability summary to Slack. Ensure the Slack bot is invited to the selected channel. | Sticky note: **Storage and notifications** |
| Go live checklist: Remove all mock Set nodes; Confirm client naming consistency across platforms; Validate Google Sheets column mapping; Review configuration values; Run a manual test before enabling schedule. | Sticky note: **Go live checklist** |
| Workflow concept summary + setup steps (credentials, remove mocks, update config, verify headers, test before schedule). | Sticky note: **Agency profitability monitor** |