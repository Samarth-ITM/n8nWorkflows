Create a new Freshdesk ticket

https://n8nworkflows.xyz/workflows/create-a-new-freshdesk-ticket-448


# Create a new Freshdesk ticket

### 1. Workflow Overview

This workflow is designed to create a new support ticket in Freshdesk when manually triggered. It serves as a companion example for documenting the Freshdesk node in n8n. The workflow consists of two logical blocks:

- **1.1 Trigger Block:** Manual initiation of the workflow by the user.
- **1.2 Freshdesk Ticket Creation Block:** Creating a new Freshdesk ticket with predefined parameters such as status and requester email.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger Block

- **Overview:**  
  This block enables manual execution of the workflow, allowing users to initiate the ticket creation process on demand.

- **Nodes Involved:**  
  - On clicking 'execute'

- **Node Details:**  
  - **Name:** On clicking 'execute'  
  - **Type:** Manual Trigger (n8n-nodes-base.manualTrigger)  
  - **Technical Role:** Entry point of the workflow, initiates execution without any input data.  
  - **Configuration:** Default manual trigger settings with no additional parameters.  
  - **Key Expressions/Variables:** None.  
  - **Input Connections:** None (start node).  
  - **Output Connections:** Connects to the Freshdesk node.  
  - **Version-specific Requirements:** Compatible with all n8n versions supporting manual triggers.  
  - **Potential Failures:** None expected, as it is a manual trigger.  
  - **Sub-workflow Reference:** None.  

#### 1.2 Freshdesk Ticket Creation Block

- **Overview:**  
  This block creates a new ticket in Freshdesk using the Freshdesk API credentials and predefined ticket parameters.

- **Nodes Involved:**  
  - Freshdesk

- **Node Details:**  
  - **Name:** Freshdesk  
  - **Type:** Freshdesk Node (n8n-nodes-base.freshdesk)  
  - **Technical Role:** Sends a request to Freshdesk API to create a ticket.  
  - **Configuration:**  
    - **Status:** open (ticket is created as open)  
    - **Requester Identification:** email with value "user@example.com" (the email of the ticket requester)  
    - **Options:** None specified (default options)  
  - **Credentials:** Uses "freshdesk-api" credential configured with valid Freshdesk API key and domain.  
  - **Key Expressions/Variables:** Static values for status and requester email; no dynamic expressions used.  
  - **Input Connections:** Receives trigger from manual trigger node.  
  - **Output Connections:** None (end of workflow).  
  - **Version-specific Requirements:** Requires Freshdesk node available in n8n version used.  
  - **Potential Failures:**  
    - Authentication errors if the API key or domain are invalid.  
    - Network timeouts or API rate limits.  
    - Invalid requester email format or non-existent user.  
  - **Sub-workflow Reference:** None.  

---

### 3. Summary Table

| Node Name           | Node Type             | Functional Role           | Input Node(s)       | Output Node(s) | Sticky Note |
|---------------------|-----------------------|--------------------------|---------------------|----------------|-------------|
| On clicking 'execute'| Manual Trigger        | Workflow initiation      | -                   | Freshdesk      |             |
| Freshdesk           | Freshdesk API Node     | Creates a new Freshdesk ticket | On clicking 'execute' | -              | Companion workflow for Freshdesk node documentation |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a new node of type **Manual Trigger**.  
   - No additional parameters are needed.  
   - This node will serve as the starting point to manually execute the workflow.

2. **Create Freshdesk Node**  
   - Add a new node of type **Freshdesk**.  
   - Configure the node with the following parameters:  
     - **Status:** Select "open" from the dropdown to set the ticket status as open.  
     - **Requester:** Select "email" to identify the requester by email.  
     - **Requester Identification Value:** Enter "user@example.com" (replace with the actual requester email as needed).  
     - Leave Options empty unless custom options are required.

3. **Set Up Credentials for Freshdesk Node**  
   - Create or select existing Freshdesk API credentials.  
   - Credentials must include:  
     - Freshdesk domain (e.g., yourcompany.freshdesk.com)  
     - API key with appropriate permissions to create tickets.

4. **Connect Nodes**  
   - Connect the output of the Manual Trigger node to the input of the Freshdesk node.  
   - This connection ensures that when the workflow is manually triggered, a new Freshdesk ticket is created.

5. **Save and Execute**  
   - Save the workflow.  
   - Execute manually by clicking the trigger node’s "Execute" button.  
   - Verify that a new ticket is created in Freshdesk with the specified parameters.

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                         |
|-----------------------------------------------------------------------------------------------|---------------------------------------|
| This workflow is a companion example specifically for Freshdesk node documentation in n8n.   | Workflow description                   |
| Freshdesk API credentials must be correctly configured for successful ticket creation.        | Credential setup in n8n                |
| For more details on Freshdesk API usage with n8n, see: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.freshdesk/ | Official n8n Freshdesk node documentation |

---

This completes the comprehensive reference documentation for the "Create a new Freshdesk ticket" workflow.