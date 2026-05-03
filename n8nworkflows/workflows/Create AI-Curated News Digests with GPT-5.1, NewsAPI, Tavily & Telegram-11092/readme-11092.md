Create AI-Curated News Digests with GPT-5.1, NewsAPI, Tavily & Telegram

https://n8nworkflows.xyz/workflows/create-ai-curated-news-digests-with-gpt-5-1--newsapi--tavily---telegram-11092


# Create AI-Curated News Digests with GPT-5.1, NewsAPI, Tavily & Telegram

---

### 1. Workflow Overview

This n8n workflow automates the creation of AI-curated news digests by combining multiple advanced services. It runs on a weekly schedule to fetch recent news articles on specified topics, refines the selection with a GPT-5.1 AI model, enriches the content with fact-checking and additional sources via Tavily, and finally delivers a concise Markdown newsletter through Telegram.

The workflow is logically divided into five main blocks:

- **1.1 Schedule & Initialization**: Sets the weekly trigger and user-configured parameters for topics and language.
- **1.2 News Retrieval & Selection**: Queries NewsAPI for recent articles and filters the most relevant five using GPT-5.1.
- **1.3 Article Splitting**: Breaks down the selected articles list into individual items for per-article processing.
- **1.4 AI-Enrichment & Summarization**: For each article, uses an AI Agent integrating GPT-5.1 and Tavily to generate concise, fact-checked summaries.
- **1.5 Aggregation & Delivery**: Combines enriched articles into a single list and sends the formatted newsletter to a Telegram chat.

---

### 2. Block-by-Block Analysis

#### 1.1 Schedule & Initialization

- **Overview:**  
  This block triggers the workflow weekly and establishes the core parameters for topics and language that drive all subsequent steps.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Set topics and language

- **Node Details:**

  - **Schedule Trigger**  
    - Type: `Schedule Trigger`  
    - Role: Initiates workflow every Sunday at 09:00 by default.  
    - Configuration: Interval set to weekly, trigger at hour 9 (9 AM).  
    - Input: None (time-based trigger).  
    - Output: Triggers the next node to start processing.  
    - Edge Cases: Missed triggers if n8n instance is down; timezone considerations may affect timing.

  - **Set topics and language**  
    - Type: `Set`  
    - Role: Defines user preferences for news topics and output language.  
    - Configuration:  
      - `topics`: String of comma-separated keywords, default "AI,n8n".  
      - `language`: Output language for summaries, default "English".  
      - Includes other fields from input (timestamp from trigger).  
    - Input: Trigger from Schedule Trigger.  
    - Output: Passes parameters downstream.  
    - Edge Cases: Empty or invalid topics can cause empty API results or AI confusion.

#### 1.2 News Retrieval & Selection

- **Overview:**  
  Fetches recent news articles from NewsAPI based on topics and filters them to the top five relevant non-overlapping articles using GPT-5.1.

- **Nodes Involved:**  
  - Call NewsAPI  
  - AI Topic Selector

- **Node Details:**

  - **Call NewsAPI**  
    - Type: `HTTP Request`  
    - Role: Queries NewsAPI’s `/v2/everything` endpoint for news in the past 7 days matching topics.  
    - Configuration:  
      - Query parameters:  
        - `from`: 7 days before current timestamp.  
        - `q`: topics replaced commas with " OR " for broad search.  
        - `sortBy`: relevancy.  
      - Authentication: HTTP Query Auth with NewsAPI key.  
      - Response format: JSON.  
    - Input: Parameters from "Set topics and language" node (timestamp, topics).  
    - Output: News articles JSON array.  
    - Edge Cases: API key limits, network errors, empty results, query syntax issues.

  - **AI Topic Selector**  
    - Type: `OpenAI (GPT-5.1)`  
    - Role: Uses GPT-5.1 to select 5 most relevant non-overlapping articles from NewsAPI results, generating short summaries.  
    - Configuration:  
      - Model: GPT-5.1.  
      - Input prompt includes user topics, language, and all NewsAPI articles with title, description, content, source, and url.  
      - Output: JSON array of 5 articles with fields: title, summary, source, url.  
    - Input: NewsAPI articles JSON.  
    - Output: Filtered and summarized articles JSON.  
    - Edge Cases: Model misinterpretation, output parsing failures, rate limits.

#### 1.3 Article Splitting

- **Overview:**  
  Splits the array of 5 selected articles into individual items for per-article AI enrichment.

- **Nodes Involved:**  
  - Split Out

- **Node Details:**

  - **Split Out**  
    - Type: `Split Out`  
    - Role: Splits the field `message.content.articles` into separate items for parallel processing.  
    - Configuration: Field to split: `message.content.articles`.  
    - Input: JSON array of selected articles from AI Topic Selector.  
    - Output: Individual article objects, each passed separately downstream.  
    - Edge Cases: Empty article list, improper field path.

#### 1.4 AI-Enrichment & Summarization

- **Overview:**  
  Enriches each article with additional research sourced via Tavily, refines and rewrites summaries with GPT-5.1, and enforces structured JSON output.

- **Nodes Involved:**  
  - Newsletter AI Agent  
  - GPT-5.1  
  - Tavily  
  - Parser

- **Node Details:**

  - **Newsletter AI Agent**  
    - Type: `LangChain Agent`  
    - Role: Orchestrates calls to GPT-5.1 (language model), Tavily (external research tool), and Parser for output formatting.  
    - Configuration:  
      - Input text includes original article title, summary, source, URL, and desired language.  
      - Instructions:  
        1. Use Tavily to find 2–3 reliable, recent sources.  
        2. Update title if better one exists.  
        3. Write 1–3 sentence concise article combining original and new info.  
        4. Output JSON with fields: title, content, source, url.  
      - Output parser enabled to enforce structured JSON.  
    - Input: Single article item from Split Out node.  
    - Output: Enriched article JSON.  
    - Edge Cases: API limits, incomplete Tavily results, parser failures, language mismatches.

  - **GPT-5.1**  
    - Type: `LangChain OpenAI LLM`  
    - Role: Provides the language model (GPT-5.1) for the AI Agent to generate summaries and rewrites.  
    - Configuration: Model set to GPT-5.1, with default options.  
    - Input: Prompt from AI Agent.  
    - Output: Text completions used by AI Agent.  
    - Edge Cases: Rate limits, timeouts.

  - **Tavily**  
    - Type: `Tavily Tool`  
    - Role: Searches for additional sources and performs fact-checking during AI Agent execution.  
    - Configuration: Query parameter dynamically set by AI Agent.  
    - Input: Query from AI Agent.  
    - Output: Additional research data.  
    - Edge Cases: API errors, empty or irrelevant results.

  - **Parser**  
    - Type: `LangChain Output Parser Structured`  
    - Role: Validates and enforces the AI Agent’s output to match JSON schema with title, content, source, and url.  
    - Configuration: Manual JSON schema defined with required string fields.  
    - Input: Raw AI text output.  
    - Output: Structured JSON.  
    - Edge Cases: Parsing errors if AI output is malformed.

#### 1.5 Aggregation & Delivery

- **Overview:**  
  Aggregates all enriched article objects into a single list and sends a formatted Markdown newsletter via Telegram.

- **Nodes Involved:**  
  - Aggregate  
  - Send a text message

- **Node Details:**

  - **Aggregate**  
    - Type: `Aggregate`  
    - Role: Combines individual enriched articles into one aggregated list under field `output`.  
    - Configuration: Aggregates field named `output`.  
    - Input: Enriched article JSON items from AI Agent.  
    - Output: Single JSON object with combined list.  
    - Edge Cases: Empty input, partial aggregation.

  - **Send a text message**  
    - Type: `Telegram`  
    - Role: Sends the final newsletter as a Markdown-formatted message to a Telegram chat.  
    - Configuration:  
      - Chat ID specified (numeric).  
      - Message text dynamically generated by mapping over aggregated articles, formatting each with markdown: bold title, content, and source as a hyperlink.  
      - Attribution disabled.  
      - Telegram Bot credentials configured.  
    - Input: Aggregated enriched articles list.  
    - Output: Message sent confirmation.  
    - Edge Cases: Invalid chat ID, Telegram API errors, message size limits.

---

### 3. Summary Table

| Node Name               | Node Type                     | Functional Role                            | Input Node(s)              | Output Node(s)            | Sticky Note                                                                                                                     |
|-------------------------|-------------------------------|--------------------------------------------|----------------------------|---------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger        | Schedule Trigger              | Initiates workflow weekly at 9 AM          | —                          | Set topics and language    | • Runs at 9am every Sunday by default.                                                                                          |
| Set topics and language | Set                           | Defines user topics and output language    | Schedule Trigger            | Call NewsAPI              | • Configure your topics (comma-separated) and output language.                                                                 |
| Call NewsAPI            | HTTP Request                  | Fetches recent news articles from NewsAPI  | Set topics and language     | AI Topic Selector         | • Queries last 7 days of news for your topics.                                                                                   |
| AI Topic Selector       | OpenAI GPT-5.1                | Selects 5 most relevant, non-overlapping articles | Call NewsAPI               | Split Out                 | • Uses GPT-5.1 to pick 5 relevant articles.                                                                                      |
| Split Out               | Split Out                     | Splits article list into individual items  | AI Topic Selector           | Newsletter AI Agent       | • Turns selected list into one item per article.                                                                                 |
| Newsletter AI Agent     | LangChain Agent               | Enriches and summarizes articles with AI  | Split Out                   | Aggregate                 | • Orchestrates GPT-5, Tavily, and parser per article.                                                                            |
| GPT-5.1                 | LangChain OpenAI LLM          | Language model for AI Agent                 | — (invoked by AI Agent)     | —                         | • Writes short, clear newsletter-style summaries.                                                                                |
| Tavily                  | Tavily Tool                   | Adds fresh sources for grounding and fact-checking | — (invoked by AI Agent)  | —                         | • Adds fresh sources for grounding and factual checks.                                                                           |
| Parser                  | LangChain Output Parser       | Enforces structured JSON output             | Newsletter AI Agent (output)| Newsletter AI Agent (input)| • Enforces JSON with `title`, `content`, `source`, `url`.                                                                       |
| Aggregate               | Aggregate                     | Combines all enriched articles into list   | Newsletter AI Agent         | Send a text message       | • Combines all enriched article objects into one list.                                                                           |
| Send a text message     | Telegram                      | Sends formatted newsletter to Telegram chat| Aggregate                   | —                         | • Formats newsletter in Markdown and posts it to Telegram.                                                                      |
| Sticky Note             | Sticky Note                   | Informational notes                         | —                          | —                         | # AI Newsletter Workflow: overview and setup steps.                                                                              |
| Sticky Note1            | Sticky Note                   | Schedule and configuration notes           | —                          | —                         | • Schedule Trigger runs weekly at 9am Sunday. • Set topics and language configuration.                                           |
| Sticky Note2            | Sticky Note                   | Fetch and select articles notes             | —                          | —                         | • Call NewsAPI queries last 7 days. • AI Topic Selector picks 5 relevant articles. • Split Out prepares articles for processing.  |
| Sticky Note3            | Sticky Note                   | AI enrichment notes                         | —                          | —                         | • Newsletter AI Agent orchestrates GPT-5, Tavily, and parser.                                                                   |
| Sticky Note4            | Sticky Note                   | Aggregation and sending notes               | —                          | —                         | • Aggregate combines articles. • Send a text message posts newsletter to Telegram.                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create `Schedule Trigger` node**  
   - Type: Schedule Trigger  
   - Set to run weekly at 9:00 AM (Sunday default).  
   - No credentials required.

2. **Create `Set topics and language` node**  
   - Type: Set  
   - Assign two string fields:  
     - `topics`: e.g., "AI,n8n" (comma-separated topics).  
     - `language`: e.g., "English".  
   - Connect input from Schedule Trigger.

3. **Create `Call NewsAPI` node**  
   - Type: HTTP Request  
   - Set method to GET, URL: `https://newsapi.org/v2/everything`.  
   - Add query parameters:  
     - `from`: Expression to subtract 7 days from current timestamp: `={{ DateTime.fromISO($json.timestamp).minus({ days: 7 }) }}`.  
     - `q`: Expression replacing commas with " OR " in topics: `={{ $json.topics.replaceAll(",", " OR ") }}`.  
     - `sortBy`: set to `"relevancy"`.  
   - Authentication: HTTP Query Auth using your NewsAPI key credential.  
   - Connect input from Set topics and language.

4. **Create `AI Topic Selector` node**  
   - Type: OpenAI (GPT-5.1)  
   - Model: Select GPT-5.1.  
   - Input prompt setup:  
     - Provide instructions to select top 5 relevant non-overlapping articles with title, summary, source, url.  
     - Pass NewsAPI articles formatted as text.  
     - Include user topics and output language from previous node.  
   - JSON output enabled.  
   - Connect input from Call NewsAPI.

5. **Create `Split Out` node**  
   - Type: Split Out  
   - Field to split: `message.content.articles` (path to AI Topic Selector output articles array).  
   - Connect input from AI Topic Selector.

6. **Create `Newsletter AI Agent` node**  
   - Type: LangChain Agent  
   - Configure text prompt with instructions:  
     - Use Tavily Search to find reliable sources.  
     - Update title if better.  
     - Write 1-3 sentence concise article combining original summary and new info.  
     - Return JSON with title, content, source, url.  
     - Use user language parameter from "Set topics and language".  
   - Enable output parser with a manual JSON schema enforcing all required fields as strings.  
   - Assign GPT-5.1 as language model.  
   - Assign Tavily as research tool, dynamically passing query.  
   - Assign Parser as output parser.  
   - Connect input from Split Out.

7. **Create `GPT-5.1` node**  
   - Type: LangChain OpenAI LLM  
   - Model: GPT-5.1.  
   - No additional options needed.  
   - Used internally by Newsletter AI Agent.

8. **Create `Tavily` node**  
   - Type: Tavily Tool  
   - No static query; query passed dynamically by AI Agent.  
   - Requires Tavily API credentials.  
   - Used internally by Newsletter AI Agent.

9. **Create `Parser` node**  
   - Type: LangChain Output Parser Structured  
   - Manual JSON schema with properties: title, content, source, url (all strings).  
   - Used internally by Newsletter AI Agent.

10. **Create `Aggregate` node**  
    - Type: Aggregate  
    - Aggregate field: `output` (combine enriched articles into one list).  
    - Connect input from Newsletter AI Agent.

11. **Create `Send a text message` node**  
    - Type: Telegram  
    - Set Chat ID to your Telegram chat or channel ID.  
    - Text parameter: expression that maps over aggregated articles to produce Markdown-formatted message:  
      ```
      ={{ 
        $json.output.map(article => {
          const title = JSON.stringify(article.title).slice(1, -1);
          const content = JSON.stringify(article.content).slice(1, -1);
          const source = JSON.stringify(article.source).slice(1, -1);
          const url = JSON.stringify(article.url).slice(1, -1);

          return `*${title}*\n${content}\nSource: [${source}](${url})`;
        }).join('\n\n')
      }}
      ```  
    - Disable attribution.  
    - Provide Telegram Bot credentials.  
    - Connect input from Aggregate.

12. **Add Sticky Notes for documentation** (optional)  
    - Add notes explaining workflow overview, schedule, fetching, AI enrichment, and delivery steps as per original content.

---

### 5. General Notes & Resources

| Note Content                                                                                                                           | Context or Link                                                                                                  |
|----------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| This workflow runs weekly, fetching fresh news for your chosen topics, letting AI pick and enrich articles, then sending a Telegram newsletter. | Workflow purpose and high-level description.                                                                    |
| Setup steps: Configure schedule, topics, language, and add credentials for NewsAPI, OpenAI, Tavily, and Telegram Bot.                   | Essential setup for correct operation.                                                                           |
| Customization tips: Adjust topics/language, schedule timing, number of selected articles, and AI prompts for different newsletter styles.| Workflow adaptability and tuning hints.                                                                          |
| Sources combined: NewsAPI for discovery, GPT-5 for reasoning and writing, Tavily for fact-checked enrichment.                          | Integration summary.                                                                                             |
| Telegram message format uses Markdown with bold titles and clickable source links.                                                      | Output formatting details.                                                                                        |
| External links: NewsAPI (https://newsapi.org/), OpenAI (https://openai.com/), Tavily (https://tavily.com/), Telegram Bot API (https://core.telegram.org/bots/api) | Credential and API providers for integration.                                                                    |

---

**Disclaimer:**  
The provided text is derived solely from an automated workflow created in n8n, respecting all applicable content policies, containing no illegal or protected material. All processed data is legal and publicly accessible.

---