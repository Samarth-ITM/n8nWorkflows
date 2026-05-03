Automated Lead Capture & Business Q&A with GPT-4o, Pinecone, and Google Sheets

https://n8nworkflows.xyz/workflows/automated-lead-capture---business-q-a-with-gpt-4o--pinecone--and-google-sheets-6080


# Automated Lead Capture & Business Q&A with GPT-4o, Pinecone, and Google Sheets

### 1. Workflow Overview

This n8n workflow, titled **"Automated Lead Capture & Business Q&A with GPT-4o, Pinecone, and Google Sheets"**, orchestrates two primary functionalities in a unified automation:

- **Part One: Automated Ingestion of Company Documents into a Vector Database**  
  It monitors a specified Google Drive folder for new files containing company information, processes and chunks them, creates embeddings using OpenAI, and stores these embeddings in a Pinecone vector database. This allows the company’s knowledge base to be structured and searchable for AI-powered queries.

- **Part Two: AI Chatbot for Customer Interaction and Lead Capture**  
  A publicly accessible chatbot interface receives customer messages, leverages the vector store to answer business-related questions, maintains conversational context with memory buffers, and collects potential lead information (name, email, phone, interests). Captured leads are appended automatically into a Google Sheet for follow-up.

---

The workflow consists of two main logical blocks:

- **1.1 Company Document Processing and Vector Storage**  
  Triggered by new files in Google Drive, it downloads, splits, embeds, and inserts data into Pinecone.

- **1.2 AI Chatbot Interaction and Lead Capture**  
  Triggered by incoming chat messages on a webhook, it uses LangChain tools to process queries with GPT-4o, access the Pinecone knowledge base, maintain conversation memory, and save leads into Google Sheets.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Company Document Processing and Vector Storage

**Overview:**  
This block automatically processes new company info files uploaded to a specific Google Drive folder. It downloads the files, splits the content into manageable chunks, generates embeddings using OpenAI, and inserts these vectors into a Pinecone vector database under the "Q&A" namespace for efficient retrieval.

**Nodes Involved:**  
- Google Drive Trigger  
- Google Drive  
- Recursive Character Text Splitter  
- Default Data Loader  
- Embeddings OpenAI  
- Pinecone Vector Store  
- Sticky Note (documentation)

**Node Details:**

- **Google Drive Trigger**  
  - *Type:* googleDriveTrigger  
  - *Role:* Polls a specified Google Drive folder every minute for new files.  
  - *Config:* Watches folder ID `1UDedbXYGMpviGjiHLbYpTXmERrnQYvy4` for file creation events.  
  - *Connections:* Output to Google Drive node.  
  - *Failures:* Possible API quota limits or connectivity issues.  
  - *Notes:* Polling mode; may introduce slight delay on new file detection.

- **Google Drive**  
  - *Type:* googleDrive (download operation)  
  - *Role:* Downloads the newly detected file from Google Drive using its file ID.  
  - *Config:* Uses the `id` from the trigger node.  
  - *Connections:* Output to Pinecone Vector Store and Default Data Loader (indirectly via splitter).  
  - *Failures:* File access permission errors, large file download timeouts.

- **Recursive Character Text Splitter**  
  - *Type:* textSplitterRecursiveCharacterTextSplitter  
  - *Role:* Splits the file content into chunks with 500 characters size and 20 characters overlap to optimize embedding quality.  
  - *Config:* Chunk size=500, overlap=20.  
  - *Connections:* Output to Default Data Loader.  
  - *Failures:* May fail on corrupted or non-text files.

- **Default Data Loader**  
  - *Type:* documentDefaultDataLoader  
  - *Role:* Prepares binary data for embedding generation.  
  - *Config:* Set to process binary data from the file.  
  - *Connections:* Output to Pinecone Vector Store.  
  - *Failures:* Unexpected file formats or encoding issues.

- **Embeddings OpenAI**  
  - *Type:* embeddingsOpenAi  
  - *Role:* Converts text chunks into vector embeddings using OpenAI embedding models.  
  - *Config:* Default OpenAI embedding settings (requires OpenAI API credentials).  
  - *Connections:* Output to Pinecone Vector Store embeddings input.  
  - *Failures:* API rate limits, invalid API credentials.

- **Pinecone Vector Store**  
  - *Type:* vectorStorePinecone  
  - *Role:* Inserts the generated embeddings into the Pinecone index named "goldsmith" under the namespace "Q&A".  
  - *Config:* Uses Pinecone index ID (credential-based), namespace "Q&A".  
  - *Connections:* Final node in this block.  
  - *Failures:* Pinecone service unavailability, invalid index or namespace.

- **Sticky Note (Part One Documentation)**  
  - Provides detailed explanation of the process, trigger, and purpose of this block for users.

---

#### 2.2 AI Chatbot Interaction and Lead Capture

**Overview:**  
This block handles incoming chat messages via a public webhook, utilizes GPT-4o to respond to customer inquiries about the business by accessing the vector store knowledge base, maintains conversational context through memory buffers, and collects lead information to be appended in a Google Sheet for business follow-up.

**Nodes Involved:**  
- When chat message received  
- AI Agent  
- OpenAI Chat Model  
- Window Buffer Memory  
- newCompany_q (Tool Vector Store)  
- Pinecone Vector Store1  
- Embeddings OpenAI1  
- OpenAI Chat Model1  
- Append row in sheet in Google Sheets  
- Sticky Note (documentation)

**Node Details:**

- **When chat message received**  
  - *Type:* chatTrigger  
  - *Role:* Public webhook entry point receiving customer chat messages.  
  - *Config:* Publicly accessible webhook ID, no authentication.  
  - *Connections:* Output to AI Agent node.  
  - *Failures:* Webhook connectivity, spam or malformed requests.

- **AI Agent**  
  - *Type:* agent (LangChain agent)  
  - *Role:* Central AI orchestrator that processes messages, leverages tools, and manages conversation flow.  
  - *Config:*  
    - System prompt defines role as a friendly assistant for "New Company".  
    - Uses two tools:  
      - `newCompany_q` (knowledge access)  
      - `sheets` (lead data storage)  
    - Requests lead info after business inquiries.  
  - *Connections:*  
    - Inputs: chat message, memory buffer, tools.  
    - Outputs: chatbot responses, leads data to Google Sheets.  
  - *Failures:* Model API errors, tool invocation issues, prompt misconfiguration.

- **OpenAI Chat Model**  
  - *Type:* lmChatOpenAi  
  - *Role:* GPT-4o language model providing natural language generation for the agent.  
  - *Config:* Model set to "gpt-4o".  
  - *Connections:* Used by AI Agent as the language model.  
  - *Failures:* API key validity, rate limits.

- **Window Buffer Memory**  
  - *Type:* memoryBufferWindow  
  - *Role:* Maintains conversation history with a window size of 12 messages for context continuity.  
  - *Config:* Context window length = 12.  
  - *Connections:* Memory input to AI Agent.  
  - *Failures:* Memory overflow or loss if workflow interrupted.

- **newCompany_q (Tool Vector Store)**  
  - *Type:* toolVectorStore  
  - *Role:* Provides AI Agent access to Pinecone vector store for knowledge retrieval.  
  - *Config:* Described as answering questions about the company.  
  - *Connections:*  
    - Input from Pinecone Vector Store1 embeddings and OpenAI Chat Model1.  
    - Output to AI Agent as a tool.  
  - *Failures:* Vector store access errors, query mismatches.

- **Pinecone Vector Store1**  
  - *Type:* vectorStorePinecone  
  - *Role:* Retrieves embeddings from Pinecone under namespace "Q&A" for newCompany_q tool.  
  - *Config:* Same index ID as in Part One, namespace "Q&A".  
  - *Connections:* Output to newCompany_q.  
  - *Failures:* Connectivity or index errors.

- **Embeddings OpenAI1**  
  - *Type:* embeddingsOpenAi  
  - *Role:* Generates embeddings for queries in the chatbot context.  
  - *Config:* Default OpenAI embedding options.  
  - *Connections:* Feeding Pinecone Vector Store1.  
  - *Failures:* API rate limits.

- **OpenAI Chat Model1**  
  - *Type:* lmChatOpenAi  
  - *Role:* Language model generating answers in the context of tool vector store queries.  
  - *Config:* Model "gpt-4o".  
  - *Connections:* Input to newCompany_q.  
  - *Failures:* API issues.

- **Append row in sheet in Google Sheets**  
  - *Type:* googleSheetsTool  
  - *Role:* Stores captured lead data (Name, Email, Phone, Interested In) into a Google Sheet.  
  - *Config:*  
    - Maps lead fields extracted by AI Agent variables.  
    - Requires Google Sheet document ID and sheet name (IDs configured via expressions).  
  - *Connections:* Called as a tool by AI Agent.  
  - *Failures:* Sheet permission errors, invalid IDs, malformed data.

- **Sticky Note (Part Two Documentation)**  
  - Details the chatbot’s role, trigger, memory, knowledge retrieval, lead capture, and business use case.

---

### 3. Summary Table

| Node Name                         | Node Type                            | Functional Role                                 | Input Node(s)                         | Output Node(s)                      | Sticky Note                                                                                                                   |
|----------------------------------|------------------------------------|------------------------------------------------|-------------------------------------|-----------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| Google Drive Trigger              | googleDriveTrigger                 | Detects new files in Google Drive folder        | —                                   | Google Drive                      | Part One – Company Info to Pinecone Vector Store: explanation of file ingestion and embedding pipeline                        |
| Google Drive                     | googleDrive (download)             | Downloads new files from Google Drive            | Google Drive Trigger                 | Pinecone Vector Store             | Part One – Company Info to Pinecone Vector Store                                                                              |
| Recursive Character Text Splitter| textSplitterRecursiveCharacterTextSplitter | Splits file content into manageable chunks      | — (implicitly connected)             | Default Data Loader               | Part One – Company Info to Pinecone Vector Store                                                                              |
| Default Data Loader              | documentDefaultDataLoader          | Prepares binary data for embedding generation    | Recursive Character Text Splitter    | Pinecone Vector Store             | Part One – Company Info to Pinecone Vector Store                                                                              |
| Embeddings OpenAI               | embeddingsOpenAi                  | Generates vector embeddings from text            | Default Data Loader                  | Pinecone Vector Store             | Part One – Company Info to Pinecone Vector Store                                                                              |
| Pinecone Vector Store           | vectorStorePinecone               | Inserts embeddings into Pinecone index            | Google Drive, Embeddings OpenAI     | —                                | Part One – Company Info to Pinecone Vector Store                                                                              |
| When chat message received      | chatTrigger                      | Public webhook to receive chat messages           | —                                   | AI Agent                         | Part Two – AI Chatbot for Lead Capture and Business Q&A: chatbot entry point                                                  |
| AI Agent                       | agent                            | Orchestrates AI chat, uses tools, collects leads | When chat message received           | Append row in sheet in Google Sheets | Part Two – AI Chatbot for Lead Capture and Business Q&A: main AI orchestrator                                                  |
| OpenAI Chat Model              | lmChatOpenAi                     | GPT-4o model for AI Agent language generation     | —                                   | AI Agent                         | Part Two – AI Chatbot for Lead Capture and Business Q&A                                                                       |
| Window Buffer Memory           | memoryBufferWindow               | Maintains conversational context (window size 12)| —                                   | AI Agent                         | Part Two – AI Chatbot for Lead Capture and Business Q&A                                                                       |
| newCompany_q                   | toolVectorStore                  | Provides company knowledge retrieval tool         | Pinecone Vector Store1, OpenAI Chat Model1 | AI Agent                         | Part Two – AI Chatbot for Lead Capture and Business Q&A                                                                       |
| Pinecone Vector Store1         | vectorStorePinecone              | Retrieves company embeddings for knowledge tool   | Embeddings OpenAI1                  | newCompany_q                     | Part Two – AI Chatbot for Lead Capture and Business Q&A                                                                       |
| Embeddings OpenAI1             | embeddingsOpenAi                 | Embeddings generation for chatbot queries         | —                                   | Pinecone Vector Store1            | Part Two – AI Chatbot for Lead Capture and Business Q&A                                                                       |
| OpenAI Chat Model1             | lmChatOpenAi                    | GPT-4o model to support knowledge retrieval tool  | —                                   | newCompany_q                     | Part Two – AI Chatbot for Lead Capture and Business Q&A                                                                       |
| Append row in sheet in Google Sheets | googleSheetsTool             | Stores captured lead data into Google Sheets      | AI Agent                           | —                                | Part Two – AI Chatbot for Lead Capture and Business Q&A                                                                       |
| Sticky Note                    | stickyNote                      | Documentation - Part One explanation                | —                                   | —                                | See Part One note content                                                                                                     |
| Sticky Note1                   | stickyNote                      | Documentation - Part Two explanation                | —                                   | —                                | See Part Two note content                                                                                                     |

---

### 4. Reproducing the Workflow from Scratch

To recreate this workflow in n8n, follow these steps:

#### Part One - Company Document Processing and Vector Storage

1. **Create Google Drive Trigger node**  
   - Type: `googleDriveTrigger`  
   - Set event to `fileCreated`  
   - Configure polling frequency to every 1 minute  
   - Set folder to watch by ID: `1UDedbXYGMpviGjiHLbYpTXmERrnQYvy4` (replace with your folder ID)  

2. **Create Google Drive node**  
   - Type: `googleDrive`  
   - Operation: `download`  
   - File ID: Use expression `{{$json["id"]}}` from trigger output  
   - Connect trigger node's output to this node's input  

3. **Create Recursive Character Text Splitter node**  
   - Type: `textSplitterRecursiveCharacterTextSplitter`  
   - Chunk size: 500 characters  
   - Chunk overlap: 20 characters  

4. **Create Default Data Loader node**  
   - Type: `documentDefaultDataLoader`  
   - Data type: binary  
   - Connect splitter output to this node  

5. **Create Embeddings OpenAI node**  
   - Type: `embeddingsOpenAi`  
   - Use default embedding model settings  
   - Ensure OpenAI API credentials are configured in n8n  

6. **Create Pinecone Vector Store node**  
   - Type: `vectorStorePinecone`  
   - Mode: `insert`  
   - Namespace: `Q&A`  
   - Set Pinecone index ID (requires Pinecone credentials and index creation in Pinecone platform)  
   - Connect outputs from Google Drive (file content), Default Data Loader, and Embeddings OpenAI appropriately (embedding input)  

7. **Connect nodes in sequence:**  
   Google Drive Trigger → Google Drive → Recursive Character Text Splitter → Default Data Loader → Embeddings OpenAI → Pinecone Vector Store

---

#### Part Two - AI Chatbot Interaction and Lead Capture

8. **Create When Chat Message Received node**  
   - Type: `chatTrigger`  
   - Configure webhook to be public (no auth)  
   - This will be the chatbot’s entry point  

9. **Create OpenAI Chat Model node** (for AI Agent language model)  
   - Type: `lmChatOpenAi`  
   - Model: `gpt-4o`  
   - Configure OpenAI API credentials  

10. **Create Window Buffer Memory node**  
    - Type: `memoryBufferWindow`  
    - Context window length: 12 messages  

11. **Create Embeddings OpenAI1 node**  
    - Type: `embeddingsOpenAi`  
    - Default settings for embedding queries  

12. **Create Pinecone Vector Store1 node**  
    - Type: `vectorStorePinecone`  
    - Namespace: `Q&A`  
    - Use same Pinecone index ID as Part One  

13. **Create OpenAI Chat Model1 node**  
    - Type: `lmChatOpenAi`  
    - Model: `gpt-4o`  
    - For knowledge retrieval  

14. **Create newCompany_q tool node**  
    - Type: `toolVectorStore`  
    - Description: "gives answers related to the company newCompany"  
    - Connect Pinecone Vector Store1 and OpenAI Chat Model1 as inputs  

15. **Create AI Agent node**  
    - Type: `agent`  
    - Configure system message with role and instructions:  
      ```
      ## Role:
      You are a friendly assistant for a Company named *New Company*.

      ## Task:
      You answer questions about the business.

      ## Details:
      You have access to various tools, which you use correctly.

      ## Tools:
      - newCompany_q: Use this tool to answer questions with knowledge about the company.
      - sheets: Use this tool to store contact information such as name, email, interested in, and phone number.

      After a customer asks about opening hours, products, location, or business information, ask them for their name, email, specific interests and phone number.
      ```
    - Set AI Agent to use:  
      - Language model input from OpenAI Chat Model  
      - Memory input from Window Buffer Memory  
      - AI tools input from newCompany_q and Google Sheets append node (see next step)  

16. **Create Append Row in Google Sheets node**  
    - Type: `googleSheetsTool`  
    - Operation: `append`  
    - Document ID: Your Google Sheets document ID (must be shared with the service account)  
    - Sheet Name: Target sheet name or ID  
    - Columns mapping:  
      - Name: expression to extract from AI Agent lead data variable `Name`  
      - Email: from AI Agent variable `Email`  
      - Phone: from AI Agent variable `Phone`  
      - Interested in: from AI Agent variable `Interestet_in` (note: fix spelling if needed)  
    - Connect AI Agent’s tool output to this node  

17. **Connect nodes in sequence:**  
    When chat message received → AI Agent → Append row in Google Sheets  
    AI Agent uses OpenAI Chat Model, Window Buffer Memory, and newCompany_q as inputs  
    newCompany_q uses Pinecone Vector Store1, Embeddings OpenAI1, OpenAI Chat Model1  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                    | Context or Link                                                                                                      |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| The workflow is divided into two main parts: company data ingestion into Pinecone and AI chatbot interaction with lead capture.                               | Workflow overview and sticky notes in n8n provide detailed explanations.                                            |
| The public AI chat webhook allows integration of the chatbot into websites or apps for 24/7 customer engagement.                                               | Enables customer support automation and lead generation.                                                            |
| OpenAI GPT-4o model is used for both embedding generation and chat completions, requiring proper API credentials and quota management.                        | https://platform.openai.com/docs/models/gpt-4o                                                                        |
| Pinecone vector database requires index setup and API credentials; ensure namespace "Q&A" is consistent across nodes.                                          | https://www.pinecone.io/docs/                                                                                         |
| Google Drive folder and Google Sheets document must have correct sharing permissions to allow the workflow to access files and append data.                   | Google Drive and Google Sheets API documentation                                                                     |
| Chunk size and overlap in text splitter chosen to balance embedding quality and performance; may be tuned for different document types or sizes.               |                                                                                                                     |

---

This completes the detailed reference documentation for the **Automated Lead Capture & Business Q&A with GPT-4o, Pinecone, and Google Sheets** n8n workflow.