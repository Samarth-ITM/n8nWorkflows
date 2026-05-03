Convert Gumroad Sales to Beehiiv Subscribers with Sheets & Telegram Alerts

https://n8nworkflows.xyz/workflows/convert-gumroad-sales-to-beehiiv-subscribers-with-sheets---telegram-alerts-3715


# Convert Gumroad Sales to Beehiiv Subscribers with Sheets & Telegram Alerts

### 1. Workflow Overview

This workflow automates the process of converting new Gumroad buyers into newsletter subscribers on Beehiiv, logs their purchase details into a Google Sheets CRM, and sends a notification message to a Telegram channel. It is designed for creators and marketers who want to seamlessly integrate their Gumroad sales with their newsletter and CRM systems while keeping their team informed via instant messaging.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Triggered by a new sale event from Gumroad.
- **1.2 Newsletter Subscription:** Lists Beehiiv publications and subscribes the new buyer's email to the newsletter.
- **1.3 CRM Logging:** Appends the sale data into a Google Sheets document as a CRM record.
- **1.4 Team Notification:** Sends a formatted notification message to a Telegram channel.
- **1.5 Configuration & Setup Notes:** Sticky notes providing setup instructions and requirements for each integration.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block listens for new sales on Gumroad and triggers the workflow when a sale occurs.

- **Nodes Involved:**  
  - Gumroad Sale Trigger

- **Node Details:**

  - **Gumroad Sale Trigger**  
    - *Type & Role:* Trigger node that listens to Gumroad sales events via webhook.  
    - *Configuration:* Set to resource "sale" to trigger on new sales. Uses Gumroad API credentials with an API key (access token) from Gumroad application settings.  
    - *Expressions/Variables:* Outputs sale data including email, product name, sale timestamp, and buyer country.  
    - *Connections:* Output connects to "List publications" node.  
    - *Version:* v1  
    - *Potential Failures:* Webhook registration failure, invalid API key, network issues, or no sales events received.  
    - *Sub-workflow:* None.

---

#### 1.2 Newsletter Subscription

- **Overview:**  
  This block retrieves Beehiiv publications and subscribes the new Gumroad buyer's email to the first publication found.

- **Nodes Involved:**  
  - List publications  
  - Post subscription

- **Node Details:**

  - **List publications**  
    - *Type & Role:* HTTP Request node to fetch Beehiiv publications via API.  
    - *Configuration:* GET request to `https://api.beehiiv.com/v2/publications`. Uses Beehiiv API key via HTTP Header Authentication.  
    - *Expressions/Variables:* None explicitly; response JSON contains publications array.  
    - *Connections:* Output connects to "Post subscription".  
    - *Version:* 4.2  
    - *Potential Failures:* API key invalid or expired, network issues, API rate limits, or empty publications list.  
    - *Sub-workflow:* None.

  - **Post subscription**  
    - *Type & Role:* HTTP Request node to add subscriber to Beehiiv publication.  
    - *Configuration:* POST request to `https://api.beehiiv.com/v2/publications/{{ $json.data[0].id }}/subscriptions` where `{{ $json.data[0].id }}` is the first publication's ID from the previous node. Body includes the subscriber's email from Gumroad sale trigger. Uses Beehiiv API key via HTTP Header Authentication.  
    - *Expressions/Variables:* Email value from `$('Gumroad Sale Trigger').item.json.email`.  
    - *Connections:* Output connects to "append row in CRM".  
    - *Version:* 4.2  
    - *Potential Failures:* Invalid publication ID, email already subscribed, API errors, network timeouts.  
    - *Sub-workflow:* None.

---

#### 1.3 CRM Logging

- **Overview:**  
  This block appends the sale details into a Google Sheets document for CRM purposes.

- **Nodes Involved:**  
  - append row in CRM

- **Node Details:**

  - **append row in CRM**  
    - *Type & Role:* Google Sheets node appending a new row with sale data.  
    - *Configuration:* Appends to Sheet1 (gid=0) in a specified Google Sheets document. Columns appended: date (sale timestamp), email, country, product name — all extracted from Gumroad sale data. Uses Google Sheets OAuth2 credentials.  
    - *Expressions/Variables:*  
      - `date` = `$('Gumroad Sale Trigger').item.json.sale_timestamp`  
      - `email` = `$('Gumroad Sale Trigger').item.json.email`  
      - `country` = `$('Gumroad Sale Trigger').item.json.ip_country`  
      - `product name` = `$('Gumroad Sale Trigger').item.json.product_name`  
    - *Connections:* Output connects to "Set ChatID".  
    - *Version:* 4.5  
    - *Potential Failures:* Invalid or expired Google credentials, sheet access permissions, malformed data, API rate limits.  
    - *Sub-workflow:* None.

---

#### 1.4 Team Notification

- **Overview:**  
  This block sends a Telegram message to notify the team about the new sale.

- **Nodes Involved:**  
  - Set ChatID  
  - Notify in channel

- **Node Details:**

  - **Set ChatID**  
    - *Type & Role:* Set node to assign the Telegram chat ID or channel username to a variable.  
    - *Configuration:* Assigns a string value `<your chat id>` to variable `telegramChatId`. This ID should be replaced by the user with the actual chat ID or channel username where the bot is admin.  
    - *Expressions/Variables:* None, static assignment.  
    - *Connections:* Output connects to "Notify in channel".  
    - *Version:* 3.4  
    - *Potential Failures:* Missing or incorrect chat ID will cause message sending to fail.  
    - *Sub-workflow:* None.

  - **Notify in channel**  
    - *Type & Role:* Telegram node to send a message to a Telegram channel or chat.  
    - *Configuration:* Sends a formatted message containing product name, email, and country from the Gumroad sale trigger. Uses Telegram Bot credentials (Bot Token). Chat ID is taken from the variable `telegramChatId`. Attribution appended is disabled.  
    - *Expressions/Variables:*  
      - Product: `{{ $('Gumroad Sale Trigger').item.json.product_name }}`  
      - Email: `{{ $('Gumroad Sale Trigger').item.json.email }}`  
      - Country: `{{ $('Gumroad Sale Trigger').item.json.ip_country }}`  
      - Chat ID: `={{ $json.telegramChatId }}`  
    - *Connections:* None (end node).  
    - *Version:* 1.2  
    - *Potential Failures:* Invalid Bot Token, bot not admin in channel, incorrect chat ID, network issues.  
    - *Sub-workflow:* None.

---

#### 1.5 Configuration & Setup Notes

- **Overview:**  
  Sticky Note nodes provide detailed setup instructions and requirements for each integration step.

- **Nodes Involved:**  
  - Sticky Note  
  - Sticky Note1  
  - Sticky Note2  
  - Sticky Note3

- **Node Details:**

  - **Sticky Note**  
    - Content explains how to create a Gumroad application and get API key for the sale trigger.  
    - Positioned near the Gumroad Sale Trigger node.

  - **Sticky Note1**  
    - Content explains Beehiiv account setup, publication creation, and API key generation.  
    - Positioned near the Beehiiv HTTP Request nodes.

  - **Sticky Note2**  
    - Content explains Google Sheets API credentials setup and appending rows for CRM.  
    - Positioned near the Google Sheets node.

  - **Sticky Note3**  
    - Content explains Telegram Bot creation, adding bot to channel as admin, and notification setup.  
    - Positioned near Telegram nodes.

- *No input/output connections; purely informational.*

---

### 3. Summary Table

| Node Name           | Node Type                | Functional Role                         | Input Node(s)           | Output Node(s)         | Sticky Note                                                                                                     |
|---------------------|--------------------------|---------------------------------------|------------------------|-----------------------|-----------------------------------------------------------------------------------------------------------------|
| Gumroad Sale Trigger | gumroadTrigger           | Trigger on new Gumroad sale            | —                      | List publications      | ## Trigger on a new Gumroad sale<br>Requirements and setup instructions provided in sticky note.                |
| List publications    | httpRequest              | Fetch Beehiiv publications             | Gumroad Sale Trigger    | Post subscription      | ## Connection to Beehiiv newsletter<br>Requirements and API key setup instructions provided.                   |
| Post subscription    | httpRequest              | Subscribe buyer to Beehiiv newsletter | List publications       | append row in CRM      | ## Connection to Beehiiv newsletter<br>Requirements and API key setup instructions provided.                   |
| append row in CRM    | googleSheets             | Append sale data to Google Sheets CRM | Post subscription       | Set ChatID             | ## Load into CRM<br>Google Sheets API credentials and append instructions provided.                            |
| Set ChatID           | set                      | Assign Telegram chat ID for messaging | append row in CRM       | Notify in channel      |                                                                                                                 |
| Notify in channel    | telegram                 | Send notification message on Telegram | Set ChatID              | —                     | ## Notify team in Telegram<br>Telegram Bot creation and channel setup instructions provided.                   |
| Sticky Note          | stickyNote               | Setup instructions for Gumroad trigger| —                      | —                     | ## Trigger on a new Gumroad sale<br>Setup notes for Gumroad API key and application.                            |
| Sticky Note1         | stickyNote               | Setup instructions for Beehiiv        | —                      | —                     | ## Connection to Beehiiv newsletter<br>Beehiiv account and API setup notes.                                    |
| Sticky Note2         | stickyNote               | Setup instructions for Google Sheets  | —                      | —                     | ## Load into CRM<br>Google Sheets API and credentials setup instructions.                                      |
| Sticky Note3         | stickyNote               | Setup instructions for Telegram        | —                      | —                     | ## Notify team in Telegram<br>Telegram Bot and channel setup instructions.                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gumroad Sale Trigger Node**  
   - Node Type: Gumroad Trigger  
   - Set Resource to "sale"  
   - Enter Gumroad API credentials (API key from Gumroad application settings)  
   - Position: Left side, start of workflow

2. **Create HTTP Request Node to List Beehiiv Publications**  
   - Node Type: HTTP Request  
   - Method: GET  
   - URL: `https://api.beehiiv.com/v2/publications`  
   - Authentication: HTTP Header Authentication with Beehiiv API key  
   - Connect input from Gumroad Sale Trigger output

3. **Create HTTP Request Node to Post Subscription**  
   - Node Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.beehiiv.com/v2/publications/{{ $json.data[0].id }}/subscriptions`  
   - Body Parameters: JSON with `email` set to `={{ $('Gumroad Sale Trigger').item.json.email }}`  
   - Authentication: HTTP Header Authentication with Beehiiv API key  
   - Connect input from List Publications output

4. **Create Google Sheets Node to Append Row**  
   - Node Type: Google Sheets  
   - Operation: Append  
   - Document ID: Your Google Sheets document ID  
   - Sheet Name: `Sheet1` (or your target sheet)  
   - Columns to append:  
     - `date` = `={{ $('Gumroad Sale Trigger').item.json.sale_timestamp }}`  
     - `email` = `={{ $('Gumroad Sale Trigger').item.json.email }}`  
     - `country` = `={{ $('Gumroad Sale Trigger').item.json.ip_country }}`  
     - `product name` = `={{ $('Gumroad Sale Trigger').item.json.product_name }}`  
   - Credentials: Google Sheets OAuth2 API credentials with access to the sheet  
   - Connect input from Post Subscription output

5. **Create Set Node to Assign Telegram Chat ID**  
   - Node Type: Set  
   - Add assignment: `telegramChatId` (string) = `<your chat id or channel username>`  
   - Connect input from Append Row in CRM output

6. **Create Telegram Node to Send Notification**  
   - Node Type: Telegram  
   - Text:  
     ```
     🔔 New Gumroad sale!
     Product: {{ $('Gumroad Sale Trigger').item.json.product_name }}
     Email: {{ $('Gumroad Sale Trigger').item.json.email }}
     Country: {{ $('Gumroad Sale Trigger').item.json.ip_country }}
     ```  
   - Chat ID: `={{ $json.telegramChatId }}`  
   - Additional Fields: Disable attribution  
   - Credentials: Telegram Bot API token (Bot must be admin in the channel)  
   - Connect input from Set ChatID output

7. **Add Sticky Notes (Optional but Recommended)**  
   - Create sticky notes near relevant nodes with setup instructions for Gumroad, Beehiiv, Google Sheets, and Telegram integrations for ease of maintenance.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                      | Context or Link                                                                                           |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| Workflow turns Gumroad buyers into Beehiiv newsletter subscribers, logs data in Google Sheets, and sends Telegram alerts.                                                                                                        | Workflow purpose                                                                                          |
| Beehiiv API docs and account setup: https://www.beehiiv.com?via=1node-ai                                                                                                                                                         | Beehiiv official site                                                                                      |
| Google Sheets node documentation: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googlesheets/?utm_source=n8n_app&utm_medium=node_settings_modal-credential_link&utm_campaign=n8n-nodes-base.googleSheets | Google Sheets API node documentation                                                                      |
| Telegram Bot creation and channel setup instructions: Add bot as admin to channel to enable messaging.                                                                                                                           | Telegram official bot setup                                                                                |
| Discord community for help: https://discord.gg/eBZH4WHCqd                                                                                                                                                                        | Support community                                                                                          |
| Further optimizations suggested include adding more subscriber data to Beehiiv, customizing Telegram messages, enhancing CRM data, and implementing error handling workflows for retries or logging.                              | Suggestions for workflow enhancement                                                                      |

---

This document fully describes the workflow’s structure, logic, and configuration to allow replication, modification, and troubleshooting by advanced users or automation agents.