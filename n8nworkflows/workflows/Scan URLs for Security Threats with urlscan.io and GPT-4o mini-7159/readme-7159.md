Scan URLs for Security Threats with urlscan.io and GPT-4o mini

https://n8nworkflows.xyz/workflows/scan-urls-for-security-threats-with-urlscan-io-and-gpt-4o-mini-7159


# Scan URLs for Security Threats with urlscan.io and GPT-4o mini

### 1. Workflow Overview

This workflow, titled **"AI-Powered Link Checker"**, is designed to scan URLs for potential security threats by integrating urlscan.io’s scanning service with AI-powered analysis using OpenAI’s GPT-4o mini model. It accepts URLs via a webhook, submits them to urlscan.io for scanning, waits for scan results including screenshots, then leverages AI to evaluate the scan data and send a detailed security assessment via email.

**Target Use Cases:**  
- Automated URL threat detection and classification for security teams or automated monitoring systems.  
- Enriching URL scan data with AI-generated summaries and risk scoring.  
- Alerting stakeholders through email with actionable insights and scan artifacts.

**Logical Blocks:**  
- **1.1 Input Reception:** Receives incoming URL submissions via webhook.  
- **1.2 URL Scanning:** Submits URL to urlscan.io and waits for scan completion.  
- **1.3 AI Processing:** Sends urlscan.io scan results to GPT-4o mini for security classification and summarization.  
- **1.4 Notification:** Sends an email with scan details, AI analysis, and links to scan artifacts.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** Listens for incoming HTTP POST requests containing a URL to be scanned. Acts as the entry point to the workflow.  
- **Nodes Involved:**  
  - Webhook

- **Node Details:**  
  - **Webhook**  
    - *Type:* Webhook (HTTP POST endpoint)  
    - *Configuration:*  
      - Path: `url-scan`  
      - HTTP Method: POST  
      - Expects JSON body containing a `url` field (array with one URL string).  
    - *Input:* External HTTP POST request.  
    - *Output:* Passes the incoming JSON body containing the URL to the next node.  
    - *Edge Cases:*  
      - Missing or malformed URL in the request body.  
      - Invalid HTTP method or path.  
      - Potential webhook availability or permission issues.

#### 1.2 URL Scanning

- **Overview:** Submits the received URL to urlscan.io’s API to initiate a security scan, then waits for the scan to complete.  
- **Nodes Involved:**  
  - Perform a scan  
  - Wait

- **Node Details:**  
  - **Perform a scan**  
    - *Type:* urlScanIo node  
    - *Configuration:*  
      - URL to scan is extracted from webhook JSON body: `{{$json.body.url[0]}}`  
      - Uses urlscan.io API credentials (stored securely, no hard-coded keys).  
    - *Input:* URL from webhook node.  
    - *Output:* Scan submission response including scan ID (UUID).  
    - *Edge Cases:*  
      - API authentication failure.  
      - API rate limits or quota exceeded.  
      - Network timeout or connectivity issues.

  - **Wait**  
    - *Type:* Wait node  
    - *Configuration:* Waits for 30 seconds to allow urlscan.io to generate scan artifacts (e.g., screenshots).  
    - *Input:* Scan submission response from "Perform a scan".  
    - *Output:* Passes data to the AI processing node.  
    - *Edge Cases:*  
      - Insufficient wait time causing incomplete scan results.  
      - Workflow timeout if wait duration is too long.

#### 1.3 AI Processing

- **Overview:** Sends the urlscan.io JSON scan result to the GPT-4o mini model to classify the URL's threat level, assign a risk score, and generate a concise summary.  
- **Nodes Involved:**  
  - Message a model

- **Node Details:**  
  - **Message a model**  
    - *Type:* OpenAI node (via n8n’s LangChain integration)  
    - *Configuration:*  
      - Model used: `gpt-4.1-nano` (GPT-4o mini)  
      - Prompt instructs the model to:  
        1. Classify the scan as malicious, suspicious, or benign.  
        2. Assign a risk score from 1–10.  
        3. Provide a two-sentence explanation.  
        4. Output JSON with classification, riskScore, summary.  
      - The scan JSON is injected dynamically via expression referencing `{{$json.api}}` from the “Perform a scan” output.  
      - Credentials: OpenAI API key stored securely.  
    - *Input:* Scan JSON from urlscan.io after wait.  
    - *Output:* AI-generated JSON with classification and summary.  
    - *Edge Cases:*  
      - OpenAI API authentication or quota issues.  
      - Model response parsing errors if JSON output is malformed.  
      - Latency or timeout errors from API calls.

#### 1.4 Notification

- **Overview:** Sends an email notification containing the original URL, scan ID, classification, risk score, AI summary, and links to scan results and screenshots.  
- **Nodes Involved:**  
  - Send a message

- **Node Details:**  
  - **Send a message**  
    - *Type:* Gmail node (OAuth2 authenticated)  
    - *Configuration:*  
      - Recipient email address configured in node parameters (e.g., `test@gmail.com`).  
      - Subject includes the scanned URL.  
      - Email body is HTML formatted with:  
        - URL link  
        - Scan ID  
        - Classification and risk score from AI JSON output, parsed via expressions.  
        - AI summary text.  
        - Link to screenshot png on urlscan.io (constructed from scan ID).  
        - Link to full urlscan.io JSON result.  
      - Uses Gmail OAuth2 credentials securely stored.  
    - *Input:* AI JSON from "Message a model" plus previous scan data.  
    - *Output:* Email sent confirmation.  
    - *Edge Cases:*  
      - Email sending failure (OAuth token expired, quota exceeded).  
      - Missing or malformed fields causing broken links or empty placeholders.

---

### 3. Summary Table

| Node Name           | Node Type           | Functional Role              | Input Node(s)     | Output Node(s)     | Sticky Note                                                                                                                               |
|---------------------|---------------------|------------------------------|-------------------|--------------------|-------------------------------------------------------------------------------------------------------------------------------------------|
| Sticky Note — How it works | Sticky Note         | Documentation of workflow logic | None              | None               | Explains workflow steps, input example, and output artifacts.                                                                              |
| Sticky Note — Setup  | Sticky Note         | Setup and credential instructions | None              | None               | Details about credential setup, webhook paths, wait time advice, and notes on secure key storage.                                         |
| Webhook             | Webhook             | Receives URL input via HTTP POST | None              | Perform a scan     |                                                                                                                                           |
| Perform a scan      | urlScanIo           | Submits URL to urlscan.io API  | Webhook           | Wait               |                                                                                                                                           |
| Wait                | Wait                | Pauses workflow to allow scan completion | Perform a scan    | Message a model     |                                                                                                                                           |
| Message a model     | OpenAI (LangChain)  | AI analyzes scan results and classifies threat | Wait              | Send a message      |                                                                                                                                           |
| Send a message      | Gmail               | Sends email with scan and AI analysis results | Message a model   | None               |                                                                                                                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `url-scan`  
   - Purpose: Receive JSON payload with a URL array under `body.url`.  

2. **Add urlScanIo Node ("Perform a scan")**  
   - Type: urlScanIo  
   - Credentials: Connect with your urlscan.io API Key credential.  
   - URL parameter: Set to expression `{{$json.body.url[0]}}` to extract URL from webhook JSON.  
   - Connect output of Webhook node to this node.

3. **Add Wait Node ("Wait")**  
   - Type: Wait  
   - Parameters: Set wait time to 30 seconds (adjustable if scan results take longer).  
   - Connect output of "Perform a scan" to this node.

4. **Add OpenAI Node ("Message a model")**  
   - Type: OpenAI via LangChain (OpenAI)  
   - Credentials: Use your OpenAI API Key credential.  
   - Model ID: Select `gpt-4.1-nano` (GPT-4o mini).  
   - Messages: Add a system/user prompt with instructions:  
     ```
     You are a security analyst. 
     Given the following urlscan.io JSON result, do four things:
     1. Decide if the scan is malicious, suspicious or benign.
     2. Assign a risk score from 1-10 (10 = confirmed malicious).
     3. In two short sentences explain why.
     4. Output EXACTLY this JSON:
        {
          "classification": "<malicious|suspicious|benign>",
          "riskScore": <integer>,
          "summary": "<two-sentence explanation>"
        }
    
     Scan result:
     {{ $json.api }}
     ```  
   - Ensure the input JSON (`$json.api`) correctly references output from "Perform a scan".  
   - Connect output of Wait node to this node.

5. **Add Gmail Node ("Send a message")**  
   - Type: Gmail (OAuth2)  
   - Credentials: Connect with your Gmail OAuth2 credential.  
   - To: Set email recipient address (e.g., `test@gmail.com`).  
   - Subject: Use expression `=URLScan Submitted for {{ $json.url || $item(0).$node["Webhook"].json.body.url?.[0] }}` to dynamically include the scanned URL.  
   - Message (HTML): Construct an HTML email embedding:  
     - The submitted URL as a clickable link.  
     - Scan ID from "Perform a scan" node.  
     - Classification and risk score parsed from JSON output of "Message a model".  
     - AI summary text from the model output.  
     - Links to screenshot and full scan result on urlscan.io (using scan ID).  
   - Connect output of "Message a model" to this node.

6. **Set Execution Order:**  
   Webhook → Perform a scan → Wait → Message a model → Send a message

7. **Test the Workflow:**  
   - POST to your webhook URL with JSON body like:  
     ```json
     { "url": ["https://example.com"] }
     ```  
   - Verify email receipt with correct scan results and AI summary.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                               | Context or Link                                  |
|------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| Workflow uses Markdown in Sticky Notes for clear documentation.                                                                                             | n8n Sticky Note formatting                       |
| Wait node default time is 30 seconds; increase if scan screenshots or artifacts are not ready in time.                                                    | Workflow Setup                                   |
| urlscan.io API Key, OpenAI API Key, and Gmail OAuth2 credentials must be securely stored and linked in n8n credentials. No hard-coded API keys allowed. | Security best practices                          |
| The AI prompt expects strict JSON output from the model for parsing; malformed responses can cause workflow errors.                                       | AI integration robustness                        |
| Webhook test URL example: `/webhook-test/urlscan` vs production `/webhook/urlscan`.                                                                        | n8n webhook endpoint distinctions                |
| For more on urlscan.io API capabilities, refer to https://urlscan.io/about-api/.                                                                           | urlscan.io official API documentation            |
| For OpenAI API details, see https://platform.openai.com/docs/api-reference.                                                                                | OpenAI API documentation                          |
| Gmail OAuth2 setup instructions for n8n: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.gmail/                                        | Gmail credential configuration guide              |

---

**Disclaimer:** The text provided is exclusively generated from an automated n8n workflow. It complies with all content policies and contains no illegal or offensive material. All processed data is legal and publicly accessible.