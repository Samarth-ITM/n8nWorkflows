Generate Invoices for Customers with Jotform and QuickBooks

https://n8nworkflows.xyz/workflows/generate-invoices-for-customers-with-jotform-and-quickbooks-9642


# Generate Invoices for Customers with Jotform and QuickBooks

### 1. Workflow Overview

This workflow automates the process of generating and sending invoices to customers based on product or service orders submitted via a Jotform form. It integrates Jotform form submissions with QuickBooks Online (QBO), handling customer existence checks, customer creation or updating, product retrieval, invoice creation, and invoice emailing.

The logical blocks are:

- **1.1 Receive Submission**: Captures form submissions via webhook from Jotform.
- **1.2 Data Formatting**: Extracts and structures customer, address, and item details from the raw form data.
- **1.3 Customer Existence Check**: Verifies if the customer exists in QuickBooks by email.
- **1.4 Customer Handling**: Updates existing customer details or creates a new customer in QuickBooks.
- **1.5 Product Retrieval**: Fetches the product/item details from QuickBooks based on the form submission.
- **1.6 Invoice Creation**: Generates an invoice in QuickBooks using the customer and product data.
- **1.7 Invoice Sending**: Emails the created invoice to the customer.

---

### 2. Block-by-Block Analysis

#### 1.1 Receive Submission

- **Overview:**  
  This block receives the product/service order submission from Jotform via a webhook.

- **Nodes Involved:**  
  - Receive form submission (Webhook)

- **Node Details:**

  - **Receive form submission**  
    - Type: Webhook  
    - Role: Entry point; listens for POST requests at the path `/requests`.  
    - Configuration: HTTP method is POST, no additional options.  
    - Inputs: External HTTP POST from Jotform submission.  
    - Outputs: Raw JSON data of form submission.  
    - Edge Cases: Network issues, webhook misconfiguration, invalid payloads.  
    - Notes: Requires Jotform webhook setup to point to this webhook URL.

#### 1.2 Data Formatting

- **Overview:**  
  Extracts and formats customer info, billing address, and selected product/item name from the raw form data for easier use downstream.

- **Nodes Involved:**  
  - Format data (Code)

- **Node Details:**

  - **Format data**  
    - Type: Code  
    - Role: Parses HTML-style address string and extracts customer and item info into structured JSON.  
    - Configuration: JavaScript code uses regex to extract address fields and parses customer name, email, phone, and item name.  
    - Inputs: Raw JSON from webhook node.  
    - Outputs: Structured JSON with `address`, `customer`, and `item` objects.  
    - Edge Cases: Malformed or missing address fields, incomplete form data. Could return nulls for missing fields.  
    - Version-specific: Uses n8n Code node version 2 syntax.

#### 1.3 Customer Existence Check

- **Overview:**  
  Checks if the customer already exists in QuickBooks Online using their primary email address.

- **Nodes Involved:**  
  - Check if the customer exists (QuickBooks)

- **Node Details:**

  - **Check if the customer exists**  
    - Type: QuickBooks node  
    - Role: Queries QBO for customers where the primary email matches the submitted email, limit 1 result.  
    - Configuration: Operation `getAll` on `customer` resource, filter query by email.  
    - Credentials: QuickBooks OAuth2 credentials required.  
    - Inputs: Formatted data node output.  
    - Outputs: Customer data if exists, empty if not.  
    - Edge Cases: API rate limits, auth failures, no customer found.  
    - Notes: Email must be accurate; case-sensitive matching.

#### 1.4 Customer Handling

- **Overview:**  
  Based on existence check, either updates existing customer billing details or creates a new customer in QuickBooks.

- **Nodes Involved:**  
  - If (conditional)  
  - Update the customer (QuickBooks)  
  - Create the customer (QuickBooks)

- **Node Details:**

  - **If**  
    - Type: Conditional (If)  
    - Role: Branching logic; if customer exists (data found), update customer; else create new customer.  
    - Configuration: Checks existence of JSON and `Id` field in customer data.  
    - Inputs: Customer check node output.  
    - Outputs: Two branches — true (exists), false (does not exist).  
    - Edge Cases: Improper data structure causing misrouting.

  - **Update the customer**  
    - Type: QuickBooks node  
    - Role: Updates billing address, name, phone of existing customer.  
    - Configuration: Operation `update` on customer resource with customerId from JSON, setting address fields and personal info from formatted data.  
    - Credentials: QuickBooks OAuth2 required.  
    - Inputs: If node true branch.  
    - Outputs: Updated customer JSON with Id.  
    - Edge Cases: Update conflicts, invalid IDs, API errors.

  - **Create the customer**  
    - Type: QuickBooks node  
    - Role: Creates a new customer with provided billing address and contact info.  
    - Configuration: Operation `create` on customer resource, fields populated from formatted data node.  
    - Credentials: QuickBooks OAuth2 required.  
    - Inputs: If node false branch.  
    - Outputs: Created customer JSON with Id.  
    - Edge Cases: Duplicate customers, validation errors, API limits.

  - **Add customer id (Code)**  
    - Type: Code  
    - Role: Adds the customer Id from update or create node output into the working JSON data for downstream use.  
    - Inputs: Output from either update or create customer nodes.  
    - Outputs: Augmented JSON including customer id.  
    - Edge Cases: Missing Id field, null inputs.

#### 1.5 Product Retrieval

- **Overview:**  
  Retrieves the product or service item details from QuickBooks by matching the name submitted in the form.

- **Nodes Involved:**  
  - Get the product (QuickBooks)  
  - Add item id (Code)

- **Node Details:**

  - **Get the product**  
    - Type: QuickBooks node  
    - Role: Queries QBO item list filtering by the submitted item name, limiting to one result.  
    - Configuration: Operation `getAll` on `item` resource, filter query with item name from JSON.  
    - Credentials: QuickBooks OAuth2 required.  
    - Inputs: Output JSON augmented with customer id.  
    - Outputs: Item data with Id and other details.  
    - Edge Cases: Item not found, multiple items with same name, API errors.

  - **Add item id**  
    - Type: Code  
    - Role: Adds the item Id into the working JSON data for invoice creation.  
    - Inputs: Output from get product node.  
    - Outputs: JSON including customer and item ids.  
    - Edge Cases: Missing item Id, empty results.

#### 1.6 Invoice Creation

- **Overview:**  
  Creates a new invoice in QuickBooks for the given customer and item.

- **Nodes Involved:**  
  - Create the invoice (QuickBooks)

- **Node Details:**

  - **Create the invoice**  
    - Type: QuickBooks node  
    - Role: Creates an invoice with one line item for the selected product/service and customer.  
    - Configuration: Operation `create` on invoice resource, with line containing amount 1, itemId from JSON, description "Jotform submission", and customer reference by id.  
    - Credentials: QuickBooks OAuth2 required.  
    - Inputs: JSON with customer and item ids.  
    - Outputs: Created invoice JSON with invoice Id.  
    - Edge Cases: Invalid item or customer Ids, API errors, invoice creation failures.

#### 1.7 Invoice Sending

- **Overview:**  
  Sends the created invoice via email to the customer's email address.

- **Nodes Involved:**  
  - Send the invoice (QuickBooks)

- **Node Details:**

  - **Send the invoice**  
    - Type: QuickBooks node  
    - Role: Uses QuickBooks API to email the invoice to the customer.  
    - Configuration: Operation `send` on invoice resource, specifying invoiceId and customer email extracted from data.  
    - Credentials: QuickBooks OAuth2 required.  
    - Inputs: Invoice JSON with invoice Id and customer email.  
    - Outputs: Confirmation of email sent.  
    - Edge Cases: Email delivery failures, invalid email, API errors.

---

### 3. Summary Table

| Node Name              | Node Type                  | Functional Role                     | Input Node(s)           | Output Node(s)          | Sticky Note                                                                                                   |
|------------------------|----------------------------|-----------------------------------|------------------------|------------------------|---------------------------------------------------------------------------------------------------------------|
| Receive form submission| Webhook                    | Entry point, receives Jotform data| -                      | Format data             | ## Receive Submission Receives the product/service form submission from Jotform                               |
| Format data             | Code                       | Parses and formats form data      | Receive form submission | Check if the customer exists | ## Format Data Formats the data thus making it easier to be used in other nodes                                |
| Check if the customer exists | QuickBooks               | Checks if customer exists in QBO  | Format data             | If                      | ## Check If Customer exists Checks if the customer exists in QBO or not                                        |
| If                      | If                         | Branches workflow by customer existence | Check if the customer exists | Update the customer, Create the customer |                                                                                                               |
| Update the customer      | QuickBooks                 | Updates existing customer details | If (true branch)        | Add customer id          | ## Customer Exists Now since the customer exists we will update the customer details like updating the billing details with the new one |
| Create the customer      | QuickBooks                 | Creates new customer in QBO       | If (false branch)       | Add customer id          | ## Customer Doesn't Exist Now since the customer doesn't exist we will create new customer                     |
| Add customer id          | Code                       | Adds customer Id to working data  | Update the customer, Create the customer | Get the product          | ## Add Customer Id Adds customer id to the data                                                               |
| Get the product          | QuickBooks                 | Retrieves product/item from QBO   | Add customer id         | Add item id              | ## Get The Item Gets the selected product/service from QBO                                                     |
| Add item id              | Code                       | Adds item Id to working data      | Get the product         | Create the invoice       | ## Add Item Id Adds item (service/product) id to the data                                                     |
| Create the invoice       | QuickBooks                 | Creates invoice for customer      | Add item id             | Send the invoice         | ## Create The Invoice Creates a new invoice for that customer                                                  |
| Send the invoice         | QuickBooks                 | Emails invoice to customer        | Create the invoice      | -                        | ## Send The Invoice Sends the newly created invoice for that customer(via email)                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook node**  
   - Type: Webhook  
   - Name: `Receive form submission`  
   - HTTP Method: POST  
   - Path: `requests`  
   - Purpose: Receive Jotform submissions.  
   - Save credentials for n8n webhook.

2. **Create Code node to format data**  
   - Type: Code  
   - Name: `Format data`  
   - Code: Use JavaScript to parse the `billingAddress` field (HTML format) with regex extracting street, city, state, postal code, country; extract customer name, email, phone, and item name from form data.  
   - Input: Output from webhook node.  
   - Outputs structured JSON with `address`, `customer`, `item`.

3. **Create QuickBooks node to check customer existence**  
   - Type: QuickBooks  
   - Name: `Check if the customer exists`  
   - Resource: `customer`  
   - Operation: `getAll`  
   - Limit: 1  
   - Filters: Query by `PrimaryEmailAddr = '{{ $json.customer.email }}'`  
   - Credentials: Set up QuickBooks OAuth2 credentials.  
   - Input: Output from `Format data`.

4. **Create If node for branching**  
   - Type: If  
   - Name: `If`  
   - Condition: Check if customer data exists and has an `Id` field, using the condition:  
     - Object exists on `$json`  
     - String exists on `$json.Id`  
   - Input: Output from customer existence check node.

5. **Create QuickBooks node to update customer**  
   - Type: QuickBooks  
   - Name: `Update the customer`  
   - Resource: `customer`  
   - Operation: `update`  
   - CustomerId: `={{ $json.Id }}`  
   - Fields to update: `BillAddr` (with city, line1, postal code from `Format data`), `GivenName`, `PrimaryPhone` from `Format data`.  
   - Credentials: QuickBooks OAuth2.  
   - Input: If node true branch.

6. **Create QuickBooks node to create customer**  
   - Type: QuickBooks  
   - Name: `Create the customer`  
   - Resource: `customer`  
   - Operation: `create`  
   - Fields: `BillAddr`, `GivenName`, `PrimaryPhone`, `PrimaryEmailAddr` from `Format data`.  
   - Credentials: QuickBooks OAuth2.  
   - Input: If node false branch.

7. **Create Code node to add customer id**  
   - Type: Code  
   - Name: `Add customer id`  
   - Code: Return existing JSON plus add `customer.id` as `{{ $json.Id }}` from previous node (update or create customer).  
   - Input: Outputs from both `Update the customer` and `Create the customer` nodes.

8. **Create QuickBooks node to get product/item**  
   - Type: QuickBooks  
   - Name: `Get the product`  
   - Resource: `item`  
   - Operation: `getAll`  
   - Limit: 1  
   - Filters: Query where `name = '{{ $json.item.name }}'` from JSON.  
   - Credentials: QuickBooks OAuth2.  
   - Input: Output from `Add customer id`.

9. **Create Code node to add item id**  
   - Type: Code  
   - Name: `Add item id`  
   - Code: Return existing JSON plus add `item.id` from `Get the product` node output.  
   - Input: Output from `Get the product`.

10. **Create QuickBooks node to create invoice**  
    - Type: QuickBooks  
    - Name: `Create the invoice`  
    - Resource: `invoice`  
    - Operation: `create`  
    - Line: Array with one object:  
      - Amount: 1  
      - itemId: `{{ $json.item.id }}`  
      - DetailType: SalesItemLineDetail  
      - Description: "Jotform submission"  
    - CustomerRef: `{{ $json.customer.id }}`  
    - Credentials: QuickBooks OAuth2.  
    - Input: Output from `Add item id`.

11. **Create QuickBooks node to send invoice**  
    - Type: QuickBooks  
    - Name: `Send the invoice`  
    - Resource: `invoice`  
    - Operation: `send`  
    - invoiceId: `{{ $json.Id }}` from created invoice  
    - email: `{{ $('Add item id').item.json.customer.email }}`  
    - Credentials: QuickBooks OAuth2.  
    - Input: Output from `Create the invoice`.

12. **Connect nodes in this order:**  
    - Receive form submission → Format data → Check if the customer exists → If  
    - If true → Update the customer → Add customer id → Get the product → Add item id → Create the invoice → Send the invoice  
    - If false → Create the customer → Add customer id → Get the product → Add item id → Create the invoice → Send the invoice

13. **Set up credentials:**  
    - QuickBooks OAuth2 API: Create and configure in n8n with client ID, secret, and tokens as per QuickBooks developer docs.  
    - Webhook URL: Configure Jotform webhook to post submissions to n8n webhook URL.

14. **Test the workflow:**  
    - Submit a form via Jotform with valid product, customer info, and billing address.  
    - Confirm invoice creation and email delivery.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                 | Context or Link                                                                                                  |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| This workflow automates invoice generation and emailing triggered by Jotform submissions integrated with QuickBooks Online.                                                                                                 | Overview and use case                                                                                           |
| Jotform webhook setup instructions: [https://www.jotform.com/help/245-how-to-setup-a-webhook-with-jotform/](https://www.jotform.com/help/245-how-to-setup-a-webhook-with-jotform/)                                         | Jotform webhook configuration                                                                                   |
| QuickBooks Online developer credentials setup: [https://developer.intuit.com/app/developer/qbo/docs/get-started/get-client-id-and-client-secret](https://developer.intuit.com/app/developer/qbo/docs/get-started/get-client-id-and-client-secret) | QuickBooks OAuth2 credentials setup                                                                               |
| The address extraction relies on a specific HTML format in the billing address field; modifications may be required if the form structure changes.                                                                             | Address parsing assumptions                                                                                      |
| The workflow supports typical users such as freelancers, service providers, consultants, small businesses, and e-commerce sellers.                                                                                           | Intended audience                                                                                                |

---

*Disclaimer:* The provided text is exclusively derived from an automated workflow created with n8n, adhering strictly to current content policies and containing no illegal or protected elements. All data processed is legal and public.