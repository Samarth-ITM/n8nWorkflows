Smart Customer Support System with GPT-4o, Gmail, Slack & Drive Knowledge Base'

https://n8nworkflows.xyz/workflows/smart-customer-support-system-with-gpt-4o--gmail--slack---drive-knowledge-base--4543


# Smart Customer Support System with GPT-4o, Gmail, Slack & Drive Knowledge Base'

### 1. Workflow Overview

This workflow, titled **"Smart Customer Support System with GPT-4o, Gmail, Slack & Drive Knowledge Base"**, is designed to automate and enhance customer support operations by integrating Gmail email monitoring, AI-driven email classification and response generation, Slack notifications for urgent issues, and a Google Drive-based knowledge base indexed with Pinecone vector search.

The workflow is logically divided into the following main blocks:

- **1.1 Email Input and Preprocessing:** Monitors incoming Gmail messages and preprocesses email content.
- **1.2 Email Classification:** Uses AI (GPT-4o) to classify emails and filter urgent messages.
- **1.3 AI Response Generation and Agent Interaction:** Generates intelligent responses with GPT-4o and manages AI agent workflow.
- **1.4 Slack Notification and Analytics Logging:** Sends alerts for urgent emails to Slack and logs analytics data to Google Sheets.
- **1.5 Knowledge Base Management:** Monitors Google Drive for updates, processes documents, creates embeddings, and stores them in Pinecone vector store for AI retrieval.

---

### 2. Block-by-Block Analysis

#### 2.1 Email Input and Preprocessing

- **Overview:**  
  This block listens for new emails in Gmail, extracts and preprocesses their content to prepare it for classification and AI processing.

- **Nodes Involved:**  
  - Gmail Inbox Monitor  
  - Email Content Preprocessor  

- **Node Details:**

  - **Gmail Inbox Monitor**  
    - Type: Trigger node for Gmail  
    - Role: Watches Gmail inbox for new incoming emails  
    - Configuration: Default inbox monitoring without filters (implicit)  
    - Inputs: None (trigger)  
    - Outputs: Email data forwarded to next node  
    - Edge Cases: Gmail API rate limits, authentication errors, or no new emails could cause inactivity  
    - Version: 1.2  

  - **Email Content Preprocessor**  
    - Type: Code node  
    - Role: Custom script to clean, parse, or extract relevant parts of the email (e.g., subject, body text)  
    - Configuration: Custom JavaScript or TypeScript code (not shown) likely removing signatures, quoted text, or HTML tags  
    - Inputs: Raw email data from Gmail Inbox Monitor  
    - Outputs: Cleaned and structured email content for classification  
    - Edge Cases: Malformed email content, encoding issues, or script errors could cause data loss or failure  
    - Version: 2  

#### 2.2 Email Classification

- **Overview:**  
  This block classifies incoming emails using AI to determine intent or category and filters urgent emails for immediate attention.

- **Nodes Involved:**  
  - Intelligent Email Classifier  
  - GPT-4o Language Model  
  - Urgent Email Filter  

- **Node Details:**

  - **Intelligent Email Classifier**  
    - Type: LangChain Text Classifier node  
    - Role: Uses GPT-4o to classify emails into categories such as support, sales, complaint, etc.  
    - Configuration: Uses GPT-4o language model as AI backend for classification  
    - Inputs: Preprocessed email content  
    - Outputs: Classification results forwarded to filtering and AI agent nodes  
    - Edge Cases: Classification ambiguity, timeouts, or API quota exhaustion  
    - Version: 1.1  

  - **GPT-4o Language Model**  
    - Type: LangChain OpenAI Chat Language Model  
    - Role: Provides the GPT-4o AI engine for classification tasks  
    - Configuration: OpenAI credentials, model parameters (not detailed)  
    - Inputs: Text data from Email Content Preprocessor, routed via Intelligent Email Classifier  
    - Outputs: Classified labels or intent tags  
    - Edge Cases: Authentication errors, API rate limits, or model unavailability  
    - Version: 1.2  

  - **Urgent Email Filter**  
    - Type: If node  
    - Role: Conditional node that checks if the email is marked urgent based on classification  
    - Configuration: Condition on classification output (e.g., if label == "urgent")  
    - Inputs: Classification output  
    - Outputs: Routes urgent messages to Slack Notification Hub, others to AI Agent  
    - Edge Cases: Misclassification leading to missed urgent emails or false positives  
    - Version: 2  

#### 2.3 AI Response Generation and Agent Interaction

- **Overview:**  
  This block manages the AI agent that generates customer support responses based on classified emails and interacts with vector stores for knowledge retrieval.

- **Nodes Involved:**  
  - AI Agent  
  - OpenAI Chat Model  
  - Pinecone Vector Store1  
  - Embeddings OpenAI1  

- **Node Details:**

  - **AI Agent**  
    - Type: LangChain Agent node  
    - Role: Coordinates AI response generation using language model and vector store for document retrieval  
    - Configuration: Connects to GPT-4o chat model and Pinecone vector store for knowledge base access  
    - Inputs: Classified email content and context from Pinecone Vector Store1  
    - Outputs: Generated AI response to be sent back or further processed  
    - Edge Cases: Missing knowledge base documents, API failures, vector store connection issues  
    - Version: 2  

  - **OpenAI Chat Model**  
    - Type: LangChain OpenAI Chat Language Model  
    - Role: Provides GPT-4o chat completion capabilities for AI Agent  
    - Configuration: OpenAI API credentials, model parameters  
    - Inputs: Prompt from AI Agent  
    - Outputs: AI-generated text responses  
    - Edge Cases: API limits, response latency, or malformed prompts  
    - Version: 1.2  

  - **Pinecone Vector Store1**  
    - Type: LangChain Pinecone Vector Store  
    - Role: Enables semantic search over indexed documents for AI Agent  
    - Configuration: Pinecone API credentials, index name, namespace  
    - Inputs: Embeddings from Embeddings OpenAI1, queries from AI Agent  
    - Outputs: Retrieved relevant documents or vectors  
    - Edge Cases: Connectivity issues, empty index, or embedding mismatches  
    - Version: 1.2  

  - **Embeddings OpenAI1**  
    - Type: LangChain OpenAI Embeddings  
    - Role: Creates vector embeddings for text to store in Pinecone  
    - Configuration: OpenAI API credentials, embedding model parameters  
    - Inputs: Text documents or data  
    - Outputs: Vector embeddings for Pinecone ingestion  
    - Edge Cases: API errors, input text too long or malformed  
    - Version: 1.2  

#### 2.4 Slack Notification and Analytics Logging

- **Overview:**  
  Urgent emails trigger Slack notifications for immediate human attention and log data into Google Sheets for analytics purposes.

- **Nodes Involved:**  
  - Slack Notification Hub  
  - Analytics Data Logger  

- **Node Details:**

  - **Slack Notification Hub**  
    - Type: HTTP Request node  
    - Role: Sends a formatted message to Slack webhook or API to notify team about urgent emails  
    - Configuration: Slack webhook URL or API endpoint, message formatting  
    - Inputs: Urgent email data from Urgent Email Filter  
    - Outputs: Confirmation or errors forwarded to next node  
    - Edge Cases: Slack webhook failures, network issues, malformed JSON payloads  
    - Version: 4.2  

  - **Analytics Data Logger**  
    - Type: Google Sheets node  
    - Role: Records email metadata or interaction analytics into a Google Sheets document for tracking  
    - Configuration: Google Sheets API credentials, spreadsheet ID, target sheet, row insertion  
    - Inputs: Data from Slack Notification Hub (or directly from urgent emails)  
    - Outputs: Logging confirmation  
    - Edge Cases: Google API quota limits, sheet access issues, incorrect data format  
    - Version: 4.5  

#### 2.5 Knowledge Base Management

- **Overview:**  
  This block watches Google Drive for new or updated documents, processes them into vector embeddings, and updates the Pinecone vector store, ensuring the AI agent has up-to-date knowledge.

- **Nodes Involved:**  
  - Google Drive Trigger  
  - Google Drive  
  - Pinecone Vector Store  
  - Default Data Loader  
  - Embeddings OpenAI  
  - Recursive Character Text Splitter  

- **Node Details:**

  - **Google Drive Trigger**  
    - Type: Trigger node for Google Drive  
    - Role: Listens for new or changed files in Google Drive  
    - Configuration: Monitors specific folder or entire Drive (not detailed)  
    - Inputs: None (trigger)  
    - Outputs: Drive file metadata to Google Drive node  
    - Edge Cases: Google API limits, missing permissions  
    - Version: 1  

  - **Google Drive**  
    - Type: Google Drive node  
    - Role: Retrieves document content from Drive for processing  
    - Configuration: OAuth2 credentials, file download settings  
    - Inputs: Trigger metadata from Google Drive Trigger  
    - Outputs: Raw document content to Pinecone Vector Store and processing nodes  
    - Edge Cases: File not found, access denied, file format unsupported  
    - Version: 3  

  - **Pinecone Vector Store**  
    - Type: LangChain Pinecone Vector Store  
    - Role: Stores embeddings for knowledge base documents  
    - Configuration: Pinecone credentials, index parameters  
    - Inputs: Embeddings from Embeddings OpenAI, documents from Default Data Loader  
    - Outputs: Confirmation of storage  
    - Edge Cases: Network errors, index conflicts, data inconsistencies  
    - Version: 1.1  

  - **Default Data Loader**  
    - Type: LangChain Document Default Data Loader  
    - Role: Loads and processes documents into smaller chunks or structured data for embedding  
    - Configuration: Accepts input from Recursive Character Text Splitter for text chunking  
    - Inputs: Chunked text from Recursive Character Text Splitter  
    - Outputs: Structured documents to Pinecone Vector Store  
    - Edge Cases: Unsupported document types, empty data  
    - Version: 1  

  - **Embeddings OpenAI**  
    - Type: LangChain OpenAI Embeddings  
    - Role: Generates vector embeddings for document chunks  
    - Configuration: OpenAI API credentials and embedding model  
    - Inputs: Document text chunks from Default Data Loader  
    - Outputs: Embeddings to Pinecone Vector Store  
    - Edge Cases: API failures, text too large  
    - Version: 1.2  

  - **Recursive Character Text Splitter**  
    - Type: LangChain Recursive Character Text Splitter  
    - Role: Breaks large texts into manageable chunks recursively for embedding  
    - Configuration: Parameters for chunk size and overlap (default or custom)  
    - Inputs: Raw document text from Google Drive  
    - Outputs: Chunks of text to Default Data Loader  
    - Edge Cases: Very large files causing performance issues, improper splits  
    - Version: 1  

---

### 3. Summary Table

| Node Name                   | Node Type                                 | Functional Role                            | Input Node(s)                    | Output Node(s)                  | Sticky Note                                                                                         |
|-----------------------------|-------------------------------------------|--------------------------------------------|---------------------------------|---------------------------------|---------------------------------------------------------------------------------------------------|
| Gmail Inbox Monitor          | n8n-nodes-base.gmailTrigger               | Watches incoming emails                      | —                               | Email Content Preprocessor       |                                                                                                   |
| Email Content Preprocessor   | n8n-nodes-base.code                       | Cleans and prepares email content           | Gmail Inbox Monitor              | Intelligent Email Classifier     |                                                                                                   |
| Intelligent Email Classifier | @n8n/n8n-nodes-langchain.textClassifier  | Classifies email intent using GPT-4o        | Email Content Preprocessor       | Urgent Email Filter, AI Agent    |                                                                                                   |
| GPT-4o Language Model        | @n8n/n8n-nodes-langchain.lmChatOpenAi    | Provides GPT-4o AI for classification       | — (used by Intelligent Email Classifier) | Intelligent Email Classifier    |                                                                                                   |
| Urgent Email Filter          | n8n-nodes-base.if                        | Filters urgent emails                        | Intelligent Email Classifier      | Slack Notification Hub           |                                                                                                   |
| Slack Notification Hub       | n8n-nodes-base.httpRequest                | Sends Slack alerts for urgent emails        | Urgent Email Filter              | Analytics Data Logger            |                                                                                                   |
| Analytics Data Logger        | n8n-nodes-base.googleSheets               | Logs email analytics data                     | Slack Notification Hub           | —                               |                                                                                                   |
| Setup Instructions           | n8n-nodes-base.stickyNote                  | Provides user guidance or reminders          | —                               | —                               |                                                                                                   |
| Gmail                       | n8n-nodes-base.gmailTool                   | Gmail API tool for sending emails or actions| —                               | AI Agent                       |                                                                                                   |
| Google Drive Trigger         | n8n-nodes-base.googleDriveTrigger          | Watches Google Drive for new/changed files  | —                               | Google Drive                   |                                                                                                   |
| Google Drive                | n8n-nodes-base.googleDrive                  | Retrieves files from Drive                    | Google Drive Trigger             | Pinecone Vector Store           |                                                                                                   |
| Pinecone Vector Store        | @n8n/n8n-nodes-langchain.vectorStorePinecone | Stores embeddings for knowledge base         | Embeddings OpenAI, Default Data Loader | —                             |                                                                                                   |
| Embeddings OpenAI            | @n8n/n8n-nodes-langchain.embeddingsOpenAi | Generates vector embeddings                   | Document chunks                  | Pinecone Vector Store           |                                                                                                   |
| Default Data Loader          | @n8n/n8n-nodes-langchain.documentDefaultDataLoader | Loads processed document chunks              | Recursive Character Text Splitter | Pinecone Vector Store           |                                                                                                   |
| Recursive Character Text Splitter | @n8n/n8n-nodes-langchain.textSplitterRecursiveCharacterTextSplitter | Splits large text documents into chunks      | Google Drive                    | Default Data Loader             |                                                                                                   |
| Pinecone Vector Store1       | @n8n/n8n-nodes-langchain.vectorStorePinecone | Vector store for AI Agent knowledge retrieval| Embeddings OpenAI1              | AI Agent                       |                                                                                                   |
| Embeddings OpenAI1           | @n8n/n8n-nodes-langchain.embeddingsOpenAi | Embeddings for AI Agent knowledge base       | —                               | Pinecone Vector Store1          |                                                                                                   |
| AI Agent                    | @n8n/n8n-nodes-langchain.agent             | Coordinates AI responses and vector search  | Intelligent Email Classifier, Pinecone Vector Store1 | —                               |                                                                                                   |
| OpenAI Chat Model           | @n8n/n8n-nodes-langchain.lmChatOpenAi      | GPT-4o chat model used by AI Agent           | —                               | AI Agent                       |                                                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Gmail Inbox Monitor" node**  
   - Type: Gmail Trigger  
   - Configure credentials with Gmail OAuth2  
   - Set to monitor Inbox for new emails (default)  
   - No filters needed initially  

2. **Add "Email Content Preprocessor"**  
   - Type: Code node  
   - Paste custom JavaScript to clean and extract email subject and body  
   - Connect output from Gmail Inbox Monitor to this node  

3. **Add "GPT-4o Language Model" node**  
   - Type: LangChain OpenAI Chat Model  
   - Configure OpenAI credentials with GPT-4o model selected  
   - Leave input/output connections for classification use  

4. **Add "Intelligent Email Classifier" node**  
   - Type: LangChain Text Classifier  
   - Set GPT-4o node as languageModel input  
   - Connect Email Content Preprocessor output to classifier input  

5. **Add "Urgent Email Filter" node**  
   - Type: If node  
   - Condition: If classification label equals "urgent" (adjust condition expression accordingly)  
   - Input: From Intelligent Email Classifier output  

6. **Add "Slack Notification Hub" node**  
   - Type: HTTP Request  
   - Configure Slack webhook URL in the HTTP request settings  
   - Connect "true" output of Urgent Email Filter to this node  

7. **Add "Analytics Data Logger" node**  
   - Type: Google Sheets  
   - Configure Google Sheets API credentials and target spreadsheet  
   - Connect Slack Notification Hub output to this node  

8. **Add "AI Agent" node**  
   - Type: LangChain Agent  
   - Connect input from Intelligent Email Classifier output (non-urgent path)  
   - Configure to use OpenAI Chat Model and Pinecone Vector Store1 nodes as inputs  

9. **Add "OpenAI Chat Model" node**  
   - Type: LangChain OpenAI Chat Model  
   - Configure OpenAI credentials for GPT-4o  
   - Connect to AI Agent as ai_languageModel input  

10. **Add "Pinecone Vector Store1" node**  
    - Type: LangChain Pinecone Vector Store  
    - Configure Pinecone API credentials and index name  
    - Connect Embeddings OpenAI1 output to this node as ai_embedding input  
    - Connect output to AI Agent as ai_tool input  

11. **Add "Embeddings OpenAI1" node**  
    - Type: LangChain OpenAI Embeddings  
    - Configure OpenAI credentials  
    - Connect input from relevant text source (e.g., AI Agent or documents)  

12. **Add "Google Drive Trigger" node**  
    - Type: Google Drive Trigger  
    - Configure OAuth2 credentials for Google Drive  
    - Set to monitor specific folder or entire Drive for changes  

13. **Add "Google Drive" node**  
    - Type: Google Drive  
    - Configure to download or read file contents based on trigger output  
    - Connect Google Drive Trigger to this node  

14. **Add "Recursive Character Text Splitter" node**  
    - Type: LangChain Recursive Character Text Splitter  
    - Configure chunk size and overlap as needed (defaults acceptable)  
    - Connect Google Drive file content to this node  

15. **Add "Default Data Loader" node**  
    - Type: LangChain Document Default Data Loader  
    - Connect output from Recursive Character Text Splitter  
    - Connect output to Pinecone Vector Store node  

16. **Add "Embeddings OpenAI" node**  
    - Type: LangChain OpenAI Embeddings  
    - Configure OpenAI credentials  
    - Connect Default Data Loader output to this node  

17. **Add "Pinecone Vector Store" node**  
    - Type: LangChain Pinecone Vector Store  
    - Configure Pinecone credentials and index  
    - Connect Embeddings OpenAI output to this node  

18. **Connect Data Flows Appropriately**  
    - Connect Gmail Inbox Monitor → Email Content Preprocessor → Intelligent Email Classifier  
    - Intelligent Email Classifier outputs to Urgent Email Filter and AI Agent  
    - Urgent Email Filter true path → Slack Notification Hub → Analytics Data Logger  
    - AI Agent uses OpenAI Chat Model and Pinecone Vector Store1  
    - Google Drive Trigger → Google Drive → Recursive Character Text Splitter → Default Data Loader → Embeddings OpenAI → Pinecone Vector Store  

19. **Set Credentials for APIs**  
    - Gmail OAuth2 for Gmail nodes  
    - OpenAI API keys for GPT-4o and Embeddings nodes  
    - Slack webhook URL in HTTP Request node  
    - Google Sheets OAuth2 for Analytics Logger node  
    - Google Drive OAuth2 for Drive nodes  
    - Pinecone API key and environment for Vector Store nodes  

20. **Test the Workflow**  
    - Send test emails and verify classification, Slack notification, AI response generation  
    - Add or update documents in Google Drive and verify embeddings update in Pinecone  

---

### 5. General Notes & Resources

| Note Content                                                                                 | Context or Link                                                                                         |
|----------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| GPT-4o model is used extensively for advanced language understanding and generation.         | OpenAI model, requires appropriate API access and credentials.                                        |
| Pinecone is used as a scalable vector database for semantic search within the knowledge base. | https://www.pinecone.io/                                                                               |
| Slack webhook integration requires a valid Slack app and incoming webhook URL.                | https://api.slack.com/messaging/webhooks                                                              |
| Google Drive Trigger watches for file changes to keep knowledge base up to date.              | Google Drive API with OAuth2 authentication                                                           |
| Google Sheets node logs analytics data for monitoring and KPI tracking.                       | Google Sheets API                                                                                      |
| Ensure API rate limits and quota usage are monitored to avoid workflow interruptions.         | OpenAI, Google APIs, Pinecone all have quotas that must be respected                                  |

---

*Disclaimer: The provided text is derived exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.*