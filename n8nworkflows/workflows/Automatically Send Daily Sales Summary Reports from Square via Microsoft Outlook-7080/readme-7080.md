Automatically Send Daily Sales Summary Reports from Square via Microsoft Outlook

https://n8nworkflows.xyz/workflows/automatically-send-daily-sales-summary-reports-from-square-via-microsoft-outlook-7080


# Automatically Send Daily Sales Summary Reports from Square via Microsoft Outlook

### 1. Workflow Overview

This workflow automates the process of sending daily sales summary reports from Square to a designated recipient via Microsoft Outlook email. It is designed for businesses that use Square for point-of-sale and want to receive consolidated, accurate sales performance data without manual intervention. The workflow runs once daily, pulling the previous day's completed sales orders across all Square locations, compiling them into a report that matches Square’s native Sales Summary, converting the report to CSV format, and emailing it to a chosen recipient.

**Logical blocks:**

- **1.1 Scheduled Trigger**: Initiates the workflow daily to process the prior day's sales.
- **1.2 Retrieve Square Locations**: Fetches all active Square locations to process sales data per location.
- **1.3 Retrieve and Filter Sales Orders**: For each location, pulls completed orders for the previous day and filters out locations with no sales.
- **1.4 Compile Sales Reports**: Aggregates sales data including totals, taxes, tips, discounts, returns, and payment methods into a structured summary.
- **1.5 Convert Report to CSV**: Converts the compiled sales data into a CSV file.
- **1.6 Email the Report**: Sends the CSV report via Microsoft Outlook to designated recipients.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
  This node schedules the workflow to run automatically every day, triggering the process to generate the previous day’s sales report.

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**

  - **Schedule Trigger**  
    - Type: `ScheduleTrigger`  
    - Role: Initiates workflow execution on a set schedule.  
    - Configuration: Runs daily at 8:00 AM (local time).  
    - Key Expressions/Variables: The timestamp from the trigger is used to calculate the previous day’s date for sales data querying.  
    - Inputs: None (trigger node).  
    - Outputs: Triggers "Get Square Locations" node.  
    - Edge Cases: Workflow depends on correct time zone handling; misconfiguration could cause wrong dates to be fetched.

---

#### 1.2 Retrieve Square Locations

- **Overview:**  
  Retrieves all Square locations associated with the user’s account, preparing to fetch sales data for each.

- **Nodes Involved:**  
  - Get Square Locations  
  - Turn Locations Into List

- **Node Details:**

  - **Get Square Locations**  
    - Type: `HTTP Request`  
    - Role: Calls Square API endpoint `/v2/locations` to fetch all locations.  
    - Configuration:  
      - Method: GET  
      - URL: `https://connect.squareup.com/v2/locations`  
      - Authentication: Header Auth using Square API Bearer token.  
      - Headers: `Content-Type: application/json`  
    - Input: Trigger from Schedule Trigger node.  
    - Output: JSON response containing all locations.  
    - Edge Cases: Auth errors (invalid token), API rate-limiting, network issues.

  - **Turn Locations Into List**  
    - Type: `SplitOut`  
    - Role: Transforms the array of locations into individual items for iteration.  
    - Configuration: Splits on `locations` field; includes the `id` field for each location.  
    - Input: From Get Square Locations node.  
    - Output: Each location as a separate item (to be used for fetching orders).  
    - Edge Cases: Empty location list, malformed API response.

---

#### 1.3 Retrieve and Filter Sales Orders

- **Overview:**  
  For each Square location, fetches completed sales orders for the previous day. Then filters out any locations with no sales orders, ensuring only active locations are processed further.

- **Nodes Involved:**  
  - Get Sales from Square  
  - Ignore Locations w/o Sales

- **Node Details:**

  - **Get Sales from Square**  
    - Type: `HTTP Request`  
    - Role: Calls Square API `/v2/orders/search` to retrieve completed orders for the specific location and date.  
    - Configuration:  
      - Method: POST  
      - URL: `https://connect.squareup.com/v2/orders/search`  
      - Authentication: Header Auth with Square API token.  
      - Headers: `Content-Type: application/json`  
      - Body (JSON):  
        - Filters orders by location_id from current item.  
        - Filters for orders with state `COMPLETED`.  
        - Date filter for previous day (calculated dynamically using the Schedule Trigger timestamp minus one day).  
        - Limit: 1000 orders per query.  
    - Inputs: Each location item from "Turn Locations Into List".  
    - Outputs: Sales orders JSON data per location.  
    - Edge Cases: API limit exceeded, no orders found, malformed dates.

  - **Ignore Locations w/o Sales**  
    - Type: `If`  
    - Role: Filters out locations that have no orders (empty `orders` array).  
    - Configuration: Checks if `orders` field is not empty.  
    - Inputs: From Get Sales from Square node.  
    - Outputs: Only locations with sales proceed to report compilation.  
    - Edge Cases: Locations with zero orders; ensures no empty reports are generated.

---

#### 1.4 Compile Sales Reports

- **Overview:**  
  Processes the raw orders data per location to compute detailed sales metrics identical to Square’s Sales Summary report, including gross sales, discounts, returns, taxes, tips, payment breakdowns, and fees.

- **Nodes Involved:**  
  - Compile Sales Reports

- **Node Details:**

  - **Compile Sales Reports**  
    - Type: `Code` (JavaScript)  
    - Role: Aggregates and computes sales figures from orders data.  
    - Configuration:  
      - Runs once per location with orders.  
      - Processes the following:  
        - Totals for money, tax, discount, tip, rounding adjustments.  
        - Handles returns and refunds accurately by adjusting totals.  
        - Tallies payment method amounts (cash, card, gift card, others).  
        - Calculates fees including processing fee adjustments.  
      - Outputs a JSON object with computed metrics normalized to dollars (from cents).  
      - Uses expressions to access timestamp and location metadata from previous nodes.  
    - Inputs: Orders data from filtered sales nodes.  
    - Outputs: Structured sales summary JSON per location.  
    - Edge Cases: Orders without expected fields, missing refunds arrays, unexpected tender types, division by zero avoided by safe defaults.

---

#### 1.5 Convert Report to CSV

- **Overview:**  
  Converts the JSON sales summaries for each location into a CSV file for easy sharing and archival.

- **Nodes Involved:**  
  - Convert Sales Summary to CSV File

- **Node Details:**

  - **Convert Sales Summary to CSV File**  
    - Type: `ConvertToFile`  
    - Role: Generates a CSV file from the compiled sales report JSON data.  
    - Configuration:  
      - File name dynamically set as `sales_report_<timestamp>.csv` where timestamp is from Schedule Trigger.  
      - Binary property name set to `sales_report`.  
    - Inputs: Compiled sales report JSON objects.  
    - Outputs: Binary CSV file output.  
    - Edge Cases: Empty data sets, file naming collisions (unlikely due to timestamp).

---

#### 1.6 Email the Report

- **Overview:**  
  Sends the generated CSV sales report via Microsoft Outlook email to the specified recipient.

- **Nodes Involved:**  
  - Send Report

- **Node Details:**

  - **Send Report**  
    - Type: `Microsoft Outlook`  
    - Role: Composes and sends an email with the CSV file attached.  
    - Configuration:  
      - Subject line dynamically includes the readable date from the Schedule Trigger.  
      - HTML body with a greeting and description of the attached report.  
      - Recipient email address set to `user@example.com` (should be customized).  
      - Attachments: The CSV file generated in the previous node.  
      - Uses Microsoft Outlook OAuth2 credentials configured in n8n.  
    - Inputs: Binary CSV file from previous node.  
    - Outputs: Email sent confirmation.  
    - Edge Cases: Credential expiration, email sending failures, attachment size limits.

---

### 3. Summary Table

| Node Name                    | Node Type          | Functional Role                                | Input Node(s)           | Output Node(s)              | Sticky Note                                                                                                                               |
|------------------------------|--------------------|-----------------------------------------------|-------------------------|-----------------------------|-------------------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger              | ScheduleTrigger    | Initiates workflow daily                       | -                       | Get Square Locations         | ## Trigger  \n- This workflow runs every day at 8:00 AM.  \n- Each day, it pulls the previous day's sales data from Square.               |
| Get Square Locations          | HTTP Request       | Fetches all Square locations                   | Schedule Trigger        | Turn Locations Into List     | ## Get Square Locations and Process Each One Separately  \n- This HTTP node connects to the Square Locations API to fetch all your locations.|
| Turn Locations Into List      | SplitOut           | Splits locations array into individual items  | Get Square Locations    | Get Sales from Square        |                                                                                                                                           |
| Get Sales from Square         | HTTP Request       | Retrieves completed orders for each location  | Turn Locations Into List| Ignore Locations w/o Sales   | ## Get Sales from Square  \n- This HTTP node retrieves all orders for the given location on the specified date.                           |
| Ignore Locations w/o Sales    | If                 | Filters out locations with no sales            | Get Sales from Square   | Compile Sales Reports        |                                                                                                                                           |
| Compile Sales Reports         | Code               | Aggregates and computes detailed sales metrics| Ignore Locations w/o Sales| Convert Sales Summary to CSV File| ## Compile a Report for Each Location  \n- This code node calculates totals for each location.  \n- Please ensure the numbers match EXACTLY with the Square Sales Summary Dashboard.|
| Convert Sales Summary to CSV File | ConvertToFile   | Converts JSON sales summary to CSV file        | Compile Sales Reports   | Send Report                  | ## Convert the Square Sales Summary into a CSV File                                                                                       |
| Send Report                  | Microsoft Outlook   | Sends the CSV report via email                  | Convert Sales Summary to CSV File | -                         | ## Send the Report to the Finance Team / Manager                                                                                        |
| Sticky Note                  | StickyNote          | Documentation and usage instructions            | -                       | -                           | Contains detailed workflow description, prerequisites, setup, and customization options                                                  |
| Sticky Note1                 | StickyNote          | Trigger description                             | -                       | -                           | ## Trigger  \n- This workflow runs every day at 8:00 AM.  \n- Each day, it pulls the previous day's sales data from Square.               |
| Sticky Note2                 | StickyNote          | Square locations retrieval explanation          | -                       | -                           | ## Get Square Locations and Process Each One Separately  \n- This HTTP node connects to the Square Locations API to fetch all your locations.|
| Sticky Note3                 | StickyNote          | Sales retrieval explanation                      | -                       | -                           | ## Get Sales from Square  \n- This HTTP node retrieves all orders for the given location on the specified date.                           |
| Sticky Note4                 | StickyNote          | Sales aggregation explanation                    | -                       | -                           | ## Compile a Report for Each Location  \n- This code node calculates totals for each location.  \n- Please ensure the numbers match EXACTLY with the Square Sales Summary Dashboard.|
| Sticky Note5                 | StickyNote          | CSV conversion explanation                       | -                       | -                           | ## Convert the Square Sales Summary into a CSV File                                                                                       |
| Sticky Note6                 | StickyNote          | Email sending explanation                         | -                       | -                           | ## Send the Report to the Finance Team / Manager                                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Scheduled Trigger Node**  
   - Node Type: `ScheduleTrigger`  
   - Set interval to run **daily at 8:00 AM**.  
   - This node starts the workflow.

2. **Add HTTP Request Node "Get Square Locations"**  
   - Node Type: `HTTP Request`  
   - Method: `GET`  
   - URL: `https://connect.squareup.com/v2/locations`  
   - Authentication: Use **Header Auth Credential** with your Square Bearer token.  
   - Headers: Add header `Content-Type: application/json`.  
   - Connect input from Schedule Trigger.

3. **Add SplitOut Node "Turn Locations Into List"**  
   - Node Type: `SplitOut`  
   - Field to split out: `locations`  
   - Fields to include: `id`  
   - Connect input from "Get Square Locations" node.

4. **Add HTTP Request Node "Get Sales from Square"**  
   - Node Type: `HTTP Request`  
   - Method: `POST`  
   - URL: `https://connect.squareup.com/v2/orders/search`  
   - Authentication: Use the same **Header Auth Credential** as before.  
   - Headers: `Content-Type: application/json`  
   - Body (JSON): Use expressions to dynamically set:  
     ```json
     {
       "location_ids": ["{{ $json.locations.id }}"],
       "query": {
         "filter": {
           "state_filter": { "states": ["COMPLETED"] },
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
   - Send Body as JSON.  
   - Connect input from "Turn Locations Into List" node.

5. **Add If Node "Ignore Locations w/o Sales"**  
   - Node Type: `If`  
   - Condition: Check if `orders` array is not empty:  
     - Expression: `{{$json.orders}}`  
     - Operator: Array not empty  
   - Connect input from "Get Sales from Square" node.  
   - Only continue with "true" output.

6. **Add Code Node "Compile Sales Reports"**  
   - Node Type: `Code`  
   - Mode: Run once per item (for each location with sales).  
   - Paste the provided JavaScript code that aggregates sales, taxes, discounts, tips, returns, tenders, fees, and computes gross/net sales.  
   - Connect input from "Ignore Locations w/o Sales" node (true branch).

7. **Add ConvertToFile Node "Convert Sales Summary to CSV File"**  
   - Node Type: `ConvertToFile`  
   - Options:  
     - File name: `sales_report_{{ $('Schedule Trigger').item.json.timestamp }}.csv`  
     - Binary property name: `sales_report`  
   - Connect input from "Compile Sales Reports" node.

8. **Add Microsoft Outlook Node "Send Report"**  
   - Node Type: `Microsoft Outlook`  
   - Configure Microsoft Outlook OAuth2 credentials in n8n credentials.  
   - Parameters:  
     - Subject: `Your Square Sales Report for {{ $('Schedule Trigger').item.json['Readable date'].split(',')[0] }}`  
     - Body Content (HTML):  
       ```html
       <p>Hello User,</p>
       <p>Please see the attached report containing yesterday's sales!</p>
       <p>Best,<br> An Efficient Person</p>
       ```  
     - To Recipients: `user@example.com` (replace with actual recipient)  
     - Attachments: Attach the binary `sales_report` file.  
   - Connect input from "Convert Sales Summary to CSV File" node.

9. **Activate the workflow** to run daily and send reports automatically.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                    | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow is designed to match the Square Dashboard Sales Summary report exactly, ensuring accuracy in financial reporting. The code node handles complex calculations including refunds and payment method breakdowns.                                         | Main workflow functionality explanation                                                             |
| For Square API credentials, create a **Header Auth** credential in n8n with the header name `Authorization` and the value `Bearer <your-access-token>`.                                                                                                       | Square API credential setup                                                                          |
| Microsoft Outlook credentials require OAuth2 configuration in n8n to send emails securely.                                                                                                                                                                      | Microsoft Outlook integration                                                                        |
| The workflow runs daily at 8:00 AM by default but can be adjusted in the Schedule Trigger node to suit business needs.                                                                                                                                          | Customization note                                                                                   |
| Pagination is not implemented; if you have more than 1,000 orders per location per day, additional pagination logic is required.                                                                                                                                 | Potential enhancement                                                                                |
| Previous related workflow example for pulling Square data into n8n: https://n8n.io/workflows/6358                                                                                                                                                              | Reference to prior workflow                                                                          |

---

**Disclaimer:**  
The provided text and workflow are generated exclusively by n8n automation respecting all relevant content policies and handle only legal and public data. No illegal, offensive, or protected content is included.