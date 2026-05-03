Build a Multi-Agent system with n8n, Qdrant, Gmail & OpenAI

https://n8nworkflows.xyz/workflows/build-a-multi-agent-system-with-n8n--qdrant--gmail---openai-11525


# Build a Multi-Agent system with n8n, Qdrant, Gmail & OpenAI

### 1. Workflow Overview

This workflow, titled **"Build a Multi-Agent system with n8n, Qdrant, Gmail & OpenAI"**, is designed to implement a multi-agent AI system within n8n that intelligently routes user queries to specialized sub-agents or tools. It targets use cases involving document retrieval and summarization, email management via Gmail, and fetching recent news data, all orchestrated through a coordinating AI agent.

The workflow consists of the following logical blocks:

- **1.1 Input Reception**: Receives user queries and summary requests.
- **1.2 Document Retrieval and Summarization (RAG Sub-agent)**: Retrieves documents from a Qdrant vector database and generates summaries.
- **1.3 Gmail Email Management (Gmail Sub-agent)**: Handles Gmail operations like reading, sending, and drafting emails.
- **1.4 News Retrieval Tool**: Fetches recent news data via an HTTP request.
- **1.5 Multi-Agent Coordination**: The main AI agent analyzes user input and routes queries to the appropriate sub-agent or tool.
- **1.6 Conversation Memory**: Stores conversation context for multi-turn interactions.
- **1.7 Sub-workflow for Full Document Summarization**: A nested workflow that performs detailed document summarization from Google Drive files.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Input Reception

**Overview:**  
This block collects user inputs either as chat queries or document summary requests, triggering the workflow execution.

**Nodes Involved:**  
- Receive Summary Request  
- Chat with an Agent  

**Node Details:**  

- **Receive Summary Request**  
  - *Type:* `executeWorkflowTrigger`  
  - *Role:* Triggers the workflow for summary requests, expecting inputs `chatInput` (text query) and `file_id` (Google Drive file identifier).  
  - *Inputs:* External trigger with user parameters.  
  - *Outputs:* Passes data to "Get Document" node.  
  - *Edge cases:* Missing or invalid `file_id` leads to failed document retrieval.  
  - *Version:* 1.1  

- **Chat with an Agent**  
  - *Type:* `chatTrigger`  
  - *Role:* Provides webhook endpoint to receive chat queries for the multi-agent system.  
  - *Inputs:* External chat messages.  
  - *Outputs:* Routes user input to "Answer Questions" node for processing.  
  - *Edge cases:* Webhook connectivity issues or malformed messages.  
  - *Version:* 1.3  

---

#### 1.2 Document Retrieval and Summarization (RAG Sub-agent)

**Overview:**  
This block handles document search in a vector database, downloads and extracts document content from Google Drive, converts it to markdown, and creates summaries using OpenAI models.

**Nodes Involved:**  
- Search Documents  
- RAG sub-agent  
- Get Document  
- Extract Content  
- Convert to Markdown  
- Process Request  
- Generate Summary  
- Format Summary Output  
- Generate RAG sub-agent Response  
- Summarize Document (sub-workflow call)  

**Node Details:**  

- **Search Documents**  
  - *Type:* `vectorStoreQdrant`  
  - *Role:* Searches the Qdrant vector database named "n8n-rag" for relevant documents matching user queries. Returns top 10 semantically similar results.  
  - *Configuration:* Mode set to "retrieve-as-tool"; tool description specifies use to fetch facts or quotes from uploaded documents.  
  - *Credentials:* Qdrant API key required.  
  - *Outputs:* Feeds results to "RAG sub-agent".  
  - *Edge cases:* API key issues, network failures, empty search results.  
  - *Version:* 1.3  

- **RAG sub-agent**  
  - *Type:* `agentTool`  
  - *Role:* Acts as an AI assistant specialized in document knowledge base queries, strictly using document search results to answer.  
  - *Configuration:* Includes detailed system message instructing to always search documents first, cite sources, and handle unclear questions by asking for clarification. Also calls "Summarize Document" sub-workflow for summaries.  
  - *Inputs:* Receives text prompt from upstream nodes or main agent.  
  - *Outputs:* Routes answers to "Answer Questions" node.  
  - *Edge cases:* Misinterpretation of instructions, failure to call sub-workflow, tool invocation errors.  
  - *Version:* 2.2  

- **Get Document**  
  - *Type:* `googleDrive`  
  - *Role:* Downloads a specified file from Google Drive based on `file_id`.  
  - *Credentials:* Google Drive OAuth2.  
  - *Outputs:* Passes raw file data to "Extract Content".  
  - *Edge cases:* Unauthorized access, invalid file ID, download failures.  
  - *Version:* 3  

- **Extract Content**  
  - *Type:* `extractFromFile`  
  - *Role:* Extracts plain text content from the downloaded file.  
  - *Outputs:* Passes extracted text to "Convert to Markdown".  
  - *Edge cases:* Unsupported file formats, extraction errors.  
  - *Version:* 1  

- **Convert to Markdown**  
  - *Type:* `markdown`  
  - *Role:* Converts extracted HTML or text content into Markdown format to facilitate processing.  
  - *Inputs:* Receives content from "Extract Content".  
  - *Outputs:* Routes Markdown text to "Process Request".  
  - *Version:* 1  

- **Process Request**  
  - *Type:* `chainLlm`  
  - *Role:* Sends the Markdown document to an OpenAI LLM chain with a custom prompt to summarize key points relevant to the user query.  
  - *Configuration:* Uses prompt instructing expert summarization.  
  - *Outputs:* Passes generated text to "Format Summary Output".  
  - *Version:* 1.7  

- **Generate Summary**  
  - *Type:* `lmChatOpenAi`  
  - *Role:* Calls OpenAI GPT-4o model to generate the document summary.  
  - *Credentials:* OpenAI API key required.  
  - *Outputs:* Feeds results to "Process Request".  
  - *Edge cases:* API rate limits, timeouts, model errors.  
  - *Version:* 1.2  

- **Format Summary Output**  
  - *Type:* `set`  
  - *Role:* Sets the final summary text in the output JSON under the key "summary".  
  - *Outputs:* Sends structured summary downstream.  
  - *Version:* 3.4  

- **Generate RAG sub-agent Response**  
  - *Type:* `lmChatOpenAi`  
  - *Role:* Generates AI responses for the RAG sub-agent using GPT-4o model with temperature 0.4 for moderate creativity.  
  - *Credentials:* OpenAI.  
  - *Outputs:* Connects back to "RAG sub-agent".  
  - *Version:* 1.2  

- **Summarize Document**  
  - *Type:* `toolWorkflow` (Sub-workflow)  
  - *Role:* Invoked by RAG sub-agent to perform full document summarization workflow with inputs `file_id` and `chatInput`.  
  - *Outputs:* Returns summaries to RAG sub-agent.  
  - *Version:* 2.2  

---

#### 1.3 Gmail Email Management (Gmail Sub-agent)

**Overview:**  
This block enables interaction with Gmail, supporting operations such as reading emails, sending messages, creating drafts, and retrieving multiple messages.

**Nodes Involved:**  
- Gmail sub-agent  
- Get multiple messages in Gmail  
- Read a message in Gmail  
- Send a message in Gmail  
- Create a draft in Gmail  
- Generate Gmail sub-agent Response  

**Node Details:**  

- **Gmail sub-agent**  
  - *Type:* `agentTool`  
  - *Role:* Specialized AI agent to handle Gmail-related tasks including sending, reading, drafting, and searching emails.  
  - *Configuration:* System message instructs to retry searches with different parameters if no results found.  
  - *Inputs:* Receives commands from main agent or user.  
  - *Outputs:* Routes to "Answer Questions".  
  - *Version:* 2.2  

- **Get multiple messages in Gmail**  
  - *Type:* `gmailTool`  
  - *Role:* Retrieves up to 20 emails from Gmail inbox with optional search query provided dynamically.  
  - *Credentials:* Gmail OAuth2 required.  
  - *Edge cases:* OAuth token expiry, Gmail API rate limits.  
  - *Version:* 2.1  

- **Read a message in Gmail**  
  - *Type:* `gmailTool`  
  - *Role:* Retrieves full content of a single Gmail message identified by message ID.  
  - *Credentials:* Gmail OAuth2.  
  - *Edge cases:* Invalid message ID or permissions.  
  - *Version:* 2.1  

- **Send a message in Gmail**  
  - *Type:* `gmailTool`  
  - *Role:* Sends an email with dynamic recipient, subject, and HTML body content.  
  - *Credentials:* Gmail OAuth2.  
  - *Edge cases:* Sending failures, invalid email addresses.  
  - *Version:* 2.1  

- **Create a draft in Gmail**  
  - *Type:* `gmailTool`  
  - *Role:* Creates a draft email with specified subject and HTML message body.  
  - *Credentials:* Gmail OAuth2.  
  - *Edge cases:* Draft creation failures, invalid content.  
  - *Version:* 2.1  

- **Generate Gmail sub-agent Response**  
  - *Type:* `lmChatOpenRouter`  
  - *Role:* Uses OpenRouter Gemini 2.5 Flash model to generate responses for Gmail-related queries.  
  - *Credentials:* OpenRouter API key required.  
  - *Version:* 1  

---

#### 1.4 News Retrieval Tool

**Overview:**  
Fetches current news data using an external news API when requested by the user.

**Nodes Involved:**  
- Check News or Recent Data  

**Node Details:**  

- **Check News or Recent Data**  
  - *Type:* `httpRequestTool`  
  - *Role:* Queries NewsAPI.org for the latest news articles with parameters for query, sorting, and API key authentication.  
  - *Credentials:* API key for NewsAPI.org.  
  - *Edge cases:* API rate limits, invalid keys, connectivity issues.  
  - *Version:* 4.2  

---

#### 1.5 Multi-Agent Coordination

**Overview:**  
This block acts as the central coordinator AI agent that analyzes user inputs and delegates tasks to the appropriate sub-agents or tools.

**Nodes Involved:**  
- Answer Questions (Multi-Agent Coordinator)  
- Generate Agent Response  
- Store Conversation with an Agent  

**Node Details:**  

- **Answer Questions**  
  - *Type:* `agent`  
  - *Role:* Main AI agent coordinating sub-agents: RAG for document queries, Gmail agent for emails, and news tool for current events. Contains detailed system instructions on routing logic and response synthesis.  
  - *Inputs:* Receives user queries from "Chat with an Agent" or sub-agents.  
  - *Outputs:* Routes queries to sub-agents or tools and collects responses.  
  - *Version:* 2.2  

- **Generate Agent Response**  
  - *Type:* `lmChatOpenAi`  
  - *Role:* Generates final multi-agent response using OpenAI GPT-4o with moderate temperature (0.4).  
  - *Credentials:* OpenAI API key.  
  - *Outputs:* Sends response to "Answer Questions".  
  - *Edge cases:* API failures, timeout.  
  - *Version:* 1.2  

- **Store Conversation with an Agent**  
  - *Type:* `memoryBufferWindow`  
  - *Role:* Maintains conversation history with a context window of last 20 interactions to support multi-turn dialogue.  
  - *Outputs:* Feeds context back to "Answer Questions".  
  - *Version:* 1.3  

---

#### 1.6 Sub-workflow for Full Document Summarization

**Overview:**  
A nested workflow invoked by the RAG sub-agent that performs a multi-step process to download, extract, convert, and summarize documents from Google Drive.

**Nodes Involved:**  
- Receive Summary Request (entry node)  
- Get Document  
- Extract Content  
- Convert to Markdown  
- Process Request  
- Generate Summary  
- Format Summary Output  

**Node Details:**  
This sub-workflow takes inputs `file_id` and `chatInput`, downloads the file from Google Drive, extracts text, converts it to markdown, summarizes content with OpenAI models, and outputs a formatted summary string.

---

### 3. Summary Table

| Node Name                  | Node Type                      | Functional Role                                    | Input Node(s)                | Output Node(s)               | Sticky Note                                                                                   |
|----------------------------|--------------------------------|---------------------------------------------------|------------------------------|------------------------------|-----------------------------------------------------------------------------------------------|
| Receive Summary Request     | executeWorkflowTrigger          | Trigger workflow for document summary requests    | (external)                   | Get Document                 |                                                                                               |
| Get Document               | googleDrive                    | Downloads file from Google Drive                   | Receive Summary Request       | Extract Content              |                                                                                               |
| Extract Content            | extractFromFile                | Extracts text from downloaded file                 | Get Document                 | Convert to Markdown          |                                                                                               |
| Convert to Markdown        | markdown                      | Converts extracted text to Markdown                | Extract Content              | Process Request              |                                                                                               |
| Process Request            | chainLlm                      | Summarizes document content with OpenAI LLM       | Generate Summary             | Format Summary Output        |                                                                                               |
| Generate Summary           | lmChatOpenAi                  | Generates summary text from OpenAI GPT-4o          |                              | Process Request              |                                                                                               |
| Format Summary Output      | set                           | Formats and sets summary output                     | Process Request              |                              |                                                                                               |
| Search Documents           | vectorStoreQdrant             | Searches Qdrant vector DB for relevant documents   | Generate Embeddings          | RAG sub-agent               |                                                                                               |
| Generate Embeddings        | embeddingsOpenAi              | Generates text embeddings for document search      |                              | Search Documents             |                                                                                               |
| RAG sub-agent             | agentTool                     | AI sub-agent specialized in document retrieval     | Search Documents, Summarize Document | Answer Questions        | Sticky Note2: "RAG sub-agent - Searches documents in Qdrant vector database and prepares document summaries." |
| Summarize Document         | toolWorkflow                  | Sub-workflow for full document summarization       |                              | RAG sub-agent               |                                                                                               |
| Answer Questions           | agent                        | Multi-agent coordinator routing user queries       | Chat with an Agent, RAG sub-agent, Gmail sub-agent, Check News or Recent Data | Generate Agent Response     | Sticky Note6: "Main AI Agent - Routes user queries to the corresponding Tools (RAG and Gmail sub-agents, news tool)." |
| Generate Agent Response    | lmChatOpenAi                 | Generates AI response for main agent                | Answer Questions             | Answer Questions             |                                                                                               |
| Store Conversation with an Agent | memoryBufferWindow        | Stores conversation context for multi-turn dialogue|                              | Answer Questions             |                                                                                               |
| Gmail sub-agent            | agentTool                    | AI sub-agent specialized in Gmail operations        | Get multiple messages in Gmail, Read a message in Gmail, Send a message in Gmail, Create a draft in Gmail | Answer Questions             | Sticky Note1: "Gmail sub-agent - Reads, send emails and creates drafts."                      |
| Get multiple messages in Gmail | gmailTool                  | Retrieves multiple Gmail messages with optional search |                              | Gmail sub-agent             |                                                                                               |
| Read a message in Gmail    | gmailTool                    | Retrieves content of a single Gmail message         |                              | Gmail sub-agent             |                                                                                               |
| Send a message in Gmail    | gmailTool                    | Sends an email message                               |                              | Gmail sub-agent             |                                                                                               |
| Create a draft in Gmail    | gmailTool                    | Creates a draft email                               |                              | Gmail sub-agent             |                                                                                               |
| Generate Gmail sub-agent Response | lmChatOpenRouter           | Generates responses for Gmail sub-agent using OpenRouter Gemini 2.5 Flash |                              | Gmail sub-agent             |                                                                                               |
| Check News or Recent Data  | httpRequestTool              | Fetches recent news from NewsAPI                     |                              | Answer Questions             | Sticky Note3: "News Tool - Fetches latest news via API."                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Input Reception Nodes:**

   - Add an `executeWorkflowTrigger` node named **Receive Summary Request**. Configure it to accept inputs `chatInput` (string) and `file_id` (string).  
   - Add a `chatTrigger` node named **Chat with an Agent** to receive chat messages via webhook.

2. **Set Up the Document Retrieval and Summarization Block:**

   - Add a `googleDrive` node named **Get Document** to download files by `fileId` taken from `file_id` input. Configure with Google Drive OAuth2 credentials.  
   - Add an `extractFromFile` node named **Extract Content** to extract text content from the downloaded file.  
   - Add a `markdown` node named **Convert to Markdown** to convert extracted content to markdown format.  
   - Add a `lmChatOpenAi` node named **Generate Summary** configured with the GPT-4o model and your OpenAI API credentials.  
   - Add a `chainLlm` node named **Process Request** to summarize the markdown text. Use a prompt instructing expert summarization focusing on key points relevant to user queries.  
   - Add a `set` node named **Format Summary Output** to assign the summary text to the output field `summary`.  
   - Connect nodes in this order: Receive Summary Request → Get Document → Extract Content → Convert to Markdown → Generate Summary → Process Request → Format Summary Output.

3. **Set Up Qdrant Search and Embeddings:**

   - Add an `embeddingsOpenAi` node named **Generate Embeddings** with OpenAI API credentials to generate text embeddings.  
   - Add a `vectorStoreQdrant` node named **Search Documents** configured to search the Qdrant collection named "n8n-rag", retrieving top 10 results. Provide Qdrant API credentials.  
   - Connect: Generate Embeddings → Search Documents.

4. **Configure RAG Sub-agent:**

   - Add an `agentTool` node named **RAG sub-agent**.  
   - Set system message instructing to always search documents first, cite sources, never rely solely on general knowledge, ask for clarifications, and call the "Summarize Document" sub-workflow for summaries.  
   - Connect Search Documents output to this node's input.  
   - Connect this node's output to the **Answer Questions** node in the coordination block.

5. **Create Sub-workflow for Document Summarization:**

   - Create a new workflow or reuse the main one with the nodes: Receive Summary Request → Get Document → Extract Content → Convert to Markdown → Generate Summary → Process Request → Format Summary Output.  
   - Save and note the workflow ID for calling within the RAG sub-agent node as a tool workflow.

6. **Set Up Gmail Sub-agent:**

   - Add an `agentTool` node named **Gmail sub-agent** with system message instructing retries on email search failures.  
   - Add `gmailTool` nodes for:  
     - **Get multiple messages in Gmail** (operation: getAll, limit 20, with dynamic search filter input)  
     - **Read a message in Gmail** (operation: get, requires messageId)  
     - **Send a message in Gmail** (operation: send, dynamic To, Subject, and HTML message)  
     - **Create a draft in Gmail** (operation: draft, with dynamic subject and message)  
   - Connect all Gmail tool nodes’ outputs to the Gmail sub-agent input.  
   - Use Gmail OAuth2 credentials for these nodes.

7. **Configure News Retrieval Tool:**

   - Add an `httpRequestTool` node named **Check News or Recent Data** pointing to `https://newsapi.org/v2/everything`.  
   - Configure query parameters to accept `q` (search term), `sortBy=publishedAt`, and the NewsAPI key.  
   - Use HTTP Query Auth credential with NewsAPI key.

8. **Set Up Multi-Agent Coordinator:**

   - Add an `agent` node named **Answer Questions** with system message describing routing logic:  
     - Document questions → RAG sub-agent  
     - Email requests → Gmail sub-agent  
     - News requests → News tool  
     - Complex queries may invoke multiple agents  
     - Always call Gmail sub-agent to send summary emails if requested.  
   - Connect outputs from RAG sub-agent, Gmail sub-agent, and News tool nodes to this agent’s inputs.

9. **Add AI Response Generation and Memory:**

   - Add an `lmChatOpenAi` node named **Generate Agent Response** configured with GPT-4o and temperature 0.4, connected to **Answer Questions** output.  
   - Add a `memoryBufferWindow` node named **Store Conversation with an Agent** with context window length 20, connected to **Answer Questions** to maintain conversation context.

10. **Connect Main Flow:**

    - Connect **Chat with an Agent** output to **Answer Questions** input.  
    - Connect **Answer Questions** output to **Generate Agent Response** and **Store Conversation with an Agent** as shown above.

11. **Credential Setup:**

    - Configure credentials for:  
      - Google Drive OAuth2 (for file download).  
      - Gmail OAuth2 (for Gmail API access).  
      - Qdrant API key.  
      - OpenAI API key.  
      - OpenRouter API key (for Gmail sub-agent language model).  
      - NewsAPI key with HTTP Query Auth.

12. **Notes on Default Values and Constraints:**

    - Use GPT-4o model for all OpenAI LLM calls.  
    - Use temperature 0.4 for controlled creativity in AI responses.  
    - Limit Gmail message fetch to 20 messages per request.  
    - Always validate presence and correctness of `file_id` in document summarization.  
    - Ensure Qdrant collection "n8n-rag" exists and is populated with vectorized documents or upload own collection.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                    | Context or Link                                                                                     |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| **How it works:** AI Agent receives user queries and routes them to specialized sub-agents: RAG searches Qdrant vector DB and prepares document summaries; Email sub-agent manages Gmail interactions; HTTP Request Tool fetches news.          | Sticky Note7 content in workflow.                                                                 |
| **Setup instructions:** Import template, configure credentials for Google Drive, Gmail, Qdrant, OpenAI, OpenRouter, and NewsAPI; import example Qdrant collection or upload your own documents.                                                 | Sticky Note7 content in workflow.                                                                 |
| Qdrant collection example available at: https://drive.google.com/drive/u/2/folders/1BevhU5qdgNDFbK4D9oAYGeK0Dt5sEaxQ                                                                                                                          | Mentioned in workflow notes.                                                                       |
| Gmail sub-agent handles reading, sending, and drafting emails with retry logic for searches without results.                                                                                                                                   | Sticky Note1.                                                                                      |
| RAG sub-agent strictly bases answers on document knowledge base, citing sources and using direct quotes for factual information.                                                                                                             | Sticky Note2.                                                                                      |
| News Tool fetches latest news from NewsAPI.org when explicitly requested; do not use for general queries that can be answered by documents.                                                                                                  | Sticky Note3.                                                                                      |
| Main AI Agent routes queries intelligently, coordinating multiple agents and synthesizing responses.                                                                                                                                           | Sticky Note6.                                                                                      |
| For more on n8n LangChain integrations see: https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolhttprequest/                                                                                                | Linked in Sticky Note7.                                                                            |

---

**Disclaimer:** The provided text is extracted exclusively from an automated workflow created with n8n, adhering strictly to content policies. It contains no illegal, offensive, or protected material. All manipulated data is legal and public.