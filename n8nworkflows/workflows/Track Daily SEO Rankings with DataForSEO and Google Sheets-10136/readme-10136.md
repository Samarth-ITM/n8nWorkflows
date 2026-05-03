Track Daily SEO Rankings with DataForSEO and Google Sheets

https://n8nworkflows.xyz/workflows/track-daily-seo-rankings-with-dataforseo-and-google-sheets-10136


# Track Daily SEO Rankings with DataForSEO and Google Sheets

---
### 1. Workflow Overview

This workflow automates daily tracking of SEO keyword rankings using DataForSEO API and stores the results in a Google Sheet for ongoing monitoring and reporting. It is designed for SEO professionals or digital marketers who want to track the position of multiple keywords on Google search results over time.

The workflow is logically divided into the following blocks:

- **1.1 Daily Trigger:** Automatically starts the workflow once every day at a specified time.
- **1.2 Keyword Input:** Reads the list of keywords (and optional parameters) from a Google Sheets document.
- **1.3 SERP Data Retrieval:** For each keyword, queries the DataForSEO API to obtain Google search engine results page (SERP) data.
- **1.4 Data Extraction & Transformation:** Processes the raw SERP data to extract only organic search results with relevant details like rank and domain.
- **1.5 Timestamping & Formatting:** Adds the current date and prepares the data structure for output.
- **1.6 Results Storage:** Appends the processed ranking data to a dedicated Google Sheet to maintain a historical record.

---

### 2. Block-by-Block Analysis

#### 2.1 Daily Trigger

- **Overview:**  
  Initiates the workflow execution automatically every day at 8 AM to ensure fresh data is collected daily without manual intervention.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Sticky Note1

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Workflow start trigger based on time schedule  
    - Configuration: Triggers daily at hour 8 (8 AM)  
    - Inputs: None (trigger node)  
    - Outputs: Connected to "Fetch Keyword List (Google Sheets)" node  
    - Edge cases: Misconfiguration of time zone could cause unexpected trigger times; node version 1.2 used.  
    - Notes: Time and interval can be adjusted to fit local time zones or different frequencies.

  - **Sticky Note1**  
    - Type: Sticky Note  
    - Role: Documentation for the daily trigger block  
    - Content: Explains schedule trigger usage and adjustment.

---

#### 2.2 Keyword Input

- **Overview:**  
  Retrieves the list of keywords from a Google Sheets document. This list drives the subsequent keyword ranking queries.

- **Nodes Involved:**  
  - Fetch Keyword List (Google Sheets)  
  - Sticky Note

- **Node Details:**

  - **Fetch Keyword List (Google Sheets)**  
    - Type: Google Sheets node  
    - Role: Reads the keyword data from a specified Google Sheet and tab  
    - Configuration:  
      - Document ID points to a Google Sheet named "Daily Monitor Position | n8n"  
      - Sheet name or tab identified by gid=0 (first sheet)  
      - Uses a Service Account credential for authentication  
    - Input: From Schedule Trigger  
    - Output: Passes each keyword row to the next node for API querying  
    - Edge cases:  
      - If sheet is renamed or columns changed (e.g., missing `query` column), data retrieval will fail or produce incorrect results  
      - Permission errors if service account is not properly granted access  
    - Version: 4.6

  - **Sticky Note**  
    - Type: Sticky Note  
    - Role: Provides detailed instructions on how to format the Google Sheet with keywords and optional columns like `country` or `device`.  
    - Provides example sheet link and setup instructions.

---

#### 2.3 SERP Data Retrieval

- **Overview:**  
  Uses DataForSEO API to retrieve Google organic search results for each keyword read from the sheet.

- **Nodes Involved:**  
  - Fetch SERP Data (DataForSEO API)  
  - Sticky Note2

- **Node Details:**

  - **Fetch SERP Data (DataForSEO API)**  
    - Type: DataForSEO node  
    - Role: Queries keyword SERP data for each keyword input  
    - Configuration:  
      - Keyword parameter dynamically set using expression `{{$json.query}}`  
      - Resource: `serp` (Google Organic Search)  
      - Language: English  
      - Location: United States  
      - Depth: Top 10 results  
      - Credentials: DataForSEO API account  
    - Input: From Google Sheets keywords node  
    - Output: JSON results containing rich SERP data including organic, video, people also ask, etc.  
    - Execution: Runs per keyword (not once globally)  
    - Edge cases:  
      - API quota exhaustion or rate limiting can cause failures  
      - Incorrect or missing keywords may cause empty results  
      - Network timeouts or authentication errors possible  
    - Version: 1  
    - Notes: Location and language are configurable for regional tracking.

  - **Sticky Note2**  
    - Type: Sticky Note  
    - Role: Explains DataForSEO API usage, parameters, and caveats including API credit consumption.

---

#### 2.4 Data Extraction & Transformation

- **Overview:**  
  Processes the raw SERP JSON to extract only organic search results, capturing the relevant fields for storage.

- **Nodes Involved:**  
  - Split Out  
  - Extract Query, Rank & Domain

- **Node Details:**

  - **Split Out**  
    - Type: Split Out  
    - Role: Splits the array of results from the API into individual items to process separately  
    - Configuration: Splits on field `tasks[0].result` (the main results array)  
    - Input: From DataForSEO API node  
    - Output: Each organic result item individually processed  
    - Edge cases: If the field path is incorrect or empty, splitting fails or produces no output  
    - Version: 1

  - **Extract Query, Rank & Domain**  
    - Type: Code (JavaScript)  
    - Role: Filters and maps the organic results to an array of simplified objects containing `query`, `rank`, and `domain`  
    - Configuration:  
      - Script iterates all input items  
      - Selects only items where `type === "organic"`  
      - Extracts `keyword`, `rank_group` (rank), and `domain` fields  
    - Input: From Split Out node  
    - Output: Flattened array of organic result objects  
    - Edge cases:  
      - If input JSON structure changes, script may fail or miss data  
      - Empty `items` arrays produce no output  
    - Version: 2

---

#### 2.5 Timestamping & Formatting

- **Overview:**  
  Adds the current date in ISO format and prepares the final data structure with all required fields for appending to Google Sheets.

- **Nodes Involved:**  
  - Add Timestamp & Prepare Output

- **Node Details:**

  - **Add Timestamp & Prepare Output**  
    - Type: Set node  
    - Role: Creates standardized output fields including `query`, `rank`, `domain`, and current `date`  
    - Configuration:  
      - Copies `query`, `rank`, `domain` from input JSON  
      - Adds `date` field with value set to current date in `YYYY-MM-DD` format (e.g., `new Date().toISOString().split('T')[0]`)  
    - Input: From Extract Query, Rank & Domain  
    - Output: Clean JSON ready for Google Sheets appending  
    - Edge cases: Date generation unlikely to fail, but time zone differences could affect date if needed  
    - Version: 3.4

---

#### 2.6 Results Storage

- **Overview:**  
  Appends the processed ranking data to a Google Sheets document to build a historical record of keyword rankings.

- **Nodes Involved:**  
  - Append Results to Google Sheet  
  - Sticky Note3  
  - Sticky Note4

- **Node Details:**

  - **Append Results to Google Sheet**  
    - Type: Google Sheets node  
    - Role: Appends rows to the "rank" sheet in the same Google Spreadsheet as the input keywords  
    - Configuration:  
      - Columns mapped and defined explicitly: `query`, `rank`, `domain`, `date`  
      - Uses Service Account credential for authentication  
      - Operation: Append (adds rows without overwriting)  
      - Target Sheet ID: 1022700330 (specific tab)  
    - Input: From Add Timestamp & Prepare Output  
    - Output: None (end of workflow)  
    - Edge cases:  
      - Sheet tab or columns renamed cause failures or misaligned data  
      - Permission errors if service account lacks write access  
      - Large volume of appends could hit Google Sheets API limits  
    - Version: 4.6

  - **Sticky Note3**  
    - Type: Sticky Note  
    - Role: Documentation on how this node appends data, including usage for trend reports and dashboarding with Looker Studio.

  - **Sticky Note4**  
    - Type: Sticky Note  
    - Role: Sample output format showing how data rows appear in the Google Sheet.

---

### 3. Summary Table

| Node Name                      | Node Type                          | Functional Role                       | Input Node(s)                    | Output Node(s)                    | Sticky Note                                                                                         |
|--------------------------------|----------------------------------|-------------------------------------|---------------------------------|----------------------------------|---------------------------------------------------------------------------------------------------|
| Schedule Trigger               | Schedule Trigger                  | Starts workflow daily at 8 AM       | None                            | Fetch Keyword List (Google Sheets) | ## 🕒 **Daily Schedule Trigger**<br>Starts the workflow automatically `every day`.<br>⚠️ Adjust the time or interval in node settings. |
| Fetch Keyword List (Google Sheets) | Google Sheets                   | Reads keyword list from Google Sheet| Schedule Trigger                | Fetch SERP Data (DataForSEO API) | ## 📄 **Fetch Keyword List (Google Sheets)**<br>Reads your keyword list directly from Google Sheets.<br>👉 Example sheet: [Keyword List](https://docs.google.com/spreadsheets/d/1ShdLc4td6MSQf49l4tDlVohRlFxNO0SdkG0bHQ5LJmE/edit?usp=sharing)<br>Ensure sheet name and headers are correct.<br>Google account must have edit access. |
| Fetch SERP Data (DataForSEO API) | DataForSEO                      | Retrieves SERP data for each keyword| Fetch Keyword List (Google Sheets) | Split Out                      | ## 🔍 **Fetch SERP Data (DataForSEO API)**<br>Uses the DataForSEO API to retrieve Google Search results for each keyword.<br>Location: United States, Language: English.<br>Each keyword consumes one API credit.<br>Adjust location/language as needed. |
| Split Out                     | Split Out                        | Splits API response array for processing | Fetch SERP Data (DataForSEO API) | Extract Query, Rank & Domain    |                                                                                                   |
| Extract Query, Rank & Domain  | Code (JavaScript)                | Extracts organic results and fields | Split Out                      | Add Timestamp & Prepare Output   |                                                                                                   |
| Add Timestamp & Prepare Output | Set                             | Adds current date and formats data | Extract Query, Rank & Domain    | Append Results to Google Sheet    |                                                                                                   |
| Append Results to Google Sheet | Google Sheets                   | Appends processed data to Google Sheet | Add Timestamp & Prepare Output  | None                            | ## 📊 **Append Results to Google Sheet**<br>Writes data (`query`, `rank`, `domain`, `date`) to sheet.<br>Maintains history for reports and dashboards.<br>See sample output below.<br>## 📋 **Sample Output (Google Sheet)**<br>Shows example data rows saved daily.<br>Links:<br>- [Daily Rank Tracker](https://docs.google.com/spreadsheets/d/1ShdLc4td6MSQf49l4tDlVohRlFxNO0SdkG0bHQ5LJmE/edit?usp=sharing) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Set interval to trigger daily at 8:00 AM  
   - No input connections  
   - Output connects to the Google Sheets keyword fetch node

2. **Create a Google Sheets Node to Fetch Keywords**  
   - Type: Google Sheets  
   - Operation: Read rows  
   - Document ID: Use your Google Sheet containing keywords (e.g., "Daily Monitor Position | n8n")  
   - Sheet name/tab: Set to the first sheet or as per your sheet's tab (gid=0)  
   - Authentication: Use a Google Service Account credential with read access to the sheet  
   - Connect input from Schedule Trigger  
   - Output connects to DataForSEO API node

3. **Create a DataForSEO Node to Fetch SERP Data**  
   - Type: DataForSEO  
   - Resource: Search Engine Results Page (SERP)  
   - Keyword: Set dynamically with expression from previous node `{{$json.query}}`  
   - Language: English (or desired language)  
   - Location: United States (or desired region)  
   - Depth: 10 (top 10 results)  
   - Credentials: DataForSEO API account with sufficient credits  
   - Connect input from Google Sheets keyword fetch node  
   - Output connects to Split Out node

4. **Create a Split Out Node**  
   - Type: Split Out  
   - Field to split out: `tasks[0].result` (path to SERP results array)  
   - Connect input from DataForSEO node  
   - Output connects to Code node

5. **Create a Code Node to Extract Relevant Data**  
   - Type: Code (JavaScript)  
   - Paste the following code:  
     ```javascript
     const results = $input.all();
     const rows = [];
     for (const res of results) {
       const keyword = res.json.keyword;
       const items = res.json.items || [];
       for (const item of items) {
         if (item.type === "organic") {
           rows.push({
             query: keyword,
             rank: item.rank_group,
             domain: item.domain
           });
         }
       }
     }
     return rows.map(r => ({ json: r }));
     ```  
   - Connect input from Split Out  
   - Output connects to Set node

6. **Create a Set Node to Add Timestamp & Prepare Output**  
   - Type: Set  
   - Add fields:  
     - `query` = `{{$json.query}}` (string)  
     - `rank` = `{{$json.rank}}` (string)  
     - `domain` = `{{$json.domain}}` (string)  
     - `date` = `{{ new Date().toISOString().split('T')[0] }}` (string, current date in YYYY-MM-DD)  
   - Connect input from Code node  
   - Output connects to Google Sheets append node

7. **Create a Google Sheets Node to Append Results**  
   - Type: Google Sheets  
   - Operation: Append  
   - Document ID: Same Google Sheet as keywords or your target "Daily Rank Tracker" sheet  
   - Sheet name/tab: Use the tab with ID or name for storing ranking results (e.g., gid=1022700330)  
   - Column mapping: Define columns `query`, `rank`, `domain`, `date` explicitly  
   - Authentication: Use Google Service Account credential with write access  
   - Connect input from Set node

8. **Optional: Add Sticky Notes for Documentation**  
   - Add sticky notes at relevant points to explain purposes of blocks, usage instructions, configuration tips, and example links to Google Sheets.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                   | Context or Link                                                                                                              |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| The keyword list Google Sheet example: [Keyword List](https://docs.google.com/spreadsheets/d/1ShdLc4td6MSQf49l4tDlVohRlFxNO0SdkG0bHQ5LJmE/edit?usp=sharing)                                                                 | Input keyword sheet                                                                                                           |
| DataForSEO API location and language codes can be changed to track rankings regionally, e.g., UK location code 2826 or language code 'fr' for French.                                                                            | DataForSEO API documentation                                                                                                |
| The results sheet builds a growing history of keyword rankings for use in reports or visualizations like Looker Studio (Google Data Studio).                                                                                   | Output results sheet: [Daily Rank Tracker](https://docs.google.com/spreadsheets/d/1ShdLc4td6MSQf49l4tDlVohRlFxNO0SdkG0bHQ5LJmE/edit?usp=sharing) |
| Each keyword query consumes one DataForSEO API credit; monitor usage carefully to avoid quota exhaustion.                                                                                                                      | API usage caution                                                                                                            |
| Ensure Google Service Account credentials have proper edit/read permissions in the target Google Sheets.                                                                                                                       | Credential setup                                                                                                             |
| Sample saved data format includes columns: `query`, `rank`, `domain`, `date`, showing daily rankings per keyword with ranking position and domain name.                                                                       | See sticky note sample output section                                                                                       |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.