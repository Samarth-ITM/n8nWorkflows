Log failed WooCommerce orders to Airtable and send OpenAI-powered Slack alerts

https://n8nworkflows.xyz/workflows/log-failed-woocommerce-orders-to-airtable-and-send-openai-powered-slack-alerts-12376


# Log failed WooCommerce orders to Airtable and send OpenAI-powered Slack alerts

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow periodically checks a WooCommerce store for **failed payment orders**, prevents duplicate logging, stores new failures in **Airtable**, generates an **OpenAI-written Slack summary**, and posts alerts to **Slack**. If an order has already been logged, it posts a “duplicate detected” Slack message instead (and does not create a new Airtable record).

**Target use cases:**
- Customer support / finance teams needing immediate visibility into failed payments
- Building a searchable history of failed payments for reporting and follow-up
- Reducing manual monitoring of WooCommerce admin

### 1.1 Workflow Start & Domain Setup
Runs on a schedule and sets the WooCommerce domain used by subsequent API calls.

### 1.2 Fetch Failed Orders from WooCommerce
Calls the WooCommerce REST API to retrieve orders with `status=failed`.

### 1.3 Per-Order Processing + Duplicate Check (Airtable)
Processes each failed order one-by-one, searches Airtable for an existing record matching `order_id`, merges data for safe evaluation, then branches:
- **Duplicate**: send Slack info message and stop.
- **New**: format order data → create Airtable record.

### 1.4 AI Summary + Slack Alert
Uses OpenAI to generate a concise Slack message from the Airtable-created record, then posts it to a Slack channel.

---

## 2. Block-by-Block Analysis

### Block 1 — Workflow Start & Domain Setup

**Overview:**  
Triggers the workflow on a schedule and defines the WooCommerce domain as a variable (`wc_domain`) to build API URLs consistently.

**Nodes involved:**
- `Check Failed Orders (Scheduler)`
- `Set WooCommerce Domain2`
- (Sticky Note) `Sticky Note5`

#### Node: Check Failed Orders (Scheduler)
- **Type / role:** `Schedule Trigger` — workflow entry point.
- **Configuration (interpreted):** Runs on an interval rule (the JSON shows an interval object but without visible specifics; in n8n UI this must be set to something like “every X minutes/hours”).
- **Connections:**
  - Output → `Set WooCommerce Domain2`
- **Edge cases / failures:**
  - If the schedule interval is not properly configured in UI, it may not run as expected.
  - High frequency schedules may hit WooCommerce rate limits downstream.

#### Node: Set WooCommerce Domain2
- **Type / role:** `Set` — defines variables used later.
- **Configuration:**
  - Sets `wc_domain` (string) to `{{YOUR_DOMAIN}}` (placeholder).
- **Key variables:**
  - `$json.wc_domain` used to build the WooCommerce API URL.
- **Connections:**
  - Input ← `Check Failed Orders (Scheduler)`
  - Output → `Fetch Failed Orders From WooCommerce2`
- **Edge cases / failures:**
  - If `wc_domain` is left as `{{YOUR_DOMAIN}}`, the HTTP node will call an invalid host.
  - Must not include protocol here (the HTTP node already prepends `https://`).

---

### Block 2 — Fetch Failed Orders from WooCommerce

**Overview:**  
Fetches all orders in WooCommerce with status `failed` via REST API using Basic Auth.

**Nodes involved:**
- `Fetch Failed Orders From WooCommerce2`

#### Node: Fetch Failed Orders From WooCommerce2
- **Type / role:** `HTTP Request` — calls WooCommerce REST API.
- **Configuration:**
  - URL: `https://{{$json.wc_domain}}/wp-json/wc/v3/orders?status=failed`
  - Also sends query parameter `status=failed` (redundant with the URL query string, but harmless).
  - Auth: `httpBasicAuth` via “genericCredentialType”.
  - Credentials name: `Woocommerece Credentials` (Consumer Key/Secret).
- **Connections:**
  - Input ← `Set WooCommerce Domain2`
  - Output → `Process Each Failed Order`
- **Edge cases / failures:**
  - **401/403** if Consumer Key/Secret are wrong or lack permissions.
  - **404** if domain/path is wrong or permalinks/API disabled.
  - **Pagination issue:** WooCommerce orders endpoint is paginated (default per_page often 10). This node does not show pagination handling, so it may only fetch the first page unless n8n HTTP Request pagination options are configured in UI (not present in JSON).
  - **Large payloads:** many failed orders can increase memory/time.

---

### Block 3 — Failed Order Handling & Duplicate Check

**Overview:**  
Iterates through each failed order, searches Airtable for an existing record using `order_id`, merges the Airtable search result with the original order data, and routes either to a duplicate Slack message or to formatting + storage.

**Nodes involved:**
- `Process Each Failed Order`
- `Search records`
- `Attach Search Result to Order`
- `Is Order Already Logged?`
- `Send a message`
- (Sticky Notes) `Sticky Note9`, `Sticky Note8`

#### Node: Process Each Failed Order
- **Type / role:** `Split In Batches` — loops items.
- **Configuration:**
  - Batch processing with default options (batch size default is typically 1 unless changed in UI).
  - Uses split/batch second output to drive loop-style processing.
- **Connections (important):**
  - Input ← `Fetch Failed Orders From WooCommerce2`
  - Output **(index 1)** → `Search records` and → `Attach Search Result to Order`  
    (This is unusual-looking in JSON: it sends both to the merge node. In practice: one path carries the order, the other carries search results to be merged.)
- **Edge cases / failures:**
  - If batch size > 1, merge logic may behave unexpectedly unless merge is configured to match items.
  - If WooCommerce returns an empty array, nothing else runs (expected).

#### Node: Search records
- **Type / role:** `Airtable` (operation: Search) — checks duplicates.
- **Configuration:**
  - Base: `Fake Review Detector` (Base ID `appipUxLpxPEKRXrJ`)
  - Table: `OrderFailedData` (Table ID `tblP4nfLbKZdHcGFt`)
  - Filter formula: `=order_id = {{ $json.id }}`
  - `alwaysOutputData: true` (so downstream nodes still run even if no record is found).
- **Inputs/outputs:**
  - Input ← `Process Each Failed Order` (per order)
  - Output → `Attach Search Result to Order` (to merge as input 1)
- **Edge cases / failures:**
  - **FilterByFormula syntax risk:** Airtable formulas usually require braces around field names: `{order_id} = 123`. As written, `order_id = ...` may fail or return no results depending on Airtable parsing. This is a common source of false “not found”.
  - Data types: if Airtable field is numeric but `$json.id` is string, comparison may fail.
  - Auth errors if Airtable PAT lacks access to base/table.

#### Node: Attach Search Result to Order
- **Type / role:** `Merge` — combines order + Airtable search result so the IF check can safely inspect the search outcome.
- **Configuration:**
  - Default merge settings (not explicitly shown). In n8n, default is often “Append” or “Combine” depending on node version/UI. Here it is used as a junction so `Is Order Already Logged?` can reference `$items("Search records")...`.
- **Connections:**
  - Input 0 ← `Process Each Failed Order` (order data)
  - Input 1 ← `Search records` (search result)
  - Output → `Is Order Already Logged?`
- **Edge cases / failures:**
  - If merge mode mismatches item pairing, the IF node might evaluate against the wrong search result.
  - If Airtable search returns multiple matches, the IF condition only checks the first item of `"Search records"`.

#### Node: Is Order Already Logged?
- **Type / role:** `IF` — routes duplicates vs new records.
- **Configuration (interpreted):**
  - Condition: checks that `{{ $items("Search records")[0]?.json?.id }}` is **not empty**.
  - If **true** = record exists (duplicate).
  - If **false** = record does not exist (new).
- **Connections:**
  - True output → `Send a message` (duplicate notification)
  - False output → `Format Order Data1` (new record path)
- **Edge cases / failures:**
  - If the merge/search path fails and `"Search records"` has no items, the expression uses optional chaining and will evaluate to empty → treated as **new order** (risk: duplicates).
  - If Airtable filter formula is wrong and returns nothing, duplicates will be inserted again.

#### Node: Send a message
- **Type / role:** `Slack` — informational Slack message for duplicates.
- **Configuration:**
  - Posts to channel `n8n` (channel ID `C09S57E2JQ2`)
  - Message includes order id, customer_id, total, currency; includes “link to workflow”.
- **Connections:**
  - Input ← `Is Order Already Logged?` (true branch)
  - No downstream nodes.
- **Edge cases / failures:**
  - Slack auth/channel permissions issues.
  - Message uses `customer_id` which may be null/undefined for guest checkout (handled with `|| 'N/A'`).

---

### Block 4 — Prepare, Save, Summarize, Notify

**Overview:**  
Formats the WooCommerce order into a standardized schema, creates an Airtable record, asks OpenAI to generate a Slack-ready summary using Airtable fields, then posts the alert to Slack.

**Nodes involved:**
- `Format Order Data1`
- `Save Failed Order to Airtable1`
- `Generate Slack Summary Message 1`
- `Send Failed Order Alert to Slack1`
- (Sticky Notes) `Sticky Note7`, `Sticky Note6`, `Sticky Note3`

#### Node: Format Order Data1
- **Type / role:** `Code` — transforms raw WooCommerce order JSON into a finance-friendly structure.
- **Configuration:**
  - Reads `items[0].json` as `order`.
  - Derives:
    - `failure_reason_raw` from `order.payment_method_title` (fallback: `"Unknown"`)
    - `failure_reason_canonical` mapping (very limited: expired/declined else `unknown`)
  - Outputs object with:
    - `order_id`, `invoice_amount` (order.total), `currency`
    - customer name/email from `order.billing`
    - `payment_method`, reasons, `attempts_count: 1`
    - `order_items_summary` built from `line_items`
    - `links.order_admin_url` built with a hard-coded `https://{{YOUR_DOMAIN}}/...` placeholder
    - `links.retry_payment_url` from `order.payment_url`
- **Connections:**
  - Input ← `Is Order Already Logged?` (false branch)
  - Output → `Save Failed Order to Airtable1`
- **Edge cases / failures:**
  - If `order.billing` is missing, string interpolation may throw (e.g., `order.billing.first_name`). Consider guarding.
  - `order.total` is usually a string in WooCommerce; Airtable schema expects string for `invoice_amount`, so that’s OK.
  - `order_admin_url` uses `{{YOUR_DOMAIN}}` literal, not `wc_domain`. If not replaced, stored links will be wrong.
  - `order_items_summary` can be long; Airtable field limits may apply.

#### Node: Save Failed Order to Airtable1
- **Type / role:** `Airtable` (operation: Create) — persists the failed order.
- **Configuration:**
  - Base/Table: same as the search node (`Fake Review Detector` / `OrderFailedData`)
  - Maps multiple columns from `$json` (formatted structure).
  - Notable mapping issue:
    - `order_items_summary` is set to `={{ $json.links.order_admin_url }}` (appears to be a mistake; should likely be `={{ $json.order_items_summary }}`).
- **Connections:**
  - Input ← `Format Order Data1`
  - Output → `Generate Slack Summary Message 1`
- **Edge cases / failures:**
  - Airtable field type mismatches (e.g., `order_id` numeric vs string).
  - Missing required fields (if Airtable enforces required columns).
  - Rate limits on Airtable if many orders are processed at once.

#### Node: Generate Slack Summary Message 1
- **Type / role:** `OpenAI (LangChain)` — generates natural-language Slack summary.
- **Configuration:**
  - Model: `gpt-4o-mini`
  - Prompt uses Airtable node output structure: `{{$json.fields.<field>}}`
  - Instructs: short, professional summary; include details and recommended action.
- **Connections:**
  - Input ← `Save Failed Order to Airtable1` (uses created record fields)
  - Output → `Send Failed Order Alert to Slack1`
- **Edge cases / failures:**
  - If Airtable create output doesn’t include `fields` as expected (depends on node settings/version), prompt variables become empty.
  - OpenAI API errors (401, quota, rate limit, timeout).
  - Content length: very long fields can increase token usage.

#### Node: Send Failed Order Alert to Slack1
- **Type / role:** `Slack` — posts the AI-generated alert.
- **Configuration:**
  - Channel: `n8n` (C09S57E2JQ2)
  - Message text: `={{ $json.output[0].content[0].text }}`
    - This relies on the OpenAI node’s specific output shape.
- **Connections:**
  - Input ← `Generate Slack Summary Message 1`
  - No downstream nodes.
- **Edge cases / failures:**
  - If OpenAI node output format differs (common across versions), the expression may fail or post blank text.
  - Slack posting permissions/channel membership.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note3 | Sticky Note | Documentation / setup guidance | — | — | ## How It Works… (full note content about monitoring failed payments, Airtable storage, AI summary, Slack alert; setup steps for WooCommerce, Airtable, OpenAI, Slack; activate workflow) |
| Check Failed Orders (Scheduler) | Schedule Trigger | Entry point; periodic execution | — | Set WooCommerce Domain2 | ## Workflow Start & Domain Setup… |
| Sticky Note5 | Sticky Note | Documents start/domain block | — | — | ## Workflow Start & Domain Setup… |
| Set WooCommerce Domain2 | Set | Defines `wc_domain` variable | Check Failed Orders (Scheduler) | Fetch Failed Orders From WooCommerce2 | ## Workflow Start & Domain Setup… |
| Fetch Failed Orders From WooCommerce2 | HTTP Request | Retrieve WooCommerce failed orders | Set WooCommerce Domain2 | Process Each Failed Order | ## Failed Order Handling & Duplicate Check… |
| Sticky Note9 | Sticky Note | Documents failed-order handling & dedupe | — | — | ## Failed Order Handling & Duplicate Check… |
| Process Each Failed Order | Split In Batches | Iterate through orders | Fetch Failed Orders From WooCommerce2 | Search records; Attach Search Result to Order | ## Failed Order Handling & Duplicate Check… |
| Search records | Airtable (Search) | Look up order_id to prevent duplicates | Process Each Failed Order | Attach Search Result to Order | ## Failed Order Handling & Duplicate Check… |
| Attach Search Result to Order | Merge | Combine order + search result context | Process Each Failed Order; Search records | Is Order Already Logged? | ## Failed Order Handling & Duplicate Check… |
| Is Order Already Logged? | IF | Branch duplicate vs new | Attach Search Result to Order | Send a message; Format Order Data1 | ## Failed Order Handling & Duplicate Check… |
| Send a message | Slack | Notify duplicates (no action) | Is Order Already Logged? (true) | — | This odres are already log in table |
| Sticky Note8 | Sticky Note | Notes duplicates branch | — | — | This odres are already log in table |
| Format Order Data1 | Code | Normalize/standardize order fields | Is Order Already Logged? (false) | Save Failed Order to Airtable1 | ## Prepare & Save Failed Orders… |
| Sticky Note7 | Sticky Note | Documents formatting + Airtable save | — | — | ## Prepare & Save Failed Orders… |
| Save Failed Order to Airtable1 | Airtable (Create) | Store standardized record | Format Order Data1 | Generate Slack Summary Message 1 | ## Prepare & Save Failed Orders… |
| Generate Slack Summary Message 1 | OpenAI (LangChain) | Generate Slack-ready summary | Save Failed Order to Airtable1 | Send Failed Order Alert to Slack1 | ## Create Summary & Send Team Notification… |
| Sticky Note6 | Sticky Note | Documents AI + Slack alert | — | — | ## Create Summary & Send Team Notification… |
| Send Failed Order Alert to Slack1 | Slack | Post AI alert to channel | Generate Slack Summary Message 1 | — | ## Create Summary & Send Team Notification… |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Trigger**
   1. Add node: **Schedule Trigger** named `Check Failed Orders (Scheduler)`.
   2. Set the interval (e.g., every 5 minutes / hourly) as desired.

2) **Set WooCommerce Domain**
   1. Add node: **Set** named `Set WooCommerce Domain2`.
   2. Add a field:
      - Name: `wc_domain`
      - Type: `String`
      - Value: your domain only, e.g. `example.com` (no `https://`).

3) **Fetch failed orders from WooCommerce**
   1. Add node: **HTTP Request** named `Fetch Failed Orders From WooCommerce2`.
   2. Method: `GET`
   3. URL: `https://{{$json.wc_domain}}/wp-json/wc/v3/orders`
   4. Query parameters:
      - `status` = `failed`
   5. Authentication: **Basic Auth**
      - Username: WooCommerce **Consumer Key**
      - Password: WooCommerce **Consumer Secret**
      - In n8n, store these in an **HTTP Basic Auth** credential.

4) **Loop through orders**
   1. Add node: **Split In Batches** named `Process Each Failed Order`.
   2. Keep default batch size (commonly 1) unless you also adjust merge strategy.

5) **Search Airtable for duplicates**
   1. Add node: **Airtable** named `Search records`.
   2. Operation: **Search**
   3. Select your **Base** and **Table** (create a table like `OrderFailedData`).
   4. Set *Filter by formula* to match the current WooCommerce order id. Recommended robust formula:
      - `{order_id} = {{$json.id}}`
   5. Enable “Always Output Data” (so downstream logic can still run when not found).
   6. Configure Airtable credentials using a **Personal Access Token** with base/table scopes.

6) **Merge search result with order context**
   1. Add node: **Merge** named `Attach Search Result to Order`.
   2. Connect:
      - `Process Each Failed Order` → Merge **Input 1** (order)
      - `Search records` → Merge **Input 2** (search result)
   3. Choose a merge mode that preserves both streams safely (commonly “Wait for both inputs” / “Combine” depending on n8n version).

7) **IF: already logged?**
   1. Add node: **IF** named `Is Order Already Logged?`.
   2. Condition: check whether an Airtable record id exists:
      - Value 1 (expression): `{{ $items("Search records")[0]?.json?.id }}`
      - Operation: “is not empty”
   3. True branch = duplicate; False branch = new.

8) **Duplicate branch Slack message**
   1. Add node: **Slack** named `Send a message`.
   2. Operation: “Send Message” to a channel.
   3. Text (expression) similar to:
      - `Duplicate failed order detected... Order ID: {{ $json.id }} ...`
   4. Configure Slack credentials (OAuth2) and select channel.

9) **Format order data (new orders)**
   1. Add node: **Code** named `Format Order Data1`.
   2. Paste the transformation logic (adapt domain usage):
      - Prefer: build admin URL using `wc_domain` rather than a placeholder.
      - Ensure you output keys you want to store in Airtable (order_id, invoice_amount, etc.).

10) **Create Airtable record**
   1. Add node: **Airtable** named `Save Failed Order to Airtable1`.
   2. Operation: **Create**
   3. Map Airtable columns to expressions from `Format Order Data1` output, e.g.:
      - `order_id` → `{{ $json.order_id }}`
      - `order_items_summary` → `{{ $json.order_items_summary }}` (avoid the current workflow’s apparent mis-mapping)
      - `order_admin_url` → `{{ $json.links.order_admin_url }}`
   4. Use the same Airtable credential as the search node.

11) **Generate Slack summary with OpenAI**
   1. Add node: **OpenAI** (the n8n LangChain OpenAI node) named `Generate Slack Summary Message 1`.
   2. Model: `gpt-4o-mini` (or your preferred model).
   3. Prompt: reference Airtable output fields:
      - `{{$json.fields.order_id}}`, etc.
   4. Configure OpenAI API credential (API key).

12) **Send AI alert to Slack**
   1. Add node: **Slack** named `Send Failed Order Alert to Slack1`.
   2. Channel: choose where alerts should go.
   3. Text expression: adapt to the OpenAI node output format used in your n8n version. In this workflow it is:
      - `{{ $json.output[0].content[0].text }}`
   4. Test by executing once and verifying the data path.

13) **Connect nodes in order**
   - Schedule Trigger → Set → HTTP Request → Split In Batches  
   - Split In Batches → Airtable Search and → Merge  
   - Airtable Search → Merge → IF  
   - IF true → Slack (duplicate)  
   - IF false → Code → Airtable Create → OpenAI → Slack (alert)

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The workflow includes embedded setup guidance covering WooCommerce domain, WooCommerce API Basic Auth, Airtable base/table selection, OpenAI API key, and Slack channel configuration. | Sticky Note “How It Works / Setup Steps” (Sticky Note3) |
| Duplicate handling exists: if `order_id` is found in Airtable, the workflow avoids creating a new record and posts an informational Slack message. | Sticky Note9 + duplicate Slack node |
| Potential configuration bug: Airtable “order_items_summary” mapping currently uses `links.order_admin_url` instead of the actual item summary. | `Save Failed Order to Airtable1` mapping |
| Potential Airtable formula issue: `order_id = {{ $json.id }}` may need Airtable field braces: `{order_id} = {{$json.id}}`. | `Search records` filterByFormula |

