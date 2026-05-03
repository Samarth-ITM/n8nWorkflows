Build a Tax Code Assistant with Qdrant, Mistral.ai and OpenAI

https://n8nworkflows.xyz/workflows/build-a-tax-code-assistant-with-qdrant--mistral-ai-and-openai-2341


# Build a Tax Code Assistant with Qdrant, Mistral.ai and OpenAI

### 1. Workflow Overview

This workflow builds a Tax Code Assistant chatbot leveraging n8n, Qdrant vector search, Mistral.ai embeddings and AI models, and OpenAI for conversational capabilities. It is designed specifically to ingest, structure, and query a government tax code document for the State of Texas by:

- Downloading and extracting tax code PDFs from a zipped archive.
- Splitting the document into chapters and further into sections to preserve contextual boundaries.
- Enriching each section with metadata (chapter, section number, title) to support precise scoped queries.
- Inserting the structured data into a Qdrant vector store using Mistral.ai embeddings.
- Providing an AI Agent chatbot interface to answer user questions by querying the vector store with two distinct tools:
  - An **Ask Tool** that generates embeddings of user queries and performs vector similarity search.
  - A **Search Tool** that filters and scrolls through sections in Qdrant by metadata keys.
- Using OpenAI's GPT chat model and window buffer memory to maintain conversational context.

Logical Blocks:

- **1.1 Data Acquisition & Extraction**: Download and unzip tax code PDFs.
- **1.2 Document Parsing & Sectioning**: Extract PDF text, split into chapters and sections.
- **1.3 Vector Store Ingestion**: Chunk sections and insert into Qdrant with metadata.
- **1.4 AI Agent Chatbot Setup**: Setup AI tools and agent with memory to answer queries.
- **1.5 Qdrant API Tools**: Custom workflows to query Qdrant for relevant data retrieval.

---

### 2. Block-by-Block Analysis

#### 1.1 Data Acquisition & Extraction

**Overview:**  
Downloads the zipped tax code PDFs from the government website and extracts individual PDF files for further processing.

**Nodes Involved:**  
- When clicking ‘Test workflow’  
- Get Tax Code Zip File  
- Extract Zip Files  
- Files as Items  

**Node Details:**  

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Entry point to start workflow manually.  
  - Input/Output: No input; outputs to "Get Tax Code Zip File".  
  - Failure cases: Manual trigger may be idle if not initiated.  

- **Get Tax Code Zip File**  
  - Type: HTTP Request  
  - Role: Downloads the zipped PDF archive from a public URL (`https://statutes.capitol.texas.gov/Docs/Zips/TX.pdf.zip`).  
  - Config: Response format set to "file" to handle binary zip content.  
  - Inputs: From manual trigger. Outputs to "Extract Zip Files".  
  - Edge cases: HTTP errors, network timeouts, or URL unavailability.  

- **Extract Zip Files**  
  - Type: Compression (Unzip)  
  - Role: Unzips the downloaded archive to extract individual PDF files.  
  - Inputs: Zip file binary from HTTP Request. Outputs to "Files as Items".  
  - Edge cases: Corrupt zip archive, extraction errors.  

- **Files as Items**  
  - Type: Split Out  
  - Role: Splits array of extracted PDF files into individual items for processing.  
  - Input: Binary zip file contents from "Extract Zip Files".  
  - Output: Each PDF file as a separate item to "Extract PDF Contents".  

---

#### 1.2 Document Parsing & Sectioning

**Overview:**  
Extracts raw text from PDFs, then parses this text to split the document into well-defined chapters and sections using regex-based text manipulations and array transformations.

**Nodes Involved:**  
- Extract PDF Contents  
- Extract From Chapter  
- Map To Sections  
- Sections To List  
- Only Valid Sections  
- For Each Section...  

**Node Details:**  

- **Extract PDF Contents**  
  - Type: Extract From File  
  - Role: Extracts raw text from each PDF binary file.  
  - Config: Operation set to PDF, binary property uses dynamic filename per item.  
  - Input: Individual PDF files from "Files as Items".  
  - Output: Extracted text to "Extract From Chapter".  
  - Failures: Malformed PDFs, unsupported formats, extraction errors.  

- **Extract From Chapter**  
  - Type: Set  
  - Role: Parses and splits raw text into chapters and sections using regex to isolate section titles and content.  
  - Config: Uses JavaScript expressions to split text on patterns like `Sec.\nA<number>.<number>.AA` and maps to objects with `title` and `content` fields.  
  - Input: Extracted text from "Extract PDF Contents".  
  - Output: Parsed arrays of section titles and contents to "Map To Sections".  
  - Edge cases: Regex mismatch if document format changes, missing sections.  

- **Map To Sections**  
  - Type: Set  
  - Role: Combines extracted labels and content arrays into unified section objects, attaching metadata like chapter name.  
  - Config: Uses expressions to map labels and contents by index, extracting numeric section IDs, titles, and chapter data.  
  - Input: From "Extract From Chapter".  
  - Output: Array of section objects to "Sections To List".  

- **Sections To List**  
  - Type: Split Out  
  - Role: Splits the array of sections into individual items for filtering.  
  - Input: Sections array from "Map To Sections".  
  - Output: Individual section items to "Only Valid Sections".  

- **Only Valid Sections**  
  - Type: Filter  
  - Role: Filters out any sections with empty `content`.  
  - Config: Checks `content` field not empty.  
  - Input: Single section items.  
  - Output: Valid sections to "For Each Section...".  

- **For Each Section...**  
  - Type: Split In Batches  
  - Role: Processes valid sections in batches (size 5) for throttling downstream processing.  
  - Input: Valid sections from filter.  
  - Output: To chunking and embedding nodes.  
  - Edge cases: Batch size tuning needed to avoid API rate limits.  

---

#### 1.3 Vector Store Ingestion

**Overview:**  
Chunks large section contents into smaller parts, generates embeddings via Mistral.ai, and inserts these vectors with metadata into Qdrant collection for efficient retrieval.

**Nodes Involved:**  
- Content Chunking @ 50k Chars  
- Split Out Chunks  
- Embeddings Mistral Cloud  
- Default Data Loader  
- Recursive Character Text Splitter  
- Qdrant Vector Store  
- 1sec (Wait node)  

**Node Details:**  

- **Content Chunking @ 50k Chars**  
  - Type: Set  
  - Role: Splits each section content into chunks of up to 30,000 characters (up to ~50k max based on code).  
  - Config: Uses expression creating an array of substrings with chunk size limits.  
  - Input: Section content from "For Each Section...".  
  - Output: Array of content chunks to "Split Out Chunks".  

- **Split Out Chunks**  
  - Type: Split Out  
  - Role: Splits chunk arrays into individual chunk items for embedding.  
  - Input: Chunk array from "Content Chunking @ 50k Chars".  
  - Output: Individual chunk items to "Embeddings Mistral Cloud".  

- **Embeddings Mistral Cloud**  
  - Type: Mistral.ai Embeddings Node  
  - Role: Generates vector embeddings for each text chunk.  
  - Config: Uses Mistral Cloud API credentials.  
  - Input: Text chunk content.  
  - Output: Embeddings to "Default Data Loader".  
  - Edge cases: API rate limits or downtime, malformed input.  

- **Default Data Loader**  
  - Type: Langchain Document Data Loader  
  - Role: Prepares documents with embedded metadata (chapter, section, title, content order) for vector store insertion.  
  - Config: Metadata values are dynamically assigned from the current section item context.  
  - Input: Embedding output and original content.  
  - Output: Document objects to "Qdrant Vector Store".  

- **Qdrant Vector Store**  
  - Type: Langchain VectorStore Qdrant Node  
  - Role: Inserts document vectors into the Qdrant collection named `texas_tax_codes`.  
  - Config: Insert mode, using Qdrant API credentials.  
  - Input: Documents from "Default Data Loader".  
  - Output: To wait node "1sec".  
  - Edge cases: Connectivity issues, insertion errors, metadata mismatch.  

- **1sec**  
  - Type: Wait  
  - Role: Waits 1 second between batches to avoid API rate limits.  
  - Input: From Qdrant insert.  
  - Output: Back to "For Each Section..." for next batch.  

---

#### 1.4 AI Agent Chatbot Setup

**Overview:**  
Defines an AI agent chatbot that interacts with users, maintains conversational state, and uses two custom tools to query the vector store for answers and specific document sections.

**Nodes Involved:**  
- When chat message received  
- Window Buffer Memory1  
- AI Agent  
- Window Buffer Memory  
- OpenAI Chat Model  
- Ask Tool  
- Search Tool  
- Switch  
- Execute Workflow Trigger  

**Node Details:**  

- **When chat message received**  
  - Type: Langchain Chat Trigger  
  - Role: Webhook entry point for messages from users.  
  - Config: Public webhook, loads previous conversation memory.  
  - Output: To "AI Agent".  
  - Edge cases: Webhook accessibility, message format issues.  

- **Window Buffer Memory1**  
  - Type: Langchain Memory Buffer Window  
  - Role: Maintains recent chat history for context.  
  - Input: From chat trigger.  
  - Output: To "When chat message received" (feedback loop).  

- **AI Agent**  
  - Type: Langchain Agent  
  - Role: Central AI agent handling user queries, orchestrating tools, and generating responses.  
  - Config: System message sets role as helpful assistant on Texas tax code; uses OpenAI chat model and window buffer memory.  
  - Inputs: From chat trigger, OpenAI model, tools.  
  - Edge cases: Model API errors, memory overflow, tool misconfigurations.  

- **Window Buffer Memory**  
  - Type: Langchain Memory Buffer Window  
  - Role: Additional memory node connected to AI Agent for context retention.  

- **OpenAI Chat Model**  
  - Type: Langchain OpenAI Chat Model  
  - Role: Provides GPT-based conversational language model for the agent.  
  - Credentials: OpenAI API key configured.  
  - Edge cases: API quota, latency.  

- **Ask Tool**  
  - Type: Langchain Tool Workflow  
  - Role: Handles embedding generation and Qdrant vector similarity search for user questions.  
  - Config: Links to current workflow with route "ask_tool". Description instructs to query tax code as questions.  
  - Input: From AI Agent tool interface.  

- **Search Tool**  
  - Type: Langchain Tool Workflow  
  - Role: Filters and scrolls through Qdrant collection by section or chapter metadata to fetch full text.  
  - Config: Linked workflow, route "search_tool", with JSON schema specifying possible inputs (chapter, section).  
  - Input: From AI Agent tool interface.  

- **Switch**  
  - Type: Switch  
  - Role: Routes incoming workflow trigger requests to either "Ask Tool" or "Search Tool" based on the `route` parameter.  
  - Inputs: From "Execute Workflow Trigger".  
  - Outputs: To "Get Mistral Embeddings" (ask_tool) or "Use Qdrant Scroll API" (search_tool).  
  - Failure modes: Incorrect route values.  

- **Execute Workflow Trigger**  
  - Type: Execute Workflow Trigger  
  - Role: Internal node to invoke the workflow tools based on the route selection.  
  - Input: From AI Agent tool calls.  
  - Output: To "Switch".  

---

#### 1.5 Qdrant API Tools

**Overview:**  
Implements the two Qdrant API calls used by the Ask and Search tools for querying vector similarity and scrolling with metadata filters.

**Nodes Involved:**  
- Get Mistral Embeddings  
- Use Qdrant Search API1  
- Get Ask Response  
- Use Qdrant Scroll API  
- Get Search Response  

**Node Details:**  

- **Get Mistral Embeddings**  
  - Type: HTTP Request  
  - Role: Generates embeddings for AI Agent queries by calling Mistral API (`https://api.mistral.ai/v1/embeddings`).  
  - Config: POST with JSON body specifying model, encoding, and input query.  
  - Credentials: Mistral Cloud API.  
  - Output: Embeddings vector to "Use Qdrant Search API1".  
  - Edge cases: API rate limiting, malformed queries.  

- **Use Qdrant Search API1**  
  - Type: HTTP Request  
  - Role: Performs vector similarity search on Qdrant collection `texas_tax_codes` using the embedding vector.  
  - Config: POST to Qdrant `/points/search` endpoint with limit=4 and `with_payload=true`.  
  - Credentials: Qdrant API.  
  - Output: Search results to "Get Ask Response".  
  - Edge cases: Network errors, invalid embeddings, empty results.  

- **Get Ask Response**  
  - Type: Set  
  - Role: Formats the Qdrant search results into a markdown table summarizing chapter, section, title, and content.  
  - Input: Search results JSON.  
  - Output: Formatted response back to AI Agent.  

- **Use Qdrant Scroll API**  
  - Type: HTTP Request  
  - Role: Uses Qdrant `/points/scroll` endpoint with filters on metadata (chapter or section) to retrieve full document sections.  
  - Config: POST request with limit=100, `with_payload=true`, and filter constructed dynamically based on input query parameters.  
  - Pagination enabled to iterate through results until all points retrieved.  
  - Credentials: Qdrant API.  
  - Output: Results to "Get Search Response".  
  - Edge cases: Pagination failures, filter mismatches, empty results.  

- **Get Search Response**  
  - Type: Set  
  - Role: Formats scrolling results into a markdown block showing chapter, section, title, and concatenated content sorted by content order.  
  - Output: To AI Agent response.  

---

### 3. Summary Table

| Node Name                   | Node Type                                  | Functional Role                                | Input Node(s)                           | Output Node(s)                       | Sticky Note                                                                                             |
|-----------------------------|--------------------------------------------|------------------------------------------------|---------------------------------------|------------------------------------|-------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’| Manual Trigger                             | Manual start of workflow                        | -                                     | Get Tax Code Zip File               | ## Try Me Out! This workflow builds an AI powered Legal assistant who answers questions about tax codes.|
| Get Tax Code Zip File        | HTTP Request                              | Downloads zipped tax code PDFs                  | When clicking ‘Test workflow’          | Extract Zip Files                  | ## Step 1. Download the Tax Code PDF [Read more about handling Zip Files](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.compression/) |
| Extract Zip Files            | Compression                               | Extracts PDFs from zip archive                   | Get Tax Code Zip File                  | Files as Items                    |                                                                                                       |
| Files as Items               | Split Out                                | Splits extracted PDFs into individual items     | Extract Zip Files                      | Extract PDF Contents              |                                                                                                       |
| Extract PDF Contents         | Extract From File                         | Extracts raw text from each PDF                  | Files as Items                        | Extract From Chapter              | ## Step 2. Extract and Partition Into Chapters & Sections [Learn more about reading PDF Files](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.extractfromfile) |
| Extract From Chapter         | Set                                      | Splits raw text into sections using regex       | Extract PDF Contents                  | Map To Sections                  |                                                                                                       |
| Map To Sections             | Set                                      | Combines titles and content into section objects| Extract From Chapter                  | Sections To List                 |                                                                                                       |
| Sections To List             | Split Out                                | Splits array of sections into individual items  | Map To Sections                      | Only Valid Sections              |                                                                                                       |
| Only Valid Sections          | Filter                                   | Filters out sections with empty content          | Sections To List                    | For Each Section...              |                                                                                                       |
| For Each Section...          | Split In Batches                         | Processes sections in batches                     | Only Valid Sections                 | Content Chunking @ 50k Chars / Content Chunking @ 50k Chars (error output) | ## Step 3. Save into Qdrant VectorStore [Read more about using the Qdrant Vectorstore](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.vectorstoreqdrant) |
| Content Chunking @ 50k Chars | Set                                      | Splits section content into manageable chunks   | For Each Section... (error output)  | Split Out Chunks                |                                                                                                       |
| Split Out Chunks            | Split Out                                | Splits content chunks into individual items      | Content Chunking @ 50k Chars         | Embeddings Mistral Cloud        |                                                                                                       |
| Embeddings Mistral Cloud    | Mistral.ai Embeddings Node               | Generates embeddings for chunks                   | Split Out Chunks                    | Default Data Loader             |                                                                                                       |
| Default Data Loader          | Langchain Document Data Loader           | Prepares documents with metadata for vector store| Embeddings Mistral Cloud             | Qdrant Vector Store             |                                                                                                       |
| Qdrant Vector Store          | Langchain VectorStore Qdrant Node        | Inserts embeddings and metadata into Qdrant      | Default Data Loader                 | 1sec                          |                                                                                                       |
| 1sec                        | Wait                                     | Adds delay between batches to avoid rate limits  | Qdrant Vector Store                 | For Each Section...              |                                                                                                       |
| When chat message received   | Langchain Chat Trigger                   | Webhook entry point for user chat messages       | -                                 | AI Agent                      | ## Step 4. Build a Tax Code Assistant ChatBot [Learn more about using AI Agents in n8n](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent) |
| Window Buffer Memory1        | Langchain Memory Buffer Window           | Maintains chat session state                       | When chat message received          | When chat message received       |                                                                                                       |
| AI Agent                    | Langchain Agent                          | Orchestrates chatbot response and tool calls     | When chat message received, OpenAI Chat Model, Window Buffer Memory | Execute Workflow Trigger       |                                                                                                       |
| Window Buffer Memory         | Langchain Memory Buffer Window           | Provides conversational context to AI Agent      | AI Agent                          | AI Agent                      |                                                                                                       |
| OpenAI Chat Model            | Langchain OpenAI Chat Model              | Provides GPT conversational model                 | AI Agent                          | AI Agent                      |                                                                                                       |
| Execute Workflow Trigger     | Execute Workflow Trigger                 | Triggers sub-workflows for tool routing           | AI Agent                          | Switch                        |                                                                                                       |
| Switch                      | Switch                                   | Routes to Ask or Search tool workflows             | Execute Workflow Trigger            | Get Mistral Embeddings / Use Qdrant Scroll API | ## Step 5. Use Qdrant API as Tools [Learn more about using AI Agents in n8n](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent) |
| Get Mistral Embeddings       | HTTP Request                            | Generates embeddings for user query                | Switch (ask_tool)                  | Use Qdrant Search API1           |                                                                                                       |
| Use Qdrant Search API1       | HTTP Request                            | Vector similarity search in Qdrant collection      | Get Mistral Embeddings            | Get Ask Response                |                                                                                                       |
| Get Ask Response             | Set                                      | Formats Qdrant search results into markdown table | Use Qdrant Search API1            | AI Agent                      |                                                                                                       |
| Use Qdrant Scroll API        | HTTP Request                            | Filters and scrolls Qdrant collection by metadata  | Switch (search_tool)               | Get Search Response            |                                                                                                       |
| Get Search Response          | Set                                      | Formats Qdrant scroll results into markdown block | Use Qdrant Scroll API             | AI Agent                      |                                                                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: Entry point to start the workflow manually.

2. **Create HTTP Request Node to Download Zip File**  
   - Name: "Get Tax Code Zip File"  
   - URL: `https://statutes.capitol.texas.gov/Docs/Zips/TX.pdf.zip`  
   - Response Format: File (binary)  
   - Connect from Manual Trigger.

3. **Create Compression Node to Extract Zip**  
   - Mode: Unzip  
   - Input: From HTTP Request node.

4. **Create Split Out Node to Split Extracted Files**  
   - Field to Split Out: `$binary`  
   - Input: From Compression node.

5. **Create Extract From File Node to Extract PDF Text**  
   - Operation: PDF  
   - Binary Property Name: Dynamic (`file_{{$itemIndex}}`)  
   - Input: From Split Out node.

6. **Create Set Node "Extract From Chapter"**  
   - Assign arrays for `contents` and `labels` by parsing the extracted text using JavaScript regex expressions to isolate sections.  
   - Input: From PDF extraction node.

7. **Create Set Node "Map To Sections"**  
   - Combine labels and contents into section objects with metadata: chapter title (from PDF info), section label, title, and content fields.  
   - Input: From "Extract From Chapter".

8. **Create Split Out Node "Sections To List"**  
   - Field to Split Out: `section`  
   - Input: From "Map To Sections".

9. **Create Filter Node "Only Valid Sections"**  
   - Condition: Keep items where `content` is not empty.  
   - Input: From "Sections To List".

10. **Create Split In Batches Node "For Each Section..."**  
    - Batch Size: 5 (to throttle downstream processing)  
    - Input: From Filter node.

11. **Create Set Node "Content Chunking @ 50k Chars"**  
    - Split section content into chunks of max 30,000 characters using expressions creating arrays of substrings.  
    - Input: From batch node error branch.

12. **Create Split Out Node "Split Out Chunks"**  
    - Field to Split Out: `content`  
    - Input: From chunking node.

13. **Create Mistral.ai Embeddings Node**  
    - Credentials: Mistral Cloud API credentials.  
    - Input: Text chunk content.  
    - Output: Embeddings vectors.

14. **Create Langchain Default Data Loader Node**  
    - Assign metadata fields: `chapter`, `section`, `title`, `content_order` from current section context.  
    - Input: Embeddings node output.

15. **Create Qdrant Vector Store Node**  
    - Mode: Insert  
    - Collection: `texas_tax_codes`  
    - Credentials: Qdrant API.  
    - Input: From Data Loader node.

16. **Create Wait Node "1sec"**  
    - Duration: 1 second  
    - Input: From Qdrant insert node.  
    - Output: Connect back to "For Each Section..." to process next batch.

17. **Create Langchain Chat Trigger Node**  
    - Public: True  
    - Load Previous Session: Memory enabled  
    - Input: External user messages.

18. **Create Window Buffer Memory Node** (two instances)  
    - One connected to chat trigger, one connected to AI Agent for conversation context.

19. **Create OpenAI Chat Model Node**  
    - Credentials: OpenAI API account.

20. **Create Langchain Agent Node**  
    - System Message: "You are a helpful assistant answering user questions on the tax code legislation for the state of Texas, United States of America. Along with your response also note in which chapter and section number the information was found."  
    - Connect OpenAI model and memory nodes as inputs.

21. **Create Execute Workflow Trigger Node**  
    - Input: From AI Agent, to route tool calls.

22. **Create Switch Node**  
    - Route condition: Based on `route` JSON parameter equals `ask_tool` or `search_tool`.  
    - Outputs: To respective tool workflows.

23. **Create HTTP Request Node "Get Mistral Embeddings" for Ask Tool**  
    - URL: `https://api.mistral.ai/v1/embeddings`  
    - Method: POST  
    - Body: JSON with model "mistral-embed", `encoding_format` "float", and `input` as the user query.  
    - Credentials: Mistral Cloud API.  
    - Input: From Switch node when route is "ask_tool".

24. **Create HTTP Request Node "Use Qdrant Search API1"**  
    - URL: `http://qdrant:6333/collections/texas_tax_codes/points/search`  
    - Method: POST  
    - Body: JSON with `limit:4`, `vector` embedding from previous node, `with_payload:true`.  
    - Credentials: Qdrant API.  
    - Input: From Mistral Embeddings node.

25. **Create Set Node "Get Ask Response"**  
    - Format search results into markdown table with chapter, section, title, and content.  
    - Input: From Qdrant Search API node.

26. **Create HTTP Request Node "Use Qdrant Scroll API" for Search Tool**  
    - URL: `http://qdrant:6333/collections/texas_tax_codes/points/scroll`  
    - Method: POST  
    - Body: JSON with `limit: 100`, `with_payload: true`, and dynamic filter JSON based on passed `chapter` or `section` metadata keys.  
    - Pagination enabled using `next_page_offset`.  
    - Credentials: Qdrant API.  
    - Input: From Switch node when route is "search_tool".

27. **Create Set Node "Get Search Response"**  
    - Format scroll results into markdown block with chapter, section, title, and concatenated content sorted by content order.  
    - Input: From Qdrant Scroll API node.

28. **Connect "Ask Tool" and "Search Tool" Langchain Tool Workflow Nodes**  
    - Both referencing current workflow ID with routes `ask_tool` and `search_tool` respectively.  
    - Inputs to AI Agent's tool interface.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                | Context or Link                                                                                                                                                                                   |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| This workflow demonstrates a strategic approach to ingesting tax code documents preserving chapters and sections for better chatbot results.                               | Workflow overview and design rationale.                                                                                                                                                          |
| Using metadata filtering in Qdrant enhances scoped queries and improves result relevance.                                                                                    | Qdrant filtering capabilities in vector stores.                                                                                                                                                  |
| Mistral.ai is used here for embeddings and AI models; alternative providers must match Qdrant vector dimension and distance settings.                                       | Model compatibility note.                                                                                                                                                                        |
| Consider returning actual PDF pages or links in chatbot responses for enhanced user trust and verification.                                                                  | Customising this workflow section.                                                                                                                                                               |
| Step 1: Download tax code zip archive and unzip with n8n Compression node.                                                                                                  | https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.compression/                                                                                                                  |
| Step 2: Extract and partition tax code PDFs into chapters and sections using n8n Extract From File node and regex parsing.                                                   | https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.extractfromfile                                                                                                               |
| Step 3: Ingest data into Qdrant vector store using Langchain integration, applying metadata for filtering, and throttle to comply with Mistral.ai rate limits.               | https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.vectorstoreqdrant                                                                                            |
| Step 4: Build AI Agent chatbot using Langchain Agent node, OpenAI GPT model, and memory nodes to maintain conversational context.                                          | https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent                                                                                                       |
| Step 5: Implement two Qdrant API tools (search and scroll) for flexible querying capabilities integrated as Langchain tool workflows.                                      | https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent                                                                                                       |
| Join the n8n community on [Discord](https://discord.com/invite/XPKeKXeB7d) or [Forum](https://community.n8n.io/) for support and discussion.                                 | Community Help and Support                                                                                                                                                                       |

---

This structured analysis and instructions enable advanced users or AI agents to fully understand, reproduce, and extend the Tax Code Assistant workflow with awareness of integration specifics, potential failure modes, and best practices.