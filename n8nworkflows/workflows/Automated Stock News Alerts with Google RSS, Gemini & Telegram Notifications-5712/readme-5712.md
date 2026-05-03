Automated Stock News Alerts with Google RSS, Gemini & Telegram Notifications

https://n8nworkflows.xyz/workflows/automated-stock-news-alerts-with-google-rss--gemini---telegram-notifications-5712


# Automated Stock News Alerts with Google RSS, Gemini & Telegram Notifications

### 1. Workflow Overview

This workflow automates the process of monitoring stock-related news via Google Alerts RSS feeds, extracting and summarizing the content using AI, and delivering alerts through Telegram while maintaining a structured log in Google Sheets. It targets investors, analysts, and financial enthusiasts who require timely, concise, and actionable stock news updates.

The workflow is logically divided into the following blocks:

- **1.1 Scheduler & Triggering:** Periodically initiates the workflow to check for new stock news.
- **1.2 RSS Feed Collection:** Fetches stock news from multiple Google Alerts RSS feeds.
- **1.3 URL Normalization & Deduplication:** Merges feed data, extracts real article URLs from redirect links, and checks against Google Sheets to filter duplicates.
- **1.4 Content Extraction:** Uses Jina AI to extract the main article text from the real URL.
- **1.5 AI Summarization & Analysis:** Summarizes and analyzes the extracted content via an LLM through OpenRouter.
- **1.6 Data Storage & Notification:** Saves summarized data to Google Sheets and sends alerts via Telegram.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Scheduler & Triggering

- **Overview:** This block triggers the entire workflow every 15 minutes, ensuring continuous monitoring of stock news without manual intervention.
- **Nodes Involved:**  
  - `Schedule Trigger1`  
  - `Sticky Note1` (documentation)
- **Node Details:**

  - **Schedule Trigger1**  
    - Type: Schedule Trigger  
    - Role: Periodic execution every 15 minutes  
    - Configuration: Interval set to 15 minutes (adjustable)  
    - Input: None (start node)  
    - Output: Triggers three RSS feed nodes simultaneously  
    - Edge Cases: Possible clock drift; ensure server time is accurate  
    - Version: 1.2  

  - **Sticky Note1**  
    - Type: Sticky Note  
    - Role: Documentation of scheduling function  
    - Content: Explains scheduling purpose and configuration  

---

#### Block 1.2: RSS Feed Collection

- **Overview:** Collects stock-related news articles from three separate Google Alerts RSS feeds filtered by specific keywords or stock symbols.
- **Nodes Involved:**  
  - `NVDA Stock`  
  - `Stock Buyback`  
  - `Stock Acquisition`  
  - `Sticky Note` (RSS Feed explanation)
- **Node Details:**

  - **NVDA Stock, Stock Buyback, Stock Acquisition**  
    - Type: RSS Feed Read  
    - Role: Fetch latest news articles from Google Alerts RSS feeds (distinct feeds)  
    - Configuration: Each node configured with unique RSS feed URL (user-provided)  
    - Input: Trigger from Schedule Trigger1  
    - Output: Feeds data into `Merge` node  
    - Edge Cases: Feed may be empty, inaccessible, or malformed; handle gracefully  
    - Version: 1.2  

  - **Sticky Note**  
    - Type: Sticky Note  
    - Role: Documentation of RSS feed function and configuration  
    - Content: Explains source of RSS feeds and keywords used  

---

#### Block 1.3: URL Normalization & Deduplication

- **Overview:** This block merges RSS feed outputs, extracts the actual article URLs (removing Google redirect wrappers), appends or updates entries in Google Sheets to avoid duplicates, and filters out already processed articles.
- **Nodes Involved:**  
  - `Merge`  
  - `Get Real URL`  
  - `Append or update row in sheet`  
  - `Get row(s) in sheet`  
  - `If`  
  - `Sticky Note2` (Merge & URL extraction)  
  - `Sticky Note3` (Content extraction pre-check)
- **Node Details:**

  - **Merge**  
    - Type: Merge  
    - Role: Combine inputs from three RSS feed nodes  
    - Configuration: Number of inputs set to 3  
    - Input: RSS feed nodes outputs  
    - Output: Passes combined data to `Get Real URL`  

  - **Get Real URL**  
    - Type: Code (JavaScript)  
    - Role: Extracts the real URL from Google redirect URLs or passes original if not redirect  
    - Configuration: Runs once for each item; parses `link` field; extracts URL from `url` query parameter if link contains "google.com/url"  
    - Input: Merged RSS feed data  
    - Output: Provides `realUrl` field for downstream use  
    - Edge Cases: May fail if URL format changes; ensure regex is robust  

  - **Append or update row in sheet**  
    - Type: Google Sheets  
    - Role: Logs new articles or updates existing entries by ID  
    - Configuration: Matches rows by article `ID` field; appends or updates columns: ID, URL, Date, Title  
    - Input: Output from `Get Real URL`  
    - Output: Triggers `Get row(s) in sheet`  
    - Credentials: Requires Google Sheets API credentials  
    - Edge Cases: API quota limits, authorization failures  

  - **Get row(s) in sheet**  
    - Type: Google Sheets  
    - Role: Retrieves entries from the sheet to check if article is already processed (for deduplication)  
    - Configuration: Reads all rows from the sheet with ID and URL columns  
    - Input: Output from `Append or update row in sheet`  
    - Output: Connected to `If` node  

  - **If**  
    - Type: If  
    - Role: Checks whether the article summary field is empty or missing to decide if content extraction is needed  
    - Condition: `$json.Summary` equals empty string  
    - Input: Output from `Get row(s) in sheet`  
    - Output:  
      - If True: Triggers content extraction  
      - If False: Passes to no-operation node (skips extraction)  

  - **Sticky Note2**  
    - Type: Sticky Note  
    - Role: Documentation of merge and real URL extraction steps  
    - Content: Explains why URLs are normalized and merged  

  - **Sticky Note3**  
    - Type: Sticky Note  
    - Role: Documentation of content extraction pre-check logic  

---

#### Block 1.4: Content Extraction

- **Overview:** Extracts the main textual content of the article from the real URL using Jina AI’s article reader API, only if not already extracted.
- **Nodes Involved:**  
  - `Read URL content` (Jina AI)  
  - `No Operation, do nothing`  
- **Node Details:**

  - **Read URL content**  
    - Type: Jina AI  
    - Role: Extracts article body text from URL  
    - Configuration: URL set dynamically from Google Sheets data field `URL`  
    - Input: From `If` node when content is missing  
    - Output: Passes full text content for summarization  
    - Credentials: Requires Jina AI API credentials  
    - Edge Cases: API timeout, invalid URL, extraction failure  

  - **No Operation, do nothing**  
    - Type: No Operation  
    - Role: Acts as a placeholder for skipping extraction  
    - Input: From `If` node when summary exists  
    - Output: None  

---

#### Block 1.5: AI Summarization & Analysis

- **Overview:** Uses a Large Language Model (LLM) via OpenRouter to generate a concise summary and stock sentiment analysis from the extracted content.
- **Nodes Involved:**  
  - `AI Agent` (Langchain agent)  
  - `OpenRouter Chat Model` (LLM interface)  
  - `Sticky Note4` (Summarization explanation)
- **Node Details:**

  - **AI Agent**  
    - Type: Langchain Agent  
    - Role: Sends extracted content to LLM with custom prompt for summary, sentiment, and recommendation  
    - Configuration:  
      - Prompt instructs summary up to 200 words, stock category, sentiment (uptrend, downtrend, etc.), recommendation, plain text with emojis  
      - System message positions AI as professional stock analyst  
    - Input: Extracted content from `Read URL content`  
    - Output: Summarized text passed to Google Sheets append/update node  
    - Version: 2  

  - **OpenRouter Chat Model**  
    - Type: Langchain OpenRouter Language Model  
    - Role: Provides LLM capabilities (e.g., Gemini 2.0)  
    - Configuration: Model set to `google/gemini-2.0-flash-exp:free`  
    - Input: Connected to AI Agent as language model backend  
    - Output: Feeds AI Agent with model responses  
    - Credentials: Requires OpenRouter API key  
    - Version: 1  

  - **Sticky Note4**  
    - Type: Sticky Note  
    - Role: Describes summarization and analysis step  
    - Content: Explains use of LLM via OpenRouter and customization options  

---

#### Block 1.6: Data Storage & Notification

- **Overview:** Stores the summarized article along with metadata in Google Sheets and sends a real-time alert message via Telegram with the summary.
- **Nodes Involved:**  
  - `Append or update row in sheet1` (Google Sheets)  
  - `Send a text message` (Telegram)  
  - `Sticky Note5` (Storage and notification explanation)
- **Node Details:**

  - **Append or update row in sheet1**  
    - Type: Google Sheets  
    - Role: Updates existing row or appends new row with summary, URL, and Telegram sent flag  
    - Configuration: Matches by URL column; updates Summary and SentToTelegram fields  
    - Input: Output from AI Agent node  
    - Output: Triggers Telegram message node  
    - Credentials: Google Sheets API required  
    - Edge Cases: API limits, authorization errors  

  - **Send a text message**  
    - Type: Telegram  
    - Role: Sends the summarized news text to configured Telegram chat  
    - Configuration: Text set dynamically from `Summary` field; disables attribution  
    - Input: After Google Sheets update  
    - Credentials: Telegram Bot Token and Chat ID required  
    - Edge Cases: Telegram API failures, invalid chat ID  

  - **Sticky Note5**  
    - Type: Sticky Note  
    - Role: Explains final data persistence and notification steps  

---

### 3. Summary Table

| Node Name                   | Node Type                  | Functional Role                                     | Input Node(s)                     | Output Node(s)                  | Sticky Note                                                                                                 |
|-----------------------------|----------------------------|----------------------------------------------------|----------------------------------|--------------------------------|-------------------------------------------------------------------------------------------------------------|
| Schedule Trigger1            | Schedule Trigger           | Periodic workflow trigger every 15 minutes         | None                             | NVDA Stock, Stock Buyback, Stock Acquisition | Sticky Note1: Explains scheduling every 15 minutes                                                         |
| NVDA Stock                  | RSS Feed Read              | Fetch stock news feed 1                             | Schedule Trigger1                | Merge                          | Sticky Note: RSS feed explanation                                                                          |
| Stock Buyback               | RSS Feed Read              | Fetch stock news feed 2                             | Schedule Trigger1                | Merge                          | Sticky Note: RSS feed explanation                                                                          |
| Stock Acquisition           | RSS Feed Read              | Fetch stock news feed 3                             | Schedule Trigger1                | Merge                          | Sticky Note: RSS feed explanation                                                                          |
| Merge                      | Merge                     | Combine multiple RSS feed outputs                   | NVDA Stock, Stock Buyback, Stock Acquisition | Get Real URL                   | Sticky Note2: Explains URL extraction and merging                                                          |
| Get Real URL               | Code                      | Extract real article URL from Google redirect links | Merge                           | Append or update row in sheet  | Sticky Note2: URL normalization details                                                                    |
| Append or update row in sheet | Google Sheets             | Log or update article metadata by ID                | Get Real URL                    | Get row(s) in sheet            |                                                                                                             |
| Get row(s) in sheet        | Google Sheets             | Retrieve sheet rows for deduplication check          | Append or update row in sheet   | If                            |                                                                                                             |
| If                         | If                        | Check if article summary is empty                     | Get row(s) in sheet             | Read URL content (true), No Operation (false) | Sticky Note3: Explains content extraction pre-check                                                        |
| Read URL content           | Jina AI                   | Extract main article content                          | If (true branch)                | AI Agent                      | Sticky Note3: Content extraction explanation                                                               |
| No Operation, do nothing   | No Operation              | Skip content extraction                               | If (false branch)               | None                          |                                                                                                             |
| AI Agent                   | Langchain Agent           | Summarize and analyze article using LLM              | Read URL content                | Append or update row in sheet1 | Sticky Note4: Explains summarization via OpenRouter                                                        |
| OpenRouter Chat Model      | Langchain OpenRouter Model | Provides LLM model (Gemini 2.0)                       | AI Agent (ai_languageModel input) | AI Agent (ai_languageModel output) | Sticky Note4: Model usage details                                                                          |
| Append or update row in sheet1 | Google Sheets             | Store summary, mark as sent to Telegram               | AI Agent                       | Send a text message           | Sticky Note5: Explains saving to sheets and Telegram sending                                              |
| Send a text message        | Telegram                  | Send summarized news to Telegram chat                 | Append or update row in sheet1  | None                          | Sticky Note5: Telegram notification explanation                                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Set interval to run every 15 minutes  
   - This node starts the workflow periodically  

2. **Create three RSS Feed Read nodes**  
   - Names: `NVDA Stock`, `Stock Buyback`, `Stock Acquisition`  
   - Configure each with a Google Alerts RSS URL for relevant stock keywords or companies  
   - Connect the output of Schedule Trigger to all three RSS feed nodes  

3. **Create a Merge node**  
   - Set number of inputs to 3  
   - Connect outputs of all three RSS feed nodes to this Merge node  

4. **Create a Code node named `Get Real URL`**  
   - Set to run once per item  
   - Paste JavaScript to extract real URL from Google redirect links:  
     ```js
     return {
       realUrl: $json.link.includes("google.com/url")
         ? decodeURIComponent($json.link.match(/url=([^&]+)/)[1])
         : $json.link
     };
     ```  
   - Connect output of Merge node to this node  

5. **Create a Google Sheets node `Append or update row in sheet`**  
   - Operation: Append or update  
   - Document ID: Set your Google Sheet ID (use the provided template or custom)  
   - Sheet Name: Use the default or target sheet (e.g., gid=0)  
   - Configure columns with mapping:  
     - ID: `={{ $('Merge').item.json.id }}`  
     - URL: `={{ $json.realUrl }}`  
     - Date: `={{ $('Merge').item.json.pubDate }}`  
     - Title: `={{ $('Merge').item.json.title }}`  
   - Matching column: `ID` (to avoid duplicates)  
   - Connect output of `Get Real URL` to this node  
   - Add Google Sheets credentials  

6. **Create a Google Sheets node `Get row(s) in sheet`**  
   - Operation: Read from same document and sheet as above  
   - Connect output of the previous Google Sheets node to this node  

7. **Create an If node**  
   - Condition: Check if `Summary` field is empty string (`={{ $json.Summary }}` equals `""`)  
   - Connect output of `Get row(s) in sheet` to this node  

8. **Create a Jina AI node `Read URL content`**  
   - Set URL parameter dynamically: `={{ $('Get row(s) in sheet').item.json.URL }}`  
   - Connect the `true` output branch of If node to this node  
   - Add Jina AI credentials  

9. **Create a No Operation node `No Operation, do nothing`**  
   - Connect the `false` output branch of If node here  

10. **Create a Langchain OpenRouter Chat Model node**  
    - Select model: `google/gemini-2.0-flash-exp:free`  
    - Add OpenRouter credentials  

11. **Create a Langchain Agent node `AI Agent`**  
    - Text prompt:  
      ```
      Create a summary of up to 200 words from {{ $json.content }} Add a description of the affected stocks (stock code). Add a sentiment description (uptrend, downtrend, sideways, positive, negative or others). The writing format is as follows: Title\nCategory\nSummary\nSentiment\nReccomendation. Write in plain text without punctuation, use emoji to more interactive.
      ```  
    - System message:  
      ```
      You are a professional analyst who has studied a lot about stock movements. Use the following news articles to see the company's fundamentals, sentiment, and future projections.
      ```  
    - Connect output of `Read URL content` to this node  
    - Link the OpenRouter Chat Model node as the language model for this agent  

12. **Create a Google Sheets node `Append or update row in sheet1`**  
    - Operation: Append or update  
    - Matching column: `URL`  
    - Columns mapped:  
      - URL: `={{ $('Read URL content').item.json.url }}`  
      - Summary: `={{ $json.output }}` (from AI Agent)  
      - SentToTelegram: `"true"` (hardcoded)  
    - Connect output of AI Agent to this node  
    - Use same Google Sheets document and sheet as before  

13. **Create a Telegram node `Send a text message`**  
    - Message text: `={{ $json.Summary }}`  
    - Disable attribution  
    - Connect output of previous Google Sheets node to this node  
    - Configure with Telegram Bot Token and your Chat ID  

14. **Connect the `No Operation` node to end (no further action)**  

15. **Double-check all credentials:**  
    - Google Sheets (read and write)  
    - OpenRouter API key  
    - Jina AI API key  
    - Telegram Bot Token and Chat ID  

16. **Test the workflow:**  
    - Manually trigger or wait for scheduled run  
    - Verify that RSS feeds are read, URLs normalized, content extracted, summarized, saved, and Telegram message sent  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Context or Link                                                                                                                             |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------|
| This workflow is designed to integrate Google Alerts RSS feeds with AI summarization and Telegram notifications for real-time stock news alerts.                                                                                                                                                                                                                                                                                                                                                 | General Workflow Purpose                                                                                                                    |
| Google Sheets template recommended for logging news: [Stock Alert Google Sheet](https://docs.google.com/spreadsheets/d/109kj97ABR37XviIpxARCFviZwq8opOoe--rayOeFDSo/edit?usp=sharing)                                                                                                                                                                                                                                                                                                               | Google Sheets Template                                                                                                                      |
| Required credentials include Telegram Bot token and Chat ID, OpenRouter API key, Jina AI API key, and Google Sheets credentials for API access. All sensitive data must be added through n8n credentials manager before activation.                                                                                                                                                                                                                                                                | Security Note                                                                                                                               |
| Google Alerts RSS URLs must be created manually by the user with keywords relevant to stocks (e.g., IPO, company mergers, stock symbols like AAPL, NVDA).                                                                                                                                                                                                                                                                                                                                          | RSS Feed Setup                                                                                                                              |
| The AI prompt is customizable. Adjust system and user messages in the Langchain Agent node to tailor the summarization style, tone, and content.                                                                                                                                                                                                                                                                                                                                                   | AI Prompt Customization                                                                                                                     |
| The URL extraction logic assumes Google Alerts RSS provides redirect URLs in the format `https://www.google.com/url?...`. The extraction uses a regex to parse the real target URL from the `url` query parameter. Changes in Google’s URL format may break this parsing.                                                                                                                                                                                                                       | URL Extraction Logic                                                                                                                        |
| Telegram notifications are sent without attribution to keep messages clean. Ensure Telegram Bot is added to your chat and has permission to post.                                                                                                                                                                                                                                                                                                                                                 | Telegram Bot Setup                                                                                                                          |
| This workflow uses the OpenRouter platform to access various LLMs including Gemini 2.0, GPT-4, and Claude. Model selection can be changed in the OpenRouter Chat Model node.                                                                                                                                                                                                                                                                                                                      | OpenRouter & LLM Models                                                                                                                     |
| The workflow handles duplicate articles by matching on unique article IDs and URLs stored in Google Sheets, preventing repeated alerts for the same news item.                                                                                                                                                                                                                                                                                                                                  | Deduplication Strategy                                                                                                                      |

---

**Disclaimer:** The provided content is solely derived from an automated n8n workflow. It complies fully with all relevant content policies and contains no illegal, offensive, or protected material. All data processed is legal and publicly available.