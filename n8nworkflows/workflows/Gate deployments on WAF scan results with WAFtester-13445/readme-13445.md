Gate deployments on WAF scan results with WAFtester

https://n8nworkflows.xyz/workflows/gate-deployments-on-waf-scan-results-with-waftester-13445


# Gate deployments on WAF scan results with WAFtester

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow acts as a **CI/CD deployment gate** based on **WAF (Web Application Firewall) security scan results** produced by **WAFtester** (via an MCP/JSON-RPC HTTP endpoint). A pipeline calls an n8n webhook with a target URL and optional test categories; the workflow runs detection + scanning, polls for results, computes a pass/fail decision against a threshold, and responds with an HTTP status code the pipeline can use to allow or block deployment.

**Primary use cases**
- DevOps/Platform: automated security gates before deploy
- Security: enforcing minimum WAF effectiveness (detection rate)
- CI/CD: standard webhook gate returning 200 (pass) / 422 (fail)

### Logical blocks
1. **1.1 Input Reception (Webhook Trigger)**
2. **1.2 WAFtester Interaction: Detection + Start Async Scan**
3. **1.3 Async Handling: Wait + Poll Task Status**
4. **1.4 Result Parsing & Gate Decision**
5. **1.5 Webhook Response to Pipeline (Pass/Fail)**

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception (Webhook Trigger)

**Overview:** Receives a POST request from a CI/CD pipeline containing the scan target and optional test categories, and keeps the execution open until a later “Respond to Webhook” node returns the final decision.

**Nodes involved**
- **Deploy Webhook**

#### Node: Deploy Webhook
- **Type / role:** `Webhook` (Trigger). Entry point for the workflow.
- **Key configuration (interpreted):**
  - **HTTP Method:** POST
  - **Path:** `waf-security-gate` (final URL depends on n8n host)
  - **Response mode:** “Respond with Respond to Webhook node” (execution will wait for a response node)
- **Expected request body (as documented in node notes):**
  - `{"target": "https://...", "categories": ["sqli","xss"]}`
- **Key expressions / variables used by downstream nodes:**
  - `$('Deploy Webhook').item.json.body.target`
  - `$('Deploy Webhook').item.json.body.categories`
- **Connections:**
  - Output → **Detect WAF**
- **Edge cases / failures:**
  - Missing/invalid JSON body (e.g., no `target`) will cause downstream expressions to resolve to `undefined` and likely break scanning or produce incorrect JSON-RPC arguments.
  - Because response mode is delegated, if the workflow errors before a Respond node runs, the CI/CD caller may receive a timeout or generic error depending on n8n config.

---

### 2.2 WAFtester Interaction: Detection + Start Async Scan

**Overview:** Calls the WAFtester MCP server to (1) identify the WAF vendor and (2) start an asynchronous scan for the requested categories.

**Nodes involved**
- **Detect WAF**
- **Start Scan**

#### Node: Detect WAF
- **Type / role:** `HTTP Request` (JSON-RPC client to WAFtester MCP).
- **Key configuration (interpreted):**
  - **URL:** `{{$env.WAFTESTER_MCP_URL || 'http://waftester:8080/mcp'}}`
    - Uses environment variable `WAFTESTER_MCP_URL`; falls back to a Docker-network style hostname.
  - **Method:** POST
  - **Timeout:** 30,000 ms
  - **Headers:** `Content-Type: application/json`
  - **Body:** JSON-RPC 2.0 payload calling MCP tool `detect_waf` with:
    - `target`: from webhook body
- **Connections:**
  - Input ← **Deploy Webhook**
  - Output → **Start Scan**
- **Edge cases / failures:**
  - MCP endpoint unreachable/DNS failure (`http://waftester:8080/mcp`) → request error.
  - `target` undefined → WAFtester may error or return a failed tool result.
  - Non-JSON response or unexpected structure could break assumptions later (though this node’s output is not directly parsed in this workflow).

#### Node: Start Scan
- **Type / role:** `HTTP Request` (starts async scan via MCP tool `scan`).
- **Key configuration (interpreted):**
  - **URL:** same env-based MCP URL as above
  - **Method:** POST
  - **Timeout:** 30,000 ms
  - **Headers:** `Content-Type: application/json`
  - **Body:** JSON-RPC tool call `scan` with arguments:
    - `target`: from webhook body
    - `categories`: if webhook provides categories, they are JSON-stringified; otherwise defaults to `["sqli","xss"]`
      - Expression:  
        `{{ $('Deploy Webhook').item.json.body.categories ? JSON.stringify($('Deploy Webhook').item.json.body.categories) : '["sqli", "xss"]' }}`
- **Connections:**
  - Input ← **Detect WAF**
  - Output → **Wait for Scan**
- **Edge cases / failures:**
  - If `categories` is provided but is not an array, `JSON.stringify(...)` still produces something, but WAFtester may reject it.
  - If WAFtester returns a task id in an unexpected format, the later regex extraction may fail.

**Version-specific notes:** HTTP Request node is `typeVersion 4.2`; behavior and UI fields can differ slightly across n8n versions, particularly around body configuration and timeout/options placement.

---

### 2.3 Async Handling: Wait + Poll Task Status

**Overview:** Introduces a fixed delay, then polls WAFtester for scan completion/results using a task ID extracted from the scan start response.

**Nodes involved**
- **Wait for Scan**
- **Poll Task Status**

#### Node: Wait for Scan
- **Type / role:** `Wait` (delays execution).
- **Key configuration:**
  - Waits **30 seconds**
- **Connections:**
  - Input ← **Start Scan**
  - Output → **Poll Task Status**
- **Edge cases / failures:**
  - A fixed wait may be insufficient for longer scans, causing polling to return “still running” or partial results depending on WAFtester behavior.
  - Conversely, it may add unnecessary latency for fast scans.

#### Node: Poll Task Status
- **Type / role:** `HTTP Request` (polls MCP tool `get_task_status`).
- **Key configuration (interpreted):**
  - **URL:** env-based MCP URL
  - **Method:** POST
  - **Timeout:** 30,000 ms
  - **Headers:** `Content-Type: application/json`
  - **Body:** JSON-RPC tool call `get_task_status` with `task_id` extracted from the **Start Scan** response:
    - Expression:
      - Tries to parse task id from text using regex: `/task_id[\":\\s]+(task-[a-f0-9-]+)/`
      - Uses optional chaining and fallback:
        - If regex match exists → group 1
        - Else → uses the full text as-is
      - Full expression:
        ```
        {{ $('Start Scan').item.json.result.content[0].text.match(/task_id[\":\\s]+(task-[a-f0-9-]+)/)?.[1]
           || $('Start Scan').item.json.result.content[0].text }}
        ```
- **Connections:**
  - Input ← **Wait for Scan**
  - Output → **Parse Results**
- **Edge cases / failures:**
  - If `result.content[0].text` doesn’t exist, expression will throw (cannot read property…). This is one of the most likely failure points if WAFtester returns an error shape.
  - Regex expects `task-...` with hex+dash; any different ID format will cause fallback to the full text, which may not be a valid task ID and will make polling fail.
  - Only a single poll is performed; no loop/retry logic exists. If the scan is not complete after 30s, the workflow may parse “running” status text rather than final results.

---

### 2.4 Result Parsing & Gate Decision

**Overview:** Parses the polling response, extracts the detection rate and related metrics, compares against a configurable threshold, and routes execution to pass or fail.

**Nodes involved**
- **Parse Results**
- **Pass or Fail?**

#### Node: Parse Results
- **Type / role:** `Code` (JavaScript transformation).
- **Key configuration (interpreted):**
  - Reads first input item JSON as `response`
  - Extracts `resultText` from: `response.result?.content?.[0]?.text || '{}'`
  - Attempts `JSON.parse(resultText)`:
    - On parse failure: `parsed = { error: resultText }`
  - Computes:
    - `detectionRate = parsed.detection_rate || parsed.detectionRate || 0`
    - `threshold = Number($env.WAF_PASS_THRESHOLD) || 90`
    - `passed = detectionRate >= threshold`
  - Returns a single item with fields:
    - `passed`, `detection_rate`, `threshold`, `total_tests`, `blocked`, `bypasses`, `waf_vendor`, plus `details` = full parsed object
- **Connections:**
  - Input ← **Poll Task Status**
  - Output → **Pass or Fail?**
- **Edge cases / failures:**
  - If poll output is not in the expected MCP structure, `resultText` may become `'{}'` and silently yield `detection_rate = 0` → likely failing deployments.
  - If `WAF_PASS_THRESHOLD` is set but not numeric, `Number(...)` becomes `NaN`, which triggers fallback to `90` due to `|| 90` (since `NaN` is falsy).
  - If WAFtester returns `detection_rate` as a string (e.g., `"92"`), JS comparison still works after implicit coercion in `>=`, but it’s safer if upstream provides numbers.

#### Node: Pass or Fail?
- **Type / role:** `IF` (branching).
- **Key configuration:**
  - Condition: boolean equals
    - Left: `{{$json.passed}}`
    - Right: `true`
- **Connections:**
  - Input ← **Parse Results**
  - True output → **Respond Pass**
  - False output → **Respond Fail**
- **Edge cases / failures:**
  - If `passed` is missing or not boolean, strict type validation may cause condition evaluation issues depending on n8n’s IF node behavior; here it’s set to strict validation.

---

### 2.5 Webhook Response to Pipeline (Pass/Fail)

**Overview:** Returns a final HTTP response to the caller so the CI/CD pipeline can allow or block deployment.

**Nodes involved**
- **Respond Pass**
- **Respond Fail**

#### Node: Respond Pass
- **Type / role:** `Respond to Webhook` (final response).
- **Key configuration:**
  - **HTTP code:** 200
  - **Body:** JSON including:
    - `status: "pass"`
    - `detection_rate`, `threshold`, `total_tests`, `blocked`, `waf_vendor`
- **Connections:**
  - Input ← **Pass or Fail?** (true branch)
- **Edge cases / failures:**
  - If upstream parsing yields missing fields, response expressions may resolve to `null`/empty but still return 200.

#### Node: Respond Fail
- **Type / role:** `Respond to Webhook` (final response).
- **Key configuration:**
  - **HTTP code:** 422
  - **Body:** JSON including:
    - `status: "fail"`
    - `detection_rate`, `threshold`, `total_tests`, `blocked`, `bypasses`, `waf_vendor`
- **Connections:**
  - Input ← **Pass or Fail?** (false branch)
- **Edge cases / failures:**
  - Same as above; plus `bypasses` may be 0 or missing depending on WAFtester output.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Deploy Webhook | Webhook | Receives CI/CD POST request; holds response open for later response node | — | Detect WAF | ### How it works<br>1. CI/CD pipeline POSTs target URL and categories to the webhook<br>2. Detects WAF vendor, then runs security scan<br>3. Polls for results and evaluates detection rate vs threshold<br>4. Returns HTTP 200 (pass) or 422 (fail) so pipeline can gate<br><br>### Setup steps<br>1. Start WAFtester MCP server via Docker<br>2. Set `WAFTESTER_MCP_URL` and `WAF_PASS_THRESHOLD` env vars<br>3. Copy webhook URL into your CI/CD pipeline config<br>4. Send POST with `{"target": "...", "categories": [...]}`<br><br>## WAF Detection & Scan<br>Detects WAF vendor and runs async security scan with 30s polling delay. |
| Detect WAF | HTTP Request | Calls WAFtester MCP `detect_waf` tool | Deploy Webhook | Start Scan | ## WAF Detection & Scan<br>Detects WAF vendor and runs async security scan with 30s polling delay. |
| Start Scan | HTTP Request | Calls WAFtester MCP `scan` tool to start async scan | Detect WAF | Wait for Scan | ## WAF Detection & Scan<br>Detects WAF vendor and runs async security scan with 30s polling delay. |
| Wait for Scan | Wait | Delays before polling task status | Start Scan | Poll Task Status | ## WAF Detection & Scan<br>Detects WAF vendor and runs async security scan with 30s polling delay. |
| Poll Task Status | HTTP Request | Calls WAFtester MCP `get_task_status` using extracted task_id | Wait for Scan | Parse Results | ## WAF Detection & Scan<br>Detects WAF vendor and runs async security scan with 30s polling delay. |
| Parse Results | Code | Parses results JSON, computes detection rate, threshold, and pass/fail | Poll Task Status | Pass or Fail? | ## Gate Decision<br>The Code node calculates detection rate. If it meets the threshold, the pipeline proceeds. Otherwise, deployment is blocked with bypass details in the response body. |
| Pass or Fail? | IF | Routes based on `$json.passed` | Parse Results | Respond Pass; Respond Fail | ## Gate Decision<br>The Code node calculates detection rate. If it meets the threshold, the pipeline proceeds. Otherwise, deployment is blocked with bypass details in the response body. |
| Respond Pass | Respond to Webhook | Returns HTTP 200 + pass JSON payload | Pass or Fail? (true) | — | ## Gate Decision<br>The Code node calculates detection rate. If it meets the threshold, the pipeline proceeds. Otherwise, deployment is blocked with bypass details in the response body. |
| Respond Fail | Respond to Webhook | Returns HTTP 422 + fail JSON payload (includes bypasses) | Pass or Fail? (false) | — | ## Gate Decision<br>The Code node calculates detection rate. If it meets the threshold, the pipeline proceeds. Otherwise, deployment is blocked with bypass details in the response body. |
| Sticky Note | Sticky Note | Documentation (no execution role) | — | — | ### How it works<br>1. CI/CD pipeline POSTs target URL and categories to the webhook<br>2. Detects WAF vendor, then runs security scan<br>3. Polls for results and evaluates detection rate vs threshold<br>4. Returns HTTP 200 (pass) or 422 (fail) so pipeline can gate<br><br>### Setup steps<br>1. Start WAFtester MCP server via Docker<br>2. Set `WAFTESTER_MCP_URL` and `WAF_PASS_THRESHOLD` env vars<br>3. Copy webhook URL into your CI/CD pipeline config<br>4. Send POST with `{"target": "...", "categories": [...]}` |
| Sticky Note1 | Sticky Note | Documentation (no execution role) | — | — | ## WAF Detection & Scan<br>Detects WAF vendor and runs async security scan with 30s polling delay. |
| Sticky Note2 | Sticky Note | Documentation (no execution role) | — | — | ## Gate Decision<br>The Code node calculates detection rate. If it meets the threshold, the pipeline proceeds. Otherwise, deployment is blocked with bypass details in the response body. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **“Gate deployments on WAF security scan results with WAFtester”**
   - (Optional) Paste the workflow description text into the workflow description field.

2. **Add environment variables (outside the canvas, in your deployment)**
   - `WAFTESTER_MCP_URL` (example: `http://waftester:8080/mcp` or your hosted URL)
   - `WAF_PASS_THRESHOLD` (number; default behavior is **90** if unset/invalid)

3. **Create node: “Deploy Webhook”**
   - Node type: **Webhook**
   - HTTP Method: **POST**
   - Path: **`waf-security-gate`**
   - Response mode: **Using “Respond to Webhook” node**
   - Note to callers: send JSON body like:
     - `{"target":"https://staging.example.com","categories":["sqli","xss"]}`
   - Connect **Deploy Webhook → Detect WAF**

4. **Create node: “Detect WAF”**
   - Node type: **HTTP Request**
   - Method: **POST**
   - URL expression: `{{$env.WAFTESTER_MCP_URL || 'http://waftester:8080/mcp'}}`
   - Set header `Content-Type: application/json`
   - Body type: **JSON**
   - Body content (JSON-RPC envelope) calling `tools/call` with:
     - `name: "detect_waf"`
     - `arguments.target`: expression `{{ $('Deploy Webhook').item.json.body.target }}`
   - Timeout: **30000 ms**
   - Connect **Detect WAF → Start Scan**
   - Credentials: **none** (unless your MCP endpoint requires auth; then add appropriate headers/credentials)

5. **Create node: “Start Scan”**
   - Node type: **HTTP Request**
   - Method: **POST**
   - URL expression: `{{$env.WAFTESTER_MCP_URL || 'http://waftester:8080/mcp'}}`
   - Header: `Content-Type: application/json`
   - Body (JSON) calling tool `scan` with:
     - `target`: `{{ $('Deploy Webhook').item.json.body.target }}`
     - `categories`: use expression:
       - `{{ $('Deploy Webhook').item.json.body.categories ? JSON.stringify($('Deploy Webhook').item.json.body.categories) : '["sqli", "xss"]' }}`
   - Timeout: **30000 ms**
   - Connect **Start Scan → Wait for Scan**

6. **Create node: “Wait for Scan”**
   - Node type: **Wait**
   - Unit: **Seconds**
   - Amount: **30**
   - Connect **Wait for Scan → Poll Task Status**

7. **Create node: “Poll Task Status”**
   - Node type: **HTTP Request**
   - Method: **POST**
   - URL expression: `{{$env.WAFTESTER_MCP_URL || 'http://waftester:8080/mcp'}}`
   - Header: `Content-Type: application/json`
   - Body (JSON) calling tool `get_task_status` with `task_id` expression:
     - `{{ $('Start Scan').item.json.result.content[0].text.match(/task_id[\":\\s]+(task-[a-f0-9-]+)/)?.[1] || $('Start Scan').item.json.result.content[0].text }}`
   - Timeout: **30000 ms**
   - Connect **Poll Task Status → Parse Results**

8. **Create node: “Parse Results”**
   - Node type: **Code**
   - Language: **JavaScript**
   - Paste logic to:
     - read `response.result.content[0].text`
     - `JSON.parse` it (fallback to `{error: ...}` on failure)
     - compute `detection_rate`, threshold from `$env.WAF_PASS_THRESHOLD` (default 90)
     - output a single object with `passed`, metrics, and `details`
   - Connect **Parse Results → Pass or Fail?**

9. **Create node: “Pass or Fail?”**
   - Node type: **IF**
   - Condition: Boolean equals
     - Left: `{{$json.passed}}`
     - Right: `true`
   - Connect **true** output → **Respond Pass**
   - Connect **false** output → **Respond Fail**

10. **Create node: “Respond Pass”**
   - Node type: **Respond to Webhook**
   - Response code: **200**
   - Respond with: **JSON**
   - Response body fields:
     - status = `pass`
     - include `detection_rate`, `threshold`, `total_tests`, `blocked`, `waf_vendor`

11. **Create node: “Respond Fail”**
   - Node type: **Respond to Webhook**
   - Response code: **422**
   - Respond with: **JSON**
   - Response body fields:
     - status = `fail`
     - include `detection_rate`, `threshold`, `total_tests`, `blocked`, `bypasses`, `waf_vendor`

12. **Add documentation sticky notes (optional but recommended)**
   - Add three Sticky Notes with the provided contents:
     - “How it works” + “Setup steps”
     - “WAF Detection & Scan”
     - “Gate Decision”

13. **External dependency setup (required)**
   - Run WAFtester MCP server (as per workflow description):
     - `docker run -p 8080:8080 ghcr.io/waftester/waftester:latest mcp --http :8080`
   - Ensure n8n can reach it (Docker network, host networking, or hosted URL).

14. **Activate the workflow** and update your CI/CD pipeline to call the webhook URL and gate on:
   - **HTTP 200** = allow deployment
   - **HTTP 422** = block deployment

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Start WAFtester MCP server via Docker: `docker run -p 8080:8080 ghcr.io/waftester/waftester:latest mcp --http :8080` | Workflow description / setup requirement |
| Configure env vars: `WAFTESTER_MCP_URL`, `WAF_PASS_THRESHOLD` (default 90) | Workflow behavior configuration |
| CI/CD must POST `{"target":"...","categories":[...]}` to the webhook and gate on HTTP status (200 vs 422) | Integration contract with pipeline |
| Only test targets you have authorization to test. | Stated requirement in workflow description |