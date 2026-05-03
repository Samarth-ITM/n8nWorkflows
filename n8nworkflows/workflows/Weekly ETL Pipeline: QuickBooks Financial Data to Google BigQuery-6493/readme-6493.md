Weekly ETL Pipeline: QuickBooks Financial Data to Google BigQuery

https://n8nworkflows.xyz/workflows/weekly-etl-pipeline--quickbooks-financial-data-to-google-bigquery-6493


# Weekly ETL Pipeline: QuickBooks Financial Data to Google BigQuery

### 1. Workflow Overview

This workflow implements a **weekly ETL (Extract, Transform, Load) pipeline** that synchronizes financial transaction data from **QuickBooks Online** into **Google BigQuery** for advanced reporting, analysis, and archiving. It automates fetching the previous week's transactions every Monday, cleans and classifies the data based on business logic, formats it for SQL insertion, and loads it into a structured BigQuery table.

**Target Use Cases:**  
- Data Analysts and BI developers needing structured, queryable financial data in a data warehouse.  
- Financial professionals requiring enriched transaction data for custom queries beyond QuickBooks native reports.  
- Business owners desiring historical transaction archives and reports in BigQuery.

**Logical Blocks:**  
- **1.1 Schedule Trigger:** Initiates the workflow weekly on Monday.  
- **1.2 Extraction:** Retrieves last week’s transactions from QuickBooks Online.  
- **1.3 Transformation:** Cleans, classifies, and enriches transaction data with custom JavaScript logic.  
- **1.4 Formatting:** Converts transaction objects into a bulk-insert-ready SQL statement string.  
- **1.5 Loading:** Executes an SQL INSERT query to upload the data into a BigQuery table.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Schedule Trigger  
- **Overview:** Triggers the entire ETL pipeline every Monday, ensuring timely weekly data sync.  
- **Nodes Involved:**  
  - Start: Weekly on Monday  
- **Node Details:**  
  - **Start: Weekly on Monday**  
    - Type: Schedule Trigger  
    - Configuration: Set to trigger on a weekly interval, specifically on Monday (day 1 of the week).  
    - Inputs: None (start node)  
    - Outputs: Connects to the QuickBooks extraction node.  
    - Edge Cases: Misconfiguration of schedule may cause no runs or multiple runs; confirm server timezone matches expected schedule.

#### Block 1.2: Extraction  
- **Overview:** Fetches all financial transactions from the previous week using QuickBooks Online API.  
- **Nodes Involved:**  
  - 1. Get Last Week's Transactions  
- **Node Details:**  
  - **1. Get Last Week's Transactions**  
    - Type: QuickBooks Node  
    - Configuration:  
      - Resource: Transaction  
      - Filter: Date macro set to "Last Week" to retrieve previous week’s data.  
      - Simple mode disabled for detailed response.  
    - Credentials: Uses OAuth2 credentials for QuickBooks Online.  
    - Inputs: Receives trigger from schedule node.  
    - Outputs: Sends raw transaction data to transformation node.  
    - Edge Cases: Authentication failure, API rate limits, no transactions returned, partial data due to API paging (pagination not shown, potential enhancement).  

#### Block 1.3: Transformation  
- **Overview:** Applies business logic to clean, classify, and enrich each transaction record, generating stable IDs and categorizing transactions as income, expense, or internal transfer.  
- **Nodes Involved:**  
  - 2. Clean & Classify Transactions  
- **Node Details:**  
  - **2. Clean & Classify Transactions**  
    - Type: Code (JavaScript) Node  
    - Configuration:  
      - Custom JS script processes each transaction row:  
        - Generates a UUID based on transaction details and a random seed for stability.  
        - Classifies transactions into expense, income, or internal transfer based on QuickBooks transaction types and user-defined account/category lists (`internalTransferAccounts`, `expenseCategories`, `incomeCategories`).  
        - Handles currency-specific amounts (USD, CAD).  
        - Cleans and extracts relevant transaction fields.  
      - Requires user to populate account/category arrays to match their QuickBooks Chart of Accounts.  
    - Inputs: Raw QuickBooks transactions.  
    - Outputs: Array of cleaned, classified transaction objects.  
    - Edge Cases:  
      - Missing or malformed transaction rows are skipped.  
      - If classification arrays are not updated, classification may be inaccurate.  
      - Parsing errors or missing fields handled gracefully by fallbacks.  
      - Potential random element in UUID generation could cause ID instability on reruns — intended to be mostly stable with some randomness.  
    - Sticky Note: Explicit instruction to customize classification arrays before production use.

#### Block 1.4: Formatting  
- **Overview:** Converts the cleaned transaction objects into a single string representing a bulk SQL VALUES clause, suitable for insertion into BigQuery.  
- **Nodes Involved:**  
  - 3. Format Data for SQL  
- **Node Details:**  
  - **3. Format Data for SQL**  
    - Type: Code (JavaScript) Node  
    - Configuration:  
      - Defines helper functions to safely format SQL values: handle null, booleans, numbers, strings with sanitization (removes smart quotes, non-ASCII, question marks, apostrophes, new lines, tabs, and backslashes).  
      - Converts dates into SQL DATE literals.  
      - Maps all transaction fields into formatted SQL tuples.  
      - Joins all tuples into a single string assigned to `valuesString`.  
    - Inputs: Cleaned and classified transaction objects.  
    - Outputs: JSON containing a single property with the SQL VALUES string.  
    - Edge Cases:  
      - Unexpected data types will be stringified safely.  
      - Potential data loss due to aggressive string sanitization (e.g., removal of question marks and apostrophes).  
      - Null or missing fields converted to SQL NULL.  
    - Sticky Note: Typically no user modification needed here.

#### Block 1.5: Loading  
- **Overview:** Executes a parameterized SQL INSERT query on Google BigQuery to load the transformed transaction data into a specified table.  
- **Nodes Involved:**  
  - 4. Load Data to BigQuery  
- **Node Details:**  
  - **4. Load Data to BigQuery**  
    - Type: Google BigQuery Node  
    - Configuration:  
      - SQL Query: Bulk INSERT into `quickbooks.transactions` table, mapping all fields.  
      - Uses the formatted `valuesString` from previous node.  
      - Project ID selected from Google Cloud projects list.  
    - Credentials: Uses OAuth2 credentials for Google BigQuery.  
    - Inputs: Receives SQL values string from formatting node.  
    - Outputs: No downstream nodes; final step.  
    - Edge Cases:  
      - SQL syntax errors if `valuesString` is malformed.  
      - BigQuery quota limits or permissions errors.  
      - Project or dataset/table misconfiguration causing query failure.  
      - Partial inserts on error are not handled; transactional control not shown.

---

### 3. Summary Table

| Node Name                   | Node Type               | Functional Role                            | Input Node(s)              | Output Node(s)             | Sticky Note                                                                                                             |
|-----------------------------|-------------------------|--------------------------------------------|----------------------------|----------------------------|-------------------------------------------------------------------------------------------------------------------------|
| Start: Weekly on Monday      | Schedule Trigger        | Initiates workflow weekly on Monday         | None                       | 1. Get Last Week's Transactions |                                                                                                                         |
| 1. Get Last Week's Transactions | QuickBooks             | Fetches last week’s transactions             | Start: Weekly on Monday     | 2. Clean & Classify Transactions |                                                                                                                         |
| 2. Clean & Classify Transactions | Code (JavaScript)      | Cleans and classifies transactions           | 1. Get Last Week's Transactions | 3. Format Data for SQL      | This is the core logic of the workflow. **ACTION: (MANDATORY)** You must edit the JavaScript in this node. Update the lists of account and category names (internalTransferAccounts, expenseCategories, etc.) to match your specific QuickBooks Chart of Accounts. |
| 3. Format Data for SQL       | Code (JavaScript)      | Formats transactions as a bulk SQL VALUES string | 2. Clean & Classify Transactions | 4. Load Data to BigQuery    | This node takes the cleaned data and transforms it into a single, safe string for a bulk SQL INSERT command. No changes are typically needed here. |
| 4. Load Data to BigQuery     | Google BigQuery        | Loads data into BigQuery table                | 3. Format Data for SQL      | None                       |                                                                                                                         |
| Sticky Note                 | Sticky Note             | Documentation and setup instructions          | None                       | None                       | ## Sync and Enrich QuickBooks Transactions to Google BigQuery This template sets up a weekly ETL (Extract, Transform, Load) pipeline that pulls financial data from QuickBooks Online into Google BigQuery. It not only transfers data, but also cleans, classifies, and enriches each transaction using your own business logic. <br> [Full setup and usage details included.] |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Schedule Trigger Node**  
   - Name: `Start: Weekly on Monday`  
   - Type: Schedule Trigger  
   - Parameters:  
     - Set interval to weekly.  
     - Set trigger day to Monday (day 1).  
   - No credentials required.  
   - Connect output to the QuickBooks node.

3. **Add a QuickBooks Node**  
   - Name: `1. Get Last Week's Transactions`  
   - Type: QuickBooks  
   - Parameters:  
     - Resource: Transaction  
     - Filter: Date macro = "Last Week"  
     - Simple mode: Disabled (false) to get full transaction details.  
   - Credentials: Connect your QuickBooks Online OAuth2 credentials.  
   - Connect output to the transformation code node.

4. **Add a Code Node for Transformation**  
   - Name: `2. Clean & Classify Transactions`  
   - Type: Code (JavaScript)  
   - Parameters: Paste the provided JavaScript code that:  
     - Generates stable UUIDs per transaction.  
     - Classifies transactions as income, expense, or internal transfer.  
     - Handles multiple currencies (USD, CAD).  
     - Cleans and extracts necessary fields.  
   - **Important:** Edit the JavaScript arrays `internalTransferAccounts`, `expenseCategories`, and `incomeCategories` to match your QuickBooks Chart of Accounts exactly.  
   - Connect output to the SQL formatting node.

5. **Add a Code Node for SQL Formatting**  
   - Name: `3. Format Data for SQL`  
   - Type: Code (JavaScript)  
   - Parameters: Paste the provided JavaScript code that:  
     - Sanitizes string fields for SQL safety.  
     - Formats dates and other types correctly.  
     - Constructs a single SQL VALUES string for bulk insertion.  
   - Connect output to the BigQuery node.

6. **Add a Google BigQuery Node**  
   - Name: `4. Load Data to BigQuery`  
   - Type: Google BigQuery  
   - Parameters:  
     - SQL Query: Use the provided multi-line INSERT statement referencing the table `quickbooks.transactions` and inserting the values from `{{ $json.valuesString }}`.  
     - Project ID: Select your Google Cloud project containing the BigQuery dataset.  
   - Credentials: Connect your Google BigQuery OAuth2 credentials.  
   - No further connections; this is the terminal node.

7. **Add Sticky Notes**  
   - Add a sticky note with detailed documentation and setup instructions as provided.  
   - Add another sticky note near the transformation node reminding to customize the classification arrays.  
   - Add a sticky note near the formatting node indicating that no changes are typically needed there.

8. **Verify Credentials and Permissions**  
   - Ensure QuickBooks OAuth2 credentials have read access to transactions.  
   - Ensure Google BigQuery OAuth2 credentials have permissions to insert into the target dataset and table.

9. **Activate the Workflow**  
   - Save and activate the workflow. It will run every Monday automatically.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Context or Link                                                    |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------|
| This workflow template facilitates syncing and enriching QuickBooks Online transaction data into Google BigQuery on a weekly schedule. It is designed for users needing advanced BI and historic archiving capabilities beyond QuickBooks native reporting. Customization of classification logic is mandatory to reflect your Chart of Accounts for accurate financial categorization.                                                                                                                                                                                                                                                                                                                                                                                              | Overview and usage instructions embedded in Sticky Note node     |
| The workflow requires a BigQuery dataset and table with a schema matching the SQL INSERT statement defined in the load node. Create this ahead of time in Google Cloud Console.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | Setup prerequisite                                                 |
| For stability and repeatability, ensure the JavaScript arrays for internal transfer, expense, and income categories are carefully maintained as per your QuickBooks accounts. Failure to do so may produce incorrect classification or duplications.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Critical workflow customization note                              |
| The SQL formatting node aggressively sanitizes text fields to avoid SQL injection and syntax errors but may strip certain characters (apostrophes, question marks). Adjust sanitization logic if necessary for your data.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Data sanitization caution                                         |
| BigQuery node uses standard SQL INSERT syntax and expects the project, dataset, and table to exist. Confirm OAuth2 credentials are valid and have appropriate permissions.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | BigQuery integration notes                                        |
| Related resources and examples for QuickBooks API and BigQuery can be found on their official documentation sites.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | External documentation                                            |

---

**Disclaimer:** The provided text is exclusively derived from an n8n automated workflow and complies fully with applicable content policies. All data processed are legal and public.