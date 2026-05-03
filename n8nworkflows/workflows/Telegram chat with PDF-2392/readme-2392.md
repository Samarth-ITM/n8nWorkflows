Telegram chat with PDF

https://n8nworkflows.xyz/workflows/telegram-chat-with-pdf-2392


# Telegram chat with PDF

### 1. Workflow Overview

This workflow implements a Telegram chatbot that allows users to interact with PDF content via chat. It supports two main use cases:  
- **Uploading PDFs:** When a user sends a PDF document, the workflow extracts, processes, and stores the document's content into a Pinecone vector database for later retrieval.  
- **Asking Questions:** When a user sends a message without a document, the workflow searches the vector store for relevant information and uses an AI language model (Groq Chat Model) to generate a precise answer based on the stored PDF content.

**Logical Blocks:**

- **1.1 Input Reception and Document Check:** Receives Telegram messages, determines if the message contains a document (PDF).  
- **1.2 PDF Processing and Storage:** Downloads the PDF, sets its MIME type, splits text into chunks, generates embeddings, and inserts data into Pinecone vector store.  
- **1.3 Query Handling:** For non-document messages, retrieves relevant information from Pinecone using a vector retriever, then uses the Groq Chat model to generate an answer.  
- **1.4 Telegram Response:** Sends back responses or confirmation messages to the user, including error handling.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Document Check

**Overview:**  
This block listens for incoming Telegram updates (messages) and checks if the message contains a document (specifically a PDF). Depending on the presence of a document, it branches the workflow accordingly.

**Nodes Involved:**  
- Telegram Trigger  
- Check If is a document

**Node Details:**

- **Telegram Trigger**  
  - Type: Telegram Trigger node  
  - Role: Listens for new Telegram messages (updates type "message").  
  - Config: Monitors "message" updates; no additional filters.  
  - Inputs: None (trigger node).  
  - Outputs: Sends message data to "Check If is a document".  
  - Failures: Possible Telegram API connection or authorization errors.  
  - Credentials: Telegram API (OAuth token from BotFather).  

- **Check If is a document**  
  - Type: If node  
  - Role: Checks if the incoming message contains a document object (i.e., a PDF).  
  - Config: Condition tests if `message.document` exists in the incoming JSON.  
  - Inputs: From Telegram Trigger.  
  - Outputs:  
    - True branch: For messages with a document → continues to "Telegram get File".  
    - False branch: For messages without document → continues to "Question and Answer Chain".  
  - Failures: Expression evaluation errors if message format is unexpected.  
  - Version-specific: Uses version 2 of If node for improved expression handling.

---

#### 2.2 PDF Processing and Storage

**Overview:**  
Processes the received PDF document by downloading it, enforcing correct MIME type, splitting text into chunks, embedding the text, and inserting it into Pinecone vector store. Finally, confirms the upload to the user.

**Nodes Involved:**  
- Telegram get File  
- Change to application/pdf  
- Pinecone Vector Store (Insert mode)  
- Limit to 1  
- Telegram Response about Database  
- Recursive Character Text Splitter  
- Default Data Loader  
- Embeddings OpenAI  
- Pinecone Vector Store1  
- Vector Store Retriever  
- Stop and Error1

**Node Details:**

- **Telegram get File**  
  - Type: Telegram node (file resource)  
  - Role: Downloads the file from Telegram using the document's file_id.  
  - Config: Uses `message.document.file_id` from input JSON to fetch the file.  
  - Inputs: True branch of "Check If is a document".  
  - Outputs: Passes binary file data downstream.  
  - Failures: Telegram API errors, file not found, download timeout.  
  - Credentials: Telegram API.  

- **Change to application/pdf**  
  - Type: Code node (JavaScript)  
  - Role: Modifies binary metadata to ensure the file is identified as PDF (MIME type and filename extension).  
  - Config:  
    - Sets MIME type to `application/pdf`.  
    - Ensures filename ends with `.pdf`.  
    - Updates contentType in fileType metadata if present.  
  - Inputs: From "Telegram get File".  
  - Outputs: Modified binary data forwarded.  
  - Failures: Code execution errors if binary data structure is unexpected.  

- **Pinecone Vector Store (Insert mode)**  
  - Type: Langchain Pinecone vector store node  
  - Role: Inserts document chunks and embeddings into Pinecone index named "telegram".  
  - Config: Mode set to "insert"; uses Pinecone index "telegram".  
  - Inputs: From "Change to application/pdf" → "Default Data Loader" → "Embeddings OpenAI".  
  - Outputs: Confirmation of insertion.  
  - Failures: Pinecone API errors, authentication failures, indexing errors.  
  - Credentials: Pinecone API key.  

- **Limit to 1**  
  - Type: Limit node  
  - Role: Restricts output items to 1 (probably to limit response size).  
  - Inputs: From "Pinecone Vector Store".  
  - Outputs: To "Telegram Response about Database".  
  - Failures: Unlikely, but misconfiguration can cause unexpected behavior.  

- **Telegram Response about Database**  
  - Type: Telegram node (send message)  
  - Role: Sends a confirmation message back to the user indicating how many pages were saved in Pinecone.  
  - Config: Sends text with total pages info from metadata (`metadata.pdf.totalPages`).  
  - Inputs: From "Limit to 1".  
  - Outputs: None (terminal).  
  - Failures: Telegram API errors, invalid chat ID.  
  - Credentials: Telegram API.  

- **Recursive Character Text Splitter**  
  - Type: Langchain text splitter node  
  - Role: Splits PDF text into chunks of 3000 characters with 200 character overlaps for embedding.  
  - Config: chunkSize=3000, chunkOverlap=200.  
  - Inputs: From "Default Data Loader".  
  - Outputs: To "Embeddings OpenAI".  
  - Failures: Text processing errors if input text is malformed.  

- **Default Data Loader**  
  - Type: Langchain Document Data Loader  
  - Role: Loads binary PDF data and converts it to text for further processing.  
  - Config: dataType set to "binary".  
  - Inputs: From "Recursive Character Text Splitter".  
  - Outputs: To "Pinecone Vector Store".  
  - Failures: Failures in reading or parsing PDF.  

- **Embeddings OpenAI**  
  - Type: Langchain OpenAI Embeddings node  
  - Role: Generates vector embeddings from text chunks for Pinecone indexing.  
  - Config: Default OpenAI embedding model.  
  - Inputs: From "Recursive Character Text Splitter".  
  - Outputs: To "Pinecone Vector Store".  
  - Failures: OpenAI API errors, rate limits, invalid credentials.  
  - Credentials: OpenAI API key.  

- **Pinecone Vector Store1**  
  - Type: Langchain Pinecone vector store node  
  - Role: Used during retrieval, linked to "Vector Store Retriever". Not part of insert flow.  
  - Inputs: None for insert flow.  

- **Vector Store Retriever**  
  - Type: Langchain retriever node  
  - Role: Retrieves relevant vector chunks from Pinecone during query.  
  - Inputs: From "Pinecone Vector Store1" (retrieval flow).  

- **Stop and Error1**  
  - Type: Stop and Error node  
  - Role: Sends a controlled error message "An error occurred." if needed.  
  - Inputs: From "Telegram Response about Database" on error.  

---

#### 2.3 Query Handling

**Overview:**  
When the Telegram message does not contain a document, this block retrieves relevant information from Pinecone and generates an answer using the Groq Chat model.

**Nodes Involved:**  
- Question and Answer Chain  
- Vector Store Retriever  
- Pinecone Vector Store1  
- Groq Chat Model  
- Telegram Response  
- Stop and Error

**Node Details:**

- **Question and Answer Chain**  
  - Type: Langchain Retrieval QA Chain node  
  - Role: Uses the question text and retrieved documents to generate an answer.  
  - Config: Prompt text uses incoming message text `{{$json.message.text}}`.  
  - Inputs:  
    - From "Check If is a document" false branch (user question).  
    - Retriever input from "Vector Store Retriever".  
  - Outputs: To "Telegram Response".  
  - Failures: Model errors, prompt parsing issues.  

- **Vector Store Retriever**  
  - Type: Langchain retriever node  
  - Role: Retrieves relevant document vectors from Pinecone for the QA chain.  
  - Inputs: From "Pinecone Vector Store1".  
  - Outputs: To "Question and Answer Chain".  
  - Failures: Pinecone query failures, connection issues.  

- **Pinecone Vector Store1**  
  - Type: Langchain Pinecone vector store node  
  - Role: Vector store used for retrieval mode.  
  - Config: Uses Pinecone index "telegram".  
  - Inputs: From "Embeddings OpenAI" for inserting, from "Vector Store Retriever" for querying.  

- **Groq Chat Model**  
  - Type: Langchain Groq Chat Model node  
  - Role: Language model that consumes retrieved documents and forms a response.  
  - Config: Model set to "llama-3.1-70b-versatile".  
  - Inputs: From "Question and Answer Chain" as ai_languageModel input.  
  - Outputs: To "Question and Answer Chain" (loop in chain).  
  - Failures: Model API errors, token limits, authentication issues.  
  - Credentials: Groq API key.  

- **Telegram Response**  
  - Type: Telegram node (send message)  
  - Role: Sends the AI-generated answer back to Telegram chat.  
  - Config: Sends text from `{{$json.response.text}}`.  
  - Inputs: From "Question and Answer Chain".  
  - Outputs: On failure, triggers "Stop and Error".  
  - Failures: Telegram API errors, invalid chat id.  
  - Credentials: Telegram API.  

- **Stop and Error**  
  - Type: Stop and Error node  
  - Role: Sends a generic error message when the Telegram response fails.  
  - Inputs: From "Telegram Response" error output.  
  - Failure: Stops execution with message "An error occurred".  

---

#### 2.4 Telegram Response & Error Handling

**Overview:**  
Sends responses back to the user either confirming PDF upload or answering questions. Handles errors gracefully by stopping execution with clear error messages.

**Nodes Involved:**  
- Telegram Response  
- Telegram Response about Database  
- Stop and Error  
- Stop and Error1

**Node Details:**

- **Telegram Response**  
  - Sends AI-generated answers to user.  

- **Telegram Response about Database**  
  - Sends confirmation of successful PDF upload (number of pages saved).  

- **Stop and Error**  
  - Stops workflow and sends error message on Telegram response failure during Q&A.  

- **Stop and Error1**  
  - Stops workflow and sends error message on Telegram response failure during PDF upload confirmation.  

---

### 3. Summary Table

| Node Name                  | Node Type                              | Functional Role                           | Input Node(s)              | Output Node(s)                 | Sticky Note                                                                                                                   |
|----------------------------|--------------------------------------|------------------------------------------|----------------------------|-------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| Telegram Trigger            | Telegram Trigger                     | Receives Telegram messages                | None                       | Check If is a document          |                                                                                                                              |
| Check If is a document      | If                                  | Branches on presence of document          | Telegram Trigger            | Telegram get File (true), Question and Answer Chain (false) |                                                                                                                              |
| Telegram get File           | Telegram (file resource)             | Downloads PDF document                     | Check If is a document      | Change to application/pdf       |                                                                                                                              |
| Change to application/pdf   | Code                                | Ensures binary file metadata is PDF       | Telegram get File           | Pinecone Vector Store           |                                                                                                                              |
| Pinecone Vector Store       | Langchain Pinecone Vector Store     | Inserts embeddings and chunks into Pinecone| Change to application/pdf, Default Data Loader, Embeddings OpenAI | Limit to 1                    | # Load data into database\nFetch file from **Telegram**, split it into chunks and insert into **Pinecone** index, a message from **Telegram** will be sent just to let the user know that the process finished |
| Limit to 1                 | Limit                               | Limits output to 1 item                    | Pinecone Vector Store       | Telegram Response about Database|                                                                                                                              |
| Telegram Response about Database | Telegram (send message)            | Sends confirmation of PDF upload          | Limit to 1                 | Stop and Error1 (on error)      |                                                                                                                              |
| Recursive Character Text Splitter | Langchain Text Splitter           | Splits PDF text into chunks for embedding | Default Data Loader         | Embeddings OpenAI              |                                                                                                                              |
| Default Data Loader         | Langchain Document Data Loader      | Loads PDF binary and extracts text        | Recursive Character Text Splitter | Pinecone Vector Store          |                                                                                                                              |
| Embeddings OpenAI           | Langchain OpenAI Embeddings         | Generates embeddings from text chunks     | Recursive Character Text Splitter | Pinecone Vector Store          |                                                                                                                              |
| Pinecone Vector Store1      | Langchain Pinecone Vector Store     | Vector store for retrieval mode            | Embeddings OpenAI          | Vector Store Retriever         |                                                                                                                              |
| Vector Store Retriever      | Langchain Retriever                 | Retrieves relevant chunks from Pinecone   | Pinecone Vector Store1     | Question and Answer Chain      |                                                                                                                              |
| Question and Answer Chain   | Langchain Retrieval QA Chain        | Uses retrieved info and question to answer| Check If is a document (false branch), Vector Store Retriever | Telegram Response              | # Chat with Database\n\n1. **Receive** the incoming chat message.\n2. **Retrieve** relevant chunks from the _vector store_.\n3. **Pass** these chunks to the model.\n\nThe model will use the retrieved information to **formulate a precise response**. |
| Groq Chat Model             | Langchain Groq Chat Model           | Language model to generate answers        | Question and Answer Chain   | Question and Answer Chain      |                                                                                                                              |
| Telegram Response          | Telegram (send message)              | Sends Q&A answer to user                   | Question and Answer Chain   | Stop and Error (on error)       |                                                                                                                              |
| Stop and Error             | Stop and Error                      | Handles error on Telegram Response         | Telegram Response           | None                          |                                                                                                                              |
| Stop and Error1            | Stop and Error                      | Handles error on Telegram Response about DB| Telegram Response about Database | None                          |                                                                                                                              |
| Sticky Note                | Sticky Note                        | Notes on PDF data loading                   | None                       | None                          | # Load data into database\nFetch file from **Telegram**, split it into chunks and insert into **Pinecone** index, a message from **Telegram** will be sent just to let the user know that the process finished |
| Sticky Note1               | Sticky Note                        | Notes on Chat with Database logic           | None                       | None                          | # Chat with Database\n\n1. **Receive** the incoming chat message.\n2. **Retrieve** relevant chunks from the _vector store_.\n3. **Pass** these chunks to the model.\n\nThe model will use the retrieved information to **formulate a precise response**. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Configure to listen for "message" updates.  
   - Set Telegram credentials (API token from BotFather).  

2. **Create If Node "Check If is a document"**  
   - Condition: Check if `{{$json.message.document}}` exists (object exists).  
   - Connect Telegram Trigger main output to this node input.  

3. **On True Branch (message contains document):**  

   a. **Telegram get File Node**  
      - Type: Telegram  
      - Resource: File  
      - Parameter: `fileId` = `{{$json.message.document.file_id}}`  
      - Connect from "Check If is a document" true output.  
      - Set Telegram credentials.  

   b. **Code Node "Change to application/pdf"**  
      - JavaScript to modify MIME type and filename:  
        - Set `mimeType` to `application/pdf`  
        - Append `.pdf` to filename if missing  
        - Update `fileType.contentType` if exists  
      - Connect from Telegram get File output.  

   c. **Recursive Character Text Splitter Node**  
      - Type: Recursive Character Text Splitter  
      - Chunk Size: 3000  
      - Chunk Overlap: 200  
      - Connect from "Default Data Loader" output (see next).  

   d. **Default Data Loader Node**  
      - Type: Document Default Data Loader  
      - Data Type: binary  
      - Connect from "Change to application/pdf" output.  

   e. **Embeddings OpenAI Node**  
      - Type: Langchain OpenAI Embeddings  
      - Connect from Recursive Character Text Splitter output.  
      - Configure OpenAI credentials.  

   f. **Pinecone Vector Store Node (Insert Mode)**  
      - Mode: Insert  
      - Pinecone Index: "telegram"  
      - Connect from Embeddings OpenAI output and Default Data Loader output (ai_embedding and ai_document).  
      - Configure Pinecone credentials.  

   g. **Limit Node "Limit to 1"**  
      - No parameters needed  
      - Connect from Pinecone Vector Store output.  

   h. **Telegram Response about Database Node**  
      - Type: Telegram (send message)  
      - Text: `{{$json.metadata.pdf.totalPages}} pages saved on Pinecone`  
      - Chat ID: `{{$json.message.chat.id}}` from the original Telegram Trigger  
      - Connect from Limit node output.  
      - Configure Telegram credentials.  

   i. **Stop and Error1 Node**  
      - Error message: "An error occurred."  
      - Connect to the error output of Telegram Response about Database.  

4. **On False Branch (message does not contain document):**  

   a. **Pinecone Vector Store Node1 (Retrieval Mode)**  
      - Mode: Default (retrieval)  
      - Pinecone Index: "telegram"  
      - Configure Pinecone credentials.  

   b. **Vector Store Retriever Node**  
      - Connect from Pinecone Vector Store1 output.  

   c. **Groq Chat Model Node**  
      - Model: "llama-3.1-70b-versatile"  
      - Configure Groq API credentials.  

   d. **Question and Answer Chain Node**  
      - Prompt: Use incoming message text (`{{$json.message.text}}`) to search database.  
      - Connect retriever input from Vector Store Retriever.  
      - Connect language model input from Groq Chat Model.  
      - Connect from false branch of "Check If is a document".  

   e. **Telegram Response Node**  
      - Text: `{{$json.response.text}}`  
      - Chat ID: `{{$json.message.chat.id}}`  
      - Connect from Question and Answer Chain output.  
      - Configure Telegram credentials.  

   f. **Stop and Error Node**  
      - Error message: "An error occurred"  
      - Connect to error output of Telegram Response node.  

5. **Add Sticky Notes for Documentation**  
   - Add two sticky notes with content summarizing the two main workflow blocks (PDF upload & chat).  

6. **Set Workflow Settings**  
   - Timezone: America/Sao_Paulo  
   - Save manual executions enabled  
   - Execution order: v1  

7. **Test the Workflow**  
   - Send a PDF document to the Telegram bot, verify it processes and confirms pages saved.  
   - Send a question message, verify it replies with an AI-generated answer based on stored PDF content.  

---

### 5. General Notes & Resources

| Note Content                                                                                             | Context or Link                                                                                                  |
|--------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| To create a Telegram bot, use @BotFather in Telegram, follow `/newbot` command to get API token.       | Telegram Bot API setup instructions.                                                                            |
| You can replace the Groq chat model with any other supported AI model in Langchain nodes.              | Model flexibility note.                                                                                          |
| Pinecone vector store can be swapped with other vector stores like Supabase, Postgres, or QDrant.      | Vector store flexibility note.                                                                                   |
| This workflow uses Langchain nodes from n8n for embedding, document loading, splitting, and retrieval. | Langchain integration reference.                                                                                 |
| The workflow expects PDF files; other document types are not handled explicitly and may cause errors.  | Input data format note.                                                                                           |
| Error nodes provide graceful error messages on Telegram to improve user experience.                     | Error handling best practices.                                                                                    |
| Sticky notes in the workflow provide additional context on logic blocks.                               | Workflow documentation aid.                                                                                       |
| Workflow timezone is set to America/Sao_Paulo; adjust if needed for your region.                        | Workflow setting note.                                                                                            |

---

This completes the structured, comprehensive documentation of the "Telegram chat with PDF" n8n workflow. It facilitates understanding, reproduction, modification, and error anticipation for users and AI agents alike.