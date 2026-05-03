PDF RAG Agent with Telegram Chat & Auto-Ingestion from Google Drive

https://n8nworkflows.xyz/workflows/pdf-rag-agent-with-telegram-chat---auto-ingestion-from-google-drive-8860


# PDF RAG Agent with Telegram Chat & Auto-Ingestion from Google Drive

### 1. Workflow Overview

This workflow implements a **PDF Retrieval-Augmented Generation (RAG) Agent** integrated with **Telegram chat** and **automatic PDF ingestion from Google Drive**. The design allows users to interact with an AI agent via Telegram, querying knowledge extracted from ingested PDF documents. The documents are automatically fetched from Google Drive, processed, embedded, and stored in a PostgreSQL PGVector database for efficient retrieval.

The workflow is logically divided into the following blocks:

- **1.1 PDF Ingestion Pipeline**: Automatically lists PDF files from Google Drive, downloads, processes (splitting, embedding), and stores vector embeddings into the PostgreSQL PGVector store.
- **1.2 Telegram Chat Interface**: Handles Telegram message triggers, distinguishes between text queries and PDF file uploads, coordinating AI agent responses or ingestion approval flows.
- **1.3 AI Agent Query Processing**: Uses Azure OpenAI chat models with LangChain agents, querying the vector store to answer user queries based on embedded PDFs.
- **1.4 User Approval Workflow**: Manages ingestion approval via Telegram messages before processing PDF files uploaded by users.
- **1.5 Supporting Utilities**: Various helper nodes such as token splitters, memory buffers, and HTTP requests to send files or responses.

---

### 2. Block-by-Block Analysis

#### 2.1 PDF Ingestion Pipeline

**Overview:**  
This block automates the ingestion of PDF files from a Google Drive folder, processes them to create vector embeddings, and stores those embeddings in a PostgreSQL PGVector database for later retrieval.

**Nodes Involved:**  
- Run Ingestion (Manual Trigger)  
- List All File Names (Google Drive)  
- Loop Over Items (Split in Batches)  
- Download Corresponding File (Google Drive)  
- Send PDF File (HTTP Request)  
- Default Data Loader (LangChain Document Loader)  
- Token Splitter (LangChain Text Splitter)  
- Embeddings Mistral Cloud (LangChain Embeddings)  
- Postgres PGVector Store (LangChain Vector Store)

**Node Details:**

- **Run Ingestion**  
  - Type: Manual Trigger  
  - Role: Initiates the ingestion process manually.  
  - Inputs: None  
  - Outputs: Triggers listing of files from Google Drive.  
  - Failure Modes: None typical; user-initiated.

- **List All File Names**  
  - Type: Google Drive  
  - Role: Lists all files in a configured Google Drive folder, presumably PDFs.  
  - Configuration: Authenticated with Google Drive credentials; parameters set to list files in target folder.  
  - Inputs: Trigger from "Run Ingestion"  
  - Outputs: List of file metadata to batch processing node.  
  - Failure Modes: Authentication errors, API quota limits, empty folder.

- **Loop Over Items**  
  - Type: Split In Batches  
  - Role: Processes each file metadata item individually or in batches.  
  - Inputs: List of files from "List All File Names"  
  - Outputs: Each file metadata item to "Download Corresponding File"  
  - Failure Modes: Empty input, batch size misconfiguration.

- **Download Corresponding File**  
  - Type: Google Drive  
  - Role: Downloads the actual PDF file content by file ID.  
  - Configuration: Uses file ID from upstream node.  
  - Inputs: Single file metadata from "Loop Over Items"  
  - Outputs: Raw PDF data to "Send PDF File"  
  - Failure Modes: Missing file, download errors, permission denied.

- **Send PDF File**  
  - Type: HTTP Request  
  - Role: Forwards the downloaded PDF to an endpoint that processes ingestion (likely another service or workflow webhook).  
  - Configuration: HTTP POST with PDF file as payload.  
  - Inputs: PDF data from "Download Corresponding File"  
  - Outputs: Trigger next process; no direct outputs connected here.  
  - Failure Modes: Network errors, endpoint unavailable.

- **Default Data Loader**  
  - Type: LangChain Document Default Data Loader  
  - Role: Parses the PDF content into documents suitable for embedding.  
  - Configuration: Default loader capable of processing PDFs.  
  - Inputs: PDF file from ingestion endpoint or previous node  
  - Outputs: Document objects to "Token Splitter"  
  - Failure Modes: Unsupported file format, parsing errors.

- **Token Splitter**  
  - Type: LangChain TextSplitter TokenSplitter  
  - Role: Splits large documents into smaller token-sized chunks for embedding.  
  - Configuration: Token size and overlap parameters tuned to optimize embedding quality.  
  - Inputs: Documents from "Default Data Loader"  
  - Outputs: Text chunks to embedding node  
  - Failure Modes: Improper splitting parameters causing empty or oversized chunks.

- **Embeddings Mistral Cloud**  
  - Type: LangChain Embeddings Mistral Cloud  
  - Role: Converts text chunks into vector embeddings using Mistral Cloud service.  
  - Configuration: Authenticated via Mistral API credentials.  
  - Inputs: Text chunks from "Token Splitter"  
  - Outputs: Embeddings to "Postgres PGVector Store"  
  - Failure Modes: API rate limits, auth failure, network errors.

- **Postgres PGVector Store**  
  - Type: LangChain Vector Store PGVector  
  - Role: Stores vector embeddings for similarity search retrieval.  
  - Configuration: Connected to PostgreSQL with PGVector extension; configured with DB credentials and table specification.  
  - Inputs: Embeddings from "Embeddings Mistral Cloud"  
  - Outputs: Confirmation or empty output as terminal node  
  - Failure Modes: DB connection errors, write failures, indexing issues.

---

#### 2.2 Telegram Chat Interface

**Overview:**  
Handles Telegram updates via webhook triggers, distinguishing between text queries and PDF uploads. Manages user interactions including approval workflows for ingestion and AI conversation.

**Nodes Involved:**  
- Telegram Trigger  
- If Text Message Received (If node)  
- If PDF File Received (If node)  
- Approve Ingestion (Telegram)  
- If User Approved (If node)  
- User Dissapproved (Telegram)  
- Send Refusal Message (Telegram)  
- Download PDF File (Telegram)  
- Send Downloaded PDF File (HTTP Request)  
- Send Finished Message (Telegram)

**Node Details:**

- **Telegram Trigger**  
  - Type: Telegram Trigger  
  - Role: Webhook entry point for Telegram messages and updates.  
  - Configuration: Bound to Telegram bot token and webhook URL.  
  - Inputs: External Telegram events  
  - Outputs: Routes to "If Text Message Received" node  
  - Failure Modes: Webhook misconfiguration, bot offline.

- **If Text Message Received**  
  - Type: If (Conditional)  
  - Role: Branches flow based on whether the Telegram update contains text or a file.  
  - Configuration: Checks message type or content presence.  
  - Inputs: Telegram updates  
  - Outputs:  
    - True: "AI Agent2" for text queries  
    - False: "If PDF File Received" for file uploads  
  - Failure Modes: Misclassification, message format changes.

- **If PDF File Received**  
  - Type: If (Conditional)  
  - Role: Checks if an incoming Telegram message contains a PDF file.  
  - Inputs: Telegram update from previous node  
  - Outputs:  
    - True: "Approve Ingestion" node  
    - False: "Send Refusal Message"  
  - Failure Modes: File type misidentification.

- **Approve Ingestion**  
  - Type: Telegram  
  - Role: Sends Telegram message prompting user to approve ingestion of uploaded PDF.  
  - Inputs: PDF file detected  
  - Outputs: "If User Approved" node  
  - Failure Modes: Telegram API errors, user ignoring message.

- **If User Approved**  
  - Type: If (Conditional)  
  - Role: Checks user approval response message (e.g., buttons or text callback).  
  - Inputs: Response from "Approve Ingestion"  
  - Outputs:  
    - True: "Download PDF File" to proceed with ingestion  
    - False: "User Dissapproved" message node  
  - Failure Modes: Timeout on user response, invalid reply.

- **User Dissapproved**  
  - Type: Telegram  
  - Role: Sends a Telegram message to user confirming ingestion denial.  
  - Inputs: Negative approval from "If User Approved"  
  - Outputs: Terminal for this branch  
  - Failure Modes: Telegram API errors.

- **Download PDF File**  
  - Type: Telegram  
  - Role: Downloads PDF file sent by user from Telegram servers.  
  - Inputs: Approved PDF message  
  - Outputs: "Send Downloaded PDF File" HTTP Request  
  - Failure Modes: Telegram file URL invalid or expired.

- **Send Downloaded PDF File**  
  - Type: HTTP Request  
  - Role: Forwards the downloaded PDF to ingestion endpoint (likely the same endpoint used in automated ingestion).  
  - Inputs: PDF file content  
  - Outputs: "Send Finished Message"  
  - Failure Modes: Network errors, endpoint down.

- **Send Finished Message**  
  - Type: Telegram  
  - Role: Confirms ingestion completion to user.  
  - Inputs: Confirmation from ingestion endpoint  
  - Outputs: Terminal node  
  - Failure Modes: Telegram API errors.

---

#### 2.3 AI Agent Query Processing

**Overview:**  
Processes incoming user text queries from Telegram using Azure OpenAI chat models and LangChain agents, retrieving information from the vector store with embedded PDFs.

**Nodes Involved:**  
- AI Agent2 (LangChain Agent)  
- Azure OpenAI Chat Model3 (Language Model)  
- Postgres PGVector Store3 (Vector Store)  
- Embeddings Mistral Cloud3 (Embeddings)  
- Simple Memory (LangChain Memory Buffer)  
- Send a Text Message (Telegram)

**Node Details:**

- **AI Agent2**  
  - Type: LangChain Agent  
  - Role: Coordinates question answering by querying vector store and invoking language model.  
  - Configuration: Uses Azure OpenAI model and PGVector store as tools and memory.  
  - Inputs: Telegram text messages from "If Text Message Received"  
  - Outputs: Text responses to "Send a Text Message"  
  - Failure Modes: API limits, incorrect query format, vector store downtime.

- **Azure OpenAI Chat Model3**  
  - Type: LangChain Azure OpenAI Chat Model  
  - Role: Provides natural language generation capabilities via Azure OpenAI service.  
  - Configuration: Authenticated with Azure credentials; model and deployment specified.  
  - Inputs: Prompt from "AI Agent2"  
  - Outputs: Generated text to "AI Agent2"  
  - Failure Modes: API quota, authentication failure, model errors.

- **Postgres PGVector Store3**  
  - Type: LangChain Vector Store PGVector  
  - Role: Provides similarity search over embedded documents for the AI agent.  
  - Configuration: PostgreSQL connection parameters, table, index.  
  - Inputs: Query from "AI Agent2"  
  - Outputs: Relevant document vectors for answering.  
  - Failure Modes: DB connection failure.

- **Embeddings Mistral Cloud3**  
  - Type: LangChain Embeddings Mistral Cloud  
  - Role: Converts input queries to vectors for similarity search.  
  - Inputs: Text query from AI agent  
  - Outputs: Vector embeddings to PGVector store node  
  - Failure Modes: API errors.

- **Simple Memory**  
  - Type: LangChain Memory Buffer Window  
  - Role: Maintains short-term conversation context for the AI agent.  
  - Inputs: Conversation history  
  - Outputs: Context to "AI Agent2"  
  - Failure Modes: Memory overflow, context loss.

- **Send a Text Message**  
  - Type: Telegram  
  - Role: Sends AI-generated text responses back to Telegram users.  
  - Inputs: Text from "AI Agent2"  
  - Outputs: Terminal.  
  - Failure Modes: Telegram API errors.

---

#### 2.4 User Approval Workflow

**Overview:**  
Manages explicit user consent for PDF ingestion upon file upload in Telegram, ensuring ingestion only proceeds after approval.

**Nodes Involved:**  
- Approve Ingestion (Telegram)  
- If User Approved (If node)  
- User Dissapproved (Telegram)  

**Node Details:**

- Covered in 2.2 Telegram Chat Interface above.

---

#### 2.5 Supporting Utilities

**Overview:**  
Nodes that support the main flows such as memory buffers for maintaining chat context, additional vector stores and embeddings for dual agent configurations, and sticky notes for documentation.

**Nodes Involved:**  
- Simple Memory2 (LangChain Memory Buffer)  
- Postgres PGVector Store2 (Vector Store)  
- Embeddings Mistral Cloud2 (Embeddings)  
- AI Agent (LangChain Agent)  
- Azure OpenAI Chat Model1 (LangChain Azure OpenAI)  
- Sticky Notes (Documentation)

**Node Details:**

- **Simple Memory2**: Similar to Simple Memory but used for another agent instance.  
- **Postgres PGVector Store2**: Vector store for first AI Agent instance.  
- **Embeddings Mistral Cloud2**: Embeddings node for the same agent.  
- **AI Agent**: Another LangChain agent instance used for different chat trigger.  
- **Azure OpenAI Chat Model1**: Language model used by the first AI Agent.  
- **Sticky Notes**: Provide inline documentation; most are empty or disabled.

---

### 3. Summary Table

| Node Name                 | Node Type                          | Functional Role                              | Input Node(s)                     | Output Node(s)                   | Sticky Note                      |
|---------------------------|----------------------------------|----------------------------------------------|----------------------------------|---------------------------------|---------------------------------|
| Run Ingestion             | Manual Trigger                   | Starts PDF ingestion from Google Drive       | -                                | List All File Names              |                                 |
| List All File Names       | Google Drive                    | Lists PDF files in target folder              | Run Ingestion                    | Loop Over Items                  |                                 |
| Loop Over Items           | Split In Batches                | Processes files one by one                     | List All File Names              | Download Corresponding File      |                                 |
| Download Corresponding File| Google Drive                    | Downloads each PDF file                        | Loop Over Items                  | Send PDF File                   |                                 |
| Send PDF File             | HTTP Request                   | Sends PDF to ingestion endpoint                | Download Corresponding File      | Loop Over Items (next batch)     |                                 |
| Default Data Loader       | LangChain Document Loader      | Parses PDFs into documents                      | PDF ingest webhook              | Token Splitter                  |                                 |
| Token Splitter            | LangChain Text Splitter        | Splits documents into chunks for embeddings    | Default Data Loader              | Embeddings Mistral Cloud        |                                 |
| Embeddings Mistral Cloud  | LangChain Embeddings           | Generates vector embeddings                      | Token Splitter                  | Postgres PGVector Store         |                                 |
| Postgres PGVector Store   | LangChain Vector Store         | Stores embeddings in PostgreSQL PGVector       | Embeddings Mistral Cloud        | -                               |                                 |
| Telegram Trigger          | Telegram Trigger               | Entry point for Telegram messages               | -                              | If Text Message Received        |                                 |
| If Text Message Received  | If (Conditional)               | Branches between text message or file           | Telegram Trigger               | AI Agent2 / If PDF File Received|                                 |
| AI Agent2                 | LangChain Agent               | Processes text queries with AI agent            | If Text Message Received        | Send a Text Message             |                                 |
| Azure OpenAI Chat Model3  | LangChain Language Model      | Generates text responses via Azure OpenAI      | AI Agent2                      | AI Agent2                      |                                 |
| Postgres PGVector Store3  | LangChain Vector Store         | Vector store for AI Agent2                       | AI Agent2                      | AI Agent2                      |                                 |
| Embeddings Mistral Cloud3 | LangChain Embeddings           | Embeddings for AI Agent2                        | AI Agent2                      | Postgres PGVector Store3       |                                 |
| Simple Memory             | LangChain Memory Buffer        | Maintains AI Agent2 conversation context       | -                              | AI Agent2                      |                                 |
| Send a Text Message       | Telegram                       | Sends AI-generated replies to Telegram users   | AI Agent2                      | -                             |                                 |
| If PDF File Received      | If (Conditional)               | Checks if Telegram message contains a PDF      | If Text Message Received        | Approve Ingestion / Send Refusal|                                 |
| Approve Ingestion         | Telegram                       | Requests user approval for PDF ingestion       | If PDF File Received            | If User Approved               |                                 |
| If User Approved          | If (Conditional)               | Branches based on user approval response       | Approve Ingestion              | Download PDF File / User Dissapproved |                             |
| Download PDF File         | Telegram                       | Downloads PDF uploaded by user                  | If User Approved               | Send Downloaded PDF File       |                                 |
| Send Downloaded PDF File  | HTTP Request                  | Sends user PDF to ingestion endpoint            | Download PDF File              | Send Finished Message          |                                 |
| Send Finished Message     | Telegram                       | Confirms ingestion completion to user          | Send Downloaded PDF File       | -                             |                                 |
| User Dissapproved         | Telegram                       | Notifies user that ingestion was declined       | If User Approved (false branch)| -                             |                                 |
| AI Agent                 | LangChain Agent               | Alternative AI agent for chat                     | Chat Trigger                  | Send a Text Message             |                                 |
| Azure OpenAI Chat Model1  | LangChain Language Model      | Language model for AI Agent                       | AI Agent                      | AI Agent                      |                                 |
| Postgres PGVector Store2  | LangChain Vector Store         | Vector store for AI Agent                         | AI Agent                      | AI Agent                      |                                 |
| Embeddings Mistral Cloud2 | LangChain Embeddings           | Embeddings for AI Agent                           | AI Agent                      | Postgres PGVector Store2       |                                 |
| Simple Memory2            | LangChain Memory Buffer        | Maintains AI Agent conversation context          | -                            | AI Agent                      |                                 |
| Chat Trigger             | LangChain Chat Trigger         | Alternative chat webhook entry point              | -                            | AI Agent                      |                                 |
| Sticky Note(s)           | Sticky Note                   | Documentation and comments                        | -                            | -                             | Multiple sticky notes present (content mostly empty) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Manual Trigger Node**  
   - Type: Manual Trigger  
   - Name: "Run Ingestion"  
   - Purpose: To start ingestion manually.

2. **Add Google Drive Node to List Files**  
   - Type: Google Drive  
   - Name: "List All File Names"  
   - Configure with Google Drive OAuth2 credentials.  
   - Set parameters to list files in the target folder containing PDFs.  
   - Connect "Run Ingestion" → "List All File Names".

3. **Add Split in Batches Node**  
   - Type: SplitInBatches  
   - Name: "Loop Over Items"  
   - Connect "List All File Names" → "Loop Over Items".

4. **Add Google Drive Node to Download Files**  
   - Type: Google Drive  
   - Name: "Download Corresponding File"  
   - Configure to download a file by ID from "Loop Over Items" output.  
   - Connect "Loop Over Items" → "Download Corresponding File".

5. **Add HTTP Request Node to Send PDF for Ingestion**  
   - Type: HTTP Request  
   - Name: "Send PDF File"  
   - Configure POST request to ingestion webhook endpoint (URL to be set).  
   - Attach downloaded PDF file as payload.  
   - Connect "Download Corresponding File" → "Send PDF File".  
   - Connect "Send PDF File" → "Loop Over Items" (to continue batches).

6. **Set up Webhook for PDF Ingestion** (Outside this workflow or as a separate workflow)  
   - This webhook receives PDFs and triggers processing steps.

7. **Add LangChain Default Data Loader Node**  
   - Type: LangChain Document Default Data Loader  
   - Name: "Default Data Loader"  
   - Configure for PDF parsing defaults.  
   - Connect Webhook output → "Default Data Loader".

8. **Add Token Splitter Node**  
   - Type: LangChain Text Splitter TokenSplitter  
   - Name: "Token Splitter"  
   - Configure token size and overlap for chunking.  
   - Connect "Default Data Loader" → "Token Splitter".

9. **Add Embeddings Node**  
   - Type: LangChain Embeddings Mistral Cloud  
   - Name: "Embeddings Mistral Cloud"  
   - Configure API credentials for Mistral Cloud embeddings service.  
   - Connect "Token Splitter" → "Embeddings Mistral Cloud".

10. **Add Vector Store Node**  
    - Type: LangChain Vector Store PGVector  
    - Name: "Postgres PGVector Store"  
    - Configure PostgreSQL connection with PGVector extension.  
    - Connect "Embeddings Mistral Cloud" → "Postgres PGVector Store".

11. **Add Telegram Trigger Node**  
    - Type: Telegram Trigger  
    - Name: "Telegram Trigger"  
    - Configure with Telegram Bot Token and set webhook URL.  
    - Connect to "If Text Message Received".

12. **Add If Node to Detect Text Messages**  
    - Type: If  
    - Name: "If Text Message Received"  
    - Condition checks if incoming Telegram message contains text.  
    - Connect "Telegram Trigger" → "If Text Message Received".

13. **Add If Node to Detect PDF Files**  
    - Type: If  
    - Name: "If PDF File Received"  
    - Condition to detect if message contains PDF file.  
    - Connect "If Text Message Received" (false branch) → "If PDF File Received".

14. **Add Telegram Node to Request Approval**  
    - Type: Telegram  
    - Name: "Approve Ingestion"  
    - Sends message with inline buttons (Approve/Deny).  
    - Connect "If PDF File Received" (true branch) → "Approve Ingestion".

15. **Add If Node for User Approval**  
    - Type: If  
    - Name: "If User Approved"  
    - Checks user's approval response.  
    - Connect "Approve Ingestion" → "If User Approved".

16. **Add Telegram Node for Download PDF File**  
    - Type: Telegram  
    - Name: "Download PDF File"  
    - Downloads file from Telegram servers.  
    - Connect "If User Approved" (true branch) → "Download PDF File".

17. **Add HTTP Request Node for Sending Downloaded PDF**  
    - Type: HTTP Request  
    - Name: "Send Downloaded PDF File"  
    - Sends PDF to ingestion webhook endpoint.  
    - Connect "Download PDF File" → "Send Downloaded PDF File".

18. **Add Telegram Node to Confirm Completion**  
    - Type: Telegram  
    - Name: "Send Finished Message"  
    - Sends confirmation to user that ingestion is complete.  
    - Connect "Send Downloaded PDF File" → "Send Finished Message".

19. **Add Telegram Node to Notify Refusal**  
    - Type: Telegram  
    - Name: "Send Refusal Message"  
    - Notifies user if no PDF was detected or ingestion declined.  
    - Connect "If PDF File Received" (false branch) → "Send Refusal Message".

20. **Add Telegram Node to Notify User Disapproval**  
    - Type: Telegram  
    - Name: "User Dissapproved"  
    - Sends message if user denies ingestion approval.  
    - Connect "If User Approved" (false branch) → "User Dissapproved".

21. **Add LangChain Agent Nodes for Text Queries**  
    - **AI Agent2** (LangChain Agent)  
      - Configure with Azure OpenAI Chat Model3, Postgres PGVector Store3, Embeddings Mistral Cloud3, and Simple Memory.  
      - Connect "If Text Message Received" (true branch) → "AI Agent2".
    - **Azure OpenAI Chat Model3**  
      - Configure with Azure OpenAI credentials and deployment.  
      - Connect to "AI Agent2" as language model.
    - **Postgres PGVector Store3**  
      - Configure as vector store for AI Agent2.  
      - Connect to "AI Agent2" as similarity store.
    - **Embeddings Mistral Cloud3**  
      - Configure embeddings for AI Agent2.  
      - Connect to "Postgres PGVector Store3".
    - **Simple Memory**  
      - Configure memory buffer for conversation context.  
      - Connect to "AI Agent2".
    - **Send a Text Message**  
      - Telegram node to send AI responses.  
      - Connect "AI Agent2" output → "Send a Text Message".

22. **Optionally Set Up Second Chat Trigger and Agent**  
    - Configure "Chat Trigger", "AI Agent", "Azure OpenAI Chat Model1", "Postgres PGVector Store2", "Embeddings Mistral Cloud2", "Simple Memory2", and connected Telegram nodes similarly.

23. **Test All Connections and Credentials**  
    - Verify Google Drive OAuth2 credentials  
    - Verify Telegram bot token and webhook setup  
    - Verify Azure OpenAI and Mistral Cloud API keys  
    - Verify PostgreSQL PGVector database connection

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                            |
|-------------------------------------------------------------------------------------------------|------------------------------------------------------------|
| The workflow integrates LangChain nodes with Azure OpenAI and Mistral Cloud for embeddings.     | Use latest n8n LangChain nodes and ensure API keys are valid.|
| Google Drive nodes require OAuth2 credentials with file read permissions.                        | Ensure Google Drive API is enabled and credentials refreshed regularly. |
| Telegram bot webhook must be configured with a publicly accessible URL for updates.              | See Telegram Bot API documentation for webhook setup.       |
| PostgreSQL PGVector extension must be installed and configured for vector similarity searches.  | https://pgvector.org/                                       |
| The ingestion endpoint URL used in HTTP Request nodes must be reachable and handle PDF ingestion.| Usually a separate workflow or external service.            |
| LangChain Token Splitter node parameters (token size, overlap) should be tuned to use case.     | Avoid too large chunks causing API timeouts or too small losing context. |
| Azure OpenAI API usage may incur costs; monitor usage carefully.                                 | https://learn.microsoft.com/en-us/azure/cognitive-services/openai/overview |
| Mistral Cloud embeddings API credentials must be kept secure.                                   | https://mistral.ai/                                         |
| Telegram message approval uses inline buttons or text commands for user input.                   | Consider user experience for smooth approval flow.          |

---

**Disclaimer:**  
The provided description and analysis are based exclusively on a workflow automated with n8n, respecting all content policies. No illegal or protected content is included. All manipulated data is legal and public.