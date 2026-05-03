Sync Shopify orders to Odoo sales orders with customer and product mapping

https://n8nworkflows.xyz/workflows/sync-shopify-orders-to-odoo-sales-orders-with-customer-and-product-mapping-14828


# Sync Shopify orders to Odoo sales orders with customer and product mapping

# 1. Workflow Overview

This workflow receives a Shopify order webhook and creates a corresponding draft Sales Order in Odoo. It also ensures the customer exists in Odoo, maps Shopify line items to Odoo products via SKU/barcode, adds shipping as a service line, and builds internal order notes from Shopify metadata.

Typical use cases:
- Sync new Shopify orders into Odoo for downstream fulfillment or accounting
- Avoid manual sales order entry
- Keep Odoo customer records aligned with incoming Shopify buyers
- Flag unmatched products without blocking order creation

## 1.1 Input Reception and Order Normalization
The workflow starts from a webhook that accepts Shopify order payloads. A code node then transforms the raw Shopify payload into a simplified internal structure with customer, shipping, totals, payment, and line items.

## 1.2 Customer Lookup and Conditional Creation
The workflow searches Odoo for an existing contact using the customer phone number. If no customer is found, it looks up the state/province record in Odoo and creates a new contact.

## 1.3 Product Matching and Sales Order Line Construction
The workflow splits Shopify line items into one item per execution item, searches Odoo products by barcode/SKU, computes discounts, and re-aggregates all lines into Odoo’s expected `order_line` command format.

## 1.4 Order Finalization and Odoo Sales Order Creation
The workflow optionally adds a shipping service line, composes order notes and references, and creates a draft `sale.order` in Odoo.

---

# 2. Block-by-Block Analysis

## Block 1 — Input Reception and Order Normalization

### Overview
This block receives the Shopify webhook and converts the raw payload into a cleaner, structured object used by all later nodes. It centralizes field extraction and fallback logic so downstream nodes work with normalized data.

### Nodes Involved
- Shopify Webhook
- Parse Shopify Order1

### Node Details

#### 1. Shopify Webhook
- **Type and technical role:** `n8n-nodes-base.webhook`; entry point that listens for incoming HTTP POST requests from Shopify.
- **Configuration choices:**
  - HTTP Method: `POST`
  - Path: `shopify-order-webhook`
  - No special response handling configured
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input: none
  - Output: `Parse Shopify Order1`
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:**
  - Shopify webhook not configured correctly in Shopify Admin
  - Wrong webhook event type
  - Invalid public URL or inactive workflow
  - Payload shape may differ if Shopify sends test payloads or custom app payloads

#### 2. Parse Shopify Order1
- **Type and technical role:** `n8n-nodes-base.code`; parses raw Shopify order data into a normalized internal schema.
- **Configuration choices:**
  - Reads order from `body` if present, otherwise directly from input JSON
  - Extracts:
    - Shopify order ID and order name
    - Customer identity and contact info
    - Shipping address
    - Shipping method and shipping cost
    - Payment and currency info
    - Totals and discount info
    - Simplified line items
    - Note and tags
  - Uses fallback logic for customer name and phone from either `customer`, `shipping_address`, or order-level fields
- **Key expressions or variables used:**
  - `const order = $input.first().json.body || $input.first().json;`
  - Constructs nested objects:
    - `customer`
    - `shipping`
    - `payment`
    - `totals`
    - `line_items`
- **Input and output connections:**
  - Input: `Shopify Webhook`
  - Output: `Search Customer in Odoo1`
- **Version-specific requirements:** Type version `2`
- **Edge cases or potential failure types:**
  - Missing `customer` object
  - Missing `shipping_address`
  - Empty `line_items`
  - Non-numeric prices causing `parseFloat` to become `NaN`
  - Order payload shape may vary between Shopify event types
- **Sub-workflow reference:** None

---

## Block 2 — Customer Lookup and Conditional Creation

### Overview
This block checks whether the Shopify customer already exists in Odoo by phone number. If the customer is missing, it resolves the Odoo state/province and creates a new contact before continuing.

### Nodes Involved
- Search Customer in Odoo1
- Customer Exists?1
- Get state id1
- Create Contact1

### Node Details

#### 3. Search Customer in Odoo1
- **Type and technical role:** `n8n-nodes-base.odoo`; searches Odoo contacts in `res.partner`.
- **Configuration choices:**
  - Resource: `custom`
  - Custom resource: `res.partner`
  - Operation: `getAll`
  - Returns all matching records
  - Limited output fields: `id`
  - Filter by `phone_mobile_search` equal to parsed customer phone
  - `alwaysOutputData: true` ensures the workflow continues even if no match is found
- **Key expressions or variables used:**
  - `={{ $json.customer.phone }}`
- **Input and output connections:**
  - Input: `Parse Shopify Order1`
  - Output: `Customer Exists?1`
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:**
  - Odoo credential/authentication issues
  - `phone_mobile_search` may not exist or behave differently depending on Odoo setup/custom modules
  - Empty phone numbers may return no result or incorrect broad matches depending on Odoo implementation
  - Multiple contacts may match; downstream logic assumes first available record
- **Sub-workflow reference:** None

#### 4. Customer Exists?1
- **Type and technical role:** `n8n-nodes-base.if`; routes execution based on whether a customer ID was found.
- **Configuration choices:**
  - Condition checks whether `{{$json.id}}` is empty
  - If empty: customer not found
  - If not empty: customer exists
- **Key expressions or variables used:**
  - `={{ $json.id }}`
- **Input and output connections:**
  - Input: `Search Customer in Odoo1`
  - Output 0 (true): `Get state id1`
  - Output 1 (false): `Split Line Items`
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:**
  - If Odoo returns multiple items, IF evaluates per item; behavior is acceptable here but should be understood
  - If returned object shape changes, `.id` may not exist
- **Sub-workflow reference:** None

#### 5. Get state id1
- **Type and technical role:** `n8n-nodes-base.odoo`; searches Odoo states/provinces in `res.country.state`.
- **Configuration choices:**
  - Resource: `custom`
  - Custom resource: `res.country.state`
  - Operation: `getAll`
  - Return all matches
  - Filter where state `name` is like Shopify shipping province
- **Key expressions or variables used:**
  - `={{ $('Parse Shopify Order1').first().json.shipping.province }}`
- **Input and output connections:**
  - Input: `Customer Exists?1` (true branch)
  - Output: `Create Contact1`
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:**
  - Province names may not match Odoo naming conventions
  - `like` may return multiple states
  - Empty province can return no result
  - If no state is found, downstream `state_id` may be empty or invalid
- **Sub-workflow reference:** None

#### 6. Create Contact1
- **Type and technical role:** `n8n-nodes-base.odoo`; creates a new contact in Odoo `res.partner`.
- **Configuration choices:**
  - Resource: `custom`
  - Custom resource: `res.partner`
  - Creates fields:
    - `name` from parsed customer full name
    - `phone`
    - `email`
    - `street`
    - `city`
    - `country_id` hardcoded to `65`
    - `lang` hardcoded to `ar_001`
    - `state_id` from previous Odoo state lookup
- **Key expressions or variables used:**
  - `={{ $('Parse Shopify Order1').first().json.customer.full_name }}`
  - `={{ $('Parse Shopify Order1').first().json.customer.phone }}`
  - `={{ $('Parse Shopify Order1').first().json.customer.email }}`
  - `={{ $('Parse Shopify Order1').first().json.shipping.address1 }}`
  - `={{ $('Parse Shopify Order1').first().json.shipping.city }}`
  - `={{ $json.id }}`
- **Input and output connections:**
  - Input: `Get state id1`
  - Output: `Split Line Items`
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:**
  - Hardcoded `country_id=65` may be wrong in another Odoo database
  - `state_id` may be missing if no state matched
  - Required fields may differ depending on Odoo customizations
  - Duplicate customer creation may happen if phone search is unreliable or phone format differs
- **Sub-workflow reference:** None

---

## Block 3 — Product Matching and Order Build

### Overview
This block turns each Shopify line item into an individual lookup, searches products in Odoo by barcode, then reassembles the results into Odoo sales order lines. It also preserves unmatched products as descriptive manual lines so order creation still succeeds.

### Nodes Involved
- Split Line Items
- Search Product by Barcode1
- Process Product Result1
- Aggregate Products1

### Node Details

#### 7. Split Line Items
- **Type and technical role:** `n8n-nodes-base.code`; explodes the order into one item per Shopify line item.
- **Configuration choices:**
  - Reads normalized order from `Parse Shopify Order1`
  - Reads current input item for `partnerId`; this works whether the customer came from search or creation
  - Emits one item per line item with:
    - `partner_id`
    - `sku`
    - `title`
    - `variant_title`
    - `quantity`
    - `price_unit`
    - `total_discount`
    - Shopify product/variant IDs
    - `_order_meta` only on the first emitted item
- **Key expressions or variables used:**
  - `const orderData = $('Parse Shopify Order1').first().json;`
  - `const inputData = $input.first().json;`
  - `const partnerId = inputData.id;`
- **Input and output connections:**
  - Input: from either `Create Contact1` or `Customer Exists?1` false branch
  - Output: `Search Product by Barcode1`
- **Version-specific requirements:** Type version `2`
- **Edge cases or potential failure types:**
  - If no line items exist, returns an empty array and downstream nodes may not execute
  - If customer search returned multiple records, only first input item is used
  - If `id` is missing on incoming item, partner mapping fails
- **Sub-workflow reference:** None

#### 8. Search Product by Barcode1
- **Type and technical role:** `n8n-nodes-base.odoo`; looks up Odoo products for each split line item.
- **Configuration choices:**
  - Resource: `custom`
  - Custom resource: `product.product`
  - Operation: `getAll`
  - Limit: `1`
  - Requested fields:
    - `id`
    - `name`
    - `default_code`
    - `barcode`
    - `uom_id`
  - Filter by `barcode` equal to item SKU
  - `alwaysOutputData: true` preserves flow if no product is found
- **Key expressions or variables used:**
  - `={{ $json.sku }}`
- **Input and output connections:**
  - Input: `Split Line Items`
  - Output: `Process Product Result1`
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:**
  - SKU and barcode may not be the same in Odoo
  - Empty SKU means search may fail for all such items
  - If product exists but barcode field is blank in Odoo, it will not match
  - Odoo auth, permission, or model access issues
- **Sub-workflow reference:** None

#### 9. Process Product Result1
- **Type and technical role:** `n8n-nodes-base.code`; reconciles product search results with original split items and computes discount percentages.
- **Configuration choices:**
  - Loads all product search results with `$input.all()`
  - Loads all original split items with `$('Split Line Items').all()`
  - Builds a `barcodeMap`
  - For each split item:
    - attaches `product_id` if found
    - falls back to original title if not found
    - builds `display_name`
    - computes Odoo-compatible discount percentage from Shopify total discount
- **Key expressions or variables used:**
  - `const allSearchResults = $input.all();`
  - `const allSplitItems = $('Split Line Items').all();`
  - Discount formula:
    - `(totalDiscount / (qty * unitPrice)) * 100`
- **Input and output connections:**
  - Input: `Search Product by Barcode1`
  - Output: `Aggregate Products1`
- **Version-specific requirements:** Type version `2`
- **Edge cases or potential failure types:**
  - If multiple items share the same SKU, barcode map collapses to one product entry; usually acceptable
  - Discount calculation guards against division by zero
  - If Odoo returns no `barcode`, the result will not map
  - Reliance on parallel item ordering is reduced by barcode mapping, but duplicate barcodes can still be ambiguous
- **Sub-workflow reference:** None

#### 10. Aggregate Products1
- **Type and technical role:** `n8n-nodes-base.code`; aggregates all processed items into a single Odoo order payload fragment.
- **Configuration choices:**
  - Reads all processed products
  - Takes `partner_id` from the first item
  - Extracts `_order_meta` from the first item carrying it
  - Splits products into:
    - matched products → proper Odoo product lines
    - unmatched products → descriptive manual lines prefixed with `[NOT FOUND]`
  - Builds Odoo many2many/one2many command arrays:
    - `[0, 0, {...}]`
- **Key expressions or variables used:**
  - `const allProducts = $input.all();`
  - `const foundProducts = ...product_id !== null`
  - `const notFoundProducts = ...product_id === null`
- **Input and output connections:**
  - Input: `Process Product Result1`
  - Output: `Add Shipping to Order1`
- **Version-specific requirements:** Type version `2`
- **Edge cases or potential failure types:**
  - If there are no input items, referencing `allProducts[0]` will fail
  - Manual lines for not found products may be acceptable in Odoo, but accounting/tax behavior can differ
  - Tax list is forced empty via `tax_id: [[6, 0, []]]`
- **Sub-workflow reference:** None

---

## Block 4 — Order Finalization and Odoo Sales Order Creation

### Overview
This block finalizes the order payload by appending shipping, composing internal notes, and creating the Odoo sales order. It relies on several hardcoded Odoo IDs that must match the target Odoo instance.

### Nodes Involved
- Add Shipping to Order1
- Create Sales Order in Odoo1

### Node Details

#### 11. Add Shipping to Order1
- **Type and technical role:** `n8n-nodes-base.code`; prepares the final sales order payload.
- **Configuration choices:**
  - Reads aggregated order line data
  - If shipping cost is greater than zero, appends a shipping line with:
    - `product_id: 16`
    - Arabic line name `مصاريف الشحن`
  - Builds note text including:
    - Shopify order number
    - Shopify ID
    - Payment status and gateways
    - Shipping method
    - Customer note
  - Outputs final order-level fields:
    - `partner_id`
    - `order_line`
    - `client_order_ref`
    - `origin`
    - `note`
    - `source_id`
    - product match counters
- **Key expressions or variables used:**
  - `const shippingCost = orderMeta.shipping_cost || 0;`
  - hardcoded `product_id: 16`
  - hardcoded `source_id: 17`
- **Input and output connections:**
  - Input: `Aggregate Products1`
  - Output: `Create Sales Order in Odoo1`
- **Version-specific requirements:** Type version `2`
- **Edge cases or potential failure types:**
  - Shipping product ID `16` may not exist in another Odoo database
  - Shipping line uses empty taxes; may not match business rules
  - If `order_meta` is null, note content becomes sparse
  - Arabic label may be intentional but should match ERP language expectations
- **Sub-workflow reference:** None

#### 12. Create Sales Order in Odoo1
- **Type and technical role:** `n8n-nodes-base.odoo`; creates the final Odoo `sale.order`.
- **Configuration choices:**
  - Resource: `custom`
  - Custom resource: `sale.order`
  - Creates fields:
    - `partner_id` from current item
    - `client_order_ref`
    - `origin`
    - `note`
    - `source_id` hardcoded to `17`
    - `order_line` from `Add Shipping to Order1`
    - `pricelist_id` hardcoded to `9`
    - `user_id` hardcoded to `12`
- **Key expressions or variables used:**
  - `={{ $json.partner_id }}`
  - `={{ $json.client_order_ref }}`
  - `={{ $json.origin }}`
  - `={{ $json.note }}`
  - `={{ $('Add Shipping to Order1').first().json.order_line }}`
- **Input and output connections:**
  - Input: `Add Shipping to Order1`
  - Output: none
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:**
  - Hardcoded IDs `17`, `9`, and `12` must exist in target Odoo
  - Odoo validation may fail if any required field or relation is invalid
  - Duplicate orders may be created if webhook retries and no deduplication exists
  - If order lines are malformed, Odoo will reject the create request
- **Sub-workflow reference:** None

---

## Additional Non-Executable Nodes

These nodes document the workflow visually and do not affect execution.

### 13. Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`; documentation note for general setup and operating logic.
- **Configuration choices:** Contains setup guidance and hardcoded ID reminders.
- **Connections:** none
- **Potential issues:** none

### 14. Sticky Note 1
- **Type and technical role:** `n8n-nodes-base.stickyNote`; labels the receive/parse section.
- **Connections:** none

### 15. Sticky Note 2
- **Type and technical role:** `n8n-nodes-base.stickyNote`; labels the customer lookup/creation section.
- **Connections:** none

### 16. Sticky Note 3
- **Type and technical role:** `n8n-nodes-base.stickyNote`; labels the product matching and order build section.
- **Connections:** none

### 17. Sticky Note 4
- **Type and technical role:** `n8n-nodes-base.stickyNote`; labels the finalization section.
- **Connections:** none

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Shopify Webhook | Webhook | Receives Shopify order creation webhook |  | Parse Shopify Order1 | ### How it works<br>This workflow automatically syncs new Shopify orders to Odoo as Sales Orders.<br>It parses the order payload, checks if the customer exists by phone number (creating them if not), matches products via SKU/Barcode, applies shipping costs, and generates a draft Sales Order in Odoo.<br><br>### Setup steps<br>1. **Webhook:** Add the Webhook URL to your Shopify Admin > Notifications > Webhooks (Order Creation).<br>2. **Odoo Credentials:** Connect your Odoo account in n8n.<br>3. **Shipping Product:** Open the "Add Shipping to Order" code node and update `product_id: 16` to match your Odoo Shipping service ID.<br>4. **Sales Order Details:** Open the "Create Sales Order" node and adjust `pricelist_id`, `user_id`, and `source_id` to match your system.<br>## 1. Receive & Parse Order<br>Catches the incoming webhook payload from Shopify and uses custom code to extract and format only the required customer, shipping, and product details. |
| Parse Shopify Order1 | Code | Normalizes raw Shopify order payload | Shopify Webhook | Search Customer in Odoo1 | ## 1. Receive & Parse Order<br>Catches the incoming webhook payload from Shopify and uses custom code to extract and format only the required customer, shipping, and product details. |
| Search Customer in Odoo1 | Odoo | Searches Odoo contact by phone | Parse Shopify Order1 | Customer Exists?1 | ## 2. Customer Lookup & Creation<br>Searches Odoo for an existing customer using their phone number. If they don't exist, it resolves their State/Province ID and dynamically creates a new Contact profile before proceeding. |
| Customer Exists?1 | If | Branches based on whether customer record was found | Search Customer in Odoo1 | Get state id1 / Split Line Items | ## 2. Customer Lookup & Creation<br>Searches Odoo for an existing customer using their phone number. If they don't exist, it resolves their State/Province ID and dynamically creates a new Contact profile before proceeding. |
| Get state id1 | Odoo | Finds Odoo state/province ID for new customer | Customer Exists?1 | Create Contact1 | ## 2. Customer Lookup & Creation<br>Searches Odoo for an existing customer using their phone number. If they don't exist, it resolves their State/Province ID and dynamically creates a new Contact profile before proceeding. |
| Create Contact1 | Odoo | Creates Odoo contact when no existing customer is found | Get state id1 | Split Line Items | ## 2. Customer Lookup & Creation<br>Searches Odoo for an existing customer using their phone number. If they don't exist, it resolves their State/Province ID and dynamically creates a new Contact profile before proceeding. |
| Split Line Items | Code | Emits one workflow item per Shopify line item | Customer Exists?1 / Create Contact1 | Search Product by Barcode1 | ## 3. Product Matching & Order Build<br>Splits the Shopify line items to search Odoo by SKU/Barcode individually. It then matches the results, calculates equivalent discounts, and aggregates everything back into a valid Odoo `order_line` payload format. |
| Search Product by Barcode1 | Odoo | Searches Odoo product by SKU mapped to barcode | Split Line Items | Process Product Result1 | ## 3. Product Matching & Order Build<br>Splits the Shopify line items to search Odoo by SKU/Barcode individually. It then matches the results, calculates equivalent discounts, and aggregates everything back into a valid Odoo `order_line` payload format. |
| Process Product Result1 | Code | Merges search results with split items and computes discounts | Search Product by Barcode1 | Aggregate Products1 | ## 3. Product Matching & Order Build<br>Splits the Shopify line items to search Odoo by SKU/Barcode individually. It then matches the results, calculates equivalent discounts, and aggregates everything back into a valid Odoo `order_line` payload format. |
| Aggregate Products1 | Code | Builds Odoo order line command array | Process Product Result1 | Add Shipping to Order1 | ## 3. Product Matching & Order Build<br>Splits the Shopify line items to search Odoo by SKU/Barcode individually. It then matches the results, calculates equivalent discounts, and aggregates everything back into a valid Odoo `order_line` payload format. |
| Add Shipping to Order1 | Code | Appends shipping line and final order metadata | Aggregate Products1 | Create Sales Order in Odoo1 | ## 3. Product Matching & Order Build<br>Splits the Shopify line items to search Odoo by SKU/Barcode individually. It then matches the results, calculates equivalent discounts, and aggregates everything back into a valid Odoo `order_line` payload format.<br>## 4. Finalize Order<br>Builds the final double-entry payload (including shipping costs and dynamic notes) and pushes the Draft Sales Order directly to your Odoo ERP. |
| Create Sales Order in Odoo1 | Odoo | Creates draft sales order in Odoo | Add Shipping to Order1 |  | ## 4. Finalize Order<br>Builds the final double-entry payload (including shipping costs and dynamic notes) and pushes the Draft Sales Order directly to your Odoo ERP. |
| Sticky Note | Sticky Note | Documentation/setup note |  |  |  |
| Sticky Note 1 | Sticky Note | Visual block annotation |  |  |  |
| Sticky Note 2 | Sticky Note | Visual block annotation |  |  |  |
| Sticky Note 3 | Sticky Note | Visual block annotation |  |  |  |
| Sticky Note 4 | Sticky Note | Visual block annotation |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n**
   - Give it a name such as: `Sync Shopify orders to Odoo sales orders with customer and product mapping`.

2. **Add a Webhook node**
   - Node type: `Webhook`
   - Name: `Shopify Webhook`
   - HTTP Method: `POST`
   - Path: `shopify-order-webhook`
   - Keep response settings at default unless you need a custom acknowledgment.
   - Copy the production webhook URL later into Shopify.

3. **Add a Code node after the webhook**
   - Node type: `Code`
   - Name: `Parse Shopify Order1`
   - Paste logic that:
     - reads `body` if present
     - extracts customer, shipping, totals, payment, line items, note, tags
     - converts price strings to numbers with `parseFloat`
   - Use this node to produce a single normalized object containing:
     - `shopify_order_id`
     - `shopify_order_name`
     - `created_at`
     - `customer`
     - `shipping`
     - `shipping_method`
     - `shipping_cost`
     - `payment`
     - `totals`
     - `line_items`
     - `note`
     - `tags`

4. **Connect `Shopify Webhook` to `Parse Shopify Order1`**

5. **Add an Odoo node to search customers**
   - Node type: `Odoo`
   - Name: `Search Customer in Odoo1`
   - Credentials: connect valid Odoo credentials
   - Resource: `Custom`
   - Custom Resource: `res.partner`
   - Operation: `Get All`
   - Return All: `true`
   - Fields list: include `id`
   - Add filter:
     - Field Name: `phone_mobile_search`
     - Value: `={{ $json.customer.phone }}`
   - Enable `Always Output Data`

6. **Connect `Parse Shopify Order1` to `Search Customer in Odoo1`**

7. **Add an If node**
   - Node type: `If`
   - Name: `Customer Exists?1`
   - Configure a string condition:
     - Value 1: `={{ $json.id }}`
     - Operation: `isEmpty`
   - This means:
     - **true branch** = customer not found
     - **false branch** = customer exists

8. **Connect `Search Customer in Odoo1` to `Customer Exists?1`**

9. **Add an Odoo node to resolve state/province**
   - Node type: `Odoo`
   - Name: `Get state id1`
   - Credentials: same Odoo credentials
   - Resource: `Custom`
   - Custom Resource: `res.country.state`
   - Operation: `Get All`
   - Return All: `true`
   - Filter:
     - Field Name: `name`
     - Operator: `like`
     - Value: `={{ $('Parse Shopify Order1').first().json.shipping.province }}`

10. **Connect the true output of `Customer Exists?1` to `Get state id1`**

11. **Add an Odoo node to create a contact**
   - Node type: `Odoo`
   - Name: `Create Contact1`
   - Resource: `Custom`
   - Custom Resource: `res.partner`
   - Configure create/update fields:
     - `name` = `={{ $('Parse Shopify Order1').first().json.customer.full_name }}`
     - `phone` = `={{ $('Parse Shopify Order1').first().json.customer.phone }}`
     - `email` = `={{ $('Parse Shopify Order1').first().json.customer.email }}`
     - `street` = `={{ $('Parse Shopify Order1').first().json.shipping.address1 }}`
     - `city` = `={{ $('Parse Shopify Order1').first().json.shipping.city }}`
     - `country_id` = `65`
     - `lang` = `ar_001`
     - `state_id` = `={{ $json.id }}`
   - Important: replace `65` with the correct country record ID in your Odoo database if needed.

12. **Connect `Get state id1` to `Create Contact1`**

13. **Add a Code node to split line items**
   - Node type: `Code`
   - Name: `Split Line Items`
   - Configure code to:
     - read the normalized order from `Parse Shopify Order1`
     - read the current customer/contact item from input
     - extract `partner_id` from `inputData.id`
     - output one item per Shopify line item
     - include `_order_meta` only on the first line item
   - Ensure each output item contains:
     - `index`
     - `partner_id`
     - `sku`
     - `title`
     - `variant_title`
     - `quantity`
     - `price_unit`
     - `total_discount`
     - `shopify_product_id`
     - `shopify_variant_id`
     - optional `_order_meta`

14. **Connect the false output of `Customer Exists?1` to `Split Line Items`**
   - This path is used when customer already exists.

15. **Connect `Create Contact1` to `Split Line Items`**
   - This path is used when a new customer was created.

16. **Add an Odoo node to search products**
   - Node type: `Odoo`
   - Name: `Search Product by Barcode1`
   - Resource: `Custom`
   - Custom Resource: `product.product`
   - Operation: `Get All`
   - Limit: `1`
   - Enable `Always Output Data`
   - Fields list:
     - `id`
     - `name`
     - `default_code`
     - `barcode`
     - `uom_id`
   - Filter:
     - Field Name: `barcode`
     - Value: `={{ $json.sku }}`
   - Note: this workflow assumes Shopify SKU equals Odoo barcode.

17. **Connect `Split Line Items` to `Search Product by Barcode1`**

18. **Add a Code node to process product matches**
   - Node type: `Code`
   - Name: `Process Product Result1`
   - Configure logic to:
     - collect all product search results with `$input.all()`
     - collect original line items with `$('Split Line Items').all()`
     - build a barcode-to-product map
     - for each line item:
       - set `product_id` if found
       - build `display_name`
       - compute Odoo discount percent from Shopify `total_discount`
   - Expected output fields per item:
     - `product_id`
     - `product_name`
     - `display_name`
     - `quantity`
     - `price_unit`
     - `discount`
     - `sku`
     - `partner_id`
     - `_order_meta`

19. **Connect `Search Product by Barcode1` to `Process Product Result1`**

20. **Add a Code node to aggregate products**
   - Node type: `Code`
   - Name: `Aggregate Products1`
   - Configure logic to:
     - read all processed products
     - separate matched and unmatched items
     - build Odoo `order_lines` arrays using `[0, 0, {...}]`
     - set empty taxes with `tax_id: [[6, 0, []]]`
     - convert unmatched products into manual lines prefixed with `[NOT FOUND]`
   - Output fields:
     - `partner_id`
     - `order_lines`
     - `products_found`
     - `products_not_found`
     - `products_total`
     - `order_meta`

21. **Connect `Process Product Result1` to `Aggregate Products1`**

22. **Add a Code node to append shipping and finalize payload**
   - Node type: `Code`
   - Name: `Add Shipping to Order1`
   - Configure logic to:
     - read `shipping_cost` from `order_meta`
     - if shipping cost > 0, add an order line using a shipping service product
     - use shipping `product_id: 16`
     - create note text with Shopify order reference, payment, shipping method, and customer note
     - output:
       - `partner_id`
       - `order_line`
       - `client_order_ref`
       - `origin`
       - `note`
       - `source_id`
       - match counters
   - Replace `product_id: 16` with your actual shipping service product in Odoo.
   - The current line name is Arabic: `مصاريف الشحن`.

23. **Connect `Aggregate Products1` to `Add Shipping to Order1`**

24. **Add an Odoo node to create the sales order**
   - Node type: `Odoo`
   - Name: `Create Sales Order in Odoo1`
   - Resource: `Custom`
   - Custom Resource: `sale.order`
   - Configure fields:
     - `partner_id` = `={{ $json.partner_id }}`
     - `client_order_ref` = `={{ $json.client_order_ref }}`
     - `origin` = `={{ $json.origin }}`
     - `note` = `={{ $json.note }}`
     - `source_id` = `17`
     - `order_line` = `={{ $('Add Shipping to Order1').first().json.order_line }}`
     - `pricelist_id` = `9`
     - `user_id` = `12`
   - Replace hardcoded IDs to fit your Odoo database:
     - `source_id`
     - `pricelist_id`
     - `user_id`

25. **Connect `Add Shipping to Order1` to `Create Sales Order in Odoo1`**

26. **Add Odoo credentials**
   - In n8n, create/connect credentials for the Odoo node.
   - Ensure the Odoo user has permission to:
     - read `res.partner`
     - create `res.partner`
     - read `res.country.state`
     - read `product.product`
     - create `sale.order`

27. **Configure Shopify webhook**
   - In Shopify Admin, add a webhook for order creation.
   - Point it to the production URL of `Shopify Webhook`.
   - Use `POST`.
   - Ensure the payload format matches standard Shopify order JSON.

28. **Optionally add the same sticky notes for maintainability**
   - General setup note
   - Receive/parse block note
   - Customer lookup/creation note
   - Product matching/build note
   - Finalization note

29. **Test with a Shopify sample order**
   - Verify:
     - customer lookup succeeds or contact creation works
     - province name matches an Odoo state
     - SKU maps to product barcode
     - shipping service product ID is valid
     - sales order appears in draft state in Odoo

30. **Harden before production**
   - Recommended enhancements:
     - deduplicate orders using Shopify order ID
     - normalize phone formatting before searching customers
     - handle empty line item arrays safely
     - use product SKU/default code fallback if barcode search fails
     - add explicit error workflow or notifications

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow automatically syncs new Shopify orders to Odoo as Sales Orders. It parses the order payload, checks if the customer exists by phone number, creates them if missing, matches products via SKU/barcode, applies shipping costs, and generates a draft Sales Order in Odoo. | General workflow behavior |
| Add the Webhook URL to Shopify Admin > Notifications > Webhooks for order creation. | Shopify configuration |
| Connect your Odoo account in n8n before testing. | Credentials setup |
| Update `product_id: 16` in the shipping code node so it matches your Odoo shipping service product ID. | Odoo shipping product dependency |
| Adjust `pricelist_id`, `user_id`, and `source_id` in the sales order creation node to match your Odoo system. | Odoo instance-specific configuration |
| The workflow uses Arabic defaults in places, including contact language `ar_001` and shipping line label `مصاريف الشحن`. | Localization behavior |
| Product matching assumes Shopify SKU corresponds to Odoo `barcode`, not `default_code`. | Product mapping rule |
| Customer lookup assumes Odoo supports searching with `phone_mobile_search`. | Odoo model/customization dependency |
| There is no built-in deduplication against repeated Shopify webhook deliveries. | Operational consideration |