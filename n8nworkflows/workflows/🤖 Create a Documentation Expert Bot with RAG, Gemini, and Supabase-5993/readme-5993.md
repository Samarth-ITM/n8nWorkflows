🤖 Create a Documentation Expert Bot with RAG, Gemini, and Supabase

https://n8nworkflows.xyz/workflows/---create-a-documentation-expert-bot-with-rag--gemini--and-supabase-5993


# 🤖 Create a Documentation Expert Bot with RAG, Gemini, and Supabase

---

# 🤖 Create a Documentation Expert Bot with RAG, Gemini, and Supabase

---

### 1. Workflow Overview

This workflow implements a Retrieval-Augmented Generation (RAG) chatbot specialized in the official n8n documentation. It consists of two main logical parts:

- **1.1 Knowledge Base Construction (Indexing):**  
  This block scrapes the entire n8n documentation website, extracts and cleans the content, splits it into manageable chunks, converts those chunks into vector embeddings using Google Gemini, and stores them in a Supabase vector database. It is designed to be run once or periodically to build and update the knowledge base.

- **1.2 Chatbot Interaction (Querying):**  
  The live chatbot interface receives user questions, converts them into embeddings, retrieves relevant documents from the Supabase vector store, and composes precise answers based exclusively on the documentation content using a Google Gemini language model with a strict system prompt.

- **1.3 Maintenance and Support:**  
  Auxiliary nodes ensure the Supabase instance remains active and credentials are properly managed. Additional sticky notes provide setup instructions and user guidance.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Knowledge Base Construction (Indexing)

**Overview:**  
This block reads the n8n documentation website, extracts all documentation pages, processes each page individually to avoid memory issues, splits content into chunks, creates embeddings, and stores them in Supabase.

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
- Recursive Character Text Splitter (Text Splitter)  
- Gemini Chunk Embedding (Embedding Generation)  
- Your Supabase Vector Store (Insert Vector)  

**Node Details:**

- **Start Indexing (Manual Trigger)**  
  - Role: Starts the indexing process manually.  
  - Configuration: No parameters; manual trigger to begin scraping.  
  - Outputs: Starts the chain to get all documentation links.

- **Get All n8n Documentation Links (HTTP Request)**  
  - Role: Fetches the main n8n documentation homepage HTML.  
  - Configuration: GET request to `https://docs.n8n.io/`.  
  - Outputs: HTML content for link extraction.

- **Extract Links from HTML (HTML Extract)**  
  - Role: Extracts all `<a>` tag `href` attributes from the main page HTML.  
  - Configuration: CSS selector `a`, returns array of href attributes.  
  - Outputs: List of URLs linking to documentation pages.

- **Split Out Links (Split Out)**  
  - Role: Converts the array of links into individual items for processing.  
  - Configuration: Splits on field `links` to generate individual `link` items.  
  - Outputs: Each item contains a single documentation page URL.

- **Remove Duplicate Links (Remove Duplicates)**  
  - Role: Removes duplicate URLs to avoid redundant processing.  
  - Configuration: Compares on `link` field.  
  - Outputs: Unique documentation page URLs.

- **Only Keep Doc Paths (Filter)**  
  - Role: Filters out URLs that do not point to documentation pages.  
  - Configuration: Keeps links that end with `/` but do not start with `https://`.  
  - Outputs: Cleaned list of documentation page paths.

- **Loop Over Documentation Pages (Split In Batches)**  
  - Role: Processes documentation pages in batches of 10 to limit memory usage.  
  - Configuration: Batch size of 10 items per execution cycle.  
  - Outputs: Each batch triggers the sub-workflow for each page.

- **Add Documentation Page to Vector Store (Execute Workflow)**  
  - Role: Executes the sub-workflow on each documentation page URL to process and index it.  
  - Configuration: Passes `path` parameter (documentation page URL) to sub-workflow inputs; waits for completion before next item.  
  - Outputs: Completion status of sub-workflow for each page.

---

**Sub-Workflow: Page Ingestion and Indexing**

- **Ingest Web Page (Execute Workflow Trigger)**  
  - Role: Entry node for sub-workflow, receives `path` of documentation page.  
  - Configuration: Input parameter `path`.  
  - Outputs: Triggers next node.

- **Get Documentation Page (HTTP Request)**  
  - Role: Fetches the full HTML of the individual documentation page.  
  - Configuration: GET request to `https://docs.n8n.io/{{ $json.path }}`.  
  - Error Handling: Continues on error to avoid halting entire workflow.  
  - Outputs: HTML content for extraction.

- **Extract Documentation Content (HTML Extract)**  
  - Role: Extracts main article text from the documentation page.  
  - Configuration: Extracts using CSS selector `article`, skips images, footers, forms.  
  - Outputs: Raw documentation text content.

- **Clean Documentation (Set)**  
  - Role: Cleans extracted text by adjusting markdown headers and removing URLs.  
  - Configuration: Regex replacements to clean formatting and content.  
  - Outputs: Cleaned documentation text.

- **Remove Duplicate Documentation Content (Remove Duplicates)**  
  - Role: Checks if the exact content has been processed before to avoid duplication.  
  - Configuration: Removes items seen in all previous executions by comparing `documentation` field.  
  - Outputs: Passes unique new content only.

- **Recursive Character Text Splitter (Text Splitter)**  
  - Role: Splits the cleaned documentation text into overlapping chunks of max 1500 characters with 200 characters overlap.  
  - Configuration: Uses markdown splitting code.  
  - Outputs: Array of text chunks.

- **Gemini Chunk Embedding (Embedding Generation)**  
  - Role: Converts each text chunk into a 768-dimensional vector embedding using Google Gemini API.  
  - Configuration: Uses Google Palm API credentials.  
  - Outputs: Vector embeddings corresponding to chunks.

- **Your Supabase Vector Store (Insert Vector)**  
  - Role: Inserts text chunks and embeddings into Supabase documents table for storage and retrieval.  
  - Configuration: Uses Supabase API credentials, batch size of 30 for inserts.  
  - Outputs: Confirmation of storage.

---

**Edge Cases & Potential Failures:**  
- HTTP Request nodes may fail due to network issues or page unavailability; sub-workflow handles errors by continuing.  
- Duplicate removal relies on exact match of documentation text; minor formatting changes might bypass this.  
- Memory usage is managed by batching and sub-workflow isolation, but very large pages could still pose risks.  
- Supabase API connectivity requires valid credentials; failures here block storage/retrieval.  
- Embedding generation requires valid Google Gemini credentials; quota limits or API errors may cause failures.  

---

#### 2.2 Chatbot Interaction (Querying)

**Overview:**  
This block provides a public chat interface where users can ask questions about n8n. The system converts queries into embeddings, retrieves relevant documentation chunks, and generates accurate answers using a strict AI agent configured for fact-based responses.

**Nodes Involved:**  
- RAG Chatbot (Chat Trigger)  
- n8n Docs AI Agent (Agent Node)  
- Gemini 2.5 Flash (Language Model)  
- Gemini Query Embedding (Embedding Generation)  
- Official n8n Documentation (Supabase Vector Store Retrieval)  
- Simple Memory (Memory Buffer Window)  

**Node Details:**

- **RAG Chatbot (Chat Trigger)**  
  - Role: Public-facing chat interface to receive user messages.  
  - Configuration: Publicly accessible webhook, customized CSS for glass/glow theme, placeholder text, and initial greeting message.  
  - Outputs: User messages to the AI agent.

- **n8n Docs AI Agent (Agent Node)**  
  - Role: Orchestrates the query answering process using RAG.  
  - Configuration:  
    - System message with strict instructions to use exclusively the official n8n documentation for answers.  
    - Enforces accuracy, no hallucination, factual tone.  
    - Uses the Gemini 2.5 Flash LLM as language model.  
    - Uses the Official n8n Documentation vector store as a retrieval tool.  
    - Uses Simple Memory for short-term conversation context.  
  - Inputs: User query, conversation history.  
  - Outputs: Final answer message.

- **Gemini 2.5 Flash (Language Model)**  
  - Role: Generates human-readable answers based on retrieved context and query.  
  - Configuration: Temperature set to 0 for deterministic output.  
  - Inputs: Prompt and context from the AI agent.  
  - Outputs: Answer text.

- **Gemini Query Embedding (Embedding Generation)**  
  - Role: Converts user queries into vector embeddings (768 dimensions) to search in vector store.  
  - Configuration: Uses Google Palm API credentials.  
  - Inputs: User question text.  
  - Outputs: Query embeddings.

- **Official n8n Documentation (Supabase Vector Store Retrieval)**  
  - Role: Retrieves the top 10 most relevant document chunks matching query embeddings.  
  - Configuration: Retrieval mode, topK=10, no metadata included.  
  - Inputs: Query embeddings.  
  - Outputs: Relevant documentation chunks.

- **Simple Memory (Memory Buffer Window)**  
  - Role: Stores recent conversation messages (short-term memory).  
  - Configuration: Default buffer window, no additional parameters.  
  - Inputs/Outputs: Conversation history to AI agent.

---

**Edge Cases & Potential Failures:**  
- Chat trigger relies on proper webhook setup; misconfiguration can block access.  
- AI agent must receive valid credentials for Gemini and Supabase; absence or expiry causes failure.  
- Gemini API can have rate limits; overload may delay or block responses.  
- Vector retrieval might return empty results for vague or out-of-scope queries.  
- Memory buffer could grow large if conversation is very long, potentially slowing responses.

---

#### 2.3 Maintenance and Support

**Overview:**  
Auxiliary nodes maintain Supabase connection alive and provide user guidance via sticky notes.

**Nodes Involved:**  
- Every 6 Days (Schedule Trigger)  
- Keep Supabase Instance Alive (Supabase Query)  
- Multiple Sticky Note Nodes  

**Node Details:**

- **Every 6 Days (Schedule Trigger)**  
  - Role: Triggers periodic wake-up calls to keep Supabase active.  
  - Configuration: Interval of 6 days.  
  - Outputs: Triggers Supabase maintenance query.

- **Keep Supabase Instance Alive (Supabase Node)**  
  - Role: Runs a simple query (`getAll` operation on the `documents` table with a limit of 1) to prevent Supabase from going to sleep.  
  - Configuration: Uses Supabase API credentials.  
  - Inputs: Trigger from schedule.  
  - Outputs: Query result (ignored).

- **Sticky Notes**  
  - Role: Provide extensive documentation, setup instructions, best practices, and user guidance directly on the canvas.  
  - Content includes: SQL schema setup, workflow explanation, credential setup steps, coaching & consulting links, and usage tips.

---

### 3. Summary Table

| Node Name                      | Node Type                                   | Functional Role                              | Input Node(s)                 | Output Node(s)               | Sticky Note                                                                                       |
|--------------------------------|---------------------------------------------|----------------------------------------------|------------------------------|-----------------------------|-------------------------------------------------------------------------------------------------|
| Simple Memory                  | Memory Buffer Window (LangChain)            | Stores recent conversation messages          | n8n Docs AI Agent (ai_memory) | n8n Docs AI Agent (ai_memory) |                                                                                                 |
| Official n8n Documentation     | Supabase Vector Store (LangChain)           | Retrieves relevant docs from Supabase        | Gemini Query Embedding (ai_embedding) | n8n Docs AI Agent (ai_tool)   |                                                                                                 |
| Default Data Loader            | Document Data Loader (LangChain)             | Loads text chunks for embedding insertion    | Recursive Character Text Splitter (ai_textSplitter) | Your Supabase Vector Store (ai_document) |                                                                                                 |
| Recursive Character Text Splitter | Text Splitter (LangChain)                  | Splits text into overlapping chunks          | Clean Documentation (main)     | Default Data Loader (ai_textSplitter) |                                                                                                 |
| Remove Duplicate Documentation Content | Remove Duplicates                          | Avoids reprocessing identical documentation  | Clean Documentation (main)     | Your Supabase Vector Store (main) |                                                                                                 |
| Only Keep Doc Paths            | Filter Node                                  | Filters to keep only valid documentation URLs| Remove Duplicate Links (main)  | Loop Over Documentation Pages (main) |                                                                                                 |
| Clean Documentation           | Set Node                                     | Cleans and formats extracted documentation   | Extract Documentation Content (main) | Remove Duplicate Documentation Content (main) |                                                                                                 |
| Get All n8n Documentation Links | HTTP Request                                | Fetches main documentation homepage HTML     | Start Indexing (main)          | Extract Links from HTML (main) |                                                                                                 |
| Extract Links from HTML        | HTML Extract                                 | Extracts all `<a>` tag hrefs from HTML       | Get All n8n Documentation Links (main) | Split Out Links (main)         |                                                                                                 |
| Split Out Links               | Split Out                                    | Splits array of links into individual items  | Extract Links from HTML (main) | Remove Duplicate Links (main)  |                                                                                                 |
| Remove Duplicate Links        | Remove Duplicates                            | Removes duplicate links                       | Split Out Links (main)          | Only Keep Doc Paths (main)     |                                                                                                 |
| Loop Over Documentation Pages | Split In Batches                             | Processes pages in batches                    | Only Keep Doc Paths (main)      | Add Documentation Page to Vector Store (main) |                                                                                                 |
| Add Documentation Page to Vector Store | Execute Workflow                          | Calls sub-workflow to process each page      | Loop Over Documentation Pages (main, batch) | Loop Over Documentation Pages (main, continue) |                                                                                                 |
| Ingest Web Page               | Execute Workflow Trigger                     | Entry for sub-workflow with page `path`      | Add Documentation Page to Vector Store (sub-workflow) | Get Documentation Page (main) |                                                                                                 |
| Get Documentation Page       | HTTP Request                                 | Fetches HTML of a single documentation page  | Ingest Web Page (main)          | Extract Documentation Content (main) |                                                                                                 |
| Extract Documentation Content | HTML Extract                                 | Extracts main article text from page HTML    | Get Documentation Page (main)  | Clean Documentation (main)     |                                                                                                 |
| Remove Duplicate Documentation Content | Remove Duplicates                          | Avoids reprocessing identical page content   | Clean Documentation (main)     | Recursive Character Text Splitter (main) |                                                                                                 |
| Clean Documentation           | Set Node                                     | Cleans text formatting                        | Extract Documentation Content (main) | Remove Duplicate Documentation Content (main) |                                                                                                 |
| Recursive Character Text Splitter | Text Splitter                              | Splits cleaned text into chunks               | Remove Duplicate Documentation Content (main) | Default Data Loader (ai_textSplitter) |                                                                                                 |
| Default Data Loader           | Document Data Loader                         | Loads chunks for embedding generation         | Recursive Character Text Splitter (ai_textSplitter) | Gemini Chunk Embedding (ai_embedding) |                                                                                                 |
| Gemini Chunk Embedding        | Embeddings Generator (Google Gemini)         | Generates vector embeddings for text chunks  | Default Data Loader (ai_embedding) | Your Supabase Vector Store (ai_embedding) |                                                                                                 |
| Your Supabase Vector Store    | Vector Store (Supabase)                      | Stores chunks and embeddings                   | Gemini Chunk Embedding (ai_embedding) | Loop Over Documentation Pages (main, continue) |                                                                                                 |
| Gemini 2.5 Flash              | Language Model (Google Gemini)                | Generates answer from retrieved context       | n8n Docs AI Agent (ai_languageModel) | n8n Docs AI Agent (main)       |                                                                                                 |
| Gemini Query Embedding        | Embeddings Generator (Google Gemini)         | Generates query embedding for retrieval       | n8n Docs AI Agent (ai_embedding) | Official n8n Documentation (ai_embedding) |                                                                                                 |
| n8n Docs AI Agent             | Agent Node (LangChain)                        | Orchestrates retrieval and answer generation  | RAG Chatbot (main), Simple Memory (ai_memory), Official n8n Documentation (ai_tool), Gemini 2.5 Flash (ai_languageModel) | RAG Chatbot (main)             |                                                                                                 |
| RAG Chatbot                  | Chat Trigger                                  | Public chat interface for user questions       | External user input              | n8n Docs AI Agent (main)       |                                                                                                 |
| Every 6 Days                 | Schedule Trigger                              | Periodic trigger to keep Supabase awake        | —                              | Keep Supabase Instance Alive (main) |                                                                                                 |
| Keep Supabase Instance Alive | Supabase Query                                | Queries Supabase to prevent sleep               | Every 6 Days (main)              | —                             |                                                                                                 |
| Sticky Notes                 | Sticky Note                                   | Various explanations, instructions, and setup | —                              | —                             | Multiple sticky notes contain detailed setup, usage instructions, SQL schema, and links.         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Supabase Project and Prepare Database:**
   - Sign up at [supabase.com](https://supabase.com/).  
   - Create a new project.  
   - Run the provided SQL script in the Supabase SQL Editor to create the `documents` table and `match_documents` function. Adjust vector dimension to 768 for Gemini embeddings or 1536 for OpenAI embeddings.  
   - Enable the `pgvector` extension if not already enabled.

2. **Create Credentials in n8n:**
   - Create a Supabase API credential with your Supabase Project URL and Service Role Key.  
   - Create a Google Palm API credential with your Gemini API key.

3. **Build Knowledge Base Indexing Flow:**

   - Add a **Manual Trigger** node named `Start Indexing`.  
   - Add an **HTTP Request** node `Get All n8n Documentation Links`, configured to GET `https://docs.n8n.io/`. Connect from `Start Indexing`.  
   - Add an **HTML Extract** node `Extract Links from HTML`, set to extract all `<a>` tags' `href` attributes from the response, output as array `links`. Connect from `Get All n8n Documentation Links`.  
   - Add a **Split Out** node `Split Out Links` to split the `links` array into individual `link` items. Connect from `Extract Links from HTML`.  
   - Add a **Remove Duplicates** node `Remove Duplicate Links` on the `link` field to remove duplicates. Connect from `Split Out Links`.  
   - Add a **Filter** node `Only Keep Doc Paths` with conditions: `link` ends with `/` AND does NOT start with `https://`. Connect from `Remove Duplicate Links`.  
   - Add a **Split In Batches** node `Loop Over Documentation Pages` with batch size 10. Connect from `Only Keep Doc Paths`.

4. **Create Sub-Workflow for Page Processing:**

   - Create a new workflow with an **Execute Workflow Trigger** node named `Ingest Web Page` accepting input parameter `path`.  
   - Add an **HTTP Request** node `Get Documentation Page` to GET `https://docs.n8n.io/{{ $json.path }}`. Connect from `Ingest Web Page`. Set error handling to "Continue on Error".  
   - Add an **HTML Extract** node `Extract Documentation Content` with selector `article`, skipping `img`, `footer`, and `form`. Connect from `Get Documentation Page`.  
   - Add a **Set** node `Clean Documentation` with regex replacements to fix markdown headers and remove URLs from the extracted `data`. Connect from `Extract Documentation Content`.  
   - Add a **Remove Duplicates** node `Remove Duplicate Documentation Content` removing items seen in previous executions, comparing on `documentation` field. Connect from `Clean Documentation`.  
   - Add a **Recursive Character Text Splitter** node with chunk size 1500, overlap 200, split code set to "markdown". Connect from `Remove Duplicate Documentation Content`.  
   - Add a **Default Data Loader** node to prepare chunks for embedding. Connect from `Recursive Character Text Splitter`.  
   - Add a **Gemini Chunk Embedding** node using Google Palm API credentials to generate embeddings for chunks. Connect from `Default Data Loader`.  
   - Add a **Supabase Vector Store** node `Your Supabase Vector Store` configured in insert mode, batch size 30, connected to your Supabase credentials to store chunks and embeddings. Connect from `Gemini Chunk Embedding`.

5. **Connect Sub-Workflow in Main Flow:**

   - Add an **Execute Workflow** node `Add Documentation Page to Vector Store` in the main workflow. Configure it to call the sub-workflow with the input parameter `path` set to `={{ $json.link }}`. Connect from `Loop Over Documentation Pages`. Set to wait for sub-workflow completion before next batch.

6. **Build Chatbot Interaction Flow:**

   - Add a **Chat Trigger** node `RAG Chatbot` configured as public webhook with custom CSS for glass/glow theme and initial greeting message.  
   - Add an **Agent** node `n8n Docs AI Agent` configured with:  
     - System message enforcing strict use of official documentation only.  
     - Tools:  
       - `Official n8n Documentation` (Supabase Vector Store retrieval node) with retrieval mode, topK=10, using Supabase credentials.  
       - `Gemini Query Embedding` node generating query embeddings with Google Palm API credentials.  
     - Language Model: `Gemini 2.5 Flash` node with temperature 0, Google Palm API credentials.  
     - Memory: `Simple Memory` node for conversation context.  
   - Connect `RAG Chatbot` to `n8n Docs AI Agent`. Connect `n8n Docs AI Agent` back to `RAG Chatbot` output.

7. **Add Maintenance Nodes:**

   - Add a **Schedule Trigger** node `Every 6 Days` set to trigger every 6 days.  
   - Add a **Supabase** query node `Keep Supabase Instance Alive` performing a simple `getAll` query with limit 1 on `documents` table, using Supabase credentials. Connect from `Every 6 Days`.

8. **Add Sticky Notes:**

   - Add sticky notes on the canvas with the provided detailed instructions, SQL code, setup steps, coaching links, and explanations for user clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                  | Context or Link                                                                                             |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| Enable the `pgvector` extension in Supabase to work with embedding vectors.                                                                                                   | SQL setup note in sticky note on canvas.                                                                   |
| Use Gemini embeddings dimension 768; if using OpenAI embeddings, adjust vector dimension to 1536 in SQL schema.                                                              | SQL setup note.                                                                                            |
| The use of a sub-workflow for page processing is critical for memory management and stability when processing many pages.                                                     | Sticky note explaining sub-workflow benefits.                                                             |
| Free Supabase projects sleep after inactivity; periodic query every 6 days keeps the DB awake.                                                                                | Maintenance sticky note.                                                                                    |
| The RAG chatbot strictly uses retrieved documentation to answer, never hallucinating or inventing info.                                                                        | System message in AI agent node; sticky notes describing RAG principles.                                   |
| Public chat interface includes custom "glass & glow" theme styling for improved UI experience.                                                                                 | CSS in the Chat Trigger node.                                                                               |
| Feedback and consulting forms are provided via links in sticky notes for user support and coaching services.                                                                  | Links to forms in sticky notes for feedback and coaching.                                                  |
| The workflow is designed to be executed first by running the indexing (Part 1), then activating the chatbot (Part 2).                                                         | Workflow usage instructions in sticky notes.                                                              |
| The workflow is built with n8n version supporting LangChain nodes v1.3+ and n8n core nodes v4+. Credentials for Google Palm API (Gemini) and Supabase API are essential.      | Credentials setup steps.                                                                                    |
| The knowledge base update is incremental: duplicate detection avoids reprocessing identical pages, allowing safe repeated runs to update documentation over time.            | Duplicate removal node explanation.                                                                        |

---

**Disclaimer:** The text provided originates solely from an automated workflow created with n8n, a workflow automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly available.

---