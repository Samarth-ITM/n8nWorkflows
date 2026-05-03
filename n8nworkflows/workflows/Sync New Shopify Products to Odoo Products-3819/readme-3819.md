Sync New Shopify Products to Odoo Products

https://n8nworkflows.xyz/workflows/sync-new-shopify-products-to-odoo-products-3819


# Sync New Shopify Products to Odoo Products

### 1. Workflow Overview

This workflow automates the synchronization of newly created products in Shopify with the Odoo ERP system, ensuring no duplicate products are created in Odoo. It listens for new product creation events in Shopify, checks if the product already exists in Odoo based on a unique identifier (Internal Reference / Default Code), and conditionally creates the product in Odoo if it does not exist.

**Logical Blocks:**

- **1.1 Input Reception:** Triggered by a Shopify webhook when a new product is created.
- **1.2 Existing Product Lookup:** Searches Odoo for any product matching the Shopify product’s unique identifier.
- **1.3 Product Existence Check & Branching:** Determines if the product exists and either stops the workflow or proceeds.
- **1.4 Product Creation in Odoo:** Creates the new product in Odoo with selected fields from Shopify.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  Listens to Shopify’s "products/create" webhook to start the workflow whenever a new product is added in Shopify.

- **Nodes Involved:**  
  - Shopify Trigger

- **Node Details:**  
  - **Shopify Trigger**  
    - Type: Trigger node for Shopify webhook  
    - Configuration:  
      - Topic: `products/create` (fires on product creation)  
      - Authentication: Uses Shopify Access Token OAuth2 credentials  
      - Webhook ID assigned automatically  
    - Input: None (start node)  
    - Output: JSON payload of the newly created Shopify product, including product details and variants  
    - Edge Cases:  
      - Webhook misconfiguration or invalid credentials may cause failure to trigger  
      - Shopify API limits or downtime could delay or prevent triggers  

#### 1.2 Existing Product Lookup

- **Overview:**  
  Queries Odoo to find if a product with the same unique identifier (Internal Reference / Default Code) already exists.

- **Nodes Involved:**  
  - Odoo6

- **Node Details:**  
  - **Odoo6**  
    - Type: Odoo node performing a custom GET operation  
    - Configuration:  
      - Resource: `custom` with customResource set to `product.product`  
      - Operation: `getAll` with limit 1 to retrieve at most one matching product  
      - Filter: Field `default_code` equals Shopify product’s first variant’s `product_id`  
        - Expression: `={{ $('Shopify Trigger').all()[0].json.variants[0].product_id }}`  
    - Input: Shopify Trigger output (used via expression)  
    - Output: Odoo product data if found; empty if not found  
    - Version requirements: Odoo API credentials required  
    - Edge Cases:  
      - If Odoo is down or credentials invalid, query will fail  
      - If Shopify product_id is missing or malformed, query may return no results or error  

#### 1.3 Product Existence Check & Branching

- **Overview:**  
  Processes the Odoo query result to determine if a matching product exists and filters the workflow path accordingly.

- **Nodes Involved:**  
  - Code  
  - Filter2

- **Node Details:**  
  - **Code**  
    - Type: Code node (JavaScript) running once per item  
    - Configuration:  
      - Retrieves Shopify product details from the trigger node directly  
      - Reads Odoo lookup result (`existing_product`)  
      - Sets boolean `existing` to true if Odoo product ID exists, false otherwise  
      - Returns an object with `existing` and full `product_detail` for downstream use  
    - Input: Odoo6 node output  
    - Output: JSON with fields `existing` (boolean) and `product_detail` (full Shopify product JSON)  
    - Edge Cases:  
      - If Odoo6 output is empty or malformed, may cause runtime errors  
      - Expression failures if JSON paths change or data is missing  

  - **Filter2**  
    - Type: Filter node  
    - Configuration:  
      - Condition: Boolean `existing` is true  
      - If true: passes output (indicating product exists)  
      - If false: blocks output, allowing creation node to proceed  
    - Input: Code node output  
    - Output:  
      - True branch: product exists → stops or no further action  
      - False branch: product does not exist → proceeds to product creation  
    - Edge Cases:  
      - Filter logic depends on exact boolean value; unexpected values may cause misrouting  

#### 1.4 Product Creation in Odoo

- **Overview:**  
  Creates a new product in Odoo using Shopify product data if no existing product was found.

- **Nodes Involved:**  
  - Odoo7

- **Node Details:**  
  - **Odoo7**  
    - Type: Odoo node performing a custom create operation  
    - Configuration:  
      - Resource: `custom` with customResource `product.product`  
      - Fields to create:  
        - `name`: Shopify product title  
        - `default_code`: Shopify product first variant `product_id`  
        - `description`: Shopify product description (`body_html`)  
        - `list_price`: Shopify product first variant `price`  
      - Data mapped via expressions from `product_detail` JSON  
    - Input: Filter2 false branch output (product_detail from Code node)  
    - Output: Created Odoo product object (not always outputted)  
    - Credentials: Same Odoo API credentials as Odoo6 node  
    - Edge Cases:  
      - Missing Shopify fields could cause incomplete product data or errors  
      - Odoo API failures (auth, validation, network) could fail creation  
      - Price or product_id format mismatches may cause rejection  

---

### 3. Summary Table

| Node Name       | Node Type           | Functional Role                | Input Node(s)   | Output Node(s) | Sticky Note                                                                                   |
|-----------------|---------------------|-------------------------------|-----------------|----------------|-----------------------------------------------------------------------------------------------|
| Shopify Trigger | Shopify Trigger     | Start workflow on new product | None            | Odoo6          |                                                                                               |
| Odoo6           | Odoo (GET custom)   | Lookup product by internal ref| Shopify Trigger | Code           |                                                                                               |
| Code            | Code (JS)           | Check existence, prepare data | Odoo6           | Filter2        |                                                                                               |
| Filter2         | Filter              | Branch based on existence      | Code            | Odoo7          | If product exists, workflow stops; else proceeds to create product in Odoo                    |
| Odoo7           | Odoo (CREATE custom)| Create new product in Odoo     | Filter2         | None           |                                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Shopify Trigger Node**
   - Type: Shopify Trigger  
   - Configure:  
     - Topic: `products/create`  
     - Authentication: Use Shopify Access Token OAuth2 credentials  
     - Leave webhook ID to auto-generate  
   - Position: Start node (e.g., x=80, y=0)  

2. **Create Odoo Lookup Node (Odoo6)**
   - Type: Odoo node  
   - Set resource to `custom` and customResource to `product.product`  
   - Operation: `getAll`  
   - Limit: 1  
   - Filter: Add filter where field `default_code` equals Shopify product’s first variant `product_id`:  
     Expression: `={{ $('Shopify Trigger').all()[0].json.variants[0].product_id }}`  
   - Credentials: Configure Odoo API credentials with appropriate server URL and authentication  
   - Connect Shopify Trigger main output to Odoo6 main input  

3. **Create Code Node**
   - Type: Code (JavaScript)  
   - Mode: Run once for each item  
   - JavaScript code:  
     ```js
     var product_detail = $('Shopify Trigger').first().json;
     var existing_product = $('Odoo6').item.json;
     return { existing: existing_product.id ? true : false, product_detail: product_detail };
     ```  
   - Connect Odoo6 main output to Code main input  

4. **Create Filter Node (Filter2)**
   - Type: Filter  
   - Condition: Boolean field `existing` equals true  
   - Connect Code main output to Filter2 main input  

5. **Create Odoo Product Creation Node (Odoo7)**
   - Type: Odoo node  
   - Resource: `custom`, customResource: `product.product`  
   - Operation: Create (default for custom resource)  
   - Fields to create or update:  
     - `name`: `={{ $json.product_detail.title }}`  
     - `default_code`: `={{ $json.product_detail.variants[0].product_id }}`  
     - `description`: `={{ $json.product_detail.body_html }}`  
     - `list_price`: `={{ $json.product_detail.variants[0].price }}`  
   - Credentials: Use same Odoo API credentials as Odoo6  
   - Connect Filter2 "false" output (products that do not exist) to Odoo7 main input  

6. **Set Workflow Active and Test**
   - Activate the workflow  
   - Test by creating a new product in Shopify and verify product creation in Odoo or workflow halt if product exists  

---

### 5. General Notes & Resources

| Note Content                                                                                                 | Context or Link                                                       |
|--------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------|
| This workflow prevents duplicate product creation in Odoo by checking the Shopify product's unique reference | Core functionality is ensuring data consistency between Shopify & Odoo |
| Shopify product_id is used as `default_code` in Odoo for unique identification                               | Important for matching products between systems                      |
| Ensure correct OAuth credentials are configured for Shopify and Odoo nodes                                   | Credential setup is critical for API access                          |
| Odoo API endpoint used: `http://148.66.157.208:8069`                                                        | Check network and API availability for this address                  |
| For extended logging or debugging, add additional Code nodes or error handling nodes                         | Helpful for troubleshooting workflow execution issues                |

---

This document fully describes the workflow structure, node configurations, and stepwise instructions for exact reproduction, enabling users or automation agents to understand, maintain, and extend the integration reliably.