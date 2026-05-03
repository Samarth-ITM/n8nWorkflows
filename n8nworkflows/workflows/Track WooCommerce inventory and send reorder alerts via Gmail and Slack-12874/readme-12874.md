Track WooCommerce inventory and send reorder alerts via Gmail and Slack

https://n8nworkflows.xyz/workflows/track-woocommerce-inventory-and-send-reorder-alerts-via-gmail-and-slack-12874


# Track WooCommerce inventory and send reorder alerts via Gmail and Slack

## 1. Workflow Overview

**Purpose:** Monitor WooCommerce inventory and recent sales on a schedule, determine which products need replenishment, calculate a reorder quantity, then notify a purchasing team via **Gmail** and **Slack**.

**Primary use cases:**
- Prevent stockouts by proactively reordering when stock is low or demand is high.
- Reduce manual inventory checks by automating sales aggregation and reorder logic.
- Provide immediate visibility to stakeholders through email + Slack alerts.

### 1.1 Logical Blocks
1. **Inventory & Sales Intake**
   - Scheduled trigger → fetch WooCommerce orders → compute sold quantities per SKU.
2. **Product Mapping & Eligibility Validation**
   - Fetch WooCommerce products → merge sales with inventory → filter products that meet reorder conditions.
3. **Reorder Calculation & Notifications**
   - Compute reorder quantity → send Gmail email → send Slack alert.

---

## 2. Block-by-Block Analysis

### Block 1 — Inventory & Sales Intake
**Overview:** Runs on a schedule, pulls recent WooCommerce orders, and converts order line items into aggregated sales volume per SKU (soldQty). This creates the demand signal used later to decide reordering.

**Nodes involved:**  
- Inventory Check  
- Fetch Orders  
- SKU Sales Volume  

#### Node: Inventory Check
- **Type / role:** Schedule Trigger — entry point to start workflow automatically.
- **Configuration choices:** Runs at a specific hour via an interval rule (`triggerAtHour: 21`). (No minute/day-of-week constraints shown, so it relies on n8n’s schedule trigger defaults.)
- **Inputs / outputs:** No inputs (trigger). Output goes to **Fetch Orders**.
- **Version notes:** `typeVersion 1.2` (Schedule Trigger UI differs slightly across n8n versions).
- **Edge cases / failures:**
  - Timezone differences (n8n instance timezone vs business timezone) can cause runs at unexpected times.
  - If executions overlap (long WooCommerce fetch), consider concurrency settings or add a guard.

#### Node: Fetch Orders
- **Type / role:** WooCommerce node — retrieves order data (Orders API).
- **Configuration choices:**
  - **Resource:** Order
  - **Operation:** Get All
  - **Return All:** true (pulls all available orders matching default constraints)
  - **Options:** empty (no date filters / status filters configured)
- **Credentials:** WooCommerce API credential (REST API key/secret).
- **Inputs / outputs:** Input from **Inventory Check** → output to **SKU Sales Volume**.
- **Edge cases / failures:**
  - Pulling “all orders” can be slow/heavy and may hit pagination/rate limits; consider filtering by “modified after” or date range.
  - Auth errors if WooCommerce keys lack permissions.
  - Data shape differences: missing `line_items` in some orders.

#### Node: SKU Sales Volume
- **Type / role:** Code node (JavaScript) — aggregates sold quantities per SKU.
- **Configuration choices (logic):**
  - Iterates through each order item in `items`.
  - For each order, iterates `order.json.line_items || []`.
  - Skips line items without `sku`.
  - Sums `item.quantity` into a `sales[sku]` map.
  - Outputs one n8n item per SKU: `{ sku, soldQty }`.
- **Key variables / expressions:**
  - Uses `items` (n8n Code node input array).
  - Outputs: `return Object.entries(sales).map(...)`
- **Inputs / outputs:** Input from **Fetch Orders** → output to **Fetch Products**.
- **Edge cases / failures:**
  - Variations and some product types may not include SKU at line item level → they will be ignored, undercounting demand.
  - If order dataset is huge, this aggregation can be memory-heavy.
  - Quantities may be strings in some WooCommerce setups (usually numbers, but not guaranteed).

**Sticky note associated content (covers this block):**
- “Inventory & Sales Intake” note: explains scheduled start, order fetch, sales per SKU calculation, and mentions stopping on invalid/unavailable order data (note: there is **no explicit stop node**; to enforce this, you’d add an IF/Filter node after fetching orders).

---

### Block 2 — Product Mapping & Eligibility Validation
**Overview:** Fetches product inventory fields from WooCommerce, merges inventory with sales data, then filters products using OR logic (low stock OR high sales velocity).

**Nodes involved:**  
- Fetch Products  
- Merge Sales & Inventory Data  
- Reorder Check  

#### Node: Fetch Products
- **Type / role:** WooCommerce node — retrieves product inventory data.
- **Configuration choices:**
  - **Resource:** (not explicitly set in JSON; operation is `getAll`—in WooCommerce node this commonly implies Products resource when selected)
  - **Operation:** Get All
  - **Options:** empty
- **Credentials:** WooCommerce API credential.
- **Inputs / outputs:** Input from **SKU Sales Volume** → output to **Merge Sales & Inventory Data**.
- **Edge cases / failures:**
  - If it returns *all* products, performance can degrade significantly on large catalogs.
  - Some products may have `stock_quantity` null (unmanaged stock) and/or missing `low_stock_amount`.
  - Variable products: stock may exist on variations, not on the parent product.

#### Node: Merge Sales & Inventory Data
- **Type / role:** Merge node — combines streams so that downstream steps can reference both sales and inventory.
- **Configuration choices:** No explicit mode set in parameters (defaults apply). In n8n, the Merge node behavior depends on its **Mode** (e.g., Append, Merge By Index, Merge By Key).
- **Inputs / outputs:** Input 0 from **Fetch Products** (only one input connected in this workflow) → output to **Reorder Check**.
- **Important implication:** Because only one input is connected, this node effectively just passes through product items unless the node is configured in a special way. Additionally, downstream nodes reference sales via `$()` lookups rather than via merged item fields.
- **Edge cases / failures:**
  - If Merge mode expects two inputs and only one arrives, output can be empty depending on configuration/version.
  - If you intend “merge sales per SKU into each product”, a **Merge By Key (SKU)** setup typically requires:
    - Input 1: sales items keyed by `sku`
    - Input 2: product items keyed by `sku`
    - Then join by `sku`

#### Node: Reorder Check
- **Type / role:** Filter node — selects products eligible for reorder.
- **Configuration choices:**
  - Condition combinator: **OR**
  - Condition A: `stock_quantity <= low_stock_amount`
    - Left: `{{ $json.stock_quantity }}`
    - Right: `{{ $json.low_stock_amount }}`
  - Condition B: “sales velocity exceeds threshold”
    - Left: `{{ $('SKU Sales Volume').item.json.soldQty }}`
    - Right: `{{ $json.low_stock_amount * 1.5 }}`
- **Inputs / outputs:** Input from **Merge Sales & Inventory Data** → output to **Reorder Calc** (only items passing filter).
- **Edge cases / failures:**
  - `low_stock_amount` may be null/undefined → numeric comparisons can fail or evaluate unexpectedly under strict validation.
  - `$('SKU Sales Volume').item` depends on item pairing between streams; if the current product item does not correspond to the sales item at the same index, this can compare the wrong SKU’s sales.
  - For products with unmanaged stock (`stock_quantity` null), the `lte` condition may fail validation.

**Sticky note associated content (covers this block):**
- “Product Mapping & Eligibility Validation” note: describes merging sales + inventory and filtering with OR logic. (Implementation caveat: current Merge + `$()` references do not truly align by SKU unless the item ordering matches perfectly.)

---

### Block 3 — Reorder Calculation & Notifications
**Overview:** For products that pass reorder eligibility, computes a reorder quantity using daily average sales, lead time, and safety stock, then sends notifications via Gmail and Slack.

**Nodes involved:**  
- Reorder Calc  
- Send Email  
- Slack Alert  

#### Node: Reorder Calc
- **Type / role:** Code node (JavaScript) — calculates reorder quantity.
- **Configuration choices (logic):**
  - Constants:
    - `leadDays = 7`
    - `safetyStock = 5`
  - Computes:
    - `dailyAvg = ceil( soldQty / 7 )` using the **first** item of “SKU Sales Volume”
    - `reorderQty = dailyAvg * leadDays + safetyStock`
  - Returns one item with:
    - `sku` from `$('SKU Sales Volume').first().json.sku`
    - `productName` from `$('Fetch Products').first().json.name`
    - `currentStock` from `$('Fetch Products').first().json.stock_quantity`
- **Inputs / outputs:** Input from **Reorder Check** → output to **Send Email**.
- **Critical behavior detail:** This node uses `.first()` for sales and product references, meaning **it will always use the first SKU/product from those nodes**, not the currently filtered product item. If multiple products pass the filter, notifications may contain incorrect product data.
- **Edge cases / failures:**
  - If “SKU Sales Volume” output is empty, `.first()` will throw.
  - If “Fetch Products” output is empty, `.first()` will throw.
  - Sales period is hardcoded to 7 days but orders fetched are not restricted to last 7 days → daily average may be inaccurate.

#### Node: Send Email
- **Type / role:** Gmail node — sends an HTML email reorder notification.
- **Configuration choices:**
  - To: `user@example.com`
  - Subject: `Auto PO Created`
  - Message: HTML table including SKU, product name, current stock, reorder qty, and timestamp `{{ $now }}`
- **Credentials:** Gmail OAuth2.
- **Inputs / outputs:** Input from **Reorder Calc** → output to **Slack Alert**.
- **Version notes:** `typeVersion 2.1`.
- **Edge cases / failures:**
  - Gmail OAuth token expiration / missing scopes.
  - Sending limits/quota.
  - HTML injection risk is low here since data comes from WooCommerce, but still consider sanitization if SKUs/names can contain unexpected characters.

#### Node: Slack Alert
- **Type / role:** Slack node — posts a message to a channel.
- **Configuration choices:**
  - Mode: select channel
  - Channel: `n8n-demo` (ID `C0A3WQEKQ58`)
  - Text includes productName, sku, currentStock, reorderQty.
  - Message currently includes a “📦” symbol.
- **Credentials:** Slack API credential.
- **Inputs / outputs:** Input from **Send Email** → end.
- **Version notes:** `typeVersion 2.3`.
- **Edge cases / failures:**
  - Slack token missing `chat:write` or channel access.
  - Channel ID mismatch across workspaces/environments.

**Sticky note associated content (covers this block):**
- “Reorder Calculation & Notifications” note: describes lead time/safety stock reorder math and sending both email and Slack.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | Sticky Note | Documentation (Inventory & Sales Intake) |  |  | ## Inventory & Sales Intake • Trigger starts via schedule • Fetch orders via WooCommerce • Code calculates sales per SKU • Stop if order data unavailable/invalid (not implemented as a node) |
| Sticky Note1 | Sticky Note | Documentation (Product Mapping & Eligibility) |  |  | ## Product Mapping & Eligibility Validation • Fetch inventory and low-stock • Merge sales + inventory • OR-based validation • Exclude non-eligible products |
| Sticky Note2 | Sticky Note | Documentation (How it works / setup steps) |  |  | ## How it Works + Setup Steps: schedule → orders → code → products → merge → filter → reorder calc → email → slack |
| Sticky Note3 | Sticky Note | Documentation (Reorder + notifications) |  |  | ## Reorder Calculation & Notifications • Calculate reorder qty using avg sales, lead time, safety stock • Send email + Slack |
| Inventory Check | Schedule Trigger | Scheduled start of monitoring |  | Fetch Orders | ## Inventory & Sales Intake • Trigger starts via schedule • Fetch orders via WooCommerce • Code calculates sales per SKU • Stop if order data unavailable/invalid (not implemented as a node) |
| Fetch Orders | WooCommerce | Pull order data for sales aggregation | Inventory Check | SKU Sales Volume | ## Inventory & Sales Intake • Trigger starts via schedule • Fetch orders via WooCommerce • Code calculates sales per SKU • Stop if order data unavailable/invalid (not implemented as a node) |
| SKU Sales Volume | Code | Aggregate sold quantity per SKU from orders | Fetch Orders | Fetch Products | ## Inventory & Sales Intake • Trigger starts via schedule • Fetch orders via WooCommerce • Code calculates sales per SKU • Stop if order data unavailable/invalid (not implemented as a node) |
| Fetch Products | WooCommerce | Retrieve product stock + low stock threshold fields | SKU Sales Volume | Merge Sales & Inventory Data | ## Product Mapping & Eligibility Validation • Fetch inventory and low-stock • Merge sales + inventory • OR-based validation • Exclude non-eligible products |
| Merge Sales & Inventory Data | Merge | Combine/prepare datasets for eligibility check | Fetch Products | Reorder Check | ## Product Mapping & Eligibility Validation • Fetch inventory and low-stock • Merge sales + inventory • OR-based validation • Exclude non-eligible products |
| Reorder Check | Filter | Identify products needing reorder (OR rules) | Merge Sales & Inventory Data | Reorder Calc | ## Product Mapping & Eligibility Validation • Fetch inventory and low-stock • Merge sales + inventory • OR-based validation • Exclude non-eligible products |
| Reorder Calc | Code | Compute reorder quantity and shape alert payload | Reorder Check | Send Email | ## Reorder Calculation & Notifications • Calculate reorder qty using avg sales, lead time, safety stock • Send email + Slack |
| Send Email | Gmail | Email purchasing team reorder details | Reorder Calc | Slack Alert | ## Reorder Calculation & Notifications • Calculate reorder qty using avg sales, lead time, safety stock • Send email + Slack |
| Slack Alert | Slack | Real-time Slack notification | Send Email |  | ## Reorder Calculation & Notifications • Calculate reorder qty using avg sales, lead time, safety stock • Send email + Slack |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: *WooCommerce Inventory Reorder Automation with n8n, Gmail & Slack Alerts* (or your preferred name).

2. **Add Trigger: “Schedule Trigger”**
   - Node: **Schedule Trigger**
   - Name: **Inventory Check**
   - Set it to run daily at **21:00** (match `triggerAtHour: 21`).
   - Confirm the n8n instance timezone is correct for your business.

3. **Add WooCommerce Orders node**
   - Node: **WooCommerce**
   - Name: **Fetch Orders**
   - Resource: **Order**
   - Operation: **Get All**
   - Return All: **true**
   - Credentials: create/select **WooCommerce API** credential (store URL + consumer key/secret).
   - Connect: **Inventory Check → Fetch Orders**

4. **Add Code node to compute SKU sales**
   - Node: **Code**
   - Name: **SKU Sales Volume**
   - Paste this logic (same behavior as workflow):
     - Loop through `items`
     - Aggregate `line_items[].quantity` by `line_items[].sku`
     - Output items: `{ sku, soldQty }`
   - Connect: **Fetch Orders → SKU Sales Volume**

5. **Add WooCommerce Products node**
   - Node: **WooCommerce**
   - Name: **Fetch Products**
   - Resource: **Product** (select Product in the UI)
   - Operation: **Get All**
   - Credentials: same WooCommerce credential as above.
   - Connect: **SKU Sales Volume → Fetch Products**

6. **Add Merge node**
   - Node: **Merge**
   - Name: **Merge Sales & Inventory Data**
   - Keep default configuration (to match provided workflow).
   - Connect: **Fetch Products → Merge Sales & Inventory Data**
   - Note: If your intent is a true SKU join, configure Merge “By Key” and connect sales into the second input. The provided workflow does not do this.

7. **Add Filter node for reorder eligibility**
   - Node: **Filter**
   - Name: **Reorder Check**
   - Condition group: **OR**
   - Condition 1 (number): `stock_quantity` **lte** `low_stock_amount`
     - Left expression: `{{ $json.stock_quantity }}`
     - Right expression: `{{ $json.low_stock_amount }}`
   - Condition 2 (number): sales **gte** `low_stock_amount * 1.5`
     - Left expression: `{{ $('SKU Sales Volume').item.json.soldQty }}`
     - Right expression: `{{ $json.low_stock_amount * 1.5 }}`
   - Connect: **Merge Sales & Inventory Data → Reorder Check**

8. **Add Code node to compute reorder quantity**
   - Node: **Code**
   - Name: **Reorder Calc**
   - Configure:
     - leadDays = 7
     - safetyStock = 5
     - dailyAvg computed from `$('SKU Sales Volume').first().json.soldQty / 7`
     - reorderQty = dailyAvg * leadDays + safetyStock
     - Output JSON fields: `sku`, `productName`, `reorderQty`, `currentStock`
   - Connect: **Reorder Check → Reorder Calc**
   - Note: This reproduces the original behavior; it uses `.first()` references (which may not match the current product).

9. **Add Gmail node**
   - Node: **Gmail**
   - Name: **Send Email**
   - Operation: send email
   - To: `user@example.com` (replace)
   - Subject: `Auto PO Created`
   - Body: HTML table using `{{ $json.sku }}`, `{{ $json.productName }}`, `{{ $json.currentStock }}`, `{{ $json.reorderQty }}`, `{{ $now }}`
   - Credentials: **Gmail OAuth2** (authorize account; ensure send permissions).
   - Connect: **Reorder Calc → Send Email**

10. **Add Slack node**
   - Node: **Slack**
   - Name: **Slack Alert**
   - Operation: post message (channel)
   - Select: **Channel**
   - Channel: choose your channel (e.g., `n8n-demo`)
   - Text: include `{{ $json.productName }}`, `{{ $json.sku }}`, `{{ $json.currentStock }}`, `{{ $json.reorderQty }}`
   - Credentials: **Slack API** (OAuth token with channel posting permissions).
   - Connect: **Send Email → Slack Alert**

11. **(Optional) Add the sticky notes**
   - Add Sticky Notes for documentation and paste the provided content blocks as needed.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow monitors inventory + sales on schedule, calculates reorder quantities, and notifies via Gmail and Slack. | From the “How it Works” sticky note (overview + setup steps). |
| The notes mention stopping processing if order data is unavailable/invalid. | This is described in Sticky Note, but **no explicit node** enforces it; add an IF/Filter node after “Fetch Orders” if required. |
| Reorder eligibility is OR-based: low stock OR high sales velocity. | Implemented in **Reorder Check** filter conditions. |
| Current implementation relies on cross-node references (`$('...').first()` / `$('...').item`) rather than a true SKU-based join. | Potential mismatch risk when multiple products/SKUs exist; consider Merge “By Key (sku)” to align items correctly. |