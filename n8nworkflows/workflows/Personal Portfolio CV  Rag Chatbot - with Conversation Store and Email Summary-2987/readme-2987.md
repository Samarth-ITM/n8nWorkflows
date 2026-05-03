Personal Portfolio CV  Rag Chatbot - with Conversation Store and Email Summary

https://n8nworkflows.xyz/workflows/personal-portfolio-cv--rag-chatbot---with-conversation-store-and-email-summary-2987


# Personal Portfolio CV  Rag Chatbot - with Conversation Store and Email Summary

### 1. Workflow Overview

This workflow implements a **Personal Portfolio CV RAG (Retrieval-Augmented Generation) Chatbot** with integrated conversation storage and daily email summaries. It is designed for individuals who want an interactive chatbot based on their CV/resume content and developers interested in RAG chatbot integration with conversation tracking.

The workflow is logically divided into three main blocks:

- **1.1 Training & Ingestion Block**: Automatically detects new or updated CV files in a specific Google Drive folder, downloads and processes them into chunks, generates embeddings using Google Gemini, and stores them in a Pinecone vector database for retrieval.

- **1.2 Chat & Conversation Handling Block**: Provides a webhook API endpoint for chat queries, uses the vector store to retrieve relevant CV information, runs a Google Gemini chat model agent to generate responses, and optionally stores chat conversations into a NocoDB database via another webhook.

- **1.3 Reporting & Email Summary Block**: Runs a daily scheduled trigger that fetches all conversations from NocoDB for the day, groups them by session and email, formats an HTML summary, and sends it via Gmail.

---

### 2. Block-by-Block Analysis

#### 2.1 Training & Ingestion Block

- **Overview:**  
  This block automates the ingestion of CV/resume documents from Google Drive. When a file is created or updated in a designated folder, it downloads the file, splits the content into chunks, generates embeddings using Google Gemini, and inserts the data into a Pinecone vector store index.

- **Nodes Involved:**  
  - Google Drive - Resume CV File Created  
  - Google Drive - Resume CV File Updated  
  - Download CV File From Google Drive  
  - CV content - Recursive Character Text Splitter  
  - CV File Data Loader  
  - Embeddings Google Gemini  
  - Pinecone - Vector Store forr CV Content  
  - Sticky Note (Setup instructions and training purpose)

- **Node Details:**

  1. **Google Drive - Resume CV File Created**  
     - Type: Google Drive Trigger  
     - Role: Watches for new files created in a specific Google Drive folder (folder ID configured).  
     - Config: Polls every minute, triggers on file creation in folder "SEAN-RAG-FOLDER".  
     - Inputs: None (trigger node)  
     - Outputs: File metadata JSON  
     - Credentials: Google Drive OAuth2  
     - Edge cases: Permissions errors, folder ID misconfiguration, API rate limits.

  2. **Google Drive - Resume CV File Updated**  
     - Type: Google Drive Trigger  
     - Role: Watches for file updates in the same folder as above.  
     - Config: Polls every minute, triggers on file update.  
     - Inputs: None (trigger node)  
     - Outputs: File metadata JSON  
     - Credentials: Google Drive OAuth2  
     - Edge cases: Same as above.

  3. **Download CV File From Google Drive**  
     - Type: Google Drive node  
     - Role: Downloads the file content using the file ID from trigger nodes.  
     - Config: Downloads file by ID, preserves original filename.  
     - Inputs: File metadata from triggers  
     - Outputs: Binary file data  
     - Credentials: Google Drive OAuth2  
     - Edge cases: File not found, permission denied, network errors.

  4. **CV content - Recursive Character Text Splitter**  
     - Type: Langchain Text Splitter  
     - Role: Splits the downloaded CV content into overlapping chunks for vectorization.  
     - Config: Chunk overlap set to 100 characters to preserve context.  
     - Inputs: Binary file data  
     - Outputs: Text chunks  
     - Edge cases: Large files causing memory issues, improper chunking if file format unsupported.

  5. **CV File Data Loader**  
     - Type: Langchain Document Data Loader  
     - Role: Converts binary data into text documents for processing.  
     - Config: Reads from specific binary field.  
     - Inputs: Binary file data  
     - Outputs: Text documents  
     - Edge cases: Unsupported file formats, decoding errors.

  6. **Embeddings Google Gemini**  
     - Type: Langchain Embeddings (Google Gemini)  
     - Role: Generates vector embeddings for the text chunks.  
     - Config: Uses model "models/text-embedding-004".  
     - Inputs: Text chunks from splitter  
     - Outputs: Embeddings vectors  
     - Credentials: Google Gemini (PaLM) API key  
     - Edge cases: API quota exceeded, network errors, invalid API key.

  7. **Pinecone - Vector Store forr CV Content**  
     - Type: Langchain Vector Store (Pinecone)  
     - Role: Inserts embeddings into Pinecone index named "seanrag".  
     - Config: Insert mode, index "seanrag".  
     - Inputs: Embeddings from Google Gemini and documents from data loader  
     - Outputs: Confirmation of insertion  
     - Credentials: Pinecone API key  
     - Edge cases: Index not found, API errors, quota limits.

  8. **Sticky Note (Setup Stage: Training Automatically)**  
     - Provides detailed setup instructions for Google Cloud, API keys, Pinecone, Google Drive folder, and n8n credentials.

---

#### 2.2 Chat & Conversation Handling Block

- **Overview:**  
  This block exposes a webhook API for chat queries. It uses the vector store to retrieve relevant CV information, runs a Google Gemini chat model agent to generate answers, and optionally saves conversation history to NocoDB via a separate webhook.

- **Nodes Involved:**  
  - Chat API - webhook  
  - Personal CV AI Agent Assistant  
  - Chat Memory - Window Buffer  
  - Resume lookup : Vector Store Tool  
  - Resume Vector Store (Retrieval)  
  - Resume Embeddings Google Gemini (retrieval)  
  - Resume Google Gemini Chat Model (retrieval)  
  - Chat API Response - Webhook  
  - Save Conversation API - Webhook  
  - Save Conversation - NocoDB  
  - Save Conversation API Webhook - Response  
  - Sticky Notes (Chatting Stage, Save Conversation API instructions)

- **Node Details:**

  1. **Chat API - webhook**  
     - Type: Webhook  
     - Role: Receives POST requests with chat input from frontend/backend.  
     - Config: Path "chat", POST method, responds with node output.  
     - Inputs: External HTTP request  
     - Outputs: JSON with chat input  
     - Edge cases: Unauthorized access, malformed requests.

  2. **Personal CV AI Agent Assistant**  
     - Type: Langchain Agent  
     - Role: Processes chat input, uses vector store tool to retrieve CV info, generates AI response.  
     - Config: System message defines persona "Sean Lon's assistant" with strict guidelines on data security and response style. Uses Google Gemini chat model.  
     - Inputs: Chat input from webhook, memory buffer, vector store tool  
     - Outputs: AI-generated chat response  
     - Credentials: Google Gemini (PaLM) API key  
     - Edge cases: API errors, missing data in vector store, prompt failures.

  3. **Chat Memory - Window Buffer**  
     - Type: Langchain Memory Buffer  
     - Role: Maintains conversational context using a sliding window buffer keyed by chat input.  
     - Config: Session key set to chat input text.  
     - Inputs: Chat input  
     - Outputs: Memory context for agent  
     - Edge cases: Memory overflow, session key collisions.

  4. **Resume lookup : Vector Store Tool**  
     - Type: Langchain Tool (Vector Store)  
     - Role: Retrieves top 5 relevant documents from Pinecone index "seanrag" for query.  
     - Config: topK=5, description provided.  
     - Inputs: Query from agent  
     - Outputs: Retrieved documents  
     - Credentials: Pinecone API key  
     - Edge cases: Empty results, API errors.

  5. **Resume Vector Store (Retrieval)**  
     - Type: Langchain Vector Store (Pinecone)  
     - Role: Supports retrieval operations for the vector store tool.  
     - Config: Uses index "seanrag".  
     - Inputs: Embeddings from Google Gemini (retrieval)  
     - Outputs: Retrieved vectors  
     - Credentials: Pinecone API key  
     - Edge cases: Index not found, API errors.

  6. **Resume Embeddings Google Gemini (retrieval)**  
     - Type: Langchain Embeddings  
     - Role: Generates embeddings for retrieval queries.  
     - Config: Model "models/text-embedding-004".  
     - Inputs: Query text  
     - Outputs: Embeddings vectors  
     - Credentials: Google Gemini API key  
     - Edge cases: API limits, network errors.

  7. **Resume Google Gemini Chat Model (retrieval)**  
     - Type: Langchain Chat Model  
     - Role: Supports the vector store tool with Google Gemini chat model.  
     - Config: Model "models/gemini-2.0-flash-exp".  
     - Inputs: Query and retrieved documents  
     - Outputs: Processed results for agent  
     - Credentials: Google Gemini API key  
     - Edge cases: API errors.

  8. **Chat API Response - Webhook**  
     - Type: Respond to Webhook  
     - Role: Sends the AI agent's response back to the chat API caller.  
     - Config: Responds with all incoming items.  
     - Inputs: AI agent output  
     - Outputs: HTTP response  
     - Edge cases: Response formatting errors.

  9. **Save Conversation API - Webhook**  
     - Type: Webhook  
     - Role: Receives POST requests to save conversation history from frontend apps.  
     - Config: Path "update-conversation", POST method, CORS allowed origins configured.  
     - Inputs: JSON with user, email, AI response, session ID  
     - Outputs: JSON data for saving  
     - Edge cases: Unauthorized origins, malformed data.

  10. **Save Conversation - NocoDB**  
      - Type: NocoDB node  
      - Role: Inserts conversation data into a NocoDB table "mk9sfu217ou392s".  
      - Config: Maps fields user, email, ai, sessionid from webhook body.  
      - Inputs: JSON from webhook  
      - Outputs: Confirmation of record creation  
      - Credentials: NocoDB API token  
      - Edge cases: API errors, data validation issues.

  11. **Save Conversation API Webhook - Response**  
      - Type: Respond to Webhook  
      - Role: Sends confirmation response to the save conversation API caller.  
      - Config: Responds with all incoming items.  
      - Inputs: Output from NocoDB node  
      - Outputs: HTTP response  
      - Edge cases: Response errors.

  12. **Sticky Notes (Chatting Stage)**  
      - Provide instructions on how to integrate chat API and save conversation API with frontend apps, including example curl commands.

---

#### 2.3 Reporting & Email Summary Block

- **Overview:**  
  This block runs a daily scheduled trigger to fetch all conversation records from NocoDB for the current day, groups them by unique session and email, formats an HTML email report, and sends it via Gmail.

- **Nodes Involved:**  
  - Schedule Trigger  
  - NocoDB - get all todays conversation  
  - Group Conversation By Unique Session + Email - Code  
  - Format HTML Display For email  
  - Send Report To Gmail  
  - Sticky Notes (Reporting instructions)

- **Node Details:**

  1. **Schedule Trigger**  
     - Type: Schedule Trigger  
     - Role: Triggers workflow daily at 18:00 (6 PM).  
     - Config: Interval trigger at hour 18.  
     - Inputs: None (trigger node)  
     - Outputs: Trigger event  
     - Edge cases: Timezone mismatches.

  2. **NocoDB - get all todays conversation**  
     - Type: NocoDB node  
     - Role: Retrieves all conversation records from NocoDB table "mk9sfu217ou392s" where date equals today.  
     - Config: Filter by today's date, return all records.  
     - Inputs: Trigger event  
     - Outputs: List of conversation records  
     - Credentials: NocoDB API token  
     - Edge cases: API errors, empty results.

  3. **Group Conversation By Unique Session + Email - Code**  
     - Type: Code node (JavaScript)  
     - Role: Groups conversation records by combined key of sessionid and email for aggregation.  
     - Inputs: List of conversation records  
     - Outputs: Object with grouped data  
     - Edge cases: Missing sessionid or email fields.

  4. **Format HTML Display For email**  
     - Type: HTML node  
     - Role: Generates an HTML formatted email body summarizing grouped conversations with timestamps, user and AI messages.  
     - Inputs: Grouped conversation data  
     - Outputs: HTML string  
     - Edge cases: HTML injection if data not sanitized.

  5. **Send Report To Gmail**  
     - Type: Gmail node  
     - Role: Sends the formatted HTML email to a configured recipient.  
     - Config: Recipient "lseanlon@gmail.com", subject includes current date, message body is HTML from previous node.  
     - Inputs: HTML email content  
     - Credentials: Gmail OAuth2  
     - Edge cases: Authentication errors, email delivery failures.

  6. **Sticky Notes (Reporting)**  
     - Provide instructions on configuring Gmail credentials, scheduler, and customizing email format.

---

### 3. Summary Table

| Node Name                             | Node Type                                | Functional Role                                  | Input Node(s)                          | Output Node(s)                           | Sticky Note                                                                                      |
|-------------------------------------|----------------------------------------|-------------------------------------------------|--------------------------------------|-----------------------------------------|-------------------------------------------------------------------------------------------------|
| Google Drive - Resume CV File Created | Google Drive Trigger                   | Trigger on new CV file creation in Drive folder | None                                 | Download CV File From Google Drive       | Setup instructions for Google Cloud, API keys, Pinecone, Google Drive folder, n8n credentials    |
| Google Drive - Resume CV File Updated | Google Drive Trigger                   | Trigger on CV file update in Drive folder       | None                                 | Download CV File From Google Drive       | Setup instructions (same as above)                                                              |
| Download CV File From Google Drive  | Google Drive                           | Download CV file content                          | Google Drive triggers                | Pinecone - Vector Store forr CV Content  |                                                                                                 |
| CV File Data Loader                 | Langchain Document Data Loader          | Load binary CV file data into text documents     | CV content - Recursive Character Text Splitter | Pinecone - Vector Store forr CV Content  |                                                                                                 |
| CV content - Recursive Character Text Splitter | Langchain Text Splitter               | Split CV text into chunks                          | Download CV File From Google Drive   | CV File Data Loader                      |                                                                                                 |
| Embeddings Google Gemini            | Langchain Embeddings                    | Generate embeddings for CV chunks                 | CV content - Recursive Character Text Splitter | Pinecone - Vector Store forr CV Content  |                                                                                                 |
| Pinecone - Vector Store forr CV Content | Langchain Vector Store (Pinecone)     | Insert embeddings into Pinecone index             | Embeddings Google Gemini, CV File Data Loader | None                                    |                                                                                                 |
| Chat API - webhook                  | Webhook                                | Receive chat input API requests                    | External HTTP request                | Personal CV AI Agent Assistant           | Chat API integration instructions with example curl command                                    |
| Personal CV AI Agent Assistant      | Langchain Agent                        | Generate AI chat responses based on CV data       | Chat API - webhook, Chat Memory - Window Buffer, Resume lookup : Vector Store Tool | Chat API Response - Webhook              |                                                                                                 |
| Chat Memory - Window Buffer         | Langchain Memory Buffer                 | Maintain chat context memory                        | Chat API - webhook                   | Personal CV AI Agent Assistant           |                                                                                                 |
| Resume lookup : Vector Store Tool   | Langchain Tool (Vector Store)           | Retrieve relevant CV documents from Pinecone      | Resume Vector Store (Retrieval)      | Personal CV AI Agent Assistant           |                                                                                                 |
| Resume Vector Store (Retrieval)     | Langchain Vector Store (Pinecone)       | Support retrieval operations                        | Resume Embeddings Google Gemini (retrieval) | Resume lookup : Vector Store Tool        |                                                                                                 |
| Resume Embeddings Google Gemini (retrieval) | Langchain Embeddings                    | Generate embeddings for retrieval queries          | Resume Google Gemini Chat Model (retrieval) | Resume Vector Store (Retrieval)          |                                                                                                 |
| Resume Google Gemini Chat Model (retrieval) | Langchain Chat Model                   | Support vector store tool with chat model          | Resume lookup : Vector Store Tool    | Resume Embeddings Google Gemini (retrieval) |                                                                                                 |
| Chat API Response - Webhook         | Respond to Webhook                      | Send AI chat response back to API caller           | Personal CV AI Agent Assistant       | None                                    |                                                                                                 |
| Save Conversation API - Webhook     | Webhook                                | Receive conversation save requests from frontend   | External HTTP request                | Save Conversation - NocoDB               | Save conversation API integration instructions with example curl command                       |
| Save Conversation - NocoDB          | NocoDB                                 | Store conversation data in NocoDB                   | Save Conversation API - Webhook      | Save Conversation API Webhook - Response |                                                                                                 |
| Save Conversation API Webhook - Response | Respond to Webhook                      | Confirm save conversation API request               | Save Conversation - NocoDB           | None                                    |                                                                                                 |
| Schedule Trigger                   | Schedule Trigger                       | Trigger daily report generation at 18:00           | None                                 | NocoDB - get all todays conversation    | Reporting instructions for email summary setup                                                |
| NocoDB - get all todays conversation | NocoDB                                 | Retrieve all conversations from today               | Schedule Trigger                    | Group Conversation By Unique Session + Email - Code |                                                                                                 |
| Group Conversation By Unique Session + Email - Code | Code (JavaScript)                      | Group conversations by session and email            | NocoDB - get all todays conversation | Format HTML Display For email             |                                                                                                 |
| Format HTML Display For email       | HTML                                   | Format grouped conversations into HTML email body  | Group Conversation By Unique Session + Email - Code | Send Report To Gmail                     |                                                                                                 |
| Send Report To Gmail                | Gmail                                  | Send daily conversation summary email               | Format HTML Display For email        | None                                    |                                                                                                 |
| Sticky Note1                       | Sticky Note                            | Setup instructions for Google Cloud, Pinecone, Drive, n8n | None                                 | None                                    | Setup steps for Google Cloud, API keys, Pinecone, Google Drive, n8n credentials                 |
| Sticky Note3                       | Sticky Note                            | Instructions for optional conversation save API    | None                                 | None                                    | Save conversation API integration instructions with example curl command                       |
| Sticky Note4                       | Sticky Note                            | Instructions for daily email report setup           | None                                 | None                                    | Reporting instructions for email summary setup                                                |
| Sticky Note5                       | Sticky Note                            | Instructions for chat API integration                | None                                 | None                                    | Chat API integration instructions with example curl command                                    |
| Sticky Note9                       | Sticky Note                            | Label "TRAINING" section                             | None                                 | None                                    |                                                                                                 |
| Sticky Note10                      | Sticky Note                            | Label "CHATTING" section                             | None                                 | None                                    |                                                                                                 |
| Sticky Note11                      | Sticky Note                            | Label "REPORTING" section                            | None                                 | None                                    |                                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Drive Folder and Upload CV**  
   - Create a folder in Google Drive to store your CV/resume files.  
   - Note the folder ID for configuration.

2. **Configure Credentials in n8n**  
   - Google Drive OAuth2 credential with access to your Drive folder.  
   - Google Gemini (PaLM) API key credential for embeddings and chat models.  
   - Pinecone API key credential for vector database operations.  
   - NocoDB API token credential for conversation storage.  
   - Gmail OAuth2 credential for sending email reports.

3. **Create Pinecone Index**  
   - Create a Pinecone index named `seanrag` (or your preferred name).  
   - Ensure the index is accessible with your API key.

4. **Set Up Google Drive Triggers**  
   - Add two Google Drive Trigger nodes:  
     - One for `fileCreated` event on your CV folder.  
     - One for `fileUpdated` event on the same folder.  
   - Configure polling interval to every minute.

5. **Download CV File Node**  
   - Add a Google Drive node to download files by file ID from the triggers.  
   - Configure to preserve original filename.

6. **Text Splitting and Data Loading**  
   - Add a Recursive Character Text Splitter node to split the downloaded file content into chunks with 100 characters overlap.  
   - Add a Document Data Loader node to convert binary data to text documents.

7. **Generate Embeddings**  
   - Add an Embeddings node using Google Gemini with model `models/text-embedding-004`.  
   - Connect the text splitter output to this node.

8. **Insert into Pinecone Vector Store**  
   - Add a Pinecone Vector Store node in insert mode.  
   - Configure to use the `seanrag` index.  
   - Connect embeddings and document loader outputs to this node.

9. **Create Chat API Webhook**  
   - Add a Webhook node with path `chat`, method POST.  
   - Configure to respond with node output.

10. **Add Chat Memory Buffer**  
    - Add a Memory Buffer node with session key set to the chat input text.

11. **Add Langchain Agent Node**  
    - Add an Agent node configured with:  
      - System message describing "Sean Lon's assistant" persona and instructions.  
      - Use Google Gemini Chat Model (`models/gemini-2.0-flash`).  
      - Use the vector store tool connected to Pinecone retrieval nodes.  
      - Connect memory buffer as AI memory.  
      - Input text from webhook body `chatInput`.

12. **Add Vector Store Retrieval Nodes**  
    - Add Pinecone Vector Store node for retrieval mode using `seanrag` index.  
    - Add Embeddings node for retrieval queries using Google Gemini embeddings.  
    - Add Google Gemini Chat Model node for retrieval support.

13. **Add Vector Store Tool Node**  
    - Add a Vector Store Tool node configured to retrieve top 5 documents from `seanrag`.  
    - Connect retrieval Pinecone and embeddings nodes.

14. **Connect Agent to Vector Store Tool**  
    - Connect the Vector Store Tool node to the Agent node as a tool.

15. **Add Respond to Webhook Node**  
    - Add a Respond to Webhook node connected to the Agent node to send chat responses back.

16. **Create Save Conversation API Webhook**  
    - Add a Webhook node with path `update-conversation`, method POST.  
    - Configure allowed origins for CORS as needed.

17. **Add NocoDB Node to Save Conversation**  
    - Add a NocoDB node configured to insert into your conversation table.  
    - Map fields: user, email, ai, sessionid from webhook body.

18. **Add Respond to Webhook Node for Save Conversation**  
    - Add a Respond to Webhook node connected to NocoDB node to confirm save.

19. **Create Daily Schedule Trigger**  
    - Add a Schedule Trigger node set to trigger daily at 18:00.

20. **Add NocoDB Node to Retrieve Today's Conversations**  
    - Add a NocoDB node configured to get all records where date equals today.

21. **Add Code Node to Group Conversations**  
    - Add a Code node with JavaScript to group conversations by sessionid and email.

22. **Add HTML Node to Format Email**  
    - Add an HTML node to format grouped conversations into an HTML email body.

23. **Add Gmail Node to Send Email**  
    - Add a Gmail node configured to send email to your address with the formatted HTML content.  
    - Set subject to include current date.

24. **Connect all nodes according to the logical flow described above.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                          | Context or Link                                                                                      |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Setup steps include creating a Google Cloud project, enabling Vertex AI API, obtaining Google AI API key, creating Pinecone account and index, setting up Google Drive folder, and configuring credentials in n8n.                                                                                                                                                                       | Sticky Note1                                                                                       |
| Chat API endpoint allows frontend/backend to send chat queries and receive AI responses based on CV vector store. Example curl command provided.                                                                                                                                                                                                                                     | Sticky Note5                                                                                       |
| Save Conversation API endpoint is optional and decoupled, allowing frontend apps to save conversation history to NocoDB with control over UI events. Example curl command provided.                                                                                                                                                                                                   | Sticky Note3                                                                                       |
| Daily email report scheduler fetches all conversations from NocoDB for the day, groups by session and email, formats HTML, and sends email via Gmail. Instructions to customize scheduler and email format included.                                                                                                                                                                     | Sticky Note4                                                                                       |
| The AI agent persona is carefully crafted to represent "Sean Lon" with background, skills, and response guidelines to ensure concise, relevant answers without disclosing original prompts or sensitive data.                                                                                                                                                                          | Personal CV AI Agent Assistant node parameters                                                    |
| Frontend integration requires calling the webhook endpoints (`chat` and optionally `update-conversation`) with appropriate JSON payloads and handling responses.                                                                                                                                                                                                                     | Sticky Notes 3 and 5                                                                              |
| Pinecone index name `seanrag` is used throughout but can be customized; ensure consistency in all nodes.                                                                                                                                                                                                                                                                               | Multiple sticky notes and node configurations                                                    |
| Gmail OAuth2 credentials must be configured properly to allow sending emails from n8n.                                                                                                                                                                                                                                                                                                 | Send Report To Gmail node                                                                          |

---

This document provides a detailed, structured reference for understanding, reproducing, and modifying the Personal Portfolio CV RAG Chatbot workflow with conversation storage and email summary features.