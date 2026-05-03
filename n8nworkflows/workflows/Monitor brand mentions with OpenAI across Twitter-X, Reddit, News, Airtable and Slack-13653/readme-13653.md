Monitor brand mentions with OpenAI across Twitter/X, Reddit, News, Airtable and Slack

https://n8nworkflows.xyz/workflows/monitor-brand-mentions-with-openai-across-twitter-x--reddit--news--airtable-and-slack-13653


# Monitor brand mentions with OpenAI across Twitter/X, Reddit, News, Airtable and Slack

### 1. Workflow Overview

The **AI Social Listening Agent** is an automated system designed to monitor brand mentions across multiple digital channels (Twitter/X, Reddit, and News) in real-time. It leverages Artificial Intelligence (OpenAI) to provide qualitative analysis, classifying sentiment and urgency while detecting trending topics.

The workflow is structured into four primary functional phases:
1.  **Trigger & Configuration:** Initializes the monitoring session with specific brand parameters.
2.  **Data Acquisition & Normalization:** Fetches raw data from three distinct APIs and transforms it into a unified schema.
3.  **AI Intelligence & Deduplication:** Processes text via GPT-4o-mini and ensures only new, unique mentions are tracked using n8n’s static data.
4.  **Actionable Output:** Logs all data to Airtable, triggers instant Slack alerts for critical issues, and generates an HTML-formatted daily digest for email distribution.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Configuration
This block defines when the workflow runs and the specific brand terms it should search for.
*   **Nodes Involved:** 
    *   `Monitor brand mentions every hour` (Schedule Trigger)
    *   `Set brand monitoring config` (Set)
*   **Node Details:**
    *   **Schedule Trigger:** Set to execute at an hourly interval.
    *   **Set Node:** Defines string variables for `brandName`, `keywords` (comma-separated), and numerical thresholds for `lookbackHours` and `negativeAlertThreshold`.
    *   **Failure Type:** Configuration errors (e.g., empty brand name) will halt downstream API requests.

#### 2.2 Fetch & Collect Mentions
Pulls data in parallel from various social and news platforms.
*   **Nodes Involved:** 
    *   `Fetch Twitter/X mentions` (HTTP Request)
    *   `Fetch Reddit mentions` (HTTP Request)
    *   `Fetch news article mentions` (HTTP Request)
    *   `Merge all platform mentions` (Merge)
    *   `Normalize mentions into unified schema` (Code)
*   **Node Details:**
    *   **HTTP Requests:** Standard GET requests to Twitter API v2, Reddit Search JSON, and NewsAPI. 
    *   **Merge Node:** Combines the disparate outputs from the three search streams.
    *   **Code Node:** This is critical; it iterates through the mixed JSON results from Twitter, Reddit, and News, mapping them into a consistent object structure (Platform, Text, Author, URL, Date, Reach).
    *   **Failure Type:** Authentication errors (expired API tokens) or Rate Limiting (429 errors).

#### 2.3 AI Analysis & Deduplication
Utilizes AI to add context to raw text and manages the "memory" of the workflow.
*   **Nodes Involved:** 
    *   `AI sentiment and urgency analysis` (HTTP Request)
    *   `Wait For Result` (Wait)
    *   `Process AI analysis results` (Code)
*   **Node Details:**
    *   **OpenAI HTTP Request:** Sends a structured JSON prompt to `gpt-4o-mini`. It requests sentiment (-1.0 to 1.0), urgency (low to critical), a one-sentence summary, and key topics.
    *   **Process Code Node:** 
        *   **Deduplication:** Uses `$getWorkflowStaticData('global')` to store `mentionId`. If a mention was seen in a previous run, it is marked `isDuplicate`.
        *   **State Management:** Accumulates session statistics (total mentions, positive/negative counts) in static memory.
    *   **Failure Type:** OpenAI API timeouts or JSON parsing errors if the AI returns malformed text.

#### 2.4 Store, Alert & Report
The final phase handles data persistence and communication.
*   **Nodes Involved:** 
    *   `Route by sentiment and urgency` (Switch)
    *   `Log mention to Airtable` (HTTP Request)
    *   `Send critical mention alert` (HTTP Request)
    *   `Generate HTML daily digest report` (Code)
    *   `Filter mentions requiring alerts` (Filter)
    *   `Post mention alert to Slack` (HTTP Request)
    *   `Email HTML digest to marketing team` (Email Send)
    *   `Log success and update listening statistics` (Code)
*   **Node Details:**
    *   **Switch Node:** Routes items into three streams: Critical, Negative, or Neutral/Positive.
    *   **Code (HTML Report):** Generates a full CSS-inlined HTML document summarizing the session, including sentiment charts (simulated via tables) and latest mention cards.
    *   **Email Send:** Uses SMTP credentials to send the generated HTML report.
    *   **Failure Type:** SMTP connection failures or Airtable field mapping errors (e.g., text exceeding 500 characters being truncated).

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Monitor brand mentions every hour | Schedule Trigger | Workflow initiation | (None) | Set brand monitoring config | 1. Trigger & configure |
| Set brand monitoring config | Set | Variable definition | Monitor brand mentions... | Fetch Twitter/Reddit/News | 1. Trigger & configure |
| Fetch Twitter/X mentions | HTTP Request | Data fetching | Set brand monitoring... | Merge all platform... | 2. Fetch & collect mentions |
| Fetch Reddit mentions | HTTP Request | Data fetching | Set brand monitoring... | Merge all platform... | 2. Fetch & collect mentions |
| Fetch news article mentions | HTTP Request | Data fetching | Set brand monitoring... | Merge all platform... | 2. Fetch & collect mentions |
| Merge all platform mentions | Merge | Data consolidation | Fetch Twitter/Reddit/News | Normalize mentions... | 2. Fetch & collect mentions |
| Normalize mentions into unified schema | Code | Data transformation | Merge all platform... | AI sentiment and urgency... | 2. Fetch & collect mentions |
| AI sentiment and urgency analysis | HTTP Request | AI Processing | Normalize mentions... | Wait For Result | 3. AI analyze & deduplicate |
| Wait For Result | Wait | Flow control | AI sentiment... | Process AI analysis... | 3. AI analyze & deduplicate |
| Process AI analysis results | Code | Deduplication/Stats | Wait For Result | Route by sentiment... | 3. AI analyze & deduplicate |
| Route by sentiment and urgency | Switch | Conditional routing | Process AI analysis... | Send critical alert/Airtable/Report | 4. Store, alert & report |
| Send critical mention alert | HTTP Request | Direct Alert | Route (Critical) | Generate HTML report | 4. Store, alert & report |
| Log mention to Airtable | HTTP Request | Database logging | Route (All) | Generate HTML report | 4. Store, alert & report |
| Generate HTML daily digest report | Code | Report formatting | Log Airtable/Send alert | Filter mentions... | 4. Store, alert & report |
| Filter mentions requiring alerts | Filter | Conditional logic | Generate HTML report | Slack / Email | 4. Store, alert & report |
| Post mention alert to Slack | HTTP Request | Notification | Filter mentions... | Log success... | 4. Store, alert & report |
| Email HTML digest to marketing team | Email Send | Distribution | Filter mentions... | Log success... | 4. Store, alert & report |
| Log success and update stats | Code | Final Logging | Slack / Email | (End) | 4. Store, alert & report |

---

### 4. Reproducing the Workflow from Scratch

1.  **Initiate Trigger:** Add a `Schedule Trigger` node set to 1-hour intervals.
2.  **Configuration:** Add a `Set` node. Create string values: `brandName` (e.g., "Apple"), `keywords`, and numerical values `lookbackHours` (1).
3.  **API Search Nodes:** 
    *   Create three `HTTP Request` nodes. 
    *   **Twitter:** GET `https://api.twitter.com/2/tweets/search/recent` with Query `{{ $json.brandName + ' -is:retweet lang:en' }}`. Requires Bearer Token.
    *   **Reddit:** GET `https://www.reddit.com/search.json` with Query `q: {{ $json.brandName }}` and `sort: new`.
    *   **News:** GET `https://newsapi.org/v2/everything` with Query `q: {{ $json.brandName }}`. Requires NewsAPI Key.
4.  **Data Processing:** 
    *   Connect all three to a `Merge` node (Combine mode).
    *   Add a `Code` node. Use JavaScript to map each platform's unique JSON paths (e.g., `tweet.text` vs `post.data.title`) into a single array of objects with keys: `platform`, `text`, `url`, `mentionId`.
5.  **AI Integration:**
    *   Add an `HTTP Request` node for OpenAI (POST `https://api.openai.com/v1/chat/completions`). 
    *   Set body to `json_object` and provide a System Prompt requesting JSON output with `sentiment`, `urgency`, `summary`, and `topics`.
    *   Add a `Wait` node set to a few seconds to ensure AI completion.
6.  **Intelligence & Deduplication:**
    *   Add a `Code` node. Use `$getWorkflowStaticData('global')` to check if `mentionId` exists. If not, save it. Calculate session-wide counters (Positive%, Negative%).
7.  **Logic & Storage:**
    *   Add a `Switch` node to route based on `urgency == 'critical'` or `sentiment == 'negative'`.
    *   Add an `HTTP Request` node for Airtable (POST to your Base/Table). Map fields to the normalized schema.
8.  **Reporting & Alerts:**
    *   Add a `Code` node to wrap data in an HTML template (using backticks for multiline strings).
    *   Add a `Filter` node to check if `shouldAlert == true`.
    *   Add an `HTTP Request` for Slack Webhook and an `Email Send` node for the HTML report.
9.  **Credentials:** Set up 5 credentials: Twitter Bearer, NewsAPI, OpenAI API, Airtable API, and SMTP.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Data Retention** | Deduplication logic automatically clears mention IDs older than 7 days to manage memory. |
| **Twitter API** | Requires a Developer Account and a Bearer Token from the Twitter Portal. |
| **Airtable Setup** | Ensure the target table has fields: Mention ID, Platform, Text, Sentiment, Urgency, URL. |
| **Scalability** | For high-volume brands, consider using the "Split In Batches" node before the AI analysis to avoid API rate limits. |