Summarize RSS feeds into a daily AI digest with Gemini, Slack, and Gmail

https://n8nworkflows.xyz/workflows/summarize-rss-feeds-into-a-daily-ai-digest-with-gemini--slack--and-gmail-13674


# Summarize RSS feeds into a daily AI digest with Gemini, Slack, and Gmail

# Reference Documentation: Daily AI News Digest Workflow

This document provides a technical overview and reconstruction guide for the n8n workflow designed to aggregate RSS feeds, analyze them using Google Gemini AI, and distribute a formatted digest via Slack and Gmail.

## 1. Workflow Overview

The **Daily AI News Digest** workflow automates the process of staying informed by collecting articles from multiple tech news sources, filtering them for relevance, and providing a scored summary every weekday morning. It replaces manual news checking with an AI-curated briefing delivered directly to communication platforms.

### Logical Blocks
*   **1.1 Input Reception:** A scheduled trigger initiates the process every weekday.
*   **1.2 Feed Collection:** Multiple RSS feeds are fetched simultaneously.
*   **1.3 Data Processing:** Articles are merged, deduplicated, and limited to a manageable number via JavaScript.
*   **1.4 AI Summarization:** Google Gemini analyzes the article list, creates concise summaries, and assigns an importance score (1–10).
*   **1.5 Digest Formatting:** A secondary script organizes the AI output into human-readable formats for different platforms (Slack and Email).
*   **1.6 Delivery:** The final digest is sent to a Slack channel and a Gmail recipient.

---

## 2. Block-by-Block Analysis

### 1.1 Input Reception & Feed Collection
This block handles the timing and retrieval of raw data.

*   **Nodes Involved:** `Morning Schedule`, `Tech News Feed`, `Hacker News Feed`.
*   **Node Details:**
    *   **Morning Schedule (Schedule Trigger):** Set to a Cron expression (`0 8 * * 1-5`) to trigger at 08:00 AM every Monday through Friday.
    *   **Tech News Feed / Hacker News Feed (RSS Feed Read):** These nodes fetch the latest items from specific URLs (Ars Technica and Hacker News). 
    *   **Edge Cases:** Failure to reach the RSS URL or malformed XML will stop the workflow execution.

### 1.2 Data Processing
Merges disparate data streams into a single standardized object.

*   **Nodes Involved:** `Merge and Deduplicate`.
*   **Node Details:**
    *   **Merge and Deduplicate (Code):** Uses JavaScript to combine results from all RSS nodes. It limits each source to 10 items, removes duplicate URLs, and keeps only the top 15 most recent articles.
    *   **Key Expressions:** Accesses preceding nodes via `$('Node Name').all()`.
    *   **Output:** A single JSON object containing an array of `articles`.

### 1.3 AI Summarization
Utilizes Large Language Models to interpret and rank the news.

*   **Nodes Involved:** `Summarize and Score`, `Gemini Chat Model`.
*   **Node Details:**
    *   **Summarize and Score (Chain LLM):** Acts as the orchestrator for the AI prompt. It passes the article list to Gemini.
    *   **Gemini Chat Model (Google Gemini):** Configured with `gemini-1.5-flash` and a temperature of `0.3` for consistent, factual summaries.
    *   **Potential Failures:** API quota limits (Gemini free tier), authentication errors, or the AI failing to return the requested JSON-like format.

### 1.4 Digest Formatting & Delivery
Prepares the final presentation and sends it.

*   **Nodes Involved:** `Build Digest`, `Post to Slack`, `Email Digest`.
*   **Node Details:**
    *   **Build Digest (Code):** Parses the AI's response. It uses Regex to find the JSON array in the text and creates two versions of the digest: a Markdown-heavy version for Slack and a plain-text/HTML-ready version for Email. Stories with a score of 7+ are highlighted as "Top Stories."
    *   **Post to Slack (Slack):** Sends the `slackText` variable to a specified channel.
    *   **Email Digest (Gmail):** Sends the `emailBody` to a specified recipient with a dynamic subject line showing the date and count of top stories.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Morning Schedule | Schedule Trigger | Timing | (None) | Tech News Feed, Hacker News Feed | |
| Tech News Feed | RSS Feed Read | Data Fetching | Morning Schedule | Merge and Deduplicate | Feed collection: Pulls articles from multiple RSS sources and merges them. |
| Hacker News Feed | RSS Feed Read | Data Fetching | Morning Schedule | Merge and Deduplicate | Feed collection: Pulls articles from multiple RSS sources and merges them. |
| Merge and Deduplicate | Code | Data Sanitization | Tech/Hacker Feeds | Summarize and Score | Feed collection: Pulls articles from multiple RSS sources and merges them. |
| Summarize and Score | Chain LLM | AI Logic | Merge and Deduplicate | Build Digest | AI summarization: Gemini reads each article and creates a scored summary. |
| Gemini Chat Model | Gemini Chat Model | AI Engine | (None) | Summarize and Score | AI summarization: Gemini reads each article and creates a scored summary. |
| Build Digest | Code | Formatting | Summarize and Score | Post to Slack, Email Digest | Digest delivery: Formats the digest and delivers via Slack and Gmail. |
| Post to Slack | Slack | Delivery | Build Digest | (None) | Digest delivery: Formats the digest and delivers via Slack and Gmail. |
| Email Digest | Gmail | Delivery | Build Digest | (None) | Digest delivery: Formats the digest and delivers via Slack and Gmail. |

---

## 4. Reproducing the Workflow from Scratch

1.  **Trigger Setup:**
    *   Create a **Schedule Trigger** node. Set it to "Cron" with the expression `0 8 * * 1-5`.
2.  **Source Setup:**
    *   Add two **RSS Feed Read** nodes.
    *   Node 1 URL: `https://feeds.arstechnica.com/arstechnica/technology-lab`
    *   Node 2 URL: `https://hnrss.org/frontpage`
3.  **Aggregation Logic:**
    *   Add a **Code** node named "Merge and Deduplicate". 
    *   Paste logic that collects items from both RSS nodes, checks for unique URLs, and returns an array of the 15 newest items.
4.  **AI Integration:**
    *   Add a **Basic LLM Chain** node (called "Summarize and Score").
    *   Connect a **Google Gemini Chat Model** node to its sub-input. Use model `gemini-1.5-flash`.
    *   In the Chain node, provide a prompt instructing the AI to: "Review the following articles, provide a 1-sentence summary for each, and score them 1-10 based on tech impact. Return the result in a JSON-like array."
5.  **Formatting Logic:**
    *   Add another **Code** node named "Build Digest".
    *   Write a script to iterate through the AI's scored results. Create two string variables: `slackText` (using Markdown) and `emailBody`.
6.  **Delivery Setup:**
    *   Add a **Slack** node. Action: "Post Message". Use `{{ $json.slackText }}` in the text field. Requires Slack OAuth2 credentials.
    *   Add a **Gmail** node. Action: "Send Email". Use `{{ $json.emailBody }}` in the body and `{{ $json.date }}` in the subject. Requires Google OAuth2 credentials.
7.  **Connections:** Connect nodes as per the Summary Table. Ensure the AI Chat Model is connected to the AI Model port of the Chain node.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Workflow Capabilities** | Summarizes RSS feeds, deduplicates content, and scores articles using AI. |
| **Customization** | Users can add more RSS nodes or modify the scoring prompt in the AI node. |
| **Prerequisites** | Google Gemini API Key, Slack App/OAuth, and Gmail OAuth2 setup are required. |