Forecast and report multi-channel tax liabilities with OpenAI, Gmail, Sheets and Airtable

https://n8nworkflows.xyz/workflows/forecast-and-report-multi-channel-tax-liabilities-with-openai--gmail--sheets-and-airtable-12735


# Forecast and report multi-channel tax liabilities with OpenAI, Gmail, Sheets and Airtable

## 1. Workflow Overview

**Title (provided):** Forecast and report multi-channel tax liabilities with OpenAI, Gmail, Sheets and Airtable  
**Workflow name (JSON):** Multi-Channel Revenue Tax Liability Forecasting and Reporting System

**Purpose:**  
This workflow runs on a monthly schedule to consolidate multi-channel revenue, enrich it with historical data, detect anomalous revenue patterns, calculate tax liabilities (including tier-based treatments), generate a 6‑month rolling forecast plus scenario analysis, then distribute and store a formatted report via **Gmail**, **Google Sheets**, and **Airtable**.

**Target use cases:**
- Monthly tax liability estimation for e-commerce / multi-channel businesses
- Early warning for unusual revenue patterns (audit risk / reconciliation issues)
- Producing consistent finance/tax reporting artifacts and audit trails

### 1.1 Trigger & Configuration
- Scheduled monthly run + central configuration of tax rates, thresholds, channels, and endpoints.

### 1.2 Revenue Aggregation & Historical Enrichment
- Aggregates “current” channel revenue and merges with historical revenue from an API.
- Computes summary statistics needed for anomaly detection.

### 1.3 Anomaly Detection & Alerting
- Statistical outlier detection (z-score) followed by conditional routing:
  - If anomaly: format alert and email tax agent.
  - Else: proceed to tier routing and tax logic.

### 1.4 Tax Engine (Tier Routing + Deductions/Brackets + Liabilities)
- Routes to small vs medium/large business logic.
- Combines results and computes tax liabilities.

### 1.5 Forecasting, Scenario Analysis & Reporting
- Generates rolling 6-month forecast and best/likely/worst scenario analysis.
- Formats report data and distributes/stores it.

---

## 2. Block-by-Block Analysis

### Block A — Trigger & Workflow Configuration

**Overview:**  
Starts monthly and defines all operational constants (tax rates, thresholds, channels, forecast horizon, and contact endpoints) used across the workflow.

**Nodes involved:**
- Monthly Schedule
- Workflow Configuration

#### Node: Monthly Schedule
- **Type / role:** Schedule Trigger — initiates the workflow periodically.
- **Configuration (interpreted):** Runs every month at **09:00** (server timezone).
- **Connections:**  
  - **Out:** Workflow Configuration
- **Failure modes / edge cases:**
  - Timezone mismatch vs business expectations (n8n instance timezone vs local finance timezone).
  - If monthly trigger day is implicit (n8n schedule “months” can behave as “every month” at a time; confirm intended day-of-month behavior in UI).

#### Node: Workflow Configuration
- **Type / role:** Set — central configuration object for later nodes.
- **Key fields set (selected):**
  - `incomeTaxRate` (0.25), `vatRate` (0.2), `gstRate` (0.15), `withholdingTaxRate` (0.1)
  - `forecastMonths` (6)
  - `taxAgentEmail` (**placeholder**)
  - `revenueChannels` array: `["Shopify","Amazon","Website","Wholesale"]`
  - `anomalyThreshold` (2)
  - business thresholds: `smallBusinessThreshold` (100000), `mediumBusinessThreshold` (500000)
  - `standardDeduction` (12950)
  - `historicalDataApiUrl` (**placeholder**)
- **Configuration choices:**
  - “Include other fields” enabled, so incoming fields pass through (useful if later you extend trigger payload).
- **Connections:**  
  - **Out (parallel):** Aggregate Multi-Channel Revenue; Fetch Historical Revenue Data
- **Failure modes / edge cases:**
  - Placeholders not replaced will break downstream operations (email, HTTP endpoint, storage IDs).
  - Some downstream nodes hardcode thresholds/tax values instead of referencing these config fields (see Block D/E).

**Sticky note context (applies to this block and overall setup):**
- **Prerequisites / Use cases / Benefits** (Sticky Note): multi-platform API access; automated monthly sales tax calculations; customizable rules; “reduces preparation time by 80%”.
- **Setup Steps** (Sticky Note1): configure e-commerce/payment/accounting/OpenAI/Gmail/Sheets/Airtable credentials (note: OpenAI is mentioned but not actually used by any node in this JSON).

---

### Block B — Revenue Aggregation & Historical Enrichment

**Overview:**  
Builds a current-period revenue summary by channel, fetches historical revenue data from an API, merges both datasets, and produces aggregate statistics.

**Nodes involved:**
- Aggregate Multi-Channel Revenue
- Fetch Historical Revenue Data
- Merge Historical and Current Data
- Calculate Revenue Statistics

#### Node: Aggregate Multi-Channel Revenue
- **Type / role:** Code — aggregates incoming items by `channel` and sums `revenue`.
- **Key logic:**
  - Reads all input items (`$input.all()`), expects each item to have:
    - `item.json.channel`
    - `item.json.revenue` (parseable float)
  - Outputs one item with:
    - `overallTotal`
    - `channelBreakdown` (channel, revenue, percentage)
    - `totalChannels`, `reportDate`, `period`
- **Connections:**  
  - **In:** Workflow Configuration (but note: configuration node does not generate channel revenue items; see “integration gap” below)  
  - **Out:** Calculate Tax Liabilities; Merge Historical and Current Data (as input 1)
- **Failure modes / edge cases:**
  - If it receives only the config item (as currently wired), `channel`/`revenue` will be missing → it will output `overallTotal = 0` and a breakdown containing config’s fields if any accidentally match; effectively “empty revenue”.
  - Division by zero risk: `percentage` uses `(total/overallTotal)`; if `overallTotal` is 0, this becomes `Infinity`/`NaN`. (JavaScript will produce `Infinity`, then `.toFixed(2)` throws or yields unexpected results depending on value; this can break.)
- **Integration gap to address:**  
  This node assumes upstream nodes produce **one item per transaction (or per channel) with `channel` and `revenue`**. In the provided workflow, no Shopify/Amazon/etc fetch nodes exist. You must add them (or an upstream data source) to supply channel revenue items before aggregation.

#### Node: Fetch Historical Revenue Data
- **Type / role:** HTTP Request — fetches historical revenue dataset from an external API.
- **Configuration (interpreted):**
  - URL is a placeholder; sends `Content-Type: application/json`
  - Authentication: “predefinedCredentialType” (credential not included in JSON export)
- **Connections:**  
  - **Out:** Merge Historical and Current Data (as input 0)
- **Failure modes / edge cases:**
  - Missing/incorrect credentials.
  - API returning non-JSON or unexpected schema.
  - Pagination not handled (if API returns paged results, only first page used).
  - Response shape mismatch with downstream summarization fields (expects `revenue` fields later).

#### Node: Merge Historical and Current Data
- **Type / role:** Merge — combines two inputs by position.
- **Configuration (interpreted):**
  - Mode: **Combine**
  - Combine by: **Position**
- **Connections:**  
  - **In 0:** Fetch Historical Revenue Data  
  - **In 1:** Aggregate Multi-Channel Revenue  
  - **Out:** Calculate Revenue Statistics
- **Failure modes / edge cases:**
  - Combine-by-position is fragile if input streams have different item counts; resulting merged items may not align meaningfully.
  - If historical data is a single array/object rather than multiple items, merge results may be odd (e.g., only first record merged).

#### Node: Calculate Revenue Statistics
- **Type / role:** Summarize — calculates statistical aggregates over `revenue`.
- **Configuration (interpreted):**
  - Summarizes field `revenue` with: sum, average, min, max.
- **Connections:**  
  - **Out:** Detect Anomalies
- **Failure modes / edge cases:**
  - If the merged dataset lacks `revenue`, all summaries become 0/empty.
  - If revenue is string with currency symbols, summarize may treat as non-numeric.

**Sticky note context (covers this block conceptually):**
- **Revenue Aggregation** (Sticky Note5): indicates this area should fetch transaction data from multiple platforms via scheduled triggers (not currently implemented in nodes).

---

### Block C — Anomaly Detection & Alert Routing

**Overview:**  
Computes z-score anomalies across revenue values. If anomaly is detected, formats an alert and emails the tax agent; otherwise routes the data into tier-based tax logic.

**Nodes involved:**
- Detect Anomalies
- Check Tax Threshold
- Format Alert Data
- Send Anomaly Alert
- Route by Revenue Tier (entry from “no anomaly” path)

#### Node: Detect Anomalies
- **Type / role:** Code — statistical anomaly detection (z-score).
- **Key logic:**
  - Builds `revenues` from each item using first available:
    - `revenue` OR `totalRevenue` OR `projectedRevenue`
  - Requires at least 3 data points; otherwise marks no anomaly with reason.
  - Computes mean/stddev; flags anomalies if `zScore > 2` (hardcoded).
  - Adds `anomalyDetection` object per item; adds `summary` onto the first item.
- **Connections:**  
  - **Out:** Check Tax Threshold
- **Failure modes / edge cases:**
  - Uses hardcoded `zScoreThreshold = 2` rather than `Workflow Configuration.anomalyThreshold`.
  - If mean is 0, `percentageDeviation` does `(revenue-mean)/mean` → division by zero → Infinity.
  - The output field is `anomalyDetection.isAnomaly`, but downstream IF checks `$json.hasAnomaly` (mismatch).

#### Node: Check Tax Threshold
- **Type / role:** IF — intended to branch on anomaly presence.
- **Configuration (interpreted):**
  - Condition: `{{ $json.hasAnomaly }} == true`
- **Connections:**
  - **True:** Format Alert Data
  - **False:** Route by Revenue Tier
- **Failure modes / edge cases (important):**
  - **Field mismatch:** Detect Anomalies sets `anomalyDetection.isAnomaly`, not `hasAnomaly`. As written, `$json.hasAnomaly` is likely undefined → condition is false → **alerts never trigger**.
  - Fix: change to `{{ $json.anomalyDetection.isAnomaly }}` (or add a mapping Set node).

#### Node: Format Alert Data
- **Type / role:** Set — prepares alert fields for the email.
- **Configuration (interpreted):**
  - `alertType = "Revenue Anomaly Detected"`
  - `alertMessage = "Unusual revenue pattern detected: " + $json.anomalyDetails`
  - `severity = "High"`
  - `timestamp = $now.toISO()`
- **Connections:**  
  - **Out:** Send Anomaly Alert
- **Failure modes / edge cases:**
  - `anomalyDetails` is not created by Detect Anomalies (it creates `anomalyDetection`), so the message will be incomplete or “undefined”.
  - Sends `severity`, but email template expects `severityLevel` (mismatch).

#### Node: Send Anomaly Alert
- **Type / role:** Gmail — emails anomaly alert to tax agent.
- **Configuration (interpreted):**
  - To: `{{ $('Workflow Configuration').first().json.taxAgentEmail }}`
  - Subject: `ALERT: Revenue Anomaly Detected - {{ $json.timestamp }}`
  - HTML body references many fields:
    - `severityLevel`, `anomalyType`, `expectedValue`, `actualValue`, `deviation`, `recommendedAction1/2/3`, `impactDescription`
- **Connections:** none (terminal).
- **Failure modes / edge cases:**
  - **Template/data mismatch:** preceding nodes do not populate most referenced fields → email may contain blanks.
  - Gmail OAuth credential required; quota/permission issues possible.
  - HTML includes a warning symbol; some clients render differently.
- **Credential:** Gmail OAuth2 (configured in workflow).

#### Node: Route by Revenue Tier
- Documented in Block D (it is the “no anomaly” branch destination).

**Sticky note context (covers this block conceptually):**
- **AI Anomaly Detection & Alert** (Sticky Note3): describes AI-based detection, but the implemented detection is statistical (z-score), not OpenAI.

---

### Block D — Tier Routing & Tax Calculation Preparation

**Overview:**  
Classifies the business into revenue tiers (small/medium/large) and applies different calculation strategies (deductions vs progressive brackets), then merges them for downstream tax liability computations.

**Nodes involved:**
- Route by Revenue Tier
- Calculate Tax Deductions
- Apply Progressive Tax Brackets
- Combine Tax Calculations

#### Node: Route by Revenue Tier
- **Type / role:** Switch — routes by `totalRevenue`.
- **Configuration (interpreted):**
  - **Small Business:** `totalRevenue < 100000`
  - **Medium Business:** `100000 <= totalRevenue < 500000`
  - **Fallback:** Large Business
- **Connections:**
  - Output “Small Business” → Calculate Tax Deductions
  - Output “Medium Business” → Apply Progressive Tax Brackets
  - Output “Large Business” → (also goes to Apply Progressive Tax Brackets? In this JSON it only shows two explicit outputs; fallback is “Large Business” but not connected—meaning it may be unhandled depending on n8n wiring.)
- **Failure modes / edge cases:**
  - Uses hardcoded thresholds rather than `Workflow Configuration.smallBusinessThreshold/mediumBusinessThreshold`.
  - If `totalRevenue` is missing, comparisons may fail or route to fallback unpredictably.
  - Ensure fallback output is connected, otherwise “Large Business” path may drop.

#### Node: Calculate Tax Deductions
- **Type / role:** Code — computes deduction components intended for small businesses.
- **Key logic:**
  - Revenue source: `item.json.revenue || item.json.totalRevenue || 0`
  - Deductions:
    - Standard deduction: `min(12950, revenue*0.05)` (hardcoded 12950, should ideally use config)
    - Business expenses: `expenses*0.85`
    - Depreciation: `assets*0.20`
    - Section 179: min(1,080,000, qualifyingEquipment)
    - Home office: min(squareFeet*5, 1500)
  - Outputs:
    - `deductions` object
    - `taxableIncome`, `taxSavings`, `effectiveTaxRate` (note: effectiveTaxRate here is taxableIncome/revenue, not tax rate)
- **Connections:**  
  - **Out:** Combine Tax Calculations (input 0)
- **Failure modes / edge cases:**
  - Several constants are hardcoded; may not match jurisdiction/year.
  - If inputs like `expenses/assets/...` absent, treated as 0 (safe but may understate deductions).
  - `effectiveTaxRate` naming is misleading (it’s taxable-income ratio).

#### Node: Apply Progressive Tax Brackets
- **Type / role:** Code — computes progressive tax totals for medium/large tier.
- **Key logic:**
  - Income = `totalRevenue || income || 0`
  - Brackets:
    - 0–50k @10%
    - 50k–100k @20%
    - 100k–250k @30%
    - 250k+ @37%
  - Outputs `progressiveTaxCalculation` with totalTax, effectiveTaxRate, bracket breakdown.
- **Connections:**  
  - **Out:** Combine Tax Calculations (input 1)
- **Failure modes / edge cases:**
  - Brackets are illustrative and hardcoded; not tied to config nor jurisdiction.
  - If “Large Business” fallback is not connected to this node, large businesses may not be processed.

#### Node: Combine Tax Calculations
- **Type / role:** Merge — combines deduction-based and bracket-based results.
- **Configuration (interpreted):**
  - Mode: Combine by position.
- **Connections:**  
  - **Out:** Calculate Tax Liabilities
- **Failure modes / edge cases:**
  - Combine-by-position between mutually exclusive paths is usually problematic:
    - Small-business path and medium/large path won’t both produce items for the same entity at the same position.
    - May produce partial/empty merges or misaligned data.
  - If only one branch runs, the merge may output nothing (depending on n8n merge behavior and whether it waits for both inputs).
  - Consider “Merge by Key” or redesign (e.g., compute both in one code node conditionally).

**Sticky note context (covers this block conceptually):**
- **Tax Calculation Engine** (Sticky Note4): describes routing by jurisdiction and progressive brackets; actual workflow routes by revenue tier, not jurisdiction.

---

### Block E — Core Tax Liability Calculation

**Overview:**  
Computes income tax, VAT, GST, and withholding based on revenue components and configured rates, then produces totals and net revenue.

**Nodes involved:**
- Calculate Tax Liabilities

#### Node: Calculate Tax Liabilities
- **Type / role:** Code — calculates multiple tax components and totals.
- **Key logic:**
  - Reads config from `$('Workflow Configuration').first().json`
  - Expects input revenue structure with:
    - `totalRevenue`, `domesticRevenue`, `internationalRevenue`
  - Computes:
    - `incomeTax = totalRevenue * incomeTaxRate`
    - `vat = domesticRevenue * vatRate`
    - `gst = internationalRevenue * gstRate`
    - `withholdingTax = internationalRevenue * withholdingTaxRate`
    - `totalTaxLiability`, `netRevenue`
  - Outputs:
    - `taxCalculations` object (incomeTax, vat, gst, withholdingTax, totalTaxLiability, netRevenue)
    - `taxRatesApplied`
- **Connections:**  
  - **Out:** Generate Rolling Forecast
- **Failure modes / edge cases (important):**
  - **Schema mismatch:** Upstream aggregation outputs `overallTotal`, not `totalRevenue`. This node will see `totalRevenue = 0` unless mapped.
  - The computed values are nested in `taxCalculations`, but later “Format Report Data” expects top-level `incomeTax`, `vat`, etc.
  - Default rates in code differ from config defaults (code uses fallback 0.21/0.20/0.10/0.15 if config missing).
  - If domestic/international split isn’t provided upstream, VAT/GST/withholding become 0.

---

### Block F — Forecasting, Scenario Analysis, Formatting, Distribution & Storage

**Overview:**  
Generates a simplistic rolling forecast, builds three scenarios (best/likely/worst), then formats a report and sends/stores it to multiple destinations.

**Nodes involved:**
- Generate Rolling Forecast
- Generate Scenario Analysis
- Format Report Data
- Send Report to Tax Agent
- Store in Google Sheets
- Store in Airtable

#### Node: Generate Rolling Forecast
- **Type / role:** Code — produces 6 monthly forecast items.
- **Key logic:**
  - Computes average monthly revenue from items where `data.revenue` exists.
  - Uses fixed `growthRate = 0.05`.
  - `taxRate = historicalData[0]?.taxRate || 0.25` (expects `taxRate` somewhere upstream).
  - For next 6 months outputs items with:
    - `projectedRevenue`, `projectedTaxLiability`, `netRevenue`, `growthRate`, `taxRate`
- **Connections:**
  - **Out:** Format Report Data
  - **Out:** Generate Scenario Analysis
- **Failure modes / edge cases:**
  - Uses `data.revenue` but upstream nodes largely use `totalRevenue`/other naming; avgMonthlyRevenue may become 0.
  - Forecast horizon is hardcoded to 6, ignoring `config.forecastMonths`.
  - Tax rate source is unclear; not produced by earlier nodes unless manually added.

#### Node: Generate Scenario Analysis
- **Type / role:** Code — generates 3 scenarios across 6 months.
- **Key logic:**
  - Uses baseRevenue from `forecastData[0]?.projectedRevenue || 100000`
  - baseTaxRate from `forecastData[0]?.taxRate` interpreted as percent-string; else 0.25
  - Scenarios:
    - Best: +15% monthly, efficiency 0.95
    - Most likely: +5%, efficiency 1.0
    - Worst: −5%, efficiency 1.05
  - Outputs one item per scenario with `monthlyProjections` and totals.
- **Connections:**  
  - **Out:** Format Report Data
- **Failure modes / edge cases:**
  - Depends on output shape of rolling forecast (`taxRate` is a string like “25%” in rolling forecast output; parsing logic expects it might be numeric percent string; mismatch risk).
  - Produces multiple items; merging into one report requires aggregation, but Format Report Data is not aggregating—so it will run once per scenario item, causing multiple emails/records.

#### Node: Format Report Data
- **Type / role:** Set — maps fields into a report schema.
- **Configuration (interpreted):**
  - `reportMonth = $now.format('MMMM yyyy')`
  - `reportDate = $now.toISO()`
  - Maps numeric fields from `$json`:
    - `totalRevenue = $json.totalRevenue`
    - `incomeTax = $json.incomeTax`, `vat = $json.vat`, `gst`, `withholdingTax`, `totalTaxLiability`
  - `forecast = $json.forecast`
  - `revenueByChannel = $json.revenueByChannel`
- **Connections:**  
  - **Out (fan-out):** Send Report to Tax Agent; Store in Google Sheets; Store in Airtable
- **Failure modes / edge cases (important):**
  - **Field mismatches:** Upstream likely provides `taxCalculations.incomeTax` etc, not top-level `incomeTax`. Also aggregation produced `overallTotal`, not `totalRevenue`.
  - `forecast` and `revenueByChannel` are not constructed in upstream nodes in a compatible way:
    - rolling forecast emits one item per month, not a `forecast` array field
    - aggregation emits `channelBreakdown`, not `revenueByChannel`
  - Because both rolling forecast and scenario analysis connect here, this node may execute multiple times, producing duplicate outputs.

#### Node: Send Report to Tax Agent
- **Type / role:** Gmail — sends monthly report email.
- **Configuration (interpreted):**
  - To: config `taxAgentEmail`
  - Subject includes `reportMonth`
  - HTML body references:
    - `totalRevenue`, `incomeTax`, `salesTax`, `totalTaxLiability`, `forecastRevenue`, `forecastTax`
- **Connections:** terminal.
- **Failure modes / edge cases:**
  - **Template mismatch:** workflow produces VAT/GST/withholding, but template uses `salesTax` and forecast summary fields not defined (`forecastRevenue`, `forecastTax`).
  - Gmail OAuth required and may hit sending limits.

#### Node: Store in Google Sheets
- **Type / role:** Google Sheets — append or update report row for audit trail.
- **Configuration (interpreted):**
  - Operation: **Append or Update**
  - Spreadsheet: placeholder Document ID
  - Sheet name: “Tax Forecasts”
  - Matching column: `reportDate`
  - Mapping mode: auto-map input data
- **Connections:** terminal.
- **Failure modes / edge cases:**
  - If sheet lacks `reportDate` column, append/update may fail.
  - Auto-mapping can create unexpected column placements if schema differs.
  - OAuth permission or spreadsheet sharing issues.

#### Node: Store in Airtable
- **Type / role:** Airtable — creates a new record for the report.
- **Configuration (interpreted):**
  - Operation: Create
  - Base ID and Table ID are placeholders.
- **Connections:** terminal.
- **Failure modes / edge cases:**
  - Field names in Airtable must match incoming JSON keys (or you must map explicitly).
  - Rate limits; invalid base/table IDs; auth failures.

**Sticky note context (covers this block conceptually):**
- **Report Generation** (Sticky Note6): emphasizes authority-specific templates and audit-ready reporting.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Monthly Schedule | Schedule Trigger | Monthly trigger | — | Workflow Configuration | ## How It Works … (automates tax compliance, anomaly detection, reporting, storage). |
| Workflow Configuration | Set | Central parameters (rates, thresholds, endpoints) | Monthly Schedule | Aggregate Multi-Channel Revenue; Fetch Historical Revenue Data | ## How It Works … (automates tax compliance, anomaly detection, reporting, storage). |
| Aggregate Multi-Channel Revenue | Code | Aggregate revenue by channel and total | Workflow Configuration | Calculate Tax Liabilities; Merge Historical and Current Data | ## Revenue Aggregation — Fetches transaction data… consolidates revenue streams… |
| Fetch Historical Revenue Data | HTTP Request | Pull historical revenue from API | Workflow Configuration | Merge Historical and Current Data | ## Revenue Aggregation — Fetches transaction data… consolidates revenue streams… |
| Merge Historical and Current Data | Merge | Combine current + historical datasets | Fetch Historical Revenue Data; Aggregate Multi-Channel Revenue | Calculate Revenue Statistics | ## Revenue Aggregation — Fetches transaction data… consolidates revenue streams… |
| Calculate Revenue Statistics | Summarize | Compute revenue sum/avg/min/max | Merge Historical and Current Data | Detect Anomalies | ## AI Anomaly Detection  & Alert — Analyzes… unusual patterns… |
| Detect Anomalies | Code | Z-score anomaly detection | Calculate Revenue Statistics | Check Tax Threshold | ## AI Anomaly Detection  & Alert — Analyzes… unusual patterns… |
| Check Tax Threshold | IF | Branch: anomaly alert vs tax routing | Detect Anomalies | Format Alert Data (true); Route by Revenue Tier (false) | ## AI Anomaly Detection  & Alert — Analyzes… unusual patterns… |
| Format Alert Data | Set | Prepare alert payload | Check Tax Threshold (true) | Send Anomaly Alert | ## AI Anomaly Detection  & Alert — Analyzes… unusual patterns… |
| Send Anomaly Alert | Gmail | Email anomaly alert | Format Alert Data | — | ## AI Anomaly Detection  & Alert — Analyzes… unusual patterns… |
| Route by Revenue Tier | Switch | Classify by revenue tier | Check Tax Threshold (false) | Calculate Tax Deductions; Apply Progressive Tax Brackets | ## Tax Calculation Engine — Routes… applies tax rates… progressive brackets… |
| Calculate Tax Deductions | Code | Small business deduction logic | Route by Revenue Tier | Combine Tax Calculations | ## Tax Calculation Engine — Routes… applies tax rates… progressive brackets… |
| Apply Progressive Tax Brackets | Code | Medium/large progressive tax calculation | Route by Revenue Tier | Combine Tax Calculations | ## Tax Calculation Engine — Routes… applies tax rates… progressive brackets… |
| Combine Tax Calculations | Merge | Merge tier-specific results | Calculate Tax Deductions; Apply Progressive Tax Brackets | Calculate Tax Liabilities | ## Tax Calculation Engine — Routes… applies tax rates… progressive brackets… |
| Calculate Tax Liabilities | Code | Compute income/VAT/GST/withholding and totals | Combine Tax Calculations; Aggregate Multi-Channel Revenue | Generate Rolling Forecast | ## Tax Calculation Engine — Routes… applies tax rates… progressive brackets… |
| Generate Rolling Forecast | Code | 6-month forecast generation | Calculate Tax Liabilities | Format Report Data; Generate Scenario Analysis | ## Report Generation — Formats tax data… audit-ready reports… |
| Generate Scenario Analysis | Code | Best/likely/worst scenario projections | Generate Rolling Forecast | Format Report Data | ## Report Generation — Formats tax data… audit-ready reports… |
| Format Report Data | Set | Normalize fields for reporting/storage | Generate Rolling Forecast; Generate Scenario Analysis | Send Report to Tax Agent; Store in Google Sheets; Store in Airtable | ## Report Generation — Formats tax data… audit-ready reports… |
| Send Report to Tax Agent | Gmail | Email monthly report | Format Report Data | — | ## Report Generation — Formats tax data… audit-ready reports… |
| Store in Google Sheets | Google Sheets | Persist report row (audit trail) | Format Report Data | — | ## Report Generation — Formats tax data… audit-ready reports… |
| Store in Airtable | Airtable | Persist report record | Format Report Data | — | ## Report Generation — Formats tax data… audit-ready reports… |
| Sticky Note | Sticky Note | Documentation (prereqs/use cases/customization/benefits) | — | — | ## Prerequisites… Use Cases… Customization… Benefits… |
| Sticky Note1 | Sticky Note | Documentation (setup steps) | — | — | ## Setup Steps … OpenAI/Gmail/Sheets/Airtable… |
| Sticky Note2 | Sticky Note | Documentation (how it works) | — | — | ## How It Works … end-to-end description… |
| Sticky Note3 | Sticky Note | Documentation (AI anomaly detection & alert) | — | — | ## AI Anomaly Detection & Alert … |
| Sticky Note4 | Sticky Note | Documentation (tax calculation engine) | — | — | ## Tax Calculation Engine … |
| Sticky Note5 | Sticky Note | Documentation (revenue aggregation) | — | — | ## Revenue Aggregation … |
| Sticky Note6 | Sticky Note | Documentation (report generation) | — | — | ## Report Generation … |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Name it: **Multi-Channel Revenue Tax Liability Forecasting and Reporting System** (or your preferred name).
- Keep workflow **inactive** until credentials and placeholders are set.

2) **Add trigger**
- Add **Schedule Trigger** node named **Monthly Schedule**.
- Configure: interval **Months**, trigger at **09:00**.

3) **Add configuration node**
- Add **Set** node named **Workflow Configuration** connected from Monthly Schedule.
- Add fields (Number/String/Array) at minimum:
  - `incomeTaxRate` (0.25), `vatRate` (0.2), `gstRate` (0.15), `withholdingTaxRate` (0.1)
  - `forecastMonths` (6)
  - `taxAgentEmail` (your email)
  - `revenueChannels` (array: Shopify, Amazon, Website, Wholesale)
  - `anomalyThreshold` (2)
  - `smallBusinessThreshold` (100000), `mediumBusinessThreshold` (500000)
  - `standardDeduction` (12950)
  - `historicalDataApiUrl` (your API endpoint URL)
- Enable “Include other fields” if you plan to pass through additional data.

4) **(Required) Add revenue source nodes (not present in JSON but required for correctness)**
- For each channel (Shopify, Amazon, etc.), add nodes that output items shaped like:
  - `{ channel: "Shopify", revenue: 12345.67 }` (or one item per transaction)
- Merge these streams (e.g., with **Merge** nodes in “Append” mode) into a single stream that feeds the aggregation code node.
- If you already have a single API that returns all channels, use **HTTP Request** and then **Item Lists / Code** to normalize items.

5) **Add aggregation**
- Add **Code** node named **Aggregate Multi-Channel Revenue**.
- Paste the provided aggregation JS.
- Connect:
  - Revenue-source merged output → Aggregate Multi-Channel Revenue
  - (Optionally) Workflow Configuration → Aggregate Multi-Channel Revenue only if you intentionally pass config through; otherwise do not (it pollutes aggregation inputs).

6) **Add historical fetch**
- Add **HTTP Request** node named **Fetch Historical Revenue Data**.
- Set URL to `{{ $('Workflow Configuration').first().json.historicalDataApiUrl }}` (recommended) or directly the endpoint.
- Add header `Content-Type: application/json`.
- Configure authentication to match your API (API key/OAuth2/etc).
- Connect Workflow Configuration → Fetch Historical Revenue Data.

7) **Merge current + historical**
- Add **Merge** node **Merge Historical and Current Data**:
  - Mode: Combine (or preferably “Append” if you want one list; combine-by-position is fragile).
- Connect:
  - Fetch Historical Revenue Data → Merge input 0
  - Aggregate Multi-Channel Revenue → Merge input 1

8) **Summarize statistics**
- Add **Summarize** node **Calculate Revenue Statistics**.
- Summarize field `revenue`: sum/average/min/max (as in JSON).
- Connect Merge → Summarize.

9) **Detect anomalies**
- Add **Code** node **Detect Anomalies**, paste provided JS.
- Connect Summarize → Detect Anomalies.

10) **Fix and add routing IF**
- Add **IF** node **Check Tax Threshold**.
- Use condition (recommended fix):  
  - Boolean equals: `{{ $json.anomalyDetection.isAnomaly }}` is true
- Connect Detect Anomalies → IF.

11) **Alert path**
- Add **Set** node **Format Alert Data** connected to IF (true).
- Populate fields that match the Gmail template, or simplify the template to match your fields. If keeping current email template, create fields like:
  - `severityLevel`, `anomalyType`, `expectedValue`, `actualValue`, `deviation`, `recommendedAction1/2/3`, `impactDescription`, `timestamp`
- Add **Gmail** node **Send Anomaly Alert**.
  - To: `{{ $('Workflow Configuration').first().json.taxAgentEmail }}`
  - Subject/body as desired.
- Configure **Gmail OAuth2** credential in n8n and select it in the node.

12) **Non-alert path: revenue tier switch**
- Add **Switch** node **Route by Revenue Tier** connected from IF (false).
- Rules based on `{{ $json.totalRevenue }}` (or whichever field you standardize on):
  - Small: `< {{ $('Workflow Configuration').first().json.smallBusinessThreshold }}`
  - Medium: between small and medium thresholds
  - Fallback: Large
- Ensure **fallback output is connected** (important).

13) **Small business deductions**
- Add **Code** node **Calculate Tax Deductions**, paste provided JS.
- Connect Switch (Small) → Calculate Tax Deductions.

14) **Medium/large progressive tax**
- Add **Code** node **Apply Progressive Tax Brackets**, paste provided JS.
- Connect Switch (Medium) and Switch (Fallback/Large) → Apply Progressive Tax Brackets (either by wiring both or by using a second switch output connection).

15) **Merge tier results**
- Add **Merge** node **Combine Tax Calculations**.
- Prefer a design that does not require both inputs simultaneously (current combine-by-position is risky). Options:
  - Replace with a **NoOp** merge by using a single path, or
  - Use a **Merge (Append)** and then a later **Code** node to unify schema.
- Connect:
  - Calculate Tax Deductions → Combine Tax Calculations input 0
  - Apply Progressive Tax Brackets → Combine Tax Calculations input 1

16) **Calculate tax liabilities**
- Add **Code** node **Calculate Tax Liabilities**, paste provided JS.
- Ensure upstream provides the expected fields (`totalRevenue`, `domesticRevenue`, `internationalRevenue`) or adjust the code to use `overallTotal` and your channel split logic.
- Connect Combine Tax Calculations → Calculate Tax Liabilities.

17) **Forecast**
- Add **Code** node **Generate Rolling Forecast**, paste provided JS.
- Update it to use `config.forecastMonths` if desired and ensure it reads the correct revenue field.
- Connect Calculate Tax Liabilities → Generate Rolling Forecast.

18) **Scenario analysis**
- Add **Code** node **Generate Scenario Analysis**, paste provided JS.
- Connect Generate Rolling Forecast → Generate Scenario Analysis.

19) **Report formatting**
- Add **Set** node **Format Report Data**.
- Map fields consistently from prior nodes (recommended: map from `taxCalculations.*` and aggregate forecast arrays).
- Connect:
  - Generate Rolling Forecast → Format Report Data
  - Generate Scenario Analysis → Format Report Data (only if you intend multiple outputs; otherwise aggregate scenarios first)

20) **Send report**
- Add **Gmail** node **Send Report to Tax Agent**.
- To: `{{ $('Workflow Configuration').first().json.taxAgentEmail }}`
- Configure Gmail OAuth2 credential.

21) **Store in Google Sheets**
- Add **Google Sheets** node **Store in Google Sheets**.
- Credential: Google Sheets OAuth2.
- Operation: Append or Update
- Spreadsheet ID: your document
- Sheet name: “Tax Forecasts”
- Matching column: `reportDate`
- Connect Format Report Data → Google Sheets.

22) **Store in Airtable**
- Add **Airtable** node **Store in Airtable**.
- Set Base ID + Table ID.
- Map fields explicitly to match Airtable columns.
- Connect Format Report Data → Airtable.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Configure OpenAI API key for anomaly detection and tax analysis” is mentioned, but no OpenAI node exists in the provided workflow. | From Sticky Note1 (“Setup Steps”). Consider adding an OpenAI node to generate anomaly explanations/recommended actions. |
| Several nodes contain schema mismatches (anomaly flags, tax fields, report template fields). | Impacts correctness: alerts may never trigger; report may contain empty values unless mappings are fixed. |
| Workflow concept mentions jurisdiction routing, but implementation routes by revenue tier. | From Sticky Note4/Sticky Note2; adjust switch logic if jurisdiction-based compliance is required. |
| Placeholders must be replaced: Tax agent email, API endpoint, Google Sheets document ID, Airtable base/table IDs. | Present in Workflow Configuration, HTTP Request, Google Sheets, Airtable nodes. |

Disclaimer (provided): Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.