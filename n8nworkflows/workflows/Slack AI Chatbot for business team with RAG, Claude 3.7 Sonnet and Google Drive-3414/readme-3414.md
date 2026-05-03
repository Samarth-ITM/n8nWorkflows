Slack AI Chatbot for business team with RAG, Claude 3.7 Sonnet and Google Drive

https://n8nworkflows.xyz/workflows/slack-ai-chatbot-for-business-team-with-rag--claude-3-7-sonnet-and-google-drive-3414


# Slack AI Chatbot for business team with RAG, Claude 3.7 Sonnet and Google Drive

### 1. Workflow Overview

This workflow implements a **Slack AI Chatbot** designed for business teams to automate responses to internal queries about IT requests, company policies, vacation days, and other company knowledge. It leverages **Retrieval-Augmented Generation (RAG)** to provide accurate, context-aware answers by searching a company document repository stored in a Qdrant vector database. The chatbot uses Anthropic’s Claude 3.7 Sonnet model for natural language understanding and generation, enhanced by a simple memory buffer to maintain conversational context.

The workflow is logically divided into the following blocks:

- **1.1 Slack Input Reception**: Captures Slack messages where the bot is mentioned.
- **1.2 AI Agent Processing**: Processes the query using Anthropic Claude 3.7 Sonnet, integrating RAG and memory.
- **1.3 Knowledge Retrieval (RAG)**: Retrieves relevant document chunks from Qdrant using OpenAI embeddings.
- **1.4 Response Delivery**: Sends the AI-generated response back to Slack in a formatted message.
- **2.1 Qdrant Collection Management**: Creates and refreshes the Qdrant collection for document storage.
- **2.2 Document Ingestion and Vectorization**: Downloads company documents from Google Drive, splits them into chunks, generates embeddings, and inserts them into Qdrant.
- **Supporting Tools**: Includes embedding generation nodes, a calculator tool (available for AI use), and memory buffer management.

---

### 2. Block-by-Block Analysis

#### 1.1 Slack Input Reception

- **Overview:**  
  Listens for Slack messages where the AI bot is mentioned, capturing user queries to trigger the AI processing.

- **Nodes Involved:**  
  - Get message (Slack Trigger)

- **Node Details:**  
  - **Get message**  
    - Type: Slack Trigger  
    - Role: Listens for `app_mention` events in a specified Slack channel (channelId configured).  
    - Configuration: Trigger on `app_mention` in channel `C08L6SEPWMB`.  
    - Credentials: Slack API token with appropriate scopes.  
    - Inputs: Slack event data (message text, user ID, channel, timestamp).  
    - Outputs: JSON containing Slack message blocks and metadata.  
    - Edge cases: Slack API rate limits, missing permissions, or invalid channelId may cause failures.

#### 1.2 AI Agent Processing

- **Overview:**  
  Processes the Slack message text using Anthropic Claude 3.7 Sonnet, augmented by RAG retrieval and conversation memory for context continuity.

- **Nodes Involved:**  
  - AI Agent  
  - Simple Memory  
  - Anthropic Chat Model  
  - Calculator (available as a tool for AI)

- **Node Details:**  
  - **AI Agent**  
    - Type: LangChain Agent node  
    - Role: Central AI processing node that receives the user query, integrates RAG retrieval, memory, and language model to generate a response.  
    - Configuration:  
      - Text input extracted from Slack message JSON path `blocks[0].elements[0].elements[1].text`.  
      - System message defines AI assistant behavior, response formatting, interaction guidelines, and technical context.  
      - Uses RAG tool named "company_info" to retrieve relevant documents.  
      - Uses Simple Memory buffer for last 10 messages per user-channel session.  
    - Inputs: Query text, memory context, RAG retrieval results.  
    - Outputs: Generated response text.  
    - Edge cases: Expression evaluation errors if Slack message format changes, API errors from Anthropic or RAG, memory session key misconfiguration.  
  - **Simple Memory**  
    - Type: LangChain Memory Buffer Window  
    - Role: Maintains last 10 messages per user-channel session to provide conversational context.  
    - Configuration:  
      - Session key constructed as `channel_userId` from Slack message JSON.  
      - Context window length: 10 messages.  
    - Inputs: Incoming messages from Slack trigger.  
    - Outputs: Contextual memory for AI Agent.  
    - Edge cases: Session key mismatch, memory overflow, or data loss.  
  - **Anthropic Chat Model**  
    - Type: LangChain Language Model (Anthropic)  
    - Role: Provides natural language generation using Claude 3.7 Sonnet.  
    - Configuration: Model set to `claude-3-7-sonnet-20250219`.  
    - Inputs: Prompt from AI Agent.  
    - Outputs: Text completion.  
    - Edge cases: API rate limits, authentication errors, model unavailability.  
  - **Calculator**  
    - Type: LangChain Tool Calculator  
    - Role: Provides calculation capabilities to AI Agent if needed.  
    - Configuration: Default.  
    - Inputs/Outputs: Connected as AI tool to AI Agent.  
    - Edge cases: Calculation errors or unsupported operations.

#### 1.3 Knowledge Retrieval (RAG)

- **Overview:**  
  Retrieves relevant document chunks from the Qdrant vector database using OpenAI embeddings to support AI Agent responses.

- **Nodes Involved:**  
  - RAG (Qdrant Vector Store)  
  - Embeddings OpenAI1

- **Node Details:**  
  - **RAG**  
    - Type: LangChain Vector Store Qdrant (retrieve-as-tool mode)  
    - Role: Searches Qdrant collection for top 10 relevant document chunks based on query embeddings.  
    - Configuration:  
      - Collection name parameterized as `COLLECTION`.  
      - TopK set to 10.  
      - Tool name: "company_info".  
      - Tool description: "Get business documents".  
    - Credentials: Qdrant API credentials.  
    - Inputs: Query embeddings from Embeddings OpenAI1.  
    - Outputs: Retrieved document chunks for AI Agent.  
    - Edge cases: Network errors, authentication failures, empty or missing collection, malformed queries.  
  - **Embeddings OpenAI1**  
    - Type: LangChain Embeddings OpenAI  
    - Role: Generates embeddings for the user query to feed into RAG.  
    - Configuration: Default OpenAI embedding model.  
    - Credentials: OpenAI API key.  
    - Inputs: Text query from Slack message.  
    - Outputs: Vector embeddings.  
    - Edge cases: API rate limits, invalid API key, text input errors.

#### 1.4 Response Delivery

- **Overview:**  
  Sends the AI-generated, Slack-formatted response back to the originating Slack channel, optionally in a thread.

- **Nodes Involved:**  
  - Send message

- **Node Details:**  
  - **Send message**  
    - Type: Slack node (message sender)  
    - Role: Posts the AI response text to the Slack channel where the query originated.  
    - Configuration:  
      - Text set to AI Agent output JSON path `output`.  
      - Channel ID fixed to `C08L6SEPWMB`.  
      - Message options: Markdown enabled, reply in thread with broadcast, unfurl links enabled.  
    - Credentials: Slack API token.  
    - Inputs: AI Agent output text.  
    - Outputs: Slack message confirmation.  
    - Edge cases: Slack API errors, invalid channel, permission issues, message formatting errors.

#### 2.1 Qdrant Collection Management

- **Overview:**  
  Manages the Qdrant vector database collection lifecycle: creation and refreshing (clearing) before document ingestion.

- **Nodes Involved:**  
  - Create collection  
  - Refresh collection

- **Node Details:**  
  - **Create collection**  
    - Type: HTTP Request  
    - Role: Creates a new Qdrant collection via REST API.  
    - Configuration:  
      - POST to `https://QDRANTURL/collections/COLLECTION` with empty filter JSON body.  
      - Headers: Content-Type application/json.  
      - Authentication: HTTP header auth with Qdrant API key.  
    - Inputs: Manual trigger.  
    - Outputs: API response confirming collection creation.  
    - Edge cases: Network errors, authentication failure, collection already exists.  
  - **Refresh collection**  
    - Type: HTTP Request  
    - Role: Deletes all points in the Qdrant collection to clear data before re-ingestion.  
    - Configuration:  
      - POST to `https://QDRANTURL/collections/COLLECTION/points/delete` with empty filter JSON body.  
      - Headers and authentication same as Create collection.  
    - Inputs: Manual trigger.  
    - Outputs: API response confirming deletion.  
    - Edge cases: Network errors, authentication failure, collection not found.

#### 2.2 Document Ingestion and Vectorization

- **Overview:**  
  Downloads company documents from Google Drive, splits them into manageable chunks, generates embeddings, and inserts them into Qdrant for RAG.

- **Nodes Involved:**  
  - Get folder (Google Drive)  
  - Download Files (Google Drive)  
  - Default Data Loader  
  - Token Splitter  
  - Embeddings OpenAI2  
  - Qdrant Vector Store1

- **Node Details:**  
  - **Get folder**  
    - Type: Google Drive node  
    - Role: Lists files in a specified Google Drive folder.  
    - Configuration:  
      - Drive ID: "My Drive".  
      - Folder ID: `"=test-whatsapp"` (parameterized).  
    - Credentials: Google Drive OAuth2.  
    - Inputs: Trigger from Refresh collection node.  
    - Outputs: List of files metadata.  
    - Edge cases: Permission denied, folder not found, API limits.  
  - **Download Files**  
    - Type: Google Drive node  
    - Role: Downloads each file from the folder, converting Google Docs to plain text.  
    - Configuration:  
      - File ID from Get folder output.  
      - Conversion: Google Docs to `text/plain`.  
    - Credentials: Google Drive OAuth2.  
    - Inputs: File metadata from Get folder.  
    - Outputs: Binary file content.  
    - Edge cases: Download failures, unsupported file types, conversion errors.  
  - **Default Data Loader**  
    - Type: LangChain Document Default Data Loader  
    - Role: Loads downloaded binary files into LangChain document format for processing.  
    - Configuration: Data type set to binary.  
    - Inputs: Binary file from Download Files.  
    - Outputs: Document objects.  
    - Edge cases: File format errors, empty files.  
  - **Token Splitter**  
    - Type: LangChain Text Splitter (Token Splitter)  
    - Role: Splits documents into chunks of 300 tokens with 30 token overlap for embedding.  
    - Configuration: chunkSize=300, chunkOverlap=30.  
    - Inputs: Documents from Default Data Loader.  
    - Outputs: Text chunks.  
    - Edge cases: Very short documents, tokenization errors.  
  - **Embeddings OpenAI2**  
    - Type: LangChain Embeddings OpenAI  
    - Role: Generates vector embeddings for document chunks.  
    - Configuration: Default OpenAI embedding model.  
    - Credentials: OpenAI API key.  
    - Inputs: Text chunks from Token Splitter.  
    - Outputs: Vector embeddings.  
    - Edge cases: API limits, invalid input.  
  - **Qdrant Vector Store1**  
    - Type: LangChain Vector Store Qdrant (insert mode)  
    - Role: Inserts document embeddings into the Qdrant collection.  
    - Configuration: Collection name parameterized as `COLLECTION`.  
    - Credentials: Qdrant API key.  
    - Inputs: Embeddings from Embeddings OpenAI2 and document metadata.  
    - Outputs: Confirmation of insertion.  
    - Edge cases: Network errors, authentication failure, collection not found.

---

### 3. Summary Table

| Node Name               | Node Type                                | Functional Role                          | Input Node(s)               | Output Node(s)           | Sticky Note                                                                                          |
|-------------------------|-----------------------------------------|----------------------------------------|-----------------------------|--------------------------|----------------------------------------------------------------------------------------------------|
| AI Agent                | LangChain Agent                         | Processes Slack query with RAG & memory| Get message, RAG, Simple Memory, Anthropic Chat Model, Calculator | Send message             |                                                                                                    |
| Simple Memory           | LangChain Memory Buffer Window          | Maintains conversation context         | Get message                  | AI Agent                 |                                                                                                    |
| Embeddings OpenAI1      | LangChain Embeddings OpenAI             | Generates embeddings for query          | Get message                  | RAG                      |                                                                                                    |
| RAG                     | LangChain Vector Store Qdrant           | Retrieves relevant documents            | Embeddings OpenAI1           | AI Agent                 |                                                                                                    |
| Calculator              | LangChain Tool Calculator                | Provides calculation tool for AI Agent |                             | AI Agent                 |                                                                                                    |
| Get message             | Slack Trigger                           | Captures Slack app_mention events       |                             | AI Agent                 |                                                                                                    |
| Send message            | Slack node                             | Sends AI response to Slack channel      | AI Agent                    |                          |                                                                                                    |
| When clicking ‘Test workflow’ | Manual Trigger                      | Starts collection creation and refresh  |                             | Create collection, Refresh collection |                                                                                                    |
| Create collection       | HTTP Request                           | Creates Qdrant collection                | When clicking ‘Test workflow’| Refresh collection, Get folder | # STEP 1 Create Qdrant Collection. Change QDRANTURL and COLLECTION                                  |
| Refresh collection      | HTTP Request                           | Clears Qdrant collection points          | Create collection            | Get folder                | # STEP 1 Create Qdrant Collection. Change QDRANTURL and COLLECTION                                  |
| Get folder              | Google Drive                           | Lists files in Google Drive folder       | Refresh collection           | Download Files            | # STEP 2 Documents vectorization with Qdrant and Google Drive. Change QDRANTURL and COLLECTION     |
| Download Files          | Google Drive                           | Downloads files from Google Drive        | Get folder                  | Default Data Loader       | # STEP 2 Documents vectorization with Qdrant and Google Drive. Change QDRANTURL and COLLECTION     |
| Default Data Loader     | LangChain Document Loader               | Loads downloaded files as documents      | Download Files              | Token Splitter            | # STEP 2 Documents vectorization with Qdrant and Google Drive. Change QDRANTURL and COLLECTION     |
| Token Splitter          | LangChain Text Splitter (Token Splitter)| Splits documents into chunks             | Default Data Loader         | Embeddings OpenAI2        | # STEP 2 Documents vectorization with Qdrant and Google Drive. Change QDRANTURL and COLLECTION     |
| Embeddings OpenAI2      | LangChain Embeddings OpenAI             | Generates embeddings for document chunks| Token Splitter              | Qdrant Vector Store1      | # STEP 2 Documents vectorization with Qdrant and Google Drive. Change QDRANTURL and COLLECTION     |
| Qdrant Vector Store1    | LangChain Vector Store Qdrant (insert) | Inserts embeddings into Qdrant collection| Embeddings OpenAI2          |                          | # STEP 2 Documents vectorization with Qdrant and Google Drive. Change QDRANTURL and COLLECTION     |
| Anthropic Chat Model    | LangChain Language Model (Anthropic)   | Provides natural language generation     | AI Agent                    | AI Agent                 |                                                                                                    |
| Sticky Note3            | Sticky Note                            | Instruction: Step 1 - Create Qdrant collection |                             |                          | # STEP 1 Create Qdrant Collection. Change QDRANTURL and COLLECTION                                  |
| Sticky Note5            | Sticky Note                            | Instruction: Step 2 - Document vectorization |                             |                          | # STEP 2 Documents vectorization with Qdrant and Google Drive. Change QDRANTURL and COLLECTION     |
| Sticky Note4            | Sticky Note                            | Instruction: Step 3 - Slack Bot setup    |                             |                          | # STEP 3 Create a Slack Bot with required scopes and add to channel. Change COLLECTION in RAG node |
| Sticky Note2            | Sticky Note                            | Workflow overview and purpose description|                             |                          | # Slack AI Chatbot Workflow with RAG - overview and benefits                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Slack Trigger Node ("Get message")**  
   - Type: Slack Trigger  
   - Configure to trigger on `app_mention` events.  
   - Set channel ID to your Slack channel (e.g., `C08L6SEPWMB`).  
   - Authenticate with Slack OAuth token having scopes: `app_mentions:read`, `chat:write`, `channels:read`, etc.

2. **Add Simple Memory Node ("Simple Memory")**  
   - Type: LangChain Memory Buffer Window  
   - Set `sessionKey` to combine channel and user ID from Slack message JSON:  
     `={{ $('Get message').item.json.channel }}_{{ $('Get message').item.json.blocks[0].elements[0].elements[0].user_id }}`  
   - Set context window length to 10 messages.

3. **Add Embeddings Node for Query ("Embeddings OpenAI1")**  
   - Type: LangChain Embeddings OpenAI  
   - Use OpenAI API credentials.  
   - Default embedding model.

4. **Add RAG Node ("RAG")**  
   - Type: LangChain Vector Store Qdrant (retrieve-as-tool mode)  
   - Set collection name parameter (e.g., `COLLECTION`).  
   - Set `topK` to 10.  
   - Authenticate with Qdrant API credentials.

5. **Add Anthropic Chat Model Node ("Anthropic Chat Model")**  
   - Type: LangChain Language Model (Anthropic)  
   - Select model `claude-3-7-sonnet-20250219`.  
   - Authenticate with Anthropic API credentials.

6. **Add Calculator Tool Node ("Calculator")**  
   - Type: LangChain Tool Calculator  
   - Default configuration.

7. **Add AI Agent Node ("AI Agent")**  
   - Type: LangChain Agent  
   - Set input text expression to extract Slack message text:  
     `={{ $json.blocks[0].elements[0].elements[1].text }}`  
   - Paste the provided system message defining AI assistant behavior and response format.  
   - Connect AI Agent inputs:  
     - `ai_languageModel` → Anthropic Chat Model  
     - `ai_embedding` → RAG (for retrieval)  
     - `ai_memory` → Simple Memory  
     - `ai_tool` → Calculator and RAG  
   - Connect AI Agent output to Send message node.

8. **Add Slack Send Message Node ("Send message")**  
   - Type: Slack node  
   - Set text to AI Agent output: `={{ $json.output }}`  
   - Set channel ID to Slack channel (same as trigger).  
   - Enable markdown formatting, reply in thread with broadcast, unfurl links.

9. **Connect Nodes for Slack Interaction:**  
   - Get message → AI Agent  
   - Simple Memory → AI Agent (memory input)  
   - Embeddings OpenAI1 → RAG → AI Agent (embedding and retrieval)  
   - Anthropic Chat Model → AI Agent (language model)  
   - Calculator → AI Agent (tool)  
   - AI Agent → Send message

10. **Set Up Qdrant Collection Management:**  
    - Create HTTP Request node ("Create collection")  
      - POST to `https://QDRANTURL/collections/COLLECTION`  
      - Body: `{ "filter": {} }`  
      - Headers: Content-Type: application/json  
      - Auth: HTTP header with Qdrant API key  
    - Create HTTP Request node ("Refresh collection")  
      - POST to `https://QDRANTURL/collections/COLLECTION/points/delete`  
      - Body: `{ "filter": {} }`  
      - Same headers and auth as above  
    - Connect manual trigger node ("When clicking ‘Test workflow’") to both Create collection and Refresh collection nodes.

11. **Set Up Document Ingestion Pipeline:**  
    - Google Drive node ("Get folder")  
      - Configure Drive ID (e.g., "My Drive") and Folder ID (e.g., `"=test-whatsapp"`).  
      - Authenticate with Google Drive OAuth2.  
      - Connect output to Download Files node.  
    - Google Drive node ("Download Files")  
      - Download each file by ID, convert Google Docs to plain text.  
      - Authenticate with Google Drive OAuth2.  
      - Connect output to Default Data Loader.  
    - LangChain Document Loader ("Default Data Loader")  
      - Set data type to binary.  
      - Connect output to Token Splitter.  
    - LangChain Text Splitter ("Token Splitter")  
      - Set chunk size to 300 tokens, overlap 30 tokens.  
      - Connect output to Embeddings OpenAI2.  
    - LangChain Embeddings OpenAI ("Embeddings OpenAI2")  
      - Use OpenAI API credentials.  
      - Connect output to Qdrant Vector Store1.  
    - LangChain Vector Store Qdrant ("Qdrant Vector Store1")  
      - Set collection name to `COLLECTION`.  
      - Authenticate with Qdrant API credentials.

12. **Connect Document Ingestion Flow:**  
    - Refresh collection → Get folder → Download Files → Default Data Loader → Token Splitter → Embeddings OpenAI2 → Qdrant Vector Store1

13. **Final Checks:**  
    - Verify all credentials are configured and valid (Slack, OpenAI, Anthropic, Qdrant, Google Drive).  
    - Replace placeholders `QDRANTURL` and `COLLECTION` with actual values.  
    - Test document ingestion by triggering manual node.  
    - Test Slack interaction by mentioning bot in configured channel.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                  | Context or Link                                                                                  |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Slack Bot Setup: Create a Slack bot at https://api.slack.com, add it to your Slack channel, and configure scopes including `app_mentions:read`, `chat:write`, `channels:read`, and others as listed in Sticky Note4.           | Slack API documentation: https://api.slack.com                                                  |
| The AI Agent uses Anthropic Claude 3.7 Sonnet model for natural language understanding and generation, integrated with RAG and memory for context-aware responses.                                                             | Anthropic API: https://www.anthropic.com                                                       |
| Qdrant vector database is used for storing and retrieving document embeddings; ensure the Qdrant URL and collection name are correctly set in HTTP Request and LangChain nodes.                                                 | Qdrant docs: https://qdrant.tech                                                               |
| Google Drive OAuth2 credentials must have access to the folder containing company documents; files are converted to plain text for embedding.                                                                                 | Google Drive API: https://developers.google.com/drive/api                                      |
| The system message in AI Agent defines strict formatting rules for Slack responses, including markdown usage, concise answers, source citation, and fallback instructions when no information is found.                         |                                                                                               |
| For customization or consulting support, contact info@n3w.it or connect on LinkedIn: https://www.linkedin.com/in/davideboizza/                                                                                               |                                                                                               |
| The workflow supports 24/7 availability and automates repetitive internal queries, improving team productivity and collaboration by centralizing company knowledge access via Slack.                                           |                                                                                               |

---

This documentation provides a complete, structured reference for understanding, reproducing, and maintaining the Slack AI Chatbot workflow with RAG, Anthropic Claude 3.7 Sonnet, and Google Drive integration.