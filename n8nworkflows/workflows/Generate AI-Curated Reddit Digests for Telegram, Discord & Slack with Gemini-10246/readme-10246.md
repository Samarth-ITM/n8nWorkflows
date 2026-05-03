Generate AI-Curated Reddit Digests for Telegram, Discord & Slack with Gemini

https://n8nworkflows.xyz/workflows/generate-ai-curated-reddit-digests-for-telegram--discord---slack-with-gemini-10246


# Generate AI-Curated Reddit Digests for Telegram, Discord & Slack with Gemini

---

### 1. Workflow Overview

This workflow automates the process of generating AI-curated Reddit news digests and distributing them across multiple communication platforms such as Telegram, Discord, and Slack. It is designed to help content creators, marketers, researchers, community managers, and product teams stay updated with high-quality, relevant Reddit posts across selected subreddits without manual filtering.

The workflow logic is grouped into the following key blocks:

- **1.1 Trigger Setup:** Defines how and when the workflow starts, either on a schedule or manually.
- **1.2 Configuration Setup:** Sets user preferences including which subreddits to monitor, post fetching limits, filters, and digest presentation details.
- **1.3 Data Preparation:** Splits subreddit list and loops through each subreddit to enable parallel data fetching.
- **1.4 Data Fetching:** Retrieves top posts from Reddit's public JSON API for each subreddit.
- **1.5 Data Parsing and Cleaning:** Filters and cleans raw Reddit data to remove spam, low-quality, or irrelevant posts.
- **1.6 Aggregation and Deduplication:** Combines posts from all subreddits, removes duplicates, and applies keyword-based filtering.
- **1.7 AI Curation:** Uses AI (Google Gemini recommended) to rank, select, and format the top posts into a digest.
- **1.8 Output Formatting:** Extracts and structures the AI-generated digest text for multi-platform compatibility.
- **1.9 Validation:** Checks that AI output is valid and non-empty to prevent failed deliveries.
- **1.10 Delivery:** Sends the curated digest to enabled platforms: Telegram, Discord, and Slack.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger Setup

- **Overview:** Initiates the workflow either automatically on a schedule (default daily at 9 AM UTC) or manually for testing and one-off runs.
- **Nodes Involved:** 
  - `📅 Schedule Trigger (Daily 9 AM)`
- **Node Details:**

  - **Node:** 📅 Schedule Trigger (Daily 9 AM)  
    - *Type:* Schedule Trigger  
    - *Role:* Starts the workflow daily at 9 AM UTC using a cron expression (`0 9 * * *`).  
    - *Configuration:* Cron expression set to daily 9 AM.  
    - *Input:* None (trigger node).  
    - *Output:* Starts chain by passing empty data to Configuration node.  
    - *Edge cases:* Misconfigured cron expressions may prevent triggering; time zone differences may affect run time.  
    - *Sticky Note:* Explains trigger options and cron examples.  
    - *Sub-workflow:* None.

#### 2.2 Configuration Setup

- **Overview:** Defines all user-customizable parameters such as subreddits, post limits, filters, and digest title.
- **Nodes Involved:**  
  - `⚙️ Configuration`
- **Node Details:**

  - **Node:** ⚙️ Configuration  
    - *Type:* Set Node  
    - *Role:* Sets key parameters for the workflow including:  
      - `subreddits` (comma-separated string)  
      - `posts_per_subreddit` (number, default 25)  
      - `time_filter` (string, e.g., "day")  
      - `total_posts_in_digest` (number, default 10)  
      - `digest_title` (string)  
      - `focus_keywords` (string)  
      - `exclude_keywords` (string)  
      - `min_upvotes` (number, default 10)  
    - *Input:* Trigger node output (empty).  
    - *Output:* Passes configuration data to next node.  
    - *Expressions:* Parameters are static but can be dynamically changed before execution.  
    - *Edge cases:* Invalid subreddit names, empty or malformed inputs could cause empty results downstream.  
    - *Sticky Note:* Details configuration parameters and tips for customization.

#### 2.3 Data Preparation

- **Overview:** Converts the comma-separated subreddit string into an array and loops over each subreddit to prepare for parallel fetching.
- **Nodes Involved:**  
  - `📋 Split Subreddit List`  
  - `🔄 Loop Through Subreddits`
- **Node Details:**

  - **Node:** 📋 Split Subreddit List  
    - *Type:* Set Node  
    - *Role:* Splits `subreddits` string into an array of trimmed subreddit names for iteration.  
    - *Input:* Configuration output.  
    - *Output:* Array of subreddit strings as `subreddit_array`.  
    - *Expressions:* Uses expression `{{$json.subreddits.split(',').map(s => s.trim())}}`.  
    - *Edge cases:* Empty or malformed `subreddits` string leads to empty array.  
    - *Sticky Note:* Explains the splitting logic.

  - **Node:** 🔄 Loop Through Subreddits  
    - *Type:* Set Node  
    - *Role:* Iterates over each subreddit in the array, assigns current subreddit to `subreddit` field.  
    - *Input:* Output of split node with `subreddit_array`.  
    - *Output:* One item per subreddit for parallel processing.  
    - *Expressions:* Selects current subreddit by index: `{{$json.subreddit_array[$itemIndex]}}`.  
    - *Edge cases:* Index errors if array empty or corrupted.  
    - *Sticky Note:* Describes looping behavior.

#### 2.4 Data Fetching

- **Overview:** Fetches top posts from Reddit's public JSON API for each subreddit in parallel.
- **Nodes Involved:**  
  - `🌐 Fetch Reddit Posts (JSON API)`
- **Node Details:**

  - **Node:** 🌐 Fetch Reddit Posts (JSON API)  
    - *Type:* HTTP Request  
    - *Role:* Sends GET requests to `https://www.reddit.com/r/{subreddit}/top.json` with query parameters for time filter and limit.  
    - *Input:* Current subreddit from loop.  
    - *Output:* Raw Reddit JSON data of posts.  
    - *Configuration:*  
      - URL dynamically constructed with expressions using current subreddit and configuration parameters.  
      - Timeout: 30 seconds.  
      - User-Agent header set to `n8n-reddit-automation/1.0`.  
      - No auth needed; uses generic HTTP header auth with User-Agent only.  
    - *Edge cases:*  
      - Rate limiting (approx. 60 requests/minute) may cause failures if too many subreddits.  
      - Private or banned subreddits return empty or error.  
      - Network timeouts or API changes could break fetching.  
    - *Sticky Note:* Details Reddit API usage and limitations.

#### 2.5 Data Parsing and Cleaning

- **Overview:** Processes raw Reddit post data to filter out low-quality, spammy, or irrelevant posts and prepare structured data.
- **Nodes Involved:**  
  - `🔍 Parse & Clean Posts`
- **Node Details:**

  - **Node:** 🔍 Parse & Clean Posts  
    - *Type:* Code Node (JavaScript)  
    - *Role:*  
      - Filters posts below `min_upvotes`.  
      - Skips stickied posts (rules/announcements).  
      - Excludes removed or deleted posts.  
      - Cleans titles and trims selftext to 500 chars.  
      - Converts timestamps to ISO format.  
      - Builds clean URLs.  
      - Sorts posts by score (upvotes).  
    - *Input:* Raw Reddit JSON from fetch node.  
    - *Output:* JSON object with cleaned `posts` array, subreddit name, and total posts count.  
    - *Expressions:* Uses configuration values for filtering thresholds.  
    - *Edge cases:* Empty API responses, malformed data, unexpected JSON structure.  
    - *Sticky Note:* Explains cleaning operations and output format.

#### 2.6 Aggregation and Deduplication

- **Overview:** Merges all cleaned posts from every subreddit, removes duplicates, and applies keyword filters for relevance.
- **Nodes Involved:**  
  - `📊 Aggregate & Deduplicate`
- **Node Details:**

  - **Node:** 📊 Aggregate & Deduplicate  
    - *Type:* Code Node (JavaScript)  
    - *Role:*  
      - Collects all posts from multiple inputs.  
      - Deduplicates by post ID, keeping highest-scored version.  
      - Applies keyword filters:  
        - Includes posts containing any `focus_keywords` (case-insensitive) if configured.  
        - Excludes posts containing any `exclude_keywords`.  
      - Sorts filtered posts by score descending.  
    - *Input:* Multiple cleaned posts from all subreddit iterations.  
    - *Output:* Single unified list of filtered, unique posts and summary stats.  
    - *Expressions:* Uses keywords and configuration parameters dynamically.  
    - *Edge cases:* Empty inputs, missing fields, keyword casing issues.  
    - *Sticky Note:* Details aggregation logic and filtering rules.

#### 2.7 AI Curation

- **Overview:** Uses an AI language model to rank, select, and format the top posts into a concise, structured digest.
- **Nodes Involved:**  
  - `🤖 AI Content Curator`  
  - `Google Gemini Flash 2.0`
- **Node Details:**

  - **Node:** Google Gemini Flash 2.0  
    - *Type:* AI Language Model (Google Gemini)  
    - *Role:* Provides AI model backend for content curation.  
    - *Input:* Prompt and context from AI Content Curator node.  
    - *Output:* AI-generated text response (digest).  
    - *Configuration:* Requires Google Gemini API key credential.  
    - *Edge cases:* API key missing/invalid, quota limits, network issues.  
    - *Sticky Note:* Guides credential setup and API key sources.

  - **Node:** 🤖 AI Content Curator  
    - *Type:* LangChain Agent Node  
    - *Role:*  
      - Defines a detailed prompt instructing AI to:  
        - Analyze all filtered posts.  
        - Rank posts by relevance, quality, engagement, uniqueness, and recency.  
        - Select top N posts per configuration.  
        - Format digest with titles, subreddit, upvotes, comments, and concise insights.  
      - Passes prompt to Google Gemini node and receives AI output.  
    - *Input:* Aggregated filtered posts and configuration data.  
    - *Output:* AI-generated digest text (raw).  
    - *Expressions:* Uses dynamic template expressions to inject data into prompt.  
    - *Edge cases:* Prompt formatting errors, AI hallucinations, API failures.  
    - *Sticky Note:* Explains AI curation goals and provider options.

#### 2.8 Output Formatting

- **Overview:** Extracts the AI-generated formatted digest text and prepares it for delivery to multiple platforms.
- **Nodes Involved:**  
  - `📝 Format for Multiple Platforms`
- **Node Details:**

  - **Node:** 📝 Format for Multiple Platforms  
    - *Type:* Code Node (JavaScript)  
    - *Role:*  
      - Extracts formatted digest text from AI node output, handling variations in response structure.  
      - Trims whitespace and prepares a consistent output JSON.  
      - Attaches timestamp and digest title for reference.  
    - *Input:* AI Content Curator output.  
    - *Output:* JSON with `formatted_output`, `timestamp`, and `digest_title`.  
    - *Edge cases:* Unexpected AI output format, empty or null text.  
    - *Sticky Note:* Describes formatting purpose and output expectations.

#### 2.9 Validation

- **Overview:** Ensures the AI-generated digest text is valid and non-empty before attempting delivery.
- **Nodes Involved:**  
  - `✅ Check AI Output`
- **Node Details:**

  - **Node:** ✅ Check AI Output  
    - *Type:* If Node  
    - *Role:*  
      - Checks if `formatted_output` field exists and is not empty.  
      - If true, proceeds to delivery nodes.  
      - If false, workflow stops preventing empty or faulty messages.  
    - *Input:* Formatted output node.  
    - *Output:* Conditional routing to delivery or termination.  
    - *Edge cases:* Empty AI response, malformed JSON.  
    - *Sticky Note:* Highlights importance of validation to avoid failed deliveries.

#### 2.10 Delivery

- **Overview:** Sends the curated Reddit digest message to one or more enabled communication platforms.
- **Nodes Involved:**  
  - `📱 Send to Telegram`  
  - `💬 Send to Discord`  
  - `💼 Send to Slack`
- **Node Details:**

  - **Node:** 📱 Send to Telegram  
    - *Type:* Telegram Node  
    - *Role:* Sends markdown-formatted text message to Telegram chat via bot token and chat ID.  
    - *Input:* Validated formatted output from previous node.  
    - *Configuration:* Requires Telegram bot token and chat ID credentials.  
    - *Edge cases:* Invalid bot token or chat ID, rate limits, message too long.  
    - *Sticky Note:* Provides setup instructions and tips.

  - **Node:** 💬 Send to Discord  
    - *Type:* Discord Node  
    - *Role:* Sends message content to Discord channel via webhook URL.  
    - *Input:* Validated formatted output.  
    - *Configuration:* Requires Discord webhook URL in credentials.  
    - *Edge cases:* Invalid webhook, permission issues.  
    - *Sticky Note:* Instructions for webhook creation.

  - **Node:** 💼 Send to Slack  
    - *Type:* Slack Node  
    - *Role:* Sends message to specified Slack channel via OAuth token.  
    - *Input:* Validated formatted output.  
    - *Configuration:* OAuth token with channel access required.  
    - *Edge cases:* Token permission issues, channel not found.  
    - *Sticky Note:* Details Slack app setup and permissions.

---

### 3. Summary Table

| Node Name                       | Node Type                      | Functional Role                       | Input Node(s)                        | Output Node(s)                         | Sticky Note                                                                                              |
|--------------------------------|--------------------------------|-------------------------------------|------------------------------------|--------------------------------------|--------------------------------------------------------------------------------------------------------|
| 📖 Main Documentation (START HERE) | Sticky Note                   | Documentation                       | None                               | None                                 | Contains detailed workflow overview, setup, tips, and troubleshooting instructions.                   |
| Sticky Note - Trigger Section   | Sticky Note                   | Trigger guidance                    | None                               | None                                 | Explains trigger options and cron syntax usage.                                                        |
| Sticky Note - Configuration     | Sticky Note                   | Configuration guidance              | None                               | None                                 | Details configurable parameters for the digest.                                                       |
| Sticky Note - Processing        | Sticky Note                   | Data preparation explanation       | None                               | None                                 | Describes splitting and looping through subreddits.                                                   |
| Sticky Note - Fetch Data        | Sticky Note                   | Reddit data fetching explanation   | None                               | None                                 | Explains Reddit API usage, rate limits, and data retrieved.                                           |
| Sticky Note - Parse Posts       | Sticky Note                   | Data parsing and cleaning overview | None                               | None                                 | Describes filtering and cleaning logic for Reddit posts.                                             |
| Sticky Note - Aggregate         | Sticky Note                   | Aggregation and deduplication guide| None                               | None                                 | Details merging posts and keyword filtering logic.                                                    |
| Sticky Note - AI Curation       | Sticky Note                   | AI curation overview               | None                               | None                                 | Describes AI ranking, formatting, and provider setup.                                                |
| Sticky Note - Format Output     | Sticky Note                   | Output formatting explanation      | None                               | None                                 | Explains formatting of AI output for multi-platform delivery.                                        |
| Sticky Note - Validation        | Sticky Note                   | Validation logic explanation       | None                               | None                                 | Highlights validation checks to prevent empty deliveries.                                           |
| Sticky Note - Delivery Platforms| Sticky Note                   | Delivery platform setup            | None                               | None                                 | Guides setup for Telegram, Discord, Slack, and email delivery options.                               |
| 📅 Schedule Trigger (Daily 9 AM)| Schedule Trigger              | Starts the workflow daily          | None                               | ⚙️ Configuration                      | See sticky notes about trigger options.                                                              |
| ⚙️ Configuration                | Set                           | Sets workflow parameters           | 📅 Schedule Trigger                | 📋 Split Subreddit List               | Configuration parameters explained in sticky notes.                                                 |
| 📋 Split Subreddit List          | Set                           | Converts subreddit string to array | ⚙️ Configuration                  | 🔄 Loop Through Subreddits            | Described in Processing sticky note.                                                                 |
| 🔄 Loop Through Subreddits       | Set                           | Iterates over subreddits           | 📋 Split Subreddit List            | 🌐 Fetch Reddit Posts (JSON API)      | Processing block explained in sticky note.                                                           |
| 🌐 Fetch Reddit Posts (JSON API) | HTTP Request                  | Fetches Reddit posts               | 🔄 Loop Through Subreddits          | 🔍 Parse & Clean Posts                | See Fetch Data sticky note.                                                                           |
| 🔍 Parse & Clean Posts           | Code                         | Cleans and filters Reddit posts    | 🌐 Fetch Reddit Posts              | 📊 Aggregate & Deduplicate            | Details in Parse Posts sticky note.                                                                   |
| 📊 Aggregate & Deduplicate       | Code                         | Merges, deduplicates, filters posts| 🔍 Parse & Clean Posts             | 🤖 AI Content Curator                 | Aggregate sticky note describes logic.                                                               |
| 🤖 AI Content Curator            | LangChain Agent              | AI ranks and formats posts         | 📊 Aggregate & Deduplicate          | 📝 Format for Multiple Platforms      | AI Curation sticky note with prompt details.                                                         |
| Google Gemini Flash 2.0          | AI Language Model            | AI backend for curation            | 🤖 AI Content Curator              | 🤖 AI Content Curator (AI response)  | Setup instructions included in AI Curation sticky note.                                             |
| 📝 Format for Multiple Platforms | Code                         | Extracts and cleans AI output      | 🤖 AI Content Curator              | ✅ Check AI Output                    | See Format Output sticky note.                                                                        |
| ✅ Check AI Output               | If                           | Validates AI output non-empty      | 📝 Format for Multiple Platforms   | 📱 Send to Telegram, 💬 Send to Discord, 💼 Send to Slack | Validation sticky note explains flow control.                                                        |
| 📱 Send to Telegram              | Telegram                     | Sends digest to Telegram           | ✅ Check AI Output                 | None                                 | Delivery Platforms sticky note explains setup.                                                       |
| 💬 Send to Discord               | Discord                      | Sends digest to Discord            | ✅ Check AI Output                 | None                                 | Delivery Platforms sticky note explains setup.                                                       |
| 💼 Send to Slack                 | Slack                        | Sends digest to Slack              | ✅ Check AI Output                 | None                                 | Delivery Platforms sticky note explains setup.                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node:**  
   - Add a **Schedule Trigger** node.  
   - Set cron expression to `0 9 * * *` for daily 9 AM UTC execution.

2. **Create Configuration Node:**  
   - Add a **Set** node named `⚙️ Configuration`.  
   - Add parameters:  
     - `subreddits` (string, e.g., `"AI_Agents,generativeAI,ArtificialInteligence,MachineLearning,OpenAI,ChatGPT"`)  
     - `posts_per_subreddit` (number, e.g., 25)  
     - `time_filter` (string, e.g., `"day"`)  
     - `total_posts_in_digest` (number, e.g., 10)  
     - `digest_title` (string, e.g., `"🤖 AI Daily Digest"`)  
     - `focus_keywords` (string, e.g., `"AI agents, ChatGPT, LLM, machine learning, research, tool, breakthrough"`)  
     - `exclude_keywords` (string, e.g., `"crypto, NFT, political, spam"`)  
     - `min_upvotes` (number, e.g., 10)  
   - Connect trigger node to this Configuration node.

3. **Create Subreddit List Split Node:**  
   - Add a **Set** node named `📋 Split Subreddit List`.  
   - Use the expression to split:  
     `{{$json.subreddits.split(',').map(s => s.trim())}}` assigned to a new field `subreddit_array`.  
   - Connect Configuration node output to this node.

4. **Create Loop Through Subreddits Node:**  
   - Add a **Set** node named `🔄 Loop Through Subreddits`.  
   - Use expression to pick current subreddit by index:  
     `{{$json.subreddit_array[$itemIndex]}}` assigned to field `subreddit`.  
   - Enable "Execute Once for Each Item" or equivalent looping in n8n.  
   - Connect Split Subreddit List node to this node.

5. **Create Fetch Reddit Posts Node:**  
   - Add an **HTTP Request** node named `🌐 Fetch Reddit Posts (JSON API)`.  
   - Set method to GET.  
   - URL expression:  
     `https://www.reddit.com/r/{{$json.subreddit}}/top.json?t={{$('⚙️ Configuration').first().json.time_filter}}&limit={{$('⚙️ Configuration').first().json.posts_per_subreddit}}`  
   - Set request timeout to 30s.  
   - Add HTTP header: `User-Agent` with value `n8n-reddit-automation/1.0`.  
   - No authentication required, but can use generic HTTP header auth with User-Agent.  
   - Connect Loop Through Subreddits node to this node.

6. **Create Parse & Clean Node:**  
   - Add a **Code** node named `🔍 Parse & Clean Posts`.  
   - Insert JavaScript code to:  
     - Extract posts array from Reddit JSON response.  
     - Filter posts below `min_upvotes` or stickied or removed/deleted.  
     - Clean and trim title and selftext.  
     - Convert timestamps to ISO.  
     - Sort posts by score descending.  
   - Use the provided code logic from the original node (adjust variable names as needed).  
   - Connect Fetch Reddit Posts node to this node.

7. **Create Aggregate & Deduplicate Node:**  
   - Add a **Code** node named `📊 Aggregate & Deduplicate`.  
   - Write code to:  
     - Merge all posts from all subreddit outputs.  
     - Deduplicate by post `id`.  
     - Apply keyword filters for focus and exclude keywords (case-insensitive).  
     - Sort final list by score descending.  
     - Output filtered posts and summary stats.  
   - Connect Parse & Clean Posts node to this node with multiple inputs enabled.

8. **Create AI Content Curator Node:**  
   - Add a **LangChain Agent** node named `🤖 AI Content Curator`.  
   - Configure prompt with detailed instructions for ranking and formatting posts into digest, injecting configuration values and post data dynamically.  
   - Connect Aggregate & Deduplicate node to this node.

9. **Create Google Gemini Flash 2.0 Node:**  
   - Add **Google Gemini Flash 2.0** node.  
   - Set as AI model for LangChain Agent node.  
   - Create and configure Google Gemini API credentials with API key.  
   - Link Gemini node output back to AI Content Curator node as AI model.  

10. **Create Format for Multiple Platforms Node:**  
    - Add a **Code** node named `📝 Format for Multiple Platforms`.  
    - Implement code to extract AI digest text from AI response, trim, and add timestamp and title fields.  
    - Connect AI Content Curator node to this node.

11. **Create Validation Node:**  
    - Add an **If** node named `✅ Check AI Output`.  
    - Condition: Check if `formatted_output` is not empty.  
    - Connect Format for Multiple Platforms node to this node.

12. **Create Delivery Nodes:**  
    - Add **Telegram** node named `📱 Send to Telegram`.  
      - Configure with Telegram bot token and target chat ID credentials.  
      - Use markdown parse mode.  
    - Add **Discord** node named `💬 Send to Discord`.  
      - Configure with Discord webhook URL credential.  
    - Add **Slack** node named `💼 Send to Slack`.  
      - Configure with Slack OAuth token and select channel from dropdown.  
    - Connect the "true" output of Validation node to all delivery nodes in parallel.

13. **Activate Workflow:**  
    - Test the workflow manually first.  
    - Adjust configuration parameters as needed.  
    - Enable the workflow for automatic scheduled runs.

---

### 5. General Notes & Resources

| Note Content                                                                                                 | Context or Link                                                                                   |
|--------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------|
| Workflow uses Reddit's public JSON API without requiring authentication, but respects rate limits (~60/min). | Reddit API Documentation: https://www.reddit.com/dev/api/                                       |
| AI curation uses Google Gemini Flash 2.0 recommended for cost-efficiency and free tier access.                | Gemini API keys: https://makersuite.google.com/app/apikey                                       |
| Cron expressions can be customized using https://crontab.guru for scheduling flexibility.                     | Cron Expression Generator: https://crontab.guru                                                 |
| Delivery nodes require proper bot tokens or webhook URLs with necessary permissions.                          | Telegram bot setup: https://core.telegram.org/bots                                              |
|                                                                                                              | Discord Webhooks Guide: https://support.discord.com/hc/en-us/articles/228383668                  |
|                                                                                                              | Slack app creation and OAuth: https://api.slack.com/authentication/oauth-v2                      |
| Workflow includes robust validation to avoid sending empty or malformed messages, reducing API call waste.    |                                                                                                |
| Tips are provided in sticky notes for advanced features like flair filtering, email addition, and RSS feed.  |                                                                                                |

---

This document fully describes the Reddit News Automation Pro workflow, enabling comprehensive understanding, modification, or recreation without the original JSON.