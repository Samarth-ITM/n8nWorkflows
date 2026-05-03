Detect WooCommerce order delays with Gmail and Slack alerts in real time

https://n8nworkflows.xyz/workflows/detect-woocommerce-order-delays-with-gmail-and-slack-alerts-in-real-time-12904


# Detect WooCommerce order delays with Gmail and Slack alerts in real time

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Detect WooCommerce order delays with Gmail and Slack alerts in real time  
**Workflow name (in JSON):** Order Delay Notification Flow  
**Purpose:** Listen to WooCommerce order events, normalize key order fields, validate eligibility for delay monitoring, compute delivery delay in days, and—if delayed—notify the customer by Gmail and alert an internal Slack channel.

### 1.1 Trigger & Order Intake
- Entry via WooCommerce Trigger (webhook-based).
- Normalizes incoming WooCommerce payload into a stable schema used by downstream nodes.

### 1.2 Order Eligibility & Validation
- Validates that the order status is eligible for delay tracking (as implemented: **only `processing`**).
- Ensures an estimated delivery date exists before calculating delay.

### 1.3 Delay Calculation & Decision
- Computes delay (in full days) by comparing current time to ETA.
- Continues only if delayDays > 0.

### 1.4 Notifications & Visibility
- Sends a customer email via Gmail.
- Posts an internal alert to Slack.

---

## 2. Block-by-Block Analysis

### Block 1 — Trigger & Order Intake

**Overview:** Receives WooCommerce order events and reshapes the payload into a minimal set of fields (IDs, status, customer details, ETA) for consistent downstream logic.

**Nodes involved:**
- WooCommerce Trigger
- Normalize Order

#### Node: WooCommerce Trigger
- **Type / role:** `wooCommerceTrigger` — webhook trigger for WooCommerce events.
- **Configuration (interpreted):**
  - Event: `order.created` (fires when an order is created).
  - Uses an n8n webhook internally (webhookId present).
- **Inputs/Outputs:**
  - Input: none (trigger).
  - Output: order payload from WooCommerce to **Normalize Order**.
- **Version notes:** TypeVersion 1.
- **Edge cases / failures:**
  - WooCommerce webhook not configured or failing signature/auth (depending on WooCommerce setup).
  - Event mismatch: sticky notes mention “order update”, but node listens to **order.created** only. Delay detection for later status changes will not occur unless event is adjusted.
  - Payload shape differences across WooCommerce versions/plugins (e.g., missing `meta_data`, missing `billing`).

#### Node: Normalize Order
- **Type / role:** `set` — creates a normalized schema.
- **Configuration (interpreted):**
  - Creates fields:
    - `orderId` = `$json.id` (number)
    - `orderNumber` = `$json.number` (number)
    - `status` = `$json.status` (string)
    - `email` = `$json.billing.email` (string)
    - `customerName` = `{{$json.billing.first_name}} {{$json.billing.last_name}}`
    - `estimatedDelivery` = `$json.meta_data.find(m => m.key === '_estimated_delivery')?.value`
    - `updatedAt` = `$json.date_modified`
  - Keeps default options (no explicit “keep only set fields” indicated; in recent n8n Set node it typically *adds/overwrites* these fields while keeping the rest unless configured otherwise).
- **Key expressions/variables:**
  - Uses JS `.find()` on `meta_data` with optional chaining `?.value`.
- **Inputs/Outputs:**
  - Input: WooCommerce Trigger payload.
  - Output: normalized item to **Processing**.
- **Version notes:** TypeVersion 3.4 (newer Set node model with “assignments”).
- **Edge cases / failures:**
  - `meta_data` may be missing or not an array ⇒ `.find` would throw if `meta_data` is undefined. (Optional chaining is only applied to `.value`, not to `meta_data` itself.)
    - Safer pattern: `($json.meta_data ?? []).find(...)?.value`
  - `billing` or `billing.email` may be missing ⇒ expressions may evaluate to `undefined`.
  - `_estimated_delivery` key name is store-specific; may differ by plugin.

---

### Block 2 — Order Eligibility & Validation

**Overview:** Filters orders by status and ensures ETA exists before computing delay. Orders failing checks stop because the “false” branches are not connected.

**Nodes involved:**
- Processing
- Fetch ETA
- ETA Exists?

#### Node: Processing
- **Type / role:** `if` — status eligibility gate.
- **Configuration (interpreted):**
  - Condition: `{{$json.status}} equals "processing"`
  - Case sensitivity: enabled (caseSensitive true / ignoreCase false).
- **Inputs/Outputs:**
  - Input: Normalize Order.
  - True output → **Fetch ETA**
  - False output: unconnected (workflow ends for ineligible statuses).
- **Version notes:** TypeVersion 2.2.
- **Edge cases / failures:**
  - Sticky note claims OR logic across statuses (processing/on-hold/shipped), but workflow implements **only one** check. Orders in `on-hold` or `shipped` will be ignored.
  - If WooCommerce sends status with different casing or prefixes, strict equality will fail.

#### Node: Fetch ETA
- **Type / role:** `set` — prepares time variables for delay computation.
- **Configuration (interpreted):**
  - `now` = `{{$now}}` (string representation of current time in n8n)
  - `eta` = `{{ new Date($json.estimatedDelivery) }}` (stored as a string because field type is string)
- **Inputs/Outputs:**
  - Input: Processing (true path).
  - Output: to **ETA Exists?**
- **Version notes:** TypeVersion 3.4.
- **Edge cases / failures:**
  - `estimatedDelivery` may be empty/invalid. `new Date(invalid)` produces `Invalid Date`.
  - Storing a Date object as string can lead to inconsistent formats (environment-dependent stringification).
  - Timezone ambiguity: `estimatedDelivery` parsing depends on format (ISO recommended).

#### Node: ETA Exists?
- **Type / role:** `if` — checks if ETA field exists/non-empty before calculating delay.
- **Configuration (interpreted):**
  - Condition: `{{$json.eta}} notEmpty`
- **Inputs/Outputs:**
  - Input: Fetch ETA.
  - True output → **Calculate Delivery Delay**
  - False output: unconnected (workflow stops).
- **Version notes:** TypeVersion 2.2.
- **Edge cases / failures:**
  - `eta` will often be a non-empty string even if it represents `Invalid Date` (e.g., `"Invalid Date"`), so this check may pass while delay calculation later fails or yields NaN.

---

### Block 3 — Delay Calculation & Decision

**Overview:** Calculates the number of delayed days and proceeds only if delayDays is greater than 0 (i.e., at least 1 full day late).

**Nodes involved:**
- Calculate Delivery Delay
- Validate Delivery Delay

#### Node: Calculate Delivery Delay
- **Type / role:** `code` — computes delayDays.
- **Configuration (interpreted):**
  - Reads first input item as `item`.
  - `now = new Date(item.now)`
  - `eta = new Date(JSON.parse(item.eta))`
  - `diffMs = now - eta`
  - `delayDays = Math.floor(diffMs / (1000*60*60*24))`
  - Returns item with appended `delayDays`.
- **Inputs/Outputs:**
  - Input: ETA Exists? (true path).
  - Output: to **Validate Delivery Delay**.
- **Version notes:** TypeVersion 2.
- **Edge cases / failures (important):**
  - **Likely runtime bug:** `item.eta` was produced as `new Date(...)` then stored as a string; it is not JSON. `JSON.parse(item.eta)` will usually throw (e.g., parsing `"Mon Feb 10 2026..."` fails).
    - Recommended: `const eta = new Date(item.eta);` (no JSON.parse), or store ETA as ISO string `new Date(...).toISOString()` and parse with `new Date(item.eta)`.
  - If `eta` is invalid, `now - eta` becomes `NaN`, `delayDays` becomes `NaN`, and the IF node may behave unexpectedly.
  - Using `Math.floor` means delays < 1 day are treated as 0.

#### Node: Validate Delivery Delay
- **Type / role:** `if` — gate: continue only for delayed orders.
- **Configuration (interpreted):**
  - Condition: `{{$json.delayDays}} > 0`
- **Inputs/Outputs:**
  - Input: Calculate Delivery Delay.
  - True output → **Email Customer**
  - False output: unconnected (no notifications).
- **Version notes:** TypeVersion 2.2.
- **Edge cases / failures:**
  - If `delayDays` is `NaN`, numeric comparison fails; typically evaluates to false → no alerts even if late.

---

### Block 4 — Notifications & Visibility

**Overview:** Sends a customer-facing Gmail message and then posts an internal Slack alert with delay context.

**Nodes involved:**
- Email Customer
- Delay Alert

#### Node: Email Customer
- **Type / role:** `gmail` — sends customer email.
- **Configuration (interpreted):**
  - Subject: “Update on your order”
  - Body (text): personalized message using customer name and delay days.
  - **Recipient is hard-coded:** `user@example.com` (not using normalized `email`).
- **Key expressions/variables:**
  - `{{ $('Normalize Order').item.json.customerName }}`
  - `{{$json.delayDays}}` (from Validate Delivery Delay input item)
- **Inputs/Outputs:**
  - Input: Validate Delivery Delay (true path).
  - Output: to **Delay Alert**.
- **Version notes:** TypeVersion 2.1.
- **Edge cases / failures:**
  - OAuth2 credential missing/expired; Gmail API scopes not granted.
  - Hard-coded recipient means real customers will not be emailed unless changed to `{{$json.email}}` (or reference Normalize Order output).
  - Rate limits / daily send limits.

#### Node: Delay Alert
- **Type / role:** `slack` — posts an internal Slack message.
- **Configuration (interpreted):**
  - Posts text to a selected channel (channelId is present but value empty in JSON export).
  - Message includes order number, customer name, delay days.
  - **Bug in message template:** “Estimated Delivery” line prints `delayDays` again, not the ETA/date.
- **Key expressions/variables:**
  - Order #: `{{ $('Normalize Order').item.json.orderNumber }}`
  - Customer: `{{ $('Normalize Order').item.json.customerName }}`
  - Delay: `{{ $('Validate Delivery Delay').item.json.delayDays }}`
  - Estimated Delivery: `{{ $('Validate Delivery Delay').item.json.delayDays }}` (should likely be `$('Normalize Order').item.json.estimatedDelivery` or computed `eta`)
- **Inputs/Outputs:**
  - Input: Email Customer.
  - Output: none (end).
- **Version notes:** TypeVersion 2.3.
- **Edge cases / failures:**
  - Slack credential missing/invalid; channel not selected.
  - If channelId remains empty, node may fail validation or post nowhere depending on n8n version/app settings.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | Sticky Note | Documentation: Trigger & intake block |  |  | ## Trigger & Order Intake; starts with WooCommerce webhook; normalize required fields; real-time automation. |
| Sticky Note1 | Sticky Note | Documentation: eligibility & validation |  |  | ## Order Eligibility & Validation; intended statuses processing/on-hold/shipped; chained IF for OR; checks ETA exists; stop early if invalid. |
| Sticky Note2 | Sticky Note | Documentation: delay calc & decision |  |  | ## Delay Calculation & Decision; compute delayDays; continue only if >=1. |
| Sticky Note3 | Sticky Note | Documentation: notifications |  |  | ## Notifications & Visibility; email customer + Slack alert. |
| Sticky Note4 | Sticky Note | Documentation: overall explanation + setup steps |  |  | ## How it Works + Setup Steps (1–9) describing trigger, normalization, validation, calculation, and notifications. |
| WooCommerce Trigger | WooCommerce Trigger | Entry point: receives WooCommerce order event |  | Normalize Order | ## Trigger & Order Intake; starts with WooCommerce webhook; normalize required fields; real-time automation. |
| Normalize Order | Set | Normalize payload fields for consistent processing | WooCommerce Trigger | Processing | ## Trigger & Order Intake; starts with WooCommerce webhook; normalize required fields; real-time automation. |
| Processing | IF | Filter eligible order statuses (implemented: processing only) | Normalize Order | Fetch ETA (true) | ## Order Eligibility & Validation; intended statuses processing/on-hold/shipped; chained IF for OR; checks ETA exists; stop early if invalid. |
| Fetch ETA | Set | Add `now` and `eta` fields for delay calculation | Processing (true) | ETA Exists? | ## Order Eligibility & Validation; intended statuses processing/on-hold/shipped; chained IF for OR; checks ETA exists; stop early if invalid. |
| ETA Exists? | IF | Ensure ETA is present before calculation | Fetch ETA | Calculate Delivery Delay (true) | ## Order Eligibility & Validation; intended statuses processing/on-hold/shipped; chained IF for OR; checks ETA exists; stop early if invalid. |
| Calculate Delivery Delay | Code | Compute delayDays from now vs ETA | ETA Exists? (true) | Validate Delivery Delay | ## Delay Calculation & Decision; compute delayDays; continue only if >=1. |
| Validate Delivery Delay | IF | Continue only if delayDays > 0 | Calculate Delivery Delay | Email Customer (true) | ## Delay Calculation & Decision; compute delayDays; continue only if >=1. |
| Email Customer | Gmail | Customer notification email | Validate Delivery Delay (true) | Delay Alert | ## Notifications & Visibility; email customer + Slack alert. |
| Delay Alert | Slack | Internal Slack alert with order details | Email Customer |  | ## Notifications & Visibility; email customer + Slack alert. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Name it: *Order Delay Notification Flow* (or your preferred name).

2) **Add Trigger: WooCommerce Trigger**
- Node type: **WooCommerce Trigger**
- Event: **order.created** (or change to an “order.updated” / “order.updated status” equivalent if you want true “status changes” behavior).
- Credentials:
  - Configure WooCommerce API credentials in n8n (URL, Consumer Key, Consumer Secret) as required by the trigger node.
  - Ensure the webhook can be registered/reached from WooCommerce (public URL, correct SSL).

3) **Add Set node: “Normalize Order”**
- Node type: **Set**
- Add fields (expressions):
  - `orderId` → `{{$json.id}}`
  - `orderNumber` → `{{$json.number}}`
  - `status` → `{{$json.status}}`
  - `email` → `{{$json.billing.email}}`
  - `customerName` → `{{$json.billing.first_name}} {{$json.billing.last_name}}`
  - `estimatedDelivery` → `{{$json.meta_data.find(m => m.key === '_estimated_delivery')?.value}}`
  - `updatedAt` → `{{$json.date_modified}}`
- Connect: **WooCommerce Trigger → Normalize Order**

4) **Add IF node: “Processing”**
- Node type: **IF**
- Condition (String):
  - Left: `{{$json.status}}`
  - Operation: **equals**
  - Right: `processing`
- Connect: **Normalize Order → Processing**
- Note: If you need OR logic for `on-hold` and `shipped`, either:
  - Use one IF with multiple conditions and combinator “OR” (if available in your n8n), or
  - Chain multiple IF nodes and merge, or
  - Use a Code node to check inclusion (recommended for clarity).

5) **Add Set node: “Fetch ETA”**
- Node type: **Set**
- Fields:
  - `now` → `{{$now}}`
  - `eta` → `{{new Date($json.estimatedDelivery)}}`
- Connect: **Processing (true) → Fetch ETA**

6) **Add IF node: “ETA Exists?”**
- Node type: **IF**
- Condition (String):
  - Left: `{{$json.eta}}`
  - Operation: **notEmpty**
- Connect: **Fetch ETA → ETA Exists?**

7) **Add Code node: “Calculate Delivery Delay”**
- Node type: **Code**
- Paste logic (as in workflow), but to make it work reliably, configure it like this (recommended adjustment):
  - Parse ETA with `new Date(item.eta)` and ensure ISO strings:
    - In Fetch ETA, set `eta` to `{{new Date($json.estimatedDelivery).toISOString()}}`
    - In Code: `const eta = new Date(item.eta);`
- Connect: **ETA Exists? (true) → Calculate Delivery Delay**

8) **Add IF node: “Validate Delivery Delay”**
- Node type: **IF**
- Condition (Number):
  - Left: `{{$json.delayDays}}`
  - Operation: **gt**
  - Right: `0`
- Connect: **Calculate Delivery Delay → Validate Delivery Delay**

9) **Add Gmail node: “Email Customer”**
- Node type: **Gmail**
- Operation: **Send**
- Credentials:
  - Configure Gmail OAuth2 credentials in n8n (Google Cloud project, OAuth consent, redirect URL, scopes).
- Parameters:
  - **To:** currently hard-coded as `user@example.com`. To send to the actual customer, set:
    - `{{$json.email}}` (or `{{ $('Normalize Order').item.json.email }}`)
  - Subject: `Update on your order`
  - Body (Text):
    - `Hi {{ $('Normalize Order').item.json.customerName }}, Your order is delayed by {{$json.delayDays}} day(s)...`
- Connect: **Validate Delivery Delay (true) → Email Customer**

10) **Add Slack node: “Delay Alert”**
- Node type: **Slack**
- Operation: **Post message**
- Credentials:
  - Configure Slack OAuth2/app token credentials in n8n with permission to post to the target channel.
- Parameters:
  - Select: **channel**
  - Choose **Channel**
  - Text: include order number, customer name, delayDays (and ideally ETA date).
- Connect: **Email Customer → Delay Alert**

11) **(Optional but recommended) Add failure handling**
- Add error workflows or an “Error Trigger” workflow to capture Gmail/Slack failures.
- Add a branch/logging node for non-eligible statuses and missing ETA for observability.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Sticky notes describe “Order Update Webhook” and OR logic across `processing`, `on-hold`, `shipped`, but the implemented trigger is `order.created` and only the `processing` status is accepted. | Align the WooCommerce event and status validation with the stated goal if you want real-time delay monitoring across status changes. |
| Delay calculation currently uses `JSON.parse(item.eta)` which is likely to throw because `eta` is not valid JSON. | Fix by storing ETA as ISO string and parsing via `new Date(item.eta)`. |
| Gmail “sendTo” is hard-coded (`user@example.com`) instead of using the customer’s email from the order. | Change to `{{$json.email}}` (or reference Normalize Order output) for live usage. |
| Slack message “Estimated Delivery” prints delayDays again, not a date. | Replace with the actual ETA field (e.g., `{{ $('Normalize Order').item.json.estimatedDelivery }}` or computed ISO `eta`). |