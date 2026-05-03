Monitor D2C inventory, forecast demand with GPT-4o, and send POs via Google Sheets and Gmail

https://n8nworkflows.xyz/workflows/monitor-d2c-inventory--forecast-demand-with-gpt-4o--and-send-pos-via-google-sheets-and-gmail-13961


# Monitor D2C inventory, forecast demand with GPT-4o, and send POs via Google Sheets and Gmail

# 1. Workflow Overview

This workflow implements an AI-assisted D2C supply chain control loop using Google Sheets, Gmail, HTTP, Code nodes, and OpenAI GPT-4o. It simulates daily sales, maintains a central inventory sheet, detects stock risk and dead stock, automatically raises purchase orders, and produces weekly AI-generated inventory and demand recommendations.

Primary use cases:
- Daily stock monitoring for D2C catalogs
- Automated stockout alerting
- Automatic purchase order generation for critical SKUs
- Weekly review of slow-moving inventory
- Weekly 30-day demand forecasting using GPT-4o

The workflow is organized into five logical blocks, each with its own trigger.

---

## 1.1 Daily Product Ingestion and Sales Simulation

At 8:00 AM, the workflow fetches product data from DummyJSON, simulates one day of sales per product, appends those sales to a sales log sheet, and appends inventory records to the inventory master sheet.

---

## 1.2 Daily Inventory Health Calculation and Alerting

At 8:05 AM, the workflow reads inventory and recent sales, calculates 7-day sales velocity and stock coverage, assigns health flags, updates the inventory sheet, and sends alert emails for critical and problematic SKUs.

---

## 1.3 Daily Purchase Order Automation

At 8:10 AM, the workflow reads flagged inventory, filters stockout-risk SKUs, calculates reorder quantities, skips unusable zero-velocity cases, emails suppliers a PO, logs the PO, and sends a confirmation notification.

---

## 1.4 Weekly Dead Stock AI Strategy Review

Every Sunday at 9:00 AM, the workflow reads yellow-flagged SKUs, asks GPT-4o for remediation actions, parses structured JSON recommendations, and emails a weekly digest grouped by urgency.

---

## 1.5 Weekly 30-Day Demand Forecasting

Every Sunday at 9:30 AM, the workflow aggregates 90 days of sales, merges this with inventory context, builds a forecasting prompt for GPT-4o, parses structured forecast output, filters actionable SKUs, and emails a demand forecast report.

---

# 2. Block-by-Block Analysis

## 2.1 Daily Product Ingestion and Sales Simulation

### Overview
This block initializes the operational dataset for the rest of the workflow. It fetches a sample catalog from DummyJSON, simulates daily sales activity, writes sales transactions to the Sales Log sheet, and writes product inventory rows to the Inventory Master sheet.

### Nodes Involved
- 🕐 Daily 8AM Trigger
- Fetch Products (DummyJSON)
- Simulate Daily Sales
- Log to Sales Log
- Write to Inventory Master

### Node Details

#### 1) 🕐 Daily 8AM Trigger
- **Type and role:** `scheduleTrigger`; entry point for the daily simulation run.
- **Configuration choices:** Runs every day at 8:00.
- **Key expressions or variables used:** None.
- **Input / output connections:** No input; outputs to `Fetch Products (DummyJSON)`.
- **Version-specific requirements:** Uses schedule trigger typeVersion 1.3.
- **Edge cases / failures:** Trigger timezone depends on instance settings. If server timezone is unexpected, execution time may shift.
- **Sub-workflow reference:** None.

#### 2) Fetch Products (DummyJSON)
- **Type and role:** `httpRequest`; pulls sample products from DummyJSON.
- **Configuration choices:** Sends GET request to `https://dummyjson.com/products?limit=20`.
- **Key expressions or variables used:** None.
- **Input / output connections:** Input from `🕐 Daily 8AM Trigger`; output to `Simulate Daily Sales`.
- **Version-specific requirements:** HTTP Request node typeVersion 4.3.
- **Edge cases / failures:** API downtime, network error, schema changes, or missing `products` array will break downstream code.
- **Sub-workflow reference:** None.

#### 3) Simulate Daily Sales
- **Type and role:** `code`; transforms API products into simulated SKU sales events.
- **Configuration choices:** Reads `items[0].json.products`, generates random `units_sold` between 1 and 15, computes `remaining_stock`, and enriches each item with supplier and replenishment metadata.
- **Key expressions or variables used:**
  - `items[0].json.products`
  - `Math.floor(Math.random() * 15) + 1`
  - `new Date().toISOString().split('T')[0]`
- **Input / output connections:** Input from `Fetch Products (DummyJSON)`; output to `Log to Sales Log`.
- **Version-specific requirements:** Code node typeVersion 2.
- **Edge cases / failures:**
  - Fails if `products` is absent.
  - Can produce duplicate SKUs daily if source IDs repeat and append logic is retained.
  - `current_stock` is copied from source stock, not from evolving sheet state.
- **Sub-workflow reference:** None.

#### 4) Log to Sales Log
- **Type and role:** `googleSheets`; appends daily sales rows to the Sales Log tab.
- **Configuration choices:** Append operation into sheet `Sales Log` in spreadsheet `D2C Supply Chain Brain`. Maps:
  - `date`
  - `sku_id`
  - `product_name`
  - `units_sold`
  - `remaining_stock`
- **Key expressions or variables used:** Uses current item fields such as `{{ $json.date }}` and `{{ $json.units_sold }}`.
- **Input / output connections:** Input from `Simulate Daily Sales`; output to `Write to Inventory Master`.
- **Version-specific requirements:** Google Sheets node typeVersion 4.7.
- **Edge cases / failures:**
  - OAuth expiration
  - Missing sheet/tab
  - Column mismatch
  - Rate limits on larger datasets
- **Sub-workflow reference:** None.

#### 5) Write to Inventory Master
- **Type and role:** `googleSheets`; appends inventory rows into Inventory Master.
- **Configuration choices:** Append operation into `Inventory Master`. Maps SKU metadata from `Simulate Daily Sales`, including fixed `status = active`.
- **Key expressions or variables used:**
  - `$('Simulate Daily Sales').item.json.sku_id`
  - `$('Simulate Daily Sales').item.json.product_name`
  - similar references for category, stock, reorder point, lead time, supplier email
- **Input / output connections:** Input from `Log to Sales Log`; no downstream node.
- **Version-specific requirements:** Google Sheets node typeVersion 4.7.
- **Edge cases / failures:**
  - This appends instead of updating, so repeated runs can create duplicate inventory records for the same SKU.
  - Downstream update nodes rely on `sku_id` matching, so duplicates can create ambiguous state.
- **Sub-workflow reference:** None.

---

## 2.2 Daily Inventory Health Calculation and Alerting

### Overview
This block evaluates each SKU’s stock health based on the last 7 days of sales. It computes average daily sales and days remaining, flags SKUs, updates the inventory sheet, and emails suppliers for stockout or dead-stock conditions.

### Nodes Involved
- Daily 8:05AM Trigger
- Read Inventory Master
- Read Sales Log
- Calculate Velocity & Flags
- IF Stockout Risk
- Alert — Stockout Risk
- Update Flags (Stockout)
- IF Dead or Overstock
- Alert — Dead Stock
- Update Flags (Dead Stock)

### Node Details

#### 1) Daily 8:05AM Trigger
- **Type and role:** `scheduleTrigger`; starts daily stock health analysis.
- **Configuration choices:** Runs daily at 8:05.
- **Input / output connections:** No input; outputs to `Read Inventory Master`.
- **Version-specific requirements:** typeVersion 1.1.
- **Edge cases / failures:** Sensitive to instance timezone.
- **Sub-workflow reference:** None.

#### 2) Read Inventory Master
- **Type and role:** `googleSheets`; retrieves inventory rows.
- **Configuration choices:** Reads `Inventory Master` from the Google Sheet.
- **Key expressions or variables used:** None.
- **Input / output connections:** Input from trigger; output to `Read Sales Log`.
- **Version-specific requirements:** typeVersion 4.2.
- **Edge cases / failures:** Missing sheet, malformed rows, blank `sku_id`, duplicates caused by append behavior.
- **Sub-workflow reference:** None.

#### 3) Read Sales Log
- **Type and role:** `googleSheets`; retrieves historical sales rows.
- **Configuration choices:** Reads `Sales Log`.
- **Input / output connections:** Input from `Read Inventory Master`; output to `Calculate Velocity & Flags`.
- **Version-specific requirements:** typeVersion 4.2.
- **Edge cases / failures:** Large history may affect performance; invalid dates may break comparisons.
- **Sub-workflow reference:** None.

#### 4) Calculate Velocity & Flags
- **Type and role:** `code`; computes 7-day demand metrics and assigns stock health flags.
- **Configuration choices:**
  - Loads all rows from `Read Inventory Master` and `Read Sales Log`
  - Filters sales in last 7 days
  - Computes `totalSold`, `avgDailySales`, and `daysRemaining`
  - Sets flags:
    - `🔴 Stockout Risk` if `daysRemaining < 14`
    - `🟡 Dead Stock` if `totalSold === 0`
    - `🟡 Overstock` if `daysRemaining > 60`
    - otherwise `🟢 Healthy`
- **Key expressions or variables used:**
  - `$('Read Inventory Master').all()`
  - `$('Read Sales Log').all()`
  - `new Date(today - 7 * 24 * 60 * 60 * 1000)`
  - `avgDailySales.toFixed(2)`
- **Input / output connections:** Input via upstream flow and internal node references; output to `IF Stockout Risk`.
- **Version-specific requirements:** Code node typeVersion 2.
- **Edge cases / failures:**
  - Date parsing may fail on malformed sheet values.
  - Duplicated inventory rows inflate processing.
  - `current_stock` string conversion may yield `NaN`.
  - `daysRemaining` uses floor division and a sentinel `999` when avg sales is zero.
- **Sub-workflow reference:** None.

#### 5) IF Stockout Risk
- **Type and role:** `if`; routes red-flagged SKUs.
- **Configuration choices:** String contains test: `$json.flag` contains `🔴`.
- **Input / output connections:** Input from `Calculate Velocity & Flags`; true output to `Alert — Stockout Risk`, false output to `IF Dead or Overstock`.
- **Version-specific requirements:** IF node typeVersion 2.
- **Edge cases / failures:** If `flag` is missing, expression resolves empty and item will go false branch.
- **Sub-workflow reference:** None.

#### 6) Alert — Stockout Risk
- **Type and role:** `gmail`; sends stockout warning emails.
- **Configuration choices:**
  - To: `{{ $json.supplier_email }}`
  - Subject: `🔴 Stockout Alert — <product>`
  - Text email body with product, category, stock, days remaining, and avg daily sales
- **Key expressions or variables used:** `{{ $json.product_name }}`, `{{ $json.days_remaining }}`, etc.
- **Input / output connections:** Input from `IF Stockout Risk`; output to `Update Flags (Stockout)`.
- **Version-specific requirements:** Gmail node typeVersion 2.1; requires Gmail OAuth2.
- **Edge cases / failures:**
  - Invalid or blank supplier email
  - Gmail auth/token issues
  - Sending limits
- **Sub-workflow reference:** None.

#### 7) Update Flags (Stockout)
- **Type and role:** `googleSheets`; updates flagged fields in Inventory Master.
- **Configuration choices:** Update operation matching on `sku_id`; writes `flag`, `sku_id`, `product_name`, `days_remaining`, `avg_daily_sales`.
- **Key expressions or variables used:** references to `$('IF Stockout Risk').item.json...`
- **Input / output connections:** Input from `Alert — Stockout Risk`; terminal node in this branch.
- **Version-specific requirements:** typeVersion 4.2.
- **Edge cases / failures:**
  - Duplicate `sku_id` records can update unintended rows.
  - If `sku_id` does not exist, no row may be updated.
- **Sub-workflow reference:** None.

#### 8) IF Dead or Overstock
- **Type and role:** `if`; routes yellow-flagged SKUs.
- **Configuration choices:** String contains test: `$json.flag` contains `🟡`.
- **Input / output connections:** Input from false branch of `IF Stockout Risk`; true branch to `Alert — Dead Stock`.
- **Version-specific requirements:** typeVersion 2.
- **Edge cases / failures:** Both dead stock and overstock are merged here, but only one email template is used.
- **Sub-workflow reference:** None.

#### 9) Alert — Dead Stock
- **Type and role:** `gmail`; sends warning emails for yellow-flagged SKUs.
- **Configuration choices:**
  - To: supplier email
  - Subject: `🟡 Dead Stock Warning — <product>`
  - Body says “Last 7 days sold: 0 units”
- **Input / output connections:** Input from `IF Dead or Overstock`; output to `Update Flags (Dead Stock)`.
- **Version-specific requirements:** Gmail typeVersion 2.1.
- **Edge cases / failures:**
  - Logic also catches overstock SKUs, but message content specifically describes dead stock, which can be misleading.
- **Sub-workflow reference:** None.

#### 10) Update Flags (Dead Stock)
- **Type and role:** `googleSheets`; updates yellow-flagged rows in Inventory Master.
- **Configuration choices:** Update operation matching on `sku_id`; writes `flag`, `sku_id`, `days_remaining`, `avg_daily_sales`.
- **Key expressions or variables used:** references to `$('IF Dead or Overstock').item.json...`
- **Input / output connections:** Input from `Alert — Dead Stock`; terminal.
- **Version-specific requirements:** typeVersion 4.2.
- **Edge cases / failures:** Same duplicate/matching concerns as stockout update.
- **Sub-workflow reference:** None.

---

## 2.3 Daily Purchase Order Automation

### Overview
This block automates PO generation for red-flagged SKUs. It calculates replenishment quantities using lead time and buffer stock, sends the PO by Gmail, writes a PO tracking entry, and sends a confirmation message.

### Nodes Involved
- Daily 8:10AM Trigger
- Read Inventory Master1
- Filter Stockout SKUs
- Calculate Reorder Quantity
- Skip Zero-Velocity SKUs
- Send PO to Supplier
- Log PO to Tracker
- Notify — PO Raised

### Node Details

#### 1) Daily 8:10AM Trigger
- **Type and role:** `scheduleTrigger`; starts PO automation.
- **Configuration choices:** Runs daily at 8:10.
- **Input / output connections:** No input; outputs to `Read Inventory Master1`.
- **Version-specific requirements:** typeVersion 1.1.
- **Edge cases / failures:** Depends on timezone and assumes flag-updating block already finished.
- **Sub-workflow reference:** None.

#### 2) Read Inventory Master1
- **Type and role:** `googleSheets`; re-reads inventory state for PO generation.
- **Configuration choices:** Reads Inventory Master.
- **Input / output connections:** Input from trigger; output to `Filter Stockout SKUs`.
- **Version-specific requirements:** typeVersion 4.2.
- **Edge cases / failures:** If prior block has not updated flags yet, PO decisions may be stale.
- **Sub-workflow reference:** None.

#### 3) Filter Stockout SKUs
- **Type and role:** `if`; keeps only red-flagged SKUs.
- **Configuration choices:** `$json.flag` contains `🔴`.
- **Input / output connections:** Input from `Read Inventory Master1`; true output to `Calculate Reorder Quantity`.
- **Version-specific requirements:** typeVersion 2.
- **Edge cases / failures:** Missing `flag` leads to exclusion.
- **Sub-workflow reference:** None.

#### 4) Calculate Reorder Quantity
- **Type and role:** `code`; computes reorder quantity and PO metadata.
- **Configuration choices:**
  - Reads `avg_daily_sales`, `lead_time_days`, `current_stock`
  - Skips item if `avg_daily_sales === 0`
  - Buffer stock = 7 days of average sales
  - Reorder quantity = `(avg_daily_sales * lead_time_days) + buffer - current_stock`
  - Minimum fallback quantity = 10 if result <= 0
  - Generates PO number as `PO-<sku>-<timestamp>`
- **Key expressions or variables used:**
  - `Number(item.json.avg_daily_sales)`
  - `Math.ceil(avgDailySales * 7)`
  - `Date.now()`
- **Input / output connections:** Input from `Filter Stockout SKUs`; output to `Skip Zero-Velocity SKUs`.
- **Version-specific requirements:** Code node typeVersion 2.
- **Edge cases / failures:**
  - If prior sheet update never populated `avg_daily_sales`, item is marked skipped.
  - `current_stock` or `lead_time_days` may be non-numeric.
  - Timestamp-based PO numbers can collide under rare same-millisecond cases across same SKU.
- **Sub-workflow reference:** None.

#### 5) Skip Zero-Velocity SKUs
- **Type and role:** `filter`; excludes items marked `skipped=true`.
- **Configuration choices:** keeps items where `$json.skipped == false`.
- **Input / output connections:** Input from `Calculate Reorder Quantity`; output to `Send PO to Supplier`.
- **Version-specific requirements:** Filter node typeVersion 2.3.
- **Edge cases / failures:** Loose type validation helps with boolean/string inconsistencies, but malformed values may still pass unexpectedly.
- **Sub-workflow reference:** None.

#### 6) Send PO to Supplier
- **Type and role:** `gmail`; sends PO email.
- **Configuration choices:**
  - `sendTo` is set to `={{ "forreferal04@gmail.com" || !$json.supplier_email }}`
  - Subject includes PO number and product name
  - Body includes PO details, quantity, stock, and expected stockout timing
- **Key expressions or variables used:** `{{ $json.po_number }}`, `{{ $json.reorder_qty }}`, etc.
- **Input / output connections:** Input from `Skip Zero-Velocity SKUs`; output to `Log PO to Tracker`.
- **Version-specific requirements:** Gmail typeVersion 2.1.
- **Edge cases / failures:**
  - The `sendTo` expression is effectively hardcoded to `forreferal04@gmail.com` because a non-empty string is always truthy. It does not actually use `supplier_email`.
  - Gmail auth and rate-limit issues.
- **Sub-workflow reference:** None.

#### 7) Log PO to Tracker
- **Type and role:** `googleSheets`; appends the PO to the PO Tracker sheet.
- **Configuration choices:** Append operation to `PO Tracker`, mapping:
  - `po_number`
  - `sku_id ` (note trailing space in column ID)
  - `product_name`
  - `quantity`
  - `supplier_email`
  - `status = Sent`
  - `date_raised`
- **Key expressions or variables used:** references `$('Calculate Reorder Quantity').item.json...`
- **Input / output connections:** Input from `Send PO to Supplier`; output to `Notify — PO Raised`.
- **Version-specific requirements:** typeVersion 4.7.
- **Edge cases / failures:**
  - The sheet schema contains a column ID `sku_id ` with a trailing space. This must match the sheet exactly or logging fails/misaligns.
  - Append creates a new PO row every run.
- **Sub-workflow reference:** None.

#### 8) Notify — PO Raised
- **Type and role:** `gmail`; sends PO raised confirmation.
- **Configuration choices:** Sends to `{{ $json.supplier_email }}` with summary details.
- **Key expressions or variables used:** Mixes current item fields and explicit references to `Calculate Reorder Quantity`.
- **Input / output connections:** Input from `Log PO to Tracker`; terminal.
- **Version-specific requirements:** Gmail typeVersion 2.1.
- **Edge cases / failures:**
  - If supplier email is blank, notification fails.
  - May notify supplier instead of an internal team member, depending on business intent.
- **Sub-workflow reference:** None.

---

## 2.4 Weekly Dead Stock AI Strategy Review

### Overview
This block sends yellow-flagged SKUs to GPT-4o for strategic recommendations. It expects a strict JSON response, parses it, groups the recommendations by urgency, and emails a weekly digest.

### Nodes Involved
- Weekly Sunday 9AM Trigger
- Read Inventory Master2
- Filter Dead & Overstock SKUs
- Build GPT Prompt
- Inventory Strategist
- Parse AI Recommendations
- Build Weekly Digest Email
- Send Weekly Digest

### Node Details

#### 1) Weekly Sunday 9AM Trigger
- **Type and role:** `scheduleTrigger`; starts the weekly yellow-flag AI review.
- **Configuration choices:** Weekly at 9:00 AM.
- **Input / output connections:** No input; output to `Read Inventory Master2`.
- **Version-specific requirements:** typeVersion 1.1.
- **Edge cases / failures:** Uses weekly interval settings; exact weekday behavior depends on schedule configuration and instance locale interpretation.
- **Sub-workflow reference:** None.

#### 2) Read Inventory Master2
- **Type and role:** `googleSheets`; reads current inventory records.
- **Configuration choices:** Reads `Inventory Master`.
- **Input / output connections:** Input from trigger; output to `Filter Dead & Overstock SKUs`.
- **Version-specific requirements:** typeVersion 4.2.
- **Edge cases / failures:** Duplicate SKUs or missing flags affect quality of AI prompt.
- **Sub-workflow reference:** None.

#### 3) Filter Dead & Overstock SKUs
- **Type and role:** `if`; selects only yellow-flagged items.
- **Configuration choices:** `$json.flag` contains `🟡`.
- **Input / output connections:** Input from `Read Inventory Master2`; true output to `Build GPT Prompt`.
- **Version-specific requirements:** typeVersion 2.
- **Edge cases / failures:** Overstocks and dead stock are treated identically in selection.
- **Sub-workflow reference:** None.

#### 4) Build GPT Prompt
- **Type and role:** `code`; creates a structured prompt for GPT-4o.
- **Configuration choices:** Builds an array of SKU fields and instructs the model to return JSON:
  - `recommendations[]`
  - `sku_id`
  - `product_name`
  - `flag`
  - `action`
  - `discount_percent`
  - `reason`
  - `urgency`
- **Key expressions or variables used:** `items.map(...)`, `JSON.stringify(skus, null, 2)`
- **Input / output connections:** Input from `Filter Dead & Overstock SKUs`; output to `Inventory Strategist`.
- **Version-specific requirements:** Code node typeVersion 2.
- **Edge cases / failures:** Large SKU lists may create oversized prompts.
- **Sub-workflow reference:** None.

#### 5) Inventory Strategist
- **Type and role:** OpenAI LangChain node; sends the prompt to GPT-4o.
- **Configuration choices:**
  - Model: `gpt-4o`
  - Input content: `{{ $json.prompt }}`
  - No built-in tools
- **Input / output connections:** Input from `Build GPT Prompt`; output to `Parse AI Recommendations`.
- **Version-specific requirements:** `@n8n/n8n-nodes-langchain.openAi` typeVersion 2.1; requires OpenAI credentials and compatible n8n version with LangChain nodes installed.
- **Edge cases / failures:**
  - API auth errors
  - quota/rate limits
  - malformed/non-JSON model output
  - latency/timeouts
- **Sub-workflow reference:** None.

#### 6) Parse AI Recommendations
- **Type and role:** `code`; parses GPT output into separate items.
- **Configuration choices:** Extracts `items[0].json.output[0].content[0].text`, strips Markdown fences, parses JSON, returns one item per recommendation.
- **Key expressions or variables used:**
  - `response.replace(/```json|```/g, '')`
  - `JSON.parse(cleaned)`
- **Input / output connections:** Input from `Inventory Strategist`; output to `Build Weekly Digest Email`.
- **Version-specific requirements:** Code node typeVersion 2.
- **Edge cases / failures:**
  - Any deviation from exact JSON breaks `JSON.parse`
  - Empty or partial model output fails parsing
- **Sub-workflow reference:** None.

#### 7) Build Weekly Digest Email
- **Type and role:** `code`; builds a text digest grouped by urgency.
- **Configuration choices:** Splits into high, medium, low urgency sections, includes action and optional discount percentage.
- **Key expressions or variables used:** `recs.filter(...)`, `r.json.action.toUpperCase()`
- **Input / output connections:** Input from `Parse AI Recommendations`; output to `Send Weekly Digest`.
- **Version-specific requirements:** Code node typeVersion 2.
- **Edge cases / failures:** Empty recommendations produce a sparse but valid email.
- **Sub-workflow reference:** None.

#### 8) Send Weekly Digest
- **Type and role:** `gmail`; sends the weekly digest email.
- **Configuration choices:**
  - To: literal placeholder `your_email_id`
  - Subject includes total flagged SKU count
  - Body uses `email_body`
- **Key expressions or variables used:** `{{ $json.total_flagged }}`
- **Input / output connections:** Input from `Build Weekly Digest Email`; terminal.
- **Version-specific requirements:** Gmail typeVersion 2.1.
- **Edge cases / failures:**
  - Must replace placeholder recipient before production use.
- **Sub-workflow reference:** None.

---

## 2.5 Weekly 30-Day Demand Forecasting

### Overview
This block aggregates 90 days of sales history, merges it with inventory data, asks GPT-4o for a 30-day SKU forecast, parses the output, filters actionable cases, formats an email report, and sends it.

### Nodes Involved
- Weekly Sunday 9:30AM Trigger
- Read sales log
- Read Inventory Master3
- Aggregate 90 Day Sales
- Build Forecast Prompt
- Demand Forecaster
- Parse Forecast Response
- Filter Actionable SKUs
- Build Forecast Email
- Send Forecast Report

### Node Details

#### 1) Weekly Sunday 9:30AM Trigger
- **Type and role:** `scheduleTrigger`; starts weekly forecasting.
- **Configuration choices:** Weekly at 9:30 AM.
- **Input / output connections:** No input; output to `Read sales log`.
- **Version-specific requirements:** typeVersion 1.1.
- **Edge cases / failures:** Same schedule interpretation considerations as other weekly triggers.
- **Sub-workflow reference:** None.

#### 2) Read sales log
- **Type and role:** `googleSheets`; reads sales history.
- **Configuration choices:** Reads `Sales Log`.
- **Input / output connections:** Input from trigger; output to `Read Inventory Master3`.
- **Version-specific requirements:** typeVersion 4.2.
- **Edge cases / failures:** Large data volume may affect performance.
- **Sub-workflow reference:** None.

#### 3) Read Inventory Master3
- **Type and role:** `googleSheets`; reads inventory records.
- **Configuration choices:** Reads `Inventory Master`.
- **Input / output connections:** Input from `Read sales log`; output to `Aggregate 90 Day Sales`.
- **Version-specific requirements:** typeVersion 4.2.
- **Edge cases / failures:** Duplicate SKU records can distort merged data.
- **Sub-workflow reference:** None.

#### 4) Aggregate 90 Day Sales
- **Type and role:** `code`; builds per-SKU sales aggregates for the last 90 days.
- **Configuration choices:**
  - Loads all sales and inventory rows by node reference
  - Filters sales to 90-day window
  - Groups by `sku_id`
  - Computes `total_sold_90d` and `avg_daily_sales_90d`
  - Merges sales with inventory fields and default lead time 7
- **Key expressions or variables used:**
  - `$('Read sales log').all()`
  - `$('Read Inventory Master3').all()`
  - `new Date(today - 90 * 24 * 60 * 60 * 1000)`
- **Input / output connections:** Input from `Read Inventory Master3`; output to `Build Forecast Prompt`.
- **Version-specific requirements:** Code node typeVersion 2.
- **Edge cases / failures:**
  - `avg_daily_sales_90d` is calculated using active sales days count, not 90 calendar days, which may overstate demand for intermittent sales.
  - Invalid dates or SKU type mismatches can distort aggregation.
- **Sub-workflow reference:** None.

#### 5) Build Forecast Prompt
- **Type and role:** `code`; builds GPT-4o forecasting prompt.
- **Configuration choices:** Instructs model to return JSON:
  - `forecast_month`
  - `forecasts[]`
  - SKU fields plus predicted demand, stock needed, action, seasonality note, confidence
- **Key expressions or variables used:**
  - `const month = new Date().toLocaleString('default', { month: 'long' })`
  - `JSON.stringify(skus, null, 2)`
- **Input / output connections:** Input from `Aggregate 90 Day Sales`; output to `Demand Forecaster`.
- **Version-specific requirements:** Code node typeVersion 2.
- **Edge cases / failures:** Prompt may become large if many SKUs are included.
- **Sub-workflow reference:** None.

#### 6) Demand Forecaster
- **Type and role:** OpenAI LangChain node; requests forecast from GPT-4o.
- **Configuration choices:** Model `gpt-4o`; sends prompt text as content.
- **Input / output connections:** Input from `Build Forecast Prompt`; output to `Parse Forecast Response`.
- **Version-specific requirements:** LangChain OpenAI node typeVersion 2.1.
- **Edge cases / failures:** Same as other OpenAI node: auth, quota, timeout, malformed output.
- **Sub-workflow reference:** None.

#### 7) Parse Forecast Response
- **Type and role:** `code`; parses forecast JSON into one item per forecasted SKU.
- **Configuration choices:** Extracts AI text output, removes Markdown code fences, parses JSON, and injects `forecast_month` into each item.
- **Key expressions or variables used:** `JSON.parse(cleaned)`
- **Input / output connections:** Input from `Demand Forecaster`; output to `Filter Actionable SKUs`.
- **Version-specific requirements:** Code node typeVersion 2.
- **Edge cases / failures:** Non-JSON or lightly malformed JSON will stop execution.
- **Sub-workflow reference:** None.

#### 8) Filter Actionable SKUs
- **Type and role:** `filter`; removes `sufficient` items.
- **Configuration choices:** keeps items where `action != "sufficient"`.
- **Input / output connections:** Input from `Parse Forecast Response`; output to `Build Forecast Email`.
- **Version-specific requirements:** Filter node typeVersion 2.3.
- **Edge cases / failures:** Exact string matching means capitalization differences from model output can bypass filter logic.
- **Sub-workflow reference:** None.

#### 9) Build Forecast Email
- **Type and role:** `code`; formats a text report from actionable forecasts.
- **Configuration choices:** Creates sections for:
  - `reorder now`
  - `stock up`
  - `reduce`
  Includes month, generation date, units to order, notes, and confidence.
- **Key expressions or variables used:** `forecasts.filter(f => f.json.action === 'reorder now')`
- **Input / output connections:** Input from `Filter Actionable SKUs`; output to `Send Forecast Report`.
- **Version-specific requirements:** Code node typeVersion 2.
- **Edge cases / failures:** If no actionable items remain, the node may still produce a valid report with zero count.
- **Sub-workflow reference:** None.

#### 10) Send Forecast Report
- **Type and role:** `gmail`; sends the demand forecast report.
- **Configuration choices:**
  - To: placeholder `your_email_id`
  - Subject includes total actionable SKU count and month
  - Body uses `email_body`
- **Input / output connections:** Input from `Build Forecast Email`; terminal.
- **Version-specific requirements:** Gmail typeVersion 2.1.
- **Edge cases / failures:** Placeholder recipient must be replaced before use.
- **Sub-workflow reference:** None.

---

## 2.6 Sticky Notes and Embedded Documentation

### Overview
The workflow includes visual documentation via sticky notes. These notes describe the purpose and schedule of each branch and also include setup prerequisites.

### Nodes Involved
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4
- Sticky Note5

### Node Details

#### Sticky Note
- **Type and role:** `stickyNote`; general workflow overview and setup checklist.
- **Configuration choices:** Large note placed outside flow.
- **Content summary:** Explains overall automation, daily/weekly behavior, and setup steps:
  1. Copy Google Sheet template and paste Sheet ID in all Google Sheets nodes
  2. Add Gmail OAuth2 credentials
  3. Add OpenAI API key
  4. Replace `your_email_id` with actual email
- **Connections:** None.
- **Edge cases / failures:** None.
- **Sub-workflow reference:** None.

#### Sticky Note1
- Daily 8AM simulation block description.

#### Sticky Note2
- Daily 8:05AM stock health block description.

#### Sticky Note3
- Daily 8:10AM PO automation block description.

#### Sticky Note4
- Sunday 9AM GPT recommendation block description.

#### Sticky Note5
- Sunday 9:30AM forecasting block description.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| 🕐 Daily 8AM Trigger | n8n-nodes-base.scheduleTrigger | Starts daily product pull and sales simulation |  | Fetch Products (DummyJSON) | Runs daily at 8AM. Pulls 20 products from DummyJSON, simulates daily sales, and writes everything into Inventory Master and Sales Log. This feeds all other sub-workflows. |
| Fetch Products (DummyJSON) | n8n-nodes-base.httpRequest | Fetches sample product catalog | 🕐 Daily 8AM Trigger | Simulate Daily Sales | Runs daily at 8AM. Pulls 20 products from DummyJSON, simulates daily sales, and writes everything into Inventory Master and Sales Log. This feeds all other sub-workflows. |
| Simulate Daily Sales | n8n-nodes-base.code | Generates simulated sales and inventory items | Fetch Products (DummyJSON) | Log to Sales Log | Runs daily at 8AM. Pulls 20 products from DummyJSON, simulates daily sales, and writes everything into Inventory Master and Sales Log. This feeds all other sub-workflows. |
| Log to Sales Log | n8n-nodes-base.googleSheets | Appends sales transactions to sheet | Simulate Daily Sales | Write to Inventory Master | Runs daily at 8AM. Pulls 20 products from DummyJSON, simulates daily sales, and writes everything into Inventory Master and Sales Log. This feeds all other sub-workflows. |
| Write to Inventory Master | n8n-nodes-base.googleSheets | Appends inventory rows to master sheet | Log to Sales Log |  | Runs daily at 8AM. Pulls 20 products from DummyJSON, simulates daily sales, and writes everything into Inventory Master and Sales Log. This feeds all other sub-workflows. |
| Daily 8:05AM Trigger | n8n-nodes-base.scheduleTrigger | Starts daily stock analysis |  | Read Inventory Master | Runs at 8:05AM daily. Reads last 7 days of sales, calculates velocity per SKU, flags each one as 🔴 🟡 or 🟢, updates the sheet, and fires email alerts if action is needed. |
| Read Inventory Master | n8n-nodes-base.googleSheets | Reads inventory master data | Daily 8:05AM Trigger | Read Sales Log | Runs at 8:05AM daily. Reads last 7 days of sales, calculates velocity per SKU, flags each one as 🔴 🟡 or 🟢, updates the sheet, and fires email alerts if action is needed. |
| Read Sales Log | n8n-nodes-base.googleSheets | Reads sales log data | Read Inventory Master | Calculate Velocity & Flags | Runs at 8:05AM daily. Reads last 7 days of sales, calculates velocity per SKU, flags each one as 🔴 🟡 or 🟢, updates the sheet, and fires email alerts if action is needed. |
| Calculate Velocity & Flags | n8n-nodes-base.code | Computes 7-day velocity, days remaining, and status flags | Read Sales Log | IF Stockout Risk | Runs at 8:05AM daily. Reads last 7 days of sales, calculates velocity per SKU, flags each one as 🔴 🟡 or 🟢, updates the sheet, and fires email alerts if action is needed. |
| IF Stockout Risk | n8n-nodes-base.if | Separates red stockout-risk items | Calculate Velocity & Flags | Alert — Stockout Risk; IF Dead or Overstock | Runs at 8:05AM daily. Reads last 7 days of sales, calculates velocity per SKU, flags each one as 🔴 🟡 or 🟢, updates the sheet, and fires email alerts if action is needed. |
| Alert — Stockout Risk | n8n-nodes-base.gmail | Emails stockout alert | IF Stockout Risk | Update Flags (Stockout) | Runs at 8:05AM daily. Reads last 7 days of sales, calculates velocity per SKU, flags each one as 🔴 🟡 or 🟢, updates the sheet, and fires email alerts if action is needed. |
| Update Flags (Stockout) | n8n-nodes-base.googleSheets | Updates red-flag fields in inventory sheet | Alert — Stockout Risk |  | Runs at 8:05AM daily. Reads last 7 days of sales, calculates velocity per SKU, flags each one as 🔴 🟡 or 🟢, updates the sheet, and fires email alerts if action is needed. |
| IF Dead or Overstock | n8n-nodes-base.if | Separates yellow dead-stock/overstock items | IF Stockout Risk | Alert — Dead Stock | Runs at 8:05AM daily. Reads last 7 days of sales, calculates velocity per SKU, flags each one as 🔴 🟡 or 🟢, updates the sheet, and fires email alerts if action is needed. |
| Alert — Dead Stock | n8n-nodes-base.gmail | Emails dead-stock alert | IF Dead or Overstock | Update Flags (Dead Stock) | Runs at 8:05AM daily. Reads last 7 days of sales, calculates velocity per SKU, flags each one as 🔴 🟡 or 🟢, updates the sheet, and fires email alerts if action is needed. |
| Update Flags (Dead Stock) | n8n-nodes-base.googleSheets | Updates yellow-flag fields in inventory sheet | Alert — Dead Stock |  | Runs at 8:05AM daily. Reads last 7 days of sales, calculates velocity per SKU, flags each one as 🔴 🟡 or 🟢, updates the sheet, and fires email alerts if action is needed. |
| Daily 8:10AM Trigger | n8n-nodes-base.scheduleTrigger | Starts PO automation |  | Read Inventory Master1 | Runs at 8:10AM daily. Picks up 🔴 SKUs from the sheet, calculates reorder quantity using lead time and buffer stock formula, emails the supplier a PO, and logs it in PO Tracker automatically |
| Read Inventory Master1 | n8n-nodes-base.googleSheets | Reads inventory for PO run | Daily 8:10AM Trigger | Filter Stockout SKUs | Runs at 8:10AM daily. Picks up 🔴 SKUs from the sheet, calculates reorder quantity using lead time and buffer stock formula, emails the supplier a PO, and logs it in PO Tracker automatically |
| Filter Stockout SKUs | n8n-nodes-base.if | Keeps only red-flagged SKUs | Read Inventory Master1 | Calculate Reorder Quantity | Runs at 8:10AM daily. Picks up 🔴 SKUs from the sheet, calculates reorder quantity using lead time and buffer stock formula, emails the supplier a PO, and logs it in PO Tracker automatically |
| Calculate Reorder Quantity | n8n-nodes-base.code | Computes reorder quantity, PO number, and skip status | Filter Stockout SKUs | Skip Zero-Velocity SKUs | Runs at 8:10AM daily. Picks up 🔴 SKUs from the sheet, calculates reorder quantity using lead time and buffer stock formula, emails the supplier a PO, and logs it in PO Tracker automatically |
| Skip Zero-Velocity SKUs | n8n-nodes-base.filter | Filters out skipped zero-velocity items | Calculate Reorder Quantity | Send PO to Supplier | Runs at 8:10AM daily. Picks up 🔴 SKUs from the sheet, calculates reorder quantity using lead time and buffer stock formula, emails the supplier a PO, and logs it in PO Tracker automatically |
| Send PO to Supplier | n8n-nodes-base.gmail | Sends PO email | Skip Zero-Velocity SKUs | Log PO to Tracker | Runs at 8:10AM daily. Picks up 🔴 SKUs from the sheet, calculates reorder quantity using lead time and buffer stock formula, emails the supplier a PO, and logs it in PO Tracker automatically |
| Log PO to Tracker | n8n-nodes-base.googleSheets | Appends PO details to tracker sheet | Send PO to Supplier | Notify — PO Raised | Runs at 8:10AM daily. Picks up 🔴 SKUs from the sheet, calculates reorder quantity using lead time and buffer stock formula, emails the supplier a PO, and logs it in PO Tracker automatically |
| Notify — PO Raised | n8n-nodes-base.gmail | Sends PO confirmation email | Log PO to Tracker |  | Runs at 8:10AM daily. Picks up 🔴 SKUs from the sheet, calculates reorder quantity using lead time and buffer stock formula, emails the supplier a PO, and logs it in PO Tracker automatically |
| Weekly Sunday 9AM Trigger | n8n-nodes-base.scheduleTrigger | Starts weekly dead-stock strategy review |  | Read Inventory Master2 | Runs every Sunday at 9AM. Sends all 🟡 SKUs to GPT-4o and gets back actionable recommendations — bundle, discount, or kill. Delivered as a clean digest email grouped by urgency. |
| Read Inventory Master2 | n8n-nodes-base.googleSheets | Reads inventory for weekly yellow-flag review | Weekly Sunday 9AM Trigger | Filter Dead & Overstock SKUs | Runs every Sunday at 9AM. Sends all 🟡 SKUs to GPT-4o and gets back actionable recommendations — bundle, discount, or kill. Delivered as a clean digest email grouped by urgency. |
| Filter Dead & Overstock SKUs | n8n-nodes-base.if | Selects yellow-flagged SKUs | Read Inventory Master2 | Build GPT Prompt | Runs every Sunday at 9AM. Sends all 🟡 SKUs to GPT-4o and gets back actionable recommendations — bundle, discount, or kill. Delivered as a clean digest email grouped by urgency. |
| Build GPT Prompt | n8n-nodes-base.code | Builds AI prompt for dead-stock actions | Filter Dead & Overstock SKUs | Inventory Strategist | Runs every Sunday at 9AM. Sends all 🟡 SKUs to GPT-4o and gets back actionable recommendations — bundle, discount, or kill. Delivered as a clean digest email grouped by urgency. |
| Inventory Strategist | @n8n/n8n-nodes-langchain.openAi | Calls GPT-4o for inventory strategy recommendations | Build GPT Prompt | Parse AI Recommendations | Runs every Sunday at 9AM. Sends all 🟡 SKUs to GPT-4o and gets back actionable recommendations — bundle, discount, or kill. Delivered as a clean digest email grouped by urgency. |
| Parse AI Recommendations | n8n-nodes-base.code | Parses AI JSON recommendations | Inventory Strategist | Build Weekly Digest Email | Runs every Sunday at 9AM. Sends all 🟡 SKUs to GPT-4o and gets back actionable recommendations — bundle, discount, or kill. Delivered as a clean digest email grouped by urgency. |
| Build Weekly Digest Email | n8n-nodes-base.code | Formats weekly dead-stock digest | Parse AI Recommendations | Send Weekly Digest | Runs every Sunday at 9AM. Sends all 🟡 SKUs to GPT-4o and gets back actionable recommendations — bundle, discount, or kill. Delivered as a clean digest email grouped by urgency. |
| Send Weekly Digest | n8n-nodes-base.gmail | Emails the weekly AI digest | Build Weekly Digest Email |  | Runs every Sunday at 9AM. Sends all 🟡 SKUs to GPT-4o and gets back actionable recommendations — bundle, discount, or kill. Delivered as a clean digest email grouped by urgency. |
| Weekly Sunday 9:30AM Trigger | n8n-nodes-base.scheduleTrigger | Starts weekly demand forecasting |  | Read sales log | Runs every Sunday at 9:30AM. Aggregates last 90 days of sales per SKU, sends it to GPT-4o with seasonal context, and emails a 30-day forecast showing exactly what to reorder before you run out. |
| Read sales log | n8n-nodes-base.googleSheets | Reads historical sales for forecasting | Weekly Sunday 9:30AM Trigger | Read Inventory Master3 | Runs every Sunday at 9:30AM. Aggregates last 90 days of sales per SKU, sends it to GPT-4o with seasonal context, and emails a 30-day forecast showing exactly what to reorder before you run out. |
| Read Inventory Master3 | n8n-nodes-base.googleSheets | Reads inventory for forecasting merge | Read sales log | Aggregate 90 Day Sales | Runs every Sunday at 9:30AM. Aggregates last 90 days of sales per SKU, sends it to GPT-4o with seasonal context, and emails a 30-day forecast showing exactly what to reorder before you run out. |
| Aggregate 90 Day Sales | n8n-nodes-base.code | Aggregates 90-day sales by SKU | Read Inventory Master3 | Build Forecast Prompt | Runs every Sunday at 9:30AM. Aggregates last 90 days of sales per SKU, sends it to GPT-4o with seasonal context, and emails a 30-day forecast showing exactly what to reorder before you run out. |
| Build Forecast Prompt | n8n-nodes-base.code | Builds AI forecasting prompt | Aggregate 90 Day Sales | Demand Forecaster | Runs every Sunday at 9:30AM. Aggregates last 90 days of sales per SKU, sends it to GPT-4o with seasonal context, and emails a 30-day forecast showing exactly what to reorder before you run out. |
| Demand Forecaster | @n8n/n8n-nodes-langchain.openAi | Calls GPT-4o for demand forecasting | Build Forecast Prompt | Parse Forecast Response | Runs every Sunday at 9:30AM. Aggregates last 90 days of sales per SKU, sends it to GPT-4o with seasonal context, and emails a 30-day forecast showing exactly what to reorder before you run out. |
| Parse Forecast Response | n8n-nodes-base.code | Parses AI forecast JSON | Demand Forecaster | Filter Actionable SKUs | Runs every Sunday at 9:30AM. Aggregates last 90 days of sales per SKU, sends it to GPT-4o with seasonal context, and emails a 30-day forecast showing exactly what to reorder before you run out. |
| Filter Actionable SKUs | n8n-nodes-base.filter | Removes sufficient forecasts | Parse Forecast Response | Build Forecast Email | Runs every Sunday at 9:30AM. Aggregates last 90 days of sales per SKU, sends it to GPT-4o with seasonal context, and emails a 30-day forecast showing exactly what to reorder before you run out. |
| Build Forecast Email | n8n-nodes-base.code | Formats forecast report email | Filter Actionable SKUs | Send Forecast Report | Runs every Sunday at 9:30AM. Aggregates last 90 days of sales per SKU, sends it to GPT-4o with seasonal context, and emails a 30-day forecast showing exactly what to reorder before you run out. |
| Send Forecast Report | n8n-nodes-base.gmail | Emails weekly forecast report | Build Forecast Email |  | Runs every Sunday at 9:30AM. Aggregates last 90 days of sales per SKU, sends it to GPT-4o with seasonal context, and emails a 30-day forecast showing exactly what to reorder before you run out. |
| Sticky Note | n8n-nodes-base.stickyNote | General overview and setup instructions |  |  | ## 🛒 D2C Supply Chain Brain Overview  This workflow runs your inventory on autopilot. It pulls product data daily, tracks how fast things are selling, flags what's about to run out, raises purchase orders automatically, and sends you a weekly report on dead stock — all without you touching anything.  ### HOW IT WORKS  Every morning at 8AM, it fetches product data and simulates daily sales. Then it calculates stock health per SKU and emails you if something needs attention. Stockout? PO goes to supplier automatically. Dead stock sitting around? GPT tells you what to do with it. Every Sunday you also get a 30-day demand forecast so you're always buying ahead, not behind.  ### SETUP STEPS  1. Copy the Google Sheet template and paste your Sheet ID in all Google Sheets nodes 2. Add your Gmail OAuth2 credentials 3. Add your OpenAI API key 4. Replace your_email_id with your email |
| Sticky Note1 | n8n-nodes-base.stickyNote | Visual note for 8AM block |  |  | Runs daily at 8AM. Pulls 20 products from DummyJSON, simulates daily sales, and writes everything into Inventory Master and Sales Log. This feeds all other sub-workflows. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Visual note for 8:05AM block |  |  | Runs at 8:05AM daily. Reads last 7 days of sales, calculates velocity per SKU, flags each one as 🔴 🟡 or 🟢, updates the sheet, and fires email alerts if action is needed. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Visual note for 8:10AM block |  |  | Runs at 8:10AM daily. Picks up 🔴 SKUs from the sheet, calculates reorder quantity using lead time and buffer stock formula, emails the supplier a PO, and logs it in PO Tracker automatically |
| Sticky Note4 | n8n-nodes-base.stickyNote | Visual note for Sunday 9AM block |  |  | Runs every Sunday at 9AM. Sends all 🟡 SKUs to GPT-4o and gets back actionable recommendations — bundle, discount, or kill. Delivered as a clean digest email grouped by urgency. |
| Sticky Note5 | n8n-nodes-base.stickyNote | Visual note for Sunday 9:30AM block |  |  | Runs every Sunday at 9:30AM. Aggregates last 90 days of sales per SKU, sends it to GPT-4o with seasonal context, and emails a 30-day forecast showing exactly what to reorder before you run out. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create the Google Sheet workbook**
   - Create one spreadsheet, for example `D2C Supply Chain Brain`.
   - Add at least these tabs:
     - `Inventory Master`
     - `Sales Log`
     - `PO Tracker`

2. **Prepare Inventory Master columns**
   - Add columns:
     - `sku_id`
     - `product_name`
     - `category`
     - `current_stock`
     - `reorder_point`
     - `lead_time_days`
     - `supplier_email`
     - `status`
     - `avg_daily_sales`
     - `days_remaining`
     - `flag`

3. **Prepare Sales Log columns**
   - Add columns:
     - `date`
     - `sku_id`
     - `product_name`
     - `units_sold`
     - `remaining_stock`

4. **Prepare PO Tracker columns**
   - Add columns:
     - `po_number`
     - `sku_id`
     - `product_name`
     - `quantity`
     - `supplier_email`
     - `status`
     - `date_raised`
   - Important: the original workflow schema shows `sku_id ` with a trailing space in one node. Prefer correcting this during rebuild so the sheet column is simply `sku_id`, then also correct the node mapping.

5. **Create credentials**
   - Add **Google Sheets OAuth2** credentials with access to the spreadsheet.
   - Add **Gmail OAuth2** credentials for sending emails.
   - Add **OpenAI API** credentials compatible with the LangChain OpenAI node.

6. **Create trigger: `🕐 Daily 8AM Trigger`**
   - Add a **Schedule Trigger** node.
   - Configure it to run daily at `08:00`.

7. **Create `Fetch Products (DummyJSON)`**
   - Add an **HTTP Request** node.
   - Method: `GET`
   - URL: `https://dummyjson.com/products?limit=20`
   - Connect `🕐 Daily 8AM Trigger -> Fetch Products (DummyJSON)`.

8. **Create `Simulate Daily Sales`**
   - Add a **Code** node.
   - Paste the logic that:
     - reads `items[0].json.products`
     - generates `units_sold`
     - computes `remaining_stock`
     - emits one item per product with:
       - `sku_id`
       - `product_name`
       - `category`
       - `current_stock`
       - `units_sold`
       - `remaining_stock`
       - `supplier_email`
       - `reorder_point`
       - `lead_time_days`
       - `status`
       - `date`
   - Connect `Fetch Products (DummyJSON) -> Simulate Daily Sales`.

9. **Create `Log to Sales Log`**
   - Add a **Google Sheets** node.
   - Operation: **Append**
   - Select the spreadsheet and `Sales Log` tab.
   - Map:
     - `date = $json.date`
     - `sku_id = $json.sku_id`
     - `product_name = $json.product_name`
     - `units_sold = $json.units_sold`
     - `remaining_stock = $json.remaining_stock`
   - Connect `Simulate Daily Sales -> Log to Sales Log`.

10. **Create `Write to Inventory Master`**
    - Add a **Google Sheets** node.
    - Operation: **Append**
    - Select spreadsheet and `Inventory Master`.
    - Map:
      - `sku_id`
      - `product_name`
      - `category`
      - `current_stock`
      - `reorder_point`
      - `lead_time_days`
      - `supplier_email`
      - `status = active`
    - The original uses expressions referencing `Simulate Daily Sales` directly.
    - Connect `Log to Sales Log -> Write to Inventory Master`.

11. **Create trigger: `Daily 8:05AM Trigger`**
    - Add **Schedule Trigger**.
    - Configure daily at `08:05`.

12. **Create `Read Inventory Master`**
    - Add **Google Sheets** node.
    - Operation: read rows from `Inventory Master`.
    - Enable returning all rows.
    - Connect `Daily 8:05AM Trigger -> Read Inventory Master`.

13. **Create `Read Sales Log`**
    - Add **Google Sheets** node.
    - Operation: read rows from `Sales Log`.
    - Connect `Read Inventory Master -> Read Sales Log`.

14. **Create `Calculate Velocity & Flags`**
    - Add **Code** node.
    - Implement:
      - load inventory rows from `Read Inventory Master`
      - load sales rows from `Read Sales Log`
      - consider only last 7 days
      - compute `avg_daily_sales`
      - compute `days_remaining`
      - flag as:
        - red if `< 14` days
        - yellow dead stock if sold `0`
        - yellow overstock if `> 60` days
        - green otherwise
    - Output:
      - `sku_id`
      - `product_name`
      - `category`
      - `current_stock`
      - `avg_daily_sales`
      - `days_remaining`
      - `flag`
      - `supplier_email`
      - `date`
    - Connect `Read Sales Log -> Calculate Velocity & Flags`.

15. **Create `IF Stockout Risk`**
    - Add **IF** node.
    - Condition: string contains.
    - Left value: `$json.flag`
    - Right value: `🔴`
    - Connect `Calculate Velocity & Flags -> IF Stockout Risk`.

16. **Create `Alert — Stockout Risk`**
    - Add **Gmail** node.
    - Operation: send email.
    - To: `$json.supplier_email`
    - Subject: `🔴 Stockout Alert — {{$json.product_name}}`
    - Body should include product, category, current stock, days remaining, avg daily sales, and reorder instruction.
    - Connect true branch of `IF Stockout Risk -> Alert — Stockout Risk`.

17. **Create `Update Flags (Stockout)`**
    - Add **Google Sheets** node.
    - Operation: **Update**
    - Sheet: `Inventory Master`
    - Match on `sku_id`
    - Update:
      - `flag`
      - `sku_id`
      - `product_name`
      - `days_remaining`
      - `avg_daily_sales`
    - Connect `Alert — Stockout Risk -> Update Flags (Stockout)`.

18. **Create `IF Dead or Overstock`**
    - Add **IF** node.
    - Condition: `$json.flag` contains `🟡`
    - Connect false branch of `IF Stockout Risk -> IF Dead or Overstock`.

19. **Create `Alert — Dead Stock`**
    - Add **Gmail** node.
    - To: `$json.supplier_email`
    - Subject: `🟡 Dead Stock Warning — {{$json.product_name}}`
    - Body: include product, category, current stock, days remaining, zero sales message, and suggested action.
    - Connect true branch of `IF Dead or Overstock -> Alert — Dead Stock`.

20. **Create `Update Flags (Dead Stock)`**
    - Add **Google Sheets** node.
    - Operation: **Update**
    - Match on `sku_id`
    - Update:
      - `flag`
      - `sku_id`
      - `days_remaining`
      - `avg_daily_sales`
    - Connect `Alert — Dead Stock -> Update Flags (Dead Stock)`.

21. **Create trigger: `Daily 8:10AM Trigger`**
    - Add **Schedule Trigger** for daily `08:10`.

22. **Create `Read Inventory Master1`**
    - Add **Google Sheets** read node for `Inventory Master`.
    - Connect from `Daily 8:10AM Trigger`.

23. **Create `Filter Stockout SKUs`**
    - Add **IF** node.
    - Keep items where `$json.flag` contains `🔴`.
    - Connect `Read Inventory Master1 -> Filter Stockout SKUs`.

24. **Create `Calculate Reorder Quantity`**
    - Add **Code** node.
    - Implement:
      - parse `avg_daily_sales`, `lead_time_days`, `current_stock`
      - if `avg_daily_sales === 0`, output `skipped = true`
      - `buffer_stock = ceil(avg_daily_sales * 7)`
      - `reorder_qty = ceil(avg_daily_sales * lead_time_days + buffer_stock - current_stock)`
      - minimum reorder fallback of `10`
      - generate:
        - `po_number`
        - `po_date`
        - `buffer_stock`
        - `reorder_qty`
        - `skipped`
    - Connect true branch of `Filter Stockout SKUs -> Calculate Reorder Quantity`.

25. **Create `Skip Zero-Velocity SKUs`**
    - Add **Filter** node.
    - Keep only items where `skipped = false`.
    - Connect `Calculate Reorder Quantity -> Skip Zero-Velocity SKUs`.

26. **Create `Send PO to Supplier`**
    - Add **Gmail** node.
    - To: preferably `$json.supplier_email`
    - Subject: `Purchase Order {{$json.po_number}} — {{$json.product_name}}`
    - Body: include PO number, date, product, SKU, quantity, current stock, and expected stockout timing.
    - Connect `Skip Zero-Velocity SKUs -> Send PO to Supplier`.
    - Important: the original expression is effectively hardcoded. When rebuilding, correct it so it actually sends to the supplier or to a controlled fallback like:
      - `{{$json.supplier_email || 'fallback@example.com'}}`

27. **Create `Log PO to Tracker`**
    - Add **Google Sheets** node.
    - Operation: **Append**
    - Sheet: `PO Tracker`
    - Map:
      - `po_number`
      - `sku_id`
      - `product_name`
      - `quantity`
      - `supplier_email`
      - `status = Sent`
      - `date_raised`
    - Connect `Send PO to Supplier -> Log PO to Tracker`.

28. **Create `Notify — PO Raised`**
    - Add **Gmail** node.
    - To: internal operator email or supplier email depending on desired process.
    - Subject: `✅ PO Raised — {{$json.product_name}}`
    - Body: confirm PO number, recipient, quantity, and days remaining.
    - Connect `Log PO to Tracker -> Notify — PO Raised`.

29. **Create trigger: `Weekly Sunday 9AM Trigger`**
    - Add **Schedule Trigger** set weekly at `09:00` Sunday.

30. **Create `Read Inventory Master2`**
    - Add **Google Sheets** read node for `Inventory Master`.
    - Connect from `Weekly Sunday 9AM Trigger`.

31. **Create `Filter Dead & Overstock SKUs`**
    - Add **IF** node.
    - Condition: `$json.flag` contains `🟡`.
    - Connect `Read Inventory Master2 -> Filter Dead & Overstock SKUs`.

32. **Create `Build GPT Prompt`**
    - Add **Code** node.
    - Build a prompt containing SKU list and strict JSON output instructions for recommendations:
      - `action`
      - `discount_percent`
      - `reason`
      - `urgency`
    - Connect `Filter Dead & Overstock SKUs -> Build GPT Prompt`.

33. **Create `Inventory Strategist`**
    - Add **OpenAI** LangChain node.
    - Model: `gpt-4o`
    - Response content: use `{{$json.prompt}}`
    - Connect `Build GPT Prompt -> Inventory Strategist`.

34. **Create `Parse AI Recommendations`**
    - Add **Code** node.
    - Extract the AI text response.
    - Strip code fences.
    - `JSON.parse(...)`
    - Return one item per recommendation.
    - Connect `Inventory Strategist -> Parse AI Recommendations`.

35. **Create `Build Weekly Digest Email`**
    - Add **Code** node.
    - Group recommendations by `urgency` and build a plaintext report.
    - Output:
      - `email_body`
      - `total_flagged`
    - Connect `Parse AI Recommendations -> Build Weekly Digest Email`.

36. **Create `Send Weekly Digest`**
    - Add **Gmail** node.
    - To: your real email address.
    - Subject: `📦 Weekly Dead Inventory Report — {{$json.total_flagged}} SKUs Need Attention`
    - Body: `{{$json.email_body}}`
    - Connect `Build Weekly Digest Email -> Send Weekly Digest`.

37. **Create trigger: `Weekly Sunday 9:30AM Trigger`**
    - Add **Schedule Trigger** set weekly at `09:30` Sunday.

38. **Create `Read sales log`**
    - Add **Google Sheets** read node for `Sales Log`.
    - Connect from `Weekly Sunday 9:30AM Trigger`.

39. **Create `Read Inventory Master3`**
    - Add **Google Sheets** read node for `Inventory Master`.
    - Connect `Read sales log -> Read Inventory Master3`.

40. **Create `Aggregate 90 Day Sales`**
    - Add **Code** node.
    - Implement:
      - read all sales from `Read sales log`
      - read all inventory rows from `Read Inventory Master3`
      - keep only sales from last 90 days
      - group by SKU
      - compute `total_sold_90d`
      - compute `avg_daily_sales_90d`
      - merge with inventory data
    - Connect `Read Inventory Master3 -> Aggregate 90 Day Sales`.

41. **Create `Build Forecast Prompt`**
    - Add **Code** node.
    - Build forecast prompt asking GPT-4o to return strict JSON with:
      - `forecast_month`
      - `forecasts[]`
      - `predicted_units_30d`
      - `stock_needed`
      - `action`
      - `seasonality_note`
      - `confidence`
    - Connect `Aggregate 90 Day Sales -> Build Forecast Prompt`.

42. **Create `Demand Forecaster`**
    - Add **OpenAI** LangChain node.
    - Model: `gpt-4o`
    - Provide `{{$json.prompt}}` as the content.
    - Connect `Build Forecast Prompt -> Demand Forecaster`.

43. **Create `Parse Forecast Response`**
    - Add **Code** node.
    - Extract model output text, strip code fences, parse JSON, emit one item per forecast.
    - Include `forecast_month` in each output item.
    - Connect `Demand Forecaster -> Parse Forecast Response`.

44. **Create `Filter Actionable SKUs`**
    - Add **Filter** node.
    - Keep only items where `action != sufficient`.
    - Connect `Parse Forecast Response -> Filter Actionable SKUs`.

45. **Create `Build Forecast Email`**
    - Add **Code** node.
    - Build plaintext report sections for:
      - reorder now
      - stock up
      - reduce
    - Output:
      - `email_body`
      - `total_action`
      - `month`
    - Connect `Filter Actionable SKUs -> Build Forecast Email`.

46. **Create `Send Forecast Report`**
    - Add **Gmail** node.
    - To: your real email address.
    - Subject: `📊 30-Day Demand Forecast — {{$json.total_action}} SKUs Need Action — {{$json.month}}`
    - Body: `{{$json.email_body}}`
    - Connect `Build Forecast Email -> Send Forecast Report`.

47. **Add sticky notes if desired**
    - Create visual notes describing each branch and setup steps.

48. **Replace all placeholders**
    - Replace `your_email_id`.
    - Replace dummy supplier addresses.
    - Confirm spreadsheet IDs and sheet names in all Google Sheets nodes.

49. **Test each branch independently**
    - Run the 8:00 AM branch first.
    - Verify rows appear in `Sales Log` and `Inventory Master`.
    - Then run the 8:05 AM branch and verify `avg_daily_sales`, `days_remaining`, and `flag`.
    - Then test the 8:10 AM branch.
    - Finally test both weekly GPT branches.

50. **Recommended hardening before production**
    - Change `Write to Inventory Master` from append to update/upsert by `sku_id`.
    - Fix `Send PO to Supplier` recipient expression.
    - Add error handling around `JSON.parse` in both AI parsing nodes.
    - Normalize model outputs to lowercase before filter comparisons.
    - Distinguish dead stock vs overstock email templates.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| ## 🛒 D2C Supply Chain Brain Overview | General workflow note |
| This workflow runs your inventory on autopilot. It pulls product data daily, tracks how fast things are selling, flags what's about to run out, raises purchase orders automatically, and sends you a weekly report on dead stock — all without you touching anything. | General workflow description |
| Every morning at 8AM, it fetches product data and simulates daily sales. Then it calculates stock health per SKU and emails you if something needs attention. Stockout? PO goes to supplier automatically. Dead stock sitting around? GPT tells you what to do with it. Every Sunday you also get a 30-day demand forecast so you're always buying ahead, not behind. | Operational summary |
| Copy the Google Sheet template and paste your Sheet ID in all Google Sheets nodes | Setup instruction |
| Add your Gmail OAuth2 credentials | Setup instruction |
| Add your OpenAI API key | Setup instruction |
| Replace `your_email_id` with your email | Setup instruction |

## Additional implementation notes
- The workflow has **five separate entry points**, all schedule-based.
- There are **no sub-workflows** or Execute Workflow nodes in this JSON.
- The workflow is currently **inactive** (`active: false`).
- It uses sample product input from **DummyJSON**, so it is suitable as a demo or prototype but not as-is for real ERP inventory synchronization.
- The workflow depends heavily on **Google Sheets as the system of record**, which makes data hygiene critical:
  - avoid duplicate SKU rows
  - keep sheet column names stable
  - validate date formats
- Both AI branches assume GPT-4o returns **strict JSON**. In practice, adding structured output enforcement or parser fallback logic is strongly recommended.