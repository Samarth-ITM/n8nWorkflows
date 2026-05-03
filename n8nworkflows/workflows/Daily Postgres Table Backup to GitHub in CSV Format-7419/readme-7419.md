Daily Postgres Table Backup to GitHub in CSV Format

https://n8nworkflows.xyz/workflows/daily-postgres-table-backup-to-github-in-csv-format-7419


# Daily Postgres Table Backup to GitHub in CSV Format

### 1. Workflow Overview

This workflow automates daily backups of PostgreSQL tables by exporting each table’s data into CSV files and uploading them to a GitHub repository. It targets use cases where regular, versioned backups of table data are required for audit, recovery, or data sharing purposes.

The logic is organized into these functional blocks:

- **1.1 Daily Trigger Setup:** Initiates the workflow every 24 hours.
- **1.2 GitHub Repository Inspection:** Retrieves the current list of CSV backup files stored in the GitHub repository.
- **1.3 PostgreSQL Tables Discovery:** Lists all user tables from the PostgreSQL database.
- **1.4 Data Extraction and Conversion:** For each table, fetches all data and converts it to CSV format.
- **1.5 Backup Upload Logic:** For each CSV file, checks if it exists in the repository, then either updates the existing file or uploads it as a new file.

---

### 2. Block-by-Block Analysis

#### 2.1 Daily Trigger Setup

- **Overview:** This block triggers the workflow once every 24 hours to ensure daily backups.
- **Nodes Involved:**  
  - Daily Schedule
- **Node Details:**  
  - **Daily Schedule**
    - Type: Schedule Trigger  
    - Configured with an interval of 24 hours (hourly interval set to 24)  
    - No inputs; outputs trigger signal to start the workflow  
    - Potential failures: scheduler misconfiguration or runtime error in n8n scheduler  
    - No sub-workflow references

#### 2.2 GitHub Repository Inspection

- **Overview:** Retrieves the list of files currently stored in the GitHub backup repository to determine which tables have existing backups.
- **Nodes Involved:**  
  - List files from repository [GITHUB]  
  - Combine file names [GITHUB]  
  - Sticky Note2
- **Node Details:**  
  - **List files from repository [GITHUB]**  
    - Type: GitHub node (resource: file, operation: list)  
    - Auth: OAuth2 credentials for GitHub account  
    - Parameters specify repository owner, repo name, and path (root, empty string)  
    - Outputs a list of files (including table backup CSVs and possibly other files)  
    - Always outputs data  
    - Failures can include auth errors, rate limits, or repo access issues  
  - **Combine file names [GITHUB]**  
    - Type: Item Lists node  
    - Operation: aggregate items by combining all `name` fields into a list  
    - Takes output from listing node and creates an aggregated list of filenames for lookup  
  - **Sticky Note2**  
    - Content explains the purpose of this block: obtain a list of current files in the repo, some are tables, some are README files

#### 2.3 PostgreSQL Tables Discovery

- **Overview:** Queries the PostgreSQL database to retrieve the list of tables in the public schema to prepare for data extraction.
- **Nodes Involved:**  
  - List tables1  
  - Loop Over Items  
  - Sticky Note6 (named Sticky Note in JSON, placed near this block)
- **Node Details:**  
  - **List tables1**  
    - Type: Postgres node  
    - Operation: select  
    - Queries the `information_schema.tables` to list all tables where `table_schema = 'public'`  
    - Returns all matching rows  
    - Uses PostgreSQL credentials configured for the source DB  
    - Potential failures: connection/auth errors, query timeouts  
  - **Loop Over Items**  
    - Type: SplitInBatches  
    - Splits the list of tables into batches to process sequentially  
    - This batching helps avoid overloading downstream nodes or GitHub API rate limits  
  - **Sticky Note** (named Sticky Note with color 5)  
    - Explains this block extracts PostgreSQL table data and converts to CSV  

#### 2.4 Data Extraction and Conversion

- **Overview:** For each table, extracts the entire content and converts it to a CSV file format to prepare for backup upload.
- **Nodes Involved:**  
  - Loop Over Items (connected from List tables1, also reused)  
  - List tables  
  - Code  
  - Convert to File1  
  - Sticky Note3 (image of postgres table)
- **Node Details:**  
  - **List tables**  
    - Type: Postgres node  
    - Operation: select  
    - Dynamically fetches all rows from the current table name passed from previous node using expression `{{$json.table_name}}`  
    - Reads all rows from the table in the public schema  
  - **Code**  
    - Type: Code node (JavaScript)  
    - Simply returns all input data unchanged (`return $input.all();`)  
    - Serves as a pass-through or potential extension point  
  - **Convert to File1**  
    - Type: ConvertToFile node  
    - Converts input data to CSV file binary  
    - Filename is dynamically set as current table name from the node `List tables1` (expression)  
    - Outputs binary file with CSV content in `data` property  
  - **Sticky Note3**  
    - Shows a visual diagram of the Postgres table extraction process  

#### 2.5 Backup Upload Logic

- **Overview:** Checks if each CSV file already exists in GitHub, then updates the existing file or uploads a new one accordingly.
- **Nodes Involved:**  
  - Split to single items  
  - Check if file exists in repository (If node)  
  - Update file [GITHUB]  
  - Upload file [GITHUB]  
  - Sticky Note1  
  - Sticky Note4 (github backup image)
- **Node Details:**  
  - **Split to single items**  
    - Type: SplitInBatches  
    - Batch size 1, processes one file at a time for comparison and upload  
  - **Check if file exists in repository**  
    - Type: If node  
    - Checks if the current filename from GitHub (`Combine file names [GITHUB]`) contains the CSV file name (`$binary.data.fileName`) from conversion step  
    - If true, the file exists and needs an update  
    - If false, upload a new file  
  - **Update file [GITHUB]**  
    - Type: GitHub node  
    - Operation: edit existing file  
    - Uses OAuth2 credentials  
    - Commits with message `backup-<timestamp>`  
    - Reads file path dynamically from binary file name  
    - Potential failures: file lock, auth, rate limits, conflicts  
  - **Upload file [GITHUB]**  
    - Type: GitHub node  
    - Operation: create new file in repo  
    - Same credential and commit message pattern as update node  
  - **Sticky Note1**  
    - Describes logic: create backup if new table, update if new data  
  - **Sticky Note4**  
    - Shows a GitHub repository backup visual representation

---

### 3. Summary Table

| Node Name                      | Node Type                | Functional Role                     | Input Node(s)                    | Output Node(s)                      | Sticky Note                                                                                           |
|--------------------------------|--------------------------|-----------------------------------|---------------------------------|-----------------------------------|-----------------------------------------------------------------------------------------------------|
| Daily Schedule                 | Schedule Trigger         | Triggers workflow daily            | None                            | List files from repository [GITHUB] |                                                                                                     |
| List files from repository [GITHUB] | GitHub                  | Lists existing backup files        | Daily Schedule                  | Combine file names [GITHUB]        | ## Get list of current tables Return a list of existing files in GitHub repository. Some of them are tables, Some are readme files |
| Combine file names [GITHUB]    | Item Lists              | Aggregates GitHub file names       | List files from repository [GITHUB] | List tables1                      |                                                                                                     |
| List tables1                  | Postgres                 | Lists Postgres tables in public schema | Combine file names [GITHUB]     | Loop Over Items                    | ## Get postgres table data Return a list of existing tables and data in the postgres. Convert them to csv |
| Loop Over Items               | SplitInBatches           | Batches tables for processing      | List tables1                   | Split to single items, Code        |                                                                                                     |
| Split to single items          | SplitInBatches           | Processes backups one by one       | Loop Over Items, Update file [GITHUB], Upload file [GITHUB] | Check if file exists in repository | ## Make a list of existing files Create backup if its a new table. Update backup if there is new data in the table |
| Check if file exists in repository | If                      | Decides to update or upload file   | Split to single items           | Update file [GITHUB], Upload file [GITHUB] |                                                                                                     |
| Update file [GITHUB]           | GitHub                   | Updates existing backup file       | Check if file exists in repository | Split to single items             |                                                                                                     |
| Upload file [GITHUB]           | GitHub                   | Uploads new backup file            | Check if file exists in repository | Split to single items             |                                                                                                     |
| List tables                   | Postgres                 | Extracts full table data           | Code                          | Convert to File1                  |                                                                                                     |
| Code                          | Code                     | Passes extracted data              | Loop Over Items                | List tables                      |                                                                                                     |
| Convert to File1              | ConvertToFile            | Converts table data to CSV file    | List tables                   | Loop Over Items                  |                                                                                                     |
| Sticky Note2                  | Sticky Note              | Explains GitHub files listing      | None                          | None                           | ## Get list of current tables Return a list of existing files in GitHub repository. Some of them are tables, Some are readme files |
| Sticky Note                   | Sticky Note              | Explains Postgres tables extraction| None                          | None                           | ## Get postgres table data Return a list of existing tables and data in the postgres. Convert them to csv |
| Sticky Note1                  | Sticky Note              | Explains backup file existence logic| None                          | None                           | ## Make a list of existing files Create backup if its a new table. Update backup if there is new data in the table |
| Sticky Note3                  | Sticky Note              | Visual diagram of postgres table   | None                          | None                           | ![postgres table](https://articles.emp0.com/wp-content/uploads/2025/08/backup-postgres-to-github-tables.png) |
| Sticky Note4                  | Sticky Note              | Visual diagram of GitHub backup    | None                          | None                           | ![github backup](https://articles.emp0.com/wp-content/uploads/2025/08/backup-postgres-to-github-repo.png) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**  
   - Type: Schedule Trigger  
   - Set interval to 24 hours (hoursInterval = 24)  
   - Name it `Daily Schedule`

2. **Create GitHub "List files" node:**  
   - Type: GitHub  
   - Operation: List files (resource: file)  
   - Owner: your GitHub username (`user`)  
   - Repository: your backup repository name (`github-repo`)  
   - File Path: empty string (root)  
   - Authentication: OAuth2 with configured GitHub account credentials  
   - Name it `List files from repository [GITHUB]`  
   - Connect output from `Daily Schedule` to this node

3. **Create Item Lists node to combine filenames:**  
   - Type: Item Lists  
   - Operation: aggregate items  
   - Field to aggregate: `name` (filename)  
   - Name it `Combine file names [GITHUB]`  
   - Connect output from `List files from repository [GITHUB]`

4. **Create PostgreSQL "List tables" node:**  
   - Type: Postgres  
   - Operation: Select from `information_schema.tables`  
   - Schema: `information_schema`  
   - Table: `tables`  
   - Where clause: `table_schema = 'public'`  
   - Credentials: your PostgreSQL credentials  
   - Name it `List tables1`  
   - Connect output from `Combine file names [GITHUB]`

5. **Create SplitInBatches node:**  
   - Type: SplitInBatches  
   - Batch size: default (or as desired)  
   - Name it `Loop Over Items`  
   - Connect output from `List tables1`

6. **Create Postgres node to fetch table data:**  
   - Type: Postgres  
   - Operation: Select all rows  
   - Table: dynamic expression: `{{$json.table_name}}` from current batch item  
   - Schema: `public`  
   - Credentials: same PostgreSQL credentials  
   - Name it `List tables`  
   - Connect output from `Loop Over Items`

7. **Create Code node:**  
   - Type: Code (JavaScript)  
   - JS code: `return $input.all();`  
   - Name it `Code`  
   - Connect output from `Loop Over Items` (secondary output)

8. **Create ConvertToFile node:**  
   - Type: ConvertToFile  
   - Options: fileName set dynamically to `{{$node["List tables1"].item.json.table_name}}`  
   - Binary Property Name: `data`  
   - Name it `Convert to File1`  
   - Connect output from `List tables`

9. **Connect ConvertToFile output to another SplitInBatches node:**  
   - Type: SplitInBatches  
   - Batch size: 1  
   - Name it `Split to single items`  
   - Connect output from `Convert to File1`

10. **Create If node to check file existence:**  
    - Type: If  
    - Condition: string contains  
    - Value1: `{{$node["Combine file names [GITHUB]"].json.name}}` (GitHub filenames list)  
    - Value2: `{{$binary.data.fileName}}` (current CSV filename)  
    - Name it `Check if file exists in repository`  
    - Connect output from `Split to single items`

11. **Create GitHub node to update file:**  
    - Type: GitHub  
    - Operation: Edit file  
    - Owner: your GitHub username  
    - Repository: your backup repo  
    - File path: dynamic `{{$binary.data.fileName}}`  
    - Commit message: `backup-{{$now.toMillis()}}`  
    - Use OAuth2 credentials  
    - Name it `Update file [GITHUB]`  
    - Connect True output from `Check if file exists in repository`

12. **Create GitHub node to upload new file:**  
    - Type: GitHub  
    - Operation: Create file  
    - Same parameters as update node  
    - Name it `Upload file [GITHUB]`  
    - Connect False output from `Check if file exists in repository`

13. **Connect outputs of both `Update file [GITHUB]` and `Upload file [GITHUB]` back to `Split to single items` for processing next batch.**

14. **Add Sticky Notes as needed:**  
    - For each block, add descriptive sticky notes with the content from the workflow JSON for clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Workflow includes visual guides (images) for Postgres table extraction and GitHub backups.   | ![Postgres Table](https://articles.emp0.com/wp-content/uploads/2025/08/backup-postgres-to-github-tables.png)  |
|                                                                                               | ![GitHub Backup](https://articles.emp0.com/wp-content/uploads/2025/08/backup-postgres-to-github-repo.png)       |
| The workflow uses OAuth2 credentials for GitHub API access and standard PostgreSQL credentials. | Credentials must be configured and authorized before running.                                       |
| Workflow is designed for the `public` schema in PostgreSQL; modifications required for others.| Modify schema filter in `List tables1` accordingly.                                                |

---

**Disclaimer:** The provided content is extracted exclusively from an n8n automated workflow. It complies with content policies and does not contain illegal, offensive, or protected elements. All processed data are legal and public.