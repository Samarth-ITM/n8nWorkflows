Forward Email & LinkedIn Message Notifications from Reply.io to Telegram

https://n8nworkflows.xyz/workflows/forward-email---linkedin-message-notifications-from-reply-io-to-telegram-7297


# Forward Email & LinkedIn Message Notifications from Reply.io to Telegram

### 1. Workflow Overview

This n8n workflow automates forwarding Reply.io notifications about email replies and LinkedIn message replies to a Telegram chat. It is designed for sales, outreach, or customer support teams who want real-time alerts on prospect interactions handled via Reply.io.

The workflow consists of two major logical blocks:

- **1.1 Subscription Management Utility:** Nodes that interact with Reply.io API to create and verify webhook subscriptions for email replies and LinkedIn message replies. This prepares Reply.io to send event data to the workflow webhook.
  
- **1.2 Notification Processing:** A webhook node that receives Reply.io event payloads, fetches detailed person information from Reply.io, and sends a formatted notification message to a specified Telegram chat.

---

### 2. Block-by-Block Analysis

#### 2.1 Subscription Management Utility

**Overview:**  
This block manages webhook subscriptions on Reply.io to enable delivery of email and LinkedIn message reply events to the workflow. It includes nodes to create webhooks for these events and to verify existing subscriptions.

**Nodes Involved:**  
- Get subscriptions  
- Create subscription (email)  
- Create subscription (linkedin)  
- Sticky Note (Utility instructions)

**Node Details:**

- **Get subscriptions**  
  - Type: HTTP Request  
  - Role: Retrieves list of current webhook subscriptions from Reply.io.  
  - Configuration: GET request to `https://api.reply.io/api/v2/webhooks` using HTTP header authentication with Reply.io API key.  
  - Input: None (triggered manually or on demand).  
  - Output: JSON list of active subscriptions.  
  - Potential failures: Authentication errors if API key is invalid; network timeouts; API rate limits.  
  - Credentials: HTTP Bearer Token and Header Auth (Reply.io API key).  

- **Create subscription (email)**  
  - Type: HTTP Request  
  - Role: Creates a webhook subscription for the `email_replied` event.  
  - Configuration: POST to `https://api.reply.io/api/v2/webhooks` with JSON body specifying event type, webhook URL placeholder (`<Your webhook URL here>`) and payload fields to include email URL and prospect custom fields.  
  - Input: None (manual trigger).  
  - Output: Subscription creation response.  
  - Requirements: Replace `<Your webhook URL here>` with actual webhook URL of the "Update from Reply.io" node.  
  - Potential failures: Invalid webhook URL; authentication errors; duplicate subscription errors.  
  - Credentials: Same as above.  

- **Create subscription (linkedin)**  
  - Type: HTTP Request  
  - Role: Creates a webhook subscription for the `linkedin_message_replied` event.  
  - Configuration: Same as above but for LinkedIn message event.  
  - Input/Output/Failures: Same as email subscription node.  

- **Sticky Note (Utility instructions)**  
  - Provides step-by-step instructions to set up subscriptions and check active ones using the above nodes.  
  - Emphasizes need to add API key and correct webhook URLs before triggering.  

---

#### 2.2 Notification Processing

**Overview:**  
This block handles inbound webhook calls from Reply.io when an email or LinkedIn message reply event occurs. It enriches the event data by fetching detailed person info from Reply.io and sends a formatted notification message to a Telegram chat.

**Nodes Involved:**  
- Update from Reply.io (Webhook)  
- Get person info from Reply.io (HTTP Request)  
- Send notification to Telegram (Telegram)  
- Sticky Note (Notification processing instructions)

**Node Details:**

- **Update from Reply.io**  
  - Type: Webhook  
  - Role: Entry point receiving POST requests from Reply.io webhook events.  
  - Configuration: Webhook path set to a unique ID; accepts POST method.  
  - Input: Incoming webhook JSON payload from Reply.io containing event details and contact ID.  
  - Output: Passes data downstream.  
  - Potential failures: Invalid webhook URL configuration on Reply.io side; payload format changes; network errors.  
  - Version: Webhook node v2.  

- **Get person info from Reply.io**  
  - Type: HTTP Request  
  - Role: Retrieves detailed person information using the contact ID received in the webhook payload.  
  - Configuration: GET request to `https://api.reply.io/v1/people?id={{ $json.body.contact_id }}` with header auth.  
  - Input: Receives JSON with `contact_id` from upstream webhook node.  
  - Output: Person details including first name, last name, email, LinkedIn profile, etc.  
  - Expressions: Uses `{{$json.body.contact_id}}` to dynamically set query parameter.  
  - Potential failures: Contact ID missing or invalid; authentication failure; API rate limits.  

- **Send notification to Telegram**  
  - Type: Telegram node  
  - Role: Sends a notification message to a specified Telegram chat.  
  - Configuration:  
    - Message text dynamically composed using expressions extracting event type, person info, sequence name, and links from upstream nodes.  
    - Chat ID placeholder `<Your chat ID here>` to be replaced with actual Telegram chat or group ID.  
    - Disables attribution append.  
  - Input: Person info JSON from previous node.  
  - Output: Telegram API response.  
  - Potential failures: Invalid chat ID; Telegram API errors; bot not added to group; network timeouts.  
  - Credentials: Telegram API credentials (Bot token).  

- **Sticky Note (Notification processing instructions)**  
  - Instructions to configure API keys and Telegram credentials.  
  - Reminder to add Telegram bot to group if notifications target a group chat.  

---

### 3. Summary Table

| Node Name                  | Node Type          | Functional Role                         | Input Node(s)         | Output Node(s)            | Sticky Note                                                                                              |
|----------------------------|--------------------|---------------------------------------|-----------------------|---------------------------|---------------------------------------------------------------------------------------------------------|
| Get subscriptions          | HTTP Request       | Retrieve current Reply.io webhook subscriptions | None                  | None                      | Utility for checking active subscriptions                                                              |
| Create subscription (email)| HTTP Request       | Create webhook subscription for email replies | None                  | None                      | Instructions to create subscriptions and set webhook URL                                                |
| Create subscription (linkedin) | HTTP Request   | Create webhook subscription for LinkedIn message replies | None                  | None                      | Instructions to create subscriptions and set webhook URL                                                |
| Sticky Note (Utility nodes) | Sticky Note        | Provides setup instructions for subscriptions | None                  | None                      | Contains detailed subscription creation and verification instructions                                  |
| Update from Reply.io       | Webhook            | Receive Reply.io webhook events       | None                  | Get person info from Reply.io | Instructions to configure webhook and API keys                                                         |
| Get person info from Reply.io | HTTP Request    | Fetch detailed contact info from Reply.io | Update from Reply.io   | Send notification to Telegram |                                                                                                |
| Send notification to Telegram | Telegram          | Send formatted notification to Telegram chat | Get person info from Reply.io | None                      | Instructions for Telegram bot setup and chat ID configuration; reminder to add bot to group if needed  |
| Sticky Note (Notifications processing) | Sticky Note | Instructions for notification processing setup | None                  | None                      | Contains Telegram and Reply.io API key setup instructions                                              |
| Sticky Note1                | Sticky Note        | (Empty content, visual layout purpose) | None                  | None                      |                                                                                                         |
| Sticky Note2                | Sticky Note        | (Empty content, visual layout purpose) | None                  | None                      |                                                                                                         |
| Sticky Note3                | Sticky Note        | Notification processing instructions  | None                  | None                      | See content in section 2.2                                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Get subscriptions" Node**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://api.reply.io/api/v2/webhooks`  
   - Authentication: HTTP Header Auth with your Reply.io API key  
   - Purpose: Retrieve existing webhook subscriptions  
   - No input connections  

2. **Create "Create subscription (email)" Node**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.reply.io/api/v2/webhooks`  
   - Body Type: JSON  
   - Body Content:  
     ```json
     {
       "event": "email_replied",
       "url": "<Your webhook URL here>",
       "payload": {
         "includeEmailUrl": true,
         "includeProspectCustomFields": true
       }
     }
     ```  
   - Authentication: HTTP Header Auth with Reply.io API key  
   - No input connections  

3. **Create "Create subscription (linkedin)" Node**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.reply.io/api/v2/webhooks`  
   - Body Type: JSON  
   - Body Content:  
     ```json
     {
       "event": "linkedin_message_replied",
       "url": "<Your webhook URL here>",
       "payload": {
         "includeEmailUrl": true,
         "includeProspectCustomFields": true
       }
     }
     ```  
   - Authentication: HTTP Header Auth with Reply.io API key  
   - No input connections  

4. **Create "Update from Reply.io" Node**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: Unique webhook path (e.g., autogenerated UUID or custom string)  
   - No input connections  

5. **Create "Get person info from Reply.io" Node**  
   - Type: HTTP Request  
   - Method: GET  
   - URL expression: `https://api.reply.io/v1/people?id={{ $json.body.contact_id }}`  
   - Authentication: HTTP Header Auth with Reply.io API key  
   - Connect "Update from Reply.io" node output to this node's input  

6. **Create "Send notification to Telegram" Node**  
   - Type: Telegram  
   - Chat ID: Replace `<Your chat ID here>` with your Telegram chat or group ID  
   - Text content (use expression editor):  
     ```
     New message from outreach!

     Type: {{ $('Update from Reply.io').item.json.body.event.type }}

     Contact: {{ $json.firstName }} {{ $json.lastName }}
     Email: {{ $json.email }}
     LinkedIn: {{ $json.linkedInProfile }}

     Sequence: {{ $('Update from Reply.io').item.json.body.sequence_fields.name }}

     Check inbox: https://run.reply.io/Dashboard/Material#/inbox
     ```
   - Additional Fields: Disable append attribution  
   - Credentials: Telegram API credentials (Bot token)  
   - Connect output of "Get person info from Reply.io" node here  

7. **Link nodes in sequence:**  
   - "Update from Reply.io" → "Get person info from Reply.io" → "Send notification to Telegram"  

8. **Configure Credentials:**  
   - Add Reply.io API key credentials for HTTP Header Auth in all HTTP Request nodes needing it.  
   - Add Telegram Bot API credentials in Telegram node.  

9. **Configure webhook URL in subscription creation nodes:**  
   - Replace `<Your webhook URL here>` with the full public URL of the webhook node, e.g., `https://your-n8n-instance.com/webhook/<webhook-path>`  

10. **Test setup:**  
    - Run "Create subscription (email)" and "Create subscription (linkedin)" nodes manually to register webhooks in Reply.io.  
    - Confirm subscriptions via "Get subscriptions" node.  
    - Validate Telegram bot is added to your chat or group with proper permissions.  
    - Test Reply.io event triggers to confirm notifications arrive in Telegram.  

---

### 5. General Notes & Resources

| Note Content                                                                                                    | Context or Link                                                                                  |
|----------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| To receive webhook requests from Reply.io, you must create subscriptions via the "Create subscription" nodes. | Sticky Note attached to subscription nodes explains detailed steps.                            |
| For Telegram notifications, add the bot to your chat or group before sending messages.                         | Sticky Note in notification block emphasizes this to avoid delivery failures.                  |
| Telegram message formatting uses n8n expressions combining event data and enriched person info from Reply.io.  | Enables rich, contextual notifications for outreach teams.                                     |
| Replace placeholders `<Your webhook URL here>` and `<Your chat ID here>` with actual values before deployment. | Crucial for proper integration and message delivery.                                           |
| Reply.io API documentation: https://docs.reply.io/reference/webhooks                                          | To understand webhook events and payload customization.                                        |
| Telegram Bot API documentation: https://core.telegram.org/bots/api                                            | For advanced message formatting or bot capabilities.                                           |

---

**Disclaimer:** The provided text and workflow are exclusively from an automated n8n workflow integration. All data handled respects content policies and contains only legal, public information.