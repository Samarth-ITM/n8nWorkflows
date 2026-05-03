Daily AI News Translation & Summary with GPT-4 and Telegram Delivery

https://n8nworkflows.xyz/workflows/daily-ai-news-translation---summary-with-gpt-4-and-telegram-delivery-3596


# Daily AI News Translation & Summary with GPT-4 and Telegram Delivery

### 1. Workflow Overview

This workflow automates the daily collection, summarization, translation, and delivery of AI-related news articles. Every morning at 8 a.m., it fetches up to 40 recent AI news articles from two distinct sources—GNews and NewsAPI—merges and filters them to select the 15 most relevant items, then uses GPT-4.1 to generate concise summaries in Traditional Chinese while preserving key English technical terms. Finally, the compiled digest is sent to a specified Telegram chat or group.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger:** Initiates the workflow daily at a fixed time (8 a.m.).
- **1.2 News Fetching:** Retrieves AI news articles from GNews and NewsAPI APIs.
- **1.3 Data Mapping & Merging:** Standardizes and merges articles from both sources into a single collection.
- **1.4 AI Summarization & Translation:** Uses GPT-4.1 to select, summarize, and translate the top 15 AI news articles into Traditional Chinese.
- **1.5 Telegram Delivery:** Sends the AI-generated news digest to a designated Telegram chat or group.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
  This block triggers the entire workflow automatically every day at 8 a.m.

- **Nodes Involved:**  
  - `Trigger at 8am daily`

- **Node Details:**

  - **Trigger at 8am daily**  
    - Type: Schedule Trigger  
    - Configuration: Set to trigger once daily at 8:00 AM local time.  
    - Inputs: None (start node)  
    - Outputs: Triggers downstream news fetching nodes.  
    - Edge Cases: If the n8n instance is offline or paused at trigger time, the workflow will not start. Timezone considerations may affect trigger time if not configured explicitly.  
    - Version: 1.2

---

#### 1.2 News Fetching

- **Overview:**  
  This block fetches AI-related news articles from two external APIs: GNews and NewsAPI. Each API returns up to 20 recent English-language articles about AI.

- **Nodes Involved:**  
  - `Fetch GNews articles`  
  - `Fetch NewsAPI articles`  
  - `Sticky: News APIs` (documentation note)

- **Node Details:**

  - **Fetch GNews articles**  
    - Type: HTTP Request  
    - Configuration:  
      - URL: `https://gnews.io/api/v4/search`  
      - Query parameters:  
        - `q=AI` (search keyword)  
        - `lang=en` (English language)  
        - `apikey` (user’s GNews API key, set in node credentials or parameters)  
      - Method: GET  
    - Inputs: Trigger node output  
    - Outputs: JSON response containing an `articles` array with news items.  
    - Edge Cases: API key invalid or quota exceeded; network errors; empty or malformed response.  
    - Version: 4.2

  - **Fetch NewsAPI articles**  
    - Type: HTTP Request  
    - Configuration:  
      - URL: `https://newsapi.org/v2/everything`  
      - Query parameters:  
        - `q=AI`  
        - `language=en`  
        - `sortBy=publishedAt` (sort by newest)  
        - `pageSize=20` (limit results)  
      - Headers:  
        - `X-Api-Key` (user’s NewsAPI key)  
      - Method: GET  
    - Inputs: Trigger node output  
    - Outputs: JSON response containing an `articles` array with news items.  
    - Edge Cases: API key invalid or quota exceeded; network errors; empty or malformed response.  
    - Version: 4.2

  - **Sticky: News APIs**  
    - Type: Sticky Note  
    - Content: Explains that these two nodes fetch up to 20 AI news articles each and standardize the data for merging.

---

#### 1.3 Data Mapping & Merging

- **Overview:**  
  This block extracts the `articles` array from each API response, standardizes the data structure, and merges both sources into a single collection for further processing.

- **Nodes Involved:**  
  - `GNews: Map to articles`  
  - `NewsAPI: Map to articles`  
  - `Merge GNews & NewsAPI`

- **Node Details:**

  - **GNews: Map to articles**  
    - Type: Set  
    - Configuration: Extracts the `articles` property from the GNews API response and assigns it to a new field named `articles`.  
    - Inputs: Output from `Fetch GNews articles`  
    - Outputs: JSON with `articles` array ready for merging.  
    - Edge Cases: If the response lacks `articles`, output will be empty or invalid.  
    - Version: 3.4

  - **NewsAPI: Map to articles**  
    - Type: Set  
    - Configuration: Extracts the `articles` property from the NewsAPI response and assigns it to a new field named `articles`.  
    - Inputs: Output from `Fetch NewsAPI articles`  
    - Outputs: JSON with `articles` array ready for merging.  
    - Edge Cases: Similar to GNews mapping.  
    - Version: 3.4

  - **Merge GNews & NewsAPI**  
    - Type: Merge  
    - Configuration: Merges the two incoming article arrays into one combined list.  
    - Inputs: Outputs from both mapping nodes (two inputs)  
    - Outputs: Single merged JSON object containing combined `articles`.  
    - Edge Cases: If one source fails or returns empty, the merge still proceeds with available data.  
    - Version: 3.1

---

#### 1.4 AI Summarization & Translation

- **Overview:**  
  This block uses GPT-4.1 via Langchain to select the 15 most relevant AI articles, translate summaries into Traditional Chinese (preserving English technical terms), and format the output with article URLs and the current date.

- **Nodes Involved:**  
  - `AI summarizer & translator`  
  - `GPT-4.1 Model`  
  - `Sticky: AI Processing` (documentation note)

- **Node Details:**

  - **AI summarizer & translator**  
    - Type: Langchain Agent  
    - Configuration:  
      - Prompt instructs the AI to:  
        1. Select top 15 relevant AI articles from the merged list.  
        2. Translate summaries into accurate Traditional Chinese, preserving English technical terms.  
        3. Include article URLs.  
        4. Begin output with today’s date in format: "早安，這是 yyyy/MM/dd 的 AI 新聞："  
      - Input variable: `{{$json.articles}}` (merged articles)  
      - Output: Text summary digest  
    - Inputs: Output from `Merge GNews & NewsAPI`  
    - Outputs: Summary text passed to Telegram node  
    - Edge Cases: AI model errors, prompt failures, rate limits, or incomplete data may cause incomplete or incorrect summaries.  
    - Version: 1.8

  - **GPT-4.1 Model**  
    - Type: Langchain OpenAI Chat Model  
    - Configuration:  
      - Model: GPT-4.1  
      - Credentials: OpenAI API key (configured in n8n credentials)  
    - Inputs: Connected as language model backend for the Langchain agent node  
    - Outputs: AI-generated summary text  
    - Edge Cases: API quota exceeded, authentication errors, network issues.  
    - Version: 1.2

  - **Sticky: AI Processing**  
    - Type: Sticky Note  
    - Content: Describes the AI logic using GPT-4.1 to select, translate, and enrich the top 15 AI news articles.

---

#### 1.5 Telegram Delivery

- **Overview:**  
  This block sends the AI-generated news digest to a specified Telegram chat or group using a Telegram Bot.

- **Nodes Involved:**  
  - `Send summary to Telegram`

- **Node Details:**

  - **Send summary to Telegram**  
    - Type: Telegram  
    - Configuration:  
      - Text: Uses the AI summary output (`{{$json.output}}`) as message content.  
      - Additional fields: Attribution disabled (no appended bot signature).  
      - Credentials: Telegram Bot token configured in n8n credentials.  
      - Chat ID: User-specified chat or group ID where the message is sent.  
    - Inputs: Output from `AI summarizer & translator`  
    - Outputs: None (terminal node)  
    - Edge Cases: Invalid chat ID, bot not added to group, Telegram API errors, rate limits.  
    - Version: 1.2

---

### 3. Summary Table

| Node Name                 | Node Type                      | Functional Role                      | Input Node(s)                   | Output Node(s)                  | Sticky Note                                                                                          |
|---------------------------|--------------------------------|------------------------------------|--------------------------------|--------------------------------|----------------------------------------------------------------------------------------------------|
| Workflow Overview         | Sticky Note                    | Setup and customization instructions | None                           | None                           | Setup instructions and customization tips for API keys, Telegram bot, OpenAI credentials           |
| Trigger at 8am daily      | Schedule Trigger               | Initiates workflow daily at 8 a.m. | None                           | Fetch GNews articles, Fetch NewsAPI articles |                                                                                                    |
| Fetch GNews articles      | HTTP Request                  | Fetch AI news from GNews API       | Trigger at 8am daily           | GNews: Map to articles         |                                                                                                    |
| Fetch NewsAPI articles    | HTTP Request                  | Fetch AI news from NewsAPI         | Trigger at 8am daily           | NewsAPI: Map to articles       |                                                                                                    |
| GNews: Map to articles    | Set                           | Extracts and standardizes GNews articles | Fetch GNews articles           | Merge GNews & NewsAPI          |                                                                                                    |
| NewsAPI: Map to articles  | Set                           | Extracts and standardizes NewsAPI articles | Fetch NewsAPI articles         | Merge GNews & NewsAPI          |                                                                                                    |
| Merge GNews & NewsAPI     | Merge                         | Combines articles from both sources | GNews: Map to articles, NewsAPI: Map to articles | AI summarizer & translator    |                                                                                                    |
| AI summarizer & translator| Langchain Agent               | Selects, summarizes, translates top 15 articles | Merge GNews & NewsAPI          | Send summary to Telegram       | AI Assistant Logic: Uses GPT-4.1 to generate concise Traditional Chinese summaries with URLs       |
| GPT-4.1 Model             | Langchain OpenAI Chat Model   | Provides GPT-4.1 AI model backend  | AI summarizer & translator (as LM) | AI summarizer & translator (output) |                                                                                                    |
| Send summary to Telegram  | Telegram                      | Sends final digest to Telegram chat | AI summarizer & translator     | None                          |                                                                                                    |
| Sticky: News APIs         | Sticky Note                   | Documentation for news fetching nodes | None                           | None                           | Explains data sources and standardization for merging                                              |
| Sticky: AI Processing     | Sticky Note                   | Documentation for AI summarization | None                           | None                           | Describes AI logic and GPT-4.1 usage                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Schedule Trigger node:**  
   - Name: `Trigger at 8am daily`  
   - Set to trigger once daily at 8:00 AM (local time).  
   - No credentials needed.

3. **Add two HTTP Request nodes for news fetching:**

   - **Fetch GNews articles:**  
     - Name: `Fetch GNews articles`  
     - HTTP Method: GET  
     - URL: `https://gnews.io/api/v4/search`  
     - Query Parameters:  
       - `q`: `AI`  
       - `lang`: `en`  
       - `apikey`: Your GNews API key (set in node credentials or parameters)  
     - Connect input from `Trigger at 8am daily`.

   - **Fetch NewsAPI articles:**  
     - Name: `Fetch NewsAPI articles`  
     - HTTP Method: GET  
     - URL: `https://newsapi.org/v2/everything`  
     - Query Parameters:  
       - `q`: `AI`  
       - `language`: `en`  
       - `sortBy`: `publishedAt`  
       - `pageSize`: `20`  
     - Header Parameters:  
       - `X-Api-Key`: Your NewsAPI key  
     - Connect input from `Trigger at 8am daily`.

4. **Add two Set nodes to extract articles arrays:**

   - **GNews: Map to articles:**  
     - Name: `GNews: Map to articles`  
     - Set one field:  
       - Name: `articles`  
       - Value: Expression `{{$json["articles"]}}`  
     - Connect input from `Fetch GNews articles`.

   - **NewsAPI: Map to articles:**  
     - Name: `NewsAPI: Map to articles`  
     - Set one field:  
       - Name: `articles`  
       - Value: Expression `{{$json["articles"]}}`  
     - Connect input from `Fetch NewsAPI articles`.

5. **Add a Merge node to combine articles:**

   - Name: `Merge GNews & NewsAPI`  
   - Mode: Merge by combining inputs (default)  
   - Connect inputs from both `GNews: Map to articles` and `NewsAPI: Map to articles`.

6. **Add a Langchain Agent node for AI summarization and translation:**

   - Name: `AI summarizer & translator`  
   - Set prompt type to "define" or custom prompt.  
   - Prompt text:  
     ```
     You are an AI news assistant. Your tasks:
     1. Select the 15 most relevant articles on AI technology progress and applications from {{$json.articles}}.
     2. Translate them to accurate Traditional Chinese; don't translate commonly used technical English terms.
     3. Make sure to include the article URL for each item.
     4. Begin output with today's date (e.g., '早安，這是 {{ $now.format('yyyy/MM/dd') }} 的 AI 新聞：')
     Output only the summary.
     ```
   - Connect input from `Merge GNews & NewsAPI`.

7. **Add a Langchain OpenAI Chat Model node:**

   - Name: `GPT-4.1 Model`  
   - Model: Select `gpt-4.1` or equivalent GPT-4 model.  
   - Credentials: Configure OpenAI API key credentials in n8n and assign here.  
   - Connect this node as the language model backend for the `AI summarizer & translator` node (set in the Langchain Agent node’s AI model settings).

8. **Add a Telegram node to send the summary:**

   - Name: `Send summary to Telegram`  
   - Credentials: Create and select Telegram Bot credentials with your Bot Token.  
   - Parameters:  
     - Text: Expression `{{$json.output}}` (output from AI summarizer)  
     - Chat ID: Enter your target Telegram user, group, or channel ID.  
     - Disable appended attribution.  
   - Connect input from `AI summarizer & translator`.

9. **Add Sticky Notes for documentation (optional):**

   - Add a sticky note near news fetching nodes explaining API keys and data sources.  
   - Add a sticky note near AI nodes explaining GPT-4.1 usage and summarization logic.  
   - Add a sticky note at the start with setup and customization instructions.

10. **Save and activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                      |
|---------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Register for NewsAPI and GNews accounts to obtain API keys.                                                   | https://newsapi.org/ and https://gnews.io/                                                         |
| Create a Telegram Bot via BotFather and obtain Bot Token for Telegram API credentials.                         | https://core.telegram.org/bots#6-botfather                                                         |
| Configure OpenAI API credentials in n8n for GPT-4.1 usage.                                                    | https://platform.openai.com/account/api-keys                                                        |
| Customize keywords in news fetching nodes to change the topic (e.g., "blockchain", "quantum computing").      | Workflow customization instructions                                                                |
| Adjust the schedule trigger to change delivery time as needed.                                                | Workflow customization instructions                                                                |
| Refine AI prompt to change summary style, language, or tone.                                                  | Workflow customization instructions                                                                |
| This workflow helps reduce information overload by delivering concise, translated AI news summaries daily.   | Workflow purpose                                                                                     |

---

This document provides a complete, structured reference for understanding, reproducing, and customizing the "Daily AI News Translation & Summary with GPT-4 and Telegram Delivery" workflow in n8n.