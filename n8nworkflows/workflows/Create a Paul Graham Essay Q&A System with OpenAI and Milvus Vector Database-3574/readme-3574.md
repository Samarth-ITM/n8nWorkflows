Create a Paul Graham Essay Q&A System with OpenAI and Milvus Vector Database

https://n8nworkflows.xyz/workflows/create-a-paul-graham-essay-q-a-system-with-openai-and-milvus-vector-database-3574


# Create a Paul Graham Essay Q&A System with OpenAI and Milvus Vector Database

### 1. Workflow Overview

This workflow implements a question-answering (QA) system based on the essays of Paul Graham. It is designed for users who want to query the content of these essays semantically using natural language questions. The workflow is divided into two main logical blocks:

**1.1 Data Collection & Processing**  
This block scrapes the list of Paul Graham essays from his website, fetches the essay contents, extracts the main text, splits the text into manageable chunks, generates vector embeddings using OpenAI, and loads these vectors into a Milvus vector database collection named "my_collection". This prepares the data for efficient semantic search.

**1.2 Chat Interaction & Question Answering**  
This block listens for incoming chat messages (questions), retrieves relevant essay chunks from the Milvus vector store using semantic search, and generates answers using an OpenAI chat model (GPT-4o-mini). It provides an interactive QA interface backed by the vector database.

---

### 2. Block-by-Block Analysis

#### 2.1 Data Collection & Processing

- **Overview:**  
  This block scrapes Paul Graham’s essays, extracts their text content, processes the text into chunks, generates embeddings, and inserts them into the Milvus vector store. It is triggered manually to refresh or initialize the data.

- **Nodes Involved:**  
  - When clicking "Execute Workflow" (Manual Trigger)  
  - Fetch Essay List (HTTP Request)  
  - Extract essay names (HTML Extract)  
  - Split out into items (Split Out)  
  - Limit to first 3 (Limit)  
  - Fetch essay texts (HTTP Request)  
  - Extract Text Only (HTML Extract)  
  - Recursive Character Text Splitter (Text Splitter)  
  - Default Data Loader (Document Loader)  
  - Embeddings OpenAI (Embeddings Generator)  
  - Milvus Vector Store (Vector Store Insert)  
  - Sticky Notes (3, 5, and initial instructions)

- **Node Details:**

  1. **When clicking "Execute Workflow"**  
     - Type: Manual Trigger  
     - Role: Starts the scraping and loading process on demand  
     - Input: None  
     - Output: Triggers "Fetch Essay List"  
     - Failures: None expected; manual start

  2. **Fetch Essay List**  
     - Type: HTTP Request  
     - Role: Downloads the HTML page listing Paul Graham essays from http://www.paulgraham.com/articles.html  
     - Configuration: Simple GET request, no auth or special headers  
     - Output: HTML content passed to "Extract essay names"  
     - Failures: Network errors, site down, or changed page structure

  3. **Extract essay names**  
     - Type: HTML Extract  
     - Role: Parses the HTML to extract all essay links from nested tables using CSS selector `table table a`  
     - Extracted attribute: `href` (essay URLs)  
     - Output: Array of essay URLs to "Split out into items"  
     - Failures: Selector mismatch if website structure changes

  4. **Split out into items**  
     - Type: Split Out  
     - Role: Splits the array of essay URLs into individual items for sequential processing  
     - Output: Single essay URL per item to "Limit to first 3"  
     - Failures: None expected

  5. **Limit to first 3**  
     - Type: Limit  
     - Role: Restricts processing to the first 3 essays only (likely for demo or testing)  
     - Output: Passes limited essay URLs to "Fetch essay texts"  
     - Failures: None expected

  6. **Fetch essay texts**  
     - Type: HTTP Request  
     - Role: Fetches the full HTML content of each essay by constructing URL `http://www.paulgraham.com/{{ $json.essay }}`  
     - Output: HTML content to "Extract Text Only"  
     - Failures: Network errors, 404 if essay URL invalid

  7. **Extract Text Only**  
     - Type: HTML Extract  
     - Role: Extracts the main textual content from the essay page using CSS selector `body` while skipping `img` and `nav` tags  
     - Output: Extracted essay text to "Milvus Vector Store" and "Default Data Loader"  
     - Failures: Extraction errors if page structure changes

  8. **Recursive Character Text Splitter**  
     - Type: Text Splitter  
     - Role: Splits large essay text into chunks of 6000 characters for embedding  
     - Output: Chunks to "Default Data Loader"  
     - Failures: None expected

  9. **Default Data Loader**  
     - Type: Document Loader  
     - Role: Loads the chunked text into document format for embedding generation  
     - Input: Chunked text from splitter  
     - Output: Document data to "Milvus Vector Store"  
     - Failures: Expression errors if input data missing

  10. **Embeddings OpenAI**  
      - Type: OpenAI Embeddings  
      - Role: Generates vector embeddings for each text chunk using OpenAI embeddings API  
      - Configuration: Uses default OpenAI credentials and settings  
      - Output: Embeddings to "Milvus Vector Store"  
      - Failures: API quota, auth errors, timeouts

  11. **Milvus Vector Store**  
      - Type: Vector Store (Milvus)  
      - Role: Inserts embeddings and documents into the "my_collection" Milvus collection  
      - Configuration: Insert mode with `clearCollection` set to true (clears existing data before insert)  
      - Input: Embeddings and documents  
      - Output: None  
      - Failures: Connection errors, collection not found, auth issues

  12. **Sticky Notes**  
      - Provide contextual explanations and setup instructions for the scraping and loading block.

---

#### 2.2 Chat Interaction & Question Answering

- **Overview:**  
  This block listens for incoming chat messages (questions), retrieves relevant essay chunks from Milvus using semantic search, and generates answers with an OpenAI chat model. It enables interactive querying of the stored essays.

- **Nodes Involved:**  
  - When chat message received (Chat Trigger)  
  - Milvus Vector Store1 (Milvus Vector Store Retriever)  
  - Milvus Vector Store Retriever (Retriever)  
  - Embeddings OpenAI1 (Embeddings Generator)  
  - Q&A Chain to Retrieve from Milvus and Answer Question (Retrieval QA Chain)  
  - OpenAI Chat Model (Chat Completion)  
  - Sticky Note1 (Step 2 instructions)  
  - Sticky Note2 (empty, possibly for future notes)

- **Node Details:**

  1. **When chat message received**  
     - Type: Chat Trigger (LangChain)  
     - Role: Webhook listener for incoming chat messages (questions)  
     - Output: Passes question to "Q&A Chain to Retrieve from Milvus and Answer Question"  
     - Failures: Webhook misconfiguration, network issues

  2. **Milvus Vector Store1**  
     - Type: Vector Store (Milvus)  
     - Role: Connects to the "my_collection" Milvus collection for retrieval  
     - Configuration: Uses existing Milvus credentials and collection name  
     - Output: Provides vector store to "Milvus Vector Store Retriever"  
     - Failures: Connection or auth errors

  3. **Milvus Vector Store Retriever**  
     - Type: Retriever (Vector Store)  
     - Role: Retrieves relevant document chunks from Milvus based on query embeddings  
     - Input: Vector store from "Milvus Vector Store1"  
     - Output: Retrieved documents to "Q&A Chain to Retrieve from Milvus and Answer Question"  
     - Failures: Retrieval errors, empty results

  4. **Embeddings OpenAI1**  
     - Type: OpenAI Embeddings  
     - Role: Generates embeddings for the incoming question to perform semantic search  
     - Output: Embeddings to "Milvus Vector Store1" (used internally)  
     - Failures: API errors, rate limits

  5. **Q&A Chain to Retrieve from Milvus and Answer Question**  
     - Type: Retrieval QA Chain (LangChain)  
     - Role: Combines retrieved documents and OpenAI Chat Model to generate an answer to the user’s question  
     - Inputs: Chat message, retriever, language model  
     - Output: Answer sent back to chat interface  
     - Failures: Model errors, retrieval failures, timeout

  6. **OpenAI Chat Model**  
     - Type: Chat Completion (OpenAI GPT-4o-mini)  
     - Role: Generates natural language answers based on retrieved context and user question  
     - Configuration: Model set to "gpt-4o-mini"  
     - Output: Answer to QA Chain  
     - Failures: API quota, auth errors, model unavailability

  7. **Sticky Notes**  
     - Provide instructions for Step 2 — chatting with the QA system.

---

### 3. Summary Table

| Node Name                                   | Node Type                                   | Functional Role                                  | Input Node(s)                       | Output Node(s)                          | Sticky Note                                                                                           |
|---------------------------------------------|---------------------------------------------|-------------------------------------------------|-----------------------------------|---------------------------------------|-----------------------------------------------------------------------------------------------------|
| When clicking "Execute Workflow"             | Manual Trigger                              | Starts scraping and loading process             | None                              | Fetch Essay List                      | ## Step 1<br>1. Set up a Milvus server based on [this guide](https://milvus.io/docs/install_standalone-docker-compose.md). And then create a collection named `my_collection`.<br>2. Click this workflow to load scrape and load Paul Graham essays to Milvus collection. |
| Fetch Essay List                             | HTTP Request                               | Downloads Paul Graham essays list page           | When clicking "Execute Workflow"  | Extract essay names                   | ## Scrape latest Paul Graham essays                                                                 |
| Extract essay names                         | HTML Extract                               | Extracts essay URLs from HTML                     | Fetch Essay List                  | Split out into items                  | ## Scrape latest Paul Graham essays                                                                 |
| Split out into items                        | Split Out                                  | Splits essay URLs into individual items          | Extract essay names               | Limit to first 3                     | ## Scrape latest Paul Graham essays                                                                 |
| Limit to first 3                           | Limit                                      | Limits processing to first 3 essays              | Split out into items              | Fetch essay texts                   | ## Scrape latest Paul Graham essays                                                                 |
| Fetch essay texts                          | HTTP Request                               | Fetches full HTML of each essay                   | Limit to first 3                 | Extract Text Only                   | ## Scrape latest Paul Graham essays                                                                 |
| Extract Text Only                         | HTML Extract                               | Extracts main essay text content                   | Fetch essay texts                | Milvus Vector Store, Default Data Loader | ## Scrape latest Paul Graham essays                                                                 |
| Recursive Character Text Splitter          | Text Splitter                              | Splits essay text into chunks of 6000 characters | Default Data Loader              | Default Data Loader                  | ## Load into Milvus vector store                                                                     |
| Default Data Loader                        | Document Loader                            | Loads chunked text for embedding generation       | Recursive Character Text Splitter | Milvus Vector Store                 | ## Load into Milvus vector store                                                                     |
| Embeddings OpenAI                         | OpenAI Embeddings                          | Generates vector embeddings for text chunks       | Default Data Loader              | Milvus Vector Store                 | ## Load into Milvus vector store                                                                     |
| Milvus Vector Store                       | Vector Store (Milvus)                      | Inserts embeddings and documents into Milvus      | Embeddings OpenAI, Extract Text Only, Default Data Loader | None                              | ## Load into Milvus vector store                                                                     |
| When chat message received                 | Chat Trigger                              | Listens for incoming chat questions               | None                           | Q&A Chain to Retrieve from Milvus and Answer Question | ## Step 2<br>Chat with this QA Chain with Milvus retriever                                          |
| Milvus Vector Store1                      | Vector Store (Milvus)                      | Connects to Milvus collection for retrieval       | Embeddings OpenAI1              | Milvus Vector Store Retriever       |                                                                                                     |
| Embeddings OpenAI1                        | OpenAI Embeddings                          | Generates embeddings for incoming chat queries    | None                           | Milvus Vector Store1                |                                                                                                     |
| Milvus Vector Store Retriever              | Retriever (Vector Store)                   | Retrieves relevant documents from Milvus          | Milvus Vector Store1            | Q&A Chain to Retrieve from Milvus and Answer Question |                                                                                                     |
| Q&A Chain to Retrieve from Milvus and Answer Question | Retrieval QA Chain (LangChain)             | Retrieves context and generates answer             | When chat message received, Milvus Vector Store Retriever, OpenAI Chat Model | None                              | ## Step 2<br>Chat with this QA Chain with Milvus retriever                                          |
| OpenAI Chat Model                         | Chat Completion (OpenAI GPT-4o-mini)      | Generates natural language answers                 | Q&A Chain to Retrieve from Milvus and Answer Question | Q&A Chain to Retrieve from Milvus and Answer Question |                                                                                                     |
| Sticky Note3                              | Sticky Note                               | Provides context for scraping block                | None                           | None                              | ## Scrape latest Paul Graham essays                                                                 |
| Sticky Note5                              | Sticky Note                               | Provides context for Milvus loading block          | None                           | None                              | ## Load into Milvus vector store                                                                     |
| Sticky Note                               | Sticky Note                               | Setup instructions for Step 1                       | None                           | None                              | ## Step 1<br>1. Set up a Milvus server based on [this guide](https://milvus.io/docs/install_standalone-docker-compose.md). And then create a collection named `my_collection`.<br>2. Click this workflow to load scrape and load Paul Graham essays to Milvus collection. |
| Sticky Note1                              | Sticky Note                               | Setup instructions for Step 2                       | None                           | None                              | ## Step 2<br>Chat with this QA Chain with Milvus retriever                                          |
| Sticky Note2                              | Sticky Note                               | Empty note                                           | None                           | None                              |                                                                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: To start the scraping and loading process manually.

2. **Create HTTP Request Node "Fetch Essay List"**  
   - URL: `http://www.paulgraham.com/articles.html`  
   - Method: GET  
   - Connect Manual Trigger → Fetch Essay List

3. **Create HTML Extract Node "Extract essay names"**  
   - Operation: Extract HTML Content  
   - Extraction Values: Extract attribute `href` from CSS selector `table table a`  
   - Connect Fetch Essay List → Extract essay names

4. **Create Split Out Node "Split out into items"**  
   - Field to split out: `essay` (the extracted href array)  
   - Connect Extract essay names → Split out into items

5. **Create Limit Node "Limit to first 3"**  
   - Max Items: 3 (to limit processing to first 3 essays)  
   - Connect Split out into items → Limit to first 3

6. **Create HTTP Request Node "Fetch essay texts"**  
   - URL: `http://www.paulgraham.com/{{ $json.essay }}` (use expression to construct URL)  
   - Method: GET  
   - Connect Limit to first 3 → Fetch essay texts

7. **Create HTML Extract Node "Extract Text Only"**  
   - Operation: Extract HTML Content  
   - Extraction Values: Extract text content from `body` CSS selector  
   - Skip selectors: `img,nav` to exclude images and navigation  
   - Connect Fetch essay texts → Extract Text Only

8. **Create Recursive Character Text Splitter Node**  
   - Chunk Size: 6000 characters  
   - Connect Extract Text Only → Recursive Character Text Splitter

9. **Create Default Data Loader Node**  
   - Input JSON Data: Use expression `={{ $('Extract Text Only').item.json.data }}` to pass extracted text  
   - Connect Recursive Character Text Splitter → Default Data Loader

10. **Create OpenAI Embeddings Node "Embeddings OpenAI"**  
    - Use OpenAI credentials (set up in n8n credentials)  
    - Default options  
    - Connect Default Data Loader → Embeddings OpenAI

11. **Create Milvus Vector Store Node**  
    - Mode: Insert  
    - Options: Clear collection before insert (clearCollection = true)  
    - Milvus Collection: Select or enter `my_collection`  
    - Connect Embeddings OpenAI → Milvus Vector Store  
    - Also connect Extract Text Only → Milvus Vector Store (for document data)

12. **Create Chat Trigger Node "When chat message received"**  
    - Set up webhook to receive chat messages  
    - Connect to "Q&A Chain to Retrieve from Milvus and Answer Question"

13. **Create OpenAI Embeddings Node "Embeddings OpenAI1"**  
    - For embedding incoming chat questions  
    - Use OpenAI credentials  
    - Connect to Milvus Vector Store1

14. **Create Milvus Vector Store Node "Milvus Vector Store1"**  
    - Milvus Collection: `my_collection`  
    - Connect Embeddings OpenAI1 → Milvus Vector Store1

15. **Create Retriever Node "Milvus Vector Store Retriever"**  
    - Connect Milvus Vector Store1 → Milvus Vector Store Retriever

16. **Create OpenAI Chat Model Node**  
    - Model: `gpt-4o-mini`  
    - Use OpenAI credentials  
    - Connect to Q&A Chain

17. **Create Retrieval QA Chain Node "Q&A Chain to Retrieve from Milvus and Answer Question"**  
    - Inputs: Chat message from trigger, retriever, and OpenAI Chat Model  
    - Connect When chat message received → Q&A Chain  
    - Connect Milvus Vector Store Retriever → Q&A Chain  
    - Connect OpenAI Chat Model → Q&A Chain

18. **Connect Q&A Chain output to Chat interface**  
    - This completes the chat interaction flow.

19. **Add Sticky Notes**  
    - Add notes for setup instructions and block explanations as per original workflow.

20. **Configure Credentials**  
    - OpenAI API key for embeddings and chat model nodes  
    - Milvus server credentials and connection details for vector store nodes

21. **Set up Milvus Server**  
    - Follow official Milvus standalone Docker guide: https://milvus.io/docs/install_standalone-docker.md  
    - Create collection named `my_collection`

---

### 5. General Notes & Resources

| Note Content                                                                                                       | Context or Link                                                                                             |
|--------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| Step 1: Set up a Milvus server based on [this guide](https://milvus.io/docs/install_standalone-docker-compose.md). And then create a collection named `my_collection`. | Workflow setup instructions                                                                                 |
| Scrape latest Paul Graham essays                                                                                   | Workflow block context                                                                                       |
| Load into Milvus vector store                                                                                       | Workflow block context                                                                                       |
| Step 2: Chat with this QA Chain with Milvus retriever                                                              | Workflow chat interaction instructions                                                                      |
| Milvus official installation guide                                                                                  | https://milvus.io/docs/install_standalone-docker.md                                                         |
| OpenAI embeddings documentation                                                                                     | https://platform.openai.com/docs/guides/embeddings                                                           |

---

This structured document fully describes the workflow’s architecture, node configurations, data flow, and setup instructions, enabling users or AI agents to understand, reproduce, and maintain the Paul Graham Essay Q&A system built with n8n, OpenAI, and Milvus.