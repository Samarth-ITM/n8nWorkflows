Collect Viral Instagram AI Art with GPT-4 Analysis and Telegram Delivery

https://n8nworkflows.xyz/workflows/collect-viral-instagram-ai-art-with-gpt-4-analysis-and-telegram-delivery-11421


# Collect Viral Instagram AI Art with GPT-4 Analysis and Telegram Delivery

### 1. Workflow Overview

This workflow automates the collection, analysis, and distribution of viral AI-generated Instagram art posts tagged with specific hashtags. It targets AI art enthusiasts, marketers, and content curators aiming to monitor trends and share insights in Japanese via Telegram and Slack.

The workflow is logically divided into four main blocks:

- **1.1 Scheduled Data Collection:** Automated scraping of Instagram posts from selected AI art-related hashtags, triggered daily and weekly.
- **1.2 Data Deduplication and Filtering:** Merges newly scraped data with existing records in Google Sheets to identify new posts with significant engagement.
- **1.3 AI-Powered Image Analysis and Enrichment:** Utilizes GPT-4 to analyze artwork images, extracting style, mood, and quality metrics, and translates captions to Japanese.
- **1.4 Multi-Platform Distribution and Reporting:** Sends analyzed posts to Telegram and Slack and generates weekly AI trend reports with detailed statistics and predictions.

Additionally, an error handling mechanism captures workflow errors and alerts administrators via Telegram.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Data Collection

- **Overview:**  
  Initiates periodic scraping of Instagram posts tagged with AI art-related hashtags to gather fresh content daily and weekly.

- **Nodes Involved:**  
  - Daily Collection Trigger  
  - Weekly Report Trigger  
  - Scrape Instagram Posts  
  - Save to Database  

- **Node Details:**

  1. **Daily Collection Trigger**  
     - Type: Schedule Trigger  
     - Role: Fires daily at 18:00 (6 PM) to start data collection.  
     - Configuration: Interval set to trigger at hour 18.  
     - Inputs: None  
     - Outputs: Connects to "Scrape Instagram Posts"  
     - Edge Cases: Missed triggers if workflow inactive; time zone considerations.

  2. **Weekly Report Trigger**  
     - Type: Schedule Trigger  
     - Role: Fires weekly every 7 days at 10:00 AM to generate trend reports.  
     - Configuration: Days interval set to 7, trigger hour 10.  
     - Inputs: None  
     - Outputs: Connects to "Get Weekly Data" (part of reporting block)  
     - Edge Cases: Same as above; ensures weekly reports run independently.

  3. **Scrape Instagram Posts**  
     - Type: Apify Node (Instagram Scraper Actor)  
     - Role: Scrapes up to 20 recent posts per hashtag from six AI art hashtags.  
     - Configuration: Uses Apify Instagram Scraper with direct URLs for hashtags #AIart, #midjourney, #stablediffusion, #aiartwork, #generativeart, #aiartcommunity; results type set to "posts" with search limit 1 and results limit 20.  
     - Inputs: Trigger from "Daily Collection Trigger"  
     - Outputs: Connects to "Save to Database"  
     - Authentication: Requires Apify OAuth2 credentials.  
     - Edge Cases: API limits, Instagram scraping restrictions, network timeouts, incomplete data returns.

  4. **Save to Database**  
     - Type: Google Sheets Node  
     - Role: Appends or updates Instagram posts data into a Google Sheets document in the "Posts" sheet.  
     - Configuration: Maps post fields (url, likes, caption, post_id, comments, hashtags, image_url, timestamp, owner_username) with initial empty fields for analysis and flags for distribution status (sent_to_slack, sent_to_telegram). Uses USER_ENTERED cell format.  
     - Inputs: From "Scrape Instagram Posts"  
     - Outputs: Connects to "Get Existing Posts"  
     - Credentials: Google Sheets API with a document ID credential.  
     - Edge Cases: API quota limits, data format mismatches, sheet access permission issues.

---

#### 2.2 Data Deduplication and Filtering

- **Overview:**  
  Retrieves existing posts from Google Sheets, merges new data to identify unique posts, and filters posts that have not been previously sent and exceed a like threshold.

- **Nodes Involved:**  
  - Get Existing Posts  
  - Check Duplicates  
  - Filter New Viral Posts  

- **Node Details:**

  1. **Get Existing Posts**  
     - Type: Google Sheets Node  
     - Role: Fetches all existing posts data from the "Posts" sheet for duplicate checking.  
     - Inputs: From "Save to Database"  
     - Outputs: Connects to "Check Duplicates"  
     - Edge Cases: Large datasets may cause slow retrieval; API limits.

  2. **Check Duplicates**  
     - Type: Merge Node  
     - Role: Combines new scraped posts with existing data to detect duplicates by matching post IDs.  
     - Configuration: Combine mode, no fuzzy matching, first match only.  
     - Inputs: New posts and existing posts (two inputs)  
     - Outputs: Connects to "Filter New Viral Posts"  
     - Edge Cases: Incorrect merge keys or missing fields leading to false duplicates or misses.

  3. **Filter New Viral Posts**  
     - Type: If Node  
     - Role: Filters posts to those that have not been sent to Telegram, have a non-empty caption, and have at least 100 likes.  
     - Configuration: Three conditions combined by AND:  
       - sent_to_telegram does not exist or is empty  
       - Caption is not empty  
       - likesCount >= 100  
     - Inputs: From "Check Duplicates"  
     - Outputs: True branch connects to "Calculate Engagement Score"  
     - Edge Cases: Posts missing likesCount or captions may be incorrectly filtered out.

---

#### 2.3 AI-Powered Image Analysis and Enrichment

- **Overview:**  
  Iterates over filtered posts to analyze artwork using GPT-4, parses AI-generated structured analysis, translates captions to Japanese, downloads images for posting, and calculates engagement metrics.

- **Nodes Involved:**  
  - Calculate Engagement Score  
  - Loop Over Items  
  - AI Image Analysis  
  - Analyze Artwork  
  - Parse AI Analysis  
  - Translate to Japanese  
  - Download Image  

- **Node Details:**

  1. **Calculate Engagement Score**  
     - Type: Set Node  
     - Role: Computes an engagement score based on likes, comments, and post age (hours), assigns viral tier and primary hashtag category.  
     - Configuration:  
       - engagement_score = ((likes + comments * 3) / post age in hours) * 100, rounded  
       - viral_tier: MEGA VIRAL (>=10,000 likes), VIRAL (>=1,000), TRENDING (>=500), RISING (default)  
       - primary_category: determined by keywords in caption (portrait, landscape, anime, abstract, fantasy, sci-fi, or general)  
     - Inputs: From "Filter New Viral Posts"  
     - Outputs: Connects to "Loop Over Items"  
     - Edge Cases: Posts with future timestamps or zero age cause division by zero handled by max(1). Missing likes/comments default to zero.

  2. **Loop Over Items**  
     - Type: SplitInBatches Node  
     - Role: Processes posts one at a time for AI analysis to avoid rate limits.  
     - Configuration: Batch size defaults to 1 (implicit).  
     - Inputs: From "Calculate Engagement Score"  
     - Outputs: Connects to "Analyze Artwork"  
     - Edge Cases: Large datasets cause long processing times; batch reset is disabled.

  3. **AI Image Analysis**  
     - Type: OpenAI GPT-4 Chat Node (LangChain)  
     - Role: Runs GPT-4 model for vision-based image analysis with controlled temperature and max tokens.  
     - Configuration: Model gpt-4o, maxTokens 500, temperature 0.3  
     - Inputs: Implicitly used by "Analyze Artwork" for chaining.  
     - Outputs: AI response passed to "Analyze Artwork"  
     - Credentials: OpenAI API key required.  
     - Edge Cases: API rate limits, response timeouts, partial outputs.

  4. **Analyze Artwork**  
     - Type: LangChain Chain LLM Node  
     - Role: Sends structured prompt with image URL and caption to GPT-4 for a JSON-formatted analysis including art style, color palette, mood, technique, subject, quality score, trending potential, similar artists, and Japanese description.  
     - Inputs: From "Loop Over Items"  
     - Outputs: Connects to "Parse AI Analysis"  
     - Edge Cases: Malformed or incomplete AI response; node expects strict JSON.

  5. **Parse AI Analysis**  
     - Type: Code Node (JavaScript)  
     - Role: Parses AI JSON response; falls back to default values if parsing fails; merges AI data with original post data.  
     - Inputs: From "Analyze Artwork"  
     - Outputs: Connects to "Translate to Japanese"  
     - Edge Cases: JSON parse errors handled gracefully; missing fields defaulted.

  6. **Translate to Japanese**  
     - Type: DeepL Node  
     - Role: Translates the original post caption into Japanese for local audience engagement.  
     - Inputs: From "Parse AI Analysis"  
     - Outputs: Connects to "Download Image"  
     - Credentials: DeepL API key required.  
     - Edge Cases: Translation API failures or limits.

  7. **Download Image**  
     - Type: HTTP Request Node  
     - Role: Downloads post image as binary data for sending to Telegram (and Slack optionally).  
     - Configuration: Response format set to file (binary).  
     - Inputs: From "Translate to Japanese"  
     - Outputs: Connects to "Telegram Enabled?" and "Slack Enabled?"  
     - Edge Cases: Image URL invalid or network issues.

---

#### 2.4 Multi-Platform Distribution and Reporting

- **Overview:**  
  Checks platform enablement flags, sends analyzed posts to Telegram and Slack, updates post status in Google Sheets, enforces rate limiting, generates weekly trend reports, and handles errors.

- **Nodes Involved:**  
  - Telegram Enabled?  
  - Slack Enabled?  
  - Send to Telegram1 (sendPhoto)  
  - Send to Slack  
  - Update Post Status  
  - Rate Limit Wait  
  - Get Weekly Data  
  - Calculate Weekly Stats  
  - Generate Report  
  - Create Trend Report  
  - Send Report to Telegram  
  - Send Report to Slack  
  - Error Trigger  
  - Send Error Alert  

- **Node Details:**

  1. **Telegram Enabled?**  
     - Type: If Node  
     - Role: Checks environment variable ENABLE_TELEGRAM (defaults to true) to decide if posts are sent to Telegram.  
     - Inputs: From "Download Image"  
     - Outputs: True branch connects to "Send to Telegram1"  
     - Edge Cases: Missing env var defaults to true.

  2. **Slack Enabled?**  
     - Type: If Node  
     - Role: Checks ENABLE_SLACK (defaults to false) to enable Slack posting.  
     - Inputs: From "Download Image"  
     - Outputs: True branch connects to "Send to Slack"  
     - Edge Cases: Slack integration optional; disabled by default.

  3. **Send to Telegram1**  
     - Type: Telegram Node (sendPhoto)  
     - Role: Sends the analyzed post's photo and formatted caption with viral tier, style, mood, colors, quality, trending potential, Japanese description, and engagement metrics.  
     - Inputs: From "Telegram Enabled?"  
     - Outputs: Connects to "Update Post Status"  
     - Configuration: Uses Telegram chat ID from env var TELEGRAM_CHAT_ID, Markdown parse mode, includes link to original Instagram post.  
     - Edge Cases: Telegram API failures, photo size limits.

  4. **Send to Slack**  
     - Type: Slack Node  
     - Role: Posts a brief message announcing the AI art discovery with viral tier.  
     - Inputs: From "Slack Enabled?"  
     - Outputs: Connects to "Update Post Status"  
     - Configuration: Uses Slack webhook or configured Slack credentials.  
     - Edge Cases: Slack API rate limits or auth failures.

  5. **Update Post Status**  
     - Type: Google Sheets Node  
     - Role: Updates analyzed post data in the "Posts" sheet with AI analysis, art style, color palette, processed timestamp, and sent flags for Telegram and Slack.  
     - Inputs: From both "Send to Telegram1" and "Send to Slack" (merged)  
     - Outputs: Connects to "Rate Limit Wait"  
     - Edge Cases: Sheet update failures or conflicts.

  6. **Rate Limit Wait**  
     - Type: Wait Node  
     - Role: Adds delay between processing batches to avoid hitting API rate limits.  
     - Inputs: From "Update Post Status"  
     - Outputs: Connects back to "Loop Over Items" for next batch processing.  
     - Edge Cases: Excessive delays may slow workflow; no explicit wait time configured (default wait).

  7. **Get Weekly Data**  
     - Type: Google Sheets Node  
     - Role: Retrieves all posts data for the last 7 days to prepare trend analytics report.  
     - Inputs: From "Weekly Report Trigger"  
     - Outputs: Connects to "Calculate Weekly Stats"  
     - Edge Cases: Large data volumes.

  8. **Calculate Weekly Stats**  
     - Type: Code Node (JavaScript)  
     - Role: Filters posts from last 7 days, calculates totals (posts, likes, comments), average engagement, style/mood/color breakdowns, peak activity day, top 5 posts by engagement, and trending art styles.  
     - Inputs: From "Get Weekly Data"  
     - Outputs: Connects to "Generate Report"  
     - Edge Cases: Empty dataset, missing fields.

  9. **Generate Report**  
     - Type: OpenAI GPT-4 Chat Node (LangChain)  
     - Role: Generates AI narrative summarizing weekly statistics using GPT-4 with moderate creativity and length (max 1500 tokens).  
     - Inputs: From "Calculate Weekly Stats"  
     - Outputs: Connects to "Create Trend Report"  
     - Edge Cases: API errors.

  10. **Create Trend Report**  
      - Type: LangChain Chain LLM Node  
      - Role: Formats the AI-generated report into a structured Japanese weekly trend report with sections for summary, trend analysis, creator suggestions, predictions, and highlights.  
      - Inputs: From "Generate Report" (AI languageModel output)  
      - Outputs: Connects to "Send Report to Telegram" and "Send Report to Slack"  
      - Edge Cases: Formatting errors.

  11. **Send Report to Telegram**  
      - Type: Telegram Node  
      - Role: Sends the formatted weekly trend report to Telegram chat.  
      - Inputs: From "Create Trend Report"  
      - Configuration: Markdown parse mode, uses TELEGRAM_CHAT_ID env var.  
      - Edge Cases: Telegram API issues.

  12. **Send Report to Slack**  
      - Type: Slack Node  
      - Role: Sends a short notification that the weekly AI art trend report is available.  
      - Inputs: From "Create Trend Report"  
      - Edge Cases: Slack integration optional.

  13. **Error Trigger**  
      - Type: Error Trigger Node  
      - Role: Captures any workflow execution errors.  
      - Inputs: None (system level)  
      - Outputs: Connects to "Send Error Alert"  
      - Edge Cases: Ensures no errors go unnoticed.

  14. **Send Error Alert**  
      - Type: Telegram Node  
      - Role: Sends detailed error alerts to Telegram including workflow name, node, error message, timestamp, and link to execution log.  
      - Inputs: From "Error Trigger"  
      - Configuration: Markdown parse mode, TELEGRAM_CHAT_ID required.  
      - Edge Cases: Failure to send alerts if Telegram down.

---

### 3. Summary Table

| Node Name             | Node Type                         | Functional Role                                | Input Node(s)                | Output Node(s)                   | Sticky Note                                                                                  |
|-----------------------|----------------------------------|-----------------------------------------------|-----------------------------|---------------------------------|----------------------------------------------------------------------------------------------|
| Daily Collection Trigger | Schedule Trigger                 | Triggers daily Instagram scraping             | None                        | Scrape Instagram Posts           | ### 📥 Step 1: Data Collection Scrapes Instagram posts from 6 AI art hashtags using Apify, saves to Google Sheets database |
| Weekly Report Trigger  | Schedule Trigger                 | Triggers weekly trend report generation        | None                        | Get Weekly Data                 | ### 📊 Weekly Analytics Generates AI-powered trend report analyzing data                       |
| Scrape Instagram Posts | Apify Instagram Scraper          | Scrapes posts from AI art hashtags             | Daily Collection Trigger    | Save to Database                |                                                                                              |
| Save to Database       | Google Sheets                   | Stores scraped posts in Google Sheets          | Scrape Instagram Posts      | Get Existing Posts              |                                                                                              |
| Get Existing Posts     | Google Sheets                   | Retrieves existing posts for duplicate check   | Save to Database            | Check Duplicates                | ### 🔍 Step 2: Duplicate Check & Filter Merges with existing data to identify new posts only, filters by engagement (100+ likes) |
| Check Duplicates       | Merge                           | Combines new and existing posts to find duplicates | Get Existing Posts          | Filter New Viral Posts          |                                                                                              |
| Filter New Viral Posts | If                              | Filters posts not sent and with 100+ likes     | Check Duplicates            | Calculate Engagement Score      |                                                                                              |
| Calculate Engagement Score | Set                           | Calculates engagement score and viral tier     | Filter New Viral Posts      | Loop Over Items                |                                                                                              |
| Loop Over Items        | SplitInBatches                  | Processes posts one by one for AI analysis     | Calculate Engagement Score  | Analyze Artwork                | ### 🧠 Step 3: AI Analysis OpenAI Vision analyzes each image for style, mood, colors, quality score, and generates Japanese description |
| AI Image Analysis      | LangChain OpenAI GPT-4 Chat     | GPT-4 vision-based image analysis model        | Implicit in Analyze Artwork | Analyze Artwork                |                                                                                              |
| Analyze Artwork        | LangChain Chain LLM             | Sends prompt for detailed artwork analysis     | Loop Over Items             | Parse AI Analysis              |                                                                                              |
| Parse AI Analysis      | Code                           | Parses AI JSON analysis, merges with original data | Analyze Artwork             | Translate to Japanese          |                                                                                              |
| Translate to Japanese  | DeepL                          | Translates caption to Japanese                   | Parse AI Analysis           | Download Image                 |                                                                                              |
| Download Image         | HTTP Request                   | Downloads image binary data for messaging       | Translate to Japanese       | Telegram Enabled?, Slack Enabled? |                                                                                              |
| Telegram Enabled?      | If                             | Checks if Telegram posting is enabled           | Download Image              | Send to Telegram1              | ### 📤 Step 4: Multi-Platform Distribution Sends formatted posts to Telegram and/or Slack based on configuration |
| Slack Enabled?         | If                             | Checks if Slack posting is enabled               | Download Image              | Send to Slack                  |                                                                                              |
| Send to Telegram1      | Telegram                       | Sends photo and caption to Telegram chat        | Telegram Enabled?           | Update Post Status             |                                                                                              |
| Send to Slack          | Slack                         | Sends brief post announcement to Slack          | Slack Enabled?              | Update Post Status             |                                                                                              |
| Update Post Status     | Google Sheets                 | Updates post record with analysis and sent flags| Send to Telegram1, Send to Slack | Rate Limit Wait           |                                                                                              |
| Rate Limit Wait        | Wait                          | Delays to prevent API rate limits                | Update Post Status          | Loop Over Items               |                                                                                              |
| Get Weekly Data        | Google Sheets                 | Fetches all posts data for the last 7 days      | Weekly Report Trigger       | Calculate Weekly Stats         |                                                                                              |
| Calculate Weekly Stats | Code                          | Calculates weekly statistics and trending data  | Get Weekly Data             | Generate Report               |                                                                                              |
| Generate Report        | LangChain OpenAI GPT-4 Chat    | Generates AI summary report                       | Calculate Weekly Stats      | Create Trend Report           |                                                                                              |
| Create Trend Report    | LangChain Chain LLM            | Formats the comprehensive weekly report in Japanese | Generate Report             | Send Report to Telegram, Send Report to Slack |                                                                                              |
| Send Report to Telegram| Telegram                      | Sends weekly trend report to Telegram chat       | Create Trend Report         | None                         |                                                                                              |
| Send Report to Slack   | Slack                        | Sends notification of weekly report to Slack     | Create Trend Report         | None                         |                                                                                              |
| Error Trigger          | Error Trigger                 | Catches workflow errors                           | None                       | Send Error Alert              | ### ⚠️ Error Handling Captures any workflow errors and sends detailed alerts to Telegram     |
| Send Error Alert       | Telegram                      | Sends detailed error alerts to Telegram           | Error Trigger               | None                         |                                                                                              |
| Workflow Description   | Sticky Note                  | Provides detailed workflow overview and setup instructions | None                       | None                         | ## 🎨 Instagram AI Art Intelligence Hub Advanced workflow for collecting, analyzing, and distributing viral AI art content with comprehensive trend analytics. Includes setup requirements and environment variables. |

---

### 4. Reproducing the Workflow from Scratch

1. **Set up environment variables:**  
   - TELEGRAM_CHAT_ID (Telegram chat ID)  
   - ENABLE_TELEGRAM (true/false, default true)  
   - ENABLE_SLACK (true/false, default false)  
   - SLACK_CHANNEL (optional)  
   - Google Sheets document ID (credential)

2. **Create Schedule Trigger nodes:**  
   - **Daily Collection Trigger:** Set to trigger daily at 18:00.  
   - **Weekly Report Trigger:** Set to trigger every 7 days at 10:00.

3. **Instagram Scraper node:**  
   - Use Apify Instagram Scraper actor with OAuth2 credentials.  
   - Configure direct URLs for six AI art hashtags: #AIart, #midjourney, #stablediffusion, #aiartwork, #generativeart, #aiartcommunity.  
   - Limit results to 20 posts per hashtag.

4. **Google Sheets nodes:**  
   - **Save to Database:** Append or update mode, map Instagram post fields (url, likes, caption, id, comments, hashtags, image_url, timestamp, owner_username) to "Posts" sheet.  
   - **Get Existing Posts:** Retrieve all rows from "Posts" sheet.

5. **Merge node ("Check Duplicates"):**  
   - Combine new and existing posts on post_id, no fuzzy matching, first match only.

6. **If node ("Filter New Viral Posts"):**  
   - Condition: sent_to_telegram not exists AND caption not empty AND likesCount >= 100.

7. **Set node ("Calculate Engagement Score"):**  
   - Calculate engagement_score as ((likes + comments*3) / post_age_hours) * 100, rounded.  
   - Assign viral_tier based on likes thresholds.  
   - Determine primary_category from caption keywords.

8. **SplitInBatches node ("Loop Over Items"):**  
   - Batch size 1, no reset.

9. **LangChain OpenAI GPT-4 Chat node ("AI Image Analysis"):**  
   - Model: gpt-4o, maxTokens 500, temperature 0.3.

10. **LangChain Chain LLM node ("Analyze Artwork"):**  
    - Prompt includes image URL and caption, requests structured JSON analysis of art style, color palette, mood, technique, subject, quality_score, trending_potential, similar artists, and Japanese description.

11. **Code node ("Parse AI Analysis"):**  
    - Parse JSON from AI response, fallback defaults if parse fails, merge with original post data.

12. **DeepL node ("Translate to Japanese"):**  
    - Translate original caption text to Japanese.

13. **HTTP Request node ("Download Image"):**  
    - Download post image as binary for messaging.

14. **If nodes ("Telegram Enabled?" and "Slack Enabled?"):**  
    - Check environment flags to enable posting on respective platforms.

15. **Telegram node ("Send to Telegram1"):**  
    - Send photo with caption including viral tier, style, mood, colors, quality, trending potential, Japanese description, engagement score, and link to original post.  
    - Use Telegram API with chat ID from env vars.

16. **Slack node ("Send to Slack"):**  
    - Post brief message announcing AI art discovery and viral tier.

17. **Google Sheets node ("Update Post Status"):**  
    - Update post with AI analysis, art style, color palette, processed timestamp, sent flags (Telegram, Slack), and engagement score.

18. **Wait node ("Rate Limit Wait"):**  
    - Insert delay between batches to prevent API rate limits.

19. **Weekly Analytics nodes:**  
    - **Get Weekly Data:** Retrieve all posts data.  
    - **Calculate Weekly Stats:** Filter last 7 days, compute statistics, top posts, trending styles, peak day.  
    - **Generate Report:** Use GPT-4 to generate narrative summary.  
    - **Create Trend Report:** Format report in Japanese with sections and emojis.  
    - **Send Report to Telegram:** Send weekly report to Telegram chat.  
    - **Send Report to Slack:** Send weekly report notification to Slack.

20. **Error Handling:**  
    - **Error Trigger:** Catch workflow errors.  
    - **Send Error Alert:** Send detailed error message via Telegram.

21. **Sticky Notes:** Add documentation sticky notes describing each step for user clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                       | Context or Link                                                                                              |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| Advanced workflow for collecting, analyzing, and distributing viral AI art content with comprehensive trend analytics. Key features include multi-source Instagram scraping, duplicate detection, AI-based image analysis with GPT-4, Japanese translation via DeepL, multi-platform distribution (Telegram and Slack), weekly trend reports, and real-time error alerts via Telegram. | Workflow Description Node (Sticky Note)                                                                    |
| Setup requires accounts and API keys: Apify for Instagram scraping, OpenAI for GPT-4 analysis, DeepL for translation, Google Sheets for data storage, Telegram bot for messaging, and optionally Slack App for Slack integration.                                                                                                                                     | Workflow Description Node (Sticky Note)                                                                    |
| Environment variables control platform enablement and credentials (TELEGRAM_CHAT_ID, ENABLE_TELEGRAM, ENABLE_SLACK, SLACK_CHANNEL, Google Sheets document ID).                                                                                                                                                                                                 | Workflow Description Node (Sticky Note)                                                                    |
| Video and project credits are not included in the workflow but can be added externally.                                                                                                                                                                                                                                                                           | N/A                                                                                                         |
| Workflow uses best practices for batch processing, rate limiting, error handling, and structured AI prompts to ensure robustness and maintainability.                                                                                                                                                                                                             | Analysis from workflow design                                                                                |

---

**This document enables advanced users and AI agents to fully understand, reproduce, and maintain the "Collect Viral Instagram AI Art with GPT-4 Analysis and Telegram Delivery" workflow with clear delineation of functional blocks, node configurations, data flow, and integration points.**