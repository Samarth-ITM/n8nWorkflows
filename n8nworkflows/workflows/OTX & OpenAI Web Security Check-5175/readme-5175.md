OTX & OpenAI Web Security Check

https://n8nworkflows.xyz/workflows/otx---openai-web-security-check-5175


# OTX & OpenAI Web Security Check

### 1. Workflow Overview

This n8n workflow, titled **"OTX & OpenAI Web Security Check"**, is designed to perform a comprehensive security audit of any user-submitted website URL. It leverages multiple data sources and AI-powered analysis to identify vulnerabilities and misconfigurations, and then delivers a detailed security report via email.

**Target Use Cases:**  
- Security teams or webmasters wanting an automated security posture overview of a given website.  
- Automated threat intelligence enrichment by combining HTTP response inspection with AlienVault OTX data.  
- AI-powered vulnerability explanation and mitigation guidance.  

**Logical Blocks:**  
- **1.1 Input Reception:** User submits a URL via a form trigger node.  
- **1.2 Data Gathering:** The workflow performs an HTTP request to capture site headers/response, and queries AlienVault OTX for threat intelligence on the domain.  
- **1.3 Data Preparation:** Consolidates HTTP and AlienVault data, extracts potential issues.  
- **1.4 AI Processing:** Sends the consolidated data to an OpenAI-powered LangChain agent for expert security analysis and report generation.  
- **1.5 Report Formatting:** Formats the AI output into an HTML email report.  
- **1.6 Reporting:** Sends the formatted report via Gmail to a specified recipient.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** Captures user input URL through a web form to initiate the security check.  
- **Nodes Involved:** `On form submission`  

**Node: On form submission**  
- **Type & Role:** Form Trigger node; entry point waiting for user to submit a URL.  
- **Configuration:**  
  - Form titled "URL Test" with one required field: "Landing Page" (expects a URL, e.g., https://example.com).  
  - No specific webhook ID set (uses default).  
- **Expressions/Variables:** User input is stored as `$json['Landing Page']`.  
- **Input/Output:** No input; output triggers downstream nodes with submitted data.  
- **Edge Cases:**  
  - User submits invalid or malformed URL (no validation beyond required field).  
  - No submission — workflow stays idle.  
- **Version:** 2.2  

---

#### 1.2 Data Gathering

- **Overview:** Fetches HTTP response data from the submitted URL and queries AlienVault OTX for threat intel on the domain.  
- **Nodes Involved:** `HTTP Request`, `AlienVault HTTP Request`  

**Node: HTTP Request**  
- **Type & Role:** HTTP Request node; fetches the landing page URL content and headers.  
- **Configuration:**  
  - URL dynamically set from form input: `={{ $json['Landing Page'] }}`.  
  - Redirects enabled to follow HTTP redirects.  
  - Response format set to "text" and full HTTP response captured (headers, status code, body).  
- **Input/Output:** Input from form submission; output HTTP response data to next node.  
- **Edge Cases:**  
  - Invalid URL or network errors (timeout, DNS failure).  
  - Server returns error codes or unexpected content types.  
- **Version:** 4.2  

**Node: AlienVault HTTP Request**  
- **Type & Role:** HTTP Request node; queries AlienVault OTX API for threat info on domain.  
- **Configuration:**  
  - URL: `https://otx.alienvault.com/api/v1/indicators/hostname/{{ domain }}` where domain extracted from landing page URL by stripping `https://` or `http://`.  
  - Method is POST but no body sent.  
  - Authentication uses predefined AlienVault API credential (`alienVaultApi`).  
  - No custom headers or SSL certificate setup.  
- **Input/Output:** Input from HTTP Request node; outputs AlienVault JSON data.  
- **Edge Cases:**  
  - AlienVault API failures (auth error, rate limiting, downtime).  
  - Malformed domain extraction if URL format unexpected.  
- **Version:** 4.2  

---

#### 1.3 Data Preparation

- **Overview:** Combines HTTP response and AlienVault data, checks for potential security issues, and packages all information for AI analysis.  
- **Nodes Involved:** `Prepare Data for AI`  

**Node: Prepare Data for AI**  
- **Type & Role:** Code node; consolidates inputs into a structured JSON object.  
- **Configuration:**  
  - Reads HTTP response JSON and AlienVault JSON inputs.  
  - Extracts URL, status code, security headers, AlienVault domain info, whois data, pulse counts, and threat sections.  
  - Adds heuristic checks for potential issues: missing security headers, HTTP errors (status >=400), presence of malware section, pulses count > 0.  
  - Adds timestamp and raw input copies for debugging.  
- **Expressions/Variables:** Uses `$input.first()`, `$input.all()` to access multi-input data streams.  
- **Input/Output:** Inputs from HTTP Request and AlienVault HTTP Request nodes; outputs a consolidated JSON for AI node.  
- **Edge Cases:**  
  - Missing or partial data from either source.  
  - Unexpected data structures causing JS errors.  
- **Version:** 2  

---

#### 1.4 AI Processing

- **Overview:** Uses a LangChain agent with OpenAI GPT-4 to analyze the consolidated data, identify vulnerabilities, and generate an expert security report.  
- **Nodes Involved:** `Security Configuration Audit`  

**Node: Security Configuration Audit**  
- **Type & Role:** LangChain Agent node; AI prompt executor specialized for cybersecurity analysis.  
- **Configuration:**  
  - System prompt defines role as cybersecurity expert analyzing website data and AlienVault threat intelligence.  
  - User prompt dynamically injects consolidated data fields (URL, status code, headers, potential issues, AlienVault info).  
  - Output is a concise report detailing issues, impacts, exploitation examples, and mitigations.  
  - Prompt type set to “define” (custom prompt).  
- **Credentials:** Uses OpenAI API credentials (`openAiApi`).  
- **Input/Output:** Input from `Prepare Data for AI`; output is AI-generated text report.  
- **Edge Cases:**  
  - OpenAI API errors (quota exhaustion, network issues).  
  - AI hallucinations or incomplete analysis.  
- **Version:** 1.7  

---

#### 1.5 Report Formatting

- **Overview:** Converts AI text output into styled HTML suitable for email delivery, adding sections and basic CSS.  
- **Nodes Involved:** `Format Report for Email`  

**Node: Format Report for Email**  
- **Type & Role:** Code node; formats AI plaintext report into HTML email content.  
- **Configuration:**  
  - Extracts AI output from input JSON.  
  - Wraps report in HTML structure with inline CSS for readability.  
  - Includes placeholders for raw data reference (currently static text).  
- **Input/Output:** Input from AI node; output HTML content JSON for email sending.  
- **Edge Cases:**  
  - Missing or empty AI output results in fallback message.  
- **Version:** 2  

---

#### 1.6 Reporting

- **Overview:** Sends the formatted security audit report via Gmail to an email address (configured dynamically or statically).  
- **Nodes Involved:** `Send Security Report`  

**Node: Send Security Report**  
- **Type & Role:** Gmail node; sends the email with the HTML report.  
- **Configuration:**  
  - Recipient email address dynamically set (currently empty string, to be configured).  
  - Subject line includes the tested URL for clarity.  
  - Email body set to HTML from previous node.  
  - Uses OAuth2 Gmail credentials (`gmailOAuth2`).  
- **Input/Output:** Input from `Format Report for Email`; output is email sent confirmation.  
- **Edge Cases:**  
  - Authentication failures or expired tokens.  
  - Missing or invalid recipient email.  
- **Version:** 2.1  

---

### 3. Summary Table

| Node Name               | Node Type                        | Functional Role                          | Input Node(s)          | Output Node(s)                | Sticky Note                     |
|-------------------------|---------------------------------|----------------------------------------|------------------------|------------------------------|--------------------------------|
| On form submission      | Form Trigger                    | Capture user-submitted URL              | -                      | HTTP Request                 |                                |
| HTTP Request            | HTTP Request                   | Fetch HTTP response from submitted URL | On form submission     | AlienVault HTTP Request      |                                |
| AlienVault HTTP Request | HTTP Request                   | Query AlienVault OTX for domain intel  | HTTP Request           | Prepare Data for AI          |                                |
| Prepare Data for AI     | Code                           | Consolidate and analyze input data      | AlienVault HTTP Request | Security Configuration Audit |                                |
| Security Configuration Audit | LangChain Agent             | AI-powered security analysis            | Prepare Data for AI     | Format Report for Email      |                                |
| Format Report for Email | Code                           | Convert AI output to HTML email content | Security Configuration Audit | Send Security Report        |                                |
| Send Security Report    | Gmail                          | Email the final security report          | Format Report for Email | -                            |                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node**  
   - Name: `On form submission`  
   - Type: Form Trigger  
   - Configure form: Title = "URL Test"  
   - Add one required field: Label = "Landing Page", Placeholder = "https://example.com"  
   - No webhook ID needed, default is fine.

2. **Create HTTP Request Node**  
   - Name: `HTTP Request`  
   - Type: HTTP Request  
   - Set URL to expression: `={{ $json["Landing Page"] }}`  
   - Enable redirects to follow HTTP redirects.  
   - Response format: Text, capture full response including headers and status code.

3. **Create AlienVault HTTP Request Node**  
   - Name: `AlienVault HTTP Request`  
   - Type: HTTP Request  
   - Method: POST  
   - URL expression:  
     ```
     = "https://otx.alienvault.com/api/v1/indicators/hostname/" + 
       $('On form submission').item.json['Landing Page'].replace("https://", "").replace("http://", "")
     ```  
   - Authentication: Use AlienVault API credential (OAuth2 or API key as configured).  
   - No body or additional headers.  
   - Connect input from `HTTP Request`.

4. **Create Code Node: Prepare Data for AI**  
   - Name: `Prepare Data for AI`  
   - Type: Code (JavaScript)  
   - Input: From `AlienVault HTTP Request` (multi-input with HTTP Request data and AlienVault data)  
   - Paste provided JavaScript code that:  
     - Extracts HTTP and AlienVault data  
     - Checks for potential issues (e.g., missing security headers, malware section)  
     - Builds consolidated JSON object with timestamp  
   - Output: Single JSON item for AI node.

5. **Create LangChain Agent Node: Security Configuration Audit**  
   - Name: `Security Configuration Audit`  
   - Type: LangChain Agent  
   - Credentials: OpenAI API key (configured under `openAiApi`)  
   - Prompt type: Define  
   - System prompt: cybersecurity expert role and instructions as given  
   - User prompt: Inject consolidated data fields from `Prepare Data for AI` node via expressions.

6. **Create Code Node: Format Report for Email**  
   - Name: `Format Report for Email`  
   - Type: Code (JavaScript)  
   - Input: Output from `Security Configuration Audit`  
   - Paste provided JS code that formats AI output into styled HTML content, including CSS and placeholders.

7. **Create Gmail Node: Send Security Report**  
   - Name: `Send Security Report`  
   - Type: Gmail  
   - Credentials: Gmail OAuth2 (configured as `gmailOAuth2`)  
   - Set recipient email (hard-coded or dynamic expression).  
   - Subject line expression: `"Website Security Audit - " + $('On form submission').item.json['Landing Page']`  
   - Body: Use `htmlReport` from `Format Report for Email`.

8. **Connect Nodes**  
   - Connect `On form submission` → `HTTP Request`  
   - Connect `HTTP Request` → `AlienVault HTTP Request`  
   - Connect both `HTTP Request` and `AlienVault HTTP Request` outputs into `Prepare Data for AI` (multi-input)  
   - Connect `Prepare Data for AI` → `Security Configuration Audit`  
   - Connect `Security Configuration Audit` → `Format Report for Email`  
   - Connect `Format Report for Email` → `Send Security Report`

---

### 5. General Notes & Resources

| Note Content                                                                                           | Context or Link                                  |
|------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| AlienVault OTX API requires valid API credentials; register at https://otx.alienvault.com/api/       | AlienVault API documentation                     |
| OpenAI API credentials needed for LangChain Agent node; supports GPT-4 and other models              | https://platform.openai.com/docs/api-reference  |
| Gmail OAuth2 credentials must be configured for sending emails; ensure token refresh and permissions | https://developers.google.com/gmail/api          |
| The AI prompt is designed as a cybersecurity expert system prompt, focused on vulnerabilities analysis | Custom LangChain prompt                           |
| Potential improvements: add URL validation on input, handle API failures with retries or alerts       | Workflow robustness notes                         |

---

**Disclaimer:**  
The provided text originates exclusively from an automated n8n workflow. It complies strictly with content policies and contains no illegal or offensive material. All processed data is legal and publicly available.