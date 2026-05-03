Aggregate tech trend signals from RSS feeds into Google Sheets and Slack

https://n8nworkflows.xyz/workflows/aggregate-tech-trend-signals-from-rss-feeds-into-google-sheets-and-slack-13839


# Aggregate tech trend signals from RSS feeds into Google Sheets and Slack

### 1. Workflow Overview

This workflow is an automated "Trend Intelligence Engine" designed to scan tech ecosystems (Hacker News and Product Hunt) for emerging signals. It goes beyond simple RSS reading by applying a weighted scoring system based on customizable keyword groups and clustering individual posts into high-level "Themes."

The logic is divided into four main functional blocks:
1. **Data Acquisition:** Triggers daily to fetch raw RSS XML data from multiple sources in parallel.
2. **Parsing & Scoring:** Normalizes disparate XML formats and scores each item based on its source and keyword matches within specific tech categories.
3. **Intelligence Aggregation:** Clusters individual signals into themes and calculates "Theme Strength" using volume and source diversity bonuses.
4. **Multi-Channel Distribution:** Logs granular data and theme summaries to Google Sheets and sends summarized intelligence reports via Slack and Gmail.

---

### 2. Block-by-Block Analysis

#### 2.1 Data Acquisition
This block handles the timing and collection of raw data.
*   **Nodes Involved:** `Daily Schedule Trigger`, `Fetch Hacker News RSS`, `Fetch Product Hunt RSS`, `Wait for All Feeds`.
*   **Node Details:**
    *   **Daily Schedule Trigger:** Fires every 24 hours.
    *   **Fetch Hacker News RSS (HTTP Request):** Fetches the newest items with at least 50 points (to filter for quality). Sets a 15s timeout.
    *   **Fetch Product Hunt RSS (HTTP Request):** Fetches the latest product launches.
    *   **Wait for All Feeds (Merge):** Uses "Wait" mode to ensure both HTTP requests finish before the parser starts.

#### 2.2 Signal Parsing & Scoring
This block transforms raw XML into structured data and applies an analytical layer.
*   **Nodes Involved:** `Parse All RSS Feeds`, `Score and Classify Signals`.
*   **Node Details:**
    *   **Parse All RSS Feeds (Code):** Normalizes XML fields (`title`, `link`, `description`, `pubDate`). It assigns a **Source Weight** (HN: 1.5, PH: 1.3) to influence the final importance of the signal.
    *   **Score and Classify Signals (Code):** Uses 7 keyword groups (AI, No-Code, SaaS, etc.). 
        *   *Logic:* It calculates a score based on hits in the description and gives a **1.5x bonus** for keyword matches in the title.
        *   *Edge Cases:* If no keywords match, the signal is dropped from the intelligence report to reduce noise.

#### 2.3 Intelligence Clustering
This block converts a list of articles into a strategic summary.
*   **Nodes Involved:** `Aggregate Signals into Themes`.
*   **Node Details:**
    *   **Aggregate Signals into Themes (Code):** Groups items by their primary category. 
        *   *Strength Formula:* Uses a logarithmic volume bonus and a source diversity multiplier (`1 + (sourceDiversity - 1) * 0.3`). 
        *   *Classification:* Themes are ranked as `VERY_STRONG` (score 30+), `STRONG` (15+), `MODERATE` (8+), or `WEAK`.

#### 2.4 Reporting & Distribution
This block formats the data for different stakeholders (long-term storage vs. immediate notification).
*   **Nodes Involved:** `Build Intelligence Report`, `Flatten Top Signals for Sheet`, `Log Signals to Sheet`, `Prepare Themes for Sheet`, `Log Themes to Sheet`, `Post Report to Slack`, `Email Daily Report`.
*   **Node Details:**
    *   **Build Intelligence Report (Code):** Generates a plain-text version for Slack and a styled HTML table for Gmail. Includes "Action Items" if strong signals are detected.
    *   **Log Nodes (Google Sheets):** Requires two tabs (`signal` and `themes`). It appends new rows for every run.
    *   **Post Report to Slack / Email Daily Report:** Sends the generated reports to the configured channel/recipient.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Daily Schedule Trigger | Schedule Trigger | Workflow Entry | - | Fetch HN, Fetch PH | Trigger fires daily. |
| Fetch Hacker News RSS | HTTP Request | Data Fetching | Daily Trigger | Wait for All Feeds | Fetch RSS Feeds in parallel. |
| Fetch Product Hunt RSS | HTTP Request | Data Fetching | Daily Trigger | Wait for All Feeds | Fetch RSS Feeds in parallel. |
| Wait for All Feeds | Merge | Synchronization | Fetch HN, Fetch PH | Parse RSS Feeds | Wait for both to complete. |
| Parse All RSS Feeds | Code | Data Normalization | Wait for All Feeds | Score & Classify | RSS XML is parsed into items. |
| Score and Classify Signals | Code | Analytical Scoring | Parse RSS Feeds | Aggregate Themes | Scored against 7 keyword groups. |
| Aggregate Signals into Themes | Code | Data Clustering | Score & Classify | Build Intelligence Report | Signals clustered into themes. |
| Build Intelligence Report | Code | Formatting | Aggregate Themes | Slack, Gmail, Flatten nodes | Intelligence report in text/HTML. |
| Flatten Top Signals for Sheet | Code | Data Transformation | Build Report | Log Signals to Sheet | Deliver and Log. |
| Log Signals to Sheet | Google Sheets | Archiving | Flatten Signals | - | Log signals to separate tab. |
| Prepare Themes for Sheet | Code | Data Transformation | Build Report | Log Themes to Sheet | Deliver and Log. |
| Log Themes to Sheet | Google Sheets | Archiving | Prepare Themes | - | Log theme summaries to tab. |
| Post Report to Slack | Slack | Notification | Build Report | - | Report posted to Slack. |
| Email Daily Report | Gmail | Notification | Build Report | - | Update the email address. |

---

### 4. Reproducing the Workflow from Scratch

1.  **Preparation:** Create a Google Sheet with two tabs:
    *   `signal`: Headers: `date`, `title`, `source`, `score`, `category`, `url`.
    *   `themes`: Headers: `date`, `category`, `signal_level`, `theme_strength`, `item_count`, `sources`, `top_keywords`.
2.  **Trigger:** Add a **Schedule Trigger** set to "Interval" (24 hours).
3.  **Fetch Data:** 
    *   Add two **HTTP Request** nodes. 
    *   Node 1 (HN): `https://hnrss.org/newest?points=50&count=30`.
    *   Node 2 (PH): `https://www.producthunt.com/feed`.
    *   Set both to return "Text" (raw XML).
4.  **Sync:** Add a **Merge** node, set to "Wait" (no complex logic needed, just wait for all inputs).
5.  **Processing Logic:**
    *   Add a **Code** node ("Parse All RSS Feeds") to extract data using Regex or XML parsing. Assign `weight: 1.5` to HN and `1.3` to PH.
    *   Add a **Code** node ("Score and Classify Signals"). Define a `keywordGroups` object with your categories (AI, SaaS, etc.) and use `string.includes()` to calculate scores.
    *   Add a **Code** node ("Aggregate Signals into Themes") to group the previous array by category and apply the strength formulas.
6.  **Reporting:** 
    *   Add a **Code** node ("Build Intelligence Report") to construct two strings: one for text (Slack) and one for HTML (Gmail).
7.  **Output Setup:**
    *   Add two **Code** nodes to transform the data arrays into flat rows for Google Sheets.
    *   Add **Google Sheets** nodes using "Append" operation. Map your sheet ID and GIDs.
    *   Add **Slack** (Message node) and **Gmail** (Send node) using OAuth2 credentials. Update the `To` address in Gmail.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Hacker News RSS Source** | Configured via [hnrss.org](https://hnrss.org/) |
| **Product Hunt RSS Source** | Uses standard [producthunt.com/feed](https://www.producthunt.com/feed) |
| **Keyword Group Weights** | Edit the "Score and Classify Signals" node to change industry focus. |
| **Strength Thresholds** | Themes are categorized based on scores 30, 15, and 8. |
| **Recipient Setup** | Ensure the recipient email is changed in the Gmail node before activation. |