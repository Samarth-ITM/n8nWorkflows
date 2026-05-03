Generate AI stock buy/no-buy signals using GPT‑4.1, Google Sheets and EODHD

https://n8nworkflows.xyz/workflows/generate-ai-stock-buy-no-buy-signals-using-gpt-4-1--google-sheets-and-eodhd-12333


# Generate AI stock buy/no-buy signals using GPT‑4.1, Google Sheets and EODHD

## 1. Workflow Overview

**Purpose:** This workflow reads stock tickers from Google Sheets, retrieves **fundamental** and **daily OHLCV** market data from **EODHD**, computes **technical indicators** plus a **growth potential score**, asks an **OpenAI (LangChain) agent** to produce a structured **BUY/WATCH/SELL + trade plan** in strict JSON, then appends the resulting signal back into Google Sheets.

**Primary use cases**
- Batch generation of consistent buy/no-buy signals for a watchlist
- Feeding dashboards or downstream systems with structured signals (entry/SL/TP + rationale)
- Re-running analysis periodically with updated market data

### 1.1 Input Reception (Sheet → items)
- Manual trigger starts the run.
- Google Sheets provides a list of tickers (one per row).

### 1.2 Per‑Ticker Loop & Data Collection (EODHD)
- Each ticker is processed in a loop (batching).
- Fetch fundamentals first, then fetch OHLCV using the code returned in fundamentals.

### 1.3 Transform, Compute Indicators & Merge
- OHLC data is aggregated and sorted.
- Fundamentals and OHLC are merged into a single stream.
- A normalization/compute step produces:
  - RSI, SMA20/50/200, volatility, support/resistance, returns
  - growth_potential_score (0–100)

### 1.4 AI Decisioning (JSON schema enforced by prompt)
- A LangChain Agent node calls an OpenAI chat model.
- Agent must return **only valid JSON** matching the specified schema.

### 1.5 Output Parsing & Persistence (Append to Sheet)
- The AI JSON string is parsed to an object.
- Output is reshaped to match Google Sheets columns.
- A new row is appended; workflow loops until all tickers are processed.

---

## 2. Block-by-Block Analysis

### Block A — Input & Iteration Control
**Overview:** Starts execution manually, loads tickers from Google Sheets, and iterates through them using Split In Batches.  
**Nodes involved:**  
- When clicking ‘Execute workflow’
- Get row(s) in sheet
- Loop Over Items

#### Node: **When clicking ‘Execute workflow’**
- **Type / role:** Manual Trigger; entry point for interactive runs.
- **Config:** No parameters.
- **Outputs to:** `Get row(s) in sheet`
- **Edge cases:** None (only runs when manually executed).

#### Node: **Get row(s) in sheet**
- **Type / role:** Google Sheets node; reads input tickers.
- **Config (interpreted):**
  - Operation: *Get rows* (implied by node name)
  - Document: `YOUR_GOOGLE_SHEET_ID`
  - Sheet: `Sheet1`
- **Expected input columns:** At minimum a column named `ticker` (per sticky note).
- **Outputs to:** `Loop Over Items`
- **Failure modes / edge cases:**
  - OAuth/Service Account auth failure
  - Wrong document ID / sheet name
  - Missing `ticker` column → later HTTP nodes will build invalid URLs

#### Node: **Loop Over Items** (Split In Batches)
- **Type / role:** Batch iterator; processes rows incrementally and enables looping.
- **Config:** Default options (batch size not explicitly set in JSON; n8n defaults apply).
- **Connections:**
  - **Main output (index 1) →** `Fetch stock fundamentals (EODHD)` (note: in this workflow the second output is used for processing)
  - **Main output (index 0) ←** `Append row in sheet` (used to request the next batch)
- **Edge cases / behavior notes:**
  - If batch size is too large, API rate limits may occur.
  - If it outputs on an unexpected port due to configuration changes, the workflow can stall (because the processing path currently starts from output index **1**).

---

### Block B — Data Sources (EODHD Fundamentals → OHLC)
**Overview:** For each ticker, fetches fundamentals, then fetches daily OHLCV series using the fundamentals’ code.  
**Nodes involved:**  
- Fetch stock fundamentals (EODHD)
- Fetch OHLC price data (EODHD)

#### Node: **Fetch stock fundamentals (EODHD)**
- **Type / role:** HTTP Request; retrieves fundamental data.
- **Config (interpreted):**
  - Method: GET (default)
  - URL: `https://eodhd.com/api/fundamentals/{{ $json.ticker }}.US`
  - Query params:
    - `filter=General,Highlights,Valuation,Technicals` (limits payload)
    - `api_token=YOUR_EODHD_API_KEY`
    - `fmt=json`
- **Inputs:** `ticker` from the current Google Sheets row.
- **Outputs to:**
  - `Fetch OHLC price data (EODHD)`
  - `Merge` (input 2 / index 1)
- **Failure modes / edge cases:**
  - 401/403 if token invalid
  - 404 if ticker not found or wrong market suffix
  - API payload missing `General.Code` → breaks OHLC request URL templating

#### Node: **Fetch OHLC price data (EODHD)**
- **Type / role:** HTTP Request; retrieves historical daily candles.
- **Config (interpreted):**
  - URL: `https://eodhd.com/api/eod/{{ $json.General.Code }}.US`
  - Query params:
    - `from=2023-01-01`, `to=2025-12-30`
    - `period=d`
    - `fmt=json`
    - `api_token=YOUR_EODHD_API_KEY`
- **Special setting:** `alwaysOutputData: true`  
  Ensures node continues even if request errors (but data may be empty or error-shaped).
- **Inputs:** Requires fundamentals output to contain `General.Code`.
- **Outputs to:** `Compute indicators and growth score`
- **Failure modes / edge cases:**
  - If EOD returns an error object/string instead of an array, downstream Code nodes may fail or return “Not enough OHLC data”.
  - Large date ranges can increase response size and execution time.

---

### Block C — Transform & Merge (OHLC aggregation + stream merge)
**Overview:** Aggregates candle items into a single OHLC array, then merges OHLC bundle with fundamentals into one combined stream for normalization.  
**Nodes involved:**  
- Compute indicators and growth score
- Merge

#### Node: **Compute indicators and growth score**
- **Type / role:** Code (JavaScript); aggregates OHLC candles into a single item.
- **Config (interpreted):**
  - Collects all incoming `items[i].json` into `arr`
  - Sorts by `date` ascending
  - Outputs one item: `{ ohlc: arr, ohlc_len: arr.length }`
- **Inputs:** Assumes incoming items are a list of OHLC candles (each with `date`, `open/high/low/close` etc.).
- **Outputs to:** `Merge` (input 1 / index 0)
- **Failure modes / edge cases:**
  - If OHLC response is not an array of candle objects, `arr` may be meaningless (missing `date`), later normalization will fail.
  - No candles → `ohlc_len = 0`, later “Not enough OHLC data”.

#### Node: **Merge**
- **Type / role:** Merge node; combines two streams:
  - Input 1: aggregated OHLC object
  - Input 2: fundamentals object
- **Config:** Default merge settings (no explicit mode shown). In practice, this often means pairing items by index.
- **Inputs:**
  - From `Compute indicators and growth score`
  - From `Fetch stock fundamentals (EODHD)`
- **Outputs to:** `Normalize OHLC data`
- **Failure modes / edge cases:**
  - If item counts don’t match or mode is not suitable, the merged output may not contain both `ohlc` and `General/Highlights`.
  - The downstream normalization node explicitly checks for fundamentals and candles and returns structured errors if missing.

---

### Block D — Normalize, Compute Indicators, Growth Score
**Overview:** Converts merged raw data into a clean unified object with fundamentals, computed technicals, and a growth potential score.  
**Nodes involved:**  
- Normalize OHLC data

#### Node: **Normalize OHLC data**
- **Type / role:** Code (JavaScript, ES5-safe); core transformation and indicator computation.
- **Key outputs produced:**
  - `fundamentals`: parsed fields from EODHD fundamentals (PE, forward PE, EPS, growth YOY, beta, moving averages, etc.)
  - `technical`: computed metrics from OHLC candles:
    - `last_close`, `return_30d_pct`, `return_90d_pct`
    - `volatility_ann_pct` (annualized)
    - `rsi_14`
    - `sma_20`, `sma_50`, `sma_200`
    - `support_30d` (min low last 30), `resistance_30d` (max high last 30)
  - `growth_potential_score`: 0–100 heuristic score combining growth, valuation, trend, RSI, volatility
- **Important internal checks:**
  - Finds fundamentals by searching for an item with `General` and `Highlights`
  - Finds candles either from `{ ohlc: [...] }` or direct candle items
  - Requires at least **60** closes; otherwise returns an error object
- **Outputs to:** `Generate AI stock analysis`
- **Failure modes / edge cases:**
  - Missing fundamentals/candles → returns `{ error: ... }` item (workflow continues but AI will receive error-shaped input unless you branch)
  - JSON shape drift from EODHD fields can yield many `null` values
  - If candles contain strings/non-numbers, `toNum` converts invalids to `null` which may reduce usable data

---

### Block E — AI Processing (LangChain Agent + OpenAI model)
**Overview:** Sends the unified dataset to an AI agent with a strict schema prompt to produce structured trade decision JSON.  
**Nodes involved:**  
- OpenAI Chat Model
- Generate AI stock analysis

#### Node: **OpenAI Chat Model**
- **Type / role:** LangChain Chat Model wrapper (`lmChatOpenAi`); provides the model to the agent.
- **Config (interpreted):**
  - Model: `gpt-4.1-nano`
  - Options: default
- **Connections:** Provides language model to `Generate AI stock analysis` via `ai_languageModel`.
- **Failure modes / edge cases:**
  - Missing/invalid OpenAI credentials
  - Model not available in the account/region
  - Rate limits or timeouts on large payloads (OHLC arrays can be large if accidentally included—this workflow sends only normalized metrics, not full OHLC)

#### Node: **Generate AI stock analysis**
- **Type / role:** LangChain Agent (`@n8n/n8n-nodes-langchain.agent`); executes the prompt and enforces schema via instructions.
- **Config (interpreted):**
  - Prompt includes `INPUT DATA: {{ JSON.stringify($json) }}`
  - Strong system message:
    - “Return ONLY valid JSON”
    - “Do NOT invent data”
    - Defines mandatory output schema with nested objects/arrays
    - Provides trade level logic (support/resistance/SL/TP)
- **Inputs:** The normalized object from `Normalize OHLC data`.
- **Outputs to:** `Code in JavaScript2`
- **Failure modes / edge cases:**
  - Model may still output non-JSON or partial JSON → downstream `JSON.parse` fails
  - If input contains `{error: ...}`, the agent may comply but return null-heavy output; consider adding an IF node before AI

---

### Block F — Parse, Prepare, and Write Output (Sheets append + loop)
**Overview:** Parses the agent’s JSON string, maps it into flat fields, appends a row to Google Sheets, then triggers the next batch.  
**Nodes involved:**  
- Code in JavaScript2
- Prepare data for Google Sheets
- Append row in sheet

#### Node: **Code in JavaScript2**
- **Type / role:** Code node; parses AI output string to JSON.
- **Config (interpreted):**
  - Reads `items[0].json.output`
  - If missing/non-string → returns `{ error: 'No output string to parse' }`
  - Else `JSON.parse(raw)` and returns parsed object
- **Outputs to:** `Prepare data for Google Sheets`
- **Failure modes / edge cases:**
  - Invalid JSON → node throws (execution error) because `JSON.parse` not wrapped in try/catch
  - Output might be under a different property than `.output` depending on agent node version/config; if so, parser returns “No output string to parse”

#### Node: **Prepare data for Google Sheets**
- **Type / role:** Code node; flattens nested AI JSON into sheet-friendly columns.
- **Config (interpreted):**
  - Joins arrays (`positives`, `negatives`, `thesis`, `notes`) with `" | "`
  - Extracts `decision.*`, `trade_plan.*`, `fundamental_assessment.*`, `key_metrics_used.*`
- **Outputs to:** `Append row in sheet`
- **Failure modes / edge cases:**
  - If AI output deviates from schema (e.g., `thesis` not array), join may fail or return empty
  - Missing fields become `''` or `null` (generally safe for Sheets)

#### Node: **Append row in sheet**
- **Type / role:** Google Sheets node; appends results back to sheet.
- **Config (interpreted):**
  - Operation: Append
  - Document: `YOUR_GOOGLE_SHEET_ID`
  - Sheet: `Sheet1` (despite sticky note mentioning a `Signals` tab—configuration currently uses `Sheet1`)
  - Column mapping (examples):
    - `ENTRY = {{$json.entry}}`
    - `ticker = {{$json.ticker}}`
    - `negative = {{$json.negatives}}`
    - `ENTER(YES/NO) = {{$json.would_enter}}`
    - `fundamental_thesis = {{$json.thesis}}`
- **Outputs to:** `Loop Over Items` (to continue processing next ticker)
- **Failure modes / edge cases:**
  - Sheet schema mismatch (column names must exist exactly as configured)
  - Auth failure / permissions
  - Data type issues (numbers written as strings depending on sheet formatting and node settings)

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking ‘Execute workflow’ | Manual Trigger | Manual entry point | — | Get row(s) in sheet | ## INPUT  \n![txt](https://ik.imagekit.io/agbb7sr41/eodhd_input.png) |
| Get row(s) in sheet | Google Sheets | Read tickers list | When clicking ‘Execute workflow’ | Loop Over Items | ## INPUT  \n![txt](https://ik.imagekit.io/agbb7sr41/eodhd_input.png) |
| Loop Over Items | Split In Batches | Iterate per ticker and loop control | Get row(s) in sheet; Append row in sheet | Fetch stock fundamentals (EODHD) | ## Input (Data sources)  \n- Stock tickers come from **Google Sheets**\n- Market data is fetched via **API calls**:\n  - Fundamentals\n  - Price candles (OHLCV)\n\nThis step collects all the raw data needed for the analysis. |
| Fetch stock fundamentals (EODHD) | HTTP Request | Fetch fundamentals from EODHD | Loop Over Items | Fetch OHLC price data (EODHD); Merge | ## Input (Data sources)  \n- Stock tickers come from **Google Sheets**\n- Market data is fetched via **API calls**:\n  - Fundamentals\n  - Price candles (OHLCV)\n\nThis step collects all the raw data needed for the analysis. |
| Fetch OHLC price data (EODHD) | HTTP Request | Fetch OHLCV candles from EODHD | Fetch stock fundamentals (EODHD) | Compute indicators and growth score | ## Input (Data sources)  \n- Stock tickers come from **Google Sheets**\n- Market data is fetched via **API calls**:\n  - Fundamentals\n  - Price candles (OHLCV)\n\nThis step collects all the raw data needed for the analysis. |
| Compute indicators and growth score | Code | Aggregate/sort OHLC array for merging | Fetch OHLC price data (EODHD) | Merge | ##  Transform & Merge  \n\n- Raw market data is cleaned and normalized\n- Technical indicators are calculated\n- Fundamentals and price data are **merged into one dataset**\n\nAt the end of this step, each stock has a single, structured data object. |
| Merge | Merge | Combine fundamentals + OHLC bundle | Compute indicators and growth score; Fetch stock fundamentals (EODHD) | Normalize OHLC data | ##  Transform & Merge  \n\n- Raw market data is cleaned and normalized\n- Technical indicators are calculated\n- Fundamentals and price data are **merged into one dataset**\n\nAt the end of this step, each stock has a single, structured data object. |
| Normalize OHLC data | Code | Compute technical indicators + growth score; build unified payload | Merge | Generate AI stock analysis | ##  Transform & Merge  \n\n- Raw market data is cleaned and normalized\n- Technical indicators are calculated\n- Fundamentals and price data are **merged into one dataset**\n\nAt the end of this step, each stock has a single, structured data object. |
| OpenAI Chat Model | OpenAI (LangChain) | Supplies model to agent | — | Generate AI stock analysis (as language model) |  |
| Generate AI stock analysis | LangChain Agent | Produce strict-JSON signal/trade plan | Normalize OHLC data; OpenAI Chat Model | Code in JavaScript2 | ## AI Output\n- The merged data is sent to the **AI system**\n- The AI evaluates fundamentals, technicals, and risk\n- The final output includes:\n  - BUY / WATCH / SELL signal\n  - Entry, Stop Loss, Take Profit\n  - Investment score and rationale\n\nResults are ready to use or store in Google Sheets. |
| Code in JavaScript2 | Code | Parse AI JSON string to object | Generate AI stock analysis | Prepare data for Google Sheets | ## AI Output\n- The merged data is sent to the **AI system**\n- The AI evaluates fundamentals, technicals, and risk\n- The final output includes:\n  - BUY / WATCH / SELL signal\n  - Entry, Stop Loss, Take Profit\n  - Investment score and rationale\n\nResults are ready to use or store in Google Sheets. |
| Prepare data for Google Sheets | Code | Flatten AI object into sheet columns | Code in JavaScript2 | Append row in sheet | ## OUTPUT  \n![txt](https://ik.imagekit.io/agbb7sr41/eodhd_ouput.png) |
| Append row in sheet | Google Sheets | Append final signal row | Prepare data for Google Sheets | Loop Over Items | ## OUTPUT  \n![txt](https://ik.imagekit.io/agbb7sr41/eodhd_ouput.png) |
| Sticky Note | Sticky Note | Documentation / overview | — | — | ## What this workflow does\nThis workflow automates **end-to-end stock analysis** using real market data and AI:\n\n- Reads a list of stock tickers from **Google Sheets**\n- Fetches **fundamental data** (valuation, growth, profitability) and **OHLCV price data** from **EODHD APIs**\n- Computes key **technical indicators** (RSI, SMA 20/50/200, volatility, support & resistance)\n- Uses an **AI model** to generate:\n  - Buy / Watch / Sell recommendation\n  - Entry price, stop-loss, and take-profit levels\n  - Investment thesis, pros & cons\n  - Fundamental quality score (1–10)\n- Stores the final structured analysis back into **Google Sheets**\n\nThis creates a **repeatable, no-code stock analysis pipeline** ready for decision-making or dashboards.\n\n### Data source\nMarket data is powered by **EODHD APIs**  \n👉 Get a **10% discount** using this link:  \nhttps://eodhd.com/pricing-special-10?via=kmg&ref1=Meneses\n\n## How to configure this workflow\n\n### 1. Google Sheets (Input)\nCreate a sheet with a column called:\n- `ticker` (e.g. MSFT, AAPL, AMZN)\n\nEach row represents one stock to analyze.\n\n\n### 2. EODHD APIs\n- Create an EODHD account\n- Get your API token\n- Add it to the HTTP Request nodes as:\n  - `api_token=YOUR_API_KEY`\n\nDiscount link (10% off):  \nhttps://eodhd.com/pricing-special-10?via=kmg&ref1=Meneses\n\n\n### 3. AI Model\n- Configure your AI provider (OpenAI / compatible model)\n- The AI receives:\n  - Fundamentals\n  - Technical indicators\n  - Growth potential score\n- It returns structured JSON with recommendations and trade levels\n\n\n### 4. Google Sheets (Output)\nResults are appended to a `Signals` tab with:\n- Signal (BUY / WATCH / SELL)\n- Entry, Stop Loss, Take Profit\n- Fundamental score (1–10)\n- Investment thesis and risk notes\n|
| Sticky Note1 | Sticky Note | Visual label: input area | — | — | ## INPUT  \n![txt](https://ik.imagekit.io/agbb7sr41/eodhd_input.png) |
| Sticky Note2 | Sticky Note | Visual label: AI output area | — | — | ## AI Output\n- The merged data is sent to the **AI system**\n- The AI evaluates fundamentals, technicals, and risk\n- The final output includes:\n  - BUY / WATCH / SELL signal\n  - Entry, Stop Loss, Take Profit\n  - Investment score and rationale\n\nResults are ready to use or store in Google Sheets. |
| Sticky Note3 | Sticky Note | Visual label: transform/merge area | — | — | ##  Transform & Merge  \n\n- Raw market data is cleaned and normalized\n- Technical indicators are calculated\n- Fundamentals and price data are **merged into one dataset**\n\nAt the end of this step, each stock has a single, structured data object. |
| Sticky Note4 | Sticky Note | Visual label: data sources area | — | — | ## Input (Data sources)\n- Stock tickers come from **Google Sheets**\n- Market data is fetched via **API calls**:\n  - Fundamentals\n  - Price candles (OHLCV)\n\nThis step collects all the raw data needed for the analysis. |
| Sticky Note5 | Sticky Note | Visual label: output area | — | — | ## OUTPUT  \n![txt](https://ik.imagekit.io/agbb7sr41/eodhd_ouput.png) |
| Sticky Note6 | Sticky Note | Empty note container | — | — |  |
| Sticky Note7 | Sticky Note | Empty note container | — | — |  |
| Sticky Note8 | Sticky Note | Empty note container | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
2. **Add Manual Trigger**
   - Node: *Manual Trigger*
   - Name: `When clicking ‘Execute workflow’`

3. **Add Google Sheets (read input)**
   - Node: *Google Sheets*
   - Name: `Get row(s) in sheet`
   - Credentials: Google Sheets OAuth2 (or Service Account) with access to the spreadsheet
   - Document ID: your sheet ID (replace `YOUR_GOOGLE_SHEET_ID`)
   - Sheet name: `Sheet1`
   - Operation: get all rows
   - Ensure the sheet has a column named **`ticker`**

4. **Add Split In Batches (loop)**
   - Node: *Split In Batches*
   - Name: `Loop Over Items`
   - Keep defaults initially (optionally set batch size to 1–10 to reduce rate-limit risk)
   - Connect: `Get row(s) in sheet` → `Loop Over Items`
   - Important: connect the **processing path** from the same output you intend to use (this workflow uses output **index 1**).

5. **Add HTTP Request (fundamentals)**
   - Node: *HTTP Request*
   - Name: `Fetch stock fundamentals (EODHD)`
   - Method: GET
   - URL: `https://eodhd.com/api/fundamentals/{{$json.ticker}}.US`
   - Query parameters:
     - `filter` = `General,Highlights,Valuation,Technicals`
     - `api_token` = your EODHD token
     - `fmt` = `json`
   - Connect: `Loop Over Items` → `Fetch stock fundamentals (EODHD)`

6. **Add HTTP Request (OHLC)**
   - Node: *HTTP Request*
   - Name: `Fetch OHLC price data (EODHD)`
   - Method: GET
   - URL: `https://eodhd.com/api/eod/{{$json.General.Code}}.US`
   - Query parameters:
     - `from=2023-01-01`
     - `to=2025-12-30`
     - `period=d`
     - `fmt=json`
     - `api_token=YOUR_EODHD_API_KEY`
   - Enable: **Always Output Data**
   - Connect: `Fetch stock fundamentals (EODHD)` → `Fetch OHLC price data (EODHD)`

7. **Add Code node (aggregate OHLC)**
   - Node: *Code* (JavaScript)
   - Name: `Compute indicators and growth score`
   - Paste logic that:
     - collects all candle items into an array
     - sorts by `date`
     - outputs `{ ohlc: arr, ohlc_len: arr.length }`
   - Connect: `Fetch OHLC price data (EODHD)` → `Compute indicators and growth score`

8. **Add Merge**
   - Node: *Merge*
   - Name: `Merge`
   - Use a merge mode that pairs OHLC bundle with fundamentals (default “merge by index” typically works when each side produces one item per ticker)
   - Connect:
     - `Compute indicators and growth score` → `Merge` (input 1)
     - `Fetch stock fundamentals (EODHD)` → `Merge` (input 2)

9. **Add Code node (normalize + compute indicators)**
   - Node: *Code* (JavaScript)
   - Name: `Normalize OHLC data`
   - Paste the ES5-safe script that:
     - extracts key fundamental fields
     - computes RSI/SMA/volatility/support/resistance/returns
     - computes `growth_potential_score`
     - returns one unified JSON object: `{ ticker, fundamentals, technical, growth_potential_score }`
   - Connect: `Merge` → `Normalize OHLC data`

10. **Add OpenAI Chat Model (LangChain)**
    - Node: *OpenAI Chat Model* (LangChain)
    - Name: `OpenAI Chat Model`
    - Credentials: OpenAI API key
    - Model: `gpt-4.1-nano` (or substitute with an available model)

11. **Add LangChain Agent**
    - Node: *AI Agent* (LangChain Agent)
    - Name: `Generate AI stock analysis`
    - Prompt: include `Analyze the following stock data...` and embed `{{ JSON.stringify($json) }}`
    - System message: paste the strict JSON-only instructions and the mandatory schema
    - Connect:
      - Main: `Normalize OHLC data` → `Generate AI stock analysis`
      - Model: `OpenAI Chat Model` → `Generate AI stock analysis` (language model connection)

12. **Add Code node (parse AI JSON)**
    - Node: *Code* (JavaScript)
    - Name: `Code in JavaScript2`
    - Implement:
      - read `items[0].json.output`
      - `JSON.parse`
    - Connect: `Generate AI stock analysis` → `Code in JavaScript2`

13. **Add Code node (flatten for Sheets)**
    - Node: *Code* (JavaScript)
    - Name: `Prepare data for Google Sheets`
    - Flatten nested fields; join arrays with `" | "`
    - Connect: `Code in JavaScript2` → `Prepare data for Google Sheets`

14. **Add Google Sheets (append output)**
    - Node: *Google Sheets*
    - Name: `Append row in sheet`
    - Credentials: same Google Sheets credentials
    - Document ID: your sheet ID
    - Sheet name: `Sheet1` (or change to `Signals` if you create that tab)
    - Operation: Append row
    - Map columns to fields from `Prepare data for Google Sheets` (e.g., `ticker`, `entry`, `stop_loss`, `take_profit`, `positives`, `negatives`, `thesis`, etc.)
    - Connect: `Prepare data for Google Sheets` → `Append row in sheet`

15. **Close the loop**
    - Connect: `Append row in sheet` → `Loop Over Items` (to fetch the next batch until done)

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| EODHD pricing (10% discount) | https://eodhd.com/pricing-special-10?via=kmg&ref1=Meneses |
| Input screenshot | https://ik.imagekit.io/agbb7sr41/eodhd_input.png |
| Output screenshot | https://ik.imagekit.io/agbb7sr41/eodhd_ouput.png |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.