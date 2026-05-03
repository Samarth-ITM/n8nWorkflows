Get entries from a Cockpit collection

https://n8nworkflows.xyz/workflows/get-entries-from-a-cockpit-collection-518


# Get entries from a Cockpit collection

### 1. Workflow Overview

This workflow is designed to retrieve entries from a specified Cockpit CMS collection when manually triggered. It serves as a companion example for demonstrating how to use the Cockpit node in n8n. The workflow consists of two main logical blocks:

- **1.1 Input Reception:** A manual trigger node initiates the workflow execution.
- **1.2 Cockpit Data Retrieval:** The Cockpit node fetches entries from a defined collection using provided API credentials.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block provides a manual trigger to start the workflow execution on demand. It allows users to run the workflow interactively without external events.

- **Nodes Involved:**  
  - On clicking 'execute'

- **Node Details:**  
  - **Node Name:** On clicking 'execute'  
  - **Type:** Manual Trigger node (n8n-nodes-base.manualTrigger)  
  - **Technical Role:** Starts the workflow manually  
  - **Configuration Choices:** Default setup with no additional parameters or filters  
  - **Key Expressions/Variables:** None  
  - **Input Connections:** None (entry node)  
  - **Output Connections:** Connected to the Cockpit node  
  - **Version Requirements:** Compatible with n8n version supporting manual trigger node (version 1 or higher)  
  - **Potential Failures:** None expected; manual triggers typically do not fail unless the workflow is disabled or n8n itself has issues  
  - **Sub-Workflow:** Not part of a sub-workflow  

#### 1.2 Cockpit Data Retrieval

- **Overview:**  
  This block fetches entries from a specified Cockpit CMS collection using the Cockpit API and credentials. It queries the collection named "samplecollection" and outputs the retrieved data.

- **Nodes Involved:**  
  - Cockpit

- **Node Details:**  
  - **Node Name:** Cockpit  
  - **Type:** Cockpit node (n8n-nodes-base.cockpit)  
  - **Technical Role:** Connects to Cockpit CMS and retrieves entries from a collection  
  - **Configuration Choices:**  
    - Collection selected: "samplecollection"  
    - No additional query options specified (default retrieval of all entries)  
  - **Key Expressions/Variables:** None explicitly used; static collection name  
  - **Input Connections:** Receives trigger from the manual trigger node  
  - **Output Connections:** None (end node)  
  - **Credentials:** Uses "cockpitApi" credential which must be configured with API URL and token  
  - **Version Requirements:** Compatible with n8n version supporting Cockpit node (version 1 or higher)  
  - **Potential Failures:**  
    - Authentication errors if API credentials are invalid or expired  
    - Network or timeout errors if Cockpit CMS is unreachable  
    - Empty results if collection name is incorrect or no entries exist  
  - **Sub-Workflow:** Not part of a sub-workflow  

---

### 3. Summary Table

| Node Name           | Node Type                  | Functional Role           | Input Node(s)          | Output Node(s) | Sticky Note |
|---------------------|----------------------------|--------------------------|-----------------------|----------------|-------------|
| On clicking 'execute' | Manual Trigger (manualTrigger) | Initiates workflow manually | None                  | Cockpit        |             |
| Cockpit             | Cockpit node               | Retrieves entries from Cockpit collection | On clicking 'execute' | None           |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Add a new node of type **Manual Trigger**.  
   - Leave all settings as default. This node will start the workflow manually.

2. **Create Cockpit Node:**  
   - Add a new node of type **Cockpit**.  
   - Under **Parameters**, set the **Collection** field to `"samplecollection"`.  
   - Leave other options as default (no filters or sorting).

3. **Configure Credentials for Cockpit:**  
   - In the Cockpit node, select or create credentials named `"cockpitApi"`.  
   - Enter the Cockpit API base URL and API token for authentication. Ensure the token has permission to read the collection.

4. **Connect Nodes:**  
   - Connect the output of the **Manual Trigger** node to the input of the **Cockpit** node.

5. **Save and Execute:**  
   - Save the workflow.  
   - Click **Execute Workflow** or trigger manually to test retrieval of entries from the collection.

---

### 5. General Notes & Resources

| Note Content                                              | Context or Link                                  |
|-----------------------------------------------------------|-------------------------------------------------|
| This workflow is a companion example for Cockpit node docs | Demonstrates simple integration with Cockpit CMS |
| Ensure API credentials have correct permissions            | Cockpit API documentation for authentication    |
| Screenshot attached shows node layout and connections     | [workflow-screenshot](fileId:119)                |