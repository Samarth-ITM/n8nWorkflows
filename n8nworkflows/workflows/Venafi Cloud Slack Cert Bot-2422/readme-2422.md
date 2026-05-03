Venafi Cloud Slack Cert Bot

https://n8nworkflows.xyz/workflows/venafi-cloud-slack-cert-bot-2422


# Venafi Cloud Slack Cert Bot

### 1. Workflow Overview

The **Venafi Cloud Slack Cert Bot** workflow automates security operations related to Certificate Signing Requests (CSRs) within Slack. It enables users to submit CSR requests via Slack modals, assesses domain safety using VirusTotal scans, and either automatically approves the CSR or routes it for manual security team approval with AI-generated risk assessments.

**Target Use Cases:**
- Automating CSR requests and approvals in Slack.
- Enhancing security operations with real-time domain risk analysis.
- Centralizing CSR requests with audit-friendly, real-time logging.
- Integrating Venafi certificate issuance with Slack workflows.

**Logical Blocks:**

- **1.1 Input Reception & Parsing**
  - Receives Slack webhook events and parses incoming Slack interaction payloads.

- **1.2 Slack Interaction Routing**
  - Routes Slack messages and modal submissions based on interaction type and callback IDs.

- **1.3 Modal Display & Data Collection**
  - Opens Slack modal for CSR requests and extracts user input fields.

- **1.4 Domain Risk Analysis**
  - Queries VirusTotal API for domain risk data and summarizes results.

- **1.5 User & Team Metadata Resolution**
  - Translates Slack user and team IDs to emails and names via sub-workflows.

- **1.6 Certificate Issuance Decision**
  - Conditional logic to auto-approve CSR or generate AI report for manual approval.

- **1.7 Certificate Issuance & Slack Notification**
  - Issues CSR via Venafi TLS Protect Cloud API and sends confirmation messages back to Slack.

- **1.8 Manual CSR Approval Flow**
  - Sends detailed messages with AI risk reports to Slack channels for manual security team approval.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Parsing

- **Overview:**  
  Captures incoming Slack events via webhook and extracts the payload for downstream processing.

- **Nodes Involved:**  
  - Webhook  
  - Parse Webhook

- **Node Details:**

  - **Webhook**  
    - Type: Webhook  
    - Role: Entry point receiving POST requests from Slack Event Subscription API.  
    - Configuration: Listens on path `/venafiendpoint` with POST method, response mode set to a response node.  
    - Inputs: HTTP POST from Slack  
    - Outputs: Raw Slack event JSON to next node  
    - Edge Cases: Missing or malformed webhook payloads, unexpected HTTP methods.

  - **Parse Webhook**  
    - Type: Set  
    - Role: Extracts the `payload` object from Slack's nested JSON body for easier access.  
    - Configuration: Assigns `response` to `{{$json.body.payload}}`.  
    - Inputs: Webhook output JSON  
    - Outputs: Parsed Slack interaction object  
    - Edge Cases: Payload missing or malformed, expression errors.

---

#### 1.2 Slack Interaction Routing

- **Overview:**  
  Routes Slack interactions based on type (modal submission, block action, etc.) or callback IDs to appropriate workflow branches.

- **Nodes Involved:**  
  - Route Message  
  - Respond to Slack Webhook - Vulnerability  
  - Close Modal Popup  
  - Respond to webhook success

- **Node Details:**

  - **Route Message**  
    - Type: Switch  
    - Role: Evaluates Slack `response` object properties to route messages into three channels: "Request Modal", "Submit Data", "Block Actions".  
    - Configuration:  
      - Routes on `callback_id` equals `request-certificate` → "Request Modal"  
      - Routes on `type` equals `view_submission` → "Submit Data"  
      - Routes on `type` equals `block_actions` → "Block Actions"  
    - Inputs: Parsed Slack payload  
    - Outputs: Routes to different nodes based on conditions  
    - Edge Cases: Unmatched conditions fall to fallback "none" (no output)

  - **Respond to Slack Webhook - Vulnerability**  
    - Type: RespondToWebhook  
    - Role: Sends empty 200 OK response to Slack for "Request Modal" interactions to acknowledge event.  
    - Inputs: Route Message "Request Modal" output  
    - Outputs: None  
    - Edge Cases: Slack rate limiting or network errors.

  - **Close Modal Popup**  
    - Type: RespondToWebhook  
    - Role: Closes the Slack modal on "Submit Data" interactions with no data response (closes modal UI).  
    - Inputs: Route Message "Submit Data" output  
    - Outputs: Extract Fields, Get Slack User ID, Get Slack Team ID  
    - Edge Cases: Slack API errors during modal close.

  - **Respond to webhook success**  
    - Type: RespondToWebhook  
    - Role: Sends empty 200 OK response for block action events.  
    - Inputs: Route Message "Block Actions" output  
    - Outputs: Manual Issue Certificate node  
    - Edge Cases: Slack API errors on response.

---

#### 1.3 Modal Display & Data Collection

- **Overview:**  
  Displays a Slack modal for users to request new certificates, then extracts domain, validity period, and optional notes from submissions.

- **Nodes Involved:**  
  - Venafi Request Certificate  
  - Extract Fields

- **Node Details:**

  - **Venafi Request Certificate**  
    - Type: HTTP Request  
    - Role: Opens Slack modal via `views.open` API with a form for CSR requests.  
    - Configuration:  
      - Uses `trigger_id` from parsed Slack webhook data.  
      - Modal blocks include input fields for Domain Name, Validity Period (1 or 2 years), and Optional Note.  
      - Slack credentials configured for authentication.  
    - Inputs: Respond to Slack Webhook - Vulnerability acknowledgement  
    - Outputs: Modal submission data routed back to webhook.  
    - Edge Cases: Invalid `trigger_id`, Slack API errors, modal rendering issues.

  - **Extract Fields**  
    - Type: Set  
    - Role: Extracts user inputs from modal submission into simplified variables: `domain`, `validity`, `note`.  
    - Configuration: Uses expressions to map modal state values to variables.  
    - Inputs: Close Modal Popup output (modal submission)  
    - Outputs: Domain data to VirusTotal HTTP Request  
    - Edge Cases: Missing or malformed input fields, regex errors.

---

#### 1.4 Domain Risk Analysis

- **Overview:**  
  Queries the VirusTotal API to retrieve reputation and malware scan results for the requested domain.

- **Nodes Involved:**  
  - VirusTotal HTTP Request  
  - Summarize output to save on tokens

- **Node Details:**

  - **VirusTotal HTTP Request**  
    - Type: HTTP Request  
    - Role: Fetches domain scan results from VirusTotal API v3.  
    - Configuration:  
      - URL dynamically constructed with domain input.  
      - API key set in header `X-Apikey`.  
      - Accept header: `application/json`.  
    - Inputs: Extract Fields output domain variable  
    - Outputs: Full VirusTotal domain scan JSON.  
    - Edge Cases: API key invalid, domain not found, rate limiting, network timeouts.

  - **Summarize output to save on tokens**  
    - Type: Set  
    - Role: Extracts and simplifies key VirusTotal analysis stats (malicious, suspicious, undetected, harmless, timeout, reputation) for further use.  
    - Inputs: VirusTotal HTTP Request output  
    - Outputs: Simplified risk data for conditional logic and AI processing.  
    - Edge Cases: Missing fields, unexpected data structure.

---

#### 1.5 User & Team Metadata Resolution

- **Overview:**  
  Resolves Slack user and team IDs to human-readable emails, names, and avatars to enhance message context.

- **Nodes Involved:**  
  - Get Slack User ID  
  - Translate Slack User ID to Email (sub-workflow)  
  - Get Slack Team ID  
  - Execute Workflow (Slack Team ID to Name sub-workflow)  
  - Merge User and Team Data  
  - Merge Requestor and VT Data

- **Node Details:**

  - **Get Slack User ID**  
    - Type: Set  
    - Role: Extracts Slack user ID from parsed webhook response.  
    - Inputs: Close Modal Popup output  
    - Outputs: Slack user ID to Translate Slack User ID to Email  
    - Edge Cases: Missing user ID in payload.

  - **Translate Slack User ID to Email**  
    - Type: Execute Workflow (Sub-workflow)  
    - Role: Converts Slack user ID to email and name using a dedicated sub-workflow.  
    - Configuration: Waits for sub-workflow to complete before continuing.  
    - Inputs: Get Slack User ID output  
    - Outputs: User email and name data.  
    - Sub-workflow ID: `afeVlIVyoIF8Psu4`  
    - Edge Cases: Slack API errors, missing user info.

  - **Get Slack Team ID**  
    - Type: Set  
    - Role: Extracts Slack team ID from webhook payload.  
    - Inputs: Close Modal Popup output  
    - Outputs: Team ID to Execute Workflow  
    - Edge Cases: Missing team ID.

  - **Execute Workflow (Slack Team ID to Name)**  
    - Type: Execute Workflow (Sub-workflow)  
    - Role: Converts Slack team ID to team name and avatar info.  
    - Inputs: Get Slack Team ID output  
    - Outputs: Team metadata.  
    - Sub-workflow ID: `ZIl9VdWh7BiVRRBT`  
    - Edge Cases: Slack API errors.

  - **Merge User and Team Data**  
    - Type: Merge  
    - Role: Combines user email/name and team name/avatar into one JSON object by position.  
    - Inputs: Outputs of Translate Slack User ID to Email and Execute Workflow (team)  
    - Outputs: Combined user and team metadata.  
    - Edge Cases: Mismatched data lengths.

  - **Merge Requestor and VT Data**  
    - Type: Merge  
    - Role: Combines merged user/team data with VirusTotal summary data for final CSR decision.  
    - Inputs: Merge User and Team Data output and Summarize output to save on tokens output  
    - Outputs: Data for conditional check on malicious reports.  
    - Edge Cases: Data alignment issues.

---

#### 1.6 Certificate Issuance Decision

- **Overview:**  
  Uses conditional logic to decide whether to auto-issue the CSR or generate an AI report for manual approval based on VirusTotal malicious report count.

- **Nodes Involved:**  
  - Auto Issue Certificate Based on 0 Malicious Reports  
  - Auto Issue Certificate  
  - Generate Report For Manual Approval

- **Node Details:**

  - **Auto Issue Certificate Based on 0 Malicious Reports**  
    - Type: If  
    - Role: Checks if VirusTotal malicious count is less than or equal to zero.  
    - Inputs: Merge Requestor and VT Data output  
    - Outputs:  
      - True → Auto Issue Certificate  
      - False → Generate Report For Manual Approval  
    - Edge Cases: Missing or invalid malicious count.

  - **Auto Issue Certificate**  
    - Type: NoOp (placeholder)  
    - Role: Acts as a marker node to continue auto-issue flow.  
    - Inputs: If true condition  
    - Outputs: Venafi TLS Protect Cloud  
    - Edge Cases: None.

  - **Generate Report For Manual Approval**  
    - Type: NoOp (placeholder)  
    - Role: Acts as a marker node to continue manual approval flow.  
    - Inputs: If false condition  
    - Outputs: OpenAI  
    - Edge Cases: None.

---

#### 1.7 Certificate Issuance & Slack Notification (Auto-Approval)

- **Overview:**  
  Automatically issues a CSR via Venafi TLS Protect Cloud API and sends a confirmation message back to Slack.

- **Nodes Involved:**  
  - Venafi TLS Protect Cloud  
  - Send Auto Generated Confirmation

- **Node Details:**

  - **Venafi TLS Protect Cloud**  
    - Type: Venafi TLS Protect Cloud node  
    - Role: Generates Certificate Signing Request automatically with input domain and organizational unit.  
    - Configuration:  
      - `commonName` parsed from Slack modal input domain with regex validation.  
      - CSR generation enabled.  
      - Application and Template IDs set for Venafi environment.  
      - Organizational unit set from user metadata.  
    - Credentials: Venafi TLS Protect Cloud API configured.  
    - Inputs: Auto Issue Certificate output  
    - Outputs: CSR metadata including issuance date and ID.  
    - Edge Cases: API errors, invalid domain names, missing credentials.

  - **Send Auto Generated Confirmation**  
    - Type: Slack node  
    - Role: Sends a formatted block message to a Slack channel confirming CSR auto-issuance.  
    - Configuration:  
      - Channel hardcoded by ID.  
      - Message includes team name, requestor info, CSR details, buttons for viewing or revoking CSR.  
      - Uses data from Venafi TLS Protect Cloud and merged user/team data nodes.  
    - Credentials: Slack API credentials configured.  
    - Inputs: Venafi TLS Protect Cloud output  
    - Outputs: None  
    - Edge Cases: Slack API rate limits, missing data for message formatting.

---

#### 1.8 Manual CSR Approval Flow

- **Overview:**  
  Generates an AI analysis report for risky domains and sends a detailed Slack message to a designated team channel for manual CSR approval.

- **Nodes Involved:**  
  - OpenAI  
  - Send Message Request for Manual Approval  
  - Venafi TLS Protect Cloud1  
  - Send Auto Generated Confirmation1  
  - Manual Issue Certificate

- **Node Details:**

  - **OpenAI**  
    - Type: OpenAI (LangChain)  
    - Role: Analyzes VirusTotal scan results and categorizes domain risk as Low, Medium, or High, recommending next steps.  
    - Configuration:  
      - Model GPT-4o-mini used.  
      - Prompts use VirusTotal scan stats with instructions for risk rating and recommendations.  
    - Credentials: OpenAI API key configured.  
    - Inputs: Generate Report For Manual Approval output (VirusTotal data)  
    - Outputs: AI-generated concise risk assessment message.  
    - Edge Cases: API errors, model timeout, prompt failures.

  - **Send Message Request for Manual Approval**  
    - Type: Slack node  
    - Role: Sends detailed message to Slack channel including AI analysis, CSR details, requester/team info, and action buttons for manual approval.  
    - Configuration:  
      - Channel hardcoded by ID.  
      - Message blocks contain formatted text with AI report and buttons for "Submit for Approval" and "View CSR Details".  
      - Uses merged user/team data and AI message content with formatting cleanup.  
    - Credentials: Slack API configured.  
    - Inputs: OpenAI output  
    - Outputs: None  
    - Edge Cases: Slack API errors, message formatting issues.

  - **Venafi TLS Protect Cloud1**  
    - Type: Venafi TLS Protect Cloud node  
    - Role: Issues CSR for manual approval flow based on Slack message data fields.  
    - Configuration:  
      - Extracts domain and organizational unit from Slack message text via regex.  
      - CSR generation enabled.  
      - Same Venafi application and template IDs as auto-issue.  
    - Credentials: Venafi TLS Protect Cloud API configured.  
    - Inputs: Respond to webhook success output (triggered by block actions)  
    - Outputs: CSR metadata.  
    - Edge Cases: Regex parsing errors, API errors.

  - **Send Auto Generated Confirmation1**  
    - Type: Slack node  
    - Role: Sends confirmation message to Slack after manual CSR issuance.  
    - Configuration: Similar to auto-issue confirmation, but data extracted from Slack message blocks.  
    - Credentials: Slack API configured.  
    - Inputs: Venafi TLS Protect Cloud1 output  
    - Outputs: None  
    - Edge Cases: Slack API errors.

  - **Manual Issue Certificate**  
    - Type: NoOp (marker)  
    - Role: Placeholder for manual CSR issuance flow continuation.  
    - Inputs: Respond to webhook success output  
    - Outputs: Venafi TLS Protect Cloud1  
    - Edge Cases: None.

---

### 3. Summary Table

| Node Name                             | Node Type                     | Functional Role                                      | Input Node(s)                  | Output Node(s)                         | Sticky Note                                                                                                                          |
|-------------------------------------|-------------------------------|-----------------------------------------------------|--------------------------------|---------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| Webhook                             | Webhook                       | Receives Slack events                               | —                              | Parse Webhook                         | ![Imgur](https://i.imgur.com/iKyMV0N.png) The first node receives all messages from Slack API via Subscription Events API. Setup [here](https://api.slack.com/apis/connections/events-api). |
| Parse Webhook                       | Set                           | Parses Slack payload                                 | Webhook                       | Route Message                        | (Same as above)                                                                                                                      |
| Route Message                      | Switch                        | Routes Slack interactions based on type/callback   | Parse Webhook                 | Respond to Slack Webhook - Vulnerability, Close Modal Popup, Respond to webhook success | ![n8n](https://i.imgur.com/lKnBNnH.png) Efficient Slack Interaction Handling. Routes commands/modals to appropriate actions.          |
| Respond to Slack Webhook - Vulnerability | RespondToWebhook             | Responds to Slack for modal open events             | Route Message                 | Venafi Request Certificate           |                                                                                                                                    |
| Venafi Request Certificate          | HTTP Request                  | Opens Slack modal for CSR request                    | Respond to Slack Webhook - Vulnerability | Close Modal Popup                    | ![Imgur](https://i.imgur.com/iKyMV0N.png) Display Modal Popup section. Learn more about Slack modals [here](https://api.slack.com/surfaces/modals) |
| Close Modal Popup                   | RespondToWebhook              | Closes Slack modal on submission                     | Route Message                 | Extract Fields, Get Slack User ID, Get Slack Team ID |                                                                                                                                    |
| Extract Fields                     | Set                           | Extracts domain, validity, and notes from modal     | Close Modal Popup             | VirusTotal HTTP Request              |                                                                                                                                    |
| VirusTotal HTTP Request             | HTTP Request                  | Fetches domain risk data from VirusTotal            | Extract Fields                | Summarize output to save on tokens   | ![VirusTotal](https://upload.wikimedia.org/wikipedia/commons/thumb/b/b7/VirusTotal_logo.svg/320px-VirusTotal_logo.svg.png) URL Analysis with VirusTotal |
| Summarize output to save on tokens | Set                           | Simplifies VirusTotal stats for downstream use      | VirusTotal HTTP Request       | Merge Requestor and VT Data          |                                                                                                                                    |
| Get Slack User ID                  | Set                           | Extracts Slack user ID                               | Close Modal Popup             | Translate Slack User ID to Email      |                                                                                                                                    |
| Translate Slack User ID to Email    | Execute Workflow              | Sub-workflow to convert Slack user ID to email     | Get Slack User ID             | Merge User and Team Data              | ![n8n](https://i.imgur.com/qXWqiOd.png) Runs sub-workflows to translate Slack IDs to readable data for richer messages.             |
| Get Slack Team ID                  | Set                           | Extracts Slack team ID                               | Close Modal Popup             | Execute Workflow (Slack Team ID to Name) |                                                                                                                                    |
| Execute Workflow (Slack Team ID to Name) | Execute Workflow              | Sub-workflow to convert Slack team ID to name/avatar | Get Slack Team ID             | Merge User and Team Data              |                                                                                                                                    |
| Merge User and Team Data            | Merge                         | Combines user and team metadata                      | Translate Slack User ID to Email, Execute Workflow | Merge Requestor and VT Data          |                                                                                                                                    |
| Merge Requestor and VT Data         | Merge                         | Combines user/team data with VirusTotal summary     | Merge User and Team Data, Summarize output to save on tokens | Auto Issue Certificate Based on 0 Malicious Reports |                                                                                                                                    |
| Auto Issue Certificate Based on 0 Malicious Reports | If                            | Checks if domain has zero malicious reports         | Merge Requestor and VT Data   | Auto Issue Certificate, Generate Report For Manual Approval |                                                                                                                                    |
| Auto Issue Certificate              | NoOp                          | Placeholder for auto-issue flow                      | Auto Issue Certificate Based on 0 Malicious Reports (true) | Venafi TLS Protect Cloud             |                                                                                                                                    |
| Generate Report For Manual Approval | NoOp                          | Placeholder for manual approval flow                 | Auto Issue Certificate Based on 0 Malicious Reports (false) | OpenAI                              |                                                                                                                                    |
| Venafi TLS Protect Cloud            | VenafiTlsProtectCloud         | Issues CSR automatically via Venafi                  | Auto Issue Certificate        | Send Auto Generated Confirmation     | ![VirusTotal](https://img.securityinfowatch.com/files/base/cygnus/siw/image/2022/10/Venafi_logo.63459e2b03b7b.png) Automatic CSR Generation via Venafi |
| Send Auto Generated Confirmation   | Slack                         | Sends confirmation message for auto-issued CSR      | Venafi TLS Protect Cloud      | —                                   | ![Imgur](https://i.imgur.com/iKyMV0N.png) Sends contextual message to Slack with CSR details and links.                               |
| OpenAI                             | OpenAI (LangChain)            | Generates AI risk report for manual approval        | Generate Report For Manual Approval | Send Message Request for Manual Approval | ![OpenAI](https://i.imgur.com/o89G0If.png) Parse Response with AI Model. Can be replaced with local AI LLM.                        |
| Send Message Request for Manual Approval | Slack                         | Sends detailed message to Slack for manual CSR approval | OpenAI                       | —                                   |                                                                                                                                    |
| Respond to webhook success          | RespondToWebhook              | Responds to Slack for block actions                  | Route Message                 | Manual Issue Certificate             |                                                                                                                                    |
| Manual Issue Certificate            | NoOp                          | Placeholder for manual CSR issuance flow             | Respond to webhook success    | Venafi TLS Protect Cloud1            |                                                                                                                                    |
| Venafi TLS Protect Cloud1           | VenafiTlsProtectCloud         | Issues CSR for manual approval flow                   | Manual Issue Certificate      | Send Auto Generated Confirmation1    | ![VirusTotal](https://img.securityinfowatch.com/files/base/cygnus/siw/image/2022/10/Venafi_logo.63459e2b03b7b.png) Manual CSR Generation via Venafi |
| Send Auto Generated Confirmation1  | Slack                         | Sends confirmation message for manually issued CSR   | Venafi TLS Protect Cloud1     | —                                   |                                                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - Path: `venafiendpoint`  
   - Method: POST  
   - Response Mode: Response Node

2. **Add Set Node ("Parse Webhook")**  
   - Extract `response` as `{{$json.body.payload}}` from webhook output.

3. **Add Switch Node ("Route Message")**  
   - Route based on:  
     - `callback_id` equals `request-certificate` → Output "Request Modal"  
     - `type` equals `view_submission` → Output "Submit Data"  
     - `type` equals `block_actions` → Output "Block Actions"  
   - Fallback output: none

4. **Add RespondToWebhook Node ("Respond to Slack Webhook - Vulnerability")**  
   - For "Request Modal" route output  
   - Respond with no data (200 OK)

5. **Add HTTP Request Node ("Venafi Request Certificate")**  
   - Method: POST  
   - URL: `https://slack.com/api/views.open`  
   - Authentication: Slack API credentials  
   - Body (JSON): Slack modal JSON with:  
     - Trigger ID: from parsed webhook (`$('Parse Webhook').item.json['response']['trigger_id']`)  
     - Modal with inputs: Domain Name, Validity Period (1 or 2 years), Optional Note  
     - Include Venafi logo image block  
   - Connected from "Respond to Slack Webhook - Vulnerability"

6. **Add RespondToWebhook Node ("Close Modal Popup")**  
   - For "Submit Data" route output  
   - Respond with no data (closes modal)

7. **Add Set Node ("Extract Fields")**  
   - Extract from modal submission state:  
     - `domain` from `domain_name_input`  
     - `validity` from `validity_period_select`  
     - `note` from `optional_note_input` (optional)

8. **Add HTTP Request Node ("VirusTotal HTTP Request")**  
   - Method: GET  
   - URL: `https://www.virustotal.com/api/v3/domains/{{ $json.domain }}`  
   - Headers:  
     - `accept: application/json`  
     - `X-Apikey`: your VirusTotal API key  
   - Credentials: VirusTotal API credentials  
   - Input: domain from Extract Fields

9. **Add Set Node ("Summarize output to save on tokens")**  
   - Extract key stats from VirusTotal response JSON:  
     - `malicious`, `suspicious`, `undetected`, `harmless`, `timeout`, `reputation`

10. **Add Set Node ("Get Slack User ID")**  
    - Extract Slack user ID from modal submission: `{{$json.response.user.id}}`

11. **Add Execute Workflow Node ("Translate Slack User ID to Email")**  
    - Configure to call sub-workflow `afeVlIVyoIF8Psu4`  
    - Pass Slack user ID, wait for result

12. **Add Set Node ("Get Slack Team ID")**  
    - Extract Slack team ID from modal submission: `{{$json.response.team.id}}`

13. **Add Execute Workflow Node ("Slack Team ID to Name")**  
    - Configure to call sub-workflow `ZIl9VdWh7BiVRRBT`  
    - Pass Slack team ID

14. **Add Merge Node ("Merge User and Team Data")**  
    - Combine outputs from Slack user email/name and Slack team data by position

15. **Add Merge Node ("Merge Requestor and VT Data")**  
    - Combine merged user/team data with VirusTotal summary by position

16. **Add If Node ("Auto Issue Certificate Based on 0 Malicious Reports")**  
    - Condition: `malicious` ≤ 0  
    - True: auto-issue flow  
    - False: manual approval flow

17. **Add NoOp Node ("Auto Issue Certificate")**  
    - Placeholder for auto-issue branch

18. **Add Venafi TLS Protect Cloud Node ("Venafi TLS Protect Cloud")**  
    - Generate CSR with:  
      - `commonName` from domain (validate with regex)  
      - Application and Template IDs as per Venafi config  
      - Organizational units from user/team data  
    - Credentials: Venafi TLS Protect Cloud API

19. **Add Slack Node ("Send Auto Generated Confirmation")**  
    - Send message to Slack channel with CSR details, team, requestor info, buttons for viewing/revoking CSR

20. **Add NoOp Node ("Generate Report For Manual Approval")**  
    - Placeholder for manual approval branch

21. **Add OpenAI Node ("OpenAI")**  
    - Model: GPT-4o-mini  
    - Input prompt: summarize VirusTotal risk and recommend action  
    - Credentials: OpenAI API key

22. **Add Slack Node ("Send Message Request for Manual Approval")**  
    - Send detailed message to Slack channel including AI report, CSR details, and action buttons

23. **Add RespondToWebhook Node ("Respond to webhook success")**  
    - For "Block Actions" route output  
    - Respond with no data

24. **Add NoOp Node ("Manual Issue Certificate")**  
    - Placeholder for manual CSR issuance

25. **Add Venafi TLS Protect Cloud Node ("Venafi TLS Protect Cloud1")**  
    - Issue CSR from Slack message data extracted by regex  
    - Credentials: Venafi API

26. **Add Slack Node ("Send Auto Generated Confirmation1")**  
    - Send confirmation message after manual CSR issuance

27. **Connect nodes accordingly based on the provided connections in the original workflow JSON.**

**Credentials Needed:**

- Slack API credentials with permissions for modals and chat messages.
- VirusTotal API key.
- Venafi TLS Protect Cloud API credentials.
- OpenAI API key (or replace with your AI provider).

**Sub-workflows:**

- Slack User ID to Email: workflow ID `afeVlIVyoIF8Psu4`
- Slack Team ID to Name: workflow ID `ZIl9VdWh7BiVRRBT`

---

### 5. General Notes & Resources

| Note Content                                                                                                                     | Context or Link                                                                                  |
|---------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Venafi Presentation Video showing the CertBot functionality.                                                                     | https://www.loom.com/share/5f0df8389fc34183a13af1410b997802                                      |
| Slack Subscription Events API setup documentation.                                                                               | https://api.slack.com/apis/connections/events-api                                               |
| Slack Modals documentation for creating interactive modals.                                                                      | https://api.slack.com/surfaces/modals                                                          |
| Venafi official documentation for certificate management.                                                                        | https://docs.venafi.com/                                                                        |
| n8n Community for support on workflow design and credentials setup.                                                              | https://community.n8n.io                                                                        |
| VirusTotal logo and info integrated for domain risk analysis.                                                                     | https://www.virustotal.com/                                                                     |
| OpenAI integration can be replaced with local AI LLMs supported by n8n.                                                          |                                                                                              |
| Reminder: Slack credentials must be added to Slack nodes to manage authentication smoothly.                                        |                                                                                              |

---

This documentation provides a detailed and structured understanding of the **Venafi Cloud Slack Cert Bot** workflow, enabling advanced users and automation agents to comprehend, reproduce, and extend the workflow confidently.