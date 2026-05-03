Create a table in Quest DB and insert data

https://n8nworkflows.xyz/workflows/create-a-table-in-quest-db-and-insert-data-592


# Create a table in Quest DB and insert data

### 1. Workflow Overview

This workflow automates the process of creating a table in QuestDB and inserting data into it. It is designed for scenarios where a user wants to programmatically set up a new database table and populate it with initial data in QuestDB, a high-performance time series database. The workflow is divided into three logical blocks:

- **1.1 Input Trigger:** Manual initiation of the workflow.
- **1.2 Table Creation:** Executes a SQL query to create a new table in QuestDB.
- **1.3 Data Insertion:** Prepares and inserts data rows into the newly created table.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Trigger

- **Overview:**  
  This block starts the workflow manually when a user clicks "execute." It allows manual testing and controlled initiation.

- **Nodes Involved:**  
  - On clicking 'execute'

- **Node Details:**  
  - **On clicking 'execute'**  
    - Type: Manual Trigger  
    - Configuration: No parameters; triggers workflow on manual execution.  
    - Input: None  
    - Output: Triggers downstream Query Execution node.  
    - Failures: None anticipated; manual trigger.  
    - Notes: Serves as the workflow entry point.

#### 1.2 Table Creation

- **Overview:**  
  Executes a SQL command to create a new table named `test` with two columns (`id` as integer, `name` as string) in QuestDB.

- **Nodes Involved:**  
  - QuestDB (first instance)

- **Node Details:**  
  - **QuestDB**  
    - Type: QuestDB node for executing SQL queries  
    - Configuration:  
      - Operation: `executeQuery`  
      - Query: `CREATE TABLE test (id INT, name STRING);`  
      - Credentials: QuestDB credentials must be configured to connect to the database.  
    - Input: Triggered by Manual Trigger node.  
    - Output: Passes data to the Set node for data preparation.  
    - Failures:  
      - SQL syntax errors if table already exists or query malformed.  
      - Connection/authentication issues with QuestDB credentials.  
      - Timeout if QuestDB server is unresponsive.  
    - Version Requirements: Compatible with n8n QuestDB node version 1.  
    - Notes: Ensures table creation before data insertion.

#### 1.3 Data Insertion

- **Overview:**  
  Prepares a data record and inserts it into the previously created `test` table via QuestDB.

- **Nodes Involved:**  
  - Set  
  - QuestDB1 (second QuestDB node instance)

- **Node Details:**  
  - **Set**  
    - Type: Set node used to define the data payload  
    - Configuration:  
      - Sets two fields:  
        - `id` (type: number) — no explicit value set (likely empty or to be dynamically set)  
        - `name` (type: string) — fixed value `"Tanay"`  
      - No additional options enabled.  
    - Input: Receives trigger from previous QuestDB node after table creation.  
    - Output: Provides structured data to QuestDB insertion node.  
    - Failures: Potential empty or undefined `id` value if not set properly.  
    - Notes: Prepares row data for insertion.

  - **QuestDB1**  
    - Type: QuestDB node for inserting data  
    - Configuration:  
      - Operation: Default (implied insert based on parameters)  
      - Table: `test`  
      - Columns: `id, name`  
      - Credentials: QuestDB credentials (same as first QuestDB node).  
    - Input: Receives data from Set node.  
    - Output: None (end of workflow)  
    - Failures:  
      - Data mismatch if `id` is undefined or invalid type.  
      - Connection/authentication issues.  
      - Table not existing if table creation failed.  
    - Notes: Inserts the prepared data row into the `test` table.

---

### 3. Summary Table

| Node Name           | Node Type                 | Functional Role          | Input Node(s)           | Output Node(s)     | Sticky Note                          |
|---------------------|---------------------------|-------------------------|-------------------------|--------------------|------------------------------------|
| On clicking 'execute'| Manual Trigger            | Workflow start trigger  | None                    | QuestDB            |                                    |
| QuestDB             | QuestDB (executeQuery)    | Create `test` table     | On clicking 'execute'   | Set                |                                    |
| Set                 | Set                       | Prepare data to insert  | QuestDB                 | QuestDB1           |                                    |
| QuestDB1            | QuestDB                   | Insert data into table  | Set                     | None               |                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a **Manual Trigger** node named `On clicking 'execute'`. No configuration needed.

2. **Add QuestDB Node for Table Creation**  
   - Add a **QuestDB** node named `QuestDB`.  
   - Set operation to `executeQuery`.  
   - Enter SQL query: `CREATE TABLE test (id INT, name STRING);`  
   - Configure QuestDB credentials (host, port, username, password).  
   - Connect `On clicking 'execute'` node's output to this node's input.

3. **Add Set Node to Prepare Data**  
   - Add a **Set** node named `Set`.  
   - In Values to Set:  
     - Add a Number field named `id` with no value preset (can be set dynamically or left empty).  
     - Add a String field named `name` with value `"Tanay"`.  
   - Connect the output of `QuestDB` node to this node.

4. **Add QuestDB Node for Data Insertion**  
   - Add another **QuestDB** node named `QuestDB1`.  
   - Set operation to default or insert mode.  
   - Set table to `test`.  
   - Specify columns as `id, name`.  
   - Use the same QuestDB credentials as before.  
   - Connect `Set` node's output to this node.

5. **Activate and Test Workflow**  
   - Save the workflow.  
   - Execute the manual trigger node to run end-to-end: create table and insert data.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                 |
|-------------------------------------------------------------------------------------------------|------------------------------------------------|
| QuestDB requires proper credentials and running instance to accept SQL queries and inserts.     | Official QuestDB docs: https://questdb.io/docs/ |
| The `id` field in the Set node is not preset; ensure to provide a valid number during runtime.  | Workflow data preparation best practices       |