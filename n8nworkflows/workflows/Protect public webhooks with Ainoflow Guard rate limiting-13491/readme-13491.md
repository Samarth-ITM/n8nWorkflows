Protect public webhooks with Ainoflow Guard rate limiting

https://n8nworkflows.xyz/workflows/protect-public-webhooks-with-ainoflow-guard-rate-limiting-13491


# Protect public webhooks with Ainoflow Guard rate limiting

## 1. Workflow Overview

**Purpose:** Protect a public n8n webhook endpoint from burst traffic and abuse by applying **Ainoflow Guard** rate limiting *before* executing any expensive business logic.

**Target use cases:**
- Public-facing webhook endpoints (forms, callbacks, integrations) that can be spammed
- Preventing workflow overload and reducing compute costs
- “Edge-style” allow/deny decisions early in the execution path

### 1.1 Input Reception
Receives an incoming POST request on a fixed webhook path and holds the response until a Respond to Webhook node is reached.

### 1.2 Rate Limit Decision (Guard)
Sets rate-limit configuration, derives an identity key (IP or API key), calls Ainoflow Guard to decide if the request is allowed, then routes execution to allowed/denied branches.

### 1.3 Allowed Path (Business Logic + 200)
Runs placeholder business logic (intended to be replaced) and returns HTTP 200.

### 1.4 Denied Path (429 + Retry-After)
Builds a standardized “rate limited” payload and returns HTTP 429 with a `Retry-After` header derived from Guard’s reset timer.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception
**Overview:** Accepts POST requests and defers the HTTP response until the workflow explicitly responds. This enables the workflow to return either 200 or 429 based on Guard’s decision.  
**Nodes involved:** `Webhook`

#### Node: Webhook
- **Type / role:** `Webhook` trigger; entry point for HTTP requests.
- **Key configuration (interpreted):**
  - **HTTP Method:** POST
  - **Path:** `rate-limited-endpoint`
  - **Response mode:** *Respond to Webhook* (workflow-controlled response timing)
- **Inputs / outputs:**
  - **Output →** `Config`
- **Version-specific notes:** Node typeVersion `2` (standard in modern n8n).
- **Edge cases / failures:**
  - If no Respond to Webhook node is reached, requests may hang until n8n times out.
  - If used behind proxies/CDNs, real client IP may be in `x-forwarded-for` rather than the direct connection IP (handled later).

---

### Block 2 — Rate Limit Decision (Guard)
**Overview:** Defines rate limiting parameters, derives an identity key, then calls Ainoflow Guard to get an allow/deny decision. Branching is based on `allowed === true`.  
**Nodes involved:** `Config`, `BuildIdentity`, `GuardCheck`, `IfAllowed`

#### Node: Config
- **Type / role:** `Set` node; centralizes rate limit and identity settings.
- **Key configuration:**
  - `rate_limit` (number): **30**
  - `window_sec` (number): **60**
  - `identity_mode` (string): **"ip"** (supported: `"ip"` or `"apiKey"`)
  - `route_name` (string): **"webhook"** (namespacing identity keys to avoid cross-route pollution)
- **Inputs / outputs:**
  - **Input ←** `Webhook`
  - **Output →** `BuildIdentity`
- **Version-specific notes:** Set node typeVersion `3.4`.
- **Edge cases / failures:**
  - Mis-typed values (e.g., string for `rate_limit`) can cause unexpected behavior downstream in query params.

#### Node: BuildIdentity
- **Type / role:** `Code` node; computes the identity used for rate limiting.
- **Key logic (interpreted):**
  - Reads:
    - `headers` from `$('Webhook').first().json.headers`
    - `config` from `$input.first().json` (output of `Config`)
  - If `identity_mode === 'apiKey'` **and** header `x-api-key` exists → identity = that API key
  - Else (IP mode):
    - Prefer `x-forwarded-for` first value (`split(',')[0].trim()`)
    - Else use `x-real-ip`
    - Else fallback to `'unknown'`
  - Builds `identity_key` as: `${route_name}:${identity}`
  - Outputs: `identity_key`, `identity`, `rate_limit`, `window_sec`
- **Inputs / outputs:**
  - **Input ←** `Config`
  - **Output →** `GuardCheck`
- **Version-specific notes:** Code node typeVersion `2`.
- **Edge cases / failures:**
  - If headers are missing/unexpected casing, identity may fall back to `'unknown'` (risk: many clients share the same limiter bucket).
  - If a proxy injects multiple IPs into `x-forwarded-for`, only the **first** is used (intended “real client” behavior; ensure your infra preserves ordering).
  - If `route_name` is changed, it effectively creates a new namespace (new limiter buckets).

#### Node: GuardCheck
- **Type / role:** `HTTP Request`; calls Ainoflow Guard counter endpoint to increment/check rate limit.
- **Key configuration (interpreted):**
  - **Method:** POST
  - **URL:** `https://api.ainoflow.io/api/v1/guard/{identity_key}/counter` with `identity_key` URL-encoded
  - **Query parameters:**
    - `rateMax` = `{{$json.rate_limit}}`
    - `rateWindow` = `{{$json.window_sec}}`
    - `returnSuccess` = `true` (ensures HTTP 200 responses from Guard API even on deny; decision is in JSON body)
    - `failOpen` = `true` (if Guard errors/unavailable, allow traffic through)
    - `allowPolicyOverwrite` = `true` (convenient for testing; can cause config drift in production)
  - **Timeout:** 5000 ms
  - **Authentication:** HTTP Bearer Auth credential named **“Ainoflow”**
  - **alwaysOutputData:** true (node produces output even on certain errors, improving workflow continuity)
- **Inputs / outputs:**
  - **Input ←** `BuildIdentity`
  - **Output →** `IfAllowed`
- **Version-specific notes:** HTTP Request typeVersion `4.2`.
- **Edge cases / failures:**
  - Auth failure (401/403) if bearer token missing/invalid.
  - Network timeouts; with `failOpen=true`, the service intent is “allow”, but n8n still must receive a valid response to set `allowed`. Because `alwaysOutputData=true`, you may still get output but with error-shaped data depending on n8n version/settings—ensure `IfAllowed` is robust if Guard is unreachable.
  - `allowPolicyOverwrite=true` can overwrite server-side policy parameters on each request; in production, set to false to prevent accidental changes.

#### Node: IfAllowed
- **Type / role:** `If`; branches based on Guard decision.
- **Condition:**
  - Checks boolean `{{$json.allowed}}` is **true**
- **Inputs / outputs:**
  - **Input ←** `GuardCheck`
  - **True →** `BusinessLogic`
  - **False →** `BuildDeniedResponse`
- **Version-specific notes:** If node typeVersion `2.2` using condition model “version 2”.
- **Edge cases / failures:**
  - If Guard response does not include `allowed` (e.g., on unexpected error payload), condition evaluates false → may deny by default. If you want strict fail-open behavior at the workflow level, you could add a fallback: treat missing `allowed` as true.

---

### Block 3 — Allowed Path (Business Logic + 200)
**Overview:** Executes the protected workload only when Guard allows the request, then responds with HTTP 200.  
**Nodes involved:** `BusinessLogic`, `RespondOk`

#### Node: BusinessLogic
- **Type / role:** `Code`; placeholder for the user’s actual workflow logic.
- **Current behavior:**
  - Reads `body` from `$('Webhook').first().json.body`
  - Returns `{ ok: true, data: body || { message: "Request processed successfully" } }`
- **Notable references (documented in-node):**
  - Access headers: `$('Webhook').first().json.headers`
  - Access query: `$('Webhook').first().json.query`
  - Guard metadata: `$('GuardCheck').first().json.remaining`, `...resetsIn`
- **Inputs / outputs:**
  - **Input ←** `IfAllowed` (true branch)
  - **Output →** `RespondOk`
- **Version-specific notes:** Code node typeVersion `2`.
- **Edge cases / failures:**
  - Exceptions in code will prevent responding unless error handling is added; webhook caller may see a 500 or timeout depending on n8n settings.

#### Node: RespondOk
- **Type / role:** `Respond to Webhook`; sends final HTTP response for allowed requests.
- **Key configuration:**
  - **Response code:** 200
  - Response body defaults to incoming item JSON from `BusinessLogic`
- **Inputs / outputs:**
  - **Input ←** `BusinessLogic`
  - No further outputs (terminates webhook response)
- **Version-specific notes:** typeVersion `1.1`.
- **Edge cases / failures:**
  - If the incoming data is not valid JSON for response serialization, response might fail (rare with standard n8n items).

---

### Block 4 — Denied Path (429 + Retry-After)
**Overview:** If rate limit is exceeded, creates a consistent payload and responds immediately with HTTP 429 plus `Retry-After`.  
**Nodes involved:** `BuildDeniedResponse`, `RespondRateLimited`

#### Node: BuildDeniedResponse
- **Type / role:** `Set`; constructs the response payload for denied requests.
- **Key configuration:**
  - `ok` = false
  - `error` = `"rate_limited"`
  - `retryAfter` = `{{$json.resetsIn}}` (from Guard response)
- **Inputs / outputs:**
  - **Input ←** `IfAllowed` (false branch; data comes from `GuardCheck`)
  - **Output →** `RespondRateLimited`
- **Version-specific notes:** Set node typeVersion `3.4`.
- **Edge cases / failures:**
  - If `resetsIn` is missing or non-numeric, `Retry-After` may be invalid or blank downstream.

#### Node: RespondRateLimited
- **Type / role:** `Respond to Webhook`; returns 429 response.
- **Key configuration:**
  - **Response code:** 429
  - **Response headers:**
    - `Retry-After` = `{{$json.retryAfter}}`
- **Inputs / outputs:**
  - **Input ←** `BuildDeniedResponse`
- **Version-specific notes:** typeVersion `1.1`.
- **Edge cases / failures:**
  - If `Retry-After` is not a valid value, clients may ignore it; consider coercing to integer seconds.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| README | Sticky Note | Embedded project notes and operational guidance |  |  | # Webhook Rate Limiter (Guard)\n\nProtect public webhooks from burst traffic, abuse, and overload.\nUses **Ainoflow Guard** for edge-style rate decisions BEFORE expensive workflow logic executes.\n\n## Quick Start\n\n## 1. Setup Ainoflow\n### 1.1 Create API Key\n - https://www.ainoflow.io/signup (free plan available)\n### 1.2 Setup HTTP Bearer Credentials:\n - Ainoflow\n\n## 2. Configure Rate Limits\n### Edit **Config** node:\n - `rate_limit` — Max requests per window (default: 30)\n - `window_sec` — Window size in seconds (default: 60)\n - `identity_mode` — \"ip\" or \"apiKey\" (default: ip)\n - `route_name` — Logical endpoint name (default: webhook)\n\n## 3. Add Your Business Logic\n### Replace **BusinessLogic** node:\n - Access request body: `$('Webhook').first().json.body`\n - Access headers: `$('Webhook').first().json.headers`\n - Return your response data as JSON\n\n## 4. Test\n### Burst test (terminal):\n```\nfor i in {1..50}; do\n  curl -s -o /dev/null -w \"%{http_code}\\n\" \\\n    -X POST https://your-n8n.com/webhook/rate-limited-endpoint \\\n    -H \"Content-Type: application/json\" \\\n    -d '{\"test\": true}'\ndone\n```\n\n### Expected:\n - First 30 requests → 200 OK\n - Remaining → 429 Too Many Requests\n\n## 5. How It Works\n\n1. Webhook receives POST request\n2. Identity extracted (IP or API key)\n3. Guard checks rate limit (allow/deny)\n4. Allowed → Business Logic → 200 OK\n5. Denied → 429 + Retry-After header\n\n## 6. Architecture\n\n- **Fail-open**: Guard API uses `failOpen=true`\n  (Guard down → requests allowed through)\n- **Stateless**: No queues or databases needed in n8n\n- **Edge-style**: Decision before business logic\n- **Proxy-aware**: X-Forwarded-For support\n- Guard policy auto-creates on first request\n\n## 7. Identity Modes\n\n### IP mode (default)\n - Extracts client IP from X-Forwarded-For or x-real-ip\n - Works behind Cloudflare, nginx, load balancers\n - Identity key: `webhook:185.22.xx.xx`\n\n### API Key mode\n - Uses x-api-key header\n - Falls back to IP if header missing\n - Identity key: `webhook:client_abc123`\n\n## 8. Guard API Details\n\n - Endpoint: POST /api/v1/guard/{key}/counter\n - Policy auto-created with rateMax + rateWindow\n - returnSuccess=true → always 200 OK response\n - allowPolicyOverwrite=true → easy testing (set false in production)\n - Response: { allowed, remaining, resetsIn, rateLimit }\n\n## 9. Combine with Shield\nDuplicate webhooks? Add Ainoflow Shield for\none-trigger-one-execution guarantee.\nGuard + Shield = rate limiting + dedup.\n\n## 10. Need help?\nAinova Systems: https://ainovasystems.com/ |
| SectionRateLimitCheck | Sticky Note | Section header for rate limit decision block |  |  | ## 1. Rate Limit Decision\nWebhook → Config → Build Identity → Guard Check → Allow or Deny |
| SectionAllowed | Sticky Note | Section header for allowed branch |  |  | ## 2. Allowed → Business Logic\nReplace **BusinessLogic** node with your workflow |
| SectionDenied | Sticky Note | Section header for denied branch |  |  | ## 3. Denied → 429 Rate Limited\nImmediate rejection with Retry-After header |
| StickyWebhook | Sticky Note (disabled) | Commentary about webhook entry behavior |  |  | ## Webhook Entry\nPOST requests only.\nUses \"Respond to Webhook\" mode\nso workflow controls response timing. |
| StickyConfig | Sticky Note (disabled) | Commentary about config node |  |  | ## Configuration\nEdit values here to change\nrate limits and identity mode.\nNo code changes needed. |
| StickyIdentity | Sticky Note (disabled) | Commentary about identity builder |  |  | ## Identity Builder\nExtracts client identity from\nrequest headers.\nFormat: route:identity |
| StickyGuard | Sticky Note (disabled) | Commentary about Guard decision details |  |  | ## Guard Decision\nPOST to Guard API.\nPolicy auto-creates on first call.\nreturnSuccess=true → always 200.\n\n⚠️ allowPolicyOverwrite=true\nis set for easy testing.\nFor production: set to false\nto avoid hidden config drift. |
| Webhook | Webhook | Entry point (POST) and holds response for response nodes | — | Config |  |
| Config | Set | Defines rate limiting parameters and identity mode | Webhook | BuildIdentity |  |
| BuildIdentity | Code | Extracts identity (IP/API key) and builds Guard key | Config | GuardCheck |  |
| GuardCheck | HTTP Request | Calls Ainoflow Guard counter endpoint to allow/deny | BuildIdentity | IfAllowed |  |
| IfAllowed | If | Branches based on Guard `allowed` boolean | GuardCheck | BusinessLogic; BuildDeniedResponse |  |
| BusinessLogic | Code | Placeholder protected business logic | IfAllowed (true) | RespondOk |  |
| RespondOk | Respond to Webhook | Returns 200 for allowed requests | BusinessLogic | — |  |
| BuildDeniedResponse | Set | Builds payload for 429 response (incl. retryAfter) | IfAllowed (false) | RespondRateLimited |  |
| RespondRateLimited | Respond to Webhook | Returns 429 + Retry-After header | BuildDeniedResponse | — |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add Webhook node**
   - Node type: **Webhook**
   - HTTP Method: **POST**
   - Path: `rate-limited-endpoint`
   - Response Mode: **Using “Respond to Webhook” node** (a.k.a. “responseNode”)
   - Connect **Webhook → Config**

3. **Add Config node**
   - Node type: **Set**
   - Add fields:
     - `rate_limit` (Number) = `30`
     - `window_sec` (Number) = `60`
     - `identity_mode` (String) = `ip` (or `apiKey`)
     - `route_name` (String) = `webhook`
   - Connect **Config → BuildIdentity**

4. **Add BuildIdentity node**
   - Node type: **Code**
   - Paste logic equivalent to:
     - Read `headers` from `$('Webhook').first().json.headers`
     - Read config from `$input.first().json`
     - If `identity_mode === 'apiKey'` and `headers['x-api-key']` exists → use it
     - Else use `x-forwarded-for` first IP, else `x-real-ip`, else `'unknown'`
     - Output:
       - `identity_key` = `${route_name}:${identity}`
       - `identity`, `rate_limit`, `window_sec`
   - Connect **BuildIdentity → GuardCheck**

5. **Create Ainoflow bearer credential**
   - Go to **Credentials → New**
   - Choose **HTTP Bearer Auth**
   - Name it **Ainoflow**
   - Paste your Ainoflow API key/token as the bearer token
   - Save

6. **Add GuardCheck node**
   - Node type: **HTTP Request**
   - Method: **POST**
   - URL (expression):  
     `https://api.ainoflow.io/api/v1/guard/{{ encodeURIComponent($json.identity_key) }}/counter`
   - Authentication: **Generic credential type → HTTP Bearer Auth**
     - Select credential: **Ainoflow**
   - Send Query Parameters: **ON**
   - Add query parameters:
     - `rateMax` = `{{$json.rate_limit}}`
     - `rateWindow` = `{{$json.window_sec}}`
     - `returnSuccess` = `true`
     - `failOpen` = `true`
     - `allowPolicyOverwrite` = `true` (set to `false` for production stability)
   - Options:
     - Timeout: `5000` ms
   - (Optional but recommended to match behavior) enable **Always Output Data**
   - Connect **GuardCheck → IfAllowed**

7. **Add IfAllowed node**
   - Node type: **If**
   - Condition: Boolean check `{{$json.allowed}}` is **true**
   - Connect **True output → BusinessLogic**
   - Connect **False output → BuildDeniedResponse**

8. **Add BusinessLogic node (placeholder / replace later)**
   - Node type: **Code**
   - Implement your protected logic; to access webhook data:
     - Body: `$('Webhook').first().json.body`
     - Headers: `$('Webhook').first().json.headers`
     - Query: `$('Webhook').first().json.query`
   - Return an item with `json` to be used as response body
   - Connect **BusinessLogic → RespondOk**

9. **Add RespondOk node**
   - Node type: **Respond to Webhook**
   - Response Code: **200**
   - Connect from **BusinessLogic**

10. **Add BuildDeniedResponse node**
   - Node type: **Set**
   - Set fields:
     - `ok` (Boolean) = `false`
     - `error` (String) = `rate_limited`
     - `retryAfter` (Number, expression) = `{{$json.resetsIn}}`
   - Connect **BuildDeniedResponse → RespondRateLimited**

11. **Add RespondRateLimited node**
   - Node type: **Respond to Webhook**
   - Response Code: **429**
   - Response Headers:
     - `Retry-After` = `{{$json.retryAfter}}`

12. **Activate the workflow**, then test by sending repeated POST requests to:  
   `https://<your-n8n-host>/webhook/rate-limited-endpoint`

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Ainoflow signup (free plan available) | https://www.ainoflow.io/signup |
| Vendor / help | Ainova Systems: https://ainovasystems.com/ |
| Operational behavior: “Fail-open” configured | Guard call uses `failOpen=true` so Guard downtime is intended to allow traffic through |
| Testing caveat | `allowPolicyOverwrite=true` is convenient for tests but should typically be `false` in production to avoid policy drift |