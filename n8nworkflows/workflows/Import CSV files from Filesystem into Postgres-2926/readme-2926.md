Import CSV files from Filesystem into Postgres

https://n8nworkflows.xyz/workflows/import-csv-files-from-filesystem-into-postgres-2926


# Import CSV files from Filesystem into Postgres

### 1. Workflow Overview

This workflow automates the process of importing CSV files stored on a local filesystem into a PostgreSQL database table. It is primarily designed for self-hosted n8n instances where the CSV files are accessible directly via the server’s file system. Cloud users can adapt the initial file retrieval nodes to use cloud storage services such as Google Drive or Dropbox.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Manual trigger to start the workflow.
- **1.2 File Reading:** Reading the CSV file from the local filesystem.
- **1.3 Data Conversion:** Converting the binary CSV file content into a structured spreadsheet format.
- **1.4 Database Import:** Inserting or updating the data into a PostgreSQL table.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block initiates the workflow manually, allowing the user to trigger the CSV import process on demand.

- **Nodes Involved:**  
  - On clicking 'execute'

- **Node Details:**

  - **On clicking 'execute'**  
    - Type: Manual Trigger  
    - Role: Starts the workflow execution manually.  
    - Configuration: No parameters; simply triggers the workflow when clicked.  
    - Inputs: None  
    - Outputs: Connected to "Read From File" node.  
    - Version: 1  
    - Edge Cases: None typical; user must manually trigger.  
    - Sub-workflow: None

#### 2.2 File Reading

- **Overview:**  
  Reads the CSV file from the local filesystem at a specified path.

- **Nodes Involved:**  
  - Read From File

- **Node Details:**

  - **Read From File**  
    - Type: Read Binary File  
    - Role: Reads the CSV file as binary data from the server’s file system.  
    - Configuration:  
      - File Path: `/tmp/t1.csv` (hardcoded path; must exist on the server).  
    - Inputs: Receives trigger from "On clicking 'execute'".  
    - Outputs: Binary data output connected to "Convert To Spreadsheet".  
    - Version: 1  
    - Edge Cases:  
      - File not found or inaccessible (permission issues).  
      - File path hardcoded; requires manual update for different files.  
      - Large files may cause memory or timeout issues.  
    - Sub-workflow: None

#### 2.3 Data Conversion

- **Overview:**  
  Converts the binary CSV file content into a structured spreadsheet format (JSON) that can be processed by the database node.

- **Nodes Involved:**  
  - Convert To Spreadsheet

- **Node Details:**

  - **Convert To Spreadsheet**  
    - Type: Spreadsheet File  
    - Role: Parses the binary CSV data into rows and columns as JSON objects.  
    - Configuration: Default options (no special parsing options set).  
    - Inputs: Binary data from "Read From File".  
    - Outputs: JSON data passed to "Postgres" node.  
    - Version: 1  
    - Edge Cases:  
      - Malformed CSV files may cause parsing errors.  
      - Encoding issues if CSV is not UTF-8.  
    - Sub-workflow: None

#### 2.4 Database Import

- **Overview:**  
  Inserts or updates the parsed CSV data into the PostgreSQL table `t1` in the `public` schema.

- **Nodes Involved:**  
  - Postgres

- **Node Details:**

  - **Postgres**  
    - Type: PostgreSQL  
    - Role: Inserts data into the PostgreSQL table `t1`.  
    - Configuration:  
      - Table: `t1`  
      - Schema: `public`  
      - Columns:  
        - `id` (number)  
        - `name` (string)  
      - Mapping Mode: Auto map input data columns to table columns.  
      - Matching Columns: `id` (used for upsert or update logic).  
      - Attempt to convert types: Disabled (false).  
      - Convert fields to string: Disabled (false).  
    - Inputs: JSON data from "Convert To Spreadsheet".  
    - Outputs: None (end node).  
    - Credentials: Uses stored PostgreSQL credentials named "Postgres account".  
    - Version: 2.5  
    - Edge Cases:  
      - Database connection errors (auth, network).  
      - Data type mismatches if CSV data does not conform to table schema.  
      - Duplicate key conflicts if matching columns are not unique or properly handled.  
      - Table must exist prior to running workflow.  
    - Sub-workflow: None

---

### 3. Summary Table

| Node Name           | Node Type           | Functional Role                 | Input Node(s)           | Output Node(s)         | Sticky Note                                                                                          |
|---------------------|---------------------|--------------------------------|------------------------|------------------------|----------------------------------------------------------------------------------------------------|
| On clicking 'execute'| Manual Trigger      | Starts workflow manually       | None                   | Read From File         |                                                                                                    |
| Read From File      | Read Binary File     | Reads CSV file from filesystem | On clicking 'execute'  | Convert To Spreadsheet  |                                                                                                    |
| Convert To Spreadsheet| Spreadsheet File   | Converts binary CSV to JSON    | Read From File         | Postgres               |                                                                                                    |
| Postgres            | PostgreSQL           | Inserts/updates data into DB   | Convert To Spreadsheet | None                   |                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Manual Trigger node:**  
   - Name: `On clicking 'execute'`  
   - No parameters needed.

3. **Add a Read Binary File node:**  
   - Name: `Read From File`  
   - Set `File Path` to `/tmp/t1.csv` (adjust path as needed).  
   - Connect output of `On clicking 'execute'` to input of this node.

4. **Add a Spreadsheet File node:**  
   - Name: `Convert To Spreadsheet`  
   - Use default options (no special configuration).  
   - Connect output of `Read From File` to input of this node.

5. **Add a PostgreSQL node:**  
   - Name: `Postgres`  
   - Set `Table` to `t1`.  
   - Set `Schema` to `public`.  
   - Configure columns mapping:  
     - `id` as number  
     - `name` as string  
   - Set `Mapping Mode` to `Auto Map Input Data`.  
   - Set `Matching Columns` to `id` (for upsert behavior).  
   - Disable `Attempt to Convert Types`.  
   - Disable `Convert Fields to String`.  
   - Connect output of `Convert To Spreadsheet` to input of this node.  
   - Assign PostgreSQL credentials (create or select existing credentials with access to your database).

6. **Save and activate the workflow.**

7. **Ensure prerequisites:**  
   - The CSV file `/tmp/t1.csv` exists and is readable by n8n.  
   - The PostgreSQL database `db01` is accessible.  
   - Table `t1` exists with schema:  
     ```sql
     create table t1(id int, name varchar(10));
     ```  
   - Credentials are correctly configured.

8. **Execute the workflow manually by clicking the trigger node.**

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                  |
|----------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow is designed for self-hosted users with direct file system access. Cloud users should replace the file reading nodes with cloud storage nodes like Google Drive or Dropbox. | Workflow description and disclaimer.                                                            |
| PostgreSQL CREATE TABLE command reference: https://www.postgresql.org/docs/current/sql-createtable.html | Useful for creating custom tables before importing CSV data.                                    |
| Ensure CSV files are UTF-8 encoded and properly formatted to avoid parsing errors.                  | General CSV import best practices.                                                              |
| Example CSV file content used in this workflow:                                                    | id,name<br>1,a<br>2,b<br>3,c                                                                     |
| Workflow image preview available in original documentation (not included here).                     | Refer to original workflow documentation or fileId:957 for visual reference.                    |

---

This document provides a complete and detailed reference for understanding, reproducing, and modifying the "Import CSV files from Filesystem into Postgres" n8n workflow.