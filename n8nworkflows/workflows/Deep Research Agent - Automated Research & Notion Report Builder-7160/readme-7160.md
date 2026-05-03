Deep Research Agent - Automated Research & Notion Report Builder

https://n8nworkflows.xyz/workflows/deep-research-agent---automated-research---notion-report-builder-7160


# Deep Research Agent - Automated Research & Notion Report Builder

---

## 1. Workflow Overview

**Purpose:**  
This workflow, named **Deep Research Agent - Automated Research & Notion Report Builder**, is an AI-powered system designed to automate the process of generating detailed research reports from user queries. It collects a research topic from a user, interactively refines the scope through clarifying questions, generates targeted search queries, retrieves relevant content from web sources, synthesizes the findings into a comprehensive report, and publishes the final output as a structured Notion page accessible to the user.

**Target Use Cases:**  
- Automated research assistance and report generation for knowledge workers, students, or content creators.  
- Integration with chatbots, forms, or applications via webhook to enable on-demand research reports.  
- Centralized documentation and knowledge management inside Notion databases.

**Logical Blocks:**

- **1.1 Input Reception & Clarification**  
  Receives user research topic via webhook, interacts with user to clarify research goals using AI agents, and manages conversational memory.

- **1.2 Research Strategy & Query Generation**  
  Upon confirmation, generates a research title/description and formulates 2-3 distinct search queries to explore the topic.

- **1.3 Data Gathering & Extraction**  
  For each search query, searches the web via Tavily API, selects the most relevant link, extracts detailed content, and aggregates all results.

- **1.4 Report Compilation & Formatting**  
  Uses AI to compose a structured, markdown-formatted report synthesizing all extracted data, then converts the markdown to HTML.

- **1.5 HTML to Notion Blocks Conversion**  
  Splits HTML report into manageable chunks, converts each chunk into Notion-compatible blocks using AI, filters and prepares them for insertion.

- **1.6 Final Storage & Response Delivery**  
  Updates the existing Notion page with all content blocks, marks the report as complete, and sends the Notion report link back to the user.

---

## 2. Block-by-Block Analysis

### 1.1 Input Reception & Clarification

**Overview:**  
This block handles receiving the initial user input via webhook, manages session-based memory for context, and uses an AI "Strategy Agent" to ask clarifying questions or confirm the research plan draft.

**Nodes Involved:**  
- Webhook  
- Simple Memory  
- OpenRouter Chat Model  
- Strategy Agent  
- Structured Output Parser  
- Switch  
- Respond to Webhook  
- Respond to Webhook1  
- Code  

**Node Details:**

- **Webhook**  
  - Type: HTTP Webhook, entry point for user POST requests.  
  - Configured for POST method with path `1c86c408-aeed-40c5-b4ba-aad5f4cdf0ad`.  
  - Response mode is set to respond with a node.  
  - Inputs: External user requests.  
  - Outputs: Data forwarded to Strategy Agent.  
  - Edge cases: Missing or malformed request data could cause failures.

- **Simple Memory**  
  - Type: LangChain Memory Buffer Window.  
  - Role: Maintains conversational context keyed by session ID (extracted from input JSON path `message.chat.id` or `body.session_id`).  
  - Context window length: 10 messages for context.  
  - Inputs: User messages from webhook.  
  - Outputs: Context-enriched input to Strategy Agent.  
  - Failure: Missing session ID or memory service issues.

- **OpenRouter Chat Model**  
  - Type: AI language model node using OpenRouter (Anthropic Claude 3.5 Sonnet).  
  - Role: Provides the language model backend for the Strategy Agent.  
  - Inputs: Messages with user prompt and memory context.  
  - Outputs: AI-generated responses.  
  - Edge cases: API authentication errors, timeouts.

- **Strategy Agent**  
  - Type: LangChain Agent node leveraging OpenRouter Chat Model and memory.  
  - Role: Interprets user input, asks clarifying questions (up to 3), drafts a research plan, and awaits user confirmation.  
  - Uses a custom prompt with instructions for JSON output (`is_pass_next` boolean and `message` string).  
  - Outputs parsed via Structured Output Parser.  
  - Inputs: User input + memory context.  
  - Outputs: JSON indicating whether to pass to next step or request feedback.  
  - Failures: Parsing errors, model misunderstandings.

- **Structured Output Parser**  
  - Type: LangChain output parser.  
  - Role: Parses agent's JSON output for `is_pass_next` and `message`.  
  - Inputs: Raw AI output.  
  - Outputs: Parsed structured JSON.  
  - Edge cases: Invalid or malformed JSON output.

- **Switch**  
  - Type: Conditional routing node.  
  - Role: Routes flow based on `is_pass_next` boolean from Strategy Agent output.  
  - Two outputs:  
    - Feedback (if `is_pass_next` is false) → Responds with clarifying questions.  
    - Pass (if `is_pass_next` is true) → Proceeds to next research phase.  
  - Inputs: Structured output JSON.  
  - Outputs: To Respond to Webhook or Respond to Webhook1.  
  - Edge cases: Unexpected values causing no route.

- **Respond to Webhook**  
  - Type: Respond node.  
  - Role: Sends clarifying questions back to the user if more input is needed.  
  - Inputs: Switch "Feedback" output.  
  - Outputs: HTTP response to the webhook caller.  
  - Edge cases: Network issues.

- **Respond to Webhook1**  
  - Type: Respond node.  
  - Role: Acknowledges user confirmation and informs that report preparation has started.  
  - Inputs: Switch "Pass" output.  
  - Outputs: HTTP response.  
  - Edge cases: Same as above.

- **Code**  
  - Type: JavaScript code node.  
  - Role: Generates a random 6-digit `randomId` used as a request identifier for the session/report.  
  - Inputs: Triggered after user confirmation.  
  - Outputs: `{ randomId }` JSON object.  
  - Edge cases: None significant.

---

### 1.2 Research Strategy & Query Generation

**Overview:**  
This block generates a research title and description for Notion, creates the Notion database page, and generates up to three search queries relevant to the confirmed research topic.

**Nodes Involved:**  
- OpenAI1  
- Notion  
- Search Query Agent  
- Structured Output Parser1  
- Split Out  

**Node Details:**

- **OpenAI1**  
  - Type: LangChain OpenAI node (GPT-4o-mini).  
  - Role: Generates a research report title and description from the research plan draft.  
  - Input: Draft message from the `Switch` node output.  
  - Output: JSON with `title` and `description`.  
  - Edge cases: API rate limits or invalid prompts.

- **Notion**  
  - Type: Notion node (databasePage creation).  
  - Role: Creates a new page in the configured Notion database with title, description, timestamp, status "In progress", and request ID.  
  - Inputs: From OpenAI1 node output and generated `randomId` (via expression).  
  - Outputs: Created page metadata (including page ID).  
  - Requirements: Valid Notion credentials and database ID.  
  - Edge cases: API limits or permission errors.

- **Search Query Agent**  
  - Type: LangChain Agent node (OpenRouter Chat Model).  
  - Role: Generates up to 3 keyword-based search queries based on the research draft message.  
  - Input: Research plan draft message from `Switch` node.  
  - Output: JSON array of queries with goals.  
  - Output parsed by Structured Output Parser1.  
  - Edge cases: Parsing errors or unclear prompts.

- **Structured Output Parser1**  
  - Type: LangChain output parser.  
  - Role: Parses the JSON array of queries, each with fields: `query` and `researchGoal`.  
  - Input: Raw AI output from Search Query Agent.  
  - Output: Structured JSON array of queries.  
  - Edge cases: Invalid JSON format.

- **Split Out**  
  - Type: SplitOut node.  
  - Role: Splits the array of queries into individual items for iteration.  
  - Input: Structured queries array.  
  - Output: Individual query objects.  
  - Edge cases: Empty query list.

---

### 1.3 Data Gathering & Extraction

**Overview:**  
This stage loops over each search query to fetch relevant web content using the Tavily API, selects the top matching URL based on AI evaluation, and extracts detailed content from the selected source.

**Nodes Involved:**  
- Loop Over Queries  
- HTTP Request (Search)  
- Edit Fields  
- OpenAI  
- HTTP Request1 (Extract)  
- Aggregate  

**Node Details:**

- **Loop Over Queries**  
  - Type: SplitInBatches node.  
  - Role: Processes each search query item sequentially to avoid overwhelming APIs.  
  - Inputs: Individual queries from Split Out.  
  - Outputs: Each query to HTTP Request.  
  - Edge cases: Batch size considerations.

- **HTTP Request (Search)**  
  - Type: HTTP POST request.  
  - Role: Calls Tavily Search API with the query string to retrieve search results.  
  - Authentication: Generic HTTP custom auth with Tavily API key.  
  - Input: Query string from current batch item.  
  - Output: JSON search results.  
  - Edge cases: API errors, rate limits, malformed responses.

- **Edit Fields**  
  - Type: Set node.  
  - Role: Extracts the `results` field from Tavily response into a simpler field `tavily_results` for downstream nodes.  
  - Input: Search API response.  
  - Output: Simplified JSON with results.  
  - Edge cases: Missing `results` key.

- **OpenAI**  
  - Type: LangChain OpenAI node (GPT-4.1-mini).  
  - Role: Processes query, research draft, and Tavily search results to select the single most relevant URL matching the user’s query.  
  - Input: Query, draft message, and Tavily results.  
  - Output: JSON with `final_url`.  
  - Edge cases: Model errors, incomplete data.

- **HTTP Request1 (Extract)**  
  - Type: HTTP POST request.  
  - Role: Uses Tavily Extract API to get detailed content from the selected URL.  
  - Input: URL from OpenAI node output, with `extract_depth` set to "advanced" for rich content.  
  - Output: Extracted content JSON.  
  - Edge cases: URL extraction failures or API errors.

- **Aggregate**  
  - Type: Aggregate node.  
  - Role: Aggregates all extracted content items from each query into a single array for report synthesis.  
  - Input: Extracted content items from multiple queries.  
  - Output: Aggregated array.  
  - Edge cases: Empty aggregation or partial failures.

---

### 1.4 Report Compilation & Formatting

**Overview:**  
Generates the final research report in markdown using the aggregated extracted content, retrieves the existing Notion page, converts the markdown report into HTML for structured insertion.

**Nodes Involved:**  
- Report Agent  
- Get Existing Row  
- Convert to HTML  

**Node Details:**

- **Report Agent**  
  - Type: LangChain Agent node (OpenRouter Chat Model).  
  - Role: Creates a detailed, well-structured, blog-style markdown report synthesizing all extracted research content from multiple sources.  
  - Input: Aggregated extracted content and research plan draft message.  
  - Output: Markdown formatted research report.  
  - Edge cases: Model synthesis errors or incomplete data.

- **Get Existing Row**  
  - Type: Notion node (databasePage getAll with filter).  
  - Role: Retrieves the existing Notion database page created earlier by matching the `randomId`.  
  - Input: `randomId` from Code node.  
  - Output: Notion page metadata including page ID.  
  - Edge cases: Page not found or permission issues.

- **Convert to HTML**  
  - Type: Markdown node.  
  - Role: Converts the markdown report text into HTML with support for tables.  
  - Input: Markdown report from Report Agent.  
  - Output: HTML string.  
  - Edge cases: Invalid markdown or conversion failures.

---

### 1.5 HTML to Notion Blocks Conversion

**Overview:**  
Splits the HTML report into manageable chunks, converts each chunk into JSON blocks compatible with Notion’s API using AI, and filters valid blocks.

**Nodes Involved:**  
- HTML to Array  
- Tags to Items  
- Notion Block Generator  
- Parse JSON blocks  
- Valid Blocks  

**Node Details:**

- **HTML to Array**  
  - Type: Set node.  
  - Role: Uses regex to extract block-level HTML elements (tables, lists, paragraphs) into an array named `tag`.  
  - Input: HTML string from Convert to HTML node.  
  - Output: Array of HTML tags.  
  - Edge cases: Invalid HTML or unmatched tags.

- **Tags to Items**  
  - Type: SplitOut node.  
  - Role: Splits the array of HTML tags into individual items for processing.  
  - Input: Array from HTML to Array node.  
  - Output: Single HTML tag strings.  
  - Edge cases: Empty array.

- **Notion Block Generator**  
  - Type: LangChain Chain LLM node (Google Gemini Chat Model).  
  - Role: Converts each HTML string into a Notion API block JSON structure using a detailed prompt with examples for headings, lists, tables, and invalid HTML handling.  
  - Input: Single HTML tag string.  
  - Output: JSON block(s) representing Notion blocks.  
  - Edge cases: Conversion failures, invalid HTML.

- **Parse JSON blocks**  
  - Type: Set node with JavaScript code.  
  - Role: Parses the AI-generated JSON string into JSON objects, adjusts heading levels down by one if needed, and converts single blocks into arrays for consistent processing.  
  - Input: Raw AI response text.  
  - Output: Valid JSON array of Notion blocks.  
  - Edge cases: Parsing errors, invalid JSON.

- **Valid Blocks**  
  - Type: Filter node.  
  - Role: Filters out any objects with an `error` property, keeping only valid Notion blocks for insertion.  
  - Input: Parsed blocks array.  
  - Output: Filtered valid blocks.  
  - Edge cases: All blocks filtered out if errors.

---

### 1.6 Final Storage & Response Delivery

**Overview:**  
Processes each valid Notion block, aggregates them, updates the Notion page with the full content, marks the report as done, and sends the final report URL back to the user.

**Nodes Involved:**  
- For Each Block...  
- Aggregate1  
- HTTP Request2  
- Notion1  
- HTTP Request3  

**Node Details:**

- **For Each Block...**  
  - Type: SplitInBatches node.  
  - Role: Iterates over each valid Notion block to process them individually for insertion.  
  - Output: Blocks to HTTP Request2 and Aggregate1 in parallel paths.  
  - Edge cases: Large number of blocks may affect performance.

- **Aggregate1**  
  - Type: Aggregate node.  
  - Role: Aggregates all processed blocks after insertion preparation.  
  - Input: Processed blocks output.  
  - Output: Aggregated blocks for final update.  
  - Edge cases: Empty aggregation.

- **HTTP Request2**  
  - Type: HTTP PATCH request.  
  - Role: Sends a request to the Notion API to append the block children (content) to the existing Notion page.  
  - Input: Blocks from For Each Block... node.  
  - Authentication: Notion API key.  
  - Edge cases: API limits, request timeouts.

- **Notion1**  
  - Type: Notion node (update page).  
  - Role: Updates the Notion page’s status property to "Done" and sets the "Last Updated" timestamp.  
  - Input: Page ID from earlier and current timestamp.  
  - Edge cases: Permission errors.

- **HTTP Request3**  
  - Type: HTTP POST request.  
  - Role: Sends a final summary HTTP request (likely to an external system or user notification endpoint) containing report title, URL, status, and session ID.  
  - Input: Notion page metadata and session data.  
  - Edge cases: External endpoint failures.

---

## 3. Summary Table

| Node Name              | Node Type                               | Functional Role                               | Input Node(s)                 | Output Node(s)                | Sticky Note                                                                                       |
|------------------------|---------------------------------------|-----------------------------------------------|------------------------------|------------------------------|-------------------------------------------------------------------------------------------------|
| Webhook                | n8n Webhook                           | Entry point for user input                      |                              | Strategy Agent               |                                                                                                 |
| Simple Memory          | LangChain Memory Buffer Window       | Maintains session-based conversational memory | Webhook                      | Strategy Agent               |                                                                                                 |
| OpenRouter Chat Model  | AI Language Model (OpenRouter)       | Provides LLM backend for Strategy Agent        |                              | Strategy Agent               |                                                                                                 |
| Strategy Agent         | LangChain Agent                      | Clarifies research topic and drafts plan       | Webhook, Simple Memory, OpenRouter Chat Model | Structured Output Parser      | **Strategy Agent & Clarification Stage**: Handles clarifying questions or confirmation.          |
| Structured Output Parser | LangChain Output Parser              | Parses Strategy Agent JSON output               | Strategy Agent               | Switch                      |                                                                                                 |
| Switch                 | Conditional Switch                   | Routes based on clarification status           | Structured Output Parser     | Respond to Webhook, Respond to Webhook1 |                                                                                                 |
| Respond to Webhook     | n8n Respond Node                    | Sends clarifying questions to user             | Switch (Feedback)            |                              |                                                                                                 |
| Respond to Webhook1    | n8n Respond Node                    | Sends confirmation acknowledgment              | Switch (Pass)                | Code                        |                                                                                                 |
| Code                   | n8n Code Node                      | Generates random request ID                      | Respond to Webhook1          | OpenAI1                     |                                                                                                 |
| OpenAI1                | LangChain OpenAI                    | Generates research title and description       | Code                        | Notion                      | **Notion Page Creation & Search Query Generation**: Creates Notion page and generates queries.  |
| Notion                 | n8n Notion                         | Creates Notion database page                     | OpenAI1                     | Search Query Agent           |                                                                                                 |
| Search Query Agent     | LangChain Agent                    | Generates search queries from research draft    | Notion                      | Structured Output Parser1    |                                                                                                 |
| Structured Output Parser1 | LangChain Output Parser            | Parses search query JSON output                  | Search Query Agent           | Split Out                   |                                                                                                 |
| Split Out              | n8n SplitOut                       | Splits search queries array                      | Structured Output Parser1    | Loop Over Queries            |                                                                                                 |
| Loop Over Queries      | n8n SplitInBatches                 | Iterates over each search query                  | Split Out                   | Aggregate, HTTP Request      |                                                                                                 |
| HTTP Request           | n8n HTTP Request                   | Calls Tavily Search API for results              | Loop Over Queries           | Edit Fields                 | **Search & Content Extraction**: Searches Tavily API for relevant results.                      |
| Edit Fields            | n8n Set                           | Extracts and renames Tavily results field        | HTTP Request                | OpenAI                      |                                                                                                 |
| OpenAI                 | LangChain OpenAI                   | Selects most relevant URL from search results   | Edit Fields                 | HTTP Request1               |                                                                                                 |
| HTTP Request1          | n8n HTTP Request                   | Extracts detailed content from chosen URL        | OpenAI                      | Loop Over Queries            |                                                                                                 |
| Aggregate              | n8n Aggregate                     | Aggregates extracted content from all queries   | Loop Over Queries           | Report Agent                |                                                                                                 |
| Report Agent           | LangChain Agent                   | Generates final markdown research report         | Aggregate                   | Get Existing Row            | **Report Compilation & Formatting**: Synthesizes research into markdown report.                 |
| Get Existing Row       | n8n Notion                        | Retrieves Notion page by Request ID               | Report Agent                | Convert to HTML             |                                                                                                 |
| Convert to HTML        | n8n Markdown                      | Converts markdown report to HTML                   | Get Existing Row            | HTML to Array               | **HTML to Notion Block Conversion**: Prepares HTML chunks for Notion.                           |
| HTML to Array          | n8n Set                           | Extracts HTML block-level elements into array    | Convert to HTML             | Tags to Items               |                                                                                                 |
| Tags to Items          | n8n SplitOut                      | Splits HTML array into individual tags            | HTML to Array              | Notion Block Generator      |                                                                                                 |
| Notion Block Generator | LangChain Chain LLM (Google Gemini) | Converts HTML tags into Notion block JSON         | Tags to Items              | Parse JSON blocks           |                                                                                                 |
| Parse JSON blocks      | n8n Set                           | Parses AI JSON output and normalizes headings     | Notion Block Generator     | Valid Blocks                |                                                                                                 |
| Valid Blocks           | n8n Filter                        | Filters out invalid or error blocks                | Parse JSON blocks          | For Each Block...           |                                                                                                 |
| For Each Block...      | n8n SplitInBatches                | Iterates over valid Notion blocks                  | Valid Blocks               | HTTP Request2, Aggregate1   | **Final Storage & Response**: Inserts blocks into Notion and finalizes report.                  |
| HTTP Request2          | n8n HTTP Request                 | PATCH request to append content blocks to Notion  | For Each Block...          |                              |                                                                                                 |
| Aggregate1             | n8n Aggregate                   | Aggregates processed blocks for final update       | For Each Block...          | Notion1                    |                                                                                                 |
| Notion1                | n8n Notion                      | Updates Notion page status to "Done"               | Aggregate1                 | HTTP Request3               |                                                                                                 |
| HTTP Request3          | n8n HTTP Request                | Sends final report link and status to user/system | Notion1                    |                              |                                                                                                 |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node:**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `1c86c408-aeed-40c5-b4ba-aad5f4cdf0ad`  
   - Response Mode: Response Node

2. **Add Simple Memory Node:**  
   - Type: LangChain Memory Buffer Window  
   - Session Key: Expression `{{$json?.message?.chat?.id || $json?.body?.session_id}}`  
   - Context Window Length: 10

3. **Add OpenRouter Chat Model Node:**  
   - Type: LangChain LLM Chat (OpenRouter)  
   - Model: `anthropic/claude-3.5-sonnet`

4. **Add Strategy Agent Node:**  
   - Type: LangChain Agent  
   - LLM: Connect to OpenRouter Chat Model  
   - Input Text: Custom prompt to greet, ask "What would you like to research?", clarify with up to 3 questions, output JSON with `is_pass_next` and `message`.  
   - Enable Output Parser with manual schema `{ "is_pass_next": "boolean", "message": "string" }`.

5. **Add Structured Output Parser Node:**  
   - Type: LangChain Output Parser  
   - Schema: Manual, matching Strategy Agent output.

6. **Add Switch Node:**  
   - Condition: Check if `output.is_pass_next` is `false` or `true`.  
   - Output 1 ("Feedback") if false → Respond to Webhook (clarifying questions).  
   - Output 2 ("Pass") if true → Respond to Webhook1 (confirmation) and then proceed.

7. **Add Respond to Webhook Nodes:**  
   - Respond to Webhook: Respond with clarifying questions message from Strategy Agent.  
   - Respond to Webhook1: Respond with confirmation message "Thank you for your response. We are preparing your report..."

8. **Add Code Node:**  
   - JavaScript code to generate a random 6-digit `randomId` for report identification.

9. **Add OpenAI1 Node:**  
   - Type: LangChain OpenAI (GPT-4o-mini)  
   - Prompt: Generate `title` and `description` JSON from research draft message.  
   - Input: From Strategy Agent's confirmed draft message.

10. **Add Notion Node:**  
    - Create a new page in Notion database.  
    - Database ID: Your Notion research reports database.  
    - Properties: Title, Description, Request ID (randomId), Created Time (now), Status ("In progress").

11. **Add Search Query Agent Node:**  
    - Type: LangChain Agent (OpenRouter Chat Model)  
    - Prompt: Generate up to 3 unique keyword-based search queries from research draft.  
    - Enable output parser with schema for array of queries.

12. **Add Structured Output Parser1 Node:**  
    - Parse the queries JSON array.

13. **Add Split Out Node:**  
    - Split the parsed queries array for iteration.

14. **Add Loop Over Queries Node:**  
    - Type: SplitInBatches  
    - Process each query sequentially.

15. **Add HTTP Request Node:**  
    - POST to Tavily Search API with `query`.  
    - Authentication: Generic HTTP custom auth with Tavily API key.

16. **Add Edit Fields Node:**  
    - Extract `results` field from Tavily search response into `tavily_results`.

17. **Add OpenAI Node:**  
    - GPT-4.1-mini to select most relevant URL from Tavily results.  
    - Input: Query, research draft message, tavily_results.

18. **Add HTTP Request1 Node:**  
    - POST to Tavily Extract API with chosen URL to get full content with `extract_depth=advanced`.

19. **Add Aggregate Node:**  
    - Aggregate all extracted contents from all queries.

20. **Add Report Agent Node:**  
    - LangChain Agent (OpenRouter Chat Model)  
    - Prompt: Generate a detailed markdown research report from research draft and all extracted content.  
    - Organize with headings, sources superscript, lists, tables.

21. **Add Get Existing Row Node:**  
    - Retrieve Notion page by matching `Request ID` with `randomId`.

22. **Add Convert to HTML Node:**  
    - Convert markdown report to HTML with tables enabled.

23. **Add HTML to Array Node:**  
    - Extract block-level HTML tags (tables, lists, paragraphs) into an array.

24. **Add Tags to Items Node:**  
    - Split array into individual tags for processing.

25. **Add Notion Block Generator Node:**  
    - LangChain Chain LLM (Google Gemini Chat Model).  
    - Convert each HTML tag string into Notion block JSON with detailed prompt instructions.

26. **Add Parse JSON blocks Node:**  
    - Parse JSON string from AI response, adjust heading levels, output array of blocks.

27. **Add Valid Blocks Node:**  
    - Filter out blocks with errors.

28. **Add For Each Block... Node:**  
    - SplitInBatches over valid blocks for iteration.

29. **Add HTTP Request2 Node:**  
    - PATCH request to Notion API to append block children to the page.  
    - Authentication: Notion API key.

30. **Add Aggregate1 Node:**  
    - Aggregate processed blocks.

31. **Add Notion1 Node:**  
    - Update Notion page status to "Done" and set "Last Updated" timestamp.

32. **Add HTTP Request3 Node:**  
    - POST final report metadata (title, URL, status, session_id) to an external endpoint or notification system.

33. **Connect all nodes as per described flow:**

- Webhook → Simple Memory → Strategy Agent → Structured Output Parser → Switch  
- Switch Feedback → Respond to Webhook  
- Switch Pass → Respond to Webhook1 → Code → OpenAI1 → Notion → Search Query Agent → Structured Output Parser1 → Split Out → Loop Over Queries → HTTP Request (search) → Edit Fields → OpenAI (URL selection) → HTTP Request1 (extract) → Aggregate → Report Agent → Get Existing Row → Convert to HTML → HTML to Array → Tags to Items → Notion Block Generator → Parse JSON blocks → Valid Blocks → For Each Block... → HTTP Request2 + Aggregate1 → Notion1 → HTTP Request3

**Credential Setup:**

- OpenRouter API key for OpenRouter Chat Model nodes.  
- OpenAI API key (or equivalent) for OpenAI nodes.  
- Tavily API key (generic HTTP credential) for search and extract API calls.  
- Notion integration token with database access.  
- Google Gemini API key for Notion Block Generator node.

---

## 5. General Notes & Resources

| Note Content                                                                                                                                                                                 | Context or Link                                                                                         |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| This workflow is triggered via webhook and can be integrated into chatbot or form systems to automate research report generation.                                                           | Overview section and Sticky Note at position [-1872,-48]                                              |
| The workflow uses multiple AI models: OpenRouter (Claude 3.5 Sonnet) for conversational agents, OpenAI GPT-4 variants for text generation, and Google Gemini for HTML-to-Notion block conversion. | Sticky Notes and node parameters for AI nodes                                                        |
| Tavily Search and Extract API are used for web content discovery and extraction.                                                                                                             | Sticky Note at position [1584,496]                                                                     |
| Notion API is utilized for creating and updating database pages and blocks.                                                                                                                 | Various Notion nodes with database ID and page update operations                                      |
| The workflow can handle up to 3 search queries per research topic, balancing depth and efficiency.                                                                                           | Strategy Agent and Search Query Agent prompt instructions                                             |
| Report includes markdown formatting with headings, lists, tables, and source references for transparency.                                                                                     | Report Agent prompt details                                                                           |
| Conversion from markdown to HTML and then to Notion blocks ensures rich content structure within Notion pages.                                                                              | Convert to HTML and Notion Block Generator nodes                                                      |
| The workflow uses random request IDs to track sessions and associate Notion pages and external reports.                                                                                      | Code node and usage across Notion and HTTP Request3 nodes                                            |
| For more information on Notion API structure and block types, see https://developers.notion.com/reference/block | Official Notion Developer Documentation                                                                |

---

**Disclaimer:**  
The text analyzed was exclusively extracted from an automated n8n workflow respecting all current content policies, containing no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.

---