Multi-channel Website Downtime Alerts with UptimeRobot, Slack, WhatsApp & Notion

https://n8nworkflows.xyz/workflows/multi-channel-website-downtime-alerts-with-uptimerobot--slack--whatsapp---notion-6645


# Multi-channel Website Downtime Alerts with UptimeRobot, Slack, WhatsApp & Notion

### 1. Workflow Overview

This workflow automates multi-channel alerting for website downtime events detected by UptimeRobot. Its primary purpose is to notify the relevant teams and stakeholders immediately via Slack, WhatsApp, and email, and to create a task in Notion for incident tracking and resolution. The workflow is ideal for IT operations, DevOps teams, or website administrators who need fast, multi-platform awareness and task management upon website outages.

The workflow logic is grouped into three main blocks:

- **1.1 Downtime Detection Input**  
  Receives webhook alerts from UptimeRobot when a monitored website is down.

- **1.2 Multi-Channel Alert Dispatch**  
  Sends downtime notifications through Slack, WhatsApp, and email channels simultaneously.

- **1.3 Incident Task Creation**  
  Creates a task in Notion tagging the responsible engineer to track incident resolution.

---

### 2. Block-by-Block Analysis

#### 1.1 Downtime Detection Input

- **Overview:**  
  This block listens for incoming webhook alerts from UptimeRobot indicating that a monitored website is down. It acts as the trigger point for the entire workflow.

- **Nodes Involved:**  
  - UptimeRobot Webhook Trigger

- **Node Details:**

  - **Node Name:** UptimeRobot Webhook Trigger  
    - **Type:** Webhook Trigger  
    - **Technical Role:** Entry point triggered by UptimeRobot’s POST webhook calls on downtime alerts.  
    - **Configuration:**  
      - HTTP Method: POST  
      - Path: `/uptime-alert`  
      - Response Data: Sends back `{ "status": "success" }` to acknowledge receipt  
    - **Key Expressions:** None beyond URL path and response.  
    - **Input Connections:** None (trigger node)  
    - **Output Connections:** Feeds three parallel nodes: Send Slack Alert, Send Email Alert, Send WhatsApp Message  
    - **Version Requirements:** Compatible with n8n webhook v1 nodes.  
    - **Potential Failures:**  
      - Webhook not receiving data due to network issues or incorrect UptimeRobot configuration.  
      - Malformed webhook payloads causing downstream expression errors.  
    - **Sub-Workflow:** None

#### 1.2 Multi-Channel Alert Dispatch

- **Overview:**  
  This block sends out immediate downtime notifications through three communication channels: Slack, WhatsApp, and email. It runs these in parallel to ensure rapid multi-channel alerting.

- **Nodes Involved:**  
  - Send Slack Alert  
  - Send Email Alert  
  - Send message (WhatsApp)

- **Node Details:**

  - **Node Name:** Send Slack Alert  
    - **Type:** Slack node (Message Sender)  
    - **Technical Role:** Posts a formatted alert message to the Slack channel "incidents".  
    - **Configuration:**  
      - Channel: `incidents`  
      - Text: 🚨 Website Down Alert: {{monitorURL}} is down at {{alertTime}}  
        (Uses expressions referencing `UptimeRobot Webhook Trigger` JSON data)  
      - Credentials: Slack API OAuth2 credentials linked to a Slack workspace  
    - **Inputs:** Receives from UptimeRobot Webhook Trigger  
    - **Outputs:** Sends output to Create Notion Task node  
    - **Potential Failures:**  
      - Slack API authentication/authorization errors  
      - Channel not found or invalid channel name  
      - Expression evaluation errors if webhook payload missing expected fields  

  - **Node Name:** Send Email Alert  
    - **Type:** Email Send node (SMTP)  
    - **Technical Role:** Sends an email notification about the downtime to the team.  
    - **Configuration:**  
      - To: `team@company.com`  
      - From: `admin@gmail.com`  
      - Subject: "Website Down Alert"  
      - Text body with website URL and alert time using expressions  
      - SMTP credentials configured for mail server access  
    - **Inputs:** Receives from UptimeRobot Webhook Trigger  
    - **Outputs:** Sends output to Create Notion Task node  
    - **Potential Failures:**  
      - SMTP authentication failure or connection timeout  
      - Invalid email addresses  
      - Expression evaluation errors  

  - **Node Name:** Send message (WhatsApp)  
    - **Type:** WhatsApp node (Message Sender)  
    - **Technical Role:** Sends a WhatsApp message to a specified recipient number with downtime alert.  
    - **Configuration:**  
      - Operation: send  
      - Recipient phone number: `+91998765456789789`  
      - Phone number ID: `+9198765676567` (likely the sender or business number ID)  
      - Text body with website URL and alert time expressions  
      - Credentials: WhatsApp API credentials configured  
    - **Inputs:** Receives from UptimeRobot Webhook Trigger  
    - **Outputs:** Sends output to Create Notion Task node  
    - **Potential Failures:**  
      - WhatsApp API authentication or quota limits  
      - Invalid phone number format or unregistered recipient  
      - Expression errors from missing webhook data  

#### 1.3 Incident Task Creation

- **Overview:**  
  This block creates a task entry in Notion to track the downtime incident, tagging the responsible engineer for follow-up.

- **Nodes Involved:**  
  - Create Notion Task

- **Node Details:**

  - **Node Name:** Create Notion Task  
    - **Type:** Notion node (Create page/task)  
    - **Technical Role:** Creates a new page/task in a specified Notion database or page with the downtime alert title.  
    - **Configuration:**  
      - Page Title: "🚨 Website Down Alert: {{monitorURL}} is down at {{alertTime}}"  
      - Page ID: Static Notion page/database ID where tasks are created (masked as `asdnmkco9876ytghjm`)  
      - Tags or mentions: Responsible engineer tag (example: 'engineer-john') is implied but not explicitly shown in parameters  
      - Credentials: Notion API OAuth2 credentials configured  
    - **Inputs:** Receives from all three alert nodes (Slack, Email, WhatsApp) converging here (all three connect to this node)  
    - **Outputs:** None (end of workflow)  
    - **Potential Failures:**  
      - Notion API rate limiting or authentication errors  
      - Incorrect Page ID or insufficient permissions  
      - Expression evaluation errors if webhook data is incomplete  

---

### 3. Summary Table

| Node Name                 | Node Type          | Functional Role                          | Input Node(s)                 | Output Node(s)          | Sticky Note                                                                                              |
|---------------------------|--------------------|----------------------------------------|------------------------------|-------------------------|--------------------------------------------------------------------------------------------------------|
| UptimeRobot Webhook Trigger | Webhook Trigger    | Receive downtime alerts from UptimeRobot | None                         | Send Slack Alert, Send Email Alert, Send message | This node triggers the workflow when UptimeRobot sends a webhook alert for a 'down' status, indicating the website is offline. |
| Send Slack Alert           | Slack              | Send Slack alert message about downtime | UptimeRobot Webhook Trigger  | Create Notion Task      | This node sends a Slack message to the 'incidents' channel with the down website URL and alert timestamp. |
| Send Email Alert           | Email Send         | Send email alert about downtime         | UptimeRobot Webhook Trigger  | Create Notion Task      | This node sends an email to the team with the down website URL and alert timestamp.                     |
| Send message              | WhatsApp           | Send WhatsApp alert about downtime      | UptimeRobot Webhook Trigger  | Create Notion Task      |                                                                                                        |
| Create Notion Task         | Notion             | Create a Notion task for incident tracking | Send Slack Alert, Send Email Alert, Send message | None                    | This node creates a Notion task with the website URL in the title and tags the responsible engineer (e.g., 'engineer-john'). |
| Sticky Note               | Sticky Note        | Documentation on system architecture    | None                         | None                    | ## System Architecture - Downtime Detection Pipeline (UptimeRobot Webhook Trigger) - Alert Generation Flow (Slack, WhatsApp, Email) - Task Management (Create Notion Task) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Trigger Node**  
   - Type: Webhook  
   - Name: `UptimeRobot Webhook Trigger`  
   - HTTP Method: `POST`  
   - Path: `uptime-alert`  
   - Response: JSON `{ "status": "success" }`  
   - This node will receive the downtime alert POST from UptimeRobot.

2. **Create Slack Node**  
   - Type: Slack  
   - Name: `Send Slack Alert`  
   - Channel: `incidents`  
   - Text:  
     ```
     🚨 Website Down Alert: {{$node["UptimeRobot Webhook Trigger"].json["monitorURL"]}} is down at {{$node["UptimeRobot Webhook Trigger"].json["alertTime"]}}
     ```  
   - Credentials: Configure Slack API OAuth2 credentials with permission to post messages in the `incidents` channel.

3. **Create Email Send Node**  
   - Type: Email Send  
   - Name: `Send Email Alert`  
   - To Email: `team@company.com`  
   - From Email: `admin@gmail.com`  
   - Subject: `Website Down Alert`  
   - Text:  
     ```
     🚨 The website {{$node["UptimeRobot Webhook Trigger"].json["monitorURL"]}} is down as of {{$node["UptimeRobot Webhook Trigger"].json["alertTime"]}}. Please investigate.
     ```  
   - Credentials: Setup SMTP credentials for the sending email.

4. **Create WhatsApp Node**  
   - Type: WhatsApp  
   - Name: `Send message`  
   - Operation: `send`  
   - Recipient Phone Number: `+91998765456789789` (replace with actual)  
   - Phone Number ID: `+9198765676567` (replace with your WhatsApp Business number ID)  
   - Text Body:  
     ```
     🚨 Website Down Alert: {{$node["UptimeRobot Webhook Trigger"].json["monitorURL"]}} is down at {{$node["UptimeRobot Webhook Trigger"].json["alertTime"]}}
     ```  
   - Credentials: Configure WhatsApp API credentials.

5. **Create Notion Node**  
   - Type: Notion  
   - Name: `Create Notion Task`  
   - Page ID: Set the Notion page/database ID where incident tasks are created (e.g., `asdnmkco9876ytghjm`)  
   - Title:  
     ```
     🚨 Website Down Alert: {{$node["UptimeRobot Webhook Trigger"].json["monitorURL"]}} is down at {{$node["UptimeRobot Webhook Trigger"].json["alertTime"]}}
     ```  
   - Credentials: Configure Notion API OAuth2 credentials with permissions to create pages/tasks.

6. **Connect Nodes**  
   - Connect `UptimeRobot Webhook Trigger` to `Send Slack Alert`  
   - Connect `UptimeRobot Webhook Trigger` to `Send Email Alert`  
   - Connect `UptimeRobot Webhook Trigger` to `Send message` (WhatsApp)  
   - Connect `Send Slack Alert` to `Create Notion Task`  
   - Connect `Send Email Alert` to `Create Notion Task`  
   - Connect `Send message` (WhatsApp) to `Create Notion Task`

7. **Optional: Add Sticky Note**  
   - Create a Sticky Note node with the content describing the system architecture and workflow logic for documentation purposes.

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                                                      |
|----------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| System Architecture: Downtime detection triggers alerts via Slack, WhatsApp, and Email, followed by Notion task creation. | Documentation sticky note within the workflow.                                                     |
| Ensure UptimeRobot webhook is set to POST to your n8n webhook URL at `/uptime-alert` to trigger the workflow correctly. | UptimeRobot documentation: https://uptimerobot.com/                                                |
| Slack API credentials require permission to post messages in the target channel (`incidents`).                       | Slack API documentation: https://api.slack.com/messaging/sending                                    |
| WhatsApp Business API setup is required with valid Phone Number ID and recipient number format.                      | WhatsApp Business API docs: https://developers.facebook.com/docs/whatsapp/business-api             |
| Notion API token must have access rights to the target workspace and page/database for task creation.                 | Notion API reference: https://developers.notion.com/reference/post-page                            |

---

This comprehensive reference enables understanding, modification, troubleshooting, and full reproduction of the multi-channel website downtime alerting workflow based on UptimeRobot events.