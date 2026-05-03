Document Q&A with RAG: Query PDF Content using Weaviate and OpenAI

https://n8nworkflows.xyz/workflows/document-q-a-with-rag--query-pdf-content-using-weaviate-and-openai-7170


# Document Q&A with RAG: Query PDF Content using Weaviate and OpenAI

### 1. Workflow Overview

This workflow enables users to upload a PDF document, convert its content into embeddings, store those embeddings in a Weaviate vector database, and then ask questions about the document via a chat interface. It implements a Retrieval-Augmented Generation (RAG) pattern by combining Weaviate for vector similarity search and OpenAI models for generating answers grounded in document context.

**Target Use Cases:**  
- Querying large PDF documents interactively  
- Building document question-answering systems leveraging vector search  
- Demonstrating integration of n8n with Weaviate and OpenAI for knowledge retrieval  

**Logical Blocks:**  
- **1.1 PDF Upload and Text Extraction:** Receives PDF input, extracts text.  
- **1.2 Text Splitting and Embeddings Generation:** Splits extracted text into chunks, creates embeddings with OpenAI.  
- **1.3 Embeddings Storage:** Inserts embeddings into Weaviate vector store.  
- **1.4 Query Reception and Processing:** Receives user queries via chat webhook.  
- **1.5 Vector Retrieval and Answer Generation:** Retrieves relevant chunks from Weaviate and uses OpenAI chat model to generate answers.  

---

### 2. Block-by-Block Analysis

#### 2.1 PDF Upload and Text Extraction

- **Overview:**  
  This block handles manual upload of a PDF file via a form, extracts the full text content to be processed.

- **Nodes Involved:**  
  - Upload PDF (Form Trigger)  
  - Extract from File (PDF extraction)  
  - Edit Fields (Set JSON text field)  

- **Node Details:**  

  - **Upload PDF**  
    - *Type:* Form Trigger  
    - *Role:* Entry point for PDF file upload via n8n form webhook  
    - *Config:* Single required PDF file input field  
    - *Connections:* Output → Extract from File  
    - *Failure modes:* Missing file upload, incorrect file format  

  - **Extract from File**  
    - *Type:* Extract from File (Base node)  
    - *Role:* Extracts text from the uploaded PDF file  
    - *Config:* Max 99 pages, PDF binary property input "PDF_File"  
    - *Connections:* Output → Edit Fields  
    - *Failure modes:* Corrupted PDF, extraction errors, large file timeouts  

  - **Edit Fields**  
    - *Type:* Set  
    - *Role:* Maps extracted text into a JSON attribute "text" for downstream use  
    - *Config:* Sets field "text" = extracted $json.text  
    - *Connections:* Output → Weaviate Vector Store (for insertion)  
    - *Failure modes:* Missing or empty extracted text  

---

#### 2.2 Text Splitting and Embeddings Generation

- **Overview:**  
  This block splits the extracted text into manageable chunks and generates vector embeddings for each chunk using OpenAI.

- **Nodes Involved:**  
  - Recursive Character Text Splitter1  
  - Default Data Loader  
  - Embeddings OpenAI  

- **Node Details:**  

  - **Recursive Character Text Splitter1**  
    - *Type:* Recursive Character Text Splitter  
    - *Role:* Splits text into chunks of 500 characters to optimize embedding generation and vector storage  
    - *Config:* chunkSize = 500 characters  
    - *Connections:* Output → Default Data Loader  
    - *Failure modes:* Very short or empty text input, chunking errors  

  - **Default Data Loader**  
    - *Type:* Document Default Data Loader  
    - *Role:* Converts text chunks into document objects compatible with LangChain nodes  
    - *Config:* Loads JSON data from expression $json.text, no metadata added in this example (optional metadata can be configured)  
    - *Connections:* Output → Weaviate Vector Store (as documents)  
    - *Failure modes:* Improper JSON data, missing text property  

  - **Embeddings OpenAI**  
    - *Type:* OpenAI Embeddings Node  
    - *Role:* Generates embeddings vectors for text chunks  
    - *Config:* Uses OpenAI credentials, default embedding parameters  
    - *Connections:* ai_embedding output → Weaviate Vector Store (embedding input)  
    - *Failure modes:* API key issues, rate limits, network errors  

---

#### 2.3 Embeddings Storage

- **Overview:**  
  This block inserts the generated embeddings and document chunks into a Weaviate vector store collection named "FileUpload."

- **Nodes Involved:**  
  - Weaviate Vector Store  

- **Node Details:**  

  - **Weaviate Vector Store**  
    - *Type:* LangChain Vector Store for Weaviate  
    - *Role:* Inserts embeddings and documents into Weaviate collection "FileUpload"  
    - *Config:* Mode = Insert, text key specified as "text", uses configured Weaviate credentials  
    - *Connections:* Receives documents from Default Data Loader and embeddings from Embeddings OpenAI  
    - *Failure modes:* Weaviate authentication errors, connectivity issues, schema mismatch in collection  

---

#### 2.4 Query Reception and Processing

- **Overview:**  
  This block listens for chat messages (user queries) and triggers the question-answering process.

- **Nodes Involved:**  
  - When chat message received  
  - Question and Answer Chain  

- **Node Details:**  

  - **When chat message received**  
    - *Type:* LangChain Chat Trigger  
    - *Role:* Webhook to receive chat input for querying the PDF content  
    - *Config:* Webhook ID auto-generated, no extra options  
    - *Connections:* Output → Question and Answer Chain  
    - *Failure modes:* Webhook misconfiguration, missing chat input  

  - **Question and Answer Chain**  
    - *Type:* LangChain Chain Retrieval QA  
    - *Role:* Runs a retrieval-augmented generation process to answer queries based only on Weaviate vector store context  
    - *Config:* Text prompt set as "Using only the attached Weaviate vector store collection (and no external knowledge), answer the following query: {{ $json.chatInput }}"  
    - *Connections:* ai_retriever input → Vector Store Retriever  
                    main output → downstream response or UI (not shown)  
    - *Failure modes:* Empty or malformed queries, retrieval failures, prompt formatting errors  

---

#### 2.5 Vector Retrieval and Answer Generation

- **Overview:**  
  This block retrieves contextually relevant document chunks from Weaviate and uses the OpenAI chat model to generate an answer.

- **Nodes Involved:**  
  - Weaviate Vector Store1 (retrieval)  
  - Vector Store Retriever  
  - OpenAI Chat Model  

- **Node Details:**  

  - **Weaviate Vector Store1**  
    - *Type:* LangChain Vector Store for Weaviate (retrieval mode)  
    - *Role:* Accesses the "FileUpload" collection for similarity search during query answering  
    - *Config:* Mode = list, cached result name "FileUpload"  
    - *Connections:* ai_vectorStore output → Vector Store Retriever  
    - *Failure modes:* Collection access errors, stale cache, authentication issues  

  - **Vector Store Retriever**  
    - *Type:* LangChain Retriever Vector Store  
    - *Role:* Retrieves relevant vectors/chunks from Weaviate based on the query  
    - *Config:* Default parameters  
    - *Connections:* ai_retriever output → Question and Answer Chain  
    - *Failure modes:* No relevant vectors found, retrieval timeouts  

  - **OpenAI Chat Model**  
    - *Type:* LangChain Chat OpenAI Model  
    - *Role:* Generates the final answer text using GPT-4.1-mini model  
    - *Config:* Model selected from list: "gpt-4.1-mini", uses OpenAI credentials  
    - *Connections:* ai_languageModel output → Question and Answer Chain  
    - *Failure modes:* API limits, model unavailability, network errors  

---

### 3. Summary Table

| Node Name                   | Node Type                                        | Functional Role                                | Input Node(s)                  | Output Node(s)               | Sticky Note                                                                                                            |
|-----------------------------|-------------------------------------------------|-----------------------------------------------|-------------------------------|-----------------------------|------------------------------------------------------------------------------------------------------------------------|
| Upload PDF                  | Form Trigger                                    | PDF file upload entry point                    |                               | Extract from File            |                                                                                                                        |
| Extract from File            | Extract From File (PDF)                         | Extracts text from uploaded PDF                 | Upload PDF                    | Edit Fields                 |                                                                                                                        |
| Edit Fields                 | Set                                             | Sets extracted text into JSON field             | Extract from File             | Weaviate Vector Store        |                                                                                                                        |
| Recursive Character Text Splitter1 | Recursive Character Text Splitter                | Splits text into 500-char chunks                 |                               | Default Data Loader          |                                                                                                                        |
| Default Data Loader          | Document Default Data Loader                     | Loads text chunks as documents                   | Recursive Character Text Splitter1 | Weaviate Vector Store        | _Note: No metadata added by default; can be configured in this node._                                                 |
| Embeddings OpenAI           | OpenAI Embeddings                               | Generates embeddings for text chunks             |                               | Weaviate Vector Store        |                                                                                                                        |
| Weaviate Vector Store       | LangChain Vector Store (Weaviate)               | Inserts embeddings and documents into Weaviate | Default Data Loader, Embeddings OpenAI, Edit Fields |                             |                                                                                                                        |
| When chat message received  | LangChain Chat Trigger                          | Receives user query via webhook                  |                               | Question and Answer Chain    |                                                                                                                        |
| Question and Answer Chain   | LangChain Chain Retrieval QA                    | Runs RAG QA chain using Weaviate and OpenAI     | When chat message received, Vector Store Retriever, OpenAI Chat Model |                             |                                                                                                                        |
| Weaviate Vector Store1      | LangChain Vector Store (Weaviate retrieval)    | Accesses Weaviate collection for retrieval       |                               | Vector Store Retriever       |                                                                                                                        |
| Vector Store Retriever      | LangChain Retriever Vector Store                | Retrieves relevant document chunks               | Weaviate Vector Store1        | Question and Answer Chain    |                                                                                                                        |
| OpenAI Chat Model           | LangChain Chat OpenAI Model                      | Generates final answer response                   |                               | Question and Answer Chain    |                                                                                                                        |
| Embeddings OpenAI1          | OpenAI Embeddings                               | (Connected to retrieval vector store, unused in main flow) |                               | Weaviate Vector Store1       |                                                                                                                        |
| Sticky Note                 | Sticky Note                                     | Information on manual data upload part           |                               |                             | ## Part 1: Manually upload data \nIn this example, we manually upload a 100+ page article from arXiv called ["A Survey of Large Language Models"](https://arxiv.org/pdf/2303.18223). \n\n_**Note: This is a simple implementation of loading data. You can replace this block with your own (more advanced) data pipeline!**_ |
| Sticky Note1                | Sticky Note                                     | Information on RAG query usage                   |                               |                             | ## Part 3: Perform RAG over PDF file with Weaviate\nEnter your query by running the Chat Node and get a RAG response grounded in context. |
| Sticky Note2                | Sticky Note                                     | Information on embeddings and loading data      |                               |                             | ## Part 2: Embed and load data into Weaviate collection\nWe generate embeddings for the full-text of the article and store them in Weaviate.\n\n_**Note: We don't add any metadata to Weaviate in this example. To add metadata, click on the Default Data Loader node → `Add Option` → `Metadata`.**_ |
| Sticky Note3                | Sticky Note                                     | General overview and prerequisites               |                               |                             | # RAG over a PDF file with Weaviate\nThis workflow allows you to upload a PDF file and ask questions about it using the Question and Answer Chain and the Weaviate Vector Store nodes.\n\n## Prerequisites\n1.  **An existing Weaviate cluster.** You can view instructions for setting up a **local cluster** with Docker [here](https://weaviate.io/developers/weaviate/installation/docker-compose#starter-docker-compose-file) or a **Weaviate Cloud** cluster [here](https://weaviate.io/developers/wcs/quickstart).\n2.  **API keys** to generate embeddings and power chat models. We use [OpenAI](https://openai.com/), but feel free to switch out the models as you like.\n3.  **Self-hosted n8n instance.** See this [video](https://www.youtube.com/watch?v=kq5bmrjPPAY&t=108s) for how to get set up in just three minutes.\n\n💚  Sign up [here](https://console.weaviate.cloud/?utm_source=recipe&utm_campaign=n8n&utm_content=n8n_arxiv_template) for a 14-day free trial of Weaviate Cloud (no credit card required). |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node: "Upload PDF"**  
   - Type: Form Trigger  
   - Configure form with a single required file field labeled "PDF File" that accepts only `.pdf` files.  
   - This node serves as the webhook entry point for PDF upload.

2. **Create Node: "Extract from File"**  
   - Type: Extract From File (Base node)  
   - Operation: PDF extraction  
   - Binary property input: Set to the file upload field name from the "Upload PDF" node (likely `PDF_File`).  
   - Option: Max pages set to 99.  
   - Connect output of "Upload PDF" to this node.

3. **Create Node: "Edit Fields"**  
   - Type: Set  
   - Add assignment: set a new string field named "text" to `{{$json.text}}` (the extracted text from previous node).  
   - Connect output of "Extract from File" to this node.

4. **Create Node: "Recursive Character Text Splitter1"**  
   - Type: Recursive Character Text Splitter  
   - Set chunk size to 500 characters.  
   - Connect output to the next node (Default Data Loader) in embeddings block.

5. **Create Node: "Default Data Loader"**  
   - Type: Document Default Data Loader  
   - Configure to load JSON data expression: `{{$json.text}}` from the splitter output.  
   - Optional: Add metadata fields if desired by adding options → metadata.  
   - Connect output to the Weaviate Vector Store insertion node.

6. **Create Node: "Embeddings OpenAI"**  
   - Type: OpenAI Embeddings  
   - Use your OpenAI API credentials.  
   - No special options required.  
   - Connect `ai_embedding` output to Weaviate Vector Store insertion node.

7. **Create Node: "Weaviate Vector Store" (Insert Mode)**  
   - Type: LangChain Vector Store for Weaviate  
   - Set mode to "insert".  
   - Specify Weaviate collection as "FileUpload".  
   - Set text key to "text" (the field holding document text).  
   - Use configured Weaviate API credentials.  
   - Connect inputs from Default Data Loader (documents), Embeddings OpenAI (embeddings), and Edit Fields (text).  

8. **Create Node: "When chat message received"**  
   - Type: LangChain Chat Trigger  
   - Use default options; this will create a webhook for chat input.  
   - This node triggers the question answering chain.

9. **Create Node: "Weaviate Vector Store1" (Retrieval Mode)**  
   - Type: LangChain Vector Store for Weaviate  
   - Set mode to "list" to enable retrieval from the "FileUpload" collection.  
   - Use same Weaviate credentials.  
   - Optionally enable caching with name "FileUpload".  

10. **Create Node: "Vector Store Retriever"**  
    - Type: LangChain Retriever Vector Store  
    - No special parameters required.  
    - Connect input from "Weaviate Vector Store1" ai_vectorStore output.  
    - Connect output to "Question and Answer Chain" ai_retriever input.

11. **Create Node: "OpenAI Chat Model"**  
    - Type: LangChain Chat OpenAI Model  
    - Select model "gpt-4.1-mini" from available list.  
    - Use OpenAI API credentials.  
    - Connect output to "Question and Answer Chain" ai_languageModel input.

12. **Create Node: "Question and Answer Chain"**  
    - Type: LangChain Chain Retrieval QA  
    - Set prompt text to:  
      `Using only the attached Weaviate vector store collection (and no external knowledge), answer the following query:\n{{ $json.chatInput }}`  
    - Connect main input from "When chat message received".  
    - Connect ai_retriever input from "Vector Store Retriever".  
    - Connect ai_languageModel input from "OpenAI Chat Model".  

13. **Connect outputs accordingly:**  
    - "When chat message received" → "Question and Answer Chain" (main)  
    - "Weaviate Vector Store1" → "Vector Store Retriever" → "Question and Answer Chain" (ai_retriever)  
    - "OpenAI Chat Model" → "Question and Answer Chain" (ai_languageModel)  

14. **Credentials Setup:**  
    - Configure OpenAI API credentials for Embeddings and Chat nodes.  
    - Configure Weaviate API credentials with correct endpoint and API key for vector store nodes.  

15. **Testing:**  
    - Upload a PDF file via the webhook URL of "Upload PDF" node.  
    - Wait for embedding and insertion to complete.  
    - Send a chat message to the chat webhook of "When chat message received" node.  
    - Receive a context-grounded answer.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Context or Link                                                                                          |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| This workflow is a demonstration of RAG over PDF files using Weaviate and OpenAI embeddings and chat models. It assumes you have a running Weaviate instance (local or cloud) and OpenAI API keys.                                                                                                                                                                                                                                                                                                                                                                          | See Weaviate setup instructions: https://weaviate.io/developers/weaviate/installation/docker-compose#starter-docker-compose-file and https://weaviate.io/developers/wcs/quickstart |
| To run n8n with this workflow, a self-hosted instance is recommended; quick setup video: https://www.youtube.com/watch?v=kq5bmrjPPAY&t=108s                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |                                                                                                        |
| Example PDF used in this demo is "A Survey of Large Language Models" from arXiv: https://arxiv.org/pdf/2303.18223                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |                                                                                                        |
| Free 14-day trial available for Weaviate Cloud: https://console.weaviate.cloud/?utm_source=recipe&utm_campaign=n8n&utm_content=n8n_arxiv_template                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |                                                                                                        |
| The Default Data Loader node can be customized to add metadata to each document chunk to enhance retrieval capabilities.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Metadata configuration in Default Data Loader node                                                    |
| The workflow does not currently implement error handling nodes; consider adding error catchers and retries especially for API calls and file processing steps.                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Best practice for production-grade workflows                                                           |

---

**Disclaimer:**  
The provided analysis is based exclusively on the automated n8n workflow JSON and respects all content policies. The workflow does not contain illegal, offensive, or protected content. All data processed is legal and public.