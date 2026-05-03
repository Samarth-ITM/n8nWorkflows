Tag high-risk WooCommerce SKUs and send daily alerts to Slack

https://n8nworkflows.xyz/workflows/tag-high-risk-woocommerce-skus-and-send-daily-alerts-to-slack-14531


# Tag high-risk WooCommerce SKUs and send daily alerts to Slack

# 1. Workflow Overview

This workflow automatically reviews WooCommerce product sales every day, identifies products with elevated inventory risk based on sales in the last 14 days, updates those products with risk tags in WooCommerce, and sends a daily Slack alert summarizing the risky SKUs.

Primary use cases:
- Flag fast-selling products before stock becomes a problem
- Add visual risk tags directly in WooCommerce
- Notify operations, retail, or inventory teams in Slack once per day
- Reduce manual review of sales velocity across products

The workflow is organized into the following logical blocks:

## 1.1 Scheduled Trigger
Starts the workflow once per day at a configured hour.

## 1.2 WooCommerce Data Collection
Fetches completed WooCommerce orders from the last 14 days, then retrieves the product catalog used for risk analysis.

## 1.3 Product-by-Product Sales Analysis
Processes products one at a time, counts units sold across recent orders, and assigns a risk level using hardcoded thresholds.

## 1.4 Risk-Based Routing and Product Tagging
Routes only non-OK products into separate Watchlist, High-Risk, and Critical branches and updates WooCommerce product tags accordingly.

## 1.5 Alert Preparation and Slack Notification
Combines all tagged products, formats a Slack-friendly alert message, and posts it to a Slack channel.

## 1.6 Batch Loop Control
Uses the `Split In Batches` loop pattern so the workflow continues processing products until all products are analyzed.

---

# 2. Block-by-Block Analysis

## 2.1 Scheduled Trigger

**Overview:**  
This block launches the workflow automatically every day. It is the single entry point for the workflow and starts the downstream WooCommerce data pull.

**Nodes Involved:**  
- Daily inventory risk check

### Node Details

#### Daily inventory risk check
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`  
  Time-based trigger node that starts the workflow on a schedule.
- **Configuration choices:**  
  Configured with an interval rule that triggers daily at hour `6`. The exact timezone depends on the n8n instance settings.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input: none
  - Output: `Get last 14 days orders `
- **Version-specific requirements:**  
  Uses `typeVersion 1.2`.
- **Edge cases or potential failure types:**  
  - Timezone mismatch between business expectation and server/n8n timezone
  - Workflow inactive, so trigger will not run until activated
- **Sub-workflow reference:**  
  None.

---

## 2.2 WooCommerce Data Collection

**Overview:**  
This block retrieves the data needed for analysis: first recent completed orders, then product details. The workflow uses orders to calculate sales volume and product records as the item list to evaluate.

**Nodes Involved:**  
- Get last 14 days orders 
- Get product details

### Node Details

#### Get last 14 days orders 
- **Type and technical role:** `n8n-nodes-base.wooCommerce`  
  WooCommerce API node used to retrieve all completed orders in a rolling 14-day window.
- **Configuration choices:**  
  - Resource: `order`
  - Operation: `getAll`
  - `returnAll: true`
  - Filters:
    - `after`: current time minus 14 days
    - `before`: current time
    - `status`: `completed`
- **Key expressions or variables used:**  
  - `={{ new Date(Date.now() - 14 * 24 * 60 * 60 * 1000).toISOString()}}`
  - `={{ new Date().toISOString() }}`
- **Input and output connections:**  
  - Input: `Daily inventory risk check`
  - Output: `Get product details`
- **Version-specific requirements:**  
  Uses `typeVersion 1`.
- **Edge cases or potential failure types:**  
  - WooCommerce credential failure
  - API pagination or performance issues if order volume is very large
  - Date filtering behavior depends on WooCommerce API interpretation of ISO timestamps
  - If no completed orders exist, downstream risk calculations will classify all products as `OK`
- **Sub-workflow reference:**  
  None.

#### Get product details
- **Type and technical role:** `n8n-nodes-base.wooCommerce`  
  WooCommerce API node used to retrieve all products that will be evaluated for risk.
- **Configuration choices:**  
  - Operation: `getAll`
  - No explicit filtering options
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input: `Get last 14 days orders `
  - Output: `Process Products one by one`
- **Version-specific requirements:**  
  Uses `typeVersion 1`.
- **Edge cases or potential failure types:**  
  - Credential or permission errors
  - Large catalog size may increase runtime significantly
  - Variable products and variations may require special care depending on how products are returned
- **Sub-workflow reference:**  
  None.

---

## 2.3 Product-by-Product Sales Analysis

**Overview:**  
This block loops through each product, calculates the total units sold in the last 14 days by inspecting order line items, and assigns a risk label according to predefined thresholds.

**Nodes Involved:**  
- Process Products one by one
- Calculate Sales & Risk level
- Split product by risk level

### Node Details

#### Process Products one by one
- **Type and technical role:** `n8n-nodes-base.splitInBatches`  
  Iterates through the WooCommerce products one item at a time.
- **Configuration choices:**  
  Default options; no explicit batch size shown, so standard iteration behavior applies.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input: `Get product details`
  - Output 1: loop continuation, later fed by `Notify team for Inventory alert`
  - Output 2: `Calculate Sales & Risk level`
- **Version-specific requirements:**  
  Uses `typeVersion 3`.
- **Edge cases or potential failure types:**  
  - Large product catalogs can lead to long execution times
  - Loop design means Slack notification occurs before the loop advances, which can create repeated notifications if not carefully intended
- **Sub-workflow reference:**  
  None.

#### Calculate Sales & Risk level
- **Type and technical role:** `n8n-nodes-base.code`  
  JavaScript node that computes per-product units sold and derives a risk level.
- **Configuration choices:**  
  Custom JS logic:
  - Reads current product with `$input.first().json`
  - Reads all order items using `$items("Get last 14 days orders ").map(i => i.json)`
  - Loops through each order and each `line_item`
  - Counts sold quantity when:
    - `li.product_id === product.id`, or
    - `li.variation_id === product.id`
  - Assigns risk thresholds:
    - `OK` for less than 2
    - `Watchlist` for 2 or more
    - `High-Risk` for 6 or more
    - `Critical` for 10 or more
  - Returns:
    - `product_id`
    - `product_name`
    - `sku`
    - `stock_status`
    - `units_sold_last_14_days`
    - `risk_level`
- **Key expressions or variables used:**  
  - `$input.first().json`
  - `$items("Get last 14 days orders ")`
- **Input and output connections:**  
  - Input: `Process Products one by one`
  - Output: `Split product by risk level`
- **Version-specific requirements:**  
  Uses `typeVersion 2`.
- **Edge cases or potential failure types:**  
  - Node name dependency: the code references `Get last 14 days orders ` with a trailing space; renaming the node without updating code will break it
  - If order data shape differs, missing `line_items` could lead to zero counts, though the code safely defaults with `|| []`
  - Product variations may not match perfectly depending on catalog structure
  - Thresholds are hardcoded and not configurable from input
- **Sub-workflow reference:**  
  None.

#### Split product by risk level
- **Type and technical role:** `n8n-nodes-base.switch`  
  Routes each product into a branch based on the computed `risk_level`.
- **Configuration choices:**  
  Four rules:
  - `Watchlist`
  - `High-Risk`
  - `Critical`
  - `OK`
  The first three branches continue to tagging; `OK` goes nowhere.
- **Key expressions or variables used:**  
  - `={{ $json.risk_level }}`
- **Input and output connections:**  
  - Input: `Calculate Sales & Risk level`
  - Outputs:
    - Output 0: `Prepare watchlist products`
    - Output 1: `Prepare High-risk products`
    - Output 2: `Prepare Critical products`
    - Output 3: no downstream connection for `OK`
- **Version-specific requirements:**  
  Uses `typeVersion 3.3`, conditions format `version 2`.
- **Edge cases or potential failure types:**  
  - Exact string matching; any typo in `risk_level` causes routing failure
  - `OK` products are intentionally discarded from alerting and tagging
- **Sub-workflow reference:**  
  None.

---

## 2.4 Risk-Based Routing and Product Tagging

**Overview:**  
This block prepares minimal update payloads and writes risk tags back to WooCommerce products. Each risk level has its own branch and fixed WooCommerce tag ID.

**Nodes Involved:**  
- Prepare watchlist products
- Update Product tag as Watchlist
- Prepare High-risk products
- Update Product tag as High-risk
- Prepare Critical products
- Update Product tag as Critical

### Node Details

#### Prepare watchlist products
- **Type and technical role:** `n8n-nodes-base.set`  
  Creates a compact payload for Watchlist products.
- **Configuration choices:**  
  Assigns:
  - `product_id` from current item
  - `tag` from `risk_level`
- **Key expressions or variables used:**  
  - `={{ $json.product_id }}`
  - `={{ $json.risk_level }}`
- **Input and output connections:**  
  - Input: `Split product by risk level`
  - Output: `Update Product tag as Watchlist`
- **Version-specific requirements:**  
  Uses `typeVersion 3.4`.
- **Edge cases or potential failure types:**  
  - If `product_id` is missing, WooCommerce update will fail
- **Sub-workflow reference:**  
  None.

#### Update Product tag as Watchlist
- **Type and technical role:** `n8n-nodes-base.wooCommerce`  
  Updates a WooCommerce product to assign a specific tag ID.
- **Configuration choices:**  
  - Resource: `product`
  - Operation: `update`
  - `productId`: current `product_id`
  - `updateFields.tags`: `[74]`
- **Key expressions or variables used:**  
  - `={{ $json.product_id }}`
- **Input and output connections:**  
  - Input: `Prepare watchlist products`
  - Output: `Combine tagged products` input 0
- **Version-specific requirements:**  
  Uses `typeVersion 1`.
- **Edge cases or potential failure types:**  
  - Tag ID `74` must exist in WooCommerce
  - This update may overwrite the product’s tag set depending on WooCommerce node behavior; verify whether tags are replaced or merged
  - Permission/authentication failures
- **Sub-workflow reference:**  
  None.

#### Prepare High-risk products
- **Type and technical role:** `n8n-nodes-base.set`  
  Creates a compact payload for High-Risk products.
- **Configuration choices:**  
  Assigns:
  - `product_id`
  - `tag`
- **Key expressions or variables used:**  
  - `={{ $json.product_id }}`
  - `={{ $json.risk_level }}`
- **Input and output connections:**  
  - Input: `Split product by risk level`
  - Output: `Update Product tag as High-risk`
- **Version-specific requirements:**  
  Uses `typeVersion 3.4`.
- **Edge cases or potential failure types:**  
  Same as the Watchlist preparation branch.
- **Sub-workflow reference:**  
  None.

#### Update Product tag as High-risk
- **Type and technical role:** `n8n-nodes-base.wooCommerce`  
  Updates the product with the High-Risk WooCommerce tag.
- **Configuration choices:**  
  - Resource: `product`
  - Operation: `update`
  - `productId`: current `product_id`
  - `updateFields.tags`: `[75]`
- **Key expressions or variables used:**  
  - `={{ $json.product_id }}`
- **Input and output connections:**  
  - Input: `Prepare High-risk products`
  - Output: `Combine tagged products` input 1
- **Version-specific requirements:**  
  Uses `typeVersion 1`.
- **Edge cases or potential failure types:**  
  - Tag ID `75` must exist
  - Possible replacement of existing tags
- **Sub-workflow reference:**  
  None.

#### Prepare Critical products
- **Type and technical role:** `n8n-nodes-base.set`  
  Creates a compact payload for Critical products.
- **Configuration choices:**  
  Assigns:
  - `product_id`
  - `tag`
- **Key expressions or variables used:**  
  - `={{ $json.product_id }}`
  - `={{ $json.risk_level }}`
- **Input and output connections:**  
  - Input: `Split product by risk level`
  - Output: `Update Product tag as Critical`
- **Version-specific requirements:**  
  Uses `typeVersion 3.4`.
- **Edge cases or potential failure types:**  
  Same as the other Set nodes in this section.
- **Sub-workflow reference:**  
  None.

#### Update Product tag as Critical
- **Type and technical role:** `n8n-nodes-base.wooCommerce`  
  Updates the product with the Critical WooCommerce tag.
- **Configuration choices:**  
  - Resource: `product`
  - Operation: `update`
  - `productId`: current `product_id`
  - `updateFields.tags`: `[76]`
- **Key expressions or variables used:**  
  - `={{ $json.product_id }}`
- **Input and output connections:**  
  - Input: `Prepare Critical products`
  - Output: `Combine tagged products` input 2
- **Version-specific requirements:**  
  Uses `typeVersion 1`.
- **Edge cases or potential failure types:**  
  - Tag ID `76` must exist
  - Possible replacement of existing tags
- **Sub-workflow reference:**  
  None.

---

## 2.5 Alert Preparation and Slack Notification

**Overview:**  
This block combines all products that were tagged as risky, reshapes the data, builds a human-readable alert, and sends it to Slack.

**Nodes Involved:**  
- Combine tagged products
- Prepare data for Alerts
- Build slack alert message
- Notify team for Inventory alert

### Node Details

#### Combine tagged products
- **Type and technical role:** `n8n-nodes-base.merge`  
  Combines results from the three tag-update branches into one stream.
- **Configuration choices:**  
  Configured with `numberInputs: 3`.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Inputs:
    - `Update Product tag as Watchlist`
    - `Update Product tag as High-risk`
    - `Update Product tag as Critical`
  - Output: `Prepare data for Alerts`
- **Version-specific requirements:**  
  Uses `typeVersion 3.2`.
- **Edge cases or potential failure types:**  
  - If no products fall into risky categories, this branch may produce no items and no Slack message
  - Merge behavior should be validated against the exact n8n version, especially if branch timings differ
- **Sub-workflow reference:**  
  None.

#### Prepare data for Alerts
- **Type and technical role:** `n8n-nodes-base.set`  
  Keeps only the alert fields required for the final Slack message.
- **Configuration choices:**  
  Assigns:
  - `id` from updated WooCommerce product
  - `sku`
  - `units_sold_last_14_days` from the `Split product by risk level` item context
  - `Risk` from `tags[0].name`
  - `name`
- **Key expressions or variables used:**  
  - `={{ $json.id }}`
  - `={{ $json.sku }}`
  - `={{ $('Split product by risk level').item.json.units_sold_last_14_days }}`
  - `={{ $json.tags[0].name }}`
  - `={{ $json.name }}`
- **Input and output connections:**  
  - Input: `Combine tagged products`
  - Output: `Build slack alert message`
- **Version-specific requirements:**  
  Uses `typeVersion 3.4`.
- **Edge cases or potential failure types:**  
  - The expression `$('Split product by risk level').item...` depends on item linking remaining intact after branching and merge; this can fail in some restructuring scenarios
  - `tags[0]` assumes the first tag is the risk tag; if WooCommerce returns tags in another order, the reported risk may be wrong
  - If WooCommerce update response omits tags or returns them differently, expression errors may occur
- **Sub-workflow reference:**  
  None.

#### Build slack alert message
- **Type and technical role:** `n8n-nodes-base.code`  
  Aggregates all risky products into a single Slack message string.
- **Configuration choices:**  
  Custom JS:
  - Reads all incoming items with `$input.all()`
  - Returns no output if the list is empty
  - Builds a multi-line text summary
  - For each item includes:
    - product name
    - ID
    - SKU
    - units sold
    - risk
  - Returns one item containing `slack_message`
- **Key expressions or variables used:**  
  - `$input.all()`
- **Input and output connections:**  
  - Input: `Prepare data for Alerts`
  - Output: `Notify team for Inventory alert`
- **Version-specific requirements:**  
  Uses `typeVersion 2`.
- **Edge cases or potential failure types:**  
  - If the upstream branch has no items, the node returns `[]`, which correctly suppresses Slack posting
  - Message length could become large if many products are flagged
- **Sub-workflow reference:**  
  None.

#### Notify team for Inventory alert
- **Type and technical role:** `n8n-nodes-base.slack`  
  Posts the generated inventory alert message to a Slack channel.
- **Configuration choices:**  
  - Sends `text` from `={{ $json.slack_message }}`
  - Target is a fixed channel selected by ID
  - `includeLinkToWorkflow: false`
- **Key expressions or variables used:**  
  - `={{ $json.slack_message }}`
- **Input and output connections:**  
  - Input: `Build slack alert message`
  - Output: `Process Products one by one` loop continuation input
- **Version-specific requirements:**  
  Uses `typeVersion 2.3`.
- **Edge cases or potential failure types:**  
  - Slack credential or channel permission errors
  - If channel ID becomes invalid, message delivery fails
  - Current placement in the loop means notification may be triggered multiple times during execution depending on merge/loop behavior
- **Sub-workflow reference:**  
  None.

---

## 2.6 Batch Loop Control

**Overview:**  
This is the loop-back mechanism that tells `Split In Batches` to continue with the next product after the current iteration reaches the end of the downstream branch.

**Nodes Involved:**  
- Process Products one by one
- Notify team for Inventory alert

### Node Details

#### Loop behavior
- `Process Products one by one` sends one product at a time to analysis.
- After the downstream path completes, `Notify team for Inventory alert` connects back to `Process Products one by one` input 0, which advances the batch loop.

**Important implementation note:**  
This design is unusual because the Slack node is part of the loop-closing path. In practice, this can mean:
- the workflow may send a Slack message multiple times while iterating
- the alert may represent only a partial subset depending on execution order
- if no risky product exists for a given iteration, the loop continuation path may not behave as expected

A safer structure would usually loop all products first, collect the results, and only then build/send one final Slack notification outside the batch loop.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Daily inventory risk check | Schedule Trigger | Starts the workflow daily at 06:00 |  | Get last 14 days orders  | Starts the workflow automatically on Schedule to review product sales and stock risk. |
| Get last 14 days orders  | WooCommerce | Fetch recent completed orders from last 14 days | Daily inventory risk check | Get product details | ## Get Orders & Products Process<br>**Get last 14 days orders:** Fetches all completed WooCommerce orders from the last 14 days to calculate product sales.<br>**Get product details:** Fetches product information like name, SKU and stock status from WooCommerce.<br>**Process Products one by one:** Processes each product individually to calculate sales and risk level accurately. |
| Get product details | WooCommerce | Fetch all product records for analysis | Get last 14 days orders  | Process Products one by one | ## Get Orders & Products Process<br>**Get last 14 days orders:** Fetches all completed WooCommerce orders from the last 14 days to calculate product sales.<br>**Get product details:** Fetches product information like name, SKU and stock status from WooCommerce.<br>**Process Products one by one:** Processes each product individually to calculate sales and risk level accurately. |
| Process Products one by one | Split In Batches | Iterate through products one at a time | Get product details; Notify team for Inventory alert | Calculate Sales & Risk level | ## Get Orders & Products Process<br>**Get last 14 days orders:** Fetches all completed WooCommerce orders from the last 14 days to calculate product sales.<br>**Get product details:** Fetches product information like name, SKU and stock status from WooCommerce.<br>**Process Products one by one:** Processes each product individually to calculate sales and risk level accurately. |
| Calculate Sales & Risk level | Code | Count units sold and assign risk level | Process Products one by one | Split product by risk level | ## Analyze Sales & Classify Risk<br>Analyzes product sales from the last 14 days, determines risk level and routes products to the correct action path. |
| Split product by risk level | Switch | Route products by risk label | Calculate Sales & Risk level | Prepare watchlist products; Prepare High-risk products; Prepare Critical products | ## Analyze Sales & Classify Risk<br>Analyzes product sales from the last 14 days, determines risk level and routes products to the correct action path. |
| Prepare watchlist products | Set | Shape Watchlist update payload | Split product by risk level | Update Product tag as Watchlist | ## Apply Inventory Risk Tags<br>This step checks how risky each product is and adds the correct label (Watchlist, High-Risk or Critical) to the product in the store so it’s easy to see which items need attention. |
| Update Product tag as Watchlist | WooCommerce | Update product with Watchlist tag | Prepare watchlist products | Combine tagged products | ## Apply Inventory Risk Tags<br>This step checks how risky each product is and adds the correct label (Watchlist, High-Risk or Critical) to the product in the store so it’s easy to see which items need attention. |
| Prepare High-risk products | Set | Shape High-Risk update payload | Split product by risk level | Update Product tag as High-risk | ## Apply Inventory Risk Tags<br>This step checks how risky each product is and adds the correct label (Watchlist, High-Risk or Critical) to the product in the store so it’s easy to see which items need attention. |
| Update Product tag as High-risk | WooCommerce | Update product with High-Risk tag | Prepare High-risk products | Combine tagged products | ## Apply Inventory Risk Tags<br>This step checks how risky each product is and adds the correct label (Watchlist, High-Risk or Critical) to the product in the store so it’s easy to see which items need attention. |
| Prepare Critical products | Set | Shape Critical update payload | Split product by risk level | Update Product tag as Critical | ## Apply Inventory Risk Tags<br>This step checks how risky each product is and adds the correct label (Watchlist, High-Risk or Critical) to the product in the store so it’s easy to see which items need attention. |
| Update Product tag as Critical | WooCommerce | Update product with Critical tag | Prepare Critical products | Combine tagged products | ## Apply Inventory Risk Tags<br>This step checks how risky each product is and adds the correct label (Watchlist, High-Risk or Critical) to the product in the store so it’s easy to see which items need attention. |
| Combine tagged products | Merge | Combine risky tagged products for reporting | Update Product tag as Watchlist; Update Product tag as High-risk; Update Product tag as Critical | Prepare data for Alerts | ## Prepare Inventory Alert<br><br>**Combine tagged products:** Combines all tagged products (Watchlist, High-Risk, Critical) into one list for reporting.<br>**Prepare data for Alerts:** Keeps only required fields like product name, units sold and risk level for Slack alerts.<br>**Build slack alert message:** Creates a clean, readable inventory risk alert message using product sales data. |
| Prepare data for Alerts | Set | Keep alert-ready fields only | Combine tagged products | Build slack alert message | ## Prepare Inventory Alert<br><br>**Combine tagged products:** Combines all tagged products (Watchlist, High-Risk, Critical) into one list for reporting.<br>**Prepare data for Alerts:** Keeps only required fields like product name, units sold and risk level for Slack alerts.<br>**Build slack alert message:** Creates a clean, readable inventory risk alert message using product sales data. |
| Build slack alert message | Code | Generate one formatted Slack message | Prepare data for Alerts | Notify team for Inventory alert | ## Prepare Inventory Alert<br><br>**Combine tagged products:** Combines all tagged products (Watchlist, High-Risk, Critical) into one list for reporting.<br>**Prepare data for Alerts:** Keeps only required fields like product name, units sold and risk level for Slack alerts.<br>**Build slack alert message:** Creates a clean, readable inventory risk alert message using product sales data. |
| Notify team for Inventory alert | Slack | Send final alert to Slack | Build slack alert message | Process Products one by one | Sends the inventory risk alert message to the Slack channel for the team to review. |
| Sticky Note | Sticky Note | Documentation note on scheduled start |  |  | Starts the workflow automatically on Schedule to review product sales and stock risk. |
| Sticky Note1 | Sticky Note | Documentation note on order/product retrieval and iteration |  |  | ## Get Orders & Products Process<br>**Get last 14 days orders:** Fetches all completed WooCommerce orders from the last 14 days to calculate product sales.<br>**Get product details:** Fetches product information like name, SKU and stock status from WooCommerce.<br>**Process Products one by one:** Processes each product individually to calculate sales and risk level accurately. |
| Sticky Note2 | Sticky Note | Documentation note on analysis and routing |  |  | ## Analyze Sales & Classify Risk<br>Analyzes product sales from the last 14 days, determines risk level and routes products to the correct action path. |
| Sticky Note3 | Sticky Note | Documentation note on WooCommerce tagging |  |  | ## Apply Inventory Risk Tags<br>This step checks how risky each product is and adds the correct label (Watchlist, High-Risk or Critical) to the product in the store so it’s easy to see which items need attention. |
| Sticky Note4 | Sticky Note | Documentation note on Slack notification |  |  | Sends the inventory risk alert message to the Slack channel for the team to review. |
| Sticky Note5 | Sticky Note | Documentation note on alert preparation |  |  | ## Prepare Inventory Alert<br><br>**Combine tagged products:** Combines all tagged products (Watchlist, High-Risk, Critical) into one list for reporting.<br>**Prepare data for Alerts:** Keeps only required fields like product name, units sold and risk level for Slack alerts.<br>**Build slack alert message:** Creates a clean, readable inventory risk alert message using product sales data. |
| Sticky Note6 | Sticky Note | General workflow explanation and setup guidance |  |  | ## How it Works<br><br>This workflow automatically checks your WooCommerce store every day and reviews product sales from the last 14 days. It counts how many units of each product were sold and assigns a risk level **OK** for normal sales, **Watchlist** when sales start increasing, **High-Risk** when stock may run out soon and **Critical** when immediate attention is needed. Based on this risk level, the workflow updates each product in WooCommerce with the correct tag, combines all risky products into a single list and sends a clear Slack alert showing the product name, units sold and risk level, helping you quickly identify fast-selling or low-stock items without manually checking reports.<br><br>## Setup Steps<br><br>**1.** Connect your **WooCommerce account** (API credentials)<br><br>**2.** Make sure product **Tags exist** in WooCommerce<br>(Watchlist, High-Risk, Critical)<br><br>**3.** Set the **sales thresholds** if needed (inside the “Calculate Risk” node)<br><br>**4.** Connect your **Slack account** and choose a channel<br><br>**5.** Set the **schedule time** (default: once daily)<br><br>**6.** Activate the workflow |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.  
   Name it something like: `"(Retail) Auto-Tag High-Risk SKUs"`.

2. **Add a Schedule Trigger node** named `Daily inventory risk check`.
   - Type: `Schedule Trigger`
   - Configure it to run once per day
   - Set trigger hour to `6`
   - Be aware this uses the n8n instance timezone

3. **Add a WooCommerce node** named `Get last 14 days orders `.
   - Type: `WooCommerce`
   - Credentials: connect a WooCommerce API credential with permission to read orders
   - Resource: `Order`
   - Operation: `Get Many` / `Get All`
   - Return All: enabled
   - Status filter: `completed`
   - `After` expression:
     ```js
     {{ new Date(Date.now() - 14 * 24 * 60 * 60 * 1000).toISOString()}}
     ```
   - `Before` expression:
     ```js
     {{ new Date().toISOString() }}
     ```
   - Connect `Daily inventory risk check -> Get last 14 days orders `

4. **Add another WooCommerce node** named `Get product details`.
   - Credentials: same WooCommerce credential
   - Resource: `Product`
   - Operation: `Get Many` / `Get All`
   - No extra filters required in this version
   - Connect `Get last 14 days orders  -> Get product details`

5. **Add a Split In Batches node** named `Process Products one by one`.
   - Type: `Split In Batches`
   - Keep default settings
   - Connect `Get product details -> Process Products one by one`

6. **Add a Code node** named `Calculate Sales & Risk level`.
   - Connect it from the second/main iteration output of `Process Products one by one`
   - Paste this logic:
     - Read the current product
     - Read all items from `Get last 14 days orders `
     - Sum matching quantities from order line items
     - Map quantities to risk levels
   - Implement the following thresholds:
     - `< 2` => `OK`
     - `>= 2` => `Watchlist`
     - `>= 6` => `High-Risk`
     - `>= 10` => `Critical`

   Use code equivalent to:

   ```js
   const product = $input.first().json;
   const orders = $items("Get last 14 days orders ").map(i => i.json);

   let unitsSold = 0;

   for (const order of orders) {
     for (const li of order.line_items || []) {
       if (li.product_id === product.id || li.variation_id === product.id) {
         unitsSold += li.quantity;
       }
     }
   }

   let risk = "OK";
   if (unitsSold >= 2) risk = "Watchlist";
   if (unitsSold >= 6) risk = "High-Risk";
   if (unitsSold >= 10) risk = "Critical";

   return [{
     json: {
       product_id: product.id,
       product_name: product.name,
       sku: product.sku,
       stock_status: product.stock_status,
       units_sold_last_14_days: unitsSold,
       risk_level: risk
     }
   }];
   ```

   Important:
   - Keep the node name `Get last 14 days orders ` exactly the same if you use this code as-is, including the trailing space, or update the code reference accordingly.

7. **Add a Switch node** named `Split product by risk level`.
   - Connect `Calculate Sales & Risk level -> Split product by risk level`
   - Create 4 rules based on `{{$json.risk_level}}`
   - Rule 1: equals `Watchlist`
   - Rule 2: equals `High-Risk`
   - Rule 3: equals `Critical`
   - Rule 4: equals `OK`

8. **Add a Set node** named `Prepare watchlist products`.
   - Connect from output 0 of the Switch
   - Add fields:
     - `product_id` = `{{$json.product_id}}` as Number
     - `tag` = `{{$json.risk_level}}` as String

9. **Add a WooCommerce node** named `Update Product tag as Watchlist`.
   - Connect `Prepare watchlist products -> Update Product tag as Watchlist`
   - Credentials: same WooCommerce credential
   - Resource: `Product`
   - Operation: `Update`
   - Product ID: `{{$json.product_id}}`
   - In update fields, set `Tags` to WooCommerce tag ID `74`

10. **Add a Set node** named `Prepare High-risk products`.
    - Connect from output 1 of the Switch
    - Fields:
      - `product_id` = `{{$json.product_id}}`
      - `tag` = `{{$json.risk_level}}`

11. **Add a WooCommerce node** named `Update Product tag as High-risk`.
    - Connect `Prepare High-risk products -> Update Product tag as High-risk`
    - Resource: `Product`
    - Operation: `Update`
    - Product ID: `{{$json.product_id}}`
    - Set `Tags` to tag ID `75`

12. **Add a Set node** named `Prepare Critical products`.
    - Connect from output 2 of the Switch
    - Fields:
      - `product_id` = `{{$json.product_id}}`
      - `tag` = `{{$json.risk_level}}`

13. **Add a WooCommerce node** named `Update Product tag as Critical`.
    - Connect `Prepare Critical products -> Update Product tag as Critical`
    - Resource: `Product`
    - Operation: `Update`
    - Product ID: `{{$json.product_id}}`
    - Set `Tags` to tag ID `76`

14. **Add a Merge node** named `Combine tagged products`.
    - Type: `Merge`
    - Set number of inputs to `3`
    - Connect:
      - `Update Product tag as Watchlist -> Combine tagged products` input 1
      - `Update Product tag as High-risk -> Combine tagged products` input 2
      - `Update Product tag as Critical -> Combine tagged products` input 3

15. **Add a Set node** named `Prepare data for Alerts`.
    - Connect `Combine tagged products -> Prepare data for Alerts`
    - Keep these fields:
      - `id` = `{{$json.id}}`
      - `sku` = `{{$json.sku}}`
      - `units_sold_last_14_days` = `{{ $('Split product by risk level').item.json.units_sold_last_14_days }}`
      - `Risk` = `{{$json.tags[0].name}}`
      - `name` = `{{$json.name}}`

16. **Add a Code node** named `Build slack alert message`.
    - Connect `Prepare data for Alerts -> Build slack alert message`
    - Use code equivalent to:

   ```js
   const items = $input.all();

   if (!items.length) {
     return [];
   }

   let message = `Inventory Risk Alert (Last 14 Days)\n\n`;
   message += `${items.length} product(s) require attention:\n\n`;

   for (const item of items) {
     const { id, name, sku, units_sold_last_14_days, Risk } = item.json;
     message += `• ${name} (ID: ${id})\n`;
     message += `  SKU: ${sku || 'N/A'}\n`;
     message += `  Sold: ${units_sold_last_14_days || 0}\n`;
     message += `  Risk: ${Risk}\n\n`;
   }

   return [{
     json: {
       slack_message: message.trim()
     }
   }];
   ```

17. **Add a Slack node** named `Notify team for Inventory alert`.
   - Connect `Build slack alert message -> Notify team for Inventory alert`
   - Credentials: connect a Slack app/account with permission to post messages
   - Operation: send message to channel
   - Text: `{{$json.slack_message}}`
   - Choose the target channel
   - Disable “include link to workflow” if you want to match this workflow

18. **Close the batch loop**.
   - Connect `Notify team for Inventory alert -> Process Products one by one`
   - This matches the original workflow exactly

19. **Leave the `OK` branch unconnected** from the Switch.
   - Products classified as `OK` are ignored for tagging and Slack alerts.

20. **Create WooCommerce product tags before activation**.
   - In WooCommerce, create tags corresponding to:
     - Watchlist
     - High-Risk
     - Critical
   - Replace tag IDs `74`, `75`, and `76` if your store uses different IDs

21. **Configure credentials**.
   - WooCommerce:
     - Base store URL
     - Consumer key
     - Consumer secret
     - Ensure read/write permission for products and read access to orders
   - Slack:
     - OAuth or bot token credential
     - Ensure permission to post to the selected channel

22. **Test with manual execution**.
   - Confirm that:
     - Orders are returned
     - Products are returned
     - Risk levels are assigned as expected
     - WooCommerce tags update correctly
     - Slack receives the message

23. **Activate the workflow** once validated.

### Reproduction cautions

24. **Be careful with node names used in expressions and code**.
   - `Get last 14 days orders ` includes a trailing space
   - `Split product by risk level` is referenced directly in an expression
   - Renaming these nodes requires updating all dependent expressions/code

25. **Validate WooCommerce tag behavior** before production use.
   - Some update operations may replace the full tag list rather than append one tag
   - If preserving existing tags matters, you may need to fetch current tags first and merge them manually

26. **Consider redesigning the loop for production stability**.
   - The current layout sends Slack from inside the batch loop
   - A stronger design would:
     - iterate all products
     - collect all risky products
     - send one Slack message after the loop ends

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow automatically checks your WooCommerce store every day and reviews product sales from the last 14 days. It counts how many units of each product were sold and assigns a risk level **OK** for normal sales, **Watchlist** when sales start increasing, **High-Risk** when stock may run out soon and **Critical** when immediate attention is needed. Based on this risk level, the workflow updates each product in WooCommerce with the correct tag, combines all risky products into a single list and sends a clear Slack alert showing the product name, units sold and risk level, helping you quickly identify fast-selling or low-stock items without manually checking reports. | General workflow behavior |
| Connect your **WooCommerce account** using API credentials. | Setup guidance |
| Make sure product tags exist in WooCommerce: **Watchlist**, **High-Risk**, **Critical**. | Setup guidance |
| Set the sales thresholds if needed inside the **Calculate Sales & Risk level** node. | Setup guidance |
| Connect your **Slack account** and choose a channel. | Setup guidance |
| Set the schedule time as needed; current default is once daily. | Setup guidance |
| Activate the workflow after configuration and testing. | Setup guidance |

## Additional implementation observations
- The workflow is currently **inactive** (`active: false`), so it will not run on schedule until activated.
- There is **one entry point only**: the Schedule Trigger.
- There are **no sub-workflows** and no workflow-invoking nodes.
- The title provided by the user, **“Tag high-risk WooCommerce SKUs and send daily alerts to Slack”**, matches the implemented business purpose, while the internal workflow name is **“(Retail) Auto-Tag High-Risk SKUs”**.
- The workflow uses n8n setting `executionOrder: v1`.