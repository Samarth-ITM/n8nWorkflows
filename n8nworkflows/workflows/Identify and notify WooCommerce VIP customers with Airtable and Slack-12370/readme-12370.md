Identify and notify WooCommerce VIP customers with Airtable and Slack

https://n8nworkflows.xyz/workflows/identify-and-notify-woocommerce-vip-customers-with-airtable-and-slack-12370


# Identify and notify WooCommerce VIP customers with Airtable and Slack

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** Automatically identify high-value (VIP) WooCommerce customers by analyzing their order history, then **store VIP details in Airtable** and **notify a Slack channel**.

**Target use cases**
- Flagging customers for priority support or special handling
- Building/maintaining a VIP customer list in Airtable
- Real-time-ish team alerts when VIPs are detected

### Logical blocks
**1.1 Scheduled Trigger & Store Context**
- Runs on an interval and sets the WooCommerce domain used by subsequent HTTP requests.

**1.2 Fetch & Filter Recent Orders**
- Pulls WooCommerce orders, keeps only orders in relevant states (completed/processing), and normalizes fields.

**1.3 VIP Qualification (Customer Aggregation + Order History)**
- Deduplicates customers from orders, fetches each customer’s historical orders, and computes lifetime spend + total completed orders to assign a VIP tier.

**1.4 Persist & Notify**
- Filters for VIP-eligible customers, creates a record in Airtable, and posts a formatted Slack message.

---

## 2. Block-by-Block Analysis

### 2.1 Scheduled Trigger & Store Context

**Overview:** Starts the workflow on a schedule and defines the WooCommerce domain used to build WooCommerce REST API URLs.

**Nodes involved:**
- Scheduled Check for New Orders
- Set WooCommerce Domain

#### Node: Scheduled Check for New Orders
- **Type / role:** `Schedule Trigger` — entry point that runs periodically.
- **Configuration (interpreted):** Runs every *N minutes* (the JSON shows “minutes” interval but does not specify a numeric value in the snippet—verify in UI).
- **Inputs/Outputs:** No input. Output goes to **Set WooCommerce Domain**.
- **Version notes:** typeVersion `1.2`.
- **Failure/edge cases:**
  - Misconfigured interval (too frequent) can cause rate limits on WooCommerce.
  - If n8n instance is paused/stopped, schedules won’t run.

#### Node: Set WooCommerce Domain
- **Type / role:** `Set` — defines a variable `wc_domain`.
- **Configuration choices:**
  - Creates field `wc_domain` with value: `{{Your WC Domain Here}}` (placeholder that must be replaced, e.g. `example.com`).
- **Key expressions/variables:** sets `$json.wc_domain`.
- **Inputs/Outputs:** Input from trigger; output to **Fetch Orders**.
- **Version notes:** typeVersion `3.4`.
- **Failure/edge cases:**
  - Leaving placeholder value breaks downstream HTTP URL construction.
  - Including protocol here (e.g., `https://`) would result in `https://https://...` later since URLs are built as `https://{{ $json.wc_domain }}`.

**Sticky note coverage (context):**
- “Triggers the workflow automatically at a set interval (minutes).”

---

### 2.2 Fetch & Filter Recent Orders

**Overview:** Pulls orders from WooCommerce via REST API and keeps only “completed” or “processing” orders, then extracts the customer fields required downstream.

**Nodes involved:**
- Fetch Orders
- Filter Completed & Processing Orders
- Format order data

#### Node: Fetch Orders
- **Type / role:** `HTTP Request` — calls WooCommerce REST API orders endpoint.
- **Configuration choices:**
  - **URL:** `https://{{ $json.wc_domain}}/wp-json/wc/v3/orders`
  - **Auth:** HTTP Basic Auth via n8n credential “Woocommerece Credentials” (consumer key/secret typically used as basic auth for WooCommerce REST).
  - **Query:** `sendQuery: true`, but query parameters show an incomplete placeholder (`{"name":"sta"}`) which does not apply any real filtering.
- **Key expressions/variables:**
  - Uses `$json.wc_domain` from **Set WooCommerce Domain**.
- **Inputs/Outputs:** Input from **Set WooCommerce Domain**. Output to **Filter Completed & Processing Orders**.
- **Version notes:** typeVersion `4.3`.
- **Failure/edge cases:**
  - 401/403 if credentials are wrong or WooCommerce REST API permissions insufficient.
  - 404 if domain/path incorrect.
  - Pagination: WooCommerce orders endpoint returns paginated results (default `per_page=10`). This node does not handle pagination; you may miss orders unless you add `per_page` and pagination logic.
  - The malformed query parameter may be ignored; remove or fix (e.g., `status=completed,processing`).

#### Node: Filter Completed & Processing Orders
- **Type / role:** `IF` — passes through orders whose status is `completed` OR `processing`.
- **Configuration choices:**
  - OR combinator with two string equals conditions:
    - `$json.status == "completed"`
    - `$json.status == "processing"`
- **Inputs/Outputs:** Input from **Fetch Orders**. “True” output goes to **Format order data**. (False output is unused.)
- **Version notes:** typeVersion `2.2`.
- **Failure/edge cases:**
  - If WooCommerce returns a wrapped payload (not one item per order), the IF node may not evaluate as intended. Ensure the HTTP node returns an array of orders mapped to items.
  - Status values must match WooCommerce exactly (case-sensitive).

#### Node: Format order data
- **Type / role:** `Set` — normalizes and selects order fields.
- **Configuration choices:** Creates:
  - `id` (number) = `$json.id`
  - `status` (string) = `$json.status`
  - `customer_id` (number) = `$json.customer_id`
  - `customer_name` (string) = `{{$json.billing.first_name}} {{$json.billing.last_name}}`
  - `customer_note` (string) = `$json.customer_note`
  - `date_created_gmt` (string) = `$json.date_created_gmt`
  - `payment_url` (string) = `$json.payment_url`
- **Inputs/Outputs:** Input from filter; output to **Deduplicate Customers**.
- **Version notes:** typeVersion `3.4`.
- **Failure/edge cases:**
  - Missing `billing` fields for some orders could cause blank names; use fallbacks if needed.
  - Guest orders often have `customer_id = 0`—handled later.

**Sticky note coverage (context):**
- “## Fetch and Filter WooCommerce Orders … retrieves all orders … filters only completed or processing … using configured store domain.”

---

### 2.3 VIP Qualification (Customer Aggregation + Order History)

**Overview:** Deduplicates customers found in the current order set, fetches each customer’s historical orders, and computes VIP tier based on lifetime completed spend or order count.

**Nodes involved:**
- Deduplicate Customers
- Fetch Customer Orders
- Calculate VIP Tier
- Check VIP Eligibility

#### Node: Deduplicate Customers
- **Type / role:** `Code` — reduces many order items down to one item per unique `customer_id`.
- **Configuration choices (logic):**
  - Reads all incoming items.
  - Builds a `unique` object keyed by `customer_id`.
  - Adds `wc_domain` into each deduplicated customer item by reading from node `Set WooCommerce Domain`:
    - `const domain = $('Set WooCommerce Domain').first().json.wc_domain;`
  - Returns one item per customer.
- **Inputs/Outputs:** Input from **Format order data**; output to **Fetch Customer Orders**.
- **Version notes:** typeVersion `2`.
- **Failure/edge cases:**
  - If `Set WooCommerce Domain` node name changes, the selector `$('Set WooCommerce Domain')` breaks.
  - If upstream returns no items, this returns empty and stops flow.
  - Includes customers with `customer_id = 0` (guest) here; filtered later.

#### Node: Fetch Customer Orders
- **Type / role:** `HTTP Request` — fetches historical orders for each deduped customer.
- **Configuration choices:**
  - **URL:** `https://{{ $json.wc_domain }}/wp-json/wc/v3/orders?customer={{ $json.customer_id }}&status=completed,processing`
  - Also sets query parameter `customer={{ $json.customer_id }}` redundantly while already in the URL.
  - **Auth:** same WooCommerce Basic Auth credential.
- **Inputs/Outputs:** Input from **Deduplicate Customers**; output to **Calculate VIP Tier**.
- **Version notes:** typeVersion `4.3`.
- **Failure/edge cases:**
  - Pagination again: customers with many orders will be truncated unless pagination handled (`per_page`, `page` loop).
  - Rate limiting: one request per customer per run can spike usage on larger stores.
  - The node outputs a *list of orders per request*; depending on n8n settings, you may get nested arrays—confirm that the output is flattened into separate items. If not, VIP calculation may misbehave.

#### Node: Calculate VIP Tier
- **Type / role:** `Code` — aggregates order history per customer and assigns tier.
- **Configuration choices (logic):**
  - Reads **all items** (orders) coming from previous step.
  - Ignores:
    - orders with `total <= 0`
    - orders not `completed` (note: it fetches `processing` too, but explicitly excludes them here)
  - Aggregates per `customer_id`:
    - `lifetime_value` = sum of `total`
    - `total_orders` = count of unique order IDs
  - Tier rules:
    - **Platinum** if `lifetime_value >= 20000` (currency assumed store currency)
      - `vip_reason = "lifetime_value"`
    - **Gold** if `total_orders >= 5`
      - `vip_reason = "total_orders"`
    - Else **Silver** with `vip_reason = null`
  - Removes guest checkouts: filters out customers where `customer_id` is falsy or `0`.
- **Inputs/Outputs:** Input from **Fetch Customer Orders**; output to **Check VIP Eligibility**.
- **Version notes:** typeVersion `2`.
- **Failure/edge cases:**
  - Currency: lifetime thresholds assume a specific currency and no refunds; refunded orders are not accounted for.
  - Only counts `completed` orders, even though the flow initially also considers `processing`.
  - If upstream output is not one-order-per-item, parsing may fail (e.g., `order.json` is an array).
  - Large histories can exceed execution time/memory without pagination controls.

#### Node: Check VIP Eligibility
- **Type / role:** `IF` — gates which tiers proceed to storage/notification.
- **Configuration choices:**
  - Condition: `$json.tier != "none"` (string notEquals)
- **Inputs/Outputs:** Input from **Calculate VIP Tier**; “True” output to **Save VIP Customer**. (False output unused.)
- **Version notes:** typeVersion `2.2`.
- **Failure/edge cases / logic issue:**
  - The tier values produced are **Platinum / Gold / Silver**; none equals `"none"`.
  - Result: **all customers (including Silver) pass**. If intent is “VIP only”, you likely want `tier != "Silver"` or `vip_reason is not null` or `tier IN (Gold, Platinum)`.

**Sticky note coverage (context):**
- “## VIP Qualification Process … cleans customer data, gathers all their past orders and decides whether they are a VIP…”

---

### 2.4 Persist & Notify

**Overview:** Creates a VIP record in Airtable and sends a Slack alert with VIP metrics.

**Nodes involved:**
- Save VIP Customer
- Notify Team

#### Node: Save VIP Customer
- **Type / role:** `Airtable` — creates a record in a table.
- **Configuration choices:**
  - **Base:** “n8n Demo” (`appF2iYPgVqqyXDC1`)
  - **Table:** “VIP Customer” (`tblfU9DsiUwVEowKM`)
  - **Operation:** Create record
  - **Mapped fields:**
    - `Id` = `$json.customer_id`
    - `Customer Name` = `$json.customer_name`
    - `VIP Tier` = `$json.tier`
    - `Vip reason` = `$json.vip_reason`
    - `Lifetime value` = `$json.lifetime_value`
    - `Total Orders` = `$json.total_orders`
- **Inputs/Outputs:** Input from **Check VIP Eligibility**; output to **Notify Team**.
- **Version notes:** typeVersion `2.1`.
- **Failure/edge cases:**
  - Airtable auth errors (invalid/expired PAT, insufficient base permissions).
  - “Create” will insert duplicates on every run for the same customer unless you implement upsert (search + update) logic.
  - Field types must match (numbers vs strings); conversion is disabled (`attemptToConvertTypes: false`).

#### Node: Notify Team
- **Type / role:** `Slack` — posts a channel message.
- **Configuration choices:**
  - Posts to channel “n8n” (`C09S57E2JQ2`)
  - Message text uses Airtable output structure `fields[...]`, e.g.:
    - `{{ $json.fields["Customer Name"] }}`
    - `{{ $json.fields['Lifetime value'] }}`
    - `{{ $json.fields['VIP Tier'] }}`
    - etc.
  - Disables “include link to workflow”.
- **Inputs/Outputs:** Input from **Save VIP Customer**; no downstream nodes.
- **Version notes:** typeVersion `2.3`.
- **Failure/edge cases:**
  - If Airtable node fails or returns different schema, Slack expressions will break (missing `fields`).
  - Slack permission issues (missing `chat:write`, channel access).
  - Channel ID must be valid in the connected workspace.

**Sticky note coverage (context):**
- “## Store & Notify VIP Customers … saves into Airtable and sends a Slack alert…”

---

### 2.5 Sticky Notes (documentation nodes)

These are `stickyNote` nodes used for annotation only (no runtime effect):
- Sticky Note (trigger explanation)
- Sticky Note1 (fetch/filter block)
- Sticky Note2 (VIP qualification block)
- Sticky Note3 (store/notify block)
- Sticky Note4 (overall description + setup steps)

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Scheduled Check for New Orders | scheduleTrigger | Periodic workflow entry point | — | Set WooCommerce Domain | Triggers the workflow automatically at a set interval (minutes). |
| Set WooCommerce Domain | set | Stores WooCommerce domain in `wc_domain` | Scheduled Check for New Orders | Fetch Orders | ## Fetch and Filter WooCommerce Orders; This process retrieves all orders from the WooCommerce store and filters only those that are completed or processing using the configured store domain. |
| Fetch Orders | httpRequest | Calls WooCommerce orders endpoint | Set WooCommerce Domain | Filter Completed & Processing Orders | ## Fetch and Filter WooCommerce Orders; This process retrieves all orders from the WooCommerce store and filters only those that are completed or processing using the configured store domain. |
| Filter Completed & Processing Orders | if | Keeps orders with status completed/processing | Fetch Orders | Format order data | ## Fetch and Filter WooCommerce Orders; This process retrieves all orders from the WooCommerce store and filters only those that are completed or processing using the configured store domain. |
| Format order data | set | Normalizes order fields for downstream use | Filter Completed & Processing Orders | Deduplicate Customers | ## VIP Qualification Process; This step cleans customer data, gathers all their past orders and decides whether they are a VIP based on how much they spent or how many orders they made. |
| Deduplicate Customers | code | One item per unique customer, adds `wc_domain` | Format order data | Fetch Customer Orders | ## VIP Qualification Process; This step cleans customer data, gathers all their past orders and decides whether they are a VIP based on how much they spent or how many orders they made. |
| Fetch Customer Orders | httpRequest | Pulls historical orders for each customer | Deduplicate Customers | Calculate VIP Tier | ## VIP Qualification Process; This step cleans customer data, gathers all their past orders and decides whether they are a VIP based on how much they spent or how many orders they made. |
| Calculate VIP Tier | code | Aggregates spend/orders; assigns tier + reason | Fetch Customer Orders | Check VIP Eligibility | ## VIP Qualification Process; This step cleans customer data, gathers all their past orders and decides whether they are a VIP based on how much they spent or how many orders they made. |
| Check VIP Eligibility | if | Gates who proceeds to Airtable/Slack | Calculate VIP Tier | Save VIP Customer | ## Store & Notify VIP Customers; This part makes sure the customer truly qualifies as a VIP, saves their VIP details into Airtable and sends a Slack alert to notify the team. |
| Save VIP Customer | airtable | Creates VIP record in Airtable | Check VIP Eligibility | Notify Team | ## Store & Notify VIP Customers; This part makes sure the customer truly qualifies as a VIP, saves their VIP details into Airtable and sends a Slack alert to notify the team. |
| Notify Team | slack | Sends Slack message with VIP details | Save VIP Customer | — | ## Store & Notify VIP Customers; This part makes sure the customer truly qualifies as a VIP, saves their VIP details into Airtable and sends a Slack alert to notify the team. |
| Sticky Note | stickyNote | Annotation | — | — | ## I'm a note; This workflow automatically finds your important (VIP) customers... (see full note in section 5). |
| Sticky Note1 | stickyNote | Annotation | — | — | ## Fetch and Filter WooCommerce Orders; This process retrieves all orders... |
| Sticky Note2 | stickyNote | Annotation | — | — | ## VIP Qualification Process; This step cleans customer data... |
| Sticky Note3 | stickyNote | Annotation | — | — | ## Store & Notify VIP Customers; This part makes sure... |
| Sticky Note4 | stickyNote | Annotation | — | — | ## I'm a note; This workflow automatically finds your important (VIP) customers... (see full note in section 5). |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Trigger**
   1. Add node: **Schedule Trigger**
   2. Set interval to run every **X minutes** (choose based on store volume and rate limits).

2) **Add WooCommerce domain context**
   1. Add node: **Set**
   2. Add field `wc_domain` (String), value like: `myshop.com` (no `https://`).
   3. Connect: **Schedule Trigger → Set (WooCommerce Domain)**

3) **Fetch orders from WooCommerce**
   1. Add node: **HTTP Request**
   2. Method: **GET**
   3. URL: `https://{{ $json.wc_domain }}/wp-json/wc/v3/orders`
   4. Authentication: **HTTP Basic Auth** (create credential)
      - Username: WooCommerce **Consumer Key**
      - Password: WooCommerce **Consumer Secret**
   5. (Recommended) Add query parameters:
      - `status` = `completed,processing`
      - `per_page` = `100`
      - (Optional) `after` = timestamp to limit to recent orders
   6. Connect: **Set WooCommerce Domain → Fetch Orders**

4) **Filter order statuses**
   1. Add node: **IF**
   2. Conditions (OR):
      - `{{$json.status}}` equals `completed`
      - `{{$json.status}}` equals `processing`
   3. Connect: **Fetch Orders → IF**

5) **Format order data**
   1. Add node: **Set**
   2. Add fields:
      - `id` = `{{$json.id}}`
      - `status` = `{{$json.status}}`
      - `customer_id` = `{{$json.customer_id}}`
      - `customer_name` = `{{$json.billing.first_name}} {{$json.billing.last_name}}`
      - `customer_note` = `{{$json.customer_note}}`
      - `date_created_gmt` = `{{$json.date_created_gmt}}`
      - `payment_url` = `{{$json.payment_url}}`
   3. Connect: **IF (true) → Format order data**

6) **Deduplicate customers**
   1. Add node: **Code**
   2. Paste logic equivalent to:
      - collect all items
      - keep first occurrence per `customer_id`
      - add `wc_domain` read from the “Set WooCommerce Domain” node
   3. Connect: **Format order data → Deduplicate Customers**

7) **Fetch customer order history**
   1. Add node: **HTTP Request**
   2. Method: GET
   3. URL: `https://{{ $json.wc_domain }}/wp-json/wc/v3/orders`
   4. Query parameters:
      - `customer` = `{{$json.customer_id}}`
      - `status` = `completed,processing`
      - (Recommended) `per_page` = `100` plus pagination handling if needed
   5. Same WooCommerce Basic Auth credential.
   6. Connect: **Deduplicate Customers → Fetch Customer Orders**

8) **Calculate VIP tier**
   1. Add node: **Code**
   2. Implement aggregation:
      - sum totals (completed only, total > 0)
      - count unique orders
      - assign tier: Platinum (>=20000), else Gold (>=5 orders), else Silver
      - filter out `customer_id == 0`
   3. Connect: **Fetch Customer Orders → Calculate VIP Tier**

9) **Check VIP eligibility**
   1. Add node: **IF**
   2. Current workflow condition is `tier != "none"` (but this will pass Silver too).
   3. Recommended alternatives (choose one):
      - `{{$json.vip_reason}}` is not empty
      - OR `{{$json.tier}}` is in `[Gold, Platinum]`
   4. Connect: **Calculate VIP Tier → Check VIP Eligibility**

10) **Store in Airtable**
   1. Add node: **Airtable**
   2. Credential: **Airtable Personal Access Token**
   3. Select Base and Table (create a table “VIP Customer” with fields matching):
      - `Id` (Number)
      - `Customer Name` (Single line text)
      - `VIP Tier` (Single line text)
      - `Vip reason` (Single line text)
      - `Lifetime value` (Number / currency)
      - `Total Orders` (Number)
   4. Operation: **Create**
   5. Map fields from JSON to Airtable columns.
   6. Connect: **Check VIP Eligibility (true) → Save VIP Customer**

   **Optional (recommended):** implement upsert:
   - Search record by `Id` first, then update if exists; create otherwise.

11) **Notify Slack**
   1. Add node: **Slack**
   2. Credential: Slack OAuth with `chat:write` and access to the target channel
   3. Operation: post message to channel
   4. Use text fields from Airtable output (`$json.fields[...]`) or from VIP calculation directly.
   5. Connect: **Save VIP Customer → Notify Team**

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “## I’m a note … This workflow automatically finds your important (VIP) customers… saves their information into Airtable and sends a Slack message…” | Sticky Note (overall purpose + explanation) |
| “## Setup Steps: Connect WooCommerce… Fetch Orders… Prepare Customer Data… Get Order History… Compute VIP Tier… Filter VIP Only… Send to Airtable & Slack…” | Sticky Note (embedded setup checklist) |
| Important implementation caveats: WooCommerce pagination not handled; Airtable node uses create (can duplicate); VIP eligibility IF condition currently allows Silver through. | Operational considerations derived from node logic |

