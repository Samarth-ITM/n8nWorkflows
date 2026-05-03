Process Documents & Build Semantic Search with OpenAI, Gemini & Qdrant

https://n8nworkflows.xyz/workflows/process-documents---build-semantic-search-with-openai--gemini---qdrant-7882


# Process Documents & Build Semantic Search with OpenAI, Gemini & Qdrant

### 1. Workflow Overview

This workflow automates the ingestion, processing, and semantic storage of documents from Google Drive and user file uploads into a Qdrant vector database, enabling semantic search capabilities powered by OpenAI embeddings and Google Gemini AI. It includes two primary pipelines:

- **1.1 Google Drive File Processing Pipeline:** Automatically or manually triggered, this pipeline lists files from a specified Google Drive folder, downloads them as PDFs, converts them into text embeddings, stores these embeddings into a Qdrant collection, and then deletes the original files from Drive.

- **1.2 Manual File Upload Processing Pipeline:** Handles files uploaded via a web form, processes them similarly by chunking and embedding, and inserts into Qdrant.

Additionally, the workflow supports a **1.3 AI Chatbot Interaction Block** that listens for chat messages and uses the Qdrant vector store and Google Gemini chat model to provide AI-driven semantic responses based on the stored document data.

Logical blocks:

- **1.1 Drive File Monitoring & Processing**  
- **1.2 Manual Upload Handling & Processing**  
- **1.3 AI Semantic Search & Chat Interaction**  
- **1.4 Configuration and Documentation Notes** (Sticky notes for setup, warnings, tips)

---

### 2. Block-by-Block Analysis

#### 2.1 Drive File Monitoring & Processing

**Overview:**  
This block continuously monitors a specific Google Drive folder for new files or is manually triggered to process existing files. It downloads each file as PDF, generates embeddings using OpenAI, inserts vectors into Qdrant, and deletes the processed file from Drive to maintain a clean folder.

**Nodes Involved:**  
- New File In Google Drive Folder  
- List Files in Google Drive Folder  
- Download File  
- Insert to Qdrant  
- Delete File  
- Merge  
- Manually Trigger Workflow  

**Node Details:**

- **New File In Google Drive Folder**  
  - Type: Google Drive Trigger  
  - Role: Automatically triggers workflow on new file creation in the configured folder  
  - Config: Watches folder by folder ID, polls every hour  
  - Outputs: File metadata for further processing  
  - Edge Cases: Missed triggers if polling intervals too long; Google Drive API quota limits  
  - Credential: Google Drive OAuth2 with read permissions  

- **Manually Trigger Workflow**  
  - Type: Manual Trigger  
  - Role: Allows manual start of batch processing for existing files  
  - Config: No parameters  
  - Outputs: Passes control to listing node  
  - Edge Cases: None  

- **List Files in Google Drive Folder**  
  - Type: Google Drive node (list operation)  
  - Role: Lists one file at a time (limit=1) from the target folder  
  - Config: Filters by specific folder ID, limits to 1 file per execution for batch processing  
  - Outputs: File metadata (including file ID)  
  - Edge Cases: Empty folder results in no output; API rate limits  
  - Retry enabled on failure  

- **Download File**  
  - Type: Google Drive (download operation)  
  - Role: Downloads the file content converting Google Docs/Sheets/Slides/Drawings to PDF format  
  - Config: Uses file ID from previous node; binary data saved as 'pdf-document'  
  - Outputs: Binary PDF data with metadata  
  - OnError: Continue workflow even if download fails (to avoid halting)  
  - Edge Cases: Conversion failure, network errors, file access denied  

- **Insert to Qdrant**  
  - Type: LangChain Qdrant Vector Store (insert mode)  
  - Role: Inserts document embeddings into the 'fairwork' Qdrant collection  
  - Config: Uses OpenAI embeddings output; no additional options  
  - Credential: Qdrant API key and endpoint  
  - Edge Cases: API authentication failure, collection not found, storage quota exceeded  

- **Delete File**  
  - Type: Google Drive (delete file operation)  
  - Role: Deletes the processed file permanently from Google Drive  
  - Config: Uses file ID from Download File node  
  - ExecuteOnce: True (prevents multiple deletions in parallel)  
  - OnError: Continue workflow to avoid blocking  
  - Edge Cases: Permission denied, file already deleted, network error  

- **Merge**  
  - Type: Merge node (chooseBranch mode)  
  - Role: Combines outputs from Insert to Qdrant and Delete File nodes to continue workflow  
  - Config: Chooses branch depending on flow  

---

#### 2.2 Manual Upload Handling & Processing

**Overview:**  
Processes files uploaded through a web form by splitting them into individual file items, chunking text, embedding, and storing in Qdrant. Batch processing is employed to handle one file at a time for stability.

**Nodes Involved:**  
- File Upload Form  
- Split Form Files (Code node)  
- Split Form Batches  
- Data Loader for Form Files  
- Recursive Character Text Splitter 2  
- Embeddings OpenAI1  
- Insert to Qdrant1  

**Node Details:**

- **File Upload Form**  
  - Type: Form Trigger  
  - Role: Receives user file uploads via a web form  
  - Config: Accepts PDF, DOCX, DOC, CSV files; path configured for webhook  
  - Outputs: Binary file data with metadata  
  - Edge Cases: Large files, unsupported file types, upload failures  

- **Split Form Files**  
  - Type: Code (JavaScript)  
  - Role: Extracts multiple uploaded files from form submission into separate items  
  - Key Logic: Iterates over binary properties starting with 'Files_', outputs one item per file  
  - Inputs: Raw form submission items  
  - Outputs: Separate items for each uploaded file  
  - Edge Cases: No files uploaded, malformed data  

- **Split Form Batches**  
  - Type: Split In Batches  
  - Role: Processes files one at a time sequentially to avoid memory overload  
  - Config: reset = false (maintains state across runs)  
  - Edge Cases: State reset issues causing infinite loops or skipping  

- **Data Loader for Form Files**  
  - Type: LangChain Document Default Data Loader  
  - Role: Converts binary file data into textual documents suitable for embedding  
  - Config: Data type binary, custom text splitting mode (to be paired with splitter)  

- **Recursive Character Text Splitter 2**  
  - Type: LangChain Text Splitter  
  - Role: Splits text into chunks of 1500 tokens with 250 token overlap  
  - Purpose: Maintains context for semantic search while controlling chunk size  
  - Edge Cases: Very small files produce fewer chunks, very large files produce many chunks  

- **Embeddings OpenAI1**  
  - Type: LangChain OpenAI Embeddings  
  - Role: Generates vector embeddings from text chunks using text-embedding-3-large  
  - Credential: OpenAI API key  
  - Edge Cases: API limits, embedding failures  

- **Insert to Qdrant1**  
  - Type: LangChain Qdrant Vector Store (insert mode)  
  - Role: Inserts embeddings into the same Qdrant collection ('fairwork')  
  - Credential: Qdrant API  
  - Edge Cases: Same as Insert to Qdrant above  

---

#### 2.3 AI Semantic Search & Chat Interaction

**Overview:**  
Provides a chatbot interface that listens for user chat messages, retrieves relevant documents from Qdrant, and responds using Google Gemini Chat language model enhanced by OpenAI embeddings and memory buffers.

**Nodes Involved:**  
- When chat message received  
- AI Agent  
- Qdrant Vector Store  
- Google Gemini Chat Model  
- Embeddings OpenAI2  
- Simple Memory  

**Node Details:**

- **When chat message received**  
  - Type: LangChain Chat Trigger  
  - Role: Webhook that initiates AI agent processing on incoming chat messages  
  - Outputs: Chat messages for AI Agent node  

- **AI Agent**  
  - Type: LangChain Agent  
  - Role: Orchestrates combining vector retrieval, embeddings, memory, and chat model to generate responses  
  - Connections: Uses Qdrant Vector Store as retrieval tool, Gemini Chat Model for language generation, and Simple Memory for context window  
  - Edge Cases: API failures, memory overflow, rate limiting  

- **Qdrant Vector Store**  
  - Type: LangChain Vector Store (retrieve as tool mode)  
  - Role: Retrieves top K=10 relevant documents to answer queries  
  - Credential: Qdrant API  
  - Edge Cases: Empty collection, query failures  

- **Google Gemini Chat Model**  
  - Type: LangChain LM Chat Google Gemini  
  - Role: Produces language model responses based on context and retrieved documents  
  - Config: Temperature 0.4 for balanced creativity and coherence  
  - Credential: Google PaLM API  
  - Edge Cases: API limits, latency  

- **Embeddings OpenAI2**  
  - Type: LangChain OpenAI Embeddings  
  - Role: Provides embedding support for query text or other AI agent tasks  
  - Credential: OpenAI API  

- **Simple Memory**  
  - Type: LangChain Memory Buffer Window  
  - Role: Maintains last 50 conversational exchanges to provide context  
  - Edge Cases: Memory size too small or too large may affect performance or context  

---

#### 2.4 Configuration and Documentation Notes (Sticky Notes)

**Overview:**  
Sticky notes provide critical information on setup, warnings, batch processing logic, alternative approaches, performance tips, troubleshooting, and workflow overview.

**Key notes:**

- **Workflow Overview:** Explanation of dual-trigger system, processing flow, and use cases.  
- **Trigger Configuration:** Setup instructions for Google Drive OAuth2, folder precautions, and credential updates.  
- **Batch Processing Logic:** Explains batch size 1 for safety and performance.  
- **Delete Warning:** Critical warning about permanent deletion of files.  
- **Document Pipeline:** Detailed steps in document processing and chunking strategy.  
- **Setup Requirements:** Lists required credentials and configuration reminders.  
- **Performance Tips:** Time and cost optimization advice.  
- **Troubleshooting:** Common issues and how to address them.  
- **Alternative Approaches:** Suggestions for archiving files instead of deleting, duplicate detection, and metadata tagging.  
- **Manual Form Processing Flow:** Overview of form file batch processing.

---

### 3. Summary Table

| Node Name                      | Node Type                           | Functional Role                        | Input Node(s)                     | Output Node(s)                   | Sticky Note                                                                                                       |
|--------------------------------|-----------------------------------|-------------------------------------|----------------------------------|---------------------------------|------------------------------------------------------------------------------------------------------------------|
| New File In Google Drive Folder | Google Drive Trigger               | Auto-trigger on new Drive files     | -                                | List Files in Google Drive Folder | Setup instructions and warnings about folder and deletion                                                        |
| Manually Trigger Workflow       | Manual Trigger                    | Manual start for batch processing   | -                                | List Files in Google Drive Folder | Setup instructions                                                                                               |
| List Files in Google Drive Folder| Google Drive (list)               | Lists files for processing          | New File In Google Drive Folder, Manually Trigger Workflow | Download File                   | Batch processing explanation                                                                                      |
| Download File                  | Google Drive (download)            | Downloads and converts file to PDF  | List Files in Google Drive Folder| Delete File, Insert to Qdrant     | Document pipeline details                                                                                         |
| Delete File                   | Google Drive (delete)              | Deletes processed files from Drive  | Download File                    | Merge                           | Critical deletion warning                                                                                        |
| Insert to Qdrant               | LangChain Qdrant Vector Store     | Inserts embeddings into Qdrant      | Download File                   | Merge                           | Document processing pipeline                                                                                      |
| Merge                        | Merge node                        | Combines branches                   | Delete File, Insert to Qdrant     | List Files in Google Drive Folder | Batch processing loop                                                                                            |
| File Upload Form               | Form Trigger                     | Receives user file uploads          | -                                | Split Form Files                | Manual form processing overview                                                                                   |
| Split Form Files               | Code                             | Splits multiple uploaded files      | File Upload Form                | Split Form Batches              | Manual form processing overview                                                                                   |
| Split Form Batches             | Split In Batches                  | Processes files one at a time       | Split Form Files                | Data Loader for Form Files, Insert to Qdrant1 (via branch) | Batch processing logic                                                                                            |
| Data Loader for Form Files      | LangChain Document Data Loader    | Loads binary files into text docs   | Split Form Batches              | Recursive Character Text Splitter 2 | Document pipeline                                                                                                |
| Recursive Character Text Splitter 2 | LangChain Text Splitter        | Splits text into chunks              | Data Loader for Form Files      | Embeddings OpenAI1             | Document pipeline                                                                                                |
| Embeddings OpenAI1            | LangChain OpenAI Embeddings       | Creates vector embeddings            | Recursive Character Text Splitter 2 | Insert to Qdrant1             | Document processing pipeline                                                                                      |
| Insert to Qdrant1             | LangChain Qdrant Vector Store     | Inserts embeddings                   | Embeddings OpenAI1             | Split Form Batches (branch 2) | Document processing pipeline                                                                                      |
| When chat message received    | LangChain Chat Trigger            | Starts AI chat interaction           | -                                | AI Agent                      | Chatbot demo note                                                                                                |
| AI Agent                     | LangChain Agent                   | Orchestrates chatbot AI              | When chat message received       | -                             | Chatbot demo note                                                                                                |
| Qdrant Vector Store          | LangChain Qdrant Vector Store     | Retrieves vectors for chat queries   | Embeddings OpenAI2             | AI Agent                      | Chatbot demo note                                                                                                |
| Google Gemini Chat Model     | LangChain LM Chat Google Gemini   | Generates chat responses             | AI Agent                      | AI Agent                      | Chatbot demo note                                                                                                |
| Embeddings OpenAI2           | LangChain OpenAI Embeddings       | Embeddings for chat queries          | AI Agent                      | Qdrant Vector Store           | Chatbot demo note                                                                                                |
| Simple Memory                | LangChain Memory Buffer Window    | Maintains chat context memory        | AI Agent                      | AI Agent                      | Chatbot demo note                                                                                                |
| Data Loader for Google Drive Files | LangChain Document Data Loader | Loads downloaded files into text docs | Recursive Character Text Splitter 1 | Insert to Qdrant             | Document pipeline                                                                                                |
| Recursive Character Text Splitter 1 | LangChain Text Splitter        | Splits text from Drive files         | Data Loader for Google Drive Files | Embeddings OpenAI            | Document pipeline                                                                                                |
| Embeddings OpenAI            | LangChain OpenAI Embeddings       | Embeds Drive file chunks             | Recursive Character Text Splitter 1 | Insert to Qdrant             | Document pipeline                                                                                                |
| Insert to Qdrant             | LangChain Qdrant Vector Store     | Inserts Drive file embeddings        | Embeddings OpenAI             | Merge                         | Document pipeline                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Drive OAuth2 Credential**  
   - Permissions: Read, Write, Delete  

2. **Create OpenAI API Credential**  
   - Use API key with embedding model access  

3. **Create Qdrant API Credential**  
   - Endpoint and API key for your Qdrant cloud instance  

4. **Create Google PaLM API Credential**  
   - For Google Gemini Chat Model node  

5. **Build Drive File Processing Pipeline**  
   - Add **New File In Google Drive Folder** Trigger node  
     - Configure to watch specific folder by ID  
     - Poll every hour or suitable interval  
     - Assign Google Drive OAuth2 credentials  
   - Add **Manually Trigger Workflow** node for manual batch processing  
   - Add **List Files in Google Drive Folder** node  
     - Limit to 1 file per execution for batch stability  
     - Filter by folder ID  
     - Credential: Google Drive OAuth2  
   - Connect New File trigger and Manual Trigger to List Files node  
   - Add **Download File** node  
     - Use file ID from List Files  
     - Convert Google Docs/Sheets etc. to PDF  
     - Store binary in property "pdf-document"  
     - Credential: Google Drive OAuth2  
   - Add **Delete File** node  
     - Use file ID from Download File  
     - Credential: Google Drive OAuth2  
     - Set execute once = true  
   - Add **Recursive Character Text Splitter 1** (LangChain)  
     - Chunk size 1500 tokens, overlap 250 tokens  
   - Add **Data Loader for Google Drive Files** (LangChain)  
     - Data type binary, textSplittingMode custom  
   - Add **Embeddings OpenAI** node  
     - Model: text-embedding-3-large  
     - Credential: OpenAI API  
   - Add **Insert to Qdrant** node  
     - Mode: insert  
     - Set Qdrant collection to your chosen collection (e.g., "fairwork")  
     - Credential: Qdrant API  
   - Add **Merge** node (chooseBranch mode)  
     - Connect Delete File and Insert to Qdrant outputs to Merge inputs  
     - Connect Merge output back to List Files node to loop  

6. **Build Manual Upload Processing Pipeline**  
   - Add **File Upload Form** node  
     - Configure form path and accepted file types (.pdf, .docx, .doc, .csv)  
   - Add **Split Form Files** node (Code)  
     - Use provided JavaScript to split multiple uploaded files into separate items  
   - Add **Split In Batches** node  
     - Configure reset = false  
     - Batch size: 1 by default  
   - Add **Data Loader for Form Files** (LangChain)  
     - Data type binary, textSplittingMode custom  
   - Add **Recursive Character Text Splitter 2**  
     - Same chunk and overlap settings as above  
   - Add **Embeddings OpenAI1**  
     - Model: text-embedding-3-large  
     - Credential: OpenAI API  
   - Add **Insert to Qdrant1**  
     - Mode: insert  
     - Credential: Qdrant API  
   - Connect nodes in order: File Upload Form → Split Form Files → Split In Batches → Data Loader → Splitter → Embeddings → Insert to Qdrant1  
   - Configure Split In Batches node to branch to Insert to Qdrant1 as second output  

7. **Build AI Chatbot Interaction Block**  
   - Add **When chat message received** node  
     - Configure webhook for chat messages  
   - Add **AI Agent** node  
     - Connect chat message trigger to AI Agent  
   - Add **Qdrant Vector Store** node  
     - Mode: retrieve-as-tool  
     - TopK: 10  
     - Credential: Qdrant API  
   - Add **Google Gemini Chat Model** node  
     - Temperature: 0.4  
     - Credential: Google PaLM API  
   - Add **Embeddings OpenAI2** node  
     - Model: text-embedding-3-large  
     - Credential: OpenAI API  
   - Add **Simple Memory** node  
     - Context window length: 50  
   - Connect nodes as:  
     - AI Agent uses Qdrant Vector Store as retrieval tool  
     - AI Agent uses Google Gemini Chat Model as language model  
     - AI Agent uses Embeddings OpenAI2 for embedding support  
     - AI Agent uses Simple Memory for context  

8. **Add Sticky Notes for Documentation and Warnings**  
   - Add notes on trigger setup, deletion warnings, batch processing logic, troubleshooting, performance, and alternative approaches as per provided content  

9. **Activate Workflow**  
   - Test with sample files and verify insertion and deletion  
   - Monitor logs for errors and adjust batch size or chunking parameters accordingly  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                         | Context or Link                             |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------|
| Workflow permanently deletes processed files from Google Drive. Backup important files before use.                                                                 | DELETE WARNING sticky note                  |
| Use a dedicated Google Drive folder for processing to avoid accidental deletion of important files.                                                                | Trigger Configuration sticky note           |
| For cost optimization, consider switching OpenAI embeddings model to text-embedding-3-small.                                                                        | Setup Requirements sticky note               |
| Batch size of 1 is recommended to prevent memory issues and allow precise error handling.                                                                            | Batch Processing Logic sticky note           |
| Alternative to deleting files: move processed files to an archive folder instead of deleting to preserve originals.                                                  | Alternative Approaches sticky note           |
| Chatbot demo uses Google Gemini Chat Model with Qdrant vector retrieval for semantic question answering.                                                             | Chat Bot Demo sticky note                    |
| For troubleshooting infinite loops, check split node reset settings and ensure file deletion node works properly.                                                   | Troubleshooting sticky note                   |
| For performance, keep files under 10MB, monitor API usage, and clear Qdrant periodically to maintain performance.                                                    | Performance Tips sticky note                  |
| Document chunking uses recursive character splitter with 1500 tokens chunk size and 250 token overlap to balance context retention and processing load.             | Document Pipeline sticky note                 |
| Thanks to Jeremy Dawes, Jezweb, for workflow design and sharing: www.jezweb.com.au                                                                                   | Setup Requirements1 sticky note               |
| Workflow designed for continuous ingestion and building a semantic knowledge base searchable via AI chatbot interface.                                              | Workflow Overview sticky note                 |

---

**Disclaimer:** The provided text derives exclusively from an automated n8n workflow designed for legal and public data processing. It strictly respects content policies and contains no illegal or offensive material.