Cross-Platform Brand Monitoring & Analysis with AnySite API and GPT

https://n8nworkflows.xyz/workflows/cross-platform-brand-monitoring---analysis-with-anysite-api-and-gpt-10251


# Cross-Platform Brand Monitoring & Analysis with AnySite API and GPT

---

# Cross-Platform Brand Monitoring & Analysis with AnySite API and GPT - Comprehensive Reference

---

## 1. Workflow Overview

This workflow provides an automated, multi-platform social media monitoring and analysis system designed primarily for marketing, PR, and customer success teams. It continuously tracks brand mentions and relevant keywords across Reddit, LinkedIn, Instagram, and X (Twitter) using AnySite.io APIs, enriches data with comments and metadata, analyzes sentiment and trends via an AI Agent powered by GPT-4, and sends actionable email alerts. The workflow also ensures data deduplication by storing posts in a PostgreSQL data table and preventing re-processing of previously seen posts.

### Logical Blocks

- **1.1 Triggers & Keyword Input**  
  Initiates workflow execution either on schedule or manually and retrieves the list of brand-monitoring keywords.

- **1.2 Cross-Platform Search Phase**  
  Performs parallel searches on Reddit, LinkedIn, Instagram, and X for posts matching each keyword.

- **1.3 Deduplication Check & Data Insertion**  
  Checks if posts are already stored in the database to avoid duplicates. Inserts new posts into the data table.

- **1.4 Platform-Specific Data Extraction**  
  For newly inserted posts, fetches detailed post data and comments from AnySite API endpoints.

- **1.5 Comment Processing and Post Updates**  
  Converts raw comments data into structured JSON strings and updates the database records.

- **1.6 AI Analysis & Reporting**  
  The AI Agent retrieves all new post data, performs normalization, sentiment analysis, insights generation, and composes a detailed report.

- **1.7 Email Notification**  
  Sends the AI-generated report via Gmail to specified recipients.

- **1.8 Scheduling & Manual Execution**  
  Supports both scheduled automated runs and manual trigger execution for testing or on-demand processing.

---

## 2. Block-by-Block Analysis

### 2.1 Triggers & Keyword Input

**Overview:**  
Starts the workflow execution either manually or on schedule and fetches the list of keywords to monitor.

**Nodes Involved:**  
- `When clicking ‘Execute workflow’` (Manual Trigger)  
- `Schedule Trigger` (Scheduled Trigger)  
- `Get word's list` (Data Table fetch)

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Allows manual initiation for testing or on-demand runs.  
  - Inputs: None  
  - Outputs: To `Get word's list`  
  - Failures: None expected unless manual interruption.

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Runs the workflow automatically every day at 1 AM (configurable).  
  - Inputs: None  
  - Outputs: To `Get word's list`  
  - Failures: Possible if n8n scheduler is disabled or misconfigured.

- **Get word's list**  
  - Type: Data Table (Get operation)  
  - Role: Retrieves brand keywords from "Brand Monitoring Words" data table.  
  - Parameters: Reads all rows (keywords)  
  - Inputs: From triggers  
  - Outputs: To parallel AnySite search nodes  
  - Failures: Data table connection errors or empty keyword list.

---

### 2.2 Cross-Platform Search Phase

**Overview:**  
Executes parallel API calls to AnySite.io to search for posts matching each keyword on four platforms.

**Nodes Involved:**  
- `AnySite Search Reddit Posts`  
- `AnySite Search LinkedIn Posts`  
- `AnySite Search Instagram Posts`  
- `AnySite Search X Posts`

**Node Details:**

- **AnySite Search Reddit Posts**  
  - Type: HTTP Request (POST)  
  - Role: Search Reddit posts via AnySite API with query from keyword.  
  - Parameters: JSON body with `query` = keyword, `count`=2, `sort`=relevance, timeout set.  
  - Headers: `access-token` for API authentication.  
  - Outputs: To `If Reddit post does not exist`  
  - Failures: API auth errors, rate limits, network timeouts.

- **AnySite Search LinkedIn Posts**  
  - Type: HTTP Request (POST)  
  - Role: Search LinkedIn posts matching keyword, limited to 2 posts.  
  - Parameters: `keywords` = keyword, `count`=2.  
  - Headers: `access-token`.  
  - Outputs: To `If LinkedIn post does not exist1`  
  - Failures: Auth errors, API limits, LinkedIn API changes.

- **AnySite Search Instagram Posts**  
  - Type: HTTP Request (POST)  
  - Role: Search Instagram posts via Twitter API endpoint (likely due to AnySite API design).  
  - Parameters: `query`=keyword, `count`=2, timeout.  
  - Headers: `access-token`.  
  - Outputs: To `If Instagram post does not exist2`  
  - Failures: Auth, rate limit, endpoint changes.

- **AnySite Search X Posts**  
  - Type: HTTP Request (POST)  
  - Role: Search X (Twitter) posts by keyword, count=2, timeout set.  
  - Headers: `access-token`.  
  - Outputs: To `If X post does not exist3`  
  - Failures: Same as above.

---

### 2.3 Deduplication Check & Data Insertion

**Overview:**  
Checks data table to avoid duplicates using `post_id`. Inserts new posts into the "Brand Monitoring Posts" data table.

**Nodes Involved:**  
- `If Reddit post does not exist`  
- `If LinkedIn post does not exist1`  
- `If Instagram post does not exist2`  
- `If X post does not exist3`  
- `Insert Reddit post`  
- `Insert LinkedIn post`  
- `Insert Instagram post`  
- `Insert X post`

**Node Details:**

- **If * post does not exist** (for each platform)  
  - Type: Data Table (rowNotExists)  
  - Role: Checks if `post_id` from API results exists in posts data table.  
  - Inputs: From respective AnySite search nodes.  
  - Outputs: If not exists → Insert post node; else no output (skip).  
  - Failures: Data table connectivity, malformed post_id.

- **Insert * post**  
  - Type: Data Table (Insert)  
  - Role: Inserts new post records with relevant metadata fields including type, url, text, counts, timestamps, and the keyword that triggered it.  
  - Inputs: From above checks.  
  - Outputs: For Reddit, leads to comment fetching; for others, merges to next stage.  
  - Failures: Data table write errors, missing mandatory fields.

---

### 2.4 Platform-Specific Data Extraction

**Overview:**  
For newly inserted posts, fetches detailed comments and post content to enrich stored data.

**Nodes Involved:**  
- `AnySite Get Reddit Post Comment`  
- `AnySite Get Reddit Post`  
- `AnySite Get LinkedIn Post Comments`

**Node Details:**

- **AnySite Get Reddit Post Comment**  
  - Type: HTTP Request (POST)  
  - Role: Fetches Reddit comments for a post URL with timeout 300 seconds.  
  - Inputs: From `Insert Reddit post`  
  - Outputs: To `Convert comments to the string`  
  - Failures: API rate limits, comment data missing.

- **AnySite Get Reddit Post**  
  - Type: HTTP Request (POST)  
  - Role: Fetches full Reddit post details by URL.  
  - Inputs: After comments fetch  
  - Outputs: To `Update Reddit post details`  
  - Failures: API errors, data inconsistencies.

- **AnySite Get LinkedIn Post Comments**  
  - Type: HTTP Request (POST)  
  - Role: Retrieves comments for LinkedIn posts by urn value.  
  - Inputs: From `Insert LinkedIn post`  
  - Outputs: To `Convert LN comments to the string`  
  - Failures: API auth, data structure changes.

---

### 2.5 Comment Processing and Post Updates

**Overview:**  
Processes raw comment data into structured JSON strings and updates post records in the database.

**Nodes Involved:**  
- `Convert comments to the string` (Reddit)  
- `Update Reddit post comments`  
- `Update Reddit post details`  
- `Convert LN comments to the string` (LinkedIn)  
- `Update LN comments`  

**Node Details:**

- **Convert comments to the string**  
  - Type: Code Node (JavaScript)  
  - Role: Parses raw comment JSON, filters deleted comments, normalizes timestamps, transforms nested comments, and serializes to JSON string.  
  - Inputs: Raw comment data from Reddit API.  
  - Outputs: Structured JSON string for comments.  
  - Failures: JSON parsing errors, unexpected data formats.

- **Update Reddit post comments**  
  - Type: Data Table (Update)  
  - Role: Updates comments field of Reddit post record with serialized comments string.  
  - Inputs: From `Convert comments to the string`  
  - Outputs: None  
  - Failures: DB update errors.

- **Update Reddit post details**  
  - Type: Data Table (Update)  
  - Role: Updates Reddit post text field with enriched data from AnySite API.  
  - Inputs: From `AnySite Get Reddit Post`  
  - Outputs: To `Merge` node to consolidate all platform results.  
  - Failures: DB update errors.

- **Convert LN comments to the string**  
  - Type: Code Node (JavaScript)  
  - Role: Parses LinkedIn comments, normalizes and structures them, extracts post IDs, serializes to JSON.  
  - Inputs: Raw LinkedIn comments.  
  - Outputs: Structured JSON string for LinkedIn comments.  
  - Failures: Parsing failures, missing fields.

- **Update LN comments**  
  - Type: Data Table (Update)  
  - Role: Updates LinkedIn post record with structured comments JSON string.  
  - Inputs: From LinkedIn comment conversion node.  
  - Outputs: None  
  - Failures: Data table update issues.

---

### 2.6 AI Analysis & Reporting

**Overview:**  
Aggregates all new posts, normalizes and analyzes data with GPT-4 powered AI Agent, generates a detailed report including sentiment, engagement, trends, and recommended actions.

**Nodes Involved:**  
- `Get post_id for all new posts` (Code node to extract unique IDs)  
- `AI Agent` (LangChain Agent with GPT-4o)  
- `Get Posts Info` (Data Table get node)  
- `OpenAI Chat Model` (GPT-4o model node)

**Node Details:**

- **Get post_id for all new posts**  
  - Type: Code Node (JavaScript)  
  - Role: Extracts and deduplicates all post IDs from merged platform results to feed AI Agent.  
  - Outputs: JSON with list of unique post IDs and count.  
  - Failures: JSON parsing or empty post list.

- **AI Agent**  
  - Type: LangChain Agent Node  
  - Role: Core AI logic performing:  
    - Fetch posts by IDs using internal tool `Get Posts Info`  
    - Normalize data (timestamps, comments, deduplication, platform inference)  
    - Analyze sentiment, engagement, risks, drivers  
    - Compose rich HTML & plain-text report structured per specification  
    - Send email via Gmail tool  
  - Parameters: Complex prompt with detailed instructions for social media intelligence analysis, keyword disambiguation, data normalization, and reporting format.  
  - Inputs: Post IDs and optional post objects  
  - Outputs: Email send status and JSON acknowledgment.  
  - Failures: OpenAI API quota limits, invalid inputs, email send failures.

- **Get Posts Info**  
  - Type: Data Table (Get)  
  - Role: Given post IDs, retrieves full post records from data table for AI analysis.  
  - Inputs: Invoked internally by AI Agent tool.  
  - Failures: DB access issues.

- **OpenAI Chat Model**  
  - Type: Language Model Node (GPT-4o)  
  - Role: Provides the underlying GPT-4 model for AI Agent analysis.  
  - Credentials: OpenAI API key (GPT-4o recommended).  
  - Failures: API key invalid, rate limits.

---

### 2.7 Email Notification

**Overview:**  
Sends the AI-generated social media monitoring report by email via Gmail.

**Nodes Involved:**  
- `Send a message in Gmail`

**Node Details:**

- **Send a message in Gmail**  
  - Type: Gmail Tool Node  
  - Role: Sends email to configured recipient with subject and formatted message body produced by AI Agent.  
  - Inputs: From AI Agent node (via ai_tool output).  
  - Credentials: Gmail OAuth2 credentials configured.  
  - Failures: OAuth token expiration, email delivery errors.

---

### 2.8 Scheduling & Manual Execution

**Overview:**  
Supports both manual and scheduled workflow execution with triggers.

**Nodes Involved:**  
- `Schedule Trigger`  
- `When clicking ‘Execute workflow’`

**Node Details:**

- Covered in section 2.1.

---

## 3. Summary Table

| Node Name                        | Node Type                    | Functional Role                          | Input Node(s)                          | Output Node(s)                          | Sticky Note                                                                                                                                                   |
|---------------------------------|------------------------------|----------------------------------------|--------------------------------------|----------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger               | Manual start of workflow                | None                                 | Get word's list                       | Triggers & Setup: Manual trigger for on-demand runs                                                                                                           |
| Schedule Trigger                | Schedule Trigger             | Scheduled start at set time             | None                                 | Get word's list                       | Triggers & Setup: Automated daily/hourly run                                                                                                                 |
| Get word's list                | Data Table (Get)             | Fetch keywords to monitor               | Schedule Trigger, Manual Trigger     | AnySite Search Reddit, LinkedIn, Instagram, X | Triggers & Setup: Define brand keywords                                                                                                                       |
| AnySite Search Reddit Posts     | HTTP Request (POST)          | Search Reddit posts for keywords        | Get word's list                      | If Reddit post does not exist          | Cross-Platform Search Phase: Reddit search                                                                                                                   |
| AnySite Search LinkedIn Posts   | HTTP Request (POST)          | Search LinkedIn posts                   | Get word's list                      | If LinkedIn post does not exist1       | Cross-Platform Search Phase: LinkedIn search                                                                                                                 |
| AnySite Search Instagram Posts  | HTTP Request (POST)          | Search Instagram posts                  | Get word's list                      | If Instagram post does not exist2      | Cross-Platform Search Phase: Instagram search                                                                                                                |
| AnySite Search X Posts          | HTTP Request (POST)          | Search X/Twitter posts                  | Get word's list                      | If X post does not exist3               | Cross-Platform Search Phase: X search                                                                                                                        |
| If Reddit post does not exist   | Data Table (rowNotExists)    | Check Reddit post duplication           | AnySite Search Reddit Posts          | Insert Reddit post                     | Deduplication Check & Data Insertion                                                                                                                        |
| If LinkedIn post does not exist1 | Data Table (rowNotExists)    | Check LinkedIn post duplication         | AnySite Search LinkedIn Posts        | Insert LinkedIn post                   | Deduplication Check & Data Insertion                                                                                                                        |
| If Instagram post does not exist2 | Data Table (rowNotExists)    | Check Instagram post duplication        | AnySite Search Instagram Posts       | Insert Instagram post                  | Deduplication Check & Data Insertion                                                                                                                        |
| If X post does not exist3       | Data Table (rowNotExists)    | Check X post duplication                | AnySite Search X Posts               | Insert X post                         | Deduplication Check & Data Insertion                                                                                                                        |
| Insert Reddit post              | Data Table (Insert)          | Insert new Reddit post                   | If Reddit post does not exist         | AnySite Get Reddit Post Comment         | Deduplication Check & Data Insertion                                                                                                                        |
| AnySite Get Reddit Post Comment | HTTP Request (POST)          | Fetch Reddit comments                   | Insert Reddit post                   | Convert comments to the string          | Platform-Specific Data Extraction                                                                                                                           |
| Convert comments to the string  | Code Node (JS)               | Process Reddit comments JSON             | AnySite Get Reddit Post Comment      | Update Reddit post comments             | Comment Processing and Post Updates                                                                                                                        |
| Update Reddit post comments     | Data Table (Update)          | Update Reddit post with comments        | Convert comments to the string        | None                                  | Comment Processing and Post Updates                                                                                                                        |
| AnySite Get Reddit Post         | HTTP Request (POST)          | Fetch Reddit post details               | AnySite Get Reddit Post Comment       | Update Reddit post details              | Platform-Specific Data Extraction                                                                                                                           |
| Update Reddit post details      | Data Table (Update)          | Update Reddit post with full details    | AnySite Get Reddit Post              | Merge                                 | Comment Processing and Post Updates                                                                                                                        |
| Insert LinkedIn post            | Data Table (Insert)          | Insert new LinkedIn post                 | If LinkedIn post does not exist1      | AnySite Get LinkedIn Post Comments       | Deduplication Check & Data Insertion                                                                                                                        |
| AnySite Get LinkedIn Post Comments | HTTP Request (POST)          | Fetch LinkedIn comments                 | Insert LinkedIn post                 | Convert LN comments to the string       | Platform-Specific Data Extraction                                                                                                                           |
| Convert LN comments to the string | Code Node (JS)               | Process LinkedIn comments JSON           | AnySite Get LinkedIn Post Comments    | Update LN comments                      | Comment Processing and Post Updates                                                                                                                        |
| Update LN comments              | Data Table (Update)          | Update LinkedIn post with comments       | Convert LN comments to the string      | None                                  | Comment Processing and Post Updates                                                                                                                        |
| Insert Instagram post           | Data Table (Insert)          | Insert new Instagram post                | If Instagram post does not exist2     | Merge                                 | Deduplication Check & Data Insertion                                                                                                                        |
| Insert X post                  | Data Table (Insert)          | Insert new X post                        | If X post does not exist3             | Merge                                 | Deduplication Check & Data Insertion                                                                                                                        |
| Merge                         | Merge Node                   | Consolidates posts from all platforms  | Update Reddit post details, Insert LinkedIn post, Insert Instagram post, Insert X post | Get post_id for all new posts          | Consolidates all new posts before AI analysis                                                                                                              |
| Get post_id for all new posts  | Code Node (JS)               | Extract unique post IDs for AI Agent    | Merge                               | AI Agent                             | AI Analysis & Reporting: Post ID extraction                                                                                                                 |
| AI Agent                      | LangChain AI Agent           | Analyze posts, generate report, send email | Get post_id for all new posts         | Send a message in Gmail (ai_tool)        | AI Analysis & Reporting: Core AI-powered analysis and report generation                                                                                      |
| Get Posts Info                | Data Table (Get)             | Retrieve post details for AI analysis    | AI Agent (tool call)                 | AI Agent (language model)               | AI Analysis & Reporting: Data retrieval for analysis                                                                                                       |
| OpenAI Chat Model             | Language Model (GPT-4o)      | GPT-4o model for AI Agent                 | AI Agent (languageModel)             | AI Agent                               | AI Analysis & Reporting: Language model powering AI Agent                                                                                                  |
| Send a message in Gmail       | Gmail Tool                   | Send email report                         | AI Agent (ai_tool output)            | None                                  | Email Notification: Sends AI report via Gmail                                                                                                              |
| Convert LN comments to the string | Code Node (JS)               | Process LinkedIn comments JSON           | AnySite Get LinkedIn Post Comments    | Update LN comments                      | Comment Processing and Post Updates                                                                                                                        |
| Update LN comments              | Data Table (Update)          | Update LinkedIn post with comments       | Convert LN comments to the string      | None                                  | Comment Processing and Post Updates                                                                                                                        |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes**  
   - Add a **Schedule Trigger** node configured to run daily at 1 AM (or desired frequency).  
   - Add a **Manual Trigger** node named "When clicking ‘Execute workflow’" for manual runs.

2. **Add Keyword Input Node**  
   - Add a **Data Table** node named "Get word's list" to fetch monitoring keywords from a data table "Brand Monitoring Words" (ensure this table exists and contains the field `word`).

3. **Add Parallel AnySite Search Nodes**  
   For each platform, create HTTP Request POST nodes with the following configurations:  
   - **AnySite Search Reddit Posts**  
     URL: `https://api.anysite.io/api/reddit/search/posts`  
     Body: JSON with `query` from current keyword, count=2, sort by relevance, timeout=300  
     Headers: add `access-token` with your AnySite API token  
   - **AnySite Search LinkedIn Posts**  
     URL: `https://api.anysite.io/api/linkedin/search/posts`  
     Body: JSON with `keywords` from current keyword, count=2  
     Headers: same as above  
   - **AnySite Search Instagram Posts**  
     URL: `https://api.anysite.io/api/twitter/search/posts` (Instagram posts endpoint)  
     Body: JSON with `query` from keyword, count=2, timeout=300  
     Headers: same as above  
   - **AnySite Search X Posts**  
     URL: `https://api.anysite.io/api/instagram/search/posts` (X posts endpoint)  
     Body: JSON with `query` from keyword, count=2, timeout=300  
     Headers: same as above  

4. **Add Deduplication Check Nodes**  
   For each platform, add a Data Table node with **rowNotExists** operation on "Brand Monitoring Posts" table to check if `post_id` exists:  
   - Input field to match depends on platform (`id` or `urn.value` etc.)  
   - If post does not exist, connect output to corresponding Insert Post node.

5. **Add Insert Post Nodes**  
   Create Data Table Insert nodes for each platform inserting into "Brand Monitoring Posts" table with mapping fields:  
   - Store platform type, url, post_id, created_at, votes/likes, comment counts, text, and keyword.  
   - For Reddit, after insert, connect to `AnySite Get Reddit Post Comment` node.  
   - For LinkedIn, after insert, connect to `AnySite Get LinkedIn Post Comments` node.  
   - For Instagram and X, after insert, connect to a Merge node.

6. **Add Comment & Post Details Fetching Nodes**  
   - **AnySite Get Reddit Post Comment**: POST request with `post_url` = Reddit post URL; outputs to comment conversion node.  
   - **AnySite Get Reddit Post**: POST request with `post_url` = Reddit post URL; outputs to update Reddit post details node.  
   - **AnySite Get LinkedIn Post Comments**: POST request with `urn` formatted as `urn:li:activity:post_id`; outputs to LinkedIn comment conversion node.

7. **Add Comment Conversion Code Nodes**  
   - For Reddit: JavaScript node parses comments, handles deleted comments, timestamps, nests replies, serializes JSON string.  
   - For LinkedIn: Similar code node parses comments, extracts post IDs, normalizes dates, serializes JSON.

8. **Add Update Post Comments Nodes**  
   - Data Table Update nodes for Reddit and LinkedIn posts update the `comments` field with JSON string from conversion nodes.

9. **Add Update Reddit Post Details Node**  
   - Data Table Update node updates Reddit post `text` with enriched data fetched.

10. **Add Merge Node**  
    - Merge outputs of all platform insertions and Reddit post updates into a single stream for AI analysis.

11. **Add Post ID Extraction Code Node**  
    - JavaScript node parses merged posts, extracts unique `post_id`s for AI processing.

12. **Configure AI Agent Node**  
    - Add LangChain AI Agent node with tools:  
      - Data Table Get node "Get Posts Info" to fetch full post details for given post IDs.  
      - OpenAI Chat Model node configured with GPT-4o credentials.  
    - Set AI Agent prompt with detailed instructions for data normalization, sentiment analysis, report generation, and email sending (as per workflow description).

13. **Add Data Table Get Node "Get Posts Info"**  
    - Fetch post details by `post_id` from "Brand Monitoring Posts" table.

14. **Add OpenAI Chat Model Node**  
    - Configure with OpenAI API key, use `gpt-4o` model type.

15. **Add Gmail Send Node**  
    - Set up Gmail OAuth2 credentials.  
    - Configure recipient email address (e.g. brand monitoring team).  
    - Connect AI Agent's email sending output to this node.

16. **Credentials Setup**  
    - AnySite API token: Add to all AnySite HTTP Request nodes in their header parameters.  
    - OpenAI API Key: Add to OpenAI Chat Model node.  
    - Gmail OAuth2: Add to Gmail send node.

17. **Create Data Tables**  
    - "Brand Monitoring Words" with at least one `word` field (string).  
    - "Brand Monitoring Posts" with fields: `type`, `title`, `url`, `created_at`, `subreddit_id`, `subreddit_alias`, `subreddit_url`, `subreddit_description`, `comment_count`, `vote_count`, `subreddit_member_count`, `post_id`, `text`, `comments`, `word`.

18. **Test Workflow**  
    - Run manual trigger.  
    - Verify search results appear from all platforms.  
    - Confirm no duplicates inserted.  
    - Check enrichment and comment processing.  
    - Confirm AI Agent generates report and sends email.

---

## 5. General Notes & Resources

| Note Content                                                                                                             | Context or Link                                  |
|--------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| Social media monitoring workflow designed for marketing, PR, and customer success teams using AnySite.io and OpenAI GPT. | Overview of workflow purpose and target users.  |
| Setup requires valid AnySite API tokens, OpenAI GPT-4o API key, PostgreSQL data tables, and Gmail OAuth2 credentials.      | Setup requirements for API and data sources.    |
| Workflow uses AnySite.io APIs to query Reddit, LinkedIn, Instagram, and X (Twitter) in parallel with deduplication.       | Cross-platform search and data extraction.      |
| AI Agent uses detailed prompt instructions to normalize data, analyze sentiment, detect risks, and generate email reports.| AI-driven social media intelligence analysis.   |
| Email notifications sent via Gmail with rich HTML report formatted inline for maximum compatibility.                     | Notification mechanism for monitoring alerts.   |
| Refer to AnySite API docs: https://docs.anysite.io and n8n AI Agent docs: https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent/ | API and AI integration documentation.            |
| Customization guides included for adding platforms, modifying alerts, and adjusting schedule frequency.                   | Workflow extension and adaptation instructions. |
| Testing guide recommends starting with manual trigger and verifying each stage including database and email.              | Best practices for workflow validation.          |
| PostgreSQL schema details provided for keywords and posts data tables.                                                    | Database schema for data storage.                |
| Inline CSS used in email HTML for compatibility; no external assets.                                                      | Email formatting best practices.                 |
| No external web browsing or data fabrication; all analysis based strictly on fetched data.                                | Compliance and data integrity notes.             |
| Workflow version 1.0 created by Andrew Kulikov at AnySite.io, last updated October 2025.                                  | Project credits and versioning.                   |

---

This document provides a complete, detailed reference to the Cross-Platform Brand Monitoring & Analysis workflow, enabling reproduction, modification, and error anticipation for both human operators and automation agents.

---