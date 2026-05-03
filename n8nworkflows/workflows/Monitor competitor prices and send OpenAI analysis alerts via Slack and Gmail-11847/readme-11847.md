Monitor competitor prices and send OpenAI analysis alerts via Slack and Gmail

https://n8nworkflows.xyz/workflows/monitor-competitor-prices-and-send-openai-analysis-alerts-via-slack-and-gmail-11847


# Monitor competitor prices and send OpenAI analysis alerts via Slack and Gmail

## 1. Workflow Overview

**Title:** Competitor Price Monitoring with AI-Powered Alerts  
**Purpose:** On a schedule (every 6 hours), fetch current competitor prices, compare them to a previously stored snapshot in Google Sheets, detect meaningful changes, ask OpenAI for a concise strategy impact analysis, and route alerts to **Slack** (urgent) or **Gmail** (routine). It also logs activity and updates the stored snapshot.

### 1.1 Scheduled Run + Data Collection
- Triggers periodically, creates a monitoring session ID/timestamp, fetches current prices via HTTP, and loads the previous snapshot from Google Sheets.

### 1.2 Normalization + Dataset Comparison
- Normalizes both current and previous datasets into a consistent schema, then uses **Compare Datasets** keyed by `productId` to classify items as removed/same/different/new.

### 1.3 Change Enrichment + Significance Filtering
- For each comparison result, calculates change percent and assigns an `alertLevel` (`urgent`, `routine`, `none`), then filters to only significant (`urgent|routine`) changes.

### 1.4 AI Analysis + Alert Routing + Logging + Storage Update
- Uses an AI Agent (OpenAI chat model) to produce JSON analysis, parses it safely, routes to Slack or Gmail (or logs “extra”), logs results to Sheets, updates the price history snapshot, and logs the monitoring run.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduled Run + Data Collection
**Overview:** Starts the workflow on a cron schedule, initializes run metadata, fetches current prices from an external API, and reads the prior snapshot from Google Sheets.

**Nodes involved:**
- Price Check Schedule
- Initialize Monitoring Session
- Fetch Current Prices
- Fetch Previous Prices

#### Node: Price Check Schedule
- **Type / Role:** `scheduleTrigger` — entry point; triggers every 6 hours.
- **Configuration:** Cron expression `0 */6 * * *` (at minute 0, every 6th hour).
- **Outputs:** Connects to **Initialize Monitoring Session**.
- **Potential failures:** None typical besides disabled workflow or timezone misunderstandings (cron uses n8n instance timezone).

#### Node: Initialize Monitoring Session
- **Type / Role:** `set` — creates run-scoped variables.
- **Key fields created:**
  - `monitoringId` = `MON-{{$now.format('yyyyMMddHHmmss')}}`
  - `checkTimestamp` = `{{$now.toISO()}}`
  - `competitors` = `["CompetitorA","CompetitorB","CompetitorC"]` (currently not used downstream)
- **Outputs:** Fans out to **Fetch Current Prices** and **Fetch Previous Prices**.
- **Edge cases:** If later nodes rely on `competitors`, note it’s currently informational only.

#### Node: Fetch Current Prices
- **Type / Role:** `httpRequest` — pulls current product pricing data.
- **Configuration (interpreted):**
  - GET `https://api.competitor-prices.example.com/v1/products`
  - Query params: `category=all`, `format=json`
  - Timeout: 30s
  - Auth: `httpHeaderAuth` via predefined credential
- **Outputs:** To **Normalize Current Prices**.
- **Potential failures:**
  - Auth/header credential misconfigured
  - Non-JSON response or schema drift (`products` field missing)
  - Timeout / rate limiting / 4xx–5xx responses

#### Node: Fetch Previous Prices
- **Type / Role:** `googleSheets` — reads stored snapshot (previous prices).
- **Configuration:**
  - Operation: `getMany`
  - DocumentId: **not set in JSON** (must be selected)
- **Outputs:** To **Normalize Previous Prices**.
- **Potential failures:**
  - Missing documentId/sheet selection
  - OAuth/permissions issues
  - Header name mismatches (handled partially in normalization)

---

### Block 2 — Normalization + Dataset Comparison
**Overview:** Standardizes both datasets to consistent fields, then compares them by `productId` to segment new/removed/changed/unchanged.

**Nodes involved:**
- Normalize Current Prices
- Normalize Previous Prices
- Compare Price Datasets

#### Node: Normalize Current Prices
- **Type / Role:** `code` — converts the API response into a list of standardized product records.
- **Logic highlights:**
  - Reads all items from **Fetch Current Prices**.
  - For each item: uses `item.json.products || [item.json]` and iterates.
  - Output schema per product:
    - `productId` (from `id` or `sku`, else `'unknown'`)
    - `productName` (from `name` or `title`)
    - `competitor`, `currentPrice` (float), `currency`, `category`, `url`
    - `fetchedAt` from `Initialize Monitoring Session.checkTimestamp`
- **Output connection:** To **Compare Price Datasets** (as *Input B*, index 1).
- **Edge cases / failures:**
  - If API returns nested structures not matching expected fields, you may get many `unknown` IDs (bad for compare)
  - `parseFloat` on localized prices (e.g., `"12,99"`) becomes `12` or `NaN` → coerces to `0`
  - Competitor field missing → `'Unknown'`, which can reduce usefulness

#### Node: Normalize Previous Prices
- **Type / Role:** `code` — maps Sheet rows into standardized “previous snapshot”.
- **Logic highlights:**
  - Accepts flexible column naming:
    - `productId` from `productId` or `Product ID`
    - `productName` from `productName` or `Product Name`
    - `competitor` from `competitor` or `Competitor`
    - `previousPrice` from `currentPrice` or `Price`
    - `currency`, `category`, `lastUpdated`
- **Output connection:** To **Compare Price Datasets** (as *Input A*, index 0).
- **Edge cases:**
  - If Sheet columns differ from these names, fields become defaulted (e.g., `'unknown'`, `0`)
  - If the Sheet stores numbers as formatted strings, parsing can degrade precision

#### Node: Compare Price Datasets
- **Type / Role:** `compareDatasets` — classifies items based on key match.
- **Configuration:**
  - Merge/compare key: `productId` (A.productId ↔ B.productId)
- **Outputs (fixed by node design):**
  1. Output 0: **In A only** (removed)
  2. Output 1: **Same**
  3. Output 2: **Different**
  4. Output 3: **In B only** (new)
- **Connections:** Each output goes to its processing code node.
- **Edge cases:**
  - Duplicate `productId` values in either dataset can cause ambiguous matching or unexpected results
  - If many `productId` are `'unknown'`, comparison becomes meaningless (everything collides)

---

### Block 3 — Change Enrichment + Significance Filtering
**Overview:** Converts compare outputs into unified “change events”, assigns alert levels, merges them, and filters to only `urgent`/`routine`.

**Nodes involved:**
- Process Removed Products
- Process Unchanged
- Process Changed Prices
- Process New Products
- Merge All Changes
- Filter Significant Changes

#### Node: Process Removed Products
- **Type / Role:** `code` — tags removed products.
- **Output fields added:**
  - `changeType: 'REMOVED'`
  - `changePercent: -100`
  - `alertLevel: 'routine'`
  - `analysis: 'Product no longer available from competitor'`
- **Input:** Compare Price Datasets output “In A only”
- **Output:** To **Merge All Changes**
- **Edge cases:** Removal might be temporary API issue; consider retry/backoff at fetch stage.

#### Node: Process Unchanged
- **Type / Role:** `code` — tags stable prices.
- **Adds:**
  - `changeType: 'UNCHANGED'`, `changePercent: 0`, `alertLevel: 'none'`
- **Input:** Compare output “Same”
- **Output:** To **Merge All Changes**
- **Edge cases:** None; “none” changes are later filtered out.

#### Node: Process Changed Prices
- **Type / Role:** `code` — computes percent change and severity.
- **Key computations:**
  - `prevPrice = d.input1?.previousPrice || ...`
  - `currPrice = d.input2?.currentPrice || ...`
  - `changePercent = prevPrice > 0 ? ((curr-prev)/prev*100) : 0`
  - `alertLevel`:
    - `urgent` if `abs(changePercent) >= 10`
    - `routine` if `>= 5`
    - else `none`
  - `changeType` = `PRICE_INCREASE` or `PRICE_DECREASE` (note: if `changePercent` is `0`, it becomes `PRICE_DECREASE` due to `> 0` check)
- **Input:** Compare output “Different”
- **Output:** To **Merge All Changes**
- **Edge cases / correctness notes:**
  - If `prevPrice` is `0`, changePercent forced to `0` (may hide “0→X” jumps). You may want a special case.
  - A true 0% difference arriving here should arguably be “UNCHANGED”; relies on Compare Datasets accuracy.

#### Node: Process New Products
- **Type / Role:** `code` — tags newly appearing products.
- **Adds:**
  - `changeType: 'NEW_PRODUCT'`
  - `changePercent: null`
  - `alertLevel: 'routine'`
  - `analysis: 'New product detected in competitor catalog'`
- **Input:** Compare output “In B only”
- **Output:** To **Merge All Changes**

#### Node: Merge All Changes
- **Type / Role:** `merge` — combines streams of removed/unchanged/changed/new into one flow.
- **Configuration:** Default merge behavior (in practice used as a join point for multiple inputs).
- **Outputs:**
  - To **Filter Significant Changes**
  - To **Prepare Storage Update** (parallel branch for snapshot update)
- **Edge cases:** Merge behavior can be confusing when multiple inputs arrive; ensure it’s operating as intended for your n8n version (some merge modes wait for both inputs, others pass through).

#### Node: Filter Significant Changes
- **Type / Role:** `filter` — keeps only items with `alertLevel` of `urgent` or `routine`.
- **Condition:** OR:
  - `$json.alertLevel == 'urgent'` OR `'routine'`
- **Output:** To **AI Price Analyst**
- **Edge cases:** If `alertLevel` missing, item is dropped (expected).

---

### Block 4 — AI Analysis + Alert Routing + Logging + Storage Update
**Overview:** For each significant change, request AI analysis, parse it into structured JSON, route to Slack or Gmail (or log via Sheets), then log the run and update snapshot storage.

**Nodes involved:**
- OpenAI Chat Model
- AI Price Analyst
- Parse AI Analysis
- Route by Alert Level
- Slack Urgent Alert
- Email Routine Alert
- Log Low Priority Changes
- Merge Alert Results
- Prepare Storage Update
- Update Price History
- Log Monitoring Run

#### Node: OpenAI Chat Model
- **Type / Role:** `lmChatOpenAi` (LangChain) — provides the model for the agent.
- **Configuration:**
  - Model: `gpt-4o-mini`
  - Temperature: `0.2` (low variance)
- **Connection:** Feeds the **AI Price Analyst** via `ai_languageModel`.
- **Credential requirement:** OpenAI API key configured in n8n for this node.
- **Failure modes:** Invalid key, quota exceeded, model name not available in region/account, network timeouts.

#### Node: AI Price Analyst
- **Type / Role:** `@n8n/n8n-nodes-langchain.agent` — runs an agent prompt to produce structured strategic analysis.
- **Prompt:** Injects product/competitor/category/previous/current price/change%/type and requests JSON keys:
  - `impactAssessment`, `recommendedAction`, `urgencyJustification`
- **System message:** “You are a pricing strategy analyst… Always output valid JSON.”
- **Input:** From **Filter Significant Changes**
- **Output:** To **Parse AI Analysis**
- **Failure modes / edge cases:**
  - Agent output may still include non-JSON text; handled later by regex extraction
  - Token limits if you expand prompt with lots of context
  - If multiple items pass, agent runs per item (cost consideration)

#### Node: Parse AI Analysis
- **Type / Role:** `code` — extracts JSON from AI output and attaches it to the original item.
- **Logic highlights:**
  - `aiOutput = priceData.output || '{}'`
  - Regex `\{[\s\S]*\}` to capture first JSON block; `JSON.parse`
  - Fallback default analysis if parsing fails
  - Then fetches original change data using: `$('Filter Significant Changes').first().json`
    - **Important:** this uses `.first()` from the node, not the current item, which can incorrectly attach the first change to every AI result when there are multiple changes in one run.
- **Output:** To **Route by Alert Level**
- **Edge cases / bug risk:**
  - **Data mismatch risk (batch issue):** should generally use the current `$input` item’s original fields rather than referencing `Filter Significant Changes.first()`.
  - Regex can capture too much if model returns multiple JSON objects.

#### Node: Route by Alert Level
- **Type / Role:** `switch` — routes alerts based on `alertLevel`.
- **Rules / outputs:**
  - Output 0: `alertLevel == urgent` → Slack Urgent Alert
  - Output 1: `alertLevel == routine` → Email Routine Alert
  - Fallback (“extra”) → Log Low Priority Changes
- **Edge cases:** If `alertLevel` missing/unknown, it goes to fallback.

#### Node: Slack Urgent Alert
- **Type / Role:** `slack` — sends urgent alerts to a channel.
- **Configuration:**
  - Channel: `#price-alerts`
  - Message template includes product details + `aiAnalysis.*`
- **Credential requirement:** Slack OAuth/token in n8n.
- **Failure modes:** Missing scopes, channel not found, formatting issues, rate limiting.

#### Node: Email Routine Alert
- **Type / Role:** `gmail` — emails routine alerts.
- **Configuration:**
  - To: `pricing-team@company.com`
  - Subject: `Price Change Alert: {{productName}} ({{changePercent}}%)`
  - Body includes AI assessment + recommended action
- **Credential requirement:** Gmail OAuth2.
- **Failure modes:** OAuth expired, “from” restrictions, sending limits.

#### Node: Log Low Priority Changes
- **Type / Role:** `googleSheets` — appends alert records to an `AlertLog` sheet.
- **Configuration:**
  - Operation: `append`
  - Sheet name: `AlertLog`
  - Columns mapped (e.g., Change %, Timestamp, Competitor, Product ID, AI Recommendation, etc.)
  - DocumentId: **not set in JSON** (must be selected)
- **Connection:** To **Merge Alert Results** (index 1 input).
- **Edge cases:** If routed here due to fallback, values like `aiAnalysis` might be missing unless parse succeeded.

#### Node: Merge Alert Results
- **Type / Role:** `merge` (mode: chooseBranch) — rejoins Slack/Email/Log branches.
- **Inputs:** From Slack, Email, and Low Priority log.
- **Output:** To **Log Monitoring Run**
- **Edge cases:** ChooseBranch expects one branch at a time per item; works well with switch-style routing.

#### Node: Prepare Storage Update
- **Type / Role:** `code` — prepares the current normalized dataset for writing back to Sheets.
- **Logic:** Pulls *all* items from `Normalize Current Prices` and maps to:
  - `productId, productName, competitor, currentPrice, currency, category, fetchedAt`
- **Output:** To **Update Price History**
- **Edge cases:** If current fetch is partial/failed, you may overwrite snapshot with incomplete data.

#### Node: Update Price History
- **Type / Role:** `googleSheets` — updates snapshot in `PriceHistory`.
- **Configuration:**
  - Operation: `update`
  - Sheet: `PriceHistory`
  - Mapping columns: Product ID, Product Name, Competitor, Price, Currency, Category, Last Updated
  - Cell format: `USER_ENTERED`
  - DocumentId: **not set in JSON**
- **Edge cases / important:**
  - “update” requires a row identity strategy (row number or a key-based lookup). The provided configuration does not show how the target row is determined; you may need to switch to an upsert pattern (lookup row by Product ID + Competitor, then update, else append).

#### Node: Log Monitoring Run
- **Type / Role:** `googleSheets` — appends a run-level log entry.
- **Configuration:**
  - Operation: `append`
  - Sheet: `MonitoringLog`
  - Columns include:
    - Status: `Completed`
    - Timestamp / Monitoring ID from Initialize node
    - Changes Detected: `$('Filter Significant Changes').all().length`
    - Products Checked: `$('Normalize Current Prices').all().length`
  - DocumentId: **not set in JSON**
- **Edge cases:**
  - If earlier nodes fail, this log may never run (no try/catch at workflow level)
  - If Filter node never executes (e.g., merge behavior), counts can error in expressions

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | n8n-nodes-base.stickyNote | Documentation / canvas note | — | — | ## Competitor Price Monitoring with AI-Powered Alerts; Overview; Key Features; Required Credentials (OpenAI API, Slack, Gmail, Google Sheets) |
| Sticky Note1 | n8n-nodes-base.stickyNote | Documentation / canvas note | — | — | ### Step 1: Scheduled Price Collection; Cron trigger; HTTP fetch; previous snapshot retrieval |
| Sticky Note2 | n8n-nodes-base.stickyNote | Documentation / canvas note | — | — | ### Step 2: Compare Datasets Analysis; outputs; percentage change |
| Sticky Note3 | n8n-nodes-base.stickyNote | Documentation / canvas note | — | — | ### Step 3: AI Analysis & Severity Assessment; recommendations; urgency |
| Sticky Note4 | n8n-nodes-base.stickyNote | Documentation / canvas note | — | — | ### Step 4: Smart Alert Routing & Storage; Slack urgent; Email routine; update Sheets; logs |
| Price Check Schedule | n8n-nodes-base.scheduleTrigger | Scheduled trigger | — | Initialize Monitoring Session | ### Step 1: Scheduled Price Collection; Cron trigger; HTTP fetch; previous snapshot retrieval |
| Initialize Monitoring Session | n8n-nodes-base.set | Create run metadata | Price Check Schedule | Fetch Current Prices; Fetch Previous Prices | ### Step 1: Scheduled Price Collection; Cron trigger; HTTP fetch; previous snapshot retrieval |
| Fetch Current Prices | n8n-nodes-base.httpRequest | Pull current competitor prices | Initialize Monitoring Session | Normalize Current Prices | ### Step 1: Scheduled Price Collection; Cron trigger; HTTP fetch; previous snapshot retrieval |
| Fetch Previous Prices | n8n-nodes-base.googleSheets | Read previous snapshot | Initialize Monitoring Session | Normalize Previous Prices | ### Step 1: Scheduled Price Collection; Cron trigger; HTTP fetch; previous snapshot retrieval |
| Normalize Current Prices | n8n-nodes-base.code | Standardize current price schema | Fetch Current Prices | Compare Price Datasets | ### Step 2: Compare Datasets Analysis; outputs; percentage change |
| Normalize Previous Prices | n8n-nodes-base.code | Standardize previous price schema | Fetch Previous Prices | Compare Price Datasets | ### Step 2: Compare Datasets Analysis; outputs; percentage change |
| Compare Price Datasets | n8n-nodes-base.compareDatasets | Diff previous vs current by productId | Normalize Previous Prices; Normalize Current Prices | Process Removed Products; Process Unchanged; Process Changed Prices; Process New Products | ### Step 2: Compare Datasets Analysis; outputs; percentage change |
| Process Removed Products | n8n-nodes-base.code | Tag removals | Compare Price Datasets | Merge All Changes | ### Step 2: Compare Datasets Analysis; outputs; percentage change |
| Process Unchanged | n8n-nodes-base.code | Tag unchanged | Compare Price Datasets | Merge All Changes | ### Step 2: Compare Datasets Analysis; outputs; percentage change |
| Process Changed Prices | n8n-nodes-base.code | Compute % change + severity | Compare Price Datasets | Merge All Changes | ### Step 2: Compare Datasets Analysis; outputs; percentage change |
| Process New Products | n8n-nodes-base.code | Tag new products | Compare Price Datasets | Merge All Changes | ### Step 2: Compare Datasets Analysis; outputs; percentage change |
| Merge All Changes | n8n-nodes-base.merge | Consolidate change events | Process Removed Products; Process Unchanged; Process Changed Prices; Process New Products | Filter Significant Changes; Prepare Storage Update | ### Step 4: Smart Alert Routing & Storage; Slack urgent; Email routine; update Sheets; logs |
| Filter Significant Changes | n8n-nodes-base.filter | Keep urgent/routine only | Merge All Changes | AI Price Analyst | ### Step 3: AI Analysis & Severity Assessment; recommendations; urgency |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM provider | — | AI Price Analyst (ai_languageModel) | ### Step 3: AI Analysis & Severity Assessment; recommendations; urgency |
| AI Price Analyst | @n8n/n8n-nodes-langchain.agent | Generate strategic analysis JSON | Filter Significant Changes; OpenAI Chat Model | Parse AI Analysis | ### Step 3: AI Analysis & Severity Assessment; recommendations; urgency |
| Parse AI Analysis | n8n-nodes-base.code | Parse/attach AI JSON | AI Price Analyst | Route by Alert Level | ### Step 3: AI Analysis & Severity Assessment; recommendations; urgency |
| Route by Alert Level | n8n-nodes-base.switch | Channel routing | Parse AI Analysis | Slack Urgent Alert; Email Routine Alert; Log Low Priority Changes | ### Step 4: Smart Alert Routing & Storage; Slack urgent; Email routine; update Sheets; logs |
| Slack Urgent Alert | n8n-nodes-base.slack | Send urgent Slack message | Route by Alert Level | Merge Alert Results | ### Step 4: Smart Alert Routing & Storage; Slack urgent; Email routine; update Sheets; logs |
| Email Routine Alert | n8n-nodes-base.gmail | Send routine email | Route by Alert Level | Merge Alert Results | ### Step 4: Smart Alert Routing & Storage; Slack urgent; Email routine; update Sheets; logs |
| Log Low Priority Changes | n8n-nodes-base.googleSheets | Append alert record to AlertLog | Route by Alert Level (fallback) | Merge Alert Results | ### Step 4: Smart Alert Routing & Storage; Slack urgent; Email routine; update Sheets; logs |
| Merge Alert Results | n8n-nodes-base.merge | Rejoin routed branches | Slack Urgent Alert; Email Routine Alert; Log Low Priority Changes | Log Monitoring Run | ### Step 4: Smart Alert Routing & Storage; Slack urgent; Email routine; update Sheets; logs |
| Prepare Storage Update | n8n-nodes-base.code | Prepare snapshot rows | Merge All Changes | Update Price History | ### Step 4: Smart Alert Routing & Storage; Slack urgent; Email routine; update Sheets; logs |
| Update Price History | n8n-nodes-base.googleSheets | Write current snapshot to PriceHistory | Prepare Storage Update | — | ### Step 4: Smart Alert Routing & Storage; Slack urgent; Email routine; update Sheets; logs |
| Log Monitoring Run | n8n-nodes-base.googleSheets | Append run log to MonitoringLog | Merge Alert Results | — | ### Step 4: Smart Alert Routing & Storage; Slack urgent; Email routine; update Sheets; logs |

---

## 4. Reproducing the Workflow from Scratch

1. **Create workflow** named **“Competitor Price Monitoring with AI-Powered Alerts”**.

2. **Add trigger**
   - Node: **Schedule Trigger**
   - Set cron to: `0 */6 * * *`

3. **Add “Initialize Monitoring Session” (Set node)**
   - Add fields:
     - `monitoringId` (String): `MON-{{$now.format('yyyyMMddHHmmss')}}`
     - `checkTimestamp` (String): `{{$now.toISO()}}`
     - `competitors` (Object/JSON): `["CompetitorA","CompetitorB","CompetitorC"]`
   - Connect: Schedule Trigger → Initialize Monitoring Session

4. **Add “Fetch Current Prices” (HTTP Request)**
   - Method: GET
   - URL: `https://api.competitor-prices.example.com/v1/products`
   - Query params: `category=all`, `format=json`
   - Options: Timeout 30000ms
   - Auth: Header Auth credential (configure your API key/header)
   - Connect: Initialize Monitoring Session → Fetch Current Prices

5. **Add “Fetch Previous Prices” (Google Sheets)**
   - Operation: **Get Many**
   - Select **Document** (spreadsheet) that holds prior snapshot
   - Select the appropriate sheet or range (as needed by your setup)
   - Connect: Initialize Monitoring Session → Fetch Previous Prices

6. **Add “Normalize Current Prices” (Code node)**
   - Paste logic to map API response into items with:
     `productId, productName, competitor, currentPrice, currency, category, url, fetchedAt`
   - Connect: Fetch Current Prices → Normalize Current Prices

7. **Add “Normalize Previous Prices” (Code node)**
   - Map sheet rows into:
     `productId, productName, competitor, previousPrice, currency, category, lastUpdated`
   - Connect: Fetch Previous Prices → Normalize Previous Prices

8. **Add “Compare Price Datasets” (Compare Datasets node)**
   - Set **Merge By Fields**: `productId` (A field) to `productId` (B field)
   - Connect:
     - Normalize Previous Prices → Compare Price Datasets **Input 0**
     - Normalize Current Prices → Compare Price Datasets **Input 1**

9. **Add 4 processing Code nodes**
   - **Process Removed Products**: tag `REMOVED`, `changePercent=-100`, `alertLevel=routine`
   - **Process Unchanged**: tag `UNCHANGED`, `alertLevel=none`
   - **Process Changed Prices**: compute `changePercent`, set `alertLevel` based on thresholds (>=10 urgent, >=5 routine)
   - **Process New Products**: tag `NEW_PRODUCT`, `alertLevel=routine`
   - Connect Compare outputs:
     - Output 0 → Process Removed Products
     - Output 1 → Process Unchanged
     - Output 2 → Process Changed Prices
     - Output 3 → Process New Products

10. **Add “Merge All Changes” (Merge node)**
    - Use it as a convergence point for the four processors.
    - Connect each processor → Merge All Changes

11. **Add “Filter Significant Changes” (Filter node)**
    - Condition (OR):
      - `alertLevel == "urgent"`
      - `alertLevel == "routine"`
    - Connect: Merge All Changes → Filter Significant Changes

12. **Add OpenAI model + Agent**
    - Node: **OpenAI Chat Model (LangChain)**
      - Model: `gpt-4o-mini`
      - Temperature: `0.2`
      - Configure **OpenAI credentials**
    - Node: **AI Agent**
      - Prompt: include product fields and request JSON output with keys:
        `impactAssessment`, `recommendedAction`, `urgencyJustification`
      - System message: enforce “Always output valid JSON.”
    - Connect:
      - Filter Significant Changes → AI Agent (main)
      - OpenAI Chat Model → AI Agent (ai_languageModel connection)

13. **Add “Parse AI Analysis” (Code)**
    - Parse the agent output into `aiAnalysis` JSON (with fallback if parsing fails)
    - Add `analyzedAt`
    - Connect: AI Agent → Parse AI Analysis  
    - **Recommended fix while reproducing:** avoid `$('Filter Significant Changes').first()`; instead merge parsed AI output with the *current input item* to prevent multi-item mismatch.

14. **Add “Route by Alert Level” (Switch)**
    - Rule 1: `alertLevel == urgent` → Output 0
    - Rule 2: `alertLevel == routine` → Output 1
    - Fallback output enabled → Output 2 (“extra”)
    - Connect: Parse AI Analysis → Route by Alert Level

15. **Add alert actions**
    - **Slack node** (Slack Urgent Alert)
      - Channel: `#price-alerts`
      - Message: include product details + `aiAnalysis.*`
      - Configure Slack credentials
      - Connect: Switch output 0 → Slack Urgent Alert
    - **Gmail node** (Email Routine Alert)
      - To: `pricing-team@company.com`
      - Subject/body templates as in workflow
      - Configure Gmail OAuth2 credentials
      - Connect: Switch output 1 → Email Routine Alert
    - **Google Sheets node** (Log Low Priority Changes)
      - Operation: Append
      - Sheet: `AlertLog`
      - Map columns including AI recommendation, prices, timestamp
      - Connect: Switch fallback → Log Low Priority Changes

16. **Add “Merge Alert Results” (Merge)**
    - Mode: **Choose Branch**
    - Connect Slack + Gmail + AlertLog → Merge Alert Results

17. **Add “Log Monitoring Run” (Google Sheets append)**
    - Sheet: `MonitoringLog`
    - Columns:
      - Status = Completed
      - Timestamp / Monitoring ID from Initialize Monitoring Session
      - Changes Detected = number of items from Filter Significant Changes
      - Products Checked = number of items from Normalize Current Prices
    - Connect: Merge Alert Results → Log Monitoring Run

18. **Add snapshot update branch**
    - **Prepare Storage Update** (Code)
      - Build rows from **Normalize Current Prices** items
    - **Update Price History** (Google Sheets)
      - Sheet: `PriceHistory`
      - Operation: update (or implement upsert with lookup + update/append)
    - Connect: Merge All Changes → Prepare Storage Update → Update Price History

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Automatically monitors competitor pricing on a scheduled basis, detects significant price changes using Compare Datasets, and sends intelligent alerts via Slack or Email based on AI analysis of market implications. | From main sticky note “Competitor Price Monitoring with AI-Powered Alerts” |
| Required Credentials: OpenAI API, Slack, Gmail, Google Sheets | From main sticky note |
| Cron scheduling provides flexible monitoring frequency (configured here as every 6 hours). | From Step 1 sticky note |
| Compare Datasets outputs: In A only / Same / Different / In B only | From Step 2 sticky note |
| AI agent generates strategic recommendations and classifies urgency (urgent/routine/log-only). | From Step 3 sticky note |
| Routing rule intent: Slack for urgent (>10%), Email for routine (5–10%), log/store updates in Sheets. | From Step 4 sticky note |