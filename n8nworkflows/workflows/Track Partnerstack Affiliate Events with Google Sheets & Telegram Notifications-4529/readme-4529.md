Track Partnerstack Affiliate Events with Google Sheets & Telegram Notifications

https://n8nworkflows.xyz/workflows/track-partnerstack-affiliate-events-with-google-sheets---telegram-notifications-4529


# Track Partnerstack Affiliate Events with Google Sheets & Telegram Notifications

### 1. Workflow Overview

This workflow is designed to track affiliate events from Partnerstack by capturing incoming webhook notifications, logging them into a Google Sheets document, and sending real-time notifications to a Telegram chat. It targets businesses or teams using Partnerstack to monitor affiliate/customer events and want an automated, centralized logging and alerting system.

The workflow is logically divided into three main blocks:

- **1.1 Input Reception:** Receives and captures Partnerstack event data via a webhook.
- **1.2 Data Logging:** Appends the received event data as a new row into a Google Sheets spreadsheet for persistent storage and tracking.
- **1.3 Notification Dispatch:** Sends a formatted Telegram message with event details to a specified chat ID for immediate team awareness.

Supporting these blocks are configuration and setup notes, including setting the Telegram chat ID and instructions for configuring the Partnerstack postback.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
This block receives HTTP POST requests from Partnerstack containing affiliate event data. It acts as the entry point to the workflow.

- **Nodes Involved:**  
  - Webhook

- **Node Details:**  

  - **Webhook**  
    - Type: Webhook Trigger  
    - Role: Listens for incoming POST requests at a specific URL path, capturing Partnerstack event payloads.  
    - Configuration:  
      - HTTP Method: POST  
      - Path: A unique identifier path (`32a160b4-df74-447a-ab12-c98abe411862`) for security and routing purposes.  
      - No additional options configured.  
    - Expressions/Variables:  
      - Receives JSON payload in `body` containing event data such as `event`, `data.customer_email`, `data.company.name`, etc.  
    - Input: External POST request from Partnerstack.  
    - Output: Passes entire JSON payload downstream.  
    - Version: 2 (Webhook node version)  
    - Potential Failures:  
      - HTTP method mismatch if Partnerstack sends non-POST requests.  
      - Malformed JSON or missing expected fields in the payload.  
      - Network connectivity or firewall issues blocking external requests.  
    - No sub-workflow reference.

---

#### 2.2 Data Logging

- **Overview:**  
This block takes the webhook data and appends a new row to a Google Sheets spreadsheet, effectively logging every received event for record keeping and analysis.

- **Nodes Involved:**  
  - Append Row in Sheets

- **Node Details:**  

  - **Append Row in Sheets**  
    - Type: Google Sheets (Append Operation)  
    - Role: Writes a new row to a specified Google Sheet with parsed event details.  
    - Configuration:  
      - Operation: Append  
      - Document: Google Sheets document with ID `1HOI6sdJ4mzclXy1gE5X1nhoxpjXfnVDhj2ka60SP28g`  
      - Sheet: The first sheet (`gid=0`) named "Sheet1"  
      - Columns mapped:  
        - `date`: Extracted from `created_at` timestamp, converted to ISO date string (yyyy-mm-dd)  
        - `tool`: Partnerstack company name (`data.company.name`)  
        - `email`: Customer email (`data.customer_email`)  
        - `eventName`: Event type (`event`)  
        - `customerName`: Customer name (`data.customer_name`)  
      - Mapping Mode: Explicit column-to-field mapping  
      - AttemptToConvertTypes: Disabled (values saved as strings)  
    - Expressions/Variables:  
      - Date conversion: `={{ new Date($json.body.data.created_at).toISOString().split('T')[0] }}`  
      - Values extracted directly from webhook JSON body fields.  
    - Input: JSON payload from Webhook node.  
    - Output: Passes data downstream after appending.  
    - Version: 4.6 (Google Sheets node version)  
    - Potential Failures:  
      - Authentication errors with Google Sheets OAuth2 credentials.  
      - Sheet or document ID incorrect or access revoked.  
      - API rate limits or quota exceeded on Google Sheets API.  
      - Data field missing or null causing unexpected empty columns.  
    - Credential: Google Sheets OAuth2 account with write access.

---

#### 2.3 Notification Dispatch

- **Overview:**  
After logging, this block sets the Telegram chat ID and sends a customized notification message to a Telegram chat with event details.

- **Nodes Involved:**  
  - Set Chat Id  
  - Send Notification

- **Node Details:**  

  - **Set Chat Id**  
    - Type: Set Node  
    - Role: Assigns the Telegram chat ID to a workflow variable for use in the notification message.  
    - Configuration:  
      - Assignments: Creates a string variable `chatId` with a placeholder value `"setyourTelgramChatIdHere"`  
      - This must be replaced by the user’s actual Telegram chat ID.  
    - Input: JSON from Google Sheets node.  
    - Output: Adds `chatId` field to the JSON for downstream use.  
    - Version: 3.4  
    - Potential Failures:  
      - If chat ID is not set or invalid, Telegram API call will fail.  
      - No validation on chat ID format.  

  - **Send Notification**  
    - Type: Telegram node (Send Message)  
    - Role: Sends a Telegram message with formatted affiliate event details to the specified chat.  
    - Configuration:  
      - Text: Multi-line message containing:  
        - Event type  
        - Tool (company name)  
        - Customer name  
        - Customer email  
      - Chat ID: Uses the variable `chatId` set in the previous node.  
      - Additional fields: Attribution disabled to avoid adding "via n8n" in message.  
    - Expressions/Variables:  
      - Text uses expressions like `{{ $('Webhook').item.json.body.event }}` to pull live data from the webhook JSON.  
    - Input: JSON with chat ID and event data.  
    - Output: Sends message and returns Telegram API response.  
    - Version: 1.2  
    - Potential Failures:  
      - Invalid or missing Telegram chat ID.  
      - Telegram API rate limits or downtime.  
      - Network issues causing message send failure.  
    - Credential: Telegram API credentials with valid bot token.

---

#### 2.4 Setup and Instruction Notes

- **Overview:**  
Two sticky note nodes provide user guidance for configuring the workflow and Telegram chat ID.

- **Nodes Involved:**  
  - Sticky Note  
  - Sticky Note1

- **Node Details:**  

  - **Sticky Note**  
    - Type: Sticky Note  
    - Role: Contains instructions to configure Partnerstack postbacks to send events to the n8n webhook URL.  
    - Content:  
      - Steps to create a postback in Partnerstack under "My account > Postbacks"  
      - Paste the webhook URL from n8n  
      - Select events to track  
      - Test sending an event to capture data in n8n  
    - Position: Left side of workflow canvas.

  - **Sticky Note1**  
    - Type: Sticky Note  
    - Role: Instructs user to insert the Telegram chat ID in the "Set Chat Id" node for notifications.  
    - Content: "Insert your Telegram chat id to get notified in a private channel with your team members"  
    - Positioned near the "Set Chat Id" node.

---

### 3. Summary Table

| Node Name           | Node Type            | Functional Role                  | Input Node(s)       | Output Node(s)       | Sticky Note                                                                                       |
|---------------------|----------------------|---------------------------------|---------------------|----------------------|-------------------------------------------------------------------------------------------------|
| Webhook             | Webhook Trigger      | Receive Partnerstack events     | External POST       | Append Row in Sheets  | See "Set up instructions" sticky note for Partnerstack postback setup                           |
| Append Row in Sheets | Google Sheets        | Log event data in spreadsheet   | Webhook             | Set Chat Id          |                                                                                                 |
| Set Chat Id         | Set Node             | Assign Telegram chat ID         | Append Row in Sheets| Send Notification    | "Insert your Telegram chat id to get notified in a private channel with your team members"      |
| Send Notification   | Telegram             | Send Telegram alert message     | Set Chat Id         | None                 |                                                                                                 |
| Sticky Note         | Sticky Note          | Instructions for Partnerstack   | None                | None                 | "Go into Partnerstack > My account >  Postbacks > Create a postback..."                          |
| Sticky Note1        | Sticky Note          | Instructions for Telegram chat  | None                | None                 | "Insert your Telegram chat id to get notified in a private channel with your team members"      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Webhook Node:**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: Use a unique identifier (e.g., `32a160b4-df74-447a-ab12-c98abe411862`)  
   - Purpose: Receive incoming Partnerstack webhook events.

2. **Add a Google Sheets Node:**  
   - Type: Google Sheets  
   - Operation: Append  
   - Document ID: `1HOI6sdJ4mzclXy1gE5X1nhoxpjXfnVDhj2ka60SP28g` (replace with your own sheet ID)  
   - Sheet Name: Use the first sheet or specify by GID `gid=0`  
   - Columns Mapping:  
     - `date` = Convert `created_at` timestamp from webhook payload to ISO date string: `={{ new Date($json.body.data.created_at).toISOString().split('T')[0] }}`  
     - `tool` = `{{$json.body.data.company.name}}`  
     - `email` = `{{$json.body.data.customer_email}}`  
     - `eventName` = `{{$json.body.event}}`  
     - `customerName` = `{{$json.body.data.customer_name}}`  
   - Credentials: Connect with Google Sheets OAuth2 account with write access.

3. **Connect Webhook → Google Sheets Node.**

4. **Add a Set Node:**  
   - Type: Set  
   - Create a new string variable `chatId`  
   - Value: Your Telegram chat ID as a string (replace `"setyourTelgramChatIdHere"` with your actual chat ID).  
   - Connect Google Sheets Node → Set Node.

5. **Add a Telegram Node:**  
   - Type: Telegram  
   - Operation: Send Message  
   - Text:  
     ```
     New Partnerstack event

     Event: {{ $('Webhook').item.json.body.event }}
     Tool: {{ $('Webhook').item.json.body.data.company.name }}
     Customer: {{ $('Webhook').item.json.body.data.customer_name }}
     Email: {{ $('Webhook').item.json.body.data.customer_email }}
     ```  
   - Chat ID: `={{ $json.chatId }}` (use the chatId variable from the Set node)  
   - Disable "append attribution" to avoid "via n8n" message.  
   - Credentials: Telegram API credentials (bot token).  
   - Connect Set Node → Telegram Node.

6. **Activate the workflow.**

7. **Setup External Configuration:**  
   - In Partnerstack, go to **My Account > Postbacks**.  
   - Create a new postback, paste the n8n webhook URL (e.g., `https://your-n8n-instance/webhook/32a160b4-df74-447a-ab12-c98abe411862`).  
   - Select events to track.  
   - Send a test event from Partnerstack to confirm data reception.

---

### 5. General Notes & Resources

| Note Content                                                                                                    | Context or Link                                                                                     |
|----------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| To capture Partnerstack events, configure postbacks under "My Account > Postbacks" in Partnerstack UI.         | Sticky Note in workflow with setup instructions.                                                  |
| Insert your Telegram chat ID in the "Set Chat Id" node to receive notifications in a private or group channel.  | Sticky Note1 content near "Set Chat Id" node.                                                     |
| Make sure your Google Sheets OAuth2 credentials have both read and write permissions on the target spreadsheet. | Required for "Append Row in Sheets" node to function correctly.                                   |
| The webhook listens for POST requests only; ensure Partnerstack sends POST method requests to the webhook URL. | Webhook node configuration and possible failure points.                                          |
| Telegram API rate limits may apply; consider handling retries or failures if Telegram messages do not deliver.| Possible edge case for the Telegram node.                                                         |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This process fully complies with applicable content policies and contains no illegal, offensive, or protected elements. All data processed are legal and public.