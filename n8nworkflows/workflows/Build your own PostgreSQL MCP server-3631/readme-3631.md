Build your own PostgreSQL MCP server

https://n8nworkflows.xyz/workflows/build-your-own-postgresql-mcp-server-3631


# Build your own PostgreSQL MCP server

### 1. Workflow Overview

This workflow implements a PostgreSQL MCP (Model Context Protocol) server using n8n, designed to manage a PostgreSQL database with operations such as reading, inserting, and updating records. It targets use cases like HR, Payroll, Sales, Inventory management, and more, providing a secure interface for MCP clients (e.g., Claude Desktop) to interact with the database without exposing raw SQL queries.

The workflow is logically divided into the following blocks:

- **1.1 MCP Server Trigger**: Listens for incoming MCP client requests and routes them to appropriate tools.
- **1.2 PostgreSQL Read-Only Tools**: Provides simple read-only queries to list tables and get table schema.
- **1.3 Custom Workflow Tools for CRUD Operations**: Implements controlled select, insert, and update operations via custom workflows that restrict input parameters to prevent raw SQL exposure.
- **1.4 Operation Dispatcher and SQL Execution**: Routes incoming operations (read, insert, update) to corresponding PostgreSQL nodes that execute parameterized SQL queries.
- **1.5 Response Handling**: Sends query results back to the MCP client.

---

### 2. Block-by-Block Analysis

#### 1.1 MCP Server Trigger

- **Overview:**  
  This block initializes the MCP server trigger node that listens for requests from MCP clients. It acts as the entry point for all database management operations.

- **Nodes Involved:**  
  - PostgreSQL MCP Server (MCP Trigger)  
  - Sticky Note (documentation)

- **Node Details:**

  - **PostgreSQL MCP Server**  
    - Type: `@n8n/n8n-nodes-langchain.mcpTrigger`  
    - Role: Listens for MCP client requests and triggers downstream nodes.  
    - Configuration: Uses a unique webhook path (`a5fd7047-e31b-4c0d-bd68-c36072c3da0d`) to receive requests.  
    - Input: MCP client requests containing operation type and parameters.  
    - Output: Routes requests to connected tools (PostgreSQL and custom workflows).  
    - Edge Cases: Requires authentication before production to prevent unauthorized access. Potential webhook downtime or network issues may cause request failures.  
    - Sticky Notes: Reference to MCP Server Trigger documentation [here](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-langchain.mcptrigger).

  - **Sticky Note**  
    - Purpose: Provides a reminder and link to MCP Server Trigger documentation.

---

#### 1.2 PostgreSQL Read-Only Tools

- **Overview:**  
  This block provides two simple PostgreSQL tools for read-only queries: listing all tables and retrieving a table's schema. These tools allow the MCP client to explore database structure safely.

- **Nodes Involved:**  
  - ListTables (PostgreSQL Tool)  
  - GetTableSchema (PostgreSQL Tool)  

- **Node Details:**

  - **ListTables**  
    - Type: `n8n-nodes-base.postgresTool`  
    - Role: Executes a query to list all table names in the public schema.  
    - Configuration: Query is `SELECT table_name FROM information_schema.tables WHERE table_schema = 'public'`. No parameters required.  
    - Input: Triggered by MCP Server node via AI tool connection.  
    - Output: Returns list of table names to MCP client.  
    - Edge Cases: Database connectivity issues, permission errors, or empty schema could cause failures.

  - **GetTableSchema**  
    - Type: `n8n-nodes-base.postgresTool`  
    - Role: Retrieves column names and data types for a specified table.  
    - Configuration: Parameterized query `SELECT column_name, data_type FROM information_schema.columns WHERE table_name = $1` with `tableName` provided by MCP client input.  
    - Input: Receives `tableName` parameter from MCP client via AI tool connection.  
    - Output: Returns schema details of the specified table.  
    - Edge Cases: Invalid or non-existent table name, permission errors, or database connectivity issues.

---

#### 1.3 Custom Workflow Tools for CRUD Operations

- **Overview:**  
  This block contains three custom workflow tools that encapsulate select (read), insert, and update operations. They restrict input to parameter objects rather than raw SQL, enhancing security and control.

- **Nodes Involved:**  
  - ReadTableRows (Tool Workflow)  
  - CreateTableRecords (Tool Workflow)  
  - UpdateTableRecords (Tool Workflow)  

- **Node Details:**

  - **ReadTableRows**  
    - Type: `@n8n/n8n-nodes-langchain.toolWorkflow`  
    - Role: Provides a controlled interface for reading rows from a table with optional filters.  
    - Configuration: Accepts inputs `operation` (read), `tableName`, `where` (filter object), and `values` (empty object).  
    - Input: Triggered by MCP Server node via AI tool connection.  
    - Output: Passes parameters to the main workflow for execution.  
    - Edge Cases: Invalid table or filter parameters, empty results, or malformed input.

  - **CreateTableRecords**  
    - Type: `@n8n/n8n-nodes-langchain.toolWorkflow`  
    - Role: Allows inserting new rows into a specified table with given column-value pairs.  
    - Configuration: Inputs include `operation` (insert), `tableName`, `values` (object with column-value pairs), and an empty `where` object.  
    - Input: Triggered by MCP Server node via AI tool connection.  
    - Output: Passes parameters to main workflow for execution.  
    - Edge Cases: Missing required columns, constraint violations, or invalid table names.

  - **UpdateTableRecords**  
    - Type: `@n8n/n8n-nodes-langchain.toolWorkflow`  
    - Role: Updates existing rows in a table based on filter criteria and new values.  
    - Configuration: Inputs include `operation` (update), `tableName`, `values` (columns to update), and `where` (filter object).  
    - Input: Triggered by MCP Server node via AI tool connection.  
    - Output: Passes parameters to main workflow for execution.  
    - Edge Cases: No matching rows for update, invalid filters, or constraint violations.

---

#### 1.4 Operation Dispatcher and SQL Execution

- **Overview:**  
  This block receives the operation request from the custom workflows and dispatches it to the appropriate PostgreSQL node that executes the parameterized SQL query (read, insert, or update).

- **Nodes Involved:**  
  - When Executed by Another Workflow (Execute Workflow Trigger)  
  - Operation (Switch)  
  - ReadTableRecord (PostgreSQL)  
  - CreateTableRecord (PostgreSQL)  
  - UpdateTableRecord (PostgreSQL)  

- **Node Details:**

  - **When Executed by Another Workflow**  
    - Type: `n8n-nodes-base.executeWorkflowTrigger`  
    - Role: Entry point for the custom workflow tools to trigger this main workflow with operation parameters.  
    - Configuration: Accepts inputs `operation`, `tableName`, `values`, and `where`.  
    - Input: Triggered by the three custom workflow tools.  
    - Output: Passes data to the Operation switch node.  
    - Edge Cases: Missing or malformed inputs could cause routing errors.

  - **Operation (Switch)**  
    - Type: `n8n-nodes-base.switch`  
    - Role: Routes the workflow based on the `operation` field (`read`, `insert`, or `update`).  
    - Configuration: Three outputs named READ, INSERT, and UPDATE, each triggered by matching `operation` string.  
    - Input: Receives operation data from Execute Workflow Trigger node.  
    - Output: Routes to corresponding PostgreSQL node.  
    - Edge Cases: Unsupported or missing operation values cause no routing.

  - **ReadTableRecord**  
    - Type: `n8n-nodes-base.postgres`  
    - Role: Executes a parameterized SELECT query to read rows from a table with optional WHERE filters.  
    - Configuration:  
      - Query dynamically constructed using `tableName` and `where` parameters.  
      - Uses positional parameters for safe query replacement.  
      - Example query snippet:  
        ```sql
        SELECT * FROM {{ $json.tableName }}
        WHERE key1 = $1 AND key2 = $2 ...
        ```  
      - Query replacements are values from `where` object.  
    - Input: Routed from Operation switch on `read`.  
    - Output: Returns query results to MCP client.  
    - Edge Cases: Empty or invalid filters, SQL errors, or connection issues.

  - **CreateTableRecord**  
    - Type: `n8n-nodes-base.postgres`  
    - Role: Executes an INSERT query to add a new row to a specified table.  
    - Configuration:  
      - Query dynamically constructed with columns and values from `values` object.  
      - Uses positional parameters for safe insertion.  
      - Example query snippet:  
        ```sql
        INSERT INTO {{ $json.tableName }} (col1, col2, ...)
        VALUES ($1, $2, ...)
        ```  
      - Query replacements are values from `values` object.  
    - Input: Routed from Operation switch on `insert`.  
    - Output: Returns insert confirmation or inserted row info.  
    - Edge Cases: Constraint violations, missing columns, or invalid data types.

  - **UpdateTableRecord**  
    - Type: `n8n-nodes-base.postgres`  
    - Role: Executes an UPDATE query to modify existing rows based on filter criteria.  
    - Configuration:  
      - Query dynamically constructed with SET clauses from `values` and WHERE clauses from `where`.  
      - Uses positional parameters for safe updates.  
      - Example query snippet:  
        ```sql
        UPDATE {{ $json.tableName }}
        SET col1 = $1, col2 = $2, ...
        WHERE key1 = $n, key2 = $n+1, ...
        ```  
      - Query replacements are concatenated values from `values` and `where`.  
    - Input: Routed from Operation switch on `update`.  
    - Output: Returns update confirmation or affected row count.  
    - Edge Cases: No matching rows, constraint violations, or invalid filters.

---

#### 1.5 Response Handling and Security Notes

- **Overview:**  
  This block includes sticky notes that provide important guidance on security, usage, and customization of the MCP server.

- **Nodes Involved:**  
  - Sticky Note1 (Security best practices)  
  - Sticky Note2 (Workflow overview and usage instructions)  
  - Sticky Note3 (Authentication reminder)

- **Node Details:**

  - **Sticky Note1**  
    - Content: Advises against allowing raw SQL statements from agents to prevent security risks such as data leaks or deletion. Encourages parameterized queries to mitigate SQL injection.  
    - Link: [PostgreSQL Node Documentation](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.postgres/)

  - **Sticky Note2**  
    - Content: Detailed explanation of the workflow’s purpose, how it works, usage instructions, requirements, and customization tips.  
    - Links:  
      - MCP Server Trigger docs: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-langchain.mcptrigger/#integrating-with-claude-desktop  
      - Official MCP reference implementation: https://github.com/modelcontextprotocol/servers/tree/main/src/postgres  
      - MCP Client (Claude Desktop) download: https://claude.ai/download

  - **Sticky Note3**  
    - Content: Reminder to always enable authentication on the MCP server trigger before production deployment to secure access.

---

### 3. Summary Table

| Node Name               | Node Type                              | Functional Role                                  | Input Node(s)                | Output Node(s)               | Sticky Note                                                                                      |
|-------------------------|--------------------------------------|-------------------------------------------------|-----------------------------|-----------------------------|------------------------------------------------------------------------------------------------|
| PostgreSQL MCP Server    | MCP Trigger                          | Entry point for MCP client requests              | ListTables, GetTableSchema, ReadTableRows, CreateTableRecords, UpdateTableRecords | None (trigger node)          | See Sticky Note: "Set up an MCP Server Trigger" [link](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-langchain.mcptrigger) |
| ListTables              | PostgreSQL Tool                      | Lists all tables in public schema                 | MCP Server                  | MCP Server                  |                                                                                                |
| GetTableSchema          | PostgreSQL Tool                      | Retrieves schema (columns and types) for a table | MCP Server                  | MCP Server                  |                                                                                                |
| ReadTableRows           | Tool Workflow                       | Controlled read operation interface                | MCP Server                  | MCP Server                  |                                                                                                |
| CreateTableRecords      | Tool Workflow                       | Controlled insert operation interface              | MCP Server                  | MCP Server                  |                                                                                                |
| UpdateTableRecords      | Tool Workflow                       | Controlled update operation interface              | MCP Server                  | MCP Server                  |                                                                                                |
| When Executed by Another Workflow | Execute Workflow Trigger          | Receives operation requests from custom workflows | ReadTableRows, CreateTableRecords, UpdateTableRecords | Operation                   |                                                                                                |
| Operation               | Switch                              | Routes operation to correct SQL execution node    | When Executed by Another Workflow | ReadTableRecord, CreateTableRecord, UpdateTableRecord |                                                                                                |
| ReadTableRecord         | PostgreSQL                         | Executes SELECT query with parameters             | Operation                   | None                        |                                                                                                |
| CreateTableRecord       | PostgreSQL                         | Executes INSERT query with parameters             | Operation                   | None                        |                                                                                                |
| UpdateTableRecord       | PostgreSQL                         | Executes UPDATE query with parameters             | Operation                   | None                        |                                                                                                |
| Sticky Note             | Sticky Note                        | Documentation and guidance                         | None                       | None                       | "Set up an MCP Server Trigger" with link                                                        |
| Sticky Note1            | Sticky Note                        | Security best practices reminder                   | None                       | None                       | Advises against raw SQL; link to PostgreSQL node docs                                          |
| Sticky Note2            | Sticky Note                        | Workflow overview, usage, and customization info | None                       | None                       | Detailed workflow description and links                                                        |
| Sticky Note3            | Sticky Note                        | Authentication reminder                            | None                       | None                       | "Always Authenticate Your Server!"                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create MCP Server Trigger Node**  
   - Add node: `@n8n/n8n-nodes-langchain.mcpTrigger`  
   - Set `path` parameter to a unique webhook ID (e.g., `a5fd7047-e31b-4c0d-bd68-c36072c3da0d`)  
   - This node listens for MCP client requests.

2. **Create PostgreSQL Tool Nodes for Read-Only Queries**  
   - Add node: `postgresTool` named `ListTables`  
     - Set operation to `executeQuery`  
     - Query: `SELECT table_name FROM information_schema.tables WHERE table_schema = 'public'`  
     - Credentials: Configure with your PostgreSQL credentials.  
   - Add node: `postgresTool` named `GetTableSchema`  
     - Set operation to `executeQuery`  
     - Query: `SELECT column_name, data_type FROM information_schema.columns WHERE table_name = $1`  
     - Enable query replacement with parameter `tableName` from MCP input.  
     - Credentials: Same PostgreSQL credentials.

3. **Create Custom Workflow Tools for CRUD Operations**  
   For each operation (read, insert, update), create a tool workflow node:

   - **ReadTableRows**  
     - Type: `@n8n/n8n-nodes-langchain.toolWorkflow`  
     - Name: `ReadTableRows`  
     - Description: "Call this tool to read a row in the database."  
     - Workflow ID: Set to current workflow ID (self-reference)  
     - Inputs:  
       - `operation` (string, default: "read")  
       - `tableName` (string)  
       - `where` (object)  
       - `values` (object, empty)  
     - Mapping mode: Define below with schema as above.

   - **CreateTableRecords**  
     - Type: `@n8n/n8n-nodes-langchain.toolWorkflow`  
     - Name: `CreateTableRows`  
     - Description: "Call this tool to create a row in the database."  
     - Workflow ID: Current workflow ID  
     - Inputs:  
       - `operation` (string, default: "insert")  
       - `tableName` (string)  
       - `values` (object with column-value pairs)  
       - `where` (object, empty)  
     - Mapping mode: Define below.

   - **UpdateTableRecords**  
     - Type: `@n8n/n8n-nodes-langchain.toolWorkflow`  
     - Name: `UpdateTableRows`  
     - Description: "Call this tool to update rows in the database."  
     - Workflow ID: Current workflow ID  
     - Inputs:  
       - `operation` (string, default: "update")  
       - `tableName` (string)  
       - `values` (object with columns to update)  
       - `where` (object with filter criteria)  
     - Mapping mode: Define below.

4. **Create Execute Workflow Trigger Node**  
   - Add node: `executeWorkflowTrigger` named `When Executed by Another Workflow`  
   - Configure inputs: `operation`, `tableName`, `values`, `where`  
   - This node will be triggered by the three custom workflow tools.

5. **Add Switch Node to Route Operations**  
   - Add node: `switch` named `Operation`  
   - Configure three outputs:  
     - READ: condition `operation == "read"`  
     - INSERT: condition `operation == "insert"`  
     - UPDATE: condition `operation == "update"`

6. **Add PostgreSQL Nodes for Each Operation**  
   - **ReadTableRecord**  
     - Type: `postgres`  
     - Query:  
       ```sql
       SELECT * FROM {{ $json.tableName }}
       {{ $json.where && Object.keys($json.where).length > 0
         ? `WHERE ` + Object.keys($json.where).map((key,idx) => `${key} = $${idx+1}`).join(' AND ')
         : ''
       }}
       ```  
     - Query replacement: Values from `where` object  
     - Credentials: PostgreSQL credentials

   - **CreateTableRecord**  
     - Type: `postgres`  
     - Query:  
       ```sql
       INSERT INTO {{ $json.tableName }}
         ({{ Object.keys($json.values).join(',') }})
       VALUES
         ({{ Object.keys($json.values).map((_,idx) => `$${idx+1}`).join(',') }})
       ```  
     - Query replacement: Values from `values` object  
     - Credentials: PostgreSQL credentials

   - **UpdateTableRecord**  
     - Type: `postgres`  
     - Query:  
       ```sql
       UPDATE {{ $json.tableName }}
       SET
         {{ Object.keys($json.values)
           .map((key,idx) => `${key} = $${idx+1}`)
           .join(',') 
         }}
       WHERE
         {{ Object.keys($json.where)
           .map((key,idx) => `${key} = $${idx+Object.keys($json.values).length+1}`)
           .join(' AND ')
         }}
       ```  
     - Query replacement: Concatenate values from `values` and `where` objects  
     - Credentials: PostgreSQL credentials

7. **Connect Nodes**  
   - Connect MCP Server Trigger node’s AI tool outputs to:  
     - ListTables node  
     - GetTableSchema node  
     - ReadTableRows tool workflow  
     - CreateTableRecords tool workflow  
     - UpdateTableRecords tool workflow  
   - Connect the three custom workflow tools to the Execute Workflow Trigger node (`When Executed by Another Workflow`).  
   - Connect Execute Workflow Trigger node to the Operation switch node.  
   - Connect Operation switch outputs to corresponding PostgreSQL nodes:  
     - READ → ReadTableRecord  
     - INSERT → CreateTableRecord  
     - UPDATE → UpdateTableRecord

8. **Add Sticky Notes for Documentation and Security**  
   - Add sticky notes with content:  
     - MCP Server Trigger setup and link  
     - Security best practices (avoid raw SQL, use parameterized queries) with link to PostgreSQL node docs  
     - Workflow overview, usage instructions, and links to MCP client and reference implementation  
     - Reminder to enable authentication on MCP server trigger before production

9. **Configure Credentials**  
   - Create and assign PostgreSQL credentials with appropriate access rights to the PostgreSQL nodes and tools.  
   - Ensure credentials are securely stored and not exposed.

10. **Test the Workflow**  
    - Use an MCP client (e.g., Claude Desktop) to connect to the MCP server webhook URL.  
    - Test queries such as:  
      - Checking if a user exists and creating a record if not.  
      - Querying top-selling products.  
      - Counting open high-priority support tickets.  
    - Verify responses and error handling.

---

This completes the detailed analysis and reconstruction instructions for the "Build your own PostgreSQL MCP server" workflow.