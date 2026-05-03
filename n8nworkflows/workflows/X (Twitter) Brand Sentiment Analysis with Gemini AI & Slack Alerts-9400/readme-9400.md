X (Twitter) Brand Sentiment Analysis with Gemini AI & Slack Alerts

https://n8nworkflows.xyz/workflows/x--twitter--brand-sentiment-analysis-with-gemini-ai---slack-alerts-9400


# X (Twitter) Brand Sentiment Analysis with Gemini AI & Slack Alerts

### 1. Workflow Overview

This n8n workflow automates brand sentiment analysis for Twitter (X) mentions related to a target keyword ("emergentlabs"). It fetches recent tweets, stores raw data, performs AI-driven sentiment and topic analysis, aggregates insights, and issues Slack alerts for urgent mentions. It is designed for marketing, customer support, or brand monitoring teams to maintain real-time awareness and respond effectively.

**Logical Blocks:**

- **1.1 Initialization & Tweet Collection Loop:**  
  Initializes counters and search parameters, then loops through Twitter API calls to collect batches of tweets mentioning the brand. Cleans and appends raw tweet data to a Google Sheets database.

- **1.2 Tweet Presence Check & AI Analysis Trigger:**  
  Verifies if new tweets were collected. If yes, triggers an AI-powered analysis agent that reads tweets from the database and produces detailed sentiment, topic, urgency, and summary data.

- **1.3 AI Analysis Parsing & Storage:**  
  Parses AI textual output into structured data and appends it to a secondary Google Sheets database holding enriched tweet analyses.

- **1.4 Strategic Summary & Urgent Alerts Generation:**  
  Reads unanalyzed AI analysis rows, compiles a briefing book, and runs a second AI agent to generate a high-level summary and a JSON list of urgent actionable tweets.

- **1.5 Slack Notification Dispatch:**  
  Posts the daily strategic briefing to a Slack channel and sends individual urgent alerts to a dedicated Slack channel for tweets flagged as requiring immediate attention.

---

### 2. Block-by-Block Analysis

#### 1.1 Initialization & Tweet Collection Loop

- **Overview:**  
  Sets up the initial parameters and counters, then repeatedly calls the Twitter API to collect tweets matching the query. Cleans raw Twitter data and appends each tweet as a new row in a Google Sheet.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Count (Set)  
  - counter (Set)  
  - Wait  
  - Tweet Scraper (HTTP Request)  
  - Code (Code)  
  - Append row in sheet (Google Sheets)  
  - If (If)  
  - Limit (Limit)  
  - set increase (Set)  
  - Code1 (Code)  
  - set count and cursor (Set)  

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Trigger  
    - Role: Starts the workflow at regular intervals (default: every minute).  
    - Config: Default interval (can be customized).  
    - Failure Modes: Trigger misfires if n8n server is down.

  - **Count (Set)**  
    - Type: Set  
    - Role: Initializes `count=1` and sets the search query string `"emergentlabs"`.  
    - Key variables: `count`, `query`  
    - Input: None (trigger input)  
    - Output: Initializes first page count for pagination.

  - **counter (Set)**  
    - Type: Set  
    - Role: Sets `counter` and `cursor` from incoming data (initially starts from `count=1` and no cursor).  
    - Key variables: `counter`, `cursor`

  - **Wait**  
    - Type: Wait  
    - Role: Adds delay between paginated API calls to respect rate limits (2 seconds).  
    - Input: Triggered by counter node output.  
    - Output: Passes control to Tweet Scraper.

  - **Tweet Scraper (HTTP Request)**  
    - Type: HTTP Request  
    - Role: Calls Twitter API endpoint for advanced search with pagination cursor and query string.  
    - Key config:  
      - URL: https://api.twitterapi.io/twitter/tweet/advanced_search  
      - Query params: `query` (from Count node), `queryType=Latest`, `cursor` (pagination cursor)  
      - Auth: HTTP header auth with Twitter API credentials  
    - Failure modes: API auth errors, rate limiting, network timeouts.

  - **Code (Code)**  
    - Type: Code  
    - Role: Processes raw tweets from API response; extracts relevant fields and formats each tweet into a standardized JSON structure for n8n.  
    - Key code actions: Extracts tweet details (id, text, url, author info, counts, reply flags), includes pagination cursor for next iteration.  
    - Output: Array of cleaned tweet items, each representing one tweet.

  - **Append row in sheet (Google Sheets)**  
    - Type: Google Sheets Append  
    - Role: Appends each cleaned tweet as a new row to the "Raw Tweets" Google Sheet.  
    - Key config:  
      - Spreadsheet ID and Sheet Name (Sheet1)  
      - Columns mapped from tweet JSON fields  
      - Use Append operation  
      - Credentials: Google Sheets OAuth2  
    - Failure modes: Auth failure, quota limits, spreadsheet locked.

  - **If (If)**  
    - Type: If  
    - Role: Checks if `counter` equals 1. If true, triggers aggregation; else, triggers limit node.  
    - Logic: Controls loop exit condition.

  - **Limit (Limit)**  
    - Type: Limit  
    - Role: Limits number of items processed per batch (default 1).  
    - Controls flow to set increase node.

  - **set increase (Set)**  
    - Type: Set  
    - Role: Passes current `counter` and `cursor` forward for incrementing.

  - **Code1 (Code)**  
    - Type: Code  
    - Role: Increments the `count` variable by 1, ensuring pagination continues correctly.  
    - Logic: Parses counter string to integer, increments, and returns updated count.

  - **set count and cursor (Set)**  
    - Type: Set  
    - Role: Updates `count` and `cursor` for next iteration of the loop.

#### 1.2 Tweet Presence Check & AI Analysis Trigger

- **Overview:**  
  After tweets are collected and saved, this block aggregates all new tweets, checks if any exist, and if so, triggers the AI analyst node.

- **Nodes Involved:**  
  - Aggregate (Aggregate)  
  - Switch (Switch)  
  - AI Analyst (@n8n/n8n-nodes-langchain.agent)  
  - getTweetsFromDatabase (Google Sheets Tool)  

- **Node Details:**

  - **Aggregate**  
    - Type: Aggregate  
    - Role: Aggregates tweet contents into a single field for AI input.  
    - Config: Aggregates "Content" field, renaming output to "final".

  - **Switch**  
    - Type: Switch  
    - Role: Acts as gatekeeper, checking if aggregated tweet content exists. If no tweets, workflow stops; if tweets present, proceeds to AI Analyst.  
    - Failure modes: Empty input causes no further processing.

  - **AI Analyst**  
    - Type: Langchain Agent  
    - Role: Performs AI-driven analysis on tweets fetched from the database, extracting sentiment, key topic, urgency, summary, and tweet id.  
    - Prompt: Explicit instructions with formatting rules to return only structured data with double colon separators and triple dash separators between tweets.  
    - Credentials: Google Gemini (PaLM) API  
    - Failure modes: API rate limits, malformed output, timeout.

  - **getTweetsFromDatabase**  
    - Type: Google Sheets Tool  
    - Role: Reads all tweets from the "Raw Tweets" Google Sheet to supply data for AI analysis.  
    - Filters: None, fetches all rows.  
    - Credentials: Google Sheets OAuth2

#### 1.3 AI Analysis Parsing & Storage

- **Overview:**  
  Parses the AI agent’s textual output into structured JSON objects, then appends each analysis as a new row in the "AI Analysis" Google Sheet.

- **Nodes Involved:**  
  - Translator (Code)  
  - Append row in sheet1 (Google Sheets)  

- **Node Details:**

  - **Translator (Code)**  
    - Type: Code  
    - Role: Parses the AI output text block by splitting on `---` separators and extracting keys and values using `::` delimiters. Converts keys to camelCase and returns array of JSON objects.  
    - Failure modes: Unexpected AI output format may cause parsing errors.

  - **Append row in sheet1**  
    - Type: Google Sheets Append  
    - Role: Appends structured AI analysis data to the 'AI Analysis' sheet (Sheet2).  
    - Columns: sentiment, keyTopic, urgency, summary, tweetId, action taken (default "Notmarked"), date_analyzed (current timestamp)  
    - Credentials: Google Sheets OAuth2

#### 1.4 Strategic Summary & Urgent Alerts Generation

- **Overview:**  
  Reads unanalyzed rows from the "AI Analysis" sheet, compiles a formatted text briefing book, and runs the "Strategist Agent" AI to generate a human-readable summary and a JSON array of urgent tweets requiring action.

- **Nodes Involved:**  
  - Schedule Trigger1 (Scheduled Trigger)  
  - Get row(s) in sheet (Google Sheets)  
  - parser (Code)  
  - Strategist Agent (@n8n/n8n-nodes-langchain.chainLlm)  

- **Node Details:**

  - **Schedule Trigger1**  
    - Type: Scheduled Trigger  
    - Role: Runs once daily at 9 AM.  
    - Failure: Trigger misfires if n8n server down.

  - **Get row(s) in sheet**  
    - Type: Google Sheets Read  
    - Role: Filters and fetches rows from AI Analysis sheet where "action taken" = "Notmarked" to avoid duplicate processing.  
    - Credentials: Google Sheets OAuth2

  - **parser (Code)**  
    - Type: Code  
    - Role: Combines all fetched tweet analyses into one formatted text block for AI input, while tracking row numbers for later updates.  
    - Output: `preparedText` (for AI), `rowsToUpdate`

  - **Strategist Agent**  
    - Type: Langchain Chain LLM  
    - Role: Generates a two-part report: a strategic summary and a JSON array of urgent tweets (urgency High or Medium), with reasons.  
    - Batch size: 1  
    - Credentials: Google Gemini (PaLM) API  
    - Failure modes: API errors, invalid JSON output.

#### 1.5 Slack Notification Dispatch

- **Overview:**  
  Processes the strategist agent’s output to post the daily summary to a Slack channel and send urgent alerts individually to a dedicated Slack channel.

- **Nodes Involved:**  
  - summary (Code)  
  - Send a message (Slack)  
  - parseAlerts (Code)  
  - Loop Over Items (SplitInBatches)  
  - Send a message1 (Slack)  

- **Node Details:**

  - **summary (Code)**  
    - Type: Code  
    - Role: Extracts the text portion of the AI response before the JSON delimiter to isolate the summary report.  
    - Output: `summaryReport`

  - **Send a message**  
    - Type: Slack  
    - Role: Sends the formatted daily strategic briefing report to the Slack channel "brand-alerts".  
    - Auth: Slack OAuth2  
    - Failure modes: Slack API rate limits, auth errors.

  - **parseAlerts (Code)**  
    - Type: Code  
    - Role: Extracts the JSON block from the AI output, parses it, and returns each urgent tweet as a separate item for further processing.  
    - Failure modes: JSON parse errors if AI output malformed.

  - **Loop Over Items (SplitInBatches)**  
    - Type: SplitInBatches  
    - Role: Processes each urgent alert individually to send separate Slack messages.

  - **Send a message1 (Slack)**  
    - Type: Slack  
    - Role: Sends an urgent alert message about a specific tweet to the "urgent" Slack channel, including urgency, reason, tweet ID, and direct link.  
    - Auth: Slack OAuth2

---

### 3. Summary Table

| Node Name             | Node Type                          | Functional Role                                      | Input Node(s)                   | Output Node(s)                | Sticky Note                                                                                                                           |
|-----------------------|----------------------------------|-----------------------------------------------------|--------------------------------|------------------------------|----------------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger       | Schedule Trigger                 | Starts tweet collection workflow                     | —                              | Count                        | # 🚀 Starting the Engine: initializes counter and search term                                                                          |
| Count                 | Set                              | Initializes count and search query                   | Schedule Trigger               | counter                      | # 🚀 Starting the Engine                                                                                                               |
| counter               | Set                              | Holds current counter and cursor for pagination     | Count                         | Wait                         | # 🚀 Starting the Engine                                                                                                               |
| Wait                  | Wait                             | Delays between API calls to respect rate limits     | counter                       | Tweet Scraper                | # 🐦 The Tweet Collector (Loop): loops through Twitter API calls                                                                       |
| Tweet Scraper          | HTTP Request                    | Calls Twitter API to fetch tweets                    | Wait                         | Code                         | # 🐦 The Tweet Collector (Loop)                                                                                                       |
| Code                  | Code                             | Cleans and structures raw tweet data                 | Tweet Scraper                 | Append row in sheet          | # 🐦 The Tweet Collector (Loop)                                                                                                       |
| Append row in sheet    | Google Sheets Append             | Appends cleaned tweets to "Raw Tweets" sheet        | Code                         | If                           | # 🐦 The Tweet Collector (Loop)                                                                                                       |
| If                    | If                               | Checks if first loop count to branch logic           | Append row in sheet           | Aggregate, Limit             | # 🤔 The Gatekeeper: checks if new tweets exist                                                                                       |
| Aggregate             | Aggregate                        | Aggregates tweet content for AI input                 | If                           | Switch                       | # 🤔 The Gatekeeper                                                                                                                    |
| Switch                 | Switch                          | Gates workflow, proceeds if tweets found             | Aggregate                    | AI Analyst                   | # 🤔 The Gatekeeper                                                                                                                    |
| getTweetsFromDatabase  | Google Sheets Tool               | Reads raw tweets from Google Sheets                   | —                            | AI Analyst (ai_tool)         | # 🧠 The AI Analyst: AI reads raw tweets                                                                                             |
| AI Analyst             | Langchain Agent                  | Performs sentiment, topic, urgency, summary analysis | Switch, getTweetsFromDatabase | Translator                   | # 🧠 The AI Analyst                                                                                                                    |
| Translator             | Code                             | Parses AI text output into structured JSON            | AI Analyst                   | Append row in sheet1         | # 🧹 The Translator: parses AI output into structured data                                                                            |
| Append row in sheet1   | Google Sheets Append             | Saves AI analyses to "AI Analysis" sheet              | Translator                   | —                            | # 💾 System Memory: saves analyses permanently                                                                                        |
| Schedule Trigger1      | Schedule Trigger                 | Triggers daily strategic summary workflow             | —                            | Get row(s) in sheet          | # 🎯 The Strategist Workflow: intelligence director                                                                                  |
| Get row(s) in sheet    | Google Sheets                   | Reads unanalyzed AI analysis rows                      | Schedule Trigger1            | parser                       | # 📊 1. Reading the "To-Do" List                                                                                                      |
| parser                 | Code                             | Combines rows into briefing book text for AI          | Get row(s) in sheet          | Strategist Agent             | # 📚 2. Compiling the Briefing Book                                                                                                  |
| Strategist Agent       | Langchain Chain LLM             | Generates summary report and JSON list of urgent items | parser                      | summary, parseAlerts         | # 👑 3. The Strategist AI                                                                                                             |
| summary                | Code                             | Extracts text summary from AI output                   | Strategist Agent             | Send a message               | # 📜 4a. The Summary Report Path                                                                                                      |
| Send a message         | Slack                           | Sends daily strategic briefing to Slack channel       | summary                      | —                            | # 📜 4a. The Summary Report Path                                                                                                      |
| parseAlerts            | Code                             | Extracts JSON array of urgent tweets from AI output   | Strategist Agent             | Loop Over Items              | # 🚨 4b. The Urgent Alerts Path                                                                                                       |
| Loop Over Items        | SplitInBatches                  | Processes each urgent alert one by one                 | parseAlerts                  | Send a message1              | # 🚨 4b. The Urgent Alerts Path                                                                                                       |
| Send a message1        | Slack                           | Sends urgent alert messages to Slack                   | Loop Over Items              | Loop Over Items (loop)       | # 🚨 4b. The Urgent Alerts Path                                                                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger** (Schedule Trigger Node)  
   - Configure to run as frequently as desired (e.g., every minute).  
   - No parameters needed.

2. **Create Count Node** (Set Node)  
   - Set `count` to 1 (number).  
   - Set `query` to `"emergentlabs"` (string).

3. **Create counter Node** (Set Node)  
   - Set `counter` to `={{ $json.count }}` (string).  
   - Set `cursor` to `={{ $json['next cursor'] }}` (string or empty initially).

4. **Create Wait Node** (Wait Node)  
   - No parameters; used to delay between API calls.

5. **Create Tweet Scraper Node** (HTTP Request)  
   - URL: `https://api.twitterapi.io/twitter/tweet/advanced_search`  
   - Method: GET  
   - Query Parameters:  
     - `query` = `={{ $('Count').item.json.query }}`  
     - `queryType` = `Latest`  
     - `cursor` = `={{ $json.cursor }}`  
   - Authentication: HTTP Header with Twitter API Key credentials.

6. **Create Code Node (Clean Tweets)**  
   - Paste JS code to iterate over `inputData.tweets`, extract relevant fields, and return array of cleaned tweet JSON objects.

7. **Create Append row in sheet Node** (Google Sheets)  
   - Operation: Append  
   - Document ID: Your Google Sheet with raw tweets  
   - Sheet Name: Sheet1 (or your raw tweet sheet)  
   - Map columns to tweet JSON fields (URL, Date, Likes, Views, Content, Replies, Retweets, Tweet_ID, Author_Name, Author_Username, next_cursor, isreply, Quotes)  
   - Credentials: Google Sheets OAuth2.

8. **Create If Node**  
   - Condition: Check if `counter == 1`.  
   - True output connects to Aggregate node.  
   - False output connects to Limit node.

9. **Create Aggregate Node**  
   - Aggregates `Content` field into a single string named `final`.

10. **Create Limit Node**  
    - No parameters; limits batch size to 1.

11. **Create set increase Node** (Set Node)  
    - Set `counter` to `={{ $('counter').item.json.counter }}`  
    - Set `cursor` to `={{ $('Tweet Scraper').item.json.next_cursor }}`

12. **Create Code1 Node** (Increment Counter)  
    - Paste JS code to increment count by 1.

13. **Create set count and cursor Node** (Set Node)  
    - Set `count` to `={{ $json.count }}`  
    - Set `cursor` to `={{ $json.cursor }}`

14. **Link nodes to form a loop:**  
    - Schedule Trigger → Count → counter → Wait → Tweet Scraper → Code → Append row in sheet → If → Aggregate / Limit → set increase → Code1 → set count and cursor → counter (loop back).

15. **Create Switch Node**  
    - Input: Aggregate output  
    - Condition: Check if `Content` exists (not empty).  
    - True output connects to AI Analyst.

16. **Create getTweetsFromDatabase Node** (Google Sheets Tool)  
    - Read all rows from the raw tweets Google Sheet.

17. **Create AI Analyst Node** (Langchain Agent)  
    - Prompt with instructions to analyze tweets for Sentiment, Key Topic, Urgency, Summary, Tweet ID.  
    - Credentials: Google Gemini (PaLM) API.

18. **Connect Switch true output and getTweetsFromDatabase as AI tool input to AI Analyst**.

19. **Create Translator Node** (Code)  
    - JS code to parse AI text output into structured JSON objects.

20. **Create Append row in sheet1 Node** (Google Sheets)  
    - Append AI analysis output to "AI Analysis" sheet (Sheet2).  
    - Define columns: tweetId, sentiment, keyTopic, urgency, summary, action taken (default "Notmarked"), date_analyzed (current timestamp).  
    - Credentials: Google Sheets OAuth2.

21. **Connect AI Analyst → Translator → Append row in sheet1**.

22. **Create Schedule Trigger1 Node** (Scheduled Trigger)  
    - Set to run at 9 AM daily.

23. **Create Get row(s) in sheet Node** (Google Sheets)  
    - Reads rows from AI Analysis sheet where `action taken` = "Notmarked".

24. **Create parser Node** (Code)  
    - Concatenates rows into formatted text briefing book and tracks row numbers.

25. **Create Strategist Agent Node** (Langchain Chain LLM)  
    - Prompt to generate:  
      - Part 1: Summary report (Overall Sentiment, Key Themes, Testimonials, Most Influential Post).  
      - Part 2: JSON array of urgent tweets (urgency High or Medium).  
    - Credentials: Google Gemini.

26. **Connect Get row(s) in sheet → parser → Strategist Agent**.

27. **Create summary Node** (Code)  
    - Extracts text summary portion from Strategist Agent output before JSON delimiter.

28. **Create Send a message Node** (Slack)  
    - Sends summary report to Slack channel "brand-alerts".  
    - Credentials: Slack OAuth2.

29. **Create parseAlerts Node** (Code)  
    - Extracts JSON array of urgent tweets from Strategist Agent output.

30. **Create Loop Over Items Node** (SplitInBatches)  
    - Batch size 1 to process each urgent tweet.

31. **Create Send a message1 Node** (Slack)  
    - Sends individual urgent alert messages to Slack channel "urgent".  
    - Includes urgency, reason, tweetId, and direct tweet link.

32. **Connect Strategist Agent → summary → Send a message** (summary report path).  
    - Connect Strategist Agent → parseAlerts → Loop Over Items → Send a message1 → Loop Over Items (loop) (urgent alerts path).

---

### 5. General Notes & Resources

| Note Content                                                                                                                            | Context or Link                                                                                                  |
|-----------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| The workflow is architected as the core of a “Brand Sentry” system to monitor Twitter mentions and generate actionable brand insights. | Sticky Note at workflow start explains this is the heart of the brand monitoring system.                         |
| Twitter API credentials require proper authorization and rate limit handling.                                                           | Twitter API v2 or compatible with https://api.twitterapi.io/ advanced search endpoint.                           |
| Google Sheets serves as persistent storage for raw tweets and AI analyses; OAuth2 credentials needed.                                   | Google Sheets OAuth2 credentials required for both sheets.                                                     |
| Google Gemini (PaLM) API powers AI nodes; credentials must be configured securely.                                                       | Langchain Google Gemini nodes require Google PaLM API credentials.                                             |
| Slack OAuth2 credentials must be set up for Slack message nodes; channels “brand-alerts” and “urgent” must exist in Slack workspace.    | Slack OAuth2 integration needed; channel IDs preconfigured.                                                    |
| The workflow uses pagination with cursor and a counter to fetch multiple pages of tweets; appropriate error handling for API limits is recommended. | Potential edge cases include API failures, empty result sets, malformed AI outputs, and Slack API rate limits.  |
| The AI prompt formatting requires strict adherence to double colon `::` separators and triple dash `---` to parse correctly.            | Failure to maintain format will cause parsing errors downstream.                                               |
| The daily summary and urgent alerts scheduling can be adjusted as per business needs.                                                    | Schedule Trigger1 is set to 9 AM but can be customized.                                                        |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, adhering strictly to content policies without containing illegal, offensive, or protected elements. All processed data is legal and publicly available.