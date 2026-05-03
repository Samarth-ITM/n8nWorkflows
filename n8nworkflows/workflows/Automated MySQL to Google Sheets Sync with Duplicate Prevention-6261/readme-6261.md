Automated MySQL to Google Sheets Sync with Duplicate Prevention

https://n8nworkflows.xyz/workflows/automated-mysql-to-google-sheets-sync-with-duplicate-prevention-6261


# Automated MySQL to Google Sheets Sync with Duplicate Prevention

### 1. Workflow Overview

This workflow automates the synchronization of new rows from a MySQL database table (`fifa25_customers`) to a Google Sheet, ensuring no duplicate data is appended during repeated runs. It periodically checks for unsynced records, appends them to the Google Sheet, and marks them as synced in the database to prevent reprocessing. The workflow is structured into logical blocks as follows:

- **1.1 Scheduled Trigger:** Periodically initiates the sync process every 15 minutes.
- **1.2 Data Retrieval:** Queries the MySQL database for new records that have not been synced (`sync = 0`).
- **1.3 Conditional Check:** Determines if any new records were retrieved.
- **1.4 Data Append to Google Sheets:** Appends new records to a specified Google Sheet.
- **1.5 Database Update:** Marks the synced records in the MySQL database (`sync = 1`) to avoid duplication.
- **1.6 No Operation (Fallback):** If no new records are found, the workflow performs a no-op to gracefully end.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger

- **Overview:**  
  This block triggers the workflow at fixed 15-minute intervals to initiate the sync process automatically.

- **Nodes Involved:**  
  - Schedule Trigger Every n Mins

- **Node Details:**

  - **Schedule Trigger Every n Mins**  
    - **Type:** Schedule Trigger  
    - **Configuration:** Set to trigger every 15 minutes (minutesInterval: 15).  
    - **Key Expressions:** None.  
    - **Input:** None (trigger node).  
    - **Output:** Triggers "Select rows from a table" node.  
    - **Version Requirements:** Compatible with n8n v0.148+ (Schedule Trigger v1.2).  
    - **Potential Failures:** None typical; if workflow disabled or credentials invalid, trigger won’t fire.  
    - **Sub-workflow:** None.

#### 2.2 Data Retrieval

- **Overview:**  
  Queries the MySQL database to select all rows from the `fifa25_customers` table where the `sync` column is 0 (unsynced records).

- **Nodes Involved:**  
  - Select rows from a table  
  - Sticky Note (comment related to this step)

- **Node Details:**

  - **Select rows from a table**  
    - **Type:** MySQL  
    - **Configuration:**  
      - Operation: SELECT  
      - Table: `fifa25_customers`  
      - WHERE condition: `sync = 0` (only unsynced rows)  
      - No limit specified, but sticky note suggests "Get 50 records" (note: actual limit not enforced in node).  
    - **Key Expressions:** `=0` for the `sync` filter column.  
    - **Input:** Triggered by Schedule Trigger.  
    - **Output:** Sends retrieved rows to "Check if new record returned" node.  
    - **Version Requirements:** MySQL node version 2.4.  
    - **Potential Failures:**  
      - Database connection/authentication errors.  
      - SQL syntax errors if table or column names change.  
      - Large data sets without limit may cause performance issues.  
    - **Sub-workflow:** None.

  - **Sticky Note**  
    - Content: "Get 50 records from mysql table from not synced data"  
    - Context: Advises limiting records fetched to avoid processing large batches, though not implemented in the node.  
    - Position: Near the "Select rows from a table" node.

#### 2.3 Conditional Check

- **Overview:**  
  Checks whether the database query returned any new records. If no records are found, the workflow ends gracefully.

- **Nodes Involved:**  
  - Check if new record returned

- **Node Details:**

  - **Check if new record returned**  
    - **Type:** If  
    - **Configuration:**  
      - Condition: Number of returned rows > 0  
      - Uses JMESPath expression: `length($input.all().json)` to count records.  
      - Condition logic: true branch if records exist, false branch if none.  
    - **Input:** Output from "Select rows from a table".  
    - **Output:**  
      - True: continues to append rows to Google Sheet and update DB.  
      - False: triggers "No Operation, do nothing" node.  
    - **Version Requirements:** If node v2.2 with strict type validation.  
    - **Potential Failures:**  
      - Expression evaluation errors if input format changes.  
      - Empty input handling.  
    - **Sub-workflow:** None.

#### 2.4 Data Append to Google Sheets

- **Overview:**  
  Appends the new MySQL records to a specific Google Sheet document and worksheet, mapping database fields to sheet columns.

- **Nodes Involved:**  
  - Append row in sheet  
  - Sticky Note1 (related comment)

- **Node Details:**

  - **Append row in sheet**  
    - **Type:** Google Sheets  
    - **Configuration:**  
      - Operation: Append  
      - Document ID: `1b86B_7Hcusp7ehDNjJZtCa8Rlmljf2av7Hs-gaSluoc`  
      - Sheet Name: `gid=0` (first sheet/tab in document)  
      - Columns mapping:  
        - id, name, email, phone, gender, region, datatime, birthdate  
        - Values taken dynamically from incoming JSON using expressions like `={{ $json.id }}` etc.  
      - Matching columns set to `id` (used for matching if needed, though operation is append).  
      - Data conversion disabled.  
    - **Input:** From true branch of "Check if new record returned".  
    - **Output:** Connected to "Update rows in a table" node.  
    - **Credentials:** Google Sheets OAuth2 account authorized for this document.  
    - **Version Requirements:** Google Sheets node v4.6.  
    - **Potential Failures:**  
      - Authentication/authorization errors with Google API.  
      - API rate limits or quota exceeded.  
      - Mismatched column names or schema changes.  
      - Network or transient errors.  
    - **Sub-workflow:** None.

  - **Sticky Note1**  
    - Content: "Update all the got data with column sync = 1 to prevent duplication in next run"  
    - Context: Explains purpose of next block that updates MySQL records.  
    - Position: Near the "Append row in sheet" node.

#### 2.5 Database Update

- **Overview:**  
  After successful append, updates the synced records in MySQL by setting their `sync` column to 1, preventing them from being reprocessed.

- **Nodes Involved:**  
  - Update rows in a table

- **Node Details:**

  - **Update rows in a table**  
    - **Type:** MySQL  
    - **Configuration:**  
      - Operation: UPDATE  
      - Table: `fifa25_customers`  
      - Values to send: `sync = 1`  
      - Matching column: `id` (matches each record by its database ID)  
      - Matches value: taken dynamically from each input item’s `id` field.  
    - **Input:** From "Append row in sheet" output.  
    - **Output:** None (terminal for this branch).  
    - **Version Requirements:** MySQL node v2.4.  
    - **Potential Failures:**  
      - DB connection errors.  
      - Record lock or concurrency issues.  
      - Updates failing if matching `id` is missing or incorrect.  
    - **Sub-workflow:** None.

#### 2.6 No Operation (Fallback)

- **Overview:**  
  If no new records are found in the MySQL query, this node acts as a graceful no-op to end the workflow cleanly.

- **Nodes Involved:**  
  - No Operation, do nothing

- **Node Details:**

  - **No Operation, do nothing**  
    - **Type:** NoOp  
    - **Configuration:** No parameters needed.  
    - **Input:** False branch output of "Check if new record returned".  
    - **Output:** Terminal node.  
    - **Version Requirements:** NoOp node v1.  
    - **Potential Failures:** None.  
    - **Sub-workflow:** None.

#### 2.7 Additional Sticky Note (Output)

- **Sticky Note2**  
  - Content: "## Output"  
  - Position: Near the bottom right, likely a placeholder or marker with no active role.

---

### 3. Summary Table

| Node Name                 | Node Type            | Functional Role                                | Input Node(s)                   | Output Node(s)                          | Sticky Note                                                                                   |
|---------------------------|----------------------|-----------------------------------------------|--------------------------------|----------------------------------------|-----------------------------------------------------------------------------------------------|
| Schedule Trigger Every n Mins | Schedule Trigger     | Periodic initiation every 15 minutes          | None                           | Select rows from a table                |                                                                                               |
| Select rows from a table   | MySQL                | Select unsynced rows (sync=0) from MySQL      | Schedule Trigger Every n Mins  | Check if new record returned            | Get 50 records from mysql table from not synced data                                          |
| Check if new record returned | If                   | Check if any records retrieved                 | Select rows from a table       | Append row in sheet, Update rows in a table, No Operation, do nothing |                                                                                               |
| Append row in sheet        | Google Sheets        | Append new records to Google Sheet             | Check if new record returned   | Update rows in a table                  | Update all the got data with column sync = 1 to prevent duplication in next run                |
| Update rows in a table     | MySQL                | Update synced records in DB (sync=1)           | Append row in sheet            | None                                   |                                                                                               |
| No Operation, do nothing   | NoOp                 | End workflow gracefully if no new records      | Check if new record returned   | None                                   |                                                                                               |
| Sticky Note               | Sticky Note          | Comment on data fetch limit                     | None                           | None                                   | Get 50 records from mysql table from not synced data                                          |
| Sticky Note1              | Sticky Note          | Comment on update query rationale               | None                           | None                                   | Update all the got data with column sync = 1 to prevent duplication in next run                |
| Sticky Note2              | Sticky Note          | Output placeholder                              | None                           | None                                   | ## Output                                                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node:**  
   - Type: Schedule Trigger  
   - Set to trigger every 15 minutes (interval: minutes, value: 15).

2. **Create MySQL Node to Select Rows:**  
   - Type: MySQL  
   - Operation: Select  
   - Table: `fifa25_customers`  
   - Condition: WHERE `sync = 0`  
   - Credentials: Provide MySQL credentials with read access.  
   - Connect output of Schedule Trigger to this node.

3. **Create If Node to Check for New Records:**  
   - Type: If  
   - Condition: Number of input rows > 0  
   - Use expression: `length($input.all().json)` > 0  
   - Connect output of MySQL Select node to this node.

4. **Create Google Sheets Node to Append Rows:**  
   - Type: Google Sheets  
   - Operation: Append  
   - Document ID: `1b86B_7Hcusp7ehDNjJZtCa8Rlmljf2av7Hs-gaSluoc`  
   - Sheet Name: `gid=0`  
   - Columns mapping: Map each column (`id`, `name`, `email`, `phone`, `gender`, `region`, `datatime`, `birthdate`) to corresponding JSON properties using expressions like `={{ $json.id }}`.  
   - Credentials: Configure Google Sheets OAuth2 credentials with access to the document.  
   - Connect "true" output of the If node to this node.

5. **Create MySQL Node to Update Records:**  
   - Type: MySQL  
   - Operation: Update  
   - Table: `fifa25_customers`  
   - Set field `sync` = 1  
   - Match on column `id` using the value from each item (`={{ $json.id }}`).  
   - Credentials: Same MySQL credentials as step 2.  
   - Connect output of Google Sheets Append node to this node.

6. **Create No Operation Node:**  
   - Type: NoOp  
   - No parameters needed.  
   - Connect "false" output of the If node to this node.

7. **(Optional) Add Sticky Notes:**  
   - Add comments near relevant nodes explaining the logic, such as limiting the number of records fetched, and preventing duplicates by updating the sync flag.

8. **Activate Workflow:**  
   - Ensure all credentials are valid and tested.  
   - Activate the workflow to run on schedule.

---

### 5. General Notes & Resources

| Note Content                                                                                             | Context or Link                                                       |
|---------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------|
| To avoid processing large data batches, consider adding a SQL `LIMIT 50` clause to the MySQL SELECT node. | Sticky Note near "Select rows from a table" node.                    |
| Update the `sync` flag to 1 after successful Google Sheet append to prevent duplicates on next runs.     | Sticky Note near "Append row in sheet" and "Update rows in a table". |
| Google Sheets API quotas and rate limits may affect performance; monitor usage accordingly.              | Best practice for Google Sheets integrations.                        |
| Ensure MySQL and Google Sheets credentials are securely stored and refreshed as needed.                  | Credential management best practices in n8n.                         |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.