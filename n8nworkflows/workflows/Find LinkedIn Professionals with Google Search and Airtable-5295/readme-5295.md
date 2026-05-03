Find LinkedIn Professionals with Google Search and Airtable

https://n8nworkflows.xyz/workflows/find-linkedin-professionals-with-google-search-and-airtable-5295


# Find LinkedIn Professionals with Google Search and Airtable

# Find LinkedIn Professionals with Google Search and Airtable — Workflow Reference

---

### 1. Workflow Overview

This workflow automates the process of finding LinkedIn professional profiles using Google Custom Search and storing the results in Airtable. It is designed for recruiters, sales professionals, or researchers seeking targeted LinkedIn profiles based on job titles, industries, and locations.

The workflow is logically divided into these main blocks:

- **1.1 Input Reception and Search Configuration:** Accepts keyword inputs and configures search parameters including pagination control.
- **1.2 Google Custom Search Execution:** Calls Google Custom Search API to retrieve LinkedIn profile results with pagination support.
- **1.3 Data Parsing and Cleaning:** Extracts relevant data fields from API responses and formats them for storage.
- **1.4 Data Storage and Deduplication:** Upserts the cleaned LinkedIn profile data into Airtable, preventing duplicates.
- **1.5 Pagination Control and Rate Limiting:** Determines whether to continue fetching additional pages and respects API rate limits.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Search Configuration

- **Overview:**  
  This block receives or defines the search keywords and other parameters. It sets initial values for pagination and search filters such as position, industry, and location.

- **Nodes Involved:**  
  - When clicking 'Test workflow' (Manual Trigger)  
  - ⚙️ CUSTOMIZE YOUR SEARCH KEYWORDS HERE (Set)  
  - Loop Over Items (SplitInBatches)  
  - Configure Search Settings (Set)  
  - Prepare Search Parameters (Set)

- **Node Details:**

  - **When clicking 'Test workflow'**  
    - *Type:* Manual Trigger  
    - *Role:* Starts the workflow manually for testing or execution.  
    - *Connections:* Outputs to “⚙️ CUSTOMIZE YOUR SEARCH KEYWORDS HERE”.  
    - *Failure Modes:* None typical; manual trigger.

  - **⚙️ CUSTOMIZE YOUR SEARCH KEYWORDS HERE**  
    - *Type:* Set  
    - *Role:* Defines the initial search keywords string (default: "Store Manager Retail London").  
    - *Configuration:* Sets field `keywords_string` with user-editable keywords.  
    - *Connections:* Outputs to “Loop Over Items”.  
    - *Edge Cases:* Empty or invalid keywords will result in ineffective searches.

  - **Loop Over Items**  
    - *Type:* SplitInBatches  
    - *Role:* Processes input keywords in batches (default batch size implied to 1).  
    - *Connections:* Outputs batch to “Configure Search Settings”.  
    - *Edge Cases:* Large batch sizes may cause rate limiting; empty inputs will halt processing.

  - **Configure Search Settings**  
    - *Type:* Set  
    - *Role:* Sets search parameters including Position (from keywords), Industry, Location, Maxresults (default 30), and an unused “Indian Keyword”.  
    - *Connections:* Output connects to “Prepare Search Parameters”.  
    - *Edge Cases:* Empty Industry or Location fields may reduce search precision.

  - **Prepare Search Parameters**  
    - *Type:* Set  
    - *Role:* Assigns values for Google Custom Search query including start index (pagination start), MaxPages, Position, Industry, Region, and IndianKeyword from previous node inputs.  
    - *Connections:* Outputs to “Google Custom Search API”.  
    - *Failure Modes:* Incorrect pagination indexes or missing fields may cause query errors.

---

#### 1.2 Google Custom Search Execution

- **Overview:**  
  This block sends the configured search query to Google Custom Search API, requesting LinkedIn profile results with pagination.

- **Nodes Involved:**  
  - Google Custom Search API (HTTP Request)  
  - Rate Limit Delay (Wait)

- **Node Details:**

  - **Google Custom Search API**  
    - *Type:* HTTP Request  
    - *Role:* Performs the Google Custom Search API call with authentication and query parameters.  
    - *Configuration:*  
      - URL: `https://www.googleapis.com/customsearch/v1`  
      - Query Parameters:  
        - `q`: Combines Position + Industry + "site:linkedin.com/in"  
        - `start`: Pagination start index (dynamic, depends on previous page or initial start index)  
        - `cx`: Custom Search Engine ID (hardcoded)  
      - Authentication: Uses multiple credential types (Query, Header, Custom), with active credentials set for authorization.  
    - *Connections:* Outputs to “Parse LinkedIn Profiles”.  
    - *Failure Modes:*  
      - API quota exceeded  
      - Authentication failures  
      - Invalid query parameter errors  
      - Network timeouts or rate limit errors

  - **Rate Limit Delay**  
    - *Type:* Wait  
    - *Role:* Introduces a 2-second delay between API calls to avoid hitting rate limits.  
    - *Connections:* Loops back to “Google Custom Search API” if pagination continues.  
    - *Failure Modes:* None typical; long delays may reduce throughput.

---

#### 1.3 Data Parsing and Cleaning

- **Overview:**  
  This block extracts relevant profile data from the Google API response, reformats it, and cleans the data for storage.

- **Nodes Involved:**  
  - Parse LinkedIn Profiles (Code)  
  - Clean Search Results (Set)

- **Node Details:**

  - **Parse LinkedIn Profiles**  
    - *Type:* Code (JavaScript)  
    - *Role:* Parses the JSON response to extract LinkedIn profile details such as title, link, snippet, description, and pagination indexes.  
    - *Configuration:* Custom JS code loops through all API response items; retrieves meta tags and pagination info.  
    - *Key Variables:*  
      - `response.items` for search results  
      - `response.queries.request[0].searchTerms` for original search terms  
      - `response.queries.nextPage[0].startIndex` for next page index  
    - *Connections:* Outputs structured JSON to “Clean Search Results”.  
    - *Failure Modes:*  
      - Missing or malformed API response data  
      - Null or undefined fields in results

  - **Clean Search Results**  
    - *Type:* Set  
    - *Role:* Renames and formats parsed data fields to friendly names: Title, URL, Search, Snippet, Description, nextPageStartIndex.  
    - *Connections:* Outputs to “Save to Airtable”.  
    - *Failure Modes:* Data type mismatches or missing fields.

---

#### 1.4 Data Storage and Deduplication

- **Overview:**  
  Saves the cleaned LinkedIn profile data to Airtable, using upsert to avoid duplicates based on profile URL.

- **Nodes Involved:**  
  - Save to Airtable

- **Node Details:**

  - **Save to Airtable**  
    - *Type:* Airtable node  
    - *Role:* Upserts records into an Airtable base and table specified by the user.  
    - *Configuration:*  
      - Base ID: Linked Airtable Base  
      - Table: "LinkedIn Prospects"  
      - Mapping: Title, linkedin_url, Search, Description, Snippet fields mapped from input JSON  
      - Deduplication: Matches on `linkedin_url` to prevent duplicates.  
    - *Connections:* Outputs to “Check Pagination”.  
    - *Failure Modes:*  
      - Airtable API errors (authentication, rate limits)  
      - Schema mismatch or missing table/fields

---

#### 1.5 Pagination Control and Rate Limiting

- **Overview:**  
  Controls whether the workflow should continue fetching more pages based on the nextPageStartIndex and max results limit. Applies rate limiting.

- **Nodes Involved:**  
  - Check Pagination (Switch)  
  - Rate Limit Delay (Wait)  
  - Loop Over Items (SplitInBatches)

- **Node Details:**

  - **Check Pagination**  
    - *Type:* Switch  
    - *Role:* Compares `nextPageStartIndex` against Maxresults to decide if further pages should be fetched.  
    - *Rules:*  
      - "Continue Searching" if nextPageStartIndex < Maxresults  
      - "Search Complete" if nextPageStartIndex >= Maxresults  
    - *Connections:*  
      - Continue Searching → Rate Limit Delay (to wait before next API call)  
      - Search Complete → Loop Over Items (to possibly process next batch of keywords)  
    - *Failure Modes:* Incorrect or missing pagination indexes can cause infinite loops or premature termination.

  - **Rate Limit Delay**  
    - *Explained above in 1.2*  

  - **Loop Over Items**  
    - *Explained above in 1.1*  

---

### 3. Summary Table

| Node Name                      | Node Type          | Functional Role                         | Input Node(s)                  | Output Node(s)                | Sticky Note                                                                                                   |
|-------------------------------|--------------------|---------------------------------------|-------------------------------|------------------------------|---------------------------------------------------------------------------------------------------------------|
| When clicking 'Test workflow'  | Manual Trigger     | Initiates workflow manually            | —                             | ⚙️ CUSTOMIZE YOUR SEARCH KEYWORDS HERE |                                                                                                               |
| ⚙️ CUSTOMIZE YOUR SEARCH KEYWORDS HERE | Set                | Sets initial search keywords            | When clicking 'Test workflow'  | Loop Over Items               |                                                                                                               |
| Loop Over Items                | SplitInBatches     | Processes keywords in batches           | ⚙️ CUSTOMIZE YOUR SEARCH KEYWORDS HERE | Configure Search Settings      |                                                                                                               |
| Configure Search Settings      | Set                | Sets search parameters and limits       | Loop Over Items                | Prepare Search Parameters     |                                                                                                               |
| Prepare Search Parameters      | Set                | Prepares query parameters for API       | Configure Search Settings      | Google Custom Search API      |                                                                                                               |
| Google Custom Search API       | HTTP Request       | Executes Google Custom Search API call  | Prepare Search Parameters      | Parse LinkedIn Profiles       | **Google Search Process**: Uses Google Custom Search API to find LinkedIn profiles, with pagination & rate limiting |
| Parse LinkedIn Profiles        | Code               | Parses API response for profile data    | Google Custom Search API       | Clean Search Results          |                                                                                                               |
| Clean Search Results           | Set                | Cleans and formats parsed data          | Parse LinkedIn Profiles        | Save to Airtable              |                                                                                                               |
| Save to Airtable               | Airtable           | Stores profile data with deduplication  | Clean Search Results           | Check Pagination             | **Airtable Storage**: Saves LinkedIn data with deduplication to Airtable base/table                             |
| Check Pagination              | Switch             | Controls pagination flow                 | Save to Airtable               | Rate Limit Delay / Loop Over Items |                                                                                                               |
| Rate Limit Delay              | Wait               | Enforces delay between API calls        | Check Pagination              | Google Custom Search API      |                                                                                                               |
| Workflow Overview             | Sticky Note        | Overview and instructions                | —                             | —                            | **LinkedIn Prospect Finder with Google Search**: Setup and workflow summary                                   |
| Search Configuration Guide    | Sticky Note        | Instructions on customizing keywords    | —                             | —                            | Explains keyword examples and targeting strategy                                                             |
| Google Search Process         | Sticky Note        | Explains Google Search API usage        | —                             | —                            |                                                                                                               |
| Airtable Storage             | Sticky Note        | Explains Airtable storage details       | —                             | —                            |                                                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node:**  
   - Name: When clicking 'Test workflow'  
   - Purpose: To manually start the workflow.

2. **Create a Set node to define search keywords:**  
   - Name: ⚙️ CUSTOMIZE YOUR SEARCH KEYWORDS HERE  
   - Add a string field `keywords_string` with a default value, e.g., "Store Manager Retail London".  
   - Connect manual trigger output to this node.

3. **Add a SplitInBatches node:**  
   - Name: Loop Over Items  
   - Purpose: Process keywords in manageable batches (default batch size).  
   - Connect from the keywords Set node.

4. **Create a Set node to configure search parameters:**  
   - Name: Configure Search Settings  
   - Add fields:  
     - `Position` (string), set to expression: `{{$json.keywords_string}}`  
     - `Industry` (string), empty by default  
     - `Location` (string), empty by default  
     - `Maxresults` (number), default 30  
     - `Indian Keyword` (string), empty by default (unused)  
   - Connect from SplitInBatches node.

5. **Create a Set node to prepare search parameters:**  
   - Name: Prepare Search Parameters  
   - Add fields:  
     - `start_index` (number), default 1  
     - `MaxPages` (number), set from `Maxresults` above  
     - `Position` (string), from previous `Position` field  
     - `Industry` (string)  
     - `Region` (string), from `Location` field  
     - `IndianKeyword` (string)  
   - Connect from Configure Search Settings.

6. **Add an HTTP Request node for Google Custom Search API:**  
   - Name: Google Custom Search API  
   - Method: GET  
   - URL: `https://www.googleapis.com/customsearch/v1`  
   - Authentication: Set up Google API credentials (API Key with Custom Search permissions)  
   - Query Parameters:  
     - `q`: Expression combining position, industry, and site filter: `={{ $node['Prepare Search Parameters'].json.Position }} {{ $node['Prepare Search Parameters'].json.Industry }} site:linkedin.com/in`  
     - `start`: Expression for pagination start index: `={{ $runIndex == 0 ? $('Prepare Search Parameters').json.start_index : $('Clean Search Results').json.nextPageStartIndex}}`  
     - `cx`: Your Google Custom Search Engine ID  
   - Connect from Prepare Search Parameters.

7. **Add a Code node to parse LinkedIn profiles:**  
   - Name: Parse LinkedIn Profiles  
   - Paste the provided JavaScript code that extracts titles, links, snippets, descriptions, and pagination indexes from API response.  
   - Connect from Google Custom Search API.

8. **Add a Set node to clean and rename fields:**  
   - Name: Clean Search Results  
   - Map fields:  
     - Title ← `title`  
     - URL ← `link`  
     - Search ← `searchTerms`  
     - Snippet ← `snippet`  
     - Description ← `ogDescription`  
     - nextPageStartIndex ← `nextPageStartIndex` (number)  
   - Connect from Parse LinkedIn Profiles.

9. **Add an Airtable node to upsert records:**  
   - Name: Save to Airtable  
   - Credentials: Configure Airtable Personal Access Token credentials.  
   - Base: Select your Airtable base.  
   - Table: Select the table (e.g., "LinkedIn Prospects").  
   - Operation: Upsert  
   - Mapping:  
     - Title → `Title`  
     - linkedin_url → `URL`  
     - Search → `Search`  
     - Description → `Description`  
     - Snippet → `Snippet`  
   - Matching Field: `linkedin_url` to prevent duplicates.  
   - Connect from Clean Search Results.

10. **Add a Switch node to check pagination:**  
    - Name: Check Pagination  
    - Conditions:  
      - If `nextPageStartIndex` < `Maxresults`, route to “Continue Searching”  
      - Else, route to “Search Complete”  
    - Connect input from Save to Airtable.

11. **Add a Wait node for rate limiting:**  
    - Name: Rate Limit Delay  
    - Wait time: 2 seconds  
    - Connect “Continue Searching” output from Switch node here.

12. **Loop connections:**  
    - Connect Rate Limit Delay output back to Google Custom Search API to fetch next page.  
    - Connect “Search Complete” output from Switch node to Loop Over Items to process next batch of keywords.

13. **Add Sticky Notes for documentation:**  
    - Workflow Overview: Summarize purpose and setup instructions.  
    - Search Configuration Guide: Examples of keywords to use.  
    - Google Search Process: Explain API usage and pagination.  
    - Airtable Storage: Explain data fields and deduplication.

---

### 5. General Notes & Resources

| Note Content                                                                                                          | Context or Link                                                                                  |
|-----------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Setup Google Custom Search API: Obtain API key and Custom Search Engine ID.                                           | https://developers.google.com/custom-search/v1/overview                                         |
| Airtable base must have fields: Title, linkedin_url, Search, Description, Snippet for proper data mapping and storage. | https://airtable.com/                                                                           |
| Be mindful of Google API quota limits; workflow includes a 2-second delay to reduce risk of rate limiting.            |                                                                                                |
| Keywords should be customized to target specific job titles, industries, and locations to improve search relevance.   | Sticky Note “Search Configuration Guide”                                                       |
| Deduplication in Airtable is based on LinkedIn profile URL to avoid repeated entries.                                 | Sticky Note “Airtable Storage”                                                                 |
| Pagination is handled automatically up to the maximum number of results defined in Maxresults (default 30).           |                                                                                                |

---

**Disclaimer:**  
The provided text is derived solely from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.