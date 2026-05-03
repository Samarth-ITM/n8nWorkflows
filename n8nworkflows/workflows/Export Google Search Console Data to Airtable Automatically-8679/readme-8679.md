Export Google Search Console Data to Airtable Automatically

https://n8nworkflows.xyz/workflows/export-google-search-console-data-to-airtable-automatically-8679


# Export Google Search Console Data to Airtable Automatically

### 1. Workflow Overview

This workflow automates the extraction of Google Search Console data and exports it into Airtable for easy reporting and analysis. It targets users who want to avoid manual CSV exports and data cleaning, such as SEO consultants, marketing teams, and website owners.

**Logical blocks:**

- **1.1 Schedule and Input Setup**  
  Initiates the workflow on a schedule and sets the target domain and date range.

- **1.2 Google Search Console API Data Retrieval**  
  Fetches three types of performance reports (Query, Page, Date) via API calls.

- **1.3 Data Splitting and Transformation**  
  Splits the batch data arrays into individual records and cleans/renames fields for clarity.

- **1.4 Data Export to Airtable**  
  Inserts the cleaned data into three respective Airtable tables for Queries, Pages, and Dates.

---

### 2. Block-by-Block Analysis

#### 1.1 Schedule and Input Setup

- **Overview:**  
  This block triggers the workflow on a regular schedule and sets the domain and time range parameters for the Google Search Console queries.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Set your domain

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Starts workflow execution automatically based on configured intervals (default is every day).  
    - Configuration: Runs on a defined schedule; can be customized to specific times or intervals.  
    - Inputs: None (trigger node)  
    - Outputs: Connects to "Set your domain" node  
    - Edge cases: Misconfiguration may cause missed runs or excessive triggering.

  - **Set your domain**  
    - Type: Set  
    - Role: Defines workflow variables `domain` (e.g., "funautomations.io") and `days` (default 30) that set the website and how far back data is retrieved.  
    - Key expressions: Assigns static values to `domain` and `days` fields.  
    - Inputs: From Schedule Trigger  
    - Outputs: Triggers three HTTP Request nodes for data fetching  
    - Edge cases: Incorrect domain or days values will cause invalid API queries.

---

#### 1.2 Google Search Console API Data Retrieval

- **Overview:**  
  This block queries Google Search Console API for three report types: by Query (keywords), by Page (URLs), and by Date (daily metrics).

- **Nodes Involved:**  
  - Get query Report  
  - Get Page Report  
  - date (HTTP Request node for date report)

- **Node Details:**

  - **Get query Report**  
    - Type: HTTP Request  
    - Role: Fetches search analytics data grouped by query keyword.  
    - Configuration: POST request to `https://www.googleapis.com/webmasters/v3/sites/sc-domain:{{$json.domain}}/searchAnalytics/query`  
      - JSON body includes `startDate`, `endDate`, and `dimensions:["query"]`  
      - Date range dynamically set using current date and `days` parameter.  
      - Auth: Google OAuth2 (credential required)  
    - Inputs: From "Set your domain"  
    - Outputs: Connects to "Split Out" node  
    - Edge cases: OAuth errors, API rate limits, invalid domain, empty responses.

  - **Get Page Report**  
    - Type: HTTP Request  
    - Role: Fetches data grouped by page URLs.  
    - Configuration: Similar to above but with `dimensions:["page"]`.  
    - Inputs: From "Set your domain"  
    - Outputs: Connects to "Split Out1" node  
    - Edge cases: Same as above.

  - **date**  
    - Type: HTTP Request  
    - Role: Fetches data grouped by date to track daily performance.  
    - Configuration: Similar to above but with `dimensions:["date"]`.  
    - Inputs: From "Set your domain"  
    - Outputs: Connects to "Split Out2" node  
    - Edge cases: Same as above.

---

#### 1.3 Data Splitting and Transformation

- **Overview:**  
  This block processes the API responses by splitting the bulk rows into individual items and cleaning/renaming fields for each report type.

- **Nodes Involved:**  
  - Split Out (for Query)  
  - Edit Fields (for Query)  
  - Split Out1 (for Page)  
  - Edit Fields1 (for Page)  
  - Split Out2 (for Date)  
  - Edit Fields2 (for Date)

- **Node Details:**

  - **Split Out, Split Out1, Split Out2**  
    - Type: Split Out  
    - Role: Splits the `rows` array from the API response into individual JSON items for processing.  
    - Configuration: Splits field `rows`  
    - Inputs: From respective HTTP Request nodes  
    - Outputs: To respective "Edit Fields" nodes  
    - Edge cases: If `rows` is missing or empty, downstream nodes receive no data.

  - **Edit Fields, Edit Fields1, Edit Fields2**  
    - Type: Set  
    - Role: Renames and selects key fields for each item to human-readable names and formats.  
    - Configuration:  
      - Query report: rename `keys[0]` to `Keyword`  
      - Page report: rename `keys[0]` to `page`  
      - Date report: rename `keys[0]` to `date`  
      - Retain `clicks`, `impressions`, `ctr`, `position` as numbers  
    - Inputs: From respective Split Out nodes  
    - Outputs: To respective Airtable nodes  
    - Edge cases: Missing keys array or unexpected data structure may cause expression failures.

---

#### 1.4 Data Export to Airtable

- **Overview:**  
  This block saves the cleaned and structured data into Airtable, populating three different tables: Queries, Pages, and Dates.

- **Nodes Involved:**  
  - Create a record (Queries)  
  - Create a record1 (Pages)  
  - Create a record2 (Dates)

- **Node Details:**

  - **Create a record, Create a record1, Create a record2**  
    - Type: Airtable  
    - Role: Create new records in specified Airtable base and tables for each data type.  
    - Configuration:  
      - Base: "Search Console Reports" (app ID `appzCPTyUrpA1fXpf`)  
      - Tables: Queries (`tblK0ZkihGpOc7ADu`), Pages (`tblA6tGocjen8W0iN`), Dates (`tblSJx4WuIm3feGGw`)  
      - Mapping fields:  
        - Queries: Keyword, clicks, impressions, ctr, position  
        - Pages: page, clicks, impressions, ctr, position  
        - Dates: date, clicks, impressions, ctr, position  
      - Operation: Create new record (append)  
      - Credentials: Airtable Personal Access Token  
    - Inputs: From corresponding Edit Fields nodes  
    - Outputs: None (end of flow)  
    - Edge cases: Authentication errors, rate limits, Airtable schema mismatch, network errors.

---

### 3. Summary Table

| Node Name         | Node Type           | Functional Role                        | Input Node(s)       | Output Node(s)       | Sticky Note                                                                                                               |
|-------------------|---------------------|-------------------------------------|---------------------|----------------------|---------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger   | Schedule Trigger    | Initiate workflow on schedule       | None                | Set your domain       | ## Step 1: Schedule the Workflow  \nStarts the workflow automatically to avoid manual triggering.                        |
| Set your domain    | Set                 | Define domain and date range params | Schedule Trigger    | Get query Report, Get Page Report, date | ## Step 2: Set Your Domain and Time Range  \nDefines site and how far back data is pulled.                                 |
| Get query Report   | HTTP Request        | Fetch query-based GSC data          | Set your domain     | Split Out             | ## Step 3: Fetch Data from Google Search Console  \nFetches Query report with specific API parameters.                    |
| Split Out         | Split Out            | Split query data array into items   | Get query Report    | Edit Fields           | ## Step 4: Split the Data  \nSplits rows array into single data entries for processing.                                    |
| Edit Fields       | Set                 | Rename and clean query data fields  | Split Out           | Create a record       | ## Step 5: Clean and Rename Fields  \nRenames keys[0] to Keyword and retains relevant metrics.                             |
| Create a record   | Airtable             | Create records in Queries table     | Edit Fields         | None                  | ## Step 6: Save Everything into Airtable  \nStores query performance data in Airtable.                                    |
| Get Page Report    | HTTP Request        | Fetch page-based GSC data           | Set your domain     | Split Out1            | ## Step 3: Fetch Data from Google Search Console  \nFetches Page report similarly.                                        |
| Split Out1        | Split Out            | Split page data array into items    | Get Page Report     | Edit Fields1          | ## Step 4: Split the Data  \nFor page data splitting.                                                                     |
| Edit Fields1      | Set                 | Rename and clean page data fields   | Split Out1          | Create a record1      | ## Step 5: Clean and Rename Fields  \nRenames keys[0] to page.                                                            |
| Create a record1  | Airtable             | Create records in Pages table       | Edit Fields1        | None                  | ## Step 6: Save Everything into Airtable  \nStores page performance data in Airtable.                                     |
| date              | HTTP Request        | Fetch date-based GSC data           | Set your domain     | Split Out2            | ## Step 3: Fetch Data from Google Search Console  \nFetches Date report grouped by date.                                  |
| Split Out2        | Split Out            | Split date data array into items    | date                | Edit Fields2          | ## Step 4: Split the Data  \nFor date data splitting.                                                                     |
| Edit Fields2      | Set                 | Rename and clean date data fields   | Split Out2          | Create a record2      | ## Step 5: Clean and Rename Fields  \nRenames keys[0] to date.                                                            |
| Create a record2  | Airtable             | Create records in Dates table       | Edit Fields2        | None                  | ## Step 6: Save Everything into Airtable  \nStores date-based performance data in Airtable.                              |
| Sticky Note       | Sticky Note          | Documentation and explanation       | None                | None                  | Contains detailed workflow purpose and audience description.                                                             |
| Sticky Note1      | Sticky Note          | Documentation                      | None                | None                  | Explains schedule trigger usage.                                                                                           |
| Sticky Note2      | Sticky Note          | Documentation                      | None                | None                  | Explains domain and days variable setup.                                                                                   |
| Sticky Note3      | Sticky Note          | Documentation                      | None                | None                  | Details API calls description.                                                                                             |
| Sticky Note4      | Sticky Note          | Documentation                      | None                | None                  | Describes data splitting rationale.                                                                                        |
| Sticky Note5      | Sticky Note          | Documentation                      | None                | None                  | Explains field cleaning and renaming.                                                                                      |
| Sticky Note6      | Sticky Note          | Documentation                      | None                | None                  | Explains Airtable data saving and benefits.                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Set the interval to daily or desired frequency.

2. **Add a Set node ("Set your domain")**  
   - Add two fields:  
     - `domain` (string): e.g., `"funautomations.io"`  
     - `days` (number): e.g., `30`  
   - Connect Schedule Trigger output to this node.

3. **Add three HTTP Request nodes** for fetching GSC data:

   - **Get query Report**  
     - Method: POST  
     - URL: `https://www.googleapis.com/webmasters/v3/sites/sc-domain:{{$json.domain}}/searchAnalytics/query`  
     - Authentication: Google OAuth2 (set up Google credentials with Search Console API access)  
     - Body (JSON):  
       ```json
       {
         "startDate": "{{ $now.minus($json.days, 'days').format('yyyy-MM-dd') }}",
         "endDate": "{{ $now.format('yyyy-MM-dd') }}",
         "dimensions": ["query"],
         "rowLimit": 25000
       }
       ```  
     - Connect "Set your domain" output to this node.

   - **Get Page Report**  
     - Same as above, but `"dimensions": ["page"]`.

   - **date** (Get Date Report)  
     - Same as above, but `"dimensions": ["date"]`.

4. **Add three Split Out nodes** to split the `rows` array from each HTTP Request:

   - One connected to Get query Report, splitting `rows`.
   - One connected to Get Page Report, splitting `rows`.
   - One connected to date node, splitting `rows`.

5. **Add three Set nodes** to rename fields after splitting:

   - For Query report: rename `keys[0]` to `Keyword` and keep `clicks`, `impressions`, `ctr`, `position`.
   - For Page report: rename `keys[0]` to `page` and keep other metrics.
   - For Date report: rename `keys[0]` to `date` and keep other metrics.

6. **Add three Airtable nodes** to create records:

   - **Queries table**  
     - Base: `Search Console Reports`  
     - Table: `Queries`  
     - Map fields: Keyword, clicks, impressions, ctr, position  
     - Use Airtable Personal Access Token credentials.

   - **Pages table**  
     - Base: same  
     - Table: `Pages`  
     - Map fields: page, clicks, impressions, ctr, position  

   - **Dates table**  
     - Base: same  
     - Table: `Dates`  
     - Map fields: date (dateTime), clicks, impressions, ctr, position  

7. **Connect nodes accordingly:**  
   - Schedule Trigger → Set your domain → Get query Report → Split Out → Edit Fields → Create a record  
   - Set your domain → Get Page Report → Split Out1 → Edit Fields1 → Create a record1  
   - Set your domain → date → Split Out2 → Edit Fields2 → Create a record2

8. **Configure credentials:**

   - Set up Google OAuth2 credentials with Search Console API enabled and authorized for your Google account.
   - Set up Airtable Personal Access Token with write permissions to the base and tables.

9. **Test the workflow:**  
   - Run manually to verify data retrieval, transformation, and insertion into Airtable.

---

### 5. General Notes & Resources

| Note Content                                                                                                          | Context or Link                                                         |
|-----------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------|
| This workflow eliminates manual CSV exports from Google Search Console by automating API pulls and Airtable insertion. | Branding and workflow purpose description included in sticky notes.    |
| Requires Google Cloud project with Search Console API enabled and OAuth2 credentials configured in n8n.                | Google Cloud Console: https://console.cloud.google.com/                 |
| Airtable base called "Search Console Reports" with three tables: Queries, Pages, Dates must be pre-created.            | Airtable: https://airtable.com                                          |
| Workflow is fully customizable by changing the domain and days parameters in the "Set your domain" node.               | Allows adapting reporting range and target website easily.              |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow built with n8n, an integration and automation tool. This processing strictly respects applicable content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.