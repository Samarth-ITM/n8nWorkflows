Create a new contact in Agile CRM

https://n8nworkflows.xyz/workflows/create-a-new-contact-in-agile-crm-474


# Create a new contact in Agile CRM

### 1. Workflow Overview

This workflow is designed to create a new contact record in Agile CRM upon manual execution. Its primary use case is to allow users to add fresh contact information into Agile CRM efficiently via a simple trigger. The workflow consists of two logical blocks:

- **1.1 Input Trigger:** A manual trigger node that initiates the workflow when executed by the user.
- **1.2 Contact Creation:** A single Agile CRM node that creates a new contact in the CRM system using the provided contact details.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Trigger

- **Overview:**  
  This block starts the workflow manually. It waits for a user to execute it to proceed with contact creation.

- **Nodes Involved:**  
  - On clicking 'execute'

- **Node Details:**  
  - **Name:** On clicking 'execute'  
  - **Type:** Manual Trigger  
  - **Technical Role:** Initiates the workflow by manual user interaction.  
  - **Configuration Choices:** No specific parameters are set—default manual trigger.  
  - **Key Expressions/Variables:** None used.  
  - **Input Connections:** None (starting node).  
  - **Output Connections:** Connects to the AgileCRM node.  
  - **Version Specifics:** Compatible with all n8n versions supporting manual triggers.  
  - **Potential Failures:** None typical, except if the workflow is deactivated or not properly started by the user.  
  - **Sub-Workflow:** Not applicable.

#### 1.2 Contact Creation

- **Overview:**  
  This block creates a new contact record in Agile CRM using the details passed from the manual trigger.

- **Nodes Involved:**  
  - AgileCRM

- **Node Details:**  
  - **Name:** AgileCRM  
  - **Type:** Agile CRM Node  
  - **Technical Role:** Performs the "create contact" operation in Agile CRM via API.  
  - **Configuration Choices:**  
    - Operation set to "create."  
    - Additional fields for the new contact include `firstName` and `lastName`, both currently configured as empty strings (placeholders).  
  - **Key Expressions/Variables:** No dynamic expressions; static empty strings used for contact names, indicating manual or further extension needed to supply actual data.  
  - **Input Connections:** Receives trigger from the manual trigger node.  
  - **Output Connections:** None (terminal node).  
  - **Version Specifics:** Requires proper Agile CRM API credentials configured in the node’s credentials section.  
  - **Potential Failures:**  
    - Authentication failure if API credentials are missing or invalid.  
    - API rate limiting or downtime of Agile CRM service.  
    - Failure if required contact fields are missing or invalid (currently empty).  
  - **Sub-Workflow:** Not applicable.

---

### 3. Summary Table

| Node Name            | Node Type          | Functional Role       | Input Node(s)           | Output Node(s) | Sticky Note                                  |
|----------------------|--------------------|-----------------------|------------------------|----------------|----------------------------------------------|
| On clicking 'execute' | Manual Trigger     | Workflow initiation   | —                      | AgileCRM       |                                              |
| AgileCRM             | Agile CRM Node     | Create new contact    | On clicking 'execute'  | —              |                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a new node of type **Manual Trigger**.  
   - Name it "On clicking 'execute'".  
   - Leave default settings as is (no parameters required).  

2. **Create Agile CRM Node**  
   - Add a new node of type **Agile CRM**.  
   - Set the operation to **Create Contact**.  
   - Under additional fields, set `firstName` and `lastName` as empty strings or provide default values if desired.  
   - Configure the node with valid Agile CRM credentials:  
     - Create or select existing credentials for Agile CRM API access with appropriate permissions.  

3. **Connect Nodes**  
   - Connect the output of "On clicking 'execute'" node to the input of the Agile CRM node.  

4. **Save and Activate**  
   - Save the workflow.  
   - Activate if needed, or manually execute to test.  

Note: To make this workflow fully functional, replace empty `firstName` and `lastName` fields with actual data inputs either via parameters, expressions, or prior nodes that collect user input.

---

### 5. General Notes & Resources

| Note Content                                                                           | Context or Link                     |
|----------------------------------------------------------------------------------------|-----------------------------------|
| For Agile CRM API credentials setup, refer to Agile CRM API documentation for API keys and OAuth configuration. | https://www.agilecrm.com/api/     |
| The workflow is minimal and intended as a foundational example for contact creation automation. |                                   |
| To expand functionality, consider adding input nodes like HTTP Request or Google Sheets to dynamically supply contact details. |                                   |

---

This document fully describes the "Create a new contact in Agile CRM" workflow, enabling users and developers to understand, reproduce, and extend it effectively.