Consolidate Data from 5 Sources for Automated Reporting with SQL, MongoDB & Google Tools

https://n8nworkflows.xyz/workflows/consolidate-data-from-5-sources-for-automated-reporting-with-sql--mongodb---google-tools-8890


# Consolidate Data from 5 Sources for Automated Reporting with SQL, MongoDB & Google Tools

### 1. Workflow Overview

This workflow, titled **"Integrated Data Consolidation"**, is designed to automate the aggregation and harmonization of data from five distinct sources: Google Sheets, PostgreSQL, MongoDB, Microsoft SQL Server, and Google Analytics. Its primary purpose is to consolidate disparate datasets into a unified master Google Sheet, facilitating comprehensive reporting and improved data visibility.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Scheduled trigger initiates data retrieval from the five sources.
- **1.2 Source Data Enrichment:** Each source dataset is tagged with a unique identifier to maintain traceability post-merging.
- **1.3 Data Merging:** All enriched datasets are merged into a single dataset.
- **1.4 Data Processing:** The merged dataset is cleaned, schema-aligned, and standardized.
- **1.5 Output Writing:** The processed, consolidated data is appended or updated into a designated Google Sheet for reporting.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block triggers the workflow on a scheduled basis (every Monday, Wednesday, and Friday) and fetches raw data from all five configured sources.

- **Nodes Involved:**  
  - Schedule Trigger  
  - 📄 Google Sheets Source  
  - 🐘 PostgreSQL Source  
  - 🍃 MongoDB Source  
  - Microsoft SQL Server  
  - Google Analytics

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Initiates the workflow execution on specified days of the week (Monday, Wednesday, Friday).  
    - Configuration: Interval configured to weekly trigger on days 1, 3, and 5.  
    - Inputs: None (trigger node).  
    - Outputs: Triggers five parallel source nodes.  
    - Edge Cases: Missed triggers if n8n is down; ensure timezone settings match expected schedule.

  - **📄 Google Sheets Source**  
    - Type: Google Sheets  
    - Role: Reads data from a specified Google Sheet document and sheet (gid=0).  
    - Configuration: Document ID and Sheet Name parameterized with credentials; reads all rows by default.  
    - Inputs: Trigger from Schedule Trigger.  
    - Outputs: Raw data to "Add Sheets Source ID".  
    - Edge Cases: API quota limits, permissions/authentication errors, empty sheets.

  - **🐘 PostgreSQL Source**  
    - Type: PostgreSQL  
    - Role: Executes a SELECT query to retrieve all records from the "customers" table in the "public" schema.  
    - Configuration: Operation is "select", returns all rows.  
    - Inputs: Trigger from Schedule Trigger.  
    - Outputs: Raw data to "Add PostgreSQL Source ID".  
    - Edge Cases: Connection timeouts, query errors, authentication failures.

  - **🍃 MongoDB Source**  
    - Type: MongoDB  
    - Role: Fetches up to 1000 documents from specified MongoDB collection.  
    - Configuration: Collection name provided; limit set to 1000 documents.  
    - Inputs: Trigger from Schedule Trigger.  
    - Outputs: Raw data to "Add MongoDB Source ID".  
    - Edge Cases: Connection errors, query timeouts, exceeding document limits.

  - **Microsoft SQL Server**  
    - Type: Microsoft SQL  
    - Role: Executes a SQL query (`SELECT * FROM your_table;`) to retrieve data from a SQL Server database.  
    - Configuration: Query text provided; operation set to executeQuery.  
    - Inputs: Trigger from Schedule Trigger.  
    - Outputs: Raw data to "Add SQL Server Source ID".  
    - Edge Cases: SQL syntax errors, connection/authentication issues.

  - **Google Analytics**  
    - Type: Google Analytics  
    - Role: Queries user activity metrics for a specified User ID and View ID.  
    - Configuration: User ID and GA View ID must be set; resource set to "userActivity".  
    - Inputs: Trigger from Schedule Trigger.  
    - Outputs: Raw data to "Add Analytics Source ID".  
    - Edge Cases: API quota, authentication errors, invalid IDs.

---

#### 1.2 Source Data Enrichment

- **Overview:**  
  Each source dataset is enriched by appending a unique identifier field. This tag enables traceability and prevents data overlap or confusion during the merge step.

- **Nodes Involved:**  
  - Add Sheets Source ID  
  - Add PostgreSQL Source ID  
  - Add MongoDB Source ID  
  - Add SQL Server Source ID  
  - Add Analytics Source ID

- **Node Details:**

  Each node is a **Function** node that:

  - Type: Function  
  - Role: Adds a new field (e.g., `sourceId`) with a static string indicating the data source (e.g., "Google Sheets", "PostgreSQL").  
  - Configuration: JavaScript code loops through each incoming item and appends `sourceId` property.  
  - Inputs: Raw data from respective source node.  
  - Outputs: Enriched data to the Merge node at a specified input index (0 to 4).  
  - Edge Cases: Empty input data results in no items to tag; ensure empty arrays are handled gracefully.

---

#### 1.3 Data Merging

- **Overview:**  
  This block merges the five enriched datasets into a single combined dataset.

- **Nodes Involved:**  
  - Merge

- **Node Details:**

  - **Merge**  
    - Type: Merge  
    - Role: Combines the five input datasets into one unified output.  
    - Configuration: Number of inputs set to 5; default merge mode (append).  
    - Inputs: Receives enriched datasets from all five "Add Source ID" nodes.  
    - Outputs: Combined dataset forwarded to processing.  
    - Edge Cases: Unequal dataset sizes; empty inputs; node expects all inputs connected.

---

#### 1.4 Data Processing

- **Overview:**  
  Processes the merged dataset by cleaning, aligning schemas, and standardizing fields such as Name, Email, Title, Company, Phone, LinkedIn, Notes, Function, Seniority, Confidence Score, and Status.

- **Nodes Involved:**  
  - ⚙️ Process Merged Data

- **Node Details:**

  - **⚙️ Process Merged Data**  
    - Type: Function  
    - Role: Performs data transformation and normalization on the merged dataset.  
    - Configuration: Custom JavaScript code to clean data, enforce field consistency, possibly remove duplicates, and standardize formats.  
    - Inputs: Merged dataset from Merge node.  
    - Outputs: Cleaned and standardized dataset to Final Google Sheet node.  
    - Edge Cases: Malformed data entries; missing fields; inconsistent data types.

---

#### 1.5 Output Writing

- **Overview:**  
  Writes the processed, consolidated dataset into a designated master Google Sheet for reporting and visibility.

- **Nodes Involved:**  
  - 📊 Final Google Sheet

- **Node Details:**

  - **📊 Final Google Sheet**  
    - Type: Google Sheets  
    - Role: Appends or updates rows in the specified Google Sheet document and sheet.  
    - Configuration:  
      - Document ID and Sheet Name are parameterized and must be set.  
      - Operation: "appendOrUpdate" to ensure data is added or updated based on matching columns.  
      - Schema defined with fields: Name, Email, Title, Company, Phone, LinkedIn, Notes, Function, Seniority, Confidence Score, Status.  
    - Inputs: Processed dataset from the Process Merged Data node.  
    - Outputs: None (final output).  
    - Edge Cases: API rate limits; permission issues; data mismatch on update keys.

---

### 3. Summary Table

| Node Name               | Node Type           | Functional Role                          | Input Node(s)                       | Output Node(s)              | Sticky Note                                                                                   |
|-------------------------|---------------------|----------------------------------------|-----------------------------------|-----------------------------|----------------------------------------------------------------------------------------------|
| Schedule Trigger        | Schedule Trigger    | Initiate workflow on Mon, Wed, Fri     | None                              | 📄 Google Sheets Source, 🐘 PostgreSQL Source, 🍃 MongoDB Source, Microsoft SQL Server, Google Analytics | ## Objective: \n\n*Consolidate data from 5 sources (Google Sheets, PostgreSQL, MongoDB, MS SQL, Google Analytics) into a master Google Sheet for reporting and visibility.* |
| 📄 Google Sheets Source | Google Sheets       | Fetch data from Google Sheets           | Schedule Trigger                  | Add Sheets Source ID         | See Objective sticky note                                                                    |
| 🐘 PostgreSQL Source    | PostgreSQL          | Fetch customer records from PostgreSQL | Schedule Trigger                  | Add PostgreSQL Source ID     | See Objective sticky note                                                                    |
| 🍃 MongoDB Source       | MongoDB             | Fetch documents from MongoDB            | Schedule Trigger                  | Add MongoDB Source ID        | See Objective sticky note                                                                    |
| Microsoft SQL Server    | Microsoft SQL       | Fetch data from SQL Server              | Schedule Trigger                  | Add SQL Server Source ID     | See Objective sticky note                                                                    |
| Google Analytics        | Google Analytics    | Fetch user activity metrics             | Schedule Trigger                  | Add Analytics Source ID      | See Objective sticky note                                                                    |
| Add Sheets Source ID    | Function            | Tag Google Sheets data with source ID  | 📄 Google Sheets Source           | Merge                       | ## Merge Node: \n\n*Combines all datasets into a unified structure.*                        |
| Add PostgreSQL Source ID| Function            | Tag PostgreSQL data with source ID      | 🐘 PostgreSQL Source              | Merge                       | See Merge Node sticky note                                                                   |
| Add MongoDB Source ID   | Function            | Tag MongoDB data with source ID         | 🍃 MongoDB Source                | Merge                       | See Merge Node sticky note                                                                   |
| Add SQL Server Source ID| Function            | Tag SQL Server data with source ID      | Microsoft SQL Server             | Merge                       | See Merge Node sticky note                                                                   |
| Add Analytics Source ID | Function            | Tag Google Analytics data with source ID| Google Analytics                | Merge                       | See Merge Node sticky note                                                                   |
| Merge                   | Merge               | Combine all enriched datasets           | Add Sheets Source ID, Add PostgreSQL Source ID, Add MongoDB Source ID, Add SQL Server Source ID, Add Analytics Source ID | ⚙️ Process Merged Data         | See Merge Node sticky note                                                                   |
| ⚙️ Process Merged Data  | Function            | Clean, align, and standardize merged data | Merge                            | 📊 Final Google Sheet        | ## Processing Node: \n\n*Cleans, aligns schemas, and standardizes fields (Name, Email, Title, Company, etc.).* |
| 📊 Final Google Sheet   | Google Sheets       | Append/Update consolidated data to output Google Sheet | ⚙️ Process Merged Data            | None                        | ## Final dataset is written into Google Sheets.\n\n*Configure your output Google Sheets document ID and credentials to save the consolidated data.* |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named "Integrated Data Consolidation".

2. **Add a Schedule Trigger node**  
   - Set interval to weekly trigger on days Monday (1), Wednesday (3), and Friday (5).

3. **Add five data source nodes, connecting each to the Schedule Trigger:**

   - **Google Sheets Source:**  
     - Type: Google Sheets  
     - Configure credentials for Google Sheets API.  
     - Set Document ID to your source Google Sheets document.  
     - Set Sheet Name to `gid=0` or your specific sheet.  
     - Operation: Read all rows.

   - **PostgreSQL Source:**  
     - Type: PostgreSQL  
     - Configure PostgreSQL credentials (host, port, user, password, database).  
     - Operation: Select  
     - Schema: "public"  
     - Table: "customers"  
     - Return All: true

   - **MongoDB Source:**  
     - Type: MongoDB  
     - Configure MongoDB credentials (connection string).  
     - Collection: your target collection name.  
     - Limit: 1000 documents.

   - **Microsoft SQL Server Source:**  
     - Type: Microsoft SQL  
     - Configure MS SQL credentials.  
     - Operation: executeQuery  
     - Query: `SELECT * FROM your_table;`

   - **Google Analytics Source:**  
     - Type: Google Analytics  
     - Configure Google Analytics credentials.  
     - User ID: your User ID  
     - View ID: your GA View ID  
     - Resource: "userActivity"

4. **For each source node, add a Function node to append a source identifier:**

   - Create a Function node named accordingly (e.g., "Add Sheets Source ID").  
   - Connect the source node output to this Function node.  
   - Function code snippet example:

    ```javascript
    return items.map(item => {
      item.json.sourceId = "Google Sheets"; // customize per source
      return item;
    });
    ```

5. **Add a Merge node:**

   - Set Number of Inputs to 5.  
   - Connect each "Add Source ID" Function node to a different input of the Merge node.

6. **Add a Function node "⚙️ Process Merged Data":**

   - Connect Merge node output to this node.  
   - Implement JavaScript code to clean, align, and standardize data fields across all merged items, ensuring consistent schema and removing duplicates if necessary.

7. **Add a Google Sheets node "📊 Final Google Sheet":**

   - Configure with your output Google Sheets credentials.  
   - Set Document ID and Sheet Name for the master reporting sheet.  
   - Operation: "appendOrUpdate"  
   - Define columns schema matching the final data structure (Name, Email, Title, Company, Phone, LinkedIn, Notes, Function, Seniority, Confidence Score, Status).  
   - Connect output of "⚙️ Process Merged Data" to this node.

8. **Test the workflow:**

   - Trigger the Schedule Trigger node manually or wait for scheduled run.  
   - Verify data flows through all nodes without errors.  
   - Inspect the output Google Sheet for consolidated results.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                              | Context or Link                                                                                          |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| Objective: Consolidate data from 5 sources (Google Sheets, PostgreSQL, MongoDB, Microsoft SQL Server, Google Analytics) into a master Google Sheet for reporting and visibility.            | Sticky Note covering initial input and source nodes                                                    |
| Merge Node: Combines all datasets into a unified structure. Processing Node: Cleans, aligns schemas, and standardizes fields (Name, Email, Title, Company, etc.).                          | Sticky Note adjacent to Merge and Processing nodes                                                     |
| Final dataset is written into Google Sheets. Configure your output Google Sheets document ID and credentials to save the consolidated data.                                              | Sticky Note near Final Google Sheet node                                                               |
| Ensure all Google API credentials have appropriate scopes enabled (Google Sheets API, Google Analytics API) and OAuth tokens are refreshed as needed.                                    | Credential configuration best practice                                                                 |
| MongoDB query limit set to 1000 documents to avoid performance issues; adjust based on dataset size and API constraints.                                                                   | Performance consideration for MongoDB Source node                                                      |
| The PostgreSQL and Microsoft SQL Server source queries assume read access and appropriate privileges to specified tables/schemas.                                                        | Database access prerequisite                                                                             |
| Schedule Trigger uses weekly intervals with specific days; adjust to your reporting cadence if needed.                                                                                     | Scheduling customization advice                                                                          |

---

**Disclaimer:**  
The provided text originates solely from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.