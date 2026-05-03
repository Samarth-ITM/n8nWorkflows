Sync Shopify products to Odoo in real time for create and update events

https://n8nworkflows.xyz/workflows/sync-shopify-products-to-odoo-in-real-time-for-create-and-update-events-14899


# Sync Shopify products to Odoo in real time for create and update events

Now I have all the information I need. Let me write the comprehensive document.---

### 1. Workflow Overview

This workflow performs a **real-time, bidirectional product synchronization from Shopify to Odoo**, reacting to two Shopify webhook events: **product creation** and **product update**. It ensures that every new Shopify product is created in Odoo (with category mapping and image), and every updated Shopify product has its title and price reflected in Odoo. Duplicate creation is prevented by checking for existing products by barcode before inserting.

**Logical Blocks:**

| Block | Purpose |
|---|---|
| **1.1 Entry Points** | Two Shopify webhook triggers—one for `products/create`, one for `products/update`—receive real-time payloads from Shopify. |
| **1.2 Product Creation Flow** | On product creation, searches Odoo for an existing product by barcode. If not found, resolves the Odoo product category from the Shopify vendor name, downloads and converts the product image to Base64, and creates the full product record in Odoo. |
| **1.3 Product Update Flow** | On product update, looks up the Odoo product by barcode and updates only the `name` (title) and `list_price` fields. |

---

### 2. Block-by-Block Analysis

---

#### Block 1.1 — Entry Points

**Overview:** Two independent Shopify webhook triggers serve as the workflow's entry points. Each listens for a specific Shopify event and forwards the full product payload to its respective processing block.

**Nodes Involved:** `When Product Created`, `When Product Updated`

---

**Node: When Product Created**

- **Type:** `shopifyTrigger` (n8n-nodes-base.shopifyTrigger), version 1
- **Technical Role:** Webhook entry point that fires when a new product is created in Shopify.
- **Configuration:**
  - Topic: `products/create`
  - Authentication: OAuth2
- **Credential:** `shopifyOAuth2Api` — named "Shopify account" (ID: `n58kGVwa5UyR3czy`)
- **Output:** Full Shopify product JSON (including `title`, `variants[0].barcode`, `variants[0].price`, `vendor`, `images[0].src`, etc.)
- **Connections:** Output → `Search Product by Barcode`
- **Edge Cases / Failure Types:**
  - OAuth2 token expiration or revocation → webhook delivery fails silently until re-authenticated
  - Shopify webhook not registered → no events received
  - Product payload missing `variants` array → downstream expressions will fail
  - Webhook ID: `92969dc1-48f5-4656-9aff-b803b17ecf89` (must be unique in the n8n instance)

---

**Node: When Product Updated**

- **Type:** `shopifyTrigger` (n8n-nodes-base.shopifyTrigger), version 1
- **Technical Role:** Webhook entry point that fires when an existing product is updated in Shopify.
- **Configuration:**
  - Topic: `products/update`
  - Authentication: OAuth2
- **Credential:** `shopifyOAuth2Api` — named "Shopify account" (same as above)
- **Output:** Full Shopify product JSON (same schema as create, reflecting updated values)
- **Connections:** Output → `Find Product to Update`
- **Edge Cases / Failure Types:**
  - Same OAuth2 risks as the create trigger
  - Webhook ID: `4b5096b5-bcf8-492c-8812-679aafd31bb1`
  - If the update payload doesn't include barcode in `variants[0]`, the Odoo lookup will fail

---

#### Block 1.2 — Product Creation Flow

**Overview:** When a new product is created in Shopify, this block checks whether the product already exists in Odoo (by barcode). If not found, it resolves the Odoo product category by matching the Shopify vendor name, downloads the product's first image, converts it to a Base64 string, and creates the product in Odoo with all mapped fields.

**Nodes Involved:** `Search Product by Barcode`, `Process Search Result`, `If Product Not Found`, `Get Product Category`, `Download Product Image`, `Convert Image to Base64`, `Create Product in Odoo`

---

**Node: Search Product by Barcode**

- **Type:** `odoo` (n8n-nodes-base.odoo), version 1
- **Technical Role:** Queries Odoo's `product.template` model for an existing product matching the barcode from the Shopify variant.
- **Configuration:**
  - Resource: `custom`
  - Custom Resource: `product.template`
  - Operation: `getAll`
  - Limit: 1
  - Filter: `barcode` equals `{{ $json.variants[0].barcode }}`
  - `alwaysOutputData`: **true** — ensures an empty item is emitted even if no product is found, which is critical for the downstream If node logic
- **Credential:** `odooApi` — named "odoo-v2.xqpharma" (ID: `126UpuVlrIIBfuf2`)
- **Input:** From `When Product Created`
- **Output:** Either a matching Odoo product record (with `id` populated) or an empty item (no `id`)
- **Connections:** Output → `Process Search Result`
- **Edge Cases / Failure Types:**
  - Barcode is null, empty, or undefined in Shopify → filter may return all products or none, depending on Odoo behavior
  - Odoo API unreachable or credential expired → node error, workflow stops
  - Multiple products with same barcode → only first returned (limit=1)

---

**Node: Process Search Result**

- **Type:** `splitInBatches` (n8n-nodes-base.splitInBatches), version 3
- **Technical Role:** Acts as a passthrough/batch-finalization step. In this workflow it is used to ensure all items from the Odoo search are processed before moving to the conditional check. Output 0 (per-batch) has no connections; output 1 (done) forwards items to the If node.
- **Configuration:**
  - No special parameters; default batch size
- **Input:** From `Search Product by Barcode`
- **Output 1 (Done):** → `If Product Not Found`
- **Edge Cases / Failure Types:**
  - No items at all → done output fires immediately with empty payload
  - This node is essentially a structural bridge; if removed, the If node could be connected directly, but the pattern ensures batch completeness

---

**Node: If Product Not Found**

- **Type:** `if` (n8n-nodes-base.if), version 2.3
- **Technical Role:** Conditional gate that determines whether the product already exists in Odoo. If `id` is present (product found), the workflow does nothing further (true branch is unconnected). If `id` is empty/missing (product not found), the creation pipeline continues.
- **Configuration:**
  - Condition combinator: `and`
  - Condition: `$json.id` is **not empty** (string operation: `notEmpty`)
  - `looseTypeValidation`: **true**
  - `alwaysOutputData`: **true**
- **Key Expression:** `{{ $json.id }}`
- **Input:** From `Process Search Result` (done output)
- **Output 0 (True — product exists):** *No connection* (intentionally does nothing; product already synced)
- **Output 1 (False — product not found):** → `Get Product Category`
- **Edge Cases / Failure Types:**
  - If Odoo returned an empty item (no `id` field at all), `notEmpty` evaluates to false → creation proceeds, which is the desired behavior
  - If Odoo returned a product with `id: 0` or `id: ""`, loose type validation may treat it differently; test empirically

---

**Node: Get Product Category**

- **Type:** `odoo` (n8n-nodes-base.odoo), version 1
- **Technical Role:** Looks up the Odoo `product.category` record whose name matches the Shopify product's `vendor` field. The returned category `id` is used when creating the product.
- **Configuration:**
  - Resource: `custom`
  - Custom Resource: `product.category`
  - Operation: `getAll`
  - Limit: 1
  - Filter: `name` equals `{{ $('When Product Created').item.json.vendor }}`
  - `alwaysOutputData`: **true**
- **Credential:** `odooApi` — named "odoo-v2.xqpharma" (same credential)
- **Key Expression:** `{{ $('When Product Created').item.json.vendor }}` — references the original Shopify trigger data
- **Input:** From `If Product Not Found` (false branch)
- **Output:** Odoo category record (with `id`)
- **Connections:** Output → `Download Product Image`
- **Edge Cases / Failure Types:**
  - Shopify `vendor` field is empty or doesn't match any Odoo category → `alwaysOutputData` returns empty item; `$('Get Product Category').item.json.id` will be undefined in the next nodes, potentially causing Odoo to reject the product creation
  - Multiple categories with same name → only first returned
  - Category name case sensitivity mismatch between Shopify vendor and Odoo category name

---

**Node: Download Product Image**

- **Type:** `httpRequest` (n8n-nodes-base.httpRequest), version 4.3
- **Technical Role:** Downloads the first product image from Shopify as a binary file.
- **Configuration:**
  - URL: `{{ $('When Product Created').item.json.images[0].src }}`
  - Response format: **File** (binary data)
- **Key Expression:** References Shopify trigger's `images[0].src`
- **Input:** From `Get Product Category`
- **Output:** Binary data (image file) attached to the item
- **Connections:** Output → `Convert Image to Base64`
- **Edge Cases / Failure Types:**
  - Product has no images (`images` array is empty or missing) → `images[0].src` throws an expression error, workflow stops
  - Image URL is temporary or expired → HTTP 404/403
  - Large image file → potential memory/timeout issues
  - Network timeout on download

---

**Node: Convert Image to Base64**

- **Type:** `extractFromFile` (n8n-nodes-base.extractFromFile), version 1.1
- **Technical Role:** Converts the downloaded binary image into a Base64-encoded string and stores it in a JSON property named `final_image`.
- **Configuration:**
  - Operation: `binaryToPropery`
  - Destination Key: `final_image`
- **Input:** From `Download Product Image` (binary attachment)
- **Output:** JSON item with `final_image` property containing Base64 string
- **Connections:** Output → `Create Product in Odoo`
- **Edge Cases / Failure Types:**
  - No binary data present (upstream download failed silently) → conversion fails
  - Very large image → extremely long Base64 string may cause Odoo API payload size issues

---

**Node: Create Product in Odoo**

- **Type:** `odoo` (n8n-nodes-base.odoo), version 1
- **Technical Role:** Creates a new `product.template` record in Odoo with all mapped fields from Shopify.
- **Configuration:**
  - Resource: `custom`
  - Custom Resource: `product.template`
  - Operation: `create`
  - Fields:
    | Field | Value Expression |
    |---|---|
    | `name` | `{{ $('When Product Created').item.json.title }}` |
    | `list_price` | `{{ $('When Product Created').item.json.variants[0].price }}` |
    | `barcode` | `{{ $('When Product Created').item.json.variants[0].barcode }}` |
    | `taxes_id` | `{{ [] }}` (empty array — no taxes) |
    | `supplier_taxes_id` | `{{ [] }}` (empty array — no supplier taxes) |
    | `categ_id` | `{{ $('Get Product Category').item.json.id }}` |
    | `image_1920` | `{{ $json.final_image }}` (Base64 from previous node) |
- **Credential:** `odooApi` — named "odoo-v2.xqpharma"
- **Input:** From `Convert Image to Base64`
- **Output:** Created Odoo product record (with generated `id`)
- **Connections:** No further connections (end of creation flow)
- **Edge Cases / Failure Types:**
  - Missing `categ_id` (category not found) → Odoo may reject or use default category
  - Invalid Base64 in `image_1920` → Odoo may silently ignore or error
  - `taxes_id` / `supplier_taxes_id` set to empty array `[]` — verify Odoo accepts this format for Many2many fields; some Odoo versions require `[[6, False, []]]` syntax
  - Duplicate barcode (race condition if two webhooks fire simultaneously) → Odoo may create duplicates since the check was done earlier
  - Price as string vs float — Odoo expects numeric; Shopify sends string → verify Odoo coercion

---

#### Block 1.3 — Product Update Flow

**Overview:** When an existing product is updated in Shopify, this block finds the corresponding Odoo product by barcode and updates its title and price to match Shopify.

**Nodes Involved:** `Find Product to Update`, `Update Product in Odoo`

---

**Node: Find Product to Update**

- **Type:** `odoo` (n8n-nodes-base.odoo), version 1
- **Technical Role:** Searches Odoo for the existing product template matching the barcode from the updated Shopify variant.
- **Configuration:**
  - Resource: `custom`
  - Custom Resource: `product.template`
  - Operation: `getAll`
  - Limit: 1
  - Filter: `barcode` equals `{{ $json.variants[0].barcode }}`
  - `alwaysOutputData`: **true**
- **Credential:** `odooApi` — named "odoo-v2.xqpharma"
- **Input:** From `When Product Updated`
- **Output:** Odoo product record (with `id`) or empty item if not found
- **Connections:** Output → `Update Product in Odoo`
- **Edge Cases / Failure Types:**
  - Product not found in Odoo (not yet synced) → empty item, `id` undefined; the Update node will fail because `customResourceId` will be invalid
  - Barcode missing or null → lookup may return wrong results or nothing
  - Multiple products sharing same barcode → only first is updated

---

**Node: Update Product in Odoo**

- **Type:** `odoo` (n8n-nodes-base.odoo), version 1
- **Technical Role:** Updates the Odoo product's name and list price to match Shopify.
- **Configuration:**
  - Resource: `custom`
  - Custom Resource: `product.template`
  - Operation: `update`
  - Resource ID: `{{ $json.id }}` (the Odoo product ID from the lookup)
  - Fields:
    | Field | Value Expression |
    |---|---|
    | `name` | `{{ $('When Product Updated').item.json.title }}` |
    | `list_price` | `{{ $('When Product Updated').item.json.variants[0].price }}` |
- **Credential:** `odooApi` — named "odoo-v2.xqpharma"
- **Input:** From `Find Product to Update`
- **Output:** Updated Odoo product record
- **Connections:** No further connections (end of update flow)
- **Edge Cases / Failure Types:**
  - `$json.id` is undefined (product not found in Odoo) → Odoo API error (invalid ID)
  - No validation that the product exists before attempting update — consider adding an If guard similar to the creation flow
  - Price format mismatch (string vs float)
  - Only `name` and `list_price` are updated; other changes (barcode, image, category) are ignored

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note - Overview | stickyNote | Visual documentation — describes overall workflow logic and setup steps | — | — | ## Shopify to Odoo Sync / How it works: 1. Listens for new products in Shopify, downloads images, and creates them in Odoo. 2. Listens for product updates in Shopify and syncs changes to Odoo. / Setup steps: Configure Shopify Webhooks, Configure Odoo API Credentials, Verify Category mapping |
| Sticky Note - Create Flow | stickyNote | Visual documentation — describes the creation flow | — | — | ## 1. Product Creation Flow / Checks if the product exists in Odoo by barcode. If not found, fetches the category, downloads the image, and creates the new product. |
| Sticky Note - Update Flow | stickyNote | Visual documentation — describes the update flow | — | — | ## 2. Product Update Flow / Finds the existing product in Odoo by barcode and syncs the updated title and price. |
| When Product Created | shopifyTrigger | Entry point — fires on Shopify products/create webhook | — | Search Product by Barcode | ## Shopify to Odoo Sync / How it works: 1. Listens for new products in Shopify, downloads images, and creates them in Odoo. 2. Listens for product updates in Shopify and syncs changes to Odoo. / Setup steps: Configure Shopify Webhooks, Configure Odoo API Credentials, Verify Category mapping; ## 1. Product Creation Flow / Checks if the product exists in Odoo by barcode. If not found, fetches the category, downloads the image, and creates the new product. |
| Search Product by Barcode | odoo | Searches Odoo product.template by barcode to check for duplicates | When Product Created | Process Search Result | ## 1. Product Creation Flow / Checks if the product exists in Odoo by barcode. If not found, fetches the category, downloads the image, and creates the new product. |
| Process Search Result | splitInBatches | Passthrough — ensures batch completion before conditional check | Search Product by Barcode | If Product Not Found | ## 1. Product Creation Flow / Checks if the product exists in Odoo by barcode. If not found, fetches the category, downloads the image, and creates the new product. |
| If Product Not Found | if | Conditional gate — routes to creation only if product not already in Odoo | Process Search Result | Get Product Category (false branch) | ## 1. Product Creation Flow / Checks if the product exists in Odoo by barcode. If not found, fetches the category, downloads the image, and creates the new product. |
| Get Product Category | odoo | Resolves Odoo product category by matching Shopify vendor name | If Product Not Found | Download Product Image | ## 1. Product Creation Flow / Checks if the product exists in Odoo by barcode. If not found, fetches the category, downloads the image, and creates the new product. |
| Download Product Image | httpRequest | Downloads the first Shopify product image as binary file | Get Product Category | Convert Image to Base64 | ## 1. Product Creation Flow / Checks if the product exists in Odoo by barcode. If not found, fetches the category, downloads the image, and creates the new product. |
| Convert Image to Base64 | extractFromFile | Converts binary image to Base64 string property | Download Product Image | Create Product in Odoo | ## 1. Product Creation Flow / Checks if the product exists in Odoo by barcode. If not found, fetches the category, downloads the image, and creates the new product. |
| Create Product in Odoo | odoo | Creates new product.template in Odoo with all mapped fields | Convert Image to Base64 | — | ## 1. Product Creation Flow / Checks if the product exists in Odoo by barcode. If not found, fetches the category, downloads the image, and creates the new product. |
| When Product Updated | shopifyTrigger | Entry point — fires on Shopify products/update webhook | — | Find Product to Update | ## Shopify to Odoo Sync / How it works: 1. Listens for new products in Shopify, downloads images, and creates them in Odoo. 2. Listens for product updates in Shopify and syncs changes to Odoo. / Setup steps: Configure Shopify Webhooks, Configure Odoo API Credentials, Verify Category mapping; ## 2. Product Update Flow / Finds the existing product in Odoo by barcode and syncs the updated title and price. |
| Find Product to Update | odoo | Searches Odoo product.template by barcode for the updated product | When Product Updated | Update Product in Odoo | ## 2. Product Update Flow / Finds the existing product in Odoo by barcode and syncs the updated title and price. |
| Update Product in Odoo | odoo | Updates Odoo product name and list_price from Shopify | Find Product to Update | — | ## 2. Product Update Flow / Finds the existing product in Odoo by barcode and syncs the updated title and price. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it "Sync Shopify products to Odoo in real time for create and update events".

2. **Add a Sticky Note — Overview:**
   - Type: `Sticky Note`
   - Color: 6 (blue)
   - Width: 300, Height: 300
   - Content:
     ```
     ## Shopify to Odoo Sync

     ### How it works
     1. Listens for new products in Shopify, downloads images, and creates them in Odoo.
     2. Listens for product updates in Shopify and syncs changes to Odoo.

     ### Setup steps
     - [x] Configure Shopify Webhooks
     - [x] Configure Odoo API Credentials
     - [x] Verify Category mapping
     ```

3. **Add a Sticky Note — Create Flow:**
   - Type: `Sticky Note`
   - Color: 7 (green)
   - Width: 2050, Height: 220
   - Content:
     ```
     ## 1. Product Creation Flow
     Checks if the product exists in Odoo by barcode. If not found, fetches the category, downloads the image, and creates the new product.
     ```

4. **Add a Sticky Note — Update Flow:**
   - Type: `Sticky Note`
   - Color: 7 (green)
   - Width: 800, Height: 200
   - Content:
     ```
     ## 2. Product Update Flow
     Finds the existing product in Odoo by barcode and syncs the updated title and price.
     ```

5. **Create node: When Product Created**
   - Type: `Shopify Trigger`
   - Topic: `products/create`
   - Authentication: `OAuth2`
   - Credential: Create or select a `Shopify OAuth2 API` credential (named "Shopify account"). This requires a Shopify app with read_products scope and the OAuth2 callback configured in n8n.
   - Position this node within the Create Flow sticky note area.

6. **Create node: Search Product by Barcode**
   - Type: `Odoo`
   - Credential: Create or select an `Odoo API` credential (named "odoo-v2.xqpharma"). Requires Odoo URL, database name, username, and password.
   - Resource: `custom`
   - Custom Resource: `product.template`
   - Operation: `getAll`
   - Limit: 1
   - Filter: Field `barcode` equals `{{ $json.variants[0].barcode }}`
   - Enable `alwaysOutputData`: **true**
   - Connect: `When Product Created` → `Search Product by Barcode`

7. **Create node: Process Search Result**
   - Type: `Split In Batches`
   - Version: 3
   - Default options (no special configuration)
   - Connect: `Search Product by Barcode` → `Process Search Result` (input → output 0)
   - Connect: `Process Search Result` output 1 (Done) → `If Product Not Found`

8. **Create node: If Product Not Found**
   - Type: `If`
   - Version: 2.3
   - Condition combinator: `and`
   - Condition: `{{ $json.id }}` — operator: **is not empty** (string, notEmpty)
   - Enable `looseTypeValidation`: **true**
   - Enable `alwaysOutputData`: **true**
   - Connect: Output 1 (False — product NOT found) → `Get Product Category`
   - Output 0 (True) remains **unconnected** intentionally

9. **Create node: Get Product Category**
   - Type: `Odoo`
   - Credential: Same Odoo API credential as step 6
   - Resource: `custom`
   - Custom Resource: `product.category`
   - Operation: `getAll`
   - Limit: 1
   - Filter: Field `name` equals `{{ $('When Product Created').item.json.vendor }}`
   - Enable `alwaysOutputData`: **true**
   - Connect: Output → `Download Product Image`

10. **Create node: Download Product Image**
    - Type: `HTTP Request`
    - Version: 4.3
    - URL: `{{ $('When Product Created').item.json.images[0].src }}`
    - Response Format: **File** (under Options → Response → Response Format)
    - Enable `alwaysOutputData`: **true**
    - Connect: Output → `Convert Image to Base64`

11. **Create node: Convert Image to Base64**
    - Type: `Extract From File`
    - Version: 1.1
    - Operation: `binaryToPropery`
    - Destination Key: `final_image`
    - Connect: Output → `Create Product in Odoo`

12. **Create node: Create Product in Odoo**
    - Type: `Odoo`
    - Credential: Same Odoo API credential
    - Resource: `custom`
    - Custom Resource: `product.template`
    - Operation: `create`
    - Fields to create/update:
      - `name` = `{{ $('When Product Created').item.json.title }}`
      - `list_price` = `{{ $('When Product Created').item.json.variants[0].price }}`
      - `barcode` = `{{ $('When Product Created').item.json.variants[0].barcode }}`
      - `taxes_id` = `{{ [] }}`
      - `supplier_taxes_id` = `{{ [] }}`
      - `categ_id` = `{{ $('Get Product Category').item.json.id }}`
      - `image_1920` = `{{ $json.final_image }}`
    - No further connections needed (end of creation flow)

13. **Create node: When Product Updated**
    - Type: `Shopify Trigger`
    - Topic: `products/update`
    - Authentication: `OAuth2`
    - Credential: Same Shopify OAuth2 API credential as step 5
    - Position this node within the Update Flow sticky note area.

14. **Create node: Find Product to Update**
    - Type: `Odoo`
    - Credential: Same Odoo API credential
    - Resource: `custom`
    - Custom Resource: `product.template`
    - Operation: `getAll`
    - Limit: 1
    - Filter: Field `barcode` equals `{{ $json.variants[0].barcode }}`
    - Enable `alwaysOutputData`: **true**
    - Connect: `When Product Updated` → `Find Product to Update`

15. **Create node: Update Product in Odoo**
    - Type: `Odoo`
    - Credential: Same Odoo API credential
    - Resource: `custom`
    - Custom Resource: `product.template`
    - Operation: `update`
    - Resource ID: `{{ $json.id }}`
    - Fields to update:
      - `name` = `{{ $('When Product Updated').item.json.title }}`
      - `list_price` = `{{ $('When Product Updated').item.json.variants[0].price }}`
    - Connect: `Find Product to Update` → `Update Product in Odoo`

16. **Activate the workflow** — both Shopify webhooks will be registered automatically when the workflow is activated.

17. **Post-activation verification:**
    - Confirm in Shopify Admin → Settings → Notifications that both webhooks (`products/create` and `products/update`) are registered pointing to your n8n instance.
    - Create a test product in Shopify and verify it appears in Odoo.
    - Update the test product's title/price in Shopify and verify the changes sync to Odoo.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The barcode field on `variants[0]` is the primary key for matching Shopify products to Odoo products. Ensure every Shopify product has a unique, non-empty barcode assigned. | Applies to both creation and update flows |
| The `vendor` field in Shopify is mapped to `product.category.name` in Odoo. Category names must match exactly (case-sensitive) for the lookup to succeed. | Get Product Category node |
| `taxes_id` and `supplier_taxes_id` are set to empty arrays `[]`. Depending on Odoo version, Many2many fields may require the command-writer syntax `[[6, False, []]]` instead. Verify with your Odoo instance. | Create Product in Odoo node |
| Only `name` and `list_price` are synced on product updates. Changes to barcode, image, category, or other fields in Shopify will **not** be reflected in Odoo automatically. | Update Product in Odoo node |
| The update flow has no guard for "product not found in Odoo." If a Shopify product update arrives before the product has been created in Odoo, the Update node will fail. Consider adding an If node similar to the creation flow, or adding an Error Trigger for resilience. | Find Product to Update → Update Product in Odoo |
| Shopify product images are stored as temporary URLs. The workflow downloads the first image at the moment of creation. If Shopify regenerates image URLs, previously downloaded images in Odoo remain unaffected. | Download Product Image node |
| The workflow uses two separate webhook IDs for the two Shopify triggers. If the workflow is duplicated or imported into another n8n instance, the webhooks will be re-registered with new IDs. | Both Shopify Trigger nodes |
| Products with multiple variants: the workflow only uses `variants[0]` (the first variant) for barcode and price. Multi-variant products are not fully supported. | All nodes referencing `variants[0]` |