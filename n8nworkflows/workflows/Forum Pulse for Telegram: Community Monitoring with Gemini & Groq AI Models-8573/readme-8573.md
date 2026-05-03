Forum Pulse for Telegram: Community Monitoring with Gemini & Groq AI Models

https://n8nworkflows.xyz/workflows/forum-pulse-for-telegram--community-monitoring-with-gemini---groq-ai-models-8573


# Forum Pulse for Telegram: Community Monitoring with Gemini & Groq AI Models

### 1. Workflow Overview

This workflow, titled **"Forum Pulse for Telegram: Community Monitoring with Gemini & Groq AI Models"**, is designed to provide an intelligent and automated monitoring assistant for the n8n community forums, specifically Reddit r/n8n and the official n8n Community forum. It enables Telegram users to:

- Search for trending, recent, or relevant posts on both Reddit and n8n Community.
- Request deep dives into specific posts or topics by opening and summarizing detailed content including comments.
- Engage in casual chat with the bot.
- Receive daily digest summaries of hot n8n discussions through scheduled triggers.

The workflow is divided into the following logical blocks:

- **1.1 Daily Pulse Automation**: Scheduled retrieval of trending posts and auto-delivery of daily summaries to Telegram.
- **1.2 On-demand User Interaction**: Real-time Telegram message handling, intent detection, and routing.
- **1.3 Intent-Based Branching and Clarification**: Branching according to user intent (search, open link, chat) with confidence verification and clarification prompts.
- **1.4 Multi-Platform Search & Result Normalization**: Performing API queries to Reddit and n8n Community, normalizing and merging results.
- **1.5 Deep Dive Content Extraction and Summarization**: Fetching full post content and comments, summarizing them with AI.
- **1.6 Formatting and Telegram Delivery**: Preparing AI-generated text for Telegram, chunking large messages, and sending replies.
- **1.7 AI Models and Chat Memory**: Usage of Groq and Google Gemini AI models along with MongoDB chat memory nodes for context preservation.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Daily Pulse Automation

**Overview:**  
Automates the daily retrieval of top posts from Reddit and n8n Community and sends a summarized digest to a Telegram chat at a scheduled time.

**Nodes Involved:**  
- Schedule Trigger  
- Reddit (HTTP Request)  
- n8n Community Fetch (HTTP Request)  
- Get Only Search Result1 (Code)  
- Get Only Search Result2 (Code)  
- Merge Sources1 (Merge)  
- Wrap as Data Object2 (Code)  
- Set Memory ID Session (Set)  
- MongoDB Chat Memory2 (Memory)  
- Google Gemini Chat Model1 (AI)  
- AI Summarizer Overview (AI Agent)  
- Clean and Chunk (Code)  
- Send Auto Reply (Telegram)

**Node Details:**

- **Schedule Trigger**  
  - Triggers workflow every day at 8:00 AM.  
  - Ensures daily execution of the pulse automation.

- **Reddit (HTTP Request)**  
  - Requests top Reddit posts from r/n8n (last day, limit 20).  
  - Endpoint: `https://www.reddit.com/r/n8n/top.json?t=day&limit=20`.

- **n8n Community Fetch (HTTP Request)**  
  - Searches n8n Community posts newer than yesterday, ordered by latest, limit 10.  
  - Endpoint contains date calculation for `after:` filter.

- **Get Only Search Result1 / 2 (Code)**  
  - Processes raw JSON Reddit and n8n Community API responses, extracting detailed post info into standardized objects.

- **Merge Sources1 (Merge)**  
  - Combines Reddit and n8n Community search results into a unified dataset.

- **Wrap as Data Object2 (Code)**  
  - Packages combined data into a single JSON object (`data` key) for input to AI summarizer.

- **Set Memory ID Session (Set)**  
  - Sets a static Telegram user ID (`6163095869`) for chat memory session key.

- **MongoDB Chat Memory2 (Memory)**  
  - Loads chat memory from MongoDB collection `n8n_forum_update_chat` using the static session key.

- **Google Gemini Chat Model1 (AI)**  
  - Invokes Google Gemini AI to summarize the combined search data.

- **AI Summarizer Overview (AI Agent)**  
  - Uses AI to craft a friendly, concise summary of the daily top posts.

- **Clean and Chunk (Code)**  
  - Normalizes and splits AI output into Telegram-compatible chunks (max 2000 characters).

- **Send Auto Reply (Telegram)**  
  - Sends the daily summary message chunks to the Telegram chat with the configured bot.

**Edge Cases & Failure Types:**  
- API rate limits or downtime from Reddit or n8n Community may result in missing or partial data.  
- MongoDB connection issues can break chat memory retrieval.  
- Telegram API failures (invalid token, chat ID, or network errors) can prevent message delivery.  
- AI model errors or timeout may cause missing summarization.

---

#### 1.2 On-demand User Interaction

**Overview:**  
Handles incoming Telegram user messages in real-time, detects user intent (search, open link, chat), and routes accordingly.

**Nodes Involved:**  
- Telegram Trigger - User Message  
- MongoDB Chat Memory (Memory)  
- Groq Chat Model (AI Language Model)  
- Structured Output Parser (AI Output Parser)  
- Detect User Intent (AI Agent)  
- Branch by Intent (Switch)  
- Send Typing Action (Telegram)

**Node Details:**

- **Telegram Trigger - User Message**  
  - Listens for new Telegram messages.  
  - Provides the initial input (user's text and user ID).

- **MongoDB Chat Memory**  
  - Retrieves conversation memory from MongoDB keyed by the Telegram user ID.

- **Groq Chat Model**  
  - Runs Groq-based LLM (model `openai/gpt-oss-120b`) to analyze the message for intent.

- **Structured Output Parser**  
  - Parses AI model output into structured JSON with fields like intent, keyword, platform, sorting, time, confidence, etc.

- **Detect User Intent (AI Agent)**  
  - Applies complex AI prompt to classify intent as `search`, `open_link`, or `chat`.  
  - Infers search parameters and confidence score.  
  - Enforces normalized output schema.

- **Branch by Intent (Switch)**  
  - Routes workflow to different branches based on the detected intent and confidence value.

- **Send Typing Action (Telegram)**  
  - Sends "typing..." indicator to Telegram user for better UX.

**Edge Cases & Failure Types:**  
- Parsing errors if AI output deviates from expected schema.  
- Low-confidence intent (<0.7) triggers clarification flow.  
- Telegram message format issues or missing text.  
- MongoDB or AI service outages could halt processing.

---

#### 1.3 Intent-Based Branching and Clarification

**Overview:**  
Routes the workflow differently based on user intent and confidence; includes clarification when confidence is low.

**Nodes Involved:**  
- Branch by Intent (Switch)  
- Platform? (Switch)  
- Has keyword? (If)  
- Has time? (If)  
- AI Summarizer Clarify (AI Agent)  
- MongoDB Chat Memory3 (Memory)  
- Google Gemini Chat Model2 (AI)  
- Clean and Chunk2 (Code)  
- Send reply1 (Telegram)

**Node Details:**

- **Branch by Intent**  
  - For `search` intent: passes to Platform? node.  
  - For `open_link` intent: passes to Open link Reddit or n8n Community? node.  
  - For `chat` intent: passes to Wrap as Data Object node for chat response.  
  - For confidence < 0.7: moves to AI Summarizer Clarify.

- **Platform? (Switch)**  
  - Determines which platform(s) to query: Reddit, n8n Community, or both.

- **Has keyword? (If Node)**  
  - Checks if a keyword is provided to decide the search URL pattern.

- **Has time? (If Node)**  
  - Checks if a time filter is set to choose between different Reddit API endpoints.

- **AI Summarizer Clarify (AI Agent)**  
  - When confidence is low, asks the user to clarify intent or parameters with short, friendly Telegram messages and clickable options.

- **MongoDB Chat Memory3 (Memory)**  
  - Manages chat memory for the clarification interaction.

- **Google Gemini Chat Model2 (AI)**  
  - Generates clarification prompt in Telegram HTML format.

- **Clean and Chunk2 (Code)**  
  - Normalizes and chunks clarification message for Telegram.

- **Send reply1 (Telegram)**  
  - Sends the clarification message back to the user.

**Edge Cases & Failure Types:**  
- User silence or invalid inputs during clarification may stall the flow.  
- Telegram UI constraints require strict HTML formatting.  
- Failure to update state or memory can cause repeated clarifications.

---

#### 1.4 Multi-Platform Search & Result Normalization

**Overview:**  
Executes API calls to Reddit and n8n Community, processes JSON responses, normalizes and merges search results.

**Nodes Involved:**  
- Reddit Search page JSON (has time) (HTTP Request)  
- Reddit Search page (no time) (HTTP Request)  
- Reddit Search page JSON (no keyword) (HTTP Request)  
- n8n Community Fetch JSON (HTTP Request)  
- Get Only Reddit Search Result (Code)  
- Get Only n8n community Search Result (Code)  
- Merge Search Result (Merge)  
- Merge Sources1 (Merge)  
- Wrap as Data Object (Code)

**Node Details:**

- **Reddit Search page JSON nodes**  
  - Different Reddit API endpoints based on presence of keyword and time filters.  
  - Fetch posts from r/n8n subreddit with sorting and time constraints.

- **n8n Community Fetch JSON**  
  - Queries n8n Community forum search API with keyword, time, and sort filters.

- **Get Only Reddit Search Result / Get Only n8n community Search Result**  
  - JavaScript code nodes extract and transform raw API data into structured, uniform objects including post ID, title, author, URL, timestamps, score, etc.

- **Merge Search Result / Merge Sources1**  
  - Merge nodes combine results from multiple sources by position for unified processing.

- **Wrap as Data Object**  
  - Packages merged results into a single JSON array under `data` key.

**Edge Cases & Failure Types:**  
- Changes in Reddit or n8n Community API or HTML structure may break extraction logic.  
- Empty or malformed API responses.  
- Rate limiting on Reddit or n8n Community.  
- Network timeouts or errors.

---

#### 1.5 Deep Dive Content Extraction and Summarization

**Overview:**  
For open link requests, fetches complete post content and all comments from Reddit or n8n Community, merges them, and generates a detailed AI summary.

**Nodes Involved:**  
- Open link Reddit or n8n Community? (Switch)  
- If Reddit? (HTTP Request)  
- Get Post Content (HTML Extract)  
- Comment (HTTP Request)  
- Get Comment (HTML Extract)  
- Comment Summary 1 (Code)  
- If n8n Community? (HTTP Request)  
- Get Topic Content (HTML Extract)  
- Get Comment1 (HTML Extract)  
- Comment Summary 2 (Code)  
- Merge Link Content (Merge)  
- Wrap as Data Object (Code)  
- MongoDB Chat Memory1 (Memory)  
- Google Gemini Chat Model (AI)  
- AI Summarizer Deep-dive (AI Agent)  
- Format for Telegram Output (Code)  
- Send reply (Telegram)

**Node Details:**

- **Open link Reddit or n8n Community?**  
  - Switch node to route requests based on URL domain.

- **If Reddit?**  
  - HTTP request node fetching Reddit post page HTML.

- **Get Post Content**  
  - HTML extraction node extracting post metadata: title, author, content, upvotes, comment count, flair, timestamps.

- **Comment (HTTP Request)**  
  - HTTP request to fetch Reddit comments page.

- **Get Comment**  
  - Extracts comment data: authors, content (HTML), upvotes, nesting level, IDs.

- **Comment Summary 1 (Code)**  
  - Cleans and structures Reddit comments into arrays with stripped HTML.

- **If n8n Community?**  
  - Routes to n8n Community post and comments fetching.

- **Get Topic Content**  
  - Extracts topic title, category, original poster, content, likes, post time.

- **Get Comment1**  
  - Extracts comments with authors, content, likes, post number, timestamps.

- **Comment Summary 2 (Code)**  
  - Processes n8n Community comments, strips HTML, structures data.

- **Merge Link Content (Merge)**  
  - Combines post content and comments into a single dataset.

- **Wrap as Data Object (Code)**  
  - Packages merged data into JSON for AI input.

- **MongoDB Chat Memory1 (Memory)**  
  - Loads chat memory for the user session.

- **Google Gemini Chat Model (AI)**  
  - Sends extracted data to AI for summarization.

- **AI Summarizer Deep-dive (AI Agent)**  
  - Creates detailed replies with insights, summaries, and structured info.

- **Format for Telegram Output (Code)**  
  - Normalizes and chunks AI output for Telegram HTML compliance.

- **Send reply (Telegram)**  
  - Sends formatted deep-dive summaries to the user.

**Edge Cases & Failure Types:**  
- Forum HTML structure changes break extraction rules.  
- Missing or malformed comment data.  
- Network or API failures.  
- Large content requiring chunking to avoid Telegram limits.  
- MongoDB or AI service issues.

---

#### 1.6 Formatting and Telegram Delivery

**Overview:**  
Prepares AI-generated textual responses for Telegram by converting Markdown to Telegram HTML, sanitizing, and chunking messages to fit Telegram's 2000-character limit.

**Nodes Involved:**  
- Clean and Chunk (Code)  
- Clean and Chunk2 (Code)  
- Format for Telegram Output (Code)  
- Send Auto Reply (Telegram)  
- Send reply (Telegram)  
- Send reply1 (Telegram)

**Node Details:**

- **Clean and Chunk / Clean and Chunk2**  
  - JavaScript nodes performing:  
    - Markdown to Telegram HTML conversion (b, i, u, s, code, pre, a, br, blockquote).  
    - Sanitizing disallowed tags and attributes.  
    - Splitting large text into safe chunks without breaking HTML tags.  
    - Special handling for links, code blocks, and blockquotes to maintain formatting.

- **Format for Telegram Output**  
  - Similar normalization and chunking logic for deep-dive responses.

- **Send Auto Reply / Send reply / Send reply1**  
  - Telegram nodes sending text chunks to appropriate chat IDs with HTML parse mode enabled.  
  - Some send replies referencing the original Telegram message (reply_to_message_id).

**Edge Cases & Failure Types:**  
- Large or complex HTML may sometimes cause formatting issues.  
- Telegram API may reject messages if invalid HTML or too large.  
- Network or credential errors.

---

#### 1.7 AI Models and Chat Memory

**Overview:**  
Leverages AI language models (Groq and Google Gemini) for intent detection, summarization, and clarification, while maintaining conversational context using MongoDB.

**Nodes Involved:**  
- Groq Chat Model (AI Language Model)  
- Google Gemini Chat Model / Google Gemini Chat Model1 / Google Gemini Chat Model2 (AI Language Models)  
- MongoDB Chat Memory / MongoDB Chat Memory1 / MongoDB Chat Memory2 / MongoDB Chat Memory3 (Memory)

**Node Details:**

- **Groq Chat Model**  
  - Used in intent detection stage to parse user messages into structured intent JSON.

- **Google Gemini Chat Models**  
  - Used for different AI tasks: overview summarization, deep-dive summarization, and clarification prompts.

- **MongoDB Chat Memory Nodes**  
  - Store and retrieve conversation history keyed by user session (Telegram user ID).  
  - Support context-aware AI responses and maintain state across user interactions.

**Edge Cases & Failure Types:**  
- AI service outages or rate limiting can halt processing.  
- MongoDB connection failures impact memory persistence.  
- Version-specific API changes or credential misconfiguration.

---

### 3. Summary Table

| Node Name                      | Node Type                              | Functional Role                            | Input Node(s)                         | Output Node(s)                              | Sticky Note                                         |
|--------------------------------|--------------------------------------|--------------------------------------------|-------------------------------------|---------------------------------------------|-----------------------------------------------------|
| Schedule Trigger               | Schedule Trigger                     | Triggers daily pulse automation             | -                                   | Reddit, n8n Community Fetch                 | ## 1. Daily Pulse Automation - Runs every morning 8AM, fetches top posts, summarizes, sends to Telegram. |
| Reddit                        | HTTP Request                        | Fetches top Reddit posts                     | Schedule Trigger                    | Get Only Search Result1                      |                                                     |
| n8n Community Fetch           | HTTP Request                        | Fetches recent n8n Community posts           | Schedule Trigger                    | Get Only Search Result2                      |                                                     |
| Get Only Search Result1        | Code                               | Extracts Reddit posts data                    | Reddit                             | Merge Sources1                              |                                                     |
| Get Only Search Result2        | Code                               | Extracts n8n Community posts data             | n8n Community Fetch                | Merge Sources1                              |                                                     |
| Merge Sources1                | Merge                              | Combines Reddit and n8n Community results    | Get Only Search Result1, Get Only Search Result2 | Wrap as Data Object2                         |                                                     |
| Wrap as Data Object2           | Code                               | Packages combined data for AI input           | Merge Sources1                    | Set Memory ID Session                        |                                                     |
| Set Memory ID Session          | Set                                | Sets static Telegram user ID for memory key  | Wrap as Data Object2              | MongoDB Chat Memory2                         |                                                     |
| MongoDB Chat Memory2           | MongoDB Chat Memory                 | Loads memory context for daily pulse          | Set Memory ID Session             | Google Gemini Chat Model1                    |                                                     |
| Google Gemini Chat Model1      | AI Language Model                  | Summarizes daily posts overview               | MongoDB Chat Memory2              | AI Summarizer Overview                       |                                                     |
| AI Summarizer Overview         | AI Agent                          | Creates daily summary reply                    | Google Gemini Chat Model1         | Clean and Chunk                             |                                                     |
| Clean and Chunk               | Code                               | Normalizes and chunks text for Telegram       | AI Summarizer Overview            | Send Auto Reply                             |                                                     |
| Send Auto Reply               | Telegram                          | Sends daily summary to Telegram chat          | Clean and Chunk                  | -                                           |                                                     |
| Telegram Trigger - User Message| Telegram Trigger                  | Receives user messages in real-time           | -                               | MongoDB Chat Memory, Detect User Intent     | ## 2. On-demand User Interaction - Listens to user queries, intent detection, routing. |
| MongoDB Chat Memory            | MongoDB Chat Memory                | Loads chat memory for user session             | Telegram Trigger - User Message  | Groq Chat Model                              |                                                     |
| Groq Chat Model               | AI Language Model                  | Detects user intent and parameters             | MongoDB Chat Memory              | Structured Output Parser                     |                                                     |
| Structured Output Parser       | AI Output Parser                  | Parses AI JSON output into structured format  | Groq Chat Model                 | Detect User Intent                          |                                                     |
| Detect User Intent             | AI Agent                          | Classifies user intent, infers parameters      | Structured Output Parser         | Branch by Intent, Send Typing Action        |                                                     |
| Branch by Intent              | Switch                           | Routes based on intent and confidence          | Detect User Intent              | Platform?, Wrap as Data Object, Open link Reddit or n8n Community?, AI Summarizer Clarify |                                                     |
| Send Typing Action            | Telegram                          | Sends typing indicator on Telegram             | Detect User Intent              | -                                           |                                                     |
| Platform?                     | Switch                           | Determines which platform(s) to query           | Branch by Intent               | Reddit Search or n8n Community Fetch or both | ## 3. Multi-Platform Search & Merge - Routes queries, fetches data, merges results. |
| Has keyword?                  | If                               | Checks if search keyword exists                 | Platform?                     | Reddit Search page JSON (no keyword) or Has time?  |                                                     |
| Has time?                     | If                               | Checks if time filter exists                     | Has keyword?                  | Reddit Search page JSON (has time) or Reddit Search page (no time) |                                                     |
| Reddit Search page JSON (has time) | HTTP Request                  | Reddit search with time filter                   | Has time? (true)              | Get Only Reddit Search Result                |                                                     |
| Reddit Search page (no time)  | HTTP Request                    | Reddit search without time filter                | Has time? (false)             | Get Only Reddit Search Result                |                                                     |
| Reddit Search page JSON (no keyword) | HTTP Request                | Reddit search without keyword                     | Has keyword? (false)          | Get Only Reddit Search Result                |                                                     |
| Get Only Reddit Search Result | Code                             | Extracts Reddit search results                   | Reddit Search page JSON nodes | Merge Search Result                          |                                                     |
| n8n Community Fetch JSON      | HTTP Request                    | Fetches n8n community search results             | Platform?                    | Get Only n8n community Search Result        |                                                     |
| Get Only n8n community Search Result | Code                        | Extracts n8n community search results             | n8n Community Fetch JSON      | Merge Search Result                          |                                                     |
| Merge Search Result           | Merge                            | Merges Reddit and n8n community search results | Get Only Reddit Search Result, Get Only n8n community Search Result | Wrap as Data Object                          |                                                     |
| Wrap as Data Object           | Code                             | Wraps merged search results for AI summarization | Merge Search Result          | AI Summarizer Deep-dive or AI Summarizer Overview or Wrap as Data Object (chat) |                                                     |
| Open link Reddit or n8n Community? | Switch                     | Routes open link requests by domain              | Branch by Intent             | If Reddit? or If n8n Community?              | ## 4. Deep Dive into Post Details - Fetches full post and comments, summarizes and delivers. |
| If Reddit?                   | HTTP Request                    | Fetches Reddit post page HTML                      | Open link Reddit or n8n Community? | Get Post Content                          |                                                     |
| Get Post Content              | HTML Extract                   | Extracts post metadata from Reddit HTML           | If Reddit?                   | Comment, Merge Link Content                  |                                                     |
| Comment                      | HTTP Request                   | Fetches Reddit comments HTML                        | Get Post Content             | Get Comment                                  |                                                     |
| Get Comment                  | HTML Extract                   | Extracts Reddit comments                            | Comment                     | Comment Summary 1                            |                                                     |
| Comment Summary 1            | Code                           | Cleans and structures Reddit comments              | Get Comment                  | Merge Link Content                           |                                                     |
| If n8n Community?             | HTTP Request                    | Fetches n8n community topic page HTML              | Open link Reddit or n8n Community? | Get Topic Content                       |                                                     |
| Get Topic Content            | HTML Extract                   | Extracts n8n community post metadata                | If n8n Community?            | Get Comment1                                 |                                                     |
| Get Comment1                 | HTML Extract                   | Extracts n8n community comments                      | Get Topic Content            | Comment Summary 2                            |                                                     |
| Comment Summary 2            | Code                           | Cleans and structures n8n community comments        | Get Comment1                 | Merge Link Content                           |                                                     |
| Merge Link Content           | Merge                          | Combines post content and comments                   | Get Post Content, Comment Summary 1 or Get Topic Content, Comment Summary 2 | Wrap as Data Object                        |                                                     |
| MongoDB Chat Memory1          | MongoDB Chat Memory             | Loads chat memory for deep dive session             | Wrap as Data Object          | Google Gemini Chat Model                      |                                                     |
| Google Gemini Chat Model      | AI Language Model              | Summarizes deep-dive post and comments              | MongoDB Chat Memory1         | AI Summarizer Deep-dive                       |                                                     |
| AI Summarizer Deep-dive       | AI Agent                      | Creates detailed summary reply for deep dive         | Wrap as Data Object          | Format for Telegram Output                    |                                                     |
| Format for Telegram Output    | Code                           | Normalizes and chunks deep-dive AI output            | AI Summarizer Deep-dive      | Send reply                                   |                                                     |
| Send reply                   | Telegram                      | Sends deep-dive summary to Telegram chat             | Format for Telegram Output   | -                                           |                                                     |
| AI Summarizer Clarify         | AI Agent                      | Generates clarification prompts for low-confidence intents | Branch by Intent          | Clean and Chunk2                             | ## 2.1 Confidence Check & Verification (Required when confidence < 0.7) |
| Clean and Chunk2              | Code                           | Normalizes and chunks clarification messages          | AI Summarizer Clarify        | Send reply1                                  |                                                     |
| Send reply1                  | Telegram                      | Sends clarification reply to Telegram user            | Clean and Chunk2             | -                                           |                                                     |
| MongoDB Chat Memory3          | MongoDB Chat Memory             | Loads chat memory for clarification interactions       | AI Summarizer Clarify        | Google Gemini Chat Model2                     |                                                     |
| Google Gemini Chat Model2     | AI Language Model              | Supports clarification prompt generation               | MongoDB Chat Memory3         | AI Summarizer Clarify                         |                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Configure Credentials:**  
   - Set up Telegram Bot credentials with your Telegram API key.  
   - Set up MongoDB credentials pointing to your database for chat memory.  
   - Configure Groq API credentials for Groq Chat Model.  
   - Configure Google Palm API credentials for Google Gemini Chat Models.

2. **Create Daily Pulse Automation Block:**  
   - Add a **Schedule Trigger** node set to run daily at 8:00 AM.  
   - Add **HTTP Request** nodes to fetch Reddit top posts (`https://www.reddit.com/r/n8n/top.json?t=day&limit=20`) and n8n Community posts (with dynamic date filters).  
   - Add **Code** nodes ("Get Only Search Result1" and "Get Only Search Result2") to parse and transform HTTP responses into detailed post objects.  
   - Add a **Merge** node ("Merge Sources1") to combine Reddit and n8n Community results by position.  
   - Add a **Code** node ("Wrap as Data Object2") to package merged results under a `data` key.  
   - Add a **Set** node ("Set Memory ID Session") to assign a fixed Telegram user ID for sending the daily digest.  
   - Add a **MongoDB Chat Memory** node to load memory using the above session key.  
   - Add a **Google Gemini Chat Model** node to summarize the combined data.  
   - Add an **AI Agent** node ("AI Summarizer Overview") with a prompt to create a friendly summary.  
   - Add a **Code** node ("Clean and Chunk") to convert AI output to Telegram HTML and split into ≤2000 char chunks.  
   - Add a **Telegram** node ("Send Auto Reply") to send the chunked summaries to the Telegram chat.

3. **Create On-demand User Interaction Block:**  
   - Add a **Telegram Trigger** node to listen for user messages.  
   - Add a **MongoDB Chat Memory** node to load chat memory by Telegram user ID.  
   - Add a **Groq Chat Model** node to analyze user text and infer intent.  
   - Add a **Structured Output Parser** node to parse AI output JSON.  
   - Add an **AI Agent** node ("Detect User Intent") with a prompt that classifies intent (`search`, `open_link`, `chat`), infers parameters, and sets confidence.  
   - Add a **Switch** node ("Branch by Intent") to route based on intent and confidence.  
   - Add a **Telegram** node ("Send Typing Action") to show typing indicator.

4. **Build Intent-Based Branching and Clarification:**  
   - For `search` intent: add **Switch** node ("Platform?") to route to Reddit, n8n Community, or both.  
   - Add **If** nodes ("Has keyword?" and "Has time?") to determine API query construction.  
   - Add corresponding **HTTP Request** nodes for Reddit and n8n Community search APIs with dynamic URLs based on parameters.  
   - Add **Code** nodes to extract and normalize search results.  
   - Add **Merge** nodes to combine results.  
   - Add **Wrap as Data Object** node to package search results for AI summarization.  
   - For `open_link` intent: add **Switch** node ("Open link Reddit or n8n Community?") to route by URL domain.  
   - For Reddit links: add HTTP Request to fetch post and comments pages, HTML Extract nodes to extract post content and comments, and Code nodes to clean and structure comments.  
   - For n8n Community links: similar HTTP and HTML Extract nodes for topic and comments, with cleaning Code nodes.  
   - Merge post and comments, wrap as data object, load chat memory, then summarize with AI models.  
   - Add AI Agent node ("AI Summarizer Deep-dive") for detailed summary generation.  
   - Add Code node ("Format for Telegram Output") to sanitize and chunk text.  
   - Add Telegram node to send reply.

5. **Add Clarification Flow:**  
   - For confidence < 0.7, add AI Agent node ("AI Summarizer Clarify") that produces quick clarification prompts in Telegram HTML.  
   - Add Code node ("Clean and Chunk2") for formatting.  
   - Add Telegram node ("Send reply1") to deliver clarification.  
   - Use MongoDB Chat Memory nodes to maintain state during clarification.

6. **Add Chat Response Branch:**  
   - For `chat` intent, wrap incoming data as a data object, run AI summarizer for chat, format, and send reply.

7. **Implement Message Formatting and Chunking:**  
   - Use provided JavaScript code (Clean and Chunk, Format for Telegram Output) for all AI-generated text to convert Markdown to Telegram HTML, sanitize, and split messages safely.

8. **Test and Validate:**  
   - Test Telegram message reception, intent detection accuracy, search and open link flows, daily pulse delivery, clarification prompts, and chunked message sending.  
   - Monitor logs for API errors or parsing failures.  
   - Adjust prompts or extraction CSS selectors if forum layout changes.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Context or Link                                                                                                                         |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------|
| # AI-Powered n8n Forum Assistant for Telegram using Gemini & Groq  Author: Nguyen Thieu Toan  Full guide: [https://nguyenthieutoan.com/share-n8n-workflow-ai-power-n8n-forum-assistant-for-telegram](https://nguyenthieutoan.com/share-n8n-workflow-ai-power-n8n-forum-assistant-for-telegram)  This workflow instantly gathers and summarizes latest posts and discussions from Reddit and n8n Community, supports deep dives, and replies in Vietnamese or English with a modern style. It targets n8n users and community teams to stay updated effortlessly. Prerequisites include an n8n instance, Telegram Bot, MongoDB for memory, and user Telegram ID. The workflow emphasizes confidence-based clarification, multi-platform querying, and Telegram-friendly output formatting. Troubleshooting tips include checking API tokens, handling message length, and updating extraction rules if forum layouts change. Support and feedback available through n8n Community and author contact. | Project overview and detailed workflow guide by Nguyen Thieu Toan. [Link](https://nguyenthieutoan.com/share-n8n-workflow-ai-power-n8n-forum-assistant-for-telegram) |
| On-demand User Interaction block listens for Telegram user messages, detects intent (search | open_link | chat), provides typing indicators for UX, and routes actions accordingly. Confidence check ensures accuracy with clarifications when needed. This design reflects best practices in automation UX and user-first workflow engineering.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Sticky note describing user interaction design and intent detection.                                                                  |
| Confidence Check & Verification ensures low-confidence AI intent results (<0.7) are clarified before processing. This avoids inaccurate or premature answers and improves trust. The AI agent asks concise questions with clickable options in Telegram.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Sticky note explaining confidence verification logic.                                                                                  |
| Multi-Platform Search & Merge handles routing queries to Reddit or n8n Community APIs, dynamically builds API calls with parameters, normalizes data, and merges results into unified datasets. This design ensures consistent and precise data retrieval across platforms.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Sticky note detailing multi-platform search and merging strategy.                                                                      |
| Deep Dive into Post Details fetches full post and comment content from Reddit or n8n Community, extracts structured data, merges all relevant info, and passes them to AI for detailed summarization. This approach ensures no important context is missed for user requests to open/view specific posts.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Sticky note about deep dive content extraction and summarization.                                                                     |
| Summary, Formatting, and Delivery prepares AI output by cleaning, normalizing, converting markdown to Telegram HTML, splitting into chunks ≤2000 characters, and sending replies via Telegram bot. This ensures readable, well-formatted, and user-friendly messages.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 | Sticky note describing final formatting and message delivery steps.                                                                   |

---

**Disclaimer:** The text above is exclusively derived from an automated workflow created using n8n, respecting all current content policies. It contains no illegal, offensive, or protected content. All data handled is legal and public.