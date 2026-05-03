Website Uptime Monitoring with GPT-4 Analysis and Gmail Notifications

https://n8nworkflows.xyz/workflows/website-uptime-monitoring-with-gpt-4-analysis-and-gmail-notifications-7194


# Website Uptime Monitoring with GPT-4 Analysis and Gmail Notifications

### 1. Workflow Overview

This workflow monitors website uptime by checking a provided URL’s HTTP status and analyzing the response using GPT-4. It then sends notification emails via Gmail with detailed AI-generated explanations about the site's status. The workflow is designed for automated uptime monitoring with intelligent diagnostics, suitable for web admins or monitoring services that want clear, AI-enhanced reports.

The workflow logic is structured into the following blocks:

- **1.1 Input Reception:** Receives URLs to check via a webhook POST request.
- **1.2 HTTP Status Retrieval:** Performs an HTTP request to the provided URL, capturing status and response details.
- **1.3 AI Analysis:** Uses GPT-4 (via OpenAI API) to interpret the HTTP response and determine if the site is up or down, including reasons.
- **1.4 Conditional Routing:** Evaluates AI output and HTTP status to decide if the site is considered up.
- **1.5 Notification Dispatch:** Sends Gmail notifications either confirming uptime or reporting errors based on the evaluation.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
Receives incoming POST requests with a JSON body containing the URL(s) to check. This node is the entry point for external triggers to start the uptime check.

**Nodes Involved:**  
- Webhook

**Node Details:**

- **Webhook**  
  - Type: Webhook  
  - Role: Entry point for external POST requests carrying URLs to check.  
  - Configuration:  
    - HTTP Method: POST  
    - Path: `/url-check`  
    - Expects JSON body in the form: `{ "url": ["https://example.com"] }`  
  - Expressions: The URL is accessed downstream via `$('Webhook').item.json.body.url[0]`.  
  - Inputs: External HTTP POST call.  
  - Outputs: Passes incoming JSON to HTTP Request node.  
  - Failure Modes: Missing or malformed JSON payload; non-POST requests will not trigger.  
  - Notes: Sticky note advises to adjust expressions if payload shape differs.  

---

#### 2.2 HTTP Status Retrieval

**Overview:**  
Executes an HTTP GET request to the received URL, capturing the full HTTP response (status code, headers, partial body). Configured to continue workflow on HTTP errors.

**Nodes Involved:**  
- HTTP Request

**Node Details:**

- **HTTP Request**  
  - Type: HTTP Request  
  - Role: Performs the actual URL check by requesting the website.  
  - Configuration:  
    - URL set dynamically from webhook URL input.  
    - Full HTTP response enabled to capture headers and status code.  
    - Allows unauthorized SSL certificates (for testing or self-signed certs).  
    - On error: set to continue with error output to avoid workflow failure on request errors.  
  - Expressions: `={{ $json.body.url[0] }}` used to get URL from webhook payload.  
  - Inputs: Webhook node output.  
  - Outputs: Passes response data to AI analysis node.  
  - Failure Modes: Network issues, DNS resolution failures, timeouts; these do not stop workflow due to error continuation.  
  - Notes: Sticky note recommends network/DNS errors will be explained by AI downstream.

---

#### 2.3 AI Analysis

**Overview:**  
Analyzes the HTTP response using GPT-4 to determine if the website is up or down, and provides a detailed reason based on status code, response snippet, and headers. Also handles specific DNS errors gracefully.

**Nodes Involved:**  
- Message a model

**Node Details:**

- **Message a model**  
  - Type: OpenAI (LangChain) node  
  - Role: Uses GPT-4.1-NANO to interpret the HTTP response details.  
  - Configuration:  
    - Model: `gpt-4.1-nano`  
    - Input message includes structured JSON: URL, status code, truncated response body snippet, and truncated headers.  
    - Instructions specify to detect if site is up or down and explain reasons.  
    - Special handling for DNS errors like `getaddrinfo ENOTFOUND`.  
    - JSON output enabled for structured results.  
  - Expressions reference multiple fields from HTTP Request node output and Webhook input:  
    - `$('Webhook').item.json.body.url[0]` for URL  
    - `$json.statusCode` for HTTP status  
    - `$json.data` for response snippet  
    - `JSON.stringify($json.headers).slice(0,500)` for headers  
  - Inputs: HTTP Request node output.  
  - Outputs: AI-generated JSON with site_status and reason fields.  
  - Failure Modes: API quota exceeded, network errors, malformed inputs, slow responses.  
  - Credentials: Uses stored OpenAI API credentials (no keys embedded in node).  
  - Notes: Sticky note emphasizes proper credential use and AI summarization capability.

---

#### 2.4 Conditional Routing

**Overview:**  
Checks if the AI determined the site status as "up" and verifies the HTTP status code equals 200. Routes workflow accordingly to success or error notification paths.

**Nodes Involved:**  
- If

**Node Details:**

- **If**  
  - Type: Conditional (If) node  
  - Role: Logical gate deciding if site is up based on AI’s output and HTTP status.  
  - Configuration:  
    - Condition 1: AI message content field `site_status` equals `"up"` (case-sensitive, strict validation).  
    - Condition 2: HTTP Request status code equals 200.  
    - Both must be true (AND combinator) for success path.  
  - Inputs: Output from AI analysis node.  
  - Outputs:  
    - True branch: to Gmail Success Message node.  
    - False branch: to Gmail Error Message node.  
  - Failure Modes: If AI output is missing or malformed, condition may fail unexpectedly.  
  - Notes: Sticky note advises adjusting logic if 3xx codes should be treated as "up".

---

#### 2.5 Notification Dispatch

**Overview:**  
Sends email notifications via Gmail based on site status — one for successful uptime confirmation, another for error reporting. Both include AI-generated explanations and URL details.

**Nodes Involved:**  
- Gmail Success Message  
- Gmail Error Message

**Node Details:**

- **Gmail Success Message**  
  - Type: Gmail  
  - Role: Sends notification email when site is up.  
  - Configuration:  
    - Recipient: `test@gmail.com` (replace with actual recipient)  
    - Subject: `"✅ Website is <site_status>: Status Check for <URL>"`  
    - HTML message includes URL, status, reason from AI output, and attribution.  
    - OAuth2 credentials stored in n8n.  
  - Inputs: True branch of If node.  
  - Outputs: Terminal node (no further connections).  
  - Failure Modes: Gmail OAuth2 token expiry, network issues, recipient email errors.  

- **Gmail Error Message**  
  - Type: Gmail  
  - Role: Sends notification email when site is down or error detected.  
  - Configuration:  
    - Recipient: `test@gmail.com`  
    - Subject: `"❌ Website Status Check for <URL>"`  
    - HTML message similar to success but reflects failure details.  
    - OAuth2 credentials stored separately but setup similarly.  
  - Inputs: False branch of If node.  
  - Outputs: Terminal node.  
  - Failure Modes: Same as Gmail Success Message.  

---

### 3. Summary Table

| Node Name            | Node Type                  | Functional Role                 | Input Node(s)       | Output Node(s)          | Sticky Note                                                                                               |
|----------------------|----------------------------|--------------------------------|---------------------|-------------------------|----------------------------------------------------------------------------------------------------------|
| Webhook              | Webhook                    | Input reception of URL payload | External HTTP POST  | HTTP Request            | **Input format**: POST JSON `{ "url": "https://example.com" }`. Adjust expressions if payload differs.  |
| HTTP Request         | HTTP Request               | Perform HTTP GET on URL        | Webhook             | Message a model         | **Error Handling**: Network/DNS errors explained by AI; HTTP non-200 treated as "down".                  |
| Message a model      | OpenAI (LangChain)         | Analyze HTTP response with GPT | HTTP Request        | If                      | **AI Model & Credentials**: Use OpenAI credentials; do not paste keys. AI summarizes response & errors.  |
| If                   | Conditional (If)           | Check if site is up and status 200 | Message a model    | Gmail Success Message (true), Gmail Error Message (false) | **Error Handling**: Adjust logic if 3xx codes should be "up".                                            |
| Gmail Success Message | Gmail                      | Send success notification email | If (true branch)    | None                    |                                                                                                          |
| Gmail Error Message   | Gmail                      | Send error notification email   | If (false branch)   | None                    |                                                                                                          |
| Sticky Note          | Sticky Note (Informational) | Notes on input format           | None                | None                    |                                                                                                          |
| Sticky Note1         | Sticky Note (Informational) | Notes on AI model & credentials | None                | None                    |                                                                                                          |
| Sticky Note2         | Sticky Note (Informational) | Notes on error handling         | None                | None                    |                                                                                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `url-check`  
   - Purpose: Receive JSON POST requests with a URL array under `body.url`.  
   - Credentials: None needed.  
   - Save node.

2. **Create HTTP Request Node**  
   - Type: HTTP Request  
   - URL: Set expression to `={{ $json.body.url[0] }}` to dynamically use the URL from webhook payload.  
   - Enable “Full Response” to capture headers, status code, and body.  
   - Allow Unauthorized Certificates: Enabled (true).  
   - On Error: Set to “Continue on Error” to capture HTTP errors and network issues without stopping workflow.  
   - Connect Webhook node’s main output to this node’s input.

3. **Create OpenAI Node (Message a model)**  
   - Type: OpenAI (LangChain)  
   - Model: Select `gpt-4.1-nano` or similar GPT-4 variant.  
   - Credentials: Use pre-configured OpenAI API credentials in n8n (do not paste API key).  
   - Input Messages: Configure with a template message including:  
     ```
     You are a helpful website uptime analyzer. Given an HTTP response, determine if the site is up or down, and explain why.
     {
       "url": "{{ $('Webhook').item.json.body.url[0] }}",
       "status_code": "{{ $json.statusCode }}",
       "body_snippet": "{{ $json.data }}",
       "headers": "{{ JSON.stringify($json.headers).slice(0, 500) }}"
     }
     {{ $('Webhook').item.json.body.url[0] }}

     If the results error out "getaddrinfo ENOTFOUND" respond back in a helpful manner.
     ```
   - Enable JSON output for structured AI response.  
   - Connect HTTP Request node’s main output to this node’s input.

4. **Create If Node**  
   - Type: If (Conditional)  
   - Conditions (AND):  
     - Expression 1: `{{$json.message.content.site_status}}` equals `"up"` (case-sensitive)  
     - Expression 2: HTTP status code from HTTP Request equals `200`  
   - Connect OpenAI node’s output to this node’s input.

5. **Create Gmail Success Message Node**  
   - Type: Gmail  
   - Credentials: Use OAuth2 Gmail credentials configured in n8n.  
   - Send To: Set recipient email (e.g., `test@gmail.com`).  
   - Subject: Template `"✅ Website is {{ $json.message.content.site_status }}: Status Check for {{ $('Webhook').item.json.body.url[0] }}"`  
   - Message (HTML):  
     ```
     <h2>🌐 Website Status Check Result</h2>
     <p><strong>URL:</strong> {{ $('Webhook').item.json.body.url[0] }}</p>
     <p><strong>Status:</strong> {{ $json.message.content.site_status }}</p>
     <p><strong>Reason:</strong><br>{{ $json.message.content.reason }}</p>
     <hr>
     <p>Generated by n8n + ChatGPT</p>
     ```  
   - Connect If node’s true output to this node.

6. **Create Gmail Error Message Node**  
   - Type: Gmail  
   - Credentials: Use same or separate OAuth2 Gmail credentials.  
   - Send To: Same recipient.  
   - Subject: Template `"❌ Website Status Check for {{ $('Webhook').item.json.body.url[0] }}"`  
   - Message (HTML): Similar to success but reflecting error status and reason fields from AI.  
   - Connect If node’s false output to this node.

7. **Optional: Add Sticky Notes**  
   - Add informational sticky notes for input format, AI credential usage, and error handling tips as per documented content.

8. **Activate Workflow**  
   - Test by sending POST request to `https://<your-n8n-instance>/webhook/url-check` with JSON body:  
     ```json
     { "url": ["https://example.com"] }
     ```  
   - Monitor emails sent for status and reason.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                               |
|---------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| Input JSON must be: `{ "url": ["https://example.com"] }` or update expressions accordingly if different.      | See Sticky Note on input format.                               |
| Use n8n credentials for OpenAI API, avoid pasting raw API keys.                                               | See Sticky Note1 on AI Model & Credentials.                   |
| HTTP errors and network/DNS errors are gracefully handled and explained by AI. Adjust IF logic if needed.     | See Sticky Note2 on Error Handling.                            |
| Workflow combines n8n automation with GPT-4 for intelligent uptime reports and Gmail notifications.           | Workflow overview and design rationale.                        |

---

**Disclaimer:**  
The provided text is generated exclusively from an automated n8n workflow. It adheres strictly to content policies and contains no illegal, offensive, or protected content. All data processed is legal and public.