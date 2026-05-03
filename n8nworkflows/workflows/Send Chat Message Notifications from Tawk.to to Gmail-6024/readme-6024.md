Send Chat Message Notifications from Tawk.to to Gmail

https://n8nworkflows.xyz/workflows/send-chat-message-notifications-from-tawk-to-to-gmail-6024


# Send Chat Message Notifications from Tawk.to to Gmail

### 1. Workflow Overview

This workflow automates the process of forwarding chat message notifications received from the Tawk.to live chat platform to a designated Gmail inbox. It is designed for support teams or customer service agents who want to be immediately alerted via email whenever a new chat message arrives on Tawk.to.

The workflow is logically divided into three blocks:

- **1.1 Input Reception:** Receives incoming chat message data from Tawk.to via a webhook.
- **1.2 Message Formatting:** Extracts and organizes relevant information from the incoming payload.
- **1.3 Email Notification:** Sends a formatted alert email to a specified Gmail address with the chat details.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block listens for incoming HTTP POST requests from Tawk.to containing chat message data. It acts as the entry point to the workflow.

- **Nodes Involved:**  
  - Receive Tawk.to Request

- **Node Details:**

  - **Receive Tawk.to Request**  
    - **Type:** Webhook  
    - **Role:** Listens for incoming chat messages from Tawk.to.  
    - **Configuration:**  
      - HTTP Method: POST  
      - Path: `a4bf95cd-a30a-4ae0-bd2a-6d96e6cca3b4` (unique webhook endpoint path)  
      - No additional options configured.  
    - **Expressions/Variables:** Receives payload in `$json.body` containing chat details.  
    - **Inputs:** None (entry node).  
    - **Outputs:** Sends the raw JSON payload to the next node.  
    - **Version:** 2  
    - **Potential Failures:**  
      - Misconfigured webhook path or method could cause missed requests.  
      - Network or authentication issues on Tawk.to side may prevent delivery.  
      - Payload structure changes could break downstream processing.  
    - **Sub-workflow:** None.

#### 2.2 Message Formatting

- **Overview:**  
  This block extracts pertinent details from the raw webhook payload and prepares a simplified JSON object for use in the email notification.

- **Nodes Involved:**  
  - Format the message

- **Node Details:**

  - **Format the message**  
    - **Type:** Set  
    - **Role:** Maps and assigns specific fields from the incoming payload to new JSON properties for clarity and ease of use.  
    - **Configuration:**  
      - Creates four string fields:  
        - `chat_id` ← `$json.body.chatId`  
        - `visitor_country` ← `$json.body.visitor.country`  
        - `visitor_name` ← `$json.body.visitor.name`  
        - `message` ← `$json.body.message.text`  
    - **Expressions:** Uses expressions to directly map nested JSON properties.  
    - **Inputs:** Receives raw webhook payload.  
    - **Outputs:** Sends structured JSON with selected fields.  
    - **Version:** 3.4  
    - **Potential Failures:**  
      - If the incoming payload lacks expected properties (e.g., `visitor`, `message`), expressions may fail or output empty strings.  
      - Changes in Tawk.to payload structure could require updates.  
    - **Sub-workflow:** None.

#### 2.3 Email Notification

- **Overview:**  
  Sends a notification email via Gmail that summarizes the chat message details, alerting the support team.

- **Nodes Involved:**  
  - Send alert email

- **Node Details:**

  - **Send alert email**  
    - **Type:** Gmail  
    - **Role:** Sends an email with chat details formatted in the body and subject.  
    - **Configuration:**  
      - Recipient email address: *Not specified* (must be set before activation).  
      - Subject: `Support Alert: New Chat from {{ visitor_name }} ({{ visitor_country }})`  
      - Message (text format): Includes chat ID, visitor name and country, message content, and a polite request to respond promptly.  
      - Email type: Plain text.  
      - No additional options.  
      - Requires Gmail OAuth2 credentials configured in n8n.  
    - **Expressions:** Uses placeholders to dynamically insert values from previous node outputs.  
    - **Inputs:** Receives formatted JSON with chat details.  
    - **Outputs:** None (final node).  
    - **Version:** 2.1  
    - **Potential Failures:**  
      - Missing or invalid Gmail credentials will cause authentication errors.  
      - Recipient email not set will cause failure or deliverability issues.  
      - Rate limits or Gmail API errors.  
    - **Sub-workflow:** None.

---

### 3. Summary Table

| Node Name             | Node Type    | Functional Role            | Input Node(s)           | Output Node(s)       | Sticky Note                                   |
|-----------------------|--------------|----------------------------|------------------------|----------------------|-----------------------------------------------|
| Receive Tawk.to Request| Webhook      | Receives incoming chat data| None                   | Format the message   |                                               |
| Format the message     | Set          | Extracts and formats fields| Receive Tawk.to Request | Send alert email     |                                               |
| Send alert email       | Gmail        | Sends email notification   | Format the message      | None                 | Recipient email address must be configured.  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Add a new node of type **Webhook**.  
   - Set HTTP Method to **POST**.  
   - Set the path to a unique identifier, e.g., `a4bf95cd-a30a-4ae0-bd2a-6d96e6cca3b4`.  
   - Save the node.

2. **Create Set Node ("Format the message")**  
   - Add a **Set** node.  
   - Configure four new fields:  
     - `chat_id` (string): Set value to `={{ $json.body.chatId }}`  
     - `visitor_country` (string): Set value to `={{ $json.body.visitor.country }}`  
     - `visitor_name` (string): Set value to `={{ $json.body.visitor.name }}`  
     - `message` (string): Set value to `={{ $json.body.message.text }}`  
   - Connect the output of the Webhook node to this Set node.

3. **Create Gmail Node ("Send alert email")**  
   - Add a **Gmail** node.  
   - Configure credentials with valid Gmail OAuth2 credentials.  
   - Set **Send To** to the desired recipient email address.  
   - Set **Subject** to:  
     `Support Alert: New Chat from {{ $json.visitor_name }} ({{ $json.visitor_country }})`  
   - Set **Message** (email body) as plain text:  
     ```
     Hi Team,

     You have received a new chat message.

     Chat ID: {{$json.chat_id}}
     Visitor Name: {{$json.visitor_name}}
     Visitor Country: {{$json.visitor_country}}

     Message:
     {{$json.message}}

     Please respond promptly.

     Best regards,
     Your Support Bot
     ```
   - Connect the output of the Set node to this Gmail node.

4. **Finalize Workflow**  
   - Ensure all nodes are properly connected as:  
     Webhook → Set → Gmail  
   - Activate the workflow.  
   - Configure the Tawk.to dashboard or webhook settings to send POST requests to the webhook URL provided by n8n for the specified path.

---

### 5. General Notes & Resources

| Note Content                                                               | Context or Link                                  |
|----------------------------------------------------------------------------|-------------------------------------------------|
| This workflow is designed for alerting support teams via Gmail for Tawk.to chats. | Workflow purpose summary.                        |
| Gmail node requires OAuth2 credentials configured in n8n for sending emails. | n8n Gmail node documentation.                    |
| Ensure the recipient email address in the Gmail node is set before activation. | Prevents email sending failures.                 |
| Webhook URL must be correctly configured in Tawk.to to forward chat messages. | Tawk.to webhook setup documentation.             |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow built with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.