ScriptSentry: Detect Sensitive Information in JavaScript

https://n8nworkflows.xyz/workflows/scriptsentry--detect-sensitive-information-in-javascript-5497


# ScriptSentry: Detect Sensitive Information in JavaScript

### 1. Workflow Overview

**Purpose:**  
"ScriptSentry: Detect Sensitive Information in JavaScript" is an automated security assessment workflow designed to scan a user-specified website for JavaScript files that may contain sensitive information such as API keys, email addresses, and personally identifiable information (PII). It extracts JavaScript URLs from the landing page, analyzes their content using AI, summarizes findings, and sends a formatted email report to designated recipients.

**Target Use Cases:**  
- Ethical hackers or security analysts performing automated scans for exposed secrets in JavaScript files of client websites.  
- Website owners or developers wanting a quick check of their public-facing JavaScript for accidental credential exposure or PII leaks.  
- Security operations teams integrating AI-driven static resource inspection into their vulnerability management workflows.

**Logical Blocks:**

- **1.1 Input Reception**  
  Captures the target website URL via a form interface.

- **1.2 JavaScript Retrieval and Extraction**  
  Uses Puppeteer to load the target page and extracts JavaScript file URLs from its HTML content.

- **1.3 Data Aggregation and Preparation**  
  Aggregates extracted URLs into a unified data structure and prepares it for AI analysis.

- **1.4 AI-Powered Sensitive Data Analysis & Email Composition**  
  Invokes a Langchain agent powered by OpenAI to analyze JavaScript content for sensitive data and generate a detailed email report.

- **1.5 Email Formatting and Sending**  
  Formats the AI output into an HTML report and sends it via Gmail.

- **Supplementary:** Sticky notes provide contextual instructions, prerequisites, and usage guidance throughout the workflow.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  Collects the target landing page URL from the user via a web form, triggering the workflow execution.

- **Nodes Involved:**  
  - Landing Page Url1

- **Node Details:**  
  - **Landing Page Url1**  
    - *Type:* Form Trigger  
    - *Configuration:*  
      - Form titled "Website Security Scanner" with a required field labeled "Landing Page Url" (placeholder: https://example.com)  
      - Description prompts user to submit a URL for security assessment  
    - *Input/Output:* Entry point; outputs form data JSON with the URL string  
    - *Edge Cases:*  
      - Missing or malformed URL input may cause downstream Puppeteer failure  
      - Requires user to access and submit the form via the provided webhook URL  
    - *Version:* 2.2

- **Sticky Notes:**  
  - "Target URL" explains how to execute and use the form trigger URL.

---

#### 1.2 JavaScript Retrieval and Extraction

- **Overview:**  
  Loads the specified webpage using Puppeteer headless browser, then extracts all JavaScript `<script>` URLs from the HTML content.

- **Nodes Involved:**  
  - Puppeteer1  
  - JavaScript Extractor1

- **Node Details:**  
  - **Puppeteer1**  
    - *Type:* Puppeteer node  
    - *Configuration:*  
      - URL dynamically set to the value from "Landing Page Url1" form input  
      - Waits until network is idle (`networkidle2`) to ensure page fully loads  
    - *Input:* Receives URL from Landing Page Url1  
    - *Output:* Full HTML body of the landing page  
    - *Edge Cases:*  
      - Page load timeout or errors if URL is unreachable  
      - Requires Puppeteer community node installed and configured  
    - *Version:* 1

  - **JavaScript Extractor1**  
    - *Type:* Code node (JavaScript)  
    - *Configuration:*  
      - Custom JavaScript code scans the HTML body for `<script>` tags with `src` attributes  
      - Extracts all JavaScript file URLs into output array objects (`{ json: { URL } }`)  
    - *Input:* HTML body from Puppeteer1  
    - *Output:* List of JavaScript URLs found on the page  
    - *Edge Cases:*  
      - May miss scripts loaded dynamically after initial page load  
      - Only extracts scripts with explicit `src` attributes (ignores inline scripts)  
    - *Version:* 2

- **Sticky Notes:**  
  - "Puppeteer" node installation instructions  
  - "JavaScript Crawler" indicating this block’s role

---

#### 1.3 Data Aggregation and Preparation

- **Overview:**  
  Consolidates all extracted JavaScript URLs into a single aggregated data array and prepares it for AI analysis by mapping data into the expected input format.

- **Nodes Involved:**  
  - Aggregate1  
  - Data Mapper1

- **Node Details:**  
  - **Aggregate1**  
    - *Type:* Aggregate node  
    - *Configuration:*  
      - Aggregates all incoming items into one unified data object  
    - *Input:* Multiple JavaScript URL objects from extractor  
    - *Output:* Single aggregated JSON containing an array of URLs in `data` field  
    - *Edge Cases:*  
      - Empty input results in empty array, handled downstream  
    - *Version:* 1

  - **Data Mapper1**  
    - *Type:* Set node  
    - *Configuration:*  
      - Assigns the aggregated data array from Aggregate1 to `data[0]` property for compatibility with Langchain agent input  
    - *Input:* Aggregated data from Aggregate1  
    - *Output:* JSON containing `data[0]` with array of URLs  
    - *Edge Cases:*  
      - Misconfiguration could cause empty or invalid input to AI agent  
    - *Version:* 3.4

---

#### 1.4 AI-Powered Sensitive Data Analysis & Email Composition

- **Overview:**  
  Uses a Langchain agent with OpenAI GPT-4 model to analyze JavaScript files for sensitive info and dynamically generates a professional email report summarizing findings and recommendations.

- **Nodes Involved:**  
  - OpenAI Chat Model  
  - JavaScript Search Agent w/Email Template1

- **Node Details:**  
  - **OpenAI Chat Model**  
    - *Type:* Langchain OpenAI chat language model node  
    - *Configuration:*  
      - Model: `"gpt-4.1-mini"` (GPT-4 variant with smaller footprint)  
      - No additional options set  
    - *Input:* Receives prompt text from agent node (via Langchain integration)  
    - *Output:* AI-generated text response used by agent node  
    - *Credentials:* OpenAI API key required  
    - *Edge Cases:*  
      - API quota or authentication errors possible  
      - Model version changes may affect output format or quality  
    - *Version:* 1.2

  - **JavaScript Search Agent w/Email Template1**  
    - *Type:* Langchain agent node  
    - *Configuration:*  
      - Complex prompt instructing the AI to:  
        - Analyze JavaScript contents filtered to relevant URLs from the target domain  
        - Detect sensitive info types (API keys, emails, PII) without exposing full content  
        - Generate a numbered list with findings and remediation advice  
        - Produce a plain text email addressed to "User" signed by "Admin"  
        - Include URL links extracted via Aggregate1  
      - Uses dynamic expressions to pull landing page URL and aggregated JavaScript URLs  
      - Output is plain text, formatted as an email body  
    - *Input:* Prepared JavaScript URLs and contents from Data Mapper1; AI model connection via OpenAI Chat Model  
    - *Output:* AI-generated email report text  
    - *Edge Cases:*  
      - Large input data may hit token limits  
      - Expression errors if referenced nodes’ data is missing or malformed  
      - AI hallucination risk — results should be validated before production use  
    - *Version:* 2

- **Sticky Notes:**  
  - "JavaScript Scan and Email Template" describing this block’s function

---

#### 1.5 Email Formatting and Sending

- **Overview:**  
  Wraps the AI-generated raw text into a simple HTML report, then sends it via Gmail to a configured recipient.

- **Nodes Involved:**  
  - Format Report for Email1  
  - Send a message1

- **Node Details:**  
  - **Format Report for Email1**  
    - *Type:* Code node (JavaScript)  
    - *Configuration:*  
      - Wraps the AI text output (`output` field) in minimal HTML with a heading and preformatted block for readability  
      - Defaults to "No AI report." if no data present  
    - *Input:* AI email text from JavaScript Search Agent node  
    - *Output:* JSON containing `htmlReport` field with HTML string  
    - *Edge Cases:*  
      - Empty or malformed input leads to minimal report  
    - *Version:* 2

  - **Send a message1**  
    - *Type:* Gmail node  
    - *Configuration:*  
      - Sends to hard-coded email `admin@example.com` (should be customized)  
      - Message body contains HTML report from previous node  
      - Fixed subject line (editable in node parameters)  
      - Sender name set to "n8n"  
    - *Credentials:* OAuth2 Gmail account required  
    - *Edge Cases:*  
      - Authentication failures if OAuth token expired or misconfigured  
      - Email delivery failures or spam filtering possible  
    - *Version:* 2.1

- **Sticky Notes:**  
  - "Gmail Instructions" indicating need for OAuth setup and recipient email configuration

---

### 3. Summary Table

| Node Name                           | Node Type                    | Functional Role                                    | Input Node(s)         | Output Node(s)                | Sticky Note                                               |
|-----------------------------------|------------------------------|---------------------------------------------------|-----------------------|------------------------------|-----------------------------------------------------------|
| Landing Page Url1                  | Form Trigger                 | Receive target website URL from user               | —                     | Puppeteer1                   | ## Target URL: Click Execute workflow and visit form URL  |
| Puppeteer1                        | Puppeteer                   | Load webpage and get HTML content                   | Landing Page Url1      | JavaScript Extractor1        | ## Puppeteer: Be sure to have Puppeteer installed         |
| JavaScript Extractor1             | Code (JavaScript)            | Extract JavaScript file URLs from HTML              | Puppeteer1             | Aggregate1                   | ## JavaScript Crawler                                      |
| Aggregate1                       | Aggregate                   | Aggregate extracted URLs into a single array        | JavaScript Extractor1  | Data Mapper1                 |                                                           |
| Data Mapper1                    | Set                         | Prepare aggregated data for AI input                 | Aggregate1             | JavaScript Search Agent w/Email Template1 |                                                   |
| JavaScript Search Agent w/Email Template1 | Langchain Agent (AI)         | Analyze JS files for sensitive info and create email| Data Mapper1, OpenAI Chat Model | Format Report for Email1      | ## JavaScript Scan and Email Template                      |
| OpenAI Chat Model                 | Langchain OpenAI Chat Model | Provides AI language model for agent                  | —                     | JavaScript Search Agent w/Email Template1 |                                                   |
| Format Report for Email1          | Code (JavaScript)            | Format AI text output into HTML report               | JavaScript Search Agent w/Email Template1 | Send a message1              |                                                           |
| Send a message1                  | Gmail                       | Send formatted email report via Gmail                | Format Report for Email1 | —                            | ## Gmail Instructions: Setup OAuth and recipient email    |
| Sticky Note                      | Sticky Note                 | Instructional or contextual notes                     | —                     | —                            | See individual sticky note contents above                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger node**  
   - Name: `Landing Page Url1`  
   - Type: `Form Trigger`  
   - Configure form title: "Website Security Scanner"  
   - Add a single required field: "Landing Page Url" (placeholder: https://example.com)  
   - Save and note the webhook URL for form access.

2. **Add Puppeteer node**  
   - Name: `Puppeteer1`  
   - Type: `Puppeteer` (community node, ensure installed)  
   - Set URL parameter: `={{ $json["Landing Page Url"] }}` (dynamic from form input)  
   - Set options: Wait for `networkidle2` event to ensure full page load.

3. **Add Code node for JS extraction**  
   - Name: `JavaScript Extractor1`  
   - Type: `Code` (JavaScript)  
   - Paste code to extract `<script src="...">` URLs from Puppeteer HTML body:
     ```javascript
     const output = [];
     const html = $input.first()?.json?.body || '';
     const regex = /<script[^>]*src="([^"]+)"[^>]*>/g;
     let match;
     while ((match = regex.exec(html)) !== null) {
       output.push({ json: { URL: match[1] } });
     }
     return output;
     ```
   - Connect Puppeteer1 output to this node.

4. **Add Aggregate node**  
   - Name: `Aggregate1`  
   - Type: `Aggregate`  
   - Configure to aggregate all incoming items into one array.  
   - Connect `JavaScript Extractor1` output to this node.

5. **Add Set node to prepare data**  
   - Name: `Data Mapper1`  
   - Type: `Set`  
   - Assign field `data[0]` with value `={{ $json.data[0] }}` (aggregated URLs)  
   - Connect `Aggregate1` output to this node.

6. **Add Langchain OpenAI Chat Model**  
   - Name: `OpenAI Chat Model`  
   - Type: `Langchain OpenAI Chat Model`  
   - Set model: `gpt-4.1-mini`  
   - Configure OpenAI API credentials.

7. **Add Langchain Agent node for analysis and email generation**  
   - Name: `JavaScript Search Agent w/Email Template1`  
   - Type: `Langchain Agent`  
   - Set the prompt with instructions to analyze JavaScript files for sensitive info and format an email report (use the detailed prompt from the original workflow).  
   - Connect `Data Mapper1` output to this node’s main input.  
   - Connect `OpenAI Chat Model` node to the agent’s AI language model input.

8. **Add Code node to format AI output as HTML**  
   - Name: `Format Report for Email1`  
   - Type: `Code` (JavaScript)  
   - Use code to wrap text output in simple HTML template:
     ```javascript
     const aiReport = $json.output || 'No AI report.';
     return [{
       json: {
         htmlReport: `<html><body><h1>AI Report</h1><pre>${aiReport}</pre></body></html>`
       }
     }];
     ```
   - Connect the agent node output to this node.

9. **Add Gmail node to send the email**  
   - Name: `Send a message1`  
   - Type: `Gmail`  
   - Configure:  
     - Recipient: Set to desired email address (default: admin@example.com)  
     - Subject: Static or dynamic subject line  
     - Message: Use expression `={{ $json.htmlReport }}` from previous node  
     - Sender name: "n8n" or custom  
   - Set up Gmail OAuth2 credentials properly in n8n.  
   - Connect `Format Report for Email1` output to this node.

10. **Connect nodes in order:**  
    `Landing Page Url1` → `Puppeteer1` → `JavaScript Extractor1` → `Aggregate1` → `Data Mapper1` → `JavaScript Search Agent w/Email Template1` → `Format Report for Email1` → `Send a message1`  
    Also connect `OpenAI Chat Model` to the agent node’s AI language model input.

11. **Test the workflow:**  
    - Execute the workflow and visit the form trigger URL.  
    - Submit a valid target landing page URL.  
    - Monitor execution, verify email receipt with the AI-generated security report.

---

### 5. General Notes & Resources

| Note Content                                                                                               | Context or Link                                                      |
|------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------|
| Puppeteer community node must be installed via Settings → Community Nodes in n8n for Puppeteer functionality | See official n8n docs or community forum for installation details   |
| Gmail node requires OAuth2 credentials to send emails securely and reliably                                 | Use Google Cloud Console to create OAuth consent and credentials    |
| The AI prompt uses Langchain agent integration with OpenAI GPT-4 model                                    | Refer to n8n Langchain integration docs for advanced usage          |
| The workflow is designed for ethical security testing only; ensure authorized use to avoid legal issues   | Ethical guidelines and legal compliance                               |
| Sticky notes contain important operational instructions and must be reviewed before running the workflow   | Sticky notes visible in the workflow editor UI                       |

---

This completes a detailed, structured reference for the "ScriptSentry: Detect Sensitive Information in JavaScript" workflow, enabling understanding, reproduction, and modification.