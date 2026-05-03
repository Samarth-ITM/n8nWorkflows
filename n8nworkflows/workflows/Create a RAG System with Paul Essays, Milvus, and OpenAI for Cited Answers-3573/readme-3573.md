Create a RAG System with Paul Essays, Milvus, and OpenAI for Cited Answers

https://n8nworkflows.xyz/workflows/create-a-rag-system-with-paul-essays--milvus--and-openai-for-cited-answers-3573


# Create a RAG System with Paul Essays, Milvus, and OpenAI for Cited Answers

### 1. Workflow Overview

This workflow creates a Retrieval-Augmented Generation (RAG) system designed to answer user questions based on the content of Paul Graham’s essays, with cited sources. It leverages a vector database (Milvus) to store and retrieve document embeddings created via OpenAI embeddings, and OpenAI language models to generate clear, citation-backed answers.

The workflow is logically divided into two main blocks:

**1.1 Data Collection and Processing**  
- Purpose: Scrape Paul Graham essays from his website, extract content, split it into manageable chunks, generate vector embeddings via OpenAI, and load these vectors into a Milvus collection for efficient retrieval.
- Includes HTML scraping, content extraction, splitting, embedding, and vector store insertion.

**1.2 Retrieval and Response Generation**  
- Purpose: Handle incoming chat queries, retrieve relevant document chunks from Milvus based on vector similarity, prepare the contextual data, prompt OpenAI’s language models to answer the query citing sources, then format and return the final response.
- Includes chat trigger, embedding user query, retrieval from Milvus, chunk preparation, answer generation, and citation composition.

---

### 2. Block-by-Block Analysis

#### 2.1 Data Collection and Processing

**Overview:**  
This block scrapes the list of essays by Paul Graham, fetches individual essay content, extracts text, splits into chunks, converts chunks into vector embeddings, and loads them into the Milvus vector store.

**Nodes Involved:**  
- When clicking "Execute Workflow"  
- Fetch Essay List  
- Extract essay names  
- Split out into items  
- Limit to first 3  
- Fetch essay texts  
- Extract Text Only  
- Recursive Character Text Splitter1  
- Default Data Loader  
- Embeddings OpenAI  
- Milvus Vector Store  
- Sticky Note, Sticky Note3, Sticky Note5 (contextual comments)

**Node Details:**

- **When clicking "Execute Workflow"**  
  - Type: Manual Trigger  
  - Role: Starts the scraping and ingestion process  
  - Inputs: None  
  - Outputs: Triggers "Fetch Essay List"  
  - Edge Cases: Manual start required; no errors expected in trigger

- **Fetch Essay List**  
  - Type: HTTP Request  
  - Role: Retrieve Paul Graham article index page (`http://www.paulgraham.com/articles.html`)  
  - Config: Simple HTTP GET request, no special options  
  - Inputs: Trigger from manual node  
  - Outputs: HTML content of essay list page  
  - Edge Cases: HTTP failures (timeouts, 404), site changes breaking selector

- **Extract essay names**  
  - Type: HTML extraction  
  - Role: Parse the essay list HTML to extract all links (`href` attributes) of essays using CSS selector `table table a`  
  - Config: Extract attribute "href" for multiple links, returns array of URLs (e.g., `["essay1.html", "essay2.html", ...]`)  
  - Inputs: HTML from "Fetch Essay List"  
  - Outputs: JSON array of essay URLs  
  - Edge Cases: Selector failure if site structure changes; empty results

- **Split out into items**  
  - Type: Split Out  
  - Role: Split the array of essay URLs into individual items for further processing  
  - Inputs: Array of essay URLs  
  - Outputs: Each URL as a separate item  
  - Edge Cases: Empty input array produces no output

- **Limit to first 3**  
  - Type: Limit  
  - Role: Restrict processing to only first 3 essays (configurable) to avoid overload or speed up testing  
  - Inputs: Single essay URL items  
  - Outputs: Max 3 items passed through  
  - Edge Cases: Less than 3 items input passes all

- **Fetch essay texts**  
  - Type: HTTP Request  
  - Role: Retrieve the full HTML content of each essay page (`http://www.paulgraham.com/{{ $json.essay }}`) by appending the URL fragment from split items  
  - Config: Dynamic URL formed via expression  
  - Inputs: Each essay URL  
  - Outputs: HTML content of each essay page  
  - Edge Cases: HTTP errors (404, timeouts), URL broken or misformed

- **Extract Text Only**  
  - Type: HTML extraction  
  - Role: Extract all text content from the essay page HTML, targeting `<body>`, ignoring images and navigation (`skipSelectors`: `img,nav`)  
  - Inputs: HTML from each essay page  
  - Outputs: Extracted raw text content string in `data` field  
  - Edge Cases: Incomplete extraction if HTML changes; non-text elements may be included accidentally

- **Recursive Character Text Splitter1**  
  - Type: Text Splitter (Recursive Character Splitter)  
  - Role: Split large extracted text blobs into smaller chunks of max 6000 characters, to fit Milvus vector size limits and OpenAI embedding input constraints  
  - Inputs: Raw essay text from previous node in JSON data  
  - Outputs: Array of smaller text chunk documents  
  - Edge Cases: Excessive chunk size or empty data leading to failures

- **Default Data Loader**  
  - Type: Document Data Loader  
  - Role: Prepare the text chunks for vector storage by creating LangChain document objects from raw JSON text data  
  - Inputs: Text chunks from the splitter  
  - Outputs: Standardized document format for embedding and database insertion  
  - Edge Cases: Incorrect JSON path or malformed input leading to errors

- **Embeddings OpenAI**  
  - Type: Embeddings node using OpenAI (`text-embedding-ada-002`)  
  - Role: Compute vector embeddings of the document chunks using OpenAI Embeddings API  
  - Inputs: Documents from Data Loader  
  - Outputs: Vectors aligned with document chunks  
  - Configuration: Uses OpenAI account credentials  
  - Edge Cases: API error, rate limits, auth failures

- **Milvus Vector Store**  
  - Type: Vector store (Milvus) insertion node  
  - Role: Insert newly generated embeddings and document metadata into Milvus collection `"my_collection"`  
  - Configuration:  
    - Insert mode: `insert`  
    - Option: Clears existing collection before inserting (fresh load)  
    - Uses Milvus credentials  
  - Inputs: Embeddings and documents from OpenAI node  
  - Outputs: Completion status, vector store updated  
  - Edge Cases: Connection failure to Milvus, insertion errors, credential issues

- **Sticky Notes (contextual)**  
  - Provide guidance on workflow steps such as scraping essays, loading into Milvus, and setup instructions including links to Milvus server setup and collection creation.

---

#### 2.2 Retrieval and Response Generation

**Overview:**  
This block handles live chat queries by embedding the user question, retrieving relevant document chunks from Milvus based on semantic similarity, preparing them as context, generating an AI response that cites sources, and formatting the final answer.

**Nodes Involved:**  
- When chat message received  
- Set max chunks to send to model  
- Milvus Vector Store in retrieval  
- Prepare chunks  
- Answer the query based on chunks  
- Compose citations  
- Generate response  
- Embeddings OpenAI2  
- OpenAI Chat Model  
- Sticky Note1, Sticky Note2 (contextual comments)

**Node Details:**

- **When chat message received**  
  - Type: Chat Trigger (LangChain)  
  - Role: Listens for inbound chat messages to initiate retrieval and response generation  
  - Inputs: HTTP/Webhook chat inputs containing user text  
  - Outputs: Triggers vector search  
  - Edge Cases: Missing or malformed chat input, webhook errors

- **Set max chunks to send to model**  
  - Type: Set node  
  - Role: Sets parameter `chunks=4`, controlling how many top similar chunks to retrieve from Milvus and send to OpenAI for answering  
  - Inputs: Trigger from chat trigger  
  - Outputs: Adds `chunks` field to context data  
  - Edge Cases: Too low number limits context; too high risks exceeding token limits

- **Embeddings OpenAI2**  
  - Type: OpenAI Embeddings  
  - Role: Embed the received user query text before similarity search  
  - Configuration: Model `text-embedding-ada-002` with OpenAI credentials  
  - Inputs: User chat input  
  - Outputs: Query vector embedding for Milvus similarity search  
  - Edge Cases: API or auth errors

- **Milvus Vector Store in retrieval**  
  - Type: Vector store (Milvus) retrieval node  
  - Role: Retrieve top K (here top 2 by config) document chunks similar to the user's embedded query from `"my_collection"`  
  - Configuration: Load mode; Milvus credentials; topK=2  
  - Inputs: Query vector from embeddings node and chunks limit from Set max chunks node  
  - Outputs: Relevant document chunks with metadata  
  - Edge Cases: Connection failures, empty results

- **Prepare chunks**  
  - Type: Code node  
  - Role: Concatenate the retrieved chunk texts into a single formatted string with chunk indexes (`--- CHUNK i ---`) as context for language model prompt  
  - Inputs: Retrieved chunks from Milvus node  
  - Outputs: Single `context` field with formatted text for LLM input  
  - Edge Cases: Empty chunk list leads to empty context

- **OpenAI Chat Model**  
  - Type: Language model (OpenAI chat)  
  - Role: Receives prepared context and user question, generates an answer that includes citations (references indexes of chunks used)  
  - Configuration: Model `gpt-4o-mini`, OpenAI credentials  
  - Inputs: Prompt includes context plus question extracted from webhook node  
  - Outputs: Chat response with embedded citations in the form of chunk indices  
  - Edge Cases: API errors, over token limits

- **Answer the query based on chunks**  
  - Type: Information Extractor (LangChain)  
  - Role: Using system prompt to instruct LLM to answer based on provided context, producing JSON outputs: `answer` string and `citations` array (indexes of chunks used)  
  - Inputs: Chat model outputs  
  - Outputs: JSON structured answer and citations  
  - Edge Cases: Unexpected LLM output format; missing fields

- **Compose citations**  
  - Type: Set node  
  - Role: Transform the `citations` array of chunk indexes into an array of formatted citation strings referencing source file names and line ranges from document metadata of the retrieved chunks  
  - Inputs: Citations indices and original document metadata from Milvus retrieval  
  - Outputs: String array of human readable citations e.g. `[filename, lines X-Y]`  
  - Edge Cases: Mismatched indices, missing metadata

- **Generate response**  
  - Type: Set node  
  - Role: Concatenate the final answer text with citations (if any) separated by new lines, producing the culminating output text sent back to user  
  - Inputs: `answer` and composed citation strings  
  - Outputs: Final response text with citations inline  
  - Edge Cases: No citations or empty answer conditional handled

- **Sticky Notes (contextual)**  
  - Contain summary instructions about Step 2 chat interface and citation generation.

---

### 3. Summary Table

| Node Name                    | Node Type                                | Functional Role                          | Input Node(s)                       | Output Node(s)                         | Sticky Note                                                          |
|------------------------------|----------------------------------------|----------------------------------------|------------------------------------|---------------------------------------|----------------------------------------------------------------------|
| When clicking "Execute Workflow" | Manual Trigger                         | Initiate essay scraping and ingestion  | None                               | Fetch Essay List                      | ## Step 1<br>1. Set up Milvus server and collection;<br>2. Click workflow to scrape/load essays                          |
| Fetch Essay List             | HTTP Request                           | Load Paul Graham essay index page      | When clicking "Execute Workflow"   | Extract essay names                   | ## Scrape latest Paul Graham essays                                 |
| Extract essay names          | HTML Extraction                       | Extract essay links from index page    | Fetch Essay List                   | Split out into items                  | ## Scrape latest Paul Graham essays                                 |
| Split out into items         | Split Out                             | Split essay URLs into individual items | Extract essay names                | Limit to first 3                     | ## Scrape latest Paul Graham essays                                 |
| Limit to first 3             | Limit                                | Limit essays processed to first 3      | Split out into items               | Fetch essay texts                   | ## Scrape latest Paul Graham essays                                 |
| Fetch essay texts            | HTTP Request                         | Download each essay’s HTML page         | Limit to first 3                  | Extract Text Only                    | ## Scrape latest Paul Graham essays                                 |
| Extract Text Only            | HTML Extraction                     | Extract textual content from HTML body | Fetch essay texts                 | Milvus Vector Store                  | ## Load into Milvus vector store                                   |
| Recursive Character Text Splitter1 | Text Splitter (Recursive Character Splitter) | Split long text into chunked documents | Milvus Vector Store (on insert)    | Default Data Loader                  | ## Load into Milvus vector store                                   |
| Default Data Loader          | Document Loader                      | Prepare chunks for embeddings/vector storage | Recursive Character Text Splitter1 | Embeddings OpenAI                   | ## Load into Milvus vector store                                   |
| Embeddings OpenAI            | OpenAI Embeddings                   | Generate vector embeddings from chunks | Default Data Loader               | Milvus Vector Store                  | ## Load into Milvus vector store                                   |
| Milvus Vector Store          | Vector Store (Milvus) Insert        | Insert vectors and metadata to Milvus  | Embeddings OpenAI                | None                               | ## Load into Milvus vector store                                   |
| When chat message received   | Chat Trigger (LangChain)             | Trigger for chat queries                | None                             | Set max chunks to send to model      | ## Step 2<br>Chat and get citations in response                   |
| Set max chunks to send to model | Set                                 | Define max number of chunks to query    | When chat message received        | Milvus Vector Store in retrieval     | ## Step 2<br>Chat and get citations in response                   |
| Embeddings OpenAI2           | OpenAI Embeddings                   | Embed user query for similarity search | When chat message received        | Milvus Vector Store in retrieval     | ## Step 2<br>Chat and get citations in response                   |
| Milvus Vector Store in retrieval | Vector Store (Milvus) Load          | Retrieve top similar document chunks    | Set max chunks to send to model, Embeddings OpenAI2 | Prepare chunks                      | ## Step 2<br>Chat and get citations in response                   |
| Prepare chunks              | Code                                | Format retrieved chunks as concatenated context string | Milvus Vector Store in retrieval  | Answer the query based on chunks      | ## Step 2<br>Chat and get citations in response                   |
| OpenAI Chat Model           | OpenAI Chat Model                   | Generate answer with citations          | Prepare chunks                   | Answer the query based on chunks      | ## Step 2<br>Chat and get citations in response                   |
| Answer the query based on chunks | Information Extractor (LangChain)  | Extract answer string and citation indexes | OpenAI Chat Model                | Compose citations                   | ## Step 2<br>Chat and get citations in response                   |
| Compose citations           | Set                                 | Create human-readable citations from indexes | Answer the query based on chunks | Generate response                   | ## Step 2<br>Chat and get citations in response                   |
| Generate response           | Set                                 | Combine answer text and citations for final output | Compose citations               | None                               | ## Step 2<br>Chat and get citations in response                   |
| Sticky Note3                | Sticky Note                         | Comment for scraping block              | None                             | None                               | ## Scrape latest Paul Graham essays                               |
| Sticky Note5                | Sticky Note                         | Comment for loading into Milvus vector store | None                             | None                               | ## Load into Milvus vector store                                 |
| Sticky Note1                | Sticky Note                         | Comment for chat and citation generation | None                             | None                               | ## Step 2<br>Chat and get citations in response                   |
| Sticky Note2                | Sticky Note                         | Comment summarizing retrieval step      | None                             | None                               | ## Step 2<br>Chat and get citations in response                   |
| Sticky Note                 | Sticky Note                         | Setup instructions for Milvus and workflow usage | None                             | None                               | ## Step 1 Setup instructions with Milvus link                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**
   - Type: Manual Trigger  
   - Purpose: Entry point to initiate scraping and ingestion.

2. **Add HTTP Request node "Fetch Essay List"**
   - Type: HTTP Request  
   - URL: `http://www.paulgraham.com/articles.html`  
   - Method: GET (default).

3. **Add HTML Extraction node "Extract essay names"**
   - Type: HTML  
   - Operation: Extract HTML content  
   - Extraction Values:  
     - Key: `essay`  
     - Attribute: `href`  
     - CSS Selector: `table table a`  
     - Return Array: true.

4. **Add Split Out node "Split out into items"**
   - Field to Split Out: `essay` array extracted above.

5. **Add Limit node "Limit to first 3"**
   - Max Items: 3 (limit processing to first 3 essays).

6. **Add HTTP Request node "Fetch essay texts"**
   - URL: Use expression `http://www.paulgraham.com/{{ $json.essay }}`  
   - Method: GET.

7. **Add HTML Extraction node "Extract Text Only"**
   - Operation: Extract HTML content  
   - Extraction Values: CSS selector: `body`  
   - Skip Selectors: `img,nav` (to exclude images and navigation).

8. **Add Text Splitter node "Recursive Character Text Splitter1"**
   - Type: Recursive Character Splitter  
   - Chunk Size: 6000 characters.

9. **Add Document Default Data Loader node "Default Data Loader"**
   - Data Mode: Expression Data  
   - JSON Data: Expression referencing `$('Extract Text Only').item.json.data`.

10. **Add OpenAI Embeddings node "Embeddings OpenAI"**
    - Model: `text-embedding-ada-002`  
    - Credentials: configure with your OpenAI credentials.

11. **Add Milvus Vector Store node "Milvus Vector Store" (Insert mode)**
    - Mode: Insert  
    - Options: Clear collection before insert = true  
    - Collection: Select or create `my_collection` in Milvus  
    - Credentials: configure with your Milvus API credentials.

12. **Connect nodes for data collection flow:**
    - Connect `"When clicking \"Execute Workflow\""` → `"Fetch Essay List"` → `"Extract essay names"` → `"Split out into items"` → `"Limit to first 3"` → `"Fetch essay texts"` → `"Extract Text Only"` → `"Recursive Character Text Splitter1"` → `"Default Data Loader"` → `"Embeddings OpenAI"` → `"Milvus Vector Store"`.

13. **Create Chat Trigger node "When chat message received"**
    - Type: Chat Trigger (LangChain)  
    - Configure webhook for receiving chat inputs.

14. **Create Set node "Set max chunks to send to model"**
    - Assign Field: `chunks` = 4  
    - Connect from chat trigger.

15. **Create OpenAI Embeddings node "Embeddings OpenAI2"**
    - Model: `text-embedding-ada-002`  
    - Credentials: OpenAI credentials  
    - Input: Chat input text from the trigger.

16. **Create Milvus Vector Store node "Milvus Vector Store in retrieval"**
    - Mode: Load  
    - topK: 2 (or your desired number of results)  
    - Collection: `my_collection`  
    - Credentials: Milvus API credentials.

17. **Create Code node "Prepare chunks"**
    - JavaScript code: Concatenate retrieved chunk texts with headers for context.

18. **Create OpenAI Chat Model node "OpenAI Chat Model"**
    - Model: `gpt-4o-mini` (or preferred chat model)  
    - Credentials: OpenAI credentials.

19. **Create Information Extractor node "Answer the query based on chunks"**
    - Schema: expects JSON with fields `answer` (string) and `citations` (array of numbers)  
    - Prompt Template: instruct answer with citations, do not guess unknowns.

20. **Create Set node "Compose citations"**
    - Expression to map citation indices into formatted citation strings referencing metadata such as filename and line ranges from documents retrieved.

21. **Create Set node "Generate response"**
    - Final text concatenation combining answer and citation lines.

22. **Connect nodes for response flow:**
    - `"When chat message received"` → `"Set max chunks to send to model"` → `"Milvus Vector Store in retrieval"` → `"Prepare chunks"` → `"OpenAI Chat Model"` → `"Answer the query based on chunks"` → `"Compose citations"` → `"Generate response"`.

23. **Optional: Add Sticky Notes for process clarification and documentation in canvas.**

24. **Setup Credentials:**
    - OpenAI API key for all OpenAI nodes  
    - Milvus API credentials for vector store nodes  
    - Webhook setup for chat trigger node

25. **Milvus prerequisites:**
    - Deploy Milvus server via official Docker or Docker Compose guide  
    - Create collection named `my_collection` with appropriate schema.

26. **Testing:**
    - Run manual trigger to scrape and insert essays  
    - Send chat queries to webhook URL to receive cited answers

---

### 5. General Notes & Resources

| Note Content                                                                                                                 | Context or Link                                                                          |
|------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------|
| Setup Milvus server using [Milvus standalone docker-compose guide](https://milvus.io/docs/install_standalone-docker-compose.md) | Instructions for deploying Milvus vector database                                        |
| Paul Graham essays are scraped from [https://paulgraham.com/articles.html](https://paulgraham.com/articles.html)              | Source domain for document corpus                                                        |
| OpenAI embeddings use the `text-embedding-ada-002` model for vectorization                                                   | OpenAI official embeddings guide: https://platform.openai.com/docs/guides/embeddings     |
| The retrieval process uses vector similarity search with top-k nearest neighbors in Milvus                                    | Milvus documentation: https://milvus.io/docs                                            |
| Chat response generation uses a GPT-4 based chat model with citation indices extracted                                       | Ensures answers are backed by document chunks with line references                        |
| Citations generated link chunks to source files and line ranges using metadata originating from source documents             | Improves answer transparency and trustworthiness                                         |