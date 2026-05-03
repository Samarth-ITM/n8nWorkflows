Get all orders in Squarespace to Google Sheets

https://n8nworkflows.xyz/workflows/get-all-orders-in-squarespace-to-google-sheets-3116


# Get all orders in Squarespace to Google Sheets

### 1. Workflow Overview

This workflow automates the retrieval of all orders from a Squarespace store using the Squarespace Commerce API and appends or updates them into a Google Sheets spreadsheet. It is designed to handle large datasets efficiently by using pagination and allows dynamic filtering through configurable parameters.

The workflow is logically divided into the following blocks:

- **1.1 Trigger Block:** Initiates the workflow either manually or on a schedule.
- **1.2 Global Parameters Configuration:** Sets API request parameters dynamically for flexible querying.
- **1.3 Squarespace Orders Retrieval:** Calls the Squarespace Orders API with pagination support to fetch all relevant orders.
- **1.4 Data Processing:** Splits the batch of orders into individual order records.
- **1.5 Google Sheets Integration:** Inserts or updates the order data into a specified Google Sheets spreadsheet.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger Block

- **Overview:** This block starts the workflow execution either manually by user interaction or automatically on a scheduled interval.
- **Nodes Involved:**  
  - On clicking 'execute' (Manual Trigger)  
  - Schedule Trigger

- **Node Details:**

  - **On clicking 'execute'**  
    - Type: Manual Trigger  
    - Role: Allows manual initiation of the workflow for on-demand execution.  
    - Configuration: Default, no parameters.  
    - Input: None  
    - Output: Triggers the Globals node.  
    - Edge Cases: None specific; user must manually trigger.

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Automatically triggers the workflow at defined intervals (default is every minute, but can be customized).  
    - Configuration: Default interval set to run continuously (empty interval object).  
    - Input: None  
    - Output: Triggers the Globals node.  
    - Edge Cases: Scheduling conflicts or overlapping executions if workflow runs longer than interval.

#### 1.2 Global Parameters Configuration

- **Overview:** Sets and manages global variables that define API request parameters such as API version, date filters, pagination cursor, fulfillment status, and maximum pages to fetch.
- **Nodes Involved:**  
  - Globals

- **Node Details:**

  - **Globals**  
    - Type: Set  
    - Role: Defines and assigns key parameters used in the API request dynamically.  
    - Configuration:  
      - `api-version`: Default "1.0" (required by Squarespace API).  
      - `modifiedAfter`: Empty string by default; ISO 8601 datetime to filter orders modified after this date.  
      - `modifiedBefore`: Empty string by default; ISO 8601 datetime to filter orders modified before this date.  
      - `cursor`: Empty string by default; used for pagination cursor, mutually exclusive with date filters.  
      - `fulfillmentStatus`: Empty string by default; optional filter with values PENDING, FULFILLED, or CANCELED.  
      - `maxPage`: Default -1, meaning infinite pagination to fetch all pages.  
    - Input: Trigger nodes (Manual or Schedule).  
    - Output: Passes parameters to Query Orders node.  
    - Edge Cases:  
      - Invalid date formats may cause API errors.  
      - Using cursor with other filters simultaneously may cause API conflicts.  
      - Setting maxPage to a very high number may cause long execution times or rate limiting.

#### 1.3 Squarespace Orders Retrieval

- **Overview:** Queries the Squarespace Orders API endpoint to retrieve orders in paginated batches according to the parameters set in the Globals node.
- **Nodes Involved:**  
  - Query Orders

- **Node Details:**

  - **Query Orders**  
    - Type: HTTP Request  
    - Role: Calls Squarespace Commerce API to fetch orders with pagination support.  
    - Configuration:  
      - URL dynamically constructed using `api-version` from Globals: `https://api.squarespace.com/{{ $json["api-version"] }}/commerce/orders`.  
      - Authentication: HTTP Header Auth using stored Squarespace API Key credential.  
      - Query Parameters:  
        - `modifiedAfter`, `modifiedBefore`, `cursor`, `fulfillmentStatus` dynamically set from Globals node values.  
      - Pagination:  
        - Uses the `nextPageCursor` from API response to fetch subsequent pages.  
        - `maxRequests` set to `Infinity` if `maxPage` is -1, otherwise limited by `maxPage`.  
        - Pagination completes when `nextPageCursor` is absent.  
      - HTTP Method: GET (default).  
    - Input: Globals node.  
    - Output: Passes response JSON to Split Out Order node.  
    - Edge Cases:  
      - API rate limiting or authentication failure (invalid API key).  
      - Pagination cursor missing or malformed causing incomplete data retrieval.  
      - Network timeouts or API downtime.  
      - Invalid query parameters causing API errors.

#### 1.4 Data Processing

- **Overview:** Processes the batch response by splitting the array of orders into individual order items for further processing.
- **Nodes Involved:**  
  - Split Out Order

- **Node Details:**

  - **Split Out Order**  
    - Type: Split Out  
    - Role: Splits the `result` array from the API response into individual items, enabling row-wise processing.  
    - Configuration:  
      - Field to split out: `result` (the array of orders in the API response).  
    - Input: Query Orders node.  
    - Output: Passes each order item individually to the Google Sheets node.  
    - Edge Cases:  
      - Empty or missing `result` field leads to no output items.  
      - Malformed data structure may cause errors.

#### 1.5 Google Sheets Integration

- **Overview:** Inserts or updates each individual order record into a Google Sheets spreadsheet, mapping multiple order fields to corresponding spreadsheet columns.
- **Nodes Involved:**  
  - Squarespace Orders Spreadsheet

- **Node Details:**

  - **Squarespace Orders Spreadsheet**  
    - Type: Google Sheets  
    - Role: Appends or updates rows in a specified Google Sheets document and sheet with order data.  
    - Configuration:  
      - Operation: `appendOrUpdate` — adds new rows or updates existing rows based on matching columns.  
      - Document ID: Set to a specific Google Sheets document (spreadsheet ID).  
      - Sheet Name: Numeric sheet ID corresponding to the "squarespace_orders" sheet.  
      - Matching Columns: Uses "Order ID" to identify existing rows for update.  
      - Columns Mapping: Maps multiple fields from the order JSON to spreadsheet columns, including customer email, order totals, billing/shipping addresses, fulfillment status, payment info, line items, and more.  
      - Credentials: Google Sheets OAuth2 credentials configured for access.  
    - Input: Split Out Order node (individual order items).  
    - Output: None (end of workflow).  
    - Edge Cases:  
      - Google Sheets API quota limits or authentication errors.  
      - Missing or mismatched columns in the spreadsheet causing data loss or errors.  
      - Large volumes of data may cause delays or timeouts.  
      - Data type mismatches if fields are unexpectedly null or missing.

---

### 3. Summary Table

| Node Name                 | Node Type          | Functional Role                          | Input Node(s)             | Output Node(s)               | Sticky Note                                                                                                                             |
|---------------------------|--------------------|----------------------------------------|---------------------------|-----------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| On clicking 'execute'      | Manual Trigger     | Manual start of workflow                | None                      | Globals                     |                                                                                                                                         |
| Schedule Trigger          | Schedule Trigger   | Scheduled automatic start               | None                      | Globals                     |                                                                                                                                         |
| Globals                   | Set                | Defines API parameters dynamically      | On clicking 'execute', Schedule Trigger | Query Orders                | ## Edit this node 👇                                                                                                                     |
| Query Orders              | HTTP Request       | Fetches orders from Squarespace API     | Globals                    | Split Out Order             |                                                                                                                                         |
| Split Out Order           | Split Out          | Splits batch orders into individual items | Query Orders               | Squarespace Orders Spreadsheet |                                                                                                                                         |
| Squarespace Orders Spreadsheet | Google Sheets     | Inserts or updates order data in spreadsheet | Split Out Order            | None                        |                                                                                                                                         |
| Sticky Note3              | Sticky Note        | Instruction to edit Globals node        | None                      | None                        | ## Edit this node 👇                                                                                                                     |
| Sticky Note               | Sticky Note        | Workflow description and setup instructions | None                      | None                        | ## Get all Squarespace Orders<br>Retrieves all Squarespace Orders and saves them into a Google Sheets spreadsheet using the Squarespace Commerce API<br><br>### Setup<br>Open `Globals` node and update the values below 👇<br><br>- **api-version** (string, required) – The current API version (see Squarespace Orders API documentation).<br>- **modifiedAfter**={a-datetime} (string, conditional) – Fetch orders modified after a specific date (ISO 8601 format).<br>- **modifiedBefore**={b-datetime} (string, conditional) – Fetch orders modified before a specific date (ISO 8601 format).<br>- **cursor**={c} (string, conditional) – Used for pagination, cannot be combined with other filters.<br>- **fulfillmentStatus**: PENDING, FULFILLED, or CANCELED.<br>- **maxPage** – Set -1 to enables infinite pagination to fetch all available orders.<br> |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: To allow manual execution of the workflow.  
   - No special configuration needed.

2. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Purpose: To run the workflow automatically on a schedule.  
   - Configure interval as desired (default is every minute).  
   - Connect output to the Globals node.

3. **Create Globals Node (Set Node)**  
   - Type: Set  
   - Purpose: Define API parameters for the Squarespace Orders API request.  
   - Add the following fields with default values:  
     - `api-version` (string): "1.0"  
     - `modifiedAfter` (string): "" (empty)  
     - `modifiedBefore` (string): "" (empty)  
     - `cursor` (string): "" (empty)  
     - `fulfillmentStatus` (string): "" (empty)  
     - `maxPage` (number): -1 (for infinite pagination)  
   - Connect outputs from Manual Trigger and Schedule Trigger nodes to this node.

4. **Create Query Orders Node (HTTP Request)**  
   - Type: HTTP Request  
   - Purpose: Fetch orders from Squarespace API with pagination.  
   - Configure:  
     - URL: `https://api.squarespace.com/{{ $json["api-version"] }}/commerce/orders`  
     - Authentication: HTTP Header Auth using Squarespace API Key credential (create and configure credentials beforehand).  
     - Query Parameters:  
       - `modifiedAfter` = `={{ $json.modifiedAfter }}`  
       - `modifiedBefore` = `={{ $json.modifiedBefore }}`  
       - `cursor` = `={{ $json.cursor }}`  
       - `fulfillmentStatus` = `={{ $json.fulfillmentStatus }}`  
     - Pagination:  
       - Enable pagination with parameters:  
         - Cursor parameter name: `cursor`  
         - Cursor value: `={{ $response.body.pagination.nextPageCursor }}`  
         - Maximum requests: `={{ $json.maxPage === -1 ? Infinity : $json.maxPage }}`  
         - Pagination complete when: no `nextPageCursor` in response.  
   - Connect output from Globals node to this node.

5. **Create Split Out Order Node (Split Out)**  
   - Type: Split Out  
   - Purpose: Split the array of orders into individual order items.  
   - Configure:  
     - Field to split out: `result` (the array field in API response containing orders).  
   - Connect output from Query Orders node to this node.

6. **Create Squarespace Orders Spreadsheet Node (Google Sheets)**  
   - Type: Google Sheets  
   - Purpose: Append or update order data into a Google Sheets spreadsheet.  
   - Configure:  
     - Operation: `appendOrUpdate`  
     - Document ID: Use your Google Sheets spreadsheet ID (e.g., from the provided template).  
     - Sheet Name: Use the sheet ID or name where orders will be stored.  
     - Matching Columns: Set to `Order ID` to avoid duplicates and update existing rows.  
     - Columns Mapping: Map order JSON fields to spreadsheet columns, including but not limited to:  
       - Email, Total, Currency, Order ID, Subtotal, Billing and Shipping addresses, Fulfillment Status, Payment info, Line items, etc.  
     - Credentials: Configure Google Sheets OAuth2 credentials with access to the target spreadsheet.  
   - Connect output from Split Out Order node to this node.

7. **Connect Manual Trigger and Schedule Trigger nodes to Globals node**  
   - Both trigger nodes should feed into the Globals node to set parameters before querying.

8. **Connect Globals node to Query Orders node**  
   - Pass parameters for API request.

9. **Connect Query Orders node to Split Out Order node**  
   - Pass batch order data for splitting.

10. **Connect Split Out Order node to Squarespace Orders Spreadsheet node**  
    - Pass individual order records for insertion/update.

---

### 5. General Notes & Resources

| Note Content                                                                                                                  | Context or Link                                                                                                   |
|-------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| Workflow retrieves all Squarespace orders and saves them into Google Sheets using the Squarespace Commerce API with pagination. | Workflow purpose and overview.                                                                                   |
| Configure the `Globals` node to set API version, date filters, pagination cursor, fulfillment status, and max pages to fetch.  | Parameter customization instructions.                                                                             |
| Requires Squarespace API Key credential and Google Sheets OAuth2 credentials.                                                  | Credential setup requirements.                                                                                    |
| Use the [Squarespace order export](https://beyondspace.studio/blog/how-to-export-orders-data-from-squarespace) feature to create a reference sheet. | Reference for Google Sheets template setup.                                                                       |
| Google Sheets template available: [https://docs.google.com/spreadsheets/d/1hXR3N6xr7zItmGbSlYZmgZ6hf8f3Z8xr-nSWp4AZCqY](https://docs.google.com/spreadsheets/d/1hXR3N6xr7zItmGbSlYZmgZ6hf8f3Z8xr-nSWp4AZCqY/edit?usp=sharing) | Template spreadsheet for order data.                                                                              |
| Explore more n8n templates by the creator: [https://n8n.io/creators/bangank36/](https://n8n.io/creators/bangank36/)             | Additional workflow templates for automation inspiration.                                                        |
| Pagination uses `nextPageCursor` from Squarespace API response to fetch all orders efficiently.                               | Important for handling large datasets without missing data.                                                      |
| Avoid combining `cursor` with other filters (`modifiedAfter`, `modifiedBefore`) as per Squarespace API constraints.            | API usage best practice to prevent request errors.                                                                |

---

This documentation provides a complete understanding of the workflow structure, logic, and configuration, enabling users and AI agents to reproduce, modify, or troubleshoot the workflow effectively.