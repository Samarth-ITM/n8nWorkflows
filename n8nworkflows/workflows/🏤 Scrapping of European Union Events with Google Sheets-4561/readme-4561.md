🏤 Scrapping of European Union Events with Google Sheets

https://n8nworkflows.xyz/workflows/---scrapping-of-european-union-events-with-google-sheets-4561


# 🏤 Scrapping of European Union Events with Google Sheets

### 1. Workflow Overview

This workflow automates the scraping, processing, and storage of European Union event data into a Google Sheet. It is designed to run on a scheduled basis, collecting event details from the EU website, parsing relevant information, checking for duplicates against existing records, and appending new events to a Google Sheet to maintain an up-to-date dataset.

The workflow is logically divided into the following blocks:

- **1.1 Workflow Trigger and Initialization:** Scheduled trigger starts the workflow daily and initializes global static data variables to manage pagination and event storage.

- **1.2 Scraping and Parsing EU Event Data:** Iteratively fetches event pages from the EU website, extracts HTML blocks containing event details, parses specific fields, and collects structured event information.

- **1.3 Loading and Aggregating Existing Google Sheet Records:** Loads previously stored events from a Google Sheet and aggregates them for comparison.

- **1.4 Deduplication and Conditional Storage:** Compares newly scraped events against existing ones to avoid duplicates, then appends only new events to the Google Sheet.

- **1.5 Pagination Control and Delay:** Manages pagination by incrementing page counters, enforces a limit on pages to scrape, and includes waiting periods between requests to prevent server overload.

---

### 2. Block-by-Block Analysis

#### 2.1 Workflow Trigger and Initialization

**Overview:**  
This block starts the workflow every morning at 08:00 (local time), initializes global static data variables to manage the current page number and store scraped results persistently throughout the workflow execution.

**Nodes Involved:**  
- Schedule Trigger  
- Initiate Static Data

**Node Details:**

- **Schedule Trigger**  
  - *Type:* Schedule Trigger  
  - *Role:* Starts the workflow daily at a fixed hour.  
  - *Configuration:* Set to trigger at 8 AM local time.  
  - *Connections:* Outputs to "page+1", "Initiate Static Data", "Load Old Records", and "Return Lines Scrapped" nodes.  
  - *Failure Modes:* None typical; network or system downtime might delay execution.

- **Initiate Static Data**  
  - *Type:* Code  
  - *Role:* Initializes global variables for page counting (`page`) and event storage (`results`).  
  - *Configuration:* Sets `page` to -1 and `results` to an empty array in workflow static global data.  
  - *Key Expression:* Uses `$getWorkflowStaticData('global')` to persist data across executions.  
  - *Input:* Triggered by Schedule Trigger.  
  - *Output:* Returns the initialized static data object.  
  - *Edge Cases:* Should only run once per workflow session; re-running resets stored data.  
  - *Notes:* Initialization is required once regardless of Telegram chat IDs (from sticky notes context).

---

#### 2.2 Scraping and Parsing EU Event Data

**Overview:**  
This block increments the page counter, makes HTTP requests to fetch the event pages, extracts HTML blocks representing individual events, and parses event details such as name, link, date, type, and location.

**Nodes Involved:**  
- page+1  
- Query EU Website  
- Extract Blocks  
- Parse Information  
- Collect Fields  
- Store Tables  
- If  
- 15 sec (Wait)

**Node Details:**

- **page+1**  
  - *Type:* Code  
  - *Role:* Increments the static `page` counter by 1 to paginate through event pages.  
  - *Configuration:* Reads current `page` from static data, increments by one, returns the new page number.  
  - *Input:* Triggered by Schedule Trigger or after the 15 sec wait node.  
  - *Output:* Provides the current page number for HTTP requests.  
  - *Edge Cases:* Handles initial case where `page` may be undefined.

- **Query EU Website**  
  - *Type:* HTTP Request  
  - *Role:* Fetches the HTML content of the EU events page for the current page number.  
  - *Configuration:* URL parameterized with page number (e.g., `https://european-union.europa.eu/news-and-events/events_en?page={{ $json.page }}`).  
  - *Input:* From "page+1".  
  - *Output:* Returns raw HTML content.  
  - *Failure Modes:* HTTP errors, timeouts, or website structure changes.

- **Extract Blocks**  
  - *Type:* HTML Extract  
  - *Role:* Extracts individual event blocks (HTML articles) using CSS selector `article.ecl-content-item`.  
  - *Configuration:* Returns an array of HTML snippets.  
  - *Input:* HTML content from HTTP request.  
  - *Output:* Emits array of event blocks for parsing.

- **Parse Information**  
  - *Type:* HTML Extract  
  - *Role:* Parses each event block to extract fields: event name, link, day, month, year, event type, and location.  
  - *Configuration:* Uses multiple CSS selectors and attributes to extract data.  
  - *Input:* Event blocks from Extract Blocks.  
  - *Output:* Structured JSON with event details, including fallback for month field from two selectors.  
  - *Edge Cases:* Missing or malformed HTML elements might lead to empty or incorrect fields.

- **Collect Fields**  
  - *Type:* Set  
  - *Role:* Normalizes and consolidates parsed fields into a consistent schema.  
  - *Configuration:* Maps extracted fields (`event_name`, `event_link`, `day`, `month`, `year`, `event_type`, `event_location`) with logic to handle alternative month values (`month_1 || month_2`).  
  - *Input:* Parsed JSON from Parse Information.  
  - *Output:* Structured event data ready for storage and comparison.

- **Store Tables**  
  - *Type:* Code  
  - *Role:* Appends newly scraped event data to the global static `results` array for temporary storage during the workflow run.  
  - *Configuration:* Merges current input events into `results` array in static data, returning counts of added and total stored events.  
  - *Input:* From Collect Fields.  
  - *Output:* Statistics on events added this round.  
  - *Edge Cases:* Ensures `results` is always an array, preventing type errors.

- **If**  
  - *Type:* Conditional  
  - *Role:* Controls pagination limit by checking if current page number exceeds 3 (default max pages = 4).  
  - *Configuration:* Condition: `page > 3`.  
  - *Input:* From Collect Fields.  
  - *Output:*  
    - True branch: stops pagination (no further scraping).  
    - False branch: continues to wait node and loops.  
  - *Notes:* Configurable page limit to avoid excessive scraping.

- **15 sec**  
  - *Type:* Wait  
  - *Role:* Introduces a 15-second delay between page requests to avoid server overload.  
  - *Configuration:* Waits 15 seconds before triggering next page increment.  
  - *Input:* False branch of If node.  
  - *Output:* Loops back to "page+1".  
  - *Notes:* Helps with respectful scraping and rate limiting.

---

#### 2.3 Loading and Aggregating Existing Google Sheet Records

**Overview:**  
Loads previously stored event records from the Google Sheet, then aggregates event names to enable duplicate detection.

**Nodes Involved:**  
- Load Old Records  
- Aggregate

**Node Details:**

- **Load Old Records**  
  - *Type:* Google Sheets  
  - *Role:* Reads existing event records from the configured Google Sheet spreadsheet and sheet tab.  
  - *Configuration:* Requires Google Sheets OAuth2 credentials, document ID, and sheet name (gid=0).  
  - *Input:* Triggered at workflow start (Schedule Trigger).  
  - *Output:* Returns rows from the sheet as JSON objects representing stored events.  
  - *Failure Modes:* Authentication failure, invalid sheet ID, network issues.

- **Aggregate**  
  - *Type:* Aggregate  
  - *Role:* Aggregates event names (`event_name`) into an array field named `events`.  
  - *Configuration:* Aggregates field `event_name` and renames output field to `events`.  
  - *Input:* Output from Load Old Records.  
  - *Output:* Aggregated list of existing event names for duplicate checking.

---

#### 2.4 Deduplication and Conditional Storage

**Overview:**  
Combines newly scraped events with old records, checks if each new event already exists, and appends only unique events to the Google Sheet.

**Nodes Involved:**  
- Return Lines Scrapped  
- Combine New + Old Records  
- Events Already Existing?  
- Store New Records

**Node Details:**

- **Return Lines Scrapped**  
  - *Type:* Code  
  - *Role:* Retrieves the stored scraped event results from global static data and returns them as workflow output.  
  - *Input:* Triggered by Schedule Trigger.  
  - *Output:* Array of stored event data.  
  - *Edge Cases:* Returns empty array if no results exist.

- **Combine New + Old Records**  
  - *Type:* Merge  
  - *Role:* Combines the newly scraped events with the aggregated old records based on SQL-style merging to prepare for duplicate checking.  
  - *Configuration:* Mode "combineBySql" which performs SQL-like joins.  
  - *Input:*  
    - Input 1: Output of Aggregate (existing events) or Return Lines Scrapped (new events).  
  - *Output:* Combined dataset for comparison.

- **Events Already Existing?**  
  - *Type:* If  
  - *Role:* Checks whether each new event's name exists within the aggregated old event names.  
  - *Configuration:* Condition checks if array `events` (old event names) contains the current `event_name`.  
  - *Input:* From Combine New + Old Records.  
  - *Output:*  
    - True branch: Event exists, do not store again.  
    - False branch: Event is new, proceed to storage.  
  - *Edge Cases:* Case sensitivity and string matching issues could cause false negatives/positives.

- **Store New Records**  
  - *Type:* Google Sheets  
  - *Role:* Appends new unique events to the Google Sheet.  
  - *Configuration:*  
    - Requires Google Sheets OAuth2 credentials.  
    - Operation: Append.  
    - Fields mapped: `day`, `year`, `month`, `event_link`, `event_name`, `event_type`, `event_location`.  
    - Sheet tab: gid=0.  
  - *Input:* From false branch of Events Already Existing? node.  
  - *Failure Modes:* Auth errors, quota limits, field mapping errors.

---

### 3. Summary Table

| Node Name               | Node Type           | Functional Role                            | Input Node(s)                        | Output Node(s)                         | Sticky Note                                                                                      |
|-------------------------|---------------------|-------------------------------------------|------------------------------------|--------------------------------------|-------------------------------------------------------------------------------------------------|
| Schedule Trigger        | Schedule Trigger    | Triggers workflow daily at 08:00          | —                                  | page+1, Initiate Static Data, Load Old Records, Return Lines Scrapped | "### 1. Workflow Trigger with Cron Job\nThe workflow is triggered every morning at 08:30 am (local time). It starts with the initialization..." |
| Initiate Static Data     | Code                | Initializes global static data variables   | Schedule Trigger                   | —                                    | See above                                                                                       |
| page+1                  | Code                | Increments page number for pagination      | Schedule Trigger, 15 sec           | Query EU Website                     | "### 2. Scrapping and Parsing of Events blocks\nThis starts with the HTTP node collecting HTML code..." |
| Query EU Website         | HTTP Request        | Fetches EU events page HTML content        | page+1                            | Extract Blocks                       | See above                                                                                       |
| Extract Blocks           | HTML Extract        | Extracts event blocks from HTML             | Query EU Website                  | Parse Information                   | See above                                                                                       |
| Parse Information        | HTML Extract        | Parses fields from event blocks             | Extract Blocks                   | Collect Fields                     | See above                                                                                       |
| Collect Fields           | Set                 | Normalizes and structures event data       | Parse Information                | Store Tables, If                    | See above                                                                                       |
| Store Tables             | Code                | Stores new events in global static data    | Collect Fields                   | —                                  | See above                                                                                       |
| If                      | If                  | Checks if max page number exceeded          | Collect Fields                   | 15 sec (False branch)                | See above                                                                                       |
| 15 sec                  | Wait                | Waits 15 seconds between page scrapes       | If (False branch)                | page+1                             | See above                                                                                       |
| Load Old Records         | Google Sheets       | Loads existing events from Google Sheet    | Schedule Trigger                 | Aggregate                         | "### 3. Load events recorded in the Google Sheet\nLoading the events **already scrapped** to avoid duplicates..." |
| Aggregate               | Aggregate           | Aggregates existing event names             | Load Old Records                 | Combine New + Old Records           | See above                                                                                       |
| Return Lines Scrapped    | Code                | Returns stored scraped events from static data | Schedule Trigger             | Combine New + Old Records           | See above                                                                                       |
| Combine New + Old Records| Merge               | Combines new scraped and old records        | Aggregate, Return Lines Scrapped | Events Already Existing?            | See above                                                                                       |
| Events Already Existing? | If                  | Checks whether event already exists          | Combine New + Old Records        | Store New Records (False branch)    | See above                                                                                       |
| Store New Records        | Google Sheets       | Appends new unique events to Google Sheet   | Events Already Existing?         | —                                  | "### 4. Record New Events in the Google Sheet\nRecording of new events in the google sheet..."  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Configuration: Set to trigger daily at 08:00 local time.

2. **Create Initiate Static Data node**  
   - Type: Code  
   - Code:  
     ```javascript
     let workflowStaticData = $getWorkflowStaticData('global');
     workflowStaticData.page = -1;
     workflowStaticData.results = [];
     return workflowStaticData;
     ```  
   - Connect Schedule Trigger main output to this node.

3. **Create page+1 node**  
   - Type: Code  
   - Code:  
     ```javascript
     let workflowStaticData = $getWorkflowStaticData('global');
     if (!workflowStaticData.page) {
         workflowStaticData.page = 0;
     }
     workflowStaticData.page += 1;
     return { page: workflowStaticData.page };
     ```  
   - Connect Schedule Trigger and 15 sec (to be created later) outputs to this node.

4. **Create Query EU Website node**  
   - Type: HTTP Request  
   - URL: `https://european-union.europa.eu/news-and-events/events_en?page={{ $json.page }}`  
   - Connect page+1 output to this node.

5. **Create Extract Blocks node**  
   - Type: HTML Extract  
   - Operation: Extract HTML content  
   - Extraction Values:  
     - Key: `blocks`  
     - CSS Selector: `article.ecl-content-item`  
     - Return Array: true  
     - Return Value: html  
   - Connect Query EU Website output to this node.

6. **Create Parse Information node**  
   - Type: HTML Extract  
   - Operation: Extract HTML content on `blocks` property  
   - Extraction Values:  
     - event_name: `div.ecl-content-block__title a`  
     - event_link (attribute href): `div.ecl-content-block__title a`  
     - day: `span.ecl-date-block__day`  
     - month_1 (attribute title): `abbr.ecl-date-block__month`  
     - year: `span.ecl-date-block__year`  
     - event_type: `li.ecl-content-block__primary-meta-item`  
     - event_location: `.ecl-content-block__description li`  
     - month_2: `span.ecl-date-block__month`  
   - Connect Extract Blocks output to this node.

7. **Create Collect Fields node**  
   - Type: Set  
   - Assignments:  
     - event_name = `={{ $json.event_name }}`  
     - event_link = `={{ $json.event_link }}`  
     - day = `={{ $json.day }}`  
     - month = `={{ $json.month_1 || $json.month_2 }}`  
     - year = `={{ $json.year }}`  
     - event_type = `={{ $json.event_type }}`  
     - event_location = `={{ $json.event_location }}`  
   - Connect Parse Information output to this node.

8. **Create Store Tables node**  
   - Type: Code  
   - Code:  
     ```javascript
     const workflowStaticData = $getWorkflowStaticData('global');
     if (!Array.isArray(workflowStaticData.results)) {
       workflowStaticData.results = [];
     }
     const newEvents = $input.all().map(item => item.json);
     workflowStaticData.results.push(...newEvents);
     return [{
       json: {
         addedThisRound: newEvents.length,
         totalStored: workflowStaticData.results.length
       }
     }];
     ```  
   - Connect Collect Fields output to this node.

9. **Create If node (pagination control)**  
   - Type: If  
   - Condition: Check if `page > 3` (number node expression: `={{ $('page+1').item.json.page }}` > 3)  
   - Connect Collect Fields output to this node.

10. **Create 15 sec Wait node**  
    - Type: Wait  
    - Configuration: 15 seconds delay  
    - Connect If node false branch to this node.  
    - Connect 15 sec node output back to page+1 node (loop).

11. **Create Load Old Records node**  
    - Type: Google Sheets  
    - Credentials: Configure with your Google Sheets OAuth2 credentials.  
    - Document ID: Set your Google Sheet document ID containing existing events.  
    - Sheet Name: Set to `gid=0` or appropriate sheet tab.  
    - Connect Schedule Trigger output to this node.

12. **Create Aggregate node**  
    - Type: Aggregate  
    - Configuration: Aggregate field `event_name` into array named `events` (enable rename output field).  
    - Connect Load Old Records output to this node.

13. **Create Return Lines Scrapped node**  
    - Type: Code  
    - Code:  
      ```javascript
      const workflowStaticData = $getWorkflowStaticData('global');
      if (!Array.isArray(workflowStaticData.results)) {
        return [];
      }
      return workflowStaticData.results.map(result => ({ json: result }));
      ```  
    - Connect Schedule Trigger output to this node.

14. **Create Combine New + Old Records node**  
    - Type: Merge  
    - Mode: combineBySql  
    - Connect Aggregate output as first input  
    - Connect Return Lines Scrapped output as second input

15. **Create Events Already Existing? node**  
    - Type: If  
    - Condition: Check if array `events` contains `event_name`  
      - Left: `={{ $json.events }}`  
      - Right: `={{ $json.event_name }}`  
      - Operation: array contains  
    - Connect Combine New + Old Records output to this node.

16. **Create Store New Records node**  
    - Type: Google Sheets  
    - Credentials: Use same Google Sheets OAuth2 credentials as before.  
    - Operation: Append  
    - Document ID and Sheet Name: Same as Load Old Records node.  
    - Map fields:  
      - `day` = `={{ $json.day }}`  
      - `month` = `={{ $json.month }}`  
      - `year` = `={{ $json.year }}`  
      - `event_link` = `={{ $json.event_link }}`  
      - `event_name` = `={{ $json.event_name }}`  
      - `event_type` = `={{ $json.event_type }}`  
      - `event_location` = `={{ $json.event_location }}`  
    - Connect false branch of Events Already Existing? node to this node.

17. **Connections Summary:**  
    - Schedule Trigger → Initiate Static Data, page+1, Load Old Records, Return Lines Scrapped  
    - page+1 → Query EU Website → Extract Blocks → Parse Information → Collect Fields → Store Tables, If  
    - If (false) → 15 sec → page+1  
    - Load Old Records → Aggregate  
    - Aggregate + Return Lines Scrapped → Combine New + Old Records → Events Already Existing? → Store New Records (false branch)  
   
---

### 5. General Notes & Resources

| Note Content                                                                                                                                                             | Context or Link                                                                                   |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| The initialization step (`Initiate Static Data`) needs to run only once per workflow execution, regardless of context.                                                 | Sticky Note 1                                                                                   |
| Scraping is limited to 4 pages (0 to 3), adjustable in the If node controlling pagination.                                                                                | Sticky Note 2                                                                                   |
| Wait 15 seconds between page requests to avoid overwhelming the EU website server—important for polite scraping behavior.                                              | Sticky Note 2                                                                                   |
| Google Sheets nodes require OAuth2 credentials, correct document IDs, and sheet tab IDs (`gid=0`) to function properly.                                                | Sticky Note 3                                                                                   |
| For Google Sheets node field mapping, ensure fields `event_name`, `event_link`, `day`, `month`, `year`, `event_type`, `event_location` are present and correctly mapped. | [Google Sheets Node Docs](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googlesheets) |
| The workflow respects rate limiting and uses static data storage to avoid re-scraping data within one run.                                                               | Internal design considerations                                                                  |

---

**Disclaimer:**  
The text provided comes exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.