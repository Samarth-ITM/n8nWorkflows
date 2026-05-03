N8N Automated Twitter Reply Bot Workflow

https://n8nworkflows.xyz/workflows/n8n-automated-twitter-reply-bot-workflow-8410


# N8N Automated Twitter Reply Bot Workflow

---

## 1. Workflow Overview

This workflow automates replying to tweets on X (formerly Twitter) using n8n. It targets social media managers or automation enthusiasts who want to generate intelligent, engaging replies to selected tweets based on keyword or community searches. The replies are generated using a large language model (LLM) and posted automatically, with status updates sent via Telegram and replies logged in MongoDB to avoid duplication.

The workflow is logically organized into the following blocks:

- **1.1 Input Reception & Triggering**  
  Handles manual triggers and Telegram bot commands to initiate the reply process.

- **1.2 Tweet Fetching & Filtering**  
  Uses Apify actors to scrape tweets based on keywords or community IDs, filters out already replied tweets via MongoDB, and applies further content filters.

- **1.3 Random Selection & Retry Logic**  
  Randomly selects keywords or community IDs for searching tweets, implements retry counters with failure handling and notification.

- **1.4 Tweet Processing & Reply Generation**  
  Formats the selected tweet data, inputs it to an LLM chain to generate a natural, engaging reply according to detailed style and filtering rules.

- **1.5 Posting Reply & Status Reporting**  
  Posts the generated reply on X (Twitter), sends success or failure notifications to Telegram, and saves reply data in MongoDB to prevent duplicates.

- **1.6 Scheduling & Execution Control**  
  Schedules periodic workflow runs with time-based and probabilistic limitations to mimic natural activity hours.

- **1.7 Sub-Workflow Execution**  
  Calls a dedicated sub-workflow to handle the reply generation and posting after aggregation of tweet data.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception & Triggering

**Overview:**  
This block listens for manual or Telegram bot triggers to start the reply generation process.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- Telegram Trigger  
- If3 (Telegram message filter)  
- Find documents (MongoDB fetch for replied tweets)  
- No Operation, do nothing1  

**Node Details:**  

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Allows manual initiation of the workflow for testing or on-demand execution.  
  - Config: No parameters.  
  - Connections: On trigger, fetches replied tweets from MongoDB ("Find documents").  
  - Edge cases: None specific; manual trigger is user-initiated.

- **Telegram Trigger**  
  - Type: Telegram Trigger node  
  - Role: Listens for Telegram messages to trigger workflow.  
  - Config: Watches for all message updates.  
  - Credentials: Telegram API with bot token configured.  
  - Connections: Feeds into "If3" node to filter commands.  
  - Edge cases: Requires correct Telegram bot setup and webhook configuration; message text must match command.

- **If3**  
  - Type: If node (version 2.2)  
  - Role: Checks if incoming Telegram message text equals "/reply" to proceed.  
  - Config: Case sensitive string equality check on `{{$json.message.text}}`.  
  - Connections: True branch triggers "Find documents", false branch leads to "No Operation, do nothing1".  
  - Edge cases: If command is misspelled or absent, workflow does nothing.

- **Find documents**  
  - Type: MongoDB node  
  - Role: Fetches documents of tweets already replied to, to avoid duplicates.  
  - Config: Queries "collection_name" with projection on "tweet_id".  
  - Credentials: MongoDB account configured.  
  - Connections: Outputs to aggregation node "Aggregate1".  
  - Edge cases: MongoDB connectivity or query failures possible.

- **No Operation, do nothing1**  
  - Type: No Operation node  
  - Role: Ends branch for non-matching Telegram commands without action.  
  - Config: None.  
  - Edge cases: None.

---

### 2.2 Tweet Fetching & Filtering

**Overview:**  
This block fetches tweets from Apify actors based on keywords or community IDs, filters them to exclude previously replied tweets and low-quality content.

**Nodes Involved:**  
- Keyword/Community List  
- Randomized community, keyword  
- If4 (Regex test for community ID)  
- Community search actor  
- Search keyword actor  
- Get dataset items (Apify Datasets)  
- If2 (Check dataset for results)  
- Community filter  
- If (Filter out tweets already replied)  
- Get tweet data  
- Aggregate  

**Node Details:**  

- **Keyword/Community List**  
  - Type: Set node  
  - Role: Holds an array of keywords and community IDs to search. Includes a failure counter to track retries.  
  - Config: JSON with keyword_community_list and failure counter initialized or incremented.  
  - Connections: Outputs to "Randomized community, keyword".  
  - Edge cases: Empty or malformed list will cause no searches.

- **Randomized community, keyword**  
  - Type: Set node  
  - Role: Selects one random item from the keyword/community list for search.  
  - Config: Expression to pick `randomItem()` from the list.  
  - Connections: Routes to "If4" for type determination.  
  - Edge cases: Random selection may repeatedly pick the same item without filtering.

- **If4**  
  - Type: If node  
  - Role: Uses regex to determine if the random item is a community ID (19-digit numeric string).  
  - Config: Regex `\d{19}` to identify community IDs.  
  - Connections: If true, calls "Community search actor"; else calls "Search keyword actor".  
  - Edge cases: Incorrect format strings cause routing errors.

- **Community search actor**  
  - Type: Apify node  
  - Role: Scrapes tweets from specified community IDs.  
  - Config: Actor ID for community search, custom JSON body with `communityIds` array of one item, numberOfTweets=40.  
  - Credentials: Apify API account.  
  - Connections: Outputs dataset to "Get dataset items".  
  - Edge cases: Apify actor rate limits, timeouts, or API errors.

- **Search keyword actor**  
  - Type: Apify node  
  - Role: Scrapes tweets based on keyword search with filters.  
  - Config: Actor ID for keyword search, custom JSON body with search query, language, minimum engagement, verified users, etc.  
  - Credentials: Apify API account.  
  - Connections: Outputs dataset to "Get dataset items".  
  - Edge cases: Same as above, plus keyword syntax errors.

- **Get dataset items**  
  - Type: Apify dataset fetch node  
  - Role: Retrieves scraped tweet data from Apify actors.  
  - Config: Dataset ID from previous node output.  
  - Credentials: Apify API account.  
  - Connections: Feeds into "If2".  
  - Edge cases: Dataset may be empty or inaccessible.

- **If2**  
  - Type: If node  
  - Role: Checks if dataset is non-empty before proceeding.  
  - Config: Checks if incoming JSON is not empty.  
  - Connections: True path to "Community filter", false to "If5" (retry logic).  
  - Edge cases: Empty datasets trigger retry path.

- **Community filter**  
  - Type: Filter node  
  - Role: Applies multiple filtering conditions on tweets: minimum text length, minimum followers, no duplicates, minimum replies/likes/views, English language.  
  - Config: Multiple numeric and boolean conditions using expressions and referencing aggregated tweet IDs.  
  - Connections: Outputs filtered tweets to "If".  
  - Edge cases: Overly restrictive filters may exclude all tweets.

- **If**  
  - Type: If node  
  - Role: Checks if filtered tweets list is non-empty.  
  - Config: Condition: tweets list not empty.  
  - Connections: True branch to "Get tweet data" for further processing, false branch to retry logic "If5".  
  - Edge cases: Empty filtered result triggers retry.

- **Get tweet data**  
  - Type: Set node  
  - Role: Maps tweet fields from input to standardized format required by LLM: id, screen_name, followers count, created_at, cleaned tweet text, engagement metrics.  
  - Config: Uses expressions to extract and transform data. Removes trailing URLs from tweet text.  
  - Connections: Outputs to "Aggregate".  
  - Edge cases: Missing or malformed fields may cause errors.

- **Aggregate**  
  - Type: Aggregate node  
  - Role: Aggregates all tweet data items into a single output array for batch processing.  
  - Config: Aggregate all item data.  
  - Connections: Outputs to sub-workflow execution node.  
  - Edge cases: Large datasets could cause performance issues.

---

### 2.3 Random Selection & Retry Logic

**Overview:**  
Implements retry logic for the tweet search process with failure counting, waiting, and notification after max retries.

**Nodes Involved:**  
- If5  
- Wait  
- Increment Failure Counter  
- Send a scraping failure message  

**Node Details:**  

- **If5**  
  - Type: If node  
  - Role: Determines whether failure count is less than 3 to retry or send failure notification.  
  - Config: Numeric comparison on failure counter property.  
  - Connections: True path to "Wait", false path to "Send a scraping failure message".  
  - Edge cases: Failure count improperly incremented or reset may cause infinite loops or premature failure.

- **Wait**  
  - Type: Wait node  
  - Role: Pauses workflow for 3 seconds before retrying.  
  - Config: 3 seconds wait time.  
  - Connections: Outputs to failure counter increment node.  
  - Edge cases: Wait node may cause workflow delays.

- **Increment Failure Counter**  
  - Type: Set node  
  - Role: Increments the failure count in the keyword/community list JSON.  
  - Config: Adds 1 to the existing failure count.  
  - Connections: Loops back to "Keyword/Community List".  
  - Edge cases: Increment logic depends on correct JSON path; mistakes cause counting errors.

- **Send a scraping failure message**  
  - Type: Telegram node  
  - Role: Sends a failure notification message to a configured Telegram chat after max retries.  
  - Config: Message text with instructions to manually trigger retry; uses Telegram chat ID.  
  - Credentials: Telegram API account.  
  - Edge cases: Missing chat ID or Telegram API issues prevent notification.

---

### 2.4 Tweet Processing & Reply Generation

**Overview:**  
Prepares tweet data for AI processing, generates replies using an LLM chain with detailed prompt engineering, parses output, and formats reply data.

**Nodes Involved:**  
- When Executed by Another Workflow (sub-workflow trigger)  
- modify tweet data  
- Basic LLM Chain  
- OpenRouter Chat Model1  
- Structured Output Parser  
- parse reply  

**Node Details:**  

- **When Executed by Another Workflow**  
  - Type: Execute Workflow Trigger node  
  - Role: Entry point for sub-workflow execution triggered by the main workflow after aggregation.  
  - Config: Passthrough input source.  
  - Connections: Outputs to "modify tweet data".  
  - Edge cases: Workflow ID and input must match main workflow expectations.

- **modify tweet data**  
  - Type: Set node  
  - Role: Adds current timestamp and wraps tweet data into a JSON object for LLM input.  
  - Config: Sets `time_now` with current ISO timestamp, `data` with tweet array.  
  - Connections: Outputs to "Basic LLM Chain".  
  - Edge cases: Timezone handling assumed UTC; incorrect timestamps may affect scoring.

- **Basic LLM Chain**  
  - Type: Langchain LLM Chain node  
  - Role: Uses a detailed prompt to filter, score, and generate a single optimized reply for the selected tweet.  
  - Config:  
    - Input: JSON of current time and list of tweets with metadata.  
    - Prompt: Complex instructions to exclude spammy posts, score tweets by engagement and timing, select the best, and generate a witty, human-like reply under 100 chars.  
    - Output parser: Enabled to parse structured JSON output.  
  - Connections: Outputs to "Create Tweet".  
  - Edge cases: LLM rate limits, prompt failures, or empty output cause errors.

- **OpenRouter Chat Model1**  
  - Type: Langchain OpenRouter Chat Model node  
  - Role: Provides the language model endpoint (model "x-ai/grok-3") used by the LLM chain.  
  - Config: Uses OpenRouter API credentials.  
  - Connections: AI model input for "Basic LLM Chain".  
  - Edge cases: API limits, credential issues.

- **Structured Output Parser**  
  - Type: Langchain output parser node  
  - Role: Parses LLM JSON output to extract selected tweet ID, screen name, and reply text.  
  - Config: JSON schema example for validation and auto-fixing.  
  - Connections: Parser output feeds back into "Basic LLM Chain".  
  - Edge cases: Malformed LLM responses cause parse failure.

- **parse reply**  
  - Type: Set node  
  - Role: Extracts and formats LLM output fields, adds current date and constructs tweet URL for logging.  
  - Config: Assigns variables `tweet_id`, `screen_name`, `reply`, `tweet_url`, and current date/time string.  
  - Connections: Outputs to "Insert documents".  
  - Edge cases: Missing LLM output fields cause errors.

---

### 2.5 Posting Reply & Status Reporting

**Overview:**  
Posts the generated reply on X, sends Telegram notifications on success or failure, and logs replies in MongoDB.

**Nodes Involved:**  
- Create Tweet  
- Send a success reply  
- Send a failed reply  
- Insert documents  

**Node Details:**  

- **Create Tweet**  
  - Type: Twitter node (OAuth2)  
  - Role: Posts the reply text as a reply to the selected tweet ID on X (Twitter).  
  - Config:  
    - Text set from LLM output reply.  
    - `inReplyToStatusId` references selected tweet ID.  
  - Credentials: X (Twitter) OAuth2 API credentials.  
  - Connections:  
    - On success: to "Send a success reply"  
    - On failure: to "Send a failed reply"  
  - Edge cases: Twitter API limits (17 posts/day free tier), network errors, invalid tweet IDs.

- **Send a success reply**  
  - Type: Telegram node  
  - Role: Sends confirmation message with tweet URL and reply text to Telegram chat.  
  - Config: Uses Telegram chat ID, HTML parse mode, disables notification.  
  - Connections: Outputs to "parse reply" for logging.  
  - Edge cases: Telegram API issues, missing chat ID.

- **Send a failed reply**  
  - Type: Telegram node  
  - Role: Sends failure notification message with details about Twitter API limitations.  
  - Config: Similar to success node, with failure message content.  
  - Connections: Outputs to "parse reply" to still log attempted reply.  
  - Edge cases: Same as above.

- **Insert documents**  
  - Type: MongoDB node  
  - Role: Inserts new reply data into MongoDB to record replied tweets and prevent duplication.  
  - Config: Inserts fields tweet_id, screen_name, reply, tweet_url, and date into "collection_name".  
  - Credentials: MongoDB account.  
  - Edge cases: MongoDB connection failure or insertion errors.

---

### 2.6 Scheduling & Execution Control

**Overview:**  
Periodically triggers the workflow during specific hours, with probabilistic control to mimic natural posting behavior.

**Nodes Involved:**  
- Schedule Trigger  
- Code (time and probability filter)  
- If1  
- Find documents  
- Aggregate1  
- Keyword/Community List  

**Node Details:**  

- **Schedule Trigger**  
  - Type: Schedule Trigger node  
  - Role: Runs the workflow every 20 minutes.  
  - Config: Interval set to 20 minutes; default timezone (must be edited per sticky note).  
  - Edge cases: Timezone must be configured correctly for accurate active hours.

- **Code**  
  - Type: Code node (JavaScript)  
  - Role: Filters execution based on Kyiv timezone hours (active 07:00–23:59) and applies ~28% probability to run.  
  - Config: Uses Intl.DateTimeFormat with Europe/Kyiv timezone, checks hour, and random threshold.  
  - Output: Boolean `value` indicating if workflow continues.  
  - Edge cases: Timezone mismatch, random threshold set to 1 (runs every time) — possibly a configuration oversight.

- **If1**  
  - Type: If node  
  - Role: Checks if `value` from Code node is true to proceed or do nothing.  
  - Connections: True branch to "Find documents" (start tweet filtering), false branch to "No Operation, do nothing".  
  - Edge cases: None.

- **Find documents**  
  - Same as previously described, fetches replied tweets from MongoDB.

- **Aggregate1**  
  - Type: Aggregate node  
  - Role: Aggregates replied tweet IDs for filtering new tweets.  
  - Connections: Outputs to "Keyword/Community List" to start new tweet search loop.  
  - Edge cases: Large aggregation sets may affect performance.

- **Keyword/Community List**  
  - Resets or continues retry logic for random keyword/community selection.

---

### 2.7 Sub-Workflow Execution

**Overview:**  
Delegates the reply generation and posting steps to a separate workflow for modularity and clearer separation.

**Nodes Involved:**  
- Execute Workflow  
- Get dataset items (in main workflow)  
- Aggregate (in main workflow)  

**Node Details:**  

- **Execute Workflow**  
  - Type: Execute Workflow node  
  - Role: Calls the reply posting sub-workflow ID `d2ab3cFnoWqP7E49`.  
  - Config: Passes aggregated tweets as input.  
  - Edge cases: Workflow ID must be valid and active.

- The sub-workflow handles the reply creation, posting, Telegram notification, and MongoDB logging as described in blocks 2.4 and 2.5.

---

## 3. Summary Table

| Node Name                  | Node Type                           | Functional Role                                | Input Node(s)                     | Output Node(s)                 | Sticky Note                                                                                                                                            |
|----------------------------|-----------------------------------|------------------------------------------------|----------------------------------|-------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                    | Manual start trigger                            | None                             | Find documents                 |                                                                                                                                                        |
| Telegram Trigger            | Telegram Trigger                  | Telegram bot command trigger                    | None                             | If3                           | "You can trigger the workflow manually via /reply\nYou need to setup bot credential for this"                                                          |
| If3                        | If                               | Filters Telegram messages for "/reply" command | Telegram Trigger                 | Find documents, No Operation   |                                                                                                                                                        |
| Find documents             | MongoDB                          | Fetches replied tweets from DB                  | When clicking Execute workflow, If3 | Aggregate1                    | "You have to connect your mongo DB account to store already replied tweets.\nThe node fetches already replied tweets"                                 |
| Aggregate1                 | Aggregate                       | Aggregates replied tweet IDs                    | Find documents                   | Keyword/Community List         | "Aggregate posted reply to a unified list"                                                                                                            |
| Keyword/Community List      | Set                             | Holds keywords and community IDs and failure count | Aggregate1                      | Randomized community, keyword  | "A list of keywords/communities to search\nYou have to specify them in the node"                                                                       |
| Randomized community, keyword | Set                             | Selects random keyword/community from list      | Keyword/Community List           | If4                           | "Get randomized items, route to the right API\nSet and check random item from user input list of keywords or communities"                             |
| If4                        | If                               | Routes based on whether item is community ID    | Randomized community, keyword    | Community search actor, Search keyword actor |                                                                                                                                                |
| Community search actor      | Apify                           | Scrapes tweets from community                    | If4                            | Get dataset items             | "Apify X (Twitter) scrapping\nYou must have Apify community node installed before pasting the JSON"                                                   |
| Search keyword actor        | Apify                           | Scrapes tweets based on keyword                  | If4                            | Get dataset items             | Same as above                                                                                                                                           |
| Get dataset items          | Apify                           | Fetches scraped tweets dataset                    | Community search actor, Search keyword actor | If2                           |                                                                                                                                                |
| If2                        | If                               | Checks if dataset is non-empty                    | Get dataset items               | Community filter, If5          | "Check if there are results\nBased on your query and parameters the APIs may return no results"                                                        |
| Community filter            | Filter                          | Applies multiple filters to tweets                | If2                            | If                            | "Filters tweets\nFeel free to edit filters\nNote: same filter works for both, community and search Apify actors."                                        |
| If                         | If                               | Checks if filtered tweets exist                   | Community filter                | Get tweet data, If5            |                                                                                                                                                |
| Get tweet data             | Set                             | Maps and cleans tweet fields for processing       | If                             | Aggregate                     |                                                                                                                                                |
| Aggregate                  | Aggregate                       | Aggregates tweets for sub-workflow input          | Get tweet data                 | Execute Workflow              |                                                                                                                                                |
| Execute Workflow           | Execute Workflow                | Triggers reply generation/posting sub-workflow    | Aggregate                     | When Executed by Another Workflow | "Send tweet data to Sub workflow\nDouble check it triggers the execution workflow"                                                                     |
| When Executed by Another Workflow | Execute Workflow Trigger         | Sub-workflow entry point                           | Execute Workflow               | modify tweet data             | "Execution workflow\nGenerating a reply and posting on X, send status to telegram chat, save reply to mongo db"                                       |
| modify tweet data           | Set                             | Prepares LLM input with current time and tweets   | When Executed by Another Workflow | Basic LLM Chain              | "Add current date\nso the LLM better understand the trending tweets"                                                                                   |
| Basic LLM Chain            | Langchain LLM Chain             | Generates optimized reply for selected tweet      | modify tweet data              | Create Tweet                  | "LLM tweets processing"                                                                                                                               |
| OpenRouter Chat Model1      | Langchain LLM Model             | Provides Grok3 model endpoint for LLM chain       | Basic LLM Chain (AI model input) | Structured Output Parser      |                                                                                                                                                |
| Structured Output Parser    | Langchain Output Parser         | Parses LLM JSON output                             | OpenRouter Chat Model1         | Basic LLM Chain               |                                                                                                                                                |
| Create Tweet               | Twitter                         | Posts reply on X                                   | Basic LLM Chain                | Send a success reply, Send a failed reply | "Reply to the post\nNote: official twitter API has limitations. You can post about 17 times a day before limits apply"                                  |
| Send a success reply       | Telegram                        | Sends Telegram notification on success            | Create Tweet                  | parse reply                  | "Provide status message to telegram\nYou have to specify telegram chat ID"                                                                            |
| Send a failed reply        | Telegram                        | Sends Telegram notification on failure            | Create Tweet                  | parse reply                  | Same as above                                                                                                                                           |
| parse reply                | Set                             | Formats reply data for DB insertion                 | Send a success reply, Send a failed reply | Insert documents            | "Saving reply to database to avoid duplications"                                                                                                      |
| Insert documents           | MongoDB                         | Inserts reply log document                          | parse reply                  | None                        |                                                                                                                                                |
| If5                        | If                               | Checks retries count, controls retry or fail path | If2, If                      | Wait, Send a scraping failure message | "Retry section\nIf no tweets meet criteria, try loop again up to 4 times, then alert Telegram"                                                        |
| Wait                       | Wait                           | Pauses before retrying                              | If5                           | Increment Failure Counter     |                                                                                                                                                |
| Increment Failure Counter  | Set                             | Increments retry failure counter                    | Wait                          | Keyword/Community List        |                                                                                                                                                |
| Send a scraping failure message | Telegram                        | Sends Telegram alert after max retries              | If5                           | None                        | Same as If5 note                                                                                                                                       |
| Schedule Trigger           | Schedule Trigger               | Periodic workflow trigger every 20 minutes          | None                         | Code                         | "Schedule trigger\nEdit timezone!!\nRuns 7am to midnight based on your time zone. Code mimics natural posting time"                                   |
| Code                       | Code                           | Time and probability filter to limit execution      | Schedule Trigger             | If1                          | Same as Schedule Trigger note                                                                                                                        |
| If1                        | If                             | Allows workflow to proceed based on Code output     | Code                         | Find documents, No Operation  |                                                                                                                                                |
| No Operation, do nothing   | No Operation                   | Ends branches with no action                         | If1                          | None                        |                                                                                                                                                |
| No Operation, do nothing1  | No Operation                   | Ends branches with no action                         | If3                          | None                        |                                                                                                                                                |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Credentials**  
   - Configure credentials for:  
     - MongoDB (to store replied tweets)  
     - Telegram Bot API (for status messages and triggers)  
     - Apify API (for Twitter scraping actors)  
     - Twitter OAuth2 API (X account for posting replies)  
     - OpenRouter API (LLM provider Grok3)

2. **Input Triggers**  
   - Add a **Manual Trigger** node named "When clicking ‘Execute workflow’".  
   - Add a **Telegram Trigger** node named "Telegram Trigger" configured to listen for messages.  
   - Connect "Telegram Trigger" to an **If** node ("If3") that checks if message text equals "/reply".  
   - True branch connects to "Find documents"; false branch to a **No Operation** node.

3. **Fetch Already Replied Tweets**  
   - Add a **MongoDB** node "Find documents" to query collection for existing replied tweets' tweet IDs.  
   - Configure projection to include only `tweet_id`.  
   - Connect output to an **Aggregate** node "Aggregate1" (aggregate all item data).  
   - Connect "Aggregate1" to a **Set** node "Keyword/Community List" with JSON containing:  
     - `"keyword_community_list"` array with keywords and community IDs  
     - `"failure"` counter initialized to 0 or incremented from previous run.  

4. **Select Random Keyword or Community ID**  
   - Add a **Set** node "Randomized community, keyword" that picks a random item from `"keyword_community_list"` using expression `.randomItem()`.  
   - Connect to an **If** node "If4" that checks if the selected item matches the regex `\d{19}` (community ID).  
   - True branch leads to an **Apify** node "Community search actor" with configured community search actor ID and JSON body specifying communityIds array containing the random item, requesting 40 tweets.  
   - False branch leads to an **Apify** node "Search keyword actor" configured with keyword search actor ID and JSON body with the random item as a query, and filters such as English language, minimum engagement, verified users, etc.

5. **Fetch Scraped Tweets Dataset**  
   - Both Apify actor nodes connect to a **Get dataset items** node to fetch scraped tweets using the dataset ID from the actor output.  
   - Connect dataset output to an **If** node "If2" that checks if dataset is not empty.

6. **Filter Tweets**  
   - True branch connects to a **Filter** node "Community filter" applying filters:  
     - Tweet text length > 60  
     - Author followers > 100  
     - Tweet not in replied tweet IDs list from MongoDB  
     - Replies > 3, Likes > 10, Views > 100  
     - Language = "en"  
   - Filter output connects to an **If** node "If" checking if filtered tweets are non-empty.

7. **Prepare Tweets for LLM**  
   - True branch connects to a **Set** node "Get tweet data" that extracts and maps tweet fields (id, screen_name, followers count, created_at, tweet_text cleaned of trailing URLs, likes, retweets, replies, views).  
   - Connect to an **Aggregate** node "Aggregate" to aggregate tweets for batch processing.

8. **Call Sub-Workflow for Reply Generation and Posting**  
   - Connect "Aggregate" to an **Execute Workflow** node configured with the reply posting workflow ID.  
   - The sub-workflow starts with an **Execute Workflow Trigger** node.  
   - Next, **Set** node "modify tweet data" adds current ISO timestamp and wraps tweets in a JSON object for LLM input.  
   - Connect to **Langchain LLM Chain** node "Basic LLM Chain" configured with the detailed prompt to filter, score, and generate replies based on tweet data.  
   - Use **OpenRouter Chat Model1** node with Grok3 model as LLM API.  
   - Add **Structured Output Parser** node to parse JSON LLM output.  
   - Connect output to a **Twitter** node "Create Tweet" that posts the reply with inReplyToStatusId set.  
   - On success, send a Telegram message via "Send a success reply" node; on failure, send "Send a failed reply" Telegram node.  
   - Both Telegram nodes connect to a **Set** node "parse reply" that formats reply data.  
   - Finally, insert data into MongoDB via "Insert documents" node.

9. **Retry Logic**  
   - If no tweets pass filters ("If" false branch), connect to an **If** node "If5" that checks if failure count < 3.  
   - True branch waits 3 seconds ("Wait" node), increments failure count ("Increment Failure Counter" node), loops back to "Keyword/Community List".  
   - False branch sends a Telegram failure notification ("Send a scraping failure message").

10. **Scheduling**  
    - Add a **Schedule Trigger** node configured to run every 20 minutes.  
    - Connect to a **Code** node implementing:  
      - Timezone check for Europe/Kyiv active hours 07:00–23:59  
      - Random probability threshold (~28% chance to run)  
    - Connect to an **If** node "If1" that continues only if code returns true, else to a **No Operation** node.  
    - True branch connects to "Find documents" to start the tweet searching loop.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| Find the newest workflow version and documentation at https://dziura.online/automation | Project homepage |
| Documentation and usage guide available at https://docs.google.com/document/d/13okk16lkUOgpbeahMcdmd7BuWkAp_Lx6kQ8BwScbqZk/edit?usp=sharing | User guide |
| Telegram bot setup via https://t.me/BotFather | Telegram bot creation |
| MongoDB connection tutorial https://youtu.be/gB76VdlpX7Y?si=bw_LHLTkXEk8LFO0 | Database setup |
| Apify community node required before importing JSON | Installation note |
| Apify actors used: https://apify.com/api-ninja/x-twitter-advanced-search and https://apify.com/api-ninja/x-twitter-community-search-post-scraper | Scraping resources |
| OpenRouter LLM provider: https://openrouter.ai/ with Grok3 model recommended | LLM service |
| Twitter API limits: ~17 posts/day free tier | Posting constraints |

---

**Disclaimer:** The provided text is extracted exclusively from an automated n8n workflow. All data and operations comply with applicable content policies and legal standards. The workflow handles only public and legal data sources.

---