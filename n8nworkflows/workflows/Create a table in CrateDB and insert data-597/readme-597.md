Create a table in CrateDB and insert data

https://n8nworkflows.xyz/workflows/create-a-table-in-cratedb-and-insert-data-597


# Create a table in CrateDB and insert data

### 1. Workflow Overview

This workflow demonstrates how to create a table in CrateDB and subsequently insert data into it using n8n. It is designed as a companion example for users referencing the CrateDB node documentation. The workflow is triggered manually to sequentially execute SQL commands and data insertion, showcasing the fundamental integration between n8n and CrateDB.

Logical blocks:

- **1.1 Manual Trigger**: Initiates the workflow execution.
- **1.2 Table Creation in CrateDB**: Executes a SQL query to create a new table.
- **1.3 Data Preparation**: Sets the data object to insert into the table.
- **1.4 Data Insertion into CrateDB**: Inserts the prepared data into the created table.

---

### 2. Block-by-Block Analysis

#### 1.1 Manual Trigger

- **Overview:**  
  This block serves as the starting point for the workflow. It waits for a manual execution command from the user.

- **Nodes Involved:**  
  - On clicking 'execute'

- **Node Details:**  
  - **Name:** On clicking 'execute'  
  - **Type:** Manual Trigger (n8n-nodes-base.manualTrigger)  
  - **Role:** Initiates the workflow manually.  
  - **Configuration:** Default empty configuration; no parameters needed.  
  - **Expressions/Variables:** None.  
  - **Inputs/Outputs:** No inputs; output connects to the CrateDB table creation node.  
  - **Version Requirements:** Compatible with n8n version 0.123.0+ (standard manual trigger).  
  - **Potential Failures:** None expected; manual trigger is simple.  
  - **Sub-workflow:** None.

#### 1.2 Table Creation in CrateDB

- **Overview:**  
  This block creates a new table named `test` with columns `id` (integer) and `name` (string) in the CrateDB database.

- **Nodes Involved:**  
  - CrateDB (first instance)

- **Node Details:**  
  - **Name:** CrateDB  
  - **Type:** CrateDB node (n8n-nodes-base.crateDb)  
  - **Role:** Executes a raw SQL query to create a table.  
  - **Configuration:**  
    - Operation: Execute Query  
    - Query: `CREATE TABLE test (id INT, name STRING);`  
    - Credential: Uses stored CrateDB credentials (`cratedb_creds`).  
  - **Expressions/Variables:** None, static SQL query.  
  - **Inputs/Outputs:**  
    - Input: From manual trigger node.  
    - Output: Passes data to the Set node.  
  - **Version Requirements:** Works with n8n versions supporting CrateDB node (introduced in n8n 0.143.0+).  
  - **Potential Failures:**  
    - SQL syntax errors if query malformed.  
    - Table already exists error on repeated execution.  
    - Credential authentication errors.  
    - Connection timeout or network issues.  
  - **Sub-workflow:** None.

#### 1.3 Data Preparation

- **Overview:**  
  This block prepares a single data record to be inserted into the `test` table, setting `id` to 0 and `name` to "n8n".

- **Nodes Involved:**  
  - Set

- **Node Details:**  
  - **Name:** Set  
  - **Type:** Set node (n8n-nodes-base.set)  
  - **Role:** Defines the data object for insertion.  
  - **Configuration:**  
    - Values set:  
      - `id`: Number type, value `0`  
      - `name`: String type, value `"n8n"`  
    - Options: No additional options enabled.  
  - **Expressions/Variables:** Static values set directly.  
  - **Inputs/Outputs:**  
    - Input: From CrateDB table creation node.  
    - Output: Connects to CrateDB insert node.  
  - **Version Requirements:** Standard Set node, available in all recent versions.  
  - **Potential Failures:** Minimal; errors could occur if data types mismatched but unlikely here.  
  - **Sub-workflow:** None.

#### 1.4 Data Insertion into CrateDB

- **Overview:**  
  Inserts the prepared data into the `test` table in CrateDB.

- **Nodes Involved:**  
  - CrateDB1 (second CrateDB node)

- **Node Details:**  
  - **Name:** CrateDB1  
  - **Type:** CrateDB node (n8n-nodes-base.crateDb)  
  - **Role:** Inserts data into a specified table.  
  - **Configuration:**  
    - Operation: Insert  
    - Table: `test`  
    - Columns: `id, name` (matching the set node's fields)  
    - Credential: Uses same CrateDB credentials (`cratedb_creds`).  
  - **Expressions/Variables:** Maps fields `id` and `name` from input data.  
  - **Inputs/Outputs:**  
    - Input: From Set node (data prepared for insertion).  
    - Output: Final output of the workflow; no further connections.  
  - **Version Requirements:** Requires CrateDB node with insert operation support.  
  - **Potential Failures:**  
    - Insertion errors if table does not exist or data types mismatch.  
    - Credential or connection errors.  
    - Duplicate primary key errors if constraints exist.  
  - **Sub-workflow:** None.

---

### 3. Summary Table

| Node Name             | Node Type           | Functional Role                  | Input Node(s)          | Output Node(s)       | Sticky Note |
|-----------------------|---------------------|---------------------------------|-----------------------|----------------------|-------------|
| On clicking 'execute'  | Manual Trigger      | Initiates manual workflow start | —                     | CrateDB              |             |
| CrateDB               | CrateDB             | Creates `test` table via SQL    | On clicking 'execute'  | Set                  |             |
| Set                   | Set                 | Prepares data object to insert  | CrateDB               | CrateDB1             |             |
| CrateDB1              | CrateDB             | Inserts data into `test` table  | Set                   | —                    |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**  
   - Add node of type "Manual Trigger".  
   - Leave configuration as default.  
   - Position it as the start node.

2. **Add a CrateDB Node for Table Creation**  
   - Add node of type "CrateDB".  
   - Set Operation to "Execute Query".  
   - For Query, enter: `CREATE TABLE test (id INT, name STRING);`  
   - Set Credentials: Select or create CrateDB credentials (`cratedb_creds`) with access to your CrateDB instance.  
   - Connect Manual Trigger node output to this node's input.

3. **Add a Set Node to Prepare Data**  
   - Add node of type "Set".  
   - Under Values, add two fields:  
     - `id` of type Number with value `0`.  
     - `name` of type String with value `"n8n"`.  
   - Connect CrateDB table creation node output to this Set node.

4. **Add a CrateDB Node for Data Insertion**  
   - Add another CrateDB node.  
   - Set Operation to "Insert".  
   - Set Table to `test`.  
   - Specify Columns as `id, name`.  
   - Use the same CrateDB credentials as before (`cratedb_creds`).  
   - Connect the Set node output to this node.

5. **Save and Execute**  
   - Save the workflow.  
   - Click "Execute Workflow" manually to create the table and insert the data.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                   |
|-----------------------------------------------------------------------------------------------------------|--------------------------------------------------|
| This workflow acts as a companion example specifically designed to demonstrate CrateDB node usage in n8n. | n8n CrateDB Node Documentation                    |
| Ensure your CrateDB credentials (`cratedb_creds`) are correctly configured with connection details.       | n8n Credentials Setup                             |
| Repeated execution will fail at table creation if table `test` already exists; handle with conditional logic if needed. | Best practice for idempotent workflows            |

---

This documentation fully describes the structure and functional details of the provided workflow, enabling effective understanding, reproduction, and modification.