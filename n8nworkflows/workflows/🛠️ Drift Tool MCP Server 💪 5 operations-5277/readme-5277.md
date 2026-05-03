🛠️ Drift Tool MCP Server 💪 5 operations

https://n8nworkflows.xyz/workflows/----drift-tool-mcp-server----5-operations-5277


# 🛠️ Drift Tool MCP Server 💪 5 operations

### 1. Workflow Overview

This workflow, titled **"Drift Tool MCP Server"**, serves as a backend automation endpoint designed for managing contacts through Drift's API. It is structured to handle five core contact-related operations: creating, retrieving, updating, deleting contacts, and fetching custom attributes for a contact.

The logical structure divides into two main blocks:

- **1.1 Input Reception and Routing**:  
  A single webhook-based trigger node listens for incoming requests specifying which Drift operation to perform. This node acts as the central entry point for all operations.

- **1.2 Contact Operations on Drift**:  
  Five separate nodes each execute a distinct Drift API operation:
  - Create a contact  
  - Get a contact  
  - Update a contact  
  - Delete a contact  
  - Get custom attributes for a contact  

Each of these operation nodes is directly connected to the central trigger node, which routes the requests based on operation type.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Routing

- **Overview:**  
  This block captures external requests via a webhook and serves as the gateway to all Drift contact operations. It interprets incoming requests and forwards them to the appropriate operation node.

- **Nodes Involved:**  
  - Drift Tool MCP Server (Trigger Node)  
  - Workflow Overview 0 (Sticky Note)  
  - Sticky Note 1 (Sticky Note)

- **Node Details:**

  - **Drift Tool MCP Server**  
    - Type: `@n8n/n8n-nodes-langchain.mcpTrigger`  
    - Role: Webhook trigger specialized for MCP (Multichannel Platform) requests, designed to accept and route commands for Drift contact operations.  
    - Configuration: Uses a fixed webhook ID `215f3f76-4fe0-401d-9502-2da6e0c7cb03` exposed externally. No additional parameters configured.  
    - Input: External HTTP requests (likely POST) containing operation details.  
    - Output: Routes the request payload to one of the five Drift operation nodes via the `ai_tool` output connection.  
    - Version Requirements: Requires n8n environment supporting LangChain MCP Trigger nodes.  
    - Edge Cases / Failures:  
      - Missing or malformed request payloads may cause routing failure.  
      - Webhook authentication or permission errors if exposed publicly without protection.  
      - Timeout or network issues if external clients are slow or disconnected.  
    - Sub-workflow: None  

  - **Workflow Overview 0** (Sticky Note)  
    - Type: Sticky Note  
    - Purpose: Placeholder or comment area, currently empty.  
    - No inputs or outputs.  

  - **Sticky Note 1**  
    - Type: Sticky Note  
    - Purpose: Placeholder or comment area, currently empty.  
    - No inputs or outputs.  

---

#### 2.2 Contact Operations on Drift

- **Overview:**  
  This block contains five nodes, each handling a specific Drift contact operation. Each node connects directly to the trigger node and executes the corresponding API call.

- **Nodes Involved:**  
  - Create a contact  
  - Get a contact  
  - Update a contact  
  - Delete a contact  
  - Get custom attributes for a contact

- **Node Details:**

  - **Create a contact**  
    - Type: `n8n-nodes-base.driftTool`  
    - Role: Creates a new contact record in Drift based on input data.  
    - Configuration: Default; expects incoming data from the trigger node specifying contact fields.  
    - Inputs: From `Drift Tool MCP Server` node.  
    - Outputs: Contact creation result or error response.  
    - Edge Cases:  
      - Validation errors if required fields are missing.  
      - API rate limiting or authentication failures.  
      - Data format errors causing API rejection.  

  - **Get a contact**  
    - Type: `n8n-nodes-base.driftTool`  
    - Role: Retrieves detailed information for a specified contact by ID or email.  
    - Configuration: Expects identifier parameters in incoming data.  
    - Inputs: From `Drift Tool MCP Server` node.  
    - Outputs: Contact data from Drift or error.  
    - Edge Cases:  
      - Contact not found errors.  
      - Authentication or permission errors.  
      - Network timeouts.  

  - **Update a contact**  
    - Type: `n8n-nodes-base.driftTool`  
    - Role: Updates existing contact information with new data.  
    - Configuration: Requires contact ID and fields to update.  
    - Inputs: From `Drift Tool MCP Server` node.  
    - Outputs: Confirmation of update or error.  
    - Edge Cases:  
      - Attempting to update non-existent contacts.  
      - Validation failures on update data.  

  - **Delete a contact**  
    - Type: `n8n-nodes-base.driftTool`  
    - Role: Removes a contact from Drift based on identifier.  
    - Configuration: Requires contact ID or email.  
    - Inputs: From `Drift Tool MCP Server` node.  
    - Outputs: Deletion confirmation or error.  
    - Edge Cases:  
      - Trying to delete contacts that do not exist.  
      - Permission or authentication failures.  

  - **Get custom attributes for a contact**  
    - Type: `n8n-nodes-base.driftTool`  
    - Role: Retrieves custom attributes defined on a contact in Drift.  
    - Configuration: Requires contact identifier to fetch attributes.  
    - Inputs: From `Drift Tool MCP Server` node.  
    - Outputs: List of custom attribute key-value pairs or error.  
    - Edge Cases:  
      - Contact not found.  
      - Empty custom attribute sets.  

---

### 3. Summary Table

| Node Name                         | Node Type                             | Functional Role                           | Input Node(s)          | Output Node(s)                          | Sticky Note |
|----------------------------------|-------------------------------------|-----------------------------------------|-----------------------|---------------------------------------|-------------|
| Workflow Overview 0              | Sticky Note                         | Documentation placeholder                | —                     | —                                     |             |
| Drift Tool MCP Server            | MCP Trigger (LangChain)              | Webhook trigger and router for requests | External HTTP requests | Create a contact, Get a contact, Update a contact, Delete a contact, Get custom attributes for a contact |             |
| Create a contact                | Drift Tool Node                      | Creates a new contact in Drift           | Drift Tool MCP Server  | —                                     |             |
| Get custom attributes for a contact | Drift Tool Node                  | Fetches custom attributes of a contact  | Drift Tool MCP Server  | —                                     |             |
| Delete a contact                | Drift Tool Node                      | Deletes a contact in Drift                | Drift Tool MCP Server  | —                                     |             |
| Get a contact                  | Drift Tool Node                      | Retrieves contact details                 | Drift Tool MCP Server  | —                                     |             |
| Update a contact                | Drift Tool Node                      | Updates contact details                   | Drift Tool MCP Server  | —                                     |             |
| Sticky Note 1                   | Sticky Note                         | Documentation placeholder                | —                     | —                                     |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and set the timezone to `America/New_York`.

2. **Add the MCP Trigger node**:  
   - Node Type: `@n8n/n8n-nodes-langchain.mcpTrigger`  
   - Name: `Drift Tool MCP Server`  
   - Configure the webhook with a unique webhook ID (or use auto-generated).  
   - No additional parameters needed. This node will listen for incoming HTTP requests and route them to contact operation nodes.

3. **Add the Drift Tool nodes for each operation**:  
   For each operation below, create a new node of type `Drift Tool` (`n8n-nodes-base.driftTool`):

   - **Create a contact**  
     - Name: `Create a contact`  
     - Configure to expect contact creation data from input.

   - **Get a contact**  
     - Name: `Get a contact`  
     - Configure to accept a contact identifier (ID or email).

   - **Update a contact**  
     - Name: `Update a contact`  
     - Configure to accept contact ID and fields for update.

   - **Delete a contact**  
     - Name: `Delete a contact`  
     - Configure to accept contact identifier for deletion.

   - **Get custom attributes for a contact**  
     - Name: `Get custom attributes for a contact`  
     - Configure to accept contact identifier to retrieve attributes.

4. **Connect nodes**:  
   - From the `Drift Tool MCP Server` trigger node, create output connections to each of the five Drift Tool operation nodes using the output named `ai_tool`. This routes requests to the correct operation node based on the request content.

5. **Configure credentials**:  
   - For all Drift Tool nodes, select or create the appropriate Drift API credentials with sufficient permissions to perform create, read, update, delete, and attribute read operations on contacts.

6. **Add sticky notes for documentation** (optional):  
   - Add two sticky notes named `Workflow Overview 0` and `Sticky Note 1` as placeholders for documentation or comments.

7. **Activate the workflow** to start listening for incoming requests.

---

### 5. General Notes & Resources

| Note Content                                                                                                     | Context or Link                  |
|-----------------------------------------------------------------------------------------------------------------|--------------------------------|
| The workflow uses the MCP Trigger node from LangChain integration, which requires n8n versions supporting this node type. | MCP Trigger node compatibility |
| Drift API credentials must have contact management permissions enabled to avoid authorization errors.           | Drift API documentation        |
| Consider securing the webhook with authentication or IP whitelisting to prevent unauthorized access.            | n8n webhook security best practices |
| Empty sticky notes indicate placeholders for future documentation or workflow instructions.                     | Workflow maintainability       |

---

**Disclaimer:** The text provided originates exclusively from an automated n8n workflow. It complies strictly with applicable content policies and contains no illegal, offensive, or proprietary elements. All processed data is legal and publicly accessible.