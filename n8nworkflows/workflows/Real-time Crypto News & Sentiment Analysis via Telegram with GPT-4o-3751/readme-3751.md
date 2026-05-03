Real-time Crypto News & Sentiment Analysis via Telegram with GPT-4o

https://n8nworkflows.xyz/workflows/real-time-crypto-news---sentiment-analysis-via-telegram-with-gpt-4o-3751


# Real-time Crypto News & Sentiment Analysis via Telegram with GPT-4o

### 1. Workflow Overview

This workflow enables real-time cryptocurrency news aggregation and sentiment analysis delivered directly via Telegram. It targets crypto traders, investors, analysts, and market watchers who want instant, AI-powered news summaries and market sentiment insights on any crypto-related topic or coin.

The workflow is logically divided into these blocks:

- **1.1 Telegram Input Reception:** Listens for user messages in Telegram to receive crypto keywords or company names.
- **1.2 Session Initialization:** Captures and stores the Telegram chat ID as a session identifier for contextual continuity.
- **1.3 Keyword Extraction (AI-Powered):** Uses an AI agent to parse the user’s input and extract a single-word keyword to target news filtering.
- **1.4 News Aggregation:** Collects RSS feeds from nine major crypto news sources.
- **1.5 News Merging and Filtering:** Merges all aggregated articles and filters them based on the extracted keyword.
- **1.6 AI Summarization Prompt Construction:** Builds a detailed prompt listing filtered articles for the AI summarization step.
- **1.7 AI Summarization & Sentiment Analysis:** Uses GPT-4o to generate a three-part summary: news summary, market sentiment, and article references.
- **1.8 Telegram Response Preparation and Delivery:** Formats the AI output and sends the summary back to the user via Telegram.

---

### 2. Block-by-Block Analysis

#### 1.1 Telegram Input Reception

- **Overview:**  
  This block triggers the workflow when a user sends a message to the Telegram bot.

- **Nodes Involved:**  
  - Send Crypto or Company Name for Analysis (Telegram Trigger)

- **Node Details:**  
  - **Send Crypto or Company Name for Analysis**  
    - Type: Telegram Trigger  
    - Role: Listens for incoming Telegram messages (updates of type "message").  
    - Configuration: Uses Telegram API credentials linked to the bot token.  
    - Input: Telegram user message text.  
    - Output: Raw Telegram message JSON.  
    - Edge Cases: Telegram API downtime, invalid bot token, or user sends unsupported message types.  
    - Sticky Note: Explains Telegram setup and bot token configuration.

#### 1.2 Session Initialization

- **Overview:**  
  Extracts the chat ID from the Telegram message and stores it as `sessionId` for session management.

- **Nodes Involved:**  
  - Adds the sessionId (Set Node)

- **Node Details:**  
  - **Adds the sessionId**  
    - Type: Set Node  
    - Role: Assigns the Telegram chat ID (`message.chat.id`) to a new variable `sessionId`.  
    - Configuration: Uses expression `={{ $json.message.chat.id }}` to extract chat ID.  
    - Input: Telegram message JSON from previous node.  
    - Output: JSON with `sessionId`.  
    - Edge Cases: Missing or malformed chat ID in Telegram message.  
    - Sticky Note: Describes session ID usage for conversation memory.

#### 1.3 Keyword Extraction (AI-Powered)

- **Overview:**  
  Uses an AI agent to analyze the user’s message text and extract a single-word keyword for targeted news filtering.

- **Nodes Involved:**  
  - Crypto News & Sentiment Agent (Langchain Agent)  
  - OpenAI Chat Model (Langchain LM Chat OpenAI) [used internally by Agent]

- **Node Details:**  
  - **Crypto News & Sentiment Agent**  
    - Type: Langchain Agent  
    - Role: Processes user input text to output a single keyword.  
    - Configuration:  
      - Input text: `={{ $('Send Crypto or Company Name for Analysis').item.json.message.text }}`  
      - System message: "Your job is to analyze the keyword of the question and output it as a single word."  
      - Prompt type: define  
    - Input: User message text and sessionId.  
    - Output: Single keyword string in `output`.  
    - Edge Cases: Ambiguous user input, AI misinterpretation, or API errors.  
    - Sub-workflow: Uses OpenAI Chat Model node internally.  
    - **OpenAI Chat Model**  
      - Type: Langchain LM Chat OpenAI  
      - Role: Provides GPT-4o-mini model for the agent’s language understanding.  
      - Configuration: Model set to `gpt-4o-mini`.  
      - Credentials: OpenAI API credentials required.  
      - Edge Cases: API rate limits, invalid credentials, or model unavailability.

- **Next Step:** The extracted keyword is passed to the next block.

#### 1.4 News Aggregation

- **Overview:**  
  Fetches RSS feeds from nine major cryptocurrency news sources to gather the latest articles.

- **Nodes Involved:**  
  - RSS Cointelegraph  
  - RSS Bitcoinmagazine  
  - RSS Coindesk  
  - RSS Bitcoinist  
  - RSS Newsbtc  
  - RSS Cryptopotato  
  - RSS 99bitcoins  
  - RSS Cryptobriefing  
  - RSS Crypto.news

- **Node Details:**  
  Each node:  
  - Type: RSS Feed Read  
  - Role: Pulls latest articles from the specified RSS URL.  
  - Configuration: Each node has a unique RSS feed URL for a major crypto news source.  
  - Input: Triggered after keyword extraction and query set.  
  - Output: List of articles (title, link, snippet, content).  
  - Edge Cases: RSS feed downtime, malformed XML, or network errors.  
  - Sticky Note: Notes that more RSS feeds can be added to expand coverage.

#### 1.5 News Merging and Filtering

- **Overview:**  
  Merges articles from all RSS feeds into a single list, then filters articles containing the extracted keyword.

- **Nodes Involved:**  
  - Merge All Articles (Merge Node)  
  - Filter by Query (Code Node)

- **Node Details:**  
  - **Merge All Articles**  
    - Type: Merge  
    - Role: Combines 10 inputs (9 RSS feeds + one extra input slot) into one unified list.  
    - Configuration: Set to merge by combining all inputs.  
    - Input: Articles from all RSS nodes.  
    - Output: Single array of all articles.  
    - Edge Cases: Empty inputs, duplicate articles, or large data causing performance issues.  
  - **Filter by Query**  
    - Type: Code (JavaScript)  
    - Role: Filters merged articles to keep only those whose title, snippet, or full content includes the extracted keyword (case-insensitive).  
    - Configuration: Uses the keyword from the "Set Query" node (`$node["Set Query"].json.query`).  
    - Input: Merged articles.  
    - Output: Filtered articles matching the keyword.  
    - Edge Cases: No articles match keyword, keyword is empty or malformed, expression errors.  
    - Code snippet:  
      ```js
      const term = $node["Set Query"].json.query.toLowerCase();
      return items.filter(item => {
        const j = item.json;
        const title = (j.title || "").toLowerCase();
        const snippet = (j.contentSnippet || j.description || "").toLowerCase();
        const fullContent = (j.content || "").toLowerCase();
        return title.includes(term) || snippet.includes(term) || fullContent.includes(term);
      });
      ```

#### 1.6 AI Summarization Prompt Construction

- **Overview:**  
  Builds a detailed prompt listing filtered articles to instruct GPT-4o to summarize news and analyze sentiment.

- **Nodes Involved:**  
  - Set Query (Set Node)  
  - Build Prompt (Code Node)

- **Node Details:**  
  - **Set Query**  
    - Type: Set  
    - Role: Stores the extracted keyword as `query` and sessionId for downstream use.  
    - Configuration:  
      - `query` assigned from AI agent output (`={{ $json.output }}`)  
      - `sessionId` assigned from previous node.  
    - Input: AI agent output.  
    - Output: JSON with `query` and `sessionId`.  
  - **Build Prompt**  
    - Type: Code (JavaScript)  
    - Role: Constructs a multi-line prompt string including the keyword and a bullet list of filtered article titles with links.  
    - Input: Filtered articles and `query`.  
    - Output: JSON with `prompt` string.  
    - Edge Cases: Empty article list, malformed article data.  
    - Code snippet:  
      ```js
      const q = $node["Set Query"].json.query;
      const list = items
        .map(i => `- ${i.json.title} (${i.json.link})`)
        .join("\n");
      const prompt = `
      You are a crypto-industry news analyst.
      Summarize current news and market sentiment for **${q}** based on these articles:
      ${list}

      Answer in 3 parts:
      1. Summary of News
      2. Market Sentiment
      3. Links to reference news articles
      `;
      return [{ json: { prompt } }];
      ```

- **Sticky Note:** Describes prompt preparation for GPT-4o input.

#### 1.7 AI Summarization & Sentiment Analysis

- **Overview:**  
  Sends the constructed prompt to GPT-4o to generate a structured summary and sentiment analysis.

- **Nodes Involved:**  
  - Summarize News & Sentiment (GPT-4o) (Langchain OpenAI Node)

- **Node Details:**  
  - Type: Langchain OpenAI  
  - Role: Uses OpenAI GPT-4o model to generate a concise summary of news, market sentiment, and article links.  
  - Configuration:  
    - Model: `gpt-4o`  
    - Messages:  
      - User prompt from "Build Prompt" node.  
      - Assistant system message reinforcing analyst role.  
    - Credentials: OpenAI API credentials required.  
  - Input: Prompt string.  
  - Output: AI-generated summary text.  
  - Edge Cases: API errors, rate limits, empty prompt, or malformed response.  
  - Sticky Note: Explains the summarization and sentiment analysis role.

#### 1.8 Telegram Response Preparation and Delivery

- **Overview:**  
  Formats the AI response and sends the summary back to the user via Telegram.

- **Nodes Involved:**  
  - Prepare Telegram Message (Set Node)  
  - Sends Response (Telegram Send Node)

- **Node Details:**  
  - **Prepare Telegram Message**  
    - Type: Set  
    - Role: Extracts the AI-generated summary content and assigns it to a `summary` variable for sending.  
    - Configuration: Assigns `summary` from `message.content` of the previous node.  
    - Input: AI summarization output.  
    - Output: JSON with `summary` string.  
    - Edge Cases: Missing or empty AI response.  
    - Sticky Note: Notes formatting for Telegram delivery.  
  - **Sends Response**  
    - Type: Telegram  
    - Role: Sends the summary text as a Telegram message to the user’s chat ID.  
    - Configuration:  
      - Text: `={{ $json.summary }}`  
      - Chat ID: Placeholder `<< Add Telegram ID here >>` to be replaced with dynamic sessionId or actual chat ID.  
      - Credentials: Telegram API credentials linked to the bot.  
      - Additional fields: Attribution disabled.  
    - Input: Prepared summary text.  
    - Output: Telegram message sent confirmation.  
    - Edge Cases: Invalid chat ID, Telegram API errors, message size limits.  
    - Sticky Note: Warns to replace chatId with dynamic value.

---

### 3. Summary Table

| Node Name                         | Node Type                          | Functional Role                          | Input Node(s)                                              | Output Node(s)                         | Sticky Note                                                                                       |
|----------------------------------|----------------------------------|----------------------------------------|------------------------------------------------------------|--------------------------------------|-------------------------------------------------------------------------------------------------|
| Send Crypto or Company Name for Analysis | Telegram Trigger                 | Receives user messages from Telegram   | —                                                          | Adds the sessionId                   | ## Telegram Setup This bot listens for incoming messages in Telegram. To use it, create a bot with @BotFather and paste your bot token into the credentials. |
| Adds the sessionId                | Set                              | Stores Telegram chat ID as sessionId   | Send Crypto or Company Name for Analysis                    | Crypto News & Sentiment Agent        | ## Initialize Chat Session Stores the user's chat ID as sessionId, which is used to manage conversation memory across steps. |
| Crypto News & Sentiment Agent    | Langchain Agent                  | Extracts keyword from user input       | Adds the sessionId                                         | Set Query                           | ##  Extract Keyword This AI agent parses the user input and returns a single-word keyword to help match relevant news articles. |
| OpenAI Chat Model                | Langchain LM Chat OpenAI          | Provides GPT-4o-mini model for agent   | Crypto News & Sentiment Agent (internal)                   | Crypto News & Sentiment Agent (internal) |                                                                                                 |
| Set Query                       | Set                              | Stores extracted keyword and sessionId | Crypto News & Sentiment Agent                               | RSS Cointelegraph (and other RSS nodes) |                                                                                                 |
| RSS Cointelegraph               | RSS Feed Read                    | Fetches Cointelegraph articles         | Set Query                                                  | Merge All Articles                  | ## News Aggregators Pulls articles from major crypto news sources. You can add more RSS feeds here to expand coverage. |
| RSS Bitcoinmagazine             | RSS Feed Read                    | Fetches Bitcoin Magazine articles      | Set Query                                                  | Merge All Articles                  | ## News Aggregators Pulls articles from major crypto news sources. You can add more RSS feeds here to expand coverage. |
| RSS Coindesk                   | RSS Feed Read                    | Fetches Coindesk articles               | Set Query                                                  | Merge All Articles                  | ## News Aggregators Pulls articles from major crypto news sources. You can add more RSS feeds here to expand coverage. |
| RSS Bitcoinist                 | RSS Feed Read                    | Fetches Bitcoinist articles             | Set Query                                                  | Merge All Articles                  | ## News Aggregators Pulls articles from major crypto news sources. You can add more RSS feeds here to expand coverage. |
| RSS Newsbtc                   | RSS Feed Read                    | Fetches NewsBTC articles                | Set Query                                                  | Merge All Articles                  | ## News Aggregators Pulls articles from major crypto news sources. You can add more RSS feeds here to expand coverage. |
| RSS Cryptopotato              | RSS Feed Read                    | Fetches CryptoPotato articles           | Set Query                                                  | Merge All Articles                  | ## News Aggregators Pulls articles from major crypto news sources. You can add more RSS feeds here to expand coverage. |
| RSS 99bitcoins                | RSS Feed Read                    | Fetches 99Bitcoins articles             | Set Query                                                  | Merge All Articles                  | ## News Aggregators Pulls articles from major crypto news sources. You can add more RSS feeds here to expand coverage. |
| RSS Cryptobriefing            | RSS Feed Read                    | Fetches CryptoBriefing articles          | Set Query                                                  | Merge All Articles                  | ## News Aggregators Pulls articles from major crypto news sources. You can add more RSS feeds here to expand coverage. |
| RSS Crypto.news               | RSS Feed Read                    | Fetches Crypto.news articles             | Set Query                                                  | Merge All Articles                  | ## News Aggregators Pulls articles from major crypto news sources. You can add more RSS feeds here to expand coverage. |
| Merge All Articles            | Merge                           | Combines all RSS articles into one list | All RSS Feed nodes                                        | Filter by Query                    | ## Merge & Filter News Combines all RSS articles and filters them using the extracted keyword to match only relevant results. |
| Filter by Query               | Code                            | Filters articles by extracted keyword   | Merge All Articles                                        | Build Prompt                      |                                                                                                 |
| Build Prompt                 | Code                            | Builds AI prompt with filtered articles | Filter by Query                                          | Summarize News & Sentiment (GPT-4o) | ## Prepare AI Prompt Constructs the input prompt for GPT-4o with a list of filtered articles. Output includes summary, sentiment, and article links. |
| Summarize News & Sentiment (GPT-4o) | Langchain OpenAI                | Generates news summary and sentiment    | Build Prompt                                            | Prepare Telegram Message            | ## Summarize News & Sentiment (GPT-4o) Uses OpenAI GPT-4o to generate a concise summary of the news, analyze market sentiment, and return formatted results. |
| Prepare Telegram Message      | Set                              | Extracts AI summary for Telegram message | Summarize News & Sentiment (GPT-4o)                      | Sends Response                    | ## Format for Telegram Extracts the AI summary and prepares it for Telegram delivery. Change the variable if you want to include other data. |
| Sends Response               | Telegram                         | Sends summary back to user via Telegram | Prepare Telegram Message                                | —                                | ## Send Telegram Response Sends the final AI-generated summary to the user. ⚠️ Replace chatId with a dynamic value like << Telegram ID here >> to ensure it sends to the right user. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Bot**  
   - Use @BotFather on Telegram to create a bot and obtain the bot token.

2. **Configure Telegram Credentials in n8n**  
   - Add a new Telegram API credential with the bot token.

3. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Configure to listen for "message" updates.  
   - Assign the Telegram API credential.  
   - Name it "Send Crypto or Company Name for Analysis".

4. **Add Set Node to Store Session ID**  
   - Name: "Adds the sessionId"  
   - Add assignment: `sessionId` = `={{ $json.message.chat.id }}`  
   - Connect Telegram Trigger output to this node.

5. **Add Langchain Agent Node for Keyword Extraction**  
   - Name: "Crypto News & Sentiment Agent"  
   - Input text: `={{ $('Send Crypto or Company Name for Analysis').item.json.message.text }}`  
   - System message: "Your job is to analyze the keyword of the question and output it as a single word."  
   - Prompt type: define  
   - Connect "Adds the sessionId" output to this node.  
   - Add Langchain OpenAI Chat Model node as the language model:  
     - Model: `gpt-4o-mini`  
     - Credentials: OpenAI API credentials.

6. **Add Set Node to Store Extracted Keyword and Session ID**  
   - Name: "Set Query"  
   - Assignments:  
     - `query` = `={{ $json.output }}` (from AI agent)  
     - `sessionId` = `={{ $('Adds the sessionId').item.json.sessionId }}`  
   - Connect "Crypto News & Sentiment Agent" output here.

7. **Add RSS Feed Read Nodes for Each Crypto News Source**  
   - Create 9 nodes named accordingly:  
     - RSS Cointelegraph: https://cointelegraph.com/rss  
     - RSS Bitcoinmagazine: https://bitcoinmagazine.com/.rss/full/  
     - RSS Coindesk: https://www.coindesk.com/arc/outboundfeeds/rss/  
     - RSS Bitcoinist: https://bitcoinist.com/feed/  
     - RSS Newsbtc: https://www.newsbtc.com/feed/  
     - RSS Cryptopotato: https://cryptopotato.com/feed/  
     - RSS 99bitcoins: https://99bitcoins.com/feed/  
     - RSS Cryptobriefing: https://cryptobriefing.com/feed/  
     - RSS Crypto.news: https://crypto.news/feed/  
   - Connect "Set Query" output to all RSS nodes.

8. **Add Merge Node to Combine RSS Articles**  
   - Name: "Merge All Articles"  
   - Number of inputs: 10 (to accommodate all RSS nodes)  
   - Connect all RSS nodes outputs to this merge node.

9. **Add Code Node to Filter Articles by Keyword**  
   - Name: "Filter by Query"  
   - JavaScript code:  
     ```js
     const term = $node["Set Query"].json.query.toLowerCase();
     return items.filter(item => {
       const j = item.json;
       const title = (j.title || "").toLowerCase();
       const snippet = (j.contentSnippet || j.description || "").toLowerCase();
       const fullContent = (j.content || "").toLowerCase();
       return title.includes(term) || snippet.includes(term) || fullContent.includes(term);
     });
     ```  
   - Connect "Merge All Articles" output here.

10. **Add Code Node to Build AI Prompt**  
    - Name: "Build Prompt"  
    - JavaScript code:  
      ```js
      const q = $node["Set Query"].json.query;
      const list = items
        .map(i => `- ${i.json.title} (${i.json.link})`)
        .join("\n");
      const prompt = `
      You are a crypto-industry news analyst.
      Summarize current news and market sentiment for **${q}** based on these articles:
      ${list}

      Answer in 3 parts:
      1. Summary of News
      2. Market Sentiment
      3. Links to reference news articles
      `;
      return [{ json: { prompt } }];
      ```  
    - Connect "Filter by Query" output here.

11. **Add Langchain OpenAI Node for Summarization**  
    - Name: "Summarize News & Sentiment (GPT-4o)"  
    - Model ID: `gpt-4o`  
    - Messages:  
      - User content: `={{ $node["Build Prompt"].json.prompt }}`  
      - Assistant content: "You are a crypto‐industry news analyst. Summarize sentiment clearly and concisely."  
    - Credentials: OpenAI API credentials.  
    - Connect "Build Prompt" output here.

12. **Add Set Node to Prepare Telegram Message**  
    - Name: "Prepare Telegram Message"  
    - Assignment: `summary` = `={{ $json.message.content }}` (from GPT-4o output)  
    - Connect "Summarize News & Sentiment (GPT-4o)" output here.

13. **Add Telegram Node to Send Response**  
    - Name: "Sends Response"  
    - Text: `={{ $json.summary }}`  
    - Chat ID: Replace placeholder `<< Add Telegram ID here >>` with dynamic expression `={{ $('Adds the sessionId').item.json.sessionId }}` to send to the correct user.  
    - Credentials: Telegram API credentials.  
    - Connect "Prepare Telegram Message" output here.

14. **Activate and Test Workflow**  
    - Deploy workflow.  
    - Send test messages like "Bitcoin", "NFT", or "Ethereum" to the Telegram bot.  
    - Verify that summarized news and sentiment analysis are returned promptly.

---

### 5. General Notes & Resources

| Note Content                                                                                                     | Context or Link                                                                                   |
|-----------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| To create your Telegram bot, use @BotFather on Telegram and obtain the bot token.                               | https://t.me/BotFather                                                                           |
| Replace the placeholder `<< Add Telegram ID here >>` in the Telegram Send node with a dynamic expression to send messages to the correct user chat ID. | Important for multi-user support and correct message routing.                                   |
| The workflow uses GPT-4o and GPT-4o-mini models from OpenAI for keyword extraction and summarization.           | Requires valid OpenAI API credentials with access to these models.                              |
| You can expand news coverage by adding more RSS Feed Read nodes with additional crypto news sources.            | RSS feed URLs must be valid and accessible.                                                     |
| The AI summarization prompt instructs GPT-4o to answer in three parts: news summary, market sentiment, and links.| This structured output improves readability and usefulness of the Telegram message.             |
| The workflow is designed for real-time interaction and may require handling API rate limits or network errors.  | Consider adding error handling or retries for production use.                                  |
| Sticky notes in the workflow provide helpful setup guidance and explanations for each logical block.            | Refer to sticky notes inside n8n editor for inline documentation.                               |

---

This completes the comprehensive reference document for the "Real-time Crypto News & Sentiment Analysis via Telegram with GPT-4o" workflow. It enables advanced users and AI agents to fully understand, reproduce, and maintain the workflow effectively.