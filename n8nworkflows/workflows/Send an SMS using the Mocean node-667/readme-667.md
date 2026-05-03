Send an SMS using the Mocean node

https://n8nworkflows.xyz/workflows/send-an-sms-using-the-mocean-node-667


# Send an SMS using the Mocean node

### 1. Workflow Overview

This workflow is designed to send an SMS message using the Mocean node in n8n. It targets use cases where a user wants to manually trigger the sending of an SMS through the Mocean API, for example for notifications, alerts, or testing SMS delivery. The workflow consists of two logical blocks:

- **1.1 Manual Trigger Input:** A node to manually initiate the workflow execution.
- **1.2 SMS Sending via Mocean:** A node that sends an SMS message using the Mocean API with configurable recipient, sender, and message content.

---

### 2. Block-by-Block Analysis

#### 1.1 Manual Trigger Input

- **Overview:**  
  This block acts as the entry point for the workflow. It allows manual execution of the workflow by a user in the n8n editor or via the API.

- **Nodes Involved:**  
  - On clicking 'execute'

- **Node Details:**

  - **Node Name:** On clicking 'execute'  
  - **Type:** Manual Trigger (`n8n-nodes-base.manualTrigger`)  
  - **Technical Role:** Starts the workflow execution on user command, without requiring external input or schedule.  
  - **Configuration:** No special parameters configured; it simply triggers execution when clicked.  
  - **Expressions/Variables:** None.  
  - **Inputs:** None (trigger node).  
  - **Outputs:** Connected to the Mocean node to pass execution forward.  
  - **Version Requirements:** Compatible with n8n version 0.120+ (standard feature).  
  - **Potential Failures:** None expected; manual trigger is straightforward.  
  - **Sub-workflow Reference:** None.

#### 1.2 SMS Sending via Mocean

- **Overview:**  
  This block sends an SMS message using the Mocean API. It requires configuration of the recipient number, sender ID, and message text. It leverages n8n's Mocean node with API credentials.

- **Nodes Involved:**  
  - Mocean

- **Node Details:**

  - **Node Name:** Mocean  
  - **Type:** Mocean SMS Node (`n8n-nodes-base.mocean`)  
  - **Technical Role:** Sends SMS messages by calling the Mocean API with provided parameters.  
  - **Configuration Choices:**  
    - `to`: The recipient phone number (currently empty, must be set before execution).  
    - `from`: The sender ID or phone number (currently empty).  
    - `message`: The SMS content text (currently empty).  
  - **Expressions/Variables:** None configured; all parameters are static and empty by default.  
  - **Inputs:** Receives trigger from the Manual Trigger node.  
  - **Outputs:** Returns Mocean API response after sending SMS (not connected further in this workflow).  
  - **Credentials:** Uses Mocean API credentials named "mocean" which must be configured in n8n beforehand.  
  - **Version Requirements:** Requires n8n version supporting the Mocean node (available since approximately v0.120).  
  - **Potential Failure Types:**  
    - Authentication errors due to invalid or missing Mocean credentials.  
    - Invalid phone number format or missing recipient number.  
    - Message content empty or exceeding length limits.  
    - Network or API timeout errors.  
  - **Sub-workflow Reference:** None.

---

### 3. Summary Table

| Node Name             | Node Type               | Functional Role           | Input Node(s)           | Output Node(s) | Sticky Note |
|-----------------------|-------------------------|--------------------------|------------------------|----------------|-------------|
| On clicking 'execute' | Manual Trigger          | Start workflow execution | None                   | Mocean         |             |
| Mocean                | Mocean SMS Node         | Send SMS via Mocean API  | On clicking 'execute'  | None           |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a new node of type "Manual Trigger" (`n8n-nodes-base.manualTrigger`).  
   - No special parameters required. This node will allow manual execution.

2. **Create Mocean Node**  
   - Add a new node of type "Mocean" (`n8n-nodes-base.mocean`).  
   - Configure the following parameters:  
     - **To:** Enter the recipient phone number in international format (e.g., +1234567890).  
     - **From:** Enter the sender ID or phone number registered with Mocean.  
     - **Message:** Enter the SMS message text to be sent.  
   - Assign Mocean API credentials:  
     - Go to Credentials in n8n and add new Mocean API credentials. Provide API key and secret as required by Mocean.  
     - Select the created credential in the node configuration.

3. **Connect Nodes**  
   - Connect the output of the Manual Trigger node to the input of the Mocean node.

4. **Save and Execute**  
   - Save the workflow.  
   - Click "Execute Workflow" manually to send the SMS.

---

### 5. General Notes & Resources

| Note Content                                              | Context or Link                            |
|-----------------------------------------------------------|-------------------------------------------|
| Ensure Mocean API credentials are correctly configured to avoid authentication errors. | Mocean API Documentation: https://www.moceanapi.com/docs |
| Phone numbers should be in international format for compatibility with Mocean SMS sending. | Mocean SMS Format Requirements             |
| Test with small message content to avoid exceeding SMS length limits. | SMS Character Limits                        |

---

This documentation provides a thorough understanding of the minimal SMS sending workflow using n8n's Mocean node, enabling reproduction, modification, and troubleshooting.