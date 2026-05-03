E-commerce Price Monitor with Firecrawl, Claude-Sonnet AI & Telegram Alerts

https://n8nworkflows.xyz/workflows/e-commerce-price-monitor-with-firecrawl--claude-sonnet-ai---telegram-alerts-10216


# E-commerce Price Monitor with Firecrawl, Claude-Sonnet AI & Telegram Alerts

---

### 1. Workflow Overview

This workflow automates monitoring of competitor e-commerce product prices and stock levels, focusing on shoe stores. It targets retailers, pricing analysts, and business intelligence teams aiming to track and react to market changes efficiently. The workflow scrapes competitor websites, extracts structured product data using AI, compares current data with historical records, detects significant changes, logs alerts, updates historical tracking sheets, and sends notifications via Telegram.

Logical blocks:

- **1.1 Triggering and Data Scraping**: Scheduled or manual triggers initiate competitor data scraping via Firecrawl and Apify APIs.
- **1.2 AI Data Extraction and Parsing**: Uses Claude-Sonnet AI to parse raw HTML into structured JSON product data; includes cleaning and validation.
- **1.3 Historical Data Loading and Merging**: Loads historical price and stock data from Google Sheets and merges it with current scrape results.
- **1.4 Change Detection and Alert Classification**: Analyzes price, stock, rating, and review changes; assigns alert levels and summarizes changes.
- **1.5 Data Logging and Notification**: Updates historical data sheets, logs alerts separately, and sends Telegram messages about detected changes.
- **1.6 Workflow Control and Wait Logic**: Manages timing between Apify scraping and data retrieval to ensure completeness.
- **1.7 Documentation and Setup Guidance**: Sticky notes provide detailed instructions, setup notes, and data structure requirements.

---

### 2. Block-by-Block Analysis

#### 1.1 Triggering and Data Scraping

- **Overview:** Periodically triggers the workflow and scrapes competitor shoe product data from web sources.
- **Nodes Involved:**  
  - `⏰ Monitor Every 6 Hours` (Schedule Trigger)  
  - `When clicking ‘Execute workflow’` (Manual Trigger)  
  - `Scrape a url and get its content` (Firecrawl Scraper)  
  - `Start Actor Apify` (HTTP Request to Apify actor)  
  - `Pause the workflow to let the Apify scraping task finish` (Wait node)  
  - `Get Results Apify` (HTTP Request to fetch Apify results)

- **Node Details:**

  - **⏰ Monitor Every 6 Hours**  
    - Type: Schedule trigger  
    - Configured to trigger every 6 hours automatically for periodic monitoring  
    - Outputs to `📊 Read Historical Data` for data processing start  
    - Failure risks: System downtime, delayed triggers, API rate limits downstream  

  - **When clicking ‘Execute workflow’**  
    - Type: Manual trigger  
    - Allows manual starting of the workflow for testing or on-demand runs  
    - Outputs to `Message a model`, `Scrape a url and get its content`, and `Start Actor Apify` for parallel scraping tasks  

  - **Scrape a url and get its content**  
    - Type: Firecrawl scraper node (third-party)  
    - Scrapes https://www.on.com/en-sg/shop/mens/shoes for shoe product data  
    - Extracts price information as JSON format  
    - Credential: Firecrawl API key required  
    - Output flows into merging current data with historical data  
    - Possible failures: Invalid URL, scraping blocked by site, API quota exceeded, unexpected response format  

  - **Start Actor Apify**  
    - Type: HTTP POST request  
    - Starts Apify’s Google Maps Extractor actor with parameters searching for "shoe stores" worldwide  
    - Authenticated via Apify HTTP Bearer token  
    - Triggers long-running scraping task external to n8n  
    - Outputs to a wait node to pause workflow until data is ready  
    - Failure cases: API token invalid, network errors, actor errors  

  - **Pause the workflow to let the Apify scraping task finish**  
    - Type: Wait node  
    - Pauses execution for 25 seconds to allow Apify scraping to complete before fetching results  
    - No inputs other than from Start Actor Apify  
    - Potential failure: Insufficient wait time causing incomplete data retrieval  

  - **Get Results Apify**  
    - Type: HTTP GET request  
    - Fetches latest dataset results from Apify actor run  
    - Uses HTTP Bearer token authentication  
    - Outputs to merge node for combining with other data sources  
    - Failure risks: API errors, empty or invalid dataset  

#### 1.2 AI Data Extraction and Parsing

- **Overview:** Extracts structured product details from scraped HTML content using Claude-Sonnet AI, then cleans and validates the JSON data.
- **Nodes Involved:**  
  - `Message a model` (Perplexity AI call, legacy or alternative AI usage)  
  - `🤖 AI Extract Product Data using Claude-Sonnet 4.5` (Anthropic Claude-Sonnet AI)  
  - `Converts unstructured AI text into organized, usable data fields` (Code node for parsing AI output)  

- **Node Details:**

  - **Message a model**  
    - Type: Perplexity AI node  
    - Sends prompt asking for shoe price info extraction  
    - Currently configured with minimal prompt and legacy model “sonar”  
    - Outputs to `Split Out` for citation extraction (not core to main flow)  
    - Credential: Perplexity API key  
    - Failure risks: API downtime, invalid prompt, unexpected response  

  - **🤖 AI Extract Product Data using Claude-Sonnet 4.5**  
    - Type: Anthropic Claude-Sonnet AI node via Langchain integration  
    - Input: Raw HTML content from scraping nodes  
    - Prompt instructs to extract specific e-commerce shoe product fields only, outputting pure JSON (no markdown)  
    - Fields extracted: productName, currentPrice, originalPrice, currency, inStock, stockLevel, rating, reviewCount, lastUpdated, productUrl, competitorName  
    - Credential: Anthropic API key  
    - Output flows into code node for JSON parsing  
    - Failure risks: API quota exceeded, malformed AI responses, rate limits  

  - **Converts unstructured AI text into organized, usable data fields**  
    - Type: Code node (JavaScript)  
    - Parses AI response, removes markdown code blocks, attempts JSON parse with error fallback  
    - Enriches data with metadata: scrapedAt timestamp, initializes price change tracking and alert flags  
    - On parse failure adds error item for debugging  
    - Outputs cleaned structured product data  
    - Potential issues: JSON parse errors, malformed AI output, unexpected data structures  

#### 1.3 Historical Data Loading and Merging

- **Overview:** Loads historical product data from Google Sheets and merges it with current scrape data for comparison.
- **Nodes Involved:**  
  - `📊 Read Historical Data` (Google Sheets read)  
  - `🔀 Merge Current with Historical` (Merge node)  
  - `🔀 Merge Current with Historical1` (Merge node for Apify data)  

- **Node Details:**

  - **📊 Read Historical Data**  
    - Type: Google Sheets node (read operation)  
    - Reads "Historical Data" sheet by document ID from environment variable  
    - Authenticated via Google Sheets OAuth2 account  
    - Provides previous competitor product data for comparison  
    - Failure cases: Invalid document ID, permission issues, empty sheets  

  - **🔀 Merge Current with Historical**  
    - Type: Merge node  
    - Combines current Firecrawl scrape data and Claude-Sonnet AI output with historical data  
    - Configured for three inputs (Firecrawl scrape, AI extraction, historical data)  
    - Outputs unified current dataset for next processing step  

  - **🔀 Merge Current with Historical1**  
    - Type: Merge node  
    - Combines Apify scrape results with historical data  
    - Matches on "rawResponse.message.content" field to align records  
    - Outputs to price and stock change detection node  

#### 1.4 Change Detection and Alert Classification

- **Overview:** Compares current and historical product data to detect price, stock, rating, and review changes; categorizes alert levels.
- **Nodes Involved:**  
  - `🔍 Detect Price & Stock Changes1` (Code node for change analysis)  

- **Node Details:**

  - **🔍 Detect Price & Stock Changes1**  
    - Type: Code node (JavaScript)  
    - Iterates over merged data items, skipping error entries  
    - Compares currentPrice, stockLevel, rating, reviewCount with historical values  
    - Calculates price changes and percentage differences  
    - Assigns alert levels: none, info, warning, critical based on thresholds (e.g., ≥20% price change triggers critical)  
    - Detects new lowest price hits and flags accordingly  
    - Detects stock status changes like Out of Stock or Back in Stock  
    - Summarizes detected changes in text for logging and alerts  
    - Outputs enriched dataset with alert metadata and change summaries  
    - Edge cases: Missing historical data, non-numeric values, unexpected stock statuses  

#### 1.5 Data Logging and Notification

- **Overview:** Logs detected alerts into Google Sheets, updates historical data with current info, and sends Telegram notifications.
- **Nodes Involved:**  
  - `💾 Update Historical Data1` (Google Sheets append)  
  - `📝 Log Alert Details` (Google Sheets append)  
  - `Send a text message` (Telegram notification)  

- **Node Details:**

  - **💾 Update Historical Data1**  
    - Type: Google Sheets node (append operation)  
    - Appends current product data with enriched fields to "Historical Data" sheet  
    - Maps all key fields: rating, stockLevel, prices, URLs, competitorName, timestamps  
    - Authenticated via Google Sheets OAuth2 account  
    - Failure points: Sheet limits, network errors, invalid data formats  

  - **📝 Log Alert Details**  
    - Type: Google Sheets node (append operation)  
    - Logs all alerts with detailed fields to a separate "Alert Log" sheet for record-keeping  
    - Includes alertLevel, priceChange, changesSummary, timestamps, and product metadata  
    - Uses same Google Sheets OAuth2 credentials  
    - Failures: Same as above  

  - **Send a text message**  
    - Type: Telegram node  
    - Sends a notification message "The competitors’ pricing has been finalized." after updates  
    - Authenticated via Telegram Bot API token and configured chat ID  
    - Failure scenarios: Invalid token, chat ID, Telegram service downtime  

#### 1.6 Workflow Control and Wait Logic

- **Overview:** Handles execution timing to ensure external scraping actors complete before fetching results.
- **Nodes Involved:**  
  - `Pause the workflow to let the Apify scraping task finish` (Wait node)  

- **Node Details:**  
  - Waits a fixed 25 seconds after starting Apify scraping before retrieving data  
  - Prevents race conditions or incomplete data fetches  
  - Risk of insufficient wait time requiring adjustment based on actor performance  

#### 1.7 Documentation and Setup Guidance

- **Overview:** Sticky notes provide comprehensive instructions, workflow overview, setup steps, and data structure references.
- **Nodes Involved:**  
  - `Sticky Note` (Large intro and instructions)  
  - `Sticky Note1` (Google Sheets column structure)  

- **Node Details:**  
  - **Sticky Note** includes workflow purpose, step-by-step logic, credential setup, prerequisites, and customization hints  
  - **Sticky Note1** details required Google Sheets columns for proper data logging and structure compliance  

---

### 3. Summary Table

| Node Name                                    | Node Type                      | Functional Role                                | Input Node(s)                            | Output Node(s)                          | Sticky Note                                                                                                     |
|----------------------------------------------|-------------------------------|------------------------------------------------|----------------------------------------|----------------------------------------|----------------------------------------------------------------------------------------------------------------|
| ⏰ Monitor Every 6 Hours                      | Schedule Trigger              | Periodic workflow trigger every 6 hours       | None                                   | 📊 Read Historical Data                 | Triggers the workflow every 6 hours to check competitor data                                                    |
| When clicking ‘Execute workflow’             | Manual Trigger               | Manual workflow start                          | None                                   | Message a model, Scrape a url and get its content, Start Actor Apify |                                                                                                                |
| Scrape a url and get its content              | Firecrawl Scraper             | Scrape competitor web page for product data   | When clicking ‘Execute workflow’       | 🔀 Merge Current with Historical        |                                                                                                                |
| Start Actor Apify                            | HTTP Request (POST)          | Starts Apify actor for competitor data scrape | When clicking ‘Execute workflow’       | Pause the workflow to let the Apify scraping task finish |                                                                                                                |
| Pause the workflow to let the Apify scraping task finish | Wait node                    | Waits for Apify scrape to complete             | Start Actor Apify                      | Get Results Apify                      |                                                                                                                |
| Get Results Apify                            | HTTP Request (GET)           | Retrieves results from Apify actor             | Pause the workflow to let the Apify scraping task finish | 🔀 Merge Current with Historical       |                                                                                                                |
| Message a model                             | Perplexity AI Node           | Legacy AI call for price info                   | When clicking ‘Execute workflow’       | Split Out                              |                                                                                                                |
| Split Out                                  | Split Out Node               | Extract citations from AI response              | Message a model                        | 🔀 Merge Current with Historical        |                                                                                                                |
| 🤖 AI Extract Product Data using Claude-Sonnet 4.5 | Anthropic AI Node            | Extract structured product data from HTML      | 🔀 Merge Current with Historical        | Converts unstructured AI text into organized, usable data fields |                                                                                                                |
| Converts unstructured AI text into organized, usable data fields | Code Node (JS)               | Parses AI JSON response and enriches data      | 🤖 AI Extract Product Data using Claude-Sonnet 4.5 | 🔀 Merge Current with Historical1       | Parses and validates the AI extracted data                                                                     |
| 🔀 Merge Current with Historical             | Merge Node                  | Combine Firecrawl scrape, AI extraction, and historical data | Split Out, Get Results Apify, 📊 Read Historical Data | 🤖 AI Extract Product Data using Claude-Sonnet 4.5 | Combines current scrape with historical data for comparison                                                    |
| 📊 Read Historical Data                      | Google Sheets Read           | Loads previous scan data for comparison         | ⏰ Monitor Every 6 Hours                | 🔀 Merge Current with Historical        | Loads previous scan data for comparison                                                                         |
| 🔀 Merge Current with Historical1            | Merge Node                  | Combines Apify results with historical data     | Converts unstructured AI text into organized, usable data fields, 📊 Read Historical Data | 🔍 Detect Price & Stock Changes1       | Combines current scrape with historical data for comparison                                                    |
| 🔍 Detect Price & Stock Changes1              | Code Node (JS)               | Detects price, stock, rating, review changes; assigns alert levels | 🔀 Merge Current with Historical1        | 💾 Update Historical Data1, 📝 Log Alert Details       | Intelligent change detection with alert level classification                                                   |
| 💾 Update Historical Data1                    | Google Sheets Append         | Saves current product data to historical sheet  | 🔍 Detect Price & Stock Changes1        | Send a text message                    | Saves current data to historical tracking sheet                                                                |
| 📝 Log Alert Details                         | Google Sheets Append         | Logs alerts to separate tracking sheet          | 🔍 Detect Price & Stock Changes1        | Send a text message                    | Logs all alerts to separate tracking sheet                                                                      |
| Send a text message                          | Telegram Node                | Sends Telegram notification after processing    | 💾 Update Historical Data1, 📝 Log Alert Details | None                                   |                                                                                                                |
| Sticky Note                                  | Sticky Note                  | Workflow introduction and setup instructions    | None                                   | None                                   | See "## Introduction" with detailed workflow overview and setup instructions                                    |
| Sticky Note1                                 | Sticky Note                  | Google Sheets required column structure         | None                                   | None                                   | ## Google Sheets Structure - Required columns defined                                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes**  
   - Add a **Schedule Trigger** node named `⏰ Monitor Every 6 Hours`  
     - Set to trigger every 6 hours interval.  
   - Add a **Manual Trigger** node named `When clicking ‘Execute workflow’` for manual runs.

2. **Configure Competitor Data Scraping**  
   - Add a **Firecrawl Scraper** node named `Scrape a url and get its content`  
     - Set URL to `https://www.on.com/en-sg/shop/mens/shoes`  
     - Operation: scrape  
     - Scrape options: extract JSON format for "price of the shoe"  
     - Attach Firecrawl API credentials.  
   - Add an **HTTP Request (POST)** node named `Start Actor Apify`  
     - URL: `https://api.apify.com/v2/acts/compass~google-maps-extractor/runs?token=YOUR_API_KEY`  
     - Method: POST  
     - JSON Body:  
       ```json
       {
         "language": "en",
         "locationQuery": "worldwide",
         "maxCrawledPlacesPerSearch": 50,
         "searchStringsArray": ["shoe stores", "shoe shops"],
         "skipClosedPlaces": false
       }
       ```  
     - Authentication: HTTP Bearer with Apify token.  
   - Add a **Wait** node named `Pause the workflow to let the Apify scraping task finish`  
     - Set to wait 25 seconds.  
   - Add an **HTTP Request (GET)** node named `Get Results Apify`  
     - URL: `https://api.apify.com/v2/acts/compass~google-maps-extractor/runs/last/dataset/items?token=YOUR_API_KEY`  
     - Authentication: HTTP Bearer with Apify token.

3. **Connect Triggers to Scrape Nodes**  
   - Connect `When clicking ‘Execute workflow’` to `Message a model`, `Scrape a url and get its content`, and `Start Actor Apify`.  
   - Connect `Start Actor Apify` to the `Pause the workflow to let the Apify scraping task finish`.  
   - Connect `Pause the workflow to let the Apify scraping task finish` to `Get Results Apify`.  
   - Connect `⏰ Monitor Every 6 Hours` to `📊 Read Historical Data`.

4. **Setup AI Extraction Nodes**  
   - Add an **Anthropic AI** node named `🤖 AI Extract Product Data using Claude-Sonnet 4.5`  
     - Model: `claude-sonnet-4-5-20250929`  
     - Input message: instruct AI to extract product fields as pure JSON from HTML.  
     - Attach Anthropic API credentials.  
   - Add a **Code** node named `Converts unstructured AI text into organized, usable data fields`  
     - Paste provided JavaScript code to parse AI response, remove markdown, parse JSON, add metadata.  
   - Add a **Perplexity AI** node named `Message a model` (optional legacy AI)  
     - Use "sonar" model with prompt "Price of the shoe".  
     - Attach Perplexity API credentials.  
   - Add a **Split Out** node to extract citations from Perplexity AI response.

5. **Setup Data Merging Nodes**  
   - Add two **Merge** nodes:  
     - `🔀 Merge Current with Historical` with 3 inputs: output of `Split Out`, `Get Results Apify`, and `📊 Read Historical Data`.  
     - `🔀 Merge Current with Historical1` with 2 inputs: output of `Converts unstructured AI text into organized, usable data fields` and `📊 Read Historical Data`.  
   - Connect nodes accordingly based on source data flows.

6. **Load Historical Data**  
   - Add a **Google Sheets** node named `📊 Read Historical Data`  
     - Operation: Read rows from sheet named "Historical Data"  
     - Document ID set via environment variable `GOOGLE_SHEET_ID`  
     - Attach Google Sheets OAuth2 credentials.

7. **Detect Changes**  
   - Add a **Code** node named `🔍 Detect Price & Stock Changes1`  
     - Paste provided JavaScript comparing current and historical data, calculating price changes, alert levels, and summarizing changes.

8. **Log and Update Data**  
   - Add two **Google Sheets** nodes:  
     - `💾 Update Historical Data1`  
       - Operation: Append rows to "Historical Data" sheet  
       - Map all product fields including calculated alert info.  
     - `📝 Log Alert Details`  
       - Operation: Append rows to "Alert Log" sheet  
       - Logs alert specifics and metadata.  
     - Both use Google Sheets OAuth2 credentials and same document ID.

9. **Send Notifications**  
   - Add a **Telegram** node named `Send a text message`  
     - Message: "The competitors’ pricing has been finalized."  
     - Attach Telegram API credentials with bot token and target chat ID.

10. **Connect Final Workflow**  
    - Connect outputs of `🔍 Detect Price & Stock Changes1` to both Google Sheets append nodes.  
    - Connect both Google Sheets append nodes to `Send a text message` node.

11. **Add Sticky Notes**  
    - Add two sticky notes with the provided content:  
      - General introduction and setup instructions.  
      - Google Sheets column structure requirements.

12. **Test and Validate**  
    - Run manual trigger and verify scraping, AI parsing, data merging, change detection, logging, and notifications work end-to-end.  
    - Adjust wait times or error handling as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                               | Context or Link                                                                                                     |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| Automate price monitoring for e-commerce competitors—ideal for retailers, analysts, and pricing teams. Scrapes competitor sites, extracts pricing/stock data via AI, detects changes, and sends instant alerts for dynamic pricing strategies. | Workflow purpose and summary (from Sticky Note)                                                                     |
| Setup instructions: Firecrawl API key, Apify API key, Claude/OpenAI API key, Google Sheets OAuth2, Telegram Bot token and chat ID, Spreadsheet setup with required columns.  | Workflow setup requirements (from Sticky Note)                                                                      |
| Google Sheets structure requires columns: Product Name, Current Price, Previous Price, Stock Status, Last Updated, URL, Change Detected.                                   | Data schema for sheets (from Sticky Note1)                                                                           |
| Alerts classified as: none, info, warning, critical based on price change thresholds (≥5%, ≥10%, ≥20%), stock status changes, rating and review count variations.            | Alert classification logic detailed in code node `🔍 Detect Price & Stock Changes1`                                  |
| For best accuracy, ensure API keys are valid and have sufficient quotas; adjust Apify wait time if scraping delays occur; monitor Telegram bot permissions and chat IDs.     | Operational considerations                                                                                           |
| Workflow supports extension with additional URLs, alternative alert channels (Slack, email), or integration with databases for enhanced business intelligence.              | Customization notes                                                                                                  |

---

**Disclaimer:** The text provided is exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.

---