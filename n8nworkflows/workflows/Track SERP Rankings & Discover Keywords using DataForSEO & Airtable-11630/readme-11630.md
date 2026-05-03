Track SERP Rankings & Discover Keywords using DataForSEO & Airtable

https://n8nworkflows.xyz/workflows/track-serp-rankings---discover-keywords-using-dataforseo---airtable-11630


# Track SERP Rankings & Discover Keywords using DataForSEO & Airtable

### 1. Workflow Overview

This n8n workflow, titled **"Track SERP Rankings & Discover Keywords using DataForSEO & Airtable"**, automates SEO data collection and enrichment by integrating DataForSEO API with Airtable. It targets SEO professionals and marketers who want to maintain up-to-date SERP rankings, competitor keyword insights, and keyword expansion ideas without manual research. The workflow is designed to run on demand and consists of three major logical blocks:

- **1.1 Input and Trigger Block**: Initiates the workflow manually and fetches seed keywords and competitor domains from Airtable.
- **1.2 SERP Ranking Tracking Block**: Posts keyword ranking tasks to DataForSEO, waits for results, retrieves SERP data, and stores structured results in Airtable.
- **1.3 Competitor Keyword Research and Keyword Expansion Block**: Retrieves competitor keywords from DataForSEO, aggregates keywords, fetches related keyword suggestions, and stores all results in Airtable.

These blocks are interconnected, facilitating a continuous SEO data pipeline from input data retrieval to enriched output storage.

---

### 2. Block-by-Block Analysis

#### 2.1 Input and Trigger Block

- **Overview:**  
  This block triggers the workflow manually and searches Airtable tables for seed keywords and competitor domains to use as input for subsequent processing.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - Search Keywords (Airtable)  
  - Search Competitors (Airtable)  
  - Search Keywords1 (Airtable)  

- **Node Details:**  

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Starts the workflow on user command.  
    - Configuration: No parameters; manual execution only.  
    - Input: None  
    - Output: Triggers three Airtable search nodes simultaneously.  
    - Edge cases: Workflow requires manual start; no scheduling or webhook trigger configured.

  - **Search Keywords**  
    - Type: Airtable (Search operation)  
    - Role: Fetch seed keywords from the "SERP Keywords" Airtable table.  
    - Configuration: Filters records where `{Keyword} != "BLANK"`. Uses Airtable Personal Access Token credential.  
    - Input: Trigger output  
    - Output: List of keyword entries for further processing.  
    - Edge cases: Empty or missing keyword field might result in no data; API rate limits possible.

  - **Search Competitors**  
    - Type: Airtable (Search operation)  
    - Role: Fetch competitor domains from the "Competitor Research" Airtable table filtered by `{Research} = "Yes"`.  
    - Configuration: Uses same Airtable credential.  
    - Input: Trigger output  
    - Output: List of competitor domains for competitor keyword research.  
    - Edge cases: No competitors marked "Yes" will result in empty downstream data.

  - **Search Keywords1**  
    - Type: Airtable (Search operation)  
    - Role: Duplicate of Search Keywords node, used for keyword aggregation in a later block.  
    - Configuration: Same filter as Search Keywords.  
    - Input: Trigger output  
    - Output: List of keywords for aggregation.  
    - Edge cases: Same as Search Keywords node.

---

#### 2.2 SERP Ranking Tracking Block

- **Overview:**  
  This block posts organic SERP ranking tasks to DataForSEO for each keyword, waits for processing, retrieves results, splits individual ranking items, and stores them in Airtable.

- **Nodes Involved:**  
  - Post Search Rankings (HTTP Request)  
  - Wait (Wait node)  
  - Get Search Rankings (HTTP Request)  
  - Split Out (Split Out node)  
  - Create Search Rankings Records (Airtable)  

- **Node Details:**

  - **Post Search Rankings**  
    - Type: HTTP Request  
    - Role: Sends POST request to DataForSEO API to initiate a SERP ranking task for each keyword.  
    - Configuration: JSON body includes language_code "en", location_code 2840 (USA), keyword from Airtable JSON, depth 10 results. Uses HTTP Basic Auth credential with DataForSEO token. Content-Type set to application/json.  
    - Input: Output from Search Keywords node (keywords).  
    - Output: Task creation response including task ID.  
    - Edge cases: Auth failure, invalid keyword data, API rate limiting or timeouts.

  - **Wait**  
    - Type: Wait node  
    - Role: Pauses execution for 1 minute to allow DataForSEO to process ranking task.  
    - Configuration: Wait for 1 minute.  
    - Input: Task post response from Post Search Rankings.  
    - Output: Triggers retrieval of results after delay.  
    - Edge cases: Insufficient wait time may cause incomplete data retrieval.

  - **Get Search Rankings**  
    - Type: HTTP Request  
    - Role: Retrieves SERP ranking task result from DataForSEO using task ID from Post Search Rankings.  
    - Configuration: URL dynamically constructed using task ID from previous response. Uses HTTP Basic Auth credential.  
    - Input: After Wait node triggers.  
    - Output: JSON results with SERP ranking data.  
    - Edge cases: Task not ready, invalid task ID, API errors.

  - **Split Out**  
    - Type: Split Out node  
    - Role: Extracts individual SERP ranking items from nested response JSON for processing.  
    - Configuration: Splits on field `tasks[0].result[0].items`.  
    - Input: Output of Get Search Rankings.  
    - Output: Individual SERP ranking records sent downstream.  
    - Edge cases: Empty results or unexpected JSON structure.

  - **Create Search Rankings Records**  
    - Type: Airtable (Create operation)  
    - Role: Stores each SERP ranking record as a new row in "SERP rankings" Airtable table.  
    - Configuration: Maps fields like url, page, rank, title, description, domain, etc. Links record to original keyword by Airtable record ID. Uses Airtable Personal Access Token credential.  
    - Input: Individual ranking items from Split Out node.  
    - Output: Confirmation of record creation.  
    - Edge cases: Airtable API limits, field mapping errors, missing fields.

---

#### 2.3 Competitor Keyword Research and Keyword Expansion Block

- **Overview:**  
  This block fetches competitor domains from Airtable, posts DataForSEO tasks to get competitor keywords, waits for results, retrieves and splits results, creates Airtable records for competitor keywords, then aggregates seed keywords to get related keyword suggestions and stores them as well.

- **Nodes Involved:**  
  - Post Competitor Keywords (HTTP Request)  
  - Wait1 (Wait node)  
  - Get Competitor Keywords (HTTP Request)  
  - Split Out1 (Split Out node)  
  - Create Competitor Keywords Records (Airtable)  
  - Aggregate (Aggregate node)  
  - Get Similar Keywords (HTTP Request)  
  - Split Out2 (Split Out node)  
  - Create Similar Keywords Records (Airtable)

- **Node Details:**

  - **Post Competitor Keywords**  
    - Type: HTTP Request  
    - Role: Posts a task to DataForSEO to get Google Ads keywords for each competitor domain.  
    - Configuration: JSON body includes location_code 2840 and target domain name from Airtable competitor record. Uses HTTP Basic Auth credential.  
    - Input: Output from Search Competitors node.  
    - Output: Task creation with task ID.  
    - Edge cases: Invalid domain, API errors.

  - **Wait1**  
    - Type: Wait node  
    - Role: Waits 1 minute for DataForSEO competitor keywords task processing.  
    - Input: Task post response.  
    - Output: Triggers retrieval of competitor keyword results.  
    - Edge cases: Insufficient wait time.

  - **Get Competitor Keywords**  
    - Type: HTTP Request  
    - Role: Retrieves competitor keywords task result from DataForSEO using task ID.  
    - Configuration: URL constructed dynamically with task ID. Uses HTTP Basic Auth.  
    - Input: After Wait1.  
    - Output: Competitor keyword data JSON.  
    - Edge cases: Task not ready, invalid ID.

  - **Split Out1**  
    - Type: Split Out node  
    - Role: Splits competitor keyword results from nested JSON array `tasks[0].result`.  
    - Input: Output of Get Competitor Keywords.  
    - Output: Individual competitor keyword records.  
    - Edge cases: Empty data or structure change.

  - **Create Competitor Keywords Records**  
    - Type: Airtable (Create)  
    - Role: Stores competitor keyword data into "Competitor Keywords Research" Airtable table, mapping multiple fields like CPC, search volume by month, competition level, bids, etc. Links record to originating competitor Airtable ID.  
    - Input: Split competitor keyword items.  
    - Output: Record creation confirmations.  
    - Edge cases: Airtable API limits, data type conversion errors.

  - **Aggregate**  
    - Type: Aggregate node  
    - Role: Aggregates the seed keywords list fetched earlier (from Search Keywords1) to create a consolidated structure for related keyword search.  
    - Configuration: Aggregates by "Keyword" field.  
    - Input: Output of Search Keywords1.  
    - Output: Aggregated keyword list to use for related keyword API call.  
    - Edge cases: Empty input, aggregation misconfiguration.

  - **Get Similar Keywords**  
    - Type: HTTP Request  
    - Role: Posts a request to DataForSEO API to get similar/related keywords for aggregated keyword list.  
    - Configuration: JSON body contains aggregated keywords, language_code "en", location_code 2840. Uses HTTP Basic Auth credential.  
    - Input: Output of Aggregate node.  
    - Output: Similar keyword suggestions JSON.  
    - Edge cases: Large keyword list causing API errors, auth issues.

  - **Split Out2**  
    - Type: Split Out node  
    - Role: Splits the related keyword items from nested JSON array `tasks[0].result`.  
    - Input: Output of Get Similar Keywords.  
    - Output: Individual similar keyword records.  
    - Edge cases: Empty or malformed results.

  - **Create Similar Keywords Records**  
    - Type: Airtable (Create)  
    - Role: Stores similar keyword data into "Similar Keywords" Airtable table with detailed fields including monthly search volumes, CPC, competition, bid data, etc.  
    - Input: Split similar keyword items.  
    - Output: Airtable creation confirmations.  
    - Edge cases: Airtable limits, field mapping issues.

---

### 3. Summary Table

| Node Name                   | Node Type           | Functional Role                             | Input Node(s)                      | Output Node(s)                                | Sticky Note                                                                                          |
|-----------------------------|---------------------|---------------------------------------------|----------------------------------|----------------------------------------------|----------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger       | Initiates workflow manually                  | None                             | Search Keywords, Search Competitors, Search Keywords1 |                                                                                                    |
| Search Keywords             | Airtable (Search)    | Fetch seed keywords from Airtable            | When clicking ‘Execute workflow’ | Post Search Rankings                          |                                                                                                    |
| Search Competitors          | Airtable (Search)    | Fetch competitor domains for research        | When clicking ‘Execute workflow’ | Post Competitor Keywords                      |                                                                                                    |
| Search Keywords1            | Airtable (Search)    | Fetch seed keywords for aggregation           | When clicking ‘Execute workflow’ | Aggregate                                     |                                                                                                    |
| Post Search Rankings        | HTTP Request        | Post SERP ranking task to DataForSEO          | Search Keywords                  | Wait                                         |                                                                                                    |
| Wait                       | Wait                 | Wait 1 minute for SERP task processing        | Post Search Rankings             | Get Search Rankings                           |                                                                                                    |
| Get Search Rankings         | HTTP Request        | Retrieve SERP ranking results                  | Wait                            | Split Out                                     |                                                                                                    |
| Split Out                  | Split Out            | Split individual SERP ranking items            | Get Search Rankings              | Create Search Rankings Records                 |                                                                                                    |
| Create Search Rankings Records | Airtable (Create)   | Store SERP ranking data in Airtable             | Split Out                       | None                                         |                                                                                                    |
| Post Competitor Keywords    | HTTP Request        | Post competitor keyword task to DataForSEO    | Search Competitors               | Wait1                                        |                                                                                                    |
| Wait1                      | Wait                 | Wait 1 minute for competitor keywords results  | Post Competitor Keywords         | Get Competitor Keywords                       |                                                                                                    |
| Get Competitor Keywords     | HTTP Request        | Retrieve competitor keywords results            | Wait1                           | Split Out1                                    |                                                                                                    |
| Split Out1                 | Split Out            | Split competitor keyword results                 | Get Competitor Keywords          | Create Competitor Keywords Records             |                                                                                                    |
| Create Competitor Keywords Records | Airtable (Create)   | Store competitor keywords in Airtable           | Split Out1                      | None                                         |                                                                                                    |
| Aggregate                  | Aggregate            | Aggregate seed keywords for related keyword search | Search Keywords1                | Get Similar Keywords                          |                                                                                                    |
| Get Similar Keywords       | HTTP Request        | Request related keyword suggestions from DataForSEO | Aggregate                     | Split Out2                                    |                                                                                                    |
| Split Out2                 | Split Out            | Split related keyword suggestion results         | Get Similar Keywords             | Create Similar Keywords Records                |                                                                                                    |
| Create Similar Keywords Records | Airtable (Create)   | Store related keyword data in Airtable           | Split Out2                     | None                                         |                                                                                                    |
| Sticky Note                | Sticky Note          | Keyword Ideas Tool visual label                  | None                           | None                                         | ## Keywords Ideas Tool                                                                              |
| Sticky Note2               | Sticky Note          | Competitor Keywords tracking visual label         | None                           | None                                         | ## Track Competitor Keywords                                                                        |
| Sticky Note3               | Sticky Note          | SERP ranking tracking visual label                 | None                           | None                                         | ## Track Search Rankings                                                                           |
| Sticky Note1               | Sticky Note          | Project overview and detailed description          | None                           | None                                         | ### 🚀 Automated SEO Data Engine using DataForSEO & Airtable( By Sparsh from Automation Jinn automationjinn.com)  This workflow converts keywords + competitor domains → SERP rankings, competitor keyword insights, and similar keyword ideas, all stored in Airtable with zero manual research.  #### 🔁 Flow Summary 1. **Trigger** starts the workflow on demand (or via schedule). 2. **Seed Keywords Fetch** pulls keywords from Airtable. 3. **SERP Tracking** - Posts keyword tasks to **DataForSEO** - Waits + fetches rank results - Stores structured SERP rows in **`SERP rankings`** table 4. **Competitor Research** - Fetches competitor domains from Airtable - Retrieves competitor-site keywords from **DataForSEO** - Saves to **`Competitor Keywords Research`** 5. **Related Keyword Ideas** - Aggregates seed keywords - Calls **DataForSEO** for similar/related keyword suggestions - Stores data in **`Similar Keywords`** table  #### 🎯 Goal Enable continuous SEO tracking and keyword discovery by automatically maintaining a centralised Airtable hub for SERP movement, competitor insights, and keyword expansion without manual effort. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: Start workflow on demand.

2. **Add Airtable Search Node: Search Keywords**  
   - Base: Your Airtable SEO base (e.g., appATppKMQDpZLyj7)  
   - Table: SERP Keywords (e.g., tbl9oPLA8Ul8AeU3i)  
   - Operation: Search  
   - Filter Formula: `{Keyword} != "BLANK"`  
   - Credential: Airtable Personal Access Token

3. **Add Airtable Search Node: Search Competitors**  
   - Base: Same SEO base  
   - Table: Competitor Research (e.g., tblh2877pYvZuyIiL)  
   - Operation: Search  
   - Filter Formula: `={Research} = "Yes"`  
   - Credential: Same Airtable token

4. **Add Airtable Search Node: Search Keywords1**  
   - Duplicate of Search Keywords node (same parameters)  
   - Purpose: For keyword aggregation

5. **Connect Manual Trigger outputs to Search Keywords, Search Competitors, and Search Keywords1 nodes**

6. **Add HTTP Request Node: Post Search Rankings**  
   - Method: POST  
   - URL: https://api.dataforseo.com/v3/serp/google/organic/task_post  
   - Authentication: HTTP Basic Auth with DataForSEO credentials  
   - Headers: Content-Type: application/json  
   - Body (JSON):  
     ```json
     [
       {
         "language_code": "en",
         "location_code": 2840,
         "keyword": "{{ $json.Keyword }}",
         "depth": 10
       }
     ]
     ```  
   - Connect input from Search Keywords node

7. **Add Wait Node: Wait**  
   - Wait Time: 1 minute  
   - Connect input from Post Search Rankings node

8. **Add HTTP Request Node: Get Search Rankings**  
   - Method: GET (use HTTP Request with JSON body containing empty string to force GET)  
   - URL: `https://api.dataforseo.com/v3/serp/google/organic/task_get/regular/{{ $('Post Search Rankings').item.json.tasks[0].id }}`  
   - Authentication: HTTP Basic Auth with DataForSEO credentials  
   - Headers: Content-Type: application/json  
   - Connect input from Wait node

9. **Add Split Out Node: Split Out**  
   - Field to split out: `tasks[0].result[0].items`  
   - Connect input from Get Search Rankings

10. **Add Airtable Create Node: Create Search Rankings Records**  
    - Base: SEO base  
    - Table: SERP rankings (e.g., tblpJw96Iy4GTYAAC)  
    - Operation: Create  
    - Map fields: url, page, type, title, domain, breadcrumb, rank_group, description, rank_absolute  
    - Link to original keyword via "SERP Keywords" field with Airtable record ID from Search Keywords node  
    - Credential: Airtable Personal Access Token  
    - Connect input from Split Out node

11. **Add HTTP Request Node: Post Competitor Keywords**  
    - Method: POST  
    - URL: https://api.dataforseo.com/v3/keywords_data/google_ads/keywords_for_site/task_post  
    - Authentication: HTTP Basic Auth with DataForSEO credentials  
    - Headers: Content-Type: application/json  
    - Body (JSON):  
      ```json
      [
        {
          "location_code": 2840,
          "target": "{{ $json.Name }}"
        }
      ]
      ```  
    - Connect input from Search Competitors node

12. **Add Wait Node: Wait1**  
    - Wait Time: 1 minute  
    - Connect input from Post Competitor Keywords

13. **Add HTTP Request Node: Get Competitor Keywords**  
    - Method: GET (similarly with empty JSON body)  
    - URL: `https://api.dataforseo.com/v3/keywords_data/google_ads/keywords_for_site/task_get/{{ $('Post Competitor Keywords').item.json.tasks[0].id }}`  
    - Authentication: HTTP Basic Auth with DataForSEO credentials  
    - Headers: Content-Type: application/json  
    - Connect input from Wait1

14. **Add Split Out Node: Split Out1**  
    - Field to split out: `tasks[0].result`  
    - Connect input from Get Competitor Keywords

15. **Add Airtable Create Node: Create Competitor Keywords Records**  
    - Base: SEO base  
    - Table: Competitor Keywords Research (e.g., tblewLBcS8d673dcE)  
    - Operation: Create  
    - Map multiple fields including CPC, monthly search volumes for various months, competition, bids, and link "Company Name" to competitor Airtable record ID  
    - Credential: Airtable Personal Access Token  
    - Connect input from Split Out1

16. **Add Aggregate Node: Aggregate**  
    - Aggregate by field: Keyword  
    - Connect input from Search Keywords1 node

17. **Add HTTP Request Node: Get Similar Keywords**  
    - Method: POST  
    - URL: https://api.dataforseo.com/v3/keywords_data/google_ads/keywords_for_keywords/live  
    - Authentication: HTTP Basic Auth with DataForSEO credentials  
    - Headers: Content-Type: application/json  
    - Body (JSON):  
      ```json
      [
        {
          "keywords": {{ $json.Keyword }},
          "language_code": "en",
          "location_code": 2840
        }
      ]
      ```  
    - Connect input from Aggregate

18. **Add Split Out Node: Split Out2**  
    - Field to split out: `tasks[0].result`  
    - Connect input from Get Similar Keywords

19. **Add Airtable Create Node: Create Similar Keywords Records**  
    - Base: SEO base  
    - Table: Similar Keywords (e.g., tblmh2gHzTdv24T4r)  
    - Operation: Create  
    - Map fields for CPC, monthly search volumes, competition, bids, etc.  
    - Credential: Airtable Personal Access Token  
    - Connect input from Split Out2

20. **Optional: Add Sticky Notes for documentation and visual grouping**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                              | Context or Link                                         |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------|
| 🚀 Automated SEO Data Engine using DataForSEO & Airtable by Sparsh from Automation Jinn. Converts keywords and competitor domains into SERP rankings, competitor keyword insights, and keyword ideas stored in Airtable with zero manual research. Enables continuous SEO tracking and keyword discovery by maintaining a centralized Airtable hub. | Project credit: Automation Jinn (automationjinn.com)   |
| Flow summary includes trigger, seed keywords fetch, SERP tracking, competitor research, and related keyword ideas generation.                                                                                                                                                                                                                                           | Workflow documentation embedded in Sticky Note1        |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.