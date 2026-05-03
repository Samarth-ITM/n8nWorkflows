Generate Email Newsletters from Telegram Keywords with Dumpling AI and GPT

https://n8nworkflows.xyz/workflows/generate-email-newsletters-from-telegram-keywords-with-dumpling-ai-and-gpt-8187


# Generate Email Newsletters from Telegram Keywords with Dumpling AI and GPT

---

### 1. Workflow Overview

This workflow automates the generation of email newsletters based on keywords received from Telegram messages. Its primary use case is to take a keyword provided by a Telegram user, expand it into related search queries, fetch recent news articles on those topics, clean and aggregate the content, and produce a professionally formatted newsletter email that is then sent via Gmail.

The workflow is logically divided into two main blocks:

- **1.1 Input Reception & AI-Driven Search Orchestration:**  
  Receives keywords from Telegram, utilizes an AI agent to generate Google autocomplete suggestions, then searches Google News for recent articles related to those suggestions using Dumpling AI APIs. The results are parsed into structured JSON and split into individual articles.

- **1.2 Article Processing & Newsletter Generation:**  
  Iterates over each article, scrapes and cleans the article content via Dumpling AI scraper API, aggregates all cleaned content, then employs OpenAI’s GPT model to generate a high-quality HTML newsletter and a subject line. Finally, the newsletter is sent to a target email address via Gmail.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & AI-Driven Search Orchestration

**Overview:**  
This block starts with receiving a keyword from Telegram, then uses an AI agent to expand this keyword into relevant Google autocomplete suggestions and fetches recent news articles for those suggestions. It handles fallback logic if no suggestions are found and outputs structured JSON containing news articles.

**Nodes Involved:**  
- Start: Receive Keyword from Telegram  
- Simple Memory  
- AI Agent: Expand Keyword & Orchestrate Tools  
- Google_autocomplete (Dumpling AI Autocomplete API)  
- Search_news (Dumpling AI Google News API)  
- LLM: Language Model (OpenAI GPT-4o-mini)  
- Parser: Format News JSON  
- Split Articles  
- Sticky Note (Agent Branch explanation)

**Node Details:**

- **Start: Receive Keyword from Telegram**  
  - *Type:* Telegram Trigger  
  - *Role:* Entry point, listens for new messages from Telegram users.  
  - *Config:* Waits for "message" updates, uses Telegram account credentials.  
  - *Input/Output:* No input, outputs Telegram message JSON (including sender id and text).  
  - *Edge cases:* Network/connectivity issues, invalid Telegram credentials.  
  - *Notes:* Starts the entire workflow.

- **Simple Memory**  
  - *Type:* Langchain Memory Buffer Window  
  - *Role:* Maintains conversational context keyed by Telegram user ID (`$json.message.from.id`).  
  - *Config:* Session key based on Telegram user ID, custom sessionIdType.  
  - *Input:* Telegram message JSON.  
  - *Output:* Provides memory context for AI agent.  
  - *Edge cases:* Memory overflow or session key collisions.

- **AI Agent: Expand Keyword & Orchestrate Tools**  
  - *Type:* Langchain Agent Node  
  - *Role:* Central orchestrator that uses AI to expand the keyword into autocomplete suggestions and then search news articles for those suggestions.  
  - *Config:*  
    - Prompt includes system instructions describing usage of Google_autocomplete and Search_news tools.  
    - Takes keyword text from Telegram message (`{{ $json.message.text }}`).  
    - Outputs structured JSON listing recent news articles with fields: category, title, url, source, summary, published.  
  - *Input:* Telegram message text and memory context.  
  - *Output:* Structured JSON of news articles.  
  - *Connections:* Internally calls Google_autocomplete and Search_news nodes using AI tool integration.  
  - *Edge cases:*  
    - No autocomplete suggestions → fallback to original keyword search.  
    - API auth errors, timeouts, or malformed responses.  
    - Expression evaluation errors in prompt templates.

- **Google_autocomplete**  
  - *Type:* HTTP Request Tool (Dumpling AI Autocomplete API)  
  - *Role:* Fetches autocomplete search suggestions from Dumpling AI based on input keyword.  
  - *Config:* POST to `/api/v1/get-autocomplete` with JSON body containing `"query"` from AI agent variable `keyword`.  
  - *Input:* Keyword string from AI agent.  
  - *Output:* List of autocomplete suggestions.  
  - *Credentials:* Dumpling AI API key via HTTP Header Authentication.  
  - *Edge cases:* API rate limits, invalid credentials, empty results.

- **Search_news**  
  - *Type:* HTTP Request Tool (Dumpling AI Google News API)  
  - *Role:* Searches Google News for recent articles matching autocomplete suggestions.  
  - *Config:* POST to `/api/v1/search-news` with JSON body containing `"query"` from AI agent variable `suggestion`.  
  - *Input:* Autocomplete suggestions list.  
  - *Output:* Recent news articles JSON.  
  - *Credentials:* Dumpling AI API key via HTTP Header Authentication.  
  - *Edge cases:* API failures, empty search results.

- **LLM: Language Model**  
  - *Type:* Langchain OpenAI Chat Model (GPT-4o-mini)  
  - *Role:* Used internally by AI Agent node to generate expansions and orchestrate tool calls.  
  - *Config:* Model choice is GPT-4o-mini for efficient processing.  
  - *Input/Output:* Integrated within AI agent node.  
  - *Edge cases:* OpenAI API rate limits, invalid API key, model unavailability.

- **Parser: Format News JSON**  
  - *Type:* Langchain Output Parser Structured  
  - *Role:* Parses AI agent output into strict JSON schema with list of articles.  
  - *Config:* Schema example enforces fields like category, title, url, source, summary, published.  
  - *Input:* Raw AI output text.  
  - *Output:* Structured JSON data for downstream nodes.  
  - *Edge cases:* Parsing failures if AI output deviates from schema.

- **Split Articles**  
  - *Type:* Split Out  
  - *Role:* Splits the array of articles from JSON into individual item outputs for batch processing.  
  - *Config:* Splits on JSON path `output.articles`.  
  - *Input:* Structured articles JSON.  
  - *Output:* Individual article objects.  
  - *Edge cases:* Empty or malformed article arrays.

- **Sticky Note (Agent Branch)**  
  - *Content:* Explains the logic of this branch: keyword reception, expansion via Dumpling AI Autocomplete, news retrieval, JSON parsing, and splitting for processing.

---

#### 2.2 Article Processing & Newsletter Generation

**Overview:**  
Processes each news article individually by scraping and cleaning its content, aggregates all cleaned articles, then generates a professional newsletter email using OpenAI GPT. The final newsletter is sent via Gmail.

**Nodes Involved:**  
- Loop: Process Each Article  
- Scraper: Clean Article Content (Dumpling AI Scraper API)  
- Wait  
- Aggregate: Combine Article Content  
- Generate Newsletter (OpenAI GPT-4.1-mini)  
- Send Newsletter via Email (Gmail)  
- Sticky Note1 (Newsletter Branch explanation)

**Node Details:**

- **Loop: Process Each Article**  
  - *Type:* Split In Batches  
  - *Role:* Processes each article one at a time in batches to manage API rate limits and resource usage.  
  - *Config:* Default batch size (usually 1) to handle articles sequentially.  
  - *Input:* Individual article JSON from Split Articles node.  
  - *Output:* Passes each article to Scraper node and loops.  
  - *Edge cases:* Batch processing errors, empty input.

- **Scraper: Clean Article Content**  
  - *Type:* HTTP Request  
  - *Role:* Calls Dumpling AI scraper API to fetch and clean full article content from article URLs.  
  - *Config:* POST to `/api/v1/scrape` with JSON containing `url` and `"cleaned": true`.  
  - *Input:* Article URL from current batch item.  
  - *Output:* Cleaned article content including markdown and metadata.  
  - *Credentials:* Dumpling AI API key.  
  - *Edge cases:* Invalid URLs, scraper API failures, timeout, missing content.

- **Wait**  
  - *Type:* Wait  
  - *Role:* Introduces a delay after each article processing iteration to respect API rate limits and avoid flooding.  
  - *Config:* Default wait duration (unspecified, likely seconds).  
  - *Input:* After Scraper node output.  
  - *Output:* Delays next loop iteration.  
  - *Edge cases:* Potential workflow delays if wait time is too long or too short.

- **Aggregate: Combine Article Content**  
  - *Type:* Aggregate  
  - *Role:* Combines all cleaned article contents into a single aggregated field named `content`.  
  - *Config:* Aggregates all items on the field `content`.  
  - *Input:* Cleaned article content items from loop.  
  - *Output:* Single aggregated JSON with combined content.  
  - *Edge cases:* Data size limits, aggregation failures.

- **Generate Newsletter**  
  - *Type:* Langchain OpenAI  
  - *Role:* Uses OpenAI GPT-4.1-mini model to generate a high-quality HTML newsletter and email subject line from aggregated content.  
  - *Config:*  
    - System message details strict formatting rules: produce valid JSON with fields `newsletter` (HTML) and `subject`.  
    - Clear instructions to avoid invented facts, produce inline CSS, simple language, and mobile-friendly design.  
    - Input is JSON stringified aggregated content.  
  - *Input:* Aggregated article content.  
  - *Output:* JSON with newsletter HTML and subject string.  
  - *Credentials:* OpenAI API key.  
  - *Edge cases:* Model hallucination, JSON parsing errors, invalid HTML output.

- **Send Newsletter via Email**  
  - *Type:* Gmail  
  - *Role:* Sends the generated newsletter email to a specified recipient.  
  - *Config:*  
    - Recipient email address taken from input (empty in JSON, intended to be set).  
    - Email message body is `newsletter` HTML.  
    - Email subject line is from `subject`.  
    - Uses Gmail OAuth2 credentials.  
  - *Input:* JSON output from Generate Newsletter node.  
  - *Output:* Email sent confirmation or error.  
  - *Edge cases:* Gmail API auth failure, invalid recipient, network issues.

- **Sticky Note1 (Newsletter Branch)**  
  - *Content:* Describes that from Loop Over Items node, articles are scraped and cleaned, aggregated, passed to OpenAI for newsletter generation, then sent via Gmail.

---

### 3. Summary Table

| Node Name                          | Node Type                                    | Functional Role                                   | Input Node(s)                     | Output Node(s)                        | Sticky Note                                                                                           |
|-----------------------------------|----------------------------------------------|--------------------------------------------------|----------------------------------|-------------------------------------|-----------------------------------------------------------------------------------------------------|
| Start: Receive Keyword from Telegram | Telegram Trigger                            | Entry point: receives keywords from Telegram users | None                             | AI Agent: Expand Keyword & Orchestrate Tools | ## Agent Branch: This branch begins when a keyword is received from **Telegram**.                   |
| Simple Memory                     | Langchain Memory Buffer Window               | Maintains session memory context per user         | Start: Receive Keyword from Telegram | AI Agent: Expand Keyword & Orchestrate Tools | ## Agent Branch...                                                                                   |
| AI Agent: Expand Keyword & Orchestrate Tools | Langchain Agent                           | Orchestrates autocomplete and news search via AI | Start: Receive Keyword from Telegram, Simple Memory, Google_autocomplete, Search_news, LLM, Parser | Split Articles                      | ## Agent Branch...                                                                                   |
| Google_autocomplete               | HTTP Request Tool (Dumpling AI Autocomplete) | Fetches autocomplete suggestions from Dumpling AI | AI Agent: Expand Keyword & Orchestrate Tools | AI Agent: Expand Keyword & Orchestrate Tools | ## Agent Branch...                                                                                   |
| Search_news                      | HTTP Request Tool (Dumpling AI Google News)  | Searches news articles based on autocomplete terms | AI Agent: Expand Keyword & Orchestrate Tools | AI Agent: Expand Keyword & Orchestrate Tools | ## Agent Branch...                                                                                   |
| LLM: Language Model              | Langchain OpenAI Chat Model (GPT-4o-mini)    | Provides AI model for keyword expansion and orchestration | AI Agent: Expand Keyword & Orchestrate Tools | AI Agent: Expand Keyword & Orchestrate Tools | ## Agent Branch...                                                                                   |
| Parser: Format News JSON         | Langchain Output Parser Structured            | Parses AI output into structured JSON news articles | AI Agent: Expand Keyword & Orchestrate Tools | Split Articles                      | ## Agent Branch...                                                                                   |
| Split Articles                  | Split Out                                    | Splits articles array into individual article items | AI Agent: Expand Keyword & Orchestrate Tools | Loop: Process Each Article          | ## Agent Branch...                                                                                   |
| Loop: Process Each Article       | Split In Batches                             | Processes each article in batches for scraping    | Split Articles                   | Aggregate: Combine Article Content, Wait | ## Newsletter Branch: From Loop Over Items node, articles are scraped and cleaned...                |
| Scraper: Clean Article Content   | HTTP Request (Dumpling AI Scraper API)        | Scrapes and cleans full article content           | Loop: Process Each Article       | Loop: Process Each Article          | ## Newsletter Branch...                                                                              |
| Wait                           | Wait                                         | Delays between processing articles to respect API limits | Loop: Process Each Article       | Scraper: Clean Article Content      | ## Newsletter Branch...                                                                              |
| Aggregate: Combine Article Content | Aggregate                                   | Aggregates cleaned article content into one field | Loop: Process Each Article       | Generate Newsletter                 | ## Newsletter Branch...                                                                              |
| Generate Newsletter              | Langchain OpenAI (GPT-4.1-mini)                | Generates HTML newsletter and subject from content | Aggregate: Combine Article Content | Send Newsletter via Email           | ## Newsletter Branch...                                                                              |
| Send Newsletter via Email        | Gmail                                        | Sends the generated newsletter email              | Generate Newsletter              | None                              | ## Newsletter Branch...                                                                              |
| Sticky Note                     | Sticky Note                                  | Explains Agent Branch logic                        | None                            | None                              | ## Agent Branch: Explains keyword reception, expansion, news retrieval, parsing, splitting          |
| Sticky Note1                    | Sticky Note                                  | Explains Newsletter Branch logic                   | None                            | None                              | ## Newsletter Branch: Explains article scraping, aggregation, newsletter generation, email sending  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node: "Start: Receive Keyword from Telegram"**  
   - Type: Telegram Trigger  
   - Parameters: Listen to "message" updates.  
   - Credentials: Connect your Telegram bot API credentials.  
   - Purpose: Receive keyword messages from Telegram users.

2. **Add Langchain Memory Node: "Simple Memory"**  
   - Type: Langchain Memory Buffer Window  
   - Parameters:  
     - Session Key: `={{ $json.message.from.id }}` (use Telegram user ID)  
     - Session ID Type: Custom Key  
   - Connect input from Telegram Trigger.  
   - Purpose: Maintain conversation context per user.

3. **Add Langchain OpenAI Node: "LLM: Language Model"**  
   - Type: Langchain OpenAI Chat Model  
   - Parameters:  
     - Model: `gpt-4o-mini`  
   - Credentials: Provide OpenAI API credentials.  
   - Purpose: AI model for the agent to process instructions.

4. **Add HTTP Request Node: "Google_autocomplete"**  
   - Type: HTTP Request  
   - Parameters:  
     - URL: `https://app.dumplingai.com/api/v1/get-autocomplete`  
     - Method: POST  
     - Body Type: JSON  
     - Body: `{"query": "{{ $fromAI('keyword', 'Keyword to get autocomplete suggestions', 'string') }}"}`  
   - Authentication: HTTP Header Auth with Dumpling AI API key.  
   - Purpose: Get autocomplete suggestions from Dumpling AI.

5. **Add HTTP Request Node: "Search_news"**  
   - Type: HTTP Request  
   - Parameters:  
     - URL: `https://app.dumplingai.com/api/v1/search-news`  
     - Method: POST  
     - Body Type: JSON  
     - Body: `{"query": "{{ $fromAI('suggestion', 'Autocomplete suggestion to search in Google News', 'string') }}"}`  
   - Authentication: HTTP Header Auth with Dumpling AI API key.  
   - Purpose: Search Google News for autocomplete suggestions.

6. **Add Langchain Output Parser Node: "Parser: Format News JSON"**  
   - Type: Langchain Output Parser Structured  
   - Parameters: Set JSON schema example for news articles with fields: category, title, url, source, summary, published.  
   - Purpose: Parse AI agent output into structured JSON.

7. **Add Langchain Agent Node: "AI Agent: Expand Keyword & Orchestrate Tools"**  
   - Type: Langchain Agent  
   - Parameters:  
     - Text: `=Keyword: {{ $json.message.text }}`  
     - System Message: Include detailed instructions on tool usage: first call Google_autocomplete, then Search_news with autocomplete results, fallback to keyword if none, output structured JSON.  
     - Tools: Include Google_autocomplete and Search_news nodes as AI tools.  
     - LLM: Link to "LLM: Language Model" node.  
     - Output Parser: Link to "Parser: Format News JSON" node.  
   - Connect inputs from Telegram Trigger and Simple Memory.  
   - Outputs to "Split Articles".  
   - Purpose: Expand keyword and orchestrate news search.

8. **Add Split Out Node: "Split Articles"**  
   - Type: Split Out  
   - Parameters: Split on `output.articles` field.  
   - Connect input from AI Agent node.  
   - Outputs to "Loop: Process Each Article".

9. **Add Split In Batches Node: "Loop: Process Each Article"**  
   - Type: Split In Batches  
   - Parameters: Use default batch size (1).  
   - Connect input from "Split Articles".  
   - Outputs to "Scraper: Clean Article Content" and "Wait".

10. **Add HTTP Request Node: "Scraper: Clean Article Content"**  
    - Type: HTTP Request  
    - Parameters:  
      - URL: `https://app.dumplingai.com/api/v1/scrape`  
      - Method: POST  
      - Body Type: JSON  
      - Body: `{"url": "{{ $json.url }}", "cleaned": true}`  
    - Authentication: HTTP Header Auth with Dumpling AI API key.  
    - Connect input from "Loop: Process Each Article".  
    - Output loops back to "Loop: Process Each Article" to continue batch.

11. **Add Wait Node: "Wait"**  
    - Type: Wait  
    - Parameters: Default delay (adjust as needed to respect API limits).  
    - Connect input from "Loop: Process Each Article" (parallel path).  
    - Output back to "Scraper: Clean Article Content" to pace scraping.

12. **Add Aggregate Node: "Aggregate: Combine Article Content"**  
    - Type: Aggregate  
    - Parameters:  
      - Aggregate all item data on field `content`  
      - Destination field name: `content`  
    - Connect input from "Loop: Process Each Article" (main output after batches complete).  
    - Output to "Generate Newsletter".

13. **Add Langchain OpenAI Node: "Generate Newsletter"**  
    - Type: Langchain OpenAI  
    - Parameters:  
      - Model: `gpt-4.1-mini`  
      - System Message: Detailed instructions to generate HTML newsletter and subject line in strict JSON format, with formatting, style, and tone rules.  
      - Input: JSON stringified aggregated content (`"{{ JSON.stringify($json.content) }}"`).  
      - Output: JSON with `newsletter` HTML and `subject` string.  
    - Credentials: OpenAI API credentials.  
    - Connect input from "Aggregate: Combine Article Content".  
    - Output to "Send Newsletter via Email".

14. **Add Gmail Node: "Send Newsletter via Email"**  
    - Type: Gmail  
    - Parameters:  
      - Send To: Set target email address (e.g., internal marketing list).  
      - Subject: `={{ $json.message.content.subject }}`  
      - Message: `={{ $json.message.content.newsletter }}` (HTML)  
    - Credentials: Gmail OAuth2 credentials.  
    - Connect input from "Generate Newsletter".

15. **Add Sticky Notes**  
    - Add one sticky note near the first block describing the agent branch (keyword reception, expansion, news retrieval, parsing, splitting).  
    - Add another sticky note near the second block describing the newsletter branch (article scraping, aggregation, newsletter generation, email sending).

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| This workflow leverages Dumpling AI APIs for Google autocomplete, news search, and article scraping to enrich keyword-based newsletter content. | Dumpling AI API Documentation (internal use) |
| OpenAI GPT-4o-mini and GPT-4.1-mini models are used for language understanding and newsletter generation, respectively. | OpenAI API: https://platform.openai.com/docs |
| The newsletter generation enforces strict JSON output format to ensure downstream processing is robust and predictable. | JSON Schema enforcement in Langchain Output Parser |
| The Gmail node requires OAuth2 credentials with send mail scope configured. | Gmail API OAuth2 setup instructions |
| Telegram Trigger requires a valid bot token and webhook URL configured in Telegram Bot settings. | Telegram Bot API: https://core.telegram.org/bots/api |
| The workflow processes articles sequentially with delays to avoid rate limiting on external APIs. | Rate limit best practices for Dumpling AI and OpenAI APIs |

---

**Disclaimer:** The text provided is generated based on an n8n workflow and respects all content policies. All data processed is legal and public.

---