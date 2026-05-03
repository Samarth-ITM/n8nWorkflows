Generate Stock Market Insights with Technical Analysis, AI, and Telegram Publishing

https://n8nworkflows.xyz/workflows/generate-stock-market-insights-with-technical-analysis--ai--and-telegram-publishing-8094


# Generate Stock Market Insights with Technical Analysis, AI, and Telegram Publishing

### 1. Workflow Overview

This workflow automates the generation and publication of stock market insights using technical analysis, AI-driven content creation, and Telegram for moderation and publishing. It targets fintech startups, investment bloggers, and analysts who want to create engaging, easy-to-understand stock analysis posts for social media and a proprietary platform called "Profit."

The workflow is logically divided into the following blocks:

- **1.1 Trigger & Input Reception:** Scheduled runs and Telegram callbacks trigger the workflow, defining tickers for analysis.
- **1.2 Market Data Retrieval & Technical Analysis:** Fetches historical stock data and computes a range of technical indicators.
- **1.3 AI Content Generation:** Uses an AI language model with a structured prompt to generate a viral social media post in JSON format.
- **1.4 Post Assembly & Storage:** Constructs the final post payload, saves it to PostgreSQL, then sends it to Telegram for human moderation.
- **1.5 Telegram Moderation & Publishing:** Handles Telegram inline button actions to either publish the post to the Profit platform or retry generation, including error handling and notifications.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & Input Reception

**Overview:**  
This block receives schedule triggers twice daily or Telegram callback queries from inline button interactions. It extracts action commands and ticker identifiers for further processing.

**Nodes Involved:**  
- Schedule Trigger  
- Telegram Trigger  
- Get Action Type  
- Exist type and id  
- Get Post By Id  
- Query callback  
- Развилка (Switch)  

**Node Details:**

- **Schedule Trigger**  
  - *Type:* Schedule Trigger  
  - *Role:* Starts the workflow twice daily at 11:15 and 18:15 for automatic stock analysis.  
  - *Config:* Fixed schedule times; no input required.  
  - *Inputs:* None  
  - *Outputs:* Next node "Данные для анализа (тут указываются ticker)"  
  - *Failure Cases:* Scheduler misconfiguration unlikely; workflow depends on external nodes for data.  

- **Telegram Trigger**  
  - *Type:* Telegram Trigger  
  - *Role:* Listens for Telegram callback queries (button clicks) to handle user interactions.  
  - *Config:* Filters only callback_query updates; uses Telegram Bot credentials.  
  - *Inputs:* Telegram webhook  
  - *Outputs:* Sends data to "Get Action Type" and "Query callback" nodes.  
  - *Failure Cases:* Telegram API rate limits, invalid callback data.  

- **Get Action Type**  
  - *Type:* Set Node  
  - *Role:* Parses callback_query data into two variables: `action` (e.g. "publish", "retry") and `id` (post identifier).  
  - *Config:* Splits string by "::" delimiter to extract action and id.  
  - *Inputs:* Telegram Trigger  
  - *Outputs:* "Exist type and id"  
  - *Failure Cases:* Malformed callback data causing parsing errors.  

- **Exist type and id**  
  - *Type:* If Node  
  - *Role:* Checks presence of both `action` and `id` before proceeding.  
  - *Config:* Both fields must exist and be non-empty strings.  
  - *Inputs:* Get Action Type  
  - *Outputs:*  
    - True: "Get Post By Id"  
    - False: "Ошибка публикации" (Error notification)  
  - *Failure Cases:* Missing or corrupted callback data.  

- **Get Post By Id**  
  - *Type:* PostgreSQL Select  
  - *Role:* Retrieves a post from `analytics_history` table by given id for potential republishing or retry.  
  - *Config:* Select single row where `id` equals input id.  
  - *Inputs:* Exist type and id (true path)  
  - *Outputs:* "Развилка" (Switch)  
  - *Failure Cases:* Database connection errors, missing record.  

- **Query callback**  
  - *Type:* Telegram API call (callback answer)  
  - *Role:* Acknowledges the Telegram callback query to remove loading state on client.  
  - *Config:* Uses callback_query.id from Telegram Trigger.  
  - *Inputs:* Telegram Trigger  
  - *Outputs:* None  
  - *Failure Cases:* Telegram API unavailability, network errors.  

- **Развилка (Switch)**  
  - *Type:* Switch Node  
  - *Role:* Routes based on action prefix: "publish" or "retry".  
  - *Config:* Checks if action string starts with "publish" or "retry".  
  - *Inputs:* Get Post By Id  
  - *Outputs:*  
    - publish -> "Собираем пост" (Assemble post)  
    - retry -> "Данные для анализа (тут указываются ticker)" (Restart generation)  
  - *Failure Cases:* Unknown action strings, routing errors.  

---

#### 1.2 Market Data Retrieval & Technical Analysis

**Overview:**  
This block collects ticker information, authenticates to the trade API, fetches historical market data, and computes technical indicators to prepare for AI analysis.

**Nodes Involved:**  
- Данные для анализа (тут указываются ticker) [Ticker Setup]  
- Execute Auth Trade API [Broker API Auth]  
- Исторические данные [Historical Data Request]  
- Рассчет TA [Technical Analysis Calculation]  

**Node Details:**

- **Данные для анализа (тут указываются ticker)**  
  - *Type:* Code  
  - *Role:* Provides ticker and classCode either from input or defaults to GAZP, SBER, LKOH.  
  - *Config:* Returns array of tickers for batch processing.  
  - *Inputs:* Schedule Trigger or Switch (retry path)  
  - *Outputs:* Execute Auth Trade API  
  - *Failure Cases:* Missing or malformed input JSON.  

- **Execute Auth Trade API**  
  - *Type:* Execute Workflow  
  - *Role:* Runs sub-workflow to authenticate with broker trade API and obtain access token.  
  - *Config:* Passes ticker and classCode to sub-workflow; uses stored credentials.  
  - *Inputs:* Данные для анализа  
  - *Outputs:* Исторические данные  
  - *Failure Cases:* Auth failures, expired or invalid credentials, sub-workflow errors.  
  - *Sub-workflow:* "Trade API Auth" (ID: e4nUcFlYhUsQcGi8)  

- **Исторические данные**  
  - *Type:* HTTP Request  
  - *Role:* Fetches historical candle data for each ticker from broker API using access token.  
  - *Config:* Queries last ~60 days of hourly candles; uses Authorization header with Bearer token.  
  - *Inputs:* Execute Auth Trade API  
  - *Outputs:* Рассчет TA  
  - *Failure Cases:* HTTP errors, rate limits, invalid tokens, incomplete data.  

- **Рассчет TA**  
  - *Type:* Code  
  - *Role:* Calculates multiple technical indicators (RSI, SMA, EMA, MACD, Bollinger Bands, ADX) for each ticker's historical data.  
  - *Config:* Implements JavaScript functions for indicators; outputs context, last values, and derived signals.  
  - *Inputs:* Исторические данные  
  - *Outputs:* AI Agent  
  - *Failure Cases:* Insufficient data bars (<30), invalid input data, code exceptions.  

---

#### 1.3 AI Content Generation

**Overview:**  
Uses a LangChain AI Agent configured with a detailed prompt in Russian to generate a viral social media post summarizing the technical analysis in simple language. The AI response is strictly parsed as JSON.

**Nodes Involved:**  
- AI Agent  
- OpenRouter Chat Model  
- Structured Output Parser  

**Node Details:**

- **AI Agent**  
  - *Type:* LangChain Agent  
  - *Role:* Sends prompt with technical data context to AI; expects structured JSON with "title" and "summary".  
  - *Config:*  
    - System message defines expert role, style, JSON output format, and strict formatting rules.  
    - Prompt includes ticker, timeframe, and JSON technical data.  
    - Retries up to 3 times on failure.  
  - *Inputs:* Рассчет TA  
  - *Outputs:* ID Generation (success), Ошибка генерации (failure)  
  - *Failure Cases:* AI response parsing errors, invalid JSON output, timeout, network issues.  

- **OpenRouter Chat Model**  
  - *Type:* LangChain Chat Model  
  - *Role:* Provides underlying AI model (openai/gpt-oss-120b) for the AI Agent.  
  - *Config:* Uses OpenRouter API credentials.  
  - *Inputs:* AI Agent  
  - *Outputs:* Structured Output Parser  
  - *Failure Cases:* API limits, authentication failure.  

- **Structured Output Parser**  
  - *Type:* LangChain Output Parser  
  - *Role:* Parses AI output strictly as JSON with schema: `{ title: string, summary: string }`.  
  - *Config:* Auto-fix enabled to correct minor output errors.  
  - *Inputs:* OpenRouter Chat Model  
  - *Outputs:* AI Agent (final output)  
  - *Failure Cases:* Parsing failures if AI output is malformed beyond auto-fix.  

---

#### 1.4 Post Assembly & Storage

**Overview:**  
This block assembles the final post object for publishing, saves it to PostgreSQL, and sends it as a draft message to Telegram for review.

**Nodes Involved:**  
- ID Generation  
- Сохранение поста (Save Post)  
- Собираем пост (Assemble Post)  
- Отправка поста на валидацию (Send Post for Validation)  

**Node Details:**

- **ID Generation**  
  - *Type:* Crypto (Generate ID)  
  - *Role:* Generates a unique identifier to associate with the post for tracking.  
  - *Config:* Default ID generation (likely UUID).  
  - *Inputs:* AI Agent (success path)  
  - *Outputs:* Сохранение поста  
  - *Failure Cases:* Rare cryptographic errors, UUID collisions.  

- **Сохранение поста**  
  - *Type:* PostgreSQL Insert/Update  
  - *Role:* Inserts the generated post record into `analytics_history` table with id, title, ticker, summary, and classCode.  
  - *Config:* Uses mapped fields from AI Agent output and technical analysis context.  
  - *Inputs:* ID Generation  
  - *Outputs:* Отправка поста на валидацию  
  - *Failure Cases:* Database connectivity issues, constraint violations.  

- **Собираем пост**  
  - *Type:* Code  
  - *Role:* Constructs the HTTP POST payload for the Profit platform including title, content with hashtag, instruments, tags, and empty arrays for files and strategies.  
  - *Config:* Uses ticker, classCode, title, and summary from previous nodes.  
  - *Inputs:* Развилка (publish path)  
  - *Outputs:* Execute Auth Login  
  - *Failure Cases:* Missing or invalid fields, data formatting errors.  

- **Отправка поста на валидацию**  
  - *Type:* Telegram Message  
  - *Role:* Sends the AI-generated post draft to an admin Telegram chat with inline buttons for Publish and Retry.  
  - *Config:* Uses Telegram Bot credentials; buttons send callback_data with action and post id.  
  - *Inputs:* Сохранение поста  
  - *Outputs:*  
    - Success: none (awaits user action)  
    - Failure: Send a text message with error and Retry button  
  - *Failure Cases:* Telegram API errors, message formatting issues.  

---

#### 1.5 Telegram Moderation & Publishing

**Overview:**  
Handles user actions from Telegram inline buttons to either publish the post to the Profit platform or retry the generation. Publishes via API and sends success/failure notifications.

**Nodes Involved:**  
- Execute Auth Login (Profit API Auth)  
- Публикация постав в Профите (Publish Post to Profit)  
- Успешно опубликовано (Success Notification)  
- Ошибка публикации (Error Notification)  
- Send a text message (Retry notification)  

**Node Details:**

- **Execute Auth Login**  
  - *Type:* Execute Workflow  
  - *Role:* Authenticates to Profit platform API, obtaining access token for publishing.  
  - *Config:* Calls sub-workflow "BCS Login" (ID: gz5RhsAPZztWijnP).  
  - *Inputs:* Собираем пост  
  - *Outputs:* Публикация постав в Профите  
  - *Failure Cases:* Auth failures, expired credentials, sub-workflow errors.  

- **Публикация постав в Профите**  
  - *Type:* HTTP Request  
  - *Role:* Posts the assembled post to Profit platform newsfeed via multipart form-data with authorization header.  
  - *Config:* Sends post JSON in body parameter "body".  
  - *Inputs:* Execute Auth Login  
  - *Outputs:*  
    - Success: Успешно опубликовано  
    - Failure: Ошибка публикации  
  - *Failure Cases:* HTTP errors, API rejection, token expiration.  

- **Успешно опубликовано**  
  - *Type:* Telegram Message  
  - *Role:* Notifies admin Telegram chat of successful post publication.  
  - *Config:* Sends fixed text message "Пост успешно публикован".  
  - *Inputs:* Публикация постав в Профите (success)  
  - *Outputs:* None  
  - *Failure Cases:* Telegram API errors.  

- **Ошибка публикации**  
  - *Type:* Telegram Message  
  - *Role:* Notifies admin Telegram chat of publication failure.  
  - *Config:* Sends fixed text message "Ошибка публикации".  
  - *Inputs:* Публикация постав в Профите (failure) and Exist type and id (false path)  
  - *Outputs:* None  
  - *Failure Cases:* Telegram API errors.  

- **Send a text message**  
  - *Type:* Telegram Message  
  - *Role:* Sends error message with retry inline button if AI generation failed.  
  - *Config:* Includes ticker and error details, retry button with callback data.  
  - *Inputs:* Отправка поста на валидацию (failure)  
  - *Outputs:* None  
  - *Failure Cases:* Telegram API errors.  

---

### 3. Summary Table

| Node Name                                  | Node Type                              | Functional Role                                 | Input Node(s)                                  | Output Node(s)                                | Sticky Note                                                |
|--------------------------------------------|--------------------------------------|------------------------------------------------|-----------------------------------------------|-----------------------------------------------|------------------------------------------------------------|
| Schedule Trigger                           | Schedule Trigger                     | Triggers workflow twice daily                   | None                                          | Данные для анализа (тут указываются ticker)  | See block 1.1 description                                  |
| Telegram Trigger                           | Telegram Trigger                     | Listens for Telegram callback queries           | None                                          | Get Action Type, Query callback               | See block 1.1 description                                  |
| Get Action Type                           | Set                                 | Parses Telegram callback data into action/id    | Telegram Trigger                              | Exist type and id                             |                                                            |
| Exist type and id                         | If                                  | Validates presence of action and id             | Get Action Type                               | Get Post By Id / Ошибка публикации            |                                                            |
| Get Post By Id                           | PostgreSQL Select                   | Retrieves saved post by id                       | Exist type and id (true)                       | Развилка                                     |                                                            |
| Query callback                           | Telegram API                        | Acknowledges Telegram callback query            | Telegram Trigger                              | None                                          |                                                            |
| Развилка (Switch)                        | Switch                             | Routes based on action prefix                    | Get Post By Id                                | Собираем пост / Данные для анализа            |                                                            |
| Данные для анализа (тут указываются ticker) | Code                               | Provides ticker(s) for analysis                   | Schedule Trigger / Развилка (retry path)      | Execute Auth Trade API                         |                                                            |
| Execute Auth Trade API                   | Execute Workflow                   | Authenticates to broker API                       | Данные для анализа                            | Исторические данные                           | Sub-workflow "Trade API Auth"                             |
| Исторические данные                     | HTTP Request                      | Fetches historical market data                    | Execute Auth Trade API                         | Рассчет TA                                   |                                                            |
| Рассчет TA                             | Code                               | Calculates technical indicators                   | Исторические данные                           | AI Agent                                     |                                                            |
| AI Agent                              | LangChain Agent                   | Generates viral social media posts from TA data  | Рассчет TA                                    | ID Generation / Ошибка генерации              | Uses OpenRouter Chat Model and Structured Output Parser   |
| OpenRouter Chat Model                   | LangChain Model                   | AI model for generating content                   | AI Agent                                      | Structured Output Parser                       |                                                            |
| Structured Output Parser                | LangChain Output Parser           | Parses AI JSON output                              | OpenRouter Chat Model                         | AI Agent                                      |                                                            |
| ID Generation                         | Crypto                           | Generates unique ID for posts                      | AI Agent (success)                            | Сохранение поста                             |                                                            |
| Сохранение поста                       | PostgreSQL Insert/Update          | Saves post data to analytics_history table        | ID Generation                                 | Отправка поста на валидацию                   |                                                            |
| Отправка поста на валидацию            | Telegram Message                  | Sends draft post to Telegram admin with buttons   | Сохранение поста                              | Send a text message (on failure)              |                                                            |
| Send a text message                   | Telegram Message                  | Sends error message with retry option             | Отправка поста на валидацию (failure)         | None                                          |                                                            |
| Собираем пост                        | Code                             | Assembles final post object for Profit platform   | Развилка (publish path)                       | Execute Auth Login                            |                                                            |
| Execute Auth Login                   | Execute Workflow                 | Authenticates to Profit platform API              | Собираем пост                                 | Публикация постав в Профите                   | Sub-workflow "BCS Login"                                  |
| Публикация постав в Профите          | HTTP Request                    | Publishes post to Profit platform                  | Execute Auth Login                            | Успешно опубликовано / Ошибка публикации      |                                                            |
| Успешно опубликовано                 | Telegram Message                | Notifies admin of successful publication           | Публикация постав в Профите (success)          | None                                          |                                                            |
| Ошибка публикации                   | Telegram Message                | Notifies admin of publication failure              | Публикация постав в Профите (failure) / Exist type and id (false) | None                                          |                                                            |
| Sticky Note2                       | Sticky Note                     | Workflow overview and documentation                 | None                                          | None                                          | See below section 5 for full content                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Set to trigger twice daily at 11:15 and 18:15.

2. **Create a Telegram Trigger node**  
   - Type: Telegram Trigger  
   - Configure to listen to `callback_query` updates only.  
   - Attach Telegram Bot credentials.

3. **Create a Set node "Get Action Type"**  
   - Extract `action` and `id` from the Telegram callback data string by splitting on "::".

4. **Create an If node "Exist type and id"**  
   - Condition: both `action` and `id` fields exist and are non-empty.

5. **Create a PostgreSQL node "Get Post By Id"**  
   - Select one row from `analytics_history` where `id` equals input `id`.  
   - Configure PostgreSQL credentials accordingly.

6. **Create a Telegram node "Query callback"**  
   - Use callback_query.id to answer Telegram callback queries.

7. **Create a Switch node "Развилка"**  
   - Route based on `action` string prefix:  
     - If starts with "publish" → "Собираем пост"  
     - If starts with "retry" → "Данные для анализа (тут указываются ticker)"

8. **Create a Code node "Данные для анализа (тут указываются ticker)"**  
   - Provide default tickers with ticker and classCode (e.g., GAZP, SBER, LKOH).  
   - If inputs provide ticker and classCode, use them instead.

9. **Create an Execute Workflow node "Execute Auth Trade API"**  
   - Calls a sub-workflow that authenticates to the broker trade API.  
   - Pass ticker and classCode as parameters.  
   - Ensure credentials for broker API are set in sub-workflow.

10. **Create an HTTP Request node "Исторические данные"**  
    - GET candles data for the tickers from broker API endpoint.  
    - Use access token from previous node in Authorization header.  
    - Query last ~60 days of hourly data.

11. **Create a Code node "Рассчет TA"**  
    - Implement JavaScript code that calculates RSI, SMA, EMA, MACD, Bollinger Bands, ADX on historical bars.  
    - Handle inputs for multiple tickers.  
    - Output context and derived TA signals.

12. **Create an AI Agent node "AI Agent"**  
    - Use LangChain AI Agent node with:  
      - System prompt describing expert role and JSON output structure in Russian.  
      - Input prompt includes ticker, timeframe, and technical data JSON.  
      - Configure to retry 3 times on error.  
    - Connect to OpenRouter Chat Model and Structured Output Parser nodes.

13. **Create OpenRouter Chat Model node**  
    - Use OpenRouter API with model "openai/gpt-oss-120b".  
    - Link to AI Agent node.

14. **Create Structured Output Parser node**  
    - Define schema with `title` and `summary` strings.  
    - Enable auto-fix for minor JSON errors.

15. **Create a Crypto node "ID Generation"**  
    - Generate unique post ID (UUID recommended).

16. **Create a PostgreSQL node "Сохранение поста"**  
    - Insert or update `analytics_history` table with id, title, ticker, summary, classCode.  
    - Use correct column mappings.

17. **Create a Telegram node "Отправка поста на валидацию"**  
    - Send post title and summary as message to admin chat.  
    - Add inline keyboard with buttons "Опубликовать" (publish) and "Повторить" (retry) carrying callback_data with action and post id.

18. **Create a Telegram node "Send a text message"**  
    - On AI generation failure, send error message with retry button.

19. **Create a Code node "Собираем пост"**  
    - Assemble post JSON payload for Profit platform:  
      - Include title, content (summary plus hashtag), instruments (ticker and classCode), tags, and empty arrays for files and strategies.

20. **Create an Execute Workflow node "Execute Auth Login"**  
    - Calls a sub-workflow that authenticates to the Profit platform API.  
    - Ensure credentials for Profit platform OAuth2 are configured.

21. **Create an HTTP Request node "Публикация постав в Профите"**  
    - POST assembled post to Profit API endpoint with authorization header.  
    - Use multipart-form-data content type.

22. **Create Telegram nodes "Успешно опубликовано" and "Ошибка публикации"**  
    - Send success or failure notifications to admin Telegram chat.

23. **Connect nodes according to logical flow and set error handlers to continue or notify as in original workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Context or Link                                                                                                        |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------|
| # 📈 AI Stock Analytics & BCS "Profit" Social Network Publishing Workflow  This workflow automatically generates stock market insights for selected tickers (e.g. GAZP, SBER, LKOH) using historical data, technical indicators, and an AI model. The results are then sent to Telegram for quick moderation and publishing. Key features include multi-ticker support, automated error handling, human-in-the-loop moderation, PostgreSQL integration, and flexible extension options. It runs twice daily and supports a clean, engaging social media content format in Russian. | Sticky Note node "Sticky Note2" in the workflow provides this overview and can be used as a reference document.          |
| Telegram Bot API documentation is essential for understanding callback queries, inline keyboards, and message sending features.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | https://core.telegram.org/bots/api                                                                                     |
| The broker API endpoint for historical data requires valid OAuth2 tokens and supports querying candle data by ticker, classCode, timeframe, startDate, and endDate.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | Broker API docs (internal or provided by broker)                                                                        |
| LangChain AI nodes require OpenRouter or OpenAI API credentials. Structured output parsing ensures the AI response conforms to expected JSON schema, reducing errors in downstream processing.                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | https://docs.n8n.io/integrations/builtin/n8n-nodes-langchain/                                                          |
| PostgreSQL database must have table `analytics_history` with columns `id` (primary key), `title`, `ticker`, `summary`, `classCode`, and optionally `created_at`.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | PostgreSQL schema management best practices                                                                              |
| Sub-workflows "Trade API Auth" and "BCS Login" must be implemented separately with proper API authentication flows for broker and Profit platforms respectively. Credentials must be securely stored in n8n.                                                                                                                                                                                                                                                                                                                                                                                                                                                            | Sub-workflow IDs referenced: e4nUcFlYhUsQcGi8 and gz5RhsAPZztWijnP                                                      |

---

**Disclaimer:** The provided content is extracted exclusively from an automated n8n workflow and complies with all applicable content policies. It contains no illegal, offensive, or protected material and processes only legal and public data.