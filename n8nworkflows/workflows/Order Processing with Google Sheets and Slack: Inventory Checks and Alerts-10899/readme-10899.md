Order Processing with Google Sheets and Slack: Inventory Checks and Alerts

https://n8nworkflows.xyz/workflows/order-processing-with-google-sheets-and-slack--inventory-checks-and-alerts-10899


# Order Processing with Google Sheets and Slack: Inventory Checks and Alerts

### 1. Workflow Overview

This workflow automates order processing by integrating with Google Sheets for inventory checks and Slack for real-time notifications. It is designed for e-commerce or retail operations that require automated verification of stock availability before confirming orders, thus reducing stock-outs and operational costs.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receives incoming order data via a webhook and verifies the order payload.
- **1.2 Order Data Preparation:** Extracts and structures relevant order information for downstream processing.
- **1.3 Inventory Verification:** Queries Google Sheets to check current stock levels against ordered quantities.
- **1.4 Stock Decision Logic:** Determines if stock levels are sufficient to fulfill the order.
- **1.5 Notifications:** Sends Slack alerts to notify warehouse or management about order success or stock shortages.
- **1.6 Logging:** Records order processing outcomes into a Google Sheet for tracking and auditing.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
Receives incoming order requests via an HTTP POST webhook and validates the payload to ensure required fields are present.

**Nodes Involved:**  
- Order Webhook  
- Check Order

**Node Details:**

- **Order Webhook**  
  - Type: Webhook  
  - Role: Receives HTTP POST requests at path `/ecom-order` containing order data.  
  - Configuration:  
    - HTTP Method: POST  
    - Response: Returns all entries processed, responding with the last node’s output.  
  - Inputs: External API call  
  - Outputs: Passes JSON payload to "Check Order"  
  - Edge Cases: Invalid or malformed HTTP requests; missing payload fields.

- **Check Order**  
  - Type: Function  
  - Role: Validates the order payload structure ensuring presence of order ID, customer, and items.  
  - Configuration: JavaScript function checks for mandatory fields; throws error if validation fails.  
  - Inputs: JSON from webhook  
  - Outputs: Validated order JSON or error  
  - Edge Cases: Throws error on invalid payload; this stops workflow execution for bad input.

---

#### 1.2 Order Data Preparation

**Overview:**  
Processes validated order data to extract structured order details including customer info, item SKUs, quantities, and pricing.

**Nodes Involved:**  
- Request Info  
- Item and quantity

**Node Details:**

- **Request Info**  
  - Type: Function  
  - Role: Converts raw JSON into structured order object with sanitized customer and items data.  
  - Configuration: Extracts order ID, customer name/email, array of items with sku, qty, name, price, and total amount.  
  - Inputs: Output of "Check Order"  
  - Outputs: Structured order object  
  - Edge Cases: Missing optional fields handled by defaults (empty string or 0).

- **Item and quantity**  
  - Type: Function  
  - Role: Maps order items to a simplified list containing only SKU and quantity for inventory lookup.  
  - Configuration: Iterates over items array and outputs objects with sku and qty only.  
  - Inputs: Structured order from "Request Info"  
  - Outputs: Multiple items with sku and qty for parallel inventory queries  
  - Edge Cases: Empty items array should not occur due to prior validation.

---

#### 1.3 Inventory Verification

**Overview:**  
Looks up the current stock for each SKU in Google Sheets and merges inventory data with order quantities.

**Nodes Involved:**  
- Get Inventories  
- Mapping data

**Node Details:**

- **Get Inventories**  
  - Type: Google Sheets  
  - Role: Performs a lookup on the "Inventories" sheet by SKU to retrieve stock quantities.  
  - Configuration:  
    - Sheet ID specified (Google Sheets file containing inventory data)  
    - Range: `Inventories!A:B` (columns A and B expected to contain SKU and stock)  
    - Operation: Lookup by SKU column matching order item SKU  
  - Inputs: Each order item (sku, qty) individually  
  - Outputs: Inventory data per SKU including stock levels  
  - Credentials: Google Sheets OAuth2 required  
  - Edge Cases: SKU not found returns empty; must handle missing inventory gracefully.

- **Mapping data**  
  - Type: Merge  
  - Role: Combines order quantities and inventory stock into a single data structure by matching on SKU.  
  - Configuration: Mode "combine" with matching on SKU field  
  - Inputs: Receives order item list and inventory lookup results  
  - Outputs: Combined data with sku, qty (order), and stock (inventory) fields  
  - Edge Cases: Mismatched SKUs results in missing stock info; handled downstream in stock check.

---

#### 1.4 Stock Decision Logic

**Overview:**  
Evaluates if stock is sufficient for all items and routes workflow accordingly.

**Nodes Involved:**  
- enough stock?

**Node Details:**

- **enough stock?**  
  - Type: If  
  - Role: Checks if every ordered item’s quantity is less than or equal to available stock.  
  - Configuration: Boolean condition using expression:  
    `{{$items().every(item => item.json.qty <= item.json.stock)}} === true`  
  - Inputs: Combined order and inventory data from "Mapping data"  
  - Outputs:  
    - True branch: stock sufficient  
    - False branch: stock shortage  
  - Edge Cases: Null or undefined stock values may lead to false negative; expression must handle missing data carefully.

---

#### 1.5 Notifications

**Overview:**  
Sends Slack notifications about order processing status: success with invoice, or stock shortage alert.

**Nodes Involved:**  
- Create Invoice  
- Slack notification (OK)  
- Slack notification (Stock shortage)

**Node Details:**

- **Create Invoice**  
  - Type: Function  
  - Role: Generates a simple invoice object with unique invoice ID and order details.  
  - Configuration: Uses timestamp for invoice ID, copies order and item data from upstream nodes.  
  - Inputs: Stock sufficient branch output from "enough stock?"  
  - Outputs: Invoice JSON passed to Slack notification (OK)  
  - Edge Cases: Timestamp collisions unlikely; data must be consistent.

- **Slack notification (OK)**  
  - Type: Slack  
  - Role: Posts a success message to Slack channel `#warehouse` with order and invoice details.  
  - Configuration:  
    - Uses OAuth2 credentials  
    - Message template includes orderId, invoiceId, amount, customer name, and date  
  - Inputs: Output of "Create Invoice"  
  - Outputs: Passes to success logging node  
  - Edge Cases: Slack API rate limits or authentication failures.

- **Slack notification (Stock shortage)**  
  - Type: Slack  
  - Role: Posts a warning message to `#warehouse` channel indicating out-of-stock items, including SKU and quantities.  
  - Configuration:  
    - Uses OAuth2 credentials  
    - Message dynamically references order ID, SKU, requested quantity, and available stock  
  - Inputs: Stock shortage branch output from "enough stock?"  
  - Outputs: Passes to stock shortage logging node  
  - Edge Cases: Same as above (Slack API issues).

---

#### 1.6 Logging

**Overview:**  
Logs order processing results with status to Google Sheets for record keeping.

**Nodes Involved:**  
- create data logs (success)  
- create data logs (Stock shortage)  
- Save logs

**Node Details:**

- **create data logs (success)**  
  - Type: Function  
  - Role: Creates a log entry JSON with order ID, amount, status "Success", and timestamp.  
  - Inputs: Output of "Slack notification (OK)"  
  - Outputs: Log data to be saved  
  - Edge Cases: Data consistency critical for audit.

- **create data logs (Stock shortage)**  
  - Type: Function  
  - Role: Creates a log entry JSON with order ID, amount, status "Out of Stock", and timestamp.  
  - Inputs: Output of "Slack notification (Stock shortage)"  
  - Outputs: Log data to be saved  
  - Edge Cases: Same as above.

- **Save logs**  
  - Type: Google Sheets  
  - Role: Appends log entries to the "OrderLogs" sheet in the same Google Sheets file used for inventory.  
  - Configuration:  
    - Range: `OrderLogs!A:E`  
    - Append operation with key row detection enabled  
  - Inputs: Log data from either success or shortage paths  
  - Credentials: Google Sheets OAuth2  
  - Edge Cases: Google Sheets API limits or permission issues.

---

### 3. Summary Table

| Node Name                   | Node Type        | Functional Role                               | Input Node(s)          | Output Node(s)                | Sticky Note                                                                                                         |
|-----------------------------|------------------|-----------------------------------------------|------------------------|------------------------------|---------------------------------------------------------------------------------------------------------------------|
| Order Webhook               | Webhook          | Receives order POST requests                   | External API           | Check Order                  | Overview: Receives orders from external sources via API.                                                            |
| Check Order                | Function         | Validates order payload                         | Order Webhook          | Request Info                 | Overview: Checks the validity of the order request.                                                                  |
| Request Info               | Function         | Structures order data                           | Check Order             | Item and quantity            | Order Request: Process data to extract product and quantity info.                                                    |
| Item and quantity          | Function         | Extracts SKU and quantity per item              | Request Info            | Get Inventories, Mapping data | Order Request: Process data to extract product and quantity info.                                                    |
| Get Inventories            | Google Sheets    | Looks up inventory stock for SKUs               | Item and quantity       | Mapping data                 | Check Inventory: Retrieve inventory info and compare with order.                                                    |
| Mapping data               | Merge            | Combines order quantities with inventory stock | Get Inventories, Item and quantity | enough stock?              | Check Inventory: Retrieve inventory info and compare with order.                                                    |
| enough stock?              | If               | Checks stock sufficiency for all items          | Mapping data            | Create Invoice (true), Slack notification (Stock shortage) (false) | Check Inventory: Retrieve inventory info and compare with order.                                                    |
| Create Invoice             | Function         | Generates invoice data                           | enough stock? (true)    | Slack notification (OK)      | Notifications: Notify manager of successful order.                                                                   |
| Slack notification (OK)    | Slack            | Sends success notification to Slack             | Create Invoice          | create data logs (success)   | Notifications: Notify manager of successful order.                                                                   |
| Slack notification (Stock shortage) | Slack  | Sends stock shortage alert to Slack              | enough stock? (false)   | create data logs (Stock shortage) | Notifications: Notify manager of stock shortage.                                                                     |
| create data logs (success) | Function         | Creates success log entry                         | Slack notification (OK) | Save logs                   | Logging: Log process details to Google Sheets.                                                                       |
| create data logs (Stock shortage) | Function    | Creates out-of-stock log entry                    | Slack notification (Stock shortage) | Save logs                   | Logging: Log process details to Google Sheets.                                                                       |
| Save logs                  | Google Sheets    | Appends logs to Google Sheet                      | create data logs (success), create data logs (Stock shortage) | -                          | Logging: Log process details to Google Sheets.                                                                       |
| Sticky Note                | Sticky Note      | Overview and setup instructions                   | -                      | -                            | See overview section for detailed explanation and setup instructions.                                                |
| Sticky Note1               | Sticky Note      | Order Request block explanation                    | -                      | -                            | Order Request: Process the data to determine product and quantity.                                                   |
| Sticky Note2               | Sticky Note      | Inventory check explanation                         | -                      | -                            | Check Inventory: Retrieve inventory information and compare it with the order request.                               |
| Sticky Note3               | Sticky Note      | Notifications explanation                           | -                      | -                            | Notifications: Generate Slack notifications for success or stock shortage.                                          |
| Sticky Note4               | Sticky Note      | Logging explanation                                 | -                      | -                            | Logging: Log process details to Google Sheets.                                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - Name: `Order Webhook`  
   - HTTP Method: POST  
   - Path: `ecom-order`  
   - Response Mode: Return all entries, last node output  
   - Purpose: Receive incoming order requests.

2. **Create Function Node: Check Order**  
   - Name: `Check Order`  
   - Code: Validate presence of `id`, `customer`, and non-empty `items` array in payload; throw error if missing.  
   - Connect input from `Order Webhook`.

3. **Create Function Node: Request Info**  
   - Name: `Request Info`  
   - Code: Extract order id, customer name/email, items with sku, qty (default 1), name, price, total amount.  
   - Connect input from `Check Order`.

4. **Create Function Node: Item and quantity**  
   - Name: `Item and quantity`  
   - Code: Map items to objects with only `sku` and `qty`.  
   - Connect input from `Request Info`.

5. **Create Google Sheets Node: Get Inventories**  
   - Name: `Get Inventories`  
   - Operation: Lookup  
   - Sheet ID: Use your Google Sheets file containing inventory data  
   - Range: `Inventories!A:B`  
   - Lookup Column: `sku`  
   - Lookup Value: `={{$json["sku"]}}` (expression)  
   - Credentials: Configure Google Sheets OAuth2  
   - Connect input from `Item and quantity`.

6. **Create Merge Node: Mapping data**  
   - Name: `Mapping data`  
   - Mode: Combine  
   - Fields to Match: `sku`  
   - Connect inputs from both `Get Inventories` (main input 1) and `Item and quantity` (main input 2).

7. **Create If Node: enough stock?**  
   - Name: `enough stock?`  
   - Condition: Boolean  
   - Expression: `{{$items().every(item => item.json.qty <= item.json.stock)}} === true`  
   - Connect input from `Mapping data`.

8. **Create Function Node: Create Invoice**  
   - Name: `Create Invoice`  
   - Code: Generate invoiceId with timestamp, include orderId, sku, qty, customer name, amount, and current date ISO string.  
   - Connect True output of `enough stock?` to this node.

9. **Create Slack Node: Slack notification (OK)**  
   - Name: `Slack notification (OK)`  
   - Channel: `#warehouse` (or your configured Slack channel)  
   - Text: Compose message with invoice and order details, e.g.:  
     `✅ Order {{ $json.orderId }} Invoiced. \nInvoice: {{ $json.invoiceId }}, \nAmount: ${{ $json.amount }}, \nCustomer: {{ $json.customer }}, \nDate: {{ $json.date }}`  
   - Authentication: Slack OAuth2 (configure credentials)  
   - Connect input from `Create Invoice`.

10. **Create Slack Node: Slack notification (Stock shortage)**  
    - Name: `Slack notification (Stock shortage)`  
    - Channel: `#warehouse`  
    - Text: Compose message alerting stock shortage, e.g.:  
      `⚠️ Order {{ $('Request Info').item.json.id }} is not available due to lack of stock.\nSKU: {{ $json.sku }}\nRequest quantity: {{ $json.qty }}\nStock quantity: {{ $json.stock }}`  
    - Authentication: Slack OAuth2  
    - Connect False output of `enough stock?` to this node.

11. **Create Function Node: create data logs (success)**  
    - Name: `create data logs (success)`  
    - Code: Create JSON log with order_id, amount, status "Success", created_at timestamp.  
    - Connect input from `Slack notification (OK)`.

12. **Create Function Node: create data logs (Stock shortage)**  
    - Name: `create data logs (Stock shortage)`  
    - Code: Create JSON log with order_id, amount, status "Out of Stock", created_at timestamp.  
    - Connect input from `Slack notification (Stock shortage)`.

13. **Create Google Sheets Node: Save logs**  
    - Name: `Save logs`  
    - Operation: Append  
    - Range: `OrderLogs!A:E`  
    - Sheet ID: Same Google Sheets file as inventory  
    - Credentials: Google Sheets OAuth2  
    - Connect inputs from both `create data logs (success)` and `create data logs (Stock shortage)`.

14. **Test and activate the workflow.**  
    - Use sample order JSON (see section 5) to test webhook and verify notifications and logging.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                               | Context or Link                                                                                               |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------|
| This workflow is a core part of an Order Management System that automates inventory checks and sends Slack alerts to reduce stock-outs and operational costs.                                                                             | Workflow overview and purpose.                                                                                |
| Sample order JSON payload for webhook testing:                                                                                                                                                                                           | See setup steps in sticky note.                                                                                |
| ```json                                                                                                                                                                                                                                   |                                                                                                               |
| {                                                                                                                                                                                                                                         |                                                                                                               |
|   "id": "ORDER1001",                                                                                                                                                                                                                       |                                                                                                               |
|   "customer": {                                                                                                                                                                                                                            |                                                                                                               |
|     "name": "Customer",                                                                                                                                                                                                                    |                                                                                                               |
|     "email": "customer@example.com"                                                                                                                                                                                                       |                                                                                                               |
|   },                                                                                                                                                                                                                                      |                                                                                                               |
|   "items": [                                                                                                                                                                                                                               |                                                                                                               |
|     {                                                                                                                                                                                                                                     |                                                                                                               |
|       "sku": "SKU001",                                                                                                                                                                                                                      |                                                                                                               |
|       "quantity": 2,                                                                                                                                                                                                                        |                                                                                                               |
|       "name": "Product A",                                                                                                                                                                                                                 |                                                                                                               |
|       "price": 5000                                                                                                                                                                                                                        |                                                                                                               |
|     },                                                                                                                                                                                                                                    |                                                                                                               |
|     {                                                                                                                                                                                                                                     |                                                                                                               |
|       "sku": "SKU002",                                                                                                                                                                                                                      |                                                                                                               |
|       "quantity": 2,                                                                                                                                                                                                                        |                                                                                                               |
|       "name": "Product C",                                                                                                                                                                                                                 |                                                                                                               |
|       "price": 10000                                                                                                                                                                                                                       |                                                                                                               |
|     }                                                                                                                                                                                                                                     |                                                                                                               |
|   ],                                                                                                                                                                                                                                      |                                                                                                               |
|   "total": 30000                                                                                                                                                                                                                           |                                                                                                               |
| }                                                                                                                                                                                                                                         |                                                                                                               |
| ```                                                                                                                                                                                                                                       |                                                                                                               |
| Google Sheets template for inventories and logs: Clone from [WMS Data Demo](https://docs.google.com/spreadsheets/d/1sgTaHwbTWrVCDiSwuvqFVJr0-Ez2t3LTlJaLU4cBcuU/edit?usp=sharing) and update credentials accordingly.                             | Google Sheets integration.                                                                                      |
| Slack channel `#warehouse` must exist, or update Slack node channels accordingly. OAuth2 credentials for Slack must be configured with appropriate scopes (chat:write).                                                                   | Slack integration.                                                                                              |
| Ensure Google Sheets OAuth2 credentials have edit permission on the spreadsheet to append logs and perform lookups.                                                                                                                      | Google Sheets permission requirements.                                                                         |
| Handle possible errors such as missing or malformed webhook payloads, Google Sheets API errors, and Slack API rate limits gracefully in production deployment by adding error workflows or retries.                                        | Operational robustness advice.                                                                                  |

---

**Disclaimer:**  
The provided text is extracted exclusively from an automated workflow created with n8n, an integration and automation tool. The process strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.