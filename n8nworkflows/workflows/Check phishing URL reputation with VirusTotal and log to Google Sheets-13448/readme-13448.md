Check phishing URL reputation with VirusTotal and log to Google Sheets

https://n8nworkflows.xyz/workflows/check-phishing-url-reputation-with-virustotal-and-log-to-google-sheets-13448


# Check phishing URL reputation with VirusTotal and log to Google Sheets

## 1. Workflow Overview

**Workflow name:** *Phishing URL Reputation Checker*  
**Stated title/purpose:** *Check phishing URL reputation with VirusTotal and log to Google Sheets*  

This workflow exposes an HTTP webhook that accepts a URL, normalizes and validates it, submits it to VirusTotal for analysis, polls VirusTotal until the analysis is complete (or a retry limit is reached), derives a **verdict** (SAFE / SUSPICIOUS / PHISHING) with a **risk level**, defangs suspicious/malicious URLs, and appends the results to **Google Sheets**. It also returns structured JSON errors when the URL is invalid or VirusTotal fails.

### 1.1 Input Reception & Normalization
Receives a URL via webhook, trims it, ensures it includes a scheme, and performs lightweight validation.

### 1.2 Validation & Early Error Response
Branches execution: invalid URLs receive an immediate error response; valid URLs proceed to VirusTotal.

### 1.3 Threat Intelligence Submission (VirusTotal)
Submits the normalized URL to VirusTotal for scanning; handles submission errors.

### 1.4 Asynchronous Polling, Retries & Timeout Control
Waits, polls analysis status, retries up to a max counter, and returns a timeout response if not completed.

### 1.5 Signal Extraction & Decision Engine
Extracts VirusTotal stats, applies thresholds to classify and (if needed) defangs the URL.

### 1.6 Logging
Appends the verdict and associated metrics to Google Sheets.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception & Normalization
**Overview:** Accepts the inbound POST request and transforms the provided `body.url` into a normalized, consistently formatted URL plus a boolean validity flag.  
**Nodes involved:**  
- Webhook - Submit URL for Analysis  
- Normalize Input URL  

#### Node: Webhook - Submit URL for Analysis
- **Type / role:** `Webhook` — public entry point receiving URL submissions.
- **Key configuration:**
  - **HTTP Method:** POST
  - **Path:** `phishing-check`
  - **Response Mode:** “Respond via Respond to Webhook node” (response handled later)
- **Input / output:**
  - **Output:** to `Normalize Input URL`
- **Edge cases / failures:**
  - Missing `body.url` leads to normalization node producing `is_valid:false`.
  - If clients send non-JSON payloads, `body.url` may be undefined depending on n8n webhook parsing settings.

#### Node: Normalize Input URL
- **Type / role:** `Code` — normalizes and validates user input without relying on the URL class.
- **Configuration choices (interpreted):**
  - Reads `rawUrl` from `{{$input.first().json.body.url}}`
  - Trims whitespace; if empty => returns:
    - `original_url`, `normalized_url:""`, `is_valid:false`
  - If scheme is missing, prefixes `http://`
  - “Lightweight” validation:
    - protocol must be `http` or `https`
    - host must exist and contain a dot `.`
- **Key variables / outputs:**
  - `original_url`
  - `normalized_url`
  - `is_valid`
- **Input / output:**
  - **Input:** webhook JSON body
  - **Output:** to `IF - URL is Valid?`
- **Edge cases / failures:**
  - URLs like `localhost` or internal hostnames without a dot will be marked invalid.
  - URLs with unusual but valid host forms (IDN/punycode edge cases) may pass/fail unexpectedly.
  - Prefixing `http://` may change semantics if the intended URL was `https://…`.

---

### Block 2 — Validation & Early Error Response
**Overview:** Splits the flow based on `is_valid`. Invalid inputs return a JSON error immediately.  
**Nodes involved:**  
- IF - URL is Valid?  
- Respond - Invalid URL error  

#### Node: IF - URL is Valid?
- **Type / role:** `IF` — boolean gate for validation.
- **Condition:**
  - Checks `={{ $json.is_valid }}` is **true**
- **Input / output:**
  - **True path:** `VirusTotal - Submit URL for Scan`
  - **False path:** `Respond - Invalid URL error`
- **Edge cases / failures:**
  - If `is_valid` is missing or not boolean, strict validation could route unexpectedly (depends on actual value coercion; node is set to strict validation).

#### Node: Respond - Invalid URL error
- **Type / role:** `Respond to Webhook` — returns error payload for invalid/malformed URL.
- **Response body (static JSON):**
  - `error`: “Invalid or malformed URL”
  - `message`: “Please submit a valid URL”
- **Input / output:**
  - **Input:** from invalid branch of `IF - URL is Valid?`
  - Terminates the request.
- **Edge cases / failures:**
  - None typical; only fails if webhook context is missing (e.g., node executed without webhook trigger).

---

### Block 3 — Threat Intelligence Submission (VirusTotal) + Submission Error Handling
**Overview:** Sends the normalized URL to VirusTotal’s `/urls` endpoint (form-urlencoded), then routes either to error response or to waiting/polling.  
**Nodes involved:**  
- VirusTotal - Submit URL for Scan  
- IF - VT Submit Error?  
- Respond - VT Service Error  
- Wait - VirusTotal Scan Processing  

#### Node: VirusTotal - Submit URL for Scan
- **Type / role:** `HTTP Request` — VirusTotal URL submission.
- **Key configuration:**
  - **Method:** POST
  - **URL:** `https://www.virustotal.com/api/v3/urls`
  - **Body:** form-urlencoded with parameter:
    - `url = {{ $json.normalized_url }}`
  - **Headers:** sets `Content-Type: application/x-www-form-urlencoded`
  - **Authentication:** Generic credential type → `httpHeaderAuth` (“Header Auth account”)
  - **Error handling:** `onError: continueErrorOutput` (so failures become items with an `error` field)
- **Input / output:**
  - **Input:** validated, normalized URL from `IF - URL is Valid?` true branch
  - **Output:** to `IF - VT Submit Error?`
- **Edge cases / failures:**
  - VirusTotal API key invalid/expired → 401/403; will be routed to error handling.
  - Rate limiting (429) and quota exhaustion.
  - If VirusTotal changes expected parameters, submission fails.
  - If normalized_url is empty (should not happen if validation is correct), VT returns error.

#### Node: IF - VT Submit Error?
- **Type / role:** `IF` — checks whether the HTTP node output contains an error.
- **Condition:**
  - `={{ $json.error }}` **exists**
- **Input / output:**
  - **True path (error exists):** `Respond - VT Service Error`
  - **False path:** `Wait - VirusTotal Scan Processing`
- **Edge cases / failures:**
  - If VT returns a successful HTTP response but with unexpected structure (no `data.id` later), polling will fail downstream (not caught here).

#### Node: Respond - VT Service Error
- **Type / role:** `Respond to Webhook` — standardized “service unavailable” response.
- **Response body (static JSON):**
  - `error`: “Threat intelligence service unavailable”
  - `message`: “VirusTotal request failed. Please try again later.”
- **Input / output:**
  - Used by both submission error and analysis error branches.
- **Edge cases / failures:**
  - If the flow reaches this node after the webhook was already responded to (should not happen in this design), n8n may log response errors.

#### Node: Wait - VirusTotal Scan Processing
- **Type / role:** `Wait` — pauses to allow VT analysis to progress before polling.
- **Key configuration:**
  - Wait **amount:** 10 (seconds in typical n8n wait behavior)
- **Input / output:**
  - **Input:** successful VT submission response
  - **Output:** `VirusTotal - Get Scan Analysis`
- **Edge cases / failures:**
  - Wait node requires n8n to be able to resume executions (important in queue/worker setups).
  - If n8n restarts, waiting executions may be affected depending on persistence mode.

---

### Block 4 — Asynchronous Polling, Retries & Timeout Control
**Overview:** Polls VirusTotal analysis status. If not completed, increments a retry counter, waits, and retries up to 5 attempts, then returns a timeout response.  
**Nodes involved:**  
- VirusTotal - Get Scan Analysis  
- IF - VT Analysis Error?  
- IF - VirusTotal Analysis Completed?  
- Increment Retry Counter  
- IF Max Retry Reached?  
- Wait - Retry VT Analysis Poll  
- Respond Timeout  
- (Respond - VT Service Error also participates via error branch)

#### Node: VirusTotal - Get Scan Analysis
- **Type / role:** `HTTP Request` — fetches the analysis object until it completes.
- **Key configuration:**
  - **URL:** `=https://www.virustotal.com/api/v3/analyses/{{ $json.data.id }}`
    - Uses the `data.id` returned by VT submission.
  - **Authentication:** `httpHeaderAuth` (same “Header Auth account”)
  - **Error handling:** `onError: continueErrorOutput`
- **Input / output:**
  - **Input:** output from wait or retry wait (must include `data.id`)
  - **Output:** to `IF - VT Analysis Error?`
- **Edge cases / failures:**
  - If the previous node output doesn’t include `data.id`, the URL becomes invalid and request fails.
  - VT may return transient states or throttling responses.
  - If VT returns a completed analysis but `meta.url_info` is missing, downstream extraction may fail.

#### Node: IF - VT Analysis Error?
- **Type / role:** `IF` — checks whether polling failed.
- **Condition:**
  - `={{ $json.error }}` exists
- **Input / output:**
  - **True path:** `Respond - VT Service Error`
  - **False path:** `IF - VirusTotal Analysis Completed?`

#### Node: IF - VirusTotal Analysis Completed?
- **Type / role:** `IF` — decides whether to proceed to scoring or retry.
- **Condition:**
  - `={{ $json.data.attributes.status }}` equals `"completed"`
- **Input / output:**
  - **True path:** `Extract VirusTotal Verdict Stats`
  - **False path:** `Increment Retry Counter`
- **Edge cases / failures:**
  - If `data.attributes.status` is missing, condition fails and it will retry until timeout.

#### Node: Increment Retry Counter
- **Type / role:** `Code` — adds/updates `retry_count`.
- **Logic:**
  - `retry_count` defaults to 0
  - increments by 1
  - returns `{ json: { ...$json, retry_count: newRetry } }`
- **Input / output:**
  - **Input:** non-completed analysis payload
  - **Output:** `IF Max Retry Reached?`
- **Important note:** This node wraps the result under a `json` key explicitly. n8n Code nodes typically expect returning items like `{json: {...}}`. Here it returns `[{ json: {...} }]`, which is correct; but it also spreads `$json` (which is the previous item JSON) into that new JSON.
- **Edge cases / failures:**
  - If upstream item is unusually large, spreading into JSON increases payload size across retries.

#### Node: IF Max Retry Reached?
- **Type / role:** `IF` — enforces retry limit.
- **Condition:**
  - `={{ $json.retry_count }}` >= 5
- **Input / output:**
  - **True path:** `Respond Timeout`
  - **False path:** `Wait - Retry VT Analysis Poll`
- **Edge cases / failures:**
  - If `retry_count` is missing (shouldn’t be after increment), it may evaluate as null/undefined and could keep retrying.

#### Node: Wait - Retry VT Analysis Poll
- **Type / role:** `Wait` — delay between polls.
- **Key configuration:**
  - Wait amount: 15
- **Input / output:**
  - **Output:** `VirusTotal - Get Scan Analysis`
- **Edge cases / failures:**
  - Same wait/persistence considerations as earlier.

#### Node: Respond Timeout
- **Type / role:** `Respond to Webhook` — returns timeout status when retries are exhausted.
- **Response body (mixed JSON + expressions):**
  - `status`: `"timeout"`
  - `reason`: `"VirusTotal analysis not ready after max retries"`
  - `url`: `={{$json.url}}`
  - `retry_count`: `={{$json.retry_count}}`
- **Edge cases / failures:**
  - `url` here depends on being present in the current item; after polling, VT analysis payload typically contains `meta.url_info.url` (not necessarily `$json.url`). Unless `$json.url` exists in the retry path, this may return empty/undefined. (Potential improvement: reference `{{$json.meta.url_info.url}}` or persist original URL through the loop.)

---

### Block 5 — Detection Signal Extraction & Phishing Decision Engine
**Overview:** Extracts VirusTotal engine verdict counts and converts them into a final classification; defangs non-safe URLs to reduce accidental clicks.  
**Nodes involved:**  
- Extract VirusTotal Verdict Stats  
- Build Phishing Verdict  

#### Node: Extract VirusTotal Verdict Stats
- **Type / role:** `Code` — transforms VT analysis response into a compact stats object.
- **Logic:**
  - Reads:
    - `const stats = $json.data.attributes.stats;`
    - `const url_info = $json.meta.url_info;`
  - Outputs:
    - `vt_malicious`, `vt_suspicious`, `vt_harmless`, `vt_undetected` (defaulting to 0)
    - `vt_status`
    - `url: url_info.url`
- **Input / output:**
  - **Input:** completed VT analysis
  - **Output:** `Build Phishing Verdict`
- **Edge cases / failures:**
  - If `meta.url_info` is absent, `url_info.url` will throw. (Would need optional chaining or guards.)
  - If `stats` is missing, accessing properties may throw; current code assumes it exists.

#### Node: Build Phishing Verdict
- **Type / role:** `Code` — threshold-based scoring and URL defanging.
- **Decision thresholds:**
  - If `malicious >= 3` → `risk="High"`, `verdict="PHISHING"`
  - Else if `malicious` is 1–2 → `risk="Medium"`, `verdict="SUSPICIOUS"`
  - Else if `suspicious >= 3` → `risk="Medium"`, `verdict="SUSPICIOUS"`
  - Else → `risk="Low"`, `verdict="SAFE"`
- **Defanging:**
  - If verdict is not SAFE:
    - `http://` → `hxxp://`
    - `https://` → `hxxps://`
    - all `.` → `[.]`
- **Output structure:**
  - `url`, `verdict`, `risk_level`
  - `engines.virustotal.malicious` and `.suspicious`
- **Input / output:**
  - **Input:** extracted stats
  - **Output:** `Log Scan Result`
- **Edge cases / failures:**
  - Defanging replaces every dot, including dots in paths/query; acceptable for safety, but may reduce readability.
  - Thresholds are simplistic; “suspicious” signals might be more nuanced per use case.

---

### Block 6 — Logging to Google Sheets
**Overview:** Appends one row per scan to a Google Sheet for monitoring and investigation.  
**Nodes involved:**  
- Log Scan Result  

#### Node: Log Scan Result
- **Type / role:** `Google Sheets` — append operation.
- **Key configuration:**
  - **Operation:** Append
  - **Document:** “Phishing URL” (Spreadsheet ID `14qydoIHflwd-3gYyj7g0QKH5Bfsny_OiLzdnvEKOwfo`)
  - **Sheet/tab:** “Phishing URL scan” (`gid=0`)
  - **Credentials:** Google Sheets OAuth2 (“Google Sheets account”)
  - **Columns mapped:**
    - `timestamp` = `{{ new Date().toISOString() }}`
    - `original_url` = `{{ $json.url }}` (note: at this point `url` may be defanged if verdict not SAFE)
    - `verdict` = `{{ $json.verdict }}`
    - `risk_level` = `{{ $json.risk_level }}`
    - `malicious` = `{{ $json.engines.virustotal.malicious }}`
    - `suspicious` = `{{ $json.engines.virustotal.suspicious }}`
- **Input / output:**
  - **Input:** from `Build Phishing Verdict`
  - No further outputs.
- **Edge cases / failures:**
  - OAuth token expiration/consent issues.
  - Sheet structure mismatch (missing column headers, renamed sheet).
  - The field called `original_url` is actually the post-processed `url` from verdict node (defanged for suspicious/phishing). If you need the true original URL, persist `original_url` from the normalization step through the workflow.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook - Submit URL for Analysis | Webhook | Entry point receiving URL submissions | — | Normalize Input URL | ## URL Input & Normalization<br>Accepts user URLs via webhook and ensures consistent formatting before security analysis. |
| Normalize Input URL | Code | Trim/normalize URL; add scheme; lightweight validation | Webhook - Submit URL for Analysis | IF - URL is Valid? | ## URL Input & Normalization<br>Accepts user URLs via webhook and ensures consistent formatting before security analysis. |
| IF - URL is Valid? | IF | Route invalid input to error response | Normalize Input URL | VirusTotal - Submit URL for Scan; Respond - Invalid URL error | ## Validation<br>Checks for malformed or missing URLs and returns an error if validation fails. |
| Respond - Invalid URL error | Respond to Webhook | Return JSON error for invalid URL | IF - URL is Valid? (false) | — | ## Validation<br>Checks for malformed or missing URLs and returns an error if validation fails. |
| VirusTotal - Submit URL for Scan | HTTP Request | Submit URL to VirusTotal `/urls` for scanning | IF - URL is Valid? (true) | IF - VT Submit Error? | ## Threat Intelligence Submission<br>Submits validated URLs to VirusTotal for multi-engine reputation scanning. |
| IF - VT Submit Error? | IF | Detect submission error (`$json.error` exists) | VirusTotal - Submit URL for Scan | Respond - VT Service Error; Wait - VirusTotal Scan Processing | ## Error Handling & Resilience<br>Handles VirusTotal API failures, validation errors, and timeout conditions to ensure reliable execution. |
| Wait - VirusTotal Scan Processing | Wait | Initial delay before polling analysis | IF - VT Submit Error? (no error) | VirusTotal - Get Scan Analysis | ## Asynchronous Scan Handling<br>Polls VirusTotal until the analysis is completed or retries reach the limit. |
| VirusTotal - Get Scan Analysis | HTTP Request | Poll VirusTotal `/analyses/{id}` | Wait - VirusTotal Scan Processing; Wait - Retry VT Analysis Poll | IF - VT Analysis Error? | ## Asynchronous Scan Handling<br>Polls VirusTotal until the analysis is completed or retries reach the limit. |
| IF - VT Analysis Error? | IF | Detect polling error (`$json.error` exists) | VirusTotal - Get Scan Analysis | Respond - VT Service Error; IF - VirusTotal Analysis Completed? | ## Error Handling & Resilience<br>Handles VirusTotal API failures, validation errors, and timeout conditions to ensure reliable execution. |
| IF - VirusTotal Analysis Completed? | IF | Check `data.attributes.status == completed` | IF - VT Analysis Error? (no error) | Extract VirusTotal Verdict Stats; Increment Retry Counter | ## Asynchronous Scan Handling<br>Polls VirusTotal until the analysis is completed or retries reach the limit. |
| Increment Retry Counter | Code | Increment `retry_count` for polling loop | IF - VirusTotal Analysis Completed? (not completed) | IF Max Retry Reached? | ## Asynchronous Scan Handling<br>Polls VirusTotal until the analysis is completed or retries reach the limit. |
| IF Max Retry Reached? | IF | Stop after 5 retries | Increment Retry Counter | Respond Timeout; Wait - Retry VT Analysis Poll | ## Asynchronous Scan Handling<br>Polls VirusTotal until the analysis is completed or retries reach the limit. |
| Wait - Retry VT Analysis Poll | Wait | Delay between polling retries | IF Max Retry Reached? (not reached) | VirusTotal - Get Scan Analysis | ## Asynchronous Scan Handling<br>Polls VirusTotal until the analysis is completed or retries reach the limit. |
| Respond Timeout | Respond to Webhook | Return timeout JSON after max retries | IF Max Retry Reached? (reached) | — | ## Error Handling & Resilience<br>Handles VirusTotal API failures, validation errors, and timeout conditions to ensure reliable execution. |
| Extract VirusTotal Verdict Stats | Code | Extract VT stats + URL info | IF - VirusTotal Analysis Completed? (completed) | Build Phishing Verdict | ## Detection Signal Extraction<br>Extracts VirusTotal detection statistics used for phishing classification. |
| Build Phishing Verdict | Code | Threshold-based verdict + defanging | Extract VirusTotal Verdict Stats | Log Scan Result | ## Phishing Decision Engine<br>Applies threshold logic to classify URLs as SAFE, SUSPICIOUS, or PHISHING. |
| Log Scan Result | Google Sheets | Append verdict row to Google Sheets | Build Phishing Verdict | — | ## Logging & Output<br>Stores scan results in Google Sheets for monitoring and incident tracking. |
| Respond - VT Service Error | Respond to Webhook | Return VT service failure JSON | IF - VT Submit Error? (error); IF - VT Analysis Error? (error) | — | ## Error Handling & Resilience<br>Handles VirusTotal API failures, validation errors, and timeout conditions to ensure reliable execution. |
| Sticky Note | Sticky Note | Comment block | — | — | ## URL Input & Normalization<br>Accepts user URLs via webhook and ensures consistent formatting before security analysis. |
| Sticky Note1 | Sticky Note | Comment block | — | — | ## Validation<br>Checks for malformed or missing URLs and returns an error if validation fails. |
| Sticky Note2 | Sticky Note | Comment block | — | — | ## Threat Intelligence Submission<br>Submits validated URLs to VirusTotal for multi-engine reputation scanning. |
| Sticky Note3 | Sticky Note | Comment block | — | — | ## Asynchronous Scan Handling<br>Polls VirusTotal until the analysis is completed or retries reach the limit. |
| Sticky Note4 | Sticky Note | Comment block | — | — | ## Detection Signal Extraction<br>Extracts VirusTotal detection statistics used for phishing classification. |
| Sticky Note5 | Sticky Note | Comment block | — | — | ## Phishing Decision Engine<br>Applies threshold logic to classify URLs as SAFE, SUSPICIOUS, or PHISHING. |
| Sticky Note6 | Sticky Note | Comment block | — | — | ## Logging & Output<br>Stores scan results in Google Sheets for monitoring and incident tracking. |
| Sticky Note7 | Sticky Note | Comment block | — | — | ## Error Handling & Resilience<br>Handles VirusTotal API failures, validation errors, and timeout conditions to ensure reliable execution. |
| Sticky Note8 | Sticky Note | Comment block | — | — | ## Phishing URL Reputation Checker<br>This workflow analyzes submitted URLs to determine whether they are phishing or malicious using VirusTotal’s threat intelligence data. It validates user input, submits the URL for scanning, polls for results, extracts detection signals, and generates a clear phishing verdict with risk scoring. Results are optionally logged to Google Sheets for tracking and investigation.<br><br>### How it works<br>1. A webhook accepts a URL from an API, form, chatbot, or automation.<br>2. The URL is normalized and validated to prevent malformed or unsafe input.<br>3. Valid URLs are submitted to VirusTotal for multi-engine reputation analysis.<br>4. The workflow polls VirusTotal asynchronously until the scan is complete or retries are exhausted.<br>5. Detection statistics are extracted and evaluated using threshold-based phishing logic.<br>6. Suspicious or malicious URLs are defanged to prevent accidental clicks.<br>7. The final verdict and risk level are returned and optionally logged to Google Sheets.<br><br>### Setup steps<br>1. Add your VirusTotal API key in the HTTP Header Auth credentials.<br>2. Connect Google Sheets to store scan results.<br>3. Trigger the webhook with { "url": "example.com" }.<br><br>### Customization<br>1. Adjust phishing thresholds in the “Build Phishing Verdict” node or add additional reputation sources for stronger detection.<br>2. You can add a Slack, Discord, or email notification when the verdict is not SAFE to alert security teams about potential phishing URLs in real time. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name: `Phishing URL Reputation Checker`
   - Keep workflow inactive until credentials are set.

2. **Add Webhook node**
   - Node type: *Webhook*
   - Name: `Webhook - Submit URL for Analysis`
   - Method: `POST`
   - Path: `phishing-check`
   - Response mode: `Using 'Respond to Webhook' node`

3. **Add Code node for normalization**
   - Node type: *Code*
   - Name: `Normalize Input URL`
   - Paste logic to:
     - read `{{$input.first().json.body.url}}`
     - trim
     - prefix `http://` if scheme missing
     - validate protocol http/https and host contains `.`
     - output `original_url`, `normalized_url`, `is_valid`

4. **Add IF node for URL validity**
   - Node type: *IF*
   - Name: `IF - URL is Valid?`
   - Condition: Boolean `={{ $json.is_valid }}` is `true`
   - Connect:
     - Webhook → Normalize Input URL → IF - URL is Valid?

5. **Add Respond to Webhook for invalid input**
   - Node type: *Respond to Webhook*
   - Name: `Respond - Invalid URL error`
   - Respond with: JSON
   - Body:
     - `error`: `Invalid or malformed URL`
     - `message`: `Please submit a valid URL`
   - Connect IF false → this node.

6. **Create VirusTotal API credential**
   - Credential type: *HTTP Header Auth*
   - Name: e.g. `Header Auth account`
   - Header name/value should match VirusTotal v3 requirements:
     - Common setup: `x-apikey: <YOUR_VT_API_KEY>`
   - (Keep it in n8n credentials store.)

7. **Add HTTP Request node to submit URL**
   - Node type: *HTTP Request*
   - Name: `VirusTotal - Submit URL for Scan`
   - Method: `POST`
   - URL: `https://www.virustotal.com/api/v3/urls`
   - Authentication: `Generic Credential Type` → `HTTP Header Auth` (select your credential)
   - Send body: enabled
   - Content type: `Form URL-Encoded`
   - Body parameter: `url = {{ $json.normalized_url }}`
   - Headers: set `Content-Type: application/x-www-form-urlencoded`
   - Error handling: set to “Continue on Fail” (so the node produces an `error` field)
   - Connect IF true → this node.

8. **Add IF node to detect submission error**
   - Node type: *IF*
   - Name: `IF - VT Submit Error?`
   - Condition: String → `exists` on `={{ $json.error }}`
   - Connect submit → IF.

9. **Add Respond to Webhook for VT failures**
   - Node type: *Respond to Webhook*
   - Name: `Respond - VT Service Error`
   - JSON body:
     - `error`: `Threat intelligence service unavailable`
     - `message`: `VirusTotal request failed. Please try again later.`
   - Connect IF (error exists) → respond error.

10. **Add Wait node (initial processing delay)**
    - Node type: *Wait*
    - Name: `Wait - VirusTotal Scan Processing`
    - Amount: `10` (seconds)
    - Connect IF (no error) → wait.

11. **Add HTTP Request node to poll analysis**
    - Node type: *HTTP Request*
    - Name: `VirusTotal - Get Scan Analysis`
    - Method: `GET`
    - URL: `https://www.virustotal.com/api/v3/analyses/{{ $json.data.id }}`
    - Auth: same HTTP Header Auth credential
    - Enable “Continue on Fail”
    - Connect initial wait → this node.

12. **Add IF node for analysis polling error**
    - Node type: *IF*
    - Name: `IF - VT Analysis Error?`
    - Condition: `={{ $json.error }}` exists
    - Connect poll → IF
    - Connect error path → `Respond - VT Service Error`

13. **Add IF node for completion status**
    - Node type: *IF*
    - Name: `IF - VirusTotal Analysis Completed?`
    - Condition: `={{ $json.data.attributes.status }}` equals `completed`
    - Connect no-error path → this IF.

14. **Add Code node to increment retry counter**
    - Node type: *Code*
    - Name: `Increment Retry Counter`
    - Implement increment logic for `retry_count` (default 0 → +1)
    - Connect “not completed” branch → increment.

15. **Add IF node for max retries**
    - Node type: *IF*
    - Name: `IF Max Retry Reached?`
    - Condition: Number `={{ $json.retry_count }}` `>= 5`
    - Connect increment → IF.

16. **Add Respond to Webhook for timeout**
    - Node type: *Respond to Webhook*
    - Name: `Respond Timeout`
    - JSON body including:
      - `status: "timeout"`
      - `reason: "VirusTotal analysis not ready after max retries"`
      - `url` (choose a reliable field; ideally persist original URL or use `meta.url_info.url` if present)
      - `retry_count: {{$json.retry_count}}`
    - Connect IF true (max reached) → timeout response.

17. **Add Wait node for retry delay**
    - Node type: *Wait*
    - Name: `Wait - Retry VT Analysis Poll`
    - Amount: `15`
    - Connect IF false (max not reached) → wait retry
    - Connect wait retry → back to `VirusTotal - Get Scan Analysis` (loop)

18. **Add Code node to extract VT stats**
    - Node type: *Code*
    - Name: `Extract VirusTotal Verdict Stats`
    - Extract:
      - `data.attributes.stats` counts
      - `meta.url_info.url`
    - Connect “completed” branch → this node.

19. **Add Code node to build verdict**
    - Node type: *Code*
    - Name: `Build Phishing Verdict`
    - Implement threshold rules and defanging for non-safe verdicts
    - Output: `url`, `verdict`, `risk_level`, `engines.virustotal.{malicious,suspicious}`
    - Connect extraction → verdict node.

20. **Add Google Sheets node to log**
    - Node type: *Google Sheets*
    - Name: `Log Scan Result`
    - Operation: `Append`
    - Set Google Sheets OAuth2 credential (create/authorize if needed)
    - Select target Spreadsheet and Sheet tab
    - Map columns:
      - `timestamp = {{ new Date().toISOString() }}`
      - `original_url = {{ $json.url }}`
      - `verdict`, `risk_level`, `malicious`, `suspicious` from verdict output
    - Connect verdict → sheets node.

21. **(Recommended) Add a final success response**
    - This workflow is configured for webhook “response node” mode but does not include a success `Respond to Webhook` node in the provided JSON.
    - To make the API caller receive a result, add:
      - Node type: *Respond to Webhook*
      - Respond with JSON including verdict, risk level, and URL
      - Connect it after `Build Phishing Verdict` (and optionally after logging)

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Disclaimer (provided by user): *Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n…* | Compliance/context note for this documentation |
| “Trigger the webhook with { "url": "example.com" }.” | From Sticky Note8 setup instructions |
| “Add your VirusTotal API key in the HTTP Header Auth credentials.” | From Sticky Note8 setup instructions |
| Consider adding Slack/Discord/email alert when verdict is not SAFE | From Sticky Note8 customization suggestion |
| Important implementation gap: no success Respond-to-Webhook node is present | Webhook node is set to respond via response node; add a success response for correct API behavior |

