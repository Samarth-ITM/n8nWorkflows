Dynamic MCP Server Selection with OpenAI GPT-4.1 and Contextual AI Reranker

https://n8nworkflows.xyz/workflows/dynamic-mcp-server-selection-with-openai-gpt-4-1-and-contextual-ai-reranker-8272


# Dynamic MCP Server Selection with OpenAI GPT-4.1 and Contextual AI Reranker

---

# Dynamic MCP Server Selection with OpenAI GPT-4.1 and Contextual AI Reranker  
### Comprehensive Workflow Reference Document

---

### 1. Workflow Overview

This workflow dynamically selects and ranks Model Context Protocol (MCP) servers based on a user query. It is designed to assist large language models (LLMs) in choosing the most appropriate external MCP servers from a large, frequently updated pool of over 5000 servers. The workflow addresses the challenge of manual, static server selection by automating decision-making and ranking through integration with OpenAI’s GPT-4.1 mini model and Contextual AI’s reranker API.

**Use Cases:**  
- Enhancing LLM responses by dynamically incorporating external MCP servers tailored to query needs  
- Automating server selection in environments with thousands of potential MCPs  
- Providing real-time server ranking based on user input and server metadata  

**Logical Blocks:**  
- **1.1 Input Reception:** Captures the user query via a chat trigger.  
- **1.2 Decision-Making with LLM Agent:** Determines whether MCP servers are necessary and generates instructions for reranking.  
- **1.3 MCP Server Fetching & Formatting:** Retrieves MCP server data from PulseMCP API and formats it as documents for ranking.  
- **1.4 Contextual AI Reranking:** Sends query, instructions, and formatted servers to the reranker to score and rank servers.  
- **1.5 Result Formatting & Output:** Processes reranker output, selects the top 5 servers, and returns a user-friendly response.  

---

### 2. Block-by-Block Analysis

---

#### 1.1 Input Reception  
- **Overview:** Captures the initial user query through a chat trigger to start the workflow.  
- **Nodes Involved:** User-Query  

**Node Details:**  
- **User-Query**  
  - Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
  - Role: Initiates workflow upon user chat input.  
  - Configuration: Public webhook enabled, supports file uploads, initial message “Try MCP Reranker using Contextual AI's Reranker v2.”  
  - Inputs: External user input via webhook  
  - Outputs: Passes user input JSON (`chatInput`) downstream  
  - Edge cases: Webhook availability, malformed input  
  - No sub-workflow  

---

#### 1.2 Decision-Making with LLM Agent  
- **Overview:** Uses GPT-4.1 mini via Langchain agent to analyze the user query and decide if MCP servers are required, generating reranking instructions if so.  
- **Nodes Involved:** OpenAI Chat Model, LLM Agent for Decision-Making, If  

**Node Details:**  
- **OpenAI Chat Model**  
  - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  - Role: Provides GPT-4.1-mini LLM responses to support the agent node  
  - Configuration: Model = GPT-4.1-mini, response format = JSON object  
  - Inputs: Passes LLM responses to the agent node  
  - Edge cases: API key/auth errors, rate limits, timeout, incorrect model name  
- **LLM Agent for Decision-Making**  
  - Type: `@n8n/n8n-nodes-langchain.agent`  
  - Role: Processes user query to determine MCP necessity and generates JSON output with keys: `use_mcp` (boolean), `reason` (string), and `instruction` (string or null)  
  - Configuration: Custom system prompt instructing detailed decision logic and JSON output format  
  - Inputs: Receives user query from User-Query node; uses OpenAI Chat Model as LLM  
  - Outputs: JSON output passed to If node  
  - Edge cases: Expression evaluation errors, invalid JSON output, incomplete LLM responses  
- **If**  
  - Type: `n8n-nodes-base.if`  
  - Role: Routes workflow based on whether MCP servers are required (`use_mcp` true or false)  
  - Configuration: Checks boolean expression `{{$json.output.parseJson().use_mcp}}` equals true  
  - Inputs: Output from LLM Agent for Decision-Making  
  - Outputs:  
    - True path: proceeds to fetch MCP servers  
    - False path: sends final response stating no MCP required  
  - Edge cases: Parsing errors if JSON malformed or `use_mcp` undefined  

---

#### 1.3 MCP Server Fetching & Formatting  
- **Overview:** Fetches up to 5000 MCP servers from PulseMCP API and formats them into documents with metadata for reranking.  
- **Nodes Involved:** PulseMCP Fetch MCP Servers, Merge, Parse MCP Server list into documents w metadata  

**Node Details:**  
- **PulseMCP Fetch MCP Servers**  
  - Type: `n8n-nodes-base.httpRequest`  
  - Role: HTTP GET request to PulseMCP API to retrieve server list  
  - Configuration:  
    - URL: `https://api.pulsemcp.com/v0beta/servers`  
    - Query parameters: `count_per_page=5000`, `offset=0`  
  - Inputs: True branch from If node  
  - Outputs: JSON response with server list  
  - Edge cases: HTTP errors, API rate limiting, empty or partial server list responses  
- **Merge**  
  - Type: `n8n-nodes-base.merge`  
  - Role: Merges outputs (used here to synchronize data flow)  
  - Inputs: Output from PulseMCP Fetch MCP Servers  
  - Outputs: Passes merged data downstream  
- **Parse MCP Server list into documents w metadata**  
  - Type: `n8n-nodes-base.code` (JavaScript)  
  - Role: Transforms server JSON into two arrays:  
    - `documents`: strings combining server name and description  
    - `metadata`: strings with server name, GitHub stars, and download counts  
  - Also extracts query and reranking instructions from LLM Agent output  
  - Inputs: Merged output of server list and LLM decision data  
  - Outputs: An object containing `query`, `instruction`, `documents`, `metadata`, and original `servers` JSON  
  - Edge cases: Missing server fields, empty server list, JSON parse errors  

---

#### 1.4 Contextual AI Reranking  
- **Overview:** Sends query, reranking instructions, and server documents to the Contextual AI reranker API to obtain relevance scores and rankings.  
- **Nodes Involved:** ContextualAI Reranker, Merge1  

**Node Details:**  
- **ContextualAI Reranker**  
  - Type: `n8n-nodes-base.httpRequest`  
  - Role: POST request to rerank API endpoint to score servers based on query and instructions  
  - Configuration:  
    - URL: `https://api.contextual.ai/v1/rerank`  
    - Authorization header uses environment variable `CONTEXTUALAI_API_KEY`  
    - Body includes `query`, `instruction`, `documents`, `metadata`, and reranker `model` (`ctxl-rerank-v2-instruct-multilingual`)  
    - Content-Type: application/json  
  - Inputs: Output from Parse MCP Server list into documents w metadata  
  - Outputs: JSON results with relevance scores and indices  
  - Edge cases: Auth failures (invalid API key), HTTP errors, empty or malformed response  
- **Merge1**  
  - Type: `n8n-nodes-base.merge`  
  - Role: Synchronizes outputs for next processing step  
  - Inputs: From ContextualAI Reranker and Parse MCP Server list node (for server metadata)  
  - Outputs: Combined data downstream  

---

#### 1.5 Result Formatting & Output  
- **Overview:** Extracts top 5 reranked servers, formats a readable message, and sends final response back to user.  
- **Nodes Involved:** Format the top 5 results, Final Response2, Final Response1 (for no MCP case)  

**Node Details:**  
- **Format the top 5 results**  
  - Type: `n8n-nodes-base.code` (JavaScript)  
  - Role: Maps top 5 reranked server indices to server metadata, builds human-readable list with stars, downloads, scores, and descriptions  
  - Inputs: Output from Merge1 (reranker results and server metadata)  
  - Outputs: JSON message with formatted text  
  - Edge cases: Less than 5 results, missing server metadata fields  
- **Final Response2**  
  - Type: `@n8n/n8n-nodes-langchain.chat`  
  - Role: Sends formatted ranked server list back to user  
  - Inputs: Message from Format the top 5 results  
  - Configuration: Wait for user reply disabled  
- **Final Response1**  
  - Type: `@n8n/n8n-nodes-langchain.chat`  
  - Role: Sends response explaining no MCP servers are required (from If false branch)  
  - Inputs: JSON output from LLM Agent decision parsed for reason  
  - Configuration: Wait for user reply disabled  

---

### 3. Summary Table

| Node Name                         | Node Type                              | Functional Role                             | Input Node(s)                      | Output Node(s)                      | Sticky Note                                                                                                                             |
|----------------------------------|--------------------------------------|---------------------------------------------|----------------------------------|-----------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| User-Query                       | @n8n/n8n-nodes-langchain.chatTrigger | Receives user query and starts workflow    | External webhook                 | LLM Agent for Decision-Making     |                                                                                                                                         |
| OpenAI Chat Model                | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provides GPT-4.1-mini LLM responses         | -                                | LLM Agent for Decision-Making     |                                                                                                                                         |
| LLM Agent for Decision-Making    | @n8n/n8n-nodes-langchain.agent        | Decides if MCP servers are needed; generates instructions | User-Query, OpenAI Chat Model   | If                               | ## 1. Determine whether MCP servers are needed Based on user's request, LLM determines the need for an MCP Server, provides a reason, and if needed, provides reranking instruction text which will be passed to reranker |
| If                              | n8n-nodes-base.if                      | Conditional routing based on MCP usage decision | LLM Agent for Decision-Making    | PulseMCP Fetch MCP Servers, Final Response1 |                                                                                                                                         |
| PulseMCP Fetch MCP Servers       | n8n-nodes-base.httpRequest             | Fetches MCP servers list from PulseMCP API | If (true branch)                 | Merge                            | ## 2. Fetch MCP Server list and format them We fetch 5000 MCP Servers from PulseMCP directory and parse them as documents to pass it onto the Contextual AI Reranker |
| Merge                          | n8n-nodes-base.merge                   | Synchronizes MCP servers fetching output    | PulseMCP Fetch MCP Servers       | Parse MCP Server list into documents w metadata |                                                                                                                                         |
| Parse MCP Server list into documents w metadata | n8n-nodes-base.code                    | Formats servers as documents and extracts metadata | Merge, LLM Agent for Decision-Making | ContextualAI Reranker, Merge1     |                                                                                                                                         |
| ContextualAI Reranker           | n8n-nodes-base.httpRequest             | Sends data to Contextual AI reranker API    | Parse MCP Server list into documents w metadata | Merge1                           | ## 3. Rerank the servers and display top five results We use Contextual AI's reranker to re-rank the servers and identify the top 5 servers based on the user query and re-ranker instruction |
| Merge1                         | n8n-nodes-base.merge                   | Synchronizes reranker results and server metadata | ContextualAI Reranker, Parse MCP Server list into documents w metadata | Format the top 5 results          |                                                                                                                                         |
| Format the top 5 results         | n8n-nodes-base.code                    | Extracts and formats top 5 ranked servers   | Merge1                          | Final Response2                  |                                                                                                                                         |
| Final Response2                 | @n8n/n8n-nodes-langchain.chat         | Sends formatted top 5 MCP servers to user  | Format the top 5 results         | -                               |                                                                                                                                         |
| Final Response1                 | @n8n/n8n-nodes-langchain.chat         | Sends final response if no MCP servers needed | If (false branch)              | -                               |                                                                                                                                         |
| Sticky Note                    | n8n-nodes-base.stickyNote               | Documentation and explanation notes         | -                                | -                               | # Dynamic MCP Selection PROBLEM Thousands of MCP Servers exist and many are updated daily, making server selection difficult for LLMs.  |
| Sticky Note1                   | n8n-nodes-base.stickyNote               | Describes decision-making block               | -                                | -                               | ## 1. Determine whether MCP servers are needed Based on user's request, LLM determines the need for an MCP Server, provides a reason, and if needed, provides reranking instruction text which will be passed to reranker |
| Sticky Note2                   | n8n-nodes-base.stickyNote               | Describes MCP fetching and formatting block | -                                | -                               | ## 2. Fetch MCP Server list and format them We fetch 5000 MCP Servers from PulseMCP directory and parse them as documents to pass it onto the Contextual AI Reranker |
| Sticky Note3                   | n8n-nodes-base.stickyNote               | Describes reranking and final response block | -                                | -                               | ## 3. Rerank the servers and display top five results We use Contextual AI's reranker to re-rank the servers and identify the top 5 servers based on the user query and re-ranker instruction. [Blog](https://contextual.ai/blog/introducing-instruction-following-reranker/) |

---

### 4. Reproducing the Workflow from Scratch

**Step 1: Create Chat Trigger Node**  
- Node Name: `User-Query`  
- Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
- Configure:  
  - Public webhook enabled  
  - Allow file uploads: true  
  - Initial messages: “Try MCP Reranker using Contextual AI's Reranker v2”  
- Connect output to `LLM Agent for Decision-Making` node  

**Step 2: Create OpenAI Chat Model Node**  
- Node Name: `OpenAI Chat Model`  
- Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
- Configure:  
  - Model: GPT-4.1-mini  
  - Response format: JSON object  
  - Credentials: Use OpenAI API key credentials  
- Connect this node’s output to `LLM Agent for Decision-Making` node as the AI language model  

**Step 3: Create LLM Agent for Decision-Making Node**  
- Node Name: `LLM Agent for Decision-Making`  
- Type: `@n8n/n8n-nodes-langchain.agent`  
- Configure:  
  - System message prompt (copy the detailed prompt from the workflow) instructing to analyze user query, decide if MCP is needed, and output JSON with `use_mcp`, `reason`, `instruction`  
- Inputs:  
  - Input from `User-Query` node (user chat input)  
  - AI language model connected to `OpenAI Chat Model`  
- Output to `If` node  

**Step 4: Create If Node**  
- Node Name: `If`  
- Type: `n8n-nodes-base.if`  
- Configure:  
  - Condition: Expression boolean equals true  
  - Expression: `{{$json.output.parseJson().use_mcp}}`  
- True branch output to `PulseMCP Fetch MCP Servers` node  
- False branch output to `Final Response1` node  

**Step 5: Create HTTP Request Node to Fetch MCP Servers**  
- Node Name: `PulseMCP Fetch MCP Servers`  
- Type: `n8n-nodes-base.httpRequest`  
- Configure:  
  - HTTP Method: GET  
  - URL: `https://api.pulsemcp.com/v0beta/servers`  
  - Query Parameters: `count_per_page=5000`, `offset=0`  
- Input: From If node true branch  
- Output: To `Merge` node  

**Step 6: Create Merge Node**  
- Node Name: `Merge`  
- Type: `n8n-nodes-base.merge`  
- Configure: Default settings  
- Input: From `PulseMCP Fetch MCP Servers`  
- Output: To `Parse MCP Server list into documents w metadata` node  

**Step 7: Create Code Node to Parse Server List**  
- Node Name: `Parse MCP Server list into documents w metadata`  
- Type: `n8n-nodes-base.code`  
- Configure: Paste JavaScript code to:  
  - Extract servers from HTTP response  
  - Build `documents` and `metadata` arrays with server info  
  - Retrieve query and instructions from LLM Agent output (parse JSON)  
  - Output combined object with `query`, `instruction`, `documents`, `metadata`, and `servers`  
- Input: From `Merge` node (server list) and LLM Agent node (for instruction)  
- Output: To `ContextualAI Reranker` and second input of `Merge1` node  

**Step 8: Create HTTP Request Node for Contextual AI Reranker**  
- Node Name: `ContextualAI Reranker`  
- Type: `n8n-nodes-base.httpRequest`  
- Configure:  
  - HTTP Method: POST  
  - URL: `https://api.contextual.ai/v1/rerank`  
  - Headers:  
    - Authorization: Bearer `{{$vars.CONTEXTUALAI_API_KEY}}` (ensure environment variable is set)  
    - Content-Type: application/json  
  - Body Parameters: `query`, `instruction`, `documents`, `metadata`, `model` = `ctxl-rerank-v2-instruct-multilingual` (from input JSON)  
- Input: From `Parse MCP Server list into documents w metadata`  
- Output: To `Merge1` node  

**Step 9: Create Merge Node**  
- Node Name: `Merge1`  
- Type: `n8n-nodes-base.merge`  
- Configure: Default settings  
- Inputs:  
  - Main input from `ContextualAI Reranker` (rerank results)  
  - Secondary input from `Parse MCP Server list into documents w metadata` (server metadata)  
- Output: To `Format the top 5 results` node  

**Step 10: Create Code Node to Format Top 5 Results**  
- Node Name: `Format the top 5 results`  
- Type: `n8n-nodes-base.code`  
- Configure: Paste JavaScript code to:  
  - Extract top 5 ranked servers by relevance score  
  - Format a readable message string listing server name, stars, downloads, score, and description  
- Input: From `Merge1` node  
- Output: To `Final Response2` node  

**Step 11: Create Final Response Nodes**  
- Node Name: `Final Response2`  
  - Type: `@n8n/n8n-nodes-langchain.chat`  
  - Configure: Message set to formatted text from previous node; waitUserReply disabled  
  - Input: From `Format the top 5 results`  
- Node Name: `Final Response1`  
  - Type: `@n8n/n8n-nodes-langchain.chat`  
  - Configure: Message set to reason from LLM Agent output if MCP not needed; waitUserReply disabled  
  - Input: From If node false branch  

**Step 12: Add Sticky Notes for Documentation (Optional)**  
- Add sticky notes with content describing blocks as per original workflow for clarity during editing  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                        | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Dynamic MCP Selection problem statement and rationale                                                                                                | See Sticky Note content labeled "Dynamic MCP Selection"                                           |
| Contextual AI reranker models available: ctxl-rerank-v2-instruct-multilingual, ctxl-rerank-v2-instruct-multilingual-mini, ctxl-rerank-v1-instruct    | https://contextual.ai/blog/introducing-instruction-following-reranker/                             |
| Setup instructions for CONTEXTUALAI_API_KEY environment variable and OpenAI API key                                                                  | Workflow description and sticky notes                                                             |
| Feedback and support email: reranker-feedback@contextual.ai                                                                                          | Provided in workflow sticky notes                                                                 |

---

**Disclaimer:** The provided text is generated exclusively from an automated n8n workflow adhering to current content policies and contains no illegal or protected content. All processed data is legal and publicly accessible.

---