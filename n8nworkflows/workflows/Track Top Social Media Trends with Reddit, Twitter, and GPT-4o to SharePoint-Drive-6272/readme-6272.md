Track Top Social Media Trends with Reddit, Twitter, and GPT-4o to SharePoint/Drive

https://n8nworkflows.xyz/workflows/track-top-social-media-trends-with-reddit--twitter--and-gpt-4o-to-sharepoint-drive-6272


# Track Top Social Media Trends with Reddit, Twitter, and GPT-4o to SharePoint/Drive

---

### 1. Workflow Overview

This workflow, titled **"Track Top Social Media Trends with Reddit, Twitter, and GPT-4o to SharePoint/Drive"**, is designed to gather trending topics from multiple social media platforms, analyze and rank these trends using an AI language model (GPT-4o), and then export the results into a structured Excel file stored on Microsoft SharePoint. The workflow targets content creators, marketers, and social media analysts aiming to monitor and consolidate top trends from Reddit and Twitter (X), with scope for extensibility to other platforms.

The workflow logic is organized into the following main blocks:

**1.1 Data Sources Branch**  
Collects trending posts from Reddit and Twitter via their respective APIs.

**1.2 Data Aggregation and AI Analysis Branch**  
Merges and aggregates raw data, then uses GPT-4o to analyze and rank the top five trends in a structured JSON format.

**1.3 Spreadsheet Creation Branch**  
Transforms the AI-analyzed trend data into rows and generates an Excel spreadsheet file.

**1.4 Storage Branch**  
Uploads the generated Excel file to a configured Microsoft SharePoint folder for persistent storage and sharing.

---

### 2. Block-by-Block Analysis

#### 2.1 Data Sources Branch

- **Overview:**  
This block is responsible for fetching raw trend data from social media APIs. It calls Reddit’s “hot” posts endpoint for a selected subreddit and an external custom Twitter trends service. Additional API sources can be integrated here before merging data.

- **Nodes Involved:**  
  - `Set Config`  
  - `Reddit API`  
  - `Get Twitter Trends`  
  - `Merge Data`  

- **Node Details:**

1. **Set Config**  
   - *Type:* Set node  
   - *Role:* Initializes workflow variables such as the current date (`dateToday`) and the topic of interest (`topic`), set here to "Artificial Intelligence".  
   - *Configuration:* Assigns formatted date string and topic string.  
   - *Inputs:* Manual trigger node  
   - *Outputs:* Feeds `Reddit API` and `Get Twitter Trends` nodes.  
   - *Edge cases:* Date formatting failures unlikely; topic variable must be valid string.

2. **Reddit API**  
   - *Type:* HTTP Request node with OAuth2 Reddit credentials  
   - *Role:* Fetches the hot posts from subreddit `/r/artificial`.  
   - *Configuration:* GET request to `https://oauth.reddit.com/r/artificial/hot` with predefined Reddit OAuth2 credentials.  
   - *Inputs:* `Set Config`  
   - *Outputs:* `Merge Data`  
   - *Version:* v3  
   - *Edge cases:* OAuth token expiration, rate limits, network timeouts, or invalid subreddit may cause failure.

3. **Get Twitter Trends**  
   - *Type:* HTTP Request node  
   - *Role:* Calls a custom Twitter trends endpoint (placeholder URL) with a POST request to run a Python script for dynamic Twitter trend retrieval using the topic.  
   - *Configuration:* Sends JSON body invoking a backend script with the topic parameter; allows unauthorized certificates (likely for self-signed SSL).  
   - *Inputs:* `Set Config`  
   - *Outputs:* `Merge Data` (second input)  
   - *Version:* v4.2  
   - *Edge cases:* Service unavailability, SSL errors, invalid body or headers, script execution errors.

4. **Merge Data**  
   - *Type:* Merge node  
   - *Role:* Combines the outputs from Reddit API and Twitter Trends into a single data stream for further processing.  
   - *Configuration:* Defaults to merging inputs; no special parameters set.  
   - *Inputs:* `Reddit API` and `Get Twitter Trends`  
   - *Outputs:* `Aggregate Content`  
   - *Version:* v2.1  
   - *Edge cases:* Mismatched or missing inputs could cause data loss or empty merge.

- **Sticky Note Content:**  
  *Explains the purpose of this branch, credentials needed (Reddit OAuth2, any other API keys), and user tweaks such as changing subreddit or extending with more sources.*

#### 2.2 Data Aggregation and AI Analysis Branch

- **Overview:**  
Aggregates raw social media data into summarized text, then sends it to an AI agent (GPT-4o) to extract and rank the top 5 trends in a structured JSON format.

- **Nodes Involved:**  
  - `Aggregate Content`  
  - `AI Agent`  
  - `OpenAI Chat Model`  
  - `Prepare Excel`  

- **Node Details:**

1. **Aggregate Content**  
   - *Type:* Code node (JavaScript)  
   - *Role:* Extracts the top 3 Reddit hot post titles and concatenates them into a string. Also includes Twitter trends output for AI analysis.  
   - *Configuration:* Loops over all input data to process Reddit posts and appends Twitter trends output from `Get Twitter Trends`.  
   - *Inputs:* `Merge Data`  
   - *Outputs:* `AI Agent`  
   - *Version:* v2  
   - *Edge cases:* Missing or malformed Reddit or Twitter data could cause empty or incomplete content strings.

2. **AI Agent**  
   - *Type:* LangChain Agent node  
   - *Role:* Processes aggregated trend data using GPT-4o to rank and return the top 5 trends in JSON format.  
   - *Configuration:* Uses a custom prompt embedding both Reddit and Twitter trend texts, instructing the AI to respond with a JSON array of ranked trends including rank, name, description, and platforms.  
   - *Inputs:* `Aggregate Content`  
   - *Outputs:* `Prepare Excel`  
   - *Version:* v2  
   - *Edge cases:* API rate limits, prompt failures, malformed AI response, or empty input may cause downstream parsing errors.

3. **OpenAI Chat Model**  
   - *Type:* LangChain Chat Model (OpenAI GPT)  
   - *Role:* Backend language model node specified as GPT-4o, used by `AI Agent` for processing.  
   - *Configuration:* Uses GPT-4o model with OpenAI API credentials.  
   - *Inputs:* From `AI Agent` internally  
   - *Outputs:* Feeds back to `AI Agent`  
   - *Version:* 1.2  
   - *Edge cases:* API key invalidation, network errors, model unavailability.

4. **Prepare Excel**  
   - *Type:* Code node (JavaScript)  
   - *Role:* Parses the AI agent’s JSON output to map each trend into an Excel row structure with columns: Rank, Trend, Description, Platforms, and Date.  
   - *Configuration:* Attempts to parse JSON from AI output; falls back to a default trend if parsing fails.  
   - *Inputs:* `AI Agent`  
   - *Outputs:* `Create Excel`  
   - *Version:* v2  
   - *Edge cases:* JSON parse errors, missing AI output, or unexpected schema.

- **Sticky Note Content:**  
  *Describes the aggregation and AI ranking functions, required OpenAI API credentials, and tips to customize the AI prompt or aggregation limits.*

#### 2.3 Spreadsheet Creation Branch

- **Overview:**  
Transforms the AI-generated trend data rows into an Excel spreadsheet file.

- **Nodes Involved:**  
  - `Create Excel`  

- **Node Details:**

1. **Create Excel**  
   - *Type:* Spreadsheet File node  
   - *Role:* Converts the JSON rows into an Excel `.xlsx` file.  
   - *Configuration:* Uses default options to output an Excel file to be passed downstream for storage.  
   - *Inputs:* `Prepare Excel`  
   - *Outputs:* `Microsoft SharePoint`  
   - *Version:* v2  
   - *Edge cases:* Large datasets can cause memory issues; misconfigured input may lead to invalid files.

- **Sticky Note Content:**  
  *Explains this branch’s role in generating the spreadsheet, notes no credentials required, and suggests options to swap Excel with CSV or Google Sheets nodes.*

#### 2.4 Storage Branch

- **Overview:**  
Uploads the generated Excel file to a specified folder on Microsoft SharePoint.

- **Nodes Involved:**  
  - `Microsoft SharePoint`  

- **Node Details:**

1. **Microsoft SharePoint**  
   - *Type:* Microsoft SharePoint node  
   - *Role:* Uploads the Excel file with name `social-media-trends.xls` to a configured SharePoint site and folder.  
   - *Configuration:* Site and folder specified by IDs and names; operation set to upload; file contents received from `Create Excel`.  
   - *Inputs:* `Create Excel`  
   - *Outputs:* None (terminal node)  
   - *Version:* v1  
   - *Edge cases:* OAuth token expiration, permission denied, folder or site ID incorrect, file size limits.

- **Sticky Note Content:**  
  *Details upload function and credentials required, advises on changing folder ID or replacing SharePoint with other storage services.*

---

### 3. Summary Table

| Node Name                   | Node Type                       | Functional Role                          | Input Node(s)               | Output Node(s)           | Sticky Note                                                                                                                |
|-----------------------------|--------------------------------|----------------------------------------|-----------------------------|--------------------------|----------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                 | Starts the workflow                     | None                        | Set Config               |                                                                                                                            |
| Set Config                  | Set                            | Defines date and topic variables        | When clicking ‘Execute workflow’ | Reddit API, Get Twitter Trends |                                                                                                                            |
| Reddit API                  | HTTP Request                   | Fetches hot posts from Reddit subreddit | Set Config                  | Merge Data               | Data Sources Branch: Calls Reddit hot endpoint, requires Reddit OAuth2 credentials, customizable subreddit                 |
| Get Twitter Trends          | HTTP Request                   | Calls custom Twitter trends service     | Set Config                  | Merge Data               | Data Sources Branch: Calls Twitter trends endpoint, customizable, requires relevant credentials                            |
| Merge Data                  | Merge                          | Merges Reddit and Twitter data          | Reddit API, Get Twitter Trends | Aggregate Content        | Data Sources Branch: Combines multiple social media sources                                                                |
| Aggregate Content           | Code                           | Extracts and concatenates top Reddit posts and Twitter trends | Merge Data                  | AI Agent                 | AI Analysis Branch: Aggregates content for AI, tweak slice limits and prompt                                              |
| AI Agent                   | LangChain Agent                | Analyzes trends with GPT-4o, outputs ranked JSON | Aggregate Content           | Prepare Excel            | AI Analysis Branch: Runs GPT-4o with custom prompt, requires OpenAI API key                                                |
| OpenAI Chat Model           | LangChain OpenAI Chat Model   | Provides GPT-4o language model          | AI Agent (internal)          | AI Agent (internal)       | AI Analysis Branch: Backend LM for AI Agent                                                                                |
| Prepare Excel               | Code                           | Parses AI JSON output into Excel rows   | AI Agent                    | Create Excel             | Spreadsheet Creation Branch: Prepares rows, handles JSON parse fallback                                                    |
| Create Excel                | Spreadsheet File               | Converts rows to Excel .xlsx file       | Prepare Excel               | Microsoft SharePoint     | Spreadsheet Creation Branch: Creates Excel file, no credentials needed, can swap to CSV or Sheets                          |
| Microsoft SharePoint        | Microsoft SharePoint           | Uploads Excel file to SharePoint folder | Create Excel                | None                     | Storage Branch: Uploads file, requires SharePoint OAuth2, customizable folder and filename                                  |
| Sticky Note                 | Sticky Note                   | Documentation note                      | None                        | None                     | Data Sources Branch description                                                                                           |
| Sticky Note1                | Sticky Note                   | Documentation note                      | None                        | None                     | AI Analysis Branch description                                                                                            |
| Sticky Note2                | Sticky Note                   | Documentation note                      | None                        | None                     | Spreadsheet Creation Branch description                                                                                   |
| Sticky Note3                | Sticky Note                   | Documentation note                      | None                        | None                     | Storage Branch description                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `When clicking ‘Execute workflow’`  
   - Purpose: Entry point to start workflow execution.

2. **Create Set Node**  
   - Name: `Set Config`  
   - Assign two variables:  
     - `dateToday`: Use expression `={{ $now.setLocale('en').toFormat('yyyy-MM-dd') }}`  
     - `topic`: Set as `"Artificial Intelligence"` (string)  
   - Connect input from Manual Trigger.

3. **Create HTTP Request Node for Reddit**  
   - Name: `Reddit API`  
   - Method: GET  
   - URL: `https://oauth.reddit.com/r/artificial/hot`  
   - Authentication: OAuth2 with Reddit credentials (configure Reddit OAuth2 API credentials)  
   - Connect input from `Set Config`.

4. **Create HTTP Request Node for Twitter Trends**  
   - Name: `Get Twitter Trends`  
   - Method: POST  
   - URL: `<YOUR_SERVICE_URL>` (replace with actual Twitter trends endpoint)  
   - Body type: JSON  
   - Body content:  
     ```json
     {
       "workflow_to_run": "python <PATH_TO_YOUR_SCRIPT>\\twitter_bot_dynamic.py --topic \"{{ $json.topic }}\""
     }
     ```  
   - Allow Unauthorized Certificates: enabled (only if necessary)  
   - Connect input from `Set Config`.

5. **Create Merge Node**  
   - Name: `Merge Data`  
   - Mode: Default merge (combine inputs)  
   - Connect inputs from both `Reddit API` and `Get Twitter Trends`.

6. **Create Code Node to Aggregate Content**  
   - Name: `Aggregate Content`  
   - Language: JavaScript  
   - Paste the following logic (interpreted from workflow):  
     - Iterate through all inputs; for Reddit data, extract top 3 post titles from `data.children` array.  
     - Concatenate titles with numbering and line breaks.  
     - Append Twitter trends data from `Get Twitter Trends` output.  
     - Output a JSON object with `aggregatedContent` and Twitter output.  
   - Connect input from `Merge Data`.

7. **Create LangChain Agent Node**  
   - Name: `AI Agent`  
   - Prompt:  
     ```
     Analyze these social media trends and extract the top 5 trends in JSON format:
     
     TRENDS FROM REDDIT:
     {{$json.aggregatedContent}}
     
     TRENDS FROM X:
     {{ $json.output }}
     
     Respond with:
     {
       "trends": [
         {"rank": 1, "trend": "Trend Name", "description": "Description", "platforms": ["Platform1"]}
       ]
     }
     ```  
   - Model: Connect to LangChain OpenAI Chat Model node using GPT-4o  
   - Connect input from `Aggregate Content`.

8. **Create LangChain OpenAI Chat Model Node**  
   - Name: `OpenAI Chat Model`  
   - Model: `gpt-4o`  
   - Credentials: OpenAI API key (configure with valid OpenAI credentials)  
   - Connect as language model backend to `AI Agent`.

9. **Create Code Node to Prepare Excel Rows**  
   - Name: `Prepare Excel`  
   - Language: JavaScript  
   - Logic:  
     - Parse the JSON output from `AI Agent`.  
     - If JSON parsing fails, default to a fallback trend.  
     - Map each trend to a JSON object with columns: Rank, Trend, Description, Platforms (joined string), Date (from `Set Config`).  
   - Connect input from `AI Agent`.

10. **Create Spreadsheet File Node**  
    - Name: `Create Excel`  
    - Operation: `toFile`  
    - Purpose: Convert JSON rows into Excel `.xlsx` file  
    - Connect input from `Prepare Excel`.

11. **Create Microsoft SharePoint Node**  
    - Name: `Microsoft SharePoint`  
    - Operation: `upload`  
    - Configure Site: Select site by domain and ID  
    - Configure Folder: Select folder by ID (e.g., Newsletter Workflow folder)  
    - File Name: `"social-media-trends.xls"`  
    - File Contents: Pass the file data from `Create Excel` node  
    - Credentials: Microsoft SharePoint OAuth2 credentials  
    - Connect input from `Create Excel`.

12. **Validate all connections:**  
    Manual Trigger → Set Config → Reddit API → Merge Data  
    Set Config → Get Twitter Trends → Merge Data  
    Merge Data → Aggregate Content → AI Agent → Prepare Excel → Create Excel → Microsoft SharePoint  
    AI Agent uses OpenAI Chat Model as language model backend.

13. **Additional setup:**  
    - Configure all OAuth2 credentials properly (Reddit, OpenAI, SharePoint).  
    - Replace placeholder URLs and script paths with actual endpoints.  
    - Optionally add nodes for other social media data sources before `Merge Data`.  
    - Test each node separately to verify connectivity and output.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                           | Context or Link                                                                                         |
|--------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| Data Sources Branch can be extended with YouTube, TikTok, or other social API nodes before merging.                                                    | Sticky Note on Data Sources Branch                                                                     |
| Edit AI Agent prompt to customize ranking logic, number of trends extracted, or JSON output schema.                                                    | Sticky Note on AI Analysis Branch                                                                      |
| Swap Create Excel node for CSV or Google Sheets nodes if preferred for output format.                                                                  | Sticky Note on Spreadsheet Creation Branch                                                             |
| Replace Microsoft SharePoint node with alternative storage providers like Google Drive, Dropbox, or AWS S3 if needed.                                | Sticky Note on Storage Branch                                                                           |
| Ensure all API credentials are correctly set up and authorized, especially OAuth2 tokens that may expire.                                              | General best practice                                                                                   |
| The Twitter trends node uses a custom backend service running a Python script; ensure this service is operational and secure.                         | Twitter Trends HTTP Request node notes                                                                 |
| This workflow was tagged under "Marketing" and designed for content creators and social media analysts to automate trend tracking and reporting.       | Workflow metadata                                                                                       |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected content. All manipulated data is legal and public.

---