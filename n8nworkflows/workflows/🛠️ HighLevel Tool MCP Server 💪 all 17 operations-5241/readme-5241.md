🛠️ HighLevel Tool MCP Server 💪 all 17 operations

https://n8nworkflows.xyz/workflows/----highlevel-tool-mcp-server----all-17-operations-5241


# 🛠️ HighLevel Tool MCP Server 💪 all 17 operations

### 1. Workflow Overview

This workflow titled **"HighLevel Tool MCP Server"** serves as a centralized automation server handling 17 distinct operations related to CRM-like entities such as contacts, opportunities, tasks, and calendar appointments. It is designed to receive external triggers via a webhook, then route requests to the appropriate HighLevel Tool nodes to execute specific CRUD (Create, Read, Update, Delete) operations or calendar booking tasks.

The workflow is logically organized into the following blocks based on entity type and operation:

- **1.1 Trigger Reception**: Entry point listening for incoming MCP (Multi-Channel Platform) trigger events.
- **1.2 Contact Management Operations**: Nodes handling creation, retrieval, update, and deletion of contacts.
- **1.3 Opportunity Management Operations**: Nodes managing creation, retrieval, update, and deletion of sales opportunities.
- **1.4 Task Management Operations**: Nodes for task lifecycle operations including create, read, update, and delete.
- **1.5 Calendar Appointment Operations**: Nodes managing calendar free slot queries and booking appointments.

Each operation node is connected back to the central MCP trigger node, which acts as a dispatcher to invoke the corresponding operation depending on the incoming request. Sticky notes are present but empty, presumably placeholders for documentation or future annotations.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger Reception Block

- **Overview:** This block defines the webhook listener node that receives external MCP trigger events and initiates workflow execution.

- **Nodes Involved:**  
  - HighLevel Tool MCP Server (MCP Trigger node)

- **Node Details:**

  - **HighLevel Tool MCP Server**  
    - Type: `@n8n/n8n-nodes-langchain.mcpTrigger`  
    - Role: Webhook trigger node specialized for MCP event reception.  
    - Configuration: Uses a generated webhook ID to listen for incoming HTTP requests; no additional parameters configured.  
    - Inputs: None (trigger node)  
    - Outputs: Connected to all operation nodes via ai_tool outputs.  
    - Version Requirements: Requires n8n environment supporting MCP trigger node type.  
    - Edge Cases: Potential failure if webhook URL is not registered or external calls are malformed; security considerations for exposed webhook.  
    - Sub-workflow: None.

---

#### 2.2 Contact Management Operations Block

- **Overview:** Handles all contact-related operations including create/update, delete, get single contact, get multiple contacts, and update.

- **Nodes Involved:**  
  - Create or update a contact  
  - Delete a contact  
  - Get a contact  
  - Get many contacts  
  - Update a contact  
  - Sticky Note 1 (empty)

- **Node Details:**

  - **Create or update a contact**  
    - Type: `n8n-nodes-base.highLevelTool`  
    - Role: Creates or updates a contact record in HighLevel.  
    - Configuration: Defaults; expects input data from trigger node.  
    - Inputs: Receives from MCP trigger node  
    - Outputs: None connected further (endpoint).  
    - Edge Cases: Auth errors if credentials invalid; validation errors if input data incomplete.

  - **Delete a contact**  
    - Similar role and configuration to above, for deleting contacts.

  - **Get a contact**  
    - Retrieves details for a single contact by ID.

  - **Get many contacts**  
    - Retrieves a list of contacts, likely with pagination or filtering.

  - **Update a contact**  
    - Updates an existing contact’s information.

  - **Sticky Note 1**  
    - Empty, positioned near contact operation nodes; potentially for future documentation or notes.

---

#### 2.3 Opportunity Management Operations Block

- **Overview:** Manages opportunity (sales lead or deal) lifecycle operations: create, delete, get single, get many, and update.

- **Nodes Involved:**  
  - Create an opportunity  
  - Delete an opportunity  
  - Get an opportunity  
  - Get many opportunities  
  - Update an opportunity  
  - Sticky Note 2 (empty)

- **Node Details:**

  - Nodes are all `n8n-nodes-base.highLevelTool` type, configured for respective opportunity operations.  
  - Inputs are from the MCP trigger node, output endpoints for each operation.  
  - Edge cases mirror contact operations: authentication issues, data validation errors, API timeouts.

  - **Sticky Note 2**  
    - Empty note near opportunity nodes.

---

#### 2.4 Task Management Operations Block

- **Overview:** Covers task-related operations including creation, deletion, retrieval (single and many), and update.

- **Nodes Involved:**  
  - Create a task  
  - Delete a task  
  - Get a task  
  - Get many tasks  
  - Update a task  
  - Sticky Note 3 (empty)

- **Node Details:**

  - All nodes use the HighLevel Tool node type, configured for task operations.  
  - Inputs come from the MCP trigger node, no further output chaining.  
  - Potential failure modes include auth errors, invalid input data, API limits.

  - **Sticky Note 3**  
    - Empty note positioned near task nodes.

---

#### 2.5 Calendar Appointment Operations Block

- **Overview:** Manages calendar-related operations, specifically booking an appointment and querying free slots.

- **Nodes Involved:**  
  - Book appointment in a calendar  
  - Get free slots of a calendar  
  - Sticky Note 4 (empty)

- **Node Details:**

  - Nodes utilize HighLevel Tool nodes configured for calendar operations.  
  - Triggered by the MCP trigger node.  
  - Edge cases: calendar availability conflicts, auth issues with calendar integration.

  - **Sticky Note 4**  
    - Empty note near calendar operation nodes.

---

### 3. Summary Table

| Node Name                     | Node Type                           | Functional Role                     | Input Node(s)               | Output Node(s) | Sticky Note |
|-------------------------------|-----------------------------------|-----------------------------------|-----------------------------|----------------|-------------|
| Workflow Overview 0            | stickyNote                        | Placeholder / documentation note  | None                        | None           |             |
| HighLevel Tool MCP Server      | MCP Trigger                      | Entry webhook trigger for MCP     | None                        | All operation nodes |             |
| Create or update a contact     | HighLevel Tool                   | Create or update contact           | HighLevel Tool MCP Server   | None           |             |
| Delete a contact               | HighLevel Tool                   | Delete contact                    | HighLevel Tool MCP Server   | None           |             |
| Get a contact                 | HighLevel Tool                   | Retrieve single contact           | HighLevel Tool MCP Server   | None           |             |
| Get many contacts             | HighLevel Tool                   | Retrieve multiple contacts        | HighLevel Tool MCP Server   | None           |             |
| Update a contact              | HighLevel Tool                   | Update contact                   | HighLevel Tool MCP Server   | None           |             |
| Sticky Note 1                 | stickyNote                      | Placeholder note near contacts    | None                        | None           |             |
| Create an opportunity          | HighLevel Tool                   | Create opportunity                | HighLevel Tool MCP Server   | None           |             |
| Delete an opportunity          | HighLevel Tool                   | Delete opportunity                | HighLevel Tool MCP Server   | None           |             |
| Get an opportunity             | HighLevel Tool                   | Retrieve single opportunity       | HighLevel Tool MCP Server   | None           |             |
| Get many opportunities         | HighLevel Tool                   | Retrieve multiple opportunities   | HighLevel Tool MCP Server   | None           |             |
| Update an opportunity          | HighLevel Tool                   | Update opportunity               | HighLevel Tool MCP Server   | None           |             |
| Sticky Note 2                 | stickyNote                      | Placeholder note near opportunities | None                      | None           |             |
| Create a task                 | HighLevel Tool                   | Create task                      | HighLevel Tool MCP Server   | None           |             |
| Delete a task                 | HighLevel Tool                   | Delete task                      | HighLevel Tool MCP Server   | None           |             |
| Get a task                   | HighLevel Tool                   | Retrieve single task             | HighLevel Tool MCP Server   | None           |             |
| Get many tasks               | HighLevel Tool                   | Retrieve multiple tasks          | HighLevel Tool MCP Server   | None           |             |
| Update a task                | HighLevel Tool                   | Update task                     | HighLevel Tool MCP Server   | None           |             |
| Sticky Note 3                 | stickyNote                      | Placeholder note near tasks       | None                        | None           |             |
| Book appointment in a calendar | HighLevel Tool                   | Book calendar appointment        | HighLevel Tool MCP Server   | None           |             |
| Get free slots of a calendar  | HighLevel Tool                   | Query free calendar slots        | HighLevel Tool MCP Server   | None           |             |
| Sticky Note 4                 | stickyNote                      | Placeholder note near calendar ops | None                      | None           |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the MCP Trigger Node:**
   - Add a node of type `MCP Trigger` (from Langchain n8n nodes).  
   - Configure no special parameters; save the webhook URL generated for external calls.

2. **Add Contact Management Nodes:**
   - Create 5 nodes of type `HighLevel Tool` for:  
     - "Create or update a contact"  
     - "Delete a contact"  
     - "Get a contact"  
     - "Get many contacts"  
     - "Update a contact"  
   - Connect each node’s input from the MCP Trigger node’s output.  
   - Configure each node’s operation mode inside its parameters accordingly (e.g., set to create, delete, get, etc.).  
   - Ensure credentials for HighLevel API are set in each node’s credentials section.

3. **Add Opportunity Management Nodes:**
   - Create 5 nodes of type `HighLevel Tool` named for opportunity operations: create, delete, get, get many, update.  
   - Connect their input from MCP Trigger node.  
   - Configure each node’s operation mode appropriately.  
   - Assign proper HighLevel credentials.

4. **Add Task Management Nodes:**
   - Create 5 nodes of type `HighLevel Tool` for task operations: create, delete, get, get many, update.  
   - Connect inputs from MCP Trigger node.  
   - Configure operation modes accordingly.  
   - Set credentials.

5. **Add Calendar Appointment Nodes:**
   - Create 2 nodes of type `HighLevel Tool`:  
     - "Book appointment in a calendar"  
     - "Get free slots of a calendar"  
   - Connect inputs from MCP Trigger node.  
   - Configure operation modes for calendar management.  
   - Set required calendar integration credentials.

6. **Add Sticky Notes (Optional):**
   - Add sticky note nodes near each major block for documentation or placeholders. Content can be left blank or filled as needed.

7. **Set Workflow Settings:**
   - Confirm timezone is set to "America/New_York" or as required.  
   - Save and activate the workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                              |
|-----------------------------------------------------------------------------------------------|----------------------------------------------|
| Workflow functions as a centralized MCP server handling multiple CRM-related operations.       | Workflow overview                             |
| Sticky notes are placeholders for annotations/documentation; currently empty.                  | Workflow annotations                          |
| MCP Trigger node requires proper external registration and secure handling of webhook URL.     | Security best practices for webhook exposure |
| HighLevel Tool nodes require valid API credentials configured in n8n credentials manager.      | HighLevel API documentation                   |
| Timezone set to America/New_York; adjust as needed for your locale.                            | Workflow settings                             |

---

This documentation offers a thorough, structured understanding of the "HighLevel Tool MCP Server" workflow, enabling users and AI automation systems to comprehend, reproduce, and modify it with clarity.