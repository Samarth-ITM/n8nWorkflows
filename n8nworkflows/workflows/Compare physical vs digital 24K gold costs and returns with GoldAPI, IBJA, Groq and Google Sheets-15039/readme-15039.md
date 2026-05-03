Compare physical vs digital 24K gold costs and returns with GoldAPI, IBJA, Groq and Google Sheets

https://n8nworkflows.xyz/workflows/compare-physical-vs-digital-24k-gold-costs-and-returns-with-goldapi--ibja--groq-and-google-sheets-15039


# Compare physical vs digital 24K gold costs and returns with GoldAPI, IBJA, Groq and Google Sheets

# Technical Analysis: Physical vs Digital Gold Investment Comparison

## 1. Workflow Overview
This workflow is an automated financial analysis tool designed to compare the landed cost of 1 gram of 24K gold (Digital vs. Physical) in the Indian market. It integrates real-time API data, web scraping, market news aggregation, and Generative AI to provide a daily "Buy/Wait" recommendation and sentiment analysis.

The logic is organized into five functional blocks:
- **1.1 Digital Price Retrieval:** Fetches live digital gold rates and validates API connectivity.
- **1.2 Physical Price Extraction:** Scrapes benchmark rates from the official IBJA website.
- **1.3 Landed Cost Calculation:** Computes the final cost including GST and making charges.
- **1.4 Market Context & AI Analysis:** Aggregates financial news and uses an LLM to generate a sentiment score.
- **1.5 Report Formatting & Storage:** Prepares a human-readable summary and logs the data to Google Sheets.

---

## 2. Block-by-Block Analysis

### 2.1 Digital Price Retrieval & Validation
**Overview:** Retrieves the current 24K gold price in INR and ensures the necessary credentials are present.
- **Nodes Involved:** `When clicking ‘Execute workflow’`, `Get Digital Price`, `check for result`, `Add your access token`.
- **Node Details:**
    - **When clicking ‘Execute workflow’**: Manual Trigger to start the process.
    - **Get Digital Price**: HTTP Request to `goldapi.io`. Uses a header `x-access-token` for authentication.
    - **check for result**: IF node that checks if the API response contains the error "No API Key provided".
    - **Add your access token**: NoOp node acting as a dead-end/alert if the API key is missing.
- **Failure Points:** Invalid or expired GoldAPI token; API rate limits.

### 2.2 Physical Price Extraction
**Overview:** Obtains the benchmark physical gold price by scraping the IBJA (India Bullion and Jewellers Association) website.
- **Nodes Involved:** `Get IBJA Website`, `Extract Physical Price`.
- **Node Details:**
    - **Get IBJA Website**: HTTP Request fetching the HTML content of `ibjarates.com`.
    - **Extract Physical Price**: HTML node using the CSS selector `#GoldRatesCompare999` to isolate the 24K 1-gram price.
- **Failure Points:** Change in the IBJA website DOM structure (CSS selector becomes invalid); Website downtime.

### 2.3 Landed Cost Calculation (Comparator)
**Overview:** Performs the "Market Math" to normalize prices by adding real-world taxes and fees.
- **Nodes Involved:** `Comparator`.
- **Node Details:**
    - **Comparator**: Code node (JavaScript).
    - **Logic**:
        - **Digital**: `Spot Price * 1.03 (GST) * 1.03 (Platform Spread)`.
        - **Physical**: `IBJA Price * 1.03 (GST) * 1.08 (Making Charges)`.
    - **Output**: Returns `Digital_1g_Final`, `Physical_1g_Final`, the absolute difference, and the `Cheaper_Option`.
- **Failure Points:** Non-numeric characters in scraped data (handled via regex `.replace(/,/g, '')`).

### 2.4 Market Context & AI Analysis
**Overview:** Combines quantitative price data with qualitative market news to produce a financial verdict.
- **Nodes Involved:** `Get Market News`, `News Limit`, `Data Array`, `AI Agent`, `Groq Chat Model`.
- **Node Details:**
    - **Get Market News**: RSS Feed node fetching headlines from Yahoo Finance (`GC=F` - Gold Futures).
    - **News Limit**: Limits input to the top 3 headlines to avoid token overflow.
    - **Data Array**: Aggregates multiple news items into a single array for the AI.
    - **AI Agent**: LangChain agent. The system prompt instructs the AI to act as a "Quantitative Financial Analyst" and output a strict JSON object containing `Arbitrage_Check`, `Sentiment`, and `Efficiency_Score`.
    - **Groq Chat Model**: Provides the LLM backbone (using `openai/gpt-oss-20b` via Groq).
- **Failure Points:** LLM hallucination; JSON formatting errors in AI output; Groq API timeouts.

### 2.5 Report Formatting & Storage
**Overview:** Transforms the AI's raw JSON and the calculated math into a final report and archives it.
- **Nodes Involved:** `Message Format`, `Edit Fields`, `Add Report`.
- **Node Details:**
    - **Message Format**: Code node that parses the AI response and constructs a formatted string for messaging apps (e.g., Telegram/WhatsApp).
    - **Edit Fields**: Set node that maps the formatted data into specific keys for database compatibility.
    - **Add Report**: Google Sheets node that appends a new row to the specified document.
- **Failure Points:** Google Sheets permission errors; Mapping mismatch between `Edit Fields` and the Sheet columns.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| When clicking ‘Execute workflow’ | Manual Trigger | Workflow Entry | - | Get Digital Price | Gold Price Analysis (Physical vs Digital) |
| Get Digital Price | HTTP Request | Fetch digital gold rate | Manual Trigger | check for result | Digital Price Retrieval & Validation |
| check for result | IF | Validate API Key | Get Digital Price | Add your access token / Get IBJA Website | Digital Price Retrieval & Validation |
| Add your access token | NoOp | Error handling | check for result | - | Digital Price Retrieval & Validation |
| Get IBJA Website | HTTP Request | Fetch IBJA HTML | check for result | Extract Physical Price | Physical Price Extraction |
| Extract Physical Price | HTML | Parse 24K price | Get IBJA Website | Comparator | Physical Price Extraction |
| Comparator | Code | Tax/Fee Calculation | Extract Physical Price | Get Market News | Landed Cost Calculation (Comparator) |
| Get Market News | RSS Feed | Fetch gold news | Comparator | News Limit | Market Context & News Aggregation |
| News Limit | Limit | Filter top 3 news | Get Market News | Data Array | Market Context & News Aggregation |
| Data Array | Aggregate | Combine news items | News Limit | AI Agent | Market Context & News Aggregation |
| AI Agent | AI Agent | Quantitative Analysis | Data Array | Message Format | Market Context & News Aggregation |
| Groq Chat Model | LLM | AI Processing | - | AI Agent | Market Context & News Aggregation |
| Message Format | Code | Text formatting | AI Agent | Edit Fields | Report Formatting & Storage |
| Edit Fields | Set | Data normalization | Message Format | Add Report | Report Formatting & Storage |
| Add Report | Google Sheets | Data archival | Edit Fields | - | Report Formatting & Storage |

---

## 4. Reproducing the Workflow from Scratch

### Step 1: Digital Price Acquisition
1. Create a **Manual Trigger**.
2. Add an **HTTP Request** node:
    - URL: `https://www.goldapi.io/api/XAU/INR`
    - Headers: Add `x-access-token` with your GoldAPI key.
3. Add an **IF** node to check if `$json.error` equals "No API Key provided".
4. (Optional) Connect the TRUE branch to a **NoOp** node for error signaling.

### Step 2: Physical Price Acquisition
1. Add an **HTTP Request** node (connected to the FALSE branch of the IF node):
    - URL: `https://ibjarates.com/`
2. Add an **HTML** node:
    - Operation: `Extract HTML Content`.
    - Extraction Value: Key `IBJA_Price`, CSS Selector `#GoldRatesCompare999`.

### Step 3: Landed Cost Math
1. Add a **Code** node named `Comparator`. Use the provided JavaScript logic:
    - Fetch `price_gram_24k` from "Get Digital Price".
    - Fetch `IBJA_Price` from "Extract Physical Price".
    - Apply multipliers: Digital (`1.03 * 1.03`) and Physical (`1.03 * 1.08`).
    - Output: `Digital_1g_Final`, `Physical_1g_Final`, `Price_Difference_Per_Gram`, `Cheaper_Option`.

### Step 4: AI Intelligence Integration
1. Add an **RSS Feed Read** node:
    - URL: `https://finance.yahoo.com/rss/headline?s=GC=F`
2. Add a **Limit** node: Set Max Items to `3`.
3. Add an **Aggregate** node: Set to `Aggregate All Item Data`.
4. Add an **AI Agent** node:
    - Connect a **Chat Model (Groq)** node using your Groq API key.
    - System Message: Define the persona as "Elite Quantitative Financial Analyst" and require a JSON output with keys `Arbitrage_Check`, `Sentiment`, and `Efficiency_Score`.
    - Prompt: Pass the calculated costs from the `Comparator` node and the news from the `Data Array` node.

### Step 5: Reporting & Storage
1. Add a **Code** node `Message Format`: Parse the AI JSON output and build a template string (e.g., `GOLD INTELLIGENCE REPORT...`).
2. Add a **Set** node `Edit Fields`: Create fields for `Date`, `Digital Price`, `Physical_Price`, `Sentiment`, and `Efficiency_Score`.
3. Add a **Google Sheets** node:
    - Operation: `Append`.
    - Document ID: Select your specific gold tracking sheet.
    - Mapping: Map the fields from the `Edit Fields` node to the sheet columns.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Digital Gold Fee Structure: 3% GST + 3% Platform Spread | Calculation Logic |
| Physical Gold Fee Structure: 3% GST + 8% Making Charges | Calculation Logic |
| Official Physical Gold Benchmark | [ibjarates.com](https://ibjarates.com/) |
| Digital Gold API Provider | [GoldAPI.io](https://www.goldapi.io/) |
| AI Backend | Groq (LLM) |