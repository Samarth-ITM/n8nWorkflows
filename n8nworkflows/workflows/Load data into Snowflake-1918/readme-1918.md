Load data into Snowflake

https://n8nworkflows.xyz/workflows/load-data-into-snowflake-1918


# Load data into Snowflake

### 1. Workflow Overview

This workflow automates the ingestion of CSV data into a Snowflake database. It is designed for use cases where data is regularly published as CSV files on the web and needs to be imported into Snowflake tables with matching column structures. The workflow consists of four main logical blocks:

- **1.1 Trigger and Data Download:** Initiates workflow execution manually and downloads the CSV file from a specified URL.
- **1.2 CSV Parsing:** Parses the downloaded CSV file into a structured format that n8n can process.
- **1.3 Data Mapping:** Maps the parsed CSV data fields to the target Snowflake database column names.
- **1.4 Data Insertion:** Inserts the mapped data as new rows into the configured Snowflake table.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger and Data Download

- **Overview:**  
  This block starts the workflow manually and downloads the CSV file from a web source.

- **Nodes Involved:**  
  - When clicking "Execute Workflow" (Manual Trigger)  
  - HTTP Request

- **Node Details:**

  - **When clicking "Execute Workflow"**  
    - Type: Manual Trigger  
    - Role: Initiates the workflow execution on demand.  
    - Configuration: Default manual trigger, no parameters set.  
    - Input: None (trigger node)  
    - Output: Passes execution to HTTP Request node.  
    - Edge Cases: No input failures; user must manually trigger.  

  - **HTTP Request**  
    - Type: HTTP Request  
    - Role: Downloads the CSV file from a specified URL.  
    - Configuration:  
      - URL: `https://n8niostorageaccount.blob.core.windows.net/n8nio-strapi-blobs-prod/assets/example_c0b48ce677.csv?updated_at=2023-05-30T10:36:21.820Z`  
      - Response format: File (binary data expected)  
      - HTTP Method: GET (default)  
    - Input: Trigger node output  
    - Output: Binary data of the CSV file to Spreadsheet File node  
    - Edge Cases:  
      - Network errors (timeout, unreachable URL)  
      - File format errors if the content is not CSV or is corrupted  
      - HTTP errors (404, 403, etc.)  

#### 1.2 CSV Parsing

- **Overview:**  
  Parses the binary CSV file into structured JSON data rows accessible within n8n.

- **Nodes Involved:**  
  - Spreadsheet File

- **Node Details:**

  - **Spreadsheet File**  
    - Type: Spreadsheet File  
    - Role: Parses the CSV binary file into JSON rows.  
    - Configuration:  
      - No special options set (defaults used), implying first row as header row.  
      - Input expects binary data from HTTP Request node.  
    - Input: CSV file binary from HTTP Request  
    - Output: JSON array of rows for downstream use  
    - Edge Cases:  
      - Malformed CSV files causing parse errors  
      - Large file sizes leading to performance issues or timeouts  
      - Character encoding issues if not UTF-8  
    - Version-specific: Requires n8n version supporting Spreadsheet File node (typeVersion 1 used here)  

#### 1.3 Data Mapping

- **Overview:**  
  Ensures the data fields from parsed CSV rows are correctly mapped to the Snowflake table column names.

- **Nodes Involved:**  
  - Set

- **Node Details:**

  - **Set**  
    - Type: Set  
    - Role: Remaps data fields to match Snowflake table columns and filters to only required columns.  
    - Configuration:  
      - Keeps only the fields: `id`, `first_name`, `last_name`  
      - Uses expressions to assign values from CSV JSON data:  
        - `id = {{$json.id}}`  
        - `first_name = {{$json.first_name}}`  
        - `last_name = {{$json.last_name}}`  
      - Dot notation disabled (flat JSON objects expected)  
      - Keeps only the specified fields (removes any extra data)  
    - Input: JSON rows from Spreadsheet File  
    - Output: Filtered and mapped JSON objects for Snowflake insertion  
    - Edge Cases:  
      - Missing fields in CSV data causing undefined values  
      - Data type mismatches (e.g., id expected as number, but CSV has string)  
      - CSV column name changes not reflected here causing mapping failures  
    - Version-specific: Uses typeVersion 2 for Set node  

#### 1.4 Data Insertion

- **Overview:**  
  Inserts the mapped data rows as new records into the Snowflake database table.

- **Nodes Involved:**  
  - Snowflake

- **Node Details:**

  - **Snowflake**  
    - Type: Snowflake node  
    - Role: Inserts rows into the Snowflake `users` table.  
    - Configuration:  
      - Table: `users`  
      - Columns: `id,first_name,last_name` (must correspond exactly to Set node output)  
      - Operation: Insert (default)  
    - Credentials: Uses stored Snowflake credentials named "Snowflake account" (credential ID 23)  
    - Input: JSON objects from Set node  
    - Output: Result of insertion operation (success/failure response)  
    - Edge Cases:  
      - Authentication errors (invalid or expired credentials)  
      - SQL errors (e.g., constraint violations, data type mismatches)  
      - Network connectivity issues to Snowflake  
      - Partial insert failures if batch insert is used  
    - Version-specific: typeVersion 1  
    - Sub-workflow: None  

---

### 3. Summary Table

| Node Name                    | Node Type          | Functional Role                     | Input Node(s)               | Output Node(s)          | Sticky Note                                 |
|------------------------------|--------------------|-----------------------------------|-----------------------------|-------------------------|---------------------------------------------|
| When clicking "Execute Workflow" | Manual Trigger    | Starts workflow manually           | None                        | HTTP Request            |                                             |
| HTTP Request                 | HTTP Request       | Downloads CSV file from web URL    | When clicking "Execute Workflow" | Spreadsheet File        |                                             |
| Spreadsheet File             | Spreadsheet File   | Parses CSV file to JSON rows       | HTTP Request                | Set                     |                                             |
| Set                         | Set                | Maps CSV data fields to DB columns | Spreadsheet File            | Snowflake               |                                             |
| Snowflake                   | Snowflake          | Inserts rows into Snowflake table  | Set                         | None                    |                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `When clicking "Execute Workflow"`  
   - Type: Manual Trigger  
   - Leave default settings.

2. **Create HTTP Request Node**  
   - Name: `HTTP Request`  
   - Type: HTTP Request  
   - Set URL to: `https://n8niostorageaccount.blob.core.windows.net/n8nio-strapi-blobs-prod/assets/example_c0b48ce677.csv?updated_at=2023-05-30T10:36:21.820Z`  
   - Under Options → Response, set Response Format to `File` (to get binary file)  
   - Connect output of Manual Trigger to input of HTTP Request.

3. **Create Spreadsheet File Node**  
   - Name: `Spreadsheet File`  
   - Type: Spreadsheet File  
   - No special options needed (default parses first row as header)  
   - Connect output of HTTP Request to input of Spreadsheet File.

4. **Create Set Node**  
   - Name: `Set`  
   - Type: Set  
   - Enable "Keep Only Set" option (to remove unwanted fields)  
   - Add fields with expressions:  
     - Number type:  
       - `id` with value `{{$json.id}}`  
       - `first_name` with value `{{$json.first_name}}`  
     - String type:  
       - `last_name` with value `{{$json.last_name}}`  
   - Connect output of Spreadsheet File to input of Set node.

5. **Create Snowflake Node**  
   - Name: `Snowflake`  
   - Type: Snowflake  
   - Configure credentials for Snowflake account (OAuth2 or username/password) with required permissions to insert into the target database.  
   - Set Table to `users`  
   - Set Columns to `id,first_name,last_name` (comma-separated, matching Set node fields)  
   - Connect output of Set node to input of Snowflake node.

6. **Activate and Test Workflow**  
   - Save workflow, activate if desired.  
   - Manually trigger to verify successful CSV download, parsing, mapping, and insertion into Snowflake.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                                               |
|-----------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| Ensure that the Snowflake credentials used have INSERT privileges on the target table and database. | Snowflake user permissions documentation                                                                                      |
| The CSV file URL is static; update URL or add dynamic parameterization if loading different files.   | Useful for workflows processing daily or periodic CSV updates                                                                 |
| For large CSV files, consider splitting or batching inserts to avoid timeout or memory issues.      | n8n community forums and Snowflake best practices                                                                             |
| Use the "Keep Only Set" option in the Set node to prevent accidental insertion of unwanted fields.  | n8n Set node documentation                                                                                                    |
| For troubleshooting HTTP Request node failures, check network connectivity and HTTP status codes.   | https://docs.n8n.io/nodes/n8n-nodes-base.httpRequest/                                                                         |

---

This documentation provides a complete and detailed overview of the "Load data into Snowflake" workflow, enabling efficient understanding, modification, and reproduction of the integration.