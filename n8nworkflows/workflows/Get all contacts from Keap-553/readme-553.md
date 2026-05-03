Get all contacts from Keap

https://n8nworkflows.xyz/workflows/get-all-contacts-from-keap-553


# Get all contacts from Keap

### 1. Workflow Overview

This workflow is designed to retrieve all contact records from the Keap CRM system. It serves as a companion example for the Keap node documentation in n8n, demonstrating a simple use case of fetching data from Keap using OAuth2 credentials. The flow is straightforward and consists of two primary logical blocks:

- **1.1 Manual Trigger:** Initiates the workflow execution manually.
- **1.2 Data Retrieval from Keap:** Uses the Keap node to fetch all contacts.

---

### 2. Block-by-Block Analysis

#### 1.1 Manual Trigger

- **Overview:**  
  This block allows the user to start the workflow manually by clicking the "execute" button in the n8n editor or UI. It serves as the entry point for the workflow.

- **Nodes Involved:**  
  - On clicking 'execute'

- **Node Details:**  
  - **Name:** On clicking 'execute'  
  - **Type:** Manual Trigger (n8n-nodes-base.manualTrigger)  
  - **Configuration:** No parameters set since it simply triggers the workflow manually.  
  - **Key Expressions / Variables:** None  
  - **Inputs:** None (trigger node)  
  - **Outputs:** Connects to the Keap node  
  - **Version Requirements:** Available in all current n8n versions  
  - **Potential Failures:** None expected; manual trigger nodes are stable and do not rely on external services.

#### 1.2 Data Retrieval from Keap

- **Overview:**  
  This block fetches all contacts from the Keap CRM system using the Keap node with OAuth2 authentication. It demonstrates how to query the "contact" resource and perform the "getAll" operation.

- **Nodes Involved:**  
  - Keap

- **Node Details:**  
  - **Name:** Keap  
  - **Type:** Keap (n8n-nodes-base.keap)  
  - **Configuration:**  
    - Resource: Contact  
    - Operation: Get All  
    - Options: Default (no filters or limits specified)  
  - **Key Expressions / Variables:** None used; static configuration  
  - **Inputs:** Connected from the Manual Trigger node  
  - **Outputs:** Outputs an array of contact objects retrieved from Keap  
  - **Credentials:** Uses OAuth2 credentials stored under "keap_creds"  
  - **Version Requirements:** Requires n8n version supporting the Keap node (available in recent versions)  
  - **Potential Failures:**  
    - Authentication errors if OAuth2 token is invalid or expired  
    - API rate limits or timeouts from Keap  
    - Network connectivity issues  
    - Empty results if no contacts exist in Keap

---

### 3. Summary Table

| Node Name           | Node Type             | Functional Role        | Input Node(s)         | Output Node(s) | Sticky Note                              |
|---------------------|-----------------------|-----------------------|-----------------------|----------------|------------------------------------------|
| On clicking 'execute'| Manual Trigger        | Start workflow        | —                     | Keap           |                                          |
| Keap                | Keap Node             | Retrieve all contacts | On clicking 'execute' | —              | Companion workflow for Keap node docs   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a new node of type "Manual Trigger" from the trigger nodes list.  
   - Leave all parameters at their defaults.  
   - This node will be the entry point to manually start the workflow.

2. **Create Keap Node**  
   - Add a new node of type "Keap" (under integrations category).  
   - Set the **Resource** parameter to "Contact".  
   - Set the **Operation** parameter to "Get All".  
   - Leave other options empty unless filtering is required.  
   - Configure the node to use OAuth2 credentials: create or select credentials named "keap_creds". This requires setting up OAuth2 access with Keap API credentials (client ID, secret, token URL, etc.).

3. **Connect Nodes**  
   - Connect the output of the Manual Trigger node to the input of the Keap node.

4. **Save and Execute**  
   - Save the workflow.  
   - Click "Execute Workflow" or "Execute Node" on the Manual Trigger node to start fetching contacts.

---

### 5. General Notes & Resources

| Note Content                                                | Context or Link                                              |
|-------------------------------------------------------------|--------------------------------------------------------------|
| This workflow is a companion example for the official Keap node documentation in n8n. | Keap node docs: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.keap/ |
| Ensure your Keap OAuth2 credentials are properly configured with necessary scopes for contact access. | OAuth2 credential setup in n8n Docs: https://docs.n8n.io/integrations/credentials/oauth2/ |