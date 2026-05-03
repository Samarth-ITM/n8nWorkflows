PDF Proposal Knowledge Base with S3, OpenAI GPT-4o & Qdrant RAG Agent

https://n8nworkflows.xyz/workflows/pdf-proposal-knowledge-base-with-s3--openai-gpt-4o---qdrant-rag-agent-7667


# PDF Proposal Knowledge Base with S3, OpenAI GPT-4o & Qdrant RAG Agent

---

### 1. Workflow Overview

This workflow, titled **"PDF Proposal Knowledge Base with S3, OpenAI GPT-4o & Qdrant RAG Agent"**, automates the ingestion, processing, indexing, and querying of PDF proposal documents stored in an AWS S3 bucket. It leverages advanced AI and vector search technologies to build a Retrieval-Augmented Generation (RAG) system that enables conversational question answering over the indexed knowledge base.

The workflow is logically divided into two main blocks:

- **1.1 Document Ingestion and Indexing**: Fetch PDF files from an S3 bucket, extract their text content, split the text for embedding, generate vector embeddings using OpenAI embeddings, and insert these into a Qdrant vector store for later retrieval.

- **1.2 AI Conversational Query Handling**: Triggered by incoming chat messages, this block uses the Qdrant vector store as a knowledge base tool to fetch relevant document context, employs the OpenAI GPT-4o-mini chat model as the language model, and outputs AI-generated responses based on the retrieved knowledge.

This structure supports ongoing updates to the knowledge base as new PDFs arrive and real-time AI-powered interaction for proposal-related inquiries.

---

### 2. Block-by-Block Analysis

#### 2.1 Document Ingestion and Indexing

**Overview:**  
This block automates downloading all PDFs from a configured AWS S3 bucket, processes each document by extracting its text, generating embeddings, and indexing the data into a Qdrant vector collection to build a searchable knowledge base.

**Nodes Involved:**  
- When clicking ‘Test workflow’  
- Get Files from S3  
- Loop Over Items  
- Download Files from AWS  
- Extract from File  
- Recursive Character Text Splitter  
- Default Data Loader  
- Embeddings OpenAI  
- Qdrant Vector Store  

**Node Details:**

- **When clicking ‘Test workflow’**  
  - *Type:* Manual Trigger  
  - *Role:* Starts the workflow manually for testing or batch execution.  
  - *Config:* No parameters; triggers workflow execution on demand.  
  - *Connections:* Output to "Get Files from S3".  
  - *Failure Modes:* None (manual trigger).  

- **Get Files from S3**  
  - *Type:* AWS S3 Node (Get All Operation)  
  - *Role:* Lists all files in the specified S3 bucket.  
  - *Config:* Bucket name set to `YOUR_S3_BUCKET`. Retrieves metadata of all objects.  
  - *Connections:* Outputs file list to "Loop Over Items".  
  - *Failure Modes:* Authentication errors, bucket not found, network timeouts.  
  - *Notes:* Requires AWS credentials configured with read access.  

- **Loop Over Items**  
  - *Type:* Split In Batches  
  - *Role:* Iterates through each file item from the S3 listing for sequential processing.  
  - *Config:* Batch size dynamically set to the length of the `Key` array from S3 files, meaning all items processed in one batch (acts as a loop).  
  - *Connections:* Output batch items to "Download Files from AWS".  
  - *Failure Modes:* Expression errors if input data malformed.  

- **Download Files from AWS**  
  - *Type:* AWS S3 Node (Download File)  
  - *Role:* Downloads the actual PDF binary data for each file key.  
  - *Config:* Bucket name `YOUR_S3_BUCKET`; file key dynamically from each item’s `Key` property.  
  - *Connections:* Binary output sent to "Extract from File".  
  - *Failure Modes:* File not found, permission errors, network interruptions.  

- **Extract from File**  
  - *Type:* Extract From File  
  - *Role:* Extracts text content from downloaded PDF binary data.  
  - *Config:* Operation set to `pdf`; binary input from `data` property containing the file.  
  - *Connections:* Outputs extracted text to "Qdrant Vector Store".  
  - *Failure Modes:* PDF parsing errors, corrupted files, unsupported PDF formats.  

- **Recursive Character Text Splitter**  
  - *Type:* Text Splitter (Recursive Character)  
  - *Role:* Splits the extracted text into smaller chunks suitable for embedding.  
  - *Config:* Default splitter options (recursive character-based splitting).  
  - *Connections:* Outputs chunks to "Default Data Loader".  
  - *Failure Modes:* Empty text input, improper splitting if text is malformed.  

- **Default Data Loader**  
  - *Type:* Document Default Data Loader  
  - *Role:* Converts text chunks into document objects for embedding ingestion.  
  - *Config:* Uses default loading options.  
  - *Connections:* Outputs documents to "Qdrant Vector Store".  
  - *Failure Modes:* Data format issues, empty document arrays.  

- **Embeddings OpenAI**  
  - *Type:* OpenAI Embeddings Node  
  - *Role:* Generates vector embeddings for document chunks using OpenAI embeddings API.  
  - *Config:* Uses default embedding model and options. Requires OpenAI credentials.  
  - *Connections:* Embeddings sent to "Qdrant Vector Store" as AI embeddings.  
  - *Failure Modes:* API key invalid, rate limits, network errors.  

- **Qdrant Vector Store**  
  - *Type:* Qdrant Vector Store (Insert Mode)  
  - *Role:* Inserts the generated embeddings and documents into the specified Qdrant collection.  
  - *Config:* Collection set to `YOUR_QDRANT_COLLECTION`.  
  - *Connections:* Outputs back to "Loop Over Items" to continue processing next batch.  
  - *Failure Modes:* Connectivity issues, collection not found, insertion failures.  

---

#### 2.2 AI Conversational Query Handling

**Overview:**  
This block listens for incoming chat messages, uses the Qdrant vector store to retrieve relevant proposal knowledge, and leverages the GPT-4o-mini chat model to generate AI responses based on retrieved context.

**Nodes Involved:**  
- When chat message received  
- AI Agent  
- OpenAI Chat Model  
- Qdrant Vector Store1  
- Embeddings OpenAI1  

**Node Details:**

- **When chat message received**  
  - *Type:* Langchain Chat Trigger  
  - *Role:* Entry trigger node for chat-based AI interactions. Listens for incoming chat messages.  
  - *Config:* Default options.  
  - *Connections:* Sends data to "AI Agent".  
  - *Failure Modes:* Connectivity or event listener errors.  

- **AI Agent**  
  - *Type:* Langchain Agent Node  
  - *Role:* Orchestrates AI interaction, integrates language model and vector store tools for knowledge retrieval and answer generation.  
  - *Config:* Default options, configured to use the "OpenAI Chat Model" as language model and "Qdrant Vector Store1" as a tool.  
  - *Connections:* Receives chat input from trigger; sends queries to vector store tool and language model; outputs AI response.  
  - *Failure Modes:* Misconfiguration of tools or models, timeout with API calls.  

- **OpenAI Chat Model**  
  - *Type:* Langchain OpenAI Chat Model  
  - *Role:* Provides GPT-based language generation for AI Agent.  
  - *Config:* Model set to `gpt-4o-mini` (a smaller GPT-4 variant).  
  - *Connections:* Connected as the language model for the AI Agent.  
  - *Failure Modes:* API key issues, rate limiting, model unavailability.  

- **Qdrant Vector Store1**  
  - *Type:* Qdrant Vector Store (Retrieve-As-Tool Mode)  
  - *Role:* Acts as a retrieval tool within the AI Agent to fetch relevant documents from the knowledge base.  
  - *Config:* Collection set to `YOUR_QDRANT_COLLECTION`; tool name "proposal_knowledge_base" with description guiding the agent to use it for proposal-related info.  
  - *Connections:* Connected as AI tool for the AI Agent; receives embeddings from "Embeddings OpenAI1".  
  - *Failure Modes:* Collection access issues, retrieval failures, empty context responses.  

- **Embeddings OpenAI1**  
  - *Type:* OpenAI Embeddings Node  
  - *Role:* Generates embeddings for the AI Agent’s query to facilitate vector retrieval.  
  - *Config:* Default options; requires OpenAI credentials.  
  - *Connections:* Embeddings output to Qdrant Vector Store1 as input for retrieval.  
  - *Failure Modes:* API errors, rate limits.  

---

### 3. Summary Table

| Node Name                  | Node Type                             | Functional Role                               | Input Node(s)              | Output Node(s)              | Sticky Note                                                                                                                     |
|----------------------------|-------------------------------------|----------------------------------------------|----------------------------|-----------------------------|--------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger                      | Starts the workflow manually                  | –                          | Get Files from S3           |                                                                                                                                |
| Get Files from S3           | AWS S3 (Get All)                    | Lists all files in S3 bucket                   | When clicking ‘Test workflow’ | Loop Over Items             |                                                                                                                                |
| Loop Over Items             | Split In Batches                    | Iterates over each file key                    | Get Files from S3           | Download Files from AWS      |                                                                                                                                |
| Download Files from AWS     | AWS S3 (Download)                   | Downloads the actual PDF files                  | Loop Over Items             | Extract from File            |                                                                                                                                |
| Extract from File           | Extract From File                   | Extracts text from PDF binary                   | Download Files from AWS     | Qdrant Vector Store          |                                                                                                                                |
| Recursive Character Text Splitter | Text Splitter (Recursive Character) | Splits extracted text into chunks               | –                          | Default Data Loader          |                                                                                                                                |
| Default Data Loader         | Document Default Data Loader        | Loads text chunks as documents                   | Recursive Character Text Splitter | Qdrant Vector Store          |                                                                                                                                |
| Embeddings OpenAI           | OpenAI Embeddings                   | Generates vector embeddings for documents       | –                          | Qdrant Vector Store          |                                                                                                                                |
| Qdrant Vector Store         | Qdrant Vector Store (Insert Mode)  | Inserts embeddings into vector collection        | Extract from File, Embeddings OpenAI, Default Data Loader | Loop Over Items             |                                                                                                                                |
| When chat message received  | Langchain Chat Trigger              | Triggers AI query on incoming chat messages     | –                          | AI Agent                    |                                                                                                                                |
| AI Agent                   | Langchain Agent                    | Handles AI chat logic integrating vector store and LLM | When chat message received | –                           |                                                                                                                                |
| OpenAI Chat Model           | Langchain OpenAI Chat Model         | Provides GPT-4o-mini language model              | –                          | AI Agent                    |                                                                                                                                |
| Qdrant Vector Store1        | Qdrant Vector Store (Retrieve Mode) | Acts as retrieval tool for proposal knowledge base | Embeddings OpenAI1          | AI Agent                    | Call this tool to search the vector store knowledge base for proposal-related data. If context is empty, say you don't know the answer. |
| Embeddings OpenAI1          | OpenAI Embeddings                   | Generates embeddings for queries                  | –                          | Qdrant Vector Store1        |                                                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: Manually start the workflow for testing or batch processing.  

2. **Add AWS S3 Node to List All Files**  
   - Type: AWS S3  
   - Operation: Get All  
   - Bucket Name: Set to your S3 bucket (e.g., `YOUR_S3_BUCKET`)  
   - Connect Manual Trigger output to this node’s input.  
   - Configure AWS credentials with read access.  

3. **Add Split In Batches Node**  
   - Type: Split In Batches  
   - Batch Size: Expression `={{ $json.Key.length }}` to process all files in one batch (or adjust as needed).  
   - Connect output of AWS S3 (Get All) to input of this node.  

4. **Add AWS S3 Node to Download Files**  
   - Type: AWS S3  
   - Operation: Download File  
   - Bucket Name: Same as above.  
   - File Key: Set to `={{ $json.Key }}` dynamically from batch item.  
   - Connect from Split In Batches output.  
   - Use appropriate AWS credentials.  

5. **Add Extract From File Node**  
   - Type: Extract From File  
   - Operation: `pdf`  
   - Binary Property: `data` (from downloaded file)  
   - Connect from AWS S3 (Download) node.  

6. **Add Recursive Character Text Splitter Node**  
   - Type: Text Splitter (Recursive Character)  
   - Use default splitting parameters.  
   - Connect from Extract From File node.  

7. **Add Default Data Loader Node**  
   - Type: Document Default Data Loader  
   - Use default options.  
   - Connect from Text Splitter node.  

8. **Add OpenAI Embeddings Node**  
   - Type: OpenAI Embeddings  
   - Configure with OpenAI API credentials.  
   - Use default embedding model/settings.  
   - Connect from Default Data Loader node.  

9. **Add Qdrant Vector Store Node (Insert Mode)**  
   - Type: Qdrant Vector Store  
   - Mode: Insert  
   - Collection: Set to your Qdrant vector collection name (e.g., `YOUR_QDRANT_COLLECTION`)  
   - Connect inputs from Extract From File, Embeddings OpenAI, and Default Data Loader nodes as appropriate.  
   - Connect output back to Split In Batches to continue processing.  
   - Configure Qdrant credentials and connection parameters.  

10. **Add Langchain Chat Trigger Node**  
    - Type: Langchain Chat Trigger  
    - Purpose: Trigger AI chat on incoming messages.  

11. **Add Langchain Agent Node**  
    - Type: Langchain Agent  
    - Configure with default options.  
    - Connect input from Chat Trigger node.  

12. **Add OpenAI Chat Model Node**  
    - Type: Langchain OpenAI Chat Model  
    - Model: `gpt-4o-mini`  
    - Connect output to AI Agent node as language model.  
    - Configure with OpenAI API credentials.  

13. **Add OpenAI Embeddings Node (for queries)**  
    - Type: OpenAI Embeddings  
    - Configure with OpenAI credentials.  
    - Connect output to Qdrant Vector Store node (Retrieve mode).  

14. **Add Qdrant Vector Store Node (Retrieve-As-Tool Mode)**  
    - Type: Qdrant Vector Store  
    - Mode: Retrieve-As-Tool  
    - Collection: Same as insert node (`YOUR_QDRANT_COLLECTION`)  
    - Tool Name: `proposal_knowledge_base`  
    - Tool Description: "Call this tool to search the vector store knowledge base for proposal-related data. If context is empty, say you don't know the answer."  
    - Connect AI embeddings output to this node.  
    - Connect output as AI tool to AI Agent node.  

15. **Connect AI Agent Output to desired response or output node as per your environment.**

---

### 5. General Notes & Resources

| Note Content                                                                                                     | Context or Link                                                                                         |
|-----------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| Use AWS credentials with appropriate permissions for S3 bucket access (read for listing and download).          | AWS IAM and credential setup documentation.                                                           |
| OpenAI API keys required for both embedding generation and chat completion; monitor usage and rate limits.      | https://platform.openai.com/docs/api-reference/authentication                                          |
| Qdrant collection must be pre-created and accessible; ensure correct API endpoint and authentication configured. | https://qdrant.tech/documentation/                                                                    |
| The AI Agent integrates retrieval and generation for RAG workflows; tune agent settings as needed.              | https://docs.n8n.io/nodes/n8n-nodes-langchain/agent/                                                  |
| GPT-4o-mini is a smaller GPT-4 variant ideal for cost-effective chat completions with good capability.          | https://learn.microsoft.com/en-us/azure/cognitive-services/openai/concepts/models#openai-models         |

---

**Disclaimer:** The text provided is exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected content. All data handled is legal and public.

---