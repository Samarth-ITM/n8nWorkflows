IT Support Chatbot with Google Drive, Pinecone & Gemini | AI Doc Processing

https://n8nworkflows.xyz/workflows/it-support-chatbot-with-google-drive--pinecone---gemini---ai-doc-processing-3192


# IT Support Chatbot with Google Drive, Pinecone & Gemini | AI Doc Processing

### 1. Workflow Overview

This n8n workflow automates IT support document ingestion and enables instant, context-aware query resolution via conversational AI. It integrates **Google Drive** for document storage, **Pinecone** as a vector database for semantic search, and **Google Gemini/OpenRouter** as the AI language model to generate responses.

The workflow is logically divided into two main blocks:

- **1.1 Document Ingestion Workflow:**  
  Monitors a Google Drive folder for new support documents, downloads and extracts their text, cleans and splits the text into chunks, generates embeddings using Google Gemini, and stores these embeddings in Pinecone for fast retrieval.

- **1.2 Chat Query Workflow:**  
  Listens for incoming user queries, generates embeddings for the query, retrieves the most relevant document chunks from Pinecone, constructs a detailed prompt combining context and query, and sends it to an AI agent (OpenRouter Chat Model) to generate a clear, step-by-step answer.

---

### 2. Block-by-Block Analysis

#### 2.1 Document Ingestion Workflow

**Overview:**  
This block automates the ingestion of new IT support documents from Google Drive, processes the text content, and stores semantic embeddings in Pinecone to build a searchable knowledge base.

**Nodes Involved:**  
- Monitor Google Drive for New Files  
- Download File from Google Drive  
- Extract PDF Content  
- Clean and Normalize PDF Text (Code node)  
- Split Document Text into Chunks  
- Load Document Data for Processing  
- Generate Document Embeddings (Google Gemini)  
- Insert Document into Pinecone Vector Store

**Node Details:**

- **Monitor Google Drive for New Files**  
  - Type: Google Drive Trigger  
  - Role: Watches a specific Google Drive folder (`folderToWatch` set to folder ID `1RQvAHIw8cQbtwI9ZvdVV0k0x6TM6HZwP`) for new file creation events.  
  - Configuration: Trigger on file creation in the specified folder.  
  - Inputs: None (trigger node)  
  - Outputs: Emits metadata of newly created files.  
  - Edge cases: Permissions errors if folder access is revoked; missed triggers if polling fails.

- **Download File from Google Drive**  
  - Type: Google Drive  
  - Role: Downloads the file identified by the incoming file ID.  
  - Configuration: Uses file ID from trigger node output to download the file content.  
  - Inputs: File metadata from trigger node.  
  - Outputs: Binary file data for extraction.  
  - Edge cases: File not found, permission denied, network timeouts.

- **Extract PDF Content**  
  - Type: Extract from File  
  - Role: Extracts text content from the downloaded PDF file.  
  - Configuration: Operation set to `pdf`.  
  - Inputs: Binary PDF file from download node.  
  - Outputs: Extracted raw text in JSON format.  
  - Edge cases: Corrupted PDFs, unsupported formats, extraction failures.

- **Clean and Normalize PDF Text**  
  - Type: Code (JavaScript)  
  - Role: Cleans extracted text by removing line breaks, trimming spaces, and stripping special characters to normalize the content.  
  - Key expressions:  
    ```js
    const rawData = $json["text"];
    const cleanedData = rawData
      .replace(/(\r\n|\n|\r)/gm, " ")
      .trim()
      .replace(/[^\w\s]/gi, "");
    return { cleanedData };
    ```  
  - Inputs: Extracted text JSON.  
  - Outputs: JSON with cleaned text under `cleanedData`.  
  - Edge cases: Empty or malformed text input, regex failures.

- **Split Document Text into Chunks**  
  - Type: Recursive Character Text Splitter  
  - Role: Splits cleaned text into chunks of 3000 characters with 300 characters overlap to preserve context.  
  - Configuration: `chunkSize=3000`, `chunkOverlap=300`.  
  - Inputs: Cleaned text from code node.  
  - Outputs: Array of text chunks.  
  - Edge cases: Very short documents (less than chunk size), overlapping logic errors.

- **Load Document Data for Processing**  
  - Type: Document Default Data Loader  
  - Role: Prepares the text chunks for embedding generation and vector store insertion.  
  - Inputs: Text chunks from splitter node.  
  - Outputs: Document objects with metadata.  
  - Edge cases: Empty input arrays, data format inconsistencies.

- **Generate Document Embeddings (Google Gemini)**  
  - Type: Embeddings (Google Gemini)  
  - Role: Generates vector embeddings for each document chunk using Google Gemini's `models/text-embedding-004`.  
  - Inputs: Document chunks from loader node.  
  - Outputs: Embeddings vectors.  
  - Credentials: Google Palm API credentials required.  
  - Edge cases: API rate limits, invalid credentials, network issues.

- **Insert Document into Pinecone Vector Store**  
  - Type: Pinecone Vector Store  
  - Role: Inserts the generated embeddings into the Pinecone index named `n8n-rag-demo`.  
  - Configuration: Mode set to `insert`.  
  - Inputs: Embeddings from Google Gemini node.  
  - Outputs: Confirmation of insertion.  
  - Credentials: Pinecone API credentials required.  
  - Edge cases: Index not found, authentication failure, network errors.

---

#### 2.2 Chat Query Workflow

**Overview:**  
This block handles incoming user queries, retrieves relevant document context from Pinecone, constructs a prompt combining context and query, and generates a detailed AI response.

**Nodes Involved:**  
- Chat Message Trigger  
- Generate Query Embeddings (Google Gemini)  
- Retrieve Relevant Documents from Pinecone  
- Generate Chat Prompt with Context (Code node)  
- AI Agent  
- OpenRouter Chat Model Interface

**Node Details:**

- **Chat Message Trigger**  
  - Type: Chat Trigger  
  - Role: Listens for incoming chat messages (user queries).  
  - Inputs: None (trigger node)  
  - Outputs: User query text under `chatInput`.  
  - Edge cases: Webhook misconfiguration, unauthorized access.

- **Generate Query Embeddings (Google Gemini)**  
  - Type: Embeddings (Google Gemini)  
  - Role: Converts the user query text into a vector embedding for similarity search.  
  - Configuration: Uses `models/text-embedding-004`.  
  - Inputs: User query from chat trigger.  
  - Outputs: Query embedding vector.  
  - Credentials: Google Palm API credentials required.  
  - Edge cases: API limits, invalid input, network failures.

- **Retrieve Relevant Documents from Pinecone**  
  - Type: Pinecone Vector Store  
  - Role: Searches the Pinecone index `n8n-rag-demo` for document chunks most similar to the query embedding.  
  - Configuration: Mode set to `load`.  
  - Inputs: Query embedding from previous node.  
  - Outputs: Array of documents with similarity scores.  
  - Credentials: Pinecone API credentials required.  
  - Edge cases: Empty results, index unavailability, auth errors.

- **Generate Chat Prompt with Context**  
  - Type: Code (JavaScript)  
  - Role: Sorts retrieved documents by relevance, selects top 3, and constructs a detailed prompt combining document context and user query.  
  - Key expressions:  
    ```js
    const userQuery =  $('Chat Message Trigger').first().json.chatInput;
    let documents = items.map(item => ({
      pageContent: item.json.document.pageContent,
      score: item.json.score
    }));
    documents.sort((a, b) => b.score - a.score);
    const topDocuments = documents.slice(0, 3);
    const contextContent = topDocuments
      .map((doc, index) => `Document ${index + 1}:\n${doc.pageContent}`)
      .join("\n\n");
    const prompt = `Using the following context from documents:\n\n${contextContent}\n\nAnswer the following question:\n${userQuery}\n\nAnswer:`;
    return [{ json: { prompt } }];
    ```  
  - Inputs: Retrieved documents from Pinecone and user query.  
  - Outputs: Final prompt string for AI agent.  
  - Edge cases: Less than 3 documents returned, missing fields in documents.

- **OpenRouter Chat Model Interface**  
  - Type: Language Model Chat (OpenRouter)  
  - Role: Sends the constructed prompt to the OpenRouter Chat Model (`google/gemini-2.0-flash-exp:free`) to generate a response.  
  - Inputs: Prompt from code node.  
  - Outputs: AI-generated answer.  
  - Credentials: OpenRouter API credentials required.  
  - Edge cases: API rate limits, invalid credentials, model unavailability.

- **AI Agent**  
  - Type: LangChain AI Agent  
  - Role: Receives the AI response from OpenRouter and formats it with markdown for clarity and readability.  
  - Configuration: System message instructs the agent to respond with clear, concise, detailed answers using markdown formatting (headings, bullet points, code blocks, etc.).  
  - Inputs: Prompt text from code node or output from OpenRouter node (depending on connection).  
  - Outputs: Final formatted answer to user query.  
  - Edge cases: Formatting errors, empty responses.

---

### 3. Summary Table

| Node Name                          | Node Type                              | Functional Role                              | Input Node(s)                      | Output Node(s)                          | Sticky Note                                                                                     |
|-----------------------------------|--------------------------------------|----------------------------------------------|----------------------------------|---------------------------------------|------------------------------------------------------------------------------------------------|
| Monitor Google Drive for New Files| Google Drive Trigger                  | Watches Google Drive folder for new files    | None                             | Download File from Google Drive       |                                                                                                |
| Download File from Google Drive    | Google Drive                         | Downloads new file from Drive                 | Monitor Google Drive for New Files| Extract PDF Content                   |                                                                                                |
| Extract PDF Content                | Extract from File                    | Extracts text from downloaded PDF             | Download File from Google Drive  | Clean and Normalize PDF Text           |                                                                                                |
| Clean and Normalize PDF Text       | Code                                | Cleans extracted text (remove line breaks, special chars) | Extract PDF Content              | Insert Document into Pinecone Vector Store |                                                                                                |
| Split Document Text into Chunks    | Recursive Character Text Splitter   | Splits cleaned text into chunks                | Clean and Normalize PDF Text (indirect via Load Document Data for Processing) | Load Document Data for Processing |                                                                                                |
| Load Document Data for Processing  | Document Default Data Loader         | Prepares chunks for embedding and storage     | Split Document Text into Chunks  | Insert Document into Pinecone Vector Store |                                                                                                |
| Generate Document Embeddings (Google Gemini) | Embeddings (Google Gemini)           | Generates embeddings for document chunks      | Load Document Data for Processing| Insert Document into Pinecone Vector Store |                                                                                                |
| Insert Document into Pinecone Vector Store | Pinecone Vector Store                | Inserts embeddings into Pinecone index         | Generate Document Embeddings (Google Gemini), Clean and Normalize PDF Text | None                                  |                                                                                                |
| Chat Message Trigger              | Chat Trigger                        | Receives user chat queries                      | None                             | Retrieve Relevant Documents from Pinecone |                                                                                                |
| Generate Query Embeddings (Google Gemini) | Embeddings (Google Gemini)           | Generates embedding for user query             | Chat Message Trigger             | Retrieve Relevant Documents from Pinecone |                                                                                                |
| Retrieve Relevant Documents from Pinecone | Pinecone Vector Store                | Retrieves relevant document chunks by similarity | Generate Query Embeddings (Google Gemini), Chat Message Trigger | Generate Chat Prompt with Context     |                                                                                                |
| Generate Chat Prompt with Context  | Code                                | Builds prompt combining context and user query | Retrieve Relevant Documents from Pinecone | AI Agent                             |                                                                                                |
| OpenRouter Chat Model Interface    | Language Model Chat (OpenRouter)     | Sends prompt to OpenRouter Chat Model          | Generate Chat Prompt with Context| AI Agent (via ai_languageModel input) |                                                                                                |
| AI Agent                         | LangChain AI Agent                  | Formats AI response with markdown               | Generate Chat Prompt with Context, OpenRouter Chat Model Interface | None                                  |                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Google Drive Trigger node:**  
   - Type: Google Drive Trigger  
   - Configure to watch for `fileCreated` events in a specific folder (set `folderToWatch` to your Google Drive folder ID).  
   - Connect Google Drive OAuth2 credentials.

2. **Add a Google Drive node to download files:**  
   - Type: Google Drive  
   - Operation: `download`  
   - Set `fileId` to the ID from the trigger node (`={{ $json.id }}`).  
   - Connect Google Drive OAuth2 credentials.  
   - Connect output of Google Drive Trigger to this node.

3. **Add an Extract from File node:**  
   - Type: Extract from File  
   - Operation: `pdf`  
   - Input: Binary data from Google Drive download node.  
   - Connect output of Google Drive node to this node.

4. **Add a Code node to clean and normalize text:**  
   - Type: Code (JavaScript)  
   - Paste the following code:  
     ```js
     const rawData = $json["text"];
     const cleanedData = rawData
       .replace(/(\r\n|\n|\r)/gm, " ")
       .trim()
       .replace(/[^\w\s]/gi, "");
     return { cleanedData };
     ```  
   - Input: Extracted text JSON from previous node.  
   - Connect output of Extract PDF Content node to this node.

5. **Add a Recursive Character Text Splitter node:**  
   - Type: Recursive Character Text Splitter  
   - Parameters: `chunkSize=3000`, `chunkOverlap=300`  
   - Input: Cleaned text from Code node.  
   - Connect output of Code node to this node.

6. **Add a Document Default Data Loader node:**  
   - Type: Document Default Data Loader  
   - Input: Text chunks from splitter node.  
   - Connect output of Text Splitter node to this node.

7. **Add an Embeddings node (Google Gemini):**  
   - Type: Embeddings (Google Gemini)  
   - Model: `models/text-embedding-004`  
   - Input: Document chunks from loader node.  
   - Connect Google Palm API credentials.  
   - Connect output of Document Data Loader node to this node.

8. **Add a Pinecone Vector Store node (Insert mode):**  
   - Type: Pinecone Vector Store  
   - Mode: `insert`  
   - Pinecone Index: Select or enter your Pinecone index name (e.g., `n8n-rag-demo`).  
   - Connect Pinecone API credentials.  
   - Connect outputs of Embeddings node and Code node (for cleaned text) to this node.

---

9. **Create a Chat Message Trigger node:**  
   - Type: Chat Trigger  
   - Configure webhook or chat interface as needed.  
   - No credentials required.  

10. **Add an Embeddings node (Google Gemini) for query:**  
    - Type: Embeddings (Google Gemini)  
    - Model: `models/text-embedding-004`  
    - Input: User query from Chat Message Trigger (`chatInput`).  
    - Connect Google Palm API credentials.  
    - Connect output of Chat Message Trigger to this node.

11. **Add a Pinecone Vector Store node (Load mode):**  
    - Type: Pinecone Vector Store  
    - Mode: `load`  
    - Pinecone Index: Same as ingestion (`n8n-rag-demo`).  
    - Connect Pinecone API credentials.  
    - Connect output of Query Embeddings node to this node.

12. **Add a Code node to generate chat prompt with context:**  
    - Type: Code (JavaScript)  
    - Paste the following code:  
      ```js
      const userQuery =  $('Chat Message Trigger').first().json.chatInput;
      let documents = items.map(item => ({
        pageContent: item.json.document.pageContent,
        score: item.json.score
      }));
      documents.sort((a, b) => b.score - a.score);
      const topDocuments = documents.slice(0, 3);
      const contextContent = topDocuments
        .map((doc, index) => `Document ${index + 1}:\n${doc.pageContent}`)
        .join("\n\n");
      const prompt = `Using the following context from documents:\n\n${contextContent}\n\nAnswer the following question:\n${userQuery}\n\nAnswer:`;
      return [{ json: { prompt } }];
      ```  
    - Input: Retrieved documents from Pinecone node.  
    - Connect output of Pinecone Load node to this node.

13. **Add an OpenRouter Chat Model Interface node:**  
    - Type: Language Model Chat (OpenRouter)  
    - Model: `google/gemini-2.0-flash-exp:free`  
    - Connect OpenRouter API credentials.  
    - Connect output of Code node (prompt) to this node.

14. **Add an AI Agent node (LangChain AI Agent):**  
    - Type: LangChain AI Agent  
    - Parameters:  
      - Text: `={{ $json.prompt }}`  
      - System Message:  
        ```
        You are a knowledgeable and helpful assistant. Respond with clear, concise, and detailed answers formatted in markdown. Use proper markdown formatting including headings, bullet points, numbered lists, code blocks, and other structures as needed to improve readability and clarity.
        ```  
    - Connect output of OpenRouter Chat Model Interface node to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                 |
|---------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This template enables IT support teams to automate document ingestion and provide instant AI-powered support. | Workflow description and use case overview.                                                    |
| Ensure Google Drive, Pinecone, and AI API credentials are correctly configured before running the workflow.   | Prerequisites section.                                                                          |
| Adjust chunk size and overlap in the text splitter node to optimize context retrieval based on document size. | Text splitting parameters in Document Ingestion Workflow.                                      |
| The AI Agent node formats responses in markdown for better readability in chat interfaces.                    | AI Agent node system message configuration.                                                    |
| For more information on Pinecone vector store integration, visit https://www.pinecone.io/docs/                | External resource for Pinecone usage.                                                          |
| OpenRouter Chat Model supports Google Gemini models for advanced conversational AI.                           | OpenRouter node configuration and model selection.                                            |
| The workflow uses webhook-based chat triggers; ensure your n8n instance is accessible for incoming requests.   | Chat Message Trigger node setup.                                                               |

---

This documentation provides a complete, structured reference for understanding, reproducing, and maintaining the IT Support Chatbot workflow integrating Google Drive, Pinecone, and Google Gemini AI.