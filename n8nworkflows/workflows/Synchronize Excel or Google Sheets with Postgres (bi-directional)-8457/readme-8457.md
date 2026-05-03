Synchronize Excel or Google Sheets with Postgres (bi-directional)

https://n8nworkflows.xyz/workflows/synchronize-excel-or-google-sheets-with-postgres--bi-directional--8457


# Synchronize Excel or Google Sheets with Postgres (bi-directional)

### 1. Workflow Overview

This workflow automates the bi-directional synchronization between an Excel (or Google Sheets) table and a Postgres database table. Its primary use case is to maintain data consistency between spreadsheet-based data sources and a structured Postgres database, supporting operations like manual sync trigger, scheduled synchronization, and execution from other workflows.

The workflow is logically divided into the following blocks:

- **1.1 Trigger Reception**: Enables initiation by manual trigger, scheduled trigger, or external workflow execution.
- **1.2 Data Retrieval**: Fetches rows from a specified Excel table.
- **1.3 Data Sanitization**: Normalizes and formats date fields to ensure database compatibility.
- **1.4 Database Upsert**: Inserts or updates rows in the Postgres database, avoiding duplicates by matching on a key column.
- **1.5 Documentation and Notes**: Sticky notes provide descriptive context for users.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger Reception

**Overview:**  
This block allows the workflow to start through three mechanisms: manual user initiation, scheduled periodic runs, or triggered execution from another workflow.

**Nodes Involved:**  
- Click Sync Excel -> DB (Manual Trigger)  
- Schedule Sync Excel -> DB (Schedule Trigger)  
- Exec Sync Excel -> DB (Execute Workflow Trigger)

**Node Details:**

- **Click Sync Excel -> DB**  
  - Type: Manual Trigger  
  - Role: Enables user-initiated workflow start.  
  - Configuration: No parameters; triggers immediately when clicked.  
  - Inputs: None  
  - Outputs: Connects to `Get Table` node.  
  - Edge Cases: None typical; user must manually trigger.

- **Schedule Sync Excel -> DB**  
  - Type: Schedule Trigger  
  - Role: Runs the workflow automatically on a recurring interval defined by the user.  
  - Configuration: Default interval set (implied to run periodically; exact timing not specified).  
  - Inputs: None  
  - Outputs: Connects to `Get Table` node.  
  - Edge Cases: Incorrect schedule setup may cause no runs or too frequent runs.

- **Exec Sync Excel -> DB**  
  - Type: Execute Workflow Trigger  
  - Role: Allows this workflow to be triggered by other workflows, facilitating modular automation.  
  - Configuration: Passes input data through transparently (`passthrough` mode).  
  - Inputs: External workflow input  
  - Outputs: Connects to `Get Table` node.  
  - Edge Cases: Requires correct upstream workflow setup; failure if input is malformed.

---

#### 2.2 Data Retrieval

**Overview:**  
Fetches data rows from a specified Excel table and worksheet, limiting to 200 rows per execution.

**Nodes Involved:**  
- Get Table

**Node Details:**

- **Get Table**  
  - Type: Microsoft Excel Node  
  - Role: Reads rows from an Excel table named "Clients" within the "CLIENTS" worksheet.  
  - Configuration:  
    - Operation: `getRows`  
    - Limit: 200 rows max  
    - Workbook and worksheet selected by name (cached references used).  
    - Table mode set to list with empty value (likely defaults to the "Clients" table).  
  - Credentials: Microsoft Excel OAuth2 required (credentials must be configured).  
  - Inputs: Trigger nodes (`Click Sync Excel -> DB`, `Schedule Sync Excel -> DB`, `Exec Sync Excel -> DB`)  
  - Outputs: Connects to `Sanitize Date` node.  
  - Edge Cases:  
    - Authentication or permission errors if credentials are invalid or expired.  
    - Table or worksheet not found errors if naming changes.  
    - Row limit may truncate large datasets; consider paging for large tables.

---

#### 2.3 Data Sanitization

**Overview:**  
Processes the fetched data to normalize and format the `date` field to a consistent `MM/DD/YYYY` string format, converting Excel serial date numbers or string dates as needed.

**Nodes Involved:**  
- Sanitize Date

**Node Details:**

- **Sanitize Date**  
  - Type: Code Node (JavaScript)  
  - Role: Converts `date` fields from Excel serial numbers or string formats into uniform `MM/DD/YYYY` strings.  
  - Configuration: Custom JavaScript code that:  
    - Checks if `date` is a number (Excel serial date), converts to JS Date.  
    - Else parses string dates.  
    - Outputs formatted date string or null if invalid.  
  - Key logic: Uses `Date.UTC(1899, 11, 30)` as Excel base date for serial conversion.  
  - Inputs: `Get Table` node  
  - Outputs: Connects to `Upsert Table` node.  
  - Edge Cases:  
    - Invalid or malformed date strings result in `null` date fields.  
    - Excel serial numbers must be numeric; non-numeric values remain unchanged.  
    - Timezone issues minimal since UTC is used for base date.  
  - Version: TypeVersion 2 (supports advanced JS code).

---

#### 2.4 Database Upsert

**Overview:**  
Upserts rows into the Postgres database table `clients`, matching rows based on the `no` column to avoid duplicates.

**Nodes Involved:**  
- Upsert Table

**Node Details:**

- **Upsert Table**  
  - Type: Postgres Node  
  - Role: Inserts new records or updates existing ones in the `clients` table in the `public` schema.  
  - Configuration:  
    - Operation: `upsert`  
    - Table: `clients`  
    - Schema: `public`  
    - Matching Columns: `no` (used to identify existing records)  
    - Column mapping: Auto-mapped input data fields to database columns based on matching names.  
    - No type conversion or string coercion enabled.  
  - Credentials: Postgres credentials required and must be valid.  
  - Inputs: `Sanitize Date` node  
  - Outputs: None (end of flow)  
  - Edge Cases:  
    - Authentication failures if credentials invalid.  
    - Schema or table changes may break mapping.  
    - Missing or null `no` values may cause unexpected inserts or errors.  
    - Upsert operation dependent on DB support for conflict detection on `no` column.

---

#### 2.5 Documentation and Notes

**Overview:**  
Provides users with descriptive notes explaining workflow purpose, blocks, use cases, and instructions.

**Nodes Involved:**  
- Sticky Note  
- Sticky Note2

**Node Details:**

- **Sticky Note**  
  - Type: Sticky Note  
  - Role: Describes the workflow’s overall purpose and stepwise logic for user clarity.  
  - Content highlights:  
    - Explains triggers, data fetch, sanitization, upsert, and potential for two-way sync.  
    - Use case targeting spreadsheet-heavy companies moving to SaaS or dashboards.  
  - Position: Top-left anchor of the workflow canvas.

- **Sticky Note2**  
  - Type: Sticky Note  
  - Role: Title note labeling the workflow as “Synchronize Excel Sheet with Postgres Database.”  
  - Positioned near triggers and start nodes.

---

### 3. Summary Table

| Node Name                | Node Type                  | Functional Role                      | Input Node(s)                   | Output Node(s)           | Sticky Note                                                                                                  |
|--------------------------|----------------------------|------------------------------------|--------------------------------|--------------------------|--------------------------------------------------------------------------------------------------------------|
| Sticky Note              | Sticky Note                | Documentation and overview         | None                           | None                     | Describes the workflow’s purpose, steps, and use case for Excel ↔ DB synchronization.                        |
| Sticky Note2             | Sticky Note                | Workflow title note                 | None                           | None                     | ## Synchronize Excel Sheet with Postgres Database                                                            |
| Click Sync Excel -> DB   | Manual Trigger             | Manual workflow start trigger      | None                           | Get Table                 |                                                                                                              |
| Schedule Sync Excel -> DB| Schedule Trigger           | Scheduled workflow start trigger   | None                           | Get Table                 |                                                                                                              |
| Exec Sync Excel -> DB    | Execute Workflow Trigger   | External workflow start trigger    | None                           | Get Table                 |                                                                                                              |
| Get Table                | Microsoft Excel Node       | Fetches rows from Excel table      | Click Sync Excel -> DB, Schedule Sync Excel -> DB, Exec Sync Excel -> DB | Sanitize Date            |                                                                                                              |
| Sanitize Date            | Code Node (JavaScript)     | Normalizes and formats date fields | Get Table                      | Upsert Table              |                                                                                                              |
| Upsert Table             | Postgres Node              | Upserts data rows into Postgres DB | Sanitize Date                  | None                     |                                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Create the Trigger Nodes:**

   - Add a **Manual Trigger** node, name it `Click Sync Excel -> DB`. No special parameters needed.
   - Add a **Schedule Trigger** node, name it `Schedule Sync Excel -> DB`. Set an interval as needed (e.g., every hour or daily).
   - Add an **Execute Workflow Trigger** node, name it `Exec Sync Excel -> DB`. Set parameter `Input Source` to `passthrough`.

3. **Add the Data Retrieval Node:**

   - Add a **Microsoft Excel** node, name it `Get Table`.
   - Set `Operation` to `getRows`.
   - Set `Limit` to 200.
   - Choose the `Workbook` and `Worksheet` corresponding to your Excel file and sheet (e.g., workbook containing "Clients", worksheet named "CLIENTS").
   - Select the `Table` named "Clients".
   - Configure Microsoft Excel OAuth2 credentials properly with access to the Excel file.

4. **Add Data Sanitization Node:**

   - Add a **Code** node, name it `Sanitize Date`.
   - Use the following JavaScript code:
     ```javascript
     return items.map(item => {
       const inputDate = item.json["date"];
       let formattedDate = null;
       let dateObj;

       if (typeof inputDate === "number") {
         const baseDate = new Date(Date.UTC(1899, 11, 30));
         baseDate.setDate(baseDate.getDate() + inputDate);
         dateObj = baseDate;
       } else if (typeof inputDate === "string") {
         const parsed = new Date(inputDate);
         if (!isNaN(parsed)) {
           dateObj = parsed;
         }
       }

       if (dateObj) {
         const M = String(dateObj.getMonth() + 1).padStart(2, '0');
         const D = String(dateObj.getDate()).padStart(2, '0');
         const Y = dateObj.getFullYear();
         formattedDate = `${M}/${D}/${Y}`;
       }

       item.json["date"] = formattedDate;
       return { json: item.json };
     });
     ```
   - Ensure node type version supports advanced JS (v2 or above).

5. **Add Database Upsert Node:**

   - Add a **Postgres** node, name it `Upsert Table`.
   - Set operation to `upsert`.
   - Select schema: `public`.
   - Select table: `clients`.
   - Configure columns for auto-mapping.
   - Set matching column to `no` to identify existing rows.
   - Configure Postgres credentials with appropriate access.

6. **Connect Nodes:**

   - Connect all three triggers (`Click Sync Excel -> DB`, `Schedule Sync Excel -> DB`, `Exec Sync Excel -> DB`) to `Get Table`.
   - Connect `Get Table` to `Sanitize Date`.
   - Connect `Sanitize Date` to `Upsert Table`.

7. **Add Sticky Notes (Optional for Documentation):**

   - Add a sticky note with the title “Synchronize Excel Sheet with Postgres Database” near the triggers.
   - Add a detailed sticky note describing the workflow steps and use cases near the top-left corner for user reference.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                  | Context or Link                                           |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------|
| The workflow is ideal for organizations transitioning from spreadsheet-heavy workflows to structured database-backed dashboards or SaaS solutions.                                                                           | Workflow purpose description                              |
| Ensure Excel column names exactly match Postgres column names for auto-mapping; otherwise, configure manual column mapping in the Postgres node.                                                                             | Workflow usage tip                                        |
| Date sanitization handles Excel serial date formats and string dates but will output `null` if input date is unrecognized.                                                                                                   | Data sanitization note                                   |
| Postgres upsert relies on the `no` column as a unique identifier; missing or duplicated `no` values can cause upsert issues.                                                                                                  | Database constraint advice                                |
| Microsoft Excel OAuth2 credentials must have appropriate permissions to read the target Excel workbook and worksheet.                                                                                                         | Credential setup requirement                             |
| See more about n8n's Postgres node and Microsoft Excel node capabilities for deeper customization: https://docs.n8n.io/integrations/builtin/n8n-nodes-base.postgres/ and https://docs.n8n.io/integrations/builtin/n8n-nodes-base-microsoft-excel/ | Official n8n documentation                               |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.