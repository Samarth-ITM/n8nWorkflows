Create e-commerce listings from images with UploadToURL, GPT-4o, Shopify

https://n8nworkflows.xyz/workflows/create-e-commerce-listings-from-images-with-uploadtourl--gpt-4o--shopify-13541


# Create e-commerce listings from images with UploadToURL, GPT-4o, Shopify

## 1. Workflow Overview

**Purpose:**  
This workflow receives a product image (either as a remote `fileUrl` or as a binary upload) plus basic metadata (SKU, price, platform), hosts the image via **UploadToURL**, generates e-commerce copy using **GPT-4o Vision**, then creates a product listing in **Shopify** or **WooCommerce**, optionally notifying a Slack channel and returning a structured webhook response.

**Target use cases:**
- Mobile “snap a product photo → create listing” pipeline
- Bulk/assisted catalog creation where content quality and SEO matter
- Multi-platform listing automation (Shopify/WooCommerce) with consistent outputs

### 1.1 Input Reception & Validation
Receives the webhook request and normalizes/validates fields (platform, file source, SKU, price, inventory, tags, MIME type).

### 1.2 Upload Hosting (Remote vs Binary)
Routes based on whether `fileUrl` is present and uploads the image via UploadToURL (community node), then extracts a single canonical `imageUrl`.

### 1.3 Vision AI Content Generation
Sends `imageUrl` plus user hints to GPT-4o Vision, expecting **strict JSON** back. Parses, validates, merges, and computes platform-specific status and a clean URL slug (`handle`).

### 1.4 Platform Routing & Product Creation
Switches between Shopify and WooCommerce creation branches.

### 1.5 Notification & Webhook Response
Builds a normalized response object, posts a Slack message (optional), and responds to the original webhook with HTTP 201.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception & Validation

**Overview:**  
Accepts a `POST` request, validates required inputs, normalizes and enriches product metadata, and prepares consistent fields for downstream nodes.

**Nodes involved:**
- `Webhook - Receive Product`
- `Validate & Enrich Payload`
- `Has Remote URL?`

#### Node: Webhook - Receive Product
- **Type / role:** `n8n-nodes-base.webhook` — workflow entry point receiving incoming product submissions.
- **Key configuration:**
  - **Method:** `POST`
  - **Path:** `product-catalog-upload`
  - **CORS/Origins:** `allowedOrigins: *`
  - **Response mode:** `responseNode` (workflow must end with *Respond to Webhook* node).
- **Inputs/Outputs:**
  - Output → `Validate & Enrich Payload`
- **Edge cases / failures:**
  - Requests not using POST or wrong path won’t trigger the workflow.
  - If binary file is sent, n8n must be configured to accept it (client must send multipart/form-data). The workflow logic assumes `body.filename` may be present for binary path.
- **Version notes:** node `typeVersion: 2`.

#### Node: Validate & Enrich Payload
- **Type / role:** `n8n-nodes-base.code` — validation + normalization of incoming fields.
- **What it does (interpreted):**
  - Reads request body from `$input.first().json.body` or `$input.first().json`
  - Validates `platform` ∈ `{shopify, woocommerce}` with fallback to `$vars.DEFAULT_PLATFORM` then `shopify`
  - Requires at least one of:
    - `fileUrl` (remote image)
    - `filename` (for binary upload path)
  - Derives `filename` from `fileUrl` if not provided
  - Derives `mimeType` from file extension with fallback `image/jpeg`
  - Sanitizes SKU: uppercase, strips special chars; generates fallback `SKU-<timestamp>`
  - Coerces `price` to float; errors if negative
  - Normalizes inventory to integer ≥ 0; default 1
  - Splits comma-separated tags into array
  - Passes through optional hints: `title`, `description`
- **Key variables/expressions:**
  - `$vars.DEFAULT_PLATFORM`
  - `body.fileUrl?.split('?')[0].split('/').pop()` for filename inference
- **Inputs/Outputs:**
  - Input ← `Webhook - Receive Product`
  - Output → `Has Remote URL?`
- **Edge cases / failures:**
  - Invalid platform throws error (workflow fails).
  - Missing `fileUrl` and missing `filename` throws error.
  - Negative price throws error.
  - `compareAtPrice` can become `null` if not parseable (intentional).
  - Tag parsing assumes a string; if an array/object is passed, it may throw or become empty unexpectedly.
- **Version notes:** `typeVersion: 2`.

#### Node: Has Remote URL?
- **Type / role:** `n8n-nodes-base.if` — branching based on presence of `fileUrl`.
- **Configuration (interpreted):**
  - Condition: `{{$json.fileUrl}}` **notEmpty**
- **Inputs/Outputs:**
  - Input ← `Validate & Enrich Payload`
  - **True** → `Upload to URL - Remote`
  - **False** → `Upload to URL - Binary`
- **Edge cases / failures:**
  - If `fileUrl` exists but is invalid/unreachable, upload will fail downstream.
- **Version notes:** `typeVersion: 2`.

---

### Block 2 — Upload Hosting (Remote vs Binary)

**Overview:**  
Uploads the image to UploadToURL via one of two paths (remote fetch or binary upload), then extracts a single public HTTPS URL (`imageUrl`) for AI and storefront ingestion.

**Nodes involved:**
- `Upload to URL - Remote`
- `Upload to URL - Binary`
- `Extract Image URL`

#### Node: Upload to URL - Remote
- **Type / role:** `n8n-nodes-uploadtourl.uploadToUrl` (community node) — upload operation using a remote source.
- **Configuration (interpreted):**
  - Operation: `uploadFile`
  - Credential: `Upload to URL account 3`
- **Inputs/Outputs:**
  - Input ← `Has Remote URL?` (true branch)
  - Output → `Extract Image URL`
- **Edge cases / failures:**
  - UploadToURL credential/auth failure.
  - Remote URL fetch failure (timeouts, 403/404, unsupported content-type).
  - Response schema differences (handled later in Extract Image URL).
- **Version notes:** `typeVersion: 1`
- **Requirements:** Requires installing the community node **n8n-nodes-uploadtourl**.

#### Node: Upload to URL - Binary
- **Type / role:** `n8n-nodes-uploadtourl.uploadToUrl` — upload operation using binary data received by webhook.
- **Configuration (interpreted):**
  - Operation: `uploadFile`
  - Credential: `Upload to URL account 3`
- **Inputs/Outputs:**
  - Input ← `Has Remote URL?` (false branch)
  - Output → `Extract Image URL`
- **Edge cases / failures:**
  - If the webhook request did not actually include binary data, upload will fail.
  - Size limits (n8n, reverse proxy, UploadToURL) may reject large images.
- **Version notes:** `typeVersion: 1`

#### Node: Extract Image URL
- **Type / role:** `n8n-nodes-base.code` — normalizes UploadToURL responses into a consistent `imageUrl`.
- **What it does (interpreted):**
  - Reads upload response from `$input.first().json`
  - Pulls original enriched metadata from `$('Validate & Enrich Payload').first().json`
  - Attempts to find image URL in multiple fields: `url`, `link`, `data.url`, `file.url`, `shortUrl`
  - Forces HTTPS: `imageUrl.replace(/^http:\/\//, 'https://')`
  - Outputs merged object with `imageUrl`, `uploadId`, `fileSizeBytes`
- **Inputs/Outputs:**
  - Input ← `Upload to URL - Remote` or `Upload to URL - Binary`
  - Output → `GPT-4o Vision - Analyse Product`
- **Edge cases / failures:**
  - If UploadToURL changes response format and none of the checked fields exist, node throws with raw response included.
  - If `Validate & Enrich Payload` didn’t run (shouldn’t happen in this graph), the node would fail due to missing meta.
- **Version notes:** `typeVersion: 2`.

---

### Block 3 — Vision AI Content Generation

**Overview:**  
Calls GPT-4o Vision with the hosted image URL and returns structured JSON containing title, descriptions, tags, SEO fields, and confidence, then validates and merges results with original metadata.

**Nodes involved:**
- `GPT-4o Vision - Analyse Product`
- `Parse & Merge AI Product Data`

#### Node: GPT-4o Vision - Analyse Product
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi` — OpenAI chat model invocation with vision-capable prompt (expects JSON-only output).
- **Configuration (interpreted):**
  - Model: selected via `modelId` (currently blank in JSON; must be set in UI to `gpt-4o` or equivalent vision model)
  - Temperature: `0.6`
  - Max tokens: `1200`
  - Messages:
    - **System:** instructs “return ONLY valid JSON object—no markdown”
    - **User:** includes:
      - `Image URL: {{ $json.imageUrl }}`
      - user hints: `userTitle`, `userDescription`
      - vendor/productType/tags/price context
      - strict JSON schema to return (title, seoTitle, description HTML, shortDescription, bulletFeatures[], suggestedCategory, suggestedTags[], metaDescription, handle, marketingBlurb, confidenceScore)
- **Inputs/Outputs:**
  - Input ← `Extract Image URL`
  - Output → `Parse & Merge AI Product Data`
- **Edge cases / failures:**
  - If model returns non-JSON (despite instruction), parsing will fail downstream.
  - If the model refuses/returns partial fields, downstream validations may throw (e.g., missing `title`).
  - Token limits: overly verbose descriptions may be truncated.
  - Requires valid OpenAI credentials configured in n8n.
- **Version notes:** `typeVersion: 1.5` (ensure your n8n supports this langchain OpenAI node version).

#### Node: Parse & Merge AI Product Data
- **Type / role:** `n8n-nodes-base.code` — parses GPT output, merges tags, computes handle/status fields for storefront.
- **What it does (interpreted):**
  - Reads AI response from several possible locations:
    - `aiRaw.message.content`
    - `aiRaw.choices[0].message.content`
    - `aiRaw.content` / `aiRaw.text`
  - `JSON.parse()` if string
  - Requires `ai.title`
  - Merges tags:
    - `productData.tags` + `ai.suggestedTags`
    - deduplicates and caps at 15
  - Computes `handle` slug from `ai.handle || ai.title`:
    - lowercase, remove non-alphanum/space/hyphen, spaces → hyphens, collapse hyphens
  - Computes publish status:
    - Shopify: `active` or `draft`
    - WooCommerce: `publish` or `draft`
  - Outputs a single normalized product object used by platform nodes.
- **Inputs/Outputs:**
  - Input ← `GPT-4o Vision - Analyse Product`
  - Output → `Route by Platform`
- **Edge cases / failures:**
  - JSON parsing errors if GPT output contains trailing commas, markdown, or commentary.
  - `handle` can become empty if title is only symbols; would cause storefront creation issues.
  - Tag limit assumption (15) is enforced; if platform supports more, you may be truncating unnecessarily.
- **Version notes:** `typeVersion: 2`.

---

### Block 4 — Platform Routing & Product Creation

**Overview:**  
Routes by `platform` and creates a product in Shopify or WooCommerce using the AI-generated fields plus original metadata.

**Nodes involved:**
- `Route by Platform`
- `Shopify - Create Product`
- `WooCommerce - Create Product`

#### Node: Route by Platform
- **Type / role:** `n8n-nodes-base.switch` — routes to Shopify vs WooCommerce branch.
- **Configuration (interpreted):**
  - Rule 1 output renamed to **Shopify** when `{{$json.platform}} == "shopify"`
  - Rule 2 output renamed to **WooCommerce** when `{{$json.platform}} == "woocommerce"`
  - Fallback output: `extra` (unused in connections; would silently drop unless connected)
- **Inputs/Outputs:**
  - Input ← `Parse & Merge AI Product Data`
  - Output (Shopify) → `Shopify - Create Product`
  - Output (WooCommerce) → `WooCommerce - Create Product`
- **Edge cases / failures:**
  - If `platform` is something else (should be prevented earlier), item goes to fallback `extra` which is not connected → item effectively lost (no webhook response).
- **Version notes:** `typeVersion: 3`.

#### Node: Shopify - Create Product
- **Type / role:** `n8n-nodes-base.shopify` — creates a product in Shopify.
- **Configuration (as present in JSON):**
  - Resource: `product`
  - Authentication: `oAuth2`
  - Title: `={{ $json.title }}`
  - `additionalFields` is empty in JSON (despite sticky note claiming more fields are set).
- **Inputs/Outputs:**
  - Input ← `Route by Platform` (Shopify output)
  - Output → `Build Product Response`
- **Edge cases / failures:**
  - As configured, it likely creates a product with **title only** unless additionalFields/variants/images are configured in the UI. This can cause:
    - Missing images (workflow expects image in response later)
    - Missing variants/pricing/inventory
  - OAuth2 credential issues or insufficient Shopify scopes (write_products).
- **Version notes:** `typeVersion: 1`.
- **Important discrepancy (sticky note vs node config):**
  - Sticky note says images/status/metafields/variant pricing/inventory are set. The exported parameters do not show these mappings. If you rely on these fields, you must configure them explicitly in this node.

#### Node: WooCommerce - Create Product
- **Type / role:** `n8n-nodes-base.httpRequest` — direct REST call to WooCommerce `/products`.
- **Configuration (interpreted):**
  - URL: `{{$credentials.url}}/wp-json/wc/v3/products`
  - Method: POST
  - Auth: Generic credential type → HTTP Basic Auth (`Hong Sau Resorts - Live`)
  - Response format: JSON
  - JSON body includes:
    - `name`, `slug`, `status` from computed fields
    - `description` (HTML), `short_description` (plain)
    - `sku`, `regular_price`
    - `sale_price`: set to price when `compareAtPrice` exists, else empty string  
      **Note:** This is unusual (normally sale_price < regular_price and regular_price might be compare-at). Current logic may not reflect intended sale semantics.
    - stock management, weight
    - categories by name: `{ "name": category }` (WooCommerce will create or match)
    - tags array from `tags`
    - image src: `imageUrl`
    - `meta_data` includes Yoast title/metadesc and marketing blurb
- **Inputs/Outputs:**
  - Input ← `Route by Platform` (WooCommerce output)
  - Output → `Build Product Response`
- **Edge cases / failures:**
  - Basic Auth may be blocked unless using HTTPS and supported WP configuration.
  - `{{$credentials.url}}` must exist in the credential; otherwise the URL expression fails.
  - Category/tag creation by name can create duplicates if naming varies slightly.
  - Weight must be a string in WooCommerce; workflow passes `''` if null (ok).
- **Version notes:** `typeVersion: 4.2`.

---

### Block 5 — Notification & Webhook Response

**Overview:**  
Normalizes platform responses into a single summary object, optionally posts to Slack, and returns JSON to the caller with HTTP 201.

**Nodes involved:**
- `Build Product Response`
- `Slack - Notify Team`
- `Respond to Webhook`

#### Node: Build Product Response
- **Type / role:** `n8n-nodes-base.code` — normalizes Shopify/WooCommerce responses and constructs final response payload.
- **What it does (interpreted):**
  - Detects platform from `$('Parse & Merge AI Product Data').first().json.platform`
  - Derives:
    - `productId` from Shopify `product.id` or `id`, Woo `id`
    - `productUrl`:
      - Shopify: `https://<handle>.myshopify.com/products/<handle>` (uses handle, **not your actual shop domain** unless it is `*.myshopify.com`)
      - Woo: `permalink` or `link`
    - `imageUrl` from API response else fallback to `productData.imageUrl`
  - Throws if `productId` missing
  - Outputs a consistent `success: true` object with key fields
- **Inputs/Outputs:**
  - Input ← `Shopify - Create Product` OR `WooCommerce - Create Product`
  - Output → `Slack - Notify Team`
- **Edge cases / failures:**
  - Shopify URL construction likely wrong for most real stores (custom domain). Consider using shop domain from credentials or response.
  - If Shopify node doesn’t return `product.handle` or images, URL/image fallbacks may be used.
  - Truncates error response snippet to 300 chars in thrown error.
- **Version notes:** `typeVersion: 2`.

#### Node: Slack - Notify Team
- **Type / role:** `n8n-nodes-base.slack` — sends a formatted Slack message (optional).
- **Configuration (interpreted):**
  - Auth: OAuth2 (`Mediajade Slack`)
  - Message text uses Slack formatting and expressions:
    - title, sku, price, compareAt formatting
    - platform uppercased
    - tags joined
    - confidence score percent
- **Inputs/Outputs:**
  - Input ← `Build Product Response`
  - Output → `Respond to Webhook`
- **Edge cases / failures:**
  - Slack credential/scopes missing (chat:write).
  - If tags array is missing (should not), `.join()` would fail.
  - If you disable this node, ensure the workflow still connects `Build Product Response` to `Respond to Webhook`.
- **Version notes:** `typeVersion: 2.2`.

#### Node: Respond to Webhook
- **Type / role:** `n8n-nodes-base.respondToWebhook` — returns response to initial webhook caller.
- **Configuration (interpreted):**
  - HTTP code: `201`
  - Header: `Content-Type: application/json`
  - Body: `={{ $('Build Product Response').first().json }}`
- **Inputs/Outputs:**
  - Input ← `Slack - Notify Team`
  - Final node (no outputs)
- **Edge cases / failures:**
  - If Build Product Response fails, webhook caller receives an error (depending on n8n error handling settings).
- **Version notes:** `typeVersion: 1.1`.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| 📋 Overview | n8n-nodes-base.stickyNote | Documentation / canvas annotation | — | — | 🛍️ AI Product Catalog Architect / Credentials & setup notes incl. community node and DEFAULT_PLATFORM |
| Entry & Validation | n8n-nodes-base.stickyNote | Documentation / canvas annotation | — | — | ## 🚪 Entry & Validation (describes Webhook→Validate→IF behavior) |
| Upload + Vision AI | n8n-nodes-base.stickyNote | Documentation / canvas annotation | — | — | ## ☁️ Upload to URL → Vision AI (describes upload paths + GPT + parsing) |
| Platform Routing | n8n-nodes-base.stickyNote | Documentation / canvas annotation | — | — | ## 🏪 Platform Routing & Product Creation (describes Switch + Shopify/WC mapping) |
| Notification & Response | n8n-nodes-base.stickyNote | Documentation / canvas annotation | — | — | ## 📣 Notification & Response (describes response merge + Slack + webhook response) |
| Webhook - Receive Product | n8n-nodes-base.webhook | Entry point (POST ingestion) | — | Validate & Enrich Payload | ## 🚪 Entry & Validation (describes Webhook→Validate→IF behavior) |
| Validate & Enrich Payload | n8n-nodes-base.code | Validate, normalize, enrich metadata | Webhook - Receive Product | Has Remote URL? | ## 🚪 Entry & Validation (describes Webhook→Validate→IF behavior) |
| Has Remote URL? | n8n-nodes-base.if | Branch remote URL vs binary upload | Validate & Enrich Payload | Upload to URL - Remote; Upload to URL - Binary | ## 🚪 Entry & Validation (describes Webhook→Validate→IF behavior) |
| Upload to URL - Remote | n8n-nodes-uploadtourl.uploadToUrl | Upload image by fetching remote URL | Has Remote URL? (true) | Extract Image URL | ## ☁️ Upload to URL → Vision AI (describes upload paths + GPT + parsing) |
| Upload to URL - Binary | n8n-nodes-uploadtourl.uploadToUrl | Upload binary image received by webhook | Has Remote URL? (false) | Extract Image URL | ## ☁️ Upload to URL → Vision AI (describes upload paths + GPT + parsing) |
| Extract Image URL | n8n-nodes-base.code | Normalize UploadToURL response into `imageUrl` | Upload to URL - Remote; Upload to URL - Binary | GPT-4o Vision - Analyse Product | ## ☁️ Upload to URL → Vision AI (describes upload paths + GPT + parsing) |
| GPT-4o Vision - Analyse Product | @n8n/n8n-nodes-langchain.openAi | Vision analysis + JSON copy generation | Extract Image URL | Parse & Merge AI Product Data | ## ☁️ Upload to URL → Vision AI (describes upload paths + GPT + parsing) |
| Parse & Merge AI Product Data | n8n-nodes-base.code | Parse AI JSON, merge tags, compute handle/status | GPT-4o Vision - Analyse Product | Route by Platform | ## ☁️ Upload to URL → Vision AI (describes upload paths + GPT + parsing) |
| Route by Platform | n8n-nodes-base.switch | Route to Shopify vs WooCommerce | Parse & Merge AI Product Data | Shopify - Create Product; WooCommerce - Create Product | ## 🏪 Platform Routing & Product Creation (describes Switch + Shopify/WC mapping) |
| Shopify - Create Product | n8n-nodes-base.shopify | Create Shopify product (currently minimal fields) | Route by Platform (Shopify) | Build Product Response | ## 🏪 Platform Routing & Product Creation (describes Switch + Shopify/WC mapping) |
| WooCommerce - Create Product | n8n-nodes-base.httpRequest | Create WooCommerce product via REST | Route by Platform (WooCommerce) | Build Product Response | ## 🏪 Platform Routing & Product Creation (describes Switch + Shopify/WC mapping) |
| Build Product Response | n8n-nodes-base.code | Normalize platform response to unified payload | Shopify - Create Product; WooCommerce - Create Product | Slack - Notify Team | ## 📣 Notification & Response (describes response merge + Slack + webhook response) |
| Slack - Notify Team | n8n-nodes-base.slack | Post creation notification to Slack | Build Product Response | Respond to Webhook | ## 📣 Notification & Response (describes response merge + Slack + webhook response) |
| Respond to Webhook | n8n-nodes-base.respondToWebhook | Return HTTP 201 JSON response to caller | Slack - Notify Team | — | ## 📣 Notification & Response (describes response merge + Slack + webhook response) |

---

## 4. Reproducing the Workflow from Scratch

1) **Create required credentials / prerequisites**
   1. Install community node: **n8n-nodes-uploadtourl** (via n8n Community Nodes).
   2. Create **UploadToURL** API credentials in n8n.
   3. Create **OpenAI** credentials for the LangChain OpenAI node (ensure access to a vision-capable model such as **gpt-4o**).
   4. Create **Shopify OAuth2** credential with required scopes (at minimum product write scopes).
   5. Create **WooCommerce HTTP Basic Auth** credential:
      - Base site URL stored in the credential as `url` (as referenced by `{{$credentials.url}}`)
      - Username/password = WooCommerce REST API key/secret (or WP basic auth if configured).
   6. Create **Slack OAuth2** credential (optional) with permission to post messages.

2) **Set workflow variable**
   - In n8n, set a variable named: `DEFAULT_PLATFORM`  
     Value: `shopify` or `woocommerce` (used when `platform` not provided).

3) **Add node: “Webhook - Receive Product”**
   - Node type: **Webhook**
   - Method: **POST**
   - Path: `product-catalog-upload`
   - Response mode: **Respond to Webhook node**
   - (Optional) Allowed origins: `*`

4) **Add node: “Validate & Enrich Payload” (Code)**
   - Node type: **Code**
   - Paste logic that:
     - Validates `platform` in `[shopify, woocommerce]` using `$vars.DEFAULT_PLATFORM` fallback
     - Requires `fileUrl` or `filename`
     - Derives `filename`, `mimeType`
     - Sanitizes `sku`
     - Parses `price`, `compareAtPrice`, `inventory`, `weight`
     - Parses tags from comma-separated string into array
     - Outputs normalized JSON fields including `publishImmediately`, `userTitle`, `userDescription`
   - Connect: **Webhook → Validate & Enrich Payload**

5) **Add node: “Has Remote URL?” (IF)**
   - Node type: **IF**
   - Condition: String `{{$json.fileUrl}}` **is not empty**
   - Connect: **Validate & Enrich Payload → Has Remote URL?**

6) **Add nodes: UploadToURL**
   1. **“Upload to URL - Remote”**
      - Node type: **UploadToURL (community)**
      - Operation: `uploadFile`
      - Credentials: your UploadToURL API credential
      - Connect: **Has Remote URL? (true) → Upload to URL - Remote**
   2. **“Upload to URL - Binary”**
      - Same node type/operation/credential
      - Connect: **Has Remote URL? (false) → Upload to URL - Binary**

7) **Add node: “Extract Image URL” (Code)**
   - Node type: **Code**
   - Implement logic to:
     - Read UploadToURL response
     - Extract `imageUrl` from any of: `url | link | data.url | file.url | shortUrl`
     - Force https
     - Merge with prior metadata from “Validate & Enrich Payload”
   - Connect:
     - **Upload to URL - Remote → Extract Image URL**
     - **Upload to URL - Binary → Extract Image URL**

8) **Add node: “GPT-4o Vision - Analyse Product”**
   - Node type: **OpenAI (LangChain)**
   - Select model: **gpt-4o** (or equivalent vision model available)
   - Set:
     - Temperature `0.6`
     - Max tokens `1200`
   - Messages:
     - System: instruct JSON-only response
     - User: include `Image URL: {{$json.imageUrl}}` and the required JSON schema (title, seoTitle, description HTML, shortDescription, bulletFeatures, suggestedCategory, suggestedTags, metaDescription, handle, marketingBlurb, confidenceScore)
   - Connect: **Extract Image URL → GPT-4o Vision - Analyse Product**

9) **Add node: “Parse & Merge AI Product Data” (Code)**
   - Node type: **Code**
   - Implement logic to:
     - Parse the AI response content as JSON
     - Require `title`
     - Merge tags with dedupe and cap
     - Compute `handle` slug
     - Compute `shopifyStatus` (`active|draft`) and `wcStatus` (`publish|draft`)
     - Output a single normalized product object
   - Connect: **GPT-4o Vision - Analyse Product → Parse & Merge AI Product Data**

10) **Add node: “Route by Platform” (Switch)**
   - Node type: **Switch**
   - Rule output “Shopify”: `{{$json.platform}} equals "shopify"`
   - Rule output “WooCommerce”: `{{$json.platform}} equals "woocommerce"`
   - Fallback: `extra` (optional, but recommended to connect to an error response path)
   - Connect: **Parse & Merge AI Product Data → Route by Platform**

11) **Add node: “Shopify - Create Product”**
   - Node type: **Shopify**
   - Auth: **OAuth2**
   - Resource: **Product**
   - Map at least:
     - Title: `{{$json.title}}`
   - Recommended to additionally map (to match the stated intent of the workflow):
     - Images (using `{{$json.imageUrl}}`)
     - Status using `{{$json.shopifyStatus}}`
     - Variant price, SKU, inventory, compare-at price, vendor/product type, tags, SEO fields (depending on node capabilities/metafields strategy)
   - Connect: **Route by Platform (Shopify) → Shopify - Create Product**

12) **Add node: “WooCommerce - Create Product” (HTTP Request)**
   - Node type: **HTTP Request**
   - Method: POST
   - URL: `{{$credentials.url}}/wp-json/wc/v3/products`
   - Auth: **HTTP Basic Auth** (generic credential type)
   - Body: JSON with mappings for name/slug/status/description/short_description/sku/prices/stock/weight/categories/tags/images/meta_data (Yoast + marketing blurb)
   - Connect: **Route by Platform (WooCommerce) → WooCommerce - Create Product**

13) **Add node: “Build Product Response” (Code)**
   - Node type: **Code**
   - Normalize response:
     - Determine platform
     - Extract `productId`, `productUrl`, `imageUrl`
     - Merge key fields into a single response JSON
   - Connect:
     - **Shopify - Create Product → Build Product Response**
     - **WooCommerce - Create Product → Build Product Response**

14) **Add node: “Slack - Notify Team” (optional)**
   - Node type: **Slack**
   - Auth: OAuth2
   - Message text: include product title, sku, price, productUrl, imageUrl, confidenceScore, marketingBlurb
   - Connect: **Build Product Response → Slack - Notify Team**
   - If you skip Slack, connect **Build Product Response → Respond to Webhook** instead.

15) **Add node: “Respond to Webhook”**
   - Node type: **Respond to Webhook**
   - Respond with: **JSON**
   - Status code: `201`
   - Body: `{{$('Build Product Response').first().json}}`
   - Connect: **Slack - Notify Team → Respond to Webhook** (or directly from Build Product Response if Slack removed)

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| 🛍️ AI Product Catalog Architect: mobile-to-store pipeline using UploadToURL + GPT-4o Vision + Shopify/WooCommerce. | Sticky note “📋 Overview” |
| Install community node: **n8n-nodes-uploadtourl**. | Sticky note “📋 Overview” |
| Required APIs: UploadToURL, OpenAI (GPT-4o), Shopify/WooCommerce. | Sticky note “📋 Overview” |
| Set variable `DEFAULT_PLATFORM` to `shopify` or `woocommerce`. | Sticky note “📋 Overview” |
| Disclaimer (FR): Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n… données légales et publiques. | Provided by user instruction |

