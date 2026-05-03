Detect multi-source transaction fraud and reconcile finances with OpenAI, Nvidia NIM, Gmail, Slack and Google Sheets

https://n8nworkflows.xyz/workflows/detect-multi-source-transaction-fraud-and-reconcile-finances-with-openai--nvidia-nim--gmail--slack-and-google-sheets-12732


# Detect multi-source transaction fraud and reconcile finances with OpenAI, Nvidia NIM, Gmail, Slack and Google Sheets

## 1. Workflow Overview

**Workflow name:** Multi-Source Transaction Fraud Detection and Financial Reconciliation Automation  
**Stated title:** Detect multi-source transaction fraud and reconcile finances with OpenAI, Nvidia NIM, Gmail, Slack and Google Sheets

**Purpose:**  
Automate ingestion of transactions from multiple sources (scheduled polling + real-time webhook), normalize/validate them, run AI-based fraud risk scoring (OpenAI via LangChain Agent with structured output), route transactions by risk level, trigger alerts for high-risk cases, reconcile acceptable transactions with an ERP, persist records to Postgres, and generate an aggregated Slack financial report.

**Important note vs. stated title/sticky notes:**  
- The provided workflow JSON **does not include any Nvidia NIM node** and **does not include Google Sheets nodes**. It uses **OpenAI**, **Slack**, **Gmail**, **Postgres**, and **HTTP Request** integrations.

### 1.1 Transaction Ingestion (Polling + Webhook)
Two entry points feed transaction events: a schedule trigger (poll APIs every 15 minutes) and a webhook (real-time POST). Polled data is fetched from bank and payment gateway APIs.

### 1.2 Data Standardization & Quality Gates
Merged items are normalized into a consistent schema, filtered for required fields, then deduplicated (idempotency) using workflow static data.

### 1.3 AI Fraud Risk Scoring (OpenAI + Structured Output)
A LangChain Agent sends transaction JSON to an OpenAI chat model and parses results into a strict schema (riskScore, riskLevel, indicators, etc.).

### 1.4 Risk Routing & Actions
A switch routes by `riskLevel`:
- **High**: flag for manual review + Slack + Gmail alert  
- **Medium/Low**: reconcile with ERP and persist to Postgres

*(As implemented, Medium and Low follow the same reconcile path.)*

### 1.5 Persistence, Audit, and Reporting
Reconciled transactions are stored in Postgres, an audit trail row is written, then all processed items are aggregated into a single financial summary posted to Slack.

---

## 2. Block-by-Block Analysis

### Block A — Transaction Ingestion
**Overview:** Ingests transactions from two paths: scheduled polling of APIs and a webhook for real-time events, then converges all events into a single stream.

**Nodes involved:**
- Schedule Trigger - Periodic Ingestion
- Workflow Configuration
- Fetch Bank API Transactions
- Fetch Payment Gateway Transactions
- Webhook - Real-time Transaction Events
- Merge All Transaction Sources

#### 1) Schedule Trigger - Periodic Ingestion
- **Type / role:** Schedule Trigger; periodically starts the polling branch.
- **Config:** Runs every **15 minutes**.
- **Outputs:** Connects to **Workflow Configuration**.
- **Edge cases / failures:** None typical beyond n8n scheduling/execution limits.

#### 2) Workflow Configuration
- **Type / role:** Set; centralizes runtime parameters and placeholders.
- **Config choices (interpreted):**
  - Defines placeholder URLs: `bankApiUrl`, `paymentGatewayApiUrl`, `erpApiUrl`
  - Sets thresholds: `fraudThresholdHigh = 0.8`, `fraudThresholdMedium = 0.5` *(not used later for logic; the AI sets riskLevel)*
  - Notification targets: `slackChannel`, `financeTeamEmail`
  - `includeOtherFields: true` so incoming fields are preserved.
- **Expressions/variables:** Downstream nodes reference e.g. `$('Workflow Configuration').first().json.bankApiUrl`.
- **Connections:** Outputs to **Fetch Bank API Transactions** and **Fetch Payment Gateway Transactions**.
- **Failure modes:** Misconfigured placeholders → HTTP nodes fail (invalid URL).

#### 3) Fetch Bank API Transactions
- **Type / role:** HTTP Request; pulls bank transactions within a time window.
- **Config:**
  - URL from `bankApiUrl`
  - Query params:
    - `from_date = $now.minus({ hours: 1 }).toISO()`
    - `to_date = $now.toISO()`
  - Timeout: 30s
- **Connections:** Output to **Merge All Transaction Sources** (input index 1).
- **Failure modes:** 401/403 auth, 404 endpoint, 5xx, timeout, non-JSON response.
- **Implementation note:** No authentication configured in the node; expected to be added (headers/OAuth2/API key) depending on bank API.

#### 4) Fetch Payment Gateway Transactions
- **Type / role:** HTTP Request; pulls gateway transactions within a time window.
- **Config:** Same windowing + timeout as bank fetch; URL from `paymentGatewayApiUrl`.
- **Connections:** Output to **Merge All Transaction Sources** (input index 2).
- **Failure modes:** Same as bank fetch.

#### 5) Webhook - Real-time Transaction Events
- **Type / role:** Webhook; receives real-time transactions via HTTP POST.
- **Config:**
  - Path: `transaction-webhook`
  - Method: POST
  - Response: `lastNode` (responds with output of the last executed node in this path)
- **Connections:** Output to **Merge All Transaction Sources** (input index 0).
- **Failure modes / edge cases:**
  - Caller sends unexpected schema → normalization/validation might drop it later.
  - If downstream errors occur, webhook response may be error/timeout to caller.

#### 6) Merge All Transaction Sources
- **Type / role:** Merge; combines 3 input streams into one.
- **Config:** `numberInputs: 3` (expects three incoming connections).
- **Connections:** Output to **Normalize Transaction Data**.
- **Edge cases:**
  - If one of the scheduled fetch nodes returns an array wrapped in a single item vs. individual items, you may end up normalizing an unexpected structure. Ensure HTTP responses are itemized as desired.

**Sticky note coverage (context):**
- “## Transaction Ingestion …” applies to this block’s ingestion nodes.

---

### Block B — Validation, Normalization & Idempotency
**Overview:** Standardizes fields into a consistent transaction schema, filters out malformed records, and prevents reprocessing duplicates across executions.

**Nodes involved:**
- Normalize Transaction Data
- Filter Valid Transactions
- Check Idempotency

#### 1) Normalize Transaction Data
- **Type / role:** Set; maps multiple possible field names into a canonical schema.
- **Key mappings (expressions):**
  - `transactionId = $json.id || $json.transaction_id || $json.txn_id`
  - `amount = parseFloat($json.amount || $json.value || 0)`
  - `currency = $json.currency || 'USD'`
  - `timestamp = $json.timestamp || $json.created_at || $now.toISO()`
  - `customerId = $json.customer_id || $json.user_id || 'unknown'`
  - `merchantId = $json.merchant_id || $json.vendor_id || 'unknown'`
  - `paymentMethod = $json.payment_method || $json.method || 'unknown'`
  - `source = $json.source || 'api'`
- **Connections:** Input from **Merge All Transaction Sources**; output to **Filter Valid Transactions**.
- **Edge cases:**
  - `parseFloat` on non-numeric strings → `NaN`; the filter later (`gt 0`) will fail and drop the item.
  - Defaulting `customerId` to `'unknown'` conflicts with the filter requiring non-empty; it *will pass* because `'unknown'` is non-empty (may be undesirable).

#### 2) Filter Valid Transactions
- **Type / role:** Filter; enforces basic integrity rules.
- **Conditions (AND):**
  - `transactionId` is not empty
  - `amount` > 0
  - `customerId` is not empty
- **Connections:** Output to **Check Idempotency**.
- **Failure/edge cases:**
  - If upstream data is nested (e.g., `{data:[...]}`), this filter will not see individual transactions.

#### 3) Check Idempotency
- **Type / role:** Code (JavaScript); deduplicates by transactionId using workflow static global data.
- **Logic summary:**
  - Reads `processedIds` from `$getWorkflowStaticData('global')`
  - Treats it like a Set; drops items whose `transactionId` was already seen
  - Stores updated set back to static data
- **Connections:** Output to **AI Agent - Fraud Detection**.
- **Important technical caveat:**
  - n8n static data is JSON-serialized. A native `Set` is typically **not reliably serializable**; it may be persisted as `{}` or lose contents between runs. A safer approach is storing an **array** (or object map) and rebuilding a Set each run.
- **Edge cases:**
  - Memory growth over time (ever-growing processedIds) without TTL/cleanup.
  - Transaction IDs reused across sources could unintentionally drop legitimate distinct records.

**Sticky note coverage (context):**
- “## Validation & Normalization …” applies to Normalize + Filter + Idempotency.

---

### Block C — AI Fraud Risk Scoring (OpenAI + Structured Output)
**Overview:** Sends each transaction to a LangChain Agent backed by OpenAI, returning a strictly structured fraud assessment that downstream routing can rely on.

**Nodes involved:**
- AI Agent - Fraud Detection
- OpenAI Chat Model
- Structured Output Parser - Fraud Analysis

#### 1) AI Agent - Fraud Detection
- **Type / role:** LangChain Agent; orchestrates prompting and tool/model usage.
- **Config choices:**
  - Input text: `{{ $json }}` (stringification of the transaction object as provided by n8n templating)
  - System message: detailed fraud/compliance criteria
  - Requires structured output (`hasOutputParser: true`)
- **Connections:**
  - Main input from **Check Idempotency**
  - Main output to **Route by Risk Level**
  - AI language model input from **OpenAI Chat Model**
  - Output parser from **Structured Output Parser - Fraud Analysis**
- **Failure modes:**
  - Model refusal/low-quality output; schema parser failure if output is not compliant.
  - Token limits if `$json` contains large nested payloads.

#### 2) OpenAI Chat Model
- **Type / role:** LangChain OpenAI chat model provider.
- **Config:** Model `gpt-4o`.
- **Credentials:** “OpenAi account” (OpenAI API key in n8n credentials).
- **Connections:** Feeds the agent as its `ai_languageModel`.
- **Failure modes:** Invalid API key, quota exceeded, timeouts, model deprecation.

#### 3) Structured Output Parser - Fraud Analysis
- **Type / role:** Structured output parser; enforces JSON schema.
- **Schema highlights:**
  - `riskScore` number
  - `riskLevel` enum: `high | medium | low`
  - Arrays: `fraudIndicators`, `complianceFlags`
  - Boolean: `anomalyDetected`
  - String: `recommendation`
  - All required
- **Connections:** Feeds the agent as `ai_outputParser`.
- **Failure modes:** Parser errors if the model returns malformed JSON or missing keys.

**Sticky note coverage (context):**
- “## AI Risk Scoring …” applies to these AI nodes.

---

### Block D — Risk Routing & Alerts / Reconciliation
**Overview:** Routes transactions by AI-produced risk level. High risk triggers manual review markers and notifications; medium/low are reconciled with ERP.

**Nodes involved:**
- Route by Risk Level
- Flag for Manual Review
- Notify Finance Team - High Risk
- Send Customer Alert Email
- Reconcile with ERP System

#### 1) Route by Risk Level
- **Type / role:** Switch; branches by `riskLevel`.
- **Rules:**
  - Output “High Risk” if `riskLevel == "high"`
  - Output “Medium Risk” if `riskLevel == "medium"`
  - Output “Low Risk” if `riskLevel == "low"`
  - Fallback output: `extra`
- **Connections (as implemented):**
  - Output 0 (High) → **Flag for Manual Review**
  - Output 1 (Medium) → **Reconcile with ERP System**
  - Output 2 (Low) → **Reconcile with ERP System**
- **Edge cases:**
  - If AI returns unexpected casing/spaces, it may go to fallback `extra` (not connected), effectively dropping items.

#### 2) Flag for Manual Review
- **Type / role:** Set; annotates transaction as requiring review.
- **Fields added:**
  - `requiresManualReview = true`
  - `reviewReason = $json.recommendation`
  - `flaggedAt = $now.toISO()`
- **Connections:** Output to both **Notify Finance Team - High Risk** and **Send Customer Alert Email**.
- **Edge cases:** None major; depends on downstream credentials.

#### 3) Notify Finance Team - High Risk
- **Type / role:** Slack; posts high-risk alert message.
- **Config:**
  - Channel selected by ID from `Workflow Configuration.slackChannel`
  - Uses OAuth2 Slack credential
  - Message includes `fraudIndicators.join(', ')` (assumes array)
- **Failure modes:**
  - Slack OAuth token revoked/expired, channel ID invalid, missing permissions.
  - If `fraudIndicators` is not an array, `.join` will throw expression error.

#### 4) Send Customer Alert Email
- **Type / role:** Gmail; sends an email alert.
- **Config (as implemented):**
  - `sendTo` uses `financeTeamEmail` from configuration (despite node name implying customer email)
  - Subject: “Suspicious Transaction Alert - Action Required”
- **Credentials:** Gmail OAuth2.
- **Failure modes:** Gmail OAuth expired, sending limits, invalid recipient.
- **Design mismatch:** Node name says “Customer Alert”, but it sends to finance team email; adapt to `customerEmail` if available.

#### 5) Reconcile with ERP System
- **Type / role:** HTTP Request; sends transaction to ERP for reconciliation.
- **Config:**
  - POST to `erpApiUrl`
  - JSON body includes `transactionId, amount, currency, customerId, timestamp, riskScore` and `status: "reconciled"`
  - Timeout: 30s
- **Connections:** Output to **Store Transaction Record**.
- **Failure modes:**
  - ERP auth missing/invalid (none configured here), validation errors, idempotency conflicts on ERP side.
  - JSON body uses unquoted template interpolations; if `transactionId` is a string, the ERP may receive invalid JSON unless n8n correctly injects it with quotes. Safer is to build body via structured fields rather than raw JSON string templating.

**Sticky note coverage (context):**
- “## Alert Routing …” applies to Route/Sets/Slack/Gmail in this block.

---

### Block E — Persistence, Audit Trail, and Reporting
**Overview:** Stores processed transactions and audit events in Postgres, aggregates all processed items into a summary report, and posts the report to Slack.

**Nodes involved:**
- Store Transaction Record
- Store Audit Trail
- Aggregate for Financial Report
- Generate Financial Report
- Send Financial Report to Slack

#### 1) Store Transaction Record
- **Type / role:** Postgres; inserts a transaction record into `public.transactions`.
- **Mapped columns (key fields):**
  - `transaction_id`, `amount`, `currency`, `timestamp`
  - `customer_id`, `merchant_id`, `payment_method`
  - `risk_level`, `risk_score`
  - `fraud_indicators = JSON.stringify($json.fraudIndicators)`
  - `compliance_flags = JSON.stringify($json.complianceFlags)`
  - `status = "processed"`, `created_at = $now.toISO()`
- **Connections:** Output to **Store Audit Trail**.
- **Failure modes:**
  - Missing DB credentials (not shown in JSON), table/schema mismatch, constraint violations (unique txn id), type mismatches.
  - If fraudIndicators/complianceFlags are undefined, `JSON.stringify(undefined)` becomes undefined (may violate NOT NULL).

#### 2) Store Audit Trail
- **Type / role:** Postgres; inserts into `public.audit_trail`.
- **Mapped columns:**
  - `user = "system"`
  - `action = "transaction_processed"`
  - `details = JSON.stringify({ riskScore, riskLevel, source })`
  - `timestamp = $now.toISO()`
  - `transaction_id = $json.transactionId`
- **Connections:** Output to **Aggregate for Financial Report**.
- **Failure modes:** Same Postgres concerns; table columns must exist.

#### 3) Aggregate for Financial Report
- **Type / role:** Aggregate; combines all incoming items into one item containing all item data.
- **Config:** “aggregateAllItemData” (collects all items under `.json.data`).
- **Connections:** Output to **Generate Financial Report**.
- **Edge cases:** Large batches may create large payloads and memory pressure.

#### 4) Generate Financial Report
- **Type / role:** Code; computes totals and produces HTML-ish summary string.
- **Logic:**
  - Reads `const transactions = $input.first().json.data;`
  - Computes `totalCount`, `totalAmount`, `avgRiskScore`, and counts by risk level
  - Produces `report` HTML string and numeric summary fields
- **Connections:** Output to **Send Financial Report to Slack**.
- **Edge cases / bugs:**
  - If `totalCount` is 0, division by zero → `avgRiskScore` becomes `Infinity`/`NaN`.
  - Assumes each transaction object has numeric `amount` and `riskScore`.

#### 5) Send Financial Report to Slack
- **Type / role:** Slack; posts the summary metrics to configured channel.
- **Config:** Uses `slackChannel` from Workflow Configuration; OAuth2 Slack credential.
- **Failure modes:** Slack auth/permissions; formatting errors if numbers are undefined.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger - Periodic Ingestion | Schedule Trigger | Starts periodic polling ingestion | — | Workflow Configuration | ## Transaction Ingestion  \nFetches real-time payment data from banking APIs, Stripe, PayPal using webhooks and scheduled polling.\n**Why** - Captures transactions immediately upon processing, enabling real-time fraud detection before settlement occurs. |
| Webhook - Real-time Transaction Events | Webhook | Receives real-time transaction events | — | Merge All Transaction Sources | ## Transaction Ingestion  \nFetches real-time payment data from banking APIs, Stripe, PayPal using webhooks and scheduled polling.\n**Why** - Captures transactions immediately upon processing, enabling real-time fraud detection before settlement occurs. |
| Workflow Configuration | Set | Central config (URLs, thresholds, notification targets) | Schedule Trigger - Periodic Ingestion | Fetch Bank API Transactions; Fetch Payment Gateway Transactions | ## Setup Steps\n1. Configure banking API credentials for transaction access\n2. Set up webhook endpoints for real-time transaction notifications\n3. Add OpenAI API key for fraud pattern analysis and risk scoring\n4. Configure NVIDIA NIM API for advanced anomaly detection models\n5. Set Gmail OAuth credentials for automated fraud alert delivery\n6. Connect Slack workspace and specify alert channels for urgent notifications\n7. Link Google Sheets for transaction logging and compliance audit trails |
| Fetch Bank API Transactions | HTTP Request | Poll bank transactions | Workflow Configuration | Merge All Transaction Sources | ## Transaction Ingestion  \nFetches real-time payment data from banking APIs, Stripe, PayPal using webhooks and scheduled polling.\n**Why** - Captures transactions immediately upon processing, enabling real-time fraud detection before settlement occurs. |
| Fetch Payment Gateway Transactions | HTTP Request | Poll gateway transactions | Workflow Configuration | Merge All Transaction Sources | ## Transaction Ingestion  \nFetches real-time payment data from banking APIs, Stripe, PayPal using webhooks and scheduled polling.\n**Why** - Captures transactions immediately upon processing, enabling real-time fraud detection before settlement occurs. |
| Merge All Transaction Sources | Merge | Unifies webhook + polled streams | Webhook - Real-time Transaction Events; Fetch Bank API Transactions; Fetch Payment Gateway Transactions | Normalize Transaction Data | ## Transaction Ingestion  \nFetches real-time payment data from banking APIs, Stripe, PayPal using webhooks and scheduled polling.\n**Why** - Captures transactions immediately upon processing, enabling real-time fraud detection before settlement occurs. |
| Normalize Transaction Data | Set | Canonicalize fields (id/amount/currency/…) | Merge All Transaction Sources | Filter Valid Transactions | ## Validation & Normalization\nStandardizes transaction formats, validates required fields, and filters invalid entries.\n**Why** - Ensures data quality and prevents downstream processing errors from malformed transaction records. |
| Filter Valid Transactions | Filter | Drop malformed/invalid transactions | Normalize Transaction Data | Check Idempotency | ## Validation & Normalization\nStandardizes transaction formats, validates required fields, and filters invalid entries.\n**Why** - Ensures data quality and prevents downstream processing errors from malformed transaction records. |
| Check Idempotency | Code | Prevent reprocessing duplicate transactionIds | Filter Valid Transactions | AI Agent - Fraud Detection | ## Validation & Normalization\nStandardizes transaction formats, validates required fields, and filters invalid entries.\n**Why** - Ensures data quality and prevents downstream processing errors from malformed transaction records. |
| AI Agent - Fraud Detection | LangChain Agent | AI fraud/compliance assessment + structured output | Check Idempotency | Route by Risk Level | ## AI Risk Scoring\nRoutes transaction metadata to AI models that calculate fraud probability based on behavioral patterns.\n**Why** - Identifies suspicious activities that rule-based systems miss, adapting to evolving fraud tactics dynamically. |
| OpenAI Chat Model | OpenAI Chat Model (LangChain) | LLM backend for agent | — (AI connection) | AI Agent - Fraud Detection (ai_languageModel) | ## AI Risk Scoring\nRoutes transaction metadata to AI models that calculate fraud probability based on behavioral patterns.\n**Why** - Identifies suspicious activities that rule-based systems miss, adapting to evolving fraud tactics dynamically. |
| Structured Output Parser - Fraud Analysis | Structured Output Parser | Enforces schema for fraud analysis | — (AI connection) | AI Agent - Fraud Detection (ai_outputParser) | ## AI Risk Scoring\nRoutes transaction metadata to AI models that calculate fraud probability based on behavioral patterns.\n**Why** - Identifies suspicious activities that rule-based systems miss, adapting to evolving fraud tactics dynamically. |
| Route by Risk Level | Switch | Branch by riskLevel | AI Agent - Fraud Detection | Flag for Manual Review; Reconcile with ERP System | ## Alert Routing \nSends high-risk transaction alerts to finance teams via Gmail and Slack with detailed context.\n**Why** - Enables immediate human review and action on potential fraud before financial impact escalates. |
| Flag for Manual Review | Set | Annotate high-risk items for review | Route by Risk Level (High) | Notify Finance Team - High Risk; Send Customer Alert Email | ## Alert Routing \nSends high-risk transaction alerts to finance teams via Gmail and Slack with detailed context.\n**Why** - Enables immediate human review and action on potential fraud before financial impact escalates. |
| Notify Finance Team - High Risk | Slack | Post urgent alert to Slack | Flag for Manual Review | — | ## Alert Routing \nSends high-risk transaction alerts to finance teams via Gmail and Slack with detailed context.\n**Why** - Enables immediate human review and action on potential fraud before financial impact escalates. |
| Send Customer Alert Email | Gmail | Email alert (configured to financeTeamEmail) | Flag for Manual Review | — | ## Alert Routing \nSends high-risk transaction alerts to finance teams via Gmail and Slack with detailed context.\n**Why** - Enables immediate human review and action on potential fraud before financial impact escalates. |
| Reconcile with ERP System | HTTP Request | POST reconciliation payload to ERP | Route by Risk Level (Medium/Low) | Store Transaction Record | ## Prerequisites\nActive accounts for payment processors (Stripe, PayPal) or banking APIs (Plaid)\n## Use Cases\nReal-time credit card transaction monitoring with instant fraud blocks\n## Customization\nAdjust fraud risk scoring thresholds based on business risk tolerance\n## Benefits\nReduces fraud detection time from hours to seconds through real-time monitoring. |
| Store Transaction Record | Postgres | Persist transaction row | Reconcile with ERP System | Store Audit Trail | ## Prerequisites\nActive accounts for payment processors (Stripe, PayPal) or banking APIs (Plaid)\n## Use Cases\nReal-time credit card transaction monitoring with instant fraud blocks\n## Customization\nAdjust fraud risk scoring thresholds based on business risk tolerance\n## Benefits\nReduces fraud detection time from hours to seconds through real-time monitoring. |
| Store Audit Trail | Postgres | Persist audit row | Store Transaction Record | Aggregate for Financial Report | ## Prerequisites\nActive accounts for payment processors (Stripe, PayPal) or banking APIs (Plaid)\n## Use Cases\nReal-time credit card transaction monitoring with instant fraud blocks\n## Customization\nAdjust fraud risk scoring thresholds based on business risk tolerance\n## Benefits\nReduces fraud detection time from hours to seconds through real-time monitoring. |
| Aggregate for Financial Report | Aggregate | Combine items for batch reporting | Store Audit Trail | Generate Financial Report |  |
| Generate Financial Report | Code | Compute daily/batch summary metrics | Aggregate for Financial Report | Send Financial Report to Slack |  |
| Send Financial Report to Slack | Slack | Post summary report to Slack | Generate Financial Report | — |  |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Name it: *Multi-Source Transaction Fraud Detection and Financial Reconciliation Automation* (or your preferred name).
- Keep it inactive until credentials and endpoints are configured.

2) **Add Trigger: “Schedule Trigger - Periodic Ingestion”**
- Node type: **Schedule Trigger**
- Set interval: every **15 minutes**.

3) **Add a Set node: “Workflow Configuration”**
- Node type: **Set**
- Enable “Keep Only Set” = **false** (equivalent to `includeOtherFields: true`)
- Add fields:
  - `bankApiUrl` (string): your bank transactions API endpoint
  - `paymentGatewayApiUrl` (string): your gateway endpoint
  - `erpApiUrl` (string): your ERP reconciliation endpoint
  - `fraudThresholdHigh` (number): 0.8 *(optional; currently unused)*
  - `fraudThresholdMedium` (number): 0.5 *(optional; currently unused)*
  - `slackChannel` (string): Slack channel ID
  - `financeTeamEmail` (string): email recipient for alerts
- Connect: **Schedule Trigger → Workflow Configuration**

4) **Add HTTP Request: “Fetch Bank API Transactions”**
- Node type: **HTTP Request**
- Method: GET (default)
- URL: expression from config: `{{ $('Workflow Configuration').first().json.bankApiUrl }}`
- Query parameters:
  - `from_date = {{ $now.minus({ hours: 1 }).toISO() }}`
  - `to_date = {{ $now.toISO() }}`
- Options: set timeout to **30000 ms**
- Configure **authentication** required by your bank API (header API key/OAuth2/etc.).
- Connect: **Workflow Configuration → Fetch Bank API Transactions**

5) **Add HTTP Request: “Fetch Payment Gateway Transactions”**
- Same as bank fetch, but URL from `paymentGatewayApiUrl`.
- Connect: **Workflow Configuration → Fetch Payment Gateway Transactions**

6) **Add Webhook: “Webhook - Real-time Transaction Events”**
- Node type: **Webhook**
- HTTP Method: **POST**
- Path: `transaction-webhook`
- Response mode: **Last node**
- Deploy and note the production URL to configure in upstream systems.

7) **Add Merge: “Merge All Transaction Sources”**
- Node type: **Merge**
- Set **Number of Inputs = 3**
- Connect inputs:
  - **Webhook → Merge** (input 1 / index 0)
  - **Fetch Bank API Transactions → Merge** (input 2 / index 1)
  - **Fetch Payment Gateway Transactions → Merge** (input 3 / index 2)

8) **Add Set: “Normalize Transaction Data”**
- Node type: **Set**
- Keep other fields: **true**
- Add mappings (expressions):
  - `transactionId = {{ $json.id || $json.transaction_id || $json.txn_id }}`
  - `amount = {{ parseFloat($json.amount || $json.value || 0) }}`
  - `currency = {{ $json.currency || 'USD' }}`
  - `timestamp = {{ $json.timestamp || $json.created_at || $now.toISO() }}`
  - `customerId = {{ $json.customer_id || $json.user_id || 'unknown' }}`
  - `merchantId = {{ $json.merchant_id || $json.vendor_id || 'unknown' }}`
  - `paymentMethod = {{ $json.payment_method || $json.method || 'unknown' }}`
  - `source = {{ $json.source || 'api' }}`
- Connect: **Merge → Normalize**

9) **Add Filter: “Filter Valid Transactions”**
- Node type: **Filter**
- Create conditions (AND):
  - `transactionId` is not empty
  - `amount` > 0
  - `customerId` is not empty
- Connect: **Normalize → Filter**

10) **Add Code: “Check Idempotency”**
- Node type: **Code**
- Paste JS logic to deduplicate by `transactionId` using workflow static data.
- Connect: **Filter → Code**
- Recommended improvement when rebuilding: store `processedIds` as an array/object map to avoid Set serialization issues.

11) **Add AI nodes (LangChain)**
11.1) **OpenAI Chat Model**
- Node type: **OpenAI Chat Model** (LangChain)
- Model: **gpt-4o**
- Credentials: create/select **OpenAI API** credential.

11.2) **Structured Output Parser - Fraud Analysis**
- Node type: **Structured Output Parser**
- Schema: define the manual JSON schema with required fields:
  - riskScore (number), riskLevel (enum high/medium/low), fraudIndicators (array of strings), anomalyDetected (boolean), complianceFlags (array of strings), recommendation (string)

11.3) **AI Agent - Fraud Detection**
- Node type: **AI Agent**
- Prompt/input text: `{{ $json }}`
- System message: include your fraud analysis instructions and the required output fields.
- Enable structured output and connect:
  - AI language model input: **OpenAI Chat Model**
  - Output parser input: **Structured Output Parser**
- Main connection: **Check Idempotency → AI Agent**

12) **Add Switch: “Route by Risk Level”**
- Node type: **Switch**
- Rules:
  - If `{{ $json.riskLevel }}` equals `high` → output “High Risk”
  - equals `medium` → “Medium Risk”
  - equals `low` → “Low Risk”
- Set fallback output name to `extra` (optional).
- Connect: **AI Agent → Switch**

13) **High-risk branch: add Set + Slack + Gmail**
13.1) **Set: “Flag for Manual Review”**
- Adds:
  - `requiresManualReview = true`
  - `reviewReason = {{ $json.recommendation }}`
  - `flaggedAt = {{ $now.toISO() }}`
- Connect: **Switch (High Risk) → Flag for Manual Review**

13.2) **Slack: “Notify Finance Team - High Risk”**
- Node type: **Slack**
- Auth: **OAuth2** Slack credential
- Post message to a channel by ID:
  - Channel ID: `{{ $('Workflow Configuration').first().json.slackChannel }}`
- Message includes transaction details and indicators.
- Connect: **Flag for Manual Review → Slack**

13.3) **Gmail: “Send Customer Alert Email”**
- Node type: **Gmail**
- Credentials: Gmail OAuth2
- To: `{{ $('Workflow Configuration').first().json.financeTeamEmail }}`
- Subject/body: suspicious transaction message.
- Connect: **Flag for Manual Review → Gmail**
- If you truly want customer notifications, change the “To” to a customer email field and ensure it exists.

14) **Medium/Low branch: ERP reconciliation**
14.1) **HTTP Request: “Reconcile with ERP System”**
- Node type: **HTTP Request**
- Method: **POST**
- URL: `{{ $('Workflow Configuration').first().json.erpApiUrl }}`
- Body: JSON containing transaction identifiers + AI risk score; include `status: "reconciled"`.
- Add ERP authentication.
- Connect: **Switch (Medium) → Reconcile** and **Switch (Low) → Reconcile**

15) **Postgres persistence**
15.1) **Postgres: “Store Transaction Record”**
- Node type: **Postgres**
- Credentials: configure a Postgres credential (host/db/user/password or connection string).
- Operation: insert into `public.transactions`
- Map columns to transaction fields (amount, currency, timestamp, ids, risk fields, JSON strings).
- Connect: **Reconcile → Store Transaction Record**

15.2) **Postgres: “Store Audit Trail”**
- Insert into `public.audit_trail` with action metadata.
- Connect: **Store Transaction Record → Store Audit Trail**

16) **Reporting chain**
16.1) **Aggregate: “Aggregate for Financial Report”**
- Aggregate mode: **All item data** (collect items to `.json.data`)
- Connect: **Store Audit Trail → Aggregate**

16.2) **Code: “Generate Financial Report”**
- Compute totals and risk breakdown.
- Ensure you guard against empty arrays to avoid divide-by-zero.
- Connect: **Aggregate → Generate**

16.3) **Slack: “Send Financial Report to Slack”**
- Post summary to same Slack channel (by ID from config).
- Connect: **Generate → Slack**

17) **Activate workflow**
- Test webhook path with sample JSON.
- Test scheduled branch by running manually once, then activate.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “## How It Works … AI models analyze … audit trails are logged to Google Sheets …” | Sticky note describes Google Sheets logging, but the actual workflow persists to **Postgres** and posts to **Slack**; no Google Sheets node is present. |
| “## Setup Steps … Configure NVIDIA NIM API … Link Google Sheets …” | Sticky note mentions Nvidia NIM + Google Sheets, but these integrations are not implemented in the provided JSON. Add the missing nodes if required. |
| “## Prerequisites … Stripe, PayPal … Plaid” | Conceptual prerequisites; actual HTTP nodes are generic and must be configured to match your providers. |
| Disclaimer (FR): “Le texte fourni provient exclusivement…” | Provided by user; confirms content is from an automated n8n workflow and compliant with policies. |