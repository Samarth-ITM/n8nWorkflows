Create a meeting in Zoom automatically

https://n8nworkflows.xyz/workflows/create-a-meeting-in-zoom-automatically-453


# Create a meeting in Zoom automatically

### 1. Workflow Overview

This workflow automates the creation of a Zoom meeting using n8n’s Zoom node. It is triggered manually by the user and then creates a Zoom meeting with predefined parameters. The workflow is simple and consists of two logical blocks:

- **1.1 Input Reception:** A manual trigger node initiates the workflow execution.
- **1.2 Zoom Meeting Creation:** The Zoom node creates a meeting using preset topic and authentication credentials.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block captures the user’s manual initiation of the workflow, allowing it to run on demand.

- **Nodes Involved:**  
  - On clicking 'execute'

- **Node Details:**  
  - **Name:** On clicking 'execute'  
  - **Type:** Manual Trigger (n8n-nodes-base.manualTrigger)  
  - **Technical Role:** Entry point; triggers workflow execution when manually activated.  
  - **Configuration:** No parameters set; default manual trigger.  
  - **Key Expressions/Variables:** None.  
  - **Input/Output Connections:** Outputs to the Zoom node.  
  - **Version-specific Requirements:** None.  
  - **Potential Failure Types:** No critical failures expected; manual trigger may not run if user lacks permission or UI is inaccessible.

#### 1.2 Zoom Meeting Creation

- **Overview:**  
  This block creates a Zoom meeting with predetermined settings using OAuth2 credentials.

- **Nodes Involved:**  
  - Zoom

- **Node Details:**  
  - **Name:** Zoom  
  - **Type:** Zoom node (n8n-nodes-base.zoom)  
  - **Technical Role:** Executes the API call to create a Zoom meeting.  
  - **Configuration Choices:**  
    - Topic: “Something” (placeholder, can be modified)  
    - Authentication: OAuth2 (configured via credentials)  
    - Additional Fields: None specified (uses defaults)  
  - **Key Expressions/Variables:** Fixed topic string “Something”  
  - **Input/Output Connections:** Input from Manual Trigger; no output connected (end node).  
  - **Version-specific Requirements:** OAuth2 credentials must be set up and valid for Zoom API access.  
  - **Potential Failure Types:**  
    - Authentication errors if OAuth2 token invalid or expired.  
    - API rate limits or network errors.  
    - Missing or incorrect parameters may cause API call failure.  
  - **Sub-workflow Reference:** None.

---

### 3. Summary Table

| Node Name           | Node Type             | Functional Role          | Input Node(s)       | Output Node(s) | Sticky Note |
|---------------------|-----------------------|-------------------------|---------------------|----------------|-------------|
| On clicking 'execute'| Manual Trigger        | Starts workflow manually | -                   | Zoom           |             |
| Zoom                | Zoom API node          | Creates Zoom meeting     | On clicking 'execute'| -              |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Add a **Manual Trigger** node named “On clicking 'execute'”. No special parameters needed.

2. **Create Zoom Node:**  
   - Add a **Zoom** node named “Zoom”.  
   - Set the **Operation** to “Create Meeting”.  
   - Configure the **Topic** field with the string “Something” (or your preferred meeting topic).  
   - Under **Authentication**, choose OAuth2 credentials for Zoom (ensure you have created and configured these credentials in n8n).  
   - Leave **Additional Fields** empty (or configure as needed for your use case).

3. **Connect Nodes:**  
   - Connect the output of “On clicking 'execute'” to the input of “Zoom”.

4. **Credential Setup:**  
   - In n8n, configure the Zoom OAuth2 credentials with your Zoom API credentials (client ID, client secret, and access tokens).  
   - Test credentials to ensure connectivity.

5. **Save and Activate:**  
   - Save the workflow.  
   - Optionally activate or run it manually to create a Zoom meeting.

---

### 5. General Notes & Resources

| Note Content                                                          | Context or Link                      |
|---------------------------------------------------------------------|------------------------------------|
| OAuth2 credentials must be pre-configured in n8n for Zoom node usage.| See n8n documentation on OAuth2 setup for Zoom. |
| The meeting topic “Something” is a placeholder; customize as needed.| Workflow screenshot available via provided fileId. |

---

This documentation enables understanding, modification, and reproduction of the Zoom meeting creation workflow within n8n. Potential errors mainly relate to authentication and API connectivity. Adjust the Zoom node parameters to suit specific meeting requirements.