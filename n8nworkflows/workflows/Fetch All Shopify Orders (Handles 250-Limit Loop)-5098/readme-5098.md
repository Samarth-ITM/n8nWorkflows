Fetch All Shopify Orders (Handles 250-Limit Loop)

https://n8nworkflows.xyz/workflows/fetch-all-shopify-orders--handles-250-limit-loop--5098


# Fetch All Shopify Orders (Handles 250-Limit Loop)

### 1. Workflow Overview

This workflow automates the process of fetching **all Shopify orders** created within a defined date range (last 30 days by default) from a Shopify store, handling Shopify’s API limit of 250 orders per request by using pagination (looping through pages). It then extracts detailed line item information, groups these items by vendor and SKU, calculates summary statistics (total quantity sold and total sales per SKU), and appends this summarized data to a Google Sheets spreadsheet for reporting or further analysis.

**Target Use Cases:**  
- Automated periodic extraction of Shopify order data for sales analysis.  
- Handling API pagination limits to collect large datasets beyond a single request limit.  
- Summarizing product sales data by SKU and vendor for inventory or sales tracking.  
- Storing processed data in Google Sheets for easy sharing and visualization.

**Logical Blocks:**

- **1.1 Schedule Trigger:** Initiates the workflow at a specified daily time.  
- **1.2 Date Range Setup:** Defines the dynamic date range for order retrieval (last 30 days).  
- **1.3 Shopify GraphQL Query Loop:** Retrieves orders in batches of 250, looping if more pages exist.  
- **1.4 Pagination Check:** Determines if another request cycle is needed based on Shopify’s `hasNextPage`.  
- **1.5 Data Processing & Grouping:** Collates all retrieved orders, extracts line items, and groups by SKU/vendor with sales summaries.  
- **1.6 Output to Google Sheets:** Appends the summarized sales data into a designated Google Sheet.

---

### 2. Block-by-Block Analysis

#### 1.1 Schedule Trigger

- **Overview:**  
  Starts the workflow automatically at a specific time each day.

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Configuration: Fires daily at 08:43 (hour: 8, minute: 43)  
    - Inputs: None (initiates workflow)  
    - Outputs: Triggers the `searchPeriod` node  
    - Edge Cases:  
      - Workflow will not run if n8n instance is down at trigger time  
      - Timezone considerations may affect trigger time  
    - Version: 1.2

#### 1.2 Date Range Setup

- **Overview:**  
  Sets the dynamic date range parameters for querying Shopify orders: 30 days ago up to today.

- **Nodes Involved:**  
  - searchPeriod (Set node)

- **Node Details:**

  - **searchPeriod**  
    - Type: Set  
    - Configuration:  
      - `startDay`: Current date minus 30 days, formatted as `yyyy-MM-dd`  
      - `endDay`: Current date, formatted as `yyyy-MM-dd`  
    - Inputs: Trigger from Schedule Trigger  
    - Outputs: Feeds parameters into Shopify GraphQL query node (`getOrderDoLoop`)  
    - Expressions: Uses `$now` timestamp with `.plus({ days: -30 })` and `.toFormat('yyyy-MM-dd')`  
    - Edge Cases:  
      - Date formatting errors if expression engine changes  
      - If workflow runs shortly after midnight, date range could be off by one day  
    - Version: 3.4

#### 1.3 Shopify GraphQL Query Loop

- **Overview:**  
  Queries Shopify’s GraphQL API to fetch orders in batches of 250, using pagination cursor `endCursor` to loop through all available pages within the date range.

- **Nodes Involved:**  
  - getOrderDoLoop (GraphQL node)  
  - is hasNextPage = true (IF node)

- **Node Details:**

  - **getOrderDoLoop**  
    - Type: GraphQL  
    - Configuration:  
      - Shopify Admin API endpoint: `https://lokal-eg.myshopify.com/admin/api/2025-01/graphql.json`  
      - Authentication: Header Token (Shopify private app token)  
      - Query: Fetches first 250 orders, optionally after cursor `endCursor` if exists  
      - Filters orders by creation date between `startDay` and `endDay` from `searchPeriod` node  
      - Retrieves order ID, creation date, and detailed line items (product title, vendor, SKU, variant, quantity, price, currency)  
    - Inputs: Receives date range parameters from `searchPeriod`  
    - Outputs: Passes data to `is hasNextPage = true`  
    - Retry Policy: Enabled (retry on fail with 5s wait)  
    - Edge Cases:  
      - API rate limiting or token expiration may cause failures  
      - Incorrect date format or invalid cursors may cause query errors  
      - Large data sets may cause timeouts  
    - Version: 1.1

  - **is hasNextPage = true**  
    - Type: IF  
    - Configuration:  
      - Condition: Checks if `data.orders.pageInfo.hasNextPage` is `true`  
      - If true: loops back to `getOrderDoLoop` to fetch next page  
      - If false: proceeds to data processing (`groupItemListbyVendor`)  
    - Inputs: Output from `getOrderDoLoop`  
    - Outputs:  
      - True branch: back to `getOrderDoLoop` (pagination loop)  
      - False branch: to data processing  
    - Edge Cases:  
      - Incorrect JSON path or data structure changes could cause condition failure  
      - Infinite loop risk if `hasNextPage` does not become false (unlikely if API behaves properly)  
    - Version: 2.2

#### 1.4 Data Processing & Grouping

- **Overview:**  
  Aggregates all pages of orders fetched, extracts line item details, and summarizes sales data grouped by SKU and vendor.

- **Nodes Involved:**  
  - groupItemListbyVendor (Code node)

- **Node Details:**

  - **groupItemListbyVendor**  
    - Type: Code (JavaScript)  
    - Configuration:  
      - Collects all executions of `getOrderDoLoop` node (all pages)  
      - Iterates through orders and their line items  
      - Extracts order ID, order date, product title, vendor, SKU, variant, quantity, price, currency  
      - Groups data by SKU, summing quantities and total sales  
      - Returns an array of summarized objects (one per SKU) for downstream processing  
    - Inputs: Multiple executions output from `getOrderDoLoop` (via loop or batch)  
    - Outputs: Summarized sales data to Google Sheets node  
    - Edge Cases:  
      - Missing or null product/vendor/SKU fields handled with fallback (empty string)  
      - Empty order sets or no line items handled gracefully (empty arrays)  
      - Parsing errors if Shopify data structure changes  
      - Large datasets may cause performance delays or memory issues in code execution  
    - Version: 2

#### 1.5 Output to Google Sheets

- **Overview:**  
  Appends the grouped and summarized sales data into a specific Google Sheets spreadsheet for reporting.

- **Nodes Involved:**  
  - Google Sheets

- **Node Details:**

  - **Google Sheets**  
    - Type: Google Sheets  
    - Configuration:  
      - Operation: Append rows  
      - Target Spreadsheet ID: `1ZEm8zyl__E59M6kD9UkvHHZTMbEWktjoH3FrYhBg6LU`  
      - Sheet: Sheet with GID=0 (first tab)  
      - Columns mapped: Brand, SKU, Title, total_quantity_sold, total_sales  
      - Auto-mapping enabled to match input JSON keys to columns  
    - Inputs: Summarized data from `groupItemListbyVendor`  
    - Outputs: None (end of workflow)  
    - Credentials: Google Sheets OAuth2 API  
    - Edge Cases:  
      - Authentication token expiration or permission issues cause failures  
      - Spreadsheet ID or sheet name errors cause append failures  
      - Data type mismatches may cause errors (e.g., numeric vs string)  
    - Version: 4.5

---

### 3. Summary Table

| Node Name            | Node Type          | Functional Role                       | Input Node(s)         | Output Node(s)          | Sticky Note                                     |
|----------------------|--------------------|------------------------------------|-----------------------|-------------------------|------------------------------------------------|
| Schedule Trigger      | Schedule Trigger   | Initiates workflow daily at 08:43  | None                  | searchPeriod            |                                                |
| searchPeriod         | Set                | Defines date range for orders       | Schedule Trigger      | getOrderDoLoop          |                                                |
| getOrderDoLoop       | GraphQL            | Fetches Shopify orders in pages     | searchPeriod          | is hasNextPage = true   |                                                |
| is hasNextPage = true| IF                 | Controls pagination loop            | getOrderDoLoop        | getOrderDoLoop, groupItemListbyVendor |                                           |
| groupItemListbyVendor| Code               | Aggregates and summarizes line items| is hasNextPage = true | Google Sheets           |                                                |
| Google Sheets        | Google Sheets      | Appends summarized data to spreadsheet| groupItemListbyVendor | None                    |                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Set to trigger daily at 08:43 (local time or as per your timezone)  
   - No inputs required

2. **Create Set node named `searchPeriod`**  
   - Connect Schedule Trigger output to this node  
   - Add two string fields:  
     - `startDay` with expression: `={{$now.plus({ days: -30 }).toFormat('yyyy-MM-dd')}}`  
     - `endDay` with expression: `={{$now.toFormat('yyyy-MM-dd')}}`

3. **Create GraphQL node named `getOrderDoLoop`**  
   - Connect output of `searchPeriod` to this node  
   - Configure API Endpoint: `https://lokal-eg.myshopify.com/admin/api/2025-01/graphql.json` (replace domain as appropriate)  
   - Authentication: HTTP Header Auth with Shopify private app token credential  
   - Query: Use the following GraphQL query with dynamic pagination and date filters:

     ```graphql
     {
       orders(
         first: 250
         {{ $json.data.orders.pageInfo.endCursor ? ', after: "' + $json.data.orders.pageInfo.endCursor + '"' : '' }}
         query: "created_at:>={{ $('searchPeriod').item.json.startDay }} AND created_at:<={{ $('searchPeriod').item.json.endDay }}"
       ) {
         edges {
           node {
             id
             createdAt
             lineItems(first: 250) {
               edges {
                 node {
                   product {
                     title
                     vendor
                   }
                   quantity
                   originalUnitPriceSet {
                     shopMoney {
                       amount
                       currencyCode
                     }
                   }
                   variant {
                     title
                   }
                   sku
                 }
               }
             }
           }
         }
         pageInfo {
           hasNextPage
           endCursor
         }
       }
     }
     ```
   - Enable retry on failure, with 5000ms wait between retries

4. **Create IF node named `is hasNextPage = true`**  
   - Connect output of `getOrderDoLoop` to this node  
   - Condition: Expression checking if `{{$json.data.orders.pageInfo.hasNextPage}}` equals `true` (boolean condition)  
   - True branch: Connect back to `getOrderDoLoop` (to loop and fetch next page)  
   - False branch: Connect to next processing node

5. **Create Code node named `groupItemListbyVendor`**  
   - Connect false output of IF node to this node  
   - Paste the following JavaScript code:

     ```javascript
     // Collect all executions of 'getOrderDoLoop'
     let allOrders = [];
     let counter = 0;
     while (true) {
       try {
         const items = $items("getOrderDoLoop", 0, counter);
         allOrders.push(...items);
       } catch (e) {
         break;
       }
       counter++;
     }

     // Extract line items from orders
     const allLineItems = [];
     for (const item of allOrders) {
       const edges = item.json.data.orders.edges || [];
       for (const order of edges) {
         const orderId = order.node.id.replace("gid://shopify/Order/", "");
         const orderDate = order.node.createdAt.split("T")[0];
         const lineItems = order.node.lineItems.edges;
         for (const li of lineItems) {
           allLineItems.push({
             order_id: orderId,
             order_date: orderDate,
             product_title: li.node.product?.title || "",
             vendor: li.node.product?.vendor || "",
             sku: li.node.sku || "",
             variant_name: li.node.variant?.title || "",
             quantity: li.node.quantity || 0,
             price: li.node.originalUnitPriceSet?.shopMoney?.amount || 0,
             currency: li.node.originalUnitPriceSet?.shopMoney?.currencyCode || ""
           });
         }
       }
     }

     // Group by SKU and summarize
     const summary = {};
     for (const row of allLineItems) {
       if (!row.sku) continue;
       if (!summary[row.sku]) {
         summary[row.sku] = {
           SKU: row.sku,
           Title: row.product_title,
           Brand: row.vendor,
           total_quantity_sold: 0,
           total_sales: 0
         };
       }
       summary[row.sku].total_quantity_sold += row.quantity;
       summary[row.sku].total_sales += row.quantity * parseFloat(row.price);
     }

     return Object.values(summary).map(row => ({ json: row }));
     ```

6. **Create Google Sheets node**  
   - Connect output of `groupItemListbyVendor` to this node  
   - Set operation to: Append rows  
   - Set Spreadsheet ID to your target Google Sheet (e.g., `1ZEm8zyl__E59M6kD9UkvHHZTMbEWktjoH3FrYhBg6LU`)  
   - Set Sheet Name or GID to `gid=0` (or appropriate sheet)  
   - Configure columns mapping: Brand, SKU, Title, total_quantity_sold, total_sales  
   - Use Google Sheets OAuth2 credentials with write permissions

7. **Check all nodes for correct connections and retry policies**  
   - Especially ensure `getOrderDoLoop` retries on failure with delay  
   - Validate all expressions and credentials before execution

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                   |
|-----------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Shopify API limits results to 250 orders per request; this workflow handles pagination automatically.     | Shopify Admin API GraphQL documentation                                                          |
| Date filter uses Shopify’s `created_at` field with a query string in GraphQL for efficient filtering.     | Shopify GraphQL API reference                                                                    |
| Google Sheets integration requires OAuth2 credentials with edit permissions on the target spreadsheet.   | Google Sheets API documentation                                                                  |
| The workflow uses a loop based on `hasNextPage` boolean to fetch all orders beyond the 250 limit.         | Pagination best practices in Shopify API                                                         |
| The JavaScript code node aggregates data across multiple executions of the paginated node.                | n8n documentation on `$items()` function                                                         |

---

**Disclaimer:** The text provided here is exclusively based on an automated workflow created with n8n, a no-code/low-code automation tool. It complies strictly with applicable content policies and contains no illegal, offensive, or protected content. All data processed is legal and publicly accessible.