Magento 2: Auto-Fix Missing Image Alt Tags with Product Name

https://n8nworkflows.xyz/workflows/magento-2--auto-fix-missing-image-alt-tags-with-product-name-6715


# Magento 2: Auto-Fix Missing Image Alt Tags with Product Name

### 1. Workflow Overview

This workflow is designed to automate the process of fixing missing alt tags on product images in a Magento 2 store by using the product names as alt text. It targets e-commerce managers or developers who want to improve their site's SEO and accessibility by ensuring every product image has a meaningful alt attribute.

The logical blocks include:

- **1.1 Input Reception:** Manual trigger to start the workflow.
- **1.2 Fetching Product Data:** Retrieving all product SKUs from Magento 2.
- **1.3 Batch Processing:** Splitting the list of SKUs into manageable batches for sequential processing.
- **1.4 Conditional Check:** Determining whether each product's image alt tag is missing or needs updating.
- **1.5 Alt Tag Fixing:** Applying the product name as the alt tag for images missing it via an HTTP request to Magento 2.
- **1.6 Loop Continuation:** Returning to process the next batch until all are processed.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
Starts the workflow upon manual execution.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’

- **Node Details:**  
  - Type: Manual Trigger  
  - Role: Entry point for the workflow; allows user to manually start the process.  
  - Configuration: Default manual trigger, no parameters.  
  - Input Connections: None  
  - Output Connections: To "Get All Product Skus" node  
  - Edge Cases: None, but execution depends on manual initiation.

---

#### 2.2 Fetching Product Data

- **Overview:**  
Retrieves all product SKUs from Magento 2 to identify which products need image alt tag fixes.

- **Nodes Involved:**  
  - Get All Product Skus  
  - Split Out

- **Node Details:**  
  - **Get All Product Skus**  
    - Type: HTTP Request  
    - Role: Calls Magento 2 API endpoint to retrieve all product SKUs.  
    - Configuration: Uses HTTP GET method (likely to the Magento products API endpoint). Credentials (e.g., Magento API auth) should be configured here.  
    - Inputs: From Manual Trigger  
    - Outputs: JSON array of product SKUs  
    - Potential Failures:  
      - Authentication errors (invalid API credentials)  
      - Network timeouts  
      - API rate limits or errors  
  - **Split Out**  
    - Type: Split Out  
    - Role: Separates the array of SKUs into individual items for batch processing.  
    - Inputs: From "Get All Product Skus" node  
    - Outputs: To "Loop Over Items" node  
    - Edge Cases: Empty SKU list leads to no downstream processing.

---

#### 2.3 Batch Processing

- **Overview:**  
Splits the SKU list into batches to efficiently process large product sets without overloading the API or system resources.

- **Nodes Involved:**  
  - Loop Over Items

- **Node Details:**  
  - Type: Split In Batches  
  - Role: Processes SKUs in batches, handles the iteration over each batch for subsequent checks and fixes.  
  - Configuration: Batch size likely set to a reasonable number (default not visible) to balance performance and API limits.  
  - Inputs: From "Split Out" node  
  - Outputs: Two outputs:  
    - Main output (index 0): Batch items to process  
    - Secondary output (index 1): Items for conditional check in "If" node  
  - Edge Cases:  
    - Batch size too large may cause API rate limits.  
    - Empty batches stop the loop.

---

#### 2.4 Conditional Check

- **Overview:**  
Determines which product images lack alt tags or require updates.

- **Nodes Involved:**  
  - If

- **Node Details:**  
  - Type: If  
  - Role: Checks condition(s) on each product image’s alt tag presence or correctness.  
  - Configuration: Logic likely compares alt tag field to empty or null, or checks for a placeholder.  
  - Inputs: From "Loop Over Items" secondary output  
  - Outputs: To "Code" node if condition is true (alt tags missing/incorrect), else likely loops back or skips.  
  - Edge Cases:  
    - Incorrect condition logic may skip valid items or process unnecessary ones.  
    - Null or unexpected data formats can cause expression errors.

---

#### 2.5 Alt Tag Fixing

- **Overview:**  
Updates the product image alt tags by injecting the product name through an API call.

- **Nodes Involved:**  
  - Code  
  - HTTP Request

- **Node Details:**  
  - **Code**  
    - Type: Code (JavaScript)  
    - Role: Constructs or formats the data payload to update alt tags using the product name.  
    - Configuration: Contains logic to extract product name and prepare JSON for API request.  
    - Inputs: From "If" node (items needing fix)  
    - Outputs: To "HTTP Request" node  
    - Edge Cases:  
      - Code errors due to unexpected data structure.  
      - Missing product name data causes incomplete payloads.  
  - **HTTP Request**  
    - Type: HTTP Request  
    - Role: Sends PATCH/PUT request to Magento 2 API to update the product image alt tag.  
    - Configuration:  
      - Method: PATCH or PUT  
      - Endpoint: Magento 2 product media API  
      - Auth: Magento API credentials must be configured.  
      - Payload: From "Code" node output  
    - Inputs: From "Code" node  
    - Outputs: Loops back to "Loop Over Items" node (main output) for next batch processing  
    - Edge Cases:  
      - API errors such as 400 Bad Request if payload malformed  
      - Authentication failures  
      - Network or timeout issues

---

#### 2.6 Loop Continuation

- **Overview:**  
Ensures continuous processing of SKU batches until all products have been checked and updated as needed.

- **Nodes Involved:**  
  - Loop Over Items

- **Node Details:**  
  - Same as in 2.3, but here it receives output from "HTTP Request" and continues processing next batch.  
  - Edge Cases:  
    - Infinite loops if exit condition not properly set.  
    - Stuck batches if error handling not configured.

---

### 3. Summary Table

| Node Name                  | Node Type             | Functional Role                       | Input Node(s)                 | Output Node(s)            | Sticky Note |
|----------------------------|-----------------------|------------------------------------|------------------------------|---------------------------|-------------|
| When clicking ‘Execute workflow’ | Manual Trigger        | Entry point to start workflow         |                              | Get All Product Skus       |             |
| Get All Product Skus        | HTTP Request          | Fetch all product SKUs from Magento 2 | When clicking ‘Execute workflow’ | Split Out                  |             |
| Split Out                  | Split Out             | Split SKU array into individual items | Get All Product Skus          | Loop Over Items            |             |
| Loop Over Items            | Split In Batches      | Process SKUs in batches               | Split Out, HTTP Request       | If (secondary output), Loop Over Items (main output) |             |
| If                        | If                    | Check if product image alt tag missing | Loop Over Items (secondary)   | Code                      |             |
| Code                      | Code                  | Prepare update payload with product name | If                           | HTTP Request               |             |
| HTTP Request              | HTTP Request          | Update product image alt tag via API  | Code                         | Loop Over Items            |             |
| Sticky Note2              | Sticky Note           |                                    |                              |                           |             |
| Sticky Note3              | Sticky Note           |                                    |                              |                           |             |
| Sticky Note4              | Sticky Note           |                                    |                              |                           |             |
| Sticky Note               | Sticky Note           |                                    |                              |                           |             |

*Note:* Sticky notes currently contain no content.

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node:**
   - Name: `When clicking ‘Execute workflow’`
   - Purpose: To start the workflow manually.
   - No parameters needed.

2. **Create an HTTP Request node:**
   - Name: `Get All Product Skus`
   - Purpose: Fetch all product SKUs from Magento 2.
   - Method: GET
   - URL: Magento 2 products API endpoint (e.g., `/rest/V1/products?fields=items[sku]`)
   - Authentication: Configure Magento 2 API credentials (OAuth or token) in the credentials section.
   - Connect input from `When clicking ‘Execute workflow’`.

3. **Create a Split Out node:**
   - Name: `Split Out`
   - Purpose: Split the array of SKUs into individual data items.
   - Connect input from `Get All Product Skus`.

4. **Create a Split In Batches node:**
   - Name: `Loop Over Items`
   - Purpose: Process SKUs in manageable batches.
   - Set batch size (e.g., 10–20 items depending on API limits).
   - Connect input from `Split Out` node.

5. **Create an If node:**
   - Name: `If`
   - Purpose: Check if the product image alt tag is missing or empty.
   - Configure condition:  
     - Expression to check if `alt` attribute is null, empty, or default.
   - Connect input from the secondary output of `Loop Over Items`.

6. **Create a Code node:**
   - Name: `Code`
   - Purpose: Prepare the JSON payload to update the alt tag using the product name.
   - JavaScript code should:  
     - Extract product name and SKU from input data  
     - Build JSON body for the API PATCH/PUT request updating the image alt attribute.
   - Connect input from `If` node’s true output.

7. **Create an HTTP Request node:**
   - Name: `HTTP Request`
   - Purpose: Send update request to Magento 2 to fix image alt tags.
   - Method: PATCH or PUT (depending on Magento API spec)
   - URL: Magento 2 product media update endpoint (e.g., `/rest/V1/products/{sku}/media/{entryId}`)
   - Authentication: Use Magento 2 credentials as above.
   - Body: Use output from `Code` node.
   - Connect input from `Code` node.
   - Connect output back to `Loop Over Items` main input to continue batch processing.

8. **Connect outputs:**
   - From `Loop Over Items` main output (after HTTP Request) back to `If` node (secondary output for checking next batch).
   - This creates a processing loop until all batches are handled.

9. **Credential Setup:**
   - Ensure Magento 2 API credentials are configured in n8n.
   - Credentials must allow read access to products and write access to update product media.

10. **Optional: Add Sticky Notes for documentation.**

---

### 5. General Notes & Resources

| Note Content                                                                                               | Context or Link                                    |
|------------------------------------------------------------------------------------------------------------|---------------------------------------------------|
| This workflow improves SEO and accessibility by ensuring all product images have descriptive alt attributes. | Magento 2 E-Commerce SEO Best Practices            |
| Magento 2 REST API documentation for product media endpoints: https://developer.adobe.com/commerce/webapi/rest/products/media/ | Official Magento 2 API documentation               |
| Consider API rate limits when choosing batch size in "Loop Over Items" to avoid throttling.                 | Magento 2 API performance considerations           |

---

**Disclaimer:** The text provided is exclusively derived from an n8n automated workflow. All data processed is legal and public. The workflow respects all applicable content policies and contains no illegal or offensive elements.