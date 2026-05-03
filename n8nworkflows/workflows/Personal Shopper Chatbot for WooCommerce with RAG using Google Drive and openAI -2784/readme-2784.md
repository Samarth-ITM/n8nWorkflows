Personal Shopper Chatbot for WooCommerce with RAG using Google Drive and openAI 

https://n8nworkflows.xyz/workflows/personal-shopper-chatbot-for-woocommerce-with-rag-using-google-drive-and-openai--2784


# Personal Shopper Chatbot for WooCommerce with RAG using Google Drive and openAI 

### 1. Workflow Overview

This workflow implements a **Personal Shopper Chatbot** for WooCommerce stores, enhanced with a **Retrieval-Augmented Generation (RAG)** system using Google Drive and OpenAI. It serves e-commerce businesses by providing a hybrid assistant capable of:

- **Product Search**: Understanding user queries to extract search parameters (keywords, price ranges, SKUs, categories) and retrieving matching products from WooCommerce.
- **General Store Inquiries**: Answering questions about store policies, opening hours, and other general information by semantically searching documents stored in Google Drive via a Qdrant vector database.

The workflow is logically divided into three main blocks:

- **1.1 Chat Interaction & Intent Detection**: Receives user chat messages, extracts intent and search parameters using OpenAI.
- **1.2 Product Search via WooCommerce**: If the user is searching for products, queries WooCommerce with extracted filters.
- **1.3 General Inquiries via RAG**: If the user asks general questions, retrieves relevant information from Google Drive documents embedded in Qdrant and generates responses with OpenAI.

Supporting blocks handle document ingestion, vector embedding, and memory management for conversational context.

---

### 2. Block-by-Block Analysis

#### 2.1 Chat Interaction & Intent Detection

- **Overview:**  
  This block starts the workflow upon receiving a chat message, extracts session and chat input fields, and uses OpenAI to determine if the user is searching for a product or asking a general question. It extracts detailed search parameters if applicable.

- **Nodes Involved:**  
  - When chat message received  
  - Edit Fields  
  - Information Extractor  
  - OpenAI Chat Model2  
  - AI Agent  
  - Window Buffer Memory  
  - Calculator  

- **Node Details:**

  - **When chat message received**  
    - *Type:* Chat Trigger (LangChain)  
    - *Role:* Entry point triggered by incoming user chat messages.  
    - *Config:* Default webhook trigger, no special parameters.  
    - *Connections:* Outputs to Edit Fields.  
    - *Edge cases:* Webhook failures, malformed input.

  - **Edit Fields**  
    - *Type:* Set node  
    - *Role:* Extracts and assigns `sessionId` and `chatInput` from incoming JSON for downstream use.  
    - *Config:* Assigns `sessionId` and `chatInput` from incoming data.  
    - *Connections:* Outputs to Information Extractor.  
    - *Edge cases:* Missing or malformed session/chat input.

  - **Information Extractor**  
    - *Type:* LangChain Information Extractor  
    - *Role:* Uses OpenAI to analyze chat input and extract intent and product search parameters.  
    - *Config:* Custom system prompt tailored for a shoe and accessories store, instructing extraction of `search` (boolean), `keyword`, `priceRange` (min/max), `SKU`, and `category`.  
    - *Key Expressions:* Uses `chatInput` from Edit Fields.  
    - *Connections:* Outputs to AI Agent.  
    - *Edge cases:* Ambiguous user input, extraction errors, OpenAI API failures.

  - **OpenAI Chat Model2**  
    - *Type:* OpenAI Chat Model (LangChain)  
    - *Role:* Provides language model support for the Information Extractor node.  
    - *Config:* Uses OpenAI API credentials.  
    - *Connections:* Connected internally to Information Extractor.  
    - *Edge cases:* API key issues, rate limits.

  - **AI Agent**  
    - *Type:* LangChain Agent  
    - *Role:* Central decision-maker routing requests based on extracted intent. If `search: true`, calls WooCommerce product search; else calls RAG system.  
    - *Config:* System message instructs to parse input and route accordingly, passing JSON to `personal_shopper` tool or using `RAG` tool.  
    - *Key Expressions:* Uses `chatInput` and JSON output from Information Extractor.  
    - *Connections:* Outputs to personal_shopper, RAG, and Calculator nodes.  
    - *Edge cases:* Incorrect routing, malformed JSON, API failures.

  - **Window Buffer Memory**  
    - *Type:* LangChain Memory Buffer (Window)  
    - *Role:* Maintains conversational context per session using `sessionId`.  
    - *Config:* Session key set dynamically from Edit Fields.  
    - *Connections:* Feeds memory context into AI Agent.  
    - *Edge cases:* Session key mismatches, memory overflow.

  - **Calculator**  
    - *Type:* LangChain Tool Calculator  
    - *Role:* Provides calculation capabilities to AI Agent if needed.  
    - *Config:* Default.  
    - *Connections:* Connected as a tool to AI Agent.  
    - *Edge cases:* Calculation errors.

---

#### 2.2 Product Search via WooCommerce

- **Overview:**  
  When the user intent is product search, this block queries the WooCommerce store using extracted parameters to retrieve matching in-stock products.

- **Nodes Involved:**  
  - personal_shopper (WooCommerce Tool)  

- **Node Details:**

  - **personal_shopper**  
    - *Type:* WooCommerce Tool (n8n native node)  
    - *Role:* Queries WooCommerce API for products filtered by SKU, keyword, price range, and stock status.  
    - *Config:*  
      - SKU, keyword, minPrice, maxPrice dynamically set from Information Extractor output.  
      - Filters products with `stockStatus: "instock"`.  
      - Operation: `getAll` to fetch all matching products.  
    - *Credentials:* WooCommerce API credentials configured.  
    - *Connections:* Input from AI Agent tool routing.  
    - *Edge cases:* API authentication errors, no matching products, network timeouts.

---

#### 2.3 General Inquiries via RAG System

- **Overview:**  
  For general store-related questions, this block uses a RAG approach: documents stored in Google Drive are embedded into a Qdrant vector store, which is queried to retrieve relevant information. OpenAI then generates a natural language answer.

- **Nodes Involved:**  
  - RAG (Vector Store Tool)  
  - Qdrant Vector Store  
  - OpenAI Chat Model1  
  - Google Drive2  
  - Google Drive1  
  - Default Data Loader2  
  - Token Splitter1  
  - Embeddings OpenAI  
  - Embeddings OpenAI3  
  - Qdrant Vector Store1  
  - HTTP Request (for Qdrant collection cleanup)  
  - When clicking ‘Test workflow’ (manual trigger for setup)  

- **Node Details:**

  - **Google Drive2**  
    - *Type:* Google Drive node  
    - *Role:* Lists files in a specified Google Drive folder containing store documents.  
    - *Config:* Folder ID set to the store documents folder.  
    - *Credentials:* Google Drive OAuth2.  
    - *Connections:* Outputs file metadata to Google Drive1.  
    - *Edge cases:* Folder ID errors, permission issues.

  - **Google Drive1**  
    - *Type:* Google Drive node  
    - *Role:* Downloads each document file as plain text for processing.  
    - *Config:* Converts Google Docs to plain text.  
    - *Credentials:* Google Drive OAuth2.  
    - *Connections:* Outputs binary content to Default Data Loader2.  
    - *Edge cases:* Download failures, unsupported file types.

  - **Default Data Loader2**  
    - *Type:* LangChain Document Loader  
    - *Role:* Converts downloaded binary files into documents for embedding.  
    - *Config:* Data type set to binary.  
    - *Connections:* Outputs documents to Qdrant Vector Store1.  
    - *Edge cases:* Data parsing errors.

  - **Token Splitter1**  
    - *Type:* LangChain Text Splitter (Token-based)  
    - *Role:* Splits documents into chunks of 300 tokens with 30 tokens overlap for embedding.  
    - *Config:* chunkSize=300, chunkOverlap=30.  
    - *Connections:* Feeds into Default Data Loader2.  
    - *Edge cases:* Improper chunking, tokenization errors.

  - **Embeddings OpenAI3**  
    - *Type:* OpenAI Embeddings  
    - *Role:* Generates vector embeddings for document chunks.  
    - *Config:* Uses OpenAI API credentials.  
    - *Connections:* Outputs embeddings to Qdrant Vector Store1.  
    - *Edge cases:* API failures, rate limits.

  - **Qdrant Vector Store1**  
    - *Type:* Qdrant Vector Store (Insert mode)  
    - *Role:* Inserts document embeddings into the Qdrant collection named "scarperia".  
    - *Config:* Collection name set to "scarperia".  
    - *Credentials:* Qdrant API credentials.  
    - *Connections:* Receives embeddings from Embeddings OpenAI3.  
    - *Edge cases:* Connection failures, collection misconfiguration.

  - **HTTP Request**  
    - *Type:* HTTP Request  
    - *Role:* Clears the Qdrant collection by deleting all points (used during setup or reset).  
    - *Config:* POST request to Qdrant collection delete endpoint with empty filter.  
    - *Credentials:* HTTP header auth for Qdrant API.  
    - *Connections:* Triggered manually by "When clicking ‘Test workflow’".  
    - *Edge cases:* Authentication errors, network issues.

  - **When clicking ‘Test workflow’**  
    - *Type:* Manual Trigger  
    - *Role:* Allows manual triggering of the Qdrant collection cleanup.  
    - *Connections:* Outputs to HTTP Request.  
    - *Edge cases:* None.

  - **Qdrant Vector Store**  
    - *Type:* Qdrant Vector Store (Query mode)  
    - *Role:* Queries the Qdrant collection to retrieve relevant documents based on user questions.  
    - *Config:* Collection "scarperia" used.  
    - *Credentials:* Qdrant API credentials.  
    - *Connections:* Outputs to RAG node.  
    - *Edge cases:* Query failures, empty results.

  - **RAG**  
    - *Type:* LangChain Vector Store Tool  
    - *Role:* Uses retrieved documents from Qdrant and OpenAI Chat Model1 to generate answers to user inquiries.  
    - *Config:* Description set to store-related info (opening hours, contacts, policies).  
    - *Connections:* Receives vector store results and outputs final answer.  
    - *Edge cases:* No relevant documents found, OpenAI failures.

  - **OpenAI Chat Model1**  
    - *Type:* OpenAI Chat Model  
    - *Role:* Generates natural language responses based on retrieved documents.  
    - *Config:* Uses OpenAI API credentials.  
    - *Connections:* Connected internally to RAG.  
    - *Edge cases:* API failures.

---

### 3. Summary Table

| Node Name                 | Node Type                              | Functional Role                                  | Input Node(s)               | Output Node(s)               | Sticky Note                                                                                              |
|---------------------------|--------------------------------------|-------------------------------------------------|-----------------------------|-----------------------------|--------------------------------------------------------------------------------------------------------|
| When chat message received| Chat Trigger (LangChain)              | Entry point for chat messages                    |                             | Edit Fields                 |                                                                                                        |
| Edit Fields               | Set                                  | Extracts sessionId and chatInput                 | When chat message received  | Information Extractor       |                                                                                                        |
| Information Extractor     | LangChain Information Extractor      | Extracts intent and product search parameters    | Edit Fields                 | AI Agent                   | Step 2: Extracts product-related info or routes to RAG system                                          |
| OpenAI Chat Model2        | OpenAI Chat Model                    | Supports Information Extractor                    |                             | Information Extractor       |                                                                                                        |
| AI Agent                  | LangChain Agent                      | Routes requests to product search or RAG         | Information Extractor       | personal_shopper, RAG, Calculator |                                                                                                        |
| Window Buffer Memory      | LangChain Memory Buffer (Window)    | Maintains conversation context                    |                             | AI Agent                   |                                                                                                        |
| Calculator                | LangChain Tool Calculator            | Provides calculation capabilities                  |                             | AI Agent                   |                                                                                                        |
| personal_shopper          | WooCommerce Tool                    | Queries WooCommerce for products                   | AI Agent                   |                             |                                                                                                        |
| RAG                      | LangChain Vector Store Tool          | Generates answers from retrieved documents        | AI Agent                   |                             |                                                                                                        |
| Qdrant Vector Store       | Qdrant Vector Store (Query mode)     | Queries vector database for relevant documents    |                             | RAG                        |                                                                                                        |
| OpenAI Chat Model1        | OpenAI Chat Model                    | Generates natural language answers                 |                             | RAG                        |                                                                                                        |
| Google Drive2             | Google Drive                        | Lists files in Google Drive folder                 | HTTP Request                | Google Drive1              |                                                                                                        |
| Google Drive1             | Google Drive                        | Downloads files as plain text                       | Google Drive2               | Default Data Loader2       |                                                                                                        |
| Default Data Loader2      | LangChain Document Loader            | Converts files to documents                         | Token Splitter1             | Qdrant Vector Store1       |                                                                                                        |
| Token Splitter1           | LangChain Text Splitter (Token)      | Splits documents into chunks                        |                             | Default Data Loader2       |                                                                                                        |
| Embeddings OpenAI3        | OpenAI Embeddings                   | Embeds document chunks                              |                             | Qdrant Vector Store1       |                                                                                                        |
| Qdrant Vector Store1      | Qdrant Vector Store (Insert mode)    | Inserts embeddings into Qdrant collection           | Default Data Loader2        |                             |                                                                                                        |
| HTTP Request              | HTTP Request                       | Deletes all points in Qdrant collection (cleanup) | When clicking ‘Test workflow’ | Google Drive2              | Replace the URL and Collection name with your own                                                      |
| When clicking ‘Test workflow’ | Manual Trigger                    | Manual trigger for Qdrant cleanup                   |                             | HTTP Request               |                                                                                                        |
| Sticky Note               | Sticky Note                        | Instructional notes                                 |                             |                             | Step 1: Create Qdrant collection and upload documents to Google Drive                                  |
| Sticky Note1              | Sticky Note                        | Instructional notes                                 |                             |                             | Step 1: Create a collection on your Qdrant instance and embed Google Drive documents                   |
| Sticky Note2              | Sticky Note                        | Instructional notes                                 |                             |                             | Step 1: Create a basic RAG system with documents uploaded to Google Drive and embedded in Qdrant       |
| Sticky Note3              | Sticky Note                        | Instructional notes                                 |                             |                             | Step 2: Information Extractor determines product search or general question and routes accordingly     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Chat Trigger Node**  
   - Add `When chat message received` (LangChain Chat Trigger).  
   - Configure webhook to receive chat messages.  

2. **Add Set Node to Extract Fields**  
   - Create `Edit Fields` node (Set type).  
   - Assign `sessionId` = `{{$json.sessionId}}`  
   - Assign `chatInput` = `{{$json.chatInput}}`  
   - Connect output of Chat Trigger to this node.  

3. **Add Information Extractor Node**  
   - Add `Information Extractor` (LangChain Information Extractor).  
   - Set input text to `={{ $json.chatInput }}`.  
   - Use a custom system prompt tailored to your store to extract:  
     - `search` (boolean)  
     - `keyword`  
     - `priceRange` (min and max)  
     - `SKU`  
     - `category`  
   - Connect output of Edit Fields to this node.  
   - Attach an OpenAI Chat Model node (`OpenAI Chat Model2`) with your OpenAI API credentials to support this extractor.  

4. **Add AI Agent Node**  
   - Add `AI Agent` (LangChain Agent).  
   - Set input text to `={{ $('When chat message received').item.json.chatInput }}`.  
   - Configure system message to:  
     - Check if `search` is true in the JSON output from Information Extractor.  
     - If true, route to `personal_shopper` tool with extracted parameters.  
     - Else, route to `RAG` tool for general inquiries.  
   - Connect output of Information Extractor to AI Agent.  
   - Attach `Window Buffer Memory` node configured with session key from `Edit Fields` to maintain context.  
   - Attach `Calculator` tool node for calculation support.  
   - Connect OpenAI Chat Model node (`OpenAI Chat Model`) with your OpenAI credentials to AI Agent.  

5. **Configure WooCommerce Product Search Node**  
   - Add `personal_shopper` node (WooCommerce Tool).  
   - Configure operation `getAll`.  
   - Set parameters dynamically from Information Extractor output:  
     - SKU: `={{ $('Information Extractor').item.json.output.SKU }}`  
     - Keyword: `={{ $('Information Extractor').item.json.output.keyword }}`  
     - Min Price: `={{ $('Information Extractor').item.json.output.price_min }}`  
     - Max Price: `={{ $('Information Extractor').item.json.output.price_max }}`  
     - Stock Status: `"instock"`  
   - Connect AI Agent output to this node.  
   - Add WooCommerce API credentials.  

6. **Set Up RAG System for General Inquiries**  

   - **Google Drive Integration:**  
     - Add `Google Drive2` node to list files in your store documents folder.  
     - Configure with your Google Drive OAuth2 credentials and folder ID.  
     - Connect output to `Google Drive1`.  

   - **File Download:**  
     - Add `Google Drive1` node to download files as plain text.  
     - Connect output to `Default Data Loader2`.  

   - **Document Loading and Splitting:**  
     - Add `Token Splitter1` node to split text into 300-token chunks with 30-token overlap.  
     - Connect to `Default Data Loader2` to convert to documents.  

   - **Embedding Documents:**  
     - Add `Embeddings OpenAI3` node with OpenAI credentials to embed document chunks.  
     - Connect to `Qdrant Vector Store1` node (Insert mode) configured with your Qdrant API credentials and collection name (e.g., "scarperia").  

   - **Qdrant Collection Cleanup:**  
     - Add `When clicking ‘Test workflow’` manual trigger node.  
     - Connect to `HTTP Request` node configured to POST to your Qdrant collection delete endpoint with empty filter to clear points.  
     - Configure HTTP Request with Qdrant API credentials and proper headers.  

7. **Querying RAG System:**  
   - Add `Qdrant Vector Store` node (Query mode) configured with your Qdrant collection and credentials.  
   - Connect output to `RAG` node (LangChain Vector Store Tool).  
   - Add `OpenAI Chat Model1` node with OpenAI credentials connected to `RAG` for generating answers.  
   - Connect AI Agent output to `RAG` node for general inquiries.  

8. **Finalize Connections:**  
   - Ensure AI Agent routes to `personal_shopper` node if `search` is true.  
   - Ensure AI Agent routes to `RAG` node if `search` is false.  
   - Connect memory and calculator tools to AI Agent as tools.  

9. **Credentials Setup:**  
   - Add OpenAI API credentials to all OpenAI nodes.  
   - Add WooCommerce API credentials to `personal_shopper`.  
   - Add Google Drive OAuth2 credentials to Google Drive nodes.  
   - Add Qdrant API credentials to Qdrant nodes and HTTP Request.  

10. **Testing and Validation:**  
    - Use manual trigger to clear Qdrant collection and reload documents.  
    - Test chat messages for both product search and general inquiries.  
    - Adjust system prompts and parameters as needed.  

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                      |
|----------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This template is ideal for e-commerce businesses needing a hybrid assistant for product discovery and customer support. | Workflow description                                                                                 |
| Replace the URL and Collection name with your own in the HTTP Request node for Qdrant collection cleanup. | Sticky Note on HTTP Request node                                                                    |
| Step 1: Create a collection on your Qdrant instance and upload documents to Google Drive for embedding.  | Sticky Notes 1 and 2                                                                                 |
| Step 2: The Information Extractor tries to understand if the request is product-related or a general question and routes accordingly. | Sticky Note 3                                                                                       |
| For detailed WooCommerce API setup, refer to official WooCommerce REST API documentation.                 | https://woocommerce.github.io/woocommerce-rest-api-docs/                                           |
| For OpenAI API usage and rate limits, consult OpenAI official docs.                                      | https://platform.openai.com/docs/                                                                  |
| For Qdrant vector database setup and API, see Qdrant official documentation.                             | https://qdrant.tech/documentation/                                                                 |
| For Google Drive API and OAuth2 setup, see Google Drive API docs.                                        | https://developers.google.com/drive/api/v3/about-sdk                                               |

---

This structured documentation enables developers and AI agents to understand, reproduce, and extend the Personal Shopper Chatbot workflow with confidence, anticipating integration points and potential failure modes.