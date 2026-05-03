Notion knowledge base AI assistant

https://n8nworkflows.xyz/workflows/notion-knowledge-base-ai-assistant-2413


# Notion knowledge base AI assistant

### 1. Workflow Overview

This workflow, titled **Notion Knowledge Base AI Assistant**, is designed to provide an AI-driven interface for querying and retrieving knowledge from a Notion database structured as a knowledge base. It targets users or teams managing extensive Notion data who want a fast, conversational AI assistant to search and summarize content effectively. Ideal for support teams, project managers, or knowledge workers, it allows querying both metadata (database properties like tags and questions) and the content inside individual Notion pages.

The workflow logic is organized into the following functional blocks:

- **1.1 Input Reception:** Captures chat messages from users as input triggers.
- **1.2 Database Metadata Retrieval:** Fetches Notion database details including schema and tags for dynamic filtering.
- **1.3 Input Formatting:** Prepares and structures input data and metadata for downstream processing.
- **1.4 AI Agent Orchestration:** Hosts the AI agent that controls the querying logic, decides when and how to query Notion, and manages responses.
- **1.5 Notion Database Search:** Queries the Notion database using flexible filter criteria (keyword or tags).
- **1.6 Notion Page Content Retrieval:** Retrieves detailed content blocks inside Notion pages for supplementary context.
- **1.7 AI Language Model:** Uses OpenAI GPT-4o to generate responses based on retrieved data and conversation context.
- **1.8 Memory Management:** Maintains a sliding window buffer of conversation context for the AI agent to improve continuity.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block listens for incoming chat messages from users and triggers the workflow. It acts as the entry point for user queries.

- **Nodes Involved:**  
  - When chat message received

- **Node Details:**  
  - **When chat message received:**  
    - Type: Chat Trigger node (Langchain)  
    - Configuration: Public webhook with a welcoming initial message ("Happy [weekday]! Knowledge source assistant at your service. How can I help you?")  
    - Input: External user chat message  
    - Output: JSON containing sessionId, user input text (`chatInput`), and action type  
    - Edge cases: Webhook downtime, malformed input  
    - No sub-workflows invoked  

#### 2.2 Database Metadata Retrieval

- **Overview:**  
  Retrieves metadata about the target Notion database, including its schema, tags, and ID. This information is vital for dynamic filtering and guiding AI behavior.

- **Nodes Involved:**  
  - Get database details

- **Node Details:**  
  - **Get database details:**  
    - Type: Native Notion node  
    - Configuration: Queries a specified Notion database by ID (`7ea9697d-4875-441e-b262-1105337d232e`)  
    - Credentials: Uses predefined Notion API credentials  
    - Output: Database schema details including properties, tags options, and database title  
    - Edge cases: Authorization issues if integration is not shared with database, network failures  
    - Notes: This node adds ~250-800ms latency per run; can be optimized out if desired (see sticky notes)  

#### 2.3 Input Formatting

- **Overview:**  
  Extracts and structures key information from the incoming chat message and database metadata to prepare a unified context for the AI agent.

- **Nodes Involved:**  
  - Format schema

- **Node Details:**  
  - **Format schema:**  
    - Type: Set node (data transformation)  
    - Configuration: Assigns variables including `sessionId`, `action`, `chatInput`, `notionID` (database ID), `databaseName` (title), and `tagsOptions` (list of available tags) from previous nodes  
    - Input: Output of "When chat message received" and "Get database details"  
    - Output: Structured JSON with all relevant metadata and input text for AI agent  
    - Edge cases: Missing fields if previous nodes fail or return incomplete data  

#### 2.4 AI Agent Orchestration

- **Overview:**  
  The core AI orchestrator node that receives user input, manages tools (search and retrieval), memory, and controls the overall response generation process according to defined system instructions.

- **Nodes Involved:**  
  - AI Agent

- **Node Details:**  
  - **AI Agent:**  
    - Type: Langchain Agent node  
    - Configuration:  
      - Input text from `chatInput`  
      - System message defines role, behavior, error handling, and output formatting, emphasizing factual, concise answers with URLs to Notion pages when applicable  
      - Uses two tools: "Search notion database" and "Search inside database record"  
      - Connected to memory node (Window Buffer Memory) and language model node (OpenAI Chat Model)  
    - Inputs: Formatted schema data, memory context, tools for querying Notion  
    - Outputs: Final AI-generated response  
    - Edge cases: No results found, ambiguous queries, API errors, timeout, hallucination risks mitigated by instructions  
    - Version-specific: Requires Langchain-compatible versions with agent and tool support  

#### 2.5 Notion Database Search

- **Overview:**  
  Provides an HTTP request tool to query the Notion database for matching records using flexible filters on properties like `question` and `tags`.

- **Nodes Involved:**  
  - Search notion database

- **Node Details:**  
  - **Search notion database:**  
    - Type: HTTP Request (Langchain tool)  
    - Configuration:  
      - POST request to Notion API endpoint `/v1/databases/{databaseId}/query`  
      - JSON body includes filters with OR logic on `question` (rich_text contains keyword) and `tags` (multi_select contains tag)  
      - Sorts results by `updated_at` ascending  
      - Credential: Notion API  
      - Placeholders for dynamic keywords and tags from AI Agent input  
    - Inputs: `notionID` from Format schema, search keyword/tag from AI Agent  
    - Outputs: Matching database records in JSON  
    - Edge cases: Authentication failure, rate limits, malformed queries, empty results  

#### 2.6 Notion Page Content Retrieval

- **Overview:**  
  Fetches detailed content blocks inside a specific Notion page using its page ID, enabling deeper content extraction for enhanced AI response context.

- **Nodes Involved:**  
  - Search inside database record

- **Node Details:**  
  - **Search inside database record:**  
    - Type: HTTP Request (Langchain tool)  
    - Configuration:  
      - GET request to Notion API `/v1/blocks/{page_id}/children`  
      - Requests specific fields like paragraph texts, headings, lists, and children blocks  
      - Optimized response mode enabled  
      - Credential: Notion API  
      - Placeholder for `page_id` dynamically set from database search results  
    - Input: Page ID from "Search notion database" results  
    - Output: Notion page content blocks  
    - Edge cases: Invalid page ID, rate limits, partial content loading  

#### 2.7 AI Language Model

- **Overview:**  
  This node runs the OpenAI GPT-4o model to generate natural language responses using input from the AI agent and context memory.

- **Nodes Involved:**  
  - OpenAI Chat Model

- **Node Details:**  
  - **OpenAI Chat Model:**  
    - Type: Langchain OpenAI Chat Model node  
    - Configuration:  
      - Model: GPT-4o  
      - Parameters: temperature 0.7, timeout 25000 ms  
      - Credentials: OpenAI API key  
    - Input: Text prompts from AI Agent with context from memory and search results  
    - Output: AI-generated chat response text  
    - Edge cases: API limits, timeouts, unexpected output formatting  

#### 2.8 Memory Management

- **Overview:**  
  Maintains a sliding window buffer of recent conversation turns to provide context continuity for the AI agent.

- **Nodes Involved:**  
  - Window Buffer Memory

- **Node Details:**  
  - **Window Buffer Memory:**  
    - Type: Langchain memoryBufferWindow node  
    - Configuration: Context window length set to 4 (last 4 interactions)  
    - Input: Conversation turns and AI outputs  
    - Output: Contextual memory to AI Agent  
    - Edge cases: Memory overflow, stale context if window too short or too long  

---

### 3. Summary Table

| Node Name                 | Node Type                                   | Functional Role                      | Input Node(s)                 | Output Node(s)            | Sticky Note                                                                                      |
|---------------------------|---------------------------------------------|------------------------------------|------------------------------|---------------------------|-------------------------------------------------------------------------------------------------|
| When chat message received | Langchain Chat Trigger                       | Entry point, receives user queries | (Webhook external input)      | Get database details       |                                                                                                 |
| Get database details       | Notion node                                 | Fetches Notion database metadata   | When chat message received    | Format schema              | FAQ: If "resource not found" error, share integration with database. Adds 250-800ms latency per run. |
| Format schema             | Set node                                    | Prepares input and metadata for AI | Get database details          | AI Agent                  |                                                                                                 |
| AI Agent                  | Langchain Agent                             | Controls query logic and response  | Format schema, OpenAI Chat Model, Window Buffer Memory, Search notion database, Search inside database record | -                         |                                                                                                 |
| Search notion database    | Langchain HTTP Request                      | Queries Notion database for records| AI Agent (tool input)         | AI Agent                  |                                                                                                 |
| Search inside database record | Langchain HTTP Request                   | Fetches content blocks from Notion pages | AI Agent (tool input)          | AI Agent                  |                                                                                                 |
| OpenAI Chat Model         | Langchain Language Model                    | Generates AI responses              | AI Agent                     | AI Agent                  |                                                                                                 |
| Window Buffer Memory      | Langchain Memory Buffer Window              | Maintains conversation context     | AI Agent                     | AI Agent                  |                                                                                                 |
| Sticky Note               | Sticky Note node                            | Documentation and instructions     |                              |                           | "Notion knowledge base assistant [v1]. Built as part of the 30 Day AI Sprint by @maxtkacz."      |
| Sticky Note1              | Sticky Note node                            | FAQ and troubleshooting notes      |                              |                           | "In Get database details, share connection with database. Step adds latency."                   |
| Sticky Note2              | Sticky Note node                            | Video quickstart link               |                              |                           | Quickstart video: https://www.youtube.com/watch?v=ynLZwS2Nhnc                                   |
| Sticky Note3              | Sticky Note node                            | Written setup instructions         |                              |                           | Detailed written steps for Notion and OpenAI credential setup and usage                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the input trigger node:**  
   - Add a **Langchain Chat Trigger** node named `When chat message received`.  
   - Configure as a public webhook with an initial greeting message:  
     `"Happy {{ $today.weekdayLong }}! Knowledge source assistant at your service. How can I help you?"`  
   - Save the webhook URL for testing and activation.

2. **Set up Notion credentials:**  
   - Create a Notion API credential in n8n following the [Notion integration guide](https://developers.notion.com/docs/create-a-notion-integration).  
   - Ensure the integration is shared with your Notion database.

3. **Add the Notion database metadata node:**  
   - Add a **Notion** node named `Get database details`.  
   - Set resource to `Database`.  
   - Select your Notion database from the dropdown or enter the database ID manually (`7ea9697d-4875-441e-b262-1105337d232e`).  
   - Attach the Notion API credential.  
   - Connect `When chat message received` → `Get database details`.

4. **Add a Set node for input formatting:**  
   - Add a **Set** node named `Format schema`.  
   - Assign variables:  
     - `sessionId` = `={{ $('When chat message received').item.json.sessionId }}`  
     - `action` = `={{ $('When chat message received').item.json.action }}`  
     - `chatInput` = `={{ $('When chat message received').item.json.chatInput }}`  
     - `notionID` = `={{ $('Get database details').item.json.id }}`  
     - `databaseName` = `={{ $json.title[0].text.content }}`  
     - `tagsOptions` = `={{ $json.properties.tags.multi_select.options.map(item => item.name).join(',') }}`  
   - Connect `Get database details` → `Format schema`.

5. **Add the AI Agent node:**  
   - Add a **Langchain Agent** node named `AI Agent`.  
   - Set `text` input as `={{ $json.chatInput }}`.  
   - Configure the system message with instructions on role, behavior, error handling, and output format, including mentioning the `databaseName` for context.  
   - Connect `Format schema` → `AI Agent` (main input).

6. **Add Notion database search tool:**  
   - Add a **Langchain HTTP Request** node named `Search notion database`.  
   - Set method to POST and URL to `https://api.notion.com/v1/databases/{{ $json.notionID }}/query`.  
   - Configure JSON body with filters: OR on `question` containing `{keyword}` or `tags` containing `{tag}`. Sort by `updated_at` ascending.  
   - Set credential to Notion API.  
   - Define placeholders `keyword` and `tag` based on AI Agent input.  
   - Connect `Search notion database` → `AI Agent` as a tool input.

7. **Add Notion page content retrieval tool:**  
   - Add a **Langchain HTTP Request** node named `Search inside database record`.  
   - Set method to GET and URL to `https://api.notion.com/v1/blocks/{page_id}/children`.  
   - Request fields: id, type, paragraph.text, headings, list items, to_do.text, children.  
   - Enable optimized response.  
   - Credential: Notion API.  
   - Placeholder: `page_id` from previous search results.  
   - Connect `Search inside database record` → `AI Agent` as a tool input.

8. **Add OpenAI language model node:**  
   - Add a **Langchain OpenAI Chat Model** node named `OpenAI Chat Model`.  
   - Choose model `gpt-4o`.  
   - Set temperature to 0.7 and timeout to 25000 ms.  
   - Attach OpenAI API credentials.  
   - Connect `OpenAI Chat Model` → `AI Agent` as language model input.

9. **Add memory management node:**  
   - Add a **Langchain Memory Buffer Window** node named `Window Buffer Memory`.  
   - Set context window length to 4.  
   - Connect `Window Buffer Memory` → `AI Agent` as memory input.

10. **Connect workflow outputs:**  
    - The AI Agent provides the final response output. Connect its main output to your desired end node or response handler depending on your integration (e.g., chat UI or webhook response).

11. **Add sticky notes for documentation:**  
    - Add sticky notes with:  
      - Project credits and version info  
      - FAQ and troubleshooting hints (e.g., database sharing issues)  
      - Quickstart video link: https://www.youtube.com/watch?v=ynLZwS2Nhnc  
      - Written setup instructions with steps for credentials and database sharing  

12. **Test and activate:**  
    - Test by sending chat messages to the webhook URL.  
    - Activate the workflow after successful tests.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                 |
|-----------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------|
| Notion knowledge base assistant [v1] built as part of the 30 Day AI Sprint by @maxtkacz                    | https://30dayaisprint.notion.site/                                                            |
| FAQ: If "The resource you are requesting could not be found" error appears in `Get database details`, share integration with the database in Notion | Important troubleshooting note to avoid authorization errors                                  |
| Quickstart video for setup and usage                                                                      | https://www.youtube.com/watch?v=ynLZwS2Nhnc                                                   |
| Written setup instructions: Notion credential creation, database duplication and sharing, OpenAI credential setup, and activation | Included in Sticky Note3 in the workflow                                                      |
| The Notion template used by this assistant                                                                | https://www.notion.so/templates/knowledge-base-ai-assistant-with-n8n                         |

---

This detailed, structured documentation offers a complete understanding of the **Notion knowledge base AI assistant** workflow, its components, and setup instructions to facilitate reproduction, troubleshooting, and modification by advanced users or automation agents.