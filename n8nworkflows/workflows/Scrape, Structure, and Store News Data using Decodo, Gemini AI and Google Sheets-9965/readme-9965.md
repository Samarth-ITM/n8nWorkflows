Scrape, Structure, and Store News Data using Decodo, Gemini AI and Google Sheets

https://n8nworkflows.xyz/workflows/scrape--structure--and-store-news-data-using-decodo--gemini-ai-and-google-sheets-9965


# Scrape, Structure, and Store News Data using Decodo, Gemini AI and Google Sheets

### 1. Workflow Overview

This workflow automates the process of scraping, structuring, and storing news or forum posts from specified domains using Decodo API, Google Gemini AI, and Google Sheets. It is designed for users such as data journalists, market researchers, or AI enthusiasts who want to monitor trending topics and key engagement metrics on specific news forums.

The workflow logic is divided into the following blocks:

- **1.1 Scheduled Trigger and Configuration Setup**: Periodic activation and initialization of forum URLs, geolocation, and Google Sheet ID.
- **1.2 Forum URL Processing**: Splitting and iterating over multiple forum URLs for sequential scraping.
- **1.3 Data Scraping with Decodo**: Using Decodo API to scrape raw text data from each forum URL.
- **1.4 AI-Based Structured Data Extraction**: Leveraging Google Gemini AI to parse unstructured scraped text into structured JSON news items.
- **1.5 News Item Processing and Storage**: Splitting extracted news items, generating unique keys, and appending or updating them in Google Sheets.
- **1.6 Logging and Wait Cycle**: Recording scraping results for monitoring and handling wait time between scrapes.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger and Configuration Setup

- **Overview:** This block initiates the workflow on a schedule and sets key configuration parameters like forum URLs, geolocation, and Google Sheet ID.
- **Nodes Involved:** Schedule Trigger, Workflow Config, Sticky Note2, Sticky Note3, Sticky Note

##### Node Details

- **Schedule Trigger**
  - Type: Schedule Trigger
  - Role: Starts workflow execution on a recurring schedule (default: every interval, customizable)
  - Configuration: Interval set (default empty, user to define)
  - Connections: Output → Workflow Config
  - Failures: Misconfigured schedule, time zone issues

- **Workflow Config**
  - Type: Set
  - Role: Defines key parameters — list of forum URLs, geolocation string, and Google Sheet ID
  - Configuration: 
    - `forums`: Array of forum URLs (e.g., Hacker News filtered by domains)
    - `geo`: Geographical location string ("United States")
    - `sheet_id`: Placeholder for Google Sheets document ID
  - Connections: Output → Split Forums
  - Failures: Invalid or missing sheet ID, malformed URLs

- **Sticky Note2**
  - Role: Instruction to specify forum URLs, geolocation, and Sheet ID
  - Positionally linked to Workflow Config

- **Sticky Note3**
  - Role: Instruction to adjust schedule settings (e.g., daily midnight)
  - Positionally linked to Schedule Trigger

- **Sticky Note**
  - Role: Overview and usage instructions with branding and signup links
  - Contains:
    - Workflow purpose
    - Target audience
    - Setup instructions
    - Helpful links (Decodo signup with discount: https://visit.decodo.com/discount)

---

#### 2.2 Forum URL Processing

- **Overview:** Splits the array of forum URLs into individual entries, then iterates over them in batches to process sequentially.
- **Nodes Involved:** Split Forums, Iterate Forums

##### Node Details

- **Split Forums**
  - Type: Split Out
  - Role: Separates the `forums` array into individual URLs for processing
  - Configuration:
    - Splits field `forums`
    - Includes additional field `geo` in output items
    - Destination field name for split-out elements: `url`
  - Input: From Workflow Config
  - Output: To Iterate Forums
  - Failures: Empty or malformed forum list, missing `geo` field

- **Iterate Forums**
  - Type: Split In Batches
  - Role: Processes each forum URL one at a time to avoid overload or rate limits
  - Configuration: Defaults, no batch size specified (defaults to 1)
  - Input: From Split Forums
  - Output: Parallel output to Scrape Forum Data node
  - Failures: Batch processing errors, concurrency issues

---

#### 2.3 Data Scraping with Decodo

- **Overview:** Scrapes raw forum or news text data from each forum URL using Decodo's scraping API, providing geolocation context.
- **Nodes Involved:** Scrape Forum Data

##### Node Details

- **Scrape Forum Data**
  - Type: Decodo API node
  - Role: Extracts raw unstructured text from specified forum URLs
  - Configuration:
    - URL parameter: bound dynamically from current item’s `url`
    - Geolocation parameter: bound dynamically from current item’s `geo`
  - Credentials: Decodo API (configured with user’s Decodo key)
  - Input: From Iterate Forums
  - Output: To Extract Structured News Data
  - Failures: API authentication errors, rate limits, network timeouts, invalid URLs

---

#### 2.4 AI-Based Structured Data Extraction

- **Overview:** Uses Google Gemini AI to parse unstructured scraped text into a strictly formatted JSON array of news posts with fields like title, url, source, points, comments, author, and posted date.
- **Nodes Involved:** Google Gemini Model, Extract Structured News Data, Parse JSON Output

##### Node Details

- **Google Gemini Model**
  - Type: LangChain Google Gemini Chat LLM node
  - Role: Provides language model capabilities to assist in structuring text
  - Credentials: Google Palm API with Gemini access
  - Input: None directly; used as part of Extract Structured News Data node’s internal chain
  - Output: Feeds AI output to Extract Structured News Data node
  - Failures: API quota, network issues, malformed prompts

- **Extract Structured News Data**
  - Type: LangChain Chain LLM node with custom prompt
  - Role: Core AI prompt that instructs Gemini to convert scraped text into structured JSON
  - Configuration:
    - Custom prompt templates specifying JSON schema for news posts
    - Enforces output as JSON array only, no comments or markdown
    - Uses current date/time for relative posted dates
    - Example text and expected output included in prompt for better accuracy
  - Input: Raw scraped text from Scrape Forum Data
  - Output: Parsed JSON string to Parse JSON Output node and logging node
  - Failures: AI model errors, prompt misinterpretation, empty input data

- **Parse JSON Output**
  - Type: LangChain Output Parser Structured node
  - Role: Converts AI-generated JSON string into n8n JSON data structure for downstream processing
  - Configuration: JSON schema example matching expected news post structure
  - Input: AI output from Extract Structured News Data
  - Output: To Split News Items node
  - Failures: Invalid JSON output, parsing errors

---

#### 2.5 News Item Processing and Storage

- **Overview:** Splits structured news items into individual entries, generates unique keys, and appends or updates them in a Google Sheet to maintain a record.
- **Nodes Involved:** Split News Items, Generate Unique Key, Update Google Sheet (News), Sticky Note7, Sticky Note8

##### Node Details

- **Split News Items**
  - Type: Split Out
  - Role: Splits the JSON array of news items into single news objects for individual processing
  - Configuration:
    - Field to split: `output` (the parsed JSON array)
  - Input: From Extract Structured News Data
  - Output: To Generate Unique Key
  - Failures: Empty or malformed array, missing output field

- **Generate Unique Key**
  - Type: Crypto
  - Role: Generates a unique hash key based on concatenation of news URL and author to identify unique entries
  - Configuration:
    - Value expression: `${url}+${author}`
    - Output property: `key`
    - Executes for each news item individually
  - Input: From Split News Items
  - Output: To Update Google Sheet (News)
  - Failures: Missing URL or author fields, hashing errors

- **Update Google Sheet (News)**
  - Type: Google Sheets
  - Role: Appends new or updates existing news entries in the Google Sheet based on unique key
  - Configuration:
    - Mapping columns include key, url, title, author, points, source, comments, posted_at, last_updated (timestamp)
    - Operation: appendOrUpdate (upserts based on `key`)
    - Sheet name and document ID dynamically assigned (Sheet ID from workflow config)
  - Credentials: Google Sheets OAuth2
  - Input: From Generate Unique Key
  - Output: None downstream
  - Failures: Authentication errors, sheet ID or tab mismatch, schema mismatch

- **Sticky Note7 and Sticky Note8**
  - Role: Reminders to ensure Google Sheet tab matches column schema for proper data mapping

---

#### 2.6 Logging and Wait Cycle

- **Overview:** Logs scraping results (count of news items per forum URL with timestamp) into a separate Google Sheets tab and enforces wait time between scraping cycles.
- **Nodes Involved:** Log Scrape Results, Wait Between Scrapes

##### Node Details

- **Log Scrape Results**
  - Type: Google Sheets
  - Role: Appends a log entry recording forum URL, geolocation, number of news items scraped, and timestamp
  - Configuration:
    - Columns: forum URL, geo, news_count (length of result array), scraped_at (current time)
    - Operation: append
    - Sheet tab: Logs (gid=0)
    - Document ID from workflow config
  - Credentials: Google Sheets OAuth2
  - Input: From Extract Structured News Data (main output)
  - Output: To Wait Between Scrapes
  - Failures: Authentication errors, sheet ID or tab mismatch

- **Wait Between Scrapes**
  - Type: Wait
  - Role: Pauses workflow for configured minutes (default 1) before continuing with next forum URL iteration
  - Configuration:
    - Unit: minutes
    - Amount: 1
  - Input: From Log Scrape Results
  - Output: Back to Iterate Forums (loop)
  - Failures: Misconfiguration of wait time

---

### 3. Summary Table

| Node Name               | Node Type                           | Functional Role                         | Input Node(s)            | Output Node(s)                    | Sticky Note                                                                                                  |
|-------------------------|-----------------------------------|---------------------------------------|--------------------------|---------------------------------|--------------------------------------------------------------------------------------------------------------|
| Schedule Trigger        | Schedule Trigger                  | Starts workflow on schedule           | —                        | Workflow Config                  | ### Adjust schedule (e.g., every day at midnight)                                                            |
| Workflow Config         | Set                              | Defines forum URLs, geo, Sheet ID     | Schedule Trigger          | Split Forums                    | ### Specify forum URLs, geolocation, and Sheet ID                                                           |
| Split Forums            | Split Out                        | Split forum URLs array into items     | Workflow Config           | Iterate Forums                  |                                                                                                              |
| Iterate Forums          | Split In Batches                 | Iterates over forum URLs sequentially | Split Forums              | Scrape Forum Data               |                                                                                                              |
| Scrape Forum Data       | Decodo API                      | Scrapes raw text from forum URLs      | Iterate Forums            | Extract Structured News Data    |                                                                                                              |
| Google Gemini Model     | LangChain Google Gemini LLM      | Provides AI language model             | — (internal)              | Extract Structured News Data    |                                                                                                              |
| Extract Structured News Data | LangChain Chain LLM           | Converts raw text to structured JSON  | Scrape Forum Data, Google Gemini Model | Split News Items, Log Scrape Results |                                                                                                          |
| Parse JSON Output       | LangChain Output Parser Structured | Parses AI JSON output                  | Extract Structured News Data | Split News Items                |                                                                                                              |
| Split News Items        | Split Out                       | Splits JSON array into individual items | Parse JSON Output         | Generate Unique Key             |                                                                                                              |
| Generate Unique Key     | Crypto                         | Creates unique key for news items      | Split News Items          | Update Google Sheet (News)      |                                                                                                              |
| Update Google Sheet (News) | Google Sheets                  | Stores or updates news items in Sheet | Generate Unique Key       | —                              | ### Ensure your sheet tab matches the column schema                                                          |
| Log Scrape Results      | Google Sheets                   | Logs scraping summary data             | Extract Structured News Data | Wait Between Scrapes           |                                                                                                              |
| Wait Between Scrapes    | Wait                           | Waits between scrapes                   | Log Scrape Results        | Iterate Forums                 |                                                                                                              |
| Sticky Note             | Sticky Note                    | Workflow overview and branding         | —                        | —                              | ![Waha Johan](https://drive.google.com/thumbnail?id=1SHtHQ7h1pflq_L_obGfBK29wGUY16-Vg&sz=w2000) AI-Powered News Scraper using Decodo and Gemini AI... |
| Sticky Note2            | Sticky Note                    | Instruction for config setup            | —                        | —                              | ### Specify forum URLs, geolocation, and Sheet ID                                                           |
| Sticky Note3            | Sticky Note                    | Instruction for schedule adjustment     | —                        | —                              | ### Adjust schedule (e.g., every day at midnight)                                                            |
| Sticky Note7            | Sticky Note                    | Reminder for Google Sheet tab schema    | —                        | —                              | ### Ensure your sheet tab matches the column schema                                                          |
| Sticky Note8            | Sticky Note                    | Reminder for Google Sheet tab schema    | —                        | —                              | ### Ensure your sheet tab matches the column schema                                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**
   - Type: Schedule Trigger
   - Configure interval (e.g., daily at midnight or as desired)
   - No credentials required
   - Connect output to next node

2. **Create Set Node (Workflow Config)**
   - Type: Set
   - Add variables:
     - `forums` (Array): List forum URLs, e.g.:
       ```
       [
         "https://news.ycombinator.com/from?site=openai.com",
         "https://news.ycombinator.com/from?site=anthropic.com"
       ]
       ```
     - `geo` (String): e.g., "United States"
     - `sheet_id` (String): Your Google Sheets document ID
   - Connect Schedule Trigger output to this node

3. **Create Split Out Node (Split Forums)**
   - Type: Split Out
   - Set field to split: `forums`
   - Include field: `geo`
   - Destination field name for split output: `url`
   - Connect Workflow Config output to this node

4. **Create Split In Batches Node (Iterate Forums)**
   - Type: Split In Batches
   - No special batch size (default 1)
   - Connect Split Forums output to this node

5. **Create Decodo Node (Scrape Forum Data)**
   - Type: Decodo API node
   - Credentials: Configure Decodo API credentials with your API key
   - Parameters:
     - `url`: Expression `{{$json.url}}`
     - `geo`: Expression `{{$json.geo}}`
   - Connect Iterate Forums output (main output 1) to this node

6. **Create LangChain Google Gemini LLM Node (Google Gemini Model)**
   - Type: LangChain Google Gemini Chat
   - Credentials: Set Google Palm API credentials with Gemini access
   - No direct input connection; used internally by next node

7. **Create LangChain Chain LLM Node (Extract Structured News Data)**
   - Type: LangChain Chain LLM
   - Configure prompt:
     - Instruction to extract news posts as structured JSON array with fields: title, url, source, points, comments, author, posted_at
     - Include example scraped text and JSON output in prompt
     - Ensure output is JSON only, no other text
   - Bind Google Gemini Model node as language model in chain
   - Input: Connect from Scrape Forum Data node
   - Output: Connect to Parse JSON Output and Log Scrape Results nodes

8. **Create LangChain Output Parser Structured Node (Parse JSON Output)**
   - Type: LangChain Output Parser Structured
   - Configure JSON schema example matching the news post structure
   - Input: Connect from Extract Structured News Data node’s AI output port
   - Output: Connect to Split News Items node

9. **Create Split Out Node (Split News Items)**
   - Type: Split Out
   - Field to split: `output` (the parsed JSON array)
   - Input: Connect from Parse JSON Output node
   - Output: Connect to Generate Unique Key node

10. **Create Crypto Node (Generate Unique Key)**
    - Type: Crypto
    - Parameters:
      - Value: Expression `={{ `${$json.url}+${$json.author}` }}`
      - Data Property Name: `key`
    - Executes for each item individually
    - Input: Connect from Split News Items
    - Output: Connect to Update Google Sheet (News)

11. **Create Google Sheets Node (Update Google Sheet (News))**
    - Type: Google Sheets
    - Credentials: Configure Google Sheets OAuth2 credentials
    - Parameters:
      - Operation: appendOrUpdate
      - Document ID: Expression `={{ $('Workflow Config').item.json.sheet_id }}`
      - Sheet Name: Specify sheet tab (e.g., "News")
      - Mapping columns with keys: key, url, title, author, points, source, comments, posted_at, last_updated (set to current time)
    - Input: Connect from Generate Unique Key

12. **Create Google Sheets Node (Log Scrape Results)**
    - Type: Google Sheets
    - Credentials: Same Google Sheets OAuth2 credentials
    - Parameters:
      - Operation: append
      - Document ID: Same as above
      - Sheet Name: Logs tab (gid=0)
      - Columns: forum URL, geo, news_count (length of news items), scraped_at (current timestamp)
    - Input: Connect from Extract Structured News Data node (main output)

13. **Create Wait Node (Wait Between Scrapes)**
    - Type: Wait
    - Parameters:
      - Unit: minutes
      - Amount: 1 (or as preferred)
    - Input: Connect from Log Scrape Results
    - Output: Connect back to Iterate Forums to continue processing next forum URL

14. **Add Sticky Notes**
    - Add sticky notes as instructions and reminders near relevant nodes:
      - Workflow overview and branding near start
      - Configuration instructions near Workflow Config
      - Schedule setup note near Schedule Trigger
      - Google Sheets schema reminder near Google Sheets nodes

---

### 5. General Notes & Resources

| Note Content                                                                                                                          | Context or Link                                                                                   |
|-------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Sign up for Decodo with a discount offer                                                                                             | https://visit.decodo.com/discount                                                               |
| Workflow ideal for data journalists, market researchers, AI enthusiasts monitoring trending topics                                  | Sticky Note at workflow start                                                                   |
| Ensure Google Sheets tab names and columns exactly match the configured schema for proper appendOrUpdate operations                 | Sticky Note7 and Sticky Note8                                                                   |
| Adjust schedule frequency according to your needs (e.g., daily at midnight recommended)                                              | Sticky Note3                                                                                     |
| Workflow leverages Google Gemini AI via LangChain integration for advanced language model parsing                                     | Google Gemini Model node configuration                                                          |

---

**Disclaimer:**  
The provided text originates exclusively from an automated n8n workflow. It complies strictly with content policies and contains no illegal, offensive, or protected material. All data handled is legal and publicly available.