Consolidate Stripe, PayPal, Shopify and bank revenue and prepare tax filings with OpenAI

https://n8nworkflows.xyz/workflows/consolidate-stripe--paypal--shopify-and-bank-revenue-and-prepare-tax-filings-with-openai-11899


# Consolidate Stripe, PayPal, Shopify and bank revenue and prepare tax filings with OpenAI

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow title (provided):** Consolidate Stripe, PayPal, Shopify and bank revenue and prepare tax filings with OpenAI  
**Workflow name (in JSON):** Automate revenue collection and tax submission across multiple sources  
**Status:** inactive

**Purpose:**  
On a schedule (monthly), this workflow pulls revenue/transaction data from **Stripe**, **PayPal**, **Shopify**, and a **bank feed API**, normalizes them into a unified structure, merges all transactions, uses **OpenAI (via LangChain Agent node)** to categorize income for tax purposes, computes reporting-period totals, then generates **CSV** and **XML** outputs and either **submits via a government API** or **emails a tax agent**, and finally **archives** the result to Google Drive.

**Target use cases:**
- Monthly/quarterly tax preparation for e-commerce/SaaS with multi-channel revenue
- Automated revenue reconciliation and classification
- Producing machine-consumable filing artifacts (CSV/XML) plus human review via email

### Logical blocks
1. **1.1 Scheduling & configuration** (defines tax period, endpoints, submission method)
2. **1.2 Multi-source data collection** (Stripe, PayPal, Shopify, bank feed)
3. **1.3 Data normalization** (convert each source into a standard transaction schema)
4. **1.4 Consolidation** (merge all transactions into one aggregated payload)
5. **1.5 AI categorization** (OpenAI agent + structured output parsing)
6. **1.6 Totals computation & report generation** (code-based totals; CSV + XML creation)
7. **1.7 Submission routing & delivery** (IF: API submission vs email to tax agent)
8. **1.8 Archival** (Google Drive upload)

---

## 2. Block-by-Block Analysis

### 2.1 Scheduling & configuration

**Overview:**  
Triggers the workflow on a schedule and sets shared parameters (tax period boundaries, endpoints, submission method, email). These values are referenced by later nodes via expressions.

**Nodes involved:**
- Monthly revenue aggregation
- Workflow Configuration

#### Node: Monthly revenue aggregation
- **Type / role:** Schedule Trigger — entry point that starts workflow automatically.
- **Configuration (interpreted):** Runs daily at a configured hour (rule uses `triggerAtHour: 2`). Despite the node name “Monthly…”, the schedule shown is hour-based; confirm desired cadence in UI.
- **Connections:**
  - **Output:** → Workflow Configuration
- **Edge cases / failures:**
  - Misconfigured schedule interval could run more frequently than intended (e.g., daily at 02:00 rather than monthly).
  - Timezone differences (instance timezone vs business timezone) can shift period boundaries.

#### Node: Workflow Configuration
- **Type / role:** Set — central parameter builder.
- **Key configuration choices:**
  - `taxPeriodStart` = previous month start: `{{ $now.minus({ months: 1 }).startOf('month').toISO() }}`
  - `taxPeriodEnd` = previous month end: `{{ $now.minus({ months: 1 }).endOf('month').toISO() }}`
  - `submissionMethod` = `"api"` (controls IF routing)
  - `taxAgentEmail`, `governmentApiUrl`, `bankFeedApiUrl` are placeholders to be replaced.
  - **Include other fields:** true (passes through any incoming fields; here mainly trigger metadata)
- **Connections:**
  - **Input:** Monthly revenue aggregation
  - **Outputs (fan-out):** → Get Stripe Transactions, Get PayPal Transactions, Get Shopify Orders, Get Bank Feed Data
- **Edge cases / failures:**
  - If placeholders aren’t replaced, downstream nodes will fail (invalid URL / missing email).
  - `$now` boundaries rely on server time; fiscal calendars may differ (e.g., 4-4-5 retail calendar).

---

### 2.2 Multi-source data collection

**Overview:**  
Fetches raw transactions/orders from each source for the defined period (Shopify/bank explicitly filtered by date; Stripe/PayPal currently not filtered).

**Nodes involved:**
- Get Stripe Transactions
- Get PayPal Transactions
- Get Shopify Orders
- Get Bank Feed Data

#### Node: Get Stripe Transactions
- **Type / role:** Stripe node — retrieves Stripe charges.
- **Configuration:**
  - Resource: **Charge**
  - Operation: **Get All**
  - Limit: **100**
  - **No period filter configured** (important).
- **Connections:**
  - **Input:** Workflow Configuration
  - **Output:** → Normalize Stripe Data
- **Requirements:**
  - Stripe credentials configured in n8n (API key).
- **Edge cases / failures:**
  - Only first 100 charges returned; older/more charges ignored unless pagination/returnAll configured.
  - Missing date filtering means it may pull charges outside the tax period.
  - Auth errors if key revoked; rate limiting.

#### Node: Get PayPal Transactions
- **Type / role:** PayPal node — fetches a payout item.
- **Configuration:**
  - Resource: **payoutItem**
  - Uses a single `payoutItemId` placeholder.
  - **Not a “list transactions for period”** operation; returns one payout item by ID.
- **Connections:**
  - **Input:** Workflow Configuration
  - **Output:** → Normalize PayPal Data
- **Requirements:**
  - PayPal credentials (client/secret or OAuth depending on node setup).
- **Edge cases / failures:**
  - Placeholder payout item ID must be replaced or dynamically sourced; otherwise it won’t represent a period’s revenue.
  - Output shape differs depending on PayPal API response; normalization may break if fields missing.

#### Node: Get Shopify Orders
- **Type / role:** Shopify node — retrieves orders within a date range.
- **Configuration:**
  - Operation: **Get All**
  - Return All: **true**
  - Filters:
    - `createdAtMin` = taxPeriodStart from Workflow Configuration
    - `createdAtMax` = taxPeriodEnd from Workflow Configuration
- **Connections:**
  - **Input:** Workflow Configuration
  - **Output:** → Normalize Shopify Data
- **Requirements:**
  - Shopify credentials (Admin API access token, shop domain).
- **Edge cases / failures:**
  - Timezone on Shopify timestamps vs ISO boundaries can cause off-by-one-day inclusions/exclusions.
  - Large stores: returnAll can be slow; API throttling.

#### Node: Get Bank Feed Data
- **Type / role:** HTTP Request — calls a bank feed API endpoint.
- **Configuration:**
  - URL: from `bankFeedApiUrl`
  - Query params:
    - `start_date` = taxPeriodStart
    - `end_date` = taxPeriodEnd
  - `sendQuery: true`
- **Connections:**
  - **Input:** Workflow Configuration
  - **Output:** → Normalize Bank Data
- **Requirements:**
  - Bank feed API authentication is not shown (no headers/credentials in this node); must be added (e.g., API key header, OAuth2).
- **Edge cases / failures:**
  - 401/403 if auth missing.
  - Response format must match normalization expectations (`transaction_id`, `amount`, etc.).
  - Pagination not handled.

---

### 2.3 Data normalization

**Overview:**  
Transforms each provider’s raw payload into a consistent “transaction-like” schema so they can be merged and categorized uniformly.

**Nodes involved:**
- Normalize Stripe Data
- Normalize PayPal Data
- Normalize Shopify Data
- Normalize Bank Data

#### Node: Normalize Stripe Data
- **Type / role:** Set — field mapping + normalization.
- **Key mappings:**
  - `source = "Stripe"`
  - `transactionId = $json.id`
  - `amount = $json.amount / 100` (Stripe amounts are in cents)
  - `currency = $json.currency`
  - `date = new Date($json.created * 1000).toISOString()` (Stripe created is epoch seconds)
  - `description = $json.description`
  - `customerEmail = $json.billing_details?.email` (optional chaining)
  - Include other fields: true
- **Connections:**  
  - Input: Get Stripe Transactions  
  - Output: Merge All Revenue Sources
- **Edge cases / failures:**
  - Some charges may have null `description` or missing `billing_details`.
  - Currency casing (Stripe often lower-case) may differ from other sources.

#### Node: Normalize PayPal Data
- **Type / role:** Set — field mapping.
- **Key mappings (expects payout item structure):**
  - `source = "PayPal"`
  - `transactionId = $json.payout_item_id`
  - `amount = $json.payout_item.amount.value`
  - `currency = $json.payout_item.amount.currency`
  - `date = $json.time_processed`
  - `description = $json.payout_item.transaction_note`
  - Include other fields: true
- **Connections:**  
  - Input: Get PayPal Transactions  
  - Output: Merge All Revenue Sources
- **Edge cases / failures:**
  - If PayPal response differs (or fields absent), expressions may resolve to `undefined`.
  - `amount.value` may be a string; later code should parse consistently.

#### Node: Normalize Shopify Data
- **Type / role:** Set — field mapping for orders.
- **Key mappings:**
  - `source = "Shopify"`
  - `transactionId = $json.id`
  - `amount = $json.total_price` (often string in Shopify)
  - `currency = $json.currency`
  - `date = $json.created_at`
  - `description = $json.name`
  - `customerEmail = $json.email`
  - Include other fields: true
- **Connections:**  
  - Input: Get Shopify Orders  
  - Output: Merge All Revenue Sources
- **Edge cases / failures:**
  - `total_price` may be string; downstream math must parseFloat.
  - Orders can be refunded/voided; this workflow does not adjust for refunds unless Shopify node filters them.

#### Node: Normalize Bank Data
- **Type / role:** Set — field mapping for bank feed transactions.
- **Key mappings:**
  - `source = "Bank"`
  - `transactionId = $json.transaction_id`
  - `amount = $json.amount`
  - `currency = $json.currency`
  - `date = $json.date`
  - `description = $json.description`
  - Include other fields: true
- **Connections:**  
  - Input: Get Bank Feed Data  
  - Output: Merge All Revenue Sources
- **Edge cases / failures:**
  - Bank feeds commonly include debits/credits; this workflow assumes all are “revenue” unless filtered elsewhere.
  - Date formats may not be ISO; downstream parsing may fail.

---

### 2.4 Consolidation

**Overview:**  
Aggregates all normalized items into a single array field so the AI node can categorize a complete multi-source dataset in one call.

**Nodes involved:**
- Merge All Revenue Sources

#### Node: Merge All Revenue Sources
- **Type / role:** Aggregate — “aggregate all item data” into one object.
- **Configuration:**
  - Aggregation mode: **aggregateAllItemData**
  - Destination field: `allTransactions`
- **Connections:**
  - Inputs: Normalize Stripe Data, Normalize PayPal Data, Normalize Shopify Data, Normalize Bank Data
  - Output: AI Income Categorizer
- **Edge cases / failures:**
  - If any branch returns 0 items, aggregate still works, but may reduce coverage.
  - Large transaction volumes can produce a very large JSON string later (token limits at AI step).

---

### 2.5 AI categorization (OpenAI via LangChain)

**Overview:**  
Sends the merged transaction list to an AI agent with a system message instructing categorization and taxability metadata, and enforces a strict JSON schema through a structured output parser.

**Nodes involved:**
- AI Income Categorizer
- OpenAI Model
- Structured Output Parser

#### Node: AI Income Categorizer
- **Type / role:** LangChain Agent (`@n8n/n8n-nodes-langchain.agent`) — orchestrates model + parser.
- **Configuration:**
  - Text prompt: `Categorize these revenue transactions: {{ JSON.stringify($json.allTransactions) }}`
  - System message defines required fields per transaction:
    - incomeCategory (e.g., Product Sales, Service Revenue, etc.)
    - taxable (boolean)
    - taxRate (decimal per system text, e.g. 0.10)
    - notes
  - `hasOutputParser: true`
- **Connections:**
  - Main input: Merge All Revenue Sources
  - AI language model input: from OpenAI Model node (ai_languageModel)
  - Output parser input: from Structured Output Parser node (ai_outputParser)
  - Main output: → Calculate Period Totals
- **Version requirements:** Node typeVersion 3 (ensure n8n with LangChain nodes installed/enabled).
- **Edge cases / failures:**
  - Prompt can exceed model context window (stringified transactions).
  - Model may not comply perfectly; parser will fail if output doesn’t match schema.
  - System message says `taxRate` as decimal (0.10) but later code treats rate as percent (divides by 100). This is a critical mismatch (see totals block).

#### Node: OpenAI Model
- **Type / role:** Chat model provider (`lmChatOpenAi`) used by the agent.
- **Configuration:**
  - Model: `gpt-4.1-mini`
- **Credentials:** OpenAI API credential required.
- **Connections:**
  - Output (ai_languageModel): → AI Income Categorizer
- **Edge cases / failures:**
  - API quota/rate limits.
  - Model name availability depends on account/region.

#### Node: Structured Output Parser
- **Type / role:** Structured output parser enforcing a manual JSON schema.
- **Configuration:**
  - Schema: object with `transactions: array` of objects containing:
    - transactionId, source, amount, currency, date, description
    - incomeCategory, taxable, taxRate, notes
- **Connections:**
  - Output (ai_outputParser): → AI Income Categorizer
- **Edge cases / failures:**
  - If the agent returns fields like `category` instead of `incomeCategory`, parser can still pass (schema doesn’t forbid extras) but downstream code expects different names; or parser may fail if required structure differs.
  - Schema as defined does not mark properties as required; however the parser still expects this overall shape.

---

### 2.6 Totals computation & report generation

**Overview:**  
Computes totals (by category and overall) and generates CSV and XML. As currently wired, these code nodes contain field-name mismatches with the AI output schema and likely require adjustments to work correctly.

**Nodes involved:**
- Calculate Period Totals
- Format as CSV
- Format as XML

#### Node: Calculate Period Totals
- **Type / role:** Code — computes totals for tax filing.
- **Behavior (interpreted):**
  - Iterates through **all incoming items** and aggregates:
    - totals by `category`
    - totalRevenue, totalTaxableAmount, totalTax, transactionCount
    - min/max dates
  - Returns a single item shaped like:
    - `summary` (totals)
    - `categories` (array of category summary rows)
    - `transactions` (the original items)
- **Critical data-shape mismatch:**
  - The AI parser schema outputs `incomeCategory`, `taxable`, `taxRate` (decimal per system prompt).
  - This code expects `data.category`, `data.taxableAmount`, and treats `taxRate` as a **percentage** (`taxableAmount * (taxRate / 100)`).
- **Connections:**
  - Input: AI Income Categorizer
  - Outputs: → Format as CSV, → Format as XML
- **Edge cases / failures:**
  - If AI node outputs a single object like `{ transactions: [...] }` (per schema), then this Code node will see **one item** whose `item.json` contains `transactions` array, not individual transactions—so totals will be wrong unless you split items first.
  - Tax calculation likely wrong due to rate handling (decimal vs percent).

#### Node: Format as CSV
- **Type / role:** Code — converts items to CSV and attaches as binary.
- **Behavior:**
  - Treats **each input item as one transaction row**.
  - Uses headers including Category/Tax fields.
  - Outputs:
    - `json.csv` (string)
    - `binary.data` with base64 content and filename `revenue_report_YYYY-MM-DD.csv`
- **Mismatch / missing wiring:**
  - Later nodes refer to `csvData`, but this node outputs binary under `binary.data` and JSON under `csv`.
  - Its output does **not** create a `csvData` property unless renamed.
- **Connections:**
  - Input: Calculate Period Totals
  - Output: Check Submission Method
- **Edge cases / failures:**
  - If input is a single “summary” object (not transactions), CSV will contain one row of mostly blanks.
  - Amount/tax fields may be objects/arrays if passed incorrectly.

#### Node: Format as XML
- **Type / role:** Code — builds an XML “TaxFilingReport”.
- **Behavior:**
  - Builds header and summary sections using `items[0]?.json?.totals`, `categoryTotals`, etc.
  - Then iterates over `items` as if each is a transaction.
- **Major mismatch:**
  - Calculate Period Totals outputs `summary` not `totals`, and `byCategory` not `categoryTotals`. Also it returns **one item**, not many.
  - Transaction count uses `items.length`, which would be `1` if only the summary item is passed.
- **Connections:**
  - Input: Calculate Period Totals
  - Output: Archive to Google Drive
- **Edge cases / failures:**
  - XML may be structurally valid but semantically wrong (totals zero, transactions missing).
  - Date parsing may fail if non-ISO dates are provided.

---

### 2.7 Submission routing & delivery

**Overview:**  
Routes output based on submission method: if `api` then POST to government endpoint, else email to tax agent with CSV attachment.

**Nodes involved:**
- Check Submission Method
- Submit to Government API
- Email to Tax Agent

#### Node: Check Submission Method
- **Type / role:** IF — branching logic.
- **Condition:** `{{ $('Workflow Configuration').first().json.submissionMethod }} == "api"`
- **Connections:**
  - Input: Format as CSV
  - True output: → Submit to Government API
  - False output: → Email to Tax Agent
- **Edge cases / failures:**
  - If submissionMethod contains whitespace/case differences, branch may misroute.
  - If Format as CSV doesn’t run/produce output, IF won’t execute.

#### Node: Submit to Government API
- **Type / role:** HTTP Request — submits CSV to an external government tax API endpoint.
- **Configuration:**
  - URL: `governmentApiUrl` from Workflow Configuration
  - Method: POST
  - Content-Type: raw `text/csv`
  - Body: `{{ $json.csvData }}`
- **Critical mismatch:**
  - Upstream CSV node outputs `json.csv` and `binary.data`; no `csvData` is created.
  - This will send an empty body unless you map correctly (e.g., `{{$json.csv}}`) or send binary.
- **Connections:**
  - Input: Check Submission Method (true branch)
  - Output: Archive to Google Drive
- **Edge cases / failures:**
  - API auth not configured (no headers besides Content-Type).
  - Large payload timeouts.
  - Wrong body mapping yields invalid filing.

#### Node: Email to Tax Agent
- **Type / role:** Gmail — sends an email with attachment.
- **Configuration:**
  - To: `taxAgentEmail` from Workflow Configuration
  - Subject: includes taxPeriodStart/taxPeriodEnd
  - HTML message includes revenue summary from: `$('Calculate Period Totals').first().json`
  - Attachment: binary property `csvData`
- **Critical mismatch:**
  - Format as CSV binary property is `data`, not `csvData`.
  - Attachment configuration references `csvData`, so attachment will be missing unless renamed/mapped.
- **Credentials:** Gmail OAuth2 required.
- **Connections:**
  - Input: Check Submission Method (false branch)
  - Output: Archive to Google Drive
- **Edge cases / failures:**
  - Gmail OAuth token expiry/permission issues.
  - Message body can become very large if summary JSON is large.

---

### 2.8 Archival

**Overview:**  
Archives the tax filing output to Google Drive for retention/audit.

**Nodes involved:**
- Archive to Google Drive

#### Node: Archive to Google Drive
- **Type / role:** Google Drive — uploads a file.
- **Configuration:**
  - File name: `tax_filing_YYYY-MM-DD.csv`
  - Drive: “My Drive”, folder: root
  - Simplify output: true
- **Missing parameter:**  
  The node configuration shown does not explicitly map **file content/binary property** to upload. In n8n, Google Drive “Upload” typically requires a **Binary Property** selection. This node may be incomplete or relying on defaults not visible here.
- **Connections:**
  - Inputs: Format as XML, Email to Tax Agent, Submit to Government API
- **Credentials:** Google Drive OAuth2 required.
- **Edge cases / failures:**
  - If it receives XML path, filename still `.csv` (content-type mismatch).
  - If no binary data is provided, upload will fail.
  - Permissions/quota errors.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Monthly revenue aggregation | Schedule Trigger | Periodic workflow start | — | Workflow Configuration | ## How It Works  \nConsolidates daily revenue from Stripe, PayPal, Shopify, and bank feeds into a single system. The workflow automatically normalizes data across payment sources, uses AI to categorize income transactions, calculates reporting-period totals, and generates tax-compliant CSV and XML submissions. Designed for e-commerce businesses, SaaS companies, and multi-channel retailers managing complex revenue streams, it eliminates manual reconciliation, reduces filing errors, and speeds up financial reporting by automating the entire pipeline from data collection to government submission. |
| Workflow Configuration | Set | Defines tax period, endpoints, routing | Monthly revenue aggregation | Get Stripe Transactions; Get PayPal Transactions; Get Shopify Orders; Get Bank Feed Data | ## Setup Steps\n1. Connect Stripe/PayPal/Shopify accounts with API keys to respective nodes.\n2. Configure bank feed authentication \n3. Set OpenAI credentials for AI Income Categorizer node.\n4. Link Google Drive and Gmail . |
| Get Stripe Transactions | Stripe | Pull Stripe charges | Workflow Configuration | Normalize Stripe Data | ## Multi-Source Data Collection  \n**What:** Retrieves raw transaction data from four distinct payment sources  \n**Why:** Ensures complete revenue visibility across all channels. |
| Get PayPal Transactions | PayPal | Pull PayPal payout item | Workflow Configuration | Normalize PayPal Data | ## Multi-Source Data Collection  \n**What:** Retrieves raw transaction data from four distinct payment sources  \n**Why:** Ensures complete revenue visibility across all channels. |
| Get Shopify Orders | Shopify | Pull Shopify orders for period | Workflow Configuration | Normalize Shopify Data | ## Multi-Source Data Collection  \n**What:** Retrieves raw transaction data from four distinct payment sources  \n**Why:** Ensures complete revenue visibility across all channels. |
| Get Bank Feed Data | HTTP Request | Pull bank feed transactions for period | Workflow Configuration | Normalize Bank Data | ## Multi-Source Data Collection  \n**What:** Retrieves raw transaction data from four distinct payment sources  \n**Why:** Ensures complete revenue visibility across all channels. |
| Normalize Stripe Data | Set | Standardize Stripe fields | Get Stripe Transactions | Merge All Revenue Sources | ## Data Normalization\n**What:** Applies format-standardization and transformation rules to each data source independently.\n**Why:** Creates consistent data structure |
| Normalize PayPal Data | Set | Standardize PayPal fields | Get PayPal Transactions | Merge All Revenue Sources | ## Data Normalization\n**What:** Applies format-standardization and transformation rules to each data source independently.\n**Why:** Creates consistent data structure |
| Normalize Shopify Data | Set | Standardize Shopify fields | Get Shopify Orders | Merge All Revenue Sources | ## Data Normalization\n**What:** Applies format-standardization and transformation rules to each data source independently.\n**Why:** Creates consistent data structure |
| Normalize Bank Data | Set | Standardize bank fields | Get Bank Feed Data | Merge All Revenue Sources | ## Data Normalization\n**What:** Applies format-standardization and transformation rules to each data source independently.\n**Why:** Creates consistent data structure |
| Merge All Revenue Sources | Aggregate | Combine normalized streams into one payload | Normalize Stripe Data; Normalize PayPal Data; Normalize Shopify Data; Normalize Bank Data | AI Income Categorizer |  |
| AI Income Categorizer | LangChain Agent | AI categorization with structured output | Merge All Revenue Sources | Calculate Period Totals | ## AI-Powered Income Categorization\n**What:** Use all normalized streams and categorizes income using AI-powered transaction analysis.\n**Why:** Automates income classification and detects anomalies |
| OpenAI Model | OpenAI Chat Model | Provides LLM to agent | — | AI Income Categorizer | ## AI-Powered Income Categorization\n**What:** Use all normalized streams and categorizes income using AI-powered transaction analysis.\n**Why:** Automates income classification and detects anomalies |
| Structured Output Parser | Structured Output Parser | Enforces JSON schema for AI output | — | AI Income Categorizer | ## AI-Powered Income Categorization\n**What:** Use all normalized streams and categorizes income using AI-powered transaction analysis.\n**Why:** Automates income classification and detects anomalies |
| Calculate Period Totals | Code | Compute totals and period summary | AI Income Categorizer | Format as CSV; Format as XML | ## Tax Period Calculation and submission\n**What:** Computes period totals and tax period summaries with validation checks.\n**Why:** Ensures accuracy and compliance-ready figures for regulatory submission. |
| Format as CSV | Code | Produce CSV + binary attachment | Calculate Period Totals | Check Submission Method | ## Tax Period Calculation and submission\n**What:** Computes period totals and tax period summaries with validation checks.\n**Why:** Ensures accuracy and compliance-ready figures for regulatory submission. |
| Check Submission Method | IF | Route API vs email submission | Format as CSV | Submit to Government API; Email to Tax Agent | ## Tax Period Calculation and submission\n**What:** Computes period totals and tax period summaries with validation checks.\n**Why:** Ensures accuracy and compliance-ready figures for regulatory submission. |
| Submit to Government API | HTTP Request | POST filing payload to government endpoint | Check Submission Method | Archive to Google Drive | ## Tax Period Calculation and submission\n**What:** Computes period totals and tax period summaries with validation checks.\n**Why:** Ensures accuracy and compliance-ready figures for regulatory submission. |
| Email to Tax Agent | Gmail | Send filing package for review/submission | Check Submission Method | Archive to Google Drive | ## Tax Period Calculation and submission\n**What:** Computes period totals and tax period summaries with validation checks.\n**Why:** Ensures accuracy and compliance-ready figures for regulatory submission. |
| Format as XML | Code | Produce XML filing representation | Calculate Period Totals | Archive to Google Drive | ## Tax Period Calculation and submission\n**What:** Computes period totals and tax period summaries with validation checks.\n**Why:** Ensures accuracy and compliance-ready figures for regulatory submission. |
| Archive to Google Drive | Google Drive | Store filing output | Submit to Government API; Email to Tax Agent; Format as XML | — | ## Tax Period Calculation and submission\n**What:** Computes period totals and tax period summaries with validation checks.\n**Why:** Ensures accuracy and compliance-ready figures for regulatory submission. |
| Sticky Note | Sticky Note | Comment | — | — | ## Customization\nModify normalization rules per jurisdiction; add expense categories to AI prompt;  \n\n## Benefits\nEliminates manual reconciliation; reduces tax filing time by 80%; improves accuracy; |
| Sticky Note1 | Sticky Note | Comment | — | — | ## Prerequisites\nStripe, PayPal, Shopify, or bank APIs; OpenAI account; Google Workspace;  \n\n## Use Cases\nQuarterly tax preparation for e-commerce; multi-channel revenue reconciliation; |
| Sticky Note2 | Sticky Note | Comment | — | — | ## Setup Steps\n1. Connect Stripe/PayPal/Shopify accounts with API keys to respective nodes.\n2. Configure bank feed authentication \n3. Set OpenAI credentials for AI Income Categorizer node.\n4. Link Google Drive and Gmail . |
| Sticky Note3 | Sticky Note | Comment | — | — | ## How It Works  \nConsolidates daily revenue from Stripe, PayPal, Shopify, and bank feeds into a single system. The workflow automatically normalizes data across payment sources, uses AI to categorize income transactions, calculates reporting-period totals, and generates tax-compliant CSV and XML submissions. Designed for e-commerce businesses, SaaS companies, and multi-channel retailers managing complex revenue streams, it eliminates manual reconciliation, reduces filing errors, and speeds up financial reporting by automating the entire pipeline from data collection to government submission. |
| Sticky Note4 | Sticky Note | Comment | — | — | ## AI-Powered Income Categorization\n**What:** Use all normalized streams and categorizes income using AI-powered transaction analysis.\n**Why:** Automates income classification and detects anomalies |
| Sticky Note5 | Sticky Note | Comment | — | — | ## Tax Period Calculation and submission\n**What:** Computes period totals and tax period summaries with validation checks.\n**Why:** Ensures accuracy and compliance-ready figures for regulatory submission. |
| Sticky Note6 | Sticky Note | Comment | — | — | ## Data Normalization\n**What:** Applies format-standardization and transformation rules to each data source independently.\n**Why:** Creates consistent data structure |
| Sticky Note7 | Sticky Note | Comment | — | — | ## Multi-Source Data Collection  \n**What:** Retrieves raw transaction data from four distinct payment sources  \n**Why:** Ensures complete revenue visibility across all channels. |

---

## 4. Reproducing the Workflow from Scratch (manual rebuild)

1. **Create Trigger**
   1. Add node **Schedule Trigger** named **“Monthly revenue aggregation”**.
   2. Configure schedule to match your real requirement (monthly vs daily). In the provided workflow it triggers at **02:00**; adjust interval accordingly.

2. **Add configuration node**
   1. Add **Set** node named **“Workflow Configuration”** connected from the trigger.
   2. Add fields:
      - `taxPeriodStart` (String): previous month start ISO using Luxon:  
        `{{$now.minus({ months: 1 }).startOf('month').toISO()}}`
      - `taxPeriodEnd` (String): previous month end ISO:  
        `{{$now.minus({ months: 1 }).endOf('month').toISO()}}`
      - `submissionMethod` (String): `api` (or `email`)
      - `taxAgentEmail` (String): your tax agent email
      - `governmentApiUrl` (String): your endpoint URL
      - `bankFeedApiUrl` (String): your bank feed endpoint URL
   3. Enable “Include Other Fields”.

3. **Add source collection nodes (parallel fan-out from configuration)**
   1. **Stripe**
      - Add **Stripe** node “Get Stripe Transactions”
      - Resource **Charge**, Operation **Get All**, Limit **100**
      - Add Stripe credentials (API key) in node credentials.
      - Connect: Workflow Configuration → Get Stripe Transactions
   2. **PayPal**
      - Add **PayPal** node “Get PayPal Transactions”
      - Configure to fetch a payout item by ID (as in JSON) or replace with a “list transactions” operation suitable for a period.
      - Add PayPal credentials.
      - Connect: Workflow Configuration → Get PayPal Transactions
   3. **Shopify**
      - Add **Shopify** node “Get Shopify Orders”
      - Operation **Get All**, Return All **true**
      - Set `createdAtMin` = `{{$('Workflow Configuration').first().json.taxPeriodStart}}`
      - Set `createdAtMax` = `{{$('Workflow Configuration').first().json.taxPeriodEnd}}`
      - Add Shopify credentials.
      - Connect: Workflow Configuration → Get Shopify Orders
   4. **Bank feed**
      - Add **HTTP Request** node “Get Bank Feed Data”
      - URL = `{{$('Workflow Configuration').first().json.bankFeedApiUrl}}`
      - Add query parameters:
        - `start_date` = `{{$('Workflow Configuration').first().json.taxPeriodStart}}`
        - `end_date` = `{{$('Workflow Configuration').first().json.taxPeriodEnd}}`
      - Add required auth (API key header/OAuth2) for your bank feed.
      - Connect: Workflow Configuration → Get Bank Feed Data

4. **Add normalization nodes (one per source)**
   1. Add **Set** node “Normalize Stripe Data”, connect from “Get Stripe Transactions”, map:
      - source “Stripe”; transactionId `{{$json.id}}`; amount `{{$json.amount/100}}`; currency; date from `created`; description; customerEmail from billing_details.email
   2. Add **Set** node “Normalize PayPal Data”, connect from PayPal node, map fields as in workflow.
   3. Add **Set** node “Normalize Shopify Data”, connect from Shopify node, map id/total_price/currency/created_at/name/email.
   4. Add **Set** node “Normalize Bank Data”, connect from bank HTTP node, map transaction_id/amount/currency/date/description.

5. **Add consolidation**
   1. Add **Aggregate** node “Merge All Revenue Sources”.
   2. Set aggregation mode to **Aggregate All Item Data**.
   3. Destination field name: `allTransactions`.
   4. Connect each normalization node output → Merge All Revenue Sources.

6. **Add AI categorization (LangChain)**
   1. Add **OpenAI Chat Model** node “OpenAI Model”.
      - Choose model `gpt-4.1-mini` (or an available equivalent).
      - Configure OpenAI credentials.
   2. Add **Structured Output Parser** node “Structured Output Parser”.
      - Schema type: manual
      - Define schema with top-level `{ transactions: [...] }` and per-transaction fields (transactionId, source, amount, currency, date, description, incomeCategory, taxable, taxRate, notes).
   3. Add **LangChain Agent** node “AI Income Categorizer”.
      - Text: `Categorize these revenue transactions: {{ JSON.stringify($json.allTransactions) }}`
      - System message: instruct categorization and output format (as provided).
      - Enable output parser.
   4. Connect:
      - Merge All Revenue Sources → AI Income Categorizer (main)
      - OpenAI Model → AI Income Categorizer (AI language model connection)
      - Structured Output Parser → AI Income Categorizer (AI output parser connection)

7. **Add totals and report generation**
   1. Add **Code** node “Calculate Period Totals” connected from AI Income Categorizer.
   2. Paste the provided JS code (then **fix the field mapping** to match AI output; see note below).
   3. Add **Code** node “Format as CSV” connected from Calculate Period Totals.
   4. Add **Code** node “Format as XML” also connected from Calculate Period Totals.

8. **Add submission routing**
   1. Add **IF** node “Check Submission Method”.
      - Condition: `{{$('Workflow Configuration').first().json.submissionMethod}}` equals `api`
   2. Connect: Format as CSV → Check Submission Method
   3. Add **HTTP Request** node “Submit to Government API” on the **true** branch.
      - URL: `{{$('Workflow Configuration').first().json.governmentApiUrl}}`
      - Method: POST
      - Content-Type: raw text/csv
      - Body: map from the CSV node output correctly (likely `{{$json.csv}}` or send binary)
   4. Add **Gmail** node “Email to Tax Agent” on the **false** branch.
      - To: `{{$('Workflow Configuration').first().json.taxAgentEmail}}`
      - Subject/message as desired; attach the CSV binary property (ensure property name matches).

9. **Add archival**
   1. Add **Google Drive** node “Archive to Google Drive”.
   2. Configure upload destination (Drive + folder) and ensure **Binary Property** is set to the CSV binary field (e.g., `data` if using this workflow’s CSV node unchanged).
   3. Connect:
      - Submit to Government API → Archive to Google Drive
      - Email to Tax Agent → Archive to Google Drive
      - (Optional) Format as XML → Archive to Google Drive (but ensure file extension/content matches)

**Important rebuild constraint:** as written, the workflow’s AI output structure, totals code, CSV/XML code, and submission/email nodes are not aligned (field names and item structure). Plan to add either:
- a **Split Out** step (e.g., Item Lists / Code) to turn `transactions[]` into individual items, and/or
- modify totals/CSV/XML nodes to consume the `transactions` array correctly, and
- rename/match CSV binary property to `csvData` if you want to keep downstream nodes unchanged.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Stripe, PayPal, Shopify, or bank APIs; OpenAI account; Google Workspace | Prerequisites (Sticky Note1) |
| Quarterly tax preparation for e-commerce; multi-channel revenue reconciliation | Use cases (Sticky Note1) |
| Connect Stripe/PayPal/Shopify accounts with API keys; configure bank feed auth; set OpenAI credentials; link Google Drive and Gmail | Setup steps (Sticky Note2) |
| Modify normalization rules per jurisdiction; add expense categories to AI prompt; eliminates manual reconciliation; reduces tax filing time by 80%; improves accuracy | Customization & benefits (Sticky Note) |
| Consolidates revenue streams, normalizes, AI-categorizes, totals, generates CSV/XML, submits/archives | How it works (Sticky Note3) |
| AI-powered categorization automates classification and detects anomalies | AI block note (Sticky Note4) |
| Computes period totals and compliance-ready figures for submission | Totals/submission note (Sticky Note5) |
| Standardization per data source to consistent structure | Normalization note (Sticky Note6) |
| Retrieves raw transaction data from four payment sources for full visibility | Collection note (Sticky Note7) |