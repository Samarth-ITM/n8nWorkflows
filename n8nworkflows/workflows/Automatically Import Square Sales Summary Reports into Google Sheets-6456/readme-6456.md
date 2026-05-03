Automatically Import Square Sales Summary Reports into Google Sheets

https://n8nworkflows.xyz/workflows/automatically-import-square-sales-summary-reports-into-google-sheets-6456


# Automatically Import Square Sales Summary Reports into Google Sheets

### 1. Workflow Overview

This workflow automates the daily import of sales summary reports from the Square API into a Google Sheet. It is designed for businesses using Square POS to track sales data across multiple locations. Running daily, it fetches all registered Square locations, retrieves completed orders from the previous day for each location, filters out locations without sales, compiles detailed sales summaries matching Square’s Sales Summary report, and appends the results to a Google Sheet for record-keeping and analysis.

**Logical Blocks:**

- **1.1 Trigger and Scheduling**: Initiates the workflow every day at a fixed time.
- **1.2 Retrieve Square Locations**: Fetches all business locations from Square.
- **1.3 Iterate Over Locations & Fetch Sales Data**: For each location, get all completed orders for the previous day.
- **1.4 Filter Locations Without Sales**: Skip locations with no sales data.
- **1.5 Compile Sales Summary Report**: Aggregate order data into a comprehensive sales summary.
- **1.6 Upload to Google Sheets**: Append the compiled report data to a Google Sheet.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger and Scheduling

- **Overview:**  
  This block triggers the workflow automatically every day at 4:00 AM and provides the timestamp context for querying the previous day’s sales data.

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Configuration: Runs with default daily interval (every 24 hours), implicitly at 4:00 AM as per workflow notes. Provides a timestamp (`timestamp`) used for date calculations downstream.  
    - Inputs: None (start node)  
    - Outputs: Triggers "Get Square Locations" node.  
    - Edge Cases: Misconfiguration could cause trigger to run at unintended times; timezone differences could affect the “previous day” calculation.  
    - Notes: The timestamp is used to calculate the previous day's date for all subsequent API queries.

#### 1.2 Retrieve Square Locations

- **Overview:**  
  Makes an HTTP GET request to the Square API to retrieve all locations associated with the Square account, allowing the workflow to process sales data location-by-location.

- **Nodes Involved:**  
  - Get Square Locations  
  - Turn Locations Into List

- **Node Details:**

  - **Get Square Locations**  
    - Type: HTTP Request  
    - Configuration:  
      - Method: GET  
      - URL: `https://connect.squareup.com/v2/locations`  
      - Headers: Content-Type: application/json, plus authorization header via generic HTTP Header Auth credential holding Square API Bearer token.  
    - Inputs: Trigger from Schedule Trigger  
    - Outputs: JSON containing an array of locations under `locations`.  
    - Edge Cases: API rate limits, invalid or expired credentials, network errors.  
    - Credential: Square Header Auth (HTTP Header Auth with Bearer token).  
    - Notes: Retrieves all business locations for further processing.

  - **Turn Locations Into List**  
    - Type: Split Out  
    - Configuration:  
      - Field to split out: `locations`  
      - Output each location as a separate item  
      - Includes only `id` from each location in the output (minimal data to identify locations).  
    - Inputs: "Get Square Locations" node  
    - Outputs: One output item per location to process sales individually.  
    - Edge Cases: Empty locations list (no locations configured in Square).  
    - Notes: Enables iteration over each location in subsequent steps.

#### 1.3 Iterate Over Locations & Fetch Sales Data

- **Overview:**  
  For each location, this block sends a POST request to Square’s Orders Search API to retrieve all completed orders from the previous day.

- **Nodes Involved:**  
  - Get Sales from Square  
  - Ignore Locations w/o Sales (next block)  

- **Node Details:**

  - **Get Sales from Square**  
    - Type: HTTP Request  
    - Configuration:  
      - Method: POST  
      - URL: `https://connect.squareup.com/v2/orders/search`  
      - Body: JSON with query filter for:  
        - `location_ids`: The current location's ID from `$json.locations.id`  
        - `state_filter`: Only `COMPLETED` orders  
        - `date_time_filter`: `created_at` between previous day’s start and end, computed dynamically using the timestamp from the Schedule Trigger node minus one day, formatted with timezone offset `-05:00`  
      - Limit: 1000 orders (no pagination implemented)  
      - Headers: Content-Type: application/json, plus authorization header as above  
      - Authentication: Generic HTTP Header Auth (Square Header Auth)  
    - Inputs: One per location from "Turn Locations Into List"  
    - Outputs: JSON containing orders array under `orders` for the location.  
    - Edge Cases: More than 1000 orders (would require pagination), API rate limits, invalid credentials, empty orders array.  
    - Notes: Retrieves sales data needed for report compilation.

#### 1.4 Filter Locations Without Sales

- **Overview:**  
  This block filters out any location items where no sales (orders) were found, preventing unnecessary processing downstream.

- **Nodes Involved:**  
  - Ignore Locations w/o Sales

- **Node Details:**

  - **Ignore Locations w/o Sales**  
    - Type: If  
    - Configuration:  
      - Condition: Checks if the `orders` array is NOT empty (`notEmpty` condition on `$json.orders`)  
    - Inputs: From "Get Sales from Square"  
    - Outputs:  
      - True: Locations with sales (continue to compilation)  
      - False: Locations without sales (discarded)  
    - Edge Cases: Malformed or missing `orders` property; empty sales data.  
    - Notes: Ensures only meaningful data is processed further.

#### 1.5 Compile Sales Summary Report

- **Overview:**  
  Processes the orders data for each location with JavaScript code to compute a detailed sales summary, mirroring Square’s Sales Summary report, including totals for sales, taxes, discounts, tips, returns, payment types, fees, and rounding adjustments.

- **Nodes Involved:**  
  - Compile Sales Reports

- **Node Details:**

  - **Compile Sales Reports**  
    - Type: Code (JavaScript)  
    - Configuration:  
      - Runs once per location (mode: runOnceForEachItem)  
      - Uses input JSON orders array and metadata from Schedule Trigger and Get Square Locations nodes  
      - Computes:  
        - date, location_id, location_name  
        - gross_sales, total_returns, total_discount, net_sales, total_tax, total_tip, cash_rounding  
        - payment breakdowns: cash, card, gift_card, other, fees, net_total  
      - Handles returns and refunds by adjusting totals accordingly  
      - Converts amounts from cents to dollars by dividing by 100  
    - Inputs: Filtered locations with orders  
    - Outputs: Single JSON summary object per location with aggregated sales data fields  
    - Edge Cases: Missing or incomplete order data, null or undefined properties, no orders after filtering, refunds referencing missing original sales.  
    - Notes: Critical for ensuring report accuracy and parity with Square’s dashboard.

#### 1.6 Upload to Google Sheets

- **Overview:**  
  Appends each compiled sales summary as a new row in a designated Google Sheet for centralized storage and analysis.

- **Nodes Involved:**  
  - Upload Sales to Google Sheets

- **Node Details:**

  - **Upload Sales to Google Sheets**  
    - Type: Google Sheets  
    - Configuration:  
      - Operation: Append  
      - Document ID: a specific Google Sheet document (ID `15_kD4zmytEy-Rcpti_etCYSE_T4dJJiY0FkDgE1AbnA`)  
      - Sheet Name: "Data" (sheet id `1213795200`)  
      - Columns mapped explicitly from input JSON fields (e.g., Date, Location, Gross Sales, Returns, Discounts, Net Sales, Taxes, Tips, Cash Rounding, Total Money, Cash, Card, Gift Card, Other, Fees, Net Total)  
      - Data type conversion disabled (all values treated as strings or raw)  
    - Inputs: Aggregated sales summary JSON per location  
    - Outputs: Confirmation of row append (not further processed)  
    - Credential: Google Sheets OAuth2 Account  
    - Edge Cases: Sheet access permissions, quota limits, invalid document ID or sheet name, network issues.  
    - Notes: Spreadsheet columns must be pre-created matching the mapping. See Sticky Note6 for recommended columns.

---

### 3. Summary Table

| Node Name                | Node Type          | Functional Role                       | Input Node(s)        | Output Node(s)               | Sticky Note                                                                                                                                            |
|--------------------------|--------------------|------------------------------------|----------------------|-----------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger         | Schedule Trigger   | Initiate workflow daily            | -                    | Get Square Locations         | ## Trigger  \n- This workflow runs every day at 4:00 AM.  \n- Each day, it pulls the previous day's sales data from Square.                           |
| Get Square Locations     | HTTP Request      | Fetch all Square locations         | Schedule Trigger     | Turn Locations Into List     | ## Get Square Locations and Process Each One Separately  \n- This HTTP node connects to the Square Locations API to fetch all your locations.          |
| Turn Locations Into List | Split Out         | Create one item per location       | Get Square Locations | Get Sales from Square        |                                                                                                                                                        |
| Get Sales from Square    | HTTP Request      | Fetch completed orders per location| Turn Locations Into List | Ignore Locations w/o Sales | ## Get Sales from Square  \n- This HTTP node retrieves all orders for the given location on the specified date.                                         |
| Ignore Locations w/o Sales | If               | Filter out locations without sales | Get Sales from Square| Compile Sales Reports        |                                                                                                                                                        |
| Compile Sales Reports    | Code              | Aggregate sales data per location  | Ignore Locations w/o Sales | Upload Sales to Google Sheets | ## Compile a Report for Each Location  \n- This code node calculates totals for each location.  \n- Please ensure the numbers match EXACTLY with the Square Sales Summary Dashboard. |
| Upload Sales to Google Sheets | Google Sheets | Append summary to Google Sheet     | Compile Sales Reports | -                           | ## Upload Each Report to Google Sheets  \n- Each location's sales will be uploaded to a Google Sheet for easy analysis and record-keeping.              |
| Sticky Note              | Sticky Note       | Workflow documentation and overview| -                    | -                           | ## Automatically Pull Square Report Data Into Google Sheets\n\nDetailed explanation of workflow purpose, setup, and use cases.                         |
| Sticky Note1             | Sticky Note       | Trigger explanation                | -                    | -                           | ## Trigger  \n- This workflow runs every day at 4:00 AM.  \n- Each day, it pulls the previous day's sales data from Square.                           |
| Sticky Note2             | Sticky Note       | Locations retrieval explanation    | -                    | -                           | ## Get Square Locations and Process Each One Separately  \n- This HTTP node connects to the Square Locations API to fetch all your locations.          |
| Sticky Note3             | Sticky Note       | Sales retrieval explanation        | -                    | -                           | ## Get Sales from Square  \n- This HTTP node retrieves all orders for the given location on the specified date.                                         |
| Sticky Note4             | Sticky Note       | Sales summary compilation explanation | -                  | -                           | ## Compile a Report for Each Location  \n- This code node calculates totals for each location.  \n- Please ensure the numbers match EXACTLY with the Square Sales Summary Dashboard. |
| Sticky Note5             | Sticky Note       | Google Sheets upload explanation   | -                    | -                           | ## Upload Each Report to Google Sheets  \n- Each location's sales will be uploaded to a Google Sheet for easy analysis and record-keeping.              |
| Sticky Note6             | Sticky Note       | Google Sheet columns guidance      | -                    | -                           | ## Setting up Your Google Sheet\nHere are the columns you can create in your Google sheet:\n- Date\n- Location ID\n- Location Name\n- Gross Sales\n- Discounts\n- Returns\n- Net Sales\n- Taxes\n- Tips\n- Cash Rounding\n- Total Money Collected\n- Cash\n- Card\n- Gift Card\n- Other Payment Method\n- Fees\n- Net Total |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Configure to run daily (every 24 hours).  
   - This node triggers the workflow and provides a timestamp for date calculations.

2. **Create HTTP Request Node: Get Square Locations**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://connect.squareup.com/v2/locations`  
   - Headers: Set `Content-Type: application/json`  
   - Authentication: Generic HTTP Header Auth with Square API Bearer token (Credential setup below).  
   - Connect input from Schedule Trigger node.

3. **Create Split Out Node: Turn Locations Into List**  
   - Type: Split Out  
   - Field to split out: `locations`  
   - Include only `id` field in output for each location.  
   - Connect input from "Get Square Locations".

4. **Create HTTP Request Node: Get Sales from Square**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://connect.squareup.com/v2/orders/search`  
   - Headers: `Content-Type: application/json`  
   - Body (JSON):  
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
   - Authentication: Generic HTTP Header Auth with Square API Bearer token.  
   - Connect input from "Turn Locations Into List".

5. **Create If Node: Ignore Locations w/o Sales**  
   - Type: If  
   - Condition: Check if `$json.orders` array is not empty (`notEmpty` operator).  
   - Connect input from "Get Sales from Square".  
   - Connect True output to next node; False output ignored.

6. **Create Code Node: Compile Sales Reports**  
   - Type: Code (JavaScript)  
   - Run once for each item.  
   - Copy the JavaScript code from the "Compile Sales Reports" node in the original workflow.  
   - This code aggregates sales data from orders into a summary object.  
   - Connect input from "Ignore Locations w/o Sales" True output.

7. **Create Google Sheets Node: Upload Sales to Google Sheets**  
   - Type: Google Sheets  
   - Operation: Append  
   - Document ID: Set to your target Google Sheet ID (e.g., `15_kD4zmytEy-Rcpti_etCYSE_T4dJJiY0FkDgE1AbnA`).  
   - Sheet Name: Set to your sheet tab name or ID (e.g., `"Data"`).  
   - Columns: Define columns matching the fields output by the code node (Date, Location ID, Location, Gross Sales, Returns, Total Discount, Net Sales, Taxes, Tips, Cash Rounding, Total Money, Cash, Card, Gift Card, Other, Fees, Net Total).  
   - Connect input from "Compile Sales Reports".

8. **Credential Setup: Square API (HTTP Header Auth)**  
   - Create new credential of type HTTP Header Auth.  
   - Set Header Name: `Authorization`  
   - Set Header Value: `Bearer <your-square-access-token>` (replace with your actual token).

9. **Credential Setup: Google Sheets OAuth2**  
   - Create or use existing Google Sheets OAuth2 credential with access to the target spreadsheet.

10. **Activate Workflow**  
    - Once all nodes are connected and credentials configured, activate the workflow to run automatically.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                      | Context or Link                                                                                   |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow automatically connects to the Square API and generates daily sales summaries matching the Square Dashboard's Sales Summary report. It is designed for automatic daily execution, pulling previous day's sales data for all locations into a Google Sheet.                                                                                               | Sticky Note (Workflow Overview)                                                                  |
| To set up Square credentials, create an HTTP Header Auth credential with header name `Authorization` and value `Bearer <your-api-key>`.                                                                                                                                                                                                                          | Sticky Note (Credential Setup)                                                                   |
| Google Sheet columns should be pre-created matching these headers: Date, Location ID, Location Name, Gross Sales, Discounts, Returns, Net Sales, Taxes, Tips, Cash Rounding, Total Money Collected, Cash, Card, Gift Card, Other Payment Method, Fees, Net Total to ensure proper data mapping.                                                                       | Sticky Note6 (Google Sheets Setup)                                                              |
| The workflow currently does not implement pagination for orders beyond 1,000 per location per day; consider extending if needed.                                                                                                                                                                                                                                | Sticky Note (Customization Options)                                                             |
| The code node is critical to ensure the report matches Square’s official Sales Summary exactly, including handling of returns, refunds, and payment fees.                                                                                                                                                                                                        | Sticky Note4 (Code Explanation)                                                                 |
| Error handling considerations include API rate limits, credential expiration, network errors, and potential data inconsistencies in Square order data. Proper monitoring and alerting should be implemented externally if used in production.                                                                                                                  | General best practice note                                                                       |

---

**Disclaimer:** The provided text is extracted exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.