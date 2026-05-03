Send an SMS using MSG91

https://n8nworkflows.xyz/workflows/send-an-sms-using-msg91-511


# Send an SMS using MSG91

### 1. Workflow Overview

This workflow is designed to send an SMS message using the MSG91 SMS gateway. It is triggered manually and then executes a single action block that sends an SMS through MSG91's API. This workflow is suitable for use cases where a user wants to test or manually send SMS messages via MSG91 within n8n.

Logical blocks included:  
- **1.1 Manual Trigger:** Initiates the workflow on demand.  
- **1.2 SMS Sending:** Sends an SMS via MSG91 API using provided credentials and message parameters.

---

### 2. Block-by-Block Analysis

#### 1.1 Manual Trigger

- **Overview:**  
  This block provides a manual entry point to start the workflow. It waits for the user to initiate execution.

- **Nodes Involved:**  
  - On clicking 'execute'

- **Node Details:**  
  - **Node Name:** On clicking 'execute'  
  - **Type:** Manual Trigger  
  - **Role:** Triggers workflow execution on user command  
  - **Configuration:** No parameters required; simply waits for manual start  
  - **Key expressions/variables:** None  
  - **Input/Output:** No input; output connected to "Msg91" node  
  - **Version Requirements:** Compatible with n8n core version supporting manual trigger node (version 1 or later)  
  - **Potential Failure Modes:** None (manual trigger is straightforward)  
  - **Sub-workflow:** None

#### 1.2 SMS Sending via MSG91

- **Overview:**  
  This block sends an SMS message using the MSG91 node configured with necessary parameters and credentials.

- **Nodes Involved:**  
  - Msg91

- **Node Details:**  
  - **Node Name:** Msg91  
  - **Type:** MSG91 SMS Node  
  - **Role:** Sends SMS messages using MSG91 API  
  - **Configuration Choices:**  
    - **To:** Empty string (user must fill with recipient phone number)  
    - **From:** Empty string (user must fill with sender ID)  
    - **Message:** Empty string (user must enter SMS content)  
  - **Credentials:** MSG91 API credentials must be set up and linked to this node  
  - **Key expressions/variables:** None by default; user can replace empty strings with expressions or static values  
  - **Input:** Receives trigger from manual trigger node  
  - **Output:** Outputs response from MSG91 API about SMS send status  
  - **Version Requirements:** Compatible with MSG91 node version 1 or later  
  - **Potential Failure Modes:**  
    - Authentication failure due to invalid or missing API credentials  
    - Network timeout when calling MSG91 API  
    - Invalid phone number or message causing API rejection  
    - Missing required parameters (To, From, Message) resulting in error  
  - **Sub-workflow:** None

---

### 3. Summary Table

| Node Name           | Node Type          | Functional Role           | Input Node(s)           | Output Node(s) | Sticky Note                        |
|---------------------|--------------------|--------------------------|------------------------|----------------|----------------------------------|
| On clicking 'execute'| Manual Trigger     | Initiates workflow       | None                   | Msg91          |                                  |
| Msg91               | MSG91 SMS Node     | Sends SMS via MSG91 API  | On clicking 'execute'  | None           | Sending an SMS using MSG91        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a new node of type **Manual Trigger**.  
   - This node requires no parameters. It will be the starting point of the workflow.

2. **Add MSG91 Node**  
   - Add a new node of type **MSG91 SMS**.  
   - Configure the following parameters:  
     - **To:** Enter the recipient phone number in international format (e.g., +919999999999).  
     - **From:** Enter the sender ID registered with MSG91 (alphanumeric or numeric sender ID).  
     - **Message:** Enter the SMS content text to send.  
   - Set up the **MSG91 API Credentials** in n8n Credentials section:  
     - Create new MSG91 API credentials with your MSG91 API key.  
     - Link these credentials to the MSG91 node.

3. **Connect Nodes**  
   - Connect the output of the Manual Trigger node to the input of the MSG91 node.

4. **Save and Activate Workflow**  
   - Save the workflow.  
   - Optionally, activate it for scheduled or event-based triggering.  
   - For manual execution, use the "Execute Workflow" button.

5. **Test Sending SMS**  
   - Click "Execute Workflow" manually to trigger the SMS sending.  
   - Monitor the MSG91 node output for success or error messages.

---

### 5. General Notes & Resources

| Note Content                                              | Context or Link                                     |
|-----------------------------------------------------------|----------------------------------------------------|
| MSG91 is a reliable SMS gateway service commonly used for transactional SMS | https://msg91.com                                   |
| Ensure phone numbers are in E.164 international format for MSG91 compatibility | MSG91 API Documentation                            |
| n8n MSG91 node requires valid API credentials to function | n8n Credentials configuration for MSG91           |

---

This document fully describes the "Send an SMS using MSG91" workflow, enabling users to understand, reproduce, and maintain it efficiently.