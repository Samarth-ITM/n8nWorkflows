Automatically Send Monthly Sales Reports from Square via Gmail

https://n8nworkflows.xyz/workflows/automatically-send-monthly-sales-reports-from-square-via-gmail-7090


# Automatically Send Monthly Sales Reports from Square via Gmail

### 1. Workflow Overview

This workflow automates the monthly extraction, compilation, and emailing of sales reports from Square using n8n. It is designed to run on the first day of each month at 8:00 AM, fetching the previous month’s complete sales data from all Square locations associated with the account. The workflow processes each location’s sales data individually, compiling a detailed summary report that mirrors Square’s own Sales Summary dashboard. Finally, it converts the data into a CSV file and sends it via Gmail to a specified recipient, typically a finance team or manager.

**Logical Blocks:**

- **1.1 Trigger and Date Calculation:** Starts the workflow monthly and generates a list of all dates in the previous month.
- **1.2 Fetch Locations:** Retrieves all Square locations associated with the account.
- **1.3 Process Each Location's Sales Orders:** For each location, fetches all completed orders for each date in the previous month.
- **1.4 Filter Locations Without Sales:** Skips locations on days with no completed sales.
- **1.5 Compile Sales Reports:** Aggregates sales data for each location into detailed financial metrics.
- **1.6 Convert to CSV:** Transforms the compiled sales summary into a CSV file.
- **1.7 Send Email Report:** Sends the generated CSV file as an email attachment via Gmail.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger and Date Calculation

- **Overview:**  
  Initiates the workflow monthly and calculates all calendar dates for the previous month to be used in sales data filtering.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Get Dates From Last Month

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Starts workflow execution monthly at 8:00 AM on the 1st day.  
    - Configuration: Recurrence set to monthly interval, triggered at hour 8.  
    - Inputs: None  
    - Outputs: Current timestamp to downstream nodes.  
    - Edge Cases: Workflow will not trigger if n8n instance is down or inactive at scheduled time.

  - **Get Dates From Last Month**  
    - Type: Code (JavaScript)  
    - Role: Generates ISO-formatted dates (YYYY-MM-DD) for every day of the previous calendar month.  
    - Configuration: Uses current timestamp from trigger, calculates date range, and outputs one item per day.  
    - Expressions: Utilizes `$input.first().json.timestamp` to get the trigger date.  
    - Input: Timestamp from Schedule Trigger  
    - Output: Array of JSON objects, each with a `date` field for one day of the previous month.  
    - Edge Cases: Correctly handles month boundaries and leap years; timezone assumed UTC for ISO string.  
    - Failure Modes: If timestamp is missing or malformed, code would fail.

#### 1.2 Fetch Locations

- **Overview:**  
  Retrieves all Square locations linked to the account, providing IDs and metadata for use in subsequent sales data requests.

- **Nodes Involved:**  
  - Get Square Locations  
  - Turn Locations Into List

- **Node Details:**

  - **Get Square Locations**  
    - Type: HTTP Request  
    - Role: Calls Square API endpoint `/v2/locations` to retrieve all locations.  
    - Configuration: GET request with `Content-Type: application/json` header. Uses Header Auth credential configured with Square Access Token.  
    - Input: Dates from prior node (batched per date)  
    - Output: JSON containing `locations` array.  
    - Edge Cases: API rate limits or authentication failures; network issues.  
    - Version: HTTP Request node version 4.2.

  - **Turn Locations Into List**  
    - Type: SplitOut  
    - Role: Splits the `locations` array into individual items, each containing a location ID for individual processing.  
    - Configuration: Splits on `locations` array, outputs just the `id` field per item.  
    - Input: Output from Get Square Locations  
    - Output: One item per location ID.  
    - Edge Cases: Empty locations array results in no downstream processing.

#### 1.3 Process Each Location's Sales Orders

- **Overview:**  
  For each location and each date in the previous month, retrieves all completed sales orders from Square.

- **Nodes Involved:**  
  - Get Sales from Square  
  - Ignore Locations w/o Sales

- **Node Details:**

  - **Get Sales from Square**  
    - Type: HTTP Request  
    - Role: POSTs to Square API endpoint `/v2/orders/search` to retrieve completed orders filtered by location and date.  
    - Configuration:  
      - JSON body includes:  
        - `location_ids` array with current location ID  
        - Filter for `COMPLETED` state orders  
        - `date_time_filter` for `created_at` using current date item (`start_at` and `end_at` set to the day in ISO string with fixed -05:00 timezone offset)  
      - Limit: 1000 orders per request (no pagination implemented)  
      - Headers: `Content-Type: application/json`  
      - Authentication: Header Auth credential with Square Access Token  
      - Executes once per item (location-date pairs)  
    - Input: Location ID items from SplitOut, dates from previous block  
    - Output: JSON containing `orders` array for that location and date.  
    - Edge Cases: Locations with zero orders, API failures, rate limits, timezone assumptions (fixed offset).  
    - Failure Modes: If orders exceed 1000, data truncation possible; consider pagination enhancement.

  - **Ignore Locations w/o Sales**  
    - Type: If  
    - Role: Filters out locations/dates where no sales (`orders` array is empty). Only passes on items with non-empty orders.  
    - Configuration: Condition checks if `orders` array is not empty using array operation.  
    - Input: Output from Get Sales from Square  
    - Output: Only locations/dates with sales data proceed.  
    - Edge Cases: Empty orders array branches filtered out, avoiding unnecessary processing.

#### 1.4 Compile Sales Reports

- **Overview:**  
  Aggregates sales orders per location into financial metrics approximating Square’s Sales Summary. Handles subtotals, taxes, discounts, tips, refunds, and payment tender types.

- **Nodes Involved:**  
  - Compile Sales Reports

- **Node Details:**

  - **Compile Sales Reports**  
    - Type: Code (JavaScript)  
    - Role: Processes the orders array to compute totals for gross sales, returns, discounts, tips, taxes, rounding adjustments, payment tenders (cash, card, gift card, other), and fees.  
    - Configuration:  
      - Reads date from “Get Dates From Last Month” node  
      - Matches location ID and name from “Get Square Locations” node  
      - Iterates through each sale order and its refunds  
      - Calculates net and gross sales values in cents, converts to dollars (divides by 100)  
      - Returns a JSON object with all computed fields (e.g., `gross_sales`, `net_sales`, `total_returns`, `total_discount`, `cash`, `card`, `fees`, etc.)  
    - Input: Orders array from filtered locations/dates  
    - Output: Structured JSON report per location per date  
    - Edge Cases: Handles missing data gracefully with default zeros; assumes consistent data structure from Square API.  
    - Failure Modes: Code errors if expected fields missing or data malformed; careful validation recommended.  
    - Note: Designed to match Square Sales Summary dashboard exactly.

#### 1.5 Convert to CSV

- **Overview:**  
  Converts the compiled JSON sales reports into a CSV file for easy distribution and analysis.

- **Nodes Involved:**  
  - Convert Sales Summary to CSV File

- **Node Details:**

  - **Convert Sales Summary to CSV File**  
    - Type: Convert To File  
    - Role: Converts JSON data into a CSV file binary property for attachment.  
    - Configuration:  
      - Filename set dynamically as `sales_report_<timestamp>.csv` using the timestamp from Schedule Trigger node.  
      - Output binary property named `sales_report`.  
    - Input: JSON from Compile Sales Reports  
    - Output: Binary CSV file ready for email attachment  
    - Edge Cases: Large datasets may impact performance; CSV formatting assumptions.

#### 1.6 Send Email Report

- **Overview:**  
  Sends the generated CSV sales report via Gmail to a designated recipient with a formatted message.

- **Nodes Involved:**  
  - Send Report

- **Node Details:**

  - **Send Report**  
    - Type: Gmail  
    - Role: Sends an email with the CSV file attached to a specified recipient.  
    - Configuration:  
      - `sendTo`: Configurable email address (`rosh.edwin15@gmail.com` in this example)  
      - `subject`: Dynamic subject line including “Your Last Month’s Square Sales Report”  
      - `message`: HTML email body with a friendly message  
      - Attachment included from binary property `sales_report`  
      - Uses Gmail OAuth2 credentials configured for sender’s Gmail account  
    - Input: CSV file from previous node  
    - Output: None (terminal node)  
    - Edge Cases: Gmail API quota limits, authentication errors, invalid email addresses, attachment size limits.

---

### 3. Summary Table

| Node Name                  | Node Type         | Functional Role                              | Input Node(s)             | Output Node(s)               | Sticky Note                                                 |
|----------------------------|-------------------|----------------------------------------------|---------------------------|-----------------------------|-------------------------------------------------------------|
| Schedule Trigger           | Schedule Trigger  | Starts workflow monthly at 8 AM              |                           | Get Dates From Last Month    | ## Trigger  - This workflow runs on the first day of every Month.  - Each month, it pulls the previous month's sales data from Square. |
| Get Dates From Last Month  | Code              | Generates all dates for previous month       | Schedule Trigger          | Get Square Locations         |                                                             |
| Get Square Locations       | HTTP Request      | Retrieves all Square locations                | Get Dates From Last Month | Turn Locations Into List     | ## Get Square Locations and Process Each One Separately  - This HTTP node connects to the Square Locations API to fetch all your locations.  |
| Turn Locations Into List   | SplitOut          | Splits locations array into individual items | Get Square Locations       | Get Sales from Square        |                                                             |
| Get Sales from Square      | HTTP Request      | Fetches completed orders for location/date   | Turn Locations Into List   | Ignore Locations w/o Sales   | ## Get Sales from Square  - This HTTP node retrieves all orders for the given location on the specified date. |
| Ignore Locations w/o Sales | If                | Filters out locations/dates with no sales    | Get Sales from Square      | Compile Sales Reports        |                                                             |
| Compile Sales Reports      | Code              | Aggregates orders into detailed sales report | Ignore Locations w/o Sales | Convert Sales Summary to CSV File | ## Compile a Report for Each Location  - This code node calculates totals for each location.  - Please ensure the numbers match EXACTLY with the Square Sales Summary Dashboard. |
| Convert Sales Summary to CSV File | Convert To File | Converts JSON report into CSV file            | Compile Sales Reports       | Send Report                 | ## Convert the Square Sales Summary into a CSV File         |
| Send Report               | Gmail             | Sends CSV sales report by email               | Convert Sales Summary to CSV File |                             | ## Send the Report to the Finance Team / Manager             |
| Sticky Note1              | Sticky Note       | Documentation                                 |                           |                             | ## Trigger  - This workflow runs on the first day of every Month.  - Each month, it pulls the previous month's sales data from Square. |
| Sticky Note2              | Sticky Note       | Documentation                                 |                           |                             | ## Get Square Locations and Process Each One Separately  - This HTTP node connects to the Square Locations API to fetch all your locations. |
| Sticky Note3              | Sticky Note       | Documentation                                 |                           |                             | ## Get Sales from Square  - This HTTP node retrieves all orders for the given location on the specified date. |
| Sticky Note4              | Sticky Note       | Documentation                                 |                           |                             | ## Compile a Report for Each Location  - This code node calculates totals for each location.  - Please ensure the numbers match EXACTLY with the Square Sales Summary Dashboard. |
| Sticky Note5              | Sticky Note       | Documentation                                 |                           |                             | ## Convert the Square Sales Summary into a CSV File           |
| Sticky Note6              | Sticky Note       | Documentation                                 |                           |                             | ## Send the Report to the Finance Team / Manager              |
| Sticky Note7              | Sticky Note       | Overview and setup instructions               |                           |                             | ## Automatically Send Monthly Sales Reports from Square via Gmail  [Full details and setup instructions included.] |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Set it to trigger monthly on the 1st day at 8:00 AM.  
   - No credentials required.

2. **Add a Code node named “Get Dates From Last Month”**  
   - Use JavaScript to generate an array of all dates (YYYY-MM-DD) for the previous month based on the trigger timestamp.  
   - Input: Connect from Schedule Trigger.  
   - Output: One item per date.

3. **Add an HTTP Request node named “Get Square Locations”**  
   - Method: GET  
   - URL: `https://connect.squareup.com/v2/locations`  
   - Headers: `Content-Type: application/json`  
   - Authentication: Use a Header Auth credential where the header `Authorization` is set to `Bearer <Your Square Access Token>`.  
   - Input: Connect from “Get Dates From Last Month”.  
   - Output: JSON with `locations` array.

4. **Add a SplitOut node named “Turn Locations Into List”**  
   - Set field to split out: `locations`  
   - Include only the `id` field in output items.  
   - Input: Connect from “Get Square Locations”.  
   - Output: One item per location.

5. **Add an HTTP Request node named “Get Sales from Square”**  
   - Method: POST  
   - URL: `https://connect.squareup.com/v2/orders/search`  
   - Headers: `Content-Type: application/json`  
   - Authentication: Same Header Auth credential as above.  
   - Body (JSON):  
     ```json
     {
       "location_ids": ["{{ $json.locations.id }}"],
       "query": {
         "filter": {
           "state_filter": { "states": ["COMPLETED"] },
           "date_time_filter": {
             "created_at": {
               "start_at": "{{ $json.date }}T00:00:00-05:00",
               "end_at": "{{ $json.date }}T23:59:59-05:00"
             }
           }
         }
       },
       "limit": 1000,
       "return_entries": false
     }
     ```  
   - Input: Connect from “Turn Locations Into List”.  
   - Execute once per item. Output: `orders` array.

6. **Add an If node named “Ignore Locations w/o Sales”**  
   - Condition: Check that `orders` array is not empty (`array operation: notEmpty`)  
   - Input: Connect from “Get Sales from Square”.  
   - Output: Only locations/dates with sales pass to next node.

7. **Add a Code node named “Compile Sales Reports”**  
   - Use JavaScript to iterate over all orders and compute totals for gross sales, discounts, tips, taxes, returns, payment tenders, fees, and rounding adjustments.  
   - Access location metadata from “Get Square Locations” node.  
   - Normalize amounts from cents to dollars (divide by 100).  
   - Input: Connect from “Ignore Locations w/o Sales”.  
   - Output: JSON report per location and date.

8. **Add a Convert To File node named “Convert Sales Summary to CSV File”**  
   - Configure to convert the JSON data to CSV format.  
   - Set output binary property: `sales_report`  
   - Filename: `sales_report_{{ $('Schedule Trigger').item.json.timestamp }}.csv`  
   - Input: Connect from “Compile Sales Reports”.

9. **Add a Gmail node named “Send Report”**  
   - Configure recipient email address in `sendTo` field.  
   - Subject: `"Your Last Month's Square Sales Report"`  
   - Message: HTML content welcoming the recipient and referencing the attached report.  
   - Attach the binary CSV file from `sales_report`.  
   - Credentials: Configure Gmail OAuth2 credential for the sender email account.  
   - Input: Connect from “Convert Sales Summary to CSV File”.  
   - No output (terminal node).

10. **Activate the workflow**  
    - Ensure all credentials are correctly set up.  
    - Confirm schedule trigger is enabled.  
    - Test with manual executions if needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                                  | Context or Link                                                                                                                     |
|-------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------|
| The workflow runs automatically on the 1st day of each month at 8 AM and pulls the previous month’s data.                     | Sticky Note1                                                                                                                      |
| Square API credentials require Header Auth setup with `Authorization: Bearer <token>`.                                         | Setup instructions in Sticky Note7                                                                                                |
| The sales report matches exactly the Square Sales Summary Dashboard figures, enabling consistent financial analysis.            | Sticky Note4                                                                                                                      |
| Consider adding pagination handling for scenarios with more than 1000 orders per location per day.                              | Customization suggestion in Sticky Note7                                                                                          |
| CSV attachment filename includes the timestamp of the workflow trigger for easy identification.                                | Defined in “Convert Sales Summary to CSV File” node                                                                               |
| Gmail credentials must be OAuth2, configured for the sender’s Gmail account with appropriate scopes to send emails.             | Credential setup required for “Send Report” node                                                                                   |
| Workflow example and detailed description available at https://n8n.io/workflows/6358                                             | Reference link in Sticky Note7                                                                                                    |
| Adjust timezone offsets in the HTTP Request body if your Square account operates in a different timezone than -05:00 (EST).    | Timezone fixed in HTTP node JSON body; may need adjustment based on user locale                                                     |

---

**Disclaimer:** The provided content is derived exclusively from an automated n8n workflow. It complies strictly with content policies and contains no illegal or protected material. All handled data is lawful and public.