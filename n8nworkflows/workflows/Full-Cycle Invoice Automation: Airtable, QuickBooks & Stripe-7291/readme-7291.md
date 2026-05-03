Full-Cycle Invoice Automation: Airtable, QuickBooks & Stripe

https://n8nworkflows.xyz/workflows/full-cycle-invoice-automation--airtable--quickbooks---stripe-7291


# Full-Cycle Invoice Automation: Airtable, QuickBooks & Stripe

### 1. Workflow Overview

This workflow automates the full invoicing cycle by integrating Airtable, QuickBooks, and Stripe. It is designed to streamline invoice creation and payment link generation based on new or updated records in Airtable. The workflow ensures customer data consistency across platforms and updates source records with relevant financial identifiers and payment information.

Logical blocks:

- **1.1 Input Reception and Validation**  
  Detects new or updated Airtable records and filters only those approved for invoicing.

- **1.2 Customer Verification and Creation**  
  Checks if customers exist in QuickBooks and Stripe, creating them if necessary, and merges customer data.

- **1.3 Data Synchronization and Preparation**  
  Updates Airtable with customer IDs, fetches QuickBooks products, filters for relevant items, and prepares invoice data.

- **1.4 Invoice Creation and Payment Link Generation**  
  Creates invoices in QuickBooks and Stripe payment links, then updates Airtable accordingly.

- **1.5 Workflow Completion and Exit Handling**  
  Handles graceful exits for unapproved records and marks workflow completion.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Validation

**Overview:**  
This block triggers the workflow on new Airtable records where the "Created" column changes, then searches for all relevant records and filters only those with status "Approved for Invoicing".

**Nodes Involved:**  
- Airtable Trigger  
- Search records  
- IF - Status Check  
- Exit from workflow (No Operation)

**Node Details:**

- **Airtable Trigger**  
  - Type: Airtable Trigger  
  - Role: Watches for new or changed records in the specified Airtable base/table, triggering the workflow on changes to the "Created" field.  
  - Configuration: Polls every minute; authenticates via Airtable Personal Access Token; uses specific base and table IDs.  
  - Inputs: None (trigger node)  
  - Outputs: Triggers downstream nodes with new/updated records.  
  - Edge Cases: API rate limits or authentication errors; missed triggers if Airtable API unavailable.

- **Search records**  
  - Type: Airtable node  
  - Role: Searches the same Airtable table to retrieve all records for processing.  
  - Configuration: Uses the same base and table as trigger; performs a search operation without filters (fetch all).  
  - Inputs: Trigger from Airtable Trigger  
  - Outputs: Passes full record list downstream.  
  - Edge Cases: Large record sets might cause delays; API limits.

- **IF - Status Check**  
  - Type: If node  
  - Role: Checks if the "Status" field equals "Approved for Invoicing" to filter records.  
  - Configuration: String comparison on the "Status" field.  
  - Inputs: Records from Search records node.  
  - Outputs: True path leads to customer processing; False path leads to workflow exit.  
  - Edge Cases: Status field missing or malformed; case sensitivity in status values.

- **Exit from workflow (No Operation)**  
  - Type: No Operation node  
  - Role: Gracefully ends workflow for records not approved for invoicing.  
  - Inputs: From IF node's False path  
  - Outputs: None  
  - Edge Cases: None, serves as clean exit.

---

#### 1.2 Customer Verification and Creation

**Overview:**  
This block verifies whether customers exist in QuickBooks and Stripe, creates them if absent, and merges customer data for consistent downstream processing.

**Nodes Involved:**  
- QuickBooks - Find Customer  
- IF - Customer Exists?  
- Create a customer (QuickBooks)  
- Merge QBO Customer  
- If - Stripe Customer Id  
- Stripe - Find Customer  
- IF - Stripe Customer Exists?  
- Stripe - Create Customer  
- Merge Stripe Customer  
- Merge - Stripe Customers  
- Search records by email

**Node Details:**

- **QuickBooks - Find Customer**  
  - Type: QuickBooks node  
  - Role: Searches QuickBooks customers by display name to check existence.  
  - Configuration: Uses "getAll" with filter on DisplayName equals client name.  
  - Inputs: Records from IF - Status Check node.  
  - Outputs: Customer data if found; empty if not.  
  - Edge Cases: OAuth token expiration; name mismatches causing false negatives.

- **IF - Customer Exists?**  
  - Type: If node  
  - Role: Checks if the QuickBooks customer ID field is empty to decide if customer needs creation.  
  - Inputs: Output from QuickBooks - Find Customer.  
  - Outputs: True path triggers customer creation; False path continues.  
  - Edge Cases: Null or missing IDs; improper data type handling.

- **Create a customer (QuickBooks)**  
  - Type: QuickBooks node  
  - Role: Creates a new customer record in QuickBooks.  
  - Configuration: Uses client name and email from Airtable; OAuth2 credentials required.  
  - Inputs: True path of IF - Customer Exists?  
  - Outputs: Newly created customer data.  
  - Edge Cases: API rate limits; invalid email format.

- **Merge QBO Customer**  
  - Type: Merge node  
  - Role: Merges data from existing or newly created QuickBooks customers for unified data.  
  - Inputs: From Create a customer and IF - Customer Exists? False branch.  
  - Outputs: Merged customer data.  
  - Edge Cases: Conflicting data merges if inconsistent inputs.

- **If - Stripe Customer Id**  
  - Type: If node  
  - Role: Checks if Stripe Customer ID exists in Airtable to decide on further Stripe processing.  
  - Inputs: Output from Merge QBO Customer.  
  - Outputs: True path searches Stripe; False path skips search.  
  - Edge Cases: Empty or malformed Stripe Customer ID fields.

- **Stripe - Find Customer**  
  - Type: Stripe node  
  - Role: Finds Stripe customer by Stripe Customer ID.  
  - Configuration: Uses the Stripe Customer ID from Airtable record.  
  - Inputs: True path from If - Stripe Customer Id.  
  - Outputs: Customer data if found.  
  - Edge Cases: Invalid Stripe Customer IDs; API rate limits.

- **IF - Stripe Customer Exists?**  
  - Type: If node  
  - Role: Checks if Stripe customer data exists by verifying non-empty DisplayName field.  
  - Inputs: Output from Stripe - Find Customer or Merge - Stripe Customers.  
  - Outputs: True path continues; False path leads to Stripe customer creation.  
  - Edge Cases: Missing or empty DisplayName fields; inconsistent data.

- **Stripe - Create Customer**  
  - Type: Stripe node  
  - Role: Creates a new customer in Stripe with name and email from Airtable.  
  - Configuration: Uses DisplayName and PrimaryEmailAddr.Address fields.  
  - Inputs: False path from IF - Stripe Customer Exists?  
  - Outputs: Newly created Stripe customer data.  
  - Edge Cases: Invalid email; API errors.

- **Merge Stripe Customer**  
  - Type: Merge node  
  - Role: Combines Stripe customer data from creation or search for unified processing.  
  - Inputs: Outputs from Stripe - Create Customer and IF - Stripe Customer Exists? True path.  
  - Outputs: Merged Stripe customer data.  
  - Edge Cases: Merge conflicts.

- **Merge - Stripe Customers**  
  - Type: Merge node (version 3.2)  
  - Role: Merges conditional decision data with Stripe customer retrieval data.  
  - Inputs: From If - Stripe Customer Id and Stripe - Find Customer.  
  - Outputs: Unified Stripe customer data.  
  - Edge Cases: Data conflicts.

- **Search records by email**  
  - Type: Airtable node  
  - Role: Searches Airtable records by client email to fetch full user data.  
  - Configuration: Filters by formula where Client Email equals email from previous node.  
  - Inputs: Output from Merge Stripe Customer.  
  - Outputs: Detailed Airtable user records.  
  - Edge Cases: Email mismatches; empty results.

---

#### 1.3 Data Synchronization and Preparation

**Overview:**  
This block updates Airtable with QuickBooks and Stripe customer IDs, fetches all QuickBooks products, filters relevant products matching Airtable data, and prepares invoice line items.

**Nodes Involved:**  
- Update Quickbooks and Stripe Customer Ids (Airtable)  
- Generate Payment Links (HTTP Request)  
- Get all Quickbook products (HTTP Request)  
- Filter and Return product details (Code node)

**Node Details:**

- **Update Quickbooks and Stripe Customer Ids**  
  - Type: Airtable node  
  - Role: Updates Airtable records with QuickBooks Customer ID and Stripe Customer ID.  
  - Configuration: Matches records by id; updates relevant customer ID fields.  
  - Inputs: Output from Search records by email.  
  - Outputs: Updated Airtable records.  
  - Edge Cases: Update failures due to API errors; mismatched record IDs.

- **Generate Payment Links**  
  - Type: HTTP Request node  
  - Role: Sends POST request to Stripe API to create a payment link using price ID and quantity from Airtable.  
  - Configuration: Uses Stripe API with OAuth or API key authentication; form-urlencoded body.  
  - Inputs: Output from Update Quickbooks and Stripe Customer Ids.  
  - Outputs: JSON with payment link URL.  
  - Edge Cases: API rate limits; invalid price IDs; network errors.

- **Get all Quickbook products**  
  - Type: HTTP Request node  
  - Role: Retrieves all products from QuickBooks company account.  
  - Configuration: GET request to QuickBooks API endpoint with OAuth2 credentials and company ID.  
  - Inputs: Output from Generate Payment Links.  
  - Outputs: JSON list of products.  
  - Edge Cases: OAuth expiration; API errors.

- **Filter and Return product details**  
  - Type: Code node  
  - Role: Filters QuickBooks product list to find product matching Airtable product name; calculates amount based on unit price and quantity.  
  - Configuration: Runs custom JavaScript code per item.  
  - Inputs: Output from Get all Quickbook products.  
  - Outputs: JSON with product Id, Description, and calculated Amount.  
  - Edge Cases: No matching product found; null values for prices; code exceptions.

---

#### 1.4 Invoice Creation and Payment Link Generation

**Overview:**  
This block creates an invoice in QuickBooks using prepared data, updates Airtable with invoice and payment link details, and marks the record as invoiced.

**Nodes Involved:**  
- Create an invoice (QuickBooks)  
- Update Stripe Payment Link and Quickbooks Invoice # (Airtable)  
- Workflow Completed (No Operation)

**Node Details:**

- **Create an invoice**  
  - Type: QuickBooks node  
  - Role: Creates an invoice with line items, amount, and customer reference in QuickBooks.  
  - Configuration: Uses quantity and amount from earlier nodes; customer ID from updated Airtable data.  
  - Inputs: Output from Filter and Return product details.  
  - Outputs: Invoice details including DocNumber.  
  - Edge Cases: Invoice creation failures; invalid customer references.

- **Update Stripe Payment Link and Quickbooks Invoice #**  
  - Type: Airtable node  
  - Role: Updates Airtable record to set Status to "Invoiced", and adds Stripe payment link and QuickBooks invoice number.  
  - Configuration: Matches record by ID; updates respective fields.  
  - Inputs: Output from Create an invoice.  
  - Outputs: Updated Airtable record.  
  - Edge Cases: Update failures; mismatched record IDs.

- **Workflow Completed (No Operation)**  
  - Type: No Operation node  
  - Role: Marks successful end of workflow.  
  - Inputs: From Update Stripe Payment Link and Quickbooks Invoice # node.  
  - Outputs: None.  
  - Edge Cases: None.

---

#### 1.5 Workflow Completion and Exit Handling

**Overview:**  
Ensures clean workflow termination for records not approved for invoicing and indicates successful completion for processed records.

**Nodes Involved:**  
- Exit from workflow (No Operation) [from 1.1]  
- Workflow Completed (No Operation) [from 1.4]

**Node Details:**

- Covered in previous blocks; no additional configuration.

---

### 3. Summary Table

| Node Name                            | Node Type              | Functional Role                                | Input Node(s)                | Output Node(s)                         | Sticky Note                                                                                                                 |
|------------------------------------|------------------------|-----------------------------------------------|------------------------------|---------------------------------------|-----------------------------------------------------------------------------------------------------------------------------|
| Airtable Trigger                   | Airtable Trigger       | Triggers workflow on new Airtable records     | None                         | Search records                        | Step 1: Airtable Trigger 🚦📋 - Triggers workflow on new data in "Created" column.                                           |
| Search records                    | Airtable               | Retrieves all records from Airtable table     | Airtable Trigger             | IF - Status Check                     | Step 2: Airtable Search Records 🔍📋 - Retrieves all relevant data for processing.                                            |
| IF - Status Check                 | If                     | Checks if Status is "Approved for Invoicing" | Search records               | QuickBooks - Find Customer, Exit from workflow | Step 3: Status Check (If Node) ✅❌ - Filters approved records for invoicing.                                                  |
| Exit from workflow                | No Operation           | Graceful exit for unapproved records          | IF - Status Check (False)    | None                                 | Graceful Exit (No-Op Node) 🛑✨ - Ends workflow cleanly for unapproved records.                                               |
| QuickBooks - Find Customer        | QuickBooks             | Finds customer by name in QuickBooks          | IF - Status Check (True)     | IF - Customer Exists?                 | Step 4: Find Customer in QuickBooks 🔍👤 - Locates existing customers to avoid duplicates.                                   |
| IF - Customer Exists?             | If                     | Checks if QuickBooks customer exists           | QuickBooks - Find Customer   | Create a customer, Merge QBO Customer | Step 5: Customer Existence Check (If Node) ❓✅❌ - Decides on customer creation necessity.                                     |
| Create a customer                 | QuickBooks             | Creates new QuickBooks customer                | IF - Customer Exists? (True) | Merge QBO Customer                   | Create Customer in QuickBooks ➕👤 - Adds new customers to QuickBooks.                                                        |
| Merge QBO Customer               | Merge                  | Merges existing/new QuickBooks customer data  | Create a customer, IF - Customer Exists? (False) | If - Stripe Customer Id                | Step 6: Merge Customer Data Node 🔗📊 - Unifies customer data for consistent processing.                                       |
| If - Stripe Customer Id          | If                     | Checks for existing Stripe Customer ID        | Merge QBO Customer           | Stripe - Find Customer, Merge - Stripe Customers | Step 7: Stripe Customer ID Check (If Node) 🔍💳 - Manages Stripe customer existence checks.                                   |
| Stripe - Find Customer           | Stripe                 | Finds Stripe customer by ID                    | If - Stripe Customer Id (True) | Merge - Stripe Customers             | Find Customer in Stripe 🔍💳 - Ensures accurate Stripe customer data.                                                        |
| Merge - Stripe Customers         | Merge                  | Merges Stripe customer data from decision and search | If - Stripe Customer Id (False), Stripe - Find Customer | IF - Stripe Customer Exists?          | Step 8: Merge Stripe Customer Data Node 🔗💳 - Integrates Stripe customer info for further use.                               |
| IF - Stripe Customer Exists?     | If                     | Checks if Stripe customer exists               | Merge - Stripe Customers     | Stripe - Create Customer, Merge Stripe Customer | Step 9: Stripe Customer Existence Check (If Node) 🔍✅❌ - Controls creation of Stripe customers.                             |
| Stripe - Create Customer         | Stripe                 | Creates new Stripe customer                     | IF - Stripe Customer Exists? (False) | Merge Stripe Customer                | Create Customer in Stripe ➕💳 - Adds new customers in Stripe.                                                                |
| Merge Stripe Customer            | Merge                  | Merges new or existing Stripe customer data   | Stripe - Create Customer, IF - Stripe Customer Exists? (True) | Search records by email               | Step 10: Merge New Stripe Customer Data Node 🔗💳 - Combines Stripe customer details.                                         |
| Search records by email          | Airtable               | Searches Airtable records by client email     | Merge Stripe Customer        | Update Quickbooks and Stripe Customer Ids | Step 11: Search Records in Airtable 🔍📋 - Retrieves detailed user data for processing.                                       |
| Update Quickbooks and Stripe Customer Ids | Airtable               | Updates Airtable with QuickBooks and Stripe IDs | Search records by email      | Generate Payment Links               | Step 12: Update Records in Airtable ✏️🔄 - Synchronizes IDs across systems.                                                  |
| Generate Payment Links           | HTTP Request           | Creates Stripe payment link                     | Update Quickbooks and Stripe Customer Ids | Get all Quickbook products           | Step 13: Generate Stripe Payment Link (HTTP Request) 🔗💳 - Automates payment link creation.                                 |
| Get all Quickbook products       | HTTP Request           | Retrieves all QuickBooks products               | Generate Payment Links       | Filter and Return product details    | Step 14: Fetch All Products from QuickBooks (HTTP Request) 📦🔍 - Pulls product catalog.                                      |
| Filter and Return product details | Code                   | Filters products and calculates amount         | Get all Quickbook products   | Create an invoice                   | Step 15: Filter Products by Airtable Data (Code Node) ⚙️🔍 - Matches products and computes invoice amounts.                   |
| Create an invoice                | QuickBooks             | Creates invoice in QuickBooks                    | Filter and Return product details | Update Stripe Payment Link and Quickbooks Invoice # | Step 16: Create Invoice in QuickBooks 🧾✨ - Finalizes billing with official invoice.                                         |
| Update Stripe Payment Link and Quickbooks Invoice # | Airtable               | Updates Airtable with invoice and payment link | Create an invoice            | Workflow Completed                 | Step 17: Update Airtable Records ✏️🔄 - Updates billing status and payment info in Airtable.                                  |
| Workflow Completed              | No Operation           | Marks workflow completion                        | Update Stripe Payment Link and Quickbooks Invoice # | None                                | Step 18: Workflow Completion (No-Op Node) ✅🎉 - Indicates successful end of workflow.                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Airtable Trigger**  
   - Node Type: Airtable Trigger  
   - Configure with your Airtable Base ID and Table ID.  
   - Set to trigger on changes to the "Created" field, polling every minute.  
   - Authenticate using Airtable Personal Access Token.

2. **Add Airtable Search Records Node**  
   - Node Type: Airtable  
   - Configure to use the same base and table as the trigger.  
   - Operation set to "search" without filters to fetch all records.  
   - Connect output of Airtable Trigger to this node.

3. **Add IF Node: Status Check**  
   - Node Type: If  
   - Condition: Check if field "Status" equals "Approved for Invoicing".  
   - Connect output of Search Records node to this IF node.  
   - True path proceeds; False path leads to No Operation exit.

4. **Add No Operation Node for Exit**  
   - Node Type: No Operation  
   - Connect False output of Status Check IF node here for graceful termination.

5. **Add QuickBooks Find Customer Node**  
   - Node Type: QuickBooks  
   - Operation: Get All with filter where DisplayName equals client name from Airtable.  
   - Connect True output of Status Check IF node here.  
   - Authenticate with OAuth2 QuickBooks credentials.

6. **Add IF Node: Customer Exists?**  
   - Node Type: If  
   - Condition: Check if QuickBooks Customer ID is empty.  
   - Connect output of QuickBooks Find Customer here.

7. **Add QuickBooks Create Customer Node**  
   - Node Type: QuickBooks  
   - Operation: Create customer with DisplayName and PrimaryEmailAddr from Airtable.  
   - Connect True output of Customer Exists IF node here.

8. **Add Merge Node: Merge QBO Customer**  
   - Node Type: Merge  
   - Mode: Append  
   - Connect outputs of Create Customer node and False output of Customer Exists IF node here.

9. **Add IF Node: If - Stripe Customer Id**  
   - Node Type: If  
   - Condition: Check if Stripe Customer ID is not empty in Airtable record.  
   - Connect output of Merge QBO Customer here.

10. **Add Stripe Find Customer Node**  
    - Node Type: Stripe  
    - Operation: Find customer by Stripe Customer ID.  
    - Connect True output of If - Stripe Customer Id node here.  
    - Authenticate with Stripe API key.

11. **Add Merge Node: Merge - Stripe Customers**  
    - Node Type: Merge (version 3.2)  
    - Connect False output of If - Stripe Customer Id node and output of Stripe Find Customer node here.

12. **Add IF Node: IF - Stripe Customer Exists?**  
    - Node Type: If  
    - Condition: Check if DisplayName field is not empty in Stripe data.  
    - Connect output of Merge - Stripe Customers here.

13. **Add Stripe Create Customer Node**  
    - Node Type: Stripe  
    - Operation: Create customer with name and email from Airtable.  
    - Connect False output of IF - Stripe Customer Exists? node here.

14. **Add Merge Node: Merge Stripe Customer**  
    - Node Type: Merge  
    - Connect output of Stripe Create Customer node and True output of IF - Stripe Customer Exists? node here.

15. **Add Airtable Search Records By Email Node**  
    - Node Type: Airtable  
    - Operation: Search with filter formula matching Client Email with email in merged Stripe customer data.  
    - Connect output of Merge Stripe Customer here.

16. **Add Airtable Update Node: Update QuickBooks and Stripe Customer Ids**  
    - Node Type: Airtable  
    - Operation: Update records by matching 'id', updating Stripe Customer ID and QuickBooks Customer ID fields.  
    - Connect output of Search Records By Email node here.

17. **Add HTTP Request Node: Generate Payment Links**  
    - Node Type: HTTP Request  
    - Method: POST to https://api.stripe.com/v1/payment_links  
    - Body: Form URL encoded with line item price and quantity from Airtable.  
    - Authenticate with Stripe API key.  
    - Connect output of Update QuickBooks and Stripe Customer Ids here.

18. **Add HTTP Request Node: Get All QuickBooks Products**  
    - Node Type: HTTP Request  
    - Method: GET to QuickBooks API endpoint querying all Items (products).  
    - Authenticate with QuickBooks OAuth2 credentials.  
    - Connect output of Generate Payment Links node here.

19. **Add Code Node: Filter and Return Product Details**  
    - Node Type: Code  
    - JavaScript to match Airtable product name with QuickBooks products and calculate amount.  
    - Runs once per item.  
    - Connect output of Get All QuickBooks Products here.

20. **Add QuickBooks Create Invoice Node**  
    - Node Type: QuickBooks  
    - Operation: Create invoice with line items from filtered product details, linked to customer.  
    - Connect output of Code node here.

21. **Add Airtable Update Node: Update Stripe Payment Link and QuickBooks Invoice #**  
    - Node Type: Airtable  
    - Updates record with Status set to "Invoiced", Stripe payment link URL, and QuickBooks invoice number.  
    - Connect output of Create Invoice node here.

22. **Add No Operation Node: Workflow Completed**  
    - Node Type: No Operation  
    - Connect output of Airtable update node here to mark workflow success.

---

### 5. General Notes & Resources

| Note Content                                                                                                      | Context or Link                                            |
|------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------|
| **Prerequisites:** Setup Airtable with specified columns, connect Airtable using Personal Access Token. QuickBooks OAuth2 credentials and Stripe API keys required. | Sticky Note at workflow start.                              |
| For assistance or customization, contact getstarted@intuz.com or visit https://www.intuz.com/                   | Sticky Note near workflow end.                              |
| Workflow is designed for automated invoicing and payment link generation integrating Airtable, QuickBooks, Stripe. | High-level workflow purpose.                               |

---

**Disclaimer:**  
The provided text is exclusively derived from an n8n automated workflow. It complies strictly with content policies, contains no illegal or protected elements, and handles only legal, public data.