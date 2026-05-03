Create or Find Stripe Customers and Automatically Generate Invoices

https://n8nworkflows.xyz/workflows/create-or-find-stripe-customers-and-automatically-generate-invoices-3723


# Create or Find Stripe Customers and Automatically Generate Invoices

### 1. Workflow Overview

This workflow automates customer management and invoicing in Stripe, targeting entrepreneurs, SaaS founders, and online business owners who want to streamline billing operations. It reduces manual effort by programmatically checking for existing customers, creating new customers if needed, generating draft invoices, adding product line items, and finalizing invoices for payment collection.

The workflow is logically divided into the following blocks:

- **1.1 Input Initialization:** Receives a manual trigger and sets initial invoice data including the customer email and product Price ID.
- **1.2 Customer Lookup and Creation:** Searches Stripe for an existing customer by email and conditionally creates a new customer if none is found.
- **1.3 Invoice Creation and Line Item Addition:** Creates a draft invoice for the identified customer and adds a product line item using a Stripe Price ID.
- **1.4 Invoice Finalization:** Finalizes the invoice to make it ready for payment.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Initialization

- **Overview:**  
  This block starts the workflow manually and sets the initial data required for customer lookup and invoice creation, such as the customer email and Stripe Price ID.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - Set invoice data (Set node)  
  - Get Stripe Customer (HTTP Request)

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Entry point to manually start the workflow for testing or demonstration.  
    - Configuration: No parameters; triggers workflow on manual user action.  
    - Inputs: None  
    - Outputs: Triggers next node `Set invoice data`.  
    - Edge Cases: None.

  - **Set invoice data**  
    - Type: Set  
    - Role: Defines initial variables such as customer email and Price ID for the invoice.  
    - Configuration: Sets key-value pairs, e.g., `email: test@example.com`, `priceId: price_fromStripeDashboard`.  
    - Expressions: Likely uses static values here but can be replaced with dynamic inputs.  
    - Inputs: From manual trigger  
    - Outputs: Passes data to `Get Stripe Customer`.  
    - Edge Cases: Incorrect or missing email or Price ID will cause downstream failures.

  - **Get Stripe Customer**  
    - Type: HTTP Request  
    - Role: Queries Stripe API to find an existing customer by email.  
    - Configuration:  
      - Method: GET  
      - URL: Stripe API endpoint for customer search (e.g., `/v1/customers?email=...`)  
      - Authentication: HTTP Basic Auth with Stripe secret key in password field, username empty.  
    - Expressions: Uses email from `Set invoice data` node to build query.  
    - Inputs: From `Set invoice data`  
    - Outputs: Passes customer data (if any) to `Customer exists?` node.  
    - Edge Cases:  
      - API authentication failure (invalid key)  
      - No customer found (empty result)  
      - API rate limits or network errors.

#### 2.2 Customer Lookup and Creation

- **Overview:**  
  Evaluates if a customer exists based on the search result; if not, creates a new customer in Stripe.

- **Nodes Involved:**  
  - Customer exists? (If node)  
  - Set customer id (Set node)  
  - Create customer (HTTP Request)

- **Node Details:**

  - **Customer exists?**  
    - Type: If  
    - Role: Checks if the Stripe customer search returned any existing customer.  
    - Configuration: Condition checks if the response data array length > 0 or if customer ID exists.  
    - Inputs: From `Get Stripe Customer`  
    - Outputs:  
      - True branch: Customer found → goes to `Set customer id`  
      - False branch: Customer not found → goes to `Create customer`  
    - Edge Cases: Expression failures if response format changes.

  - **Set customer id**  
    - Type: Set  
    - Role: Extracts and sets the customer ID from the search result for use in invoice creation.  
    - Configuration: Sets a variable (e.g., `customerId`) from the first customer object in the search results.  
    - Inputs: True branch from `Customer exists?`  
    - Outputs: Passes customer ID to `Create invoice`.  
    - Edge Cases: Empty or malformed customer data.

  - **Create customer**  
    - Type: HTTP Request  
    - Role: Creates a new customer in Stripe using the provided email.  
    - Configuration:  
      - Method: POST  
      - URL: Stripe API endpoint `/v1/customers`  
      - Body Parameters: email from `Set invoice data`  
      - Authentication: HTTP Basic Auth with Stripe secret key.  
    - Inputs: False branch from `Customer exists?`  
    - Outputs: Passes new customer data to `Create invoice`.  
    - Edge Cases:  
      - API errors (invalid email, rate limits)  
      - Authentication errors.

#### 2.3 Invoice Creation and Line Item Addition

- **Overview:**  
  Creates a draft invoice for the identified customer and adds a product line item using a Stripe Price ID.

- **Nodes Involved:**  
  - Create invoice (HTTP Request)  
  - Add item to invoice (HTTP Request)

- **Node Details:**

  - **Create invoice**  
    - Type: HTTP Request  
    - Role: Creates a draft invoice linked to the customer ID.  
    - Configuration:  
      - Method: POST  
      - URL: Stripe API endpoint `/v1/invoices`  
      - Body Parameters: `customer` set to customer ID from previous node  
      - Authentication: HTTP Basic Auth with Stripe secret key.  
    - Inputs: From `Set customer id` or `Create customer`  
    - Outputs: Passes invoice data to `Add item to invoice`.  
    - Edge Cases:  
      - Invalid customer ID  
      - API errors or rate limits.

  - **Add item to invoice**  
    - Type: HTTP Request  
    - Role: Adds a line item to the invoice using a Stripe Price ID.  
    - Configuration:  
      - Method: POST  
      - URL: Stripe API endpoint `/v1/invoiceitems`  
      - Body Parameters:  
        - `invoice`: invoice ID from `Create invoice`  
        - `price`: Price ID from initial data  
      - Authentication: HTTP Basic Auth with Stripe secret key.  
    - Inputs: From `Create invoice`  
    - Outputs: Passes updated invoice data to `Finalize invoice`.  
    - Edge Cases:  
      - Invalid Price ID or inactive price  
      - Invoice already finalized or locked.

#### 2.4 Invoice Finalization

- **Overview:**  
  Finalizes the invoice to make it ready for payment collection or sending to the customer.

- **Nodes Involved:**  
  - Finalize invoice (HTTP Request)

- **Node Details:**

  - **Finalize invoice**  
    - Type: HTTP Request  
    - Role: Changes invoice status from draft to finalized.  
    - Configuration:  
      - Method: POST  
      - URL: Stripe API endpoint `/v1/invoices/{invoice_id}/finalize`  
      - Path Parameter: invoice ID from `Add item to invoice`  
      - Authentication: HTTP Basic Auth with Stripe secret key.  
    - Inputs: From `Add item to invoice`  
    - Outputs: Final output of the workflow (finalized invoice data).  
    - Edge Cases:  
      - Invoice already finalized  
      - API errors or authentication failure.

---

### 3. Summary Table

| Node Name                 | Node Type          | Functional Role                          | Input Node(s)              | Output Node(s)           | Sticky Note                                                                                     |
|---------------------------|--------------------|----------------------------------------|----------------------------|--------------------------|------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger     | Entry point to start workflow manually | None                       | Set invoice data          |                                                                                                |
| Set invoice data          | Set                | Sets initial customer email and Price ID | When clicking ‘Test workflow’ | Get Stripe Customer       |                                                                                                |
| Get Stripe Customer       | HTTP Request       | Searches Stripe for existing customer by email | Set invoice data           | Customer exists?          |                                                                                                |
| Customer exists?          | If                 | Checks if customer exists in Stripe    | Get Stripe Customer         | Set customer id (true), Create customer (false) |                                                                                                |
| Set customer id           | Set                | Extracts customer ID from search result | Customer exists? (true)     | Create invoice            |                                                                                                |
| Create customer           | HTTP Request       | Creates new Stripe customer if none found | Customer exists? (false)    | Create invoice            |                                                                                                |
| Create invoice            | HTTP Request       | Creates draft invoice for customer     | Set customer id, Create customer | Add item to invoice       |                                                                                                |
| Add item to invoice       | HTTP Request       | Adds product line item to invoice      | Create invoice              | Finalize invoice          |                                                                                                |
| Finalize invoice          | HTTP Request       | Finalizes invoice to make it payable   | Add item to invoice         | None                     |                                                                                                |
| Sticky Note              | Sticky Note        | (Empty content)                        | None                       | None                     |                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `When clicking ‘Test workflow’`  
   - Type: Manual Trigger  
   - No parameters needed.

2. **Create Set Node for Invoice Data**  
   - Name: `Set invoice data`  
   - Type: Set  
   - Add fields:  
     - `email`: Set to a test email like `test@example.com` (replace with dynamic input later)  
     - `priceId`: Set to your Stripe Price ID, e.g., `price_fromStripeDashboard`  
   - Connect output of Manual Trigger to this node.

3. **Create HTTP Request Node to Get Stripe Customer**  
   - Name: `Get Stripe Customer`  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://api.stripe.com/v1/customers?email={{$json["email"]}}` (use expression to insert email)  
   - Authentication: HTTP Basic Auth  
     - Username: leave empty  
     - Password: your Stripe secret API key  
   - Connect output of `Set invoice data` to this node.

4. **Create If Node to Check Customer Existence**  
   - Name: `Customer exists?`  
   - Type: If  
   - Condition: Check if `{{$json["data"].length}} > 0` (assuming Stripe returns a `data` array)  
   - Connect output of `Get Stripe Customer` to this node.

5. **Create Set Node to Extract Customer ID**  
   - Name: `Set customer id`  
   - Type: Set  
   - Set field: `customerId` to `{{$json["data"][0]["id"]}}` (first customer ID)  
   - Connect True output of `Customer exists?` to this node.

6. **Create HTTP Request Node to Create Customer**  
   - Name: `Create customer`  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.stripe.com/v1/customers`  
   - Authentication: HTTP Basic Auth (same as before)  
   - Body Parameters (form):  
     - `email`: `{{$json["email"]}}`  
   - Connect False output of `Customer exists?` to this node.

7. **Create HTTP Request Node to Create Invoice**  
   - Name: `Create invoice`  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.stripe.com/v1/invoices`  
   - Authentication: HTTP Basic Auth  
   - Body Parameters (form):  
     - `customer`: Use expression:  
       - If coming from `Set customer id`: `{{$json["customerId"]}}`  
       - If coming from `Create customer`: `{{$json["id"]}}`  
   - Connect output of `Set customer id` and `Create customer` to this node.

8. **Create HTTP Request Node to Add Item to Invoice**  
   - Name: `Add item to invoice`  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.stripe.com/v1/invoiceitems`  
   - Authentication: HTTP Basic Auth  
   - Body Parameters (form):  
     - `invoice`: `{{$json["id"]}}` (invoice ID from previous node)  
     - `price`: `{{$json["priceId"]}}` (from initial Set node)  
   - Connect output of `Create invoice` to this node.

9. **Create HTTP Request Node to Finalize Invoice**  
   - Name: `Finalize invoice`  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.stripe.com/v1/invoices/{{$json["invoice"]}}/finalize`  
   - Authentication: HTTP Basic Auth  
   - Connect output of `Add item to invoice` to this node.

10. **Verify Connections:**  
    - Manual Trigger → Set invoice data → Get Stripe Customer → Customer exists?  
    - Customer exists? True → Set customer id → Create invoice → Add item to invoice → Finalize invoice  
    - Customer exists? False → Create customer → Create invoice → Add item to invoice → Finalize invoice

11. **Credential Setup:**  
    - Create HTTP Basic Auth credential in n8n with:  
      - Username: (leave empty)  
      - Password: Your Stripe secret API key (restricted key with Customers, Invoices, Invoice Items permissions)  

12. **Test Workflow:**  
    - Run manually with test email and Price ID.  
    - Confirm customer creation or retrieval, invoice generation, and finalization in Stripe dashboard.

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                                                                         |
|------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| Use a restricted Stripe API key with read/write permissions on Customers, Invoices, and Invoice Items for security best practices. | Stripe API key setup instructions in Stripe dashboard.                                                |
| Replace static email and Price ID with dynamic inputs from your actual data source (e.g., webhook payload). | Customization instructions in workflow description.                                                   |
| Stripe API documentation for customers: https://stripe.com/docs/api/customers                   | Official Stripe API reference.                                                                         |
| Stripe API documentation for invoices: https://stripe.com/docs/api/invoices                     | Official Stripe API reference.                                                                         |
| Stripe API documentation for invoice items: https://stripe.com/docs/api/invoiceitems            | Official Stripe API reference.                                                                         |
| Ensure HTTP Basic Auth is selected in HTTP Request nodes with the API key in the password field and username empty to avoid authentication errors. | Troubleshooting section in workflow description.                                                      |
| Test mode vs live mode: Use Stripe test keys for development and live keys for production to avoid unintended charges. | Stripe environment setup best practices.                                                              |

---

This documentation provides a complete, stepwise understanding of the workflow, enabling reproduction, modification, and troubleshooting for efficient Stripe customer and invoice automation.