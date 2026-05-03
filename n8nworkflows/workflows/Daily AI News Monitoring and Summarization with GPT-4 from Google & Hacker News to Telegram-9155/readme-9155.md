Daily AI News Monitoring and Summarization with GPT-4 from Google & Hacker News to Telegram

https://n8nworkflows.xyz/workflows/daily-ai-news-monitoring-and-summarization-with-gpt-4-from-google---hacker-news-to-telegram-9155


# Daily AI News Monitoring and Summarization with GPT-4 from Google & Hacker News to Telegram

### 1. Workflow Overview

This workflow automates daily monitoring, summarization, and delivery of AI-related news from two prominent sources—Google News and Hacker News—to a Telegram channel or chat. It is designed for users interested in receiving a concise, curated daily digest of significant AI news highlights.

**Use Cases:**  
- AI enthusiasts or professionals who want an automated daily briefing on AI news.  
- Teams or communities sharing relevant AI developments without manual curation.  
- Integration of multiple news sources into a unified, concise summary.  

**Logical Blocks:**  
- **1.1 Scheduled Trigger:** Initiates the workflow daily at 8:00 AM.  
- **1.2 Data Acquisition:** Fetches AI-related news items from Google News RSS and Hacker News RSS feeds.  
- **1.3 Data Filtering & Structuring:** Limits Google News results to 10 items, filters relevant AI news, and consolidates both sources into a structured object.  
- **1.4 AI Summarization:** Uses OpenAI GPT-4 via LangChain integration to generate a concise, professional digest of the news.  
- **1.5 Distribution:** Sends the generated digest text to a Telegram channel or chat.  
- **1.6 Documentation:** Provides a sticky note with quick reference information about the workflow.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
  Triggers the workflow execution once daily at 8:00 AM based on a cron schedule.

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**  
  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Configuration: Cron expression `0 8 * * *` (daily at 8:00 AM)  
    - Inputs: None (start node)  
    - Outputs: Triggers Google News AI and Hacker News AI nodes simultaneously  
    - Edge Cases: If n8n server is down or paused at scheduled time, trigger will be missed; ensure system uptime.  
    - Version: Compatible with n8n v1.2+  

#### 1.2 Data Acquisition

- **Overview:**  
  Fetches AI-related news articles from Google News and Hacker News RSS feeds.

- **Nodes Involved:**  
  - Google News AI  
  - Hacker News AI

- **Node Details:**  
  - **Google News AI**  
    - Type: RSS Feed Read  
    - Configuration: RSS URL for Google News AI topic filtered for US English (`https://news.google.com/rss/topics/CAAqIAgKIhpDQkFTRFFvSEwyMHZNRzFyZWhJQ1pXNG9BQVAB?hl=en-US&gl=US&ceid=US:en`)  
    - Inputs: Trigger from Schedule Trigger  
    - Outputs: Feeds into "Limit Google to 10" node  
    - Edge Cases: RSS feed could be unavailable or slow; handle empty or malformed feed gracefully.  
    - Version: RSS Feed Read v1.2  

  - **Hacker News AI**  
    - Type: RSS Feed Read  
    - Configuration: RSS URL querying Hacker News for newest AI items with at least 10 points, limited to 10 articles (`https://hnrss.org/newest?q=ai&points=10&count=10`)  
    - Inputs: Trigger from Schedule Trigger  
    - Outputs: Feeds into "Create News Object" node  
    - Edge Cases: RSS feed downtime or no qualifying articles; handle empty results.  
    - Version: RSS Feed Read v1.2  

#### 1.3 Data Filtering & Structuring

- **Overview:**  
  Limits Google News results to 10 items, filters both news sources for AI relevance, and creates a unified JSON object containing arrays of Google News and Hacker News articles.

- **Nodes Involved:**  
  - Limit Google to 10  
  - Create News Object

- **Node Details:**  
  - **Limit Google to 10**  
    - Type: Limit  
    - Configuration: Maximum of 10 items passed through  
    - Inputs: Google News AI  
    - Outputs: Connects to Create News Object  
    - Edge Cases: If fewer than 10 items fetched, passes all available.  
    - Version: Limit v1  

  - **Create News Object**  
    - Type: Code (JavaScript)  
    - Configuration:  
      - Filters input data arrays for items whose titles include "AI" (case-sensitive).  
      - Creates single output JSON with two arrays: `googleNews` and `hackerNews`.  
      - Uses `$input.all()` to access all input data from both RSS nodes combined.  
    - Inputs:  
      - From Limit Google to 10 (Google News filtered)  
      - From Hacker News AI (raw)  
    - Outputs: Single JSON object consolidating filtered news from both sources  
    - Edge Cases: If either source returns no items with "AI" in title, that array will be empty; code assumes title field exists.  
    - Version: Code Node v2  

#### 1.4 AI Summarization

- **Overview:**  
  Sends consolidated news data to an AI language model (GPT-4) via LangChain to generate a concise, professional daily digest in Markdown format suitable for Telegram.

- **Nodes Involved:**  
  - AI News Summarizer  
  - OpenAI Chat Model

- **Node Details:**  
  - **OpenAI Chat Model**  
    - Type: LangChain LM Chat OpenAI  
    - Configuration: Uses GPT-4 mini model variant (`gpt-4.1-mini`)  
    - Credentials: OpenAI API key configured  
    - Inputs: AI News Summarizer node uses this as a language model backend  
    - Outputs: Feeds back to AI News Summarizer for prompt processing  
    - Edge Cases: API rate limits, network timeout, or invalid credentials may cause failure.  
    - Version: LangChain node v1.2  

  - **AI News Summarizer**  
    - Type: LangChain Chain LLM  
    - Configuration:  
      - Prompt instructs the AI to generate a Markdown-formatted digest with two sections: Google News and Hacker News.  
      - Summarization constraints: max 4000 characters, professional tone, bullet points with title and summary, date heading from article dates.  
      - Uses expression `{{ $json.toJsonString() }}` to inject consolidated news JSON into the prompt.  
    - Inputs: Output of Create News Object (news JSON)  
    - Outputs: Formatted digest text passed to "Send to Telegram" node  
    - Edge Cases: Large inputs may exceed token limits; prompt relies on well-structured input JSON.  
    - Version: LangChain Chain LLM v1  

#### 1.5 Distribution

- **Overview:**  
  Sends the generated AI news digest text to a Telegram channel or chat configured with a Telegram Bot API.

- **Nodes Involved:**  
  - Send to Telegram

- **Node Details:**  
  - **Send to Telegram**  
    - Type: Telegram  
    - Configuration:  
      - Text message prefixed with "🤖 AI News Daily Digest" followed by the digest content from the summarizer.  
      - No attribution appended automatically.  
    - Credentials: Telegram API credentials configured with bot token  
    - Inputs: Output from AI News Summarizer node  
    - Outputs: None (terminal node)  
    - Edge Cases: Telegram API rate limits, invalid token, or network errors may cause message delivery failure.  
    - Version: Telegram node v1.2  

#### 1.6 Documentation

- **Overview:**  
  Provides a sticky note with a quick reference summary of the workflow’s schedule, data sources, AI technology, and delivery method.  

- **Nodes Involved:**  
  - Quick Reference

- **Node Details:**  
  - **Quick Reference**  
    - Type: Sticky Note  
    - Configuration: Contains key summary points about schedule, sources, AI usage, and delivery channel  
    - Inputs/Outputs: None (documentation only)  
    - Edge Cases: None  
    - Version: Sticky Note v1  

---

### 3. Summary Table

| Node Name          | Node Type                         | Functional Role               | Input Node(s)          | Output Node(s)        | Sticky Note                                                                                   |
|--------------------|----------------------------------|------------------------------|-----------------------|-----------------------|-----------------------------------------------------------------------------------------------|
| Schedule Trigger    | Schedule Trigger                 | Initiate daily workflow       | -                     | Google News AI, Hacker News AI | 📌 AI News Daily Monitor - Quick Reference<br>🕐 SCHEDULE: Daily at 8:00 AM (adjustable)<br>📰 SOURCES: Google News RSS + Hacker News<br>🤖 AI: OpenAI for filtering & summarization<br>📱 DELIVERY: Telegram channel/chat |
| Google News AI      | RSS Feed Read                   | Fetch AI news from Google RSS | Schedule Trigger      | Limit Google to 10    |                                                                                               |
| Limit Google to 10  | Limit                          | Restrict Google news to 10 items | Google News AI        | Create News Object    |                                                                                               |
| Hacker News AI      | RSS Feed Read                   | Fetch AI news from Hacker News RSS | Schedule Trigger      | Create News Object    |                                                                                               |
| Create News Object  | Code (JavaScript)               | Filter and consolidate news   | Limit Google to 10, Hacker News AI | AI News Summarizer   |                                                                                               |
| OpenAI Chat Model   | LangChain LM Chat OpenAI        | GPT-4 model backend for summarization | AI News Summarizer (ai_languageModel input) | AI News Summarizer (output) |                                                                                               |
| AI News Summarizer  | LangChain Chain LLM             | Generate digest text          | Create News Object    | Send to Telegram      |                                                                                               |
| Send to Telegram    | Telegram                       | Send digest to Telegram chat  | AI News Summarizer    | -                     |                                                                                               |
| Quick Reference     | Sticky Note                    | Documentation summary         | -                     | -                     | 📌 AI News Daily Monitor - Quick Reference<br>🕐 SCHEDULE: Daily at 8:00 AM (adjustable)<br>📰 SOURCES: Google News RSS + Hacker News<br>🤖 AI: OpenAI for filtering & summarization<br>📱 DELIVERY: Telegram channel/chat |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n** and name it "AI News Daily Monitor".

2. **Add a Schedule Trigger node:**  
   - Set type to "Schedule Trigger".  
   - Configure cron expression to `0 8 * * *` (daily at 8:00 AM).  
   - This node will start the workflow daily.

3. **Add two RSS Feed Read nodes:**  
   - Name one "Google News AI".  
     - Set RSS URL to:  
       `https://news.google.com/rss/topics/CAAqIAgKIhpDQkFTRFFvSEwyMHZNRzFyZWhJQ1pXNG9BQVAB?hl=en-US&gl=US&ceid=US:en`  
     - Leave options default.  
   - Name the other "Hacker News AI".  
     - Set RSS URL to:  
       `https://hnrss.org/newest?q=ai&points=10&count=10`  
     - Leave options default.

4. **Connect Schedule Trigger output to both "Google News AI" and "Hacker News AI" nodes.**

5. **Add a Limit node:**  
   - Name it "Limit Google to 10".  
   - Set maxItems to 10.  
   - Connect "Google News AI" output to this node.

6. **Add a Code node:**  
   - Name it "Create News Object".  
   - Set language to JavaScript.  
   - Paste the following code:

   ```javascript
   const googleNews = $input.all().filter(item => item.json.title && item.json.title.includes('AI'));
   const hackerNews = $input.all().filter(item => item.json.title && item.json.title.includes('AI'));

   return [{
     json: {
       googleNews: googleNews.map(item => item.json),
       hackerNews: hackerNews.map(item => item.json)
     }
   }];
   ```

   - Connect "Limit Google to 10" and "Hacker News AI" outputs to this node (multi-input).

7. **Add an OpenAI Chat Model node via LangChain:**  
   - Name it "OpenAI Chat Model".  
   - Select model as `gpt-4.1-mini`.  
   - Configure OpenAI API credentials (create and add your OpenAI API key).  
   - Leave options default.

8. **Add a LangChain Chain LLM node:**  
   - Name it "AI News Summarizer".  
   - Use "OpenAI Chat Model" as the language model input.  
   - Paste this prompt in the prompt field (use expression mode):  

   ```
   =You are an AI news analyst. Summarize the following AI news from Google News and Hacker News into a concise daily digest. Only include articles that are interesting, important, or notable—skip irrelevant or trivial items.

   Instructions:

   Begin the digest with a Markdown heading:

   # AI News Daily Digest – [date]

   (use the most relevant date from the articles).  
   Do not include any filler text before the heading.

   Split the digest into two main sections, each as a Markdown subheading:

   ## Google News  
   ## Hacker News

   Within each section, list the top relevant articles using Markdown bullet points. For each article, include:

   - **Title:** [Article Title]  
     **Summary:** [Short professional summary of key points/insights]

   Maintain a neutral, professional, concise tone highlighting trends, debates, or important developments.

   Important: Ensure the entire digest does not exceed 4,000 characters. If needed, prioritize the most interesting/high-impact articles and omit less important items so the digest stays within the limit.

   News to summarize:  
   {{ $json.toJsonString() }}
   ```

   - Connect "Create News Object" output to this node.

9. **Add a Telegram node:**  
   - Name it "Send to Telegram".  
   - Configure the Telegram API credentials with your bot's token.  
   - Set the message text to:  

   ```
   =🤖 AI News Daily Digest

   {{ $json.text }}
   ```

   - Connect "AI News Summarizer" output to this node.

10. **(Optional) Add a Sticky Note node for documentation:**  
    - Name it "Quick Reference".  
    - Add content:  

    ```
    📌 AI News Daily Monitor - Quick Reference

    🕐 SCHEDULE: Daily at 8:00 AM (adjustable)  
    📰 SOURCES: Google News RSS + Hacker News  
    🤖 AI: OpenAI for filtering & summarization  
    📱 DELIVERY: Telegram channel/chat
    ```

11. **Set workflow settings:**  
    - Enable saving of manual executions and execution progress.  
    - Set execution order to "v1".  
    - Configure data saving on error and success as needed.

12. **Activate workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                  |
|-------------------------------------------------------------------------------------------------|-----------------------------------------------------------------|
| Workflow is designed for daily execution at 8:00 AM but the schedule is configurable as needed. | Cron scheduling in Schedule Trigger node                        |
| Uses OpenAI GPT-4 (mini variant) with LangChain integration for advanced NLP summarization.     | Requires OpenAI API credentials configured in n8n               |
| Telegram Bot API requires prior setup of a bot and obtaining a token via BotFather.             | Telegram node credentials                                        |
| RSS feeds URLs for Google News and Hacker News are public and filter for AI-related content.    | Google News RSS and Hacker News RSS URLs configured in nodes    |
| Sticky note provides quick workflow context for users or maintainers.                           | Helps in understanding workflow at a glance                     |
| Potential failure points include RSS feed downtime, API rate limiting, and network issues.      | Recommended to monitor executions and set retry mechanisms      |

---

**Disclaimer:** The text provided is exclusively from an automated workflow created with n8n, an integration and automation tool. The processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.