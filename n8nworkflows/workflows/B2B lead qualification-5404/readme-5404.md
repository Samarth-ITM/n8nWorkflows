B2B lead qualification

https://n8nworkflows.xyz/workflows/b2b-lead-qualification-5404


# B2B lead qualification

### 1. Workflow Overview

This workflow automates B2B lead qualification by integrating Google Sheets with a voice call assistant platform (VAPI.ai). It targets sales and marketing teams seeking to streamline lead capture, qualification via automated voice calls, and structured data storage. The workflow is logically divided into these blocks:

- **1.1 New Lead Detection**: Watches a Google Sheet for new phone number entries signaling fresh leads.
- **1.2 Voice Call Initiation**: Triggers an outbound automated voice call through VAPI using captured lead data.
- **1.3 Lead Qualification Reception**: Receives detailed lead qualification data asynchronously from VAPI via webhook POST.
- **1.4 Data Persistence**: Saves qualified lead information into a dedicated Google Sheet CRM.
- **1.5 Confirmation Response**: Sends acknowledgement back to VAPI confirming successful data processing.

Sticky notes provide contextual guidance for each block, emphasizing trigger points, data capture, qualification, storage, and response confirmation.

---

### 2. Block-by-Block Analysis

#### 2.1 New Lead Detection

- **Overview:**  
Detects when a new lead phone number is added to a specified Google Sheet and triggers the workflow.

- **Nodes Involved:**  
  - New Lead Captured  
  - Sticky Note (Triggers on new phone number entry)

- **Node Details:**  

  - **New Lead Captured**  
    - Type: Google Sheets Trigger  
    - Role: Monitors a Google Sheet tab named "phone_number" for newly added rows every minute.  
    - Configuration:  
      - Spreadsheet ID set to target a specific Google Sheet document containing leads.  
      - Sheet name set to the tab containing phone numbers.  
      - Polling frequency: every minute for row additions.  
      - OAuth2 credentials configured for Google Sheets API access.  
    - Inputs: None (trigger)  
    - Outputs: Emits data on newly added rows (lead phone numbers).  
    - Edge cases:  
      - Google API rate limits or auth expiration.  
      - Delay in polling could add latency.  
      - Duplicate rows may cause repeated triggers if not managed externally.  
    - Sticky Note: "Triggers on new phone number entry."

---

#### 2.2 Voice Call Initiation

- **Overview:**  
Initiates an outbound voice call via the VAPI platform using lead phone numbers captured previously.

- **Nodes Involved:**  
  - Initiate Voice Call (VAPI)  
  - Sticky Note1 (Captures name, company, challenges via POST.)

- **Node Details:**  

  - **Initiate Voice Call (VAPI)**  
    - Type: HTTP Request  
    - Role: Sends a POST request to VAPI's call API endpoint to start an automated voice call.  
    - Configuration:  
      - URL: `https://api.vapi.ai/call`  
      - Method: POST  
      - Body: JSON with placeholder fields for assistantId, phoneNumberId, and customers array containing the agent's number. This must be replaced with actual IDs and numbers before execution.  
      - Headers: Authorization token header, must be provided with a valid bearer token.  
      - Sends body and headers as JSON.  
    - Inputs: Receives new lead phone number data from "New Lead Captured."  
    - Outputs: API response from VAPI indicating call initiation status.  
    - Edge cases:  
      - Authorization failures if token invalid/expired.  
      - Network timeouts or API errors.  
      - Misconfigured assistantId or phoneNumberId causes call initiation failure.  
    - Sticky Note1: "Captures name, company, challenges via POST." (Indicates expected data flow from voice interaction.)

---

#### 2.3 Lead Qualification Reception

- **Overview:**  
Receives qualified lead details asynchronously via webhook POST from VAPI after the call interaction completes.

- **Nodes Involved:**  
  - Receive Lead Details from VAPI (Webhook)  
  - Sticky Note2 (Qualifies lead with outbound call.)

- **Node Details:**  

  - **Receive Lead Details from VAPI**  
    - Type: Webhook  
    - Role: Listens for POST requests at a unique path to receive lead qualification data from VAPI.  
    - Configuration:  
      - HTTP Method: POST  
      - Path: Unique webhook ID path assigned by n8n.  
      - Response Mode: Uses "Send Call Data Acknowledgement" node for response.  
    - Inputs: External POST from VAPI containing JSON with lead qualification details (name, company name, company size).  
    - Outputs: Passes received data downstream for storage.  
    - Edge cases:  
      - Webhook URL must be publicly accessible and properly configured in VAPI.  
      - Data malformed or incomplete could cause downstream failures.  
      - Unauthorized or replay attacks if security not enforced externally.  
    - Sticky Note2: "Qualifies lead with outbound call."

---

#### 2.4 Data Persistence

- **Overview:**  
Stores the qualified lead data in a Google Sheet acting as a CRM database.

- **Nodes Involved:**  
  - Save Qualified Lead to CRM Sheet  
  - Sticky Note3 (Stores data in Google Sheet.)

- **Node Details:**  

  - **Save Qualified Lead to CRM Sheet**  
    - Type: Google Sheets (Append or Update)  
    - Role: Appends or updates lead info in a CRM sheet with columns for Name, Company name, and Company size.  
    - Configuration:  
      - Spreadsheet ID and sheet name targeting the CRM sheet tab.  
      - Mapping fields: Extracts name, company_name, and company_size from the webhook JSON payload using JSON expressions.  
      - Operation: Append or update based on matching "Name".  
      - Authentication: Uses Google Service Account credentials.  
    - Inputs: Receives lead data JSON from "Receive Lead Details from VAPI."  
    - Outputs: Confirmation of data write operation.  
    - Edge cases:  
      - Google API quota or auth issues.  
      - Duplicate names could cause unintended overwrites depending on data quality.  
      - Data conversion failures if types mismatch.  
    - Sticky Note3: "Stores data in Google Sheet."

---

#### 2.5 Confirmation Response

- **Overview:**  
Sends a success acknowledgement back to the VAPI platform confirming lead data was saved.

- **Nodes Involved:**  
  - Send Call Data Acknowledgement  
  - Sticky Note4 (Confirms success to VAPI.)

- **Node Details:**  

  - **Send Call Data Acknowledgement**  
    - Type: Respond to Webhook  
    - Role: Sends HTTP 200 OK response to the webhook POST caller (VAPI), signaling successful processing.  
    - Configuration: Default HTTP 200 response with no custom body.  
    - Inputs: Triggered after data persistence node completes successfully.  
    - Outputs: Terminates workflow with acknowledgement sent.  
    - Edge cases:  
      - If prior node fails, response may not be sent, causing retries or timeouts in VAPI.  
    - Sticky Note4: "Confirms success to VAPI."

---

### 3. Summary Table

| Node Name                    | Node Type               | Functional Role                        | Input Node(s)              | Output Node(s)              | Sticky Note                                |
|------------------------------|-------------------------|-------------------------------------|----------------------------|----------------------------|--------------------------------------------|
| New Lead Captured             | Google Sheets Trigger   | Detect new lead phone number entry  | -                          | Initiate Voice Call (VAPI)  | Triggers on new phone number entry.        |
| Initiate Voice Call (VAPI)    | HTTP Request            | Initiate outbound voice call via VAPI | New Lead Captured          | -                          | Captures name, company, challenges via POST. |
| Receive Lead Details from VAPI| Webhook                 | Receive qualified lead data from VAPI | -                          | Save Qualified Lead to CRM Sheet | Qualifies lead with outbound call.          |
| Save Qualified Lead to CRM Sheet | Google Sheets         | Store qualified lead info in CRM    | Receive Lead Details from VAPI | Send Call Data Acknowledgement | Stores data in Google Sheet.                 |
| Send Call Data Acknowledgement| Respond to Webhook      | Confirm successful data processing  | Save Qualified Lead to CRM Sheet | -                          | Confirms success to VAPI.                    |
| Sticky Note                  | Sticky Note             | Contextual information               | -                          | -                          | Triggers on new phone number entry.         |
| Sticky Note1                 | Sticky Note             | Contextual information               | -                          | -                          | Captures name, company, challenges via POST.|
| Sticky Note2                 | Sticky Note             | Contextual information               | -                          | -                          | Qualifies lead with outbound call.          |
| Sticky Note3                 | Sticky Note             | Contextual information               | -                          | -                          | Stores data in Google Sheet.                 |
| Sticky Note4                 | Sticky Note             | Contextual information               | -                          | -                          | Confirms success to VAPI.                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Sheets Trigger Node**  
   - Name: `New Lead Captured`  
   - Type: Google Sheets Trigger  
   - Configure credentials for Google Sheets OAuth2.  
   - Set Document ID to your Google Sheet containing lead phone numbers.  
   - Set Sheet Name to the tab holding phone numbers (e.g., "phone_number").  
   - Event: Row Added  
   - Polling Interval: Every minute  
   - Save and activate trigger.

2. **Create HTTP Request Node for VAPI Call**  
   - Name: `Initiate Voice Call (VAPI)`  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.vapi.ai/call`  
   - Headers: Add `Authorization` header with valid bearer token for VAPI API.  
   - Body Content Type: JSON  
   - JSON Body Template:  
     ```json
     {
       "assistantId": "your_assistant_id",
       "phoneNumberId": "your_phone_number_id",
       "customers": [
         {
           "number": "your_agent_phone_number"
         }
       ]
     }
     ```  
   - Connect input from `New Lead Captured` node's output.  
   - Save node.

3. **Create Webhook Node to Receive Lead Details**  
   - Name: `Receive Lead Details from VAPI`  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: Generate unique webhook path or use the provided webhook ID.  
   - Response Mode: Set to "Respond with node" and select the next node (to be created).  
   - Save node.

4. **Create Google Sheets Node to Save Qualified Leads**  
   - Name: `Save Qualified Lead to CRM Sheet`  
   - Type: Google Sheets  
   - Operation: Append or Update Row  
   - Configure Google API Service Account credentials.  
   - Set Document ID to your CRM Google Sheet (can be same or different from trigger sheet).  
   - Sheet Name: The tab where leads are saved (e.g., "Sheet1").  
   - Define columns: "Name", "Company name", "Company size".  
   - Map fields with expressions extracting values from webhook JSON payload, e.g.:  
     - Name: `{{$json.body.message.toolCallList[0].function.arguments.name}}`  
     - Company name: `{{$json.body.message.toolCallList[0].function.arguments.company_name}}`  
     - Company size: `{{$json.body.message.toolCallList[0].function.arguments.company_size}}`  
   - Matching Columns: "Name" (to avoid duplicates).  
   - Connect input from `Receive Lead Details from VAPI`.  
   - Save node.

5. **Create Respond to Webhook Node**  
   - Name: `Send Call Data Acknowledgement`  
   - Type: Respond to Webhook  
   - Leave default response (HTTP 200 OK).  
   - Connect input from `Save Qualified Lead to CRM Sheet`.  
   - Save node.

6. **Connect Nodes**  
   - Connect `New Lead Captured` → `Initiate Voice Call (VAPI)`  
   - Connect `Receive Lead Details from VAPI` → `Save Qualified Lead to CRM Sheet` → `Send Call Data Acknowledgement`  

7. **Add Sticky Notes (Optional for clarity)**  
   - Add sticky notes near related nodes with contents:  
     - "Triggers on new phone number entry." near `New Lead Captured`  
     - "Captures name, company, challenges via POST." near `Initiate Voice Call (VAPI)`  
     - "Qualifies lead with outbound call." near `Receive Lead Details from VAPI`  
     - "Stores data in Google Sheet." near `Save Qualified Lead to CRM Sheet`  
     - "Confirms success to VAPI." near `Send Call Data Acknowledgement`

8. **Activate Workflow**  
   - Ensure all credentials (Google Sheets OAuth2 and Service Account, VAPI Authorization token) are configured correctly.  
   - Activate the workflow to start listening for new leads and processing them.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                          |
|-----------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| VAPI API documentation and authentication details must be obtained from https://vapi.ai/                 | VAPI platform https://vapi.ai/                                                                            |
| Ensure Google Sheets API quotas and service account permissions are set correctly for both reading and writing | Google Sheets API documentation: https://developers.google.com/sheets/api                               |
| Webhook URL must be publicly accessible to receive POST requests from VAPI; consider using n8n cloud or tunneling | n8n webhook documentation: https://docs.n8n.io/nodes/n8n-nodes-base.webhook/                             |
| The workflow assumes the lead qualification data JSON structure is consistent with VAPI's message format | Adjust expressions if VAPI changes payload schema                                                        |
| For security, consider validating incoming webhook requests (e.g., with tokens or IP whitelisting)         | Implement additional security measures outside this workflow                                             |

---

**Disclaimer:** The text provided is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing adheres strictly to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.