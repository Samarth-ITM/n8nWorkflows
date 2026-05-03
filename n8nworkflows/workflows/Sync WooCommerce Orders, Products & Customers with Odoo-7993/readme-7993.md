Sync WooCommerce Orders, Products & Customers with Odoo

https://n8nworkflows.xyz/workflows/sync-woocommerce-orders--products---customers-with-odoo-7993


# Sync WooCommerce Orders, Products & Customers with Odoo

### 1. Workflow Overview

This workflow is designed to synchronize WooCommerce orders, products, and customers with the Odoo ERP system. It listens for new order creation events in WooCommerce, processes order and product data, manages customer contacts, and creates corresponding sales orders and order lines in Odoo. The workflow is structured into logical blocks handling event reception, data extraction and transformation, customer synchronization, product verification and creation, and order/sales line creation.

Logical blocks:

- **1.1 Event Trigger and Initial Processing:** Receives WooCommerce order creation webhook and prepares order data for processing.
- **1.2 Customer Synchronization:** Checks if the customer exists in Odoo; creates a new contact if necessary.
- **1.3 Product and Order Line Processing:** Iterates over order line items, verifies product existence in Odoo, creates products or variants if missing, and prepares sales order lines.
- **1.4 Sales Order Creation:** Creates the sales order in Odoo and links the prepared order lines.
- **1.5 Finalization and Response:** Handles any final data processing and ends workflow execution gracefully.

---

### 2. Block-by-Block Analysis

#### 1.1 Event Trigger and Initial Processing

**Overview:**  
This block listens for new order creation events from WooCommerce via webhook, extracts and formats the order data, and prepares it for customer and product processing.

**Nodes Involved:**  
- WooCommerce Trigger Order Create  
- Edit Fields Order Details  
- Check email is exist  
- Code  
- Search Odoo Contact  
- Check-Email  

**Node Details:**  

- **WooCommerce Trigger Order Create**  
  - *Type:* WooCommerce Trigger  
  - *Role:* Entry point; triggers workflow on new WooCommerce order creation.  
  - *Configuration:* Default webhook listening for order creation events.  
  - *Input:* Webhook HTTP request from WooCommerce.  
  - *Output:* Raw WooCommerce order data.  
  - *Errors:* Potential webhook misconfiguration or connectivity issues.

- **Edit Fields Order Details**  
  - *Type:* Set node  
  - *Role:* Extracts and formats necessary order fields for downstream processing.  
  - *Configuration:* Custom mappings of WooCommerce order fields to a normalized structure.  
  - *Input:* WooCommerce order data.  
  - *Output:* Structured order data.  
  - *Errors:* Expression errors if expected fields are missing.

- **Check email is exist**  
  - *Type:* Switch  
  - *Role:* Determines if the customer email exists in the order data to decide next steps.  
  - *Configuration:* Condition checks presence and validity of email field.  
  - *Input:* Structured order data.  
  - *Output:* Branches flow based on email existence.  
  - *Errors:* Incorrect branching if email field malformed or absent.

- **Code**  
  - *Type:* Code (JavaScript)  
  - *Role:* Processes data related to email validation or manipulation.  
  - *Configuration:* Custom JavaScript to verify or transform email data.  
  - *Input:* Order data.  
  - *Output:* Processed email or flags for branching.  
  - *Edge Cases:* Null or malformed email addresses could cause exceptions.

- **Search Odoo Contact**  
  - *Type:* Odoo node  
  - *Role:* Searches Odoo contacts based on customer email to detect existing customer.  
  - *Configuration:* Search parameters on email field in Odoo contacts.  
  - *Input:* Email extracted from order data.  
  - *Output:* Search results indicating existing customer or none.  
  - *Errors:* Odoo API errors, connection timeouts.

- **Check-Email**  
  - *Type:* Code (JavaScript)  
  - *Role:* Further logic to branch workflow depending on search results.  
  - *Configuration:* Evaluates search response to decide if contact exists.  
  - *Input:* Odoo search results.  
  - *Output:* Branches to customer creation or product processing.  
  - *Edge Cases:* Unexpected search results format.

---

#### 1.2 Customer Synchronization

**Overview:**  
This block creates a new contact in Odoo if the customer does not already exist, ensuring customer data is synchronized before order processing.

**Nodes Involved:**  
- Switch  
- Create contact  
- Code8  
- No Operation, do nothing  

**Node Details:**  

- **Switch**  
  - *Type:* Switch  
  - *Role:* Branches workflow based on presence or absence of Odoo customer contact.  
  - *Configuration:* Conditions based on previous node's output.  
  - *Input:* Results from Check-Email node.  
  - *Output:* To Create contact or Code8 nodes.  
  - *Errors:* Misrouting if conditions incorrectly set.

- **Create contact**  
  - *Type:* Odoo node  
  - *Role:* Creates a new customer contact in Odoo using order customer data.  
  - *Configuration:* Maps customer details (name, email, address) to Odoo contact fields.  
  - *Input:* Customer information from order.  
  - *Output:* Confirmation of contact creation.  
  - *Errors:* Odoo API errors, data validation failures.

- **Code8**  
  - *Type:* Code (JavaScript)  
  - *Role:* Processes data for downstream product extraction after contact handling.  
  - *Configuration:* Custom JavaScript to extract product details from order data.  
  - *Input:* Order and/or contact data.  
  - *Output:* Product detail array for next processing block.  
  - *Errors:* Data access errors if order structure unexpected.

- **No Operation, do nothing**  
  - *Type:* NoOp  
  - *Role:* Placeholder for cases where no action is needed (e.g., customer exists).  
  - *Input/Output:* Passes data through unchanged.  
  - *Usage:* Ensures workflow continuity without side effects.

---

#### 1.3 Product and Order Line Processing

**Overview:**  
This block iterates over order line items, verifying product existence in Odoo, creating missing products or variants, and preparing sales order lines accordingly.

**Nodes Involved:**  
- Extract product details from Shopify Trigger (misnamed, actually extracts product info from WooCommerce order)  
- Split Out Line Items  
- Loop Over Items1  
- Get Product  
- Check Product Exist1  
- Create an item  
- Code6  
- Response and move next  
- Only for product response not used  
- Get many items1  
- Create Sales Order  
- Loop Over Items2  
- Get Product Variant  
- Sale Order Line1  
- Replace Me1  

**Node Details:**  

- **Extract product details from Shopify Trigger**  
  - *Type:* Code (JavaScript)  
  - *Role:* Extracts product details from the WooCommerce order payload (despite the misleading name).  
  - *Configuration:* Parses order items into individual product objects.  
  - *Input:* Order data.  
  - *Output:* Array of product items for processing.  
  - *Errors:* Parsing errors if order format changes.

- **Split Out Line Items**  
  - *Type:* SplitOut  
  - *Role:* Splits the array of line items into individual items for batch processing.  
  - *Input:* Array of product items.  
  - *Output:* Individual product items.

- **Loop Over Items1**  
  - *Type:* SplitInBatches  
  - *Role:* Iterates over product items, processing each in sequence or batch.  
  - *Input:* Single product items.  
  - *Output:* Passes each item downstream for product verification.  
  - *Errors:* Potential batch processing timeouts.

- **Get Product**  
  - *Type:* Odoo node  
  - *Role:* Searches Odoo for existing product records matching SKU or product ID.  
  - *Input:* Product identifier.  
  - *Output:* Product found or empty.  
  - *Errors:* API connection errors, data consistency issues.

- **Check Product Exist1**  
  - *Type:* If  
  - *Role:* Branches depending on whether product exists in Odoo.  
  - *Input:* Search results from Get Product.  
  - *Output:* To Code6 (product exists) or Create an item (product missing).  
  - *Errors:* Incorrect branching if conditions misconfigured.

- **Create an item**  
  - *Type:* Odoo node  
  - *Role:* Creates a new product record in Odoo if missing.  
  - *Input:* Product details from order.  
  - *Output:* Confirmation of product creation.  
  - *Errors:* Data validation or API errors.

- **Code6**  
  - *Type:* Code (JavaScript)  
  - *Role:* Processes existing product data for creating order line entries.  
  - *Input:* Product data from Odoo.  
  - *Output:* Formatted data for order line creation.  
  - *Errors:* Data access or transformation errors.

- **Response and move next**  
  - *Type:* Code (JavaScript)  
  - *Role:* Handles response data and manages flow control after product processing.  
  - *Input:* Processed product/order line data.  
  - *Output:* Passes data to next stage.  
  - *Errors:* Logic errors if data incomplete.

- **Only for product response not used**  
  - *Type:* Code (JavaScript)  
  - *Role:* Placeholder or legacy node, not actively used in current flow.  
  - *Input/Output:* Passes data through.

- **Get many items1**  
  - *Type:* Odoo node  
  - *Role:* Retrieves bulk items possibly for validation or reference before order creation.  
  - *Input:* Criteria from previous nodes.  
  - *Output:* Bulk product or item data.

- **Create Sales Order**  
  - *Type:* Odoo node  
  - *Role:* Creates the main sales order record in Odoo with customer and order meta info.  
  - *Input:* Customer and order data.  
  - *Output:* Sales order confirmation and ID.  
  - *Errors:* API errors, data validation failures.

- **Loop Over Items2**  
  - *Type:* SplitInBatches  
  - *Role:* Iterates over order lines for variant verification and creation.  
  - *Input:* Order line items.  
  - *Output:* Individual line items.

- **Get Product Variant**  
  - *Type:* Odoo node  
  - *Role:* Gets product variant details from Odoo if applicable.  
  - *Input:* Variant identifiers.  
  - *Output:* Variant product data.

- **Sale Order Line1**  
  - *Type:* Odoo node  
  - *Role:* Creates sales order line records linked to created sales order and products.  
  - *Input:* Product/variant and sales order ID.  
  - *Output:* Confirmation of sales order line creation.  
  - *Errors:* API or data errors.

- **Replace Me1**  
  - *Type:* NoOp  
  - *Role:* Placeholder to maintain workflow structure.  
  - *Input/Output:* Pass-through.

---

#### 1.4 Sales Order Creation

**Overview:**  
This block finalizes the sales order creation by aggregating all processed order lines and linking them to the sales order in Odoo.

**Nodes Involved:**  
- Get many items1  
- Create Sales Order  
- Code3  
- Split Out3  
- Loop Over Items2  

**Node Details:**  

- **Get many items1**  
  - *Role:* Prepares or verifies bulk data before order creation.  
  - *Details:* See above.

- **Create Sales Order**  
  - *Role:* See above.

- **Code3**  
  - *Type:* Code (JavaScript)  
  - *Role:* Processes sales order data post-creation for further line item handling.  
  - *Errors:* Logic exceptions if sales order data malformed.

- **Split Out3**  
  - *Type:* SplitOut  
  - *Role:* Splits sales order data or lines for batch processing.  
  - *Errors:* Data structure errors if unexpected.

- **Loop Over Items2**  
  - *Role:* See above.

---

#### 1.5 Finalization and Response

**Overview:**  
Final nodes handle any last processing steps and ensure the workflow finishes cleanly, optionally returning data or performing no operation.

**Nodes Involved:**  
- Code5  
- Code8  
- No Operation, do nothing  

**Node Details:**  

- **Code5**  
  - *Type:* Code (JavaScript)  
  - *Role:* Final processing logic for order line or product data.  
  - *Errors:* Runtime exceptions if data unexpected.

- **Code8**  
  - *Role:* See above in customer synchronization.

- **No Operation, do nothing**  
  - *Role:* Final no-op node ensuring graceful end.

---

### 3. Summary Table

| Node Name                         | Node Type            | Functional Role                       | Input Node(s)                          | Output Node(s)                       | Sticky Note               |
|----------------------------------|----------------------|------------------------------------|--------------------------------------|------------------------------------|---------------------------|
| WooCommerce Trigger Order Create | WooCommerce Trigger  | Entry point for WooCommerce order   | -                                    | Edit Fields Order Details           |                           |
| Edit Fields Order Details         | Set                  | Format incoming order data          | WooCommerce Trigger Order Create     | Check email is exist                |                           |
| Check email is exist             | Switch               | Branch on customer email presence   | Edit Fields Order Details             | Code, Search Odoo Contact           |                           |
| Code                            | Code                 | Email validation/processing         | Check email is exist                  | Search Odoo Contact                 |                           |
| Search Odoo Contact              | Odoo                 | Search customer in Odoo             | Code                                 | Check-Email                        |                           |
| Check-Email                     | Code                 | Branch depending on contact search  | Search Odoo Contact                   | Switch                            |                           |
| Switch                         | Switch               | Decide to create or skip contact    | Check-Email                         | Create contact, Code8               |                           |
| Create contact                 | Odoo                 | Create new customer contact         | Switch                              | Code8                             |                           |
| Code8                          | Code                 | Extract product details from order  | Switch, Create contact               | Extract product details from Shopify Trigger |                           |
| Extract product details from Shopify Trigger | Code                 | Extract product info from order     | Code8                               | Split Out Line Items               |                           |
| Split Out Line Items           | SplitOut             | Split product array to items        | Extract product details from Shopify Trigger | Loop Over Items1                   |                           |
| Loop Over Items1               | SplitInBatches       | Iterate over line items              | Split Out Line Items                 | Response and move next, Get Product |                           |
| Get Product                   | Odoo                 | Search product in Odoo               | Loop Over Items1                    | Check Product Exist1                |                           |
| Check Product Exist1          | If                   | Branch on product existence          | Get Product                        | Code6, Create an item              |                           |
| Create an item               | Odoo                 | Create missing product in Odoo       | Check Product Exist1                | Code6                             |                           |
| Code6                        | Code                 | Prepare product data for order lines | Check Product Exist1, Create an item | Loop Over Items1                  |                           |
| Response and move next       | Code                 | Manage flow after product processing | Loop Over Items1                   | Only for product response not used |                           |
| Only for product response not used | Code                 | Placeholder node                     | Response and move next             | Get many items1                    |                           |
| Get many items1              | Odoo                 | Bulk retrieve data before order creation | Only for product response not used | Create Sales Order                |                           |
| Create Sales Order           | Odoo                 | Create sales order in Odoo           | Get many items1                    | Code3                             |                           |
| Code3                       | Code                 | Process sales order data             | Create Sales Order                 | Split Out3                       |                           |
| Split Out3                  | SplitOut             | Split sales order data for batch     | Code3                             | Loop Over Items2                  |                           |
| Loop Over Items2            | SplitInBatches       | Iterate over order lines              | Split Out3                       | Code5, Get Product Variant        |                           |
| Get Product Variant         | Odoo                 | Retrieve product variant details      | Loop Over Items2                  | Sale Order Line1                  |                           |
| Sale Order Line1            | Odoo                 | Create sales order lines              | Get Product Variant              | Replace Me1                      |                           |
| Replace Me1                | NoOp                 | Placeholder node                      | Sale Order Line1                | Loop Over Items2                 |                           |
| Code5                     | Code                 | Final order line processing           | Loop Over Items2                | No Operation, do nothing          |                           |
| No Operation, do nothing    | NoOp                 | End node with no operation            | Code5                           | -                                |                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create WooCommerce Trigger Order Create node:**  
   - Type: WooCommerce Trigger  
   - Purpose: Listen for new order creation webhook events.  
   - Setup: Configure webhook URL and select "Order Create" event.

2. **Create Edit Fields Order Details node (Set):**  
   - Connect from WooCommerce Trigger node.  
   - Map WooCommerce order JSON fields to normalized variables (e.g., order ID, customer email).

3. **Create Check email is exist node (Switch):**  
   - Connect from Edit Fields Order Details.  
   - Condition: Check if customer email field exists and is valid.

4. **Create Code node:**  
   - Connect from "email exists" branch of Switch.  
   - Add JavaScript to validate or transform the email.

5. **Create Search Odoo Contact node:**  
   - Connect from Code node.  
   - Configure to search Odoo contacts by email.  
   - Set credentials for Odoo API access.

6. **Create Check-Email node (Code):**  
   - Connect from Search Odoo Contact.  
   - Add JavaScript to check if contact exists (e.g., search result length).  
   - Output boolean or branch signal.

7. **Create Switch node:**  
   - Connect from Check-Email.  
   - Branch: If contact found —> No action; else —> Create contact.

8. **Create Create contact node (Odoo):**  
   - Connect from Switch "create contact" branch.  
   - Configure to create new contact with customer data (name, email).  
   - Use Odoo credentials.

9. **Create Code8 node (Code):**  
   - Connect both from Create contact and Switch node (skip contact creation).  
   - Extract product details array from order data for product processing.

10. **Create Extract product details from Shopify Trigger node (Code):**  
    - Connect from Code8.  
    - Parse and extract each product item from order.

11. **Create Split Out Line Items node (SplitOut):**  
    - Connect from previous node.  
    - Split product array into individual items.

12. **Create Loop Over Items1 node (SplitInBatches):**  
    - Connect from Split Out Line Items.  
    - Configure batch size (default 1 or as needed).

13. **Create Get Product node (Odoo):**  
    - Connect from Loop Over Items1.  
    - Search Odoo products by SKU or product ID.  
    - Use Odoo credentials.

14. **Create Check Product Exist1 node (If):**  
    - Connect from Get Product.  
    - Condition: If product exists, go to Code6; else to Create an item.

15. **Create Create an item node (Odoo):**  
    - Connect from Check Product Exist1 (false branch).  
    - Create new product in Odoo with product details.

16. **Create Code6 node (Code):**  
    - Connect from Check Product Exist1 (true branch) and Create an item node.  
    - Format product data for sales order line creation.

17. **Create Response and move next node (Code):**  
    - Connect from Loop Over Items1 and Code6.  
    - Manage and prepare data for next processing stage.

18. **Create Only for product response not used node (Code):**  
    - Connect from Response and move next.  
    - Pass-through node.

19. **Create Get many items1 node (Odoo):**  
    - Connect from Only for product response not used.  
    - Retrieve bulk product or order-related data as needed.

20. **Create Create Sales Order node (Odoo):**  
    - Connect from Get many items1.  
    - Create sales order with customer and order meta info.

21. **Create Code3 node (Code):**  
    - Connect from Create Sales Order.  
    - Prepare sales order data for line item creation.

22. **Create Split Out3 node (SplitOut):**  
    - Connect from Code3.  
    - Split sales order lines for batch processing.

23. **Create Loop Over Items2 node (SplitInBatches):**  
    - Connect from Split Out3.  
    - Iterate over each sales order line.

24. **Create Get Product Variant node (Odoo):**  
    - Connect from Loop Over Items2.  
    - Retrieve variant product info if applicable.

25. **Create Sale Order Line1 node (Odoo):**  
    - Connect from Get Product Variant.  
    - Create sales order lines linked to sales order and products.

26. **Create Replace Me1 node (NoOp):**  
    - Connect from Sale Order Line1.  
    - Placeholder node for potential future extension.

27. **Create Code5 node (Code):**  
    - Connect from Loop Over Items2.  
    - Final processing of order line data.

28. **Create No Operation, do nothing node (NoOp):**  
    - Connect from Code5.  
    - Final node to gracefully end workflow.

**Credentials required:**  
- WooCommerce API credentials for webhook setup (usually configured in node).  
- Odoo API credentials with write permission for contacts, products, sales orders, and order lines.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                              | Context or Link                                      |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| WooCommerce Trigger node is configured to respond to order creation events via webhook. Ensure WooCommerce is setup to send webhooks to the n8n instance URL.           | WooCommerce Webhook Setup Documentation              |
| Odoo nodes require API credentials with sufficient permissions for contacts, products, and sales orders. Proper API access setup in Odoo is critical.                   | Odoo API Authentication Guide                         |
| The workflow includes placeholders and no-op nodes (e.g., Replace Me1, No Operation) for future customization or extensions.                                            | n8n Workflow Design Best Practices                    |
| Product extraction node is named "Extract product details from Shopify Trigger" but processes WooCommerce data; consider renaming for clarity during customization.      | Naming clarity note                                   |
| Batch processing nodes (SplitInBatches) help avoid API timeouts by handling large order line items in manageable chunks.                                               | n8n Batch Processing Documentation                    |
| For troubleshooting, enable n8n execution logging to track errors in API calls or data transformations, especially in Code nodes.                                      | n8n Logging and Debugging Guide                        |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly available.