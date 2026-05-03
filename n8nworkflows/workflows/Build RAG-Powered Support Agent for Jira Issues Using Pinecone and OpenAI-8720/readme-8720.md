Build RAG-Powered Support Agent for Jira Issues Using Pinecone and OpenAI

https://n8nworkflows.xyz/workflows/build-rag-powered-support-agent-for-jira-issues-using-pinecone-and-openai-8720


# Build RAG-Powered Support Agent for Jira Issues Using Pinecone and OpenAI

### 1. Workflow Overview

This workflow is designed to build a Retrieval-Augmented Generation (RAG) powered support agent for Jira issues, leveraging Pinecone vector database and OpenAI models. It automates the extraction of unresolved Jira tickets, merges and cleans associated comments, generates semantic embeddings, and indexes them in Pinecone for efficient semantic search. The indexed data is then exposed as an MCP tool for AI-driven querying by a chatbot agent that provides technical support insights, including SLA (Service Level Agreement) details.

**Target Use Cases:**  
- Automate continuous ingestion and indexing of Jira open issues with comments.  
- Enable semantic search and AI-assisted querying about ticket status, severity, and SLA for clients.  
- Provide a conversational AI interface for commercial or support teams to understand issue health quickly.

**Logical Blocks:**  
- **1.1 Scheduled Extraction & Pagination:** Trigger that periodically fetches Jira issues in batches with pagination until all unresolved tickets are retrieved.  
- **1.2 Data Transformation & Comment Aggregation:** Extract relevant fields from issues, fetch and merge user comments with filtering and cleaning.  
- **1.3 Text Cleaning & Chunking:** Convert HTML content to plain text and split large texts into manageable chunks for embedding.  
- **1.4 Embedding Generation & Vector Storage:** Generate vector embeddings using OpenAI and insert into Pinecone vector store with metadata. The index is cleared on each run to reflect current unresolved issues.  
- **1.5 MCP Exposure:** Publish the Pinecone index as an MCP tool enabling external semantic queries.  
- **1.6 AI Chatbot Agent:** A conversational interface using OpenAI chat models, augmented with the vector store search and SLA knowledge, to answer user queries about open tickets.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Extraction & Pagination

**Overview:**  
This block triggers the workflow on a defined schedule, queries Jira REST API for unresolved issues in batches of 25, and paginates until all issues are loaded.

**Nodes Involved:**  
- Schedule Trigger  
- Cycles (Merge)  
- Extract Issues (HTTP Request)  
- All openissues are loaded? (Switch)  

**Node Details:**  

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Starts the workflow at 8, 11, 14, and 17 on weekdays (Mon-Fri).  
  - Configuration: Cron expression `0 8,11,14,17 * * 1-5`.  
  - Inputs: None (trigger node).  
  - Outputs: To Cycles node.  
  - Failure Modes: Misconfigured cron, scheduling service downtime.  

- **Cycles (Merge)**  
  - Type: Merge  
  - Role: Combines paginated issue batches into a single stream.  
  - Inputs: From Schedule Trigger and from pagination loop.  
  - Outputs: To Extract Issues node.  
  - Failure Modes: Merge conflicts if inputs malformed.  

- **Extract Issues**  
  - Type: HTTP Request  
  - Role: Fetches Jira issues using Jira REST API with JQL for unresolved cases.  
  - Configuration:
    - URL: `https://jira.siav.it/rest/api/2/search`  
    - Query Parameters: maxResults=25, jql=project=CS AND issuetype=Case AND resolution=Unresolved AND created >= -365d, startAt dynamically set by runIndex for pagination.  
    - Authentication: Jira PAT OAuth2 credential (configured externally).  
    - Pagination logic: Uses runIndex to fetch batches of 25 issues until total count exceeded.  
  - Inputs: From Cycles node.  
  - Outputs: To Extract Relevant Info.  
  - Failure Modes: HTTP errors, auth failure, API rate limits, malformed JQL, network issues.  

- **All openissues are loaded? (Switch)**  
  - Type: Switch  
  - Role: Evaluates if the next pagination batch exceeds total issues to decide whether to loop or exit.  
  - Conditions: Checks if `(runIndex+1)*25 > total` to exit or continue cycling.  
  - Inputs: From Pinecone Vector Store node (after insert) to continue or break loop.  
  - Outputs: To Cycles (for next pagination) or exit (end loop).  
  - Failure Modes: Expression evaluation errors, incorrect pagination logic.  

---

#### 1.2 Data Transformation & Comment Aggregation

**Overview:**  
Transforms raw Jira issue data extracting key fields, fetches comments per issue, merges and filters comments, preparing data for indexing.

**Nodes Involved:**  
- Extract Relevant Info (Code)  
- Get Comments (HTTP Request)  
- Create Comment array (Code)  
- Merge Comments (Merge)  

**Node Details:**  

- **Extract Relevant Info**  
  - Type: Code (JavaScript)  
  - Role: Extracts structured fields such as issue key, ID, summary, description, product, level, customer, status, classification, and registration date formatted for readability.  
  - Key Expressions: Date formatting, field normalization, trimming.  
  - Inputs: From Extract Issues (Jira API response).  
  - Outputs: To Merge Comments and Get Comments nodes.  
  - Failure Modes: Null or missing fields, date parsing errors.  

- **Get Comments**  
  - Type: HTTP Request  
  - Role: For each issue key, fetches associated comments from Jira REST API endpoint `/issue/{{issue_key}}/comment`.  
  - Configuration: Basic HTTP Auth credential, allow unauthorized certs true.  
  - Inputs: From Extract Relevant Info (issue keys).  
  - Outputs: To Create Comment array.  
  - Failure Modes: HTTP errors, auth failures, missing comments field, rate limits.  

- **Create Comment array**  
  - Type: Code (JavaScript)  
  - Role: Aggregates comments by issue, filters out non-informative comments (images only, single dot, empty markdown), concatenates comments into a single string.  
  - Key Expressions: Regex filters for comment content.  
  - Inputs: From Get Comments.  
  - Outputs: To Merge Comments.  
  - Failure Modes: Unexpected comment formats, empty comments, regex errors.  

- **Merge Comments**  
  - Type: Merge  
  - Role: Joins extracted issue info and aggregated comments on issue_id key, combining relevant data streams.  
  - Inputs: From Extract Relevant Info and Create Comment array.  
  - Outputs: To Convert to txt.  
  - Failure Modes: Key mismatch, merge conflicts.  

---

#### 1.3 Text Cleaning & Chunking

**Overview:**  
Cleans merged HTML comment and issue text to plain text, splits large documents into chunks for efficient embedding.

**Nodes Involved:**  
- Convert to txt (Code)  
- Document Chunker  

**Node Details:**  

- **Convert to txt**  
  - Type: Code (JavaScript)  
  - Role: Cleans HTML content by removing tags, replacing HTML entities, removing images and panel macros, compressing whitespace.  
  - Key Expressions: Regex replacements for HTML tags, entities, and unwanted elements.  
  - Inputs: From Merge Comments.  
  - Outputs: To Pinecone Vector Store node.  
  - Failure Modes: Malformed HTML, unexpected markup, regex issues.  

- **Document Chunker**  
  - Type: Recursive Character Text Splitter (Langchain)  
  - Role: Splits cleaned text into chunks of 512 characters with 50 characters overlap to preserve context between chunks.  
  - Parameters: chunkSize=512, chunkOverlap=50, splitCode=markdown.  
  - Inputs: From Convert to txt (embedded in ai_textSplitter connection).  
  - Outputs: To openIssues (Data Loader).  
  - Failure Modes: Text too short or empty, splitting errors.  

---

#### 1.4 Embedding Generation & Vector Storage

**Overview:**  
Generates vector embeddings from text chunks via OpenAI embeddings and inserts them into Pinecone vector index with metadata, clearing namespace at first run to keep data current.

**Nodes Involved:**  
- Embeddings OpenAI  
- Pinecone Vector Store  
- openIssues (Data Loader)  

**Node Details:**  

- **Embeddings OpenAI**  
  - Type: OpenAI Embeddings (Langchain)  
  - Role: Generates 512-dimensional vector embeddings of text chunks.  
  - Parameters: dimensions=512.  
  - Inputs: Text chunks from Document Chunker.  
  - Outputs: To Pinecone Vector Store.  
  - Failure Modes: API quota exceeded, network errors, malformed input data.  

- **Pinecone Vector Store**  
  - Type: Vector Store Pinecone (Langchain)  
  - Role: Inserts vectors plus metadata into Pinecone index named `openissues` under namespace `jira`. Namespace is cleared on first batch to ensure data freshness.  
  - Parameters: clearNamespace on first run only (`{{$runIndex == 0}}`).  
  - Credentials: Pinecone API account configured externally.  
  - Inputs: From Embeddings OpenAI and openIssues (Data Loader).  
  - Outputs: To Switch node (All openissues are loaded?).  
  - Failure Modes: Pinecone API errors, auth failure, network issues, namespace conflicts.  

- **openIssues (Data Loader)**  
  - Type: Document Default Data Loader (Langchain)  
  - Role: Prepares documents with metadata fields (issue_key, customer, classification, etc.) combined with text for embedding ingestion.  
  - Parameters: Metadata mapped from issue fields, JSON mode expression for input text.  
  - Inputs: From Document Chunker.  
  - Outputs: To Pinecone Vector Store.  
  - Failure Modes: Missing metadata, malformed JSON, mapping errors.  

---

#### 1.5 MCP Exposure

**Overview:**  
Exposes the Pinecone vector index as an MCP tool for semantic queries from external clients or agents.

**Nodes Involved:**  
- MCP Server Trigger  
- openissues (Vector Store Pinecone node in retrieve-as-tool mode)  
- MCP RAG (MCP Client Tool)  

**Node Details:**  

- **MCP Server Trigger**  
  - Type: MCP Trigger (Langchain)  
  - Role: Exposes a webhook endpoint `/jiraticket` allowing external AI agents to query the Pinecone index via MCP protocol.  
  - Inputs: External calls.  
  - Outputs: To openissues vector store retrieve node.  
  - Failure Modes: Webhook connectivity, security risks if exposed publicly.  

- **openissues**  
  - Type: Vector Store Pinecone (Langchain)  
  - Role: Retrieves vectors from Pinecone index for queries received via MCP.  
  - Parameters: Mode `retrieve-as-tool`, namespace `jira`, index `openissues`, with tool description for documentation.  
  - Inputs: From MCP Server Trigger (ai_tool).  
  - Outputs: To MCP RAG or AI Agent.  
  - Failure Modes: Pinecone retrieval errors, auth failures.  

- **MCP RAG**  
  - Type: MCP Client Tool (Langchain)  
  - Role: Calls the MCP server endpoint to query the indexed Jira tickets with semantic search, integrating with chatbot agent.  
  - Parameters: Endpoint URL set to local MCP server.  
  - Inputs: From AI Agent or chat flow.  
  - Outputs: Returns retrieved info for AI response generation.  
  - Failure Modes: Network errors, endpoint unavailability, MCP protocol errors.  

---

#### 1.6 AI Chatbot Agent

**Overview:**  
Conversational interface that accepts user queries, interprets intent, queries Pinecone index via embedded tools, and responds with detailed issue and SLA information.

**Nodes Involved:**  
- Chat (Chat Trigger)  
- SLA (Set node)  
- AI Agent (Langchain Agent)  
- OpenAI Chat Model  
- Simple Memory (Memory Buffer Window)  
- openIssues (Vector Store Pinecone retrieve-as-tool)  

**Node Details:**  

- **Chat**  
  - Type: Chat Trigger (Langchain)  
  - Role: Public chat interface for commercial/support team to ask questions about Jira tickets.  
  - Configuration: Custom CSS for dark theme, initial message “Come posso aiutarti oggi? 😎”, public enabled.  
  - Inputs: User messages via webhook.  
  - Outputs: To SLA node.  
  - Failure Modes: Webhook errors, UI issues.  

- **SLA**  
  - Type: Set  
  - Role: Provides SLA rules and descriptions as context for AI agent to include in responses.  
  - Configuration: Large multiline string with detailed SLA explanation assigned to variable `SLA`.  
  - Inputs: From Chat.  
  - Outputs: To AI Agent.  
  - Failure Modes: None significant; large text handling.  

- **AI Agent**  
  - Type: Langchain Agent (OpenAI)  
  - Role: Core AI that accepts user query text, uses system prompt defining role as technical support agent, calls embedded tools (`openIssues` vector store or MCP tool) to fetch relevant tickets, and generates human-friendly responses including SLA info.  
  - Configuration:
    - System message with instructions, expected response structure, SLA integration, and tone.  
    - Uses OpenAI Chat model `gpt-4o`.  
    - Uses tool `openIssues` or `openissues` (MCP) for semantic search.  
    - Uses Simple Memory buffer to maintain short-term conversation context (10 messages window).  
  - Inputs: From SLA (chat input text injected).  
  - Outputs: Chatbot responses to user.  
  - Failure Modes: API rate limiting, incorrect tool usage, memory overflow, incomplete retrievals.  

- **OpenAI Chat Model**  
  - Type: LM Chat OpenAI  
  - Role: Underlying LLM executing chat completions for AI Agent.  
  - Configuration: Model `gpt-4o`.  
  - Inputs: From AI Agent.  
  - Outputs: To AI Agent (response).  
  - Failure Modes: API failures, quota exceeded, network errors.  

- **Simple Memory**  
  - Type: Memory Buffer Window (Langchain)  
  - Role: Maintains conversational context by storing last 10 messages for AI Agent.  
  - Inputs: From AI Agent.  
  - Outputs: To AI Agent.  
  - Failure Modes: Memory overflow, context loss if misconfigured.  

- **openIssues (Vector Store Pinecone retrieve-as-tool)**  
  - Type: Vector Store Pinecone  
  - Role: Retrieves top 10 relevant tickets from Pinecone index filtered by metadata such as client.  
  - Inputs: From AI Agent (as AI tool).  
  - Outputs: To AI Agent (embedding search results).  
  - Failure Modes: Pinecone query errors, empty results, auth failure.  

---

### 3. Summary Table

| Node Name                | Node Type                                | Functional Role                                   | Input Node(s)               | Output Node(s)                | Sticky Note                                                                                  |
|--------------------------|----------------------------------------|--------------------------------------------------|-----------------------------|------------------------------|----------------------------------------------------------------------------------------------|
| Schedule Trigger         | Schedule Trigger                       | Periodic trigger to start issues extraction       | -                           | Cycles                       | See Sticky Note: Workflow overview at start                                                |
| Cycles                  | Merge                                 | Combines paged issue batches                      | Schedule Trigger, Switch     | Extract Issues               |                                                                                              |
| Extract Issues           | HTTP Request                         | Fetch Jira issues with pagination                  | Cycles                      | Extract Relevant Info        |                                                                                              |
| Extract Relevant Info    | Code                                 | Extracts key fields from Jira issues               | Extract Issues              | Merge Comments, Get Comments |                                                                                              |
| Get Comments             | HTTP Request                         | Fetch comments for each issue                       | Extract Relevant Info       | Create Comment array         |                                                                                              |
| Create Comment array     | Code                                 | Filters and aggregates comments                     | Get Comments                | Merge Comments              |                                                                                              |
| Merge Comments           | Merge                                 | Merges issue info with aggregated comments         | Extract Relevant Info, Create Comment array | Convert to txt              |                                                                                              |
| Convert to txt           | Code                                 | Cleans HTML to plain text                           | Merge Comments              | Pinecone Vector Store        |                                                                                              |
| Document Chunker         | Recursive Character Text Splitter    | Splits text into chunks for embedding              | Convert to txt              | openIssues (Data Loader)     |                                                                                              |
| openIssues (Data Loader) | Document Default Data Loader          | Prepares documents with metadata for embedding     | Document Chunker            | Pinecone Vector Store        |                                                                                              |
| Embeddings OpenAI        | Embeddings OpenAI                    | Generates vector embeddings                         | Document Chunker            | Pinecone Vector Store        |                                                                                              |
| Pinecone Vector Store    | Vector Store Pinecone                | Inserts vectors + metadata into Pinecone index     | Embeddings OpenAI, openIssues (Data Loader) | All openissues are loaded?  | Sticky Note3: Pinecone Vector Store - loads paged open issues into index                    |
| All openissues are loaded?| Switch                              | Checks if all issues loaded to control pagination | Pinecone Vector Store       | Cycles (continue) or exit    |                                                                                              |
| MCP Server Trigger       | MCP Trigger (Langchain)              | Exposes Pinecone index as MCP tool                  | External webhook calls      | openissues                  | Sticky Note3: Published also as MCP tool                                                    |
| openissues               | Vector Store Pinecone (retrieve-as-tool) | Retrieves vectors from Pinecone for MCP queries   | MCP Server Trigger          | MCP RAG                      | Sticky Note5: Tool for external semantic queries                                            |
| MCP RAG                  | MCP Client Tool                      | Client tool querying MCP server                      | openissues                  | AI Agent                    | Sticky Note5: Substitute openissue tool with RAG MCP Tool for MCP server connection         |
| Chat                     | Chat Trigger                        | Starts AI conversation with support/commercial team| External user input          | SLA                         | Sticky Note6: AI Chatbot for Jira open tickets with SLA insights, flow structure explained  |
| SLA                      | Set                                 | Provides SLA rules and descriptions                 | Chat                        | AI Agent                    |                                                                                              |
| AI Agent                 | Langchain Agent                     | AI agent for interpreting queries and generating responses | SLA, openIssues tool       | OpenAI Chat Model            | Sticky Note6: AI Chatbot explanation                                                       |
| OpenAI Chat Model        | LM Chat OpenAI                     | Executes chat completions                            | AI Agent                    | AI Agent                    |                                                                                              |
| Simple Memory            | Memory Buffer Window                | Maintains short-term conversation context          | AI Agent                    | AI Agent                    |                                                                                              |
| openIssues               | Vector Store Pinecone (retrieve-as-tool) | Retrieves top relevant Jira tickets                | AI Agent                    | AI Agent                    | Sticky Note6: Key data stored: issue key, severity, SLA, etc.                              |
| Convert to txt           | Code                                | Cleans and prepares text for embedding              | Merge Comments              | Pinecone Vector Store       |                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**  
   - Set cron expression to `0 8,11,14,17 * * 1-5` (8 AM, 11 AM, 2 PM, 5 PM weekdays).  
   - Connect output to a Merge node named `Cycles`.

2. **Create a Merge node (`Cycles`):**  
   - Mode: default (merge inputs).  
   - Connect input 1 from Schedule Trigger; input 2 from `All openissues are loaded?` node’s cycle output.  
   - Output to HTTP Request node `Extract Issues`.

3. **Create HTTP Request node `Extract Issues`:**  
   - URL: `https://jira.siav.it/rest/api/2/search`.  
   - Method: GET.  
   - Query parameters: maxResults=25, jql=`project = CS AND issuetype = Case AND resolution = Unresolved AND created >= -365d`, startAt=`={{$runIndex*25}}`.  
   - Authentication: Jira PAT OAuth2 credentials configured.  
   - Allow unauthorized certs: true.  
   - Output connected to `Extract Relevant Info`.

4. **Create Code node `Extract Relevant Info`:**  
   - JavaScript code to map Jira JSON issues to simplified structure extracting issue_key, id, summary, description, product, level, customer, status, classification, and registration date formatted as Italian locale string.  
   - Connect output to two nodes: `Get Comments` and `Merge Comments`.

5. **Create HTTP Request node `Get Comments`:**  
   - URL: `https://jira.siav.it/rest/api/2/issue/{{ $json.issue_key }}/comment`.  
   - Method: GET.  
   - Authentication: Basic HTTP Auth configured for Jira.  
   - Allow unauthorized certs: true.  
   - Output to Code node `Create Comment array`.

6. **Create Code node `Create Comment array`:**  
   - JavaScript code that aggregates comments per issue, filtering out image-only comments, single dot comments, and empty markdown. Joins comments into a single string per issue.  
   - Output to Merge node `Merge Comments`.

7. **Create Merge node `Merge Comments`:**  
   - Mode: Combine, match on `issue_id`.  
   - Inputs: from `Extract Relevant Info` and `Create Comment array`.  
   - Output to Code node `Convert to txt`.

8. **Create Code node `Convert to txt`:**  
   - JavaScript code to strip HTML tags, replace HTML entities with characters, remove images, panels, and compress whitespace in the merged text content.  
   - Output to Pinecone Vector Store node.

9. **Create Recursive Character Text Splitter node `Document Chunker`:**  
   - Chunk size: 512 characters.  
   - Chunk overlap: 50 characters.  
   - Split code: markdown.  
   - Input: connect from `Convert to txt` node (ai_textSplitter connection).  
   - Output to Document Default Data Loader `openIssues (Data Loader)`.

10. **Create Document Default Data Loader node `openIssues (Data Loader)`:**  
    - Map metadata fields: ticket = issue_key, issue_id, customer, product, classification, registration, state = status, applicationmanagementLevel = level.  
    - JSON data: expression combining Customer, Summary, and Description text fields.  
    - Connect output to Embeddings OpenAI node.

11. **Create Embeddings OpenAI node:**  
    - Dimensions: 512.  
    - Connect output to Pinecone Vector Store node.

12. **Create Pinecone Vector Store node:**  
    - Mode: Insert.  
    - Clear namespace: set expression to clear on first run only (`= {{ $runIndex === 0 }}`).  
    - Namespace: `jira`.  
    - Index: `openissues`.  
    - Credentials: Pinecone API configured.  
    - Output to Switch node `All openissues are loaded?`.

13. **Create Switch node `All openissues are loaded?`:**  
    - Condition 1 (Exit): `( ($runIndex+1)*25 > $('Extract Issues').item.json.total )` evaluates true.  
    - Condition 2 (Cycle): false condition of above.  
    - On Exit: end loop.  
    - On cycle: connect back to `Cycles` node to continue pagination.

14. **Create MCP Server Trigger node:**  
    - Path: `/jiraticket`.  
    - Used to expose Pinecone index as MCP tool.  
    - Output to Pinecone Vector Store node `openissues`.

15. **Create Pinecone Vector Store node `openissues`:**  
    - Mode: Retrieve-as-tool.  
    - Namespace: `jira`.  
    - Index: `openissues`.  
    - Tool description: "Recupera informazioni sui ticket aperti per i clienti di Siav..."  
    - Credentials: Pinecone API.  
    - Input from MCP Server Trigger.

16. **Create MCP Client Tool node `MCP RAG`:**  
    - Endpoint URL: `http://localhost:5678/mcp-test/jiraticket`.  
    - Server transport: httpStreamable.  
    - Input from AI Agent.  
    - Used optionally to substitute direct Pinecone retrieval tool.

17. **Create Chat Trigger node `Chat`:**  
    - Public enabled.  
    - Custom CSS for dark theme.  
    - Initial message: "Come posso aiutarti oggi? 😎".  
    - Output to Set node `SLA`.

18. **Create Set node `SLA`:**  
    - Assign a large SLA description string to variable `SLA`.  
    - Output to AI Agent.

19. **Create Langchain Agent node `AI Agent`:**  
    - Text input: `={{ $json.chatInput }}`.  
    - System message: detailed instructions defining expert technical support role, usage of tools for semantic search, expected response format including SLA details.  
    - Tools: link to `openIssues` vector store tool or MCP tool node.  
    - Memory: connect to `Simple Memory`.  
    - Language model: connect to OpenAI Chat Model node.  
    - Output: chatbot responses.

20. **Create OpenAI Chat Model node:**  
    - Model: `gpt-4o`.  
    - Output connected to AI Agent.

21. **Create Simple Memory node:**  
    - Context window length: 10 messages.  
    - Connect input/output to AI Agent.

22. **Create Vector Store Pinecone node `openIssues`:**  
    - Mode: Retrieve-as-tool.  
    - TopK: 10.  
    - Namespace: `jira`.  
    - Index: `openissues`.  
    - Tool name: `openIssue`.  
    - Tool description: "Retrieve information on open tickets for Siav's clients..."  
    - Credentials: Pinecone API.  
    - Connect as AI tool to AI Agent.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Context or Link                                                |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| The workflow is published also as an MCP tool, enabling external semantic queries to the Jira open issues index.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Sticky Note3                                                   |
| The Pinecone vector store node clears the namespace at each full run to keep the index synchronized with the current unresolved Jira tickets. This ensures no stale tickets are kept indexed.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Sticky Note1, Sticky Note3                                     |
| The AI agent uses a detailed system prompt instructing it to treat synonyms (issue, ticket, problem, incident) equivalently and always extract and report all relevant tickets found. It integrates SLA rules to provide comprehensive support insights.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Sticky Note6                                                   |
| OpenAI GPT-4o model is used for chat completions providing advanced language understanding capabilities.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Node configuration detail                                      |
| The Jira API credentials must be configured with necessary permissions to access issues and comments via REST API.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Setup instructions                                             |
| Pinecone index `openissues` must be created with 512 dimensions matching OpenAI embedding output dimensions.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Setup instructions                                             |
| The workflow uses a combination of Langchain nodes and custom code nodes for data transformation, semantic embeddings, and chatbot integration.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Overall architecture                                           |
| You can customize the JQL query in the `Extract Issues` node to filter tickets differently, adjust pagination size, or modify metadata handled in the transformation nodes to suit your Jira schema.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Sticky Note1                                                   |
| Enabling the MCP nodes allows you to substitute direct Pinecone queries with MCP server-client architecture to scale semantic search externally.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Sticky Note5                                                   |
| The chat interface is styled with a dark theme using custom CSS for an improved user experience.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Chat node parameters                                           |
| Project dependencies: n8n instance with Jira, Pinecone, and OpenAI credentials properly configured; Pinecone index created; OpenAI API key available.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Setup instructions                                             |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled are legal and publicly accessible.