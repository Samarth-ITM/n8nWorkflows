Automated Daily News Summaries with OpenRouter AI & Gmail Delivery

https://n8nworkflows.xyz/workflows/automated-daily-news-summaries-with-openrouter-ai---gmail-delivery-9552


# Automated Daily News Summaries with OpenRouter AI & Gmail Delivery

---

### 1. Workflow Overview

This workflow automates the generation and delivery of daily news summaries in Japanese. It is designed to run once every 24 hours, gathering the latest news on key topics such as technology, business, politics, science, and world events. Using the OpenRouter AI language model combined with the Tavily News Search API, it composes a concise and structured news briefing formatted as an HTML email, then sends it to a specified recipient via Gmail.

The workflow logic is grouped into the following blocks:

- **1.1 Scheduled Trigger & Query Preparation:** Initiates the workflow daily and prepares a natural language query for news retrieval.
- **1.2 AI News Agent Processing:** Uses the OpenRouter Chat Model and Tavily News Search tool to find, analyze, and summarize news stories in a specified JSON format.
- **1.3 Output Parsing:** Parses the AI-generated JSON to structured data for further processing.
- **1.4 Email Formatting:** Converts the parsed data into a formatted HTML email body, including styling for readability.
- **1.5 Email Delivery:** Sends the formatted news summary email via Gmail using OAuth2 credentials.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger & Query Preparation

**Overview:**  
This block triggers the workflow once every 24 hours and generates a date-specific news search query describing the desired news scope and topics.

**Nodes Involved:**  
- Schedule Trigger  
- Prepare News Query  
- Sticky Note (instructional)

**Node Details:**

- **Schedule Trigger**  
  - Type: `n8n-nodes-base.scheduleTrigger`  
  - Role: Initiates the workflow on a recurring 24-hour interval.  
  - Configuration: Interval set to 24 hours.  
  - Inputs: None  
  - Outputs: Triggers the next node "Prepare News Query".  
  - Edge Cases: Misconfigured schedule may cause missed runs; timezone settings affect trigger timing.

- **Prepare News Query**  
  - Type: `n8n-nodes-base.code` (JavaScript)  
  - Role: Creates a date-aware natural language query string for the AI agent to find relevant news.  
  - Configuration: Uses JavaScript Date object to get the current date in a human-readable English format, then constructs a query string requesting important news from technology, business, politics, science, and world events.  
  - Key Expression: `query` string built dynamically with date inclusion.  
  - Inputs: Trigger from Schedule Trigger.  
  - Outputs: JSON containing `query` and `date` fields, passed to "AI News Agent".  
  - Edge Cases: Date formatting depends on locale; if timezone or date is incorrect, query may be inaccurate.

- **Sticky Note** (positioned near Schedule Trigger)  
  - Content: Explains the workflow purpose and required API credentials setup (OpenRouter, Tavily, Gmail).  
  - Context: Provides user guidance for correct setup.

---

#### 2.2 AI News Agent Processing

**Overview:**  
This block performs the core AI-driven news retrieval and summarization. It uses the OpenRouter language model and Tavily News Search API as tools, orchestrated by the AI News Agent node. The agent is configured to produce a strict JSON output in Japanese containing the news title, overall summary, and 5-7 individual news stories.

**Nodes Involved:**  
- AI News Agent  
- OpenRouter Chat Model  
- Tavily News Search  
- News Output Parser  
- Sticky Note (agent configuration)

**Node Details:**

- **AI News Agent**  
  - Type: `@n8n/n8n-nodes-langchain.agent`  
  - Role: Central AI orchestrator that sends the query to language and data tools, processes responses, and returns a structured JSON summary.  
  - Configuration:  
    - Input: Receives `query` string from "Prepare News Query".  
    - Options: System message instructs the agent to act as a professional news analyst, focusing on Japanese language output and strict adherence to a detailed JSON schema specifying title, summary, and stories (headline, category, summary).  
    - Integrates with:  
      - OpenRouter Chat Model (language model)  
      - Tavily News Search (HTTP request tool for news)  
    - Output: Valid JSON object matching the schema.  
  - Inputs: JSON with `query` field; AI output parser input connected.  
  - Outputs: JSON news summary passed to "Format for Gmail".  
  - Version-specific: Requires n8n nodes-langchain agent v2 or higher for output parser support.  
  - Edge Cases:  
    - AI could produce malformed JSON if instructions are not followed strictly.  
    - API key failures or rate limits on OpenRouter or Tavily could cause errors.  
    - Network timeouts or malformed responses from Tavily.  
    - Language or schema mismatch errors.  

- **OpenRouter Chat Model**  
  - Type: `@n8n/n8n-nodes-langchain.lmChatOpenRouter`  
  - Role: Language model provider used by the AI agent to generate news summaries.  
  - Configuration: Connected with valid OpenRouter API credentials.  
  - Inputs: Receives prompts from AI News Agent.  
  - Outputs: Text completions for news summary.  
  - Edge Cases: API quota exceeded, invalid credentials, network issues.

- **Tavily News Search**  
  - Type: `@n8n/n8n-nodes-langchain.toolHttpRequest`  
  - Role: Provides real-time news search by querying Tavily API with the search term.  
  - Configuration:  
    - HTTP POST to `https://api.tavily.com/search` with JSON body containing API key, query from AI prompt, search depth set to "advanced", and other parameters for news filtering.  
    - Placeholder variable `{searchTerm}` dynamically replaced.  
  - Inputs: Called as a tool by AI News Agent.  
  - Outputs: News content for AI analysis.  
  - Edge Cases: Invalid or missing API key, rate limits, network errors, unexpected API response format.

- **News Output Parser**  
  - Type: `@n8n/n8n-nodes-langchain.outputParserStructured`  
  - Role: Parses AI agent output to structured JSON, ensuring it matches the expected schema (title, summary, stories array).  
  - Configuration: Manual schema specifying required fields and types.  
  - Inputs: AI agent output.  
  - Outputs: Parsed JSON for email formatting.  
  - Edge Cases: Parsing failures if AI output deviates from schema or is invalid JSON.

- **Sticky Note (near AI News Agent)**  
  - Content: Describes AI agent configuration and use of Tavily API for news search with structured JSON output.  
  - Context: Helps maintainers understand AI logic and tools integration.

---

#### 2.3 Output Parsing and Email Formatting

**Overview:**  
This block formats the structured news summary into a visually appealing HTML email using JavaScript code, preparing subject and body content for sending via Gmail.

**Nodes Involved:**  
- Format for Gmail  
- Sticky Note (email delivery instructions)

**Node Details:**

- **Format for Gmail**  
  - Type: `n8n-nodes-base.code` (JavaScript)  
  - Role: Converts the parsed news JSON into a styled HTML email body and sets an email subject line.  
  - Configuration:  
    - Reads parsed output and date from input JSON.  
    - Constructs an HTML document with inline CSS styling for readability and mobile responsiveness.  
    - Iterates through news stories adding headline, category (italic), and summary paragraphs.  
    - Includes a header with a news icon and the title, plus a footer noting "Generated by n8n Automation".  
    - Returns JSON with `subject` and `htmlBody` for the Gmail node.  
  - Inputs: Parsed news JSON and date metadata.  
  - Outputs: JSON with email subject and HTML body.  
  - Edge Cases: If input JSON is missing fields or date is undefined, formatting might be incomplete or produce errors.

- **Sticky Note (near Format for Gmail node)**  
  - Content: Instructions for Gmail OAuth2 setup, recipient email configuration, and note on customizable HTML styling.  
  - Context: Guides final email delivery configuration.

---

#### 2.4 Email Delivery

**Overview:**  
This final block sends the composed news summary email to the configured recipient via Gmail using OAuth2 authentication.

**Nodes Involved:**  
- Send a message

**Node Details:**

- **Send a message**  
  - Type: `n8n-nodes-base.gmail`  
  - Role: Sends an email with the subject and HTML body generated by the previous node.  
  - Configuration:  
    - Uses Gmail OAuth2 credentials linked to a valid Gmail account.  
    - Recipient email address set (default: `recipient@example.com`).  
    - Uses the `subject` and `message` fields from input JSON.  
    - Option to disable appending attribution text.  
  - Inputs: JSON containing `subject` and `htmlBody` from "Format for Gmail".  
  - Outputs: Email sent confirmation or error.  
  - Edge Cases: OAuth2 token expiration, invalid recipient email, Gmail API quota limits, network failures.

---

### 3. Summary Table

| Node Name            | Node Type                               | Functional Role                               | Input Node(s)          | Output Node(s)          | Sticky Note                                                                                                                 |
|----------------------|---------------------------------------|-----------------------------------------------|------------------------|-------------------------|-----------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger     | n8n-nodes-base.scheduleTrigger        | Triggers workflow every 24 hours              | None                   | Prepare News Query       | "Daily AI News Summary to Gmail" setup instructions including API credential requirements                                    |
| Prepare News Query   | n8n-nodes-base.code                   | Creates dynamic news search query with date   | Schedule Trigger        | AI News Agent            |                                                                                                                             |
| AI News Agent        | @n8n/n8n-nodes-langchain.agent        | Orchestrates AI news retrieval and summarization | Prepare News Query, News Output Parser, OpenRouter Chat Model, Tavily News Search | Format for Gmail         | "AI News Agent Configuration" describing the AI, Tavily API integration, and JSON schema                                     |
| OpenRouter Chat Model| @n8n/n8n-nodes-langchain.lmChatOpenRouter | Provides language model completions            | AI News Agent           | AI News Agent            |                                                                                                                             |
| Tavily News Search   | @n8n/n8n-nodes-langchain.toolHttpRequest | Searches latest news via Tavily API            | AI News Agent           | AI News Agent            |                                                                                                                             |
| News Output Parser   | @n8n/n8n-nodes-langchain.outputParserStructured | Parses AI JSON output into structured format   | AI News Agent           | AI News Agent            |                                                                                                                             |
| Format for Gmail     | n8n-nodes-base.code                   | Formats news JSON into HTML email content      | AI News Agent           | Send a message           | "Gmail Delivery Setup" instructions including OAuth2 credential and recipient email configuration                           |
| Send a message       | n8n-nodes-base.gmail                  | Sends formatted news summary email via Gmail  | Format for Gmail        | None                    |                                                                                                                             |
| Sticky Note          | n8n-nodes-base.stickyNote             | Provides guidance and comments                  | None                   | None                    | Multiple sticky notes with setup and configuration instructions for various workflow parts                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: `Schedule Trigger`  
   - Set interval to 24 hours (every day).  
   - Configure timezone as needed to control execution time.

2. **Add 'Prepare News Query' Code Node**  
   - Type: `Code` (JavaScript)  
   - Paste the following JS snippet to generate today's date and construct a news search query:  
     ```js
     const today = new Date();
     const dateString = today.toLocaleDateString('en-US', { 
       weekday: 'long', 
       year: 'numeric', 
       month: 'long', 
       day: 'numeric' 
     });

     return [
       {
         json: {
           query: `Search for today's most important news stories from ${dateString}. Include major headlines from technology, business, politics, science, and world events. Focus on the most significant and impactful stories.`,
           date: dateString
         }
       }
     ];
     ```  
   - Connect Schedule Trigger output to this node input.

3. **Configure OpenRouter Chat Model Node**  
   - Type: `LangChain Chat Model - OpenRouter`  
   - Add OpenRouter API credentials (must be previously set in n8n credentials).  
   - No special parameters needed here, as it serves as a language model for the agent.

4. **Configure Tavily News Search HTTP Request Node**  
   - Type: `LangChain Tool - HTTP Request`  
   - Set URL: `https://api.tavily.com/search`  
   - Method: POST  
   - Body Type: JSON  
   - JSON Body template:  
     ```json
     {
       "api_key": "your-API",
       "query": "{searchTerm}",
       "search_depth": "advanced",
       "include_answer": true,
       "topic": "news",
       "include_raw_content": true,
       "max_results": 10,
       "days": 1
     }
     ```  
   - Add placeholder `searchTerm` of type string to receive dynamic queries.  
   - Insert your Tavily API key in place of `"your-API"`.

5. **Add AI News Agent Node**  
   - Type: `LangChain Agent`  
   - Set input text expression: `={{ $json.query }}`  
   - Configure system message with instructions to act as a professional news analyst strictly outputting JSON in Japanese. Use the detailed JSON schema provided to enforce output format.  
   - Connect OpenRouter Chat Model as language model, Tavily News Search as tool, and connect AI output parser to parse its output.  
   - Input from "Prepare News Query" node.

6. **Add News Output Parser Node**  
   - Type: `LangChain Output Parser Structured`  
   - Set manual schema matching the AI agent’s JSON output: title (string), summary (string), stories (array of objects with headline, summary, category).  
   - Connect AI News Agent output to this parser.

7. **Add 'Format for Gmail' Code Node**  
   - Type: `Code` (JavaScript)  
   - Paste the provided JavaScript code that builds an HTML email body with CSS styling, iterates over news stories, sets an email subject, and returns JSON with `subject` and `htmlBody`.  
   - Input: Parsed news JSON and date.  
   - Output: JSON with email components.  
   - Connect the output of AI News Agent (after parsing) to this node.

8. **Add Gmail Node 'Send a message'**  
   - Type: `Gmail`  
   - Authenticate with Gmail OAuth2 credentials.  
   - Set recipient email address in the node parameter `Send To`.  
   - Map email subject to `{{$json.subject}}`.  
   - Map message body to `{{$json.htmlBody}}`.  
   - Connect output of "Format for Gmail" node to this node.

9. **Link workflow connections**  
   - Schedule Trigger → Prepare News Query  
   - Prepare News Query → AI News Agent  
   - AI News Agent → News Output Parser → AI News Agent (loop for parsing)  
   - AI News Agent → Format for Gmail  
   - Format for Gmail → Send a message

10. **Add Sticky Notes for Documentation (Optional but Recommended)**  
    - Add notes near groups of nodes to document setup instructions and node purposes as per the original workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow produces daily news summaries in Japanese using AI and sends via Gmail.                | Workflow purpose                                                                                     |
| Required API credentials: OpenRouter API, Tavily API key, Gmail OAuth2 credentials.                   | Credential setup                                                                                     |
| AI agent output must strictly follow the JSON schema to ensure parsing and email formatting success. | Important constraint for AI processing                                                             |
| JavaScript code nodes use modern ES6 syntax and must handle edge cases such as missing data gracefully. | Code nodes notes                                                                                    |
| Gmail node requires OAuth2 setup; make sure to configure the OAuth consent and scopes correctly.     | Gmail integration specifics                                                                          |
| Tavily API key must be valid and have sufficient quota for daily news search requests.                | Tavily API usage                                                                                     |
| OpenRouter API key must be valid and support the chat model used.                                    | OpenRouter API usage                                                                                |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---