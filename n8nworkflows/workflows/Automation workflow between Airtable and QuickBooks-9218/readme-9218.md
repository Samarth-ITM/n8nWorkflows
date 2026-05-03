Automation workflow between Airtable and QuickBooks

https://n8nworkflows.xyz/workflows/automation-workflow-between-airtable-and-quickbooks-9218


# Automation workflow between Airtable and QuickBooks

### 1. Workflow Overview

This workflow automates the synchronization of confirmed sales orders from Airtable into QuickBooks Online (QBO), focusing on creating or updating customers and invoices. It is designed for businesses that manage sales orders, customers, and products in Airtable and need to reflect those records in QuickBooks for accounting and invoicing purposes.

The workflow includes the following logical blocks:

- **1.1 Input Reception:** Webhook triggered by Airtable when a sales order is confirmed.
- **1.2 Sales Order Retrieval & Validation:** Fetch the confirmed order details from Airtable and check if an invoice already exists in QuickBooks.
- **1.3 Customer Lookup & Creation:** Retrieve customer details from Airtable, search for the customer in QuickBooks, then create the customer in QuickBooks if not found.
- **1.4 Product Retrieval & Mapping:** Retrieve order line items from Airtable and map them to corresponding QuickBooks product/service IDs.
- **1.5 Invoice Data Preparation:** Combine order, customer, and product data into a structured invoice payload compatible with QuickBooks.
- **1.6 Invoice Creation in QuickBooks:** Send invoices to QuickBooks via HTTP API call, handling one invoice per batch.
- **1.7 Post-Invoice Airtable Updates:** Record invoice details back in Airtable, update the order and customer records to reflect synchronization status.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception
- **Overview:** Triggered automatically by Airtable webhook upon order confirmation.
- **Nodes Involved:** 
  - Webhook
  - Sticky Note (for purpose)
- **Node Details:**
  - **Webhook**
    - Type: `n8n-nodes-base.webhook`
    - Configuration: Listens to POST requests at path `321f46d5-66da-49f5-a8d1-6b61e4a7321f`.
    - Inputs: External HTTP POST from Airtable automation.
    - Outputs: Passes webhook data to next node.
    - Edge Cases: Incorrect webhook URL or missing record ID in payload.
  - **Sticky Note**
    - Purpose explanation: Webhook entry point, triggered when order status = Confirmed in Airtable.

#### 1.2 Sales Order Retrieval & Validation
- **Overview:** Fetches the confirmed sales order from Airtable and checks if an invoice already exists in QuickBooks.
- **Nodes Involved:** 
  - Get Airtable Orders Records
  - If Invoice not Created
  - Sticky Notes
- **Node Details:**
  - **Get Airtable Orders Records**
    - Type: Airtable node
    - Configuration: Fetches record by ID (from webhook query `recordId`) from "Confirmed Orders" table in Airtable base "(Intuz) Operations Hub (Sample) (Copy)".
    - Inputs: Webhook output
    - Outputs: Order details JSON
    - Edge Cases: Record not found, invalid record ID.
  - **If Invoice not Created**
    - Type: If node
    - Configuration: Checks if field `"Synced to QBO (A)"` equals `"FALSE"` to detect if invoice has been created.
    - Inputs: Order record JSON
    - Outputs: True branch proceeds; false skips further processing.
    - Edge Cases: Field missing or malformed.
  - **Sticky Note2**
    - Explains that orders with existing QuickBooks invoices are skipped.

#### 1.3 Customer Lookup & Creation
- **Overview:** Retrieves customer details linked to the order, searches for the customer in QuickBooks to prevent duplicates, and creates the customer in QuickBooks if not found.
- **Nodes Involved:**
  - Get Customer Details
  - QuickBooks - Find Customer
  - IF - Customer doesn't Exists?
  - Create a customer
  - Append Customers (merge)
  - Sticky Notes
- **Node Details:**
  - **Get Customer Details**
    - Type: Airtable node
    - Configuration: Retrieves customer record by ID from "Customers" table.
    - Inputs: True branch from "If Invoice not Created".
    - Outputs: Customer details JSON.
    - Edge Cases: Customer record missing or incomplete fields.
  - **QuickBooks - Find Customer**
    - Type: QuickBooks node
    - Configuration: Searches QuickBooks customers filtering by `DisplayName` matching customer name from Airtable.
    - Inputs: Customer details output.
    - Outputs: QuickBooks customer data if found.
    - Edge Cases: API errors, no customer found.
  - **IF - Customer doesn't Exists?**
    - Type: If node
    - Configuration: Checks if QuickBooks customer ID is empty, indicating customer doesn't exist.
    - Outputs: 
      - True: Create a new customer node.
      - False: Append existing customer data for further processing.
  - **Create a customer**
    - Type: QuickBooks node
    - Configuration: Creates a customer in QuickBooks with details from Airtable customer record including billing address, phone, and email.
    - Inputs: True branch from IF node.
    - Outputs: Newly created QuickBooks customer data.
    - Edge Cases: API errors, missing required fields.
  - **Append Customers**
    - Type: Merge node
    - Configuration: Merges outputs from created customers and existing customers into a single data stream.
    - Inputs: Outputs from "Create a customer" and false branch of IF node.
    - Outputs: Combined customer list for next processing step.
  - **Sticky Notes3,4,5,6,7**
    - Explain each step: retrieving customer details, searching in QuickBooks, conditional creation, and merging customers.

#### 1.4 Product Retrieval & Mapping
- **Overview:** Fetches order line items linked to the sales order and maps each product to its QuickBooks product/service ID.
- **Nodes Involved:**
  - Get Products
  - Search Product ID
  - Sticky Notes8,9
- **Node Details:**
  - **Get Products**
    - Type: Airtable node
    - Configuration: Searches "Order Lines" table filtered by Sales Order ID matching the current order.
    - Inputs: Output from "Append Customers".
    - Outputs: Product line items linked to the order.
    - Edge Cases: No product lines found.
  - **Search Product ID**
    - Type: Airtable node
    - Configuration: Searches "Product & Service" table for each product name to find corresponding QuickBooks product IDs.
    - Inputs: Output from "Get Products".
    - Outputs: Product & Service records with QuickBooks IDs.
    - Edge Cases: Missing product mappings.
  - **Sticky Notes**
    - Explain the purpose of these nodes for product retrieval and mapping.

#### 1.5 Invoice Data Preparation
- **Overview:** Combines all gathered data (orders, customers, products) into a structured invoice JSON compatible with QuickBooks API.
- **Nodes Involved:**
  - Data Preparation (Code)
  - Parse in HTTP (Code)
  - Sticky Notes10,11
- **Node Details:**
  - **Data Preparation**
    - Type: Code node (JavaScript)
    - Configuration:
      - Joins sales order data, customer info, and products.
      - Maps QuickBooks customer IDs and product IDs.
      - Prepares invoice fields including invoice date, due date, discount, and invoice lines.
      - Includes customer email for billing.
    - Inputs: Multiple upstream nodes (orders, customers, products).
    - Outputs: Structured invoice data objects.
    - Edge Cases: Missing customer or product IDs, empty arrays.
  - **Parse in HTTP**
    - Type: Code node
    - Configuration:
      - Converts the structured invoice data into QuickBooks API-compatible JSON.
      - Builds invoice line items including sales items and discount lines.
      - Validates and formats data for API consumption.
    - Inputs: Output of Data Preparation.
    - Outputs: Final invoice payload for HTTP request.
    - Edge Cases: Incorrect data formatting, missing required fields.
  - **Sticky Notes10,11**
    - Describe core logic of data preparation and payload formatting.

#### 1.6 Invoice Creation in QuickBooks
- **Overview:** Sends invoices one-by-one to QuickBooks via HTTP POST to create invoice records.
- **Nodes Involved:**
  - Loop Over Items1 (Split In Batches)
  - Create Invoice URL (HTTP Request)
  - Sticky Notes12,13
- **Node Details:**
  - **Loop Over Items1**
    - Type: SplitInBatches node
    - Configuration: Processes invoices one at a time for controlled API calls.
    - Inputs: Output from "Parse in HTTP".
    - Outputs: Single invoice payload per iteration.
    - Edge Cases: Large payloads, API rate limits.
  - **Create Invoice URL**
    - Type: HTTP Request node
    - Configuration:
      - Method: POST
      - URL: QuickBooks Sandbox API endpoint for invoice creation (company ID and minorversion specified).
      - Authentication: OAuth2 credentials for QuickBooks.
      - Payload: JSON invoice body from previous node.
      - Retries enabled on failure.
    - Inputs: Output from batch loop node.
    - Outputs: API response with created invoice details.
    - Edge Cases: Auth failures, API errors, invalid payloads.
  - **Sticky Notes12,13**
    - Explain the batch processing and API invoice creation details.

#### 1.7 Post-Invoice Airtable Updates
- **Overview:** Logs invoice details into Airtable and updates related order and customer records to reflect successful synchronization.
- **Nodes Involved:**
  - Create Invoice record (Airtable)
  - Update Order record (Airtable)
  - Update Customer Record (Airtable)
  - Sticky Notes14,15,16
- **Node Details:**
  - **Create Invoice record**
    - Type: Airtable node
    - Configuration: Creates a record in "Invoices & Payments" table with invoice number, amount, dates, payment status, and other metadata from QuickBooks response.
    - Inputs: Output from "Create Invoice URL".
    - Outputs: Created invoice record info.
    - Edge Cases: Airtable API limits, field mapping errors.
  - **Update Order record**
    - Type: Airtable node
    - Configuration: Updates the "Confirmed Orders" table record with QuickBooks Invoice ID and Invoice Number to mark the order as invoiced.
    - Inputs: Output from "Create Invoice record".
    - Outputs: Updated order record info.
    - Edge Cases: Record mismatch, field permission issues.
  - **Update Customer Record**
    - Type: Airtable node
    - Configuration: Updates the customer record in Airtable with QuickBooks Customer ID to avoid duplicate creations in the future.
    - Inputs: Output from "Update Order record".
    - Outputs: Updated customer record info.
    - Edge Cases: Missing customer linkage.
  - **Sticky Notes14,15,16**
    - Describe the purpose of these update operations to keep Airtable and QuickBooks data in sync.

---

### 3. Summary Table

| Node Name                  | Node Type               | Functional Role                                    | Input Node(s)                         | Output Node(s)                      | Sticky Note                                                                                                      |
|----------------------------|-------------------------|---------------------------------------------------|-------------------------------------|-----------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Webhook                    | webhook                 | Entry point: receives webhook call from Airtable | -                                   | Get Airtable Orders Records        | Purpose: Entry point. Triggered automatically when a Sales Order is confirmed in Airtable.                      |
| Get Airtable Orders Records | airtable                | Fetch confirmed Sales Order details from Airtable | Webhook                             | If Invoice not Created             | Get Orders Records: Fetch details of confirmed Sales Order using record ID. Must have Sales Order ID for linkage. |
| If Invoice not Created      | if                      | Check if invoice already created in QuickBooks    | Get Airtable Orders Records         | Get Customer Details              | Invoice record check: Skip if invoice already synced (Synced to QBO = FALSE).                                   |
| Get Customer Details        | airtable                | Retrieve customer info linked to order            | If Invoice not Created              | QuickBooks - Find Customer        | Get Customer Details: Retrieve linked customer info from Airtable.                                              |
| QuickBooks - Find Customer  | quickbooks              | Search customer in QuickBooks by DisplayName       | Get Customer Details               | IF - Customer doesn't Exists?     | QuickBooks - Find Customer: Ensures no duplicate customer creation in QBO.                                      |
| IF - Customer doesn't Exists?| if                     | Check if customer exists in QuickBooks             | QuickBooks - Find Customer          | Create a customer (yes), Append Customers (no) | Customer doesn’t Exist? Checks if QBO customer found; create if not.                                             |
| Create a customer           | quickbooks              | Create customer in QuickBooks if not found         | IF - Customer doesn't Exists? (yes)| Append Customers                  | Create a Customer: Creates new customer in QuickBooks from Airtable data.                                       |
| Append Customers            | merge                   | Merge existing and newly created customers          | Create a customer, IF - Customer doesn't Exists? (no) | Get Products                    | Append Customers (Merge): Combine customers before invoice processing.                                          |
| Get Products               | airtable                | Fetch order line items linked to sales order        | Append Customers                   | Search Product ID                 | Get Products: Fetch order lines from Airtable Order Lines table.                                                |
| Search Product ID          | airtable                | Map product names to QuickBooks product IDs         | Get Products                      | Data Preparation                 | Search Product ID: Find QBO Product/Service IDs matching Airtable products.                                     |
| Data Preparation           | code                    | Prepare structured invoice payload                   | Search Product ID                 | Parse in HTTP                   | Data Preparation: Core logic combining orders, customers, and products for invoice.                             |
| Parse in HTTP              | code                    | Format invoice payload for QuickBooks API            | Data Preparation                 | Loop Over Items1                | Parse in HTTP: Convert JS objects into QBO-compatible invoice JSON format.                                      |
| Loop Over Items1           | splitInBatches          | Process invoices one-by-one for API calls            | Parse in HTTP                   | Create Invoice URL (main branch) | Split in Batches: Iterates invoices for controlled QuickBooks API calls.                                       |
| Create Invoice URL         | httpRequest             | Post invoice JSON to QuickBooks API                   | Loop Over Items1                | Create Invoice record             | Create Invoice URL: Creates invoice in QuickBooks sandbox via HTTP POST.                                       |
| Create Invoice record      | airtable                | Log created invoice details back into Airtable       | Create Invoice URL              | Update Order record              | Create Invoice Record: Logs invoice details in Airtable Invoices & Payments table.                              |
| Update Order record        | airtable                | Update order record with QBO invoice info             | Create Invoice record           | Update Customer Record          | Update Order Record: Updates Confirmed Orders table with QBO Invoice ID and Number.                             |
| Update Customer Record     | airtable                | Update customer record with QBO customer ID           | Update Order record             | Loop Over Items1                | Update Customer Record: Stores QBO Customer ID in Airtable customer record to avoid duplicates.                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**
   - Type: webhook
   - Path: `321f46d5-66da-49f5-a8d1-6b61e4a7321f`
   - Method: POST
   - Purpose: Receive webhook calls from Airtable automation when a sales order is confirmed.

2. **Create Airtable Node: Get Airtable Orders Records**
   - Base: `(Intuz) Operations Hub (Sample) (Copy)`
   - Table: `Confirmed Orders`
   - Operation: Get Record by ID
   - Record ID: Expression `{{$json["query"]["recordId"]}}` from webhook
   - Purpose: Fetch confirmed sales order details.

3. **Create If Node: If Invoice not Created**
   - Condition: Check if field `"Synced to QBO (A)"` equals `"FALSE"` in the order record.
   - True branch: Continue processing.
   - False branch: Stop.

4. **Create Airtable Node: Get Customer Details**
   - Base: `(Intuz) Operations Hub (Sample) (Copy)`
   - Table: `Customers`
   - Operation: Get Record by ID
   - Record ID: Expression `={{ $json['(Q) Customer'][0] }}`
   - Purpose: Retrieve customer details linked to the order.

5. **Create QuickBooks Node: QuickBooks - Find Customer**
   - Operation: getAll customers
   - Filter Query: `WHERE DisplayName = '{{ $json["(Q) Customer Name"] }}'`
   - Purpose: Search for customer in QuickBooks to avoid duplicates.

6. **Create If Node: IF - Customer doesn't Exists?**
   - Condition: Check if QuickBooks customer ID is empty (`{{ $json.Id }}` is empty).
   - True branch: Create customer.
   - False branch: Proceed with existing customer.

7. **Create QuickBooks Node: Create a customer**
   - Operation: create customer
   - DisplayName: From Airtable Customer Name
   - Additional fields: Billing address, phone, email from Airtable customer record
   - Purpose: Create customer in QuickBooks if not found.

8. **Create Merge Node: Append Customers**
   - Mode: Merge outputs of "Create a customer" (true branch) and false branch of IF.
   - Purpose: Combine customer data for next steps.

9. **Create Airtable Node: Get Products**
   - Base: `(Intuz) Operations Hub (Sample) (Copy)`
   - Table: `Order Lines`
   - Operation: Search Records
   - Filter: `{Sales Order} = "{{ $('Get Airtable Orders Records').item.json['Sales Order ID'] }}"`
   - Purpose: Fetch products linked to sales order.

10. **Create Airtable Node: Search Product ID**
    - Base: `(Intuz) Operations Hub (Sample) (Copy)`
    - Table: `Product & Service`
    - Operation: Search Records
    - Filter: `{(Q) Product/Service Name} = "{{ $json['(Q) Item Name (A)'][0] }}"`
    - Purpose: Find QuickBooks product/service IDs.

11. **Create Code Node: Data Preparation**
    - JavaScript: Combine sales orders, customers, and products.
    - Map QuickBooks customer IDs and product IDs.
    - Prepare invoice data with invoice date, due date, discount, and products.
    - Include customer email.
    - Inputs: Sales orders, customers, products, QuickBooks customers, search products.
    - Outputs: Structured invoice data.

12. **Create Code Node: Parse in HTTP**
    - JavaScript: Transform invoice data into QuickBooks API JSON format.
    - Build invoice lines including discounts.
    - Outputs: JSON invoice payload.

13. **Create SplitInBatches Node: Loop Over Items1**
    - Batch size: 1
    - Input: Parsed invoice payloads.
    - Purpose: Process invoices one by one.

14. **Create HTTP Request Node: Create Invoice URL**
    - Method: POST
    - URL: `https://sandbox-quickbooks.api.intuit.com/v3/company/{CompanyID}/invoice?minorversion=75`
    - Authentication: QuickBooks OAuth2 credentials
    - Headers: Content-Type: application/json
    - Body: JSON invoice payload
    - Retries enabled on failure.
    - Inputs: Batch output.
    - Outputs: QuickBooks API response.

15. **Create Airtable Node: Create Invoice record**
    - Base: `(Intuz) Operations Hub (Sample) (Copy)`
    - Table: `Invoices & Payments`
    - Operation: Create record
    - Map fields from QuickBooks response: invoice number, amount, dates, payment status.
    - Inputs: QuickBooks response.

16. **Create Airtable Node: Update Order record**
    - Base: `(Intuz) Operations Hub (Sample) (Copy)`
    - Table: `Confirmed Orders`
    - Operation: Update record
    - Matching column: Sales Order ID
    - Update fields: QuickBooks Invoice ID and Invoice Number.
    - Inputs: Invoice creation record.

17. **Create Airtable Node: Update Customer Record**
    - Base: `(Intuz) Operations Hub (Sample) (Copy)`
    - Table: `Customers`
    - Operation: Update record
    - Matching column: (Q) Customer Name
    - Update fields: QuickBooks Customer ID.
    - Inputs: Updated order record.

18. **Connect Flow**
    - Webhook → Get Airtable Orders Records → If Invoice not Created (true) → Get Customer Details → QuickBooks - Find Customer → IF - Customer doesn't Exists? (true) → Create a customer → Append Customers
    - IF - Customer doesn't Exists? (false) → Append Customers
    - Append Customers → Get Products → Search Product ID → Data Preparation → Parse in HTTP → Loop Over Items1 → Create Invoice URL → Create Invoice record → Update Order record → Update Customer Record → Loop Over Items1 (continue)

19. **Credentials Setup**
    - Airtable: API key with read and write access to relevant bases and tables.
    - QuickBooks: OAuth2 credentials for sandbox environment with invoice and customer scopes.

20. **Testing & Validation**
    - Ensure Airtable webhook triggers with correct record IDs.
    - Validate API credentials and permissions.
    - Confirm correct field mappings and existence of required fields.
    - Monitor for API rate limits and error handling.

---

### 5. General Notes & Resources

| Note Content                                                                                               | Context or Link                                                                                     |
|------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Airtable structure requirements document available: [Airtable Structure Requirement](https://drive.google.com/drive/folders/1dE4sXikesaTpLE-Pc-VU3mqg0fMZ8EDU) | Contains detailed schema and field requirements for this workflow’s Airtable base and tables.     |
| Ensure webhook URL is added as POST request in Airtable automation triggered when order status = Confirmed. | Critical for triggering the workflow correctly on sales order confirmation.                       |
| QuickBooks Sandbox API endpoint uses `minorversion=75` and requires correct company ID.                    | Important for API compatibility and versioning.                                                  |
| Discount lines in invoices are percentage-based discounts integrated in invoice line items.                | QuickBooks calculates discount amounts automatically using this setup.                            |

---

**Disclaimer:** The text above is derived solely from an automated workflow designed with n8n, adhering strictly to content policies. It contains no illegal or protected elements, only public and legal data.