Website & API Health Monitoring System with HTTP Status Validation

https://n8nworkflows.xyz/workflows/website---api-health-monitoring-system-with-http-status-validation-8412


# Website & API Health Monitoring System with HTTP Status Validation

### 1. Workflow Overview

This workflow implements a **Website & API Health Monitoring System** that validates HTTP status codes and performs automated health checks on JSON API responses. It is designed to receive monitoring requests via a webhook, perform HTTP requests ("pings") to target URLs, validate if the HTTP status code is below a configurable threshold, and analyze the JSON response body for health-related indicators. Based on these checks, it returns a structured JSON response indicating if the target is healthy or details of failure.

**Target Use Cases:**  
- API or website uptime and health monitoring with HTTP status validation  
- Automated health checks for APIs returning JSON health indicators  
- Integration with external systems via webhook to trigger checks and receive structured results  

**Logical Blocks:**  
- **1.1 Input Reception:** Receives webhook POST requests with monitoring parameters.  
- **1.2 Request Preparation:** Sets up defaults and extracts parameters for HTTP request.  
- **1.3 HTTP Ping and Status Validation:** Executes HTTP request and checks HTTP status code.  
- **1.4 Automated Health Check:** Analyzes JSON response body for health indicators.  
- **1.5 Decision Routing:** Routes based on health check pass/fail and status code validation.  
- **1.6 Response Handling:** Sends back JSON response to the webhook caller indicating results.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Receives incoming POST webhook requests containing parameters for the website/API check (e.g., URL, timeout, expected status limit).  
- **Nodes Involved:**  
  - Trigger (Webhook)  
- **Node Details:**  
  - **Trigger (Webhook)**  
    - Type: Webhook trigger node  
    - Configuration: Listens for POST requests at path `/website-check`. Response mode set to respond with node output.  
    - Key expressions: Extracts parameters from `$json.body` (e.g., `url`, `timeoutMs`, `expectStatusUnder`)  
    - Input: External HTTP POST request  
    - Output: JSON object with the request body forwarded downstream  
    - Edge cases: Missing or malformed JSON body, missing required parameters (no explicit validation in the workflow)  
    - Version: v2

#### 2.2 Request Preparation

- **Overview:** Initializes or sets default values for the incoming request parameters before the HTTP ping.  
- **Nodes Involved:**  
  - Set Defaults  
- **Node Details:**  
  - **Set Defaults**  
    - Type: Set node  
    - Configuration: No explicit fields set, presumably passes input as-is or sets defaults if configured externally  
    - Key expressions: None defined explicitly  
    - Input: Output from webhook trigger  
    - Output: Forwarded data to HTTP request node  
    - Edge cases: Missing parameters might cause HTTP request errors downstream  
    - Version: v3

#### 2.3 HTTP Ping and Status Validation

- **Overview:** Performs the HTTP request to the target URL and checks if the returned HTTP status code is below the expected threshold.  
- **Nodes Involved:**  
  - HTTP (Ping)  
  - IF Status < expect  
  - Ping → FAIL (Status)  
- **Node Details:**  
  - **HTTP (Ping)**  
    - Type: HTTP Request node  
    - Configuration:  
      - URL dynamically set using expression `{{$json.body.url}}` from the webhook payload  
      - Timeout dynamically set using expression `{{$json.body.timeoutMs}}`  
      - Response returned fully with headers and body parsed automatically  
    - Input: Set Defaults output with required parameters  
    - Output: HTTP response with statusCode and JSON body available  
    - Edge cases: Timeout errors, invalid URL, network errors, non-JSON response bodies  
    - Version: v4  
  - **IF Status < expect**  
    - Type: IF conditional node  
    - Configuration: Checks if HTTP status code (`$json.statusCode`) is smaller than expected threshold from webhook request (`$json.body.expectStatusUnder`)  
    - Input: HTTP (Ping) output  
    - Output:  
      - True branch if status < expected (proceed to health check)  
      - False branch if status ≥ expected (fail with status error)  
    - Edge cases: Missing or invalid statusCode, missing expectStatusUnder could cause expression failure  
    - Version: v2  
  - **Ping → FAIL (Status)**  
    - Type: Code node  
    - Configuration: Returns JSON indicating failure due to unacceptable HTTP status code, includes timestamp  
    - Input: IF Status < expect false branch  
    - Output: Failure JSON sent downstream to Respond node  
    - Version: v2

#### 2.4 Automated Health Check

- **Overview:** Analyzes the JSON response body from the HTTP ping to detect health indicators such as status, health, state, or ok fields. Determines if the service is healthy based on these indicators.  
- **Nodes Involved:**  
  - Health Check (Auto)  
  - IF Health passes  
  - Ping → OK  
  - Ping → FAIL (Health)  
- **Node Details:**  
  - **Health Check (Auto)**  
    - Type: Code node  
    - Configuration: JavaScript logic inspects `$json.body.body` for health indicators: fields like `status`, `health`, `state` with values `"ok"`, `"UP"`, or `"healthy"`, or boolean `ok === true`. If no indicators but valid JSON response, assumes healthy. Otherwise, marks as unhealthy.  
    - Output fields: `healthCheck`, `healthStatus`, `pass` (boolean), `responseBody` (the body analyzed)  
    - Input: IF Status < expect true branch (i.e., acceptable status code)  
    - Edge cases: Non-JSON or malformed JSON body triggers failure; unexpected JSON structure might cause false negatives  
    - Version: v2  
  - **IF Health passes**  
    - Type: IF conditional node  
    - Configuration: Checks if `pass` field from previous node is `true`  
    - Input: Health Check (Auto) output  
    - Output:  
      - True branch: Health check passed → Ping → OK  
      - False branch: Health check failed → Ping → FAIL (Health)  
    - Version: v2  
  - **Ping → OK**  
    - Type: Code node  
    - Configuration: Returns success JSON including status and health status with timestamp  
    - Input: IF Health passes true branch  
    - Output: Success JSON to Respond node  
    - Version: v2  
  - **Ping → FAIL (Health)**  
    - Type: Code node  
    - Configuration: Returns failure JSON with error message indicating health check failure, status code, health status, response body, and timestamp  
    - Input: IF Health passes false branch  
    - Output: Failure JSON to Respond node  
    - Version: v2

#### 2.5 Response Handling

- **Overview:** Sends the final JSON response back to the webhook caller.  
- **Nodes Involved:**  
  - Respond (JSON)  
- **Node Details:**  
  - **Respond (JSON)**  
    - Type: Respond to Webhook node  
    - Configuration:  
      - HTTP response code 200  
      - Responds with JSON output from preceding node (success or failure JSON)  
    - Input: Ping → OK, Ping → FAIL (Health), Ping → FAIL (Status) nodes  
    - Output: HTTP response to the original webhook request  
    - Edge cases: None significant; if no output, response may be empty or error  
    - Version: v1

---

### 3. Summary Table

| Node Name        | Node Type               | Functional Role                    | Input Node(s)              | Output Node(s)                   | Sticky Note                                                                                      |
|------------------|-------------------------|----------------------------------|---------------------------|---------------------------------|-------------------------------------------------------------------------------------------------|
| Trigger (Webhook)| Webhook Trigger         | Receives monitoring request      | (external HTTP POST)       | Set Defaults                    | ## START HERE - Send POST request to webhook with URL parameter - Configure timeout and status expectations - Example: {"url": "https://your-site.com"} |
| Set Defaults     | Set                     | Prepare/initialize parameters    | Trigger (Webhook)          | HTTP (Ping)                    |                                                                                                 |
| HTTP (Ping)      | HTTP Request            | Execute HTTP request to URL      | Set Defaults               | IF Status < expect             |                                                                                                 |
| IF Status < expect| IF                      | Validate HTTP status code        | HTTP (Ping)                | Health Check (Auto), Ping → FAIL (Status) |                                                                                                 |
| Health Check (Auto)| Code                   | Analyze JSON response health     | IF Status < expect         | IF Health passes               | ## HEALTH VALIDATION - Analyzes JSON response for health indicators - Looks for status, health, or ok fields in response body - No valid JSON response triggers health failure |
| IF Health passes | IF                      | Evaluate health check result     | Health Check (Auto)        | Ping → OK, Ping → FAIL (Health)|                                                                                                 |
| Ping → OK        | Code                    | Return success JSON result       | IF Health passes           | Respond (JSON)                 | ## RESPONSE PATHS - OK: Status and health checks pass - FAIL (Health): Health check fails despite good status - FAIL (Status): HTTP status code above expected threshold |
| Ping → FAIL (Health)| Code                  | Return failure JSON (health)     | IF Health passes           | Respond (JSON)                 |                                                                                                 |
| Ping → FAIL (Status)| Code                  | Return failure JSON (status)     | IF Status < expect         | Respond (JSON)                 |                                                                                                 |
| Respond (JSON)   | Respond to Webhook      | Send final JSON response         | Ping → OK, Ping → FAIL (Health), Ping → FAIL (Status) | (HTTP response)                 |                                                                                                 |
| Sticky Note      | Sticky Note             | Instructional note               |                           |                                 |                                                                                                 |
| Sticky Note1     | Sticky Note             | Instructional note               |                           |                                 |                                                                                                 |
| Sticky Note2     | Sticky Note             | Instructional note               |                           |                                 |                                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Trigger Node**  
   - Type: Webhook  
   - Name: `Trigger (Webhook)`  
   - HTTP Method: POST  
   - Path: `website-check`  
   - Response Mode: `responseNode` (respond with downstream node output)  
   - This node will receive JSON POST requests containing at least `url`, optionally `timeoutMs` and `expectStatusUnder`.

2. **Create Set Node for Defaults**  
   - Type: Set  
   - Name: `Set Defaults`  
   - No explicit fields configured unless you want to set default values for `timeoutMs`, `expectStatusUnder` here. Otherwise, just forward input as-is.  
   - Connect output of `Trigger (Webhook)` to this node.

3. **Create HTTP Request Node for Ping**  
   - Type: HTTP Request  
   - Name: `HTTP (Ping)`  
   - URL: Use expression `={{ $json.body.url }}`  
   - Timeout: Use expression `={{ $json.body.timeoutMs }}` (make sure the input JSON includes this or handle default)  
   - Response Format: JSON (default)  
   - Connect output of `Set Defaults` to this node.

4. **Create IF Node to Check Status Code**  
   - Type: IF  
   - Name: `IF Status < expect`  
   - Condition: Number  
     - Value 1: `={{ $json.statusCode }}` (HTTP status code from HTTP node output)  
     - Operation: Smaller  
     - Value 2: `={{ $json.body.expectStatusUnder }}` (from original webhook input)  
   - Connect output of `HTTP (Ping)` to this node.

5. **Create Code Node for Failure Due to HTTP Status**  
   - Type: Code  
   - Name: `Ping → FAIL (Status)`  
   - JavaScript:  
     ```javascript
     return {
       ok: false,
       error: 'HTTP status not acceptable',
       statusCode: $json.body.statusCode,
       timestamp: new Date().toISOString()
     };
     ```  
   - Connect the **false** output of `IF Status < expect` to this node.

6. **Create Code Node for Automated Health Check**  
   - Type: Code  
   - Name: `Health Check (Auto)`  
   - JavaScript:  
     ```javascript
     const body = $json.body.body;
     let healthStatus = 'unknown';
     let isHealthy = false;

     if (body && typeof body === 'object') {
       if (body.status === 'ok' || body.status === 'UP' || body.status === 'healthy') {
         healthStatus = body.status;
         isHealthy = true;
       } else if (body.health === 'ok' || body.health === 'UP' || body.health === 'healthy') {
         healthStatus = body.health;
         isHealthy = true;
       } else if (body.state === 'ok' || body.state === 'UP' || body.state === 'healthy') {
         healthStatus = body.state;
         isHealthy = true;
       } else if (body.ok === true) {
         healthStatus = 'ok';
         isHealthy = true;
       } else {
         healthStatus = 'response_received';
         isHealthy = true;
       }
     } else {
       healthStatus = 'no_json';
       isHealthy = false;
     }

     return {
       healthCheck: 'performed',
       healthStatus: healthStatus,
       pass: isHealthy,
       responseBody: body
     };
     ```  
   - Connect the **true** output of `IF Status < expect` to this node.

7. **Create IF Node to Evaluate Health Check Result**  
   - Type: IF  
   - Name: `IF Health passes`  
   - Condition: Boolean  
     - Value 1: `={{ $json.pass }}`  
     - Operation: True  
   - Connect output of `Health Check (Auto)` to this node.

8. **Create Code Node for Successful Ping**  
   - Type: Code  
   - Name: `Ping → OK`  
   - JavaScript:  
     ```javascript
     return {
       ok: true,
       detail: 'Status and Health OK',
       statusCode: $json.body.statusCode,
       healthStatus: $json.body.healthStatus,
       timestamp: new Date().toISOString()
     };
     ```  
   - Connect **true** output of `IF Health passes` to this node.

9. **Create Code Node for Failed Health Check**  
   - Type: Code  
   - Name: `Ping → FAIL (Health)`  
   - JavaScript:  
     ```javascript
     return {
       ok: false,
       error: 'Health check failed',
       statusCode: $json.body.statusCode,
       healthStatus: $json.body.healthStatus,
       responseBody: $json.body.responseBody,
       timestamp: new Date().toISOString()
     };
     ```  
   - Connect **false** output of `IF Health passes` to this node.

10. **Create Respond to Webhook Node**  
    - Type: Respond to Webhook  
    - Name: `Respond (JSON)`  
    - Response Code: 200  
    - Respond With: JSON  
    - Response Body: Use expression `={{ $json }}` to output the JSON from previous node  
    - Connect outputs of `Ping → OK`, `Ping → FAIL (Health)`, and `Ping → FAIL (Status)` nodes to this node.

11. **Optional: Create Sticky Notes for Documentation**  
    - Create notes explaining:  
      - How to trigger the webhook with example payload  
      - Explanation of health validation logic  
      - Description of response paths

---

### 5. General Notes & Resources

| Note Content                                                                                                                               | Context or Link                                                                                   |
|--------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| To trigger this workflow, send a POST request with JSON body e.g., `{"url":"https://your-site.com","timeoutMs":5000,"expectStatusUnder":400}` | See Sticky Note near Trigger (Webhook) node                                                    |
| Health validation inspects fields: `status`, `health`, `state` with values `ok`, `UP`, `healthy` or boolean `ok === true`                   | See Sticky Note near Health Check (Auto) node                                                  |
| Response possible paths: OK (status and health pass), FAIL (Health) (health check fails), FAIL (Status) (HTTP status code too high)         | See Sticky Note near Ping → OK node                                                             |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created using n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.