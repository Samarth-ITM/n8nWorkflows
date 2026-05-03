Create a table, and insert and update data in the table in Snowflake

https://n8nworkflows.xyz/workflows/create-a-table--and-insert-and-update-data-in-the-table-in-snowflake-824


# Create a table, and insert and update data in the table in Snowflake

### 1. Workflow Overview

This workflow automates the process of creating a table in Snowflake, inserting data into it, and then updating existing data within that table. It is designed for users who need a simple, linear procedure to manage Snowflake table lifecycle operations via n8n’s Snowflake integration. The workflow is structured into the following logical blocks:

- **1.1 Table Creation:** Creating a new table in Snowflake with specified schema.
- **1.2 Data Insertion:** Inserting a row of data into the newly created table.
- **1.3 Data Update:** Updating the data in the table based on predefined criteria.

Each block depends sequentially on the previous one, ensuring that operations are performed in the correct order.

---

### 2. Block-by-Block Analysis

#### 1.1 Table Creation

- **Overview:**  
  This block is responsible for creating a new table named `docs` with two columns: `id` (integer) and `name` (string) in the Snowflake database.

- **Nodes Involved:**  
  - `On clicking 'execute'`  
  - `Snowflake`  
  - `Set`

- **Node Details:**

  - **On clicking 'execute'**  
    - *Type & Role:* Manual Trigger node to initiate workflow execution.  
    - *Configuration:* No parameters, simply triggers downstream nodes when executed manually.  
    - *Input/Output:* No input; output connects to `Snowflake`.  
    - *Edge Cases:* None specific; if user does not trigger, workflow won’t run.

  - **Snowflake**  
    - *Type & Role:* Executes SQL query on Snowflake to create table.  
    - *Configuration:* Runs the query `CREATE TABLE docs (id INT, name STRING);` with operation set to `executeQuery`. Uses Snowflake credentials stored in n8n.  
    - *Input/Output:* Input from `On clicking 'execute'`, output to `Set`.  
    - *Edge Cases:*  
      - Table might already exist causing SQL error.  
      - Network or auth errors with Snowflake credentials.  
      - Query syntax errors if modified.  
    - *Version:* n8n Snowflake node v1.

  - **Set**  
    - *Type & Role:* Sets initial data to be inserted into the `docs` table.  
    - *Configuration:* Defines an object with fields `id: 1` (number) and `name: 'n8n'` (string). `KeepOnlySet` option enabled to discard any other data.  
    - *Input/Output:* Input from `Snowflake`, output to `Snowflake1`.  
    - *Edge Cases:* None significant; fixed data.

#### 1.2 Data Insertion

- **Overview:**  
  Inserts a new row into the `docs` table using the data prepared in the previous block.

- **Nodes Involved:**  
  - `Snowflake1`  
  - `Set1`

- **Node Details:**

  - **Snowflake1**  
    - *Type & Role:* Inserts data into the Snowflake table `docs`.  
    - *Configuration:*  
      - Operation inferred as `insert` (default for table + columns specified).  
      - Table set as `"docs"`.  
      - Columns specified as `"id, name"`.  
      - Credentials: same Snowflake credentials as before.  
    - *Input/Output:* Input from `Set`, output to `Set1`.  
    - *Edge Cases:*  
      - Data type mismatches.  
      - Duplicate primary key errors (if id is primary key).  
      - Connectivity/auth errors.

  - **Set1**  
    - *Type & Role:* Prepares updated data for the update operation.  
    - *Configuration:* Sets object with `id: 1` and `name: 'nodemation'`.  
    - *Input/Output:* Input from `Snowflake1`, output to `Snowflake2`.  
    - *Edge Cases:* None; static data.

#### 1.3 Data Update

- **Overview:**  
  Updates the `name` field in the `docs` table for the row with `id = 1`.

- **Nodes Involved:**  
  - `Snowflake2`

- **Node Details:**

  - **Snowflake2**  
    - *Type & Role:* Updates existing data in the Snowflake table.  
    - *Configuration:*  
      - Operation is `update`.  
      - Table name dynamically referenced from previous node parameter `{{$node["Snowflake1"].parameter["table"]}}` which resolves to `"docs"`.  
      - Columns specified as `"name"`.  
      - Credentials: same Snowflake credentials.  
    - *Input/Output:* Input from `Set1`. No output connections.  
    - *Edge Cases:*  
      - Update might fail if row with `id=1` does not exist.  
      - Credentials or network errors.  
      - Expression resolving table name might fail if node names change.  
    - *Version:* n8n Snowflake node v1.

---

### 3. Summary Table

| Node Name              | Node Type             | Functional Role               | Input Node(s)           | Output Node(s)       | Sticky Note                                       |
|------------------------|-----------------------|------------------------------|------------------------|----------------------|--------------------------------------------------|
| On clicking 'execute'   | Manual Trigger        | Starts workflow manually      | -                      | Snowflake             |                                                  |
| Snowflake              | Snowflake             | Creates table in Snowflake    | On clicking 'execute'   | Set                   |                                                  |
| Set                    | Set                   | Sets initial insert data      | Snowflake               | Snowflake1             |                                                  |
| Snowflake1             | Snowflake             | Inserts data into table       | Set                     | Set1                   |                                                  |
| Set1                   | Set                   | Sets data for update          | Snowflake1              | Snowflake2             |                                                  |
| Snowflake2             | Snowflake             | Updates existing data         | Set1                    | -                      |                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add node of type **Manual Trigger** named `On clicking 'execute'`.  
   - No special configuration.

2. **Create Snowflake Node for Table Creation**  
   - Add a **Snowflake** node named `Snowflake`.  
   - Set operation to `executeQuery`.  
   - Enter SQL query: `CREATE TABLE docs (id INT, name STRING);`  
   - Select Snowflake credentials (`Snowflake n8n Credentials`) or create new with required access.  
   - Connect output of `On clicking 'execute'` to input of this node.

3. **Create Set Node for Insert Data**  
   - Add a **Set** node named `Set`.  
   - Add two fields:  
     - `id` (Number) with value `1`  
     - `name` (String) with value `n8n`  
   - Enable `Keep Only Set` option to discard extraneous data.  
   - Connect output of `Snowflake` to input of this node.

4. **Create Snowflake Node for Insert Operation**  
   - Add a **Snowflake** node named `Snowflake1`.  
   - Set operation to `insert` (default when specifying table and columns).  
   - Set table name: `docs`  
   - Set columns: `id, name`  
   - Use the same Snowflake credentials.  
   - Connect output of `Set` to input of this node.

5. **Create Set Node for Update Data**  
   - Add a **Set** node named `Set1`.  
   - Add two fields:  
     - `id` (Number) with value `1`  
     - `name` (String) with value `nodemation`  
   - Enable `Keep Only Set` option.  
   - Connect output of `Snowflake1` to input of this node.

6. **Create Snowflake Node for Update Operation**  
   - Add a **Snowflake** node named `Snowflake2`.  
   - Set operation to `update`.  
   - Set table name expression: `={{$node["Snowflake1"].parameter["table"]}}` to dynamically reference the table `docs`.  
   - Set columns: `name` (only the column to be updated).  
   - Use the same Snowflake credentials.  
   - Connect output of `Set1` to input of this node.

7. **Save and activate workflow**  
   - Verify all node connections are correct.  
   - Ensure Snowflake credentials have permissions for DDL and DML operations (CREATE, INSERT, UPDATE).  
   - Execute manually using the trigger node.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                  |
|-------------------------------------------------------------------------------------------------|-------------------------------------------------|
| Snowflake credentials must have sufficient privileges to create tables and modify data.          | Snowflake user permissions                        |
| If the table `docs` already exists, the `CREATE TABLE` query will fail; consider adding `IF NOT EXISTS`. | SQL best practices                               |
| Dynamic table name referencing via expression depends on node names remaining unchanged.         | n8n expression documentation                     |
| For more complex workflows, consider error handling nodes to capture and manage SQL errors.      | n8n error handling                                |