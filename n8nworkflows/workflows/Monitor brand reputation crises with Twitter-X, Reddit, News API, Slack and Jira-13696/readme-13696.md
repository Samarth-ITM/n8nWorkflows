Monitor brand reputation crises with Twitter/X, Reddit, News API, Slack and Jira

https://n8nworkflows.xyz/workflows/monitor-brand-reputation-crises-with-twitter-x--reddit--news-api--slack-and-jira-13696


# Monitor brand reputation crises with Twitter/X, Reddit, News API, Slack and Jira

This document provides a comprehensive technical analysis of the **AI Brand Reputation Crisis Detector** workflow in n8n.

---

### 1. Workflow Overview

The **AI Brand Reputation Crisis Detector** is an automated system designed to monitor brand mentions across multiple digital channels (Twitter/X, Reddit, and global News/Blogs) in near real-time. It moves beyond simple keyword alerts by using an internal sentiment engine to calculate the "Impact Score" of mentions, taking into account engagement metrics and source credibility.

The workflow is structured into five logical phases:
1.  **Data Ingestion:** Periodic polling of social and news APIs.
2.  **Normalization & Deduplication:** Unifying disparate data structures and ensuring each mention is analyzed only once using persistent workflow memory.
3.  **AI Intelligence & Trend Analysis:** Calculating sentiment scores and comparing current activity against a 24-hour historical baseline to detect anomalies.
4.  **Crisis Activation:** Grouping negative mentions into a "Crisis Brief" with executive-level summaries and response plans.
5.  **Multi-Channel Alerting:** Dispatching notifications via Slack, Email, and Jira, and logging results for long-term reporting.

---

### 2. Block-by-Block Analysis

#### 2.1 Multi-Platform Monitoring
This block handles the retrieval of raw data from the web.
*   **Nodes Involved:** `Every 10 Minutes`, `Monitor Twitter/X`, `Monitor Reddit`, `Monitor News & Blogs`.
*   **Node Details:**
    *   **Every 10 Minutes (Schedule Trigger):** Fires every 10 minutes to ensure near real-time response.
    *   **Monitor Twitter/X (Twitter Node):** Uses OAuth2 to search for `brandName OR @brandHandle`. Returns up to 100 recent tweets.
    *   **Monitor Reddit (Reddit Node):** Searches for keywords across all subreddits. Returns up to 50 items.
    *   **Monitor News & Blogs (HTTP Request):** Connects to `newsapi.org`. Configuration requires a `q` parameter (brand name) and an API Key.
*   **Edge Cases:** API rate limits or expired credentials on X/Reddit.

#### 2.2 Data Normalization & Deduplication
This block ensures all data follows the same schema regardless of the source.
*   **Nodes Involved:** `Normalize Data Format`, `Remove Duplicates`, `Merge Platform Data`, `Track Analyzed Mentions`, `API Rate Limiter`, `Only New Mentions`.
*   **Node Details:**
    *   **Normalize Data Format (Code):** Maps platform-specific fields (e.g., `id_str` for X, `permalink` for Reddit) to a unified object containing `platform`, `text`, `author`, `engagement`, and `isVerified`.
    *   **Track Analyzed Mentions (Code):** Uses `getWorkflowStaticData('global')` to store a list of processed IDs. It automatically purges entries older than 7 days to manage memory.
    *   **API Rate Limiter (Wait):** A 1-second pause to prevent downstream processing surges.
*   **Failure Modes:** Static data overflow if thousands of mentions occur (handled by the 7-day purge logic).

#### 2.3 AI Sentiment Analysis & Trend Detection
This is the "brain" of the workflow, calculating the severity of incoming data.
*   **Nodes Involved:** `AI Sentiment Analysis Engine`, `Calculate Trend Baseline`.
*   **Node Details:**
    *   **AI Sentiment Analysis Engine (Code):** Uses a multi-tier keyword dictionary (Critical, Severe, Moderate, Mild) to score text. It adds "Amplification Scores" based on engagement (e.g., +30 impact if >1000 likes) and author verification status.
    *   **Calculate Trend Baseline (Code):** Compares current batch sentiment to a 24-hour moving average. It detects "Sharp Drops" in sentiment or "Spikes" in negative ratios.
*   **Variables:** `impactScore`, `sentimentDrop`, `trendAlert` (NORMAL, MEDIUM, HIGH, CRISIS).

#### 2.4 Crisis Response Activation
This block generates human-readable alerts when thresholds are met.
*   **Nodes Involved:** `Filter Crisis Triggers`, `Aggregate Crisis Data`, `Generate Crisis Brief`.
*   **Node Details:**
    *   **Filter Crisis Triggers (Switch):** Routes data based on the `requiresCrisisResponse` boolean.
    *   **Generate Crisis Brief (Code):** Dynamically constructs an executive report. It selects the top 5 most impactful mentions and suggests action plans (e.g., "Convene crisis management team" for Critical level).

#### 2.5 Multi-Channel Alerting & Logging
*   **Nodes Involved:** `Alert Crisis Team - Slack`, `Alert Crisis Team - Email`, `Create JIRA Crisis Ticket`, `Log Crisis Event`, `Log Routine Monitoring`.
*   **Node Details:**
    *   **Slack:** Posts a formatted message with "Activate Response" buttons.
    *   **Email:** Sends a full brief via SMTP to leadership.
    *   **Jira:** Creates a "Highest" priority issue in the specified project.
    *   **Log Routine (Code):** Saves a summary of every run to static data for historical trend reporting.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Every 10 Minutes | Schedule Trigger | Periodic Execution | None | Twitter, Reddit, News | 📡 Multi-Platform Monitoring |
| Monitor Twitter/X | Twitter | Data Sourcing | Every 10 Minutes | Normalize Data | 📡 Multi-Platform Monitoring |
| Monitor Reddit | Reddit | Data Sourcing | Every 10 Minutes | Normalize Data | 📡 Multi-Platform Monitoring |
| Monitor News & Blogs | HTTP Request | Data Sourcing | Every 10 Minutes | Normalize Data | 📡 Multi-Platform Monitoring |
| Normalize Data Format | Code | Transformation | Twitter, Reddit, News | Remove Duplicates | 🔄 Data Normalization & Deduplication |
| Remove Duplicates | Filter | Data Cleaning | Normalize Data | Merge Platform Data | 🔄 Data Normalization & Deduplication |
| Merge Platform Data | Merge | Synchronization | Remove Duplicates | Track Mentions | 🔄 Data Normalization & Deduplication |
| Track Analyzed Mentions | Code | Deduplication Logic | Merge Platform Data | API Rate Limiter | 🔄 Data Normalization & Deduplication |
| API Rate Limiter | Wait | Throttling | Track Mentions | Only New Mentions | 🔄 Data Normalization & Deduplication |
| Only New Mentions | Filter | Flow Control | API Rate Limiter | AI Sentiment Engine | 🔄 Data Normalization & Deduplication |
| AI Sentiment Analysis Engine | Code | AI Intelligence | Only New Mentions | Trend Baseline | 🤖 AI Sentiment Analysis & Trend Detection |
| Calculate Trend Baseline | Code | Trend Analysis | AI Sentiment Engine | Filter Crisis | 🤖 AI Sentiment Analysis & Trend Detection |
| Filter Crisis Triggers | Switch | Routing | Trend Baseline | Aggregate Crisis | 🚨 Crisis Response Activation |
| Aggregate Crisis Data | Code | Data Aggregation | Filter Crisis | Generate Brief | 🚨 Crisis Response Activation |
| Generate Crisis Brief | Code | Content Generation | Aggregate Crisis | Slack Alert | 🚨 Crisis Response Activation |
| Alert Crisis Team - Slack | Slack | Alerting | Generate Brief | Email Alert | 🚨 Crisis Response Activation |
| Alert Crisis Team - Email | Email Send | Alerting | Slack Alert | Jira Ticket | 🚨 Crisis Response Activation |
| Create JIRA Crisis Ticket | Jira | Tracking | Email Alert | Log Crisis | 🚨 Crisis Response Activation |
| Log Crisis Event | Code | Event Logging | Jira Ticket | Log Routine | 🚨 Crisis Response Activation |
| Log Routine Monitoring | Code | Performance Logging | Log Crisis | None | 📊 Routine Logging |

---

### 4. Reproducing the Workflow from Scratch

1.  **Trigger Setup:** Create a **Schedule Trigger** set to interval mode (10 minutes).
2.  **Source Integration:**
    *   Add a **Twitter Node** (Search operation). Use expressions for the brand name.
    *   Add a **Reddit Node** (Search operation, subreddit: all).
    *   Add an **HTTP Request Node** for News API (URL: `https://newsapi.org/v2/everything?q=BRAND`).
3.  **Data Normalization:** Add a **Code Node** (Run for each item). Use logic to map diverse keys (author, followers, engagement) into a single JSON structure.
4.  **Deduplication Strategy:** 
    *   Add a **Merge Node** to collect all inputs.
    *   Add a **Code Node** using `$getWorkflowStaticData('global')`. Logic: create a key like `platform_id`, check if it exists in the global object, if not, store it and pass the item.
5.  **Intelligence Engine:**
    *   Add a **Code Node** for Sentiment. Define arrays of keywords (positive/negative) and loop through the text to increment/decrement a `sentimentScore`.
    *   Add a **Code Node** for Trends. Store the average sentiment of the last 144 runs (24 hours) in static data and calculate the difference from the current run.
6.  **Alerting Path:**
    *   Add a **Switch Node** to filter for `requiresCrisisResponse == true`.
    *   Add a **Code Node** to build a string (the Brief) using Markdown.
    *   Connect **Slack**, **Email (SMTP)**, and **Jira** nodes in sequence.
7.  **Finalize Persistence:** Add a final **Code Node** to push the results into a `monitoringLog` array within the workflow's static data for historical dashboarding.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **News API Key Required** | Obtain from [https://newsapi.org/](https://newsapi.org/) |
| **X (Twitter) API Access** | Requires OAuth2 and at least Basic tier for search endpoints |
| **Static Data Limitations** | Static data is preserved between runs but cleared if the workflow is updated manually in some n8n versions |
| **Impact Score Weights** | Verified accounts (+15), News sources (+40), High engagement (+10 to +30) |