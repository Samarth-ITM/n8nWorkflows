Send an SMS/Whatsapp message with Twilio

https://n8nworkflows.xyz/workflows/send-an-sms-whatsapp-message-with-twilio-401


# Send an SMS/Whatsapp message with Twilio

### 1. Workflow Overview

This workflow demonstrates how to send an SMS or WhatsApp message using the Twilio integration in n8n. It is primarily designed for manual execution, allowing a user to trigger the sending of a message on demand. The workflow consists of two logical blocks:

- **1.1 Input Reception:** Manual trigger node to initiate the workflow.
- **1.2 Message Sending:** Twilio node configured to send an SMS or WhatsApp message using Twilio API credentials.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block is responsible for starting the workflow manually. It waits for user interaction to proceed with sending a message.

- **Nodes Involved:**  
  - On clicking 'execute'

- **Node Details:**  

  - **On clicking 'execute'**  
    - Type and technical role: Manual Trigger node; initiates workflow execution on user command.  
    - Configuration choices: No parameters configured; triggers immediately upon manual execution.  
    - Key expressions or variables used: None.  
    - Input and output connections: No input; outputs to the Twilio node.  
    - Version-specific requirements: None.  
    - Edge cases or potential failure types: None, unless user forgets to execute or triggers multiple times unintentionally.  
    - Sub-workflow reference: None.

#### 1.2 Message Sending

- **Overview:**  
  This block handles sending the actual SMS or WhatsApp message via Twilio API. It uses stored Twilio credentials to authenticate and send the message.

- **Nodes Involved:**  
  - Twilio

- **Node Details:**  

  - **Twilio**  
    - Type and technical role: Twilio node; sends SMS or WhatsApp messages through Twilio's API.  
    - Configuration choices: Default configuration; no parameters explicitly set in the provided JSON (likely requires manual configuration for message details before use).  
    - Key expressions or variables used: None defined; message content, recipient, and sender are typically set here but are empty in the given workflow.  
    - Input and output connections: Receives input from the manual trigger node; outputs message sending response (not connected further).  
    - Version-specific requirements: Requires valid Twilio API credentials.  
    - Edge cases or potential failure types: Authentication errors if credentials are invalid or missing; API limit exceeded; invalid phone numbers; network timeouts; message content errors.  
    - Sub-workflow reference: None.

---

### 3. Summary Table

| Node Name           | Node Type            | Functional Role                  | Input Node(s)          | Output Node(s) | Sticky Note                                      |
|---------------------|----------------------|--------------------------------|-----------------------|----------------|-------------------------------------------------|
| On clicking 'execute'| Manual Trigger       | Initiates workflow execution    | —                     | Twilio         |                                                 |
| Twilio              | Twilio node          | Sends SMS/WhatsApp message      | On clicking 'execute'  | —              | ![A workflow with the Twilio node](https://i.imgur.com/hhrzqyR.png) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a new node of type **Manual Trigger**.  
   - Leave all parameters at default.  
   - This node will serve as the entry point to execute the workflow manually.

2. **Create Twilio Node**  
   - Add a new node of type **Twilio**.  
   - Under **Credentials**, select or create new **Twilio API credentials**:  
     - Account SID and Auth Token must be provided (these can be obtained from your Twilio console).  
   - Configure the message parameters:  
     - **Resource:** Choose `Message`.  
     - **Operation:** Choose `Send`.  
     - **From:** Enter your Twilio phone number (must be a verified sender).  
     - **To:** Enter the recipient phone number in E.164 format (e.g., +1234567890).  
     - **Message:** Enter the text message content to send.  
   - If sending WhatsApp messages, use the appropriate Twilio sandbox number or approved WhatsApp sender number as **From**, and prefix recipient with `whatsapp:` (e.g., `whatsapp:+1234567890`).

3. **Connect Nodes**  
   - Connect the output of the Manual Trigger node to the input of the Twilio node.

4. **Save and Activate**  
   - Save the workflow.  
   - Execute manually to test message sending.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                  |
|-------------------------------------------------------------------------------------------------|-------------------------------------------------|
| The screenshot shows a simple example of Twilio node usage in n8n workflows.                     | ![Workflow Screenshot](https://i.imgur.com/hhrzqyR.png) |
| Ensure your Twilio account has sufficient balance and the phone numbers used are verified if needed. | Twilio official docs: https://www.twilio.com/docs/sms |
| For WhatsApp messaging, Twilio WhatsApp sandbox setup is required for testing purposes.          | Twilio WhatsApp sandbox: https://www.twilio.com/whatsapp |

---

This document covers the complete structure, configuration, and considerations for the "Send an SMS/Whatsapp message with Twilio" workflow in n8n. It facilitates understanding, modification, and reproduction without needing the original JSON export.