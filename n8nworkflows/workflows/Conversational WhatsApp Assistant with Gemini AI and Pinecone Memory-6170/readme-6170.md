Conversational WhatsApp Assistant with Gemini AI and Pinecone Memory

https://n8nworkflows.xyz/workflows/conversational-whatsapp-assistant-with-gemini-ai-and-pinecone-memory-6170


# Conversational WhatsApp Assistant with Gemini AI and Pinecone Memory

---

### 1. Workflow Overview

This workflow, named **"Conversational WhatsApp Assistant with Gemini AI and Pinecone Memory"**, is designed to serve as a smart AI assistant for WhatsApp messaging. It leverages Google Gemini AI for natural conversational responses, Pinecone vector databases for persistent memory and knowledge base retrieval, and WAMM.pro for WhatsApp API integration. The assistant can remember past conversations per user and access a general knowledge base to provide contextually relevant and human-like replies.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receiving WhatsApp messages via a webhook from WAMM.pro.
- **1.2 AI Processing:** Generating natural language responses using Google Gemini AI and a LangChain AI Agent configured with conversational context and tools.
- **1.3 Memory and Knowledge Retrieval:** Querying Pinecone vector stores to fetch relevant past conversation snippets and general knowledge for context-aware replies.
- **1.4 Message Sending:** Sending the AI-generated response back to the user via WAMM.pro.
- **1.5 Memory Update:** Processing and storing the new conversation data as vector embeddings in Pinecone for future retrieval.
- **1.6 Auxiliary Text Processing:** Preparing documents and splitting text for indexing in Pinecone.
- **1.7 Configuration and Documentation:** Sticky notes providing setup instructions, usage context, and best practices.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  Captures incoming WhatsApp messages forwarded from WAMM.pro using a dedicated webhook HTTP POST endpoint.

- **Nodes Involved:**  
  - Webhook

- **Node Details:**

  - **Webhook**  
    - *Type & Role:* HTTP POST webhook node to receive incoming WhatsApp messages from WAMM.pro integration.  
    - *Configuration:*  
      - HTTP Method: POST  
      - Path: Unique webhook path (`1c9432a6-f982-4102-ad0e-39ec15876b0a`) mapped to WAMM.pro webhook setup.  
    - *Expressions:* None directly; raw incoming JSON payload accessible for downstream nodes as `$json.body.data`.  
    - *Input/Output:* No input; outputs received request data.  
    - *Edge Cases/Potential Failures:*  
      - Malformed requests or unexpected payloads.  
      - Missing mandatory fields (e.g., `message_text`, `source_number`).  
      - Network or permission errors blocking webhook calls.  

---

#### 1.2 AI Processing

- **Overview:**  
  Processes the incoming message text to generate a natural, conversational reply using a LangChain AI Agent powered by Google Gemini. The AI Agent is configured to maintain a friendly tone, speak in the user’s language, and never mention lookup or search actions explicitly.

- **Nodes Involved:**  
  - AI Agent  
  - Google Gemini Chat Model

- **Node Details:**

  - **Google Gemini Chat Model**  
    - *Type & Role:* Language model node interfacing with Google Gemini AI (PaLM) for chat completions.  
    - *Configuration:*  
      - Model Name: `models/gemini-2.5-flash` (a conversational Gemini model).  
      - No special options configured.  
    - *Credentials:* Google Palm API key required.  
    - *Input/Output:* Inputs prompt text and system instructions; outputs AI-generated text completions.  
    - *Edge Cases:*  
      - API key invalid or rate-limited.  
      - Model service downtime or latency.  

  - **AI Agent**  
    - *Type & Role:* LangChain agent node that orchestrates AI interactions, embedding system instructions and tools for response generation.  
    - *Configuration:*  
      - Input text expression: `={{ $json.body.data.message_text }}` (incoming user message).  
      - System message instructs the AI to be warm, friendly, conversational, respond in user language, and avoid mentioning lookup/search. Includes dynamic placeholders for user phone number and current time.  
      - Prompt type: "define" (custom system prompt).  
    - *Input:* From Webhook node.  
    - *Output:* AI-generated response text, passed downstream for sending and memory update.  
    - *Edge Cases:*  
      - Expression failures if input path is wrong.  
      - Timeout or API errors from Gemini Chat Model.  
      - Unexpected user languages or characters.  

---

#### 1.3 Memory and Knowledge Retrieval

- **Overview:**  
  Retrieves relevant context from two Pinecone vector indexes: a global knowledge base and user-specific conversation history. These results are provided as AI tools for the LangChain AI Agent to incorporate into replies.

- **Nodes Involved:**  
  - Embeddings OpenAI  
  - Reading from Pinecone (from knowledge)  
  - Reading from Pinecone (at phone number)

- **Node Details:**

  - **Embeddings OpenAI**  
    - *Type & Role:* Generates vector embeddings for text using OpenAI’s embedding model to enable semantic similarity searches.  
    - *Configuration:*  
      - Model: `text-embedding-3-large` with 3072 dimensions.  
    - *Credentials:* OpenAI API key required.  
    - *Input/Output:* Accepts text data to embed; outputs embeddings to Pinecone retrieval nodes.  
    - *Edge Cases:*  
      - API key invalid or rate limits.  
      - Embedding failures on unexpected input.  

  - **Reading from Pinecone (from knowledge)**  
    - *Type & Role:* Vector store retrieval node querying the `knowledge` Pinecone index for general FAQs and knowledge base information.  
    - *Configuration:*  
      - Mode: `retrieve-as-tool` allowing the AI Agent to query this as an external tool.  
      - Tool Name: "reading_knowledge".  
      - Tool Description: Used for general knowledge base searches.  
    - *Credentials:* Pinecone API key required.  
    - *Input/Output:* Receives embeddings from OpenAI node; outputs matched documents for AI Agent tool context.  
    - *Edge Cases:*  
      - Pinecone API errors or indexing delays.  
      - No relevant matches found.  

  - **Reading from Pinecone (at phone number)**  
    - *Type & Role:* Vector store retrieval node querying the `historywa` Pinecone index for user-specific past conversation context.  
    - *Configuration:*  
      - Mode: `retrieve-as-tool`.  
      - Tool Name: "reading_historywa".  
      - Tool Description includes dynamic user phone number to limit search scope to that user.  
    - *Credentials:* Pinecone API key required.  
    - *Input/Output:* Same as above but scoped per user.  
    - *Edge Cases:*  
      - Failures in dynamic expression evaluation for phone number.  
      - Empty user history causing limited context.  

---

#### 1.4 Message Sending

- **Overview:**  
  Sends the AI-generated reply back to the user’s WhatsApp number via WAMM.pro API.

- **Nodes Involved:**  
  - WAMM: Send Message

- **Node Details:**

  - **WAMM: Send Message**  
    - *Type & Role:* WAMM.pro integration node for sending WhatsApp messages.  
    - *Configuration:*  
      - Phone Number: Extracted dynamically from webhook data (`={{ $('Webhook').item.json.body.data.source_number }}`).  
      - Message: AI Agent’s output text (`={{ $json.output }}`).  
      - Instance ID: WAMM instance identifier (masked in JSON).  
    - *Credentials:* WAMM API credentials with instance ID and access token.  
    - *Input:* From AI Agent node.  
    - *Output:* Confirmation or status of message sending.  
    - *Edge Cases:*  
      - Invalid phone number format or unregistered WhatsApp user.  
      - API rate limiting or authentication errors.  
      - Message length or content restrictions.  

---

#### 1.5 Memory Update

- **Overview:**  
  Processes and formats the conversation (incoming user message + AI response) into a structured document for insertion into the Pinecone `historywa` index to maintain persistent conversational memory.

- **Nodes Involved:**  
  - Processing data for Pinecone  
  - Default Data Loader1  
  - Recursive Character Text Splitter  
  - Pinecone Vector Store

- **Node Details:**

  - **Processing data for Pinecone**  
    - *Type & Role:* JavaScript code node preparing a combined conversation text and metadata for Pinecone indexing.  
    - *Configuration:*  
      - Extracts AI Agent output and webhook message text.  
      - Builds a conversation string including user number, user message, assistant response, and current timestamp.  
      - Returns JSON with `pageContent` and metadata (`user_number`, `timestamp`).  
    - *Input:* From WAMM: Send Message node.  
    - *Output:* Formatted JSON document.  
    - *Edge Cases:*  
      - Missing or undefined AI output or webhook data.  
      - Timezone or timestamp formatting errors.  

  - **Default Data Loader1**  
    - *Type & Role:* LangChain document loader that formats JSON text and metadata into a document object suitable for Pinecone.  
    - *Configuration:*  
      - Injects metadata fields for user number and timestamp.  
      - Input JSON text from previous node’s `pageContent`.  
    - *Input:* From Recursive Character Text Splitter.  
    - *Output:* Document ready for vector insertion.  
    - *Edge Cases:*  
      - Inconsistent metadata or empty content.  

  - **Recursive Character Text Splitter**  
    - *Type & Role:* Text splitter node to recursively split long conversation text into smaller chunks for indexing efficiency.  
    - *Configuration:* Default options, no custom parameters.  
    - *Input:* From Processing data for Pinecone.  
    - *Output:* Split text chunks for the Data Loader.  
    - *Edge Cases:*  
      - Very short or empty texts might bypass splitting.  

  - **Pinecone Vector Store**  
    - *Type & Role:* Vector store node inserting processed documents into the `historywa` Pinecone index.  
    - *Configuration:*  
      - Mode: `insert` for adding new vectors.  
      - Target index: `historywa` (user conversation history).  
    - *Credentials:* Pinecone API key required.  
    - *Input:* From Default Data Loader1 and Processing data for Pinecone.  
    - *Output:* Confirmation of vector insertion.  
    - *Edge Cases:*  
      - API limit exceeded or connectivity issues.  
      - Duplicate data insertion.  

---

#### 1.6 Auxiliary Text Processing

- **Overview:**  
  Supports the memory update block by preparing and splitting conversation text for efficient vector storage.

- **Nodes Involved:**  
  - Recursive Character Text Splitter  
  - Default Data Loader1

- **Node Details:**  
  These nodes have been described above under Memory Update and work in tandem to process conversation text before insertion into Pinecone.

---

#### 1.7 Configuration and Documentation

- **Overview:**  
  Provides extensive in-workflow documentation and setup instructions through sticky notes for users configuring and maintaining the workflow.

- **Nodes Involved:**  
  - Sticky Note  
  - Sticky Note1  
  - Sticky Note2  
  - Sticky Note3

- **Node Details:**

  - **Sticky Note**  
    - *Content:* Overview of the workflow, description, requirements (WAMM.pro, Pinecone, Google AI, OpenAI), setup steps, AI tools description, and benefits.  
    - *Position:* Top-left, serves as main documentation.  

  - **Sticky Note1**  
    - *Content:* Detailed WAMM.pro configuration instructions including account setup, webhook integration, and message filtering options.  
    - *Position:* Above WAMM Send Message node.  

  - **Sticky Note2**  
    - *Content:* Pinecone configuration guide for creating the two indexes (`historywa` and `knowledge`) with recommended parameters.  
    - *Position:* Near Pinecone retrieval nodes.  

  - **Sticky Note3**  
    - *Content:* Use cases, security considerations, and possible extensions like CRM integration, scheduling, and analytics.  
    - *Position:* Near vector store insertion nodes.  

- **Edge Cases:** None (documentation nodes).

---

### 3. Summary Table

| Node Name                          | Node Type                                     | Functional Role                               | Input Node(s)                  | Output Node(s)                             | Sticky Note                                                                                     |
|-----------------------------------|-----------------------------------------------|-----------------------------------------------|-------------------------------|--------------------------------------------|------------------------------------------------------------------------------------------------|
| Webhook                           | n8n-nodes-base.webhook                        | Receive WhatsApp messages from WAMM.pro       | -                             | AI Agent                                   | See Sticky Note1 for WAMM.pro webhook setup instructions                                       |
| AI Agent                         | @n8n/n8n-nodes-langchain.agent                | Generate conversational AI response            | Webhook, Google Gemini Chat Model, Pinecone tools | WAMM: Send Message                         | Configured to respond naturally, in user’s language, no mention of search                      |
| Google Gemini Chat Model          | @n8n/n8n-nodes-langchain.lmChatGoogleGemini  | Google Gemini AI language model                 | AI Agent                      | AI Agent                                   | Requires Google Palm API credentials                                                           |
| WAMM: Send Message                | n8n-nodes-wamm.wammpro                        | Send AI response to WhatsApp via WAMM.pro      | AI Agent                      | Processing data for Pinecone                | Requires WAMM API credentials; see Sticky Note1                                                |
| Processing data for Pinecone      | n8n-nodes-base.code                           | Format conversation text and metadata for Pinecone | WAMM: Send Message            | Pinecone Vector Store, Recursive Character Text Splitter |                                                                                               |
| Recursive Character Text Splitter | @n8n/n8n-nodes-langchain.textSplitterRecursiveCharacterTextSplitter | Split conversation text into chunks            | Processing data for Pinecone  | Default Data Loader1                        |                                                                                               |
| Default Data Loader1              | @n8n/n8n-nodes-langchain.documentDefaultDataLoader | Prepare documents with metadata for indexing   | Recursive Character Text Splitter | Pinecone Vector Store                      |                                                                                               |
| Pinecone Vector Store             | @n8n/n8n-nodes-langchain.vectorStorePinecone | Insert conversation vectors into Pinecone      | Processing data for Pinecone, Default Data Loader1 | -                                          | Requires Pinecone API credentials; see Sticky Note2 and Sticky Note3                           |
| Embeddings OpenAI                 | @n8n/n8n-nodes-langchain.embeddingsOpenAi    | Generate embeddings for text for vector search | -                             | Reading from Pinecone (knowledge), Reading from Pinecone (historywa), Pinecone Vector Store | Requires OpenAI API key                                                                        |
| Reading from Pinecone (from knowledge) | @n8n/n8n-nodes-langchain.vectorStorePinecone | Retrieve from knowledge base Pinecone index    | Embeddings OpenAI             | AI Agent                                   | Queries `knowledge` index for FAQs and general knowledge; see Sticky Note2                     |
| Reading from Pinecone (at phone number) | @n8n/n8n-nodes-langchain.vectorStorePinecone | Retrieve user-specific past conversation memory | Embeddings OpenAI             | AI Agent                                   | Queries `historywa` index scoped by user’s phone number; see Sticky Note2                      |
| Sticky Note                      | n8n-nodes-base.stickyNote                     | Documentation and instructions                  | -                             | -                                          | Overview, description, requirements, setup instructions                                       |
| Sticky Note1                     | n8n-nodes-base.stickyNote                     | WAMM.pro setup instructions                      | -                             | -                                          | WAMM.pro account, webhook config, message filtering                                           |
| Sticky Note2                     | n8n-nodes-base.stickyNote                     | Pinecone index configuration instructions       | -                             | -                                          | Index creation, dimensions, metric, embedding model                                           |
| Sticky Note3                     | n8n-nodes-base.stickyNote                     | Use cases, security, extensions                  | -                             | -                                          | Use cases, data security, possible workflow extensions                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Webhook Node**  
   - Type: Webhook (HTTP POST)  
   - Path: Unique identifier (e.g., `1c9432a6-f982-4102-ad0e-39ec15876b0a`)  
   - HTTP Method: POST  
   - Purpose: Receive WhatsApp messages from WAMM.pro.  

2. **Set Up Google Gemini Chat Model Node**  
   - Type: LangChain Google Gemini Chat Model  
   - Model Name: `models/gemini-2.5-flash`  
   - Credentials: Configure Google Palm API key.  

3. **Create AI Agent Node**  
   - Type: LangChain Agent  
   - Input Text: Expression referencing webhook message text: `={{ $json.body.data.message_text }}`  
   - System Message:  
     ```
     You are Alex, a warm and friendly assistant. Talk naturally like a good friend would - never mention searching or looking up information. Just remember and respond naturally.

     Always respond in the user's language (note: “salut” is in Romanian.).

     Talking to: {{ $json.body.data.source_number }}
     Time: {{ $now.toISO() }}
     ```  
   - Prompt Type: Define  
   - Connect input from Webhook and AI Language Model (Google Gemini Chat Model).  

4. **Create Embeddings OpenAI Node**  
   - Type: LangChain OpenAI Embeddings  
   - Model: `text-embedding-3-large`  
   - Options: Dimensions 3072  
   - Credentials: OpenAI API key.  

5. **Create Reading from Pinecone Nodes**  
   - Two nodes of type LangChain Pinecone Vector Store Retrieval:  
     - **From knowledge:**  
       - Mode: Retrieve-as-tool  
       - Index: `knowledge`  
       - Tool Name: `reading_knowledge`  
       - Description: For general FAQs and knowledge base.  
     - **At phone number:**  
       - Mode: Retrieve-as-tool  
       - Index: `historywa`  
       - Tool Name: `reading_historywa`  
       - Description: For user-specific conversation history, dynamically filtered by phone number (expression referencing webhook data).  
   - Credentials: Pinecone API key.  
   - Connect input from Embeddings OpenAI output.  
   - Connect outputs as tools for AI Agent input.  

6. **Create WAMM: Send Message Node**  
   - Type: WAMM.pro integration node  
   - Phone Number: Expression `={{ $('Webhook').item.json.body.data.source_number }}`  
   - Message: Expression `={{ $json.output }}` (AI Agent’s generated text)  
   - Instance ID: Your WAMM.pro instance ID  
   - Credentials: WAMM API credentials (Instance ID + Access Token).  
   - Connect input from AI Agent output.  

7. **Create Processing Data for Pinecone Node**  
   - Type: Code (JavaScript)  
   - Code:  
     ```javascript
     const aiAgentOutput = $('AI Agent').first().json.output;
     const webhookData = $('Webhook').first().json.body.data;

     const conversationText = `Utilizator (${webhookData.source_number}): ${webhookData.message_text}\nAsistent: ${aiAgentOutput}\nTimestamp: ${new Date().toISOString()}`;

     return {
       json: {
         pageContent: conversationText,
         metadata: {
           user_number: webhookData.source_number,
           timestamp: new Date().toISOString()
         }
       }
     };
     ```  
   - Connect input from WAMM: Send Message output.  

8. **Create Recursive Character Text Splitter Node**  
   - Type: LangChain Recursive Character Text Splitter  
   - Default options (no change).  
   - Connect input from Processing data for Pinecone node.  

9. **Create Default Data Loader Node**  
   - Type: LangChain Document Default Data Loader  
   - Options: Pass metadata fields `user_number` and `timestamp` from input JSON’s metadata.  
   - JSON Data: Expression `={{ $json.pageContent }}`  
   - JSON Mode: Expression Data  
   - Connect input from Recursive Character Text Splitter output.  

10. **Create Pinecone Vector Store Node**  
    - Type: LangChain Pinecone Vector Store  
    - Mode: Insert  
    - Index: `historywa`  
    - Credentials: Pinecone API key.  
    - Connect inputs from Processing data for Pinecone node (for main) and Default Data Loader node (for AI document input).  

11. **Configure Connections:**  
    - Webhook → AI Agent  
    - Google Gemini Chat Model → AI Agent (as AI Language Model)  
    - Embeddings OpenAI → Reading from Pinecone (knowledge), Reading from Pinecone (historywa), Pinecone Vector Store  
    - Reading from Pinecone nodes → AI Agent (as tools)  
    - AI Agent → WAMM: Send Message  
    - WAMM: Send Message → Processing data for Pinecone  
    - Processing data for Pinecone → Recursive Character Text Splitter  
    - Recursive Character Text Splitter → Default Data Loader1  
    - Default Data Loader1 → Pinecone Vector Store  
    - Processing data for Pinecone → Pinecone Vector Store (main input)  

12. **Add Sticky Notes (Optional but recommended):**  
    - Add detailed documentation notes based on Sticky Note contents for setup instructions, configuration tips, and use cases.

---

### 5. General Notes & Resources

| Note Content                                                                                                                         | Context or Link                                                                                   |
|--------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| WAMM.pro platform enables WhatsApp automation with free and PRO tiers: [https://wamm.pro](https://wamm.pro)                         | WAMM.pro account and webhook setup instructions                                                 |
| Pinecone requires two indexes: `historywa` for user conversation memory and `knowledge` for FAQ/general knowledge base              | Pinecone index creation and configuration                                                      |
| Google Gemini AI (PaLM) provides conversational AI model used via LangChain's Gemini node                                            | Google AI account and API key required                                                         |
| OpenAI embeddings generate 3072-dimensional vectors used for semantic similarity search in Pinecone                                 | OpenAI API key required                                                                         |
| AI Agent is configured to never mention searching or looking up information to keep conversations natural and friendly              | System prompt configuration in AI Agent                                                        |
| Security note: Data stored as vector embeddings in Pinecone, no plain text storage; separate memory space per user                   | Security best practices and data privacy                                                       |
| Possible workflow extensions include CRM integration, scheduling, analytics, and custom knowledge bases                              | Suggested enhancements for business use                                                        |

---

**Disclaimer:**  
The text and workflow described are exclusively derived from an automated n8n workflow. The processing strictly complies with content policies and contains no illegal or protected material. All data handled is legal and public.

---