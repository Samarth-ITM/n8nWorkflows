Synchronize your Google Sheets with Postgres

https://n8nworkflows.xyz/workflows/synchronize-your-google-sheets-with-postgres-2081


# Synchronize your Google Sheets with Postgres

---

### 1. Workflow Overview

This workflow automates the synchronization of data from a Google Sheets spreadsheet into a PostgreSQL database table. It is designed primarily for one-way synchronization (Google Sheets → PostgreSQL) but can be adapted for two-way sync with minor modifications.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger:** Initiates the synchronization process automatically at regular intervals (hourly).
- **1.2 Data Retrieval:** Fetches the full dataset from both Google Sheets and PostgreSQL to compare.
- **1.3 Data Preparation:** Extracts and prepares only relevant fields from the Google Sheets data for comparison.
- **1.4 Dataset Comparison:** Compares the two datasets to identify new or updated entries based on matching fields.
- **1.5 Database Update:** Inserts new rows and updates existing rows in PostgreSQL to mirror the Google Sheets data.

This structure ensures minimal manual intervention, enabling regular and consistent data synchronization while allowing for customization of credentials, target sheet, and table.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger

- **Overview:**  
Triggers the workflow execution every hour to automate the synchronization process.

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Trigger node  
    - Role: Initiates workflow on a time-based schedule  
    - Configuration: Set to trigger every hour (`interval` field set to hours).  
    - Inputs: None (trigger node)  
    - Outputs: Connects to "Retrieve Sheets Data" and "Select Rows in Postgres" nodes  
    - Version: 1.1  
    - Possible Failures: Misconfigured schedule or node execution delays may cause missed runs.  
    - Notes: None  

#### 2.2 Data Retrieval

- **Overview:**  
Retrieves current data from Google Sheets and PostgreSQL to prepare for comparison.

- **Nodes Involved:**  
  - Retrieve Sheets Data  
  - Select Rows in Postgres

- **Node Details:**

  - **Retrieve Sheets Data**  
    - Type: Google Sheets node  
    - Role: Reads all data from a specified Google Sheets document and sheet  
    - Configuration:  
      - Document ID and Sheet Name set to specific Google Sheets document (cached for convenience).  
      - Reads the entire sheet (gid=0).  
    - Inputs: Receives trigger from Schedule Trigger  
    - Outputs: Passes data to "Split Out Relevant Fields"  
    - Version: 4.2  
    - Edge Cases:  
      - Authentication errors if Google Sheets credentials are missing or invalid.  
      - API rate limits or connectivity issues.  
      - Empty sheet or unexpected data format.  

  - **Select Rows in Postgres**  
    - Type: PostgreSQL node  
    - Role: Selects all rows from the target Postgres table to retrieve existing data for comparison  
    - Configuration:  
      - Schema: public  
      - Table: testing (preset cached result)  
      - Operation: select, return all rows  
    - Inputs: Trigger from Schedule Trigger (runs in parallel with Google Sheets data retrieval)  
    - Outputs: Passes data to "Compare Datasets" node (as input 2)  
    - Version: 2.3  
    - Edge Cases:  
      - Database connection issues or credential errors.  
      - Large datasets may cause timeouts or performance issues.  
      - Table or schema does not exist.  

#### 2.3 Data Preparation

- **Overview:**  
Extracts only the relevant fields from the Google Sheets data to focus comparison on essential columns.

- **Nodes Involved:**  
  - Split Out Relevant Fields

- **Node Details:**

  - **Split Out Relevant Fields**  
    - Type: Split Out node  
    - Role: Filters the data fields to only include "first_name", "last_name", "town", and "age" for comparison  
    - Configuration: Field to split out set to "first_name, last_name, town, age"  
    - Inputs: Receives Google Sheets data from "Retrieve Sheets Data"  
    - Outputs: Passes filtered data to "Compare Datasets" node (as input 1)  
    - Version: 1  
    - Edge Cases:  
      - Fields missing in the incoming data will result in empty or undefined values.  
      - Data type mismatches not handled explicitly here.  

#### 2.4 Dataset Comparison

- **Overview:**  
Compares the Google Sheets filtered data against the PostgreSQL data to find new or updated records, preferring Google Sheets data in conflicts.

- **Nodes Involved:**  
  - Compare Datasets

- **Node Details:**

  - **Compare Datasets**  
    - Type: Compare Datasets node  
    - Role: Compares two datasets, identifies differences, and outputs inserts and updates accordingly  
    - Configuration:  
      - Merge by fields: matches on "first_name" from both datasets (field1 and field2 both "first_name")  
      - Resolve conflicts by preferring input 1 (Google Sheets data)  
      - No additional options set  
    - Inputs:  
      - Input 1: Filtered Google Sheets data ("Split Out Relevant Fields")  
      - Input 2: PostgreSQL data ("Select Rows in Postgres")  
    - Outputs:  
      - Output 0 (main): rows to insert (new entries)  
      - Output 1: rows unchanged (ignored)  
      - Output 2: rows to update (existing entries with changes)  
    - Version: 2.3  
    - Edge Cases:  
      - Matching only on "first_name" is simplistic and may produce false positives/negatives if names are not unique.  
      - No handling of deletions or data removal from Sheets.  
      - Potential issues if data contains duplicates or null values in matching fields.  

#### 2.5 Database Update

- **Overview:**  
Inserts new rows and updates existing rows in the PostgreSQL table to reflect the current state of Google Sheets.

- **Nodes Involved:**  
  - Insert Rows  
  - Update Rows

- **Node Details:**

  - **Insert Rows**  
    - Type: PostgreSQL node  
    - Role: Inserts new rows identified by the comparison into the PostgreSQL table  
    - Configuration:  
      - Schema: public  
      - Table: testing  
      - Columns: Automatically mapped from input data (first_name, last_name, town, age)  
      - Operation: insert  
    - Inputs: Receives data from Compare Datasets output 0 (new rows)  
    - Outputs: None connected  
    - Version: 2.3  
    - Edge Cases:  
      - Insert failures if required fields are missing or data violates DB constraints.  
      - Duplicate key errors if data is inconsistent.  

  - **Update Rows**  
    - Type: PostgreSQL node  
    - Role: Updates existing rows that have changed according to the comparison  
    - Configuration:  
      - Schema: public  
      - Table: testing  
      - Columns explicitly defined with expressions to map incoming data fields: age, town, last_name, first_name  
      - Matching columns for update: first_name and last_name (composite key)  
      - Operation: update  
    - Inputs: Receives data from Compare Datasets output 2 (rows to update)  
    - Outputs: None connected  
    - Version: 2.3  
    - Edge Cases:  
      - Update conflicts if matching rows are not unique or missing.  
      - Partial updates if data fields are null or incorrect.  
      - Database transaction errors or locks.  

---

### 3. Summary Table

| Node Name             | Node Type            | Functional Role               | Input Node(s)           | Output Node(s)               | Sticky Note                                                                                                    |
|-----------------------|----------------------|------------------------------|------------------------|-----------------------------|---------------------------------------------------------------------------------------------------------------|
| Schedule Trigger       | Schedule Trigger     | Initiates workflow hourly     | None                   | Retrieve Sheets Data, Select Rows in Postgres |                                                                                                               |
| Retrieve Sheets Data   | Google Sheets        | Fetches all data from sheet   | Schedule Trigger       | Split Out Relevant Fields    | "## Retrieving Data & Spitting Out Fields \nGet the Data you want to compare and split out the relevant fields" |
| Select Rows in Postgres| PostgreSQL           | Fetches all rows from table   | Schedule Trigger       | Compare Datasets             |                                                                                                               |
| Split Out Relevant Fields | Split Out           | Filters relevant columns      | Retrieve Sheets Data    | Compare Datasets             | "## Retrieving Data & Spitting Out Fields \nGet the Data you want to compare and split out the relevant fields" |
| Compare Datasets       | Compare Datasets     | Compares Google Sheets with Postgres data | Split Out Relevant Fields, Select Rows in Postgres | Insert Rows (new), Update Rows (updated) |                                                                                                               |
| Insert Rows           | PostgreSQL           | Inserts new rows into database | Compare Datasets (new rows) | None                        | "## Updating Your Database \nUsing Insert Rows & Update Rows as Separate Postgres Node's"                      |
| Update Rows           | PostgreSQL           | Updates existing rows          | Compare Datasets (updated rows) | None                        | "## Updating Your Database \nUsing Insert Rows & Update Rows as Separate Postgres Node's"                      |
| Sticky Note           | Sticky Note          | Setup instructions             | None                   | None                        | "## Setup ##\nIn order to make this automation work for you, you need to make a few adjustments:\n\n1. Add your Postgres & Google Sheets Credentials to the respective Nodes\n\n2. Select the Sheet (Google Sheets) and the table (Postgres) you want to sync\n\n3. Update the Insert & Update Queries so that the data is updated in the table you also selected the rows from in the first step" |
| Sticky Note1          | Sticky Note          | Notes on DB update nodes       | None                   | None                        | "## Updating Your Database \nUsing Insert Rows & Update Rows as Separate Postgres Node's"                      |
| Sticky Note2          | Sticky Note          | Notes on retrieval and splitting| None                   | None                        | "## Retrieving Data & Spitting Out Fields \nGet the Data you want to compare and split out the relevant fields" |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**  
   - Type: Schedule Trigger  
   - Configure to run every 1 hour (set interval field to hours).  
   - No credentials needed.

2. **Create a Google Sheets node ("Retrieve Sheets Data"):**  
   - Type: Google Sheets  
   - Authenticate with Google Sheets credentials (OAuth2).  
   - Set Document ID to your Google Sheet's ID.  
   - Set Sheet Name to the target sheet (e.g., "Sheet1" or gid=0).  
   - Operation: Read all rows.  
   - Connect input from Schedule Trigger node.

3. **Create a PostgreSQL node ("Select Rows in Postgres"):**  
   - Type: PostgreSQL  
   - Authenticate with Postgres credentials.  
   - Set Schema to your schema (e.g., "public").  
   - Set Table to your target table (e.g., "testing").  
   - Operation: Select, return all rows.  
   - Connect input from Schedule Trigger node (parallel to Google Sheets node).

4. **Create a Split Out node ("Split Out Relevant Fields"):**  
   - Type: Split Out  
   - Configure to keep fields: "first_name, last_name, town, age".  
   - Connect input from "Retrieve Sheets Data".

5. **Create a Compare Datasets node ("Compare Datasets"):**  
   - Type: Compare Datasets  
   - Configure Merge by Fields: match on "first_name" (field1 and field2 both set to "first_name").  
   - Set resolve conflicts to "preferInput1" (Google Sheets data preferred).  
   - Connect input 1 from "Split Out Relevant Fields".  
   - Connect input 2 from "Select Rows in Postgres".

6. **Create a PostgreSQL node ("Insert Rows"):**  
   - Type: PostgreSQL  
   - Authenticate with Postgres credentials.  
   - Set Schema and Table same as "Select Rows in Postgres".  
   - Operation: Insert rows.  
   - Columns: Auto-map inputs for "first_name", "last_name", "town", "age".  
   - Connect input from Compare Datasets output 0 (rows to insert).

7. **Create a PostgreSQL node ("Update Rows"):**  
   - Type: PostgreSQL  
   - Authenticate with Postgres credentials.  
   - Set Schema and Table same as above.  
   - Operation: Update rows.  
   - Define columns with expressions mapping input data to columns:  
     - first_name = {{$json.first_name}}  
     - last_name = {{$json.last_name}}  
     - town = {{$json.town}}  
     - age = {{$json.age}}  
   - Set matching columns to "first_name" and "last_name" to identify rows to update.  
   - Connect input from Compare Datasets output 2 (rows to update).

8. **Add Sticky Notes (optional):**  
   - Add notes for setup instructions and reminders on configuring credentials, selecting sheets/tables, and understanding insert/update logic.

9. **Validate credentials:**  
   - Ensure Google Sheets OAuth2 credentials are valid and have access to the target spreadsheet.  
   - Ensure PostgreSQL credentials have read/write access to the target database and table.

10. **Test the workflow:**  
    - Run manually or wait for scheduled trigger.  
    - Check logs for errors or warnings.  
    - Verify data in PostgreSQL matches Google Sheets after sync.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                | Context or Link                                                                                 |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow currently performs one-way synchronization from Google Sheets to PostgreSQL. To enable two-way sync, modify the comparison and update logic. | General workflow usage note                                                                    |
| Before usage, add your PostgreSQL and Google Sheets credentials in the respective nodes to enable data access.                                               | Setup requirement                                                                              |
| Google Sheets API may have rate limits; for large datasets, consider batching or optimizing reads to avoid errors.                                           | Performance consideration                                                                      |
| Matching only on "first_name" can lead to incorrect data syncing if data is not uniquely identified by this field. Adjust matching fields as needed.          | Data integrity caution                                                                         |
| Postgres Insert and Update nodes handle new and changed data separately for clarity and control.                                                             | Design note from sticky note                                                                   |

---

This documentation fully describes the workflow logic, configuration, and steps to reproduce and maintain the synchronization process between Google Sheets and PostgreSQL.