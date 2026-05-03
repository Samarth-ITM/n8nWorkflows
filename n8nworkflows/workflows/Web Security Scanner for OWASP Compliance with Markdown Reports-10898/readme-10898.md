Web Security Scanner for OWASP Compliance with Markdown Reports

https://n8nworkflows.xyz/workflows/web-security-scanner-for-owasp-compliance-with-markdown-reports-10898


# Web Security Scanner for OWASP Compliance with Markdown Reports

### 1. Workflow Overview

This workflow is designed as a **Web Security Scanner** focused on assessing a target website's compliance with OWASP security best practices. It performs a series of automated security tests on a provided URL, analyzing HTTP headers, methods, cookies, HTTPS redirects, and CORS preflight behavior. The results are aggregated, analyzed, and compiled into a detailed Markdown report. If security issues are detected, the workflow formats an alert, converts the report to a file, and sends it via email; otherwise, it formats a success message.

The workflow can be logically divided into the following blocks:

- **1.1 Input Reception and Configuration:** Receives the target URL via a form trigger and sets up necessary configuration parameters.

- **1.2 Security Tests Execution:** Executes multiple HTTP requests to test different security aspects: headers, methods, cookies, HTTPS redirects, and CORS preflight.

- **1.3 Analysis of Test Results:** Processes and analyzes the raw HTTP responses from each test to extract meaningful security insights.

- **1.4 Aggregation and Reporting:** Merges all analysis results, generates a comprehensive report, and decides if issues exist.

- **1.5 Notification and Output:** Depending on detected issues, formats alerts or success messages, generates Markdown files, and sends email notifications.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Configuration

**Overview:**  
This block captures the target website URL through a form trigger and initializes configuration parameters necessary for subsequent security tests.

**Nodes Involved:**  
- Landing Page Url (formTrigger)  
- Configuration (set)  

**Node Details:**  

- **Landing Page Url**  
  - **Type:** Form Trigger  
  - **Role:** Entry point for the workflow, waits for a user to submit a URL via a web form.  
  - **Configuration:** Accepts input fields (likely a URL field though not explicitly shown).  
  - **Connections:** Outputs to Configuration node.  
  - **Edge Cases:** Missing or malformed URLs could cause downstream HTTP request failures.

- **Configuration**  
  - **Type:** Set  
  - **Role:** Initializes or sets workflow parameters such as headers, HTTP methods to test, or other constants required for HTTP requests.  
  - **Configuration:** Parameters are set statically or derived from input; exact fields not shown.  
  - **Connections:** Outputs to multiple HTTP request nodes for different tests.  
  - **Edge Cases:** Misconfiguration can cause incorrect tests or invalid requests.

---

#### 1.2 Security Tests Execution

**Overview:**  
Executes a series of HTTP requests to test various security aspects of the target URL. Each test runs independently, allowing partial failures without stopping the entire workflow.

**Nodes Involved:**  
- Test Security Headers (httpRequest)  
- Test HTTP Methods (httpRequest)  
- Test Cookies (httpRequest)  
- Test HTTPS Redirect (httpRequest)  
- Test CORS Preflight (httpRequest)  

**Node Details:**  

- **Test Security Headers**  
  - **Type:** HTTP Request  
  - **Role:** Makes a request to retrieve and test HTTP security headers (e.g., Content-Security-Policy, X-Frame-Options).  
  - **Configuration:** Standard GET or HEAD request with `continueOnFail` enabled to avoid workflow halt.  
  - **Connections:** Outputs to Analyze Headers.  
  - **Edge Cases:** Network timeouts, invalid certificates, or missing headers.

- **Test HTTP Methods**  
  - **Type:** HTTP Request  
  - **Role:** Tests allowed HTTP methods by sending OPTIONS or similar requests.  
  - **Configuration:** Sends OPTIONS or method-specific requests; `continueOnFail` enabled.  
  - **Connections:** Outputs to Analyze Methods.  
  - **Edge Cases:** Server blocking OPTIONS requests, unexpected methods allowed.

- **Test Cookies**  
  - **Type:** HTTP Request  
  - **Role:** Retrieves cookies set by the server and tests their security attributes.  
  - **Configuration:** Standard GET request with `continueOnFail` enabled.  
  - **Connections:** Outputs to Analyze Cookies.  
  - **Edge Cases:** No cookies present, insecure cookie flags.

- **Test HTTPS Redirect**  
  - **Type:** HTTP Request  
  - **Role:** Checks if HTTP requests are properly redirected to HTTPS.  
  - **Configuration:** HTTP request to the HTTP version of the URL; `continueOnFail` enabled.  
  - **Connections:** Outputs to Analyze HTTPS.  
  - **Edge Cases:** Missing redirects, redirect loops.

- **Test CORS Preflight**  
  - **Type:** HTTP Request  
  - **Role:** Sends a CORS preflight (OPTIONS) request to test cross-origin resource sharing policies.  
  - **Configuration:** OPTIONS request with relevant headers; `retryOnFail` enabled for robust testing.  
  - **Connections:** Outputs to Analyze CORS Preflight.  
  - **Edge Cases:** Server blocking preflights, CORS misconfigurations.

---

#### 1.3 Analysis of Test Results

**Overview:**  
Processes raw HTTP responses from each security test and extracts meaningful information about security compliance.

**Nodes Involved:**  
- Analyze Headers (code)  
- Analyze Methods (code)  
- Analyze Cookies (code)  
- Analyze HTTPS (code)  
- Analyze CORS Preflight (code)  

**Node Details:**  

- **Analyze Headers**  
  - **Type:** Code  
  - **Role:** Parses HTTP header responses to identify missing or misconfigured security headers.  
  - **Configuration:** Custom JavaScript code analyzing headers.  
  - **Connections:** Outputs to Merge node (index 0).  
  - **Edge Cases:** Unexpected header formats, empty responses.

- **Analyze Methods**  
  - **Type:** Code  
  - **Role:** Interprets HTTP methods allowed by the server and flags insecure ones.  
  - **Configuration:** Code filtering allowed methods against best practices.  
  - **Connections:** Outputs to Merge (index 1).  
  - **Edge Cases:** Server responses lacking method info.

- **Analyze Cookies**  
  - **Type:** Code  
  - **Role:** Examines cookies for security flags like HttpOnly, Secure, SameSite.  
  - **Configuration:** Code inspecting cookie attributes.  
  - **Connections:** Outputs to Merge (index 2).  
  - **Edge Cases:** No cookies or malformed cookie strings.

- **Analyze HTTPS**  
  - **Type:** Code  
  - **Role:** Analyzes redirect behavior to verify HTTPS enforcement.  
  - **Configuration:** Checks status codes and redirect URLs.  
  - **Connections:** Outputs to Merge (index 3).  
  - **Edge Cases:** Redirect loops or missing redirects.

- **Analyze CORS Preflight**  
  - **Type:** Code  
  - **Role:** Examines CORS headers for permissive or restrictive policies.  
  - **Configuration:** Code parsing Access-Control-Allow-Origin and related headers.  
  - **Connections:** Outputs to Merge (index 4).  
  - **Edge Cases:** No CORS headers, overly permissive policies.

---

#### 1.4 Aggregation and Reporting

**Overview:**  
Aggregates all analyzed results, generates a report summarizing findings, and determines if any security issues were found.

**Nodes Involved:**  
- Merge (merge)  
- Generate Report (code)  
- Issues Found? (if)  

**Node Details:**  

- **Merge**  
  - **Type:** Merge  
  - **Role:** Combines the outputs from all analysis nodes into a single data structure.  
  - **Configuration:** Likely configured to merge by index to keep different analyses separate.  
  - **Connections:** Outputs to Generate Report.  
  - **Edge Cases:** Missing inputs if any analysis node failed.

- **Generate Report**  
  - **Type:** Code  
  - **Role:** Synthesizes all merged data into an overall security report including issue counts and descriptions.  
  - **Configuration:** Custom code to format report data, possibly adding severity levels.  
  - **Connections:** Outputs to Issues Found? node.  
  - **Edge Cases:** Malformed merged data causing code errors.

- **Issues Found?**  
  - **Type:** If  
  - **Role:** Decision point that routes workflow based on whether any security issues were detected.  
  - **Configuration:** Conditional check on report data for presence of issues.  
  - **Connections:**  
    - True branch: Format Alert  
    - False branch: Format Success  
  - **Edge Cases:** Ambiguous or incomplete data causing incorrect routing.

---

#### 1.5 Notification and Output

**Overview:**  
Formats alerts or success messages, creates Markdown reports, converts them to files, and sends email notifications.

**Nodes Involved:**  
- Format Alert (set)  
- Format Success (set)  
- Create Markdown Report (code)  
- Markdown to File (convertToFile)  
- Send a message4 (gmail)  

**Node Details:**  

- **Format Alert**  
  - **Type:** Set  
  - **Role:** Prepares alert message content and metadata when issues are found.  
  - **Configuration:** Sets variables such as email subject, body, and severity.  
  - **Connections:** Outputs to Create Markdown Report.  
  - **Edge Cases:** Missing data causing incomplete alert messages.

- **Format Success**  
  - **Type:** Set  
  - **Role:** Prepares a success message when no issues are found.  
  - **Configuration:** Sets variables for positive confirmation; does not proceed to email sending.  
  - **Connections:** Terminal or further handling (not detailed).  
  - **Edge Cases:** None significant.

- **Create Markdown Report**  
  - **Type:** Code  
  - **Role:** Converts the alert or success data into a Markdown-formatted report.  
  - **Configuration:** Generates Markdown syntax including headings, tables, and descriptions.  
  - **Connections:** Outputs to Markdown to File.  
  - **Edge Cases:** Formatting errors if input data is incomplete.

- **Markdown to File**  
  - **Type:** Convert To File  
  - **Role:** Converts the Markdown text into a file object suitable for attachment.  
  - **Configuration:** Sets file name and MIME type (text/markdown).  
  - **Connections:** Outputs to Send a message4.  
  - **Edge Cases:** File creation errors.

- **Send a message4**  
  - **Type:** Gmail  
  - **Role:** Sends the generated report file via email to configured recipients.  
  - **Configuration:** Uses Gmail OAuth2 credentials; email parameters are dynamically set from previous nodes.  
  - **Connections:** Terminal node.  
  - **Edge Cases:** Authentication failures, email delivery issues, API rate limits.

---

### 3. Summary Table

| Node Name                   | Node Type         | Functional Role                          | Input Node(s)                        | Output Node(s)                  | Sticky Note                                      |
|-----------------------------|-------------------|----------------------------------------|------------------------------------|--------------------------------|-------------------------------------------------|
| Landing Page Url             | Form Trigger      | Receives target URL input               | —                                  | Configuration                  |                                                 |
| Configuration               | Set               | Initializes test parameters             | Landing Page Url                   | Test Security Headers, Test HTTP Methods, Test HTTPS Redirect, Test CORS Preflight, Test Cookies |                                                 |
| Test Security Headers        | HTTP Request      | Tests HTTP security headers             | Configuration                     | Analyze Headers                |                                                 |
| Analyze Headers              | Code              | Analyzes HTTP headers                    | Test Security Headers             | Merge (index 0)                |                                                 |
| Test HTTP Methods            | HTTP Request      | Tests allowed HTTP methods               | Configuration                     | Analyze Methods                |                                                 |
| Analyze Methods             | Code              | Analyzes HTTP methods                    | Test HTTP Methods                 | Merge (index 1)                |                                                 |
| Test Cookies                | HTTP Request      | Tests cookie security attributes         | Configuration                     | Analyze Cookies               |                                                 |
| Analyze Cookies             | Code              | Analyzes cookies                         | Test Cookies                     | Merge (index 2)                |                                                 |
| Test HTTPS Redirect          | HTTP Request      | Checks HTTP to HTTPS redirects           | Configuration                     | Analyze HTTPS                 |                                                 |
| Analyze HTTPS               | Code              | Analyzes HTTPS redirect behavior          | Test HTTPS Redirect              | Merge (index 3)                |                                                 |
| Test CORS Preflight          | HTTP Request      | Tests CORS preflight requests             | Configuration                     | Analyze CORS Preflight        |                                                 |
| Analyze CORS Preflight      | Code              | Analyzes CORS headers                    | Test CORS Preflight              | Merge (index 4)                |                                                 |
| Merge                       | Merge             | Aggregates all analyzed results          | Analyze Headers, Analyze Methods, Analyze Cookies, Analyze HTTPS, Analyze CORS Preflight | Generate Report               |                                                 |
| Generate Report             | Code              | Creates overall security report           | Merge                            | Issues Found?                 |                                                 |
| Issues Found?               | If                | Decides if issues exist                    | Generate Report                  | Format Alert (true), Format Success (false) |                                                 |
| Format Alert                | Set               | Prepares alert message                     | Issues Found? (true branch)       | Create Markdown Report         |                                                 |
| Format Success              | Set               | Prepares success message                   | Issues Found? (false branch)      | —                            |                                                 |
| Create Markdown Report      | Code              | Generates Markdown report                   | Format Alert                    | Markdown to File              |                                                 |
| Markdown to File            | ConvertToFile     | Converts Markdown to file                   | Create Markdown Report            | Send a message4               |                                                 |
| Send a message4             | Gmail             | Sends email notification with report      | Markdown to File                 | —                            |                                                 |
| Sticky Note                 | Sticky Note       | Various notes (mostly empty or informational) | —                              | —                            | Multiple notes exist but content mostly empty.  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger Node:**  
   - Name: `Landing Page Url`  
   - Purpose: Receive the target website URL as input.  
   - Configure form fields to accept URL input (e.g., single text field named "url").

2. **Add a Set Node:**  
   - Name: `Configuration`  
   - Purpose: Set parameters such as HTTP headers, methods to test, and other constants.  
   - Connect `Landing Page Url` → `Configuration`.  
   - Set parameters such as:  
     - Target URL: use expression referencing form input, e.g., `{{$node["Landing Page Url"].json["url"]}}`  
     - HTTP methods array (e.g., GET, POST, OPTIONS)  
     - Headers for requests (if any)

3. **Create HTTP Request Nodes for Tests:**  
   For each security test, create an HTTP Request node with the following configurations and connect from `Configuration`:  
   - `Test Security Headers`: GET or HEAD request to target URL, enable `Continue On Fail`.  
   - `Test HTTP Methods`: OPTIONS request to target URL, enable `Continue On Fail`.  
   - `Test Cookies`: GET request to target URL, enable `Continue On Fail`.  
   - `Test HTTPS Redirect`: HTTP request (http:// version of URL) to check redirect to HTTPS, enable `Continue On Fail`.  
   - `Test CORS Preflight`: OPTIONS request with CORS headers to target URL, enable `Retry On Fail`.

4. **Create Code Nodes to Analyze Each Test Result:**  
   Connect each HTTP Request node to its corresponding Code node:  
   - `Analyze Headers` connected from `Test Security Headers`  
   - `Analyze Methods` connected from `Test HTTP Methods`  
   - `Analyze Cookies` connected from `Test Cookies`  
   - `Analyze HTTPS` connected from `Test HTTPS Redirect`  
   - `Analyze CORS Preflight` connected from `Test CORS Preflight`

   Each Code node should parse respective HTTP response data and output structured findings.

5. **Add Merge Node:**  
   - Name: `Merge`  
   - Purpose: Combine all analysis outputs into one aggregated data object.  
   - Connect all `Analyze *` nodes as inputs (in order):  
     1. Analyze Headers  
     2. Analyze Methods  
     3. Analyze Cookies  
     4. Analyze HTTPS  
     5. Analyze CORS Preflight  
   - Use "Merge by index" or equivalent to preserve individual test results.

6. **Add Code Node to Generate Report:**  
   - Name: `Generate Report`  
   - Purpose: Process merged data to create a comprehensive security report including issue summaries.  
   - Connect `Merge` → `Generate Report`.

7. **Add If Node to Check for Issues:**  
   - Name: `Issues Found?`  
   - Purpose: Branch workflow based on presence of security issues in the report data.  
   - Connect `Generate Report` → `Issues Found?`.  
   - Configure conditional logic to evaluate if any issues exist.

8. **Add Set Node for Alert Formatting:**  
   - Name: `Format Alert`  
   - Purpose: Format alert messages, set email subject, body, and severity.  
   - Connect `Issues Found?` true branch → `Format Alert`.

9. **Add Set Node for Success Formatting:**  
   - Name: `Format Success`  
   - Purpose: Prepare success message when no issues found.  
   - Connect `Issues Found?` false branch → `Format Success`.

10. **Add Code Node to Create Markdown Report:**  
    - Name: `Create Markdown Report`  
    - Purpose: Convert alert or success data into Markdown report format.  
    - Connect `Format Alert` → `Create Markdown Report`.

11. **Add Convert To File Node:**  
    - Name: `Markdown to File`  
    - Purpose: Convert Markdown text into a file attachment.  
    - Configure file name (e.g., `security_report.md`) and MIME type `text/markdown`.  
    - Connect `Create Markdown Report` → `Markdown to File`.

12. **Add Gmail Node to Send Email:**  
    - Name: `Send a message4`  
    - Purpose: Send email with Markdown report attached.  
    - Configure Gmail OAuth2 credentials.  
    - Set dynamic email parameters such as recipient, subject, body from previous nodes.  
    - Attach file from `Markdown to File`.  
    - Connect `Markdown to File` → `Send a message4`.

13. **Optional:** Add Sticky Notes for documentation or instructions at relevant points.

---

### 5. General Notes & Resources

| Note Content                                                                                                 | Context or Link                                  |
|--------------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| The workflow performs automated OWASP compliance checks covering headers, methods, cookies, HTTPS, and CORS. | Workflow purpose summary.                        |
| Use caution with URLs to avoid scanning unauthorized or sensitive sites; ensure compliance with legal policies.| Security and ethical scanning reminder.         |
| Gmail node requires OAuth2 credentials with appropriate scopes to send emails.                                | Credential setup.                                |
| Markdown report generation allows easy integration with documentation or issue tracking systems.              | Reporting format advantage.                      |
| Retry and Continue On Fail options help improve robustness in face of partial failures or network issues.     | Reliability notes.                               |
| The workflow can be adapted or extended to include additional security tests or integrate with alert systems.| Extensibility guidance.                          |

---

**Disclaimer:** The text provided is sourced exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.