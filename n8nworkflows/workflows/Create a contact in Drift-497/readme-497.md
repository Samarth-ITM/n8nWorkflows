Create a contact in Drift

https://n8nworkflows.xyz/workflows/create-a-contact-in-drift-497


# Create a contact in Drift

### 1. Workflow Overview

This workflow is designed to create a contact in Drift, a conversational marketing and sales platform. It is triggered manually and then uses the Drift node to add a new contact based on the provided email and additional fields. The workflow is straightforward, focusing exclusively on the creation of a Drift contact with minimal input.

Logical blocks:  
- **1.1 Manual Trigger:** Initiates the workflow manually.  
- **1.2 Drift Contact Creation:** Uses the Drift node to create a contact in the Drift system.

---

### 2. Block-by-Block Analysis

#### 1.1 Manual Trigger

- **Overview:**  
  This block starts the workflow manually through user interaction, allowing the workflow to be executed on demand.

- **Nodes Involved:**  
  - On clicking 'execute'

- **Node Details:**  
  - **Node Name:** On clicking 'execute'  
  - **Type:** Manual Trigger  
  - **Technical Role:** Initiates the workflow manually without any input data.  
  - **Configuration Choices:** No parameters configured; default manual trigger used.  
  - **Key Variables/Expressions:** None.  
  - **Input Connections:** None (start node).  
  - **Output Connections:** Connected to the Drift node.  
  - **Version-specific Requirements:** Compatible with n8n version 1.0 and above.  
  - **Potential Failures:** None typical; manual trigger errors are rare unless UI issues occur.  
  - **Sub-workflow Reference:** None.

#### 1.2 Drift Contact Creation

- **Overview:**  
  Sends a request to Drift’s API to create a new contact using the Drift node. Email and additional fields can be specified, although in this workflow the email input is empty by default and must be set manually or via input data for actual usage.

- **Nodes Involved:**  
  - Drift

- **Node Details:**  
  - **Node Name:** Drift  
  - **Type:** Drift node (API integration node)  
  - **Technical Role:** Creates or updates a contact in Drift via API call.  
  - **Configuration Choices:**  
    - Email field is currently empty, meaning no email is set by default.  
    - Additional fields parameter is empty, so no extra contact information is sent.  
  - **Key Variables/Expressions:** None used; static empty values configured.  
  - **Input Connections:** Receives trigger from the Manual Trigger node.  
  - **Output Connections:** None (end node).  
  - **Version-specific Requirements:** Requires proper Drift API credentials configured in n8n credentials manager.  
  - **Potential Failures:**  
    - Authentication errors if credentials are invalid or expired.  
    - API rate limits or downtime on Drift’s side.  
    - Missing required fields (e.g., email) may cause API rejection.  
  - **Sub-workflow Reference:** None.

---

### 3. Summary Table

| Node Name           | Node Type             | Functional Role         | Input Node(s)              | Output Node(s) | Sticky Note                                                                            |
|---------------------|-----------------------|------------------------|----------------------------|----------------|----------------------------------------------------------------------------------------|
| On clicking 'execute'| Manual Trigger        | Workflow initiation    | None                       | Drift          |                                                                                        |
| Drift               | Drift API integration | Create Drift contact   | On clicking 'execute'       | None           | Requires valid Drift API credentials; email field currently empty and must be set manually or via input |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a node of type **Manual Trigger**.  
   - Leave all default settings as is (no parameters needed).  
   - This node will serve as the entry point to run the workflow manually.

2. **Add Drift Node**  
   - Add a node of type **Drift** (search for “Drift” in the node list).  
   - Configure the credentials by selecting or creating a Drift API credential with your API key or OAuth setup.  
   - In the **Email** field, either leave empty or provide an expression or static email to create the contact.  
   - Optionally fill **Additional Fields** with any extra contact information (e.g., first name, last name) as key-value pairs.  
   - No other parameters are required unless your use case demands it.

3. **Connect Nodes**  
   - Draw a connection from the **Manual Trigger** node’s output to the **Drift** node’s input.

4. **Save and Test**  
   - Save the workflow.  
   - Execute the workflow manually via the manual trigger.  
   - Check the output of the Drift node and verify the contact creation in your Drift account.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                           |
|-------------------------------------------------------------------------------------------------|------------------------------------------|
| Make sure to configure valid Drift API credentials before running the workflow.                  | Drift API Credentials setup in n8n        |
| Drift API requires at least an email to create a contact; leaving it empty will cause failure.   | Drift API documentation: https://devdocs.drift.com/ |
| Manual trigger is useful for ad hoc testing or one-off executions but consider automated triggers for production. | n8n triggers documentation                |