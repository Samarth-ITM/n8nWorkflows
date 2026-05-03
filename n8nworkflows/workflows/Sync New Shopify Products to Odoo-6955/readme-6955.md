Sync New Shopify Products to Odoo

https://n8nworkflows.xyz/workflows/sync-new-shopify-products-to-odoo-6955


# Sync New Shopify Products to Odoo

### 1. Workflow Overview

This workflow automates the synchronization of new products created in a Shopify store to an Odoo ERP system. It is designed to listen for new product events in Shopify, check if the product already exists in Odoo, and if not, create the product in Odoo. The workflow prevents duplication by stopping if the product is already found in Odoo.

Logical blocks:

- **1.1 Input Reception:** Trigger node that listens for new product events from Shopify.
- **1.2 Product Existence Check in Odoo:** Searches Odoo to determine if the product already exists.
- **1.3 Conditional Branching:** Decides whether to create a new product or stop processing based on existence.
- **1.4 Product Creation in Odoo:** Creates the product in Odoo if it does not exist.
- **1.5 Workflow Termination for Duplicates:** Ends the execution gracefully if a duplicate is found.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** Receives real-time notifications from Shopify when a new product is added.
- **Nodes Involved:**  
  - Shopify Trigger

- **Node Details:**

  - **Shopify Trigger**  
    - Type: Trigger node; event listener for Shopify.  
    - Configuration: Defaults to listen for product creation events (implicit as no additional parameters set).  
    - Key Expressions: Uses webhook mechanism to receive Shopify product creation events.  
    - Input: None (trigger node).  
    - Output: Product data payload from Shopify.  
    - Version Requirements: Compatible with Shopify API version used in n8n.  
    - Edge Cases:  
      - Missed webhook events if Shopify webhook is deleted or fails.  
      - Authentication errors if Shopify credentials expire.  
      - Data schema changes on Shopify product payload may cause mapping issues.

#### 1.2 Product Existence Check in Odoo

- **Overview:** Queries Odoo to check whether the incoming Shopify product already exists to avoid duplicates.  
- **Nodes Involved:**  
  - Search for Existing Product in Odoo

- **Node Details:**

  - **Search for Existing Product in Odoo**  
    - Type: Odoo API node configured for search/read operations.  
    - Configuration: Set to search products by a unique identifier (likely the Shopify product ID or SKU) passed from the Shopify Trigger.  
    - Key Expressions: Uses expressions to extract Shopify product ID from input data to query Odoo.  
    - Input: Shopify product data from Shopify Trigger.  
    - Output: Search results (empty if product not found).  
    - Version Requirements: Odoo API version compatibility must be ensured.  
    - Edge Cases:  
      - Connection/authentication failures to Odoo.  
      - Query timeouts or malformed queries if product ID is missing or invalid.  
      - Multiple products found (should be handled or prevented by unique constraints).

#### 1.3 Conditional Branching

- **Overview:** Evaluates if the product was found in Odoo and branches execution accordingly.  
- **Nodes Involved:**  
  - Is Product Found?

- **Node Details:**

  - **Is Product Found?**  
    - Type: If node (conditional logic).  
    - Configuration: Condition checks if the output of the Odoo search node contains any product (e.g., if search results array length > 0).  
    - Key Expressions: Expression that inspects the length or presence of returned products.  
    - Input: Output from "Search for Existing Product in Odoo."  
    - Output:  
      - True branch: Product exists.  
      - False branch: Product does not exist.  
    - Version Requirements: None specific.  
    - Edge Cases:  
      - Unexpected data structure in search results causing misinterpretation.  
      - False negatives if search query is incorrect.

#### 1.4 Product Creation in Odoo

- **Overview:** Creates a new product record in Odoo when it does not already exist.  
- **Nodes Involved:**  
  - Create Odoo Product

- **Node Details:**

  - **Create Odoo Product**  
    - Type: Odoo API node configured for create operations.  
    - Configuration: Maps Shopify product fields (e.g., name, SKU, price, description) to Odoo product fields.  
    - Key Expressions: Uses expressions to extract and transform Shopify product data for Odoo.  
    - Input: False branch output from "Is Product Found?" node (new product data).  
    - Output: Confirmation of created product or error response.  
    - Version Requirements: Odoo API write permission and version compatibility.  
    - Edge Cases:  
      - Validation errors if required Odoo fields are missing.  
      - API rate limits or timeouts.  
      - Data type mismatches or missing fields in Shopify payload.

#### 1.5 Workflow Termination for Duplicates

- **Overview:** Stops workflow execution cleanly when duplicate products are detected.  
- **Nodes Involved:**  
  - Stop - Duplicate Found

- **Node Details:**

  - **Stop - Duplicate Found**  
    - Type: NoOp (no operation) node used for graceful workflow stopping.  
    - Configuration: None; simply used to end processing on duplicate detection.  
    - Input: True branch output from "Is Product Found?" node.  
    - Output: None.  
    - Version Requirements: None.  
    - Edge Cases: None significant; serves as logical endpoint.

---

### 3. Summary Table

| Node Name                      | Node Type           | Functional Role                          | Input Node(s)               | Output Node(s)                  | Sticky Note                                  |
|-------------------------------|---------------------|----------------------------------------|-----------------------------|--------------------------------|----------------------------------------------|
| Start                         | Start               | Workflow entry point                    |                             | Shopify Trigger                |                                              |
| Shopify Trigger               | Shopify Trigger     | Listens for new Shopify product events | Start                       | Search for Existing Product in Odoo |                                              |
| Search for Existing Product in Odoo | Odoo               | Searches Odoo for existing product      | Shopify Trigger             | Is Product Found?              |                                              |
| Is Product Found?             | If                  | Checks if product exists in Odoo        | Search for Existing Product in Odoo | Create Odoo Product, Stop - Duplicate Found |                                              |
| Create Odoo Product           | Odoo                | Creates product in Odoo if new          | Is Product Found? (false)    |                              |                                              |
| Stop - Duplicate Found        | NoOp                 | Stops workflow if duplicate product     | Is Product Found? (true)     |                              |                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Start Node**  
   - Add a “Start” node (type: Start). No configuration needed.

2. **Add Shopify Trigger Node**  
   - Node type: Shopify Trigger  
   - Connect Start node output to this node input.  
   - Configure credentials for Shopify API (API key, password, shop domain).  
   - Set trigger event to listen for “Product creation” webhook events.  
   - Save.

3. **Add Odoo Search Node**  
   - Node type: Odoo (search/read operation).  
   - Connect Shopify Trigger output to this node input.  
   - Configure Odoo credentials (URL, database, username, password).  
   - Set model to “product.product” or appropriate product model.  
   - Configure search filters to find product by unique Shopify identifier (e.g., external ID, SKU). Use expression to map Shopify product ID from trigger payload.  
   - Set fields to return relevant product identifiers.

4. **Add If Node for Product Existence Check**  
   - Node type: If  
   - Connect output of Odoo search node to this node input.  
   - Condition: Check if the returned array of products length is greater than 0 (meaning product exists).  
   - Configure two outputs:  
     - True (product found)  
     - False (product not found)

5. **Add Odoo Create Node**  
   - Node type: Odoo (create operation).  
   - Connect False output of If node to this node input.  
   - Configure Odoo credentials (reuse from search node).  
   - Set model to “product.product.”  
   - Map Shopify product fields from trigger payload to Odoo product fields (name, SKU, price, description, etc.) using expressions.  
   - Save.

6. **Add NoOp Node for Duplicate Handling**  
   - Node type: NoOp (no operation).  
   - Connect True output of If node to this node.  
   - This node serves as a graceful end point when the product is already in Odoo.

7. **Review and Test**  
   - Check all credentials and permissions.  
   - Test the workflow by adding a new product in Shopify and monitoring the workflow execution.  
   - Handle errors such as authentication failures, missing fields, or API limits by adding error workflows or notifications if needed.

---

### 5. General Notes & Resources

| Note Content                                                                                 | Context or Link                           |
|----------------------------------------------------------------------------------------------|------------------------------------------|
| Ensure Shopify webhook is registered properly in your Shopify admin to trigger the workflow. | Shopify Webhooks documentation           |
| Odoo API credentials require proper permissions for product read and write operations.       | https://www.odoo.com/documentation       |
| Consider adding error handling nodes for robustness (e.g., retry on API failure).            | n8n documentation on error handling       |
| Workflow prevents duplicates by searching Odoo before creating a product.                    | Best practice for data synchronization   |

---

**Disclaimer:** The content above is generated from an automated n8n workflow designed to synchronize Shopify products to Odoo. It complies with all content policies and handles only legal and public data.