Track Amazon Product Prices with ScrapeGraphAI, Google Sheets & Telegram Alerts

https://n8nworkflows.xyz/workflows/track-amazon-product-prices-with-scrapegraphai--google-sheets---telegram-alerts-10115


# Track Amazon Product Prices with ScrapeGraphAI, Google Sheets & Telegram Alerts

### 1. Workflow Overview

This workflow automates the tracking of Amazon product prices using ScrapeGraphAI for web scraping, Google Sheets for data storage and management, and Telegram for real-time alerts. It is designed for e-commerce monitoring, price comparison, and deal hunting scenarios where users want to be notified when product prices drop below a specified threshold.

The workflow logic is divided into the following functional blocks:

- **1.1 Trigger Block**: Initiates the workflow either on a scheduled basis or manually.
- **1.2 Product Retrieval Block**: Loads the list of products and their details from Google Sheets.
- **1.3 Processing Loop Block**: Iterates over each product to scrape current prices.
- **1.4 Price Scraping Block**: Uses ScrapeGraphAI to extract the latest Amazon price.
- **1.5 Price Comparison Block**: Compares scraped prices with stored minimum prices.
- **1.6 Data Update Block**: Updates Google Sheets with new minimum prices if found.
- **1.7 Alert Notification Block**: Sends Telegram alerts if a new minimum price is detected.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger Block

**Overview:**  
Starts the workflow either on a recurring schedule or via manual execution.

**Nodes Involved:**  
- Schedule Trigger  
- When clicking ‘Execute workflow’

**Node Details:**  

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Automatically triggers the workflow every hour at 10 minutes past the hour.  
  - Configuration: Interval trigger set to trigger at minute 10 of every hour.  
  - Connections: Output connects to "Get products" node via the workflow's implicit entry point.  
  - Edge cases: Workflow may miss scheduled executions if n8n is down or paused.

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Allows manual initiation of the workflow for testing or on-demand runs.  
  - Configuration: No parameters; simply triggers workflow when clicked.  
  - Connections: Outputs to "Get products" node.  
  - Edge cases: Manual trigger depends on user action.

---

#### 1.2 Product Retrieval Block

**Overview:**  
Retrieves product data (product name, Amazon URL, minimum price, row number) from Google Sheets to process.

**Nodes Involved:**  
- Get products

**Node Details:**  

- **Get products**  
  - Type: Google Sheets (OAuth2)  
  - Role: Reads all rows from a specified Google Sheet containing product info.  
  - Configuration: Reads from sheet "gid=0" in the document with ID "1_FBegOUXt3657og_ScfuiLCERWHtxDWn14799HkVrUo".  
  - Output: JSON array with each product’s data fields (PRODUCT, URL, MIN PRICE, row_number).  
  - Connections: Outputs to "Loop" node.  
  - Edge cases: Requires valid OAuth2 credentials; fails if sheet ID or permissions are incorrect.

---

#### 1.3 Processing Loop Block

**Overview:**  
Splits the retrieved list of products into individual batches for iterative processing.

**Nodes Involved:**  
- Loop

**Node Details:**  

- **Loop**  
  - Type: SplitInBatches  
  - Role: Splits product list into batches of one (default) for sequential processing.  
  - Configuration: No special options; processes one product at a time.  
  - Input: Receives array of products from "Get products".  
  - Output: Sends each product item individually to "Scrape Amazon Product Price".  
  - Edge cases: Batch size and order affect throughput and performance.

---

#### 1.4 Price Scraping Block

**Overview:**  
Uses ScrapeGraphAI node to scrape the current product price from the Amazon URL provided.

**Nodes Involved:**  
- Scrape Amazon Product Price

**Node Details:**  

- **Scrape Amazon Product Price**  
  - Type: ScrapeGraphAI  
  - Role: Extracts the current price from the product’s Amazon page.  
  - Configuration:  
    - `userPrompt`: "You are an expert extraction algorithm. Extract the price of the Amazon product"  
    - `websiteUrl`: Dynamically set via expression `={{ $json.URL }}` for the current product.  
    - `renderHeavyJs`: Enabled to allow JavaScript rendering for dynamic content.  
    - `useOutputSchema`: Enabled to structure output.  
  - Credentials: Uses ScrapeGraphAI API key (credential required).  
  - Input: Receives product URL from "Loop".  
  - Output: Provides JSON with extracted price under `result.price`.  
  - Connections: Outputs to "Min price?" node.  
  - Edge cases:  
    - Scraping may fail if page layout changes or ScrapeGraphAI service is down.  
    - JavaScript rendering might increase latency.  
    - API quota or auth failures possible.

---

#### 1.5 Price Comparison Block

**Overview:**  
Compares the newly scraped price with the stored minimum price to determine if update and alert are required.

**Nodes Involved:**  
- Min price?  
- Merge

**Node Details:**  

- **Min price?**  
  - Type: If (conditional)  
  - Role: Checks if scraped price is lower than the saved minimum price for the product.  
  - Configuration:  
    - Condition checks if `{{$json.result.price}}` < `{{$('Loop').item.json['MIN PRICE']}}`.  
  - Input: Receives scraped price JSON.  
  - Outputs:  
    - True branch: New minimum price detected → to "Update price".  
    - False branch: No update needed → to "Merge".  
  - Edge cases:  
    - Price data type mismatches can cause comparison errors.  
    - Missing or malformed price data leads to false negatives.

- **Merge**  
  - Type: Merge  
  - Role: Combines branches to continue workflow after conditional check.  
  - Configuration: Default merge mode (union).  
  - Inputs:  
    - From "Min price?" false branch (no update), and  
    - From "Update price" node after successful update.  
  - Output: Connects to "Send alert".  
  - Edge cases: None significant.

---

#### 1.6 Data Update Block

**Overview:**  
Updates the Google Sheet with the new minimum price and current date when a new lower price is found.

**Nodes Involved:**  
- Update price

**Node Details:**  

- **Update price**  
  - Type: Google Sheets (OAuth2)  
  - Role: Updates one row in Google Sheets corresponding to the product with the new minimum price and date.  
  - Configuration:  
    - Operation: Update row by matching "row_number" column.  
    - Columns to update:  
      - DATE: Current date formatted as dd/MM/yyyy (`{{$now.format('dd/LL/yyyy')}}`)  
      - MIN PRICE: New price (`{{$('Scrape Amazon Product Price').item.json.result.price}}`)  
      - row_number: From loop item to identify the row.  
    - Sheet name: "gid=0"  
    - Document ID: Same as "Get products".  
  - Credentials: Uses Google Sheets OAuth2.  
  - Input: Product info from "Loop" and new price from "Scrape Amazon Product Price".  
  - Output: Connects to "Send alert".  
  - Edge cases:  
    - Update fails if row number is invalid or sheet permissions insufficient.  
    - Date formatting errors possible.

---

#### 1.7 Alert Notification Block

**Overview:**  
Sends a Telegram message alerting about the new lowest price detected for a product.

**Nodes Involved:**  
- Send alert

**Node Details:**  

- **Send alert**  
  - Type: Telegram  
  - Role: Sends a Telegram message to a configured chat ID with product and price information.  
  - Configuration:  
    - Text template:  
      ```
      The product {{ $('Loop').item.json['PRODUCT '] }} (sold by {{ $json['MIN PRICE'] }}) has just hit its lowest price yet on Amazon! 🎉 Check it out: {{ $('Min price?').item.json.website_url }}
      ```  
    - Chat ID: Must be set by user (placeholder "XXX" present).  
    - Disable attribution message appended by Telegram node.  
  - Credentials: Telegram API credentials required.  
  - Input: Receives data from "Update price" or "Merge" nodes.  
  - Output: None (terminal node).  
  - Edge cases:  
    - Failure to send if chat ID is incorrect or token invalid.  
    - Telegram API rate limits or downtime.

---

### 3. Summary Table

| Node Name                | Node Type                  | Functional Role                          | Input Node(s)               | Output Node(s)               | Sticky Note                                                                                                                                                          |
|--------------------------|----------------------------|----------------------------------------|-----------------------------|-----------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger          | Schedule Trigger           | Starts workflow on schedule             |                             | Get products                |                                                                                                                                                                    |
| When clicking ‘Execute workflow’ | Manual Trigger             | Starts workflow manually                |                             | Get products                |                                                                                                                                                                    |
| Get products             | Google Sheets (OAuth2)      | Retrieves product list from Google Sheets | Schedule Trigger, Manual Trigger | Loop                        | ## STEP 2 - Clone [this sheet](https://docs.google.com/spreadsheets/d/1_FBegOUXt3657og_ScfuiLCERWHtxDWn14799HkVrUo/edit?usp=sharing) - Set Product name, Product Amazon Url and Min Price (when you add a new product, please insert 9999) |
| Loop                     | SplitInBatches              | Loops over each product individually    | Get products                | Scrape Amazon Product Price |                                                                                                                                                                    |
| Scrape Amazon Product Price | ScrapeGraphAI             | Scrapes current product price from Amazon | Loop                        | Min price?                  | ## STEP 1 - Install the ScrapeGraph Community node - Register for FREE to [ScrapeGraphAI](https://dashboard.scrapegraphai.com/?via=n3witalia) and get API KEY              |
| Min price?               | If                         | Checks if scraped price is lower than stored minimum price | Scrape Amazon Product Price | Update price (true), Merge (false) |                                                                                                                                                                    |
| Update price             | Google Sheets (OAuth2)      | Updates Google Sheet with new minimum price | Min price?                  | Send alert                  |                                                                                                                                                                    |
| Merge                    | Merge                      | Merges workflow branches after price check | Min price? (false), Update price | Send alert                  |                                                                                                                                                                    |
| Send alert               | Telegram                   | Sends Telegram alert about new lowest price | Update price, Merge         |                             | ## STEP 3 - Add your Telegram ID in "Send Alert" node                                                                                                              |
| Sticky Note              | Sticky Note                | Documentation and instructions          |                             |                             | ## Track Amazon Product Prices with ScrapeGraphAI - This workflow automates the process of **monitoring Amazon product prices** and **sending alerts** when a product’s price drops below a defined threshold. - It integrates **ScrapeGraphAI**, **Google Sheets**, and **Telegram** to provide a complete end-to-end price tracking system. - This automated workflow tracks Amazon product prices and sends an alert via Telegram when a product hits a new lowest price. |
| Sticky Note1             | Sticky Note                | Documentation and instructions          |                             |                             | ## STEP 1 - Install the ScrapeGraph Community node - Register for FREE to [ScrapeGraphAI](https://dashboard.scrapegraphai.com/?via=n3witalia) and get API KEY           |
| Sticky Note2             | Sticky Note                | Documentation and instructions          |                             |                             | ## STEP 2 - Clone [this sheet](https://docs.google.com/spreadsheets/d/1_FBegOUXt3657og_ScfuiLCERWHtxDWn14799HkVrUo/edit?usp=sharing) - Set Product name, Product Amazon Url and Min Price (when you add a new product, please insert 9999) |
| Sticky Note3             | Sticky Note                | Documentation and instructions          |                             |                             | ## STEP 3 - Add your Telegram ID in "Send Alert" node                                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes**  
   - Add a **Schedule Trigger** node:  
     - Set to trigger at minute 10 every hour.  
   - Add a **Manual Trigger** node for manual execution.

2. **Create Google Sheets Node to Get Products**  
   - Add a **Google Sheets** node named "Get products".  
   - Configure OAuth2 credentials.  
   - Set Document ID to your Google Sheet ID (e.g., `1_FBegOUXt3657og_ScfuiLCERWHtxDWn14799HkVrUo`).  
   - Set Sheet Name to "gid=0".  
   - Operation: Read all rows (default).

3. **Create Loop Node**  
   - Add a **SplitInBatches** node named "Loop".  
   - Connect output of "Get products" to "Loop".  
   - Default batch size is 1 (process one product at a time).

4. **Create ScrapeGraphAI Node**  
   - Add a **ScrapeGraphAI** node named "Scrape Amazon Product Price".  
   - Configure ScrapeGraphAI API credentials.  
   - Set `userPrompt` to: "You are an expert extraction algorithm. Extract the price of the Amazon product".  
   - Set `websiteUrl` to `={{ $json.URL }}` (uses current product’s URL).  
   - Enable `renderHeavyJs` to true.  
   - Enable `useOutputSchema`.

5. **Create Conditional Node for Price Comparison**  
   - Add an **If** node named "Min price?".  
   - Condition: numeric comparison, check if scraped price `{{$json.result.price}}` is less than stored minimum price `{{$('Loop').item.json['MIN PRICE']}}`.  
   - True path goes to "Update price".  
   - False path goes to "Merge".

6. **Create Google Sheets Update Node**  
   - Add a **Google Sheets** node named "Update price".  
   - Configure OAuth2 credentials with access to the same sheet.  
   - Operation: Update row.  
   - Use `row_number` from loop item to identify row.  
   - Update columns:  
     - DATE: `{{$now.format('dd/LL/yyyy')}}`  
     - MIN PRICE: `{{$('Scrape Amazon Product Price').item.json.result.price}}`  
     - row_number: from loop item.  
   - Sheet Name: "gid=0".  
   - Document ID: same as "Get products".

7. **Create Merge Node**  
   - Add a **Merge** node named "Merge".  
   - Connect "Min price?" false branch to second input.  
   - Connect "Update price" output to first input.

8. **Create Telegram Alert Node**  
   - Add a **Telegram** node named "Send alert".  
   - Configure Telegram API credentials and set your chat ID.  
   - Set message text to:  
     ```
     The product {{ $('Loop').item.json['PRODUCT '] }} (sold by {{ $json['MIN PRICE'] }}) has just hit its lowest price yet on Amazon! 🎉 Check it out: {{ $('Min price?').item.json.website_url }}
     ```  
   - Disable appended attribution.

9. **Connect All Nodes**  
   - Connect both "Schedule Trigger" and "Manual Trigger" nodes to "Get products".  
   - Connect "Get products" to "Loop".  
   - Connect "Loop" to "Scrape Amazon Product Price".  
   - Connect "Scrape Amazon Product Price" to "Min price?".  
   - Connect "Min price?" true to "Update price".  
   - Connect "Min price?" false to "Merge" input 2.  
   - Connect "Update price" output to "Send alert".  
   - Connect "Merge" output to "Send alert".

10. **Test and Validate**  
    - Ensure Google Sheets document has columns: PRODUCT, URL, DATE, MIN PRICE, row_number.  
    - Insert products with URL and set MIN PRICE to a high number (e.g., 9999) initially.  
    - Verify ScrapeGraphAI API key is valid and Telegram credentials are set with correct chat ID.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                   | Context or Link                                                                                                             |
|------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------|
| Workflow automates **monitoring Amazon product prices** and **sending alerts** when price drops below threshold using ScrapeGraphAI, Google Sheets, and Telegram. | General workflow purpose                                                                                                    |
| Step 1: Install ScrapeGraphAI community node and register for free API key at [ScrapeGraphAI](https://dashboard.scrapegraphai.com/?via=n3witalia) | Setup instructions                                                                                                           |
| Step 2: Clone Google Sheet template at [Google Sheet](https://docs.google.com/spreadsheets/d/1_FBegOUXt3657og_ScfuiLCERWHtxDWn14799HkVrUo/edit?usp=sharing) and populate with products and URLs | Data preparation                                                                                                            |
| Step 3: Add your Telegram chat ID in the "Send alert" node to receive price drop notifications                                                 | Notification setup                                                                                                          |

---

**Disclaimer:**  
The text provided comes exclusively from an automated workflow created with n8n, an integration and automation tool. The process strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.