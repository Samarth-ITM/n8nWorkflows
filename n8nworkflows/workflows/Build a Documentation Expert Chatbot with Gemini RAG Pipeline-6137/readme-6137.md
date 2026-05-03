Build a Documentation Expert Chatbot with Gemini RAG Pipeline

https://n8nworkflows.xyz/workflows/build-a-documentation-expert-chatbot-with-gemini-rag-pipeline-6137


# Build a Documentation Expert Chatbot with Gemini RAG Pipeline

### 1. Workflow Overview

This workflow is designed to build a specialized AI chatbot expert on the official n8n documentation, leveraging a Retrieval-Augmented Generation (RAG) pipeline powered by Google Gemini AI models. It enables users to query a dynamically built knowledge base extracted from the entire n8n docs, ensuring that answers are accurate and strictly sourced from official documentation content.

The workflow consists of two main logical blocks:

**1.1 Knowledge Base Construction (Indexing Phase)**  
- Crawls the n8n documentation website to extract all relevant documentation page URLs.  
- Fetches each documentation page content, cleans and splits it into manageable chunks.  
- Creates vector embeddings of these chunks using Google Gemini embeddings.  
- Stores these embeddings in an in-memory vector store for fast retrieval.

**1.2 Chatbot Interaction (Query Phase)**  
- Provides a public-facing chat interface where users submit questions.  
- Converts user queries into embeddings and retrieves the most relevant document chunks from the vector store.  
- Uses a Gemini language model agent to generate precise, context-based answers strictly from the retrieved documentation data.  
- Maintains short-term conversational memory to handle follow-up questions coherently.

---

### 2. Block-by-Block Analysis

#### 2.1 Knowledge Base Construction (Indexing Phase)

**Overview:**  
This block indexes the entire n8n documentation into an in-memory vector database. It runs once per n8n session and processes the documentation pages sequentially to manage memory efficiently.

**Nodes Involved:**  
- Start Indexing (Manual Trigger)  
- Get All n8n Documentation Links (HTTP Request)  
- Extract Links from HTML (HTML Extract)  
- Split Out Links (Split Out)  
- Remove Duplicate Links (Remove Duplicates)  
- Only Keep Doc Paths (Filter)  
- Loop Over Documentation Pages (Split In Batches)  
- Add Documentation Page to Vector Store (Execute Workflow - Sub-workflow)  

**Sub-Workflow:**  
- Ingest Web Page  
- Get Documentation Page (HTTP Request)  
- Extract Documentation Content (HTML Extract)  
- Clean Documentation (Set)  
- Remove Duplicate Documentation Content (Remove Duplicates)  
- Default Data Loader (Document Data Loader)  
- Recursive Character Text Splitter (Text Splitter)  
- Gemini Chunk Embedding (Google Gemini Embeddings)  
- Simple Vector Store (Vector Store Insert)

---

**Node Details:**

- **Start Indexing**  
  - Type: Manual Trigger  
  - Role: Entry point to start indexing process manually.  
  - Connections: Outputs to "Get All n8n Documentation Links".  
  - Failure: None expected, manual start.

- **Get All n8n Documentation Links**  
  - Type: HTTP Request  
  - Role: Downloads the main n8n documentation homepage HTML.  
  - Config: GET https://docs.n8n.io/  
  - Output: Raw HTML of documentation home.  
  - Failure: Possible network errors or site downtime.

- **Extract Links from HTML**  
  - Type: HTML Extract  
  - Role: Extracts all `<a>` tags' href attributes from the homepage.  
  - Config: CSS selector `a`, returns array of href attributes in field "links".  
  - Failure: Invalid HTML or empty page may cause empty output.

- **Split Out Links**  
  - Type: Split Out  
  - Role: Creates individual items for each extracted link to process separately.  
  - Config: Field to split out - "links", output field named "link".  
  - Failure: Empty input leads to no output.

- **Remove Duplicate Links**  
  - Type: Remove Duplicates  
  - Role: Removes duplicate URLs to avoid redundant processing.  
  - Config: Deduplication on "link" field, compares selected fields.  
  - Failure: Unlikely; empty input leads to empty output.

- **Only Keep Doc Paths**  
  - Type: Filter  
  - Role: Filters out links that do not end with '/' or start with 'https://' to keep only relative documentation paths.  
  - Config: Conditions:  
    - link ends with '/'  
    - link does not start with 'https://'  
  - Failure: Misconfigured filter could exclude valid docs.

- **Loop Over Documentation Pages**  
  - Type: Split In Batches  
  - Role: Processes documentation pages in batches of 10 for efficient memory usage.  
  - Config: Batch size = 10.  
  - Output: Main output to "Add Documentation Page to Vector Store" as sub-workflow calls.  
  - Failure: Large batch size could cause memory issues.

- **Add Documentation Page to Vector Store**  
  - Type: Execute Workflow (Sub-workflow)  
  - Role: For each URL, triggers sub-workflow to fetch, process, embed, and store page content.  
  - Config: Passes current link as parameter "path" to sub-workflow.  
  - Failure: Sub-workflow errors forwarded; memory or HTTP errors possible.

---

**Sub-Workflow: Page Content Ingestion**

- **Ingest Web Page**  
  - Type: Execute Workflow Trigger  
  - Role: Entry point for sub-workflow, accepts "path" parameter.  
  - Failure: Invalid or missing path input.

- **Get Documentation Page**  
  - Type: HTTP Request  
  - Role: Downloads HTML content of the specific documentation page.  
  - Config: URL constructed as `https://docs.n8n.io/{{ $json.path }}`.  
  - OnError: Continue on error to avoid halt on single page failure.  
  - Failure: 404 or network errors possible.

- **Extract Documentation Content**  
  - Type: HTML Extract  
  - Role: Extracts main textual content from `<article>`; excludes images, footer, forms.  
  - Failure: HTML structure changes may break extraction.

- **Clean Documentation**  
  - Type: Set  
  - Role: Cleans extracted text by fixing markdown heading formatting and removing URLs.  
  - Expression: Uses regex replacements on `$json.data`.  
  - Failure: Expression errors if input missing.

- **Remove Duplicate Documentation Content**  
  - Type: Remove Duplicates  
  - Role: Prevents re-processing of identical documentation content across workflow executions.  
  - Config: Deduplication based on "documentation" field; stores history of size 10,000.  
  - Failure: History accumulation could impact performance if very large.

- **Default Data Loader**  
  - Type: Document Data Loader  
  - Role: Loads the cleaned documentation string as a document for vector embedding.  
  - Config: Reads from `$json.documentation`; text splitting mode set to "custom".  
  - Failure: Invalid document format or empty content.

- **Recursive Character Text Splitter**  
  - Type: Text Splitter  
  - Role: Splits document text into chunks of 1500 characters with 200 characters overlap.  
  - Config: Split code set to "markdown" for semantic chunking.  
  - Failure: Edge cases on unusual markdown formatting.

- **Gemini Chunk Embedding**  
  - Type: Google Gemini Embeddings  
  - Role: Converts text chunks into vector embeddings.  
  - Credentials: Uses Google Palm API credentials.  
  - Failure: API key errors, rate limits, or network issues.

- **Simple Vector Store**  
  - Type: In-Memory Vector Store Insert  
  - Role: Stores text chunks and embeddings in memory under key "n8n_documentation_vector_store".  
  - Config: Batch size 30 for embedding inserts.  
  - Failure: Memory overflow if too large; data loss on instance restart.

---

#### 2.2 Chatbot Interaction (Query Phase)

**Overview:**  
This block implements the live chatbot interface that users interact with. It converts user queries into embeddings, retrieves relevant documentation chunks from the vector store, and uses a Gemini-based language model agent to generate accurate, context-aware answers strictly sourced from the knowledge base.

**Nodes Involved:**  
- RAG Chatbot (Chat Trigger)  
- Simple Memory (Memory Buffer Window)  
- Gemini Query Embedding (Google Gemini Embeddings)  
- Official n8n Documentation (In-Memory Vector Store Retrieve)  
- n8n Docs AI Agent (Langchain Agent)  
- Gemini 2.5 Flash (Google Gemini Chat Model)  

---

**Node Details:**

- **RAG Chatbot**  
  - Type: Chat Trigger  
  - Role: Public-facing chatbot interface that accepts user messages and triggers the RAG pipeline.  
  - Config: Public webhook enabled; custom CSS for glass/glow theme; placeholder text guides user to ask n8n-related questions.  
  - Failure: Webhook errors, network issues.

- **Simple Memory**  
  - Type: Memory Buffer Window (Langchain)  
  - Role: Stores recent conversation messages to provide context for follow-up questions.  
  - Config: Default settings; linked to AI agent memory input.  
  - Failure: Memory overflow unlikely; resets on workflow restart.

- **Gemini Query Embedding**  
  - Type: Google Gemini Embeddings  
  - Role: Converts user query text into vector embeddings to perform similarity search.  
  - Credentials: Uses Google Palm API credentials.  
  - Failure: API errors, invalid credentials, rate limits.

- **Official n8n Documentation**  
  - Type: In-Memory Vector Store Retrieve  
  - Role: Retrieves the top 10 most relevant documentation chunks based on query embedding similarity.  
  - Config: Memory key matches indexing vector store; tool description provided for agent use.  
  - Failure: Empty or stale vector store leads to poor retrieval.

- **n8n Docs AI Agent**  
  - Type: Langchain Agent  
  - Role: Orchestrates the AI response generation process. Receives system prompt enforcing strict factual answers from docs only.  
  - Config: System message defines expert assistant role and response format (Markdown, clarity, professional tone).  
  - Inputs: Receives user query, retrieved docs, and memory context.  
  - Linked to Gemini 2.5 Flash as language model.  
  - Failure: LLM errors, API failures, prompt misconfiguration.

- **Gemini 2.5 Flash**  
  - Type: Google Gemini Chat Model  
  - Role: Generates final natural language response based on user input and retrieved document context.  
  - Credentials: Google Palm API.  
  - Config: Temperature set to 0 for deterministic responses.  
  - Failure: API or credential errors, network issues.

---

### 3. Summary Table

| Node Name                         | Node Type                                | Functional Role                             | Input Node(s)                       | Output Node(s)                         | Sticky Note                                                                                           |
|----------------------------------|-----------------------------------------|---------------------------------------------|-----------------------------------|---------------------------------------|-----------------------------------------------------------------------------------------------------|
| Start Indexing                   | Manual Trigger                          | Entry point to start indexing                |                                   | Get All n8n Documentation Links       | © 2025 Lucas Peyrin                                                                                   |
| Get All n8n Documentation Links  | HTTP Request                           | Fetch main docs homepage HTML                 | Start Indexing                    | Extract Links from HTML                | © 2025 Lucas Peyrin                                                                                   |
| Extract Links from HTML          | HTML Extract                          | Extract all links `<a href>`                  | Get All n8n Documentation Links  | Split Out Links                       | © 2025 Lucas Peyrin                                                                                   |
| Split Out Links                 | Split Out                            | Split links array into individual items       | Extract Links from HTML           | Remove Duplicate Links                | © 2025 Lucas Peyrin                                                                                   |
| Remove Duplicate Links          | Remove Duplicates                     | Remove duplicate links                         | Split Out Links                  | Only Keep Doc Paths                   | © 2025 Lucas Peyrin                                                                                   |
| Only Keep Doc Paths             | Filter                               | Filter to keep only relative documentation paths | Remove Duplicate Links           | Loop Over Documentation Pages         | © 2025 Lucas Peyrin                                                                                   |
| Loop Over Documentation Pages   | Split In Batches                     | Batch process doc pages for memory efficiency | Only Keep Doc Paths              | Add Documentation Page to Vector Store (main) / Loop (second output) | © 2025 Lucas Peyrin                                                                                   |
| Add Documentation Page to Vector Store | Execute Workflow                 | Sub-workflow call for page ingestion          | Loop Over Documentation Pages    | Loop Over Documentation Pages         | © 2025 Lucas Peyrin                                                                                   |
| Ingest Web Page                | Execute Workflow Trigger              | Sub-workflow entry point with "path" parameter | Add Documentation Page to Vector Store | Get Documentation Page               | © 2025 Lucas Peyrin                                                                                   |
| Get Documentation Page         | HTTP Request                         | Fetch HTML content of a single doc page       | Ingest Web Page                  | Extract Documentation Content         | © 2025 Lucas Peyrin                                                                                   |
| Extract Documentation Content   | HTML Extract                        | Extract main article text from page            | Get Documentation Page           | Clean Documentation                   | © 2025 Lucas Peyrin                                                                                   |
| Clean Documentation            | Set                                 | Clean and format extracted documentation text | Extract Documentation Content    | Remove Duplicate Documentation Content | © 2025 Lucas Peyrin                                                                                   |
| Remove Duplicate Documentation Content | Remove Duplicates             | Avoid re-processing identical content across runs | Clean Documentation             | Default Data Loader                   | © 2025 Lucas Peyrin                                                                                   |
| Default Data Loader            | Document Data Loader                  | Load cleaned docs as documents for splitting  | Remove Duplicate Documentation Content | Recursive Character Text Splitter    | © 2025 Lucas Peyrin                                                                                   |
| Recursive Character Text Splitter | Text Splitter                     | Split docs into chunks suitable for embedding  | Default Data Loader              | Gemini Chunk Embedding                | © 2025 Lucas Peyrin                                                                                   |
| Gemini Chunk Embedding         | Google Gemini Embeddings              | Embed text chunks into vectors                  | Recursive Character Text Splitter | Simple Vector Store                   | © 2025 Lucas Peyrin                                                                                   |
| Simple Vector Store            | Vector Store In-Memory (Insert)       | Store embedded chunks in temporary vector DB   | Gemini Chunk Embedding           | Loop Over Documentation Pages (second output) | © 2025 Lucas Peyrin                                                                                   |
| RAG Chatbot                   | Chat Trigger                         | Public chat interface to receive user queries  |                                   | n8n Docs AI Agent                    | © 2025 Lucas Peyrin                                                                                   |
| Simple Memory                 | Memory Buffer Window (Langchain)      | Store recent conversation context               |                                   | n8n Docs AI Agent (ai_memory)        | © 2025 Lucas Peyrin                                                                                   |
| Gemini Query Embedding        | Google Gemini Embeddings               | Embed user query for retrieval                    | RAG Chatbot                     | Official n8n Documentation            | © 2025 Lucas Peyrin                                                                                   |
| Official n8n Documentation    | Vector Store In-Memory (Retrieve)     | Retrieve top relevant document chunks            | Gemini Query Embedding           | n8n Docs AI Agent (ai_tool)           | © 2025 Lucas Peyrin                                                                                   |
| n8n Docs AI Agent             | Langchain Agent                       | Orchestrates retrieval and generation of answers | RAG Chatbot, Simple Memory, Official n8n Documentation, Gemini 2.5 Flash | RAG Chatbot (response)                | © 2025 Lucas Peyrin                                                                                   |
| Gemini 2.5 Flash              | Google Gemini Chat Model              | Large Language Model generating final answers   | n8n Docs AI Agent (ai_languageModel) | n8n Docs AI Agent                    | © 2025 Lucas Peyrin                                                                                   |

---

### 4. Reproducing the Workflow from Scratch

**Step 1: Setup Credentials**  
1. Obtain a Google AI API key with access to Gemini models.  
2. In n8n, create a new credential of type "Google Palm API" and paste your API key.

---

**Step 2: Build the Knowledge Base (Indexing Flow)**

1. Create a **Manual Trigger** node named `Start Indexing`.  
2. Add an **HTTP Request** node named `Get All n8n Documentation Links` configured to GET `https://docs.n8n.io/`, connected from `Start Indexing`.  
3. Add an **HTML Extract** node `Extract Links from HTML` to extract all `<a>` tags' href attributes into a field `links`; connect from previous node.  
4. Add a **Split Out** node `Split Out Links` to split the `links` array into individual items under field `link`.  
5. Add a **Remove Duplicates** node `Remove Duplicate Links` configured to deduplicate on the `link` field.  
6. Add a **Filter** node `Only Keep Doc Paths` with conditions: keep only links ending with `/` and NOT starting with `https://`.  
7. Add a **Split In Batches** node `Loop Over Documentation Pages` with batch size 10.  
8. Add an **Execute Workflow** node `Add Documentation Page to Vector Store` calling a sub-workflow (to be defined) passing current `link` as parameter `path`. Connect it to the second output of `Loop Over Documentation Pages` to continue looping.

---

**Step 3: Create Sub-Workflow for Page Ingestion**

1. Create a new workflow with an **Execute Workflow Trigger** node named `Ingest Web Page` accepting an input parameter `path`.  
2. Add an **HTTP Request** node `Get Documentation Page` with URL `https://docs.n8n.io/{{ $json.path }}`, set `On Error` to continue.  
3. Add an **HTML Extract** node `Extract Documentation Content` extracting `<article>` content excluding `img, footer, form` into `data`.  
4. Add a **Set** node `Clean Documentation` to clean the extracted text with an expression:  
   ```js
   {{$json.data.replace(/([^#\n]+)\s*#/g, '# $1').trim().replace(/^\s*https?:\/\/\S+\s*/, '')}}
   ```  
   Store result in `documentation`.  
5. Add a **Remove Duplicates** node `Remove Duplicate Documentation Content` set to remove items seen in *previous executions* based on `documentation` field with history size 10,000.  
6. Add a **Document Data Loader** node `Default Data Loader` loading content from `documentation` with custom text splitting mode.  
7. Add a **Recursive Character Text Splitter** node `Recursive Character Text Splitter` configured with chunk size 1500 chars and overlap 200 chars, split code set to "markdown".  
8. Add a **Google Gemini Embeddings** node `Gemini Chunk Embedding` using your Google Palm API credential.  
9. Add a **In-Memory Vector Store (Insert)** node `Simple Vector Store` set to mode "insert", memory key `n8n_documentation_vector_store`, embedding batch size 30.  
10. Connect nodes in the above order. Finally, connect `Simple Vector Store` back to the sub-workflow's end.

---

**Step 4: Build the Chatbot Interaction Flow**

1. Create a **Chat Trigger** node `RAG Chatbot` with public webhook enabled, custom CSS for style, input placeholder "Type your n8n related question..", and initial greeting message.  
2. Add a **Memory Buffer Window** node `Simple Memory` to store conversation history, connect to AI memory input of agent node.  
3. Add a **Google Gemini Embeddings** node `Gemini Query Embedding` with your Google Palm API credential.  
4. Add an **In-Memory Vector Store (Retrieve)** node `Official n8n Documentation` configured to retrieve top 10 results from memory key `n8n_documentation_vector_store`.  
5. Add a **Langchain Agent** node `n8n Docs AI Agent` with system message instructing to answer only from documentation excerpts, professional tone, output in Markdown. Connect:  
   - `RAG Chatbot` output to agent input.  
   - `Simple Memory` to agent AI memory input.  
   - `Gemini Query Embedding` to vector store embedding input.  
   - `Official n8n Documentation` to agent AI tool input.  
6. Add a **Google Gemini Chat Model** node `Gemini 2.5 Flash` with temperature 0 and your Google Palm API credentials. Connect it as AI language model input for the agent node.  
7. Connect the agent output back to `RAG Chatbot` to send responses.

---

**Step 5: Activate and Use**

- Run the `Start Indexing` manual trigger once per n8n session to build the knowledge base.  
- Ensure all Gemini nodes use the correct Google Palm API credentials.  
- Activate the entire workflow for the chatbot to be live.  
- Use the public URL of the `RAG Chatbot` node to interact with the AI assistant.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Context or Link                                                                                           |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| This workflow is a practical demonstration of RAG (Retrieval-Augmented Generation) using n8n’s built-in tools and Google Gemini AI models. It allows building an AI chatbot that is an expert on a specific documentation set and strictly answers based on that knowledge base without hallucinations.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | General workflow description                                                                              |
| The in-memory vector store is temporary and will be erased if n8n restarts. Re-run the indexing to rebuild the knowledge base.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 | Important operational note                                                                                 |
| Memory management is handled by processing documentation pages one-by-one in a sub-workflow, which clears memory after each run to prevent overload.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | Explanation of sub-workflow design                                                                         |
| The system prompt for the AI agent enforces strict adherence to the documentation as the single source of truth, forbidding hallucination or inclusion of external knowledge.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 | Agent system prompt description                                                                            |
| The workflow includes detailed sticky notes (tutorial-style) explaining each step, from crawling docs to chatbot interaction, along with setup instructions for Google AI credentials and usage tips.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | Embedded documentation in sticky notes                                                                    |
| Feedback and coaching links are provided for users interested in improving their n8n skills or consulting services:  
  - Feedback form: https://api.ia2s.app/form/templates/feedback?template=n8n%20Simple%20RAG  
  - Coaching sessions: https://api.ia2s.app/form/templates/coaching?template=n8n%20Simple%20RAG  
  - Consulting inquiries: https://api.ia2s.app/form/templates/consulting?template=n8n%20Simple%20RAG                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | External feedback and coaching resources                                                                   |

---

**Disclaimer:**  
The provided text and workflow are generated exclusively with n8n automation tools and comply fully with content policies. No illegal or offensive content is included. All handled data is public and lawful.