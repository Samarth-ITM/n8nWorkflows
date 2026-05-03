Sync Shopify customers to Google Sheets + Squarespace compatible csv

https://n8nworkflows.xyz/workflows/sync-shopify-customers-to-google-sheets---squarespace-compatible-csv-3280


# Sync Shopify customers to Google Sheets + Squarespace compatible csv

### 1. Workflow Overview

This workflow automates the synchronization of Shopify customers into a Google Sheets spreadsheet and generates a Squarespace-compatible CSV file for bulk importing contacts. It is designed for Shopify store owners and Squarespace website managers who want to maintain up-to-date customer lists and facilitate marketing or contact management workflows.

The workflow is logically divided into the following blocks:

- **1.1 Trigger Block:** Initiates the workflow manually or on a schedule.
- **1.2 Shopify Customers Retrieval:** Fetches all customers from Shopify using the Admin REST API with cursor-based pagination.
- **1.3 Pagination Handling:** Extracts and manages the `page_info` cursor to paginate through all customer data.
- **1.4 Data Aggregation:** Merges paginated customer data into a single collection.
- **1.5 Data Processing and Storage:** Splits aggregated data, updates Google Sheets, and formats data for Squarespace CSV export.
- **1.6 CSV Conversion:** Converts the processed data into a CSV file compatible with Squarespace Contacts import.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger Block

- **Overview:** Starts the workflow either manually or on a scheduled interval.
- **Nodes Involved:**  
  - `When clicking ‘Test workflow’` (Manual Trigger)  
  - `Schedule Trigger` (Scheduled Trigger)

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Enables manual execution for testing or on-demand runs.  
    - Configuration: Default manual trigger, no parameters.  
    - Connections: Outputs to `Get Customers` node.  
    - Edge Cases: None specific; manual trigger requires user interaction.

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Automatically triggers workflow at defined intervals (default is every minute).  
    - Configuration: Uses default interval rule (every minute).  
    - Connections: Outputs to `Get Customers` node.  
    - Edge Cases: Ensure schedule frequency matches API rate limits.

---

#### 2.2 Shopify Customers Retrieval

- **Overview:** Fetches customers from Shopify API using HTTP requests with cursor-based pagination.
- **Nodes Involved:**  
  - `Get Customers` (HTTP Request)  
  - `Assign page_info parameter` (Set)  
  - `Check page_info existence` (If)

- **Node Details:**

  - **Get Customers**  
    - Type: HTTP Request  
    - Role: Calls Shopify Admin REST API endpoint `/customers.json` to retrieve customer data.  
    - Configuration:  
      - URL: `https://{your-store}.myshopify.com/admin/api/2025-01/customers.json` (replace `{your-store}` with actual store domain).  
      - Authentication: Uses Shopify Access Token credential.  
      - Query Parameters:  
        - `limit`: 250 (max per Shopify docs)  
        - `fields`: `id,email,first_name,last_name` (retrieves only necessary fields)  
        - Conditional parameter: If `page_info` exists, uses it; otherwise, uses `status=any` to get all customers.  
      - Response: Full response with headers enabled (`Include Response Headers and Status` enabled).  
    - Inputs: From `Assign page_info parameter` or triggers.  
    - Outputs: JSON with customer data and response headers.  
    - Edge Cases:  
      - API rate limits may cause throttling errors.  
      - Missing or invalid credentials cause auth errors.  
      - Network timeouts or Shopify API downtime.  
      - Incorrect store URL or permissions.  
    - Notes: Requires manual editing of the URL to insert the correct store domain (see sticky note).

  - **Assign page_info parameter**  
    - Type: Set  
    - Role: Sets the `page_info` parameter for the next API request based on extracted pagination cursor.  
    - Configuration: Assigns `page_info` from previous node’s JSON.  
    - Inputs: From `Extract page_info` node.  
    - Outputs: To `Get Customers` node.  
    - Edge Cases: If `page_info` is empty or invalid, may cause request failure.

  - **Check page_info existence**  
    - Type: If  
    - Role: Checks if the response headers contain a `rel="next"` link indicating more pages.  
    - Configuration:  
      - Condition: Checks if the `link` header contains `rel="next"`.  
      - If **false**: No more pages; proceeds to data aggregation.  
      - If **true**: More pages exist; extracts `page_info`.  
    - Inputs: From `Get Customers`.  
    - Outputs:  
      - True branch to `Extract page_info`.  
      - False branch to `Merge Loop items`.  
    - Edge Cases:  
      - Missing or malformed `link` header causes false negatives.  
      - Pagination stops prematurely if header parsing fails.

---

#### 2.3 Pagination Handling

- **Overview:** Extracts the `page_info` cursor from the Shopify API response headers to enable cursor-based pagination.
- **Nodes Involved:**  
  - `Extract page_info` (Code)

- **Node Details:**

  - **Extract page_info**  
    - Type: Code (JavaScript)  
    - Role: Parses the `link` header from Shopify API response to extract the `page_info` query parameter for the next page.  
    - Configuration:  
      - Reads `headers.link` from the first input item.  
      - Uses regex to find URL with `rel="next"`.  
      - Parses query parameters into an object and returns them.  
    - Inputs: From `Check page_info existence` (true branch).  
    - Outputs: JSON with extracted `page_info`.  
    - Edge Cases:  
      - No `rel="next"` link returns null, ending pagination.  
      - Malformed headers or unexpected formats may cause parsing errors.

---

#### 2.4 Data Aggregation

- **Overview:** Collects all paginated customer data into a single array for further processing.
- **Nodes Involved:**  
  - `Merge Loop items` (Code)  
  - `List Customers` (Split Out)

- **Node Details:**

  - **Merge Loop items**  
    - Type: Code (JavaScript)  
    - Role: Concatenates all paginated results from multiple executions of `Get Customers` into one array.  
    - Configuration:  
      - Iterates through all outputs of `Get Customers` node.  
      - Concatenates data until no more pages are available or an error occurs.  
    - Inputs: From `Check page_info existence` (false branch) and looped `Get Customers` calls.  
    - Outputs: Single array of all customers.  
    - Edge Cases:  
      - Infinite loop risk if pagination does not terminate properly.  
      - Errors in accessing node data may cause partial results.

  - **List Customers**  
    - Type: Split Out  
    - Role: Splits the aggregated customer array into individual items for downstream processing.  
    - Configuration: Splits on `body.customers` field.  
    - Inputs: From `Merge Loop items`.  
    - Outputs: Individual customer JSON objects.  
    - Edge Cases: Empty arrays result in no output items.

---

#### 2.5 Data Processing and Storage

- **Overview:** Updates Google Sheets with customer data and prepares data for Squarespace CSV export.
- **Nodes Involved:**  
  - `Customers Spreadsheet` (Google Sheets)  
  - `Extract customers data` (Set)

- **Node Details:**

  - **Customers Spreadsheet**  
    - Type: Google Sheets  
    - Role: Appends or updates customer records in a Google Sheets document.  
    - Configuration:  
      - Operation: Append or Update  
      - Sheet: Named `sqs_contacts` (sheet ID 1358690917)  
      - Document ID: Google Sheets document ID provided (replace with your own if needed)  
      - Columns mapped:  
        - Email address ← `email`  
        - First name ← `first_name`  
        - Last name ← `last_name`  
        - Shopify Customer ID ← `id` (used as matching column)  
      - Matching column: `Shopify Customer ID` to avoid duplicates.  
    - Inputs: From `List Customers`.  
    - Outputs: To `Extract customers data`.  
    - Credentials: Requires Google Sheets OAuth2 credentials.  
    - Edge Cases:  
      - API quota limits or permission errors.  
      - Sheet or document not found.  
      - Data type mismatches.

  - **Extract customers data**  
    - Type: Set  
    - Role: Selects and renames fields to match Squarespace CSV import format.  
    - Configuration:  
      - Sets fields:  
        - `Email address`  
        - `First name`  
        - `Last name`  
      - Drops Shopify Customer ID (not needed for CSV).  
    - Inputs: From `Customers Spreadsheet`.  
    - Outputs: To `Convert to Squarespace contacts csv`.  
    - Edge Cases: Missing fields cause empty CSV columns.

---

#### 2.6 CSV Conversion

- **Overview:** Converts the processed customer data into a CSV file without a header row, compatible with Squarespace Contacts import.
- **Nodes Involved:**  
  - `Convert to Squarespace contacts csv` (Convert To File)

- **Node Details:**

  - **Convert to Squarespace contacts csv**  
    - Type: Convert To File  
    - Role: Converts JSON data into a CSV file.  
    - Configuration:  
      - Output format: CSV  
      - Header row: Disabled (Squarespace requires no header)  
    - Inputs: From `Extract customers data`.  
    - Outputs: Final CSV file ready for download or further use.  
    - Edge Cases:  
      - Empty input results in empty CSV.  
      - Data encoding or special characters may require validation.

---

### 3. Summary Table

| Node Name                     | Node Type          | Functional Role                             | Input Node(s)                   | Output Node(s)                     | Sticky Note                                                                                                                  |
|-------------------------------|--------------------|---------------------------------------------|--------------------------------|-----------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’  | Manual Trigger     | Manual start trigger                         | —                              | Get Customers                     |                                                                                                                              |
| Schedule Trigger              | Schedule Trigger   | Scheduled start trigger                       | —                              | Get Customers                     |                                                                                                                              |
| Get Customers                | HTTP Request       | Fetch Shopify customers with pagination      | When clicking ‘Test workflow’, Schedule Trigger, Assign page_info parameter | Check page_info existence          | "## Edit this node 👇\nGet your store URL and replace in the GET url: https://{your-store}.myshopify.com/admin/api/2025-01/customers.json" |
| Check page_info existence     | If                 | Checks for next page in pagination            | Get Customers                  | Merge Loop items (false), Extract page_info (true) |                                                                                                                              |
| Extract page_info             | Code               | Parses pagination cursor from response header | Check page_info existence (true) | Assign page_info parameter          |                                                                                                                              |
| Assign page_info parameter    | Set                | Sets `page_info` query parameter for next request | Extract page_info              | Get Customers                     |                                                                                                                              |
| Merge Loop items             | Code               | Aggregates all paginated customer data       | Check page_info existence (false) | List Customers                   |                                                                                                                              |
| List Customers               | Split Out          | Splits aggregated customers into individual items | Merge Loop items              | Customers Spreadsheet, Extract customers data |                                                                                                                              |
| Customers Spreadsheet        | Google Sheets      | Updates or appends customer data in Google Sheets | List Customers                | Extract customers data            | "## Clone this spreadsheet\n\nhttps://docs.google.com/spreadsheets/d/1E8i98hwiFW7XG9HuxIZrOWfuLxGFaDm3EOAGQBZjhfk/edit?usp=sharing\n\nYour spreadsheet can have up to three columns, and need to be arranged in this order (no header):\n\nEmail address\nFirst name (optional)\nLast name (optional)\nShopify Customer ID (will be ignored)" |
| Extract customers data       | Set                | Prepares data fields for Squarespace CSV export | Customers Spreadsheet          | Convert to Squarespace contacts csv |                                                                                                                              |
| Convert to Squarespace contacts csv | Convert To File | Converts JSON data to CSV without header     | Extract customers data          | —                                 |                                                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**
   - Add a **Manual Trigger** node named `When clicking ‘Test workflow’`.
   - Add a **Schedule Trigger** node named `Schedule Trigger` with default interval (every minute or as desired).

2. **Create Shopify Customers Retrieval:**
   - Add an **HTTP Request** node named `Get Customers`.
     - Set HTTP Method: GET.
     - URL: `https://{your-store}.myshopify.com/admin/api/2025-01/customers.json` (replace `{your-store}`).
     - Enable "Include Response Headers and Status".
     - Authentication: Use Shopify Access Token credential.
     - Query Parameters:
       - `limit`: 250
       - `fields`: `id,email,first_name,last_name`
       - Add conditional parameter:  
         - Name: `={{ $json.page_info ? "page_info" : "status" }}`  
         - Value: `={{ $json.page_info ? $json.page_info : 'any' }}`
   - Connect outputs of both trigger nodes to `Get Customers`.

3. **Add Pagination Check:**
   - Add an **If** node named `Check page_info existence`.
     - Condition: Check if `{{$json.headers.link}}` contains `rel="next"`.
     - Connect `Get Customers` output to this node.

4. **Add Pagination Extraction:**
   - Add a **Code** node named `Extract page_info`.
     - Paste the provided JavaScript code to parse `page_info` from `headers.link`.
     - Connect the true output of `Check page_info existence` to this node.

5. **Add Set Node for Pagination Parameter:**
   - Add a **Set** node named `Assign page_info parameter`.
     - Assign `page_info` from the output of `Extract page_info`.
     - Connect `Extract page_info` output to this node.
     - Connect this node back to `Get Customers` to loop with updated `page_info`.

6. **Add Data Aggregation:**
   - Add a **Code** node named `Merge Loop items`.
     - Paste the provided JavaScript code that concatenates all paginated results.
     - Connect the false output of `Check page_info existence` to this node.

7. **Split Aggregated Customers:**
   - Add a **Split Out** node named `List Customers`.
     - Set field to split out: `body.customers`.
     - Connect `Merge Loop items` output to this node.

8. **Add Google Sheets Node:**
   - Add a **Google Sheets** node named `Customers Spreadsheet`.
     - Operation: Append or Update.
     - Document ID: Use your Google Sheets document ID.
     - Sheet Name: Use the sheet named `sqs_contacts` or your target sheet.
     - Map columns:
       - Email address ← `email`
       - First name ← `first_name`
       - Last name ← `last_name`
       - Shopify Customer ID ← `id`
     - Matching Columns: `Shopify Customer ID`.
     - Connect `List Customers` output to this node.
     - Configure Google Sheets OAuth2 credentials.

9. **Prepare Data for CSV Export:**
   - Add a **Set** node named `Extract customers data`.
     - Assign fields:
       - `Email address` ← `Email address` (from Google Sheets output)
       - `First name` ← `First name`
       - `Last name` ← `Last name`
     - Connect `Customers Spreadsheet` output to this node.

10. **Convert to CSV:**
    - Add a **Convert To File** node named `Convert to Squarespace contacts csv`.
      - Format: CSV.
      - Disable header row.
      - Connect `Extract customers data` output to this node.

11. **Connect all nodes as per the described flow:**
    - Triggers → Get Customers → Check page_info existence →  
      - True → Extract page_info → Assign page_info parameter → Get Customers (loop)  
      - False → Merge Loop items → List Customers → Customers Spreadsheet → Extract customers data → Convert to Squarespace contacts csv.

12. **Credentials Setup:**
    - Shopify Access Token API credentials must be created and linked to `Get Customers`.
    - Google Sheets OAuth2 credentials must be created and linked to `Customers Spreadsheet`.

13. **Final Notes:**
    - Replace `{your-store}` in the `Get Customers` node URL with your actual Shopify store domain.
    - Clone the provided Google Sheets template or create your own with the required columns and sheet name.
    - Test the workflow manually first to ensure correct data retrieval and storage.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                         | Context or Link                                                                                          |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| Clone this spreadsheet for Google Sheets setup: https://docs.google.com/spreadsheets/d/1E8i98hwiFW7XG9HuxIZrOWfuLxGFaDm3EOAGQBZjhfk/edit?usp=sharing                                                                                 | Google Sheets Template                                                                                   |
| Squarespace CSV import requires no header row and columns arranged as Email Address, First Name (optional), Last Name (optional). Shopify Customer ID column is ignored.                                                             | Squarespace Contacts import format                                                                       |
| Shopify Admin REST API documentation: https://shopify.dev/docs/api/admin-rest/2025-01/resources/customer                                                                                                                           | Shopify API reference                                                                                    |
| Shopify pagination uses cursor-based `page_info` parameter; only `limit` and `fields` allowed with `page_info`. Pagination info is in response headers under `link`.                                                                | Shopify API Pagination docs                                                                              |
| Squarespace CSV import instructions: https://support.squarespace.com/hc/en-us/articles/360001280708-Building-mailing-lists#h_01H95WGA8Y7F3D2AJ4K7J30CMY                                                                             | Squarespace Contacts import guide                                                                        |
| Replace `{your-store}` in HTTP Request node URL with your actual Shopify store domain.                                                                                                                                               | Sticky note on `Get Customers` node                                                                      |
| Explore more n8n templates by the author: https://n8n.io/creators/bangank36/                                                                                                                                                        | Author’s template collection                                                                             |

---

This documentation provides a detailed, stepwise understanding of the workflow, enabling advanced users and AI agents to reproduce, modify, and troubleshoot the Shopify-to-Google Sheets synchronization and Squarespace CSV export process effectively.