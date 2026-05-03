🛠️ Metabase Tool MCP Server 💪 all 10 operations

https://n8nworkflows.xyz/workflows/----metabase-tool-mcp-server----all-10-operations-5183


# 🛠️ Metabase Tool MCP Server 💪 all 10 operations

### 1. Workflow Overview

This n8n workflow titled **"🛠️ Metabase Tool MCP Server 💪 all 10 operations"** is designed to provide a comprehensive integration layer between n8n and Metabase, exposing all 10 core Metabase operations through a single MCP (Metabase Connector Plugin) Server trigger. It acts as a centralized API endpoint for triggering any of the following Metabase operations programmatically:

- Alerts (single and multiple retrieval)
- Databases (add, get multiple, get fields)
- Metrics (single and multiple retrieval)
- Questions (single and multiple retrieval)
- Retrieving results from a question

The workflow is logically structured into blocks representing different categories of Metabase operations, all connected downstream from a single **Metabase Tool MCP Server** trigger node. This trigger handles incoming requests and routes them to the corresponding Metabase operation node. Each operation node uses the **Metabase Tool** node type configured for a specific API call.

Logical blocks:

- **1.1 MCP Server Trigger** – Serves as the single entry point for all 10 Metabase operations.
- **1.2 Alerts Operations** – Handles retrieving one or many alerts.
- **1.3 Databases Operations** – Handles adding a database, getting many databases, and getting fields for a database.
- **1.4 Metrics Operations** – Handles retrieving one or many metrics.
- **1.5 Questions Operations** – Handles retrieving one or many questions and fetching results from a question.

Sticky notes are placed throughout for visual grouping but contain no content.

---

### 2. Block-by-Block Analysis

#### 1.1 MCP Server Trigger

- **Overview:**  
  This node serves as the single webhook entry point that listens for incoming requests specifying which Metabase operation to execute. It triggers the workflow and routes the request to the appropriate downstream Metabase Tool node based on the operation selected.

- **Nodes Involved:**  
  - Metabase Tool MCP Server

- **Node Details:**  
  - Type: `@n8n/n8n-nodes-langchain.mcpTrigger`  
  - Role: Webhook trigger node specialized for MCP server operations.  
  - Configuration: No additional parameters configured; uses webhook ID `"f21f2e0d-652d-4f46-b17c-ef0957caf0da"`.  
  - Inputs: None (trigger node)  
  - Outputs: Multiple output connections to all Metabase Tool nodes via the `ai_tool` output.  
  - Edge cases:  
    - Failure to receive HTTP requests (network issues).  
    - Incorrect payload format leading to routing errors.  
    - MCP server-specific authorization or authentication failures if configured externally.  
  - Notes: Acts as the core dispatcher for all subsequent operations.

#### 1.2 Alerts Operations

- **Overview:**  
  Contains nodes to retrieve a single alert or multiple alerts from Metabase.

- **Nodes Involved:**  
  - Get an alert  
  - Get many alerts

- **Node Details:**  

  **Get an alert**  
  - Type: `n8n-nodes-base.metabaseTool`  
  - Role: Retrieves details for a specific alert.  
  - Configuration: No explicit parameters visible; configured to use the "Get an alert" Metabase API operation internally.  
  - Inputs: From Metabase Tool MCP Server node output (`ai_tool`)  
  - Outputs: None (endpoint output)  
  - Edge cases:  
    - Alert ID missing or invalid.  
    - API authentication failures.  
    - Network timeouts.

  **Get many alerts**  
  - Type: `n8n-nodes-base.metabaseTool`  
  - Role: Retrieves a list of alerts.  
  - Configuration: Uses "Get many alerts" Metabase API operation.  
  - Inputs: From MCP Server node output (`ai_tool`)  
  - Outputs: None (endpoint output)  
  - Edge cases:  
    - Pagination or large data sets causing timeouts.  
    - Authentication or permission failures.

#### 1.3 Databases Operations

- **Overview:**  
  Manages database-related API calls: adding a database, listing databases, and retrieving fields for a database.

- **Nodes Involved:**  
  - Add a databases  
  - Get many databases  
  - Get Fields a databases

- **Node Details:**  

  **Add a databases**  
  - Type: `n8n-nodes-base.metabaseTool`  
  - Role: Adds a new database to Metabase.  
  - Configuration: Uses "Add a databases" API operation (likely requires database connection details as input).  
  - Inputs: From MCP Server node output (`ai_tool`)  
  - Outputs: None (endpoint output)  
  - Edge cases:  
    - Missing or invalid database configuration details.  
    - Permission denied errors.  
    - Conflicts if database already exists.

  **Get many databases**  
  - Type: `n8n-nodes-base.metabaseTool`  
  - Role: Retrieves a list of databases.  
  - Configuration: "Get many databases" API operation.  
  - Inputs: From MCP Server node output (`ai_tool`)  
  - Outputs: None  
  - Edge cases: Large numbers of databases may cause pagination issues.

  **Get Fields a databases**  
  - Type: `n8n-nodes-base.metabaseTool`  
  - Role: Retrieves fields (schema) for a given database.  
  - Configuration: "Get Fields a databases" API operation.  
  - Inputs: From MCP Server node output (`ai_tool`)  
  - Outputs: None  
  - Edge cases: Invalid database ID, schema access permission errors.

#### 1.4 Metrics Operations

- **Overview:**  
  Handles fetching a single metric or multiple metrics from Metabase.

- **Nodes Involved:**  
  - Get a metric  
  - Get many metrics

- **Node Details:**  

  **Get a metric**  
  - Type: `n8n-nodes-base.metabaseTool`  
  - Role: Retrieves details of a specific metric.  
  - Configuration: Uses "Get a metric" API operation.  
  - Inputs: From MCP Server node output (`ai_tool`)  
  - Outputs: None  
  - Edge cases: Invalid metric ID, permission errors.

  **Get many metrics**  
  - Type: `n8n-nodes-base.metabaseTool`  
  - Role: Retrieves a list of metrics.  
  - Configuration: "Get many metrics" API operation.  
  - Inputs: From MCP Server node output (`ai_tool`)  
  - Outputs: None  
  - Edge cases: Large datasets, pagination, permission issues.

#### 1.5 Questions Operations

- **Overview:**  
  Retrieves questions metadata and their results.

- **Nodes Involved:**  
  - Get a questions  
  - Get many questions  
  - Get the results from a question

- **Node Details:**  

  **Get a questions**  
  - Type: `n8n-nodes-base.metabaseTool`  
  - Role: Retrieves metadata about a specific question.  
  - Configuration: Uses "Get a questions" API operation.  
  - Inputs: From MCP Server node output (`ai_tool`)  
  - Outputs: None  
  - Edge cases: Invalid question ID, permission denied.

  **Get many questions**  
  - Type: `n8n-nodes-base.metabaseTool`  
  - Role: Retrieves a list of questions.  
  - Configuration: "Get many questions" API operation.  
  - Inputs: From MCP Server node output (`ai_tool`)  
  - Outputs: None  
  - Edge cases: Large numbers of questions, pagination.

  **Get the results from a question**  
  - Type: `n8n-nodes-base.metabaseTool`  
  - Role: Fetches query results for a specified question.  
  - Configuration: Uses "Get the results from a question" API operation.  
  - Inputs: From MCP Server node output (`ai_tool`)  
  - Outputs: None  
  - Edge cases: Query execution timeouts, invalid question ID, permission errors.

---

### 3. Summary Table

| Node Name                 | Node Type                            | Functional Role                                  | Input Node(s)            | Output Node(s)             | Sticky Note          |
|---------------------------|------------------------------------|-------------------------------------------------|--------------------------|----------------------------|----------------------|
| Workflow Overview 0       | Sticky Note                        | Visual grouping (no content)                     | None                     | None                       |                      |
| Metabase Tool MCP Server  | MCP Trigger                       | Core webhook trigger routing to Metabase ops    | None                     | All Metabase Tool nodes    |                      |
| Get an alert              | Metabase Tool                     | Retrieve a single alert                           | Metabase Tool MCP Server | None                       |                      |
| Get many alerts           | Metabase Tool                     | Retrieve multiple alerts                          | Metabase Tool MCP Server | None                       |                      |
| Sticky Note 1             | Sticky Note                       | Visual grouping (no content)                      | None                     | None                       |                      |
| Add a databases           | Metabase Tool                     | Add a new database                               | Metabase Tool MCP Server | None                       |                      |
| Get many databases        | Metabase Tool                     | Retrieve multiple databases                       | Metabase Tool MCP Server | None                       |                      |
| Get Fields a databases    | Metabase Tool                     | Retrieve fields/schema of a database              | Metabase Tool MCP Server | None                       |                      |
| Sticky Note 2             | Sticky Note                       | Visual grouping (no content)                      | None                     | None                       |                      |
| Get a metric              | Metabase Tool                     | Retrieve a single metric                          | Metabase Tool MCP Server | None                       |                      |
| Get many metrics          | Metabase Tool                     | Retrieve multiple metrics                         | Metabase Tool MCP Server | None                       |                      |
| Sticky Note 3             | Sticky Note                       | Visual grouping (no content)                      | None                     | None                       |                      |
| Get a questions           | Metabase Tool                     | Retrieve a single question                        | Metabase Tool MCP Server | None                       |                      |
| Get many questions        | Metabase Tool                     | Retrieve multiple questions                       | Metabase Tool MCP Server | None                       |                      |
| Get the results from a question | Metabase Tool               | Retrieve query results of a question             | Metabase Tool MCP Server | None                       |                      |
| Sticky Note 4             | Sticky Note                       | Visual grouping (no content)                      | None                     | None                       |                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the MCP Server Trigger Node**  
   - Add node: Type `@n8n/n8n-nodes-langchain.mcpTrigger`.  
   - Name: `Metabase Tool MCP Server`.  
   - Configure webhook ID (auto-generated or custom).  
   - No additional parameters needed.  
   - This node acts as a webhook listening for incoming requests to trigger Metabase operations.

2. **Add Metabase Tool Nodes for Each Operation**  
   For each Metabase operation below, add a node of type `n8n-nodes-base.metabaseTool` and configure it accordingly:

   a. **Get an alert**  
   - Name: `Get an alert`.  
   - Operation: Set to "Get an alert" API call inside the node.  
   - Connect input from `Metabase Tool MCP Server` node's `ai_tool` output.

   b. **Get many alerts**  
   - Name: `Get many alerts`.  
   - Operation: Set to "Get many alerts".  
   - Connect input from `Metabase Tool MCP Server` node.

   c. **Add a databases**  
   - Name: `Add a databases`.  
   - Operation: Set to "Add a databases".  
   - Configure required parameters for database addition (connection info, type).  
   - Connect input from MCP Server node.

   d. **Get many databases**  
   - Name: `Get many databases`.  
   - Operation: Set to "Get many databases".  
   - Connect input from MCP Server node.

   e. **Get Fields a databases**  
   - Name: `Get Fields a databases`.  
   - Operation: Set to "Get Fields a databases".  
   - Connect input from MCP Server node.

   f. **Get a metric**  
   - Name: `Get a metric`.  
   - Operation: Set to "Get a metric".  
   - Connect input from MCP Server node.

   g. **Get many metrics**  
   - Name: `Get many metrics`.  
   - Operation: Set to "Get many metrics".  
   - Connect input from MCP Server node.

   h. **Get a questions**  
   - Name: `Get a questions`.  
   - Operation: Set to "Get a questions".  
   - Connect input from MCP Server node.

   i. **Get many questions**  
   - Name: `Get many questions`.  
   - Operation: Set to "Get many questions".  
   - Connect input from MCP Server node.

   j. **Get the results from a question**  
   - Name: `Get the results from a question`.  
   - Operation: Set to "Get the results from a question".  
   - Connect input from MCP Server node.

3. **Configure Credentials**  
   - For all `Metabase Tool` nodes, configure Metabase API credentials (API token, base URL).  
   - Ensure the credentials have permissions for all required operations (alerts, databases, metrics, questions).

4. **Add Sticky Notes (Optional)**  
   - For visual clarity, add sticky notes grouping related nodes (alerts, databases, metrics, questions).  
   - Content is optional; in the original workflow, sticky notes are empty.

5. **Test Workflow**  
   - Trigger the MCP Server webhook with payloads specifying which Metabase operation to invoke.  
   - Verify each operation returns correct data or performs the intended action.

---

### 5. General Notes & Resources

| Note Content                                                                 | Context or Link                                       |
|------------------------------------------------------------------------------|------------------------------------------------------|
| This workflow provides a full set of 10 Metabase operations accessible via a single MCP Server webhook, enabling easy programmatic access to Metabase resources. | Workflow description and architectural intent        |
| Metabase Tool nodes require appropriate API credentials configured in n8n for authentication and authorization. | Credential configuration guidance                     |
| Sticky notes are used solely for visual grouping and contain no operational instructions or comments. | Workflow visual organization                          |
| For more information on Metabase API endpoints, refer to official Metabase documentation: https://www.metabase.com/docs/latest/api-documentation.html | Metabase API docs                                    |
| MCP Trigger Node is part of n8n's LangChain integration for multi-connector plugin server use. | n8n LangChain MCP Trigger overview                    |

---

**Disclaimer:** The provided content is based exclusively on an automated workflow created with n8n, adhering strictly to current content policies and containing no illegal, offensive, or protected elements. All data processed is legal and publicly accessible.