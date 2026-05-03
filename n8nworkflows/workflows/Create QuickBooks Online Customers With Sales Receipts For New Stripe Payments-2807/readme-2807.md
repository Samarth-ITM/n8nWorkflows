Create QuickBooks Online Customers With Sales Receipts For New Stripe Payments

https://n8nworkflows.xyz/workflows/create-quickbooks-online-customers-with-sales-receipts-for-new-stripe-payments-2807


# Create QuickBooks Online Customers With Sales Receipts For New Stripe Payments

### 1. Workflow Overview

This workflow automates the creation of QuickBooks Online (QBO) customers and sales receipts triggered by successful Stripe payment events. It is designed for businesses that want to reduce manual accounting data entry by syncing Stripe payments directly into QuickBooks Online.

The workflow is logically divided into the following blocks:

- **1.1 Stripe Payment Trigger & Customer Retrieval**: Listens for successful payment events from Stripe and fetches detailed customer data from Stripe.
- **1.2 QuickBooks Customer Lookup**: Checks if the Stripe customer already exists in QuickBooks Online by querying customers via email.
- **1.3 Customer Creation or Selection**: Decides whether to create a new QuickBooks customer or use an existing one based on the lookup result.
- **1.4 Sales Receipt Generation**: Creates a sales receipt in QuickBooks Online using payment and customer data.

---

### 2. Block-by-Block Analysis

#### 2.1 Stripe Payment Trigger & Customer Retrieval

- **Overview:**  
  This block initiates the workflow when a new successful payment intent event occurs in Stripe. It then retrieves detailed customer information from Stripe for further processing.

- **Nodes Involved:**  
  - New Payment (Stripe Trigger)  
  - Get Stripe Customer (Stripe API)

- **Node Details:**

  - **New Payment**  
    - Type: Stripe Trigger  
    - Role: Listens for `payment_intent.succeeded` events from Stripe webhook.  
    - Configuration: Subscribed to the `payment_intent.succeeded` event, using Stripe API credentials.  
    - Inputs: External Stripe webhook event.  
    - Outputs: Emits payment intent data including customer ID and payment details.  
    - Edge Cases: Webhook misconfiguration, missing or delayed events, Stripe API downtime.

  - **Get Stripe Customer**  
    - Type: Stripe API node  
    - Role: Fetches detailed customer data from Stripe using the customer ID from the payment event.  
    - Configuration: Uses the `customer` resource with dynamic customer ID from the trigger event (`{{$json.data.object.customer}}`).  
    - Inputs: Payment event data from the trigger node.  
    - Outputs: Detailed Stripe customer JSON including name, email, and metadata.  
    - Edge Cases: Customer ID missing or invalid, Stripe API errors, rate limits.

---

#### 2.2 QuickBooks Customer Lookup

- **Overview:**  
  This block queries QuickBooks Online to check if a customer with the Stripe customer's email already exists.

- **Nodes Involved:**  
  - GET Quickbooks Customer (HTTP Request)  
  - If Customer Exists (IF node)

- **Node Details:**

  - **GET Quickbooks Customer**  
    - Type: HTTP Request  
    - Role: Queries QuickBooks Online customer database filtering by primary email address.  
    - Configuration:  
      - Method: GET  
      - URL: QuickBooks query endpoint with SQL-like query filtering by email (`PrimaryEmailAddr = '{{ $json.email }}'`).  
      - Authentication: OAuth2 with QuickBooks Online credentials.  
    - Inputs: Stripe customer data (email) from previous node.  
    - Outputs: Query response containing matching customers or empty if none found.  
    - Edge Cases: OAuth token expiration, query syntax errors, no matching customer found.

  - **If Customer Exists**  
    - Type: IF node  
    - Role: Checks if the QuickBooks query returned a customer by verifying if the email address exists in the response.  
    - Configuration: Condition checks existence of `QueryResponse.Customer[0].PrimaryEmailAddr.Address`.  
    - Inputs: Output from GET Quickbooks Customer.  
    - Outputs: Two branches — true (customer exists) and false (customer does not exist).  
    - Edge Cases: Unexpected response structure, empty or null fields.

---

#### 2.3 Customer Creation or Selection

- **Overview:**  
  Based on the IF node result, this block either merges data to create a new QuickBooks customer or uses the existing customer data.

- **Nodes Involved:**  
  - Use Stripe Customer (Merge)  
  - Create QuickBooks Customer (QuickBooks node)  
  - Merge Stripe and QuickBooks Data (Merge)  

- **Node Details:**

  - **Use Stripe Customer**  
    - Type: Merge  
    - Role: Passes Stripe customer data forward when no QuickBooks customer exists.  
    - Configuration: Default merge mode, executes once.  
    - Inputs: False branch from IF node and Stripe customer data.  
    - Outputs: Data for customer creation.  
    - Edge Cases: Data mismatch or missing fields.

  - **Create QuickBooks Customer**  
    - Type: QuickBooks node  
    - Role: Creates a new customer in QuickBooks Online using Stripe customer details.  
    - Configuration:  
      - Operation: Create customer  
      - Display Name: From Stripe customer name  
      - Additional Fields: Balance and PrimaryEmailAddr from Stripe data  
      - Authentication: QuickBooks OAuth2  
    - Inputs: Merged Stripe customer data.  
    - Outputs: Newly created QuickBooks customer record.  
    - Edge Cases: API rate limits, validation errors, OAuth token expiry.

  - **Merge Stripe and QuickBooks Data**  
    - Type: Merge  
    - Role: Combines data from Stripe payment, QuickBooks customer query, and IF node results to prepare for sales receipt creation.  
    - Configuration: Number of inputs set to 3.  
    - Inputs: Outputs from GET Quickbooks Customer, IF node, and Use Stripe Customer or Create QuickBooks Customer nodes.  
    - Outputs: Combined data for sales receipt creation.  
    - Edge Cases: Data synchronization issues, missing inputs.

---

#### 2.4 Sales Receipt Generation

- **Overview:**  
  This block creates a sales receipt in QuickBooks Online using the payment details and the resolved QuickBooks customer.

- **Nodes Involved:**  
  - Merge Payment and QuickBooks Customer (Merge)  
  - POST Sales Receipt To QuickBooks (HTTP Request)

- **Node Details:**

  - **Merge Payment and QuickBooks Customer**  
    - Type: Merge  
    - Role: Combines payment data from Stripe and customer data from QuickBooks for final sales receipt creation.  
    - Configuration: Default merge mode, executes once.  
    - Inputs: Payment data and QuickBooks customer data.  
    - Outputs: Combined data for sales receipt.  
    - Edge Cases: Missing or inconsistent data.

  - **POST Sales Receipt To QuickBooks**  
    - Type: HTTP Request  
    - Role: Sends a POST request to QuickBooks Online API to create a sales receipt.  
    - Configuration:  
      - URL: QuickBooks sales receipt endpoint with minorversion=73  
      - Method: POST  
      - Body: JSON constructed dynamically with:  
        - Line items including description, quantity, unit price (converted from cents), item reference (hardcoded as "Subscription" with value "10"), and amount.  
        - Customer reference using QuickBooks customer ID and display name.  
        - Currency code from Stripe payment (uppercased).  
        - Private note referencing Stripe payment intent ID.  
      - Authentication: QuickBooks OAuth2 credentials.  
    - Inputs: Merged payment and customer data.  
    - Outputs: Confirmation of sales receipt creation.  
    - Edge Cases: API errors, invalid data formats, authentication failures, network timeouts.

---

### 3. Summary Table

| Node Name                      | Node Type           | Functional Role                                  | Input Node(s)                  | Output Node(s)                  | Sticky Note                                                                                   |
|--------------------------------|---------------------|-------------------------------------------------|-------------------------------|--------------------------------|----------------------------------------------------------------------------------------------|
| New Payment                    | Stripe Trigger      | Trigger workflow on successful Stripe payment   | -                             | Get Stripe Customer             | Configure Stripe webhook to send `payment_intent.succeeded` events to this workflow.         |
| Get Stripe Customer            | Stripe API          | Retrieve detailed Stripe customer data           | New Payment                   | GET Quickbooks Customer, Use Stripe Customer |                                                                                              |
| GET Quickbooks Customer        | HTTP Request        | Query QuickBooks for customer by email           | Get Stripe Customer           | If Customer Exists, Merge Stripe and QuickBooks Data |                                                                                              |
| If Customer Exists             | IF                  | Check if QuickBooks customer exists              | GET Quickbooks Customer       | Merge Stripe and QuickBooks Data (true), Use Stripe Customer (false) |                                                                                              |
| Use Stripe Customer            | Merge               | Pass Stripe customer data for new customer creation | If Customer Exists (false)    | Create QuickBooks Customer      |                                                                                              |
| Create QuickBooks Customer     | QuickBooks          | Create new customer in QuickBooks                 | Use Stripe Customer           | Merge Payment and QuickBooks Customer |                                                                                              |
| Merge Stripe and QuickBooks Data | Merge             | Combine Stripe payment and QuickBooks customer query data | GET Quickbooks Customer, If Customer Exists, Use Stripe Customer | POST Sales Receipt               |                                                                                              |
| Merge Payment and QuickBooks Customer | Merge          | Combine payment and customer data for sales receipt | Create QuickBooks Customer, Merge Stripe and QuickBooks Data | POST Sales Receipt To QuickBooks |                                                                                              |
| POST Sales Receipt             | HTTP Request        | Create sales receipt in QuickBooks (legacy node) | Merge Stripe and QuickBooks Data | -                              |                                                                                              |
| POST Sales Receipt To QuickBooks | HTTP Request      | Create sales receipt in QuickBooks (final node)  | Merge Payment and QuickBooks Customer | -                              |                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Stripe Trigger Node**  
   - Node Type: Stripe Trigger  
   - Name: `New Payment`  
   - Configure to listen for event: `payment_intent.succeeded`  
   - Connect Stripe credentials.  
   - Position: Start of workflow.

2. **Add Stripe Customer Retrieval Node**  
   - Node Type: Stripe  
   - Name: `Get Stripe Customer`  
   - Resource: Customer  
   - Customer ID: Expression `{{$json.data.object.customer}}` from `New Payment` node output.  
   - Connect Stripe credentials.  
   - Connect output of `New Payment` to this node.

3. **Add HTTP Request Node to Query QuickBooks Customer**  
   - Node Type: HTTP Request  
   - Name: `GET Quickbooks Customer`  
   - Method: GET  
   - URL:  
     ```
     https://sandbox-quickbooks.api.intuit.com/v3/company/9341453851324714/query?query=select * from Customer Where PrimaryEmailAddr = '{{ $json.email }}'&minorversion=73
     ```  
   - Authentication: OAuth2 with QuickBooks Online credentials.  
   - Connect output of `Get Stripe Customer` to this node.

4. **Add IF Node to Check Customer Existence**  
   - Node Type: IF  
   - Name: `If Customer Exists`  
   - Condition: Check if `{{$json.QueryResponse.Customer[0].PrimaryEmailAddr.Address}}` exists (string operation: exists).  
   - Connect output of `GET Quickbooks Customer` to this node.

5. **Add Merge Node to Pass Stripe Customer Data**  
   - Node Type: Merge  
   - Name: `Use Stripe Customer`  
   - Default merge mode, execute once.  
   - Connect false branch of `If Customer Exists` to this node.  
   - Also connect output of `Get Stripe Customer` to this node.

6. **Add QuickBooks Node to Create Customer**  
   - Node Type: QuickBooks  
   - Name: `Create QuickBooks Customer`  
   - Operation: Create customer  
   - Display Name: Expression `{{$input.all()[0].json.name}}` (from Stripe customer)  
   - Additional Fields:  
     - Balance: `{{$input.all()[0].json.balance}}`  
     - PrimaryEmailAddr: `{{$input.all()[0].json.email}}`  
   - Connect output of `Use Stripe Customer` to this node.  
   - Connect QuickBooks OAuth2 credentials.

7. **Add Merge Node to Combine Stripe and QuickBooks Data**  
   - Node Type: Merge  
   - Name: `Merge Stripe and QuickBooks Data`  
   - Number of inputs: 3  
   - Connect outputs of:  
     - `GET Quickbooks Customer`  
     - `If Customer Exists` (true branch)  
     - `Use Stripe Customer` or `Create QuickBooks Customer` (depending on branch)  
   - Connect output to next node.

8. **Add Merge Node to Combine Payment and Customer Data**  
   - Node Type: Merge  
   - Name: `Merge Payment and QuickBooks Customer`  
   - Default merge mode, execute once.  
   - Connect outputs of:  
     - `Create QuickBooks Customer`  
     - `Merge Stripe and QuickBooks Data`  
   - Connect output to next node.

9. **Add HTTP Request Node to Create Sales Receipt in QuickBooks**  
   - Node Type: HTTP Request  
   - Name: `POST Sales Receipt To QuickBooks`  
   - Method: POST  
   - URL:  
     ```
     https://sandbox-quickbooks.api.intuit.com/v3/company/9341453851324714/salesreceipt?minorversion=73
     ```  
   - Authentication: OAuth2 with QuickBooks Online credentials.  
   - Body Type: JSON  
   - JSON Body (use expression editor):  
     ```json
     {
       "Line": [
         {
           "Description": "{{ $json.data.object.description }}",
           "DetailType": "SalesItemLineDetail",
           "SalesItemLineDetail": {
             "TaxCodeRef": { "value": "NON" },
             "Qty": 1,
             "UnitPrice": {{ $json.data.object.amount_received / 100 }},
             "ItemRef": { "name": "Subscription", "value": "10" }
           },
           "Amount": {{ $json.data.object.amount / 100 }},
           "LineNum": 1
         }
       ],
       "CustomerRef": {
         "value": {{ $input.all()[1].json.Id }},
         "name": "{{ $input.all()[1].json.DisplayName }}"
       },
       "CurrencyRef": {
         "value": "{{ $json.data.object.currency.toUpperCase() }}"
       },
       "PrivateNote": "Payment from Stripe Payment Intent ID: {{ $json.data.object.id }}"
     }
     ```  
   - Connect output of `Merge Payment and QuickBooks Customer` to this node.

10. **Finalize Connections and Test**  
    - Ensure all nodes are connected as per the logical flow:  
      `New Payment` → `Get Stripe Customer` → `GET Quickbooks Customer` → `If Customer Exists` → (`Use Stripe Customer` → `Create QuickBooks Customer`) or (true branch) → `Merge Stripe and QuickBooks Data` → `Merge Payment and QuickBooks Customer` → `POST Sales Receipt To QuickBooks`.  
    - Authenticate Stripe and QuickBooks credentials.  
    - Set up Stripe webhook to point to the `New Payment` node webhook URL.  
    - Test with a successful Stripe payment intent event.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                   |
|----------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Configure Stripe webhook to send `payment_intent.succeeded` events to the workflow's webhook URL.  | Stripe Dashboard Webhook Settings                                                                |
| QuickBooks Online API requires OAuth2 authentication; ensure tokens are refreshed periodically.    | QuickBooks Online API Documentation: https://developer.intuit.com/app/developer/qbo/docs/api/accounting/most-commonly-used/account |
| Amounts from Stripe are in cents; conversion to dollars (or main currency unit) is done by dividing by 100. | Important for correct financial data representation.                                             |
| The item reference in sales receipt is hardcoded as "Subscription" with value "10"; customize as needed. | Adjust according to your QuickBooks item catalog.                                               |
| Minorversion=73 is used in QuickBooks API calls to ensure compatibility with recent API features.  | QuickBooks API Versioning Documentation                                                          |

---

This document provides a detailed, structured, and comprehensive reference for understanding, reproducing, and maintaining the "Create QuickBooks Online Customers With Sales Receipts For New Stripe Payments" workflow in n8n.