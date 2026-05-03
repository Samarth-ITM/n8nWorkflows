Automatically Send Weekly Sales Reports from Square via Gmail

https://n8nworkflows.xyz/workflows/automatically-send-weekly-sales-reports-from-square-via-gmail-7088


# Automatically Send Weekly Sales Reports from Square via Gmail

### 1. Workflow Overview

This workflow automates the generation and emailing of weekly sales reports from Square using Gmail. It is designed to run every Monday at 8:00 AM and pulls sales data from the previous week for all Square locations.

**Target Use Cases:**  
- Automating weekly sales report delivery to management or finance teams.  
- Sharing sales data with external parties such as landlords, agents, or bookkeepers.  
- Reducing manual effort in extracting and compiling sales data from Square.

**Logical Blocks:**

- **1.1 Trigger and Date Calculation**  
  Initiates the workflow weekly and calculates the dates for the previous week.

- **1.2 Fetch Square Locations**  
  Retrieves all Square locations linked to the account and prepares them for individual processing.

- **1.3 Fetch and Filter Sales Orders**  
  For each location, retrieves completed orders for the previous week and filters out locations without sales.

- **1.4 Compile Sales Reports**  
  Processes order data per location to calculate sales metrics matching Square’s Sales Summary dashboard.

- **1.5 Generate CSV File**  
  Converts the compiled sales data into a CSV file.

- **1.6 Email the Report**  
  Sends the CSV report via Gmail to a specified recipient.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger and Date Calculation

**Overview:**  
This block triggers the workflow every Monday at 8:00 AM and calculates the dates for the previous week (Monday to Sunday) to use in subsequent API requests.

**Nodes Involved:**  
- Schedule Trigger  
- Get Dates From Last Week

**Node Details:**

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Initiates workflow execution weekly on Monday at 8:00 AM.  
  - Configuration: Interval set to weekly, triggering on Monday at 08:00.  
  - Inputs: None (trigger node).  
  - Outputs: Timestamp for current run.  
  - Potential Failures: None typical; ensure system time is correct.  

- **Get Dates From Last Week**  
  - Type: Code Node (JavaScript)  
  - Role: Computes date strings for the last week based on trigger timestamp.  
  - Configuration: Uses the trigger timestamp and subtracts 7 days, outputs an array of date JSON objects in ISO format without time (YYYY-MM-DD).  
  - Inputs: Timestamp from Schedule Trigger.  
  - Outputs: List of dates for the previous week as JSON objects.  
  - Expressions: Uses JavaScript date manipulation.  
  - Edge Cases: Time zone handling assumes ISO format; ensure time zone consistency with Square API.  

---

#### 1.2 Fetch Square Locations

**Overview:**  
Fetches all Square locations associated with the account, then splits the returned locations array into individual items for further processing.

**Nodes Involved:**  
- Get Square Locations  
- Turn Locations Into List

**Node Details:**

- **Get Square Locations**  
  - Type: HTTP Request  
  - Role: Calls Square API endpoint `/v2/locations` to retrieve all locations.  
  - Configuration:  
    - URL: `https://connect.squareup.com/v2/locations`  
    - Method: GET (default)  
    - Headers: Content-Type: application/json  
    - Authentication: Header Auth using Square API key (Bearer token).  
  - Inputs: Dates array from previous block (single item triggers request).  
  - Outputs: JSON containing all locations.  
  - Potential Failures: Authentication errors, API rate limits, network errors.  

- **Turn Locations Into List**  
  - Type: Split Out  
  - Role: Splits the locations array into individual items with location IDs.  
  - Configuration: Field to split out: `locations`; includes only `id` field for each location.  
  - Inputs: Output of Get Square Locations node.  
  - Outputs: One item per location ID.  
  - Edge Cases: Empty locations array results in no further processing.  

---

#### 1.3 Fetch and Filter Sales Orders

**Overview:**  
For each location, fetches completed orders from the previous week, then filters out locations with no sales data.

**Nodes Involved:**  
- Get Sales from Square  
- Ignore Locations w/o Sales

**Node Details:**

- **Get Sales from Square**  
  - Type: HTTP Request  
  - Role: Calls Square API `/v2/orders/search` endpoint to retrieve completed orders for a specific location and date.  
  - Configuration:  
    - Method: POST  
    - URL: `https://connect.squareup.com/v2/orders/search`  
    - Headers: Content-Type: application/json  
    - Body: JSON with `location_ids` set to current location ID, filter for orders with state "COMPLETED", and date filter for the current date from previous week dates.  
    - Limit: 1000 orders  
    - Authentication: Header Auth with Square API key.  
  - Expressions: Uses location ID from split list and date from "Get Dates From Last Week".  
  - Inputs: Location IDs from previous node.  
  - Outputs: Orders JSON for that location and date.  
  - Edge Cases: No orders returned, API errors, request limit exceeding 1000 orders (pagination not implemented).  

- **Ignore Locations w/o Sales**  
  - Type: If Node  
  - Role: Checks if the orders array is not empty; only passes locations with sales.  
  - Configuration: Condition checks if `$json.orders` array is not empty.  
  - Inputs: Orders data from previous node.  
  - Outputs: Continues flow only for locations with sales orders.  
  - Edge Cases: Empty orders array leads to skipping location.  

---

#### 1.4 Compile Sales Reports

**Overview:**  
Calculates detailed sales metrics per location based on the orders data to match Square’s Sales Summary report exactly.

**Nodes Involved:**  
- Compile Sales Reports

**Node Details:**

- **Compile Sales Reports**  
  - Type: Code Node (JavaScript)  
  - Role: Processes all orders for a location, summing totals, taxes, discounts, tips, returns, tenders, fees, and rounding adjustments.  
  - Configuration: Runs once for each item (location) with detailed looping and calculations in JavaScript.  
  - Inputs: Orders JSON from previous node. Also accesses location metadata from "Get Square Locations" and date from "Get Dates From Last Week".  
  - Outputs: JSON containing aggregated sales metrics, including gross sales, net sales, discounts, returns, taxes, tips, tenders (cash, card, gift card), fees, and net total.  
  - Expressions: Uses complex JavaScript to parse nested order data and handle refunds and rounding adjustments carefully.  
  - Edge Cases:  
    - Missing fields in orders or tenders may cause undefined values; code uses defaults.  
    - Refunds and returns are handled specifically; ensure data consistency.  
    - Large orders array may impact performance.  
  - Note: Ensure the calculated numbers match Square’s Sales Summary dashboard exactly to maintain data trustworthiness.  

---

#### 1.5 Generate CSV File

**Overview:**  
Converts the compiled sales reports into a CSV file with a dynamic filename based on the trigger timestamp.

**Nodes Involved:**  
- Convert Sales Summary to CSV File

**Node Details:**

- **Convert Sales Summary to CSV File**  
  - Type: Convert To File  
  - Role: Converts JSON sales report data into a CSV file binary object.  
  - Configuration:  
    - File name template: `sales_report_{{ $('Schedule Trigger').item.json.timestamp }}.csv` (timestamp used to ensure uniqueness).  
    - Binary property name: `sales_report`.  
  - Inputs: Aggregated sales report JSON from previous node.  
  - Outputs: Binary CSV file ready for attachment.  
  - Edge Cases: Ensure data fields are flattened properly for CSV format.  

---

#### 1.6 Email the Report

**Overview:**  
Sends the generated CSV sales report via Gmail to a specified recipient with a formatted message.

**Nodes Involved:**  
- Send Report

**Node Details:**

- **Send Report**  
  - Type: Gmail Node  
  - Role: Sends an email with the CSV sales report attached.  
  - Configuration:  
    - Recipient email: `example@user.com` (to be customized).  
    - Email subject: Includes the readable date from the schedule trigger (e.g., "Your Square Sales Report for ...").  
    - Message: HTML formatted body with greeting and report description.  
    - Attachment: Uses the CSV binary file from the previous node.  
  - Credentials: Gmail OAuth2 credential configured for the user sending the email.  
  - Inputs: Binary CSV file from Convert to File node.  
  - Outputs: None.  
  - Edge Cases: Authentication failures, quota limits on Gmail API, invalid email addresses.  

---

### 3. Summary Table

| Node Name                    | Node Type           | Functional Role                         | Input Node(s)                | Output Node(s)               | Sticky Note                                                                                                                                                                               |
|------------------------------|---------------------|---------------------------------------|-----------------------------|-----------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger              | Schedule Trigger    | Starts workflow weekly at Monday 8 AM | None                        | Get Dates From Last Week     | ## Trigger  - This workflow runs every Monday at 8:00 AM.  - Each week, it pulls the previous week's sales data from Square.                                                             |
| Get Dates From Last Week      | Code                | Calculates previous week's dates      | Schedule Trigger            | Get Square Locations         |                                                                                                                                                                                           |
| Get Square Locations          | HTTP Request        | Retrieves all Square locations         | Get Dates From Last Week    | Turn Locations Into List     | ## Get Square Locations and Process Each One Separately  - This HTTP node connects to the Square Locations API to fetch all your locations.                                               |
| Turn Locations Into List      | Split Out           | Splits locations array into single items | Get Square Locations        | Get Sales from Square        |                                                                                                                                                                                           |
| Get Sales from Square         | HTTP Request        | Retrieves orders for each location     | Turn Locations Into List    | Ignore Locations w/o Sales   | ## Get Sales from Square  - This HTTP node retrieves all orders for the given location on the specified date.                                                                              |
| Ignore Locations w/o Sales    | If                  | Filters out locations with no sales    | Get Sales from Square       | Compile Sales Reports        |                                                                                                                                                                                           |
| Compile Sales Reports         | Code                | Calculates sales summary per location  | Ignore Locations w/o Sales  | Convert Sales Summary to CSV File | ## Compile a Report for Each Location  - This code node calculates totals for each location.  - Please ensure the numbers match EXACTLY with the Square Sales Summary Dashboard.             |
| Convert Sales Summary to CSV File | Convert To File    | Converts JSON report to CSV file       | Compile Sales Reports       | Send Report                 | ## Convert the Square Sales Summary into a CSV File                                                                                                                                       |
| Send Report                  | Gmail               | Sends email with CSV attachment        | Convert Sales Summary to CSV File | None                       | ## Send the Report to the Finance Team / Manager                                                                                                                                          |
| Sticky Note1                 | Sticky Note         | Documentation note                     | None                        | None                        | ## Trigger  - This workflow runs every Monday at 8:00 AM.  - Each week, it pulls the previous week's sales data from Square.                                                             |
| Sticky Note2                 | Sticky Note         | Documentation note                     | None                        | None                        | ## Get Square Locations and Process Each One Separately  - This HTTP node connects to the Square Locations API to fetch all your locations.                                               |
| Sticky Note3                 | Sticky Note         | Documentation note                     | None                        | None                        | ## Get Sales from Square  - This HTTP node retrieves all orders for the given location on the specified date.                                                                              |
| Sticky Note4                 | Sticky Note         | Documentation note                     | None                        | None                        | ## Compile a Report for Each Location  - This code node calculates totals for each location.  - Please ensure the numbers match EXACTLY with the Square Sales Summary Dashboard.             |
| Sticky Note5                 | Sticky Note         | Documentation note                     | None                        | None                        | ## Convert the Square Sales Summary into a CSV File                                                                                                                                       |
| Sticky Note6                 | Sticky Note         | Documentation note                     | None                        | None                        | ## Send the Report to the Finance Team / Manager                                                                                                                                          |
| Sticky Note7                 | Sticky Note         | Documentation note (full workflow description) | None                        | None                        | ## Automatically Send Weekly Sales Reports from Square via Gmail  \n\n## What It Does  \nThis workflow automatically connects to the Square API and generates a **weekly** sales summary report for all your Square locations... |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node:**  
   - Type: Schedule Trigger  
   - Configure to run every week on Monday at 08:00 (set interval weekly, triggerAtDay: 1, triggerAtHour: 8).  

2. **Create Code Node "Get Dates From Last Week":**  
   - Input: Schedule Trigger output.  
   - JavaScript code to calculate dates for the previous 7 days from the trigger timestamp, returning an array of JSON objects with `date` (YYYY-MM-DD).  
   - Run mode: default.  

3. **Create HTTP Request Node "Get Square Locations":**  
   - Method: GET  
   - URL: `https://connect.squareup.com/v2/locations`  
   - Authentication: Create and assign HTTP Header Auth credential named `Square Header Auth` with header: `Authorization: Bearer <your_square_api_key>`.  
   - Headers: Content-Type: application/json  
   - Input: Output from "Get Dates From Last Week".  

4. **Create Split Out Node "Turn Locations Into List":**  
   - Split array in field `locations`.  
   - Include field: `id` (location ID).  
   - Input: Output from "Get Square Locations".  

5. **Create HTTP Request Node "Get Sales from Square":**  
   - Method: POST  
   - URL: `https://connect.squareup.com/v2/orders/search`  
   - Headers: Content-Type: application/json  
   - Authentication: Same Square Header Auth credential as above.  
   - Body Type: JSON  
   - Body: Use expressions to set:  
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
               "start_at": "{{ $('Get Dates From Last Week').item.json.date }}T00:00:00-05:00",
               "end_at": "{{ $('Get Dates From Last Week').item.json.date }}T23:59:59-05:00"
             }
           }
         }
       },
       "limit": 1000,
       "return_entries": false
     }
     ```  
   - Input: Output from "Turn Locations Into List".  

6. **Create If Node "Ignore Locations w/o Sales":**  
   - Condition: Check if `$json.orders` array is not empty (`notEmpty`).  
   - Input: Output from "Get Sales from Square".  
   - Continue only if true.  

7. **Create Code Node "Compile Sales Reports":**  
   - Run once for each item.  
   - JavaScript code to sum sales totals, taxes, discounts, tips, returns, tenders, fees, and rounding adjustments by iterating over all orders for the location.  
   - Access location metadata from "Get Square Locations" and date from "Get Dates From Last Week".  
   - Output aggregated sales report JSON with fields: date, location_id, location_name, gross_sales, total_returns, total_discount, net_sales, total_tax, total_tip, cash_rounding, total_payments_collected, cash, card, gift_card, other, fees, net_total.  
   - Input: Output from "Ignore Locations w/o Sales".  

8. **Create Convert To File Node "Convert Sales Summary to CSV File":**  
   - Convert JSON sales report into CSV format.  
   - Set file name to `sales_report_{{ $('Schedule Trigger').item.json.timestamp }}.csv`.  
   - Set binary property name to `sales_report`.  
   - Input: Output from "Compile Sales Reports".  

9. **Create Gmail Node "Send Report":**  
   - Configure Gmail OAuth2 credentials for the sender email.  
   - Set recipient email in "Send To" field (e.g., `example@user.com`).  
   - Email subject: `Your Square Sales Report for {{ $('Schedule Trigger').item.json['Readable date'].split(',')[0] }}`.  
   - Message body: HTML formatted with greeting and report description.  
   - Attach the CSV file from binary property `sales_report`.  
   - Input: Output from "Convert Sales Summary to CSV File".  

10. **Connect Nodes as Per Flow:**  
    - Schedule Trigger → Get Dates From Last Week → Get Square Locations → Turn Locations Into List → Get Sales from Square → Ignore Locations w/o Sales → Compile Sales Reports → Convert Sales Summary to CSV File → Send Report.  

11. **Activate Workflow:**  
    - Ensure all credentials are properly configured and tested.  
    - Activate workflow for automatic weekly runs.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                       | Context or Link                                                                                                                                                  |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------|
| This workflow builds upon a previous n8n template that automates pulling data from Square API into n8n for processing. Reference workflow link: https://n8n.io/workflows/6358                                                                                     | Background and inspiration for this workflow                                                                                                                   |
| To set up Square API credentials, create an HTTP Header Auth credential in n8n with header name `Authorization` and value `Bearer <your-api-key>`.                                                                                                                 | Credential setup instructions                                                                                                                                  |
| The workflow is designed to match the Square Sales Summary dashboard exactly — verify calculations carefully when modifying.                                                                                                                                     | Accuracy and trustworthiness of data                                                                                                                           |
| Customization suggestions include adding pagination for locations with more than 1,000 orders weekly and using advanced date libraries (e.g., Luxon) for more flexible date range calculations.                                                                   | Enhancement ideas                                                                                                                                               |
| The Gmail node requires OAuth2 credentials and appropriate scopes to send emails with attachments. Make sure Gmail API quotas are considered for high-frequency email sending.                                                                                     | Gmail integration notes                                                                                                                                        |
| The workflow runs every Monday at 8:00 AM and pulls data for the previous Monday through Sunday, enabling weekly reports without manual intervention.                                                                                                              | Scheduling rationale                                                                                                                                           |

---

**Disclaimer:** The content above originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.