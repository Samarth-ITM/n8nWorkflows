RAG-Powered Document Chatbot with OpenAI & Gemini for Multi-Format Documents

https://n8nworkflows.xyz/workflows/rag-powered-document-chatbot-with-openai---gemini-for-multi-format-documents-7371


# RAG-Powered Document Chatbot with OpenAI & Gemini for Multi-Format Documents

### 1. Workflow Overview

This workflow implements a **Retrieval-Augmented Generation (RAG)-powered document chatbot** designed to interact with users by answering questions based on content extracted from uploaded files of multiple formats. It leverages **OpenAI and Google Gemini (PaLM)** AI models to process and embed document content, building an in-memory vector store for efficient retrieval and conversational context management.

The workflow is structured into the following logical blocks:

- **1.1 Input Reception & Preprocessing:** Captures user chat messages and optionally uploaded files, prepares these files for extraction.
- **1.2 File Type Classification & Extraction:** Determines file MIME types, converts data to binary files, and extracts text/content based on file format using specialized extractors.
- **1.3 Data Aggregation & Vectorization:** Aggregates extracted content, splits text into chunks, generates embeddings with Google Gemini, and stores them in a vector store.
- **1.4 Conversational AI & Memory Management:** Receives user queries, retrieves relevant vectors from the vector store, manages session memory buffer, and generates AI responses via OpenAI chat models.
- **1.5 Supporting Utilities:** Includes looping over batch items, splitting files, and handling fallback cases.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Preprocessing

**Overview:**  
This block listens for incoming chat messages and file uploads via a chat interface, processes the input to isolate files, and prepares them for downstream processing.

**Nodes Involved:**  
- When chat message received  
- Code  
- Split Out  

**Node Details:**

- **When chat message received**  
  - Type: Chat trigger (LangChain)  
  - Role: Entry point for user messages and file uploads via webhook; supports public access and all MIME types.  
  - Configurations:  
    - `allowedOrigins` set to "*" (any origin)  
    - File uploads enabled with no MIME restrictions  
    - Session memory enabled ("memory") for context continuity  
  - Inputs: External webhook invocation  
  - Outputs: JSON containing chat input, sessionId, and optionally files  
  - Edge cases: Large files or unsupported file types may cause issues; session memory persistence depends on external settings.

- **Code**  
  - Type: JavaScript code execution  
  - Role: Checks if files are included in the input and returns them separately; else passes input through.  
  - Key logic: Checks existence and keys of binary files in input; returns object with `files` property if present.  
  - Inputs: Output of "When chat message received"  
  - Outputs: Either files object or original input  
  - Edge cases: If no files, proceeds with chat input only.

- **Split Out**  
  - Type: SplitOut node  
  - Role: Splits the `files` array into individual items to process each file separately.  
  - Configurations: Splits on field "files" including binary data  
  - Inputs: Output from Code node  
  - Outputs: Individual file items for parallel processing  
  - Edge cases: Empty or malformed files array may cause no output.

---

#### 2.2 File Type Classification & Extraction

**Overview:**  
Identifies the MIME type of each file, converts its data to binary format, then extracts textual content or data using format-specific extractors.

**Nodes Involved:**  
- Loop Over Items  
- Switch  
- Convert to File (multiple instances: Convert to File, Convert to File1, ..., Convert to File8)  
- Extract from File (multiple instances: Extract from File, Extract from File1, ..., Extract from File7)  

**Node Details:**

- **Loop Over Items**  
  - Type: SplitInBatches  
  - Role: Manages batch processing of files to avoid overloading downstream nodes; passes items one by one or in small sets.  
  - Inputs: Individual files from Split Out  
  - Outputs: Feeds each file to Switch for routing

- **Switch**  
  - Type: Switch (conditional routing)  
  - Role: Routes files based on MIME type to appropriate Convert to File node for binary conversion.  
  - Conditions:  
    - Image types (starts with "image/")  
    - Plain Text ("text/plain")  
    - PDF ("application/pdf")  
    - CSV ("text/csv")  
    - JSON ("application/json")  
    - XML ("application/xml" or "text/xml")  
    - Excel XLS/XLSX (via regex match on MIME)  
    - RTF ("application/rtf")  
    - Fallback to "extra" output  
  - Inputs: Files from Loop Over Items  
  - Outputs: To Convert to File nodes for each MIME type

- **Convert to File / Convert to File1 ... Convert to File8**  
  - Type: ConvertToFile  
  - Role: Converts file data (usually from base64 or other format) into binary property for extraction nodes.  
  - Key Configurations:  
    - Uses filename and MIME type from JSON for naming and MIME assignment  
    - Source property varies (mostly "data")  
    - Binary property name usually "base64DataBinary" or "base64DataUrlBinary" depending on input  
  - Inputs: Files routed from Switch  
  - Outputs: Binary files to corresponding Extract from File nodes  
  - Edge cases: Improper file data or missing properties can cause conversion failure.

- **Extract from File / Extract from File1 ... Extract from File7**  
  - Type: ExtractFromFile  
  - Role: Extracts text or structured data from binary content depending on file format.  
  - Operations used:  
    - Text extraction (default)  
    - PDF extraction  
    - JSON deserialization  
    - XML parsing  
    - XLSX parsing  
    - RTF extraction  
  - Inputs: Binary files from Convert to File nodes  
  - Outputs: Extracted content passed back to Loop Over Items for aggregation  
  - Edge cases: Malformed files, unsupported versions, or corrupted content may cause extraction errors.

---

#### 2.3 Data Aggregation & Vectorization

**Overview:**  
Aggregates all extracted file contents, applies text splitting for manageable chunking, computes embeddings using Google Gemini, and inserts them into an in-memory vector store for retrieval.

**Nodes Involved:**  
- Aggregate  
- Recursive Character Text Splitter  
- Default Data Loader  
- Embeddings Google Gemini  
- Simple Vector Store  
- On form submission (for file uploads outside chat)  

**Node Details:**

- **Aggregate**  
  - Type: Aggregate all item data  
  - Role: Recombines batch-processed extracted texts into a single collection under property `extractedFile`.  
  - Inputs: Extracted content items from Loop Over Items  
  - Outputs: Aggregated JSON array of extracted file contents to AI Agent

- **Recursive Character Text Splitter**  
  - Type: Text Splitter  
  - Role: Splits large texts recursively into chunks with overlap (50 characters) to preserve context for embedding.  
  - Inputs: Extracted texts from Default Data Loader  
  - Outputs: Smaller text chunks for embedding generation  
  - Edge cases: Very large documents may require tuning chunk size or overlap.

- **Default Data Loader**  
  - Type: Document Data Loader  
  - Role: Prepares extracted text chunks to be passed into vector store and embedding generation.  
  - Inputs: Output of Recursive Character Text Splitter  
  - Outputs: Documents ready for embedding and storage

- **Embeddings Google Gemini**  
  - Type: Embeddings (Google Gemini/PaLM)  
  - Role: Generates vector embeddings for text chunks from documents.  
  - Credentials: Requires Google PaLM API key  
  - Inputs: Documents from Default Data Loader  
  - Outputs: Embedding vectors to Simple Vector Store  
  - Edge cases: API rate limits, network errors, invalid API key

- **Simple Vector Store**  
  - Type: Vector store in memory  
  - Role: Stores embeddings with an internal key `"vector_store_key"` for retrieval. Supports insert and retrieval modes.  
  - Inputs: Embeddings from Embeddings Google Gemini  
  - Outputs: Stored vectors for later querying  
  - Edge cases: Store size limited by memory; persistence across executions depends on workflow runtime.

- **On form submission**  
  - Type: Form trigger for standalone file upload  
  - Role: Alternative entry point allowing PDF upload, triggering embedding and storage  
  - Outputs: Triggers Simple Vector Store insertion of new document vectors  
  - Edge cases: Limited to PDF files as per form settings.

---

#### 2.4 Conversational AI & Memory Management

**Overview:**  
Manages conversational context memory, retrieves relevant vectors from the vector store based on user query, and generates AI responses via OpenAI chat model.

**Nodes Involved:**  
- Window Buffer Memory  
- Query Data Tool1  
- Embeddings Google Gemini1  
- AI Agent  
- OpenAI Chat Model  
- Simple Memory  

**Node Details:**

- **Simple Memory**  
  - Type: Memory Buffer Window  
  - Role: Maintains a windowed conversational context buffer for each session with default length 200 tokens.  
  - Inputs: None (initializes context)  
  - Outputs: Context for AI Agent  
  - Edge cases: Memory size constraints; older messages dropped as new arrive.

- **Window Buffer Memory**  
  - Type: Memory Buffer Window (LangChain)  
  - Role: Session-specific memory management keyed by `sessionId` from chat trigger, with window length 200 tokens.  
  - Inputs: Session ID from "When chat message received" node  
  - Outputs: Context passed to AI Agent for informed answers  
  - Edge cases: Session ID missing or inconsistent may cause memory loss.

- **Embeddings Google Gemini1**  
  - Type: Embeddings (Google Gemini/PaLM)  
  - Role: Computes embeddings of the user query to enable vector retrieval.  
  - Inputs: User chat input text  
  - Outputs: Query embeddings to Query Data Tool1  
  - Credentials: Google PaLM API key

- **Query Data Tool1**  
  - Type: Vector store retrieval as tool  
  - Role: Retrieves top 50 matching vectors from vector store using query embeddings. Described as "knowledge_base" tool for AI Agent.  
  - Inputs: Query embeddings from Embeddings Google Gemini1, vector store memory key  
  - Outputs: Relevant context documents to AI Agent

- **AI Agent**  
  - Type: LangChain Agent (AI orchestrator)  
  - Role: Central AI node combining chat input, extracted file contents, vector store retrievals, and memory context to generate answers.  
  - Configurations:  
    - System message defines assistant role and user file/image upload awareness  
    - Passthrough binary images enabled for visual data in context  
    - Text prompt includes extracted files (filtered for non-empty) and user chat input  
  - Inputs:  
    - Aggregated extracted files  
    - Query data tool outputs  
    - Memory buffers  
    - OpenAI chat model for answer generation  
  - Outputs: Final AI-generated response for user  
  - Edge cases: Model API failures, malformed prompt data, missing context.

- **OpenAI Chat Model**  
  - Type: Language model (OpenAI Chat)  
  - Role: Generates natural language responses based on prompts constructed by AI Agent.  
  - Credentials: OpenAI API key required  
  - Inputs: Prompt from AI Agent  
  - Outputs: Textual response passed back to AI Agent for output  
  - Edge cases: API quota, latency, invalid API key.

---

#### 2.5 Supporting Utilities

**Overview:**  
Additional nodes supporting batch processing and workflow orchestration.

**Nodes Involved:**  
- Split Out (already covered)  
- Loop Over Items (already covered)  

**Additional Notes:**  
- The workflow relies on batch processing to handle multiple file uploads efficiently, avoiding timeouts or memory overload.  
- The vector store is in-memory, implying no persistence beyond workflow execution unless externally managed.

---

### 3. Summary Table

| Node Name                 | Node Type                           | Functional Role                            | Input Node(s)                      | Output Node(s)                      | Sticky Note                            |
|---------------------------|-----------------------------------|-------------------------------------------|----------------------------------|-----------------------------------|--------------------------------------|
| When chat message received | LangChain Chat Trigger             | Entry point for chat and file upload      | External webhook                 | Code                              |                                      |
| Code                      | Code                              | Detects and extracts files from input     | When chat message received       | Split Out                         |                                      |
| Split Out                 | SplitOut                         | Splits files array into individual items  | Code                            | Loop Over Items                   |                                      |
| Loop Over Items           | SplitInBatches                   | Processes files in batches                  | Split Out                      | Aggregate, Switch                 |                                      |
| Switch                   | Switch                           | Routes files by MIME type                   | Loop Over Items                | Convert to File nodes, AI Agent   |                                      |
| Convert to File           | ConvertToFile                   | Converts file data to binary                | Switch                        | Loop Over Items                  |                                      |
| Convert to File1          | ConvertToFile                   | Converts file data to binary                | Switch                        | Extract from File                |                                      |
| Convert to File2          | ConvertToFile                   | Converts file data to binary                | Switch                        | Extract from File1               |                                      |
| Convert to File3          | ConvertToFile                   | Converts file data to binary                | Switch                        | Extract from File2               |                                      |
| Convert to File4          | ConvertToFile                   | Converts file data to binary                | Switch                        | Extract from File3               |                                      |
| Convert to File5          | ConvertToFile                   | Converts file data to binary                | Switch                        | Extract from File4               |                                      |
| Convert to File7          | ConvertToFile                   | Converts file data to binary                | Switch                        | Extract from File6               |                                      |
| Convert to File8          | ConvertToFile                   | Converts file data to binary                | Switch                        | Extract from File7               |                                      |
| Extract from File         | ExtractFromFile                 | Extracts text from plain text files         | Convert to File1              | Loop Over Items                  |                                      |
| Extract from File1        | ExtractFromFile                 | Extracts text from PDF                       | Convert to File2              | Loop Over Items                  |                                      |
| Extract from File2        | ExtractFromFile                 | Extracts text from CSV                       | Convert to File3              | Loop Over Items                  |                                      |
| Extract from File3        | ExtractFromFile                 | Extracts JSON data                           | Convert to File4              | Loop Over Items                  |                                      |
| Extract from File4        | ExtractFromFile                 | Extracts XML data                            | Convert to File5              | Loop Over Items                  |                                      |
| Extract from File6        | ExtractFromFile                 | Extracts XLSX data                           | Convert to File7              | Loop Over Items                  |                                      |
| Extract from File7        | ExtractFromFile                 | Extracts RTF data                            | Convert to File8              | Loop Over Items                  |                                      |
| Aggregate                 | Aggregate                       | Aggregates all extracted file contents      | Loop Over Items               | AI Agent                       |                                      |
| Recursive Character Text Splitter | Text Splitter                 | Splits text into chunks with overlap        | Default Data Loader           | Default Data Loader             |                                      |
| Default Data Loader       | Document Data Loader            | Prepares documents for embedding             | Recursive Character Text Splitter | Simple Vector Store          |                                      |
| Embeddings Google Gemini  | Embeddings (Google Gemini)       | Generates embeddings for documents           | Default Data Loader           | Simple Vector Store             |                                      |
| Simple Vector Store       | Vector Store (In-Memory)         | Stores embeddings for retrieval               | Embeddings Google Gemini      |                                |                                      |
| On form submission        | Form Trigger                    | Alternative PDF upload entry point            | External webhook              | Simple Vector Store             |                                      |
| Window Buffer Memory      | Memory Buffer Window            | Manages session-specific conversational memory| When chat message received    | AI Agent                      |                                      |
| Simple Memory             | Memory Buffer Window            | Maintains general conversational memory      | None                        | AI Agent                      |                                      |
| Embeddings Google Gemini1 | Embeddings (Google Gemini)       | Embeds user query for vector retrieval        | AI Agent                     | Query Data Tool1               |                                      |
| Query Data Tool1          | Vector Store Retrieval Tool     | Retrieves relevant vectors for user query     | Embeddings Google Gemini1     | AI Agent                      |                                      |
| AI Agent                 | LangChain Agent                  | Central AI orchestrator combining inputs      | Aggregate, Query Data Tool1, Window Buffer Memory, OpenAI Chat Model | Outputs user response |                                      |
| OpenAI Chat Model         | Language Model (OpenAI)          | Generates AI response text                      | AI Agent                     | AI Agent                      |                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Chat Trigger Node ("When chat message received")**  
   - Type: LangChain Chat Trigger  
   - Configure webhook with public access, allow all origins (`*`), allow file uploads with all MIME types, session memory enabled.  
   - Set input placeholder to "Entrez votre question..." or custom as desired.

2. **Add JavaScript Code Node ("Code")**  
   - Purpose: Detect presence of uploaded files.  
   - Code snippet:  
     ```javascript
     const files = $input.all().at(0)?.binary;
     const isfilesSent = typeof files === 'object' && Object.keys(files)?.length;
     
     if(isfilesSent){
       return { files };
     }
     return $input.all();
     ```  
   - Connect output of chat trigger to Code node.

3. **Add SplitOut Node ("Split Out")**  
   - Field to split: `files`  
   - Include binary data  
   - Connect Code node output to Split Out.

4. **Add SplitInBatches Node ("Loop Over Items")**  
   - Configure batch processing (default batch size or custom as needed).  
   - Connect Split Out node to Loop Over Items.

5. **Add Switch Node ("Switch")**  
   - Add rules based on MIME type of each file:  
     - Starts with "image/" → Output "Image"  
     - Equals "text/plain" → "Plain Text"  
     - Equals "application/pdf" → "PDF"  
     - Equals "text/csv" → "CSV"  
     - Equals "application/json" → "JSON"  
     - Regex for XML MIME types → "XML"  
     - Regex for XLS/XLSX MIME types → "XLS, XLSX"  
     - Equals "application/rtf" → "RTF"  
     - Fallback to "extra"  
   - Connect Loop Over Items output to Switch.

6. **Add ConvertToFile Nodes** for each MIME type output path from Switch:  
   - Configure to convert the file data to binary using proper `fileName` and `mimeType` from JSON.  
   - For example, for PDF path: set source property to "data", output binary property "base64DataBinary".  
   - Repeat for each file type, adjusting binary property names accordingly.

7. **Add ExtractFromFile Nodes** corresponding to each ConvertToFile node:  
   - Configure extraction operation per file type:  
     - Plain Text: default text extraction  
     - PDF: operation "pdf"  
     - CSV: default or custom if needed  
     - JSON: operation "fromJson"  
     - XML: operation "xml"  
     - XLSX: operation "xlsx"  
     - RTF: operation "rtf"  
   - Connect each ConvertToFile node to corresponding ExtractFromFile node.

8. **Connect ExtractFromFile nodes back to Loop Over Items** for aggregation.

9. **Add Aggregate Node ("Aggregate")**  
   - Aggregate all extracted file contents into one field `extractedFile`.  
   - Connect Loop Over Items to Aggregate.

10. **Add Recursive Character Text Splitter Node ("Recursive Character Text Splitter")**  
    - Configure chunk overlap: 50 characters  
    - Connect Aggregate output to this node for text chunking.

11. **Add Default Data Loader Node ("Default Data Loader")**  
    - Configure with custom text splitting mode or defaults as needed.  
    - Connect Recursive Character Text Splitter output here.

12. **Add Embeddings Node ("Embeddings Google Gemini")**  
    - Configure credentials with Google PaLM API key.  
    - Connect Default Data Loader output here.

13. **Add Vector Store Node ("Simple Vector Store")**  
    - Mode: insert  
    - Memory key: `"vector_store_key"`  
    - Connect Embeddings node output here.

14. **Optional: Add Form Trigger Node ("On form submission")**  
    - Configure form with file upload field accepting PDFs.  
    - Connect output to Simple Vector Store node for insertion.

15. **Create Memory Buffer Nodes:**  
    - "Simple Memory": default buffer length 200 tokens (no session key)  
    - "Window Buffer Memory": buffer length 200 tokens with session key from chat message's `sessionId` field.

16. **Add Embeddings Node for Queries ("Embeddings Google Gemini1")**  
    - Credentials: Google PaLM API key  
    - Input: User chat input text from "When chat message received"

17. **Add Vector Store Retrieval Node ("Query Data Tool1")**  
    - Mode: retrieve-as-tool  
    - Top K: 50  
    - Tool name: "knowledge_base"  
    - Memory key: same `"vector_store_key"`  
    - Tool description: "Use this knowledge base to answer questions from the user"  
    - Connect Embeddings Google Gemini1 output here.

18. **Add OpenAI Chat Model Node ("OpenAI Chat Model")**  
    - Credentials: OpenAI API key  
    - Connect output to AI Agent node.

19. **Add AI Agent Node ("AI Agent")**  
    - Parameters:  
      - System message specifying assistant role including file/image awareness  
      - Prompt includes extracted file contents (filtered for non-empty) and chat input  
      - Passthrough binary images enabled  
    - Connect inputs from:  
      - Aggregate (extracted files)  
      - Query Data Tool1 (vector retrieval)  
      - Window Buffer Memory (session memory)  
      - OpenAI Chat Model (language model)  
    - Connect output to user response (e.g., webhook response or chat interface).

20. **Wire all nodes respecting the flow described in connections: Input → Code → Split Out → Loop Over Items → Switch → Convert to File → Extract → Aggregate → AI Agent with embeddings, memory and model nodes as described.**

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                                 |
|----------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------|
| This workflow supports various document formats including PDF, CSV, JSON, XML, XLSX, RTF, plain text and images.   | Multi-format document ingestion and processing                                                                |
| Google Gemini (PaLM) is used for embedding generation, requiring a valid Google PaLM API credential. | https://developers.generativeai.google/products/palm                                                          |
| OpenAI API key is required for generating chat completions via OpenAI Chat Model.                   | https://platform.openai.com/docs/api-reference/chat/create                                                     |
| Conversation memory is managed per session using a window buffer to maintain context without overload. | Ensures user conversations are coherent and context-aware                                                     |
| The AI Agent node acts as the orchestrator, combining user input, document context, vector retrievals and memory to produce answers. | https://docs.n8n.io/integrations/builtin/nodes/langchain/                                                     |
| The vector store is in-memory and ephemeral; for persistent knowledge bases, external storage or databases would be needed. | Important for workflow scaling and state management                                                            |
| Batch processing of files using SplitInBatches node prevents workflow timeout or memory issues with large uploads. | Recommended best practice for handling multiple documents                                                       |

---

**Disclaimer:**  
The provided analysis is based exclusively on an automated n8n workflow. The workflow complies with current content policies and contains no illegal or protected elements. All processed data is legal and publicly accessible.