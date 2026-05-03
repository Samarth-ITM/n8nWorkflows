Track Amazon Product Prices with ScrapeOps API & Google Sheets Alerts

https://n8nworkflows.xyz/workflows/track-amazon-product-prices-with-scrapeops-api---google-sheets-alerts-3806


# Track Amazon Product Prices with ScrapeOps API & Google Sheets Alerts

### 1. Workflow Overview

This workflow, titled **Amazon Product Price Tracker**, automates the monitoring of Amazon product prices using the ScrapeOps structured data API. It is designed for users who want to track multiple products simultaneously, detect significant price changes, and receive timely email alerts. The workflow also maintains a historical record of prices in Google Sheets for trend analysis.

The workflow is logically divided into the following blocks:

- **1.1 Initialization & Setup**: Defines configuration parameters such as API keys, spreadsheet URLs, and email addresses.
- **1.2 Input Reception**: Reads the list of Amazon products (ASINs) and alert thresholds from a Google Sheets document.
- **1.3 Iterative Processing Loop**: Processes each product individually in batches.
- **1.4 Data Fetching & Parsing**: Calls the ScrapeOps Amazon Product API to retrieve current pricing and product details.
- **1.5 Price Validation & Calculation**: Validates fetched prices, calculates absolute and percentage price changes compared to the last known price.
- **1.6 Alert Status Determination**: Determines if a price change crosses user-defined thresholds to trigger alerts.
- **1.7 Data Persistence**: Updates the Google Sheets with the latest product data and appends new price history records.
- **1.8 Alerting**: Sends customizable email alerts if significant price changes are detected.
- **1.9 Control Flow & Error Handling**: Manages looping, conditional branching, and fallback for invalid prices.

---

### 2. Block-by-Block Analysis

#### 1.1 Initialization & Setup

- **Overview:** This block initializes workflow-wide configuration variables, including API keys, spreadsheet URLs, and email addresses for alerts.
- **Nodes Involved:**  
  - `Schedule Trigger`  
  - `Setup`  
  - `Sticky Note` (documentation)  
  - `Sticky Note2` (documentation)

- **Node Details:**

  - **Schedule Trigger**  
    - *Type:* Schedule Trigger  
    - *Role:* Starts the workflow on a recurring schedule (default: hourly).  
    - *Configuration:* Interval set to every 1 hour.  
    - *Input/Output:* No input; outputs trigger event to `Setup`.  
    - *Edge Cases:* Scheduling misconfiguration or disabled trigger will stop workflow execution.

  - **Setup**  
    - *Type:* Set  
    - *Role:* Defines key workflow variables:  
      - `spreadsheet_url`: URL of the Google Sheets document (default template URL).  
      - `scrapeops_apikey`: User’s ScrapeOps API key (empty by default).  
      - `from_email`: Email address used as sender for alerts (empty by default).  
      - `to_email`: Recipient email address for alerts (empty by default).  
    - *Input/Output:* Receives trigger from `Schedule Trigger`, outputs config data to `Products to Monitor`.  
    - *Edge Cases:* Missing or invalid API key or email addresses will cause failures downstream.

  - **Sticky Note** and **Sticky Note2**  
    - *Type:* Sticky Note  
    - *Role:* Provide detailed documentation and setup instructions embedded in the workflow canvas.  
    - *Content Highlights:*  
      - Workflow purpose and features.  
      - API registration and documentation links (https://scrapeops.io/app/register/main, https://scrapeops.io/docs/data-api/amazon-product-api/).  
      - Google Sheets setup instructions and template link.  
    - *Input/Output:* None (purely informational).

---

#### 1.2 Input Reception

- **Overview:** Reads the list of Amazon products (ASINs) and their alert thresholds from the Google Sheets document.
- **Nodes Involved:**  
  - `Products to Monitor`  
  - `Loop Over Items`

- **Node Details:**

  - **Products to Monitor**  
    - *Type:* Google Sheets (Read)  
    - *Role:* Fetches product data from the "Products to Monitor" sheet in the configured spreadsheet.  
    - *Configuration:*  
      - Document ID and sheet name dynamically set from `Setup` node’s `spreadsheet_url`.  
      - Uses OAuth2 credentials for Google Sheets access.  
    - *Input/Output:* Receives config from `Setup`, outputs list of products to `Loop Over Items`.  
    - *Edge Cases:*  
      - Incorrect spreadsheet URL or permissions cause read failures.  
      - Empty or malformed sheet data leads to empty or invalid product lists.

  - **Loop Over Items**  
    - *Type:* SplitInBatches  
    - *Role:* Processes each product individually to avoid API rate limits and manage flow control.  
    - *Configuration:* Default batch size (not explicitly set, so likely 1 item per batch).  
    - *Input/Output:* Receives product list from `Products to Monitor`, outputs single product item to `Last Price` and empty batch to loop back.  
    - *Edge Cases:* Large product lists may increase execution time; batch size tuning may be needed.

---

#### 1.3 Data Fetching & Parsing

- **Overview:** Fetches current product data from ScrapeOps API and extracts relevant fields.
- **Nodes Involved:**  
  - `Last Price`  
  - `Scrapeops - Amazon Product`  
  - `Fields`

- **Node Details:**

  - **Last Price**  
    - *Type:* Set  
    - *Role:* Prepares the last known price from the current item for comparison.  
    - *Configuration:* Copies `pricing` field from input JSON to `last_pricing` as a number.  
    - *Input/Output:* Receives product item from `Loop Over Items`, outputs to `Scrapeops - Amazon Product`.  
    - *Edge Cases:* Missing or zero last price handled gracefully by downstream logic.

  - **Scrapeops - Amazon Product**  
    - *Type:* HTTP Request  
    - *Role:* Calls ScrapeOps API to retrieve structured Amazon product data for the given ASIN.  
    - *Configuration:*  
      - URL: `https://proxy.scrapeops.io/v1/structured-data/amazon/product`  
      - Query parameters:  
        - `asin` from current product item.  
        - `api_key` from `Setup` node.  
      - Uses HTTP GET method.  
    - *Input/Output:* Receives last price info, outputs API response JSON to `Fields`.  
    - *Edge Cases:*  
      - API key invalid or missing causes auth errors.  
      - API rate limits or network issues cause timeouts or failures.  
      - Unexpected API response formats may cause expression errors.

  - **Fields**  
    - *Type:* Set  
    - *Role:* Extracts and normalizes key fields from API response:  
      - `name`: product name.  
      - `pricing`: numeric price parsed from string (removes currency symbols).  
      - `product_url`: constructed Amazon product URL using ASIN.  
    - *Input/Output:* Receives API data from `Scrapeops - Amazon Product`, outputs to `Check Valid Price`.  
    - *Edge Cases:* Missing or malformed price strings default to 0.

---

#### 1.4 Price Validation & Calculation

- **Overview:** Validates the fetched price and calculates price changes compared to the last known price.
- **Nodes Involved:**  
  - `Check Valid Price`  
  - `Price Change`

- **Node Details:**

  - **Check Valid Price**  
    - *Type:* If  
    - *Role:* Checks if the fetched price is greater than zero to proceed.  
    - *Configuration:* Condition: `pricing > 0` (strict number comparison).  
    - *Input/Output:* Receives normalized fields, outputs to `Price Change` if valid, else loops back to `Loop Over Items` to skip invalid data.  
    - *Edge Cases:* Zero or negative prices cause skipping of current item.

  - **Price Change**  
    - *Type:* Set  
    - *Role:* Calculates:  
      - `price_change`: absolute difference between current and last price (rounded to 2 decimals).  
      - `percent_change`: relative percentage change (rounded to 2 decimals), zero if last price is zero or undefined.  
    - *Input/Output:* Receives valid price data, outputs to `Alert Status`.  
    - *Edge Cases:* Division by zero avoided by conditional logic.

---

#### 1.5 Alert Status Determination

- **Overview:** Determines if the price change triggers an alert based on user-defined thresholds.
- **Nodes Involved:**  
  - `Alert Status`

- **Node Details:**

  - **Alert Status**  
    - *Type:* Set  
    - *Role:* Sets `alert_status` string to:  
      - `"High"` if `percent_change` exceeds `alert_threshold_high` (from product data).  
      - `"Low"` if `percent_change` is below `alert_threshold_low`.  
      - Empty string otherwise (no alert).  
    - *Input/Output:* Receives price change data, outputs to `Update - Products to Monitor`.  
    - *Edge Cases:* Threshold fields missing or malformed may cause incorrect alert status.

---

#### 1.6 Data Persistence

- **Overview:** Updates the product data in the main sheet and appends a new record to the price history sheet.
- **Nodes Involved:**  
  - `Update - Products to Monitor`  
  - `Insert - Price History`

- **Node Details:**

  - **Update - Products to Monitor**  
    - *Type:* Google Sheets (Update)  
    - *Role:* Updates the "Products to Monitor" sheet with latest product info:  
      - ASIN, name, pricing, product URL, alert status, last updated timestamp, price change, average rating, percent change.  
    - *Configuration:*  
      - Matches rows by `asin`.  
      - Uses OAuth2 credentials for Google Sheets.  
      - Timestamp formatted as `MM/dd/yyyy HH:mm:ss`.  
    - *Input/Output:* Receives alert status, outputs to `Insert - Price History`.  
    - *Edge Cases:* Sheet permission or connectivity issues cause update failures.

  - **Insert - Price History**  
    - *Type:* Google Sheets (Append)  
    - *Role:* Appends a new row to the "Price History" sheet recording:  
      - ASIN, pricing (numeric), timestamp.  
    - *Configuration:*  
      - Uses OAuth2 credentials.  
      - Timestamp formatted as `MM/dd/yyyy HH:mm:ss`.  
    - *Input/Output:* Receives updated product data, outputs to `Alert Decision`.  
    - *Edge Cases:* Append failures due to sheet access or API limits.

---

#### 1.7 Alerting

- **Overview:** Sends an email alert if a significant price change is detected.
- **Nodes Involved:**  
  - `Alert Decision`  
  - `Send Email`

- **Node Details:**

  - **Alert Decision**  
    - *Type:* If  
    - *Role:* Checks if `alert_status` is non-empty (either "High" or "Low") to decide whether to send an alert.  
    - *Configuration:* Condition: `alert_status` is not empty string.  
    - *Input/Output:*  
      - If true, routes to `Send Email`.  
      - If false, loops back to `Loop Over Items` to process next product.  
    - *Edge Cases:* Missing alert status or empty string prevents alerting.

  - **Send Email**  
    - *Type:* Email Send  
    - *Role:* Sends a richly formatted HTML email alert with price change details.  
    - *Configuration:*  
      - Subject includes product name and price change notification.  
      - Recipient and sender emails from `Setup` node.  
      - SMTP credentials configured externally.  
      - Email body includes: price change percentage, previous/current prices, product name, ASIN, last updated timestamp, and a link to the product page.  
      - Styled with CSS for clarity and branding.  
    - *Input/Output:* Receives alert decision, outputs to `Loop Over Items` to continue processing.  
    - *Edge Cases:* SMTP misconfiguration or network issues cause email send failures.

---

#### 1.8 Control Flow & Error Handling

- **Overview:** Manages looping over all products and handles invalid or skipped items gracefully.
- **Nodes Involved:**  
  - `Loop Over Items` (second output)  
  - `Check Valid Price` (false branch)  
  - `Alert Decision` (false branch)  
  - `Send Email` (main output loops back)

- **Node Details:**

  - The `Loop Over Items` node uses two outputs:  
    - First output processes the current item through the price check and update flow.  
    - Second output loops back to itself to continue processing remaining items or to retry after invalid price detection.  
  - The workflow ensures that invalid or zero prices do not break the flow but skip to the next product.  
  - After sending an email or deciding no alert is needed, the workflow continues processing remaining products until all are done.

---

### 3. Summary Table

| Node Name                | Node Type           | Functional Role                              | Input Node(s)            | Output Node(s)                 | Sticky Note                                                                                                                             |
|--------------------------|---------------------|----------------------------------------------|--------------------------|-------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger          | Schedule Trigger    | Starts workflow on schedule (hourly)         | None                     | Setup                         |                                                                                                                                         |
| Setup                    | Set                 | Defines API keys, spreadsheet URL, emails    | Schedule Trigger         | Products to Monitor            |                                                                                                                                         |
| Products to Monitor       | Google Sheets       | Reads product ASINs and alert thresholds     | Setup                    | Loop Over Items               |                                                                                                                                         |
| Loop Over Items           | SplitInBatches      | Iterates over each product individually       | Products to Monitor       | Last Price, Loop Over Items   |                                                                                                                                         |
| Last Price               | Set                 | Prepares last known price for comparison      | Loop Over Items           | Scrapeops - Amazon Product    |                                                                                                                                         |
| Scrapeops - Amazon Product| HTTP Request        | Fetches product data from ScrapeOps API       | Last Price                | Fields                       |                                                                                                                                         |
| Fields                   | Set                 | Extracts and normalizes product fields        | Scrapeops - Amazon Product| Check Valid Price             |                                                                                                                                         |
| Check Valid Price         | If                  | Validates price > 0                            | Fields                    | Price Change, Loop Over Items |                                                                                                                                         |
| Price Change             | Set                 | Calculates absolute and percentage price changes | Check Valid Price         | Alert Status                 |                                                                                                                                         |
| Alert Status             | Set                 | Determines alert status based on thresholds   | Price Change              | Update - Products to Monitor  |                                                                                                                                         |
| Update - Products to Monitor | Google Sheets    | Updates product info in main sheet             | Alert Status              | Insert - Price History        |                                                                                                                                         |
| Insert - Price History   | Google Sheets        | Appends new price record to history sheet     | Update - Products to Monitor | Alert Decision             |                                                                                                                                         |
| Alert Decision           | If                   | Decides whether to send alert email           | Insert - Price History    | Send Email, Loop Over Items   |                                                                                                                                         |
| Send Email               | Email Send           | Sends price alert email                         | Alert Decision            | Loop Over Items               |                                                                                                                                         |
| Sticky Note              | Sticky Note          | Workflow overview and features documentation  | None                     | None                        | # Amazon Product Price Tracker; automates price monitoring, alerts, and historical tracking.                                            |
| Sticky Note2             | Sticky Note          | API and integration setup instructions         | None                     | None                        | API key registration and documentation links; Google Sheets setup instructions and template link.                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Schedule Trigger node:**  
   - Set to trigger every 1 hour (or desired frequency).

3. **Add a Set node named "Setup":**  
   - Add variables:  
     - `spreadsheet_url` (string): default to the Google Sheets template URL.  
     - `scrapeops_apikey` (string): leave empty for user input.  
     - `from_email` (string): sender email for alerts.  
     - `to_email` (string): recipient email for alerts.  
   - Connect Schedule Trigger → Setup.

4. **Add a Google Sheets node named "Products to Monitor":**  
   - Operation: Read rows.  
   - Document ID: set to `={{ $json.spreadsheet_url }}` from Setup node.  
   - Sheet Name: "Products to Monitor" (gid=0).  
   - Credentials: configure Google Sheets OAuth2.  
   - Connect Setup → Products to Monitor.

5. **Add a SplitInBatches node named "Loop Over Items":**  
   - Default batch size (1).  
   - Connect Products to Monitor → Loop Over Items.

6. **Add a Set node named "Last Price":**  
   - Assign `last_pricing` = `={{ $json.pricing }}` (number).  
   - Connect Loop Over Items → Last Price.

7. **Add an HTTP Request node named "Scrapeops - Amazon Product":**  
   - Method: GET.  
   - URL: `https://proxy.scrapeops.io/v1/structured-data/amazon/product`.  
   - Query Parameters:  
     - `asin` = `={{ $('Loop Over Items').item.json.asin }}`  
     - `api_key` = `={{ $('Setup').item.json.scrapeops_apikey }}`  
   - Connect Last Price → Scrapeops - Amazon Product.

8. **Add a Set node named "Fields":**  
   - Assign:  
     - `name` = `={{ $json.data.name }}`  
     - `pricing` = `={{ parseFloat(($json.data.pricing || "").replace(/[^\\d.-]/g, "")) || 0 }}`  
     - `product_url` = `=https://www.amazon.com/dp/{{ $('Loop Over Items').item.json.asin }}?th=1&psc=1`  
   - Connect Scrapeops - Amazon Product → Fields.

9. **Add an If node named "Check Valid Price":**  
   - Condition: `pricing > 0` (number, strict).  
   - Connect Fields → Check Valid Price.

10. **Add a Set node named "Price Change":**  
    - Assign:  
      - `price_change` =  
        ```
        ={{ 
          $('Last Price').item.json.last_pricing !== "" && $('Last Price').item.json.last_pricing !== undefined ? 
          ($json.pricing - $('Last Price').item.json.last_pricing).toFixed(2) : 0 
        }}
        ```  
      - `percent_change` =  
        ```
        ={{ 
          $('Last Price').item.json.last_pricing !== "" && $('Last Price').item.json.last_pricing !== undefined && parseFloat($('Last Price').item.json.last_pricing) !== 0 ? 
          ((($json.pricing - $('Last Price').item.json.last_pricing) / $('Last Price').item.json.last_pricing)).toFixed(2) : 0 
        }}
        ```  
    - Connect Check Valid Price (true) → Price Change.  
    - Connect Check Valid Price (false) → Loop Over Items (to skip invalid price).

11. **Add a Set node named "Alert Status":**  
    - Assign `alert_status` =  
      ```
      ={{ 
        $json.percent_change > $('Loop Over Items').item.json.alert_threshold_high ? "High" : 
        ($json.percent_change < $('Loop Over Items').item.json.alert_threshold_low ? "Low" : "") 
      }}
      ```  
    - Connect Price Change → Alert Status.

12. **Add a Google Sheets node named "Update - Products to Monitor":**  
    - Operation: Update row by matching `asin`.  
    - Document ID: `={{ $('Setup').item.json.spreadsheet_url }}`.  
    - Sheet Name: "Products to Monitor".  
    - Columns to update: `asin`, `name`, `pricing`, `product_url`, `alert_status`, `last_updated` (current timestamp), `price_change`, `average_rating`, `percent_change`.  
    - Credentials: Google Sheets OAuth2.  
    - Connect Alert Status → Update - Products to Monitor.

13. **Add a Google Sheets node named "Insert - Price History":**  
    - Operation: Append row.  
    - Document ID: `={{ $('Setup').item.json.spreadsheet_url }}`.  
    - Sheet Name: "Price History".  
    - Columns: `asin`, `pricing`, `timestamp` (current timestamp).  
    - Credentials: Google Sheets OAuth2.  
    - Connect Update - Products to Monitor → Insert - Price History.

14. **Add an If node named "Alert Decision":**  
    - Condition: `alert_status` is not empty string.  
    - Connect Insert - Price History → Alert Decision.

15. **Add an Email Send node named "Send Email":**  
    - SMTP credentials configured externally.  
    - From: `={{ $('Setup').item.json.from_email }}`  
    - To: `={{ $('Setup').item.json.to_email }}`  
    - Subject: `=Amazon Price Tracker Alert: {{ $('Update - Products to Monitor').item.json.name }} Price Change Detected`  
    - HTML Body: Use the provided styled HTML template with dynamic fields for price change, product name, ASIN, previous/current prices, timestamp, and product URL.  
    - Connect Alert Decision (true) → Send Email.  
    - Connect Alert Decision (false) → Loop Over Items (to continue).  
    - Connect Send Email → Loop Over Items (to continue).

16. **Connect Loop Over Items (second output) → Last Price** to continue processing next batch.

17. **Add Sticky Notes** for documentation and setup instructions as per original workflow.

18. **Configure all credentials:**  
    - Google Sheets OAuth2 with access to the spreadsheet.  
    - SMTP account for sending emails.  
    - ScrapeOps API key entered in the Setup node.

19. **Test the workflow:**  
    - Ensure the spreadsheet is properly set up with ASINs and alert thresholds.  
    - Run the workflow manually or wait for scheduled trigger.  
    - Verify price data updates, history appends, and email alerts.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                   | Context or Link                                                                                                  |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| ScrapeOps API handles anti-bot measures, proxy rotation, and parsing complexities, providing a maintenance-free Amazon product data extraction solution.                                                                       | https://scrapeops.io                                                                                             |
| Google Sheets template for product monitoring is shared read-only; users must make a personal copy to avoid data conflicts.                                                                                                    | https://docs.google.com/spreadsheets/d/1hRv-TBXrpN6rkIU65WorttNHt-IPWas_An0sF4Of39U                              |
| API documentation for the Amazon Product API endpoint used in this workflow is available for detailed parameter and response format reference.                                                                                | https://scrapeops.io/docs/data-api/amazon-product-api/                                                           |
| Email alert HTML template is styled for clarity and branding, including dynamic color coding for price increases (red) and decreases (green).                                                                                  | Embedded in the "Send Email" node                                                                                |
| This workflow is ideal for deal hunters, price comparison services, and e-commerce analytics professionals. Alerting can be extended to other channels like Slack or Telegram by adding respective nodes and integrations.      | Workflow description and design notes                                                                            |

---

This document provides a complete, structured reference for understanding, reproducing, and maintaining the Amazon Product Price Tracker workflow in n8n.