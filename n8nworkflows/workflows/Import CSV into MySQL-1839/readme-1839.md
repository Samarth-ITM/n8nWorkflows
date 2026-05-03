Import CSV into MySQL

https://n8nworkflows.xyz/workflows/import-csv-into-mysql-1839


# Import CSV into MySQL

### 1. Workflow Overview

This workflow automates the process of importing data from a CSV file into an existing MySQL database table. It is designed for users who need to regularly ingest structured CSV data into MySQL without manual intervention, ensuring seamless data integration.

The workflow is organized into the following logical blocks:

- **1.1 Manual Trigger:** Initiates the workflow execution manually.
- **1.2 File Reading:** Reads the CSV file from the local server filesystem.
- **1.3 CSV Parsing:** Converts the raw CSV file content into a structured spreadsheet format.
- **1.4 Database Insertion:** Inserts the parsed data into a predefined MySQL table.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Manual Trigger

- **Overview:**  
  This block provides a manual control point to start the workflow execution. It is suitable for testing or triggering the import on demand.

- **Nodes Involved:**  
  - On clicking 'execute'

- **Node Details:**

  - **Node Name:** On clicking 'execute'  
    - **Type:** Manual Trigger  
    - **Role:** Starts the workflow when the user clicks the execute button in n8n.  
    - **Configuration:** No parameters needed; default manual trigger settings.  
    - **Inputs:** None (trigger node).  
    - **Outputs:** Connected to the "Read From File" node.  
    - **Version Requirements:** None specific; available in all current n8n versions.  
    - **Potential Errors:** None typical; user must initiate manually.  
    - **Sub-Workflow:** None.

---

#### 1.2 File Reading

- **Overview:**  
  Reads the CSV file from the server's local filesystem at a specified path.

- **Nodes Involved:**  
  - Read From File

- **Node Details:**

  - **Node Name:** Read From File  
    - **Type:** Read Binary File  
    - **Role:** Reads the CSV file `/home/node/.n8n/concerts-2023.csv` as binary data.  
    - **Configuration:**  
      - File path: `/home/node/.n8n/concerts-2023.csv` (must exist and be accessible by n8n).  
    - **Inputs:** Triggered by "On clicking 'execute'" node.  
    - **Outputs:** Passes binary content to "Convert To Spreadsheet".  
    - **Version Requirements:** None specific.  
    - **Potential Errors:**  
      - File not found or inaccessible.  
      - Permission denied errors.  
      - Empty or corrupted file content.  
    - **Sub-Workflow:** None.

---

#### 1.3 CSV Parsing

- **Overview:**  
  Converts the raw CSV binary data read from the file into a structured JSON format suitable for database insertion.

- **Nodes Involved:**  
  - Convert To Spreadsheet

- **Node Details:**

  - **Node Name:** Convert To Spreadsheet  
    - **Type:** Spreadsheet File  
    - **Role:** Parses the CSV content into structured rows and columns.  
    - **Configuration:**  
      - Options:  
        - `rawData`: true (process data exactly as present)  
        - `readAsString`: true (interprets the input as string rather than binary buffer)  
      - This ensures the CSV content is correctly parsed into JSON objects.  
    - **Inputs:** Receives binary data from "Read From File".  
    - **Outputs:** JSON-formatted data passed to "Insert into MySQL".  
    - **Version Requirements:** None specific.  
    - **Potential Errors:**  
      - Parsing errors if the CSV format is invalid or inconsistent.  
      - Malformed CSV may lead to incorrect data rows or columns.  
    - **Sub-Workflow:** None.

---

#### 1.4 Database Insertion

- **Overview:**  
  Inserts the parsed CSV data rows into a specific MySQL table with matching columns.

- **Nodes Involved:**  
  - Insert into MySQL

- **Node Details:**

  - **Node Name:** Insert into MySQL  
    - **Type:** MySQL  
    - **Role:** Inserts data into the `concerts_2023_csv` table in MySQL.  
    - **Configuration:**  
      - Table mode: name-based selection of `concerts_2023_csv` table.  
      - Columns specified: `Date, Band, ConcertName, Country, City, Location, LocationAddress` (must match the CSV headers and the MySQL table columns).  
      - No additional options configured, meaning default insert behavior applies.  
    - **Inputs:** Receives JSON data from "Convert To Spreadsheet".  
    - **Outputs:** None (terminal node).  
    - **Credentials:** Uses pre-configured MySQL credentials named "MySQL n8n articles".  
    - **Version Requirements:** None specific; ensure MySQL node supports the used features.  
    - **Potential Errors:**  
      - MySQL connection or authentication failures.  
      - Table or column mismatch errors if schema differs.  
      - Data type mismatches or insertion constraints violations.  
      - Network timeouts or query execution errors.  
    - **Sub-Workflow:** None.

---

### 3. Summary Table

| Node Name             | Node Type             | Functional Role        | Input Node(s)          | Output Node(s)         | Sticky Note                                                                                                 |
|-----------------------|-----------------------|-----------------------|------------------------|------------------------|-------------------------------------------------------------------------------------------------------------|
| On clicking 'execute'  | Manual Trigger        | Starts workflow       | —                      | Read From File         |                                                                                                             |
| Read From File        | Read Binary File       | Reads CSV from disk   | On clicking 'execute'  | Convert To Spreadsheet  | Before running the workflow please make sure you have a file on the server: `/home/node/.n8n/concerts-2023.csv` |
| Convert To Spreadsheet | Spreadsheet File       | Parses CSV data       | Read From File         | Insert into MySQL       |                                                                                                             |
| Insert into MySQL      | MySQL                  | Inserts data into DB  | Convert To Spreadsheet | —                      | See tutorial for detailed process: https://blog.n8n.io/import-csv-into-mysql                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a **Manual Trigger** node named `"On clicking 'execute'"`.  
   - Use default settings with no parameters. This node will manually initiate the workflow.

2. **Create Read Binary File Node**  
   - Add a **Read Binary File** node named `"Read From File"`.  
   - Set the `File Path` parameter to `/home/node/.n8n/concerts-2023.csv`.  
   - This node reads the CSV file from the local filesystem.

3. **Connect Manual Trigger to File Reader**  
   - Connect the output of `"On clicking 'execute'"` to the input of `"Read From File"`.

4. **Create Spreadsheet File Node**  
   - Add a **Spreadsheet File** node named `"Convert To Spreadsheet"`.  
   - In `Options`, enable:  
     - `rawData` set to `true`  
     - `readAsString` set to `true`  
   - This node converts the CSV text into structured JSON data.

5. **Connect File Reader to Spreadsheet Node**  
   - Connect the output of `"Read From File"` to the input of `"Convert To Spreadsheet"`.

6. **Create MySQL Node**  
   - Add a **MySQL** node named `"Insert into MySQL"`.  
   - Set the `Operation` to insert rows.  
   - In `Table` selection, choose or type `"concerts_2023_csv"` as the target table.  
   - Specify the `Columns` explicitly as:  
     `Date, Band, ConcertName, Country, City, Location, LocationAddress`  
   - Provide MySQL credentials: select or create credentials with access to the target database (e.g., `"MySQL n8n articles"`). Ensure credentials have insert privileges.

7. **Connect Spreadsheet Node to MySQL Node**  
   - Connect the output of `"Convert To Spreadsheet"` to the input of `"Insert into MySQL"`.

8. **Validation and Testing**  
   - Confirm that the CSV file `/home/node/.n8n/concerts-2023.csv` exists and contains the expected data.  
   - Run the workflow manually via the `"On clicking 'execute'"` node.  
   - Monitor for any errors related to file access, parsing, or database insertion.

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                  |
|-----------------------------------------------------------------------------------------------|-------------------------------------------------|
| Before running the workflow, ensure the CSV file `/home/node/.n8n/concerts-2023.csv` exists on the server with correct permissions. | File system prerequisite                         |
| Detailed tutorial explaining the import process and best practices is available here:         | https://blog.n8n.io/import-csv-into-mysql       |
| CSV file sample content includes columns: Date, Band, ConcertName, Country, City, Location, LocationAddress | CSV format specification                         |

---

This documentation provides an exhaustive and clear understanding of the workflow, enabling both manual reproduction and automated parsing for modification or integration purposes.