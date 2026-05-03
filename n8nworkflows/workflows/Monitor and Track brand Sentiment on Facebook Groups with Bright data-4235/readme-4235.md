Monitor and Track brand Sentiment on Facebook Groups with Bright data

https://n8nworkflows.xyz/workflows/monitor-and-track-brand-sentiment-on-facebook-groups-with-bright-data-4235


# Monitor and Track brand Sentiment on Facebook Groups with Bright data

### 1. Workflow Overview

This workflow is designed to monitor and track brand sentiment on Facebook Groups using data scraped through Bright Data’s API and analyzed with AI-driven sentiment and information extraction models. It retrieves posts from specified Facebook groups, filters posts mentioning the brand, performs sentiment analysis, extracts insights, and updates Google Sheets for reporting and further analysis.

The workflow is logically divided into the following blocks:

- **1.1 Initialization and Scheduling:** Setting API keys, scheduling the workflow trigger, loading brand names and Facebook group links.
- **1.2 Data Retrieval and Scraping:** Triggering Bright Data API to scrape Facebook groups, monitoring scraping progress, and retrieving scraped data.
- **1.3 Data Processing and Filtering:** Limiting and filtering posts that mention the brand.
- **1.4 AI Analysis:** Performing sentiment analysis and information extraction using OpenRouter language models.
- **1.5 Post-Processing and Storage:** Merging AI results, structuring data, and updating Google Sheets with detailed sentiment and insights.
- **1.6 Webhook Reception:** Receiving asynchronous scraping results via webhook and appending to storage.

---

### 2. Block-by-Block Analysis

#### 1.1 Initialization and Scheduling

**Overview:**  
This block initializes necessary credentials and inputs, schedules the workflow to run daily, and loads brand names and Facebook group URLs from Google Sheets.

**Nodes Involved:**  
- Schedule Trigger  
- Set up KEYS  
- Get Brand names  
- Aggregate brand names  
- Get links  
- Aggregate group links  
- Sticky Note7 (instructional)  
- Sticky Note1 (instructional)

**Node Details:**

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Starts workflow daily at 9 AM.  
  - Config: Interval trigger at hour 9.  
  - Inputs: None  
  - Outputs: Set up KEYS  
  - Failures: None expected; if schedule fails, workflow won’t start.

- **Set up KEYS**  
  - Type: Set  
  - Role: Stores Bright Data API key and webhook endpoint URL as variables for reuse.  
  - Config: Manually set "endpoint" and "API" strings; placeholders to be replaced with actual values.  
  - Inputs: Schedule Trigger  
  - Outputs: Get Brand names  
  - Failures: Missing or invalid keys cause authentication errors downstream.

- **Get Brand names**  
  - Type: Google Sheets  
  - Role: Loads brand names from a configured Google Sheet (tab "Brand Name").  
  - Config: OAuth2 credentials for Google Sheets, specific document and sheet ID.  
  - Inputs: Set up KEYS  
  - Outputs: Aggregate brand names  
  - Failures: OAuth errors, sheet access denied, or empty data.

- **Aggregate brand names**  
  - Type: Aggregate  
  - Role: Aggregates all brand names into a single collection for filtering.  
  - Config: Aggregates field "Brand names".  
  - Inputs: Get Brand names  
  - Outputs: Get links  
  - Failures: Empty input leads to no filtering.

- **Get links**  
  - Type: Google Sheets  
  - Role: Loads Facebook group URLs to monitor from Google Sheets (tab "Facebook group Links").  
  - Config: OAuth2 credentials, specific document and sheet ID.  
  - Inputs: Aggregate brand names  
  - Outputs: Aggregate group links  
  - Failures: OAuth errors, sheet access denied, or empty data.

- **Aggregate group links**  
  - Type: Aggregate  
  - Role: Aggregates all group URLs into a list for scraping.  
  - Config: Aggregates field "url".  
  - Inputs: Get links  
  - Outputs: facebook groups (scraping trigger)  
  - Failures: Empty input leads to no scraping.

- **Sticky Note7**  
  - Role: Instructional note - reminds user to configure Bright Data API keys and webhook endpoint.  
  - Applies to: Set up KEYS node.

- **Sticky Note1**  
  - Role: Instructional note - describes that this block gets Facebook group links to monitor.

#### 1.2 Data Retrieval and Scraping

**Overview:**  
This block triggers Bright Data’s dataset scraping API with group URLs, monitors scraping progress, waits if scraping is in progress, and retrieves the scraped posts once ready.

**Nodes Involved:**  
- facebook groups (HTTP Request)  
- get progress (HTTP Request)  
- Switch  
- Wait  
- Get data (HTTP Request)  
- set urls  
- Split Out  
- update last Scrap  
- Sticky Note2, Sticky Note3, Sticky Note4

**Node Details:**

- **facebook groups**  
  - Type: HTTP Request  
  - Role: Initiates scraping job on Bright Data with the list of Facebook group URLs.  
  - Config: POST request to Bright Data datasets trigger endpoint with JSON body mapping each URL and start date as today.  
  - Headers: Authorization Bearer token from Set up KEYS node.  
  - Inputs: Aggregate group links  
  - Outputs: get progress  
  - Failures: API auth errors, invalid URLs, network timeouts.

- **get progress**  
  - Type: HTTP Request  
  - Role: Polls Bright Data API for scraping job progress using snapshot_id from previous node.  
  - Config: GET request with Authorization header.  
  - Inputs: facebook groups or Wait  
  - Outputs: Switch  
  - Failures: API errors, invalid snapshot_id, network issues.

- **Switch**  
  - Type: Switch  
  - Role: Routes workflow based on scraping status ("ready" or "running").  
  - Config: Checks if $json.status equals "ready" or "running".  
  - Inputs: get progress  
  - Outputs:  
    - "ready" → Get data and set urls nodes in parallel  
    - "running" → Wait node  
  - Failures: Unexpected status values cause no output branch.

- **Wait**  
  - Type: Wait  
  - Role: Delays workflow before rechecking progress to avoid rapid polling.  
  - Config: Default wait (seconds not specified).  
  - Inputs: Switch ("running" branch)  
  - Outputs: get progress (poll again)  
  - Failures: None expected.

- **Get data**  
  - Type: HTTP Request  
  - Role: Fetches scraped posts data once scraping is ready.  
  - Config: GET request to datasets snapshot endpoint with snapshot_id, requesting JSON format.  
  - Inputs: Switch ("ready" branch)  
  - Outputs: Limit  
  - Failures: API errors or invalid snapshot_id.

- **set urls**  
  - Type: Set  
  - Role: Prepares the list of URLs for next processing steps (splitting).  
  - Config: Sets field "url" from aggregated group links.  
  - Inputs: Switch ("ready" branch)  
  - Outputs: Split Out  
  - Failures: Empty or invalid data.

- **Split Out**  
  - Type: Split Out  
  - Role: Splits array of URLs into individual items to update scraping timestamps.  
  - Config: Splits on field "url".  
  - Inputs: set urls  
  - Outputs: update last Scrap  
  - Failures: Empty arrays cause no output.

- **update last Scrap**  
  - Type: Google Sheets  
  - Role: Updates Google Sheet with last scraped timestamp for each URL.  
  - Config: Updates rows in the "Facebook group Links" sheet matching on "url" field, sets "last Scraped" to current date/time.  
  - Inputs: Split Out  
  - Outputs: None  
  - Failures: Google Sheets API errors, permission issues.

- **Sticky Note2**  
  - Instruction: "Scrap with Bright Data API to get posts from the group and get progress."

- **Sticky Note3**  
  - Instruction: "Wait for a few seconds if still running."

- **Sticky Note4**  
  - Instruction: "Get results from the GROUPS and update the urls scraped; Next we filter our brand name from all these posts."

#### 1.3 Data Processing and Filtering

**Overview:**  
This block limits the number of posts processed, filters posts mentioning the brand, and prepares data for sentiment analysis.

**Nodes Involved:**  
- Limit  
- Filter brand name  
- Sentiment Analysis  
- Sticky Note5, Sticky Note7 (contextual overlap)

**Node Details:**

- **Limit**  
  - Type: Limit  
  - Role: Restricts processing to a maximum of 20 posts per run to manage load.  
  - Config: maxItems set to 20.  
  - Inputs: Get data  
  - Outputs: Filter brand name  
  - Failures: None expected.

- **Filter brand name**  
  - Type: Filter  
  - Role: Filters posts that mention any of the stored brand names.  
  - Config: Checks if post content includes any of the brand names (case insensitive). Uses expression iterating over brand names array.  
  - Inputs: Limit  
  - Outputs: Sentiment Analysis  
  - Failures: Empty brand names cause no posts to pass filter.

- **Sentiment Analysis**  
  - Type: Langchain Sentiment Analysis  
  - Role: Performs AI-powered sentiment analysis categorizing posts as Positive, Negative, or Neutral.  
  - Config: Uses system prompt guiding model to output JSON with sentiment category and detailed results. Input text is post content.  
  - Inputs: Filter brand name  
  - Outputs: Merge (3 outputs)  
  - Failures: Model errors, API rate limits, malformed input text.

- **Sticky Note5**  
  - Instruction: "We do a sentiment analysis to understand the posts and how our users are feeling."

- **Sticky Note6** (appears related to next block, but contextually close)  
  - Instruction: "Extract the information from The post i.e. the summary, Category and Insights."

#### 1.4 AI Analysis

**Overview:**  
This block extracts structured insights from posts including summary, category, and key insights using a specialized AI information extractor.

**Nodes Involved:**  
- Merge  
- Information Extractor  
- OpenRouter Chat Model  
- OpenRouter Chat Model1  
- pull group results  
- Sticky Note6

**Node Details:**

- **Merge**  
  - Type: Merge  
  - Role: Merges multiple inputs: original post data, sentiment analysis, and confidence scores to unify data for further processing.  
  - Config: Number of inputs set to 3 (likely merging original post, sentiment, and confidence data).  
  - Inputs: Sentiment Analysis (3 outputs)  
  - Outputs: Information Extractor  
  - Failures: Mismatched input size or missing data.

- **Information Extractor**  
  - Type: Langchain Information Extractor  
  - Role: Summarizes posts and extracts insights, categorizes posts (e.g., Technical Support, Bug, Review).  
  - Config: Uses system prompt to analyze post content and brand name; outputs structured JSON with Insight, Sentiment, Category fields.  
  - Inputs: Merge  
  - Outputs: pull group results  
  - Failures: Model errors, API limits.

- **OpenRouter Chat Model**  
  - Type: Langchain LM Chat OpenRouter  
  - Role: AI language model used by Sentiment Analysis node.  
  - Config: Credentials use OpenRouter API.  
  - Inputs: Sentiment Analysis (ai_languageModel)  
  - Outputs: Sentiment Analysis node  
  - Failures: API authentication, rate limits.

- **OpenRouter Chat Model1**  
  - Type: Langchain LM Chat OpenRouter  
  - Role: AI language model used by Information Extractor node.  
  - Config: Same OpenRouter credentials.  
  - Inputs: Information Extractor (ai_languageModel)  
  - Outputs: Information Extractor node  
  - Failures: API failures.

- **pull group results**  
  - Type: Set  
  - Role: Pulls filtered and analyzed group post results for final formatting.  
  - Config: Sets JSON output to filtered brand name posts.  
  - Inputs: Information Extractor  
  - Outputs: insights and sentiments  
  - Failures: None expected.

- **Sticky Note6**  
  - Instruction: "Extract the information from The post i.e. the summary, Category and Insights."

#### 1.5 Post-Processing and Storage

**Overview:**  
This block formats enriched data, combines post metadata with sentiment and insights, and updates Google Sheets to store results.

**Nodes Involved:**  
- insights and sentiments  
- Update sentiments  
- Sticky Note8  
- update last Scrap

**Node Details:**

- **insights and sentiments**  
  - Type: Set  
  - Role: Combines post metadata with sentiment analysis, confidence, and extracted insights.  
  - Config: Assigns multiple fields such as url, post_id, content, dates, group info, sentiment, confidence, Insight, Category, etc., mostly via expressions pulling from previous nodes.  
  - Inputs: pull group results  
  - Outputs: Update sentiments  
  - Failures: Missing fields in inputs cause incomplete data.

- **Update sentiments**  
  - Type: Google Sheets  
  - Role: Appends or updates the "Groups" sheet with enriched sentiment and insights data.  
  - Config: Matches rows on "post_id"; auto-maps input data to columns; handles append or update.  
  - Inputs: insights and sentiments  
  - Outputs: None  
  - Failures: Google Sheets API errors, permission issues, rate limits.

- **update last Scrap** (also part of prior block, updates scrape timestamps)  
  - See above in 1.2

- **Sticky Note8**  
  - Instruction: "Format and update your results to Google Sheets for further analysis."

#### 1.6 Webhook Reception

**Overview:**  
Receives asynchronous scraping results posted by Bright Data webhook and appends all raw posts to a Google Sheet for record-keeping.

**Nodes Involved:**  
- Receive results (Webhook)  
- Split Out1  
- Get Brand names1  
- Sticky Note9

**Node Details:**

- **Receive results**  
  - Type: Webhook  
  - Role: Receives POST data from Bright Data containing scraped Facebook group posts.  
  - Config: Webhook path configured uniquely, accepts POST requests.  
  - Inputs: None (start node)  
  - Outputs: Split Out1  
  - Failures: Webhook URL must be correctly configured in Bright Data; unsupported HTTP methods cause failure.

- **Split Out1**  
  - Type: Split Out  
  - Role: Splits incoming webhook JSON array field "body" into individual post items.  
  - Config: Splits on "body".  
  - Inputs: Receive results  
  - Outputs: Get Brand names1  
  - Failures: Empty or malformed payloads.

- **Get Brand names1**  
  - Type: Google Sheets  
  - Role: Appends each individual post item to the "All POSTS" sheet for logging.  
  - Config: OAuth2 credentials, specific document and sheet ID, operation "append".  
  - Inputs: Split Out1  
  - Outputs: None  
  - Failures: Sheet access errors, data mapping errors.

- **Sticky Note9**  
  - Instruction: "Receives the results from the Facebook group scrap and stores all the results to Google sheets."

---

### 3. Summary Table

| Node Name           | Node Type                      | Functional Role                                              | Input Node(s)                  | Output Node(s)                | Sticky Note                                                                                      |
|---------------------|--------------------------------|--------------------------------------------------------------|-------------------------------|------------------------------|------------------------------------------------------------------------------------------------|
| Schedule Trigger     | Schedule Trigger               | Starts workflow daily at 9 AM                                | None                          | Set up KEYS                  |                                                                                                |
| Set up KEYS         | Set                            | Stores API keys and webhook endpoint                         | Schedule Trigger              | Get Brand names              | 1. Set your bright data API and webhook endpoint to receive the results                         |
| Get Brand names      | Google Sheets                  | Loads brand names                                            | Set up KEYS                  | Aggregate brand names        |                                                                                                |
| Aggregate brand names| Aggregate                     | Aggregates brand names into one collection                   | Get Brand names              | Get links                   |                                                                                                |
| Get links            | Google Sheets                  | Loads Facebook group URLs                                    | Aggregate brand names        | Aggregate group links        | - 2.Get the links for the facebook groups we want to monitor                                   |
| Aggregate group links| Aggregate                     | Aggregates group URLs into array                             | Get links                   | facebook groups             |                                                                                                |
| facebook groups      | HTTP Request                   | Triggers Bright Data scraping job                            | Aggregate group links        | get progress                | - 3.Scrap with Bright Data API to get posts from the group and get progress                    |
| get progress         | HTTP Request                   | Polls scraping job progress                                  | facebook groups, Wait        | Switch                      |                                                                                                |
| Switch               | Switch                        | Routes based on scraping job status                          | get progress                | Get data, set urls, Wait    |                                                                                                |
| Wait                 | Wait                          | Waits before polling progress again                          | Switch                      | get progress                | 4.Wait for a few seconds if still running                                                      |
| Get data             | HTTP Request                   | Retrieves scraped data                                       | Switch                      | Limit                      |                                                                                                |
| set urls             | Set                            | Prepares URLs list for updating scrape timestamps           | Switch                      | Split Out                  |                                                                                                |
| Split Out            | Split Out                     | Splits URL array into individual items                       | set urls                    | update last Scrap           | 5.Get results from the GROUPS and update the urls scraped; Next we filter our brand name       |
| update last Scrap    | Google Sheets                  | Updates last scraped timestamp for each group URL           | Split Out                   | None                       |                                                                                                |
| Limit                | Limit                         | Limits number of posts processed                             | Get data                   | Filter brand name           |                                                                                                |
| Filter brand name    | Filter                        | Filters posts mentioning brand names                         | Limit                      | Sentiment Analysis          |                                                                                                |
| Sentiment Analysis   | Langchain Sentiment Analysis  | Performs sentiment analysis                                  | Filter brand name           | Merge                      | 7.We do a sentiment analysis to understand the posts and how our users are feeling             |
| Merge                | Merge                         | Merges sentiment and confidence data with posts             | Sentiment Analysis          | Information Extractor       |                                                                                                |
| Information Extractor| Langchain Information Extractor| Extracts insights, category, and summary from posts         | Merge                      | pull group results          | 8.Extract the information from The post ie the summary, Category and Insights                  |
| pull group results   | Set                           | Prepares formatted post data with insights                   | Information Extractor       | insights and sentiments     |                                                                                                |
| insights and sentiments| Set                          | Combines all data fields for final output                    | pull group results          | Update sentiments           |                                                                                                |
| Update sentiments    | Google Sheets                  | Updates or appends enriched posts with insights             | insights and sentiments     | None                       | 9.Format and update your results to Google Sheets for further analysis                         |
| Receive results      | Webhook                       | Receives asynchronous scraped data from Bright Data         | None                       | Split Out1                 | Receives the results from the Facebook group scrap and stores all the results to Google sheets |
| Split Out1           | Split Out                     | Splits webhook body array into individual posts              | Receive results             | Get Brand names1            |                                                                                                |
| Get Brand names1     | Google Sheets                  | Appends raw scraped posts to "All POSTS" Google Sheet       | Split Out1                 | None                       |                                                                                                |
| Sticky Note(s)       | Sticky Note                   | Instructional notes for user guidance                        | Various                     | None                       | See detailed nodes above for each note content                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Set trigger to run daily at 9:00 AM.

2. **Create a Set node named "Set up KEYS"**  
   - Add two string fields:  
     - "endpoint" with your POST webhook URL (from Bright Data).  
     - "API" with your Bright Data API key.  
   - Connect Schedule Trigger → Set up KEYS.

3. **Create a Google Sheets node "Get Brand names"**  
   - Configure OAuth2 credentials for Google Sheets.  
   - Select your Google Sheet document and sheet/tab containing brand names (e.g., "Brand Name").  
   - Connect Set up KEYS → Get Brand names.

4. **Create an Aggregate node "Aggregate brand names"**  
   - Aggregate the "Brand names" column into an array.  
   - Connect Get Brand names → Aggregate brand names.

5. **Create a Google Sheets node "Get links"**  
   - Configure OAuth2 credentials.  
   - Select Google Sheet and sheet/tab containing Facebook group URLs (e.g., "Facebook group Links").  
   - Connect Aggregate brand names → Get links.

6. **Create an Aggregate node "Aggregate group links"**  
   - Aggregate the "url" column into an array.  
   - Connect Get links → Aggregate group links.

7. **Create an HTTP Request node "facebook groups"**  
   - Method: POST  
   - URL: https://api.brightdata.com/datasets/v3/trigger  
   - Headers: Authorization: Bearer {{API}} (use expression to get from Set up KEYS)  
   - Query parameters: dataset_id=gd_lz11l67o2cb3r0lkj3, endpoint={{endpoint}}, format=json, uncompressed_webhook=true, include_errors=true  
   - Body (JSON): Map each URL in aggregated group links to an object with:  
     - url: URL  
     - start_date: current date (yyyy-MM-dd)  
     - end_date: empty string  
   - Connect Aggregate group links → facebook groups.

8. **Create an HTTP Request node "get progress"**  
   - Method: GET  
   - URL: https://api.brightdata.com/datasets/v3/progress/{{snapshot_id}} (use snapshot_id from facebook groups output)  
   - Header Authorization: Bearer {{API}}  
   - Connect facebook groups → get progress.

9. **Create a Switch node "Switch"**  
   - Evaluate $json.status against "ready" and "running".  
   - Output "ready" branch connects to "Get data" and "set urls".  
   - Output "running" branch connects to "Wait".  
   - Connect get progress → Switch.

10. **Create a Wait node "Wait"**  
   - Default wait time (few seconds).  
   - Connect Switch ("running") → Wait → get progress (loop).

11. **Create an HTTP Request node "Get data"**  
   - Method: GET  
   - URL: https://api.brightdata.com/datasets/v3/snapshot/{{snapshot_id}}  
   - Query parameter: format=json  
   - Header Authorization: Bearer {{API}}  
   - Connect Switch ("ready") → Get data.

12. **Create a Set node "set urls"**  
   - Set field "url" to the aggregated group links URLs.  
   - Connect Switch ("ready") → set urls.

13. **Create a Split Out node "Split Out"**  
   - Field to split: "url".  
   - Connect set urls → Split Out.

14. **Create a Google Sheets node "update last Scrap"**  
   - Update rows in "Facebook group Links" sheet matched on "url".  
   - Update "last Scraped" column with current timestamp.  
   - Connect Split Out → update last Scrap.

15. **Create a Limit node "Limit"**  
   - Max items: 20 to limit posts processed.  
   - Connect Get data → Limit.

16. **Create a Filter node "Filter brand name"**  
   - Condition: post content includes any brand name (case-insensitive) using expression that compares post content against brand names array from Aggregate brand names.  
   - Connect Limit → Filter brand name.

17. **Create a Langchain Sentiment Analysis node "Sentiment Analysis"**  
   - Categories: Positive, Negative, Neutral  
   - System prompt guides the model to output JSON sentiment classification.  
   - Input text: post content.  
   - Connect Filter brand name → Sentiment Analysis.

18. **Create a Merge node "Merge"**  
   - Number of inputs: 3  
   - Connect Sentiment Analysis outputs → Merge.

19. **Create a Langchain Information Extractor node "Information Extractor"**  
   - System prompt: Summarize post, identify segment, suggest category (e.g., Technical Support, Bug, etc.) with brand name reference.  
   - Input text: post content.  
   - Connect Merge → Information Extractor.

20. **Create an OpenRouter Chat Model credential and nodes "OpenRouter Chat Model" and "OpenRouter Chat Model1"**  
   - Configure OpenRouter API credentials.  
   - Assign these credentials to Sentiment Analysis and Information Extractor respectively.

21. **Create a Set node "pull group results"**  
   - Set JSON output to filtered brand name posts.  
   - Connect Information Extractor → pull group results.

22. **Create a Set node "insights and sentiments"**  
   - Combine fields from post metadata, sentiment analysis, confidence, and AI insights into structured JSON.  
   - Use expressions referencing Merge and Information Extractor outputs.  
   - Connect pull group results → insights and sentiments.

23. **Create a Google Sheets node "Update sentiments"**  
   - Append or update rows in "Groups" sheet matching on "post_id".  
   - Auto map inputs to columns.  
   - Connect insights and sentiments → Update sentiments.

24. **Create a Webhook node "Receive results"**  
   - Configure webhook path uniquely.  
   - Accept POST requests from Bright Data webhook.  
   - Connect to Split Out1.

25. **Create a Split Out node "Split Out1"**  
   - Split on field "body" of webhook payload.  
   - Connect Receive results → Split Out1.

26. **Create a Google Sheets node "Get Brand names1"**  
   - Append each post item to "All POSTS" sheet for record-keeping.  
   - Connect Split Out1 → Get Brand names1.

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                                                                               |
|-----------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| This workflow keeps track of brand mentions across Facebook groups and analyzes sentiment.     | Sticky Note overview at start of workflow.                                                                   |
| Make a copy of this Google Sheet to get started easily: https://docs.google.com/spreadsheets/d/1TXF_xLPF7XJJakoWB5Ix-tTduvX3GRxocJcp6DA-U_A/edit?usp=sharing | Initial setup for brand names and group links.                                                               |
| Bright Data API documentation is essential for understanding scraping dataset triggers.        | https://brightdata.com/docs/datasets/api                                                                    |
| OpenRouter API is used for AI sentiment and information extraction models.                      | https://docs.openrouter.ai/                                                                                   |
| Ensure Google Sheets OAuth2 credentials have sufficient permissions to read/write specified sheets.| Workflow heavily relies on Google Sheets for input and output storage.                                        |
| The webhook URL from "Receive results" node must be configured in Bright Data dataset settings.| Critical for asynchronous scraping result delivery.                                                          |

---

_Disclaimer: The text above is generated exclusively from an n8n automated workflow JSON export. It adheres strictly to content policies and contains no illegal or protected elements. All processed data is legal and public._