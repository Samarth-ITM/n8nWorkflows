Automatically Send Weekly Sales Reports from Square via Outlook

https://n8nworkflows.xyz/workflows/automatically-send-weekly-sales-reports-from-square-via-outlook-7089


# Automatically Send Weekly Sales Reports from Square via Outlook

### 1. Workflow Overview

This workflow automates the process of sending weekly sales reports from Square to a designated recipient via Outlook email. It is designed to run every Monday at 8:00 AM, pulling the previous week's sales data for all Square locations, compiling a report matching the Square Sales Summary dashboard, converting the report into a CSV file, and emailing it to a specified address.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger and Date Calculation:** Initiates the workflow weekly and computes the list of dates representing the previous week.
- **1.2 Fetch Square Locations:** Retrieves all Square account locations to process sales data location-wise.
- **1.3 Retrieve and Filter Sales Data:** For each location, fetches completed orders for the previous week and filters out locations without sales.
- **1.4 Sales Data Aggregation:** Processes orders to calculate detailed sales metrics exactly replicating Square’s Sales Summary numbers.
- **1.5 Report Generation and Conversion:** Converts the compiled sales summary data into a CSV file.
- **1.6 Email Dispatch:** Sends the CSV sales report via Outlook to the intended recipient.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger and Date Calculation

- **Overview:** This block triggers the workflow every Monday at 8:00 AM and generates the list of dates covering the entire previous week (Monday to Sunday).
- **Nodes Involved:** `Schedule Trigger`, `Get Dates From Last Week`
  
**Node Details:**

- **Schedule Trigger**
  - Type: Schedule Trigger
  - Role: Initiates the workflow weekly every Monday at 8:00 AM.
  - Configuration: Trigger interval set to weekly on day 1 (Monday) at hour 8.
  - Inputs: None (trigger node)
  - Outputs: Emits timestamp for the trigger event.
  - Potential Failures: Misconfigured schedule could cause non-execution.
  - Version: 1.2

- **Get Dates From Last Week**
  - Type: Code (JavaScript)
  - Role: Computes each date of the previous week based on the trigger timestamp.
  - Configuration: Runs JavaScript that subtracts 7 days from the trigger timestamp to create an array of date strings in "YYYY-MM-DD" format.
  - Key Expression: Uses `$input.first().json.timestamp` for the base date.
  - Inputs: Receives trigger event data.
  - Outputs: Emits an array of JSON objects each with one date representing each day of the previous week.
  - Edge Cases: Changes in timezone might affect date correctness; assumes trigger runs Monday morning.
  - Version: 2

#### 2.2 Fetch Square Locations

- **Overview:** Fetches all Square locations associated with the account to process sales data for each location individually.
- **Nodes Involved:** `Get Square Locations`, `Turn Locations Into List`

**Node Details:**

- **Get Square Locations**
  - Type: HTTP Request
  - Role: Calls Square API endpoint `/v2/locations` to retrieve locations.
  - Configuration: GET request with header authentication using Square API token.
  - Headers: Content-Type set to `application/json`.
  - Credentials: Uses Header Auth credential with Square Bearer token.
  - Inputs: Receives dates from the previous node.
  - Outputs: Returns JSON response containing all locations.
  - Edge Cases: API failures, expired token, network errors.
  - Version: 4.2

- **Turn Locations Into List**
  - Type: Split Out
  - Role: Converts the array of location objects into individual items, each representing one location (with `id` field).
  - Configuration: Field to split out is `locations`, includes the `id` field only.
  - Inputs: Receives locations array.
  - Outputs: Emits one item per location for downstream processing.
  - Edge Cases: Empty location list results in no further processing.
  - Version: 1

#### 2.3 Retrieve and Filter Sales Data

- **Overview:** For each location, retrieves all completed orders from the previous week and filters out any locations with no sales orders.
- **Nodes Involved:** `Get Sales from Square`, `Ignore Locations w/o Sales`

**Node Details:**

- **Get Sales from Square**
  - Type: HTTP Request
  - Role: Calls Square API endpoint `/v2/orders/search` to fetch completed orders for each location on each day of the previous week.
  - Configuration:
    - Method: POST
    - Body: JSON with location ID, filter for order state "COMPLETED", and date_time filter using the dates from "Get Dates From Last Week".
    - Limit: 1000 orders per request.
    - Headers: Content-Type `application/json`.
    - Authentication: Header Auth with Square token.
  - Key Expressions:
    - Uses `{{ $json.locations.id }}` for location IDs.
    - Uses date from `$('Get Dates From Last Week').item.json.date` with timezone offset -05:00.
  - Inputs: Location IDs from previous node.
  - Outputs: JSON containing orders for each location.
  - Edge Cases: Pagination not handled if orders exceed 1000; API rate limits; authentication failure.
  - Version: 4.2

- **Ignore Locations w/o Sales**
  - Type: If
  - Role: Filters out locations that have no orders (empty `orders` array).
  - Configuration: Condition checks if `orders` array is not empty.
  - Inputs: Receives orders data.
  - Outputs: Passes only locations with sales forward.
  - Edge Cases: Locations with zero sales are skipped, ensuring no empty reports.
  - Version: 2.2

#### 2.4 Sales Data Aggregation

- **Overview:** Aggregates sales data for each location, computing detailed metrics such as gross sales, taxes, discounts, tips, returns, payment methods, and fees, ensuring the results match Square’s Sales Summary exactly.
- **Nodes Involved:** `Compile Sales Reports`

**Node Details:**

- **Compile Sales Reports**
  - Type: Code (JavaScript)
  - Role: Processes the orders JSON array to calculate all relevant sales totals.
  - Configuration:
    - Runs once per item (each location’s orders).
    - Extracts location metadata and iterates over each sale.
    - Calculates totals for money amounts, taxes, discounts, tips, cash rounding, returns, tenders by type (cash, card, gift card, other), and fees.
    - Handles refunds by identifying original sales and adjusting amounts accordingly.
    - Computes net and gross sales, total payments collected, and net total.
    - Outputs a JSON object summarizing the sales per location.
  - Key Expressions: Accesses nodes with `$`, uses optional chaining for safe access.
  - Inputs: Orders data filtered for locations with sales.
  - Outputs: JSON summary for each location.
  - Edge Cases: Missing or malformed data may cause calculation errors; ensures no division by zero.
  - Version: 2

#### 2.5 Report Generation and Conversion

- **Overview:** Converts the compiled sales summary JSON data into a CSV file for easy sharing.
- **Nodes Involved:** `Convert Sales Summary to CSV File`

**Node Details:**

- **Convert Sales Summary to CSV File**
  - Type: Convert To File
  - Role: Converts JSON sales summary data into a CSV file.
  - Configuration:
    - File name dynamically generated as `sales_report_<timestamp>.csv` where timestamp is from the schedule trigger.
    - Binary property name for the file is `sales_report`.
  - Inputs: Receives sales summary JSON.
  - Outputs: Binary CSV file attached to the workflow.
  - Edge Cases: Large data sets may impact performance.
  - Version: 1.1

#### 2.6 Email Dispatch

- **Overview:** Sends the CSV sales report via Microsoft Outlook to a specified recipient.
- **Nodes Involved:** `Send Report`

**Node Details:**

- **Send Report**
  - Type: Microsoft Outlook
  - Role: Sends an email with the sales report CSV file attached.
  - Configuration:
    - Subject line includes the date from the schedule trigger in human-readable format.
    - Email body is HTML formatted with a polite message.
    - Recipient email is set to `user@example.com` (should be customized).
    - Attachment is the CSV file generated in the previous node.
  - Inputs: Receives binary CSV file.
  - Outputs: None (end node).
  - Credentials: Requires configured Microsoft Outlook OAuth2 credential.
  - Edge Cases: Email send failures due to authentication, network issues, or invalid recipient address.
  - Version: 2

---

### 3. Summary Table

| Node Name                     | Node Type             | Functional Role                     | Input Node(s)               | Output Node(s)                 | Sticky Note                                                                                                           |
|-------------------------------|-----------------------|-----------------------------------|-----------------------------|-------------------------------|-----------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger               | Schedule Trigger      | Weekly trigger at Monday 8:00 AM  | None                        | Get Dates From Last Week       | This workflow runs every Monday at 8:00 AM. Each week, it pulls the previous week's sales data from Square.           |
| Get Dates From Last Week       | Code                  | Generates dates for previous week | Schedule Trigger            | Get Square Locations           |                                                                                                                       |
| Get Square Locations           | HTTP Request          | Retrieves all Square locations    | Get Dates From Last Week    | Turn Locations Into List       | This HTTP node connects to the Square Locations API to fetch all your locations.                                      |
| Turn Locations Into List       | Split Out             | Splits locations array into items | Get Square Locations        | Get Sales from Square          |                                                                                                                       |
| Get Sales from Square          | HTTP Request          | Retrieves orders for each location | Turn Locations Into List    | Ignore Locations w/o Sales     | This HTTP node retrieves all orders for the given location on the specified date.                                     |
| Ignore Locations w/o Sales     | If                    | Filters out locations with no sales | Get Sales from Square       | Compile Sales Reports          |                                                                                                                       |
| Compile Sales Reports          | Code                  | Aggregates and calculates sales data | Ignore Locations w/o Sales  | Convert Sales Summary to CSV File | This code node calculates totals for each location. Please ensure the numbers match EXACTLY with the Square Sales Summary Dashboard. |
| Convert Sales Summary to CSV File | Convert To File       | Converts JSON summary to CSV file  | Compile Sales Reports       | Send Report                   | Convert the Square Sales Summary into a CSV File                                                                     |
| Send Report                   | Microsoft Outlook     | Sends the CSV report via email   | Convert Sales Summary to CSV File | None                        | Send the Report to the Finance Team / Manager                                                                         |
| Sticky Note1                  | Sticky Note           | Documentation                    | None                        | None                         | This workflow runs every Monday at 8:00 AM. Each week, it pulls the previous week's sales data from Square.           |
| Sticky Note2                  | Sticky Note           | Documentation                    | None                        | None                         | This HTTP node connects to the Square Locations API to fetch all your locations.                                      |
| Sticky Note3                  | Sticky Note           | Documentation                    | None                        | None                         | This HTTP node retrieves all orders for the given location on the specified date.                                     |
| Sticky Note4                  | Sticky Note           | Documentation                    | None                        | None                         | This code node calculates totals for each location. Please ensure the numbers match EXACTLY with the Square Sales Summary Dashboard. |
| Sticky Note5                  | Sticky Note           | Documentation                    | None                        | None                         | Convert the Square Sales Summary into a CSV File                                                                     |
| Sticky Note6                  | Sticky Note           | Documentation                    | None                        | None                         | Send the Report to the Finance Team / Manager                                                                         |
| Sticky Note7                  | Sticky Note           | Documentation overview           | None                        | None                         | See section 5 below for full workflow explanation and setup instructions.                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**
   - Type: Schedule Trigger
   - Parameters: Set trigger to run weekly on Monday at 8:00 AM.
   - No credentials required.
   - Connect output to "Get Dates From Last Week".

2. **Create Get Dates From Last Week Node**
   - Type: Code
   - Parameters: JavaScript code to generate previous week’s dates (7 days before the trigger date).
   - Input: Connected from Schedule Trigger.
   - Output: Connect to "Get Square Locations".

3. **Create Get Square Locations Node**
   - Type: HTTP Request
   - Method: GET
   - URL: `https://connect.squareup.com/v2/locations`
   - Headers: `Content-Type: application/json`
   - Authentication: Header Auth using your Square API token (Bearer token).
   - Input: Connected from "Get Dates From Last Week".
   - Output: Connect to "Turn Locations Into List".

4. **Create Turn Locations Into List Node**
   - Type: Split Out
   - Parameters: Split out the `locations` field, include only the `id` field.
   - Input: Connected from "Get Square Locations".
   - Output: Connect to "Get Sales from Square".

5. **Create Get Sales from Square Node**
   - Type: HTTP Request
   - Method: POST
   - URL: `https://connect.squareup.com/v2/orders/search`
   - Headers: `Content-Type: application/json`
   - Authentication: Header Auth using Square API token.
   - Body: JSON with parameters:
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
   - Input: Connected from "Turn Locations Into List".
   - Output: Connect to "Ignore Locations w/o Sales".

6. **Create Ignore Locations w/o Sales Node**
   - Type: If
   - Parameters: Condition to check if `orders` array is not empty.
   - Input: Connected from "Get Sales from Square".
   - Output: Connect true branch to "Compile Sales Reports".

7. **Create Compile Sales Reports Node**
   - Type: Code
   - Parameters: JavaScript code to aggregate sales data per location computing totals (gross sales, net sales, taxes, tips, discounts, returns, tenders, fees).
   - Runs once per item (per location).
   - Input: Connected from "Ignore Locations w/o Sales".
   - Output: Connect to "Convert Sales Summary to CSV File".

8. **Create Convert Sales Summary to CSV File Node**
   - Type: Convert To File
   - Parameters:
     - File name: `sales_report_{{ $('Schedule Trigger').item.json.timestamp }}.csv`
     - Binary Property Name: `sales_report`
   - Input: Connected from "Compile Sales Reports".
   - Output: Connect to "Send Report".

9. **Create Send Report Node**
   - Type: Microsoft Outlook
   - Parameters:
     - Subject: `Your Square Sales Report for {{ $('Schedule Trigger').item.json['Readable date'].split(',')[0] }}`
     - Body Content: HTML formatted message (greeting, explanation, signature).
     - To Recipients: Set to desired recipient email address.
     - Attachments: Attach CSV file from binary property `sales_report`.
   - Credentials: Microsoft Outlook OAuth2 credential configured.
   - Input: Connected from "Convert Sales Summary to CSV File".
   - Output: End node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                 | Context or Link                                                                                                  |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| This workflow automatically connects to the Square API and generates a weekly sales summary report for all Square locations matching the Square Dashboard > Reports > Sales Summary figures.                                                                                                 | Full workflow description in Sticky Note7 node.                                                                |
| Requires Square API credential configured as Header Auth with Authorization header set to `Bearer <your-access-token>`.                                                                                                                                                                      | Credential setup instructions in Sticky Note7.                                                                  |
| Requires Microsoft Outlook OAuth2 credential for sending emails.                                                                                                                                                                                                                              | Credential required for Send Report node.                                                                        |
| Designed to run every Monday at 8:00 AM to pull data for the previous week (Monday-Sunday). Adjust schedule as needed.                                                                                                                                                                        | Schedule Trigger node configuration.                                                                             |
| If locations have more than 1000 orders per week, pagination needs to be added to the "Get Sales from Square" HTTP Request node.                                                                                                                                                             | Customization note in Sticky Note7.                                                                               |
| The workflow calculates sales data precisely to match the Square Sales Summary dashboard; verify totals after initial setup.                                                                                                                                                                 | Important note in Compile Sales Reports node and Sticky Note4.                                                  |
| Example use cases include sending data to management, external agents, or bookkeepers for QuickBooks entry.                                                                                                                                                                                   | Use case descriptions in Sticky Note7.                                                                            |
| Previous related workflow for pulling Square data into n8n: https://n8n.io/workflows/6358                                                                                                                                                                                                   | Reference link in Sticky Note7.                                                                                   |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.