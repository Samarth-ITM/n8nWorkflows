🛠️ Affinity Tool MCP Server 💪 all 16 operations

https://n8nworkflows.xyz/workflows/----affinity-tool-mcp-server----all-16-operations-5335


# 🛠️ Affinity Tool MCP Server 💪 all 16 operations

### 1. Workflow Overview

This workflow, titled **"🛠️ Affinity Tool MCP Server 💪 all 16 operations"**, serves as a comprehensive MCP (Multi-Channel Platform) server interface for the Affinity Tool API within n8n. It enables all 16 core operations supported by the Affinity Tool, providing CRUD (Create, Read, Update, Delete) functionalities for three primary entities: lists, organizations, and people, as well as list entries.

The workflow is logically divided into the following blocks based on entity and operation type:

- **1.1 MCP Trigger Input Reception:** Receives and triggers workflow execution from an external MCP interface.
- **1.2 List Operations:** Handles fetching single/multiple lists and manipulating list entries.
- **1.3 Organization Operations:** Supports creating, reading, updating, and deleting organizations.
- **1.4 Person Operations:** Supports creating, reading, updating, and deleting persons.

Each block consists of Affinity Tool nodes configured for specific API operations, all triggered from the central MCP trigger node.

---

### 2. Block-by-Block Analysis

#### 2.1 MCP Trigger Input Reception

**Overview:**  
This block serves as the entry point for the workflow. It listens for incoming MCP requests and routes them to the appropriate Affinity Tool operation nodes.

**Nodes Involved:**  
- Affinity Tool MCP Server

**Node Details:**  
- **Node:** Affinity Tool MCP Server  
- **Type:** `@n8n/n8n-nodes-langchain.mcpTrigger`  
- **Role:** Initiates workflow execution on receiving MCP requests, acting as an API endpoint webhook.  
- **Configuration:** Uses a unique webhook ID to receive external triggers. No additional parameters configured.  
- **Input:** External MCP API requests.  
- **Output:** Routes to all 16 Affinity Tool operation nodes via the `ai_tool` connection type.  
- **Version Requirements:** Requires n8n version supporting MCP trigger nodes and Affinity Tool integration.  
- **Potential Failures:** Webhook misconfiguration, invalid request payloads, or MCP client communication errors.  
- **Sub-workflow:** None.

---

#### 2.2 List Operations

**Overview:**  
Manages list-related operations, including retrieving single or multiple lists and managing list entries (create, get, delete, get many).

**Nodes Involved:**  
- Get a list  
- Get many lists  
- Create a list entry  
- Get a list entry  
- Get many list entries  
- Delete a list entry  
- Sticky Note 1  
- Sticky Note 2

**Node Details:**

- **Node:** Get a list  
  - **Type:** `n8n-nodes-base.affinityTool`  
  - **Role:** Fetches details for a single list by ID.  
  - **Configuration:** Parameters set to retrieve a specific list entity.  
  - **Input:** Triggered from MCP Server node.  
  - **Output:** List data output.  
  - **Potential Failures:** Invalid list ID, API authentication errors, network timeouts.

- **Node:** Get many lists  
  - **Type:** `n8n-nodes-base.affinityTool`  
  - **Role:** Retrieves multiple lists, supports pagination or filtering.  
  - **Configuration:** Parameters may include filters or paging options.  
  - **Input/Output:** Connected to MCP Server.  
  - **Potential Failures:** API limits exceeded, response parsing errors.

- **Node:** Create a list entry  
  - **Type:** `n8n-nodes-base.affinityTool`  
  - **Role:** Adds a new entry to a specified list.  
  - **Configuration:** Requires list ID and entry data.  
  - **Input/Output:** From MCP Server, outputs created entry details.  
  - **Potential Failures:** Invalid data, missing required fields.

- **Node:** Get a list entry  
  - **Type:** `n8n-nodes-base.affinityTool`  
  - **Role:** Retrieves a specific list entry by ID.  
  - **Configuration:** Entry and list IDs required.  
  - **Potential Failures:** Entry not found, auth errors.

- **Node:** Get many list entries  
  - **Type:** `n8n-nodes-base.affinityTool`  
  - **Role:** Fetches multiple entries from a list.  
  - **Configuration:** Supports filters, paging.  
  - **Potential Failures:** Large data sets causing timeouts.

- **Node:** Delete a list entry  
  - **Type:** `n8n-nodes-base.affinityTool`  
  - **Role:** Deletes a specific list entry.  
  - **Configuration:** Needs entry ID.  
  - **Potential Failures:** Trying to delete non-existent entry.

- **Sticky Notes:** Positioned near list nodes, currently empty, intended for user notes or instructions.

---

#### 2.3 Organization Operations

**Overview:**  
Enables full CRUD operations on organizations within Affinity Tool: creating, retrieving (single/multiple), updating, and deleting organizations.

**Nodes Involved:**  
- Create an organization  
- Get an organization  
- Get many organizations  
- Update an organization  
- Delete an organization  
- Sticky Note 3

**Node Details:**

- **Node:** Create an organization  
  - **Type:** `n8n-nodes-base.affinityTool`  
  - **Role:** Adds a new organization entity.  
  - **Configuration:** Requires organization data fields.  
  - **Potential Failures:** Data validation errors.

- **Node:** Get an organization  
  - **Type:** `n8n-nodes-base.affinityTool`  
  - **Role:** Retrieves an organization by ID.  
  - **Potential Failures:** Not found, auth errors.

- **Node:** Get many organizations  
  - **Type:** `n8n-nodes-base.affinityTool`  
  - **Role:** Retrieves multiple organizations, supports filters/page.  
  - **Potential Failures:** API limits, large responses.

- **Node:** Update an organization  
  - **Type:** `n8n-nodes-base.affinityTool`  
  - **Role:** Modifies an existing organization.  
  - **Configuration:** Requires org ID and update data.  
  - **Potential Failures:** Conflicts, invalid input.

- **Node:** Delete an organization  
  - **Type:** `n8n-nodes-base.affinityTool`  
  - **Role:** Removes an organization from the system.  
  - **Potential Failures:** Deleting non-existent org.

- **Sticky Note 3:** Located near organization nodes, currently empty.

---

#### 2.4 Person Operations

**Overview:**  
Handles CRUD operations on person entities, including creating, retrieving (single/multiple), updating, and deleting persons.

**Nodes Involved:**  
- Create a person  
- Get a person  
- Get many people  
- Update a person  
- Delete a person  
- Sticky Note 4

**Node Details:**

- **Node:** Create a person  
  - **Type:** `n8n-nodes-base.affinityTool`  
  - **Role:** Creates a new person record.  
  - **Configuration:** Requires person data.  
  - **Potential Failures:** Invalid or incomplete data.

- **Node:** Get a person  
  - **Type:** `n8n-nodes-base.affinityTool`  
  - **Role:** Retrieves a person by ID.  
  - **Potential Failures:** Not found, auth issues.

- **Node:** Get many people  
  - **Type:** `n8n-nodes-base.affinityTool`  
  - **Role:** Retrieves multiple person records.  
  - **Potential Failures:** Pagination or large data handling.

- **Node:** Update a person  
  - **Type:** `n8n-nodes-base.affinityTool`  
  - **Role:** Updates person details.  
  - **Configuration:** Person ID and new data needed.  
  - **Potential Failures:** Data conflicts.

- **Node:** Delete a person  
  - **Type:** `n8n-nodes-base.affinityTool`  
  - **Role:** Deletes a person record.  
  - **Potential Failures:** Deleting non-existent person.

- **Sticky Note 4:** Positioned near person nodes, empty.

---

### 3. Summary Table

| Node Name               | Node Type                              | Functional Role                              | Input Node(s)           | Output Node(s)          | Sticky Note |
|-------------------------|--------------------------------------|----------------------------------------------|-------------------------|-------------------------|-------------|
| Workflow Overview 0     | n8n-nodes-base.stickyNote             | Placeholder for overview notes                |                         |                         |             |
| Affinity Tool MCP Server| @n8n/n8n-nodes-langchain.mcpTrigger  | Entry point - receives MCP API triggers       |                         | All Affinity Tool nodes |             |
| Get a list              | n8n-nodes-base.affinityTool           | Retrieve single list by ID                     | Affinity Tool MCP Server |                         |             |
| Get many lists          | n8n-nodes-base.affinityTool           | Retrieve multiple lists                        | Affinity Tool MCP Server |                         |             |
| Sticky Note 1           | n8n-nodes-base.stickyNote             | User notes near list nodes                      |                         |                         |             |
| Create a list entry     | n8n-nodes-base.affinityTool           | Add new list entry                            | Affinity Tool MCP Server |                         |             |
| Delete a list entry     | n8n-nodes-base.affinityTool           | Remove a list entry                           | Affinity Tool MCP Server |                         |             |
| Get a list entry        | n8n-nodes-base.affinityTool           | Retrieve a specific list entry                | Affinity Tool MCP Server |                         |             |
| Get many list entries   | n8n-nodes-base.affinityTool           | Retrieve multiple list entries                 | Affinity Tool MCP Server |                         |             |
| Sticky Note 2           | n8n-nodes-base.stickyNote             | User notes near list entry nodes               |                         |                         |             |
| Create an organization  | n8n-nodes-base.affinityTool           | Create organization record                     | Affinity Tool MCP Server |                         |             |
| Delete an organization  | n8n-nodes-base.affinityTool           | Delete organization by ID                      | Affinity Tool MCP Server |                         |             |
| Get an organization     | n8n-nodes-base.affinityTool           | Retrieve organization by ID                    | Affinity Tool MCP Server |                         |             |
| Get many organizations  | n8n-nodes-base.affinityTool           | Retrieve multiple organizations                | Affinity Tool MCP Server |                         |             |
| Update an organization  | n8n-nodes-base.affinityTool           | Update organization data                       | Affinity Tool MCP Server |                         |             |
| Sticky Note 3           | n8n-nodes-base.stickyNote             | User notes near organization nodes             |                         |                         |             |
| Create a person         | n8n-nodes-base.affinityTool           | Create new person record                        | Affinity Tool MCP Server |                         |             |
| Delete a person         | n8n-nodes-base.affinityTool           | Delete person by ID                             | Affinity Tool MCP Server |                         |             |
| Get a person            | n8n-nodes-base.affinityTool           | Retrieve person by ID                           | Affinity Tool MCP Server |                         |             |
| Get many people         | n8n-nodes-base.affinityTool           | Retrieve multiple persons                       | Affinity Tool MCP Server |                         |             |
| Update a person         | n8n-nodes-base.affinityTool           | Update person data                             | Affinity Tool MCP Server |                         |             |
| Sticky Note 4           | n8n-nodes-base.stickyNote             | User notes near person nodes                    |                         |                         |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the MCP Trigger Node:**  
   - Add the node `Affinity Tool MCP Server` of type `@n8n/n8n-nodes-langchain.mcpTrigger`.  
   - Configure it with a unique webhook ID (auto-generated or custom).  
   - This node will serve as the webhook entry point to receive MCP requests.

2. **Create List Operation Nodes:**  
   - Add nodes of type `Affinity Tool` for the following operations:  
     - `Get a list` (fetch one list)  
     - `Get many lists` (fetch multiple lists)  
     - `Create a list entry` (add an entry to a list)  
     - `Get a list entry` (fetch a specific list entry)  
     - `Get many list entries` (fetch multiple list entries)  
     - `Delete a list entry` (remove a list entry)  
   - For each node, configure the operation type accordingly in the node parameters.  
   - Set required input parameters like list IDs or entry IDs, either as expressions or placeholders to be dynamically filled by the MCP trigger.  
   - Connect the `ai_tool` output from the MCP trigger node to each of these nodes.

3. **Create Organization Operation Nodes:**  
   - Add `Affinity Tool` nodes for:  
     - `Create an organization`  
     - `Get an organization`  
     - `Get many organizations`  
     - `Update an organization`  
     - `Delete an organization`  
   - Configure each node’s parameters for the respective CRUD operation.  
   - Ensure input parameters for org IDs or data payloads are set dynamically or via expressions.  
   - Connect these nodes to the MCP trigger’s `ai_tool` output.

4. **Create Person Operation Nodes:**  
   - Add `Affinity Tool` nodes for:  
     - `Create a person`  
     - `Get a person`  
     - `Get many people`  
     - `Update a person`  
     - `Delete a person`  
   - Configure each node’s parameters for person data and IDs as required.  
   - Connect these nodes to the MCP trigger node’s output.

5. **Add Sticky Notes (Optional):**  
   - Add sticky note nodes near each logical group for documentation or instructions, leaving them blank or adding user comments as needed.

6. **Credential Setup:**  
   - Configure the Affinity Tool API credentials in n8n credentials manager.  
   - Assign these credentials to each Affinity Tool node to authorize API calls.

7. **Workflow Settings:**  
   - Set the timezone to “America/New_York” or as preferred.  
   - Save and activate the workflow.

8. **Testing:**  
   - Trigger the workflow via the MCP API webhook with various operation requests.  
   - Verify that each Affinity Tool node processes data and returns correct responses.

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                          |
|------------------------------------------------------------------------------------------------|----------------------------------------|
| This workflow covers all 16 core operations of the Affinity Tool API via an MCP webhook trigger. | Workflow description                    |
| Affinity Tool API documentation can provide detailed parameter and data schema references.      | https://docs.affinity.co/api/           |
| MCP Trigger node requires n8n version supporting Langchain MCP integration.                     | n8n documentation on MCP nodes          |

---

**Disclaimer:** The provided content is exclusively derived from an n8n automated workflow and adheres strictly to content policies. No illegal, offensive, or protected materials are included. All data manipulated is legal and public.