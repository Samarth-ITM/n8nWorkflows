Display ServiceNow Incident Details in Slack using Slash Commands

https://n8nworkflows.xyz/workflows/display-servicenow-incident-details-in-slack-using-slash-commands-2727


# Display ServiceNow Incident Details in Slack using Slash Commands

### 1. Workflow Overview

This workflow enables Slack users to retrieve ServiceNow incident details directly within Slack by using a Slash Command. It is designed for teams that rely on Slack for communication and ServiceNow for incident management, streamlining the process of incident lookup without switching platforms.

The workflow is logically divided into three main blocks:

- **1.1 Input Reception:** Listens for Slack Slash Command requests and extracts the incident ID from the incoming payload.
- **1.2 Incident Lookup in ServiceNow:** Queries ServiceNow for the incident matching the extracted ID and evaluates the response.
- **1.3 Slack Response Handling:** Sends a formatted message back to Slack with incident details or appropriate notifications depending on the query outcome.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block captures the incoming Slack Slash Command POST request and extracts the incident ID from the payload to initiate the lookup process.

**Nodes Involved:**  
- Webhook  
- Extract Incident ID from Response

**Node Details:**

- **Webhook**  
  - *Type:* Webhook  
  - *Role:* Entry point that listens for HTTP POST requests from Slack Slash Commands.  
  - *Configuration:*  
    - HTTP Method: POST  
    - Path: Unique webhook path (`f6ec2074-6c23-410e-ad31-ac1eaf7381ad`)  
    - Response Mode: Response node (delays response until downstream nodes complete)  
  - *Input:* Incoming HTTP POST from Slack Slash Command  
  - *Output:* JSON payload containing Slack command data  
  - *Edge Cases:*  
    - Invalid or missing payload from Slack  
    - Unauthorized requests if Slack verification is not configured  
  - *Sticky Note:* Explains the Slack webhook reception and incident ID extraction process.

- **Extract Incident ID from Response**  
  - *Type:* Set  
  - *Role:* Extracts the incident ID from the Slack command text field.  
  - *Configuration:*  
    - Assigns a new variable `incident_id` from the expression `{{$json.body.text}}` which contains the incident ID typed by the user in Slack.  
  - *Input:* JSON from Webhook node  
  - *Output:* JSON with added `incident_id` property  
  - *Edge Cases:*  
    - Empty or malformed incident ID input  
    - Unexpected Slack payload structure

---

#### 2.2 Incident Lookup in ServiceNow

**Overview:**  
This block queries the ServiceNow instance for the incident matching the extracted ID and routes the workflow based on the query result.

**Nodes Involved:**  
- Search For Incident in ServiceNow  
- Parse ServiceNow Response

**Node Details:**

- **Search For Incident in ServiceNow**  
  - *Type:* ServiceNow  
  - *Role:* Queries ServiceNow incidents using the incident ID.  
  - *Configuration:*  
    - Resource: Incident  
    - Operation: Get All (search)  
    - Query: `number={{ $json.incident_id }}` (searches for incident number matching the extracted ID)  
    - Display Value: true (returns human-readable field values)  
    - Authentication: Basic Auth with configured ServiceNow credentials  
    - On Error: Continue regular output (prevents workflow failure on errors)  
  - *Input:* JSON containing `incident_id`  
  - *Output:* JSON array of incident records or error object  
  - *Edge Cases:*  
    - No incident found (empty result)  
    - Authentication failure or connectivity issues  
    - Malformed query or invalid incident ID  
  - *Sticky Note:* Describes the ServiceNow query process and response evaluation.

- **Parse ServiceNow Response**  
  - *Type:* Switch  
  - *Role:* Routes workflow based on ServiceNow query results.  
  - *Configuration:*  
    - Three outputs based on conditions:  
      - **ServiceNow Error:** If `$json.error` exists (indicates API or connection error)  
      - **Incident Not Found:** If `$json.number` does not exist (no incident matched)  
      - **Incident Found:** If `$json.number` exists (incident found)  
  - *Input:* Output from ServiceNow node  
  - *Output:* Routes to appropriate response nodes  
  - *Edge Cases:*  
    - Unexpected response structure  
    - Multiple incidents returned (only first is used downstream)  

---

#### 2.3 Slack Response Handling

**Overview:**  
This block sends a formatted response back to Slack depending on the ServiceNow query outcome: detailed incident info, no incident found message, or error notification.

**Nodes Involved:**  
- Send Incident Details to Slack  
- Notify User no Incident was Found  
- Notify User of Error with ServiceNow

**Node Details:**

- **Send Incident Details to Slack**  
  - *Type:* Respond to Webhook  
  - *Role:* Sends a detailed Slack message with incident information.  
  - *Configuration:*  
    - HTTP Response Code: 200  
    - Content-Type: application/json  
    - Response Body: JSON Slack Block Kit formatted message including:  
      - Incident ID, Description, Severity, Caller, Priority, State, Category, Date Opened  
      - A button linking to the incident in ServiceNow using the `sys_id`  
    - Responds publicly (`response_type: in_channel`) so all Slack channel members see the message  
  - *Input:* Incident data from ServiceNow node  
  - *Output:* HTTP response to Slack  
  - *Edge Cases:*  
    - Missing fields in incident data  
    - Slack API rate limits or formatting errors  
  - *Sticky Note:* Explains the Slack response formatting and sending.

- **Notify User no Incident was Found**  
  - *Type:* Respond to Webhook  
  - *Role:* Sends a Slack message notifying that no incident matched the provided ID.  
  - *Configuration:*  
    - HTTP Response Code: 200  
    - Content-Type: application/json  
    - Response Body: Slack Block Kit message with warning emoji and user-friendly text  
  - *Input:* Routed from Switch node when no incident is found  
  - *Output:* HTTP response to Slack  
  - *Edge Cases:* None significant

- **Notify User of Error with ServiceNow**  
  - *Type:* Respond to Webhook  
  - *Role:* Sends a Slack message notifying about a ServiceNow connection or query error.  
  - *Configuration:*  
    - HTTP Response Code: 200  
    - Content-Type: application/json  
    - Response Body: Slack Block Kit message with alert emoji and error notification text  
  - *Input:* Routed from Switch node when ServiceNow returns an error  
  - *Output:* HTTP response to Slack  
  - *Edge Cases:* None significant

---

### 3. Summary Table

| Node Name                        | Node Type              | Functional Role                          | Input Node(s)                  | Output Node(s)                        | Sticky Note                                                                                                      |
|---------------------------------|------------------------|----------------------------------------|-------------------------------|-------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Webhook                         | Webhook                | Receives Slack Slash Command POST      | -                             | Extract Incident ID from Response    | ![Slack](https://uploads.n8n.io/templates/slack.png) This section begins with the `Webhook` node, which listens for incoming Slack Slash Command requests. When triggered, it extracts the incident ID from the request payload using the `Extract Incident ID from Response` node. The incident ID is then passed forward for further processing. This setup allows users to initiate ServiceNow incident lookups directly from Slack. |
| Extract Incident ID from Response | Set                    | Extracts incident ID from Slack payload | Webhook                       | Search For Incident in ServiceNow    | See above                                                                                                       |
| Search For Incident in ServiceNow | ServiceNow             | Queries ServiceNow for incident details | Extract Incident ID from Response | Parse ServiceNow Response             | ![ServiceNow](https://uploads.n8n.io/templates/servicenow.png) In this section, the `Search For Incident in ServiceNow` node queries the ServiceNow platform using the extracted incident ID. If the query returns a valid incident, the details are prepared for the Slack response. If no incident is found, the workflow routes this outcome for a corresponding Slack notification. The `Parse ServiceNow Response` node evaluates the outcome of the ServiceNow query. This ensures accurate and responsive communication with ServiceNow. |
| Parse ServiceNow Response        | Switch                 | Routes workflow based on ServiceNow query result | Search For Incident in ServiceNow | Notify User of Error with ServiceNow, Notify User no Incident was Found, Send Incident Details to Slack | See above                                                                                                       |
| Send Incident Details to Slack  | Respond to Webhook     | Sends detailed incident info to Slack  | Parse ServiceNow Response (Incident Found) | -                                   | ![Slack](https://uploads.n8n.io/templates/webhook.png) Based on the ServiceNow result: - The `Send Incident Details to Slack` node formats and sends detailed incident information to Slack. - The `Notify User no Incident was Found` node sends a user-friendly notification indicating the incident ID was invalid. - The `Notify User of Error with ServiceNow` node alerts the user if the ServiceNow connection fails. This ensures users receive the right response for every scenario, enabling seamless incident management directly from Slack. |
| Notify User no Incident was Found | Respond to Webhook     | Notifies Slack user no incident found  | Parse ServiceNow Response (Incident Not Found) | -                                   | See above                                                                                                       |
| Notify User of Error with ServiceNow | Respond to Webhook     | Notifies Slack user of ServiceNow error | Parse ServiceNow Response (ServiceNow Error) | -                                   | See above                                                                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: Choose a unique path (e.g., `f6ec2074-6c23-410e-ad31-ac1eaf7381ad`)  
   - Response Mode: Response Node  
   - Purpose: Receive Slack Slash Command POST requests.

2. **Create Set Node ("Extract Incident ID from Response")**  
   - Type: Set  
   - Add a string field named `incident_id`  
   - Set value to expression: `{{$json.body.text}}` (extracts incident ID from Slack command text)  
   - Connect Webhook node output to this node.

3. **Create ServiceNow Node ("Search For Incident in ServiceNow")**  
   - Type: ServiceNow  
   - Resource: Incident  
   - Operation: Get All  
   - Authentication: Basic Auth (configure with your ServiceNow credentials)  
   - Query: `number={{ $json.incident_id }}`  
   - Enable "Return Display Values" (sysparm_display_value = true)  
   - On Error: Continue Regular Output  
   - Connect "Extract Incident ID from Response" node output to this node.

4. **Create Switch Node ("Parse ServiceNow Response")**  
   - Type: Switch  
   - Add three outputs with conditions:  
     - **ServiceNow Error:** `$json.error` exists  
     - **Incident Not Found:** `$json.number` does not exist  
     - **Incident Found:** `$json.number` exists  
   - Connect "Search For Incident in ServiceNow" output to this node.

5. **Create Respond to Webhook Node ("Send Incident Details to Slack")**  
   - Type: Respond to Webhook  
   - Response Code: 200  
   - Response Headers: Content-Type: application/json  
   - Respond With: JSON  
   - Response Body: Use Slack Block Kit JSON to format incident details including fields: Incident ID, Description, Severity, Caller, Priority, State, Category, Date Opened, and a button linking to the incident in ServiceNow using `sys_id`.  
   - Connect "Parse ServiceNow Response" output "Incident Found" to this node.

6. **Create Respond to Webhook Node ("Notify User no Incident was Found")**  
   - Type: Respond to Webhook  
   - Response Code: 200  
   - Response Headers: Content-Type: application/json  
   - Respond With: JSON  
   - Response Body: Slack Block Kit message with warning emoji and text: "No incident was found with that ID. Please double check and try again."  
   - Connect "Parse ServiceNow Response" output "Incident Not Found" to this node.

7. **Create Respond to Webhook Node ("Notify User of Error with ServiceNow")**  
   - Type: Respond to Webhook  
   - Response Code: 200  
   - Response Headers: Content-Type: application/json  
   - Respond With: JSON  
   - Response Body: Slack Block Kit message with alert emoji and text: "Issue connecting to ServiceNow. Please investigate in n8n."  
   - Connect "Parse ServiceNow Response" output "ServiceNow Error" to this node.

8. **Credential Setup**  
   - Configure ServiceNow Basic Auth credentials with username and password or token for API access.  
   - Ensure Slack Slash Command is configured to send POST requests to your n8n webhook URL.

9. **Activate Workflow**  
   - Save and activate the workflow in n8n.  
   - Test by invoking the Slack Slash Command with a valid incident ID.

---

### 5. General Notes & Resources

| Note Content                                                                                                          | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| For detailed Slack app setup with Slash Commands and n8n integration, watch this video: [Slack Slash Command Setup](https://youtu.be/z4JuK4qPJ1E) | Slack Slash Command configuration and n8n webhook integration                                      |
| Customize the ServiceNow query in the `Search For Incident in ServiceNow` node to add filters or retrieve additional fields as needed. | Workflow customization for organizational needs                                                    |
| Enhance error handling by adding logging or alerting nodes to notify admins of repeated failures or connectivity issues. | Best practices for production readiness and monitoring                                             |
| Slack response messages use Block Kit formatting for rich, interactive messages including buttons and sections.       | Slack Block Kit documentation: https://api.slack.com/block-kit                                    |
| Ensure your ServiceNow account has API access and permissions to read incident records.                                | ServiceNow API and permissions documentation                                                       |

---

This document provides a complete and detailed reference for understanding, reproducing, and modifying the "Display ServiceNow Incident Details in Slack using Slash Commands" workflow in n8n.