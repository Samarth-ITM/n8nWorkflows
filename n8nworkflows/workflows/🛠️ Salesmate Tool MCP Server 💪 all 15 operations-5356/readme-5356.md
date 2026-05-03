🛠️ Salesmate Tool MCP Server 💪 all 15 operations

https://n8nworkflows.xyz/workflows/----salesmate-tool-mcp-server----all-15-operations-5356


# 🛠️ Salesmate Tool MCP Server 💪 all 15 operations

### 1. Workflow Overview

This workflow, titled **"Salesmate Tool MCP Server"**, is designed as a multi-conditional processor (MCP) server to handle various Salesmate CRM operations via a single webhook trigger. It exposes 15 distinct Salesmate operations related to three main entities: Activities, Companies, and Deals. Each operation corresponds to a CRUD (Create, Read, Update, Delete) or list action.

The workflow is logically divided into the following blocks based on entity and operation type:

- **1.1 MCP Trigger Input Reception:** Receives incoming requests to trigger specific Salesmate operations.
- **1.2 Activities Operations Block:** Handles creation, retrieval (single and multiple), update, and deletion of Activities.
- **1.3 Companies Operations Block:** Handles creation, retrieval (single and multiple), update, and deletion of Companies.
- **1.4 Deals Operations Block:** Handles creation, retrieval (single and multiple), update, and deletion of Deals.
- **1.5 Sticky Notes:** Visual annotations grouping the nodes logically by entity type for clarity.

This structure enables centralized handling of multiple Salesmate API operations, simplifying integration with other systems by routing all requests through one MCP trigger node.

---

### 2. Block-by-Block Analysis

#### 2.1 MCP Trigger Input Reception

- **Overview:**  
  This block waits for incoming webhook requests and routes them to the appropriate Salesmate operation node based on the requested operation.

- **Nodes Involved:**  
  - Salesmate Tool MCP Server (MCP Trigger)

- **Node Details:**

  - **Node Name:** Salesmate Tool MCP Server  
    - **Type:** `@n8n/n8n-nodes-langchain.mcpTrigger` (Multi-Conditional Processor Trigger)  
    - **Role:** Entry point that listens for incoming webhook requests specifying which Salesmate operation to perform.  
    - **Configuration:** Uses a webhook ID to uniquely identify the trigger URL. No additional parameters needed; dynamically routes to child nodes based on input.  
    - **Input:** External webhook calls with operation commands.  
    - **Output:** Routes data to one of the 15 Salesmate Tool nodes based on the operation requested.  
    - **Version Requirements:** Requires n8n version supporting MCP Trigger nodes and the Salesmate Tool nodes.  
    - **Potential Failures:** Invalid or missing webhook call, malformed operation request, routing errors if operation name does not match any child node.  
    - **Sub-workflow:** None.

#### 2.2 Activities Operations Block

- **Overview:**  
  Handles all Salesmate activity-related operations: create, get (single), get many, update, and delete.

- **Nodes Involved:**  
  - Create an activity  
  - Get an activity  
  - Get many activities  
  - Update an activity  
  - Delete an activity  
  - Sticky Note 1 (visual grouping, no logic)

- **Node Details (common for all Salesmate Tool nodes):**

  Each node is of type `n8n-nodes-base.salesmateTool` configured to a specific Salesmate activity operation.

  - **Create an activity:**  
    - Role: Creates a new activity record in Salesmate.  
    - Key config: Parameters to specify activity details (subject, due date, etc.) expected in input.  
    - Input from MCP Trigger node.  
    - Output: Created activity data.  
    - Failures: API authentication errors, invalid data, rate limits.

  - **Get an activity:**  
    - Role: Retrieves a single activity by ID.  
    - Key config: Requires activity ID from input.  
    - Output: Activity details.  
    - Failures: Not found errors, invalid IDs.

  - **Get many activities:**  
    - Role: Retrieves multiple activities with optional filters.  
    - Key config: Pagination and filter parameters.  
    - Output: List of activities.  
    - Failures: Large data volume issues, API limits.

  - **Update an activity:**  
    - Role: Updates properties of an existing activity.  
    - Key config: Activity ID and fields to update.  
    - Output: Updated activity data.  
    - Failures: Not found, invalid update fields.

  - **Delete an activity:**  
    - Role: Deletes an activity by ID.  
    - Key config: Activity ID.  
    - Output: Confirmation of deletion.  
    - Failures: Not found, permission errors.

  - **Sticky Note 1:**  
    - No functional role, used to visually group this block.

#### 2.3 Companies Operations Block

- **Overview:**  
  Handles all Salesmate company-related operations: create, get (single), get many, update, and delete.

- **Nodes Involved:**  
  - Create a company  
  - Get a company  
  - Get many companies  
  - Update a company  
  - Delete a company  
  - Sticky Note 2 (visual grouping)

- **Node Details:**

  Similar to Activities block, each node is a `salesmateTool` node configured for company operations.

  - **Create a company:** Creates new company record with input data.  
  - **Get a company:** Retrieves company by ID.  
  - **Get many companies:** Lists companies with filters/pagination.  
  - **Update a company:** Updates company details by ID.  
  - **Delete a company:** Removes company by ID.  
  - **Sticky Note 2:** Visual grouping.

  Potential failures and input/output patterns match those described in Activities block.

#### 2.4 Deals Operations Block

- **Overview:**  
  Handles all Salesmate deal-related operations: create, get (single), get many, update, and delete.

- **Nodes Involved:**  
  - Create a deal  
  - Get a deal  
  - Get many deals  
  - Update a deal  
  - Delete a deal  
  - Sticky Note 3 (visual grouping)

- **Node Details:**

  Each node is a `salesmateTool` node tailored for deal operations.

  - **Create a deal:** Creates new deal record.  
  - **Get a deal:** Retrieve deal by ID.  
  - **Get many deals:** List deals with optional filters.  
  - **Update a deal:** Modify deal fields by ID.  
  - **Delete a deal:** Remove deal by ID.  
  - **Sticky Note 3:** Visual grouping.

  Input/output and error considerations follow the same pattern.

#### 2.5 Sticky Notes Overview

- **Nodes:**  
  - Workflow Overview 0 (empty)  
  - Sticky Note 1 (Activities group)  
  - Sticky Note 2 (Companies group)  
  - Sticky Note 3 (Deals group)

- **Role:**  
  Provide visual structure in the editor to ease navigation and understanding of grouped Salesmate operations.

---

### 3. Summary Table

| Node Name               | Node Type                  | Functional Role                        | Input Node(s)              | Output Node(s)            | Sticky Note                      |
|-------------------------|----------------------------|-------------------------------------|----------------------------|---------------------------|---------------------------------|
| Workflow Overview 0     | Sticky Note                | Visual overview placeholder          | -                          | -                         |                                 |
| Salesmate Tool MCP Server | MCP Trigger                | Receives webhook, routes operations | -                          | All Salesmate Tool nodes  |                                 |
| Create an activity      | Salesmate Tool             | Creates a Salesmate activity         | Salesmate Tool MCP Server  | -                         | Sticky Note 1                   |
| Delete an activity      | Salesmate Tool             | Deletes a Salesmate activity         | Salesmate Tool MCP Server  | -                         | Sticky Note 1                   |
| Get an activity         | Salesmate Tool             | Gets a Salesmate activity by ID      | Salesmate Tool MCP Server  | -                         | Sticky Note 1                   |
| Get many activities     | Salesmate Tool             | Gets multiple Salesmate activities   | Salesmate Tool MCP Server  | -                         | Sticky Note 1                   |
| Update an activity      | Salesmate Tool             | Updates a Salesmate activity         | Salesmate Tool MCP Server  | -                         | Sticky Note 1                   |
| Sticky Note 1           | Sticky Note                | Visual group: Activities              | -                          | -                         |                                 |
| Create a company        | Salesmate Tool             | Creates a Salesmate company          | Salesmate Tool MCP Server  | -                         | Sticky Note 2                   |
| Delete a company        | Salesmate Tool             | Deletes a Salesmate company          | Salesmate Tool MCP Server  | -                         | Sticky Note 2                   |
| Get a company           | Salesmate Tool             | Gets a Salesmate company by ID       | Salesmate Tool MCP Server  | -                         | Sticky Note 2                   |
| Get many companies      | Salesmate Tool             | Gets multiple Salesmate companies    | Salesmate Tool MCP Server  | -                         | Sticky Note 2                   |
| Update a company        | Salesmate Tool             | Updates a Salesmate company          | Salesmate Tool MCP Server  | -                         | Sticky Note 2                   |
| Sticky Note 2           | Sticky Note                | Visual group: Companies               | -                          | -                         |                                 |
| Create a deal           | Salesmate Tool             | Creates a Salesmate deal             | Salesmate Tool MCP Server  | -                         | Sticky Note 3                   |
| Delete a deal           | Salesmate Tool             | Deletes a Salesmate deal             | Salesmate Tool MCP Server  | -                         | Sticky Note 3                   |
| Get a deal              | Salesmate Tool             | Gets a Salesmate deal by ID          | Salesmate Tool MCP Server  | -                         | Sticky Note 3                   |
| Get many deals          | Salesmate Tool             | Gets multiple Salesmate deals        | Salesmate Tool MCP Server  | -                         | Sticky Note 3                   |
| Update a deal           | Salesmate Tool             | Updates a Salesmate deal             | Salesmate Tool MCP Server  | -                         | Sticky Note 3                   |
| Sticky Note 3           | Sticky Note                | Visual group: Deals                   | -                          | -                         |                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add MCP Trigger Node:**  
   - Node Type: `@n8n/n8n-nodes-langchain.mcpTrigger`  
   - Name: "Salesmate Tool MCP Server"  
   - Configure webhook: Generate or assign a webhook ID. This node will serve as the entry point for all Salesmate operations.  
   - Leave parameters empty (default routing behavior).  

3. **Add Salesmate Tool Nodes for Activities:**  
   For each operation below, add a node of type `n8n-nodes-base.salesmateTool` and configure the operation accordingly:  
   - "Create an activity" → Operation: Create Activity  
   - "Get an activity" → Operation: Get Activity (requires activity ID parameter)  
   - "Get many activities" → Operation: List Activities (optional filters/pagination)  
   - "Update an activity" → Operation: Update Activity (requires ID and fields)  
   - "Delete an activity" → Operation: Delete Activity (requires ID)  
   Connect each node’s input from the MCP Trigger node's output port.

4. **Add Salesmate Tool Nodes for Companies:**  
   Repeat the same steps for company operations:  
   - "Create a company"  
   - "Get a company"  
   - "Get many companies"  
   - "Update a company"  
   - "Delete a company"  
   Connect all to MCP Trigger node.

5. **Add Salesmate Tool Nodes for Deals:**  
   Repeat for deal operations:  
   - "Create a deal"  
   - "Get a deal"  
   - "Get many deals"  
   - "Update a deal"  
   - "Delete a deal"  
   Connect all to MCP Trigger node.

6. **Add Sticky Notes:**  
   - Add three sticky notes to visually group the above Salesmate tool nodes into Activities, Companies, and Deals blocks for better editor organization.  
   - Optionally add a top-level overview sticky note.

7. **Credential Setup:**  
   - For each Salesmate Tool node, configure the Salesmate API credentials (OAuth2 or API key as per your Salesmate account). Ensure credentials are valid and authorized.  
   - The MCP Trigger node requires no credentials but must be accessible via its webhook URL.

8. **Parameter Setup:**  
   - Ensure each Salesmate Tool node has its operation parameter set explicitly per operation.  
   - Configure input mapping if necessary to pass parameters dynamically from the MCP Trigger requests.

9. **Test the Workflow:**  
   - Trigger the webhook with different operation requests to verify each operation executes correctly.  
   - Handle error scenarios by verifying error messages and API call limits.

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                 |
|-----------------------------------------------------------------------------------------------|--------------------------------|
| This workflow consolidates 15 Salesmate operations into a single MCP-triggered server workflow. | Workflow purpose                |
| MCP Trigger node requires n8n version supporting `@n8n/n8n-nodes-langchain` package.          | Node version requirement       |
| Salesmate Tool nodes require valid Salesmate API credentials set up in n8n credentials manager. | Integration setup              |
| Grouping nodes with sticky notes improves readability and maintenance.                         | Editor usability best practice |
| The webhook ID for the MCP Trigger node must be kept secret and secure.                        | Security recommendation        |

---

*Disclaimer:* The provided text is exclusively extracted and analyzed from an automated n8n workflow. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.