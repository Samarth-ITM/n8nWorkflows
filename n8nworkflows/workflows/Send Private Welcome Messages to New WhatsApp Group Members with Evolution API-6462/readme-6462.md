Send Private Welcome Messages to New WhatsApp Group Members with Evolution API

https://n8nworkflows.xyz/workflows/send-private-welcome-messages-to-new-whatsapp-group-members-with-evolution-api-6462


# Send Private Welcome Messages to New WhatsApp Group Members with Evolution API

---

### 1. Workflow Overview

This workflow automates sending personalized private welcome messages to new members joining a specific WhatsApp group using the Evolution API. It is designed for WhatsApp Business accounts integrated with Evolution API, allowing group admins to greet newcomers privately.

The workflow is organized into the following logical blocks:

- **1.1 Webhook Reception:** Captures incoming HTTP POST requests from Evolution API notifying about group events (e.g., member joins).
- **1.2 Variable Configuration:** Centralized node to define credentials, group ID, API endpoint, welcome message, and timing variables.
- **1.3 Group Filtering:** Ensures that only events related to the specified target WhatsApp group are processed.
- **1.4 Membership Event Validation:** Confirms that the event corresponds to a new member joining (not leaving or other actions).
- **1.5 Natural Delay:** Introduces a wait time before sending the message to simulate a natural response delay.
- **1.6 Welcome Message Dispatch:** Sends the personalized welcome message via Evolution API HTTP request.

Supporting the main logic are sticky notes providing setup instructions, configuration guidance, and success confirmation.

---

### 2. Block-by-Block Analysis

#### 2.1 Webhook Reception

- **Overview:** Waits for HTTP POST requests from Evolution API notifying about group events, specifically when a member joins or leaves the WhatsApp group.
- **Nodes Involved:**  
  - `Webhook - Receive Group Events`  
  - Sticky Note (Webhook Configuration)

- **Node Details:**

  - **Webhook - Receive Group Events**  
    - Type: Webhook  
    - Role: Entry point for group event notifications from Evolution API.  
    - Configuration:  
      - HTTP Method: POST  
      - Path: `whatsapp-group-welcome` (customizable webhook path)  
      - No additional authentication configured here; security depends on Evolution API webhook setup.  
    - Input: Incoming HTTP POST from Evolution API.  
    - Output: JSON payload containing event details forwarded to the next node.  
    - Edge cases:  
      - Missing or malformed payloads may cause expression failures downstream.  
      - Unauthorized calls are not explicitly blocked here; rely on external webhook security.  
    - Sticky Note: Advises copying this webhook URL into the Evolution API instance.

---

#### 2.2 Variable Configuration

- **Overview:** Centralizes configuration of critical variables such as API keys, group ID, instance name, API URL, welcome message text, and delay duration.
- **Nodes Involved:**  
  - `Set Variables - Configure Here`  
  - Sticky Note (Variable Configuration Instructions)

- **Node Details:**

  - **Set Variables - Configure Here**  
    - Type: Set node  
    - Role: Defines constant variables used throughout the workflow to avoid hardcoding.  
    - Configuration:  
      - `groupId`: Target WhatsApp group ID (e.g., "your_group_id@g.us")  
      - `apiKey`: API key for authenticating with Evolution API  
      - `instanceName`: Evolution API instance identifier  
      - `evolutionApiUrl`: Base URL for Evolution API requests  
      - `welcomeMessage`: Custom welcome message to send to new members  
      - `waitMinutes`: Delay before sending message (default: 1 minute)  
    - Input: Receives event JSON from webhook.  
    - Output: Passes the original event JSON augmented with the variables.  
    - Edge cases:  
      - Variables must be updated manually before deploying; incorrect values cause failures in filtering and message sending.  
    - Sticky Note: Highlights the need to configure all variables here.

---

#### 2.3 Group Filtering

- **Overview:** Filters incoming events to process only those related to the configured WhatsApp group.
- **Nodes Involved:**  
  - `Filter - Check if Target Group`

- **Node Details:**

  - **Filter - Check if Target Group**  
    - Type: Filter  
    - Role: Validates that the event's group ID matches the configured target group ID.  
    - Configuration:  
      - Condition: `$json.body.data.id` equals `{{$node["Set Variables - Configure Here"].json["groupId"]}}`  
      - Case sensitive, strict type validation enabled.  
    - Input: Event JSON with augmented variables.  
    - Output: Routes matching events to the next step; non-matching events are discarded silently.  
    - Edge cases:  
      - If the group ID field is missing or malformed, no match occurs, and processing stops.  
      - Case sensitivity might cause false negatives if IDs vary in case.

---

#### 2.4 Membership Event Validation

- **Overview:** Checks if the event action indicates a new member joining the group.
- **Nodes Involved:**  
  - `If - New Member Joined`

- **Node Details:**

  - **If - New Member Joined**  
    - Type: If node  
    - Role: Confirms that the event action is `"add"`, meaning a member joined.  
    - Configuration:  
      - Condition: `$json.body.data.action` equals `"add"`  
      - Case sensitive, strict type validation.  
    - Input: Filtered event JSON.  
    - Output: Allows only new member join events to proceed.  
    - Edge cases:  
      - Other actions like "remove" (member left) or unknown actions will not trigger message sending.

---

#### 2.5 Natural Delay

- **Overview:** Adds a configurable wait period before sending the welcome message to create a more natural interaction.
- **Nodes Involved:**  
  - `Wait - Natural Delay`

- **Node Details:**

  - **Wait - Natural Delay**  
    - Type: Wait node  
    - Role: Delays workflow execution for a specified number of minutes.  
    - Configuration:  
      - Unit: Minutes  
      - Amount: `{{$node["Set Variables - Configure Here"].json["waitMinutes"]}}` (default 1)  
    - Input: New member join event.  
    - Output: Passes event forward after delay.  
    - Edge cases:  
      - Long delays may cause workflow timeouts depending on n8n instance limits.  
      - Short or zero delays reduce natural feel.

---

#### 2.6 Welcome Message Dispatch

- **Overview:** Sends a private welcome message to the newly joined member using Evolution API.
- **Nodes Involved:**  
  - `Send Welcome Message`  
  - Sticky Note (Success confirmation)

- **Node Details:**

  - **Send Welcome Message**  
    - Type: HTTP Request  
    - Role: Calls Evolution API to send a text message to the new member's WhatsApp number.  
    - Configuration:  
      - HTTP Method: POST  
      - URL: `{{evolutionApiUrl}}/message/sendText/{{instanceName}}` (constructed dynamically)  
      - Headers: `apikey` set to configured API key  
      - Body Parameters:  
        - `number`: First participant in the event (`$json.body.data.participants[0]`) representing the new member's WhatsApp number  
        - `text`: Configured welcome message text  
      - Allows unauthorized SSL certificates (for self-hosted instances)  
      - Sends both headers and body as parameters (not raw JSON)  
    - Input: Event JSON after delay  
    - Output: API response (not further processed in this workflow)  
    - Edge cases:  
      - API key invalid or expired leads to authentication errors  
      - Participant array empty or malformed leads to message not sent  
      - Network or API downtime causes HTTP request failures  
    - Sticky Note: Confirms success of message sending.

---

### 3. Summary Table

| Node Name                   | Node Type       | Functional Role                           | Input Node(s)                 | Output Node(s)                 | Sticky Note                                                                                                     |
|-----------------------------|-----------------|-----------------------------------------|------------------------------|-------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Sticky Note                 | Sticky Note     | Workflow overview and instructions      |                              |                               | ## WhatsApp Group Welcome Message Automation ... [full introductory content with YouTube link]                 |
| Webhook - Receive Group Events | Webhook         | Receives group event notifications      |                              | Set Variables - Configure Here | 📝 **Webhook Configuration** Copy the webhook URL from this node and configure it in your Evolution API instance |
| Sticky Note 1               | Sticky Note     | Configuration instructions               |                              |                               | ⚙️ **Configure all variables here** Update these values with your own credentials and preferences               |
| Set Variables - Configure Here | Set             | Defines credentials and parameters       | Webhook - Receive Group Events | Filter - Check if Target Group |                                                                                                                 |
| Filter - Check if Target Group | Filter          | Filters events for the target group      | Set Variables - Configure Here | If - New Member Joined          |                                                                                                                 |
| If - New Member Joined      | If              | Checks if event is a new member join     | Filter - Check if Target Group | Wait - Natural Delay            |                                                                                                                 |
| Wait - Natural Delay        | Wait            | Adds delay before sending message        | If - New Member Joined          | Send Welcome Message            |                                                                                                                 |
| Send Welcome Message        | HTTP Request    | Sends welcome message via Evolution API | Wait - Natural Delay            |                               | ✅ **Success!** The welcome message has been sent to the new member                                            |
| Sticky Note 2               | Sticky Note     | Webhook setup reminder                    |                              |                               | 📝 **Webhook Configuration** Copy the webhook URL from this node and configure it in your Evolution API instance |
| Sticky Note 3               | Sticky Note     | Success confirmation                      |                              |                               | ✅ **Success!** The welcome message has been sent to the new member                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Name: `Webhook - Receive Group Events`  
   - Type: Webhook (n8n-nodes-base.webhook)  
   - HTTP Method: POST  
   - Path: `whatsapp-group-welcome` (or your preferred path)  
   - Leave other settings default.  
   - This node will receive POST requests from Evolution API whenever group events occur.

2. **Create Set Node for Variables**  
   - Name: `Set Variables - Configure Here`  
   - Type: Set (n8n-nodes-base.set)  
   - Add these variables (all of type string except waitMinutes which is number):  
     - `groupId` = your target group ID (e.g., `your_group_id@g.us`)  
     - `apiKey` = your Evolution API key  
     - `instanceName` = your Evolution API instance name  
     - `evolutionApiUrl` = base URL of your Evolution API (e.g., `https://your-evolution-api.com`)  
     - `welcomeMessage` = your personalized welcome message text  
     - `waitMinutes` = number of minutes to delay before sending (default 1)  
   - Connect the output of the Webhook node to this Set node.

3. **Create Filter Node to Check Group**  
   - Name: `Filter - Check if Target Group`  
   - Type: Filter (n8n-nodes-base.filter)  
   - Add condition:  
     - Check if `$json.body.data.id` equals `{{$node["Set Variables - Configure Here"].json["groupId"]}}`  
     - Use strict type and case sensitive matching  
   - Connect Set node output to this Filter node.

4. **Create If Node to Detect New Member Join**  
   - Name: `If - New Member Joined`  
   - Type: If (n8n-nodes-base.if)  
   - Condition:  
     - `$json.body.data.action` equals `"add"`  
     - Strict type and case sensitive  
   - Connect Filter node’s output (true branch) to this If node.

5. **Create Wait Node for Natural Delay**  
   - Name: `Wait - Natural Delay`  
   - Type: Wait (n8n-nodes-base.wait)  
   - Set Unit: minutes  
   - Set Amount: `{{$node["Set Variables - Configure Here"].json["waitMinutes"]}}`  
   - Connect If node’s true output to this Wait node.

6. **Create HTTP Request Node to Send Welcome Message**  
   - Name: `Send Welcome Message`  
   - Type: HTTP Request (n8n-nodes-base.httpRequest)  
   - Set method: POST  
   - URL: `{{$node["Set Variables - Configure Here"].json["evolutionApiUrl"]}}/message/sendText/{{$node["Set Variables - Configure Here"].json["instanceName"]}}`  
   - Headers: Add header `apikey` with value `{{$node["Set Variables - Configure Here"].json["apiKey"]}}`  
   - Body parameters (form or JSON parameters depending on API):  
     - `number`: `{{$json.body.data.participants[0]}}` (new member's number)  
     - `text`: `{{$node["Set Variables - Configure Here"].json["welcomeMessage"]}}`  
   - Allow unauthorized SSL certificates if needed (for self-hosted Evolution API).  
   - Connect Wait node output to this node.

7. **Connect Workflow**  
   - Connect `Webhook - Receive Group Events` → `Set Variables - Configure Here`  
   - Connect `Set Variables - Configure Here` → `Filter - Check if Target Group`  
   - Connect Filter node’s true output → `If - New Member Joined`  
   - Connect If node’s true output → `Wait - Natural Delay`  
   - Connect Wait node output → `Send Welcome Message`

8. **Configure Evolution API Webhook**  
   - Copy the Webhook URL from the `Webhook - Receive Group Events` node (displayed in n8n UI)  
   - Set this URL as the webhook endpoint in your Evolution API instance for group events.

9. **Test the Workflow**  
   - Have a new member join the target WhatsApp group.  
   - Observe the workflow execution and confirm the welcome message is sent after the configured delay.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                             | Context or Link                                         |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------|
| Workflow automates private welcome messages for new WhatsApp group members using Evolution API. Requires Evolution API instance and WhatsApp Business account with group admin rights.                                               | Workflow purpose                                       |
| YouTube video with setup instructions and demo: https://youtu.be/WO2MJoQqLvo                                                                                                                                                             | Video tutorial link                                    |
| Ensure your Evolution API instance webhook is configured to send group event notifications to this workflow’s webhook URL.                                                                                                            | Webhook setup instruction                              |
| Variables such as API key, group ID, and instance name must be updated before use to match your environment.                                                                                                                            | Critical configuration note                            |
| The workflow assumes the event payload contains `body.data.id` (group ID), `body.data.action` (join/leave), and `body.data.participants` array with the new member's WhatsApp number as first element. Payload structure must match Evolution API. | Payload structure requirement                           |
| Allow unauthorized SSL certificates option in HTTP Request node is enabled for compatibility with self-hosted Evolution API instances using self-signed certificates.                                                                | Security consideration                                 |

---

**Disclaimer:** The text provided is exclusively derived from a workflow automated with n8n, an integration and automation tool. This processing strictly adheres to content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.

---