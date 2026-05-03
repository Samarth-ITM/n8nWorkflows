Triage AWS Security Misconfigurations with GPT-4.1 Mini and Send Alerts to Gmail

https://n8nworkflows.xyz/workflows/triage-aws-security-misconfigurations-with-gpt-4-1-mini-and-send-alerts-to-gmail-7202


# Triage AWS Security Misconfigurations with GPT-4.1 Mini and Send Alerts to Gmail

### 1. Workflow Overview

This workflow automates the triage of AWS security misconfigurations detected by AWS Security Hub or AWS Config, using AI-powered prioritization and alerting via Gmail. It is designed to receive AWS Security Hub findings or Config notifications via webhook, normalize the data, apply AI analysis to prioritize and recommend remediation steps, log the triaged findings into Airtable, and send alert emails.

The workflow is logically divided into the following functional blocks:

- **1.1 Input Reception & Validation**: Receives incoming AWS event notifications via webhook and validates AWS SNS subscription tokens.
- **1.2 Event Normalization**: Parses and normalizes diverse AWS Security Hub or Config finding formats into a consistent JSON schema.
- **1.3 AI Prioritization & Triage**: Sends normalized findings to an OpenAI GPT-4.1 Mini model to obtain prioritized severity, rationale, remediation steps, and tags.
- **1.4 Persistence in Airtable**: Creates or updates a record in Airtable with the enriched finding data.
- **1.5 Alerting via Gmail**: Sends a formatted alert email to a configured Gmail address with details and AI recommendations.
- **1.6 Response Finalization**: Prepares and returns a response summarizing processing status and priority.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Validation

- **Overview:**  
  This block receives incoming AWS Security Hub or Config events via an HTTP POST webhook endpoint, processes AWS SNS subscription confirmation if needed, and performs token-based authorization.

- **Nodes Involved:**  
  - Webhook  
  - SNS Handler (Code)  
  - If (Conditional)  
  - SNS Confirm (HTTP Request)  
  - Edit Fields1 (Set)

- **Node Details:**

  - **Webhook**  
    - Type: HTTP Webhook  
    - Role: Entry point for incoming AWS Security Hub or AWS Config event notifications sent via HTTP POST at path `/aws-misconfig`.  
    - Config: POST method, response returned from the last node executed.  
    - Inputs: External HTTP POST requests.  
    - Outputs: Passes event JSON to SNS Handler.  
    - Edge cases: Missing or invalid POST data, unexpected HTTP methods.

  - **SNS Handler**  
    - Type: Code (JavaScript)  
    - Role: Handles AWS SNS message validation including subscription confirmation and notification processing.  
    - Config: Extracts body or raw JSON; validates a secret query or payload token (`MY_SUPER_TOKEN`); confirms SNS subscription if requested; parses nested SNS Message JSON.  
    - Key expressions: Validates `token === 'MY_SUPER_TOKEN'` for authorization; distinguishes `SubscriptionConfirmation` from `Notification` message types.  
    - Inputs: JSON from Webhook.  
    - Outputs: Object with mode (`confirm` or `notify`) and event data.  
    - Edge cases: Unauthorized token triggers error; malformed JSON in SNS Message; missing SubscribeURL.

  - **If**  
    - Type: Conditional  
    - Role: Routes flow based on SNS Handler output mode (`confirm` or `notify`).  
    - Config: Checks if mode equals 'confirm'.  
    - Inputs: Output from SNS Handler.  
    - Outputs: True branch → SNS Confirm; False branch → Normalize Finding.  
    - Edge cases: Unexpected mode values.

  - **SNS Confirm**  
    - Type: HTTP Request  
    - Role: Sends HTTP GET request to SNS subscription confirmation URL to finalize subscription.  
    - Config: URL dynamically set from SNS Handler output; 15s timeout; expects JSON response.  
    - Inputs: True branch from If node.  
    - Outputs: Confirmation response to Edit Fields1.  
    - Edge cases: Network timeouts; invalid SubscribeURL; HTTP errors.

  - **Edit Fields1**  
    - Type: Set  
    - Role: Formats a JSON response indicating successful subscription confirmation, including HTTP status code.  
    - Config: Creates an object `{ resp: { status: "subscribed", statusCode: <statusCode> } }`.  
    - Inputs: Response from SNS Confirm.  
    - Outputs: Terminates flow on subscription confirmation path.  
    - Edge cases: Missing statusCode in HTTP response.

---

#### 2.2 Event Normalization

- **Overview:**  
  Normalizes different AWS Security Hub and AWS Config finding formats into a unified JSON schema for downstream AI processing.

- **Nodes Involved:**  
  - Normalize Finding (Code)

- **Node Details:**

  - **Normalize Finding**  
    - Type: Code (JavaScript)  
    - Role: Extracts key finding attributes from various AWS event formats, including Security Hub EventBridge, raw Security Hub findings, and Config notifications.  
    - Config: Runs once per item; logic includes detection of finding source; extracts severity, title, description, resource ID, service, account, region, and hints for common misconfig types (e.g., S3, IAM, SG, RDS).  
    - Key expressions: Uses optional chaining and regex matching to infer AWS service and misconfiguration hints; returns a single normalized object.  
    - Inputs: AWS event JSON from SNS Handler (notify path).  
    - Outputs: Normalized finding object for AI Prioritizer.  
    - Edge cases: Missing expected fields; unrecognized event formats; fallback default values.

---

#### 2.3 AI Prioritization & Triage

- **Overview:**  
  Uses an OpenAI GPT-4.1 Mini language model to analyze normalized finding data, assign priority levels, provide rationale, recommend remediation steps, and generate tags.

- **Nodes Involved:**  
  - AI Prioritizer (OpenAI node)

- **Node Details:**

  - **AI Prioritizer**  
    - Type: OpenAI (LangChain)  
    - Role: Passes normalized finding JSON to GPT-4.1 Mini model with a system prompt describing triage guidelines and expected JSON output format.  
    - Config: Uses model `gpt-4.1-mini`; sends a single message with detailed instructions and embedded normalized finding JSON; expects JSON output parsed automatically by the node.  
    - Key expressions: The prompt instructs strict JSON return with fields: priority (P0-P3), rationale (text), remediation (array of steps), and tags (array).  
    - Inputs: Normalized finding JSON from Normalize Finding.  
    - Outputs: AI triage JSON for Airtable creation.  
    - Edge cases: Model response parse errors; API rate limits; malformed output JSON.

---

#### 2.4 Persistence in Airtable

- **Overview:**  
  Stores or updates the enriched finding records in an Airtable base for tracking and audit.

- **Nodes Involved:**  
  - Airtable - Create Record

- **Node Details:**

  - **Airtable - Create Record**  
    - Type: Airtable  
    - Role: Upserts (creates or updates) a record in a specified Airtable base and table with detailed finding and AI triage data.  
    - Config: Uses Airtable base and table IDs; matching column is `id` derived from normalized finding ID; maps multiple fields including Title, Severity, Priority, Resource, Account, Region, Tags, Rationale, and Remediation steps from AI output and normalized finding.  
    - Key expressions: Uses expressions to access multiple remediation steps and tags; concatenates tags; handles missing data gracefully.  
    - Inputs: AI triage JSON from AI Prioritizer.  
    - Outputs: Airtable record metadata (including record ID) for email.  
    - Edge cases: Airtable API authentication errors; field mapping mismatches; rate limits.

---

#### 2.5 Alerting via Gmail

- **Overview:**  
  Sends a richly formatted alert email to a configured Gmail address containing the finding details, AI prioritization, remediation guidance, tags, and raw JSON data.

- **Nodes Involved:**  
  - Send a message (Gmail)

- **Node Details:**

  - **Send a message**  
    - Type: Gmail (OAuth2)  
    - Role: Sends an email to a hardcoded recipient (`test@gmail.com`) with subject indicating priority, title, resource, and service.  
    - Config: Email body dynamically composed as HTML using expressions that combine normalized finding data, AI triage output, and Airtable record ID; includes headings, paragraphs, ordered remediation steps, tags, and a collapsible raw JSON block.  
    - Key expressions: Parses AI response content JSON safely; constructs HTML with inline styles and fallback values.  
    - Inputs: Airtable record info and AI triage data.  
    - Outputs: Email send result to Edit Fields.  
    - Edge cases: Gmail OAuth token expiry; email send quota; invalid recipient.

---

#### 2.6 Response Finalization

- **Overview:**  
  Prepares the final response JSON to return to the original webhook caller indicating processing status and triage priority.

- **Nodes Involved:**  
  - Edit Fields (Set)

- **Node Details:**

  - **Edit Fields**  
    - Type: Set  
    - Role: Constructs a JSON object containing processing status, priority from Airtable record, and finding ID for response.  
    - Config: Uses raw JSON mode; accesses priority from Airtable node and finding_id from normalization node.  
    - Inputs: Email send node output.  
    - Outputs: Final response returned by webhook.  
    - Edge cases: Missing priority or finding_id if prior nodes fail.

---

### 3. Summary Table

| Node Name             | Node Type               | Functional Role                      | Input Node(s)           | Output Node(s)             | Sticky Note                                                                                                                 |
|-----------------------|-------------------------|------------------------------------|------------------------|----------------------------|-----------------------------------------------------------------------------------------------------------------------------|
| Webhook               | HTTP Webhook            | Entry point for AWS event POST     | External HTTP POST      | SNS Handler                | You must have the AWS Side pre-configured before testing / starting this workflow                                            |
| SNS Handler           | Code                    | Handles SNS subscription & messages| Webhook                | If                         |                                                                                                                             |
| If                    | Conditional             | Routes based on SNS mode           | SNS Handler            | SNS Confirm (true), Normalize Finding (false) |                                                                                                                             |
| SNS Confirm           | HTTP Request            | Confirms SNS subscription          | If (true branch)       | Edit Fields1               |                                                                                                                             |
| Edit Fields1          | Set                     | Formats subscription confirmation  | SNS Confirm            | (ends)                     |                                                                                                                             |
| Normalize Finding     | Code                    | Normalizes diverse AWS findings    | If (false branch)      | AI Prioritizer             |                                                                                                                             |
| AI Prioritizer        | OpenAI (LangChain)      | AI triage and prioritization       | Normalize Finding      | Airtable - Create Record   |                                                                                                                             |
| Airtable - Create Record | Airtable              | Persists triaged findings          | AI Prioritizer         | Send a message             |                                                                                                                             |
| Send a message        | Gmail                   | Sends alert email                  | Airtable - Create Record| Edit Fields                |                                                                                                                             |
| Edit Fields           | Set                     | Final response formatting           | Send a message         | (ends, response to webhook)|                                                                                                                             |
| Sticky Note           | Sticky Note             | Informational note                  |                        |                            | You must have the AWS Side pre-configured before testing / starting this workflow                                            |
| Sticky Note1          | Sticky Note             | Workflow wiring and purpose summary|                        |                            | Flow: Webhook → SNS Handler → IF → (true) SNS Confirm → Done / (false) Normalize → AI → Airtable → Gmail → Respond. Purpose: triage AWS misconfigs and alert the team. Responds when last node finishes with JSON.|

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node:**
   - Type: HTTP Webhook  
   - Set HTTP Method to POST  
   - Set Path to `aws-misconfig`  
   - Leave response mode as `last node`  
   - No authentication; open to receive JSON payloads.

2. **Create SNS Handler Node (Code):**
   - Type: Code  
   - Mode: Run once per execution  
   - Paste the JavaScript code that:  
     - Extracts token from query or payload and compares to `MY_SUPER_TOKEN`  
     - Throws error if unauthorized  
     - Handles SNS `SubscriptionConfirmation` (returns mode and SubscribeURL)  
     - Parses nested SNS Message JSON if present  
     - Returns object `{ mode: 'confirm' | 'notify', event: <event data> }`

3. **Create If Node:**
   - Condition: Check if `{{$json.mode}}` equals `"confirm"` (strict equality)  
   - True branch leads to SNS Confirm; False branch to Normalize Finding

4. **Create SNS Confirm Node (HTTP Request):**
   - Type: HTTP Request  
   - Method: GET  
   - URL: `{{$json.subscribeUrl}}` (expression from input JSON)  
   - Timeout: 15000 ms (15 seconds)  
   - Expect JSON response (fullResponse)

5. **Create Edit Fields1 Node (Set):**
   - Type: Set  
   - Mode: Raw JSON  
   - Output JSON:  
     ```json
     {
       "resp": {
         "status": "subscribed",
         "statusCode": {{$json.statusCode || 200}}
       }
     }
     ```
   - Connect SNS Confirm output to this node. This ends the subscription path.

6. **Create Normalize Finding Node (Code):**
   - Type: Code  
   - Execute once per item  
   - Paste JavaScript code that normalizes AWS Security Hub and AWS Config findings into:  
     - finding_id, title, description, severity, resource_id, service, account, region, product_types, misconfig_hints, raw  
   - Connect False output of If node here.

7. **Create AI Prioritizer Node (OpenAI LangChain):**
   - Provider: OpenAI  
   - Model: `gpt-4.1-mini`  
   - Parameters:  
     - Messages: System prompt instructing AI to analyze JSON normalized finding and return strict JSON with fields: priority (P0-P3), rationale, remediation steps, tags  
     - Input: Pass normalized finding JSON stringified inside the prompt  
   - Enable JSON output parsing  
   - Connect Normalize Finding output here.

8. **Create Airtable - Create Record Node:**
   - Provider: Airtable  
   - Operation: Upsert record  
   - Base and Table: Configure with your Airtable base and table IDs  
   - Matching Column: `id` equal to normalized finding ID  
   - Map columns:  
     - `id` → normalized finding ID  
     - `Title`, `Severity`, `Priority`, `Resource`, `Service`, `Account`, `Region`, `Tags`, `Rationale`, `Remediation` → from AI triage JSON and normalized finding  
   - Connect AI Prioritizer output here.

9. **Create Send a message Node (Gmail):**
   - Provider: Gmail OAuth2  
   - Recipient: `test@gmail.com` (replace with your alert destination)  
   - Subject: Construct dynamically with priority, title, resource, service from normalized finding and AI output  
   - Message Body: Compose HTML using expressions combining normalized finding, AI triage data, and Airtable record ID; include remediation steps as ordered list, tags, rationale, and collapsible raw JSON.  
   - Connect Airtable output here.

10. **Create Edit Fields Node (Set):**
    - Type: Set  
    - Mode: Raw JSON  
    - Output JSON:  
      ```json
      {
        "resp": {
          "status": "processed",
          "priority": {{$node["Airtable - Create Record"].json.fields.Priority}},
          "finding_id": {{$node["Normalize Finding"].json.finding_id}}
        }
      }
      ```
    - Connect Send a message output here; this node returns the final response for the webhook.

11. **Connect the nodes as follows:**
    - Webhook → SNS Handler → If  
    - If (true) → SNS Confirm → Edit Fields1 (ends)  
    - If (false) → Normalize Finding → AI Prioritizer → Airtable - Create Record → Send a message → Edit Fields

12. **Credentials Setup:**
    - Gmail OAuth2: Configure OAuth2 credentials with Gmail API access.  
    - OpenAI API: Configure with your OpenAI API key.  
    - Airtable Access Token: Configure with a valid Airtable API token with write access to the target base.

13. **Set environment and security notes:**
    - Replace `MY_SUPER_TOKEN` in SNS Handler code with your own secret token used to authenticate incoming SNS messages.  
    - Before testing, ensure AWS Security Hub or Config is configured to send findings via SNS to your webhook URL and that SNS subscription confirmation is completed.

---

### 5. General Notes & Resources

| Note Content                                                                                             | Context or Link                                                                                   |
|--------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| You must have the AWS Side pre-configured before testing / starting this workflow                       | Sticky Note attached near Webhook node                                                          |
| Workflow flow summary: Webhook → SNS Handler → IF → (true) SNS Confirm → Done / (false) Normalize → AI → Airtable → Gmail → Respond | Sticky Note1 describing wiring and purpose of the workflow                                       |
| AI Prioritizer prompt instructs strict JSON output for clear, structured triage results                 | Embedded in AI Prioritizer node parameters                                                      |
| Airtable base and table IDs must be replaced with your own to persist findings                          | Airtable node parameters                                                                        |
| Gmail recipient email is hardcoded; replace with your team or alert mailbox                             | Gmail node parameters                                                                           |

---

**Disclaimer:** The provided text is exclusively derived from an automated n8n workflow designed for AWS security alert triage. It strictly adheres to content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.