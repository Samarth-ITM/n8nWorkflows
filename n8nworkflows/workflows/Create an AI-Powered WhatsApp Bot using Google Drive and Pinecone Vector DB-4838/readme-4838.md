Create an AI-Powered WhatsApp Bot using Google Drive and Pinecone Vector DB

https://n8nworkflows.xyz/workflows/create-an-ai-powered-whatsapp-bot-using-google-drive-and-pinecone-vector-db-4838


# Create an AI-Powered WhatsApp Bot using Google Drive and Pinecone Vector DB

---

### 1. Workflow Overview

This workflow creates an AI-powered WhatsApp Bot that integrates Google Drive and a Pinecone Vector Database to offer intelligent, context-aware responses. It is designed to serve businesses or teams who want to automate WhatsApp conversations with an AI assistant that references custom documents stored in Google Drive.

Logical blocks of the workflow:

- **1.1 Knowledge Base Loading**  
  Periodically loads or updates a document from Google Drive, splits the content into manageable chunks, generates embeddings using OpenAI, and indexes them into a Pinecone vector database.

- **1.2 WhatsApp Webhook Listener & Message Filtering**  
  Listens for incoming WhatsApp messages via a webhook, checks if messages are from individual chats or groups (with group message filtering), and prepares the request for AI processing.

- **1.3 Vector Search and Context Preparation**  
  Uses the incoming message text as a query to retrieve the most relevant chunks from Pinecone, prepares and consolidates these chunks as context for the AI agent.

- **1.4 AI Agent Interaction and Response Generation**  
  Applies the OpenAI chat model with a custom system prompt to generate a brand-aligned, professional, and friendly response based on retrieved context and conversation memory.

- **1.5 Response Dispatch**  
  Sends the generated AI response back through WhatsApp via the webhook response node.

---

### 2. Block-by-Block Analysis

#### 2.1 Knowledge Base Loading

- **Overview:**  
  This block automatically fetches a Google Drive document every minute, processes its content into smaller chunks, embeds these chunks with OpenAI, and indexes them into a Pinecone vector store for fast retrieval.

- **Nodes Involved:**  
  - Every minute check if file is updated  
  - Download the file from Google Drive  
  - Recursive Character Text Splitter  
  - Default Data Loader  
  - Embeddings OpenAI  
  - Index Pinecone Vector Store  
  - Sticky Note (instructions and benefits)

- **Node Details:**

  - **Every minute check if file is updated**  
    - Type: Google Drive Trigger  
    - Watches a specific Google Drive file for updates every minute using service account authentication.  
    - Input: None (trigger node)  
    - Output: Provides the file metadata (including file ID) when updated.  
    - Edge cases: API quota limits, file permission errors.

  - **Download the file from Google Drive**  
    - Type: Google Drive node (download operation)  
    - Downloads the updated file binary content using the ID from the trigger node.  
    - Auth: Service account credentials configured.  
    - Input: File ID from trigger  
    - Output: Binary file data for further processing.  
    - Failure: Invalid file ID, permission denied, network issues.

  - **Recursive Character Text Splitter**  
    - Type: Langchain Text Splitter  
    - Splits large documents into chunks of 3000 characters with 200 character overlap to preserve context.  
    - Input: Binary document data  
    - Output: Array of text chunks  
    - Edge cases: Improper document format, chunk size too large/small.

  - **Default Data Loader**  
    - Type: Langchain Document Loader  
    - Processes chunks into a format suitable for embedding generation.  
    - Input: Text chunks from splitter  
    - Output: Document objects with page content.  

  - **Embeddings OpenAI**  
    - Type: Langchain OpenAI Embeddings node  
    - Uses OpenAI API to generate vector embeddings for document chunks.  
    - Credentials: OpenAI API key configured.  
    - Input: Document chunks  
    - Output: Embeddings for each chunk.  
    - Edge cases: API rate limits, invalid API key.

  - **Index Pinecone Vector Store**  
    - Type: Langchain Pinecone Vector Store node (insert mode)  
    - Inserts embeddings into Pinecone index "3vansales", clearing namespace before insert to avoid duplicates.  
    - Credentials: Pinecone API key configured.  
    - Input: Embeddings from OpenAI node  
    - Output: Confirmation of indexing  
    - Edge cases: Pinecone API errors, network timeouts.

  - **Sticky Notes**  
    - Provide step-by-step setup instructions and highlight key benefits such as no Meta account needed, group chat support, auto-syncing knowledge base, and use cases.

#### 2.2 WhatsApp Webhook Listener & Message Filtering

- **Overview:**  
  Listens to incoming WhatsApp messages via a webhook, filters messages by type (individual or group), and determines if the message should be processed or ignored.

- **Nodes Involved:**  
  - Listen to Whatsapp webhook  
  - Check for individual or group messages  
  - Set max chunks to send to model  
  - Sticky Note1 (overview of chat workflow)  
  - Sticky Note4 & Sticky Note5 (testing instructions)

- **Node Details:**

  - **Listen to Whatsapp webhook**  
    - Type: Webhook node (POST)  
    - Receives incoming WhatsApp messages with webhook ID configured.  
    - Input: Incoming HTTP POST from WhatsApp integration  
    - Output: JSON containing message details (fromName, fromNumber, textBody, threadType, etc.)  
    - Edge cases: Webhook misconfiguration, payload format changes.

  - **Check for individual or group messages**  
    - Type: If node  
    - Checks if incoming message's threadType equals "individual".  
    - Input: JSON from webhook node  
    - Output: Routes individual messages forward, others can be filtered or handled separately.  
    - Edge cases: Missing or unexpected threadType fields.

  - **Set max chunks to send to model**  
    - Type: Set node  
    - Sets a numerical variable "chunks" to 4, limiting how many chunks to fetch from Pinecone.  
    - Input: JSON from previous node  
    - Output: JSON with "chunks" field assigned.

  - **Sticky Notes**  
    - Explain the chat functionality with vector DB, message handling including group support, and testing procedures.

#### 2.3 Vector Search and Context Preparation

- **Overview:**  
  Uses the message text to query Pinecone for the most relevant chunks, then consolidates those chunks into a single context string for the AI agent.

- **Nodes Involved:**  
  - Get top chunks matching query  
  - Prepare chunks  
  - Embeddings OpenAI2  

- **Node Details:**

  - **Get top chunks matching query**  
    - Type: Langchain Pinecone Vector Store node (load mode)  
    - Queries Pinecone index "3vansales" for top K chunks matching the textBody of the incoming message.  
    - Uses variable "chunks" (max 4) to limit results.  
    - Credentials: Pinecone API key  
    - Input: Query text from webhook, chunk limit from Set node  
    - Output: List of matching document chunks with metadata.  
    - Edge cases: No matching chunks, API errors.

  - **Prepare chunks**  
    - Type: Code node (JavaScript)  
    - Concatenates retrieved chunk texts into a single "context" string with chunk separators.  
    - Also extracts original message text, sender number, and sender name from webhook JSON for further use.  
    - Input: Matching chunks from Pinecone query, original webhook data  
    - Output: JSON with keys: context, textBody, fromNumber, fromName.

  - **Embeddings OpenAI2**  
    - Type: Langchain OpenAI Embeddings node  
    - Generates embeddings for the user message textBody for potential additional usage (e.g., memory or further processing).  
    - Credentials: OpenAI API key  
    - Input: User message text  
    - Output: Embeddings vector.

#### 2.4 AI Agent Interaction and Response Generation

- **Overview:**  
  Uses a custom-configured AI agent with a detailed system prompt to generate personalized WhatsApp replies based on retrieved context, conversation history, and user input.

- **Nodes Involved:**  
  - Simple Memory  
  - OpenAI Chat Model  
  - Question & Answer  

- **Node Details:**

  - **Simple Memory**  
    - Type: Langchain Memory Buffer Window  
    - Stores conversation memory keyed by a custom session key (fromNumber) to maintain context across messages.  
    - Context window length: 20 messages.  
    - Input: Question & Answer node output for memory update  
    - Output: Maintains conversational state.  
    - Edge cases: Memory overflow, session key missing.

  - **OpenAI Chat Model**  
    - Type: Langchain OpenAI Chat Model  
    - Uses GPT-4o-mini model with custom system prompt defining the AI personality "Van" - a digital sales agent for GrayBox.  
    - Custom instructions specify tone, brand voice, behaviors, promotions, and fallback strategies.  
    - Credentials: OpenAI API key  
    - Input: Text prompt including user message and context  
    - Output: AI-generated response text.  
    - Edge cases: API rate limits, prompt formatting errors.

  - **Question & Answer**  
    - Type: Langchain Agent node  
    - Combines input text with system prompt, context from vector DB, and memory to produce an AI answer.  
    - Receives context from "Prepare chunks" node and memory from "Simple Memory".  
    - Output: Final AI response to be sent back.  
    - Edge cases: Context missing, incomplete prompts.

#### 2.5 Response Dispatch

- **Overview:**  
  Sends the AI-generated response back through the WhatsApp webhook to the user.

- **Nodes Involved:**  
  - Respond to Whatsapp Webhook  

- **Node Details:**

  - **Respond to Whatsapp Webhook**  
    - Type: Respond to Webhook node  
    - Sends the AI response message as the HTTP response to the WhatsApp webhook POST request, delivering the reply in real-time.  
    - Input: AI response from Question & Answer node  
    - Output: HTTP response to WhatsApp.  
    - Edge cases: Timeout, malformed response, webhook connection issues.

---

### 3. Summary Table

| Node Name                         | Node Type                                            | Functional Role                                | Input Node(s)                         | Output Node(s)                      | Sticky Note                                                                                           |
|----------------------------------|-----------------------------------------------------|-----------------------------------------------|-------------------------------------|-----------------------------------|-----------------------------------------------------------------------------------------------------|
| Every minute check if file is updated | Google Drive Trigger                                | Periodic trigger for Google Drive file update | None                                | Download the file from Google Drive |                                                                                                     |
| Download the file from Google Drive | Google Drive (download)                             | Retrieves updated Google Drive file content   | Every minute check if file is updated | Index Pinecone Vector Store        |                                                                                                     |
| Recursive Character Text Splitter  | Langchain Text Splitter                             | Splits document into chunks                    | Download the file from Google Drive | Default Data Loader                |                                                                                                     |
| Default Data Loader                | Langchain Document Loader                           | Prepares document chunks for embedding        | Recursive Character Text Splitter    | Embeddings OpenAI                 |                                                                                                     |
| Embeddings OpenAI                 | Langchain OpenAI Embeddings                         | Generates vector embeddings for chunks         | Default Data Loader                 | Index Pinecone Vector Store        |                                                                                                     |
| Index Pinecone Vector Store       | Langchain Pinecone Vector Store (insert mode)      | Inserts embeddings into Pinecone index         | Embeddings OpenAI, Download node   | None                             |                                                                                                     |
| Sticky Note                      | Sticky Note                                         | Overview of loading data to Pinecone            | None                               | None                             | ## Load Information from a Google Drive File into a Pinecone Vector Database ...                      |
| Sticky Note2                     | Sticky Note                                         | Google Drive credentials setup instructions    | None                               | None                             | ## Step 1: Connect Your Google Drive ...                                                             |
| Sticky Note3                     | Sticky Note                                         | Pinecone Vector Database setup instructions    | None                               | None                             | ## Step 2: Set Up Your Pinecone Vector Database ...                                                   |
| Sticky Note4                     | Sticky Note                                         | WhatsApp integration testing instructions      | None                               | None                             | ## Step 4: Test the WhatsApp Integration ...                                                         |
| Sticky Note5                     | Sticky Note                                         | WhatsApp integration testing instructions (cont)| None                             | None                             | ## Step 4: Test the WhatsApp Integration ...                                                         |
| Sticky Note6                     | Sticky Note                                         | Key benefits and use cases of the bot           | None                               | None                             | ## 🔑 Key Benefits ...                                                                                 |
| Listen to Whatsapp webhook        | Webhook (POST)                                      | Receives incoming WhatsApp messages             | None                               | Check for individual or group messages |                                                                                                     |
| Check for individual or group messages | If node                                           | Filters messages by chat type                    | Listen to Whatsapp webhook          | Set max chunks to send to model    |                                                                                                     |
| Set max chunks to send to model   | Set node                                            | Limits number of chunks retrieved from Pinecone| Check for individual or group messages | Get top chunks matching query      |                                                                                                     |
| Get top chunks matching query     | Langchain Pinecone Vector Store (load mode)         | Queries Pinecone for relevant chunks            | Set max chunks to send to model     | Prepare chunks                   |                                                                                                     |
| Prepare chunks                   | Code node (JavaScript)                              | Aggregates chunk texts and extracts metadata    | Get top chunks matching query       | Question & Answer                |                                                                                                     |
| Embeddings OpenAI2               | Langchain OpenAI Embeddings                         | Embeds user message text                         | Prepare chunks                    | Get top chunks matching query (embedding input) |                                                                                                     |
| Simple Memory                   | Langchain Memory Buffer Window                       | Maintains conversation memory                    | Question & Answer                  | Question & Answer (memory update) |                                                                                                     |
| OpenAI Chat Model                | Langchain OpenAI Chat Model                         | Generates AI responses with custom system prompt| Simple Memory                    | Question & Answer                |                                                                                                     |
| Question & Answer               | Langchain Agent                                     | Coordinates AI Q&A using context and memory     | Prepare chunks, Simple Memory, OpenAI Chat Model | Respond to Whatsapp Webhook        |                                                                                                     |
| Respond to Whatsapp Webhook      | Respond to Webhook node                             | Sends AI response back to WhatsApp               | Question & Answer                  | None                             |                                                                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Setup Google Drive Credentials**  
   - Create Google Drive OAuth2 credentials or use a Service Account.  
   - Follow official n8n docs: https://docs.n8n.io/integrations/builtin/credentials/google/

2. **Create Google Drive Trigger Node**  
   - Type: Google Drive Trigger  
   - Configure to trigger every minute on a specific file (set file ID).  
   - Authentication: Use your Google Drive credentials.

3. **Add Google Drive Download Node**  
   - Type: Google Drive (download)  
   - Configure operation "download" with dynamic file ID from trigger.  
   - Authentication: Same Google Drive credentials.

4. **Add Recursive Character Text Splitter Node**  
   - Type: Langchain Text Splitter Recursive Character Text Splitter  
   - Set chunkSize to 3000 and chunkOverlap to 200.

5. **Add Default Data Loader Node**  
   - Type: Langchain Document Default Data Loader  
   - Input dataType: binary (file content).

6. **Add Embeddings OpenAI Node**  
   - Type: Langchain OpenAI Embeddings  
   - Configure with your OpenAI API credentials.

7. **Add Pinecone Vector Store Node (Insert Mode)**  
   - Type: Langchain Pinecone Vector Store  
   - Mode: insert  
   - Option: clearNamespace = true  
   - Select your Pinecone index (e.g., "3vansales")  
   - Provide Pinecone API credentials.

8. **Connect the above nodes sequentially:**  
   Google Drive Trigger → Download node → Text Splitter → Data Loader → Embeddings → Pinecone Insert Node.

9. **Create Webhook Node for WhatsApp Messages**  
   - Type: Webhook (POST)  
   - Configure webhook path and method.  
   - This serves as the entry point for incoming WhatsApp messages.

10. **Add If Node to Filter Individual Messages**  
    - Type: If  
    - Condition: $json.body.threadType equals "individual".

11. **Add Set Node to Define Max Chunks**  
    - Type: Set  
    - Add numeric field "chunks" with value 4.

12. **Add Pinecone Vector Store Node (Load Mode)**  
    - Type: Langchain Pinecone Vector Store  
    - Mode: load  
    - Set topK to value of "chunks" from Set node.  
    - Query: Use incoming message textBody from webhook.  
    - Use Pinecone API credentials and same index.

13. **Add Code Node to Prepare Chunks**  
    - Concatenate retrieved chunk texts into a single "context" string with separators.  
    - Extract message textBody, fromNumber, and fromName from original webhook JSON.

14. **Add Embeddings OpenAI Node (for user message embedding)**  
    - To embed the incoming message textBody (optional for advanced memory or processing).

15. **Add Memory Buffer Node**  
    - Type: Langchain Memory Buffer Window  
    - Session key: Use fromNumber from webhook to maintain per-user memory.  
    - Context window length: 20 messages.

16. **Add OpenAI Chat Model Node**  
    - Model: "gpt-4o-mini"  
    - Configure custom system prompt defining AI "Van" personality and instructions.  
    - Use OpenAI credentials.

17. **Add Langchain Agent Node (Question & Answer)**  
    - Input text combines sender name and message.  
    - Include system message with agent instructions.  
    - Connect memory, chat model, and context (prepared chunks) as inputs.

18. **Add Respond to Webhook Node**  
    - Send the output text from Question & Answer agent as HTTP response to WhatsApp webhook.

19. **Connect Nodes in Message Handling Flow:**  
    Webhook → If (individual) → Set (chunks) → Pinecone Load → Prepare chunks → Question & Answer (with memory and chat model) → Respond to Webhook.

20. **Test the Workflow:**  
    - Upload a Google Drive document and ensure it is indexed.  
    - Send WhatsApp messages to the webhook URL and verify AI responses.  
    - Adjust system prompt and chunk size as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                                         | Context or Link                                                                                       |
|--------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| Google Drive Credentials Setup Guide                                                                                                | https://docs.n8n.io/integrations/builtin/credentials/google/                                        |
| Pinecone Index Setup Instructions                                                                                                  | Pinecone index dimension must be 1536; select index in both Pinecone nodes                          |
| WhatsApp Access Request Form                                                                                                        | https://docs.google.com/forms/d/e/1FAIpQLSd-bW5tSJu_rRvJ4NmFrxXSAwaNbO7MbGJtUIS-mBA23B7BWQ/viewform?usp=dialog |
| Brand Voice and AI Agent Personality Details                                                                                       | Custom prompt embedded in Question & Answer node defining "Van", the digital sales assistant        |
| WhatsApp Integration Testing Steps                                                                                                 | Instructions for one-on-one and group testing included in Sticky Notes                              |
| Key Benefits of the Bot                                                                                                            | No Meta account required, group chat support, auto-syncing knowledge base, use cases                |
| GrayBox Promotional Links for AI Agent to Share                                                                                   | - Discovery call: https://calendar.app.google/gJYWhw3LCJBPywGW6                                     |
|                                                                                                                                      | - Project portfolio: https://graybox.site/graybox-0-0                                               |
|                                                                                                                                      | - Meeting scheduling: https://calendar.app.google/rDzpGdMb3WmGsLFz6                                 |
|                                                                                                                                      | - Project submission form: https://forms.gle/5AddUQbqveFPsTDQ9                                     |
|                                                                                                                                      | - Office location: https://maps.app.goo.gl/nT7VDTtLQYSztN7s9                                       |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---