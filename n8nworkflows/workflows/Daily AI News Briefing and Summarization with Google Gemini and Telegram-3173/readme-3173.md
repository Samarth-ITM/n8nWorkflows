Daily AI News Briefing and Summarization with Google Gemini and Telegram

https://n8nworkflows.xyz/workflows/daily-ai-news-briefing-and-summarization-with-google-gemini-and-telegram-3173


# Daily AI News Briefing and Summarization with Google Gemini and Telegram

### 1. Workflow Overview

This n8n workflow automates the daily delivery of personalized AI news summaries directly to a Telegram chat. It targets users interested in staying updated on AI, cryptocurrency, stock market trends, or any customizable topic by aggregating news from two free API sources (GNewsAPI and NewsAPI). The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger & Date Preparation:** Initiates the workflow daily at 6 AM and calculates the date for filtering recent news.
- **1.2 News Retrieval:** Fetches news articles from two distinct free news APIs (GNewsAPI and NewsAPI) based on the prepared date and search queries.
- **1.3 News Extraction & Merging:** Extracts relevant news data from both API responses and merges them into a single dataset.
- **1.4 AI Processing:** Uses Google Gemini Chat Model via an AI Agent node to intelligently filter and summarize the merged news articles.
- **1.5 Telegram Delivery:** Sends the AI-generated daily news briefing directly to the configured Telegram chat.

This structure ensures automated, intelligent, and customizable news delivery with minimal manual intervention.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger & Date Preparation

- **Overview:**  
  This block triggers the workflow automatically every day at 6 AM and computes the date one day prior to the current date. This date is used to filter news articles to ensure freshness.

- **Nodes Involved:**  
  - Trigger workflow at 6am everyday  
  - Substract Current date by one

- **Node Details:**

  - **Trigger workflow at 6am everyday**  
    - *Type:* Schedule Trigger  
    - *Role:* Initiates the workflow daily at 6:00 AM (Europe/Paris timezone)  
    - *Configuration:* Default daily schedule at 6 AM; timezone set to Europe/Paris  
    - *Connections:* Outputs to "Substract Current date by one"  
    - *Edge Cases:* Workflow will not trigger if n8n instance is down or paused; timezone misconfiguration may cause unexpected trigger times.

  - **Substract Current date by one**  
    - *Type:* DateTime  
    - *Role:* Calculates the date one day before the current date to use as a filter parameter for news APIs  
    - *Configuration:* Subtracts 1 day from current date; output format compatible with API query parameters  
    - *Connections:* Outputs to both "News Source: NewsAPI" and "News Source: GNewsAPI" nodes  
    - *Edge Cases:* Date calculation errors if system clock is incorrect; ensure output date format matches API requirements.

#### 2.2 News Retrieval

- **Overview:**  
  This block fetches news articles from two free news APIs (NewsAPI and GNewsAPI) using HTTP requests, applying the date filter and search queries.

- **Nodes Involved:**  
  - News Source: NewsAPI  
  - News Source: GNewsAPI

- **Node Details:**

  - **News Source: NewsAPI**  
    - *Type:* HTTP Request  
    - *Role:* Retrieves news articles from NewsAPI based on query parameters including date and topic  
    - *Configuration:*  
      - HTTP Method: GET  
      - URL: Configured with query parameters such as `q=AI` (default topic), `from` date (from previous node), language, and API key in headers or query  
      - Authentication: API key credential required (user must input their free NewsAPI key)  
    - *Connections:* Outputs to "ExtractAllNews1"  
    - *Edge Cases:* API rate limits, invalid API key, network timeouts, malformed query parameters.

  - **News Source: GNewsAPI**  
    - *Type:* HTTP Request  
    - *Role:* Retrieves news articles from GNewsAPI with similar filtering parameters  
    - *Configuration:*  
      - HTTP Method: GET  
      - URL: Includes query parameters like `q=AI`, `lang=en`, date filters, and API key  
      - Authentication: API key credential required (user must input their free GNewsAPI key)  
    - *Connections:* Outputs to "ExtractAllNews"  
    - *Edge Cases:* Similar to NewsAPI node; includes API quota limits and potential response format differences.

#### 2.3 News Extraction & Merging

- **Overview:**  
  This block extracts relevant news data from the raw API responses and merges the two news datasets into one unified list for AI processing.

- **Nodes Involved:**  
  - ExtractAllNews  
  - ExtractAllNews1  
  - Merge

- **Node Details:**

  - **ExtractAllNews**  
    - *Type:* Set  
    - *Role:* Extracts and formats news data from GNewsAPI response to a consistent structure  
    - *Configuration:* Uses expressions to map API response fields (e.g., title, description, URL) into a simplified dataset  
    - *Connections:* Outputs to "Merge" (second input)  
    - *Edge Cases:* API response structure changes; missing or empty fields; expression evaluation errors.

  - **ExtractAllNews1**  
    - *Type:* Set  
    - *Role:* Extracts and formats news data from NewsAPI response similarly to ExtractAllNews  
    - *Configuration:* Maps NewsAPI response fields to the same structure as ExtractAllNews for consistency  
    - *Connections:* Outputs to "Merge" (first input)  
    - *Edge Cases:* Same as ExtractAllNews.

  - **Merge**  
    - *Type:* Merge  
    - *Role:* Combines the two extracted news datasets into a single array for unified processing  
    - *Configuration:* Default merge mode (append) to concatenate datasets  
    - *Connections:* Outputs to "AI Agent"  
    - *Edge Cases:* Large datasets may cause performance issues; empty inputs from either source.

#### 2.4 AI Processing

- **Overview:**  
  This block uses an AI Agent powered by the Google Gemini Chat Model to intelligently filter, prioritize, and summarize the merged news articles into a concise briefing.

- **Nodes Involved:**  
  - Google Gemini Chat Model  
  - AI Agent

- **Node Details:**

  - **Google Gemini Chat Model**  
    - *Type:* LangChain Language Model (Google Gemini)  
    - *Role:* Provides the underlying AI language model for the AI Agent node  
    - *Configuration:* Requires Google Gemini API credentials; configured for chat-based summarization tasks  
    - *Connections:* Connected as the language model input to "AI Agent"  
    - *Edge Cases:* API quota limits, authentication failures, network issues.

  - **AI Agent**  
    - *Type:* LangChain Agent  
    - *Role:* Orchestrates AI prompt construction, sends merged news data to Google Gemini, and receives summarized output  
    - *Configuration:* Uses the Google Gemini Chat Model; prompt templates likely include instructions to filter and summarize news  
    - *Connections:* Outputs summarized news to "Telegram" node  
    - *Edge Cases:* Prompt failures, unexpected AI responses, timeouts.

#### 2.5 Telegram Delivery

- **Overview:**  
  This block sends the AI-generated daily news briefing directly to the user's Telegram chat for convenient access.

- **Nodes Involved:**  
  - Telegram

- **Node Details:**

  - **Telegram**  
    - *Type:* Telegram Node  
    - *Role:* Sends messages to a configured Telegram chat or channel  
    - *Configuration:* Requires Telegram Bot API credentials (bot token); configured to send the AI Agent's output as a message  
    - *Connections:* Receives input from "AI Agent"  
    - *Edge Cases:* Invalid bot token, chat ID misconfiguration, Telegram API rate limits, message size limits.

---

### 3. Summary Table

| Node Name                     | Node Type                        | Functional Role                          | Input Node(s)                  | Output Node(s)              | Sticky Note                                                                                   |
|-------------------------------|---------------------------------|----------------------------------------|-------------------------------|-----------------------------|----------------------------------------------------------------------------------------------|
| Trigger workflow at 6am everyday | Schedule Trigger                | Initiates workflow daily at 6 AM       | —                             | Substract Current date by one |                                                                                              |
| Substract Current date by one  | DateTime                        | Calculates date one day prior           | Trigger workflow at 6am everyday | News Source: NewsAPI, News Source: GNewsAPI |                                                                                              |
| News Source: NewsAPI           | HTTP Request                   | Fetches news from NewsAPI                | Substract Current date by one  | ExtractAllNews1             | Requires free NewsAPI key; customize query parameters for topic and language                 |
| News Source: GNewsAPI          | HTTP Request                   | Fetches news from GNewsAPI               | Substract Current date by one  | ExtractAllNews              | Requires free GNewsAPI key; customize query parameters for topic and language                |
| ExtractAllNews1                | Set                            | Extracts and formats NewsAPI data       | News Source: NewsAPI           | Merge                      |                                                                                              |
| ExtractAllNews                 | Set                            | Extracts and formats GNewsAPI data      | News Source: GNewsAPI          | Merge                      |                                                                                              |
| Merge                         | Merge                          | Combines news data from both sources    | ExtractAllNews1, ExtractAllNews | AI Agent                   |                                                                                              |
| Google Gemini Chat Model       | LangChain Language Model       | Provides AI language model for summarization | —                             | AI Agent (ai_languageModel) | Requires Google Gemini API credentials                                                      |
| AI Agent                      | LangChain Agent                | Filters and summarizes news with AI    | Merge, Google Gemini Chat Model | Telegram                   |                                                                                              |
| Telegram                      | Telegram                       | Sends summarized news to Telegram chat | AI Agent                      | —                           | Requires Telegram Bot API token; configure chat ID                                         |
| Sticky Note                   | Sticky Note                   | Comments/notes                          | —                             | —                           | Various sticky notes present but empty content in this export                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Set to trigger daily at 6:00 AM  
   - Timezone: Europe/Paris  

2. **Add a DateTime Node**  
   - Type: DateTime  
   - Configure to subtract 1 day from the current date  
   - Output format compatible with ISO 8601 or API date format  

3. **Add Two HTTP Request Nodes for News APIs**  
   - **News Source: NewsAPI**  
     - Method: GET  
     - URL: `https://newsapi.org/v2/everything?q=AI&from={{$json["date"]}}&language=en&apiKey=YOUR_NEWSAPI_KEY`  
     - Replace `YOUR_NEWSAPI_KEY` with your actual API key  
     - Use expression to insert date from DateTime node  
   - **News Source: GNewsAPI**  
     - Method: GET  
     - URL: `https://gnews.io/api/v4/search?q=AI&lang=en&from={{$json["date"]}}&token=YOUR_GNEWSAPI_KEY`  
     - Replace `YOUR_GNEWSAPI_KEY` with your actual API key  
     - Use expression to insert date from DateTime node  

4. **Add Two Set Nodes to Extract News Data**  
   - **ExtractAllNews1 (for NewsAPI)**  
     - Map relevant fields from NewsAPI response (e.g., `articles.title`, `articles.description`, `articles.url`) into a simplified array  
   - **ExtractAllNews (for GNewsAPI)**  
     - Map relevant fields from GNewsAPI response similarly to maintain consistent structure  

5. **Add a Merge Node**  
   - Merge Mode: Append  
   - Connect outputs of both Set nodes as inputs to Merge node  

6. **Add Google Gemini Chat Model Node**  
   - Configure with Google Gemini API credentials  
   - Set model parameters suitable for chat-based summarization  

7. **Add AI Agent Node**  
   - Configure to use Google Gemini Chat Model as language model  
   - Set prompt templates to instruct AI to filter and summarize news articles  
   - Connect Merge node output as input to AI Agent  

8. **Add Telegram Node**  
   - Configure with Telegram Bot API token  
   - Set chat ID to target recipient (user or channel)  
   - Connect AI Agent output to Telegram node to send summarized news  

9. **Connect Nodes in Order:**  
   - Schedule Trigger → DateTime → NewsAPI & GNewsAPI HTTP Requests → ExtractAllNews1 & ExtractAllNews → Merge → AI Agent (with Google Gemini Chat Model) → Telegram  

10. **Credential Setup:**  
    - Obtain and configure API keys for NewsAPI and GNewsAPI in HTTP Request nodes  
    - Obtain and configure Google Gemini API credentials in the language model node  
    - Obtain Telegram Bot token and configure Telegram node credentials  

11. **Test Workflow:**  
    - Run once manually to verify API responses, AI summarization, and Telegram message delivery  
    - Adjust query parameters or prompt templates as needed for customization  

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                     |
|---------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Video tutorial included for Telegram integration setup within n8n instance                                    | Refer to the original workflow package or documentation for video guidance                         |
| Free API keys required for NewsAPI and GNewsAPI; instructions provided within the workflow comments           | Visit https://newsapi.org/ and https://gnews.io/ to obtain free API keys                           |
| Easily customizable search queries and language parameters in HTTP Request nodes                              | Modify `q` and `lang` parameters in HTTP Request URLs to change news topics and language          |
| Workflow timezone set to Europe/Paris; adjust if deploying in different regions                               | Change timezone in workflow settings if needed                                                   |
| AI summarization powered by Google Gemini Chat Model requires valid Google Cloud credentials                   | Ensure Google Cloud project is set up with access to Gemini API                                   |
| Telegram Bot setup requires creating a bot via BotFather and obtaining the bot token                          | See https://core.telegram.org/bots#6-botfather for Telegram bot creation instructions             |

---

This document provides a comprehensive understanding of the "Daily AI News Briefing and Summarization with Google Gemini and Telegram" workflow, enabling reproduction, customization, and troubleshooting.