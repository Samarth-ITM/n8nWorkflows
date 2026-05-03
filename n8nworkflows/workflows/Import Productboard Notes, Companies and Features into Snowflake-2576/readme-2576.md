Import Productboard Notes, Companies and Features into Snowflake

https://n8nworkflows.xyz/workflows/import-productboard-notes--companies-and-features-into-snowflake-2576


# Import Productboard Notes, Companies and Features into Snowflake

---

### 1. Workflow Overview

This workflow automates the extraction and import of Productboard data (features, companies, and notes) into Snowflake data warehouse tables. It is designed for recurring weekly execution to keep Snowflake tables synchronized with Productboard’s latest data, including data cleansing by truncating tables before loading fresh data. It also summarizes new and unprocessed notes and sends a Slack notification with key insights and a link to a dashboard.

The workflow is logically divided into the following blocks:

- **1.1 Schedule Trigger & Setup:** Initiates the workflow weekly and manages Snowflake table creation and truncation.
- **1.2 Fetch & Process Productboard Features:** Retrieves features, splits them, maps fields for Snowflake, and updates the features table.
- **1.3 Fetch & Process Productboard Companies:** Retrieves companies, splits them, maps fields, and updates the companies table.
- **1.4 Fetch & Process Productboard Notes:** Retrieves notes, splits them, maps fields, updates the notes table, and processes note-feature relationships.
- **1.5 Process Note-Feature Relationships:** Extracts and maps feature IDs related to notes, updates the note-feature linking table.
- **1.6 Summary & Notification:** Counts recent and unprocessed notes, then sends a Slack message summarizing the weekly update.

---

### 2. Block-by-Block Analysis

#### 1.1 Schedule Trigger & Setup

- **Overview:** This block initiates the workflow every Monday at 8 AM weekly and ensures Snowflake tables exist with the correct schema. It truncates existing tables to prepare for fresh import.
- **Nodes Involved:** Schedule Trigger, [CREATE] PRODUCTBOARD_FEATURES, [CREATE] PRODUCTBOARD_COMPANIES, [CREATE] PRODUCTBOARD_NOTES, [CREATE] PRODUCTBOARD_NOTES_FEATURES, Empty Table Productboard Features, Empty Table Productboard Companies, Empty Table Productboard Notes, Empty Table Productboard Note and Feature IDs
- **Node Details:**

  - **Schedule Trigger**
    - Type: Schedule Trigger
    - Config: Trigger weekly, every Monday at 08:00
    - Inputs: None
    - Outputs: To Empty Table Productboard Features
    - Failures: None expected; ensure correct timezone if needed.
  
  - **[CREATE] PRODUCTBOARD_FEATURES**
    - Type: Snowflake (execute query)
    - Config: Creates or replaces PRODUCTBOARD_FEATURES table with specified columns.
    - Inputs: Trigger from Schedule Trigger -> Empty Table Productboard Features -> get productboard features (block 1.2)
    - Failures: SQL syntax errors if schema mismatches Snowflake setup.
  
  - **[CREATE] PRODUCTBOARD_COMPANIES**
    - Type: Snowflake (execute query)
    - Config: Creates or replaces PRODUCTBOARD_COMPANIES table.
    - Inputs: Triggered in parallel with features and notes creation.
  
  - **[CREATE] PRODUCTBOARD_NOTES**
    - Type: Snowflake (execute query)
    - Config: Creates or replaces PRODUCTBOARD_NOTES table with detailed note info columns.
  
  - **[CREATE] PRODUCTBOARD_NOTES_FEATURES**
    - Type: Snowflake (execute query)
    - Config: Creates or replaces PRODUCTBOARD_NOTES_FEATURES linking table.
  
  - **Empty Table Productboard Features / Companies / Notes / Note and Feature IDs**
    - Type: Snowflake (execute query)
    - Config: Executes TRUNCATE TABLE commands to clear data before insert.
    - Executes once per workflow run.
    - Failure cases: Permission errors, table not existing.
  
- **Edge Cases:**  
  - Tables missing or permission denied in Snowflake.
  - Schedule misconfiguration causing missed runs.
  - SQL execution failures.

---

#### 1.2 Fetch & Process Productboard Features

- **Overview:** Retrieves all Productboard features via API with pagination, splits the data into individual features, maps each feature's fields to a normalized format, batches them, and inserts into Snowflake.
- **Nodes Involved:** get productboard features, Split features, Manual mapping feature, Loop Over Items features, Empty Table Productboard Companies (triggered here for parallel flow), Manual mapping features db, Update Productboard Features
- **Node Details:**

  - **get productboard features**
    - Type: HTTP Request
    - Config: GET `https://api.productboard.com/features`, paginated using `links.next` URL with 3s intervals.
    - Auth: Productboard HTTP header auth.
    - Output: JSON array of features under `data`.
    - Failure: API rate limits, auth failure, network issues.
  
  - **Split features**
    - Type: Split Out
    - Config: Splits `data` array into individual feature objects.
  
  - **Manual mapping feature**
    - Type: Set
    - Config: Extracts and renames fields from raw feature data to match Snowflake columns (feature_id, feature_name, feature_status, feature_start_date, feature_end_date, feature_owner, feature_created_at).
    - Uses expressions like `={{ $json.id }}`.
  
  - **Loop Over Items features**
    - Type: Split In Batches
    - Config: Batches of 100 features for insertion.
  
  - **Manual mapping features db**
    - Type: Set
    - Config: Maps fields to uppercase column names for Snowflake (FEATURE_ID, FEATURE_NAME, etc.).
  
  - **Update Productboard Features**
    - Type: Snowflake
    - Config: Inserts or updates PRODUCTBOARD_FEATURES table with batched feature data.
    - Failure: DB insert errors, connectivity.
  
- **Edge Cases:**  
  - API pagination failures.
  - Missing or null fields in feature data.
  - Batch size too large causing DB timeouts.

---

#### 1.3 Fetch & Process Productboard Companies

- **Overview:** Similar to features, this block retrieves all companies, splits and maps company data, and updates Snowflake.
- **Nodes Involved:** get productboard companies, Split companies, Manual mapping companies, Manual mapping companies db, Update Productboard Companies, Empty Table Productboard Companies
- **Node Details:**

  - **get productboard companies**
    - Type: HTTP Request
    - Config: GET `https://api.productboard.com/companies` with pagination via `links.next`.
    - Auth: Productboard HTTP header.
  
  - **Split companies**
    - Type: Split Out
    - Splits company list into individual entries.
  
  - **Manual mapping companies**
    - Type: Set
    - Maps id, name, domain fields to company_id, company_name, company_domain.
  
  - **Manual mapping companies db**
    - Type: Set
    - Renames fields to uppercase for DB insert.
  
  - **Update Productboard Companies**
    - Type: Snowflake
    - Inserts/updates PRODUCTBOARD_COMPANIES table.
  
  - **Empty Table Productboard Companies**
    - Truncates table before loading.
  
- **Edge Cases:**  
  - Pagination or API limit issues.
  - Missing company domain or name fields.
  - DB insertion errors.

---

#### 1.4 Fetch & Process Productboard Notes

- **Overview:** Retrieves notes data with pagination, splits notes, maps note fields, batches inserts into Snowflake, and initiates note-feature relationship processing.
- **Nodes Involved:** get productboard notes, Split notes, Manual mapping notes, Loop Over Items notes, Manual mapping notes db, Update Productboard Notes, Empty Table Productboard Notes
- **Node Details:**

  - **get productboard notes**
    - Type: HTTP Request
    - Config: GET `https://api.productboard.com/notes` with pagination via `pageCursor`.
    - Auth: Productboard HTTP header.
    - Note: URL has a leading space that should be cleaned.
    - Failure: API errors, pagination issues.
  
  - **Split notes**
    - Type: Split Out
    - Splits notes array under `data`.
  
  - **Manual mapping notes**
    - Type: Set
    - Extracts fields like note_id, note_title, note_state, note_company_id, note_source, note_content, note_created_at, note_created_by, note_owner, note_url.
  
  - **Loop Over Items notes**
    - Type: Split In Batches
    - Batch size 100 notes.
  
  - **Manual mapping notes db**
    - Type: Set
    - Uppercase field names for DB.
  
  - **Update Productboard Notes**
    - Type: Snowflake
    - Inserts/updates PRODUCTBOARD_NOTES table.
  
  - **Empty Table Productboard Notes**
    - Truncates PRODUCTBOARD_NOTES before insert.
  
- **Edge Cases:**  
  - Notes without company or features.
  - Large content fields causing DB issues.
  - Pagination cursor errors.

---

#### 1.5 Process Note-Feature Relationships

- **Overview:** Extracts features linked to notes, creates combined records of note and feature IDs, batches and inserts into a linking Snowflake table.
- **Nodes Involved:** Split features in notes, Combine Feature ID + Note ID, Loop Over Items features notes, Manual mapping feature note IDs db, Update Productboard Note and Feature IDs, Empty Table Productboard Note and Feature IDs
- **Node Details:**

  - **Split features in notes**
    - Type: Split Out
    - Splits the `features` array within each note.
    - Includes `id` field from feature.
  
  - **Combine Feature ID + Note ID**
    - Type: Set
    - Creates objects with note_id and feature_id by extracting from split items.
  
  - **Loop Over Items features notes**
    - Type: Split In Batches
    - Batch size 100.
  
  - **Manual mapping feature note IDs db**
    - Type: Set
    - Uppercase fields for DB insert.
  
  - **Update Productboard Note and Feature IDs**
    - Type: Snowflake
    - Inserts/updates PRODUCTBOARD_NOTES_FEATURES linking table.
  
  - **Empty Table Productboard Note and Feature IDs**
    - Truncates linking table before load.
  
- **Edge Cases:**  
  - Notes without features (may produce empty arrays).
  - Feature IDs missing or null.
  - Batch insert failures.

---

#### 1.6 Summary & Notification

- **Overview:** Queries Snowflake to count new notes (last 7 days) and unprocessed notes, then sends a formatted Slack message with these stats and a dashboard link.
- **Nodes Involved:** Count Notes Last 7 days and Unprocessed, Slack, Sticky Note1
- **Node Details:**

  - **Count Notes Last 7 days and Unprocessed**
    - Type: Snowflake (query)
    - Query counts distinct notes created in last 7 days and notes with state 'unprocessed'.
    - Outputs JSON with `notes_7_days` and `notes_unprocessed`.
    - Failure: DB connection or query errors.
  
  - **Slack**
    - Type: Slack node (message)
    - Sends a Slack block message to channel `#product-notifications`.
    - Message uses variables from previous node for counts.
    - Includes a button linking to Metabase dashboard URL (replace placeholder).
    - Credentials: Slack API bot.
    - Executes once.
    - Errors continue without breaking workflow.
  
  - **Sticky Note1**
    - Type: Sticky Note
    - Content previews the Slack message text.
  
- **Edge Cases:**  
  - Slack API failure or credential issues.
  - Missing or empty counts.
  - Dashboard URL placeholder not updated.

---

### 3. Summary Table

| Node Name                           | Node Type             | Functional Role                              | Input Node(s)                                | Output Node(s)                                  | Sticky Note                                  |
|-----------------------------------|-----------------------|----------------------------------------------|----------------------------------------------|------------------------------------------------|----------------------------------------------|
| Schedule Trigger                   | Schedule Trigger      | Initiates workflow weekly                     | None                                         | Empty Table Productboard Features               | Setup snowflake tables                       |
| Empty Table Productboard Features  | Snowflake             | Truncates features table                      | Schedule Trigger                             | get productboard features                        | Setup snowflake tables                       |
| get productboard features          | HTTP Request          | Fetches Productboard features                 | Empty Table Productboard Features             | Split features                                  |                                              |
| Split features                    | Split Out             | Splits features array                         | get productboard features                      | Manual mapping feature                           |                                              |
| Manual mapping feature             | Set                   | Maps feature fields for Snowflake             | Split features                               | Loop Over Items features                         |                                              |
| Loop Over Items features           | Split In Batches      | Batches features for DB insert                | Manual mapping feature                        | Empty Table Productboard Companies (branch 1), Manual mapping features db (main)| Setup snowflake tables                       |
| Empty Table Productboard Companies | Snowflake             | Truncates companies table                     | Loop Over Items features                      | get productboard companies                       | Setup snowflake tables                       |
| get productboard companies         | HTTP Request          | Fetches Productboard companies                | Empty Table Productboard Companies             | Split companies                                 |                                              |
| Split companies                  | Split Out             | Splits companies array                        | get productboard companies                     | Manual mapping companies                         |                                              |
| Manual mapping companies           | Set                   | Maps company fields for Snowflake             | Split companies                              | Manual mapping companies db                      |                                              |
| Manual mapping companies db        | Set                   | Maps company fields uppercase for DB          | Manual mapping companies                      | Update Productboard Companies                    |                                              |
| Update Productboard Companies      | Snowflake             | Inserts/updates companies table               | Manual mapping companies db                   | Empty Table Productboard Note and Feature IDs   | Setup snowflake tables                       |
| Empty Table Productboard Note and Feature IDs | Snowflake   | Truncates note-feature linking table          | Update Productboard Companies                 | Empty Table Productboard Notes                   | Setup snowflake tables                       |
| Empty Table Productboard Notes     | Snowflake             | Truncates notes table                         | Empty Table Productboard Note and Feature IDs | get productboard notes                          | Setup snowflake tables                       |
| get productboard notes             | HTTP Request          | Fetches Productboard notes                     | Empty Table Productboard Notes                 | Split notes                                     |                                              |
| Split notes                      | Split Out             | Splits notes array                            | get productboard notes                         | Manual mapping notes, Split features in notes, Count Notes Last 7 days and Unprocessed |                                              |
| Manual mapping notes              | Set                   | Maps note fields for Snowflake                 | Split notes                                   | Loop Over Items notes                            |                                              |
| Loop Over Items notes             | Split In Batches      | Batches notes for DB insert                    | Manual mapping notes                           | Manual mapping notes db, (also loops back to Update Productboard Notes) |                                              |
| Manual mapping notes db           | Set                   | Maps note fields uppercase for DB              | Loop Over Items notes                          | Update Productboard Notes                        |                                              |
| Update Productboard Notes         | Snowflake             | Inserts/updates notes table                    | Manual mapping notes db                        | Loop Over Items notes (loop for next batch)     |                                              |
| Split features in notes           | Split Out             | Splits features array inside notes             | Split notes                                   | Combine Feature ID + Note ID                     |                                              |
| Combine Feature ID + Note ID      | Set                   | Combines note and feature IDs for linking      | Split features in notes                        | Loop Over Items features notes                   |                                              |
| Loop Over Items features notes    | Split In Batches      | Batches note-feature relationships             | Combine Feature ID + Note ID                   | Manual mapping feature note IDs db               |                                              |
| Manual mapping feature note IDs db| Set                   | Maps note-feature IDs uppercase for DB         | Loop Over Items features notes                 | Update Productboard Note and Feature IDs         |                                              |
| Update Productboard Note and Feature IDs | Snowflake       | Inserts/updates note-feature linking table     | Manual mapping feature note IDs db             | Loop Over Items features notes (loop)            |                                              |
| Empty Table Productboard Notes Features (alias) | Snowflake  | (Handled in Empty Table Productboard Note and Feature IDs) | -                                             | -                                              |                                              |
| Manual mapping features db        | Set                   | Maps feature fields uppercase for DB           | Loop Over Items features                       | Update Productboard Features                     |                                              |
| Update Productboard Features      | Snowflake             | Inserts/updates features table                  | Manual mapping features db                     | Loop Over Items features (loop)                  |                                              |
| Count Notes Last 7 days and Unprocessed | Snowflake         | Queries note counts for summary                 | Split notes                                   | Slack                                           | Preview Slack Message\n:productboard: Weekly Update in :snowflake_logo: Completed\n27 new insights added in the last 7 days.\n88 insights remain unprocessed.\nYou can view the updated :metabase: dashboard below:\n<link metabase> |
| Slack                            | Slack                 | Sends weekly notification message               | Count Notes Last 7 days and Unprocessed       | None                                            | Preview Slack Message (shared with Count node)|
| Sticky Note1                     | Sticky Note           | Preview Slack message content                    | None                                         | None                                            | Preview Slack Message\n:productboard: Weekly Update in :snowflake_logo: Completed\n27 new insights added in the last 7 days.\n88 insights remain unprocessed.\nYou can view the updated :metabase: dashboard below:\n<link metabase> |
| Sticky Note2                     | Sticky Note           | Setup snowflake tables reminder                   | None                                         | None                                            | Setup snowflake tables                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**
   - Type: Schedule Trigger
   - Set to trigger weekly on Mondays at 08:00.

2. **Create Snowflake Table Creation Nodes**
   - For each table (PRODUCTBOARD_FEATURES, PRODUCTBOARD_COMPANIES, PRODUCTBOARD_NOTES, PRODUCTBOARD_NOTES_FEATURES):
     - Use Snowflake node with "executeQuery" operation.
     - Provide appropriate CREATE OR REPLACE TABLE SQL schema (see node details in 2.1).
   - Connect Schedule Trigger node output to these creation nodes (can run in parallel or sequence).

3. **Create Snowflake Table Truncation Nodes**
   - For each table, create a Snowflake node executing `TRUNCATE TABLE <table_name>;`
   - Set these nodes to execute once per workflow run.
   - Chain these after the respective table creation nodes.

4. **Create HTTP Request Nodes to Fetch Data from Productboard**
   - Features: GET `https://api.productboard.com/features`
     - Set HTTP header auth credential with Productboard API token.
     - Enable pagination using `links.next` URL from response.
   - Companies: GET `https://api.productboard.com/companies`
     - Same auth, pagination config.
   - Notes: GET `https://api.productboard.com/notes`
     - Pagination using `pageCursor` parameter.

5. **Create Split Out Nodes**
   - For each data type, split the response array `data` into individual items.

6. **Create Set Nodes for Manual Mapping (Productboard fields to workflow fields)**
   - Features: Map id, name, status.name, timeframe.startDate, timeframe.endDate, owner.email, createdAt to feature fields.
   - Companies: Map id, name, domain.
   - Notes: Map id, title, state, company.id, source.origin, content, createdAt, createdBy.name, owner.name, displayUrl.

7. **Create Split In Batches Nodes**
   - For features, companies, notes, and note-feature relationships.
   - Set batch size to 100 for manageable DB inserts.

8. **Create Set Nodes for DB Mapping**
   - For each batch, map fields to uppercase column names matching Snowflake schema.

9. **Create Snowflake Update Nodes**
   - Insert or update data into respective tables.
   - Configure columns to match table schema.
   - Use Snowflake credentials configured in n8n.

10. **Process Note-Feature Relationships**
    - After splitting notes, add a Split Out node on the `features` array within each note.
    - Set node to combine note_id and feature_id.
    - Batch and map fields to uppercase for DB.
    - Insert into PRODUCTBOARD_NOTES_FEATURES table.

11. **Create Snowflake Query Node for Summary**
    - Query counts of notes created in last 7 days and notes with state 'unprocessed'.

12. **Create Slack Node**
    - Configure Slack API credentials.
    - Set message type to "block".
    - Use variables from previous Snowflake query for counts.
    - Provide Slack channel name or ID.
    - Include button linking to dashboard URL.
    - Set to execute once.

13. **Add Sticky Notes**
    - For Slack message preview.
    - For Snowflake setup reminder.

14. **Connect all nodes according to dependencies as described in the workflow connections.**

15. **Credential Setup**
    - Configure Productboard API credentials with HTTP Header Auth using API token.
    - Configure Snowflake credentials with connection details.
    - Configure Slack API credentials with bot token and required permissions.

16. **Activate the workflow**
    - Test run manually or wait for schedule.
    - Monitor for errors and adjust batch sizes or timeouts as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                 |
|---------------------------------------------------------------------------------------------------------------|--------------------------------|
| Replace Slack channel ID and dashboard URL in Slack notification node before activation.                       | Slack node configuration       |
| Productboard API requires HTTP header authentication with a valid API token.                                   | Productboard API docs          |
| The workflow truncates tables before loading fresh data, ensure no concurrent edits to data in Snowflake.      | Snowflake data consistency     |
| Slack message uses block kit formatting with button linking to Metabase dashboard (replace placeholder URL).   | Slack Block Kit guide          |
| Batch size of 100 is set for DB inserts to balance performance and avoid timeouts. Adjust if needed.           | Performance tuning             |
| Pagination uses "next" URL or pageCursor per Productboard API pagination specification.                         | Productboard API pagination    |
| Workflow inactive by default; activation requires credential setup and configuration adjustments.              | n8n workflow management        |

---

This documentation provides a detailed and structured reference to understand, reproduce, or modify the Productboard-to-Snowflake data import workflow in n8n, with considerations for error handling and integration nuances.