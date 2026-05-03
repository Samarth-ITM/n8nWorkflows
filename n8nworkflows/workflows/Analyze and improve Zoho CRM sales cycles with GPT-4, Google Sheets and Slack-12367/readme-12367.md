Analyze and improve Zoho CRM sales cycles with GPT-4, Google Sheets and Slack

https://n8nworkflows.xyz/workflows/analyze-and-improve-zoho-crm-sales-cycles-with-gpt-4--google-sheets-and-slack-12367


# Analyze and improve Zoho CRM sales cycles with GPT-4, Google Sheets and Slack

## 1. Workflow Overview

This workflow retrieves recently closed deals from **Zoho CRM**, compares their sales-cycle duration to **historical cycle data stored in Google Sheets**, enriches the analysis with **GPT‑4** (sentiment, outcome prediction, and recommendations), generates lightweight visualization metadata, then posts a formatted summary to **Slack** and appends the results to a historical Google Sheet.

### 1.1 Manual Start / Entry Point
Manually starts the workflow (intended to be replaceable by a Schedule trigger).

### 1.2 Data Collection (Zoho + Filtering + Historical Sheet)
Pulls deals from Zoho, filters to “recent” (last 7 days, max 10), and fetches historical rows from Google Sheets.

### 1.3 Core Cycle Analytics (Deterministic Calculations)
Computes cycle duration, historical statistics (avg/median/p90), best close day, and stage dwell bottlenecks, then produces insights + baseline suggestions.

### 1.4 AI Intelligence Layer (GPT‑4)
Runs GPT‑4 sentiment analysis and predictive analytics in parallel, aggregates outputs, builds visualization metadata, then asks GPT‑4 for prioritized recommendations.

### 1.5 Validation, Notifications & Logging
Validates that the dataset is usable, then posts to Slack and appends a row to Google Sheets.

---

## 2. Block-by-Block Analysis

### Block 1.1 — Manual Start / Entry Point

**Overview:** Provides a manual entry point for testing and ad-hoc execution. Typically replaced with a Schedule Trigger in production.

**Nodes Involved:**
- **When clicking 'Execute workflow'** (Manual Trigger)

#### Node: When clicking 'Execute workflow'
- **Type / Role:** `Manual Trigger` — starts the workflow on demand.
- **Configuration (interpreted):** No parameters; default manual trigger.
- **Input/Output:**
  - **Output →** Zoho CRM - Deal Trigger
- **Edge cases / failures:** None (other than workflow misconfiguration downstream).
- **Version notes:** TypeVersion 1.

---

### Block 1.2 — Data Collection (Zoho + Filtering + Historical Sheet)

**Overview:** Fetches deals from Zoho CRM, filters to the last 7 days and caps volume to 10 items, then loads historical rows from Google Sheets to support benchmarking.

**Nodes Involved:**
- **Zoho CRM - Deal Trigger**
- **Filter Recent Deals**
- **Fetch Historical Averages**

#### Node: Zoho CRM - Deal Trigger
- **Type / Role:** `Zoho CRM` — reads deal records from Zoho.
- **Configuration (interpreted):**
  - **Resource:** Deal
  - **Operation:** Get All (pulls multiple deals; not an actual “trigger” despite the name)
  - **Options:** none specified
- **Connections:**
  - **Input ←** Manual Trigger
  - **Output →** Filter Recent Deals
- **Key data fields expected downstream:** `Created_Time`, `Closed_Time`, `Modified_Time`, `Deal_Name`, `Stage`, `Owner`, `Amount`, optionally `Stage_History`.
- **Edge cases / failures:**
  - Zoho auth failure / expired token.
  - Pagination / large dataset: “getAll” can pull many deals depending on account size; can be slow or hit API limits.
  - Missing fields (e.g., `Closed_Time`) affects later computations.
- **Version notes:** TypeVersion 1 (older Zoho node behavior may differ from newer versions in pagination/options).

#### Node: Filter Recent Deals
- **Type / Role:** `Code` — reduces noise and caps Slack output volume.
- **Configuration (interpreted):**
  - Filters to deals whose **Closed_Time** (preferred) or **Modified_Time** is within the last **7 days**.
  - Drops items without either timestamp.
  - Limits output to **first 10** filtered deals.
- **Key logic/variables:**
  - `sevenDaysAgo = now - 7 days`
  - `relevantTime = Closed_Time || Modified_Time`
  - `return filtered.slice(0, 10)`
- **Connections:**
  - **Input ←** Zoho CRM - Deal Trigger (many items)
  - **Output →** Fetch Historical Averages (up to 10 items)
- **Edge cases / failures:**
  - Date parsing issues if Zoho returns non-ISO formats.
  - Time zone boundary effects (a deal “7 days ago” near midnight can flip inclusion).
  - Limiting to 10 means additional recent deals will be ignored.
- **Version notes:** TypeVersion 2 (Code node; `$item()` usage appears later in workflow).

#### Node: Fetch Historical Averages
- **Type / Role:** `Google Sheets` — reads historical deal rows used to compute averages/percentiles.
- **Configuration (interpreted):**
  - Uses Google Sheets credentials.
  - **Document ID** and **Sheet name** are present but currently **blank** in the JSON (must be configured).
  - Operation is not explicitly shown in parameters; with this node it typically defaults to a “read/get many rows” style operation (depending on node UI defaults).
- **Connections:**
  - **Input ←** Filter Recent Deals
  - **Output →** Analyze Cycle
- **Edge cases / failures:**
  - Missing Google credentials or insufficient permissions.
  - Empty/blank Document ID or Sheet name will fail.
  - Header mismatch: downstream expects columns like `Created_Time`, `Closed_Time`, `Stage`, `Deal_Name`, optionally `Stage_History` (with flexible aliasing).
- **Version notes:** TypeVersion 3 (Google Sheets node v3 has different parameter structure than v2).

---

### Block 1.3 — Core Cycle Analytics (Deterministic Calculations)

**Overview:** Computes the cycle length for a deal, benchmarks it against historical durations, detects bottlenecks using stage history if available, and produces insights + baseline suggestions.

**Nodes Involved:**
- **Analyze Cycle**

#### Node: Analyze Cycle
- **Type / Role:** `Code` — core analytics computation and normalization.
- **Configuration (interpreted):**
  - Selects the deal mainly using:
    - `$item(0).$node["Filter Recent Deals"].json` OR `$item(0).$node["Zoho CRM - Deal Trigger"].json`
  - Calculates:
    - `totalDays` from `Created_Time` to `Closed_Time` (fallback `Modified_Time`), minimum 1 day.
    - Builds historical rows from **incoming items** (coming from Google Sheets output).
    - Normalizes column names across multiple variants.
    - Historical cycle stats: **avg**, **median**, **p90**.
    - “Best close day of week” using rows whose Stage contains “won”.
    - Stage dwell bottleneck using `Stage_History` if parseable as JSON array.
    - Average dwell by stage from historical `Stage_History` (if present).
    - Produces `insights[]`, `suggestions[]`, `hasData`.
- **Key expressions/variables:**
  - Deal selection:
    - `const deal = $item(0).$node["Filter Recent Deals"].json || ...`
  - Historical row normalization:
    - `Created_Time: row.Created_Time || row['Created Time'] || ...`
  - Bottleneck logic compares `longestDwellDays` to `avgStageDwells[stage] * 1.5`.
- **Connections:**
  - **Input ←** Fetch Historical Averages
  - **Output →** AI Sentiment Analysis AND AI Predictive Analytics (parallel branches)
- **Important implementation caveat (data-shape mismatch):**
  - This node assumes `items` contains **Google Sheets rows**, but it also tries to pull the **deal** from a different node via `$item(0).$node[...]`.
  - If multiple deals (up to 10) are passed through, `$item(0)` always refers to the first item’s context; this can cause:
    - Analysis always using the first deal, not each deal.
    - Misalignment between which deal is analyzed and which sheet rows are used.
- **Edge cases / failures:**
  - Missing `Created_Time` or `Closed_Time` yields `totalDays = null` and reduced insights.
  - `Stage_History` parsing failures (non-JSON, unexpected schema) are silently skipped.
  - If the Google Sheet returns no usable rows, `avgDays/median/p90` become null and insights degrade.
- **Version notes:** TypeVersion 2.

---

### Block 1.4 — AI Intelligence Layer (GPT‑4)

**Overview:** Enriches deterministic metrics with GPT‑4: sentiment analysis, predictive outcome analytics, aggregates results, builds chart metadata, and generates prioritized actionable recommendations.

**Nodes Involved:**
- **AI Sentiment Analysis**
- **AI Predictive Analytics**
- **AI Data Aggregation**
- **Advanced Data Visualization**
- **AI Smart Recommendations**

#### Node: AI Sentiment Analysis
- **Type / Role:** `OpenAI` — sentiment/risk inference.
- **Configuration (interpreted):**
  - **Model:** `gpt-4`
  - Prompt requests: sentiment (pos/neutral/neg), confidence 0–100, indicators, risk level.
  - Injects: `Deal Data: {{ JSON.stringify($json.deal) }}`
- **Connections:**
  - **Input ←** Analyze Cycle
  - **Output →** AI Data Aggregation
- **Edge cases / failures:**
  - OpenAI credential missing / quota exceeded / rate limit.
  - Prompt returns non-JSON free text; later aggregation attempts to JSON.parse and may fail (handled with try/catch).
- **Version notes:** TypeVersion 1 (older OpenAI node; response shape may differ from current “Chat” nodes).

#### Node: AI Predictive Analytics
- **Type / Role:** `OpenAI` — forecasts win probability and expected close date.
- **Configuration (interpreted):**
  - **Model:** `gpt-4`
  - Prompt requests: win probability, expected close date, risk factors, confidence.
  - Injects:
    - `Historical Context: {{ JSON.stringify($json.historicalData) }}`
    - `Current Deal: {{ JSON.stringify($json.deal) }}`
- **Connections:**
  - **Input ←** Analyze Cycle
  - **Output →** AI Data Aggregation
- **Critical data issue:**
  - `Analyze Cycle` does **not** output `historicalData`; it outputs stats like `avgDays`, etc. So `{{ JSON.stringify($json.historicalData) }}` will typically be `undefined`.
- **Edge cases / failures:** same as other OpenAI calls plus “missing historical context reduces quality”.
- **Version notes:** TypeVersion 1.

#### Node: AI Data Aggregation
- **Type / Role:** `Code` — merges outputs from parallel branches.
- **Configuration (interpreted):**
  - Assumes 3 inbound items:
    - `mainData = $input.first().json`
    - `sentimentData = $input.item(1)?.json`
    - `predictiveData = $input.item(2)?.json`
  - Extracts likely response payload via `.text` or raw object.
  - Attempts `JSON.parse()` on string responses; keeps as string on failure.
  - Outputs merged JSON with `sentimentAnalysis`, `predictiveAnalytics`, and ensures `suggestions` exists.
- **Connections:**
  - **Inputs ←** AI Sentiment Analysis, AI Predictive Analytics (and implicitly “main” item)
  - **Output →** Advanced Data Visualization
- **Critical wiring/logic risk:**
  - In n8n, a merge/aggregation of parallel branches typically requires a **Merge** node or careful item pairing. This code node expects multiple input items, but the workflow connections show only the two AI nodes feeding it; there is no explicit third “main” branch feeding aggregation at the same junction beyond how n8n bundles items.
  - Index-based `$input.item(1)` / `$input.item(2)` is fragile and can break if item counts differ or execution order changes.
- **Edge cases / failures:**
  - If only one input item arrives, sentiment/predictive will be empty.
  - Response formats vary by OpenAI node version; `.text` may not exist.
- **Version notes:** TypeVersion 2.

#### Node: Advanced Data Visualization
- **Type / Role:** `Code` — produces chart-ready structures and a summary string.
- **Configuration (interpreted):**
  - Expects:
    - `deal = $json.deal`
    - `historical = $json.historicalData || []` (but `historicalData` is not produced earlier)
  - Generates:
    - `visualizationData.cycleTimeTrend` from `historical`
    - `stageDistribution` and `performanceMetrics` from `historical`
    - `riskFactors` from sentiment/cycle/prediction
  - Produces `charts` objects (line/pie/gauge metadata) and `chartSummary`.
- **Connections:**
  - **Input ←** AI Data Aggregation
  - **Output →** AI Smart Recommendations
- **Edge cases / failures:**
  - If `historical` is empty, winRate computation divides by `historical.length` (0), resulting in `NaN`. This may propagate into Slack.
  - If `avgDays` is null, `$json.totalDays > ($json.avgDays * 1.5)` yields `false` or `NaN` behavior depending on JS coercion.
- **Version notes:** TypeVersion 2.

#### Node: AI Smart Recommendations
- **Type / Role:** `OpenAI` — creates final prioritized recommendations.
- **Configuration (interpreted):**
  - **Model:** `gpt-4`
  - Prompt instructs: 3–5 actionable recommendations, include priority and expected impact.
  - Injects:
    - `Deal Analysis: {{ JSON.stringify($json) }}`
    - `Sentiment Analysis: {{ $json.sentimentAnalysis || 'Not available' }}`
    - `Predictive Analytics: {{ $json.predictiveAnalytics || 'Not available' }}`
- **Connections:**
  - **Input ←** Advanced Data Visualization
  - **Output →** Check Valid Data
- **Edge cases / failures:**
  - Output format is free text; downstream Slack template expects `$json.text` (works if OpenAI node returns `.text`) but may differ by node/version.
- **Version notes:** TypeVersion 1.

---

### Block 1.5 — Validation, Notifications & Logging

**Overview:** Ensures minimum viable fields exist, then posts a formatted Slack message and appends the analyzed record to Google Sheets.

**Nodes Involved:**
- **Check Valid Data**
- **Slack Notification**
- **Log to Historical Sheet**

#### Node: Check Valid Data
- **Type / Role:** `IF` — data quality gate.
- **Configuration (interpreted):** Requires BOTH:
  1) `deal.Deal_Name` is not empty  
  2) `totalDays > 0`
- **Connections:**
  - **Input ←** AI Smart Recommendations
  - **True Output →** Slack Notification AND Log to Historical Sheet
  - **False Output →** (not connected; data is dropped)
- **Edge cases / failures:**
  - If OpenAI node output overwrote or changed the JSON structure, `deal.Deal_Name` may be missing.
  - No false-branch handling means silent failure (no alert).
- **Version notes:** TypeVersion 2.2.

#### Node: Slack Notification
- **Type / Role:** `Slack` — sends formatted insights to a channel.
- **Configuration (interpreted):**
  - Posts to `#sales-insights`.
  - Message includes: deal meta, cycle stats, “performanceScore” and “riskLevel” (but these are not produced anywhere deterministically), insights list, sentiment/prediction fields, recommendations (prefers `$json.text`, fallback to `$json.suggestions`), chart summary.
- **Connections:**
  - **Input ←** Check Valid Data (true)
  - **Output:** none
- **Edge cases / failures:**
  - Slack auth failure / missing scopes (chat:write).
  - Channel not found or bot not invited to channel.
  - Template references missing fields:
    - `performanceScore`, `riskLevel` are never set by earlier nodes (will show “N/A/Unknown”).
    - `sentimentAnalysis?.sentiment` assumes a JSON structure that may not match GPT output.
- **Version notes:** TypeVersion 1.

#### Node: Log to Historical Sheet
- **Type / Role:** `Google Sheets` — appends analyzed data to a sheet.
- **Configuration (interpreted):**
  - **Operation:** Append
  - **Document ID / Sheet name:** present but blank in JSON; must be configured.
  - No column mapping is shown; append behavior depends on node UI config (often “auto-map by fields” or explicit columns).
- **Connections:**
  - **Input ←** Check Valid Data (true)
  - **Output:** none
- **Edge cases / failures:**
  - Missing/blank Sheet ID or name.
  - If the appended JSON contains nested objects (deal, charts), the node may fail or stringify unexpectedly unless mapped to flat columns.
- **Version notes:** TypeVersion 3.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking 'Execute workflow' | Manual Trigger | Manual workflow entry point | — | Zoho CRM - Deal Trigger | ## How it works<br><br>This workflow automatically analyzes sales cycle performance for closed deals (won/lost) from Zoho CRM. It compares each deal to historical averages, identifies bottlenecks, calculates optimal timing patterns, and suggests process improvements. Results are sent to Slack and logged to Google Sheets for continuous optimization.<br><br>## Setup steps<br><br>1. Configure Zoho CRM credentials in the "Zoho CRM - Deal Trigger" node<br>2. Set up Google Sheets credentials and configure "Fetch Historical Averages" with your sheet ID and range<br>3. Configure "Log to Historical Sheet" to append new deals to your historical data<br>4. Update Slack channel in "Slack Notification" node (default: #sales-insights)<br>5. Ensure your Google Sheet has headers: Created_Time, Closed_Time, Stage, Deal_Name (or similar)<br>6. Optionally replace Manual Trigger with Schedule Trigger for automatic daily/hourly runs |
| Zoho CRM - Deal Trigger | Zoho CRM | Fetch deals from Zoho CRM | When clicking 'Execute workflow' | Filter Recent Deals | ## Data Collection<br><br>Fetches closed deals (won/lost) from Zoho CRM, filters to recent deals (last 7 days, max 10), and retrieves historical cycle data from Google Sheets for comparison and analysis. |
| Filter Recent Deals | Code | Filter recent deals and cap volume | Zoho CRM - Deal Trigger | Fetch Historical Averages | ## Data Collection<br><br>Fetches closed deals (won/lost) from Zoho CRM, filters to recent deals (last 7 days, max 10), and retrieves historical cycle data from Google Sheets for comparison and analysis. |
| Fetch Historical Averages | Google Sheets | Load historical data rows | Filter Recent Deals | Analyze Cycle | ## Data Collection<br><br>Fetches closed deals (won/lost) from Zoho CRM, filters to recent deals (last 7 days, max 10), and retrieves historical cycle data from Google Sheets for comparison and analysis. |
| Analyze Cycle | Code | Compute cycle stats, insights, suggestions | Fetch Historical Averages | AI Sentiment Analysis; AI Predictive Analytics | ## AI-Powered Intelligence Layer<br><br>This section adds advanced AI capabilities: sentiment analysis for emotional intelligence, predictive analytics for outcome forecasting, smart recommendations for process optimization, and data visualization for trend analysis. |
| AI Sentiment Analysis | OpenAI | GPT‑4 sentiment & risk inference | Analyze Cycle | AI Data Aggregation | ## AI-Powered Intelligence Layer<br><br>This section adds advanced AI capabilities: sentiment analysis for emotional intelligence, predictive analytics for outcome forecasting, smart recommendations for process optimization, and data visualization for trend analysis. |
| AI Predictive Analytics | OpenAI | GPT‑4 win probability & close-date prediction | Analyze Cycle | AI Data Aggregation | ## AI-Powered Intelligence Layer<br><br>This section adds advanced AI capabilities: sentiment analysis for emotional intelligence, predictive analytics for outcome forecasting, smart recommendations for process optimization, and data visualization for trend analysis. |
| AI Data Aggregation | Code | Merge AI outputs into one payload | AI Sentiment Analysis; AI Predictive Analytics | Advanced Data Visualization | ## AI-Powered Intelligence Layer<br><br>This section adds advanced AI capabilities: sentiment analysis for emotional intelligence, predictive analytics for outcome forecasting, smart recommendations for process optimization, and data visualization for trend analysis. |
| Advanced Data Visualization | Code | Build chart metadata & risk factors | AI Data Aggregation | AI Smart Recommendations | ## AI-Powered Intelligence Layer<br><br>This section adds advanced AI capabilities: sentiment analysis for emotional intelligence, predictive analytics for outcome forecasting, smart recommendations for process optimization, and data visualization for trend analysis. |
| AI Smart Recommendations | OpenAI | GPT‑4 prioritized recommendations | Advanced Data Visualization | Check Valid Data | ## AI-Powered Intelligence Layer<br><br>This section adds advanced AI capabilities: sentiment analysis for emotional intelligence, predictive analytics for outcome forecasting, smart recommendations for process optimization, and data visualization for trend analysis. |
| Check Valid Data | IF | Validate minimum data quality | AI Smart Recommendations | Slack Notification; Log to Historical Sheet | ## Notifications & Logging<br><br>Validates data quality, then sends formatted insights to Slack channel and automatically logs the analyzed deal to Google Sheets for building historical dataset over time. |
| Slack Notification | Slack | Post formatted results to Slack | Check Valid Data | — | ## Notifications & Logging<br><br>Validates data quality, then sends formatted insights to Slack channel and automatically logs the analyzed deal to Google Sheets for building historical dataset over time. |
| Log to Historical Sheet | Google Sheets | Append analyzed record to sheet | Check Valid Data | — | ## Notifications & Logging<br><br>Validates data quality, then sends formatted insights to Slack channel and automatically logs the analyzed deal to Google Sheets for building historical dataset over time. |
| Main Overview | Sticky Note | Documentation / operator notes | — | — | ## How it works<br><br>This workflow automatically analyzes sales cycle performance for closed deals (won/lost) from Zoho CRM. It compares each deal to historical averages, identifies bottlenecks, calculates optimal timing patterns, and suggests process improvements. Results are sent to Slack and logged to Google Sheets for continuous optimization.<br><br>## Setup steps<br><br>1. Configure Zoho CRM credentials in the "Zoho CRM - Deal Trigger" node<br>2. Set up Google Sheets credentials and configure "Fetch Historical Averages" with your sheet ID and range<br>3. Configure "Log to Historical Sheet" to append new deals to your historical data<br>4. Update Slack channel in "Slack Notification" node (default: #sales-insights)<br>5. Ensure your Google Sheet has headers: Created_Time, Closed_Time, Stage, Deal_Name (or similar)<br>6. Optionally replace Manual Trigger with Schedule Trigger for automatic daily/hourly runs |
| Section - Data Collection | Sticky Note | Documentation / section header | — | — | ## Data Collection<br><br>Fetches closed deals (won/lost) from Zoho CRM, filters to recent deals (last 7 days, max 10), and retrieves historical cycle data from Google Sheets for comparison and analysis. |
| Section - AI Intelligence | Sticky Note | Documentation / section header | — | — | ## AI-Powered Intelligence Layer<br><br>This section adds advanced AI capabilities: sentiment analysis for emotional intelligence, predictive analytics for outcome forecasting, smart recommendations for process optimization, and data visualization for trend analysis. |
| Section - Notifications & Logging | Sticky Note | Documentation / section header | — | — | ## Notifications & Logging<br><br>Validates data quality, then sends formatted insights to Slack channel and automatically logs the analyzed deal to Google Sheets for building historical dataset over time. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow** in n8n and name it: *Analyze and improve Zoho CRM sales cycles with GPT-4, Google Sheets and Slack*.

2) **Add Trigger**
   - Add node: **Manual Trigger**
   - Name: **When clicking 'Execute workflow'**

3) **Zoho CRM: Get deals**
   - Add node: **Zoho CRM**
   - Name: **Zoho CRM - Deal Trigger**
   - **Credentials:** Configure Zoho CRM OAuth/connection in n8n.
   - **Resource:** Deal
   - **Operation:** Get All
   - Connect: Manual Trigger → Zoho CRM - Deal Trigger

4) **Filter to recent deals (7 days, max 10)**
   - Add node: **Code**
   - Name: **Filter Recent Deals**
   - Paste logic that:
     - Computes `sevenDaysAgo`
     - Filters by `Closed_Time` else `Modified_Time`
     - Returns first 10
   - Connect: Zoho CRM - Deal Trigger → Filter Recent Deals

5) **Google Sheets: fetch historical rows**
   - Add node: **Google Sheets** (v3)
   - Name: **Fetch Historical Averages**
   - **Credentials:** Connect Google account with Sheets access.
   - Set:
     - **Document ID:** your historical spreadsheet
     - **Sheet name:** the tab containing historical deals
     - Configure the node to **read rows** (e.g., “Read” / “Get Many” depending on your n8n version UI).
   - Ensure your sheet headers include at least:
     - `Created_Time`, `Closed_Time`, `Stage`, `Deal_Name` (aliases are partially supported by the code)
   - Connect: Filter Recent Deals → Fetch Historical Averages

6) **Compute cycle analytics**
   - Add node: **Code**
   - Name: **Analyze Cycle**
   - Paste the “Analyze Cycle” JS logic (adapt if you want per-deal processing).
   - Connect: Fetch Historical Averages → Analyze Cycle

7) **AI sentiment**
   - Add node: **OpenAI**
   - Name: **AI Sentiment Analysis**
   - **Credentials:** OpenAI API key (or configured OpenAI credential in n8n).
   - **Model:** `gpt-4`
   - Prompt: sentiment/risk prompt using `{{ JSON.stringify($json.deal) }}`
   - Connect: Analyze Cycle → AI Sentiment Analysis

8) **AI predictive analytics**
   - Add node: **OpenAI**
   - Name: **AI Predictive Analytics**
   - **Model:** `gpt-4`
   - Prompt references historical context and current deal.
   - Connect: Analyze Cycle → AI Predictive Analytics

9) **Aggregate AI outputs**
   - Add node: **Code**
   - Name: **AI Data Aggregation**
   - Paste aggregation JS (merging main + AI outputs).
   - Connect:
     - AI Sentiment Analysis → AI Data Aggregation
     - AI Predictive Analytics → AI Data Aggregation
   - Note: if you want robust merging, consider inserting a **Merge** node configured to “Wait for both branches”, then aggregate deterministically.

10) **Visualization metadata**
   - Add node: **Code**
   - Name: **Advanced Data Visualization**
   - Paste visualization JS.
   - Connect: AI Data Aggregation → Advanced Data Visualization

11) **AI smart recommendations**
   - Add node: **OpenAI**
   - Name: **AI Smart Recommendations**
   - **Model:** `gpt-4`
   - Prompt: request 3–5 prioritized recommendations using the merged JSON.
   - Connect: Advanced Data Visualization → AI Smart Recommendations

12) **Validation gate**
   - Add node: **IF**
   - Name: **Check Valid Data**
   - Conditions (AND):
     - String: `{{ $json.deal && $json.deal.Deal_Name }}` **is not empty**
     - Number: `{{ $json.totalDays }}` **larger than** `0`
   - Connect: AI Smart Recommendations → Check Valid Data

13) **Slack notification**
   - Add node: **Slack**
   - Name: **Slack Notification**
   - **Credentials:** Slack OAuth/App with permission to post messages.
   - **Channel:** `#sales-insights` (or your channel)
   - **Text:** use the provided Slack template (with expressions).
   - Connect: Check Valid Data (true) → Slack Notification

14) **Log to Google Sheets**
   - Add node: **Google Sheets** (v3)
   - Name: **Log to Historical Sheet**
   - **Operation:** Append
   - Set **Document ID** and **Sheet name** to your logging destination.
   - Map fields explicitly (recommended) to avoid nested JSON issues (e.g., deal name, stage, created/closed times, totalDays, avgDays, recommendations text).
   - Connect: Check Valid Data (true) → Log to Historical Sheet

15) **(Optional) Replace manual trigger**
   - Swap Manual Trigger with **Schedule Trigger** (daily/hourly) to automate.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Ensure your Google Sheet has headers: Created_Time, Closed_Time, Stage, Deal_Name (or similar). | Required by “Analyze Cycle” normalization logic and sheet parsing. |
| Optionally replace Manual Trigger with Schedule Trigger for automatic daily/hourly runs. | Operational improvement for production runs. |
| Workflow computes “performanceScore” and “riskLevel” in Slack template, but these fields are not generated by current nodes. | Consider adding a Code node to calculate them or adjust Slack message. |
| Prompts reference `$json.historicalData`, but the workflow does not populate `historicalData`. | Consider adding a node to structure historical rows into `historicalData` or adjust prompts/visualization logic. |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.