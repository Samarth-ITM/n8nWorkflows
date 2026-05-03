Create autonomous payment agents using AgentGatePay and multi-chain tokens

https://n8nworkflows.xyz/workflows/create-autonomous-payment-agents-using-agentgatepay-and-multi-chain-tokens-11873


# Create autonomous payment agents using AgentGatePay and multi-chain tokens

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow name:** 🤖 AgentGatePay - Buyer Agent MCP [TEMPLATE]  
**Provided title:** Create autonomous payment agents using AgentGatePay and multi-chain tokens

**Purpose:**  
This workflow implements a “buyer agent” that programmatically purchases a protected seller resource using **AgentGatePay** mandates (spending authorization) and an on-chain payment flow. It **reuses an existing mandate token** stored in an n8n **Data Table** to track budget across runs, and requires **manual renewal** when the mandate expires or is depleted.

**Target use cases**
- Autonomous agents purchasing API resources (reports, datasets, premium endpoints) with controlled spending.
- Repeatable purchases with budget tracking via mandate token reuse.
- Multi-chain token payments (chain/token values are passed through; example uses Base + USDC).

### Logical blocks
**1.1 Execution start & configuration**
- Loads all runtime configuration (buyer identity, seller endpoint, AgentGatePay endpoints, blockchain settings, Render signing service URL).

**1.2 Mandate storage lookup & normalization**
- Reads the Data Table `AgentPay_Mandates` and normalizes output so downstream nodes always receive predictable fields.

**1.3 Mandate decision & verification**
- If a token exists: verify validity with AgentGatePay.
- If invalid: stop with explicit manual renewal steps (delete stored token row, rerun).

**1.4 New mandate issuance, verification, and persistence**
- If no token exists: issue a new mandate through AgentGatePay MCP, verify it, validate token format, then store it in the Data Table.

**1.5 Purchase & payment execution**
- Request seller resource; expect **HTTP 402 Payment Required** with payment instructions.
- Call an external signing service (Render) to execute/sign payment transactions.
- Extract transaction hashes as proof.

**1.6 Submit proof & receive resource**
- Submit payment proof to AgentGatePay.
- Retrieve the paid resource from seller.
- Produce a completion summary including updated remaining budget.

---

## 2. Block-by-Block Analysis

### 2.1 Execution start & configuration

**Overview:** Initializes the workflow and produces a single configuration object (`config`) plus a `session` object used throughout the run.

**Nodes involved:** ▶️ START, 1️⃣ Load Config

#### Node: ▶️ START
- **Type / role:** Manual Trigger (`n8n-nodes-base.manualTrigger`) — entry point for interactive runs.
- **Configuration choices:** No parameters; starts workflow on demand.
- **Inputs/Outputs:**  
  - Input: none  
  - Output → `1️⃣ Load Config`
- **Edge cases:** None (manual start).

#### Node: 1️⃣ Load Config
- **Type / role:** Code (`n8n-nodes-base.code`) — builds the full config and initial session record.
- **Configuration choices (interpreted):**
  - Defines `CONFIG` with sections:
    - `buyer`: name/company/email/task, `api_key`, budget, TTL, scope.
    - `seller`: seller identity and resource API URL, `selected_resource_id`.
    - `render`: external service base URL for signing payments.
    - `blockchain`: chain/token/RPC URL (passed through; RPC not directly used in this workflow).
    - `agentgatepay`: API base and `mcp_endpoint`.
  - Outputs `{ config: CONFIG, session: { id, started_at, agent } }`.
- **Key variables/expressions:**
  - `session_${Date.now()}` for run ID.
  - `new Date().toISOString()` for timestamps.
- **Outputs:**  
  - Output → `2️⃣ 📊 Get Mandate Token`
- **Edge cases / failures:**
  - Misconfigured placeholders (email, API key, seller webhook URL, Render URL) will break later HTTP calls.
- **Version notes:** Code node v2 syntax supports `$()` node references used later.

---

### 2.2 Mandate storage lookup & normalization

**Overview:** Reads the saved mandate token (if any) from an n8n Data Table and normalizes output so the next decision node always sees `mandate_token` as a string.

**Nodes involved:** 2️⃣ 📊 Get Mandate Token, 2B️⃣ Normalize Result

#### Node: 2️⃣ 📊 Get Mandate Token
- **Type / role:** Data Table (`n8n-nodes-base.dataTable`) — fetch stored mandate row(s).
- **Configuration choices (interpreted):**
  - Operation: **get**
  - Return: **all rows** (`returnAll: true`)
  - **continueOnFail: true** and **alwaysOutputData: true** to keep flow alive even if empty or misconfigured.
  - Must point to Data Table: `AgentPay_Mandates`.
- **Inputs/Outputs:**
  - Input ← `1️⃣ Load Config`
  - Output → `2B️⃣ Normalize Result`
- **Edge cases / failures:**
  - Table not selected / wrong table: may error or return unexpected schema.
  - Multiple rows: downstream normalization warns and uses first.
  - Empty table: passes through to normalization.
- **Version notes:** Data Table node v1.

#### Node: 2B️⃣ Normalize Result
- **Type / role:** Code — ensures consistent downstream schema.
- **Configuration choices (interpreted):**
  - If input has **0 rows**, returns `{ mandate_token: '' , config }` (empty string, not null).
  - If rows exist, passes first row merged with `config`.
- **Key variables/expressions:**
  - `const config = $('1️⃣ Load Config').first().json.config;`
  - `$input.all()` to detect empty result.
- **Inputs/Outputs:**
  - Input ← `2️⃣ 📊 Get Mandate Token`
  - Output → `3️⃣ Has Token?`
- **Edge cases / failures:**
  - If Data Table returns items without `mandate_token`, IF check may route incorrectly (treated empty).
  - If table returns non-string token values, IF node still checks `notEmpty` but later verification may fail.

---

### 2.3 Mandate decision & verification (existing token path)

**Overview:** If a token exists, the workflow verifies it with AgentGatePay MCP. If invalid/expired, it halts with explicit manual renewal steps (security by design: no auto-renew).

**Nodes involved:** 3️⃣ Has Token?, 4️⃣ ✅ Verify Existing Token, 4B️⃣ Check Verification

#### Node: 3️⃣ Has Token?
- **Type / role:** IF (`n8n-nodes-base.if`) — routes based on presence of `mandate_token`.
- **Configuration choices (interpreted):**
  - Condition: `{{$json.mandate_token}}` **is not empty**.
  - **True** → verify existing token.
  - **False** → create new mandate.
- **Inputs/Outputs:**
  - Input ← `2B️⃣ Normalize Result`
  - True → `4️⃣ ✅ Verify Existing Token`
  - False → `5️⃣ 🆕 Create New Mandate`
- **Edge cases / failures:**
  - Token exists but is whitespace/invalid format: still “notEmpty”, verification will fail downstream.

#### Node: 4️⃣ ✅ Verify Existing Token
- **Type / role:** HTTP Request (`n8n-nodes-base.httpRequest`) — calls AgentGatePay MCP JSON-RPC tool `agentpay_verify_mandate`.
- **Configuration choices (interpreted):**
  - POST to `config.agentgatepay.mcp_endpoint`
  - JSON body: JSON-RPC `tools/call` with:
    - `name`: `agentpay_verify_mandate`
    - `arguments.mandate_token`: `{{$json.mandate_token}}`
  - Headers:
    - `Content-Type: application/json`
    - `x-api-key`: `config.buyer.api_key`
- **Inputs/Outputs:**
  - Input ← `3️⃣ Has Token?` (true branch)
  - Output → `4B️⃣ Check Verification`
- **Potential failures:**
  - 401/403 if API key invalid.
  - Network errors/timeouts to MCP endpoint.
  - MCP response not matching expected structure (`result.content[0].text`).
- **Version notes:** HTTP Request node v4.2.

#### Node: 4B️⃣ Check Verification
- **Type / role:** Code — parses MCP response and enforces manual renewal policy.
- **Configuration choices (interpreted):**
  - Expects `verify_response.result.content[0].text` JSON string; parses to `verification`.
  - If `verification.valid` is false:
    - Throws an error containing step-by-step instructions to delete the stored token row from `AgentPay_Mandates` and rerun.
  - If valid:
    - Constructs `mandate` object from stored token + verification metadata and outputs a unified payload for later merge.
- **Key variables/expressions:**
  - `const stored_token = $('2️⃣ 📊 Get Mandate Token').first().json.mandate_token;`
  - Output includes:
    - `was_reused: true`
    - `source: 'existing_token'`
- **Inputs/Outputs:**
  - Input ← `4️⃣ ✅ Verify Existing Token`
  - Output → `8️⃣ 🔀 Merge Paths`
- **Edge cases / failures:**
  - JSON parsing errors if MCP returns non-JSON text.
  - If Data Table had multiple rows, it always uses `.first()` token (may not be the newest).

---

### 2.4 New mandate issuance, verification, validation & persistence (new token path)

**Overview:** If no token exists, the workflow issues a new mandate, verifies it, performs strict token validation (JWT-like), then stores it in the Data Table and restores the original structured payload for the main flow.

**Nodes involved:** 5️⃣ 🆕 Create New Mandate, 5B️⃣ Parse New Mandate, 6️⃣ ✅ Verify New Mandate, 6B️⃣ Check New Mandate, 7️⃣ 💾 Insert Token, 7B️⃣ Restore Data

#### Node: 5️⃣ 🆕 Create New Mandate
- **Type / role:** HTTP Request — calls AgentGatePay MCP tool `agentpay_issue_mandate`.
- **Configuration choices (interpreted):**
  - POST to MCP endpoint.
  - JSON-RPC tool call `agentpay_issue_mandate` with:
    - `subject`: buyer email
    - `budget_usd`: `config.buyer.budget_usd`
    - `scope`: `config.buyer.mandate_scope`
    - `ttl_minutes`: `config.buyer.mandate_ttl_days * 1440`
  - Headers: `x-api-key` and JSON content-type.
- **Inputs/Outputs:**
  - Input ← `3️⃣ Has Token?` (false branch)
  - Output → `5B️⃣ Parse New Mandate`
- **Potential failures:**
  - Auth errors, invalid scope, budget limits rejected by API.
  - TTL conversion issues if `mandate_ttl_days` is not numeric.

#### Node: 5B️⃣ Parse New Mandate
- **Type / role:** Code — extracts `mandate_token` from MCP response.
- **Configuration choices (interpreted):**
  - Parses `result.content[0].text` as JSON.
  - Requires `result.mandate_token`.
  - Builds normalized payload containing:
    - `mandate: { token, id, budget_remaining, expires_at }`
    - `verification: { valid: true, budget_remaining, status: 'active' }` (local placeholder; real verification happens next)
    - `was_reused: false`
- **Inputs/Outputs:**
  - Input ← `5️⃣ 🆕 Create New Mandate`
  - Output → `6️⃣ ✅ Verify New Mandate`
- **Edge cases / failures:**
  - MCP returns malformed/partial content.
  - `budget_remaining` may be string; code coerces via `parseFloat`.

#### Node: 6️⃣ ✅ Verify New Mandate
- **Type / role:** HTTP Request — verifies newly issued token with `agentpay_verify_mandate`.
- **Configuration choices:**
  - Same MCP endpoint and headers as other verification.
  - Uses `{{$json.mandate.token}}`.
- **Inputs/Outputs:**
  - Input ← `5B️⃣ Parse New Mandate`
  - Output → `6B️⃣ Check New Mandate`
- **Potential failures:** Same as existing-token verification.

#### Node: 6B️⃣ Check New Mandate
- **Type / role:** Code — validates verification response and performs strict token sanity checks before saving.
- **Configuration choices (interpreted):**
  - Parses MCP verification; must be `valid: true`.
  - Validates token:
    1. Exists and is a string
    2. Not equal to `'null'`, `'undefined'`, or `''`
    3. Length ≥ 50
    4. Contains `.` (JWT-like structure)
  - Outputs:
    - `mandate_token` (string) for Data Table insert
    - `_original_data` containing the full structured payload from 5B (used later)
- **Inputs/Outputs:**
  - Input ← `6️⃣ ✅ Verify New Mandate`
  - Output → `7️⃣ 💾 Insert Token`
- **Edge cases / failures:**
  - False positives: a non-JWT token format would be blocked even if AgentGatePay accepts it.
  - JSON parsing errors if MCP returns unexpected text.

#### Node: 7️⃣ 💾 Insert Token
- **Type / role:** Data Table — persists the verified token.
- **Configuration choices (interpreted):**
  - Operation: insert (via column mapping)
  - Column mapping: `mandate_token = {{$json.mandate_token}}`
  - Must target the Data Table: `AgentPay_Mandates` (single string column `mandate_token`).
- **Inputs/Outputs:**
  - Input ← `6B️⃣ Check New Mandate`
  - Output → `7B️⃣ Restore Data`
- **Edge cases / failures:**
  - Table not selected or schema mismatch (missing `mandate_token` column).
  - Multiple inserts across runs if user doesn’t delete old rows (workflow assumes single-row semantics).

#### Node: 7B️⃣ Restore Data
- **Type / role:** Code — restores the payload structure lost by Data Table insert response.
- **Configuration choices (interpreted):**
  - Ignores Data Table response and returns `_original_data` from node `6B️⃣ Check New Mandate`.
- **Inputs/Outputs:**
  - Input ← `7️⃣ 💾 Insert Token`
  - Output → `8️⃣ 🔀 Merge Paths`
- **Edge cases / failures:**
  - If `6B` output missing `_original_data`, merge will break.

---

### 2.5 Purchase & payment execution

**Overview:** Both mandate paths converge. The workflow requests the resource, expects a 402 with payment details, calls an external signing service to execute the payment, then prepares payment proof.

**Nodes involved:** 8️⃣ 🔀 Merge Paths, 9️⃣ Request Resource (x402), 9B️⃣ Parse 402 Response, 🔟 🔒 Sign Payment (Render), 1️⃣1️⃣ Extract TX Hashes

#### Node: 8️⃣ 🔀 Merge Paths
- **Type / role:** Code — unifies both branches into one consistent payload.
- **Configuration choices (interpreted):**
  - Uses incoming branch data + `session` from `1️⃣ Load Config`.
  - Defines `selected_resource` (hardcoded title/description; ID from config).
  - Builds `resource_request` with mandate token, sender/receiver emails, timestamp.
- **Inputs/Outputs:**
  - Input ← `4B️⃣ Check Verification` OR `7B️⃣ Restore Data`
  - Output → `9️⃣ Request Resource (x402)`
- **Edge cases / failures:**
  - If seller config is wrong, next HTTP request fails.
  - Selected resource metadata is partially hardcoded; mismatch with seller’s real catalog possible.

#### Node: 9️⃣ Request Resource (x402)
- **Type / role:** HTTP Request — calls seller resource endpoint expecting payment required.
- **Configuration choices (interpreted):**
  - URL: `{{$json.config.seller.api_url}}/resource/{{$json.config.seller.selected_resource_id}}`
  - Headers:
    - `x-agent-id`: buyer email
    - `x-mandate`: mandate token
  - Response options:
    - `neverError: true` and `fullResponse: true` so 402 does not stop execution.
- **Inputs/Outputs:**
  - Input ← `8️⃣ 🔀 Merge Paths`
  - Output → `9B️⃣ Parse 402 Response`
- **Edge cases / failures:**
  - Seller may return 200 (free) or 401/403/404; node won’t throw automatically due to `neverError`, but the next node expects 402 and will throw.
  - Seller response body format might not match expected payment schema.

#### Node: 9B️⃣ Parse 402 Response
- **Type / role:** Code — validates that seller responded with 402 and extracts payment instructions.
- **Configuration choices (interpreted):**
  - Requires `statusCode === 402` (from fullResponse) or body `statusCode === 402`.
  - Stores payment payload under `payment_required`.
- **Inputs/Outputs:**
  - Input ← `9️⃣ Request Resource (x402)`
  - Output → `🔟 🔒 Sign Payment (Render)`
- **Edge cases / failures:**
  - Throws if not 402.
  - If seller returns 402 but missing fields (e.g., `payTo`, `priceUsd`, `token`, `chain`), later nodes may fail.

#### Node: 🔟 🔒 Sign Payment (Render)
- **Type / role:** HTTP Request — calls external service to sign/execute payment transactions.
- **Configuration choices (interpreted):**
  - POST to: `config.render.service_url + /sign-payment`
  - JSON body includes:
    - `merchant_address`: `payment_required.payTo`
    - `total_amount`: `payment_required.amount`
    - `token`, `chain`
  - Headers:
    - `x-api-key`: buyer API key (note: uses AgentGatePay API key as provided)
- **Inputs/Outputs:**
  - Input ← `9B️⃣ Parse 402 Response`
  - Output → `1️⃣1️⃣ Extract TX Hashes`
- **Edge cases / failures:**
  - Render service might require a different API key than AgentGatePay (template uses same field).
  - `payment_required.amount` vs `payment_required.priceUsd`: amount precision and denomination must match what Render expects.
  - If Render signs but transactions not mined/confirmed yet, later “confirmed” status is optimistic.

#### Node: 1️⃣1️⃣ Extract TX Hashes
- **Type / role:** Code — validates Render response and constructs `payment_proof`.
- **Configuration choices (interpreted):**
  - Requires:
    - `render_response.success === true`
    - `tx_hash` and `tx_hash_commission`
  - Builds `payment_proof` including:
    - tx hashes, from address, commission address, USD splits
    - Explorer URLs hardcoded to BaseScan:
      - `https://basescan.org/tx/{hash}`
- **Inputs/Outputs:**
  - Input ← `🔟 🔒 Sign Payment (Render)`
  - Output → `1️⃣2️⃣ Submit Payment (MCP)`
- **Edge cases / failures:**
  - If chain is not Base, the explorer URLs will be wrong (template assumes BaseScan).
  - If Render returns different field names, extraction fails.

---

### 2.6 Submit proof, receive resource, and complete

**Overview:** Submits on-chain payment proof to AgentGatePay, then calls the seller again with `x-payment` to fetch the paid resource, and finally outputs a completion summary.

**Nodes involved:** 1️⃣2️⃣ Submit Payment (MCP), 1️⃣3️⃣ Receive Resource, 1️⃣4️⃣ Complete Task

#### Node: 1️⃣2️⃣ Submit Payment (MCP)
- **Type / role:** HTTP Request — submits payment proof to AgentGatePay MCP tool `agentpay_submit_payment`.
- **Configuration choices (interpreted):**
  - JSON-RPC `tools/call`:
    - `mandate_token`, `tx_hash`, `tx_hash_commission`
    - `chain`, `token`
    - `price_usd` from `payment_required.priceUsd`
  - Header `x-api-key` from buyer config.
- **Inputs/Outputs:**
  - Input ← `1️⃣1️⃣ Extract TX Hashes`
  - Output → `1️⃣3️⃣ Receive Resource`
- **Edge cases / failures:**
  - If AgentGatePay requires confirmation depth, submission may fail if tx not finalized.
  - Mismatch between submitted `chain/token/price_usd` and seller’s quote may be rejected.

#### Node: 1️⃣3️⃣ Receive Resource
- **Type / role:** HTTP Request — retrieves the now-paid seller resource.
- **Configuration choices (interpreted):**
  - URL: seller resource endpoint again.
  - Header: `x-payment: {tx_hash}` (uses merchant payment tx hash only).
  - Response: `fullResponse: true`, JSON format.
- **Inputs/Outputs:**
  - Input ← `1️⃣2️⃣ Submit Payment (MCP)`
  - Output → `1️⃣4️⃣ Complete Task`
- **Edge cases / failures:**
  - Seller may require different proof header(s) or both hashes.
  - If seller uses async fulfillment, resource might not be immediately available.

#### Node: 1️⃣4️⃣ Complete Task
- **Type / role:** Code — produces final structured output and user-facing summary.
- **Configuration choices (interpreted):**
  - Extracts `resource_data` from HTTP response body.
  - Computes:
    - `spent = parseFloat(payment_required.priceUsd)`
    - `budget_after = budget_before - spent` (uses verification budget_remaining as “before”)
  - Outputs `summary` and a detailed `message`.
- **Inputs/Outputs:**
  - Input ← `1️⃣3️⃣ Receive Resource`
  - Output: terminal (no downstream nodes)
- **Edge cases / failures:**
  - If `priceUsd` is not numeric, `spent` becomes NaN and budget math breaks.
  - Budget “before” is taken from verification at start of run; may differ from real-time budget after concurrent spending.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| ▶️ START | Manual Trigger | Manual entry point | — | 1️⃣ Load Config | # Buyer Agent Workflow / Quick setup + link: https://github.com/AgentGatePay/agentgatepay-examples/tree/main/n8n |
| 1️⃣ Load Config | Code | Build config + session payload | ▶️ START | 2️⃣ 📊 Get Mandate Token | # Buyer Agent Workflow / Quick setup + link: https://github.com/AgentGatePay/agentgatepay-examples/tree/main/n8n ; ## Mandate Management |
| 2️⃣ 📊 Get Mandate Token | Data Table | Retrieve stored mandate token(s) | 1️⃣ Load Config | 2B️⃣ Normalize Result | ## Mandate Management |
| 2B️⃣ Normalize Result | Code | Normalize empty/multi-row table output | 2️⃣ 📊 Get Mandate Token | 3️⃣ Has Token? | ## Mandate Management |
| 3️⃣ Has Token? | IF | Route: reuse token vs create new mandate | 2B️⃣ Normalize Result | 4️⃣ ✅ Verify Existing Token; 5️⃣ 🆕 Create New Mandate | ## Mandate Management |
| 4️⃣ ✅ Verify Existing Token | HTTP Request | Verify existing mandate via AgentGatePay MCP | 3️⃣ Has Token? (true) | 4B️⃣ Check Verification | ## Token Verification |
| 4B️⃣ Check Verification | Code | Parse verification; error with renewal steps if invalid | 4️⃣ ✅ Verify Existing Token | 8️⃣ 🔀 Merge Paths | ## Token Verification |
| 5️⃣ 🆕 Create New Mandate | HTTP Request | Issue new mandate via AgentGatePay MCP | 3️⃣ Has Token? (false) | 5B️⃣ Parse New Mandate | ## New Mandate Creation |
| 5B️⃣ Parse New Mandate | Code | Extract token/id/budget/expiry from response | 5️⃣ 🆕 Create New Mandate | 6️⃣ ✅ Verify New Mandate | ## New Mandate Creation |
| 6️⃣ ✅ Verify New Mandate | HTTP Request | Verify newly created mandate token | 5B️⃣ Parse New Mandate | 6B️⃣ Check New Mandate | ## New Mandate Creation |
| 6B️⃣ Check New Mandate | Code | Validate verification + strict JWT-like token checks | 6️⃣ ✅ Verify New Mandate | 7️⃣ 💾 Insert Token | ## New Mandate Creation |
| 7️⃣ 💾 Insert Token | Data Table | Persist mandate_token to AgentPay_Mandates | 6B️⃣ Check New Mandate | 7B️⃣ Restore Data | ## New Mandate Creation |
| 7B️⃣ Restore Data | Code | Restore original structured payload after insert | 7️⃣ 💾 Insert Token | 8️⃣ 🔀 Merge Paths | ## New Mandate Creation |
| 8️⃣ 🔀 Merge Paths | Code | Unify old/new mandate paths and build request context | 4B️⃣ Check Verification; 7B️⃣ Restore Data | 9️⃣ Request Resource (x402) | ## Payment Flow |
| 9️⃣ Request Resource (x402) | HTTP Request | Call seller resource endpoint; expect 402 | 8️⃣ 🔀 Merge Paths | 9B️⃣ Parse 402 Response | ## Payment Flow |
| 9B️⃣ Parse 402 Response | Code | Enforce 402 and store payment instructions | 9️⃣ Request Resource (x402) | 🔟 🔒 Sign Payment (Render) | ## Payment Flow |
| 🔟 🔒 Sign Payment (Render) | HTTP Request | Ask external signing service to pay merchant + commission | 9B️⃣ Parse 402 Response | 1️⃣1️⃣ Extract TX Hashes | ## Payment Flow |
| 1️⃣1️⃣ Extract TX Hashes | Code | Validate signing response; build payment_proof + explorer links | 🔟 🔒 Sign Payment (Render) | 1️⃣2️⃣ Submit Payment (MCP) | ## Payment Flow |
| 1️⃣2️⃣ Submit Payment (MCP) | HTTP Request | Submit payment proof to AgentGatePay MCP | 1️⃣1️⃣ Extract TX Hashes | 1️⃣3️⃣ Receive Resource | ## Resource Delivery |
| 1️⃣3️⃣ Receive Resource | HTTP Request | Retrieve paid resource from seller using tx hash proof | 1️⃣2️⃣ Submit Payment (MCP) | 1️⃣4️⃣ Complete Task | ## Resource Delivery |
| 1️⃣4️⃣ Complete Task | Code | Compute budget deltas, package final output/summary | 1️⃣3️⃣ Receive Resource | — | ## Resource Delivery |

---

## 4. Reproducing the Workflow from Scratch

1) **Create the Data Table**
   1. In n8n, go to **Data → Data Tables**.
   2. Create a table named **`AgentPay_Mandates`**.
   3. Add **one column**:
      - Name: `mandate_token`
      - Type: **String**

2) **Create nodes (in order) and configure**

3) **▶️ START**
   - Add node: **Manual Trigger**
   - No configuration needed.

4) **1️⃣ Load Config (Code)**
   - Add node: **Code**
   - Paste code that defines a `CONFIG` object with:
     - Buyer: `email`, `api_key`, `budget_usd`, `mandate_ttl_days`, `mandate_scope`, etc.
     - Seller: `api_url`, `selected_resource_id`, etc.
     - Render: `service_url`
     - AgentGatePay: `mcp_endpoint` (e.g., `https://mcp.agentgatepay.com`)
   - Return one item with `{ config: CONFIG, session: {...} }`.

5) **2️⃣ 📊 Get Mandate Token (Data Table)**
   - Add node: **Data Table**
   - Operation: **Get**
   - Return all: **true**
   - Enable:
     - **Continue On Fail**
     - **Always Output Data**
   - In the Data Table dropdown, select **`AgentPay_Mandates`**.

6) **2B️⃣ Normalize Result (Code)**
   - Add node: **Code**
   - Logic:
     - If `$input.all().length === 0`, output `{ mandate_token: '' , config }`
     - Else output `{ ...firstRow, config }`
   - Reference config using `$('1️⃣ Load Config').first().json.config`.

7) **3️⃣ Has Token? (IF)**
   - Add node: **IF**
   - Condition:
     - Left value: `={{ $json.mandate_token }}`
     - Operator: **notEmpty**
   - True branch: existing token flow
   - False branch: new token flow

8) **4️⃣ ✅ Verify Existing Token (HTTP Request)**
   - Add node: **HTTP Request**
   - Method: POST
   - URL: `={{ $('1️⃣ Load Config').first().json.config.agentgatepay.mcp_endpoint }}`
   - Send JSON body (JSON-RPC `tools/call`) for `agentpay_verify_mandate` with `mandate_token`.
   - Headers:
     - `Content-Type: application/json`
     - `x-api-key: ={{ $('1️⃣ Load Config').first().json.config.buyer.api_key }}`

9) **4B️⃣ Check Verification (Code)**
   - Add node: **Code**
   - Parse MCP response from `result.content[0].text` as JSON.
   - If invalid: `throw new Error(...)` with manual renewal steps.
   - If valid: output unified structure:
     - `config`, `mandate: { token }`, `verification`, `was_reused: true`

10) **5️⃣ 🆕 Create New Mandate (HTTP Request)**
   - Add node: **HTTP Request** on IF false branch
   - POST to MCP endpoint with JSON-RPC for `agentpay_issue_mandate`
   - Arguments: `subject`, `budget_usd`, `scope`, `ttl_minutes = mandate_ttl_days*1440`
   - Same headers as above.

11) **5B️⃣ Parse New Mandate (Code)**
   - Add node: **Code**
   - Extract `mandate_token`, `mandate_id`, `expires_at`, `budget_remaining`.
   - Output unified structure with `was_reused: false`.

12) **6️⃣ ✅ Verify New Mandate (HTTP Request)**
   - Add node: **HTTP Request**
   - POST MCP `agentpay_verify_mandate` using `{{$json.mandate.token}}`.

13) **6B️⃣ Check New Mandate (Code)**
   - Add node: **Code**
   - Require verification valid.
   - Enforce strict token checks (string, not empty, length, contains `.`).
   - Output `{ mandate_token, _original_data }`.

14) **7️⃣ 💾 Insert Token (Data Table)**
   - Add node: **Data Table**
   - Operation: **Insert** (map columns)
   - Select table **`AgentPay_Mandates`** (important: re-select in dropdown).
   - Map column: `mandate_token = {{$json.mandate_token}}`

15) **7B️⃣ Restore Data (Code)**
   - Add node: **Code**
   - Return `_original_data` from node `6B️⃣ Check New Mandate` so the main flow receives the unified payload.

16) **8️⃣ 🔀 Merge Paths (Code)**
   - Add node: **Code**
   - Input: either from `4B` (existing) or `7B` (new).
   - Attach `session` from `1️⃣ Load Config`.
   - Build `resource_request` and `selected_resource`.

17) **9️⃣ Request Resource (x402) (HTTP Request)**
   - Add node: **HTTP Request**
   - GET (default) to: `={{ $json.config.seller.api_url }}/resource/{{ $json.config.seller.selected_resource_id }}`
   - Headers:
     - `x-agent-id = {{$json.config.buyer.email}}`
     - `x-mandate = {{$json.mandate.token}}`
   - Response options:
     - **Full Response: true**
     - **Never Error: true** (so 402 continues)

18) **9B️⃣ Parse 402 Response (Code)**
   - Add node: **Code**
   - Throw unless HTTP status is **402**
   - Store payment payload under `payment_required`.

19) **🔟 🔒 Sign Payment (Render) (HTTP Request)**
   - Add node: **HTTP Request**
   - POST: `={{ $json.config.render.service_url }}/sign-payment`
   - JSON body: merchant address, total amount, token, chain.
   - Header: `x-api-key` (ensure your signing service expects this; template uses buyer api_key).

20) **1️⃣1️⃣ Extract TX Hashes (Code)**
   - Add node: **Code**
   - Require `success`, `tx_hash`, `tx_hash_commission`.
   - Build `payment_proof` and (optionally) explorer links.

21) **1️⃣2️⃣ Submit Payment (MCP) (HTTP Request)**
   - Add node: **HTTP Request**
   - POST to MCP endpoint calling `agentpay_submit_payment`
   - Include mandate token and both tx hashes, plus chain/token/priceUsd.

22) **1️⃣3️⃣ Receive Resource (HTTP Request)**
   - Add node: **HTTP Request**
   - GET seller resource URL again
   - Header: `x-payment = {merchant tx hash}`
   - Response: Full Response true, JSON.

23) **1️⃣4️⃣ Complete Task (Code)**
   - Add node: **Code**
   - Extract resource body, compute budget delta, output summary and final message.

24) **Wire connections**
   - START → Load Config → Get Mandate Token → Normalize Result → Has Token?
   - Has Token? true → Verify Existing Token → Check Verification → Merge Paths
   - Has Token? false → Create New Mandate → Parse → Verify New → Check New → Insert Token → Restore Data → Merge Paths
   - Merge Paths → Request Resource → Parse 402 → Sign Payment → Extract TX Hashes → Submit Payment → Receive Resource → Complete Task

25) **Credentials / secrets**
   - AgentGatePay: store `buyer.api_key` securely (prefer n8n credentials or environment variables rather than hardcoding in Code).
   - Seller API: may require additional authentication depending on implementation.
   - Render signing service: provide whatever API key it requires (template reuses buyer.api_key; adjust as needed).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Buyer Agent Workflow: creates/reuses mandate token stored in `AgentPay_Mandates`, budget tracked across runs, manual renewal by deleting row | Sticky note “START HERE” |
| External examples and context | https://github.com/AgentGatePay/agentgatepay-examples/tree/main/n8n |
| Explorer links are hardcoded to BaseScan (`https://basescan.org/tx/{hash}`) | Node 1️⃣1️⃣ Extract TX Hashes (adjust for other chains) |
| Seller webhook URL and Render URL are placeholders and must be replaced | Node 1️⃣ Load Config (`seller.api_url`, `render.service_url`) |
| Security posture: mandate expiry/depletion is not auto-renewed; user must manually delete stored token row to create a new mandate | Node 4B️⃣ Check Verification message/design |