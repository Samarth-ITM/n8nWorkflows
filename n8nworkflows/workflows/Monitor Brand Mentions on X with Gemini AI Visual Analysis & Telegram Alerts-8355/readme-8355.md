Monitor Brand Mentions on X with Gemini AI Visual Analysis & Telegram Alerts

https://n8nworkflows.xyz/workflows/monitor-brand-mentions-on-x-with-gemini-ai-visual-analysis---telegram-alerts-8355


# Monitor Brand Mentions on X with Gemini AI Visual Analysis & Telegram Alerts

### 1. Workflow Overview

This workflow monitors brand mentions related to Tesla on Twitter (X) by scraping recent tweets containing specific search terms. It uses Google Gemini AI to analyze both the textual content and any images attached to tweets, scoring their relevance and sentiment toward the brand. Based on this analysis, the workflow categorizes posts into relevance levels and logs them accordingly in Airtable. High-relevance posts trigger immediate Telegram alerts to a monitoring group for timely action.

Logical blocks:

- **1.1 Trigger and Configuration**: Periodic or manual start, and setting configuration parameters.
- **1.2 Data Acquisition**: Scraping tweets from Twitter (X) via an Apify actor.
- **1.3 Tweet Filtering and Preparation**: Filtering for valid tweets matching language and non-empty content, and extracting necessary fields.
- **1.4 Duplicate Detection**: Checking if tweets have already been logged in Airtable.
- **1.5 Media Handling and AI Analysis**: Conditional analysis of attached photos using Google Gemini, parsing AI outputs, and combining text and photo assessments.
- **1.6 Relevance Categorization and Logging**: Assigning relevance levels based on AI scores and logging posts into Airtable under appropriate categories.
- **1.7 Notifications**: Sending Telegram alerts for high-relevance posts.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger and Configuration

**Overview:**  
Starts the workflow either manually or on a schedule (every 4 hours). Sets the configuration parameters used throughout the workflow such as language, search terms, Airtable base/table IDs, and scraping limits.

**Nodes Involved:**  
- Manual Executing  
- Schedule Trigger  
- Config  

**Node Details:**  

- **Manual Executing**  
  - Type: Manual Trigger  
  - Role: Allows manual start of the workflow for testing or on-demand runs.  
  - Connections: Output → Config  
  - Failure Modes: None expected beyond manual start omission.

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Automatically triggers the workflow every 4 hours.  
  - Parameters: Interval set to 4 hours.  
  - Connections: Output → Config  
  - Failure Modes: None expected unless scheduler service fails.

- **Config**  
  - Type: Set  
  - Role: Defines constants and runtime parameters such as:  
    - `lang`: "en" (English)  
    - `actorId`: Apify actor ID for Twitter scraping  
    - `searchTerms`: Twitter search query string targeting Tesla and related keywords excluding “Nikola”  
    - `min_faves`: Minimum favorites count filter set to 10  
    - `tweetsToScrape`: Limit to 20 tweets per run  
    - `airtableBaseId`, `airtableTableId`: Airtable identifiers for logging  
  - Connections: Output → Scrape X  
  - Edge Cases: Configuration errors (e.g., wrong IDs or invalid search syntax) could cause scraping or Airtable failures.

---

#### 2.2 Data Acquisition

**Overview:**  
Fetches tweets from Twitter (X) using Apify’s Twitter scraper actor with specified filters and parameters.

**Nodes Involved:**  
- Scrape X  

**Node Details:**  

- **Scrape X**  
  - Type: HTTP Request  
  - Role: Executes a POST request to Apify API to retrieve tweets matching the configured search terms and filters.  
  - Configuration:  
    - URL dynamically built using `actorId` from Config node  
    - Body includes filters disabling certain media types, language filter, search terms, time window (last 24 hours), max items, and min favorites.  
    - Authentication: Bearer token (Apify API)  
  - Connections: Output → If Tweet  
  - Failure Modes: HTTP errors, API limits exceeded, malformed query, expired or invalid bearer token.

---

#### 2.3 Tweet Filtering and Preparation

**Overview:**  
Filters out non-tweet items (such as ads or other X content), matches language, and ensures tweets have content. Prepares standardized fields for downstream processing.

**Nodes Involved:**  
- If Tweet  
- Set Required Fields  
- Not Interested! (noOp fallback)  
- Sticky Note (instructional)  

**Node Details:**  

- **If Tweet**  
  - Type: If  
  - Role: Filters for items where:  
    - type === 'tweet'  
    - language matches Config.lang  
    - the tweet card is empty (no embedded media cards)  
  - Connections:  
    - True → Set Required Fields  
    - False → Not Interested!  
  - Edge Cases: Expression failures if JSON structure differs, missing keys, or unexpected types.

- **Set Required Fields**  
  - Type: Set  
  - Role: Extracts and normalizes key tweet properties into a consistent schema, including:  
    - post ID, URL, text, language, creation date  
    - author username and name  
    - arrays of photo URLs and video URLs extracted from extended media entities  
  - Connections: Output → Loop Over Items  
  - Edge Cases: Missing media fields; empty arrays handled gracefully.

- **Not Interested!**  
  - Type: NoOp  
  - Role: Placeholder node for tweets filtered out at If Tweet.  
  - Connections: None  
  - No failure modes.

- **Sticky Note**  
  - Content: Explains the purpose of filtering to only get tweets, excluding other X content like ads.  
  - Context: Covers If Tweet node.

---

#### 2.4 Duplicate Detection

**Overview:**  
Checks if the current tweet has already been logged in Airtable to avoid duplicates.

**Nodes Involved:**  
- Loop Over Items  
- Post Exists  
- If New  
- Done! (noOp)  

**Node Details:**  

- **Loop Over Items**  
  - Type: Split In Batches  
  - Role: Processes each tweet item individually for sequential handling.  
  - Connections:  
    - Output 1 → Done!  
    - Output 2 → Post Exists  
  - Edge Cases: Large batches may slow processing.

- **Post Exists**  
  - Type: Airtable Search  
  - Role: Searches Airtable for records matching the current post’s ID (`postID`).  
  - Configuration: Uses Airtable API with base and table IDs from Config, filter formula: `{postID}='{{ $json.post.id }}'`.  
  - Connections: Output → If New  
  - Failure Modes: Airtable API errors, authentication issues, or query malformation.

- **If New**  
  - Type: If  
  - Role: Checks if the Airtable search returned empty, indicating the post is new.  
  - Connections:  
    - True → If Has Photos  
    - False → Loop Over Items (next iteration)  
  - Edge Cases: Expression evaluation failure if search result is malformed.

- **Done!**  
  - Type: NoOp  
  - Role: Marks end of processing for current item batch.  
  - No failure modes.

---

#### 2.5 Media Handling and AI Analysis

**Overview:**  
If the tweet contains photos, analyzes them with Google Gemini AI for descriptions and sentiment. Combines photo analysis with post text for a final AI-driven relevance and sentiment assessment.

**Nodes Involved:**  
- If Has Photos  
- Analyze Post Photos (Google Gemini Image Analysis)  
- Get Final Photos Analysis Results (Code)  
- Analyze Text and Photos (Langchain agent with Google Gemini Chat)  
- Google Gemini Chat Model (AI language model)  
- Structured Output Parser  
- Sticky Notes (instructions)  

**Node Details:**  

- **If Has Photos**  
  - Type: If  
  - Role: Checks if photos array is non-empty.  
  - Connections:  
    - True → Analyze Post Photos  
    - False → Analyze Text and Photos (without photo input)  
  - Edge Cases: Expression failures if photos array undefined or null.

- **Analyze Post Photos**  
  - Type: Langchain Google Gemini Image Analysis  
  - Role: Sends photo URLs to Gemini model to generate brief descriptions and sentiment for each image.  
  - Model: Gemini 2.5 Flash  
  - Parameters: Text prompt instructs generating JSON with image index, description, sentiment.  
  - Connections: Output → Get Final Photos Analysis Results  
  - Credentials: Google Gemini API account  
  - Failure Modes: API timeouts, quota exceeded, malformed URLs.

- **Get Final Photos Analysis Results**  
  - Type: Code (JavaScript)  
  - Role: Parses AI response text to extract clean JSON from code blocks and consolidates all image analyses into an array.  
  - Connections: Output → Analyze Text and Photos  
  - Edge Cases: Missing or malformed JSON blocks in AI response.

- **Analyze Text and Photos**  
  - Type: Langchain AI Agent  
  - Role: Synthesizes post text and photo analyses to produce final sentiment, relevance score (0-10), reasoning, and photo summary.  
  - System Message: Defines brand context (Tesla, related keywords), instructions to ignore unrelated topics, output format as JSON with four keys.  
  - Inputs: Post text and photo analysis JSON from previous nodes.  
  - Connections: Output → Switch  
  - Failure Modes: AI model errors, parsing failures, invalid JSON outputs.

- **Google Gemini Chat Model**  
  - Type: Langchain LM Chat Google Gemini  
  - Role: Provides the underlying AI language model for the Langchain agent node.  
  - Credentials: Google Gemini API account  
  - Connections: AI languageModel input and output linked with Analyze Text and Photos node.

- **Structured Output Parser**  
  - Type: Langchain Output Parser Structured  
  - Role: Validates and parses AI agent’s JSON output to a structured object for downstream logic.  
  - Connections: Output → Analyze Text and Photos  
  - Edge Cases: Invalid JSON or schema validation failures.

- **Sticky Notes**  
  - "Analyze the Post Photos" - explains the need and effect of photo analysis on relevance and sentiment.  
  - "Check Post Relevance to the Brand" - explains relevance score thresholds and categories.

---

#### 2.6 Relevance Categorization and Logging

**Overview:**  
Classifies posts into relevance buckets based on AI score and logs them in Airtable with detailed metadata. Medium and high relevance posts are logged separately, with high relevance posts marked as "New" status and medium ones as "For Weekly Review".

**Nodes Involved:**  
- Switch  
- Log the High Relevance Post  
- Log the Medium Relevance Post  

**Node Details:**  

- **Switch**  
  - Type: Switch  
  - Role: Routes posts based on `relevanceScore` from AI output:  
    - High Relevance: score > 7  
    - Medium Relevance: score ≥ 4 and ≤ 7  
    - Not Relevance: score < 4 (loops back to Loop Over Items, effectively skipping logging)  
  - Connections:  
    - High Relevance → Log the High Relevance Post  
    - Medium Relevance → Log the Medium Relevance Post  
    - Not Relevance → Loop Over Items  
  - Edge Cases: Score missing or out of range.

- **Log the High Relevance Post**  
  - Type: Airtable Create  
  - Role: Logs high relevance posts into Airtable with status "New" and detailed fields such as post info, author, sentiment, media URLs, AI reasoning, and photo analysis summary.  
  - Connections: Output → Send High Relevance Post to Monitoring Group  
  - Credentials: Airtable API  
  - Failure Modes: Airtable API limits, auth errors.

- **Log the Medium Relevance Post**  
  - Type: Airtable Create  
  - Role: Logs medium relevance posts with status "For Weekly Review" and similar detailed fields as above.  
  - Connections: Output → Loop Over Items (continue processing)  
  - Credentials: Airtable API  
  - Failure Modes: Same as above.

---

#### 2.7 Notifications

**Overview:**  
Sends Telegram alerts for high relevance posts to a dedicated monitoring group with formatted message including author, sentiment, relevance score, text, and link.

**Nodes Involved:**  
- Send High Relevance Post to Monitoring Group  

**Node Details:**  

- **Send High Relevance Post to Monitoring Group**  
  - Type: Telegram  
  - Role: Sends a formatted HTML message to a Telegram chat/group ID for monitoring purposes.  
  - Message includes:  
    - Author name and username  
    - Sentiment and relevance score  
    - Post text (quoted)  
    - Link to original tweet  
  - Parameters: Chat ID predefined, parse mode HTML, no attribution appended.  
  - Credentials: Telegram API OAuth2 credentials  
  - Connections: Output → Loop Over Items (continue processing)  
  - Failure Modes: Telegram API errors, invalid chat ID, network issues.

---

### 3. Summary Table

| Node Name                         | Node Type                          | Functional Role                             | Input Node(s)                 | Output Node(s)                          | Sticky Note                                                                                               |
|----------------------------------|----------------------------------|---------------------------------------------|------------------------------|---------------------------------------|----------------------------------------------------------------------------------------------------------|
| Manual Executing                  | Manual Trigger                   | Manual start trigger                         |                              | Config                                |                                                                                                          |
| Schedule Trigger                 | Schedule Trigger                 | Periodic start every 4 hours                  |                              | Config                                |                                                                                                          |
| Config                          | Set                              | Set parameters for scraping and logging      | Manual Executing, Schedule Trigger | Scrape X                          |                                                                                                          |
| Scrape X                       | HTTP Request                     | Fetch tweets from Apify Twitter scraper      | Config                       | If Tweet                             |                                                                                                          |
| If Tweet                       | If                               | Filter for tweets with correct language and content | Scrape X                    | Set Required Fields, Not Interested! | ## Get Only Tweets\nFilter out any results that aren’t tweets, such as X advertisements or other non-post content. |
| Set Required Fields             | Set                              | Normalize tweet fields for downstream use    | If Tweet                    | Loop Over Items                      |                                                                                                          |
| Not Interested!                 | NoOp                             | Placeholder for discarded items              | If Tweet                    |                                       |                                                                                                          |
| Loop Over Items                | Split In Batches                 | Process tweets individually                    | Set Required Fields           | Done!, Post Exists                   |                                                                                                          |
| Done!                          | NoOp                             | Marks batch processing completion             | Loop Over Items              |                                       |                                                                                                          |
| Post Exists                   | Airtable Search                  | Check if tweet already logged                  | Loop Over Items              | If New                             |                                                                                                          |
| If New                        | If                               | Filter out already logged tweets               | Post Exists                 | If Has Photos, Loop Over Items       |                                                                                                          |
| If Has Photos                 | If                               | Check if tweet contains photos                 | If New                      | Analyze Post Photos, Analyze Text and Photos | ## Analyze the Post Photos\nIf the post includes photos, they must also be analyzed to improve the relevance score and sentiment status. Additionally, a photo summary should be generated to better capture the mood of the post. |
| Analyze Post Photos           | Langchain Google Gemini Image Analysis | Analyze images for description and sentiment | If Has Photos               | Get Final Photos Analysis Results    |                                                                                                          |
| Get Final Photos Analysis Results | Code                             | Parse AI photo analysis JSON                   | Analyze Post Photos          | Analyze Text and Photos              |                                                                                                          |
| Analyze Text and Photos       | Langchain AI Agent               | Combine text and photo analysis for final sentiment and relevance | If Has Photos, Get Final Photos Analysis Results, If New (no photos) | Switch                        | ## Check Post Relevance to the Brand\n- High relevance: relevance score > 7\n- Medium relevance: relevance score ≥ 4 and ≤ 7\n- Not relevance: relevance score < 4 |
| Google Gemini Chat Model      | Langchain LM Chat Google Gemini | Underlying AI model for analysis               |                               | Analyze Text and Photos             |                                                                                                          |
| Structured Output Parser      | Langchain Output Parser Structured | Parse and validate AI JSON output               | Google Gemini Chat Model     | Analyze Text and Photos             |                                                                                                          |
| Switch                       | Switch                          | Route posts by relevance score                 | Analyze Text and Photos      | Log the High Relevance Post, Loop Over Items, Log the Medium Relevance Post |                                                                                                          |
| Log the High Relevance Post  | Airtable Create                 | Log high relevance posts with status "New"     | Switch                      | Send High Relevance Post to Monitoring Group |                                                                                                          |
| Log the Medium Relevance Post | Airtable Create                 | Log medium relevance posts with status "For Weekly Review" | Switch                      | Loop Over Items                    |                                                                                                          |
| Send High Relevance Post to Monitoring Group | Telegram                        | Send Telegram alert for high relevance posts   | Log the High Relevance Post | Loop Over Items                    |                                                                                                          |
| Sticky Note                  | Sticky Note                     | Instructional notes covering filtering and photo analysis |                              |                                     | See relevant nodes in sections 2.3 and 2.5                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**  
   - Add a **Manual Trigger** node named `Manual Executing`.  
   - Add a **Schedule Trigger** node named `Schedule Trigger`, configure interval to trigger every 4 hours.

2. **Create Configuration Node:**  
   - Add a **Set** node named `Config`.  
   - Define string fields:  
     - `lang` = "en"  
     - `actorId` = "kaitoeasyapi~twitter-x-data-tweet-scraper-pay-per-result-cheapest"  
     - `searchTerms` = `(Tesla OR $TSLA OR Cybertruck OR "Model Y" OR FSD) -Nikola`  
     - `min_faves` = "10"  
     - `tweetsToScrape` = "20"  
     - `airtableBaseId` = your Airtable base ID (e.g., "appfoRsukEzLhzNwS")  
     - `airtableTableId` = your Airtable table ID (e.g., "tblO50wK1GsrweF7j")  
   - Connect outputs from `Manual Executing` and `Schedule Trigger` to `Config`.

3. **Create Scrape Node:**  
   - Add an **HTTP Request** node named `Scrape X`.  
   - Method: POST  
   - URL: `https://api.apify.com/v2/acts/{{ $json.actorId }}/run-sync-get-dataset-items` (use expression to inject `actorId`)  
   - Request Body (JSON):  
     ```json
     {
       "filter:blue_verified": false,
       "filter:consumer_video": false,
       "filter:has_engagement": false,
       "filter:hashtags": false,
       "filter:images": true,
       "filter:links": false,
       "filter:media": false,
       "filter:mentions": false,
       "filter:native_video": false,
       "filter:nativeretweets": false,
       "filter:news": false,
       "filter:pro_video": false,
       "filter:quote": false,
       "filter:replies": false,
       "filter:safe": false,
       "filter:spaces": false,
       "filter:twimg": false,
       "filter:videos": false,
       "filter:vine": false,
       "include:nativeretweets": false,
       "lang": "{{ $json.lang }}",
       "searchTerms": [{{ $json.searchTerms.toJsonString() }}],
       "since": "{{ new Date(Date.now() - 86400000).toISOString().slice(0, 19).replace('T', '_') + '_UTC' }}",
       "maxItems": {{ $json.tweetsToScrape }},
       "queryType": "Latest",
       "min_retweets": 0,
       "min_faves": {{ $json.min_faves }},
       "min_replies": 0,
       "-min_retweets": 0,
       "-min_faves": 0,
       "-min_replies": 0
     }
     ```  
   - Authentication: Bearer token (use your Apify API credentials).  
   - Connect `Config` output to `Scrape X`.

4. **Filter Valid Tweets:**  
   - Add an **If** node named `If Tweet`.  
   - Conditions:  
     - `$json.type === 'tweet'` (boolean true)  
     - `$json.lang === $('Config').item.json.lang` (boolean true)  
     - `$json.card.isEmpty()` (boolean true)  
   - True → `Set Required Fields`  
   - False → `Not Interested!`

5. **Prepare Tweet Fields:**  
   - Add a **Set** node named `Set Required Fields`.  
   - Map fields from input JSON to normalized keys:  
     - `post.id` = `{{$json.id}}`  
     - `post.url` = `{{$json.url}}`  
     - `post.text` = `{{$json.text}}`  
     - `post.lang` = `{{$json.lang}}`  
     - `post.createdAt` = `{{$json.createdAt.toDateTime().toISO()}}`  
     - `author.userName` = `{{$json.author.userName}}`  
     - `author.name` = `{{$json.author.name}}`  
     - `media.photos` = extract photo URLs from `$json.extendedEntities.media` where type is "photo"  
     - `media.videos` = extract highest bitrate mp4 video URLs from `$json.extendedEntities.media`  
   - Connect `If Tweet` (true) → `Set Required Fields`.

6. **Split Items for Individual Processing:**  
   - Add a **Split In Batches** node named `Loop Over Items`.  
   - Connect `Set Required Fields` → `Loop Over Items`.

7. **Detect Duplicates in Airtable:**  
   - Add an **Airtable** node named `Post Exists`.  
   - Operation: Search  
   - Base ID and Table ID from `Config` node fields.  
   - Filter formula: `{postID}='{{ $json.post.id }}'`  
   - Connect `Loop Over Items` (output 2) → `Post Exists`.

8. **Filter New Posts:**  
   - Add an **If** node named `If New`.  
   - Condition: `$json.isEmpty()` is true (means no existing record found)  
   - True → `If Has Photos`  
   - False → `Loop Over Items` (to continue next item)  
   - Connect `Post Exists` → `If New`.

9. **Check for Photos:**  
   - Add an **If** node named `If Has Photos`.  
   - Condition: `$('Loop Over Items').item.json.media.photos.isNotEmpty()` is true  
   - True → `Analyze Post Photos`  
   - False → `Analyze Text and Photos` (direct, no photo input)  
   - Connect `If New` → `If Has Photos`.

10. **Analyze Photos with Google Gemini:**  
    - Add a **Langchain Google Gemini Image Analysis** node named `Analyze Post Photos`.  
    - Input: Photos URLs joined as comma-separated string from current item.  
    - Text prompt instructs analysis of each image: description + sentiment in JSON format.  
    - Model: "models/gemini-2.5-flash"  
    - Credentials: Google Gemini API  
    - Connect `If Has Photos` (true) → `Analyze Post Photos`.

11. **Parse Photo Analysis Results:**  
    - Add a **Code** node named `Get Final Photos Analysis Results`.  
    - JavaScript to parse JSON from Gemini output code blocks and aggregate into a single array `photos_results`.  
    - Connect `Analyze Post Photos` → `Get Final Photos Analysis Results`.

12. **Analyze Text and Photos Combined:**  
    - Add a **Langchain AI Agent** node named `Analyze Text and Photos`.  
    - Input text: post text + JSON photo analysis or fallback string if no photos.  
    - System message: Brand strategist instructions with Tesla context and output JSON format with fields:  
      - `overallSentiment` (Positive/Negative/Neutral)  
      - `relevanceScore` (0-10)  
      - `reasoning` (one sentence)  
      - `photosSummary` (one sentence or empty string if no photos)  
    - Pass through binary images enabled.  
    - Connect `If Has Photos` (false) and `Get Final Photos Analysis Results` → `Analyze Text and Photos`.

13. **Connect AI Language Model and Output Parser:**  
    - Add **Google Gemini Chat Model** node with Google Gemini credentials.  
    - Connect to `Analyze Text and Photos` AI language model input.  
    - Add **Structured Output Parser** node with example JSON schema matching output fields.  
    - Connect `Google Gemini Chat Model` output → `Structured Output Parser` → back to `Analyze Text and Photos` (output parser input).

14. **Route by Relevance Score:**  
    - Add a **Switch** node named `Switch`.  
    - Rules:  
      - High Relevance: `relevanceScore` > 7  
      - Medium Relevance: `relevanceScore` >= 4 and <=7  
      - Not Relevant: `relevanceScore` < 4  
    - Connect `Analyze Text and Photos` output → `Switch`.

15. **Log High Relevance Posts:**  
    - Add an **Airtable** create node named `Log the High Relevance Post`.  
    - Map all relevant fields from current item and AI output to Airtable columns.  
    - Set status = "New".  
    - Connect `Switch` (High Relevance) → `Log the High Relevance Post`.

16. **Log Medium Relevance Posts:**  
    - Add an **Airtable** create node named `Log the Medium Relevance Post`.  
    - Map fields similarly.  
    - Set status = "For Weekly Review".  
    - Connect `Switch` (Medium Relevance) → `Log the Medium Relevance Post`.

17. **Handle Not Relevant Posts:**  
    - Connect `Switch` (Not Relevance) → `Loop Over Items` (to continue loop without logging).

18. **Send Telegram Alerts:**  
    - Add a **Telegram** node named `Send High Relevance Post to Monitoring Group`.  
    - Set chat ID to your Telegram monitoring group.  
    - Compose HTML message with author, sentiment, relevance score, post text as blockquote, and tweet URL.  
    - Connect `Log the High Relevance Post` → `Send High Relevance Post to Monitoring Group`.

19. **Loop Continuation:**  
    - Connect outputs of `Send High Relevance Post to Monitoring Group`, `Log the Medium Relevance Post`, and `Done!` back to `Loop Over Items` to continue processing remaining tweets.

20. **Add Sticky Notes:**  
    - Add sticky notes with instructions for:  
      - Tweet filtering rationale near `If Tweet` node.  
      - Photo analysis explanation near `Analyze Post Photos`.  
      - Relevance score interpretation near `Switch`.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                          | Context or Link                                                                                            |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------|
| This workflow uses Google Gemini (PaLM) API via Langchain nodes for advanced AI analysis of text and images.                                                                                                                                                                                      | Google Gemini API: https://developers.generativeai.google/                                                |
| Apify’s Twitter Scraper actor is used to gather tweets based on defined search terms and filters. Ensure your Apify account has enough quota and the actor ID is valid.                                                                                                                           | Apify Twitter Scraper: https://apify.com/kaitoeasyapi/twitter-x-data-tweet-scraper-pay-per-result-cheapest |
| Airtable is used to store and manage logged posts, differentiating statuses for workflow tracking and review. Correct API keys and permission scopes are required.                                                                                                                                  | Airtable API Docs: https://airtable.com/api                                                                    |
| Telegram Bot API is used for sending alerts; ensure your bot token and chat ID are correctly configured, and the bot is added to the monitoring group with proper permissions.                                                                                                                     | Telegram Bot API: https://core.telegram.org/bots/api                                                        |
| The workflow filters out non-tweet X content to avoid noise such as advertisements or native retweets, focusing analysis on original user posts.                                                                                                                                                 | See Sticky Note on "Get Only Tweets"                                                                        |
| Relevance scoring thresholds are defined as: High (>7), Medium (4–7), and Not Relevant (<4), facilitating differentiated handling and prioritization.                                                                                                                                             | See Sticky Note on "Check Post Relevance to the Brand"                                                     |
| The photo analysis improves the quality of sentiment and relevance scoring by integrating visual context, which is crucial for image-heavy tweets.                                                                                                                                               | See Sticky Note on "Analyze the Post Photos"                                                               |

---

**Disclaimer:** The provided text and workflow description originate exclusively from an automated n8n workflow. The process respects all content policies and contains no illegal, offensive, or protected elements. All data manipulated is legal and publicly available.