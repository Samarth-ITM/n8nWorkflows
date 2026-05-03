🌐 Confluence Page AI Chatbot Workflow

https://n8nworkflows.xyz/workflows/---confluence-page-ai-chatbot-workflow-3012


# 🌐 Confluence Page AI Chatbot Workflow

### 1. Workflow Overview

The **🌐 Confluence Page AI Chatbot Workflow** is designed to enable conversational interaction with Confluence page content through an AI-powered chatbot. It integrates Confluence’s REST API to fetch page data, converts the content into a processable format, and uses an AI agent to answer user queries contextually.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures user chat messages as queries.
- **1.2 Global Configuration:** Sets global variables such as Confluence page IDs.
- **1.3 Confluence Page Retrieval:** Searches for a Confluence page by ID and fetches its detailed content in storage format.
- **1.4 Content Conversion:** Converts the retrieved HTML content of the page into Markdown.
- **1.5 AI Processing:** Uses an AI language model and conversational agent to process the Markdown content and user query, maintaining session memory.
- **1.6 Response Handling:** Formats and sends the AI-generated response back to the user via Telegram.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block triggers the workflow when a user sends a chat message. It initiates the process by capturing the user’s input and session information.

- **Nodes Involved:**  
  - When chat message received

- **Node Details:**  
  - **When chat message received**  
    - Type: `manualChatTrigger` (LangChain)  
    - Role: Entry point capturing chat input and session ID.  
    - Configuration: No parameters; listens for chat messages.  
    - Inputs: External chat message trigger.  
    - Outputs: Emits JSON containing `chatInput` and `sessionId`.  
    - Edge Cases: Failure if chat message format is invalid or missing session ID.  
    - Version: 1.1

#### 2.2 Global Configuration

- **Overview:**  
  Sets global variables for Confluence page IDs used later in the workflow.

- **Nodes Involved:**  
  - Globals

- **Node Details:**  
  - **Globals**  
    - Type: `Set`  
    - Role: Defines static page ID variables for different Confluence pages.  
    - Configuration: Assigns string values to variables like `page_id_tekla`, `page_id_n8n`, etc.  
    - Inputs: From "When chat message received" node.  
    - Outputs: JSON with page ID variables.  
    - Edge Cases: Hardcoded IDs require manual update if pages change.  
    - Version: 3.4

#### 2.3 Confluence Page Retrieval

- **Overview:**  
  Searches Confluence for a page by ID, extracts page metadata, then fetches the full page content in storage format.

- **Nodes Involved:**  
  - Search By ID  
  - Split Out  
  - Page Schema  
  - Confluence Page Storage View

- **Node Details:**  
  - **Search By ID**  
    - Type: `HTTP Request`  
    - Role: Queries Confluence REST API search endpoint to find page by ID.  
    - Configuration: URL uses page ID from Globals; authenticates with HTTP header auth (Basic Auth with API token).  
    - Inputs: From Globals node.  
    - Outputs: JSON search results.  
    - Edge Cases: API errors, invalid ID, authentication failure, rate limits.  
    - Version: 4.2

  - **Split Out**  
    - Type: `Split Out`  
    - Role: Extracts the first search result from the results array.  
    - Configuration: Splits on field `results`.  
    - Inputs: From Search By ID.  
    - Outputs: Single page object.  
    - Edge Cases: Empty results array leads to no output.  
    - Version: 1

  - **Page Schema**  
    - Type: `Set`  
    - Role: Normalizes and extracts key page metadata fields for downstream use.  
    - Configuration: Sets fields like `content._links.webui`, `title`, `excerpt`, `id`, etc. from the split search result.  
    - Inputs: From Split Out.  
    - Outputs: Structured page metadata JSON.  
    - Edge Cases: Missing fields in API response.  
    - Version: 3.4

  - **Confluence Page Storage View**  
    - Type: `HTTP Request`  
    - Role: Fetches full page content in `storage` format (HTML) using page ID.  
    - Configuration: URL built with page ID from Page Schema; authenticates with HTTP header auth. Query parameter `body-format=storage`.  
    - Inputs: From Page Schema.  
    - Outputs: JSON containing page body HTML.  
    - Edge Cases: API errors, invalid page ID, auth failure.  
    - Version: 4.2

#### 2.4 Content Conversion

- **Overview:**  
  Converts the HTML content of the Confluence page body into Markdown format for easier AI processing.

- **Nodes Involved:**  
  - HTML to Markdown

- **Node Details:**  
  - **HTML to Markdown**  
    - Type: `Markdown`  
    - Role: Transforms HTML content (`body.storage.value`) into Markdown text.  
    - Configuration: Input HTML expression set to `{{$json.body.storage.value}}`.  
    - Inputs: From Confluence Page Storage View.  
    - Outputs: Markdown text.  
    - Edge Cases: Malformed HTML may cause conversion issues.  
    - Version: 1

#### 2.5 AI Processing

- **Overview:**  
  Processes the user query and page content using an AI conversational agent with memory, generating context-aware answers.

- **Nodes Involved:**  
  - gpt-4o-mini  
  - Window Buffer Memory  
  - AI Agent

- **Node Details:**  
  - **gpt-4o-mini**  
    - Type: `lmChatOpenAi` (LangChain)  
    - Role: Language model node providing AI completions.  
    - Configuration: Uses OpenAI API credentials; no special options set.  
    - Inputs: From Window Buffer Memory (as AI language model).  
    - Outputs: AI-generated text.  
    - Edge Cases: API quota exceeded, network issues, invalid credentials.  
    - Version: 1

  - **Window Buffer Memory**  
    - Type: `memoryBufferWindow` (LangChain)  
    - Role: Maintains conversational context window per session.  
    - Configuration: Uses session key from `When chat message received` node’s `sessionId`.  
    - Inputs: From AI Agent (memory input).  
    - Outputs: Contextual memory for AI Agent.  
    - Edge Cases: Session ID missing or malformed; memory overflow.  
    - Version: 1.2

  - **AI Agent**  
    - Type: `agent` (LangChain)  
    - Role: Core conversational AI agent that answers user queries using provided context.  
    - Configuration:  
      - Prompt instructs to answer only using provided context or respond “I don’t know.”  
      - User question injected from `When chat message received` node’s `chatInput`.  
      - Context injected from previous node’s Markdown content (`$json.data`).  
      - Uses conversational agent type.  
    - Inputs: Receives Markdown content and memory context.  
    - Outputs: AI-generated answer.  
    - Edge Cases: Prompt formatting errors, API failures, incomplete context.  
    - Version: 1.6

#### 2.6 Response Handling

- **Overview:**  
  Prepares the AI-generated response and sends it back to the user via Telegram.

- **Nodes Involved:**  
  - Chat Response  
  - Send Telegram Message

- **Node Details:**  
  - **Chat Response**  
    - Type: `Set`  
    - Role: Extracts and assigns the AI agent’s output to a field named `output`.  
    - Configuration: Sets `output` to `{{$json.output}}`.  
    - Inputs: From AI Agent.  
    - Outputs: JSON with `output` string.  
    - Edge Cases: Missing AI output.  
    - Version: 3.4

  - **Send Telegram Message**  
    - Type: `Telegram`  
    - Role: Sends the chatbot’s response as a Telegram message to a configured chat ID.  
    - Configuration:  
      - Text set to `{{$json.output}}`.  
      - Chat ID from environment variable `TELEGRAM_CHAT_ID`.  
      - Parse mode: HTML.  
      - No attribution appended.  
      - Uses Telegram API credentials.  
    - Inputs: From Chat Response.  
    - Outputs: Telegram message sent confirmation.  
    - Edge Cases: Invalid chat ID, Telegram API errors, network issues.  
    - Version: 1.2

---

### 3. Summary Table

| Node Name                  | Node Type                      | Functional Role                          | Input Node(s)                 | Output Node(s)                       | Sticky Note                                                                                                          |
|----------------------------|--------------------------------|----------------------------------------|------------------------------|------------------------------------|----------------------------------------------------------------------------------------------------------------------|
| Sticky Note                | Sticky Note                    | Documentation on Confluence API basics | None                         | None                               | ## Confluence<br>https://developer.atlassian.com/cloud/confluence/basic-auth-for-rest-apis/ ... (full content above) |
| Sticky Note2               | Sticky Note                    | Documentation on Confluence Page API   | None                         | None                               | ## Using Rest API to GET Confluence Page Body<br>https://developer.atlassian.com/cloud/confluence/rest/v2/api-group-page/#api-pages-id-get |
| When chat message received | manualChatTrigger (LangChain) | Entry point for user chat input        | External chat trigger         | Globals                            |                                                                                                                      |
| Globals                   | Set                            | Sets global Confluence page IDs        | When chat message received    | Search By ID                      |                                                                                                                      |
| Search By ID              | HTTP Request                   | Searches Confluence page by ID          | Globals                      | Split Out                        |                                                                                                                      |
| Split Out                 | Split Out                     | Extracts first search result            | Search By ID                 | Page Schema                      |                                                                                                                      |
| Page Schema               | Set                            | Normalizes page metadata                 | Split Out                   | Confluence Page Storage View     |                                                                                                                      |
| Confluence Page Storage View | HTTP Request                 | Fetches full page content in storage format | Page Schema               | HTML to Markdown                 |                                                                                                                      |
| HTML to Markdown          | Markdown                      | Converts HTML page body to Markdown     | Confluence Page Storage View | AI Agent                        |                                                                                                                      |
| gpt-4o-mini               | lmChatOpenAi (LangChain)       | Language model for AI completions       | Window Buffer Memory         | AI Agent (as language model)     |                                                                                                                      |
| Window Buffer Memory      | memoryBufferWindow (LangChain) | Maintains conversational memory         | AI Agent                    | AI Agent (as memory)             |                                                                                                                      |
| AI Agent                  | agent (LangChain)              | Processes user query with context       | HTML to Markdown, gpt-4o-mini, Window Buffer Memory | Chat Response, Send Telegram Message |                                                                                                                      |
| Chat Response             | Set                            | Assigns AI output to `output` field     | AI Agent                    | Send Telegram Message            |                                                                                                                      |
| Send Telegram Message     | Telegram                      | Sends AI response to Telegram chat      | Chat Response               | None                           |                                                                                                                      |
| Sticky Note3              | Sticky Note                    | Chatbot for Confluence Pages note       | None                         | None                           | ## Chatbot for Confluence Pages                                                                                      |
| Sticky Note4              | Sticky Note                    | Confluence Search By ID note             | None                         | None                           | ## Confluence Search By ID                                                                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node**  
   - Add `manualChatTrigger` node named **When chat message received**.  
   - No special parameters. This node listens for chat messages and provides `chatInput` and `sessionId`.

2. **Create Globals Node**  
   - Add `Set` node named **Globals**.  
   - Define string variables for Confluence page IDs:  
     - `page_id_tekla` = "688157"  
     - `page_id_n8n` = "491546"  
     - `page_id_more_n8n` = "983041"  
     - `page_id_tekla_clash_checking` = "753691"  
   - Connect **When chat message received** → **Globals**.

3. **Create Search By ID Node**  
   - Add `HTTP Request` node named **Search By ID**.  
   - Set method to GET.  
   - URL: `https://example.atlassian.net/wiki/rest/api/search?limit=1&cql=id={{ $json.page_id_n8n }}` (replace domain and use dynamic page ID).  
   - Authentication: HTTP Header Auth with Confluence API credentials (Basic Auth with Base64 encoded email:API token).  
   - Connect **Globals** → **Search By ID**.

4. **Create Split Out Node**  
   - Add `Split Out` node named **Split Out**.  
   - Configure to split on field `results`.  
   - Connect **Search By ID** → **Split Out**.

5. **Create Page Schema Node**  
   - Add `Set` node named **Page Schema**.  
   - Map fields from split result:  
     - `content._links.webui` = `{{$json.content._links.webui}}`  
     - `content._links.self` = `{{$json.content._links.self}}`  
     - `title` = `{{$json.title}}`  
     - `url` = `{{$json.url}}`  
     - `excerpt` = `{{$json.excerpt}}`  
     - `id` = `{{$json.content.id}}`  
   - Connect **Split Out** → **Page Schema**.

6. **Create Confluence Page Storage View Node**  
   - Add `HTTP Request` node named **Confluence Page Storage View**.  
   - Method: GET  
   - URL: `https://example.atlassian.net/wiki/api/v2/pages/{{ $json.id }}` (replace domain).  
   - Query parameter: `body-format=storage`  
   - Authentication: HTTP Header Auth with Confluence API credentials.  
   - Connect **Page Schema** → **Confluence Page Storage View**.

7. **Create HTML to Markdown Node**  
   - Add `Markdown` node named **HTML to Markdown**.  
   - Input HTML: `{{$json.body.storage.value}}`  
   - Connect **Confluence Page Storage View** → **HTML to Markdown**.

8. **Create AI Agent Nodes**  
   - Add `lmChatOpenAi` node named **gpt-4o-mini**.  
     - Credentials: OpenAI API credentials.  
     - No special options.  
   - Add `memoryBufferWindow` node named **Window Buffer Memory**.  
     - Session key: `={{ $('When chat message received').item.json.sessionId }}`  
     - Session ID type: customKey  
   - Add `agent` node named **AI Agent**.  
     - Prompt:  
       ```
       Answer questions from user with the context provided. Only respond using the context. If you do not know the answer simply respond with "I don't know."

       User question: {{ $('When chat message received').item.json.chatInput }}

       Context: {{ $json.data }}
       ```  
     - Agent type: conversationalAgent  
     - Connect **gpt-4o-mini** → AI Agent (as language model input).  
     - Connect **Window Buffer Memory** → AI Agent (as memory input).  
     - Connect **HTML to Markdown** → AI Agent (main input).

9. **Create Chat Response Node**  
   - Add `Set` node named **Chat Response**.  
   - Assign field `output` = `{{$json.output}}` (AI Agent output).  
   - Connect **AI Agent** → **Chat Response**.

10. **Create Send Telegram Message Node**  
    - Add `Telegram` node named **Send Telegram Message**.  
    - Text: `{{$json.output}}`  
    - Chat ID: `={{ $env.TELEGRAM_CHAT_ID }}` (set environment variable with your Telegram chat ID).  
    - Parse mode: HTML  
    - Credentials: Telegram API credentials.  
    - Connect **Chat Response** → **Send Telegram Message**.

11. **Add Sticky Notes for Documentation** (optional but recommended)  
    - Add sticky notes with the provided Confluence API documentation and usage instructions near relevant nodes for clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Context or Link                                                                                              |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| Confluence API authentication requires generating an API token from Atlassian account portal and encoding credentials in Base64 for Basic Auth headers.                                                                                                                                                                                                                                                                                                                                     | https://developer.atlassian.com/cloud/confluence/basic-auth-for-rest-apis/                                   |
| Confluence REST API v2 documentation provides detailed info on endpoints, including page retrieval by ID and supported content formats.                                                                                                                                                                                                                                                                                                                                                       | https://developer.atlassian.com/cloud/confluence/rest/v2/intro/                                              |
| Example cURL commands for testing API authentication and requests are included in sticky notes for quick reference.                                                                                                                                                                                                                                                                                                                                                                           | See sticky notes in workflow                                                                                  |
| OpenAI API credentials must be configured in n8n for the AI language model nodes to function.                                                                                                                                                                                                                                                                                                                                                                                                 | https://platform.openai.com/docs/api-reference/authentication                                                 |
| Telegram Bot API credentials and environment variable `TELEGRAM_CHAT_ID` must be set for sending messages.                                                                                                                                                                                                                                                                                                                                                                                    | https://core.telegram.org/bots/api                                                                             |
| The workflow uses LangChain nodes for conversational AI and memory management, requiring n8n LangChain integration.                                                                                                                                                                                                                                                                                                                                                                           | https://docs.n8n.io/integrations/builtin/nodes/langchain/                                                     |

---

This documentation provides a detailed and structured reference for understanding, reproducing, and maintaining the Confluence Page AI Chatbot workflow in n8n. It covers all nodes, configurations, and integration points, enabling advanced users and automation agents to work with the workflow confidently.