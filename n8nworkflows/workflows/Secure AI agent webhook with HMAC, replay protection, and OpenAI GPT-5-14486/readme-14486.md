Secure AI agent webhook with HMAC, replay protection, and OpenAI GPT-5

https://n8nworkflows.xyz/workflows/secure-ai-agent-webhook-with-hmac--replay-protection--and-openai-gpt-5-14486


# Secure AI agent webhook with HMAC, replay protection, and OpenAI GPT-5

# 1. Workflow Overview

This workflow exposes a secured HTTP webhook that accepts a JSON payload containing a single `prompt` field, validates the request authenticity with header-based authentication plus HMAC signature verification, applies replay protection using a timestamp, enforces strict JSON schema validation, and finally forwards the sanitized prompt to an AI Agent powered by OpenAI GPT-5 Mini.

Its primary use case is to provide a hardened inbound endpoint for AI interactions where request integrity and payload control matter, such as internal AI gateways, secure application-to-agent integrations, or backend-triggered prompt execution.

## 1.1 Input Reception and Raw Request Extraction

The workflow begins with a `Webhook` node configured for `POST` requests with header-based authentication. The raw binary request body is decoded and the security headers `x-timestamp` and `x-signature` are extracted for downstream verification.

## 1.2 HMAC Verification and Replay Protection

The workflow computes an HMAC over the exact string `x-timestamp.rawBody`, then performs a timing-safe comparison against the incoming signature. It also checks timestamp freshness within a 5-minute tolerance window to reduce replay risk. Invalid requests are terminated with HTTP 403.

## 1.3 Strict JSON Validation

If the signature is valid, the raw request body is parsed as JSON. The payload must contain exactly one allowed field, `prompt`, and no additional keys. Invalid JSON or disallowed fields lead to HTTP 400.

## 1.4 AI Processing

Validated input is sent to an `AI Agent` node using the prompt from `data.prompt`. The agent is backed by the `OpenAI Chat Model` configured with `gpt-5-mini` and streaming enabled.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception and Raw Request Extraction

### Overview

This block receives the HTTP request and converts the raw binary body back into a UTF-8 string so the exact original payload can be used for HMAC verification. It also extracts the required signature-related headers.

### Nodes Involved

- `Webhook`
- `Extract rawBody`

### Node Details

#### Webhook

- **Type and technical role:** `n8n-nodes-base.webhook`  
  Entry point for external HTTP requests.
- **Configuration choices:**
  - HTTP method: `POST`
  - Path: `7ca64bd2`
  - Authentication: `headerAuth`
  - Response mode: `streaming`
- **Key expressions or variables used:** None in parameters.
- **Input and output connections:**
  - Input: none, this is an entry node
  - Output: goes to `Extract rawBody`
- **Version-specific requirements:**
  - Uses `typeVersion: 2.1`
  - Streaming response mode should be supported in the deployed n8n version
- **Edge cases or potential failure types:**
  - Missing or invalid header auth credentials will block access before workflow logic runs
  - If the request body is not delivered in binary as expected, downstream extraction may fail
  - Clients must send headers and payload exactly as expected for signature verification to work
- **Sub-workflow reference:** None

#### Extract rawBody

- **Type and technical role:** `n8n-nodes-base.code`  
  Decodes the binary request body and extracts security headers.
- **Configuration choices:**
  - JavaScript code reads `items[0].binary.data`
  - Converts Base64 content into UTF-8 raw string
  - Reads `headers['x-timestamp']` and `headers['x-signature']`
- **Key expressions or variables used:**
  - `items[0].binary.data`
  - `items[0].json.headers`
  - Outputs:
    - `rawBody`
    - `xTimestamp`
    - `xSignature`
- **Input and output connections:**
  - Input: `Webhook`
  - Output: `Crypto`
- **Version-specific requirements:**
  - Uses `typeVersion: 2`
  - Assumes n8n stores request body under `binary.data`
- **Edge cases or potential failure types:**
  - If `binary.data` is absent, the code will throw
  - If request encoding differs or body is empty, decoding can fail or produce wrong data
  - Missing headers result in undefined `xTimestamp` or `xSignature`, which later causes validation failure
- **Sub-workflow reference:** None

---

## 2.2 HMAC Verification and Replay Protection

### Overview

This block reconstructs the canonical HMAC input, computes the signature using a shared secret, verifies timestamp freshness, and compares the computed and received signatures using a timing-safe method. Requests failing these checks are rejected with HTTP 403.

### Nodes Involved

- `Crypto`
- `Timing-Safe HMAC Check`
- `If2`
- `Response to webhook : Forbidden`

### Node Details

#### Crypto

- **Type and technical role:** `n8n-nodes-base.crypto`  
  Computes the HMAC for the incoming request.
- **Configuration choices:**
  - Action: `hmac`
  - Input value: `={{ $json.xTimestamp + '.' + $json.rawBody }}`
  - Uses dedicated Crypto credentials containing the shared secret
- **Key expressions or variables used:**
  - `$json.xTimestamp`
  - `$json.rawBody`
  - Output field typically contains HMAC result in `data`
- **Input and output connections:**
  - Input: `Extract rawBody`
  - Output: `Timing-Safe HMAC Check`
- **Version-specific requirements:**
  - Uses `typeVersion: 2`
  - Correct HMAC algorithm and output format depend on node defaults and credential setup; these must match the sender’s implementation exactly
- **Edge cases or potential failure types:**
  - Wrong secret in credentials causes all signatures to fail
  - Mismatch in canonical string format, whitespace, encoding, or HMAC digest format causes false negatives
  - Missing timestamp or body yields invalid signature base string
- **Sub-workflow reference:** None

#### Timing-Safe HMAC Check

- **Type and technical role:** `n8n-nodes-base.code`  
  Verifies replay window and performs timing-safe comparison of HMAC values.
- **Configuration choices:**
  - Parses `xTimestamp` as integer Unix timestamp
  - Rejects if missing or older/newer than 300 seconds from current time
  - Uses Node.js `crypto.timingSafeEqual`
  - Reads computed HMAC from `$json.data`
  - Reads received HMAC from `$json.xSignature`
- **Key expressions or variables used:**
  - `$json.xTimestamp`
  - `$json.data`
  - `$json.xSignature`
  - Returns:
    - `isValid`
    - `rawBody`
- **Input and output connections:**
  - Input: `Crypto`
  - Output: `If2`
- **Version-specific requirements:**
  - Uses `typeVersion: 2`
  - Requires Code node support for `require('crypto')`
- **Edge cases or potential failure types:**
  - Timestamp format must be Unix seconds, not milliseconds
  - The node only returns `rawBody`; it does not return headers or reason except on failure branches inside the code output
  - Signature comparison assumes both values are UTF-8 strings in identical encoding and format
  - Any mismatch in hex/base64 representation between sender and n8n causes invalid result
- **Sub-workflow reference:** None

#### If2

- **Type and technical role:** `n8n-nodes-base.if`  
  Branches based on whether HMAC/timestamp validation passed.
- **Configuration choices:**
  - Checks boolean truthiness of `={{ $json.isValid }}`
  - Strict type validation enabled
- **Key expressions or variables used:**
  - `$json.isValid`
- **Input and output connections:**
  - Input: `Timing-Safe HMAC Check`
  - True output: `Strict Payload Validation`
  - False output: `Response to webhook : Forbidden`
- **Version-specific requirements:**
  - Uses `typeVersion: 2.3`
  - Condition format reflects newer IF node condition model
- **Edge cases or potential failure types:**
  - If `isValid` is missing or non-boolean, strict type validation may force false behavior
  - The stored condition JSON also includes a right-side expression that is effectively irrelevant to the chosen operator; future edits should verify condition semantics in UI
- **Sub-workflow reference:** None

#### Response to webhook : Forbidden

- **Type and technical role:** `n8n-nodes-base.respondToWebhook`  
  Sends a `403 Forbidden` response when signature or replay protection fails.
- **Configuration choices:**
  - Respond with: `noData`
  - HTTP status code: `403`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input: `If2` false branch
  - Output: none
- **Version-specific requirements:**
  - Uses `typeVersion: 1.5`
- **Edge cases or potential failure types:**
  - If workflow execution reaches another responding node unexpectedly, webhook response conflicts may occur
  - No response body is returned, which is secure but less descriptive for debugging
- **Sub-workflow reference:** None

---

## 2.3 Strict JSON Validation

### Overview

This block parses the raw JSON body only after authenticity checks pass, then applies a strict schema-like filter: the payload must be valid JSON, must include `prompt`, and must not include any other fields. Invalid requests are rejected with HTTP 400.

### Nodes Involved

- `Strict Payload Validation`
- `If1`
- `Bad Request`

### Node Details

#### Strict Payload Validation

- **Type and technical role:** `n8n-nodes-base.code`  
  Parses and sanitizes payload structure.
- **Configuration choices:**
  - Uses `JSON.parse($json.rawBody)`
  - Rejects malformed JSON
  - Requires `data.prompt`
  - Allows only keys from `['prompt']`
  - Returns either:
    - `{ valid: false, error: ... }`
    - `{ valid: true, data }`
- **Key expressions or variables used:**
  - `$json.rawBody`
  - `data.prompt`
  - `allowedKeys = ['prompt']`
- **Input and output connections:**
  - Input: `If2` true branch
  - Output: `If1`
- **Version-specific requirements:**
  - Uses `typeVersion: 2`
- **Edge cases or potential failure types:**
  - `prompt` presence check is truthiness-based; empty string, `0`, `false`, or `null` would be rejected even if intentionally provided
  - Nested JSON inside `prompt` is not separately validated
  - Additional harmless metadata fields are rejected by design
- **Sub-workflow reference:** None

#### If1

- **Type and technical role:** `n8n-nodes-base.if`  
  Branches based on payload validation result.
- **Configuration choices:**
  - Checks whether `={{ $json.valid }}` equals `true`
  - Strict type validation enabled
- **Key expressions or variables used:**
  - `$json.valid`
- **Input and output connections:**
  - Input: `Strict Payload Validation`
  - True output: `AI Agent`
  - False output: `Bad Request`
- **Version-specific requirements:**
  - Uses `typeVersion: 2.3`
- **Edge cases or potential failure types:**
  - If `valid` is missing or not boolean, strict comparison will route to failure
- **Sub-workflow reference:** None

#### Bad Request

- **Type and technical role:** `n8n-nodes-base.respondToWebhook`  
  Sends a `400 Bad Request` response for invalid JSON or non-whitelisted payloads.
- **Configuration choices:**
  - Respond with: `noData`
  - HTTP status code: `400`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input: `If1` false branch
  - Output: none
- **Version-specific requirements:**
  - Uses `typeVersion: 1.5`
- **Edge cases or potential failure types:**
  - No body is returned, so clients receive only the status code
- **Sub-workflow reference:** None

---

## 2.4 AI Processing

### Overview

This block takes the verified and sanitized prompt and sends it to an AI Agent backed by the OpenAI Chat Model. Streaming is enabled, so the webhook can stream the agent response back to the caller.

### Nodes Involved

- `AI Agent`
- `OpenAI Chat Model`

### Node Details

#### AI Agent

- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  Runs an AI agent using the validated prompt.
- **Configuration choices:**
  - Prompt type: `define`
  - Text prompt: `={{ $json.data.prompt }}`
  - Streaming enabled
- **Key expressions or variables used:**
  - `$json.data.prompt`
- **Input and output connections:**
  - Main input: `If1` true branch
  - AI language model input: `OpenAI Chat Model`
  - Main output: none connected
- **Version-specific requirements:**
  - Uses `typeVersion: 3.1`
  - Requires compatible n8n LangChain/AI node support
- **Edge cases or potential failure types:**
  - If `data.prompt` is missing despite validation, execution fails
  - Prompt content may still contain unsafe instructions from a business perspective; schema validation only controls structure, not semantic prompt safety
  - Streaming behavior depends on frontend/client support and webhook response mode compatibility
  - No post-processing node is attached, so returned behavior depends on the agent/webhook streaming integration
- **Sub-workflow reference:** None

#### OpenAI Chat Model

- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  Supplies the language model used by the AI Agent.
- **Configuration choices:**
  - Model selected: `gpt-5-mini`
  - No additional options configured
  - No built-in tools enabled
  - Uses OpenAI API credentials
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Output via `ai_languageModel` connection to `AI Agent`
- **Version-specific requirements:**
  - Uses `typeVersion: 1.3`
  - The selected model must be available on the connected OpenAI account and supported by the installed node version
- **Edge cases or potential failure types:**
  - Missing/invalid API key
  - Model access restrictions
  - Rate limits, provider timeouts, or upstream API errors
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook | n8n-nodes-base.webhook | Receives authenticated POST requests |  | Extract rawBody | ## 1. Webhook & Extraction<br>Receives raw body payload and extracts the timestamp and signature security headers. |
| Extract rawBody | n8n-nodes-base.code | Decodes raw request body and reads security headers | Webhook | Crypto | ## 1. Webhook & Extraction<br>Receives raw body payload and extracts the timestamp and signature security headers. |
| Crypto | n8n-nodes-base.crypto | Computes HMAC over timestamp and raw payload | Extract rawBody | Timing-Safe HMAC Check | ## 2. HMAC & Replay Protection<br>Computes a SHA-256 HMAC and performs a timing-safe signature comparison. Returns a 403 Forbidden if invalid or expired. |
| Timing-Safe HMAC Check | n8n-nodes-base.code | Verifies timestamp freshness and signature equality | Crypto | If2 | ## 2. HMAC & Replay Protection<br>Computes a SHA-256 HMAC and performs a timing-safe signature comparison. Returns a 403 Forbidden if invalid or expired. |
| If2 | n8n-nodes-base.if | Routes valid vs invalid authenticated requests | Timing-Safe HMAC Check | Strict Payload Validation; Response to webhook : Forbidden | ## 2. HMAC & Replay Protection<br>Computes a SHA-256 HMAC and performs a timing-safe signature comparison. Returns a 403 Forbidden if invalid or expired. |
| Response to webhook : Forbidden | n8n-nodes-base.respondToWebhook | Returns HTTP 403 for failed signature or replay checks | If2 |  | ## 2. HMAC & Replay Protection<br>Computes a SHA-256 HMAC and performs a timing-safe signature comparison. Returns a 403 Forbidden if invalid or expired. |
| Strict Payload Validation | n8n-nodes-base.code | Parses and whitelists JSON payload fields | If2 | If1 | ## 3. Strict Payload Validation<br>Safely parses JSON and applies whitelist filtering to prevent injection. Returns a 400 Bad Request if unauthorized keys are found. |
| If1 | n8n-nodes-base.if | Routes valid vs invalid payload structure | Strict Payload Validation | AI Agent; Bad Request | ## 3. Strict Payload Validation<br>Safely parses JSON and applies whitelist filtering to prevent injection. Returns a 400 Bad Request if unauthorized keys are found. |
| Bad Request | n8n-nodes-base.respondToWebhook | Returns HTTP 400 for malformed or unauthorized payloads | If1 |  | ## 3. Strict Payload Validation<br>Safely parses JSON and applies whitelist filtering to prevent injection. Returns a 400 Bad Request if unauthorized keys are found. |
| AI Agent | @n8n/n8n-nodes-langchain.agent | Executes the AI agent on sanitized prompt input | If1; OpenAI Chat Model |  | ## 4. AI Agent<br>Securely processes the sanitized and verified prompt. |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provides GPT model to the AI agent |  | AI Agent | ## 4. AI Agent<br>Securely processes the sanitized and verified prompt. |
| Sticky Note8 | n8n-nodes-base.stickyNote | Documentation and setup guidance |  |  | # Secure AI Agent Webhook<br><br>### **⚠️ Disclaimer: I am not a cybersecurity expert.**<br>> This workflow was built through research and with the assistance of an LLM (Claude 3.5 Sonnet / Opus). While it implements well-established security patterns (HMAC-SHA256, timing-safe comparison, replay protection, strict payload validation), please review the logic carefully and ensure it meets your own security requirements before deploying it in production.<br><br>## How it works<br>This workflow provides an enterprise-grade, highly secure entry point for AI Agents. It receives raw JSON payloads via a Webhook and performs rigorous security validations before processing.<br><br>It checks for a valid Header Auth token, protects against replay attacks by verifying an `X-Timestamp`, and computes a SHA-256 HMAC signature against the raw payload to ensure byte-for-byte integrity. Finally, it strictly validates the JSON schema to prevent parameter injection before passing the clean prompt to the AI Agent.<br><br>### Setup<br>1. Configure the **Webhook** node with your HTTP Header Auth credentials.<br>2. Add a **Crypto** credential to the Crypto node containing your secure shared HMAC secret (generate via `openssl rand -hex 32`).<br>3. Configure the **OpenAI Chat Model** with your OpenAI API key.<br>4. Send requests to the Webhook URL including `X-Timestamp` and `X-Signature` headers. |
| Sticky Note | n8n-nodes-base.stickyNote | Visual documentation for block 1 |  |  | ## 1. Webhook & Extraction<br>Receives raw body payload and extracts the timestamp and signature security headers. |
| Sticky Note1 | n8n-nodes-base.stickyNote | Visual documentation for block 2 |  |  | ## 2. HMAC & Replay Protection<br>Computes a SHA-256 HMAC and performs a timing-safe signature comparison. Returns a 403 Forbidden if invalid or expired. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Visual documentation for block 3 |  |  | ## 3. Strict Payload Validation<br>Safely parses JSON and applies whitelist filtering to prevent injection. Returns a 400 Bad Request if unauthorized keys are found. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Visual documentation for block 4 |  |  | ## 4. AI Agent<br>Securely processes the sanitized and verified prompt. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `Secure AI Agent Webhook with HMAC Signature & Strict JSON Validation`.
   - In workflow settings:
     - Set **Binary Data Mode** to `separate` if your n8n version exposes this setting.
     - Keep execution order at default compatible mode if needed.

2. **Add the Webhook node**
   - Node type: `Webhook`
   - Configure:
     - **HTTP Method**: `POST`
     - **Path**: choose a path such as `7ca64bd2`
     - **Authentication**: `Header Auth`
     - **Response Mode**: `Streaming`
   - Create or assign an **HTTP Header Auth** credential.
   - The credential should require whatever inbound auth header your clients must send.

3. **Add a Code node after the Webhook**
   - Name: `Extract rawBody`
   - Connect `Webhook -> Extract rawBody`
   - Paste this logic conceptually:
     - Read `items[0].binary.data`
     - Decode Base64 to UTF-8 text
     - Read `items[0].json.headers`
     - Output:
       - `rawBody`
       - `xTimestamp` from `headers['x-timestamp']`
       - `xSignature` from `headers['x-signature']`
   - The effective code behavior should match:
     - decode raw request body exactly
     - do not normalize or re-stringify JSON before signature verification

4. **Add the Crypto node**
   - Name: `Crypto`
   - Connect `Extract rawBody -> Crypto`
   - Configure:
     - **Operation/Action**: `HMAC`
     - **Value to sign**: `={{ $json.xTimestamp + '.' + $json.rawBody }}`
   - Create a **Crypto** credential containing the shared HMAC secret.
   - Ensure the sender uses the exact same:
     - secret
     - algorithm
     - payload format
     - output encoding

5. **Add a second Code node**
   - Name: `Timing-Safe HMAC Check`
   - Connect `Crypto -> Timing-Safe HMAC Check`
   - Configure it to:
     - import Node’s `crypto` module
     - parse `xTimestamp` as Unix seconds
     - compare with current Unix time
     - reject if older/newer than 300 seconds
     - read computed HMAC from `$json.data`
     - read received signature from `$json.xSignature`
     - compare with `crypto.timingSafeEqual`
     - output:
       - `isValid`
       - `rawBody`
   - Important: the code should preserve `rawBody` for downstream JSON validation.

6. **Add an IF node for auth decision**
   - Name: `If2`
   - Connect `Timing-Safe HMAC Check -> If2`
   - Configure condition:
     - Boolean check on `={{ $json.isValid }}`
     - route **true** to valid requests
     - route **false** to invalid requests
   - Use strict type validation if available.

7. **Add a Respond to Webhook node for forbidden requests**
   - Name: `Response to webhook : Forbidden`
   - Connect the **false** output of `If2` to this node
   - Configure:
     - **Respond With**: `No Data`
     - **Response Code**: `403`

8. **Add a Code node for payload validation**
   - Name: `Strict Payload Validation`
   - Connect the **true** output of `If2` to this node
   - Configure code with these rules:
     1. `JSON.parse($json.rawBody)` inside a try/catch
     2. If parse fails, output `valid: false`
     3. Require `prompt`
     4. Only allow keys in `['prompt']`
     5. On success, output `valid: true` and `data`
   - The logic should reject any unexpected key to prevent parameter injection.

9. **Add another IF node**
   - Name: `If1`
   - Connect `Strict Payload Validation -> If1`
   - Configure condition:
     - `={{ $json.valid }}` equals `true`
   - Use strict type validation if available.

10. **Add a Respond to Webhook node for bad payloads**
    - Name: `Bad Request`
    - Connect the **false** output of `If1` to this node
    - Configure:
      - **Respond With**: `No Data`
      - **Response Code**: `400`

11. **Add the AI Agent node**
    - Name: `AI Agent`
    - Connect the **true** output of `If1` to this node
    - Configure:
      - **Prompt Type**: `Define`
      - **Text**: `={{ $json.data.prompt }}`
      - Enable **Streaming**
    - This node will process only sanitized prompt input.

12. **Add the OpenAI Chat Model node**
    - Name: `OpenAI Chat Model`
    - Connect it to the `AI Agent` using the AI language model connector, not the main connector
    - Configure:
      - **Model**: `gpt-5-mini`
      - Leave extra options empty unless you need deterministic settings, temperature tuning, or tool usage
    - Create or assign **OpenAI API** credentials.

13. **Verify connection layout**
    - Main chain:
      - `Webhook -> Extract rawBody -> Crypto -> Timing-Safe HMAC Check -> If2`
    - Valid-auth branch:
      - `If2 (true) -> Strict Payload Validation -> If1`
    - Invalid-auth branch:
      - `If2 (false) -> Response to webhook : Forbidden`
    - Valid-payload branch:
      - `If1 (true) -> AI Agent`
    - Invalid-payload branch:
      - `If1 (false) -> Bad Request`
    - AI model side connection:
      - `OpenAI Chat Model -> AI Agent`

14. **Optionally add sticky notes**
    - Add one general note describing the security model and setup steps.
    - Add separate sticky notes over each logical block:
      - Webhook & Extraction
      - HMAC & Replay Protection
      - Strict Payload Validation
      - AI Agent

15. **Configure client request format**
    - Client must send:
      - authenticated header for Header Auth
      - `X-Timestamp`
      - `X-Signature`
      - raw JSON body like:
        ```json
        {"prompt":"Your question here"}
        ```
    - Signature input must be exactly:
      - `<timestamp>.<raw JSON body>`
    - The HMAC digest format used by the sender must match what the `Crypto` node outputs.

16. **Test invalid scenarios**
    - Missing auth header → blocked by Webhook auth
    - Missing `X-Timestamp` → 403
    - Expired timestamp → 403
    - Invalid HMAC → 403
    - Malformed JSON → 400
    - Missing `prompt` → 400
    - Extra keys like `{"prompt":"x","system":"hack"}` → 400

17. **Test valid scenario**
    - Send a properly signed request with fresh timestamp and body containing only `prompt`
    - Confirm the request reaches `AI Agent`
    - Confirm streaming behavior works with your HTTP client

18. **Production hardening recommendations**
    - Consider returning structured JSON error bodies during development only
    - Log rejection reasons internally but avoid exposing them publicly
    - Ensure sender and receiver agree on:
      - timestamp format
      - body encoding
      - HMAC digest encoding
      - header casing expectations
    - If stricter replay defense is required, add nonce storage in addition to timestamp validation

### Credential configuration summary

- **HTTP Header Auth credential**
  - Used by `Webhook`
  - Protects the endpoint before workflow logic starts

- **Crypto credential**
  - Used by `Crypto`
  - Stores the shared HMAC secret
  - Suggested secret generation:
    - `openssl rand -hex 32`

- **OpenAI API credential**
  - Used by `OpenAI Chat Model`
  - Must have access to `gpt-5-mini`

### Sub-workflow setup

This workflow does **not** invoke any sub-workflows and does not contain any `Execute Workflow` node.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Secure AI Agent Webhook | General workflow concept |
| Disclaimer: the author explicitly notes they are not a cybersecurity expert and recommends reviewing the implementation before production use. | General security caution |
| The workflow was built through research and with assistance from an LLM: Claude 3.5 Sonnet / Opus. | Project credit |
| Shared HMAC secret generation example: `openssl rand -hex 32` | Setup instruction |
| Requests must include `X-Timestamp` and `X-Signature` headers. | Client integration requirement |
| Uses security patterns: HMAC-SHA256, timing-safe comparison, replay protection, strict payload validation. | Architecture summary |

**Disclaimer:** Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.