🛠️ Grafana Tool MCP Server 💪 all 16 operations

https://n8nworkflows.xyz/workflows/----grafana-tool-mcp-server----all-16-operations-5245


# 🛠️ Grafana Tool MCP Server 💪 all 16 operations

### 1. Workflow Overview

This workflow, titled **"🛠️ Grafana Tool MCP Server 💪 all 16 operations"**, is designed to serve as a robust API interface for managing Grafana resources via n8n. It exposes 16 distinct operations covering dashboards, teams, team members, and users within a Grafana instance. The workflow is triggered by an MCP trigger node, which acts as the entry point to route requests to specific operations.

**Target use cases**:  
- Automating Grafana dashboard lifecycle management (create, update, delete, retrieve single or multiple dashboards)  
- Managing Grafana teams and their members (create, update, delete teams; add/remove team members; retrieve team info)  
- Managing Grafana users (update, delete, retrieve multiple users)  
- Serving as a centralized Grafana API proxy with node-based orchestration in n8n  

**Logical blocks** grouped by functional domain and direct dependencies:  

- **1.1 Trigger and Request Reception**  
  - Entry point that receives external requests and routes them internally.

- **1.2 Dashboard Management Operations**  
  - Nodes handling create, update, delete, and retrieval of dashboards.

- **1.3 Team Management Operations**  
  - Nodes managing creation, update, deletion, and retrieval of teams.

- **1.4 Team Member Management Operations**  
  - Nodes handling addition, removal, and listing of team members.

- **1.5 User Management Operations**  
  - Nodes managing deletion, update, and retrieval of users.

- **1.6 Sticky Notes**  
  - Notes associated with logical blocks, currently empty placeholders.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger and Request Reception

- **Overview:**  
  This block contains the single entry point node that triggers the workflow upon receiving a request. It serves as the dispatcher, routing incoming requests to the appropriate Grafana operation node.

- **Nodes Involved:**  
  - Grafana Tool MCP Server (MCP Trigger node)

- **Node Details:**  
  - **Node Name:** Grafana Tool MCP Server  
  - **Type:** MCP Trigger (from `@n8n/n8n-nodes-langchain.mcpTrigger` package)  
  - **Configuration:** No additional parameters; uses a webhook ID to receive external calls.  
  - **Expressions/Variables:** None explicitly configured here; acts as a trigger.  
  - **Input Connections:** None (trigger node)  
  - **Output Connections:** Routes outputs to all operation nodes handling dashboards, teams, team members, and users.  
  - **Version Requirements:** Requires n8n version supporting MCP trigger node and the `@n8n/n8n-nodes-langchain` package.  
  - **Edge Cases / Failure Types:**  
    - Webhook misconfiguration or network issues could prevent triggering.  
    - Unauthorized or malformed requests could cause failures downstream.  
  - **Sub-workflow:** None.

---

#### 1.2 Dashboard Management Operations

- **Overview:**  
  This block manages all operations related to Grafana dashboards, including creating, updating, deleting, and retrieving dashboards (single or multiple).

- **Nodes Involved:**  
  - Create a dashboard  
  - Update a dashboard  
  - Delete a dashboard  
  - Get a dashboard  
  - Get many dashboards  
  - Sticky Note 1 (empty)

- **Node Details:**  
  For all these nodes:  
  - **Type:** Grafana Tool node (`n8n-nodes-base.grafanaTool`)  
  - **Configuration:** Each node is pre-configured to perform a specific dashboard operation by setting the appropriate API endpoint and method internally in the Grafana Tool node. No explicit parameters are shown in the exported JSON, implying default or dynamically injected parameters at runtime.  
  - **Expressions/Variables:** None explicitly shown; likely uses input data passed from the trigger node for resource identifiers or payloads.  
  - **Input Connections:** Each node receives input from the MCP Trigger node under the "ai_tool" connection type.  
  - **Output Connections:** None (end nodes for these operations).  
  - **Version Requirements:** Requires the Grafana Tool node available in n8n.  
  - **Edge Cases / Failure Types:**  
    - API authentication failures or expired tokens.  
    - Invalid dashboard IDs or payloads causing API errors.  
    - Network timeouts or Grafana server unavailability.  
    - Missing or malformed request data leading to operation failure.  
  - **Sub-workflow:** None.

---

#### 1.3 Team Management Operations

- **Overview:**  
  This block handles team-related operations: creating, updating, deleting teams, and fetching one or many teams.

- **Nodes Involved:**  
  - Create a team  
  - Update a team  
  - Delete a team  
  - Get a team  
  - Get many teams  
  - Sticky Note 2 (empty)

- **Node Details:**  
  Similar to the dashboard nodes:  
  - **Type:** Grafana Tool node  
  - **Configuration:** Each node corresponds to a specific team API action. Parameters are expected to be injected dynamically from the trigger input.  
  - **Expressions/Variables:** Not explicitly detailed; uses workflow context or input data.  
  - **Input Connections:** From MCP Trigger node.  
  - **Output Connections:** None.  
  - **Version Requirements:** Grafana Tool node availability.  
  - **Edge Cases / Failure Types:**  
    - Authentication and permission errors.  
    - Invalid team IDs or data.  
    - API rate limits or server errors.  
  - **Sub-workflow:** None.

---

#### 1.4 Team Member Management Operations

- **Overview:**  
  This block manages team members by adding, removing, and listing members within teams.

- **Nodes Involved:**  
  - Add a team member  
  - Remove a team member  
  - Get many team members  
  - Sticky Note 3 (empty)

- **Node Details:**  
  - **Type:** Grafana Tool node  
  - **Configuration:** Each node is configured to execute a specific team member API endpoint.  
  - **Expressions/Variables:** Expected to receive user and team identifiers from input data.  
  - **Input Connections:** From MCP Trigger node.  
  - **Output Connections:** None.  
  - **Version Requirements:** Grafana Tool node.  
  - **Edge Cases / Failure Types:**  
    - Adding a member who already exists or removing a non-existent member.  
    - Permission or authentication failures.  
    - Invalid user or team IDs.  
  - **Sub-workflow:** None.

---

#### 1.5 User Management Operations

- **Overview:**  
  This block handles user-level operations including deleting users, updating user details, and retrieving multiple users.

- **Nodes Involved:**  
  - Delete a user  
  - Update a user  
  - Get many users  
  - Sticky Note 4 (empty)

- **Node Details:**  
  - **Type:** Grafana Tool node  
  - **Configuration:** Each node corresponds to a Grafana user API endpoint for the respective operation.  
  - **Expressions/Variables:** Dynamic parameters for user IDs and update data expected from incoming requests.  
  - **Input Connections:** From MCP Trigger node.  
  - **Output Connections:** None.  
  - **Version Requirements:** Grafana Tool node.  
  - **Edge Cases / Failure Types:**  
    - Attempting to delete or update non-existent users.  
    - Permission or authentication failures.  
    - API errors due to invalid payloads.  
  - **Sub-workflow:** None.

---

#### 1.6 Sticky Notes

- **Overview:**  
  These are purely informational notes associated visually with the respective operation blocks. All currently contain empty content and serve as placeholders for future documentation or instructions.

- **Nodes Involved:**  
  - Workflow Overview 0  
  - Sticky Note 1 (Dashboards)  
  - Sticky Note 2 (Teams)  
  - Sticky Note 3 (Team Members)  
  - Sticky Note 4 (Users)

- **Node Details:**  
  - **Type:** Sticky Note (`n8n-nodes-base.stickyNote`)  
  - **Configuration:** Empty content, positioned for visual grouping.  
  - **Connections:** None.  
  - **Edge Cases:** None.

---

### 3. Summary Table

| Node Name             | Node Type                        | Functional Role                     | Input Node(s)           | Output Node(s)          | Sticky Note                  |
|-----------------------|---------------------------------|-----------------------------------|------------------------|-------------------------|------------------------------|
| Workflow Overview 0   | Sticky Note                     | Placeholder overview note          | None                   | None                    |                              |
| Grafana Tool MCP Server | MCP Trigger                    | Entry point, routes requests       | None                   | All operation nodes      |                              |
| Create a dashboard     | Grafana Tool                   | Dashboard creation                 | Grafana Tool MCP Server | None                    |                              |
| Delete a dashboard     | Grafana Tool                   | Dashboard deletion                 | Grafana Tool MCP Server | None                    |                              |
| Get a dashboard        | Grafana Tool                   | Retrieve single dashboard          | Grafana Tool MCP Server | None                    |                              |
| Get many dashboards    | Grafana Tool                   | Retrieve multiple dashboards       | Grafana Tool MCP Server | None                    |                              |
| Update a dashboard     | Grafana Tool                   | Update dashboard                   | Grafana Tool MCP Server | None                    |                              |
| Sticky Note 1          | Sticky Note                   | Placeholder note for dashboards    | None                   | None                    |                              |
| Create a team          | Grafana Tool                   | Team creation                     | Grafana Tool MCP Server | None                    |                              |
| Delete a team          | Grafana Tool                   | Team deletion                     | Grafana Tool MCP Server | None                    |                              |
| Get a team             | Grafana Tool                   | Retrieve single team               | Grafana Tool MCP Server | None                    |                              |
| Get many teams         | Grafana Tool                   | Retrieve multiple teams            | Grafana Tool MCP Server | None                    |                              |
| Update a team          | Grafana Tool                   | Update team                       | Grafana Tool MCP Server | None                    |                              |
| Sticky Note 2          | Sticky Note                   | Placeholder note for teams         | None                   | None                    |                              |
| Add a team member      | Grafana Tool                   | Add member to team                 | Grafana Tool MCP Server | None                    |                              |
| Get many team members  | Grafana Tool                   | List team members                  | Grafana Tool MCP Server | None                    |                              |
| Remove a team member   | Grafana Tool                   | Remove member from team            | Grafana Tool MCP Server | None                    |                              |
| Sticky Note 3          | Sticky Note                   | Placeholder note for team members  | None                   | None                    |                              |
| Delete a user          | Grafana Tool                   | User deletion                     | Grafana Tool MCP Server | None                    |                              |
| Get many users         | Grafana Tool                   | List users                       | Grafana Tool MCP Server | None                    |                              |
| Update a user          | Grafana Tool                   | Update user details               | Grafana Tool MCP Server | None                    |                              |
| Sticky Note 4          | Sticky Note                   | Placeholder note for users         | None                   | None                    |                              |

---

### 4. Reproducing the Workflow from Scratch

**Step 1:** Create the Trigger Node  
- Add an **MCP Trigger** node named "Grafana Tool MCP Server".  
- Configure webhook settings as needed to expose an endpoint for external calls. No special parameters are required. This node will act as the workflow’s entry point.

**Step 2:** Add Dashboard Operation Nodes  
- Create five **Grafana Tool** nodes named as follows:  
  - "Create a dashboard"  
  - "Delete a dashboard"  
  - "Get a dashboard"  
  - "Get many dashboards"  
  - "Update a dashboard"  
- For each node, configure the Grafana Tool with the corresponding dashboard API operation. This typically involves selecting the API path and HTTP method related to dashboards in the node's settings.  
- Connect each node's input to the MCP Trigger node’s output using "ai_tool" connection.

**Step 3:** Add Team Operation Nodes  
- Create five **Grafana Tool** nodes named:  
  - "Create a team"  
  - "Delete a team"  
  - "Get a team"  
  - "Get many teams"  
  - "Update a team"  
- Configure each node for the respective team API endpoint for create, delete, get (single), get (many), update.  
- Connect inputs from the MCP Trigger node under "ai_tool".

**Step 4:** Add Team Member Operation Nodes  
- Create three **Grafana Tool** nodes named:  
  - "Add a team member"  
  - "Get many team members"  
  - "Remove a team member"  
- Configure for the respective team member API endpoints.  
- Connect inputs from the MCP Trigger node.

**Step 5:** Add User Operation Nodes  
- Create three **Grafana Tool** nodes named:  
  - "Delete a user"  
  - "Get many users"  
  - "Update a user"  
- Configure for the respective user API endpoints.  
- Connect inputs from the MCP Trigger node.

**Step 6:** Add Sticky Notes (Optional)  
- Add five **Sticky Note** nodes placed visually near each logical block:  
  - One for Workflow Overview (top-left).  
  - One each for Dashboard, Team, Team Member, and User blocks.  
- Leave content empty or add descriptive notes as needed.

**Step 7:** Credential Setup  
- For all Grafana Tool nodes, configure the Grafana credentials with:  
  - Grafana URL.  
  - API authentication token or other authentication method supported by the Grafana Tool node.  
- Ensure credentials are valid and have sufficient permissions for all intended operations.

**Step 8:** Test and Validate  
- Trigger the workflow via the MCP trigger webhook with various payloads to invoke different operations.  
- Verify that each Grafana Tool node correctly performs its respective API call and handles errors gracefully.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| This workflow demonstrates the usage of the n8n Grafana Tool node to perform a comprehensive suite of Grafana API operations via a single MCP trigger. | Workflow title and structure |
| MCP Trigger node requires n8n versions supporting the `@n8n/n8n-nodes-langchain` package for multi-command processing. | Node versioning note |
| Grafana API tokens must have adequate permissions for dashboards, teams, team members, and users management. | Credential setup advice |
| Sticky Notes are placeholders intended for documentation or process explanation in the workflow canvas. | UI usage note |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created in n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.