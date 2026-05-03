Send sales forecast charts and answer Q&A on WhatsApp with OpenAI

https://n8nworkflows.xyz/workflows/send-sales-forecast-charts-and-answer-q-a-on-whatsapp-with-openai-12289


# Send sales forecast charts and answer Q&A on WhatsApp with OpenAI

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow automatically generates a **monthly sales forecast**, turns it into an **executive business summary**, renders a **forecast chart (PNG)**, sends the report to **WhatsApp**, and then supports **interactive Q&A on WhatsApp** using the latest saved forecast context.

**Target use cases:**
- Monthly sales reporting pushed to a phone (Director/Owner).
- Lightweight forecasting with deterministic math (no AI-generated numbers).
- Follow-up Q&A (“Why is next month lower?”, “What’s the trend?”, etc.) grounded in the most recent forecast + historical sales.

### 1.1 Scheduled Forecast & Report Delivery (Top branch)
- Trigger monthly run → load sales history from Google Sheets → clean data → run deterministic forecasting tournament → ask OpenAI to translate results into business language (structured JSON) → create chart config → render chart via QuickChart → store latest forecast context in n8n Data Table → send WhatsApp image + caption summary.

### 1.2 WhatsApp Q&A Assistant (Bottom branch)
- WhatsApp inbound message → filter allowed sender → normalize incoming payload → retrieve “latest” forecast row → parse stored history JSON → run OpenAI agent with memory → respond on WhatsApp.

---

## 2. Block-by-Block Analysis

### Block 2.1 — Monthly Trigger & Configuration
**Overview:** Kicks off the forecasting pipeline on a monthly schedule and sets basic configuration values (intended for sheet selection).  
**Nodes involved:** `Monthly Schedule`, `Workflow Configuration`

#### Node: Monthly Schedule
- **Type / role:** `scheduleTrigger` — time-based entry point (monthly).
- **Configuration (interpreted):** Runs **every month at 09:00** (server/project timezone).
- **Connections:** Outputs to `Workflow Configuration`.
- **Edge cases / failures:**
  - Timezone differences can cause “unexpected” trigger hour.
  - If workflow is inactive (it is currently `active: false`), schedule won’t run.

#### Node: Workflow Configuration
- **Type / role:** `Set` — sets config fields for downstream use.
- **Configuration choices:**
  - Adds fields: `spreadsheetId = "SalesSheet"`, `sheetName = "Sales"`.
  - Keeps other fields (`includeOtherFields: true`).
- **Connections:** Outputs to `Get Sales Data`.
- **Important note:** In this JSON, `Get Sales Data` is configured with explicit document/sheet IDs; these set fields are not referenced later. They look like template placeholders for a more dynamic version.
- **Edge cases:**
  - If you later refactor to use these fields in expressions, missing/incorrect values will break the Sheets read.

**Sticky note context (applies to this area):**  
“## 1. Data Pipeline — ETL & Normalization Fetches raw Google Sheets data and normalizes formats…”  
(Sticky Note content preserved in the Summary Table per-node.)

---

### Block 2.2 — Sales Data Ingestion (Google Sheets)
**Overview:** Reads monthly sales history from Google Sheets using a Service Account credential.  
**Nodes involved:** `Get Sales Data`

#### Node: Get Sales Data
- **Type / role:** `Google Sheets` — reads rows from a specific sheet/tab.
- **Configuration choices:**
  - **Authentication:** Service Account.
  - **Document:** `Sample_Monthly_Sales` (ID: `1GqPjqmHcl-WZcadEoFy8idUnOsRiYTQce0Y0fC6uyYk`).
  - **Sheet/tab:** `Sample_Monthly_Sales.csv` (gid cached).
  - Operation defaults to “read/get all rows” behavior for the node version.
- **Connections:** Outputs to `Data Cleaning`.
- **Credentials required:** Google API credential (Service Account JSON or configured integration).
- **Edge cases / failures:**
  - Auth failure / missing permission to the sheet.
  - Sheet structure changes (missing columns like `Year`, `Month`, `MonthlySales`) will break cleaning and forecasting.
  - Partial data / non-numeric cells will be filtered out downstream, possibly leaving too few rows.

---

### Block 2.3 — Data Cleaning & Normalization
**Overview:** Converts sheet rows into a normalized monthly time series: numeric `Year`, `Month`, standardized `Sales`, and formatted `period = YYYY-MM`.  
**Nodes involved:** `Data Cleaning`

#### Node: Data Cleaning
- **Type / role:** `Code` — transforms many input items into one consolidated dataset.
- **Configuration choices (logic):**
  - Filters rows where `Year`, `Month`, and `MonthlySales` are numeric.
  - Produces normalized rows:
    - `Year` (number)
    - `Month` (number)
    - `Sales` = `MonthlySales` (number)
    - `period` string formatted `YYYY-MM`
  - Sorts ascending by `(Year, Month)`.
  - Outputs **ONE item**: `{ data: rows, rowCount }`.
- **Connections:**
  - Output goes to:
    - `Forecast Engine` (for modeling)
    - `Combine Data for Chart` (merge input 2)
- **Edge cases / failures:**
  - If all rows are filtered out (bad parsing), `rowCount` becomes 0; downstream forecast code will throw (not enough data).
  - If months are not 1–12, period formatting still happens but seasonality logic becomes unreliable.
  - Duplicates (same Year+Month) are not deduplicated; they will be treated as separate months.

**Sticky note context:**  
“## 2. Forecasting — Statistical Tournament … selects mathematically superior model.”

---

### Block 2.4 — Deterministic Forecasting Tournament
**Overview:** Runs 7 forecasting methods, performs a rolling one-step backtest, selects the best model by lowest MAPE (tie-break RMSE), then forecasts the next month.  
**Nodes involved:** `Forecast Engine`

#### Node: Forecast Engine
- **Type / role:** `Code` — deterministic forecasting + model selection.
- **Key configuration choices (logic highlights):**
  - Requires:
    - Ideally **24+ months** for reliable seasonality.
    - Hard failure if **< 13 months** (`throw Error`).
  - Methods evaluated (rolling backtest):
    1. `seasonal_naive` (lag-12)
    2. `seasonal_average` (avg of same month across years)
    3. `moving_average` (window=3)
    4. `trend_linear` (linear regression over time)
    5. `exp_smoothing` (simple exponential smoothing; alpha tested in [0.2,0.4,0.6,0.8])
    6. `seasonal_regression` (intercept + 11 monthly dummies)
    7. `seasonal_regression_trend` (intercept + time + 11 dummies)
  - Backtest minimum points: `minBacktestPoints = 6` predictions required per method.
  - Selection rule: **lowest MAPE; tie-breaker RMSE**.
  - Output:
    - `series`: `[ { period, Sales }, ... ]`
    - `candidates`: metrics per method (`rmse`, `mape`, `mapeFormatted`, `backtest_points`)
    - `recommended`: `{ forecastHorizon, forecastedSales (rounded), forecastingTechnique, rmse, mape, mapeFormatted, backtestPoints }`
- **Connections:** Outputs to `Analyst AI Agent`.
- **Edge cases / failures:**
  - Too little data (<13 months) → hard stop.
  - If none of the methods produce enough backtest points → hard stop.
  - If best method cannot forecast next month (rare but possible with borderline history length) → hard stop.
  - If source data has gaps (missing months), model assumptions break; it still forecasts “next label” by incrementing last period, not by checking continuity.

---

### Block 2.5 — AI Interpretation with Structured Output
**Overview:** OpenAI converts technical forecast output into an executive narrative + structured fields (confidence, trend, factors), while keeping forecast numbers unchanged.  
**Nodes involved:** `OpenAI Chat Model1`, `Structured Output Parser`, `Analyst AI Agent`, `Format AI Output`

#### Node: OpenAI Chat Model1
- **Type / role:** LangChain Chat Model (`lmChatOpenAi`) — provides the LLM for the analyst agent.
- **Configuration:**
  - Model: `gpt-5`
  - Timeout: 180,000 ms
- **Connections:** Provides `ai_languageModel` input to `Analyst AI Agent`.
- **Credentials:** OpenAI API credential.
- **Edge cases:**
  - Model availability / account access; `gpt-5` must exist on your account/region.
  - Timeouts with large inputs (long history) though timeout is set high.

#### Node: Structured Output Parser
- **Type / role:** LangChain structured parser — enforces a strict JSON schema.
- **Configuration:**
  - Manual JSON schema requiring fields:
    - `forecastHorizon`, `forecastedSales`, `forecastingTechnique`, `confidenceLevel`, `rmse`, `mape`, `mapeFormatted`, `trend`, `seasonality`, `keyFactors`, `reasoning`
  - Disallows additional properties.
- **Connections:** Supplies `ai_outputParser` to `Analyst AI Agent`.
- **Edge cases:**
  - If the model outputs invalid JSON or misses required fields → parser failure.
  - If the model adds extra keys → failure (additionalProperties false).

#### Node: Analyst AI Agent
- **Type / role:** LangChain Agent — “Data Analyst” persona translating math → business narrative.
- **Configuration choices:**
  - Input text: entire `$json` from Forecast Engine (series, candidates, recommended).
  - System message contains strict rules:
    - Must not change `forecastedSales`.
    - Must avoid technical jargon (RMSE, MAPE, OLS, etc.) in narrative fields.
    - Confidence rule based on MAPE (<5 high, <10 medium, else low).
    - Trend computed by comparing last 12 months avg vs previous 12.
    - Seasonality detected if method name contains “seasonal”.
    - Must output only one JSON object matching schema.
  - `hasOutputParser: true` → uses `Structured Output Parser`.
- **Connections:** Output to `Format AI Output`.
- **Edge cases:**
  - If history length is <24, the “trend” instruction (compare last 12 vs previous 12) may be impossible; the agent may guess and risk parser mismatch or logical errors. (The system message does not provide an explicit fallback.)
  - If the LLM violates “no technical jargon”, content may still pass schema but violate business requirements.

#### Node: Format AI Output
- **Type / role:** `Set` — extracts narrative for WhatsApp caption/storage.
- **Configuration:**
  - Sets `summaryReport = $json.output.reasoning`
  - Keeps other fields.
- **Connections:** Output to `Combine Data for Chart` (merge input 1).
- **Edge cases:**
  - Assumes agent output is under `$json.output`. If agent node outputs differently (depending on n8n/LangChain node changes), expression may break.

---

### Block 2.6 — Data Merge for Chart + Chart Rendering (QuickChart)
**Overview:** Combines cleaned historical data with the AI/forecast output, builds a Chart.js config, renders a PNG via QuickChart.  
**Nodes involved:** `Combine Data for Chart`, `Prepare Chart Config`, `QuickChart`

#### Node: Combine Data for Chart
- **Type / role:** `Merge` — combines two inputs by position.
- **Configuration:** Mode `combine`, `combineByPosition`.
- **Inputs:**
  - Input 1: from `Format AI Output` (contains AI output + summary)
  - Input 2: from `Data Cleaning` (contains `data` array)
- **Output:** Single combined item expected (fields from both).
- **Connections:** Output to `Prepare Chart Config`.
- **Edge cases:**
  - If one branch fails or produces 0 items, merge may output nothing or misalign items.
  - “Combine by position” assumes each side produces exactly one item.

#### Node: Prepare Chart Config
- **Type / role:** `Code` — builds Chart.js configuration for historical + next-month forecast.
- **Key logic:**
  - Reads `items[0].json.data` and `items[0].json.output`.
  - Builds labels from historical `period` and appends `forecastHorizon`.
  - Historical series ends with `null` to stop the line at last historical point.
  - Forecast series is mostly `null`, but includes:
    - last historical value at the second-to-last index
    - forecast value at the last index
  - Styling: blue historical line, pink dashed forecast line, larger forecast dot.
- **Connections:** Output to `QuickChart`.
- **Edge cases:**
  - Throws if `json.data` missing/empty.
  - If forecast output missing `forecastHorizon` or `forecastedSales`, chart may be incorrect or throw.

#### Node: QuickChart
- **Type / role:** `HTTP Request` — renders chart image.
- **Configuration:**
  - POST `https://quickchart.io/chart`
  - Body params:
    - `chart` = JSON-stringified chart config
    - `format=png`, `width=1200`, `height=600`, `backgroundColor=white`
  - Response format: **file**, saved to binary property `chart`
- **Connections:** Outputs to both:
  - `Upsert Latest Forecast`
  - `Send WhatsApp Report`
- **Edge cases:**
  - Network failure / QuickChart downtime.
  - Large payload could exceed service limits (very long label series).
  - If response isn’t an image, WhatsApp send will fail.

---

### Block 2.7 — Storage (Latest Forecast Context)
**Overview:** Stores the latest forecast + narrative + historical data snapshot in an n8n Data Table under a fixed key (`latest`) for retrieval by the Q&A assistant.  
**Nodes involved:** `Upsert Latest Forecast`

#### Node: Upsert Latest Forecast
- **Type / role:** `Data Table` — persistent storage inside n8n.
- **Operation:** `upsert` with filter `key == "latest"`.
- **Schema / stored fields:**
  - `key`: set to `"latest"`
  - `forecastPeriod`: from `$json.output.forecastHorizon`
  - `forecastSales`: from `$json.output.forecastedSales`
  - `summaryReport`: from `$json.summaryReport`
  - `historySales`: JSON string created from **original Google Sheets rows**:
    - `year: i.json.Year`, `month: i.json.Month`, `sales: i.json.MonthlySales`
- **Connections:** No downstream nodes.
- **Edge cases:**
  - Data Table not created / wrong `dataTableId` → failure.
  - Stored history is stringified JSON; if it grows large, may exceed storage limits.
  - Uses `$items("Get Sales Data")...` which depends on execution context; if node is renamed or not executed, expression fails.

---

### Block 2.8 — WhatsApp Report Delivery (Image + Caption)
**Overview:** Sends the rendered chart image to a WhatsApp recipient with the executive summary as caption.  
**Nodes involved:** `Send WhatsApp Report`

#### Node: Send WhatsApp Report
- **Type / role:** `WhatsApp` — outbound media message.
- **Configuration:**
  - Operation: `send`
  - Message type: `image`
  - `mediaPropertyName = "chart"` (binary from QuickChart)
  - Caption: `{{$json.summaryReport}}`
  - `phoneNumberId = "978279622026190"`
  - Recipient phone: `"1234"` (template placeholder)
- **Connections:** none.
- **Credentials:** WhatsApp Cloud API credential.
- **Edge cases:**
  - Invalid phone number / not opted-in / WhatsApp policy restrictions.
  - Missing binary property `chart` → send fails.
  - Caption length limits.

---

### Block 2.9 — WhatsApp Trigger, Sender Filter, and Input Normalization
**Overview:** Receives WhatsApp inbound messages, allows only a specific sender, and extracts the user’s text into a stable `chatInput` field despite payload variations.  
**Nodes involved:** `WhatsApp Trigger`, `Filter Text Messages`, `Normalize Whatsapp Input`

#### Node: WhatsApp Trigger
- **Type / role:** `whatsAppTrigger` — webhook entry point.
- **Configuration:** Listens to `updates: ["messages"]`.
- **Connections:** Outputs to `Filter Text Messages`.
- **Credentials:** WhatsApp Trigger API credential.
- **Edge cases:**
  - Webhook verification / subscription issues.
  - Payload shape differs depending on WhatsApp API version.

#### Node: Filter Text Messages
- **Type / role:** `IF` — gate inbound messages by sender.
- **Configuration:**
  - Condition: `{{$json.messages[0].from}} == "16727551224"`
- **Connections:** True branch goes to `Normalize Whatsapp Input` (only that is connected).
- **Edge cases:**
  - If inbound payload doesn’t contain `messages[0].from`, expression evaluation can fail or be empty → message blocked.
  - Hardcoded allowlist is brittle; multiple authorized numbers require OR logic.

#### Node: Normalize Whatsapp Input
- **Type / role:** `Set` — normalizes inbound payload into `chatInput`.
- **Configuration:**
  - `chatInput` expression tries multiple common WhatsApp webhook shapes:
    - `$json.messages[0].text.body`
    - `$json.value.messages[0].text.body`
    - `$json.entry[0].changes[0].value.messages[0].text.body`
    - fallback `""`
- **Connections:** Output to `Get Latest Forecast`.
- **Edge cases:**
  - Non-text messages (image/audio) won’t have `.text.body` → results in empty string.
  - If you want to handle non-text, you must extend the extraction logic.

---

### Block 2.10 — Retrieve Latest Forecast + Parse Stored History
**Overview:** Loads the stored “latest” forecast record and parses `historySales` into a normalized array for the Q&A agent’s context.  
**Nodes involved:** `Get Latest Forecast`, `Parsing Input`

#### Node: Get Latest Forecast
- **Type / role:** `Data Table` — fetch stored row.
- **Operation:** `get` with filter `key == "latest"`.
- **Connections:** Output to `Parsing Input`.
- **Edge cases:**
  - If no row exists yet (forecast pipeline never ran), output may be empty → downstream nodes may not run or will error.
  - Schema mismatch if Data Table was manually changed.

#### Node: Parsing Input
- **Type / role:** `Code` — parses JSON string history safely.
- **Configuration choices (logic):**
  - `history = JSON.parse(row.historySales || "[]")` with try/catch fallback `[]`
  - Normalizes keys to lowercase fields:
    - `year = Number(item.year || item.Year)`
    - `month = Number(item.month || item.Month)`
    - `sales = Number(item.sales || item.Sales || item.MonthlySales)`
  - Outputs `historySalesParsed` appended to row.
- **Connections:** Output to `Q&A AI Agent`.
- **Edge cases:**
  - If `historySales` contains invalid JSON, it silently becomes empty history.
  - `Number(undefined)` becomes `NaN`; downstream prompt will print `NaN` values unless filtered.

---

### Block 2.11 — Q&A AI Agent with Memory + WhatsApp Reply
**Overview:** Answers user questions about the latest forecast using stored context and a per-sender memory window; sends the response back to WhatsApp.  
**Nodes involved:** `OpenAI Chat Model2`, `Simple Memory`, `Q&A AI Agent`, `Q&A Message`

#### Node: OpenAI Chat Model2
- **Type / role:** LangChain Chat Model — provides LLM for Q&A.
- **Configuration:** Model `gpt-4.1-nano`.
- **Connections:** `ai_languageModel` → `Q&A AI Agent`.
- **Edge cases:** Model availability; response quality may be limited compared to larger models.

#### Node: Simple Memory
- **Type / role:** LangChain memory buffer window — conversational memory keyed per sender.
- **Configuration:**
  - `sessionKey = {{$('WhatsApp Trigger').item.json.messages[0].from}}`
  - Session ID type: customKey
- **Connections:** `ai_memory` → `Q&A AI Agent`.
- **Edge cases:**
  - If trigger payload missing `messages[0].from`, session key fails.
  - Memory is ephemeral depending on n8n execution/memory backend and node behavior; not a long-term database.

#### Node: Q&A AI Agent
- **Type / role:** LangChain Agent — “Consultant” persona for answering questions.
- **Configuration:**
  - User text: `{{$('Normalize Whatsapp Input').item.json.chatInput}}`
  - System context includes:
    - Latest forecast period, forecast sales, summary report
    - Full historical sales list formatted as `YYYY-MM: Sales`
- **Connections:** Output to `Q&A Message`.
- **Edge cases:**
  - If `chatInput` is empty, it may respond generically or poorly.
  - Large history list may bloat prompt; consider truncating or summarizing if sheet grows.

#### Node: Q&A Message
- **Type / role:** `WhatsApp` — outbound text reply.
- **Configuration:**
  - Text body: `{{$json.output}}`
  - Recipient phone: `"1234"` (placeholder)
- **Connections:** none.
- **Edge cases:**
  - Assumes agent response is in `$json.output`; if agent output changes format, message may be empty.
  - Recipient phone should be the inbound sender; currently hardcoded, so it will not reply to the actual user unless changed.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Monthly Schedule | scheduleTrigger | Monthly entry point for forecast pipeline | — | Workflow Configuration | ## 1. Data Pipeline<br>**ETL & Normalization** Fetches raw Google Sheets data and normalizes formats to ensure the statistical engine receives clean inputs. |
| Workflow Configuration | Set | Define sheet identifiers (template config) | Monthly Schedule | Get Sales Data | ## 1. Data Pipeline<br>**ETL & Normalization** Fetches raw Google Sheets data and normalizes formats to ensure the statistical engine receives clean inputs. |
| Get Sales Data | Google Sheets | Ingest historical sales rows | Workflow Configuration | Data Cleaning | ## 1. Data Pipeline<br>**ETL & Normalization** Fetches raw Google Sheets data and normalizes formats to ensure the statistical engine receives clean inputs. |
| Data Cleaning | Code | Normalize + sort; output dataset array | Get Sales Data | Forecast Engine; Combine Data for Chart | ## 1. Data Pipeline<br>**ETL & Normalization** Fetches raw Google Sheets data and normalizes formats to ensure the statistical engine receives clean inputs. |
| Forecast Engine | Code | Run 7-model backtest tournament; pick best; forecast next month | Data Cleaning | Analyst AI Agent | ## 2. Forecasting<br>**Statistical Tournament** Runs 7 distinct models (Seasonal Regression, Exp Smoothing, Moving Avg, etc.). It performs a rolling backtest to calculate RMSE & MAPE, strictly selecting the mathematically superior model. |
| OpenAI Chat Model1 | lmChatOpenAi | LLM for executive interpretation | — | Analyst AI Agent (ai_languageModel) | ## 3. AI Intelligence & Visualization<br>**Synthesis & Rendering** The AI interprets the math into a business narrative, while QuickChart renders the data points into a visual trend graph PNG. |
| Structured Output Parser | outputParserStructured | Enforce schema for AI output | — | Analyst AI Agent (ai_outputParser) | ## 3. AI Intelligence & Visualization<br>**Synthesis & Rendering** The AI interprets the math into a business narrative, while QuickChart renders the data points into a visual trend graph PNG. |
| Analyst AI Agent | LangChain Agent | Convert technical forecast into structured executive JSON | Forecast Engine; OpenAI Chat Model1; Structured Output Parser | Format AI Output | ## 3. AI Intelligence & Visualization<br>**Synthesis & Rendering** The AI interprets the math into a business narrative, while QuickChart renders the data points into a visual trend graph PNG. |
| Format AI Output | Set | Extract `summaryReport` from AI reasoning | Analyst AI Agent | Combine Data for Chart | ## 3. AI Intelligence & Visualization<br>**Synthesis & Rendering** The AI interprets the math into a business narrative, while QuickChart renders the data points into a visual trend graph PNG. |
| Combine Data for Chart | Merge | Merge cleaned data + AI output | Format AI Output; Data Cleaning | Prepare Chart Config | ## 3. AI Intelligence & Visualization<br>**Synthesis & Rendering** The AI interprets the math into a business narrative, while QuickChart renders the data points into a visual trend graph PNG. |
| Prepare Chart Config | Code | Build Chart.js config with forecast dot/segment | Combine Data for Chart | QuickChart | ## 3. AI Intelligence & Visualization<br>**Synthesis & Rendering** The AI interprets the math into a business narrative, while QuickChart renders the data points into a visual trend graph PNG. |
| QuickChart | HTTP Request | Render PNG chart from chart config | Prepare Chart Config | Upsert Latest Forecast; Send WhatsApp Report | ## 3. AI Intelligence & Visualization<br>**Synthesis & Rendering** The AI interprets the math into a business narrative, while QuickChart renders the data points into a visual trend graph PNG. |
| Upsert Latest Forecast | Data Table | Save latest forecast context for Q&A | QuickChart | — | ## 4. Delivery & Storage<br>**Notification System** Saves the forecast context to the database for future reference and pushes the final Report + Chart to WhatsApp. |
| Send WhatsApp Report | WhatsApp | Send chart + executive caption to phone | QuickChart | — | ## 4. Delivery & Storage<br>**Notification System** Saves the forecast context to the database for future reference and pushes the final Report + Chart to WhatsApp. |
| WhatsApp Trigger | whatsAppTrigger | Entry point for inbound WhatsApp messages | — | Filter Text Messages | ## 5. Follow-Up Q&A with Sales Forecast AI Agent<br>**Data Consultant** Retrieves the stored forecast data to answer user questions, acting as an on-demand analyst with full knowledge of the report. |
| Filter Text Messages | IF | Allowlist sender filter | WhatsApp Trigger | Normalize Whatsapp Input | ## 5. Follow-Up Q&A with Sales Forecast AI Agent<br>**Data Consultant** Retrieves the stored forecast data to answer user questions, acting as an on-demand analyst with full knowledge of the report. |
| Normalize Whatsapp Input | Set | Extract user message text into `chatInput` | Filter Text Messages | Get Latest Forecast | ## 5. Follow-Up Q&A with Sales Forecast AI Agent<br>**Data Consultant** Retrieves the stored forecast data to answer user questions, acting as an on-demand analyst with full knowledge of the report. |
| Get Latest Forecast | Data Table | Fetch stored “latest” forecast row | Normalize Whatsapp Input | Parsing Input | ## 5. Follow-Up Q&A with Sales Forecast AI Agent<br>**Data Consultant** Retrieves the stored forecast data to answer user questions, acting as an on-demand analyst with full knowledge of the report. |
| Parsing Input | Code | Parse `historySales` JSON; normalize keys | Get Latest Forecast | Q&A AI Agent | ## 5. Follow-Up Q&A with Sales Forecast AI Agent<br>**Data Consultant** Retrieves the stored forecast data to answer user questions, acting as an on-demand analyst with full knowledge of the report. |
| OpenAI Chat Model2 | lmChatOpenAi | LLM for Q&A agent | — | Q&A AI Agent (ai_languageModel) | ## 5. Follow-Up Q&A with Sales Forecast AI Agent<br>**Data Consultant** Retrieves the stored forecast data to answer user questions, acting as an on-demand analyst with full knowledge of the report. |
| Simple Memory | memoryBufferWindow | Conversation memory per sender | — | Q&A AI Agent (ai_memory) | ## 5. Follow-Up Q&A with Sales Forecast AI Agent<br>**Data Consultant** Retrieves the stored forecast data to answer user questions, acting as an on-demand analyst with full knowledge of the report. |
| Q&A AI Agent | LangChain Agent | Answer questions using latest forecast + history | Parsing Input; OpenAI Chat Model2; Simple Memory | Q&A Message | ## 5. Follow-Up Q&A with Sales Forecast AI Agent<br>**Data Consultant** Retrieves the stored forecast data to answer user questions, acting as an on-demand analyst with full knowledge of the report. |
| Q&A Message | WhatsApp | Send text answer to WhatsApp | Q&A AI Agent | — | ## 5. Follow-Up Q&A with Sales Forecast AI Agent<br>**Data Consultant** Retrieves the stored forecast data to answer user questions, acting as an on-demand analyst with full knowledge of the report. |
| Sticky Note15 | stickyNote | Workspace documentation | — | — | ## How It Works … (as displayed in canvas) |
| Sticky Note17 | stickyNote | Prerequisites note | — | — | ## Prerequisites<br>Data: At least 12 periods of historical sales data. |
| Sticky Note18 | stickyNote | Use cases & benefits note | — | — | ## Use Cases & Benefits<br>**Business Owners:** … (as displayed in canvas) |
| Sticky Note | stickyNote | Block label: Data Pipeline | — | — | ## 1. Data Pipeline … |
| Sticky Note1 | stickyNote | Block label: Forecasting | — | — | ## 2. Forecasting … |
| Sticky Note2 | stickyNote | Block label: AI + Visualization | — | — | ## 3. AI Intelligence & Visualization … |
| Sticky Note3 | stickyNote | Block label: Delivery & Storage | — | — | ## 4. Delivery & Storage … |
| Sticky Note4 | stickyNote | Block label: Q&A | — | — | ## 5. Follow-Up Q&A … |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Trigger (monthly)**
   - Add **Schedule Trigger** node named `Monthly Schedule`.
   - Set interval to **Months**, trigger at **09:00**.

2) **Add configuration node**
   - Add **Set** node `Workflow Configuration`.
   - Add fields:
     - `spreadsheetId` (string) e.g. `SalesSheet`
     - `sheetName` (string) e.g. `Sales`
   - Connect: `Monthly Schedule` → `Workflow Configuration`.

3) **Read historical sales from Google Sheets**
   - Add **Google Sheets** node `Get Sales Data`.
   - Authentication: **Service Account**.
   - Select the Spreadsheet and Sheet containing columns:
     - `Year`, `Month`, `MonthlySales` (or adjust cleaning code)
   - Connect: `Workflow Configuration` → `Get Sales Data`.
   - **Credential setup:** Create/attach a Google Service Account credential that has access to the sheet.

4) **Normalize/clean the data**
   - Add **Code** node `Data Cleaning` (Run once for all items).
   - Paste logic to:
     - parse `Year`, `Month`, `MonthlySales`
     - output one item `{ data: [...], rowCount }` with `Sales` and `period`
   - Connect: `Get Sales Data` → `Data Cleaning`.

5) **Forecast tournament**
   - Add **Code** node `Forecast Engine`.
   - Paste the deterministic forecasting code (seasonal naive/avg, MA, linear trend, SES, seasonal regression, seasonal regression + trend).
   - Connect: `Data Cleaning` → `Forecast Engine`.

6) **Add OpenAI model + structured parser (analyst)**
   - Add **OpenAI Chat Model** node `OpenAI Chat Model1`.
     - Model: `gpt-5`
     - Timeout: 180000 ms
     - Attach OpenAI API credential.
   - Add **Structured Output Parser** node `Structured Output Parser`.
     - Configure with the provided JSON schema (manual).
   - Add **AI Agent** node `Analyst AI Agent`.
     - Input text: `{{$json}}`
     - System message: the business-advisor instruction set (including “do not change forecastedSales” and schema-only output).
     - Enable output parsing and select `Structured Output Parser`.
   - Connect:
     - `Forecast Engine` → `Analyst AI Agent`
     - `OpenAI Chat Model1` → `Analyst AI Agent` (language model connection)
     - `Structured Output Parser` → `Analyst AI Agent` (output parser connection)

7) **Extract summary caption**
   - Add **Set** node `Format AI Output`.
   - Set `summaryReport = {{$json.output.reasoning}}`, include other fields.
   - Connect: `Analyst AI Agent` → `Format AI Output`.

8) **Merge AI output with historical data**
   - Add **Merge** node `Combine Data for Chart`.
   - Mode: **Combine by position**.
   - Connect:
     - `Format AI Output` → `Combine Data for Chart` (Input 1)
     - `Data Cleaning` → `Combine Data for Chart` (Input 2)

9) **Build chart config**
   - Add **Code** node `Prepare Chart Config`.
   - Build a Chart.js line chart with:
     - Historical series + trailing null
     - Forecast dashed segment from last actual to forecast point
   - Connect: `Combine Data for Chart` → `Prepare Chart Config`.

10) **Render chart image**
   - Add **HTTP Request** node `QuickChart`.
   - Method: POST
   - URL: `https://quickchart.io/chart`
   - Response: **File**, binary property name `chart`
   - Body params: `chart={{JSON.stringify($json.chartConfig)}}`, `format=png`, `width=1200`, `height=600`, `backgroundColor=white`
   - Connect: `Prepare Chart Config` → `QuickChart`.

11) **Create Data Table for persistence**
   - In n8n, create a **Data Table** named (example) `Latest_Forecast` with columns:
     - `key` (string)
     - `forecastPeriod` (string)
     - `forecastSales` (number)
     - `summaryReport` (string)
     - `historySales` (string)
   - Add **Data Table** node `Upsert Latest Forecast`.
   - Operation: `upsert`
   - Filter: `key == "latest"`
   - Values:
     - `key = "latest"`
     - `forecastPeriod = {{$json.output.forecastHorizon}}`
     - `forecastSales = {{$json.output.forecastedSales}}`
     - `summaryReport = {{$json.summaryReport}}`
     - `historySales = {{ JSON.stringify($items("Get Sales Data").map(i => ({year:i.json.Year, month:i.json.Month, sales:i.json.MonthlySales}))) }}`
   - Connect: `QuickChart` → `Upsert Latest Forecast`.

12) **Send WhatsApp report (image + caption)**
   - Add **WhatsApp** node `Send WhatsApp Report`.
   - Operation: `send`, message type `image`
   - `mediaPropertyName = "chart"`
   - Caption: `{{$json.summaryReport}}`
   - Set `phoneNumberId` and recipient (or dynamically use inbound sender in a different design).
   - Attach WhatsApp Cloud API credential.
   - Connect: `QuickChart` → `Send WhatsApp Report`.

13) **Add WhatsApp inbound trigger**
   - Add **WhatsApp Trigger** node `WhatsApp Trigger` for updates `messages`.
   - Attach WhatsApp Trigger credential/webhook configuration.

14) **Filter allowed sender**
   - Add **IF** node `Filter Text Messages`.
   - Condition: `{{$json.messages[0].from}} equals "<allowed_number>"`
   - Connect: `WhatsApp Trigger` → `Filter Text Messages` (true output onward).

15) **Normalize inbound text**
   - Add **Set** node `Normalize Whatsapp Input`.
   - Create `chatInput` with a robust expression that checks common payload shapes and falls back to `""`.
   - Connect: `Filter Text Messages` (true) → `Normalize Whatsapp Input`.

16) **Retrieve latest forecast for Q&A**
   - Add **Data Table** node `Get Latest Forecast`.
   - Operation: `get`, filter `key == "latest"`.
   - Connect: `Normalize Whatsapp Input` → `Get Latest Forecast`.

17) **Parse stored history JSON**
   - Add **Code** node `Parsing Input`.
   - Parse `historySales` string to array; normalize keys to `{year, month, sales}`; output `historySalesParsed`.
   - Connect: `Get Latest Forecast` → `Parsing Input`.

18) **Q&A agent with memory**
   - Add **OpenAI Chat Model** node `OpenAI Chat Model2` (e.g., `gpt-4.1-nano`) with OpenAI credential.
   - Add **Memory Buffer Window** node `Simple Memory`.
     - sessionKey: `{{$('WhatsApp Trigger').item.json.messages[0].from}}`
   - Add **AI Agent** node `Q&A AI Agent`.
     - User text: `{{$('Normalize Whatsapp Input').item.json.chatInput}}`
     - System message: include forecast period, forecast sales, summary, and formatted history list using `historySalesParsed`.
   - Connect:
     - `Parsing Input` → `Q&A AI Agent`
     - `OpenAI Chat Model2` → `Q&A AI Agent` (language model)
     - `Simple Memory` → `Q&A AI Agent` (memory)

19) **Send WhatsApp reply**
   - Add **WhatsApp** node `Q&A Message` (text send).
   - Text: `{{$json.output}}`
   - Configure recipient (ideally dynamic: the inbound sender).
   - Connect: `Q&A AI Agent` → `Q&A Message`.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “## How It Works … (Top Branch Workflow / Bottom Branch Workflow) … Setup Steps (Google Sheet columns, Forecast Engine no config, Database table latest_forecast, Credentials)” | From the canvas sticky note “Sticky Note15”. |
| “## Prerequisites — Data: At least 12 periods of historical sales data.” | From sticky note “Sticky Note17”. Note: the Forecast Engine actually hard-requires **13+** months; it recommends **24+** for seasonality reliability. |
| “## Use Cases & Benefits … Virtual Data Team … Precision & Trust … Decision-Ready Insights …” | From sticky note “Sticky Note18”. |

