Match WooCommerce orders to new Zendesk tickets and send confirmation emails

https://n8nworkflows.xyz/workflows/match-woocommerce-orders-to-new-zendesk-tickets-and-send-confirmation-emails-12906


# Match WooCommerce orders to new Zendesk tickets and send confirmation emails

## 1. Workflow Overview

**Purpose:**  
This workflow automatically links **new Zendesk support tickets** to the most relevant **WooCommerce order** by matching customer emails. When a match is found, it enriches the Zendesk ticket with order details (internal note + tags). If the order is **completed**, it also sends a **confirmation email** to the customer via Gmail.

**Primary use cases:**
- Support teams handling order-related inquiries in Zendesk and needing quick access to WooCommerce order context.
- Reducing manual lookups and ensuring consistent tagging and customer confirmation messaging.

### 1.1 Ticket Intake (Zendesk)
Triggered on creation of a new ticket, then retrieves requester details (notably email).

### 1.2 Order Retrieval (WooCommerce)
Fetches a small set of recent WooCommerce orders for comparison.

### 1.3 Matching & Enrichment
Compares ticket requester email to WooCommerce billing email; if matched, generates status-based Zendesk tags and prepares a structured payload.

### 1.4 Ticket Update + Conditional Customer Email
Updates the Zendesk ticket with an internal comment and tags, then checks if the order is completed and sends a confirmation email if so.

---

## 2. Block-by-Block Analysis

### Block 1 — Receive new Zendesk ticket and fetch requester
**Overview:** Starts the workflow on a new Zendesk ticket and fetches the requester’s user record to obtain the email used later for matching.

**Nodes involved:**
- Zendesk – New Ticket Trigger
- Zendesk – Fetch Ticket Requester

#### Node: Zendesk – New Ticket Trigger
- **Type / role:** `zendeskTrigger` — entry point; listens for Zendesk ticket events.
- **Key configuration:**
  - Triggers when ticket condition matches **status = "new"** (as configured via conditions).
  - **Authentication:** Zendesk OAuth2.
- **Inputs/Outputs:**
  - **Input:** none (trigger).
  - **Output:** Ticket event payload including `ticket_id` and `requester.id` (used downstream).
- **Edge cases / failures:**
  - OAuth2 misconfiguration/expired tokens.
  - Trigger might fire for “new” tickets only; if your Zendesk setup uses different initial states, it may not trigger.
  - Payload fields may differ depending on Zendesk trigger event schema; missing `requester.id` would break the next node.

#### Node: Zendesk – Fetch Ticket Requester
- **Type / role:** `zendesk` — fetches a Zendesk user by ID.
- **Key configuration:**
  - **Resource:** User
  - **Operation:** Get
  - **User ID expression:** `{{ $json.requester.id }}`
  - **Authentication:** Zendesk OAuth2.
- **Inputs/Outputs:**
  - **Input:** Output of trigger (ticket data).
  - **Output:** Zendesk user object; workflow expects `.email` at `item.json.email`.
- **Edge cases / failures:**
  - Requester ID missing/undefined → expression resolves to empty → API error.
  - User record may not have an email (rare; or hidden/permissions).
  - Zendesk API rate limits / temporary failures.

**Sticky note (applies to this block):**  
“## Receive New Support Ticket … fetches the customer’s basic details … needed to find the related order in WooCommerce.”

---

### Block 2 — Fetch recent WooCommerce orders
**Overview:** Retrieves the latest WooCommerce orders and sends them one-by-one downstream for matching.

**Nodes involved:**
- WooCommerce – Fetch Recent Orders

#### Node: WooCommerce – Fetch Recent Orders
- **Type / role:** `wooCommerce` — pulls orders from WooCommerce.
- **Key configuration:**
  - **Resource:** Order
  - **Operation:** Get Many (`getAll`)
  - **Limit:** 5 (only the 5 most recent orders are returned)
- **Inputs/Outputs:**
  - **Input:** requester user object (not used directly by this node, but required for flow).
  - **Output:** A list of order items (each item includes `billing.email`, `status`, `number`, `currency`, `line_items`, etc.).
- **Edge cases / failures:**
  - If the customer’s order is not within the most recent 5 orders, it will not be found.
  - WooCommerce credentials / permissions issues.
  - Pagination is not used here; only a small subset is searched.
  - Stores with heavy volume may need more robust querying (by email) instead of scanning recent orders.

**Sticky note:**  
“## Get Orders from WooCommerce … retrieves recent orders … used to look for a match with the customer’s email address …”

---

### Block 3 — Match order by email
**Overview:** Filters WooCommerce orders by comparing each order’s billing email to the Zendesk requester email. Only matching orders continue.

**Nodes involved:**
- Match Customer Email (Zendesk vs Woo)

#### Node: Match Customer Email (Zendesk vs Woo)
- **Type / role:** `if` — conditional filter (acts like a gate).
- **Key configuration:**
  - Condition:  
    - Left: `{{ $('Zendesk – Fetch Ticket Requester').item.json.email }}`
    - Operator: equals (string)
    - Right: `{{ $json.billing.email }}`
  - Case sensitive: true (default shown in options)
- **Inputs/Outputs:**
  - **Input:** WooCommerce order items.
  - **True output:** Orders where emails match.
  - **False output:** Unmatched orders (not connected further, so dropped).
- **Edge cases / failures:**
  - Email case sensitivity can cause false negatives (e.g., `John@x.com` vs `john@x.com`).
  - Missing `billing.email` or requester email → comparison fails or evaluates false.
  - If multiple orders match (same email), all will pass and downstream actions may run multiple times (multiple ticket updates + multiple emails).

**Sticky note:**  
“## Match Customer Email … compares the customer’s email … Only orders with a matching email are allowed to continue …”

---

### Block 4 — Generate tags, prepare payload, and update Zendesk ticket
**Overview:** Creates status-based Zendesk tags, builds a clean payload of order details, and updates the Zendesk ticket with an internal comment plus tags.

**Nodes involved:**
- Generate Zendesk Tags from Order Status
- Prepare Ticket Update Payload
- Zendesk – Update Ticket with Order Details

#### Node: Generate Zendesk Tags from Order Status
- **Type / role:** `code` — transforms each matching order by adding a `zendesk_tags` array.
- **Key configuration (logic):**
  - Always adds base tags: `['woocommerce', 'order_found']`
  - Adds one status tag based on `item.json.status`:
    - `completed` → `order_completed`
    - `processing` → `order_processing`
    - `pending` → `order_pending`
    - `cancelled` → `order_cancelled`
    - `refunded` → `order_refunded`
    - else → `order_unknown_status`
  - Writes result to `item.json.zendesk_tags`
- **Inputs/Outputs:**
  - **Input:** Matched WooCommerce order(s).
  - **Output:** Same order items, enriched with `zendesk_tags`.
- **Edge cases / failures:**
  - Unexpected/missing `status` → falls into `order_unknown_status` (safe).
  - If multiple matched orders exist, tags will be generated for each and each may update the same ticket downstream.

#### Node: Prepare Ticket Update Payload
- **Type / role:** `set` — constructs a stable payload schema used for Zendesk update and email.
- **Key configuration (fields created):**
  - `ticket_id` (number): from trigger `{{ $('Zendesk – New Ticket Trigger').item.json.ticket_id }}`
  - `order_number`: `{{ $json.number }}`
  - `order_status`: `{{ $json.status }}`
  - `currency`: `{{ $json.currency }}`
  - `customer_email`: `{{ $json.billing.email }}`
  - `items`: `{{ $json.line_items.map(i => `${i.name} (x${i.quantity})`).join(', ') }}`
  - `zendesk_tags` (array): `{{ $json.zendesk_tags }}`
- **Inputs/Outputs:**
  - **Input:** Enriched order item(s) from Code node.
  - **Output:** New simplified JSON per matched order with the fields above.
- **Edge cases / failures:**
  - If `line_items` is missing/not an array, the `.map(...)` expression will error.
  - If `ticket_id` is missing from the trigger payload, Zendesk update will fail.
  - Multiple matches → multiple payloads → repeated ticket updates.

#### Node: Zendesk – Update Ticket with Order Details
- **Type / role:** `zendesk` — updates the Zendesk ticket.
- **Key configuration:**
  - **Operation:** Update ticket by ID `{{ $json.ticket_id }}`
  - **JSON parameters enabled**
  - Adds an **internal (non-public) comment** with order details, including:
    - Order number, status, currency, items, customer email
  - Updates tags with `{{ $json.zendesk_tags }}`
- **Inputs/Outputs:**
  - **Input:** Prepared payload from Set node.
  - **Output:** Zendesk API response (ticket updated).
- **Edge cases / failures:**
  - **Tags type mismatch risk:** Zendesk expects tags as an array of strings. The node uses `"tags": "{{ $json.zendesk_tags }}"` which may serialize as a string depending on templating; safer is to pass the array directly (see notes in reproduction section).
  - Zendesk permission issues (agent role restrictions).
  - Rate limiting / transient API failures.
  - If multiple matching orders exist, ticket may be updated multiple times with different order details.

**Sticky notes:**  
- “## Identify Order Status and Create Tags … creates clear Zendesk tags …”  
- “## Add Order Details to Ticket … adds important order details … internal note … applies tags …”

---

### Block 5 — Conditional email for completed orders
**Overview:** If the matched order status is `completed`, the workflow emails the customer confirming the order was found.

**Nodes involved:**
- Check Order Status = Completed
- Send Order Confirmation Email

#### Node: Check Order Status = Completed
- **Type / role:** `if` — branches based on order status.
- **Key configuration:**
  - Left: `{{ $('Prepare Ticket Update Payload').item.json.order_status }}`
  - Operator: equals
  - Right: `completed`
- **Inputs/Outputs:**
  - **Input:** Output of Zendesk update node.
  - **True output:** continues to send email.
  - **False output:** not connected; workflow ends.
- **Edge cases / failures:**
  - This node references data from `Prepare Ticket Update Payload` via node lookup; if multiple items are in flight, ensure item pairing behaves as expected.
  - If status values differ (e.g., `complete` vs `completed`), no email will be sent.

#### Node: Send Order Confirmation Email
- **Type / role:** `gmail` — sends outbound email through Gmail OAuth2.
- **Key configuration:**
  - **To:** `{{ $('Prepare Ticket Update Payload').item.json.customer_email }}`
  - **Subject:** `Your order #{{ ...order_number }} – details confirmed`
  - **Body:** Plain text message including order number/status/currency/items.
  - `appendAttribution: false`
- **Important data issue:**
  - The message uses `{{ $json.billing_first_name || 'there' }}`, but **no node sets `billing_first_name`** in the current payload or upstream flow. This will likely always fall back to `"there"` unless the Gmail node is actually receiving a JSON that contains that field (it won’t, given the current path).
- **Inputs/Outputs:**
  - **Input:** From the IF node (completed path), but uses node lookups for most fields.
  - **Output:** Gmail send result.
- **Edge cases / failures:**
  - OAuth2 token expiry / Gmail API not enabled.
  - Customer email missing/invalid.
  - Duplicate emails if multiple orders match and are completed.

**Sticky note:**  
“## Send Email for Completed Orders … checks … completed … sends a confirmation email …”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Zendesk – New Ticket Trigger | zendeskTrigger | Entry trigger on new Zendesk tickets | — | Zendesk – Fetch Ticket Requester | ## Receive New Support Ticket  This section starts the workflow whenever a new support ticket is created in Zendesk. It also fetches the customer’s basic details, such as their email address, which is needed to find the related order in WooCommerce. |
| Zendesk – Fetch Ticket Requester | zendesk | Fetch requester user record (email) | Zendesk – New Ticket Trigger | WooCommerce – Fetch Recent Orders | ## Receive New Support Ticket  This section starts the workflow whenever a new support ticket is created in Zendesk. It also fetches the customer’s basic details, such as their email address, which is needed to find the related order in WooCommerce. |
| WooCommerce – Fetch Recent Orders | wooCommerce | Retrieve recent orders for matching | Zendesk – Fetch Ticket Requester | Match Customer Email (Zendesk vs Woo) | ## Get Orders from WooCommerce  This section retrieves recent orders from WooCommerce. These orders are used to look for a match with the customer’s email address so the correct order can be linked to the support ticket. |
| Match Customer Email (Zendesk vs Woo) | if | Filter orders by matching emails | WooCommerce – Fetch Recent Orders | Generate Zendesk Tags from Order Status (true path) | ## Match Customer Email  This section compares the customer’s email from the Zendesk ticket with the billing email on WooCommerce orders. Only orders with a matching email are allowed to continue in the workflow. |
| Generate Zendesk Tags from Order Status | code | Add status-based Zendesk tags to matched order | Match Customer Email (Zendesk vs Woo) | Prepare Ticket Update Payload | ## Identify Order Status and Create Tags  This section checks the order status, such as completed, processing or cancelled. Based on the status, it creates clear Zendesk tags that help support agents quickly understand the order situation. |
| Prepare Ticket Update Payload | set | Normalize data for ticket update + email | Generate Zendesk Tags from Order Status | Zendesk – Update Ticket with Order Details | ## Add Order Details to Ticket  This section adds important order details to the Zendesk ticket as an internal note. It also applies order-related tags, giving support agents all necessary information in one place. |
| Zendesk – Update Ticket with Order Details | zendesk | Update ticket with internal note + tags | Prepare Ticket Update Payload | Check Order Status = Completed | ## Add Order Details to Ticket  This section adds important order details to the Zendesk ticket as an internal note. It also applies order-related tags, giving support agents all necessary information in one place. |
| Check Order Status = Completed | if | Branch only if order is completed | Zendesk – Update Ticket with Order Details | Send Order Confirmation Email (true path) | ## Send Email for Completed Orders  This section checks whether the customer’s order is marked as completed. If the condition is met, the workflow automatically sends a confirmation email with order details, assuring the customer that their order was found and their support request is being handled. |
| Send Order Confirmation Email | gmail | Send confirmation email to customer | Check Order Status = Completed | — | ## Send Email for Completed Orders  This section checks whether the customer’s order is marked as completed. If the condition is met, the workflow automatically sends a confirmation email with order details, assuring the customer that their order was found and their support request is being handled. |
| Sticky Note | stickyNote | Comment block (documentation) | — | — | ## Receive New Support Ticket This section starts the workflow whenever a new support ticket is created in Zendesk. It also fetches the customer’s basic details, such as their email address, which is needed to find the related order in WooCommerce. |
| Sticky Note1 | stickyNote | Comment block (documentation) | — | — | ## Get Orders from WooCommerce This section retrieves recent orders from WooCommerce. These orders are used to look for a match with the customer’s email address so the correct order can be linked to the support ticket. |
| Sticky Note2 | stickyNote | Comment block (documentation) | — | — | ## Match Customer Email This section compares the customer’s email from the Zendesk ticket with the billing email on WooCommerce orders. Only orders with a matching email are allowed to continue in the workflow. |
| Sticky Note3 | stickyNote | Comment block (documentation) | — | — | ## Identify Order Status and Create Tags This section checks the order status, such as completed, processing or cancelled. Based on the status, it creates clear Zendesk tags that help support agents quickly understand the order situation. |
| Sticky Note4 | stickyNote | Comment block (documentation) | — | — | ## Add Order Details to Ticket This section adds important order details to the Zendesk ticket as an internal note. It also applies order-related tags, giving support agents all necessary information in one place. |
| Sticky Note5 | stickyNote | Comment block (documentation) | — | — | ## Send Email for Completed Orders This section checks whether the customer’s order is marked as completed. If the condition is met, the workflow automatically sends a confirmation email with order details, assuring the customer that their order was found and their support request is being handled. |
| Sticky Note6 | stickyNote | Comment block (documentation) | — | — | ##  How Workflow Works … (full note content in section 5) |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Set workflow name (optional): *Zendesk–WooCommerce Order Matching, Ticket Enrichment & Conditional Email Workflow*
- Execution order: default (`v1`) is fine.

2) **Add “Zendesk – New Ticket Trigger”**
- Node type: **Zendesk Trigger**
- Authentication: **OAuth2**
- Trigger condition: Ticket status equals **new**
- Credentials:
  - Create/attach **Zendesk OAuth2** credentials (subdomain, client/app setup as required by n8n’s Zendesk OAuth2).
- This is the **entry node**.

3) **Add “Zendesk – Fetch Ticket Requester”**
- Node type: **Zendesk**
- Resource: **User**
- Operation: **Get**
- ID: expression referencing trigger output: `{{$json.requester.id}}`
- Authentication: OAuth2 (same credentials as above)
- **Connect:** Zendesk Trigger → Fetch Ticket Requester

4) **Add “WooCommerce – Fetch Recent Orders”**
- Node type: **WooCommerce**
- Resource: **Order**
- Operation: **Get Many (getAll)**
- Limit: **5**
- Credentials:
  - Configure **WooCommerce API** credentials (typically consumer key/secret; ensure read permissions).
- **Connect:** Fetch Ticket Requester → Fetch Recent Orders

5) **Add “Match Customer Email (Zendesk vs Woo)”**
- Node type: **IF**
- Condition (String equals):
  - Left: `{{ $('Zendesk – Fetch Ticket Requester').item.json.email }}`
  - Right: `{{ $json.billing.email }}`
- **Connect:** Fetch Recent Orders → IF
- Use the **true** output path only (false path can remain unconnected).

   *Recommended adjustment:* make the comparison case-insensitive by comparing lowercased values:
   - Left: `{{ $('Zendesk – Fetch Ticket Requester').item.json.email.toLowerCase() }}`
   - Right: `{{ $json.billing.email.toLowerCase() }}`

6) **Add “Generate Zendesk Tags from Order Status”**
- Node type: **Code**
- Paste logic that:
  - Creates `zendesk_tags` array with base tags + one status tag
- **Connect:** IF (true) → Code

7) **Add “Prepare Ticket Update Payload”**
- Node type: **Set**
- Add fields:
  - `ticket_id` (Number): `{{ $('Zendesk – New Ticket Trigger').item.json.ticket_id }}`
  - `order_number` (String): `{{ $json.number }}`
  - `order_status` (String): `{{ $json.status }}`
  - `currency` (String): `{{ $json.currency }}`
  - `customer_email` (String): `{{ $json.billing.email }}`
  - `items` (String): `{{ $json.line_items.map(i => `${i.name} (x${i.quantity})`).join(', ') }}`
  - `zendesk_tags` (Array): `{{ $json.zendesk_tags }}`
- **Connect:** Code → Set

   *Recommended hardening:* guard `line_items`:
   - `{{ Array.isArray($json.line_items) ? $json.line_items.map(i => `${i.name} (x${i.quantity})`).join(', ') : '' }}`

8) **Add “Zendesk – Update Ticket with Order Details”**
- Node type: **Zendesk**
- Operation: **Update** (Ticket)
- Ticket ID: `{{ $json.ticket_id }}`
- Enable **JSON parameters**
- Provide update JSON that:
  - Adds a **private** comment containing order details
  - Updates tags

  *Recommended correction for tags:* ensure `tags` is passed as an array, not a string. Use an expression that returns JSON, e.g. set the field in a way n8n treats it as array (depends on node UI). Conceptually:
  - `tags`: `{{$json.zendesk_tags}}`

- **Connect:** Set → Zendesk Update

9) **Add “Check Order Status = Completed”**
- Node type: **IF**
- Condition: `{{ $('Prepare Ticket Update Payload').item.json.order_status }}` equals `completed`
- **Connect:** Zendesk Update → IF

10) **Add “Send Order Confirmation Email”**
- Node type: **Gmail**
- Operation: **Send**
- Credentials: **Gmail OAuth2** (Google Cloud project OAuth consent + Gmail API enabled)
- To: `{{ $('Prepare Ticket Update Payload').item.json.customer_email }}`
- Subject: `Your order #{{ $('Prepare Ticket Update Payload').item.json.order_number }} – details confirmed`
- Body: include order number/status/currency/items using the same node lookup pattern.
- **Connect:** IF (true path) → Gmail

   *Recommended fix for first name:* either remove `billing_first_name` usage or add it in the Set node (e.g., from Woo order billing fields if available).

11) **Add sticky notes (optional but matches the provided workflow)**
- Add sticky notes describing each block (content as in section 5).

12) **Activate workflow after testing**
- Run a manual test by creating a Zendesk ticket from an email that matches a recent WooCommerce order’s billing email.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| **How Workflow Works**: Links Zendesk tickets with WooCommerce orders; fetch requester email; searches for matching recent order; captures order details; adds tags based on status; updates ticket with a private note; sends confirmation email if completed; saves time and ensures consistent communication. | Sticky note “## How Workflow Works” (overview and setup steps) |
| **Workflow Setup Steps** (as written in the canvas note): 1) Zendesk trigger on new ticket 2) Fetch requester email 3) Retrieve recent WooCommerce orders 4) Match emails 5) Extract order details 6) Add order-based tags 7) Update ticket with private comment 8) If completed send email 9) Activate workflow. | Sticky note “## Workflow Setup Steps” |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.