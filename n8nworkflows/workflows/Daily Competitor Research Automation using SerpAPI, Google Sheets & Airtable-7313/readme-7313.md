Daily Competitor Research Automation using SerpAPI, Google Sheets & Airtable

https://n8nworkflows.xyz/workflows/daily-competitor-research-automation-using-serpapi--google-sheets---airtable-7313


# Daily Competitor Research Automation using SerpAPI, Google Sheets & Airtable

### 1. Workflow Overview

This workflow automates daily competitor research by integrating Google Sheets, SerpAPI, and Airtable. Its primary purpose is to fetch a list of target companies from a Google Sheet, query SerpAPI to retrieve competitor information for each company, parse and clean the results, and then log the findings back into Google Sheets and Airtable for centralized tracking.

The workflow is logically divided into these blocks:

- **1.1 Auto Trigger**: Scheduled start of the workflow at a specific time daily.
- **1.2 Company Data Input & Preparation**: Reading company names from Google Sheets and cleaning the list.
- **1.3 Competitor Data Retrieval**: Querying SerpAPI for each company to obtain competitor search data.
- **1.4 Data Extraction & Validation**: Parsing SerpAPI JSON responses to extract competitor names and metadata, then checking if competitors were found.
- **1.5 Results Logging & Sync**: Logging successful and unsuccessful results into separate Google Sheets and syncing all data to Airtable.

---

### 2. Block-by-Block Analysis

#### 2.1 Auto Trigger

**Overview:**  
Starts the workflow automatically every day at 9 AM without manual intervention.

**Nodes Involved:**  
- 🕒 Auto Run (Scheduled)

**Node Details:**  
- **Node:** 🕒 Auto Run (Scheduled)  
- **Type:** Schedule Trigger  
- **Configuration:** Runs once daily at 9:00 AM (UTC or configured timezone).  
- **Inputs:** None (trigger node).  
- **Outputs:** Triggers the next node to read companies from Google Sheets.  
- **Edge Cases:** Timezone misconfiguration may cause unexpected trigger times.  
- **Notes:** This node is the entry point of the workflow.

---

#### 2.2 Company Data Input & Preparation

**Overview:**  
Reads a list of companies from a Google Sheet, cleans the data to remove empty entries and formats it by attaching row numbers, then splits the list for batch processing.

**Nodes Involved:**  
- 📄 Read Companies Sheet  
- 🧹 Clean & Format Company List  
- 🔁 Loop Over Companies

**Node Details:**  

- **Node:** 📄 Read Companies Sheet  
  - **Type:** Google Sheets  
  - **Role:** Reads the "Companies List" sheet to fetch company names.  
  - **Configuration:** Reads from the first sheet (gid=0) of a Google Sheets document specified by `YOUR_GOOGLE_SHEET_ID_HERE`. Uses OAuth2 credentials for authorized access.  
  - **Input:** Trigger from schedule node.  
  - **Output:** Outputs raw company data rows.  
  - **Edge Cases:** Possible issues include invalid credentials, incorrect document ID, or read permission errors.

- **Node:** 🧹 Clean & Format Company List  
  - **Type:** Code (JavaScript)  
  - **Role:** Filters out empty or whitespace-only company names and attaches row numbers for tracking.  
  - **Key Logic:** Iterates over all input items, trims company names, skips empty entries, associates a row number (either from source or calculated), and returns cleaned items as separate entities.  
  - **Input:** Raw data from Google Sheets node.  
  - **Output:** Cleaned list of companies with row metadata.  
  - **Edge Cases:** If the input sheet format changes (e.g., column key changes), the code will fail to extract company names correctly.

- **Node:** 🔁 Loop Over Companies  
  - **Type:** SplitInBatches  
  - **Role:** Processes the company list in batches of 100 to avoid API rate limits or memory overload.  
  - **Configuration:** Batch size set to 100 companies per batch.  
  - **Input:** Cleaned company list items.  
  - **Output:** Sends each batch downstream for competitor search.  
  - **Edge Cases:** Large lists may need smaller batch sizes to prevent timeout; empty batches are skipped.

---

#### 2.3 Competitor Data Retrieval

**Overview:**  
For each company batch, sends a GET request to SerpAPI to retrieve JSON search results about competitors.

**Nodes Involved:**  
- 🌍 Search Company Competitors (SerpAPI)

**Node Details:**  

- **Node:** 🌍 Search Company Competitors (SerpAPI)  
  - **Type:** HTTP Request  
  - **Role:** Queries SerpAPI with the search term "{company} competitors" to get structured search results.  
  - **Configuration:**  
    - Method: GET  
    - URL: `https://serpapi.com/search.json`  
    - Query parameters:  
      - `q` = dynamic company name + "competitors"  
      - `hl` = "en" (language)  
      - `gl` = "us" (country)  
      - `api_key` = `YOUR_SERPAPI_KEY_HERE` (credential placeholder)  
  - **Input:** Batch of companies from the split node.  
  - **Output:** JSON search results from SerpAPI.  
  - **Edge Cases:** API key invalid or quota exceeded causes errors; network issues may cause timeouts; empty or malformed responses need handling.

---

#### 2.4 Data Extraction & Validation

**Overview:**  
Parses the SerpAPI JSON response to extract the company name, up to 10 competitors, the top search result source URL, and other metadata. Then checks if competitors were found to decide the logging path.

**Nodes Involved:**  
- 🧠 Extract Competitor Data from Search  
- 🧐 Has Competitors?

**Node Details:**  

- **Node:** 🧠 Extract Competitor Data from Search  
  - **Type:** Code (JavaScript)  
  - **Role:** Extracts relevant competitor information and metadata from SerpAPI JSON.  
  - **Key Logic:**  
    - Extracts company name from the search query or organic results title.  
    - Collects competitors from multiple JSON segments: related questions, AI overview, organic snippets, and answer boxes.  
    - Limits competitor list to top 10 unique names.  
    - Retrieves the top organic search result link as a source.  
    - Returns structured JSON with fields: company, competitors (comma separated), competitor count, top source, search query, total results, and extraction timestamp.  
    - On failure, returns an error object with default empty values.  
  - **Input:** Raw JSON response from SerpAPI HTTP Request node.  
  - **Output:** Clean structured competitor data or error object.  
  - **Edge Cases:** Unexpected JSON structure may cause extraction failure; malformed data; empty competitor lists.

- **Node:** 🧐 Has Competitors?  
  - **Type:** If  
  - **Role:** Checks if the competitors field is not equal to "No competitors found" (string).  
  - **Configuration:** Condition: `$json.competitors != "No competitors found"`.  
  - **Input:** Parsed competitor data JSON.  
  - **Output:**  
    - If TRUE (competitors found): routes to logging successful results.  
    - If FALSE (no competitors): routes to logging failed searches.  
  - **Edge Cases:** Case sensitivity or unexpected competitor strings might cause misrouting.

---

#### 2.5 Results Logging & Sync

**Overview:**  
Logs successful competitor results and failed searches separately into dedicated Google Sheets, and syncs all records to Airtable for unified data storage.

**Nodes Involved:**  
- 📊 Log to Result Sheet  
- ❌ Log Companies Without Results  
- 🗃️ Sync to Airtable

**Node Details:**  

- **Node:** 📊 Log to Result Sheet  
  - **Type:** Google Sheets  
  - **Role:** Appends or updates successful competitor data in a "Results" Google Sheet.  
  - **Configuration:**  
    - Sheet: First sheet (gid=0) of the specified results Google Sheet (`YOUR_RESULTS_GOOGLE_SHEET_ID_HERE`).  
    - Columns mapped: Source, Company, Competitors, Total Results.  
    - Matching column: Company (for update detection).  
    - Append or update operation with user-entered cell formatting.  
    - Uses OAuth2 credentials.  
  - **Input:** Competitor data where competitors were found.  
  - **Output:** Data logged in Google Sheets, then passed to Airtable sync.  
  - **Edge Cases:** Permission errors, rate limits, or malformed data may cause write failures.

- **Node:** ❌ Log Companies Without Results  
  - **Type:** Google Sheets  
  - **Role:** Logs companies with no competitors found into the same or separate Google Sheet for tracking.  
  - **Configuration:**  
    - Same sheet as above.  
    - Columns mapped with "No Competetitors Found" as Competitors and zero total results.  
    - Append only (no matching).  
  - **Input:** Competitor data where no competitors were found.  
  - **Output:** Logged failed searches, then passed to Airtable sync.  
  - **Edge Cases:** Same as above.

- **Node:** 🗃️ Sync to Airtable  
  - **Type:** Airtable  
  - **Role:** Upserts all logged data (both successful and failed) into an Airtable base and table for centralized database management.  
  - **Configuration:**  
    - Base and Table IDs specified (`YOUR_AIRTABLE_BASE_ID_HERE`, `YOUR_AIRTABLE_TABLE_ID_HERE`).  
    - Columns mapped: Source, Company, Competitors, Total Results.  
    - Matching column: Company (to update existing records).  
    - Uses Airtable Personal Access Token credentials.  
  - **Input:** Data from both Google Sheets logging nodes.  
  - **Output:** Data synchronized with Airtable records.  
  - **Edge Cases:** API rate limits, authentication failures, or schema mismatches.

---

### 3. Summary Table

| Node Name                       | Node Type              | Functional Role                                     | Input Node(s)                  | Output Node(s)                | Sticky Note                                                                                                     |
|--------------------------------|------------------------|----------------------------------------------------|-------------------------------|------------------------------|-----------------------------------------------------------------------------------------------------------------|
| 🕒 Auto Run (Scheduled)         | Schedule Trigger       | Starts workflow daily at 9 AM                       | None                          | 📄 Read Companies Sheet       | **Auto Run (Scheduled):** ⏰ This is the starting point of the workflow. It runs automatically based on schedule. |
| 📄 Read Companies Sheet         | Google Sheets          | Reads list of companies from Google Sheet           | 🕒 Auto Run (Scheduled)        | 🧹 Clean & Format Company List | **Loop Over Companies:** 📋 Fetch companies from Google Sheet, clean, and prepare for processing.                 |
| 🧹 Clean & Format Company List  | Code                   | Cleans and formats company list                      | 📄 Read Companies Sheet        | 🔁 Loop Over Companies        | See above                                                                                                       |
| 🔁 Loop Over Companies          | SplitInBatches         | Splits company list into batches for API calls      | 🧹 Clean & Format Company List | 🌍 Search Company Competitors | See above                                                                                                       |
| 🌍 Search Company Competitors   | HTTP Request           | Sends competitor search query to SerpAPI            | 🔁 Loop Over Companies         | 🧠 Extract Competitor Data    | **Search Company Competitors (SerpAPI):** 🔍 Sends GET request to SerpAPI with "{company} competitors" query.     |
| 🧠 Extract Competitor Data      | Code                   | Extracts company and competitor info from SerpAPI   | 🌍 Search Company Competitors  | 🧐 Has Competitors?           | **Extract & Check Competitors:** Extracts company, competitors, top source, checks results, routes flow accordingly. |
| 🧐 Has Competitors?             | If                     | Checks if competitors were found                     | 🧠 Extract Competitor Data     | 📊 Log to Result Sheet (True), ❌ Log Companies Without Results (False) | See above                                                                                                       |
| 📊 Log to Result Sheet          | Google Sheets          | Logs successful competitor data                      | 🧐 Has Competitors? (True)     | 🗃️ Sync to Airtable           | **Log Results to Sheets & Airtable:** Logs success to Google Sheets and Airtable.                               |
| ❌ Log Companies Without Results| Google Sheets          | Logs companies without competitors                   | 🧐 Has Competitors? (False)    | 🗃️ Sync to Airtable           | See above                                                                                                       |
| 🗃️ Sync to Airtable             | Airtable               | Syncs all data to Airtable base/table                | 📊 Log to Result Sheet, ❌ Log Companies Without Results | None                         | See above                                                                                                       |
| Sticky Note                    | Sticky Note            | Descriptive notes for Auto Run block                  | None                          | None                         | See content in section 5                                                                                        |
| Sticky Note1                   | Sticky Note            | Descriptive notes for Loop Over Companies block       | None                          | None                         | See section 5                                                                                                   |
| Sticky Note2                   | Sticky Note            | Descriptive notes for Search Company Competitors block| None                          | None                         | See section 5                                                                                                   |
| Sticky Note3                   | Sticky Note            | Descriptive notes for Extract & Check Competitors block| None                          | None                         | See section 5                                                                                                   |
| Sticky Note4                   | Sticky Note            | Descriptive notes for Result Logging & Sync block     | None                          | None                         | See section 5                                                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Name: "🕒 Auto Run (Scheduled)"  
   - Set to trigger daily at 9:00 AM.  
   - No credentials needed.

2. **Add a Google Sheets node**  
   - Name: "📄 Read Companies Sheet"  
   - Operation: Read rows from a sheet.  
   - Document ID: Enter your Google Sheet ID containing the companies list.  
   - Sheet Name: Use the first sheet (gid=0).  
   - Use OAuth2 credentials with read access to the Google Sheet.  
   - Connect output from Schedule Trigger node.

3. **Add a Code node**  
   - Name: "🧹 Clean & Format Company List"  
   - Language: JavaScript  
   - Paste the cleaning code that:  
     - Iterates input rows, extracts the "List" column (company name).  
     - Trims names and filters out empty entries.  
     - Assigns row numbers (existing or calculated).  
     - Returns each company as a separate JSON item.  
   - Connect output from Google Sheets node.

4. **Add a SplitInBatches node**  
   - Name: "🔁 Loop Over Companies"  
   - Batch Size: 100  
   - Connect output from Code node.

5. **Add an HTTP Request node**  
   - Name: "🌍 Search Company Competitors (SerpAPI)"  
   - Method: GET  
   - URL: `https://serpapi.com/search.json`  
   - Query parameters:  
     - `q`: Expression `{{$json.company}} competitors`  
     - `hl`: "en"  
     - `gl`: "us"  
     - `api_key`: Your SerpAPI API key (configured in credentials or parameter).  
   - Connect the second output of SplitInBatches node (batch items) to this node.

6. **Add a Code node**  
   - Name: "🧠 Extract Competitor Data from Search"  
   - Language: JavaScript  
   - Paste the extraction script that:  
     - Extracts company name, competitors (up to 10), top source URL, total results, and timestamp.  
     - Handles errors gracefully by returning error info and empty competitor data.  
   - Connect output from HTTP Request node.

7. **Add an If node**  
   - Name: "🧐 Has Competitors?"  
   - Condition: Check if `$json.competitors != "No competitors found"` (string comparison, case-sensitive).  
   - Connect output from Code extraction node.

8. **Add two Google Sheets nodes for logging:**  

   - **Successful results:**  
     - Name: "📊 Log to Result Sheet"  
     - Operation: Append or update rows in a results Google Sheet (specified by `YOUR_RESULTS_GOOGLE_SHEET_ID_HERE`).  
     - Columns: Map Source, Company, Competitors, Total Results from input JSON.  
     - Matching column: Company  
     - Use OAuth2 credentials.  
     - Connect the TRUE output of If node to this node.

   - **Failed searches:**  
     - Name: "❌ Log Companies Without Results"  
     - Operation: Append rows to the same or another Google Sheet.  
     - Columns: Source = "Null", Company, Competitors = "No Competetitors Found", Total Results = 0.  
     - No matching column for update (append only).  
     - Use OAuth2 credentials.  
     - Connect the FALSE output of If node to this node.

9. **Add an Airtable node**  
   - Name: "🗃️ Sync to Airtable"  
   - Operation: Upsert records in Airtable base and table specified by `YOUR_AIRTABLE_BASE_ID_HERE` and `YOUR_AIRTABLE_TABLE_ID_HERE`.  
   - Map fields: Source, Company, Competitors, Total Results.  
   - Matching field: Company  
   - Use Airtable Personal Access Token credentials.  
   - Connect outputs of both Google Sheets logging nodes to this node (merge data).

10. **Add Sticky Notes (optional)** to document workflow sections as per the original description.

11. **Activate and test workflow** ensuring all credentials are set and API keys are valid.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                                       |
|---------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------|
| Auto Run (Scheduled): This node triggers the workflow daily at 9 AM automatically.                             | Workflow Entry Point                                                                                                  |
| Loop Over Companies: Fetches companies from Google Sheet, cleans input, and processes companies in batches.    | Data Input & Preparation                                                                                              |
| Search Company Competitors (SerpAPI): Queries SerpAPI with "{company} competitors" to get structured results. | Competitor Data Retrieval                                                                                             |
| Extract & Check Competitors: Extracts up to 10 competitors, validates presence, routes data accordingly.       | Data Extraction & Validation                                                                                          |
| Log Results to Sheets & Airtable: Logs all results to Google Sheets and syncs data to Airtable.                | Results Logging & Sync                                                                                                |
| SerpAPI Documentation: https://serpapi.com/                                                                    | Reference for HTTP Request query parameters                                                                           |
| Google Sheets API OAuth2 Setup: https://developers.google.com/sheets/api/guides/authorizing                      | Required for Google Sheets nodes                                                                                      |
| Airtable API Reference: https://airtable.com/api                                                                | Required for Airtable node setup                                                                                       |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, a powerful integration and automation tool. All processing complies strictly with current content policies and contains no illegal, offensive, or protected content. All data handled is legal and publicly available.