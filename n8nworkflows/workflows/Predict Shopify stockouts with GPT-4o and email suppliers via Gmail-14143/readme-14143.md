Predict Shopify stockouts with GPT-4o and email suppliers via Gmail

https://n8nworkflows.xyz/workflows/predict-shopify-stockouts-with-gpt-4o-and-email-suppliers-via-gmail-14143


# Predict Shopify stockouts with GPT-4o and email suppliers via Gmail

# 1. Workflow Overview

This workflow performs a daily inventory risk assessment for a Shopify store. It retrieves products and recent orders, calculates sales velocity for each product variant, asks GPT-4o to assess stockout risk and reorder needs, then either emails a supplier and logs the decision, or just logs the result.

Its main use case is proactive replenishment for e-commerce operations where variant-level stock can run out quickly and manual monitoring is impractical.

## 1.1 Scheduled Trigger

The workflow starts automatically every day at 06:00 UTC.

## 1.2 Shopify Data Collection

It fetches:
- all Shopify products, including variant inventory levels
- all recent orders, intended for use in a 7-day sales analysis

## 1.3 Inventory Velocity Calculation

A Code node joins product variants with ordered quantities from recent orders and computes:
- units sold over 7 days
- daily sales velocity
- estimated days until stockout

## 1.4 AI-Based Reorder Decision

An AI Agent powered by GPT-4o classifies each variant into a stockout risk level and returns structured output:
- risk level
- recommended reorder quantity
- reasoning
- whether to reorder now

## 1.5 Conditional Routing and Notifications

If the AI says `should_reorder = true`, the workflow sends a reorder email to the supplier and then logs the result to Google Sheets.

If `should_reorder = false`, it skips the email and logs the decision only.

---

# 2. Block-by-Block Analysis

## 2.1 Workflow Documentation and Setup Notes

### Overview
This block consists of sticky notes embedded in the canvas. They explain the workflow purpose, prerequisites, setup requirements, and execution logic for operators or future maintainers.

### Nodes Involved
- Overview
- Prerequisites
- Setup Required
- How It Works

### Node Details

#### Overview
- **Type and role:** Sticky Note; visual documentation only.
- **Configuration choices:** Describes the business goal, version, sequence, and major processing stages.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** None.
- **Edge cases or failure types:** None; purely informational.
- **Sub-workflow reference:** None.

#### Prerequisites
- **Type and role:** Sticky Note; setup checklist.
- **Configuration choices:** Lists required systems: Shopify Admin API, OpenAI, Gmail, Google Sheets.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** None.
- **Edge cases or failure types:** None.
- **Sub-workflow reference:** None.

#### Setup Required
- **Type and role:** Sticky Note; implementation guidance.
- **Configuration choices:** Explains which credentials must be connected and what placeholders must be replaced.
- **Key expressions or variables used:** References the `should_reorder` condition conceptually.
- **Input and output connections:** None.
- **Version-specific requirements:** None.
- **Edge cases or failure types:** Misconfiguration risks are described here, but the note itself cannot fail.
- **Sub-workflow reference:** None.

#### How It Works
- **Type and role:** Sticky Note; operational explanation.
- **Configuration choices:** Describes the scheduled execution sequence and decision path.
- **Key expressions or variables used:** Mentions `should_reorder`.
- **Input and output connections:** None.
- **Version-specific requirements:** None.
- **Edge cases or failure types:** None.
- **Sub-workflow reference:** None.

---

## 2.2 Scheduled Execution

### Overview
This block defines the daily entry point for the workflow. It triggers the inventory analysis once per day at a fixed UTC time.

### Nodes Involved
- Daily Inventory Check

### Node Details

#### Daily Inventory Check
- **Type and role:** `n8n-nodes-base.scheduleTrigger`; workflow entry point.
- **Configuration choices:** Uses a cron expression `0 6 * * *`, which means every day at 06:00 UTC.
- **Key expressions or variables used:** None.
- **Input and output connections:** No input; outputs to **Fetch All Products**.
- **Version-specific requirements:** Uses node type version `1.2`. Cron behavior depends on n8n scheduler support in the deployment environment.
- **Edge cases or failure types:**
  - Workflow will not run if inactive.
  - Server timezone misunderstandings can cause confusion, though the note explicitly describes UTC.
  - Self-hosted instances require working queue/scheduler infrastructure if applicable.
- **Sub-workflow reference:** None.

---

## 2.3 Shopify Data Retrieval

### Overview
This block gathers the raw operational data needed for stockout prediction. It first retrieves products and variants, then retrieves orders used to estimate recent sales demand.

### Nodes Involved
- Fetch Products note
- Fetch All Products
- Fetch Orders note
- Fetch Recent Orders

### Node Details

#### Fetch Products note
- **Type and role:** Sticky Note; explains the purpose of product retrieval.
- **Configuration choices:** States that product variants and `inventory_quantity` are being pulled.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** None.
- **Edge cases or failure types:** None.
- **Sub-workflow reference:** None.

#### Fetch All Products
- **Type and role:** `n8n-nodes-base.shopify`; pulls all Shopify products.
- **Configuration choices:**
  - Resource: `product`
  - Operation: `getAll`
  - `returnAll: true`
  - Retries enabled: 3 attempts, 2-second wait
  - `onError: continueErrorOutput`
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from **Daily Inventory Check**; output to **Fetch Recent Orders**.
- **Version-specific requirements:** Node version `1`. Requires Shopify credentials using an Admin API access token.
- **Edge cases or failure types:**
  - Invalid or expired Shopify credential
  - Missing `read_products` scope
  - Rate limiting on large catalogs
  - Pagination/load issues for very large stores
  - Since `continueErrorOutput` is enabled, downstream nodes may continue with an error item rather than normal product data
- **Sub-workflow reference:** None.

#### Fetch Orders note
- **Type and role:** Sticky Note; explains order retrieval purpose.
- **Configuration choices:** Indicates orders from the last 7 days are intended for velocity calculations.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** None.
- **Edge cases or failure types:** None.
- **Sub-workflow reference:** None.

#### Fetch Recent Orders
- **Type and role:** `n8n-nodes-base.shopify`; retrieves orders.
- **Configuration choices:**
  - Operation: `getAll`
  - `returnAll: true`
  - Order status option: `any`
  - Retries enabled: 3 attempts, 2-second wait
  - `onError: continueErrorOutput`
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from **Fetch All Products**; output to **Compute Sales Velocity**.
- **Version-specific requirements:** Node version `1`. Requires Shopify credential with `read_orders`.
- **Edge cases or failure types:**
  - Invalid/expired Shopify credential
  - Missing `read_orders` scope
  - No date filter is actually configured, despite the note saying “last 7 days”
  - Cancelled, archived, test, or old orders may be included depending on Shopify node defaults and account data
  - Large order history may affect runtime and API consumption
  - With `continueErrorOutput`, downstream logic may receive unexpected structure
- **Sub-workflow reference:** None.

**Important implementation observation:**  
The documentation notes say this node fetches orders from the last 7 days, but the actual node configuration does **not** apply a date filter. The Code node still divides sold quantity by 7, so if older orders are included, the computed velocity will be inflated and inaccurate. This is the main logical mismatch in the current workflow.

---

## 2.4 Sales Velocity Computation

### Overview
This block merges Shopify products with ordered quantities per variant and derives the inventory metrics used for AI evaluation. It transforms raw catalog and order data into one item per variant.

### Nodes Involved
- Velocity note
- Compute Sales Velocity

### Node Details

#### Velocity note
- **Type and role:** Sticky Note; describes the calculation purpose.
- **Configuration choices:** Explains the join between products and orders and the metrics produced.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** None.
- **Edge cases or failure types:** None.
- **Sub-workflow reference:** None.

#### Compute Sales Velocity
- **Type and role:** `n8n-nodes-base.code`; custom JavaScript transformation.
- **Configuration choices:**
  - Reads all items from **Fetch Recent Orders**
  - Builds a `salesMap` keyed by `variant_id`
  - Iterates through all products and all variants from **Fetch All Products**
  - Produces one output item per variant
- **Key expressions or variables used:**
  - `$('Fetch Recent Orders').all()`
  - `$('Fetch All Products').all()`
  - `order.json.line_items || []`
  - `product.json.variants || []`
  - Calculated fields:
    - `product_title`
    - `variant_id`
    - `variant_title`
    - `sku`
    - `stock`
    - `sold_7d`
    - `velocity_per_day`
    - `days_until_stockout`
- **Calculation logic:**
  - `sold7d = total quantity sold for the variant`
  - `velocity = sold7d / 7`
  - `days_until_stockout = Math.floor(stock / velocity)` if velocity > 0, otherwise `999`
  - `velocity_per_day` rounded to 2 decimals
- **Input and output connections:** Triggered after **Fetch Recent Orders**; outputs to **Predict Stockout Risk**.
- **Version-specific requirements:** Node version `2`; compatible with the newer Code node execution model.
- **Edge cases or failure types:**
  - If Shopify nodes return error items due to `continueErrorOutput`, the code may fail because expected arrays/fields are missing
  - If products have no variants, they produce no output
  - If `inventory_quantity` is null or undefined, stock defaults to `0`
  - If `variant_id` in orders is missing, values may aggregate under `"undefined"`
  - No filtering for order date, refunds, cancellations, or fulfillment status
  - Uses `999` as a sentinel value for no-demand cases, which downstream logic must interpret correctly
- **Sub-workflow reference:** None.

---

## 2.5 AI Risk Assessment

### Overview
This block sends each variant’s metrics to GPT-4o through an AI Agent, enforces structured output, and returns a machine-usable reorder decision.

### Nodes Involved
- Predict note
- Predict Stockout Risk
- OpenAI — Predict
- Stockout Risk Schema

### Node Details

#### Predict note
- **Type and role:** Sticky Note; explains the AI purpose.
- **Configuration choices:** States that reorder quantity is based on 30 days of demand plus 20% safety stock.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** None.
- **Edge cases or failure types:** None.
- **Sub-workflow reference:** None.

#### Predict Stockout Risk
- **Type and role:** `@n8n/n8n-nodes-langchain.agent`; AI Agent orchestrating prompt + model + structured parser.
- **Configuration choices:**
  - Prompt is fully defined in the node
  - Uses product/variant metrics from the current item
  - Includes a system message defining risk thresholds:
    - critical: within 3 days
    - high: within 7 days
    - medium: 7–14 days
    - low: 14+ days
  - Explicitly asks for:
    - risk level
    - reorder quantity for 30 days demand + 20% safety stock
    - reasoning
    - immediate reorder decision
  - `hasOutputParser: true`
  - `returnIntermediateSteps: false`
- **Key expressions or variables used:**
  - `{{ $json.product_title }}`
  - `{{ $json.variant_title }}`
  - `{{ $json.sku }}`
  - `{{ $json.stock }}`
  - `{{ $json.sold_7d }}`
  - `{{ $json.velocity_per_day }}`
  - `{{ $json.days_until_stockout }}`
- **Input and output connections:** Input from **Compute Sales Velocity**; outputs to **Should Reorder?**. Receives model input from **OpenAI — Predict** and parser input from **Stockout Risk Schema**.
- **Version-specific requirements:** Node version `1.8`; requires n8n installation with LangChain AI nodes available.
- **Edge cases or failure types:**
  - OpenAI credential/model access errors
  - Model output not matching schema
  - Prompt ambiguity for low-sales or zero-sales products
  - AI latency or rate limits on many variants
  - If upstream values are malformed, the decision may be poor or the prompt may contain blanks
- **Sub-workflow reference:** None.

#### OpenAI — Predict
- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; language model backend for the AI Agent.
- **Configuration choices:**
  - Model: `gpt-4o`
  - Temperature: `0` for deterministic output
- **Key expressions or variables used:** None.
- **Input and output connections:** Connected as `ai_languageModel` to **Predict Stockout Risk**.
- **Version-specific requirements:** Node version `1.2`; requires an OpenAI credential and account access to `gpt-4o`.
- **Edge cases or failure types:**
  - Invalid API key
  - Model unavailable for the account
  - Rate limiting or quota exhaustion
  - Regional/network restrictions
- **Sub-workflow reference:** None.

#### Stockout Risk Schema
- **Type and role:** `@n8n/n8n-nodes-langchain.outputParserStructured`; structured output validation.
- **Configuration choices:**
  - Manual JSON schema requiring:
    - `risk_level`: enum `critical|high|medium|low`
    - `recommended_reorder_qty`: integer
    - `reasoning`: string
    - `should_reorder`: boolean
- **Key expressions or variables used:** Manual schema, no expressions.
- **Input and output connections:** Connected as `ai_outputParser` to **Predict Stockout Risk**.
- **Version-specific requirements:** Node version `1.2`; requires AI parser support in the installed n8n version.
- **Edge cases or failure types:**
  - AI output may fail parsing if values are missing or wrong type
  - Integer enforcement may reject decimal reorder quantities
  - If the model generates non-conforming values, node execution can fail
- **Sub-workflow reference:** None.

---

## 2.6 Reorder Decision Routing

### Overview
This block evaluates the AI’s boolean reorder recommendation and splits the flow into “email then log” or “log only”.

### Nodes Involved
- Should Reorder note
- Should Reorder?

### Node Details

#### Should Reorder note
- **Type and role:** Sticky Note; describes the branching behavior.
- **Configuration choices:** States the true/false routing outcome.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** None.
- **Edge cases or failure types:** None.
- **Sub-workflow reference:** None.

#### Should Reorder?
- **Type and role:** `n8n-nodes-base.if`; conditional branching node.
- **Configuration choices:**
  - Single boolean condition
  - Left value: `{{ $json.output.should_reorder }}`
  - Operation: boolean equals
  - Right value: `true`
  - Type validation mode: `loose`
- **Key expressions or variables used:**
  - `={{ $json.output.should_reorder }}`
- **Input and output connections:** Input from **Predict Stockout Risk**.
  - True branch -> **Send Reorder Email**
  - False branch -> **Log to Google Sheets**
- **Version-specific requirements:** Node version `2.2`.
- **Edge cases or failure types:**
  - If AI output parsing failed upstream, this node may never run
  - If `output.should_reorder` is missing, loose validation may still produce unexpected behavior
  - Non-boolean values may pass/fail differently under loose validation
- **Sub-workflow reference:** None.

---

## 2.7 Supplier Notification

### Overview
This block emails the supplier when a reorder is recommended. It formats product, stock, velocity, and AI decision data into an HTML message.

### Nodes Involved
- Email note
- Send Reorder Email

### Node Details

#### Email note
- **Type and role:** Sticky Note; explains the email step.
- **Configuration choices:** Notes that full variant details are sent.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** None.
- **Edge cases or failure types:** None.
- **Sub-workflow reference:** None.

#### Send Reorder Email
- **Type and role:** `n8n-nodes-base.gmail`; sends an outbound supplier email.
- **Configuration choices:**
  - Recipient: `user@example.com` placeholder
  - Subject includes product title and SKU
  - HTML body includes:
    - Product
    - SKU
    - Current stock
    - Sales velocity
    - Days until stockout
    - Risk level
    - Recommended reorder quantity
    - Reasoning
  - Retries enabled: 3 attempts, 2-second wait
  - `onError: continueErrorOutput`
- **Key expressions or variables used:**
  - Subject:
    - `{{ $('Compute Sales Velocity').item.json.product_title }}`
    - `{{ $('Compute Sales Velocity').item.json.sku }}`
  - Body:
    - `{{ $('Compute Sales Velocity').item.json.product_title }}`
    - `{{ $('Compute Sales Velocity').item.json.variant_title }}`
    - `{{ $('Compute Sales Velocity').item.json.sku }}`
    - `{{ $('Compute Sales Velocity').item.json.stock }}`
    - `{{ $('Compute Sales Velocity').item.json.velocity_per_day }}`
    - `{{ $('Compute Sales Velocity').item.json.days_until_stockout }}`
    - `{{ $json.output.risk_level.toUpperCase() }}`
    - `{{ $json.output.recommended_reorder_qty }}`
    - `{{ $json.output.reasoning }}`
- **Input and output connections:** Input from true branch of **Should Reorder?**; output to **Log to Google Sheets**.
- **Version-specific requirements:** Node version `2.1`; requires Gmail OAuth2 credentials configured in n8n.
- **Edge cases or failure types:**
  - Placeholder recipient not replaced
  - Gmail OAuth token expired or missing permissions
  - Gmail sending limits or anti-abuse restrictions
  - HTML rendering issues if fields are null
  - `toUpperCase()` will fail if `risk_level` is null or undefined
  - Because `continueErrorOutput` is enabled, logging may still continue after email failure
- **Sub-workflow reference:** None.

---

## 2.8 Decision Logging

### Overview
This block appends or updates a Google Sheets log for all variants, whether a reorder was triggered or not. It acts as the operational audit trail for the workflow.

### Nodes Involved
- Log note
- Log to Google Sheets

### Node Details

#### Log note
- **Type and role:** Sticky Note; describes logging behavior.
- **Configuration choices:** States that both triggered and skipped reorder decisions are tracked.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** None.
- **Edge cases or failure types:** None.
- **Sub-workflow reference:** None.

#### Log to Google Sheets
- **Type and role:** `n8n-nodes-base.googleSheets`; writes decision records to a sheet.
- **Configuration choices:**
  - Operation: `appendOrUpdate`
  - Target sheet name: `Reorder Log`
  - Document ID placeholder: `YOUR_REORDER_LOG_SHEET_ID`
  - Mapping mode: define fields explicitly
  - Logged columns:
    - SKU
    - Stock
    - Product
    - Sold 7d
    - Variant
    - Reasoning
    - Timestamp
    - Risk Level
    - Reorder Qty
    - Velocity/day
    - Days to Stockout
    - Reorder Triggered
  - Retries enabled: 3 attempts, 2-second wait
  - `onError: continueErrorOutput`
- **Key expressions or variables used:**
  - From **Compute Sales Velocity**:
    - `sku`
    - `stock`
    - `product_title`
    - `sold_7d`
    - `variant_title`
    - `velocity_per_day`
    - `days_until_stockout`
  - From **Predict Stockout Risk**:
    - `output.reasoning`
    - `output.risk_level`
    - `output.recommended_reorder_qty`
    - `output.should_reorder`
  - Timestamp:
    - `{{ DateTime.now().toISO() }}`
- **Input and output connections:**
  - Input from false branch of **Should Reorder?**
  - Input from **Send Reorder Email** after a true branch email is attempted
  - No downstream node
- **Version-specific requirements:** Node version `4.5`; requires Google Sheets credentials and a valid spreadsheet.
- **Edge cases or failure types:**
  - Placeholder document ID not replaced
  - Target sheet tab missing
  - Append/update behavior may require a matching key configuration depending on n8n version and node expectations
  - Credential access denied
  - Luxon `DateTime` availability depends on expression runtime support in n8n, though this is commonly available
  - Because the node accepts input from both branches, records may still be logged even when email fails
- **Sub-workflow reference:** None.

**Important implementation observation:**  
The node uses `appendOrUpdate`, but no explicit matching key is visible in the exported parameters. In some setups, this can behave unexpectedly or require additional configuration. If the intention is only to append historical records, `append` may be safer.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Overview | Sticky Note | Documents workflow purpose and main sequence |  |  | ## Inventory Reorder Prediction<br>Version 1.0.0 — E-Commerce<br><br>Runs daily, pulls all Shopify product variants with inventory levels, fetches orders from the last 7 days to compute sales velocity, then uses GPT-4o to predict stockout dates and determine whether to reorder. Reorder emails are sent to the supplier and all reorder decisions are logged to Google Sheets.<br><br>Flow: Schedule => Fetch Products (Shopify) => Fetch Orders (Shopify) => Compute Sales Velocity (Code) => Predict Stockout Risk (AI Agent) => Should Reorder? (IF) => Send Reorder Email + Log to Sheets |
| Prerequisites | Sticky Note | Lists required external systems and API access |  |  | ## Prerequisites<br>- Shopify store with Admin API access (Custom App)<br>- OpenAI API key (GPT-4o)<br>- Gmail account for reorder notifications<br>- Google Sheets for reorder log<br><br>Shopify setup: Admin => Apps and sales channels => Develop apps => Create an app => Enable read_products and read_orders API scopes => Install app => Copy Admin API access token. |
| Setup Required | Sticky Note | Lists configuration tasks before use |  |  | ## Setup Required<br>1. Fetch Products + Fetch Orders — connect 'Shopify Access Token API' credential<br>2. OpenAI sub-node — connect OpenAI credential<br>3. Send Reorder Email — connect Gmail OAuth2 credential, update supplier@example.com to real address<br>4. Log to Google Sheets — connect Google Sheets credential, replace YOUR_REORDER_LOG_SHEET_ID<br>5. Adjust reorder threshold in the Should Reorder? IF node (default: should_reorder = true from AI) |
| How It Works | Sticky Note | Explains runtime behavior step by step |  |  | ## How It Works<br>1. Runs daily at 06:00 UTC via cron schedule<br>2. Fetches all Shopify products and their variant inventory levels<br>3. Fetches all orders from the past 7 days and sums units sold per variant<br>4. Code node joins the two datasets to compute sales velocity (units/day) and estimated days until stockout<br>5. AI Agent evaluates each variant's risk level and recommended reorder quantity<br>6. IF should_reorder is true: sends email to supplier and logs to Google Sheets<br>7. IF should_reorder is false: logs to Google Sheets with low risk status |
| Daily Inventory Check | Schedule Trigger | Starts the workflow daily |  | Fetch All Products |  |
| Fetch Products note | Sticky Note | Documents product retrieval |  |  | Pulls all product variants with current inventory_quantity from Shopify |
| Fetch All Products | Shopify | Retrieves all Shopify products and variants | Daily Inventory Check | Fetch Recent Orders | Pulls all product variants with current inventory_quantity from Shopify |
| Fetch Orders note | Sticky Note | Documents order retrieval |  |  | Fetches all orders from the last 7 days to calculate sales velocity |
| Fetch Recent Orders | Shopify | Retrieves Shopify orders for sales analysis | Fetch All Products | Compute Sales Velocity | Fetches all orders from the last 7 days to calculate sales velocity |
| Velocity note | Sticky Note | Documents metric calculation block |  |  | Joins products + orders, computes velocity (units/day) and days until stockout per variant |
| Compute Sales Velocity | Code | Calculates sold units, daily velocity, and stockout estimate per variant | Fetch Recent Orders | Predict Stockout Risk | Joins products + orders, computes velocity (units/day) and days until stockout per variant |
| Predict note | Sticky Note | Documents AI decision block |  |  | AI Agent assesses stockout risk and recommends reorder quantity with 30-day + 20% safety buffer |
| Predict Stockout Risk | AI Agent | Produces structured reorder recommendations | Compute Sales Velocity; OpenAI — Predict; Stockout Risk Schema | Should Reorder? | AI Agent assesses stockout risk and recommends reorder quantity with 30-day + 20% safety buffer |
| OpenAI — Predict | OpenAI Chat Model | Provides GPT-4o model to the AI Agent |  | Predict Stockout Risk | AI Agent assesses stockout risk and recommends reorder quantity with 30-day + 20% safety buffer |
| Stockout Risk Schema | Structured Output Parser | Enforces machine-readable AI output schema |  | Predict Stockout Risk | AI Agent assesses stockout risk and recommends reorder quantity with 30-day + 20% safety buffer |
| Should Reorder note | Sticky Note | Documents branching logic |  |  | Routes to email supplier (true) or skip to log only (false) |
| Should Reorder? | IF | Branches on AI reorder decision | Predict Stockout Risk | Send Reorder Email; Log to Google Sheets | Routes to email supplier (true) or skip to log only (false) |
| Email note | Sticky Note | Documents supplier notification step |  |  | Sends reorder request to supplier with full variant details |
| Send Reorder Email | Gmail | Sends reorder request email to supplier | Should Reorder? | Log to Google Sheets | Sends reorder request to supplier with full variant details |
| Log note | Sticky Note | Documents audit logging |  |  | Logs all reorder decisions (triggered or skipped) to Google Sheets for tracking |
| Log to Google Sheets | Google Sheets | Records decisions and metrics in spreadsheet | Should Reorder?; Send Reorder Email |  | Logs all reorder decisions (triggered or skipped) to Google Sheets for tracking |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it something like:  
   **Predict Shopify stockouts with GPT-4o and email suppliers via Gmail**

2. **Add a Schedule Trigger node**
   - Node type: **Schedule Trigger**
   - Name: **Daily Inventory Check**
   - Set the schedule rule to **Cron**
   - Use cron expression: `0 6 * * *`
   - This runs daily at 06:00 UTC

3. **Add a Shopify node to fetch products**
   - Node type: **Shopify**
   - Name: **Fetch All Products**
   - Connect it from **Daily Inventory Check**
   - Resource: **Product**
   - Operation: **Get All**
   - Enable **Return All**
   - Credentials: connect a **Shopify Access Token API** credential
   - Recommended Shopify app scopes:
     - `read_products`
     - `read_orders`
   - Optional resilience settings:
     - Enable retry on fail
     - Max tries: 3
     - Wait between tries: 2000 ms
     - Error handling: continue on error output if you want partial workflow continuation

4. **Add a second Shopify node to fetch orders**
   - Node type: **Shopify**
   - Name: **Fetch Recent Orders**
   - Connect it from **Fetch All Products**
   - Operation: **Get All**
   - Enable **Return All**
   - In options, set status to **any**
   - Credentials: reuse the same Shopify credential
   - Retry/error handling same as above

5. **Important correction when rebuilding**
   - The exported workflow says “last 7 days” but does not actually filter orders by date.
   - If you want correct 7-day sales velocity, configure this node to request only orders created in the last 7 days if your installed Shopify node version supports that filter.
   - If the node does not support it directly, add a filtering step after retrieval or switch to an HTTP Request against Shopify Admin API with a created date filter.

6. **Add a Code node**
   - Node type: **Code**
   - Name: **Compute Sales Velocity**
   - Connect it from **Fetch Recent Orders**
   - Use JavaScript mode
   - Paste the following logic conceptually:
     - read all orders from **Fetch Recent Orders**
     - sum `line_items[].quantity` by `variant_id`
     - read all products from **Fetch All Products**
     - iterate through each variant
     - compute:
       - stock
       - sold over 7 days
       - velocity per day
       - days until stockout
     - emit one item per variant
   - The output fields should be:
     - `product_title`
     - `variant_id`
     - `variant_title`
     - `sku`
     - `stock`
     - `sold_7d`
     - `velocity_per_day`
     - `days_until_stockout`

7. **Use this Code node logic**
   - Recreate the script with the same logic:
     - Orders:
       - `const orders = $('Fetch Recent Orders').all();`
     - Products:
       - `const products = $('Fetch All Products').all();`
     - Sales map:
       - key by variant ID converted to string
     - Velocity:
       - `sold7d / 7`
     - Days to stockout:
       - `Math.floor(stock / velocity)` when velocity > 0
       - otherwise `999`
   - Keep defensive defaults such as:
     - `line_items || []`
     - `variants || []`
     - `inventory_quantity || 0`
     - `sku || ''`

8. **Add an AI Agent node**
   - Node type: **AI Agent**
   - Name: **Predict Stockout Risk**
   - Connect it from **Compute Sales Velocity**
   - Set prompt type to **Define**
   - In the main text prompt, include the current variant fields:
     - product title
     - variant title
     - SKU
     - stock
     - sold in last 7 days
     - sales velocity
     - days until stockout
   - Ask the model to:
     - assess risk level
     - recommend reorder quantity for 30 days plus 20% safety stock
     - decide whether immediate reorder is needed

9. **Configure the AI Agent system message**
   - Add a system message similar to:
     - You are an inventory analyst for an e-commerce business
     - critical = stockout within 3 days
     - high = within 7 days
     - medium = 7–14 days
     - low = 14+ days
     - reorder quantity should cover 30-day demand plus 20% safety stock
     - if stock is 0 and velocity is 0, classify as low risk
   - Disable intermediate steps unless you explicitly want trace output

10. **Add an OpenAI Chat Model node**
    - Node type: **OpenAI Chat Model**
    - Name: **OpenAI — Predict**
    - Connect it to the AI Agent using the **Language Model** connection
    - Model: **gpt-4o**
    - Temperature: **0**
    - Credentials: connect your **OpenAI API credential**

11. **Add a Structured Output Parser node**
    - Node type: **Structured Output Parser**
    - Name: **Stockout Risk Schema**
    - Connect it to the AI Agent using the **Output Parser** connection
    - Choose **Manual Schema**
    - Define a schema with:
      - `risk_level`: string enum of `critical`, `high`, `medium`, `low`
      - `recommended_reorder_qty`: integer
      - `reasoning`: string
      - `should_reorder`: boolean
    - Mark all fields as required

12. **Add an IF node**
    - Node type: **IF**
    - Name: **Should Reorder?**
    - Connect it from **Predict Stockout Risk**
    - Add one boolean condition:
      - Left value: `{{ $json.output.should_reorder }}`
      - Operator: **equals**
      - Right value: `true`
    - Type validation can remain **loose** to match the source workflow

13. **Add a Gmail node for supplier notification**
    - Node type: **Gmail**
    - Name: **Send Reorder Email**
    - Connect it to the **true** output of **Should Reorder?**
    - Operation: send email
    - Credentials: connect **Gmail OAuth2**
    - Replace the placeholder recipient with the real supplier email address
    - Set subject to include product title and SKU
    - Use HTML body containing:
      - product and variant
      - SKU
      - current stock
      - velocity/day
      - days until stockout
      - AI risk level
      - recommended reorder quantity
      - AI reasoning
      - request for delivery timeline confirmation

14. **Configure Gmail expressions**
    - Reference the Code node for variant fields, for example:
      - `$('Compute Sales Velocity').item.json.product_title`
      - `$('Compute Sales Velocity').item.json.variant_title`
      - `$('Compute Sales Velocity').item.json.sku`
    - Reference the AI Agent output for:
      - `risk_level`
      - `recommended_reorder_qty`
      - `reasoning`
    - If you use `.toUpperCase()` on `risk_level`, ensure it is always present

15. **Add a Google Sheets node**
    - Node type: **Google Sheets**
    - Name: **Log to Google Sheets**
    - Connect it to:
      - the **false** output of **Should Reorder?**
      - the output of **Send Reorder Email**
    - Credentials: connect a **Google Sheets** credential
    - Operation: either:
      - **Append or Update** to match the exported workflow, or
      - **Append** if you want a pure audit trail and simpler behavior
    - Spreadsheet ID: replace `YOUR_REORDER_LOG_SHEET_ID`
    - Sheet/tab name: `Reorder Log`

16. **Map the Google Sheets columns**
    - Map these fields:
      - `Timestamp` -> current ISO timestamp
      - `Product` -> product title
      - `Variant` -> variant title
      - `SKU` -> SKU
      - `Stock` -> stock
      - `Sold 7d` -> sold quantity
      - `Velocity/day` -> velocity
      - `Days to Stockout` -> days until stockout
      - `Risk Level` -> AI risk level
      - `Reorder Qty` -> AI reorder quantity
      - `Reasoning` -> AI reasoning
      - `Reorder Triggered` -> AI boolean decision

17. **Use expressions for logging**
    - From **Compute Sales Velocity**:
      - `product_title`
      - `variant_title`
      - `sku`
      - `stock`
      - `sold_7d`
      - `velocity_per_day`
      - `days_until_stockout`
    - From **Predict Stockout Risk**:
      - `output.reasoning`
      - `output.risk_level`
      - `output.recommended_reorder_qty`
      - `output.should_reorder`
    - Timestamp:
      - `{{ DateTime.now().toISO() }}`

18. **Enable retry and error handling for external service nodes**
    - For **Fetch All Products**
    - **Fetch Recent Orders**
    - **Send Reorder Email**
    - **Log to Google Sheets**
   Use:
   - Retry on fail: enabled
   - Max tries: 3
   - Wait between tries: 2000 ms
   - Optionally `continueErrorOutput` if you want logging or partial continuation after nonfatal failures

19. **Add optional sticky notes for maintainability**
    - Recreate notes for:
      - overview
      - prerequisites
      - setup required
      - how it works
      - fetch products
      - fetch orders
      - velocity
      - predict
      - should reorder
      - email
      - log

20. **Validate credentials before activation**
    - Shopify credential: token valid and scopes enabled
    - OpenAI credential: API key active and model access confirmed
    - Gmail OAuth2: sending mailbox authorized
    - Google Sheets: spreadsheet accessible and tab exists

21. **Test with manual execution**
    - Run the workflow manually
    - Confirm:
      - products are returned
      - orders are returned
      - Code node outputs one item per variant
      - AI output is parsed into `output.*`
      - IF branch splits correctly
      - Gmail sends only when `should_reorder = true`
      - Google Sheets receives records in both branch scenarios

22. **Activate the workflow**
    - Once validated, activate it so the schedule trigger runs daily.

### Sub-workflow setup
This workflow does **not** invoke any sub-workflows and does not expose itself as a sub-workflow.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Version stated in canvas note: Inventory Reorder Prediction, Version 1.0.0 — E-Commerce | Internal workflow branding |
| Shopify setup guidance included in the workflow: Admin => Apps and sales channels => Develop apps => Create an app => Enable `read_products` and `read_orders` scopes => Install app => Copy Admin API access token | Shopify Admin setup |
| Placeholder values must be replaced before production use: supplier email and Google Sheets document ID | Operational setup |
| Main design caveat: the workflow text says orders from the last 7 days are used, but the current Shopify order node does not show an actual date filter | Important logic correction |
| Main design caveat: Google Sheets uses `appendOrUpdate`, which may require additional match configuration depending on node version and intended behavior | Logging reliability consideration |