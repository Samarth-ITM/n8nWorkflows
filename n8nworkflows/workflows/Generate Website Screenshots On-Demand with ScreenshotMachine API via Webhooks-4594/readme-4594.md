Generate Website Screenshots On-Demand with ScreenshotMachine API via Webhooks

https://n8nworkflows.xyz/workflows/generate-website-screenshots-on-demand-with-screenshotmachine-api-via-webhooks-4594


# Generate Website Screenshots On-Demand with ScreenshotMachine API via Webhooks

### 1. Workflow Overview

This workflow provides an on-demand website screenshot generation service using the ScreenshotMachine API, triggered via HTTP POST webhooks. It securely processes incoming requests containing a target website URL, validates the URL to prevent SSRF (Server-Side Request Forgery) vulnerabilities, and returns either an error message or the screenshot image data to the requester.

Logical blocks:

- **1.1 Input Reception:** Receive incoming webhook POST requests containing the target website URL in JSON format.
- **1.2 URL Resolution and Validation:** Perform a HEAD request to resolve the URL, check accessibility, and validate the URL structure and security constraints to prevent SSRF.
- **1.3 Screenshot Request:** If validation passes, request a screenshot from ScreenshotMachine API using the validated URL.
- **1.4 Response Handling:** Return either the screenshot data or an error message back to the webhook caller.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block listens for incoming POST requests on a webhook endpoint and expects a JSON body containing a 'url' property representing the target website to screenshot.

- **Nodes Involved:**  
  - Receive URL Webhook  
  - Note: Webhook Input (sticky note)

- **Node Details:**

  - **Receive URL Webhook**  
    - Type: Webhook node  
    - Role: Entry point for the workflow; listens for HTTP POST requests at the configured path.  
    - Configuration:  
      - HTTP Method: POST  
      - Path: `caf8f5dc-4834-45bb-96b0-d4b508f93e1b` (unique webhook path)  
      - Response Mode: `responseNode` (workflow sends response explicitly via Respond nodes)  
    - Inputs: External HTTP POST request  
    - Outputs: Passes the received data to next node  
    - Edge cases: Invalid or missing 'url' property in JSON body may lead to downstream errors or validation failures.

  - **Note: Webhook Input**  
    - Type: Sticky Note  
    - Content: Explains the expected input format (JSON with 'url' property).

---

#### 2.2 URL Resolution and Validation

- **Overview:**  
  This block ensures the provided URL is reachable and secure by performing a HEAD request to resolve the final URL after redirects and checks for SSRF vulnerabilities by validating protocol, hostname, and IP addresses.

- **Nodes Involved:**  
  - Resolve URL (HEAD Request) (HTTP Request node)  
  - Validate URL for SSRF (Code node)  
  - IF URL Valid (If node)  
  - Note: Resolve URL (HEAD Request) (sticky note)  
  - Note: URL Validation & Security (sticky note)

- **Node Details:**

  - **Resolve URL (HEAD Request)**  
    - Type: HTTP Request  
    - Role: Sends a HEAD request to the URL received from the webhook to check connectivity and resolve any redirects.  
    - Configuration:  
      - URL: `={{$json.body.url}}` (dynamic from incoming webhook JSON property)  
      - Method: HEAD  
      - On Error: `continueErrorOutput` (allows workflow to proceed even if request errors)  
    - Inputs: Webhook data  
    - Outputs: Response with HTTP status code, headers, and resolved URL information  
    - Edge cases: Request timeouts, unreachable URL, redirects, or errors are logged and handled later.

  - **Validate URL for SSRF**  
    - Type: Code (JavaScript)  
    - Role: Validates the resolved URL for security: ensures protocol is HTTP/HTTPS, hostname is not localhost or private IP, and extracts final validated URL for safe use.  
    - Key logic:  
      - Checks HEAD response status code (must be 2xx or 3xx)  
      - Parses protocol and hostname manually from resolved URL string  
      - Rejects non-http(s) protocols  
      - Rejects localhost (`localhost`, `127.0.0.1`) and private IP ranges (10.x.x.x, 172.16.x.x–172.31.x.x, 192.168.x.x)  
      - Sets validation flags and error messages accordingly in workflow data  
    - Inputs: HEAD request response  
    - Outputs: Validation status (`isValidUrl`), error messages, validated URL, and HTTP status code for debugging  
    - Edge cases: Missing resolved URL info, invalid protocol, inability to parse hostname, private IP detection, unreachable URL.

  - **IF URL Valid**  
    - Type: If node  
    - Role: Branches workflow based on validation result from the Code node.  
    - Condition: Passes if no error message or error exists in JSON (i.e. URL is valid)  
    - Inputs: Validation node output  
    - Outputs:  
      - True branch: URL is valid, continue to screenshot call  
      - False branch: URL invalid, respond with error  
    - Edge cases: Misconfiguration of condition could incorrectly route workflow.

  - **Note: Resolve URL (HEAD Request)** and **Note: URL Validation & Security**  
    - Sticky notes clarifying these nodes’ purpose, emphasizing SSRF security checks, and describing the approach taken due to environment limitations (lack of URL object, reliance on string parsing).

---

#### 2.3 Screenshot Request

- **Overview:**  
  If the URL passes validation, this block makes a GET request to the ScreenshotMachine API to capture a screenshot of the target website.

- **Nodes Involved:**  
  - Take Screenshot (HTTP Request)  
  - Note: Screenshot API Call (GET) (sticky note)

- **Node Details:**

  - **Take Screenshot**  
    - Type: HTTP Request  
    - Role: Requests screenshot image from ScreenshotMachine API using the validated URL.  
    - Configuration:  
      - Method: GET  
      - URL: `https://api.screenshotmachine.com?key=YOUR_API_KEY&url={{$json.validatedUrl}}`  
      - Note: The placeholder `YOUR_API_KEY` must be replaced with an actual ScreenshotMachine API key before use.  
    - Inputs: Validated URL from previous if node  
    - Outputs: Screenshot data (image or other response)  
    - Edge cases: API key missing or invalid, API request failures, network timeouts.

  - **Note: Screenshot API Call (GET)**  
    - Sticky note reminding users to replace the placeholder API key and describing that ScreenshotMachine uses GET for screenshot requests.

---

#### 2.4 Response Handling

- **Overview:**  
  Sends back appropriate response data to the original webhook caller, either the screenshot data if successful or a detailed validation error message.

- **Nodes Involved:**  
  - Respond with Screenshot Data (Respond to Webhook)  
  - Respond with Validation Error (Respond to Webhook)  
  - Note: Webhook Response (sticky note)

- **Node Details:**

  - **Respond with Screenshot Data**  
    - Type: Respond to Webhook  
    - Role: Sends the screenshot data received from ScreenshotMachine back to the requester.  
    - Configuration: Responds with all incoming items (full HTTP response from screenshot node).  
    - Inputs: Screenshot HTTP Request node output  
    - Outputs: Final HTTP response to webhook caller  
    - Edge cases: Large image payloads may affect performance or timeout.

  - **Respond with Validation Error**  
    - Type: Respond to Webhook  
    - Role: Sends a text response containing the validation error message if URL validation fails.  
    - Configuration:  
      - Response Body: `={{ $json.error.  }}` (likely a minor syntax typo, intended to output error message text)  
      - Responds with plain text  
    - Inputs: From IF node’s false branch (validation failed)  
    - Outputs: HTTP error message response  
    - Edge cases: If error message is empty or malformed, response may be unclear.

  - **Note: Webhook Response**  
    - Sticky note describing the two possible response scenarios: error message on validation failure or screenshot data on success.

---

### 3. Summary Table

| Node Name                  | Node Type              | Functional Role                         | Input Node(s)               | Output Node(s)                 | Sticky Note                                                                                              |
|----------------------------|------------------------|---------------------------------------|----------------------------|-------------------------------|--------------------------------------------------------------------------------------------------------|
| Receive URL Webhook         | Webhook                | Entry point: receive POST with URL    | -                          | Resolve URL (HEAD Request)     | This node listens for incoming POST requests. It expects a JSON body with a 'url' property (the website URL you want to screenshot). |
| Resolve URL (HEAD Request)  | HTTP Request           | Resolve URL redirects and check reachability | Receive URL Webhook         | Validate URL for SSRF, IF URL Valid | This node performs a HEAD request to the incoming URL. It checks if the URL is reachable and resolves any redirects. This acts as an initial connectivity and basic validity check before more granular SSRF validation. |
| Validate URL for SSRF       | Code                   | Validate URL protocol and hostname to prevent SSRF | Resolve URL (HEAD Request)  | IF URL Valid                  | This crucial node validates the incoming 'url' to prevent Server-Side Request Forgery (SSRF) vulnerabilities. It checks for valid HTTP/HTTPS protocols and ensures the URL does not point to internal/private IP addresses or localhost. Since 'URL' object is unavailable, it uses string-based parsing and relies on the preceding HEAD request for initial URL resolution and connectivity check. |
| IF URL Valid               | If                     | Branch based on URL validation outcome | Validate URL for SSRF       | Take Screenshot (true), Respond with Validation Error (false) |                                                                                                        |
| Take Screenshot            | HTTP Request           | Request screenshot from ScreenshotMachine API | IF URL Valid (true)         | Respond with Screenshot Data   | This node makes an HTTP GET request to the ScreenshotMachine API using the validated URL. Remember to replace 'YOUR_API_KEY' in the URL parameter with your actual API key. This method is critical: ScreenshotMachine typically uses GET for this type of request. |
| Respond with Screenshot Data| Respond to Webhook     | Send screenshot data back to caller   | Take Screenshot            | -                             | 1. If the URL is invalid or blocked by security checks, it sends a clear error message. 2. If the screenshot is successful, it sends the data received from ScreenshotMachine API back to the original webhook caller. |
| Respond with Validation Error| Respond to Webhook    | Send validation error message to caller | IF URL Valid (false)        | -                             |                                                                                                        |
| Note: Webhook Input        | Sticky Note            | Describes expected webhook input format | -                          | -                             | This node listens for incoming POST requests. It expects a JSON body with a 'url' property (the website URL you want to screenshot). |
| Note: Resolve URL (HEAD Request) | Sticky Note       | Explains HEAD request role for URL resolution | -                          | -                             | This node performs a HEAD request to the incoming URL. It checks if the URL is reachable and resolves any redirects. This acts as an initial connectivity and basic validity check before more granular SSRF validation. |
| Note: URL Validation & Security | Sticky Note         | Explains SSRF prevention validation logic | -                          | -                             | This crucial node validates the incoming 'url' to prevent Server-Side Request Forgery (SSRF) vulnerabilities. It checks for valid HTTP/HTTPS protocols and ensures the URL does not point to internal/private IP addresses or localhost. Since 'URL' object is unavailable, it uses string-based parsing and relies on the preceding HEAD request for initial URL resolution and connectivity check. |
| Note: Screenshot API Call (GET) | Sticky Note         | Reminder about ScreenshotMachine API GET request and API key replacement | -                          | -                             | This node makes an HTTP GET request to the ScreenshotMachine API using the validated URL. Remember to replace 'YOUR_API_KEY' in the URL parameter with your actual API key. This method is critical: ScreenshotMachine typically uses GET for this type of request. |
| Note: Webhook Response     | Sticky Note            | Explains response logic for success and error | -                          | -                             | 1. If the URL is invalid or blocked by security checks, it sends a clear error message. 2. If the screenshot is successful, it sends the data received from ScreenshotMachine API back to the original webhook caller. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node: "Receive URL Webhook"**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `caf8f5dc-4834-45bb-96b0-d4b508f93e1b` (can be customized)  
   - Response Mode: `responseNode` (to send responses manually)  
   - No credentials needed  
   - Connect next to "Resolve URL (HEAD Request)"  

2. **Create HTTP Request Node: "Resolve URL (HEAD Request)"**  
   - Type: HTTP Request  
   - Method: HEAD  
   - URL: `={{$json.body.url}}` (dynamic, from incoming webhook JSON)  
   - On Error: `continueErrorOutput` (to handle unreachable URLs without stopping workflow)  
   - Connect next to "Validate URL for SSRF"  

3. **Create Code Node: "Validate URL for SSRF"**  
   - Type: Code (JavaScript)  
   - Paste the provided validation code (see below) that:  
     - Checks HEAD response status code  
     - Extracts protocol and hostname via string parsing  
     - Validates protocol is http or https  
     - Rejects localhost and private IP addresses  
     - Sets `isValidUrl`, `errorMessage`, `validatedUrl` in output JSON  
   - Connect next to "IF URL Valid"  

4. **Create If Node: "IF URL Valid"**  
   - Condition: Check if error message or error does not exist or is empty:  
     - Expression: Check `$json.errorMessage` or `$json.error` does NOT exist (not empty)  
   - True branch: connect to "Take Screenshot"  
   - False branch: connect to "Respond with Validation Error"  

5. **Create HTTP Request Node: "Take Screenshot"**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://api.screenshotmachine.com?key=YOUR_API_KEY&url={{$json.validatedUrl}}`  
   - Replace `YOUR_API_KEY` with your actual ScreenshotMachine API key  
   - Connect next to "Respond with Screenshot Data"  

6. **Create Respond to Webhook Node: "Respond with Screenshot Data"**  
   - Respond With: All incoming items (full response)  
   - Connect no further nodes (end of successful flow)  

7. **Create Respond to Webhook Node: "Respond with Validation Error"**  
   - Respond With: Text  
   - Response Body: Use expression to output error message from validation node, e.g.:  
     `={{ $json.errorMessage || $json.error || 'Invalid URL provided.' }}`  
   - Connect no further nodes (end of error flow)  

8. **Create Sticky Notes** (optional but recommended for clarity):  
   - Near "Receive URL Webhook": explain expected JSON input with 'url' property  
   - Near "Resolve URL (HEAD Request)": describe purpose of HEAD request for connectivity and redirects  
   - Near "Validate URL for SSRF": describe SSRF protections and parsing logic  
   - Near "Take Screenshot": remind to replace API key and note GET method for ScreenshotMachine  
   - Near response nodes: explain response behavior for errors and successes  

9. **Save and activate the workflow.**  

---

### 5. General Notes & Resources

| Note Content                                                                                                                     | Context or Link                                                                                         |
|----------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| The workflow prevents SSRF by validating URLs using manual parsing and IP checks due to the absence of the URL object in the environment. | Security best practice for safely handling user-provided URLs in HTTP requests.                        |
| ScreenshotMachine API requires a valid API key and uses GET requests to generate screenshots on-demand.                          | API documentation: https://www.screenshotmachine.com/api-documentation.php                            |
| To test the webhook, send a POST request with JSON body `{ "url": "https://example.com" }` to the webhook URL.                   | Use tools like Postman or curl for manual testing.                                                    |
| Error handling on HEAD request node is set to continue on error to prevent workflow failure on unreachable URLs.                 | This allows the workflow to respond gracefully with validation errors.                                |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, a powerful integration and automation tool. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected material. All handled data is legal and public.