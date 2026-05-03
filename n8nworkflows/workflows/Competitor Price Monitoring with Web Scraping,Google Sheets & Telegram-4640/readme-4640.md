Competitor Price Monitoring with Web Scraping,Google Sheets & Telegram

https://n8nworkflows.xyz/workflows/competitor-price-monitoring-with-web-scraping-google-sheets---telegram-4640


# Competitor Price Monitoring with Web Scraping,Google Sheets & Telegram

### 1. Workflow Overview

This workflow automates monitoring competitor product prices by scraping product pages daily, comparing current prices with previously recorded prices, logging changes, and sending alerts via Telegram. It targets e-commerce managers or analysts seeking a free, automated price tracking system that integrates web scraping, Google Sheets for data storage, and Telegram for real-time notifications.

The workflow’s logic is organized into the following blocks:

- **1.1 Scheduled Trigger and Data Input**: Initiates the workflow daily at 8:00 AM and fetches product URLs and last recorded prices from a Google Sheet.
- **1.2 Batch Processing and Rate Limiting**: Processes products one at a time to avoid overwhelming target websites, inserting pauses between requests.
- **1.3 Web Scraping and Price Extraction**: Loads each product webpage, extracts the current price using CSS selectors.
- **1.4 Data Normalization and Price Change Calculation**: Cleans extracted prices, compares with last stored prices, and calculates percentage price changes.
- **1.5 Filtering Changed Prices and Preparing Alerts**: Filters out unchanged products and formats alert messages for Telegram.
- **1.6 Logging and Data Update**: Logs price history to a Google Sheet, waits briefly, then updates the master sheet with new prices.
- **1.7 Alert Dispatch**: Sends formatted price change notifications to a configured Telegram chat.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger and Data Input

**Overview:**  
Triggers workflow execution daily at 8 AM server time and retrieves the list of products with their last recorded prices from a Google Sheet.

**Nodes Involved:**  
- Daily 8 AM Trigger  
- Fetch Product List from Sheet  
- Sticky Note1 (documentation)

**Node Details:**

- **Daily 8 AM Trigger**  
  - *Type*: Schedule Trigger  
  - *Config*: Fires every day at 8:00 AM server time.  
  - *Input/Output*: No inputs. Outputs trigger signal to fetch product list.  
  - *Potential Failures*: Server time mismatch, scheduler downtime.

- **Fetch Product List from Sheet**  
  - *Type*: Google Sheets (read/list)  
  - *Config*: Reads from the “product_data” tab (gid=0) in the Google Sheet identified by its document ID. Pulls all rows, fetching at least the `product_url` and `last_price` columns. Uses OAuth2 credentials.  
  - *Input*: Trigger from schedule node.  
  - *Output*: List of product URLs and last prices for processing.  
  - *Edge Cases*: Sheet access denied, empty or malformed rows, connection timeouts, credential expiration.

- **Sticky Note1**  
  - Describes the block’s purpose: daily trigger and reading master sheet for product URLs and last prices.

---

#### 1.2 Batch Processing and Rate Limiting

**Overview:**  
Splits product list into batches of one to process sequentially, inserting pauses between requests to avoid overloading target sites.

**Nodes Involved:**  
- Process Each Product in Batches of 1  
- Pause Between Requests  
- Sticky Note (iteration note)  
- Sticky Note3 (HTTP fetch description)

**Node Details:**

- **Process Each Product in Batches of 1**  
  - *Type*: Split In Batches  
  - *Config*: Batch size set to 1, processing products sequentially.  
  - *Input*: List of products from Google Sheets node.  
  - *Output*: Single product per output iteration.  
  - *Edge Cases*: If input list is empty, node outputs none; potential infinite loops if reset misconfigured.

- **Pause Between Requests**  
  - *Type*: Wait  
  - *Config*: Waits 20 seconds between each product request to prevent overwhelming external servers.  
  - *Input*: Triggered after each batch iteration.  
  - *Output*: Passes control to HTTP request node.  
  - *Edge Cases*: Delay may accumulate if product list is long; workflow execution time limits.

- **Sticky Note (iteration note)**  
  - Explains batch processing is set to 1 product at a time to avoid parallel requests.

- **Sticky Note3**  
  - Notes that HTTP GET requests are made for each product URL followed by CSS selector extraction.

---

#### 1.3 Web Scraping and Price Extraction

**Overview:**  
Loads product webpage HTML and extracts the current price using a CSS selector.

**Nodes Involved:**  
- Load Product Page HTML  
- Extract Current Price from HTML  
- Sticky Note3 (referenced above)

**Node Details:**

- **Load Product Page HTML**  
  - *Type*: HTTP Request  
  - *Config*: Sends HTTP GET to the product URL (`product_url` from current item). Custom headers simulate a Firefox browser for compatibility and to reduce blocking risk.  
  - *Input*: Product URL from batch node.  
  - *Output*: Raw HTML content of the product page.  
  - *Edge Cases*: HTTP errors (404, 503), IP blocking, network timeouts, CAPTCHA, malformed URLs.

- **Extract Current Price from HTML**  
  - *Type*: HTML Extract  
  - *Config*: Uses CSS selector `.price__regular > span.price-item--regular` to extract the price text from the HTML.  
  - *Input*: HTML content from HTTP Request.  
  - *Output*: Extracted `current_price` string.  
  - *Edge Cases*: Selector mismatch if webpage structure changes, empty or null extraction.

---

#### 1.4 Data Normalization and Price Change Calculation

**Overview:**  
Cleans and converts extracted price strings to numeric values, compares with last recorded prices, and calculates percentage price changes.

**Nodes Involved:**  
- Normalize Price Values  
- Compute Price Change  
- Clean Up Parsed Fields  
- Sticky Note4 (Normalization explanation)  
- Sticky Note5 (Field cleanup)

**Node Details:**

- **Normalize Price Values**  
  - *Type*: Code  
  - *Config*:  
    - Removes currency symbols and commas from `current_price`.  
    - Converts to float.  
    - Reads last price as float.  
    - Outputs normalized fields: product URL, row number, last price, current price.  
  - *Input*: Extracted price, original product data.  
  - *Output*: Normalized numeric prices.  
  - *Edge Cases*: Parsing failures if price format unexpected, NaN values.

- **Compute Price Change**  
  - *Type*: Code  
  - *Config*:  
    - Compares last and current prices.  
    - Sets `price_changed` boolean.  
    - Calculates percentage difference, rounded to 2 decimals.  
    - Adds ISO timestamp.  
  - *Input*: Normalized price data.  
  - *Output*: Price comparison results with change flags.  
  - *Edge Cases*: Division by zero if last price zero, null values.

- **Clean Up Parsed Fields**  
  - *Type*: Code  
  - *Config*:  
    - Removes stray whitespace and tab characters from field keys.  
    - Ensures consistent field naming for downstream nodes.  
  - *Input*: Computed price change data.  
  - *Output*: Cleaned JSON with consistent keys.  
  - *Edge Cases*: Missing fields or empty input array.

- **Sticky Note4**  
  - Explains stripping currency symbols, commas; calculating % change and price_changed flag.

- **Sticky Note5**  
  - Notes cleanup of malformed or whitespace-polluted keys.

---

#### 1.5 Filtering Changed Prices and Preparing Alerts

**Overview:**  
Filters products where price has changed and formats human-readable alert messages for Telegram.

**Nodes Involved:**  
- Is Price Changed? (If node)  
- Build Telegram Alert Message  
- Sticky Note6 (filter description)  
- Sticky Note7 (message formatting)

**Node Details:**

- **Is Price Changed?**  
  - *Type*: If  
  - *Config*: Checks if `price_changed` is true.  
  - *Input*: Cleaned price change data.  
  - *Output*: Routes changed prices to alert/logging; unchanged ones exit workflow.  
  - *Edge Cases*: Boolean casting errors, missing flag.

- **Build Telegram Alert Message**  
  - *Type*: Code  
  - *Config*:  
    - For each product with changed price:  
      - Determines alert type (price drop or hike).  
      - Converts timestamp to Indian Standard Time (IST).  
      - Formats message with product URL, old/new prices, % change, and timestamp.  
    - Outputs messages ready to send.  
  - *Input*: Filtered changed price items.  
  - *Output*: JSON with key `message` for Telegram.  
  - *Edge Cases*: Time zone conversion errors, missing fields.

- **Sticky Note6**  
  - Notes that only changed products proceed to alert and logging.

- **Sticky Note7**  
  - Describes message formatting and Telegram push.

---

#### 1.6 Logging and Data Update

**Overview:**  
Appends price change records to a Google Sheet log, waits 1 minute, then updates the master sheet with the latest prices.

**Nodes Involved:**  
- Log Price History to Sheet  
- Pause Before Updating Sheet  
- Update Last Price in Master Sheet  
- Sticky Note8 (logging and update description)

**Node Details:**

- **Log Price History to Sheet**  
  - *Type*: Google Sheets (append)  
  - *Config*:  
    - Appends a row to the “price_tracking” tab (gid=2110110902) in the Google Sheet.  
    - Columns include timestamp (IST formatted), product URL, last price, current price, price change %, and change flag.  
    - Uses OAuth2 credentials.  
  - *Input*: Changed price data with alert messages.  
  - *Output*: Confirmation of append operation.  
  - *Edge Cases*: Sheet write permission errors, rate limits.

- **Pause Before Updating Sheet**  
  - *Type*: Wait  
  - *Config*: Waits 1 minute before updating master sheet.  
  - *Input*: After logging price history.  
  - *Output*: Controls pacing of updates.  
  - *Edge Cases*: Workflow timeout if paused too long.

- **Update Last Price in Master Sheet**  
  - *Type*: Google Sheets (update)  
  - *Config*:  
    - Updates the master sheet (“product_data” tab, gid=0) by matching `product_url` column.  
    - Sets new `price` column to current price.  
    - Uses OAuth2 credentials.  
  - *Input*: After pause node.  
  - *Output*: Confirmation of update.  
  - *Edge Cases*: Matching failures if URLs changed, write permission issues.

- **Sticky Note8**  
  - Explains the log append followed by delayed update of master sheet prices.

---

#### 1.7 Alert Dispatch

**Overview:**  
Sends the formatted price change alert messages to a configured Telegram chat.

**Nodes Involved:**  
- Send Price Alert via Telegram  
- Sticky Note7 (message formatting and push)

**Node Details:**

- **Send Price Alert via Telegram**  
  - *Type*: Telegram  
  - *Config*:  
    - Sends text message using the `message` field from previous node.  
    - Chat ID is specified.  
    - Uses Telegram API credentials.  
  - *Input*: Output from message formatting code node.  
  - *Output*: Telegram message delivery confirmation.  
  - *Edge Cases*: Telegram API rate limits, invalid chat ID, credential expiration.

- **Sticky Note7** (also referenced here)  
  - Notes human-readable message creation and Telegram push.

---

### 3. Summary Table

| Node Name                       | Node Type             | Functional Role                             | Input Node(s)                  | Output Node(s)                        | Sticky Note                                          |
|--------------------------------|-----------------------|--------------------------------------------|-------------------------------|-------------------------------------|------------------------------------------------------|
| Daily 8 AM Trigger              | Schedule Trigger      | Initiates workflow daily at 8 AM            | None                          | Fetch Product List from Sheet       | Fires every morning at 8:00 AM server time...        |
| Fetch Product List from Sheet   | Google Sheets         | Reads product URLs and last prices from sheet | Daily 8 AM Trigger            | Process Each Product in Batches of 1 | Fires every morning at 8:00 AM server time...        |
| Process Each Product in Batches of 1 | Split In Batches       | Processes products sequentially, batch size 1 | Fetch Product List from Sheet  | Pause Between Requests / (empty batch exit) | Iterates through all products in groups (1 at a time). |
| Pause Between Requests          | Wait                  | Waits 20 seconds between requests           | Process Each Product in Batches of 1 | Load Product Page HTML             | Iterates through all products in groups (1 at a time). |
| Load Product Page HTML          | HTTP Request          | Loads product webpage HTML                   | Pause Between Requests         | Extract Current Price from HTML / Process Each Product in Batches of 1 | Fires HTTP GET for each product URL...                 |
| Extract Current Price from HTML | HTML Extract          | Extracts current price using CSS selector   | Load Product Page HTML         | Normalize Price Values              | Fires HTTP GET for each product URL...                 |
| Normalize Price Values          | Code                  | Cleans and converts price to numeric        | Extract Current Price from HTML | Compute Price Change               | Strips currency symbols/commas; calculates % change. |
| Compute Price Change            | Code                  | Compares prices and computes % difference   | Normalize Price Values         | Clean Up Parsed Fields              | Strips currency symbols/commas; calculates % change. |
| Clean Up Parsed Fields          | Code                  | Removes whitespace/tab from keys             | Compute Price Change           | Is Price Changed?                   | Removes stray whitespace or malformed keys.           |
| Is Price Changed?               | If                    | Filters items where price has changed        | Clean Up Parsed Fields         | Build Telegram Alert Message / Log Price History to Sheet | Only products with price_changed=true proceed.        |
| Build Telegram Alert Message    | Code                  | Formats alert message for Telegram           | Is Price Changed? (true branch) | Send Price Alert via Telegram       | Formats message and pushes to Telegram chat.          |
| Send Price Alert via Telegram   | Telegram               | Sends alert message to Telegram chat         | Build Telegram Alert Message   | Process Each Product in Batches of 1 | Formats message and pushes to Telegram chat.          |
| Log Price History to Sheet      | Google Sheets         | Logs price change history to Google Sheet   | Is Price Changed? (true branch) | Pause Before Updating Sheet        | Appends row to Price_History, then waits 1 minute.    |
| Pause Before Updating Sheet     | Wait                  | Waits 1 minute before updating master sheet  | Log Price History to Sheet     | Update Last Price in Master Sheet   | Appends row to Price_History, then waits 1 minute.    |
| Update Last Price in Master Sheet | Google Sheets         | Updates master sheet with new current price  | Pause Before Updating Sheet    | None                              | Appends row to Price_History, then waits 1 minute.    |
| Sticky Note1                   | Sticky Note           | Workflow start documentation                  | None                          | None                              | Fires every morning at 8:00 AM server time...          |
| Sticky Note                    | Sticky Note           | Iteration explanation                          | None                          | None                              | Iterates through all products in groups (1 at a time). |
| Sticky Note3                   | Sticky Note           | HTTP request and extraction explanation       | None                          | None                              | Fires HTTP GET for each product URL, then extracts.   |
| Sticky Note4                   | Sticky Note           | Price normalization and comparison explanation | None                          | None                              | Strips currency symbols/commas; calculates % change. |
| Sticky Note5                   | Sticky Note           | Field cleanup explanation                       | None                          | None                              | Removes stray whitespace or malformed keys.           |
| Sticky Note6                   | Sticky Note           | Filtering changed prices explanation            | None                          | None                              | Only products with price_changed=true proceed.        |
| Sticky Note7                   | Sticky Note           | Telegram message formatting explanation         | None                          | None                              | Formats message and pushes to Telegram chat.          |
| Sticky Note8                   | Sticky Note           | Logging and update explanation                   | None                          | None                              | Appends price history, then updates master sheet.     |
| Sticky Note9                   | Sticky Note           | External resources and tutorials                  | None                          | None                              | Links to blog and YouTube tutorial.                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Configure to trigger once daily at 8:00 AM server time.

2. **Add a Google Sheets node to fetch product list**  
   - Operation: Read/List Rows  
   - Document ID: Use your Google Sheet ID containing product data.  
   - Sheet Name/Tab: “product_data” (gid=0) or equivalent.  
   - Credentials: Set up OAuth2 credentials with access to your Google Sheet.  
   - Output: Expect columns `product_url` and `last_price`.

3. **Add a Split In Batches node**  
   - Batch Size: 1 to process products one at a time sequentially.  
   - Connect input from Google Sheets node.

4. **Add a Wait node for rate limiting**  
   - Duration: 20 seconds to pause between requests.  
   - Connect output from Split In Batches node (false branch or sequentially).

5. **Add an HTTP Request node to load product pages**  
   - Method: GET  
   - URL: Set expression to `{{$json["product_url"]}}`  
   - Headers: Set User-Agent to mimic Firefox browser; add Accept, Accept-Language, Accept-Encoding as in workflow.  
   - Connect output from Wait node.

6. **Add an HTML Extract node**  
   - Operation: Extract HTML Content  
   - CSS Selector: `.price__regular > span.price-item--regular` to extract price string.  
   - Connect output from HTTP Request node.

7. **Add a Code node to normalize price values**  
   - JavaScript: Remove currency symbols, commas; convert to float. Extract last price from original input for comparison.  
   - Connect output from HTML Extract node.

8. **Add a Code node to compute price changes**  
   - JavaScript: Compare last and current prices, compute `price_changed` boolean and percentage difference with 2 decimal precision. Add timestamp ISO string.  
   - Connect output from normalization code node.

9. **Add a Code node to clean up parsed fields**  
   - JavaScript: Strip whitespace and tabs from keys to ensure consistent field names.  
   - Connect output from compute price change node.

10. **Add an If node to filter changed prices**  
    - Condition: Check if `price_changed` is true.  
    - Connect output from cleanup code node.

11. **On True branch of If node, add a Code node to build Telegram alert message**  
    - JavaScript: Format a human-readable message including alert type (drop/hike), product URL, last price, current price, percentage change with sign, and timestamp converted to IST.  
    - Connect input from If node true branch.

12. **Add a Telegram node to send alerts**  
    - Text: Use the message field from the previous node.  
    - Chat ID: Your Telegram chat ID.  
    - Credentials: Set up Telegram API credentials.  
    - Connect output from message formatting node.

13. **Add a Google Sheets node to log price history**  
    - Operation: Append  
    - Document ID and Sheet Name: Same Google Sheet, tab named “price_tracking” (gid=2110110902).  
    - Columns: timestamp (IST formatted), product_url, last_price, current_price, price_diff_pct, price_changed.  
    - Credentials: Same Google Sheets OAuth2 credentials.  
    - Connect input from If node true branch.

14. **Add a Wait node to pause before updating master sheet**  
    - Duration: 1 minute.  
    - Connect output from logging node.

15. **Add a Google Sheets node to update last price in master sheet**  
    - Operation: Update row by matching `product_url`.  
    - Update `price` column with `current_price`.  
    - Document ID and Sheet Name: Same as initial product list sheet.  
    - Credentials: Same Google Sheets OAuth2 credentials.  
    - Connect output from wait node.

16. **Connect the false branch of the If node (price unchanged) back to the Split In Batches node’s next iteration or end flow.**

17. **Ensure all nodes are connected properly for the batch iteration loop:**  
    - After sending Telegram alert and updating sheets, connect back to Split In Batches node to process next product.  
    - The Split In Batches node without batches ends the loop.

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                                                                  |
|---------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Detailed step-by-step instructions and setup guide for this workflow are available at the blog post:                | https://www.blog.datahut.co/post/free-n8n-web-scraping-competitor-price-tracking                                |
| A video tutorial demonstrating the workflow is also available here:                                                | https://www.youtube.com/watch?v=a9esT732mmE                                                                     |
| The workflow respects polite scraping practices by limiting requests to one product at a time with pauses in between.| Best practices for avoiding IP blocking and server overload                                                    |
| Timezone conversions assume server uses UTC; timestamps are converted to Indian Standard Time (IST) for alerts/logs. | Important for users in Indian time zone for accurate alert timing                                               |
| Google Sheets credentials use OAuth2; ensure token refresh is handled for long-running automation.                   | Credential stability is critical for uninterrupted workflow execution                                           |
| Telegram node requires Chat ID and API key; verify chat ID corresponds to intended alert group or user.             | Prevents misdirected alerts and API error failures                                                              |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This process strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly available.