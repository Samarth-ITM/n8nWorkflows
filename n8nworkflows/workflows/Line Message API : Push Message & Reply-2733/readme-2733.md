Line Message API : Push Message & Reply

https://n8nworkflows.xyz/workflows/line-message-api---push-message---reply-2733


# Line Message API : Push Message & Reply

### 1. Workflow Overview

This workflow demonstrates integration with the **LINE Messaging API** to handle two primary use cases:

- **1.1 Replying to Incoming Messages:**  
  When a user sends a message to the LINE bot, the workflow receives the event via a webhook, verifies the event type, and replies to the user using the provided reply token with a confirmation message echoing the received text.

- **1.2 Sending Push Messages to Specific Users:**  
  Independently, the workflow allows manual triggering to send a push message to a specific LINE user by their unique user ID.

The workflow is logically divided into two main blocks:

- **Block 1: Incoming Message Handling and Reply**  
  Nodes involved: Webhook reception → Conditional check → Reply HTTP request

- **Block 2: Manual Push Message Sending**  
  Nodes involved: Manual trigger → Prepare user ID → Push message HTTP request

---

### 2. Block-by-Block Analysis

#### Block 1: Incoming Message Handling and Reply

- **Overview:**  
  This block listens for incoming webhook events from LINE, filters for message events, and replies to the user with a text message echoing their original message.

- **Nodes Involved:**  
  - Webhook from Line Message  
  - If  
  - Line : Reply with token  
  - Sticky Note (explanatory)

- **Node Details:**

  1. **Webhook from Line Message**  
     - **Type:** Webhook  
     - **Role:** Entry point to receive POST requests from LINE Messaging API containing event data.  
     - **Configuration:**  
       - HTTP Method: POST  
       - Path: `638c118e-1c98-4491-b6ff-14e2e75380b6` (unique webhook endpoint)  
     - **Input:** Incoming HTTP POST from LINE platform  
     - **Output:** JSON payload containing event data  
     - **Edge Cases:**  
       - Invalid or malformed webhook payloads  
       - Unauthorized requests (should be filtered by LINE platform)  
       - Network timeouts or dropped requests  
     - **Version:** n8n webhook node v2  

  2. **If**  
     - **Type:** Conditional (If)  
     - **Role:** Checks if the event type in the webhook payload is exactly `"message"`.  
     - **Configuration:**  
       - Condition: `{{ $json.body.events[0].type }} equals "message"`  
       - Case sensitive, strict type validation  
     - **Input:** Output from Webhook node  
     - **Output:**  
       - True branch: Event is a message → proceed to reply  
       - False branch: Ignore or no further action  
     - **Edge Cases:**  
       - Events with no `events` array or empty array  
       - Event types other than "message" (e.g., follow, join)  
       - Expression evaluation errors if JSON structure changes  
     - **Version:** n8n If node v2.2  

  3. **Line : Reply with token**  
     - **Type:** HTTP Request  
     - **Role:** Sends a reply message to the user using LINE’s reply API endpoint.  
     - **Configuration:**  
       - Method: POST  
       - URL: `https://api.line.me/v2/bot/message/reply`  
       - Authentication: Header Auth credential named `Line n8n demo auth` (contains `Authorization: Bearer {line token}`)  
       - JSON Body:  
         ```json
         {
           "replyToken": "{{ $('Webhook from Line Message').item.json.body.events[0].replyToken }}",
           "messages": [
             {
               "type": "text",
               "text": "收到您的訊息 : {{ $('Webhook from Line Message').item.json.body.events[0].message.text }}"
             }
           ]
         }
         ```  
       - Sends back a text message echoing the user's original message text.  
     - **Input:** True branch from If node  
     - **Output:** HTTP response from LINE API (success or error)  
     - **Edge Cases:**  
       - Expired or invalid reply token (reply tokens are short-lived)  
       - Network errors or API rate limits  
       - Malformed JSON or missing fields in the webhook payload  
     - **Version:** HTTP Request node v4.2  

  4. **Sticky Note**  
     - **Type:** Sticky Note  
     - **Role:** Provides explanation about the reply flow and event type filtering.  
     - **Content:**  
       ```
       ## Line Message API Reply

       Received Message from user and reply with same text by using reply token  

       There are many event types. So we need to determine if the type is message.
       ```  
     - **No input/output connections**  

---

#### Block 2: Manual Push Message Sending

- **Overview:**  
  This block allows manual triggering of the workflow to send a push message to a specific LINE user by their user ID.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - Edit Fields (Set node)  
  - Line : Push Message (HTTP Request)  
  - Sticky Note (explanatory)

- **Node Details:**

  1. **When clicking ‘Test workflow’**  
     - **Type:** Manual Trigger  
     - **Role:** Starts the workflow manually for testing purposes.  
     - **Configuration:** No parameters needed.  
     - **Input:** User manually triggers execution  
     - **Output:** Triggers next node  
     - **Edge Cases:** None  

  2. **Edit Fields**  
     - **Type:** Set  
     - **Role:** Defines the LINE user ID (`line_uid`) to which the push message will be sent.  
     - **Configuration:**  
       - Field: `line_uid` = `Uxxxxxxxxxxxx` (placeholder for actual LINE user ID)  
     - **Input:** Manual Trigger output  
     - **Output:** JSON with `line_uid` field  
     - **Edge Cases:**  
       - Invalid or incorrect user ID will cause push message failure  
       - Missing field or empty string  
     - **Version:** Set node v3.4  

  3. **Line : Push Message**  
     - **Type:** HTTP Request  
     - **Role:** Sends a push message to the specified LINE user ID.  
     - **Configuration:**  
       - Method: POST  
       - URL: `https://api.line.me/v2/bot/message/push`  
       - Authentication: Header Auth credential `Line n8n demo auth`  
       - JSON Body:  
         ```json
         {
           "to": "{{ $json.line_uid }}",
           "messages": [
             {
               "type": "text",
               "text": "推播測試"
             }
           ]
         }
         ```  
     - **Input:** Output from Edit Fields node  
     - **Output:** HTTP response from LINE API  
     - **Edge Cases:**  
       - Invalid user ID causes API error  
       - Network issues or API rate limiting  
       - Authentication failure  
     - **Version:** HTTP Request node v4.2  

  4. **Sticky Note1**  
     - **Type:** Sticky Note  
     - **Role:** Explains the push message flow and the need to obtain the LINE user ID first.  
     - **Content:**  
       ```
       ## Line Message API Send Message

       You need to get the Line UID first.
       Every user is differnt.

       If you have the Line UID. Then you can push the message to the User.
       ```  
     - **No input/output connections**  

---

### 3. Summary Table

| Node Name                 | Node Type       | Functional Role                          | Input Node(s)              | Output Node(s)          | Sticky Note                                                                                         |
|---------------------------|-----------------|----------------------------------------|----------------------------|------------------------|---------------------------------------------------------------------------------------------------|
| Webhook from Line Message | Webhook         | Receive incoming LINE webhook events   | -                          | If                     | ## Line Message API Reply<br>Received Message from user and reply with same text by using reply token<br>There are many event types. So we need to determine if the type is message. |
| If                        | If Condition    | Check if event type is "message"       | Webhook from Line Message  | Line : Reply with token |                                                                                                   |
| Line : Reply with token    | HTTP Request    | Reply to user message using reply token| If                         | -                      |                                                                                                   |
| Sticky Note               | Sticky Note     | Explanation for reply message flow     | -                          | -                      | ## Line Message API Reply<br>Received Message from user and reply with same text by using reply token<br>There are many event types. So we need to determine if the type is message. |
| When clicking ‘Test workflow’ | Manual Trigger | Manual start for push message test     | -                          | Edit Fields             | ## Line Message API Send Message<br>You need to get the Line UID first.<br>Every user is differnt.<br>If you have the Line UID. Then you can push the message to the User.            |
| Edit Fields               | Set             | Prepare LINE user ID for push message  | When clicking ‘Test workflow’ | Line : Push Message    |                                                                                                   |
| Line : Push Message       | HTTP Request    | Send push message to LINE user          | Edit Fields                | -                      |                                                                                                   |
| Sticky Note1              | Sticky Note     | Explanation for push message flow      | -                          | -                      | ## Line Message API Send Message<br>You need to get the Line UID first.<br>Every user is differnt.<br>If you have the Line UID. Then you can push the message to the User.            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Header Auth Credential**  
   - Name: `Line n8n demo auth`  
   - Type: Header Auth  
   - Header Name: `Authorization`  
   - Header Value: `Bearer {your LINE channel access token}`  
   - Save credential for use in HTTP Request nodes.

2. **Create Webhook Node: "Webhook from Line Message"**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: Use a unique string, e.g., `638c118e-1c98-4491-b6ff-14e2e75380b6`  
   - This node will receive incoming webhook events from LINE Messaging API.

3. **Create If Node: "If"**  
   - Type: If  
   - Condition:  
     - Expression: `{{ $json.body.events[0].type }}`  
     - Operator: Equals  
     - Value: `message`  
   - Connect input from "Webhook from Line Message" node.

4. **Create HTTP Request Node: "Line : Reply with token"**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.line.me/v2/bot/message/reply`  
   - Authentication: Use the Header Auth credential created in step 1.  
   - Body Content Type: JSON  
   - JSON Body (use expression mode):  
     ```json
     {
       "replyToken": "{{ $('Webhook from Line Message').item.json.body.events[0].replyToken }}",
       "messages": [
         {
           "type": "text",
           "text": "收到您的訊息 : {{ $('Webhook from Line Message').item.json.body.events[0].message.text }}"
         }
       ]
     }
     ```  
   - Connect input from the **True** output of the "If" node.

5. **Create Manual Trigger Node: "When clicking ‘Test workflow’"**  
   - Type: Manual Trigger  
   - No special configuration.

6. **Create Set Node: "Edit Fields"**  
   - Type: Set  
   - Add a field:  
     - Name: `line_uid`  
     - Type: String  
     - Value: `Uxxxxxxxxxxxx` (replace with actual LINE user ID)  
   - Connect input from "When clicking ‘Test workflow’".

7. **Create HTTP Request Node: "Line : Push Message"**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.line.me/v2/bot/message/push`  
   - Authentication: Use the Header Auth credential created in step 1.  
   - Body Content Type: JSON  
   - JSON Body (expression mode):  
     ```json
     {
       "to": "{{ $json.line_uid }}",
       "messages": [
         {
           "type": "text",
           "text": "推播測試"
         }
       ]
     }
     ```  
   - Connect input from "Edit Fields".

8. **Add Sticky Notes (Optional but Recommended)**  
   - Add explanatory sticky notes near the respective blocks to document the logic and usage.

9. **Activate the Workflow**  
   - Save and activate the workflow.  
   - Configure your LINE Messaging API webhook URL to point to the webhook URL generated by the "Webhook from Line Message" node.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                   |
|-----------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| To simplify authentication, create a Header Auth credential in n8n with `Authorization: Bearer {token}` | Credential setup instructions in the workflow description                                        |
| LINE reply tokens are short-lived; reply must be sent promptly after receiving the webhook event.    | LINE Messaging API documentation: https://developers.line.biz/en/reference/messaging-api/#send-reply-message |
| Push messages require the recipient’s LINE user ID, which must be obtained beforehand.               | LINE Messaging API documentation: https://developers.line.biz/en/reference/messaging-api/#send-push-message |
| Workflow storyboard image available: `Line_n8n_demo.png` (referenced in original documentation)      | Visual aid for workflow structure                                                                |

---

This document provides a detailed, structured reference for understanding, reproducing, and maintaining the LINE Messaging API integration workflow in n8n.