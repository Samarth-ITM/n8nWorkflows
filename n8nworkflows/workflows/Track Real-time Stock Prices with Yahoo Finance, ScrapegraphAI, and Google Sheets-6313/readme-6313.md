Track Real-time Stock Prices with Yahoo Finance, ScrapegraphAI, and Google Sheets

https://n8nworkflows.xyz/workflows/track-real-time-stock-prices-with-yahoo-finance--scrapegraphai--and-google-sheets-6313


# Track Real-time Stock Prices with Yahoo Finance, ScrapegraphAI, and Google Sheets

### 1. Workflow Overview

This workflow automates the process of tracking real-time stock prices by scraping data from Yahoo Finance, formatting the extracted information, and storing it in Google Sheets for further analysis or record-keeping. It is designed for users or systems needing automated updates of stock market data at scheduled intervals without manual intervention.

The workflow consists of four logical blocks:

- **1.1 Scheduled Trigger:** Initiates the workflow periodically based on a defined schedule.
- **1.2 Web Scraping with ScrapegraphAI:** Extracts structured stock data from the Yahoo Finance webpage using an AI-driven scraping node.
- **1.3 Data Formatting:** Processes and shapes the scraped raw data into a structured format suitable for storage.
- **1.4 Google Sheets Integration:** Appends the formatted stock data into a specified Google Sheets document for persistent storage and easy access.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
  This block triggers the entire workflow automatically at scheduled intervals, enabling real-time or periodic stock price updates without manual initiation.

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**

  - **Schedule Trigger**  
    - *Type & Role:* `scheduleTrigger` node; acts as the entry point to initiate the workflow based on time intervals.  
    - *Configuration:* Set to trigger at a recurring interval (default is every minute as per the empty interval array).  
    - *Expressions/Variables:* None.  
    - *Connections:* Outputs to ScrapegraphAI node.  
    - *Version Requirements:* Compatible with n8n version 1.2 or higher.  
    - *Potential Failures:* Incorrect scheduling parameters, system downtime, or misconfigured timezone settings could cause missed triggers.

#### 1.2 Web Scraping with ScrapegraphAI

- **Overview:**  
  This block uses ScrapegraphAI to scrape live stock information from Yahoo Finance. It leverages AI to extract data points based on a user-defined schema and natural language instructions.

- **Nodes Involved:**  
  - ScrapegraphAI

- **Node Details:**

  - **ScrapegraphAI**  
    - *Type & Role:* Custom AI scraping node (`scrapegraphAi`) designed to extract structured data from websites using natural language prompts.  
    - *Configuration:*  
      - `Website URL:` "https://finance.yahoo.com/quote/AAPL/..." (Yahoo Finance page for Apple stock).  
      - `User Prompt:` Instructs the node to extract stock information using a JSON schema with keys like symbol, current_price, change, change_percent, volume, and market_cap.  
    - *Expressions/Variables:* None dynamic; fixed URL and prompt.  
    - *Connections:* Output feeds into the Code node for formatting.  
    - *Credentials:* Requires valid ScrapegraphAI API credentials configured externally.  
    - *Version Requirements:* Version 1.  
    - *Potential Failures:*  
      - API authentication errors due to invalid or missing credentials.  
      - Website structure changes breaking extraction.  
      - Network timeouts or scraping rate limits.  
      - Unexpected response format causing parsing issues.

#### 1.3 Data Formatting

- **Overview:**  
  This block formats the raw JSON data returned by ScrapegraphAI into a clean structure compatible with Google Sheets. It handles both single and multiple stock data formats, ensuring consistent output.

- **Nodes Involved:**  
  - Code

- **Node Details:**

  - **Code**  
    - *Type & Role:* JavaScript Code node used for data transformation and formatting.  
    - *Configuration:*  
      - Script extracts `result` from incoming JSON, checks if it contains multiple stocks or a single stock, and maps the data into a flat array of objects with standard fields: symbol, current_price, change, change_percent, volume, market_cap.  
    - *Expressions/Variables:*  
      - Uses `$input.all()[0].json` to read input data.  
      - Conditional logic on presence of `result.stocks` or `result`.  
    - *Connections:* Outputs formatted data to Google Sheets node.  
    - *Version Requirements:* Version 2.  
    - *Potential Failures:*  
      - Input data missing expected keys leading to runtime errors.  
      - Syntax errors in code.  
      - Unexpected input structure from ScrapegraphAI node.

#### 1.4 Google Sheets Integration

- **Overview:**  
  This block appends the formatted stock data as new rows into a Google Sheets spreadsheet, preserving historical data and enabling easy access for reporting or analysis.

- **Nodes Involved:**  
  - Google Sheets

- **Node Details:**

  - **Google Sheets**  
    - *Type & Role:* Built-in Google Sheets node used to append rows to a spreadsheet.  
    - *Configuration:*  
      - Operation: Append rows.  
      - Target Sheet: "Sheet1" (by name).  
      - Document ID: Provided as a URL (needs to be configured).  
      - Columns mapped automatically based on keys: symbol, current_price, change, change_percent, volume, market_cap.  
    - *Expressions/Variables:* None dynamic, except document ID must be set.  
    - *Connections:* Receives data from Code node.  
    - *Credentials:* Requires Google Sheets OAuth2 credentials properly set up.  
    - *Version Requirements:* Version 4.5.  
    - *Potential Failures:*  
      - Authentication or permission errors with Google API.  
      - Incorrect or missing document ID or sheet name.  
      - API rate limits or quota exceeded.  
      - Data format mismatch causing append failures.

---

### 3. Summary Table

| Node Name        | Node Type                 | Functional Role                    | Input Node(s)        | Output Node(s)  | Sticky Note                                                                                                                  |
|------------------|---------------------------|----------------------------------|----------------------|-----------------|------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger | scheduleTrigger           | Starts workflow on schedule       | —                    | ScrapegraphAI   | # Step 1: Trigger ⏱️ This trigger will invoke the workflow on the mentioned time. You can customize the trigger interval.    |
| ScrapegraphAI    | scrapegraphAi             | Scrapes stock data from website   | Schedule Trigger     | Code            | # Step 2: ScrapeGraphAI Node 🤖 Core node scraping website based on URL and natural language instructions.                    |
| Code             | code                      | Formats scraped data for Sheets   | ScrapegraphAI        | Google Sheets   | # Step 3: Format the response 🧱 Prepares data for Google Sheets compatibility, each stock becomes a separate row.            |
| Google Sheets    | googleSheets              | Appends formatted data to Sheets  | Code                 | —               | # Step 4: Google Sheets 📊 Saves data to your Google Sheets, appending new rows with scraped data.                            |
| Sticky Note1     | stickyNote                | Documentation                    | —                    | —               | # Step 2: ScrapeGraphAI Node 🤖 This is the core node which will scrape the website that you want. How to use instructions...  |
| Sticky Note2     | stickyNote                | Documentation                    | —                    | —               | # Step 3: Format the response 🧱 This node will format the results for sheets. What it does and why it is important.           |
| Sticky Note3     | stickyNote                | Documentation                    | —                    | —               | # Step 1: Trigger ⏱️ This trigger will invoke the workflow on the mentioned time. Configuration options explained.            |
| Sticky Note      | stickyNote                | Documentation                    | —                    | —               | # Step 4: Google Sheets 📊 This node will save the formatted data to your Google Sheets. Configuration and capabilities.       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it appropriately (e.g., "Stock").

2. **Add a Schedule Trigger node:**
   - Set the trigger interval as per your requirement (default is every minute).
   - This node will start the workflow automatically at scheduled times.

3. **Add a ScrapegraphAI node:**
   - Set the `websiteUrl` parameter to `"https://finance.yahoo.com/quote/AAPL/?guccounter=1&guce_referrer=..."`
   - In `userPrompt`, enter:  
     `Extract stock information from this site. Use the following schema for response { "symbol": "AAPL", "current_price": "225.50", "change": "+2.15", "change_percent": "+0.96%", "volume": "45,234,567", "market_cap": "3.45T" }`
   - Configure and link valid ScrapegraphAI API credentials.
   - Connect the output of Schedule Trigger to this node.

4. **Add a Code node:**
   - Paste the following JavaScript code to format results:

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
         market_cap: stock.market_cap,
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
         market_cap: inputData.result.market_cap,
       }
     }];
   }
   ```
   - Connect the output of ScrapegraphAI to this Code node.

5. **Add a Google Sheets node:**
   - Set operation to `Append`.
   - Specify the target spreadsheet by URL in `documentId`.
   - Specify the target sheet name as `"Sheet1"`.
   - Map fields automatically to columns: symbol, current_price, change, change_percent, volume, market_cap.
   - Configure and link Google Sheets OAuth2 credentials.
   - Connect the output of the Code node to this Google Sheets node.

6. **Test the workflow manually or wait for the scheduled trigger to run.**

7. **Optional:** Add sticky notes as documentation within the n8n editor to describe each step for clarity and maintenance.

---

### 5. General Notes & Resources

| Note Content                                                                                                              | Context or Link                                                                                   |
|---------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| ScrapegraphAI node enables natural language based scraping, greatly simplifying data extraction from complex websites.    | Node documentation and API details at ScrapegraphAI official resources.                          |
| Google Sheets node requires OAuth2 credentials with appropriate permissions to append rows to the target spreadsheet.     | Google OAuth2 setup instructions: https://developers.google.com/identity/protocols/oauth2       |
| The workflow assumes stability of Yahoo Finance page structure; changes may require prompt or schema updates.             | Monitor Yahoo Finance page for layout changes affecting scraping accuracy.                       |
| The workflow is designed to be extensible for multiple stocks by adjusting the URL or prompt in the ScrapegraphAI node.  | Can be modified to loop over multiple stock symbols for batch processing.                        |
| Sticky notes in the workflow provide helpful inline guidance for each step, improving maintainability and collaboration. | Use within n8n editor for documentation purposes.                                               |

---

Disclaimer: The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and publicly accessible.