WhatsApp RAG Chatbot with Supabase, Gemini 2.5 Flash, and OpenAI Embeddings

https://n8nworkflows.xyz/workflows/whatsapp-rag-chatbot-with-supabase--gemini-2-5-flash--and-openai-embeddings-6771


# WhatsApp RAG Chatbot with Supabase, Gemini 2.5 Flash, and OpenAI Embeddings

### 1. Workflow Overview

This workflow implements a WhatsApp-based Retrieval-Augmented Generation (RAG) chatbot that integrates Supabase vector storage, Google Gemini 2.5 Flash language model, and OpenAI embeddings. It enables users to interact with the bot via WhatsApp to either upload documents or ask natural language queries. Uploaded documents are converted to text, embedded using OpenAI embeddings, and stored in Supabase for semantic search. User queries are transformed into embeddings, matched against the stored vectors to retrieve relevant context, and the context is passed to Gemini 2.5 Flash to generate concise, context-aware answers. The answers are sent back to the user on WhatsApp.

The workflow can be logically divided into these blocks:

- **1.1 WhatsApp Message Reception & Classification:** Triggered by incoming WhatsApp messages; determines if the message is a text query or a document upload.
- **1.2 Document Processing Flow:** For document uploads, downloads the file, converts it to text, generates embeddings, and stores them in Supabase.
- **1.3 Query Processing Flow:** For text queries, generates embeddings, retrieves relevant context from Supabase, sends context and query to Google Gemini LLM, and returns the generated answer.
- **1.4 WhatsApp Reply:** Sends the generated answer back to the WhatsApp user.

---

### 2. Block-by-Block Analysis

---

#### 1.1 WhatsApp Message Reception & Classification

**Overview:**  
This block listens to incoming WhatsApp messages and classifies them into either a text query or a document upload, routing the workflow accordingly.

**Nodes Involved:**  
- New WhatsApp Message  
- Check if Query or Document  
- Sticky Note2 (explanatory)

**Node Details:**  

- **New WhatsApp Message**  
  - *Type:* WhatsApp Trigger  
  - *Role:* Entry point; triggers the workflow on every new WhatsApp message.  
  - *Configuration:* Watches for message updates (`messages`), authorized via WhatsApp OAuth2 credentials.  
  - *Inputs:* Webhook event from WhatsApp Business API.  
  - *Outputs:* Message JSON data forwarded to the switch node.  
  - *Failures:* Possible webhook connectivity or OAuth token expiration errors.  
  - *Sticky Note:* Sticky Note2 explains this node’s function.

- **Check if Query or Document**  
  - *Type:* Switch  
  - *Role:* Routes messages to different flows based on presence of text or document.  
  - *Configuration:* Two conditions:  
    - If message has `messages[0].text` property → route to "query" output.  
    - If message has `messages[0].document` property → route to "document" output.  
  - *Inputs:* JSON message from "New WhatsApp Message".  
  - *Outputs:* Two outputs — "query" output leads to query processing; "document" output leads to document processing.  
  - *Failures:* Errors if message structure unexpected or missing required fields.  
  - *Sticky Note:* Sticky Note2 describes the decision logic here.

---

#### 1.2 Document Processing Flow

**Overview:**  
Handles WhatsApp document uploads by downloading the document, converting it to text, generating embeddings, and storing them in Supabase.

**Nodes Involved:**  
- Get Document URL  
- Download WhatsApp Document  
- Convert File to Text  
- Generate OpenAI Embeddings  
- Store Embeddings in Supabase  
- Sticky Note (Document Flow)  
- Sticky Note4 (Upload Flow example)  

**Node Details:**  

- **Get Document URL**  
  - *Type:* WhatsApp node (Media URL Get)  
  - *Role:* Retrieves the downloadable URL for the WhatsApp document.  
  - *Configuration:* Uses `mediaGetId` extracted from incoming message document ID.  
  - *Inputs:* Document ID from switch node output.  
  - *Outputs:* JSON with document download URL.  
  - *Failures:* Invalid media ID, WhatsApp API errors, auth failures.  

- **Download WhatsApp Document**  
  - *Type:* HTTP Request  
  - *Role:* Downloads the document content from the URL obtained.  
  - *Configuration:* Uses HTTP Header Authentication with WhatsApp credentials. URL dynamically set from previous node output.  
  - *Inputs:* URL from "Get Document URL".  
  - *Outputs:* Binary data of the document file.  
  - *Failures:* HTTP errors, auth token expiration, timeout, invalid URL.  

- **Convert File to Text**  
  - *Type:* LangChain Document Default Data Loader  
  - *Role:* Converts binary document file into readable text for embedding.  
  - *Configuration:* Input data type is binary. Default internal document parsing used.  
  - *Inputs:* Binary file from HTTP request node.  
  - *Outputs:* Text content extracted from the document.  
  - *Failures:* Unsupported file format, conversion errors.  

- **Generate OpenAI Embeddings**  
  - *Type:* LangChain OpenAI Embeddings  
  - *Role:* Generates vector embeddings from the extracted text.  
  - *Configuration:* Uses OpenAI API credentials. Default embedding model and parameters.  
  - *Inputs:* Text from document conversion node.  
  - *Outputs:* Embeddings vector data for storage.  
  - *Failures:* OpenAI API rate limits, invalid API keys.  

- **Store Embeddings in Supabase**  
  - *Type:* LangChain Supabase Vector Store node  
  - *Role:* Inserts the generated embeddings and document metadata into the Supabase vector store table `documents`.  
  - *Configuration:* Insert mode; target table `"documents"`; Supabase API credentials provided.  
  - *Inputs:* Embeddings and document metadata.  
  - *Outputs:* Confirmation of storage.  
  - *Failures:* Supabase connectivity, auth failures, data constraints.  

- **Sticky Note** (Document Flow)  
  - Describes the steps in the document upload flow.  

- **Sticky Note4**  
  - Visual example showing document upload detection in workflow.  

---

#### 1.3 Query Processing Flow

**Overview:**  
Processes user text queries by generating embeddings, retrieving relevant context from Supabase, sending context and query to Google Gemini LLM, and producing a response.

**Nodes Involved:**  
- RAG Query Agent  
- Retrieve Context from Supabase  
- Google Gemini LLM  
- Generate OpenAI Embeddings (shared with document flow)  
- Sticky Note1 (Query Flow)  
- Sticky Note5 (Contextual Answer example)  

**Node Details:**  

- **RAG Query Agent**  
  - *Type:* LangChain Agent  
  - *Role:* Central orchestrator that accepts the query text, integrates retrieved context, and requests an answer from the LLM.  
  - *Configuration:* Receives query text from incoming WhatsApp message, configured with a “define” prompt type.  
  - *Inputs:* Query text, context from Supabase retrieval, and LLM output.  
  - *Outputs:* Generated answer JSON with output text.  
  - *Failures:* Missing input data, prompt misconfiguration, LLM API errors.  

- **Retrieve Context from Supabase**  
  - *Type:* LangChain Supabase Vector Store (retrieve-as-tool mode)  
  - *Role:* Retrieves top matching documents from Supabase vector store according to the query embedding.  
  - *Configuration:* Retrieval mode, table `"documents"`, with a tool description to guide usage.  
  - *Inputs:* Embeddings generated from query text.  
  - *Outputs:* Array of relevant document context for the agent.  
  - *Failures:* Supabase access errors, empty results, or malformed queries.  

- **Google Gemini LLM**  
  - *Type:* LangChain Google Gemini Chat LLM  
  - *Role:* Language model that generates the final answer based on the query and retrieved context.  
  - *Configuration:* Uses Google Palm API credentials, default parameters.  
  - *Inputs:* Prompt text and context from RAG agent.  
  - *Outputs:* LLM-generated answer text.  
  - *Failures:* API quota limits, auth failures, network errors.  

- **Generate OpenAI Embeddings**  
  - *Shared node with document flow*  
  - *Role:* Generates embeddings from user query text to enable semantic search.  
  - *Inputs:* Query text.  
  - *Outputs:* Embeddings fed to Supabase retrieval node.  

- **Sticky Note1**  
  - Explains the query flow steps.  

- **Sticky Note5**  
  - Shows example of contextual answer retrieval and response.  

---

#### 1.4 WhatsApp Reply

**Overview:**  
Sends the generated response from the RAG agent back to the user on WhatsApp.

**Nodes Involved:**  
- Send WhatsApp Reply  

**Node Details:**  

- **Send WhatsApp Reply**  
  - *Type:* WhatsApp node (Send Message)  
  - *Role:* Sends the generated answer text back to the user's WhatsApp number.  
  - *Configuration:* Uses WhatsApp API credentials; dynamically sets recipient phone number from incoming message contact (`wa_id`); sends text body from RAG Query Agent output.  
  - *Inputs:* Text answer from RAG Query Agent.  
  - *Outputs:* Confirmation of message sent.  
  - *Failures:* WhatsApp API rate limits, invalid phone numbers, auth errors.  

---

### 3. Summary Table

| Node Name                   | Node Type                                  | Functional Role                          | Input Node(s)                    | Output Node(s)                    | Sticky Note                                                                                                            |
|-----------------------------|--------------------------------------------|----------------------------------------|---------------------------------|----------------------------------|------------------------------------------------------------------------------------------------------------------------|
| New WhatsApp Message         | WhatsApp Trigger                           | Receive incoming WhatsApp messages     | —                               | Check if Query or Document       | See Sticky Note2: Message Check - determines if message is query or document upload                                     |
| Check if Query or Document   | Switch                                    | Classify message as query or document  | New WhatsApp Message             | RAG Query Agent, Get Document URL| See Sticky Note2                                                                                                        |
| Get Document URL             | WhatsApp (Media URL Get)                   | Get downloadable URL for document      | Check if Query or Document       | Download WhatsApp Document        |                                                                                                                        |
| Download WhatsApp Document   | HTTP Request                              | Download document binary file          | Get Document URL                 | Convert File to Text              |                                                                                                                        |
| Convert File to Text         | LangChain Document Data Loader            | Convert file binary to text            | Download WhatsApp Document       | Store Embeddings in Supabase     | See Sticky Note (Document Flow)                                                                                        |
| Generate OpenAI Embeddings   | LangChain OpenAI Embeddings                | Generate embeddings from text          | Convert File to Text, Query Text | Store Embeddings in Supabase, Retrieve Context from Supabase |                                                                                                                        |
| Store Embeddings in Supabase | LangChain Supabase Vector Store (Insert) | Store embeddings and metadata          | Convert File to Text, Generate OpenAI Embeddings | —                            |                                                                                                                        |
| Retrieve Context from Supabase | LangChain Supabase Vector Store (Retrieve) | Retrieve relevant context vectors      | Generate OpenAI Embeddings       | RAG Query Agent                  |                                                                                                                        |
| Google Gemini LLM            | LangChain Google Gemini Chat LLM           | Generate answer from context & query  | RAG Query Agent                 | RAG Query Agent                  |                                                                                                                        |
| RAG Query Agent             | LangChain Agent                            | Orchestrate query processing & response | Check if Query or Document (query), Retrieve Context from Supabase, Google Gemini LLM | Send WhatsApp Reply             |                                                                                                                        |
| Send WhatsApp Reply          | WhatsApp Send Message                      | Send answer back to WhatsApp user      | RAG Query Agent                 | —                              |                                                                                                                        |
| Sticky Note                 | Sticky Note                               | Documentation and explanations         | —                               | —                              | See Sticky Note (Document Flow)                                                                                        |
| Sticky Note1                | Sticky Note                               | Documentation of Query Flow             | —                               | —                              |                                                                                                                        |
| Sticky Note2                | Sticky Note                               | Explanation of message classification   | —                               | —                              |                                                                                                                        |
| Sticky Note3                | Sticky Note                               | Overview, use cases, instructions       | —                               | —                              | Covers entire workflow overview and usage instructions                                                                  |
| Sticky Note4                | Sticky Note                               | Document Upload example                  | —                               | —                              |                                                                                                                        |
| Sticky Note5                | Sticky Note                               | Contextual Answer example                | —                               | —                              |                                                                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create WhatsApp Trigger Node**  
   - Node: *New WhatsApp Message*  
   - Type: WhatsApp Trigger  
   - Configure to listen for new messages (`messages` update event).  
   - Set up WhatsApp OAuth2 credentials with WhatsApp Business API access.  
   - Position: Entry node.

2. **Add Switch Node to Classify Message**  
   - Node: *Check if Query or Document*  
   - Type: Switch  
   - Add two rules:  
     - Output "query": If `messages[0].text` exists.  
     - Output "document": If `messages[0].document` exists.  
   - Connect *New WhatsApp Message* main output to this node.

3. **Document Processing Branch:**  
   a. **WhatsApp Media URL Get Node**  
      - Node: *Get Document URL*  
      - Type: WhatsApp (Media URL Get)  
      - Configure `mediaGetId` with expression referencing `messages[0].document.id`.  
      - Use WhatsApp API credentials.  
      - Connect "document" output from switch here.

   b. **HTTP Request Node to Download Document**  
      - Node: *Download WhatsApp Document*  
      - Type: HTTP Request  
      - URL: Use expression to get `url` from previous node output.  
      - Authentication: HTTP Header Auth with WhatsApp credentials.  
      - Connect output of *Get Document URL* here.

   c. **Document Conversion Node**  
      - Node: *Convert File to Text*  
      - Type: LangChain Document Default Data Loader  
      - Input data type: binary.  
      - Connect output of HTTP Request node here.

   d. **OpenAI Embeddings Node**  
      - Node: *Generate OpenAI Embeddings*  
      - Type: LangChain OpenAI Embeddings  
      - Configure OpenAI API credentials.  
      - Connect output text from document conversion node here.

   e. **Store Embeddings in Supabase Node**  
      - Node: *Store Embeddings in Supabase*  
      - Type: LangChain Supabase Vector Store (Insert mode)  
      - Set target table to `documents`.  
      - Provide Supabase API credentials.  
      - Connect output from embeddings node here.

4. **Query Processing Branch:**  
   a. **Generate OpenAI Embeddings for Query**  
      - Reuse *Generate OpenAI Embeddings* node or create new instance.  
      - Input: Expression referencing `messages[0].text`.  
      - Connect "query" output from switch node here.

   b. **Retrieve Context from Supabase**  
      - Node: *Retrieve Context from Supabase*  
      - Type: LangChain Supabase Vector Store (Retrieve mode, retrieve-as-tool)  
      - Table: `documents`.  
      - Connect output of embeddings node here.

   c. **Google Gemini LLM Node**  
      - Node: *Google Gemini LLM*  
      - Type: LangChain Google Gemini Chat LLM  
      - Configure Google Palm API credentials.  
      - Connect output from *Retrieve Context from Supabase* to this node’s `ai_tool` input.

   d. **RAG Query Agent Node**  
      - Node: *RAG Query Agent*  
      - Type: LangChain Agent  
      - Set input text as expression `{{ $json.messages[0].text }}`.  
      - Connect `ai_embedding` input from *Generate OpenAI Embeddings* (query).  
      - Connect `ai_tool` input from *Retrieve Context from Supabase*.  
      - Connect `ai_languageModel` input from *Google Gemini LLM*.  
      - Connect "main" output of switch node (query output) to this node.

5. **WhatsApp Reply Node:**  
   - Node: *Send WhatsApp Reply*  
   - Type: WhatsApp Send Message  
   - Text body: Use expression to map answer text from *RAG Query Agent* output JSON (e.g., `{{ $json.output }}`).  
   - Recipient phone number: `{{ $('New WhatsApp Message').item.json.contacts[0].wa_id }}`.  
   - Configure WhatsApp API credentials for sending messages.  
   - Connect main output of *RAG Query Agent* here.

6. **Sticky Notes (Optional):**  
   - Add sticky notes for documentation and clarity as per logical blocks and overall workflow description.

7. **Credential Setup:**  
   - WhatsApp OAuth2 API credential with WhatsApp Business API or Twilio sandbox.  
   - Supabase API credential with access to the vector database and `documents` table.  
   - OpenAI API credential for embeddings generation.  
   - Google Palm API credential for Gemini 2.5 Flash LLM access.  
   - HTTP Header Auth credential for WhatsApp document download.

8. **Validation & Testing:**  
   - Test document upload via WhatsApp to verify storage flow.  
   - Test text query to verify context retrieval and response generation.  
   - Ensure all API keys are valid and rate limits are handled gracefully.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Context or Link                                                                                              |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| This workflow turns WhatsApp into a knowledge assistant using RAG with Supabase vector store, OpenAI embeddings, and Google Gemini 2.5 Flash LLM. Suitable for FAQs, customer support, or internal knowledge bases.                                                                                                                                                                                                                                                                                                                         | Sticky Note3 provides detailed overview, use cases, and instructions.                                        |
| Example screenshots demonstrating document upload flow and contextual answer flow are available in Sticky Note4 and Sticky Note5 respectively.                                                                                                                                                                                                                                                                                                                                                                                            | Visual aids included in workflow sticky notes.                                                               |
| For help or feedback, reach out on X (formerly Twitter) at [https://x.com/manav170303](https://x.com/manav170303) or email titanfactz@gmail.com.                                                                                                                                                                                                                                                                                                                                                                                           | Contact information for support and collaboration.                                                           |
| Requirements include WhatsApp Business API (or Twilio sandbox), Supabase vector storage, OpenAI API key, and Gemini API access.                                                                                                                                                                                                                                                                                                                                                                                                             | Critical to prepare credentials and accounts before deployment.                                              |
| The workflow strictly respects data privacy and content policies; all processed data is legal and public.                                                                                                                                                                                                                                                                                                                                                                                                                                  | Disclaimer for legal and ethical compliance.                                                                  |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow built with n8n, respecting all content policies and containing no illegal or offensive material. All data handled is legal and publicly accessible.