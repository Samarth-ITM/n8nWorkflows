Segment retail customers by purchase behavior with CRM and Google Sheets

https://n8nworkflows.xyz/workflows/segment-retail-customers-by-purchase-behavior-with-crm-and-google-sheets-12870


# Segment retail customers by purchase behavior with CRM and Google Sheets

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow name:** (Retail) Auto-Segment Customers by Purchase Behavior  
**Purpose:** Receive customer/order events (e.g., from Shopify/WooCommerce/CRM), normalize the payload, classify customers into purchase-behavior segments (VIP/new/repeat/inactive), sync the segment tag to a CRM via HTTP API, and log each segmentation event to Google Sheets.

### 1.1 Event Intake & Data Normalization
- Entry via **Webhook**.
- Convert raw incoming payload into a consistent schema for downstream rules (customer stats, order details, source/event type).

### 1.2 Validation & Segmentation Rules
- Validate the event has meaningful purchase stats (order_count not 0).
- Determine segment using a Switch with rule-based outputs:
  - VIP: lifetime spend > 1000
  - New: order_count = 1
  - Repeat: order_count > 1
  - Inactive: last order older than 60 days

### 1.3 Output Normalization, CRM Sync, and Logging
- Produce a clean “segmentation event” object (customer_id, email, segment_tag, source, event_time).
- POST the segment tag to a CRM endpoint (placeholder).
- Append/update a row in Google Sheets for auditing/reporting.

---

## 2. Block-by-Block Analysis

### Block 1 — Event Intake & Data Normalization

**Overview:** Receives external events and reshapes them into a stable internal format used by all segmentation logic.

**Nodes involved:**
- Receive Customer Order or Update Event
- Normalize Customer Purchase Data

#### Node: Receive Customer Order or Update Event
- **Type / role:** Webhook trigger (`n8n-nodes-base.webhook`) — entry point.
- **Configuration (interpreted):**
  - Method: `POST`
  - Path: `customer-segmentation` (your source system must POST to this endpoint)
- **Inputs/Outputs:**
  - Input: external HTTP request body.
  - Output: one item containing `$json.body` (expected).
- **Version notes:** TypeVersion 2.
- **Edge cases / failures:**
  - If the source payload doesn’t include `body` with the expected nested fields, downstream expressions will error.
  - If the webhook is called with non-JSON or wrong content-type, fields may not parse as expected.

#### Node: Normalize Customer Purchase Data
- **Type / role:** Set node (`n8n-nodes-base.set`) — schema normalization.
- **Configuration (interpreted):** Creates standardized fields from the incoming payload:
  - `customer_id` ← `$json.body.customer.customer_id`
  - `email`, `first_name`, `last_name`
  - `order_id`, `order_value` (number), `order_date`
  - `order_count` (number) ← `$json.body.customer_stats.order_count`
  - `lifetime_spend` (number) ← `$json.body.customer_stats.lifetime_spend`
  - `last_order_date` ← `$json.body.customer_stats.last_order_date`
  - `product_categories` (array) ← `$json.body.order.product_categories`
  - `source_system` ← `$json.body.source`
  - `event_type` ← `$json.body.event_type`
- **Inputs/Outputs:**
  - Input: Webhook output.
  - Output: normalized item used by validation and segmentation.
- **Version notes:** TypeVersion 3.
- **Edge cases / failures:**
  - Missing nested properties can cause expression evaluation issues.
  - `order_value`, `order_count`, `lifetime_spend` must be numeric-compatible or may become `null`/fail type conversion.
  - `product_categories` must be an array; otherwise downstream usage may break if assumed array.

**Sticky note coverage for this block:**
- “## Event Intake & Data Normalization … standardizes here for reliable downstream processing.”

---

### Block 2 — Validation & Segmentation Rules

**Overview:** Ensures the event includes purchase activity, then routes the item through rule branches that assign a segment tag.

**Nodes involved:**
- Validate Customer Event
- Determine Customer Segment
- Assign VIP Customer Segment
- Assign New Customer Segment
- Assign Repeat Customer Segment
- Assign Inactive Customer Segment

#### Node: Validate Customer Event
- **Type / role:** IF node (`n8n-nodes-base.if`) — gatekeeping condition.
- **Configuration (interpreted):**
  - Condition: `body.customer_stats.order_count != 0`
  - Uses strict type validation.
- **Inputs/Outputs:**
  - Input: normalized item from “Normalize Customer Purchase Data” (though the IF condition references original `$json.body...`).
  - Output:
    - **True** → Determine Customer Segment
    - **False** → no connection (processing stops for invalid/irrelevant events)
- **Version notes:** TypeVersion 2.2.
- **Edge cases / failures:**
  - If `$json.body.customer_stats.order_count` is missing, this may evaluate to `undefined` and fail strict number comparison.
  - Logical inconsistency: the workflow normalizes `order_count` but this IF checks the *non-normalized* path (`$json.body...`). If upstream structure changes, validation may fail even if normalized fields exist.

#### Node: Determine Customer Segment
- **Type / role:** Switch node (`n8n-nodes-base.switch`) — rule router with named outputs.
- **Configuration (interpreted):** 4 rules, each with “renameOutput”:
  1. **vip_customer** if `lifetime_spend > 1000`  
     - Left: `$('Normalize Customer Purchase Data').item.json.lifetime_spend`
  2. **new_customer** if `order_count == 1`
  3. **repeat_customer** if `order_count > 1`
  4. **inactive_customer** if last order is older than 60 days  
     - Left: `new Date($json.last_order_date).getTime()`
     - Right: `Date.now() - (60 * 24 * 60 * 60 * 1000)`
- **Inputs/Outputs:**
  - Input: True output of Validate Customer Event.
  - Outputs:
    - Output 0 → Assign VIP Customer Segment
    - Output 1 → Assign New Customer Segment
    - Output 2 → Assign Repeat Customer Segment
    - Output 3 → Assign Inactive Customer Segment
- **Version notes:** TypeVersion 3.3.
- **Edge cases / failures:**
  - Rule precedence matters: if multiple rules could match, Switch behavior depends on node settings (in many n8n configs it routes to the first matching rule). Here VIP is first, so a VIP with `order_count > 1` will likely be tagged VIP.
  - The “inactive_customer” rule references `$json.last_order_date` (current item) while earlier rules reference normalized node output explicitly. If the current item does not contain `last_order_date` (e.g., structure changes), date parsing may return `NaN`.
  - `last_order_date` must be parseable by `new Date(...)`.
  - Time window is fixed at 60 days; no timezone normalization is applied.

#### Node: Assign VIP Customer Segment
- **Type / role:** Set node — assigns `segment_tag = "vip_customer"`.
- **Inputs/Outputs:** Input from Switch “vip_customer” output; output to “Normalize Customer Segment Output”.
- **Version notes:** TypeVersion 3.4.
- **Edge cases:** None beyond upstream routing.

#### Node: Assign New Customer Segment
- **Type / role:** Set node — assigns `segment_tag = "new_customer"`.
- **Inputs/Outputs:** Input from Switch “new_customer”; output to “Normalize Customer Segment Output”.
- **Version notes:** TypeVersion 3.4.

#### Node: Assign Repeat Customer Segment
- **Type / role:** Set node — assigns `segment_tag = "repeat_customer"`.
- **Inputs/Outputs:** Input from Switch “repeat_customer”; output to “Normalize Customer Segment Output”.
- **Version notes:** TypeVersion 3.4.

#### Node: Assign Inactive Customer Segment
- **Type / role:** Set node — assigns `segment_tag = "inactive_customer"`.
- **Inputs/Outputs:** Input from Switch “inactive_customer”; output to “Normalize Customer Segment Output”.
- **Version notes:** TypeVersion 3.4.

**Sticky note coverage for this block:**
- “## Customer Segmentation Rules … classified as VIP, new, repeat, or inactive…”

---

### Block 3 — Output Normalization, CRM Sync, and Logging

**Overview:** Consolidates the chosen segment with customer identifiers, then syncs tags to a CRM and logs the event to Google Sheets.

**Nodes involved:**
- Normalize Customer Segment Output
- Sync Customer Segment to CRM
- Append or update row in sheet

#### Node: Normalize Customer Segment Output
- **Type / role:** Set node — creates a clean “segmentation event” payload.
- **Configuration (interpreted):**
  - `customer_id` from normalized purchase data
  - `email` from normalized purchase data
  - `segment_tag` from the current branch output (`$json.segment_tag`)
  - `source` from normalized purchase data (`source_system`)
  - `event_time` = `$now`
- **Inputs/Outputs:**
  - Input: any of the segment assignment nodes.
  - Output: Sync Customer Segment to CRM.
- **Version notes:** TypeVersion 3.4.
- **Edge cases / failures:**
  - Uses `$('Normalize Customer Purchase Data').item.json...` which assumes that node is in the same execution context and item pairing is consistent. If items are split/merged in future edits, this can mis-reference.
  - `$now` is runtime timestamp; format is n8n default (ISO-like string). If you need a specific format, it should be explicitly formatted.

#### Node: Sync Customer Segment to CRM
- **Type / role:** HTTP Request (`n8n-nodes-base.httpRequest`) — outbound CRM sync.
- **Configuration (interpreted):**
  - Method: POST
  - Body: JSON with
    - `customer_id`, `email`
    - `tags`: segment tag
    - `source`: `{{ $json.source }}`
  - Response: full response enabled.
  - **Important:** The node does **not** show a configured URL in the provided JSON snippet; you must set the CRM endpoint and authentication.
- **Inputs/Outputs:**
  - Input: normalized segmentation event.
  - Output: Append or update row in sheet.
- **Version notes:** TypeVersion 4.3.
- **Edge cases / failures:**
  - Missing URL/credentials will hard-fail.
  - CRM API may reject invalid tag values or unknown customer_id/email.
  - Network timeouts / non-2xx responses. Because fullResponse is enabled, downstream data structure includes statusCode/headers/body—be aware if later nodes expect only body.

#### Node: Append or update row in sheet
- **Type / role:** Google Sheets (`n8n-nodes-base.googleSheets`) — audit log storage.
- **Configuration (interpreted):**
  - Operation: Append or Update
  - Document: “Log Customer Segmentation Event” (Spreadsheet ID `1vPSuaAEv77tzbPLWzFM-t5iDVca9_DJBDS2rh6m5N6s`)
  - Sheet tab: “Sheet1” (gid=0)
  - Mapping mode: define columns explicitly:
    - Email, Source, Event Time, Customer Id, Segment Tag
  - Matching columns: **Event Time**
- **Inputs/Outputs:**
  - Input: output from “Sync Customer Segment to CRM” but the mapped values explicitly reference `$('Normalize Customer Segment Output').item.json...` (so it logs segmentation data regardless of CRM response).
  - Output: end.
- **Credentials:** Google Sheets OAuth2 (“Google Sheets account 36”).
- **Version notes:** TypeVersion 4.7.
- **Edge cases / failures:**
  - Using **Event Time** as the matching key is unusual because it is always “now”; updates will almost never match and will effectively append every time. If you intend to update per customer, match on Customer Id or Email instead.
  - If sheet columns don’t exist or names differ (case/spacing), mapping fails.
  - Google API quota/auth expiration issues.

**Sticky note coverage for this block:**
- “## Sync Segment & Log Activity … synced to a CRM… logged in Google Sheets…”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Receive Customer Order or Update Event | Webhook | Entry point event receiver | — | Normalize Customer Purchase Data | ## Event Intake & Data Normalization\n\nThis section receives customer or order events via webhook and converts raw payloads into a consistent structure. Key attributes like customer ID, email, order count, lifetime spend, last order date and source system are standardized here for reliable downstream processing. |
| Normalize Customer Purchase Data | Set | Normalizes incoming payload fields | Receive Customer Order or Update Event | Validate Customer Event | ## Event Intake & Data Normalization\n\nThis section receives customer or order events via webhook and converts raw payloads into a consistent structure. Key attributes like customer ID, email, order count, lifetime spend, last order date and source system are standardized here for reliable downstream processing. |
| Validate Customer Event | IF | Rejects events without purchase history | Normalize Customer Purchase Data | Determine Customer Segment (true branch only) | ## Event Intake & Data Normalization\n\nThis section receives customer or order events via webhook and converts raw payloads into a consistent structure. Key attributes like customer ID, email, order count, lifetime spend, last order date and source system are standardized here for reliable downstream processing. |
| Determine Customer Segment | Switch | Routes to segment branches by rules | Validate Customer Event | Assign VIP Customer Segment; Assign New Customer Segment; Assign Repeat Customer Segment; Assign Inactive Customer Segment | ## Customer Segmentation Rules\n\nThis section determines which segment a customer belongs to using rule-based logic. Customers are classified as VIP, new, repeat, or inactive based on purchase frequency, lifetime spend and last activity date. Each branch assigns a clear segment tag for downstream syncing. |
| Assign VIP Customer Segment | Set | Sets segment_tag to vip_customer | Determine Customer Segment | Normalize Customer Segment Output | ## Customer Segmentation Rules\n\nThis section determines which segment a customer belongs to using rule-based logic. Customers are classified as VIP, new, repeat, or inactive based on purchase frequency, lifetime spend and last activity date. Each branch assigns a clear segment tag for downstream syncing. |
| Assign New Customer Segment | Set | Sets segment_tag to new_customer | Determine Customer Segment | Normalize Customer Segment Output | ## Customer Segmentation Rules\n\nThis section determines which segment a customer belongs to using rule-based logic. Customers are classified as VIP, new, repeat, or inactive based on purchase frequency, lifetime spend and last activity date. Each branch assigns a clear segment tag for downstream syncing. |
| Assign Repeat Customer Segment | Set | Sets segment_tag to repeat_customer | Determine Customer Segment | Normalize Customer Segment Output | ## Customer Segmentation Rules\n\nThis section determines which segment a customer belongs to using rule-based logic. Customers are classified as VIP, new, repeat, or inactive based on purchase frequency, lifetime spend and last activity date. Each branch assigns a clear segment tag for downstream syncing. |
| Assign Inactive Customer Segment | Set | Sets segment_tag to inactive_customer | Determine Customer Segment | Normalize Customer Segment Output | ## Customer Segmentation Rules\n\nThis section determines which segment a customer belongs to using rule-based logic. Customers are classified as VIP, new, repeat, or inactive based on purchase frequency, lifetime spend and last activity date. Each branch assigns a clear segment tag for downstream syncing. |
| Normalize Customer Segment Output | Set | Builds clean output payload for syncing/logging | Assign VIP/New/Repeat/Inactive Customer Segment | Sync Customer Segment to CRM | ## Sync Segment & Log Activity\n\nAfter a segment is assigned, the workflow prepares a clean output payload. Customer segment tags are synced to a CRM or email platform and each segmentation event is logged in Google Sheets for visibility, reporting and audit purposes. |
| Sync Customer Segment to CRM | HTTP Request | Sends segment tag to CRM API | Normalize Customer Segment Output | Append or update row in sheet | ## Sync Segment & Log Activity\n\nAfter a segment is assigned, the workflow prepares a clean output payload. Customer segment tags are synced to a CRM or email platform and each segmentation event is logged in Google Sheets for visibility, reporting and audit purposes. |
| Append or update row in sheet | Google Sheets | Logs segmentation event | Sync Customer Segment to CRM | — | ## Sync Segment & Log Activity\n\nAfter a segment is assigned, the workflow prepares a clean output payload. Customer segment tags are synced to a CRM or email platform and each segmentation event is logged in Google Sheets for visibility, reporting and audit purposes. |
| Sticky Note | Sticky Note | Overview/commentary | — | — | ## (Retail) Auto-Segment Customers by Purchase Behavior (Overview)\n\n### How it works\nThis workflow listens for customer or order events from platforms like WooCommerce, Shopify, or a CRM using a webhook. Incoming payloads are first normalized to extract customer and behavioral data such as order count, lifetime spend, last order date, product categories and source system.\n\nBased on these values, the workflow evaluates customer behavior using rule-based conditions. Customers are automatically classified into meaningful segments such as new customers, repeat customers, VIP customers, or inactive customers. Once a segment is assigned, the workflow prepares a clean output and syncs the segment tag to a CRM or email platform. Each segmentation event is also logged into Google Sheets for tracking and auditing.\n\n### Setup steps\n1. Configure your source system to send customer or order events to the webhook.\n2. Ensure required fields like order count, lifetime spend, last order date and source are included in the payload.\n3. Update the CRM HTTP Request node with your actual API endpoint and authentication.\n4. Connect Google Sheets and ensure required columns exist.\n5. Test with sample payloads and activate the workflow. |
| Sticky Note1 | Sticky Note | Section label/commentary | — | — |  |
| Sticky Note2 | Sticky Note | Section label/commentary | — | — |  |
| Sticky Note3 | Sticky Note | Section label/commentary | — | — |  |

> Note: Sticky notes are included as nodes in the workflow; the “Sticky Note” column is primarily relevant for functional nodes, but sticky note nodes themselves are also listed here to satisfy “do not skip any nodes”.

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **(Retail) Auto-Segment Customers by Purchase Behavior**
   - Keep it inactive until testing is complete.

2. **Add Trigger: Webhook**
   - Node type: **Webhook**
   - Name: **Receive Customer Order or Update Event**
   - HTTP Method: **POST**
   - Path: **customer-segmentation**
   - Save and copy the test/production webhook URL for your source platform.

3. **Add Set node for normalization**
   - Node type: **Set**
   - Name: **Normalize Customer Purchase Data**
   - Add fields (use expressions):
     - `customer_id` = `{{$json.body.customer.customer_id}}`
     - `email` = `{{$json.body.customer.email}}`
     - `first_name` = `{{$json.body.customer.first_name}}`
     - `last_name` = `{{$json.body.customer.last_name}}`
     - `order_id` = `{{$json.body.order.order_id}}`
     - `order_value` (Number) = `{{$json.body.order.order_value}}`
     - `order_date` = `{{$json.body.order.order_date}}`
     - `order_count` (Number) = `{{$json.body.customer_stats.order_count}}`
     - `lifetime_spend` (Number) = `{{$json.body.customer_stats.lifetime_spend}}`
     - `last_order_date` = `{{$json.body.customer_stats.last_order_date}}`
     - `product_categories` (Array) = `{{$json.body.order.product_categories}}`
     - `source_system` = `{{$json.body.source}}`
     - `event_type` = `{{$json.body.event_type}}`
   - Connect: **Webhook → Normalize Customer Purchase Data**

4. **Add IF node to validate**
   - Node type: **IF**
   - Name: **Validate Customer Event**
   - Condition (Number): `{{$json.body.customer_stats.order_count}}` **notEquals** `0`
   - Connect: **Normalize Customer Purchase Data → Validate Customer Event**
   - Use the **true** output for downstream; leave false unconnected (or connect to a “NoOp/Log” node if you want observability).

5. **Add Switch node for segmentation**
   - Node type: **Switch**
   - Name: **Determine Customer Segment**
   - Add rules with renamed outputs:
     1. Output key `vip_customer`: `{{$('Normalize Customer Purchase Data').item.json.lifetime_spend}}` **gt** `1000`
     2. Output key `new_customer`: `{{$('Normalize Customer Purchase Data').item.json.order_count}}` **equals** `1`
     3. Output key `repeat_customer`: `{{$('Normalize Customer Purchase Data').item.json.order_count}}` **gt** `1`
     4. Output key `inactive_customer`:
        - Left value: `{{ new Date($json.last_order_date).getTime() }}`
        - Operator: **lt**
        - Right value: `{{ Date.now() - (60 * 24 * 60 * 60 * 1000) }}`
   - Connect: **Validate Customer Event (true) → Determine Customer Segment**

6. **Add 4 Set nodes to assign segment tags**
   - Add Set node: **Assign VIP Customer Segment** → set `segment_tag = "vip_customer"`
   - Add Set node: **Assign New Customer Segment** → set `segment_tag = "new_customer"`
   - Add Set node: **Assign Repeat Customer Segment** → set `segment_tag = "repeat_customer"`
   - Add Set node: **Assign Inactive Customer Segment** → set `segment_tag = "inactive_customer"`
   - Connect each Switch output to its matching Set node.

7. **Add Set node to normalize output**
   - Node type: **Set**
   - Name: **Normalize Customer Segment Output**
   - Assign:
     - `customer_id` = `{{$('Normalize Customer Purchase Data').item.json.customer_id}}`
     - `email` = `{{$('Normalize Customer Purchase Data').item.json.email}}`
     - `segment_tag` = `{{$json.segment_tag}}`
     - `source` = `{{$('Normalize Customer Purchase Data').item.json.source_system}}`
     - `event_time` = `{{$now}}`
   - Connect each of the 4 “Assign … Segment” nodes → **Normalize Customer Segment Output**

8. **Add HTTP Request node to sync to CRM**
   - Node type: **HTTP Request**
   - Name: **Sync Customer Segment to CRM**
   - Method: **POST**
   - URL: set your CRM endpoint (e.g., `https://api.yourcrm.com/customers/tag`)
   - Authentication: configure as required by your CRM (Header token, OAuth2, etc.)
   - Body type: **JSON**
   - JSON body (expressions):
     - `customer_id`: `{{$json.customer_id}}`
     - `email`: `{{$json.email}}`
     - `tags`: `{{$json.segment_tag}}`
     - `source`: `{{$json.source}}`
   - (Optional) Enable “Full Response” if you want status codes/headers stored.
   - Connect: **Normalize Customer Segment Output → Sync Customer Segment to CRM**

9. **Add Google Sheets node for logging**
   - Node type: **Google Sheets**
   - Name: **Append or update row in sheet**
   - Credentials: connect **Google Sheets OAuth2**
   - Operation: **Append or Update**
   - Select Spreadsheet: create/select **Log Customer Segmentation Event**
   - Select Sheet: **Sheet1**
   - Define columns mapping:
     - `Email` = `{{$('Normalize Customer Segment Output').item.json.email}}`
     - `Source` = `{{$('Normalize Customer Segment Output').item.json.source}}`
     - `Event Time` = `{{$('Normalize Customer Segment Output').item.json.event_time}}`
     - `Customer Id` = `{{$('Normalize Customer Segment Output').item.json.customer_id}}`
     - `Segment Tag` = `{{$('Normalize Customer Segment Output').item.json.segment_tag}}`
   - Matching columns: `Event Time` (replicate current behavior)  
     - If you want dedup/update per customer, change matching to `Customer Id` or `Email`.
   - Connect: **Sync Customer Segment to CRM → Append or update row in sheet**

10. **Test end-to-end**
   - Send a sample POST to the webhook containing:
     - `body.customer`, `body.order`, `body.customer_stats`, `body.source`, `body.event_type`
   - Verify:
     - Correct branch selection (VIP/New/Repeat/Inactive)
     - CRM receives expected payload
     - Sheet row appended with correct values

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “(Retail) Auto-Segment Customers by Purchase Behavior (Overview)… Setup steps… Configure source system webhook; ensure required fields; update CRM node endpoint/auth; connect Google Sheets; test and activate.” | Sticky note overview embedded in the workflow |
| Segmentation logic is rule-based with thresholds: VIP spend > 1000; Inactive > 60 days since last order. | Customer Segmentation Rules sticky note context |
| Logging uses Google Sheets “Append or Update” but matches on “Event Time”, which usually behaves like append-only because event_time is always new. | Operational consideration for auditing/deduplication |

