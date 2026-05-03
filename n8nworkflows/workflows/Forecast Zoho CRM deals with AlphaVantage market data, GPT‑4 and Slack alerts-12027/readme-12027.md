Forecast Zoho CRM deals with AlphaVantage market data, GPT‑4 and Slack alerts

https://n8nworkflows.xyz/workflows/forecast-zoho-crm-deals-with-alphavantage-market-data--gpt-4-and-slack-alerts-12027


# Forecast Zoho CRM deals with AlphaVantage market data, GPT‑4 and Slack alerts

Disclaimer: Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## Forecast Zoho CRM deals with AlphaVantage market data, GPT‑4 and Slack alerts  
Workflow name (in JSON): **Zoho Deal Forecasting with External Market Factor**

---

# 1. Workflow Overview

This workflow runs on a schedule (weekly) to **forecast Zoho CRM deal outcomes** by combining:
- **Zoho deal pipeline data** (amount, probability, close date, stage, etc.)
- **External market signal** (AlphaVantage “GLOBAL_QUOTE” on SPY)
- **Computed seasonality factor** from historical close dates
- **AI evaluation (GPT‑4 Turbo)** to score each deal’s alignment with market conditions (match ratio, confidence, reason)

It then:
1) **Updates Zoho CRM deals** with computed forecast fields  
2) **Stores the enriched forecast record in Supabase** (for tracking and analytics)  
3) **Sends a Slack summary** per updated deal

### 1.1 Scheduling / Entry
- Weekly Cron trigger starts the workflow and fans out into (a) deals fetch and (b) market fetch.

### 1.2 Data Collection (Zoho + Market)
- Fetch all deals from Zoho.
- Fetch current market signal from AlphaVantage.

### 1.3 Forecast Metrics Computation
- Merge deals + market signal, then compute:
  - expected revenue (fallback logic)
  - market factor
  - seasonality factor derived from historical close dates
  - adjusted forecast

### 1.4 AI Scoring (Deal vs Market)
- Send deal + computed metrics to GPT‑4 Turbo.
- Parse model output into structured JSON.

### 1.5 Enrichment, Update, Storage, Notification
- Merge computed forecast metrics + AI fields.
- Update Zoho deal custom fields.
- Insert a record into Supabase.
- Send Slack summary message.

---

# 2. Block-by-Block Analysis

## Block A — Scheduling / Entry

**Overview:** Triggers the workflow automatically and initiates parallel fetching of deals and market data.  
**Nodes involved:** Weekly Trigger1, Sticky Note6

### Node: Weekly Trigger1
- **Type / role:** `Cron` node; scheduled trigger (entry point).
- **Key configuration:** Runs at **02:00** (timezone depends on n8n instance settings).
- **Connections:**
  - **Outputs →** Fetch open Deals1, Get Market Signal1 (fan-out)
- **Failure/edge cases:**
  - Instance timezone mismatch can cause runs at unexpected local time.
  - If workflow is inactive (`active: false`), it will not run.

### Node: Sticky Note6
- **Type / role:** Sticky Note (documentation only).
- **Content:** “Runs this workflow automatically every week.”
- **Applies to:** Weekly Trigger1 (visually contextual)

---

## Block B — Data Collection (Zoho Deals + AlphaVantage Market Signal)

**Overview:** Retrieves all deals from Zoho and pulls live market data from AlphaVantage for use as a multiplier signal.  
**Nodes involved:** Fetch open Deals1, Get Market Signal1, Combine Deal & Market Info1, Sticky Note7, Sticky Note8

### Node: Fetch open Deals1
- **Type / role:** `Zoho CRM` node; fetches deals.
- **Configuration choices:**
  - **Resource:** deal
  - **Operation:** getAll
  - **Options:** none configured (so it will use defaults for pagination/filters depending on node defaults)
- **Connections:**
  - **Input ←** Weekly Trigger1
  - **Output →** Combine Deal & Market Info1 (input 0)
- **Credentials:** Zoho OAuth2 (`Zoho account 8`)
- **Failure/edge cases:**
  - OAuth token expiration / scope issues.
  - “getAll” may pull **closed + historical** deals unless filters are applied; the sticky note says “active deals”, but the node as configured does not enforce that.
  - Large datasets can cause pagination/timeouts or performance issues.

### Node: Get Market Signal1
- **Type / role:** `HTTP Request`; fetch market quote from AlphaVantage.
- **Configuration choices:**
  - URL: `https://www.alphavantage.co/query`
  - Query params:
    - `function=GLOBAL_QUOTE`
    - `symbol=SPY`
    - `apikey={{Your_Alphavantage_API_key}}` (placeholder)
- **Connections:**
  - **Input ←** Weekly Trigger1
  - **Output →** Combine Deal & Market Info1 (input 1)
- **Failure/edge cases:**
  - The API key is a placeholder; if not replaced, request fails.
  - AlphaVantage rate limits are strict; may return throttle messages (HTTP 200 but error payload).
  - The workflow expects a `marketSignal` field later, but this HTTP node **does not include parsing logic** to map API output to `marketSignal`. Without an extra parsing step, `marketSignal` will default to **1** in later calculations (see Generate Forecast Metrics1).

### Node: Combine Deal & Market Info1
- **Type / role:** `Merge` node; combines Zoho deals stream with the market data stream.
- **Configuration choices:** No explicit mode shown; in n8n Merge defaults can be “Combine” behavior depending on version. Given the subsequent Function node uses `$input.all()`, it relies on both streams’ items being present in the incoming data set.
- **Connections:**
  - **Inputs ←** Fetch open Deals1 (index 0), Get Market Signal1 (index 1)
  - **Output →** Generate Forecast Metrics1
- **Failure/edge cases:**
  - If the merge mode does not produce a dataset containing **both** deals and market item(s), the Function node may not find the market signal item.
  - If one branch fails (market or deals), merge behavior depends on configuration; could output only one side or nothing.

### Node: Sticky Note7
- **Type / role:** Sticky Note (documentation only)
- **Content:** “Fetches all active deals from Zoho that are still in early/mid pipeline stages.”
- **Important mismatch:** The Zoho node is not configured with filters, so this statement may be aspirational rather than true.

### Node: Sticky Note8
- **Type / role:** Sticky Note (documentation only)
- **Content:** “Fetches real-time market trend data (SPY index) from AlphaVantage.”

---

## Block C — Forecast Metrics Computation (Seasonality + Market Adjustment)

**Overview:** Calculates expected revenue and a market/seasonality-adjusted forecast for each valid deal.  
**Nodes involved:** Generate Forecast Metrics1, Sticky Note (Forecast Calculation Process)

### Node: Generate Forecast Metrics1
- **Type / role:** `Function` node; custom JavaScript forecasting logic.
- **Key logic & variables:**
  - `allItems = $input.all()` reads all incoming items from the Merge.
  - `deals = allItems.map(...).filter(d => d.Amount && d.Probability)`
    - Filters only deals that have Amount and Probability (truthy).
  - Market signal detection:
    - `marketItem = allItems.find(item => item.json.marketSignal !== undefined)`
    - `marketSignal = marketItem?.json?.marketSignal || 1`
    - **Implication:** If no `marketSignal` field exists (likely unless you add a parsing step), it uses `1` (neutral).
  - Seasonality:
    - Uses `Close_Date` to compute monthly pipeline-weighted totals.
    - `seasonalityFactors[month] = total / avgRevenue` (defaults to 1 if avgRevenue is 0)
    - `seasonalFactor = seasonalityFactors[currentMonth] || 1`
  - Deal-level metrics:
    - `pipelineWeighted = amount * probability`
    - `expectedRevenue = d.Expected_Revenue ?? pipelineWeighted`
    - `adjustedForecast = pipelineWeighted * marketSignal * seasonalFactor`
- **Output:** One item per valid deal with:
  - `id`, `Deal_Name`, `Stage`, `Amount`, `Probability` (as fraction, not percent), `Expected_Revenue`,
  - `Market_Signal`, `Seasonal_Factor`, `Market_Seasonal_Adjusted_Forecast`
- **Connections:**
  - **Input ←** Combine Deal & Market Info1
  - **Output →** Deal Match Evaluator
- **Failure/edge cases:**
  - If Zoho returns `Probability` as a percentage string (e.g., `"60"`), conversion works; if missing or non-numeric, `safeNumber` falls back to 0.
  - `new Date(d.Close_Date)` can yield invalid dates if Zoho format differs; invalid dates lead to `getMonth()` being `NaN`, which would break array indexing (currently not guarded).
  - Seasonality uses all deals with `Close_Date` as “historical”; it does not distinguish closed-won vs open or date ranges.

### Node: Sticky Note (Forecast Calculation Process)
- **Type / role:** Sticky Note (documentation only)
- **Content:** Describes the forecast calculation and AI alignment scoring process for this mid-workflow section.
- **Applies to:** Combine Deal & Market Info1, Generate Forecast Metrics1, Deal Match Evaluator, Parse AI Output, Merge Forecast & AI data (context)

---

## Block D — AI Scoring and Parsing

**Overview:** Uses GPT‑4 Turbo to score each deal’s alignment with the market signal and returns structured JSON, then parses it safely.  
**Nodes involved:** Deal Match Evaluator, Parse AI Output

### Node: Deal Match Evaluator
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi` (OpenAI via LangChain); LLM call per deal.
- **Configuration choices:**
  - **Model:** `gpt-4-turbo`
  - Prompt instructs:
    - compute `match_ratio` (0–100), `confidence` (LOW/MEDIUM/HIGH), `reason` (one line)
    - respond strictly as JSON with keys: `match_ratio`, `confidence`, `reason`
  - Uses dynamic deal fields via expressions:
    - `{{ $json.Amount }}`, `{{ $json.Stage }}`, `{{ $json.Probability }}`, etc.
- **Connections:**
  - **Input ←** Generate Forecast Metrics1
  - **Output →** Parse AI Output
- **Credentials:** OpenAI (`OpenAi account 17`)
- **Failure/edge cases:**
  - Model may return JSON wrapped in Markdown fences; parsing node handles this.
  - Model may still output non-JSON or missing keys; parsing node sets fallbacks.
  - Token/usage limits, latency, or transient 429/5xx errors.

### Node: Parse AI Output
- **Type / role:** `Code` node; parses the LLM response into structured fields.
- **Key logic:**
  - Reads text from: `item.json.output?.[0]?.content?.[0]?.text`
    - **Version-specific risk:** This path is tied to the LangChain OpenAI node’s output schema; schema changes can break parsing.
  - Strips ```json fences and parses JSON.
  - On parse failure returns:
    - `match_ratio: null`
    - `confidence: "UNKNOWN"`
    - `reason: "Failed to parse AI response"`
- **Connections:**
  - **Input ←** Deal Match Evaluator
  - **Output →** Merge Forecast & AI data
- **Failure/edge cases:**
  - If the OpenAI node returns a different structure, `rawText` becomes empty and parse fails (falls back to UNKNOWN).
  - No validation that `match_ratio` is numeric 0–100.

---

## Block E — Enrichment + Update + Store + Notify

**Overview:** Combines computed forecast metrics with AI outputs, updates the Zoho deal custom fields, stores a record in Supabase, then posts a Slack message.  
**Nodes involved:** Merge Forecast & AI data, Update Deal Forecast1, Store Forecast, Send Forecast Summary1, Sticky Note1, Sticky Note2

### Node: Merge Forecast & AI data
- **Type / role:** `Set` node; constructs a single unified JSON object per deal containing both forecast fields and AI fields.
- **Configuration choices:**
  - Explicit field mapping:
    - Forecast fields pulled from `$('Generate Forecast Metrics1').item.json...`
    - AI fields pulled from current input `$json...` (output of Parse AI Output)
  - Output includes:
    - `id`, `Deal_Name`, `Stage`, `Amount`, `Probability`, `Expected_Revenue`
    - `Market_Signal`, `Seasonal_Factor`, `Market_Seasonal_Adjusted_Forecast`
    - `match_ratio`, `confidence`, `reason`
- **Connections:**
  - **Input ←** Parse AI Output
  - **Outputs →** Update Deal Forecast1, Store Forecast (fan-out)
- **Failure/edge cases:**
  - Uses `$('Generate Forecast Metrics1').item` referencing “paired item” behavior. If item pairing is misaligned (e.g., batch differences), fields can mismatch deals.
  - If Generate Forecast Metrics1 didn’t output a corresponding item, expressions may evaluate to null/empty.

### Node: Update Deal Forecast1
- **Type / role:** `Zoho CRM` node; updates each deal with forecast + AI-related fields.
- **Configuration choices:**
  - **Resource:** deal
  - **Operation:** update
  - **dealId:** `={{ $json.id }}`
  - Updates:
    - `Amount` to `$json.Amount`
    - Custom fields:
      - `Market_Signal` ← `$json.Market_Signal`
      - `Seasonal_Factor` ← `$json.Seasonal_Factor`
      - `Market_Signal_Adjust_forecast` ← `$json.Market_Seasonal_Adjusted_Forecast`
      - `Revenue` ← `$json.Expected_Revenue`
  - **Important:** The JSON uses `fieldId` values like `"Market_Signal"`, which may need to be Zoho **API names** or internal field identifiers depending on node expectations.
- **Connections:**
  - **Input ←** Merge Forecast & AI data
  - **Output →** Send Forecast Summary1
- **Credentials:** Zoho OAuth2 (`Zoho account 8`)
- **Failure/edge cases:**
  - Field mapping errors if custom fields do not exist or API names are wrong.
  - Permissions: user/token may not have update rights on Deals.
  - Zoho may reject numeric formats (e.g., too many decimals) depending on field type.

### Node: Store Forecast
- **Type / role:** `Supabase` node; persists enriched results into a database table.
- **Configuration choices:**
  - Table: `deal_forecast`
  - Inserts/updates fields (operation not explicitly shown, typical node defaults apply—ensure it’s set to “Insert” or equivalent in UI):
    - `id`, `deal_name`, `stage`, `amount`, `match_ratio`, `confidence`, `reason`
- **Connections:**
  - **Input ←** Merge Forecast & AI data
- **Credentials:** Supabase (`Supabase account 5`)
- **Failure/edge cases:**
  - Table/column mismatch, constraints, or type mismatch (e.g., `match_ratio` null).
  - If `id` is primary key, repeated weekly inserts may fail unless using upsert.

### Node: Send Forecast Summary1
- **Type / role:** `Slack` node; posts a summary message to a channel.
- **Configuration choices:**
  - Posts to channel ID `C09S57E2JQ2` (cached name: `n8n`)
  - Message text references:
    - `{{ $('Merge Forecast & AI data').item.json.Deal_Name }}` etc.
  - “Include link to workflow”: false
- **Connections:**
  - **Input ←** Update Deal Forecast1
  - **Output:** none
- **Credentials:** Slack (`Slack account 15`)
- **Failure/edge cases:**
  - Slack auth scopes: missing `chat:write` or channel access.
  - If channelId changes or bot not in channel, message fails.
  - Same “paired item” risk when referencing `$('Merge Forecast & AI data').item`.

### Node: Sticky Note1
- **Type / role:** Sticky Note (documentation only)
- **Content:** “This node saves all updated deal forecasts and AI insights into a database.”
- **Applies to:** Store Forecast

### Node: Sticky Note2
- **Type / role:** Sticky Note (documentation only)
- **Content:** Describes the Zoho update + Slack share section.
- **Applies to:** Update Deal Forecast1, Send Forecast Summary1, Store Forecast (context)

### Node: Sticky Note5
- **Type / role:** Sticky Note (documentation only)
- **Content:** High-level “How it works” + setup checklist (custom fields, credentials, field IDs, Slack mapping, testing).
- **Applies to:** Whole workflow (global note)

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Weekly Trigger1 | Cron | Weekly scheduled entry point | — | Fetch open Deals1; Get Market Signal1 | Runs this workflow automatically every week. |
| Fetch open Deals1 | Zoho CRM | Retrieve deals from Zoho | Weekly Trigger1 | Combine Deal & Market Info1 | Fetches all active deals from Zoho that are still in early/mid pipeline stages. |
| Get Market Signal1 | HTTP Request | Pull external market quote (SPY) | Weekly Trigger1 | Combine Deal & Market Info1 | Fetches real-time market trend data (SPY index) from AlphaVantage. |
| Combine Deal & Market Info1 | Merge | Combine Zoho deals stream with market stream | Fetch open Deals1; Get Market Signal1 | Generate Forecast Metrics1 | ## Forecast Calculation Process<br>This part of the workflow collects all active deals from Zoho and real-time market signals. It calculates the expected revenue using deal amount, probability, seasonality, and market trends. An AI node then evaluates each deal’s alignment with market conditions, giving a match ratio, confidence level, and reasoning. Finally, all metrics and AI insights are merged into one complete dataset ready for updating or storing. |
| Generate Forecast Metrics1 | Function | Compute expected revenue, seasonality, adjusted forecast | Combine Deal & Market Info1 | Deal Match Evaluator | ## Forecast Calculation Process<br>This part of the workflow collects all active deals from Zoho and real-time market signals... |
| Deal Match Evaluator | OpenAI (LangChain) | Score deal vs market; return JSON fields | Generate Forecast Metrics1 | Parse AI Output | ## Forecast Calculation Process<br>This part of the workflow collects all active deals from Zoho and real-time market signals... |
| Parse AI Output | Code | Parse LLM response into match_ratio/confidence/reason | Deal Match Evaluator | Merge Forecast & AI data | ## Forecast Calculation Process<br>This part of the workflow collects all active deals from Zoho and real-time market signals... |
| Merge Forecast & AI data | Set | Build unified record combining forecast + AI | Parse AI Output | Update Deal Forecast1; Store Forecast | ## Forecast Calculation Process<br>This part of the workflow collects all active deals from Zoho and real-time market signals... |
| Update Deal Forecast1 | Zoho CRM | Update deal with forecast + factors | Merge Forecast & AI data | Send Forecast Summary1 | ## Update & Share Forecast<br>This part of the workflow updates each deal in Zoho CRM with the new forecast values... |
| Store Forecast | Supabase | Store forecast + AI insight for tracking | Merge Forecast & AI data | — | This node saves all updated deal forecasts and AI insights into a database.<br>## Update & Share Forecast<br>This part of the workflow updates each deal in Zoho CRM with the new forecast values... |
| Send Forecast Summary1 | Slack | Notify team in Slack per deal | Update Deal Forecast1 | — | ## Update & Share Forecast<br>This part of the workflow updates each deal in Zoho CRM with the new forecast values... |
| Sticky Note5 | Sticky Note | Global documentation | — | — | ## How it works<br>… (full note content) |
| Sticky Note6 | Sticky Note | Documentation for trigger | — | — | Runs this workflow automatically every week. |
| Sticky Note7 | Sticky Note | Documentation for Zoho fetch | — | — | Fetches all active deals from Zoho that are still in early/mid pipeline stages. |
| Sticky Note8 | Sticky Note | Documentation for market fetch | — | — | Fetches real-time market trend data (SPY index) from AlphaVantage. |
| Sticky Note | Sticky Note | Documentation for forecast/AI section | — | — | ## Forecast Calculation Process<br>… (full note content) |
| Sticky Note1 | Sticky Note | Documentation for Supabase storage | — | — | This node saves all updated deal forecasts and AI insights into a database. |
| Sticky Note2 | Sticky Note | Documentation for update/notify section | — | — | ## Update & Share Forecast<br>… (full note content) |

---

# 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
   - Name it: *Zoho Deal Forecasting with External Market Factor* (or your preferred name).
   - Ensure workflow execution order is default (`v1`) unless you have a reason to change.

2) **Add the trigger**
   - Add **Cron** node → name it **Weekly Trigger1**
   - Set schedule to weekly (or daily) and set **hour = 2** (02:00).

3) **Add Zoho deal fetch**
   - Add **Zoho CRM** node → name **Fetch open Deals1**
   - Credentials: create/select **Zoho OAuth2**
     - Configure client/app in Zoho, then authorize in n8n.
   - Resource: **Deal**
   - Operation: **Get All**
   - (Recommended) Add filters/options to truly fetch only open pipeline deals (the provided workflow does not enforce this).

4) **Add AlphaVantage request**
   - Add **HTTP Request** node → name **Get Market Signal1**
   - Method: GET
   - URL: `https://www.alphavantage.co/query`
   - Query parameters:
     - `function` = `GLOBAL_QUOTE`
     - `symbol` = `SPY`
     - `apikey` = your AlphaVantage key (store as n8n credential/variable; do not leave placeholder)

   **Important (to match the Function node’s expectation):**
   - Add an extra **Set** or **Code** node after this HTTP node to map the API response into a numeric field called **`marketSignal`** (because later logic searches `item.json.marketSignal`).  
   - Example behavior to implement: parse quote change/percent change into a multiplier (e.g., 0.95–1.05) and output `{ marketSignal: <number> }`.

5) **Add Merge node to combine streams**
   - Add **Merge** node → name **Combine Deal & Market Info1**
   - Connect:
     - Weekly Trigger1 → Fetch open Deals1
     - Weekly Trigger1 → Get Market Signal1
     - Fetch open Deals1 → Combine Deal & Market Info1 (Input 1 / index 0)
     - Get Market Signal1 (or its parsing node) → Combine Deal & Market Info1 (Input 2 / index 1)
   - Configure merge mode so the downstream Function node can read **both** deals and the market item from `$input.all()`.

6) **Add forecast computation**
   - Add **Function** node → name **Generate Forecast Metrics1**
   - Paste the forecasting JS logic (seasonality computation + adjusted forecast).
   - Connect: Combine Deal & Market Info1 → Generate Forecast Metrics1

7) **Add OpenAI scoring**
   - Add **OpenAI (LangChain)** node → name **Deal Match Evaluator**
   - Credentials: create/select OpenAI credential (API key).
   - Model: **gpt-4-turbo**
   - Prompt: instruct to output strict JSON with `match_ratio`, `confidence`, `reason`, using the deal fields.
   - Connect: Generate Forecast Metrics1 → Deal Match Evaluator

8) **Add parsing step**
   - Add **Code** node → name **Parse AI Output**
   - Implement:
     - Extract model text from the OpenAI node output
     - Strip markdown fences
     - `JSON.parse` with try/catch fallback
   - Connect: Deal Match Evaluator → Parse AI Output

9) **Add “Set” node to unify forecast + AI fields**
   - Add **Set** node → name **Merge Forecast & AI data**
   - Add fields:
     - From Generate Forecast Metrics1 item: `id`, `Deal_Name`, `Stage`, `Amount`, `Probability`, `Expected_Revenue`, `Market_Signal`, `Seasonal_Factor`, `Market_Seasonal_Adjusted_Forecast`
     - From Parse AI Output: `match_ratio`, `confidence`, `reason`
   - Connect: Parse AI Output → Merge Forecast & AI data  
   - Ensure item pairing is consistent (test with multiple deals).

10) **Update Zoho deal**
   - Add **Zoho CRM** node → name **Update Deal Forecast1**
   - Credentials: same Zoho OAuth2
   - Resource: **Deal**
   - Operation: **Update**
   - Deal ID: `{{$json.id}}`
   - Map fields:
     - Amount = `{{$json.Amount}}`
     - Custom fields (create these in Zoho first):
       - Market signal field (API name) ← `{{$json.Market_Signal}}`
       - Seasonal factor field ← `{{$json.Seasonal_Factor}}`
       - Adjusted forecast field ← `{{$json.Market_Seasonal_Adjusted_Forecast}}`
       - Revenue field/API name ← `{{$json.Expected_Revenue}}`
   - Connect: Merge Forecast & AI data → Update Deal Forecast1

11) **Store record in Supabase**
   - Add **Supabase** node → name **Store Forecast**
   - Credentials: Supabase URL + service role key (or appropriate key with insert permissions)
   - Table: `deal_forecast`
   - Map columns:
     - `id`, `deal_name`, `stage`, `amount`, `match_ratio`, `confidence`, `reason`
   - Connect: Merge Forecast & AI data → Store Forecast
   - Consider enabling **Upsert** if you want one row per deal updated weekly.

12) **Send Slack notification**
   - Add **Slack** node → name **Send Forecast Summary1**
   - Credentials: Slack OAuth or bot token with `chat:write`
   - Operation: Post message
   - Channel: choose the target channel
   - Message text: include deal name, stage, amount, market signal, seasonal factor, AI fields, adjusted forecast (as in the workflow).
   - Connect: Update Deal Forecast1 → Send Forecast Summary1

13) **Add sticky notes (optional but helpful)**
   - Add notes for: global description, trigger, Zoho fetch, market fetch, forecast block, update/share block, storage block.

14) **Test and activate**
   - Execute manually with a small set of deals first.
   - Validate:
     - marketSignal is actually populated (not always defaulting to 1)
     - Zoho custom field API names are correct
     - Supabase inserts succeed
     - Slack posts appear
   - Activate the workflow.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow automatically updates your Zoho CRM deals every week with AI-enhanced forecasts… then updates Zoho, stores results in Supabase, and sends Slack summaries. | Sticky note “How it works” (global) |
| Setup steps: create custom fields in Zoho; add Zoho OAuth; configure deal fetch; add market API key; replace placeholder Zoho field IDs with API names; map Slack channel; test then activate cron. | Sticky note “Setup steps” (global) |
| **Implementation caution:** The Function node expects `item.json.marketSignal`, but the AlphaVantage HTTP node as provided does not map API output to `marketSignal`. Add a parsing/mapping node to produce a numeric `marketSignal`. | Derived from node-to-node dependency |

If you want, I can propose a robust AlphaVantage parsing step that converts SPY percent change into a stable multiplier (e.g., 0.97–1.03) and outputs the exact `marketSignal` field this workflow expects.