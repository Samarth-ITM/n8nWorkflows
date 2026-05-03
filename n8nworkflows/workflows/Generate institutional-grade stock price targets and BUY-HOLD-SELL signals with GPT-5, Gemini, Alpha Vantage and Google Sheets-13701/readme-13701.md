Generate institutional-grade stock price targets and BUY/HOLD/SELL signals with GPT-5, Gemini, Alpha Vantage and Google Sheets

https://n8nworkflows.xyz/workflows/generate-institutional-grade-stock-price-targets-and-buy-hold-sell-signals-with-gpt-5--gemini--alpha-vantage-and-google-sheets-13701


# Generate institutional-grade stock price targets and BUY/HOLD/SELL signals with GPT-5, Gemini, Alpha Vantage and Google Sheets

This reference document provides a detailed technical breakdown of the **AI Institutional Stock Valuation Engine** workflow. This system automates fundamental financial analysis, risk scoring (Piotroski F-Score), and sentiment analysis to generate professional-grade price targets and investment signals.

---

### 1. Workflow Overview

The workflow is a sophisticated equity research pipeline that mimics the behavior of a senior institutional analyst. It operates in a loop to process a portfolio of stocks, deciding whether to fetch fresh data from Alpha Vantage or use cached data from Google Sheets to optimize API costs.

**Logical Blocks:**
*   **1.1 Data Orchestration:** Triggers the workflow every 3 days and manages the iteration over stock tickers.
*   **1.2 Intelligence Cache:** Checks a Google Sheet "Cache" to determine if financial data is fresh (under 7 days old) or needs updating.
*   **1.3 Multi-Source Data Harvesting:** Fetching real-time Balance Sheets, Income Statements, Cash Flow, and Market Prices via Alpha Vantage, alongside recent Analyst News via Seeking Alpha.
*   **1.4 Quantitative Engineering:** A complex JavaScript engine calculates the **Piotroski F-Score** and normalizes financial metrics.
*   **1.5 Dual-Model AI Valuation:** Simultaneously consults **GPT-5** and **Gemini 1.5 Pro** using institutional prompts to generate Bear, Base, and Bull price targets.
*   **1.6 Results Persistence:** Consolidates AI opinions and appends the final valuation report to a Google Sheet.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Input
This block initiates the process and retrieves the list of target companies.
*   **Nodes Involved:** `Schedule Trigger`, `Read_tickers_from_Sheet`, `loop_over_tickers`.
*   **Node Details:**
    *   **Schedule Trigger:** Runs every 3 days at 4:00 PM.
    *   **Read_tickers_from_Sheet:** Connects to a Google Sheet to fetch a list of tickers.
    *   **loop_over_tickers:** A "Split in Batches" node that ensures each stock is processed individually to stay within API rate limits.

#### 2.2 Financial Data & Cache Logic
Determines if expensive API calls are necessary.
*   **Nodes Involved:** `Cache Lookup`, `Check the cache`, `Is Cache Valid?`, `Get row(s) in sheet`.
*   **Node Details:**
    *   **Check the cache (Code):** Calculates the age of the data in days. Sets `shouldFetch` to `true` if data is > 7 days old.
    *   **Is Cache Valid? (If):** Routes the workflow: "True" leads to Alpha Vantage APIs; "False" leads to `Get row(s) in sheet` to reuse existing data.

#### 2.3 Financial API Harvesting (Alpha Vantage)
Fetches the raw quantitative data required for valuation.
*   **Nodes Involved:** `alphavantage - Balance Sheet`, `Income Statement`, `Profile`, `CashFlow`, `Current Price`.
*   **Node Details:**
    *   **Configuration:** All use HTTP Request nodes targeting `alphavantage.co`.
    *   **Clean [Nodes] (Code):** Multiple small JavaScript nodes that wrap raw API responses into structured JSON objects (e.g., `balance_data`, `income_data`) to prevent variable name collisions during merging.

#### 2.4 Quantitative Analysis & Storage
Calculates the risk score and updates the cache.
*   **Nodes Involved:** `Clean Read Financial`, `If2`, `Update row in sheet`, `Insert Row`.
*   **Node Details:**
    *   **Clean Read Financial (Code):** The "Engine" of the workflow. It computes the **Piotroski F-Score (0-9)** by analyzing profitability, leverage, and operating efficiency. It also calculates TTM (Trailing Twelve Months) revenue growth and margins.
    *   **Storage Logic:** Uses `If2` to decide whether to `Update` an existing row in the cache sheet or `Append` (Insert) a new one for a new ticker.

#### 2.5 Qualitative News Context
Gathers market sentiment from high-credibility sources.
*   **Nodes Involved:** `Seekingalpha Articles`, `Clean the news`, `Final version of news`.
*   **Node Details:**
    *   **Seekingalpha Articles:** Fetches the RSS feed for a specific ticker.
    *   **Final version of news (Code):** Deduplicates articles and formats them into a compact text block for the AI prompt, prioritizing news from the last 80 hours.

#### 2.6 AI Valuation Analysis (The "Brain")
Simulates an institutional investment committee.
*   **Nodes Involved:** `Message a model` (OpenAI), `Message a model1` (Gemini), `Clean Up ChatGPT`, `Clean up Gemini`, `Merge`.
*   **Node Details:**
    *   **Prompt Logic:** A 150-line "Institutional" prompt. It enforces:
        *   **Graham Number** check for mature stocks.
        *   **Growth-based anchors** for transition companies.
        *   **Specific Banding:** Bear targets must be 15-35% discounts based on F-Score.
    *   **Format:** Enforces Strict JSON output to ensure the data can be written back to the spreadsheet.

#### 2.7 Data Output
Finalizes the report.
*   **Nodes Involved:** `Code in JavaScript` (Post-Processing), `write_sentiment_to_sheets`, `Send a text message` (Telegram).
*   **Node Details:**
    *   **Code in JavaScript:** Deduplicates results between the two AI models. It picks the version with the highest confidence score and formats numbers to 2 decimal places.
    *   **write_sentiment_to_sheets:** Appends the date, stock, price targets, and rationale to the final tracking sheet.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Schedule Trigger | Schedule | Daily Trigger | None | Read_tickers_from_Sheet | Run automatically every 3 days at 4:00 PM. |
| Read_tickers_from_Sheet | Google Sheets | Ticker Retrieval | Schedule Trigger | loop_over_tickers | Source of stocks to analyze. |
| loop_over_tickers | Split in Batches | Iteration Logic | Read_tickers_from_Sheet | Cache Lookup, News | Processes tickers one by one. |
| Cache Lookup | Google Sheets | Cache Check | loop_over_tickers | Check the cache | Check if financial data exists. |
| Clean Read Financial | Code | Quant Engine | Merge 4 | If2 | Calculates F-Score and TTM metrics. |
| Message a model | OpenAI | AI Analyst | Merge 1 | Clean Up ChatGPT | Uses GPT-5 for valuation. |
| Message a model1 | Gemini | AI Analyst | Final version of news | Clean up Gemini | Uses Gemini 1.5 Pro for valuation. |
| write_sentiment_to_sheets | Google Sheets | Data Storage | If | loop_over_tickers | Appends scores and rationale to Sheet1. |
| Send a text message | Telegram | Notification | loop_over_tickers | None | Report completion alert. |

---

### 4. Reproducing the Workflow from Scratch

1.  **Google Sheets Setup:**
    *   Create a "Tickers" sheet with a column named `stock`.
    *   Create a "Financial_Data_Cache" sheet with columns for `stock`, `f_score`, `last_updated`, `current_price`, and other metrics.
    *   Create a "Results" sheet with columns for `date`, `stock`, `pt_base`, `pt_bear`, `pt_bull`, `rationale`, and `confidence`.
2.  **Ticker Loop:**
    *   Add a **Google Sheets (Download)** node followed by a **Split in Batches** node.
3.  **Alpha Vantage Setup:**
    *   Obtain a free/premium API Key.
    *   Create 5 **HTTP Request** nodes (GET). Endpoints: `OVERVIEW`, `INCOME_STATEMENT`, `BALANCE_SHEET`, `CASH_FLOW`, `GLOBAL_QUOTE`.
    *   Use the expression `{{ $node["loop_over_tickers"].json["stock"] }}` in the URL.
4.  **Quantitative Code:**
    *   Create a **Code** node. Implement logic to extract `annualReports[0]` vs `annualReports[1]` to calculate the Piotroski factors (e.g., `Net Income > 0`, `Current Ratio Increase`).
5.  **Seeking Alpha RSS:**
    *   Add an **HTTP Request** node targeting `https://seekingalpha.com/api/sa/combined/TICKER.xml`.
    *   Use an **XML** node to parse the response.
6.  **AI Integration:**
    *   Add **OpenAI** and **Google Gemini** nodes.
    *   **Prompt:** Paste the institutional analyst prompt (provided in the JSON). Ensure it references financial variables from the "Clean Read Financial" node and news from the "Final news" node.
    *   Set **Response Format** to `json_object`.
7.  **Final Logic:**
    *   Add a **Code** node to clean strings (removing markdown ` ```json ` tags if present).
    *   Add a **Google Sheets (Append)** node to save the final row.
    *   Connect the final node back to the **Split in Batches** node to process the next ticker.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Alpha Vantage API Limits** | Free tier is limited to 25 requests per day; workflow is set to 3-day intervals to accommodate. |
| **Seeking Alpha RSS** | Public RSS feeds are used to avoid scraping complexities. |
| **Piotroski F-Score** | Used for quality filtering. Scores < 3 trigger a risk-adjusted discount in AI valuation. |
| **Graham Number** | Formula: `sqrt(22.5 * EPS * BVPS)`. Only applied to mature profitable firms. |