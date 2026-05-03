Pull Square Sales Summary Reports for Automated Reporting and Analysis

https://n8nworkflows.xyz/workflows/pull-square-sales-summary-reports-for-automated-reporting-and-analysis-6358


# Pull Square Sales Summary Reports for Automated Reporting and Analysis

### 1. Workflow Overview

This workflow automates the retrieval and compilation of daily sales summary reports from the Square API for all registered locations. It is designed as a reusable sub-workflow to be triggered by another workflow that provides a specific report date. The workflow fetches all locations, retrieves their completed sales orders for the specified date, filters out locations without sales, and calculates detailed financial metrics matching the Square Sales Summary Dashboard.

**Target Use Cases:**  
- Automated daily sales reporting for finance and operations teams  
- Historical sales data archiving into databases or spreadsheets  
- Feeding sales data into accounting or commission systems  
- Triggering notifications or alerts based on sales metrics  

**Logical Blocks:**  
- **1.1 Input Reception:** Receives the report date from an external workflow.  
- **1.2 Retrieve Locations:** Fetches all Square locations via API.  
- **1.3 Process Locations List:** Splits the list of locations to process each individually.  
- **1.4 Retrieve Sales Data:** For each location, fetches completed orders on the report date.  
- **1.5 Filter Locations Without Sales:** Excludes locations with no sales orders.  
- **1.6 Compile Sales Report:** Aggregates and calculates sales metrics per location.  
- **1.7 Documentation and Configuration Notes:** Provides detailed instructions and usage notes via sticky notes.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception
- **Overview:** Receives the report date from an external workflow trigger.  
- **Nodes Involved:**  
  - When Executed by Another Workflow  
- **Node Details:**  
  - Type: Execute Workflow Trigger  
  - Role: Entry point that accepts workflow input parameters  
  - Configuration: Accepts one input parameter named `report_date` formatted as `YYYY-MM-DD`  
  - Connections: Outputs to "Get Square Locations"  
  - Edge Cases: Missing or incorrectly formatted date input will cause errors downstream; no validation is implemented here.  
  - Notes: Should be triggered externally; this node does not run independently.

#### 1.2 Retrieve Locations
- **Overview:** Fetches all Square locations associated with the account using the Square Locations API endpoint.  
- **Nodes Involved:**  
  - Get Square Locations  
- **Node Details:**  
  - Type: HTTP Request  
  - Role: Retrieves all registered locations  
  - Configuration:  
    - Method: GET  
    - URL: `https://connect.squareup.com/v2/locations`  
    - Authentication: Header Auth containing a Bearer token (Square API key)  
    - Headers: Content-Type set to application/json  
  - Connections: Outputs to "Turn Locations Into List"  
  - Edge Cases: Authentication failures, API rate limits, or network issues may cause failure.  
  - Version: Requires n8n supporting HTTP Request v4.2 or higher.

#### 1.3 Process Locations List
- **Overview:** Splits the locations response array into individual items to process each location separately.  
- **Nodes Involved:**  
  - Turn Locations Into List  
- **Node Details:**  
  - Type: Split Out  
  - Role: Converts the `locations` array in the JSON response to individual items for downstream processing  
  - Configuration:  
    - Field to split out: `locations`  
    - Fields to include: `id` only (minimal data for subsequent query)  
  - Connections: Outputs each location item to "Get Sales from Square"  
  - Edge Cases: If the `locations` field is empty or missing, downstream nodes may receive no data.

#### 1.4 Retrieve Sales Data
- **Overview:** For each location, fetches all completed orders from the Square Orders API for the specified report date.  
- **Nodes Involved:**  
  - Get Sales from Square  
- **Node Details:**  
  - Type: HTTP Request  
  - Role: Retrieves sales orders filtered by location and date  
  - Configuration:  
    - Method: POST  
    - URL: `https://connect.squareup.com/v2/orders/search`  
    - Body: JSON specifying  
      - `location_ids` with current location ID  
      - Filter for order state "COMPLETED"  
      - Date-time filter for `created_at` from start to end of report date, timezone set to `-05:00` (Toronto/New York)  
      - Limit of 1000 orders (pagination not implemented)  
    - Authentication: Header Auth with Square API token  
    - Headers: Content-Type application/json  
  - Connections: Outputs to "Ignore Locations w/o Sales"  
  - Edge Cases:  
    - If orders exceed 1000, some may be missed (pagination needed for high-volume locations).  
    - Timezone mismatch if user not in EST - may cause incorrect date filtering.  
    - Authentication or network failures.  
  - Notes: Timezone offset should be updated for other regions as detailed in sticky notes.

#### 1.5 Filter Locations Without Sales
- **Overview:** Checks if the current location has any sales orders; if none, the location is skipped from further processing.  
- **Nodes Involved:**  
  - Ignore Locations w/o Sales  
- **Node Details:**  
  - Type: If  
  - Role: Filter node checking if `orders` array is not empty  
  - Configuration:  
    - Condition: `$json.orders` is a non-empty array  
  - Connections:  
    - True (has sales) → "Compile Sales Reports"  
    - False (no sales) → no output (end of branch)  
  - Edge Cases: If API returns unexpected data format, condition may fail.

#### 1.6 Compile Sales Report
- **Overview:** Aggregates sales data for the location and date, computing detailed financial metrics matching Square’s sales summary dashboard.  
- **Nodes Involved:**  
  - Compile Sales Reports  
- **Node Details:**  
  - Type: Code (JavaScript)  
  - Role: Processes the array of orders to compute totals for sales, tax, discounts, tips, returns, tender types, fees, and rounding adjustments  
  - Configuration:  
    - Runs once per location item  
    - Uses data from `orders` array and metadata from other nodes  
    - Calculates:  
      - gross sales, net sales, total returns, total discount, total tax, total tip, cash rounding, payments by cash/card/gift/other, fees, net totals  
    - Divides all monetary values by 100 to convert from cents to dollars  
  - Connections: Final output (not connected to further nodes in this workflow)  
  - Edge Cases:  
    - Assumes all order data fields present; missing fields default to zero  
    - Complex refund handling with lookups for original sales  
    - Potential mismatch if Square data structure changes  
  - Notes: Designed to produce totals exactly matching Square’s dashboard for accuracy.

#### 1.7 Documentation and Configuration Notes
- **Overview:** Provides detailed instructions, warnings, credentials setup, and usage notes to users implementing or customizing the workflow.  
- **Nodes Involved:**  
  - Multiple Sticky Note nodes (Sticky Note, Sticky Note1, Sticky Note2, Sticky Note3, Sticky Note4, Sticky Note5)  
- **Node Details:**  
  - Type: Sticky Note  
  - Content covers:  
    - Overall program purpose and prerequisites  
    - Trigger usage instructions  
    - HTTP nodes’ roles  
    - Timezone warnings and customization advice  
    - Credential setup guidance (Square API header auth)  
    - Suggestions for expansion (pagination, integration with email, databases, etc.)  
  - Positioned to group logically with respective functional blocks  
  - No connections (informational only)

---

### 3. Summary Table

| Node Name                  | Node Type                | Functional Role                                  | Input Node(s)                     | Output Node(s)                  | Sticky Note                                                                                      |
|----------------------------|--------------------------|-------------------------------------------------|----------------------------------|--------------------------------|------------------------------------------------------------------------------------------------|
| When Executed by Another Workflow | Execute Workflow Trigger | Receives report date input from external workflow | None                             | Get Square Locations            | ## Trigger - This workflow should be executed by another workflow. - The main workflow should specify the report date in this format (YYYY-MM-DD) |
| Get Square Locations        | HTTP Request             | Retrieves all Square locations via API           | When Executed by Another Workflow | Turn Locations Into List        | ## Get Square Locations and Process Each One Separately - This HTTP node will connect to the Square Locations API to fetch all of your locations  |
| Turn Locations Into List    | Split Out                | Splits locations array into individual items     | Get Square Locations             | Get Sales from Square           | ## Get Square Locations and Process Each One Separately - This HTTP node will connect to the Square Locations API to fetch all of your locations  |
| Get Sales from Square       | HTTP Request             | Fetches completed orders for each location/date  | Turn Locations Into List         | Ignore Locations w/o Sales      | ## Get Sales from Square - This HTTP node will get all of the orders for the given location on the specified report date                                |
| Ignore Locations w/o Sales  | If                       | Filters out locations without sales orders       | Get Sales from Square            | Compile Sales Reports           |                                                                                               |
| Compile Sales Reports       | Code                     | Calculates detailed sales summary metrics        | Ignore Locations w/o Sales       | None                           | ## Compile a Report for Each Location - This code node will calculate the totals for each location. Please ensure the numbers match EXACTLY with the Square Sales Summary Dashboard |
| Sticky Note                | Sticky Note              | Documentation and usage instructions              | None                            | None                           | ## Programatically Pull Square Report Data Into N8N (Detailed usage, credential setup, customizations, use cases)                                   |
| Sticky Note1               | Sticky Note              | Trigger usage instructions                         | None                            | None                           | ## Trigger - This workflow should be executed by another workflow. - The main workflow should specify the report date in this format (YYYY-MM-DD)     |
| Sticky Note2               | Sticky Note              | Explains purpose of retrieving locations          | None                            | None                           | ## Get Square Locations and Process Each One Separately - This HTTP node will connect to the Square Locations API to fetch all of your locations       |
| Sticky Note3               | Sticky Note              | Explains purpose of fetching sales orders         | None                            | None                           | ## Get Sales from Square - This HTTP node will get all of the orders for the given location on the specified report date                              |
| Sticky Note4               | Sticky Note              | Explains purpose of sales data compilation         | None                            | None                           | ## Compile a Report for Each Location - This code node will calculate the totals for each location. Please ensure the numbers match EXACTLY with the Square Sales Summary Dashboard |
| Sticky Note5               | Sticky Note              | Timezone warning and customization advice         | None                            | None                           | ## Timezone Warning - This HTTP Node is currently set to the **Toronto Timezone.** - If you have a different timezone, please update the setting by changing the "start_at" and "end_at" parameters from "-05:00" to your local timezone. - For example, in Brazil this would be "-04:00" |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node:**  
   - Add an **Execute Workflow Trigger** node named `When Executed by Another Workflow`.  
   - Configure it to accept input parameter:  
     - Name: `report_date`  
     - Format: String, expected as `YYYY-MM-DD`.  
   - Position: Starting point of the workflow.

2. **Create "Get Square Locations" HTTP Request Node:**  
   - Add an **HTTP Request** node named `Get Square Locations`.  
   - Set HTTP Method to `GET`.  
   - Set URL to `https://connect.squareup.com/v2/locations`.  
   - Under Headers, add `Content-Type: application/json`.  
   - Authentication: Use Header Auth credential with your Square API token as `Authorization: Bearer <your-api-key>`.  
   - Connect output of `When Executed by Another Workflow` to this node.

3. **Create "Turn Locations Into List" Split Out Node:**  
   - Add a **Split Out** node named `Turn Locations Into List`.  
   - Set Field to Split Out to `locations`.  
   - Set Fields to Include: `id`.  
   - Connect output of `Get Square Locations` to this node.

4. **Create "Get Sales from Square" HTTP Request Node:**  
   - Add an **HTTP Request** node named `Get Sales from Square`.  
   - Set HTTP Method to `POST`.  
   - Set URL to `https://connect.squareup.com/v2/orders/search`.  
   - Set the request body type to JSON.  
   - Use the following JSON body with expressions:  
     ```json
     {
       "location_ids": ["{{ $json.locations.id }}"],
       "query": {
         "filter": {
           "state_filter": { "states": ["COMPLETED"] },
           "date_time_filter": {
             "created_at": {
               "start_at": "{{ $json.report_date }}T00:00:00-05:00",
               "end_at": "{{ $json.report_date }}T23:59:59-05:00"
             }
           }
         }
       },
       "limit": 1000,
       "return_entries": false
     }
     ```
   - Replace `$json.report_date` with expression referencing `When Executed by Another Workflow` node’s input parameter.  
   - Headers: Add `Content-Type: application/json`.  
   - Authentication: Use Header Auth credential with Square API token.  
   - Connect output of `Turn Locations Into List` to this node.

5. **Create "Ignore Locations w/o Sales" If Node:**  
   - Add an **If** node named `Ignore Locations w/o Sales`.  
   - Condition: Check if `$json.orders` array is not empty.  
   - If True, continue; if False, stop branch.  
   - Connect output of `Get Sales from Square` to this node.

6. **Create "Compile Sales Reports" Code Node:**  
   - Add a **Code** node named `Compile Sales Reports`.  
   - Mode: Run Once For Each Item.  
   - Paste the JavaScript code that computes sales aggregates (as per original code).  
   - Connect True output of `Ignore Locations w/o Sales` to this node.

7. **Add Sticky Notes (Optional but Recommended):**  
   - Add Sticky Note nodes containing the detailed information on credentials setup, timezone warnings, usage instructions, and customization notes as per original workflow.  
   - Position them logically near related nodes for clarity.

8. **Credential Setup:**  
   - Create a new Credential of type **HTTP Header Auth** named e.g. `Square Header Auth`.  
   - Set header name to `Authorization`.  
   - Set value to `Bearer <your-square-access-token>`.  
   - Assign this credential to both HTTP Request nodes.

9. **Timezone Adjustments:**  
   - By default, the workflow uses `-05:00` (Toronto/New York).  
   - Modify the `"start_at"` and `"end_at"` time suffixes in the sales search JSON body to match your local timezone offset if necessary.

10. **Testing:**  
    - Trigger the workflow manually or from another workflow passing a valid `report_date` string (e.g., `2025-07-21`).  
    - Verify that the sales summary output matches your Square dashboard for that date and locations.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                               | Context or Link                                                             |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------|
| This workflow is designed to be reusable as a sub-workflow integrated into larger automation pipelines. It expects an input parameter for the report date and outputs detailed sales summaries per location. Use it to automate daily reporting and analytics.                                                                                                                                                  | General usage note                                                        |
| Square API credentials must be created and configured as HTTP Header Auth credentials in n8n, with the header name `Authorization` and value formatted as `Bearer <your-token>`.                                                                                                                                                                                                                            | Credential Setup instruction                                              |
| Timezone handling is critical: the workflow currently uses fixed timezone offsets for filtering orders by date. Adjust the `start_at` and `end_at` fields in the "Get Sales from Square" HTTP Request node to your local timezone offset to ensure accurate daily reporting.                                                                                                                                      | Timezone warning (see Sticky Note5)                                      |
| The workflow currently does not handle pagination for locations with more than 1000 orders per day. To extend functionality, implement pagination in the "Get Sales from Square" node.                                                                                                                                                                                                                      | Customization suggestion                                                 |
| Potential integration expansions include sending compiled reports via email or Slack, storing data in databases (MySQL, PostgreSQL), or pushing to accounting software like QuickBooks or Xero.                                                                                                                                                                                                             | Use cases and expansion ideas                                            |
| The JavaScript code node `Compile Sales Reports` is designed to precisely match Square’s Sales Summary figures, including complex refund and tender fee calculations. Any changes to Square’s API data structure may require code updates.                                                                                                                                                                    | Critical note on code accuracy and maintenance                           |
| For more information about Square API endpoints and data structure, consult Square’s official developer documentation: https://developer.squareup.com/reference/square/orders-api/search-orders                                                                                                                                                                                                              | Official API documentation link                                          |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created using n8n, an integration and automation tool. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.