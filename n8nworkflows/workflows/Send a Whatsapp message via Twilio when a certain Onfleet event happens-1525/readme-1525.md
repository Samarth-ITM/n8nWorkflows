Send a Whatsapp message via Twilio when a certain Onfleet event happens

https://n8nworkflows.xyz/workflows/send-a-whatsapp-message-via-twilio-when-a-certain-onfleet-event-happens-1525


# Send a Whatsapp message via Twilio when a certain Onfleet event happens

### 1. Workflow Overview

This workflow automates sending a WhatsApp message via Twilio when a specified event occurs in Onfleet, a last-mile delivery management platform. It is designed for businesses needing real-time delivery notifications to customers or support teams via WhatsApp.

**Logical Blocks:**

- **1.1 Onfleet Event Trigger:** Listens for a specific event (default: task creation) in Onfleet using its webhook trigger node.
- **1.2 Twilio WhatsApp Notification:** Sends a WhatsApp message through Twilio API containing a dynamic tracking URL extracted from the Onfleet event data.

---

### 2. Block-by-Block Analysis

#### 1.1 Onfleet Event Trigger

- **Overview:**  
  This block captures the occurrence of a specific event in Onfleet, triggering the workflow when a delivery task is created.

- **Nodes Involved:**  
  - Onfleet Trigger

- **Node Details:**  
  - **Node Name:** Onfleet Trigger  
  - **Type:** Onfleet Trigger Node (Webhook-based event listener)  
  - **Configuration Choices:**  
    - Trigger set to `taskCreated` (fires when a new task/delivery is created in Onfleet)  
    - Webhook ID auto-generated to receive POST requests from Onfleet  
    - No additional filters or fields specified  
  - **Expressions/Variables Used:** None inside this node (raw event data is passed downstream)  
  - **Input Connections:** None (trigger node)  
  - **Output Connections:** Connected directly to the Twilio node  
  - **Version Requirements:** Compatible with n8n version supporting Onfleet Trigger node (v1 used here)  
  - **Potential Failure Points:**  
    - Invalid or expired Onfleet API credentials may prevent event subscription  
    - Network or webhook delivery failures from Onfleet could cause missed triggers  
  - **Sub-workflows:** None  

#### 1.2 Twilio WhatsApp Notification

- **Overview:**  
  Sends a WhatsApp message to a recipient with a tracking URL from the Onfleet task data.

- **Nodes Involved:**  
  - Twilio

- **Node Details:**  
  - **Node Name:** Twilio  
  - **Type:** Twilio Node (Communication API)  
  - **Configuration Choices:**  
    - Sends a WhatsApp message (`toWhatsapp` is enabled)  
    - Message body includes a dynamic expression that extracts the delivery tracking URL from the Onfleet event JSON payload:  
      `"Your delivery is on the way, please visit {{$json["body"]["data"]["task"]["trackingURL"]}} to track your driver's location."`  
    - The recipient phone number is configured dynamically (not explicitly shown in JSON, expected to be set or modified as required)  
  - **Expressions/Variables Used:**  
    - Accesses nested JSON properties to obtain the `trackingURL` from the Onfleet webhook payload  
  - **Input Connections:** Receives output from the Onfleet Trigger node  
  - **Output Connections:** None (end of workflow)  
  - **Version Requirements:** Compatible with n8n version supporting Twilio node v1 or higher  
  - **Potential Failure Points:**  
    - Invalid Twilio credentials or insufficient permissions  
    - Incorrect phone number format or missing recipient number  
    - Twilio API rate limits or network errors  
    - Expression failure if `trackingURL` is missing or malformed in Onfleet payload  
  - **Sub-workflows:** None  

---

### 3. Summary Table

| Node Name       | Node Type                | Functional Role              | Input Node(s)   | Output Node(s) | Sticky Note                                                                                                               |
|-----------------|--------------------------|-----------------------------|-----------------|----------------|---------------------------------------------------------------------------------------------------------------------------|
| Onfleet Trigger | Onfleet Trigger          | Listens to Onfleet events   | None            | Twilio         | Update the Onfleet trigger with your API key from https://onfleet.com/signup; Learn about events at https://support.onfleet.com/hc/en-us/articles/360045763852-Webhooks |
| Twilio          | Twilio Node              | Sends WhatsApp notifications| Onfleet Trigger | None           | Update Twilio node with your credentials; set recipient number dynamically; toggle WhatsApp or SMS mode as needed          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Onfleet Trigger Node**  
   - Add a new node of type "Onfleet Trigger".  
   - Configure credentials by creating or selecting an Onfleet API key credential (register at https://onfleet.com/signup).  
   - Set the trigger event to `taskCreated` or any other desired Onfleet event.  
   - Note: Upon activation, this node generates a webhook URL to receive event callbacks from Onfleet.

2. **Create Twilio Node**  
   - Add a new node of type "Twilio".  
   - Add Twilio credentials (Account SID and Auth Token) via the n8n credential manager.  
   - Configure the node parameters:  
     - Enable `To Whatsapp` toggle to send via WhatsApp.  
     - In the `Message` field, add the expression:  
       `Your delivery is on the way, please visit {{$json["body"]["data"]["task"]["trackingURL"]}} to track your driver's location.`  
     - Set the `To` phone number:  
       - Ideally, configure this dynamically by extracting the recipient’s phone number from the Onfleet webhook JSON payload (e.g., `{{$json["body"]["data"]["task"]["recipients"][0]["phone"]}}` or similar depending on actual payload).  
       - Alternatively, enter a fixed number for testing.

3. **Connect Nodes**  
   - Connect the `Onfleet Trigger` node output to the `Twilio` node input.

4. **Activate Workflow**  
   - Save and activate the workflow to start listening for Onfleet events and sending WhatsApp notifications.

---

### 5. General Notes & Resources

| Note Content                                                                                                           | Context or Link                                                                                          |
|------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| Onfleet API key registration and signup instructions are available at https://onfleet.com/signup                       | Onfleet official website                                                                                 |
| Detailed documentation on Onfleet webhooks (events, payloads, setup) is available at https://support.onfleet.com       | Onfleet Support Webhooks Documentation                                                                   |
| To switch from WhatsApp to SMS messaging in Twilio, toggle the `To Whatsapp` option off in the Twilio node             | Twilio API messaging options                                                                             |
| Ensure recipient phone numbers are in E.164 format with country code for Twilio API to work correctly                   | Twilio number formatting guidelines                                                                      |

---

This documentation provides a thorough understanding of the workflow, enabling users and automation agents to maintain, reproduce, and troubleshoot it effectively.