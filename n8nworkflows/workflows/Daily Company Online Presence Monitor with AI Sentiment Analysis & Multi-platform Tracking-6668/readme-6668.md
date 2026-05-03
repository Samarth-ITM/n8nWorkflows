Daily Company Online Presence Monitor with AI Sentiment Analysis & Multi-platform Tracking

https://n8nworkflows.xyz/workflows/daily-company-online-presence-monitor-with-ai-sentiment-analysis---multi-platform-tracking-6668


# Daily Company Online Presence Monitor with AI Sentiment Analysis & Multi-platform Tracking

### 1. Workflow Overview

This workflow automates a **daily monitoring report of a company’s online presence** across multiple platforms (Google News, Reddit, and YouTube). It collects mentions related to the company using customizable keywords, deduplicates previously processed mentions, analyzes sentiment and summarizes each mention using OpenAI’s GPT model, categorizes mentions by sentiment, stores processed mentions in a local SQLite database to avoid repeats, and finally sends a formatted email report summarizing the daily findings.

The workflow is logically divided into these blocks:

- **1.1 Trigger & Setup:** Daily automatic trigger and defining company details and keywords.
- **1.2 Data Collection:** Fetching mentions from Google News RSS, Reddit posts, and YouTube videos.
- **1.3 Data Preparation & Merging:** Standardizing data formats and merging all mentions into one list.
- **1.4 Deduplication via SQLite:** Ensuring only new mentions are processed by checking a local SQLite database.
- **1.5 AI Processing:** Using OpenAI to analyze sentiment and summarize each new mention.
- **1.6 Categorization & Storage:** Categorizing mentions by sentiment and recording processed mentions.
- **1.7 Reporting:** Formatting and sending an email report with the daily summary.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & Setup

**Overview:**  
Starts the workflow daily at 9:00 AM server time and sets the company name and keywords for monitoring.

**Nodes Involved:**  
- Daily Morning Trigger (9 AM)  
- Set Company Details

**Node Details:**

- **Daily Morning Trigger (9 AM)**  
  - Type: Cron Trigger  
  - Role: Initiates workflow every day at 9:00 AM.  
  - Configuration: Runs every day at hour=9, minute=0.  
  - Inputs: None  
  - Outputs: Triggers "Set Company Details" node.  
  - Edge Cases: Timezone differences if server timezone doesn’t match expected timezone; check if n8n server time matches desired time zone.

- **Set Company Details**  
  - Type: Set  
  - Role: Defines the target company name and the list of keywords for searching mentions.  
  - Configuration:  
    - `companyName`: Must be replaced with the exact company name (e.g., "Google").  
    - `searchKeywords`: Array of keywords including misspellings, hashtags, product names. Used for search queries across platforms.  
  - Inputs: Triggered by "Daily Morning Trigger (9 AM)"  
  - Outputs: Feeds three search nodes (Google News RSS, Reddit, YouTube).  
  - Edge Cases: Incorrect keywords reduce mention coverage; keywords too generic may produce noise.

---

#### 1.2 Data Collection

**Overview:**  
Collects mentions from three platforms: Google News via RSS, Reddit posts, and YouTube videos.

**Nodes Involved:**  
- Fetch Google News RSS  
- Prepare News for Merge  
- Search Reddit Posts  
- Prepare Reddit for Merge  
- Search YouTube Videos  
- Prepare YouTube for Merge

**Node Details:**

- **Fetch Google News RSS**  
  - Type: RSS Feed  
  - Role: Retrieves news articles mentioning the company.  
  - Configuration:  
    - URL dynamically built using `companyName` from "Set Company Details".  
    - Language (hl=en-US), region (gl=US), and content edition (ceid=US:en) parameters configurable.  
  - Inputs: From "Set Company Details"  
  - Outputs: To "Prepare News for Merge"  
  - Edge Cases: Google News RSS may miss some articles; network or RSS format issues.

- **Prepare News for Merge**  
  - Type: Function  
  - Role: Normalizes Google News data into a standard format with fields: `source`, `title`, `text`, `link`, `publishedAt`.  
  - Configuration: No user configuration needed. Uses fallback values if some fields missing.  
  - Inputs: From "Fetch Google News RSS"  
  - Outputs: To "Merge All Mentions"  
  - Edge Cases: Missing data fields; malformed dates.

- **Search Reddit Posts**  
  - Type: Reddit  
  - Role: Searches Reddit posts matching `searchKeywords`.  
  - Configuration:  
    - Credential: Reddit OAuth2 API (requires app creation and OAuth setup).  
    - Query: Keywords joined by " OR ".  
    - Limit: 20 posts, sorted by "hot".  
  - Inputs: From "Set Company Details"  
  - Outputs: To "Prepare Reddit for Merge"  
  - Edge Cases: Reddit API rate limits; OAuth token expiration; network issues.

- **Prepare Reddit for Merge**  
  - Type: Function  
  - Role: Standardizes Reddit post data into common format.  
  - Configuration: No user changes needed. Converts Unix timestamps to ISO strings.  
  - Inputs: From "Search Reddit Posts"  
  - Outputs: To "Merge All Mentions"  
  - Edge Cases: Missing or malformed post data.

- **Search YouTube Videos**  
  - Type: YouTube  
  - Role: Searches YouTube videos matching `searchKeywords`.  
  - Configuration:  
    - Credential: Google OAuth2 API with YouTube Data API enabled.  
    - Search Query: Keywords joined by space.  
    - Limit: 10 videos, order by relevance.  
  - Inputs: From "Set Company Details"  
  - Outputs: To "Prepare YouTube for Merge"  
  - Edge Cases: YouTube API quota limits; OAuth token expiration.

- **Prepare YouTube for Merge**  
  - Type: Function  
  - Role: Normalizes YouTube video data into standard mention format.  
  - Configuration: No user config needed.  
  - Inputs: From "Search YouTube Videos"  
  - Outputs: To "Merge All Mentions"  
  - Edge Cases: Missing video fields; malformed dates; invalid video IDs.

---

#### 1.3 Data Preparation & Merging

**Overview:**  
Merges all collected mentions into a single unified list for further processing.

**Nodes Involved:**  
- Merge All Mentions

**Node Details:**

- **Merge All Mentions**  
  - Type: Item Lists  
  - Role: Combines mentions from News, Reddit, and YouTube into one list.  
  - Configuration: Mode set to "merge" (concatenate items).  
  - Inputs: From all three "Prepare ... for Merge" nodes  
  - Outputs: To "SQLite: Ensure Table Exists" (deduplication block)  
  - Edge Cases: Large volume could cause performance issues; empty inputs produce empty output.

---

#### 1.4 Deduplication via SQLite

**Overview:**  
Checks each mention against a local SQLite database to filter out previously processed mentions, ensuring only new mentions proceed.

**Nodes Involved:**  
- SQLite: Ensure Table Exists  
- Filter New Mentions (Deduplication)  
- SQLite: Check If Processed

**Node Details:**

- **SQLite: Ensure Table Exists**  
  - Type: SQLite  
  - Role: Creates local table `processed_mentions` if not existing.  
  - Configuration: Database file `company_monitor.db` in n8n data directory; query creates table with columns: `link_hash` (PK), `source`, `title`, `link`, `processed_date`.  
  - Inputs: From "Merge All Mentions"  
  - Outputs: To "Filter New Mentions (Deduplication)"  
  - Edge Cases: File system permission issues; SQLite database lock.

- **Filter New Mentions (Deduplication)**  
  - Type: Function  
  - Role:  
    - Generates MD5 hash of each mention’s link or title to uniquely identify it.  
    - Calls "SQLite: Check If Processed" for each item to verify if already stored.  
    - Passes only new mentions forward.  
  - Configuration:  
    - Uses Node Parameter to execute the SQLite check node programmatically.  
    - Must have 'Run Once Per Item' off to handle multiple items properly.  
  - Inputs: From "SQLite: Ensure Table Exists"  
  - Outputs:  
    - Main output: new mentions to "AI: Analyze Sentiment & Summarize"  
    - Additional output: items to "SQLite: Check If Processed" (internal check)  
  - Edge Cases: Async execution errors; hash collisions unlikely but possible; database query failures.

- **SQLite: Check If Processed**  
  - Type: SQLite  
  - Role: Helper node that checks if a given mention’s hash exists in the database.  
  - Configuration: Query selects `link_hash` matching the hash passed from Function node.  
  - Inputs: Called internally from Function node (not direct connection)  
  - Outputs: Returns rows to Function node for filtering logic.  
  - Edge Cases: Empty results meaning new mention, query errors.

---

#### 1.5 AI Processing

**Overview:**  
Uses OpenAI’s GPT model to analyze the sentiment of each new mention and generate a two-sentence summary.

**Nodes Involved:**  
- AI: Analyze Sentiment & Summarize

**Node Details:**

- **AI: Analyze Sentiment & Summarize**  
  - Type: OpenAI  
  - Role:  
    - Sends each mention’s text to OpenAI GPT-3.5-turbo model with a system prompt instructing it to classify sentiment (Positive, Negative, Neutral) and summarize.  
    - Outputs structured JSON with sentiment and summary.  
  - Configuration:  
    - Model: `gpt-3.5-turbo` (optionally upgrade to `gpt-4o`).  
    - Uses templated system and user prompts pulling mention details dynamically.  
    - Requires OpenAI API key credential.  
  - Inputs: From "Filter New Mentions (Deduplication)"  
  - Outputs: To "Process AI Results & Categorize"  
  - Edge Cases: API rate limits, token limits, malformed AI responses, network errors.

---

#### 1.6 Categorization & Storage

**Overview:**  
Parses AI output for sentiment categorization, handles errors gracefully, and stores processed mentions in the SQLite database.

**Nodes Involved:**  
- Process AI Results & Categorize  
- SQLite: Record Processed Mentions

**Node Details:**

- **Process AI Results & Categorize**  
  - Type: Function  
  - Role:  
    - Parses JSON output from AI node.  
    - Sorts mentions into arrays by sentiment: positive, negative, neutral, and noAnalysis (errors).  
    - Passes categorized data forward as one item.  
  - Configuration: No user config needed. Includes try/catch for JSON parsing errors.  
  - Inputs: From "AI: Analyze Sentiment & Summarize"  
  - Outputs: To both "SQLite: Record Processed Mentions" and "Format Report Email"  
  - Edge Cases: Malformed AI JSON output; empty inputs.

- **SQLite: Record Processed Mentions**  
  - Type: SQLite  
  - Role: Inserts all newly processed mentions into `processed_mentions` table to prevent future duplicates.  
  - Configuration:  
    - Query inserts `link_hash`, `source`, `title`, `link`, and current timestamp.  
    - Database: `company_monitor`.  
  - Inputs: From "Process AI Results & Categorize"  
  - Outputs: None (end node for DB update)  
  - Edge Cases: DB write failures; race conditions if multiple workflows running simultaneously.

---

#### 1.7 Reporting

**Overview:**  
Formats the daily report email grouped by sentiment and sends it using Gmail.

**Nodes Involved:**  
- Format Report Email  
- Send Report Email

**Node Details:**

- **Format Report Email**  
  - Type: Function  
  - Role:  
    - Builds Markdown-formatted email body summarizing mentions by sentiment category.  
    - Includes mention count, source, title, summary, link, and publish date.  
    - Handles no new mentions case with a friendly message.  
  - Configuration: No user config needed, but text and formatting can be customized.  
  - Inputs: From "Process AI Results & Categorize"  
  - Outputs: To "Send Report Email"  
  - Edge Cases: Empty results; date formatting issues.

- **Send Report Email**  
  - Type: Gmail  
  - Role: Sends the daily email to the user.  
  - Configuration:  
    - Gmail OAuth2 API credential required.  
    - From email must match authenticated account.  
    - To email must be updated to desired recipient.  
    - Subject and body dynamically pulled from previous node.  
  - Inputs: From "Format Report Email"  
  - Outputs: None (final node)  
  - Edge Cases: OAuth expiration; email quota limits; incorrect recipient address.

---

### 3. Summary Table

| Node Name                      | Node Type           | Functional Role                                      | Input Node(s)                   | Output Node(s)                                  | Sticky Note                                                                                                                                                                                                                      |
|-------------------------------|---------------------|----------------------------------------------------|--------------------------------|------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Daily Morning Trigger (9 AM)   | Cron Trigger        | Triggers workflow daily at 9:00 AM                  | None                           | Set Company Details                            | This `Cron` node triggers the workflow automatically every **day at 9:00 AM** (based on your n8n server's local time zone). To change, adjust 'Hour' and 'Minute' fields.                                                        |
| Set Company Details            | Set                 | Defines company name and search keywords            | Daily Morning Trigger (9 AM)   | Fetch Google News RSS, Search Reddit Posts, Search YouTube Videos | Change `companyName` and `searchKeywords` to match the monitored company.                                                                                                                                                        |
| Fetch Google News RSS          | RSS Feed            | Fetches news articles mentioning the company        | Set Company Details            | Prepare News for Merge                         | Pre-configured Google News RSS URL; adjust language and geo params if needed.                                                                                                                                                   |
| Prepare News for Merge         | Function            | Standardizes news data format                         | Fetch Google News RSS          | Merge All Mentions                            | No configuration needed.                                                                                                                                                                                                       |
| Search Reddit Posts            | Reddit              | Searches Reddit posts with company keywords          | Set Company Details            | Prepare Reddit for Merge                      | Requires Reddit OAuth2 credential. Be mindful of Reddit API rate limits.                                                                                                                                                        |
| Prepare Reddit for Merge       | Function            | Standardizes Reddit post data                         | Search Reddit Posts            | Merge All Mentions                            | No configuration needed. Converts timestamps to ISO strings.                                                                                                                                                                  |
| Search YouTube Videos          | YouTube             | Searches YouTube videos with company keywords        | Set Company Details            | Prepare YouTube for Merge                     | Requires Google OAuth2 credential with YouTube Data API enabled. Watch for API quota limits.                                                                                                                                    |
| Prepare YouTube for Merge      | Function            | Standardizes YouTube video data                       | Search YouTube Videos          | Merge All Mentions                            | No configuration needed.                                                                                                                                                                                                       |
| Merge All Mentions             | Item Lists          | Combines all mentions into a single list             | Prepare News for Merge, Prepare Reddit for Merge, Prepare YouTube for Merge | SQLite: Ensure Table Exists                   | No configuration needed.                                                                                                                                                                                                       |
| SQLite: Ensure Table Exists    | SQLite              | Ensures local DB table for processed mentions exists | Merge All Mentions             | Filter New Mentions (Deduplication)           | Creates `processed_mentions` table if missing.                                                                                                                                                                                 |
| Filter New Mentions (Deduplication) | Function        | Hashes mentions and filters out already processed    | SQLite: Ensure Table Exists    | AI: Analyze Sentiment & Summarize             | Uses `crypto` for hashing; calls SQLite check node internally; 'Run Once Per Item' must be OFF.                                                                                                                                 |
| SQLite: Check If Processed     | SQLite              | Checks DB for existing mention hash                   | Called internally by Function  | Returns result to Function node                | No direct config; used as subquery in deduplication step.                                                                                                                                                                      |
| AI: Analyze Sentiment & Summarize | OpenAI           | Analyzes sentiment and summarizes each new mention   | Filter New Mentions (Deduplication) | Process AI Results & Categorize              | Uses `gpt-3.5-turbo`; requires OpenAI API key; outputs structured JSON sentiment and summary.                                                                                                                                  |
| Process AI Results & Categorize | Function           | Parses AI output and categorizes mentions by sentiment | AI: Analyze Sentiment & Summarize | SQLite: Record Processed Mentions, Format Report Email | Handles JSON parse errors; groups mentions into positive, neutral, negative, noAnalysis.                                                                                                                                        |
| SQLite: Record Processed Mentions | SQLite           | Inserts new mentions into DB to prevent duplicates   | Process AI Results & Categorize | None                                           | Inserts mention hashes and metadata; no further config needed.                                                                                                                                                                 |
| Format Report Email            | Function            | Formats the daily email report with grouped mentions | Process AI Results & Categorize | Send Report Email                              | Outputs Markdown email body; customizable formatting; handles no new mentions case.                                                                                                                                             |
| Send Report Email              | Gmail               | Sends the daily report email                          | Format Report Email            | None                                           | Requires Gmail OAuth2 credential; update From and To email addresses accordingly.                                                                                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Cron Trigger node named "Daily Morning Trigger (9 AM)"**  
   - Type: Cron  
   - Settings: Mode = everyDay, Hour = 9, Minute = 0  
   - Purpose: Trigger workflow daily at 9 AM server time.

2. **Add a Set node named "Set Company Details"**  
   - Connected from "Daily Morning Trigger (9 AM)"  
   - Add fields:  
     - `companyName`: string, value your company exact name (e.g., "Google")  
     - `searchKeywords`: array of strings, multiple keywords including hashtags and misspellings (e.g., `["Google", "#Google", "GoogleCloud"]`)  
   - Purpose: Define monitoring targets for searches.

3. **Add an RSS Feed node named "Fetch Google News RSS"**  
   - Connected from "Set Company Details"  
   - URL: `https://news.google.com/rss/search?q={{ encodeURIComponent($node["Set Company Details"].json.companyName) }}&hl=en-US&gl=US&ceid=US:en`  
   - Purpose: Fetch news mentioning company.

4. **Add a Function node named "Prepare News for Merge"**  
   - Connected from "Fetch Google News RSS"  
   - Paste function code that maps each RSS item to JSON with fields: `source` ("News Article"), `title`, `text`, `link`, `publishedAt` (ISO date)  
   - Purpose: Normalize news data.

5. **Add a Reddit node named "Search Reddit Posts"**  
   - Connected from "Set Company Details"  
   - Credential: Create Reddit OAuth2 API credential with your Reddit app credentials.  
   - Query: `={{ $node["Set Company Details"].json.searchKeywords.join(' OR ') }}`  
   - Limit: 20 posts  
   - Sort: hot  
   - Purpose: Search Reddit posts mentioning the company.

6. **Add a Function node named "Prepare Reddit for Merge"**  
   - Connected from "Search Reddit Posts"  
   - Paste function code to normalize Reddit posts with fields: `source` ("Reddit Post"), `title`, `text`, `link`, `publishedAt` (convert Unix timestamp to ISO string)  
   - Purpose: Normalize Reddit data.

7. **Add a YouTube node named "Search YouTube Videos"**  
   - Connected from "Set Company Details"  
   - Credential: Create Google OAuth2 API credential with YouTube Data API enabled.  
   - Search: `={{ $node["Set Company Details"].json.searchKeywords.join(' ') }}`  
   - Limit: 10 videos  
   - Order: relevance  
   - Purpose: Search YouTube videos mentioning the company.

8. **Add a Function node named "Prepare YouTube for Merge"**  
   - Connected from "Search YouTube Videos"  
   - Paste function code to normalize YouTube videos with fields: `source` ("YouTube Video"), `title`, `text`, `link` (YouTube watch URL), `publishedAt`  
   - Purpose: Normalize YouTube data.

9. **Add an Item Lists node named "Merge All Mentions"**  
   - Connect inputs from all three "Prepare ... for Merge" nodes  
   - Mode: merge (concatenate)  
   - Purpose: Combine all mentions into one list.

10. **Add a SQLite node named "SQLite: Ensure Table Exists"**  
    - Connected from "Merge All Mentions"  
    - Database: Create new SQLite DB named `company_monitor.db` (default location in n8n data folder)  
    - Query:  
      ```sql
      CREATE TABLE IF NOT EXISTS processed_mentions (
        link_hash TEXT PRIMARY KEY,
        source TEXT,
        title TEXT,
        link TEXT,
        processed_date TEXT
      )
      ```  
    - Purpose: Prepare DB table for deduplication.

11. **Add a Function node named "Filter New Mentions (Deduplication)"**  
    - Connected from "SQLite: Ensure Table Exists"  
    - Paste function code that:  
      - Generates MD5 hash of `link` or `title`  
      - Calls "SQLite: Check If Processed" node asynchronously per item  
      - Returns only new mentions  
    - Ensure "Run Once Per Item" is OFF.  
    - Purpose: Filter out mentions already processed.

12. **Add a SQLite node named "SQLite: Check If Processed"**  
    - No direct connection; called programmatically from function node  
    - Database: `company_monitor`  
    - Query: `SELECT link_hash FROM processed_mentions WHERE link_hash = '{{ $json.linkHash }}'`  
    - Purpose: Check if mention hash exists.

13. **Add an OpenAI node named "AI: Analyze Sentiment & Summarize"**  
    - Connected from "Filter New Mentions (Deduplication)"  
    - Credential: OpenAI API key credential  
    - Model: `gpt-3.5-turbo`  
    - Messages:  
      - System: Prompt instructing to analyze sentiment and summarize mention text, output JSON with `"sentiment"` and `"summary"` fields, referencing `companyName`.  
      - User: Includes mention `source`, `title`, and `text`.  
    - Purpose: Generate sentiment analysis and summary per mention.

14. **Add a Function node named "Process AI Results & Categorize"**  
    - Connected from "AI: Analyze Sentiment & Summarize"  
    - Paste function code that:  
      - Parses AI JSON output  
      - Categorizes mentions into arrays by sentiment (positive, neutral, negative, noAnalysis)  
      - Handles JSON parsing errors gracefully  
    - Outputs two branches:  
      - To "SQLite: Record Processed Mentions"  
      - To "Format Report Email"  
    - Purpose: Organize AI results and prepare for storage and reporting.

15. **Add a SQLite node named "SQLite: Record Processed Mentions"**  
    - Connected from "Process AI Results & Categorize"  
    - Database: `company_monitor`  
    - Query:  
      ```sql
      INSERT INTO processed_mentions (link_hash, source, title, link, processed_date) VALUES ('{{ $json.linkHash }}', '{{ $json.source }}', '{{ $json.title }}', '{{ $json.link }}', '{{ new Date().toISOString() }}')
      ```  
    - Purpose: Store new mentions to prevent duplicates.

16. **Add a Function node named "Format Report Email"**  
    - Connected from "Process AI Results & Categorize"  
    - Paste function code that:  
      - Builds Markdown email body grouped by sentiment categories  
      - Includes mention count, source, title, summary, link, and published date  
      - Provides message if no new mentions found  
    - Purpose: Prepare human-readable daily report.

17. **Add a Gmail node named "Send Report Email"**  
    - Connected from "Format Report Email"  
    - Credential: Gmail OAuth2 API credential  
    - From Email: Your Gmail address (must match authenticated account)  
    - To Email: Replace with actual recipient email address  
    - Subject: `={{ $json.emailSubject }}` (from previous node)  
    - Text: `={{ $json.emailBody }}` (from previous node)  
    - Purpose: Send daily report email.

---

### 5. General Notes & Resources

| Note Content                                                                                             | Context or Link                                                                                                            |
|---------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------|
| To create Reddit OAuth2 API credentials, follow n8n docs for creating a "script" type Reddit app.       | https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.reddit/                                                   |
| You must enable YouTube Data API v3 and create OAuth2 credentials in Google Cloud Console for YouTube. | https://developers.google.com/youtube/registering_an_application                                                            |
| OpenAI API keys can be generated from https://platform.openai.com/account/api-keys                      | https://platform.openai.com/docs/api-reference/authentication                                                               |
| Gmail OAuth2 credentials require enabling Gmail API and creating OAuth2 client credentials in Google Console | https://developers.google.com/gmail/api/quickstart/js                                                                       |
| Google News RSS may not cover all news sources; consider supplementing with additional news APIs if needed | https://news.google.com/rss                                                                                                 |
| SQLite database file `company_monitor.db` is stored locally in n8n data directory; backup if workflow is critical |                                                                                                                            |
| Markdown email formatting supports bold, lists, and simple links for clear presentation                  | See https://commonmark.org/help/ for Markdown syntax                                                                        |

---

**Disclaimer:** The provided text is exclusively derived from a workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.