Track Stock Prices with ScrapeGraphAI, Yahoo Finance & Google Sheets

https://n8nworkflows.xyz/workflows/track-stock-prices-with-scrapegraphai--yahoo-finance---google-sheets-6726


# Track Stock Prices with ScrapeGraphAI, Yahoo Finance & Google Sheets

### 1. Workflow Overview

This workflow automates the tracking of stock prices by scraping data from Yahoo Finance, formatting it, and appending it to a Google Sheet for historical tracking and analysis. It is designed for users who want to monitor stock market data for one or multiple stocks on a scheduled basis without manual intervention.

The workflow is logically divided into four functional blocks:

- **1.1 Schedule Trigger:** Initiates the workflow at defined intervals.
- **1.2 Stock Data Scraping:** Uses ScrapeGraphAI to extract structured stock information from Yahoo Finance.
- **1.3 Data Formatting:** Processes and formats the scraped stock data for compatibility with Google Sheets.
- **1.4 Data Logging:** Appends the formatted data as new rows into a Google Sheets document for persistent storage and analysis.

---

### 2. Block-by-Block Analysis

#### 1.1 Schedule Trigger

- **Overview:**  
  This block triggers the workflow execution automatically on a defined schedule (default is every minute, but configurable).

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**  
  - **Node Name:** Schedule Trigger  
  - **Type:** Schedule Trigger (n8n-nodes-base.scheduleTrigger)  
  - **Role:** Initiates the workflow based on time intervals.  
  - **Configuration:**  
    - Interval set to every minute by default (empty interval object implies 1-minute interval).  
  - **Inputs:** None (trigger node).  
  - **Outputs:** Connected to "Yahoo Finance Stock Scraper".  
  - **Version:** 1.2  
  - **Potential Failures:** Misconfiguration of schedule could lead to unwanted execution frequency; no authentication needed.  
  - **Notes:** Sticky note describes the trigger role and options to customize scheduling.

#### 1.2 Stock Data Scraping

- **Overview:**  
  This block calls ScrapeGraphAI to scrape stock data from Yahoo Finance based on a user prompt and URL, extracting stock symbols, prices, changes, volume, and market cap in a structured JSON format.

- **Nodes Involved:**  
  - Yahoo Finance Stock Scraper

- **Node Details:**  
  - **Node Name:** Yahoo Finance Stock Scraper  
  - **Type:** ScrapeGraphAI Node (n8n-nodes-scrapegraphai.scrapegraphAi)  
  - **Role:** Scrapes web page content via AI-driven extraction based on natural language instructions.  
  - **Configuration:**  
    - `websiteUrl` set to the Yahoo Finance page for Apple (AAPL).  
    - `userPrompt` instructs extraction of stock data with a precise JSON schema for symbol, current price, change, change percent, volume, and market cap.  
  - **Inputs:** Receives trigger signal from Schedule Trigger.  
  - **Outputs:** Passes scraped JSON data to Stock Data Formatter node.  
  - **Credentials:** Requires ScrapeGraphAI API credentials (not provided).  
  - **Version:** 1  
  - **Potential Failures:**  
    - Authentication failure if ScrapeGraphAI credentials missing/invalid.  
    - Website structure changes could reduce accuracy.  
    - Network or timeout errors.  
    - Parsing errors if returned data does not conform to expected schema.  
  - **Notes:** Sticky note explains how to provide URL and prompt to define scraping behavior.

#### 1.3 Data Formatting

- **Overview:**  
  This block processes the scraped data to convert it into a format compatible with Google Sheets, handling either single or multiple stock entries.

- **Nodes Involved:**  
  - Stock Data Formatter (Code Node)

- **Node Details:**  
  - **Node Name:** Stock Data Formatter  
  - **Type:** Code Node (n8n-nodes-base.code)  
  - **Role:** Transforms raw scraped JSON into structured items suitable for Google Sheets appending, ensuring each stock is a separate row with expected fields.  
  - **Configuration:**  
    - JavaScript code inspects input data:  
      - If multiple stocks exist under `result.stocks`, maps each to an output item.  
      - If single stock data under `result`, returns one item.  
    - Extracts fields: symbol, current_price, change, change_percent, volume, market_cap.  
  - **Inputs:** Receives data from Yahoo Finance Stock Scraper.  
  - **Outputs:** Sends formatted data to Google Sheets Stock Logger.  
  - **Version:** 2  
  - **Potential Failures:**  
    - Code execution errors if input JSON structure differs.  
    - Missing or malformed fields in scraped data may cause undefined outputs.  
  - **Notes:** Sticky note details the purpose of formatting for sheets compatibility.

#### 1.4 Data Logging

- **Overview:**  
  This block appends the formatted stock data as new rows into a specified Google Sheets document, enabling persistent tracking.

- **Nodes Involved:**  
  - Google Sheets Stock Logger

- **Node Details:**  
  - **Node Name:** Google Sheets Stock Logger  
  - **Type:** Google Sheets (n8n-nodes-base.googleSheets)  
  - **Role:** Appends rows with stock data into "Sheet1" of a Google Sheets document.  
  - **Configuration:**  
    - Operation: Append  
    - Sheet name: "Sheet1"  
    - Document ID: URL provided but empty in this export (user must specify).  
    - Columns: Auto-mapped with keys symbol, current_price, change, change_percent, volume, market_cap.  
  - **Inputs:** Receives formatted JSON items from Stock Data Formatter.  
  - **Outputs:** None (terminal node).  
  - **Credentials:** Requires Google Sheets OAuth2 API credentials (user must configure).  
  - **Version:** 4.5  
  - **Potential Failures:**  
    - Authentication failure if credentials missing or expired.  
    - Incorrect or missing documentId or sheet name causes errors.  
    - Rate limits or API errors.  
  - **Notes:** Sticky note explains configuration and usage details for this node.

---

### 3. Summary Table

| Node Name                   | Node Type                      | Functional Role                       | Input Node(s)          | Output Node(s)         | Sticky Note                                                                                                                             |
|-----------------------------|--------------------------------|-------------------------------------|-----------------------|------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger            | Schedule Trigger               | Initiates workflow on schedule      | None                  | Yahoo Finance Stock Scraper | # Step 1: Trigger ⏱️ This trigger will invoke the workflow on the mentioned time. Configuration options include custom or alternate triggers. |
| Yahoo Finance Stock Scraper | ScrapeGraphAI Node             | Scrapes stock data from Yahoo Finance | Schedule Trigger      | Stock Data Formatter    | # Step 2: ScrapeGraphAI Node 🤖 Core node to scrape website via AI. Provide URL and extraction instructions in natural language.        |
| Stock Data Formatter        | Code Node                     | Formats scraped data for sheets     | Yahoo Finance Stock Scraper | Google Sheets Stock Logger | # Step 3: Format the response 🧱 Prepares data for Google Sheets compatibility, each stock becomes a separate row with metadata preserved. |
| Google Sheets Stock Logger  | Google Sheets Node             | Appends data rows to Google Sheets  | Stock Data Formatter   | None                   | # Step 4: Google Sheets 📊 Saves formatted data to Google Sheets with append operation; maps fields and maintains history.               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Configure interval to your desired frequency (default is every minute).  
   - No credentials required.  

2. **Create ScrapeGraphAI Node**  
   - Type: ScrapeGraphAI  
   - Set `websiteUrl` to the target Yahoo Finance stock page, e.g., https://finance.yahoo.com/quote/AAPL/  
   - Set `userPrompt` to:  
     `Extract stock information from this site. Use the following schema for response { "symbol": "AAPL", "current_price": "225.50", "change": "+2.15", "change_percent": "+0.96%", "volume": "45,234,567", "market_cap": "3.45T" }`  
   - Provide valid ScrapeGraphAI API credentials.  
   - Connect output of Schedule Trigger to input of this node.

3. **Create Code Node for Data Formatting**  
   - Type: Code  
   - Paste the following JavaScript code:  
     ```javascript
     const inputData = $input.all()[0].json;
     if (inputData.result && inputData.result.stocks) {
       const stocks = inputData.result.stocks;
       return stocks.map(stock => ({
         json: {
           symbol: stock.symbol,
           current_price: stock.current_price,
           change: stock.change,
           change_percent: stock.change_percent,
           volume: stock.volume,
           market_cap: stock.market_cap
         }
       }));
     } else if (inputData.result) {
       return [{
         json: {
           symbol: inputData.result.symbol,
           current_price: inputData.result.current_price,
           change: inputData.result.change,
           change_percent: inputData.result.change_percent,
           volume: inputData.result.volume,
           market_cap: inputData.result.market_cap
         }
       }];
     }
     ```  
   - Connect output of ScrapeGraphAI node to input of this Code node.

4. **Create Google Sheets Node for Logging**  
   - Type: Google Sheets  
   - Operation: Append  
   - Sheet Name: "Sheet1"  
   - Document ID: Enter the URL or ID of your target Google Sheets document.  
   - Columns: Use auto-map to match the fields symbol, current_price, change, change_percent, volume, market_cap.  
   - Configure Google Sheets OAuth2 credentials properly.  
   - Connect output of Code node to input of this node.

5. **Workflow Connections**  
   - Schedule Trigger → Yahoo Finance Stock Scraper → Stock Data Formatter (Code Node) → Google Sheets Stock Logger.

6. **Final Configuration**  
   - Activate the workflow to run automatically per schedule.  
   - Test with a single stock URL initially.  
   - Update URLs and prompts to track additional stocks or multiple stocks as needed.  
   - Ensure credentials for ScrapeGraphAI and Google Sheets are valid and authorized.

---

### 5. General Notes & Resources

| Note Content                                                                                                        | Context or Link                                                                                       |
|--------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| The workflow uses ScrapeGraphAI, a powerful AI tool for website scraping by natural language instructions.          | ScrapeGraphAI documentation or API reference (not included)                                         |
| Google Sheets node requires OAuth2 credentials configured with access to the target spreadsheet for appending data. | Google Sheets API documentation: https://developers.google.com/sheets/api                            |
| Schedule Trigger interval can be customized for less frequent updates to avoid API rate limits or quota exhaustion. | n8n Schedule Trigger node documentation: https://docs.n8n.io/nodes/n8n-nodes-base.scheduleTrigger/   |
| Sticky notes in the workflow provide step-by-step guidance on node usage and configuration.                        | Visible inside the workflow editor for user reference                                               |

---

**Disclaimer:**  
The text provided is exclusively from an automated workflow created using n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.