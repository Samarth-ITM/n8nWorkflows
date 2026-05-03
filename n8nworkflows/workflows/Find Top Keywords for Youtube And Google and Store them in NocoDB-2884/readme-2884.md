Find Top Keywords for Youtube And Google and Store them in NocoDB

https://n8nworkflows.xyz/workflows/find-top-keywords-for-youtube-and-google-and-store-them-in-nocodb-2884


# Find Top Keywords for Youtube And Google and Store them in NocoDB

### 1. Workflow Overview

The **Find Top Keywords** workflow automates comprehensive keyword research for Google and YouTube platforms, integrating data collection, filtering, enrichment, and storage into NocoDB. It is designed to support SEO specialists, digital marketers, content creators, and agencies by providing scalable, data-driven insights into keyword opportunities.

The workflow is logically divided into the following blocks:

- **1.1 Trigger and Date Generation**: Initiates the workflow manually or on a schedule and generates the reference date for data queries.
- **1.2 Base Keyword Retrieval**: Fetches base keywords from NocoDB to serve as seeds for autocomplete suggestions.
- **1.3 Keyword Expansion via Autocomplete APIs**: Generates second-order keyword suggestions for Google and YouTube using autocomplete APIs.
- **1.4 Keyword Filtering and Combination**: Cleans, deduplicates, and batches the expanded keyword lists for further processing.
- **1.5 Search Volume Data Retrieval**: Queries DataForSEO API for monthly search volume and CPC data for both Google and YouTube keywords.
- **1.6 Keyword Data Storage and Update in NocoDB**: Adds or updates keyword records in NocoDB tables for Google and YouTube.
- **1.7 Monthly Search Volume Bulk Import**: Formats and bulk imports monthly search volume statistics into NocoDB for detailed analysis.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger and Date Generation

- **Overview:** This block starts the workflow either manually or on a scheduled cron (every 4 hours) and generates the date string for "yesterday" to be used in API queries.
- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - Schedule Trigger  
  - Gen Time (Code)  
  - Sticky Note (explanatory)

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Allows manual execution for testing or on-demand runs  
    - Inputs: None  
    - Outputs: Triggers "Gen Time" node  
    - Edge cases: None significant; manual trigger depends on user action

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Automatically triggers workflow every 4 hours via cron expression `0 */4 * * *`  
    - Inputs: None  
    - Outputs: Triggers "Gen Time" node  
    - Edge cases: Cron misconfiguration or n8n downtime could delay runs

  - **Gen Time**  
    - Type: Code (JavaScript)  
    - Role: Calculates and formats the previous day's date as `yyyy-mm-dd` string  
    - Key expressions: Uses JavaScript Date API to subtract one day from current date  
    - Inputs: Trigger node output  
    - Outputs: JSON with `previousDay` property  
    - Edge cases: Timezone differences could affect date accuracy if server time is not UTC

  - **Sticky Note**  
    - Content: Explains the purpose of date generation for news article search range  
    - Role: Documentation only

#### 2.2 Base Keyword Retrieval

- **Overview:** Retrieves the list of base keywords from a NocoDB table to serve as input for keyword expansion.
- **Nodes Involved:**  
  - NocoDB (Get All base keywords)  
  - Sticky Note1

- **Node Details:**

  - **NocoDB**  
    - Type: NocoDB node (Get All)  
    - Role: Fetches all records from the "Base Keyword Search" table, specifically the `keyword` field  
    - Configuration: Returns all records, authenticated via NocoDB API token  
    - Inputs: Output from "Gen Time" node  
    - Outputs: List of base keywords for further processing  
    - Edge cases: API token expiration, network errors, empty table

  - **Sticky Note1**  
    - Content: "Grab a list of base keywords from NocoDB"  
    - Role: Documentation

#### 2.3 Keyword Expansion via Autocomplete APIs

- **Overview:** For each base keyword, generates second-order keyword suggestions from Google and YouTube autocomplete APIs.
- **Nodes Involved:**  
  - Second Order Google Autocomplete Keywords (HTTP Request)  
  - Second Order YouTube Autocomplete Keywords (HTTP Request)  
  - Sticky Note2

- **Node Details:**

  - **Second Order Google Autocomplete Keywords**  
    - Type: HTTP Request  
    - Role: Calls local Social Flood API endpoint for Google autocomplete keywords  
    - Configuration: Sends query parameters including base keyword, country (US), proxy usage, output format, language, and spell check  
    - Authentication: HTTP Header Auth with Social Flood API Key  
    - Inputs: Base keywords from NocoDB node (iterated)  
    - Outputs: JSON with autocomplete keyword suggestions  
    - Edge cases: API downtime, proxy failures, malformed responses

  - **Second Order YouTube Autocomplete Keywords**  
    - Type: HTTP Request  
    - Role: Calls same Social Flood API with `ds=yt` parameter to get YouTube autocomplete keywords  
    - Configuration: Similar to Google node, with YouTube-specific parameter  
    - Authentication: Same as Google node  
    - Inputs: Base keywords from NocoDB node  
    - Outputs: JSON with YouTube autocomplete suggestions  
    - Edge cases: Same as Google node

  - **Sticky Note2**  
    - Content: "Generate YouTube and Google Keywords from base keywords"  
    - Role: Documentation

#### 2.4 Keyword Filtering and Combination

- **Overview:** Cleans, filters, deduplicates, and batches the keyword suggestions from autocomplete APIs to prepare for search volume queries.
- **Nodes Involved:**  
  - Combine G Keywords and Filter (Code)  
  - Combine YT Keywords and Filter (Code)

- **Node Details:**

  - **Combine G Keywords and Filter**  
    - Type: Code (JavaScript)  
    - Role:  
      - Extracts all keyword suggestions from Google autocomplete response  
      - Filters keywords by length (max 80 chars) and word count (max 10 words)  
      - Removes special characters and trims whitespace  
      - Deduplicates keywords  
      - Batches keywords into groups of 1000 for API requests  
    - Inputs: Items from "Second Order Google Autocomplete Keywords" node  
    - Outputs: Batches of cleaned keyword strings ready for search volume API  
    - Edge cases: Empty or malformed keyword data, expression errors

  - **Combine YT Keywords and Filter**  
    - Type: Code (JavaScript)  
    - Role: Same as Google node but for YouTube keyword suggestions  
    - Inputs: Items from "Second Order YouTube Autocomplete Keywords" node  
    - Outputs: Batches of cleaned YouTube keywords  
    - Edge cases: Same as Google node

#### 2.5 Search Volume Data Retrieval

- **Overview:** Queries DataForSEO API to retrieve monthly search volume and CPC data for both Google and YouTube keyword batches.
- **Nodes Involved:**  
  - Google Search Volume (HTTP Request)  
  - YouTube Search Volume (HTTP Request)  
  - Split Out Google Search (Split Out)  
  - Split Out YT Search (Split Out)  
  - Sticky Note3  
  - Sticky Note4

- **Node Details:**

  - **Google Search Volume**  
    - Type: HTTP Request (POST)  
    - Role: Sends batches of Google keywords to DataForSEO API for live search volume data  
    - Configuration: JSON body includes location code (2840 = US), language code (en), keywords array, date from 2021-08-01, no search partners  
    - Authentication: HTTP Basic Auth with DataForSEO credentials  
    - Inputs: Batches from "Combine G Keywords and Filter"  
    - Outputs: API response with keyword metrics  
    - Edge cases: API rate limits, auth errors, malformed requests

  - **YouTube Search Volume**  
    - Type: HTTP Request (POST)  
    - Role: Similar to Google node but includes `search_partners: true` and sorts by search volume  
    - Inputs: Batches from "Combine YT Keywords and Filter"  
    - Outputs: YouTube keyword metrics  
    - Edge cases: Same as Google node

  - **Split Out Google Search**  
    - Type: Split Out  
    - Role: Splits the nested results array from Google Search Volume API response for individual processing  
    - Inputs: Google Search Volume node output  
    - Outputs: Individual keyword metric items  
    - Edge cases: Empty or unexpected response structure

  - **Split Out YT Search**  
    - Type: Split Out  
    - Role: Same as Google split node but for YouTube data  
    - Inputs: YouTube Search Volume node output  
    - Outputs: Individual YouTube keyword metrics

  - **Sticky Note3**  
    - Content: "Query YouTube and Google Keyword search volume."  
    - Role: Documentation

  - **Sticky Note4**  
    - Content: "Process and filter Keywords for monthly traffic and CPC"  
    - Role: Documentation

#### 2.6 Keyword Data Storage and Update in NocoDB

- **Overview:** Checks if keywords already exist in NocoDB tables and either adds new records or updates existing ones for both Google and YouTube keywords.
- **Nodes Involved:**  
  - Check for Google Keyword (HTTP Request)  
  - Is Google Keyword Available (If)  
  - Add Second Tier G Keyword Data (NocoDB Create)  
  - Update Second Tier G Keyword Data (NocoDB Update)  
  - Loop Over Google Keywords (Split In Batches)  
  - Check for YT Keyword (HTTP Request)  
  - Is YT Keyword Avaliable (If)  
  - Add Second Tier YT Keyword Data (NocoDB Create)  
  - Update Second Tier YT Keyword Data (NocoDB Update)  
  - Loop Over YT Keywords (Split In Batches)  
  - Sticky Note5

- **Node Details:**

  - **Check for Google Keyword**  
    - Type: HTTP Request (GET)  
    - Role: Queries NocoDB "Second Order Google Keywords" table to check if keyword exists  
    - Inputs: Individual keyword from "Loop Over Google Keywords"  
    - Outputs: JSON with `pageInfo.totalRows` indicating existence  
    - Edge cases: API errors, empty responses

  - **Is Google Keyword Available**  
    - Type: If  
    - Role: Branches workflow based on whether keyword exists (`totalRows == 0` means not found)  
    - Inputs: Output from "Check for Google Keyword"  
    - Outputs: True branch (Add new), False branch (Update existing)  

  - **Add Second Tier G Keyword Data**  
    - Type: NocoDB Create  
    - Role: Inserts new Google keyword record into NocoDB table  
    - Inputs: Keyword data from "Split Out Google Search"  
    - Outputs: Created record info  
    - Edge cases: API failures, duplicate inserts

  - **Update Second Tier G Keyword Data**  
    - Type: NocoDB Update  
    - Role: Updates existing Google keyword record using record ID from check node  
    - Inputs: Keyword data and record ID  
    - Outputs: Updated record info  
    - Edge cases: Record not found, update conflicts

  - **Loop Over Google Keywords**  
    - Type: Split In Batches  
    - Role: Processes Google keywords in batches of 1000 for efficient API and DB operations  
    - Inputs: Filtered Google keywords  
    - Outputs: Individual keyword items for checking and updating

  - **Check for YT Keyword**  
    - Type: HTTP Request (GET)  
    - Role: Same as Google check but for YouTube keywords in "Second Order YouTube Keywords" table  
    - Inputs: Individual YouTube keyword  
    - Outputs: Existence info

  - **Is YT Keyword Avaliable**  
    - Type: If  
    - Role: Branches workflow for YouTube keywords based on existence  
    - Inputs: Output from "Check for YT Keyword"  
    - Outputs: True branch (Add new), False branch (Update existing)

  - **Add Second Tier YT Keyword Data**  
    - Type: NocoDB Create  
    - Role: Inserts new YouTube keyword record  
    - Inputs: YouTube keyword data  
    - Outputs: Created record info

  - **Update Second Tier YT Keyword Data**  
    - Type: NocoDB Update  
    - Role: Updates existing YouTube keyword record  
    - Inputs: YouTube keyword data and record ID  
    - Outputs: Updated record info

  - **Loop Over YT Keywords**  
    - Type: Split In Batches  
    - Role: Processes YouTube keywords in batches of 1000

  - **Sticky Note5**  
    - Content: "Add or update YouTube or Google Tables in NocoDB"  
    - Role: Documentation

#### 2.7 Monthly Search Volume Bulk Import

- **Overview:** Formats monthly search volume data by combining keyword records with monthly statistics and bulk imports them into a dedicated NocoDB table.
- **Nodes Involved:**  
  - Format G Data (Code)  
  - Format YT Data (Code)  
  - Bulk Import G Monthly Search Volume (HTTP Request)  
  - Bulk Import YT Monthly Search Volume (HTTP Request)

- **Node Details:**

  - **Format G Data**  
    - Type: Code (JavaScript)  
    - Role:  
      - Combines monthly search volume data from "Loop Over Google Keywords" with second-tier Google keyword records  
      - Validates presence of required fields (`keyword`, `Id`, `year`, `month`, `search_volume`)  
      - Constructs unique IDs for each monthly record  
      - Batches results into groups of 1000 for bulk import  
    - Inputs: Items from "Loop Over Google Keywords" and "Add Second Tier G Keyword Data"  
    - Outputs: Batches of formatted monthly search volume data  
    - Edge cases: Missing data fields, empty inputs, batch size limits

  - **Format YT Data**  
    - Type: Code (JavaScript)  
    - Role: Same as Format G Data but for YouTube keyword data  
    - Inputs: Items from "Loop Over YT Keywords" and "Add Second Tier YT Keyword Data"  
    - Outputs: Batches of formatted YouTube monthly search volume data

  - **Bulk Import G Monthly Search Volume**  
    - Type: HTTP Request (POST)  
    - Role: Bulk imports Google monthly search volume batches into NocoDB "Search Volume" table  
    - Configuration: Uses batching with batch size 1000, authenticated with NocoDB API token  
    - Inputs: Batches from "Format G Data"  
    - Outputs: Confirmation of records created  
    - Edge cases: API rate limits, network errors, partial failures

  - **Bulk Import YT Monthly Search Volume**  
    - Type: HTTP Request (POST)  
    - Role: Same as Google bulk import but for YouTube data  
    - Inputs: Batches from "Format YT Data"

---

### 3. Summary Table

| Node Name                          | Node Type               | Functional Role                                      | Input Node(s)                              | Output Node(s)                              | Sticky Note                                                                                                  |
|-----------------------------------|-------------------------|-----------------------------------------------------|-------------------------------------------|---------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’      | Manual Trigger          | Manual workflow start                               | None                                      | Gen Time                                    |                                                                                                              |
| Schedule Trigger                   | Schedule Trigger        | Scheduled workflow start every 4 hours              | None                                      | Gen Time                                    |                                                                                                              |
| Gen Time                          | Code                    | Generates yesterday's date string                    | When clicking ‘Test workflow’, Schedule Trigger | NocoDB                                     | Create time for yesterday and today. This will be used to gather and search for news articles within a specific range. |
| Sticky Note                       | Sticky Note             | Documentation                                        | None                                      | None                                        | Create time for yesterday and today. This will be used to gather and search for news articles within a specific range. |
| NocoDB                           | NocoDB Get All          | Retrieves base keywords                              | Gen Time                                  | Second Order YouTube Autocomplete Keywords, Second Order Google Autocomplete Keywords | Grab a list of base keywords from NocoDB                                                                    |
| Sticky Note1                     | Sticky Note             | Documentation                                        | None                                      | None                                        | Grab a list of base keywords from NocoDB                                                                    |
| Second Order Google Autocomplete Keywords | HTTP Request           | Gets Google autocomplete keyword suggestions        | NocoDB                                    | Combine G Keywords and Filter                | Generate YouTube and Google Keywords from base keywords                                                      |
| Second Order YouTube Autocomplete Keywords | HTTP Request           | Gets YouTube autocomplete keyword suggestions       | NocoDB                                    | Combine YT Keywords and Filter               | Generate YouTube and Google Keywords from base keywords                                                      |
| Sticky Note2                     | Sticky Note             | Documentation                                        | None                                      | None                                        | Generate YouTube and Google Keywords from base keywords                                                      |
| Combine G Keywords and Filter     | Code                    | Cleans, filters, deduplicates Google keywords       | Second Order Google Autocomplete Keywords | Google Search Volume                         |                                                                                                              |
| Combine YT Keywords and Filter    | Code                    | Cleans, filters, deduplicates YouTube keywords      | Second Order YouTube Autocomplete Keywords | YouTube Search Volume                        |                                                                                                              |
| Google Search Volume              | HTTP Request            | Queries DataForSEO for Google keyword metrics       | Combine G Keywords and Filter              | Split Out Google Search                      | Query YouTube and Google Keyword search volume.                                                             |
| YouTube Search Volume             | HTTP Request            | Queries DataForSEO for YouTube keyword metrics      | Combine YT Keywords and Filter             | Split Out YT Search                          | Query YouTube and Google Keyword search volume.                                                             |
| Split Out Google Search           | Split Out               | Splits Google search volume results                  | Google Search Volume                       | Google Filter                               |                                                                                                              |
| Split Out YT Search               | Split Out               | Splits YouTube search volume results                 | YouTube Search Volume                      | YT Filter                                   |                                                                                                              |
| Google Filter                    | Filter                  | Filters Google keywords by monthly searches and CPC | Split Out Google Search                    | Loop Over Google Keywords                    | Process and filter Keywords for monthly traffic and CPC                                                     |
| YT Filter                       | Filter                  | Filters YouTube keywords by monthly searches and CPC| Split Out YT Search                        | Loop Over YT Keywords                        | Process and filter Keywords for monthly traffic and CPC                                                     |
| Loop Over Google Keywords         | Split In Batches        | Processes Google keywords in batches                 | Google Filter                             | Check for Google Keyword, Update Second Tier G Keyword Data |                                                                                                              |
| Loop Over YT Keywords             | Split In Batches        | Processes YouTube keywords in batches                | YT Filter                                | Check for YT Keyword, Update Second Tier YT Keyword Data |                                                                                                              |
| Check for Google Keyword          | HTTP Request            | Checks if Google keyword exists in NocoDB            | Loop Over Google Keywords                  | Is Google Keyword Available                  |                                                                                                              |
| Is Google Keyword Available       | If                      | Branches based on Google keyword existence            | Check for Google Keyword                   | Add Second Tier G Keyword Data, Update Second Tier G Keyword Data |                                                                                                              |
| Add Second Tier G Keyword Data    | NocoDB Create           | Adds new Google keyword record                         | Is Google Keyword Available (true branch) | Format G Data                               | Add or update YouTube or Google Tables in NocoDB                                                           |
| Update Second Tier G Keyword Data | NocoDB Update           | Updates existing Google keyword record                 | Is Google Keyword Available (false branch) | Loop Over Google Keywords                    | Add or update YouTube or Google Tables in NocoDB                                                           |
| Check for YT Keyword             | HTTP Request            | Checks if YouTube keyword exists in NocoDB            | Loop Over YT Keywords                      | Is YT Keyword Avaliable                      |                                                                                                              |
| Is YT Keyword Avaliable          | If                      | Branches based on YouTube keyword existence            | Check for YT Keyword                       | Add Second Tier YT Keyword Data, Update Second Tier YT Keyword Data |                                                                                                              |
| Add Second Tier YT Keyword Data   | NocoDB Create           | Adds new YouTube keyword record                         | Is YT Keyword Avaliable (true branch)     | Format YT Data                              | Add or update YouTube or Google Tables in NocoDB                                                           |
| Update Second Tier YT Keyword Data| NocoDB Update           | Updates existing YouTube keyword record                 | Is YT Keyword Avaliable (false branch)    | Loop Over YT Keywords                        | Add or update YouTube or Google Tables in NocoDB                                                           |
| Format G Data                    | Code                    | Formats Google monthly search volume data for import | Add Second Tier G Keyword Data             | Bulk Import G Monthly Search Volume          |                                                                                                              |
| Format YT Data                   | Code                    | Formats YouTube monthly search volume data for import| Add Second Tier YT Keyword Data            | Bulk Import YT Monthly Search Volume         |                                                                                                              |
| Bulk Import G Monthly Search Volume | HTTP Request            | Bulk imports Google monthly search volume data        | Format G Data                             | Loop Over Google Keywords                    |                                                                                                              |
| Bulk Import YT Monthly Search Volume | HTTP Request            | Bulk imports YouTube monthly search volume data       | Format YT Data                            | Loop Over YT Keywords                        |                                                                                                              |
| Sticky Note3                    | Sticky Note             | Documentation                                        | None                                      | None                                        | Query YouTube and Google Keyword search volume.                                                             |
| Sticky Note4                    | Sticky Note             | Documentation                                        | None                                      | None                                        | Process and filter Keywords for monthly traffic and CPC                                                     |
| Sticky Note5                    | Sticky Note             | Documentation                                        | None                                      | None                                        | Add or update YouTube or Google Tables in NocoDB                                                           |
| Sticky Note6                    | Sticky Note             | Documentation                                        | None                                      | None                                        | Setup Instuctions: Required: NocoDB, N8N, DataforSEO Account, Social Flood Docker Instance; NocoDB tables list |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes**  
   - Add a **Manual Trigger** node named "When clicking ‘Test workflow’" for manual runs.  
   - Add a **Schedule Trigger** node named "Schedule Trigger" with cron expression `0 */4 * * *` to run every 4 hours.

2. **Create Date Generation Node**  
   - Add a **Code** node named "Gen Time" with JavaScript code to calculate yesterday's date in `yyyy-mm-dd` format and output as `previousDay`.

3. **Fetch Base Keywords from NocoDB**  
   - Add a **NocoDB** node named "NocoDB" configured to:  
     - Operation: Get All  
     - Table: Base Keyword Search (table ID or name)  
     - Fields: `keyword`  
     - Authentication: Use NocoDB API token credentials.

4. **Generate Autocomplete Keywords**  
   - Add two **HTTP Request** nodes:  
     - "Second Order Google Autocomplete Keywords":  
       - Method: GET  
       - URL: `http://<SocialFlood_IP>:8000/google-search/autocomplete-keywords`  
       - Query Parameters:  
         - `input_keyword` = base keyword from NocoDB node  
         - `input_country` = "US"  
         - `use_proxy` = "true"  
         - `output` = "toolbar"  
         - `spell` = "1"  
         - `hl` = "en"  
       - Authentication: HTTP Header Auth with Social Flood API Key  
     - "Second Order YouTube Autocomplete Keywords":  
       - Same as Google node but add query parameter `ds=yt` for YouTube.

5. **Combine and Filter Keywords**  
   - Add two **Code** nodes:  
     - "Combine G Keywords and Filter" and "Combine YT Keywords and Filter"  
     - JavaScript code to:  
       - Extract keywords from autocomplete responses  
       - Filter by length (≤80 chars) and word count (≤10)  
       - Remove special characters and trim  
       - Deduplicate  
       - Batch into groups of 1000 keywords as comma-separated strings

6. **Retrieve Search Volume Data from DataForSEO**  
   - Add two **HTTP Request** nodes:  
     - "Google Search Volume":  
       - Method: POST  
       - URL: `https://api.dataforseo.com/v3/keywords_data/google_ads/search_volume/live`  
       - Body: JSON including location_code=2840, language_code="en", keywords array, date_from="2021-08-01", search_partners=false  
       - Authentication: HTTP Basic Auth with DataForSEO credentials  
     - "YouTube Search Volume":  
       - Same as Google node but with `search_partners=true` and `sort_by=search_volume`

7. **Split Search Volume Results**  
   - Add two **Split Out** nodes:  
     - "Split Out Google Search" splitting on `tasks[0].result` from Google Search Volume  
     - "Split Out YT Search" splitting on `tasks[0].result` from YouTube Search Volume

8. **Filter Keywords by Metrics**  
   - Add two **Filter** nodes:  
     - "Google Filter" and "YT Filter"  
     - Conditions: Check that `monthly_searches` and `cpc` fields exist and are valid

9. **Process Keywords in Batches**  
   - Add two **Split In Batches** nodes:  
     - "Loop Over Google Keywords" and "Loop Over YT Keywords" with batch size 1000

10. **Check Keyword Existence in NocoDB**  
    - Add two **HTTP Request** nodes:  
      - "Check for Google Keyword": GET request to NocoDB API with filter `(keyword,eq,<keyword>)` on Google keywords table  
      - "Check for YT Keyword": GET request to NocoDB API with filter `(keyword,eq,<keyword>)` on YouTube keywords table

11. **Branch Based on Existence**  
    - Add two **If** nodes:  
      - "Is Google Keyword Available": Checks if `pageInfo.totalRows == 0`  
      - "Is YT Keyword Avaliable": Same check for YouTube

12. **Add or Update Keyword Records**  
    - Add four **NocoDB** nodes:  
      - "Add Second Tier G Keyword Data": Create operation on Google keywords table with fields from split data  
      - "Update Second Tier G Keyword Data": Update operation with record ID from check node  
      - "Add Second Tier YT Keyword Data": Create operation on YouTube keywords table  
      - "Update Second Tier YT Keyword Data": Update operation with record ID

13. **Format Monthly Search Volume Data**  
    - Add two **Code** nodes:  
      - "Format G Data" and "Format YT Data"  
      - Combine monthly search volume data with keyword records, validate fields, create unique IDs, batch into 1000-item arrays

14. **Bulk Import Monthly Search Volume**  
    - Add two **HTTP Request** nodes:  
      - "Bulk Import G Monthly Search Volume" and "Bulk Import YT Monthly Search Volume"  
      - POST to NocoDB API endpoint for "Search Volume" table  
      - Use batching with batch size 1000  
      - Authentication: NocoDB API token

15. **Connect Nodes According to Workflow Logic**  
    - Connect triggers to "Gen Time"  
    - "Gen Time" to "NocoDB" (base keywords)  
    - "NocoDB" to both autocomplete keyword nodes  
    - Autocomplete nodes to respective combine/filter code nodes  
    - Combine/filter nodes to DataForSEO search volume nodes  
    - Search volume nodes to split out nodes  
    - Split out nodes to filter nodes  
    - Filter nodes to batch loops  
    - Batch loops to check keyword existence nodes  
    - Existence nodes to If nodes  
    - If nodes branch to add or update NocoDB keyword data nodes  
    - Add keyword data nodes to format monthly data nodes  
    - Format nodes to bulk import nodes  
    - Bulk import nodes loop back to batch loops for continuous processing

16. **Credential Setup**  
    - Configure NocoDB API token credentials for all NocoDB nodes  
    - Configure HTTP Header Auth for Social Flood API key  
    - Configure HTTP Basic Auth for DataForSEO API credentials

17. **Create NocoDB Tables**  
    - Base Keyword Search (fields: keyword)  
    - Second Order Google Keywords (fields: keyword, location_code, language_code, search_partners, competition, competition_index, search_volume, cpc, low_top_of_page_bid, high_top_of_page_bid)  
    - Second Order YouTube Keywords (same fields as Google)  
    - Search Volume (fields: unique_id, year, month, search_volume, youtube_keyword_id, google_keyword_id)

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                                                      |
|----------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Setup Instructions: Required tools include NocoDB, n8n, DataForSEO Account (affiliate link), and Social Flood Docker Instance | [DataForSEO](https://app.dataforseo.com/?aff=184401), [Social Flood GitHub](https://github.com/rainmanjam/social-flood), [NocoDB](https://www.nocodb.com/), [n8n](https://n8n.io/) |
| NocoDB Tables required: Base Keyword Search, Second Order Google Keywords, Second Order YouTube Keywords, Search Volume | Detailed in Sticky Note6                                                                             |
| Ensure compliance with local regulations and API terms of service when using this workflow                            | Workflow description                                                                                |
| Social Flood Docker Instance serves as local integration hub for autocomplete APIs                                    | Workflow description                                                                                |

---

This documentation provides a comprehensive understanding of the **Find Top Keywords** workflow, enabling reproduction, modification, and troubleshooting by advanced users and automation agents.