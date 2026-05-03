Template for Magento 2, Google Sheets, and Email Automation

https://n8nworkflows.xyz/workflows/template-for-magento-2--google-sheets--and-email-automation-6953


# Template for Magento 2, Google Sheets, and Email Automation

### 1. Workflow Overview

This workflow automates the process of retrieving last week's order data from a Magento 2 store, processing and summarizing it, logging the results into Google Sheets, and finally emailing a summary report. It is designed for e-commerce managers, analysts, or automated reporting systems that require weekly sales insights without manual intervention.

The workflow is logically divided into the following blocks:

- **1.1 Schedule Trigger & Initialization:** Starts the workflow on a scheduled basis and calculates the relevant week dates.
- **1.2 Data Retrieval:** Fetches last week's orders from Magento 2 via an HTTP request.
- **1.3 Data Processing:** Processes raw order data to create a weekly summary and a product breakdown.
- **1.4 Data Logging:** Logs the processed summaries into specific Google Sheets spreadsheets.
- **1.5 Reporting:** Sends an email containing the weekly summary report.

---

### 2. Block-by-Block Analysis

#### 1.1 Schedule Trigger & Initialization

- **Overview:** This block triggers the workflow on a schedule and initializes the necessary date range for the report.
- **Nodes Involved:**
  - Schedule Trigger
  - GET Week Dates
  - Create spreadsheet
- **Node Details:**

1. **Schedule Trigger**
   - **Type:** scheduleTrigger
   - **Role:** Initiates workflow execution on a defined schedule (e.g., weekly).
   - **Configuration:** Default scheduling parameters (not explicitly shown).
   - **Connections:** Outputs to GET Week Dates.
   - **Potential Failures:** Misconfigured schedule may cause missed runs.

2. **GET Week Dates**
   - **Type:** code
   - **Role:** Calculates the start and end dates of the previous week.
   - **Configuration:** Contains JavaScript code that computes week boundaries.
   - **Inputs:** Trigger from Schedule Trigger.
   - **Outputs:** Passes calculated dates to Create spreadsheet.
   - **Potential Failures:** Code errors or date miscalculations; ensure timezone consistency.

3. **Create spreadsheet**
   - **Type:** googleSheets
   - **Role:** Creates or prepares the Google Sheets document to log order data.
   - **Configuration:** Connects to Google Sheets with appropriate credentials; likely uses date info to create or select the sheet.
   - **Inputs:** Receives week dates from GET Week Dates.
   - **Outputs:** Passes control to Get Last Week Orders.
   - **Potential Failures:** Authentication errors, Google Sheets API limits, or permission issues.

---

#### 1.2 Data Retrieval

- **Overview:** Retrieves last week's orders from Magento 2 using an HTTP request.
- **Nodes Involved:**
  - Get Last Week Orders
- **Node Details:**

1. **Get Last Week Orders**
   - **Type:** httpRequest
   - **Role:** Calls Magento 2 API to fetch order data for the specified week.
   - **Configuration:** HTTP method, endpoint URL, authentication (likely API key or OAuth), and query parameters (e.g., date range).
   - **Inputs:** Receives spreadsheet creation confirmation.
   - **Outputs:** Sends raw order data to Code node.
   - **Special:** `onError` set to "continueRegularOutput" to avoid stopping workflow on errors.
   - **Potential Failures:** Network timeout, authentication failure, invalid API endpoints, or empty data sets.

---

#### 1.3 Data Processing

- **Overview:** Transforms raw order data into structured summaries for reporting.
- **Nodes Involved:**
  - Code
  - Product Breakdown
  - Weekly Summary
- **Node Details:**

1. **Code**
   - **Type:** code
   - **Role:** Parses and initially processes the raw order data.
   - **Configuration:** JavaScript code handling JSON parsing and data preparation.
   - **Inputs:** Receives raw data from Get Last Week Orders.
   - **Outputs:** Sends processed data concurrently to Product Breakdown and Weekly Summary.
   - **Potential Failures:** Parsing errors if API response format changes.

2. **Product Breakdown**
   - **Type:** code
   - **Role:** Generates detailed product-level sales data (quantities, revenue per product).
   - **Configuration:** JavaScript logic for aggregating order items.
   - **Inputs:** From Code node.
   - **Outputs:** Passes product data to Log Products Breakdown.
   - **Potential Failures:** Data inconsistencies, missing fields.

3. **Weekly Summary**
   - **Type:** code
   - **Role:** Compiles overall weekly sales summary metrics (total sales, orders, average order value).
   - **Configuration:** JavaScript aggregation logic.
   - **Inputs:** From Code node.
   - **Outputs:** Passes summary data to Log Weekly Summary.
   - **Potential Failures:** Calculation errors, missing data handling.

---

#### 1.4 Data Logging

- **Overview:** Logs processed data into Google Sheets for record-keeping and further analysis.
- **Nodes Involved:**
  - Log Weekly Summary
  - Log Products Breakdown
- **Node Details:**

1. **Log Weekly Summary**
   - **Type:** googleSheets
   - **Role:** Writes the weekly summary data into a designated Google Sheet.
   - **Configuration:** Uses Google Sheets API with configured spreadsheet ID and worksheet.
   - **Inputs:** Receives data from Weekly Summary.
   - **Outputs:** Triggers Send a message node.
   - **Potential Failures:** API quota exceeded, authentication issues, sheet access permissions.

2. **Log Products Breakdown**
   - **Type:** googleSheets
   - **Role:** Writes detailed product breakdown data into a separate Google Sheet tab.
   - **Configuration:** Similar to Log Weekly Summary but for product data.
   - **Inputs:** Receives data from Product Breakdown.
   - **Outputs:** Terminal node (no further outputs).
   - **Potential Failures:** Same as above.

---

#### 1.5 Reporting

- **Overview:** Sends an email containing the weekly sales summary.
- **Nodes Involved:**
  - Send a message
- **Node Details:**

1. **Send a message**
   - **Type:** gmail
   - **Role:** Sends an email with the weekly summary report.
   - **Configuration:** Uses Gmail OAuth2 credentials; email content likely includes summary data from Log Weekly Summary.
   - **Inputs:** Triggered by Log Weekly Summary node.
   - **Outputs:** None (terminal).
   - **Potential Failures:** Authentication errors, Gmail sending limits, malformed email.

---

### 3. Summary Table

| Node Name           | Node Type         | Functional Role               | Input Node(s)         | Output Node(s)          | Sticky Note                                        |
|---------------------|-------------------|------------------------------|-----------------------|-------------------------|--------------------------------------------------|
| Schedule Trigger    | scheduleTrigger   | Initiates workflow execution |                       | GET Week Dates          |                                                  |
| GET Week Dates       | code              | Calculates previous week dates | Schedule Trigger      | Create spreadsheet      |                                                  |
| Create spreadsheet   | googleSheets      | Prepares Google Sheet for logs | GET Week Dates        | Get Last Week Orders    |                                                  |
| Get Last Week Orders | httpRequest       | Fetches last week's orders    | Create spreadsheet    | Code                    |                                                  |
| Code                 | code              | Parses and processes raw data | Get Last Week Orders  | Product Breakdown, Weekly Summary |                                                  |
| Product Breakdown    | code              | Aggregates product-level data | Code                  | Log Products Breakdown  |                                                  |
| Weekly Summary       | code              | Aggregates weekly summary data | Code                  | Log Weekly Summary      |                                                  |
| Log Weekly Summary   | googleSheets      | Logs weekly summary to sheet  | Weekly Summary        | Send a message          |                                                  |
| Log Products Breakdown | googleSheets    | Logs product breakdown to sheet | Product Breakdown     |                         |                                                  |
| Send a message       | gmail             | Sends weekly summary email    | Log Weekly Summary    |                         |                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add `Schedule Trigger` node:**
   - Type: scheduleTrigger
   - Configure to run weekly on desired day/time.
   - No credentials needed.
   - Connect its output to next node.

3. **Add `GET Week Dates` node:**
   - Type: code
   - Paste JavaScript to calculate start and end dates of the previous week.
   - Input: none (triggered from Schedule Trigger).
   - Output: pass computed dates.
   - Connect Schedule Trigger output to this node's input.

4. **Add `Create spreadsheet` node:**
   - Type: googleSheets
   - Operation: Create Spreadsheet (or open existing).
   - Use the dates from GET Week Dates to name or select the sheet.
   - Configure Google Sheets credentials with OAuth2.
   - Connect GET Week Dates output to this node.

5. **Add `Get Last Week Orders` node:**
   - Type: httpRequest
   - Configure HTTP method (GET or POST depending on Magento API).
   - Set URL to Magento 2 orders endpoint.
   - Add query parameters for date range obtained from previous node.
   - Provide authentication (API key or OAuth as required).
   - Set `On Error` to “continueRegularOutput” to prevent workflow failure.
   - Link Create spreadsheet node output to this node.

6. **Add `Code` node:**
   - Type: code
   - Add JavaScript to parse raw JSON order data, handle errors.
   - Output two separate datasets: product-level and weekly summary.
   - Connect Get Last Week Orders output to this node.

7. **Add `Product Breakdown` node:**
   - Type: code
   - Write JavaScript to aggregate sales data by product.
   - Connect Code node output (product data) to this node.

8. **Add `Weekly Summary` node:**
   - Type: code
   - Write JavaScript to compute total sales, orders, averages.
   - Connect Code node output (summary data) to this node.

9. **Add `Log Products Breakdown` node:**
   - Type: googleSheets
   - Operation: Append or update rows in the product breakdown sheet.
   - Use Google Sheets credentials.
   - Connect Product Breakdown output to this node.

10. **Add `Log Weekly Summary` node:**
    - Type: googleSheets
    - Operation: Append or update rows in the weekly summary sheet.
    - Use Google Sheets credentials.
    - Connect Weekly Summary output to this node.

11. **Add `Send a message` node:**
    - Type: gmail
    - Configure to send email with summary report.
    - Use Gmail OAuth2 credentials.
    - Connect Log Weekly Summary output to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                 | Context or Link                          |
|----------------------------------------------------------------------------------------------|-----------------------------------------|
| This workflow requires OAuth2 credentials for both Google Sheets and Gmail nodes.           | n8n credential setup documentation      |
| Magento 2 API must be configured to allow access and provide order data for the specified week. | Magento 2 REST API documentation         |
| Scheduling frequency should be adapted to your reporting needs, typically weekly.            | n8n scheduleTrigger node documentation   |

---

**Disclaimer:** The provided text is derived solely from an n8n automated workflow. All data processed is legal and public. No illegal or protected content is included.