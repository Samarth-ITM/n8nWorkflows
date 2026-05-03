Automatically solve reCAPTCHA v2 with CapSolver

https://n8nworkflows.xyz/workflows/automatically-solve-recaptcha-v2-with-capsolver-13996


# Automatically solve reCAPTCHA v2 with CapSolver

# 1. Workflow Overview

This workflow is designed to solve **Google reCAPTCHA v2** challenges through **CapSolver** in three different ways:

1. **On-demand via webhook**: external systems send a POST request containing reCAPTCHA parameters, and the workflow returns the solved token.
2. **On a recurring schedule**: every hour, the workflow solves a predefined demo reCAPTCHA and submits the token to Google’s demo page to verify whether the solver still works.
3. **Manually for testing**: a manual trigger runs a fixed reCAPTCHA solve and formats the result for inspection.

The workflow is therefore both:
- a **solver API endpoint**, and
- a **health-monitoring workflow** for CapSolver / reCAPTCHA solving.

## 1.1 Input Reception and Request-Based Solving
This block receives HTTP requests through a webhook, extracts task parameters from the request body, sends them to CapSolver, and returns the solver output as JSON.

## 1.2 Scheduled Monitoring and Token Validation
This block runs every hour, builds a fixed reCAPTCHA target configuration, requests a token from CapSolver, submits that token to Google’s demo endpoint, and checks whether the returned HTML indicates success.

## 1.3 Manual Test Execution
This block allows a user to manually run a test solve against Google’s demo reCAPTCHA and formats the most useful response fields.

## 1.4 Documentation / In-Canvas Guidance
The workflow includes sticky notes that explain the purpose of each area, setup tasks, and expected operating flow.

---

# 2. Block-by-Block Analysis

## Block 1 — Input Reception and Request-Based Solving

### Overview
This block exposes a webhook endpoint for external callers that need a reCAPTCHA v2 token. It dynamically maps request body values into the CapSolver node and returns the resulting solved token payload to the caller.

### Nodes Involved
- Receive Solver Request
- CapSolver [Webhook]
- Return Solver Result

### Node Details

#### 1. Receive Solver Request
- **Type and technical role:** `n8n-nodes-base.webhook`  
  Entry point for HTTP POST requests.
- **Configuration choices:**
  - HTTP method: `POST`
  - Path: `solve-recaptcha-v2`
  - Response mode: `responseNode`, meaning a separate Respond to Webhook node must send the response.
  - `onError` is set to `continueRegularOutput`, so even if this node encounters certain issues, workflow behavior favors continuation on the normal output path.
- **Key expressions or variables used:**  
  No internal expression here, but downstream nodes read request content from:
  - `$json.body.websiteKey`
  - `$json.body.websiteURL`
  - `$json.body.taskType`
  - `$json.body.proxy`
  - `$json.body.apiDomain`
  - `$json.body.isSession`
  - `$json.body.pageAction`
  - `$json.body.isInvisible`
  - `$json.body.enterprisePayload`
- **Input and output connections:**
  - No input node
  - Output to `CapSolver [Webhook]`
- **Version-specific requirements:** `typeVersion: 2.1`
- **Edge cases or potential failure types:**
  - Invalid webhook payload shape
  - Missing `websiteKey` or `websiteURL`
  - Wrong HTTP method
  - If webhook URL is not active in production/test mode, requests may fail
  - If caller sends non-JSON content or unexpected nested fields, expressions may resolve to empty values
- **Sub-workflow reference:** None

#### 2. CapSolver [Webhook]
- **Type and technical role:** `n8n-nodes-capsolver.capSolver`  
  Sends a reCAPTCHA solving request to CapSolver using values provided by the webhook body.
- **Configuration choices:**
  - `type`: dynamic expression, defaults to `ReCaptchaV2TaskProxyLess` if not provided
  - `proxy`: optional dynamic expression, defaults to empty string
  - `websiteKey`: required from request body
  - `websiteURL`: required from request body
  - Optional fields are also mapped dynamically:
    - `apiDomain`
    - `isSession`
    - `pageAction`
    - `isInvisible`
    - `enterprisePayload`
- **Key expressions or variables used:**
  - `={{ $json.body.taskType || 'ReCaptchaV2TaskProxyLess' }}`
  - `={{ $json.body.proxy || '' }}`
  - `={{ $json.body.apiDomain || '' }}`
  - `={{ $json.body.isSession || false }}`
  - `={{ $json.body.pageAction || '' }}`
  - `={{ $json.body.isInvisible || false }}`
  - `={{ $json.body.enterprisePayload || '' }}`
  - `={{ $json.body.websiteKey }}`
  - `={{ $json.body.websiteURL }}`
- **Input and output connections:**
  - Input from `Receive Solver Request`
  - Output to `Return Solver Result`
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:**
  - Missing or invalid CapSolver credentials
  - Unsupported `taskType`
  - Invalid proxy format when proxy-based solving is requested
  - CapSolver API errors such as low balance, bad parameters, timeout, rate limiting
  - Expression resolution failures if `body` is absent
  - `enterprisePayload` may need a specific structure depending on the target site
- **Sub-workflow reference:** None

#### 3. Return Solver Result
- **Type and technical role:** `n8n-nodes-base.respondToWebhook`  
  Sends the final HTTP response back to the original webhook caller.
- **Configuration choices:**
  - Respond with: `json`
  - Response body: serialized `data` object from the CapSolver output
- **Key expressions or variables used:**
  - `={{ JSON.stringify($json.data) }}`
- **Input and output connections:**
  - Input from `CapSolver [Webhook]`
  - No output node
- **Version-specific requirements:** `typeVersion: 1.5`
- **Edge cases or potential failure types:**
  - If CapSolver output does not contain `data`, the response may be empty or malformed
  - Since `respondWith` is `json`, but the body is already stringified, callers may receive a JSON string rather than a native object depending on n8n runtime behavior
  - If the webhook path is called but this node is not reached, the request may hang until timeout
- **Sub-workflow reference:** None

---

## Block 2 — Scheduled Monitoring and Token Validation

### Overview
This block acts as a recurring monitoring process. Every hour, it solves the Google reCAPTCHA v2 demo challenge, submits the returned token back to the demo form, then checks whether the response indicates a successful validation.

### Nodes Involved
- Schedule Trigger (Every 1h)
- Set Target Params
- CapSolver [Schedule]
- Submit Token
- Check Result
- Monitor Passed
- Monitor Failed

### Node Details

#### 1. Schedule Trigger (Every 1h)
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`  
  Starts the monitoring flow automatically every hour.
- **Configuration choices:**
  - Interval rule set to every `1` hour
- **Key expressions or variables used:** None
- **Input and output connections:**
  - No input node
  - Output to `Set Target Params`
- **Version-specific requirements:** `typeVersion: 1.3`
- **Edge cases or potential failure types:**
  - Workflow must be active for schedule execution
  - Server timezone may affect expected execution timing
  - n8n instance downtime will skip scheduled runs
- **Sub-workflow reference:** None

#### 2. Set Target Params
- **Type and technical role:** `n8n-nodes-base.set`  
  Creates a static payload containing the target page URL and site key for the Google demo reCAPTCHA.
- **Configuration choices:**
  - Sets:
    - `websiteURL = https://www.google.com/recaptcha/api2/demo`
    - `websiteKey = 6Le-wvkSAAAAAPBMRTvw0Q4Muexq9bi0DJwx_mJ-`
- **Key expressions or variables used:** None; values are static
- **Input and output connections:**
  - Input from `Schedule Trigger (Every 1h)`
  - Output to `CapSolver [Schedule]`
- **Version-specific requirements:** `typeVersion: 3.4`
- **Edge cases or potential failure types:**
  - Site key may change if Google updates the demo page
  - Hardcoded values make this monitor specific to one page
- **Sub-workflow reference:** None

#### 3. CapSolver [Schedule]
- **Type and technical role:** `n8n-nodes-capsolver.capSolver`  
  Requests a reCAPTCHA v2 solution for the predefined demo page.
- **Configuration choices:**
  - Uses `websiteKey` and `websiteURL` from the previous Set node
  - Optional settings are left empty
  - Explicit credentials are configured:
    - `CapSolver account`
- **Key expressions or variables used:**
  - `={{ $json.websiteKey }}`
  - `={{ $json.websiteURL }}`
- **Input and output connections:**
  - Input from `Set Target Params`
  - Output to `Submit Token`
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:**
  - Invalid or expired CapSolver credentials
  - CapSolver balance exhaustion
  - Solver timeout or poor token quality
  - Page/site key mismatch if the target page changes
- **Sub-workflow reference:** None

#### 4. Submit Token
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Posts the solved token back to the Google demo page as a form submission.
- **Configuration choices:**
  - Method: `POST`
  - URL: `https://www.google.com/recaptcha/api2/demo`
  - Content type: `form-urlencoded`
  - Sends body field:
    - `g-recaptcha-response = {{$json.data.solution.gRecaptchaResponse}}`
  - Sends headers:
    - `content-type: application/x-www-form-urlencoded`
    - browser-like `user-agent`
- **Key expressions or variables used:**
  - `={{ $json.data.solution.gRecaptchaResponse }}`
- **Input and output connections:**
  - Input from `CapSolver [Schedule]`
  - Output to `Check Result`
- **Version-specific requirements:** `typeVersion: 4.3`
- **Edge cases or potential failure types:**
  - Missing `gRecaptchaResponse` in solver output
  - HTTP request blocked or rate limited
  - Target page may change expected form fields
  - The demo page may require additional hidden fields in future
  - If Google returns unexpected HTML, the next IF check may produce false negatives
- **Sub-workflow reference:** None

#### 5. Check Result
- **Type and technical role:** `n8n-nodes-base.if`  
  Evaluates whether the HTTP response content appears to indicate success.
- **Configuration choices:**
  - String condition using `contains`
  - Left value: `{{$json.data}}`
  - Right value: `recaptcha-success`
  - Strict type validation enabled
- **Key expressions or variables used:**
  - `={{ $json.data }}`
- **Input and output connections:**
  - Input from `Submit Token`
  - True output to `Monitor Passed`
  - False output to `Monitor Failed`
- **Version-specific requirements:** `typeVersion: 2.3`
- **Edge cases or potential failure types:**
  - Assumes the HTTP Request node stores response body in `data`
  - If response format changes, the string check may fail
  - HTML minification or wording changes may break detection
  - If HTTP Request returns binary or structured JSON instead of text, type mismatch can occur
- **Sub-workflow reference:** None

#### 6. Monitor Passed
- **Type and technical role:** `n8n-nodes-base.set`  
  Produces a simple success status object when token validation appears successful.
- **Configuration choices:**
  - Sets:
    - `status = passed`
    - `message = reCAPTCHA v2 monitor passed successfully`
    - `timestamp = current ISO datetime`
- **Key expressions or variables used:**
  - `={{ new Date().toISOString() }}`
- **Input and output connections:**
  - Input from true branch of `Check Result`
  - No output node
- **Version-specific requirements:** `typeVersion: 3.4`
- **Edge cases or potential failure types:**
  - Minimal risk; only expression time formatting could theoretically fail
- **Sub-workflow reference:** None

#### 7. Monitor Failed
- **Type and technical role:** `n8n-nodes-base.set`  
  Produces a failure status object when token validation does not appear successful.
- **Configuration choices:**
  - Sets:
    - `status = failed`
    - `message = reCAPTCHA v2 monitor FAILED — token rejected`
    - `timestamp = current ISO datetime`
- **Key expressions or variables used:**
  - `={{ new Date().toISOString() }}`
- **Input and output connections:**
  - Input from false branch of `Check Result`
  - No output node
- **Version-specific requirements:** `typeVersion: 3.4`
- **Edge cases or potential failure types:**
  - Minimal risk; mostly dependent on correctness of prior IF result
- **Sub-workflow reference:** None

---

## Block 3 — Manual Test Execution

### Overview
This block exists for ad hoc validation. A user runs the workflow manually, CapSolver solves the fixed demo challenge, and the result is reduced to a concise object containing the token, task ID, and timestamps.

### Nodes Involved
- Manual Trigger (Test)
- CapSolver [Manual]
- Format Result

### Node Details

#### 1. Manual Trigger (Test)
- **Type and technical role:** `n8n-nodes-base.manualTrigger`  
  Starts the test flow from the n8n editor.
- **Configuration choices:** No custom parameters
- **Key expressions or variables used:** None
- **Input and output connections:**
  - No input node
  - Output to `CapSolver [Manual]`
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:**
  - Only available in manual/editor executions
  - Does not run in production automatically
- **Sub-workflow reference:** None

#### 2. CapSolver [Manual]
- **Type and technical role:** `n8n-nodes-capsolver.capSolver`  
  Solves the fixed Google demo reCAPTCHA using explicitly configured credentials.
- **Configuration choices:**
  - `websiteKey`: hardcoded Google demo key
  - `websiteURL`: hardcoded Google demo page
  - Optional parameters left empty
  - Uses `CapSolver account` credentials
- **Key expressions or variables used:** None; values are static
- **Input and output connections:**
  - Input from `Manual Trigger (Test)`
  - Output to `Format Result`
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:**
  - Same CapSolver API issues as in other solver nodes
  - Demo page/site key could become outdated
- **Sub-workflow reference:** None

#### 3. Format Result
- **Type and technical role:** `n8n-nodes-base.set`  
  Creates a simplified result payload for easier inspection during manual tests.
- **Configuration choices:**
  - Sets:
    - `token = {{$json.data.solution.gRecaptchaResponse}}`
    - `taskId = {{$json.data.taskId}}`
    - `status = solved`
    - `solvedAt = current ISO datetime`
- **Key expressions or variables used:**
  - `={{ $json.data.solution.gRecaptchaResponse }}`
  - `={{ $json.data.taskId }}`
  - `={{ new Date().toISOString() }}`
- **Input and output connections:**
  - Input from `CapSolver [Manual]`
  - No output node
- **Version-specific requirements:** `typeVersion: 3.4`
- **Edge cases or potential failure types:**
  - Missing `data.solution.gRecaptchaResponse`
  - Missing `taskId`
  - If CapSolver response structure changes, expressions break
- **Sub-workflow reference:** None

---

## Block 4 — Documentation / In-Canvas Guidance

### Overview
These sticky notes are not executable nodes but provide critical operational context. They document setup steps, explain the major execution paths, and describe the validation branch.

### Nodes Involved
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3

### Node Details

#### 1. Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Canvas documentation for overall workflow behavior and setup checklist.
- **Configuration choices:**
  - Large note covering all trigger paths
  - Includes setup checklist
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### 2. Sticky Note1
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Documents the webhook request/response path.
- **Configuration choices:** Colored note labeling the request handling block
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### 3. Sticky Note2
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Documents the scheduled monitoring block.
- **Configuration choices:** Colored note labeling the schedule-based process
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### 4. Sticky Note3
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Documents the token submission and validation branch.
- **Configuration choices:** Colored note labeling the validation block
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | stickyNote | Overall workflow documentation and setup checklist |  |  | ## reCAPTCHA v2 — All Triggers<br>### How it works<br>1. Receives solver requests through a webhook.<br>2. Processes these requests using CapSolver and returns the results.<br>3. Regularly triggers checks every hour using a schedule.<br>4. Executes the scheduled process and checks the status of tasks.<br>5. Allows manual testing of CapSolver through a manual trigger.<br>### Setup steps<br>- [ ] Set up webhook credentials for 'Receive Solver Request'.<br>- [ ] Configure CapSolver with necessary keys for all CapSolver nodes.<br>- [ ] Assign necessary permissions for HTTP request to reCAPTCHA endpoint.<br>- [ ] Review and adjust the schedule in 'Schedule Trigger'.<br>- [ ] Test manual execution with 'Manual Trigger'. |
| Sticky Note1 | stickyNote | Documentation for webhook handling section |  |  | ## Receive and handle requests<br>Starts with receiving a solver request via webhook, processes it, and returns the solver result. |
| Sticky Note2 | stickyNote | Documentation for scheduled section |  |  | ## Scheduled recurrent process<br>Initiates the process on an hourly schedule, sets target parameters, and sends them to CapSolver. |
| Sticky Note3 | stickyNote | Documentation for validation section |  |  | ## Process and validate token<br>Posts to the reCAPTCHA validation endpoint and checks results, branching into pass or fail paths. |
| Receive Solver Request | webhook | Receives POST solver requests |  | CapSolver [Webhook] | ## reCAPTCHA v2 — All Triggers<br>### How it works<br>1. Receives solver requests through a webhook.<br>2. Processes these requests using CapSolver and returns the results.<br>3. Regularly triggers checks every hour using a schedule.<br>4. Executes the scheduled process and checks the status of tasks.<br>5. Allows manual testing of CapSolver through a manual trigger.<br>### Setup steps<br>- [ ] Set up webhook credentials for 'Receive Solver Request'.<br>- [ ] Configure CapSolver with necessary keys for all CapSolver nodes.<br>- [ ] Assign necessary permissions for HTTP request to reCAPTCHA endpoint.<br>- [ ] Review and adjust the schedule in 'Schedule Trigger'.<br>- [ ] Test manual execution with 'Manual Trigger'.<br>## Receive and handle requests<br>Starts with receiving a solver request via webhook, processes it, and returns the solver result. |
| CapSolver [Webhook] | n8n-nodes-capsolver.capSolver | Solves reCAPTCHA dynamically from webhook payload | Receive Solver Request | Return Solver Result | ## reCAPTCHA v2 — All Triggers<br>### How it works<br>1. Receives solver requests through a webhook.<br>2. Processes these requests using CapSolver and returns the results.<br>3. Regularly triggers checks every hour using a schedule.<br>4. Executes the scheduled process and checks the status of tasks.<br>5. Allows manual testing of CapSolver through a manual trigger.<br>### Setup steps<br>- [ ] Set up webhook credentials for 'Receive Solver Request'.<br>- [ ] Configure CapSolver with necessary keys for all CapSolver nodes.<br>- [ ] Assign necessary permissions for HTTP request to reCAPTCHA endpoint.<br>- [ ] Review and adjust the schedule in 'Schedule Trigger'.<br>- [ ] Test manual execution with 'Manual Trigger'.<br>## Receive and handle requests<br>Starts with receiving a solver request via webhook, processes it, and returns the solver result. |
| Return Solver Result | respondToWebhook | Returns CapSolver response to webhook caller | CapSolver [Webhook] |  | ## reCAPTCHA v2 — All Triggers<br>### How it works<br>1. Receives solver requests through a webhook.<br>2. Processes these requests using CapSolver and returns the results.<br>3. Regularly triggers checks every hour using a schedule.<br>4. Executes the scheduled process and checks the status of tasks.<br>5. Allows manual testing of CapSolver through a manual trigger.<br>### Setup steps<br>- [ ] Set up webhook credentials for 'Receive Solver Request'.<br>- [ ] Configure CapSolver with necessary keys for all CapSolver nodes.<br>- [ ] Assign necessary permissions for HTTP request to reCAPTCHA endpoint.<br>- [ ] Review and adjust the schedule in 'Schedule Trigger'.<br>- [ ] Test manual execution with 'Manual Trigger'.<br>## Receive and handle requests<br>Starts with receiving a solver request via webhook, processes it, and returns the solver result. |
| Schedule Trigger (Every 1h) | scheduleTrigger | Launches hourly monitoring flow |  | Set Target Params | ## reCAPTCHA v2 — All Triggers<br>### How it works<br>1. Receives solver requests through a webhook.<br>2. Processes these requests using CapSolver and returns the results.<br>3. Regularly triggers checks every hour using a schedule.<br>4. Executes the scheduled process and checks the status of tasks.<br>5. Allows manual testing of CapSolver through a manual trigger.<br>### Setup steps<br>- [ ] Set up webhook credentials for 'Receive Solver Request'.<br>- [ ] Configure CapSolver with necessary keys for all CapSolver nodes.<br>- [ ] Assign necessary permissions for HTTP request to reCAPTCHA endpoint.<br>- [ ] Review and adjust the schedule in 'Schedule Trigger'.<br>- [ ] Test manual execution with 'Manual Trigger'.<br>## Scheduled recurrent process<br>Initiates the process on an hourly schedule, sets target parameters, and sends them to CapSolver. |
| Set Target Params | set | Defines fixed website URL and site key for scheduled monitor | Schedule Trigger (Every 1h) | CapSolver [Schedule] | ## reCAPTCHA v2 — All Triggers<br>### How it works<br>1. Receives solver requests through a webhook.<br>2. Processes these requests using CapSolver and returns the results.<br>3. Regularly triggers checks every hour using a schedule.<br>4. Executes the scheduled process and checks the status of tasks.<br>5. Allows manual testing of CapSolver through a manual trigger.<br>### Setup steps<br>- [ ] Set up webhook credentials for 'Receive Solver Request'.<br>- [ ] Configure CapSolver with necessary keys for all CapSolver nodes.<br>- [ ] Assign necessary permissions for HTTP request to reCAPTCHA endpoint.<br>- [ ] Review and adjust the schedule in 'Schedule Trigger'.<br>- [ ] Test manual execution with 'Manual Trigger'.<br>## Scheduled recurrent process<br>Initiates the process on an hourly schedule, sets target parameters, and sends them to CapSolver. |
| CapSolver [Schedule] | n8n-nodes-capsolver.capSolver | Solves fixed scheduled reCAPTCHA target | Set Target Params | Submit Token | ## reCAPTCHA v2 — All Triggers<br>### How it works<br>1. Receives solver requests through a webhook.<br>2. Processes these requests using CapSolver and returns the results.<br>3. Regularly triggers checks every hour using a schedule.<br>4. Executes the scheduled process and checks the status of tasks.<br>5. Allows manual testing of CapSolver through a manual trigger.<br>### Setup steps<br>- [ ] Set up webhook credentials for 'Receive Solver Request'.<br>- [ ] Configure CapSolver with necessary keys for all CapSolver nodes.<br>- [ ] Assign necessary permissions for HTTP request to reCAPTCHA endpoint.<br>- [ ] Review and adjust the schedule in 'Schedule Trigger'.<br>- [ ] Test manual execution with 'Manual Trigger'.<br>## Scheduled recurrent process<br>Initiates the process on an hourly schedule, sets target parameters, and sends them to CapSolver. |
| Submit Token | httpRequest | Posts solved token to Google demo page | CapSolver [Schedule] | Check Result | ## reCAPTCHA v2 — All Triggers<br>### How it works<br>1. Receives solver requests through a webhook.<br>2. Processes these requests using CapSolver and returns the results.<br>3. Regularly triggers checks every hour using a schedule.<br>4. Executes the scheduled process and checks the status of tasks.<br>5. Allows manual testing of CapSolver through a manual trigger.<br>### Setup steps<br>- [ ] Set up webhook credentials for 'Receive Solver Request'.<br>- [ ] Configure CapSolver with necessary keys for all CapSolver nodes.<br>- [ ] Assign necessary permissions for HTTP request to reCAPTCHA endpoint.<br>- [ ] Review and adjust the schedule in 'Schedule Trigger'.<br>- [ ] Test manual execution with 'Manual Trigger'.<br>## Process and validate token<br>Posts to the reCAPTCHA validation endpoint and checks results, branching into pass or fail paths. |
| Check Result | if | Detects whether validation succeeded | Submit Token | Monitor Passed; Monitor Failed | ## reCAPTCHA v2 — All Triggers<br>### How it works<br>1. Receives solver requests through a webhook.<br>2. Processes these requests using CapSolver and returns the results.<br>3. Regularly triggers checks every hour using a schedule.<br>4. Executes the scheduled process and checks the status of tasks.<br>5. Allows manual testing of CapSolver through a manual trigger.<br>### Setup steps<br>- [ ] Set up webhook credentials for 'Receive Solver Request'.<br>- [ ] Configure CapSolver with necessary keys for all CapSolver nodes.<br>- [ ] Assign necessary permissions for HTTP request to reCAPTCHA endpoint.<br>- [ ] Review and adjust the schedule in 'Schedule Trigger'.<br>- [ ] Test manual execution with 'Manual Trigger'.<br>## Process and validate token<br>Posts to the reCAPTCHA validation endpoint and checks results, branching into pass or fail paths. |
| Monitor Passed | set | Emits success monitoring payload | Check Result |  | ## reCAPTCHA v2 — All Triggers<br>### How it works<br>1. Receives solver requests through a webhook.<br>2. Processes these requests using CapSolver and returns the results.<br>3. Regularly triggers checks every hour using a schedule.<br>4. Executes the scheduled process and checks the status of tasks.<br>5. Allows manual testing of CapSolver through a manual trigger.<br>### Setup steps<br>- [ ] Set up webhook credentials for 'Receive Solver Request'.<br>- [ ] Configure CapSolver with necessary keys for all CapSolver nodes.<br>- [ ] Assign necessary permissions for HTTP request to reCAPTCHA endpoint.<br>- [ ] Review and adjust the schedule in 'Schedule Trigger'.<br>- [ ] Test manual execution with 'Manual Trigger'.<br>## Process and validate token<br>Posts to the reCAPTCHA validation endpoint and checks results, branching into pass or fail paths. |
| Monitor Failed | set | Emits failure monitoring payload | Check Result |  | ## reCAPTCHA v2 — All Triggers<br>### How it works<br>1. Receives solver requests through a webhook.<br>2. Processes these requests using CapSolver and returns the results.<br>3. Regularly triggers checks every hour using a schedule.<br>4. Executes the scheduled process and checks the status of tasks.<br>5. Allows manual testing of CapSolver through a manual trigger.<br>### Setup steps<br>- [ ] Set up webhook credentials for 'Receive Solver Request'.<br>- [ ] Configure CapSolver with necessary keys for all CapSolver nodes.<br>- [ ] Assign necessary permissions for HTTP request to reCAPTCHA endpoint.<br>- [ ] Review and adjust the schedule in 'Schedule Trigger'.<br>- [ ] Test manual execution with 'Manual Trigger'.<br>## Process and validate token<br>Posts to the reCAPTCHA validation endpoint and checks results, branching into pass or fail paths. |
| Manual Trigger (Test) | manualTrigger | Starts manual test branch |  | CapSolver [Manual] | ## reCAPTCHA v2 — All Triggers<br>### How it works<br>1. Receives solver requests through a webhook.<br>2. Processes these requests using CapSolver and returns the results.<br>3. Regularly triggers checks every hour using a schedule.<br>4. Executes the scheduled process and checks the status of tasks.<br>5. Allows manual testing of CapSolver through a manual trigger.<br>### Setup steps<br>- [ ] Set up webhook credentials for 'Receive Solver Request'.<br>- [ ] Configure CapSolver with necessary keys for all CapSolver nodes.<br>- [ ] Assign necessary permissions for HTTP request to reCAPTCHA endpoint.<br>- [ ] Review and adjust the schedule in 'Schedule Trigger'.<br>- [ ] Test manual execution with 'Manual Trigger'. |
| CapSolver [Manual] | n8n-nodes-capsolver.capSolver | Solves fixed demo reCAPTCHA for manual testing | Manual Trigger (Test) | Format Result | ## reCAPTCHA v2 — All Triggers<br>### How it works<br>1. Receives solver requests through a webhook.<br>2. Processes these requests using CapSolver and returns the results.<br>3. Regularly triggers checks every hour using a schedule.<br>4. Executes the scheduled process and checks the status of tasks.<br>5. Allows manual testing of CapSolver through a manual trigger.<br>### Setup steps<br>- [ ] Set up webhook credentials for 'Receive Solver Request'.<br>- [ ] Configure CapSolver with necessary keys for all CapSolver nodes.<br>- [ ] Assign necessary permissions for HTTP request to reCAPTCHA endpoint.<br>- [ ] Review and adjust the schedule in 'Schedule Trigger'.<br>- [ ] Test manual execution with 'Manual Trigger'. |
| Format Result | set | Reduces manual solver output to key fields | CapSolver [Manual] |  | ## reCAPTCHA v2 — All Triggers<br>### How it works<br>1. Receives solver requests through a webhook.<br>2. Processes these requests using CapSolver and returns the results.<br>3. Regularly triggers checks every hour using a schedule.<br>4. Executes the scheduled process and checks the status of tasks.<br>5. Allows manual testing of CapSolver through a manual trigger.<br>### Setup steps<br>- [ ] Set up webhook credentials for 'Receive Solver Request'.<br>- [ ] Configure CapSolver with necessary keys for all CapSolver nodes.<br>- [ ] Assign necessary permissions for HTTP request to reCAPTCHA endpoint.<br>- [ ] Review and adjust the schedule in 'Schedule Trigger'.<br>- [ ] Test manual execution with 'Manual Trigger'. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it something like `reCAPTCHA v2`.

2. **Add a Webhook node** named `Receive Solver Request`.
   - Type: `Webhook`
   - HTTP Method: `POST`
   - Path: `solve-recaptcha-v2`
   - Response Mode: `Using Respond to Webhook Node`
   - If available, set error behavior to continue regular output as in the source workflow.

3. **Add a CapSolver node** named `CapSolver [Webhook]`.
   - Type: `CapSolver`
   - Connect `Receive Solver Request` → `CapSolver [Webhook]`
   - Configure:
     - `websiteKey` = `{{$json.body.websiteKey}}`
     - `websiteURL` = `{{$json.body.websiteURL}}`
     - `type` = `{{$json.body.taskType || 'ReCaptchaV2TaskProxyLess'}}`
     - `proxy` = `{{$json.body.proxy || ''}}`
   - Optional fields:
     - `apiDomain` = `{{$json.body.apiDomain || ''}}`
     - `isSession` = `{{$json.body.isSession || false}}`
     - `pageAction` = `{{$json.body.pageAction || ''}}`
     - `isInvisible` = `{{$json.body.isInvisible || false}}`
     - `enterprisePayload` = `{{$json.body.enterprisePayload || ''}}`
   - Attach CapSolver credentials if the node requires explicit credential assignment in your n8n instance.

4. **Add a Respond to Webhook node** named `Return Solver Result`.
   - Connect `CapSolver [Webhook]` → `Return Solver Result`
   - Set:
     - Respond With: `JSON`
     - Response Body: `{{ JSON.stringify($json.data) }}`
   - Note: if you prefer returning a native JSON object rather than a stringified body, adjust this during implementation.

5. **Add a Schedule Trigger node** named `Schedule Trigger (Every 1h)`.
   - Type: `Schedule Trigger`
   - Configure interval rule for every 1 hour.

6. **Add a Set node** named `Set Target Params`.
   - Connect `Schedule Trigger (Every 1h)` → `Set Target Params`
   - Add two string fields:
     - `websiteURL` = `https://www.google.com/recaptcha/api2/demo`
     - `websiteKey` = `6Le-wvkSAAAAAPBMRTvw0Q4Muexq9bi0DJwx_mJ-`

7. **Add a CapSolver node** named `CapSolver [Schedule]`.
   - Connect `Set Target Params` → `CapSolver [Schedule]`
   - Configure:
     - `websiteKey` = `{{$json.websiteKey}}`
     - `websiteURL` = `{{$json.websiteURL}}`
   - Leave optional parameters empty.
   - Assign the `CapSolver account` credential, or equivalent credential containing your CapSolver API key.

8. **Add an HTTP Request node** named `Submit Token`.
   - Connect `CapSolver [Schedule]` → `Submit Token`
   - Configure:
     - Method: `POST`
     - URL: `https://www.google.com/recaptcha/api2/demo`
     - Send Body: enabled
     - Body Content Type: `Form URL Encoded`
   - Add body parameter:
     - `g-recaptcha-response` = `{{$json.data.solution.gRecaptchaResponse}}`
   - Enable custom headers and add:
     - `content-type` = `application/x-www-form-urlencoded`
     - `user-agent` = `Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/125.0.0.0 Safari/537.36`

9. **Add an IF node** named `Check Result`.
   - Connect `Submit Token` → `Check Result`
   - Configure a string condition:
     - Left value: `{{$json.data}}`
     - Operation: `contains`
     - Right value: `recaptcha-success`
   - Keep strict type validation enabled if available.

10. **Add a Set node** named `Monitor Passed`.
    - Connect the **true** output of `Check Result` to `Monitor Passed`
    - Add fields:
      - `status` = `passed`
      - `message` = `reCAPTCHA v2 monitor passed successfully`
      - `timestamp` = `{{ new Date().toISOString() }}`

11. **Add another Set node** named `Monitor Failed`.
    - Connect the **false** output of `Check Result` to `Monitor Failed`
    - Add fields:
      - `status` = `failed`
      - `message` = `reCAPTCHA v2 monitor FAILED — token rejected`
      - `timestamp` = `{{ new Date().toISOString() }}`

12. **Add a Manual Trigger node** named `Manual Trigger (Test)`.
    - This creates a test-only branch for editor execution.

13. **Add a CapSolver node** named `CapSolver [Manual]`.
    - Connect `Manual Trigger (Test)` → `CapSolver [Manual]`
    - Configure:
      - `websiteKey` = `6Le-wvkSAAAAAPBMRTvw0Q4Muexq9bi0DJwx_mJ-`
      - `websiteURL` = `https://www.google.com/recaptcha/api2/demo`
    - Assign the same CapSolver credential used in the scheduled node.

14. **Add a Set node** named `Format Result`.
    - Connect `CapSolver [Manual]` → `Format Result`
    - Add fields:
      - `token` = `{{$json.data.solution.gRecaptchaResponse}}`
      - `taskId` = `{{$json.data.taskId}}`
      - `status` = `solved`
      - `solvedAt` = `{{ new Date().toISOString() }}`

15. **Add sticky notes** to document the canvas structure.
    - One general note for all triggers and setup steps
    - One note for webhook handling
    - One note for scheduled processing
    - One note for token validation

16. **Configure credentials**.
    - Create or reuse a **CapSolver API credential**.
    - Attach it to:
      - `CapSolver [Schedule]`
      - `CapSolver [Manual]`
      - and to `CapSolver [Webhook]` if your instance does not inherit credentials automatically.
    - No special credential is required for the Google demo HTTP POST unless your environment enforces outbound proxy/auth rules.

17. **Test the manual path**.
    - Run `Manual Trigger (Test)`
    - Confirm that `Format Result` contains:
      - a non-empty `token`
      - a `taskId`
      - status `solved`

18. **Test the schedule path manually**.
    - Execute from `Schedule Trigger (Every 1h)` or run the workflow manually through that branch
    - Confirm whether `Check Result` routes to `Monitor Passed`

19. **Test the webhook path** with a POST request to the webhook URL.
    - Example payload fields expected:
      - `websiteKey` required
      - `websiteURL` required
      - `taskType` optional
      - `proxy` optional
      - `apiDomain` optional
      - `isSession` optional
      - `pageAction` optional
      - `isInvisible` optional
      - `enterprisePayload` optional
    - Confirm a JSON response is returned from `Return Solver Result`

20. **Activate the workflow** if you want the hourly schedule and production webhook endpoint to run continuously.

### Expected webhook input shape
A caller should send a JSON body containing at least:
- `websiteKey`
- `websiteURL`

Optional fields:
- `taskType`
- `proxy`
- `apiDomain`
- `isSession`
- `pageAction`
- `isInvisible`
- `enterprisePayload`

### Expected CapSolver output assumptions
This workflow assumes the CapSolver node returns data in a structure similar to:
- `data.solution.gRecaptchaResponse`
- `data.taskId`

If your CapSolver node version returns a different structure, update:
- `Return Solver Result`
- `Submit Token`
- `Format Result`

### Sub-workflow setup
This workflow does **not** invoke any sub-workflows and does not require any child workflow configuration.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The workflow uses Google’s public reCAPTCHA v2 demo page as the validation target for both scheduled and manual test flows. | https://www.google.com/recaptcha/api2/demo |
| The webhook branch is intended to behave like a solver API endpoint that external systems can call on demand. | Internal workflow design |
| The scheduled branch is a monitoring mechanism, not just a solver path: it verifies token usefulness by posting the returned token to the target page. | Internal workflow design |
| The workflow is inactive by default (`active: false`), so scheduled runs and production webhook use require activation. | n8n workflow state |
| Execution order is configured as `v1`, and binary mode is set to `separate`. | Workflow settings |