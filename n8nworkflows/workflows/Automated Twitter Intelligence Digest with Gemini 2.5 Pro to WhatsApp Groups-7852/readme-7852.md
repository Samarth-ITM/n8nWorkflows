Automated Twitter Intelligence Digest with Gemini 2.5 Pro to WhatsApp Groups

https://n8nworkflows.xyz/workflows/automated-twitter-intelligence-digest-with-gemini-2-5-pro-to-whatsapp-groups-7852


# Automated Twitter Intelligence Digest with Gemini 2.5 Pro to WhatsApp Groups

---

### 1. Workflow Overview

This workflow automates the process of scraping Twitter (X) content from selected accounts and keywords, analyzing this data with advanced AI (Google Gemini 2.5 Pro), and delivering curated, formatted intelligence digests directly to WhatsApp groups. It is designed for users who want to stay updated on tech industry trends without manual monitoring.

The workflow is logically divided into three main functional blocks:

- **1.1 Scraping Phase:** Periodic data fetching from Twitter accounts and keyword searches using TwitterAPI.io. The raw tweet data is extracted and normalized.

- **1.2 AI Analysis Phase:** Aggregates and consolidates tweet data, then sends it through an AI-powered analysis engine (Gemini 2.5 Pro). This phase filters noise, extracts intelligence, and produces an executive-style daily report.

- **1.3 WhatsApp Sending Phase:** Formats the AI-generated report into WhatsApp-optimized messages, handles JSON parsing and error correction, splits messages into batches, applies rate limiting, and sends them to a specified WhatsApp group via Evolution API.

Supporting the main blocks are nodes for scheduling, merging data streams, and utility notes for setup and configuration.

---

### 2. Block-by-Block Analysis

#### 2.1 Scraping Phase

**Overview:**  
This block fetches the latest tweets from several monitored Twitter accounts and keywords via TwitterAPI.io. It extracts relevant data fields and normalizes them for downstream processing.

**Nodes Involved:**  
- Daily Intelligence Scan (Schedule Trigger)  
- Monitor: Account 1 - Lovable (HTTP Request)  
- Monitor: Account 2 - n8n (HTTP Request)  
- Monitor: Account 3 - ElevenLabs (HTTP Request)  
- Monitor: Account 3 - OpenAI (HTTP Request)  
- Monitor: Account 4 - Anthropic (HTTP Request)  
- Search: Keyword 1 - vibecoding (HTTP Request)  
- Search: Keyword 2 - ai news (HTTP Request)  
- Search: Keyword 3 - ai agents (HTTP Request)  
- Extract Tweet Data (Account 1–5) (Code Nodes)  
- Extract Search Data (Keyword 1–3) (Code Nodes)  
- Merge (Merge Node)  

**Node Details:**

- **Daily Intelligence Scan**  
  - Type: Schedule Trigger  
  - Role: Triggers the workflow daily at 10:00 AM to initiate scraping.  
  - Config: Runs once daily on the set hour.  
  - Inputs: None (trigger node).  
  - Outputs: Triggers all HTTP requests for scraping in parallel.  
  - Errors: Trigger misconfiguration or timezone issues could cause missed runs.

- **Monitor: Account X (HTTP Request)**  
  - Type: HTTP Request  
  - Role: Fetches last tweets from specified Twitter user IDs using TwitterAPI.io.  
  - Config: Uses HTTP Header Authentication with TwitterAPI.io credentials; queries by userId.  
  - Inputs: Trigger from schedule.  
  - Outputs: JSON response with tweets.  
  - Errors: API key invalidation, rate limits, network errors, or malformed responses.

- **Search: Keyword X (HTTP Request)**  
  - Type: HTTP Request  
  - Role: Performs advanced Twitter searches with specific keywords and "Latest" query type.  
  - Config: Same authentication as above; queries include keyword and queryType.  
  - Inputs: Trigger from schedule.  
  - Outputs: JSON search results.  
  - Errors: Same as account monitors plus possible empty results.

- **Extract Tweet Data (Account X) / Extract Search Data (Keyword X)**  
  - Type: Code Node (JavaScript)  
  - Role: Parses raw Twitter API JSON responses, validates structure, and extracts key tweet fields: text, retweetCount, replyCount, likeCount, viewCount, id, url, createdAt, author.  
  - Config: Custom JS code for robust extraction and filtering invalid data.  
  - Inputs: HTTP Request JSON responses.  
  - Outputs: Array of simplified tweet objects for merging.  
  - Errors: JSON format errors, missing fields, empty arrays cause empty outputs.

- **Merge**  
  - Type: Merge Node  
  - Role: Combines outputs of all extract nodes (8 inputs total) into a single data stream for further processing.  
  - Config: Number of inputs set to 8; merges all incoming data into one list.  
  - Inputs: Outputs from all extract nodes.  
  - Outputs: Combined tweet data array.  
  - Errors: Partial inputs or empty arrays reduce data completeness.

**Sticky Note:**  
- "X SCRAPING PHASE" explains account monitoring, keyword search, data processing, and error handling.

---

#### 2.2 AI Analysis Phase

**Overview:**  
This block consolidates the merged tweet data, normalizes field names, aggregates content, and sends it to the Gemini 2.5 Pro AI model for advanced analysis and report generation.

**Nodes Involved:**  
- Normalize Field Names (Set Node)  
- Aggregate Tweet Data (Aggregate Node)  
- Prepare for AI Analysis (Summarize Node)  
- Gemini 2.5 Pro (AI Language Model Node)  
- AI Analysis Engine (Chain LLM Node)  

**Node Details:**

- **Normalize Field Names**  
  - Type: Set Node  
  - Role: Standardizes property names to uniform field names used downstream (e.g., `contenido` from `text`).  
  - Config: Maps original tweet fields to normalized strings for content and engagement metrics.  
  - Inputs: Merged tweets array.  
  - Outputs: Normalized tweet objects.  
  - Errors: Misconfigured assignments may cause missing or incorrect data.

- **Aggregate Tweet Data**  
  - Type: Aggregate Node  
  - Role: Aggregates all tweet items into a single consolidated data object or list.  
  - Config: Uses "aggregate all item data" option.  
  - Inputs: Normalized tweets.  
  - Outputs: Aggregated data for summarization.  
  - Errors: Empty input results in no aggregation.

- **Prepare for AI Analysis**  
  - Type: Summarize Node  
  - Role: Concatenates all tweet contents into a single field (`contenido`) to prepare a comprehensive text input for AI.  
  - Config: Splits by `contenido` field and concatenates with aggregation.  
  - Inputs: Aggregated tweet data.  
  - Outputs: Single summarized text string.  
  - Errors: Empty or malformed input may lead to empty summaries.

- **Gemini 2.5 Pro**  
  - Type: AI Language Model (OpenRouter)  
  - Role: Processes the batch of concatenated tweet data to generate a structured intelligence report.  
  - Config: Uses "google/gemini-2.5-pro" model with OpenRouter API credentials.  
  - Inputs: Text containing concatenated tweets.  
  - Outputs: AI-generated JSON report with daily insights.  
  - Errors: Authentication failures, API rate limits, model downtime.

- **AI Analysis Engine**  
  - Type: Chain LLM Node  
  - Role: Executes a detailed AI prompt logic to parse, filter noise, extract key news, synthesize highlights, and assemble final JSON with daily report and data items.  
  - Config: Uses a custom prompt instructing the AI to act as a tech intelligence analyst "Mr. X ❄️" from Antarctica, applying filtering and summarization logic.  
  - Inputs: Summarized tweet data.  
  - Outputs: JSON with `daily_report` and `data_for_sheets` keys.  
  - Errors: Prompt errors, output formatting issues, API timeouts.

**Sticky Note:**  
- "AI ANALYSIS PHASE" details AI-powered filtering, trend identification, report generation, JSON validation, and quality control.

---

#### 2.3 WhatsApp Sending Phase

**Overview:**  
This block formats the AI-generated report for WhatsApp, fixes any JSON errors, parses the JSON structure, splits messages into batches, applies rate limiting to avoid spam detection, and sends the messages to a WhatsApp group using the Evolution API.

**Nodes Involved:**  
- GPT-4.1 Formatter (AI Language Model Node)  
- Auto-fix JSON Errors (Output Parser Autofixing Node)  
- JSON Structure Parser (Output Parser Structured Node)  
- Format for WhatsApp (Chain LLM Node)  
- Split WhatsApp Messages (Split Out Node)  
- Process Message Batches (Split In Batches Node)  
- Rate Limit Protection (Wait Node)  
- Send to WhatsApp Group (Evolution API Node)  

**Node Details:**

- **Format for WhatsApp**  
  - Type: Chain LLM Node  
  - Role: Receives AI report, converts it into a WhatsApp-optimized format with mobile-friendly markdown and splits content into digestible chunks.  
  - Config: Uses a strict system prompt specifying JSON output with an array of message strings; enforces word count matching input.  
  - Inputs: AI-generated daily report JSON.  
  - Outputs: JSON object with `messages` array.  
  - Errors: Malformed input or output exceeding limits.

- **GPT-4.1 Formatter**  
  - Type: AI Language Model (OpenRouter)  
  - Role: Further refines the WhatsApp message formatting using GPT-4.1 to ensure clarity and compliance with constraints.  
  - Config: Temperature set to 0 for deterministic output.  
  - Inputs: Output from Format for WhatsApp.  
  - Outputs: Refined WhatsApp messages JSON.  
  - Errors: API or formatting errors.

- **Auto-fix JSON Errors**  
  - Type: Output Parser Autofixing Node  
  - Role: Detects and corrects JSON formatting errors in AI outputs automatically.  
  - Config: Uses instructions to retry and fix errors until valid JSON is produced.  
  - Inputs: AI output text.  
  - Outputs: Valid JSON object.  
  - Errors: Persistent errors may cause workflow failure.

- **JSON Structure Parser**  
  - Type: Output Parser Structured Node  
  - Role: Parses the corrected JSON string into structured data with a required schema (object containing an array of strings under `mensajes`).  
  - Config: Manual schema definition matching expected message array.  
  - Inputs: Corrected JSON from autofix node.  
  - Outputs: Parsed JSON object for splitting.  
  - Errors: Schema mismatches.

- **Split WhatsApp Messages**  
  - Type: Split Out Node  
  - Role: Splits the array of WhatsApp messages into individual items for batch processing.  
  - Config: Splits by field `output.mensajes`.  
  - Inputs: Parsed JSON object.  
  - Outputs: Individual message items.  
  - Errors: Empty arrays cause no output.

- **Process Message Batches**  
  - Type: Split In Batches Node  
  - Role: Processes messages in batches, allowing controlled sending flow.  
  - Config: Default batch size (not specified).  
  - Inputs: Individual message items.  
  - Outputs: Batches of messages.  
  - Errors: Batch misconfiguration or empty messages.

- **Rate Limit Protection**  
  - Type: Wait Node  
  - Role: Implements a 0.1-second delay between sending messages to avoid WhatsApp spam detection and maintain message order.  
  - Config: Wait time set to 0.1 seconds.  
  - Inputs: Message batches.  
  - Outputs: Delayed messages.  
  - Errors: Timing errors or excessive delays.

- **Send to WhatsApp Group**  
  - Type: Evolution API Node  
  - Role: Sends each WhatsApp message to a specified WhatsApp group using Evolution API credentials.  
  - Config: Uses `messages-api` resource, group ID set (e.g., "120363419788967600@g.us"), sends text message with link preview enabled.  
  - Inputs: Delayed message batches.  
  - Outputs: Message send confirmations.  
  - Errors: Authentication failures, group ID errors, network issues.

**Sticky Note:**  
- "WHATSAPP SENDING PHASE" describes message formatting, batch processing, rate limiting, and configuration requirements for WhatsApp delivery.

---

### 3. Summary Table

| Node Name                      | Node Type                          | Functional Role                               | Input Node(s)                                               | Output Node(s)                      | Sticky Note                                               |
|--------------------------------|----------------------------------|-----------------------------------------------|-------------------------------------------------------------|-----------------------------------|-----------------------------------------------------------|
| Daily Intelligence Scan         | Schedule Trigger                 | Starts workflow daily at 10:00 AM              | None                                                        | Search: Keyword 1 - vibecoding, Monitor Accounts, etc. |                                                           |
| Monitor: Account 1 - Lovable    | HTTP Request                    | Fetch tweets from Lovable Twitter account      | Daily Intelligence Scan                                     | Extract Tweet Data (Account 1)    |                                                           |
| Monitor: Account 2 - n8n        | HTTP Request                    | Fetch tweets from n8n Twitter account          | Daily Intelligence Scan                                     | Extract Tweet Data (Account 2)    |                                                           |
| Monitor: Account 3 - ElevenLabs | HTTP Request                    | Fetch tweets from ElevenLabs Twitter account   | Daily Intelligence Scan                                     | Extract Tweet Data (Account 3)    |                                                           |
| Monitor: Account 3 - OpenAI     | HTTP Request                    | Fetch tweets from OpenAI Twitter account       | Daily Intelligence Scan                                     | Extract Tweet Data (Account 4)    |                                                           |
| Monitor: Account 4 - Anthropic  | HTTP Request                    | Fetch tweets from Anthropic Twitter account    | Daily Intelligence Scan                                     | Extract Tweet Data (Account 5)    |                                                           |
| Search: Keyword 1 - vibecoding  | HTTP Request                    | Perform keyword search for "vibecoding" tweets | Daily Intelligence Scan                                     | Extract Search Data (Keyword 1)   |                                                           |
| Search: Keyword 2 - ai news     | HTTP Request                    | Perform keyword search for "ai news" tweets    | Daily Intelligence Scan                                     | Extract Search Data (Keyword 2)   |                                                           |
| Search: Keyword 3 - ai agents   | HTTP Request                    | Perform keyword search for "ai agents" tweets  | Daily Intelligence Scan                                     | Extract Search Data (Keyword 3)   |                                                           |
| Extract Tweet Data (Account 1)  | Code Node                      | Extracts and normalizes tweet fields (Account1)| Monitor: Account 1                                         | Merge                            |                                                           |
| Extract Tweet Data (Account 2)  | Code Node                      | Extracts and normalizes tweet fields (Account2)| Monitor: Account 2                                         | Merge                            |                                                           |
| Extract Tweet Data (Account 3)  | Code Node                      | Extracts and normalizes tweet fields (Account3)| Monitor: Account 3                                         | Merge                            |                                                           |
| Extract Tweet Data (Account 4)  | Code Node                      | Extracts and normalizes tweet fields (Account4)| Monitor: Account 3 - OpenAI                                | Merge                            |                                                           |
| Extract Tweet Data (Account 5)  | Code Node                      | Extracts and normalizes tweet fields (Account5)| Monitor: Account 4 - Anthropic                              | Merge                            |                                                           |
| Extract Search Data (Keyword 1) | Code Node                      | Extracts and normalizes tweet fields (Keyword1)| Search: Keyword 1                                         | Merge                            |                                                           |
| Extract Search Data (Keyword 2) | Code Node                      | Extracts and normalizes tweet fields (Keyword2)| Search: Keyword 2                                         | Merge                            |                                                           |
| Extract Search Data (Keyword 3) | Code Node                      | Extracts and normalizes tweet fields (Keyword3)| Search: Keyword 3                                         | Merge                            |                                                           |
| Merge                          | Merge Node                     | Combines all extracted tweet data into one array| Extract Tweet Data and Extract Search Data nodes (8 inputs)| Normalize Field Names             |                                                           |
| Normalize Field Names           | Set Node                      | Standardizes field names for uniform processing| Merge                                                      | Aggregate Tweet Data             |                                                           |
| Aggregate Tweet Data            | Aggregate Node                | Aggregates all tweet data into consolidated form| Normalize Field Names                                      | Prepare for AI Analysis          |                                                           |
| Prepare for AI Analysis         | Summarize Node               | Concatenates tweet texts for AI input          | Aggregate Tweet Data                                       | AI Analysis Engine               |                                                           |
| Gemini 2.5 Pro                 | AI Language Model (OpenRouter)| Processes concatenated tweet batch with AI     | AI Analysis Engine                                        | AI Analysis Engine               | See "AI ANALYSIS PHASE" sticky note                       |
| AI Analysis Engine             | Chain LLM Node               | Performs detailed AI analysis and report creation| Prepare for AI Analysis                                   | Format for WhatsApp              |                                                           |
| Format for WhatsApp             | Chain LLM Node               | Converts AI report into WhatsApp message format| AI Analysis Engine                                        | Split WhatsApp Messages          | See "WHATSAPP SENDING PHASE" sticky note                  |
| GPT-4.1 Formatter              | AI Language Model (OpenRouter)| Refines WhatsApp message formatting             | Format for WhatsApp                                      | Auto-fix JSON Errors             |                                                           |
| Auto-fix JSON Errors            | Output Parser Autofixing     | Auto-corrects JSON formatting errors            | JSON Structure Parser                                   | Format for WhatsApp              |                                                           |
| JSON Structure Parser          | Output Parser Structured     | Parses JSON output into structured data         | GPT-4.1 Formatter                                       | Auto-fix JSON Errors             |                                                           |
| Split WhatsApp Messages         | Split Out Node               | Splits WhatsApp messages array into individual messages| Format for WhatsApp                                   | Process Message Batches          |                                                           |
| Process Message Batches         | Split In Batches Node        | Processes messages in batches                    | Split WhatsApp Messages                                | Send to WhatsApp Group / End     |                                                           |
| Rate Limit Protection           | Wait Node                   | 0.1 second delay between messages to avoid spam | Send to WhatsApp Group                                  | Process Message Batches          |                                                           |
| Send to WhatsApp Group          | Evolution API Node          | Sends messages to WhatsApp group using Evolution API| Rate Limit Protection                                 | Rate Limit Protection            |                                                           |
| Sticky Note, Sticky Note1-5, 15| Sticky Note                 | Documentation and instructions                   | None                                                    | None                           | See notes in sections above                               |
| Find your Groups               | Evolution API Node (disabled) | Helper node to find WhatsApp groups for setup   | None                                                    | None                           | Disabled; only for setup assistance                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Name: `Daily Intelligence Scan`  
   - Type: Schedule Trigger  
   - Trigger: Daily at 10:00 AM (local timezone e.g., Europe/Madrid)  

2. **Create HTTP Request Nodes for Twitter Account Monitoring**  
   - For each account (Lovable, n8n, ElevenLabs, OpenAI, Anthropic):  
     - Name: e.g., `Monitor: Account 1 - Lovable`  
     - Type: HTTP Request  
     - Method: GET  
     - URL: `https://api.twitterapi.io/twitter/user/last_tweets`  
     - Query Parameters: `userId` with the target Twitter user ID  
     - Authentication: HTTP Header Auth with TwitterAPI.io credentials  
     - Connect each node’s input to the schedule trigger output in parallel  

3. **Create HTTP Request Nodes for Keyword Searches**  
   - For each keyword (`vibecoding`, `ai news`, `ai agents`):  
     - Name: e.g., `Search: Keyword 1 - vibecoding`  
     - Type: HTTP Request  
     - Method: GET  
     - URL: `https://api.twitterapi.io/twitter/tweet/advanced_search`  
     - Query Parameters: `query` = keyword, `queryType` = `Latest`  
     - Authentication: Same as above  
     - Connect each node’s input to the schedule trigger output in parallel  

4. **Create Code Nodes to Extract Tweet Data**  
   - For each account and keyword search node, create a corresponding code node named `Extract Tweet Data (Account X)` or `Extract Search Data (Keyword X)`  
   - Use the provided JavaScript code to validate input JSON and extract fields: text, retweetCount, replyCount, likeCount, viewCount, id, url, createdAt, author  
   - Connect each HTTP Request node output to its corresponding code node  

5. **Create a Merge Node**  
   - Name: `Merge`  
   - Type: Merge  
   - Configure number of inputs to 8 (5 accounts + 3 keyword searches)  
   - Connect all extract code nodes outputs to the merge inputs  

6. **Create Set Node to Normalize Field Names**  
   - Name: `Normalize Field Names`  
   - Type: Set  
   - Map fields: `contenido` = `text`, `retweetCount`, `likeCount`, `viewCount` copied as strings  
   - Connect merge output to this node  

7. **Create Aggregate Node**  
   - Name: `Aggregate Tweet Data`  
   - Type: Aggregate  
   - Aggregate all items together  
   - Connect normalize node output to this node  

8. **Create Summarize Node**  
   - Name: `Prepare for AI Analysis`  
   - Type: Summarize  
   - Fields to split by: `contenido`  
   - Summarize aggregation: concatenate field `data`  
   - Connect aggregate node output to this node  

9. **Create Chain LLM Node for AI Analysis Engine**  
   - Name: `AI Analysis Engine`  
   - Type: Chain LLM (Langchain)  
   - Model: Gemini 2.5 Pro via OpenRouter API (configured with credentials)  
   - Prompt: Use detailed prompt instructing to parse batch tweets, remove noise, extract insights, create JSON output with `daily_report` and `data_for_sheets`  
   - Connect summarize node output to this node  

10. **Create Chain LLM Node to Format for WhatsApp**  
    - Name: `Format for WhatsApp`  
    - Type: Chain LLM (Langchain)  
    - Model: Gemini 2.5 Pro or GPT-4.1 (if preferred)  
    - Prompt: Format AI report into WhatsApp-optimized JSON messages array, preserving word count and strict JSON structure  
    - Connect AI Analysis Engine output to this node  

11. **Create Output Parser Structured Node**  
    - Name: `JSON Structure Parser`  
    - Type: Output Parser Structured  
    - Schema: Object with `mensajes` array of strings  
    - Connect Format for WhatsApp output to this node  

12. **Create Output Parser Autofixing Node**  
    - Name: `Auto-fix JSON Errors`  
    - Type: Output Parser Autofixing  
    - Instructions: Retry with corrected JSON if parsing errors occur  
    - Connect JSON Structure Parser output to this node  

13. **Create AI Language Model Node**  
    - Name: `GPT-4.1 Formatter`  
    - Type: AI Language Model (OpenRouter)  
    - Model: GPT-4.1 with temperature 0  
    - Connect Auto-fix JSON Errors output to this node  

14. **Create Split Out Node**  
    - Name: `Split WhatsApp Messages`  
    - Type: Split Out  
    - Field to split: `output.mensajes`  
    - Connect GPT-4.1 Formatter output to this node  

15. **Create Split In Batches Node**  
    - Name: `Process Message Batches`  
    - Type: Split In Batches  
    - Batch size: default or configured per need  
    - Connect Split WhatsApp Messages output to this node  

16. **Create Wait Node for Rate Limiting**  
    - Name: `Rate Limit Protection`  
    - Type: Wait  
    - Wait time: 0.1 seconds  
    - Connect Process Message Batches output to this node  

17. **Create Evolution API Node to Send Messages**  
    - Name: `Send to WhatsApp Group`  
    - Type: Evolution API (messages-api)  
    - Resource: messages-api  
    - Operation: send text message  
    - Parameters:  
      - `remoteJid`: WhatsApp group ID (e.g., "120363419788967600@g.us")  
      - `messageText`: use expression from current item message  
      - `options_message`: enable `linkPreview`  
    - Credentials: Evolution API credentials with WhatsApp instance access  
    - Connect Rate Limit Protection output to this node  

18. **Connect nodes in the following order:**  
    - Schedule Trigger → all HTTP Requests → corresponding Extract Code Nodes → Merge → Normalize Fields → Aggregate → Summarize → AI Analysis Engine → Format for WhatsApp → JSON Structure Parser → Auto-fix JSON Errors → GPT-4.1 Formatter → Split WhatsApp Messages → Process Message Batches → Rate Limit Protection → Send to WhatsApp Group  

19. **Additional Setup:**  
    - Create credentials for TwitterAPI.io, OpenRouter, and Evolution API in n8n.  
    - Replace all placeholder user IDs and keywords as per your monitoring needs.  
    - Update WhatsApp group ID and test sending with a private group before production.  

20. **Optional:**  
    - Use the disabled `Find your Groups` Evolution API node to discover WhatsApp groups for configuration.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                          | Context or Link                                                                                  |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow is branded as an “Automated Twitter Intelligence Digest with Gemini 2.5 Pro to WhatsApp Groups.” It targets teams and content creators needing automated social updates delivered via WhatsApp.                          | Workflow title and description                                                                 |
| Setup requires accounts and credentials for TwitterAPI.io, OpenRouter (for Gemini 2.5 Pro and GPT-4.1 access), and Evolution API for WhatsApp messaging.                                                                             | Configuration checklist sticky note                                                           |
| The AI analyst persona “Mr. X ❄️” operates from Antarctica, providing unique and analytical tech insights.                                                                                                                          | AI Analysis Engine prompt details                                                             |
| Feedback and consulting services are offered by the workflow author Daniel Lianes, with links to schedule calls and follow on LinkedIn provided in sticky notes.                                                                    | See Sticky Note15 for contact and consulting links                                            |
| The workflow respects strict content fidelity to original tweet content and AI outputs; no unauthorized additions or fabrications are permitted.                                                                                     | Content fidelity rules in AI prompt                                                          |
| TwitterAPI.io is used for scraping, which requires valid API keys and respects rate limits; monitor API usage carefully.                                                                                                            | X SCRAPING PHASE sticky note                                                                  |
| Rate limiting and message batching avoid WhatsApp spam detection, preserving message order and timing.                                                                                                                              | WHATSAPP SENDING PHASE sticky note                                                           |
| For any issues with WhatsApp group IDs or sending, use the disabled “Find your Groups” node to list groups accessible by your Evolution API instance.                                                                               | Sticky Note5 and disabled Find your Groups node                                              |
| For further automation assistance or customization, consider personalized consulting sessions with the author.                                                                                                                     | See Sticky Note15 contact info                                                                |

---

**Disclaimer:** The provided text is derived exclusively from an automated n8n workflow. All data processed is legal and public, and the workflow complies with content policies. No illegal, offensive, or protected content is involved.

---