Automatically Send Monthly Sales Reports from Square via Outlook

https://n8nworkflows.xyz/workflows/automatically-send-monthly-sales-reports-from-square-via-outlook-7091


# Automatically Send Monthly Sales Reports from Square via Outlook

### 1. Workflow Overview

This workflow automates the generation and delivery of monthly sales reports from Square to a specified email via Microsoft Outlook. It is designed for businesses using Square to manage multiple locations, providing a consolidated sales summary report for the entire previous calendar month.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger and Date Calculation:** Initiates the workflow monthly and computes all dates for the previous month.
- **1.2 Location Retrieval and Processing:** Fetches all Square locations and processes each location individually.
- **1.3 Sales Data Retrieval and Filtering:** For each location, retrieves completed sales orders from the previous month; filters out locations without sales.
- **1.4 Sales Data Compilation:** Aggregates and calculates the sales summary metrics to match Square's Sales Summary Dashboard.
- **1.5 CSV Conversion and Email Sending:** Converts the sales summary into a CSV file and sends it via Outlook email to the designated recipient.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger and Date Calculation

- **Overview:**  
  This block triggers the workflow automatically on the first day of every month at 8:00 AM and generates a list of all dates of the previous calendar month to be used for filtering sales data.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Get Dates From Last Month

- **Node Details:**

  - **Schedule Trigger**  
    - *Type & Role:* Schedule trigger node; initiates the workflow monthly.  
    - *Configuration:* Set to trigger at 8:00 AM on the first day of each month.  
    - *Key Expressions:* None (fixed schedule).  
    - *Connections:* Output to "Get Dates From Last Month".  
    - *Failure Modes:* Misconfigured schedule or disabled workflow may prevent execution.

  - **Get Dates From Last Month**  
    - *Type & Role:* Code node; calculates all dates for the previous month to allow batch processing per day.  
    - *Configuration:* JavaScript code extracts year and month from the trigger timestamp, computes first and last days of previous month, and outputs an array of dates formatted as ISO strings.  
    - *Key Expressions:* Uses `$input.first().json.timestamp` for current date/time.  
    - *Input:* From Schedule Trigger.  
    - *Output:* Array of JSON objects with a `date` property for each day in the previous month.  
    - *Failure Modes:* Date parsing errors if input timestamp is malformed; timezone assumptions (-05:00) may need adjustment for other locales.

---

#### 2.2 Location Retrieval and Processing

- **Overview:**  
  Retrieves all Square locations via the Square API and splits the location array to process each location separately in subsequent nodes.

- **Nodes Involved:**  
  - Get Square Locations  
  - Turn Locations Into List

- **Node Details:**

  - **Get Square Locations**  
    - *Type & Role:* HTTP Request node; calls Square API endpoint to fetch all locations associated with the account.  
    - *Configuration:* GET request to `https://connect.squareup.com/v2/locations` with `Content-Type: application/json`. Uses Header Auth credential for Square API authorization (`Authorization: Bearer <token>`).  
    - *Key Expressions:* None; static URL and headers.  
    - *Input:* From "Get Dates From Last Month".  
    - *Output:* JSON response with a `locations` array.  
    - *Failure Modes:* Authentication errors (invalid token), API rate limits, network timeouts.

  - **Turn Locations Into List**  
    - *Type & Role:* SplitOut node; splits the `locations` array into individual items for sequential processing.  
    - *Configuration:* Splits the `locations` field, including only the `id` field for downstream use.  
    - *Input:* From "Get Square Locations".  
    - *Output:* One item per location with `id` field.  
    - *Failure Modes:* Empty or missing `locations` array causes no downstream processing.

---

#### 2.3 Sales Data Retrieval and Filtering

- **Overview:**  
  For each location, this block queries Square orders completed in the previous month and filters out locations with no sales.

- **Nodes Involved:**  
  - Get Sales from Square  
  - Ignore Locations w/o Sales (If node)

- **Node Details:**

  - **Get Sales from Square**  
    - *Type & Role:* HTTP Request node; POST request to Square Orders Search API to fetch completed orders for the location and date.  
    - *Configuration:*  
      - URL: `https://connect.squareup.com/v2/orders/search`  
      - Method: POST  
      - Body: JSON specifying the location ID (dynamic from current item), filter for `COMPLETED` state orders, and date range set to the current date from "Get Dates From Last Month" (start and end at 00:00:00 to 23:59:59 with -05:00 timezone offset).  
      - Limit: 1000 orders per request.  
      - Authentication: Header Auth with Square token.  
    - *Key Expressions:* Uses dynamic expressions to insert `location.id` and date from previous node.  
    - *Input:* From "Turn Locations Into List".  
    - *Output:* JSON with an `orders` array.  
    - *Failure Modes:* Authentication errors, API rate limiting, date format errors, pagination not handled (max 1000 orders).

  - **Ignore Locations w/o Sales**  
    - *Type & Role:* If node; filters out locations where `orders` array is empty.  
    - *Configuration:* Condition checks if `orders` field is non-empty array.  
    - *Input:* From "Get Sales from Square".  
    - *Output:* Passes only locations with sales to next node.  
    - *Failure Modes:* Incorrect field references or data structure changes may cause false negatives.

---

#### 2.4 Sales Data Compilation

- **Overview:**  
  Aggregates the retrieved sales data into a summarized report matching Square’s Sales Summary dashboard, including calculations of totals, taxes, discounts, tips, returns, payment types, and fees.

- **Nodes Involved:**  
  - Compile Sales Reports (Code node)

- **Node Details:**

  - **Compile Sales Reports**  
    - *Type & Role:* Code node; processes the `orders` array to compute various sales metrics.  
    - *Configuration:*  
      - Runs once for each item (location with sales).  
      - JavaScript code loops through orders, adds and subtracts amounts for sales, taxes, discounts, tips, returns, and handles tender types (CARD, CASH, GIFT_CARD, OTHER).  
      - Adjusts for rounding and processing fees.  
      - Returns a JSON object with computed values normalized to decimal currency (dividing by 100).  
      - Extracts location name from the earlier fetched locations list.  
    - *Key Expressions:* Uses `$json.orders`, `$json.orders[0].location_id`, and cross-references with `Get Square Locations` node to find location name.  
    - *Input:* From "Ignore Locations w/o Sales".  
    - *Output:* JSON object with summarized sales report for one location.  
    - *Failure Modes:* Missing or malformed sales data, unexpected nulls, or deviations in API data structure may cause calculation errors; ensure data matches expected Square API formats.

---

#### 2.5 CSV Conversion and Email Sending

- **Overview:**  
  Converts the compiled sales reports into a CSV file and sends it as an email attachment through Microsoft Outlook.

- **Nodes Involved:**  
  - Convert Sales Summary to CSV File  
  - Send Report

- **Node Details:**

  - **Convert Sales Summary to CSV File**  
    - *Type & Role:* ConvertToFile node; converts JSON sales data into a CSV file.  
    - *Configuration:*  
      - Filename dynamically includes the timestamp from the Schedule Trigger (e.g., `sales_report_<timestamp>.csv`).  
      - Output binary property named `sales_report`.  
    - *Input:* From "Compile Sales Reports".  
    - *Output:* Binary CSV file for attachment.  
    - *Failure Modes:* Data inconsistencies could cause CSV formatting issues.

  - **Send Report**  
    - *Type & Role:* Microsoft Outlook node; sends an email with the CSV file attached.  
    - *Configuration:*  
      - Subject: "Your Last Month's Square Sales Report"  
      - Body: HTML content with greeting and message  
      - To Recipients: Configured to a specified email (example: `user@example.com`)  
      - Attachments: Uses binary property `sales_report` from previous node  
      - Requires valid Microsoft Outlook OAuth2 credentials.  
    - *Input:* From "Convert Sales Summary to CSV File".  
    - *Output:* None (end node).  
    - *Failure Modes:* Authentication errors, invalid email addresses, attachment size limits, network errors.

---

### 3. Summary Table

| Node Name                     | Node Type                | Functional Role                      | Input Node(s)              | Output Node(s)                  | Sticky Note                                                                                     |
|-------------------------------|--------------------------|------------------------------------|---------------------------|--------------------------------|------------------------------------------------------------------------------------------------|
| Schedule Trigger              | Schedule Trigger          | Monthly workflow initiation        |                           | Get Dates From Last Month       | ## Trigger  \n- This workflow runs on the first day of every Month.  \n- Each month, it pulls the previous month's sales data from Square. |
| Get Dates From Last Month     | Code                     | Calculate all dates for previous month | Schedule Trigger          | Get Square Locations            |                                                                                                |
| Get Square Locations          | HTTP Request             | Fetch all Square account locations | Get Dates From Last Month  | Turn Locations Into List        | ## Get Square Locations and Process Each One Separately  \n- This HTTP node connects to the Square Locations API to fetch all your locations. |
| Turn Locations Into List      | SplitOut                 | Split array of locations into items | Get Square Locations       | Get Sales from Square           |                                                                                                |
| Get Sales from Square         | HTTP Request             | Retrieve completed sales orders per location per date | Turn Locations Into List | Ignore Locations w/o Sales      | ## Get Sales from Square  \n- This HTTP node retrieves all orders for the given location on the specified date. |
| Ignore Locations w/o Sales    | If                       | Filter out locations with no sales | Get Sales from Square      | Compile Sales Reports           |                                                                                                |
| Compile Sales Reports         | Code                     | Aggregate and summarize sales data | Ignore Locations w/o Sales | Convert Sales Summary to CSV File | ## Compile a Report for Each Location  \n- This code node calculates totals for each location.  \n- Please ensure the numbers match EXACTLY with the Square Sales Summary Dashboard. |
| Convert Sales Summary to CSV File | ConvertToFile          | Convert JSON sales summary to CSV file | Compile Sales Reports     | Send Report                    | ## Convert the Square Sales Summary into a CSV File \n |
| Send Report                  | Microsoft Outlook          | Email the CSV report to recipient  | Convert Sales Summary to CSV File |                            | ## Send the Report to the Finance Team / Manager                                               |
| Sticky Note1                 | Sticky Note               | Documentation                      |                           |                                | ## Trigger  \n- This workflow runs on the first day of every Month.  \n- Each month, it pulls the previous month's sales data from Square.      |
| Sticky Note2                 | Sticky Note               | Documentation                      |                           |                                | ## Get Square Locations and Process Each One Separately  \n- This HTTP node connects to the Square Locations API to fetch all your locations.    |
| Sticky Note3                 | Sticky Note               | Documentation                      |                           |                                | ## Get Sales from Square  \n- This HTTP node retrieves all orders for the given location on the specified date.                                     |
| Sticky Note4                 | Sticky Note               | Documentation                      |                           |                                | ## Compile a Report for Each Location  \n- This code node calculates totals for each location.  \n- Please ensure the numbers match EXACTLY with the Square Sales Summary Dashboard. |
| Sticky Note5                 | Sticky Note               | Documentation                      |                           |                                | ## Convert the Square Sales Summary into a CSV File \n                                                                                                  |
| Sticky Note6                 | Sticky Note               | Documentation                      |                           |                                | ## Send the Report to the Finance Team / Manager                                                                                                        |
| Sticky Note                  | Sticky Note               | Full workflow documentation       |                           |                                | ## Automatically Send Monthly Sales Reports from Square via Outlook\n\n## What It Does  \nThis workflow automatically connects to the Square API and generates a **monthly** sales summary report for all your Square locations. The report matches the figures displayed in **Square Dashboard > Reports > Sales Summary**.\n\nIt's designed to run monthly and pull the **previous month’s** sales into a CSV file, which is then sent to a manager/finance team for analysis.\n\nThis workflow builds on my previous template, which allows users to automatically pull data from the Square API into n8n for processing. (See here: https://n8n.io/workflows/6358)\n\n## Prerequisites  \nTo use this workflow, you'll need:\n- A Square API credential (configured as a Header Auth credential)\n- A Microsoft Outlook credential\n\n## How to Set Up Square Credentials:  \n- Go to **Credentials > Create New**  \n- Choose **Header Auth**  \n- Set the **Name** to `Authorization`  \n- Set the **Value** to your Square Access Token (e.g., `Bearer <your-api-key>`)\n\n## How It Works  \n1. **Trigger:** The workflow runs on the **1st of every month at 8:00 AM**  \n2. **Fetch Locations:** An HTTP request retrieves all Square locations linked to your account  \n3. **Fetch Orders:** For each location, an HTTP request pulls completed orders for the **previous calendar month**  \n4. **Filter Empty Locations:** Locations with no sales are ignored  \n5. **Aggregate Sales Data:** A Code node processes the order data and produces a summary identical to Square’s built-in Sales Summary report  \n6. **Create CSV File:** A CSV file is created containing the relevant data  \n7. **Send Email:** An email is sent using Microsoft Outlook to the chosen third party  \n\n## Example Use Cases  \n- Automatically send monthly Square sales data to management for forecasting and planning  \n- Automatically send data to an external third party, such as a landlord or agent, who is paid via commission  \n- Automatically send data to a bookkeeper for entry into QuickBooks  \n\n## How to Use  \n- Configure both HTTP Request nodes to use your Square API credential  \n- Set the workflow to **Active** so it runs automatically  \n- Enter the email address of the person you want to send the report to and update the message body  \n- If you want to remove the n8n attribution, you can do so in the last node  \n\n## Customization Options  \n- Add pagination to handle locations with more than 1,000 orders per month\n- Adjust the date filters in the HTTP node to cover the full calendar month (e.g., use Luxon or JavaScript to calculate `start_date` and `end_date`)\n\n## Why It's Useful  \nThis workflow saves time, reduces manual report pulling from Square, and enables smarter automation around sales data — whether for operations, finance, or performance monitoring. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Schedule Trigger node:**  
   - Name: `Schedule Trigger`  
   - Set to trigger monthly on the 1st day at 08:00 AM.

3. **Add a Code node to calculate previous month dates:**  
   - Name: `Get Dates From Last Month`  
   - Input: Connect from `Schedule Trigger`  
   - Paste the JavaScript code that extracts the year/month from the trigger timestamp and outputs all dates of the previous month as JSON items with a `date` field.

4. **Add an HTTP Request node to get Square locations:**  
   - Name: `Get Square Locations`  
   - Method: GET  
   - URL: `https://connect.squareup.com/v2/locations`  
   - Headers: `Content-Type: application/json`  
   - Authentication: Create and select a Header Auth credential named `Square Header Auth` with:  
     - Name: `Authorization`  
     - Value: `Bearer <your-square-access-token>`  
   - Connect input from `Get Dates From Last Month`.

5. **Add a SplitOut node to process each location separately:**  
   - Name: `Turn Locations Into List`  
   - Field to Split Out: `locations`  
   - Fields to Include: `id` only  
   - Connect input from `Get Square Locations`.

6. **Add an HTTP Request node to get sales orders for each location and date:**  
   - Name: `Get Sales from Square`  
   - Method: POST  
   - URL: `https://connect.squareup.com/v2/orders/search`  
   - Headers: `Content-Type: application/json`  
   - Authentication: Use the same `Square Header Auth` credential as before  
   - Body Type: JSON  
   - Body Content: Use an expression to dynamically insert location id and date from previous nodes, filtering for orders with state `COMPLETED` and date range from 00:00:00 to 23:59:59 of the current date item, including a limit of 1000.  
   - Connect input from `Turn Locations Into List`.

7. **Add an If node to filter locations with no sales:**  
   - Name: `Ignore Locations w/o Sales`  
   - Condition: Check if `orders` field is a non-empty array (operator: array not empty).  
   - Connect input from `Get Sales from Square`.

8. **Add a Code node to compile sales reports:**  
   - Name: `Compile Sales Reports`  
   - Mode: Run once per item  
   - Paste the JavaScript code that aggregates sales, taxes, discounts, tips, returns, tender types, and fees.  
   - Connect input from the "true" output of `Ignore Locations w/o Sales`.

9. **Add a ConvertToFile node to create CSV:**  
   - Name: `Convert Sales Summary to CSV File`  
   - Configure output filename as `sales_report_{{ $('Schedule Trigger').item.json.timestamp }}.csv`.  
   - Binary Property Name: `sales_report`  
   - Connect input from `Compile Sales Reports`.

10. **Add a Microsoft Outlook node to send the report:**  
    - Name: `Send Report`  
    - Subject: `Your Last Month's Square Sales Report`  
    - Body Content (HTML):  
      ```html
      <p>Hello User,</p>
      <p>Please see the attached report containing last month's sales!</p>
      <p>Best,<br>An Efficient Person</p>
      ```  
    - To Recipients: Enter target email addresses (e.g., `user@example.com`)  
    - Attachments: Add the binary property `sales_report` from the previous node.  
    - Credentials: Configure Microsoft Outlook OAuth2 credentials.  
    - Connect input from `Convert Sales Summary to CSV File`.

11. **Activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Context or Link                                   |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------|
| This workflow requires a Square API token to be set as a Header Auth credential named `Square Header Auth` with the header `Authorization: Bearer <token>`. The Square API token must have permission to access locations and orders. For Outlook, OAuth2 credentials must be configured. The workflow is designed to be timezone-aware with a fixed `-05:00` offset; adjust if your timezone differs. | Square and Microsoft Outlook API credentials setup |
| The workflow currently does not handle pagination beyond 1000 orders per location per day. If your sales volume exceeds this, consider implementing pagination logic in the `Get Sales from Square` node. | Pagination limitation                             |
| The compiled sales report code is complex and depends on the structure of Square's order data. If the Square API changes, review and update the code accordingly. Always verify outputs against the Square Sales Summary dashboard to ensure accuracy. | Data accuracy and maintenance                     |
| For customization, you can modify the email recipient, subject, and body content in the `Send Report` node. You may also remove the n8n attribution or add additional recipients. | Email customization                               |
| Reference workflow used as a base for this automation: https://n8n.io/workflows/6358                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Original template workflow                         |

---

**Disclaimer:**  
The text provided here is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected content. All data handled is legal and public.