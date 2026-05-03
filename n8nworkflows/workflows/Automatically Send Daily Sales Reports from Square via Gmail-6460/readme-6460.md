Automatically Send Daily Sales Reports from Square via Gmail

https://n8nworkflows.xyz/workflows/automatically-send-daily-sales-reports-from-square-via-gmail-6460


# Automatically Send Daily Sales Reports from Square via Gmail

### 1. Workflow Overview

This workflow automates the daily collection and email delivery of sales summary reports from Square to a designated recipient via Gmail. It is designed for businesses using Square POS who want to receive daily sales summaries for all their locations without manual intervention.

The workflow logically separates into these blocks:

- **1.1 Schedule Trigger:** Initiates the workflow daily to process the previous day’s sales.
- **1.2 Fetch Locations:** Retrieves all business locations from Square.
- **1.3 Process Each Location’s Sales:** For each location, fetch completed orders for the previous day.
- **1.4 Filter Locations Without Sales:** Skip locations that had no sales on the prior day.
- **1.5 Compile Sales Report:** Aggregate sales, taxes, discounts, tips, tender types, and fees into a summary matching Square’s Sales Summary report.
- **1.6 Convert to CSV:** Format the aggregated data into a CSV file.
- **1.7 Send Email Report:** Email the CSV file as an attachment to the specified recipient via Gmail.

---

### 2. Block-by-Block Analysis

#### 1.1 Schedule Trigger

- **Overview:** Triggers the workflow once daily to process sales from the previous day.
- **Nodes Involved:**  
  - `Schedule Trigger`
- **Node Details:**  
  - **Type:** Schedule Trigger  
  - **Configuration:** Runs daily (default interval). The node’s timestamp is used for date calculations.  
  - **Key Expressions:** Uses the current timestamp to calculate the previous day’s date for querying sales.  
  - **Input:** None (trigger node)  
  - **Output:** Initiates the rest of the workflow  
  - **Edge Cases:** Timezone differences could affect the exact date range for sales data; ensure the timezone offset (-05:00) matches the business location.  
  - **Version:** v1.2  

#### 1.2 Fetch Locations

- **Overview:** Retrieves all active Square locations connected to the account to prepare for per-location sales queries.
- **Nodes Involved:**  
  - `Get Square Locations`  
  - `Turn Locations Into List`
- **Node Details:**

  - **Get Square Locations:**  
    - Type: HTTP Request  
    - Role: Calls Square API endpoint `/v2/locations` with authentication to get location data.  
    - Configuration: Uses Header Auth credential named "Square Header Auth" with `Authorization: Bearer <token>`. Adds `Content-Type: application/json` header.  
    - Input: Output from Schedule Trigger  
    - Output: JSON object containing locations array  
    - Edge Cases:  
      - Auth errors if token is invalid or expired  
      - API rate limiting errors  
      - Network timeouts  

  - **Turn Locations Into List:**  
    - Type: Split Out  
    - Role: Converts the array of locations into individual items to process each location separately downstream.  
    - Configuration: Splits on `locations` field, includes only `id` field for output.  
    - Input: `Get Square Locations` output  
    - Output: One item per location containing location id  
    - Edge Cases: If no locations are returned, downstream nodes receive no data  

#### 1.3 Process Each Location’s Sales

- **Overview:** For each location, fetch completed orders from Square for the previous day.
- **Nodes Involved:**  
  - `Get Sales from Square`
- **Node Details:**  
  - Type: HTTP Request  
  - Role: POST request to Square’s `/v2/orders/search` endpoint to retrieve orders filtered by location and date.  
  - Configuration:  
    - Method: POST  
    - Body: JSON with `location_ids` array containing current location id, filters for orders with state `COMPLETED`, and a date range for the previous day based on the Schedule Trigger timestamp.  
    - Limit: 1000 orders max per request (pagination not implemented).  
    - Headers: `Content-Type: application/json`, authenticated via Header Auth credential.  
  - Key Expressions:  
    - Uses `{{ $json.locations.id }}` from split location items.  
    - Date filter uses `{{$('Schedule Trigger').item.json.timestamp.toDateTime().minus(1, 'days').format('yyyy-MM-dd')}}` to get the previous day.  
  - Input: Output from `Turn Locations Into List` (each location)  
  - Output: JSON with orders array for that location  
  - Edge Cases:  
    - More than 1000 orders per location would require pagination (not currently handled).  
    - Auth errors, API rate limits, network issues.  
    - No orders returned (empty sales).  

#### 1.4 Filter Locations Without Sales

- **Overview:** Skips further processing for locations that have no sales/orders for the previous day.
- **Nodes Involved:**  
  - `Ignore Locations w/o Sales` (If node)
- **Node Details:**  
  - Type: If  
  - Role: Checks if the orders array from the previous node is non-empty.  
  - Configuration: Condition checks that `$json.orders` array is not empty.  
  - Input: Output from `Get Sales from Square`  
  - Output: Only passes locations with orders to next node; others are discarded.  
  - Edge Cases: If orders field missing or null, node may not behave as expected; strict type validation is enabled.  

#### 1.5 Compile Sales Report

- **Overview:** Aggregates sales data for each location into summary metrics matching Square’s Sales Summary report.
- **Nodes Involved:**  
  - `Compile Sales Reports` (Code node)
- **Node Details:**  
  - Type: Code (JavaScript)  
  - Role: Processes all orders for a location and calculates totals for gross sales, net sales, discounts, taxes, tips, tender types (cash, card, gift card, other), refunds, fees, and rounding adjustments.  
  - Configuration: Runs once per item (each location’s sales data).  
  - Key Expressions:  
    - Uses order data and location metadata from previous nodes.  
    - Performs complex calculations to handle returns and refunds by referencing original sales transactions.  
    - Converts amounts from cents to dollars by dividing by 100.  
  - Input: Output from `Ignore Locations w/o Sales` (locations with sales)  
  - Output: JSON with aggregated report fields for each location and date  
  - Edge Cases:  
    - Assumes order and refund objects have consistent structure.  
    - Potential for errors if any expected fields are missing.  
    - Large data sets could impact performance.  

#### 1.6 Convert to CSV

- **Overview:** Converts the aggregated sales summary JSON into a CSV file for easy sharing.
- **Nodes Involved:**  
  - `Convert Sales Summary to CSV File`
- **Node Details:**  
  - Type: Convert To File  
  - Role: Converts JSON data into CSV format and stores it in binary property `sales_report`.  
  - Configuration: Filename set dynamically using timestamp from Schedule Trigger, e.g., `sales_report_2023-05-01T08:00:00.csv`.  
  - Input: Output from `Compile Sales Reports`  
  - Output: Binary CSV file attachment  
  - Edge Cases:  
    - If no data is provided, CSV file will be empty.  
    - Filename collisions unlikely due to timestamp but possible if multiple executions run simultaneously.  

#### 1.7 Send Email Report

- **Overview:** Email the CSV sales report to the designated recipient.
- **Nodes Involved:**  
  - `Send Report`
- **Node Details:**  
  - Type: Gmail (OAuth2) node  
  - Role: Sends an email with the generated CSV file attached.  
  - Configuration:  
    - Recipient email: configurable (`user@example.com` placeholder).  
    - Subject includes date extracted from Schedule Trigger readable date.  
    - Message body is HTML formatted with a polite note.  
    - Attachment uses binary property `sales_report` from previous node.  
    - Append Attribution enabled (includes n8n branding in signature).  
    - Uses Gmail OAuth2 credential configured with user's Google account.  
  - Input: Output from `Convert Sales Summary to CSV File`  
  - Output: None (end of workflow)  
  - Edge Cases:  
    - Gmail quota limits or OAuth token expiry may cause sending failures.  
    - Invalid recipient email could cause errors.  
    - Attachment size limits apply.  

---

### 3. Summary Table

| Node Name                 | Node Type           | Functional Role                          | Input Node(s)           | Output Node(s)                 | Sticky Note                                                  |
|---------------------------|---------------------|----------------------------------------|-------------------------|-------------------------------|--------------------------------------------------------------|
| Schedule Trigger          | Schedule Trigger    | Initiates daily workflow                | —                       | Get Square Locations           | ## Trigger  - This workflow runs every day at 8:00 AM. - Pulls previous day's sales data. |
| Get Square Locations      | HTTP Request        | Retrieves all Square locations          | Schedule Trigger         | Turn Locations Into List       | ## Get Square Locations and Process Each One Separately - Connects to Square Locations API. |
| Turn Locations Into List  | Split Out           | Splits locations array into individual items | Get Square Locations     | Get Sales from Square          | ## Get Square Locations and Process Each One Separately - Connects to Square Locations API. |
| Get Sales from Square     | HTTP Request        | Fetches completed orders for each location | Turn Locations Into List | Ignore Locations w/o Sales     | ## Get Sales from Square - Retrieves orders for location/date. |
| Ignore Locations w/o Sales| If                  | Filters locations without sales         | Get Sales from Square    | Compile Sales Reports          |                                                              |
| Compile Sales Reports     | Code                | Calculates aggregated sales metrics     | Ignore Locations w/o Sales | Convert Sales Summary to CSV File | ## Compile a Report for Each Location - Calculates totals matching Square Sales Summary. |
| Convert Sales Summary to CSV File | Convert To File  | Converts aggregated report to CSV       | Compile Sales Reports    | Send Report                   | ## Convert the Square Sales Summary into a CSV File          |
| Send Report              | Gmail OAuth2        | Sends email with CSV report attached    | Convert Sales Summary to CSV File | —                         | ## Send the Report to the Finance Team / Manager             |
| Sticky Note               | Sticky Note         | Documentation, instructions             | —                       | —                             | ## Automatically Send Square Summary Report for Yesterday's Sales via Gmail - Full instructions and explanations. |
| Sticky Note1              | Sticky Note         | Trigger description                     | —                       | —                             | ## Trigger  - This workflow runs every day at 8:00 AM. - Pulls previous day's sales data. |
| Sticky Note2              | Sticky Note         | Fetch locations explanation             | —                       | —                             | ## Get Square Locations and Process Each One Separately - Connects to Square Locations API. |
| Sticky Note3              | Sticky Note         | Get sales explanation                   | —                       | —                             | ## Get Sales from Square - Retrieves orders for location/date. |
| Sticky Note4              | Sticky Note         | Compile report explanation              | —                       | —                             | ## Compile a Report for Each Location - Calculates totals matching Square Sales Summary. |
| Sticky Note5              | Sticky Note         | CSV conversion explanation              | —                       | —                             | ## Convert the Square Sales Summary into a CSV File          |
| Sticky Note6              | Sticky Note         | Sending report explanation              | —                       | —                             | ## Send the Report to the Finance Team / Manager             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**  
   - Set to run daily at 8:00 AM (default interval).  
   - This node initiates the workflow and provides timestamp for date calculations.

2. **Add HTTP Request node "Get Square Locations":**  
   - Method: GET  
   - URL: `https://connect.squareup.com/v2/locations`  
   - Headers: `Content-Type: application/json`  
   - Authentication: Use Header Auth credential with name `Authorization` and value `Bearer <Square Access Token>`.  
   - Connect input from Schedule Trigger.

3. **Add Split Out node "Turn Locations Into List":**  
   - Split on field: `locations`  
   - Include only field: `id`  
   - Connect input from "Get Square Locations".

4. **Add HTTP Request node "Get Sales from Square":**  
   - Method: POST  
   - URL: `https://connect.squareup.com/v2/orders/search`  
   - Headers: `Content-Type: application/json`  
   - Authentication: Same Header Auth credential as before.  
   - Body (JSON, raw input mode):  
     ```json
     {
       "location_ids": ["{{ $json.locations.id }}"],
       "query": {
         "filter": {
           "state_filter": {
             "states": ["COMPLETED"]
           },
           "date_time_filter": {
             "created_at": {
               "start_at": "{{ $('Schedule Trigger').item.json.timestamp.toDateTime().minus(1, 'days').format('yyyy-MM-dd') }}T00:00:00-05:00",
               "end_at": "{{ $('Schedule Trigger').item.json.timestamp.toDateTime().minus(1, 'days').format('yyyy-MM-dd') }}T23:59:59-05:00"
             }
           }
         }
       },
       "limit": 1000,
       "return_entries": false
     }
     ```  
   - Connect input from "Turn Locations Into List".

5. **Add If node "Ignore Locations w/o Sales":**  
   - Condition: Check if `$json.orders` array is **not empty** (operator: "array notEmpty", strict validation).  
   - Connect input from "Get Sales from Square".  
   - True output connects to next node; false output stops processing for that location.

6. **Add Code node "Compile Sales Reports":**  
   - Run mode: Run once for each item.  
   - JavaScript code: Use the comprehensive script that:  
     - Extracts date from Schedule Trigger.  
     - Finds location name from "Get Square Locations" node.  
     - Iterates orders to sum totals for sales, tax, discounts, tips, rounding, tender types, refunds, and fees.  
     - Converts cents to dollars.  
     - Returns JSON with keys: date, location_id, location_name, gross_sales, total_returns, total_discount, net_sales, total_tax, total_tip, cash_rounding, total_payments_collected, cash, card, gift_card, other, fees, net_total.  
   - Connect input from "Ignore Locations w/o Sales".

7. **Add Convert To File node "Convert Sales Summary to CSV File":**  
   - Conversion type: JSON to CSV (default).  
   - Binary property name: `sales_report`.  
   - Filename: `sales_report_{{ $('Schedule Trigger').item.json.timestamp }}.csv` (dynamic).  
   - Connect input from "Compile Sales Reports".

8. **Add Gmail node "Send Report":**  
   - Operation: Send Email  
   - Recipient email: Set your recipient’s email address (replace `user@example.com`).  
   - Subject: `Your Square Sales Report for {{ $('Schedule Trigger').item.json['Readable date'].split(',')[0] }}`  
   - Message body: HTML with polite message (can be customized).  
   - Attachments: Use binary property `sales_report` from previous node.  
   - Append attribution: Enabled or disabled as desired.  
   - Credentials: Configure Gmail OAuth2 credential with appropriate Google account credentials.  
   - Connect input from "Convert Sales Summary to CSV File".

9. **Activate the workflow** and test to ensure the report is generated and emailed successfully.

10. **Optional Enhancements:**  
    - Add pagination handling for orders exceeding 1000 per location per day.  
    - Modify Schedule Trigger for weekly reports.  
    - Customize email content or recipient list dynamically.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                       |
|---------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| This workflow builds on a previous template for pulling data from Square API into n8n: https://n8n.io/workflows/6358 | Reference workflow for Square API integration                                                      |
| To create Square credentials: Use Header Auth with name `Authorization` and value `Bearer <your-access-token>` | Credential setup instructions                                                                       |
| To customize report sending time, adjust the Schedule Trigger node's interval and timezone settings.          | Scheduling and timezone considerations                                                              |
| The workflow currently does not handle pagination for >1000 orders per location per day. Consider adding it.  | Limitation and possible enhancement                                                                 |
| Gmail OAuth2 credentials require Google API consent and appropriate scopes for sending emails.                 | Gmail API setup instructions                                                                         |
| Attribution in email can be disabled in Gmail node options if desired.                                        | Branding customization                                                                               |

---

**Disclaimer:** The provided workflow is created exclusively with n8n, respecting all applicable content policies. It processes only legal and publicly available data.