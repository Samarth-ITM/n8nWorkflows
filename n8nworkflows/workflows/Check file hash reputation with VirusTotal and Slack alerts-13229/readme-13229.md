Check file hash reputation with VirusTotal and Slack alerts

https://n8nworkflows.xyz/workflows/check-file-hash-reputation-with-virustotal-and-slack-alerts-13229


# Check file hash reputation with VirusTotal and Slack alerts

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Check file hash reputation with VirusTotal and Slack alerts  
**Workflow name (n8n):** File Hash Reputation Checker  
**Purpose:** Accept a file hash (MD5/SHA1/SHA256) over an HTTP webhook, validate/normalize it, query VirusTotal for reputation, compute a verdict (Malicious/Suspicious/Clean/Unknown), and notify a Slack channel (with a special alert when malicious). The workflow also returns a JSON response to the webhook caller.

### 1.1 Input Reception & Normalization
Receives the incoming request and extracts/normalizes the hash from the webhook payload (`body.text`), lowercasing and trimming it.

### 1.2 Hash Validation
Validates that the provided hash length matches MD5 (32), SHA1 (40), or SHA256 (64). Invalid input returns an error response immediately.

### 1.3 VirusTotal Lookup
Performs a VirusTotal v3 `/files/{hash}` lookup via HTTP Request using a VirusTotal API credential. If VT does not have the hash (or an API error occurs), returns “Unknown”.

### 1.4 Verdict Calculation
Computes detection totals and sets the verdict based on VirusTotal `last_analysis_stats`, plus optionally extracts a malware family label.

### 1.5 Alerts & Responses
If verdict is “Malicious”, posts a high-priority Slack alert. In all cases that reached verdict computation, the webhook gets a JSON response; afterwards a Slack status message is also posted.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception & Normalization
**Overview:** Entry point via webhook; normalizes the incoming hash for consistent downstream validation and lookup.  
**Nodes involved:** `Receive File Hash`, `Normalize Hash from Webhook`

#### Node: Receive File Hash
- **Type / role:** `Webhook` — workflow trigger and HTTP input endpoint.
- **Configuration (interpreted):**
  - **HTTP Method:** `POST`
  - **Path:** `/hash-check`
  - **Response Mode:** `responseNode` (execution expects a Respond to Webhook node to return HTTP response).
- **Key expectations / variables:**
  - Expects a payload containing `body.text` (string). (Sticky note mentions `{ "text": "<file_hash>" }`, but node expressions actually read `body.text`.)
- **Connections:**
  - Output → `Normalize Hash from Webhook`
- **Edge cases / failures:**
  - Missing `body.text` will cause the next Set node expression to fail (`toLowerCase()` on undefined).
  - If multiple sources post different schema (e.g., `{hash: ...}`), normalization must be adapted.

#### Node: Normalize Hash from Webhook
- **Type / role:** `Set` — transforms input into a single canonical field `hash`.
- **Configuration (interpreted):**
  - Creates field `hash` = `{{$json.body.text.toLowerCase().trim()}}`
- **Connections:**
  - Input ← `Receive File Hash`
  - Output → `IF - Valid Hash Format`
- **Edge cases / failures:**
  - If `body.text` is not a string (or absent), the expression throws.
  - If the incoming hash includes surrounding quotes or other characters, they are not removed (only trimmed). Consider stripping non-hex characters if needed.

---

### Block 2 — Hash Validation
**Overview:** Ensures the hash is plausibly MD5/SHA1/SHA256 based on length; rejects everything else early.  
**Nodes involved:** `IF - Valid Hash Format`, `Respond - Invalid Hash`

#### Node: IF - Valid Hash Format
- **Type / role:** `IF` — branching based on hash length.
- **Configuration (interpreted):**
  - Uses **OR** conditions:
    - `hash.length == 32` (MD5)
    - `hash.length == 40` (SHA1)
    - `hash.length == 64` (SHA256)
  - Implemented by converting length to string (`$json.hash.length.toString()`) and comparing to `"32"`, `"40"`, `"64"`.
- **Connections:**
  - **True** → `HTTP - VirusTotal Hash Lookup`
  - **False** → `Respond - Invalid Hash`
- **Edge cases / failures:**
  - Length-only validation allows non-hex content (e.g., 32 characters not in `[0-9a-f]`).
  - If `hash` is missing, expression fails.

#### Node: Respond - Invalid Hash
- **Type / role:** `Respond to Webhook` — returns an error response to the caller.
- **Configuration (interpreted):**
  - Returns JSON:
    - `status: "error"`
    - `message: "Invalid hash format. Provide MD5, SHA1, or SHA256."`
- **Connections:**
  - Input ← `IF - Valid Hash Format` (false branch)
  - No outputs (ends webhook response path)
- **Edge cases / failures:**
  - Because the webhook is in `responseNode` mode, this node must execute to complete the HTTP request for invalid input. If removed, requests may hang until timeout.

---

### Block 3 — VirusTotal Lookup
**Overview:** Queries VirusTotal for the hash reputation and branches depending on whether the hash is found / response is usable.  
**Nodes involved:** `HTTP - VirusTotal Hash Lookup`, `IF - Hash Exists in VT`, `Respond - Unknown Hash`

#### Node: HTTP - VirusTotal Hash Lookup
- **Type / role:** `HTTP Request` — calls VirusTotal API to fetch file report.
- **Configuration (interpreted):**
  - **Method:** GET
  - **URL:** `https://www.virustotal.com/api/v3/files/{{ $json.hash }}`
  - **Authentication:** predefined credential type `virusTotalApi`
  - **Send headers:** enabled (credential injects required API header, typically `x-apikey`)
  - **Error handling:** `onError: continueRegularOutput`
    - Important: even on HTTP error, it continues and outputs an item that includes an `error` object.
- **Connections:**
  - Input ← `IF - Valid Hash Format` (true branch)
  - Output → `IF - Hash Exists in VT`
- **Edge cases / failures:**
  - Invalid/expired VT API key → HTTP 401/403; because `continueRegularOutput`, flow goes to “Unknown Hash” branch.
  - Rate limiting (429) similarly becomes “Unknown” unless explicitly handled.
  - Network timeouts or VT downtime: same behavior.
  - VirusTotal response shape changes could break downstream code node.

#### Node: IF - Hash Exists in VT
- **Type / role:** `IF` — determines whether VT lookup succeeded.
- **Configuration (interpreted):**
  - Condition: `{{$json.error === undefined}}` must be true to treat as “exists”.
  - This relies on the HTTP node’s “continue on error” behavior.
- **Connections:**
  - **True** → `Function - Calculate Verdict`
  - **False** → `Respond - Unknown Hash`
- **Edge cases / failures:**
  - If VT returns a non-error payload but still not found in a different format, this check may misclassify.
  - Some HTTP errors may not populate `error` as expected depending on n8n version/HTTP node behavior.

#### Node: Respond - Unknown Hash
- **Type / role:** `Respond to Webhook` — returns “Unknown” when VT lookup fails or hash not found.
- **Configuration (interpreted):**
  - Returns JSON using expression:
    - `verdict: "Unknown"`
    - `message: "{{ $json.error.message }}"`
- **Connections:**
  - Input ← `IF - Hash Exists in VT` (false branch)
  - No outputs (ends webhook response path)
- **Edge cases / failures:**
  - If `$json.error.message` is absent, the message field may be empty or expression may evaluate to `undefined`.
  - This does not differentiate “not found” vs “auth/rate limit”; caller receives generic “Unknown”.

---

### Block 4 — Verdict Calculation
**Overview:** Converts VirusTotal analysis stats into a simplified verdict and detection counts, plus optional threat label extraction.  
**Nodes involved:** `Function - Calculate Verdict`

#### Node: Function - Calculate Verdict
- **Type / role:** `Code` — computes verdict and normalizes output structure.
- **Configuration (interpreted):**
  - Reads:
    - `$json.data.attributes.last_analysis_stats` (object of counts, e.g. malicious/suspicious/undetected/harmless/timeout)
  - Computes:
    - `malicious = stats.malicious || 0`
    - `suspicious = stats.suspicious || 0`
    - `total = sum(Object.values(stats))`
  - Verdict logic:
    - If `malicious > 0` → `Malicious`
    - Else if `suspicious > 0` → `Suspicious`
    - Else → `Clean`
  - Output JSON fields:
    - `hash: $json.data.id`
    - `verdict, malicious, suspicious, total`
    - `malware_family` from `popular_threat_classification.suggested_threat_label` or `"N/A"`
- **Connections:**
  - Input ← `IF - Hash Exists in VT` (true branch)
  - Output → `IF - Malicious Verdict`
- **Edge cases / failures:**
  - If VT returns an unexpected schema (missing `data.attributes.last_analysis_stats`), code will throw.
  - “Total” sums all stat categories; if VT adds new categories, total changes (usually OK).
  - Verdict thresholds are simplistic; one malicious detection may be a false positive depending on engines.

---

### Block 5 — Alerts & Responses
**Overview:** Routes malicious results to a dedicated Slack alert; returns JSON response for non-malicious path; then posts an informational Slack message summarizing the verdict.  
**Nodes involved:** `IF - Malicious Verdict`, `Slack - Malicious Hash Alert`, `Respond - Clean or Unknown`, `Send a message`

#### Node: IF - Malicious Verdict
- **Type / role:** `IF` — branches on computed verdict.
- **Configuration (interpreted):**
  - Condition checks boolean expression: `{{ $json.verdict === "Malicious" }}`
- **Connections:**
  - **True** → `Slack - Malicious Hash Alert`
  - **False** → `Respond - Clean or Unknown`
- **Edge cases / failures:**
  - If verdict field is missing (code node failure), condition may be false or expression error depending on runtime.

#### Node: Slack - Malicious Hash Alert
- **Type / role:** `Slack` — sends high-priority alert to a channel.
- **Configuration (interpreted):**
  - Posts to channel ID `C0A252GLT70` (cached name: `all-team-sawi`)
  - Message text includes:
    - Hash
    - Detections (`malicious/total`)
    - Family label
  - `includeLinkToWorkflow: false`
  - Uses Slack credential **“Slack account 2”**
- **Connections:**
  - Input ← `IF - Malicious Verdict` (true branch)
  - No further outputs configured (malicious branch does not send a webhook response in this workflow as currently wired).
- **Edge cases / failures:**
  - Slack auth/token scope issues, revoked token, channel access errors.
  - Message uses emoji and Slack formatting; if Slack policies restrict, message might be blocked.
- **Important behavior note:**
  - Because the Webhook trigger uses `responseNode` mode, the malicious branch currently **does not reach a Respond to Webhook node**. This can cause the webhook request to **hang/time out** for malicious hashes unless Slack node is followed by a response node (not present here).

#### Node: Respond - Clean or Unknown
- **Type / role:** `Respond to Webhook` — returns verdict stats for non-malicious path.
- **Configuration (interpreted):**
  - Responds with JSON expression:
    - `hash`, `verdict`, `malicious`, `suspicious`, `detections: "malicious/total"`
- **Connections:**
  - Input ← `IF - Malicious Verdict` (false branch)
  - Output → `Send a message`
- **Edge cases / failures:**
  - If any referenced field is missing, output JSON may contain `null/undefined` or expression evaluation issues.
  - The node name implies “Clean or Unknown”, but it only receives the **non-malicious verdict path** from `Function - Calculate Verdict` (i.e., Clean or Suspicious). “Unknown” is handled earlier by `Respond - Unknown Hash`.

#### Node: Send a message
- **Type / role:** `Slack` — posts a status message for Suspicious/Clean/Unknown.
- **Configuration (interpreted):**
  - Posts to channel ID `C0A252GLT70`
  - Message text is generated via a ternary expression:
    - If verdict is `Suspicious` → “SUSPICIOUS FILE” + suspicious/total
    - Else if verdict is `Clean` → “CLEAN FILE” + 0/total
    - Else → “UNKNOWN FILE” + “Not found in VirusTotal”
  - Uses Slack credential **“Slack account 2”**
- **Connections:**
  - Input ← `Respond - Clean or Unknown`
  - No outputs
- **Edge cases / failures:**
  - This node runs after responding to webhook, so Slack failures won’t block the HTTP response (good), but will still mark execution as failed unless Slack node error handling is changed.
  - “Unknown” branch text exists but is not reachable in current wiring because Unknown is returned by a separate respond node that ends execution.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Receive File Hash | Webhook | Entry point: receive hash via HTTP POST `/hash-check` | — | Normalize Hash from Webhook | ## 1. Webhook Input & Normalization |
| Normalize Hash from Webhook | Set | Normalize `body.text` into `hash` (lowercase/trim) | Receive File Hash | IF - Valid Hash Format | ## 1. Webhook Input & Normalization |
| IF - Valid Hash Format | IF | Validate hash length (32/40/64) | Normalize Hash from Webhook | HTTP - VirusTotal Hash Lookup; Respond - Invalid Hash | ## 2. Hash Validation |
| Respond - Invalid Hash | Respond to Webhook | Return error JSON for invalid hash | IF - Valid Hash Format | — | ## 2. Hash Validation |
| HTTP - VirusTotal Hash Lookup | HTTP Request | Call VirusTotal `/api/v3/files/{hash}` | IF - Valid Hash Format | IF - Hash Exists in VT | ## 3. VirusTotal Lookup |
| IF - Hash Exists in VT | IF | Detect VT lookup success (`error` absent) | HTTP - VirusTotal Hash Lookup | Function - Calculate Verdict; Respond - Unknown Hash | ## 3. VirusTotal Lookup |
| Respond - Unknown Hash | Respond to Webhook | Return verdict “Unknown” with error message | IF - Hash Exists in VT | — | ## 3. VirusTotal Lookup |
| Function - Calculate Verdict | Code | Compute verdict/detections + threat label | IF - Hash Exists in VT | IF - Malicious Verdict | ## 4. Verdict Calculation |
| IF - Malicious Verdict | IF | Branch on `verdict === "Malicious"` | Function - Calculate Verdict | Slack - Malicious Hash Alert; Respond - Clean or Unknown | ## 5. Alerts & Responses |
| Slack - Malicious Hash Alert | Slack | Send urgent alert to Slack channel | IF - Malicious Verdict | — | ## 5. Alerts & Responses |
| Respond - Clean or Unknown | Respond to Webhook | Return verdict JSON for non-malicious results | IF - Malicious Verdict | Send a message | ## 5. Alerts & Responses |
| Send a message | Slack | Post status (Clean/Suspicious/Unknown text) | Respond - Clean or Unknown | — | ## 5. Alerts & Responses |

> Global note present in the canvas: “## File Hash Reputation Checker …” (see Section 5).

---

## 4. Reproducing the Workflow from Scratch

1) **Create Workflow**
   - Name: **File Hash Reputation Checker**
   - (Optional) Set workflow timezone to **Asia/Manila** (matches current settings).

2) **Add Webhook Trigger**
   - Node: **Webhook**
   - Name: **Receive File Hash**
   - Method: **POST**
   - Path: **/hash-check**
   - Response: **Using “Respond to Webhook” node** (responseNode mode)

3) **Normalize input**
   - Node: **Set**
   - Name: **Normalize Hash from Webhook**
   - Add field:
     - `hash` (string) = `{{$json.body.text.toLowerCase().trim()}}`
   - Connect: `Receive File Hash` → `Normalize Hash from Webhook`

4) **Validate hash format**
   - Node: **IF**
   - Name: **IF - Valid Hash Format**
   - Condition group: **OR**
   - Conditions (String equals):
     - `{{$json.hash.length.toString()}}` equals `32`
     - `{{$json.hash.length.toString()}}` equals `40`
     - `{{$json.hash.length.toString()}}` equals `64`
   - Connect: `Normalize Hash from Webhook` → `IF - Valid Hash Format`

5) **Invalid hash response**
   - Node: **Respond to Webhook**
   - Name: **Respond - Invalid Hash**
   - Respond with: **JSON**
   - Body:
     - `{"status":"error","message":"Invalid hash format. Provide MD5, SHA1, or SHA256."}`
   - Connect: `IF - Valid Hash Format` (false) → `Respond - Invalid Hash`

6) **VirusTotal credential**
   - Create credential: **VirusTotal API**
   - Store API key (VirusTotal v3). Ensure it injects the required header (commonly `x-apikey`).

7) **VirusTotal lookup**
   - Node: **HTTP Request**
   - Name: **HTTP - VirusTotal Hash Lookup**
   - Method: **GET**
   - URL: `https://www.virustotal.com/api/v3/files/{{$json.hash}}`
   - Authentication: **Predefined credential type → VirusTotal API**
   - Error behavior: set **On Error → Continue (regular output)** (so errors appear in output as `error`)
   - Connect: `IF - Valid Hash Format` (true) → `HTTP - VirusTotal Hash Lookup`

8) **Branch on VT presence**
   - Node: **IF**
   - Name: **IF - Hash Exists in VT**
   - Condition:
     - `{{$json.error === undefined}}` is **true**
   - Connect: `HTTP - VirusTotal Hash Lookup` → `IF - Hash Exists in VT`

9) **Unknown hash response**
   - Node: **Respond to Webhook**
   - Name: **Respond - Unknown Hash**
   - Respond with: **JSON**
   - Body (expression):
     - `{"verdict":"Unknown","message":"{{$json.error.message}}"}`
   - Connect: `IF - Hash Exists in VT` (false) → `Respond - Unknown Hash`

10) **Verdict calculation**
   - Node: **Code**
   - Name: **Function - Calculate Verdict**
   - Paste logic (equivalent):
     - Read `data.attributes.last_analysis_stats`
     - Compute `malicious`, `suspicious`, `total`
     - Set `verdict` to Malicious if malicious>0, else Suspicious if suspicious>0, else Clean
     - Output `hash`, `verdict`, `malicious`, `suspicious`, `total`, `malware_family`
   - Connect: `IF - Hash Exists in VT` (true) → `Function - Calculate Verdict`

11) **Branch on malicious**
   - Node: **IF**
   - Name: **IF - Malicious Verdict**
   - Condition:
     - `{{$json.verdict === "Malicious"}}` is **true**
   - Connect: `Function - Calculate Verdict` → `IF - Malicious Verdict`

12) **Slack credential**
   - Create credential: **Slack API**
   - Ensure scopes allow posting to the target channel (e.g., `chat:write`) and the app is in the channel.

13) **Malicious Slack alert**
   - Node: **Slack**
   - Name: **Slack - Malicious Hash Alert**
   - Operation: **Post message** (to a channel)
   - Channel: select your channel (example used: `C0A252GLT70`)
   - Text template:
     - `🚨 MALICIOUS FILE DETECTED ...` including hash and detections and family
   - Connect: `IF - Malicious Verdict` (true) → `Slack - Malicious Hash Alert`

14) **Webhook response for non-malicious**
   - Node: **Respond to Webhook**
   - Name: **Respond - Clean or Unknown**
   - Respond with: **JSON**
   - Body (expression) returning fields:
     - `hash`, `verdict`, `malicious`, `suspicious`, `detections: "malicious/total"`
   - Connect: `IF - Malicious Verdict` (false) → `Respond - Clean or Unknown`

15) **Slack informational message**
   - Node: **Slack**
   - Name: **Send a message**
   - Channel: same channel as above
   - Text: conditional expression producing different messages for Clean/Suspicious/Unknown
   - Connect: `Respond - Clean or Unknown` → `Send a message`

16) **Important fix to consider (recommended)**
   - Add another **Respond to Webhook** node after `Slack - Malicious Hash Alert` so malicious requests also receive an HTTP response (otherwise callers may time out in `responseNode` mode).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| **## File Hash Reputation Checker** This is a security automation workflow that validates files hashes (**MD5**, **SHA1**, **SHA256**) and checks their reputation using the **VirusTotal API**. The workflow accepts a file hash via an **HTTP webhook**, normalizes and validates the input, then queries VirusTotal to retrieve detection statistics. Based on the analysis results, it determines whether the file is **Malicious**, **Suspicious**, **Clean**, or **Unknown**. **How it works** 1) Receives a file hash via webhook **(POST)** or Slack using Slash command. 2) Normalizes and validates hash format. 3) Queries **VirusTotal** for hash reputation. 4) Calculates verdict from detection stats. 5) Sends Slack alert if **malicious**. 6) Returns a **JSON** response with verdict and detections. **Setup steps** 1) Add your **VirusTotal API** key to n8n credentials. 2) Configure **Slack API** credentials and channel. 3) Activate the workflow. 4) Send a **POST** request with `{ "text": "<file_hash>" }` or submit file hash directly from Slack using `/hash-check <file hash>`. **Customization** Adjust verdict thresholds; add other notifications; extend to file uploads or multiple hashes. | Canvas sticky note (global description). Note: implementation reads `body.text` (not top-level `text`) and there is no Slack slash-command trigger node in this workflow. |

If you want, I can propose a minimal change-set to (a) reliably respond to the webhook for malicious results, and (b) properly support Slack slash commands as an additional entry point.