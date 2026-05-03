Process Sales CSVs into Invoices with Data Tables and Email Notifications

https://n8nworkflows.xyz/workflows/process-sales-csvs-into-invoices-with-data-tables-and-email-notifications-10274


# Process Sales CSVs into Invoices with Data Tables and Email Notifications

### 1. Workflow Overview

This workflow, titled **Process Sales CSVs into Invoices with Data Tables and Email Notifications**, automates the transformation of sales data submitted as CSV files into enriched, validated invoices stored within n8n's native Data tables. It also generates email notifications confirming the orders. The workflow is designed for e-commerce, internal accounting, ERP data imports, or as an API endpoint for sales systems.

The logic is divided into the following main blocks:

- **1.1 Input Reception and CSV Extraction:** Receives CSV data via webhook, detects input type (file upload or raw text), and extracts CSV content.
- **1.2 CSV Parsing and Validation:** Parses CSV content, validates required columns and field formats, and rejects invalid data.
- **1.3 Product Data Enrichment:** Enriches each order row with product details from the `Products` Data table and calculates line item totals including tax.
- **1.4 Invoice Aggregation and Totals Calculation:** Groups enriched order lines by customer email, calculates invoice totals, and generates invoice IDs.
- **1.5 Duplicate Detection:** Checks for duplicate invoices based on customer email and order date, blocking duplicates with a 409 response.
- **1.6 Invoice Storage:** Inserts validated invoices into the `Invoices` Data table.
- **1.7 Email Preparation and Notification:** Prepares email contents for each invoice and simulates sending notifications.
- **1.8 Final Response Construction:** Merges processing results and returns a structured JSON response to the webhook caller.
- **Documentation and Testing Notes:** Sticky notes provide detailed documentation and cURL test examples.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and CSV Extraction

- **Overview:** Receives incoming POST requests with sales CSV data, either as a file upload or raw CSV text, then extracts CSV content accordingly.
- **Nodes Involved:** Receive Sales CSV, Check Upload Type, Extract from File, Extract CSV Text

##### Node: Receive Sales CSV
- **Type:** Webhook (POST)
- **Role:** Entry point accepting CSV data at `/process-sales`.
- **Config:** Expects binary data under property `file` if file upload; response mode delegates response to downstream nodes.
- **Connections:** Outputs to Check Upload Type.
- **Edge Cases:** Missing or malformed payload; invalid HTTP method.

##### Node: Check Upload Type
- **Type:** If
- **Role:** Determines if input is a file upload or raw CSV text.
- **Config:** Checks for presence of binary property `file0` in webhook data.
- **Outputs:**  
  - True branch: Binary file present → Extract from File node  
  - False branch: No file → Extract CSV Text node
- **Edge Cases:** Incorrect input format; empty payload.

##### Node: Extract from File
- **Type:** Extract From File
- **Role:** Extracts textual CSV content from uploaded binary file.
- **Config:** Reads text content from binary property `file0`.
- **Connections:** Outputs CSV text to Parse & Validate CSV.
- **Edge Cases:** Binary file corrupt or unreadable.

##### Node: Extract CSV Text
- **Type:** Extract From File
- **Role:** Extracts text directly from binary property `file` (named differently than `file0`).
- **Config:** Reads text content.
- **Connections:** Outputs CSV text to Parse & Validate CSV.
- **Edge Cases:** Text content invalid or empty.

---

#### 1.2 CSV Parsing and Validation

- **Overview:** Parses CSV lines, validates required columns (`sku`, `quantity`, `customer_email`, `order_date`), and field formats (email, positive quantity, date format). Throws error with detailed info for invalid rows.
- **Nodes Involved:** Parse & Validate CSV, Return Validation Error

##### Node: Parse & Validate CSV
- **Type:** Code (JavaScript)
- **Role:** Parses CSV string, normalizes CSV delimiters, checks mandatory columns, validates each row's data.
- **Config:** Custom JS code that:
  - Cleans up Bash-style inputs.
  - Auto-detects delimiter (`;` or `,`).
  - Validates email with regex.
  - Checks quantity is positive integer.
  - Checks date format `YYYY-MM-DD`.
  - Collects row-level errors.
  - Returns array of valid rows or throws error containing validation details.
- **Connections:**  
  - On success: Load Product Catalog  
  - On error: Return Validation Error  
- **Edge Cases:** Missing columns, empty rows, invalid email/quantity/date, partial failures.
- **Error Handling:** `onError` set to continue and output error node.

##### Node: Return Validation Error
- **Type:** Respond to Webhook
- **Role:** Returns HTTP 400 with JSON describing CSV validation failure.
- **Config:** Response code 400, Content-Type application/json, includes error message.
- **Connections:** Receives error from Parse & Validate CSV.
- **Edge Cases:** Triggered when CSV validation fails.

---

#### 1.3 Product Data Enrichment

- **Overview:** Loads product catalog from `Products` Data table, enriches each order line by joining product details (name, price, tax rate), calculates line totals, tax, and total with tax. Throws error if SKU missing from catalog.
- **Nodes Involved:** Load Product Catalog, Enrich with Product Data

##### Node: Load Product Catalog
- **Type:** Data Table
- **Role:** Fetches entire `Products` Data table (fields: sku, name, price, tax_rate).
- **Config:** `get` operation, return all rows.
- **Connections:** Outputs product list to Enrich with Product Data.
- **Execute Once:** true (only fetches once per workflow run).
- **Edge Cases:** Empty product catalog, Data table access issues.

##### Node: Enrich with Product Data
- **Type:** Code (JavaScript)
- **Role:** Joins order rows with product catalog by SKU, calculates line totals and tax amounts, formats numeric results.
- **Config:**  
  - Creates SKU lookup map from product catalog.  
  - Iterates parsed CSV rows, uppercases SKU for matching.  
  - Throws error if SKU missing.  
  - Calculates line total = price * quantity, tax_amount = line_total * tax_rate, total_with_tax = line_total + tax_amount.  
- **Connections:** Outputs enriched line items to Calculate Invoice Totals.
- **Edge Cases:** Missing SKUs, numeric conversion errors.

---

#### 1.4 Invoice Aggregation and Totals Calculation

- **Overview:** Groups enriched order lines by customer email, aggregates line totals and taxes into invoice subtotals, generates unique invoice IDs, and formats invoice objects.
- **Nodes Involved:** Calculate Invoice Totals

##### Node: Calculate Invoice Totals
- **Type:** Code (JavaScript)
- **Role:**  
  - Groups items by customer email.  
  - Aggregates subtotal, total tax, grand total.  
  - Creates line item summaries per invoice.  
  - Generates invoice IDs using timestamp and index.  
  - Sets invoice status to `pending`.  
- **Connections:** Outputs invoices to Check for Duplicates.
- **Edge Cases:** Large numbers rounding issues, empty input.

---

#### 1.5 Duplicate Detection

- **Overview:** Checks aggregated invoices against existing orders for duplicates based on customer email and order date. Returns valid invoices or duplicates error.
- **Nodes Involved:** Check for Duplicates, Has Valid Invoices?, Return Duplicate Error

##### Node: Check for Duplicates
- **Type:** Code (JavaScript)
- **Role:**  
  - Examines invoices against simulated existing orders array.  
  - Separates duplicates and valid invoices.  
  - Stores duplicates in a global variable for error reporting.  
  - Returns valid invoices or a "no valid invoices" flag with duplicates.
- **Connections:** Outputs to Has Valid Invoices? (If node).
- **Edge Cases:** In real production, would query actual database or Data table to detect duplicates.

##### Node: Has Valid Invoices?
- **Type:** If
- **Role:** Checks if any valid invoices remain after duplicate filtering.
- **Config:** Condition: `$json.no_valid_invoices !== true`
- **Outputs:**  
  - True branch: Valid invoices → Insert row (save invoices)  
  - False branch: All duplicates → Return Duplicate Error
- **Edge Cases:** No invoices at all, or all duplicates.

##### Node: Return Duplicate Error
- **Type:** Respond to Webhook
- **Role:** Responds with HTTP 409 Conflict and JSON describing duplicate orders, no new invoices created.
- **Config:** Response code 409, Content-Type application/json.
- **Connections:** Terminal node if all invoices are duplicates.

---

#### 1.6 Invoice Storage

- **Overview:** Inserts each validated invoice as a new row in the `Invoices` Data table.
- **Nodes Involved:** Insert row, Aggregate

##### Node: Insert row
- **Type:** Data Table
- **Role:** Inserts validated invoice data into `Invoices` Data table.
- **Config:**  
  - Defines mapping for columns: invoice_id, customer_email, order_date, subtotal, total_tax, grand_total.  
  - Uses `defineBelow` mapping mode with direct JSON references.  
- **Connections:** Outputs inserted rows to Aggregate node.
- **Edge Cases:** Data table permission or schema mismatch.

##### Node: Aggregate
- **Type:** Aggregate
- **Role:** Aggregates all inserted invoice rows for downstream processing.
- **Config:** Default aggregation of all input items.
- **Connections:** Outputs aggregated invoices to Prepare Email Notifications.
- **Edge Cases:** Empty input.

---

#### 1.7 Email Preparation and Notification

- **Overview:** Prepares personalized email content per invoice and simulates sending emails. (In production, would connect to Gmail or other email node.)
- **Nodes Involved:** Prepare Email Notifications, Email Output Preview, Merge Results, Email Info (sticky note)

##### Node: Prepare Email Notifications
- **Type:** Code (JavaScript)
- **Role:**  
  - Flattens input invoice data.  
  - Creates email subject and body text including invoice details.  
  - Returns email objects with `to`, `subject`, `body`.
- **Connections:** Outputs to Email Output Preview.
- **Edge Cases:** Missing invoice data, formatting errors.

##### Node: Email Output Preview
- **Type:** Set
- **Role:** Extracts and sets email fields `to`, `subject`, and `body` explicitly for preview or sending.
- **Connections:** Outputs to Merge Results.
- **Edge Cases:** None specific.

##### Node: Merge Results
- **Type:** Code (JavaScript)
- **Role:** Combines invoice data and email notification results into one JSON response, includes success flag and timestamp.
- **Connections:** Outputs to Return Success Response.
- **Edge Cases:** Mismatched inputs; empty arrays.

##### Node: Email Info (Sticky Note)
- **Role:** Documentation note explaining that in production, the email node would be Gmail or similar, but currently emails are simulated.
- **Sticky Note Content:**  
  "## 📧 Email Node  
  In production, connect:  
  - Gmail  
  For now, emails are simulated."

---

#### 1.8 Final Response Construction

- **Overview:** Returns HTTP 200 success response with JSON containing invoice and email notification data.
- **Nodes Involved:** Return Success Response

##### Node: Return Success Response
- **Type:** Respond to Webhook
- **Role:** Sends back JSON response with HTTP 200, including success message, processed count, invoice data, and email notification info.
- **Config:** Sets header Content-Type application/json.
- **Connections:** Terminal node for successful workflow completion.

---

#### Documentation and Testing Notes

- **Nodes Involved:** Documentation (sticky note), Sticky Note (test instructions)
- **Content Highlights:**  
  - Detailed description of workflow features, use cases, setup instructions.  
  - cURL examples for valid and invalid CSV POST requests, with expected HTTP responses and JSON output.  
  - Instructions for creating `Products` and `Invoices` Data tables.  
  - Example JSON response for success and failure.

---

### 3. Summary Table

| Node Name              | Node Type             | Functional Role                              | Input Node(s)              | Output Node(s)                 | Sticky Note                                                                                                                              |
|------------------------|-----------------------|----------------------------------------------|----------------------------|-------------------------------|------------------------------------------------------------------------------------------------------------------------------------------|
| Documentation          | Sticky Note           | Workflow overview and documentation          |                            |                               | See documentation content describing Smart Sales Invoice Processor features, setup, and outputs.                                         |
| Receive Sales CSV      | Webhook               | Entry point receiving sales CSV POST data    |                            | Check Upload Type              |                                                                                                                                          |
| Check Upload Type      | If                    | Detect if input is file upload or raw text   | Receive Sales CSV           | Extract from File, Extract CSV Text |                                                                                                                                          |
| Extract from File      | Extract From File     | Extract CSV text from file upload             | Check Upload Type           | Parse & Validate CSV           |                                                                                                                                          |
| Extract CSV Text       | Extract From File     | Extract CSV text from raw input               | Check Upload Type           | Parse & Validate CSV           |                                                                                                                                          |
| Parse & Validate CSV   | Code                  | Parse CSV, validate structure and data        | Extract from File, Extract CSV Text | Load Product Catalog, Return Validation Error |                                                                                                                                          |
| Return Validation Error| Respond to Webhook    | Return 400 error for invalid CSV               | Parse & Validate CSV (error) |                               |                                                                                                                                          |
| Load Product Catalog   | Data Table            | Load product catalog from Data table          | Parse & Validate CSV        | Enrich with Product Data       |                                                                                                                                          |
| Enrich with Product Data| Code                  | Join order lines with product data, calculate totals | Load Product Catalog        | Calculate Invoice Totals       |                                                                                                                                          |
| Calculate Invoice Totals| Code                  | Aggregate orders by customer, compute invoice totals | Enrich with Product Data    | Check for Duplicates           |                                                                                                                                          |
| Check for Duplicates   | Code                  | Detect duplicate invoices                      | Calculate Invoice Totals    | Has Valid Invoices?            |                                                                                                                                          |
| Has Valid Invoices?    | If                    | Branch based on presence of valid invoices    | Check for Duplicates        | Insert row, Return Duplicate Error |                                                                                                                                          |
| Return Duplicate Error | Respond to Webhook    | Return 409 error if all invoices are duplicates | Has Valid Invoices?         |                               |                                                                                                                                          |
| Insert row             | Data Table            | Insert validated invoices into Invoices table | Has Valid Invoices? (true)  | Aggregate                     |                                                                                                                                          |
| Aggregate              | Aggregate             | Aggregate inserted invoice rows                | Insert row                 | Prepare Email Notifications    |                                                                                                                                          |
| Prepare Email Notifications| Code                | Prepare personalized email content per invoice | Aggregate                  | Email Output Preview           |                                                                                                                                          |
| Email Output Preview   | Set                   | Format email fields for preview or sending    | Prepare Email Notifications | Merge Results                 |                                                                                                                                          |
| Merge Results          | Code                  | Combine invoices and email results into final response | Email Output Preview       | Return Success Response        |                                                                                                                                          |
| Return Success Response| Respond to Webhook    | Send HTTP 200 with JSON success response       | Merge Results              |                               |                                                                                                                                          |
| Email Info             | Sticky Note           | Notes on email node usage and simulation       |                            |                               | "In production, connect: Gmail. For now, emails are simulated."                                                                         |
| Sticky Note            | Sticky Note           | cURL test examples and usage instructions      |                            |                               | Detailed cURL examples for valid/invalid CSV POST requests and expected responses.                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Data Tables:**
   - Create a `Products` Data table with columns: `sku` (string), `name` (string), `price` (number), `tax_rate` (number).
   - Create an `Invoices` Data table with columns: `invoice_id` (string), `customer_email` (string), `order_date` (dateTime), `subtotal` (number), `total_tax` (number), `grand_total` (number).

2. **Create Webhook Node:**
   - Add a Webhook node named `Receive Sales CSV`.
   - Set HTTP method to POST.
   - Set path to `/process-sales`.
   - Set binary property name to `file`.
   - Set response mode to "response node".

3. **Add If Node (Check Upload Type):**
   - Name: `Check Upload Type`.
   - Condition: Check if binary property `file0` exists and is not empty on `Receive Sales CSV` output.
   - True output → `Extract from File`.
   - False output → `Extract CSV Text`.

4. **Add Extract From File Nodes:**
   - `Extract from File`: Extract text from binary property `file0`.
   - `Extract CSV Text`: Extract text from binary property `file`.
   - Both output to `Parse & Validate CSV`.

5. **Add Code Node (Parse & Validate CSV):**
   - Name: `Parse & Validate CSV`.
   - Paste the JavaScript code that:
     - Cleans CSV input.
     - Detects delimiter.
     - Checks for required columns (`sku`, `quantity`, `customer_email`, `order_date`).
     - Validates each row's fields.
     - Throws error on validation failure.
   - On error, connect to `Return Validation Error`.
   - On success, connect to `Load Product Catalog`.

6. **Add Respond to Webhook Node (Return Validation Error):**
   - HTTP code 400.
   - JSON response with `success: false`, message, and error details.

7. **Add Data Table Node (Load Product Catalog):**
   - Operation: get all rows.
   - Point to the `Products` Data table.
   - Set execute once = true.
   - Connect output to `Enrich with Product Data`.

8. **Add Code Node (Enrich with Product Data):**
   - Use code that:
     - Builds SKU map from product catalog.
     - Joins parsed CSV rows by SKU.
     - Calculates line totals, tax, and totals with tax.
     - Throws error if SKU missing.
   - Output to `Calculate Invoice Totals`.

9. **Add Code Node (Calculate Invoice Totals):**
   - Group enriched items by customer email.
   - Sum subtotal, tax, grand total.
   - Generate unique invoice IDs.
   - Return array of invoices.
   - Connect to `Check for Duplicates`.

10. **Add Code Node (Check for Duplicates):**
    - Simulate or query existing orders.
    - Separate duplicates and valid invoices.
    - Store duplicates in global variable.
    - Return valid invoices or flag no valid invoices.
    - Connect to `Has Valid Invoices?`.

11. **Add If Node (Has Valid Invoices?):**
    - Condition: `$json.no_valid_invoices !== true`.
    - True branch → `Insert row`.
    - False branch → `Return Duplicate Error`.

12. **Add Respond to Webhook Node (Return Duplicate Error):**
    - HTTP 409.
    - JSON response indicating duplicate orders and no new invoices.

13. **Add Data Table Node (Insert row):**
    - Operation: insert.
    - Target `Invoices` Data table.
    - Map columns: invoice_id, customer_email, order_date, subtotal, total_tax, grand_total.
    - Connect to `Aggregate`.

14. **Add Aggregate Node:**
    - Aggregate all inserted invoice rows.
    - Connect to `Prepare Email Notifications`.

15. **Add Code Node (Prepare Email Notifications):**
    - Flatten invoice data.
    - Build email subject and body per invoice.
    - Return array of email objects.
    - Connect to `Email Output Preview`.

16. **Add Set Node (Email Output Preview):**
    - Map fields `to`, `subject`, and `body` from input JSON.
    - Connect to `Merge Results`.

17. **Add Code Node (Merge Results):**
    - Combine invoice data and email results.
    - Add success flag and timestamp.
    - Connect to `Return Success Response`.

18. **Add Respond to Webhook Node (Return Success Response):**
    - HTTP 200.
    - JSON response containing invoices and email notifications.

19. **Add Sticky Notes for Documentation and Email Info:**
    - Create sticky notes with workflow overview, cURL test instructions, and email node notes.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | Context or Link                                                                                     |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Workflow uses native n8n Data tables (`Products` and `Invoices`) for storage and enrichment, requiring no external databases or integrations.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Setup instructions section                                                                          |
| cURL examples demonstrate how to POST valid and invalid CSV data to the webhook, illustrating expected HTTP 200 success and HTTP 400 validation error responses.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Sticky Note with test instructions                                                                 |
| Email sending is simulated in this workflow; replace the simulation with a Gmail or SMTP node in production environments for live email notifications.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Email Info sticky note                                                                              |
| Invoice IDs are generated using a timestamp and index pattern: `INV-[timestamp]-[index]`. This ensures uniqueness per workflow run but may require adjustment in production for distributed systems.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Calculate Invoice Totals node                                                                         |
| Duplicate detection is currently simulated with an empty array; for production, integrate with real database queries or Data table lookups to prevent duplicate invoices.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Check for Duplicates node                                                                            |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, adhering strictly to content policies. It contains no illegal, offensive, or protected content and operates exclusively on legal and public data.