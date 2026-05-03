Build a RAG-based Q&A System with Qdrant, Mistral OCR, and GPT-4

https://n8nworkflows.xyz/workflows/build-a-rag-based-q-a-system-with-qdrant--mistral-ocr--and-gpt-4-8944


# Build a RAG-based Q&A System with Qdrant, Mistral OCR, and GPT-4

### 1. Workflow Overview

This n8n workflow, titled **"Build a RAG-based Q&A System with Qdrant, Mistral OCR, and GPT-4"**, is designed to create and evaluate an AI agent capable of answering questions based on a PDF knowledge base. It integrates document ingestion, OCR processing, embedding generation, vector storage, retrieval with reranking, AI question answering, and evaluation of output correctness.

The workflow is logically divided into the following blocks:

- **1.1 Vector Store Setup & PDF Ingestion**  
  Handles creation and refreshing of the Qdrant vector collection, retrieval of PDFs from Google Drive, and text extraction via Mistral OCR.

- **1.2 Document Embedding & Storage**  
  Processes extracted text, splits it, creates embeddings with OpenAI, and inserts them into Qdrant.

- **1.3 Retrieval-Augmented Generation (RAG) Question Answering**  
  Uses the vector store with a reranker and GPT-4 based AI agent to answer user questions by retrieving relevant information.

- **1.4 Evaluation Workflow**  
  Compares AI answers against ground truth using a detailed evaluation prompt and stores results in Google Sheets.

- **1.5 Workflow Triggers & Utilities**  
  Includes manual and sub-workflow triggers, batching utilities, and helper nodes for controlling flow.

---

### 2. Block-by-Block Analysis

#### 1.1 Vector Store Setup & PDF Ingestion

**Overview:**  
This block sets up the Qdrant vector collection, clears existing data, retrieves PDFs from Google Drive, and extracts text using Mistral OCR.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (manual trigger)  
- Refresh collection (HTTP Request to clear Qdrant)  
- Search PDFs (Google Drive)  
- Loop Over Items1 (split batches for processing)  
- Get File ID (Set node extracts file ID)  
- Call 'Agent Arena' (Executes sub-workflow for each PDF)  
- When Executed by Another Workflow (executes sub-workflow)  
- Get PDF (Google Drive file download)  
- Mistral Upload (upload file to Mistral OCR API)  
- Mistral Signed URL (retrieve temporary file URL from Mistral)  
- Mistral DOC OCR (request OCR processing)  
- Code (processes OCR pages to markdown)

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Entry point for refreshing the Qdrant collection and reindexing PDFs.  
  - Outputs to: Refresh collection.

- **Refresh collection**  
  - Type: HTTP Request  
  - Role: Sends POST to Qdrant to delete all points from the "agentic-arena" collection (clear vector store).  
  - Config: HTTP Header Auth, JSON body with empty filter (delete all).  
  - Potential failures: Network errors, auth issues, Qdrant unavailability.

- **Search PDFs**  
  - Type: Google Drive node  
  - Role: Lists all PDF files in a specified Google Drive folder.  
  - Config: Folder ID set to "Agentic Arena" folder, returns all files.  
  - Potential failures: Auth errors, folder access issues.

- **Loop Over Items1**  
  - Type: SplitInBatches  
  - Role: Processes files one at a time for safe sequential handling.  
  - Outputs to: Get File ID.

- **Get File ID**  
  - Type: Set  
  - Role: Extracts file ID from each file item for downstream usage.  
  - Output variable: file_id.

- **Call 'Agent Arena'**  
  - Type: Execute Workflow (sub-workflow)  
  - Role: Calls the same workflow or a sub-workflow to process each file (PDF ingestion and embedding).  
  - Config: Mode each, waits for sub-workflow completion.

- **When Executed by Another Workflow**  
  - Type: Execute Workflow Trigger  
  - Role: Entry point for the sub-workflow called by Call 'Agent Arena'.  
  - Input: Pass-through.

- **Get PDF**  
  - Type: Google Drive  
  - Role: Downloads PDF file by file_id.  
  - Output: Binary data of the PDF.

- **Mistral Upload**  
  - Type: HTTP Request  
  - Role: Uploads binary PDF file to Mistral API for OCR processing.  
  - Config: multipart/form-data, body includes purpose "ocr" and file data. Uses Mistral Cloud API credentials.  
  - Edge cases: Upload failures, API rate limits, invalid files.

- **Mistral Signed URL**  
  - Type: HTTP Request  
  - Role: Requests a signed URL for the uploaded file to allow OCR processing.  
  - Config: GET with expiry query param for 24 hours.  
  - Edge cases: URL generation failures, auth errors.

- **Mistral DOC OCR**  
  - Type: HTTP Request  
  - Role: Requests OCR processing on the document URL using Mistral "mistral-ocr-latest" model.  
  - Config: JSON body with document URL, requests image base64 inclusion.  
  - Edge cases: OCR processing failures, timeouts.

- **Code**  
  - Type: Code (JavaScript)  
  - Role: Maps the OCR response pages to a simplified JSON array with markdown text for each page.  
  - Input: OCR JSON with pages.  
  - Output: Array of objects with markdown property.

---

#### 1.2 Document Embedding & Storage

**Overview:**  
This block splits the extracted text, generates embeddings via OpenAI, and inserts them into the Qdrant vector database.

**Nodes Involved:**  
- Loop Over Items (split batches)  
- Set page (Set node for text assignment)  
- Character Text Splitter (Text splitter by character with overlap)  
- Default Data Loader (Document loader for LangChain)  
- Embeddings OpenAI (embedding generation)  
- Qdrant Vector Store (insert embeddings)  
- Wait (wait node to manage timing)

**Node Details:**

- **Loop Over Items**  
  - Type: SplitInBatches  
  - Role: Processes document pages one by one.  
  - Outputs to: Set page.

- **Set page**  
  - Type: Set  
  - Role: Assigns markdown text from OCR output to a "text" variable for embedding.  
  - Output: text string.

- **Character Text Splitter**  
  - Type: LangChain Text Splitter  
  - Role: Splits text by character "#" with 100 characters overlap to create chunks for embedding.  
  - Config: separator "#", chunkOverlap 100.

- **Default Data Loader**  
  - Type: LangChain Document Loader  
  - Role: Loads and prepares document chunks with metadata (filename from PDF).  
  - Output: Document objects for embedding.

- **Embeddings OpenAI**  
  - Type: LangChain OpenAI Embeddings  
  - Role: Generates vector embeddings for document chunks using OpenAI API.  
  - Config: No new lines stripped (stripNewLines: false).  
  - Credential: OpenAI API key.

- **Qdrant Vector Store**  
  - Type: LangChain Qdrant Vector Store  
  - Role: Inserts embeddings into the "agentic-arena" collection in Qdrant.  
  - Config: Mode "insert".  
  - Credential: Qdrant API key.

- **Wait**  
  - Type: Wait  
  - Role: Adds delay after insertion to avoid rate limits or overload.  
  - No parameters specified, default wait.

---

#### 1.3 Retrieval-Augmented Generation (RAG) Question Answering

**Overview:**  
This block handles user queries by retrieving relevant documents from Qdrant using embeddings, reranking results with Cohere, and generating answers via GPT-4.

**Nodes Involved:**  
- Filter Empty Rows (filter input questions)  
- AI Agent (LangChain Agent node)  
- Simple Memory (memory buffer for conversation context)  
- RAG (Qdrant vector store for retrieval as tool)  
- Reranker Cohere (reranker for retrieval results)  
- Embeddings OpenAI1 (embeddings for query text)  
- OpenAI Chat Model (GPT-4 based chat model)  
- Respond to Chat (send generated output back)

**Node Details:**

- **Filter Empty Rows**  
  - Type: Filter  
  - Role: Drops any rows where "chatInput" is empty to avoid processing invalid queries.

- **AI Agent**  
  - Type: LangChain Agent  
  - Role: Main AI agent that processes the input question, uses RAG retrieval tool, and formulates answers with policy compliance logic.  
  - Config: System message defines specialized Ministry of Finance assistant with strict instructions about using RAG, no hallucination, professional tone, citations, etc.  
  - Inputs: chatInput text.  
  - Outputs: Answer JSON with detailed response.

- **Simple Memory**  
  - Type: LangChain Memory Buffer Window  
  - Role: Maintains conversational context window for the AI agent.

- **RAG**  
  - Type: LangChain Qdrant Vector Store (retrieval mode)  
  - Role: Retrieves relevant document embeddings from Qdrant based on input query embedding.  
  - Config: Mode "retrieve-as-tool", uses Cohere reranker.  
  - Credentials: Qdrant API.

- **Reranker Cohere**  
  - Type: LangChain Reranker  
  - Role: Uses Cohere API to rerank retrieved documents to improve relevance.  
  - Credential: Cohere API key.

- **Embeddings OpenAI1**  
  - Type: LangChain OpenAI Embeddings  
  - Role: Generates embeddings for the input query text used in retrieval.  
  - Credential: OpenAI API.

- **OpenAI Chat Model**  
  - Type: LangChain Chat OpenAI (GPT-4 model)  
  - Role: Generates final answer text from retrieved context and agent prompt.  
  - Config: Model "gpt-4.1", temperature 0.1 for low randomness.  
  - Credential: OpenAI API.

- **Respond to Chat**  
  - Type: LangChain Chat  
  - Role: Sends generated answer back (e.g., to UI or next step).

---

#### 1.4 Evaluation Workflow

**Overview:**  
This block evaluates the AI agent's answers against ground truth answers stored in Google Sheets, using an expert evaluation prompt and records the results back in the Sheet.

**Nodes Involved:**  
- Eval Set (Evaluation trigger, reads questions and answers)  
- Eval Input (Set node to prepare input for evaluation)  
- Filter Empty Rows (filters input)  
- Only if we are evaluating (conditional evaluation check)  
- Run Evaluation (OpenAI LLM evaluates answer correctness)  
- Save Eval (writes evaluation results to Google Sheet)  
- LLM as a Judge (OpenAI LLM for scoring)

**Node Details:**

- **Eval Set**  
  - Type: Evaluation Trigger  
  - Role: Reads a Google Sheet with evaluation questions and expected answers.  
  - Config: Google Sheets OAuth2 credentials, reads from public "Agentic Arena Evaluation Set" Sheet.

- **Eval Input**  
  - Type: Set  
  - Role: Extracts "chatInput" and generates a unique sessionId for evaluation run.

- **Filter Empty Rows**  
  - Same as in RAG block, ensures valid inputs.

- **Only if we are evaluating**  
  - Type: Evaluation (conditional)  
  - Role: Checks if evaluation mode is active before running scoring.

- **Run Evaluation**  
  - Type: Evaluation  
  - Role: Sends prompt to OpenAI GPT-4 with detailed instructions for scoring answers from 0 to 5 based on factual accuracy and citation correctness.  
  - Config: Complex prompt defining scoring criteria, output JSON format, examples included.  
  - Inputs: AI Agent output and ground truth answer.

- **Save Eval**  
  - Type: Evaluation Node  
  - Role: Saves evaluation results including correctness score and agent answer back to Google Sheet.  
  - Config: Uses Google Sheets OAuth2 credentials.

- **LLM as a Judge**  
  - Type: LangChain OpenAI Chat  
  - Role: Executes the evaluation prompt with GPT-4, outputs JSON score and analysis.

---

#### 1.5 Workflow Triggers & Utilities

**Overview:**  
Supporting nodes for triggering workflows, batching, and notes for user guidance.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (manual trigger)  
- When Executed by Another Workflow (sub-workflow trigger)  
- Loop Over Items / Loop Over Items1 (batch processing)  
- Wait (delay node)  
- Various Sticky Notes (documentation and instructions)

---

### 3. Summary Table

| Node Name                  | Node Type                                  | Functional Role                                | Input Node(s)                    | Output Node(s)                   | Sticky Note                                                                                     |
|----------------------------|--------------------------------------------|-----------------------------------------------|---------------------------------|---------------------------------|------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                            | Entry point to refresh collection and PDF ingestion | None                            | Refresh collection              |                                                                                                |
| Refresh collection          | HTTP Request                              | Clears Qdrant vector store                     | When clicking ‘Execute workflow’ | Search PDFs                    |                                                                                                |
| Search PDFs                | Google Drive                              | Lists PDF files in Drive folder                | Refresh collection              | Loop Over Items1                |                                                                                                |
| Loop Over Items1           | SplitInBatches                            | Processes files one by one                      | Search PDFs                    | Get File ID                    |                                                                                                |
| Get File ID                | Set                                       | Extracts file ID from file item                 | Loop Over Items1               | Call 'Agent Arena'             |                                                                                                |
| Call 'Agent Arena'          | Execute Workflow                          | Executes sub-workflow for PDF processing       | Get File ID                    | Loop Over Items1 (loop cycle)  |                                                                                                |
| When Executed by Another Workflow | Execute Workflow Trigger                 | Sub-workflow entry for PDF processing           | Call 'Agent Arena'             | Get PDF                       |                                                                                                |
| Get PDF                    | Google Drive                              | Downloads PDF binary                            | When Executed by Another Workflow | Mistral Upload                |                                                                                                |
| Mistral Upload             | HTTP Request                              | Uploads PDF to Mistral OCR API                  | Get PDF                       | Mistral Signed URL             |                                                                                                |
| Mistral Signed URL         | HTTP Request                              | Retrieves signed URL for uploaded file          | Mistral Upload                | Mistral DOC OCR               |                                                                                                |
| Mistral DOC OCR            | HTTP Request                              | Requests OCR processing                          | Mistral Signed URL            | Code                         |                                                                                                |
| Code                       | Code                                      | Extracts markdown pages from OCR response       | Mistral DOC OCR               | Loop Over Items               |                                                                                                |
| Loop Over Items            | SplitInBatches                            | Processes pages one by one for embedding        | Code                         | Set page                     |                                                                                                |
| Set page                   | Set                                       | Sets text from markdown for embedding           | Loop Over Items               | Character Text Splitter       |                                                                                                |
| Character Text Splitter    | LangChain Text Splitter                   | Splits text into chunks with overlap            | Set page                     | Default Data Loader           |                                                                                                |
| Default Data Loader        | LangChain Document Loader                 | Prepares documents for embedding                 | Character Text Splitter        | Qdrant Vector Store           |                                                                                                |
| Embeddings OpenAI          | LangChain Embeddings                      | Generates embeddings for document chunks        | Default Data Loader            | Qdrant Vector Store           |                                                                                                |
| Qdrant Vector Store        | LangChain Vector Store                    | Inserts embeddings into Qdrant collection       | Embeddings OpenAI             | Wait                         |                                                                                                |
| Wait                       | Wait                                      | Adds delay to manage request pacing              | Qdrant Vector Store           | Loop Over Items               |                                                                                                |
| Filter Empty Rows          | Filter                                    | Filters out empty chat inputs                     | Eval Input / Eval Set          | AI Agent                     |                                                                                                |
| AI Agent                   | LangChain Agent                           | Processes question using RAG and generates answer | Filter Empty Rows             | Only if we are evaluating     |                                                                                                |
| Simple Memory              | LangChain Memory Buffer                   | Maintains conversation context                    | AI Agent (memory input)        | AI Agent                     |                                                                                                |
| RAG                        | LangChain Vector Store (retrieve)         | Retrieves documents from Qdrant as tool          | Embeddings OpenAI1 / Reranker Cohere | AI Agent                 |                                                                                                |
| Reranker Cohere            | LangChain Reranker                        | Reranks retrieved documents                        | RAG                          | RAG                         |                                                                                                |
| Embeddings OpenAI1         | LangChain Embeddings                      | Embeds query text                                  | AI Agent                     | RAG                         |                                                                                                |
| OpenAI Chat Model          | LangChain Chat OpenAI                     | GPT-4 model generating final answer               | AI Agent                     | AI Agent                     |                                                                                                |
| Only if we are evaluating  | Evaluation                                | Conditional check to run evaluation                | AI Agent                     | Run Evaluation, Respond to Chat |                                                                                                |
| Run Evaluation             | Evaluation                                | Runs GPT-4 evaluation prompt for correctness      | Only if we are evaluating     | Save Eval                    |                                                                                                |
| Save Eval                  | Evaluation                                | Saves evaluation results back to Google Sheets    | Run Evaluation               | None                        |                                                                                                |
| Eval Set                   | Evaluation Trigger                        | Reads evaluation questions and answers from Sheet | None                        | Eval Input                   |                                                                                                |
| Eval Input                 | Set                                       | Prepares evaluation inputs                          | Eval Set                     | Filter Empty Rows            |                                                                                                |
| LLM as a Judge             | LangChain OpenAI Chat                     | Executes evaluation prompt for scoring             | Run Evaluation               | Run Evaluation               |                                                                                                |
| Respond to Chat            | LangChain Chat                           | Sends AI response back                              | Only if we are evaluating    | None                        |                                                                                                |
| Sticky Note, Sticky Note1, Sticky Note3, Sticky Note4, Sticky Note5, Sticky Note6, Sticky Note7 | Sticky Note                              | Documentation, instructions, and notes             | None                       | None                        | See section 5 for details including links and competition info |

---

### 4. Reproducing the Workflow from Scratch

1. **Manual Trigger Setup**  
   - Create a Manual Trigger node named "When clicking ‘Execute workflow’".

2. **Setup Qdrant Collection Refresh**  
   - Add HTTP Request node "Refresh collection".  
   - Method: POST  
   - URL: `http://XX.XX.XX:6333/collections/agentic-arena/points/delete`  
   - Body: `{ "filter": {} }` (JSON)  
   - Authentication: HTTP Header Auth with Qdrant API key.  
   - Connect Manual Trigger → Refresh collection.

3. **Google Drive PDF Search**  
   - Add Google Drive node "Search PDFs".  
   - Resource: fileFolder, Operation: list files.  
   - Filter: Folder ID set to Agentic Arena folder.  
   - Credentials: Google Drive OAuth2.  
   - Connect Refresh collection → Search PDFs.

4. **Batch Processing of Files**  
   - Add SplitInBatches node "Loop Over Items1".  
   - Connect Search PDFs → Loop Over Items1.

5. **Extract File ID**  
   - Add Set node "Get File ID".  
   - Assign variable `file_id` from `{{$json.id}}`.  
   - Connect Loop Over Items1 → Get File ID.

6. **Call Sub-Workflow**  
   - Add Execute Workflow node "Call 'Agent Arena'".  
   - Set mode to "each", enable "Wait for Sub-Workflow".  
   - Select the sub-workflow (same workflow or imported sub-workflow).  
   - Map workflow inputs if needed.  
   - Connect Get File ID → Call 'Agent Arena'.

7. **Sub-Workflow: PDF Processing**  
   - Add Execute Workflow Trigger node "When Executed by Another Workflow" (entry point).  
   - Add Google Drive node "Get PDF" to download file by `file_id`.  
   - Add HTTP Request node "Mistral Upload" to upload PDF for OCR: multipart/form-data with file data and purpose "ocr".  
   - Add HTTP Request node "Mistral Signed URL" to get signed URL with expiry 24h.  
   - Add HTTP Request node "Mistral DOC OCR" to request OCR with JSON body referencing signed URL and model "mistral-ocr-latest".  
   - Add Code node to map OCR pages to markdown text array.  
   - Add SplitInBatches node "Loop Over Items" to process markdown pages.  
   - Add Set node "Set page" to assign markdown text to `text`.  
   - Add LangChain Text Splitter node "Character Text Splitter" with separator "#" and chunk overlap 100.  
   - Add LangChain Document Loader node "Default Data Loader" with metadata setting filename.  
   - Add LangChain Embeddings node "Embeddings OpenAI" using OpenAI API credentials.  
   - Add LangChain Vector Store node "Qdrant Vector Store" configured for insertion into "agentic-arena" collection with Qdrant API credentials.  
   - Add Wait node for pacing.  
   - Connect nodes sequentially in above order.

8. **RAG Question Answering Setup**  
   - Add Filter node "Filter Empty Rows" to filter out empty `chatInput`.  
   - Add LangChain Agent node "AI Agent" with system message defining policy compliance assistant, RAG tool usage, and response structure.  
   - Add LangChain Memory Buffer Window node "Simple Memory" connected as memory input to AI Agent.  
   - Add LangChain Vector Store node "RAG" configured in retrieval mode with reranker enabled, linked to Cohere reranker node "Reranker Cohere".  
   - Add LangChain Embeddings node "Embeddings OpenAI1" to embed queries.  
   - Add LangChain Chat OpenAI node "OpenAI Chat Model" with GPT-4.1 model and temperature 0.1.  
   - Connect nodes: Filter Empty Rows → AI Agent; RAG uses Embeddings OpenAI1 and Reranker Cohere; AI Agent uses RAG tool and OpenAI Chat Model; Simple Memory connected to AI Agent memory input.

9. **Evaluation Workflow Setup**  
   - Add Evaluation Trigger node "Eval Set" configured to read evaluation questions and answers from Google Sheets (public "Agentic Arena Evaluation Set").  
   - Add Set node "Eval Input" assigning `chatInput` from question and sessionId with random suffix.  
   - Connect Eval Set → Eval Input → Filter Empty Rows.  
   - Add Evaluation node "Only if we are evaluating" to conditionally check evaluation mode.  
   - Add LangChain OpenAI Chat node "LLM as a Judge" to run the evaluation prompt with detailed scoring instructions.  
   - Add Evaluation node "Run Evaluation" with prompt and expected/actual answers mapped.  
   - Add Evaluation node "Save Eval" to write scores and agent answers back to Google Sheets.  
   - Connect nodes in order: Filter Empty Rows → Only if we are evaluating → Run Evaluation → Save Eval.  
   - Connect LLM as a Judge as language model for Run Evaluation.

10. **Respond to Chat**  
    - Add LangChain Chat node "Respond to Chat" to output final AI answers.  
    - Connect Only if we are evaluating → Respond to Chat.

11. **Sticky Notes**  
    - Add Sticky Notes nodes as per documentation and instructions for user clarity.

12. **Credentials Setup**  
    - Setup and configure credentials for:  
      - Google Drive OAuth2  
      - Google Sheets OAuth2  
      - OpenAI API (for embeddings, chat, evaluation)  
      - Cohere API (reranker)  
      - Mistral Cloud API (OCR)  
      - Qdrant API (vector store)

13. **Test Workflow**  
    - Run manual trigger to refresh collection and ingest PDFs.  
    - Submit questions and verify AI Agent responses.  
    - Run evaluation to check scoring correctness.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                   | Context or Link                                                                                                               |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| # Agentic Arena Community Contest - Overview: Build a RAG AI system in n8n that answers questions from PDFs with citations.                                                                                                                                     | Sticky Note5                                                                                                                  |
| Rules: Use provided [PDF files folder](https://drive.google.com/drive/folders/1FqVwbNrAPn2dHhIwSEtlu5kl3z2jEC0U?usp=sharing) and [evaluation set](https://docs.google.com/spreadsheets/d/1cgZzr0-D5Kpd6HrKowyoN_fI2dY0dkP9Ljljxix0AlI/edit?usp=sharing). Download starter workflow. | Sticky Note6                                                                                                                  |
| Solution outline: 1) Create Qdrant collection, 2) Retrieve PDFs from Google Drive, 3) OCR with Mistral, 4) Embed and store in Qdrant, 5) Setup evaluation workflow, 6) Save outputs to Google Sheets.                                                           | Sticky Note7                                                                                                                  |
| Evaluation prompt includes detailed scoring criteria (0-5), step-by-step reasoning, citation checks, and JSON output format for automated correctness assessment.                                                                                              | Run Evaluation node prompt                                                                                                    |
| AI Agent system message specifies strict policy compliance assistant role, RAG usage mandate, no hallucinations, citation format, and detailed response structure.                                                                                              | AI Agent node system message                                                                                                  |
| Includes use of multiple APIs requiring proper credential configuration: OpenAI, Google Drive, Google Sheets, Mistral OCR, Cohere, and Qdrant.                                                                                                                | Credential notes in node configurations                                                                                       |

---

This reference document fully describes the workflow components, their interconnections, configurations, and usage. It enables advanced users and AI agents to understand, reproduce, and modify the system, anticipate errors, and maintain integrations effectively.

---

**Disclaimer:** The text provided is extracted exclusively from an n8n automated workflow. All data processed is legal and public, and the workflow complies strictly with content policies.