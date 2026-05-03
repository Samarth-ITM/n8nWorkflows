Build an n8n Documentation Expert Chatbot with OpenAI RAG Pipeline

https://n8nworkflows.xyz/workflows/build-an-n8n-documentation-expert-chatbot-with-openai-rag-pipeline-6281


# Build an n8n Documentation Expert Chatbot with OpenAI RAG Pipeline

### 1. Workflow Overview

This workflow builds a Retrieval-Augmented Generation (RAG) chatbot specialized in the official n8n documentation. It consists of two main parts:

- **1.1 Knowledge Base Construction (Indexing flow):**  
  This block scrapes the entire n8n documentation website, extracts and cleans the text content of each documentation page, splits it into manageable chunks, converts those chunks into embeddings (vectors), and stores them in an in-memory vector store. This indexing process needs to run once per n8n session and can be re-run to update the knowledge base.

- **1.2 Chatbot Interaction (Chat flow):**  
  This block provides a public chat interface where users can ask questions related to n8n. The system converts the user query into an embedding, retrieves the most relevant documentation chunks from the vector store, and uses a specialized AI agent to generate factual, precise answers strictly based on the documentation content.

Logical blocks and their node groupings:

- **Block 1: Knowledge Base Construction**  
  - Input: Manual Trigger (`Start Indexing`)  
  - Web scraping and link extraction (`Get All n8n Documentation Links`, `Extract Links from HTML`, `Split Out Links`, `Remove Duplicate Links`, `Only Keep Doc Paths`)  
  - Loop and Sub-Workflow invocation (`Loop Over Documentation Pages`, `Add Documentation Page to Vector Store`)  
  - Sub-Workflow nodes: (`Ingest Web Page`, `Get Documentation Page`, `Extract Documentation Content`, `Clean Documentation`, `Remove Duplicate Documentation Content`, `Default Data Loader`, `Recursive Character Text Splitter`, `Embeddings OpenAI`, `Simple Vector Store`)  

- **Block 2: Chatbot Interaction**  
  - Chat trigger (`RAG Chatbot`)  
  - Memory buffer (`Simple Memory`)  
  - AI agent orchestration (`n8n Docs AI Agent`)  
  - Query embedding (`Embeddings OpenAI1`)  
  - Vector store retrieval (`Official n8n Documentation`)  
  - Language model for answer generation (`OpenAI Chat Model`)  

---

### 2. Block-by-Block Analysis

---

#### Block 1: Knowledge Base Construction

**Overview:**  
This block builds the knowledge base by crawling the n8n documentation website, extracting each page's content, processing it into text chunks, embedding those chunks, and storing them in an in-memory vector store for quick retrieval during chat.

**Nodes Involved:**  
- `Start Indexing` (Manual Trigger)  
- `Get All n8n Documentation Links` (HTTP Request)  
- `Extract Links from HTML` (HTML Extract)  
- `Split Out Links` (Split Out)  
- `Remove Duplicate Links` (Remove Duplicates)  
- `Only Keep Doc Paths` (Filter)  
- `Loop Over Documentation Pages` (Split In Batches)  
- `Add Documentation Page to Vector Store` (Execute Workflow)  

**Node Details:**

- **Start Indexing**  
  - Type: Manual Trigger  
  - Role: Initiates the indexing process manually.  
  - Connections: Outputs to `Get All n8n Documentation Links`.  
  - Edge Cases: None.

- **Get All n8n Documentation Links**  
  - Type: HTTP Request  
  - Role: Fetches the HTML of the main n8n documentation homepage (`https://docs.n8n.io/`).  
  - Configuration: Simple GET request to the homepage URL.  
  - Input: Triggered by `Start Indexing`.  
  - Output: HTML content passed to `Extract Links from HTML`.  
  - Failure Modes: Network errors, site unavailability.

- **Extract Links from HTML**  
  - Type: HTML Extract  
  - Role: Extracts all `<a>` tag href values from the fetched homepage HTML.  
  - Configuration: CSS selector `a`, extracting `href` attribute, returning as array.  
  - Input: HTML content from previous node.  
  - Output: Array of links for splitting.  
  - Edge Cases: Empty or malformed HTML, missing links.

- **Split Out Links**  
  - Type: Split Out  
  - Role: Converts array of links into individual items for further processing.  
  - Configuration: Splits field `links` into individual `link` items.  
  - Input: Array of links.  
  - Output: Single link per item.  
  - Edge Cases: Empty input array.

- **Remove Duplicate Links**  
  - Type: Remove Duplicates  
  - Role: Removes duplicate link entries based on the `link` field.  
  - Configuration: Compares selected fields, field used: `link`.  
  - Input: Items with individual links.  
  - Output: Unique link items.  
  - Edge Cases: Duplicate links, empty list.

- **Only Keep Doc Paths**  
  - Type: Filter  
  - Role: Filters links to keep only those that point to documentation paths.  
  - Configuration: Conditions: link ends with `/` AND does not start with `https://`.  
  - Input: Unique links.  
  - Output: Filtered documentation page links.  
  - Edge Cases: Non-doc URLs, external links.

- **Loop Over Documentation Pages**  
  - Type: Split In Batches  
  - Role: Batches the filtered list into groups of 10 links to process.  
  - Configuration: Batch size 10.  
  - Input: Filtered doc paths.  
  - Output: Passes batch to `Add Documentation Page to Vector Store`.  
  - Edge Cases: Handling last batch with fewer items.

- **Add Documentation Page to Vector Store**  
  - Type: Execute Workflow (Sub-workflow)  
  - Role: For each batch item, calls sub-workflow to ingest and process the single documentation page.  
  - Configuration: Passes `link` as `path` parameter to sub-workflow (which is the same workflow, recursive use).  
  - Input: Batch item with `link`.  
  - Output: Returns to continue loop.  
  - Edge Cases: Sub-workflow failures, timeouts.

---

**Sub-Workflow: Processing a Single Documentation Page**

*Triggered by `Add Documentation Page to Vector Store` with parameter `path`*

**Nodes Involved:**  
- `Ingest Web Page` (Execute Workflow Trigger)  
- `Get Documentation Page` (HTTP Request)  
- `Extract Documentation Content` (HTML Extract)  
- `Clean Documentation` (Set)  
- `Remove Duplicate Documentation Content` (Remove Duplicates)  
- `Default Data Loader` (Document Loader)  
- `Recursive Character Text Splitter` (Text Splitter)  
- `Embeddings OpenAI` (Embedding Generator)  
- `Simple Vector Store` (Vector Store Insert)  

**Node Details:**

- **Ingest Web Page**  
  - Type: Execute Workflow Trigger  
  - Role: Entry point for ingesting a single documentation page, receives `path` parameter.  
  - Input: `path` from parent workflow.  
  - Output: Passes `path` to `Get Documentation Page`.  
  - Edge Cases: Missing or malformed `path`.

- **Get Documentation Page**  
  - Type: HTTP Request  
  - Role: Fetches HTML content of a single documentation page by URL constructed as `https://docs.n8n.io/{{path}}`.  
  - Configuration: Continuation on error enabled (`continueErrorOutput`) to avoid stopping on 404 or network errors.  
  - Input: `path` parameter from `Ingest Web Page`.  
  - Output: HTML content to `Extract Documentation Content`.  
  - Edge Cases: 404 Not Found, network errors.

- **Extract Documentation Content**  
  - Type: HTML Extract  
  - Role: Extracts the main article content from the page HTML, excluding images, footers, and forms.  
  - Configuration: CSS selector `article`, skip selectors `img, footer, form`.  
  - Input: HTML from `Get Documentation Page`.  
  - Output: Raw article text to `Clean Documentation`.  
  - Edge Cases: Missing article tag, malformed HTML.

- **Clean Documentation**  
  - Type: Set  
  - Role: Cleans and formats extracted article text by fixing Markdown headers formatting and removing URLs.  
  - Configuration: Uses JavaScript expression to replace misformatted headers and strip leading URLs.  
  - Input: Raw article text.  
  - Output: Cleaned text assigned to `documentation` field.  
  - Edge Cases: Irregular text, regex failures.

- **Remove Duplicate Documentation Content**  
  - Type: Remove Duplicates  
  - Role: Avoids re-processing the same page content if already processed in previous runs.  
  - Configuration: Operation set to remove items seen in previous executions, dedupe on `documentation` field, history size 10000.  
  - Input: Cleaned documentation text.  
  - Output: Continues only if unique content.  
  - Edge Cases: Possible false negatives, history size limits.

- **Default Data Loader**  
  - Type: Document Default Data Loader (Langchain)  
  - Role: Loads the cleaned documentation text into a document format for further processing, using custom text splitting mode.  
  - Configuration: Loads JSON data from `documentation` field, uses custom text splitting.  
  - Input: Unique documentation text.  
  - Output: Document object to `Recursive Character Text Splitter`.  
  - Edge Cases: Empty documents.

- **Recursive Character Text Splitter**  
  - Type: Recursive Character Text Splitter (Langchain)  
  - Role: Splits the document text into overlapping chunks for embedding, chunk size 1500 characters, overlap 200, split code: markdown.  
  - Input: Document from data loader.  
  - Output: Text chunks to `Embeddings OpenAI`.  
  - Edge Cases: Very large texts, splitting failures.

- **Embeddings OpenAI**  
  - Type: OpenAI Embedding Node  
  - Role: Converts each text chunk into vector embeddings using OpenAI API.  
  - Configuration: Uses OpenAI credentials.  
  - Input: Text chunks.  
  - Output: Embeddings to `Simple Vector Store`.  
  - Failure Modes: API rate limits, auth errors.

- **Simple Vector Store**  
  - Type: Vector Store In-Memory (Insert Mode)  
  - Role: Inserts embeddings and original chunks into an in-memory vector store labeled as `n8n_documentation_vector_store`.  
  - Configuration: Batch embedding size 30, memory key set for consistent retrieval.  
  - Input: Embeddings and documents.  
  - Output: Completion signal back to parent workflow.  
  - Edge Cases: Memory limits, data eviction on n8n restart.

---

#### Block 2: Chatbot Interaction

**Overview:**  
This block listens for user chat input, maintains short-term memory of the conversation, converts queries into embeddings, retrieves relevant documentation chunks from the in-memory vector store, and generates precise, factual answers using a strictly constrained AI agent.

**Nodes Involved:**  
- `RAG Chatbot` (Chat Trigger)  
- `Simple Memory` (Memory Buffer)  
- `n8n Docs AI Agent` (Langchain Agent)  
- `Embeddings OpenAI1` (OpenAI Embedding)  
- `Official n8n Documentation` (Vector Store Retrieve)  
- `OpenAI Chat Model` (Language Model Chat)  

**Node Details:**

- **RAG Chatbot**  
  - Type: Langchain Chat Trigger  
  - Role: Public-facing chat interface that triggers the entire RAG retrieval and answer generation process.  
  - Configuration: Public webhook enabled, custom CSS styling for chat UI, placeholder text, initial greeting messages.  
  - Output: User messages to `n8n Docs AI Agent`.  
  - Edge Cases: Network/webhook errors, unauthorized access (public=true allows open access).

- **Simple Memory**  
  - Type: Langchain Memory Buffer Window  
  - Role: Maintains a short-term memory buffer of recent messages to provide context for follow-up questions.  
  - Configuration: Default settings (window size implicit).  
  - Input: Conversation stream from chat.  
  - Output: Memory context to AI agent.  
  - Edge Cases: Memory overflow, conversation reset.

- **n8n Docs AI Agent**  
  - Type: Langchain Agent  
  - Role: The core AI orchestrator that uses system messages to strictly answer based on documentation excerpts without hallucinations.  
  - Configuration: System prompt instructs the agent to only use documentation data, mandates accuracy, no mention of internal process, and formats output professionally.  
  - Inputs: User query, memory context, retrieval tool outputs.  
  - Outputs: Final AI answer to chat response node.  
  - Edge Cases: Model hallucination, API errors.

- **Embeddings OpenAI1**  
  - Type: OpenAI Embeddings  
  - Role: Converts user query into embedding vector for similarity search.  
  - Configuration: Uses OpenAI credentials.  
  - Input: User question text.  
  - Output: Embedding vector to vector store retrieval node.  
  - Edge Cases: API limits, auth errors.

- **Official n8n Documentation**  
  - Type: Vector Store In-Memory (Retrieve Mode)  
  - Role: Searches the in-memory vector store for the top 10 most relevant document chunks matching the query embedding.  
  - Configuration: Retrieve mode as tool, topK=10, memory key matches vector store insert node.  
  - Input: Query embedding.  
  - Output: Relevant document chunks to AI agent as context.  
  - Edge Cases: Empty store (knowledge base not built), memory cleared on restart.

- **OpenAI Chat Model**  
  - Type: Langchain Chat Model (OpenAI)  
  - Role: The language model (GPT 4.1-nano) responsible for generating the final human-readable answer.  
  - Configuration: Model set to `gpt-4.1-nano`, uses OpenAI API credentials.  
  - Input: Contextual prompt from AI agent.  
  - Output: Chatbot answer.  
  - Edge Cases: API errors, model unavailability.

---

### 3. Summary Table

| Node Name                      | Node Type                                  | Functional Role                             | Input Node(s)                       | Output Node(s)                      | Sticky Note                                                                                   |
|--------------------------------|--------------------------------------------|---------------------------------------------|-----------------------------------|-----------------------------------|------------------------------------------------------------------------------------------------|
| Start Indexing                 | Manual Trigger                             | Initiates indexing process                   |                                   | Get All n8n Documentation Links   |                                                                                                |
| Get All n8n Documentation Links| HTTP Request                              | Fetch main doc homepage HTML                 | Start Indexing                    | Extract Links from HTML            | Step 1.1: Find All the 'Books' - visits main n8n docs page                                    |
| Extract Links from HTML        | HTML Extract                              | Extract all href links from homepage         | Get All n8n Documentation Links  | Split Out Links                   | Step 1.2: Read the List of Books - pull out all links                                         |
| Split Out Links               | Split Out                                 | Turn link array into individual items        | Extract Links from HTML           | Remove Duplicate Links            | Step 1.3: Process One Book at a Time - create separate item per link                          |
| Remove Duplicate Links         | Remove Duplicates                         | Remove duplicate links                        | Split Out Links                  | Only Keep Doc Paths               | Step 1.4: Tidy Up the To-Do List - remove duplicates and non-doc links                        |
| Only Keep Doc Paths           | Filter                                    | Keep only links ending with "/" and local    | Remove Duplicate Links           | Loop Over Documentation Pages    |                                                                                                |
| Loop Over Documentation Pages | Split In Batches                          | Batch process doc page links (batch size 10) | Only Keep Doc Paths              | Add Documentation Page to Vector Store | Step 1.5: The Librarian's Reading Loop - batch processing and sub-workflow calls             |
| Add Documentation Page to Vector Store | Execute Workflow                      | Process each doc page via sub-workflow       | Loop Over Documentation Pages    | Loop Over Documentation Pages    |                                                                                                |
| Ingest Web Page                | Execute Workflow Trigger                  | Sub-workflow entry for processing one page   | Add Documentation Page to Vector Store | Get Documentation Page           |                                                                                                |
| Get Documentation Page         | HTTP Request                             | Fetch HTML of a single doc page               | Ingest Web Page                 | Extract Documentation Content    | Step 1.5.1: Read a Single Page - fetches page HTML                                           |
| Extract Documentation Content  | HTML Extract                             | Extract main article content from page HTML  | Get Documentation Page          | Clean Documentation              | Step 1.5.2: Get the Good Stuff - extract main text, ignore menus/footers                      |
| Clean Documentation            | Set                                      | Clean and format extracted text               | Extract Documentation Content   | Remove Duplicate Documentation Content |                                                                                                |
| Remove Duplicate Documentation Content | Remove Duplicates                    | Skip already processed identical page content| Clean Documentation             | Default Data Loader             | Step 1.5.3: Avoid Re-reading - remove duplicates across executions                            |
| Default Data Loader            | Document Default Data Loader (Langchain) | Load cleaned text into document object        | Remove Duplicate Documentation Content | Recursive Character Text Splitter |                                                                                                |
| Recursive Character Text Splitter | Text Splitter (Langchain)              | Split document text into overlapping chunks  | Default Data Loader              | Embeddings OpenAI               | Step 1.5.4: Create & Store 'Magic Index Cards' - chunk text for embeddings                    |
| Embeddings OpenAI             | OpenAI Embeddings                        | Convert chunks to vector embeddings           | Recursive Character Text Splitter | Simple Vector Store             |                                                                                                |
| Simple Vector Store           | Vector Store In-Memory (Insert)          | Store embeddings and text chunks in memory   | Embeddings OpenAI               |                                 |                                                                                                |
| RAG Chatbot                   | Langchain Chat Trigger                   | Public chat interface                         |                                 | n8n Docs AI Agent                | The Front Desk: Ask Your Question Here - public chat UI, start RAG process                    |
| Simple Memory                 | Langchain Memory Buffer Window           | Store recent conversation messages           |                                 | n8n Docs AI Agent                | Short-Term Memory - remembers recent messages                                               |
| n8n Docs AI Agent             | Langchain Agent                          | Orchestrates query answering using docs      | RAG Chatbot, Simple Memory, Official n8n Documentation |                               | The Brains: The Expert Librarian - strict factual answers from docs only                     |
| Embeddings OpenAI1            | OpenAI Embeddings                        | Convert user query to embedding vector        | n8n Docs AI Agent (tool input)  | Official n8n Documentation       | The Tool: The 'Magic Filing Cabinet' Retriever - query embedding                             |
| Official n8n Documentation    | Vector Store In-Memory (Retrieve)        | Retrieve top 10 relevant doc chunks           | Embeddings OpenAI1              | n8n Docs AI Agent                |                                                                                                |
| OpenAI Chat Model             | Langchain Chat Model (OpenAI)             | Generate final human-readable answer          | n8n Docs AI Agent               | RAG Chatbot                     | The 'Synthesizer' & 'Strategist' - LLM creates answer based on retrieved docs                |

---

### 4. Reproducing the Workflow from Scratch

**Step 1: Set Up OpenAI Credentials**  
- Create OpenAI API credential in n8n.  
- Assign this credential to all relevant nodes: `OpenAI Chat Model`, `Embeddings OpenAI`, `Embeddings OpenAI1`.

**Step 2: Create the Knowledge Base Construction Flow**

1. Create a **Manual Trigger** node named `Start Indexing`.  
2. Create an **HTTP Request** node `Get All n8n Documentation Links` configured with URL `https://docs.n8n.io/` (GET). Connect from `Start Indexing`.  
3. Add an **HTML Extract** node `Extract Links from HTML` extracting all `<a>` elements' `href` attributes as an array field `links`. Connect from previous node.  
4. Add a **Split Out** node `Split Out Links` to split the `links` array into individual items with field name `link`. Connect from previous node.  
5. Add a **Remove Duplicates** node `Remove Duplicate Links` to remove duplicates comparing the `link` field. Connect from previous node.  
6. Add a **Filter** node `Only Keep Doc Paths` with conditions:  
   - `link` ends with `/`  
   - `link` does not start with `https://`  
   Connect from previous node.  
7. Add a **Split In Batches** node `Loop Over Documentation Pages` with batch size 10. Connect from previous node.  
8. Add an **Execute Workflow** node `Add Documentation Page to Vector Store` that calls the same workflow as sub-workflow, mapping `link` to `path` parameter. Connect from `Loop Over Documentation Pages`.

**Step 3: Create the Sub-Workflow for Ingesting a Single Page**

1. Create an **Execute Workflow Trigger** node `Ingest Web Page` with input parameter `path`.  
2. Create an **HTTP Request** node `Get Documentation Page` with URL `=https://docs.n8n.io/{{ $json.path }}`, set to continue on error. Connect from trigger.  
3. Create an **HTML Extract** node `Extract Documentation Content` to extract content from CSS selector `article`, skipping `img, footer, form`. Connect from previous node.  
4. Create a **Set** node `Clean Documentation` to assign a new field `documentation` with the cleaned text using expression:  
   ```
   {{ $json.data.replace(/([^#\n]+)\s*#/g, '# $1').trim().replace(/^\s*https?:\/\/\S+\s*/, '') }}
   ```  
   Connect from previous node.  
5. Create a **Remove Duplicates** node `Remove Duplicate Documentation Content` configured to remove duplicates seen in previous executions using the `documentation` field. Connect from previous node.  
6. Add a **Default Data Loader** node (Langchain) `Default Data Loader` configured to load JSON data from `documentation` field with custom text splitting mode. Connect from previous node.  
7. Add a **Recursive Character Text Splitter** node configured with chunk size 1500, overlap 200, split code `markdown`. Connect from previous node.  
8. Add an **Embeddings OpenAI** node with OpenAI credentials to generate embeddings from text chunks. Connect from previous node.  
9. Add a **Simple Vector Store** node (insert mode) configured with memory key `n8n_documentation_vector_store` and embedding batch size 30. Connect from previous node.  

**Step 4: Create the Chatbot Interaction Flow**

1. Add a **Langchain Chat Trigger** node `RAG Chatbot` configured as public webhook with custom CSS and placeholder text.  
2. Add a **Simple Memory** node with default memory buffer settings. Connect output to `n8n Docs AI Agent` in the `ai_memory` input.  
3. Add a **Langchain Agent** node `n8n Docs AI Agent` with system message instructing strict factual answers based solely on official docs, no hallucinations, professional tone. Connect `ai_languageModel` input to `OpenAI Chat Model`, `ai_tool` input to `Official n8n Documentation` node, and `ai_memory` input to `Simple Memory`. Connect output to chatbot response.  
4. Add an **OpenAI Embeddings** node `Embeddings OpenAI1` with credentials to embed user queries. Connect input to agent’s tool input.  
5. Add a **Vector Store In-Memory** node `Official n8n Documentation` in retrieve mode with memory key `n8n_documentation_vector_store`, topK=10. Connect input from embeddings node, output to agent.  
6. Add an **OpenAI Chat Model** node configured with model `gpt-4.1-nano` and OpenAI credentials. Connect output to agent's language model input.

**Step 5: Activate and Test**

- Run the `Start Indexing` manual trigger once to build the knowledge base. This process takes ~15-20 minutes.  
- After indexing completes, activate the entire workflow.  
- Use the public URL from `RAG Chatbot` node or the internal "Open Chat" button to interact with the chatbot.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                    | Context or Link                                                                                                 |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------|
| The in-memory vector store is temporary and will be cleared if n8n restarts. Re-run the indexing flow to rebuild the knowledge base.                                                                                                         | Sticky Note2, Sticky Note28                                                                                     |
| The sub-workflow approach for ingesting individual pages is designed to manage memory efficiently and prevent crashes when processing large numbers of pages.                                                                                 | Sticky Note12                                                                                                   |
| The AI agent is strictly instructed to answer only based on documentation excerpts to avoid hallucination, maintaining factual accuracy and professionalism.                                                                                   | Sticky Note18                                                                                                   |
| The workflow uses OpenAI models (GPT-4.1-nano for chat and embeddings) and requires valid OpenAI API credentials configured in all relevant nodes.                                                                                            | Sticky Note24, Sticky Note27                                                                                     |
| Public chat interface is styled with a custom glass & glow theme for improved UI experience.                                                                                                                                                   | `RAG Chatbot` node parameters                                                                                   |
| Recommended testing queries: "How does the IF node work?" or "What is a sub-workflow?"                                                                                                                                                         | Sticky Note1                                                                                                    |
| Credits: Workflow authored by Lucas Peyrin and copyrighted 2025.                                                                                                                                                                              | Multiple sticky notes                                                                                           |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected material. All handled data is legal and publicly available.