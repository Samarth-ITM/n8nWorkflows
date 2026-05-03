Monitor Twitter Accounts & Generate Intelligence Summaries with Gemini AI & Telegram

https://n8nworkflows.xyz/workflows/monitor-twitter-accounts---generate-intelligence-summaries-with-gemini-ai---telegram-7316


# Monitor Twitter Accounts & Generate Intelligence Summaries with Gemini AI & Telegram

### 1. Workflow Overview

This workflow, titled **"Twitter Intelligence Analysis"**, automates the monitoring of specified Twitter accounts, leveraging AI-powered analysis to generate intelligence summaries from new tweets and delivering these insights via Telegram messages. The workflow is designed for users who want to keep informed about specific Twitter accounts with enhanced contextual analysis, filtering, and categorized alerts.

The workflow is logically organized into the following blocks:

- **1.1 Scheduled Trigger & Configuration Loading:** Automatically triggers the workflow at set intervals and reads monitoring configurations from a PostgreSQL database.
- **1.2 Tweet Retrieval via RSS Feed:** Fetches recent tweets from specified accounts using an RSSHub instance.
- **1.3 Filtering New Tweets:** Filters tweets to only process those published after the last recorded update time.
- **1.4 Conditional Flow Control:** Checks if new tweets exist to continue or stop the workflow.
- **1.5 AI Analysis Pipeline:** Sends new tweets to Google Gemini AI for in-depth analysis, parses the AI’s JSON response, and filters the results by intelligence level.
- **1.6 Notification Dispatch:** Sends filtered intelligence summaries to a Telegram chat.
- **1.7 State Update:** Identifies the latest processed tweet timestamp and updates the database to prevent reprocessing.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger & Configuration Loading

- **Overview:**  
  This block initiates the workflow at a fixed interval and retrieves monitoring configurations for Twitter accounts from a PostgreSQL database.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Config

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Initiates the workflow automatically on a set time interval (minutes-based).  
    - Configuration: Triggers every X minutes (configurable; example suggests every 15 minutes is recommended).  
    - Input/Output: No inputs; outputs trigger signal.  
    - Failures: None typical; ensure n8n instance uptime.  
    - Sticky Note: Explains interval setup.

  - **Config**  
    - Type: PostgreSQL Query  
    - Role: Reads configuration data including Twitter accounts to monitor, last update timestamps, AI prompts, and notes from the `twitter_following_config` table.  
    - Configuration: Executes SQL query to fetch row with `id = 1`.  
    - Input: Trigger from Schedule Trigger.  
    - Output: JSON object with configuration fields: twitter_account, last_update_time, prompt, note.  
    - Failures: DB connection issues, query failures, empty results.  
    - Sticky Note: Describes expected DB table structure and fields.

#### 2.2 Tweet Retrieval via RSS Feed

- **Overview:**  
  Fetches recent tweets from the configured Twitter account using an RSS feed provided by a custom RSSHub instance.

- **Nodes Involved:**  
  - RSS

- **Node Details:**

  - **RSS**  
    - Type: RSS Feed Read  
    - Role: Reads the latest tweets as RSS items from a URL templated with the Twitter account handle.  
    - Configuration: URL constructed dynamically using `twitter_account` field from Config node.  
    - Input: Configuration data.  
    - Output: List of tweet items with metadata including ISO publication date.  
    - Failures: Network issues, invalid RSS URL, empty feed.  
    - Sticky Note: Provides link to RSSHub Twitter route documentation; notes requirement of self-hosted RSSHub.

#### 2.3 Filtering New Tweets

- **Overview:**  
  Filters the fetched tweets to only include those published after the last recorded update timestamp to avoid duplicate processing.

- **Nodes Involved:**  
  - Filter New Tweets

- **Node Details:**

  - **Filter New Tweets**  
    - Type: Filter  
    - Role: Compares each tweet’s `isoDate` with the `last_update_time` from Config, passing only newer tweets forward.  
    - Configuration: Condition defined with date/time operation 'after'.  
    - Input: RSS feed tweets.  
    - Output: Filtered tweets (new only).  
    - Failures: Expression errors if dates malformed, empty input leads to no output.  
    - Sticky Note: Explains filtering logic per tweet publication date.

#### 2.4 Conditional Flow Control

- **Overview:**  
  Determines whether new tweets exist to proceed with analysis or stop workflow execution.

- **Nodes Involved:**  
  - Check if New Tweets Exist

- **Node Details:**

  - **Check if New Tweets Exist**  
    - Type: If condition  
    - Role: Checks if output array from `Filter New Tweets` is not empty.  
    - Configuration: Condition to test array non-emptiness.  
    - Input: Filtered new tweets.  
    - Output: Two branches — True (proceed), False (stop).  
    - Failures: None expected, but empty input leads to stopping workflow.  
    - Sticky Note: Explains stopping condition if no new tweets.

#### 2.5 AI Analysis Pipeline

- **Overview:**  
  Processes each new tweet through Google Gemini AI to generate an intelligence summary, parses the AI response for structured JSON output, and filters analysis by intelligence level.

- **Nodes Involved:**  
  - Analyze Tweet with AI  
  - Parse AI Response  
  - Filter by Level

- **Node Details:**

  - **Analyze Tweet with AI**  
    - Type: Google Gemini (Langchain)  
    - Role: Sends tweet content with a configurable prompt to Gemini AI model `gemini-2.5-flash` for analysis.  
    - Configuration:  
      - Text input dynamically built combining prompt (from Config) and tweet details (title, content, link).  
      - Model ID fixed to `models/gemini-2.5-flash`.  
      - Resource type set to `document`.  
    - Input: New tweets passing conditional check.  
    - Output: AI text response containing analysis embedded in JSON block.  
    - Failures: API authentication, rate limits, model unavailability, malformed prompts.  
    - Version: TypeVersion 1.

  - **Parse AI Response**  
    - Type: Code (JavaScript)  
    - Role: Extracts JSON object from the AI's text response using regex and parses it into usable JSON.  
    - Configuration: Runs once per item; returns error JSON if parsing fails or no JSON found.  
    - Input: AI raw response text.  
    - Output: Parsed JSON analysis or error info.  
    - Failures: Malformed AI output, JSON parse errors.  
    - Sticky Note: Details parsing logic.

  - **Filter by Level**  
    - Type: Filter  
    - Role: Filters analysis results to pass only those with `level` equal to "A" for further processing.  
    - Configuration: String equality condition on `level` field.  
    - Input: Parsed AI responses.  
    - Output: Filtered intelligence level A results.  
    - Failures: Missing `level` field, case sensitivity issues.  
    - Sticky Note: Specifies filter purpose.

#### 2.6 Notification Dispatch

- **Overview:**  
  Sends the filtered intelligence summary messages to a Telegram chat.

- **Nodes Involved:**  
  - Send Telegram Message

- **Node Details:**

  - **Send Telegram Message**  
    - Type: Telegram  
    - Role: Sends formatted message containing intelligence summary, keywords, and original tweet link to a configured Telegram chat ID.  
    - Configuration:  
      - Text dynamically built using template with summary, keywords (formatted as hashtags), and link.  
      - Chat ID statically set (replace `"YOUR_CHAT_ID"` with actual).  
      - Attribution disabled.  
    - Input: Filtered intelligence level A items.  
    - Output: Telegram message sent confirmation.  
    - Failures: Telegram API auth errors, invalid chat ID, network failures.  
    - Sticky Note: Explains message composition.

#### 2.7 State Update

- **Overview:**  
  Determines the latest tweet timestamp from processed tweets and updates the database record to prevent reprocessing old tweets in future runs.

- **Nodes Involved:**  
  - Get Latest Timestamp  
  - Update Latest Timestamp

- **Node Details:**

  - **Get Latest Timestamp**  
    - Type: Code (JavaScript)  
    - Role: Iterates over all new tweets, finds the latest `isoDate`. Returns it for DB update.  
    - Configuration: Returns empty if no input. Uses reduce for max date comparison.  
    - Input: New tweets after filtering and conditional check.  
    - Output: JSON object with `latestTimestamp` field.  
    - Failures: Empty input leads to empty output (stops update).  
    - Sticky Note: Explains purpose.

  - **Update Latest Timestamp**  
    - Type: PostgreSQL Update  
    - Role: Updates the `last_update_time` field in the database for the monitored Twitter account.  
    - Configuration: Updates row matching `id` with the new timestamp value obtained from previous node.  
    - Input: Latest timestamp JSON object.  
    - Output: Update confirmation.  
    - Failures: DB connection issues, concurrency conflicts.  
    - Sticky Note: No specific note.

---

### 3. Summary Table

| Node Name               | Node Type                      | Functional Role                              | Input Node(s)            | Output Node(s)             | Sticky Note                                                                                   |
|-------------------------|--------------------------------|----------------------------------------------|--------------------------|----------------------------|----------------------------------------------------------------------------------------------|
| Schedule Trigger        | Schedule Trigger               | Triggers workflow on interval                | -                        | Config                     | Set the interval for automatic triggering (e.g., every 15 minutes).                         |
| Config                  | PostgreSQL                    | Loads Twitter monitoring config from DB      | Schedule Trigger          | RSS                        | Describes DB table structure template and field meanings.                                  |
| RSS                     | RSS Feed Read                 | Fetches tweets via RSSHub feed                 | Config                    | Filter New Tweets           | Needs own RSSHub instance; see RSSHub Guide: https://docs.rsshub.app/routes/social-media#x-twitter |
| Filter New Tweets        | Filter                       | Filters tweets newer than last update         | RSS                       | Check if New Tweets Exist   | Filters tweets to only include new ones based on publication date (IsoDate).                |
| Check if New Tweets Exist| If                           | Checks if new tweets exist to continue flow   | Filter New Tweets          | Get Latest Timestamp, Analyze Tweet with AI | Stops workflow if no new tweets; proceeds otherwise.                                   |
| Get Latest Timestamp     | Code                         | Finds latest tweet timestamp to update DB     | Check if New Tweets Exist  | Update Latest Timestamp     | Finds latest timestamp among new tweets for DB update.                                     |
| Update Latest Timestamp  | PostgreSQL                   | Updates last processed tweet timestamp in DB | Get Latest Timestamp       | -                          |                                                                                              |
| Analyze Tweet with AI    | Google Gemini (Langchain)    | Sends tweet content for AI analysis            | Check if New Tweets Exist  | Parse AI Response           | Sends content with prompt to Gemini AI for intelligence summary.                            |
| Parse AI Response        | Code                         | Extracts and parses JSON from AI text output  | Analyze Tweet with AI      | Filter by Level             | Parses JSON block from AI output; handles errors.                                          |
| Filter by Level          | Filter                       | Filters analyses by intelligence level "A"   | Parse AI Response          | Send Telegram Message       | Filters only level A analysis results.                                                     |
| Send Telegram Message    | Telegram                     | Sends analyzed intelligence via Telegram      | Filter by Level            | -                          | Formats and sends summary, keywords, and link to Telegram chat.                            |
| Sticky Notes (various)   | Sticky Note                  | Informational annotations                      | -                        | -                          | Multiple notes provide documentation and guidance on nodes and logic.                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**  
   - Set to trigger every X minutes (e.g., 15 minutes).  
   - No input required.

2. **Create a PostgreSQL node named "Config":**  
   - Set operation to "Execute Query".  
   - Write SQL: `SELECT * FROM twitter_following_config WHERE id = 1;`  
   - Configure connection credentials to your PostgreSQL instance.  
   - Connect Schedule Trigger’s output to this node’s input.

3. **Create an RSS Feed Read node named "RSS":**  
   - Set URL to: `https://your-rsshub-instance.com/twitter/user/{{ $json.twitter_account }}` (dynamic from Config node).  
   - Configure retry on fail.  
   - Connect Config output to RSS input.

4. **Create a Filter node named "Filter New Tweets":**  
   - Set condition: `{{ $json.isoDate }} > {{ $('Config').first().json.last_update_time }}` (date comparison).  
   - Connect RSS output to this node.

5. **Create an If node named "Check if New Tweets Exist":**  
   - Condition: Check if array `{{$('Filter New Tweets').all()}}` is not empty.  
   - Connect Filter New Tweets output to this node.

6. **Create a Code node named "Get Latest Timestamp":**  
   - JavaScript code to find max `isoDate` from input items.  
   - Connect True branch output of If node to this node.

7. **Create a PostgreSQL node named "Update Latest Timestamp":**  
   - Operation: Update  
   - Table: `twitter_following_config`, Schema: `public`  
   - Map columns:  
     - `id`: `={{ $('Config').first().json.id }}`  
     - `last_update_time`: `={{ $json.latestTimestamp }}`  
   - Connect Get Latest Timestamp output to this node.

8. **Create a Google Gemini AI node named "Analyze Tweet with AI":**  
   - Set model ID to `models/gemini-2.5-flash`.  
   - Resource: Document  
   - Text parameter:  
     ```
     {{ $('Config').item.json.prompt }}

     Here is the [Input Tweet]
     Title: {{ $json.title }}
     Content: {{ $json.content }}
     Original Link: {{ $json.link }}
     ```  
   - Connect True branch output of If node also to this node.

9. **Create a Code node named "Parse AI Response":**  
   - JavaScript code to extract JSON block from AI text response using regex and parse it; return error JSON if fail.  
   - Connect Analyze Tweet with AI output to this node.

10. **Create a Filter node named "Filter by Level":**  
    - Condition: `level` equals `"A"`  
    - Connect Parse AI Response output to this node.

11. **Create a Telegram node named "Send Telegram Message":**  
    - Set chat ID to your Telegram chat’s ID.  
    - Configure text with template:  
      ```
      🚨 Intelligence Level A

      Tweet Analysis:
      {{ $json.summary }}

      Keywords: {{ $json.keywords.map(k => `#${k}`).join(' ') }}

      Original Link:
      {{ $json.link }}
      ```  
    - Connect Filter by Level output to this node.

12. **Configure connections:**  
    - Schedule Trigger → Config → RSS → Filter New Tweets → Check if New Tweets Exist.  
    - Check if New Tweets Exist (True branch) → Analyze Tweet with AI → Parse AI Response → Filter by Level → Send Telegram Message.  
    - Check if New Tweets Exist (True branch) → Get Latest Timestamp → Update Latest Timestamp.

13. **Set retry on fail for nodes interacting with external systems (Config, RSS, Analyze Tweet with AI, Parse AI Response).**

14. **Replace all placeholders:**  
    - PostgreSQL credentials.  
    - Telegram chat ID.  
    - RSSHub URL with your RSSHub instance.  
    - AI prompt text in Config database.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                       |
|---------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| RSSHub Twitter route documentation for feed URL format and options.                                           | https://docs.rsshub.app/routes/social-media#x-twitter                                                |
| Database table template for configuration usage in the workflow.                                              | Provided in sticky note with field descriptions (`id`, `twitter_account`, `last_update_time`, etc.)  |
| Set the schedule trigger interval according to desired monitoring frequency (e.g., every 15 minutes).         | Sticky note explaining trigger settings                                                              |
| AI model used is Google Gemini "gemini-2.5-flash" via n8n Langchain integration.                               | Ensure API credentials and model availability                                                        |
| Telegram node requires valid bot token and chat ID set in credentials and node parameters.                     | Telegram Bot API official docs for setup                                                            |

---

**Disclaimer:** The provided text and details originate exclusively from an automated workflow created with n8n, respecting applicable content policies. No illegal or offensive content is included. All data processed is legal and public.