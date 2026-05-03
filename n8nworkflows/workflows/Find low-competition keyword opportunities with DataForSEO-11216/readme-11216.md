Find low-competition keyword opportunities with DataForSEO

https://n8nworkflows.xyz/workflows/find-low-competition-keyword-opportunities-with-dataforseo-11216


# Find low-competition keyword opportunities with DataForSEO

### 1. Workflow Overview

This workflow is designed to identify low-competition keyword opportunities using the DataForSEO API combined with Google Sheets for input and output management. It targets SEO professionals and marketers seeking to discover untapped keywords with low ranking difficulty for specific domains or seed keywords.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Scheduling:**  
  Periodically triggers the workflow and reads seed data (domains or keywords) from a Google Sheet, including parameters such as location, language, and result limits.

- **1.2 Keyword Retrieval & Difficulty Analysis:**  
  For each seed, it fetches related keywords from DataForSEO's `keywords_for_site` endpoint and concurrently retrieves keyword difficulty scores from the `bulk_keyword_difficulty` endpoint.

- **1.3 Data Aggregation and Formatting:**  
  Merges keyword data with difficulty scores, formats the combined dataset with additional metadata (search volume, trends, intent, backlinks), and aggregates all results into a flat structure.

- **1.4 Output Writing:**  
  Writes the processed low-competition keyword data back into a designated Google Sheet for analysis and further use.

- **1.5 Utility - IP Check:**  
  A utility sub-process that fetches the public IP address of the n8n server, useful for debugging or environment verification.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Scheduling

- **Overview:**  
  This block schedules the workflow to run monthly and reads seed entries (domains/keywords) from a Google Sheet, which include details such as location, language, and the number of keywords to retrieve.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Read Seeds  
  - Loop Over Domains  
  - Sticky Note1 (comment)

- **Node Details:**

  - **Schedule Trigger**  
    - *Type:* scheduleTrigger  
    - *Role:* Initiates the workflow on a monthly interval  
    - *Config:* Interval set to trigger once every month  
    - *Inputs:* None (trigger node)  
    - *Outputs:* Passes trigger to "Read Seeds"  
    - *Failures:* None typical; possible misconfiguration of schedule

  - **Read Seeds**  
    - *Type:* Google Sheets node  
    - *Role:* Reads seed data from a specific sheet and document  
    - *Config:* Reads from a named sheet by ID and document URL (credentials required)  
    - *Inputs:* Trigger from schedule  
    - *Outputs:* Passes rows containing seed keywords/domains, location, language, and limit to "Loop Over Domains"  
    - *Failures:* Authentication errors, sheet access issues, empty or malformed data  
    - *Credentials:* Google Sheets OAuth2 required

  - **Loop Over Domains**  
    - *Type:* splitInBatches  
    - *Role:* Iterates over each seed entry individually to process keywords per domain/seed  
    - *Config:* Default batch size (1), processes each domain sequentially  
    - *Inputs:* Seed data array from "Read Seeds"  
    - *Outputs:* Splits data stream to "Write to Sheet" and "Get Keywords" nodes  
    - *Failures:* Empty input arrays, batch misconfiguration

  - **Sticky Note1**  
    - *Type:* stickyNote (comment)  
    - *Content:* Explains that this block loads domain seeds from Google Sheets including location, language, and limit parameters

#### 1.2 Keyword Retrieval & Difficulty Analysis

- **Overview:**  
  This block fetches keywords that a domain ranks for and obtains keyword difficulty scores in parallel, preparing data for merging.

- **Nodes Involved:**  
  - Get Keywords  
  - Get Keyword Difficulty  
  - Merge  
  - Sticky Note2, Sticky Note3 (comments)

- **Node Details:**

  - **Get Keywords**  
    - *Type:* HTTP Request  
    - *Role:* Calls DataForSEO API `keywords_for_site` to retrieve ranked keywords for the domain  
    - *Config:* POST request with JSON body containing seed, location, language, and limit parameters from current item context  
    - *Inputs:* One domain item from "Loop Over Domains"  
    - *Outputs:* JSON response with keywords data to "Get Keyword Difficulty" and then "Merge"  
    - *Retries:* Up to 2 retries on failure, 2 seconds wait between tries  
    - *Failures:* API errors, network issues, invalid parameters, auth problems

  - **Get Keyword Difficulty**  
    - *Type:* HTTP Request  
    - *Role:* Calls DataForSEO API `bulk_keyword_difficulty` to get difficulty scores for keywords obtained above  
    - *Config:* POST request with keywords array, location, and language parameters taken from the current domain context and previous node output  
    - *Inputs:* Keywords data from "Get Keywords"  
    - *Outputs:* Difficulty scores JSON to "Merge"  
    - *Retries:* Same as above  
    - *Failures:* API limits, timeouts, missing keywords in request

  - **Merge**  
    - *Type:* Merge node  
    - *Role:* Combines keyword data and difficulty scores by position (corresponding items) into a single item  
    - *Config:* Mode set to "combine" by position  
    - *Inputs:* Two inputs from "Get Keywords" and "Get Keyword Difficulty"  
    - *Outputs:* Combined data to "Format Data"  
    - *Failures:* Mismatch in array lengths, missing data

  - **Sticky Note2**  
    - *Content:* Explains "Fetch Keywords" step and API usage

  - **Sticky Note3**  
    - *Content:* Explains "Get Difficulty Scores" step and API usage

#### 1.3 Data Aggregation and Formatting

- **Overview:**  
  This block formats the combined keyword and difficulty data into a structured, enriched dataset and aggregates all results across batches.

- **Nodes Involved:**  
  - Format Data  
  - Collect All Results  
  - Flatten Data  
  - Sticky Note5, Sticky Note6 (comments)

- **Node Details:**

  - **Format Data**  
    - *Type:* Code node (JavaScript)  
    - *Role:* Maps and merges keyword info and difficulty into rows with fields like search volume, trends, intent, backlinks, and timestamps  
    - *Config:* Extracts keyword list and difficulty data from inputs; creates objects with specific property mappings for output  
    - *Inputs:* Merged keyword and difficulty data  
    - *Outputs:* Array of formatted JSON objects to "Collect All Results"  
    - *Failures:* Runtime JavaScript errors, missing fields in input JSON

  - **Collect All Results**  
    - *Type:* Aggregate node  
    - *Role:* Aggregates all formatted items into a single collection to handle batch outputs  
    - *Config:* Aggregates all incoming items  
    - *Inputs:* Multiple formatted data items  
    - *Outputs:* Single aggregated array to "Flatten Data"  
    - *Failures:* Large data volumes causing memory issues

  - **Flatten Data**  
    - *Type:* Code node (JavaScript)  
    - *Role:* Flattens nested arrays within aggregated data into a simple flat list suitable for writing  
    - *Config:* Iterates over all input items, extracts `data` arrays, and returns flattened list  
    - *Inputs:* Aggregated results  
    - *Outputs:* Flattened array of keyword rows to "Loop Over Domains" for final writing  
    - *Failures:* Improper data format causing empty or invalid outputs

  - **Sticky Note5**  
    - *Content:* Describes the combine and format process including trends, intent, and backlinks

  - **Sticky Note6**  
    - *Content:* Describes aggregation and flattening preparing data for output

#### 1.4 Output Writing

- **Overview:**  
  This block writes the final low-competition keyword data back into a Google Sheet for review and further analysis.

- **Nodes Involved:**  
  - Write to Sheet  
  - Loop Over Domains (output branch)  

- **Node Details:**

  - **Write to Sheet**  
    - *Type:* Google Sheets node  
    - *Role:* Appends rows of keyword data to a pre-defined Google Sheet tab  
    - *Config:* Columns mapped explicitly to fields such as keywords, search volume, keyword difficulty, trends, backlink counts, intents, and timestamps; append operation  
    - *Inputs:* Individual formatted keyword rows from "Loop Over Domains"  
    - *Outputs:* None (end node)  
    - *Failures:* Authentication errors, sheet access errors, quota limits

#### 1.5 Utility - IP Check

- **Overview:**  
  A utility block to check and log the public IP address of the n8n server instance.

- **Nodes Involved:**  
  - Find n8n's IP  
  - Sticky Note  

- **Node Details:**

  - **Find n8n's IP**  
    - *Type:* HTTP Request  
    - *Role:* Calls https://api.ipify.org to retrieve the server's public IP in JSON format  
    - *Config:* Simple GET request without additional options  
    - *Inputs:* None (manual or test trigger)  
    - *Outputs:* IP address JSON  
    - *Failures:* Network issues, API downtime

  - **Sticky Note**  
    - *Content:* "Run to find your n8n's server IP" - a helper comment

---

### 3. Summary Table

| Node Name           | Node Type           | Functional Role                             | Input Node(s)           | Output Node(s)                 | Sticky Note                                                                                   |
|---------------------|---------------------|--------------------------------------------|-------------------------|-------------------------------|----------------------------------------------------------------------------------------------|
| Schedule Trigger     | scheduleTrigger     | Periodic workflow start                     | None                    | Read Seeds                    |                                                                                              |
| Read Seeds          | Google Sheets       | Reads seed domain/keyword data              | Schedule Trigger        | Loop Over Domains             | **Load Domain Seeds** Reads domains from sheet and includes location, language, and limit parameters |
| Loop Over Domains   | splitInBatches      | Iterates over each seed/domain              | Read Seeds              | Write to Sheet, Get Keywords  |                                                                                              |
| Get Keywords        | HTTP Request        | Fetches keywords ranked for domain          | Loop Over Domains       | Get Keyword Difficulty, Merge | **Fetch Keywords** API: keywords_for_site Returns all keywords the domain ranks for Limited by "limit" parameter from sheet |
| Get Keyword Difficulty | HTTP Request      | Fetches keyword difficulty scores           | Get Keywords            | Merge                        | **Get Difficulty Scores** API: bulk_keyword_difficulty Checks ranking difficulty (0-100) for all keywords Runs in parallel with keyword fetch |
| Merge               | Merge               | Combines keyword data with difficulty scores| Get Keywords, Get Keyword Difficulty | Format Data            | **Combine & Format** Merges keyword data with difficulty scores Formats for Google Sheets output Includes trends, intent, backlinks |
| Format Data         | Code                | Formats and enriches keyword data            | Merge                   | Collect All Results           |                                                                                              |
| Collect All Results | Aggregate           | Aggregates all formatted keyword rows       | Format Data              | Flatten Data                 | **Aggregate Results** Collects all formatted rows Flattens nested data structure Prepares final dataset for writing |
| Flatten Data        | Code                | Flattens nested aggregated data              | Collect All Results      | Loop Over Domains            |                                                                                              |
| Write to Sheet      | Google Sheets       | Writes final keyword data to Google Sheet   | Loop Over Domains       | None                         |                                                                                              |
| Find n8n's IP       | HTTP Request        | Retrieves public IP of n8n server            | None                    | None                         | **Run to find your n8n's server IP**                                                        |
| Sticky Note         | Sticky Note         | Comments and explanations                     | None                    | None                         | See associated notes above for each sticky note content                                     |
| Sticky Note1        | Sticky Note         | Comments on loading domain seeds              | None                    | None                         | **Load Domain Seeds** Reads domains from sheet and includes location, language, and limit parameters |
| Sticky Note2        | Sticky Note         | Comments on fetching keywords                  | None                    | None                         | **Fetch Keywords** API: keywords_for_site Returns all keywords the domain ranks for Limited by "limit" parameter from sheet |
| Sticky Note3        | Sticky Note         | Comments on getting difficulty scores          | None                    | None                         | **Get Difficulty Scores** API: bulk_keyword_difficulty Checks ranking difficulty (0-100) for all keywords Runs in parallel with keyword fetch |
| Sticky Note4        | Sticky Note         | High-level overview and setup instructions    | None                    | None                         | **Low competition keyword finder** Explains workflow purpose, steps, and setup requirements |
| Sticky Note5        | Sticky Note         | Comments on combining and formatting data      | None                    | None                         | **Combine & Format** Merges keyword data with difficulty scores Formats for Google Sheets output Includes trends, intent, backlinks |
| Sticky Note6        | Sticky Note         | Comments on aggregation and flattening         | None                    | None                         | **Aggregate Results** Collects all formatted rows Flattens nested data structure Prepares final dataset for writing |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Set to trigger monthly (interval on months field)  
   - This will start the workflow automatically once per month.

2. **Add a Google Sheets node ("Read Seeds")**  
   - Operation: Read rows from sheet  
   - Credentials: Configure with Google Sheets OAuth2 (ensure access to your Google Sheets)  
   - Document: Use your Google Sheet URL or ID containing seed data  
   - Sheet Name: Use the sheet/tab with your seeds (domains/keywords + location, language, limit)  
   - Output: This node outputs an array of seed entries.

3. **Add a SplitInBatches node ("Loop Over Domains")**  
   - Connect "Read Seeds" output to this node's input  
   - Set batch size 1 to process each seed individually.

4. **Add an HTTP Request node ("Get Keywords")**  
   - Connect from "Loop Over Domains"  
   - Method: POST  
   - URL: `https://api.dataforseo.com/v3/dataforseo_labs/google/keywords_for_site/live`  
   - Body Format: JSON  
   - Body Content (use expression mode):  
     ```json
     [{
       "target": "{{ $json.seed }}",
       "location_name": "{{ $json.location_name }}",
       "language_name": "{{ $json.language_name }}",
       "limit": {{ $json.limit }}
     }]
     ```  
   - Set retries: 2 tries with 2000 ms wait between tries  
   - Credentials: Provide DataForSEO API credentials for authentication.

5. **Add a second HTTP Request node ("Get Keyword Difficulty")**  
   - Connect from "Get Keywords" node output  
   - Method: POST  
   - URL: `https://api.dataforseo.com/v3/dataforseo_labs/google/bulk_keyword_difficulty/live`  
   - Body Format: JSON  
   - Body Content (expression):  
     ```json
     [
       {
         "keywords": $json.tasks[0].result[0].items.map(i => i.keyword),
         "location_name": "{{ $json.location_name }}",
         "language_name": "{{ $json.language_name }}"
       }
     ]
     ```  
   - Set retries and wait same as previous node  
   - Credentials: DataForSEO API credentials.

6. **Add a Merge node ("Merge")**  
   - Connect the first input to "Get Keywords" output  
   - Connect the second input to "Get Keyword Difficulty" output  
   - Mode: Combine by position (to merge corresponding items)  

7. **Add a Code node ("Format Data")**  
   - Connect from "Merge" node output  
   - Paste the JavaScript code that maps combined keyword and difficulty data into structured rows:  
     ```javascript
     const keywordsData = $input.first().json;
     const difficultyData = $input.last().json;

     const keywords = keywordsData.tasks[0].result[0].items;
     const difficulties = difficultyData.tasks[0].result[0].items;

     const difficultyMap = {};
     difficulties.forEach(item => {
       difficultyMap[item.keyword] = item.keyword_difficulty;
     });

     const rows = keywords.map(kw => ({
       seed: keywordsData.tasks[0].data.target,
       keywords: kw.keyword,
       'Search Volume': kw.keyword_info.search_volume || 0,
       'Trend Monthly': kw.keyword_info.search_volume_trend?.monthly ?? null,
       'Trend Quarterly': kw.keyword_info.search_volume_trend?.quarterly ?? null,
       'Trend Yearly': kw.keyword_info.search_volume_trend?.yearly ?? null,
       'SE Type': kw.se_type,
       'Main Intent': kw.search_intent_info?.main_intent || '',
       'Foreign Intent': kw.search_intent_info?.foreign_intent?.join(', ') || '',
       'Last Updated Time': kw.keyword_info.last_updated_time,
       'Backlinks': kw.avg_backlinks_info?.backlinks || 0,
       'Keyword Difficulty': difficultyMap[kw.keyword] ?? null
     }));

     return rows.map(row => ({ json: row }));
     ```  
   - This formats data for Google Sheets.

8. **Add an Aggregate node ("Collect All Results")**  
   - Connect from "Format Data"  
   - Operation: Aggregate all items into a single batch (aggregateAllItemData)

9. **Add another Code node ("Flatten Data")**  
   - Connect from "Collect All Results"  
   - Use code to flatten nested arrays:  
     ```javascript
     const allRows = [];

     for (const item of $input.all()) {
       if (item.json.data && Array.isArray(item.json.data)) {
         allRows.push(...item.json.data);
       }
     }

     return allRows.map(row => ({ json: row }));
     ```

10. **Connect the output of "Flatten Data" back to "Loop Over Domains"**  
    - This completes the batch processing flow.

11. **Add a Google Sheets node ("Write to Sheet")**  
    - Connect from "Loop Over Domains" output branch  
    - Operation: Append rows  
    - Configure columns with explicit mapping for keyword data fields: keywords, Search Volume, Trend Monthly, Trend Quarterly, Trend Yearly, SE Type, Main Intent, Foreign Intent, Last Updated Time, Backlinks, Keyword Difficulty  
    - Document & Sheet ID: Target your output sheet and tab  
    - Credentials: Google Sheets OAuth2

12. **Optional Utility Setup - Find n8n’s IP**  
    - Add an HTTP Request node  
    - Method: GET  
    - URL: https://api.ipify.org?format=jso  
    - Use for debugging or environment verification

---

### 5. General Notes & Resources

| Note Content                                                                                                                           | Context or Link                                                                                                             |
|----------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------|
| Workflow requires DataForSEO API account with API access credentials.                                                                   | https://dataforseo.com/                                                                                                      |
| Google Sheets template recommended to duplicate for input seeds and output results:                                                     | https://docs.google.com/spreadsheets/d/13ioeuFckLX4qEesbJwQ4C04I0-TPppdMoJVKEAPXCSI/edit?usp=sharing                         |
| Keyword Difficulty (KD) scores range from 0 to 100; keywords with KD < 30 considered low competition.                                  | Workflow filters focus on such keywords to identify opportunities                                                           |
| This workflow runs monthly by default but can be adjusted in the Schedule Trigger node.                                                 |                                                                                                                             |
| Sticky notes within the workflow provide detailed comments explaining each functional block and usage instructions.                   |                                                                                                                             |
| The workflow merges and formats multiple data points per keyword including search volume trends, backlink counts, and search intent.  | Enables comprehensive SEO keyword research analysis                                                                          |
| Retry logic with wait time is configured on API calls to handle transient failures or rate limits gracefully.                         |                                                                                                                             |

---

**Disclaimer:**  
The text provided is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.