Monitor cryptocurrency payments across multiple blockchains with AgentGatePay

https://n8nworkflows.xyz/workflows/monitor-cryptocurrency-payments-across-multiple-blockchains-with-agentgatepay-12015


# Monitor cryptocurrency payments across multiple blockchains with AgentGatePay

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

# 1. Workflow Overview

**Workflow name:** 💲 📊 AgentGatePay - Monitoring Dashboard  
**Purpose:** This workflow builds monitoring dashboards and exportable reports for **AgentGatePay** cryptocurrency payment activity across multiple blockchains. It contains **two independent monitoring pipelines**:
- **Buyer Monitoring Dashboard**: tracks a buyer’s spending, payment history, audit logs, active mandates, optional on-chain verification, statistics, alerts, and exports.
- **Seller Monitoring Dashboard**: tracks a merchant’s revenue, payments received, webhook configuration, audit logs, optional on-chain verification, statistics, alerts, and exports.

**Target use cases**
- Manual, on-demand monitoring (no webhook required)
- Troubleshooting transactions (optional `tx_hash` verification)
- Accounting/ops: CSV exports and aggregated metrics
- Risk/operations: alerting on budget depletion, failed payments, webhook failures

## 1.1 Buyer Pipeline — Logical blocks
1. **Input & Configuration** (Manual trigger + config validation)
2. **Data Collection (Buyer)** (analytics, payment history, audit logs, mandates)
3. **Optional Blockchain Verification (Buyer)**
4. **Analytics & Alerts (Buyer)**
5. **Output Generation (Buyer)** (dashboard formatting + CSV + final report)

## 1.2 Seller Pipeline — Logical blocks
1. **Input & Configuration (Seller)**
2. **Data Collection (Seller)** (merchant revenue, payment list, webhooks, audit logs)
3. **Optional Blockchain Verification (Seller)**
4. **Analytics & Alerts (Seller)**
5. **Output Generation (Seller)** (dashboard formatting + CSV + final report)

**Entry points**
- Only **one actual trigger node** exists: **▶️ Manual Trigger** (Buyer pipeline).
- The Seller pipeline has **no trigger** in this JSON; it appears to be a second, separate branch that must be triggered by adding a trigger node and connecting it to `2️⃣ Load Config1`, or by executing nodes manually in the editor.

---

# 2. Block-by-Block Analysis

## Block A — Buyer: Input & Configuration

### Overview
Starts the workflow manually and creates a normalized `config` object (API key, user identity, wallet, limits, endpoints). Enforces that an API key is configured.

### Nodes involved
- ▶️ Manual Trigger
- 2️⃣ Load Config

### Node details

#### ▶️ Manual Trigger
- **Type / role:** `n8n-nodes-base.manualTrigger` — manual execution entry point.
- **Configuration choices:** No parameters.
- **Outputs:** A single empty item is emitted.
- **Connections:** Outputs to **2️⃣ Load Config**.
- **Edge cases:**
  - None; always runs when user clicks “Execute Workflow”.
- **Sticky note:**  
  From “START HERE”: Buyer Monitoring Dashboard quick start + what you’ll see + export note.

#### 2️⃣ Load Config
- **Type / role:** `n8n-nodes-base.code` — validates/sets runtime configuration.
- **Key configuration produced (interpreted):**
  - `user_email` (default `"user@example.com"` if not provided)
  - `api_key` (default `"YOUR_API_KEY"`; **must be replaced**)
  - `buyer_wallet` (default `"YOUR_WALLET_ADDRESS"`; should be replaced with 0x…)
  - `tx_hash` optional (default `null`)
  - Monitoring: `time_range_hours=24`, `payment_history_limit=50`, `audit_log_limit=50`
  - Endpoints: `api_url=https://api.agentgatepay.com`, `mcp_endpoint=https://mcp.agentgatepay.com`
  - `session.id=monitor_<timestamp>`
- **Validation behavior:**
  - Throws error if `api_key === "YOUR_API_KEY"`.
- **Expressions/variables:**
  - Uses `$input.first().json` to check for externally provided values.
  - `isManual = !input.user_email`
- **Connections:**
  - Main output → **3️⃣ 📈 Get User Analytics**
- **Edge cases / failure types:**
  - Misconfiguration: API key not set → workflow hard-fails with explicit error.
  - `buyer_wallet` not validated here; downstream API calls may return empty logs if wrong.
- **Sticky note:**  
  From “START HERE” and “Data Collection” (covers the buyer collection area): explains what is fetched.

---

## Block B — Buyer: Data Collection

### Overview
Pulls buyer analytics, recent payment history (via MCP JSON-RPC), audit logs filtered to the buyer wallet, and “active mandates” (mandate-issued events) from the AgentGatePay API.

### Nodes involved
- 3️⃣ 📈 Get User Analytics
- 4️⃣ 💳 Get Payment History
- 5️⃣ 📋 Get Audit Logs (24h)
- 6️⃣ 🔑 Get Active Mandates

### Node details

#### 3️⃣ 📈 Get User Analytics
- **Type / role:** `n8n-nodes-base.httpRequest` — calls AgentGatePay analytics endpoint for the authenticated user.
- **Request:**
  - `GET {{$json.config.api_url}}/v1/analytics/me`
  - Header: `x-api-key: {{$json.config.api_key}}`
- **Connections:**
  - Input from **2️⃣ Load Config**
  - Output to **4️⃣ 💳 Get Payment History**
- **Edge cases:**
  - 401/403 if API key invalid.
  - API schema mismatch: downstream code expects `total_spent_usd` and `transaction_count` (buyer stats node accounts for this).
  - Network timeouts or non-JSON response.

#### 4️⃣ 💳 Get Payment History
- **Type / role:** `n8n-nodes-base.httpRequest` — calls AgentGatePay MCP endpoint using JSON-RPC `tools/call`.
- **Request:**
  - `POST {{config.mcp_endpoint}}`
  - JSON body calls tool: `agentpay_get_payment_history` with `limit={{config.payment_history_limit}}`
  - Headers: `Content-Type: application/json`, `x-api-key`
- **Important parsing detail:**
  - Downstream code expects an MCP response shaped like:
    - `result.content[0].text` containing JSON text with `{ "payments": [...] }`.
- **Connections:** Output → **5️⃣ 📋 Get Audit Logs (24h)**
- **Edge cases:**
  - MCP may return different `result` format; JSON parsing can fail (handled with try/catch, but payments become `[]`).
  - Tool name or MCP endpoint may change server-side.

#### 5️⃣ 📋 Get Audit Logs (24h) (Buyer)
- **Type / role:** `n8n-nodes-base.httpRequest` — fetches audit logs for the **buyer wallet**.
- **Request:**
  - `GET {{api_url}}/audit/logs?client_id={{buyer_wallet}}&event_type=x402_payment_settled&hours={{time_range_hours}}&limit={{audit_log_limit}}`
  - Header: `x-api-key`
- **Connections:** Output → **6️⃣ 🔑 Get Active Mandates**
- **Edge cases:**
  - If `buyer_wallet` is unset/placeholder, logs likely empty.
  - If `details` in logs are strings vs objects, downstream nodes sometimes parse them (mandate parsing does).
  - Pagination not handled (hard limited to 50).

#### 6️⃣ 🔑 Get Active Mandates
- **Type / role:** `n8n-nodes-base.httpRequest` — fetches mandate issuance events.
- **Request:**
  - `GET {{api_url}}/audit/logs?client_id={{user_email}}&event_type=mandate_issued`
  - Header: `x-api-key`
- **Important nuance:**
  - Uses `client_id={{user_email}}` (not wallet). This assumes mandate issuance events are keyed by email.
- **Connections:** Output → **7️⃣ 🔍 Prepare Verification**
- **Edge cases:**
  - If mandates are keyed by wallet rather than email, results may be empty.
  - `logs[].details` may be JSON string; downstream parsing handles this.

**Sticky note (applies to these nodes):**  
“## Data Collection — Fetches analytics, payment history, audit logs, and active mandates … filtered by your wallet address.”

---

## Block C — Buyer: Optional Blockchain Verification

### Overview
If a `tx_hash` is provided, verifies it on-chain via AgentGatePay verification endpoint; otherwise skips verification and merges a default “not verified” result.

### Nodes involved
- 7️⃣ 🔍 Prepare Verification
- 7B️⃣ Has TX Hash?
- 8️⃣ ✅ Verify on Blockchain
- 8B️⃣ Skip Verification
- 9️⃣ Merge Verification

### Node details

#### 7️⃣ 🔍 Prepare Verification
- **Type / role:** `n8n-nodes-base.code` — sets `should_verify` based on presence of `config.tx_hash`.
- **Behavior:**
  - If no `tx_hash`: returns `{ verified:false, reason:"No transaction hash provided", config }`
  - If present: returns `{ tx_hash, config, should_verify:true }`
- **Connections:** Output → **7B️⃣ Has TX Hash?**
- **Edge cases:**
  - If config node uses `tx_hash=null`, verification is skipped as intended.

#### 7B️⃣ Has TX Hash?
- **Type / role:** `n8n-nodes-base.if` — routes based on boolean `should_verify`.
- **Condition:** `{{$json.should_verify}} is true`
- **Connections:**
  - True → **8️⃣ ✅ Verify on Blockchain**
  - False → **8B️⃣ Skip Verification**
- **Edge cases:**
  - If `should_verify` is undefined (no tx_hash path returns object without it), condition evaluates false → goes to skip path (correct).

#### 8️⃣ ✅ Verify on Blockchain
- **Type / role:** `n8n-nodes-base.httpRequest` — verifies transaction via AgentGatePay API.
- **Request:**
  - `GET {{api_url}}/v1/payments/verify/{{$json.tx_hash}}`
  - Header: `x-api-key`
- **Connections:** Output → **9️⃣ Merge Verification** (input index 0)
- **Edge cases:**
  - 404 if tx hash not known or not yet indexed.
  - Verification response schema might not include `verified`; downstream code checks `verification.verified || false`.
  - Chain explorer delays / eventual consistency.

#### 8B️⃣ Skip Verification
- **Type / role:** `n8n-nodes-base.code` — outputs a default “skipped” verification object.
- **Connections:** Output → **9️⃣ Merge Verification** (input index 1)

#### 9️⃣ Merge Verification
- **Type / role:** `n8n-nodes-base.merge` — combines the true/false branch results.
- **Mode:** `combine` with `mergeByPosition`
- **Connections:** Output → **🔟 📊 Calculate Statistics**
- **Edge cases:**
  - If either branch returns multiple items (not expected), merge-by-position can misalign items.

**Sticky note (applies to this block):**  
“## Blockchain Verification — If you provide a tx_hash, verifies the payment on-chain…”

---

## Block D — Buyer: Analytics & Alerts

### Overview
Aggregates all data sources, computes stats (spending, budget utilization, event counts), then generates alerts for low budget, expiring mandates, failures, etc.

### Nodes involved
- 🔟 📊 Calculate Statistics
- 1️⃣1️⃣ 🚨 Check Alerts

### Node details

#### 🔟 📊 Calculate Statistics
- **Type / role:** `n8n-nodes-base.code` — computes buyer metrics from multiple upstream nodes.
- **Inputs consumed via node lookups:**
  - `config` from **2️⃣ Load Config**
  - `analytics` from **3️⃣ 📈 Get User Analytics**
  - MCP response from **4️⃣ 💳 Get Payment History** (parsed)
  - `audit logs` from **5️⃣ 📋 Get Audit Logs (24h)**
  - `mandates` from **6️⃣ 🔑 Get Active Mandates**
  - `verification` from merge input
- **Key calculations:**
  - Uses analytics fields: `total_spent_usd`, `transaction_count` (**explicit fix in code**)
  - Derives payments last 24h and spent last 24h from payment history timestamps
  - Active mandates count from mandate logs where `details.status in ['active','issued']`
  - Budget totals from sum of `details.budget_usd` and `details.budget_remaining`
  - Audit log event category counts based on substring match of `event_type`
- **Outputs:**
  - `{ config, analytics, stats, payments, logs, mandates, verification }`
- **Edge cases / failure types:**
  - MCP parsing: `result.content[0].text` may not be JSON → payments fall back to `[]`.
  - Mandate details parsing: `JSON.parse(m.details)` can throw if malformed; this is not try/catched in mandate loops (potential crash).
  - Timestamp fields: uses `p.timestamp || p.created_at`; invalid dates could produce `Invalid Date` and filter logic may behave unexpectedly.

#### 1️⃣1️⃣ 🚨 Check Alerts
- **Type / role:** `n8n-nodes-base.code` — generates an `alerts[]` array.
- **Alert rules:**
  - Budget utilization > 90% → high; > 75% → medium
  - Mandate `ttl_remaining_hours < 24` → high
  - No payments in last 24h but historical payments exist → low
  - Any payments with `status === 'failed'` → high
  - High spending: spent_24h > average_payment * 10 and payment_count > 10 → medium
  - Verification failure when a tx_hash was provided → medium
- **Outputs:** Adds `alerts` to the data object.
- **Edge cases:**
  - Assumes mandate details are parseable JSON if string; can throw.
  - Some systems may use `status` values other than `failed/completed/confirmed`.

**Sticky note (applies to this block):**  
“## Analysis & Alerts — Calculates spending trends, budget utilization, and checks for issues…”

---

## Block E — Buyer: Output Generation

### Overview
Builds a dashboard structure for display, generates a CSV report, then creates a comprehensive “final report” with corrected commission/merchant breakdown derived from audit logs.

### Nodes involved
- 1️⃣2️⃣ 📱 Format Dashboard
- 1️⃣3️⃣ 📄 Generate CSV Export
- 1️⃣4️⃣ 📋 Final Report

### Node details

#### 1️⃣2️⃣ 📱 Format Dashboard
- **Type / role:** `n8n-nodes-base.code` — creates a `dashboard` object with:
  - metrics cards, alert summary, quick stats, API links, export availability
- **Note:** This dashboard is later removed in Final Report (`delete data.dashboard`) to avoid duplication.
- **Connections:** Output → **1️⃣3️⃣ 📄 Generate CSV Export**
- **Edge cases:**
  - Assumes numeric stats; uses `toFixed()` extensively.

#### 1️⃣3️⃣ 📄 Generate CSV Export
- **Type / role:** `n8n-nodes-base.code` — creates a CSV string in `csv_export`.
- **What it includes:**
  - Summary metrics
  - Alerts table
  - “Merchant payments” from logs (`event_type === x402_payment_settled`)
  - Commission lines where `details.commission_tx_hash` exists (still from `x402_payment_settled`)
  - Mandates table
  - Event summary counts
- **Connections:** Output → **1️⃣4️⃣ 📋 Final Report**
- **Edge cases:**
  - CSV quoting is partial; values containing commas/newlines could break CSV.
  - Assumes `log.details.timestamp` is epoch seconds.

#### 1️⃣4️⃣ 📋 Final Report
- **Type / role:** `n8n-nodes-base.code` — produces final structured output:
  - `report`: formatted object with sections (key metrics, alerts, payments, mandates, curl commands, endpoints, export)
  - `raw_data`: underlying data (minus dashboard)
- **Notable “correctness fixes” embedded in code:**
  - Commission is **embedded** in `x402_payment_settled` events (not separate events).
  - Computes:
    - `commission_rate = 0.005`
    - `total_commission` sum from embedded commission amounts
    - `total_merchant` sum of merchant amounts
    - `total_spent` = merchant + commission (fallback: `commission/0.005` if merchant data missing)
  - Recomputes budget remaining as `total_budget - total_spent` (default `total_budget=100` if no mandates)
- **Edge cases / caveats:**
  - Mandate remaining spend is simplified: `mandate_spent = total_spent` (assumes one mandate).
  - Mixed chains/explorers: defaults explorer URLs to BaseScan; multi-chain explorers may differ.
  - If audit logs don’t contain `merchant_tx_hash` / `commission_tx_hash`, some lists become empty and totals use fallbacks.

**Sticky note (applies to this block):**  
“## Output Generation — Formats dashboard … generates CSV … Node 14 is your main output.”

---

## Block F — Seller: Input & Configuration (not triggered)

### Overview
Creates merchant config for seller monitoring (wallet + API key). Enforces both API key and merchant wallet are configured.

### Nodes involved
- 2️⃣ Load Config1

### Node details

#### 2️⃣ Load Config1
- **Type / role:** `n8n-nodes-base.code` — seller config creation/validation.
- **Key configuration:**
  - `merchant_wallet` (default `"0xYOUR_WALLET_ADDRESS"`; must be replaced)
  - `api_key` must be replaced
  - `tx_hash` optional
  - Similar limits/endpoints/session as buyer.
- **Validation:**
  - Throws if API key placeholder
  - Throws if merchant wallet placeholder
- **Connections:** Output → **3️⃣ 💰 Get Merchant Revenue**
- **Edge cases:**
  - No trigger node wired to it in this workflow; will not run unless manually executed or you add a trigger.
- **Sticky note:**  
  From “START HERE1”: Seller dashboard quick start, expectations, export note.

---

## Block G — Seller: Data Collection

### Overview
Fetches merchant revenue, payment list, webhook list, and audit logs for recent payment events.

### Nodes involved
- 3️⃣ 💰 Get Merchant Revenue
- 4️⃣ 💳 Get Payment List
- 5️⃣ 🔗 Get Webhooks
- 6️⃣ 📋 Get Audit Logs (24h) (Seller)

### Node details

#### 3️⃣ 💰 Get Merchant Revenue
- **Type / role:** `n8n-nodes-base.httpRequest`
- **Request:**
  - `GET {{api_url}}/v1/merchant/revenue?wallet={{config.merchant_wallet}}`
  - Header: `x-api-key`
- **Connection:** → **4️⃣ 💳 Get Payment List**
- **Edge cases:** auth errors; schema differences handled downstream (expects `total_usd`, `count`, `average_usd`).

#### 4️⃣ 💳 Get Payment List
- **Type / role:** `n8n-nodes-base.httpRequest`
- **Request:**
  - `GET {{api_url}}/v1/payments/list?wallet={{merchant_wallet}}&limit={{payment_history_limit}}`
- **Connection:** → **5️⃣ 🔗 Get Webhooks**
- **Edge cases:** empty payments for new merchants; pagination not handled.

#### 5️⃣ 🔗 Get Webhooks
- **Type / role:** `n8n-nodes-base.httpRequest`
- **Request:** `GET {{api_url}}/v1/webhooks/list`
- **Connection:** → **6️⃣ 📋 Get Audit Logs (24h)** (Seller)
- **Edge cases:** Some accounts may not have permissions to list webhooks.

#### 6️⃣ 📋 Get Audit Logs (24h) (Seller)
- **Type / role:** `n8n-nodes-base.httpRequest`
- **Request:** `GET {{api_url}}/audit/logs?event_type=x402_payment_settled&hours={{...}}&limit={{...}}`
- **Note:** Does **not** filter by merchant wallet; it pulls all settlement events visible to the key.
- **Connection:** → **7️⃣ 🔍 Prepare Verification1**
- **Edge cases:** Very noisy logs if account sees many merchants; might need `client_id` filter.

**Sticky note (applies to this block):**  
“## Revenue Collection — Fetches merchant revenue, payments received, webhook config, and audit logs…”

---

## Block H — Seller: Optional Blockchain Verification

### Overview
Same pattern as buyer verification but intended for payments received.

### Nodes involved
- 7️⃣ 🔍 Prepare Verification1
- 7B️⃣ Has TX Hash?1
- 8️⃣ ✅ Verify on Blockchain1
- 8B️⃣ Skip Verification1
- 9️⃣ Merge Verification1

### Node details (differences vs buyer)
- Uses `2️⃣ Load Config1` for config lookups.
- Verification endpoint is the same: `/v1/payments/verify/{tx_hash}`.
- Merge settings identical (`combine` / `mergeByPosition`).

**Sticky note (applies to this block):**  
“## Payment Verification — verifies the payment you received on-chain…”

---

## Block I — Seller: Analytics & Alerts

### Overview
Computes merchant KPIs (revenue, payments, webhook success, top buyers), then produces alerts related to webhooks and payment success.

### Nodes involved
- 🔟 📊 Calculate Statistics1
- 1️⃣1️⃣ 🚨 Check Alerts1

### Node details

#### 🔟 📊 Calculate Statistics1
- **Type / role:** `n8n-nodes-base.code`
- **Inputs consumed via lookups:**
  - Config, revenue, payment list, webhooks, audit logs, verification.
- **Key calculations:**
  - Uses revenue fields: `total_usd`, `count` (**explicit fix**), derives average
  - Webhook metrics: active count, delivery/failure totals, computed success rate
  - Payment success rate from payment statuses
- **Output:** `{ config, revenue, stats, payments, webhooks, logs, verification }`
- **Edge cases:**
  - Assumes webhook list fields: `status`, `delivery_count`, `failure_count` (may differ by API).
  - `stats.payment_success_rate` computed only from known status values.

#### 1️⃣1️⃣ 🚨 Check Alerts1
- **Type / role:** `n8n-nodes-base.code`
- **Alert rules (seller-specific):**
  - webhook success rate < 95% → high
  - no payments 24h but historical exists → medium
  - failed payments > 0 → high
  - last payment > 10x average → low
  - revenue spike last 24h → low
  - no webhooks configured while having >5 payments → medium
  - verification failure when tx hash provided → medium
  - payment success rate < 90% and payment_count > 10 → high
- **Edge cases:** Depends on accurate webhook and payment status fields.

**Sticky note (applies to this block):**  
“## Analysis & Alerts — Calculates revenue trends, webhook success rates, top buyers…”

---

## Block J — Seller: Output Generation

### Overview
Formats a seller dashboard, generates merchant CSV export, then produces a seller final report with corrected commission/original amount logic.

### Nodes involved
- 1️⃣2️⃣ 📱 Format Dashboard1
- 1️⃣3️⃣ 📄 Generate CSV Export1
- 1️⃣4️⃣ 📋 Final Report1

### Node details

#### 1️⃣2️⃣ 📱 Format Dashboard1
- **Type / role:** `n8n-nodes-base.code`
- **Output:** `dashboard` with merchant-centric metrics and links.
- **Note:** `full_dashboard` link states admin-only.
- **Edge cases:** Uses `toFixed()`; requires numeric stats.

#### 1️⃣3️⃣ 📄 Generate CSV Export1
- **Type / role:** `n8n-nodes-base.code`
- **CSV contents:**
  - Summary
  - Alerts
  - Payments received (from audit logs)
  - Commission deducted (embedded in settlement logs)
  - Top buyers (from stats)
  - Webhooks breakdown
- **Edge cases:** Similar CSV quoting limitations.

#### 1️⃣4️⃣ 📋 Final Report1
- **Type / role:** `n8n-nodes-base.code`
- **Notable logic:**
  - Derives merchant received totals and computes original paid amount:
    - Merchant receives ~99.5% when commission rate is 0.5%.
    - If merchant totals present: `original = merchant_received / 0.995`
    - Else derive from commission: `original = commission/0.005`, merchant received = `original*0.995`
  - Computes unique buyers from payer fields.
  - Webhook “active” detection uses `w.active` in some places, while earlier code used `w.status === 'active'` (possible mismatch).
- **Outputs:** `{ report, raw_data }` with dashboard removed from raw_data.
- **Edge cases:**
  - Webhook field inconsistency (`status` vs `active`) can misreport active count.
  - `audit_logs` link uses `client_id={{config.user_email}}` though seller config does not define `user_email` → likely a bug in link construction.

**Sticky note (applies to this block):**  
“## Output Generation — Formats dashboard … generates CSV export for accounting, and creates the final report…”

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| ▶️ Manual Trigger | Manual Trigger | Buyer pipeline entry point | — | 2️⃣ Load Config | # Buyer Monitoring Dashboard<br>**What it does:** Shows your spending, payment history, mandates, and alerts.<br>**Quick start:** 1. Edit Node 2… 2. Click Execute Workflow 3. Check Node 14…<br>**Export:** Node 13 has CSV data you can copy and save |
| 2️⃣ Load Config | Code | Buyer config normalization + validation | ▶️ Manual Trigger | 3️⃣ 📈 Get User Analytics | # Buyer Monitoring Dashboard (same as above)<br>## Data Collection<br>Fetches analytics, payment history, audit logs, and active mandates… |
| 3️⃣ 📈 Get User Analytics | HTTP Request | Fetch buyer analytics (`/v1/analytics/me`) | 2️⃣ Load Config | 4️⃣ 💳 Get Payment History | ## Data Collection<br>Fetches analytics, payment history, audit logs, and active mandates… |
| 4️⃣ 💳 Get Payment History | HTTP Request | Fetch payment history via MCP tool call | 3️⃣ 📈 Get User Analytics | 5️⃣ 📋 Get Audit Logs (24h) | ## Data Collection<br>Fetches analytics, payment history, audit logs, and active mandates… |
| 5️⃣ 📋 Get Audit Logs (24h) | HTTP Request | Fetch buyer wallet settlement logs | 4️⃣ 💳 Get Payment History | 6️⃣ 🔑 Get Active Mandates | ## Data Collection<br>Fetches analytics, payment history, audit logs, and active mandates… |
| 6️⃣ 🔑 Get Active Mandates | HTTP Request | Fetch mandate issuance logs | 5️⃣ 📋 Get Audit Logs (24h) | 7️⃣ 🔍 Prepare Verification | ## Data Collection<br>Fetches analytics, payment history, audit logs, and active mandates… |
| 7️⃣ 🔍 Prepare Verification | Code | Determine if tx_hash verification should run | 6️⃣ 🔑 Get Active Mandates | 7B️⃣ Has TX Hash? | ## Blockchain Verification<br>If you provide a tx_hash, verifies the payment on-chain… |
| 7B️⃣ Has TX Hash? | IF | Route verify vs skip | 7️⃣ 🔍 Prepare Verification | 8️⃣ ✅ Verify on Blockchain; 8B️⃣ Skip Verification | ## Blockchain Verification<br>If you provide a tx_hash, verifies the payment on-chain… |
| 8️⃣ ✅ Verify on Blockchain | HTTP Request | Verify transaction (`/v1/payments/verify/{tx}`) | 7B️⃣ Has TX Hash? (true) | 9️⃣ Merge Verification | ## Blockchain Verification<br>If you provide a tx_hash, verifies the payment on-chain… |
| 8B️⃣ Skip Verification | Code | Produce “skipped verification” result | 7B️⃣ Has TX Hash? (false) | 9️⃣ Merge Verification | ## Blockchain Verification<br>If you provide a tx_hash, verifies the payment on-chain… |
| 9️⃣ Merge Verification | Merge | Merge verification paths | 8️⃣ ✅ Verify on Blockchain; 8B️⃣ Skip Verification | 🔟 📊 Calculate Statistics | ## Blockchain Verification<br>If you provide a tx_hash, verifies the payment on-chain… |
| 🔟 📊 Calculate Statistics | Code | Compute buyer stats from collected data | 9️⃣ Merge Verification | 1️⃣1️⃣ 🚨 Check Alerts | ## Analysis & Alerts<br>Calculates spending trends, budget utilization… |
| 1️⃣1️⃣ 🚨 Check Alerts | Code | Generate buyer alerts | 🔟 📊 Calculate Statistics | 1️⃣2️⃣ 📱 Format Dashboard | ## Analysis & Alerts<br>Calculates spending trends, budget utilization… |
| 1️⃣2️⃣ 📱 Format Dashboard | Code | Build buyer dashboard object | 1️⃣1️⃣ 🚨 Check Alerts | 1️⃣3️⃣ 📄 Generate CSV Export | ## Output Generation<br>Formats dashboard… Node 14 is your main output. |
| 1️⃣3️⃣ 📄 Generate CSV Export | Code | Build buyer CSV export string | 1️⃣2️⃣ 📱 Format Dashboard | 1️⃣4️⃣ 📋 Final Report | ## Output Generation<br>Formats dashboard… Node 14 is your main output. |
| 1️⃣4️⃣ 📋 Final Report | Code | Produce buyer final report + raw_data | 1️⃣3️⃣ 📄 Generate CSV Export | — | ## Output Generation<br>Formats dashboard… Node 14 is your main output. |
| START HERE | Sticky Note | Comment / instructions | — | — | (sticky note node) |
| Sticky Note 1 | Sticky Note | Comment / section header | — | — | (sticky note node) |
| Sticky Note 2 | Sticky Note | Comment / section header | — | — | (sticky note node) |
| Sticky Note 3 | Sticky Note | Comment / section header | — | — | (sticky note node) |
| Sticky Note 4 | Sticky Note | Comment / section header | — | — | (sticky note node) |
| 2️⃣ Load Config1 | Code | Seller config normalization + validation | — (no trigger connected) | 3️⃣ 💰 Get Merchant Revenue | # Seller Monitoring Dashboard<br>**What it does:** Shows your revenue, payments received, webhooks, and alerts… |
| 3️⃣ 💰 Get Merchant Revenue | HTTP Request | Fetch merchant revenue | 2️⃣ Load Config1 | 4️⃣ 💳 Get Payment List | ## Revenue Collection<br>Fetches merchant revenue, payments received, webhook config, and audit logs… |
| 4️⃣ 💳 Get Payment List | HTTP Request | List payments to merchant wallet | 3️⃣ 💰 Get Merchant Revenue | 5️⃣ 🔗 Get Webhooks | ## Revenue Collection<br>Fetches merchant revenue, payments received, webhook config, and audit logs… |
| 5️⃣ 🔗 Get Webhooks | HTTP Request | List configured webhooks | 4️⃣ 💳 Get Payment List | 6️⃣ 📋 Get Audit Logs (24h) (Seller) | ## Revenue Collection<br>Fetches merchant revenue, payments received, webhook config, and audit logs… |
| 6️⃣ 📋 Get Audit Logs (24h) | HTTP Request | Fetch settlement logs (unfiltered) | 5️⃣ 🔗 Get Webhooks | 7️⃣ 🔍 Prepare Verification1 | ## Revenue Collection<br>Fetches merchant revenue, payments received, webhook config, and audit logs… |
| 7️⃣ 🔍 Prepare Verification1 | Code | Determine if seller tx verification should run | 6️⃣ 📋 Get Audit Logs (24h) (Seller) | 7B️⃣ Has TX Hash?1 | ## Payment Verification<br>If you provide a tx_hash, verifies the payment you received on-chain… |
| 7B️⃣ Has TX Hash?1 | IF | Route verify vs skip (seller) | 7️⃣ 🔍 Prepare Verification1 | 8️⃣ ✅ Verify on Blockchain1; 8B️⃣ Skip Verification1 | ## Payment Verification<br>If you provide a tx_hash, verifies the payment you received on-chain… |
| 8️⃣ ✅ Verify on Blockchain1 | HTTP Request | Verify seller-side transaction | 7B️⃣ Has TX Hash?1 (true) | 9️⃣ Merge Verification1 | ## Payment Verification<br>If you provide a tx_hash, verifies the payment you received on-chain… |
| 8B️⃣ Skip Verification1 | Code | Produce “skipped verification” (seller) | 7B️⃣ Has TX Hash?1 (false) | 9️⃣ Merge Verification1 | ## Payment Verification<br>If you provide a tx_hash, verifies the payment you received on-chain… |
| 9️⃣ Merge Verification1 | Merge | Merge seller verification paths | 8️⃣ ✅ Verify on Blockchain1; 8B️⃣ Skip Verification1 | 🔟 📊 Calculate Statistics1 | ## Payment Verification<br>If you provide a tx_hash, verifies the payment you received on-chain… |
| 🔟 📊 Calculate Statistics1 | Code | Compute seller stats (revenue/webhooks/payments) | 9️⃣ Merge Verification1 | 1️⃣1️⃣ 🚨 Check Alerts1 | ## Analysis & Alerts<br>Calculates revenue trends, webhook success rates, top buyers… |
| 1️⃣1️⃣ 🚨 Check Alerts1 | Code | Generate seller alerts | 🔟 📊 Calculate Statistics1 | 1️⃣2️⃣ 📱 Format Dashboard1 | ## Analysis & Alerts<br>Calculates revenue trends, webhook success rates, top buyers… |
| 1️⃣2️⃣ 📱 Format Dashboard1 | Code | Build seller dashboard object | 1️⃣1️⃣ 🚨 Check Alerts1 | 1️⃣3️⃣ 📄 Generate CSV Export1 | ## Output Generation<br>Formats dashboard… Node 14 is your main output. |
| 1️⃣3️⃣ 📄 Generate CSV Export1 | Code | Build seller CSV export string | 1️⃣2️⃣ 📱 Format Dashboard1 | 1️⃣4️⃣ 📋 Final Report1 | ## Output Generation<br>Formats dashboard… Node 14 is your main output. |
| 1️⃣4️⃣ 📋 Final Report1 | Code | Produce seller final report + raw_data | 1️⃣3️⃣ 📄 Generate CSV Export1 | — | ## Output Generation<br>Formats dashboard… Node 14 is your main output. |
| START HERE1 | Sticky Note | Comment / instructions | — | — | (sticky note node) |
| Sticky Note  | Sticky Note | Comment / section header | — | — | (sticky note node) |
| Sticky Note 5 | Sticky Note | Comment / section header | — | — | (sticky note node) |
| Sticky Note 6 | Sticky Note | Comment / section header | — | — | (sticky note node) |
| Sticky Note 7 | Sticky Note | Comment / section header | — | — | (sticky note node) |

---

# 4. Reproducing the Workflow from Scratch

## Part A — Buyer Monitoring Dashboard (fully runnable)

1. **Create node:** *Manual Trigger*  
   - Name: `▶️ Manual Trigger`

2. **Create node:** *Code*  
   - Name: `2️⃣ Load Config`  
   - Paste/implement logic equivalent to:
     - Build `config` with: `user_email`, `api_key`, `buyer_wallet`, optional `tx_hash`, limits (24h/50/50), endpoints, session info
     - Throw error if API key is placeholder
   - **Connect:** Manual Trigger → Load Config

3. **Create node:** *HTTP Request*  
   - Name: `3️⃣ 📈 Get User Analytics`  
   - Method: `GET`  
   - URL: `{{$json.config.api_url}}/v1/analytics/me`  
   - Header: `x-api-key = {{$json.config.api_key}}`  
   - **Connect:** Load Config → Get User Analytics

4. **Create node:** *HTTP Request* (MCP JSON-RPC)  
   - Name: `4️⃣ 💳 Get Payment History`  
   - Method: `POST`  
   - URL: `{{$('2️⃣ Load Config').first().json.config.mcp_endpoint}}`  
   - Body (JSON): call `tools/call` with `name=agentpay_get_payment_history` and `limit={{payment_history_limit}}`  
   - Headers: `Content-Type: application/json`, `x-api-key` from config  
   - **Connect:** Get User Analytics → Get Payment History

5. **Create node:** *HTTP Request*  
   - Name: `5️⃣ 📋 Get Audit Logs (24h)`  
   - Method: `GET`  
   - URL template:  
     `{{$('2️⃣ Load Config').first().json.config.api_url}}/audit/logs?client_id={{buyer_wallet}}&event_type=x402_payment_settled&hours={{time_range_hours}}&limit={{audit_log_limit}}`  
   - Header: `x-api-key`  
   - **Connect:** Get Payment History → Get Audit Logs (24h)

6. **Create node:** *HTTP Request*  
   - Name: `6️⃣ 🔑 Get Active Mandates`  
   - Method: `GET`  
   - URL: `{{api_url}}/audit/logs?client_id={{user_email}}&event_type=mandate_issued`  
   - Header: `x-api-key`  
   - **Connect:** Get Audit Logs (24h) → Get Active Mandates

7. **Create node:** *Code*  
   - Name: `7️⃣ 🔍 Prepare Verification`  
   - If no `config.tx_hash`, output a `{verified:false, reason:...}` object; else output `{should_verify:true, tx_hash,...}`  
   - **Connect:** Get Active Mandates → Prepare Verification

8. **Create node:** *IF*  
   - Name: `7B️⃣ Has TX Hash?`  
   - Condition: boolean `{{$json.should_verify}}` is true  
   - **Connect:** Prepare Verification → IF

9. **Create node:** *HTTP Request*  
   - Name: `8️⃣ ✅ Verify on Blockchain`  
   - Method: `GET`  
   - URL: `{{api_url}}/v1/payments/verify/{{$json.tx_hash}}`  
   - Header: `x-api-key`  
   - **Connect:** IF (true) → Verify on Blockchain

10. **Create node:** *Code*  
    - Name: `8B️⃣ Skip Verification`  
    - Output a default skip object  
    - **Connect:** IF (false) → Skip Verification

11. **Create node:** *Merge*  
    - Name: `9️⃣ Merge Verification`  
    - Mode: `Combine` → `Merge By Position`  
    - **Connect:** Verify on Blockchain → Merge (Input 1)  
    - **Connect:** Skip Verification → Merge (Input 2)

12. **Create node:** *Code*  
    - Name: `🔟 📊 Calculate Statistics`  
    - Implement calculations:
      - Parse MCP response text JSON into `payments[]`
      - Use analytics fields `total_spent_usd` and `transaction_count`
      - Compute 24h spend/count, mandate budgets, audit event counts
      - Output `{config, analytics, stats, payments, logs, mandates, verification}`
    - **Connect:** Merge Verification → Calculate Statistics

13. **Create node:** *Code*  
    - Name: `1️⃣1️⃣ 🚨 Check Alerts`  
    - Implement alert rules (budget utilization, mandate TTL, failures, etc.)
    - **Connect:** Calculate Statistics → Check Alerts

14. **Create node:** *Code*  
    - Name: `1️⃣2️⃣ 📱 Format Dashboard`  
    - Create `dashboard` object with metrics, alerts_summary, links, export flags
    - **Connect:** Check Alerts → Format Dashboard

15. **Create node:** *Code*  
    - Name: `1️⃣3️⃣ 📄 Generate CSV Export`  
    - Build `csv_export` string containing summary, alerts, payments/commissions, mandates, events
    - **Connect:** Format Dashboard → Generate CSV Export

16. **Create node:** *Code*  
    - Name: `1️⃣4️⃣ 📋 Final Report`  
    - Implement corrected totals from audit logs (commission embedded in settlement logs)
    - Output `{ report, raw_data }`
    - **Connect:** Generate CSV Export → Final Report

17. **Credentials**
    - No n8n credential object is used; API key is passed via header.
    - Ensure the AgentGatePay API key is stored safely (consider n8n credentials or environment variables instead of hardcoding).

---

## Part B — Seller Monitoring Dashboard (requires adding a trigger)

1. **Add a trigger node** (choose one):
   - Manual Trigger (recommended for parity), Cron, or Webhook.
2. **Create/keep node:** `2️⃣ Load Config1` (Code)
   - Must set `merchant_wallet` and `api_key` (validate both).
3. **Connect trigger → 2️⃣ Load Config1**.
4. Then create and connect the remaining nodes in order:
   - `3️⃣ 💰 Get Merchant Revenue` (GET `/v1/merchant/revenue?wallet=...`)
   - `4️⃣ 💳 Get Payment List` (GET `/v1/payments/list?wallet=...&limit=...`)
   - `5️⃣ 🔗 Get Webhooks` (GET `/v1/webhooks/list`)
   - `6️⃣ 📋 Get Audit Logs (24h)` (GET `/audit/logs?event_type=x402_payment_settled&hours=...&limit=...`)
   - Verification router: `7️⃣ 🔍 Prepare Verification1` → `7B️⃣ Has TX Hash?1` → (`8️⃣ ✅ Verify on Blockchain1` / `8B️⃣ Skip Verification1`) → `9️⃣ Merge Verification1`
   - `🔟 📊 Calculate Statistics1` → `1️⃣1️⃣ 🚨 Check Alerts1` → `1️⃣2️⃣ 📱 Format Dashboard1` → `1️⃣3️⃣ 📄 Generate CSV Export1` → `1️⃣4️⃣ 📋 Final Report1`

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Buyer Monitoring Dashboard quick start: edit Node 2 with wallet/email/API key, run manually, check Node 14 for full report; Node 13 provides CSV export. | Sticky note: “START HERE” |
| Seller Monitoring Dashboard quick start: edit Node 2 (seller version) with merchant wallet/API key, run, check Node 14; Node 13 provides CSV export. | Sticky note: “START HERE1” |
| Data Collection block notes (buyer): analytics, MCP payment history, audit logs, mandates; filtered by wallet. | Sticky note: “Data Collection” |
| Revenue Collection block notes (seller): revenue, payments received, webhooks, audit logs. | Sticky note: “Revenue Collection” |
| Blockchain verification is optional and requires providing `tx_hash`. | Sticky notes: “Blockchain Verification” / “Payment Verification” |
| Reports assume commission rate is **0.5% (0.005)**; totals are recomputed accordingly in Final Report nodes. | Implemented in both Final Report nodes |

