Daily News Curation with Gemini AI: Auto-filter & Publish to Notion & Telegram

https://n8nworkflows.xyz/workflows/daily-news-curation-with-gemini-ai--auto-filter---publish-to-notion---telegram-11834


# Daily News Curation with Gemini AI: Auto-filter & Publish to Notion & Telegram

### 1. Workflow Overview

This workflow automates daily news curation using Gemini AI, aggregating news from multiple sources, filtering and normalizing the content, applying AI-driven editorial processing, and then publishing curated results to Notion and Telegram. It targets users who want an automated, intelligent pipeline for collecting relevant news updates, filtering by recency and keywords, enhancing content with AI, and distributing via key communication channels.

The workflow is organized into the following logical blocks:

- **1.1 Scheduled Trigger & Data Collection:** Initiates the workflow daily and fetches news data from multiple RSS and HTTP sources.
- **1.2 Data Merging and Filtering:** Merges and normalizes incoming news items, filters them by recency (last 24 hours) and keywords to reduce noise.
- **1.3 AI Editorial Processing:** Uses Gemini AI (Google Gemini via Langchain node) to analyze and enhance the filtered news batch.
- **1.4 Output Processing and Publishing:** Processes AI output for Notion compatibility, adds the curated content to Notion, and sends notifications via Telegram.
- **1.5 Conditional Publishing Control:** Decides whether to publish based on classification results, otherwise stops the workflow.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger & Data Collection

- **Overview:** This block triggers the workflow daily and collects news items from multiple sources including RSS feeds and HTTP requests.
- **Nodes Involved:**  
  - Daily Schedule  
  - n8n Community Announcements (RSS)  
  - GitHub n8n Releases (HTTP Request)  
  - Reddit n8n News (RSS)  
  - n8n Blog RSS (RSS)  

- **Node Details:**

  - **Daily Schedule**  
    - Type: Schedule Trigger  
    - Role: Starts the workflow on a daily schedule without additional parameters.  
    - Connections: Outputs to all news source nodes in parallel.  
    - Failure Types: Scheduler downtime, misconfiguration.

  - **n8n Community Announcements**  
    - Type: RSS Feed Read  
    - Role: Fetches community announcements RSS feed.  
    - Config: Standard RSS URL (not explicitly shown).  
    - Connections: Output to "Merge All Sources" node (index 1).  
    - Failure Types: Network errors, feed unavailability.

  - **GitHub n8n Releases**  
    - Type: HTTP Request  
    - Role: Fetches GitHub releases via HTTP API.  
    - Config: HTTP GET request to GitHub API (URL implicit).  
    - Connections: Output to "Merge All Sources" node (index 2).  
    - Failure Types: API rate limiting, auth errors, network issues.

  - **Reddit n8n News**  
    - Type: RSS Feed Read  
    - Role: Reads Reddit RSS feed for n8n news.  
    - Connections: Output to "Merge All Sources" node (index 3).  
    - Failure Types: Feed changes, network failures.

  - **n8n Blog RSS**  
    - Type: RSS Feed Read  
    - Role: Fetches official n8n blog posts.  
    - Connections: Output to "Merge All Sources" node (index 0).  
    - Failure Types: Same as other RSS nodes.

---

#### 2.2 Data Merging and Filtering

- **Overview:** Merges all source data into a single stream, normalizes and filters items from the last 24 hours, then applies keyword pre-filtering to focus on relevant items.
- **Nodes Involved:**  
  - Merge All Sources  
  - Normalize & Filter (24h) (Code Node)  
  - Keyword Pre-filter (Code Node)  
  - Batch Items (Aggregate Node)  

- **Node Details:**

  - **Merge All Sources**  
    - Type: Merge  
    - Role: Combines multiple incoming data streams into one unified output for processing.  
    - Connections: Inputs from all data sources; output to "Normalize & Filter (24h)".  
    - Failure Types: Data inconsistency, incompatible formats.

  - **Normalize & Filter (24h)**  
    - Type: Code  
    - Role: Custom JavaScript code to normalize data structure and filter items published within the last 24 hours only.  
    - Inputs: Merged feed items.  
    - Outputs: Filtered recent news items.  
    - Failure Types: Code exceptions, date parsing errors.

  - **Keyword Pre-filter**  
    - Type: Code  
    - Role: Further filters items based on presence of specific keywords to refine content relevance.  
    - Inputs: Recent filtered items.  
    - Outputs: Keyword-matched news items.  
    - Failure Types: Coding errors, missing keyword list.

  - **Batch Items**  
    - Type: Aggregate  
    - Role: Aggregates filtered news items into batches for AI processing.  
    - Inputs: Keyword-filtered items.  
    - Outputs: Batches sent to AI node.  
    - Failure Types: Large batch sizes causing memory issues.

---

#### 2.3 AI Editorial Processing

- **Overview:** Passes the curated batch of news items to Gemini AI via Langchain to act as an "Editor-in-Chief," analyzing and summarizing content.
- **Nodes Involved:**  
  - Editor-in-Chief (AI Agent)  

- **Node Details:**

  - **Editor-in-Chief (AI Agent)**  
    - Type: @n8n/n8n-nodes-langchain.googleGemini  
    - Role: Uses Google Gemini AI model to analyze and enhance news batches.  
    - Retry Logic: Retries 2 times with 5 seconds wait in case of failure.  
    - Inputs: Batched news items from "Batch Items".  
    - Outputs: AI-generated editorial content.  
    - Failure Types: API quota limits, network errors, malformed input.  
    - Version: Requires n8n version supporting Langchain Google Gemini integration.  
    - Credentials: Configured with Google Gemini API credentials.

---

#### 2.4 Output Processing and Publishing

- **Overview:** Parses AI output to comply with Notion's content constraints, then publishes curated news to Notion and sends a Telegram notification.
- **Nodes Involved:**  
  - Parse & Chunk Output (Notion Guardrail) (Code)  
  - Random Wait (Wait)  
  - Add to Notion (Content DB) (Notion)  
  - Telegram Notification (Telegram)  

- **Node Details:**

  - **Parse & Chunk Output (Notion Guardrail)**  
    - Type: Code  
    - Role: Processes AI output, splits or formats content to fit Notion's block size limits or guardrails.  
    - Inputs: AI editorial output.  
    - Outputs: Prepared content for Notion insertion.  
    - Failure Types: Formatting errors, content truncation issues.

  - **Random Wait**  
    - Type: Wait  
    - Role: Introduces a delay before publishing to avoid API rate limits or batching.  
    - Inputs: Parsed content.  
    - Outputs: To Notion node.  
    - Failure Types: Timeout misconfiguration.

  - **Add to Notion (Content DB)**  
    - Type: Notion  
    - Role: Adds the curated content as pages or database entries in Notion.  
    - Inputs: Formatted content after wait node.  
    - Outputs: Classification Gate.  
    - Credentials: Requires Notion API credentials with write access.  
    - Failure Types: API errors, permission issues.

  - **Telegram Notification**  
    - Type: Telegram  
    - Role: Sends notifications about published news to Telegram channels or users.  
    - Inputs: From Classification Gate node (on positive classification).  
    - Credentials: Telegram Bot API token required.  
    - Failure Types: Bot token invalid, chat ID errors, connectivity.

---

#### 2.5 Conditional Publishing Control

- **Overview:** Decides whether the curated content meets criteria for publishing; if not, the workflow stops.
- **Nodes Involved:**  
  - Classification Gate (If)  
  - Stop Process (NoOp)  

- **Node Details:**

  - **Classification Gate**  
    - Type: If  
    - Role: Conditional node that evaluates if content should be published or discarded based on classification output or quality metrics.  
    - Inputs: From Notion node.  
    - Outputs:  
      - True: Telegram Notification  
      - False: Stop Process  
    - Failure Types: Logic misconfigurations, missing fields.

  - **Stop Process**  
    - Type: NoOp  
    - Role: Terminates workflow gracefully when content does not qualify for publishing.  
    - Inputs: From Classification Gate (false branch).  
    - Outputs: None.

---

### 3. Summary Table

| Node Name                      | Node Type                     | Functional Role                      | Input Node(s)                           | Output Node(s)                      | Sticky Note                        |
|--------------------------------|-------------------------------|------------------------------------|---------------------------------------|-----------------------------------|----------------------------------|
| Daily Schedule                 | Schedule Trigger              | Starts daily workflow               | None                                  | n8n Community Announcements, GitHub n8n Releases, Reddit n8n News, n8n Blog RSS |                                  |
| n8n Community Announcements    | RSS Feed Read                | Fetches community announcements    | Daily Schedule                        | Merge All Sources (index 1)        |                                  |
| GitHub n8n Releases            | HTTP Request                | Fetches GitHub releases            | Daily Schedule                        | Merge All Sources (index 2)        |                                  |
| Reddit n8n News                | RSS Feed Read                | Fetches Reddit news                | Daily Schedule                        | Merge All Sources (index 3)        |                                  |
| n8n Blog RSS                  | RSS Feed Read                | Fetches official blog posts        | Daily Schedule                        | Merge All Sources (index 0)        |                                  |
| Merge All Sources              | Merge                       | Combines all news data             | n8n Blog RSS, n8n Community Announcements, GitHub n8n Releases, Reddit n8n News | Normalize & Filter (24h)           |                                  |
| Normalize & Filter (24h)       | Code                        | Normalize & filter recent news     | Merge All Sources                     | Keyword Pre-filter                 |                                  |
| Keyword Pre-filter             | Code                        | Filter news by keywords            | Normalize & Filter (24h)              | Batch Items                       |                                  |
| Batch Items                   | Aggregate                   | Batch news for AI processing       | Keyword Pre-filter                    | Editor-in-Chief (AI Agent)         |                                  |
| Editor-in-Chief (AI Agent)     | Langchain Google Gemini AI  | AI editorial processing            | Batch Items                          | Parse & Chunk Output (Notion Guardrail) |                                  |
| Parse & Chunk Output (Notion Guardrail) | Code                        | Format AI output for Notion        | Editor-in-Chief (AI Agent)            | Random Wait                      |                                  |
| Random Wait                   | Wait                        | Delay to avoid API rate limits     | Parse & Chunk Output                  | Add to Notion (Content DB)          |                                  |
| Add to Notion (Content DB)     | Notion                      | Publish curated content            | Random Wait                         | Classification Gate               |                                  |
| Classification Gate           | If                          | Decide if content should publish   | Add to Notion                       | Telegram Notification (true), Stop Process (false) |                                  |
| Telegram Notification         | Telegram                    | Send notification to Telegram      | Classification Gate (true branch)    | None                             |                                  |
| Stop Process                 | NoOp                        | Terminates workflow if no publish  | Classification Gate (false branch)   | None                             |                                  |
| Sticky Note, Sticky Note1, Sticky Note AI1, Sticky Note Time1, Sticky Note Time2, Sticky Note4, Sticky Note7, Sticky Note2, Sticky Note33, Sticky Note3 | Sticky Note                 | Notes and comments                 | None                                  | None                             | Invisible content, no text provided |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**
   - Name: `Daily Schedule`
   - Set to trigger once daily at the desired time.
   - No credentials needed.

2. **Create RSS Feed Read nodes for each source:**
   - `n8n Community Announcements`
     - RSS URL: community announcements feed URL.
   - `Reddit n8n News`
     - RSS URL: Reddit feed URL for n8n news.
   - `n8n Blog RSS`
     - RSS URL: official n8n blog feed.
   - Connect all these nodes to receive input from `Daily Schedule`.

3. **Create an HTTP Request node**
   - Name: `GitHub n8n Releases`
   - Set HTTP method to GET.
   - Use GitHub API URL for n8n releases.
   - Connect input from `Daily Schedule`.

4. **Create a Merge node**
   - Name: `Merge All Sources`
   - Set to merge incoming data streams from all four data source nodes.
   - Connect outputs of all RSS and HTTP request nodes to this node.

5. **Create a Code node**
   - Name: `Normalize & Filter (24h)`
   - Write JavaScript code to:
     - Normalize fields (e.g., title, link, pubDate).
     - Filter items published within the last 24 hours.
   - Connect input from `Merge All Sources`.

6. **Create another Code node**
   - Name: `Keyword Pre-filter`
   - Implement keyword-based filtering logic.
   - Provide a list or array of keywords to check in item titles or content.
   - Connect input from `Normalize & Filter (24h)`.

7. **Create an Aggregate node**
   - Name: `Batch Items`
   - Configure to batch items into groups suitable for AI processing (for example, batch size or time window).
   - Connect input from `Keyword Pre-filter`.

8. **Create a Langchain Google Gemini node**
   - Name: `Editor-in-Chief (AI Agent)`
   - Configure with Google Gemini API credentials.
   - Set retry logic: max 2 tries with 5 seconds wait.
   - Input: batched news items from `Batch Items`.
   - Output: AI-processed editorial content.

9. **Create a Code node**
   - Name: `Parse & Chunk Output (Notion Guardrail)`
   - Implement code to process AI output, chunk or format content respecting Notion API limits.
   - Connect input from `Editor-in-Chief (AI Agent)`.

10. **Create a Wait node**
    - Name: `Random Wait`
    - Add a small delay (configurable duration).
    - Connect input from `Parse & Chunk Output (Notion Guardrail)`.

11. **Create a Notion node**
    - Name: `Add to Notion (Content DB)`
    - Configure with Notion API credentials with write access.
    - Map content fields to Notion database properties.
    - Connect input from `Random Wait`.

12. **Create an If node**
    - Name: `Classification Gate`
    - Define condition(s) based on classification or quality fields from Notion response.
    - If true: proceed to Telegram Notification.
    - If false: proceed to Stop Process.

13. **Create a Telegram node**
    - Name: `Telegram Notification`
    - Configure with Telegram Bot API token.
    - Set chat ID and message content parameters.
    - Connect input from `Classification Gate` true output.

14. **Create a NoOp node**
    - Name: `Stop Process`
    - Connect input from `Classification Gate` false output.

15. **Connect all nodes as per the input/output relationships described.**

---

### 5. General Notes & Resources

| Note Content                                                                                                     | Context or Link                                          |
|-----------------------------------------------------------------------------------------------------------------|---------------------------------------------------------|
| The workflow uses Google Gemini AI via Langchain integration in n8n, requiring appropriate API credentials.     | Refer to n8n docs: https://docs.n8n.io/integrations/builtin/nodes/langchain/ |
| Telegram notifications require a Telegram bot token and valid chat ID for sending messages.                     | Telegram Bot API: https://core.telegram.org/bots/api    |
| Notion integration requires Notion API credentials and database setup to accept news content entries.           | Notion API docs: https://developers.notion.com/         |
| The workflow includes multiple code nodes for data normalization and filtering; ensure code is updated if source formats change. |                                                        |
| Rate limiting considerations addressed by Random Wait node to prevent API throttling.                           |                                                         |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.