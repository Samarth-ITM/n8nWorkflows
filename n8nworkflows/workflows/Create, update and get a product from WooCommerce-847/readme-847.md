Create, update and get a product from WooCommerce

https://n8nworkflows.xyz/workflows/create--update-and-get-a-product-from-woocommerce-847


# Create, update and get a product from WooCommerce

### 1. Workflow Overview

This workflow automates the process of managing a WooCommerce product by sequentially creating, updating, and retrieving product details. It is designed for use cases where product lifecycle management in WooCommerce is needed programmatically, such as automated inventory updates or product catalog synchronization.

The workflow is logically divided into three main blocks:

- **1.1 Input Trigger:** Manual initiation of the workflow.
- **1.2 Product Creation:** Creates a new product in WooCommerce with specified details.
- **1.3 Product Update:** Updates the newly created product’s stock quantity.
- **1.4 Product Retrieval:** Retrieves the final product details after update to confirm changes.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Trigger

- **Overview:**  
  This block initiates the workflow execution manually by the user.

- **Nodes Involved:**  
  - On clicking 'execute'

- **Node Details:**  
  - **Name:** On clicking 'execute'  
  - **Type:** Manual Trigger  
  - **Role:** Starts the workflow when the user clicks the execute button in the n8n editor or UI.  
  - **Configuration:** No parameters set; uses default manual trigger behavior.  
  - **Expressions/Variables:** None used here.  
  - **Input/Output:** No input; output triggers the next node.  
  - **Version Requirements:** n8n version supporting manual triggers (common in all recent versions).  
  - **Potential Failures:** None expected; unless the user forgets to trigger manually.  
  - **Sub-workflow:** None.

#### 1.2 Product Creation

- **Overview:**  
  Creates a new product in WooCommerce with predefined attributes such as name, description, and price.

- **Nodes Involved:**  
  - WooCommerce

- **Node Details:**  
  - **Name:** WooCommerce  
  - **Type:** WooCommerce node (API integration)  
  - **Role:** Creates a new product in WooCommerce.  
  - **Configuration:**  
    - Product name set to "n8n Sweatshirt"  
    - Description: "Stay warm with this sweatshirt!"  
    - Regular price: 30 (currency depends on WooCommerce settings)  
    - No product images or metadata are provided (empty arrays).  
  - **Expressions/Variables:** None hardcoded; static values used for product creation.  
  - **Input/Output:**  
    - Input: Triggered by manual node output  
    - Output: Product creation response containing product ID and details.  
  - **Credentials:** Requires WooCommerce API credentials named "woocommerce".  
  - **Version Requirements:** Compatible with WooCommerce API version supported by n8n WooCommerce node.  
  - **Potential Failures:**  
    - Authentication errors if credentials are invalid.  
    - API errors if WooCommerce service is unreachable or product creation fails (e.g., duplicate SKU, invalid data).  
  - **Sub-workflow:** None.

#### 1.3 Product Update

- **Overview:**  
  Updates the stock quantity of the product created in the previous step.

- **Nodes Involved:**  
  - WooCommerce1

- **Node Details:**  
  - **Name:** WooCommerce1  
  - **Type:** WooCommerce node  
  - **Role:** Updates an existing product in WooCommerce.  
  - **Configuration:**  
    - Operation: "update"  
    - Product ID: Dynamically taken from the output of the previous WooCommerce node (`{{$node["WooCommerce"].json["id"]}}`)  
    - Updates stock quantity to 100 units.  
  - **Expressions/Variables:** Uses expression to reference product ID from the creation node.  
  - **Input/Output:**  
    - Input: Output from WooCommerce product creation node.  
    - Output: Updated product details.  
  - **Credentials:** Uses same WooCommerce API credentials.  
  - **Version Requirements:** Same as creation node.  
  - **Potential Failures:**  
    - Reference error if product ID is missing or malformed.  
    - API errors if update fails due to invalid product ID or WooCommerce service issues.  
  - **Sub-workflow:** None.

#### 1.4 Product Retrieval

- **Overview:**  
  Fetches the updated product details to confirm the changes applied.

- **Nodes Involved:**  
  - WooCommerce2

- **Node Details:**  
  - **Name:** WooCommerce2  
  - **Type:** WooCommerce node  
  - **Role:** Retrieves product information from WooCommerce.  
  - **Configuration:**  
    - Operation: "get"  
    - Product ID: Dynamically references the product ID from the creation node (`{{$node["WooCommerce"].json["id"]}}`).  
  - **Expressions/Variables:** Uses expression for product ID.  
  - **Input/Output:**  
    - Input: Output from product update node.  
    - Output: Final product details after update.  
  - **Credentials:** Same WooCommerce API credentials.  
  - **Version Requirements:** Same as other WooCommerce nodes.  
  - **Potential Failures:**  
    - Missing or invalid product ID references.  
    - API or connectivity issues.  
  - **Sub-workflow:** None.

---

### 3. Summary Table

| Node Name           | Node Type          | Functional Role             | Input Node(s)         | Output Node(s)       | Sticky Note                                    |
|---------------------|--------------------|-----------------------------|-----------------------|----------------------|------------------------------------------------|
| On clicking 'execute'| Manual Trigger     | Starts workflow execution    | -                     | WooCommerce          |                                                |
| WooCommerce         | WooCommerce node    | Create product in WooCommerce| On clicking 'execute' | WooCommerce1         |                                                |
| WooCommerce1        | WooCommerce node    | Update product stock quantity| WooCommerce           | WooCommerce2         |                                                |
| WooCommerce2        | WooCommerce node    | Retrieve updated product     | WooCommerce1          | -                    |                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a node of type **Manual Trigger**  
   - Name it: `On clicking 'execute'`  
   - No parameters need to be set.

2. **Add WooCommerce Node for Product Creation**  
   - Add a node of type **WooCommerce**  
   - Name it: `WooCommerce`  
   - Set operation to **create** (default, or explicitly if available)  
   - Configure product details:  
     - Name: `n8n Sweatshirt`  
     - Description: `Stay warm with this sweatshirt!`  
     - Regular Price: `30`  
     - Leave images and metadata empty.  
   - Set credentials: Select or create WooCommerce API credentials with access rights to create products in your WooCommerce store.

3. **Connect Manual Trigger to WooCommerce (Creation)**  
   - Connect `On clicking 'execute'` output to `WooCommerce` node input.

4. **Add WooCommerce Node for Product Update**  
   - Add a new WooCommerce node.  
   - Name it: `WooCommerce1`  
   - Set operation to **update**  
   - Set Product ID: Use expression referencing the creation node's output, e.g., `{{$node["WooCommerce"].json["id"]}}`  
   - Set update fields:  
     - Stock Quantity: `100`  
   - Use the same WooCommerce credentials.

5. **Connect Product Creation Node to Product Update Node**  
   - Connect `WooCommerce` node output to `WooCommerce1` node input.

6. **Add WooCommerce Node for Product Retrieval**  
   - Add another WooCommerce node.  
   - Name it: `WooCommerce2`  
   - Set operation to **get**  
   - Set Product ID: Use expression referencing the creation node's output, `{{$node["WooCommerce"].json["id"]}}`  
   - Use the same WooCommerce credentials.

7. **Connect Product Update Node to Product Retrieval Node**  
   - Connect `WooCommerce1` node output to `WooCommerce2` node input.

8. **Save and Test**  
   - Save the workflow.  
   - Manually trigger execution to verify product creation, update, and retrieval work as expected.

---

### 5. General Notes & Resources

| Note Content                                                    | Context or Link                                             |
|-----------------------------------------------------------------|-------------------------------------------------------------|
| WooCommerce API credentials must have appropriate permissions.  | WooCommerce documentation on API keys and permissions.      |
| Ensure WooCommerce store supports REST API and version matches. | https://woocommerce.github.io/woocommerce-rest-api-docs/    |
| Product ID is critical for update and get operations.           | Referenced dynamically from creation node output.           |
| Product price is in store’s default currency.                   | Set in WooCommerce store settings.                           |
| Manual Trigger node must be executed by user to start workflow. | n8n UI manual execution process.                             |

---

This documentation provides a complete understanding of the workflow’s structure, functionality, and detailed guidance for recreation and troubleshooting.