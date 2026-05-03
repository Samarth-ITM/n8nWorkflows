Gold Market Prediction System with Perplexity Sonar-Pro, FRED Data, and WordPress Reporting

https://n8nworkflows.xyz/workflows/gold-market-prediction-system-with-perplexity-sonar-pro--fred-data--and-wordpress-reporting-10379


# Gold Market Prediction System with Perplexity Sonar-Pro, FRED Data, and WordPress Reporting

---
### 1. Workflow Overview

This workflow automates the collection, analysis, and reporting of gold market conditions, integrating live gold price data, financial news, and key macroeconomic indicators to produce AI-driven market forecasts and insights. It targets financial analysts, investors, and market watchers aiming to track gold price trends and underlying economic factors with up-to-date, data-driven intelligence.

Logical blocks:

- **1.1 Scheduling and Data Acquisition:** Periodically triggers the workflow and fetches live market data and macroeconomic indicators from multiple APIs.
- **1.2 Data Formatting and Merging:** Normalizes and consolidates heterogeneous data sources into a structured dataset prepared for AI analysis.
- **1.3 AI Analysis:** Feeds the combined data into an AI agent specialized in financial markets to generate detailed forecasts and trend insights.
- **1.4 Reporting and Publishing:** Formats the AI output into a final report, publishes it to WordPress, and sends notifications via Slack and email.
- **1.5 Supporting Components:** Includes trigger scheduling, API authentication setup, and utility notes explaining workflow usage and prerequisites.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduling and Data Acquisition

- **Overview:** This block triggers the workflow every 6 hours and collects current gold prices, financial news, inflation, interest rate, and employment data from external APIs.

- **Nodes Involved:**  
  - Every 6 Hours  
  - Get Current Gold Price  
  - Fetch Financial News  
  - Get Inflation Data  
  - Get Interest Rate Data  
  - Get Employment Data  

- **Node Details:**

  - **Every 6 Hours**  
    - Type: Schedule Trigger  
    - Role: Initiates the workflow at 6-hour intervals automatically.  
    - Configuration: Set to run every 6 hours using the built-in cron-like scheduling.  
    - Inputs: None  
    - Outputs: Triggers all data retrieval nodes simultaneously.  
    - Edge Cases: Missed execution if n8n instance downtime; ensure persistent execution enabled.

  - **Get Current Gold Price**  
    - Type: HTTP Request  
    - Role: Fetches latest gold price from MetalPriceAPI.  
    - Configuration: HTTP GET to `https://api.metalpriceapi.com/v1/latest` with query parameters including API key, base currency USD, and target currency XAU (gold).  
    - Credentials: Requires MetalPriceAPI API key configured as generic query authentication.  
    - Inputs: Trigger from schedule node  
    - Outputs: Raw JSON with current gold price and timestamp.  
    - Edge Cases: API rate limits, network errors, invalid API key, response format changes.

  - **Fetch Financial News**  
    - Type: HTTP Request  
    - Role: Retrieves recent financial news articles relevant to gold market and macroeconomic themes.  
    - Configuration: HTTP GET to NewsAPI `everything` endpoint with query for gold market-related keywords, English language, sorted by recent publication, max 20 articles.  
    - Credentials: NewsAPI API key as generic query authentication.  
    - Inputs: Trigger from schedule node  
    - Outputs: News articles JSON array containing titles, sources, timestamps, descriptions, and URLs.  
    - Edge Cases: API key quota exceeded, no recent articles, network latency.

  - **Get Inflation Data**  
    - Type: HTTP Request  
    - Role: Fetches Consumer Price Index (CPI) data from FRED API to infer inflation trends.  
    - Configuration: API call to FRED series "CPIAUCSL", last 12 observations, JSON format, descending order.  
    - Credentials: FRED API key as a query parameter.  
    - Inputs: Trigger from schedule node  
    - Outputs: JSON array of recent inflation data points with dates and values.  
    - Edge Cases: FRED API downtime, invalid API key, data unavailability.

  - **Get Interest Rate Data**  
    - Type: HTTP Request  
    - Role: Retrieves Federal Funds Rate data series from FRED.  
    - Configuration: API call to FRED series "FEDFUNDS", parameters same as inflation data node.  
    - Credentials: Same as above.  
    - Inputs: Trigger from schedule node  
    - Outputs: JSON array of recent interest rate observations.  
    - Edge Cases: Same as inflation data node.

  - **Get Employment Data**  
    - Type: HTTP Request  
    - Role: Fetches unemployment rate data from FRED.  
    - Configuration: API call to FRED series "UNRATE" with similar parameters as above.  
    - Credentials: Same as above.  
    - Inputs: Trigger from schedule node  
    - Outputs: JSON array with unemployment rate observations.  
    - Edge Cases: Same as inflation data node.

---

#### 2.2 Data Formatting and Merging

- **Overview:** This block structures and normalizes the raw data into concise, AI-friendly formats, then merges all datasets into a single combined JSON object for analysis.

- **Nodes Involved:**  
  - Format Gold Price  
  - Format News Data  
  - Format Inflation  
  - Format Interest Rates  
  - Format Employment  
  - Merge All Data  
  - Prepare AI Input  

- **Node Details:**

  - **Format Gold Price**  
    - Type: Set Node  
    - Role: Extracts gold price and timestamp into named fields `gold_price_usd` (number) and `price_timestamp` (string).  
    - Configuration: Assigns values from JSON path `$json.rates.XAU` and `$json.timestamp`.  
    - Inputs: Output from Get Current Gold Price  
    - Outputs: A simplified JSON object with key gold price data.  
    - Edge Cases: Missing or malformed API response.

  - **Format News Data**  
    - Type: Set Node  
    - Role: Extracts top 10 news articles with title, source, publication date, description, and URL into an array named `news_summary`.  
    - Configuration: Maps input articles into concise objects, slices to 10.  
    - Inputs: Output from Fetch Financial News  
    - Outputs: JSON with array of summarized news articles.  
    - Edge Cases: Empty articles array, incomplete article fields.

  - **Format Inflation**  
    - Type: Set Node  
    - Role: Selects 6 most recent inflation observations, parsing values as numbers, stores as array `inflation_trend`.  
    - Inputs: Output from Get Inflation Data  
    - Outputs: Array of date-value pairs for inflation.  
    - Edge Cases: Missing observations, non-numeric values.

  - **Format Interest Rates**  
    - Type: Set Node  
    - Role: Similar to inflation formatting but for Federal Funds Rate as array `fed_funds_rate`.  
    - Inputs: Output from Get Interest Rate Data  
    - Outputs: Array of date-value pairs representing interest rates.  
    - Edge Cases: Same as inflation node.

  - **Format Employment**  
    - Type: Set Node  
    - Role: Formats unemployment data into array `unemployment_rate` with dates and numeric values.  
    - Inputs: Output from Get Employment Data  
    - Outputs: Array of unemployment rate observations.  
    - Edge Cases: Same as inflation node.

  - **Merge All Data**  
    - Type: Merge Node (combine mode)  
    - Role: Combines all formatted datasets into a single composite JSON object.  
    - Inputs: All formatting nodes (Format Gold Price, Format News Data, Format Inflation, Format Interest Rates, Format Employment) connected as inputs with separate input indexes.  
    - Outputs: Single merged data object for AI input preparation.  
    - Edge Cases: Missing data from any input stream results in incomplete merge.

  - **Prepare AI Input**  
    - Type: Set Node  
    - Role: Constructs a detailed multi-section string summarizing all merged data for AI consumption, including gold price, news summaries, and macroeconomic indicators formatted as strings.  
    - Inputs: Output from Merge All Data  
    - Outputs: Field `analysis_input` containing formatted string with embedded dynamic data references from previous nodes.  
    - Edge Cases: Expression failures if any referenced data is missing or malformed.

---

#### 2.3 AI Analysis

- **Overview:** This block uses a LangChain AI agent powered by a specialized OpenRouter Chat Model (Perplexity Sonar-Pro) to analyze the prepared data and generate a structured gold market forecast report.

- **Nodes Involved:**  
  - AI Agent - Gold Market Analysis  
  - OpenRouter Chat Model  

- **Node Details:**

  - **AI Agent - Gold Market Analysis**  
    - Type: LangChain Agent Node  
    - Role: Receives the formatted data string and prompts an expert financial analysis with structured output including executive summary, key drivers, short- and medium-term price predictions, and a watchlist of upcoming events.  
    - Configuration:  
      - System message defines the agent as a senior financial analyst specializing in precious metals with over 15 years’ experience.  
      - Prompt includes detailed instructions on output structure and reasoning requirements.  
    - Inputs: From Prepare AI Input node (field `analysis_input`)  
    - Outputs: Detailed AI-generated text report (`output` field).  
    - Sub-node: Uses `OpenRouter Chat Model` as underlying language model.  
    - Edge Cases: API latency, token limits, prompt misinterpretation, or incomplete data causing poor output.

  - **OpenRouter Chat Model**  
    - Type: LangChain Language Model  
    - Role: Language model configured to use Perplexity Sonar-Pro via OpenRouter API for inference.  
    - Parameters: Model name `perplexity/sonar-pro` with default options.  
    - Credentials: Uses configured OpenRouter API key credential.  
    - Inputs: Request from AI Agent node.  
    - Outputs: AI language model response text.  
    - Edge Cases: API key expiry, rate limits, network issues.

---

#### 2.4 Reporting and Publishing

- **Overview:** Formats the AI output into a final report object, then publishes it as a WordPress post and sends notifications via Slack and email.

- **Nodes Involved:**  
  - Format Final Report  
  - Publish to WordPress  
  - Send Slack Summary  
  - Send Email Summary  

- **Node Details:**

  - **Format Final Report**  
    - Type: Set Node  
    - Role: Packages AI output text into `final_report`, adds timestamp and current gold price for reporting.  
    - Inputs: AI Agent - Gold Market Analysis output  
    - Outputs: Structured object ready for publishing and notifications.  
    - Edge Cases: Timestamp generation failure unlikely; missing AI output.

  - **Publish to WordPress**  
    - Type: WordPress Node  
    - Role: Creates a new WordPress post with the generated report as content.  
    - Configuration:  
      - Title includes current date/time.  
      - Post status set to "publish".  
      - Author ID 1 (default admin).  
    - Inputs: From Format Final Report  
    - Credentials: WordPress OAuth2 or Application Password credentials required.  
    - Outputs: WordPress post metadata including post URL.  
    - Edge Cases: Authentication failure, WordPress API downtime, post creation errors.

  - **Send Slack Summary**  
    - Type: Slack Node  
    - Role: Sends a Slack message summarizing the report publication and key highlights from the executive summary.  
    - Configuration:  
      - Uses Slack webhook ID for authorization.  
      - Message dynamically includes current gold price, generation time, and executive summary snippet extracted via string manipulation of AI output.  
    - Inputs: From Publish to WordPress (for link) and Format Final Report (for gold price), AI Agent node (for text snippet).  
    - Edge Cases: Slack webhook failure, message formatting errors.

  - **Send Email Summary**  
    - Type: Email Send Node  
    - Role: Sends an email summary of the forecast to a fixed recipient list.  
    - Configuration:  
      - Subject includes current date.  
      - From and To emails preset.  
      - Email body not explicitly set but likely includes default or attached report snippet.  
    - Inputs: From Publish to WordPress or Format Final Report (implicit).  
    - Credentials: SMTP/Gmail credentials required.  
    - Edge Cases: SMTP errors, invalid email addresses, spam filtering.

---

#### 2.5 Supporting Components

- **Sticky Notes:**  
  - Provide workflow introduction, explanation of operation, prerequisites, use cases, customization ideas, and a concise workflow template overview.  
  - Positioned visually near respective logical blocks for user guidance.

---

### 3. Summary Table

| Node Name                 | Node Type                         | Functional Role                                         | Input Node(s)                              | Output Node(s)                          | Sticky Note                                                                                                                              |
|---------------------------|----------------------------------|--------------------------------------------------------|--------------------------------------------|----------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------|
| Every 6 Hours             | Schedule Trigger                 | Triggers workflow every 6 hours                         | None                                       | Get Current Gold Price, Fetch Financial News, Get Inflation Data, Get Interest Rate Data, Get Employment Data |                                                                                                                                           |
| Get Current Gold Price    | HTTP Request                    | Fetches live gold price from MetalPriceAPI              | Every 6 Hours                              | Format Gold Price                      |                                                                                                                                           |
| Fetch Financial News      | HTTP Request                    | Retrieves recent financial news articles                 | Every 6 Hours                              | Format News Data                      |                                                                                                                                           |
| Get Inflation Data        | HTTP Request                    | Gets inflation (CPI) data from FRED                      | Every 6 Hours                              | Format Inflation                     |                                                                                                                                           |
| Get Interest Rate Data    | HTTP Request                    | Retrieves Federal Funds Rate data from FRED              | Every 6 Hours                              | Format Interest Rates                |                                                                                                                                           |
| Get Employment Data       | HTTP Request                    | Fetches unemployment rate data from FRED                 | Every 6 Hours                              | Format Employment                   |                                                                                                                                           |
| Format Gold Price         | Set                             | Normalizes gold price and timestamp                       | Get Current Gold Price                     | Merge All Data                      |                                                                                                                                           |
| Format News Data          | Set                             | Summarizes top 10 financial news articles                 | Fetch Financial News                       | Merge All Data                      |                                                                                                                                           |
| Format Inflation          | Set                             | Extracts recent inflation trend data                      | Get Inflation Data                         | Merge All Data                      |                                                                                                                                           |
| Format Interest Rates     | Set                             | Extracts recent interest rate data                         | Get Interest Rate Data                     | Merge All Data                      |                                                                                                                                           |
| Format Employment         | Set                             | Extracts recent unemployment rate data                     | Get Employment Data                        | Merge All Data                      |                                                                                                                                           |
| Merge All Data            | Merge (combine mode)             | Combines all formatted datasets into one                  | Format Gold Price, Format News Data, Format Inflation, Format Interest Rates, Format Employment | Prepare AI Input                   |                                                                                                                                           |
| Prepare AI Input          | Set                             | Builds detailed AI prompt string with combined data       | Merge All Data                             | AI Agent - Gold Market Analysis     |                                                                                                                                           |
| OpenRouter Chat Model     | LangChain Language Model         | Provides Perplexity Sonar-Pro AI model for analysis       | AI Agent - Gold Market Analysis (ai_languageModel) | AI Agent - Gold Market Analysis     |                                                                                                                                           |
| AI Agent - Gold Market Analysis | LangChain Agent            | Performs expert AI financial analysis and forecasting     | Prepare AI Input, OpenRouter Chat Model   | Format Final Report                 |                                                                                                                                           |
| Format Final Report       | Set                             | Packages AI output with timestamp and gold price          | AI Agent - Gold Market Analysis           | Publish to WordPress                |                                                                                                                                           |
| Publish to WordPress      | WordPress                       | Publishes final report as WordPress blog post             | Format Final Report                        | Send Slack Summary, Send Email Summary |                                                                                                                                           |
| Send Slack Summary        | Slack                          | Sends Slack notification with report highlights           | Publish to WordPress, Format Final Report, AI Agent - Gold Market Analysis | None                             |                                                                                                                                           |
| Send Email Summary        | Email Send                     | Emails the report summary to internal team                | Publish to WordPress                       | None                               |                                                                                                                                           |
| Sticky Note               | Sticky Note                    | Describes workflow introduction and operation             | None                                       | None                               | ## **Introduction** Automates gold market tracking using AI forecasting by collecting live prices, financial news, and macro indicators. |
| Sticky Note1              | Sticky Note                    | Lists prerequisites, use cases, customization, benefits   | None                                       | None                               | ## Prerequisites n8n v1.0+, API keys, OAuth credentials, and internet access.                                                             |
| Sticky Note2              | Sticky Note                    | Summarizes workflow template and setup instructions       | None                                       | None                               | ## **Workflow Template** Trigger → Fetch → Format → Merge → AI Analyze → Report → Publish → Notify                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node:**  
   - Add a **Schedule Trigger** node named `Every 6 Hours`.  
   - Configure to run every 6 hours using the interval option.

2. **Add Data Fetch Nodes:**  
   For each, create an **HTTP Request** node with the following:

   - `Get Current Gold Price`  
     - URL: `https://api.metalpriceapi.com/v1/latest`  
     - Method: GET  
     - Query Parameters:  
       - `api_key` from MetalPriceAPI credential  
       - `base=USD`  
       - `currencies=XAU`  
     - Authentication: Generic Query Auth with API key.  
     - Connect `Every 6 Hours` node output to this node.

   - `Fetch Financial News`  
     - URL: `https://newsapi.org/v2/everything`  
     - Method: GET  
     - Query Parameters:  
       - `apiKey` from NewsAPI credential  
       - `q=gold market OR precious metals OR inflation OR federal reserve`  
       - `language=en`  
       - `sortBy=publishedAt`  
       - `pageSize=20`  
     - Authentication: Generic Query Auth with API key.  
     - Connect `Every 6 Hours` node output to this node.

   - `Get Inflation Data`  
     - URL: `https://api.stlouisfed.org/fred/series/observations`  
     - Method: GET  
     - Query Parameters:  
       - `series_id=CPIAUCSL`  
       - `api_key` from FRED credential  
       - `file_type=json`  
       - `limit=12`  
       - `sort_order=desc`  
     - Connect `Every 6 Hours` node output to this node.

   - `Get Interest Rate Data`  
     - Same as Inflation Data node but `series_id=FEDFUNDS`.

   - `Get Employment Data`  
     - Same as Inflation Data node but `series_id=UNRATE`.

3. **Add Data Formatting Nodes:**  
   Create **Set** nodes to extract and normalize data:

   - `Format Gold Price`  
     - Extract `gold_price_usd` = `{{$json["rates"]["XAU"]}}` (number)  
     - Extract `price_timestamp` = `{{$json["timestamp"]}}` (string)  
     - Connect from `Get Current Gold Price`.

   - `Format News Data`  
     - Map articles to an array of objects with `title`, `source`, `publishedAt`, `description`, `url`.  
     - Limit to 10 articles.  
     - Connect from `Fetch Financial News`.

   - `Format Inflation`  
     - Take first 6 observations, map to `{date, value: parseFloat(value)}` array.  
     - Connect from `Get Inflation Data`.

   - `Format Interest Rates`  
     - Same as inflation node but for interest rate data.  
     - Connect from `Get Interest Rate Data`.

   - `Format Employment`  
     - Same pattern for unemployment rate data.  
     - Connect from `Get Employment Data`.

4. **Merge Formatted Data:**  
   - Add a **Merge** node named `Merge All Data`.  
   - Set mode to `Combine`.  
   - Connect all formatting nodes (Format Gold Price, Format News Data, Format Inflation, Format Interest Rates, Format Employment) as separate inputs to the merge node.

5. **Prepare AI Input:**  
   - Create a **Set** node named `Prepare AI Input`.  
   - Use expressions to build a multi-section string combining all merged data:  
     - Gold price and timestamp  
     - News articles summary (numbered list with title, source, date, description)  
     - Macroeconomic indicators (inflation, fed funds rate, unemployment) as date-value lists  
   - Connect from `Merge All Data`.

6. **Add AI Analysis Nodes:**  
   - Add a **LangChain Chat Agent** node named `AI Agent - Gold Market Analysis`.  
   - Configure with:  
     - System message defining financial analyst role and instructions.  
     - Text parameter uses `analysis_input` from `Prepare AI Input`.  
   - Add **LangChain Language Model** node named `OpenRouter Chat Model` with model `perplexity/sonar-pro`, linked as language model for the agent node.  
   - Configure OpenRouter API credentials.  
   - Connect `Prepare AI Input` to AI Agent input.  
   - Connect AI Agent node’s `ai_languageModel` input to `OpenRouter Chat Model`.

7. **Format Final Report:**  
   - Add a **Set** node named `Format Final Report`.  
   - Assign:  
     - `final_report` = AI Agent output `output` field  
     - `report_timestamp` = current timestamp (ISO format)  
     - `current_gold_price` = gold price from `Format Gold Price`  
   - Connect from AI Agent node.

8. **Publish Report:**  
   - Add a **WordPress** node named `Publish to WordPress`.  
   - Configure credentials with WordPress OAuth2 or Application Password.  
   - Set post title to `Gold Market Analysis - {{ $now.format('MMM dd, yyyy HH:mm') }}`.  
   - Set post status to publish, author ID 1.  
   - Connect from `Format Final Report`.

9. **Send Notifications:**  
   - Add a **Slack** node named `Send Slack Summary`.  
   - Configure with Slack webhook ID.  
   - Compose message including:  
     - Current gold price  
     - Report generation time  
     - Extracted executive summary snippet from AI output  
     - Link to WordPress post  
   - Connect from `Publish to WordPress` and relevant nodes for data.

   - Add an **Email Send** node named `Send Email Summary`.  
   - Configure SMTP or Gmail credentials.  
   - Set subject and recipients accordingly.  
   - Connect from `Publish to WordPress`.

10. **Add Sticky Notes:**  
    - Add informational sticky notes near respective blocks with introduction, prerequisites, usage instructions, and workflow template overview.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                             | Context or Link                                                                                             |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| Automates gold market tracking using AI forecasting by collecting live prices, financial news, and macroeconomic indicators to produce real-time insights and trend predictions for analysts and investors.                                                                | Workflow introduction sticky note.                                                                          |
| Prerequisites: n8n v1.0+, API keys (MetalPrice, NewsAPI, FRED, OpenAI, WordPress, Slack, Gmail), OAuth credentials, and internet access.                                                                                                                                 | Sticky note listing requirements.                                                                           |
| Use Cases: Investment forecasting, financial newsletter automation, and market monitoring dashboards.                                                                                                                                                                   | Sticky note on use cases.                                                                                    |
| Customization: Add cryptocurrency or stock tracking, modify AI prompts, or route summaries to Telegram, Notion, or Google Sheets.                                                                                                                                       | Sticky note on customization options.                                                                       |
| Benefits: Saves analyst time, ensures consistent insights, enhances accuracy, delivers timely AI-driven financial intelligence.                                                                                                                                          | Sticky note on workflow benefits.                                                                            |
| Workflow Steps: Trigger → Fetch → Format → Merge → AI Analyze → Report → Publish → Notify.                                                                                                                                        | Workflow template overview sticky note.                                                                     |
| Setup Instructions: Add API keys and OAuth credentials, configure scheduling and authentication, predefine WordPress post format and Slack message templates.                                                                    | Workflow setup instructions sticky note.                                                                    |

---

_Disclaimer: The text provided comes exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and publicly available._