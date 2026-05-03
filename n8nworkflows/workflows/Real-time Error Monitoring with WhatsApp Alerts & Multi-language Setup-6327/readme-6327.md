Real-time Error Monitoring with WhatsApp Alerts & Multi-language Setup

https://n8nworkflows.xyz/workflows/real-time-error-monitoring-with-whatsapp-alerts---multi-language-setup-6327


# Real-time Error Monitoring with WhatsApp Alerts & Multi-language Setup

### 1. Workflow Overview

This workflow, named **"ErrorFlow WhatsApp Alarm"**, is designed to provide **real-time error monitoring** for other n8n workflows by sending automated WhatsApp alerts whenever an error occurs. It acts as a centralized **Error Workflow** that can be selected in the settings of any other workflow to catch and report their runtime errors.

The workflow includes the following logical blocks:

- **1.1 Error Detection:** Listens for any errors triggered by linked workflows.
- **1.2 WhatsApp Alert Sending:** Sends a formatted text message via WhatsApp Business Cloud API to notify a specified recipient about the error details.
- **1.3 Setup & Usage Instructions:** Provides multilingual, detailed setup instructions and usage guidelines to ensure correct integration and configuration.

---

### 2. Block-by-Block Analysis

#### 2.1 Error Detection Block

- **Overview:**  
  This block acts as the entry point for error events from other workflows. It automatically triggers execution when an error occurs in a workflow configured to use this as its Error Workflow.

- **Nodes Involved:**  
  - Error Trigger

- **Node Details:**  
  - **Error Trigger**  
    - *Type & Role:* n8n native trigger node specialized in capturing errors from other workflows.  
    - *Configuration:* No parameters required; listens passively for error events.  
    - *Expressions/Variables:* Passes entire error execution context downstream as JSON.  
    - *Connections:* Output is connected to the **Send message** node.  
    - *Version Requirements:* Version 1, standard in n8n.  
    - *Edge Cases / Failures:* Will not trigger if this workflow is not set as the Error Workflow in other workflows; does not capture errors internal to itself.  
    - *Sub-workflow:* None.

#### 2.2 WhatsApp Alert Sending Block

- **Overview:**  
  Upon receiving an error event, this block sends a WhatsApp text message to a predefined recipient, relaying key error information such as workflow name, error message, and last executed node.

- **Nodes Involved:**  
  - Send message

- **Node Details:**  
  - **Send message**  
    - *Type & Role:* WhatsApp node using WhatsApp Business Cloud API to send messages programmatically.  
    - *Configuration:*  
      - Operation: `Send`  
      - Resource: `Message` (implicit)  
      - Sender phone number ID configured (`phoneNumberId`: "708748555659713")  
      - Recipient phone number is a placeholder `"!!!!!!!! Insert your number !!!!!!!!"` to be replaced with actual international format number.  
      - Message type: Text  
      - Text body uses expressions to dynamically include error details:  
        ```
        Error on WorkFlow: {{ $json.workflow.name }}  
        Message: {{ $json.execution.error.message }}  
        lastNodeExecuted: {{ $json.execution.lastNodeExecuted }}
        ```  
    - *Expressions:* Uses n8n expressions to access the error context passed from the Error Trigger node.  
    - *Connections:* Input from **Error Trigger** node; no outputs.  
    - *Credentials:* Requires WhatsApp Business Cloud API credentials, referenced as "WhatsApp account".  
    - *Version Requirements:* Version 1; requires WhatsApp Business Cloud API integration configured in n8n.  
    - *Edge Cases / Failures:*  
      - Failure if WhatsApp credentials are invalid or API limits exceeded.  
      - Failure if recipient phone number is not set or incorrectly formatted.  
      - Timeout or network issues may prevent message delivery.  
    - *Sub-workflow:* None.

#### 2.3 Setup & Usage Instructions Block

- **Overview:**  
  Provides detailed multilingual instructions for users to properly configure the workflow, WhatsApp credentials, and usage guidelines to ensure effective error alerting.

- **Nodes Involved:**  
  - SETUP INSTRUCTIONS (sticky note)  
  - Sticky Note (additional brief note)

- **Node Details:**  
  - **SETUP INSTRUCTIONS (sticky note)**  
    - *Type & Role:* Documentation embedded within the workflow as a sticky note for user guidance.  
    - *Content Highlights:*  
      - Multilingual (English, Spanish, German, French, Russian) setup steps.  
      - Instructions emphasize that this workflow must remain inactive.  
      - Step-by-step WhatsApp configuration details for the Send Message node.  
      - How to select this workflow as the Error Workflow in other workflows' settings.  
    - *Position:* Placed prominently for visibility.  
    - *Edge Cases:* None (informational only).  

  - **Sticky Note (colored, small)**  
    - *Type & Role:* Quick reminder note emphasizing configuration of this workflow as Error Workflow in other workflows.  
    - *Content:* "IMPORTANT: In your other workflows → open **Settings** → select this as **Error Workflow**."  
    - *Color:* Blue (color code 7) to draw attention.  
    - *Edge Cases:* None (informational only).

---

### 3. Summary Table

| Node Name          | Node Type               | Functional Role                 | Input Node(s)    | Output Node(s) | Sticky Note                                                                                                                                    |
|--------------------|-------------------------|--------------------------------|------------------|----------------|------------------------------------------------------------------------------------------------------------------------------------------------|
| Error Trigger      | Error Trigger           | Detects errors from other workflows | None             | Send message   |                                                                                                                                                 |
| Send message       | WhatsApp                | Sends WhatsApp alert with error details | Error Trigger    | None           |                                                                                                                                                 |
| SETUP INSTRUCTIONS | Sticky Note             | Multilingual setup and usage instructions | None             | None           | # 🚨 ERROR-MONITORING FLOW — WHATSAPP SETUP GUIDE  (Full multilingual setup instructions including WhatsApp credentials and usage steps)      |
| Sticky Note        | Sticky Note             | Reminder to set this as Error Workflow in other workflows | None             | None           | IMPORTANT: In your other workflows → open **Settings** → select this as **Error Workflow**                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named `"ErrorFlow WhatsApp Alarm"`.

2. **Add an "Error Trigger" node**:  
   - Node Type: `Error Trigger` (n8n native).  
   - No parameters to configure.  
   - This node will automatically catch errors from workflows that select this as their Error Workflow.

3. **Add a "WhatsApp" node to send messages**:  
   - Node Type: `WhatsApp` (WhatsApp Business Cloud API integration).  
   - Credentials: Create or select existing WhatsApp Business Cloud API credentials in n8n.  
   - Parameters:  
     - Operation: `Send`  
     - Phone Number ID: Enter your WhatsApp Business Cloud phone number ID (e.g., `"708748555659713"`).  
     - Recipient Phone Number: Replace the placeholder `"!!!!!!!! Insert your number !!!!!!!!"` with the actual recipient number in international format (e.g., `+34612345678`).  
     - Text Body: Use the expression editor to input the following template:  
       ```
       Error on WorkFlow: {{ $json.workflow.name }}  
       Message: {{ $json.execution.error.message }}  
       lastNodeExecuted: {{ $json.execution.lastNodeExecuted }}
       ```
   - Connect the output of the **Error Trigger** node to the input of this WhatsApp node.

4. **Add a Sticky Note node** for multilingual setup instructions:  
   - Node Type: `Sticky Note`.  
   - Content: Copy the multilingual setup instructions including WhatsApp setup, usage notes, and the requirement that this workflow remains inactive.  
   - Adjust size and position for readability.

5. **Add a smaller Sticky Note node** as a reminder:  
   - Content: "IMPORTANT: In your other workflows → open **Settings** → select this as **Error Workflow**."  
   - Optional: Set color to blue for emphasis.

6. **Save the workflow**, but **do not activate it**.  
   - This workflow must remain inactive and only be triggered by errors from other workflows.

7. **In any workflow you want to monitor for errors:**  
   - Open workflow settings.  
   - Set this `"ErrorFlow WhatsApp Alarm"` workflow as the **Error Workflow**.  
   - Save those workflows.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                            | Context or Link                                                                                                    |
|---------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------|
| This workflow is designed to be a centralized error alerting mechanism using WhatsApp Business Cloud API for real-time notifications.                   | Core purpose                                                                                                      |
| The workflow must remain inactive and is only triggered via the Error Trigger node from other workflows.                                               | Usage instruction                                                                                                 |
| WhatsApp phone number and API credentials must be correctly configured to ensure delivery of error messages.                                           | Credential setup                                                                                                  |
| Multilingual instructions facilitate wide adoption and correct setup across different language speakers.                                              | Included in SETUP INSTRUCTIONS sticky note                                                                        |
| Reminder: In other workflows → Settings → select this workflow as the Error Workflow to enable centralized error reporting.                            | Sticky Note reminder                                                                                              |
| For WhatsApp Business Cloud API setup, refer to official WhatsApp and n8n documentation for credential creation and phone number ID retrieval.        | External resource suggestion                                                                                      |
| The error message template can be customized by modifying the expression in the WhatsApp node’s Text Body field.                                       | Customization hint                                                                                                 |

---

**Disclaimer:** The provided text is exclusively generated from an automated workflow created with n8n, respecting all current content policies and containing no illegal, offensive, or protected elements. All data handled is legal and public.