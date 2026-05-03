Match Medical Symptoms to Products with OpenAI, Qdrant & Google Sheets RAG

https://n8nworkflows.xyz/workflows/match-medical-symptoms-to-products-with-openai--qdrant---google-sheets-rag-6699


# Match Medical Symptoms to Products with OpenAI, Qdrant & Google Sheets RAG

### 1. Workflow Overview

This workflow is designed to recommend medical products based on symptom descriptions using a Retrieval-Augmented Generation (RAG) approach with OpenAI, Qdrant vector database, and Google Sheets. It targets use cases where a user inputs medical symptoms or queries, and the system returns matched products by semantically searching a product database enriched with embeddings.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures user input either manually or via chat messages.
- **1.2 Product Data Preparation:** Retrieves product data from Google Sheets, processes it into embeddings, splits text for vector storage, and indexes it in Qdrant.
- **1.3 Vector Database Querying:** Uses embeddings of user queries to fetch relevant product information from Qdrant.
- **1.4 AI Processing and Response Generation:** Utilizes LangChain’s RAG agent with OpenAI LLM and memory buffers to generate contextual product recommendations.
- **1.5 Batch Processing Loop:** Iterates over products to process individual embeddings and queries.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
Captures user input either by manual triggering or chat webhook to initiate the recommendation process.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- When chat message received (Chat Trigger)

**Node Details:**  

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Starts workflow execution manually for testing or batch processing  
  - Connections: Outputs to “Get all products” node  
  - Edge cases: None (manual trigger requires user action)  

- **When chat message received**  
  - Type: LangChain Chat Trigger (Webhook)  
  - Role: Listens for incoming chat messages to trigger recommendation process dynamically  
  - Webhook ID: Present; allows external chat systems to send queries  
  - Output: Feeds into “RAG Agent” for processing  
  - Edge cases: Webhook authentication or network failure could cause missed triggers  

---

#### 2.2 Product Data Preparation

**Overview:**  
Loads product information from Google Sheets, processes textual data into manageable chunks, creates embeddings, and stores them properly in the Qdrant vector database.

**Nodes Involved:**  
- Get all products  
- Loop over each product  
- Create Embedding  
- Split text into chunks  
- Set Data Properly in vector database  
- Qdrant Vector Database  
- Create Embedding2  
- Get data from Qdrant database

**Node Details:**  

- **Get all products**  
  - Type: Google Sheets  
  - Role: Retrieves a list of all medical products with descriptions from a configured Google Sheets document  
  - Output connections: Feeds into “Loop over each product”  
  - Configuration: Requires Google Sheets credentials, sheet name, and range setup  
  - Edge cases: API quota limits, missing or malformed data  

- **Loop over each product**  
  - Type: Split In Batches  
  - Role: Iterates over each product row from Google Sheets for individual processing  
  - Inputs: From “Get all products” and from “Qdrant Vector Database” (for query results)  
  - Outputs: One branch to “Qdrant Vector Database” for vector queries; another branch empty (possibly for synchronous control)  
  - Edge cases: Large product lists may cause processing delays or timeouts  

- **Create Embedding**  
  - Type: OpenAI Embeddings (LangChain)  
  - Role: Generates vector embeddings for product textual data to enable semantic search  
  - Input: Product text or chunks from “Split text into chunks”  
  - Output: Feeds into “Qdrant Vector Database” for indexing  
  - Credential: OpenAI API key required  
  - Edge cases: API quota, input text length limits  

- **Split text into chunks**  
  - Type: Character Text Splitter (LangChain)  
  - Role: Divides large product descriptions into smaller chunks suitable for embedding and vector storage  
  - Input: Product descriptions  
  - Output: Feeds into “Set Data Properly in vector database”  
  - Edge cases: Improper chunk size could degrade embedding quality  

- **Set Data Properly in vector database**  
  - Type: Document Default Data Loader (LangChain)  
  - Role: Prepares and formats documents/chunks for insertion into Qdrant vector database  
  - Input: Chunks from text splitter  
  - Output: Feeds into “Qdrant Vector Database” as documents  
  - Edge cases: Data schema mismatch or missing metadata could cause insertion failures  

- **Qdrant Vector Database**  
  - Type: LangChain Qdrant Vector Store  
  - Role: Stores product embeddings and supports vector similarity search queries  
  - Inputs:  
    - Embeddings and documents for insertion (from “Set Data Properly in vector database”)  
    - Queries from “Loop over each product” for searching  
  - Outputs: Feeds retrieved product matches back into “Loop over each product”  
  - Credentials: Qdrant API or local instance connection details  
  - Edge cases: Connection issues, indexing errors, search failures  

- **Create Embedding2**  
  - Type: OpenAI Embeddings (LangChain)  
  - Role: Creates embeddings from query or data retrieved from Qdrant to be used in the RAG process  
  - Input: Data from “Get data from Qdrant database”  
  - Outputs: To “Get data from Qdrant database” (seems like a loop for refining embeddings)  
  - Edge cases: Same as “Create Embedding”  

- **Get data from Qdrant database**  
  - Type: LangChain Qdrant Vector Store (Query)  
  - Role: Retrieves nearest neighbor products based on query embeddings  
  - Input: Embeddings from “Create Embedding2”  
  - Output: Feeds back to “RAG Agent” as tool data  
  - Edge cases: Query errors, empty results  

---

#### 2.3 AI Processing and Response Generation

**Overview:**  
Processes user queries with LangChain’s RAG agent combining OpenAI’s LLM, memory buffer for chat context, and vector search results for contextual product recommendations.

**Nodes Involved:**  
- RAG Agent  
- OpenAI LLM  
- Store Chats (Memory Buffer Window)

**Node Details:**  

- **RAG Agent**  
  - Type: LangChain Agent  
  - Role: Core logic combining vector search results, chat memory, and OpenAI LLM to generate product recommendations  
  - Inputs:  
    - Main input from “When chat message received” or manual trigger chain  
    - AI memory input from “Store Chats” to provide conversational context  
    - AI tool input from “Get data from Qdrant database” for retrieval results  
    - AI language model input from “OpenAI LLM” for text generation  
  - Outputs: Final recommendation (not connected further explicitly in JSON, assumed to output to chat or UI)  
  - Edge cases: API rate limits, failed retrievals, incomplete memory context  

- **OpenAI LLM**  
  - Type: LangChain OpenAI Chat Model  
  - Role: Language model generating natural language responses based on input prompt and retrieval context  
  - Input: From “RAG Agent” as language model component  
  - Credentials: OpenAI API key  
  - Edge cases: Latency, token limits, API errors  

- **Store Chats**  
  - Type: Memory Buffer Window (LangChain)  
  - Role: Maintains windowed conversational context to enable multi-turn dialogue consistency  
  - Input: From “RAG Agent” (stores chat exchanges)  
  - Output: Provides chat memory back into “RAG Agent”  
  - Edge cases: Memory overflow or state desynchronization  

---

#### 2.4 Batch Processing Loop

**Overview:**  
Handles processing of products in batches for embedding generation and querying to avoid overload and organize the workflow logically.

**Nodes Involved:**  
- Loop over each product (already described in 2.2)  
- Qdrant Vector Database (receives batched inputs)

**Node Details:**  

- **Loop over each product**  
  - Splits the Google Sheets products list and processes each product individually or in batches  
  - Sends product data for embedding creation and Qdrant insertion or querying  
  - Supports scalable processing of large product catalogs  

- **Qdrant Vector Database**  
  - Receives batch inserts or queries, enabling scalable vector operations  

---

### 3. Summary Table

| Node Name                    | Node Type                            | Functional Role                           | Input Node(s)                      | Output Node(s)               | Sticky Note              |
|------------------------------|------------------------------------|-----------------------------------------|----------------------------------|-----------------------------|--------------------------|
| Sticky Note                  | Sticky Note                        | Visual annotation                       | —                                | —                           |                          |
| RAG Agent                   | LangChain Agent                    | Core AI processing and response generation | When chat message received, OpenAI LLM, Store Chats, Get data from Qdrant database | —                           |                          |
| When clicking ‘Execute workflow’ | Manual Trigger                    | Manual start of workflow                 | —                                | Get all products             |                          |
| Sticky Note1                 | Sticky Note                        | Visual annotation                       | —                                | —                           |                          |
| Qdrant Vector Database       | LangChain Qdrant Vector Store     | Vector database indexing & querying     | Set Data Properly in vector database, Loop over each product, Create Embedding | Loop over each product       |                          |
| Create Embedding             | OpenAI Embeddings (LangChain)     | Generate embeddings for product texts   | Split text into chunks            | Qdrant Vector Database       |                          |
| Set Data Properly in vector database | Document Default Data Loader (LangChain) | Format data for vector db insertion     | Split text into chunks            | Qdrant Vector Database       |                          |
| Split text into chunks       | Character Text Splitter (LangChain)| Split long texts for embedding          | Set Data Properly in vector database | Set Data Properly in vector database |                          |
| Create Embedding2            | OpenAI Embeddings (LangChain)     | Generate embeddings for query or retrieved data | Get data from Qdrant database    | Get data from Qdrant database |                          |
| Loop over each product       | Split In Batches                   | Batch processing of products             | Get all products, Qdrant Vector Database | Qdrant Vector Database       |                          |
| Get data from Qdrant database | LangChain Qdrant Vector Store     | Query vector database for matches        | Create Embedding2                | RAG Agent                   |                          |
| When chat message received   | LangChain Chat Trigger (Webhook)  | Receives chat queries                    | —                                | RAG Agent                   |                          |
| OpenAI LLM                  | LangChain OpenAI Chat Model        | Language generation for recommendations | RAG Agent (AI languageModel)      | RAG Agent                   |                          |
| Sticky Note2                 | Sticky Note                        | Visual annotation                       | —                                | —                           |                          |
| Sticky Note3                 | Sticky Note                        | Visual annotation                       | —                                | —                           |                          |
| Store Chats                 | Memory Buffer Window (LangChain)   | Maintains conversational context        | RAG Agent (ai_memory)             | RAG Agent (ai_memory)        |                          |
| Get all products             | Google Sheets                      | Retrieves products data                   | When clicking ‘Execute workflow’ | Loop over each product       |                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: To manually start the workflow for testing or batch processing.

2. **Create Google Sheets Node ("Get all products")**  
   - Type: Google Sheets  
   - Configure with credentials and specify the spreadsheet and sheet range containing product data (medical product names, descriptions).  
   - Connect output to next step.

3. **Create Split In Batches Node ("Loop over each product")**  
   - Type: Split In Batches  
   - Connect input from Google Sheets node.  
   - This will iterate over each product row for separate processing.

4. **Create Character Text Splitter Node ("Split text into chunks")**  
   - Type: LangChain Text Splitter (Character based)  
   - Configure chunk size appropriate to OpenAI embedding limits (~500-1000 characters).  
   - Connect input from product description field in current batch.

5. **Create Document Default Data Loader Node ("Set Data Properly in vector database")**  
   - Type: LangChain Document Loader  
   - Format chunks for vector database insertion, ensuring metadata like product ID/name is included.  
   - Connect input from Text Splitter output.

6. **Create OpenAI Embeddings Node ("Create Embedding")**  
   - Type: LangChain OpenAI Embeddings  
   - Configure with OpenAI API credentials.  
   - Connect input from Document Loader output to generate vector embeddings.

7. **Create Qdrant Vector Database Node ("Qdrant Vector Database")**  
   - Type: LangChain Qdrant Vector Store  
   - Configure connection settings for Qdrant instance or cloud API.  
   - Connect embeddings output from Create Embedding node for indexing.  
   - Also configure for querying later.

8. **Create OpenAI Embeddings Node ("Create Embedding2")**  
   - Type: LangChain OpenAI Embeddings  
   - Used to embed user queries or retrieved data for similarity search.  
   - Connect input from the "Get data from Qdrant database" node outputs.

9. **Create Qdrant Vector Database Node ("Get data from Qdrant database")**  
   - Type: LangChain Qdrant Vector Store (Query mode)  
   - Use to perform similarity search on the vector database for matching products.  
   - Connect input from Create Embedding2 node.  
   - Connect output to RAG Agent as AI tool data.

10. **Create LangChain Chat Trigger Node ("When chat message received")**  
    - Type: LangChain Chat Trigger (Webhook)  
    - Configure webhook URL to receive chat messages (symptoms or queries).  
    - Connect output to RAG Agent main input.

11. **Create LangChain Agent Node ("RAG Agent")**  
    - Type: LangChain Agent  
    - Configure with OpenAI LLM, memory buffer, and vector search tool inputs.  
    - Inputs:  
      - Main from chat trigger or manual trigger path  
      - AI language model from OpenAI LLM node  
      - AI memory from Store Chats node  
      - AI tool from Qdrant query node  
    - Outputs the final product recommendations.

12. **Create OpenAI LLM Node ("OpenAI LLM")**  
    - Type: LangChain OpenAI Chat Model  
    - Configure with OpenAI API credentials.  
    - Connect as AI language model input to the RAG Agent.

13. **Create Memory Buffer Node ("Store Chats")**  
    - Type: LangChain Memory Buffer Window  
    - Configure window size for conversational context.  
    - Connect input and output to/from RAG Agent for chat memory.

14. **Connect Manual Trigger to "Get all products"**  
    - To enable batch processing and indexing of products on demand.

15. **Connect "Loop over each product" to "Qdrant Vector Database"**  
    - For batch insertion and querying operations.

16. **Test the workflow manually by triggering and sending test chat messages via webhook**  
    - Validate response generation and product matching.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                 |
|----------------------------------------------------------------------------------------------------------|----------------------------------------------------------------|
| Qdrant vector database is used for high-performance vector similarity search to enable semantic matching.| https://qdrant.tech/                                            |
| OpenAI API credentials are required for embedding generation and language model usage.                    | https://platform.openai.com/docs/api-reference/authentication  |
| Google Sheets node requires OAuth2 credentials and correct sheet/range setup for product data retrieval. | https://developers.google.com/sheets/api                      |
| LangChain framework nodes are utilized for RAG, memory buffer, and text splitting functionalities.       | https://python.langchain.com/en/latest/index.html              |
| This workflow supports multi-turn chat via memory buffer to maintain context in recommendations.          |                                                                |

---

**Disclaimer:**  
The provided content derives exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.