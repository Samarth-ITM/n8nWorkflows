Generate weekly AI equity research reports with Google Sheets, FMP, NewsAPI, OpenAI, and Gmail

https://n8nworkflows.xyz/workflows/generate-weekly-ai-equity-research-reports-with-google-sheets--fmp--newsapi--openai--and-gmail-12949


# Generate weekly AI equity research reports with Google Sheets, FMP, NewsAPI, OpenAI, and Gmail

## 1. Workflow Overview

**Purpose:** This workflow generates **weekly AI equity research reports** (HTML + PDF) for a configurable list of public companies. It pulls **5 years of financial statements** from Financial Modeling Prep (FMP), retrieves **recent news** from NewsAPI, computes quantitative signals, asks OpenAI to produce structured insights (SWOT + risks + growth outlook), renders a report, converts it to PDF, logs metadata to Google Sheets, and emails the PDF link via Gmail.

**Primary use cases:**
- Automating weekly coverage of a watchlist (portfolio, sector coverage, internal research ops).
- Producing consistent, repeatable “first-draft” research packets grounded in fetched data.

### 1.1 Scheduling & Input Reception
Runs weekly, loads a “companies” list from Google Sheets.

### 1.2 Company Selection & Looping
Filters only enabled rows and iterates companies one at a time (batch loop).

### 1.3 Data Collection (Financials + News)
For each ticker, fetches income statement, balance sheet, cash flow (FMP) and recent news (NewsAPI).

### 1.4 Normalization & Signal Computation
Normalizes raw API payloads into consistent structures; computes key signals (growth, margins, leverage, FCF).

### 1.5 AI Insight Generation (SWOT + Risks/Growth)
Uses OpenAI (LangChain nodes) to produce strict JSON outputs, then parses them.

### 1.6 Report Assembly, PDF Rendering, Logging & Delivery
Merges insights, builds HTML, converts to PDF, validates success, logs metadata to Sheets, emails the link, then continues to the next company.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduling & Company List Loading
**Overview:** Triggers weekly and reads the company universe from a Google Sheet tab.  
**Nodes involved:** `Weekly Equity Research Trigger`, `Fetch Company List`

#### Node: Weekly Equity Research Trigger
- **Type / role:** Schedule Trigger — workflow entry point.
- **Configuration (interpreted):** Runs every week, **Monday at 08:00** (based on `triggerAtDay: [1]`, `triggerAtHour: 8`).
- **Inputs/Outputs:** No input; outputs a single trigger item to `Fetch Company List`.
- **Version notes:** `typeVersion 1.3` (schedule UI differs slightly across n8n versions).
- **Failure/edge cases:** Timezone depends on n8n instance settings; DST shifts may affect perceived runtime.

#### Node: Fetch Company List
- **Type / role:** Google Sheets — read rows from a sheet (company configuration source).
- **Configuration (interpreted):**
  - Document: **“NEWS IMPACT TRACKER”** (Spreadsheet ID `1Mz-wo...`).
  - Sheet tab: **“companies”**.
  - Operation appears to be default “read/get many” behavior (no explicit operation shown in JSON).
- **Key fields expected in rows:** at least `ticker` and `enabled` (used later).
- **Credentials:** Google Sheets OAuth2 (`automations@techdome.ai`).
- **Outputs:** Items representing rows; sent to `Filter Enabled Companies`.
- **Failure/edge cases:** Missing permissions, revoked OAuth token, renamed sheet/tab, missing columns (`enabled`, `ticker`).

---

### Block 2 — Company Selection & Looping
**Overview:** Filters only enabled companies and processes them sequentially.  
**Nodes involved:** `Filter Enabled Companies`, `Iterate Companies`

#### Node: Filter Enabled Companies
- **Type / role:** IF — gatekeeper to keep only enabled rows.
- **Configuration (interpreted):**
  - Condition: `{{ $json.enabled }}` is boolean **true**.
  - Strict type validation enabled.
- **Inputs/Outputs:** Input from `Fetch Company List`; true-output to `Iterate Companies`. False path unused.
- **Failure/edge cases:**
  - If `enabled` is `"TRUE"` (string) instead of boolean, strict validation may fail to match.
  - If the sheet uses 1/0 or yes/no, condition won’t pass unless adapted.

#### Node: Iterate Companies
- **Type / role:** Split In Batches — iteration controller.
- **Configuration (interpreted):** Default batch settings (batch size not specified; n8n default is typically 1).
- **Connections:**
  - **Output 1 (main index 0):** used as “continue” input (fed by `Send Email…` to process next company).
  - **Output 2 (main index 1):** fans out to API fetch nodes: `Fetch Income Statement`, `Fetch Balance Sheet`, `Fetch Cash Flow`, `Fetch Market Data`.
- **Failure/edge cases:**
  - If an iteration path errors and execution stops, subsequent companies won’t run.
  - Batch size > 1 would complicate “combineByPosition” merges later (the workflow assumes positional alignment).

---

### Block 3 — Financials & News Data Collection
**Overview:** Fetches annual statements (5 years) from FMP and latest news from NewsAPI for the current ticker.  
**Nodes involved:** `Fetch Income Statement`, `Fetch Balance Sheet`, `Fetch Cash Flow`, `Fetch Market Data` (note: name suggests “market data” but it’s news)

#### Node: Fetch Income Statement
- **Type / role:** HTTP Request — call FMP income statement endpoint.
- **Configuration (interpreted):**
  - URL: `https://financialmodelingprep.com/stable/income-statement`
  - Query params:
    - `period=annual`
    - `limit=5`
    - `symbol={{ $json.ticker }}`
    - `apikey=your_api_key` (placeholder)
- **Input/Output:** Input is the current company item from `Iterate Companies`; output to `Normalize Income Statement Data`.
- **Failure/edge cases:** Invalid API key, rate limits, unexpected schema changes, empty results for ticker, HTTP errors.

#### Node: Fetch Balance Sheet
- **Type / role:** HTTP Request — call FMP balance sheet endpoint.
- **Configuration:** Similar pattern:
  - URL: `https://financialmodelingprep.com/stable/balance-sheet-statement`
  - same query params structure (`symbol={{ $json.ticker }}`)
- **Output:** `Normalize Balance Sheet Data`.
- **Failure/edge cases:** Same as above, plus some tickers may not have `netDebt` etc.

#### Node: Fetch Cash Flow
- **Type / role:** HTTP Request — call FMP cash flow endpoint.
- **Configuration:**
  - URL: `https://financialmodelingprep.com/stable/cash-flow-statement`
  - `period=annual`, `limit=5`, `symbol={{ $json.ticker }}`, `apikey=your_api_key`
- **Output:** `Normalize Cash Flow Data`.
- **Failure/edge cases:** Missing fields such as `freeCashFlow` depending on provider/ticker.

#### Node: Fetch Market Data (NewsAPI)
- **Type / role:** HTTP Request — fetch recent news articles.
- **Configuration:**
  - URL: `https://newsapi.org/v2/everything`
  - Query params:
    - `q={{ $json.ticker }}`
    - `language=en`
    - `sortBy=publishedAt`
    - `pageSize=10`
    - `apiKey=your_newsapi_key` (placeholder)
- **Output:** `Normalize Market News Data`.
- **Failure/edge cases:** NewsAPI dev-plan limits, `q=ticker` may return noisy/unrelated results, empty `articles`.

---

### Block 4 — Normalization & Data Merge
**Overview:** Converts raw API outputs into a stable schema and merges the four streams (income, balance, cash flow, news) into a single item.  
**Nodes involved:** `Normalize Income Statement Data`, `Normalize Balance Sheet Data`, `Normalize Cash Flow Data`, `Normalize Market News Data`, `Merge Financials and News`

#### Node: Normalize Income Statement Data
- **Type / role:** Code — reshape FMP income statement array into a compact 5-year structure.
- **Configuration (interpreted):**
  - Reads all incoming items (`$input.all()`), takes first 5, maps fields to:
    - `fiscal_year`, `revenue`, `gross_profit`, `operating_income`, `ebitda`, `net_income`, `eps_diluted`
  - Outputs a single item: `{ income_statement: [...] }`
- **Connections:** From `Fetch Income Statement` → into `Merge Financials and News` input 1 (index 0).
- **Failure/edge cases:** If HTTP node returns an object instead of array items as expected, mapping may break. Field names must match FMP response.

#### Node: Normalize Balance Sheet Data
- **Type / role:** Code — reshape balance sheet.
- **Output schema:** `{ balance_sheet: [ { fiscal_year, assets:{...}, liabilities:{...}, debt:{...}, equity:{...} } ] }`
- **Connections:** To `Merge Financials and News` input 2 (index 1).
- **Failure/edge cases:** Division later uses `total_equity`; if 0/null, downstream `debt_to_equity` becomes `Infinity/NaN`.

#### Node: Normalize Cash Flow Data
- **Type / role:** Code — reshape cash flow.
- **Output schema:** `{ cash_flow: [ { fiscal_year, cash_generation:{...}, earnings_to_cash:{...}, capital_allocation:{...} } ] }`
- **Connections:** To `Merge Financials and News` input 3 (index 2).
- **Failure/edge cases:** Missing `freeCashFlow` causes later `fcf_margin` to become `null/NaN`.

#### Node: Normalize Market News Data
- **Type / role:** Code — reshape NewsAPI results.
- **Configuration (important behavior):**
  - Uses `$json.articles || []`, keeps first 5.
  - Maps to `{ title, summary, source, published_at, url }`.
  - **Returns an object**: `return { news: normalized };`
    - In n8n Code node, returning an object is accepted, but it must resolve to item JSON; if your n8n version expects `return [{json: ...}]`, adjust.
- **Connections:** To `Merge Financials and News` input 4 (index 3).
- **Failure/edge cases:** If NewsAPI returns error structure (e.g., `{status:"error", message:...}`), `articles` absent → empty news list (still OK).

#### Node: Merge Financials and News
- **Type / role:** Merge — combines 4 inputs by position.
- **Configuration (interpreted):**
  - Mode: **combine**
  - Combine by: **position** (`combineByPosition`)
  - `numberInputs=4`
- **Inputs:** Normalized income (0), balance (1), cash flow (2), news (3).
- **Output:** Single merged item with keys: `income_statement`, `balance_sheet`, `cash_flow`, `news`.
- **Failure/edge cases:** If any branch outputs 0 items (e.g., failed request or code crash), positional merge may misalign or produce no output.

---

### Block 5 — Signal Computation
**Overview:** Computes quantitative signals for the AI prompts and packages original normalized data.  
**Nodes involved:** `Compute Financial and Market Signals`

#### Node: Compute Financial and Market Signals
- **Type / role:** Code — derive metrics for AI reasoning.
- **Computed signals (from code):**
  - `revenue_growth_5y = (latestIncome.revenue - oldestIncome.revenue) / oldestIncome.revenue`
  - `operating_margin = latestIncome.operating_income / latestIncome.revenue`
  - `debt_to_equity = latestBalance.debt.total_debt / latestBalance.equity.total_equity`
  - `fcf_margin = latestCash.cash_generation.free_cash_flow / latestIncome.revenue`
  - `stock_buybacks = latestCash.capital_allocation.stock_repurchased`
  - `news` and `news_count`
- **Outputs:**
  - `swot_signals: {...}`
  - `original_data: { income_statement, balance_sheet, cash_flow, news }`
- **Connections:** Feeds both AI agent nodes: `Generate SWOT Analysis` and `Generate Risks and Growth Outlook`.
- **Failure/edge cases:**
  - Division by zero / null: revenue, equity.
  - Assumes arrays exist and are non-empty (`income[0]`, etc.). Empty arrays will throw.

---

### Block 6 — AI Insight Generation (SWOT, Risks/Growth) + Parsing
**Overview:** Two OpenAI-backed LangChain agents generate strict JSON, then code nodes parse the JSON safely.  
**Nodes involved:** `LLM – SWOT Analysis Model`, `Generate SWOT Analysis`, `Parse SWOT Output`, `LLM – Risk and Growth Model`, `Generate Risks and Growth Outlook`, `Parse Risks and Growth Output`

#### Node: LLM – SWOT Analysis Model
- **Type / role:** LangChain Chat Model (OpenAI) — provides the model to the agent.
- **Configuration:** Model `gpt-4.1-mini`. No special tools enabled.
- **Credentials:** OpenAI API credential `OpenAi account 2`.
- **Connections:** Connected via `ai_languageModel` to `Generate SWOT Analysis`.
- **Failure/edge cases:** Model availability, org-level limits, token limits if inputs are large.

#### Node: Generate SWOT Analysis
- **Type / role:** LangChain Agent — prompts the model to output SWOT JSON only.
- **Prompt behavior:**
  - Input: `{{ $json.toJsonString() }}` (entire signals + original data item).
  - Rules: “Do NOT invent data”, must base each point on numeric signal or news item.
  - Output: JSON with arrays `strengths/weaknesses/opportunities/threats`.
- **Connections:** Output to `Parse SWOT Output`.
- **Failure/edge cases:** Model may return invalid JSON (extra prose, markdown), causing parse fallback.

#### Node: Parse SWOT Output
- **Type / role:** Code — JSON.parse with fallback.
- **Logic:** Reads `$json.output`; tries JSON.parse; on failure returns empty arrays for SWOT fields.
- **Connections:** To `Merge All AI Insights` input 1 (index 0).
- **Failure/edge cases:** If agent output is not in `$json.output` (schema change), parser returns fallback.

#### Node: LLM – Risk and Growth Model
- **Type / role:** LangChain Chat Model (OpenAI).
- **Configuration:** Model `gpt-4.1-mini`.
- **Connections:** `ai_languageModel` to `Generate Risks and Growth Outlook`.

#### Node: Generate Risks and Growth Outlook
- **Type / role:** LangChain Agent — outputs risks + growth outlook JSON only.
- **Prompt behavior:**
  - Uses `{{ $json.swot_signals.toJsonString() }}` and `{{ $json.original_data.toJsonString() }}`
  - Output JSON: `{ "risks": [], "growth_outlook": "" }`
- **Connections:** Output to `Parse Risks and Growth Output`.
- **Failure/edge cases:** Same invalid JSON risk; token size if news list grows.

#### Node: Parse Risks and Growth Output
- **Type / role:** Code — JSON.parse with fallback.
- **Fallback:** `{ risks: [], growth_outlook: "" }`
- **Connections:** To `Merge All AI Insights` input 2 (index 1).

---

### Block 7 — Report Generation, PDF, Logging, Email, Loop Continuation
**Overview:** Merges AI outputs, renders HTML report, converts to PDF, validates, logs metadata, emails the link, then triggers next company iteration.  
**Nodes involved:** `Merge All AI Insights`, `Build Equity Research Report`, `Render Research Report to PDF`, `Validate PDF Generation`, `Log Report Metadata`, `Send Email Equity Research Report`

#### Node: Merge All AI Insights
- **Type / role:** Merge — combine SWOT + risks/growth outputs.
- **Configuration:** Combine by position (two inputs).
- **Output:** Single item containing `strengths/weaknesses/opportunities/threats/risks/growth_outlook`.
- **Connections:** To `Build Equity Research Report`.
- **Failure/edge cases:** If one AI branch returns 0 items, merge may drop output.

#### Node: Build Equity Research Report
- **Type / role:** Code — builds an HTML document for the report.
- **Key expressions/variables:**
  - Pulls company ticker from a different node context:
    - `$('Iterate Companies').item.json.ticker`
  - Uses lists from current `$json` (merged AI insights).
  - Produces: `{ report_html: "<!DOCTYPE html>..." }`
- **Connections:** To `Render Research Report to PDF`.
- **Failure/edge cases:**
  - If `Iterate Companies` context is unavailable (changed execution settings or node renamed), ticker interpolation fails.
  - If arrays are not arrays, `map` fails in `renderList`.

#### Node: Render Research Report to PDF
- **Type / role:** HTMLCSS to PDF — external PDF rendering service node.
- **Configuration:** `html_content={{ $json.report_html }}`
- **Credentials:** `HTML to PDF account` (htmlcsstopdfApi).
- **Expected output fields:** includes `success`, `pdf_url`, `file_deletion_date`, `file_size_bytes` (as implied by downstream usage).
- **Failure/edge cases:** Rendering service downtime, HTML too large, credential issues, URL expiry too short for recipients.

#### Node: Validate PDF Generation
- **Type / role:** IF — ensures PDF generation succeeded.
- **Condition:** `{{ $json.success }}` is boolean true.
- **True path:** `Log Report Metadata` and `Send Email Equity Research Report`.
- **False path:** unused (no explicit failure handling).
- **Failure/edge cases:** If service returns `"true"` string, strict boolean compare may fail.

#### Node: Log Report Metadata
- **Type / role:** Google Sheets — append report record to a log tab.
- **Configuration (interpreted):**
  - Operation: **append**
  - Sheet tab: **“AI Equity Research Report”**
  - Columns appended:
    - Company: `{{ $('Iterate Companies').item.json.ticker }}`
    - PDF URL: `{{$json.pdf_url}}`
    - Timestamp: `{{$now}}`
    - Expiry Date: `{{$json.file_deletion_date}}`
    - Report Type: `AI Equity Research`
    - File Size (KB): `{{ Math.round($json.file_size_bytes / 1024) }}`
- **Credentials:** Google Sheets OAuth2 (`automations@techdome.ai`).
- **Failure/edge cases:** Sheet schema mismatch, permission errors, API quotas.

#### Node: Send Email Equity Research Report
- **Type / role:** Gmail — sends recipient the PDF link.
- **Configuration:**
  - To: `your_email_id` (placeholder)
  - Subject: “AI Equity Research Report – Ready for Review”
  - HTML message includes:
    - Link: `{{ $json.pdf_url }}`
    - File size KB, link expiry date
  - Attribution disabled.
- **Credentials:** Gmail OAuth2 (`Gmail credentials`).
- **Connections:** After sending, it routes back to `Iterate Companies` (continue loop).
- **Failure/edge cases:** OAuth token revoked, Gmail sending limits, spam filtering, placeholder recipient not set.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Weekly Equity Research Trigger | Schedule Trigger | Weekly workflow entry point | — | Fetch Company List | ### Company Selection and Looping<br>These nodes load companies from Google Sheets, filters only enabled entries, and loops through each company one at a time. It ensures controlled execution and allows users to manage company coverage directly from the sheet. |
| Fetch Company List | Google Sheets | Load company rows from spreadsheet | Weekly Equity Research Trigger | Filter Enabled Companies | ### Company Selection and Looping<br>These nodes load companies from Google Sheets, filters only enabled entries, and loops through each company one at a time. It ensures controlled execution and allows users to manage company coverage directly from the sheet. |
| Filter Enabled Companies | IF | Keep only enabled companies | Fetch Company List | Iterate Companies | ### Company Selection and Looping<br>These nodes load companies from Google Sheets, filters only enabled entries, and loops through each company one at a time. It ensures controlled execution and allows users to manage company coverage directly from the sheet. |
| Iterate Companies | Split In Batches | Loop through companies sequentially | Filter Enabled Companies; Send Email Equity Research Report | Fetch Income Statement; Fetch Balance Sheet; Fetch Cash Flow; Fetch Market Data | ### Company Selection and Looping<br>These nodes load companies from Google Sheets, filters only enabled entries, and loops through each company one at a time. It ensures controlled execution and allows users to manage company coverage directly from the sheet. |
| Fetch Income Statement | HTTP Request | Get annual income statement data (FMP) | Iterate Companies | Normalize Income Statement Data | ### Financial and News Data Collection<br>These nodes fetch financial statements and recent market news from external APIs. Keeping data collection separate helps maintain clarity, simplifies troubleshooting, and ensures reliable inputs for later normalization and analysis steps. |
| Fetch Balance Sheet | HTTP Request | Get annual balance sheet data (FMP) | Iterate Companies | Normalize Balance Sheet Data | ### Financial and News Data Collection<br>These nodes fetch financial statements and recent market news from external APIs. Keeping data collection separate helps maintain clarity, simplifies troubleshooting, and ensures reliable inputs for later normalization and analysis steps. |
| Fetch Cash Flow | HTTP Request | Get annual cash flow data (FMP) | Iterate Companies | Normalize Cash Flow Data | ### Financial and News Data Collection<br>These nodes fetch financial statements and recent market news from external APIs. Keeping data collection separate helps maintain clarity, simplifies troubleshooting, and ensures reliable inputs for later normalization and analysis steps. |
| Fetch Market Data | HTTP Request | Fetch recent news (NewsAPI) | Iterate Companies | Normalize Market News Data | ### Financial and News Data Collection<br>These nodes fetch financial statements and recent market news from external APIs. Keeping data collection separate helps maintain clarity, simplifies troubleshooting, and ensures reliable inputs for later normalization and analysis steps. |
| Normalize Income Statement Data | Code | Normalize income statement schema | Fetch Income Statement | Merge Financials and News | ### Financial and News Data Collection<br>These nodes fetch financial statements and recent market news from external APIs. Keeping data collection separate helps maintain clarity, simplifies troubleshooting, and ensures reliable inputs for later normalization and analysis steps. |
| Normalize Balance Sheet Data | Code | Normalize balance sheet schema | Fetch Balance Sheet | Merge Financials and News | ### Financial and News Data Collection<br>These nodes fetch financial statements and recent market news from external APIs. Keeping data collection separate helps maintain clarity, simplifies troubleshooting, and ensures reliable inputs for later normalization and analysis steps. |
| Normalize Cash Flow Data | Code | Normalize cash flow schema | Fetch Cash Flow | Merge Financials and News | ### Financial and News Data Collection<br>These nodes fetch financial statements and recent market news from external APIs. Keeping data collection separate helps maintain clarity, simplifies troubleshooting, and ensures reliable inputs for later normalization and analysis steps. |
| Normalize Market News Data | Code | Normalize NewsAPI articles | Fetch Market Data | Merge Financials and News | ### Financial and News Data Collection<br>These nodes fetch financial statements and recent market news from external APIs. Keeping data collection separate helps maintain clarity, simplifies troubleshooting, and ensures reliable inputs for later normalization and analysis steps. |
| Merge Financials and News | Merge | Combine normalized datasets | Normalize Income Statement Data; Normalize Balance Sheet Data; Normalize Cash Flow Data; Normalize Market News Data | Compute Financial and Market Signals | ### AI Analysis and Signal Generation<br>This group calculates financial signals and uses AI to generate SWOT insights, risks, and growth outlook. The AI relies only on computed metrics and recent news, avoiding assumptions or unsupported conclusions. |
| Compute Financial and Market Signals | Code | Compute signals for AI | Merge Financials and News | Generate SWOT Analysis; Generate Risks and Growth Outlook | ### AI Analysis and Signal Generation<br>This group calculates financial signals and uses AI to generate SWOT insights, risks, and growth outlook. The AI relies only on computed metrics and recent news, avoiding assumptions or unsupported conclusions. |
| LLM – SWOT Analysis Model | OpenAI Chat Model (LangChain) | Provides model to SWOT agent | — | Generate SWOT Analysis (ai_languageModel) | ### AI Analysis and Signal Generation<br>This group calculates financial signals and uses AI to generate SWOT insights, risks, and growth outlook. The AI relies only on computed metrics and recent news, avoiding assumptions or unsupported conclusions. |
| Generate SWOT Analysis | LangChain Agent | Generate SWOT JSON | Compute Financial and Market Signals; LLM – SWOT Analysis Model | Parse SWOT Output | ### AI Analysis and Signal Generation<br>This group calculates financial signals and uses AI to generate SWOT insights, risks, and growth outlook. The AI relies only on computed metrics and recent news, avoiding assumptions or unsupported conclusions. |
| Parse SWOT Output | Code | Parse/validate SWOT JSON | Generate SWOT Analysis | Merge All AI Insights | ### AI Analysis and Signal Generation<br>This group calculates financial signals and uses AI to generate SWOT insights, risks, and growth outlook. The AI relies only on computed metrics and recent news, avoiding assumptions or unsupported conclusions. |
| LLM – Risk and Growth Model | OpenAI Chat Model (LangChain) | Provides model to risk/growth agent | — | Generate Risks and Growth Outlook (ai_languageModel) | ### AI Analysis and Signal Generation<br>This group calculates financial signals and uses AI to generate SWOT insights, risks, and growth outlook. The AI relies only on computed metrics and recent news, avoiding assumptions or unsupported conclusions. |
| Generate Risks and Growth Outlook | LangChain Agent | Generate risks + growth JSON | Compute Financial and Market Signals; LLM – Risk and Growth Model | Parse Risks and Growth Output | ### AI Analysis and Signal Generation<br>This group calculates financial signals and uses AI to generate SWOT insights, risks, and growth outlook. The AI relies only on computed metrics and recent news, avoiding assumptions or unsupported conclusions. |
| Parse Risks and Growth Output | Code | Parse/validate risks+growth JSON | Generate Risks and Growth Outlook | Merge All AI Insights | ### AI Analysis and Signal Generation<br>This group calculates financial signals and uses AI to generate SWOT insights, risks, and growth outlook. The AI relies only on computed metrics and recent news, avoiding assumptions or unsupported conclusions. |
| Merge All AI Insights | Merge | Combine SWOT + risk/growth outputs | Parse SWOT Output; Parse Risks and Growth Output | Build Equity Research Report | ### Report Generation and Delivery<br>These nodes assemble the final report, convert it into a PDF, log key details in Google Sheets, and send the report link by email once generation is confirmed successful. |
| Build Equity Research Report | Code | Render final HTML report | Merge All AI Insights | Render Research Report to PDF | ### Report Generation and Delivery<br>These nodes assemble the final report, convert it into a PDF, log key details in Google Sheets, and send the report link by email once generation is confirmed successful. |
| Render Research Report to PDF | HTMLCSS to PDF | Convert HTML to PDF and host it | Build Equity Research Report | Validate PDF Generation | ### Report Generation and Delivery<br>These nodes assemble the final report, convert it into a PDF, log key details in Google Sheets, and send the report link by email once generation is confirmed successful. |
| Validate PDF Generation | IF | Gate based on PDF success | Render Research Report to PDF | Log Report Metadata; Send Email Equity Research Report | ### Report Generation and Delivery<br>These nodes assemble the final report, convert it into a PDF, log key details in Google Sheets, and send the report link by email once generation is confirmed successful. |
| Log Report Metadata | Google Sheets | Append PDF metadata to log sheet | Validate PDF Generation | — | ### Report Generation and Delivery<br>These nodes assemble the final report, convert it into a PDF, log key details in Google Sheets, and send the report link by email once generation is confirmed successful. |
| Send Email Equity Research Report | Gmail | Email PDF link to recipient | Validate PDF Generation | Iterate Companies | ### Report Generation and Delivery<br>These nodes assemble the final report, convert it into a PDF, log key details in Google Sheets, and send the report link by email once generation is confirmed successful. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n (keep execution order setting as default; this workflow uses `executionOrder: v1`).
2. **Add Schedule Trigger** node:
   - Name: `Weekly Equity Research Trigger`
   - Set rule: weekly, **Monday**, at **08:00** (confirm instance timezone).
3. **Add Google Sheets node** (`Fetch Company List`):
   - Credential: Google Sheets OAuth2
   - Select Spreadsheet: your tracker file (e.g., “NEWS IMPACT TRACKER”)
   - Select Sheet tab: `companies`
   - Configure to read all rows (default “get many” behavior).
   - Ensure sheet has at minimum:
     - `ticker` (e.g., AAPL)
     - `enabled` (boolean TRUE/FALSE)
4. **Add IF node** (`Filter Enabled Companies`):
   - Condition: boolean equals true
   - Left value: `{{$json.enabled}}`
   - Connect: Trigger → Fetch Company List → IF (true path only used).
5. **Add Split In Batches** node (`Iterate Companies`):
   - Batch size: **1** (recommended to keep merges aligned).
   - Connect IF(true) → SplitInBatches.
6. **Add 4 HTTP Request nodes** connected from `Iterate Companies` **output 2** (the “batch” output):
   - `Fetch Income Statement`
     - URL: `https://financialmodelingprep.com/stable/income-statement`
     - Query: `period=annual`, `limit=5`, `symbol={{$json.ticker}}`, `apikey=<YOUR_FMP_KEY>`
   - `Fetch Balance Sheet`
     - URL: `https://financialmodelingprep.com/stable/balance-sheet-statement`
     - Same query pattern
   - `Fetch Cash Flow`
     - URL: `https://financialmodelingprep.com/stable/cash-flow-statement`
     - Same query pattern
   - `Fetch Market Data` (NewsAPI)
     - URL: `https://newsapi.org/v2/everything`
     - Query: `q={{$json.ticker}}`, `language=en`, `sortBy=publishedAt`, `pageSize=10`, `apiKey=<YOUR_NEWSAPI_KEY>`
7. **Add 4 Code nodes** to normalize outputs:
   - `Normalize Income Statement Data` (map to `income_statement`)
   - `Normalize Balance Sheet Data` (map to `balance_sheet`)
   - `Normalize Cash Flow Data` (map to `cash_flow`)
   - `Normalize Market News Data` (map to `news`, limit to 5 articles)
   - Connect each HTTP node → its normalizer.
8. **Add Merge node** (`Merge Financials and News`):
   - Mode: **Combine**
   - Combine by: **Position**
   - Number of inputs: **4**
   - Connect the four normalization nodes into inputs 1–4 in a stable order (income, balance, cash, news).
9. **Add Code node** (`Compute Financial and Market Signals`):
   - Compute the key ratios and embed `news` and counts.
   - Output two top-level keys: `swot_signals` and `original_data`.
   - Connect Merge → this node.
10. **Add OpenAI Chat Model (LangChain) node** (`LLM – SWOT Analysis Model`):
    - Credential: OpenAI API key
    - Model: `gpt-4.1-mini`
11. **Add LangChain Agent node** (`Generate SWOT Analysis`):
    - Prompt type: “define”
    - Prompt requires JSON-only output for SWOT.
    - Connect:
      - `Compute Financial and Market Signals` → agent main input
      - `LLM – SWOT Analysis Model` → agent `ai_languageModel` input
12. **Add Code node** (`Parse SWOT Output`):
    - Parse `$json.output` as JSON; fallback to empty SWOT arrays.
    - Connect agent → parser.
13. **Repeat steps 10–12** for risks/growth:
    - Model node: `LLM – Risk and Growth Model` (same model)
    - Agent: `Generate Risks and Growth Outlook` (JSON-only `{risks:[], growth_outlook:""}`)
    - Parser: `Parse Risks and Growth Output`
    - Connect from `Compute Financial and Market Signals` into the agent.
14. **Add Merge node** (`Merge All AI Insights`):
    - Combine by position (2 inputs)
    - Inputs: `Parse SWOT Output` and `Parse Risks and Growth Output`
15. **Add Code node** (`Build Equity Research Report`):
    - Generate HTML with sections for SWOT + risks + growth outlook
    - Reference the ticker from the loop context: `$('Iterate Companies').item.json.ticker`
16. **Add HTML to PDF node** (`Render Research Report to PDF`):
    - Credential: HTMLCSS-to-PDF provider (htmlcsstopdf)
    - HTML content: `{{$json.report_html}}`
17. **Add IF node** (`Validate PDF Generation`):
    - Condition: `{{$json.success}}` boolean true
18. **Add Google Sheets node** (`Log Report Metadata`) on the IF **true** branch:
    - Operation: append
    - Sheet tab: `AI Equity Research Report`
    - Append columns: Company, PDF URL, Timestamp, Expiry Date, Report Type, File Size (KB)
19. **Add Gmail node** (`Send Email Equity Research Report`) on the IF **true** branch:
    - Credential: Gmail OAuth2
    - To: your recipient(s)
    - Subject + HTML body including `{{$json.pdf_url}}`, `{{$json.file_size_bytes}}`, `{{$json.file_deletion_date}}`
20. **Close the loop:** Connect `Send Email Equity Research Report` → back to `Iterate Companies` (to trigger the next batch item).

**Credentials to configure:**
- Google Sheets OAuth2 (read companies + append report log)
- OpenAI API (for both model nodes)
- Gmail OAuth2 (send report email)
- HTMLCSS-to-PDF API (render and host PDF)
- External API keys in HTTP nodes:
  - Financial Modeling Prep: `apikey`
  - NewsAPI: `apiKey`

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “This workflow automatically creates a professional equity research report for selected companies on a weekly basis…” plus setup steps (companies sheet with enabled column; configure FMP/NewsAPI/OpenAI keys; connect Sheets + Gmail) | Sticky note: **Workflow Overview** |
| Company coverage is controlled from Google Sheets via an `enabled` boolean column. | Sticky note: **Company Selection and Looping** |
| Financial/news collection is intentionally separated to simplify troubleshooting and keep inputs reliable. | Sticky note: **Financial and News Data Collection** |
| AI is instructed to rely only on computed metrics and news; avoid unsupported conclusions. | Sticky note: **AI Analysis and Signal Generation** |
| Final stage assembles report, converts to PDF, logs to Sheets, and emails once success is confirmed. | Sticky note: **Report Generation and Delivery** |