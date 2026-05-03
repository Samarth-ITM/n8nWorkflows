Run weekly WAF security audits with WAFtester and Slack alerts

https://n8nworkflows.xyz/workflows/run-weekly-waf-security-audits-with-waftester-and-slack-alerts-13444


# Run weekly WAF security audits with WAFtester and Slack alerts

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Run weekly WAF security audits with WAFtester and Slack alerts  
**Workflow name (in JSON):** Run scheduled WAF security audits with WAFtester and Slack  
**Purpose:** Automatically run a weekly security assessment against a WAF-protected target using the **WAFtester MCP server** (JSON-RPC over HTTP), then post either a detailed alert or a pass confirmation to **Slack**, based on the resulting “grade”.

**Primary use cases**
- Continuous monitoring of WAF effectiveness (detect drift / misconfigurations)
- Security/DevOps compliance evidence (regular checks)
- Rapid notification when protection quality drops

**Logical blocks**
1. **1.1 Scheduling / Entry Point** – starts every Monday at 03:00.
2. **1.2 WAF Identification (fingerprinting)** – calls WAFtester “detect_waf”.
3. **1.3 Async Security Assessment (launch + wait + poll)** – starts an async assessment, waits 30s, then fetches results by task id.
4. **1.4 Decision & Slack Reporting** – checks whether grade is acceptable (“Grade: A”) and posts to Slack accordingly.
5. **1.5 Documentation (Sticky Notes)** – embedded operational guidance.

---

## 2. Block-by-Block Analysis

### 2.1 Scheduling / Entry Point

**Overview:** Triggers the workflow on a weekly cadence (Mondays at 3 AM). This is the only entry point.

**Nodes involved**
- **Weekly Schedule**

#### Node: Weekly Schedule
- **Type / role:** `Schedule Trigger` – time-based trigger.
- **Configuration (interpreted):**
  - Runs **weekly**
  - **Day:** Monday (day index `1` in node config)
  - **Hour:** `03:00`
- **Key expressions / variables:** none.
- **Connections:**
  - **Output →** Detect WAF
- **Version notes:** typeVersion **1.2**.
- **Edge cases / failures:**
  - Workflow will not run if inactive.
  - Server timezone affects “3 AM” unless n8n instance timezone is configured accordingly.

---

### 2.2 WAF Detection & Assessment Kickoff

**Overview:** First fingerprints the WAF vendor, then starts an **async** assessment job with specified attack categories and performance parameters.

**Nodes involved**
- **Detect WAF**
- **Start Assessment**

#### Node: Detect WAF
- **Type / role:** `HTTP Request` – JSON-RPC call to WAFtester MCP method `tools/call` using tool `detect_waf`.
- **Configuration (interpreted):**
  - **Method:** POST
  - **URL:** uses environment variable with fallback  
    - `{{$env.WAFTESTER_MCP_URL || 'http://waftester:8080/mcp'}}`
  - **Timeout:** 30s
  - **Headers:** `Content-Type: application/json`
  - **Body:** JSON-RPC 2.0 payload calling tool:
    - tool name: `detect_waf`
    - arguments: `{ "target": "{{ $env.WAF_TARGET_URL }}" }`
- **Key expressions / variables:**
  - `$env.WAFTESTER_MCP_URL` (optional)
  - `$env.WAF_TARGET_URL` (required to be set in environment)
- **Connections:**
  - **Input ←** Weekly Schedule
  - **Output →** Start Assessment
- **Version notes:** typeVersion **4.2**.
- **Edge cases / failures:**
  - Missing `WAF_TARGET_URL` → tool likely fails or returns invalid result.
  - WAFtester MCP unreachable (DNS, container not running, network policy) → request error/timeout.
  - Non-JSON response / JSON-RPC error response → downstream expressions may break if expected fields absent.

#### Node: Start Assessment
- **Type / role:** `HTTP Request` – starts an async WAF assessment via WAFtester tool `assess`.
- **Configuration (interpreted):**
  - **Method:** POST
  - **URL:** `{{$env.WAFTESTER_MCP_URL || 'http://waftester:8080/mcp'}}`
  - **Timeout:** 60s
  - **Headers:** `Content-Type: application/json`
  - **Body:** JSON-RPC call:
    - tool: `assess`
    - arguments include:
      - `target`: `{{ $env.WAF_TARGET_URL }}`
      - `categories`: `["sqli","xss","traversal","cmdi","ssrf"]`
      - `rate_limit`: `20`
      - `concurrency`: `10`
  - **Expected outcome:** returns a **task_id** (used later for polling).
- **Key expressions / variables:**
  - `$env.WAF_TARGET_URL`
  - `$env.WAFTESTER_MCP_URL`
- **Connections:**
  - **Input ←** Detect WAF
  - **Output →** Wait for Assessment
- **Version notes:** typeVersion **4.2**.
- **Edge cases / failures:**
  - If WAFtester returns an error or does not return a task id in the expected path, polling will fail.
  - Target rate limiting / WAF blocking may cause assessment to be incomplete or degrade grade.
  - Concurrency/rate may be too high for target → false negatives/instability; too low → task may not finish within 30s wait.

---

### 2.3 Async Completion Handling (Wait + Poll)

**Overview:** Waits a fixed interval for async assessment completion, then polls WAFtester for results using the task id produced by Start Assessment.

**Nodes involved**
- **Wait for Assessment**
- **Poll Task Status**

#### Node: Wait for Assessment
- **Type / role:** `Wait` – delays execution before polling.
- **Configuration (interpreted):**
  - Wait **30 seconds**
- **Connections:**
  - **Input ←** Start Assessment
  - **Output →** Poll Task Status
- **Version notes:** typeVersion **1.1**.
- **Edge cases / failures:**
  - Fixed 30s may be insufficient; poll may return “running”/partial results depending on WAFtester behavior.
  - If the assessment takes longer, workflow has no retry loop (single wait + single poll).

#### Node: Poll Task Status
- **Type / role:** `HTTP Request` – JSON-RPC call to WAFtester tool `get_task_status`.
- **Configuration (interpreted):**
  - **Method:** POST
  - **URL:** `{{$env.WAFTESTER_MCP_URL || 'http://waftester:8080/mcp'}}`
  - **Timeout:** 30s
  - **Headers:** `Content-Type: application/json`
  - **Body:** JSON-RPC call:
    - tool: `get_task_status`
    - argument `task_id` is read from the **incoming JSON**:
      - `task_id`: `{{ $json.result.content[0].text }}`
- **Key expressions / variables:**
  - `{{ $json.result.content[0].text }}` (expects Start Assessment output structure to contain task id at this path)
- **Connections:**
  - **Input ←** Wait for Assessment (and thus Start Assessment)
  - **Output →** Check Results
- **Version notes:** typeVersion **4.2**.
- **Edge cases / failures:**
  - **Expression path fragility:** if `Start Assessment` returns task_id in a different field, this expression fails or sends wrong task id.
  - If status is not ready, result text may not contain “Grade: …”, causing decision logic to misroute.
  - HTTP/JSON-RPC errors should be handled (currently not); may stop workflow.

---

### 2.4 Decision & Slack Reporting

**Overview:** Evaluates the assessment “grade” and posts either a detailed alert (if below threshold) or a brief “passed” message to Slack.

**Nodes involved**
- **Check Results**
- **Slack Alert**
- **Slack OK**

#### Node: Check Results
- **Type / role:** `IF` – branching based on grade text.
- **Configuration (interpreted):**
  - Condition checks the polled results string:
    - **Left value:** `{{ $json.result.content[0].text }}`
    - **Operator:** `notContains`
    - **Right value:** `Grade: A`
  - **Meaning:**
    - **True branch:** grade text does **not** contain `"Grade: A"` → alert
    - **False branch:** contains `"Grade: A"` → OK message
- **Connections:**
  - **Input ←** Poll Task Status
  - **True →** Slack Alert
  - **False →** Slack OK
- **Version notes:** typeVersion **2.2**.
- **Edge cases / failures:**
  - If results are empty / status still running → may not contain “Grade: A” → triggers alert (potential false positive).
  - If WAFtester output formatting changes (e.g., “Grade A” without colon) → misrouting.
  - If you want threshold “A-” etc., you must adjust the condition logic.

#### Node: Slack Alert
- **Type / role:** `Slack` – posts detailed report to a channel.
- **Configuration (interpreted):**
  - **Message type:** text
  - **Channel selection by name:**  
    `{{ $env.SLACK_CHANNEL || '#security-alerts' }}`
  - **Message content includes:**
    - Target: `{{$env.WAF_TARGET_URL}}`
    - Timestamp: `{{$now.format('yyyy-MM-dd HH:mm')}}`
    - Assessment results pulled from **Poll Task Status** node output:
      - `{{ $('Poll Task Status').item.json.result.content[0].text }}`
    - WAF detection output from **Detect WAF** node:
      - `{{ $('Detect WAF').item.json.result.content[0].text }}`
- **Credentials:** Slack OAuth2 API (must be configured in n8n and selected here).
- **Connections:**
  - **Input ←** Check Results (true branch)
  - **Output:** none
- **Version notes:** typeVersion **2.2**.
- **Edge cases / failures:**
  - Slack credential missing/expired scopes → auth error.
  - Channel name not found or bot not in channel → posting failure.
  - If referenced node output paths don’t exist → message renders blanks or throws expression error (depends on n8n settings).

#### Node: Slack OK
- **Type / role:** `Slack` – posts a short “passed” confirmation.
- **Configuration (interpreted):**
  - **Channel:** `{{ $env.SLACK_CHANNEL || '#security-alerts' }}`
  - **Text includes:**
    - `WAF Audit Passed - <target> - <date>`
    - Includes `Poll Task Status` text:
      - `{{ $('Poll Task Status').item.json.result.content[0].text }}`
- **Credentials:** Slack OAuth2 API (same requirement as Slack Alert).
- **Connections:**
  - **Input ←** Check Results (false branch)
  - **Output:** none
- **Version notes:** typeVersion **2.2**.
- **Edge cases / failures:** same as Slack Alert.

---

### 2.5 Embedded Documentation (Sticky Notes)

**Overview:** Three sticky notes provide operational guidance, setup commands, environment variables, and customization points. They don’t affect execution but are important for reproducibility.

**Nodes involved**
- Sticky Note
- Sticky Note1
- Sticky Note2

#### Node: Sticky Note
- **Type / role:** `Sticky Note` – documentation.
- **Content highlights:**
  - Explains node sequence and purpose
  - Setup:
    - Docker command to run WAFtester MCP server  
      `docker run -p 8080:8080 ghcr.io/waftester/waftester:latest mcp --http :8080`
    - Env vars: `WAF_TARGET_URL`, `WAFTESTER_MCP_URL`, `SLACK_CHANNEL`
    - Slack credentials setup path in n8n
  - Customization: schedule, grade threshold, categories

#### Node: Sticky Note1
- **Type / role:** `Sticky Note` – documentation.
- **Content:** “WAF Detection & Assessment” summary.

#### Node: Sticky Note2
- **Type / role:** `Sticky Note` – documentation.
- **Content:** “Results & Alerting” summary.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Weekly Schedule | Schedule Trigger | Weekly entry point (Mon 3 AM) | — | Detect WAF | ### How it works  / Runs a WAF security assessment every Monday at 3 AM and posts results to Slack. / 1. **Weekly Schedule** triggers the workflow / 2. **Detect WAF** identifies the vendor via fingerprinting / 3. **Start Assessment** fires test payloads (SQLi, XSS, traversal, cmdi, SSRF) / 4. **Wait** 30 seconds for async completion / 5. **Poll Task Status** retrieves the WAF grade / 6. **Check Results** routes to alert or confirmation / 7. **Slack** posts results to your security channel / ### Setup steps / 1. Start WAFtester MCP server: `docker run -p 8080:8080 ghcr.io/waftester/waftester:latest mcp --http :8080` / 2. Set environment variables: `WAF_TARGET_URL`, `WAFTESTER_MCP_URL`, `SLACK_CHANNEL` / 3. Add Slack credentials: **Settings → Credentials → New → Slack OAuth2 API** / 4. Select the credential in both Slack nodes / 5. Activate the workflow / ### Customization tips / Change schedule; change grade threshold; add categories |
| Detect WAF | HTTP Request | Call WAFtester `detect_waf` via JSON-RPC | Weekly Schedule | Start Assessment | ## WAF Detection & Assessment / Detects WAF vendor, then runs async security assessment with 30s polling delay. |
| Start Assessment | HTTP Request | Start async WAF assessment (`assess`) | Detect WAF | Wait for Assessment | ## WAF Detection & Assessment / Detects WAF vendor, then runs async security assessment with 30s polling delay. |
| Wait for Assessment | Wait | Delay before polling async task | Start Assessment | Poll Task Status | ## WAF Detection & Assessment / Detects WAF vendor, then runs async security assessment with 30s polling delay. |
| Poll Task Status | HTTP Request | Poll WAFtester `get_task_status` using task_id | Wait for Assessment | Check Results | ## WAF Detection & Assessment / Detects WAF vendor, then runs async security assessment with 30s polling delay. |
| Check Results | IF | Route based on grade threshold (“Grade: A”) | Poll Task Status | Slack Alert; Slack OK | ## Results & Alerting / Routes to Slack alert or confirmation based on the WAF grade. |
| Slack Alert | Slack | Post full report when below threshold | Check Results (true) | — | ## Results & Alerting / Routes to Slack alert or confirmation based on the WAF grade. |
| Slack OK | Slack | Post confirmation when passing | Check Results (false) | — | ## Results & Alerting / Routes to Slack alert or confirmation based on the WAF grade. |
| Sticky Note | Sticky Note | Documentation / setup instructions | — | — |  |
| Sticky Note1 | Sticky Note | Documentation: detection & assessment | — | — |  |
| Sticky Note2 | Sticky Note | Documentation: results & alerting | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **Run scheduled WAF security audits with WAFtester and Slack** (or your preferred name).
   - Ensure workflow is **inactive** until credentials/env are set.

2. **Add the trigger**
   - Add node: **Schedule Trigger**
   - Name: **Weekly Schedule**
   - Configure rule: **Weekly**
     - Day: **Monday**
     - Time: **03:00**
   - Connect: **Weekly Schedule → Detect WAF** (created next)

3. **Add HTTP Request: Detect WAF**
   - Add node: **HTTP Request**
   - Name: **Detect WAF**
   - Method: **POST**
   - URL expression: `{{$env.WAFTESTER_MCP_URL || 'http://waftester:8080/mcp'}}`
   - Response: default (JSON)
   - Options: set **Timeout** to **30000 ms**
   - Headers: `Content-Type = application/json`
   - Body type: **JSON**
   - JSON body content (JSON-RPC):
     - `jsonrpc: "2.0"`, `id: 1`, `method: "tools/call"`
     - `params.name: "detect_waf"`
     - `params.arguments.target: "{{ $env.WAF_TARGET_URL }}"`
   - Connect: **Detect WAF → Start Assessment**

4. **Add HTTP Request: Start Assessment**
   - Add node: **HTTP Request**
   - Name: **Start Assessment**
   - Method: **POST**
   - URL expression: `{{$env.WAFTESTER_MCP_URL || 'http://waftester:8080/mcp'}}`
   - Timeout: **60000 ms**
   - Headers: `Content-Type = application/json`
   - Body type: **JSON**
   - JSON-RPC body:
     - `id: 2`, tool `assess`
     - arguments:
       - `target: "{{ $env.WAF_TARGET_URL }}"`
       - `categories: ["sqli","xss","traversal","cmdi","ssrf"]`
       - `rate_limit: 20`
       - `concurrency: 10`
   - Connect: **Start Assessment → Wait for Assessment**

5. **Add Wait**
   - Add node: **Wait**
   - Name: **Wait for Assessment**
   - Unit: **Seconds**
   - Amount: **30**
   - Connect: **Wait for Assessment → Poll Task Status**

6. **Add HTTP Request: Poll Task Status**
   - Add node: **HTTP Request**
   - Name: **Poll Task Status**
   - Method: **POST**
   - URL expression: `{{$env.WAFTESTER_MCP_URL || 'http://waftester:8080/mcp'}}`
   - Timeout: **30000 ms**
   - Headers: `Content-Type = application/json`
   - Body type: **JSON**
   - JSON-RPC body:
     - `id: 3`, tool `get_task_status`
     - arguments:
       - `task_id: "{{ $json.result.content[0].text }}"`
       - (This assumes the previous node returns the task id at that path.)
   - Connect: **Poll Task Status → Check Results**

7. **Add IF: Check Results**
   - Add node: **IF**
   - Name: **Check Results**
   - Condition:
     - Left value (expression): `{{ $json.result.content[0].text }}`
     - Operator: **String → does not contain**
     - Right value: `Grade: A`
   - Connect outputs:
     - **True → Slack Alert**
     - **False → Slack OK**

8. **Add Slack node: Alert**
   - Add node: **Slack**
   - Name: **Slack Alert**
   - Operation/message type: **Post message (text)**
   - Channel selection: **by name**
     - Channel expression: `{{$env.SLACK_CHANNEL || '#security-alerts'}}`
   - Text field (compose using expressions):
     - Include target: `{{$env.WAF_TARGET_URL}}`
     - Include assessment text: `{{ $('Poll Task Status').item.json.result.content[0].text }}`
     - Include detection text: `{{ $('Detect WAF').item.json.result.content[0].text }}`
     - Include date: `{{$now.format('yyyy-MM-dd HH:mm')}}`
   - **Credentials:** select/create **Slack OAuth2 API** credential (see step 10).

9. **Add Slack node: OK**
   - Add node: **Slack**
   - Name: **Slack OK**
   - Channel: `{{$env.SLACK_CHANNEL || '#security-alerts'}}`
   - Text includes:
     - `WAF Audit Passed - {{$env.WAF_TARGET_URL}} - {{$now.format('yyyy-MM-dd')}}`
     - `{{ $('Poll Task Status').item.json.result.content[0].text }}`
   - Use the **same Slack OAuth2 credential**.

10. **Configure credentials and environment**
   - **Slack OAuth2**
     - In n8n: **Settings → Credentials → New → Slack OAuth2 API**
     - Authorize a Slack app/bot with permission to post messages to the target channel(s).
     - Ensure the bot is invited to the channel if required by Slack workspace settings.
   - **Environment variables (required/optional)**
     - `WAF_TARGET_URL` (required): the URL to assess (only with authorization).
     - `WAFTESTER_MCP_URL` (optional): defaults to `http://waftester:8080/mcp`
     - `SLACK_CHANNEL` (optional): defaults to `#security-alerts`

11. **Start WAFtester MCP server**
   - Run (as provided in the note):
     - `docker run -p 8080:8080 ghcr.io/waftester/waftester:latest mcp --http :8080`
   - Ensure n8n can reach that host/port (Docker network considerations if both are containerized).

12. **Activate workflow**
   - Run a manual test first (temporarily change Schedule Trigger to “Every minute” or use manual execution with a different trigger) and confirm:
     - Detect WAF returns content
     - Start Assessment returns a usable task id at the expected expression path
     - Poll returns a text that contains “Grade: …”
     - Slack posts successfully
   - Revert schedule to weekly and activate.

**Sub-workflow setup:** none (no Execute Workflow / Sub-workflow nodes present).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Runs a WAF security assessment every Monday at 3 AM and posts results to Slack; includes detection, async assessment, waiting, polling, grade routing, Slack output. | Embedded “How it works” sticky note |
| Start WAFtester MCP server: `docker run -p 8080:8080 ghcr.io/waftester/waftester:latest mcp --http :8080` | Embedded setup command |
| Required/optional environment variables: `WAF_TARGET_URL` (required), `WAFTESTER_MCP_URL` (default `http://waftester:8080/mcp`), `SLACK_CHANNEL` (default `#security-alerts`) | Embedded setup steps |
| Customize: schedule, grade threshold (“Grade: A”), add categories in `categories` array | Embedded customization tips |
| Authorization reminder: only test targets you are authorized to test. | Workflow description |