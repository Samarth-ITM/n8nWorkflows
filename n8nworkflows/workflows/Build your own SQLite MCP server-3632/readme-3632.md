Build your own SQLite MCP server

https://n8nworkflows.xyz/workflows/build-your-own-sqlite-mcp-server-3632


# Build your own SQLite MCP server

### 1. Workflow Overview

This workflow implements a self-hosted SQLite MCP (Model Context Protocol) server designed to perform local database operations and support business intelligence queries. It enables compatible MCP clients or agents (e.g., Claude Desktop) to interact with a SQLite database by performing select, insert, and update operations in a secure and controlled manner.

The workflow is logically divided into the following blocks:

- **1.1 MCP Server Trigger**: Listens for incoming MCP client requests and acts as the entry point.
- **1.2 Operation Routing**: Routes the incoming request based on the requested database operation (read, insert, update).
- **1.3 Database Operation Execution**: Executes the actual SQLite queries using Code nodes for read, insert, and update operations.
- **1.4 Custom Workflow Tools**: Defines three custom workflows (tools) that expose restricted schemas for select, insert, and update operations to MCP clients, ensuring safer parameterized queries.
- **1.5 Utility Tools**: Additional tools to list tables and describe table schemas for enhanced business intelligence capabilities.
- **1.6 Documentation and Security Notes**: Sticky notes provide guidance on setup, security best practices, and usage instructions.

---

### 2. Block-by-Block Analysis

#### 2.1 MCP Server Trigger

- **Overview:**  
  This block contains the MCP server trigger node that listens for incoming MCP client requests and initiates the workflow. It acts as the main entry point for all database operation requests.

- **Nodes Involved:**  
  - `SQLite MCP Server`

- **Node Details:**  
  - **Type:** MCP Trigger (`@n8n/n8n-nodes-langchain.mcpTrigger`)  
  - **Role:** Listens for MCP client requests on a specific webhook path.  
  - **Configuration:**  
    - Webhook path set to a unique identifier (`3124a4cd-4e93-4c1b-b4db-b5599f4889b1`).  
  - **Input/Output:**  
    - Outputs MCP client request data to connected nodes.  
  - **Edge Cases:**  
    - Requires self-hosted n8n instance to access local SQLite file.  
    - Should be secured with authentication before production use to prevent unauthorized access.  
  - **Sticky Notes:**  
    - Guidance on MCP Server Trigger usage and authentication best practices.

#### 2.2 Operation Routing

- **Overview:**  
  Routes incoming requests to the appropriate database operation handler based on the `operation` field in the input JSON (`read`, `insert`, or `update`).

- **Nodes Involved:**  
  - `When Executed by Another Workflow`  
  - `Operation`

- **Node Details:**  
  - **When Executed by Another Workflow**  
    - **Type:** Execute Workflow Trigger (`n8n-nodes-base.executeWorkflowTrigger`)  
    - **Role:** Allows this workflow to be executed by other workflows or tools.  
    - **Configuration:** Defines input parameters: `operation`, `tableName`, `values`, and `where`.  
    - **Input/Output:** Receives parameters from invoking workflows, outputs to the `Operation` node.  
    - **Edge Cases:** Input validation failures if parameters are missing or malformed.  

  - **Operation**  
    - **Type:** Switch (`n8n-nodes-base.switch`)  
    - **Role:** Routes data based on the `operation` string value.  
    - **Configuration:**  
      - Routes `"read"` to `ReadRecords`.  
      - Routes `"insert"` to `CreateRecord`.  
      - Routes `"update"` to `UpdateRecord`.  
    - **Input/Output:** Receives JSON input, outputs to one of three Code nodes.  
    - **Edge Cases:**  
      - If `operation` is missing or invalid, no route is taken (no output).  
      - Case-sensitive matching enforced.

#### 2.3 Database Operation Execution

- **Overview:**  
  Executes the actual SQLite queries for read, insert, and update operations using Node.js SQLite3 library wrapped in Code nodes. These nodes perform parameterized queries to prevent SQL injection.

- **Nodes Involved:**  
  - `ReadRecords`  
  - `CreateRecord`  
  - `UpdateRecord`

- **Node Details:**  

  - **ReadRecords**  
    - **Type:** Code Node (`n8n-nodes-base.code`)  
    - **Role:** Performs a SELECT query on the specified table with optional WHERE conditions.  
    - **Configuration:**  
      - Uses SQLite3 library with `db.all` promisified to fetch all matching rows.  
      - Constructs SQL: `SELECT * FROM <tableName> [WHERE col1 = ? AND col2 = ? ...]`  
      - Parameters passed as array from `where` object values.  
    - **Input:** JSON with `tableName` and optional `where` object.  
    - **Output:** JSON with `output` (array of rows) or `error`.  
    - **Edge Cases:**  
      - Missing or invalid `tableName` causes query failure.  
      - Empty or missing `where` performs full table scan.  
      - Database file access errors.  

  - **CreateRecord**  
    - **Type:** Code Node (`n8n-nodes-base.code`)  
    - **Role:** Performs an INSERT operation into the specified table with given column values.  
    - **Configuration:**  
      - Uses SQLite3 library with `db.run` promisified.  
      - Constructs SQL: `INSERT INTO <tableName> (col1, col2, ...) VALUES (?, ?, ...)`  
      - Parameters are the values object entries.  
    - **Input:** JSON with `tableName` and `values` object.  
    - **Output:** JSON with status `ok` or error info.  
    - **Edge Cases:**  
      - Missing or invalid `tableName` or `values` causes failure.  
      - Database file access errors.  

  - **UpdateRecord**  
    - **Type:** Code Node (`n8n-nodes-base.code`)  
    - **Role:** Performs an UPDATE operation on specified table rows matching `where` conditions.  
    - **Configuration:**  
      - Uses SQLite3 library with `db.run` promisified.  
      - Constructs SQL: `UPDATE <tableName> SET col1 = ?, col2 = ? WHERE cond1 = ? AND cond2 = ?`  
      - Parameters are concatenated values from `values` and `where` objects.  
    - **Input:** JSON with `tableName`, `values`, and `where` objects.  
    - **Output:** JSON with status `ok` or error info.  
    - **Edge Cases:**  
      - Missing or invalid `tableName`, `values`, or `where` causes failure.  
      - Database file access errors.  

#### 2.4 Custom Workflow Tools

- **Overview:**  
  These three custom workflows expose restricted schemas to MCP clients for select, insert, and update operations. They ensure that clients provide only parameters, not raw SQL, enhancing security.

- **Nodes Involved:**  
  - `ReadRows`  
  - `CreateRecords`  
  - `UpdateRows`

- **Node Details:**  

  - Each is a **Custom Workflow Tool** (`@n8n/n8n-nodes-langchain.toolWorkflow`)  
  - **Role:** Acts as a callable tool by the MCP server trigger, forwarding structured input to this workflow via the `When Executed by Another Workflow` node.  
  - **Configuration:**  
    - Each defines input schema with fields: `operation`, `tableName`, `values`, `where`.  
    - Default values set for `operation` to match the intended action (`read`, `insert`, `update`).  
    - They invoke this same workflow by ID, enabling recursive calls for operation routing.  
  - **Input/Output:** Accepts structured input from MCP clients, outputs results from the core workflow.  
  - **Edge Cases:**  
    - Input validation errors if schema is violated.  
    - Recursive call failures if workflow ID is incorrect or workflow is disabled.

#### 2.5 Utility Tools

- **Overview:**  
  These tools provide additional database introspection capabilities: listing all tables and describing table schemas.

- **Nodes Involved:**  
  - `ListTables`  
  - `DescribeTables`

- **Node Details:**  

  - **ListTables**  
    - **Type:** Tool Code Node (`@n8n/n8n-nodes-langchain.toolCode`)  
    - **Role:** Lists all user tables in the SQLite database (excluding internal SQLite tables).  
    - **Configuration:**  
      - Runs SQL: `SELECT name FROM sqlite_master WHERE type='table' AND name NOT LIKE 'sqlite_%'`  
      - Returns list of table names as newline-separated string.  
    - **Edge Cases:**  
      - Database access errors.  

  - **DescribeTables**  
    - **Type:** Tool Code Node (`@n8n/n8n-nodes-langchain.toolCode`)  
    - **Role:** Describes schema of a specified table using SQLite PRAGMA.  
    - **Configuration:**  
      - Runs SQL: `PRAGMA table_info(<tableName>)`  
      - Returns formatted string describing columns, types, nullability, and defaults.  
    - **Input:** Requires `tableName` string.  
    - **Edge Cases:**  
      - Missing or invalid `tableName` causes failure.  
      - Database access errors.

#### 2.6 Documentation and Security Notes

- **Overview:**  
  Several sticky notes provide important information on setup, security, usage, and customization.

- **Nodes Involved:**  
  - `Sticky Note`  
  - `Sticky Note1`  
  - `Sticky Note2`  
  - `Sticky Note3`  
  - `Sticky Note4`

- **Node Details:**  
  - Contain guidance on:  
    - MCP Server Trigger documentation link.  
    - Importance of preventing raw SQL statements for security.  
    - Recommendation to enable authentication before production.  
    - Usage instructions and example queries.  
    - Reminder that this template works only on self-hosted n8n instances due to local SQLite file access.  
  - These notes do not affect workflow execution but are critical for safe and correct deployment.

---

### 3. Summary Table

| Node Name                  | Node Type                               | Functional Role                          | Input Node(s)                | Output Node(s)                 | Sticky Note                                                                                                                        |
|----------------------------|---------------------------------------|----------------------------------------|-----------------------------|-------------------------------|-----------------------------------------------------------------------------------------------------------------------------------|
| SQLite MCP Server           | MCP Trigger (`@n8n/n8n-nodes-langchain.mcpTrigger`) | Entry point for MCP client requests    |                             | ListTables, DescribeTables, ReadRows, CreateRecords, UpdateRows | See Sticky Note, Sticky Note1, Sticky Note2, Sticky Note3, Sticky Note4 for setup and security guidance                           |
| ListTables                 | Tool Code Node (`@n8n/n8n-nodes-langchain.toolCode`) | Lists all SQLite tables                 | SQLite MCP Server (ai_tool) | SQLite MCP Server (ai_tool)    | See Sticky Note2                                                                                                                  |
| DescribeTables             | Tool Code Node (`@n8n/n8n-nodes-langchain.toolCode`) | Describes schema of a SQLite table     | SQLite MCP Server (ai_tool) | SQLite MCP Server (ai_tool)    | See Sticky Note2                                                                                                                  |
| ReadRows                   | Custom Workflow Tool (`toolWorkflow`) | Tool for reading rows (select)          | SQLite MCP Server (ai_tool) | SQLite MCP Server (ai_tool)    | See Sticky Note2                                                                                                                  |
| CreateRecords              | Custom Workflow Tool (`toolWorkflow`) | Tool for inserting rows                  | SQLite MCP Server (ai_tool) | SQLite MCP Server (ai_tool)    | See Sticky Note2                                                                                                                  |
| UpdateRows                 | Custom Workflow Tool (`toolWorkflow`) | Tool for updating rows                   | SQLite MCP Server (ai_tool) | SQLite MCP Server (ai_tool)    | See Sticky Note2                                                                                                                  |
| When Executed by Another Workflow | Execute Workflow Trigger (`executeWorkflowTrigger`) | Receives operation requests from tools |                             | Operation                     |                                                                                                                                    |
| Operation                  | Switch (`switch`)                     | Routes operation to correct handler    | When Executed by Another Workflow | ReadRecords, CreateRecord, UpdateRecord |                                                                                                                                    |
| ReadRecords                | Code Node (`code`)                    | Executes SELECT queries                 | Operation                   |                               | See Sticky Note1 (security best practices)                                                                                       |
| CreateRecord               | Code Node (`code`)                    | Executes INSERT queries                 | Operation                   |                               | See Sticky Note1 (security best practices)                                                                                       |
| UpdateRecord               | Code Node (`code`)                    | Executes UPDATE queries                 | Operation                   |                               | See Sticky Note1 (security best practices)                                                                                       |
| Sticky Note                | Sticky Note                          | Documentation and guidance              |                             |                               | MCP Server Trigger documentation link                                                                                            |
| Sticky Note1               | Sticky Note                          | Security guidance on raw SQL prevention |                             |                               | Prevent raw SQL to avoid security risks                                                                                          |
| Sticky Note2               | Sticky Note                          | Full workflow usage instructions       |                             |                               | Detailed usage, requirements, and customization notes                                                                            |
| Sticky Note3               | Sticky Note                          | Authentication reminder                 |                             |                               | Always authenticate your MCP server before production                                                                            |
| Sticky Note4               | Sticky Note                          | Self-hosted instance warning            |                             |                               | Template only works on self-hosted n8n instances                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create MCP Server Trigger Node**  
   - Type: `@n8n/n8n-nodes-langchain.mcpTrigger`  
   - Set webhook path to a unique identifier (e.g., `3124a4cd-4e93-4c1b-b4db-b5599f4889b1`).  
   - This node listens for MCP client requests.  
   - Ensure your n8n instance is self-hosted and has access to the SQLite database file on disk.  
   - Optional: Configure authentication on this trigger before production.

2. **Create Custom Workflow Tools for Database Operations**  
   For each operation (`read`, `insert`, `update`), create a custom workflow tool node:  

   - Type: `@n8n/n8n-nodes-langchain.toolWorkflow`  
   - Set `workflowId` to the current workflow’s ID (to enable recursive calls).  
   - Define input schema with fields:  
     - `operation` (string, default: `"read"` / `"insert"` / `"update"`)  
     - `tableName` (string)  
     - `values` (object)  
     - `where` (object)  
   - Set default input values accordingly for each tool.  
   - Name them `ReadRows`, `CreateRecords`, and `UpdateRows` respectively.  
   - Connect their `ai_tool` output to the MCP Server Trigger node’s `ai_tool` input.

3. **Create Execute Workflow Trigger Node**  
   - Type: `n8n-nodes-base.executeWorkflowTrigger`  
   - Configure input parameters:  
     - `operation` (string)  
     - `tableName` (string)  
     - `values` (object)  
     - `where` (object)  
   - This node allows the custom workflows to invoke this main workflow with parameters.

4. **Create Switch Node for Operation Routing**  
   - Type: `n8n-nodes-base.switch`  
   - Add rules to route based on `operation` field:  
     - If `operation` equals `"read"`, route to `ReadRecords` node.  
     - If `operation` equals `"insert"`, route to `CreateRecord` node.  
     - If `operation` equals `"update"`, route to `UpdateRecord` node.  
   - Connect the output of the Execute Workflow Trigger node to this switch node.

5. **Create Code Nodes for SQLite Operations**  
   - **ReadRecords** (Code Node):  
     - Use Node.js `sqlite3` library with promisified `db.all` to run parameterized SELECT queries.  
     - Construct SQL with optional WHERE clause based on `where` input.  
     - Return query results or error.  
     - Connect from `Operation` switch node’s `READ` output.  

   - **CreateRecord** (Code Node):  
     - Use `sqlite3` with promisified `db.run` to perform parameterized INSERT queries.  
     - Construct SQL with columns and placeholders from `values` input.  
     - Return success status or error.  
     - Connect from `Operation` switch node’s `INSERT` output.  

   - **UpdateRecord** (Code Node):  
     - Use `sqlite3` with promisified `db.run` to perform parameterized UPDATE queries.  
     - Construct SQL with SET and WHERE clauses from `values` and `where` inputs.  
     - Return success status or error.  
     - Connect from `Operation` switch node’s `UPDATE` output.

6. **Create Utility Tool Code Nodes**  
   - **ListTables** (Tool Code Node):  
     - Query SQLite system tables to list user tables excluding internal ones.  
     - Return list of table names.  
     - Connect `ai_tool` output to MCP Server Trigger’s `ai_tool` input.  

   - **DescribeTables** (Tool Code Node):  
     - Use SQLite PRAGMA to describe table schema.  
     - Accept `tableName` as input.  
     - Return formatted schema description.  
     - Connect `ai_tool` output to MCP Server Trigger’s `ai_tool` input.

7. **Add Sticky Notes for Documentation and Security**  
   - Add sticky notes with:  
     - MCP Server Trigger documentation link.  
     - Security advice to avoid raw SQL statements.  
     - Reminder to enable authentication before production.  
     - Usage instructions and example queries.  
     - Note about self-hosted n8n requirement.

8. **Final Connections**  
   - Connect the custom workflow tools (`ReadRows`, `CreateRecords`, `UpdateRows`) to the MCP Server Trigger node’s `ai_tool` input.  
   - Connect `ListTables` and `DescribeTables` tool code nodes similarly to MCP Server Trigger’s `ai_tool` input.  
   - Connect the Execute Workflow Trigger node’s output to the `Operation` switch node.  
   - Connect the `Operation` switch node’s outputs to the respective Code nodes (`ReadRecords`, `CreateRecord`, `UpdateRecord`).  

9. **Credentials and Environment Setup**  
   - Ensure the n8n instance has access to the SQLite database file at `/home/node/test.db` or update the path accordingly in all Code nodes.  
   - No external credentials are required for SQLite access since it is file-based.  
   - MCP clients connecting to this server must be configured according to n8n MCP integration guidelines: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-langchain.mcptrigger/#integrating-with-claude-desktop

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Context or Link                                                                                                   |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| This template is for Self-Hosted N8N Instances only because it requires direct access to the SQLite database file on disk.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Sticky Note4                                                                                                     |
| MCP Server Trigger documentation and integration guide: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-langchain.mcptrigger                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Sticky Note                                                                                                      |
| Prevent raw SQL statements from MCP clients to avoid SQL injection and data leaks. Use parameterized queries and restrict input schemas to improve security.                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Sticky Note1                                                                                                     |
| Always enable authentication on your MCP server trigger before deploying to production to prevent unauthorized access.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Sticky Note3                                                                                                     |
| Usage instructions and example queries for MCP clients:  
- "Please create a table to store business insights and add the following..."  
- "What business insights do we have on current retail trends?"  
- "Who has contributed the most business insights in the past week?"  
Also includes links to official MCP reference implementation: https://github.com/modelcontextprotocol/servers/tree/main/src/sqlite and Claude Desktop download: https://claude.ai/download | Sticky Note2                                                                                                     |

---

This documentation provides a complete and detailed reference for understanding, reproducing, and safely operating the SQLite MCP server workflow in n8n.