Sync retail product prices between Shopify and WooCommerce with alerts

https://n8nworkflows.xyz/workflows/sync-retail-product-prices-between-shopify-and-woocommerce-with-alerts-12902


# Sync retail product prices between Shopify and WooCommerce with alerts

## 1. Workflow Overview

**Workflow name:** *(Retail) Price Change Sync*  
**Stated purpose/title:** *Sync retail product prices between Shopify and WooCommerce with alerts*

This workflow listens for pricing-related updates originating from either **WooCommerce** (product.updated) or **Shopify** (orders/updated), normalizes the incoming payload into a common schema, checks whether the change exceeds a configurable threshold (per source platform), optionally emails an alert for major price drops, applies platform-specific pricing rules (e.g., WooCommerce “.99” psychological rounding), then pushes updates to **both** platforms. Finally it merges results, logs the change into **Google Sheets**, and emails a “sync complete” message.

### Logical blocks
**1.1 Triggers & Source Normalization (Shopify vs WooCommerce)**  
Two entry points feed into platform-specific “Configuration” nodes that map each platform’s payload into a shared structure.

**1.2 Price Extraction & Threshold Alerting**  
A standard “Extract Price Data” node ensures required fields exist, then an IF node checks the % change against a threshold and triggers an email alert if large.

**1.3 Pricing Rules & Channel Formatting**  
A Code node calculates the WooCommerce and Shopify target prices, then two Set nodes format each platform’s expected field names/types.

**1.4 Platform Updates (Context Fetch + Update)**  
Each branch fetches context (Get many products/orders) and then updates the corresponding platform.

**1.5 Merge, Logging, and Completion Notification**  
Results from both updates are merged, written to Google Sheets (append or update), then a final Gmail notification is sent.

---

## 2. Block-by-Block Analysis

### 2.1 Triggers & Source Normalization (Shopify vs WooCommerce)

**Overview:**  
Starts the workflow from either platform and standardizes fields (oldPrice, newPrice, productId, sku, sourceSystem, threshold) so downstream logic can be platform-agnostic.

**Nodes involved:**
- **WooCommerce Price Update**
- **WooCommerce Configuration**
- **Shopify Price Update**
- **Shopify Configuration**

#### Node: WooCommerce Price Update
- **Type / role:** `wooCommerceTrigger` — webhook trigger for WooCommerce events.
- **Configuration (interpreted):**
  - Event: `product.updated`
  - Uses WooCommerce API credentials (WooCommerce account 9).
- **Outputs:** Sends the product payload into **WooCommerce Configuration**.
- **Edge cases / failures:**
  - Webhook not registered/expired in WooCommerce.
  - Credential or permission issues (401/403).
  - Event payload may omit fields (e.g., `regular_price` empty for variable products).
- **Version notes:** TypeVersion 1.

#### Node: WooCommerce Configuration
- **Type / role:** `Set` — maps WooCommerce product payload into normalized fields.
- **Key mappings:**
  - `oldPrice` = product `price`
  - `newPrice` = product `regular_price`
  - `priceChangeThreshold` = `"150"` (string)
  - `sourceSystem` = `"wooCommerce"`
  - `productId` = product `id` (number)
  - `sku` = product `sku`
  - Keeps other fields (`includeOtherFields: true`)
- **Outputs:** To **Extract Price Data**.
- **Edge cases:**
  - `price` vs `regular_price` can differ (sale pricing): may produce misleading “old vs new”.
  - For variable products, price fields may be empty or represent ranges.

#### Node: Shopify Price Update
- **Type / role:** `shopifyTrigger` — webhook trigger for Shopify topic.
- **Configuration (interpreted):**
  - Topic: `orders/updated`
  - Authentication: Access Token
  - Uses Shopify Access Token credentials (account 4).
- **Outputs:** To **Shopify Configuration**.
- **Edge cases:**
  - Topic is **orders** rather than **products**; “price sync” is derived from order totals/line item price and may not represent product catalog pricing.
  - Webhook delivery retries can cause duplicates.
- **Version notes:** TypeVersion 1.

#### Node: Shopify Configuration
- **Type / role:** `Set` — maps Shopify order payload into normalized fields.
- **Key mappings:**
  - `oldPrice` = first line item price: `$json.line_items[0].price`
  - `newPrice` = order `current_total_price`
  - `priceChangeThreshold` = `"500"` (string)
  - `sourceSystem` = `"shopify"`
  - `productId` = order `id` (number) *(note: this is an order id, not a product id)*
  - `sku` = line item `title` *(title is not SKU; SKU is often `line_items[0].sku`)*
  - Keeps other fields (`includeOtherFields: true`)
- **Outputs:** To **Extract Price Data**.
- **Edge cases:**
  - If `line_items` is empty or missing, expressions will fail.
  - Using `current_total_price` ties “price” to order totals, not product price.
  - Using title as SKU risks mismatch across systems.

**Sticky note coverage (applies to these nodes):**  
**“Multi-Platform Price Sync Triggers & Configuration”**  
> The process begins with either a Shopify or WooCommerce trigger that monitors for order or product updates within your stores. Following the trigger, dedicated Configuration nodes standardize the incoming data by mapping product IDs, SKUs and pricing details while establishing platform-specific thresholds to govern how price changes are synchronized.

**Also general workflow note covering setup:**  
**“How It Works”** (applies broadly; repeated in table where relevant)

---

### 2.2 Price Extraction & Threshold Alerting

**Overview:**  
Ensures downstream nodes have consistent field names, checks whether the percent price change exceeds a per-platform threshold, and sends an alert email for major changes.

**Nodes involved:**
- **Extract Price Data**
- **Check Price Change Threshold**
- **Alert Team - Major Price Drop**

#### Node: Extract Price Data
- **Type / role:** `Set` — normalizes/reshapes the incoming item to a small set of fields.
- **Key mappings:**
  - `oldPrice` = `$json.oldPrice`
  - `newPrice` = `$json.newPrice`
  - `sourceSystem` = `$json.sourceSystem`
  - `productId` = `$json.id` *(note: overwrites prior `productId` mapping intention; it uses `id` from current payload)*
  - `sku` = `$json.sku`
  - `priceChangeThreshold` = `$json.priceChangeThreshold`
- **Outputs:** To **Check Price Change Threshold**.
- **Edge cases:**
  - Confusion between `productId` and `id`: for Shopify orders, `id` is the order id; for WooCommerce, `id` is product id.
  - `oldPrice/newPrice` are strings; downstream relies on `.toNumber()`.

#### Node: Check Price Change Threshold
- **Type / role:** `If` — decides whether to send an alert.
- **Condition logic (AND):**
  1. `oldPrice.toNumber() > 0`
  2. `abs(((newPrice-oldPrice)/oldPrice)*100) >= priceChangeThreshold.toNumber()`
- **Outputs / branching:**
  - **True** → **Alert Team - Major Price Drop**
  - **False** → **Apply Platform-Specific Rules**
  - After the alert node, the workflow continues into **Apply Platform-Specific Rules** anyway.
- **Edge cases / failures:**
  - `.toNumber()` will yield `NaN` if prices are non-numeric strings (currency symbols, commas).
  - Division by zero prevented by condition #1, but only if oldPrice converts correctly.
  - Very high thresholds (e.g., 500%) mean alerts almost never fire.

#### Node: Alert Team - Major Price Drop
- **Type / role:** `gmail` — sends an alert email when threshold is exceeded.
- **Configuration (interpreted):**
  - To: `user@example.com`
  - Subject includes SKU: `Major Price Drop Alert - SKU: {{ $json.sku }}`
  - Body includes old/new price and percentage change:
    - `((newPrice-oldPrice)/oldPrice)*100` formatted to 2 decimals
  - Uses Gmail OAuth2 credential (Gmail account 8).
- **Outputs:** Continues to **Apply Platform-Specific Rules**.
- **Edge cases:**
  - Email formula uses `$json.timestamp` but no node sets `timestamp`; message will show blank unless present upstream.
  - If prices are strings, subtraction relies on implicit conversion—can produce `NaN` in the email body.

**Sticky note coverage (applies to these nodes):**  
**“Price Extraction, Validation & Channel Formatting”**  
> This segment extracts critical product identifiers and pricing data to calculate the percentage change between old and new values. An automated check compares this change against a set threshold, triggering a Gmail alert for major price drops while simultaneously applying platform-specific logic—such as rounding for psychological pricing on WooCommerce and standard formatting for Shopify—to ensure consistent and accurate pricing across all retail channels.

---

### 2.3 Pricing Rules & Channel Formatting

**Overview:**  
Computes the target platform prices (WooCommerce vs Shopify) using simple rules, then formats payloads into the shapes expected by each platform update node.

**Nodes involved:**
- **Apply Platform-Specific Rules**
- **FormatWooCommerce**
- **FormatShopify**

#### Node: Apply Platform-Specific Rules
- **Type / role:** `Code` — custom JavaScript to compute platform-specific prices.
- **Key logic:**
  - `basePrice = item.json.newPrice` *(no numeric conversion; potential type issue)*
  - WooCommerce: `wooPrice = Math.floor(basePrice) + 0.99`
  - Shopify: `shopifyPrice = Math.round(basePrice * 100) / 100`
  - Min thresholds: `wooPrice >= 0.99`, `shopifyPrice >= 1.00`
  - Writes: `item.json.wooPrice`, `item.json.shopifyPrice`, preserves `sku` and `productId`
- **Outputs:** Splits to both **FormatWooCommerce** and **FormatShopify**.
- **Edge cases:**
  - If `newPrice` is a string (common), `Math.floor("100.00")` works, but `"100.00" * 100` also works; if it contains currency formatting it will break.
  - For WooCommerce: `floor(basePrice)+0.99` ignores cents; a basePrice of 100.01 becomes 100.99 (increase).
  - If `basePrice` < 1, logic forces to 0.99/1.00.

#### Node: FormatWooCommerce
- **Type / role:** `Set` — formats fields for WooCommerce update.
- **Key mappings:**
  - `regular_price` = `wooPrice.toFixed(2)` (string)
  - `id` = `productId` (number)
  - `sku` = `sku`
- **Outputs:** To **Get many products**.
- **Edge cases:**
  - If `wooPrice` is undefined/NaN, `.toFixed(2)` throws an error.
  - WooCommerce update node later uses `$json.id`; this node ensures `id` is present.

#### Node: FormatShopify
- **Type / role:** `Set` — formats fields intended for Shopify update.
- **Key mappings:**
  - `price` = `shopifyPrice.toFixed(2)` (string)
  - `productId` = `productId`
  - `sku` = `sku`
  - Keeps other fields (`includeOtherFields: true`)
- **Outputs:** To **Get many orders**.
- **Edge cases:**
  - Same `.toFixed(2)` failure risk if `shopifyPrice` is not a number.
  - Downstream Shopify update node does **not** reference `price`—it references `$('FormatShopify').item.json.shopifyPrice`, which this node does not set (it sets `price`). This is a functional mismatch.

**Sticky note coverage (applies to these nodes):**  
Same as “Price Extraction, Validation & Channel Formatting” (above).

---

### 2.4 Platform Updates (Context Fetch + Update)

**Overview:**  
Fetches context from each platform (currently “get all” with limit 1) and then attempts to update prices in each system. Results are passed to the merge step.

**Nodes involved:**
- **Get many products**
- **Update WooCommerce Price**
- **Get many orders**
- **Update Shopify Price**

#### Node: Get many products
- **Type / role:** `wooCommerce` — retrieves products before updating (context fetch).
- **Configuration (interpreted):**
  - Operation: `getAll`
  - Limit: `1`
- **Input:** From **FormatWooCommerce**.
- **Output:** To **Update WooCommerce Price**.
- **Edge cases / concerns:**
  - This fetch is not filtered by `productId` or `sku`; it returns an arbitrary first page product, not necessarily the target product.
  - If the intent was to validate existence, it needs filtering or a “Get” by ID.
  - API pagination and permissions can affect results.

#### Node: Update WooCommerce Price
- **Type / role:** `wooCommerce` — updates a WooCommerce product’s regular price.
- **Configuration (interpreted):**
  - Resource: `product`
  - Operation: `update`
  - `productId` = `{{$json.id}}`
  - Update field `regularPrice` = `{{ $('FormatWooCommerce').item.json.regular_price }}`
- **Input:** From **Get many products** (not from the formatted target item).
- **Outputs:** To **Merge Platform Updates** (input 0).
- **Edge cases / failures:**
  - Because input comes from “Get many products”, `$json.id` is the fetched product id, not necessarily the intended `productId`.
  - If the “getAll” returns a different product, you may update the wrong product.
  - WooCommerce expects `regular_price` in API; node UI uses `regularPrice` but maps internally—ensure correct field mapping for your node version.
  - Authentication/permissions issues (401/403), validation errors (400).

#### Node: Get many orders
- **Type / role:** `shopify` — retrieves orders before updating.
- **Configuration (interpreted):**
  - Operation: `getAll`
  - Limit: `1`
  - Auth: access token
  - `alwaysOutputData: true`
- **Input:** From **FormatShopify**.
- **Output:** To **Update Shopify Price**.
- **Edge cases / concerns:**
  - Not filtered by orderId; returns first page order, not necessarily related to the triggering order.
  - If no orders exist, might output empty but continue due to `alwaysOutputData`.

#### Node: Update Shopify Price
- **Type / role:** `shopify` — updates a Shopify order (not a product) and writes the “price” into the **order note**.
- **Configuration (interpreted):**
  - Operation: `update`
  - `orderId` = `{{$json.id}}` (from input item)
  - Update field: `note` = `{{ $('FormatShopify').item.json.shopifyPrice }}`
  - Auth: access token
  - `alwaysOutputData: true`
- **Input:** From **Get many orders**.
- **Outputs:** To **Merge Platform Updates** (input 1).
- **Critical mismatches:**
  - It updates **orders**, not products/variants, so it does not actually change product prices in Shopify.
  - It references `shopifyPrice` from FormatShopify, but FormatShopify sets `price`, not `shopifyPrice`. This will likely resolve to `undefined`.
- **Edge cases / failures:**
  - Order update permissions may be restricted.
  - If `note` is set to undefined, you may overwrite notes unexpectedly or send invalid payloads.

**Sticky note coverage (applies to these nodes):**  
**“Multi-Platform Formatting and Price Synchronization”**  
> This phase begins with formatting nodes that structure the pricing data, IDs and SKUs into the specific requirements for Shopify and WooCommerce. This is followed by retrieval steps that fetch the necessary product and order context, ensuring the final update nodes can accurately synchronize the new prices across both e-commerce platforms.

---

### 2.5 Merge, Logging, and Completion Notification

**Overview:**  
Combines results from both platform update branches, logs the outcome to Google Sheets (“Price_Changes_Log”), then emails a completion notice.

**Nodes involved:**
- **Merge Platform Updates**
- **Log Price Changes**
- **Notify Team - Sync Complete**

#### Node: Merge Platform Updates
- **Type / role:** `Merge` — combines the two update result streams.
- **Configuration (interpreted):**
  - No explicit merge mode configured (defaults apply for Merge v3).
  - `alwaysOutputData: true`
- **Inputs:**
  - Input 0 from **Update WooCommerce Price**
  - Input 1 from **Update Shopify Price**
- **Output:** To **Log Price Changes**
- **Edge cases:**
  - If one branch errors or outputs zero items, merge behavior depends on default mode; may produce empty output or partial.
  - Item pairing can be inconsistent if each branch outputs different item counts.

#### Node: Log Price Changes
- **Type / role:** `googleSheets` — appends or updates a row in a Google Sheet.
- **Configuration (interpreted):**
  - Operation: `appendOrUpdate`
  - Document: `Price_Changes_Log` (spreadsheet ID `12mLi6...`)
  - Sheet name: `Sheet1`
  - Matching column: `id`
  - Mapping mode: auto-map input data
- **Output:** To **Notify Team - Sync Complete**
- **Edge cases / failures:**
  - Auto-mapping expects columns present; if the merged output doesn’t contain the schema, updates may not work as expected.
  - Matching on `id` is ambiguous: Shopify order IDs and WooCommerce product IDs may collide or be stored inconsistently.
  - Google Sheets rate limits, permission issues, or header/schema mismatch.

#### Node: Notify Team - Sync Complete
- **Type / role:** `gmail` — sends completion notification.
- **Configuration (interpreted):**
  - To: `user@example.com`
  - Subject: `Price Sync Complete - {{ $now.toFormat('yyyy-MM-dd HH:mm') }}`
  - Body includes timestamp and indicates updates + logging completed.
- **Edge cases:**
  - If upstream logging fails, this node won’t run unless error handling is configured.
  - Static recipient; consider environment-based configuration.

**Sticky note coverage (applies to these nodes):**  
**“Final Sync Logging and Notification”**  
> This final section handles the post-synchronization logic. The Merge Platform Updates node consolidates the results from both WooCommerce and Shopify. This unified data is then recorded in a Google Sheets log ("Price_Changes_Log") to maintain a history of all price adjustments. Finally, a Gmail notification is dispatched to the team, providing a timestamped summary to confirm that the entire synchronization process was completed successfully across all platforms.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Shopify Price Update | shopifyTrigger | Entry point from Shopify (orders/updated) | — | Shopify Configuration | Multi-Platform Price Sync Triggers & Configuration |
| WooCommerce Price Update | wooCommerceTrigger | Entry point from WooCommerce (product.updated) | — | WooCommerce Configuration | Multi-Platform Price Sync Triggers & Configuration |
| Shopify Configuration | Set | Normalize Shopify payload into common fields | Shopify Price Update | Extract Price Data | Multi-Platform Price Sync Triggers & Configuration |
| WooCommerce Configuration | Set | Normalize WooCommerce payload into common fields | WooCommerce Price Update | Extract Price Data | Multi-Platform Price Sync Triggers & Configuration |
| Extract Price Data | Set | Ensure required pricing fields exist for checks | Shopify Configuration, WooCommerce Configuration | Check Price Change Threshold | Price Extraction, Validation & Channel Formatting |
| Check Price Change Threshold | If | Compare % change vs threshold, route alert | Extract Price Data | Alert Team - Major Price Drop (true), Apply Platform-Specific Rules (false) | Price Extraction, Validation & Channel Formatting |
| Alert Team - Major Price Drop | Gmail | Email alert for large price changes | Check Price Change Threshold (true) | Apply Platform-Specific Rules | Price Extraction, Validation & Channel Formatting |
| Apply Platform-Specific Rules | Code | Compute WooCommerce/Shopify target prices | Check Price Change Threshold (false) / Alert Team - Major Price Drop | FormatWooCommerce, FormatShopify | Price Extraction, Validation & Channel Formatting |
| FormatWooCommerce | Set | Map computed price into WooCommerce fields | Apply Platform-Specific Rules | Get many products | Multi-Platform Formatting and Price Synchronization |
| Get many products | WooCommerce | Fetch products (currently unfiltered, limit 1) | FormatWooCommerce | Update WooCommerce Price | Multi-Platform Formatting and Price Synchronization |
| Update WooCommerce Price | WooCommerce | Update WooCommerce product regular price | Get many products | Merge Platform Updates | Multi-Platform Formatting and Price Synchronization |
| FormatShopify | Set | Map computed price into Shopify fields | Apply Platform-Specific Rules | Get many orders | Multi-Platform Formatting and Price Synchronization |
| Get many orders | Shopify | Fetch orders (currently unfiltered, limit 1) | FormatShopify | Update Shopify Price | Multi-Platform Formatting and Price Synchronization |
| Update Shopify Price | Shopify | Updates Shopify order note (not product price) | Get many orders | Merge Platform Updates | Multi-Platform Formatting and Price Synchronization |
| Merge Platform Updates | Merge | Combine update results from both platforms | Update WooCommerce Price, Update Shopify Price | Log Price Changes | Final Sync Logging and Notification |
| Log Price Changes | Google Sheets | Append/update change log in spreadsheet | Merge Platform Updates | Notify Team - Sync Complete | Final Sync Logging and Notification |
| Notify Team - Sync Complete | Gmail | Email completion notice | Log Price Changes | — | Final Sync Logging and Notification |
| Sticky Note10 | Sticky Note | Documentation (“How It Works”, setup steps) | — | — | # How It Works… (contains setup steps and connections guidance) |
| Sticky Note3 | Sticky Note | Documentation (trigger/config explanation) | — | — | Multi-Platform Price Sync Triggers & Configuration |
| Sticky Note | Sticky Note | Documentation (extraction/validation/formatting) | — | — | Price Extraction, Validation & Channel Formatting |
| Sticky Note1 | Sticky Note | Documentation (formatting + synchronization phase) | — | — | Multi-Platform Formatting and Price Synchronization |
| Sticky Note2 | Sticky Note | Documentation (logging + notification) | — | — | Final Sync Logging and Notification |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named **“(Retail) Price Change Sync”** (inactive by default, as in the JSON).

2. **Add trigger node (WooCommerce):**
   - Node: **WooCommerce Trigger**
   - Event: **product.updated**
   - Credentials: create/select **WooCommerce API** credential (consumer key/secret, store URL).
   - Name it: **WooCommerce Price Update**

3. **Add trigger node (Shopify):**
   - Node: **Shopify Trigger**
   - Topic: **orders/updated**
   - Authentication: **Access Token**
   - Credentials: create/select **Shopify Access Token API** credential.
   - Name it: **Shopify Price Update**

4. **Add normalization Set node for WooCommerce:**
   - Node: **Set**
   - Name: **WooCommerce Configuration**
   - Enable “Keep Only Set” = **off** (include other fields).
   - Add fields:
     - `oldPrice` = `{{$json.price}}`
     - `newPrice` = `{{$json.regular_price}}`
     - `priceChangeThreshold` = `150` *(as text or number, but downstream expects convertible)*
     - `sourceSystem` = `wooCommerce`
     - `productId` = `{{$json.id}}` (number)
     - `sku` = `{{$json.sku}}`
   - Connect: **WooCommerce Price Update → WooCommerce Configuration**

5. **Add normalization Set node for Shopify:**
   - Node: **Set**
   - Name: **Shopify Configuration**
   - Keep other fields: **on**
   - Add fields:
     - `oldPrice` = `{{$json.line_items[0].price}}`
     - `newPrice` = `{{$json.current_total_price}}`
     - `priceChangeThreshold` = `500`
     - `sourceSystem` = `shopify`
     - `productId` = `{{$json.id}}`
     - `sku` = `{{$json.line_items[0].title}}`
   - Connect: **Shopify Price Update → Shopify Configuration**

6. **Add “Extract Price Data” Set node:**
   - Node: **Set**
   - Name: **Extract Price Data**
   - Add fields:
     - `oldPrice` = `{{$json.oldPrice}}`
     - `newPrice` = `{{$json.newPrice}}`
     - `sourceSystem` = `{{$json.sourceSystem}}`
     - `productId` = `{{$json.id}}` *(mirrors JSON, though you may want `{{$json.productId}}` instead)*
     - `sku` = `{{$json.sku}}`
     - `priceChangeThreshold` = `{{$json.priceChangeThreshold}}`
   - Connect:
     - **WooCommerce Configuration → Extract Price Data**
     - **Shopify Configuration → Extract Price Data**

7. **Add IF node for threshold check:**
   - Node: **IF**
   - Name: **Check Price Change Threshold**
   - Conditions (AND):
     - `{{$json.oldPrice.toNumber()}}` **>** `0`
     - `{{ Math.abs((($json.newPrice.toNumber() - $json.oldPrice.toNumber()) / $json.oldPrice.toNumber()) * 100) }}` **>=** `{{$json.priceChangeThreshold.toNumber()}}`
   - Connect: **Extract Price Data → Check Price Change Threshold**

8. **Add Gmail node for major-change alert:**
   - Node: **Gmail** → “Send”
   - Name: **Alert Team - Major Price Drop**
   - Credentials: **Gmail OAuth2**
   - To: `user@example.com`
   - Subject: `Major Price Drop Alert - SKU: {{ $json.sku }}`
   - Message body: include old/new price and the % calculation (as in the workflow).
   - Connect: **Check Price Change Threshold (true) → Alert Team - Major Price Drop**

9. **Add Code node to compute platform-specific prices:**
   - Node: **Code**
   - Name: **Apply Platform-Specific Rules**
   - Paste the JS logic from the workflow (psychological pricing for WooCommerce, rounding for Shopify).
   - Connect:
     - **Check Price Change Threshold (false) → Apply Platform-Specific Rules**
     - **Alert Team - Major Price Drop → Apply Platform-Specific Rules**

10. **Add formatting nodes:**
   - Node: **Set**, name **FormatWooCommerce**
     - `regular_price` = `{{$json.wooPrice.toFixed(2)}}`
     - `id` = `{{$json.productId}}`
     - `sku` = `{{$json.sku}}`
   - Node: **Set**, name **FormatShopify**
     - `price` = `{{$json.shopifyPrice.toFixed(2)}}`
     - `productId` = `{{$json.productId}}`
     - `sku` = `{{$json.sku}}`
     - Keep other fields: **on**
   - Connect: **Apply Platform-Specific Rules → FormatWooCommerce** and **→ FormatShopify**

11. **Add WooCommerce “Get many products” node:**
   - Node: **WooCommerce**
   - Operation: **Get All**
   - Limit: **1**
   - Name: **Get many products**
   - Credentials: same WooCommerce API
   - Connect: **FormatWooCommerce → Get many products**

12. **Add WooCommerce update node:**
   - Node: **WooCommerce**
   - Resource: **Product**
   - Operation: **Update**
   - Product ID: `{{$json.id}}`
   - Update field regular price: `{{ $('FormatWooCommerce').item.json.regular_price }}`
   - Name: **Update WooCommerce Price**
   - Connect: **Get many products → Update WooCommerce Price**

13. **Add Shopify “Get many orders” node:**
   - Node: **Shopify**
   - Operation: **Get All**
   - Limit: **1**
   - Name: **Get many orders**
   - Credentials: Shopify Access Token
   - Connect: **FormatShopify → Get many orders**

14. **Add Shopify update node:**
   - Node: **Shopify**
   - Operation: **Update**
   - Order ID: `{{$json.id}}`
   - Update field `note`: `{{ $('FormatShopify').item.json.shopifyPrice }}`
   - Name: **Update Shopify Price**
   - (Optional) enable **Always Output Data** to match the workflow.
   - Connect: **Get many orders → Update Shopify Price**

15. **Add Merge node:**
   - Node: **Merge** (v3)
   - Name: **Merge Platform Updates**
   - Connect:
     - **Update WooCommerce Price → Merge Platform Updates (Input 1)**
     - **Update Shopify Price → Merge Platform Updates (Input 2)**

16. **Add Google Sheets logging node:**
   - Node: **Google Sheets**
   - Operation: **Append or Update**
   - Document: select spreadsheet **Price_Changes_Log**
   - Sheet: `Sheet1`
   - Matching column: `id`
   - Mapping: **Auto-map input data**
   - Name: **Log Price Changes**
   - Credentials: **Google Sheets OAuth2**
   - Connect: **Merge Platform Updates → Log Price Changes**

17. **Add Gmail completion notification node:**
   - Node: **Gmail** → “Send”
   - Name: **Notify Team - Sync Complete**
   - To: `user@example.com`
   - Subject/body include `{{$now.toISO()}}` / `{{$now.toFormat(...)}}`
   - Connect: **Log Price Changes → Notify Team - Sync Complete**

18. **Add sticky notes** (optional, for documentation):
   - Copy the content from the workflow’s sticky notes and place them over the respective sections.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| **How It Works**: automates synchronization and optimization of pricing between Shopify and WooCommerce; setup steps for triggers, connections (credentials), thresholds, logging, and Gmail notifications. | Sticky note “How It Works” (covers overall workflow) |
| Multi-Platform Price Sync Triggers & Configuration: triggers + config nodes standardize payload and thresholds. | Sticky note for initial block |
| Price Extraction, Validation & Channel Formatting: percent-change calculation, alerting, and platform-specific logic. | Sticky note for validation/rules block |
| Multi-Platform Formatting and Price Synchronization: formatting nodes + retrieval steps + update nodes. | Sticky note for sync block |
| Final Sync Logging and Notification: merge results, log to Google Sheets, send completion email. | Sticky note for final block |
| Disclaimer (FR): *Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n…* | Provided by user with the workflow |

