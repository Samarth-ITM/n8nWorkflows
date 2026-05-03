Create ticket on specific customer messages in Telegram

https://n8nworkflows.xyz/workflows/create-ticket-on-specific-customer-messages-in-telegram-368


# Create ticket on specific customer messages in Telegram

### 1. Workflow Overview

This workflow automates customer support ticket creation and tracking based on specific keywords in messages received via a Telegram support channel. When a customer sends a message, the workflow analyzes the content for keywords like "refund" or "complaint." It then creates a corresponding ticket in Freshdesk with appropriate tags, sends an acknowledgment message back to the customer on Telegram, and creates tracking items on Monday.com boards.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures incoming Telegram messages.
- **1.2 Keyword Evaluation:** Checks message content for keywords to determine ticket type.
- **1.3 Ticket Creation in Freshdesk:** Creates tickets with tags in Freshdesk.
- **1.4 Customer Notification on Telegram:** Sends a tailored response message to the customer.
- **1.5 Tracking Item Creation in Monday.com:** Creates items on Monday.com boards for ticket tracking.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Listens for incoming messages on a Telegram support channel to initiate the workflow.

- **Nodes Involved:**  
  - Telegram Trigger

- **Node Details:**  
  - **Node Name:** Telegram Trigger  
  - **Type:** Telegram Trigger node (event listener)  
  - **Configuration:** Listens for the "message" update type from Telegram.  
  - **Key Expressions:** Accesses incoming message text and chat ID from `{{$json["message"]["text"]}}` and `{{$json["message"]["chat"]["id"]}}`.  
  - **Input Connections:** None (trigger node).  
  - **Output Connections:** Connects to IF1 node for keyword evaluation.  
  - **Potential Failures:** Telegram API connectivity issues, invalid credentials, or no incoming messages.  
  - **Version Requirements:** n8n version supporting Telegram Trigger node v1.  

#### 2.2 Keyword Evaluation

- **Overview:**  
  Checks if the incoming Telegram message contains the keyword "refund" to decide the ticket creation path.

- **Nodes Involved:**  
  - IF1

- **Node Details:**  
  - **Node Name:** IF1  
  - **Type:** If node (conditional branching)  
  - **Configuration:** Checks if the message text contains the substring "refund".  
  - **Key Expressions:** `{{$node["Telegram Trigger"].json["message"]["text"]}}` used for string containment check.  
  - **Input Connections:** From Telegram Trigger node.  
  - **Output Connections:**  
    - True branch → Freshdesk node (refund ticket).  
    - False branch → Freshdesk1 node (complaint ticket).  
  - **Potential Failures:** Expression evaluation errors if message text is missing or malformed.  
  - **Version Requirements:** Standard n8n If node v1.  

#### 2.3 Ticket Creation in Freshdesk

- **Overview:**  
  Creates support tickets in Freshdesk tagged appropriately based on the keyword detected.

- **Nodes Involved:**  
  - Freshdesk (refund tag)  
  - Freshdesk1 (complaint tag)

- **Node Details:**  
  - **Node Name:** Freshdesk  
  - **Type:** Freshdesk node (create ticket)  
  - **Configuration:**  
    - Tags: "refund"  
    - Subject: Uses the message text from IF1 node's JSON (`{{$node["IF1"].json["message"]["text"]}}`)  
    - Requester identification via email; value empty (requires setup)  
  - **Input Connections:** True branch from IF1.  
  - **Output Connections:** Connects to Telegram node to send refund acknowledgment.  
  - **Potential Failures:** Authentication with Freshdesk API, empty or invalid requester identification, API rate limits.  

  - **Node Name:** Freshdesk1  
  - **Type:** Freshdesk node (create ticket)  
  - **Configuration:**  
    - Tags: "complaint"  
    - Subject: Same as above, from IF1 node JSON  
    - Requester identification setup as above  
  - **Input Connections:** False branch from IF1.  
  - **Output Connections:** Connects to Telegram1 node to send complaint acknowledgment.  
  - **Potential Failures:** Same as Freshdesk node.  

#### 2.4 Customer Notification on Telegram

- **Overview:**  
  Sends automated acknowledgment messages to the customer on Telegram, tailored according to ticket type.

- **Nodes Involved:**  
  - Telegram  
  - Telegram1

- **Node Details:**  
  - **Node Name:** Telegram  
  - **Type:** Telegram node (send message)  
  - **Configuration:**  
    - Text: "Hi, thanks for sending this. We will review your request for refund as soon as possible 💶 💵 💷"  
    - Chat ID: Extracted from original Telegram message (`{{$node["Telegram Trigger"].json["message"]["chat"]["id"]}}`)  
  - **Input Connections:** From Freshdesk node (refund ticket).  
  - **Output Connections:** Connects to Monday.com node for tracking.  
  - **Potential Failures:** Telegram API errors, invalid chat ID, or message sending failures.  

  - **Node Name:** Telegram1  
  - **Type:** Telegram node (send message)  
  - **Configuration:**  
    - Text: "Hi, thanks for sending this. We will review your complaint as soon as possible 📬 ☀️ ✅"  
    - Chat ID: same as above  
  - **Input Connections:** From Freshdesk1 node (complaint ticket).  
  - **Output Connections:** Connects to Monday.com1 node for tracking.  
  - **Potential Failures:** Same as Telegram node.  

#### 2.5 Tracking Item Creation in Monday.com

- **Overview:**  
  Creates tracking items on two different groups within a Monday.com board to monitor the tickets.

- **Nodes Involved:**  
  - Monday.com  
  - Monday.com1

- **Node Details:**  
  - **Node Name:** Monday.com  
  - **Type:** Monday.com node (create board item)  
  - **Configuration:**  
    - Board ID: "565971708"  
    - Group ID: "new_group"  
    - Item Name: Uses Freshdesk ticket subject (`{{$node["Freshdesk"].json["subject"]}}`)  
  - **Input Connections:** From Telegram node (refund path).  
  - **Output Connections:** None (end node).  
  - **Potential Failures:** API authentication issues, invalid board/group IDs, rate limits.  

  - **Node Name:** Monday.com1  
  - **Type:** Monday.com node (create board item)  
  - **Configuration:**  
    - Board ID: "565971708"  
    - Group ID: "topics"  
    - Item Name: Uses original Telegram message text (`{{$node["Telegram Trigger"].json["message"]["text"]}}`)  
  - **Input Connections:** From Telegram1 node (complaint path).  
  - **Output Connections:** None (end node).  
  - **Potential Failures:** Same as Monday.com node.  

---

### 3. Summary Table

| Node Name       | Node Type                | Functional Role                      | Input Node(s)         | Output Node(s)           | Sticky Note                                                                                      |
|-----------------|--------------------------|------------------------------------|-----------------------|--------------------------|------------------------------------------------------------------------------------------------|
| Telegram Trigger| Telegram Trigger         | Receives incoming Telegram messages| -                     | IF1                      |                                                                                                |
| IF1             | If                       | Evaluates message for "refund" keyword | Telegram Trigger       | Freshdesk (true), Freshdesk1 (false) |                                                                                                |
| Freshdesk       | Freshdesk                | Creates refund ticket in Freshdesk | IF1 (true)            | Telegram                  |                                                                                                |
| Freshdesk1      | Freshdesk                | Creates complaint ticket in Freshdesk | IF1 (false)           | Telegram1                 |                                                                                                |
| Telegram        | Telegram                 | Sends refund acknowledgment message| Freshdesk             | Monday.com                |                                                                                                |
| Telegram1       | Telegram                 | Sends complaint acknowledgment message | Freshdesk1            | Monday.com1               |                                                                                                |
| Monday.com      | Monday.com               | Creates tracking item for refund tickets | Telegram               | -                        |                                                                                                |
| Monday.com1     | Monday.com               | Creates tracking item for complaint tickets | Telegram1              | -                        |                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node:**  
   - Set "Updates" to listen for "message".  
   - Connect your Telegram API credentials under "Credentials".  
   - Position this as the start node.

2. **Add an If node (named IF1):**  
   - Under "Conditions" → "String", configure:  
     - Value 1: Expression → `{{$node["Telegram Trigger"].json["message"]["text"]}}`  
     - Operation: "contains"  
     - Value 2: "refund"  
   - Connect Telegram Trigger node output to IF1 input.

3. **Add Freshdesk node (named Freshdesk):**  
   - Set to create a ticket with:  
     - Tags: "refund"  
     - Subject: Expression → `{{$node["IF1"].json["message"]["text"]}}`  
     - Requester: select "email" and provide the identification value (configure appropriately).  
   - Connect IF1 "true" output to this node.  
   - Configure Freshdesk API credentials.

4. **Add Freshdesk node (named Freshdesk1):**  
   - Similar to the above but:  
     - Tags: "complaint"  
     - Subject: Expression same as above  
     - Requester settings same as above.  
   - Connect IF1 "false" output to this node.  
   - Use same Freshdesk credentials.

5. **Add Telegram node (named Telegram):**  
   - Set message text:  
     "Hi, thanks for sending this. We will review your request for refund as soon as possible 💶 💵 💷"  
   - Chat ID: Expression → `{{$node["Telegram Trigger"].json["message"]["chat"]["id"]}}`  
   - Connect Freshdesk output to Telegram input.  
   - Use Telegram API credentials.

6. **Add Telegram node (named Telegram1):**  
   - Set message text:  
     "Hi, thanks for sending this. We will review your complaint as soon as possible 📬 ☀️ ✅"  
   - Chat ID: same expression as above.  
   - Connect Freshdesk1 output to Telegram1 input.  
   - Use Telegram API credentials.

7. **Add Monday.com node (named Monday.com):**  
   - Create item on board ID "565971708", group ID "new_group".  
   - Item name: Expression → `{{$node["Freshdesk"].json["subject"]}}`  
   - Connect Telegram output to Monday.com input.  
   - Configure Monday.com API credentials.

8. **Add Monday.com node (named Monday.com1):**  
   - Create item on board ID "565971708", group ID "topics".  
   - Item name: Expression → `{{$node["Telegram Trigger"].json["message"]["text"]}}`  
   - Connect Telegram1 output to Monday.com1 input.  
   - Use same Monday.com credentials.

9. **Validate all credentials and test flow:**  
   - Ensure Telegram API credentials have bot permissions.  
   - Ensure Freshdesk API key and requester identification values are valid.  
   - Ensure Monday.com API token has write access for the specified board and groups.

---

### 5. General Notes & Resources

| Note Content                                                                                               | Context or Link                                                        |
|------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------|
| The workflow uses tags ("refund" and "complaint") to categorize tickets in Freshdesk for better filtering. | Freshdesk ticket management best practice.                           |
| Monday.com board ID "565971708" and group IDs "new_group" and "topics" must exist prior to execution.      | Monday.com board setup requirement.                                  |
| Telegram chat ID is dynamically extracted from incoming messages to ensure responses are sent to the sender.| Telegram API messaging standard.                                     |
| Ensure requester identification (email) is configured properly in Freshdesk nodes to associate tickets with users.| Freshdesk API integration detail.                                    |

---

This document fully describes the workflow titled "Create ticket on specific customer messages in Telegram" and provides all necessary information to understand, reproduce, and troubleshoot it.