Paginate Shopify Products with GraphQL Cursor-based Navigation

https://n8nworkflows.xyz/workflows/paginate-shopify-products-with-graphql-cursor-based-navigation-5663


# Paginate Shopify Products with GraphQL Cursor-based Navigation

---
### 1. Workflow Overview

This workflow demonstrates how to paginate through Shopify products using GraphQL cursor-based navigation. It is designed for users who need to retrieve all products from a Shopify store in batches, handling Shopify’s cursor-based pagination manually within n8n. The workflow consists of three logical blocks:

- **1.1 Input Trigger**: Manual start of the workflow.
- **1.2 Shopify GraphQL Query**: Fetches a page of products using a GraphQL query with cursor-based pagination.
- **1.3 Pagination Control and Delay**: Checks if more products exist and waits before fetching the next page, enabling controlled iteration over product pages.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Trigger Block

- **Overview:** This block initiates the workflow manually by a user to start fetching products.
- **Nodes Involved:**  
  - Start Workflow

- **Node Details:**

  - **Start Workflow**
    - **Type & Role:** Manual Trigger node; starts the execution on demand.
    - **Configuration:** No special parameters configured; triggers when manually activated.
    - **Expressions/Variables:** None.
    - **Input/Output:** No input; output triggers the next node.
    - **Version-Specific Requirements:** Compatible with all n8n versions.
    - **Potential Failures:** None expected.
    - **Sub-workflows:** None.

#### 1.2 Shopify GraphQL Query Block

- **Overview:** Executes a Shopify GraphQL query to retrieve a page of products with pagination cursors.
- **Nodes Involved:**
  - Shopify, products

- **Node Details:**

  - **Shopify, products**
    - **Type & Role:** GraphQL node; queries Shopify Admin API for products using cursor pagination.
    - **Configuration:**
      - **Endpoint:** `https://store99563.myshopify.com/admin/api/2025-04/graphql.json`
      - **Query:** Retrieves the first `pageSize` products after a cursor, including product details (`id`, `title`, `handle`, `createdAt`, `updatedAt`) and pagination info (`hasNextPage`, `endCursor`).
      - **Variables:** 
        - `pageSize` fixed at 5 (small for illustration).
        - `cursor` dynamically set using an expression:
          ```js
          $if($json?.data?.products?.pageInfo?.hasNextPage !== undefined, 
              "\"" + $json.data.products.pageInfo.endCursor + "\"", 
              "null");
          ```
          This means the cursor uses the previous page’s `endCursor` if more pages exist; otherwise, null for the initial call.
      - **Authentication:** Header-based authentication via stored Shopify credentials.
    - **Expressions/Variables:** Cursor expression dynamically binds to previous response.
    - **Input/Output:** Input from Start or Wait node; outputs product data and pagination info.
    - **Version-Specific Requirements:** Uses GraphQL node version 1.1 for variable support.
    - **Edge Cases / Failures:**
      - API rate limits from Shopify.
      - Invalid credentials or endpoint changes.
      - Expression failures if previous run data is missing or malformed.
      - Empty product sets or no further pages.
    - **Sub-workflows:** None.

#### 1.3 Pagination Control and Delay Block

- **Overview:** Evaluates if more pages exist to fetch, waits 1 second to avoid API rate limits, then loops back to fetch the next page.
- **Nodes Involved:**
  - hasMoreProducts
  - Wait 1s

- **Node Details:**

  - **hasMoreProducts**
    - **Type & Role:** If node; checks boolean condition whether more products remain.
    - **Configuration:**
      - Condition: Checks if `data.products.pageInfo.hasNextPage` is true.
    - **Expressions/Variables:** Condition uses expression `={{ $json.data.products.pageInfo.hasNextPage }}`
    - **Input/Output:** Input from Shopify products node; if true, outputs to Wait node; if false, no further output (ends loop).
    - **Edge Cases / Failures:**
      - Missing or malformed pageInfo data causing false negatives.
      - Strict type validation avoids false positives.
    - **Sub-workflows:** None.

  - **Wait 1s**
    - **Type & Role:** Wait node; pauses execution for 1 second before next iteration.
    - **Configuration:** Wait time set to 1 second.
    - **Input/Output:** Input from If node; output loops back to Shopify products node.
    - **Edge Cases / Failures:** None significant; prevents API rate-limit issues.
    - **Sub-workflows:** None.

- **Connections:**  
  - `Shopify, products` → `hasMoreProducts` → if true → `Wait 1s` → loops back to `Shopify, products`.  
  - Loop continues until `hasMoreProducts` is false.

#### Additional Notes

- **Sticky Note Node:**  
  - Provides explanation of the cursor-based pagination concept for Shopify GraphQL in n8n.  
  - Highlights that the example uses a small page size (5) for demonstration and that the endpoint must be updated per store.

---

### 3. Summary Table

| Node Name         | Node Type         | Functional Role                     | Input Node(s)       | Output Node(s)      | Sticky Note                                                                                           |
|-------------------|-------------------|-----------------------------------|---------------------|---------------------|-----------------------------------------------------------------------------------------------------|
| Start Workflow    | Manual Trigger    | Initiates workflow execution      | -                   | Shopify, products   |                                                                                                     |
| Shopify, products | GraphQL           | Fetches products page via Shopify | Start Workflow, Wait 1s | hasMoreProducts    |                                                                                                     |
| hasMoreProducts   | If                | Checks if more product pages exist| Shopify, products    | Wait 1s             |                                                                                                     |
| Wait 1s           | Wait              | Delays before fetching next page  | hasMoreProducts      | Shopify, products   |                                                                                                     |
| Sticky Note       | Sticky Note       | Documentation of pagination logic | -                   | -                   | ## Shopify GraphQL cursor loop\nMany Shopify GraphQL queries have the ability to return a cursor which you can loop over, however the N8N GraphQL node does not natively have the ability to fetch pages.  This simple 3 node workflow displays how to setup a cursor to fetch all items in a collection.\n\nNote : The pageSize in the \"Shopify, products\" node is set to 5 to illustrate how querying by cursor works. In production you would set this to a much larger value. Also, Update the Endpoint in GraphQL node to reflect your Shopify store. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Manual Trigger node:**
   - Name: `Start Workflow`
   - Purpose: To manually start the workflow.
   - No special parameters needed.

3. **Add a GraphQL node to query Shopify products:**
   - Name: `Shopify, products`
   - Set node type to `GraphQL`.
   - Configure the following:
     - **Endpoint URL:**  
       `https://<your-shopify-store>.myshopify.com/admin/api/2025-04/graphql.json`  
       Replace `<your-shopify-store>` with your actual Shopify store name.
     - **Query:**  
       ```
       query ($pageSize: Int!, $cursor: String) {
         products(first: $pageSize, after: $cursor) {
           edges {
             node {
               id
               title
               handle
               createdAt
               updatedAt
             }
           }
           pageInfo {
             hasNextPage
             endCursor
           }
         }
       }
       ```
     - **Variables:**  
       Use the expression editor to set:
       ```json
       {
         "pageSize": 5,
         "cursor": {{$if($json?.data?.products?.pageInfo?.hasNextPage !== undefined, "\"" + $json.data.products.pageInfo.endCursor + "\"", "null")}}
       }
       ```
       This dynamically sets the cursor based on the previous page’s `endCursor` or null for the first page.
     - **Authentication:**  
       Configure HTTP Header Authentication credentials for Shopify API (API password/token with `X-Shopify-Access-Token` header).
       - Go to Credentials, create `HTTP Header Auth` credentials with header name `X-Shopify-Access-Token` and your store’s Admin API access token.
     - **Node Version:** Use at least version 1.1 for variable interpolation support.

4. **Add an If node to check if more products exist:**
   - Name: `hasMoreProducts`
   - Condition:
     - Type: Boolean
     - Expression: `{{$json.data.products.pageInfo.hasNextPage}}`
     - Operator: `is true`
   - This condition determines if the workflow should continue paginating.

5. **Add a Wait node to pause between queries:**
   - Name: `Wait 1s`
   - Set wait duration to 1 second.
   - This helps avoid Shopify API rate limits.

6. **Connect nodes as follows:**
   - `Start Workflow` → `Shopify, products`
   - `Shopify, products` → `hasMoreProducts`
   - `hasMoreProducts` (true output) → `Wait 1s`
   - `Wait 1s` → loops back into `Shopify, products`
   - `hasMoreProducts` (false output) → no further connections (ends workflow)

7. **Optional: Add a Sticky Note node with content explaining the cursor-based pagination concept and instructions to update endpoint and page size for production.**

8. **Test the workflow:**
   - Trigger manually.
   - Observe product pages being fetched iteratively in batches of 5 until no more pages exist.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                         | Context or Link                                                                                      |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Shopify GraphQL cursor loop: Many Shopify GraphQL queries return cursors that enable pagination, but n8n’s GraphQL node does not natively paginate. This workflow demonstrates manual looping using cursors and a delay node to handle rate limiting.                   | See sticky note content inside the workflow for detailed explanation.                              |
| Adjust the `pageSize` to a larger number in production to reduce API calls and improve efficiency. The example uses 5 for clarity.                                                                                                                                   | Best practice for production workflows.                                                           |
| Update the GraphQL node’s endpoint URL to match your Shopify store domain and API version.                                                                                                                                                                            | Critical to avoid API errors.                                                                       |
| Use Shopify Admin API access token with header `X-Shopify-Access-Token` for authentication. Ensure token permissions include reading products.                                                                                                                     | Shopify API authentication requirements.                                                          |
| Shopify API limits: Be mindful of Shopify's API rate limits; using a Wait node is recommended to avoid hitting those limits when paginating extensively.                                                                                                           | See Shopify API rate limit documentation: https://shopify.dev/api/usage/rate-limits                |

---

**Disclaimer:** The provided text is derived exclusively from an automated workflow created with n8n, a workflow automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly available.