Create a cryptocurrency-powered API for selling resources with AgentGatePay

https://n8nworkflows.xyz/workflows/create-a-cryptocurrency-powered-api-for-selling-resources-with-agentgatepay-11874


# Create a cryptocurrency-powered API for selling resources with AgentGatePay

## 1. Workflow Overview

**Purpose:** Provide a public GET API endpoint that “sells” a digital resource to AI agents. If the buyer has not paid, it returns an **HTTP 402 Payment Required** response (x402-style) with payment instructions. If the buyer includes a payment transaction hash in the request headers, the workflow verifies the payment via **AgentGatePay** and then delivers the paid resource.

**Target use cases:**
- Monetizing downloadable datasets/reports/API responses for agent-to-agent commerce
- Paywalled endpoints where the response body is the purchased content
- Lightweight payment verification gate in front of digital delivery

### 1.1 Input Reception & Request Parsing
Receives the GET request, loads seller configuration (wallet + API key), extracts resourceId and headers, and decides whether to request payment (402) or verify an included payment.

### 1.2 Payment Required Path (402)
If no payment tx hash is provided, generates a structured 402 response containing token/chain, recipient wallet, amount, nonce, and retry instructions.

### 1.3 Payment Verification & Validation
If a payment tx hash is provided, calls AgentGatePay verification endpoint, validates recipient and amount, then routes to delivery or error.

### 1.4 Resource Delivery & Response Handling
On validated payment, returns the full resource payload with HTTP 200. On failure, returns an error body (typically HTTP 400/404/500 depending on where it failed).

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception & Request Parsing
**Overview:** Exposes the public endpoint and normalizes inputs (path/query, headers). Loads the seller catalog and routes execution based on whether `x-payment` header exists.

**Nodes involved:**
- `📡 GET /resource/{id}`
- `1️⃣ Parse Request`
- `2️⃣ Has Payment?`

#### Node: 📡 GET /resource/{id}
- **Type / role:** Webhook node (`n8n-nodes-base.webhook`) acting as the public HTTP entrypoint.
- **Key configuration:**
  - **Path:** `resource/:resourceId` (captures `resourceId` as a path param)
  - **Response mode:** `responseNode` (workflow must end with Respond to Webhook nodes)
  - **rawBody:** false
- **Inputs / outputs:**
  - **Output:** to `1️⃣ Parse Request`
- **Version notes:** `typeVersion 1.1`; behavior depends on n8n webhook handling but standard.
- **Edge cases / failures:**
  - Workflow not active → endpoint not reachable
  - Method mismatch (this template implies GET; if caller uses POST, depends on webhook default settings in n8n instance)
  - Missing `resourceId` in path → parse node may fall back to query param, else “Resource not found”

#### Node: 1️⃣ Parse Request
- **Type / role:** Code node (`n8n-nodes-base.code`) performing configuration + request parsing + routing decision.
- **Key configuration choices (interpreted):**
  - Defines `SELLER_CONFIG` with:
    - `merchant`: name/company/email, **wallet_address**, **api_key** (intended to be edited)
    - `payment`: token + chain (defaults `USDC` on `base`)
    - `catalog`: dictionary keyed by resource id, with `title`, `price_usd`, `preview`
  - Validates configuration (but see “edge cases” below).
  - Extracts:
    - `resource_id` from `request.params.resourceId` or `request.query.resourceId`
    - `x-payment` header (case variants) → `payment_tx_hash`
    - `x-agent-id` header (optional)
  - Looks up the resource in `SELLER_CONFIG.catalog`
  - Sets routing field:
    - `route = "verify_payment"` if payment tx hash exists
    - else `route = "return_402"`
- **Key variables / fields produced:**
  - `$json.config`, `$json.resource`, `$json.request.{resource_id, agent_id, payment_tx_hash, has_payment}`, `$json.route`
- **Inputs / outputs:**
  - **Input:** webhook payload (params/query/headers)
  - **Output:** to `2️⃣ Has Payment?` and also connected to `9️⃣ Send Error` (see below)
- **Important connection note (potential bug):**
  - This node is connected directly to `9️⃣ Send Error` in parallel with `2️⃣ Has Payment?`. Since `9️⃣ Send Error` is a Respond to Webhook node, it can prematurely respond (or cause “response already sent” conflicts) unless it’s never executed due to execution constraints. In most cases, **this is unsafe** and should be removed or gated by an IF route.
- **Edge cases / failures:**
  - **Misleading validation constants:** The code checks:
    - `wallet_address === "0x742d35Cc6634C0532925a3b844Bc9e7595f0bEbB"`
    - `api_key === "YOUR_AGENTPAY_API_KEY"`
    - But the placeholders set in the config are `"YOUR_WALLET_ADDRESS"` and `"YOUR AGENTGATEPAY API KEY"`, so validation may **not** catch an unedited template (and will proceed with invalid config).
  - Missing/unknown `resourceId` → returns a `route: "error_404"` object with `http_response.statusCode = 404`
  - Header case differences handled for `x-payment` and `x-agent-id`
  - If upstream webhook data structure differs (n8n version/setting), `request.params` may be undefined; the code already guards with `?.`

#### Node: 2️⃣ Has Payment?
- **Type / role:** IF node (`n8n-nodes-base.if`) routes based on `route`.
- **Condition:** `$json.route == "verify_payment"`
  - **True:** to `5️⃣ Verify Payment`
  - **False:** to `3️⃣ Generate 402`
- **Edge cases / failures:**
  - If `1️⃣ Parse Request` returns `route: "error_404"` or `route: "error_config"`, this IF will treat it as “no payment” path and send 402 unless separately handled. (In current design, errors were intended to go to `9️⃣ Send Error`, but the wiring is problematic.)

---

### Block 2 — Payment Required Path (402)
**Overview:** When no payment is included, returns a structured 402 response telling the client where/how much to pay and how to retry.

**Nodes involved:**
- `3️⃣ Generate 402`
- `4️⃣ Send 402`

#### Node: 3️⃣ Generate 402
- **Type / role:** Code node generating the “Payment Required” response body.
- **Key configuration / logic:**
  - Reads `config` and `resource` from incoming JSON
  - Determines token decimals:
    - `DAI` → 18
    - else → 6 (works for USDC; may be wrong for other tokens)
  - Computes integer amount in smallest units:
    - `amount = floor(price_usd * 10^decimals)`
  - Creates a random nonce
  - Constructs `response_402` with:
    - `statusCode: 402`, `protocol: "x402"`
    - `payTo`, `amount`, `token`, `chain`, `priceUsd`, `nonce`
    - `resource` preview + merchant info + instructions
- **Outputs:** `{ http_response: response_402 }` to `4️⃣ Send 402`
- **Edge cases / failures:**
  - Floating point rounding: `Math.floor(price_usd * 10^decimals)` can undercharge for certain values; consider using precise decimal math.
  - Token decimals assumption may break for non-USDC/non-DAI tokens.

#### Node: 4️⃣ Send 402
- **Type / role:** Respond to Webhook node returning the JSON body.
- **Key configuration:**
  - Respond with: JSON
  - Response body: `={{ JSON.stringify($json.http_response) }}`
- **Edge cases / failures:**
  - Body is JSON-stringified; depending on n8n settings, this may return a JSON string instead of an object. Typically you’d pass the object directly rather than `JSON.stringify`. If clients expect JSON object, adjust to return `$json.http_response` without stringifying.

---

### Block 3 — Payment Verification & Validation
**Overview:** Verifies the transaction hash with AgentGatePay and checks recipient + amount against the configured resource price, then routes success vs error.

**Nodes involved:**
- `5️⃣ Verify Payment`
- `6️⃣ Validate Payment`
- `6B️⃣ Route: Valid?`

#### Node: 5️⃣ Verify Payment
- **Type / role:** HTTP Request node calling AgentGatePay verification API.
- **Key configuration:**
  - **URL:** `https://api.agentgatepay.com/v1/payments/verify/{{ payment_tx_hash }}`
    - Pulls tx hash from: `$('1️⃣ Parse Request').first().json.request.payment_tx_hash`
  - Sends header `x-api-key` from: `$('1️⃣ Parse Request').first().json.config.merchant.api_key`
- **Inputs / outputs:**
  - Input: from `2️⃣ Has Payment?` (true path)
  - Output: to `6️⃣ Validate Payment`
- **Version notes:** `typeVersion 4.2` (newer HTTP node behavior; options differ across versions).
- **Edge cases / failures:**
  - Invalid/expired API key → 401/403
  - Invalid tx hash or not found → 404/400 (depends on AgentGatePay API behavior)
  - Network timeouts / rate limits
  - If AgentGatePay returns a schema different from what `6️⃣ Validate Payment` expects, validation may fail or throw (e.g., missing `to_address`)

#### Node: 6️⃣ Validate Payment
- **Type / role:** Code node validating verification response vs expected merchant + resource price.
- **Key logic:**
  - Reads:
    - `request_data` from `1️⃣ Parse Request`
    - `verification` from current node input (AgentGatePay response)
  - Checks:
    1. `verification.verified` must be truthy
    2. recipient address matches `config.merchant.wallet_address` (case-insensitive)
    3. USD amount matches resource price within `< 0.01`
  - On failure: returns `{ route: "validation_failed", http_response: { statusCode: 400, ... } }`
  - On success: returns merged data plus `route: "validation_passed"` and includes `verification`
- **Sticky note warning vs actual code:**
  - Sticky note claims “TEST MODE ACTIVE” and “ALWAYS returns validation_passed”, but the code **does perform real checks**. Treat the sticky note as potentially outdated.
- **Edge cases / failures:**
  - Field naming mismatch: later delivery node uses `verification.amountUsd`, `verification.txHash`, `verification.sender`, while validation uses `verification.amount_usd` and `verification.to_address`. If AgentGatePay returns one style, one of the nodes may break.
  - If `verification.to_address` is undefined, `.toLowerCase()` throws. Add guards.
  - Amount comparison uses floating point; small rounding differences could fail.

#### Node: 6B️⃣ Route: Valid?
- **Type / role:** IF node routing by `route == "validation_passed"`.
  - True → `7️⃣ Deliver Resource`
  - False → `9️⃣ Send Error`
- **Edge cases / failures:**
  - If `6️⃣ Validate Payment` throws an exception, this node is never reached; workflow fails unless error handling is configured in n8n.

---

### Block 4 — Resource Delivery & Response Handling
**Overview:** Builds the final paid content response (example competitor dataset) and returns it as HTTP 200. Errors are returned via a dedicated error responder.

**Nodes involved:**
- `7️⃣ Deliver Resource`
- `8️⃣ Send 200 OK`
- `9️⃣ Send Error`

#### Node: 7️⃣ Deliver Resource
- **Type / role:** Code node generating the “delivered resource” payload.
- **Key logic:**
  - Reads `resource` and `verification`
  - Builds `full_data`:
    - metadata: `success`, `resource_id`, `title`, timestamps
    - `data`: example competitors list + market analysis
    - `payment`: includes tx hash, payer, amount USD
  - Returns `{ http_response: full_data }`
- **Potential field mismatches:**
  - Uses `verification.amountUsd`, `verification.txHash`, `verification.sender`
  - Validation node used `verification.amount_usd`, `verification.to_address`, etc.
  - Ensure consistency with actual AgentGatePay response schema.
- **Outputs:** to `8️⃣ Send 200 OK`

#### Node: 8️⃣ Send 200 OK
- **Type / role:** Respond to Webhook returning success payload.
- **Key configuration:**
  - Respond with JSON
  - Response body: `={{ JSON.stringify($json.http_response) }}`
- **Edge cases / failures:**
  - Same JSON-stringification concern as other Respond nodes (may return a JSON string instead of JSON object).

#### Node: 9️⃣ Send Error
- **Type / role:** Respond to Webhook returning an error payload.
- **Key configuration:**
  - Respond with JSON
  - Response body: `={{ JSON.stringify($json.http_response) }}`
- **Inputs / outputs:**
  - Receives from `6B️⃣ Route: Valid?` (false path)
  - Also (problematically) receives directly from `1️⃣ Parse Request`
- **Edge cases / failures:**
  - If executed in parallel with another Respond node, n8n may error (“response already sent”) or return the wrong response first.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| 📡 GET /resource/{id} | Webhook | Public GET endpoint / entrypoint | — | 1️⃣ Parse Request | PUBLIC ENDPOINT\n\n✅ NO EDITS NEEDED\n\nCopy webhook URL after activating! |
| 1️⃣ Parse Request | Code | Seller config + request parsing + routing | 📡 GET /resource/{id} | 2️⃣ Has Payment?, 9️⃣ Send Error | 🔧 EDIT THIS NODE!\n\nReplace 2 values:\n1. wallet_address\n2. api_key |
| 2️⃣ Has Payment? | IF | Branch: verify payment vs return 402 | 1️⃣ Parse Request | 5️⃣ Verify Payment (true), 3️⃣ Generate 402 (false) | ✅ NO EDITS NEEDED |
| 3️⃣ Generate 402 | Code | Build HTTP 402 Payment Required payload | 2️⃣ Has Payment? (false) | 4️⃣ Send 402 | ✅ NO EDITS NEEDED |
| 4️⃣ Send 402 | Respond to Webhook | Return 402 response | 3️⃣ Generate 402 | — | ✅ NO EDITS NEEDED |
| 5️⃣ Verify Payment | HTTP Request | Call AgentGatePay verify endpoint | 2️⃣ Has Payment? (true) | 6️⃣ Validate Payment | ✅ NO EDITS NEEDED |
| 6️⃣ Validate Payment | Code | Validate recipient + amount + verified flag | 5️⃣ Verify Payment | 6B️⃣ Route: Valid? | 🧪 TEST MODE ACTIVE!\n\nBypasses AgentGatePay verification.\nALWAYS returns validation_passed.\n\n⚠️ REMOVE BEFORE PRODUCTION! |
| 6B️⃣ Route: Valid? | IF | Route: deliver vs error | 6️⃣ Validate Payment | 7️⃣ Deliver Resource (true), 9️⃣ Send Error (false) | ✅ NEW v3.2:\nRoutes valid payments to resource delivery |
| 7️⃣ Deliver Resource | Code | Generate full paid resource response | 6B️⃣ Route: Valid? (true) | 8️⃣ Send 200 OK | ✅ v3.2:\nCreates full resource response |
| 8️⃣ Send 200 OK | Respond to Webhook | Return 200 OK payload | 7️⃣ Deliver Resource | — | ✅ Sends success response with body |
| 9️⃣ Send Error | Respond to Webhook | Return error payload | 6B️⃣ Route: Valid? (false), 1️⃣ Parse Request | — | ✅ Sends error response with body |
| START HERE | Sticky Note | Documentation / setup notes | — | — | # Seller Resource API\n\n**What it does:** Webhook that sells digital resources to AI agents. Verifies payments via AgentGatePay before delivering content.\n\n**Quick setup (3 min):**\n1. Edit Node 1: Add your wallet address and AgentGatePay API key\n2. Choose payment: coin and network\n3. Activate workflow (toggle top-right)\n4. Copy webhook URL from Node \"📡 GET /resource/{id}\"\n5. Share URL with buyer agents\n\n**Payment flow:**\n- Request without payment → Returns 402 with payment details\n- Request with tx_hash → Verifies via Gateway → Delivers resource\n\n**Revenue:** You get 99.5% · AgentGatePay takes 0.5% commission\n\n**Customize:** Node 1 for price/resources · Node 7 for resource data\n\nFor more info:\nhttps://github.com/AgentGatePay/agentgatepay-examples/tree/main/n8n |
| Sticky Note 1 | Sticky Note | Section label: request processing | — | — | ## Request Processing\n\nWebhook receives GET request, parses your merchant config and headers, checks if buyer included a payment tx_hash, then routes accordingly. |
| Sticky Note 2 | Sticky Note | Section label: payment verification | — | — | ## Payment Verification\n\nAsks AgentGatePay to verify the tx_hash. Checks if payment went to your wallet and matches the resource price. Routes to delivery or error. |
| Sticky Note 3 | Sticky Note | Section label: response handling | — | — | ## Response Handling\n\nGenerates 402 response when no payment, or delivers the paid resource with 200 OK. Handles errors like wrong amount or invalid tx_hash. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Name it: `💲AgentGatePay - Seller Resource API [TEMPLATE]`
- Ensure workflow setting **Execution Order** is `v1` (optional but matches template behavior).

2) **Add Webhook node**
- Node name: `📡 GET /resource/{id}`
- Type: **Webhook**
- Configure:
  - Path: `resource/:resourceId`
  - Response Mode: **Using ‘Respond to Webhook’ node** (responseNode)
  - Keep raw body disabled
- This is your public endpoint after activation.

3) **Add Code node for parsing/config**
- Node name: `1️⃣ Parse Request`
- Type: **Code**
- Paste logic that:
  - Defines `SELLER_CONFIG` (merchant + payment + catalog)
  - Extracts `resourceId` from path/query
  - Reads headers `x-payment` and `x-agent-id`
  - Finds resource in `catalog`
  - Outputs:
    - `config`, `resource`, `request` object, and `route` (“verify_payment” or “return_402”)
  - Also returns 404 payload if resource not found; returns 500 payload if config invalid.
- **Edit required values:**
  - `SELLER_CONFIG.merchant.wallet_address` → your Base/EVM wallet address
  - `SELLER_CONFIG.merchant.api_key` → your AgentGatePay API key
- Connect: `📡 GET /resource/{id}` → `1️⃣ Parse Request`

4) **Add IF node to detect payment header**
- Node name: `2️⃣ Has Payment?`
- Type: **IF**
- Condition: String equals
  - Left: `={{ $json.route }}`
  - Right: `verify_payment`
- Connect: `1️⃣ Parse Request` → `2️⃣ Has Payment?`

5) **Add Code node to generate 402**
- Node name: `3️⃣ Generate 402`
- Type: **Code**
- Configure logic to:
  - Read `config` and `resource`
  - Compute `amount` based on token decimals (USDC 6, DAI 18)
  - Build 402 JSON with fields: `statusCode`, `payTo`, `amount`, `token`, `chain`, `nonce`, `instructions`, and resource preview
  - Output as `{ http_response: response_402 }`
- Connect: `2️⃣ Has Payment?` **false** → `3️⃣ Generate 402`

6) **Add Respond to Webhook for 402**
- Node name: `4️⃣ Send 402`
- Type: **Respond to Webhook**
- Respond with: **JSON**
- Response body expression:
  - Prefer: `={{ $json.http_response }}` (object)
  - If matching template exactly: `={{ JSON.stringify($json.http_response) }}`
- Connect: `3️⃣ Generate 402` → `4️⃣ Send 402`

7) **Add HTTP Request node for verification**
- Node name: `5️⃣ Verify Payment`
- Type: **HTTP Request**
- Configure:
  - Method: GET (default)
  - URL: `=https://api.agentgatepay.com/v1/payments/verify/{{ $('1️⃣ Parse Request').first().json.request.payment_tx_hash }}`
  - Send headers: enabled
  - Header: `x-api-key: ={{ $('1️⃣ Parse Request').first().json.config.merchant.api_key }}`
- Credentials:
  - None needed if using header API key (as in template)
- Connect: `2️⃣ Has Payment?` **true** → `5️⃣ Verify Payment`

8) **Add Code node to validate verification result**
- Node name: `6️⃣ Validate Payment`
- Type: **Code**
- Implement:
  - If `verified` false → output `{ route: "validation_failed", http_response: { statusCode: 400, ... } }`
  - Compare `to_address` to configured `wallet_address`
  - Compare paid USD amount to `resource.price_usd` within tolerance
  - On success output `{ ...request_data, verification, route: "validation_passed" }`
- Connect: `5️⃣ Verify Payment` → `6️⃣ Validate Payment`

9) **Add IF node to route success vs error**
- Node name: `6B️⃣ Route: Valid?`
- Type: **IF**
- Condition: `={{ $json.route }} == "validation_passed"`
- Connect: `6️⃣ Validate Payment` → `6B️⃣ Route: Valid?`

10) **Add Code node to deliver resource**
- Node name: `7️⃣ Deliver Resource`
- Type: **Code**
- Build final payload:
  - Include resource metadata, full paid content (`data`), payment fields, timestamp
  - Output `{ http_response: full_data }`
- Connect: `6B️⃣ Route: Valid?` **true** → `7️⃣ Deliver Resource`

11) **Add Respond to Webhook for 200**
- Node name: `8️⃣ Send 200 OK`
- Type: **Respond to Webhook**
- Respond with JSON
- Response body: ideally `={{ $json.http_response }}`
- Connect: `7️⃣ Deliver Resource` → `8️⃣ Send 200 OK`

12) **Add Respond to Webhook for errors**
- Node name: `9️⃣ Send Error`
- Type: **Respond to Webhook**
- Respond with JSON
- Response body: ideally `={{ $json.http_response }}`
- Connect: `6B️⃣ Route: Valid?` **false** → `9️⃣ Send Error`

13) **Important wiring fix (recommended)**
- Do **not** connect `1️⃣ Parse Request` directly to `9️⃣ Send Error` unless you gate it with an IF/Switch on `route` (e.g., only when `route` starts with `error_`).
- Otherwise you risk sending two webhook responses for one request.

14) **Activate and test**
- Activate workflow, copy the webhook URL from `📡 GET /resource/{id}`.
- Test unpaid:
  - `GET /resource/saas-competitors-2025` → expect 402 JSON with payment instructions.
- Test paid:
  - Repeat request with header `x-payment: <tx_hash>` → expect verification + 200 content (if verified).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Revenue: You get 99.5% · AgentGatePay takes 0.5% commission” | From sticky note “START HERE” |
| Customization points: Node 1 for wallet/key/catalog; Node 7 for delivered data | From sticky note “START HERE” |
| AgentGatePay examples for n8n | https://github.com/AgentGatePay/agentgatepay-examples/tree/main/n8n |
| Disclaimer: Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n... | Provided by user prompt (compliance/context statement) |