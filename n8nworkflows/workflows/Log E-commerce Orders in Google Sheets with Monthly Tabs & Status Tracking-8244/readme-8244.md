Log E-commerce Orders in Google Sheets with Monthly Tabs & Status Tracking

https://n8nworkflows.xyz/workflows/log-e-commerce-orders-in-google-sheets-with-monthly-tabs---status-tracking-8244


# Log E-commerce Orders in Google Sheets with Monthly Tabs & Status Tracking

### 1. Workflow Overview

This workflow is designed to automate the logging of e-commerce orders into a Google Sheets spreadsheet, organizing data by monthly tabs and tracking order statuses. It is intended for online stores (e.g., Shopify) that want to maintain a live, structured record of orders with status updates in a Google Sheet.

The workflow logically divides into the following blocks:

- **1.1 Input Reception:** Receives webhooks triggered by order events (creation, update, fulfillment) from the e-commerce platform.
- **1.2 Configuration and Metadata Fetch:** Sets spreadsheet ID configuration and fetches metadata about existing sheets (monthly tabs).
- **1.3 Sheet Naming and Existence Check:** Generates a dynamic sheet name based on the current month and year, and checks if the corresponding monthly tab exists.
- **1.4 Sheet Creation and Setup:** If the monthly tab does not exist, creates it and writes headers, applies formatting, data validation, and conditional formatting rules.
- **1.5 Data Preparation:** Formats order data extracted from the webhook for insertion, including date conversion, order line item formatting, and payment mode interpretation.
- **1.6 Data Append:** Appends the new order record to either the existing monthly sheet or the newly created one.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Captures incoming HTTP POST webhooks from Shopify (or similar platform) when orders are created or updated.

- **Nodes Involved:**  
  - `Order created`

- **Node Details:**  
  - Type: Webhook  
  - Role: Entry point for order event data.  
  - Configuration: Listens on a unique webhook path with HTTP method POST.  
  - Inputs: None (entry node).  
  - Outputs: Emits webhook payload including order details.  
  - Edge Cases: Webhook authentication failure (not configured here), malformed payload, missed events if webhook URL changes.  

---

#### 2.2 Configuration and Metadata Fetch

- **Overview:**  
  Sets the Google Sheets spreadsheet ID to target, then retrieves metadata about existing sheets to determine if the current month’s tab exists.

- **Nodes Involved:**  
  - `Config (set spreadsheetId)`  
  - `Get Order Sheets metadata`

- **Node Details:**  
  - `Config (set spreadsheetId)`  
    - Type: Set  
    - Role: Stores spreadsheetId as a variable for downstream usage.  
    - Configuration: Hardcoded or parameterized string value `<spreadsheetID>`.  
    - Inputs: Receives data from webhook node.  
    - Outputs: Provides spreadsheetId to subsequent HTTP requests.  
    - Edge Cases: Missing or invalid spreadsheetId can break all sheet operations.  

  - `Get Order Sheets metadata`  
    - Type: HTTP Request (Google Sheets API)  
    - Role: Fetches sheet titles from the spreadsheet to detect existing monthly tabs.  
    - Configuration: GET request to Google Sheets API endpoint for spreadsheet metadata, requesting only sheet titles.  
    - Authentication: Google Sheets OAuth2 credentials required.  
    - Inputs: Receives spreadsheetId from config node.  
    - Outputs: Sheet metadata JSON used for logic branching.  
    - Edge Cases: API authentication failure, quota limits, invalid spreadsheetId.  

---

#### 2.3 Sheet Naming and Existence Check

- **Overview:**  
  Generates the monthly sheet name based on current date in “MONTH ORDERS’YY” format, then checks if this sheet already exists in the spreadsheet.

- **Nodes Involved:**  
  - `Generate Sheet Name`  
  - `If`

- **Node Details:**  
  - `Generate Sheet Name`  
    - Type: Set  
    - Role: Creates a formatted string representing the current month and year (e.g., "JUNE ORDERS'24").  
    - Configuration: Uses JavaScript date functions with timezone 'Asia/Kolkata' to build the name, cleans line breaks and apostrophes.  
    - Inputs: Receives sheet metadata JSON from previous node.  
    - Outputs: Sheet name string for branch decision and sheet creation.  
    - Edge Cases: Timezone misconfiguration, date parsing errors.  

  - `If`  
    - Type: If (Boolean condition)  
    - Role: Checks if the generated sheet name exists in the array of sheet titles fetched from the spreadsheet metadata.  
    - Configuration: Runs a JavaScript function that compares cleaned sheet names to determine presence.  
    - Inputs: Receives generated sheet name and metadata sheet titles.  
    - Outputs: Two branches: true (sheet exists), false (sheet does not exist).  
    - Edge Cases: Case sensitivity, unexpected formatting in sheet titles.  

---

#### 2.4 Sheet Creation and Setup

- **Overview:**  
  When the monthly sheet does not exist, creates it and writes headers along with data validation and conditional formatting for the status column.

- **Nodes Involved:**  
  - `Set Sheet Starting row col`  
  - `Create Month Sheet`  
  - `Write Headers (A1:I1)`

- **Node Details:**  
  - `Set Sheet Starting row col`  
    - Type: Set  
    - Role: Prepares the A1 notation for the newly created sheet range, escaping special characters.  
    - Inputs: Generated sheet name string.  
    - Outputs: Prepared range string for Google Sheets API.  
    - Edge Cases: Apostrophes or special characters causing invalid range notation.  

  - `Create Month Sheet`  
    - Type: HTTP Request (Google Sheets API)  
    - Role: Sends batchUpdate request to add a new sheet with the generated name and frozen header row.  
    - Configuration: POST request with JSON body to create sheet, includes frozen row count =1.  
    - Authentication: Google Sheets OAuth2.  
    - Inputs: Spreadsheet ID and prepared sheet name.  
    - Outputs: Response containing sheetId for next steps.  
    - Edge Cases: Sheet name conflicts, API limits, insufficient permissions.  

  - `Write Headers (A1:I1)`  
    - Type: HTTP Request (Google Sheets API)  
    - Role: Writes column headers ("OrderId", "Date", "Customer Name", etc.) into the first row and applies cell formatting (date format for Date column), data validation for Status column, and conditional formatting rules for status colors.  
    - Configuration: BatchUpdate POST with multiple requests for updateCells, repeatCell, setDataValidation, addConditionalFormatRule.  
    - Inputs: Spreadsheet ID and sheetId from sheet creation response.  
    - Outputs: Confirmation of formatting applied.  
    - Edge Cases: API errors, formatting conflicts, malformed requests.  

---

#### 2.5 Data Preparation

- **Overview:**  
  Formats the order data payload to match the Google Sheets schema, including date conversion to serial number, concatenation of line items, and payment mode normalization.

- **Nodes Involved:**  
  - `Google Sheets Row values`  
  - `Google Sheets Row values existing`

- **Node Details:**  
  - Both nodes are of type Set and prepare values for appending to sheets. They differ in naming to distinguish appending to new vs existing sheets but perform equivalent functions.

  - Key data transformations:  
    - Date: Converts order update timestamp to Google Sheets date serial number format (accounting for timezone Asia/Kolkata).  
    - OrderName: Joins line_items into a multiline string with quantity annotations (e.g., "T-shirt x2").  
    - PaymentMode: Maps financial_status to "Prepaid", "COD", "voided", or original string.  
  - Inputs: Raw order JSON from webhook or previous node.  
  - Outputs: Structured object with keys matching Google Sheets columns.  
  - Edge Cases: Missing or malformed line_items, time parsing errors, unknown payment statuses.  

---

#### 2.6 Data Append

- **Overview:**  
  Appends the prepared order data into the appropriate monthly sheet tab, either newly created or existing.

- **Nodes Involved:**  
  - `Append to Orders Sheet`  
  - `Append to Existing Orders Sheet`

- **Node Details:**  
  - Type: Google Sheets node (append operation)  
  - Role: Inserts one row of order data into the monthly tab.  
  - Configuration: Defines explicit column mapping to sheet columns ("Date", "Order", "Status", etc.), sets operation to append.  
  - Inputs: Output of data preparation nodes, spreadsheet ID, and monthly sheet name.  
  - Outputs: Append operation success confirmation.  
  - Authentication: Google Sheets OAuth2 credentials.  
  - Edge Cases: API quota limits, sheet protection preventing append, mismatched column schema, network errors.  

---

### 3. Summary Table

| Node Name                   | Node Type             | Functional Role                         | Input Node(s)                     | Output Node(s)                          | Sticky Note                                                                                                       |
|-----------------------------|-----------------------|---------------------------------------|----------------------------------|---------------------------------------|-------------------------------------------------------------------------------------------------------------------|
| Order created               | Webhook               | Receive Shopify order webhook          | None                             | Config (set spreadsheetId)             | Input from Shopify webhook URL (Settings → Notifications → Webhooks). Events: Order creation, update, fulfillment. |
| Config (set spreadsheetId)  | Set                   | Set spreadsheet ID for API calls       | Order created                   | Get Order Sheets metadata              | Update your Google spreadsheetId (docs.google.com/spreadsheets/d/<id>/)                                          |
| Get Order Sheets metadata   | HTTP Request (Google Sheets) | Fetch existing sheet metadata          | Config (set spreadsheetId)       | Generate Sheet Name                    | Fetch the existing sheet details                                                                                  |
| Generate Sheet Name         | Set                   | Generate current month sheet name      | Get Order Sheets metadata        | If                                    |                                                                                                                   |
| If                         | If                    | Check if month sheet exists             | Generate Sheet Name              | Google Sheets Row values existing, Set Sheet Starting row col |                                                                                                                   |
| Google Sheets Row values existing | Set            | Prepare order data for existing sheet | If (true branch)                 | Append to Existing Orders Sheet        |                                                                                                                   |
| Set Sheet Starting row col  | Set                   | Prepare A1 notation for new sheet range | If (false branch)               | Create Month Sheet                     |                                                                                                                   |
| Create Month Sheet          | HTTP Request (Google Sheets) | Create new monthly sheet tab           | Set Sheet Starting row col       | Write Headers (A1:I1)                  |                                                                                                                   |
| Write Headers (A1:I1)       | HTTP Request (Google Sheets) | Write headers and apply formatting    | Create Month Sheet               | Google Sheets Row values               |                                                                                                                   |
| Google Sheets Row values    | Set                   | Prepare order data for new sheet        | Write Headers (A1:I1)            | Append to Orders Sheet                 |                                                                                                                   |
| Append to Existing Orders Sheet | Google Sheets       | Append order to existing monthly sheet  | Google Sheets Row values existing |                                   |                                                                                                                   |
| Append to Orders Sheet      | Google Sheets          | Append order to newly created monthly sheet | Google Sheets Row values        |                                   |                                                                                                                   |
| Sticky Note2                | Sticky Note           | Documentation: Input details             | None                           | None                                  | Input from Shopify webhook URL (Settings → Notifications → Webhooks). Events: Order creation, update, fulfillment. |
| Sticky Note1                | Sticky Note           | Documentation: Update spreadsheetId     | None                           | None                                  | Update your Google spreadsheetId (docs.google.com/spreadsheets/d/<id>/)                                          |
| Sticky Note3                | Sticky Note           | Documentation: Sheet metadata fetch visual | None                         | None                                  | Fetch the existing sheet details                                                                                  |
| Sticky Note4                | Sticky Note           | Documentation: Output description and example | None                        | None                                  | Auto-create monthly sub-sheet, append orders, status column with options and coloring                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - Name: `Order created`  
   - HTTP Method: POST  
   - Path: Unique identifier (e.g., "0ad91e1c-4e03-499c-9376-c096c30a354d")  
   - Purpose: Receive order event data from Shopify.  
   - No authentication configured here; secure via external means if needed.  

2. **Create Set Node for Spreadsheet ID**  
   - Type: Set  
   - Name: `Config (set spreadsheetId)`  
   - Add a string field: `spreadsheetId` with your Google Sheets spreadsheet ID (from URL).  
   - Connect from `Order created`.  

3. **Create HTTP Request Node to Fetch Sheet Metadata**  
   - Type: HTTP Request  
   - Name: `Get Order Sheets metadata`  
   - Method: GET  
   - URL: `https://sheets.googleapis.com/v4/spreadsheets/{{$json.spreadsheetId}}?fields=sheets.properties.title`  
   - Authentication: Select Google Sheets OAuth2 credentials.  
   - Connect from `Config (set spreadsheetId)`.  

4. **Create Set Node to Generate Monthly Sheet Name**  
   - Type: Set  
   - Name: `Generate Sheet Name`  
   - Add a string field `Sheet name` with expression:  
     ```javascript
     (() => {
       const tz = 'Asia/Kolkata';
       const d = new Date();
       const MONTH = d.toLocaleString('en-US', { month: 'long', timeZone: tz }).toUpperCase();
       const YY = String(d.getFullYear()).slice(-2);
       const raw = `${MONTH} ORDERS'${YY}`;
       return raw.split(/(?:\r?\n|\\n|[\u2028\u2029])/)[0].trim();
     })()
     ```
   - Connect from `Get Order Sheets metadata`.  

5. **Create If Node to Check Sheet Existence**  
   - Type: If  
   - Name: `If`  
   - Condition (Expression):  
     ```javascript
     (() => {
       const target = String($node["Generate Sheet Name"].json["Sheet name"] ?? "").replace(/[\r\n\u2028\u2029]/g, "").replace(/\\n/g, "").trim();
       const titles = ($node["Get Order Sheets metadata"].json.sheets || []).map(s => String(s.properties?.title ?? "").replace(/[\r\n\u2028\u2029]/g, "").replace(/\\n/g, "").trim());
       return titles.includes(target);
     })()
     ```  
   - Connect from `Generate Sheet Name`.  

6. **Create Set Node for Preparing Data for Existing Sheet**  
   - Type: Set  
   - Name: `Google Sheets Row values existing`  
   - Fields:  
     - Date (number): Convert order updated_at timestamp to Google Sheets serial date number (see detailed expression below).  
     - OrderName (string): Concatenate line items with quantities.  
     - PaymentMode (string): Map financial_status to "Prepaid", "COD", or original.  
   - Connect from `If` node’s True branch.  

7. **Create Google Sheets Node to Append Rows to Existing Sheet**  
   - Type: Google Sheets  
   - Name: `Append to Existing Orders Sheet`  
   - Operation: Append  
   - Document ID: Use `{{$node["Config (set spreadsheetId)"].json.spreadsheetId}}`  
   - Sheet Name: Use `{{$node["Generate Sheet Name"].json["Sheet name"].replace(/(?:\r?\n|\\n)+$/,'').trimEnd()}}`  
   - Columns Mapping: Map fields such as Date, Order, Status (default "Not Shipped"), OrderId, Order Value, Customer Name.  
   - Connect from `Google Sheets Row values existing`.  
   - Credentials: Google Sheets OAuth2.  

8. **Create Set Node for Preparing Sheet Range for New Sheet Creation**  
   - Type: Set  
   - Name: `Set Sheet Starting row col`  
   - Single string field `Sheet Starting Row Col` with expression:  
     ```javascript
     (() => {
       const raw = String($json['Sheet name'] ?? '');
       const clean = raw.replace(/[\r\n\u2028\u2029]/g, '').replace(/\\n/g, '').trim();
       const quoted = `'${clean.replace(/'/g, "''")}'`;
       return `${quoted}!A1`;
     })()
     ```  
   - Connect from `If` node’s False branch.  

9. **Create HTTP Request Node to Create New Sheet**  
   - Type: HTTP Request  
   - Name: `Create Month Sheet`  
   - Method: POST  
   - URL: `https://sheets.googleapis.com/v4/spreadsheets/{{$node["Config (set spreadsheetId)"].json.spreadsheetId}}:batchUpdate`  
   - Body: JSON to addSheet with title from `Generate Sheet Name` and freeze first row.  
   - Authentication: Google Sheets OAuth2 credentials.  
   - Connect from `Set Sheet Starting row col`.  

10. **Create HTTP Request Node to Write Headers and Formatting**  
    - Type: HTTP Request  
    - Name: `Write Headers (A1:I1)`  
    - Method: POST  
    - URL: Same as for `Create Month Sheet`.  
    - Body: BatchUpdate JSON to write headers, apply date format to Date column, set data validation for Status column with allowed values, and add conditional formatting rules for status colors.  
    - Connect from `Create Month Sheet`.  

11. **Create Set Node for Preparing Data for New Sheet**  
    - Type: Set  
    - Name: `Google Sheets Row values`  
    - Same fields and expressions as `Google Sheets Row values existing`.  
    - Connect from `Write Headers (A1:I1)`.  

12. **Create Google Sheets Node to Append Rows to New Sheet**  
    - Type: Google Sheets  
    - Name: `Append to Orders Sheet`  
    - Same configuration as `Append to Existing Orders Sheet`.  
    - Connect from `Google Sheets Row values`.  

13. **Connect Workflow**  
    - Connect nodes according to the logical flow described above.  
    - Ensure credentials for Google Sheets OAuth2 are configured and authorized for all Google Sheets API nodes.  

---

### 5. General Notes & Resources

| Note Content                                                                                                 | Context or Link                                                                                         |
|--------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| Input webhook URL must be registered in Shopify under Settings → Notifications → Webhooks                    | See Sticky Note2 with sample Shopify order webhook screenshot                                         |
| Update the `spreadsheetId` in the `Config (set spreadsheetId)` node to your Google Sheets document ID        | Google Sheet URL format: docs.google.com/spreadsheets/d/<spreadsheetId>/                              |
| The workflow auto-creates monthly tabs named like "JUNE ORDERS'24" and applies frozen header row and formatting | See Sticky Note4 for example output image                                                             |
| Status column options with conditional coloring: Not Shipped, Pickup Scheduled, Shipped, InTransit, Delivered, Cancelled | Conditional formatting applied via Google Sheets API batchUpdate                                      |
| Timezone used throughout is Asia/Kolkata for date conversions                                                | Change in Set nodes if different timezone is needed                                                   |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, a workflow automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.