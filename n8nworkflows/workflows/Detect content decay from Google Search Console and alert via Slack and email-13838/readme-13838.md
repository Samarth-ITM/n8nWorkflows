Detect content decay from Google Search Console and alert via Slack and email

https://n8nworkflows.xyz/workflows/detect-content-decay-from-google-search-console-and-alert-via-slack-and-email-13838


# Detect content decay from Google Search Console and alert via Slack and email

This document provides a comprehensive technical reference for the n8n workflow: **Detect content decay and alert via Slack and email**.

### 1. Workflow Overview

The purpose of this workflow is to proactively identify "content decay"—a decline in organic search performance—by monitoring Google Search Console (GSC) data. It compares recent performance (last 7 days) against a historical baseline (previous 28 days), classifies the health of each page, and triggers multi-channel alerts and actionable remediation tasks.

**Logical Blocks:**
*   **1.1 Data Acquisition:** Triggers weekly and fetches two distinct time-period datasets from GSC.
*   **1.2 Performance Analysis:** Normalizes data and applies logic to categorize pages (e.g., Critical Decay vs. Growing).
*   **1.3 Data Persistence:** Logs all analyzed data into a master "Decay Log" Google Sheet.
*   **1.4 Reporting & Alerting:** Generates a summary report sent via Slack and Email.
*   **1.5 Automated Task Generation:** Identifies critical failures and suggests specific SEO fixes in a "Fix Tasks" Google Sheet.

---

### 2. Block-by-Block Analysis

#### 2.1 Data Acquisition
*   **Overview:** Sets the timeframe and retrieves raw SEO metrics for the site.
*   **Nodes Involved:** `Weekly Monday 8AM Trigger`, `Calculate Date Ranges`, `Fetch GSC Recent 7 Days`, `Fetch GSC Previous 28 Days`.
*   **Node Details:**
    *   **Weekly Monday 8AM Trigger:** (Schedule Trigger) Set to run every Monday at 08:00.
    *   **Calculate Date Ranges:** (Code) Uses JavaScript to define four dates: `recentStart/End` (last 7 days, offset by 3 days for GSC latency) and `prevStart/End` (the 28 days prior).
    *   **Fetch GSC Nodes:** (HTTP Request) Calls the Google Search Analytics API. Uses `$env.GSC_SITE_URL` and OAuth2. It requests `page` dimensions with a 500-row limit.
    *   **Edge Cases:** API Quota limits or expired OAuth2 credentials.

#### 2.2 Performance Analysis
*   **Overview:** The "brain" of the workflow where the actual comparison happens.
*   **Nodes Involved:** `Compare Periods and Detect Decay`.
*   **Node Details:**
    *   **Type:** Code Node (JavaScript).
    *   **Logic:** It maps the 28-day data to a weekly average (dividing by 4). It then calculates `% change` in clicks/impressions and absolute change in average position.
    *   **Classification Criteria:**
        *   *CRITICAL:* Clicks down >50% OR (Position down >5 AND Clicks down >30%).
        *   *DECAYING:* Clicks down >30% OR Position down >3.
        *   *EARLY_DECAY:* Clicks down >15% OR Position down >1.5.
        *   *GROWING:* Clicks up >20%.

#### 2.3 Reporting & Alerting
*   **Overview:** Filters for problematic pages and notifies the team.
*   **Nodes Involved:** `Filter Only Decaying Pages`, `Build Weekly Decay Report`, `Post Report to Slack`, `Email Weekly Report`.
*   **Node Details:**
    *   **Filter Only Decaying Pages:** Keeps only items with severity `critical`, `warning`, or `watch`.
    *   **Build Weekly Decay Report:** Aggregates statistics (total clicks lost, count of pages per status) and builds a Markdown-formatted string for Slack and HTML/Text for Email.
    *   **Communication Nodes:** Slack sends the text to a configured channel; Email Send requires SMTP or OAuth2 setup.

#### 2.4 Automated Task Generation
*   **Overview:** Translates data into SEO actions for the worst-performing pages.
*   **Nodes Involved:** `Check If Any Critical Pages`, `Generate Fix Tasks for Critical Pages`, `Log Fix Tasks to Sheet`.
*   **Node Details:**
    *   **Logic:** If a page is `CRITICAL`, it analyzes *why* (e.g., if CTR dropped, it suggests updating meta titles; if Position dropped, it suggests a backlink audit).
    *   **Output:** Appends individual rows to the "Fix Tasks" tab in Google Sheets.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Weekly Monday 8AM Trigger | Schedule Trigger | Workflow Entry | (None) | Calculate Date Ranges | (None) |
| Calculate Date Ranges | Code | Variable Setup | Trigger | Fetch GSC Nodes | ## 1. Fetch GSC Data |
| Fetch GSC Recent 7 Days | HTTP Request | Data Retrieval | Date Ranges | Compare Periods | ## 1. Fetch GSC Data |
| Fetch GSC Previous 28 Days | HTTP Request | Data Retrieval | Date Ranges | Compare Periods | ## 1. Fetch GSC Data |
| Compare Periods... | Code | Analysis Engine | Fetch GSC Nodes | Filter, Log Sheet | ## 2. Detect Decay |
| Log All Results... | Google Sheets | Data Logging | Compare Periods | (None) | ## 2. Detect Decay |
| Filter Only Decaying... | Filter | Data Reduction | Compare Periods | Build Report | (None) |
| Build Weekly Decay Report | Code | Content Gen | Filter | Slack, Email, Check | ## 3. Report and Alert |
| Post Report to Slack | Slack | Notification | Build Report | (None) | ## 3. Report and Alert |
| Email Weekly Report | Email Send | Notification | Build Report | (None) | ⚠️ Update the email address in this node... |
| Check If Any Critical... | Filter | Logic Gate | Build Report | Generate Fix Tasks | ## 4. Auto-Generate Fix Tasks |
| Generate Fix Tasks... | Code | Advice Logic | Check Filter | Log Fix Tasks | ## 4. Auto-Generate Fix Tasks |
| Log Fix Tasks to Sheet | Google Sheets | Task Logging | Generate Tasks | (None) | ## 4. Auto-Generate Fix Tasks |

---

### 4. Reproducing the Workflow from Scratch

1.  **Environment Setup:**
    *   Create two environment variables in n8n: `GSC_SITE_URL` (the full URL of your site property) and `DECAY_SHEET_URL` (the link to your Google Sheet).
2.  **Google Sheet Preparation:**
    *   Tab 1: `Decay Log` with headers: `date`, `page_path`, `signal`, `clicks_now`, `clicks_before`, `click_change_pct`, `position_now`, `position_before`, `position_change`, `impressions_now`, `impression_change_pct`, `ctr_now`.
    *   Tab 2: `Fix Tasks` with headers: `created`, `priority`, `page_path`, `page_url`, `signal`, `click_change_pct`, `position_change`, `recommended_action`.
3.  **Trigger & Dates:**
    *   Add a **Schedule Trigger** (Mondays, 08:00).
    *   Add a **Code Node** to calculate the ISO date strings for the recent (7-day) and baseline (28-day) periods.
4.  **API Integration:**
    *   Add two **HTTP Request Nodes**. Connect to Google Search Console API.
    *   Configure Method: `POST`. URL: `https://www.googleapis.com/webmasters/v3/sites/{{$env.GSC_SITE_URL}}/searchAnalytics/query`.
    *   Body: JSON containing `startDate`, `endDate`, and `dimensions: ["page"]`.
5.  **Comparison Logic:**
    *   Add a **Code Node** to merge the two JSON arrays by `page` URL. Calculate metrics and assign `signal` (CRITICAL, DECAYING, etc.) and `severity` strings.
6.  **Persistence & Filtering:**
    *   Add a **Google Sheets Node** (Append) to log all outputs from the Comparison node to the `Decay Log` tab.
    *   Add a **Filter Node** after the Comparison node to pass only items where `severity` is not "ok" or "good".
7.  **Reporting:**
    *   Add a **Code Node** to summarize the filtered data into a single string.
    *   Add **Slack** and **Email Send** nodes using the report string as the body.
8.  **Remediation:**
    *   Add a **Filter Node** to check if `has_critical` is true.
    *   Add a **Code Node** to map specific performance drops to text recommendations.
    *   Add a final **Google Sheets Node** to append these tasks to the `Fix Tasks` tab.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| GSC Data Delay | Google Search Console data is typically delayed by 2-3 days; the workflow accounts for this in the Date Ranges node. |
| Ranking Drop Thresholds | You can modify the "Compare" node to be more or less sensitive to position changes (default is 1.5, 3, or 5 spots). |
| Environment Variables | [How to set variables in n8n](https://docs.n8n.io/hosting/configuration/environment-variables/) |
| Recommended Refresh | For critical pages, a content refresh (adding 500+ words of new value) is the standard recommended action. |