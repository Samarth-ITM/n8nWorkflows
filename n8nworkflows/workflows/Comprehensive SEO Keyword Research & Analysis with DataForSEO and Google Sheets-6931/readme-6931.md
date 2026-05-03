Comprehensive SEO Keyword Research & Analysis with DataForSEO and Google Sheets

https://n8nworkflows.xyz/workflows/comprehensive-seo-keyword-research---analysis-with-dataforseo-and-google-sheets-6931


# Comprehensive SEO Keyword Research & Analysis with DataForSEO and Google Sheets

### 1. Workflow Overview

This workflow automates comprehensive SEO keyword research and analysis by integrating the DataForSEO API with Google Sheets. It targets digital marketers, SEO specialists, and content strategists who want to gather, analyze, and organize keyword data efficiently.

The workflow is logically divided into the following blocks:

- **1.1 Trigger and Initial Spreadsheet Setup**: Manual start and creation of a new dated Google Sheet with predefined tabs for keyword data.
- **1.2 File Management**: Moves the newly created spreadsheet into a designated Google Drive folder for organization.
- **1.3 DataForSEO API Calls**: Multiple HTTP requests to gather various keyword-related data such as related keywords, keyword suggestions, keyword ideas, autocomplete suggestions, subtopics, and SERP data including "People Also Ask".
- **1.4 Data Processing and Transformation**: Uses split and set nodes to parse and restructure API responses into consistent, flat records with selected fields.
- **1.5 Data Storage**: Appends processed data to corresponding sheets within the created Google Sheet and a master sheet for consolidated analysis.
- **1.6 Filtering and Additional Data Handling**: Filters organic and "People Also Ask" results separately for specialized processing and storage.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger and Initial Spreadsheet Setup

- **Overview**: Starts workflow manually and creates a new Google Sheet with multiple sheets/tabs named for organizing different SEO keyword data categories.
- **Nodes Involved**:  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - Google Sheets1 (Create new spreadsheet)  
  - Google Sheets (Read initial input spreadsheet)  
- **Node Details**:  
  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Initiates workflow on demand  
    - Inputs: None  
    - Outputs: Trigger data flow to next node  
    - Edge Cases: None, manual trigger  
  - **Google Sheets1**  
    - Type: Google Sheets (Spreadsheet creation)  
    - Role: Creates a new Google Sheet titled with the current date and "seo pro" suffix. It predefines sheet tabs: keyword, SERP, Content, related keyword, keyword ideas, suggested keyword, sub topics, auto complete.  
    - Configuration: Dynamic title `={{ $now.format('yyyy-MM-dd') }}-seo pro`  
    - Inputs: Trigger data  
    - Outputs: Spreadsheet metadata including spreadsheet ID  
    - Edge Cases: API quota, Google Sheets API errors, authentication errors  
  - **Google Sheets**  
    - Type: Google Sheets (Read)  
    - Role: Reads from an existing spreadsheet providing primary keywords and contextual data for API calls.  
    - Configuration: Reading "Sheet1" of a specific document by ID (hardcoded in this workflow)  
    - Inputs: Trigger  
    - Outputs: Rows containing primary keywords, location, language, etc.  
    - Edge Cases: Sheet not found, empty data, auth issues  

#### 1.2 File Management

- **Overview**: Moves the newly created spreadsheet to a specific Google Drive folder for organized storage.
- **Nodes Involved**:  
  - Google Drive  
- **Node Details**:  
  - **Google Drive**  
    - Type: Google Drive (Move file)  
    - Role: Moves the spreadsheet created in Google Sheets1 to the "seo pro" folder on Google Drive.  
    - Configuration: Uses spreadsheet ID dynamically from previous node output; Folder ID is fixed to a known folder.  
    - Inputs: Spreadsheet metadata from Google Sheets1  
    - Outputs: Confirmation of file move  
    - Edge Cases: Permission errors, folder not found, invalid spreadsheet ID  

#### 1.3 DataForSEO API Calls

- **Overview**: Executes multiple HTTP POST requests to DataForSEO API endpoints to fetch keyword-related data.
- **Nodes Involved**:  
  - related keyword  
  - keyword suggestion  
  - keyword ideas  
  - get autocomplete  
  - get_subtopics  
  - people also ask  
- **Node Details**:

  For all DataForSEO HTTP Request nodes:

  - Type: HTTP Request  
  - Role: Query DataForSEO API endpoints with primary keyword data and location/language parameters.  
  - Authentication: HTTP Basic Auth using stored credentials.  
  - Inputs: Data from Google Sheets node (primary keyword, location, language)  
  - Outputs: JSON responses with keyword data arrays  
  - Edge Cases: API rate limits, auth failures, malformed request, network timeouts  

  Individual specifics:

  - **related keyword**  
    - Endpoint: `/v3/dataforseo_labs/google/related_keywords/live`  
    - Includes SERP info, keyword difficulty, search volume, CPC, competition, etc.  
  - **keyword suggestion**  
    - Endpoint: `/v3/dataforseo_labs/google/keyword_suggestions/live`  
    - Provides keyword suggestions without SERP info.  
  - **keyword ideas**  
    - Endpoint: `/v3/dataforseo_labs/google/keyword_ideas/live`  
    - Accepts an array of keywords, returns keyword ideas.  
  - **get autocomplete**  
    - Endpoint: `/v3/serp/google/autocomplete/live/advanced`  
    - Gets Google autocomplete suggestions for the primary keyword.  
  - **get_subtopics**  
    - Endpoint: `/v3/content_generation/generate_sub_topics/live`  
    - Generates content subtopics based on the primary keyword.  
  - **people also ask**  
    - Endpoint: `/v3/serp/google/organic/live/advanced`  
    - Retrieves organic results and "People Also Ask" questions.  

#### 1.4 Data Processing and Transformation

- **Overview**: Processes the complex API responses by splitting arrays and mapping fields into uniform objects for easier sheet appending.
- **Nodes Involved**:  
  - Split Out (multiple instances: Split Out, Split Out1, Split Out2, Split Out3, Split Out4, Split Out5, Split Out6)  
  - Edit Fields (multiple instances: Edit Fields, Edit Fields1, Edit Fields2, Edit Fields3, Edit Fields4, Edit Fields5, Edit Fields6)  
  - Filter, Filter1  
- **Node Details**:

  - **Split Out nodes**  
    - Type: Split Out  
    - Role: Extracts each item from array results (e.g., keyword lists) into individual workflow items for further processing.  
    - Configuration: Field to split is typically `tasks[0].result[0].items` or `tasks[0].result[0].sub_topics`.  
    - Inputs/Outputs: Receives bulk array JSON, outputs individual items.  
    - Edge Cases: Empty arrays, unexpected response structure.  

  - **Edit Fields nodes**  
    - Type: Set  
    - Role: Maps and renames fields from API data to a consistent format with fields like "keyword", "keyword difficulty", "search intent", "last update", "CPC", "competition", "type" (e.g., related_keyword, suggested_keyword).  
    - Configuration: Uses expressions to extract and assign fields from JSON, often referencing data from the original Google Sheets input for "primary keyword".  
    - Inputs: Single keyword data object from Split Out nodes  
    - Outputs: Single transformed object per keyword  
    - Edge Cases: Missing fields in API response, expression errors due to nulls  

  - **Filter nodes**  
    - Type: Filter  
    - Role: Separates "people_also_ask" type records and "organic" type records from combined API responses for specialized handling.  
    - Configuration: Filter conditions check `$json.type` equals "people_also_ask" or "organic".  
    - Inputs: Items from Split Out5 (People Also Ask + Organic combined)  
    - Outputs: Filtered streams for each type  
    - Edge Cases: Unexpected type values, empty inputs  

#### 1.5 Data Storage

- **Overview**: Appends the processed and transformed keyword data into appropriate sheets/tabs in both the dynamically created SEO spreadsheet and a master spreadsheet for aggregate data.
- **Nodes Involved**:  
  - Multiple Google Sheets nodes appending data (Google Sheets2, Google Sheets3, Google Sheets4, Google Sheets5, Google Sheets6, Google Sheets7, Google Sheets8, Google Sheets9, Google Sheets10, Google Sheets11, Google Sheets12, Google Sheets13)  
- **Node Details**:

  For each Google Sheets node:

  - Type: Google Sheets (Append operation)  
  - Role: Append a single record or bulk records to a specific sheet/tab within a spreadsheet.  
  - Configuration:  
    - Spreadsheet IDs are mostly hardcoded or dynamically passed.  
    - SheetName is set to the corresponding tab (e.g., "master sheet", "related keywords", "suggested keyword", "keyword ideas", "auto complete", "sub topics").  
    - Columns mappings defined explicitly to match transformed fields.  
  - Inputs: Data from Edit Fields or Edit Fields1-6 nodes  
  - Outputs: Confirmation of append operation  
  - Edge Cases: API limits, auth errors, data schema mismatches, network failures  

#### 1.6 Filtering and Additional Data Handling

- **Overview**: Processes "People Also Ask" and organic SERP results separately for detailed keyword analysis and appending.
- **Nodes Involved**:  
  - Filter  
  - Filter1  
  - Edit Fields1  
  - Google Sheets2  
  - Split Out6  
  - Edit Fields5  
  - Google Sheets13  
- **Node Details**:

  - **Filter**  
    - Filters items where type is "people_also_ask"  
  - **Filter1**  
    - Filters items where type is "organic"  
  - **Edit Fields1**  
    - Maps filtered data fields to structured fields for downstream storage including domain, URL, title, description.  
  - **Google Sheets2**  
    - Appends organic results data to the master sheet.  
  - **Split Out6**  
    - Further splits the filtered "people_also_ask" results to individual items.  
  - **Edit Fields5**  
    - Maps "people_also_ask" items including title, description, URL, domain.  
  - **Google Sheets13**  
    - Appends "people_also_ask" data to master sheet.  

---

### 3. Summary Table

| Node Name                  | Node Type               | Functional Role                                   | Input Node(s)                    | Output Node(s)                      | Sticky Note                                                                                                     |
|----------------------------|-------------------------|--------------------------------------------------|---------------------------------|-----------------------------------|----------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger          | Starts the workflow manually                      | -                               | Google Sheets                     |                                                                                                                |
| Google Sheets              | Google Sheets           | Reads input keywords from existing sheet         | When clicking ‘Test workflow’    | Google Sheets1                    |                                                                                                                |
| Google Sheets1             | Google Sheets           | Creates new dated SEO spreadsheet with tabs      | Google Sheets                   | Google Drive                     | Spreadsheet Creation: Creates a new Google Sheet with dynamic title and predefined sheets. [Guide](https://docs.n8n.io/workflows/sticky-notes/) |
| Google Drive               | Google Drive            | Moves new spreadsheet to "seo pro" folder        | Google Sheets1                  | related keyword, keyword suggestion, keyword ideas, get autocomplete, get_subtopics, people also ask | Google Drive Integration: Moves spreadsheet to designated folder. [Guide](https://docs.n8n.io/workflows/sticky-notes/) |
| related keyword            | HTTP Request (DataForSEO) | Fetches related keywords + SERP info              | Google Drive                   | Split Out                       | API Data Collection: Uses DataForSEO related_keywords endpoint. [Guide](https://docs.n8n.io/workflows/sticky-notes/) |
| Split Out                  | Split Out               | Splits related keywords array to individual items| related keyword                | Edit Fields                     |                                                                                                                |
| Edit Fields                | Set                     | Maps related keyword data fields                  | Split Out                     | Google Sheets3                  |                                                                                                                |
| Google Sheets3             | Google Sheets           | Appends related keyword data to master sheet     | Edit Fields                   | Google Sheets4                 |                                                                                                                |
| Google Sheets4             | Google Sheets           | Appends related keyword data to related keywords sheet | Google Sheets3               | -                             |                                                                                                                |
| keyword suggestion         | HTTP Request (DataForSEO) | Fetches keyword suggestions                       | Google Drive                   | Split Out1                     | API Data Collection: Uses DataForSEO keyword_suggestions endpoint.                                               |
| Split Out1                 | Split Out               | Splits keyword suggestions array                  | keyword suggestion             | Edit Fields2                   |                                                                                                                |
| Edit Fields2               | Set                     | Maps suggested keyword data fields                | Split Out1                    | Google Sheets5                 |                                                                                                                |
| Google Sheets5             | Google Sheets           | Appends suggested keywords data                   | Edit Fields2                  | Google Sheets6                 |                                                                                                                |
| Google Sheets6             | Google Sheets           | Appends suggested keyword data to suggested keyword sheet | Google Sheets5               | -                             |                                                                                                                |
| keyword ideas             | HTTP Request (DataForSEO) | Fetches keyword ideas                             | Google Drive                   | Split Out2                     | API Data Collection: Uses DataForSEO keyword_ideas endpoint.                                                     |
| Split Out2                 | Split Out               | Splits keyword ideas array                         | keyword ideas                 | Edit Fields3                   |                                                                                                                |
| Edit Fields3               | Set                     | Maps keyword ideas data                            | Split Out2                    | Google Sheets7                 |                                                                                                                |
| Google Sheets7             | Google Sheets           | Appends keyword ideas data                         | Edit Fields3                  | Google Sheets8                 |                                                                                                                |
| Google Sheets8             | Google Sheets           | Appends keyword ideas data to keyword ideas sheet| Google Sheets7                | -                             |                                                                                                                |
| get autocomplete           | HTTP Request (DataForSEO) | Fetches autocomplete suggestions                  | Google Drive                   | Split Out3                     | API Data Collection: Uses DataForSEO autocomplete endpoint.                                                     |
| Split Out3                 | Split Out               | Splits autocomplete suggestions                    | get autocomplete              | Edit Fields4                   |                                                                                                                |
| Edit Fields4               | Set                     | Maps autocomplete data                            | Split Out3                    | Google Sheets9                 |                                                                                                                |
| Google Sheets9             | Google Sheets           | Appends autocomplete data to master sheet         | Edit Fields4                  | Google Sheets10                |                                                                                                                |
| Google Sheets10            | Google Sheets           | Appends autocomplete data to autocomplete sheet   | Google Sheets9                | -                             |                                                                                                                |
| get_subtopics              | HTTP Request (DataForSEO) | Generates subtopics from primary keyword          | Google Drive                   | Split Out4                     | API Data Collection: Uses DataForSEO generate_sub_topics endpoint.                                              |
| Split Out4                 | Split Out               | Splits subtopics array                             | get_subtopics                 | Edit Fields6                   |                                                                                                                |
| Edit Fields6               | Set                     | Maps subtopics data                               | Split Out4                    | Google Sheets11                |                                                                                                                |
| Google Sheets11            | Google Sheets           | Appends subtopics data                             | Edit Fields6                  | Google Sheets12                |                                                                                                                |
| Google Sheets12            | Google Sheets           | Appends subtopics data to sub topics sheet         | Google Sheets11               | -                             |                                                                                                                |
| people also ask            | HTTP Request (DataForSEO) | Fetches People Also Ask & organic results          | Google Drive                   | Split Out5                     | API Data Collection: Uses DataForSEO organic/live endpoint                                                        |
| Split Out5                 | Split Out               | Splits combined People Also Ask + organic results  | people also ask              | Filter, Filter1               |                                                                                                                |
| Filter                     | Filter                  | Filters items with type "people_also_ask"          | Split Out5                   | Split Out6                    |                                                                                                                |
| Split Out6                 | Split Out               | Splits People Also Ask items                        | Filter                       | Edit Fields5                  |                                                                                                                |
| Edit Fields5               | Set                     | Maps People Also Ask data                           | Split Out6                   | Google Sheets13               |                                                                                                                |
| Google Sheets13            | Google Sheets           | Appends People Also Ask data                        | Edit Fields5                 | -                             |                                                                                                                |
| Filter1                    | Filter                  | Filters items with type "organic"                   | Split Out5                   | Edit Fields1                 |                                                                                                                |
| Edit Fields1               | Set                     | Maps organic results data                           | Filter1                      | Google Sheets2                |                                                                                                                |
| Google Sheets2             | Google Sheets           | Appends organic results to master sheet             | Edit Fields1                 | -                             |                                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Name: `When clicking ‘Test workflow’`

2. **Create Google Sheets Node to Read Input Keywords**  
   - Type: Google Sheets (Read)  
   - Name: `Google Sheets`  
   - Credentials: Google Sheets OAuth2  
   - Config: Select existing spreadsheet by ID and sheet "Sheet1"  
   - Connect trigger to this node

3. **Create Google Sheets Node to Create New Spreadsheet**  
   - Type: Google Sheets (Spreadsheet)  
   - Name: `Google Sheets1`  
   - Credentials: Google Sheets OAuth2  
   - Operation: Create Spreadsheet  
   - Title: Use expression `={{ $now.format('yyyy-MM-dd') }}-seo pro`  
   - Define sheets: keyword, SERP, Content, related keyword, keyword ideas, suggested keyword, sub topics, auto complete  
   - Connect `Google Sheets` node output to this node input

4. **Create Google Drive Node to Move Spreadsheet to Folder**  
   - Type: Google Drive (Move File)  
   - Name: `Google Drive`  
   - Credentials: Google Drive OAuth2  
   - File ID: Set dynamically from `Google Sheets1` output (spreadsheetId)  
   - Folder ID: Set to target folder ("seo pro") by ID  
   - Connect `Google Sheets1` to `Google Drive`

5. **Create HTTP Request Nodes for DataForSEO API Calls**  
   - Use HTTP Basic Auth credentials for DataForSEO.  
   - Create nodes with following configurations and connect in parallel from `Google Drive`:

   a. `related keyword`  
      - Endpoint: `/v3/dataforseo_labs/google/related_keywords/live`  
      - POST body: JSON with primary keyword, location_name, language_name, depth=3, limit=100, include_serp_info=true  

   b. `keyword suggestion`  
      - Endpoint: `/v3/dataforseo_labs/google/keyword_suggestions/live`  
      - POST body: JSON with primary keyword, location_name, language_name, depth=3, limit=100  

   c. `keyword ideas`  
      - Endpoint: `/v3/dataforseo_labs/google/keyword_ideas/live`  
      - POST body: JSON with keywords array containing primary keyword, location_name, language_name, depth=3, limit=100  

   d. `get autocomplete`  
      - Endpoint: `/v3/serp/google/autocomplete/live/advanced`  
      - POST body: JSON with primary keyword, location_name, language_name, client="gws-wiz-serp"  

   e. `get_subtopics`  
      - Endpoint: `/v3/content_generation/generate_sub_topics/live`  
      - POST body: JSON with topic = primary keyword  

   f. `people also ask`  
      - Endpoint: `/v3/serp/google/organic/live/advanced`  
      - POST body: JSON with primary keyword, location_name, language_name, depth=20, people_also_ask_click_depth=1  

6. **Create Split Out Nodes for Each API Response**  
   - Add a Split Out node per API response to extract array items from `tasks[0].result[0].items` or `sub_topics` for subtopics.  
   - Connect each HTTP Request node to corresponding Split Out node.

7. **Create Set Nodes to Map Fields for Each Data Type**  
   - For each Split Out node, add a Set node that maps the API fields to a uniform format. Examples:  
     - related keyword fields: keyword, keyword difficulty, search intent, last update, CPC, competition, type="related_keyword"  
     - suggested keywords, keyword ideas, autocomplete, subtopics similarly mapped with appropriate field names and types.  
   - Use expressions referencing both original input (primary keyword) and API response fields.

8. **Create Google Sheets Append Nodes for Each Data Type**  
   - Add Google Sheets nodes to append the processed data to appropriate sheets in the created spreadsheet or a master spreadsheet.  
   - Configure each node with corresponding spreadsheet ID, sheet name, and column mappings matching the Set node output fields.  
   - Connect each Set node to its corresponding Google Sheets append node.

9. **Additional Processing for People Also Ask & Organic Results**  
   - Use a Split Out node on `people also ask` HTTP response to split combined items.  
   - Add two Filter nodes: one filtering type "people_also_ask", another filtering "organic".  
   - Connect filters to Set nodes that map fields accordingly.  
   - Append filtered data to master sheet with Google Sheets nodes.  
   - For "people_also_ask" filtered items, add another Split Out and Set node to flatten nested data before appending.

10. **Credential Setup**  
    - Setup Google Sheets OAuth2 credential with appropriate permissions.  
    - Setup Google Drive OAuth2 credential for file operations.  
    - Setup HTTP Basic Auth credentials for DataForSEO API with valid username and password.

11. **Test Workflow**  
    - Use manual trigger to start workflow.  
    - Verify each node executes successfully and data populates into the Google Sheets as expected.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                       | Context or Link                                                                                         |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| Spreadsheet Creation: Creates a new Google Sheet with tabs for keyword data categories, dynamically titled by current date.                                                                                                                                                                                                        | [Guide](https://docs.n8n.io/workflows/sticky-notes/)                                                  |
| Google Drive Integration: Moves the newly created spreadsheet to a specific folder named "seo pro" for organized file management.                                                                                                                                                                                                 | [Guide](https://docs.n8n.io/workflows/sticky-notes/)                                                  |
| API Data Collection: Uses DataForSEO API for extensive keyword research including related keywords, suggestions, ideas, autocomplete, subtopics, and SERP data (People Also Ask, organic results).                                                                                                                                  | [Guide](https://docs.n8n.io/workflows/sticky-notes/)                                                  |
| DataForSEO API requires HTTP Basic Authentication with credentials configured in n8n.                                                                                                                                                                                                                                              | https://dataforseo.com/                                                                                 |
| Workflow handles complex nested JSON responses with split and set nodes to normalize data before appending to sheets.                                                                                                                                                                                                              |                                                                                                       |
| Potential failure points include API rate limits, authentication errors for Google APIs or DataForSEO, malformed JSON responses, and network timeouts. Proper error handling or retries should be considered in production use.                                                                                                      |                                                                                                       |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, respecting content policies and containing no illegal or protected elements. All data processed is legal and publicly available.