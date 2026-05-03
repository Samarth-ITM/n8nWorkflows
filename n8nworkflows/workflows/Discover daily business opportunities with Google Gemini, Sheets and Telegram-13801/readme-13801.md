Discover daily business opportunities with Google Gemini, Sheets and Telegram

https://n8nworkflows.xyz/workflows/discover-daily-business-opportunities-with-google-gemini--sheets-and-telegram-13801


# Discover daily business opportunities with Google Gemini, Sheets and Telegram

# Reference Document: Daily Business Opportunity Digest

This n8n workflow automates the process of scouting for business opportunities. It aggregates data from multiple professional sources (Google News, Crunchbase, Product Hunt, Twitter), utilizes Google Gemini AI to filter and summarize relevant events, and then logs these opportunities into Google Sheets while alerting the user via Telegram.

---

### 1. Workflow Overview

The workflow is designed to run on a daily schedule, transforming raw social and news data into actionable business intelligence. It is organized into three main functional phases:

*   **1.1 Data Gathering:** Triggers simultaneously across multiple APIs to fetch the latest news, funding rounds, and product launches.
*   **1.2 AI Analysis & Filtering:** Normalizes the disparate data formats into a single feed, then uses an AI Agent powered by Google Gemini to identify high-value events (funding, hiring, etc.) and discard noise.
*   **1.3 Data Enrichment & Distribution:** Enriching the identified company data, logging the results in a centralized spreadsheet, and sending real-time notifications.

---

### 2. Block-by-Block Analysis

#### 2.1 Gather Info
**Overview:** This block acts as the entry point, fetching raw data from various news and social platforms.
**Nodes Involved:** `Daily Trigger`, `Google News`, `Crunchbase RSS`, `Twitter API`, `Product Hunt API`, `HTTP Request` (LinkedIn).

*   **Daily Trigger (Cron):**
    *   **Role:** Initiates the workflow once per day.
    *   **Configuration:** Set to standard daily intervals.
*   **Google News (HTTP Request):**
    *   **Role:** Fetches articles related to "startup funding", "expansion", or "hiring".
    *   **Key Parameters:** `pageSize=5`, requires NewsAPI token.
*   **Crunchbase RSS (HTTP Request):**
    *   **Role:** Pulls the latest funding round feed from Crunchbase.
*   **Twitter API (HTTP Request):**
    *   **Role:** Searches for recent tweets containing startup launch or hiring keywords.
    *   **Auth:** Header Auth required.
*   **Product Hunt API (HTTP Request):**
    *   **Role:** Retrieves the latest product posts and launches.
*   **Potential Failures:** Authentication expiration on API tokens or rate-limiting from Twitter/NewsAPI.

#### 2.2 Find Opportunities
**Overview:** This block consolidates the raw inputs and uses AI to determine if an item is a legitimate business opportunity.
**Nodes Involved:** `Merge2`, `Merge Sources`, `AI Agent`, `Google Gemini Chat Model`, `Code`.

*   **Merge Sources (Function):**
    *   **Role:** Normalizes the JSON structure. It maps different fields (e.g., `articles` from News vs `entities` from Crunchbase) into a unified object.
    *   **Output:** Standardized items containing `source`, `type`, `title/name`, and `description`.
*   **AI Agent / Google Gemini Chat Model:**
    *   **Role:** Acts as a logic filter. It evaluates the "Interest" of a news item.
    *   **Configuration:** Instructed to look for funding, launches, expansions, and hiring.
    *   **Prompt:** Strictly requires a JSON output format including an "interest" key (Yes/No).
*   **Code:**
    *   **Role:** Clean up the AI output. It removes Markdown code blocks (```json) often returned by LLMs to ensure the data is parsable JSON.

#### 2.3 Enrich Data & Log Opportunity
**Overview:** Final processing of validated opportunities, including external enrichment and multi-channel reporting.
**Nodes Involved:** `If`, `Enrich Contacts`, `Save to Google Sheets`, `Send Telegram Alert`, `No Operation, do nothing`.

*   **If (Filter):**
    *   **Role:** Checks if the AI marked the item with `interest: "yes"`.
*   **Enrich Contacts (HTTP Request):**
    *   **Role:** Calls an external enrichment API (placeholder URL) using the `company` name identified by the AI.
*   **Save to Google Sheets:**
    *   **Role:** Appends the opportunity details (Company, Event, Summary, Link) to a specific spreadsheet.
*   **Send Telegram Alert:**
    *   **Role:** Sends a message to a Telegram Chat ID with a summary of the new opportunity.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Daily Trigger | Cron | Workflow Scheduler | None | Multiple APIs | Set your desired schedule. |
| Google News | HTTP Request | Data Extraction | Daily Trigger | Merge2 | Add API keys for Google News. |
| Crunchbase RSS | HTTP Request | Data Extraction | Daily Trigger | None* | Gather Info |
| Twitter API | HTTP Request | Data Extraction | Daily Trigger | Merge2 | Add API keys for Twitter. |
| Product Hunt API | HTTP Request | Data Extraction | Daily Trigger | None* | Add API keys for Product Hunt. |
| Merge2 | Merge | Data Aggregation | News, Twitter | Merge Sources | Find opportunities |
| Merge Sources | Function | Data Normalization | Merge2 | AI Agent | Find opportunities |
| AI Agent | AI Agent | Opportunity Analysis | Merge Sources | Code | Find opportunities |
| Google Gemini | Gemini Chat | AI Reasoning | AI Agent | AI Agent | Connect your Google Gemini account. |
| Code | Code | JSON Cleaning | AI Agent | If | Find opportunities |
| If | If | Logic Gate | Code | Enrich / NoOp | enrich data and log opportunity |
| Enrich Contacts | HTTP Request | Lead Enrichment | If (True) | Sheets, Telegram | Review and update enrichment URL. |
| Save to Google Sheets | Google Sheets | Data Storage | Enrich Contacts | None | Specify 'Sheet ID' and 'Range'. |
| Send Telegram Alert | Telegram | Notification | Enrich Contacts | None | Set the 'Chat ID'. |
| No Operation | NoOp | Flow Termination | If (False) | None | enrich data and log opportunity |

*\*Note: In the provided JSON, some extraction nodes are not fully connected to the Merge2 node but are logically grouped within the "Gather Info" block.*

---

### 4. Reproducing the Workflow from Scratch

1.  **Trigger Setup:** Create a **Schedule (Cron)** node set to "Daily".
2.  **API Extraction:** 
    *   Create **HTTP Request** nodes for Google News, Twitter (v2 Search), and Product Hunt. 
    *   Configure each with their respective API URLs and Auth Headers.
    *   Connect the Trigger to all API nodes.
3.  **Aggregation:** 
    *   Add a **Merge** node to combine outputs from the API nodes.
    *   Add a **Function** node (Merge Sources) to normalize the JSON. Use a script to map `json.articles`, `json.entities`, and `json.data.posts` into a common structure (`title`, `description`, `source`).
4.  **AI Integration:**
    *   Add an **AI Agent** node. Set the prompt to act as a "Business Opportunity Filter".
    *   Connect a **Google Gemini Chat Model** node to the AI Agent.
    *   Provide the standardized feed via `{{ JSON.stringify($json) }}`.
5.  **Post-Processing:**
    *   Add a **Code** node after the AI Agent to perform `JSON.parse()` on the string output and strip Markdown backticks.
    *   Add an **If** node to check if `{{ $json.interest }}` contains "yes".
6.  **Action Steps:**
    *   Connect the "True" branch of the If node to an **HTTP Request** node for company enrichment (e.g., Clearbit or Apollo).
    *   Connect the enrichment output to a **Google Sheets** node (Action: Append) and a **Telegram** node (Action: Send Message).
    *   Connect the "False" branch to a **NoOp** node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Google News API Documentation | [https://newsapi.org/docs](https://newsapi.org/docs) |
| Twitter API Developer Portal | [https://developer.x.com/en/docs](https://developer.x.com/en/docs) |
| n8n AI Documentation | [https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.agent/](https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.agent/) |
| Telegram Bot Setup Guide | [https://core.telegram.org/bots#how-do-i-create-a-bot](https://core.telegram.org/bots#how-do-i-create-a-bot) |