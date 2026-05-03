Recover abandoned WooCommerce carts with custom coupon emails

https://n8nworkflows.xyz/workflows/recover-abandoned-woocommerce-carts-with-custom-coupon-emails-12461


# Recover abandoned WooCommerce carts with custom coupon emails

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow detects WooCommerce “add to cart” events, waits a configurable period, checks whether the customer completed a purchase, and—if not—creates a short-lived unique coupon and emails the customer a cart recovery message containing the coupon.

**Typical use cases:**
- Recover abandoned carts with time-limited incentives.
- Send personalized cart content reminders via email after a delay.
- Create one-time, customer-scoped coupons automatically.

### 1.1 Input Reception (Cart event ingestion)
Receives cart payload from WooCommerce via webhook when a customer adds an item to cart.

### 1.2 Delay Window (grace period before recovery action)
Waits 15 minutes to allow customers to complete checkout naturally.

### 1.3 Purchase Verification (prevent sending to buyers)
Queries WooCommerce orders to detect whether the customer placed a completed order during/after the event timestamp.

### 1.4 Decision Branch (order exists vs. abandoned)
If an order exists: stop. If no order exists: continue to coupon generation and email flow.

### 1.5 Coupon Generation & Creation (WooCommerce REST)
Generates a unique coupon code, calculates expiry, then creates the coupon in WooCommerce via REST API.

### 1.6 Email Rendering & Sending (SMTP)
Builds an HTML email using cart data and coupon details, then sends it via SMTP.

---

## 2. Block-by-Block Analysis

### Block 1.1 — Input Reception
**Overview:** Receives cart details from WooCommerce immediately when a cart event occurs. This payload becomes the reference for customer_id, email, cart items, totals, and timestamp.  
**Nodes involved:** `Fetching Cart Contents`

#### Node: Fetching Cart Contents
- **Type / role:** Webhook (Trigger) — entry point; receives POST requests from WooCommerce.
- **Key configuration:**
  - **HTTP Method:** POST  
  - **Path:** `fde986cf-7c26-42e9-885d-cb44ee305863` (also used as webhookId)
- **Key variables/fields expected in payload:** referenced later as `body.arg.*`, notably:
  - `body.arg.customer_id`
  - `body.arg.billing_email`
  - `body.arg.cart_contents` (array of items)
  - `body.arg.cart_total`
  - `body.arg.timestamp`
- **Connections:**
  - Output → `Waiting Time`
- **Failure/edge cases:**
  - Payload shape mismatch (missing `body.arg.*`) will break downstream expressions.
  - If WooCommerce webhook is not configured to send full cart details, email/coupon logic will have incomplete data.
- **Version notes:** Webhook node `typeVersion: 2`.

---

### Block 1.2 — Delay Window
**Overview:** Waits 15 minutes after the add-to-cart event before checking for an order, reducing unnecessary coupon creation.  
**Nodes involved:** `Waiting Time`

#### Node: Waiting Time
- **Type / role:** Wait — pauses workflow for a fixed duration.
- **Key configuration:**
  - **Unit:** minutes
  - **Amount:** 15
- **Connections:**
  - Input ← `Fetching Cart Contents`
  - Output → `Fetching If Customer has placed order`
- **Failure/edge cases:**
  - Wait nodes require n8n to be able to resume executions (persistence/storage and webhook resume if applicable).
  - If executions are pruned before resumption, the flow may not continue.
- **Version notes:** `typeVersion: 1.1`

---

### Block 1.3 — Purchase Verification
**Overview:** Queries WooCommerce for completed orders tied to the customer and after the cart timestamp, to avoid sending coupons to customers who already bought.  
**Nodes involved:** `Fetching If Customer has placed order`, `Set Default Order Data`

#### Node: Fetching If Customer has placed order
- **Type / role:** WooCommerce — fetch orders via WooCommerce API.
- **Key configuration (interpreted):**
  - **Resource:** Order
  - **Operation:** Get All
  - **Limit:** 10
  - **Filters/Options:**
    - **after:** `$('Fetching Cart Contents').item.json.body.arg.timestamp`  
      (intended: only orders after the cart event)
    - **search:** `$json.body.arg.customer_id`  
      (intended: locate orders by customer reference)
    - **status:** `completed`
  - **Credentials:** “WooCommerce account”
  - **alwaysOutputData:** true (ensures downstream node runs even if no orders)
- **Connections:**
  - Input ← `Waiting Time`
  - Output → `Set Default Order Data`
- **Failure/edge cases:**
  - **Auth/permissions:** invalid WooCommerce credentials or missing REST permissions.
  - **Filter semantics:** WooCommerce “search” may not reliably match `customer_id` (often search matches email/name/order fields). A more reliable approach is using `customer` parameter (customer ID) if available in the node/version/API.
  - **Timestamp format:** `after` must be RFC3339/ISO format for WooCommerce REST; a custom “YYYY-MM-DD HH:mm:ss” string may not behave as expected.
- **Version notes:** `typeVersion: 1`

#### Node: Set Default Order Data
- **Type / role:** Code — normalizes WooCommerce response and merges it with the original webhook payload for consistent downstream access.
- **Configuration choices:**
  - Reads the **first item** of input from WooCommerce node (`$input.first()`).
  - Reads original webhook payload via `$node["Fetching Cart Contents"].json`.
  - Ensures `orders` is always an array; falls back to `[]` on invalid formats.
  - Returns a single item: `{ orders: [...], ...webhookData }`.
- **Key variables/expressions:**
  - `const inputData = $input.first() || {};`
  - `const webhookData = $node["Fetching Cart Contents"].json || {};`
  - `let orders = inputData.json;` then normalization to array.
- **Connections:**
  - Input ← `Fetching If Customer has placed order`
  - Output → `If, There's a Order Placed`
- **Failure/edge cases:**
  - If WooCommerce node returns an array of items (typical n8n behavior), using `$input.first().json` may only capture the first order rather than all orders; you may want `$input.all().map(i => i.json)` instead.
  - The node logs to console; in some n8n environments logs may be restricted.
- **Version notes:** Code node `typeVersion: 2`

---

### Block 1.4 — Decision Branch
**Overview:** Determines whether to stop (order exists) or proceed (abandoned cart).  
**Nodes involved:** `If, There's a Order Placed`, `No Operation, do nothing1`

#### Node: If, There's a Order Placed
- **Type / role:** IF — branching logic.
- **Current condition (as configured):**
  - Checks if `{{ $node["Set Default Order Data"].json.length }}` is `> 0`
- **Connections:**
  - Input ← `Set Default Order Data`
  - Output 0 (true branch) → `No Operation, do nothing1`
  - Output 1 (false branch) → `Generate Unique Coupon Code`
- **Critical issue / edge case:**
  - `Set Default Order Data` returns an object with `orders`, not an array at root. Therefore `.json.length` is likely `undefined`.
  - Intended condition should be something like:
    - `{{ $node["Set Default Order Data"].json.orders.length }}` > 0
- **Version notes:** IF node `typeVersion: 2.2`

#### Node: No Operation, do nothing1
- **Type / role:** NoOp — ends the “customer purchased” branch.
- **Connections:**
  - Input ← `If, There's a Order Placed` (true branch)
  - No outputs
- **Failure/edge cases:** None (used as terminator).

---

### Block 1.5 — Coupon Generation & Creation
**Overview:** Generates a unique coupon code, computes expiry timestamp, then creates the coupon in WooCommerce using REST.  
**Nodes involved:** `Generate Unique Coupon Code`, `Create Abandoned Cart Coupon (HTTP)`

#### Node: Generate Unique Coupon Code
- **Type / role:** Code — creates a short coupon code and expiry metadata.
- **Configuration choices:**
  - Uses a local `CONFIG` object:
    - `COUPON_PREFIX` (empty by default)
    - `EXPIRY_MINUTES` = 30
    - `TIMEZONE` = 'UTC'
  - Uses `customer_id` + timestamp to derive a 3-digit suffix (001–1000) and handles collisions via `usedCodes`.
  - Computes:
    - `expiry_time` formatted as `HH:mm` in configured timezone
    - `date_expires` formatted as `YYYY-MM-DD HH:mm:ss` in configured timezone
- **Key variables/expressions:**
  - `const customerId = $input.first().json.body.arg.customer_id;`
  - `dateExpires` via `toLocaleString('sv-SE', ...)` then replace `'T'` (note: `sv-SE` typically yields `"YYYY-MM-DD HH:mm:ss"` already)
- **Connections:**
  - Input ← `If, There's a Order Placed` (false branch)
  - Output → `Create Abandoned Cart Coupon (HTTP)`
- **Failure/edge cases:**
  - If webhook payload is missing `body.arg.customer_id`, code throws.
  - `usedCodes` only persists within the execution item, not globally; it does **not** prevent collisions across separate workflow executions.
  - Coupon code length/format restrictions in WooCommerce may apply.
- **Version notes:** Code node `typeVersion: 2`

#### Node: Create Abandoned Cart Coupon (HTTP)
- **Type / role:** HTTP Request — creates the coupon in WooCommerce via REST API using WooCommerce credentials.
- **Configuration choices:**
  - **Method:** POST
  - **Authentication:** predefined credential type `wooCommerceApi`
  - **Body:** JSON with:
    - `code` from generated code
    - `discount_type`: percent
    - `amount`: "5" (5% discount)
    - `date_expires`: generated date
    - `individual_use`: true
    - `usage_limit`: 1
    - `usage_limit_per_user`: 1
    - `customer_id`: from webhook payload
    - `meta_data`: includes `expiry_time`
- **Key expressions:**
  - `{{ $json.code }}`
  - `{{ $json.date_expires }}`
  - `{{ $('Set Default Order Data').item.json.body.arg.customer_id }}`
- **Connections:**
  - Input ← `Generate Unique Coupon Code`
  - Output → `Generate Email HTML`
- **Critical issue / edge case:**
  - **URL is empty** in the node. To work, it must point to your WooCommerce REST endpoint, typically:
    - `https://YOUR_DOMAIN/wp-json/wc/v3/coupons`
  - `customer_id` is not a standard coupon field in all WooCommerce versions; WooCommerce supports `email_restrictions` for limiting coupon usage by email. If `customer_id` is rejected, use `email_restrictions: [billing_email]` instead.
  - `date_expires` may require ISO8601; some WooCommerce versions accept RFC3339 (e.g., `2026-01-19T12:34:56`).
- **Version notes:** HTTP Request node `typeVersion: 4.2`

---

### Block 1.6 — Email Rendering & Sending
**Overview:** Builds an HTML email containing cart items and coupon details, then sends it via SMTP to the billing email captured from the cart event.  
**Nodes involved:** `Generate Email HTML`, `Abandoned Cart Recovery Email`

#### Node: Generate Email HTML
- **Type / role:** Code — transforms cart + coupon into an email payload.
- **Configuration choices:**
  - `CONFIG` includes `TIMEZONE`, `CURRENCY_SYMBOL`, `LOCALE_FORMAT`.
  - Reads cart data from `Set Default Order Data` webhook merge:
    - `cart_contents`, `billing_email`, `cart_total`, `timestamp`
  - Reads coupon info from:
    - `Generate Unique Coupon Code` (code)
    - `Create Abandoned Cart Coupon (HTTP)` (date_expires, meta_data expiry_time)
  - Builds HTML table rows per cart item (image/name/qty/price).
  - Constructs `cartUrl` using a `baseUrl` (currently empty).
- **Key expressions/variables:**
  - `const cartContents = $('Set Default Order Data').first().json.body.arg.cart_contents || [];`
  - `const couponCode = $node["Generate Unique Coupon Code"].json.code;`
  - `const dateExpires = $node["Create Abandoned Cart Coupon (HTTP)"].json.date_expires;`
- **Connections:**
  - Input ← `Create Abandoned Cart Coupon (HTTP)`
  - Output → `Abandoned Cart Recovery Email`
- **Critical issues / edge cases:**
  - `const baseUrl = "";` must be set (e.g., `https://yourstore.com`) or derived from config/environment.
  - `htmlContent` is currently an empty template string; no actual email body is produced unless you add HTML.
  - `textContent` is referenced in the return payload but **is not defined** (will throw a ReferenceError). You must define `const textContent = '...'`.
  - Item image fallback: `item.image_url || 'https:'` is not a valid placeholder URL.
- **Version notes:** Code node `typeVersion: 2`

#### Node: Abandoned Cart Recovery Email
- **Type / role:** Email Send (SMTP) — sends final email to customer.
- **Configuration choices:**
  - **To:** `$('Set Default Order Data').item.json.body.arg.billing_email`
  - **Subject:** `Greetings! You left [Your Product Name] in your cart!` (placeholder)
  - **HTML Body:** `{{ $json.email_html }}`
  - **Append attribution:** false
  - **From Email:** empty (must be set for most SMTP servers)
  - **Credentials:** SMTP
- **Connections:**
  - Input ← `Generate Email HTML`
  - Output: none
- **Failure/edge cases:**
  - SMTP authentication issues, missing from address, SPF/DKIM problems, provider blocking.
  - If `billing_email` is missing/empty, node may fail or send to blank.
- **Version notes:** Email node `typeVersion: 2.1`

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Fetching Cart Contents | Webhook | Receives cart event payload from WooCommerce | — | Waiting Time | # Recover abandoned carts with custom coupons for WooCommerce… Need Help? Links: https://cal.com/mattakshitij/workflow-troubleshoot , https://x.com/matta_kshitij |
| Waiting Time | Wait | Delays execution 15 minutes | Fetching Cart Contents | Fetching If Customer has placed order | # Recover abandoned carts with custom coupons for WooCommerce… Need Help? Links: https://cal.com/mattakshitij/workflow-troubleshoot , https://x.com/matta_kshitij |
| Fetching If Customer has placed order | WooCommerce | Checks completed orders after timestamp | Waiting Time | Set Default Order Data | # Recover abandoned carts with custom coupons for WooCommerce… Need Help? Links: https://cal.com/mattakshitij/workflow-troubleshoot , https://x.com/matta_kshitij |
| Set Default Order Data | Code | Normalizes order data + merges webhook payload | Fetching If Customer has placed order | If, There's a Order Placed | # Recover abandoned carts with custom coupons for WooCommerce… Need Help? Links: https://cal.com/mattakshitij/workflow-troubleshoot , https://x.com/matta_kshitij |
| If, There's a Order Placed | IF | Branch: stop if order exists; else create coupon + email | Set Default Order Data | No Operation, do nothing1; Generate Unique Coupon Code | # Recover abandoned carts with custom coupons for WooCommerce… Need Help? Links: https://cal.com/mattakshitij/workflow-troubleshoot , https://x.com/matta_kshitij |
| No Operation, do nothing1 | NoOp | Terminates “order exists” branch | If, There's a Order Placed | — | # Recover abandoned carts with custom coupons for WooCommerce… Need Help? Links: https://cal.com/mattakshitij/workflow-troubleshoot , https://x.com/matta_kshitij |
| Generate Unique Coupon Code | Code | Generates unique code + expiry times | If, There's a Order Placed | Create Abandoned Cart Coupon (HTTP) | ## 3. Customize the coupon code according to your needs… (COUPON_PREFIX, EXPIRY_MINUTES, TIMEZONE) |
| Create Abandoned Cart Coupon (HTTP) | HTTP Request | Creates coupon in WooCommerce via REST | Generate Unique Coupon Code | Generate Email HTML | # Recover abandoned carts with custom coupons for WooCommerce… Need Help? Links: https://cal.com/mattakshitij/workflow-troubleshoot , https://x.com/matta_kshitij |
| Generate Email HTML | Code | Builds dynamic HTML email content | Create Abandoned Cart Coupon (HTTP) | Abandoned Cart Recovery Email | ## 4. Customize email HTML according to your needs… (TIMEZONE, CURRENCY_SYMBOL, variables) |
| Abandoned Cart Recovery Email | Email Send (SMTP) | Sends recovery email to customer | Generate Email HTML | — | # Recover abandoned carts with custom coupons for WooCommerce… Need Help? Links: https://cal.com/mattakshitij/workflow-troubleshoot , https://x.com/matta_kshitij |

Sticky notes present but not directly “attached” to a single node in JSON; content replicated contextually:
- “Configuring webhook…” applies to `Fetching Cart Contents`
- “Add PHP snippet…” applies to the overall WooCommerce → webhook payload enrichment
- “Customize coupon…” applies to `Generate Unique Coupon Code`
- “Customize email HTML…” applies to `Generate Email HTML`
- The large overview sticky note applies broadly to the workflow

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Name it: **Recover abandoned carts with custom coupons for WooCommerce**
- Ensure workflow settings use default execution order (`v1`).

2) **Add Trigger: Webhook**
- Node: **Webhook** named `Fetching Cart Contents`
- Method: **POST**
- Path: generate or set to `fde986cf-7c26-42e9-885d-cb44ee305863`
- Activate “Test URL” while configuring WooCommerce, then switch to Production URL when activating.
- **WooCommerce setup:** WooCommerce → Settings → Advanced → Webhooks
  - Topic: **Action**
  - Action Event: `woocommerce_add_to_cart`
  - Delivery URL: your n8n webhook URL for this node

3) **Add Wait**
- Node: **Wait** named `Waiting Time`
- Amount: **15**
- Unit: **minutes**
- Connect: `Fetching Cart Contents` → `Waiting Time`

4) **Add WooCommerce “Get All Orders”**
- Node: **WooCommerce** named `Fetching If Customer has placed order`
- Resource: **Order**
- Operation: **Get All**
- Limit: **10**
- Status: **completed**
- “After” filter: set expression to webhook timestamp, e.g.  
  `$('Fetching Cart Contents').item.json.body.arg.timestamp`
- “Search” filter: currently uses `customer_id`, but for correctness prefer a customer-id filter if supported by your node/version (or filter by billing email).
- Credentials: configure **WooCommerce API** credentials in n8n:
  - Base URL (store URL)
  - Consumer Key / Consumer Secret (WooCommerce REST API keys)
- Connect: `Waiting Time` → `Fetching If Customer has placed order`

5) **Add Code node to normalize data**
- Node: **Code** named `Set Default Order Data`
- Paste logic that:
  - takes WooCommerce node output
  - merges with `Fetching Cart Contents` payload
  - always returns `{ orders: [...] , ...webhookData }`
- Connect: `Fetching If Customer has placed order` → `Set Default Order Data`

6) **Add IF node (fix condition)**
- Node: **IF** named `If, There's a Order Placed`
- Condition: Number `>`  
  Left value (expression): `{{ $node["Set Default Order Data"].json.orders.length }}`
  Right value: `0`
- Connect: `Set Default Order Data` → `If, There's a Order Placed`

7) **Add NoOp to end “order exists” path**
- Node: **No Operation** named `No Operation, do nothing1`
- Connect: `If, There's a Order Placed` **true** output → `No Operation, do nothing1`

8) **Add Code node to generate coupon**
- Node: **Code** named `Generate Unique Coupon Code`
- Set:
  - `COUPON_PREFIX` (e.g., `HELLO`)
  - `EXPIRY_MINUTES` (e.g., 30)
  - `TIMEZONE` matching WordPress/WooCommerce timezone
- Ensure it reads customer id from webhook merge (or directly from webhook node).
- Connect: `If, There's a Order Placed` **false** output → `Generate Unique Coupon Code`

9) **Add HTTP Request to create coupon in WooCommerce**
- Node: **HTTP Request** named `Create Abandoned Cart Coupon (HTTP)`
- Method: **POST**
- URL: `https://YOUR_DOMAIN/wp-json/wc/v3/coupons`
- Authentication: **Predefined credential type** → `wooCommerceApi`
- Body: JSON with fields like:
  - `code`, `discount_type`, `amount`, `date_expires`, usage limits
  - Prefer `email_restrictions: ["{{ billing_email }}"]` if `customer_id` is not accepted by your WooCommerce version.
  - Include `meta_data` with `expiry_time` if you want to display it in email.
- Connect: `Generate Unique Coupon Code` → `Create Abandoned Cart Coupon (HTTP)`

10) **Add Code node to generate email HTML**
- Node: **Code** named `Generate Email HTML`
- Set configuration:
  - `TIMEZONE`
  - `CURRENCY_SYMBOL` (e.g., `$`)
- Implement:
  - Proper `baseUrl` (e.g., `https://yourstore.com`)
  - Real `htmlContent` template using `cartContents`, `couponCode`, `cartUrl`, totals
  - Define `textContent` to avoid runtime error
- Connect: `Create Abandoned Cart Coupon (HTTP)` → `Generate Email HTML`

11) **Add Email Send node (SMTP)**
- Node: **Email Send** named `Abandoned Cart Recovery Email`
- SMTP credentials: configure in n8n (host/port/user/pass or provider-specific).
- From: set a valid sender (e.g., `sales@yourstore.com`)
- To: expression: `{{ $('Set Default Order Data').item.json.body.arg.billing_email }}`
- Subject: customize (avoid placeholder)
- HTML: `{{ $json.email_html }}`
- Connect: `Generate Email HTML` → `Abandoned Cart Recovery Email`

12) **WordPress/WooCommerce PHP snippet**
- Add the referenced snippet to `functions.php` or a snippets plugin so WooCommerce can send the required cart payload fields (cart contents, totals, email, timestamp, etc.).  
  Link provided by the author: https://pastebin.com/tp0yFSRN

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Configuring webhook in n8n and WooCommerce… Topic: Action, Action Event: woocommerce_add_to_cart” | WooCommerce Settings → Advanced → Webhooks |
| “Add the provided code to your WP website… Access the code from here” | https://pastebin.com/tp0yFSRN |
| “Need Help? Schedule a 1:1 Troubleshooting Call / Ping me on X” | https://cal.com/mattakshitij/workflow-troubleshoot ; https://x.com/matta_kshitij |
| Workflow requires WooCommerce REST API access + SMTP credentials | Environment prerequisites |

Key implementation warnings (cross-cutting):
- Fix IF condition to use `orders.length`.
- Set missing values: coupon creation URL, email HTML body, `textContent`, `baseUrl`, and SMTP “From”.
- Validate WooCommerce coupon fields (`customer_id` may not be supported; consider `email_restrictions`).