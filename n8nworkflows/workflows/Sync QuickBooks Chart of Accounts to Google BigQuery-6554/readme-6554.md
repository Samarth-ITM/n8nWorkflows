Sync QuickBooks Chart of Accounts to Google BigQuery

https://n8nworkflows.xyz/workflows/sync-quickbooks-chart-of-accounts-to-google-bigquery-6554


# Sync QuickBooks Chart of Accounts to Google BigQuery

### 1. Workflow Overview

This workflow automates the weekly synchronization of the QuickBooks Chart of Accounts into a Google BigQuery table, preserving a historical and structured archive for financial analysis and reporting. It is designed primarily for data analysts, financial analysts, accountants, and business owners who want to maintain an up-to-date and historical record of their Chart of Accounts for robust financial modeling and auditing.

The workflow is logically divided into four main blocks:

- **1.1 Schedule Trigger:** Initiates the workflow automatically every Monday.
- **1.2 Data Extraction:** Queries QuickBooks Online for accounts created or updated in the past 7 days.
- **1.3 Data Transformation:** Processes and structures the raw API data, including generating stable UUIDs and managing currency balances.
- **1.4 Data Loading:** Formats the processed data into an SQL-compatible insert statement and loads the data into BigQuery.

---

### 2. Block-by-Block Analysis

#### 1.1 Schedule Trigger

**Overview:**  
This block triggers the entire workflow on a weekly basis, specifically every Monday, ensuring regular synchronization of the latest account changes.

**Nodes Involved:**  
- Start: Weekly on Monday

**Node Details:**

- **Node Name:** Start: Weekly on Monday  
- **Type & Role:** Schedule Trigger — initiates workflow execution based on a calendar schedule.  
- **Configuration:**  
  - Interval set to weekly.  
  - Trigger set to day 1 of the week (Monday).  
- **Inputs:** None (trigger node).  
- **Outputs:** Connects to `1. Get Updated Accounts from QuickBooks`.  
- **Version-specific Requirements:** None.  
- **Potential Failures:** Misconfiguration of schedule leading to missed triggers or unintended run frequency.  
- **Sub-workflow:** None.

---

#### 1.2 Data Extraction

**Overview:**  
This block fetches updated accounts from QuickBooks that were created or updated in the last 7 days, using the QuickBooks Online API with OAuth2 authentication.

**Nodes Involved:**  
- 1. Get Updated Accounts from QuickBooks  
- Sticky Note (provides instructions on replacing the company ID)

**Node Details:**

- **Node Name:** 1. Get Updated Accounts from QuickBooks  
- **Type & Role:** HTTP Request — performs a REST API query to QuickBooks.  
- **Configuration:**  
  - URL: `https://quickbooks.api.intuit.com/v3/company/{COMPANY_ID}/query` (requires manual replacement of `{COMPANY_ID}` with user's QuickBooks company ID).  
  - Query Parameter: SQL-like query to select accounts with `MetaData.LastUpdatedTime` within the last 7 days, using an expression to dynamically calculate the date.  
  - Headers: `Content-Type: application/json`.  
  - Authentication: Uses predefined QuickBooks OAuth2 credentials configured in n8n.  
- **Key Expressions:**  
  - Query uses expression: `select * from Account Where MetaData.LastUpdatedTime > '{{ $now.minus(7,'days') }}'` to fetch recent updates.  
- **Inputs:** Triggered by schedule node.  
- **Outputs:** Passes response data to `2. Structure Account Data`.  
- **Version-specific Requirements:** HTTP Request node version 4.2 or higher recommended for latest features.  
- **Potential Failures:**  
  - Authentication errors (expired or invalid OAuth2 tokens).  
  - API rate limits or connectivity issues.  
  - Failure to replace `{COMPANY_ID}` leading to API errors.  
  - Query syntax errors.  
- **Sticky Note:** Instructs user to replace `{COMPANY_ID}` manually and highlights the critical nature of this action.

---

#### 1.3 Data Transformation

**Overview:**  
This block transforms the raw JSON response from QuickBooks into a structured format suitable for database insertion. It generates stable UUIDs for each account record, manages currency balances for USD and CAD, and extracts relevant metadata.

**Nodes Involved:**  
- 2. Structure Account Data

**Node Details:**

- **Node Name:** 2. Structure Account Data  
- **Type & Role:** Code Node (JavaScript) — processes input JSON to create cleaned, structured output records.  
- **Configuration:**  
  - Custom JavaScript code reads API response, iterates over accounts, and:  
    - Generates a stable UUID based on a seed combining `account.Id` and `account.Name` with randomness to avoid collisions.  
    - Extracts `date_created_at` from the top-level `time` field if available.  
    - Sets balances for USD and CAD based on the account’s currency reference.  
    - Extracts multiple fields such as `id`, `account_id`, `name`, `active`, `classification`, `account_type`, `currency_origin`, `domain`, and timestamps.  
  - Handles missing or null fields gracefully.  
- **Key Variables/Expressions:** Uses `$input.all()` to access all input items.  
- **Inputs:** Receives HTTP response from QuickBooks node.  
- **Outputs:** Emits an array of JSON objects representing cleaned account records to `3. Format Data for SQL`.  
- **Version-specific Requirements:** Code node version 2 or higher supports modern JavaScript syntax.  
- **Potential Failures:**  
  - Unexpected API response structure leading to runtime errors.  
  - Null or undefined properties causing exceptions if not handled.  
  - UUID generation may produce collisions in extremely rare cases due to randomness.  
- **Sub-workflow:** None.

---

#### 1.4 Data Loading

**Overview:**  
This block formats the transformed account data into an SQL INSERT statement and executes it against a BigQuery table to update the stored chart of accounts.

**Nodes Involved:**  
- 3. Format Data for SQL  
- 4. Load Accounts to BigQuery  
- Sticky Note (provides instructions on BigQuery project selection and table verification)

**Node Details:**

- **Node Name:** 3. Format Data for SQL  
- **Type & Role:** Code Node (JavaScript) — converts structured JSON records into a string formatted as SQL values for insertion.  
- **Configuration:**  
  - Contains utility functions to sanitize and format values for SQL insertion:  
    - Removes smart quotes, non-ASCII characters, apostrophes, and escapes backslashes for strings.  
    - Converts booleans to SQL TRUE/FALSE.  
    - Converts dates to `DATE 'YYYY-MM-DD'` format or NULL.  
    - Ensures numeric fields are passed as-is or NULL if missing.  
  - Constructs a multi-row VALUES string for SQL insertion, concatenating all records with proper commas and line breaks.  
- **Inputs:** Receives structured account JSON from `2. Structure Account Data`.  
- **Outputs:** Passes a single JSON object containing the SQL values string to the BigQuery node.  
- **Potential Failures:**  
  - Malformed data causing incorrect SQL syntax.  
  - Missing or unexpected field values leading to improper formatting.  
- **Node Name:** 4. Load Accounts to BigQuery  
- **Type & Role:** Google BigQuery Node — executes the SQL INSERT query to load data into BigQuery.  
- **Configuration:**  
  - SQL Query: An INSERT INTO statement targeting the `quickbooks.accounts` table, using the formatted values string injected via expression.  
  - Project ID: Selected from a list, e.g. `n8n-self-host-461314`.  
  - Credentials: Uses OAuth2 credentials for Google BigQuery configured in n8n.  
- **Inputs:** Receives SQL values string from the formatting code node.  
- **Outputs:** Provides execution result data (success or error).  
- **Version-specific Requirements:** BigQuery node version 2.1 or higher recommended for SQL query support.  
- **Potential Failures:**  
  - Authentication errors or expired OAuth2 tokens.  
  - SQL syntax errors causing query failure.  
  - Table or dataset name mismatches.  
  - Permissions issues on BigQuery project or dataset.  
- **Sticky Note:** Reminds user to verify GCP project selection and SQL table name before activation.

---

### 3. Summary Table

| Node Name                           | Node Type                | Functional Role                     | Input Node(s)                    | Output Node(s)                  | Sticky Note                                                                                       |
|-----------------------------------|--------------------------|-----------------------------------|---------------------------------|--------------------------------|-------------------------------------------------------------------------------------------------|
| Sticky Note                       | Sticky Note              | Documentation / Instructions      | None                            | None                           | Contains full workflow overview and setup instructions.                                         |
| Start: Weekly on Monday            | Schedule Trigger         | Triggers workflow weekly on Monday| None                            | 1. Get Updated Accounts from QuickBooks |                                                                                                 |
| 1. Get Updated Accounts from QuickBooks | HTTP Request            | Fetches updated accounts from QuickBooks API| Start: Weekly on Monday          | 2. Structure Account Data         | Fetches accounts from QuickBooks updated in the last 7 days. ACTION: Replace {COMPANY_ID} in URL. |
| 2. Structure Account Data          | Code                     | Transforms raw API data into structured records with UUIDs| 1. Get Updated Accounts from QuickBooks| 3. Format Data for SQL           |                                                                                                 |
| 3. Format Data for SQL             | Code                     | Formats data into SQL insert values string| 2. Structure Account Data         | 4. Load Accounts to BigQuery      |                                                                                                 |
| 4. Load Accounts to BigQuery       | Google BigQuery          | Inserts formatted data into BigQuery| 3. Format Data for SQL           | None                           | Inserts the new account data into your accounts table. ACTION: Select your GCP Project and verify table name in SQL. |
| Sticky Note1                      | Sticky Note              | Instructional note for QuickBooks API node | None                            | None                           | Fetches accounts from QuickBooks updated in the last 7 days. ACTION: Replace companyId in URL.  |
| Sticky Note2                      | Sticky Note              | Instructional note for BigQuery node | None                            | None                           | Inserts the new account data into your accounts table. ACTION: Select your GCP Project and verify the table name in the SQL query. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node:**  
   - Node Type: Schedule Trigger  
   - Configure to run weekly on Monday (day 1).  
   - Name it `Start: Weekly on Monday`.

2. **Create HTTP Request Node:**  
   - Node Type: HTTP Request  
   - Name it `1. Get Updated Accounts from QuickBooks`.  
   - Set URL to `https://quickbooks.api.intuit.com/v3/company/{COMPANY_ID}/query`.  
   - Replace `{COMPANY_ID}` with your actual QuickBooks Company ID.  
   - Add Query Parameter:  
     - Name: `query`  
     - Value: `select * from Account Where MetaData.LastUpdatedTime > '{{ $now.minus(7,'days') }}'` (using expression syntax).  
   - Add Header: `Content-Type: application/json`.  
   - Authentication: Select QuickBooks OAuth2 credentials (must be set up prior).  
   - Connect output of `Start: Weekly on Monday` to this node.

3. **Create Code Node for Structuring Data:**  
   - Node Type: Code (JavaScript)  
   - Name it `2. Structure Account Data`.  
   - Paste the provided JavaScript code that:  
     - Generates stable UUIDs from `account.Id` and `account.Name`.  
     - Extracts and maps fields such as name, classification, balances, and timestamps.  
     - Handles currency differentiation (USD, CAD).  
   - Connect output of HTTP Request node to this node.

4. **Create Code Node for Formatting SQL:**  
   - Node Type: Code (JavaScript)  
   - Name it `3. Format Data for SQL`.  
   - Paste the provided JavaScript code that:  
     - Formats each field according to SQL syntax.  
     - Escapes problematic characters and normalizes strings.  
     - Constructs a multi-row VALUES string for the SQL INSERT.  
   - Connect output of `2. Structure Account Data` to this node.

5. **Create Google BigQuery Node:**  
   - Node Type: Google BigQuery  
   - Name it `4. Load Accounts to BigQuery`.  
   - Set credentials to Google BigQuery OAuth2 (must be configured with access to your GCP project).  
   - Set Project ID to your GCP project (e.g., `n8n-self-host-461314`).  
   - SQL Query:  
     ```
     INSERT INTO `quickbooks.accounts`
     (
       id,
       account_id,
       date_created_at,
       name,
       active,
       classification,
       account_type,
       current_balance_usd,
       current_balance_cad,
       currency_origin,
       domain,
       account_create_time,
       account_last_update_time,
       description
     )
     VALUES
     {{ $json.valuesString }};
     ```  
   - Connect output of `3. Format Data for SQL` to this node.

6. **Add Sticky Notes (Optional but Recommended):**  
   - Add a sticky note with the workflow overview and setup instructions.  
   - Add sticky notes near nodes `1. Get Updated Accounts from QuickBooks` and `4. Load Accounts to BigQuery` with respective instructions to replace company ID and verify BigQuery project/table names.

7. **Activate Workflow:**  
   - Save and activate the workflow. It will now run automatically every Monday.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                              | Context or Link                                                                                                                                                                |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| The workflow maintains a historical archive of your QuickBooks Chart of Accounts in BigQuery, enabling time-based financial analysis and reporting.                                                                                                                      | General workflow purpose                                                                                                                                                       |
| To find your QuickBooks Company ID, press `Ctrl + Alt + ?` within QuickBooks Online.                                                                                                                                                                                      | QuickBooks Company ID discovery tip                                                                                                                                            |
| Make sure your BigQuery table schema matches the columns used in the SQL INSERT statement, including the `description` field which is hardcoded as `"description"` in this workflow and can be modified as needed.                                                         | BigQuery schema alignment note                                                                                                                                                |
| You can change the sync frequency by editing the schedule trigger node. To perform an initial full backfill, temporarily modify the QuickBooks API query to `select * from Account`.                                                                                     | Customization options                                                                                                                                                          |
| QuickBooks API uses OAuth2; ensure your token is valid and refreshed to avoid authentication errors. Similarly, Google BigQuery credentials must have sufficient permissions to insert into the target dataset/table.                                                      | Authentication and authorization note                                                                                                                                         |
| The UUID generation in the code node combines account ID and name with randomness to produce a stable but unique identifier; this avoids collisions but may be customized or replaced with a standard UUID generator if preferred.                                         | UUID generation detail                                                                                                                                                         |
| See official n8n documentation for detailed credential setup for QuickBooks Online and Google BigQuery.                                                                                                                                                                  | Credential setup guidance                                                                                                                                                       |

---

**Disclaimer:** The provided content originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.