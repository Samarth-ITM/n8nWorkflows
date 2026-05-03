🛠️ Harvest Tool MCP Server 💪 all 51 operations

https://n8nworkflows.xyz/workflows/----harvest-tool-mcp-server----all-51-operations-5242


# 🛠️ Harvest Tool MCP Server 💪 all 51 operations

### 1. Workflow Overview

This workflow, titled **"Harvest Tool MCP Server"**, serves as a comprehensive automation interface for interacting with the Harvest Tool API, covering all 51 available operations. It is designed as a server-side executor that listens for incoming API requests through a specialized MCP (Multi-Channel Platform) trigger and routes these requests to appropriate Harvest Tool nodes to perform CRUD (Create, Read, Update, Delete) operations on various Harvest entities.

**Target Use Cases:**  
- Automating tasks related to clients, contacts, projects, tasks, time entries, invoices, expenses, estimates, and users within the Harvest ecosystem.  
- Serving as a centralized API handler for Harvest operations, enabling external systems or interfaces to invoke these operations via a single webhook endpoint.

**Logical Blocks:**  
- **1.1 Input Reception:** The MCP Trigger node receives incoming requests and determines which Harvest operation to perform.  
- **1.2 Harvest Operations:** A large set of Harvest Tool nodes, each responsible for a specific operation on entities like clients, contacts, projects, etc.  
- **1.3 Organizational Info:** Retrieval of company and authenticated user information.  
- **1.4 Operational Groupings:** Nodes are logically grouped by entity types (clients, contacts, estimates, expenses, invoices, projects, tasks, time entries, users), each block managing the full set of CRUD operations for that entity.  
- **1.5 Sticky Notes:** Visual annotations scattered throughout the workflow, possibly placeholders for documentation or reminders.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block receives and triggers the workflow based on external requests, acting as the entry point for all subsequent Harvest operations.

- **Nodes Involved:**  
  - Harvest Tool MCP Server (MCP Trigger node)

- **Node Details:**  
  - **Node:** Harvest Tool MCP Server  
    - Type: `@n8n/n8n-nodes-langchain.mcpTrigger`  
    - Role: MCP Trigger node that listens for incoming requests on a dedicated webhook.  
    - Configuration: Uses a webhook with ID `80f8e84d-8ffc-4d53-9f93-8d4bdc6bfb83`. No additional parameters configured, implying default listening behavior.  
    - Inputs: None (trigger node)  
    - Outputs: Connected to all subsequent Harvest nodes via their `ai_tool` input connections.  
    - Version: 1  
    - Edge Cases:  
      - Webhook connectivity issues or webhook ID mismatch could prevent firing.  
      - Unauthorized or malformed requests could cause unexpected behavior.  
    - Sub-workflow: None.

#### 1.2 Organizational Info

- **Overview:**  
  Nodes that retrieve information about the company for the authenticated user and about the authenticated user themselves, useful for context or validation.

- **Nodes Involved:**  
  - Retrieve the company for the currently authenticated user  
  - Get data of authenticated user

- **Node Details:**  
  - **Retrieve the company for the currently authenticated user**  
    - Type: `harvestTool`  
    - Role: Retrieves company details associated with the authenticated Harvest account.  
    - Configuration: Default settings, presumably using the authenticated Harvest credential.  
    - Input: Connected from MCP Trigger node via `ai_tool`.  
    - Output: Not connected further, likely returns company info as response data.  
    - Edge Cases: Auth failure, insufficient permissions, network timeout.

  - **Get data of authenticated user**  
    - Type: `harvestTool`  
    - Role: Retrieves details of the user authenticated via Harvest credentials.  
    - Configuration: Default.  
    - Input: Connected from MCP Trigger node.  
    - Output: No further connections.  
    - Edge Cases: Auth errors, data unavailability.

#### 1.3 Clients Operations

- **Overview:**  
  Manages all client-related operations: create, delete, retrieve single/all, and update.

- **Nodes Involved:**  
  - Create a client  
  - Delete a client  
  - Get data of a client  
  - Get data of all clients  
  - Update a client

- **Node Details:**  
  Each node is of type `harvestTool` configured for the specific client operation.  
  - Inputs: Each node receives input from the MCP Trigger node (via `ai_tool` connection).  
  - Outputs: No further node connections; each node handles the request independently.  
  - Configuration: Each node is preset for its operation (e.g., "Create a client" performs the POST operation to create clients).  
  - Edge Cases:  
    - Invalid client IDs for retrieval, update, or deletion.  
    - Validation errors on required fields during creation or update.  
    - Network or API rate limit errors.

#### 1.4 Contacts Operations

- **Overview:**  
  Full set of contact management operations: create, delete, get single/all, update.

- **Nodes Involved:**  
  - Create a contact  
  - Delete a contact  
  - Get data of a contact  
  - Get data of all contacts  
  - Update a contact

- **Node Details:**  
  Same structure as client operations, each configured for contacts.

- **Edge Cases:** Similar to clients (invalid IDs, validation, auth, API errors).

#### 1.5 Estimates Operations

- **Overview:**  
  Operations managing estimates: create, delete, get single/all, update.

- **Nodes Involved:**  
  - Create an estimate  
  - Delete an estimate  
  - Get data of an estimate  
  - Get data of all estimates  
  - Update an estimate

- **Node Details:**  
  Each node operates on estimate data entities with standard inputs and no further chained outputs.

- **Edge Cases:** Similar to previous blocks.

#### 1.6 Expenses Operations

- **Overview:**  
  Handle expense entries with full CRUD capabilities.

- **Nodes Involved:**  
  - Create an expense  
  - Delete an expense  
  - Get data of an expense  
  - Get data of all expenses  
  - Update an expense

- **Node Details:**  
  Typical CRUD nodes for expenses.

#### 1.7 Invoices Operations

- **Overview:**  
  Manage invoice-related operations: create, delete, get single/all, update.

- **Nodes Involved:**  
  - Create an invoice  
  - Delete an invoice  
  - Get data of an invoice  
  - Get data of all invoices  
  - Update an invoice

#### 1.8 Projects Operations

- **Overview:**  
  Operations on projects: create, delete, get single/all, update.

- **Nodes Involved:**  
  - Create a project  
  - Delete a project  
  - Get data of a project  
  - Get data of all projects  
  - Update a project

#### 1.9 Tasks Operations

- **Overview:**  
  Manage tasks including create, delete, get single/all, and update.

- **Nodes Involved:**  
  - Create a task  
  - Delete a task  
  - Get data of a task  
  - Get data of all tasks  
  - Update a task

#### 1.10 Time Entries Operations

- **Overview:**  
  Operations for time entries, including creation (via duration or start-end time), deletion, retrieval, restart, stop, and update.

- **Nodes Involved:**  
  - Create a time entry via duration  
  - Create a time entry via start and end time  
  - Delete a time entry  
  - Delete a time entry’s external reference  
  - Get data of a time entry  
  - Get data of all time entries  
  - Restart a time entry  
  - Stop a time entry  
  - Update a time entry

- **Edge Cases:**  
  - Time format validation errors for start/end time creation.  
  - Conflicts with overlapping time entries.  
  - Authorization issues.  
  - API limits.

#### 1.11 Users Operations

- **Overview:**  
  Manage Harvest user entities: create, delete, get single/all, get authenticated user data, and update.

- **Nodes Involved:**  
  - Create a user  
  - Delete a user  
  - Get data of a user  
  - Get data of all users  
  - Get data of authenticated user  
  - Update a user

- **Edge Cases:**  
  - User permission restrictions.  
  - Validation errors on user data.  
  - Auth errors.

#### 1.12 Sticky Notes

- **Overview:**  
  Visual sticky notes placed near groups of nodes, currently empty. They appear to serve as annotations or placeholders for documentation.

- **Nodes Involved:**  
  - Workflow Overview 0  
  - Sticky Note 1 through Sticky Note 10

- **Node Details:**  
  - Type: `stickyNote`  
  - Content: Empty strings, no additional info provided.  
  - Positions: Placed strategically near grouped Harvest nodes by entity types.

---

### 3. Summary Table

| Node Name                                 | Node Type                        | Functional Role                     | Input Node(s)           | Output Node(s)          | Sticky Note |
|-------------------------------------------|---------------------------------|-----------------------------------|------------------------|-------------------------|-------------|
| Workflow Overview 0                       | Sticky Note                     | Visual annotation (empty)          | None                   | None                    |             |
| Harvest Tool MCP Server                   | MCP Trigger                     | Entry trigger for all Harvest ops | None                   | All Harvest nodes       |             |
| Retrieve the company for the currently authenticated user | Harvest Tool                 | Get company info                   | Harvest Tool MCP Server | None                    |             |
| Create a client                          | Harvest Tool                    | Create client                     | Harvest Tool MCP Server | None                    |             |
| Delete a client                          | Harvest Tool                    | Delete client                     | Harvest Tool MCP Server | None                    |             |
| Get data of a client                     | Harvest Tool                    | Retrieve single client data       | Harvest Tool MCP Server | None                    |             |
| Get data of all clients                  | Harvest Tool                    | Retrieve all clients data         | Harvest Tool MCP Server | None                    |             |
| Update a client                          | Harvest Tool                    | Update client                    | Harvest Tool MCP Server | None                    |             |
| Sticky Note 1                           | Sticky Note                    | Visual annotation (empty)          | None                   | None                    |             |
| Create a contact                        | Harvest Tool                    | Create contact                   | Harvest Tool MCP Server | None                    |             |
| Delete a contact                        | Harvest Tool                    | Delete contact                   | Harvest Tool MCP Server | None                    |             |
| Get data of a contact                   | Harvest Tool                    | Retrieve single contact data     | Harvest Tool MCP Server | None                    |             |
| Get data of all contacts                | Harvest Tool                    | Retrieve all contacts data       | Harvest Tool MCP Server | None                    |             |
| Update a contact                       | Harvest Tool                    | Update contact                   | Harvest Tool MCP Server | None                    |             |
| Sticky Note 3                           | Sticky Note                    | Visual annotation (empty)          | None                   | None                    |             |
| Create an estimate                     | Harvest Tool                    | Create estimate                 | Harvest Tool MCP Server | None                    |             |
| Delete an estimate                     | Harvest Tool                    | Delete estimate                 | Harvest Tool MCP Server | None                    |             |
| Get data of an estimate                | Harvest Tool                    | Retrieve single estimate data   | Harvest Tool MCP Server | None                    |             |
| Get data of all estimates              | Harvest Tool                    | Retrieve all estimates data     | Harvest Tool MCP Server | None                    |             |
| Update an estimate                    | Harvest Tool                    | Update estimate                | Harvest Tool MCP Server | None                    |             |
| Sticky Note 4                           | Sticky Note                    | Visual annotation (empty)          | None                   | None                    |             |
| Create an expense                     | Harvest Tool                    | Create expense                | Harvest Tool MCP Server | None                    |             |
| Delete an expense                     | Harvest Tool                    | Delete expense                | Harvest Tool MCP Server | None                    |             |
| Get data of an expense                | Harvest Tool                    | Retrieve single expense data  | Harvest Tool MCP Server | None                    |             |
| Get data of all expenses              | Harvest Tool                    | Retrieve all expenses data    | Harvest Tool MCP Server | None                    |             |
| Update an expense                    | Harvest Tool                    | Update expense               | Harvest Tool MCP Server | None                    |             |
| Sticky Note 5                           | Sticky Note                    | Visual annotation (empty)          | None                   | None                    |             |
| Create an invoice                    | Harvest Tool                    | Create invoice               | Harvest Tool MCP Server | None                    |             |
| Delete an invoice                    | Harvest Tool                    | Delete invoice               | Harvest Tool MCP Server | None                    |             |
| Get data of an invoice               | Harvest Tool                    | Retrieve single invoice data | Harvest Tool MCP Server | None                    |             |
| Get data of all invoices             | Harvest Tool                    | Retrieve all invoices data   | Harvest Tool MCP Server | None                    |             |
| Update an invoice                   | Harvest Tool                    | Update invoice              | Harvest Tool MCP Server | None                    |             |
| Sticky Note 6                           | Sticky Note                    | Visual annotation (empty)          | None                   | None                    |             |
| Create a project                    | Harvest Tool                    | Create project              | Harvest Tool MCP Server | None                    |             |
| Delete a project                    | Harvest Tool                    | Delete project              | Harvest Tool MCP Server | None                    |             |
| Get data of a project               | Harvest Tool                    | Retrieve single project data | Harvest Tool MCP Server | None                    |             |
| Get data of all projects             | Harvest Tool                    | Retrieve all projects data   | Harvest Tool MCP Server | None                    |             |
| Update a project                   | Harvest Tool                    | Update project              | Harvest Tool MCP Server | None                    |             |
| Sticky Note 7                           | Sticky Note                    | Visual annotation (empty)          | None                   | None                    |             |
| Create a task                     | Harvest Tool                    | Create task                 | Harvest Tool MCP Server | None                    |             |
| Delete a task                     | Harvest Tool                    | Delete task                 | Harvest Tool MCP Server | None                    |             |
| Get data of a task                | Harvest Tool                    | Retrieve single task data   | Harvest Tool MCP Server | None                    |             |
| Get data of all tasks              | Harvest Tool                    | Retrieve all tasks data     | Harvest Tool MCP Server | None                    |             |
| Update a task                   | Harvest Tool                    | Update task                | Harvest Tool MCP Server | None                    |             |
| Sticky Note 8                           | Sticky Note                    | Visual annotation (empty)          | None                   | None                    |             |
| Create a time entry via duration | Harvest Tool                    | Create time entry by duration | Harvest Tool MCP Server | None                    |             |
| Create a time entry via start and end time | Harvest Tool                | Create time entry by start/end | Harvest Tool MCP Server | None                    |             |
| Delete a time entry             | Harvest Tool                    | Delete time entry           | Harvest Tool MCP Server | None                    |             |
| Delete a time entry’s external reference | Harvest Tool                | Delete external ref of time entry | Harvest Tool MCP Server | None                    |             |
| Get data of a time entry        | Harvest Tool                    | Retrieve single time entry  | Harvest Tool MCP Server | None                    |             |
| Get data of all time entries    | Harvest Tool                    | Retrieve all time entries   | Harvest Tool MCP Server | None                    |             |
| Restart a time entry            | Harvest Tool                    | Restart time entry          | Harvest Tool MCP Server | None                    |             |
| Stop a time entry               | Harvest Tool                    | Stop time entry             | Harvest Tool MCP Server | None                    |             |
| Update a time entry             | Harvest Tool                    | Update time entry           | Harvest Tool MCP Server | None                    |             |
| Sticky Note 9                           | Sticky Note                    | Visual annotation (empty)          | None                   | None                    |             |
| Create a user                   | Harvest Tool                    | Create user                 | Harvest Tool MCP Server | None                    |             |
| Delete a user                   | Harvest Tool                    | Delete user                 | Harvest Tool MCP Server | None                    |             |
| Get data of a user              | Harvest Tool                    | Retrieve single user data   | Harvest Tool MCP Server | None                    |             |
| Get data of all users          | Harvest Tool                    | Retrieve all users data     | Harvest Tool MCP Server | None                    |             |
| Get data of authenticated user | Harvest Tool                    | Retrieve authenticated user | Harvest Tool MCP Server | None                    |             |
| Update a user                  | Harvest Tool                    | Update user                | Harvest Tool MCP Server | None                    |             |
| Sticky Note 10                          | Sticky Note                    | Visual annotation (empty)          | None                   | None                    |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the MCP Trigger Node**  
   - Add a node of type `@n8n/n8n-nodes-langchain.mcpTrigger`.  
   - Set up a webhook (default or custom) with a unique webhook ID (e.g., `80f8e84d-8ffc-4d53-9f93-8d4bdc6bfb83`).  
   - This node acts as the entry point for all operations.

2. **Set Up Harvest Tool Credentials**  
   - In n8n, configure Harvest Tool API credentials (OAuth2 or API token as required).  
   - Assign this credential to all Harvest Tool nodes created below.

3. **Create Harvest Tool Nodes for Each Operation**  
   For each entity type below, add Harvest Tool nodes for all CRUD operations listed:

   - **Clients:**  
     - Create a client (Create)  
     - Delete a client (Delete)  
     - Get data of a client (Retrieve one)  
     - Get data of all clients (Retrieve all)  
     - Update a client (Update)

   - **Contacts:**  
     Same operations as clients.

   - **Estimates:**  
     Same operations as clients.

   - **Expenses:**  
     Same operations as clients.

   - **Invoices:**  
     Same operations as clients.

   - **Projects:**  
     Same operations as clients.

   - **Tasks:**  
     Same operations as clients.

   - **Time Entries:**  
     - Create a time entry via duration  
     - Create a time entry via start and end time  
     - Delete a time entry  
     - Delete a time entry’s external reference  
     - Get data of a time entry  
     - Get data of all time entries  
     - Restart a time entry  
     - Stop a time entry  
     - Update a time entry

   - **Users:**  
     - Create a user  
     - Delete a user  
     - Get data of a user  
     - Get data of all users  
     - Get data of authenticated user  
     - Update a user

   - **Organizational Info:**  
     - Retrieve the company for the currently authenticated user

4. **Configure Each Harvest Tool Node**  
   - Set the operation type (Create, Delete, Get, Update).  
   - Specify required parameters (e.g., client ID for delete/get/update).  
   - Leave optional parameters unset or set defaults as per Harvest API.  
   - Use the Harvest Tool credentials configured earlier.

5. **Connect All Harvest Tool Nodes to the MCP Trigger Node**  
   - Connect the output of the MCP Trigger to the input of each Harvest Tool node on the `ai_tool` channel (or the main input).  
   - This design allows the MCP trigger to route incoming commands to the appropriate operation node.

6. **Add Sticky Notes (Optional)**  
   - For better organization, add sticky notes near groups of nodes to label logical blocks (e.g., "Clients Operations", "Contacts Operations").  
   - Currently, sticky notes are empty but can be used for documentation purposes.

7. **Set Workflow Settings**  
   - Set workflow timezone to `America/New_York` to match original if relevant.  
   - Activate the workflow and test each operation with proper inputs.

---

### 5. General Notes & Resources

| Note Content                                                                                             | Context or Link                                         |
|---------------------------------------------------------------------------------------------------------|--------------------------------------------------------|
| The workflow employs a single MCP Trigger node to centralize all Harvest Tool API operations.            | Architectural design choice for easy external calls.   |
| Each Harvest Tool node operates independently, allowing modular handling of all 51 Harvest operations.    | Ensures maintainability and extensibility.             |
| Sticky Notes are currently empty but can be used to add documentation or instructions on usage.          | Recommended for workflow maintainers.                   |
| Workflow timezone set to America/New_York, adjust if needed for other locales.                           | Workflow general setting.                               |
| Ensure Harvest Tool credentials used have sufficient permissions for all operations to avoid auth errors. | Credential setup best practice.                         |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.