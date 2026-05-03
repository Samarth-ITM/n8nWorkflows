Twitter Keyword & Public-Figure Monitor with Automated Airtable Archiving

https://n8nworkflows.xyz/workflows/twitter-keyword---public-figure-monitor-with-automated-airtable-archiving-5006


# Twitter Keyword & Public-Figure Monitor with Automated Airtable Archiving

### 1. Workflow Overview

This n8n workflow automates the daily monitoring and archiving of Twitter data related to the keyword "Narendra Modi." It fetches the latest tweets matching the keyword, processes and formats the data, compares it against existing records in an Airtable base to prevent duplication, and appends only new tweets to Airtable for archival. This automated pipeline runs daily at 8 AM and is designed for users such as political analysts, researchers, journalists, and archivists who require fresh, deduplicated tweet data for analysis or record-keeping.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger:** Initiates the workflow at a fixed daily time (8 AM).
- **1.2 Twitter Data Retrieval:** Searches Twitter for recent tweets matching the keyword.
- **1.3 Airtable Existing Records Fetch:** Retrieves current tweet records from Airtable.
- **1.4 Data Formatting:** Prepares Twitter and Airtable data into a consistent structure for comparison.
- **1.5 Duplicate Filtering:** Compares new tweets with existing Airtable entries and filters out duplicates.
- **1.6 Data Archiving:** Appends only the new, unique tweets to the Airtable base.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger

- **Overview:**  
  This block triggers the workflow execution automatically every day at 8 AM.

- **Nodes Involved:**  
  - `8 AM`

- **Node Details:**  
  - **Node:** `8 AM`  
  - **Type:** Schedule Trigger  
  - **Configuration:** Configured to trigger once daily at 8:00 AM local time.  
  - **Expressions/Variables:** None  
  - **Inputs:** None (start node)  
  - **Outputs:** Connects to `Twitter` node  
  - **Version Requirements:** Requires n8n version supporting Schedule Trigger v1.2 or higher for time-based triggers.  
  - **Potential Failures:** Misconfiguration of time zone or trigger time could cause incorrect trigger timing.  
  - **Sub-workflow:** None  

#### 2.2 Twitter Data Retrieval

- **Overview:**  
  This block queries Twitter API to fetch the latest 100 tweets containing the keyword "Narendra Modi," collecting tweet text, likes, IDs, URLs, authorship, and timestamps.

- **Nodes Involved:**  
  - `Twitter`  
  - `set twitter data`

- **Node Details:**  
  - **Node:** `Twitter`  
    - **Type:** Twitter node (API integration)  
    - **Operation:** Search tweets  
    - **Parameters:**  
      - Search text: `"Narendra Modi"`  
      - Result type: Mixed (includes popular and recent tweets)  
      - Limit: 100 tweets  
    - **Inputs:** From `8 AM`  
    - **Outputs:** To `set twitter data`  
    - **Version:** Twitter API v1 node  
    - **Potential Failures:**  
      - Twitter API rate limits or authentication errors  
      - Network timeouts or connectivity issues  
      - Empty or malformed API response  
  - **Node:** `set twitter data`  
    - **Type:** Set node (data transformation)  
    - **Purpose:** Structures Twitter API response fields into a simplified format for comparison and Airtable compatibility.  
    - **Configured Fields:**  
      - Likes: Favorite count from tweet  
      - Tweet: Tweet text  
      - Tweet_id: Tweet unique ID  
      - Tweet URL: Constructed URL to the tweet using screen name and tweet ID string  
      - Author: `in_reply_to_screen_name` field (note: this is the username the tweet is replying to, may be null)  
      - Time: Tweet creation timestamp  
    - **Inputs:** From `Twitter` node  
    - **Outputs:** To `Leave only new tweets` node  
    - **Potential Failures:**  
      - Missing or null API fields causing expression errors  
      - Incorrect URL construction if user data missing or malformed  

#### 2.3 Airtable Existing Records Fetch

- **Overview:**  
  Retrieves all existing tweet records from the specified Airtable table to enable duplicate detection.

- **Nodes Involved:**  
  - `get airtable list`  
  - `Set_AT_list`

- **Node Details:**  
  - **Node:** `get airtable list`  
    - **Type:** Airtable node  
    - **Operation:** List records  
    - **Parameters:**  
      - Table: `tbl6rexxFBodzKVoC` (specific Airtable table)  
      - Application ID: `app36P08S3Jzki6qJ`  
      - No filters; retrieves all records  
    - **Inputs:** From `Twitter` node (runs in parallel after Twitter fetch)  
    - **Outputs:** To `Set_AT_list`  
    - **Potential Failures:**  
      - Airtable API authentication issues  
      - Rate limiting or network errors  
      - Large data sets may cause pagination or timeout issues  
  - **Node:** `Set_AT_list`  
    - **Type:** Set node  
    - **Purpose:** Extracts and formats relevant fields from Airtable records into a consistent structure matching the Twitter data format for comparison.  
    - **Configured Fields:**  
      - Likes (default 0 if undefined)  
      - Tweet, Tweet_id, Tweet URL, Author, Time (all from Airtable fields)  
    - **Inputs:** From `get airtable list`  
    - **Outputs:** To `Leave only new tweets` node (secondary input)  
    - **Potential Failures:**  
      - Missing fields in Airtable records causing expression failures  
      - Null or undefined field values  

#### 2.4 Duplicate Filtering

- **Overview:**  
  Compares newly fetched tweets and existing Airtable records by Tweet ID and filters out any tweets already archived.

- **Nodes Involved:**  
  - `Leave only new tweets`

- **Node Details:**  
  - **Node:** `Leave only new tweets`  
    - **Type:** Merge node  
    - **Operation:** Remove key matches (`Tweet_id`) between two data streams  
    - **Inputs:**  
      - Primary input: Twitter data from `set twitter data`  
      - Secondary input: Airtable data from `Set_AT_list`  
    - **Outputs:** To `Append new tweets to airtable`  
    - **Potential Failures:**  
      - Mismatch in field names or data types causing failed comparisons  
      - If Tweet IDs are missing or malformed, duplicates may not be filtered correctly  
      - Large data sets may affect performance  

#### 2.5 Data Archiving

- **Overview:**  
  Appends only the filtered new tweets into the Airtable base, ensuring the archive remains up-to-date without duplicates.

- **Nodes Involved:**  
  - `Append new tweets to airtable`

- **Node Details:**  
  - **Node:** `Append new tweets to airtable`  
    - **Type:** Airtable node  
    - **Operation:** Append record(s)  
    - **Parameters:**  
      - Table: `tbl6rexxFBodzKVoC`  
      - Application ID: `app36P08S3Jzki6qJ`  
      - Append all fields from incoming data  
    - **Inputs:** From `Leave only new tweets`  
    - **Outputs:** None (terminal node)  
    - **Potential Failures:**  
      - Airtable API authentication or rate limiting errors  
      - Validation errors if incoming data fields do not match Airtable schema  
      - Network or API timeouts  

#### 2.6 Documentation and Metadata

- **Overview:**  
  Provides a comprehensive sticky note with workflow description, usage, and instructions.

- **Nodes Involved:**  
  - `Sticky Note`

- **Node Details:**  
  - **Node:** `Sticky Note`  
    - **Type:** Sticky Note (documentation)  
    - **Content:** Markdown-formatted explanation of workflow purpose, daily automation schedule, stepwise processing, key features, and suggested use cases.  
    - **Position:** Visually detached; does not affect execution  
    - **Potential Failures:** None  

---

### 3. Summary Table

| Node Name              | Node Type            | Functional Role                      | Input Node(s)          | Output Node(s)          | Sticky Note                                                                                                                                                                                                                                                                                                                                                                  |
|------------------------|----------------------|------------------------------------|------------------------|-------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 8 AM                   | Schedule Trigger     | Initiates workflow daily at 8 AM   | None                   | Twitter                 | # 🐦 Twitter to Airtable Archiver<br>Automatically archive fresh tweets about "Narendra Modi" to Airtable, eliminating duplicates.<br>Daily at 8 AM, searches latest 100 tweets containing "Narendra Modi".                                                                                                                                                                     |
| Twitter                | Twitter API          | Fetches latest tweets with keyword | 8 AM                   | set twitter data        | See above                                                                                                                                                                                                                                                                                                                                                                    |
| set twitter data       | Set                  | Formats Twitter data for Airtable  | Twitter                | Leave only new tweets    | See above                                                                                                                                                                                                                                                                                                                                                                    |
| get airtable list      | Airtable             | Fetch existing Airtable records    | Twitter (parallel)     | Set_AT_list             | See above                                                                                                                                                                                                                                                                                                                                                                    |
| Set_AT_list            | Set                  | Formats Airtable data for compare  | get airtable list      | Leave only new tweets    | See above                                                                                                                                                                                                                                                                                                                                                                    |
| Leave only new tweets  | Merge                | Filters out tweets already archived| set twitter data, Set_AT_list | Append new tweets to airtable | See above                                                                                                                                                                                                                                                                                                                                                                    |
| Append new tweets to airtable | Airtable         | Appends new unique tweets to base  | Leave only new tweets  | None                    | See above                                                                                                                                                                                                                                                                                                                                                                    |
| Sticky Note            | Sticky Note          | Documentation & explanation        | None                   | None                    | # 🐦 Twitter to Airtable Archiver<br>Automatically archive fresh tweets about "Narendra Modi" to Airtable, eliminating duplicates.<br>Daily 8 AM run, fetches 100 tweets, processes, filters duplicates, archives new tweets.<br>Perfect for analysts, researchers, journalists, archivists.<br>Pro Tip: Change search term to monitor any topic. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node:**  
   - Add a **Schedule Trigger** node named `8 AM`.  
   - Configure to trigger daily at `8:00` AM local time.  
   - No credentials required.

2. **Add Twitter Node:**  
   - Add a **Twitter** node named `Twitter`.  
   - Set operation to `Search`.  
   - Set `Search Text` to `"Narendra Modi"`.  
   - Set `Result Type` to `Mixed`.  
   - Set `Limit` to 100.  
   - Connect input from `8 AM`.  
   - Configure Twitter API credentials (OAuth1 or Bearer Token as per your setup).

3. **Add Airtable List Node:**  
   - Add an **Airtable** node named `get airtable list`.  
   - Set operation to `List`.  
   - Set Application to your Airtable app (e.g., `app36P08S3Jzki6qJ`).  
   - Set Table to your target table (e.g., `tbl6rexxFBodzKVoC`).  
   - Connect input from `Twitter` node (parallel branch).  
   - Configure Airtable API credentials.

4. **Add Set Node for Twitter Data Formatting:**  
   - Add a **Set** node named `set twitter data`.  
   - Configure fields:  
     - Number: `Likes` = `{{$node["Twitter"].json["favorite_count"]}}`  
     - String:  
       - `Tweet` = `{{$node["Twitter"].json["text"]}}`  
       - `Tweet_id` = `{{$node["Twitter"].json["id"]}}`  
       - `Tweet URL` = `https://twitter.com/{{$node["Twitter"].json["user"]["screen_name"]}}/status/{{$node["Twitter"].json["id_str"]}}`  
       - `Author` = `{{$node["Twitter"].json["in_reply_to_screen_name"]}}` (may be null)  
       - `Time` = `{{$node["Twitter"].json["created_at"]}}`  
   - Connect input from `Twitter`.

5. **Add Set Node for Airtable Data Formatting:**  
   - Add a **Set** node named `Set_AT_list`.  
   - Configure fields:  
     - Number: `Likes` = `{{$node["get airtable list"].json["fields"]["Likes"] ? $node["get airtable list"].json["fields"]["Likes"] : 0}}`  
     - String:  
       - `Tweet` = `{{$node["get airtable list"].json["fields"]["Tweet"]}}`  
       - `Tweet_id` = `{{$node["get airtable list"].json["fields"]["Tweet_id"]}}`  
       - `Tweet URL` = `{{$node["get airtable list"].json["fields"]["Tweet URL"]}}`  
       - `Author` = `{{$node["get airtable list"].json["fields"]["Author"]}}`  
       - `Time` = `{{$node["get airtable list"].json["fields"]["Time"]}}`  
   - Connect input from `get airtable list`.

6. **Add Merge Node for Duplicate Filtering:**  
   - Add a **Merge** node named `Leave only new tweets`.  
   - Set mode to `Remove Key Matches`.  
   - Set Property Name 1 to `Tweet_id` (from Twitter data).  
   - Set Property Name 2 to `Tweet_id` (from Airtable data).  
   - Connect primary input from `set twitter data`.  
   - Connect secondary input from `Set_AT_list`.

7. **Add Airtable Append Node:**  
   - Add an **Airtable** node named `Append new tweets to airtable`.  
   - Set operation to `Append`.  
   - Set Application and Table same as in step 3.  
   - Set to append all fields from incoming data.  
   - Connect input from `Leave only new tweets`.

8. **Add Sticky Note:**  
   - Add a **Sticky Note** with the workflow description and usage instructions as provided in the documentation section for ease of understanding and future reference.

9. **Verify and Activate Workflow:**  
   - Ensure all credentials are configured and valid (Twitter and Airtable).  
   - Test run manually or wait for scheduled trigger at 8 AM.  
   - Monitor for errors or rate limit issues.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                      | Context or Link                                                                                                                                    |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------|
| Pro Tip: Change the Twitter search term to monitor any topic such as other politicians, brands, or trending subjects.                                                                                                                                                                                                              | Sticky Note content                                                                                                                                 |
| The workflow preserves full tweet metadata including likes, timestamp, author, and URL for comprehensive archival and analysis.                                                                                                                                                                                                   | Sticky Note content                                                                                                                                 |
| Recommended to monitor Twitter API rate limits and Airtable API usage quotas to avoid failures during high volume data fetch or append operations.                                                                                                                                                                                  | Best practices for API usage                                                                                                                      |
| Workflow structure supports easy modifications for alternative data sources or additional processing (e.g., sentiment analysis) by inserting nodes after Twitter data retrieval and before Airtable append.                                                                                                                        | Architectural flexibility                                                                                                                         |
| Airtable table schema must include fields: Tweet, Tweet_id, Tweet URL, Author, Time, Likes to match the Set nodes' field mapping.                                                                                                                                                                                                   | Airtable setup requirement                                                                                                                        |
| Twitter API requires valid credentials with read access; consider using OAuth or Bearer token with appropriate scopes.                                                                                                                                                                                                            | Twitter API documentation: https://developer.twitter.com/en/docs/twitter-api                                                                              |
| Airtable API key or OAuth credentials must have write access to the specified base and table.                                                                                                                                                                                                                                       | Airtable API documentation: https://airtable.com/api                                                                                              |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, a no-code integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.