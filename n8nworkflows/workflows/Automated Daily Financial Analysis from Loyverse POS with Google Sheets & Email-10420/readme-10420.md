Automated Daily Financial Analysis from Loyverse POS with Google Sheets & Email

https://n8nworkflows.xyz/workflows/automated-daily-financial-analysis-from-loyverse-pos-with-google-sheets---email-10420


# Automated Daily Financial Analysis from Loyverse POS with Google Sheets & Email

---

### 1. Workflow Overview

This workflow automates the daily financial analysis of sales data from the Loyverse Point of Sale (POS) system, integrating data retrieval, processing, historical comparison, and reporting via Google Sheets and email. It targets small to medium retail businesses that use Loyverse POS and want automated daily insights delivered by email and stored in spreadsheets for ongoing analysis.

The workflow is logically divided into these blocks:

- **1.1 Trigger & Configuration Setup:** Manual or scheduled workflow start; loading user configuration parameters.
- **1.2 Product Data Retrieval & Formatting:** Fetch all product variants from Loyverse and prepare detailed item data.
- **1.3 Shift Timing Calculation:** Determine the exact business day window for sales data based on configurable shift times and timezone.
- **1.4 Sales & Shift Data Retrieval:** Retrieve yesterday’s shifts and receipts from Loyverse API within the calculated time window.
- **1.5 Historical Data Reading:** Load past sales data from Google Sheets for comparison and trend analysis.
- **1.6 Metrics Calculation:** Compute key financial metrics, best sellers, category breakdowns, and performance trends using current and historical data.
- **1.7 Data Saving & Reporting:** Append the latest product and sales data to Google Sheets and send a summarized report email.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Trigger & Configuration Setup

- **Overview:** Initiates the workflow manually or on a scheduled basis and loads all user-configurable settings (Google Sheets IDs, business hours, timezone, Loyverse IDs).
- **Nodes Involved:**  
  - `When clicking ‘Execute workflow’`  
  - `Run Daily at 8:15AM (open to change)`  
  - `MASTER CONFIG`

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Allows manual execution for testing or ad hoc runs  
    - Connections: Outputs to `MASTER CONFIG`  
    - Failure Modes: None typical; manual start only

  - **Run Daily at 8:15AM (open to change)**  
    - Type: Schedule Trigger  
    - Role: Starts workflow automatically every day at 8:15 AM  
    - Configuration: Interval trigger with hour=8, minute=15  
    - Failure Modes: Timezone mismatches can cause wrong trigger time

  - **MASTER CONFIG**  
    - Type: Code Node  
    - Role: Central configuration holder, defining spreadsheet IDs, shift start/end times, timezone, email recipients, Loyverse POS device and payment type IDs, and product categories  
    - Key Expressions: All config returned as a JSON object for downstream use  
    - Failure Modes: Missing or incorrect config values can break API queries or sheet interactions  
    - Notes: This is the only node that requires user edits before running the workflow

---

#### 2.2 Product Data Retrieval & Formatting

- **Overview:** Fetches all product variants from Loyverse API and formats them into a flattened list with relevant properties for profit calculation and mapping.
- **Nodes Involved:**  
  - `Get all products from Loyverse`  
  - `Format Product Data`  
  - `Save Product List`

- **Node Details:**

  - **Get all products from Loyverse**  
    - Type: HTTP Request  
    - Role: Calls Loyverse API endpoint `/v1.0/items` with a limit parameter to fetch product data  
    - Credentials: Uses HTTP Bearer token for authentication (Loyverse API token)  
    - Failure Modes: Authentication failure, API rate limits, network errors

  - **Format Product Data**  
    - Type: Code Node  
    - Role: Processes raw API response to flatten product variants into individual entries with detailed fields (variant ID, barcode, cost, price, category, etc.)  
    - Logic: Iterates items and variants; skips items without variants logging a warning  
    - Outputs: Array of product variant objects for downstream use  
    - Failure Modes: Missing variant arrays cause warnings; malformed API response may cause runtime errors

  - **Save Product List**  
    - Type: Google Sheets  
    - Role: Appends or updates the formatted product data into a configured Google Sheet tab  
    - Configuration: Uses spreadsheet ID and sheet name from `MASTER CONFIG`  
    - Credentials: OAuth2 for Google Sheets  
    - Failure Modes: Credential expiration, permissions, sheet not shared or missing

---

#### 2.3 Shift Timing Calculation

- **Overview:** Calculates the exact 24-hour window representing "yesterday's" business day based on configured shift end time and timezone, accounting for shifts that cross midnight.
- **Nodes Involved:**  
  - `Calculate Shift Time`

- **Node Details:**

  - **Calculate Shift Time**  
    - Type: Code Node  
    - Role: Computes start and end timestamps for the sales data query window, adjusting for shifts ending early morning next day  
    - Logic: Parses shift end time, adds 2 hours for "changeover", compares current time with changeover to decide reporting day, returns ISO strings for API calls and formatted business date  
    - Key Expressions: Uses timezone-aware JavaScript Date conversions via `toLocaleString` with timezone argument  
    - Failure Modes: Incorrect timezone or shift times lead to wrong data window; daylight savings time changes may cause subtle errors

---

#### 2.4 Sales & Shift Data Retrieval

- **Overview:** Retrieves yesterday’s shift and receipt data from the Loyverse API using the calculated time window.
- **Nodes Involved:**  
  - `Get Yesterday's Shifts From Loyverse`  
  - `Get Yesterday's Receipts From Loyverse`

- **Node Details:**

  - **Get Yesterday's Shifts From Loyverse**  
    - Type: HTTP Request  
    - Role: Fetches shift records created within the calculated business day window  
    - Query Parameters: `created_at_min` and `created_at_max` from `Calculate Shift Time` output  
    - Credentials: Same Bearer token as product fetch  
    - Failure Modes: API errors, token expiry, empty results

  - **Get Yesterday's Receipts From Loyverse**  
    - Type: HTTP Request  
    - Role: Fetches receipt records created within the same time window, limited to 250 records per query  
    - Credentials: Same as shifts  
    - Failure Modes: API limits, token issues, network errors

---

#### 2.5 Historical Data Reading

- **Overview:** Reads previously saved historical sales data from a configured Google Sheet to enable performance comparisons and trends.
- **Nodes Involved:**  
  - `Read Historical Data`

- **Node Details:**

  - **Read Historical Data**  
    - Type: Google Sheets  
    - Role: Reads entire sheet defined in config for sales data history  
    - Configuration: Uses spreadsheet ID and sales data sheet name from `MASTER CONFIG`  
    - Credentials: Google OAuth2  
    - Failure Modes: Access denied, sheet missing, empty data

---

#### 2.6 Metrics Calculation

- **Overview:** Core logic node computing a comprehensive set of financial and sales metrics by combining current shifts, receipts, products, and historical data.
- **Nodes Involved:**  
  - `Calculate All Metrics`

- **Node Details:**

  - **Calculate All Metrics**  
    - Type: Code Node  
    - Role: Aggregates data and computes metrics including gross/net profit, best sellers by quantity and profit, discount totals, cash and QR payments, average transaction values, category-level revenue/profit, weekday/month performance vs average, rolling 30-day and week-to-date net profits, and profit trend indicators  
    - Inputs: Outputs from shift and receipt queries, formatted product data, historical sales data, and calculated business date  
    - Logic Highlights:  
      - Builds lookup maps for products by variant ID  
      - Processes shifts for cash and payment reporting  
      - Calculates profit per line item adjusting for VAT and cost  
      - Computes ranking and percentage changes vs historical averages  
      - Uses date filtering to calculate rolling and WTD aggregates  
      - Generates emoji indicators for profit trends  
    - Failure Modes: Missing or malformed input data, division by zero if no receipts, date parsing errors, config mismatches  

---

#### 2.7 Data Saving & Reporting

- **Overview:** Saves the latest calculated sales data to Google Sheets and sends a formatted summary report via email.
- **Nodes Involved:**  
  - `Save Latest Sales Data`  
  - `Send email`

- **Node Details:**

  - **Save Latest Sales Data**  
    - Type: Google Sheets  
    - Role: Appends or updates the sales metrics data into the configured historical sales data sheet  
    - Configuration: Uses spreadsheet ID and sales data sheet name from `MASTER CONFIG`  
    - Credentials: Google OAuth2  
    - Failure Modes: Credential or permission issues, sheet access errors

  - **Send email**  
    - Type: Email Send  
    - Role: Sends a plain text email summarizing key metrics from the latest calculation  
    - Subject: "Daily Report for [date]" dynamically set from metrics  
    - To: Email address configured in `MASTER CONFIG`  
    - From: Fixed sender email "report@yourbusiness.com"  
    - Content: Includes total income, net profit, cash difference, performance vs average, cumulative profits, and profit trend emoji  
    - Credentials: SMTP account configured in n8n  
    - Failure Modes: SMTP authentication errors, invalid recipient email, network errors

---

### 3. Summary Table

| Node Name                          | Node Type           | Functional Role                      | Input Node(s)                          | Output Node(s)                                   | Sticky Note                                                                                      |
|-----------------------------------|---------------------|------------------------------------|--------------------------------------|-------------------------------------------------|------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’  | Manual Trigger      | Manual workflow execution start     | None                                 | MASTER CONFIG                                    |                                                                                                |
| Run Daily at 8:15AM (open to change) | Schedule Trigger    | Scheduled daily workflow start      | None                                 | Get all products from Loyverse                   |                                                                                                |
| MASTER CONFIG                     | Code                | Central user configuration          | When clicking ‘Execute workflow’     | Get all products from Loyverse                    | # SETUP STEP 2/4: Edit config variables; see note for spreadsheet copy link                    |
| Get all products from Loyverse    | HTTP Request        | Fetch product catalog from Loyverse | MASTER CONFIG                        | Format Product Data                              | ## Get Product Data from Loyverse; No changes required here!                                   |
| Format Product Data               | Code                | Flatten and format product variants | Get all products from Loyverse       | Save Product List                                | ## Get Product Data from Loyverse; No changes required here!                                   |
| Save Product List                | Google Sheets       | Save formatted product data         | Format Product Data                  | Calculate Shift Time                             | # Setup Step 3/4: Setup Google Sheets document and sheet mapping as per instructions           |
| Calculate Shift Time             | Code                | Calculate shift based business day window | Save Product List                   | Get Yesterday's Shifts From Loyverse               | ## Get Sales Data From Last Shift; No changes required here! (Change shift times in config)    |
| Get Yesterday's Shifts From Loyverse | HTTP Request        | Fetch shifts data for reporting day | Calculate Shift Time                | Get Yesterday's Receipts From Loyverse           | ## Get Sales Data From Last Shift; No changes required here! (Change shift times in config)    |
| Get Yesterday's Receipts From Loyverse | HTTP Request       | Fetch receipts data for reporting day | Get Yesterday's Shifts From Loyverse | Read Historical Data                           | ## Get Sales Data From Last Shift; No changes required here! (Change shift times in config)    |
| Read Historical Data             | Google Sheets       | Load historical sales data from sheet | Get Yesterday's Receipts From Loyverse | Calculate All Metrics                          | # Setup Step 3/4: Setup Spreadsheet and sheet for historical data as per instructions          |
| Calculate All Metrics            | Code                | Compute detailed financial and sales metrics | Read Historical Data                | Send email, Save Latest Sales Data               | ## Calculate Metrics                                                                          |
| Send email                      | Email Send          | Send daily report email             | Calculate All Metrics                | None                                            | ## Send Report                                                                               |
| Save Latest Sales Data          | Google Sheets       | Save computed sales metrics         | Calculate All Metrics                | None                                            | # SETUP STEP 4/4: Setup spreadsheet and sheet for saving latest sales data with mapping       |
| Sticky Note                    (multiple)         | Sticky Note          | Setup instructions and guidance      | N/A                                 | N/A                                             | Various notes providing setup instructions, spreadsheet links, and configuration tips         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**  
   - Add a **Manual Trigger** node named `When clicking ‘Execute workflow’`.  
   - Add a **Schedule Trigger** node named `Run Daily at 8:15AM (open to change)` configured to trigger daily at 08:15.

2. **Create Configuration Node:**  
   - Add a **Code** node named `MASTER CONFIG`.  
   - Paste user configuration JSON including:  
     - Google Sheets Spreadsheet ID and sheet names for product list and sales data.  
     - Business shift start/end times (e.g., "08:00" and "02:00").  
     - Timezone string (e.g., "Asia/Bangkok").  
     - Email address to receive the report.  
     - Loyverse POS device IDs array, QR payment type IDs array, and product category mappings.  
   - Connect `When clicking ‘Execute workflow’` output to `MASTER CONFIG` input.  
   - Connect `Run Daily at 8:15AM (open to change)` output to next block (below).

3. **Fetch and Format Product Data:**  
   - Add an **HTTP Request** node `Get all products from Loyverse`.  
     - URL: `https://api.loyverse.com/v1.0/items`  
     - Authentication: HTTP Bearer with Loyverse API token credential.  
     - Query parameter `limit=250`.  
   - Connect `MASTER CONFIG` output to this node’s input (for scheduling runs, connect schedule trigger as well).  
   - Add a **Code** node `Format Product Data`.  
     - JavaScript code to flatten product variants and extract fields (variant_id, cost, category_id, etc.).  
   - Connect `Get all products from Loyverse` to `Format Product Data`.

4. **Save Product List to Google Sheets:**  
   - Add a **Google Sheets** node `Save Product List`.  
     - Operation: Append or Update  
     - Document ID and Sheet Name: Use expressions from `MASTER CONFIG` config object.  
     - Credentials: Google Sheets OAuth2.  
   - Connect `Format Product Data` to `Save Product List`.

5. **Calculate Shift Time Window:**  
   - Add a **Code** node `Calculate Shift Time`.  
   - JavaScript code to:  
     - Parse shift end time and timezone from config.  
     - Calculate changeover time 2 hours after shift end.  
     - Determine if current time is before or after changeover to decide reporting day.  
     - Return ISO strings for start and end of the business day window and formatted date string.  
   - Connect `Save Product List` to `Calculate Shift Time`.

6. **Retrieve Yesterday’s Shifts and Receipts:**  
   - Add **HTTP Request** node `Get Yesterday's Shifts From Loyverse`.  
     - URL: `https://api.loyverse.com/v1.0/shifts`  
     - Query parameters: `created_at_min` and `created_at_max` from `Calculate Shift Time` outputs.  
     - Authentication: Bearer token credential.  
   - Connect `Calculate Shift Time` to this node.  
   - Add **HTTP Request** node `Get Yesterday's Receipts From Loyverse`.  
     - URL: `https://api.loyverse.com/v1.0/receipts`  
     - Query parameters: `created_at_min`, `created_at_max` from `Calculate Shift Time`; `limit=250`.  
     - Authentication: Bearer token credential.  
   - Connect `Get Yesterday's Shifts From Loyverse` to `Get Yesterday's Receipts From Loyverse`.

7. **Read Historical Sales Data from Google Sheets:**  
   - Add a **Google Sheets** node `Read Historical Data`.  
     - Operation: Read (Get Rows) from sales data sheet.  
     - Document ID and Sheet Name: Expressions from `MASTER CONFIG`.  
     - Credentials: Google Sheets OAuth2.  
   - Connect `Get Yesterday's Receipts From Loyverse` to `Read Historical Data`.

8. **Calculate Metrics:**  
   - Add a **Code** node `Calculate All Metrics`.  
   - Use provided JavaScript logic to:  
     - Read inputs from previous nodes (shifts, receipts, product data, historical data).  
     - Compute all defined metrics including profit, best sellers, category revenue, performance vs average, rolling sums, and trend indicators.  
   - Connect `Read Historical Data` to `Calculate All Metrics`.

9. **Save Latest Sales Data:**  
   - Add a **Google Sheets** node `Save Latest Sales Data`.  
     - Operation: Append or Update  
     - Document ID and Sheet Name: Expressions from `MASTER CONFIG`.  
     - Credentials: Google Sheets OAuth2.  
   - Connect `Calculate All Metrics` to `Save Latest Sales Data`.

10. **Send Daily Report Email:**  
    - Add an **Email Send** node `Send email`.  
    - Configure SMTP credentials for your mail server.  
    - From: `report@yourbusiness.com`  
    - To: Expression referencing email from `MASTER CONFIG`.  
    - Subject: `"Daily Report for {{date}}"` using metric date.  
    - Body: Plain text containing key metrics and performance data with templated expressions from `Calculate All Metrics`.  
    - Connect `Calculate All Metrics` to `Send email`.

---

### 5. General Notes & Resources

| Note Content                                                                                       | Context or Link                                                                                                   |
|--------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| Setup Step 1/4: Create credentials for Loyverse API token, Google Sheets API, and SMTP/email sender. | Sticky Note at workflow start; Loyverse token must be created in Loyverse Integrations > Access Tokens.           |
| Setup Step 2/4: Copy the example Google Sheets template: https://docs.google.com/spreadsheets/d/1DlEUo3mQUaxn2HEp34m7VAath8L3RDuPy5zFCljSZHE/edit?usp=sharing | Sticky Note near MASTER CONFIG node.                                                                             |
| Setup Step 3/4: Configure Google Sheets nodes with spreadsheet and sheet names from MASTER CONFIG; map columns manually for product and sales data sheets. | Sticky Notes near Save Product List and Read Historical Data nodes.                                              |
| Setup Step 4/4: Configure "Save Latest Sales Data" Google Sheets node with correct spreadsheet and sheet; map columns to metrics output. | Sticky Note near Save Latest Sales Data node.                                                                     |
| The workflow uses timezone-aware date calculations and assumes consistent timezone configuration in MASTER CONFIG and n8n workflow settings. | Important to avoid date mismatches and incorrect data windows.                                                  |
| Limit of 250 records in Loyverse API requests; consider pagination if more data expected.          | May require workflow extension for larger data sets.                                                             |

---

**Disclaimer:** The provided content is exclusively derived from an automated workflow created with n8n, respecting all applicable content policies. No illegal or protected data is processed.

---