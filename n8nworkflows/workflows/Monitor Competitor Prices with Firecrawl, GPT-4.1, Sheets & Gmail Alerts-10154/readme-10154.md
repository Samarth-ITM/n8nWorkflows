Monitor Competitor Prices with Firecrawl, GPT-4.1, Sheets & Gmail Alerts

https://n8nworkflows.xyz/workflows/monitor-competitor-prices-with-firecrawl--gpt-4-1--sheets---gmail-alerts-10154


# Monitor Competitor Prices with Firecrawl, GPT-4.1, Sheets & Gmail Alerts

### 1. Workflow Overview

This workflow automates the monitoring of competitor pricing and stock status for e-commerce products, specifically targeting shoe retailers and pricing analysts. It scrapes product pages from multiple competitor websites, uses AI to extract structured product data, compares this data against historical records, detects significant price or stock changes, logs these changes, and sends alert emails summarizing critical updates.

The workflow is logically divided into the following blocks:

- **1.1 Input Trigger and URL Scraping:** Initiates scraping of competitor URLs using Firecrawl nodes.
- **1.2 AI Data Extraction:** Processes scraped HTML through GPT-4.1-mini to extract structured product information.
- **1.3 Data Parsing and Validation:** Cleans and parses AI output into usable JSON format.
- **1.4 Historical Data Retrieval:** Loads historical competitor data from Google Sheets for comparison.
- **1.5 Data Merging and Change Detection:** Merges current and historical data, detects price/stock/rating changes, and classifies alerts.
- **1.6 Data Logging and Alerting:** Updates Google Sheets with current and alert data and aggregates alerts into a daily digest.
- **1.7 Notification Delivery:** Sends summarized alert emails via Gmail.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Trigger and URL Scraping

**Overview:**  
This block triggers the workflow manually and scrapes competitor product pages using Firecrawl nodes for three major shoe retailers.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- Scrape URL: nike.com (Firecrawl)  
- Scrape URL: adidas.com (Firecrawl)  
- Scrape URL: sneakerpricer.com (Firecrawl)

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Workflow initiation node requiring manual user start.  
  - Configuration: Default manual trigger, no parameters.  
  - Input: None  
  - Output: Triggers all three scraper nodes in parallel.  
  - Edge Cases: None, but workflow only runs on manual trigger.

- **Scrape URL: nike.com / adidas.com / sneakerpricer.com**  
  - Type: Firecrawl Scraper Node  
  - Role: Scrape competitor URLs for raw HTML content of product listings.  
  - Configuration:  
    - URL set to competitor website product pages (Nike, Adidas, SneakerPricer).  
    - Scrape operation with JSON format prompt specifying "price of the shoe" extraction.  
  - Credentials: Firecrawl API key required.  
  - Input: Trigger from manual node.  
  - Output: Raw HTML responses passed downstream for AI extraction.  
  - Edge Cases:  
    - Possible HTTP errors, timeouts, or blocking by competitor sites.  
    - Firecrawl API quota or auth errors.

---

#### 2.2 AI Data Extraction

**Overview:**  
This block sends the raw HTML scraped data to OpenAI GPT-4.1-mini to extract structured e-commerce product data as JSON.

**Nodes Involved:**  
- 🤖 AI Extract Product Data using GPT-4.1-mini

**Node Details:**

- **🤖 AI Extract Product Data using GPT-4.1-mini**  
  - Type: OpenAI Chat Node  
  - Role: Transform unstructured HTML into structured product data fields using AI.  
  - Configuration:  
    - Prompt instructs the model to extract specific fields (productName, currentPrice, originalPrice, currency, inStock, stockLevel, rating, reviewCount, lastUpdated, productUrl, competitorName).  
    - Output expected is pure JSON without markdown formatting.  
    - Model: GPT-4.1-mini, temperature 0.1 for deterministic output.  
  - Credentials: OpenAI API key required.  
  - Input: Raw HTML from Firecrawl scraping nodes.  
  - Output: AI-generated JSON response with product data.  
  - Edge Cases:  
    - Parsing errors if AI returns malformed JSON.  
    - API rate limits or auth failures.

---

#### 2.3 Data Parsing and Validation

**Overview:**  
Parses the AI's JSON text output, cleans markdown artifacts, and enriches the data with metadata while handling errors gracefully.

**Nodes Involved:**  
- Converts unstructured AI text into organized, usable data fields (Code node)

**Node Details:**

- **Converts unstructured AI text into organized, usable data fields**  
  - Type: JavaScript Code Node  
  - Role: Parse AI response JSON, handle errors, add metadata fields.  
  - Configuration:  
    - Removes markdown code blocks from AI output.  
    - Attempts JSON.parse with fallback to regex extraction.  
    - Adds fields: scrapedAt (ISO timestamp), initial priceChange = 0, priceChangePercent = 0, isNewLow = false, alertLevel = 'none'.  
    - Catches parse errors and outputs error objects for diagnostics.  
  - Input: AI JSON string responses.  
  - Output: Parsed JSON objects ready for comparison.  
  - Edge Cases:  
    - Malformed AI output resulting in parse failure.  
    - Unexpected data structures.

---

#### 2.4 Historical Data Retrieval

**Overview:**  
Reads previously stored competitor product data from a Google Sheets document to enable change detection.

**Nodes Involved:**  
- 📊 Read Historical Data (Google Sheets node)

**Node Details:**

- **📊 Read Historical Data**  
  - Type: Google Sheets Read Node  
  - Role: Load historical product data from Google Sheets "Historical Data" tab.  
  - Configuration:  
    - Reads by sheet name "Historical Data".  
    - Document ID taken from environment variable GOOGLE_SHEET_ID.  
  - Credentials: Google Sheets OAuth2 with appropriate access scopes.  
  - Input: None (triggered downstream after parsing).  
  - Output: Historical competitor data records.  
  - Edge Cases:  
    - OAuth token expiration or invalid scopes.  
    - Sheet missing or renamed.  
    - API rate limits.

---

#### 2.5 Data Merging and Change Detection

**Overview:**  
Merges current scraped data with historical records to detect price, stock, rating, and review changes, then classifies alert levels accordingly.

**Nodes Involved:**  
- 🔀 Merge Current with Historical (Merge node)  
- 🔍 Detect Price & Stock Changes (Code node)

**Node Details:**

- **🔀 Merge Current with Historical**  
  - Type: Merge Node  
  - Role: Combine current product data with historical data based on matching rawResponse.message.content fields.  
  - Configuration: Mode "combine".  
  - Input: Current parsed data & historical data.  
  - Output: Merged data items for comparison.  
  - Edge Cases: Matching failures if keys differ or missing.

- **🔍 Detect Price & Stock Changes**  
  - Type: JavaScript Code Node  
  - Role: Detects significant changes and assigns alert levels: critical, warning, info, or none.  
  - Logic highlights:  
    - Compares old and new prices, calculates absolute and percentage changes.  
    - Flags critical if price changes ≥ 20%, warning if ≥ 10%, info if ≥ 5%.  
    - Checks for new lowest price and flags info alerts.  
    - Detects stock changes including out of stock (critical), low stock (warning), back in stock (info).  
    - Detects rating changes ≥ 0.5 stars (info).  
    - Identifies new reviews count increments.  
  - Adds fields: alertLevel, changesSummary, hasChanges, lowestPrice, previousPrice.  
  - Input: Merged data from previous node.  
  - Output: Annotated data with change detection results.  
  - Edge Cases:  
    - Missing or malformed numeric fields.  
    - Historical data not found (first-time tracking).  
    - Code runtime exceptions.

---

#### 2.6 Data Logging and Alerting

**Overview:**  
Logs all detected alerts to a separate Google Sheets tab and compiles a daily digest summarizing critical, warning, and informational alerts.

**Nodes Involved:**  
- 💾 Update Historical Data (Google Sheets Append)  
- 📝 Log Alert Details (Google Sheets Append)  
- 📊 Aggregate Daily Digest (Code node)

**Node Details:**

- **💾 Update Historical Data**  
  - Type: Google Sheets Append Node  
  - Role: Append current scraped data (with changes) to "Historical Data" sheet for ongoing tracking.  
  - Configuration: Maps all relevant product fields including pricing, stock, ratings, timestamps.  
  - Input: Change detection output.  
  - Credentials: Google Sheets OAuth2.  
  - Output: Confirmation of append operation.  
  - Edge Cases: API limits, sheet write failures.

- **📝 Log Alert Details**  
  - Type: Google Sheets Append Node  
  - Role: Logs alerts with detailed info (alertLevel, changesSummary, price changes etc.) into "Alert Log" sheet.  
  - Configuration: Maps alert-related fields for audit and review.  
  - Input: Change detection output.  
  - Credentials: Google Sheets OAuth2.  
  - Output: Confirmation of log entry.  
  - Edge Cases: Same as above.

- **📊 Aggregate Daily Digest**  
  - Type: JavaScript Code Node  
  - Role: Aggregates all alerts into categorized lists (critical, warning, info), counts totals, and builds a summary object for email.  
  - Configuration:  
    - Filters alerts by alertLevel.  
    - Constructs arrays of alert details with competitor, product name, price, stock, and URL.  
    - Includes counts and ISO timestamp.  
  - Input: Output from change detection node (all alerts).  
  - Output: Single JSON object summarizing the day’s alerts.  
  - Edge Cases: No alerts or empty input.

---

#### 2.7 Notification Delivery

**Overview:**  
Sends the compiled daily digest email to a specified Gmail recipient with alert details.

**Nodes Involved:**  
- Send a message (Gmail node)

**Node Details:**

- **Send a message**  
  - Type: Gmail Send Node  
  - Role: Email delivery of the daily alert summary.  
  - Configuration:  
    - Recipient email set to "info@example.com" (placeholder).  
    - Subject line "Shoes pricing".  
    - Message body fixed text "The pricing of the competitors is attached" (likely expects attachment or could be customized).  
  - Credentials: Gmail OAuth2 with send email scope.  
  - Input: Aggregated daily digest output.  
  - Output: Email sent confirmation.  
  - Edge Cases: OAuth token expiry, Gmail API quota, invalid email format.

---

### 3. Summary Table

| Node Name                             | Node Type                   | Functional Role                              | Input Node(s)                      | Output Node(s)                           | Sticky Note                                                  |
|-------------------------------------|-----------------------------|----------------------------------------------|----------------------------------|----------------------------------------|--------------------------------------------------------------|
| When clicking ‘Execute workflow’    | Manual Trigger              | Workflow start trigger                        | None                             | Scrape URL: nike.com, adidas.com, sneakerpricer.com |                                                              |
| Scrape URL: nike.com                 | Firecrawl Scraper           | Scrape Nike product page HTML                 | When clicking ‘Execute workflow’ | 🤖 AI Extract Product Data using GPT-4.1-mini           |                                                              |
| Scrape URL: adidas.com               | Firecrawl Scraper           | Scrape Adidas product page HTML               | When clicking ‘Execute workflow’ | 🤖 AI Extract Product Data using GPT-4.1-mini           |                                                              |
| Scrape URL: sneakerpricer.com       | Firecrawl Scraper           | Scrape SneakerPricer product page HTML       | When clicking ‘Execute workflow’ | 🤖 AI Extract Product Data using GPT-4.1-mini           |                                                              |
| 🤖 AI Extract Product Data using GPT-4.1-mini | OpenAI Chat Node           | Extract structured product data from HTML    | Scrape URL nodes                 | Converts unstructured AI text into organized, usable data fields |                                                              |
| Converts unstructured AI text into organized, usable data fields | Code Node                   | Parse and validate AI JSON output             | 🤖 AI Extract Product Data using GPT-4.1-mini | 🔀 Merge Current with Historical                         |                                                              |
| 📊 Read Historical Data             | Google Sheets Read          | Load historical competitor data               | None                             | 🔀 Merge Current with Historical             | Loads previous scan data for comparison                      |
| 🔀 Merge Current with Historical    | Merge Node                 | Combine current and historical data           | Converts unstructured AI text..., 📊 Read Historical Data | 🔍 Detect Price & Stock Changes                | Combines current scrape with historical data for comparison |
| 🔍 Detect Price & Stock Changes     | Code Node                  | Detect significant changes and assign alerts | 🔀 Merge Current with Historical  | 📊 Aggregate Daily Digest, 💾 Update Historical Data, 📝 Log Alert Details | Intelligent change detection with alert level classification |
| 💾 Update Historical Data           | Google Sheets Append        | Append current data to historical sheet       | 🔍 Detect Price & Stock Changes   | None                                   | Saves current data to historical tracking sheet             |
| 📝 Log Alert Details                | Google Sheets Append        | Log alert details into alert log sheet         | 🔍 Detect Price & Stock Changes   | None                                   | Logs all alerts to separate tracking sheet                  |
| 📊 Aggregate Daily Digest           | Code Node                  | Aggregate alerts into daily summary            | 🔍 Detect Price & Stock Changes   | Send a message                            | Combines all alerts into a comprehensive summary            |
| Send a message                     | Gmail Send                  | Send alert summary email                        | 📊 Aggregate Daily Digest         | None                                   |                                                              |
| Sticky Note                        | Sticky Note                 | Introduction and setup instructions            | None                             | None                                   | See detailed content in the workflow overview               |
| Sticky Note1                       | Sticky Note                 | Google Sheets structure guidance                | None                             | None                                   | See detailed content in the workflow overview               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Add a Manual Trigger node named "When clicking ‘Execute workflow’".

2. **Add Firecrawl Scraper Nodes for URLs:**  
   - Create three Firecrawl nodes named:  
     - "Scrape URL: nike.com" with URL: `https://www.nike.com/sg/w/mens-shoes-nik1zy7ok`  
     - "Scrape URL: adidas.com" with URL: `https://www.adidas.com/us/men-shoes`  
     - "Scrape URL: sneakerpricer.com" with URL: `https://www.sneakerpricer.com/us-EN`  
   - Configure operation as "scrape" with JSON format prompt requesting "price of the shoe".  
   - Authenticate using Firecrawl API credentials.

3. **Connect Manual Trigger Output to Each Firecrawl Scraper Node Input.**

4. **Add OpenAI Node for AI Extraction:**  
   - Create an OpenAI Chat node named "🤖 AI Extract Product Data using GPT-4.1-mini".  
   - Set model to GPT-4.1-mini, temperature 0.1.  
   - Use prompt instructing extraction of productName, currentPrice, originalPrice, currency, inStock, stockLevel, rating, reviewCount, lastUpdated, productUrl, competitorName as JSON only (no markdown).  
   - Pass scraped HTML and metadata (URL, competitor) as variables in the prompt.  
   - Authenticate with OpenAI API credentials.

5. **Connect Each Firecrawl Scraper node output to this OpenAI node input.**

6. **Add Code Node to Parse AI Output:**  
   - Create a Code node named "Converts unstructured AI text into organized, usable data fields".  
   - Use JavaScript code that:  
     - Removes markdown code blocks.  
     - Parses JSON safely with error handling.  
     - Enriches data with scrapedAt timestamp and initializes alert fields.  
   - Connect OpenAI node output to this Code node.

7. **Add Google Sheets Read Node for Historical Data:**  
   - Create Google Sheets node "📊 Read Historical Data".  
   - Set operation to read sheet named "Historical Data".  
   - Use environment variable or explicit Google Sheet ID.  
   - Authenticate with Google Sheets OAuth2 credentials.

8. **Add Merge Node:**  
   - Create Merge node "🔀 Merge Current with Historical".  
   - Set mode to "combine".  
   - Connect outputs of Code node (current data) and Google Sheets node (historical data) as inputs to Merge node.

9. **Add Code Node for Change Detection:**  
   - Create Code node "🔍 Detect Price & Stock Changes" with JavaScript logic to:  
     - Compare current vs historical prices, stock levels, ratings, reviews.  
     - Assign alert levels (critical, warning, info, none).  
     - Summarize changes in text.  
   - Connect Merge node output to this node.

10. **Add Google Sheets Append Nodes:**  
    - "💾 Update Historical Data" to append all current data to "Historical Data" sheet.  
    - "📝 Log Alert Details" to append alert info to "Alert Log" sheet.  
    - Configure both with proper column mappings matching data fields.  
    - Authenticate with Google Sheets OAuth2.  
    - Connect output of change detection node to both append nodes.

11. **Add Code Node to Aggregate Alerts:**  
    - Create "📊 Aggregate Daily Digest" Code node.  
    - Aggregate alerts by alertLevel, count totals, and build JSON summary object.  
    - Connect change detection node output to this node.

12. **Add Gmail Node to Send Alerts:**  
    - Create Gmail Send node "Send a message".  
    - Set recipient email to intended alert recipient (e.g., info@example.com).  
    - Set subject and message body (customize as needed).  
    - Authenticate with Gmail OAuth2 credentials.  
    - Connect aggregate digest node output to Gmail node.

13. **Add Sticky Notes:**  
    - Add informational Sticky Note nodes for introduction and Google Sheets column structure as per workflow.

14. **Verify all connections and credentials.**  
    - Firecrawl API key  
    - OpenAI API key  
    - Google Sheets OAuth2 with read/write scopes  
    - Gmail OAuth2 with send email scope

15. **Test workflow end-to-end by manual trigger.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                  | Context or Link                                           |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------|
| Automate price monitoring for e-commerce competitors—ideal for retailers, analysts, and pricing teams. Requires self-hosted n8n instance. Scrapes competitor URLs, extracts data via AI, detects price/stock changes, logs to Google Sheets, sends email alerts. Customize URLs, thresholds, or integrate Slack alerts. Saves 2+ hours daily. | Workflow introduction (Sticky Note)                       |
| Google Sheets structure requires columns: Product Name, Current Price, Previous Price, Stock Status, Last Updated, URL, Change Detected.                                                                                                                                                                                                                     | Google Sheets Setup (Sticky Note1)                         |
| Firecrawl API key required: get from Firecrawl dashboard and add to n8n credentials.                                                                                                                                                                                                                                                                           | Setup instructions in workflow notes                       |
| OpenAI API key required: get from OpenAI platform and add to n8n credentials.                                                                                                                                                                                                                                                                                   | Setup instructions in workflow notes                       |
| Google Sheets OAuth2: Create OAuth2 client in Google Cloud Console, authorize in n8n, enable Sheets API.                                                                                                                                                                                                                                                       | Setup instructions in workflow notes                       |
| Gmail OAuth2: Use same Google Cloud project, authorize in n8n, enable Gmail API for sending emails.                                                                                                                                                                                                                                                           | Setup instructions in workflow notes                       |
| Spreadsheet Setup: Create Google Sheet with required columns, copy Sheet ID, paste in n8n environment variable GOOGLE_SHEET_ID.                                                                                                                                                                                                                                | Setup instructions in workflow notes                       |
| Workflow designed for self-hosted n8n instances due to API key secrecy and control.                                                                                                                                                                                                                                                                             | Workflow intro note                                        |
| Consider adding retry logic or error handling for Firecrawl and OpenAI API rate limits or timeouts.                                                                                                                                                                                                                                                           | Recommended best practice                                  |
| To extend, add more competitor URLs by duplicating Firecrawl and AI nodes with new URLs and competitor names.                                                                                                                                                                                                                                                 | Customization advice                                       |

---

**Disclaimer:** The provided text derives exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly respects applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly available.