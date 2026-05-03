Send Daily Mailchimp Subscriber Reports to Slack

https://n8nworkflows.xyz/workflows/send-daily-mailchimp-subscriber-reports-to-slack-7603


# Send Daily Mailchimp Subscriber Reports to Slack

### 1. Workflow Overview

This workflow automates the daily reporting of Mailchimp subscriber counts by sending a summary message to a specified Slack channel. It is designed for marketing and growth teams to receive timely updates on audience growth without needing to log in to Mailchimp. The workflow consists of three logical blocks:

- **1.1 Scheduled Trigger:** Initiates the workflow daily at a configured time (default 9:00 AM).
- **1.2 Mailchimp Data Retrieval:** Fetches the total number of subscribers from a specified Mailchimp audience list.
- **1.3 Slack Notification:** Posts a formatted message containing the subscriber count to a designated Slack channel.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
  This block triggers the entire workflow once per day at a scheduled time.

- **Nodes Involved:**  
  - Start: Daily at 09:00

- **Node Details:**  
  - **Node Name:** Start: Daily at 09:00  
  - **Type:** Cron Trigger  
  - **Technical Role:** Initiates the workflow execution on a time schedule.  
  - **Configuration:**  
    - Triggers daily at 09:00 AM (hour set to 9, no minutes specified).  
  - **Key Expressions/Variables:** None.  
  - **Input Connections:** None (trigger node).  
  - **Output Connections:** Connected to "Mailchimp: Get Subscribers" node.  
  - **Version Requirements:** Compatible with n8n version supporting Cron Trigger node v1.  
  - **Potential Failures:**  
    - Cron misconfiguration (e.g., invalid time settings).  
    - Workflow not enabled, resulting in no trigger.  
  - **Sub-Workflow:** None.

#### 1.2 Mailchimp Data Retrieval

- **Overview:**  
  This block fetches all subscribers from a specific Mailchimp audience list to retrieve the total subscriber count.

- **Nodes Involved:**  
  - Mailchimp: Get Subscribers

- **Node Details:**  
  - **Node Name:** Mailchimp: Get Subscribers  
  - **Type:** Mailchimp Node  
  - **Technical Role:** Retrieves subscriber data from Mailchimp.  
  - **Configuration:**  
    - Operation set to "getAll" subscribers.  
    - Requires Mailchimp credentials linked to the account holding the audience list.  
    - The audience list ID must be specified in the credentials or node parameters (implied setup).  
  - **Key Expressions/Variables:** None on input; outputs subscriber data including a `_total` field representing total subscribers.  
  - **Input Connections:** From "Start: Daily at 09:00".  
  - **Output Connections:** To "Daily Mailchimp Report Message".  
  - **Version Requirements:** Compatible with Mailchimp node v1.  
  - **Potential Failures:**  
    - Authentication errors if Mailchimp credentials expire or are invalid.  
    - Network issues or API rate limits from Mailchimp.  
    - Missing or incorrect audience list ID leading to empty or error responses.  
  - **Sub-Workflow:** None.

#### 1.3 Slack Notification

- **Overview:**  
  This block formats and sends a Slack message summarizing the Mailchimp subscriber count.

- **Nodes Involved:**  
  - Daily Mailchimp Report Message

- **Node Details:**  
  - **Node Name:** Daily Mailchimp Report Message  
  - **Type:** Slack Node  
  - **Technical Role:** Posts a message to a Slack channel.  
  - **Configuration:**  
    - Text message template:  
      `📊 Daily Mailchimp Report:|Total Subscribers: {{$json["_total"]}}`  
      This expression dynamically inserts the total subscriber count fetched from the previous node.  
    - Target Slack channel specified in credentials (e.g., `#marketing`).  
    - Uses Slack API credentials configured with OAuth2 or token-based authentication.  
  - **Key Expressions/Variables:**  
    - `{{$json["_total"]}}` extracts the total subscriber count from the incoming JSON.  
  - **Input Connections:** From "Mailchimp: Get Subscribers".  
  - **Output Connections:** None (final node).  
  - **Version Requirements:** Slack node v2.3 or higher for webhookId and message formatting support.  
  - **Potential Failures:**  
    - Slack authentication errors if token is invalid or expired.  
    - Slack API rate limits or connectivity issues.  
    - Formatting errors if the `_total` field is missing or malformed.  
  - **Sub-Workflow:** None.

---

### 3. Summary Table

| Node Name                     | Node Type        | Functional Role            | Input Node(s)            | Output Node(s)                | Sticky Note                                                                                                                                                                                                                                                                                                                                                   |
|-------------------------------|------------------|----------------------------|--------------------------|------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Start: Daily at 09:00          | Cron Trigger     | Scheduled Workflow Trigger | None                     | Mailchimp: Get Subscribers    |                                                                                                                                                                                                                                                                                                                                                                |
| Mailchimp: Get Subscribers     | Mailchimp Node   | Retrieve Mailchimp Subscribers | Start: Daily at 09:00    | Daily Mailchimp Report Message |                                                                                                                                                                                                                                                                                                                                                                |
| Daily Mailchimp Report Message | Slack Node       | Post Subscriber Count to Slack | Mailchimp: Get Subscribers | None                         |                                                                                                                                                                                                                                                                                                                                                                |
| Sticky Note                   | Sticky Note      | Documentation/Instructions | None                     | None                         | # Daily Slack Summary of Mailchimp Subscribers  \n\n## What this workflow does\nThis workflow sends a **daily Slack report** of your Mailchimp subscriber count.  \nIt helps marketing and growth teams stay up to date without opening Mailchimp.\n\n## How it works\n1. **Cron Trigger** runs once per day (default: 9:00 AM).  \n2. **Mailchimp node** fetches the total number of subscribers in a list.  \n3. **Slack node** posts a message with the subscriber count into your chosen Slack channel.\n\n## Setup\n- **Mailchimp:** Connect your Mailchimp account and replace `{{MAILCHIMP_LIST_ID}}` with the ID of your audience list.  \n- **Slack:** Connect your Slack account and replace `{{SLACK_CHANNEL}}` with the target channel (e.g. `#marketing`).  \n- Adjust the Cron schedule if you prefer another time. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add a Cron Trigger node:**  
   - Name: `Start: Daily at 09:00`  
   - Set trigger time to daily at 09:00 AM (hour: 9, minute: 0).  
   - No credentials needed.  

3. **Add a Mailchimp node:**  
   - Name: `Mailchimp: Get Subscribers`  
   - Operation: `getAll` to retrieve all subscribers.  
   - Connect input from `Start: Daily at 09:00`.  
   - Configure Mailchimp credentials:  
     - Set up OAuth2 or API key credentials with Mailchimp.  
     - Ensure the audience/list ID is specified in the node parameters or credentials (replace placeholder `{{MAILCHIMP_LIST_ID}}` with your actual audience ID).  

4. **Add a Slack node:**  
   - Name: `Daily Mailchimp Report Message`  
   - Operation: Send message to channel.  
   - Connect input from `Mailchimp: Get Subscribers`.  
   - Configure Slack credentials:  
     - Use OAuth2 token or Slack app credentials.  
     - Specify the target Slack channel in credentials or node parameters (replace `{{SLACK_CHANNEL}}` with your channel, e.g., `#marketing`).  
   - Set message text to:  
     ```
     📊 Daily Mailchimp Report:|Total Subscribers: {{$json["_total"]}}
     ```  
     This expression dynamically inserts the subscriber count.  

5. **Connect the nodes:**  
   - `Start: Daily at 09:00` → `Mailchimp: Get Subscribers` → `Daily Mailchimp Report Message`.  

6. **Save and activate the workflow.**

7. **Optional:** Add a Sticky Note with the workflow purpose and setup instructions for team reference.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                  | Context or Link                          |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------|
| This workflow is designed to improve marketing team efficiency by automating daily subscriber reports without manual Mailchimp login. Adjust the cron schedule to fit your preferred reporting time (e.g., 8:00 AM).                                                                                                                                             | Workflow scheduling best practices      |
| Replace placeholders `{{MAILCHIMP_LIST_ID}}` and `{{SLACK_CHANNEL}}` with your actual Mailchimp audience list ID and Slack channel name respectively.                                                                                                                                                                                                       | Setup instructions                      |
| Slack message formatting uses Slack's Block Kit simple text formatting. For advanced messages, consider using Blocks JSON formatting in the Slack node.                                                                                                                                                                                                      | Slack API documentation: https://api.slack.com/block-kit |
| Mailchimp API rate limits may apply; for large lists or frequent queries, monitor API usage to avoid throttling.                                                                                                                                                                                                                                            | Mailchimp API docs: https://mailchimp.com/developer/ |
| For secure credential handling, use n8n's built-in credential manager. Rotate API keys or tokens periodically for security.                                                                                                                                                                                                                                  | n8n Credential Management               |

---

**Disclaimer:** The provided content originates exclusively from an n8n automated workflow and adheres strictly to applicable content policies. It contains no illegal, offensive, or protected elements. All data processed is lawful and public.