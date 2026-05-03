Daily News Digest & Weekly Trends with AI Filtering, Slack & Google Sheets

https://n8nworkflows.xyz/workflows/daily-news-digest---weekly-trends-with-ai-filtering--slack---google-sheets-10977


# Daily News Digest & Weekly Trends with AI Filtering, Slack & Google Sheets

### 1. Workflow Overview

This workflow automates the daily and weekly processing of news articles using AI filtering, summarization, translation, and dissemination via Slack and Google Sheets. It is designed for users who want to receive a curated daily news digest on specific topics, with additional weekly trend reports generated from accumulated data.

**Target Use Cases:**
- Daily retrieval and filtering of news articles on a chosen keyword.
- AI-based quality filtering to remove clickbait or low-quality content.
- Summarization and formatting of top articles for Slack distribution.
- Translation of summaries to Japanese.
- Archiving structured article data in Google Sheets.
- Weekly trend analysis from stored data with AI-generated insights.

**Logical Blocks:**

- **1.1 Input Reception and Scheduling**  
  Receives manual triggers via webhook or scheduled triggers via cron jobs to initiate news fetching.

- **1.2 News Retrieval and AI Filtering**  
  Fetches articles from NewsAPI, then filters them using an AI agent to remove low-quality content.

- **1.3 Article Structuring and Archiving**  
  Structures filtered articles into discrete items, stores them in Google Sheets.

- **1.4 Daily Digest Preparation and Slack Messaging**  
  Summarizes top three articles in English, translates to Japanese, and sends both versions to Slack.

- **1.5 Weekly Trend Report Generation**  
  Weekly cron trigger reads stored data, AI analyzes trends and insights, then sends a summary to Slack.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Scheduling

**Overview:**  
This block initiates workflow execution either by a webhook receiving manual requests or by scheduled triggers to automate daily or weekly runs.

**Nodes Involved:**  
- Webhook  
- Cron Daily Digest  
- Cron Weekly Report

**Node Details:**

- **Webhook**  
  - Type: Webhook  
  - Role: Receives manual HTTP POST requests at path `/iphone-news` to trigger the workflow immediately.  
  - Config: Path set to "iphone-news", no authentication or additional options.  
  - Connections: Output → Set Keyword  
  - Edge Cases: Possible HTTP request failures or invalid payloads; no explicit validation shown.

- **Cron Daily Digest**  
  - Type: Cron Trigger  
  - Role: Automatically triggers the workflow daily at 08:00 AM to generate the daily news digest.  
  - Config: Trigger time set to hour 8 (8 AM), no timezone specified (defaults to server timezone).  
  - Connections: Output → Set Keyword  
  - Edge Cases: Timezone mismatches can cause unexpected trigger times.

- **Cron Weekly Report**  
  - Type: Cron Trigger  
  - Role: Triggers weekly on Mondays at 09:00 AM to generate the weekly trend report.  
  - Config: Weekly mode, hour 9, weekday unspecified in JSON but implied Monday (may require explicit setting).  
  - Connections: Output → Read sheet (weekly)  
  - Edge Cases: Incorrect weekday setting may cause no trigger; time zone considerations apply.

---

#### 2.2 News Retrieval and AI Filtering

**Overview:**  
Fetches news articles based on a dynamic keyword, then uses an AI agent to filter out low-quality or clickbait articles, preserving relevant and trustworthy content.

**Nodes Involved:**  
- Set Keyword  
- Get News  
- AI Agent (Filter)  
- OpenRouter Chat Model

**Node Details:**

- **Set Keyword**  
  - Type: Set  
  - Role: Defines the search keyword for news retrieval; defaults to "technology" if no input provided.  
  - Config: Sets `chatInput` value from webhook JSON or defaults to "technology". Keeps only this key in output.  
  - Connections: Output → Get News  
  - Edge Cases: Empty or invalid keyword input could lead to poor or no results.

- **Get News**  
  - Type: HTTP Request  
  - Role: Calls NewsAPI's `/v2/everything` endpoint to fetch news articles matching the keyword.  
  - Config:  
    - URL: https://newsapi.org/v2/everything  
    - Query Parameters:  
      - `q` = `chatInput` value  
      - `language` = "en"  
      - `sortBy` = "publishedAt" (latest first)  
      - `pageSize` = 10 articles  
    - Header: API key passed via `X-Api-Key` header (credential required).  
  - Connections: Output → AI Agent (Filter)  
  - Edge Cases: API key invalid or rate limits reached; HTTP errors; empty result sets.

- **AI Agent (Filter)**  
  - Type: LangChain Agent  
  - Role: Processes the raw articles JSON to filter out low-quality or clickbait articles based on system instructions.  
  - Config:  
    - Input Text: JSON stringified articles from Get News node.  
    - System Message: Instructions to filter and keep trustworthy, relevant articles with key details (Title, URL, Author).  
    - Uses OpenRouter Chat Model as backend language model.  
  - Connections: Output → AI Agent (Structure), AI Agent (Slack)  
  - Edge Cases: AI may misclassify articles; token limits for large JSON; API errors.

- **OpenRouter Chat Model**  
  - Type: LangChain Language Model (OpenRouter)  
  - Role: Provides the AI backend for the AI Agent (Filter) node processing.  
  - Config: Uses default OpenRouter credentials.  
  - Connections: AI Agent (Filter) → this node.  
  - Edge Cases: Authentication issues; latency; rate limits.

---

#### 2.3 Article Structuring and Archiving

**Overview:**  
Structures the AI-filtered articles into an array of individual items, then appends each as a row in a Google Sheet for archiving and further analysis.

**Nodes Involved:**  
- AI Agent (Structure)  
- OpenRouter Chat Model1  
- Structured Output Parser  
- Split Articles  
- Append row in sheet

**Node Details:**

- **AI Agent (Structure)**  
  - Type: LangChain Agent  
  - Role: From the filtered articles text, selects the top 3 articles and returns structured JSON with title, author, summary, and URL.  
  - Config:  
    - Input text: output from AI Agent (Filter).  
    - Prompt instructs to return an array named "articles" with requested fields, strictly as valid JSON.  
    - Output parser enabled with Structured Output Parser node.  
    - Uses OpenRouter Chat Model1 as backend.  
  - Connections: Output → Structured Output Parser, AI Agent (Slack)  
  - Edge Cases: AI output formatting errors; parsing failures if output is malformed.

- **OpenRouter Chat Model1**  
  - Type: LangChain Language Model (OpenRouter)  
  - Role: Backend AI model for AI Agent (Structure).  
  - Config: Default OpenRouter credentials.  
  - Connections: AI Agent (Structure) → this node.

- **Structured Output Parser**  
  - Type: LangChain Output Parser  
  - Role: Ensures the AI Agent (Structure) outputs valid JSON matching the schema of articles array with title, author, summary, URL.  
  - Config: JSON schema example provided for validation.  
  - Connections: Output → AI Agent (Structure)  
  - Edge Cases: Parser errors on malformed JSON.

- **Split Articles**  
  - Type: Item Lists  
  - Role: Splits the "articles" array into individual items for further processing.  
  - Config: Splitting by field "articles".  
  - Connections: Output → Append row in sheet  
  - Edge Cases: Empty or missing "articles" field.

- **Append row in sheet**  
  - Type: Google Sheets  
  - Role: Appends each article as a new row into a Google Sheet for data archiving.  
  - Config:  
    - Operation: append  
    - Document ID and Sheet Name must be configured to target correct sheet with columns: title, author, summary, url.  
  - Credentials: Google Sheets OAuth2.  
  - Connections: Input from Split Articles.  
  - Edge Cases: Authentication failures; incorrect sheet or column mapping; API quota limits.

---

#### 2.4 Daily Digest Preparation and Slack Messaging

**Overview:**  
Generates a readable summary of the top 3 articles formatted for Slack, translates it to Japanese, and posts both English and Japanese versions to specified Slack channels.

**Nodes Involved:**  
- AI Agent (Slack)  
- OpenRouter Chat Model2  
- Send English Message  
- Translate to Japanese  
- Send Japanese Message

**Node Details:**

- **AI Agent (Slack)**  
  - Type: LangChain Agent  
  - Role: Converts structured article data into a Slack-friendly markdown summary of all 3 articles.  
  - Config:  
    - Input text: output from AI Agent (Structure).  
    - System message instructs formatting with titles, authors, summaries, URLs, markdown, no code blocks.  
    - Uses OpenRouter Chat Model2 as backend.  
  - Connections: Output → Send English Message, Translate to Japanese  
  - Edge Cases: AI formatting errors; token limits.

- **OpenRouter Chat Model2**  
  - Type: LangChain Language Model (OpenRouter)  
  - Role: Backend AI model for AI Agent (Slack).  
  - Connections: AI Agent (Slack) → this node.

- **Send English Message**  
  - Type: Slack  
  - Role: Posts the English summary message to a configured Slack channel.  
  - Config:  
    - Text: output from AI Agent (Slack).  
    - Channel ID must be set to target channel.  
    - Authentication via OAuth2.  
  - Edge Cases: Slack API rate limits; invalid channel ID; auth token expiry.

- **Translate to Japanese**  
  - Type: DeepL  
  - Role: Translates the English summary into Japanese.  
  - Config:  
    - Text: output from AI Agent (Slack).  
    - Target language: Japanese (JA).  
  - Edge Cases: API key issues; untranslated or partial results.

- **Send Japanese Message**  
  - Type: Slack  
  - Role: Posts the translated Japanese message to Slack channel (could be same or different channel).  
  - Config:  
    - Text: translatedText from DeepL node.  
    - Channel ID required.  
    - OAuth2 authentication.  
  - Edge Cases: Slack API errors, translation missing.

---

#### 2.5 Weekly Trend Report Generation

**Overview:**  
Triggered weekly, this block reads the archived article data from Google Sheets, uses AI to analyze weekly trends, key insights, and suggests team actions, then sends the summary to Slack.

**Nodes Involved:**  
- Cron Weekly Report  
- Read sheet (weekly)  
- AI Agent Weekly Report  
- OpenRouter Weekly  
- Send Weekly Summary

**Node Details:**

- **Cron Weekly Report**  
  - Already described in block 2.1.

- **Read sheet (weekly)**  
  - Type: Google Sheets  
  - Role: Reads accumulated article data from the configured Google Sheet for the past week.  
  - Config:  
    - Document ID and Sheet Name must match those in Append row in sheet node.  
  - Credentials: Google Sheets OAuth2.  
  - Connections: Output → AI Agent Weekly Report  
  - Edge Cases: Reading incorrect or empty sheet; auth errors.

- **AI Agent Weekly Report**  
  - Type: LangChain Agent  
  - Role: Analyzes the weekly articles to generate a trend report including top topics, key insights, and suggested actions in Slack markdown format.  
  - Config:  
    - Input text: JSON stringified sheet data.  
    - System message provides detailed instructions for trend analysis and formatting.  
    - Uses OpenRouter Weekly as backend.  
  - Connections: Output → Send Weekly Summary

- **OpenRouter Weekly**  
  - Type: LangChain Language Model (OpenRouter)  
  - Role: Backend AI model for weekly report agent.

- **Send Weekly Summary**  
  - Type: Slack  
  - Role: Sends the AI-generated weekly trend report to the configured Slack channel.  
  - Config:  
    - Text: output from AI Agent Weekly Report node.  
    - Channel ID configured.  
    - OAuth2 authentication.  
  - Edge Cases: Slack API errors; empty report if no data.

---

### 3. Summary Table

| Node Name             | Node Type                         | Functional Role                          | Input Node(s)            | Output Node(s)                            | Sticky Note                                                                                                     |
|-----------------------|----------------------------------|----------------------------------------|--------------------------|------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Webhook               | Webhook                          | Manual trigger input                    | —                        | Set Keyword                              |                                                                                                                 |
| Cron Daily Digest      | Cron                             | Daily scheduled trigger                 | —                        | Set Keyword                              |                                                                                                                 |
| Set Keyword            | Set                              | Defines search keyword for news query  | Webhook, Cron Daily Digest| Get News                                |                                                                                                                 |
| Get News              | HTTP Request                     | Fetches news articles from NewsAPI     | Set Keyword               | AI Agent (Filter)                        |                                                                                                                 |
| AI Agent (Filter)      | LangChain Agent                  | Filters out clickbait/low-quality news | Get News                  | AI Agent (Structure), AI Agent (Slack) |                                                                                                                 |
| OpenRouter Chat Model  | LangChain LM                    | AI backend for filtering agent         | AI Agent (Filter)         | AI Agent (Filter)                        |                                                                                                                 |
| AI Agent (Structure)   | LangChain Agent                  | Structures top 3 articles into JSON    | AI Agent (Filter)         | Structured Output Parser, AI Agent (Slack) |                                                                                                                 |
| OpenRouter Chat Model1 | LangChain LM                    | AI backend for structuring agent       | AI Agent (Structure)      | AI Agent (Structure)                     |                                                                                                                 |
| Structured Output Parser| LangChain Output Parser         | Validates structured JSON output       | AI Agent (Structure)      | AI Agent (Structure)                     |                                                                                                                 |
| Split Articles         | Item Lists                      | Splits articles array into individual items | AI Agent (Structure)      | Append row in sheet                      |                                                                                                                 |
| Append row in sheet    | Google Sheets                   | Archives articles into Google Sheets   | Split Articles            | —                                        |                                                                                                                 |
| AI Agent (Slack)       | LangChain Agent                  | Creates Slack formatted daily summary  | AI Agent (Structure)      | Send English Message, Translate to Japanese |                                                                                                                 |
| OpenRouter Chat Model2 | LangChain LM                    | AI backend for Slack formatting agent  | AI Agent (Slack)          | AI Agent (Slack)                         |                                                                                                                 |
| Send English Message   | Slack                          | Posts English summary to Slack channel | AI Agent (Slack)          | —                                        |                                                                                                                 |
| Translate to Japanese  | DeepL                          | Translates summary to Japanese          | AI Agent (Slack)          | Send Japanese Message                    |                                                                                                                 |
| Send Japanese Message  | Slack                          | Posts Japanese summary to Slack channel | Translate to Japanese     | —                                        |                                                                                                                 |
| Cron Weekly Report     | Cron                           | Weekly scheduled trigger                | —                        | Read sheet (weekly)                      |                                                                                                                 |
| Read sheet (weekly)    | Google Sheets                  | Reads past week’s data from Google Sheets | Cron Weekly Report       | AI Agent Weekly Report                   |                                                                                                                 |
| AI Agent Weekly Report | LangChain Agent                | Generates weekly trend report           | Read sheet (weekly)       | Send Weekly Summary                      |                                                                                                                 |
| OpenRouter Weekly      | LangChain LM                  | AI backend for weekly report agent     | AI Agent Weekly Report    | AI Agent Weekly Report                   |                                                                                                                 |
| Send Weekly Summary    | Slack                         | Posts weekly trend report to Slack     | AI Agent Weekly Report    | —                                        |                                                                                                                 |
| Sticky Note9           | Sticky Note                   | Workflow overview and requirements      | —                        | —                                        | ## What it does\n1. **Fetches News:** Pulls daily articles via NewsAPI based on your chosen keyword (default: "technology").\n2. **AI Filtering:** Uses an AI Agent (via OpenRouter) to filter out low-quality or irrelevant clickbait.\n3. **Daily Digest (Slack):**\n   - Summarizes the top 3 articles in English.\n   - Translates the summaries to Japanese using DeepL (optional).\n   - Posts both versions to a Slack channel.\n4. **Data Archiving (Sheets):** Extracts structured data (Title, Author, Summary, URL) and saves it to Google Sheets.\n5. **Weekly Trend Report:** Every Monday, it reads the past week's data from Google Sheets and uses AI to generate a high-level trend report and strategic insights.\n\n## Requirements\n- **n8n** (Self-hosted or Cloud)\n- **NewsAPI** Key (Free tier available)\n- **OpenRouter** (or any LangChain compatible Chat Model like OpenAI)\n- **DeepL** API Key (for translation)\n- **Google Sheets** account\n- **Slack** Workspace\n |
| Sticky Note10          | Sticky Note                   | Setup instructions                      | —                        | —                                        | ## How to set up\n1. **Configure Credentials:** You will need API keys/auth for NewsAPI, OpenRouter (or OpenAI), DeepL, Google Sheets, and Slack.\n2. **Setup Google Sheet:** Create a sheet with the following headers in the first row: `title`, `author`, `summary`, `url`.\n3. **Map the Sheet:** In the "Append row in sheet" and "Read sheet (weekly)" nodes, select your file and map the columns.\n4. **Define Keyword:** Open the "Set Keyword" node and change `chatInput` to the topic you want to track (e.g., "Crypto", "SaaS", "Climate Change").\n5. **Slack Setup:** Select your desired channel in the Slack nodes.\n |
| Sticky Note            | Sticky Note                   | Weekly report configuration             | —                        | —                                        | ##  Weekly Trend Report Configuration\n\nThis section automates the weekly analysis every Monday at 9:00 AM. It reads the accumulated articles from your Google Sheet and uses AI to identify trends.\n\n**Setup Steps:**\n1. **Google Sheets:** Open the `Read sheet (weekly)` node. Select the **same Document and Sheet** that you configured in the "Append row in sheet" node above. This allows the AI to access the past week's data.\n2. **AI Model:** Ensure the `OpenRouter Weekly` node has your credentials selected.\n3. **Slack:** In the `Send Weekly Summary` node, select the channel where you want the weekly insights report to be posted.\n4. **Schedule:** (Optional) You can change the reporting day and time in the `Cron Weekly Report` node. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Webhook Node**  
   - Type: Webhook  
   - Path: `iphone-news`  
   - Purpose: Manual trigger to start the workflow with optional input keyword.

2. **Create a Cron Node for Daily Digest**  
   - Type: Cron  
   - Trigger Time: Hour 8 (8:00 AM daily)  
   - Purpose: Automatically trigger the daily news digest.

3. **Create a Set Node named "Set Keyword"**  
   - Define a string field `chatInput`  
   - Expression: `{{$json.chatInput || "technology"}}`  
   - Purpose: Set or default the search keyword for news retrieval.  
   - Connect Webhook and Cron Daily Digest outputs to this node.

4. **Create HTTP Request Node "Get News"**  
   - Method: GET  
   - URL: `https://newsapi.org/v2/everything`  
   - Query Parameters:  
     - `q`: expression from `chatInput` field  
     - `language`: "en"  
     - `sortBy`: "publishedAt"  
     - `pageSize`: "10"  
   - Header: `X-Api-Key` with NewsAPI credential  
   - Connect output of Set Keyword node here.

5. **Create LangChain Agent "AI Agent (Filter)"**  
   - Input Text: `{{ JSON.stringify($json.articles) }}` from Get News  
   - System Message: Instructions to filter out low-quality/clickbait, preserve title, URL, author.  
   - Use OpenRouter Chat Model node as AI model backend.

6. **Create LangChain Language Model Node "OpenRouter Chat Model"**  
   - Use OpenRouter credentials  
   - Connect to AI Agent (Filter).

7. **Create LangChain Agent "AI Agent (Structure)"**  
   - Input: `{{$json.output}}` from AI Agent (Filter)  
   - System Message: Instructions to select top 3 articles and return JSON array `articles` with title, author, summary, URL.  
   - Enable output parser with Structured Output Parser node.  
   - Use OpenRouter Chat Model1 as backend.

8. **Create LangChain Language Model Node "OpenRouter Chat Model1"**  
   - Use OpenRouter credentials  
   - Connect to AI Agent (Structure).

9. **Create Structured Output Parser Node "Structured Output Parser"**  
   - JSON Schema Example:  
     ```json
     {
       "articles": [
         {
           "title": "string",
           "author": "string",
           "summary": "string",
           "url": "string"
         }
       ]
     }
     ```  
   - Connect output to AI Agent (Structure).

10. **Create Item Lists Node "Split Articles"**  
    - Field to split: `articles`  
    - Connect AI Agent (Structure) output here.

11. **Create Google Sheets Node "Append row in sheet"**  
    - Operation: Append  
    - Document ID & Sheet Name: configure to target your Google Sheet with headers `title`, `author`, `summary`, `url`  
    - Credentials: Google Sheets OAuth2  
    - Connect Split Articles output here.

12. **Create LangChain Agent "AI Agent (Slack)"**  
    - Input: `{{$json.output}}` from AI Agent (Structure)  
    - System Message: Format each of the 3 articles into Slack-friendly markdown with title, author, summary, URL. No code blocks.  
    - Use OpenRouter Chat Model2 backend.

13. **Create LangChain Language Model Node "OpenRouter Chat Model2"**  
    - Use OpenRouter credentials  
    - Connect to AI Agent (Slack).

14. **Create Slack Node "Send English Message"**  
    - Text: `{{$json.output}}` from AI Agent (Slack)  
    - Channel: set your Slack channel ID  
    - Authentication: OAuth2  
    - Connect AI Agent (Slack) output here.

15. **Create DeepL Node "Translate to Japanese"**  
    - Text: `{{$json.output}}` from AI Agent (Slack)  
    - Target Language: JA (Japanese)  
    - Connect AI Agent (Slack) output here.

16. **Create Slack Node "Send Japanese Message"**  
    - Text: `{{$json.translatedText}}` from DeepL node  
    - Channel: set Slack channel (can be same or different)  
    - Authentication: OAuth2  
    - Connect Translate to Japanese output here.

17. **Create Cron Node "Cron Weekly Report"**  
    - Trigger Time: Weekly on Monday at 9:00 AM  
    - Connect this node to "Read sheet (weekly)".

18. **Create Google Sheets Node "Read sheet (weekly)"**  
    - Operation: Read  
    - Document ID & Sheet Name: same as "Append row in sheet" node  
    - Credentials: Google Sheets OAuth2  
    - Connect output to AI Agent Weekly Report.

19. **Create LangChain Agent "AI Agent Weekly Report"**  
    - Input: `{{ JSON.stringify($json) }}` from Read sheet (weekly)  
    - System Message: Instructions to generate weekly trend report with top 3 topics, key insights, suggested actions, Slack markdown format.  
    - Use OpenRouter Weekly backend.

20. **Create LangChain Language Model Node "OpenRouter Weekly"**  
    - Use OpenRouter credentials  
    - Connect to AI Agent Weekly Report.

21. **Create Slack Node "Send Weekly Summary"**  
    - Text: `{{$json.output}}` from AI Agent Weekly Report  
    - Channel: set Slack channel ID for weekly report  
    - Authentication: OAuth2  
    - Connect AI Agent Weekly Report output here.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | Context or Link                                              |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------|
| ## What it does<br>1. **Fetches News:** Pulls daily articles via NewsAPI based on your chosen keyword (default: "technology").<br>2. **AI Filtering:** Uses an AI Agent (via OpenRouter) to filter out low-quality or irrelevant clickbait.<br>3. **Daily Digest (Slack):**<br>   - Summarizes the top 3 articles in English.<br>   - Translates the summaries to Japanese using DeepL (optional).<br>   - Posts both versions to a Slack channel.<br>4. **Data Archiving (Sheets):** Extracts structured data (Title, Author, Summary, URL) and saves it to Google Sheets.<br>5. **Weekly Trend Report:** Every Monday, it reads the past week's data from Google Sheets and uses AI to generate a high-level trend report and strategic insights.<br><br>## Requirements<br>- **n8n** (Self-hosted or Cloud)<br>- **NewsAPI** Key (Free tier available)<br>- **OpenRouter** (or any LangChain compatible Chat Model like OpenAI)<br>- **DeepL** API Key (for translation)<br>- **Google Sheets** account<br>- **Slack** Workspace | Sticky Note9: Workflow Overview and Requirements             |
| ## How to set up<br>1. **Configure Credentials:** You will need API keys/auth for NewsAPI, OpenRouter (or OpenAI), DeepL, Google Sheets, and Slack.<br>2. **Setup Google Sheet:** Create a sheet with the following headers in the first row: `title`, `author`, `summary`, `url`.<br>3. **Map the Sheet:** In the "Append row in sheet" and "Read sheet (weekly)" nodes, select your file and map the columns.<br>4. **Define Keyword:** Open the "Set Keyword" node and change `chatInput` to the topic you want to track (e.g., "Crypto", "SaaS", "Climate Change").<br>5. **Slack Setup:** Select your desired channel in the Slack nodes.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Sticky Note10: Setup Instructions                             |
| ##  Weekly Trend Report Configuration<br><br>This section automates the weekly analysis every Monday at 9:00 AM. It reads the accumulated articles from your Google Sheet and uses AI to identify trends.<br><br>**Setup Steps:**<br>1. **Google Sheets:** Open the `Read sheet (weekly)` node. Select the **same Document and Sheet** that you configured in the "Append row in sheet" node above. This allows the AI to access the past week's data.<br>2. **AI Model:** Ensure the `OpenRouter Weekly` node has your credentials selected.<br>3. **Slack:** In the `Send Weekly Summary` node, select the channel where you want the weekly insights report to be posted.<br>4. **Schedule:** (Optional) You can change the reporting day and time in the `Cron Weekly Report` node.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | Sticky Note: Weekly Trend Report Configuration                |

---

**Disclaimer:**  
The provided text and workflow are generated from an automated n8n workflow. It complies strictly with content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.