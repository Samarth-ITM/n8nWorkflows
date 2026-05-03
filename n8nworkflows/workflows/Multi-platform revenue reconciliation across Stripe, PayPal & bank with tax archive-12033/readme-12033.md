Multi-platform revenue reconciliation across Stripe, PayPal & bank with tax archive

https://n8nworkflows.xyz/workflows/multi-platform-revenue-reconciliation-across-stripe--paypal---bank-with-tax-archive-12033


# Multi-platform revenue reconciliation across Stripe, PayPal & bank with tax archive

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow name:** Automated Revenue Reconciliation & Tax Evidence Archive  
**Stated title:** Multi-platform revenue reconciliation across Stripe, PayPal & bank with tax archive

**Purpose:**  
Runs monthly to pull revenue data from Stripe and PayPal plus bank statement deposits, normalize them into a consistent schema, reconcile processor revenue vs bank deposits (with a tolerance), generate an audit-ready monthly report (HTML intended for PDF), upload the report to Google Drive as an archival evidence artifact, and email a tax agent with a summary and archive link.

**Target use cases:** Accounting/finance teams and e-commerce operators needing consistent monthly reconciliation and audit evidence; detection of missing/late settlements and discrepancies.

### Logical blocks
1.1 **Scheduling & configuration**: Trigger monthly run and set environment parameters (API URLs, archive folder, tax agent email, thresholds).  
1.2 **Parallel data acquisition (Stripe/PayPal/Bank)**: Fetch last month’s transactions from each system via HTTP APIs.  
1.3 **Normalization & consolidation**: Map each source’s payload to common fields and merge processor datasets.  
1.4 **Reconciliation & mismatch decisioning**: Compare totals and decide whether to flag discrepancies.  
1.5 **Aggregation, reporting, archiving, notification**: Aggregate items, generate a detailed audit report, upload to Drive, notify the tax agent via Gmail.

---

## 2. Block-by-Block Analysis

### 1.1 Scheduling & configuration

**Overview:**  
Initiates the workflow on a monthly schedule and defines configurable parameters (API endpoints, archive folder, recipient email, tolerance). These values are referenced by later nodes via expressions.

**Nodes involved:**  
- Monthly Schedule  
- Workflow Configuration

#### Node: Monthly Schedule
- **Type / role:** `Schedule Trigger` — entrypoint, runs on a time-based rule.
- **Configuration (interpreted):** Triggers every month at **02:00** (server/instance timezone).
- **Inputs/outputs:** No inputs; outputs to **Workflow Configuration**.
- **Version notes:** typeVersion `1.3`.
- **Edge cases / failures:**
  - Timezone ambiguity (instance timezone vs expected business timezone).
  - If n8n instance is down at trigger time, run may be skipped unless catch-up is configured at instance level.

#### Node: Workflow Configuration
- **Type / role:** `Set` — central parameter store for the run.
- **Configuration (interpreted):**
  - `stripeApiUrl` (placeholder)
  - `paypalApiUrl` (placeholder)
  - `bankApiUrl` (placeholder)
  - `archiveFolderId` (placeholder; Google Drive folder ID)
  - `taxAgentEmail` (placeholder)
  - `reconciliationThreshold` = `0.01`
  - **includeOtherFields:** true (keeps incoming fields; usually irrelevant here since trigger has minimal JSON)
- **Key expressions/variables used downstream:**
  - `$('Workflow Configuration').first().json.<name>`
- **Outputs:** Fans out to three HTTP Request nodes (Stripe, PayPal, Bank).
- **Version notes:** typeVersion `3.4`.
- **Edge cases / failures:**
  - Placeholders not replaced → HTTP nodes fail (invalid URL), Drive upload fails (invalid folder), email fails (invalid recipient).
  - Threshold is defined here but **not actually used** by reconciliation code (hard-coded `0.01` and `$1.00` elsewhere).

---

### 1.2 Parallel data acquisition (Stripe/PayPal/Bank)

**Overview:**  
Retrieves last month’s activity from Stripe and PayPal and the corresponding bank statements for the same date range. All calls are via HTTP Request nodes using predefined credentials.

**Nodes involved:**  
- Get Stripe Revenue  
- Get PayPal Revenue  
- Get Bank Statements

#### Node: Get Stripe Revenue
- **Type / role:** `HTTP Request` — calls Stripe API endpoint for charges/balance transactions (endpoint is a placeholder).
- **Configuration (interpreted):**
  - URL from config: `stripeApiUrl`
  - Authentication: **predefinedCredentialType** (credential not shown in JSON; must exist in n8n)
  - Query parameters:
    - `created[gte]` = Unix timestamp for **start of previous month**
    - `created[lt]` = Unix timestamp for **start of current month**
  - `sendQuery: true`
- **Expressions:**
  - `{{ $now.minus({ months: 1 }).startOf('month').toUnixInteger() }}`
  - `{{ $now.startOf('month').toUnixInteger() }}`
- **Output:** To **Normalize Stripe Data**.
- **Version notes:** typeVersion `4.3`.
- **Edge cases / failures:**
  - Stripe API pagination not handled (only first page returned unless endpoint returns all).
  - Stripe may require specific headers/auth format; predefined credential must match.
  - Rate limits; transient 429/5xx.
  - Data shape mismatch if endpoint does not return items in the form the Set node expects (`$json.id`, `$json.amount`, etc.).

#### Node: Get PayPal Revenue
- **Type / role:** `HTTP Request` — calls PayPal reporting/transactions endpoint (placeholder URL).
- **Configuration (interpreted):**
  - URL from config: `paypalApiUrl`
  - Authentication: predefinedCredentialType
  - Query parameters:
    - `start_date` = ISO datetime start of previous month
    - `end_date` = ISO datetime start of current month
- **Expressions:**
  - `{{ $now.minus({ months: 1 }).startOf('month').toISO() }}`
  - `{{ $now.startOf('month').toISO() }}`
- **Output:** To **Normalize PayPal Data**.
- **Version notes:** typeVersion `4.3`.
- **Edge cases / failures:**
  - Pagination not handled.
  - PayPal APIs can require specific date formats/timezones; ISO may be fine but endpoint-dependent.
  - Returned schema must include `transaction_info.*` as used later.

#### Node: Get Bank Statements
- **Type / role:** `HTTP Request` — calls bank statements API (placeholder URL).
- **Configuration (interpreted):**
  - URL from config: `bankApiUrl`
  - Authentication: predefinedCredentialType
  - Query parameters:
    - `from_date` = ISO date start of previous month
    - `to_date` = ISO date start of current month
- **Expressions:**
  - `{{ $now.minus({ months: 1 }).startOf('month').toISODate() }}`
  - `{{ $now.startOf('month').toISODate() }}`
- **Output:** To **Normalize Bank Data**.
- **Version notes:** typeVersion `4.3`.
- **Edge cases / failures:**
  - Bank APIs often require additional auth steps (OAuth refresh, signing).
  - Statement availability delays; missing transactions for end-of-month settlements.
  - Returned schema must include `transaction_id`, `booking_date`, etc.

---

### 1.3 Normalization & consolidation

**Overview:**  
Standardizes each source into a shared transaction structure, then merges Stripe + PayPal into one combined revenue stream to be reconciled alongside bank data.

**Nodes involved:**  
- Normalize Stripe Data  
- Normalize PayPal Data  
- Normalize Bank Data  
- Combine Revenue Sources

#### Node: Normalize Stripe Data
- **Type / role:** `Set` — field mapping and type conversion.
- **Configuration (interpreted):**
  - `transactionId` = `$json.id`
  - `amount` = `$json.amount / 100` (Stripe cents → major units)
  - `currency` = `$json.currency`
  - `date` = `$json.created` (likely Unix epoch; stored as string)
  - `source` = `"Stripe"`
  - `description` = `$json.description`
- **Output:** To **Combine Revenue Sources** (input 0).
- **Version notes:** typeVersion `3.4`.
- **Edge cases / failures:**
  - If `amount` is missing/non-numeric → NaN.
  - `created` may be integer seconds; downstream code treats dates inconsistently.
  - **Important:** sets `source` to `"Stripe"` (capital S). Reconciliation code later checks lowercase `'stripe'`.

#### Node: Normalize PayPal Data
- **Type / role:** `Set` — field mapping from PayPal transaction schema.
- **Configuration (interpreted):**
  - `transactionId` = `$json.transaction_info.transaction_id`
  - `amount` = `$json.transaction_info.transaction_amount.value`
  - `currency` = `$json.transaction_info.transaction_amount.currency_code`
  - `date` = `$json.transaction_info.transaction_initiation_date`
  - `source` = `"PayPal"`
  - `description` = `$json.transaction_info.transaction_subject`
- **Output:** To **Combine Revenue Sources** (input 1).
- **Version notes:** typeVersion `3.4`.
- **Edge cases / failures:**
  - `amount` often comes as string; later parseFloat is used (OK).
  - Missing `transaction_subject` common; may yield undefined.
  - `source` capitalization mismatch with later reconciliation checks.

#### Node: Normalize Bank Data
- **Type / role:** `Set` — field mapping from bank statement schema.
- **Configuration (interpreted):**
  - `transactionId` = `$json.transaction_id`
  - `amount` = `$json.amount`
  - `currency` = `$json.currency`
  - `date` = `$json.booking_date`
  - `source` = `"Bank"`
  - `description` = `$json.remittance_information`
- **Output:** Directly to **Reconcile Revenue vs Bank** (in parallel with combined revenue stream).
- **Version notes:** typeVersion `3.4`.
- **Edge cases / failures:**
  - Some banks provide credits/debits with sign conventions; reconciliation may be wrong if deposits are negative or mixed.
  - `source` capitalization mismatch with reconciliation checks.

#### Node: Combine Revenue Sources
- **Type / role:** `Merge` — combines Stripe and PayPal normalized items.
- **Configuration (interpreted):** Default merge behavior (no explicit mode shown). With two inputs, this commonly **passes through** items from both inputs depending on merge mode defaults.
- **Inputs:** Input 0 from Normalize Stripe Data; input 1 from Normalize PayPal Data.
- **Output:** To **Reconcile Revenue vs Bank**.
- **Version notes:** typeVersion `3.2`.
- **Edge cases / failures:**
  - If merge mode is not “Append/Combine”, results may be unexpected (pairing by index rather than concatenation).
  - If one source returns zero items, some merge modes output zero items.

---

### 1.4 Reconciliation & mismatch decisioning

**Overview:**  
Computes total processor revenue vs total bank deposits, calculates variance and mismatch status, then branches based on mismatch detection to optionally flag the output.

**Nodes involved:**  
- Reconcile Revenue vs Bank  
- Check for Mismatches  
- Flag Mismatches

#### Node: Reconcile Revenue vs Bank
- **Type / role:** `Code` — performs reconciliation calculations and returns a single reconciliation object.
- **Configuration (interpreted):**
  - Separates `$input.all()` into:
    - revenueItems when `item.json.source === 'stripe' || 'paypal'`
    - bankItems when `item.json.source === 'bank'`
  - Sums amounts for totals, computes difference and percent.
  - `hasMismatch: Math.abs(difference) > 0.01`
  - Returns one item: `{ json: reconciliation }` with totals, breakdown, and `allTransactions`.
- **Inputs:** From **Combine Revenue Sources** and **Normalize Bank Data** (both feed into this node).
- **Output:** To **Check for Mismatches**.
- **Version notes:** typeVersion `2`.
- **Critical issues / edge cases:**
  - **Case mismatch bug:** normalization sets `source` to `"Stripe"`, `"PayPal"`, `"Bank"`, but code checks `'stripe'/'paypal'/'bank'`. Result: both arrays stay empty → totals 0 → false reconciliation.
  - **Threshold not sourced from config:** hard-coded `0.01` despite `reconciliationThreshold` being defined.
  - If multiple currencies exist, totals are meaningless without FX normalization.
  - Bank statements include fees/chargebacks; comparing gross processor revenue to net bank deposits can produce “expected” variance.
  - Inputs can be large; code node may hit memory/time limits.

#### Node: Check for Mismatches
- **Type / role:** `IF` — branches execution based on mismatch flag.
- **Configuration (interpreted):** Checks boolean equals `true` using expression:
  - Left value: `{{ $('Reconcile Revenue vs Bank').item.json.hasMismatches }}`
- **Outputs:**
  - True branch → **Flag Mismatches**
  - False branch → **Aggregate Monthly Data**
- **Version notes:** typeVersion `2.3`.
- **Critical issues / edge cases:**
  - **Field name bug:** reconciliation code outputs `hasMismatch` (singular) but IF checks `hasMismatches` (plural). This will evaluate as undefined/false and route to “false” branch.
  - Expression uses `$('Reconcile Revenue vs Bank').item` which can be brittle if multiple items exist; here the code returns one item so it’s fine.
  - In strict validation mode, undefined may cause evaluation quirks depending on n8n behavior/version.

#### Node: Flag Mismatches
- **Type / role:** `Set` — annotates output when discrepancies detected.
- **Configuration (interpreted):**
  - Adds `mismatchFlag = "ATTENTION REQUIRED"`
  - Adds `reviewRequired = true`
  - includeOtherFields: true
- **Input:** From IF true branch.
- **Output:** To **Aggregate Monthly Data**.
- **Version notes:** typeVersion `3.4`.
- **Edge cases / failures:**
  - If IF condition never triggers due to the bugs above, this node never runs (no mismatch labeling).

---

### 1.5 Aggregation, reporting, archiving, notification

**Overview:**  
Aggregates the monthly dataset, creates an audit-style report (JSON + HTML), uploads to Google Drive, then emails the tax agent with a summary and Drive link.

**Nodes involved:**  
- Aggregate Monthly Data  
- Generate Audit Report  
- Upload to Archive  
- Notify Tax Agent

#### Node: Aggregate Monthly Data
- **Type / role:** `Aggregate` — aggregates all incoming items into a combined structure.
- **Configuration (interpreted):** `aggregateAllItemData` (collects/aggregates item data).
- **Inputs:** From IF false branch and from **Flag Mismatches**.
- **Output:** To **Generate Audit Report**.
- **Version notes:** typeVersion `1`.
- **Edge cases / failures:**
  - Depending on aggregate behavior, the output schema may not match what later nodes expect (especially the Gmail template referencing `totalRevenue`, `stripeRevenue`, etc.).
  - If only one reconciliation object comes through (from code node), aggregation may be redundant.

#### Node: Generate Audit Report
- **Type / role:** `Code` — builds a detailed report JSON object and an HTML document suitable for PDF conversion.
- **Configuration (interpreted):**
  - Iterates over `$input.all()` and:
    - Builds `transactions[]` when `data.source` exists.
    - Sums totals when `data.source === 'Stripe'/'PayPal'/'Bank'`.
    - Builds `mismatches[]` when `data.mismatch_flag` or `data.reconciliation_status === 'mismatch'`.
  - Creates `report` with metadata, executive summary, breakdown, transactions, mismatches, audit trail.
  - Returns one item containing:
    - `report`
    - `htmlReport`
    - `fileName` ending with `.html`
    - `pdfReady: true`
    - `summary` numeric totals and status
- **Input:** From **Aggregate Monthly Data**.
- **Output:** To **Upload to Archive**.
- **Version notes:** typeVersion `2`.
- **Edge cases / failures:**
  - Potential **schema mismatch**: upstream reconciliation code returns a single object without `source` fields at top level (it nests transactions in `allTransactions`). This code expects per-transaction items with `source` directly.
  - The mismatches detection keys (`mismatch_flag`) do not match `Flag Mismatches` output (`mismatchFlag` camelCase).
  - Date parsing: `new Date(t.date)` can break if `t.date` is Unix seconds string, not ISO.
  - Report labels tolerance as `'$1.00'` but reconciliation uses `0.01` (and config has `0.01`).

#### Node: Upload to Archive
- **Type / role:** `Google Drive` — uploads report artifact to a Drive folder.
- **Configuration (interpreted):**
  - File name: `Revenue_Report_YYYY-MM.pdf` (previous month)
  - Drive: “My Drive”
  - Folder ID: from config `archiveFolderId`
  - **Important:** No binary/content mapping is configured; workflow produces HTML, but node name implies uploading a PDF.
- **Credentials:** Google Drive OAuth2 (`googleDriveOAuth2Api Credential`)
- **Input:** From **Generate Audit Report**.
- **Output:** To **Notify Tax Agent**.
- **Version notes:** typeVersion `3`.
- **Edge cases / failures:**
  - **Content upload missing:** Drive node typically needs binary data or a “File Content” field; as-is, it may create an empty file or fail depending on configuration defaults.
  - File extension mismatch: `.pdf` name but HTML content exists; may confuse recipients/auditors.
  - Folder permission issues; invalid folderId; OAuth scope limitations.

#### Node: Notify Tax Agent
- **Type / role:** `Gmail` — sends email with summary and Drive link.
- **Configuration (interpreted):**
  - To: `taxAgentEmail` from config
  - Subject: `Monthly Revenue Report Ready - <previous month>`
  - HTML message includes summary fields and conditional mismatch section.
  - Links to `$('Upload to Archive').first().json.webViewLink`
- **Credentials:** Gmail OAuth2 (`gmailOAuth2 Credential`)
- **Input:** From **Upload to Archive**.
- **Version notes:** typeVersion `2.2`.
- **Edge cases / failures:**
  - Template references `$('Aggregate Monthly Data').first().json.totalRevenue` and similar fields that may not exist.
  - Conditional block uses `$('Flag Mismatches').all().length`—if the node never ran, `.all()` returns empty; OK, but logic may mask real mismatches due to upstream bugs.
  - Gmail OAuth: token expiry, restricted scopes, sending limits.
  - `webViewLink` depends on Drive node returning it; not guaranteed if upload fails.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Monthly Schedule | Schedule Trigger | Monthly time-based entrypoint | — | Workflow Configuration | ## Fetch Multi-Source Data<br>What: Retrieves revenue records from Stripe, PayPal, and bank statements in parallel.<br>Why: Captures complete revenue picture across all payment and settlement channels. |
| Workflow Configuration | Set | Centralized runtime parameters | Monthly Schedule | Get Stripe Revenue; Get PayPal Revenue; Get Bank Statements | ## Fetch Multi-Source Data<br>What: Retrieves revenue records from Stripe, PayPal, and bank statements in parallel.<br>Why: Captures complete revenue picture across all payment and settlement channels. |
| Get Stripe Revenue | HTTP Request | Fetch Stripe transactions for prior month | Workflow Configuration | Normalize Stripe Data | ## Fetch Multi-Source Data<br>What: Retrieves revenue records from Stripe, PayPal, and bank statements in parallel.<br>Why: Captures complete revenue picture across all payment and settlement channels. |
| Normalize Stripe Data | Set | Normalize Stripe fields to common schema | Get Stripe Revenue | Combine Revenue Sources | ## Fetch Multi-Source Data<br>What: Retrieves revenue records from Stripe, PayPal, and bank statements in parallel.<br>Why: Captures complete revenue picture across all payment and settlement channels. |
| Get PayPal Revenue | HTTP Request | Fetch PayPal transactions for prior month | Workflow Configuration | Normalize PayPal Data | ## Fetch Multi-Source Data<br>What: Retrieves revenue records from Stripe, PayPal, and bank statements in parallel.<br>Why: Captures complete revenue picture across all payment and settlement channels. |
| Normalize PayPal Data | Set | Normalize PayPal fields to common schema | Get PayPal Revenue | Combine Revenue Sources | ## Fetch Multi-Source Data<br>What: Retrieves revenue records from Stripe, PayPal, and bank statements in parallel.<br>Why: Captures complete revenue picture across all payment and settlement channels. |
| Combine Revenue Sources | Merge | Consolidate Stripe + PayPal normalized items | Normalize Stripe Data; Normalize PayPal Data | Reconcile Revenue vs Bank | ## Reconcile & Flag Mismatches<br>What: Compares aggregated revenue against bank statement totals and identifies discrepancies.<br>Why: Detects timing differences, missing transactions, and potential fraud |
| Get Bank Statements | HTTP Request | Fetch bank statement transactions for prior month | Workflow Configuration | Normalize Bank Data | ## Fetch Multi-Source Data<br>What: Retrieves revenue records from Stripe, PayPal, and bank statements in parallel.<br>Why: Captures complete revenue picture across all payment and settlement channels. |
| Normalize Bank Data | Set | Normalize bank fields to common schema | Get Bank Statements | Reconcile Revenue vs Bank | ## Fetch Multi-Source Data<br>What: Retrieves revenue records from Stripe, PayPal, and bank statements in parallel.<br>Why: Captures complete revenue picture across all payment and settlement channels. |
| Reconcile Revenue vs Bank | Code | Compute totals, variance, mismatch boolean | Combine Revenue Sources; Normalize Bank Data | Check for Mismatches | ## Reconcile & Flag Mismatches<br>What: Compares aggregated revenue against bank statement totals and identifies discrepancies.<br>Why: Detects timing differences, missing transactions, and potential fraud |
| Check for Mismatches | IF | Branch based on mismatch status | Reconcile Revenue vs Bank | Flag Mismatches (true); Aggregate Monthly Data (false) | ## Reconcile & Flag Mismatches<br>What: Compares aggregated revenue against bank statement totals and identifies discrepancies.<br>Why: Detects timing differences, missing transactions, and potential fraud |
| Flag Mismatches | Set | Annotate run output when mismatch detected | Check for Mismatches (true) | Aggregate Monthly Data | ## Reconcile & Flag Mismatches<br>What: Compares aggregated revenue against bank statement totals and identifies discrepancies.<br>Why: Detects timing differences, missing transactions, and potential fraud |
| Aggregate Monthly Data | Aggregate | Aggregate items into consolidated monthly dataset | Check for Mismatches (false); Flag Mismatches | Generate Audit Report | ## Generate Audit Report & Notification<br>What: Compiles flagged mismatches, monthly aggregates into audit documentation.<br>Why: Provides compliance evidence, supports tax filing |
| Generate Audit Report | Code | Build audit report JSON + HTML | Aggregate Monthly Data | Upload to Archive | ## Generate Audit Report & Notification<br>What: Compiles flagged mismatches, monthly aggregates into audit documentation.<br>Why: Provides compliance evidence, supports tax filing |
| Upload to Archive | Google Drive | Upload archival report to Drive | Generate Audit Report | Notify Tax Agent | ## Generate Audit Report & Notification<br>What: Compiles flagged mismatches, monthly aggregates into audit documentation.<br>Why: Provides compliance evidence, supports tax filing |
| Notify Tax Agent | Gmail | Email tax agent with summary + Drive link | Upload to Archive | — | ## Generate Audit Report & Notification<br>What: Compiles flagged mismatches, monthly aggregates into audit documentation.<br>Why: Provides compliance evidence, supports tax filing |
| Sticky Note | Sticky Note | Documentation | — | — | ## Prerequisites<br>Stripe, PayPal, and bank statement accounts; API credentials for each source<br>## Use Cases<br>Accounting firms automating client revenue verification; multi-channel e-commerce businesses<br>## Customization<br>Add additional payment sources (Square, Shopify), adjust normalization rules for regional formats<br>## Benefits<br>Eliminates manual reconciliation, detects discrepancies automatically |
| Sticky Note1 | Sticky Note | Documentation | — | — | ## Setup Steps<br>1. Configure Stripe, PayPal.<br>2. Set up normalization rules for date, currency, and transaction ID mappings.<br>3. Connect Google Drive for report archiving and Gmail for agent notifications.<br>4. Define mismatch thresholds and reconciliation tolerance parameters. |
| Sticky Note2 | Sticky Note | Documentation | — | — | ## How It Works<br>This workflow automates monthly revenue reconciliation across Stripe, PayPal, and bank statements by standardizing data formats, detecting discrepancies, and producing audit-ready reports. It concurrently retrieves revenue data from multiple sources, normalizes datasets into consistent structures, consolidates records, and reconciles transactions against bank statements with intelligent mismatch detection. The system aggregates monthly totals, generates detailed audit reports with clearly flagged discrepancies, archives finalized outputs to Google Drive, and notifies tax agents. Designed for accounting firms, finance teams, and businesses, it enables automated revenue verification, multi-channel reconciliation, discrepancy identification, and compliance audit documentation without manual record matching or error-prone spreadsheet workflows. |
| Sticky Note4 | Sticky Note | Documentation | — | — | ## Generate Audit Report & Notification<br>What: Compiles flagged mismatches, monthly aggregates into audit documentation.<br>Why: Provides compliance evidence, supports tax filing |
| Sticky Note5 | Sticky Note | Documentation | — | — | ## Reconcile & Flag Mismatches<br>What: Compares aggregated revenue against bank statement totals and identifies discrepancies.<br>Why: Detects timing differences, missing transactions, and potential fraud |
| Sticky Note6 | Sticky Note | Documentation | — | — | ## Fetch Multi-Source Data<br>What: Retrieves revenue records from Stripe, PayPal, and bank statements in parallel.<br>Why: Captures complete revenue picture across all payment and settlement channels. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create workflow**
- Name it: `Automated Revenue Reconciliation & Tax Evidence Archive`
- Keep it inactive until credentials and endpoints are verified.

2) **Add trigger**
- Add node: **Schedule Trigger** named `Monthly Schedule`
- Configure: interval **Months**, trigger at **02:00**.

3) **Add configuration Set node**
- Add node: **Set** named `Workflow Configuration`
- Add fields:
  - `stripeApiUrl` (string) – your Stripe endpoint (e.g., charges, balance transactions, or payouts depending on your methodology)
  - `paypalApiUrl` (string) – PayPal transactions/reporting endpoint
  - `bankApiUrl` (string) – bank statement endpoint
  - `archiveFolderId` (string) – Google Drive folder ID
  - `taxAgentEmail` (string) – recipient
  - `reconciliationThreshold` (number) – tolerance (e.g., `0.01` or `1.00`)
- Enable **Include Other Fields**.
- Connect: `Monthly Schedule` → `Workflow Configuration`.

4) **Add Stripe HTTP retrieval**
- Add node: **HTTP Request** named `Get Stripe Revenue`
- URL: `={{ $('Workflow Configuration').first().json.stripeApiUrl }}`
- Authentication: **Predefined Credential Type** (configure your Stripe credential in n8n)
- Query params:
  - `created[gte]` = `={{ $now.minus({ months: 1 }).startOf('month').toUnixInteger() }}`
  - `created[lt]` = `={{ $now.startOf('month').toUnixInteger() }}`
- Connect: `Workflow Configuration` → `Get Stripe Revenue`.

5) **Add PayPal HTTP retrieval**
- Add node: **HTTP Request** named `Get PayPal Revenue`
- URL: `={{ $('Workflow Configuration').first().json.paypalApiUrl }}`
- Authentication: Predefined Credential Type (PayPal)
- Query params:
  - `start_date` = `={{ $now.minus({ months: 1 }).startOf('month').toISO() }}`
  - `end_date` = `={{ $now.startOf('month').toISO() }}`
- Connect: `Workflow Configuration` → `Get PayPal Revenue`.

6) **Add Bank HTTP retrieval**
- Add node: **HTTP Request** named `Get Bank Statements`
- URL: `={{ $('Workflow Configuration').first().json.bankApiUrl }}`
- Authentication: Predefined Credential Type (Bank)
- Query params:
  - `from_date` = `={{ $now.minus({ months: 1 }).startOf('month').toISODate() }}`
  - `to_date` = `={{ $now.startOf('month').toISODate() }}`
- Connect: `Workflow Configuration` → `Get Bank Statements`.

7) **Add normalization Set nodes**
- Add **Set** named `Normalize Stripe Data` mapping:
  - `transactionId` = `{{$json.id}}`
  - `amount` = `{{$json.amount / 100}}`
  - `currency` = `{{$json.currency}}`
  - `date` = `{{$json.created}}`
  - `source` = `Stripe`
  - `description` = `{{$json.description}}`
- Connect: `Get Stripe Revenue` → `Normalize Stripe Data`.

- Add **Set** named `Normalize PayPal Data` mapping:
  - `transactionId` = `{{$json.transaction_info.transaction_id}}`
  - `amount` = `{{$json.transaction_info.transaction_amount.value}}`
  - `currency` = `{{$json.transaction_info.transaction_amount.currency_code}}`
  - `date` = `{{$json.transaction_info.transaction_initiation_date}}`
  - `source` = `PayPal`
  - `description` = `{{$json.transaction_info.transaction_subject}}`
- Connect: `Get PayPal Revenue` → `Normalize PayPal Data`.

- Add **Set** named `Normalize Bank Data` mapping:
  - `transactionId` = `{{$json.transaction_id}}`
  - `amount` = `{{$json.amount}}`
  - `currency` = `{{$json.currency}}`
  - `date` = `{{$json.booking_date}}`
  - `source` = `Bank`
  - `description` = `{{$json.remittance_information}}`
- Connect: `Get Bank Statements` → `Normalize Bank Data`.

8) **Merge Stripe + PayPal**
- Add node: **Merge** named `Combine Revenue Sources`
- Set merge mode to **Append** (or equivalent “combine both input item lists”) to avoid index-pairing surprises.
- Connect:
  - `Normalize Stripe Data` → `Combine Revenue Sources` (input 0)
  - `Normalize PayPal Data` → `Combine Revenue Sources` (input 1)

9) **Reconcile totals**
- Add node: **Code** named `Reconcile Revenue vs Bank`
- Paste reconciliation JS logic (adapt as needed).
- Connect:
  - `Combine Revenue Sources` → `Reconcile Revenue vs Bank`
  - `Normalize Bank Data` → `Reconcile Revenue vs Bank`

10) **Decision: mismatch?**
- Add node: **IF** named `Check for Mismatches`
- Configure condition to check the boolean mismatch field produced by the code node.
- Connect: `Reconcile Revenue vs Bank` → `Check for Mismatches`.

11) **Optional flagging**
- Add node: **Set** named `Flag Mismatches`
- Add fields:
  - `mismatchFlag` = `ATTENTION REQUIRED`
  - `reviewRequired` = `true`
- Connect: `Check for Mismatches` (true) → `Flag Mismatches`.

12) **Aggregate**
- Add node: **Aggregate** named `Aggregate Monthly Data`
- Mode: **Aggregate All Item Data**
- Connect:
  - `Check for Mismatches` (false) → `Aggregate Monthly Data`
  - `Flag Mismatches` → `Aggregate Monthly Data`

13) **Generate report**
- Add node: **Code** named `Generate Audit Report`
- Paste report generation JS (adapt to actual upstream schema).
- Connect: `Aggregate Monthly Data` → `Generate Audit Report`.

14) **Archive to Google Drive**
- Add node: **Google Drive** named `Upload to Archive`
- Configure Google Drive OAuth2 credentials.
- Folder ID: `={{ $('Workflow Configuration').first().json.archiveFolderId }}`
- File name: `={{ 'Revenue_Report_' + $now.minus({ months: 1 }).toFormat('yyyy-MM') + '.pdf' }}`
- Ensure the node is configured to upload **actual content** (binary) from the previous node (you will likely need an extra step that converts `htmlReport` to a PDF binary or uploads HTML with `.html` extension).
- Connect: `Generate Audit Report` → `Upload to Archive`.

15) **Email tax agent**
- Add node: **Gmail** named `Notify Tax Agent`
- Configure Gmail OAuth2 credentials.
- To: `={{ $('Workflow Configuration').first().json.taxAgentEmail }}`
- Subject: `={{ 'Monthly Revenue Report Ready - ' + $now.minus({ months: 1 }).toFormat('MMMM yyyy') }}`
- Message: HTML body referencing summary fields and Drive `webViewLink`.
- Connect: `Upload to Archive` → `Notify Tax Agent`.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Prerequisites: Stripe, PayPal, and bank statement accounts; API credentials for each source | Workflow sticky note “Prerequisites” |
| Use cases: Accounting firms automating client revenue verification; multi-channel e-commerce businesses | Workflow sticky note “Use Cases” |
| Customization: Add additional payment sources (Square, Shopify), adjust normalization rules for regional formats | Workflow sticky note “Customization” |
| Benefits: Eliminates manual reconciliation, detects discrepancies automatically | Workflow sticky note “Benefits” |
| Setup steps: configure sources, normalization rules, Google Drive + Gmail, define mismatch thresholds | Workflow sticky note “Setup Steps” |
| How it works: parallel fetch → normalize → consolidate → reconcile → report → archive → notify | Workflow sticky note “How It Works” |

### Notable implementation gaps to address before production use (high impact)
- **Fix source casing consistency** (either set `source` to lowercase in normalization, or update the reconciliation code to match `"Stripe"/"PayPal"/"Bank"`).
- **Fix mismatch field naming** (`hasMismatch` vs `hasMismatches`).
- **Implement pagination** for Stripe/PayPal/bank APIs if needed.
- **Align report generation with actual upstream schema** (reconciliation currently outputs one summary object; report code expects many per-transaction items).
- **Implement HTML→PDF conversion** (or upload HTML as `.html`) so the Drive upload and `.pdf` naming are consistent.
- **Use `reconciliationThreshold` from configuration** rather than hard-coding.