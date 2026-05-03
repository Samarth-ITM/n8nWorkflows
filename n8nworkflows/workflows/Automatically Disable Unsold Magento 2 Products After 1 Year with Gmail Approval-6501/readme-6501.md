Automatically Disable Unsold Magento 2 Products After 1 Year with Gmail Approval

https://n8nworkflows.xyz/workflows/automatically-disable-unsold-magento-2-products-after-1-year-with-gmail-approval-6501


# Automatically Disable Unsold Magento 2 Products After 1 Year with Gmail Approval

### 1. Workflow Overview

This workflow automates the identification and deactivation of Magento 2 products that have not been sold for at least one year. It is designed to help merchants maintain an optimized product catalog by disabling stale inventory items after managerial approval via Gmail. The core logic involves fetching order data from Magento, extracting sold SKUs, comparing them to all available SKUs, identifying unsold products, sending an approval request email, and upon approval, disabling those products and notifying the team with a report.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger and Date Calculation**: Periodic initiation of the workflow and calculation of the cutoff date (1 year ago).
- **1.2 Fetch and Process Order Data**: Retrieve orders since the cutoff date and extract sold SKUs.
- **1.3 Identify Unsold Products**: Fetch all product SKUs and filter those not sold in the last year.
- **1.4 Build and Send Approval Email**: Prepare an HTML report of unsold products and send an approval request via Gmail.
- **1.5 Process Approval Response**: Handle manager’s approval or rejection.
- **1.6 Disable Unsold Products**: For approved products, disable them in Magento and send a final report email.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger and Date Calculation

- **Overview:**  
Triggers the workflow monthly at 8 AM and calculates the date exactly one year ago in ISO format to use as a filter for order data.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Calculate date an year ago (ISO)

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Initiates the workflow monthly at 08:00  
    - Configuration: Interval set to 1 month, trigger at 8 AM  
    - Inputs: None (trigger node)  
    - Outputs: Triggers next node every month  
    - Potential Failures: None significant, but workflow will not trigger if n8n instance is offline.

  - **Calculate date an year ago (ISO)**  
    - Type: Code  
    - Role: Calculates the ISO date string for 12 months ago  
    - Configuration: JavaScript code subtracts 12 months from current date and returns the date string (YYYY-MM-DD)  
    - Key Expression: `date.setMonth(date.getMonth() - 12)`  
    - Inputs: Trigger from Schedule Trigger  
    - Outputs: JSON with property `sixMonthsAgo` (misnamed, actually 12 months ago)  
    - Edge Cases: Date calculation handles month boundaries correctly.

---

#### 1.2 Fetch and Process Order Data

- **Overview:**  
Fetches orders created since the computed cutoff date and extracts the SKUs of products sold in those orders.

- **Nodes Involved:**  
  - Fetch Orders since 1 year ago  
  - If (to check if orders exist)  
  - Extract Sold SKUs from orders

- **Node Details:**

  - **Fetch Orders since 1 year ago**  
    - Type: HTTP Request  
    - Role: Calls Magento 2 REST API to get orders since cutoff date  
    - Configuration:  
      - URL dynamically built using `sixMonthsAgo` date, with filtering on `created_at` field and page size 0 (to fetch all)  
      - Authentication: HTTP Bearer (generic credential)  
    - Inputs: Date from previous node  
    - Outputs: JSON with orders array under `items`  
    - Edge Cases:  
      - API failure or auth errors, network issues  
      - Large response sizes (pagination not handled explicitly)  

  - **If (Check orders length)**  
    - Type: If  
    - Role: Checks if any orders were returned (`items.length > 0`)  
    - Configuration: Boolean condition on `$json.items.length` being true (non-zero)  
    - Inputs: Orders data  
    - Outputs: Proceeds only if orders exist; otherwise, workflow terminates or halts  
    - Edge Cases: Empty orders response, malformed data  

  - **Extract Sold SKUs from orders**  
    - Type: Code  
    - Role: Parses orders and extracts unique SKUs of sold products into a set  
    - Configuration: JavaScript iterates over each order’s items, collects SKUs  
    - Inputs: Orders JSON from If node  
    - Outputs: JSON with array `soldSkus` containing unique sold SKUs  
    - Edge Cases: Orders without items, missing SKU fields  

---

#### 1.3 Identify Unsold Products

- **Overview:**  
Fetches all product SKUs from Magento and filters out those that were sold in the last year, resulting in a list of unsold products.

- **Nodes Involved:**  
  - Get All Product Skus  
  - Filter products NOT sold in last year  
  - Merge (to combine data branches)  
  - Build Decision Email

- **Node Details:**

  - **Get All Product Skus**  
    - Type: HTTP Request  
    - Role: Retrieves full product list from Magento 2 REST API  
    - Configuration:  
      - URL: `/rest/V1/products?searchCriteria[pageSize]=0` to fetch all products  
      - Authentication: HTTP Bearer (generic credential)  
    - Inputs: Sold SKUs list  
    - Outputs: JSON array of all product items  
    - Edge Cases: Large product catalogs requiring pagination (not explicitly handled)  

  - **Filter products NOT sold in last year**  
    - Type: Code  
    - Role: Filters all products to keep only those whose SKUs are not in the sold SKUs list  
    - Configuration: JavaScript compares all SKUs against sold SKUs set  
    - Inputs: All products and sold SKUs (via expression)  
    - Outputs: JSON with array `unsold` of unsold product objects  
    - Edge Cases: Empty product list, missing SKU fields  

  - **Merge**  
    - Type: Merge  
    - Role: Combines the filtered unsold products with the original data stream, choosing the second input branch’s data  
    - Configuration: Mode “chooseBranch” with useDataOfInput = 2 (takes data from second input)  
    - Inputs: Outputs from filter node and from If1 node (approval)  
    - Outputs: Combined data for next steps  

  - **Build Decision Email**  
    - Type: Code  
    - Role: Builds an HTML table summarizing unsold products for approval email  
    - Configuration:  
      - Defines headers: sku, name, price, status  
      - Creates an HTML table with one row per unsold product  
      - Escapes HTML special characters to avoid rendering issues  
    - Inputs: Unsold products array  
    - Outputs: JSON with `htmlBody`, `count`, and `subject` fields for email  
    - Edge Cases: No unsold products (empty table), missing product fields  

---

#### 1.4 Build and Send Approval Email

- **Overview:**  
Sends an approval request email to a designated Gmail account including the unsold products report, and waits for an approval or rejection response.

- **Nodes Involved:**  
  - Gmail User for Approval

- **Node Details:**

  - **Gmail User for Approval**  
    - Type: Gmail node (send and wait for approval)  
    - Role: Sends email with unsold products report and waits for double approval or decline from user  
    - Configuration:  
      - Send To: `kmyprojects@gmail.com`  
      - Subject dynamically includes count of unsold products  
      - Message body: explanatory text + embedded HTML table from `htmlBody`  
      - Approval options: double approval required  
      - No attribution appended to email  
    - Inputs: Email content JSON from previous node  
    - Outputs: JSON including approval decision under `data.approved`  
    - Edge Cases: Email delivery failure, no user response within timeout, malformed HTML content  

---

#### 1.5 Process Approval Response

- **Overview:**  
Processes the manager’s response. If approved, it proceeds to disable each unsold product; otherwise, no changes are made.

- **Nodes Involved:**  
  - If1 (approval check)  
  - Merge  
  - Split Out  
  - Loop Over Items

- **Node Details:**

  - **If1**  
    - Type: If  
    - Role: Checks if the approval response is true (approved)  
    - Configuration: Boolean condition on `$json.data.approved`  
    - Inputs: Gmail approval output  
    - Outputs: Proceeds only if approved; else halts or skips disabling  

  - **Merge**  
    - Type: Merge  
    - Role: Combines approved unsold product data for processing  
    - Configuration: Mode “chooseBranch” with data from second input branch  
    - Inputs: Filtered unsold products and approval node outputs  
    - Outputs: Combined data with product details for further processing  

  - **Split Out**  
    - Type: Split Out  
    - Role: Splits the `unsold` array into individual items to process one by one  
    - Configuration: Field to split out is `unsold` array property  
    - Inputs: Merged data  
    - Outputs: Single product objects per iteration  

  - **Loop Over Items**  
    - Type: Split In Batches  
    - Role: Processes products sequentially or in batches (default reset false)  
    - Configuration: Default batch size (implicit), no reset of batch index  
    - Inputs: Single product objects from Split Out  
    - Outputs: Each product passed to disable node and aggregate node  

---

#### 1.6 Disable Unsold Products and Reporting

- **Overview:**  
For each approved unsold product, disables it in Magento by setting status to 1, aggregates processed products, builds a final report, and sends a notification email.

- **Nodes Involved:**  
  - Disable Products  
  - Aggregate  
  - Built Reporting Email  
  - Send a message

- **Node Details:**

  - **Disable Products**  
    - Type: HTTP Request  
    - Role: Sends a PUT request to Magento API to update product status (disable)  
    - Configuration:  
      - URL: `/rest/default/V1/products/{{ $json.sku }}` dynamically using SKU  
      - Method: PUT  
      - JSON Body: `{ "product": { "status": 1 } }` (1 typically means “disabled” in Magento)  
      - Authentication: HTTP Bearer (generic credential)  
    - Inputs: Single product JSON from Loop Over Items  
    - Outputs: Response from Magento API  
    - Edge Cases: API errors, product not found, auth failure, SKU mismatch  

  - **Aggregate**  
    - Type: Aggregate  
    - Role: Collects data from processed products for reporting  
    - Configuration:  
      - Aggregate all item data  
      - Include fields: sku, name, price, status  
    - Inputs: Processed products stream  
    - Outputs: Aggregated array of product details  

  - **Built Reporting Email**  
    - Type: Code  
    - Role: Builds an HTML table report for disabled products  
    - Configuration: Similar to decision email node, with headers sku, name, price, status  
    - Inputs: Aggregated product data  
    - Outputs: JSON with `htmlBody`, `count`, and `subject` for reporting email  

  - **Send a message**  
    - Type: Gmail  
    - Role: Sends notification email with list of disabled products to the same Gmail address  
    - Configuration:  
      - To: `kmyprojects@gmail.com`  
      - Subject includes count of disabled products  
      - Message body includes HTML table and count  
      - No attribution appended  
    - Inputs: Reporting email data  
    - Edge Cases: Email delivery failure  

---

### 3. Summary Table

| Node Name                     | Node Type          | Functional Role                                  | Input Node(s)                  | Output Node(s)                  | Sticky Note                                                                                   |
|-------------------------------|--------------------|-------------------------------------------------|-------------------------------|--------------------------------|-----------------------------------------------------------------------------------------------|
| Schedule Trigger               | Schedule Trigger   | Periodic trigger to start workflow              | None                          | Calculate date an year ago (ISO) | # Magento 2 Inventory Cleanup: Disable Unsold Stale Products After 1 Year with Approval...     |
| Calculate date an year ago (ISO) | Code               | Calculates cutoff date (1 year ago)              | Schedule Trigger              | Fetch Orders since 1 year ago   |                                                                                               |
| Fetch Orders since 1 year ago | HTTP Request       | Fetches orders since cutoff date                 | Calculate date an year ago (ISO) | If                            | Fetches All Ordered SKUS last 1 Year                                                          |
| If                            | If                 | Checks if orders exist                            | Fetch Orders since 1 year ago | Extract Sold SKUs from orders   |                                                                                               |
| Extract Sold SKUs from orders | Code               | Extracts sold SKUs from orders                    | If                            | Get All Product Skus            |                                                                                               |
| Get All Product Skus           | HTTP Request       | Fetches all product SKUs                          | Extract Sold SKUs from orders  | Filter products NOT sold in last year |                                                                                               |
| Filter products NOT sold in last year | Code          | Filters unsold products                           | Get All Product Skus           | Merge, Build Decision Email     | Filters All UnOrders SKUS last 1 Year                                                         |
| Merge                         | Merge              | Combines unsold products with approval data     | Filter products NOT sold in last year, If1 | Split Out                    |                                                                                               |
| Build Decision Email          | Code               | Builds HTML report for approval email            | Filter products NOT sold in last year | Gmail User for Approval         |                                                                                               |
| Gmail User for Approval       | Gmail              | Sends approval request email and waits response | Build Decision Email           | If1                           |                                                                                               |
| If1                          | If                 | Checks approval response                          | Gmail User for Approval        | Merge                         | If Approved, Processes the Data                                                               |
| Split Out                    | Split Out          | Splits unsold products array into individual items | Merge                         | Loop Over Items               |                                                                                               |
| Loop Over Items               | Split In Batches   | Processes products sequentially                   | Split Out                     | Aggregate, Disable Products     | Disable Products and Reports via email                                                       |
| Disable Products              | HTTP Request       | Disables product in Magento via API               | Loop Over Items               | Loop Over Items                |                                                                                               |
| Aggregate                    | Aggregate          | Aggregates disabled product data                   | Loop Over Items               | Built Reporting Email          |                                                                                               |
| Built Reporting Email         | Code               | Builds HTML report for disabled products          | Aggregate                    | Send a message                |                                                                                               |
| Send a message               | Gmail              | Sends notification email about disabled products | Built Reporting Email          | None                          |                                                                                               |
| Sticky Note                  | Sticky Note        | Comment: Fetches All Ordered SKUS last 1 Year    | None                         | None                         | Fetches All Ordered SKUS last 1 Year                                                          |
| Sticky Note1                 | Sticky Note        | Comment: Filters All UnOrders SKUS last 1 Year   | None                         | None                         | Filters All UnOrders SKUS last 1 Year                                                         |
| Sticky Note2                 | Sticky Note        | Comment: If Approved, Processes the Data          | None                         | None                         | If Approved, Processes the Data                                                               |
| Sticky Note3                 | Sticky Note        | Comment: Disable Products and Reports via email   | None                         | None                         | Disable Products and Reports via email                                                       |
| Sticky Note4                 | Sticky Note        | Project description and purpose                    | None                         | None                         | # Magento 2 Inventory Cleanup: Disable Unsold Stale Products After 1 Year with Approval...    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node:**  
   - Type: Schedule Trigger  
   - Set to trigger every month at 08:00 AM.

2. **Create Code Node to Calculate Date 1 Year Ago:**  
   - Type: Code  
   - JavaScript code:  
     ```js
     const date = new Date();
     date.setMonth(date.getMonth() - 12);
     return [{ json: { sixMonthsAgo: date.toISOString().split('T')[0] } }];
     ```  
   - Connect output of Schedule Trigger to this node.

3. **Create HTTP Request Node to Fetch Orders:**  
   - Type: HTTP Request  
   - URL:  
     ```
     https://magekwik.com/rest/V1/orders?searchCriteria[filter_groups][0][filters][0][field]=created_at&searchCriteria[filter_groups][0][filters][0][value]={{$json["sixMonthsAgo"]}} 00:00:00&searchCriteria[filter_groups][0][filters][0][condition_type]=gteq&searchCriteria[pageSize]=0
     ```  
   - Method: GET (default)  
   - Auth: HTTP Bearer with Magento credentials set up  
   - Connect output of Date Calculation node to this.

4. **Create If Node to Check Orders Existence:**  
   - Type: If  
   - Condition: Boolean true if `{{$json.items.length}}` > 0  
   - Connect output of Fetch Orders node here.

5. **Create Code Node to Extract Sold SKUs:**  
   - Type: Code  
   - JavaScript code:  
     ```js
     const soldSkusSet = new Set();
     const orders = items[0].json.items || [];
     orders.forEach(order => {
       if (order.items && Array.isArray(order.items)) {
         order.items.forEach(item => {
           if(item.sku) soldSkusSet.add(item.sku);
         });
       }
     });
     return [{ json: { soldSkus: Array.from(soldSkusSet) } }];
     ```  
   - Connect If node’s true output to this node.

6. **Create HTTP Request Node to Get All Product SKUs:**  
   - Type: HTTP Request  
   - URL: `https://magekwik.com/rest/V1/products?searchCriteria[pageSize]=0`  
   - Auth: HTTP Bearer with Magento credentials  
   - Connect output of Extract Sold SKUs node here.

7. **Create Code Node to Filter Unsold Products:**  
   - Type: Code  
   - JavaScript code:  
     ```js
     const allProducts =  $input.first().json.items || [];
     const soldSkus = new Set($('Extract Sold SKUs from orders').first().json.soldSkus || []);
     const unsoldProducts = allProducts.filter(product => !soldSkus.has(product.sku));
     return [{ json: { unsold: unsoldProducts } }];
     ```  
   - Connect output of Get All Product SKUs node here.

8. **Create Code Node to Build Approval Email HTML:**  
   - Type: Code  
   - JavaScript code (builds HTML table):  
     ```js
     const headers = ['sku', 'name', 'price', 'status'];
     const unsold = $json.unsold || [];
     let html = `<table border="1" cellpadding="6" cellspacing="0" style="border-collapse: collapse; font-family: Arial, sans-serif; font-size: 14px;">`;
     html += '<tr style="background-color: #f2f2f2;">' + headers.map(h => `<th>${h}</th>`).join('') + '</tr>';
     for (const product of unsold) {
       html += '<tr>' + headers.map(field => {
         const value = product[field] ?? '';
         return `<td>${String(value).replace(/</g, '&lt;').replace(/>/g, '&gt;')}</td>`;
       }).join('') + '</tr>';
     }
     html += '</table>';
     return [{ json: { htmlBody: html, count: unsold.length, subject: `Unsold Products Report (${unsold.length})` } }];
     ```  
   - Connect output of Filter Unsold Products node here.

9. **Create Gmail Node for Approval:**  
   - Type: Gmail  
   - Operation: Send and wait for approval  
   - Send To: `kmyprojects@gmail.com`  
   - Subject: `Approve disabling {{ $json.count }} unsold products`  
   - Message: Explanatory text + embedded `{{ $json.htmlBody }}`  
   - Approval Type: Double approval  
   - No attribution appended  
   - Connect output of Build Approval Email node here.

10. **Create If Node to Check Approval:**  
    - Type: If  
    - Condition: Check if `$json.data.approved` is true  
    - Connect output of Gmail approval node here.

11. **Create Merge Node:**  
    - Type: Merge  
    - Mode: Choose Branch, use input 2 data  
    - Connect output of Filter Unsold Products node to input 1, and output of approval If node to input 2.

12. **Create Split Out Node:**  
    - Type: Split Out  
    - Field to split: `unsold`  
    - Connect output of Merge node here.

13. **Create Split In Batches Node:**  
    - Type: Split In Batches  
    - Options: Reset = false (default)  
    - Connect output of Split Out node here.

14. **Create HTTP Request Node to Disable Product:**  
    - Type: HTTP Request  
    - Method: PUT  
    - URL: `https://magekwik.com/rest/default/V1/products/{{ $json.sku }}`  
    - Body (JSON): `{ "product": { "status": 1 } }`  
    - Auth: HTTP Bearer with Magento credentials  
    - Connect output of Split In Batches node here (first output).

15. **Create Aggregate Node:**  
    - Type: Aggregate  
    - Aggregate All Item Data  
    - Include fields: sku, name, price, status  
    - Connect output of Split In Batches node here (second output).

16. **Create Code Node to Build Final Report Email:**  
    - Type: Code  
    - Similar HTML table build as step 8, but uses aggregated disabled products  
    - Connect output of Aggregate node here.

17. **Create Gmail Node to Send Final Report:**  
    - Type: Gmail  
    - Send To: `kmyprojects@gmail.com`  
    - Subject: `{{ $json.count }} products have been deactivated`  
    - Message: Includes `{{ $json.htmlBody }}`  
    - No attribution appended  
    - Connect output of final report build node here.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                           | Context or Link                                                                                     |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| This workflow is designed for merchants aiming to automate catalog cleanup by disabling unsold Magento 2 products after 1 year, with explicit managerial approval via Gmail. It balances automation and control to optimize inventory.                    | Project purpose                                                                                        |
| Approval email includes detailed product tables and clear instructions for approval or decline, with a 30-minute response window to ensure timely decisions.                                                                                          | Email design and UX consideration                                                                   |
| Magento API pagination is not explicitly handled; for large catalogs or order volumes, consider implementing pagination to avoid timeouts or incomplete data.                                                                                         | Potential scalability improvement                                                                   |
| Product disablement is done by setting status to `1` (assumed “disabled” status in Magento); verify Magento status codes to ensure correct behavior.                                                                                                   | Magento API specifics                                                                                |
| Gmail node uses “send and wait for approval” operation with double approval configured; this requires appropriate Gmail OAuth2 credentials with necessary scopes configured in n8n.                                                                    | Credential setup                                                                                    |
| Workflow logs and error handling are implicit; for production use, consider adding error catching and notifications on failures (e.g. API errors, auth failures, or timeout in approval response).                                                        | Suggested enhancements                                                                              |
| Sticky notes in the workflow provide helpful annotations summarizing major steps, useful for documentation and onboarding.                                                                                                                             | Workflow design aid                                                                                  |
| Link for Gmail node documentation: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.gmail/                                                                                                                                           | Official n8n Gmail node documentation                                                               |

---

**Disclaimer:**  
The provided text is extracted exclusively from an automated n8n workflow. All operations comply with applicable content policies and manipulate only legal, public data. No illegal or offensive content is present.