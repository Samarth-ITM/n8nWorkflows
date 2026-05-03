Process Documents with Recursive Chunking using Google Drive, OpenAI & Gemini RAG

https://n8nworkflows.xyz/workflows/process-documents-with-recursive-chunking-using-google-drive--openai---gemini-rag-5521


# Process Documents with Recursive Chunking using Google Drive, OpenAI & Gemini RAG

### 1. Workflow Overview

This workflow automates the processing of documents stored in a specified Google Drive folder by using recursive chunking and advanced AI models (OpenAI and Google Gemini) combined with vector search (Supabase). It targets use cases such as document ingestion, contextual chunking, embedding creation, and AI-assisted querying for hybrid retrieval-augmented generation (RAG). The workflow is split into three main logical blocks:

- **1.1 Document Ingestion & Extraction:** Detects new files in Google Drive, retrieves metadata and content, and extracts text based on file type (PDF or plain text).
- **1.2 Content Processing & Chunking:** Converts raw text into manageable chunks using a recursive splitting algorithm, then processes these chunks through language models for contextualization and summarization.
- **1.3 Embedding, Storage & Querying:** Creates vector embeddings of document chunks, stores them in Supabase, and enables querying through AI agents combining vector similarity and language models (OpenAI and Google Gemini).

Additionally, there is a user-triggered querying section that interfaces with the embedded data and executes hybrid search plus AI response generation.

---

### 2. Block-by-Block Analysis

#### 2.1 Document Ingestion & Extraction

**Overview:**  
This block monitors a specific Google Drive folder for new files, processes each file individually, extracts metadata, downloads the file content, and routes the content extraction based on file type (PDF or plain text).

**Nodes Involved:**  
- Google Drive Trigger  
- Loop Over Items  
- File info  
- Google Drive  
- Switch  
- Extract from PDF  
- Extract from TEXT  
- PDF to DATA  
- Document Data

**Node Details:**

- **Google Drive Trigger**  
  - Type: Trigger node to monitor Google Drive folder for new files.  
  - Configuration: Watches folder with ID `10c4lWGMSkqZc5JhsyDxZ-jzgBaEEItqU`. Polls every minute for file creation events.  
  - Credentials: Google Drive OAuth2.  
  - Input/Output: No input; outputs new file metadata.  
  - Potential Failures: Auth errors, API quota limits, network issues.

- **Loop Over Items**  
  - Type: SplitInBatches node to process each file individually.  
  - Configuration: Default batch settings to iterate items one by one.  
  - Input: Google Drive Trigger output.  
  - Output: Sends each file metadata downstream.

- **File info**  
  - Type: Set node to extract and assign relevant metadata fields.  
  - Configuration: Extracts File_id, File_Type (MIME type), File_url, File_name from incoming JSON.  
  - Input: Loop Over Items output.  
  - Output: Prepares data for file download.

- **Google Drive**  
  - Type: Google Drive file download node.  
  - Configuration: Downloads file by File_id; converts Google Docs files to plain text format (`text/plain`).  
  - Credentials: Google Drive OAuth2.  
  - Input: File info node.  
  - Output: Raw file content.

- **Switch**  
  - Type: Switch node to route based on File_Type MIME.  
  - Configuration: Routes files to either PDF extraction path (`application/pdf`) or text extraction path (`text/plain`).  
  - Input: Google Drive node output.  
  - Output: Branches to Extract from PDF or Extract from TEXT.

- **Extract from PDF**  
  - Type: ExtractFromFile node.  
  - Configuration: Operation set to extract text from PDF files.  
  - Input: Switch node (PDF branch).  
  - Output: Extracted raw text from PDF.

- **Extract from TEXT**  
  - Type: ExtractFromFile node.  
  - Configuration: Operation set to extract raw text from plain text files.  
  - Input: Switch node (text branch).  
  - Output: Extracted text.

- **PDF to DATA**  
  - Type: Set node.  
  - Configuration: Converts extracted PDF text to JSON format under key `data`.  
  - Input: Extract from PDF.  
  - Output: JSON formatted text content.

- **Document Data**  
  - Type: Set node.  
  - Configuration: Ensures text data is stringified JSON under `data` key for downstream processing.  
  - Input: Either Extract from TEXT or PDF to DATA.  
  - Output: Standardized document text data.

**Edge Cases & Failure Types:**  
- Unsupported file types are ignored (only PDF and plain text processed).  
- Conversion failures if Google Docs conversion fails.  
- Extraction errors for corrupted files or unsupported PDFs.  
- Rate limits or quota issues with Google Drive API.

---

#### 2.2 Content Processing & Chunking

**Overview:**  
This block recursively splits large document text into meaningful chunks preserving context and structure, then prepares these chunks for semantic processing.

**Nodes Involved:**  
- Recursive Splitter  
- Chunk Splitting  
- Basic LLM Chain  
- Summarize

**Node Details:**

- **Recursive Splitter**  
  - Type: Code node executing custom JavaScript.  
  - Configuration:  
    - Splits text into chunks of 1000 characters with 200 overlap.  
    - Attempts to split on paragraphs (`\n\n`), then sentences (`. `), then words, else hard split.  
    - Trims and preserves chunk boundaries for context integrity.  
  - Input: Document Data node output (JSON text).  
  - Output: Array of text chunks under key `chunks`.  
  - Edge Cases:  
    - Short documents produce fewer or single chunk.  
    - Text with minimal punctuation may cause hard splits.  
    - Excessive overlap could duplicate content unnecessarily.

- **Chunk Splitting**  
  - Type: SplitOut node.  
  - Configuration: Splits array of chunks into individual items for batch processing.  
  - Input: Recursive Splitter output.  
  - Output: Individual chunk per item.

- **Basic LLM Chain**  
  - Type: LangChain LLM Chain node.  
  - Configuration:  
    - Uses OpenAI Chat Model to contextualize chunks within the entire document.  
    - Prompt instructs to provide succinct context situating the chunk, correct incomplete numbers, reconstruct cut sentences, preserve tables, without extra formatting.  
  - Input: Chunk Splitting output.  
  - Output: Contextualized chunk text.  
  - Edge Cases:  
    - Prompt failures if chunk content is malformed.  
    - Rate limits on OpenAI API.  
    - Potential hallucinations if chunk data insufficient.

- **Summarize**  
  - Type: Summarize node.  
  - Configuration: Concatenates multiple chunk texts using custom separator `###SPLIT###` for final summary generation.  
  - Input: Basic LLM Chain output (contextualized chunks).  
  - Output: Summarized document text.

---

#### 2.3 Embedding, Storage & Querying

**Overview:**  
This block converts processed document chunks into vector embeddings, stores these embeddings in a Supabase vector store with metadata for retrieval, and provides user-triggered querying through AI agents combining hybrid search and language models.

**Nodes Involved:**  
- Embeddings OpenAI  
- Supabase Vector Store  
- Default Data Loader  
- OpenAI Chat Model  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- OpenAI  
- AI Agent  
- Google Gemini Chat Model  
- Supabase Query (HTTP Request Tool)

**Node Details:**

- **Embeddings OpenAI**  
  - Type: LangChain embeddings node.  
  - Configuration: Uses OpenAI API to generate vector embeddings from chunk text.  
  - Credentials: OpenAI API.  
  - Input: Summarize output or chunk text.  
  - Output: Vector embeddings.

- **Supabase Vector Store**  
  - Type: Vector store node for Supabase.  
  - Configuration: Inserts embeddings into `documents` table along with metadata (e.g., File_url).  
  - Credentials: Supabase API key.  
  - Input: Embeddings OpenAI and Default Data Loader.  
  - Output: Confirmation of insert.  
  - Edge Cases:  
    - Insert failures due to network or permission errors.  
    - Data schema mismatches.

- **Default Data Loader**  
  - Type: LangChain document loader.  
  - Configuration: Attaches metadata (File_url) to embeddings for contextual retrieval.  
  - Input: Summarized or chunked document data.  
  - Output: Document data with metadata for vector store.

- **OpenAI Chat Model**  
  - Type: LangChain OpenAI Chat LM node.  
  - Configuration: Uses GPT-4o-mini model for contextual understanding of document chunks.  
  - Credentials: OpenAI API.  
  - Input: Basic LLM Chain (as language model backend).

- **When clicking ‘Execute workflow’**  
  - Type: Manual trigger node for user-initiated queries.  
  - Configuration: None.  
  - Output: Triggers query processing chain.

- **OpenAI**  
  - Type: LangChain OpenAI node configured for image analysis.  
  - Configuration: Uses chatgpt-4o-latest model for input in base64 image format with analyze operation.  
  - Credentials: OpenAI API.  
  - Input: Manual Trigger.  
  - Output: AI processing of input images.

- **AI Agent**  
  - Type: LangChain Agent node.  
  - Configuration: AI Examiner Agent designed to mark student exam answers using Supabase DB for markschemes.  
  - Prompt details specify marking criteria, error handling, bulk and individual marking, and response formatting.  
  - Input: OpenAI and Google Gemini Chat Model outputs.  
  - Output: Final marking and feedback.

- **Google Gemini Chat Model**  
  - Type: LangChain LM node for Google PaLM Gemini.  
  - Configuration: Uses model "models/gemini-2.5-flash-preview-04-17" for generating final responses in the query flow.  
  - Credentials: Google Palm API.  
  - Input: AI Agent.  
  - Output: Hybrid RAG answer generation.

- **SupaBase Query (HTTP Request Tool)**  
  - Type: HTTP Request node.  
  - Configuration: POST request to Supabase edge function for querying markschemes or documents.  
  - Headers include Authorization with Supabase Key.  
  - Input: AI Agent as an external AI tool.  
  - Output: Query results for marking or retrieval.  
  - Edge Cases: Auth failures, API downtime, rate limits.

---

### 3. Summary Table

| Node Name                 | Node Type                             | Functional Role                                 | Input Node(s)                   | Output Node(s)                   | Sticky Note                                           |
|---------------------------|-------------------------------------|------------------------------------------------|--------------------------------|---------------------------------|-------------------------------------------------------|
| Google Drive Trigger       | Google Drive Trigger                 | Detect new files in monitored folder           | -                              | Loop Over Items                 | 1. Document Ingestion & Processing                    |
| Loop Over Items           | SplitInBatches                      | Process each detected file individually         | Google Drive Trigger            | File info                      | 1. Document Ingestion & Processing                    |
| File info                 | Set                                | Extract and set file metadata                    | Loop Over Items                | Google Drive                   | 1. Document Ingestion & Processing                    |
| Google Drive              | Google Drive                        | Download file content                            | File info                     | Switch                        | 1. Document Ingestion & Processing                    |
| Switch                    | Switch                             | Route processing by file MIME type               | Google Drive                  | Extract from PDF, Extract from TEXT | 1. Document Ingestion & Processing                    |
| Extract from PDF          | ExtractFromFile                    | Extract text from PDF                            | Switch (PDF branch)            | PDF to DATA                   | 1. Document Ingestion & Processing                    |
| Extract from TEXT         | ExtractFromFile                    | Extract text from plain text files               | Switch (text branch)           | Document Data                 | 1. Document Ingestion & Processing                    |
| PDF to DATA               | Set                                | Format extracted PDF text as JSON                | Extract from PDF              | Document Data                 | 1. Document Ingestion & Processing                    |
| Document Data             | Set                                | Prepare standardized JSON text data              | Extract from TEXT, PDF to DATA | Recursive Splitter            | 1. Document Ingestion & Processing                    |
| Recursive Splitter        | Code                              | Recursively chunk document text                   | Document Data                 | Chunk Splitting               | 2. Content Processing & Chunking                      |
| Chunk Splitting           | SplitOut                          | Split chunks into individual items                | Recursive Splitter            | Basic LLM Chain               | 2. Content Processing & Chunking                      |
| Basic LLM Chain           | LangChain LLM Chain               | Contextualize chunks with AI prompt                | Chunk Splitting              | Summarize                    | 2. Content Processing & Chunking                      |
| Summarize                 | Summarize                         | Aggregate contextualized chunks into summary      | Basic LLM Chain              | Supabase Vector Store         | 2. Content Processing & Chunking                      |
| Embeddings OpenAI         | LangChain Embeddings              | Create vector embeddings from text                 | Summarize                    | Supabase Vector Store         | 3. Embedding, Storage & Querying                      |
| Supabase Vector Store     | LangChain Vector Store            | Store embeddings and metadata in Supabase          | Embeddings OpenAI, Default Data Loader | -                         | 3. Embedding, Storage & Querying                      |
| Default Data Loader       | LangChain Document Loader         | Attach metadata to document chunks                  | Summarize                    | Supabase Vector Store         | 3. Embedding, Storage & Querying                      |
| OpenAI Chat Model         | LangChain OpenAI Chat LM          | Provide language model backend for chunk context   | Basic LLM Chain (for AI)     | Basic LLM Chain               | 3. Embedding, Storage & Querying                      |
| When clicking ‘Execute workflow’ | Manual Trigger               | User-triggered query initiation                     | -                            | OpenAI                       | 3. Embedding, Storage & Querying                      |
| OpenAI                    | LangChain OpenAI                  | Analyze user input (image analysis)                 | Manual Trigger               | AI Agent                    | 3. Embedding, Storage & Querying                      |
| AI Agent                  | LangChain Agent                   | Mark student answers and manage hybrid search       | OpenAI Chat Model, Google Gemini Chat Model | -                       | 3. Embedding, Storage & Querying                      |
| Google Gemini Chat Model  | LangChain Google Gemini LM        | Generate final hybrid RAG responses                  | AI Agent                    | AI Agent                    | 3. Embedding, Storage & Querying                      |
| SupaBase Query            | HTTP Request Tool                 | Query Supabase DB for markschemes or data           | AI Agent (ai_tool)            | AI Agent                    | 3. Embedding, Storage & Querying                      |
| Sticky Note               | Sticky Note                      | Explains high-level block structure                 | -                            | -                           | Covers Document Ingestion, Chunking, Embedding blocks |
| Sticky Note1              | Sticky Note                      | Explains query processing and retrieval flow         | -                            | -                           | Covers Query Processing & Retrieval                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Drive Trigger Node**  
   - Type: Google Drive Trigger  
   - Credentials: Google Drive OAuth2 API account.  
   - Configure: Watch specific folder (`10c4lWGMSkqZc5JhsyDxZ-jzgBaEEItqU`) for new file creation events, polling every minute.

2. **Create Loop Over Items Node**  
   - Type: SplitInBatches  
   - Connect Google Drive Trigger output to Loop Over Items input.  
   - Use default batch size to process each new file individually.

3. **Create File info Node**  
   - Type: Set  
   - Connect Loop Over Items output to File info input.  
   - Set variables to extract:  
     - `File_id` = `{{$json.id}}`  
     - `File_Type` = `{{$json.mimeType}}`  
     - `File_url` = `{{$json.webViewLink}}`  
     - `File_name` = `{{$json.name}}`

4. **Create Google Drive Node**  
   - Type: Google Drive (Operation: Download)  
   - Credentials: Google Drive OAuth2 API account.  
   - File ID: Use expression `={{ $json.File_id }}`  
   - Enable Google Docs conversion to `text/plain`.

5. **Create Switch Node**  
   - Type: Switch  
   - Condition: Check `{{$json.File_Type}}`  
   - Branch 1: If equals `application/pdf` → Extract from PDF  
   - Branch 2: If equals `text/plain` → Extract from TEXT

6. **Create Extract from PDF Node**  
   - Type: ExtractFromFile  
   - Operation: PDF  
   - Connect Switch PDF branch output to this node.

7. **Create Extract from TEXT Node**  
   - Type: ExtractFromFile  
   - Operation: Text  
   - Connect Switch TEXT branch output to this node.

8. **Create PDF to DATA Node**  
   - Type: Set  
   - Connect Extract from PDF output here.  
   - Set mode to Raw and JSON output:  
     ```json
     {
       "data": {{JSON.stringify($json.text)}}
     }
     ```

9. **Create Document Data Node**  
   - Type: Set  
   - Connect Extract from TEXT and PDF to DATA nodes to this node (merge branches).  
   - Set mode to Raw and JSON output:  
     ```json
     {
       "data": {{ JSON.stringify($json.data) }}
     }
     ```

10. **Create Recursive Splitter Node**  
    - Type: Code  
    - Connect Document Data output here.  
    - Paste provided JavaScript for recursive chunking (chunk size 1000, overlap 200).  
    - Output key: `chunks` array.

11. **Create Chunk Splitting Node**  
    - Type: SplitOut  
    - Connect Recursive Splitter output here.  
    - Set field to split out: `chunks`.

12. **Create Basic LLM Chain Node**  
    - Type: LangChain LLM Chain  
    - Connect Chunk Splitting output here.  
    - Configure prompt to contextualize each chunk using entire document data (as per original prompt).  
    - Use OpenAI Chat Model (GPT-4o-mini) as language model backend.

13. **Create Summarize Node**  
    - Type: Summarize  
    - Connect Basic LLM Chain output here.  
    - Configure to concatenate chunk texts with separator `###SPLIT###`.

14. **Create Embeddings OpenAI Node**  
    - Type: LangChain Embeddings OpenAI  
    - Connect Summarize output here.  
    - Use OpenAI API credentials.

15. **Create Default Data Loader Node**  
    - Type: LangChain Document Default Data Loader  
    - Connect Summarize output here.  
    - Attach metadata, e.g., `File_url` from File info node.

16. **Create Supabase Vector Store Node**  
    - Type: LangChain Vector Store Supabase  
    - Connect Embeddings OpenAI (embedding input) and Default Data Loader (document input) here.  
    - Use Supabase API credentials.  
    - Table name: `documents`.  
    - Mode: Insert.

17. **Create OpenAI Chat Model Node**  
    - Type: LangChain OpenAI Chat LM  
    - Use GPT-4o-mini model.  
    - Connect it as AI language model backend for Basic LLM Chain.

18. **Create Manual Trigger Node**  
    - Type: Manual Trigger (When clicking ‘Execute workflow’)  
    - For user-initiated queries.

19. **Create OpenAI Node**  
    - Type: LangChain OpenAI  
    - Configure for image analysis using GPT-4o-latest with input type base64.  
    - Connect Manual Trigger to this node.

20. **Create AI Agent Node**  
    - Type: LangChain Agent  
    - Configure with detailed prompt for marking student exam answers using Supabase DB via HTTP API.  
    - Connect OpenAI and Google Gemini Chat Model outputs to this agent.

21. **Create Google Gemini Chat Model Node**  
    - Type: LangChain Google Gemini LM  
    - Model name: `models/gemini-2.5-flash-preview-04-17`.  
    - Connect AI Agent input and output accordingly.

22. **Create SupaBase Query Node**  
    - Type: HTTP Request Tool  
    - POST to Supabase edge function URL for querying DB.  
    - Set Authorization header with Supabase Key.  
    - Connect AI Agent as external AI tool input.

23. **Connect workflow nodes as per original connections:**  
    - Google Drive Trigger → Loop Over Items → File info → Google Drive → Switch → Extract from PDF / Extract from TEXT → PDF to DATA / Document Data → Recursive Splitter → Chunk Splitting → Basic LLM Chain → Summarize → Embeddings OpenAI + Default Data Loader → Supabase Vector Store  
    - Manual Trigger → OpenAI → AI Agent → Google Gemini Chat Model → AI Agent (loop) → SupaBase Query

24. **Validate all credentials:**  
    - Google Drive OAuth2  
    - OpenAI API keys (for embeddings, chat models, image analysis)  
    - Google Palm API for Gemini  
    - Supabase API key

25. **Test each block separately:**  
    - File ingestion and extraction  
    - Chunk splitting and AI contextualization  
    - Embedding insertion and querying  
    - User query handling and response generation

---

### 5. General Notes & Resources

| Note Content                                                                                                                             | Context or Link                                                                                  |
|------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Workflow enables hybrid RAG combining vector similarity and keyword matching for document query and AI-assisted marking of exam answers | Sticky Note and Sticky Note1 content that describes block-level logic and query processing flow |
| Google Drive folder watched is `10c4lWGMSkqZc5JhsyDxZ-jzgBaEEItqU` (OpenAI IMG Gen 1)                                                    | Google Drive Trigger configuration                                                             |
| OpenAI models used: GPT-4o-mini (contextualization), GPT-4o-latest (image analysis)                                                       | Node configurations                                                                             |
| Google Gemini model used: `models/gemini-2.5-flash-preview-04-17`                                                                          | Google Gemini Chat Model node                                                                   |
| Supabase used as vector store and database for markscheme retrieval                                                                        | Supabase Vector Store and HTTP Request Tool node                                               |
| Recursive chunking code carefully balances paragraph, sentence, and word splits with overlap to preserve context                         | Recursive Splitter code node                                                                    |

---

**Disclaimer:** The provided description and analysis are based exclusively on the given n8n workflow JSON. The workflow respects current content policies and processes only legal and public data.