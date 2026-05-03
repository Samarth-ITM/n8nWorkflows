Daily AI News Summaries with Gemini 2.5 Flash to Telegram

https://n8nworkflows.xyz/workflows/daily-ai-news-summaries-with-gemini-2-5-flash-to-telegram-6095


# Daily AI News Summaries with Gemini 2.5 Flash to Telegram

---

### 1. Workflow Overview

This workflow automates the daily curation and delivery of AI technology news summaries by integrating real-time news scraping, AI-based analysis, and messaging. It is designed to:

- Continuously monitor and fetch new AI news articles from a specified RSS feed.
- Extract full article content using AI-powered scraping.
- Use Google’s Gemini 2.5 language model (via Langchain integration) to synthesize and generate a concise, well-structured daily briefing.
- Format the briefing for Telegram-compatible HTML and deliver it as a message to a designated Telegram chat.

**Target Use Cases:**

- Automated daily briefings for AI researchers, analysts, or enthusiasts.
- Centralized digest of news from multiple AI news sources.
- Integration into team communication platforms for up-to-date information sharing.

**Logical Blocks:**

- **1.1 Input Reception:** Trigger and fetch new articles from the AI-News RSS feed.
- **1.2 Content Extraction:** Retrieve full article content from each news link using Jina AI.
- **1.3 AI Processing & Report Generation:** Use Gemini 2.5 LLM with a custom prompt to analyze and generate a structured news summary.
- **1.4 Delivery:** Send the formatted briefing as a Telegram message.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block initiates the workflow by polling the AI-News RSS feed every minute to detect new articles. It outputs metadata including article titles and links.

- **Nodes Involved:**  
  - AI-News Feed (RSS Feed Read Trigger)

- **Node Details:**

  - **AI-News Feed**  
    - Type: RSS Feed Read Trigger  
    - Role: Polls the RSS feed at `https://www.artificialintelligence-news.com/feed/` every minute for new items.  
    - Configuration: Poll mode set to every minute; no filters applied.  
    - Inputs: None (trigger node)  
    - Outputs: Emits new article metadata (title, link, publication date, etc.)  
    - Version Requirements: Standard RSS trigger, no special version dependencies.  
    - Potential Failures: Network errors, invalid feed URL, feed downtime.  
    - Sticky Note: Describes this as the workflow start, triggering automatically every minute.

#### 1.2 Content Extraction

- **Overview:**  
  Processes each new article link from the RSS feed to fetch full article content using Jina AI’s scraping capabilities.

- **Nodes Involved:**  
  - Read News from AI News Website

- **Node Details:**

  - **Read News from AI News Website**  
    - Type: Jina AI Node  
    - Role: Scrapes or retrieves the full textual content of the article from the provided URL.  
    - Configuration: Uses the `link` field from the RSS feed as input URL; no additional options set.  
    - Inputs: Receives article metadata including the link from AI-News Feed node.  
    - Outputs: Article full content, cleaned and ready for analysis.  
    - Credentials: Requires Jina AI API credentials.  
    - Potential Failures: Link unreachable, content extraction failure, API rate limits or timeouts.  
    - Sticky Note: Explains it scrapes full article text using Jina AI.

#### 1.3 AI Processing & Report Generation

- **Overview:**  
  Synthesizes the scraped news articles into a concise, insightful report using the Gemini 2.5 language model via Langchain. The prompt instructs the AI to focus on key news, themes, and formatting compatible with Telegram HTML.

- **Nodes Involved:**  
  - Gemini 2.5 Flash  
  - Generate Report

- **Node Details:**

  - **Gemini 2.5 Flash**  
    - Type: Langchain Google Gemini Chat Node (lmChatGoogleGemini)  
    - Role: Provides AI language model capabilities to process input data and generate text.  
    - Configuration: Default options; references Google Gemini API credentials for authentication.  
    - Inputs: Receives prompt and input data from Generate Report node as an AI language model input.  
    - Outputs: AI-generated text (structured briefing).  
    - Credentials: Google Palm API credentials required for Gemini access.  
    - Potential Failures: API quota exceeded, authentication errors, network issues, model timeouts.  

  - **Generate Report**  
    - Type: Langchain Chain LLM Node  
    - Role: Constructs a detailed prompt embedding scraped article content from two sources, instructing the Gemini model to produce a structured daily intelligence briefing with specific formatting rules for Telegram.  
    - Configuration:  
      - The prompt includes placeholders for article title, content, and publish date.  
      - Output restricted to a subset of HTML tags supported by Telegram (<b>, <i>, <u>, <s>, <code>, <pre>, <a>), with lists formed by hyphenated lines.  
      - The current date is dynamically inserted in British English format.  
    - Inputs: Receives article content from “Read News from AI News Website” and AI model output from Gemini 2.5 Flash.  
    - Outputs: Final formatted briefing text for Telegram.  
    - Version: Version 1.7 of the Langchain Chain LLM node.  
    - Potential Failures: Expression evaluation errors, prompt formatting issues, incomplete input data, model response inconsistencies.  
    - Sticky Note: Described as the “brain” of the workflow that uses Gemini to generate the report.

#### 1.4 Delivery

- **Overview:**  
  Sends the final AI-generated news briefing as an HTML-formatted message to a predefined Telegram chat.

- **Nodes Involved:**  
  - Send a text message

- **Node Details:**

  - **Send a text message**  
    - Type: Telegram Node  
    - Role: Sends a text message to a Telegram chat via bot API.  
    - Configuration:  
      - Message text set dynamically from the AI-generated report (`={{ $json.text }}`).  
      - Chat ID set as `CHAT_ID` (to be replaced with actual destination chat).  
      - Parse mode set to HTML to enable formatted output.  
    - Inputs: Receives the final text from the Generate Report node.  
    - Credentials: Requires Telegram API credentials (bot token).  
    - Potential Failures: Invalid chat ID, bot permissions missing, network errors, message size limits.  
    - Sticky Note: Final step delivering the briefing to Telegram.

---

### 3. Summary Table

| Node Name                     | Node Type                         | Functional Role                          | Input Node(s)               | Output Node(s)           | Sticky Note                                                                                 |
|-------------------------------|----------------------------------|----------------------------------------|-----------------------------|--------------------------|---------------------------------------------------------------------------------------------|
| AI-News Feed                  | RSS Feed Read Trigger             | Triggers on new AI-News RSS articles   | None                        | Read News from AI News Website | Starting point; triggers every minute; fetches new articles from RSS feed.                 |
| Read News from AI News Website| Jina AI Node                     | Extracts full article content           | AI-News Feed                | Generate Report           | Scrapes full article content using Jina AI from article link.                              |
| Gemini 2.5 Flash              | Langchain Google Gemini Chat Node| Provides AI model processing             | Generate Report (ai_languageModel input) | Generate Report (ai_languageModel output) | Uses Gemini 2.5 LLM for text generation; requires Google Palm API credentials.             |
| Generate Report               | Langchain Chain LLM Node          | Creates structured AI news summary      | Read News from AI News Website, Gemini 2.5 Flash  | Send a text message      | “Brain” node generating final formatted report; uses prompt with Telegram HTML formatting.|
| Send a text message           | Telegram Node                    | Delivers report to Telegram chat        | Generate Report             | None                     | Sends AI-generated briefing to Telegram chat with HTML parse mode enabled.                 |
| Sticky Note                  | Sticky Note                      | Documentation and explanations           | None                        | None                     | Various sticky notes describe each major block and nodes in detail.                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the RSS Feed Trigger Node:**  
   - Add a node of type **RSS Feed Read Trigger**.  
   - Configure the feed URL as: `https://www.artificialintelligence-news.com/feed/`.  
   - Set polling to occur **every minute**.  
   - Name this node: `AI-News Feed`.

2. **Add Jina AI Node for Article Content Extraction:**  
   - Add a node of type **Jina AI**.  
   - Set the **URL** parameter to: `={{ $json.link }}` to dynamically use the article link from RSS feed.  
   - Leave other options default.  
   - Connect the output of `AI-News Feed` to this node’s input.  
   - Provide **Jina AI API Credentials** under the node’s credentials section.  
   - Name this node: `Read News from AI News Website`.

3. **Set up Gemini 2.5 Flash Node (Google Gemini LLM):**  
   - Add a node of type **Langchain Google Gemini Chat Node** (`lmChatGoogleGemini`).  
   - No special parameters needed under options.  
   - Assign valid **Google Palm API Credentials** to enable Gemini access.  
   - Name this node: `Gemini 2.5 Flash`.

4. **Add Langchain Chain LLM Node for Report Generation:**  
   - Add a node of type **Langchain Chain LLM**.  
   - Configure the prompt to include:  
     - Instruction as an expert AI analyst synthesizing two news sources.  
     - Input template embedding scraped article fields: `title`, `content`, `publishedTime`.  
     - Required output format restricting to Telegram-supported HTML tags only: `<b>, <i>, <u>, <s>, <code>, <pre>, <a>`.  
     - Use hyphen-prefixed lines for lists instead of `<ul>/<ol>/<li>`.  
     - Include dynamic date insertion using JavaScript expression:  
       `{{ new Date().toLocaleDateString('en-GB', { day: 'numeric', month: 'long', year: 'numeric' }) }}`.  
   - Connect the output of `Read News from AI News Website` as input.  
   - Set the AI language model input to the `Gemini 2.5 Flash` node.  
   - Name this node: `Generate Report`.

5. **Add Telegram Node to Send Message:**  
   - Add a node of type **Telegram**.  
   - Configure the **Chat ID** to your target Telegram chat (replace `CHAT_ID` with actual chat ID or username).  
   - Set **Text** field to: `={{ $json.text }}` to send the text output from the previous node.  
   - Enable **Parse Mode** and set it to **HTML** to ensure formatting is rendered correctly.  
   - Provide **Telegram Bot API Credentials** to allow sending messages.  
   - Connect the output of `Generate Report` to this node.  
   - Name this node: `Send a text message`.

6. **Connect Nodes in Order:**  
   - `AI-News Feed` → `Read News from AI News Website` → `Generate Report` → `Send a text message`.  
   - Also connect `Gemini 2.5 Flash` as the AI language model processor input to `Generate Report`.

7. **Set Credentials:**  
   - Ensure you have valid credentials for:  
     - Jina AI API  
     - Google Palm API (for Gemini)  
     - Telegram Bot API (bot token)

8. **Testing and Deployment:**  
   - Activate the workflow.  
   - Verify that new articles trigger the workflow, full content is scraped, the AI generates reports, and the Telegram messages arrive formatted correctly.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                  |
|-------------------------------------------------------------------------------------------------|-----------------------------------------------------------------|
| The workflow uses Telegram-supported HTML tags only for message formatting: `<b>, <i>, <u>, <s>, <code>, <pre>, <a>`. Lists are created with hyphen-prefixed lines. | Important for Telegram message compatibility.                   |
| Gemini 2.5 Flash node requires Google Palm API credentials with sufficient quota and access.    | https://developers.generativeai.google/                         |
| Jina AI node scrapes article content from URLs; ensure links are publicly accessible.            | https://docs.jina.ai/                                           |
| Telegram Bot must have permissions to send messages to the target chat, and chat ID must be correct. | https://core.telegram.org/bots/api#sendmessage                   |
| Workflow triggers every minute; consider API rate limits and polling frequency for scalability. | Can be adjusted for lower frequency if necessary.               |
| Sticky notes in the workflow provide detailed explanations of node roles and parameters.        | N8N UI sticky notes for workflow documentation and maintenance. |

---

**Disclaimer:**  
The text processed and generated by this workflow originates exclusively from publicly available AI news sources and automated AI processing. All operations comply with current content policies and legal guidelines.

---