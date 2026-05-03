Get all orders in Shopify to Google Sheets

https://n8nworkflows.xyz/workflows/get-all-orders-in-shopify-to-google-sheets-3218


# Get all orders in Shopify to Google Sheets

### 1. Workflow Overview

This workflow automates the retrieval of all Shopify orders and appends or updates them into a Google Sheets spreadsheet. It is designed for Shopify store owners or users who want a flexible, scalable way to export order data with full control over API parameters and pagination.

The workflow is logically divided into the following blocks:

- **1.1 Trigger Block:** Initiates the workflow either manually or on a schedule.
- **1.2 Shopify Orders Retrieval Block:** Fetches orders from Shopify using the HTTP Request node with cursor-based pagination.
- **1.3 Pagination Handling Block:** Extracts the `page_info` cursor from response headers and loops to retrieve all pages.
- **1.4 Data Processing Block:** Merges paginated results and splits out individual orders.
- **1.5 Google Sheets Integration Block:** Inserts or updates the orders into a Google Sheets spreadsheet.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger Block

**Overview:**  
This block starts the workflow either manually via a button or automatically on a schedule.

**Nodes Involved:**  
- When clicking ‘Test workflow’ (Manual Trigger)  
- Schedule Trigger

**Node Details:**

- **When clicking ‘Test workflow’**  
  - *Type:* Manual Trigger  
  - *Role:* Allows manual execution of the workflow for testing or on-demand runs.  
  - *Configuration:* No parameters; triggers workflow on user action.  
  - *Input/Output:* No input; outputs trigger signal to "Get Orders" node.  
  - *Edge Cases:* None specific; manual trigger depends on user interaction.

- **Schedule Trigger**  
  - *Type:* Schedule Trigger  
  - *Role:* Automatically triggers the workflow at defined intervals (default is every minute).  
  - *Configuration:* Default interval set to run every minute (can be customized).  
  - *Input/Output:* No input; outputs trigger signal to "Get Orders" node.  
  - *Edge Cases:* Potential for overlapping runs if execution time exceeds interval; consider adjusting interval accordingly.

---

#### 1.2 Shopify Orders Retrieval Block

**Overview:**  
This block performs the core API call to Shopify to retrieve orders using the Admin REST API with cursor-based pagination.

**Nodes Involved:**  
- Get Orders  
- Sticky Note (instructional)

**Node Details:**

- **Get Orders**  
  - *Type:* HTTP Request  
  - *Role:* Sends GET requests to Shopify's orders endpoint to fetch orders data.  
  - *Configuration:*  
    - URL: `https://{store}.myshopify.com/admin/api/2025-01/orders.json` (replace `{store}` with your Shopify store name).  
    - Authentication: Shopify Access Token via predefined credentials.  
    - Query Parameters:  
      - `limit=250` (max allowed per request)  
      - `fields=id,note,email,processed_at,customer` (customizable fields to retrieve)  
      - Conditional parameter: either `status=any` for initial request or `page_info` for pagination.  
    - Response: Full response including headers enabled (`Include Response Headers and Status` option).  
  - *Expressions:*  
    - Uses expression to set query parameter key dynamically: if `$json.page_info` exists, uses `page_info` parameter; otherwise uses `status=any`.  
  - *Input/Output:*  
    - Input: Trigger from manual or schedule trigger, or from pagination loop.  
    - Output: Full HTTP response with headers and body to "Check page_info existence".  
  - *Edge Cases:*  
    - Authentication errors if token invalid or expired.  
    - Rate limiting by Shopify API (consider adjusting `limit` or adding delays).  
    - Network timeouts or API downtime.  
    - Malformed or missing response headers could break pagination logic.

- **Sticky Note**  
  - *Content:* Instruction to replace `{your-store}` in the URL with actual Shopify store URL.  
  - *Role:* User guidance.

---

#### 1.3 Pagination Handling Block

**Overview:**  
Handles cursor-based pagination by checking for the presence of a `next` page link in response headers, extracting the `page_info` cursor, and looping the requests until all pages are retrieved.

**Nodes Involved:**  
- Check page_info existence (If node)  
- Extract page_info (Code node)  
- Assign page_info parameter (Set node)  
- Merge Loop items (Code node)

**Node Details:**

- **Check page_info existence**  
  - *Type:* If  
  - *Role:* Checks if the response headers contain a `rel="next"` link indicating more pages.  
  - *Configuration:*  
    - Condition: The `link` header does **not** contain `rel="next"`.  
    - If TRUE (no next page), ends pagination loop.  
    - If FALSE (next page exists), continues pagination.  
  - *Input/Output:*  
    - Input: Full HTTP response from "Get Orders".  
    - Output:  
      - TRUE branch to "Merge Loop items" (end loop).  
      - FALSE branch to "Extract page_info" (continue loop).  
  - *Edge Cases:*  
    - Missing or malformed `link` header may cause false negatives.  
    - If Shopify changes pagination header format, this logic may break.

- **Extract page_info**  
  - *Type:* Code  
  - *Role:* Parses the `link` header to extract the `page_info` query parameter for the next page.  
  - *Configuration:*  
    - JavaScript function parses the `link` header string to find URL with `rel="next"`, extracts query parameters, and returns them as JSON.  
  - *Input/Output:*  
    - Input: Full HTTP response headers from "Get Orders".  
    - Output: JSON object containing `page_info` string.  
  - *Edge Cases:*  
    - If `link` header is missing or does not contain `rel="next"`, returns null or empty object.  
    - Parsing errors if header format changes.

- **Assign page_info parameter**  
  - *Type:* Set  
  - *Role:* Assigns the extracted `page_info` value to the workflow data for the next API request.  
  - *Configuration:*  
    - Sets variable `page_info` with value from previous node's JSON.  
  - *Input/Output:*  
    - Input: JSON with `page_info` from "Extract page_info".  
    - Output: Data with `page_info` to "Get Orders" node to fetch next page.  
  - *Edge Cases:*  
    - If `page_info` is missing or invalid, next request may fail or loop indefinitely.

- **Merge Loop items**  
  - *Type:* Code  
  - *Role:* Attempts to concatenate all paginated results from the "Get Orders" node into a single array.  
  - *Configuration:*  
    - JavaScript loop tries to collect all outputs from "Get Orders" node indexed by iteration until an error occurs (likely when no more pages).  
  - *Input/Output:*  
    - Input: Outputs from multiple runs of "Get Orders" node.  
    - Output: Merged array of all orders data.  
  - *Edge Cases:*  
    - If "Get Orders" outputs are missing or incomplete, may return partial data.  
    - Infinite loop risk if error handling fails.

---

#### 1.4 Data Processing Block

**Overview:**  
Splits the merged orders array into individual order items for further processing.

**Nodes Involved:**  
- List Orders

**Node Details:**

- **List Orders**  
  - *Type:* Split Out  
  - *Role:* Splits the array of orders (`body.orders`) into individual items for downstream nodes.  
  - *Configuration:*  
    - Field to split out: `body.orders` (array of orders).  
  - *Input/Output:*  
    - Input: Merged orders array from "Merge Loop items".  
    - Output: Individual order JSON objects to "Google Sheets".  
  - *Edge Cases:*  
    - If `body.orders` is empty or missing, downstream nodes receive no data.

---

#### 1.5 Google Sheets Integration Block

**Overview:**  
Appends or updates the individual Shopify orders into a specified Google Sheets spreadsheet.

**Nodes Involved:**  
- Google Sheets  
- Sticky Note1 (spreadsheet clone link)

**Node Details:**

- **Google Sheets**  
  - *Type:* Google Sheets  
  - *Role:* Inserts or updates rows in Google Sheets with order data.  
  - *Configuration:*  
    - Operation: `appendOrUpdate` (adds new rows or updates existing rows based on matching columns).  
    - Document ID: Spreadsheet ID of the target Google Sheets document.  
    - Sheet Name: Sheet ID or name where orders are stored.  
    - Columns mapped: `id`, `note`, `email`, `processed_at` from Shopify order JSON.  
    - Matching column: `id` (used to detect existing rows for update).  
    - Data type conversion: Disabled (fields inserted as strings).  
  - *Credentials:* Google Sheets OAuth2 credentials required.  
  - *Input/Output:*  
    - Input: Individual order JSON objects from "List Orders".  
    - Output: Confirmation of data insertion (not used downstream).  
  - *Edge Cases:*  
    - Authentication errors if credentials expire or are invalid.  
    - API rate limits or quota exceeded errors.  
    - Data type mismatches or schema changes in the spreadsheet.

- **Sticky Note1**  
  - *Content:* Link to clone the Google Sheets template used for this workflow.  
  - *Role:* User guidance to prepare the target spreadsheet.

---

### 3. Summary Table

| Node Name                 | Node Type          | Functional Role                         | Input Node(s)                  | Output Node(s)                 | Sticky Note                                                                                     |
|---------------------------|--------------------|---------------------------------------|-------------------------------|-------------------------------|------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger     | Manual start of workflow               | —                             | Get Orders                    |                                                                                                |
| Schedule Trigger          | Schedule Trigger   | Scheduled start of workflow            | —                             | Get Orders                    |                                                                                                |
| Get Orders                | HTTP Request      | Fetch Shopify orders with pagination   | When clicking ‘Test workflow’, Schedule Trigger, Assign page_info parameter | Check page_info existence      | "## Edit this node 👇\n\nGet your store URL and replace in the GET url: https://{your-store}.myshopify.com/admin/api/2025-01/orders.json" |
| Check page_info existence | If                | Check if more pages exist (pagination) | Get Orders                    | Merge Loop items (if no next), Extract page_info (if next) |                                                                                                |
| Extract page_info         | Code               | Parse next page cursor from headers    | Check page_info existence     | Assign page_info parameter     |                                                                                                |
| Assign page_info parameter| Set                | Assign extracted cursor for next request| Extract page_info             | Get Orders                    |                                                                                                |
| Merge Loop items          | Code               | Merge all paginated results            | Check page_info existence     | List Orders                   |                                                                                                |
| List Orders               | Split Out          | Split merged orders array into items   | Merge Loop items              | Google Sheets                 |                                                                                                |
| Google Sheets             | Google Sheets      | Append or update orders in spreadsheet | List Orders                  | —                             | "## Clone this spreadsheet\n\nhttps://docs.google.com/spreadsheets/d/1KRl6aCCU2SE3Z6vB2EbTnSwSUAre0BLf9Wu6fyPlrIE/edit?usp=sharing" |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**  
   - Add a **Manual Trigger** node named "When clicking ‘Test workflow’" with default settings.  
   - Add a **Schedule Trigger** node named "Schedule Trigger" with interval set to your desired schedule (default every minute).

2. **Create Shopify Orders HTTP Request Node:**  
   - Add an **HTTP Request** node named "Get Orders".  
   - Set HTTP Method to `GET`.  
   - Set URL to `https://{your-store}.myshopify.com/admin/api/2025-01/orders.json` (replace `{your-store}` with your Shopify store domain).  
   - Under **Authentication**, select Shopify Access Token credentials (set up Shopify API Key credentials beforehand).  
   - Enable **Include Response Headers and Status** under options to get full response headers.  
   - Add Query Parameters:  
     - `limit` = `250` (max allowed)  
     - `fields` = `id,note,email,processed_at,customer` (adjust fields as needed)  
     - Add a parameter with key expression: `={{ $json.page_info ? "page_info" : "status" }}` and value expression: `={{ $json.page_info ? $json.page_info : "any" }}` to handle initial and paginated requests.  
   - Connect both trigger nodes ("When clicking ‘Test workflow’" and "Schedule Trigger") to this node.

3. **Add If Node to Check Pagination:**  
   - Add an **If** node named "Check page_info existence".  
   - Set condition to check if the response header `link` does **not** contain `rel="next"`:  
     - Expression: `{{$json.headers.link}}` does not contain `rel="next"`.  
   - Connect "Get Orders" node output to this node.

4. **Add Code Node to Extract page_info:**  
   - Add a **Code** node named "Extract page_info".  
   - Paste the JavaScript code to parse the `link` header and extract `page_info` parameter (see code in Block 1.3).  
   - Connect the FALSE output (next page exists) of "Check page_info existence" to this node.

5. **Add Set Node to Assign page_info:**  
   - Add a **Set** node named "Assign page_info parameter".  
   - Add a field named `page_info` with value set to `={{ $json.page_info }}`.  
   - Connect "Extract page_info" node to this node.

6. **Loop Back to Get Orders:**  
   - Connect "Assign page_info parameter" node output back to "Get Orders" node input to continue pagination.

7. **Handle End of Pagination:**  
   - Connect the TRUE output (no next page) of "Check page_info existence" node to a **Code** node named "Merge Loop items".  
   - In "Merge Loop items", write JavaScript code to concatenate all paginated results from "Get Orders" node outputs into one array.  
   - Connect "Merge Loop items" output to next processing node.

8. **Split Orders Array:**  
   - Add a **Split Out** node named "List Orders".  
   - Configure it to split the field `body.orders` into individual items.  
   - Connect "Merge Loop items" output to this node.

9. **Add Google Sheets Node:**  
   - Add a **Google Sheets** node named "Google Sheets".  
   - Set operation to `appendOrUpdate`.  
   - Configure Document ID with your target Google Sheets spreadsheet ID (use the provided template or your own).  
   - Set Sheet Name to the appropriate sheet/tab name or ID.  
   - Define columns mapping: map `id`, `note`, `email`, `processed_at` fields from Shopify order JSON to corresponding columns.  
   - Set matching column to `id` for update detection.  
   - Connect "List Orders" output to this node.  
   - Configure Google Sheets OAuth2 credentials.

10. **Add Sticky Notes (Optional):**  
    - Add sticky notes with instructions for replacing store URL and cloning the Google Sheets template for user guidance.

---

### 5. General Notes & Resources

| Note Content                                                                                                           | Context or Link                                                                                         |
|------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| Clone the Google Sheets template here: https://docs.google.com/spreadsheets/d/1KRl6aCCU2SE3Z6vB2EbTnSwSUAre0BLf9Wu6fyPlrIE/edit?usp=sharing | Google Sheets template used for storing Shopify orders.                                               |
| Shopify Admin REST API documentation: https://shopify.dev/docs/api/admin-rest/2025-01/resources/order#get-orders?status=any | Official API docs for Shopify orders endpoint.                                                        |
| Shopify pagination guide: https://shopify.dev/docs/api/admin-rest/usage/pagination                                      | Details on cursor-based pagination used in Shopify API.                                               |
| Explore more n8n templates by the author: https://n8n.io/creators/bangank36/                                            | Additional workflow templates for automation inspiration.                                            |

---

This reference document fully describes the workflow’s structure, logic, and configuration, enabling advanced users or AI agents to understand, reproduce, and modify the workflow effectively.